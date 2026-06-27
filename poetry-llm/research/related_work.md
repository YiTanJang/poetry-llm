---
type: Reference
title: 관련 연구 현황 (2026년 6월 업데이트)
description: AI 시 생성 및 창의적 텍스트 생성 관련 주요 논문, 프로젝트, 데이터셋 — 최신 연구 반영.
tags: [research, related-work, papers, nlp, creativity]
timestamp: 2026-06-27T00:00:00Z
---

# 관련 연구 현황

> 최종 업데이트: 2026-06-27. 기존 내용(2026-06-26)에 웹 검색 결과를 대폭 보강.

---

## 1. AI 시 생성 주요 연구

### 1.1 대형 언어모델 기반 시 생성 (기존 + 업데이트)

- **GPT 계열의 시 생성 평가**: 범용 LLM이 시를 생성할 수 있으나
  독창성(novelty)과 문화적 특수성 측면에서 한계 확인
- **PoetryBench** (Yi et al., 2023): 영어 시 생성 평가 벤치마크
- **CreativeNLP**: 창의적 텍스트 생성을 위한 평가 프레임워크
- **ByGPT5** (Belouadi & Eger, 2023): 토큰 없는(token-free) 언어모델로 리듬·각운·두운을
  end-to-end 조건부 생성. GPT-2, ByT5, ChatGPT를 자동 평가와 인간 평가 모두에서 능가함.
  → **이 프로젝트 관련성**: 형식 제약(행갈이, 연 구조)을 특수 토큰으로 처리하는 접근과 유사한 문제의식.
- **Tang Poetry 평가** (arXiv 2510.15313, 2025): 고전 한시(당시) 생성에서 LLM의 "에코챔버 효과"
  확인 — 비슷한 학습 데이터로 학습한 모델들이 유사한 시적 품질 기준으로 수렴하는 현상.
  자동 메트릭이 이 문제를 탐지하지 못함.
  → **이 프로젝트 관련성**: 한국 현대시에서도 동일 문제 발생 가능. 평가 메트릭 설계 시 필수 고려.

### 1.2 파인튜닝 기반 접근

- **GPT-2 Poetry Fine-tuning** (Popescu-Belis et al., 2022):
  시집으로 파인튜닝 시 생성 품질 향상 확인. GPT-2는 리듬·운율 같은
  시 특수 특성 학습에 어려움이 있음.
- **Hafez** (Ghazvininejad et al., 2016):
  형식 제약(리듬, 각운)을 강제한 시 생성 시스템
- **Instruction Tuning for Poetry** (Chakrabarty et al., 2022, arXiv 2210.13669):
  "시 쓰기를 도와달라" 형태의 instruction tuning이 협력적 시 작성의 매개체가 될 수 있음을 탐구.
  → **이 프로젝트 관련성**: 창작 노트(CoT)를 instruction 형태로 학습하는 설계에 직접 참조 가능.
- **Can Good Writing Be Generative?** (arXiv 2601.18353, CHI 2026):
  MFA 작가 28명과 LLM 경쟁 실험. 표준 프롬프팅에서 전문가들이 82.7% 인간 글쓰기 선호.
  **고품질 책으로 파인튜닝 후 AI 선호가 62%로 역전됨.**
  일반 독자는 AI 글쓰기를 선호하는 경향, 전문가들은 AI 선호 시 "심미적 정체성 위기" 경험.
  → **이 프로젝트 관련성**: 고품질 시집 데이터(~2,500권) 기반 학습이 타당함을 지지하는 핵심 근거.

### 1.3 창의성/Novelty 측정

- **Measuring LLM Novelty** (arXiv 2504.09389, 2025):
  "LLM novelty = 원본성(originality) × 품질(quality)"의 조화 평균으로 측정.
  시 쓰기를 포함한 세 태스크에서 평가. **스케일링은 품질을 높이지만 원본성은 안정적.
  Post-training이 베이스 모델 대비 novelty를 일관되게 높임.**
  → **이 프로젝트 관련성**: novelty 측정 방식(unseen n-gram fraction)을 평가 프레임워크에 채택 가능.
