---
type: Design
title: 특수 토큰 추가 방법
description: 행갈이/연갈이 특수 토큰을 HuggingFace 토크나이저와 모델에 추가하는 구체적 절차.
tags: [model, tokenizer, special-tokens]
timestamp: 2026-06-28T00:00:00Z
---

# 특수 토큰 추가 방법

## 정의

| 토큰 | 역할 |
|------|------|
| `<시작>` | 시의 시작 |
| `<끝>` | 시의 끝 |
| `<행갈이>` | 행(줄) 바꿈 |
| `<연갈이>` | 연(스탠자) 구분 빈 줄 |

---

## 1. 토크나이저에 추가

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("google/gemma-4-27b")

SPECIAL_TOKENS = {
    "additional_special_tokens": ["<시작>", "<끝>", "<행갈이>", "<연갈이>"]
}
num_added = tokenizer.add_special_tokens(SPECIAL_TOKENS)
print(f"토큰 {num_added}개 추가됨. 새 vocab 크기: {len(tokenizer)}")

# 저장 (반드시 저장해야 재로드 시 토큰이 유지됨)
tokenizer.save_pretrained("./tokenizer_with_special")
```

> **주의**: `add_special_tokens`는 이미 vocab에 있는 토큰은 건너뜁니다.
> `num_added`를 확인해 실제로 추가됐는지 검증하세요.

---

## 2. 모델 임베딩 확장 및 어휘 사전 경계 정렬 프로토콜 (Vocabulary Boundary Alignment)

토크나이저에 특수 토큰이 추가되어 vocab 크기가 변경되면, 모델의 입력 임베딩(Embedding) 및 출력 임베딩(LM Head) 레이어 크기를 이에 맞게 조정해야 합니다. 이를 위해 HuggingFace의 `resize_token_embeddings`를 호출합니다.

### 2.1. 128 배수 정렬 프로토콜 (128-Alignment Protocol)

Nvidia Ampere/Hopper 등 현대 GPU의 Tensor Core는 행렬 곱 연산 시 차원 크기가 128의 배수로 정렬되어 있을 때 메모리 액세스 효율이 극대화되고 연산 처리 속도가 최적화됩니다. 

- **정렬 원리**: 토크나이저 어휘 사전 크기 `len(tokenizer)`가 128의 배수가 아닐 경우, 부족한 크기만큼 패딩(Padding) 토큰을 가상으로 추가하여 어휘 사전 차원을 128의 배수로 강제 정렬합니다.
- **적용 방법**: `padding_needed = (128 - (len(tokenizer) % 128)) % 128` 계산을 통해 필요한 패딩 크기를 산출한 후, `model.resize_token_embeddings(len(tokenizer) + padding_needed)`를 호출하여 GPU 정렬 및 최적의 Tensor Core 성능을 보장합니다.

```python
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained(
    "google/gemma-4-27b",
    torch_dtype="auto",
    device_map="auto",
)

# 128 배수 정렬 크기 계산 및 확장
vocab_size = len(tokenizer)
padding_needed = (128 - (vocab_size % 128)) % 128
aligned_vocab_size = vocab_size + padding_needed

