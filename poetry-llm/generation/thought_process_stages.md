---
type: Design
title: 추론 구조의 창발 — 프로세스 감독 설계
description: 8단계 처방적 추론 프레임을 폐기하고, 프로세스 감독을 통해 추론 구조가 학습에서 창발되도록 설계한다. 템플릿 준수가 아닌 트레이스 품질이 학습 신호다.
tags: [generation, cot, process-supervision, emergent-reasoning]
timestamp: 2026-06-28T00:00:00Z
---

# 추론 구조의 창발 — 프로세스 감독 설계

## 이전 설계의 무엇이 잘못됐는가

이 문서는 한때 8단계 추론 프레임워크를 정의했다:
감각 포착 → 개념 추상화 → 긴장관계 설정 → 언어 탐색 → 형식 결정 → 초안 생성 → 갭 분석 → 수정 계획.

이 프레임워크를 폐기한다.

**왜:**

추론 단계를 처방하는 것은 두 가지 방식으로 실패한다.

첫째, **형식이 사고를 모방한다.** 모델은 "감각 포착:" 다음에 올 내용이 어떻게 생겼는지 학습한다. 하지만 실제로 감각을 포착하는 사고 과정을 학습하는 것과 다르다. 레이블에 이어지는 텍스트를 생성하는 것은, 그 레이블이 의미하는 인지 작업을 수행하는 것이 아니다.

둘째, **좋은 사고는 고정된 순서가 없다.** 어떤 시인은 형식을 먼저 결정하고 내용을 채운다. 어떤 시인은 언어 탐색에서 출발하여 역으로 씨앗을 찾는다. 어떤 시인은 금지를 먼저 정한다. 8단계 순서를 강제하는 것은 이 다양성을 제거한다.

---

## 프론티어 랩의 접근: 추론 구조는 창발된다

o1과 DeepSeek-R1은 추론 단계를 처방받지 않았다.

이 모델들의 학습 과정:
1. 문제와 그 풀이(시 생성의 경우: 소재와 좋은 시)의 쌍을 수집한다.
2. 여러 추론 경로를 시도한다.
3. 좋은 결과로 이어진 경로에 프로세스 감독 신호를 부여한다.
4. 반복 학습을 통해 모델은 고품질 결과와 상관관계가 있는 추론 패턴을 내면화한다.

결과: 모델은 특정 문제 유형에서 **스스로 적합한 추론 구조를 발견한다.**
이 구조는 인간이 처방한 것이 아니며, 데이터와 신호에서 창발된 것이다.

우리도 같은 방향이다.

---

## 프로세스 감독: 시 생성에서의 의미

### 결과 감독 vs 프로세스 감독

**결과 감독 (outcome supervision):**
- 최종 시의 품질에만 보상
- 스크래치패드 내용은 신호에 포함되지 않음
- 문제: 모델이 좋은 시를 생성하기 위해 어떤 사고 과정을 거치든 상관없다
- 결과: 추론 품질이 통제되지 않음

**프로세스 감독 (process supervision):**
- 좋은 결과로 이어진 스크래치패드 전체를 긍정 신호로 사용
- 나쁜 결과로 이어진 스크래치패드는 부정 신호
- 결과: 모델이 고품질 시와 상관관계가 있는 사고 패턴을 학습

우리는 프로세스 감독을 채택한다.

### 무엇을 감독하는가

**감독하는 것:**
- 스크래치패드 → 시의 전체 트레이스
- 트레이스가 외부 판정 모델에서 높은 평가를 받는가

**감독하지 않는 것:**
- 스크래치패드가 특정 형식을 따르는가
- 특정 단계가 존재하는가
- 각 섹션의 길이

**핵심 원칙:** 좋은 시로 이어진 추론은 좋은 추론이다. 그 추론이 어떻게 생겼든.

---

## 추론 구조가 창발되는 방식

충분한 트레이스를 학습하면, 모델은 특정 패턴을 내면화할 것이다.

우리가 기대하는(하지만 처방하지 않는) 패턴들:

