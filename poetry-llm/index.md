---
type: Bundle Index
title: LLM 시 생성 프로젝트 — Poetry-LLM
description: 20~40B 언어모델을 풀 파인튜닝하여 미학적으로 유의미한 한국어 시를 생성하는 연구 프로젝트의 지식 번들.
tags: [poetry, llm, fine-tuning, korean, generative-ai]
timestamp: 2026-06-26T00:00:00Z
---

# Poetry-LLM 지식 번들

이 번들은 LLM 시 생성 프로젝트의 아이디어, 설계, 연구를 체계적으로 기록하는 OKF 지식 저장소입니다.
여러 에이전트가 각자의 도메인을 담당하여 병렬로 기여합니다.

## 번들 구조

| 경로 | 내용 |
|------|------|
| [overview/](overview/index.md) | 프로젝트 비전, 목표, 로드맵 |
| [data/](data/index.md) | 학습 데이터 수집 전략 및 구성 |
| [preprocessing/](preprocessing/index.md) | 데이터 전처리 및 토큰화 설계 |
| [model/](model/index.md) | 모델 선택, 파인튜닝 전략, 아키텍처 |
| [generation/](generation/index.md) | 시 생성 파이프라인 및 CoT 설계 |
| [evaluation/](evaluation/index.md) | 품질 평가 지표 및 novelty 측정 |
| [research/](research/index.md) | 관련 연구 및 참고문헌 |

## 에이전트 기여 구조

각 에이전트는 전담 브랜치에서 작업 후 `main`으로 머지합니다.

| 브랜치 | 담당 에이전트 | 영역 |
|--------|-------------|------|
| `agent/data` | 데이터 에이전트 | `data/` |
| `agent/model` | 모델 에이전트 | `model/`, `preprocessing/` |
| `agent/generation` | 생성 에이전트 | `generation/` |
| `agent/evaluation` | 평가 에이전트 | `evaluation/` |
| `agent/research` | 리서치 에이전트 | `research/` |

## 핵심 가설

> 시는 언어의 극한 형식이다. 따라서 시를 잘 생성하는 모델은
> 언어 일반을 깊이 이해한 모델이다.

이 프로젝트는 단순히 "그럴듯한 시"를 출력하는 것이 아니라,
**기존에 없던 미학적 언어**를 발견하는 모델을 목표로 합니다.
