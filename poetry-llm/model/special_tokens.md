---
type: Design
title: 특수 토큰 추가 방법
description: 행갈이/연갈이 특수 토큰을 HuggingFace 토크나이저와 모델에 추가하는 구체적 절차.
tags: [model, tokenizer, special-tokens]
timestamp: 2026-06-27T00:00:00Z
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

tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-32B")

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

## 2. 모델 임베딩 확장

토크나이저 vocab이 늘어났으므로 반드시 `resize_token_embeddings`를 호출해야 합니다.

```python
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen2.5-32B",
    torch_dtype="auto",
    device_map="auto",
)

# vocab 크기를 토크나이저에 맞게 확장
model.resize_token_embeddings(len(tokenizer))
```

> [!CAUTION]
> `resize_token_embeddings` 호출을 빠뜨리면 `lm_head`와 `embed_tokens`의
> shape이 vocab 크기와 맞지 않아 **런타임 에러 또는 잘못된 logit**이 발생합니다.

---

## 3. 임베딩 초기화 — 랜덤 대신 유사 토큰 복사

새 토큰은 기본적으로 랜덤 초기화됩니다. 의미적으로 유사한 기존 토큰에서
임베딩을 복사하면 학습 초기 안정성이 높아집니다.

```python
import torch

with torch.no_grad():
    emb = model.get_input_embeddings()   # nn.Embedding
    lm_head = model.get_output_embeddings()  # nn.Linear

    # 참조 토큰 ID 가져오기
    newline_id  = tokenizer.encode("\n",  add_special_tokens=False)[0]
    # 연갈이는 \n\n (두 줄 바꿈) 임베딩의 평균을 사용
    newline2_id = tokenizer.encode("\n\n", add_special_tokens=False)

    ref_single = emb.weight[newline_id].clone()
    ref_double = emb.weight[newline2_id].mean(dim=0).clone()

    mapping = {
        "<행갈이>": ref_single,
        "<연갈이>": ref_double,
        "<시작>": ref_single,   # bos 역할; \n 에서 시작
        "<끝>":   ref_single,   # eos 역할; 동일
    }

    for token_str, init_vec in mapping.items():
        tid = tokenizer.convert_tokens_to_ids(token_str)
        emb.weight[tid] = init_vec
        # lm_head는 대부분 weight tying — 별도 복사 불필요
        # weight tying 없는 모델이라면 아래 줄도 추가:
        # lm_head.weight[tid] = init_vec

print("임베딩 초기화 완료")
```

### Weight Tying 확인 방법

```python
tied = model.config.tie_word_embeddings  # 대부분 True
print("weight tying:", tied)
```

`True`이면 `embed_tokens`와 `lm_head`가 같은 텐서를 공유하므로
`emb.weight` 수정만으로 충분합니다.

---

## 4. 검증

```python
# 1) vocab 크기 일치
assert len(tokenizer) == model.config.vocab_size or \
       model.get_input_embeddings().weight.shape[0] == len(tokenizer), \
       "vocab size mismatch!"

# 2) 특수 토큰이 단일 ID로 인코딩되는지 확인
for tok in ["<시작>", "<끝>", "<행갈이>", "<연갈이>"]:
    ids = tokenizer.encode(tok, add_special_tokens=False)
    assert len(ids) == 1, f"{tok} → {ids}  (단일 토큰이어야 함)"
    print(f"{tok} → id {ids[0]}")

# 3) 간단한 생성 테스트
sample = tokenizer("<시작> 봄이 왔다 <행갈이> 꽃이 피었다 <끝>",
                   return_tensors="pt").to(model.device)
with torch.no_grad():
    out = model(**sample, labels=sample["input_ids"])
print("loss:", out.loss.item())  # nan/inf 아니면 OK
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
| 임베딩 초기화 후 loss 발산 | learning rate 너무 높거나 새 토큰만 warm-up 없이 학습 | 새 토큰 파라미터에 별도로 낮은 lr 적용 또는 warm-up 스텝 확보 |

---

## 미결 사항

- [ ] Qwen2.5 tokenizer가 `\n`을 단일 토큰으로 처리하는지 실험 확인 필요
- [ ] `<시작>`/`<끝>`을 `bos_token`/`eos_token`으로도 등록할지 결정
      (`tokenizer.bos_token = "<시작>"` 방식)
- [ ] 파일럿(10.7B)에서 임베딩 복사 vs 랜덤 초기화 수렴 속도 비교 실험

# Citations

- HuggingFace Docs — [Adding tokens to a tokenizer](https://huggingface.co/docs/transformers/main/en/add_new_model#adding-tokens-to-the-tokenizer)
- HuggingFace Docs — [`resize_token_embeddings`](https://huggingface.co/docs/transformers/main_classes/model#transformers.PreTrainedModel.resize_token_embeddings)