**탐색과 거부 패턴:**
좋은 시는 종종 나쁜 방향을 먼저 탐색하고 거부한 흔적이 있다.
"이 표현은 이미 본 것 같다 → 다른 방향으로" 같은 움직임.
이 패턴이 트레이스에 충분히 많으면 모델은 그것을 학습한다.

**제약 명시 패턴:**
쓰지 않을 것을 명시하는 것이 생성 품질과 상관관계가 있다면,
모델은 좋은 시를 쓰기 전에 금지를 명시하는 패턴을 학습할 것이다.

**언어 비교 패턴:**
단어나 구문의 대안을 비교하고 선택하는 것이 품질과 상관관계가 있다면,
모델은 그 비교 패턴을 내면화할 것이다.

이 패턴들이 창발되는지, 창발된다면 어떤 형태로 나타나는지는 **실험으로 확인한다.**

---

## 8단계 프레임워크의 위치

폐기된 8단계(감각 포착, 개념 추상화, 긴장관계 설정, 언어 탐색, 형식 결정, 초안 생성, 갭 분석, 수정 계획)는 **추론 시 사용되지 않는다.**

그러나 학습 데이터 트레이스 생성 시 **느슨한 영감**으로 참조할 수 있다.
3가지 시인 페르소나(A/B/C)가 초기 트레이스를 생성할 때, 각 페르소나의 스타일에 맞게 이 개념들을 자유롭게 사용할 수 있다.

단, 트레이스를 평가할 때 "8단계를 따랐는가"는 기준이 아니다. 기준은 오직 "이 트레이스가 좋은 시로 이어졌는가"다.

---

## 중간 보상 신호 (탐색적)

프로세스 감독을 더 세밀하게 구현하려면 트레이스의 중간 지점에 보상을 줘야 한다.

시 생성 맥락에서 가능한 중간 신호:

**긍정 중간 신호:**
- 스크래치패드에서 명시적으로 거부 결정을 내린 뒤 대안을 탐색하는 흐름
- 금지를 명시하는 행위
- 초안을 생성하고 그것의 문제를 발견하는 흐름
- 구체적 언어 선택에 이유를 붙이는 것

**부정 중간 신호:**
- 씨앗 없이 즉시 추상으로 시작하는 스크래치패드
- 탐색 없이 첫 번째 선택을 그대로 채택하는 패턴
- 금지 항목이 최종 시에 등장하는 경우

이 신호들은 가설이다. 실제로 품질과 상관관계가 있는지 실험이 필요하다.

> 가설: 중간 보상을 추가하면 결과 감독만 사용할 때보다 추론의 다양성과 시의 novelty가 높아진다.

---

## 사고-최종 시 정렬 검증 프레임워크 (Alignment & Traceability)

추론 과정(CoT)의 품질이 단순히 형식적 충족에 그치지 않고, 최종 생성된 시에 실제로 유의미하게 투영되었는지 확인하기 위한 **정렬 검증 프레임워크(Traceability Framework)**를 설계한다. 이 프레임워크는 사고 단계에서의 의도와 결정사항이 최종 출력물에 충실히 반영되었는지를 객관적이고 자동화된 지표로 평가한다.

### 1. 자동화된 정렬 매칭 지표 (Automated Alignment Metrics)