- **NoveltyBench** (Zhang et al., arXiv 2504.05228, 2025):
  LLM이 주관성·무작위성·창의성을 요구하는 요청에 얼마나 다양한 답을 낼 수 있는지 평가.
  Distinct-k 메트릭 활용. COLM 2024·ICLR 2025의 90% 이상 평가 논문이 단일/최선 생성만 평가한다는 문제를 지적.
  → **이 프로젝트 관련성**: 반복 수정 파이프라인의 다양성 평가에 직접 활용 가능.
- **CreativeBench** (arXiv 2603.11863, ACL 2026 Findings):
  LLM의 창의성을 조합형 창의성(Combinatorial Creativity: 익숙한 개념의 낯선 조합)과 탐색형 창의성(Exploratory Creativity: 구조화된 개념 공간 탐색 및 발견)으로 분류하여 벤치마킹하는 프레임워크.
  * **핵심 방법론**: 역공학(Reverse Engineering) 및 셀프 플레이(Self-play)를 활용한 자동 파이프라인을 구축하고, **품질(Quality)과 독창성(Novelty)의 곱(Product)**을 통합 메트릭으로 사용.
  * **주요 발견**: 스케일링은 조합형 창의성을 높이지만 탐색형 창의성에는 한계가 있으며, 대형 모델일수록 '스케일링에 의한 수렴(convergence-by-scaling)'으로 인해 정답성은 높지만 발산도가 감소함. 추론(Reasoning) 능력은 제약 조건 하의 탐색에 더 유리함.
  * **창의성 향상 기법**: EvoRePE(Evolutionary Reverse Prompt Engineering) 기법을 통해 추론 시점에 진화적 탐색 패턴을 내재화하여 창의성을 제어.
  → **이 프로젝트 관련성**:
    1) 한국어 현대시 창의성을 '조합형(소재·은어의 낯선 결합)'과 '탐색형(새로운 연 구조 및 메타포 탐색)'으로 분리 평가하는 기준 설계에 기여.
    2) 모델 크기(Qwen2.5-32B)가 커짐에 따라 발생할 수 있는 '초인적 평범함'과 '수렴' 현상을 탐색형 창의성 한계 관점에서 분석하고, 진화적 역프롬프트(EvoRePE)나 추론(CoT) 단계 강화를 통해 이를 보완하도록 파이프라인 설계.
    3) 평가 메트릭에서 품질과 독창성의 곱 형태의 다중 목표 평가 체계 차용.
- **Creative Preference Optimization (CrPO)** (arXiv 2505.14442, EMNLP 2025):
  LLM 창의성(novelty, diversity, surprise, quality) 향상을 위한 선호 최적화 방법론.
  DPO 변형으로 창의성을 직접 최적화.
  → **이 프로젝트 관련성**: 반복 수정 단계에서 비평 신호로 CrPO 목표를 활용할 수 있음.
- **Rethinking Creativity Evaluation** (arXiv 2508.05470, 2025):
  기존 창의성 평가들의 비판적 분석. 대부분 평가가 발산적 사고(divergent thinking)만 측정하고
  수렴적 완성도를 무시함을 지적.
- **Beyond Divergent Creativity** (arXiv 2601.20546, 2026):
  인간 기반 평가로 LLM 창의성을 측정한 연구. 발산적 창의성만으로는 부족함을 보임.

---

## 2. Chain-of-Thought와 창의적 작업

- **CoT의 창의적 태스크 적용** (Wei et al., 2022 이후):
  CoT가 수학/논리 외 창의적 작업에도 유효한지 논쟁 중
- **Creative Chain-of-Thought** 관련 연구들:
  창작 노트 방식이 출력 품질을 향상시킨다는 초기 근거 있음
