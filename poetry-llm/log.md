---
type: Bundle Log
title: 변경 이력
timestamp: 2026-06-26T00:00:00Z
---

# 변경 이력

## 2026-06-28 (CPT 스케일링 법칙 업데이트 — PTPP 반박, D-CPT Law 인용, Synthetic CPT, Replay Buffer 실험)
- `poetry-llm/model/continual_pretraining.md`:
  - **PTPP 오용 주의 섹션 신설**: PTPP(Tokens-Per-Parameter)가 스크래치 프리트레이닝 지표이며 CPT에 부적합함을 명시. 외부 리뷰어의 "300M 토큰 부족" 주장에 대한 위키 내 반박 근거 확립.
  - **실증적 근거 섹션 확장**: D-CPT Law (Que et al. 2024), Learning Dynamics in Continual Pre-Training (2024), PTPP-Aware Adaptation Scaling Laws (2024) 3편 인용 추가. CPT 수렴 핵심 변수(혼합 비율, LR 스케줄)를 스케일링 법칙으로 뒷받침.
  - **Synthetic CPT 전략 명시**: 데이터 부족 대응으로 프론티어 모델 기반 합성 데이터 확장(요약·해석·비평·패러프레이즈) 방법을 CPT 건너뛰기 옵션 항목에 추가.
  - **미결 사항 갱신**: [Ph1] Replay Buffer 5% vs 20% 비교 파일럿, [Ph1] Synthetic CPT 유효성 검증 추가.
- `poetry-llm/data/token_budget.md`:
  - **미결 사항 갱신**: [Ph1] Replay Buffer 비율 5% vs 20% 비교 파일럿 추가 (D-CPT Law 혼합 비율 권장과 현재 설계 트레이드오프 명시).

## 2026-06-28 (파이프라인 블로커 17-point 분석 — open_questions 및 미결 사항 추가)
- `poetry-llm/research/open_questions.md`: Q29~Q35 신규 추가. 상태 요약 미답 20 → 27. Phase 0/1/2/3 단계 매핑 테이블 갱신.
  - Q29 (CoT 포맷 통일 Showstopper), Q30 (리워드 모델 콜드스타트 역설), Q31 (GRPO 파이프라인 미구축 Showstopper)
  - Q32 (평가 루브릭 3파일 비정합 Showstopper), Q33 (역추론 CoT 인과 역전)
  - Q34 (GRPO 페르소나 모드 붕괴), Q35 (심사위원-생성기 공진화 드리프트)
- `poetry-llm/model/finetuning_strategy.md`: G=8 vs 3 페르소나 산술 갭 해소 방안 [TODO], GRPO 모드 붕괴 방지 다양성 보너스 [Ph2] 미결 사항 추가.
- `poetry-llm/generation/cot_training_data_design.md`: 역추론 CoT 인과 역전 비율 관리 [Ph1], 보상 모델 필터 순환 의존 탈출 전략 [Ph1] 미결 사항 추가.
- `poetry-llm/evaluation/llm_as_judge.md`: 심사위원-생성기 공진화 드리프트 재보정 프로토콜 [Ph2], 필연성 루브릭 자동 측정 불가 proxy 탐색 [Ph1] 미결 사항 추가.
- `poetry-llm/data/token_budget.md`: 실제 가용 데이터 상한 재산정 [TODO], 합성 데이터 상한 모순 해소 [TODO], 발음 토큰 오버헤드 반영 여부 확인 [TODO] 미결 사항 추가.
- `poetry-llm/data/acquisition.md`: 저작권 조항 내부 불일치(제35조의5 vs 제35조의3) 해소 [TODO] 미결 사항 추가.

