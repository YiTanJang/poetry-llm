---
type: Design
title: 자동화 평가 지표
description: 자동 지표는 sanity gate(최소 필터)로만 사용. 핵심 신호는 학습된 리워드 모델 + 외부 LLM 심사 + 인간 평가.
tags: [evaluation, metrics, automation, reward-model, sanity-gate]
timestamp: 2026-06-28T00:00:00Z
---

# 자동화 평가 지표

## 핵심 원칙: 자동 지표는 Sanity Gate다

자동 지표의 역할은 단 하나다 — **명백히 망가진 출력물을 걸러내는 것**.

자동 지표는 시의 품질을 측정하지 않는다. 시의 품질은 학습된 리워드 모델과 인간이 측정한다.
자동 지표를 품질 지표로 쓰려는 시도는 다음 이유로 실패한다.

---

## 1. N-gram 지표가 시에서 실패하는 이유

N-gram 기반 독창성 지표(Unseen N-gram Fraction, BLEU 등)는 일반 텍스트에서도 한계가 있지만, 시에서는 **원칙적으로 부적합하다.**

**근본적 문제: 의도적 반복은 핵심 시 기법이다**

아나포라(anaphora, 首句 반복), 반복법, 후렴 — 이 기법들은 현대시에서 리듬과 강조를 만드는 핵심 수단이다. 김혜순의 시, 최승자의 시, 이상의 시에서 반복은 결함이 아니라 의도다. N-gram 중복이 높다고 시가 나쁜 것이 아니다.

```
반례:
"나는 모른다 / 나는 모른다 / 나는 다만 / 여기 있다"
→ 4-gram 독창성 점수: 매우 낮음 (반복)
→ 실제 미학적 가치: 의도적 아나포라로 높음
```

**추가 문제:**
- 코퍼스에 없는 무의미한 음절 나열도 "novel n-gram"으로 분류됨 (표층 novelty ≠ 미학적 novelty)
- 짧은 시는 분모가 작아 n-gram 통계가 불안정함
- 한국어 형태소 경계에 따라 n-gram 계산 결과가 크게 달라짐

**결론**: N-gram 독창성 임계값(≥ 0.7 등)은 이 프로젝트에서 **사용하지 않는다.** 소재 novelty 임계값도 동일하게 제거한다.

---

## 2. Sanity Gate: 실제로 사용하는 자동 체크

자동 필터는 3개 조건으로만 구성한다. 이것이 전부다.

### 체크 1: 비어 있지 않음 (Not Empty)

```python
def check_not_empty(poem: str) -> bool:
    """출력이 존재하고 최소한의 내용이 있는가"""
    stripped = poem.strip()
    return len(stripped) > 0
```

### 체크 2: 순수 반복이 아님 (Not Pure Repetition)

의도적 반복(아나포라, 후렴)과 모델 루프(루프 붕괴)를 구분한다.

```python
def check_not_pure_repetition(poem: str, threshold: float = 0.8) -> bool:
    """
    80% 이상의 행이 동일하면 모델 루프로 판정.
    의도적 반복(아나포라)은 100%가 아니면 일반적으로 통과.
    """
    lines = [l.strip() for l in poem.split('\n') if l.strip()]
    if len(lines) <= 1:
        return True
    unique_lines = set(lines)
    unique_ratio = len(unique_lines) / len(lines)
    return unique_ratio > (1 - threshold)  # 80% 이상 중복이면 실패
```

### 체크 3: 길이 범위 (Reasonable Length)

```python
def check_length(poem: str, min_lines: int = 3, max_lines: int = 100) -> bool:
    """
    최소 3행: 2행 이하는 구나 문장, 시로 보기 어려움
    최대 100행: 상한은 느슨하게. 산문시·장시 허용.
    """
    lines = [l.strip() for l in poem.split('\n') if l.strip()]
    return min_lines <= len(lines) <= max_lines
```

### Sanity Gate 통과 기준

```python
def passes_sanity_gate(poem: str) -> bool:
    return (
        check_not_empty(poem) and
        check_not_pure_repetition(poem) and
        check_length(poem)
    )
```

