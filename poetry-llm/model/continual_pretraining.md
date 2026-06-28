---
type: Design
title: Continual Pretraining 전략
description: 시/문학 코퍼스 CPT → CoT SFT 2단계 학습 전략과 커리큘럼 학습과의 관계.
tags: [model, training, infrastructure]
timestamp: 2026-06-28T00:00:00Z
---

# Continual Pretraining 전략

## DAPT/CPT의 시작점: Base 모델 선택 (Core Design Decision)

DAPT를 시작할 때 이미 지시어 정렬(RLHF/DPO)이 적용된 **Instruct/Chat 모델**을 사용할지, 혹은 일반 **사전학습 Base 모델**을 사용할지 결정해야 한다.

*   **결정**: 본 프로젝트는 **사전학습 Base 모델(Base Model)**에서 DAPT/CPT를 진행하는 것을 원칙으로 삼는다.
*   **근거**:
    1.  **지시 가중치(Instruction Weights) 손상 방지**: 이미 Instruct 튜닝이 완료된 모델은 DAPT 단계에서 원시 텍스트(Raw Text)를 대규모로 노출시키면 이전에 형성된 지시 이행 능력(Instruction-following) 가중치가 급격히 붕괴(drift)한다.
    2.  **스타일 왜곡 방지**: Instruct 모델은 정돈되고 가독성 높은 일상 구어를 생성하는 편향(Bias)이 주입되어 있으므로, 통사 파격이 필요한 현대시의 스타일 변환에 저항성이 생긴다.
    3.  **Instruction Replay 비율의 최적화**: Base 모델에서 출발하므로 CPT 단계에서 Catastrophic Forgetting을 방어하기 위한 대화형(Instruction) 리플레이 버퍼를 대량으로 구축할 필요가 없으며, 일반 한국어 Replay Buffer의 비율을 최소화(5% 이하)하여 순수 시 코퍼스의 학습 밀도를 높일 수 있다.

---

## Mid-training과의 관계

현대 LLM 파이프라인에서 우리가 CPT(Continual Pretraining)라고 부르는 단계는 업계에서 **mid-training**이라고도 불린다. Llama 3, Gemma, Qwen2.5 모두 이 단계를 명시적으로 가진다.

```
Pretraining (범용 웹 데이터, 수T 토큰)
      ↓
Mid-training / CPT  ← 우리가 하는 단계 (Base 모델에서 출발)
      (도메인 특화 + 고품질 데이터 믹스, next-token prediction 유지)
      ↓
SFT → DPO/GRPO
```

**Annealing**: mid-training 마지막 구간에서 학습률을 급격히 낮추면서 동시에 고품질 데이터 비율을 높이는 기법. Llama 3 / Qwen2.5 모두 사용. 우리 DAPT 레시피의 후반부에 적용을 고려할 수 있다 — 예를 들어 마지막 10% 스텝에서 LR을 $1 \times 10^{-6}$으로 고정하고 Replay Buffer 비율을 일시적으로 높이는 방식.

### 스타일 도메인 vs 지식 도메인 DAPT: 토큰 예산

DAPT 토큰 예산 논쟁은 도메인 유형을 구분하지 않으면 오해를 낳는다.

| 도메인 유형 | 예시 | 필요 토큰 규모 | 이유 |
|------------|------|-------------|------|
| **지식 도메인** | 의료, 법률, 코드 | 10B~500B+ 토큰 | 새로운 사실·개념·API를 처음부터 내재화해야 함. Meditron-70B(48B 토큰), Code Llama 34B(500B 토큰) 등 |
| **스타일 도메인** | 시, 소설, 특정 문체 | 수백M 토큰도 효과 가능 | 모델이 이미 한국어를 안다. 출력 분포를 스타일 방향으로 이동시키는 것 |

우리 목표는 **스타일 도메인**이다. 베이스 모델은 이미 한국어를 충분히 알고 있으며, 우리가 원하는 것은 그 출력 분포를 "현대시적 언어"로 밀어주는 것이다. 의학 지식을 새로 가르치는 것과 본질적으로 다르다.

> 가설: 300M 토큰 CPT + 고품질 SFT 조합이 시 스타일 적응에는 충분할 수 있다. 단, 이것이 사실인지는 파일럿 실험 없이 확인 불가.