## 2026-06-28 (SOTA 정렬 조율 및 토크나이저 검증 프로토콜 반영)
- `poetry-llm/data/augmentation.md`: 역번역(Back-translation)을 시 본문 증강에서 전면 배제하고, 구문 분석(Dependency Parser)의 시적 문법 파괴에 따른 스타일 보존적 섭동의 취약점을 명시하며 퇴고 이력 합성(Trace Augmentation)을 최우선순위로 설정.
- `poetry-llm/model/finetuning_strategy.md`: SFT 커리큘럼을 5단계로 개편. SFT Stage 1에 특수 토큰과 제어 포맷을 완전히 배제한 '순수 시 언어 감각 학습' 단계를 신설하여 포맷 과적합 방지.
- `poetry-llm/evaluation/llm_as_judge.md`: 절대 채점 방식을 배제하고 예선(Qwen2.5-7B-Inst / Swiss-system)과 본선(Claude 3.5 Sonnet / Pairwise 3쌍 비교)으로 나뉜 하이브리드 토너먼트 아키텍처 및 마니페스토 프롬프팅 도입. 워드 샐러드 차단 규칙 명시.
- `poetry-llm/model/model_selection.md` & `poetry-llm/model/continual_pretraining.md`: DAPT/CPT의 시작점으로 '사전 학습 Base 모델'을 채택하도록 명시하여 Instruct 모델의 정렬 가중치 붕괴 방지 및 Replay Buffer 최소화 전략 확정.
- `poetry-llm/preprocessing/line_break_tokens.md`: 한글 특수 토큰의 안정성과 공백 병합(Prefix Space Merging) 및 BPE 바이트 분절 오작동을 검증하는 파이썬 테스트용 '토크나이저 특수 토큰 병합 검증 프로토콜' 신설.
- `poetry-llm/overview/vision.md`: 환각(Hallucination) 문헌 인용(Yi, S., 2023)을 검증된 백낙청(2009)의 "현대시와 근대성, 그리고 대중의 삶" 실제 비평 논문으로 정정 반영.

## 2026-06-28 (Antigravity Agent - 30 Wiki Improvements)
- `poetry-llm/overview/goals.md`: Added ## 미결 사항 (other arts, prose, copyright opt-out).
- `poetry-llm/overview/roadmap.md`: Updated roadmap open questions (persona, multilingual crossover).
- `poetry-llm/overview/agent_collaboration.md`: Documented Claude Code and Antigravity collaboration roles and file lock signals.
- `poetry-llm/data/korean_contemporary.md`: Expanded OCR dictionary rules for archaic Hangul.
- `poetry-llm/data/korean_classical.md`: Detailed public domain copyright deadlines (1956, 1962 cutoff) and compilation/translation rights.
- `poetry-llm/data/poetry_criticism.md`: Detailed SFT Stage 3 dialogue conversions for raw criticism.
- `poetry-llm/data/foreign_poetry.md`: Detailed bilingual alignment formats and cross-lingual sentiment mapping.
- `poetry-llm/data/acquisition.md`: Detailed collaborative royalty schemes and secure sandboxes.
- `poetry-llm/data/quality_and_cleaning.md`: Specified PPL thresholding rules and poetic spacing preservation.
- `poetry-llm/data/token_budget.md`: Refined replay buffer ratios (1% to 5%) and token blending schedules.
- `poetry-llm/preprocessing/line_break_tokens.md`: Detailed tokenizer parsing and decoding mechanics for special line tokens.
- `poetry-llm/preprocessing/pronunciation.md`: Documented phonetic representation constraints for loanwords and mimetic words.
- `poetry-llm/preprocessing/korean_g2p_alignment.md`: Detailed fallback strategies for G2P alignment failures.
- `poetry-llm/model/model_selection.md`: Compared Qwen2.5-32B vs EXAONE-3.5-32B tokenization fertilities and representation capacity.
- `poetry-llm/model/finetuning_strategy.md`: Specified learning rate cosine decay and checkpoint saving frequencies.
- `poetry-llm/model/special_tokens.md`: Defined average embedding copying rules and 128-alignment resizing.
- `poetry-llm/model/continual_pretraining.md`: Documented cosine annealing schedules, warmup, and data blends for DAPT.
- `poetry-llm/model/training_infrastructure.md`: Specified memory profiling for 2x A100 80GB setup and prefetch policies.
- `poetry-llm/generation/cot_schema.md`: Detailed step compression guidelines (Lite/Medium/Full) and context-based triggering.
- `poetry-llm/generation/thought_process_stages.md`: Defined automated semantic evaluation of CoT trace fidelity.
- `poetry-llm/generation/cot_training_data_design.md`: Elaborated on poetic hesitation and self-corrections in synthetic CoT data.
- `poetry-llm/generation/multi_agent_council.md`: Elaborate on distinct persona prompts and role contamination defense.
- `poetry-llm/generation/iterative_refinement.md`: Defined refinement loop termination criteria (oscillation, repetition, stagnation, max limit).
- `poetry-llm/evaluation/evaluation_pipeline.md`: Specified optimal chosen/rejected margin threshold calculations ($\Delta \ge 0.15$) for DPO data.
- `poetry-llm/evaluation/auto_metrics.md`: Documented Mecab and SBERT configuration for lexical and semantic novelty.
- `poetry-llm/evaluation/human_evaluation.md`: Defined Krippendorff's Alpha scoring protocol for expert panels and low-agreement mitigation.
- `poetry-llm/evaluation/aesthetic_quality.md`: Detailed 1-5 scale aesthetic rubrics across 6 dimensions with Korean examples.
- `poetry-llm/research/related_work.md`: Documented research lessons from COIG-Writer (process supervision) and CreativeBench (novelty steering).
- `poetry-llm/research/open_questions.md`: Mapped open questions Q1-Q25 to validation phases and milestones.
- `poetry-llm/research/prior_art_failures.md`: Refined failure-mitigation matrix to address cliché generation and structural rhythmic breakages.