이 세 체크를 통과한 시만 다음 단계(리워드 모델 채점)로 넘어간다.

---

## 3. 실제 품질 신호: 리워드 모델 + 외부 LLM + 인간 평가

Sanity Gate는 Floor(바닥)다. 실제 신호는 다음 계층에서 온다.

### 3.1 학습된 리워드 모델 (Primary Signal)

시 미학 선호도 데이터로 학습된 소형 리워드 모델이 핵심 필터다.

- **학습 방식**: 인간 시인·독자의 선호 판단 데이터(pairwise comparison)로 학습
- **입력**: 시 텍스트 (창작 노트 포함 또는 미포함)
- **출력**: 스칼라 점수 (미학적 선호도)
- **모델 규모**: 1~7B 파라미터 충분 (판별 태스크, 생성 불필요)
- **이점**: 규칙 기반 필터가 놓치는 "뻔하지 않음", "이미지 선명도", "긴장감"을 학습으로 포착 가능

> 가설: 리워드 모델은 파일럿(M1) 이전에 구축되어야 한다. 구축 전까지는 외부 LLM 심사로 대체한다.

### 3.2 외부 LLM 심사 (Secondary Signal)

리워드 모델이 없거나 보완이 필요한 시점에 사용.

```
심사 모델: GPT-4o 또는 Claude Opus (생성 모델과 다른 계열 사용)
프롬프트 구조:
  - 시 텍스트 제공
  - 6개 미학 기준(경제성, 필연성, 긴장, 낯설게하기, 열린 끝, 음악성)으로 각 1~5점 평가
  - 총점과 한 줄 근거 요구
```

주의: LLM 심사는 에코챔버 위험이 있다 — 생성 모델과 심사 모델의 사전학습 코퍼스가 겹칠수록 자기 참조적 고평가가 생긴다. 이 때문에 생성 모델과 다른 계열의 모델을 심사에 사용한다.

### 3.3 인간 평가 (Milestone Signal)

자동·LLM 심사는 인간 평가를 보조한다. 마일스톤(M1, M2, M3)마다 인간 전문가 평가를 진행한다. 세부 설계는 [human_evaluation.md](human_evaluation.md) 참조.

---

## 4. Novelty의 수학적 공식화 및 모니터링 지표

기존 자동 지표(N-gram 중복도 및 임베딩 다양성)는 단일 시의 합격/불합격을 결정하는 필터로 사용하지 않지만, 모델의 다양성을 모니터링하고 배치 레벨의 모드 붕괴(mode collapse)를 감지하기 위해 수학적으로 정의하여 사용한다.

### 4.1 N-gram Overlap (N-gram 중복도 및 독창성)

배치 및 단일 시 수준에서의 N-gram 기반 독창성은 다음과 같이 공식화한다.

#### 1) Distinct-N (배치 내 고유 N-gram 비율)
배치 $B$ 내에서 생성된 모든 시의 N-gram 다양성을 측정하기 위해, 고유 N-gram 수와 전체 N-gram 수의 비율을 계산한다.
$$\text{Distinct-N}(B) = \frac{|\bigcup_{p \in B} G_n(p)|}{\sum_{p \in B} |G_n(p)|}$$
여기서 $G_n(p)$는 시 $p$에 존재하는 모든 $n$-gram의 다중집합(multiset)이며, 분자의 $\bigcup$는 중복을 제거한 고유 $n$-gram들의 합집합이다.

#### 2) Unseen N-gram Fraction (참조 코퍼스 대비 독창성)
생성된 시 $p$가 기존 참조 시집 코퍼스 $C_{\text{ref}}$에 없는 새로운 N-gram을 얼마나 포함하고 있는지 측정한다.
$$\text{Novelty}_n(p) = 1 - \frac{\sum_{g \in G_n(p)} \mathbb{I}(g \in C_{\text{ref}})}{|G_n(p)|}$$
여기서 $\mathbb{I}(\cdot)$는 지시 함수(indicator function)로, 해당 $n$-gram이 참조 코퍼스에 존재하면 1, 존재하지 않으면 0을 반환한다.

