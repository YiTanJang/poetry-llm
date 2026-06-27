---
okf_version: "0.1"
---

# Poetry-LLM 지식 번들

20~40B 언어모델을 풀 파인튜닝하여 **기존에 없는 미학적 언어**를 가진 한국어 시를 생성하는 연구 프로젝트.
단순히 "그럴듯한 시"가 아닌, novelty(새로운 미학 아이디어·수사·소재 조합)를 목표로 한다.

## 섹션

* [overview/](overview/index.md) — 프로젝트 비전, 목표, 로드맵, 협업 방식
* [data/](data/index.md) — 학습 데이터 수집 전략 및 구성
* [preprocessing/](preprocessing/index.md) — 데이터 전처리 및 토큰화 설계
* [model/](model/index.md) — 모델 선택, 파인튜닝 전략, 아키텍처
* [generation/](generation/index.md) — 시 생성 파이프라인 및 CoT 설계
* [evaluation/](evaluation/index.md) — 품질 평가 지표 및 novelty 측정
* [research/](research/index.md) — 관련 연구 및 미답 질문

## 핵심 가설

> 시는 언어의 극한 형식이다. 따라서 시를 잘 생성하는 모델은
> 언어 일반을 깊이 이해한 모델이다.

## 핵심 설계 결정

| 항목 | 결정 |
|------|------|
| 베이스 모델 | 미결 — 현재 검토 중: Gemma 4 27B/31B (→ `model/model_selection.md`) |
| 파인튜닝 | (CPT 선택) → SFT → GRPO 또는 DPO (풀 파인튜닝, LoRA는 파일럿만) (→ `model/finetuning_strategy.md`) |
| 특수 토큰 | `<행갈이>`, `<연갈이>`, `<시작>`, `<끝>` (→ `preprocessing/line_break_tokens.md`) |
| 생성 파이프라인 | 자유형 CoT 스크래치패드 → 3 시인 페르소나 → 하이브리드 토너먼트 심사 → 퇴고 (최대 5회) (→ `generation/index.md`) |
| 자동 필터 | 최소 sanity gate만 (공백 여부, 반복 80%, 3-100행) (→ `generation/output_goals.md`) |
| 평가 | 0.50×인간 + 0.30×리워드 모델 + 0.20×LLM 심사 (→ `evaluation/evaluation_pipeline.md`) |

