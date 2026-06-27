---
type: Design
title: 파인튜닝 전략
description: 전체 학습 파이프라인 — CPT(도메인 적응) → SFT(창작 노트+시 포맷) → DPO(3 페르소나 심사 기반 선호 정렬). 풀 파인튜닝 vs LoRA 결정 근거, 하이퍼파라미터 전략 포함.
tags: [model, fine-tuning, full-finetuning, lora, training, dpo, cpt, sft]
timestamp: 2026-06-28T00:00:00Z
---

# 파인튜닝 전략

## 전체 학습 파이프라인

```
CPT (Continued Pre-Training)
  └── 도메인 언어 적응 — 시 코퍼스, 시론, 평론
        ↓
SFT (Supervised Fine-Tuning)
  └── 창작 노트(CoT) → 시 포맷 학습, 특수 토큰 통합
        ↓
DPO (Direct Preference Optimization)
  └── 3 페르소나 초안 → 외부 심사 → (winner, loser) 선호 정렬
```

각 단계는 순차적이다. CPT 없이 SFT를 하면 도메인 언어 적응이 부족하고, SFT 없이 DPO를 하면 기본 포맷 자체가 불안정하다.

---

## 풀 파인튜닝 vs LoRA

사용자는 **풀 파인튜닝**을 계획하고 있다.

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

## 1단계: CPT (Continued Pre-Training) — 도메인 적응

### 목적

베이스 모델(Qwen2.5-32B)이 한국 현대시 언어에 적응하도록 도메인 특화 사전학습을 진행한다.
CPT는 SFT의 선행 조건이다. 도메인 언어 없이 포맷을 학습하면 생성물의 어휘·리듬이 일반 언어 모델 수준에 머문다.

### CPT 커리큘럼 및 토큰 버젯 (총 120M 토큰)

```
Stage 1 — 언어 기반 (공공도메인 시, 외국어 시): 50M tokens
  └── 시의 기본 언어 패턴 및 풍부한 표현력 습득

Stage 2 — 형식 학습 (행갈이/연갈이 토큰 데이터): 30M tokens
  └── 구조적 결정 및 행/연 단위 제어 패턴 학습

Stage 3 — 메타 지식 (시론, 시평론): 20M tokens
  └── 시의 논리와 미학적 의도 습득

Stage 4 — CoT 창작 (창작노트 → 시 데이터): 10M tokens
  └── 창작 과정의 추론(Chain-of-Thought)과 창작 의도 반영 학습

Stage 5 — 수정/반복 (수정 과정 데이터): 10M tokens
  └── 시적 자기 비평과 다듬기(Refinement) 모델 구축
```

### CPT 커리큘럼 이행 세부 설계

- **Stage 간 LR 제어**: 각 Stage 진입 시 Warm Restart를 수행하거나 학습률 스케줄러를 점진적으로 감쇠시킨다 (catastrophic forgetting 방지).
- **평가 데이터 분리**: 각 Stage 데이터의 5~10%를 Held-out Evaluation Set으로 사전 분리하여 과적합 모니터링.

---

## 2단계: SFT (Supervised Fine-Tuning) — 포맷 학습

### 목적

창작 노트(CoT) → 시 초안 → 반복 수정 포맷을 학습한다. 특수 토큰이 완전히 통합되어야 한다. CPT 단계에서 확보한 한국 현대시의 언어 감각을 바탕으로, 구조화된 창작 지시와 미학적 피드백을 수용하는 정교한 생성 능력을 배양한다.

### 핵심 하이퍼파라미터 (Qwen2.5-32B 풀 파인튜닝, A100 80GB × 2장 기준)

> 탐색중 — 각 파라미터 범위는 파일럿 실험 후 좁혀질 수 있음

