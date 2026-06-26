---
type: Design
title: 학습 인프라 설계
description: Qwen2.5-32B 풀 파인튜닝 기준 분산 학습 설정, 비용 추정, 안정화 전략, 모니터링 지표.
tags: [model, training, infrastructure]
timestamp: 2026-06-27T00:00:00Z
---

# 학습 인프라 설계

## FSDP vs DeepSpeed ZeRO-3 선택 기준

두 프레임워크 모두 모델 파라미터·그래디언트·옵티마이저 상태를 GPU 간에 샤딩하여
단일 GPU VRAM 한계를 넘는 모델을 풀 파인튜닝할 수 있게 한다.
시 파인튜닝 맥락에서의 선택 기준은 다음과 같다.

### 비교표

| 항목 | FSDP (PyTorch 내장) | DeepSpeed ZeRO-3 |
|------|---------------------|------------------|
| 설치 복잡도 | 낮음 — PyTorch 2.x 기본 포함 | 높음 — 별도 패키지, CUDA 커스텀 커널 |
| HuggingFace 연동 | `Trainer` / `accelerate` 에서 바로 사용 | `deepspeed` config JSON 별도 필요 |
| 통신 효율 | NCCL AllGather 기반 — 안정적 | ZeRO Infinity 지원 (NVMe offload 가능) |
| GPU 2장 환경 | **추천** — 오버헤드 낮음 | 가능하나 설정 부담 대비 이득 적음 |
| GPU 8장 이상 | 가능하나 ZeRO보다 느릴 수 있음 | **추천** — 대규모 샤딩에 최적화 |
| CPU/NVMe offload | 지원 약함 | ZeRO-Infinity로 지원 |
| 디버깅 용이성 | PyTorch 표준 — 익숙함 | 별도 로그 파싱 필요 |

### 시 파인튜닝 권장 선택

```
Qwen2.5-32B + A100 80GB × 2장 → FSDP
  └── 이유: GPU 2장 규모에서 DeepSpeed 설정 복잡도 대비 이득이 없음
  └── HuggingFace accelerate + FSDP_AUTO_WRAP_POLICY로 충분
  └── 추후 GPU 4장 이상으로 확장 시 DeepSpeed ZeRO-3 재검토

만약 A100 80GB × 4장 이상 확보 시 → DeepSpeed ZeRO-3
  └── ZeRO Stage 3: 파라미터 + 그래디언트 + 옵티마이저 상태 모두 샤딩
  └── GPU당 메모리 요구량 ~1/N로 감소
```

### FSDP 기본 설정 (accelerate config 기준)

```yaml
# accelerate_config.yaml
compute_environment: LOCAL_MACHINE
distributed_type: FSDP
fsdp_config:
  fsdp_auto_wrap_policy: TRANSFORMER_BASED_WRAP
  fsdp_backward_prefetch_policy: BACKWARD_PRE
  fsdp_cpu_ram_efficient_loading: true
  fsdp_offload_params: false          # A100 VRAM 충분하면 off
  fsdp_sharding_strategy: FULL_SHARD  # ZeRO-3 동등
  fsdp_state_dict_type: SHARDED_STATE_DICT
num_machines: 1
num_processes: 2                       # GPU 2장
```

---

## bf16 Mixed Precision + Gradient Checkpointing 설정

### bf16 Mixed Precision

A100은 bf16 하드웨어 가속을 지원한다. fp16 대비 장점:
- 동적 범위가 fp32와 동일 → loss spike 위험 감소
- 별도 loss scaling 불필요 (fp16은 GradScaler 필요)

```python
# HuggingFace TrainingArguments
from transformers import TrainingArguments

args = TrainingArguments(
    bf16=True,
    bf16_full_eval=True,
    tf32=True,              # A100 Tensor Core 최대 활용
    # fp16은 사용하지 않음 — bf16으로 통일
)
```

**주의**: bf16은 Ampere(A100, A6000) 이상에서만 하드웨어 가속.
V100 등 구형 GPU라면 fp16 + GradScaler로 전환 필요.

### Gradient Checkpointing

32B 모델의 activation 메모리는 파라미터 메모리보다 클 수 있다.
Gradient checkpointing은 순전파 중 activation을 버리고 역전파 시 재계산하여
메모리를 ~60~70% 절감하는 대신 학습 시간이 ~30% 증가한다.

```python
model.gradient_checkpointing_enable(
    gradient_checkpointing_kwargs={"use_reentrant": False}
    # use_reentrant=False: FSDP와 호환성 높음, 최신 PyTorch 권장
)
```

### 메모리 예산 (Qwen2.5-32B, A100 80GB × 2, FSDP Full Shard 기준)

