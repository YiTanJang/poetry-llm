# Antigravity 만능 프롬프트

아래 코드블록 내용만 복사해서 Antigravity에 붙여넣으면 됨.

---

```
You are working on a Korean poetry LLM research wiki at D:\Documents\HomeLab\OKFPoetryproject.

## Step 0 — Read project context first

1. Read AGENTS.md
2. Read poetry-llm/index.md
3. Read every section's index.md to get a full picture of what exists

Project goal: fine-tune Qwen2.5-32B (full fine-tuning, 2×A100 80GB) to generate aesthetically novel Korean contemporary poetry. Wiki uses OKF format — every .md file needs YAML frontmatter: type, title, description, tags, timestamp.

---

## Step 1 — Git setup

```bash
git checkout master
git pull
git checkout -b task/YYYYMMDD-brief-description   # today's date + topic
git status   # must be clean
```

---

## Step 2 — Plan 30 tasks before starting

Survey every section and every file. Build a task list of exactly 30 valuable improvements:
- Read all existing .md files and their ## 미결 사항 sections
- Identify: missing docs referenced but not written, stale info, vague sections needing concrete examples, open questions that can now be answered
- Assign each task to exactly one file (no two tasks touch the same file)
- Spread tasks across all sections: data/, preprocessing/, model/, generation/, evaluation/, research/

Write out the full 30-task plan before spawning any agents. Format:

```
Task 01 | section | file.md | what to do
Task 02 | section | file.md | what to do
...
Task 30 | section | file.md | what to do
```

---

## Step 3 — Execute in 10 cycles of 3 parallel agents

Run 10 cycles. Each cycle:
1. Pick the next 3 tasks from your plan (tasks must be in DIFFERENT files)
2. Spawn 3 parallel sub-agents, one per task
3. Wait for all 3 to finish and commit
4. Check git log to confirm 3 new commits landed
5. Move to the next cycle

Do not stop between cycles. Complete all 10 cycles before the final merge.

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
type: <Design | Reference | Playbook | Pipeline | Prompt Templates | ...>
title: <Korean title>
description: <one sentence in Korean>
tags: [tag1, tag2, tag3]
timestamp: <today's date>T00:00:00Z
---
```

End every file with:
```markdown
## 미결 사항

- <genuine open question 1>
- <genuine open question 2>
- <genuine open question 3>
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
- Adding numerical thresholds or hyperparameters without evidence ("use lr=5e-7" unless it's already decided)
- "Solving" a problem you found in `## 미결 사항` — instead, enrich the question with context and leave it open
- Adding Phase 2/3 features to Phase 0 design docs
- Introducing new components (new agent roles, new evaluation systems) not already in the design

**When you find a problem or gap:**
- Do NOT solve it unsolicited
- Add it to `## 미결 사항` in the relevant file with a clear description
- If it conflicts with an existing design decision, note the conflict in `## 미결 사항`

**The "design vs implement" test:**
- Design = "what we will build and why" → OK
- Pseudocode showing the concept = OK
- Full working code = NOT OK in Phase 0

---

## What counts as valuable work

**Good:**
- Fixing wrong or outdated information
- Writing a design doc that is referenced but not yet written
- Adding pseudocode or examples that clarify a concept (not full implementations)
- Filling in a `## 미결 사항` item with more context or sub-questions
- Expanding a stub file into a full design doc

**Bad (do not do):**
- Adding full Python implementations or complete class definitions
- Introducing design decisions not found in existing docs (especially for multi-agent roles, evaluation systems, or training pipeline components)
- Cosmetic rewording with no new information
- Adding obvious caveats or disclaimers
- Writing content that duplicates another file
- Touching files outside your assigned task
- Changing persona names, role definitions, or pipeline steps that have already been decided

---

## Step 4 — After all 10 cycles (30 commits), merge to master

```bash
git checkout master
git merge task/YYYYMMDD-brief-description
git branch -d task/YYYYMMDD-brief-description
```

If merge conflict: resolve by combining both sides' content, not discarding either.

Final report:
- Full list of 30 tasks completed (section, file, what changed)
- Any new open questions that surfaced
- Confirm merge landed on master (`git log master --oneline -10`)
```