| 파라미터 | 초기값 | 비고 |
|----------|--------|------|
| Learning rate | 1e-5 ~ 3e-6 | 커리큘럼 단계(Stage 1~4)별로 점진적 감쇠 및 재시작 |
| LR schedule | Cosine with warmup | 각 Stage 진입 시 Warm restart 적용 |
| Warmup ratio | 0.05 | 총 학습 스텝의 5% (Sharp transition 대응) |
| Batch size (Effective) | 16 ~ 64 | Per-device: 1~2 / Gradient Accum: 8~16 |
| Token budget | 50M ~ 200M tokens | 전체 4개 Stage 총합 토큰 기준 |
| Max seq length | 4096 | CoT + 시 초안 + 비평 및 반복 수정 텍스트 전체 포함 |
| Early stopping | patience = 3 | 각 Stage별 eval loss 기준 조기 종료 |

### 커리큘럼 단계별 학습률 감쇠 및 체크포인트 저장 전략 (LR Decay & Checkpoint Saving Strategy)

#### 1. 학습률 감쇠 프로필 (Learning Rate Decay Profiles)
- **코사인 어닐링 및 웜 리스타트 (Cosine Annealing with Warm Restarts)**:
  - 각 커리큘럼 Stage가 시작될 때마다 학습률은 해당 Stage의 피크 학습률(Stage 1: $1 \times 10^{-5}$, Stage 2: $8 \times 10^{-6}$, Stage 3: $5 \times 10^{-6}$, Stage 4: $3 \times 10^{-6}$)을 향해 선형적으로 상승하는 웜업(Warmup Ratio 5%) 구간을 거친 후, 코사인 함수 곡선을 따라 감쇠한다.
  - 각 Stage의 최종 시점에는 최소 학습률($1 \times 10^{-6}$)에 도달하여 가중치 업데이트가 안정화되도록 설계한다.
  - 이 웜 리스타트 전략을 통해 이전 Stage에서 수렴되었던 매개변수가 급격히 변동하지 않도록 방어하는 동시에, 새로운 Stage의 특화 데이터 분포에 대해 매끄러운 최적화 경로를 재탐색할 수 있게 유도한다.

#### 2. 옵티마이저 상태 리셋 (Optimizer Resets)
- **리셋 방식**: Curriculum Stage가 전환되는 경계면에서 AdamW 옵티마이저의 모멘텀(Momentum) 및 배리언스(Variance) 상태 값을 완전히 초기화(Reset)한다.
- **도입 근거**: 각 커리큘럼 Stage의 데이터 분포와 학습 목표(기본 형식 $\rightarrow$ 시론 정렬 $\rightarrow$ CoT 추론 $\rightarrow$ 자기 비평 수정)는 성격이 크게 다르다. 이전 Stage에서 축적된 모멘텀 정보가 신규 Stage 초기에 유지될 경우, 잘못된 업데이트 방향으로 관성이 작용하여 수렴 지연 및 파라미터 왜곡이 유발될 수 있다. 따라서 Stage 전환 시에는 가중치만 승계하고 옵티마이저는 새로 인스턴스화한다.

#### 3. 체크포인트 저장 주기 및 선택 기준 (Checkpoint Saving Frequencies & Criteria)
- **구조 중심 단계 (Stage 1 & Stage 2) 저장 전략**:
  - **저장 빈도**: 형식 학습 및 메타 매핑 단계에서는 형식 규칙의 빠른 수득과 과적합 제어가 핵심이므로, 비교적 잦은 주기(예: 매 200~400 steps 단위 또는 0.2 Epoch 마다)로 세밀하게 중간 체크포인트를 저장한다.
  - **저장/선택 기준 (Structural Metric)**: 주로 검증 손실(Validation Loss) 및 구조적 평가지표(특수 토큰 `<행갈이>`, `<연갈이>`의 규칙적 위치 예측 F1-Score)에 기반한다. 형식 왜곡이나 파괴가 발생하기 직전, 형식 규칙 준수율이 극되화되는 시점의 체크포인트를 최종적으로 보존한다.