| 구성 요소 | 메모리 (per GPU) |
|-----------|-----------------|
| 모델 파라미터 (bf16) | ~32 GB → FSDP 샤딩 후 ~16 GB |
| 그래디언트 (bf16) | ~16 GB → 샤딩 후 ~8 GB |
| 옵티마이저 상태 (AdamW fp32) | ~128 GB → 샤딩 후 ~64 GB |
| Activation (gradient checkpointing on) | ~4~8 GB (시퀀스 길이 의존) |
| **합계 (추정)** | **~90~96 GB → 위험 구간** |

> 옵티마이저 상태가 병목. 8-bit AdamW (bitsandbytes) 또는 Adafactor 사용 시
> 옵티마이저 메모리를 절반 이하로 줄일 수 있다.

```python
# 8-bit AdamW로 옵티마이저 메모리 절감
from bitsandbytes.optim import AdamW8bit

optimizer = AdamW8bit(model.parameters(), lr=1e-5, weight_decay=0.01)
```

---

## 실제 A100 학습 비용 추정 (Qwen2.5-32B 기준)

### 전제 조건

| 항목 | 값 |
|------|-----|
| 학습 토큰 수 | 50M~200M 토큰 (시 코퍼스 규모 의존) |
| A100 80GB 클라우드 단가 | $2.0~$3.5/시간·장 (Lambda Labs, RunPod 기준 2025) |
| GPU 수 | A100 × 2장 |
| 처리량 (추정) | ~1,000~1,500 토큰/초 (bf16 + gradient checkpointing, seq_len=4096) |

### 비용 계산

```
학습 시간 = 총 학습 토큰 ÷ 처리량

시나리오 A (소규모: 50M 토큰):
  학습 시간 = 50,000,000 ÷ 1,200 = ~11.6시간
  GPU 비용 = 11.6 × 2장 × $2.5 = ~$58

시나리오 B (중규모: 100M 토큰):
  학습 시간 = 100,000,000 ÷ 1,200 = ~23.1시간
  GPU 비용 = 23.1 × 2장 × $2.5 = ~$116

시나리오 C (커리큘럼 전체: 200M 토큰):
  학습 시간 = 200,000,000 ÷ 1,200 = ~46.3시간
  GPU 비용 = 46.3 × 2장 × $2.5 = ~$231
```

### 토큰 수 → 실제 데이터 규모 환산

```
한국어 시 1편 ≈ 100~500 토큰
창작노트+시 페어 ≈ 500~2,000 토큰

50M 토큰 ≈ 시 단독 100,000~500,000편
           또는 CoT 페어 25,000~100,000건

현실적 데이터셋 규모가 수천~수만 편 수준이라면
epoch를 2~5회 반복하거나 데이터 증강 필수
```

### 비용 최적화 전략

1. **스팟 인스턴스 활용**: RunPod Spot, Lambda Spot — 단가 50~70% 절감, 단 중단 위험
2. **파일럿 먼저**: SOLAR-10.7B로 파이프라인 검증 → 비용 ~1/10
3. **LoRA 파일럿 후 풀 파인튜닝**: 하이퍼파라미터 확정 후 본 학습

---

## Loss Spike 방지 전략

Loss spike는 학습 중 loss가 갑자기 치솟는 현상으로,
대형 모델 파인튜닝에서 흔히 발생한다. 주요 원인: 큰 그래디언트, 비정상 데이터 배치.

### Gradient Clipping

```python
TrainingArguments(
    max_grad_norm=1.0,   # 표준값; 불안정 시 0.5로 낮춤
)
```

그래디언트 노름이 `max_grad_norm`을 초과하면 스케일다운.
32B 풀 파인튜닝에서는 1.0이 기본값이나, 초기 학습이 불안정하면 0.3~0.5로 조정.

### Warmup 설정

```python
TrainingArguments(
    warmup_ratio=0.03,        # 총 스텝의 3% — 절대값보다 비율 권장
    # warmup_steps=100,       # 피할 것: 총 스텝 수 모르면 의미 없음
    lr_scheduler_type="cosine",
)
```

**왜 비율이 중요한가**: 총 학습 스텝이 500이면 warmup 100스텝 = 20% (과도).
총 스텝이 10,000이면 100스텝 = 1% (부족). `warmup_ratio=0.03`이 일반적으로 안전.

### Learning Rate 선택

```
풀 파인튜닝 (32B): 5e-6 ~ 2e-5
  └── 너무 크면 catastrophic forgetting 위험
  └── 너무 작으면 시 도메인 적응 부족
  └── 권장 시작점: 1e-5

커리큘럼 Stage별 LR:
  Stage 1~2 (언어 기반): 1e-5
  Stage 3~4 (CoT/창작): 5e-6 (더 섬세한 조정 필요)
  Stage 5 (수정/반복): 3e-6
```

