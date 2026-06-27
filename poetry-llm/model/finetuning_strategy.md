---
type: Design
title: 파인튜닝 전략
description: 풀 파인튜닝 vs LoRA 결정 근거, 학습 하이퍼파라미터 전략, 커리큘럼 설계.
tags: [model, fine-tuning, full-finetuning, lora, training]
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

## 커리큘럼 학습 (Curriculum Learning)

데이터를 난이도/추상성 순서로 제시하면 학습 효율이 높아진다.

### 제안 커리큘럼 및 토큰 버젯 (Token Budget)

학습의 단계별 목표 토큰 수와 세부 전략은 다음과 같이 구성한다 (총 120M 토큰 규모).

```
Stage 1 — 언어 기반 (공공도메인 시, 외국어 시): 50M tokens
  └── 모델이 시의 기본 언어 패턴 및 풍부한 표현력 습득

Stage 2 — 형식 학습 (행갈이/연갈이 토큰 데이터): 30M tokens
  └── 시의 구조적 결정 및 행/연 단위 제어 패턴 학습

Stage 3 — 메타 지식 (시론, 시평론): 20M tokens
  └── 시의 논리와 미학적 의도 습득

Stage 4 — CoT 창작 (창작노트 → 시 데이터): 10M tokens
  └── 창작 과정의 추론(Chain-of-Thought)과 창작 의도 반영 학습

Stage 5 — 수정/반복 (수정 과정 데이터): 10M tokens
  └── 시적 자기 비평과 다듬기(Refinement) 모델 구축
```

#### 커리큘럼 이행 세부 설계
- **Stage 간 LR (Learning Rate) 제어**: 급격한 데이터 특성 변화로 인한 모델 파괴적 망각(catastrophic forgetting) 또는 불안정성을 예방하기 위해, 각 Stage 진입 시 Warm Restart를 수행하거나 학습률 스케줄러를 점진적으로 감쇠시킨다.
- **평가 데이터(Eval Dataset) 분리**: 각 Stage에 사용되는 데이터셋 중 5%~10%를 독립적인 검증셋(Held-out Evaluation Set)으로 사전 분리하여, 각 단계에서의 과적합 여부를 모니터링한다.

## 핵심 하이퍼파라미터 및 학습 제어

Qwen2.5-32B 모델의 풀 파인튜닝을 위해 분산 환경(A100 80GB × 2장 기준)에 맞춘 현실적인 하이퍼파라미터 및 스케줄링 설정이다.

| 파라미터 | 초기값 | 비고 |
|----------|--------|------|
| Learning rate | 1e-5 | 풀 파인튜닝 기준 기본 학습률 |
| LR schedule | Cosine with warmup | 코사인 감쇠 스케줄러 적용 |
| Warmup ratio | 0.03 | 총 학습 스텝의 3% 비율로 웜업 진행 (절대 스텝 대신 비율 관리) |
| Batch size (Effective) | 16 ~ 64 | Per-device Batch Size: 1~2<br>Gradient Accumulation Steps: 8~16<br>GPU 수(2장)와 연계하여 조정하여 VRAM OOM 방지 |
| Token budget | 50M ~ 200M tokens | epoch 대신 총 학습 토큰 수 기준으로 학습량 제어 |
| Max seq length | 4096 | 창작 노트(CoT) + 시 초안 및 반복 수정 텍스트 포함 |
| Early stopping | patience = 3 | eval loss 기준, 3회 평가 주기 연속 개선 없을 시 조기 종료 |

### 조기 종료 (Early Stopping) 상세
소규모 시 데이터셋 학습 시 발생할 수 있는 과적합(Overfitting) 위험을 방지하기 위해 다음과 같이 HuggingFace `EarlyStoppingCallback`을 적용한다.
```python
from transformers import EarlyStoppingCallback

# Trainer 정의 시 callbacks에 추가
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
    callbacks=[EarlyStoppingCallback(early_stopping_patience=3)]
)
# eval_loss 기준, 3회 평가 주기 연속으로 개선이 발생하지 않으면 학습을 중단한다.
```

## 특수 토큰 처리 및 임베딩 초기화

