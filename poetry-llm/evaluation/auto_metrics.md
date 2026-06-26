---
type: Design
title: 자동화 평가 지표
description: 생성된 시를 자동으로 평가하는 지표들 — novelty, 구조 정확도, 다양성.
tags: [evaluation, metrics, automation, novelty]
timestamp: 2026-06-26T00:00:00Z
---

# 자동화 평가 지표

## 1. Novelty 지표

### 1.1 n-gram 독창성 (N-gram Originality)
```
novelty_ngram = 1 - max_overlap(시, 코퍼스, n=4)
범위: [0, 1] — 1에 가까울수록 novel
기준값: ≥ 0.7 통과
```

### 1.2 임베딩 다양성 (Embedding Diversity)
```
문장 임베딩 모델 (ex: KoSimCSE, KoBERT)로 시 전체를 벡터화
코퍼스 내 가장 유사한 시와의 코사인 거리 측정
기준값: 거리 ≥ 0.3 통과
```

### 1.3 소재 novelty
```
핵심 명사/이미지 추출 → "시적 소재 빈도 사전"과 비교
시적 소재 사전: 기존 코퍼스에서 빈도 상위 명사 1000개
점수: 1 - (시의 핵심 소재 중 사전에 있는 비율)
```

## 2. 구조 정확도 지표

### 2.1 행갈이 논리 점수
행갈이 위치가 의미 단위(어절, 구, 절)와 일치하는지 측정.
```
형태소 분석 기반으로 행갈이가 의미 경계에 있는지 측정
점수: 의미 경계에서의 행갈이 / 전체 행갈이
기준값: ≥ 0.6
```

### 2.2 연갈이 일관성
각 연(stanza)이 하나의 의미 단위를 이루는지.
- 연 내 주제 일관성 (TF-IDF 코사인 유사도로 측정)

### 2.3 길이 분포 적합성
한국 현대시 평균 행 길이, 연 수 대비 생성시가 너무 벗어나는지.

## 3. 언어 품질 지표

### 3.1 문법성 (Grammaticality)
```
형태소 분석기로 비문법적 토큰 비율 측정
시는 의도적 비문법을 사용할 수 있으므로 낮은 가중치 부여
```

### 3.2 반복 지수 (Repetition Index)
의도치 않은 반복 구문 탐지:
```
동일 n-gram이 같은 시 내에서 반복 등장 비율
낮을수록 좋음 (단, 의도적 반복은 예외)
```

## 4. 다양성 지표

배치(batch) 수준에서 생성된 여러 시들의 다양성:
```
batch_diversity = 평균 pairwise 코사인 거리
높을수록 좋음 — 같은 시만 반복하지 않음을 보장
```

## 종합 점수 (Composite Score)

```
score = 0.4 * novelty_ngram
      + 0.2 * embedding_diversity
      + 0.2 * structure_accuracy
      + 0.1 * material_novelty
      + 0.1 * batch_diversity
```

> 이 가중치는 파일럿 인간 평가 결과와의 상관 분석 후 조정.

## 한계

자동 지표는 **필요 조건**이지 **충분 조건**이 아니다.
novelty 점수가 높아도 미학적으로 실패한 시일 수 있다.
최종 판단은 [human_evaluation.md](human_evaluation.md) 참조.