1. **CoT 트레이스 충실도(Fidelity)의 자동화된 시맨틱 평가 (SBERT Semantic Similarity)**
   - **대상**: CoT 스크래치패드 내에서 브레인스토밍된 구체적인 발상 세그먼트(개념, 은유, 정서)와 최종 출력된 시의 본문 및 개별 행/연.
   - **방법**: 한국어 문장 임베딩 모델(예: KoSentenceBERT, KoE5)을 활용하여 CoT 세그먼트와 시 구절 간의 코사인 유사도(Cosine Similarity)를 측정한다.
     - **개념 매칭 (Concept Matching)**: CoT의 개념 설계 세그먼트 $T_{concept}$와 시 전체의 결합 임베딩 간의 유사도를 측정하여 시의 거시적 정체성을 평가한다.
       $$\text{SemanticSim}_{\text{concept}} = \cos(E(T_{concept}), E(Poem))$$
     - **은유 매칭 (Metaphor Matching)**: CoT에서 계획된 개별 은유 표현 $M_k$가 시의 특정 행 $L_i$에 투영되었는지 추적하기 위해, 최댓값 풀링(Max Pooling) 유사도를 적용한다. 은유가 시의 임의의 위치에 유해했는지를 탐색한다.
       $$\text{SemanticSim}_{\text{metaphor}} = \frac{1}{|M|} \sum_{M_k \in M} \max_{L_i \in Poem} \cos(E(M_k), E(L_i))$$
     - **정서 매칭 (Emotion Matching)**: CoT에서 지향하고자 선언한 감정 상태나 분위기를 담은 세그먼트 $T_{emotion}$과 시의 각 연(Stanza) $S_j$ 간의 최대 유사도를 측정한다.
       $$\text{SemanticSim}_{\text{emotion}} = \max_{S_j \in Poem} \cos(E(T_{emotion}), E(S_j))$$
   - **목적**: 스크래치패드에서 구상한 미학적 아이디어(개념, 은유, 정서)가 최종 시에 충실하게 반영되었는지 검증하고, 사고 과정과 생성물이 서로 괴리되는 현상을 차단한다.

2. **키워드 커버리지 및 재현율 (Keyword Coverage & Recall)**
   - **긍정 재현율 (Positive Recall)**: CoT 과정에서 "이 시의 핵심 이미지로 채택하겠다"고 결정한 고유 키워드 집합 $K_{CoT}$가 실제 최종 시 단어/형태소 집합 $W_{Poem}$에 포함된 비율을 계산한다.
     $$\text{Recall}_{\text{keyword}} = \frac{|K_{CoT} \cap W_{Poem}|}{|K_{CoT}|}$$
   - **키워드 커버리지 (Keyword Coverage)**: 형태소 분석기(예: Mecab 등)를 활용하여 형태소(Morpheme) 단위 매칭을 수행함으로써, 한국어 조사나 어미 변화로 인한 오차를 방지하고 발상의 키워드가 시에 고루 분포하는지 측정한다.
   - **부정 제약 준수도 (Negative Constraint Adherence)**: CoT 과정에서 "상투적이므로 배제하겠다"고 선언한 금지 단어가 최종 시에 등장했는지 여부를 판별한다. 등장할 경우 감점을 부과한다.

3. **구조적 준수율 (Structural Adherence)**
   - **대상**: CoT에서 설정한 형식적 제약(예: "3연 구성", "각 연은 2행씩", "특수 토큰 사용 계획")과 최종 시의 실제 구조.
   - **방법**: 정규표현식 및 파서를 통해 최종 시의 연(Stanza)과 행(Line) 개수를 세고, CoT의 계획값과 일치하는지 비율을 산출한다.
   - **특수 토큰 검증**: CoT에서 의도한 위치에 `<행갈이>`, `<연갈이>`가 규칙적으로 삽입되었는지 체크한다.

---

### 2. 역할 오염 페널티 (Role-Contamination Penalty)

**역할 오염(Role-Contamination)**이란 생성 모델이 추론(CoT/Scratchpad) 영역과 최종 출력(시) 영역의 경계를 혼동하여 상호 오염시키는 현상이다.
- **예시 A (CoT 오염)**: 스크래치패드 내에 생각 과정이 아닌, 이미 완성된 형태의 시 구절을 무단으로 미리 나열하는 경우.
- **예시 B (시 오염)**: 최종 시 출력 영역에 메타 지시어나 설명적 어조("~를 시로 표현해 보았습니다", "1연을 작성하겠습니다" 등), 혹은 추론 단계의 마크다운 헤더가 침범하는 경우.