**CPT 건너뛰기 옵션**: 고품질 시 데이터를 instruction-output 형식으로 충분히 준비할 수 있다면, CPT 없이 SFT 직행도 유효한 전략이다. 최근 연구(BioMistral-7B: 3B 토큰 CPT만으로 유의미한 도메인 적응 달성, 문체 전환 중심 연구들)는 스타일 도메인에서 소규모 CPT 또는 SFT 단독도 의미 있음을 시사한다. Phase 1 파일럿에서 두 전략을 비교 실험해야 한다.

**Synthetic CPT 옵션**: 실제 시 코퍼스가 목표치에 미달할 경우, 프론티어 모델(Claude/GPT-4)을 활용하여 소량의 원본 코퍼스를 요약·재해석·비평·패러프레이즈 등 다양한 지식 표현 구조로 확장하는 Synthetic CPT 전략이 데이터 부족 대응으로 유효하다. 원본의 미학적 다양성 손상 위험이 있으므로 합성 데이터 비율 상한 실험이 필요하다.

---

### PTPP (Tokens-Per-Parameter) 비율의 오용 주의

외부 리뷰에서 자주 제기되는 비판 중 하나가 PTPP 비율($\text{300M} / \text{31B} \approx 0.01$)을 근거로 한 "토큰 절대 부족" 주장이다. 이 주장은 두 가지 이유에서 우리 상황에 잘못 적용된 것이다.

**PTPP는 스크래치 프리트레이닝 지표다**: Chinchilla / Kaplan 스케일링 법칙에서 파생된 PTPP는 모델을 아무것도 없는 상태에서 처음부터 학습할 때 컴퓨팅 예산을 최적화하기 위한 기준이다. 18T 토큰으로 사전학습이 완료된 모델의 CPT에 이 비율을 그대로 대입하면 동역학이 근본적으로 다르다.

**실제 CPT 수렴을 결정하는 두 변수**: 최신 CPT 스케일링 법칙 연구(D-CPT Law, Que et al. 2024; Learning Dynamics in Continual Pre-Training for Large Language Models, 2024; PTPP-Aware Adaptation Scaling Laws, 2024)에 따르면 CPT의 성공은 두 변수가 결정한다:

| 변수 | 내용 |
|------|------|
| **데이터 혼합 비율 (Mixture Ratio)** | 도메인 데이터와 일반 코퍼스의 비율이 손실 수렴의 핵심. 비율 설정 실패가 파국적 망각 또는 과적합의 주원인 |
| **학습률 스케줄링 (LR Annealing)** | 사전학습 최종 LR의 10~20%를 피크로 코사인 감쇠. 완성된 모델 가중치를 과격하게 업데이트하지 않아야 붕괴를 방지 |

두 변수 모두 이미 이 파일의 학습 레시피(코퍼스 블렌드 비율, 학습률 상세 설정)에 반영되어 있다.

---

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

## 1단계 CPT (DAPT): 시/문학 코퍼스로 언어 감각 주입

### 데이터 구성 및 최적 코퍼스 블렌드 (Optimal Corpus Blend)

시 창작 모델의 독창성(Novelty)을 확보하고 일반 언어 능력의 Catastrophic Forgetting(치명적 망각)을 방지하기 위해 설계된 **DAPT(Domain-Adaptive Pre-training) 데이터 믹스**의 최적 블렌드 비율 및 상세 구성은 다음과 같다. 본 비율은 Full-Scale Phase의 **총 300M 토큰 예산**을 기준으로 산출되었다.

| 세부 코퍼스 구분 | 권장 비율 (%) | 토큰 수 (Token Count) | 코퍼스 역할 및 기여 영역 (Aesthetic & Grammar Role) |
| :--- | :---: | :---: | :--- |
| **한국 시 코퍼스** (현대시, 고전시, 근대시, 번역 시 등) | **85.0%** | 255M | 한국어 시의 지배적인 현대적/전통적 행·연 구조, 이미지 연계 방식, 번역 시의 이국적 비유 및 낯선 어휘 패턴 학습. |
| **시 평론 및 시론** (문학 비평 및 해설 문헌) | **10.0%** | 30M | 시의 메타 지식, 문학적 비평 이론, 수사학적 해설에 사용되는 고차원 추상 언어 습득. |
| **일반 한국어 Replay Buffer** (위키, 뉴스 등) | **5.0%** | 15M | CPT 과정 중 일반 상식, 표준 문법 구조, 논리적 문맥 파악력을 보존하여 catastrophic forgetting 차단. |
| **합계** | **100.0%** | **300M** | **DAPT 통합 데이터 믹스 (Qwen2.5 토크나이저 기준)** |

