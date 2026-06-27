---
type: Playbook
title: Claude Code × Antigravity 협업 방식
description: 두 에이전트가 task 브랜치 단위로 작업하고 즉시 master로 머지하는 워크플로우와 역할 분담 및 협업 패턴.
tags: [collaboration, claude-code, antigravity, git, workflow]
timestamp: 2026-06-28T00:00:00Z
---

# Claude Code × Antigravity 협업 방식

## 기본 구조

장기 에이전트 브랜치 없음. 모든 작업은 **task 브랜치**에서 진행하고 완료 즉시 master로 머지한다.

```
master          ← 항상 최신. 모든 작업의 기점이자 종점.
└── task/날짜-설명  ← 하나의 작업 세션 동안만 존재. 완료 후 삭제.
```

## 역할 분담 (Agent Roles)

| 에이전트 | 역할 정의 | 주요 책임 |
| :--- | :--- | :--- |
| **Claude Code** | 로컬 구현 및 테스트 실행 엔진<br>(Primary Local Implementation & Test Execution Engine) | - 구체적인 Python 소스 코드 및 파이프라인 구현<br>- 로컬 환경에서의 테스트 스크립트 작성 및 실행<br>- 디버깅 및 하위 레벨 최적화 수행 |
| **Antigravity** | 글로벌 아키텍처 설계자, 멀티태스크 감독자, 검증자<br>(Global Architectural Designer, Multi-Task Supervisor, & Verifier) | - 고수준 프로젝트 설계, 위키 아키텍처 및 연구 로드맵 수립<br>- 서브에이전트 작업 분배 및 멀티태스크 진행 현황 조율<br>- 구현 결과물이 프로젝트 설계 원칙과 성능 지표를 준수하는지 검증 |

## 협업 및 충돌 방지 패턴 (Coordination Patterns)

두 에이전트가 동일한 작업 공간에서 충돌 없이 협업하기 위해 다음 규칙을 준수한다.

### 1. 디렉토리 소유권 (Directory Ownership)
- 각 작업 세션 및 서브에이전트 호출 시 담당 디렉토리 소유권을 엄격히 분리한다.
- 예를 들어, 데이터 가공 스크립트 구현 작업 중에는 Claude Code가 `poetry-llm/data/` 내부의 특정 구현 파일을 독점적으로 소유하고, Antigravity는 이에 대응하는 위키 설계 문서(`poetry-llm/data/*.md` 등) 수준에서 작업하여 소스 코드 수준의 충돌을 방지한다.

### 2. task 브랜치 내 파일 락 시그널링 (File Lock Signaling)
- 공통 참조 파일(예: `index.md`, `log.md`, `roadmap.md` 등)은 여러 에이전트가 동시에 수정할 가능성이 높다.
- 이를 방지하기 위해 **파일 락 시그널링** 메커니즘을 사용한다.
  - **작업 선점 표시**: 에이전트가 공유 파일을 수정하기 전, 해당 파일의 작업 착수 상태를 커밋 또는 태스크 로그(예: `poetry-llm/log.md` 또는 전용 시그널링 파일)에 기록하여 락을 획득한다.
  - **락 해제**: 작업 완료 후 수정한 파일들을 개별 스테이징 및 커밋(`git commit -m "[section] commit message"`)하여 master 브랜치로 머지함으로써 락을 즉시 해제한다.

---

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
- [TODO] 파일 락 시그널링 시 시그널링 메커니즘의 구현 방식 구체화: 별도 파일(예: `.lock`)을 사용할 것인지, 아니면 git commit 메시지 및 log.md의 상태 기록(예: `> 작업중: 파일경로`)으로 대체할 것인지 결정 필요.
- [TODO] 두 에이전트가 동시에 작업할 경우 파일 분배 및 우선순위 결정을 자동화할 수 있는 마스터 에이전트 조율기(Orchestrator)의 도입 가능성 검토.
