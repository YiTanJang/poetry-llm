---
type: Design
title: 자동화 평가 지표
description: 생성된 시를 자동으로 평가하는 지표들 — novelty, 구조 정확도, 다양성 및 최신 창의성 평가지표 통합.
tags: [evaluation, metrics, automation, novelty]
timestamp: 2026-06-27T00:00:00Z
---

# 자동화 평가 지표

## 1. 기존 평가 지표의 한계 (BLEU/ROUGE의 부적합성)

일반적인 텍스트 요약이나 기계 번역에서 사용되는 **BLEU(Bilingual Evaluation Understudy)**와 **ROUGE(Recall-Oriented Understudy for Gisting Evaluation)**는 시 생성 모델 평가에 전혀 부합하지 않는다.

1. **미학적 일탈 감점의 오류**:
   BLEU/ROUGE는 참조(Reference) 텍스트와 N-gram이 일치하는 비율을 측정한다. 그러나 현대 시 창작의 핵심 가치는 모방이 아니라 **일탈(Deviation)**과 **낯설게 하기(Defamiliarization)**에 있다. 인간 시인처럼 참신한 비유를 만들어 낼수록 기존 시 텍스트와의 N-gram 일치도가 떨어져 BLEU 점수가 낮게 측정되는 기만적 역설이 발생한다.
2. **표층적 독창성(Surface Novelty) vs 의미론적 독창성(Semantic Novelty)의 괴리**:
   자모 결합 오류나 단순 오타, 무의미한 자음 나열도 기존 코퍼스에 없으므로 표층적으로는 'unseen n-gram'으로 분류되어 독창성 점수를 올린다. 그러나 이는 의미론적 맥락 속에서 작동하는 시적 미학(Semantic Novelty)이 아니다. 자동화 지표는 형태소 및 임베딩 분석을 결합하여 이 둘을 철저히 구분해야 한다.

---

## 2. Novelty 지표

### 2.1 n-gram 독창성 (N-gram Originality) 및 Unseen N-gram Fraction
기존 코퍼스 내 텍스트와 겹치지 않는 새로운 단어 조합의 비율을 측정한다.

```
novelty_ngram = 1 - max_overlap(시, 코퍼스, n=4)
범위: [0, 1] — 1에 가까울수록 novel
기준값: ≥ 0.7 통과
```

- **Unseen N-gram Fraction (arXiv 2504.09389)**:
  생성된 시 집합 $G$ 내에서 기존 훈련 코퍼스 $C$에 단 한 번도 등장하지 않은 고유한 $n$-gram의 비율을 측정한다.
  \[\text{Unseen } n\text{-gram Ratio} = \frac{|T_n(G) \setminus T_n(C)|}{|T_n(G)|}\]
  - $T_n(X)$는 텍스트 집합 $X$ 내의 모든 고유한 $n$-gram 토큰의 집합.
  - 이 프로젝트에서는 $n=3$ 및 $n=4$에 가중치를 두어 측정한다.

### 2.2 임베딩 다양성 (Embedding Diversity)
```
문장 임베딩 모델(ex: KR-SBERT)로 생성시 전체를 벡터화
코퍼스 내 가장 유사한 시와의 코사인 거리 측정
기준값: 거리 ≥ 0.3 통과
```

### 2.3 소재 novelty
```
핵심 명사/이미지 추출 → "시적 소재 빈도 사전"과 비교
시적 소재 사전: 기존 코퍼스에서 빈도 상위 명사 1000개
점수: 1 - (시의 핵심 소재 중 사전에 있는 비율)
```