**페널티 부과 방식:**
1. **오염 감지**: 최종 시 영역에서 종결어미(`-습니다`, `-한다` 등의 메타 서술), 특정 마크다운 패턴(`##`, `- `), 혹은 생각 프로세스 특수 토큰의 부적절한 노출을 탐색한다.
2. **수식 적용**: 오염이 감지되면 정렬도 점수(Alignment Score)에 무조건적인 멀티플라이어 페널티($\alpha_{penalty} = 0.0$ 또는 $0.1$)를 곱해 해당 트레이스의 가치를 무력화한다. RLHF/RLAIF 보상 모델에서 해당 트레이스의 보상값을 최하점으로 강제한다.

---

### 3. 검증 엔진 파이썬 의사코드 (Python Pseudocode)

아래는 사고 과정과 최종 시 간의 정렬도 및 역할 오염을 종합적으로 자동 평가하는 검증 프레임워크의 의사코드이다.

```python
import re
from typing import Dict, List, Tuple

def calculate_fidelity_semantic_similarity(cot_segments: Dict[str, str], poem: str) -> Dict[str, float]:
    """
    Sentence-BERT 등을 활용하여 CoT 내 세그먼트(개념, 은유, 정서)와 
    최종 시(및 행, 연) 간의 세부 코사인 유사도를 계산합니다.
    """
    # 1. 개념 매칭 (Concept Matching)
    # concept_vector = embedding_model.encode(cot_segments.get("concept", ""))
    # poem_vector = embedding_model.encode(poem)
    # concept_sim = cosine_similarity(concept_vector, poem_vector)
    
    # 2. 은유 매칭 (Metaphor Matching - Max-pooling over poem lines)
    # poem_lines = [line for line in poem.split('\n') if line.strip()]
    # line_vectors = [embedding_model.encode(l) for l in poem_lines]
    # metaphor_sims = []
    # for metaphor in cot_segments.get("metaphors", []):
    #     m_vector = embedding_model.encode(metaphor)
    #     max_sim = max([cosine_similarity(m_vector, l_vec) for l_vec in line_vectors])
    #     metaphor_sims.append(max_sim)
    # metaphor_sim = sum(metaphor_sims) / len(metaphor_sims) if metaphor_sims else 1.0
    
    # 3. 정서 매칭 (Emotion Matching - Max-pooling over stanzas)
    # stanzas = [s for s in poem.split('\n\n') if s.strip()]
    # stanza_vectors = [embedding_model.encode(s) for s in stanzas]
    # emotion_vector = embedding_model.encode(cot_segments.get("emotion", ""))
    # emotion_sim = max([cosine_similarity(emotion_vector, s_vec) for s_vec in stanza_vectors]) if stanza_vectors else 1.0
    
    return {
        "concept_similarity": 0.82,
        "metaphor_similarity": 0.79,
        "emotion_similarity": 0.85
    }

def evaluate_keyword_adherence(chosen_keywords: List[str], 
                               prohibited_keywords: List[str], 
                               poem: str) -> Tuple[float, float]:
    """
    긍정 키워드 재현율 및 부정 키워드 위반 여부를 계산합니다.
    """
    if not chosen_keywords:
        recall = 1.0
    else:
        matched_positive = [w for w in chosen_keywords if w in poem]
        recall = len(matched_positive) / len(chosen_keywords)
        
    violation_count = sum(1 for w in prohibited_keywords if w in poem)
    # 위반 키워드 하나당 0.2씩 감점 (최소 0.0)
    negative_score = max(0.0, 1.0 - (violation_count * 0.2))
    
    return recall, negative_score

def evaluate_structural_adherence(expected_stanzas: int, poem: str) -> float:
    """
    최종 시의 연(stanza) 개수가 CoT의 계획과 일치하는지 평가합니다.
    """
    # 연갈이 토큰 '<연갈이>' 또는 연속 개행('\n\n') 기준으로 연 분할
    stanzas = [s for s in re.split(r'<연갈이>|\n\n', poem.strip()) if s.strip()]
    actual_stanzas = len(stanzas)
    
    if expected_stanzas == 0:
        return 1.0
    
    # 계획된 연 수와의 오차율 기반 점수 산출
    error = abs(expected_stanzas - actual_stanzas)
    return max(0.0, 1.0 - (error / expected_stanzas))

def detect_role_contamination(poem: str) -> bool:
    """
    최종 시 내부에 메타 설명, 생각 마크다운, 혹은 대화형 어미가 포함되어 
    역할 오염이 일어났는지 감지합니다.
    """
    # 마크다운 헤더, 목록 지시자, 또는 특정 메타 어투 분석
    contamination_patterns = [
        r"^#+\s",                    # 마크다운 헤더 침범
        r"^[-\*\+]\s",                # 리스트 불릿 침범
        r"(작성하겠습니다|작성함|생각 과정)", # 메타 서술어
        r"(<시작>|<끝>|<행갈이>|<연갈이>)\s*:\s*" # 태깅 메타 구조 잔재
    ]
    
    for pattern in contamination_patterns:
        if re.search(pattern, poem, re.MULTILINE):
            return True
            
    # 시의 마지막 부분에 사설성 멘트가 길게 들어가는 경우 감지
    if "시를" in poem or "주제" in poem:
        if len(poem) > 100 and any(end in poem for end in ["감사합니다", "도움이 되셨기를"]):
            return True
            
    return False

def calculate_alignment_score(trace: Dict) -> Dict[str, float]:
    """
    CoT와 최종 시의 전체 정렬 상태를 평가하여 종합 점수를 반환합니다.
    """
    cot_segments = trace.get("cot_segments", {})
    poem = trace.get("poem", "")
    chosen_keywords = trace.get("chosen_keywords", [])
    prohibited_keywords = trace.get("prohibited_keywords", [])
    expected_stanzas = trace.get("expected_stanzas", 0)
    
    # 1. 개별 지표 계산
    sim_scores = calculate_fidelity_semantic_similarity(cot_segments, poem)
    avg_semantic_sim = sum(sim_scores.values()) / len(sim_scores) if sim_scores else 0.0
    
    recall, neg_adherence = evaluate_keyword_adherence(chosen_keywords, prohibited_keywords, poem)
    struct_adherence = evaluate_structural_adherence(expected_stanzas, poem)
    
    # 2. 역할 오염 체크 (오염 시 페널티 배수 0.0 적용)
    is_contaminated = detect_role_contamination(poem)
    penalty_multiplier = 0.0 if is_contaminated else 1.0
    
    # 3. 가중합 산출
    raw_score = (
        0.4 * avg_semantic_sim +
        0.3 * ((recall + neg_adherence) / 2.0) +
        0.3 * struct_adherence
    )
    
    final_score = raw_score * penalty_multiplier
    
    result = {
        "keyword_recall": recall,
        "negative_constraint_score": neg_adherence,
        "structural_adherence": struct_adherence,
        "is_contaminated": is_contaminated,
        "final_alignment_score": final_score
    }
    result.update(sim_scores)
    return result
```