### 4.2 Embedding Distance (임베딩 다양성 및 거리)

텍스트의 표층적 중복을 넘어 의미적 다양성을 측정하기 위해 문장/시 수준의 임베딩 모델 $E(\cdot)$ (예: KR-SBERT)을 활용한다.

#### 1) Pairwise Cosine Distance
두 시 $p_i, p_j$ 사이의 의미적 거리는 다음과 같이 코사인 거리로 정의한다.
$$d(p_i, p_j) = 1 - \text{CosineSimilarity}(E(p_i), E(p_j)) = 1 - \frac{E(p_i) \cdot E(p_j)}{\|E(p_i)\|_2 \|E(p_j)\|_2}$$

#### 2) Batch Diversity (배치 내 다양성)
크기 $B$를 가진 생성 배치 내의 평균 및 최소 거리를 모니터링하여 다양성을 추적한다.
$$\text{Mean Pairwise Distance}(B) = \frac{2}{B(B-1)} \sum_{1 \le i < j \le B} d(p_i, p_j)$$
$$\text{Min Pairwise Distance}(B) = \min_{1 \le i < j \le B} d(p_i, p_j)$$
이 값이 특정 임계값 이하로 떨어지면 생성 모델의 다양성 부족 및 반복 생성을 의미하므로, 디코딩 하이퍼파라미터(Temperature, Top-p) 재검토 신호로 사용한다.

#### 3) Embedding-based Novelty
참조 코퍼스 $C_{\text{ref}}$의 시들과의 거리를 기반으로, 생성된 시 $p$의 의미적 참신성을 평가한다.
$$\text{Novelty}_{\text{emb}}(p) = \min_{r \in C_{\text{ref}}} d(p, r)$$
또는 최인접 $k$개 참조 시와의 평균 거리로 확장하여 평가할 수 있다.

```python
# 필터로 쓰지 않음. 모니터링 용도로만.
def batch_diversity_monitor(poems: list[str]) -> dict:
    """
    배치 전체의 다양성을 추적.
    단일 시를 통과/실패 판정하는 데 쓰지 않음.
    """
    embeddings = encode_poems(poems)  # KR-SBERT 등
    pairwise_distances = compute_pairwise_cosine_distances(embeddings)
    return {
        "mean_pairwise_distance": pairwise_distances.mean(),
        "min_pairwise_distance": pairwise_distances.min(),
        # 낮으면 모드 붕괴 징후 — 하이퍼파라미터 재검토 신호
    }
```

Distinct-k(배치 내 고유 k-gram 비율)도 동일하게 모니터링 전용으로 활용한다.

#### 4.3 Mecab 형태소 분석기 구성 및 어휘적 독창성(Lexical Novelty) 계산 규칙

한국어는 교착어적 특성으로 인해 단순 공백(어절) 기준의 N-gram 분석을 수행할 경우 조사, 어미 등 기능어의 접합 상태에 따라 동일 어휘가 서로 다른 토큰으로 취급되는 오류가 발생한다. 이는 Distinct-k 및 Unseen N-gram 분석의 왜곡을 초래하므로, 형태소 분석기 Mecab을 활용하여 텍스트를 형태소 단위로 분절한 뒤 계산한다.

1. **Mecab 형태소 세그먼테이션 규칙**
   - **실질 형태소 중심 필터링**: 조사(J*), 어미(E*), 접사(X*) 등 기능어와 문장 부호(SF, SP 등)의 반복은 시적 운율 형성(아나포라 등)에 기여하지만 어휘적 다양성(Distinct-k) 평가에는 왜곡 요소가 될 수 있다. 따라서 어휘적 독창성 평가 시에는 체언(N*), 용언(V*), 수식언(MM, MAG), 독립언(IC) 등 실질 형태소에 해당하는 품사 태그만 추출하여 토큰 시퀀스를 구성하는 것을 기본 규칙으로 한다.
   - **분절 기본 규칙**:
     - 조사 및 어미 제거: `N*` (명사 계열), `V*` (용언 계열의 어간), `M*` (관형사/부사 계열)만 남김.
     - 용언(동사, 형용사)의 원형(lemma) 추출: 문맥에 따른 활용형 변경으로 인한 중복 계산 방지를 위해 Mecab의 품사 태깅 결과에서 용언(VV, VA)은 기본형으로 복원하여 분석에 사용한다.
     - 복합명사 분해: 복합명사는 Mecab 사전 구성에 따라 하나의 토큰으로 나오거나 분해될 수 있으므로, 복합명사 분해 규칙(`n-gram` 계산 시 단일 토큰으로 처리할지 개별 형태소로 분해할지)을 일관되게 유지한다. (기본 설정: 형태소 단위 완전 분해).

