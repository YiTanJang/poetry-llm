---
type: Design
title: 에이전트 하네스 설계
description: Poetry Council을 구현하는 Python 하네스 — 모델 선택, 컨텍스트 관리, 상태 추적, Antigravity 연동 가능성.
tags: [generation, multi-agent, council, harness]
timestamp: 2026-06-27T00:00:00Z
---

# 에이전트 하네스 설계

## 핵심 설계 질문에 대한 답변

### Q1. 각 에이전트는 같은 모델인가, 다른 모델인가?

**기본 권장 구성: 역할 프롬프트가 다른 같은 모델 (파인튜닝 전 단계)**

Council의 각 에이전트는 동일한 기반 모델(예: Claude claude-sonnet-4-6)에 역할별 시스템 프롬프트를 주입하는 방식으로 운영한다.
이유:
- 파인튜닝 이전 단계에서는 역할별 모델을 별도로 학습시키는 비용이 정당화되지 않는다.
- 역할 프롬프트만으로도 비평가/독자/시인의 관점 분리가 충분히 가능하다는 것을 실험으로 먼저 검증한다.

**예외: 비평 역할에 더 큰 모델 사용 (선택적)**

악마의 변호인과 비평가는 더 강한 추론 능력이 요구된다.
비용 대비 효과를 따진다면, 이 두 에이전트만 더 큰 모델(예: claude-opus-4)로 교체하는 것을 고려한다.

```python
AGENT_MODELS = {
    "poet": "claude-sonnet-4-6",           # 기본 모델
    "critic": "claude-opus-4",             # 선택적으로 큰 모델
    "editor": "claude-sonnet-4-6",
    "reader": "claude-haiku-4-5",          # 단순 반응 → 소형 모델로 충분
    "devil": "claude-opus-4",              # 공격적 추론 → 큰 모델
}
```

파인튜닝 단계에 진입하면, 시인 역할은 poetry-finetuned 모델로 교체하고
비평 역할은 범용 모델 그대로 유지하는 분리가 가능하다.

---

### Q2. 각 라운드에서 컨텍스트를 어떻게 전달하는가?

**전략: 역할에 따라 선택적 이력 전달**

모든 에이전트에게 전체 이력을 전달하면 컨텍스트 윈도우가 빠르게 소진된다.
각 에이전트는 자신의 역할 수행에 필요한 부분만 받는다.

| 에이전트 | 전달 내용 |
|---------|---------|
| 시인 | 창작 노트 + 이전 버전 시 + 이전 라운드 비평 요약 |
| 비평가 | 창작 노트 + 현재 시 + 이전 비평 (개선 여부 판단용) |
| 편집자 | 창작 노트 + 현재 시 + 비평가 출력 (전체) |
| 독자 | 현재 시만 (창작 노트 없음 — 의도적 격리) |
| 악마의 변호인 | 창작 노트 + 현재 시 + 이전 비평 이력 (공격 방향 탐색용) |

**요약 생성:** 라운드가 3 이상으로 길어지면, 비평 이력 전체를 전달하는 대신
오케스트레이터가 `비평_요약` 필드를 생성해 전달한다.

```python
def summarize_history(history: list[dict], max_rounds: int = 2) -> str:
    """
    라운드 수가 max_rounds를 초과하면 초기 라운드를 요약 문자열로 압축.
    최근 max_rounds 개 라운드는 전체 전달.
    """
    if len(history) <= max_rounds:
        return history
    
    old_rounds = history[:-max_rounds]
    summary = f"[초기 {len(old_rounds)}라운드 요약] "
    summary += " | ".join(
        f"R{r['round']}: {r['critic']['총평'][:60]}" for r in old_rounds
    )
    return [{"요약": summary}] + history[-max_rounds:]
```

---

### Q3. 컨텍스트 윈도우 관리

**문제:** 창작 노트 + 시 전체 이력 + 에이전트 출력이 누적되면 빠르게 128K를 초과한다.

**대응 전략:**

1. **독자 에이전트는 항상 최소 컨텍스트:** 창작 노트 없이 시 텍스트만 전달. 독자 에이전트가 가장 낮은 토큰을 소비한다.

2. **비평 이력 압축:** 위의 `summarize_history` 로직으로 오래된 라운드를 요약으로 대체한다.

3. **에이전트 출력 필드 선택적 전달:** 편집자의 `대안제안` 전체가 다음 라운드 시인에게 필요하지 않다. 시인에게는 "채택된 대안"만 전달한다.

4. **창작 노트 트림:** 창작 노트의 `금지목록`과 `씨앗이미지`는 항상 전달하되, `개념탐색` 전체는 Round 0 이후 트림한다.