## 2026-06-28 (선호 정렬 전략 GRPO 전환, Best-of-N + PRM 추가)
- `poetry-llm/model/finetuning_strategy.md`: 파이프라인을 `DPO` → `GRPO (권장) / DPO (대안)`으로 개편. GRPO 알고리즘(G=8 그룹 샘플링, Reference model 불필요, 그룹 상대 보상) 및 데이터 포맷 신설. DPO를 오프라인 대안으로 재위치. 미결 사항에 GRPO vs DPO 비교 실험, G 최적값, β 탐색 항목 추가 [Ph2].
- `poetry-llm/generation/output_goals.md`: "Best-of-N 샘플링 + PRM" 섹션 신설. 추론 시점 품질 선택 파이프라인, step-level PRM 보상 설계 방향, GRPO와의 역할 분담 명시. 미결 사항에 N 최적값, PRM 레이블 수집 방법론, Best-of-N vs GRPO 비용 비교 항목 추가 [Ph2].

## 2026-06-28 (DAPT 스케일링 법칙 보완, G2P CoT 격리 원칙 명시, 음향 모델 감시 조건 추가)
- `poetry-llm/model/continual_pretraining.md`: "스타일 도메인 vs 지식 도메인 DAPT" 섹션 추가. 의료/코드 DAPT(10B~500B 토큰)와 스타일 적응(수백M으로도 효과 가능)의 차이 명시. CPT 건너뛰기(SFT 직행) 옵션 및 Phase 1 비교 실험 필요성 기술. 미결 사항에 [Ph1] 항목 추가.
- `poetry-llm/model/model_selection.md`: 미결 사항에 음향 내재화 모델 등장 시 재검토 조건 추가 [TODO]. Qwen2-Audio(7B)/LLaMA-3-Omni(8B) 스케일 부적합, Gemini/GPT-4o 클로즈드 현황 기술. CoT scratchpad 방식이 현실적 대안임을 명시.
- `poetry-llm/preprocessing/korean_g2p_alignment.md`: 섹션 6 "G2P 처리 원칙: CoT 스크래치패드 내부 격리" 신설. 최종 시 출력에 G2P 태그 제거 원칙, Hangul phonetic transparency 특성 기술. 인라인 태그 OOD 문제 명시.
- `poetry-llm/generation/cot_schema.md`: 스크래치패드 패턴 목록에 음운 계획 메모 추가. "클린 출력 원칙(Clean Output Principle)" 섹션 신설 — `[시]` 블록은 클린 한국어 텍스트만. 미결 사항에 음운 계획 포함 CoT vs 미포함 CoT 비교 실험 항목 추가 [Ph1].