2. **Distinct-k (어휘 다양성) 및 Unseen N-grams 계산 절차**
   - **Distinct-k 계산**:
     - 생성된 시 $p$의 텍스트에 대해 위 규칙에 따라 Mecab 형태소 분석 및 실질 형태소 추출을 수행하여 토큰 시퀀스 $T(p) = [t_1, t_2, \dots, t_m]$를 생성한다.
     - 토큰 시퀀스 $T(p)$ 상에서 슬라이딩 윈도우 방식으로 $k$-gram 집합 $G_k(p)$을 구축한다.
     - 배치 단위 Distinct-k 계산 시, 위에서 구축한 형태소 기반 $k$-gram을 수식에 대입한다.
   - **Unseen N-grams (참조 코퍼스 대비 독창성) 계산**:
     - 참조 코퍼스 $C_{\text{ref}}$에 존재하는 모든 시에 대해 동일한 Mecab 분절 및 실질 형태소 추출 규칙을 적용하여 참조 형태소 $n$-gram 사전 $G_n(C_{\text{ref}})$을 구축한다.
     - 생성된 시 $p$에서 추출된 형태소 $n$-gram 중 $G_n(C_{\text{ref}})$에 존재하지 않는 비율을 계산하여 어휘적 Novelty 점수로 활용한다.

#### 4.4 SBERT 문장 임베딩 모델 구성 및 의미적 독창성(Semantic Novelty) 계산 규칙

표층적인 형태소 중복을 넘어, 시의 문맥적/의미적 수준에서의 독창성을 모니터링하기 위해 문장/텍스트 임베딩 모델인 SBERT를 구성하여 활용한다.

1. **SBERT 모델 및 입력 포맷 구성**
   - **기준 모델**: 한국어 문맥과 시적 표현의 의미 유사도를 정밀하게 보존하기 위해 사전학습된 한국어 전용 SBERT 모델인 `snunlp/KR-SBERT-V1` (또는 성능 검증을 거친 허깅페이스 내 최신 한국어 SBERT 모델 계열)을 사용한다.
   - **입력 텍스트 처리 규칙**:
     - **시 전체 입력 (Global Representation)**: 행갈이 토큰(`<행갈이>`)과 연갈이 토큰(`<연갈이>`)을 공백 또는 특수 구분자로 치환한 후 시 전체 텍스트를 인코딩하여 고정 차원의 벡터 $E(p)$를 추출한다.
     - **풀링(Pooling) 전략**: 문맥의 의미 보존을 위해 기본적으로 Mean Pooling(평균 풀링) 결과를 문장 임베딩 벡터로 채택한다.