#### 각 서브 코퍼스의 기여 및 구성 방식
1. **한국 시 코퍼스 (85% / 255M)**: 시 창작의 중추적 데이터로, 고유 토큰화 효율을 극대화하기 위해 불필요한 공백을 정제하고, `<행갈이>` 및 `<연갈이>` 특수 토큰을 개행 문자에 매핑하여 강제 삽입한다. 현대 자유시, 고전 시조/한시 번역본, 외국 시 번역을 폭넓게 결합하여 운율과 묘사의 다양성을 극대화한다.
2. **시 평론 및 시론 (10% / 30M)**: 단순 생성에 그치지 않고 생성 모델이 '자신의 시에 대해 설명할 수 있는 메타 인지'를 갖추도록 평론의 논리 전개 패턴을 모델링한다.
3. **Replay Buffer (5% / 15M)**: catastrophic forgetting 방지를 위한 일반 한국어 텍스트. **원시 텍스트 형태만** 사용 (instruction-output 쌍은 SFT 단계로 분리). 구체적 데이터셋 후보:

   | 데이터셋 | 출처 | 특징 |
   |----------|------|------|
   | **모두의말뭉치** | 국립국어원 | 신문·구어·문어·웹 도메인 분리, 연구 목적 무료 신청. 문학/신문 서브셋 선택적 사용 가능 |
   | **CulturaX Korean** | HuggingFace | CC + mC4 한국어 정제본, 약 167B 토큰. 바로 다운로드 가능 |
   | **AI-Hub 한국어 텍스트** | aihub.or.kr | 정부 지원, 도메인 세분화(도서요약·대화·뉴스). 연구 목적 무료 신청 |
   | **나무위키 덤프** | 비공식 | 현대 한국어 어투, 방대한 분량. 품질 편차로 필터링 필요 |
   | **FineWeb-Edu** | HuggingFace | 영어 고품질 교육 텍스트. 논리·추론 능력 보존용으로 소량 추가 가능 |

   > 탐색중 — 아래 목록은 후보이며 라이선스·품질 검증 후 확정. 항목 추가 가능.

   > 가설: 모두의말뭉치(문학/신문 서브셋) + CulturaX Korean 조합이 품질 대비 접근성 면에서 가장 현실적.
   >
   > 주의: **나무위키**는 CC BY-NC-SA(비상업적 조건)로 배포되어 상업적 이용이 불분명하고, 사용자 생성 콘텐츠 특성상 품질 편차·편향이 심함. 사용 여부 재검토 필요.

**원시 텍스트 형식 (Raw Text Format)**
DAPT 단계의 데이터는 별도의 지시어(instruction) 템플릿 없이, 아래와 같이 특수 토큰이 태깅된 원시 텍스트를 연속으로 이어붙여 구성한다.

```markdown
<시작>눈이 오는가 북쪽엔<행갈이>함박눈 쏟아져 내리는가<행갈이>어머니...<연갈이>봄은 고양이로다<행갈이>꽃가루와 같이 부드러운 고양이의 털에<행갈이>고운 봄의 다사함이 어리우도다<끝>
```

---

### DAPT 상세 학습 레시피 (Training Recipe)

> 탐색중 — 하이퍼파라미터 수치(학습률, 배치 크기, 웜업 비율 등)는 파일럿 실험 결과에 따라 조정될 수 있음

Qwen2.5-32B 모델의 풀 파인튜닝을 위해 설계된 DAPT 학습 하이퍼파라미터 및 하드웨어 연동 전략은 다음과 같다.

