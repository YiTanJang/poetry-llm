---
type: Design
title: PEFT vs 풀 파인튜닝 비교
description: Qwen2.5-32B 모델의 시 생성에 대한 LoRA(PEFT)와 풀 파인튜닝의 비교 분석 및 전환 전략.
tags: [model, fine-tuning, full-finetuning, lora, peft, training]
timestamp: 2026-06-27T00:00:00Z
---

# PEFT vs 풀 파인튜닝 비교 (Q4)

한국어 현대 시 창작이라는 극도의 예술적/미학적 태스크에서 **Qwen2.5-32B** 모델을 최적화하기 위해, 매개변수 효율적 미세조정(PEFT/LoRA)과 풀 파인튜닝(Full Fine-Tuning)의 기술적 차이점을 비교하고 상호 보완적인 하이브리드 전환 전략을 수립합니다.

---

## 1. 상세 비교 매트릭스 (Comparison Matrix)

현대 시 생성의 특수성(새로운 결합 수사학, 고유 토큰 제어, 압도적 미학적 자율성)을 고려한 두 기법의 비교는 다음과 같습니다.

| 비교 항목 | PEFT / LoRA | 풀 파인튜닝 (Full Fine-Tuning) |
| :--- | :--- | :--- |
| **학습 다이내믹스 (Learning Dynamics)** | - 사전 학습된 가중치 $W_0$를 동결하고, 저차원 분해 행렬 $B$와 $A$만 업데이트합니다 ($W = W_0 + \Delta W$).<br>- 기존 모델의 표현 공간을 제한적으로 변형하므로 일반 상식 및 기초 문법 유지력이 뛰어납니다.<br>- 그러나 시적 허용, 낯설게 하기(Novelty)와 같은 **기존 언어 질서의 근본적 파괴 및 새로운 미학적 패턴 생성**에는 표현 용량(Representation Capacity) 부족으로 제약이 따릅니다. | - 모델의 모든 매개변수($32.5\text{B}$)를 업데이트합니다.<br>- 어텐션 헤드와 MLP 층의 모든 결합 강도를 전면 재배치하여, 시적 은유와 파격적인 문장 구조를 표현하는 데 필요한 고유의 잠재 공간(Latent Space)을 완벽하게 재구성할 수 있습니다.<br>- 미학적 표현력과 창의적 다양성(Novelty) 측면에서 압도적으로 유리하지만, 정규화 실패 시 붕괴(Catastrophic Forgetting) 위험이 존재합니다. |
| **어휘 조정 (Vocabulary Adjustments)** | - 기본적으로 임베딩 및 LM Head 레이어를 동결합니다.<br>- `<행갈이>`, `<연갈이>` 등의 시 창작 전용 특수 토큰을 추가할 경우, 임베딩 레이어를 학습 대상에 별도로 포함(`modules_to_save`)해야 하므로 순수 LoRA 설계가 복잡해지며 수렴 속도가 느려집니다. | - 임베딩 및 LM Head를 포함한 전체 파라미터를 동시에 최적화합니다.<br>- 새로 도입된 시 특수 토큰과 기존 어휘 간의 상호 작용 및 문맥적 가중치 배정이 자연스럽고 빠르며, 시적 리듬과 발음 레이어 반영이 매끄럽습니다. |
| **메모리 오버헤드 (Memory Overhead)** | - 업데이트 대상 파라미터가 전체의 $1\%$ 미만(일반적으로 $0.2\sim 0.5\%$)입니다.<br>- 32B 모델 기준 Optimizer States 메모리가 수십 GB에서 2~4GB 내외로 격감하여, 단일 80GB GPU 혹은 최소한의 다중 GPU 노드에서 쉽게 학습이 가능합니다. | - $32.5\text{B}$ 전체 파라미터의 가중치, 그래디언트, 그리고 Optimizer States를 모두 메모리에 적재해야 합니다.<br>- FP16/BF16 가중치 저장에만 약 65GB가 필요하며, FP32 Master Weight를 사용하는 AdamW 최적화 도구 사용 시 최소 520GB 이상의 VRAM 오버헤드가 발생하여 FSDP/ZeRO-3 분산 학습 인프라가 필수적입니다. |
| **최적화 복잡도 (Optimization Complexity)** | - 랭크($r$), 스케일링 계수($\alpha$), 적용 모듈 설정 등 튜닝해야 할 하이퍼파라미터가 추가됩니다.<br>- 극단적인 수사학적 최적화 과정에서 로컬 미니마에 갇히거나 학습 곡선이 조기에 플래토(Plateau)를 형성할 위험이 있습니다. | - 초정밀한 하이퍼파라미터 제어가 필요합니다 (예: 극도로 낮은 LR, 정밀한 Warmup).<br>- 분산 환경(FSDP) 인프라 세팅 및 통신 병목 해결을 위한 엔지니어링 복잡도가 매우 높습니다. |