```python
def trim_creative_notes_for_round(notes: dict, round_num: int) -> dict:
    if round_num == 0:
        return notes
    # Round 1 이후: 핵심 필드만 유지
    return {
        "씨앗이미지": notes["씨앗이미지"],
        "금지목록": notes["금지목록"],
        "형식결정": notes["형식결정"],
        # 개념탐색은 제거
    }
```

---

### Q4. 상태 관리: Python 데이터 구조

Council의 전체 상태는 `CouncilState` dataclass로 관리한다.

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class AgentOutput:
    agent: str          # "poet" | "critic" | "editor" | "reader" | "devil"
    round_num: int
    raw_output: dict    # 각 에이전트의 JSON 출력 전체
    timestamp: str

@dataclass
class RoundState:
    round_num: int
    poem_version: str           # 이 라운드의 시 텍스트
    creative_notes: dict        # 이 라운드의 창작 노트 (수정 반영)
    agent_outputs: list[AgentOutput] = field(default_factory=list)
    termination_check: Optional[dict] = None  # 조기 종료 판단 결과

@dataclass
class CouncilState:
    session_id: str
    rounds: list[RoundState] = field(default_factory=list)
    final_poem: Optional[str] = None
    final_round: Optional[int] = None
    termination_reason: Optional[str] = None  # "criteria_met" | "max_rounds" | "deadlock"
    novelty_scores: Optional[dict] = None     # output_goals.md의 필터 결과
```

---

## 구체적 구현 (Python pseudo-code)

```python
import anthropic
import json
from dataclasses import dataclass, field
from typing import Optional

# 시스템 프롬프트는 별도 파일로 분리 권장
SYSTEM_PROMPTS = {
    "poet": open("prompts/poet.txt").read(),
    "critic": open("prompts/critic.txt").read(),
    "editor": open("prompts/editor.txt").read(),
    "reader": open("prompts/reader.txt").read(),
    "devil": open("prompts/devil.txt").read(),
}

AGENT_MODELS = {
    "poet": "claude-sonnet-4-6",
    "critic": "claude-sonnet-4-6",
    "editor": "claude-sonnet-4-6",
    "reader": "claude-haiku-4-5",
    "devil": "claude-sonnet-4-6",
}