## 2026-06-27 (편집 상태 마커 전 문서 적용)
- 전체 wiki 파일에 `[Ph1]`/`[Ph2]`/`[TODO]` 마커를 `## 미결 사항` 항목에 적용 완료.
- `> 탐색중` 마커를 열린 목록 섹션에 추가 (training_data_formats.md 등).
- 적용 범위: preprocessing/ (4), overview/ (2), generation/ (9), evaluation/ (5), research/ (3), data/ (9) — 총 32개 파일.
- `problems.md`: OKF 구조 없음, 마커 적용 대상 외.

## 2026-06-27 (설계 정렬 — CoT·하네스·아키텍처 수정)
- `poetry-llm/data/acquisition.md`: DRM 관련 언급 제거 (프라이버시 수정).
- `poetry-llm/generation/cot_schema.md`: 에이전트가 재도입한 8단계 창작 사고 과정 섹션 전체 제거. 스크래치패드는 자유형이며 단계를 강제하지 않는 원칙 복원. 미결 사항을 자유형 설계 관련 질문으로 교체.
- `poetry-llm/generation/multi_agent_council.md`: 시인 페르소나 이름 수정 (전통주의자/모더니스트/아방가르드 → 감각형/개념형/형식실험형). 비평가/독자 에이전트를 Phase 1 ablation 실험 대상으로 이동. 8단계 언급 제거. 판정 기준 프롬프트 페르소나명 수정.
- `poetry-llm/generation/harness_design.md`: 전면 재작성. 300줄 Python 구현 제거, 개념 수준으로 정리. 창작 도구 하네스(random_words, news_snippet, dice, syllable_count, creative_note) 설계 추가. 인간 개입 지점 정의. Phase 1 실험 대상 명시.

