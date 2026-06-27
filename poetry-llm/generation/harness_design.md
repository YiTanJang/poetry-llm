---
type: Design
title: 에이전트 하네스 설계
description: 3개 시인 페르소나, 비평가, 독자 역할을 조율하고 DPO 데이터를 생성하는 PoetryCouncilHarness Python 클래스 인터페이스 및 컨텍스트 격리 설계.
tags: [generation, multi-agent, council, harness, DPO]
timestamp: 2026-06-27T00:00:00Z
---

# 에이전트 하네스 설계

## 핵심 설계 질문에 대한 답변

### Q1. 각 에이전트는 같은 모델인가, 다른 모델인가?

**지향점: 미학적 목적과 리소스 제약에 맞춘 멀티 모델 구성**

3-시인 앙상블 DPO 학습 루프에서는 역할에 따라 최적의 성능과 비용을 달성하기 위해 에이전트 모델을 다르게 설정한다.

1. **시인 페르소나 (Traditionalist, Modernist, Avant-garde):**
   - **학습 대상 모델:** 파인튜닝(SFT 및 DPO)을 수행할 대상 모델(예: Qwen2.5-32B-Instruct 기반으로 SFT 학습을 마친 중간 모델 체크포인트)을 탑재한다.
2. **비평가 에이전트 (Critic):**
   - **고추론 모델:** 텍스트 내부의 미학적 긴장과 상투성을 논리적으로 집어내야 하므로 높은 추론 성능이 필요하다. `Qwen2.5-72B-Instruct` 혹은 상용 모델인 `Claude 3.5 Sonnet`을 사용한다.
3. **독자 에이전트 (Reader):**
   - **경량화 모델:** 창작 노트나 미학 이론의 영향 없이 일반인의 관점에서 직관적인 독서 반응만을 출력하므로, 대형 모델보다는 속도가 빠르고 직관적 반응에 능한 `Qwen2.5-14B-Instruct` 또는 `Claude 3.5 Haiku` 정도로 충분하다.
4. **외부 판정 모델 (External Judge):**
   - **최상위 모델:** 세 가지 페르소나 시의 최종 수작을 비교 분석해 DPO 선호 신호(chosen/rejected)를 생성하는 핵심 판정관이므로, 최상의 미학적 판단력을 가진 `Claude 3.5 Sonnet` 또는 `GPT-4o`를 고정 배정한다.

```python
AGENT_MODELS = {
    "poet_traditionalist": "poetry-sft-v0.1-32b",
    "poet_modernist": "poetry-sft-v0.1-32b",
    "poet_avant_garde": "poetry-sft-v0.1-32b",
    "critic": "claude-3-5-sonnet",
    "reader": "claude-3-5-haiku",
    "judge": "claude-3-5-sonnet-heavy"
}
```

---

### Q2. 각 라운드에서 컨텍스트를 어떻게 전달하는가 (컨텍스트 격리)?

**전략: 미학적 독립성 및 신선함 검증을 위한 'CoT 흔적 격리' 및 '창작 노트 차단'**

컨텍스트 오염을 원천 차단하고 각 에이전트가 본연의 미학적 역할을 하도록 선택적으로 컨텍스트를 제어한다.

| 에이전트 | 입력 컨텍스트 | 격리/필터 처리 | 목적 |
| :--- | :--- | :--- | :--- |
| **시인 페르소나** | 창작 노트 + 피드백 이력 + 직전 비평가/독자 피드백 | 슬라이딩 윈도우 방식으로 t-2 이전 비평은 압축 요약 | 피드백 집중도 향상 및 토큰 절약 |
| **비평가** | 현재 생성된 시 본문 + 원래의 창작 노트 | **시인의 생각과정(CoT/Scratchpad) 일체 삭제** | 시인의 창작 변명이나 해설을 배제하고 오직 텍스트 자체만으로 미학적 규칙(금지어, 이미지 상투성) 준수 여부 검증 |
| **독자** | 현재 생성된 시 본문 | **시인의 CoT 및 창작 노트(의도) 일체 차단** | 시인의 의도나 테마를 선입견으로 주입받지 않은 상태에서 독자의 날것의 반응(Raw Reader Reaction)과 인지적 마찰 지점 포착 |
| **외부 판정 모델** | 3개 페르소나의 최종 시 본문 + 각각의 생각과정(CoT) | 격리 없음 | 스크래치패드에 기록된 미학적 사유의 진정성이 실제 작품으로 어떻게 구현되었는지 총체적으로 비교 판정 |

