# Antigravity 만능 프롬프트

아래 코드블록 내용만 복사해서 Antigravity에 붙여넣으면 됨.

---

```
You are working on a Korean poetry LLM research wiki at D:\Documents\HomeLab\OKFPoetryproject.

## Step 0 — Read project context first

1. Read AGENTS.md
2. Read poetry-llm/index.md
3. Read poetry-llm/editing-system/okf_format.md  ← editing rules, markers, handoff format
4. Read every section's index.md to get a full picture of what exists

Project goal: fine-tune a 20~40B model (full fine-tuning, 2×A100 80GB) to generate aesthetically novel Korean contemporary poetry. Base model not yet decided — current leading candidates are Gemma 4 27B/31B. See `poetry-llm/model/model_selection.md`.

---

## Step 1 — Git setup

```bash
git checkout master
git pull
git checkout -b task/YYYYMMDD-brief-description   # today's date + topic
git status   # must be clean
```

---

## Step 2 — Plan 15 tasks before starting

Survey every section and every file. Build a task list of exactly 15 valuable improvements:
- Read all existing .md files and their ## 미결 사항 sections
- Run `grep -r "[TODO]" poetry-llm/` and `grep -r "> 탐색중" poetry-llm/` to find open work
- Identify: missing docs referenced but not written, stale info, vague sections needing concrete examples, open questions that can now be answered
- Assign each task to exactly one file (no two tasks touch the same file)
- Spread tasks across sections: data/, preprocessing/, model/, generation/, evaluation/, research/

**Task type quota — strictly enforced:**

Each task must be labeled with one of these types. Enforce the minimum counts:

| Task type | Min | Examples |
|-----------|-----|---------|
| A. 누락 파일 작성 | 3 | 참조만 되고 아직 없는 파일, 스텁 확장 |
| B. 설계 결정 구체화 | 4 | 수치·근거 없는 섹션에 구체적 수식·기준·예시 추가 |
| C. 오래된 정보 수정 | 2 | 이미 결정된 내용과 충돌하는 stale 내용 교정 |
| D. 미결 사항 구조화 | 최대 3 | 기존 [TODO]에 서브 질문 추가 — **전체의 20% 이하** |
| E. [User Review] 격상 | 나머지 | [TODO] 중 에이전트 독단 불가 항목을 [User Review]로 교체 |

**D 타입(서브 질문 추가)이 15개 중 3개를 초과하면 계획을 다시 짜라.** 이것이 가장 쉬운 작업이지만 가장 낮은 가치를 생산한다.

Write out the full 15-task plan before spawning any agents. Format:

```
Task 01 | type:A | section | file.md | what to do (새 파일 작성 or 스텁 확장)
Task 02 | type:B | section | file.md | what to do (설계 결정 구체화)
...
Task 15 | type:E | section | file.md | what to do ([User Review] 격상)
```

Task type 분포를 계획 상단에 요약:
```
Type A: X개, Type B: X개, Type C: X개, Type D: X개 (3 이하), Type E: X개
```

---

## Step 3 — Execute in 5 cycles of 3 parallel agents

Run 5 cycles. Each cycle:
1. Pick the next 3 tasks from your plan (tasks must be in DIFFERENT files)
2. Spawn 3 parallel sub-agents, one per task
3. Wait for all 3 to finish and commit
4. Check git log to confirm 3 new commits landed
5. Move to the next cycle

Do not stop between cycles. Complete all 5 cycles before the final merge.

---

## Rules each sub-agent MUST follow

### Git rules

```bash
# Confirm you are on the task branch first:
git branch   # must show task/YYYYMMDD-... as current

# Stage only your assigned file:
git add poetry-llm/<section>/<your-file.md>

# Commit:
git commit -m "[section] what you did"

# NEVER run:
# git reset, git rebase, git push --force
# git checkout master, git merge, git stash pop
# git add -A, git add .
```

### OKF format rules

Every file you create or edit must have this frontmatter:

```yaml
---
type: <Design | Reference | Playbook | Data Source | Open Questions | Bundle Log | ...>
title: <Korean title>
description: <현재 상태와 핵심 결정을 담은 구체적 TL;DR — 예: "모델 선택 (진행중) - Gemma 27B 우선 검토, 풀 파인튜닝 확정">
tags: [tag1, tag2, tag3]
timestamp: <today's date>T00:00:00Z
---
```

End every file with:
```markdown
## 미결 사항