class PoetryCouncil:
    def __init__(
        self,
        creative_notes: dict,
        max_rounds: int = 3,
        novelty_filter=None,
    ):
        self.client = anthropic.Anthropic()
        self.creative_notes = creative_notes
        self.max_rounds = max_rounds
        self.novelty_filter = novelty_filter
        self.state = CouncilState(session_id=generate_session_id())

    def _call_agent(self, agent: str, user_message: dict) -> dict:
        """단일 에이전트 API 호출. JSON 출력을 파싱해 반환."""
        response = self.client.messages.create(
            model=AGENT_MODELS[agent],
            max_tokens=2048,
            system=SYSTEM_PROMPTS[agent],
            messages=[
                {"role": "user", "content": json.dumps(user_message, ensure_ascii=False)}
            ],
        )
        return json.loads(response.content[0].text)

    def run_round_0(self) -> RoundState:
        """Round 0: 시인이 창작 노트를 기반으로 초안 생성."""
        poet_input = {
            "창작노트": self.creative_notes,
            "이전버전": None,
            "비평이력": [],
        }
        poet_output = self._call_agent("poet", poet_input)

        round_state = RoundState(
            round_num=0,
            poem_version=poet_output["신규버전"],
            creative_notes=self.creative_notes,
            agent_outputs=[AgentOutput("poet", 0, poet_output, now())],
        )
        self.state.rounds.append(round_state)
        return round_state

    def run_round(self, round_num: int) -> RoundState:
        """Round 1~N: 전체 Council 세션 실행."""
        prev_round = self.state.rounds[-1]
        current_poem = prev_round.poem_version
        trimmed_notes = trim_creative_notes_for_round(self.creative_notes, round_num)
        history_summary = summarize_history(
            [r.__dict__ for r in self.state.rounds], max_rounds=2
        )

        # 1. 비평가 + 악마의 변호인 (독립적으로, 병렬 실행 가능)
        critic_input = {
            "창작노트": trimmed_notes,
            "현재시": current_poem,
            "라운드": round_num,
        }
        devil_input = {
            "창작노트": trimmed_notes,
            "현재시": current_poem,
            "라운드": round_num,
            "이전비평이력": history_summary,
        }
        # 실제 구현에서는 asyncio.gather 또는 ThreadPoolExecutor로 병렬화
        critic_output = self._call_agent("critic", critic_input)
        devil_output = self._call_agent("devil", devil_input)

        # 2. 독자 (현재 시만 전달 — 창작 노트 격리)
        reader_input = {"시": current_poem}
        reader_output = self._call_agent("reader", reader_input)

        # 3. 편집자 (비평가 출력 기반 대안 제시)
        editor_input = {
            "창작노트": trimmed_notes,
            "현재시": current_poem,
            "비평지적사항": critic_output["약점목록"],
        }
        editor_output = self._call_agent("editor", editor_input)

        # 4. 시인 (모든 피드백 종합 후 수정)
        poet_input = {
            "창작노트": trimmed_notes,
            "이전버전": current_poem,
            "비평이력": history_summary,
            "이번라운드피드백": {
                "비평가": critic_output,
                "악마의변호인": devil_output,
                "독자": reader_output,
                "편집자_대안": editor_output,
            },
        }
        poet_output = self._call_agent("poet", poet_input)

        agent_outputs = [
            AgentOutput("critic", round_num, critic_output, now()),
            AgentOutput("devil", round_num, devil_output, now()),
            AgentOutput("reader", round_num, reader_output, now()),
            AgentOutput("editor", round_num, editor_output, now()),
            AgentOutput("poet", round_num, poet_output, now()),
        ]

        round_state = RoundState(
            round_num=round_num,
            poem_version=poet_output["신규버전"],
            creative_notes=trimmed_notes,
            agent_outputs=agent_outputs,
            termination_check=self._check_termination(
                critic_output, devil_output, reader_output
            ),
        )
        self.state.rounds.append(round_state)
        return round_state

    def _check_termination(
        self, critic_out: dict, devil_out: dict, reader_out: dict
    ) -> dict:
        """조기 종료 조건을 평가한다."""
        criteria = {
            "critic_severity_ok": critic_out.get("심각도") == "경미",
            "devil_not_rejected": "자격 없음" not in devil_out.get("판결", ""),
            "reader_not_negative": reader_out.get("추천여부", "").startswith("아니오") is False,
        }
        return {
            "should_terminate": all(criteria.values()),
            "criteria": criteria,
        }

    def _check_deadlock(self) -> bool:
        """3라운드 연속 동일 행 미수정 — 교착 상태 감지."""
        if len(self.state.rounds) < 4:
            return False
        last_three = [r.poem_version for r in self.state.rounds[-3:]]
        return last_three[0] == last_three[1] == last_three[2]

    def select_final(self) -> str:
        """최종 시 선택: Novelty 필터 통과 + 비평 기준 충족 버전 중 최선."""
        candidates = []
        for round_state in self.state.rounds[1:]:  # Round 0 제외
            term = round_state.termination_check
            if term and term["should_terminate"]:
                poem = round_state.poem_version
                novelty = (
                    self.novelty_filter(poem) if self.novelty_filter else {"passed": True}
                )
                if novelty["passed"]:
                    candidates.append((round_state.round_num, poem))

        if candidates:
            # 조기 종료 조건을 처음 충족한 버전 선택
            return candidates[0][1]

        # 조기 종료 조건 미충족 — 마지막 버전 반환
        return self.state.rounds[-1].poem_version

    def run(self) -> CouncilState:
        """전체 Council 실행 진입점."""
        self.run_round_0()

        for round_num in range(1, self.max_rounds + 1):
            round_state = self.run_round(round_num)

            # 교착 상태 감지
            if self._check_deadlock():
                self.state.termination_reason = "deadlock"
                break

            # 조기 종료
            if round_state.termination_check and round_state.termination_check["should_terminate"]:
                self.state.termination_reason = "criteria_met"
                break
        else:
            self.state.termination_reason = "max_rounds"

        self.state.final_poem = self.select_final()
        self.state.final_round = self.state.rounds.index(
            next(r for r in self.state.rounds if r.poem_version == self.state.final_poem)
        )
        return self.state
```

---

## 실행 예시

```python
creative_notes = {
    "씨앗이미지": "지하철에서 잠든 남자의 손 위에 낙엽이 한 장 떨어졌다",
    "개념탐색": ["낙엽+잠 → 계절과 의식의 교차", "무지 → 비극인가 축복인가"],
    "다루고싶은것": "직접 말하지 않고 이 순간 자체가 말하게 하고 싶다",
    "형식결정": {"길이": "짧게", "행갈이원칙": "시선 이동에 따라", "시제": "현재형"},
    "금지목록": ["직유", "감정 직접 서술", "결론"],
}

council = PoetryCouncil(
    creative_notes=creative_notes,
    max_rounds=3,
    novelty_filter=run_novelty_filters,   # output_goals.md 필터 구현체
)