풀 파인튜닝 시 추가되는 특수 토큰(`<행갈이>`, `<연갈이>` 등)의 임베딩이 랜덤하게 남아있으면 학습 초기 수렴이 느려지거나 주변 컨텍스트 표현이 망가질 수 있다. 이를 방지하기 위해 의미적으로 가장 유사한 기존 표준 토큰들의 임베딩을 복사하거나 평균을 내어 초기화한다.

### 임베딩 초기화 파이썬 스크립트

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

    # 1. 새 특수 토큰 추가 및 모델 토큰 임베딩 레이어 확장
    special_tokens = ["<행갈이>", "<연갈이>", "<시작>", "<끝>"]
    tokenizer.add_special_tokens({"additional_special_tokens": special_tokens})
    model.resize_token_embeddings(len(tokenizer))

    # 2. 신규 특수 토큰과 기존 토큰 간 매핑 정의
    # - <행갈이>는 줄바꿈(\n) 토큰 복사
    # - <연갈이>는 줄바꿈 연속 입력 또는 빈 줄에 준하는 개행들의 임베딩 평균 복사
    # - <시작>, <끝>은 기존 챗 템플릿의 시작/끝 또는 BOS/EOS 토큰 복사
    token_mapping = {
        "<행갈이>": ["\n"],
        "<연갈이>": ["\n\n", "\n"],
        "<시작>": ["<|im_start|>", "<s>"],
        "<끝>": ["<|im_end|>", "</s>"]
    }

    embeddings = model.get_input_embeddings()
    embedding_weight = embeddings.weight.data

    # 3. 매핑 기반 임베딩 초기화 수행
    for new_token, source_tokens in token_mapping.items():
        new_id = tokenizer.convert_tokens_to_ids(new_token)
        if new_id == tokenizer.unk_token_id:
            print(f"Warning: {new_token} is not registered in tokenizer.")
            continue

        source_ids = []
        for src in source_tokens:
            src_id = tokenizer.convert_tokens_to_ids(src)
            # 토크나이저에 해당 토큰이 실존하는 경우에만 사용
            if src_id != tokenizer.unk_token_id:
                source_ids.append(src_id)

        # fallback: 매핑된 소스 토큰이 없다면 기본 개행(\n) 사용
        if not source_ids:
            source_ids = [tokenizer.convert_tokens_to_ids("\n")]

        # 평균 임베딩 계산 및 주입
        with torch.no_grad():
            source_embeddings = embedding_weight[source_ids]
            mean_embedding = source_embeddings.mean(dim=0).clone()
            embedding_weight[new_id] = mean_embedding
        
        print(f"Success: Initialized {new_token} (ID: {new_id}) with mean embedding of {source_tokens} (IDs: {source_ids})")

if __name__ == "__main__":
    initialize_special_token_embeddings()
```

## 평가 체크포인트

매 N 스텝마다 다음을 평가:
1. 행갈이/연갈이 위치 정확도
2. 소수의 시 프롬프트로 생성 샘플 수동 평가
3. Perplexity on held-out poetry set

## 미결 사항

1. **Stage 전환 시점의 옵티마이저 상태 제어**: Stage가 전환될 때 Adam optimizer의 모멘텀 및 배리언스 상태를 완전히 초기화(Warm Restart)할 것인가, 아니면 스케줄러의 연속성을 유지할 것인가? 초기화하지 않을 경우 이전 Stage 데이터의 그래디언트 바이어스가 새 Stage 학습 초기 단계에 미치는 악영향을 어떻게 최소화할 것인가?
2. **CoT(창작노트) 영역과 시 본문 영역의 Loss Masking 여부**: 모델이 시 창작 과정의 메타 사고(CoT)를 학습하도록 유도하되, 최종 결과물인 시 자체의 퀄리티와 특수 토큰 제어력을 극대화하기 위해 CoT 영역의 loss 가중치를 마스킹하거나 낮추는 설계가 성능 향상에 기여할 것인가?
3. **다국어 시 데이터 학습 시 어휘사전(Vocabulary) 확장 방식**: 다국어 및 외국어 시를 학습할 때, 한국어 기반 모델의 기본 토크나이저 어휘사전으로 인해 발생하는 심각한 토큰 파편화(Token fragmentation)를 극대화하지 않고, 시적 뉘앙스와 외래어 표현을 완벽히 보존하기 위한 추가적인 Vocabulary 최적화 방법은 무엇인가?