- **미학/인지 중심 단계 (Stage 3 & Stage 4) 저장 전략**:
  - **저장 빈도**: 고차원적 시상 전개와 비평 교정을 다루는 단계로 손실 함수의 수렴이 완만하므로, 비교적 긴 주기(예: 매 500~1000 steps 단위 또는 0.5 Epoch 마다)로 저장 빈도를 완화한다.
  - **저장/선택 기준 (Aesthetic Reward Score)**: 단순히 텍스트 예측의 validation loss에만 의존하지 않고, **미학적 리워드 모델(Aesthetic Reward Model)**의 평가 점수와 창작 노트(CoT)-최종 시 간의 미학적 일관성 점수를 모니터링한다. Validation loss가 낮아지더라도 미학적 novelty나 시의 음악성이 감소하는 체크포인트는 배제하고, 리워드 모델 기준의 미학적 완성도 점수가 정점을 찍는 최적의 밸런스 체크포인트를 선별하여 보존한다.


### SFT 커리큘럼 학습 (Stage 1~4)

SFT 단계는 단순한 형태의 시 완성에서 출발하여 고차원적 비평 및 수정 능력을 갖추기까지 4단계의 커리큘럼으로 나누어 학습을 순차적으로 전개한다.

#### SFT Stage 1: 기본 형식 및 구조 학습 (Format 1 & Format 6)
- **목적**: 특수 토큰(`<행갈이>`, `<연갈이>`, `<시작>`, `<끝>`)의 올바른 경계 배치 규칙 및 운율/발음 태그의 연계를 기본 체화한다.
- **데이터 구성**: `training_data_formats.md`의 [포맷 1 (완성시)](../preprocessing/training_data_formats.md#포맷-1---완성시-기본) 40% + [포맷 6 (빈칸 채우기)](../preprocessing/training_data_formats.md#포맷-6---빈칸-채우기-구조-학습) 60%.
- **하이퍼파라미터**: Learning Rate $1 \times 10^{-5}$, Effective Batch Size 32, Epoch 2.
- **Loss Masking**: 입력과 출력 구분 없이 전체 시 텍스트 및 빈칸 정답 영역에 대해 Loss를 100% 계산한다.
- **전환/종료 조건**: Validation Set에서의 특수 토큰 예측 F1-Score > 0.95 및 Held-out Perplexity < 5.0 만족 시 다음 Stage로 진입.

#### SFT Stage 2: 메타 지식 및 시론 정렬 (Format 2 & Format 5)
- **목적**: 문학적 비평, 수사 기법 등의 추상적 메타 서사와 시 본문 사이의 미학적 정렬(Alignment)을 구축한다.
- **데이터 구성**: [포맷 2 (시 + 시론 페어)](../preprocessing/training_data_formats.md#포맷-2---시--시론-페어) 60% + [포맷 5 (시 $\rightarrow$ 설명 역방향)](../preprocessing/training_data_formats.md#포맷-5---시--설명-역방향) 40%.
- **하이퍼파라미터**: Learning Rate $8 \times 10^{-6}$, Effective Batch Size 32, Epoch 3.
- **Loss Masking**:
  - 포맷 2: 시(Poem) 입력부에 대한 Loss 계산을 제외하고, 시론(Analysis) 설명부에만 Loss를 집중 계산한다.
  - 포맷 5: 완성시 입력 영역은 `-100`으로 마스킹 처리하고, 역방향 해설(Commentary) 영역만 Loss를 계산한다.
- **전환/종료 조건**: Validation Set의 평론 Perplexity 수렴 및 정성 샘플링 평가(5점 척도 기준 분석 타당성 평균 3.5점 이상) 달성 시 진입.

#### SFT Stage 3: CoT 창작 프로세스 이식 (Format 3 & Format 7)
- **목적**: 창작 아이디어 구상(창작 노트)을 기반으로 일관성 있는 시를 창작하는 Chain-of-Thought 경로를 모델 내부에 이식한다.
- **데이터 구성**: [포맷 3 (창작 노트 $\rightarrow$ 시)](../preprocessing/training_data_formats.md#포맷-3---창작-노트--시-cot-파이프라인의-핵심) 70% + [포맷 7 (소재 $\rightarrow$ 시)](../preprocessing/training_data_formats.md#포맷-7---소재--시-창작-요청) 30%.
- **하이퍼파라미터**: Learning Rate $5 \times 10^{-6}$ (세밀한 가중치 조정), Effective Batch Size 16, Epoch 3.
- **Loss Masking**:
  - 입력 프롬프트(주제/소재)는 완전히 마스킹한다.
  - 창작 노트(CoT) 영역과 최종 시(Poem) 영역 모두 학습 신호로 사용하되, 의도 설계에 집중하기 위해 창작 노트의 Loss 가중치는 0.5, 최종 시 본문의 Loss 가중치는 1.0으로 스케일링하여 적용한다.
- **전환/종료 조건**: 창작 노트에 제시된 핵심 미학 지침(예: 이미지 종류, 행갈이 특징)과 생성된 시 본문 간의 일치도 검증 스코어 > 90% 달성 시 진입.

#### SFT Stage 4: 자기 비평 및 반복 수정 (Format 4)
- **목적**: 초고의 단점(정서적 직설, 상투적 비유)을 피드백(Critique)에 기반하여 정교한 언어로 재고하는 교정 루프를 완성한다.
- **데이터 구성**: [포맷 4 (수정 과정 데이터)](../preprocessing/training_data_formats.md#포맷-4---수정-과정-데이터) 100%.
- **하이퍼파라미터**: Learning Rate $3 \times 10^{-6}$ (극도로 미세한 잔차 업데이트), Effective Batch Size 16, Epoch 3.
- **Loss Masking**: 초안(Draft) 및 비평(Critique) 영역은 손실 계산에서 배제하고, 최종 수정본(Revised Poem) 영역에만 Loss를 100% 적용한다.
- **전환/종료 조건**: 외부 심사 LLM 평가를 통해 비평 의견의 개선 요구사항 수용률 > 85% 이상 검증 시 SFT 최종 완료 및 DPO 단계 이행.

### 조기 종료 (Early Stopping)

```python
from transformers import EarlyStoppingCallback

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
    callbacks=[EarlyStoppingCallback(early_stopping_patience=3)]
)
# 각 SFT 커리큘럼 Stage별로 eval_loss 기준 3회 평가 주기 연속 개선 없으면 자동 학습 중단 후 다음 Stage로 이행
```

### 평가 체크포인트

매 N 스텝마다 다음 지표를 모니터링한다:
1. 행갈이/연갈이 위치 정확도 (F1 Score)
2. 소수의 시 프롬프트로 생성 샘플 수동/자동 미학 평가
3. 각 Stage별 Held-out poetry evaluation set의 Perplexity

---

## 3단계: DPO (Direct Preference Optimization) — 선호 정렬

### 목적

SFT로 학습한 포맷 위에 미학적 선호도를 정렬한다. "어떤 시가 더 나은가"라는 인간 판단을 학습 신호로 내재화한다.

### 선호 쌍 구성 방법 (3 페르소나 방식)

```
1. 동일한 프롬프트(창작 노트 시드)로 3개의 시인 페르소나가 각각 초안 생성:
   - 페르소나 A: 이미지 중심 (황지우 계열 — 낯선 이미지 연결, 시각적 충격)
   - 페르소나 B: 리듬·음악성 중심 (김혜순 계열 — 반복, 음운, 몸의 언어)
   - 페르소나 C: 관념·해체 중심 (이상 계열 — 논리 해체, 개념 충돌)

2. 외부 심사 모델(GPT-4o 또는 Claude Opus)이 3개 초안을 비교:
   - 6개 미학 기준(경제성, 필연성, 긴장, 낯설게하기, 열린 끝, 음악성)으로 평가
   - 또는 학습된 리워드 모델로 대체 (M1 이후)

3. 최고 점수 초안 → chosen (winner)
   최저 점수 초안 → rejected (loser)

4. (chosen, rejected) 쌍을 DPO 학습 데이터로 저장
```

### DPO 데이터 포맷

```json
{
  "prompt": "창작 노트 시드 텍스트",
  "chosen": "선택된 시 (winner)",
  "rejected": "거부된 시 (loser)",
  "score_chosen": 0.82,
  "score_rejected": 0.41,
  "persona_chosen": "image",
  "persona_rejected": "concept",
  "judge_model": "gpt-4o",
  "timestamp": "2026-06-27T00:00:00Z"
}
```

저장 경로: `data/dpo_pairs/`

### DPO 하이퍼파라미터

> 탐색중 — β 값 및 학습률은 Phase 2 실험으로 결정됨

| 파라미터 | 초기값 | 비고 |
|----------|--------|------|
| β (KL penalty) | 0.1 | 낮으면 공격적 정렬, 높으면 SFT 근방 유지 |
| Learning rate | 5e-7 | SFT보다 훨씬 낮게 |
| Batch size | 8~16 | 쌍(pair) 기준 |
| Max length | 4096 | prompt + chosen/rejected 합산 |
| Epochs | 1~3 | 과적합 주의 — 짧게 |

### DPO 후 평가 및 피드백

DPO 이후 생성 샘플을 평가 파이프라인([evaluation_pipeline.md](evaluation_pipeline.md))에 넣어 리워드 모델 점수와 인간 평가 점수 변화를 추적한다.

점수가 개선되지 않으면:
- 선호 쌍의 margin이 너무 작은지 확인 (score_chosen - score_rejected < 0.2이면 weak pair)
- 심사 모델 변경 또는 리워드 모델 재보정 검토

### SFT 단계 일반 데이터 믹스 (Catastrophic Forgetting 2차 방어)

CPT Replay Buffer가 1차 방어라면, SFT 단계에서 일반 instruction 데이터를 혼합하는 것이 2차 방어다. CPT와 달리 SFT는 instruction-output 쌍 형태의 데이터를 수용할 수 있다.

```
SFT 데이터 구성 (총량 기준)
─────────────────────────────────
시 특화 데이터 (포맷 3,4,7 등)    80~90%
일반 instruction replay             10~20%
─────────────────────────────────
```

일반 instruction replay 후보 데이터셋:

| 데이터셋 | 출처 | 특징 |
|----------|------|------|
| **Tulu 3 SFT Mixture** | AI2 / HuggingFace | 다양한 소스 큐레이션, 라이선스 명확 |
| **OpenHermes-2.5** | HuggingFace | 합성 instruction 데이터, 품질 높음 |
| **KoAlpaca** | GitHub | 한국어 instruction following, 규모 소형 |

> 주의: CPT Replay Buffer는 **원시 텍스트만** 허용. instruction-output 쌍은 반드시 SFT 단계에서 혼합할 것.

---

### 미래 방향: PRM (Process Reward Model)

DPO가 최종 시 텍스트에 대한 선호를 정렬하는 데 반해, **PRM(Process Reward Model)**은 창작 과정(CoT, 반복 수정 각 단계)에 중간 보상을 부여한다.

> 가설: PRM은 단순 DPO보다 창작 과정 자체의 질을 높이는 데 효과적일 것이다. 그러나 시의 창작 과정에 대한 step-level 레이블링이 필요하며, 이는 인간 전문가 개입 비용이 높다. M3 이후 Phase 1에서 탐색한다.

### 미래 방향: On-Policy Self-Distillation

현재 DPO는 외부 judge(GPT-4o/Claude)에 의존하는 **off-policy** 방식에 가깝다. On-policy self-distillation은 모델 자신이 데이터 생성과 판정에 모두 참여하는 방식이다.

| 방법 | 핵심 아이디어 | 우리 적용 시 고려점 |
|------|-------------|-------------------|
| **SPIN** (Self-Play FT) | 현재 모델 출력 → rejected, 레퍼런스 → chosen | SFT 데이터가 레퍼런스 역할 |
| **Self-Rewarding LM** | 모델이 스스로 judge 역할 | 외부 judge 미학 편향 문제 완화 가능 |
| **On-Policy DPO** | 현재 학습 중인 모델로 실시간 쌍 생성 | 학습 루프 복잡도 증가 |

> 가설: 시 도메인에서 외부 judge(GPT-4o)의 미학 편향 문제를 줄이는 방향으로 Self-Rewarding 방식이 유효할 수 있다. 단, 자기 선호 편향(self-preference bias) 관리가 선행 과제. Phase 2 이후 탐색.

---

## 특수 토큰 처리 및 임베딩 초기화

풀 파인튜닝 시 추가되는 특수 토큰(`<행갈이>`, `<연갈이>` 등)의 임베딩이 랜덤하면 학습 초기 수렴이 느려진다. 의미적으로 유사한 기존 토큰 임베딩으로 초기화한다.

```python
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM

def initialize_special_token_embeddings():
    model_id = "Qwen/Qwen2.5-32B"
    tokenizer = AutoTokenizer.from_pretrained(model_id)
    model = AutoModelForCausalLM.from_pretrained(
        model_id,
        torch_dtype=torch.bfloat16,
        device_map="auto"
    )

    special_tokens = ["<행갈이>", "<연갈이>", "<시작>", "<끝>"]
    tokenizer.add_special_tokens({"additional_special_tokens": special_tokens})
    model.resize_token_embeddings(len(tokenizer))

    token_mapping = {
        "<행갈이>": ["\n"],
        "<연갈이>": ["\n\n", "\n"],
        "<시작>": ["<|im_start|>", "<s>"],
        "<끝>": ["<|im_end|>", "</s>"]
    }

    embeddings = model.get_input_embeddings()
    embedding_weight = embeddings.weight.data

    for new_token, source_tokens in token_mapping.items():
        new_id = tokenizer.convert_tokens_to_ids(new_token)
        if new_id == tokenizer.unk_token_id:
            print(f"Warning: {new_token} is not registered in tokenizer.")
            continue

        source_ids = []
        for src in source_tokens:
            src_id = tokenizer.convert_tokens_to_ids(src)
            if src_id != tokenizer.unk_token_id:
                source_ids.append(src_id)

        if not source_ids:
            source_ids = [tokenizer.convert_tokens_to_ids("\n")]

        with torch.no_grad():
            source_embeddings = embedding_weight[source_ids]
            mean_embedding = source_embeddings.mean(dim=0).clone()
            embedding_weight[new_id] = mean_embedding

        print(f"Success: Initialized {new_token} (ID: {new_id}) with mean embedding of {source_tokens}")

if __name__ == "__main__":
    initialize_special_token_embeddings()
```

---

## 미결 사항

- [Ph2] **DPO β 값 및 SFT 체크포인트 선정 최적화**: DPO 학습을 개시할 SFT 체크포인트를 어느 커리큘럼 단계(Stage 3 또는 Stage 4)의 어느 에포크 시점으로 고정하는 것이 DPO 선호 정렬 학습의 수렴 속도 및 최종 생성 품질(Novelty) 측면에서 가장 유리한가?
- [Ph1] **CoT Loss Masking 비율에 따른 시상 일치율-창작자유도 트레이드오프**: SFT Stage 3에서 CoT(창작 노트) 영역의 Loss 가중치(0.0 vs 0.5 vs 1.0)를 다르게 할 때, 학습된 모델의 시상 일치율(창작 노트의 의도를 따르는 정도)과 시 창작의 독창성(Novelty) 및 자유도 간의 정량적 트레이드오프는 어떻게 나타나는가?
- [Ph1] **SFT 단계 일반 instruction 데이터 혼합 비율**: 10% vs 20% 혼합 시 일반 언어 능력 보존 효과와 시 생성 품질 저하 간 트레이드오프. Tulu 3 vs KoAlpaca 중 한국어 instruction following 보존에 더 효과적인 소스는?
- [Ph2] **On-Policy Self-Distillation 도입 가능성**: SPIN 또는 Self-Rewarding LM 방식을 시 도메인에 적용 시, 외부 judge 의존 DPO 대비 생성 novelty 및 미학적 선호도 점수 변화. 자기 선호 편향 관리 방안 포함.
- [TODO] **다국어 시 데이터의 어휘사전 확장과 토큰 파편화 방지**: 외국어 시 번역 및 학습 시, 한국어와 외국어 토큰 파편화(fragmentation)를 최소화하면서 시적 운율과 뉘앙스를 보존할 수 있는 최적의 Vocabulary 크기 및 추가 토큰 확장 기법은 무엇인가?