2. **의미적 독창성(Semantic Novelty) 결정을 위한 코사인 거리 임계값 규칙**
   - **의미적 거리(Semantic Distance)의 정의**:
     생성된 시 $p$와 참조 코퍼스 $C_{\text{ref}}$ 간의 최소 코사인 거리 $d_{\min}(p) = \min_{r \in C_{\text{ref}}} (1 - \text{CosineSimilarity}(E(p), E(r)))$를 계산한다.
   - **임계값 설정 규칙**:
     - 단일 시의 의미론적 독창성을 판정하기 위한 임계값 $\theta_{\text{semantic}}$은 고정된 상수가 아닌, **참조 코퍼스 내 기존 시들 간의 상호 거리(Self-Pairwise Distance) 분포**를 기준으로 동적으로 도출한다.
     - **동적 임계값 설정 수식**:
       $$\theta_{\text{semantic}} = \mu_{\text{ref}} - k \cdot \sigma_{\text{ref}}$$
       여기서 $\mu_{\text{ref}}$는 참조 코퍼스 내 임의의 시 쌍(pair) 간의 평균 코사인 거리이며, $\sigma_{\text{ref}}$는 표준편차이다. 계수 $k$는 미학적 기준 및 실험을 통해 조정된다 (기본 제안: $k = 1.0$).
     - **초기 실험 가이드라인**: `snunlp/KR-SBERT-V1` 기준, 두 독립적인 한국어 시 텍스트의 코사인 유사도는 분포 상 약 $0.6$ ~ $0.7$ 내외에 집중되며(코사인 거리 $0.3$ ~ $0.4$), 이에 따라 의미적으로 참신함을 확보했다고 간주할 수 있는 초기 임계값 기준은 **코사인 거리 $\theta_{\text{semantic}} \ge 0.35$**로 설정한다. 이 임계값보다 먼 거리에 있는 생성된 시는 기존 참조 코퍼스의 시들과 의미적으로 명확히 구별되는 독창적 시로 판정한다.
     - 이 임계값은 절대적인 필터가 아니며, 배치 단위의 의미 다양성 모니터링 및 Novelty 점수 시각화를 위한 기준으로만 활용된다.

---

## 5. 일반 사물 Novelty (Common-Object Novelty) 및 예외 처리

단순히 사전에 등장하지 않는 '희귀 단어(rare words)'나 '어색한 합성어'를 나열하는 것을 넘어, **누구나 아는 일상적인 사물(예: 꽃, 시계, 돌, 눈)을 지극히 낯선 문맥과 통사 구조로 결합하여 미학적 가치를 창출하는 경우**를 '일반 사물 Novelty'로 정의하고, 기존 평가 지표가 이를 결함으로 오인하지 않도록 예외 처리 규칙을 수립한다.

### 5.1 문제 상황: 기존 지표의 한계
1. **단어 빈도(IDF) 패널티**: '꽃', '시계' 등 흔히 사용되는 단어가 포함되면, 단순 어휘 기반 Novelty 계산 시 평이한 시로 과소평가된다.
2. **정적 단어 임베딩의 한계**: static word embedding 기반의 유사도 비교는 '시계'와 '자라다'의 비정상적 결합을 감지하지 못하고, 개별 단어의 일상성만 반영한다.

### 5.2 예외 처리 및 측정 모델
흔한 어휘들이 결합하여 발생하는 구조적/의미적 novelty를 평가하기 위해 다음과 같은 구성적(compositional) 평가 방식을 설계한다.

#### 1) 통사-의미론적 괴리도 (Syntactic-Semantic Divergence)
개별 단어의 빈도는 높지만, 이들이 문장 구조 내에서 관계를 맺는 결합 확률이 극히 낮음을 수치화한다.
의존 구문 분석(Dependency Parsing) 결과로 도출된 두 단어의 쌍 $(w_d, w_h)$ (의존어와 지배어, 예: '시계가' - '자란다')에 대해:
- 일반 코퍼스 $C_{\text{general}}$에서의 개별 단어 빈도: $P(w_d)$, $P(w_h)$ (매우 높음)
- 일반 코퍼스에서의 결합(의존 관계) 빈도: $P(w_d \to w_h)$ (매우 낮음)
- **통사적 결합 Novelty Score**:
  $$\text{Novelty}_{\text{syntax}}(w_d, w_h) = -\log \frac{P(w_d \to w_h)}{P(w_d)P(w_h)}$$
  이 점수가 임계값 $\theta_{\text{syntax}}$을 초과하는 결합 관계가 시 내에 존재할 경우, 개별 어휘가 흔하더라도 **최종 Novelty 가중치를 상향 조정**하는 예외 처리를 수행한다.