## 2026-06-27 (Antigravity Agent Wiki Improvements)
- `poetry-llm/data/korean_contemporary.md`: Added OCR quality assurance pipeline descriptions and Python pseudocode, typography preservation strategies (spacing preserver, layout tag schemas), and 3 open questions.
- `poetry-llm/data/korean_classical.md`: Added a comprehensive legal and editing guide for public domain poetry (1956 cutoff, compilation copyrights, translation rights) and 3 open questions.
- `poetry-llm/data/poetry_criticism.md`: Formulated mixing ratios (CPT vs SFT), training stages assigning guidelines, and Translation QA guidelines for foreign poetics, and 3 open questions.
- `poetry-llm/data/foreign_poetry.md`: Defined parallel/interleaving translation formats and cross-lingual aesthetic transfer strategies (Western modernism, Haiku, Misty Poetry, Dinggedicht), and 3 open questions.
- `poetry-llm/data/other_arts.md`: Detailed translation methods for non-linguistic arts (Visual, Auditory, Audiovisual) into poetry meta-prompts with concrete Korean cultural examples, and 3 open questions.
- `poetry-llm/data/acquisition.md`: Outlined web-scraping script architecture (asynchronous scrapers with Python code), fair-use legal arguments, and publisher collaboration sandbox/royalty models, and 3 open questions.
- `poetry-llm/preprocessing/line_break_tokens.md`: Defined structural rules for enjambment (`<행갈이:걸침>`) and spacing (`<여백:N>`) tokens, and updated code blocks to support enjambment/spacing, and 3 open questions.
- `poetry-llm/preprocessing/pronunciation.md`: Specified pronunciation representation levels (KSP, IPA, Grapheme) and cross-lingual phonetic transfer rules, and 3 open questions.
- `poetry-llm/preprocessing/korean_g2p_alignment.md`: Designed Kiwi + g2pk alignment pipeline, G2P dictionary schema, and Python helper class `KoreanG2PAligner` for Hangul decomposition, and 3 open questions.
- `poetry-llm/preprocessing/training_data_formats.md`: Added detailed SFT vs CPT tokenization schemas and SFT JSON-L dataset records for all 7 reorganized formats, and 3 open questions.
- `poetry-llm/model/model_selection.md`: Compared Qwen2.5-32B, LLaMA-3-70B, solar-10.7B models across efficiency, size, and licenses, and 3 open questions.
- `poetry-llm/model/finetuning_strategy.md`: Detailed SFT stage learning rate, warm restarts, optimizer resets across curriculum stages 1-4, and 3 open questions.
- `poetry-llm/model/special_tokens.md`: Verified Qwen2.5 newline tokenization (`\n` as 198) and added full HuggingFace model/tokenizer initialization script with 128-alignment resizing and average embedding copying, and 3 open questions.
- `poetry-llm/model/continual_pretraining.md`: Expanded DAPT training recipe with corpus blend (300M token budget), sequence packing, sequence lengths, learning rates, epochs, and 3 open questions.
- `poetry-llm/model/training_infrastructure.md`: Wrote HuggingFace Accelerate FSDP YAML and DeepSpeed ZeRO-3 JSON configuration templates, detailed VRAM OOM analysis, and cloud GPU cost/speed simulations, and 3 open questions.
- `poetry-llm/generation/cot_schema.md`: Refined the 8-step thought process stages, and defined 3 levels of step compression (Lite, Medium, Full) based on resource and length limits, and 3 open questions.
- `poetry-llm/generation/thought_process_stages.md`: Defined automated metrics (embeddings similarity, keyword recall, structure) and role-contamination penalty multipliers for CoT traceability verification, and 3 open questions.
- `poetry-llm/generation/cot_training_data_design.md`: Elaborated the synthetic CoT reverse-inference pipeline and bias mitigation tactics (artificial hesitation, correction injections, asymmetric 2-pass prompting), and 3 open questions.
- `poetry-llm/generation/multi_agent_council.md`: Detailed prompts for 3 Poet Personas + Critic + Reader roles, role contamination defense mechanisms (DEC scratchpads, orchestration filters), and context summarization/isolation protocols, and 3 open questions.
- `poetry-llm/generation/iterative_refinement.md`: Formulated iterative refinement feedback loop orchestrator with 4 termination conditions (repetition, stagnation, oscillation, max limit) in Python, and 3 open questions.
- `poetry-llm/generation/harness_design.md`: Outlined class definitions (`PoetryCouncilHarness`) implementing personas and roles, with selective context isolation (CoT stripping), deadlock detection, and context compression, and 3 open questions.
- `poetry-llm/generation/output_goals.md`: Specified post-processing sanity gate filters (3-100 lines, blank line density) and novelty thresholds (Distinct-N and cosine embedding distance), and 3 open questions.
- `poetry-llm/evaluation/evaluation_pipeline.md`: Outlined Reward Model pairwise dataset sizing constraints (M1: 1k-2k pairs, M2-M3: 5k-10k pairs) and optimal DPO margin thresholds (0.15) with filtering pseudocode, and 3 open questions.
- `poetry-llm/evaluation/auto_metrics.md`: Defined formulas for Distinct-N and Unseen n-grams, embedding cosine distances, and common-object novelty syntactic/semantic exception detection, and 3 open questions.
- `poetry-llm/evaluation/human_evaluation.md`: Outlined screening protocols for experts and readers, Google Form survey questions, inter-rater statistical metrics (Fleiss' Kappa, Krippendorff's Alpha) with Python libraries, and 3 open questions.
- `poetry-llm/evaluation/aesthetic_quality.md`: Formalized scoring rubrics (1-5 scale) across 6 aesthetic dimensions with high/medium/low quality Korean contemporary poetry examples and critical commentary, and 3 open questions.
- `poetry-llm/evaluation/llm_as_judge.md`: Designed Echo Chamber bias correction prompts and quantitative penalty multipliers ($\lambda_{\text{self}} = 0.92$, $\lambda_{\text{family}} = 0.95$), and 3 open questions.
- `poetry-llm/research/related_work.md`: Extracted pipeline lessons from COIG-Writer (process-supervision) and CreativeBench (quality x novelty steering), and 3 open questions.
- `poetry-llm/research/open_questions.md`: Mapped open questions Q1-Q25 to Project Phase 0-3 roadmaps and updated statuses using literature citations, and 3 open questions.
- `poetry-llm/research/prior_art_failures.md`: Created a Failure-to-Mitigation Matrix mapping AI failures to DPO, G2P, CoT, etc., and detailed technical mitigation tactics for clichés, logical drift, and rhythm break, and 3 open questions.