- **COIG-Writer** (arXiv 2510.14763, ICLR 2026 제출):
  중국어 창의적 글쓰기 능력을 강화하기 위한 프롬프트-사고과정-최종본 데이터셋. **고품질 텍스트로부터 사고 과정을 역공학(Reverse-Engineering)하여 1,665개 triplet(프롬프트 + 창의적 추론 + 최종 글) 구성.**
  * **핵심 발견**: 창의적 글쓰기는 서사 논리(Narrative Logic)와 언어 표현(Linguistic Expression)의 이중 구조로 나뉨. 전자는 과정 수준의 지도학습(Process-level Supervision)으로 학습되고, 후자는 일반 웹 데이터 등의 표현력으로 유지됨.
  * 비영어권 LLM의 창의성 결핍(클리셰 반복 등)은 언어적 표현력 부족이 아니라 창의적 발상 및 전개라는 '과정 감독 데이터'의 부재에서 비롯됨을 실증.
  * **데이터 혼합 전략**: 안정적인 학습을 위해 창의적 데이터와 일반 데이터의 적절한 혼합 비율(최소 1:12)이 성능 저하(general degradation) 방지에 필수적임을 규명.
  → **이 프로젝트 관련성**:
    1) 한국어 시 생성 파이프라인의 **'창작 노트(CoT) → 시 초안 → 반복 수정'** 흐름에서, 창작 노트가 단순한 설명이 아닌 '시상 전개 및 시적 논리(Narrative/Poetic Logic)'를 교정하는 과정 감독 데이터(Process Supervision)로서 동작하도록 정교하게 설계해야 함.
    2) 한국어 현대시 풀 파인튜닝(CPT/SFT) 시, 시집 데이터만을 단독으로 학습시킬 경우 발생할 수 있는 치명적인 언어 능력 붕괴나 과적합을 막기 위해, COIG-Writer의 권장사항인 **창의적 데이터 대 일반 한국어 코퍼스의 혼합 비율(1:12 이상)**을 데이터 혼합 전략에 반영해야 함.
- **LLM Discussion Framework** (arXiv 2405.06373, 2024):
  LLM 토론·역할극을 통한 창의성 향상. 여러 관점이 창의적 결과물의 다양성을 높임.
- **Multi-Novelty** (arXiv 2502.12700, 2025):
  추론 시간(inference-time) 다중 시각 브레인스토밍으로 다양성·novelty 향상.
  → **이 프로젝트 관련성**: 소재 선택 단계에서 inference-time 다양성 강화 기법 참조 가능.

---

## 3. 멀티 에이전트 창의적 글쓰기

- **LLM-based Multi-agent Poetry Generation** (arXiv 2409.03659, 2024):
  비협력적 환경에서 멀티 에이전트 시 생성. Distinct/Novel n-gram 기준 **다양성 3.0-3.7%p 향상,
  novelty 5.6-11.3%p 향상.**
  → **이 프로젝트 관련성**: 단일 모델의 반복 수정 대비 멀티 에이전트 비평이 더 효과적일 수 있음을 시사.
- **Plug-and-Play Dramaturge** (arXiv 2510.05188, 2025):
  협력적 LLM 에이전트를 통한 분할 정복(divide-and-conquer) 방식 서사 대본 반복 정제.
- **Agents' Room** (2024): 스토리 생성을 다단계 협력 문제로 접근 — 계획 에이전트 + 글쓰기 에이전트 분리.
- **CRITICS** (2024): 협력적 멀티 에이전트 비평 과정으로 장형 스토리 생성 강화.
- **HoLLMwood** (arXiv 2406.11683, 2024): 역할극을 통한 시나리오 창의적 생성.
- **LLM Review** (arXiv 2601.08003, 2026): 블라인드 동료 평가(blind peer review) 피드백으로
  창의적 글쓰기 향상. 동료 평가 구조가 자기 비평보다 유의미하게 효과적임.
  → **이 프로젝트 관련성**: Q8(자기 비평 vs 외부 비평) 질문에 직접 답하는 근거.