## 미결 사항

- [Ph1] 자동화된 시맨틱 유사도 임계치 설정: CoT 계획과 시 본문 간의 문맥적 유사도를 Sentence-BERT로 측정할 때, 시의 은유적 특성으로 인해 발생하는 낮은 유사도 점수와 실제 의도된 이탈을 구별할 수 있는 최적의 임계치(Threshold)는 얼마인가? 특히 개념(Concept), 은유(Metaphor), 정서(Emotion) 세그먼트별로 고유한 언어적 추상 수준이 다르므로, 각 세그먼트별로 차등 임계치를 적용해야 할 것인가에 대한 검증이 필요하다.
- [Ph1] 부정 제약(금지어) 리스트의 다이내믹 피드백: 시인의 스타일이나 주제별로 유동적으로 바뀌어야 하는 금지어 사전을 CoT 내에서 어떻게 효율적으로 추론하고 검증 프레임워크에 인자로 전달할 것인가?
- [Ph1] RLHF/RLAIF 파이프라인에서의 페널티 스케일링: 역할 오염이 발생했을 때 보상값을 완전 제로(0.0)화하는 강한 페널티가 PPO/DPO 등 정책 최적화 과정에서 학습 불안정성(Reward hacking or Optimization collapse)을 유발하는지 여부와 이를 완화할 부드러운 감점 스케일링 방식의 유효성은 어떠한가?