### 2.4 NoveltyBench Distinct-k Metric
- **Distinct-k (arXiv 2504.05228)**:
  단일 생성 결과물 내가 아닌, 여러 차례 생성된 시 집합(Batch) 내부에서 어휘 및 표현의 다양성을 측정한다.
  \[\text{Distinct-}k = \frac{\text{고유한 } k\text{-gram 개수}}{\text{생성된 총 } k\text{-gram 개수}}\]
  - **목적**: 모델이 특정 프롬프트에 대해 고정된 템플릿의 문장(모드 붕괴)을 생성하는 현상을 포착하고, 생성 다양성을 수치화한다. ($k \in \{2, 3\}$ 권장)

---

## 3. 구조 정확도 지표

### 3.1 행갈이 논리 점수
행갈이 위치가 의미 단위(어절, 구, 절)와 일치하는지 측정한다.
```
형태소 분석 기반으로 행갈이가 의미 경계(조사 결합부, 용언 어미 경계)에 있는지 측정
점수: 의미 경계에서의 행갈이 / 전체 행갈이
기준값: ≥ 0.6
```

### 3.2 연갈이 일관성
각 연(stanza)이 하나의 의미 단위를 이루는지 평가한다.
- 연 내 주제 일관성 (TF-IDF 및 문장 임베딩 코사인 유사도로 측정)

### 3.3 길이 분포 적합성
한국 현대시 평균 행 길이, 연 수 대비 생성시가 너무 벗어나는지 확인한다.

---

## 4. 언어 품질 지표

### 4.1 문법성 (Grammaticality)
```
형태소 분석기로 비문법적 토큰 및 조사 결합 오류 비율 측정
시어의 특성상 의도적 비문법(시적 허용)을 감안하여 감점 가중치는 0.1 이하로 매우 낮게 설정
```

### 4.2 반복 지수 (Repetition Index)
의도치 않은 웅얼거림이나 무한 루프 구문 탐지:
```
동일 n-gram이 같은 시 내에서 반복 등장하는 비율
낮을수록 좋음 (단, 시적 병치나 반복법의 사용은 형태소 분석을 통해 예외 처리 적용)
```

---

## 5. 생성 다양성 및 정렬 최적화 지표

### 5.1 배치 다양성 (Batch Diversity)
```
batch_diversity = 평균 pairwise 코사인 거리
높을수록 좋음 — 모델이 매 실행마다 상이한 시적 비유를 생산함을 보장
```

### 5.2 CrPO (Creative Preference Optimization) 정렬 지표
- **CrPO 목적 함수 반영 (arXiv 2505.14442)**:
  선호 정렬(DPO) 단계에서 시의 novelty와 surprise 요소를 직접 보상으로 부여한다.
  - **Surprise Metric**: 생성된 각 토큰의 정보량(Information Content)인 $-\log P(w_t \mid w_{<t})$의 평균치(Perplexity)를 측정하되, 문법적 한계 내에서의 의도적인 '중간 수준의 일탈'을 최적 지점으로 설정한다.
  - 지나치게 낮은 PPL(클리셰)과 지나치게 높은 PPL(완전 무작위 난해함)의 사잇값을 목표로 정렬한다.

### 5.3 평가 에코챔버 효과 탐지 (Echo Chamber Detection)
- **에코챔버 탐지 프로토콜 (arXiv 2510.15313)**:
  평가용 임베딩 모델과 생성 모델의 사전학습 말뭉치가 유사할 때 발생하는 평가 점수의 인위적 상승(자기 참조적 고평가)을 탐지한다.
  - **탐지 방법**: 평가 지표 산출 시 `KR-SBERT` 외에도 다국어 범용 모델(`Cohere Multilingual`) 및 다른 아키텍처 모델의 임베딩을 이용한 교차 검증 점수의 편차를 확인하여 편차가 $\ge 0.15$ 이상일 시 지표 왜곡 경보를 발령한다.

---

## 6. 구체적인 구현 세부사항

자동 평가 스크립트는 Python 환경에서 아래 라이브러리 조합을 이용해 구동된다.

1. **형태소 분석기 및 토크나이저**:
   - `KoNLPy`의 `Mecab` 분석기 활용 (빠른 속도 및 현대시 특유의 신조어 형태소 분리 정밀도 확보).