- **정보 흐름 제한이 창의성 보존**: 다중 에이전트 글쓰기에서 정보 흐름을 전략적으로 제한할 때
  발산적 창의 궤적이 더 잘 유지된다는 발견 (멀티 에이전트 협력 서베이, arXiv 2501.06322).

---

## 4. 소규모 모델의 창의적 글쓰기

- **Igniting Creative Writing in Small LMs** (arXiv 2508.21476):
  LLM-as-a-Judge + 멀티 에이전트 정제 보상으로 소규모 모델의 창의적 글쓰기 능력 향상.
  루브릭 기반 채점과 CoT 추론을 평가에 통합.
  → **이 프로젝트 관련성**: 대규모 모델이 아닌 도메인 특화 소형 모델로도 고품질 시 생성이
  가능함을 시사. 비평 모델로 더 큰 LLM을 활용하는 설계 근거.
- **BILLY** (arXiv 2510.10157, 2025): 페르소나 벡터 병합으로 LLM 창의적 생성 조종.

---

## 5. 합성 데이터와 파인튜닝

- **합성 데이터의 다양성 효과** (arXiv 2511.01490, ACL 2026):
  합성 데이터 다양성이 LLM 파인튜닝에 미치는 영향 분석.
  **중요 발견: 20% 저품질 데이터가 포함된 100K 행 데이터셋이 필터링된 80K 행보다 나쁨.**
  → **이 프로젝트 관련성**: 시 데이터 품질 필터링이 양보다 중요함을 뒷받침.
- **모델의 자기 학습 문제**: 모델이 자신의 출력물로 반복 파인튜닝되면 어휘·문법 다양성이
  점진적으로 감소하는 현상 확인.
  → **이 프로젝트 관련성**: 창작 노트(CoT) 합성 데이터를 생성할 때 원본 인간 텍스트 앵커를 유지해야 함.

---

## 6. 도메인 적응과 지속 사전학습 (Continual Pretraining)

- **LLaMA-Pro** (2024): 새 코퍼스 학습을 위해 모델 블록을 확장하고 추가 파라미터만 튜닝.
  도메인 망각(catastrophic forgetting) 없이 지식 주입.
- **ADEPT** (arXiv 2510.10071, 2025): 적응적 확장과 동적 분리 튜닝을 통한 지속 사전학습.
- **지속 학습 서베이** (Wang et al., CSUR 2025): LLM 지속 학습의 포괄적 서베이.
  데이터 재플레이(data replay)와 합성 코퍼스 구성 전략이 주요 완화책.
  → **이 프로젝트 관련성**: 시 도메인 특화 CPT 시 일반 언어 능력 유지와의 균형 설계에 참고.

---

## 7. 한국어 시 생성 (현황과 공백)

- **한국어 특화 시 생성 연구는 현재까지 매우 부족**:
  주로 GPT 계열 범용 모델 활용 실험 수준. 이 프로젝트의 핵심 공백(gap).
- **KoGPT2**: 40GB 이상 한국어 텍스트로 학습한 디코더 언어모델. 가사 생성 등에 활용되었으나
  시 특화 학습 사례 없음.
- **Transformer-based Korean PLM 서베이** (arXiv 2112.03014, 2021): 한국어 사전학습 모델 3년간
  발전 서베이. 문학 생성 태스크는 다루지 않음.
- **중국어 고전시 연구의 시사점** (Tang Poetry 평가, arXiv 2510.15313): 한자문화권 언어로서
  중국어 시 생성 연구가 한국 현대시에 주는 참조점. 형식 제약(형식적 언어 패턴, 운율) 학습의 어려움.
