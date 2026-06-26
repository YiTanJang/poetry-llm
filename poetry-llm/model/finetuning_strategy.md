---
type: Design
title: 파인튜닝 전략
description: 풀 파인튜닝 vs LoRA 결정 근거, CPT/SFT 단계별 하이퍼파라미터, 커리큘럼 설계, 특수 토큰 초기화, 조기 종료 기준.
tags: [model, fine-tuning, full-finetuning, lora, training, cpt, sft]
timestamp: 2026-06-27T00:00:00Z
---

# 파인튜닝 전략

## 풀 파인튜닝 vs LoRA

사용자는 **풀 파인튜닝**을 계획하고 있다. 이에 대한 근거와 트레이드오프:

### 풀 파인튜닝의 장점
- 모든 파라미터가 업데이트 → 깊은 도메인 적응 가능
- 시의 언어 감각은 표면 레이어만이 아니라 깊은 표현 레이어까지 영향을 미침
- 특수 토큰(`<행갈이>`, `<연갈이>` 등) 완전 통합

### LoRA의 장점
- 컴퓨팅 비용 대폭 절감 (풀 파인튜닝의 1/10~1/20)
- 과적합 위험 감소
- 빠른 실험 사이클

### 권장 접근
```
초기: LoRA (r=64, alpha=128)로 파일럿 실험
  └── 데이터 파이프라인 검증
  └── 하이퍼파라미터 탐색
  └── 시 생성 품질 초기 평가

이후: 풀 파인튜닝으로 전환
  └── 파일럿에서 최적화된 설정으로
  └── 특수 토큰 완전 통합
```

---

## 학습 파이프라인: CPT → 커리큘럼 SFT

전체 파인튜닝은 두 단계로 구성된다. 상세 내용은 `continual_pretraining.md` 참조.

```
[베이스 모델: Qwen2.5-32B]
        ↓
[1단계: CPT — 시/문학 원시 텍스트 (100M~500M 토큰)]
        ↓
[CPT 체크포인트]
        ↓
[2단계: 커리큘럼 SFT — Stage 1~4 순차 학습]
        ↓
[최종 파인튜닝 모델]
```

---

## 1단계: CPT 하이퍼파라미터

### 환경 기준
- 모델: Qwen2.5-32B
- GPU: A100 80GB × 2장, FSDP Full Shard
- bf16 + gradient checkpointing

### 배치 크기 (seq_len=4096 기준)

| 항목 | 값 | 비고 |
|------|-----|------|
| Per-device batch size | 2~4 | A100 80GB VRAM 한계 |
| Gradient accumulation steps | 128~256 | effective batch size 확보 |
| Effective batch size | ~2M~4M 토큰/스텝 | 대규모 코퍼스 CPT 표준 |

> **주의**: per-device batch size 16~32는 seq_len=4096에서 A100 80GB 기준 OOM 발생.
> FSDP Full Shard 환경에서도 per-device 2~4가 안전한 상한선.

### 핵심 하이퍼파라미터

| 파라미터 | 권장값 | 근거 |
|----------|--------|------|
| Learning rate | 5e-6 ~ 1e-5 | 베이스 모델 보존 + 도메인 적응 균형 |
| LR schedule | Cosine with warmup | 안정적 감쇠 |
| Warmup ratio | 0.01~0.03 | 절대 스텝 대신 비율 — 데이터 규모 변화에 강건 |
| Max seq length | 4096 | 시 + 시론 함께 포함 가능 |
| Weight decay | 0.1 | 정규화 |
| Gradient clipping | 1.0 | loss spike 방지 |

### 토큰 예산 (CPT)

| 시나리오 | 코퍼스 규모 | 목표 학습 토큰 | 예상 학습 시간 (A100×2) |
|----------|------------|--------------|------------------------|
| 소규모 | 50M 토큰 | 50M~100M (1~2 epoch) | ~12~24시간 |
| 중규모 | 200M 토큰 | 200M~400M (1~2 epoch) | ~46~92시간 |
| 대규모 | 500M 토큰 | 500M (1 epoch) | ~115시간 |

epoch 대신 **총 학습 토큰 수**로 목표를 관리한다. 소규모 코퍼스(수십만 편)라면
2~3 epoch 반복하여 목표 토큰 예산을 채운다.

### CPT 종료 기준