model.resize_token_embeddings(aligned_vocab_size)
print(f"Vocab size aligned to multiple of 128: {aligned_vocab_size} (padded +{padding_needed})")
```

> [!CAUTION]
> `resize_token_embeddings` 호출을 누락하거나 패딩 없이 `len(tokenizer)` 그대로 확장할 경우:
> 1. `lm_head`와 `embed_tokens` 크기가 불일치하여 런타임 Shape 에러가 발생합니다.
> 2. GPU Tensor Core 최적화 조건(128 배수 정렬)을 충족하지 못해, 대규모 분산 학습이나 FP16/BF16 정밀도 연산 시 Matrix Multiplication 성능 저하가 발생할 수 있습니다.

---

## 3. 임베딩 할당 규칙 및 Gemma 4 토큰화 검증

새로운 특수 토큰이 추가되면 모델의 임베딩 레이어 가중치 테이블에 새로 할당된 인덱스들은 기본적으로 무작위(Random Noise)로 초기화됩니다. 이는 학습 초기에 모델이 이 토큰들을 만났을 때 그래디언트의 요동(Gradient Instability)을 유발하거나 수렴 속도를 현저히 늦추는 원인이 됩니다. 따라서 의미적으로 깊게 연관된 기존 토큰(Semantically-related existing tokens)의 임베딩 벡터를 복사 및 평균화하여 초기화하는 할당 프로토콜을 적용합니다.

### 3.1. 평균 임베딩 할당 규칙 (Average Embedding Assignment Rules)

1. **무작위 초기화 지양**: 무작위 가중치는 사전 학습된 의미 공간(Embedding Space Manifold)의 기하학적 분포와 완전히 어긋나 있으므로, 기존 학습 완료된 안정적인 벡터들의 대표값을 활용합니다.
2. **단일 의미 토큰 매핑**: 새 토큰이 기존 단일 토큰의 기능과 완전히 일치할 때, 해당 기존 토큰의 가중치를 복사(Clone)하여 할당합니다.
   - `<행갈이>` $\rightarrow$ `\n` (ID 198)
   - `<시작>`, `<끝>` $\rightarrow$ `\n` (BOS/EOS 역할의 학습 안전성을 위해 줄바꿈 임베딩 활용)
3. **복합/평균 의미 토큰 매핑**: 새 토큰이 기존의 특정 토큰 시퀀스(예: 연 구분 빈 줄 `\n\n`)에 대응될 경우, 해당 시퀀스를 이루는 개별 토큰 임베딩들의 **산술 평균(Mean Vector)**을 구하여 주입합니다.
   - `<연갈이>` $\rightarrow$ `\n\n` (토큰화 결과인 `[198, 198]` 복수 토큰의 임베딩 벡터 평균)
   - 향후 다중 토큰 의미 매핑 시에도 동일하게 평균 임베딩 복사 로직을 일반화하여 적용합니다.

### 3.2. Gemma 4 개행 문자(`\n`) 토큰화 실험 검증

Gemma 4의 SentencePiece 토크나이저는 줄바꿈 기호(`\n`)와 관련해 다음과 같이 작동하는 것이 실험적으로 확인되었습니다:

1. **`\n` (단일 줄바꿈) 검증**:
   - `tokenizer.encode("\n", add_special_tokens=False)` 결과값은 `[198]`로, 단일 토큰(ID 198)으로 원자적(Atomic)으로 토큰화됩니다.
2. **`\n\n` (두 줄바꿈 / 연갈이) 검증**:
   - `tokenizer.encode("\n\n", add_special_tokens=False)` 결과값은 `[108, 108]`(예시)로, Gemma 4 어휘 사전(Vocabulary)에 별도의 `\n\n` 병합 토큰이 존재하지 않고 두 개의 `\n` 토큰으로 나뉘어 토큰화됩니다.

이 분석을 기반으로 다음과 같이 가중치 초기화 설계를 적용합니다:
* **`<행갈이>` 토큰**: 단일 줄바꿈 `\n` (ID 198)의 임베딩 벡터를 그대로 복사합니다.
* **`<연갈이>` 토큰**: 두 줄바꿈 `\n\n`에 대응하는 토큰들(`[108, 108]`)의 임베딩 벡터들의 평균값(Mean vector)을 구해 복사합니다. 비록 Gemma 4에서는 `[108, 108]`이 동일한 ID의 반복이라 평균을 구하든 단일 임베딩을 쓰든 결과가 같지만, 토크나이저 아키텍처가 변경되거나 다른 베이스 모델로 변경되어 `\n\n`이 멀티 토큰 시퀀스 또는 별개의 단일 토큰으로 인코딩되는 경우를 대비해 **평균 임베딩 복사 로직**을 일반화하여 작성합니다.
* **`<시작>` / `<끝>` 토큰**: 시작과 끝의 경계 제어 토큰도 초기 학습 안정성을 위해 줄바꿈(`\n`)의 임베딩 벡터를 복사합니다. (BOS/EOS 역할에 준함)

### 3.3. 파이썬 기반 평균 임베딩 초기화 메커니즘 (Initialization Logic)

의미적으로 연관된 토큰 리스트의 ID로부터 임베딩 행렬 내 가중치를 추출하고, PyTorch를 이용하여 복사 및 평균 연산을 진행하는 흐름은 다음과 같습니다. 무작위 노이즈를 배제하고 안정적인 초깃값을 생성하기 위해 반드시 `torch.no_grad()` 블록 내에서 인플레이스(In-place) 복사 연산인 `copy_()`를 사용합니다.

```python
import torch

# 1. 대상 토크나이저 내 참조 대상 토큰 시퀀스 인코딩
target_sequence = "\n\n"
referenced_ids = tokenizer.encode(target_sequence, add_special_tokens=False)

# 2. 모델의 입력(Embeddings) 및 출력(LM Head) 임베딩 레이어 확보
input_embeddings = model.get_input_embeddings()
output_embeddings = model.get_output_embeddings()