- **Persian Literary Text 창의성 평가** (arXiv 2509.18401, 2025): 비영어권 문학 텍스트에서
  LLM 창의성 평가. 비영어 문학 연구의 드문 선례.
  → **이 프로젝트 관련성**: 한국어 시 창의성 평가 방법론 설계의 비교 대상.

---

## 8. 관련 데이터셋

| 데이터셋 | 언어 | 내용 | 관련성 |
|----------|------|------|--------|
| PoetryFoundation Dataset | EN | 14,000여 편 영어시 | 간접 참조 |
| McGill-Poetry Dataset | EN | 형식 분석 포함 | 형식 분석 참조 |
| 구텐베르크 시 코퍼스 | 다국어 | 공공도메인 시 | 다국어 전이 |
| KorLit (비공개) | KO | 한국문학 코퍼스 (접근 제한) | 직접 참조 |
| COIG-Writer | ZH | 창의적 글쓰기 사고 과정 포함 | 방법론 참조 |
| Tang Poetry Corpus | ZH | 고전 한시 대규모 코퍼스 | 동아시아 시 참조 |

---

## 9. 관련 프로젝트

### Co-Poet (Google Research)
- LLM이 시인과 협업하는 인터랙티브 시스템
- 시인의 피드백을 실시간 반영

### Verse by Verse (Google Arts)
- 미국 고전시 스타일 모방 시 생성
- → 이 프로젝트와 차별점: 모방이 아닌 novelty 추구

### Bing/ChatGPT 시 생성
- 범용 LLM의 시 생성 한계: 클리셰, 예측 가능성, 문화 편향
- ChatGPT의 "초인적 평범함(superhuman banality)" 문제 (Antislop, arXiv 2510.15061)

---

## 10. 이 프로젝트의 포지셔닝 (업데이트)

| 차원 | 기존 연구 | 이 프로젝트 |
|------|----------|------------|
| 언어 | 주로 영어·중국어 | 한국어 특화 |
| 접근 | 범용 모델 프롬프팅 | 도메인 특화 풀 파인튜닝 |
| 목표 | "좋은 시" 생성 | novelty + 미학적 품질 |
| 과정 | 단일 생성 | CoT + 반복 수정 |
| 데이터 | 소규모 | 대규모 (시집 ~2,500권 + 시론) |
| 과정 감독 | 없음 | 창작 노트(사고 과정) 포함 |
| novelty 측정 | 없거나 영어 중심 | 한국어 특화 평가 프레임워크 |

---

## 미결 참고문헌 (찾아볼 것)

- [TODO] 한국 현대시 자동 분석 관련 NLP 논문 (형태소 분석 기반 운율 분석 등)
- [TODO] 문학적 창의성(literary creativity) 계산 측정 연구 — Birkhoff 1933부터 최신까지
- [TODO] 다국어 미학 전이(multilingual aesthetic transfer) 연구
- [TODO] 커리큘럼 학습이 창의적 생성에 미치는 영향
- [TODO] 일본 하이쿠 NLP 생성 연구 (여백의 미 전이 가능성)
- [TODO] CrPO(Creative Preference Optimization)를 시 도메인에 직접 적용한 선례

---

# Citations

