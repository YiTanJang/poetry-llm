---
type: Design
title: Continual Pretraining 전략
description: 시/문학 코퍼스 CPT → CoT SFT 2단계 학습 전략과 커리큘럼 학습과의 관계.
tags: [model, training, infrastructure]
timestamp: 2026-06-27T00:00:00Z
---

# Continual Pretraining 전략

## Continual Pretraining (CPT)이란

Continual Pretraining(CPT, 계속 사전학습)은 이미 사전학습된 베이스 모델을
새로운 도메인 코퍼스로 **다음 토큰 예측(next token prediction)** 목표로 추가 학습하는 단계다.

Instruction tuning(SFT)과 구분되는 핵심 차이:

| 구분 | CPT | SFT (Instruction Tuning) |
|------|-----|--------------------------|
| 학습 목표 | 다음 토큰 예측 (언어 모델링) | 지시 따르기 (instruction-output 매핑) |
| 데이터 형식 | 원시 텍스트 코퍼스 | (instruction, output) 페어 |
| 레이블 | 텍스트 자체가 레이블 | output 토큰만 loss 계산 |
| 목적 | 도메인 언어 감각 내재화 | 창작/추론 행동 학습 |
| 학습량 | 수억~수십억 토큰 | 수백만 토큰 이하도 효과적 |

**비유**: CPT는 "시 수천 편을 혼자 읽으며 감각을 기르는 것",
SFT는 "선생님과 함께 시를 쓰고 피드백 받는 것".

---

## 2단계 전략: CPT → SFT

### 개요

```
[베이스 모델: Qwen2.5-32B]
        ↓
[1단계: CPT — 시/문학 코퍼스 언어 모델링]
        ↓
[CPT 체크포인트]
        ↓
[2단계: SFT — CoT 창작 노트 + 시 생성 instruction tuning]
        ↓
[최종 파인튜닝 모델]
```

### 장단점

**장점**

1. **점진적 도메인 적응**: 베이스 모델이 시 언어의 어휘, 리듬, 이미지 패턴을
   먼저 흡수한 뒤 창작 지시를 학습하므로 SFT 효율이 높아진다.
2. **Catastrophic forgetting 완화**: SFT 단독보다 중간 CPT 모델이 베이스 역량을
   더 잘 보존한다. (아래 상세 설명 참고)
3. **SFT 데이터 희소성 보완**: CoT 창작 페어 데이터는 수천 건 수준이지만,
   CPT용 시 코퍼스는 수십만 건 확보 가능 — 두 단계가 서로 보완.
4. **중간 체크포인트 활용**: CPT 모델만으로도 시 완성, 시 분류 등 기초 작업에
   사용 가능 — 실험 유연성 증가.

**단점**

1. **학습 비용 증가**: CPT 단계 추가로 총 컴퓨팅 비용 1.5~2배 증가.
2. **파이프라인 복잡도**: 두 단계의 하이퍼파라미터를 각각 최적화해야 함.
3. **CPT 코퍼스 품질 의존성**: 저품질 코퍼스로 CPT 시 오히려 모델 능력 저하 가능.
4. **실험 사이클 증가**: "CPT 결과가 안 좋으면 SFT도 재시작" — 시간 비용.

---

## 1단계 CPT: 시/문학 코퍼스로 언어 감각 주입

### 데이터 구성

```
CPT 코퍼스 구성 (총 목표: 100M~500M 토큰)

비중    | 데이터 종류
--------|--------------------------------------------------
40%     | 한국 현대시 (공공도메인, CC 라이선스)
20%     | 외국어 시 번역본 (한국어 번역 포함)
15%     | 시론 / 시 평론 / 시 해설서
10%     | 시인 에세이 / 인터뷰 / 창작 산문
10%     | 문학 소설 (시적 문체 산문 — 이상, 김승옥 등)
5%      | 시 잡지 / 동인지 (비공공도메인은 라이선스 확인 필요)
```

**원시 텍스트 형식** — 특별한 포맷 없이 텍스트를 연속으로 이어붙인다:

```
# CPT 데이터 예시 (연속 텍스트)
눈이 오는가 북쪽엔
함박눈 쏟아져 내리는가
<행갈이>
어머니...
<연갈이>
봄은 고양이로다
...
[다음 시]
...
```

행갈이/연갈이 특수 토큰은 CPT 단계부터 포함하여 모델이 구조 토큰을 자연스럽게 습득하게 한다.

### CPT 하이퍼파라미터