with torch.no_grad():
    # 3. 참조 토큰 ID 시퀀스에 해당하는 기존 임베딩 벡터들을 추출하여 평균(Mean) 계산
    # referenced_ids에 대응하는 임베딩 벡터 텐서 크기: [len(referenced_ids), hidden_size]
    referenced_vectors = input_embeddings.weight[referenced_ids]
    
    # 세로(dim=0) 축을 기준으로 평균 벡터를 산출하여 1D 텐서 [hidden_size] 생성
    mean_embedding_vector = referenced_vectors.mean(dim=0)
    
    # 4. 새로 추가된 특수 토큰의 ID 획득
    new_token_id = tokenizer.convert_tokens_to_ids("<연갈이>")
    
    # 5. 복제한 평균 벡터를 새 특수 토큰 인덱스 위치에 복사 (In-place copy)
    input_embeddings.weight[new_token_id].copy_(mean_embedding_vector)
    
    # Weight Tying이 비활성화된 경우 출력(LM Head) 임베딩 테이블도 동일하게 평균 연산 후 복사
    if not model.config.tie_word_embeddings:
        referenced_out_vectors = output_embeddings.weight[referenced_ids]
        mean_out_vector = referenced_out_vectors.mean(dim=0)
        output_embeddings.weight[new_token_id].copy_(mean_out_vector)
```

---

## 4. 통합 초기화 및 검증 샘플 코드

HuggingFace 환경에서 토크나이저와 모델을 동시에 로드하고, 특수 토큰 추가, 128 배수 임베딩 정렬(GPU Tensor Core 최적화), 임베딩 복사 초기화, 그리고 최종 저장까지 한 번에 수행하는 완성형 스크립트 예제입니다.

```python
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM

def initialize_poetry_model_and_tokenizer(model_id: str, save_dir: str):
    print(f"Loading base model and tokenizer: {model_id}")
    tokenizer = AutoTokenizer.from_pretrained(model_id)
    model = AutoModelForCausalLM.from_pretrained(
        model_id,
        torch_dtype=torch.bfloat16,
        device_map="auto"
    )
    
    # 1. 특수 토큰 정의 및 추가
    special_tokens = ["<시작>", "<끝>", "<행갈이>", "<연갈이>"]
    num_added = tokenizer.add_special_tokens({"additional_special_tokens": special_tokens})
    print(f"Added {num_added} special tokens to tokenizer.")
    
    # 2. GPU Tensor Core 성능 최적화를 위한 128 배수 정렬 및 임베딩 레이어 확장
    original_vocab_size = len(tokenizer)
    aligned_vocab_size = original_vocab_size
    if aligned_vocab_size % 128 != 0:
        aligned_vocab_size = ((aligned_vocab_size // 128) + 1) * 128
    
    model.resize_token_embeddings(aligned_vocab_size)
    print(f"Resized embedding layers from {original_vocab_size} to {aligned_vocab_size} (128-aligned).")
    
    # 3. Gemma 4 개행 토큰화 실험 검증 및 참조 ID 획득
    newline_ids = tokenizer.encode("\n", add_special_tokens=False)
    double_newline_ids = tokenizer.encode("\n\n", add_special_tokens=False)
    
    print(f"Experimental verification - '\\n' tokenized as: {newline_ids}")
    print(f"Experimental verification - '\\n\\n' tokenized as: {double_newline_ids}")
    
    assert len(newline_ids) == 1, "Expected '\\n' to be tokenized as a single token."
    newline_id = newline_ids[0]
    
    # 4. 임베딩 가중치 복사 초기화
    with torch.no_grad():
        emb = model.get_input_embeddings()      # Input embeddings (nn.Embedding)
        lm_head = model.get_output_embeddings()  # Output embeddings / LM Head (nn.Linear)
        
        # 참조 벡터 추출 (clone)
        ref_single = emb.weight[newline_id].clone()
        ref_double = emb.weight[double_newline_ids].mean(dim=0).clone()
        
        # Output embedding도 복사를 위해 별도 추출 (weight tying이 아니거나 다른 경우 대비)
        ref_single_out = lm_head.weight[newline_id].clone()
        ref_double_out = lm_head.weight[double_newline_ids].mean(dim=0).clone()
        
        # 신규 특수 토큰과 참조 벡터 매핑
        mapping_in = {
            "<행갈이>": ref_single,
            "<연갈이>": ref_double,
            "<시작>": ref_single,
            "<끝>": ref_single,
        }
        
        mapping_out = {
            "<행갈이>": ref_single_out,
            "<연갈이>": ref_double_out,
            "<시작>": ref_single_out,
            "<끝>": ref_single_out,
        }
        
        # Input Embedding 가중치 업데이트
        for tok_str, vec in mapping_in.items():
            tid = tokenizer.convert_tokens_to_ids(tok_str)
            emb.weight[tid].copy_(vec)
            
        # Weight Tying 여부 확인 및 Output Embedding 가중치 업데이트
        if not model.config.tie_word_embeddings:
            print("Weight tying is disabled. Initializing LM Head embeddings separately.")
            for tok_str, vec in mapping_out.items():
                tid = tokenizer.convert_tokens_to_ids(tok_str)
                lm_head.weight[tid].copy_(vec)
        else:
            print("Weight tying is enabled. LM Head shares weight tensor with Input Embeddings.")
            
    print("Embedding initialization completed successfully.")
    
    # 5. 검증 코드 수행
    # 단일 토큰 변환 검증
    for tok in special_tokens:
        ids = tokenizer.encode(tok, add_special_tokens=False)
        assert len(ids) == 1, f"Token {tok} was not added as a single atomic token (got {ids})."
        
    # 간단한 포워드 패스 검증 (Nan/Inf 발생 검사)
    sample_text = "<시작> 가을날의 들판 <행갈이> 벼가 익어간다 <연갈이> 허수아비 하나 서 있다 <끝>"
    inputs = tokenizer(sample_text, return_tensors="pt").to(model.device)
    with torch.no_grad():
        outputs = model(**inputs, labels=inputs["input_ids"])
    print(f"Forward pass verification: loss = {outputs.loss.item():.4f}")
    
    # 6. 토크나이저와 모델 저장
    tokenizer.save_pretrained(save_dir)
    model.save_pretrained(save_dir)
    print(f"Successfully saved initialized tokenizer and model to: {save_dir}")

if __name__ == "__main__":
    initialize_poetry_model_and_tokenizer(
        model_id="google/gemma-4-27b",
        save_dir="./poetry_model_initialized"
    )
```

---

## 5. 흔한 함정 정리

| 함정 | 원인 | 해결 |
|------|------|------|
| `resize_token_embeddings` 누락 | vocab 크기 불일치 → 런타임 에러 | 모델 로드 직후 반드시 호출 |
| vocab 크기 불일치로 체크포인트 저장 실패 | 저장 전 `model.config.vocab_size` 미갱신 | `resize_token_embeddings` 가 자동으로 config 업데이트함 — 저장 전 확인 |
| 토큰이 여러 서브워드로 분리됨 | `add_tokens` 대신 `add_special_tokens` 미사용 | `add_special_tokens({"additional_special_tokens": [...]})` 사용 |
| 재로드 후 특수 토큰 사라짐 | tokenizer 미저장 | `tokenizer.save_pretrained(path)` 필수 |
| lm_head 크기 부족 (weight tying 없는 경우) | head만 resize 누락 | `resize_token_embeddings` 가 두 레이어 모두 처리 |
| 128 배수 정렬 누락 | GPU Tensor Core 최적 연산 조건 불충족으로 인한 FP16/BF16 연산 효율 저하 | `resize_token_embeddings(len(tokenizer) + padding_needed)` 형식으로 128 배수 맞춤 호출 |
| 임베딩 초기화 후 loss 발산 | learning rate 너무 높거나 새 토큰만 warm-up 없이 학습 | 새 토큰 파라미터에 별도로 낮은 lr 적용 또는 warm-up 스텝 확보 |

---

# Citations

- HuggingFace Docs — [Adding tokens to a tokenizer](https://huggingface.co/docs/transformers/main/en/add_new_model#adding-tokens-to-the-tokenizer)
- HuggingFace Docs — [`resize_token_embeddings`](https://huggingface.co/docs/transformers/main_classes/model#transformers.PreTrainedModel.resize_token_embeddings)

---

## 미결 사항

- [TODO] `tokenizer.bos_token = "<시작>"` 및 `tokenizer.eos_token = "<끝>"`으로 등록할 때, vLLM이나 HuggingFace pipeline 등 외부 추론 엔진과의 호환성에 부작용이 없는가?
- [Ph1] 추가된 특수 토큰들의 학습 초기 경사 발산(Gradient Explosion)을 막기 위해, 이 토큰 임베딩 레이어에만 별도의 낮은 학습률(Learning Rate)을 적용하거나 Warm-up 단계를 차등 적용해야 하는가?
- [Ph1] `<행갈이>`와 `<연갈이>`의 임베딩을 단순히 복사/평균하여 사용하는 것보다, 이들의 조합이 문맥에서 올바르게 수렴되도록 하기 위해 continual pretraining 코퍼스에 이 토큰들을 임의 비율로 혼합하여 사전 적응 훈련을 거쳐야 하는가?