```
다음 중 하나 충족 시 CPT 종료:
  1. 목표 학습 토큰 수 도달 (예: 200M 토큰)
  2. eval perplexity가 3 평가 주기 연속 개선 없음 (patience=3)
  3. 생성 샘플에서 시적 언어 감각 확인 (수동 평가)
```

---

## 2단계: 커리큘럼 SFT

### 커리큘럼 구조

데이터를 난이도/추상성 순서로 제시하면 학습 효율이 높아진다.

```
Stage 1 — 언어 기반 + 형식 학습
  └── 공공도메인 시, 외국어 시 번역본
  └── 행갈이/연갈이 토큰이 포함된 구조적 시 데이터

Stage 2 — 메타 지식
  └── 시론, 시 평론
  └── 시의 논리와 의도 학습

Stage 3 — CoT 창작
  └── 창작노트 → 시 페어 데이터
  └── 창작 과정의 추론 학습

Stage 4 — 수정/반복
  └── 수정 과정 데이터
  └── 자기 비평과 개선 학습
```

### SFT 배치 크기

| 항목 | 값 | 비고 |
|------|-----|------|
| Per-device batch size | 1~2 | seq_len=4096, A100 80GB |
| Gradient accumulation steps | 8~16 | |
| Effective batch size | 1~2 × 2(GPU) × 8~16 = **16~64** | instruction-output 페어 기준 |

### SFT 핵심 하이퍼파라미터

| 파라미터 | 권장값 | CPT와의 차이 |
|----------|--------|------------|
| Learning rate | 3e-6 ~ 8e-6 | CPT보다 낮게 — 세밀한 조정 |
| LR schedule | Cosine with warmup | 동일 |
| Warmup ratio | 0.03~0.05 | CPT보다 높게 — 데이터 수 적음 |
| Max seq length | 4096 | 창작 노트 + 시 충분히 포함 |

### Stage별 토큰 예산 (SFT)

| Stage | 데이터 종류 | 목표 토큰 수 | LR | LR 리셋 |
|-------|------------|------------|-----|---------|
| Stage 1 | 언어 기반 + 형식 | ~10M | 8e-6 | CPT 후 리셋 |
| Stage 2 | 메타 지식 (시론/평론) | ~5M | 5e-6 | Stage 1 후 리셋 권장 |
| Stage 3 | CoT 창작 노트 → 시 | ~10M | 5e-6 | Stage 2 후 리셋 권장 |
| Stage 4 | 수정/반복 | ~5M | 3e-6 | Stage 3 후 리셋 권장 |
| **SFT 합계** | | **~30M 토큰** | | |

> **가설:** Stage 간 LR 리셋(새로운 warmup 포함)이 연속 학습보다 안정적일 수 있다.
> 실험으로 검증 필요. 총 SFT 예산은 데이터 수집 결과에 따라 5M~50M 범위 조정.

### Stage별 eval 데이터 분리

각 Stage에서 전체 데이터의 5~10%를 validation set으로 고정 분리:
- Stage 1~2: 시 held-out set perplexity
- Stage 3~4: 생성 시 품질 수동 평가 (novelty 지표 포함)

---

## 조기 종료 기준

소규모 시 데이터셋(수천 편)에서 epoch 반복은 과적합 위험이 높다.
eval_loss 기준 조기 종료를 필수로 적용한다.

```python
from transformers import EarlyStoppingCallback

trainer = Trainer(
    # ...
    callbacks=[
        EarlyStoppingCallback(early_stopping_patience=3)
        # eval_loss 기준, 3 평가 주기 연속 개선 없으면 중단
    ],
    args=TrainingArguments(
        evaluation_strategy="steps",
        eval_steps=200,
        load_best_model_at_end=True,
        metric_for_best_model="eval_loss",
        greater_is_better=False,
    )
)
```

추가 종료 조건:
- 생성 샘플에서 반복 루프 3회 이상 발생 → 즉시 학습 중단 검토
- `grad_norm`이 10 이상으로 지속 → LR 낮추고 재시작

---

## 특수 토큰 처리

### 토큰 추가 및 임베딩 리사이즈

```python
# 새 특수 토큰 추가
special_tokens = ["<행갈이>", "<연갈이>", "<시작>", "<끝>", "<음절:N>"]
tokenizer.add_special_tokens({"additional_special_tokens": special_tokens})

# 임베딩 레이어 리사이즈 (필수 — 누락 시 IndexError 발생)
model.resize_token_embeddings(len(tokenizer))
```

