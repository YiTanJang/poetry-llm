---
type: Design
title: 파인튜닝 전략
description: 전체 학습 파이프라인 — CPT(도메인 적응) → SFT(창작 노트+시 포맷) → DPO(3 페르소나 심사 기반 선호 정렬). 풀 파인튜닝 vs LoRA 결정 근거, 하이퍼파라미터 전략 포함.
tags: [model, fine-tuning, full-finetuning, lora, training, dpo, cpt, sft]
timestamp: 2026-06-27T00:00:00Z
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

창작 노트(CoT) → 시 초안 → 반복 수정 포맷을 학습한다. 특수 토큰이 완전히 통합되어야 한다.

### 핵심 하이퍼파라미터 (Qwen2.5-32B 풀 파인튜닝, A100 80GB × 2장 기준)

| 파라미터 | 초기값 | 비고 |
|----------|--------|------|
| Learning rate | 1e-5 | 풀 파인튜닝 기준 |
| LR schedule | Cosine with warmup | 코사인 감쇠 스케줄러 |
| Warmup ratio | 0.03 | 총 학습 스텝의 3% |
| Batch size (Effective) | 16 ~ 64 | Per-device: 1~2 / Gradient Accum: 8~16 |
| Token budget | 50M ~ 200M tokens | epoch 대신 총 토큰 수 기준 |
| Max seq length | 4096 | CoT + 시 초안 + 반복 수정 텍스트 포함 |
| Early stopping | patience = 3 | eval loss 기준 |

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
# eval_loss 기준, 3회 평가 주기 연속 개선 없으면 학습 중단
```

### 평가 체크포인트

매 N 스텝마다:
1. 행갈이/연갈이 위치 정확도
2. 소수의 시 프롬프트로 생성 샘플 수동 평가
3. Perplexity on held-out poetry set

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

### 미래 방향: PRM (Process Reward Model)

DPO가 최종 시 텍스트에 대한 선호를 정렬하는 데 반해, **PRM(Process Reward Model)**은 창작 과정(CoT, 반복 수정 각 단계)에 중간 보상을 부여한다.

> 가설: PRM은 단순 DPO보다 창작 과정 자체의 질을 높이는 데 효과적일 것이다. 그러나 시의 창작 과정에 대한 step-level 레이블링이 필요하며, 이는 인간 전문가 개입 비용이 높다. M3 이후 Phase 1에서 탐색한다.

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

1. **Stage 전환 시점의 옵티마이저 상태 제어**: Stage가 전환될 때 Adam optimizer의 모멘텀 및 배리언스 상태를 완전히 초기화(Warm Restart)할 것인가, 아니면 스케줄러의 연속성을 유지할 것인가?

2. **CoT 영역의 Loss Masking**: 창작 노트(CoT) 영역의 loss 가중치를 마스킹하거나 낮추면 시 본문 품질이 향상되는가? SFT에서 CoT를 학습 신호로 쓸 때와 마스킹할 때의 ablation 실험 필요.

3. **다국어 시 데이터의 어휘사전 확장**: 외국어 시 학습 시 토큰 파편화(token fragmentation)를 최소화하면서 시적 뉘앙스를 보존하는 vocabulary 최적화 방법.

4. **DPO β 값 탐색**: β가 너무 낮으면 리워드 해킹, 너무 높으면 SFT 결과물과 무차별. 시 도메인에서 적절한 β 범위는 파일럿에서 실측 필요.

5. **3 페르소나 간 점수 margin이 작을 때 처리 방침**: score_chosen - score_rejected < 0.2인 weak pair를 DPO 데이터에 포함할지 여부.

6. **PRM 탐색 시점**: M3 이후 Phase 1에서 step-level 보상 모델 실험을 시작할 때 인간 레이블링 비용과 효과 비교 연구.