#### 1. 학습률 및 옵티마이저 상세 설정 (Learning Rate & Optimizer)
- **학습률 스케줄러 (LR Scheduler)**: 선형 웜업(Linear Warmup)과 코사인 어닐링(Cosine Annealing) 스케줄러를 사용한다.
- **최대 학습률 (Peak Learning Rate)**: $5 \times 10^{-5}$ ($5.0 \times 10^{-5}$) (DAPT의 적응 속도를 극대화하기 위해 $5 \times 10^{-5}$를 기본값으로 하되, 하드웨어 및 수렴 안정성에 따라 $1 \times 10^{-4}$까지 상향 조정 가능).
- **웜업 스텝 비율 (Warmup Step Ratio)**: 총 학습 스텝의 **5.0%** ($0.05$). 이 구간 동안 학습률을 $0$에서 최대 학습률까지 선형적으로 증가시켜 새로 추가된 특수 토큰(`<행갈이>`, `<연갈이>` 등)의 임베딩 벡터가 급격하게 변동하여 다른 파라미터를 왜곡하는 현상을 방지한다.
- **최소 학습률 (Target Minimum Learning Rate)**: $1 \times 10^{-6}$ ($1.0 \times 10^{-6}$). 최대 학습률 도달 후 코사인 어닐링 방식으로 감쇠하며, DAPT 종료 시 최종적으로 $1 \times 10^{-6}$에 도달하게 함으로써 국소 최적점 주변에 가중치가 안정적으로 수렴하도록 유도한다.
- **옵티마이저 (Optimizer)**: AdamW 옵티마이저를 사용하며, 하이퍼파라미터는 $\beta_1 = 0.9, \beta_2 = 0.95, \epsilon = 1 \times 10^{-8}$로 설정한다.
- **가중치 감쇠 (Weight Decay)**: 0.1 (Bias 및 LayerNorm 가중치를 제외한 모든 가중치에 적용하여 과적합을 정규화).
- **그래디언트 클리핑 (Gradient Clipping)**: 1.0 (L2 Norm 임계값을 1.0으로 제한하여 그라디언트 스파이크 및 발산 방지).

#### 2. 배치 크기 및 분산 학습 토폴로지 (Batch Size & Parallelism)
- **개별 장비 배치 크기 (Per-Device Batch Size)**: 4 (시퀀스 길이 4,096 기준)
- **그래디언트 누적 스텝 (Gradient Accumulation Steps)**: 128
- **병렬화 H/W 환경**: 8 × A100 80GB GPU 1개 노드 환경
- **유효 배치 크기 (Effective Batch Size)**:
  - **시퀀스 수 기준**: $8 \text{ (GPUs)} \times 4 \text{ (Per-device)} \times 128 \text{ (Accumulation)} = 4,096 \text{ sequences}$
  - **토큰 수 기준**: $4,096 \text{ sequences} \times 4,096 \text{ (seq\_len)} = 16,777,216 \text{ tokens/batch}$ (약 16.8M 토큰/배치)
  - *도입 근거*: DAPT의 대규모 언어 모델링 학습 안정을 확보하기 위해 10M~20M 토큰 스케일의 큰 배치 사이즈를 운용하여 그라디언트 노이즈를 상쇄한다.

#### 3. 시퀀스 길이 및 패킹 전략 (Sequence Length & Packing Strategy)
- **최대 시퀀스 길이 (Max Sequence Length)**: **4,096 tokens** (주의 집중 윈도우 크기는 4,096으로 제한하여 A100 VRAM 내 FSDP/ZeRO-3 동작 마진을 확보).
- **문서 패킹 전략 (Document Packing Strategy)**:
  - 시 데이터의 특성상 평균 토큰 길이가 100~500 토큰 내외로 짧아, 개별 시마다 패딩(`<pad>`) 토큰을 채울 경우 연산 및 메모리 손실이 극대화된다.
  - 이를 방지하기 위해 데이터 파이프라인에서 여러 편의 시 및 평론 텍스트를 `<시작>`과 `<끝>` 토큰으로 경계를 지어 4,096 크기에 맞게 **Concatenation(이어붙이기)**하여 패킹(Packing)한다.
  - 패킹 시 다른 문서 간의 관심도 전이(Attention Leakage)를 원천 차단하기 위해 **Block-Diagonal Attention** (예: FlashAttention-2의 `flash_attn_varlen_func` 또는 `DataCollatorForCompletionOnlyLM`의 마스킹 기능 활용)을 선택적으로 적용하되, 일반 CPT에서는 단순 연속 토큰 예측으로도 모델이 특수 토큰 경계를 충분히 인지하므로 기본 주의 집중 맵을 사용할 수도 있다. 본 설계에서는 학습 속도 극대화를 위해 Block-Diagonal Attention 적용을 기본 권장한다.

#### 4. 에포크 및 총 학습 스텝 (Epochs & Total Training Steps)
- **학습 에포크 (Training Epochs)**: **2 Epochs** (300M 크기의 DAPT 코퍼스를 2회 순회 학습하여 총 **600M 유효 토큰**을 모델에 주입).
- **총 학습 스텝 수 (Total Steps Calculation)**:
  - 총 처리 토큰: $600,000,000 \text{ tokens}$
  - 유효 배치 토큰: $16,777,216 \text{ tokens/batch}$
  - 총 학습 스텝: $600,000,000 / 16,777,216 \approx 36 \text{ steps}$
  - *비고*: GPU 노드 수나 메모리 제한에 따라 Gradient Accumulation Steps를 64로 낮추는 경우, 총 학습 스텝은 **72 steps**로 세분화되며 유효 배치는 약 8.4M 토큰이 된다. 디버깅 및 손실 모니터링 주기는 매 5스텝마다 수행한다.

