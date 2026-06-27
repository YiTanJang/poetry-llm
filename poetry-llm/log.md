---
type: Bundle Log
title: 변경 이력
timestamp: 2026-06-26T00:00:00Z
---

# 변경 이력

## 2026-06-27
- Antigravity 에이전트 파일 편집 기능 확인 (테스트)

## 2026-06-26
- 번들 초기 생성
- 프로젝트 개요, 데이터, 전처리, 모델, 생성, 평가, 연구 영역 초안 작성
- git 저장소 초기화 및 에이전트 브랜치 전략 수립

## 2026-06-27 (Antigravity Agent)
- `poetry-llm/data/augmentation.md` 수정: 역번역 메트릭(COMET-QE, PEPR), 모델 선정 기준, 세대 품질 퇴화(Model Collapse) 대책, 파이프라인 워크플로우 추가 및 실제 시인 퇴고 데이터 획득 전략 추가.
- `poetry-llm/data/token_budget.md` 수정: 한국어 토크나이저 효율성 정밀 비교, 도메인 적응 scaling laws 및 1% replay buffer 전략 적용, 8x A100 80GB 하드웨어 VRAM 소모 구조 분석 및 GPU-hours 산출, 단계별 학습 로드맵 및 토큰 임계점 설정.
- `poetry-llm/research/prior_art_failures.md` 수정: 최신 AI 시 창작 실패 패턴(2024-2026), 디코딩 샘플링(T, p)의 확률분포적 관점 한계, 한국어 띄어쓰기/조사 오류 사례 및 시적 리듬 파괴 예시 추가, 음성 상징어 창작 실패 원인 분석, 신체성 및 미학적 결단 부재로 인한 감흥 실패 요인 분석, 대응 전략 테이블 내 검증 방법 컬럼 추가.
- `poetry-llm/research/open_questions_updated.md` 수정: 최신 연구 동향을 반영한 Q1-Q20 연구 질문 상태 갱신, 신규 연구 질문 Q21-Q25(커리큘럼 러닝, 발음 레이어, 계보별 파인튜닝, Few-shot ICL, 양자화 경량화가 창의성에 미치는 영향) 추가, 프로젝트 로드맵(Phase 0/1/2)에 따른 연구 질문 배정 매핑 다이어그램 및 우선순위 종합 업데이트.
- `poetry-llm/evaluation/auto_metrics.md` 수정: BLEU/ROUGE 지표의 미학적 한계 분석, Unseen n-gram fraction 및 NoveltyBench Distinct-k 다양성 지표 공식 도입, CrPO 창의성 선호 최적화 목표 및 평가 모델 에코챔버 탐지 프로토콜 추가, Mecab 및 KR-SBERT 문장 임베딩 기반 구체적 구현 및 가중치 최적화 상관관계 분석 계획 상세화.
- `poetry-llm/evaluation/llm_as_judge.md` 신규 생성: 시 창작 전문 LLM 심사위원(LLM-as-a-Judge) 설계, 다중 에이전트 합의체(Committee) 비평 메커니즘, 6대 미학 기준 점수 루브릭 정의, 자가 선호/위치/길이 편향 제어 및 분포 보정 전략, Weighted Kappa 및 Kendall's Tau 기반 인간 평가 상관관계 합치도 분석 보정 파이프라인 상세 기술.
- `poetry-llm/preprocessing/korean_g2p_alignment.md` 신규 생성 및 `index.md` 수정: Kiwi 형태소 분석기와 g2pk를 활용한 한국어 시 발음 변환(G2P) 파이프라인 설계 및 자소-음소 정렬(Syllable-to-Syllable, Onset-Nucleus-Coda) 스키마 상세 명세 추가, 트랜스포머 모델 주입 전략(인라인 토큰, 듀얼 스트림, 자소 분리) 제안.
- `poetry-llm/evaluation/human_evaluation.md` 수정: 인간 평가 설계 및 가이드라인 보완 (전문가/일반 패널 구체적 요건, 5점 척도 상세 루브릭, Fleiss' Kappa/Weighted Kappa/Krippendorff's Alpha 등 평가자 일치도 통계량 설계, 불일치 조정 프로토콜 추가).
- `poetry-llm/evaluation/index.md` 수정: 인간 평가 가이드라인 제목 수정 및 누락 문서(llm_as_judge.md, evaluation_pipeline.md) 리스트 보완.
- `poetry-llm/data/quality_and_cleaning.md` 신규 생성 및 `index.md` 수정: 데이터 정제 및 품질 필터링 가이드라인 작성 (메타데이터 표준 스키마, OCR 오류 정밀 보정, 띄어쓰기 및 구두점 보존 규칙, Layout/PPL/MinHash 중복 품질 필터, 공정이용 및 Opt-out 저작권 전략 명세).
- `poetry-llm/generation/index.md` 수정: 누락된 5개 문서(`cot_schema.md`, `cot_training_data_design.md`, `harness_design.md`, `multi_agent_council.md`, `thought_process_stages.md`) 등록.
- `poetry-llm/generation/iterative_refinement.md` 수정: 비평-수정 루프 오케스트레이션 설계 상세화, 구체적 비평 기준용 시스템/요청 프롬프트 템플릿 추가, 무한 루프 방지를 위한 4단계 방어 기제(하드 반복 제한, 시맨틱 코사인 유사도 정체 감지, Temperature 감쇠, Fallback Selector) 상세 명세 추가, 신규 미결 사항 업데이트.
- `poetry-llm/model/finetuning_strategy.md` 수정: 배치 사이즈(Effective 16~64, GPU-accumulation 설정), warmup ratio(0.03), token budget(50M~200M), 커리큘럼 단계별 토큰량 상세 추정(총 120M), 조기 종료 기준(early_stopping_patience=3) 및 특수 토큰 임베딩 평균/클론 초기화용 파이썬 스크립트 작성, 미결 사항 갱신.
- `poetry-llm/model/index.md` 수정: `continual_pretraining.md` 및 `training_infrastructure.md` 문서 등록.
- `poetry-llm/preprocessing/training_data_formats.md` 수정: 8가지 학습 데이터 생성 포맷에 대한 개요 및 학습 목적, 표준 JSON 스키마, 특수 토큰과 발음 정보 레이어가 내재된 상세한 한국어 시 예시 추가 및 신규 미결 사항 정리.

