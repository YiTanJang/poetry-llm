---
type: Design
title: 파인튜닝 전략
description: 풀 파인튜닝 vs LoRA 결정 근거, 학습 하이퍼파라미터 전략, 커리큘럼 설계.
tags: [model, fine-tuning, full-finetuning, lora, training]
timestamp: 2026-06-26T00:00:00Z
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

## 커리큘럼 학습 (Curriculum Learning)

데이터를 난이도/추상성 순서로 제시하면 학습 효율이 높아진다.

### 제안 커리큘럼

```
Stage 1 — 언어 기반 (공공도메인 시, 외국어 시)
  └── 모델이 시의 기본 언어 패턴 습득

Stage 2 — 형식 학습 (행갈이/연갈이 토큰 데이터)
  └── 시의 구조적 결정 학습

Stage 3 — 메타 지식 (시론, 시평론)
  └── 시의 논리와 의도 학습

Stage 4 — CoT 창작 (창작노트→시 데이터)
  └── 창작 과정의 추론 학습

Stage 5 — 수정/반복 (수정 과정 데이터)
  └── 자기 비평과 개선 학습
```

## 핵심 하이퍼파라미터 (초안)

| 파라미터 | 초기값 | 비고 |
|----------|--------|------|
| Learning rate | 1e-5 | 풀 파인튜닝 기준 |
| LR schedule | Cosine with warmup | |
| Warmup steps | 100 | |
| Batch size | 16~32 (gradient accumulation 활용) | |
| Epochs | 2~3 | 과적합 모니터링 필요 |
| Max seq length | 4096 | 창작 노트 + 시 |

## 특수 토큰 처리

풀 파인튜닝 시 특수 토큰 임베딩 초기화 전략:

```python
# 새 특수 토큰 추가
special_tokens = ["<행갈이>", "<연갈이>", "<시작>", "<끝>", "<음절:N>"]
tokenizer.add_special_tokens({"additional_special_tokens": special_tokens})
model.resize_token_embeddings(len(tokenizer))

# 새 토큰 임베딩 초기화: 의미적으로 가까운 토큰의 임베딩 평균으로 초기화
# (랜덤 초기화보다 빠른 수렴)
```

## 평가 체크포인트

매 N 스텝마다 다음을 평가:
1. 행갈이/연갈이 위치 정확도
2. 소수의 시 프롬프트로 생성 샘플 수동 평가
3. Perplexity on held-out poetry set

## 미결 사항

- Stage별 학습 스텝 수 결정
- 다국어 데이터(외국어 시)를 전체 커리큘럼에서 어느 단계에 배치할 것인가
- 조기 종료(Early stopping) 기준 정의
- 체크포인트 저장 전략 (디스크 공간 vs 실험 추적)

---

## ⚠️ 발견된 문제점

### 1. Batch size 표기 모호 — 32B 모델에 비현실적

> 현재: `Batch size 16~32 (gradient accumulation 활용)`

**문제**: A100 80GB × 2장 환경에서 Qwen2.5-32B 풀 파인튜닝 시
per-device batch size 16~32는 **VRAM 초과로 OOM 발생**.

seq_len=4096 기준 실제 현실적 값:
- per-device batch size: **1~2**
- gradient accumulation steps: **8~16**
- effective batch size: 1~2 × 2(GPU) × 8~16(accumulation) = **16~64**

수정 제안: "per-device batch size 1~2, gradient accumulation 8~16"으로 명확히 분리 기재.

---

### 2. Warmup steps 100 — 총 스텝 수 없이 의미 없음

> 현재: `Warmup steps: 100`

**문제**: 총 학습 스텝 수가 명시되지 않아 warmup 비율 계산 불가.
- 총 스텝 500이면: warmup 100 = **20%** (과도 — 학습 초기 비효율)
- 총 스텝 10,000이면: warmup 100 = **1%** (너무 짧아 spike 위험)

수정 제안: `warmup_ratio=0.03` (총 스텝의 3%)으로 변경.
절대값 대신 비율로 관리해야 데이터셋 크기 변화에 강건하다.

---

### 3. Epochs 2~3 — 시 데이터 규모에 따라 의미가 완전히 달라짐

> 현재: `Epochs: 2~3`

**문제**: epoch 수는 데이터셋 규모에 따라 총 학습 토큰 수가 크게 달라진다.
- 시 1,000편(~50만 토큰) × 3 epoch = **150만 토큰** (과소학습 가능)
- 시 100,000편(~5천만 토큰) × 3 epoch = **1억 5천만 토큰** (충분)

수정 제안: epoch 대신 **총 학습 토큰 수 (token budget)** 기준으로 목표 설정.
일반적으로 32B 풀 파인튜닝 도메인 적응에는 50M~200M 토큰이 권장.

---

### 4. 커리큘럼 Stage별 스텝 수 미결 — 비용과 직결되는 핵심 누락

> 현재: 미결 사항으로만 언급

**문제**: Stage 5개 각각의 학습 스텝(또는 토큰) 수가 없으면
총 학습 비용 추정 자체가 불가능하다. 설계 단계에서 최소한의 초기값이라도 필요.

추가 필요 항목:
- 각 Stage별 목표 토큰 수 (예: Stage 1 = 50M, Stage 4 = 10M)
- Stage 간 LR 리셋 여부
- 각 Stage에서 eval 데이터셋을 어떻게 분리할 것인가

---

### 5. 조기 종료 기준 미정 — 과적합 위험 미대응

> 현재: 미결 사항으로만 언급

**문제**: 소규모 시 데이터셋(수천 편)에서 2~3 epoch 반복은
과적합(eval_loss 증가) 위험이 높다. 조기 종료 기준 없이 학습하면
비용 낭비 및 모델 품질 저하 가능.

수정 제안:
```python
EarlyStoppingCallback(early_stopping_patience=3)
# eval_loss 기준, 3 평가 주기 연속 개선 없으면 중단
```

---

### 6. 특수 토큰 임베딩 초기화 전략 불완전

> 현재: "의미적으로 가까운 토큰의 임베딩 평균으로 초기화" (코드 주석)

**문제**: 어떤 토큰이 `<행갈이>`와 의미적으로 가까운지 선택 기준이 없음.
`<연갈이>`는 문단 구분 역할이므로 `\n\n`에 대응하는 토큰 임베딩 평균으로
초기화하는 것이 합리적이나, 이를 명시적으로 코드에 구현해야 한다.

```python
# 구체화 예시
newline_token_id = tokenizer.convert_tokens_to_ids("\n")
model.get_input_embeddings().weight.data[new_token_id] = \
    model.get_input_embeddings().weight.data[newline_token_id].clone()
```