result = council.run()
print(result.final_poem)
print(f"최종 선택: Round {result.final_round}, 종료 이유: {result.termination_reason}")
```

---

## Antigravity Harness와의 연동 가능성

### Google Antigravity Agent Harness로 Council 구현

Antigravity harness는 에이전트를 "skills" 단위로 정의하고,
orchestrator가 skill을 순서대로 또는 병렬로 호출하는 구조다.

Poetry Council을 Antigravity로 구현한다면:

**각 Council 에이전트를 하나의 Skill로 정의:**

```yaml
# AGENTS.md 스타일의 skill 정의

skills:
  - name: poet
    description: |
      창작 노트를 기반으로 시 초안을 작성하거나 비평을 반영해 수정한다.
      입력: 창작 노트, 이전 버전, 비평 이력
      출력: 포지션(수용/방어), 신규 버전, 변경 사항 설명
    model: claude-sonnet-4-6
    system_prompt_file: prompts/poet.txt
    output_schema: schemas/poet_output.json

  - name: critic
    description: |
      시의 미학적 약점을 행 단위로 지적한다.
      창작 노트 대비 충실도를 평가한다.
      입력: 창작 노트, 현재 시, 라운드 번호
      출력: 약점 목록, 심각도, 총평
    model: claude-sonnet-4-6
    system_prompt_file: prompts/critic.txt
    output_schema: schemas/critic_output.json

  - name: editor
    description: |
      비평가의 지적을 구체적 언어 대안으로 변환한다.
      시인의 목소리를 유지하는 제안에 집중한다.
    model: claude-sonnet-4-6
    system_prompt_file: prompts/editor.txt

  - name: reader
    description: |
      창작 노트 없이 시 텍스트만 읽고 일반 독자로서 반응한다.
      전문 용어 없이 직관적 반응을 서술한다.
    model: claude-haiku-4-5
    system_prompt_file: prompts/reader.txt

  - name: devil
    description: |
      시가 왜 실패작인지를 최대한 공격적으로 비판한다.
      novelty 주장, 창작 노트 충실도, 기존 코퍼스 중복을 공격한다.
    model: claude-sonnet-4-6
    system_prompt_file: prompts/devil.txt

orchestration:
  type: sequential_with_parallel_sections
  rounds:
    - name: round_0
      sequence:
        - skill: poet
          inputs: [creative_notes]

    - name: council_round
      repeat_until:
        condition: termination_criteria_met
        max_iterations: 3
      sequence:
        - parallel:
            - skill: critic
            - skill: devil
        - skill: reader
          context_filter: poem_only    # 창작 노트 격리
        - skill: editor
          depends_on: [critic]
        - skill: poet
          depends_on: [critic, devil, reader, editor]
      context_management:
        history_compression_after: 2  # 2라운드 이후 요약 압축
```

**Antigravity 연동의 장점:**
- skill 재사용: 비평가 skill을 Poetry Council 외 다른 파이프라인에서도 활용 가능.
- 병렬 실행: `parallel` 섹션이 critic + devil을 동시에 실행해 지연을 줄인다.
- 상태 추적: Antigravity의 run 이력이 `CouncilState`와 동일한 역할을 한다.

**Antigravity 연동의 제약:**
- context_filter (독자 에이전트의 창작 노트 격리) 같은 커스텀 로직은 harness 레벨에서 별도 구현 필요.
- 조기 종료 조건 (`termination_criteria_met`)을 harness가 지원하는 방식으로 표현해야 한다.
  → Antigravity가 조건부 반복을 지원하지 않으면, 외부 orchestrator 레이어를 유지해야 한다.

---

## 미결 사항 및 실험 우선순위

| 질문 | 우선순위 | 비고 |
|------|---------|------|
| 역할 프롬프트만으로 비평가/독자/시인 분리가 실제로 작동하는가? | 높음 | 첫 번째 실험 대상 |
| 독자 에이전트가 창작 노트 없이도 유의미한 피드백을 주는가? | 높음 | 격리 설계의 타당성 검증 |
| 악마의 변호인이 시인을 잘못된 방향으로 끌고 가는 경우가 얼마나 자주 발생하는가? | 중간 | 악마 억제 로직 필요 여부 |
| 비평가와 편집자를 합치면 (비평 + 대안 동시 제시) 품질 차이가 있는가? | 낮음 | Council 단순화 옵션 |
| Haiku 모델로 독자 에이전트를 운영해도 충분한가? | 중간 | 비용 최적화 |

---

## 연결 문서

- [multi_agent_council.md](multi_agent_council.md) — Council 구성원 설계 및 오케스트레이션 로직
- [cot_creative_notes.md](cot_creative_notes.md) — 창작 노트 구조 (하네스 입력)
- [output_goals.md](output_goals.md) — Novelty 필터 구현체 (select_final에서 사용)
- [iterative_refinement.md](iterative_refinement.md) — 단일 모델 자기 비평 루프 (Council 이전 단계)