| 파라미터 | 권장값 | 근거 |
|----------|--------|------|
| Learning rate | 5e-6 ~ 1e-5 | 베이스 모델 보존하면서 도메인 적응 |
| LR schedule | Cosine with warmup | 안정적인 감쇠 |
| Warmup ratio | 0.01~0.03 | 총 스텝의 1~3% |
| Batch size (effective) | 2M~4M 토큰/배치 | 대규모 코퍼스 학습 표준 |
| Per-device batch size | 2~4 (seq_len=4096) | A100 80GB VRAM 한계 |
| Gradient accumulation | 128~256 | effective batch size 확보 |
| Max seq length | 4096 | 시 + 시론 함께 포함 가능 |
| 총 학습 토큰 | 1~3 epoch (코퍼스 규모에 따라) | 과적합 전에 중단 |
| Weight decay | 0.1 | 정규화 |
| Gradient clipping | 1.0 | spike 방지 |

**CPT에서 epoch 개념 주의**: 대규모 코퍼스는 1 epoch도 충분.
소규모 시 코퍼스(수십만 편)라면 2~3 epoch 반복.
`학습 토큰 = 코퍼스 토큰 수 × epoch 수`로 관리.

### CPT 종료 기준

```
다음 중 하나 충족 시 CPT 종료:
  1. 목표 학습 토큰 수 도달 (예: 200M 토큰)
  2. eval perplexity가 N 평가 주기 연속 개선 없음 (patience=3)
  3. 생성 샘플에서 시적 언어 감각 확인 (수동 평가)

CPT 완료 확인 방법:
  "봄"이라는 단어로 시작하는 텍스트를 자유 생성(no instruction)
  → 시적 형태로 자연스럽게 이어지면 CPT 성공 신호
  → 산문 형태로 이어지면 CPT 추가 필요
```

---

## 2단계 SFT: CoT 창작 노트 + 시 생성 Instruction Tuning

### SFT 데이터 구성

CPT 완료 모델에 instruction-following 능력을 부여하는 단계.

```
SFT 데이터 구성 (총 목표: 5,000~50,000건)

비중    | 포맷 (training_data_formats.md 기준)
--------|--------------------------------------------------
30%     | 포맷 3 — 창작노트 → 시 (CoT 핵심)
20%     | 포맷 7 — 소재 → 시 (창작 요청)
20%     | 포맷 4 — 수정 과정 데이터
15%     | 포맷 2 — 시 + 시론 페어
10%     | 포맷 5 — 시 → 설명 (역방향)
5%      | 포맷 6,8 — 빈칸 채우기, Novelty 실험
```

### SFT 하이퍼파라미터

| 파라미터 | 권장값 | CPT와의 차이 |
|----------|--------|-------------|
| Learning rate | 3e-6 ~ 8e-6 | CPT보다 낮게 — 세밀한 조정 |
| LR schedule | Cosine with warmup | 동일 |
| Warmup ratio | 0.03~0.05 | CPT보다 높게 — 데이터 수 적음 |
| Effective batch size | 32~128 | CPT보다 작게 — 페어 데이터 |
| Epochs | 2~4 | 데이터 수 적으면 epoch 늘림 |
| Max seq length | 4096 | 창작노트 + 시 충분히 포함 |

### SFT에서 loss mask 처리

instruction 토큰에는 loss를 계산하지 않고 output(시) 토큰에만 계산:

```python
# instruction 부분은 label을 -100으로 마스킹
# HuggingFace DataCollatorForSeq2Seq 또는 커스텀 collator 사용

def preprocess_function(examples):
    model_inputs = tokenizer(
        examples["instruction"],
        examples["output"],
        truncation=True,
        max_length=4096,
    )
    # instruction 길이만큼 label을 -100으로 마스킹
    instruction_len = len(tokenizer(examples["instruction"])["input_ids"])
    model_inputs["labels"] = [-100] * instruction_len + \
                              model_inputs["input_ids"][instruction_len:]
    return model_inputs
```

---

## 이 방식이 단일 파인튜닝보다 나은 이유: Catastrophic Forgetting 관점

### Catastrophic Forgetting이란

대형 언어 모델을 새 도메인 데이터로 파인튜닝하면 기존 능력(한국어 문법,
일반 지식, 추론 능력)이 손상되는 현상. 특히 SFT 데이터가 소규모일 때 심각.

### 단일 SFT의 문제

```
[베이스 모델] → [SFT 직접 적용 (수천 건)]
                       ↓
문제: 소량의 SFT 데이터가 베이스 모델의 일반 언어 능력을 덮어씀
  → 시 도메인 외 질문에 대한 응답 품질 저하
  → 모델이 "시만 아는" 좁은 모델이 될 위험
```

### CPT → SFT의 장점