- [TODO] <genuine open question 1>
- [Ph1] <phase-gated question>
```

**문서 편집 상태 마커:**

섹션 레벨:
- `> 탐색중` — 섹션 전체가 열린 목록. 항목 추가 가능, 기존 항목 삭제 금지.
- 마커 없음 — 확정된 사항. **직접 수정 금지.**

확정된 섹션을 수정해야 할 경우 — 직접 덮어쓰지 말고 하단에 블록 추가:
```markdown
> 수정 제안: [이유] 기존 내용을 X에서 Y로 변경 제안
```
사용자가 승인하면 원본을 업데이트한다.

`## 미결 사항` 항목 앞 마커:
- `[TODO]` — 지금 바로 작업 가능
- `[User Review]` — 사용자 컨펌 필수. 에이전트 스스로 해결 시도 금지, 대기/질문
- `[Ph1]` — Phase 1 완료 후 결정 가능
- `[Ph2]` — Phase 2 완료 후 결정 가능

해결된 미결 사항은 삭제하지 말고 `## 결정 사항` 섹션으로 이동:
```markdown
## 결정 사항

- [해결됨: 2026-06-28] 프롬프트 분리 처리 결정 (이유: CoT와 생성 단계를 나누면 품질 향상)
```

### index.md update rules

If you create a new file, you MUST update that section's index.md.

Link format — bare filename only, no path prefix:
```markdown
| [filename.md](filename.md) | 한 줄 설명 |
```

index.md has NO frontmatter. Do not add frontmatter to index.md files.

---

## Scope rules — Phase 0 is design only

**The single most important rule:** Ask yourself before every sentence you write —
> "Am I documenting a design decision, or am I making one?"

This wiki is in **Phase 0 — 기반 설계**. That means:

**In scope:**
- Documenting decisions that have already been made (record them clearly)
- Writing design docs that are referenced but not yet written
- Adding examples or diagrams that clarify existing decisions
- Surfacing open questions in `## 미결 사항`

**Out of scope — do NOT do these:**
- Writing Python implementation code (classes, functions, full scripts)
- Inventing design decisions not already stated in the existing docs
- Adding numerical thresholds or hyperparameters without evidence
- "Solving" a problem you found in `## 미결 사항` — instead, enrich the question with context and leave it open
- Adding Phase 2/3 features to Phase 0 design docs
- Introducing new components not already in the design

**When you find a problem or gap:**
- Do NOT solve it unsolicited
- Add it to `## 미결 사항` with `[TODO]` or `[User Review]` as appropriate
- If it conflicts with an existing design decision, note the conflict and mark `[User Review]`

---

## What counts as valuable work

**Good (우선순위 순):**
1. 참조는 있으나 파일이 없는 문서 작성 (가장 가치 높음)
2. 스텁 파일을 완전한 설계 문서로 확장
3. 구체적 근거 없이 막연한 섹션에 수치·수식·예시 추가
4. 이미 결정된 내용과 충돌하는 stale 정보 수정
5. 해결된 미결 사항을 `## 결정 사항`으로 이동 (이유 포함)
6. 기존 `[TODO]` 항목에 서브 질문 추가 **(15개 중 최대 3개만 허용, Type D)**

**Bad (do not do):**
- Adding full Python implementations or complete class definitions
- Introducing design decisions not found in existing docs
- Cosmetic rewording with no new information
- Adding obvious caveats or disclaimers
- Writing content that duplicates another file
- Touching files outside your assigned task
- Directly modifying confirmed (non-탐색중) sections
- **서브 질문 추가(Type D)를 15개 중 4개 이상 배정하는 것** ← quota 위반, 계획 재작성 필요

---

## Step 4 — After all 5 cycles (15 commits), merge to master

```bash
git checkout master
git merge task/YYYYMMDD-brief-description
git branch -d task/YYYYMMDD-brief-description
```

If merge conflict: resolve by combining both sides' content, not discarding either.

Add a Handoff log entry to `poetry-llm/log.md`:
```markdown
- **[YYYY-MM-DD] 작업 요약**: (예: 15개 개선 작업)
  - **수정 파일**: (파일 목록)
  - **핵심 결정**: (이번 세션에서 확정된 것)
  - **다음 단계 (Next Steps)**: (다음 세션이 이어받아야 할 것)
```

Final report:
- Full list of 15 tasks completed (section, file, what changed)
- Any new open questions that surfaced
- Confirm merge landed on master (`git log master --oneline -10`)
```