---

## 2. 하이브리드 / 전환 전략 (Hybrid / Transition Strategy)

우리는 자원의 효율성과 모델의 미학적 극대화를 동시에 달성하기 위해 **LoRA 파일럿 단계에서 시작하여 정량적 트리거를 만족할 때 풀 파인튜닝으로 전환**하는 투 트랙(Two-Track) 하이브리드 전략을 채택합니다.

```mermaid
graph TD
    A[Phase 1: LoRA 파일럿 학습] --> B{전환 트리거 만족 여부 검사}
    B -- "만족 (Trigger Fire)" --> C[Phase 2: 풀 파인튜닝 (Full FT) 전환]
    B -- "미달 (Iterative Tuning)" --> A
    
    subgraph 전환 트리거 평가 기준
    T1[1. Novelty 지표 정체: 3-gram 중복도 대비 독창성 평탄화]
    T2[2. 특수 토큰 제어 오류율 > 5% 발생]
    T3[3. 인간/LLM 평가지표 한계 직면: 점수 3.5/5.0 이하]
    end
    
    T1 -.-> B
    T2 -.-> B
    T3 -.-> B
```

### 2.1 단계별 학습 목표
1.  **Phase 1: LoRA SFT (신속한 프레임워크 검증)**
    *   데이터 파이프라인 오류 수정, 토큰 결합 상태 모니터링, 기본 창작 노트(CoT) 포맷 제어가 작동하는지 빠르게 실험합니다.
2.  **Phase 2: Full Fine-Tuning (미학적 고도화)**
    *   토큰 수준의 미세한 문체 형성 및 예측 불가능한 은유적 결합력을 극대화하여 독창성(Novelty)이 확보된 최종 시 생산 단계로 진입합니다.

### 2.2 전환 정량적 트리거 (Metrics & Triggers)
*   **Novelty 지표 임계치 달성 실패**: 시 코퍼스 내 3-gram 중복 비율이 LoRA 학습 진행 중 3 Epoch 이상 감소하지 않거나, 생성된 시의 코사인 유사도(Embedding Distance)가 베이스 모델 대비 유의미하게 멀어지지 못할 때 (즉, 기존 학습 데이터의 모방 수준에 그칠 때).
*   **특수 토큰 제어 오류율 (Formatting Error Rate)**: `<행갈이>`, `<연갈이>` 등의 구조적 특수 토큰의 미학적 배치 규칙 오류 및 누락율이 LoRA SFT 완료 후에도 $5\%$를 초과할 때 (LoRA 가중치 제한으로 인해 강력한 포맷팅 제약이 내재화되지 못하는 현상).
*   **Aesthetic Score 정체**: LLM-as-a-judge 평가 및 1차 인간 평가에서 '비유의 참신함' 항목의 평균 점수가 5점 만점에 3.5점 이하로 정체될 때.

---

## 3. 세부 하이퍼파라미터 설정 (Hyperparameter Configurations)

Qwen2.5-32B 모델의 안정적인 학습을 위한 기법별 권장 하이퍼파라미터 셋업입니다.

### 3.1 LoRA (PEFT) 하이퍼파라미터

> 탐색중 — rank, alpha, LR 등 수치는 파일럿 실험으로 최적화됨

| 하이퍼파라미터 | 권장 설정값 | 비고 / 목적 |
| :--- | :--- | :--- |
| **Base Model** | Qwen2.5-32B | BF16 / FP16 정밀도 로드 |
| **Learning Rate (LR)** | $1\times 10^{-4}$ (or $2\times 10^{-4}$) | Cosine Decay 스케줄러 적용 |
| **Global Batch Size** | $64$ | Micro-batch size $2$ ~ $4$ + Gradient Accumulation |
| **Rank ($r$)** | $64$ | 어텐션 및 MLP 계층 전반의 표현 공간 확보 |
| **Alpha ($\alpha$)** | $128$ | 스케일링 계수 ($2\times r$) |
| **Target Modules** | `q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj` | 모델 내 모든 Linear 레이어에 어댑터 적용 |
| **Modules to Save** | `embed_tokens`, `lm_head` | 추가된 시 특수 토큰 가중치 튜닝 허용 |
| **LoRA Dropout** | $0.05$ | 어댑터 오버피팅 완화 |
| **Token Budget** | $10\text{M}$ tokens | 파일럿 테스트를 위한 경량 예산 |

### 3.2 풀 파인튜닝 (Full Fine-Tuning) 하이퍼파라미터

