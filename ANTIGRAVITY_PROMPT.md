# Antigravity 만능 프롬프트

아래 코드블록 내용만 복사해서 Antigravity에 붙여넣으면 됨.

---

```
You are working on a Korean poetry LLM research wiki at D:\Documents\HomeLab\OKFPoetryproject.

## Step 0 — Read project context first

1. Read AGENTS.md
2. Read poetry-llm/index.md

Project goal: fine-tune Qwen2.5-32B (full fine-tuning, 2×A100 80GB) to generate aesthetically novel Korean contemporary poetry. Wiki uses OKF format — every .md file needs YAML frontmatter: type, title, description, tags, timestamp.

---

## Step 1 — Git setup (do this BEFORE spawning sub-agents)

```bash
git checkout master
git pull
git checkout -b task/YYYYMMDD-brief-description   # replace with today's date and topic
git status   # must be clean before starting
```

Use today's date in YYYYMMDD format. Example: task/20260627-dpo-pipeline

---

## Step 2 — Spawn exactly 3 parallel sub-agents

Each agent works on a DIFFERENT section AND different files. No two agents may touch the same file.

Assign agents to 3 different sections from:
- poetry-llm/data/
- poetry-llm/preprocessing/
- poetry-llm/model/
- poetry-llm/generation/
- poetry-llm/evaluation/
- poetry-llm/research/

---

## Rules each sub-agent MUST follow

### Git rules

```bash
# Always start by confirming you are on the task branch:
git branch   # must show task/YYYYMMDD-... as current

# Commit only your own section's files — specific files only:
git add poetry-llm/<your-section>/<changed-files>

# Commit message format:
git commit -m "[section] what you did"

# NEVER run:
# git reset, git rebase, git push --force
# git checkout master, git merge, git stash pop
# git add -A, git add .   (too broad — may stage unrelated files)
```

Only stage the specific files you changed. Never use `git add .` or `git add -A`.

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

Link format — EXACTLY this pattern, filename only, no path prefix:
```markdown
| [filename.md](filename.md) | 한 줄 설명 |
```

Before adding a link to index.md:
1. Confirm the file actually exists at that path
2. Use ONLY the bare filename (e.g. `new_file.md`), never `./new_file.md` or `../section/new_file.md`
3. The link text must match the actual filename exactly, including `.md`

---

## What counts as valuable work

**Good:**
- Fixing wrong or outdated information
- Writing a design doc that is referenced but missing
- Adding concrete code, formulas, or examples to a vague section
- Filling in a `## 미결 사항` item from another file

**Bad (do not do):**
- Cosmetic rewording with no new information
- Adding obvious caveats or disclaimers
- Writing content that duplicates another file
- Touching files outside your assigned section
- Touching files another sub-agent is working on

---

## Step 3 — After all 3 agents finish, merge to master

```bash
git checkout master
git merge task/YYYYMMDD-brief-description
git branch -d task/YYYYMMDD-brief-description
```

If merge conflict: resolve by combining both sides' content, not discarding either.

Report:
- Which 3 sections were chosen and why
- What each agent did (file created or edited, key additions)
- Any new open questions that surfaced
- Confirm merge landed on master (`git log master --oneline -5`)
```