---

### Q3. 컨텍스트 창 고갈 및 토큰 제어 전략

1. **CoT 흔적 제거:** 시인이 출력한 8단계 사고 과정(CoT)은 회당 1,000~1,500 토큰에 달한다. 비평가와 독자에게 시를 넘겨주기 전에 `<시작>`과 `<끝>` 태그 내부의 시 텍스트만 파싱하여 전달한다.
2. **슬라이딩 윈도우 피드백 요약:** 수정 라운드가 진행됨에 따라 이전 라운드의 전체 대화 기록을 보존하지 않고, 직전 라운드(t-1)의 상세 비평만 전달한다. 그 이전(t-2 이하)의 비평 및 수정 사실은 오케스트레이터가 한 줄 요약 형식으로 누적 압축하여 컨텍스트에 삽입한다.

---

### Q4. 상태 관리 및 DPO 데이터셋 구조

하네스는 각 페르소나별 수정 루프 상태를 `PoemRevisionState` 단위로 기록하며, 최종 판정 결과를 통합하여 DPO 선호 데이터(`DPOPreferencePair`) 형태로 디스크에 영구 보관한다.

---

## PoetryCouncilHarness 클래스 인터페이스 및 구현

아래는 3가지 시인 페르소나와 비평가, 독자, 외부 판정 모델을 오케스트레이션하고 컨텍스트 격리 및 포스트 필터링을 수행하는 Python 구현체 뼈대이다.

