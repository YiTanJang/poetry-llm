---
type: Design
title: 자동화 평가 지표
description: 자동 지표는 sanity gate(최소 필터)로만 사용. 핵심 신호는 학습된 리워드 모델 + 외부 LLM 심사 + 인간 평가.
tags: [evaluation, metrics, automation, reward-model, sanity-gate]
timestamp: 2026-06-27T00:00:00Z
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

## 4. 기존 자동 지표의 보조적 활용

N-gram 지표와 임베딩 다양성 지표는 필터 조건에서는 제거하지만, **배치 레벨 다양성 모니터링**에는 유용하다.

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

Distinct-k(배치 내 고유 k-gram 비율)도 동일하게 모니터링 전용.

---

## 5. 구현 구조

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

- [ ] 리워드 모델 학습에 필요한 pairwise 선호도 데이터 최소 수량 결정 — 500쌍 vs 2000쌍.
- [ ] 리워드 모델의 기반 모델 선정 — KR-ELECTRA, KoSimCSE 등 한국어 판별 모델 비교 필요.
- [ ] 외부 LLM 심사 프롬프트 최적화 — 6개 기준 중 어떤 순서로 제시하면 앵커링 편향이 줄어드는가.
- [ ] 배치 다양성 모니터링 임계값 설정 — mean pairwise distance < 얼마일 때 모드 붕괴 경고를 발령하는가.
- [ ] 에코챔버 탐지 프로토콜 구체화 — 생성 모델과 심사 모델의 사전학습 코퍼스 겹침을 어떻게 측정할 것인가.