```
[베이스 모델]
    ↓ CPT (대규모 텍스트, next token prediction)
[CPT 모델] ← 베이스 능력 대부분 보존 + 시 언어 감각 추가
    ↓ SFT (소량 페어 데이터)
[최종 모델] ← SFT가 이미 시 언어 감각을 가진 모델 위에서 작동
               → 더 적은 데이터로 더 깊은 창작 능력 달성
```

**왜 CPT가 forgetting을 줄이는가**:
- CPT의 next token prediction은 기존 사전학습과 **동일한 목표**이므로 
  베이스 모델의 기존 파라미터를 "대화" 방식으로 천천히 조정.
- SFT는 새로운 행동 패턴을 강제하므로 기존 패턴과 충돌 가능.
- CPT로 시 언어를 먼저 "자연스러운 방식"으로 내재화하면
  SFT가 더 작은 변화로도 창작 지시를 따를 수 있게 된다.

### 실증적 근거

- Gururangan et al. (2020) "Don't Stop Pretraining": 도메인 CPT 후 SFT가
  SFT 단독 대비 일관되게 성능 향상을 보임.
- LLaMA 계열 모델의 의료/법률 특화: CPT → SFT 2단계가 표준 파이프라인으로 정착.

---

## finetuning_strategy.md의 커리큘럼 학습과의 관계 및 차이점

### 개념 비교

| 항목 | 커리큘럼 학습 (finetuning_strategy.md) | CPT → SFT 2단계 |
|------|----------------------------------------|----------------|
| 핵심 아이디어 | 데이터 난이도 순서로 SFT 진행 | 학습 목표 자체를 CPT → SFT로 전환 |
| 데이터 형식 | 모두 SFT 포맷 (instruction-output) | CPT: 원시 텍스트 / SFT: 페어 |
| 단계 구분 기준 | 데이터 복잡도/추상성 | 학습 패러다임 |
| 모델 목표 | Stage마다 다른 SFT 데이터 학습 | Stage 1은 언어 모델링, Stage 2는 지시 따르기 |
| Catastrophic forgetting 대응 | 점진적 난이도로 간접적 완화 | CPT 단계 자체가 직접적 완화 |

### 통합 방안: CPT + 커리큘럼 SFT

두 전략은 배타적이지 않다. 다음과 같이 결합 가능:

```
[베이스 모델: Qwen2.5-32B]
        ↓
[CPT — 시/문학 코퍼스 (100M~500M 토큰)]
  └── 행갈이/연갈이 특수 토큰 포함 원시 텍스트
        ↓
[CPT 체크포인트]
        ↓
[커리큘럼 SFT Stage 1 — 언어 기반 + 형식 학습]
        ↓
[커리큘럼 SFT Stage 2 — 메타 지식 (시론/평론)]
        ↓
[커리큘럼 SFT Stage 3 — CoT 창작 (창작노트→시)]
        ↓
[커리큘럼 SFT Stage 4 — 수정/반복 학습]
        ↓
[최종 모델]
```

**이 통합 전략의 장점**:
- CPT가 언어 감각 기반을 탄탄히 쌓음 → 커리큘럼 SFT의 Stage 1~2를 더 빨리 흡수
- 커리큘럼의 점진적 난이도가 CPT 모델의 창작 능력을 체계적으로 개발
- 각 Stage 체크포인트를 보존하면 실험 유연성 최대화

### 현재 finetuning_strategy.md와의 차이점 정리

1. **finetuning_strategy.md의 Stage 1 ("언어 기반")은 CPT로 대체 가능**:
   현재 설계에서 Stage 1이 "공공도메인 시, 외국어 시"를 SFT 포맷으로 학습하는데,
   이를 원시 텍스트 CPT로 전환하면 더 자연스럽고 효과적.

2. **CPT는 훨씬 많은 데이터를 소화할 수 있다**:
   SFT의 instruction-output 포맷으로 만들기 어려운 시 텍스트 수십만 편을
   CPT로 그대로 학습에 활용 가능.

3. **커리큘럼 학습은 SFT 내부 순서 문제**:
   CPT → SFT는 학습 패러다임 전환이고, 커리큘럼은 SFT 데이터 제시 순서.
   서로 다른 차원의 전략이므로 동시에 적용 가능.

---

## 미결 사항

- CPT 코퍼스 규모 확정 (수집 가능한 시 텍스트 총량 조사 필요)
- CPT 단계에서 특수 토큰 처리 방식 확정 (행갈이 토큰이 원시 텍스트에 이미 포함되는가?)
- CPT → SFT 전환 시 learning rate reset 여부 (일반적으로 reset 권장)
- CPT 단독 모델 vs CPT+SFT 모델 벤치마크 비교 계획
- 컴퓨팅 비용 추가 발생분 예산 확보 (CPT 추가로 총 비용 1.5~2배 증가)