### DAPT 종료 기준

```
다음 중 하나 충족 시 DAPT 종료 및 SFT 단계 전환:
  1. 목표 학습 토큰 수 도달 (예: 600M 토큰 처리 완료)
  2. 검증용 데이터셋(Held-out Eval Set) Perplexity가 3회 연속 평가 주기 동안 개선 없음 (patience=3)
  3. 무작위 시드 단어("봄", "슬픔" 등)로 프롬프트 없이 자유 생성(no instruction) 테스트 시, 현대시 규격(행/연 구분 및 특수 토큰 사용)에 부합하는 형태로 자연스럽게 구조화된 시적 문체가 생성되는 경우
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

> 탐색중 — 각 파라미터 권장 범위는 파일럿 실험 결과에 따라 좁혀질 수 있음

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
- **Que et al. (2024) "D-CPT Law: Domain-specific Continual Pre-Training Scaling Law for Large Language Models"**: 도메인 CPT에서 수렴을 결정하는 핵심 변수는 PTPP가 아닌 **도메인:일반 데이터 혼합 비율**임을 스케일링 법칙으로 증명.
- **"Learning Dynamics in Continual Pre-Training for Large Language Models" (2024)**: CPT 학습 중 손실 역학(loss dynamics)이 혼합 비율과 LR 스케줄에 의해 지배됨을 실험적으로 확인.
- **"PTPP-Aware Adaptation Scaling Laws: Predicting Domain-Adaptation Performance at Unseen Pre-Training Budgets" (2024)**: 사전학습 예산(PTPP)이 CPT 성능 예측에 일부 기여하지만, 스타일 도메인과 지식 도메인 간의 성격 차이에 따라 토큰 효율이 크게 달라짐을 보임. 스타일 전이(output distribution shift)는 지식 주입 대비 훨씬 적은 토큰으로 달성 가능.

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

- [TODO] Replay Buffer 후보 데이터셋(모두의말뭉치, CulturaX Korean, AI-Hub) 라이선스 확인 및 파이프라인 연동 우선순위 결정 (명확한 상업적 이용 조항이 없는 데이터셋이나 엄격하게 CC BY-NC-SA가 적용된 데이터셋을 파이프라인에서 어떻게 처리/분리할 것인가?)
  - a. 상업적 이용 조항이 없는 데이터셋을 학습에서 제외할 것인가?
  - b. CC BY-NC-SA 데이터셋은 비상업적 목적의 Phase 1 연구에서만 사용하고 분리 폐기할 것인가?
- [Ph1] Replay Buffer 비율 5% vs 20% 비교 실험 — D-CPT Law에 따르면 도메인:일반 혼합 비율이 CPT 수렴의 핵심 변수. 현재 5% 설정을 20%로 늘렸을 때(총 데이터 ~375M 또는 1.5B 믹스 기준) catastrophic forgetting 방지 효과와 시 도메인 특화 성능 트레이드오프 측정 필요
- [Ph1] Synthetic CPT 유효성 검증 — 프론티어 모델로 원본 코퍼스를 확장한 합성 데이터가 원본 시 코퍼스 단독 CPT 대비 손실 수렴과 이후 SFT 품질에 미치는 효과 측정. 합성 데이터 비율 상한(30%? 45%?) 결정 필요
- [Ph1] Annealing 구간(마지막 10% 스텝) 도입 시 최종 체크포인트의 Perplexity와 시 생성 품질 변화
- [Ph1] 추가되는 특수 토큰(`<행갈이>`, `<연갈이>`)의 초기 임베딩 벡터 학습이 다른 기존 임베딩의 표현 공간을 왜곡하지 않고 신속히 정렬되는지 여부 검증
- [Ph1] CPT 건너뛰기(SFT 직행) vs CPT → SFT 2단계: 스타일 도메인 특성상 소규모 CPT 또는 SFT 단독이 충분한지 Phase 1 파일럿에서 병행 비교 실험
- [Ph2] DAPT 학습 중 Perplexity의 하락이 최종 시 창작 단계에서의 미학적 독창성(Novelty) 및 다양성 지표 향상과 일치하는지에 대한 통계적 상관관계 연구
