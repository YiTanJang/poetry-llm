---
type: Playbook
title: Claude Code × Antigravity 협업 방식
description: 두 에이전트가 동일한 위키를 각자 브랜치에서 자유롭게 발전시키고 머지하는 워크플로우.
tags: [collaboration, claude-code, antigravity, git, workflow]
timestamp: 2026-06-27T00:00:00Z
---

# Claude Code × Antigravity 협업 방식

## 기본 구조

각 에이전트는 **전용 브랜치**에서 위키 전체를 자유롭게 읽고 수정한다.
영역 제한 없음. 문제를 찾으면 고치고, 아이디어가 생기면 추가한다.
머지할 때 충돌을 정리한다.

```
master          ← 안정 버전. 머지 결과만.
├── claude      ← Claude Code 전용 작업 브랜치
└── antigravity ← Antigravity 전용 작업 브랜치
```

## 각 에이전트의 작업 방식

### Claude Code
```bash
git checkout claude
# 위키 전체 읽고, 문제 찾고, 아이디어 추가, 자유롭게 수정
git add -A && git commit -m "..."
```

### Antigravity
```bash
git checkout antigravity
# 위키 전체 읽고, 문제 찾고, 아이디어 추가, 자유롭게 수정
git add -A && git commit -m "..."
```

## 머지

두 브랜치가 충분히 쌓이면 머지를 요청한다.
충돌은 에이전트에게 정리를 맡긴다:

> "claude 브랜치랑 antigravity 브랜치 master로 머지해줘. 충돌은 두 의견을 합치는 방향으로."

## 인스트럭션 파일

| 시스템 | 파일 | 역할 |
|--------|------|------|
| Claude Code | `CLAUDE.md` | 프로젝트 컨텍스트 자동 로드 |
| Antigravity | `AGENTS.md` | 프로젝트 컨텍스트 자동 로드 |