## 2026-06-27 (Antigravity Agent)
- Antigravity 에이전트 파일 편집 기능 확인 (테스트)

## 2026-06-26
- 번들 초기 생성
- 프로젝트 개요, 데이터, 전처리, 모델, 생성, 평가, 연구 영역 초안 작성
- git 저장소 초기화 및 에이전트 브랜치 전략 수립

## 2026-06-27 (Antigravity Agent)
- `poetry-llm/model/peft_vs_full_finetuning.md` 신규 생성 및 `index.md` 수정: Qwen2.5-32B 모델의 시 생성에 대한 LoRA(PEFT)와 풀 파인튜닝 비교 분석, 하이브리드/전환 전략, 세부 하이퍼파라미터 및 VRAM 메모리 프로파일링 추정, LoraConfig 및 FSDP Trainer 코드 예시 작성.
- `poetry-llm/preprocessing/line_break_tokens.md` 수정: 행갈이/연갈이 특수 토큰 설계 설명 보강 및 Python 기반 `PoetryPreprocessor`/`PoetryPostprocessor`/`PoetryValidator` 구체적 구현 및 검증 파이프라인 추가.
- `poetry-llm/evaluation/evaluation_pipeline.md` 수정: 3단계 평가 파이프라인(Sanity Gate, Reward Model, LLM-as-a-Judge)의 구체적인 Python 클래스 및 API 호출 실패 예외 처리(exponential backoff/fallback) 구현 추가.
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
- `poetry-llm/evaluation/aesthetic_quality.md` 수정: ABS(Aesthetic Balance Score) 가중치 매트릭스 도입으로 전문가/일반 독자 편향 해소, '좋은 새로움'과 '나쁜 새로움'을 구별하기 위한 3단계 필터링 파이프라인 구축 및 Mermaid 다이어그램 추가, 6대 미학 기준별 1-5점 상세 루브릭 및 한국어 대비 예시 보강.
- `poetry-llm/model/model_selection.md` 수정: Qwen2.5-32B vs EXAONE-3.5-32B 비교 심층 분석(토크나이저 효율, VRAM 임베딩 크기, 라이선스, 특수 토큰 확장성 및 다국어 전이 능력), DeepSpeed ZeRO-3 설정 파일 및 모델 가중치 초기화 스크립트 작성, Phase 2 인프라 비용 추정 테이블 추가.
- `poetry-llm/preprocessing/pronunciation.md` 수정: 한국어 표준 발음법 및 선택적 자소-음소 정렬 하이브리드 발음 표기 전략 확정, Kiwi와 g2pK 기반 전처리 파이프라인 Mermaid 다이어그램 구축, 영어/일본어 다국어 처리 명세화, 인라인 특수 XML 유사 태그 방식 채택 및 컨텍스트 윈도우 오버헤드 분석.


