---
type: Reference
title: 관련 연구 현황
description: AI 시 생성 및 창의적 텍스트 생성 관련 주요 논문, 프로젝트, 데이터셋.
tags: [research, related-work, papers, nlp, creativity]
timestamp: 2026-06-26T00:00:00Z
---

# 관련 연구 현황

## AI 시 생성 주요 연구

### 대형 언어모델 기반 시 생성
- **GPT 계열의 시 생성 평가**: 범용 LLM이 시를 생성할 수 있으나
  독창성(novelty)과 문화적 특수성 측면에서 한계 확인
- **PoetryBench** (Yi et al., 2023): 영어 시 생성 평가 벤치마크
- **CreativeNLP**: 창의적 텍스트 생성을 위한 평가 프레임워크

### 파인튜닝 기반 접근
- **GPT-2 Poetry Fine-tuning** (Popescu-Belis et al., 2022):
  시집으로 파인튜닝 시 생성 품질 향상 확인
- **Hafez** (Ghazvininejad et al., 2016):
  형식 제약(리듬, 각운)을 강제한 시 생성 시스템

### 한국어 시 생성
- 한국어 특화 시 생성 연구는 현재까지 **매우 부족**
- 주로 GPT 계열 범용 모델 활용 실험 수준
- → 이 프로젝트의 핵심 공백(gap)

## 관련 데이터셋

| 데이터셋 | 언어 | 내용 |
|----------|------|------|
| PoetryFoundation Dataset | EN | 14,000여 편 영어시 |
| McGill-Poetry Dataset | EN | 형식 분석 포함 |
| 구텐베르크 시 코퍼스 | 다국어 | 공공도메인 시 |
| KorLit (비공개) | KO | 한국문학 코퍼스 (접근 제한) |

## Chain-of-Thought in Creative Tasks

- **CoT의 창의적 태스크 적용** (Wei et al., 2022 이후):
  CoT가 수학/논리 외 창의적 작업에도 유효한지 논쟁 중
- **Creative Chain-of-Thought** 관련 연구들:
  창작 노트 방식이 출력 품질을 향상시킨다는 초기 근거 있음

## 관련 프로젝트

### Co-Poet (Google Research)
- LLM이 시인과 협업하는 인터랙티브 시스템
- 시인의 피드백을 실시간 반영

### Verse by Verse (Google Arts)
- 미국 고전시 스타일 모방 시 생성
- → 이 프로젝트와 차별점: 모방이 아닌 novelty 추구

### Bing/ChatGPT 시 생성
- 범용 LLM의 시 생성 한계: 클리셰, 예측 가능성, 문화 편향

## 이 프로젝트의 포지셔닝

| 차원 | 기존 연구 | 이 프로젝트 |
|------|----------|------------|
| 언어 | 주로 영어 | 한국어 특화 |
| 접근 | 범용 모델 프롬프팅 | 도메인 특화 풀 파인튜닝 |
| 목표 | "좋은 시" 생성 | novelty + 미학적 품질 |
| 과정 | 단일 생성 | CoT + 반복 수정 |
| 데이터 | 소규모 | 대규모 (시집 ~2,500권 + 시론) |

## 미결 참고문헌 (찾아볼 것)

- [ ] 한국 현대시 자동 분석 관련 NLP 논문
- [ ] 문학적 창의성(literary creativity) 계산 측정 연구
- [ ] 다국어 미학 전이(multilingual aesthetic transfer) 연구
- [ ] 커리큘럼 학습이 창의적 생성에 미치는 영향