### 데이터 품질 사전 검증

spike의 숨은 원인 중 하나는 비정상 샘플(빈 텍스트, 이상 긴 시퀀스):

```python
# 전처리 시 토큰 길이 분포 확인
from collections import Counter
lengths = [len(tokenizer(text)["input_ids"]) for text in dataset]
# 최대 길이 outlier 제거: max_seq_length의 2배 초과 샘플 제거
```

---

## 체크포인트 전략

### 저장 주기

```python
TrainingArguments(
    save_strategy="steps",
    save_steps=200,             # 총 스텝의 약 2~5%마다 저장
    save_total_limit=5,         # 최신 5개만 보관 (디스크 절약)
    load_best_model_at_end=True,
    metric_for_best_model="eval_loss",
    greater_is_better=False,
)
```

### 디스크 용량 계산 (Qwen2.5-32B)

| 저장 형식 | 체크포인트 1개 크기 |
|-----------|-------------------|
| FSDP Sharded (bf16) | ~64 GB (샤드 파일 합산) |
| HuggingFace SafeTensors (bf16) | ~64 GB |
| LoRA 어댑터만 | ~500 MB~1 GB |

```
체크포인트 5개 보관 기준:
  풀 파인튜닝: 64 GB × 5 = ~320 GB 필요
  → NVMe SSD 500 GB 이상 확보 권장

커리큘럼 학습 (Stage 5개 × 최종 체크포인트 1개):
  64 GB × 5 = ~320 GB 추가
  → 총 ~640 GB (Stage 중간 저장 포함)
```

### 체크포인트 보관 정책

```
학습 중: save_total_limit=5 (롤링 보관)
Stage 완료 시: 해당 Stage 최종 체크포인트는 별도 보관 (삭제 금지)
  └── 이유: 커리큘럼 Stage 간 비교 실험 및 롤백을 위해
최종 모델: SafeTensors 포맷으로 별도 변환 후 보관
```

---

## 학습 모니터링 지표

### 필수 지표 (학습이 잘 되고 있는지 판단)

| 지표 | 정상 범위 | 이상 신호 |
|------|-----------|-----------|
| **train/loss** | 초기 대비 꾸준히 감소 | spike 또는 발산 |
| **eval/loss** | train/loss와 유사한 추세 | train < eval 격차 벌어짐 = 과적합 |
| **grad_norm** | 대부분 1.0 이하 | 지속적으로 1.0 클리핑 = LR 너무 큼 |
| **learning_rate** | warmup 후 cosine 감소 | 변동 없으면 scheduler 오류 |
| **GPU 메모리 사용률** | 70~90% | OOM 직전 신호: 95% 이상 |
| **처리량 (tokens/sec)** | 안정적 유지 | 갑자기 감소 = 통신 병목 또는 OOM |

### 시 도메인 특화 지표

```
매 500~1000 스텝마다 다음 프롬프트로 샘플 생성 후 수동 확인:
  1. "봄 / 비 / 어머니를 소재로 시를 써라"
  2. "다음 소재로 시를 써라: 새벽 지하철 첫 차"

확인 항목:
  - 행갈이(<행갈이>) 토큰이 자연스러운 위치에 나타나는가?
  - 시가 완결되는가 (중간에 끊기지 않는가)?
  - 반복 루프에 빠지는가? (생성 품질 저하 초기 신호)
  - 언어가 시적 감각을 유지하는가? (산문화 여부 확인)
```

### 권장 모니터링 도구

```
WandB (Weights & Biases):
  - loss curve, grad_norm, LR 자동 시각화
  - 생성 샘플 텍스트 로깅 가능
  - 무료 플랜으로 개인 프로젝트 충분

TensorBoard (대안):
  - 별도 서버 설정 필요, 기능은 WandB보다 제한적

설정:
  TrainingArguments(report_to="wandb", run_name="okf-poetry-stage1")
```

### 조기 종료 기준

```
eval_loss가 N 평가 주기 연속 개선되지 않으면 중단:
  early_stopping_patience=5  (500 스텝 × 5 = 2500 스텝 개선 없으면 중단)

추가 기준:
  - 생성 샘플에서 반복 루프 3회 이상 발생 → 즉시 학습 중단 검토
  - grad_norm이 10 이상으로 지속 → LR 낮추고 재시작
```

---

## 미결 사항

- 실제 시 코퍼스 토큰 수 집계 후 비용 재추정 필요
- 클라우드 vs 로컬 GPU 서버 결정 (초기 비용 vs 운영 비용 트레이드오프)
- FSDP + 8-bit AdamW 조합의 실제 메모리 측정값 확인 필요
- WandB 프로젝트 네이밍 컨벤션 사전 정의