1. Belouadi, J., & Eger, S. (2023). *ByGPT5: End-to-End Style-conditioned Poetry Generation with Token-free Language Models*. ResearchGate / arXiv.
2. Chakrabarty, T. et al. (2022). *Help me write a poem: Instruction Tuning as a Vehicle for Collaborative Poetry Writing*. arXiv:2210.13669.
3. Zhang, et al. (2025). *Measuring LLM Novelty As The Frontier Of Original And High-Quality Output*. arXiv:2504.09389.
4. Zhang, et al. (2025). *NoveltyBench: Evaluating Language Models for Humanlike Diversity*. arXiv:2504.05228.
5. COIG-Writer Team (2025). *COIG-Writer: A High-Quality Dataset for Chinese Creative Writing with Thought Processes*. arXiv:2510.14763.
6. Tang Poetry Evaluation (2025). *Capabilities and Evaluation Biases of Large Language Models in Classical Chinese Poetry Generation*. arXiv:2510.15313.
7. LLM-based Multi-agent Poetry (2024). *LLM-based multi-agent poetry generation in non-cooperative environments*. arXiv:2409.03659.
8. Igniting Creative Writing (2025). *Igniting Creative Writing in Small Language Models: LLM-as-a-Judge versus Multi-Agent Refined Rewards*. arXiv:2508.21476.
9. Creative Preference Optimization (2025). *Creative Preference Optimization*. arXiv:2505.14442. EMNLP 2025 Findings.
10. Bom, T. et al. (2026). *Can Good Writing Be Generative? Expert-Level AI Writing Emerges through Fine-Tuning on High-Quality Books*. arXiv:2601.18353. CHI 2026.
11. LLM Review (2026). *LLM Review: Enhancing Creative Writing via Blind Peer Review Feedback*. arXiv:2601.08003.
12. Plug-and-Play Dramaturge (2025). *Plug-and-Play Dramaturge: A Divide-and-Conquer Approach for Iterative Narrative Script Refinement via Collaborative LLM Agents*. arXiv:2510.05188.
13. Multi-Novelty (2025). *Multi-Novelty: Improve the Diversity and Novelty of Contents Generated by Large Language Models via inference-time Multi-Views Brainstorming*. arXiv:2502.12700.
14. Antislop (2025). *Antislop: A Comprehensive Framework for Identifying and Removing Clichés*. arXiv:2510.15061.
15. Rethinking Creativity Evaluation (2025). *Rethinking Creativity Evaluation: A Critical Analysis of Existing Creativity Evaluations*. arXiv:2508.05470.
16. Persian Literary Creativity (2025). *Evaluating the Creativity of LLMs in Persian Literary Text Generation*. arXiv:2509.18401.
17. Synthetic Data Diversity (2026). *Synthetic Eggs in Many Baskets: The Impact of Synthetic Data Diversity on LLM Fine-Tuning*. arXiv:2511.01490. ACL 2026 Findings.
18. Wang et al. (2025). *Continual Learning of Large Language Models: A Comprehensive Survey*. CSUR 2025.
19. Evaluating Diversity in Automatic Poetry Generation (2024). arXiv:2406.15267.
20. CreativeBench (2026). arXiv:2603.11863.
21. Beyond Divergent Creativity (2026). arXiv:2601.20546.
22. BILLY (2025). *BILLY: Steering Large Language Models via Merging Persona Vectors for Creative Generation*. arXiv:2510.10157.
23. Popescu-Belis, A. et al. (2022). *GPT-2 Poetry Fine-tuning*. (참조).
24. Ghazvininejad, M. et al. (2016). *Hafez: An Interactive Poetry Generation System*.

## 미결 사항

- [Ph1] COIG-Writer의 데이터 혼합 전략(1:12 비율)이 한국어 현대시 풀 파인튜닝 시 성능 저하(general degradation) 방지와 시적 독창성 유지의 관점에서도 최적의 비율인지 어떻게 실증할 것인가?
- [Ph1] CreativeBench의 탐색형 창의성(Exploratory Creativity) 측정 개념을 도입할 때, 실행(execution) 검증이 불가능한 한국어 현대시 영역에서 참신한 시상(Novelty)과 무작위 환각(Hallucination)의 경계를 구분하는 정량적 평가 메트릭을 어떻게 정립할 것인가?
- [Ph1] CreativeBench에서 제안된 진화적 역프롬프트(EvoRePE)와 같은 추론 시점의 다양성 제어 및 지향적 탐색 기법을 우리의 '창작 노트(CoT) → 시 초안 → 반복 수정' 3단계 생성 파이프라인에 어떻게 이식할 것인가?
