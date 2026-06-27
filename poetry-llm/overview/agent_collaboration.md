---
type: Playbook
title: Claude Code × Antigravity 협업 방식
description: 두 에이전트가 task 브랜치 단위로 작업하고 즉시 master로 머지하는 워크플로우.
tags: [collaboration, claude-code, antigravity, git, workflow]
timestamp: 2026-06-27T00:00:00Z
---

# Claude Code × Antigravity 협업 방식

## 기본 구조

장기 에이전트 브랜치 없음. 모든 작업은 **task 브랜치**에서 진행하고 완료 즉시 master로 머지한다.

```
master          ← 항상 최신. 모든 작업의 기점이자 종점.
└── task/날짜-설명  ← 하나의 작업 세션 동안만 존재. 완료 후 삭제.
```

## 왜 task 브랜치인가

장기 `claude` / `antigravity` 브랜치는 두 가지 문제를 만든다:

1. **누적 충돌**: 브랜치가 오래 살수록 master에서 멀어져 머지 시 충돌이 쌓인다.
2. **서브에이전트 간 충돌**: 한 브랜치 내에서 서브에이전트 여러 개가 같은 파일을 건드리면 충돌.

task 브랜치는 수명이 한 세션으로 짧아서 두 문제 모두 해소된다.

## 작업 흐름

### 세션 시작

```bash
git checkout master && git pull
git checkout -b task/YYYYMMDD-간략설명
```

### 작업 중

- 서브에이전트를 병렬로 돌릴 경우, 각자 **겹치지 않는 파일**을 담당
- 각 커밋은 담당 파일만 `git add poetry-llm/섹션/파일.md` 로 스테이징

### 세션 종료

```bash
git checkout master
git merge task/YYYYMMDD-간략설명
git branch -d task/YYYYMMDD-간략설명
```

충돌 발생 시: 양쪽 내용을 합치는 방향으로 수동 해결. 한쪽을 버리지 않는다.

## 인스트럭션 파일

| 시스템 | 파일 | 역할 |
|--------|------|------|
| Claude Code | `CLAUDE.md` | 프로젝트 컨텍스트 자동 로드 |
| Antigravity | `AGENTS.md` | 프로젝트 컨텍스트 자동 로드 |
| Antigravity | `ANTIGRAVITY_PROMPT.md` | 작업 시작 시 복사해서 붙여넣는 만능 프롬프트 |

## 미결 사항

- [TODO] AGENTS.md가 없을 경우 Antigravity가 CLAUDE.md를 읽는지 확인 필요
- [TODO] task 브랜치 네이밍 컨벤션을 더 구체화할 것인가 (섹션 접두사 등)
- [TODO] 두 에이전트가 동시에 작업할 경우 파일 분배를 누가 조율하는가