> 탐색중 — LR, 배치 크기, token budget은 파일럿 결과 후 조정 가능

| 하이퍼파라미터 | 권장 설정값 | 비고 / 목적 |
| :--- | :--- | :--- |
| **Base Model** | Qwen2.5-32B | CPT(Continual Pretraining) 완료 가중치 우선 적용 |
| **Learning Rate (LR)** | $5\times 10^{-6}$ (max $1\times 10^{-5}$) | 붕괴 및 발산 방지를 위한 극소 LR 채택 |
| **Global Batch Size** | $128$ (or $256$) | Micro-batch size $1$ ~ $2$ + 대형 Gradient Accumulation |
| **Optimizer** | AdamW (32-bit Master Weights) | FSDP를 통한 상태 분산 |
| **Weight Decay** | $0.1$ | 강력한 L2 정규화를 통해 Catastrophic Forgetting 방지 |
| **Warmup Ratio** | $0.08$ | 안정적인 초기 수렴을 위한 부드러운 시작 |
| **Token Budget** | $30\text{M} \sim 50\text{M}$ tokens | 고정밀 현대 시 데이터셋 집중 반복 학습 |

---

## 4. VRAM 및 메모리 프로파일링 추정 (Memory Profiling)

Qwen2.5-32B 모델 학습 시 하드웨어(GPU VRAM) 가용성에 따른 최적화 최적 경로 도출을 위한 프로파일링 수치입니다.

### 4.1 파라미터당 고정 메모리 공식
*   **모델 가중치(BF16)**: $2 \text{ bytes} \times N$
*   **그래디언트(BF16)**: $2 \text{ bytes} \times N$
*   **Optimizer States (AdamW 32-bit)**: $12 \text{ bytes} \times N$ (Master Weight 4B + Momentum 4B + Variance 4B)
*   **Optimizer States (AdamW 8-bit)**: $6 \text{ bytes} \times N$ (Master Weight 4B + quantized Momentum/Variance 2B)
*   **Optimizer States (Adafactor)**: $\approx 0.05 \text{ bytes} \times N$ (행/열 팩터 분해 기법 적용)

### 4.2 최적화 기법에 따른 VRAM 정적 요구량 비교 (32.5B 파라미터 기준)
> [!NOTE]
> 아래 수치는 Activation Memory(배치 사이즈 및 시퀀스 길이에 따라 비례적으로 늘어나는 실행 메모리)를 제외한 오직 **정적 모델 상태(Static Model States)**만을 추정한 최소 기준입니다.

| 학습 기법 | Optimizer | 모델 가중치 (GB) | 그래디언트 (GB) | Optimizer (GB) | 총 정적 VRAM (GB) | 8x A100 (80G) 노드 분산 시 (GPU당 최소 정적 VRAM) |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| **LoRA (r=64)** | AdamW 32-bit | $65$ (동결) | $\approx 0.5$ | $\approx 3$ | **$\approx 68.5$** | 단일 GPU 구동 가능 ($\approx 70\text{ GB}$) |
| **Full FT** | AdamW 32-bit | $65$ | $65$ | $390$ | **$520$** | **$\approx 65.0\text{ GB}$** (FSDP Full Shard 필수) |
| **Full FT** | AdamW 8-bit | $65$ | $65$ | $195$ | **$325$** | **$\approx 40.6\text{ GB}$** (안정적 마진 확보) |
| **Full FT** | Adafactor | $65$ | $65$ | $\approx 1.6$ | **$\approx 131.6$** | **$\approx 16.5\text{ GB}$** (활성화 메모리 추가 할당 극대화) |

> [!IMPORTANT]
> **그래디언트 체크포인팅(Gradient Checkpointing)**을 반드시 활성화해야 합니다. 이를 적용하지 않을 경우, 시퀀스 길이 $4,096$ 기준 역전파(Backpropagation) 단계에서 생성되는 Activation VRAM이 $200\text{ GB}$를 초과하여 대규모 GPU 클러스터에서도 OOM(Out of Memory)이 발생합니다.

---

## 5. 학습 설정 코드 예시 (Implementation Reference)

### 5.1 PEFT/LoRA 설정을 위한 `LoraConfig`
아래는 HuggingFace의 `peft` 라이브러리를 활용하여 Qwen2.5-32B의 전체 Linear Layer 및 시 특수 토큰 보존 임베딩 레이어를 튜닝 타겟으로 삼는 파이썬 코드 예시입니다.