#### 2) 문맥화 임베딩 변이도 (Contextualized Embedding Shift)
트랜스포머 기반 언어모델(KoBERT, KoGPT 등)의 레이어에서 추출한 문맥화된 단어 임베딩 $E_{\text{context}}(w | S)$과 사전학습 코퍼스 전체에서 평균화된 정적 표상 $E_{\text{static}}(w)$ 사이의 편차를 계산한다.
$$\text{Shift}(w | S) = 1 - \text{CosineSimilarity}(E_{\text{context}}(w | S), E_{\text{static}}(w))$$
- 일상적 맥락("시계를 본다"): 문맥 임베딩이 정적 표상과 유사함 ($\text{Shift} \approx 0$)
- 낯설게 하기 맥락("시계를 심는다"): 주변 맥락에 의해 단어의 표상이 꼬여 변이도가 급증함 ($\text{Shift} \gg 0$)
- 시에 등장하는 일반 사물 어휘들의 평균 $\text{Shift}$가 높을수록, 평이한 단어를 극도로 독창적인 이미지로 결합하여 사용했음을 의미한다.

#### 3) 생성 확률 역전 (Likelihood Inversion under Reference LM)
일반 언어모델(예: KoGPT2)과 파인튜닝된 시 생성 모델 사이의 당혹도(Perplexity) 차이를 측정한다.
- 일반적인 문장: 두 모델 모두 높은 확률(낮은 PPL)을 부여.
- 일반 사물 Novelty 문장: 일반 LM은 매우 낮은 확률(높은 PPL)을 주지만, 미학적으로 잘 훈련된 생성 모델/리워드 모델은 합리적인 확률을 부여.

---

## 6. 구현 구조

```
poetry-llm/evaluation/metrics/
├── sanity_gate.py      ← check_not_empty, check_not_pure_repetition, check_length
├── reward_model.py     ← 학습된 리워드 모델 추론 인터페이스
├── llm_judge.py        ← GPT-4o / Claude 외부 심사 프롬프트 래퍼
└── batch_monitor.py    ← 다양성 모니터링 (필터 아님)
```

---

# Citations

1. Zhang et al. (2025). *Measuring LLM Novelty As The Frontier Of Original And High-Quality Output*. arXiv:2504.09389.
2. Zhang et al. (2025). *NoveltyBench: Evaluating Language Models for Humanlike Diversity*. arXiv:2504.05228.
3. Creative Preference Optimization (2025). *Creative Preference Optimization: Aligning LLMs with Creative Metrics*. arXiv:2505.14442. EMNLP 2025.
4. Tang Poetry (2025). *Capabilities and Evaluation Biases of Large Language Models in Classical Chinese Poetry Generation*. arXiv:2510.15313.
5. Reimers & Gurevych (2019). *Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks*. EMNLP 2019.

---

## 미결 사항

- [Ph1] 통사-의미론적 괴리도(Syntactic-Semantic Divergence)를 계산할 때, 의존 구문 분석기(예: Kiwipiepy)의 한국어 시 분석 오차율을 보정할 수 있는 통계적 신뢰 범위 설정법.
- [Ph1] 문맥화 임베딩 변이도(Contextualized Embedding Shift) 측정 시, 시적 낯설게 하기의 양상을 가장 정밀하게 표상하는 트랜스포머 레이어층(중간층 vs 최상위층) 선정 실험 설계.
- [Ph1] 일반 사물 Novelty가 높은 생성 결과를 리워드 모델이 '단순 오작동(비문)'이 아닌 '미학적 낯설게 하기'로 인식하도록 훈련 데이터셋을 라벨링하는 기준 정의.
- [Ph1] Mecab의 품사 태깅 필터링 과정에서 조사와 어미를 완전히 배제하는 것이 종결어미의 독창적 활용 등 시적 형태(Lexical Form)의 독창성 평가에 미치는 왜곡 정도 분석 및 최적 태그 필터 조합 도출.
- [Ph1] `snunlp/KR-SBERT-V1` 외에 최근 한국어 거대 모델 임베딩(예: KoSimCSE 등)과의 성능 비교 실험을 통한 의미론적 코사인 거리 임계값 $\theta_{\text{semantic}}$의 경험적 최적화 및 동적 조정 알고리즘 정밀화.