### 의미 기반 임베딩 초기화

랜덤 초기화 대신, 각 특수 토큰의 기능에 의미적으로 가까운 기존 토큰의
임베딩으로 초기화하면 수렴이 빠르다.

```python
import torch

def init_special_token_embedding(model, tokenizer, new_token, ref_tokens):
    """
    new_token: 초기화할 특수 토큰 문자열 (예: "<행갈이>")
    ref_tokens: 의미적으로 가까운 기존 토큰 문자열 리스트
    """
    embed = model.get_input_embeddings()
    new_id = tokenizer.convert_tokens_to_ids(new_token)

    ref_ids = [
        tokenizer.convert_tokens_to_ids(t)
        for t in ref_tokens
        if tokenizer.convert_tokens_to_ids(t) != tokenizer.unk_token_id
    ]
    if not ref_ids:
        return  # 참조 토큰 없으면 랜덤 초기화 유지

    with torch.no_grad():
        ref_vecs = embed.weight.data[ref_ids]
        embed.weight.data[new_id] = ref_vecs.mean(dim=0)

# 적용 예시
init_special_token_embedding(model, tokenizer, "<행갈이>", ["\n", "▲"])
init_special_token_embedding(model, tokenizer, "<연갈이>", ["\n\n", "\n"])
init_special_token_embedding(model, tokenizer, "<시작>", ["[", "《", "<"])
init_special_token_embedding(model, tokenizer, "<끝>",   ["]", "》", ">"])
```

> **주의**: `resize_token_embeddings` 호출 후 새 토큰의 초기값은 랜덤이므로,
> 반드시 위 초기화를 학습 시작 전에 적용해야 한다.

---

## 학습 안정화

### 배치 크기 요약 (Qwen2.5-32B, A100 80GB × 2, seq_len=4096)

| 단계 | Per-device BS | Grad. Accum. | Effective BS |
|------|-------------|-------------|------------|
| CPT | 2~4 | 128~256 | ~2M~4M 토큰 |
| SFT | 1~2 | 8~16 | 16~64 샘플 |

### Warmup ratio vs 절대값

```python
TrainingArguments(
    warmup_ratio=0.03,        # 총 스텝의 3% — 권장
    # warmup_steps=100,       # 피할 것: 총 스텝 수 모르면 비율이 달라짐
    lr_scheduler_type="cosine",
    max_grad_norm=1.0,        # gradient clipping
)
```

`warmup_ratio=0.03`은 데이터셋 크기나 epoch 수가 바뀌어도 비율이 유지된다.
총 스텝이 500이면 warmup 100스텝은 20%(과도), 10,000이면 1%(부족).

---

## 평가 체크포인트

매 200 스텝(또는 eval_steps 설정)마다:
1. `eval_loss` 및 perplexity 기록
2. 소수의 고정 프롬프트로 생성 샘플 수동 확인
   - 행갈이/연갈이 토큰이 자연스러운 위치에 나타나는가?
   - 시가 완결되는가?
   - 반복 루프가 발생하는가?
3. novelty 지표 (n-gram 독창성) 자동 계산 (evaluation/ 참조)

---

## 미결 사항

- **Stage별 실제 토큰 예산 확정**: 데이터 수집 완료 후 `data/token_budget.md`
  집계 결과를 바탕으로 Stage 1~4 토큰 수 재조정 필요
- **Stage 간 LR 리셋 효과 검증**: 연속 학습 vs Stage별 재시작 중 어느 쪽이
  novelty 지표에 유리한지 파일럿(10.7B) 실험으로 확인 필요
- **다국어 데이터 배치**: 외국어 시 번역본을 CPT에 포함할지, SFT Stage 1에 포함할지 미결
- **SFT eval 데이터 분리 방법**: Stage별로 독립 validation set을 구성할지,
  단일 held-out set을 공유할지 결정 필요
- **<음절:N> 토큰 초기화 전략**: 숫자 N이 가변이므로 고정 토큰 외 동적 처리 방식 검토
- **CPT ↔ SFT 전환 시 옵티마이저 상태 리셋**: AdamW의 모멘텀 통계를 리셋하고
  새로운 warmup을 적용하는 것이 일반적이나, 프로젝트 맥락에서 실험 필요