2. **임베딩 모델**:
   - 서울대학교의 `snunlp/KR-SBERT-V4` (한국어 문맥 및 시적 정서적 유사성 판별에 특화).
3. **구현 패키지 설계**:
   - `poetry-llm/evaluation/metrics/` 모듈 내에 `originality.py`, `structure.py`, `diversity.py`로 나누어 구현하고, CLI 형태로 개별 생성물의 자동 측정이 가능하도록 래핑한다.

---

## 7. 가중치 결정 및 지표 간 상관관계 분석 계획

자동화 지표가 인간 문학 전문가의 정성적 미학 평가와 정렬(Alignment)되도록 하기 위한 실험 설계를 수립한다.

1. **파일럿 평가 데이터 구축**:
   - 모델이 생성한 시 200편에 대해 전문가(시인, 평론가)의 6대 미학 기준(Aesthetic Criteria) 평점 획득.
2. **지표 간 상관관계 분석 (Correlation Analysis)**:
   - 각 자동화 개별 지표(Unseen N-gram, Distinct-k 등)와 전문가 종합 평점 간의 **Pearson $r$** 및 **Spearman $\rho$** 상관계수를 계산한다.
3. **가중치 미세조정 (Weight Optimization)**:
   - 인간 평가 점수와의 잔차 제곱합(RSS)을 최소화하도록 그리드 서치(Grid Search)를 통해 종합 점수의 가중치를 재조정한다.

---

## 종합 점수 (Composite Score)

현재 설정된 기본 종합 미학 평점 수식은 다음과 같다:

\[\text{Score} = 0.35 \cdot \text{Novelty}_{\text{ngram}} + 0.15 \cdot \text{Embedding}_{\text{diversity}} + 0.15 \cdot \text{Distinct\_k} + 0.15 \cdot \text{Structure}_{\text{accuracy}} + 0.10 \cdot \text{Material}_{\text{novelty}} + 0.10 \cdot \text{Batch}_{\text{diversity}}\]

> 이 가중치는 파일럿 인간 평가 결과와의 상관 분석 이후 최적화 알고리즘에 의해 자동 업데이트된다.

---

## 한계

자동 지표는 **필요 조건**이지 **충분 조건**이 아니다.
novelty 점수가 높아도 미학적으로 실패한 시일 수 있다.
최종 평가는 [human_evaluation.md](file:///d:/Documents/HomeLab/OKFPoetryproject/poetry-llm/evaluation/human_evaluation.md)를 참고하여 이루어진다.

---

# Citations

1. Zhang et al. (2025). *Measuring LLM Novelty As The Frontier Of Original And High-Quality Output*. arXiv:2504.09389.
2. Zhang et al. (2025). *NoveltyBench: Evaluating Language Models for Humanlike Diversity*. arXiv:2504.05228.
3. Creative Preference Optimization (2025). *Creative Preference Optimization: Aligning LLMs with Creative Metrics*. arXiv:2505.14442. EMNLP 2025.
4. Tang Poetry (2025). *Capabilities and Evaluation Biases of Large Language Models in Classical Chinese Poetry Generation*. arXiv:2510.15313.
5. Reimers & Gurevych (2019). *Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks*. EMNLP 2019. (SBERT 기반 구조)

---

## 미결 사항

- [ ] `Mecab` 형태소 분석 사전에 시집 코퍼스에서 추출한 미등록 시어 및 조어 20,000건을 추가하는 정규식 사전 확장 스크립트 개발.
- [ ] 문장 단위가 아닌 연 단위의 미학적 유사도를 정밀하게 측정하기 위한 Multi-vector Embedding Pooling 기법 구현.
- [ ] 생성 다양성을 보장하기 위해 Distinct-2 수치가 0.5 미만으로 떨어지는 에이전트에 대한 강제 페널티 필터 코드 작성.