```python
from peft import LoraConfig, get_peft_model
from transformers import AutoModelForCausalLM

# 1. 베이스 모델 로드 (Qwen2.5-32B)
model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen2.5-32B",
    torch_dtype="auto",
    device_map="auto"
)

# 2. LoRA 구성 설정
peft_config = LoraConfig(
    r=64,
    lora_alpha=128,
    target_modules=[
        "q_proj",
        "k_proj",
        "v_proj",
        "o_proj",
        "gate_proj",
        "up_proj",
        "down_proj"
    ],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
    # 새로운 특수 토큰 학습을 위한 임베딩 및 LM Head 타겟 모듈 저장 설정
    modules_to_save=["embed_tokens", "lm_head"]
)

# 3. 모델에 LoRA 어댑터 래핑
model = get_peft_model(model, peft_config)
model.print_trainable_parameters()
```

### 5.2 HuggingFace Trainer + FSDP 풀 파인튜닝 초기화
FSDP(Fully Sharded Data Parallel)를 활용한 풀 파인튜닝 시, HuggingFace `Trainer`의 학습 인자(`TrainingArguments`) 설정을 통한 통합 가이드라인입니다.

```python
from transformers import Trainer, TrainingArguments, AutoModelForCausalLM, AutoTokenizer

# 1. FSDP 설정을 담은 TrainingArguments 정의
training_args = TrainingArguments(
    output_dir="./qwen-32b-poetry-full-ft",
    overwrite_output_dir=True,
    num_train_epochs=3,
    per_device_train_batch_size=1,        # Micro-batch size (VRAM 한계 고려)
    gradient_accumulation_steps=32,       # Global Batch Size = 32 * World Size
    learning_rate=5e-6,
    weight_decay=0.1,
    warmup_ratio=0.08,
    lr_scheduler_type="cosine",
    logging_steps=10,
    save_strategy="steps",
    save_steps=100,
    fp16=False,                           # FSDP 환경에서는 bf16 정밀도 권장
    bf16=True,
    gradient_checkpointing=True,          # Activation 메모리 최적화 필수 활성화
    # HuggingFace FSDP 인자 구성 정의 (ZeRO-3 Full Shard 설정 매핑)
    fsdp="full_shard auto_wrap",
    fsdp_config={
        "fsdp_transformer_layer_cls_to_wrap": "Qwen2RotaryEmbedding", # Qwen 아키텍처에 맞게 조정 필요
        "forward_prefetch": True,
        "limit_all_gathers": True
    },
    deepspeed=None                        # FSDP 네이티브 래퍼 사용 시 None 지정
)

# 2. 모델 및 토크나이저 초기화
model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen2.5-32B",
    torch_dtype="auto"
)
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-32B")

# 3. Trainer 인스턴스 초기화
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,          # 사전에 토큰화 처리된 한국어 시 데이터셋
    tokenizer=tokenizer
)

# 4. 풀 파인튜닝 실행
# trainer.train()
```

---

## 6. 미결 사항

- [TODO] **파일 전체가 Qwen2.5-32B 특정 수치에 의존 중 — 모델 후보 변경됨:** 제목·파라미터 수(32.5B)·VRAM 계산(520GB+)·코드 예시 전체가 Qwen2.5-32B 기준. 현재 베이스 모델 후보는 Gemma 4 27B/31B로 변경됨(`model_selection.md` 참조). 분석 프레임워크는 재사용 가능하나 모델 이름·파라미터 수·VRAM 계산·코드 예시를 확정된 모델 기준으로 전면 재작성 필요. 모델 확정 전까지는 수치를 신뢰하지 말 것.
1. [Ph1] Qwen2.5-32B의 Vocab Expansion 시, 새로운 문맥 특수 토큰(`<행갈이>`, `<연갈이>`)의 임베딩 학습 속도와 일반 파라미터 학습 속도의 불일치를 해결하기 위해 Embeddings layer와 LM Head에만 더 큰 Learning Rate를 적용하는 Grouped LR 전략이 Full FT 및 LoRA 환경에서 각각 학습 안정성에 어떤 영향을 미치는가?
2. [Ph1] 풀 파인튜닝 시 한국어 일반 코퍼스(CPT 데이터 포함)와 현대 시 코퍼스의 적절한 혼합 비율(Mixing Ratio)은 Catastrophic Forgetting(시각적/문법적 붕괴)을 방지하면서 미학적 Novelty를 극대화하는 임계점이 어디인가?
3. [Ph1] LoRA의 Rank ($r$)를 극단적으로 높이는 것(예: $r=256, \alpha=512$)이 Low-Rank 제약을 완화하여 Full Fine-Tuning의 미학적 성능에 근접할 수 있는지, 아니면 Full Fine-Tuning 고유의 파라미터 간 비선형 상호작용 및 표현 유연성을 대체하기 어려운지 실험적 검증 방법은 무엇인가?