```python
import asyncio
import json
import re
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Dict, List, Optional, Tuple

class PersonaType(str, Enum):
    TRADITIONALIST = "traditionalist"  # 페르소나 A: 전통주의 서정시인
    MODERNIST = "modernist"            # 페르소나 B: 주지주의 모더니스트
    AVANT_GARDE = "avant-garde"        # 페르소나 C: 아방가르드 실험시인

@dataclass
class PoetOutput:
    persona: PersonaType
    round_num: int
    raw_response: str            # 생각과정(CoT) + 시 본문을 포함한 모델 출력 전체
    poem_text: str               # <시작>과 <끝> 사이에서 추출된 순수 시 본문
    scratchpad: str              # <시작> 이전의 사고 과정 (CoT trace)
    decision: str                # 피드백 수용 결정: "Accept" | "Defend" | "Negotiate"
    explanation: str             # 수정 방향 또는 방어 논리 설명

@dataclass
class CriticOutput:
    round_num: int
    raw_response: str
    flaws: List[Dict[str, Any]]  # [{"line_num": int, "flaw_type": str, "description": str}]
    severity: str                # "심각" | "보통" | "경미"
    summary: str                 # 전체 비평 요약

@dataclass
class ReaderOutput:
    round_num: int
    raw_response: str
    residual_image: str          # 독서 후 남은 잔상
    cognitive_friction: str      # 인지적 마찰을 일으킨 부분
    emotional_tone: str          # 느껴진 감정적 기류
    recommend: bool              # 추천 여부

@dataclass
class PoemRevisionState:
    round_num: int
    poet_output: PoetOutput
    critic_output: Optional[CriticOutput] = None
    reader_output: Optional[ReaderOutput] = None

@dataclass
class JudgeOutput:
    winner: PersonaType
    reasoning: str
    loser_analyses: Dict[PersonaType, str]


class PoetryCouncilHarness:
    """
    3개 시인 페르소나, 비평가, 독자, 판정 에이전트를 오케스트레이션하여
    시적 완성도 향상 루프를 돌리고 최종 DPO 선호 학습 쌍을 생산하는 하네스.
    """
    def __init__(
        self,
        creative_notes: Dict[str, Any],
        max_rounds: int = 3,
        system_prompts_dir: str = "prompts"
    ):
        self.creative_notes = creative_notes
        self.max_rounds = max_rounds
        self.system_prompts_dir = system_prompts_dir
        
        # 실제 구현 시에는 API 클라이언트 초기화
        # self.client = AsyncAnthropic() / AsyncOpenAI()

    # ==========================================
    # 1. 컨텍스트 격리 및 텍스트 파싱 유틸리티
    # ==========================================
    
    def _strip_cot_trace(self, raw_poet_output: str) -> str:
        """
        시인 모델의 raw 출력에서 <시작>과 <끝> 사이의 순수 시 본문만 추출 (CoT 격리).
        """
        match = re.search(r"<시작>(.*?)<끝>", raw_poet_output, re.DOTALL)
        if match:
            return match.group(1).strip()
        # 특수 태그가 유실되었을 경우의 대비 로직
        clean_text = re.sub(r"<[^>]+>", "", raw_poet_output)
        return clean_text.strip()

    def _extract_scratchpad(self, raw_poet_output: str) -> str:
        """
        시인의 raw 출력에서 <시작> 태그 이전의 사고 흐름(CoT)만 분리.
        """
        parts = raw_poet_output.split("<시작>")
        if len(parts) > 1:
            return parts[0].strip()
        return ""

    def _prepare_critic_payload(self, poem_text: str) -> Dict[str, Any]:
        """
        비평가 에이전트를 위한 컨텍스트 구성: 시인의 CoT는 제거하고 시와 창작 노트만 주입.
        """
        return {
            "creative_notes": {
                "씨앗이미지": self.creative_notes.get("씨앗이미지"),
                "금지목록": self.creative_notes.get("금지목록"),
                "형식결정": self.creative_notes.get("형식결정")
            },
            "poem": poem_text
        }

    def _prepare_reader_payload(self, poem_text: str) -> Dict[str, Any]:
        """
        독자 에이전트를 위한 컨텍스트 구성: 창작 노트와 시인의 CoT를 모두 감추고 오직 시 본문만 제공.
        """
        return {
            "poem": poem_text
        }

    # ==========================================
    # 2. 역할 오염 방지용 포스트 필터링 (Post-Filtering)
    # ==========================================

    def _validate_critic_output(self, critic_text: str) -> bool:
        """
        비평가가 시인의 영역을 침범하여 구체적인 대체 시구나 행을 제시했는지 검증.
        따옴표 기호("") 내에 4어절 이상의 한국어 문장이 연속되거나 대안 제시형 종결어미가 확인될 시 기각.
        """
        # 따옴표 안의 구절이 긴 경우 (시구 제안으로 간주)
        quoted_phrases = re.findall(r'["\'“‘]([^"\'”’]{15,})["\'”’]', critic_text)
        for phrase in quoted_phrases:
            if len(phrase.split()) >= 4:
                return False
                
        # 대안 제시 전형 패턴
        banned_patterns = [
            r"대신에?\s+['\"].+?['\"](?:을|를)?\s+제안",
            r"['\"].+?['\"](?:으)?로\s+(?:수정|교체|변경)",
            r"['\"].+?['\"](?:이|가)\s+더\s+자연스럽"
        ]
        for pattern in banned_patterns:
            if re.search(pattern, critic_text):
                return False
        return True

    def _validate_reader_output(self, reader_text: str) -> bool:
        """
        독자 응답에 전문 비평 어휘가 포착되는지 검사하여 주관적 감상 무결성 유지.
        """
        banned_terms = ["메타포", "은유법", "주제 의식", "상호텍스트", "포스트모던", "알레고리", "의인화", "병치"]
        for term in banned_terms:
            if term in reader_text:
                return False
        return True

    # ==========================================
    # 3. 루프 제어 및 중단 조건 감지
    # ==========================================

    def _detect_deadlock(self, history: List[PoemRevisionState]) -> bool:
        """
        교착 상태 감지: 최근 2회 이상의 수정 라운드 동안 시 본문에 변화가 없는지 비교.
        """
        if len(history) < 2:
            return False
        
        last_poem = history[-1].poet_output.poem_text
        prev_poem = history[-2].poet_output.poem_text
        return last_poem == prev_poem

    def _check_termination_criteria(
        self,
        critic_out: CriticOutput,
        reader_out: ReaderOutput
    ) -> bool:
        """
        동적 종료 기준 평가: 비평 심각도가 경미하고 독자의 부정적 흐름 장애(추천 여부)가 발견되지 않을 시 즉시 안착.
        """
        is_critic_ok = critic_out.severity == "경미"
        is_reader_ok = reader_out.recommend is True
        return is_critic_ok and is_reader_ok

    # ==========================================
    # 4. 에이전트 개별 호출 래퍼
    # ==========================================

    async def _call_poet(self, persona: PersonaType, system_prompt: str, user_payload: Dict[str, Any]) -> PoetOutput:
        # 실제 환경에서는 LLM API 호출 및 JSON 파싱 수행
        # 여기서는 동작 뼈대만 명시
        raw_response = "[Mode: Medium]\n<사전 설계>...\n<시작>\n잠든 남자의 손등 위로\n마지막 기류가 얹힌다\n<끝>"
        poem_text = self._strip_cot_trace(raw_response)
        scratchpad = self._extract_scratchpad(raw_response)
        
        return PoetOutput(
            persona=persona,
            round_num=user_payload.get("round_num", 0),
            raw_response=raw_response,
            poem_text=poem_text,
            scratchpad=scratchpad,
            decision="Accept",
            explanation="비평 반영하여 기류 이미지로 수정"
        )

    async def _call_critic(self, user_payload: Dict[str, Any]) -> CriticOutput:
        # 비평가 호출 및 포스트 필터링 루프
        raw_critic = '{"severity": "경미", "summary": "금지어 우회 양호", "flaws": []}'
        # _validate_critic_output(raw_critic) 검증 생략
        critic_json = json.loads(raw_critic)
        return CriticOutput(
            round_num=user_payload.get("round_num", 0),
            raw_response=raw_critic,
            flaws=critic_json["flaws"],
            severity=critic_json["severity"],
            summary=critic_json["summary"]
        )

    async def _call_reader(self, user_payload: Dict[str, Any]) -> ReaderOutput:
        # 독자 호출 및 포스트 필터링 루프
        raw_reader = '{"residual_image": "남자의 잠", "cognitive_friction": "없음", "emotional_tone": "고요함", "recommend": true}'
        reader_json = json.loads(raw_reader)
        return ReaderOutput(
            round_num=user_payload.get("round_num", 0),
            raw_response=raw_reader,
            residual_image=reader_json["residual_image"],
            cognitive_friction=reader_json["cognitive_friction"],
            emotional_tone=reader_json["emotional_tone"],
            recommend=reader_json["recommend"]
        )

    # ==========================================
    # 5. 오케스트레이션 실행 메인 루프
    # ==========================================

    async def run_refinement_loop(
        self,
        persona: PersonaType,
        initial_raw_output: str
    ) -> List[PoemRevisionState]:
        """
        단일 시인 페르소나에 대한 피드백-수정 루프 실행.
        """
        history: List[PoemRevisionState] = []
        
        # Round 0 상태 기록
        init_poem = self._strip_cot_trace(initial_raw_output)
        init_scratch = self._extract_scratchpad(initial_raw_output)
        poet_out = PoetOutput(
            persona=persona,
            round_num=0,
            raw_response=initial_raw_output,
            poem_text=init_poem,
            scratchpad=init_scratch,
            decision="Accept",
            explanation="초안 생성 완료"
        )
        history.append(PoemRevisionState(round_num=0, poet_output=poet_out))
        
        for r in range(1, self.max_rounds + 1):
            current_poem = history[-1].poet_output.poem_text
            
            # [격리 실행] 1. 비평가 호출 (CoT 격리)
            critic_payload = self._prepare_critic_payload(current_poem)
            critic_payload["round_num"] = r
            critic_out = await self._call_critic(critic_payload)
            
            # [격리 실행] 2. 독자 호출 (CoT & 창작노트 차단)
            reader_payload = self._prepare_reader_payload(current_poem)
            reader_payload["round_num"] = r
            reader_out = await self._call_reader(reader_payload)
            
            # 동적 종료 검사
            if self._check_termination_criteria(critic_out, reader_out):
                # 최종 라운드에 비평가/독자 반응만 매핑 후 조기 종료
                history[-1].critic_output = critic_out
                history[-1].reader_output = reader_out
                break
                
            # 피드백 이력 압축 (Sliding Window 요약 구성)
            condensed_feedback = self._compress_feedback_history(history, r)
            
            # 3. 시인 페르소나 호출 (피드백 전달 및 수정 지시)
            poet_payload = {
                "round_num": r,
                "creative_notes": self.creative_notes,
                "previous_poem": current_poem,
                "feedback_history": condensed_feedback,
                "current_feedback": {
                    "critic": critic_out.summary,
                    "reader": reader_out.residual_image
                }
            }
            
            # 시인 수정 모델 호출
            poet_out = await self._call_poet(persona, f"poet_{persona.value}", poet_payload)
            
            # 상태 기록
            state = PoemRevisionState(
                round_num=r,
                poet_output=poet_out,
                critic_output=critic_out,
                reader_output=reader_out
            )
            history.append(state)
            
            # 교착 상태 감지 시 강제 중단
            if self._detect_deadlock(history):
                break
                
        return history

    def _compress_feedback_history(self, history: List[PoemRevisionState], current_round: int) -> List[str]:
        """
        t-2 라운드 이전의 모든 세부 피드백을 한 줄 요약 문자열로 축소하여 토큰 팽창을 방어.
        """
        summary_list = []
        for state in history:
            if state.round_num < current_round - 1:
                summary_list.append(
                    f"Round {state.round_num} 수정내역: "
                    f"의사결정={state.poet_output.decision}, "
                    f"사유={state.poet_output.explanation}"
                )
        return summary_list

    # ==========================================
    # 6. 외부 판정 및 DPO 데이터셋 변환
    # ==========================================

    async def evaluate_and_select_winner(
        self,
        final_states: Dict[PersonaType, List[PoemRevisionState]]
    ) -> JudgeOutput:
        """
        각 페르소나가 도출한 최종 수정본들을 외부 판정관(Judge)에게 전송하여 비교 평가.
        """
        judge_payload = {}
        for persona, states in final_states.items():
            last_state = states[-1]
            judge_payload[persona.value] = {
                "scratchpad": last_state.poet_output.scratchpad,
                "poem": last_state.poet_output.poem_text
            }
        
        # 외부 대형 모델 API 호출 (여기서는 모의 응답)
        # judge_response = await self.client.call(model="claude-3-5-sonnet", prompt=...)
        
        return JudgeOutput(
            winner=PersonaType.MODERNIST,
            reasoning="전통 서정성을 비틀어 기하학적 이미지로 재해석한 설계와 시구의 필연성이 훌륭함.",
            loser_analyses={
                PersonaType.TRADITIONALIST: "이미지는 유려하나 금지 단어를 우회한 방식이 다소 평이함.",
                PersonaType.AVANT_GARDE: "파격의 형식이 과도하여 정서적 울림이 분산됨."
            }
        )

    def export_dpo_pairs(
        self,
        final_states: Dict[PersonaType, List[PoemRevisionState]],
        judge_out: JudgeOutput
    ) -> List[Dict[str, Any]]:
        """
        외부 판정 결과를 바탕으로 chosen-rejected DPO 학습 선호 쌍을 조립.
        """
        dpo_pairs = []
        winner_state = final_states[judge_out.winner][-1].poet_output
        
        chosen_data = {
            "scratchpad": winner_state.scratchpad,
            "poem": winner_state.poem_text,
            "persona": judge_out.winner.value
        }
        
        rejected_list = []
        for persona, states in final_states.items():
            if persona == judge_out.winner:
                continue
            loser_state = states[-1].poet_output
            rejected_list.append({
                "scratchpad": loser_state.scratchpad,
                "poem": loser_state.poem_text,
                "persona": persona.value
            })
            
        # 3-way 비교 결과에서 chosen-rejected 쌍 2개 추출
        for rejected in rejected_list:
            pair = {
                "prompt": self.creative_notes.get("씨앗이미지", ""),
                "chosen": chosen_data,
                "rejected": rejected,
                "judge_reasoning": judge_out.reasoning
            }
            dpo_pairs.append(pair)
            
        return dpo_pairs


# ==========================================
# 7. 실행 흐름 예제
# ==========================================

async def main():
    creative_notes = {
        "씨앗이미지": "한밤중에 열려 있는 24시간 무인 세탁소 안에서 춤을 추는 회전 세탁조와 유리창의 빗방울.",
        "금지목록": ["쓸쓸함", "외로움", "빗소리", "돌아간다"],
        "형식결정": {"길이": "10행 내외", "행갈이원칙": "세탁기의 회전 주기에 따라 불규칙 분절", "시제": "현재진행형"}
    }
    
    harness = PoetryCouncilHarness(creative_notes=creative_notes, max_rounds=3)
    
    # 각 페르소나별 초기 초안 생성 (실제 API 비동기 병렬 요청)
    # 여기서는 임시 Mock 문자열
    persona_drafts = {
        PersonaType.TRADITIONALIST: "<사전 설계>...\n<시작>\n세탁조가 돈다\n유리창엔 비가 내리고\n남자는 기다린다\n<끝>",
        PersonaType.MODERNIST: "<사전 설계>...\n<시작>\n원통형 드럼이 그리는 각도\n원심력에 붙잡힌 양말 하나\n빗방울이 유리벽을 쪼갠다\n<끝>",
        PersonaType.AVANT_GARDE: "<사전 설계>...\n<시작>\n탁 탁 탁\n돌아가 돌아 회전하는 유\n리창\n빗 물방울들이 흩어진다\n<끝>"
    }
    
    # 3개 페르소나에 대한 피드백 루프를 비동기 병렬로 동시 수행
    tasks = []
    for persona, draft in persona_drafts.items():
        tasks.append(harness.run_refinement_loop(persona, draft))
        
    results = await asyncio.gather(*tasks)
    final_states = {
        PersonaType.TRADITIONALIST: results[0],
        PersonaType.MODERNIST: results[1],
        PersonaType.AVANT_GARDE: results[2]
    }
    
    # 외부 판정
    judge_out = await harness.evaluate_and_select_winner(final_states)
    print(f"★ 우승 페르소나: {judge_out.winner}")
    
    # DPO 데이터셋용 선호 쌍 생성
    dpo_data = harness.export_dpo_pairs(final_states, judge_out)
    print(json.dumps(dpo_data, indent=2, ensure_ascii=False))

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Antigravity Harness와의 연동 방안

Google Antigravity Harness 에이전트 시스템에서 이 앙상블 수정 및 판정 파이프라인을 조율하는 경우, 다음과 같은 구조적 이점을 갖는다.

1. **병렬 Skill 실행:**
   - 3개 페르소나의 초안 작성과 각각의 `run_refinement_loop` 내 비평가/독자 동시 호출 단계를 Antigravity의 비동기 병렬 태스크 실행 환경에서 고속 처리하여 앙상블의 가장 큰 단점인 대기 지연(Latency) 문제를 극복한다.
2. **선택적 컨텍스트 필터 지원:**
   - Antigravity가 제공하는 `context_filter` 또는 API 호출 데코레이터를 통해, 비평가 호출 시 Poet의 `<scratchpad>`를 제거하는 전처리 규칙과 독자 호출 시 창작 노트를 누락시키는 격리 필터를 선언적(Declarative)으로 선언하여 코드를 간소화한다.
3. **학습 영속화:**
   - DPO 데이터셋이 성공적으로 디스크에 쌓이면 Antigravity의 fine-tuning 도구를 이용해 지정된 시기에 맞추어 Poet 모델에 대한 DPO 최적화 사이클을 자동 호출할 수 있다.

---

## 미결 사항

- [ ] 세 가지 시인 페르소나의 DPO 선택율(chosen ratio) 격차가 누적될 때, 파인튜닝된 단일 모델의 미학적 정체성이 특정 한쪽 페르소나로 편중(Collapse)되는 현상을 제어하기 위한 동적 손실 가중치(Dynamic Loss Weighting) 보정 설계.
- [ ] 비평가/독자의 포스트 필터 검증이 반려되어 재시도가 트리거될 시, 무한 루프에 빠지거나 지나치게 긴 추론 시간 지연을 야기하는 현상을 막기 위한 소프트 페널티(Soft Penalty - 프롬프트를 통한 경고 누적식 재출력 지시) 최적 피드백 한계 정의.
- [ ] 슬라이딩 윈도우 피드백 요약 단계에서, 초기 라운드 비평 중 수정 계획에 결정적 영향을 미쳤던 핵심 맥락(예: 형식 원칙 재조정 등)이 요약 과정에서 유실되는 현상을 막기 위해, 단순 시간 순 요약이 아닌 미학적 중요도 기반 필터링 알고리즘 설계.

---

## 연결 문서

- [multi_agent_council.md](multi_agent_council.md) — 앙상블 및 DPO 선호 학습 루프 전체 설계
- [cot_creative_notes.md](cot_creative_notes.md) — 창작 노트 구조 및 시인 입력 스키마
- [output_goals.md](output_goals.md) — 미학적 가이드라인 및 필터링 기준
- [iterative_refinement.md](iterative_refinement.md) — 단일 모델 자가 비평 루프에 관한 선행 설계
