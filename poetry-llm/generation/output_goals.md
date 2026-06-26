---
type: Design
title: 산출물 목표 및 Novelty 필터
description: 프로젝트가 추구하는 최종 시의 기준 — novelty, 미학적 아이디어, 새로운 수사와 소재.
tags: [generation, output, novelty, aesthetics]
timestamp: 2026-06-26T00:00:00Z
---

# 산출물 목표 및 Novelty 필터

## 목표: 기존에 없던 시

이 프로젝트의 최종 산출물은 단순히 "그럴듯한 시"가 아니다.
**기존 코퍼스에 없는 새로운 미학적 언어**를 가진 시다.

Novelty는 세 차원에서 정의된다:

### 1. 미학적 아이디어의 novelty
시가 제시하는 세계 해석 방식이 새로운가.
단순히 새로운 주제가 아니라, 세계를 보는 **프레임**이 새로운가.

예:
- 낡은 아이디어: "죽음 앞에서 삶의 의미를 생각한다"
- 새로운 아이디어: "죽음은 소음이고 침묵이 삶이다 — 우리는 소음을 향해 걷는다"

### 2. 수사·표현 운용의 novelty
기존 시에서 사용된 적 없는 방식으로 언어를 배치한다.

예:
- 낡은 수사: 계절을 감정에 직접 대응 ("가을처럼 슬픈")
- 새로운 수사: 계절의 물리적 현상을 인간 심리의 다른 차원에 연결

### 3. 소재의 novelty
시에서 잘 다루지 않는 소재를 시적 언어로 전환.

예:
- 낡은 소재: 자연, 사랑, 이별, 고향
- 새로운 소재: 반도체 에칭 공정, 알고리즘, 물류 창고, 데이터센터 소음

## Novelty 필터 (자동화)

생성된 시를 코퍼스 대비 자동으로 평가:

### 필터 1 — n-gram 중복률
```python
# 생성 시와 전체 코퍼스의 n-gram 겹침 비율
# 4-gram 겹침 < 30% → novelty 통과
novelty_score = 1 - ngram_overlap(generated_poem, corpus, n=4)
```

### 필터 2 — 임베딩 거리
```python
# 생성 시의 임베딩과 코퍼스 내 가장 가까운 시의 거리
# 거리 > threshold → novelty 통과
min_distance = min(cosine_distance(embed(poem), embed(p)) for p in corpus)
```

### 필터 3 — 소재 분류
```python
# 추출된 핵심 소재가 "시적 소재 목록"에 있는 비율
# 비율 낮을수록 novel
known_materials = classify_materials(poem)
novel_ratio = 1 - len(known_materials & common_poetry_materials) / len(known_materials)
```

## 예외: "좋은 새로움" vs "나쁜 새로움"

novelty가 높다고 무조건 좋은 시가 아니다.
의미 없는 난해함, 단순한 비문법, 무작위 조합도 novelty가 높을 수 있다.

따라서 Novelty 필터 통과 이후 **인간 평가** 단계가 필요:
→ [evaluation/aesthetic_quality.md](../evaluation/aesthetic_quality.md)

## 이상적 산출물의 예시적 기술

> 새로운 소재(ex: 배터리 공장 라인)가
> 한국 현대시의 정서 언어(ex: 한, 침묵, 내면화된 고통)와 만나
> 기존에 존재하지 않았던 이미지를 만든다.
> 행갈이는 리듬의 단위이자 의미의 단위이며,
> 창작 노트에서 설계된 의도가 시 텍스트에서 자연스럽게 흘러나온다.
