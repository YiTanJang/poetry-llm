---
type: Playbook
title: Claude Code × Antigravity 분업 가이드
description: 두 에이전트 시스템의 강점 차이와 Poetry-LLM 프로젝트에서의 역할 분담 방법.
tags: [collaboration, claude-code, antigravity, multi-agent, workflow]
timestamp: 2026-06-27T00:00:00Z
---

# Claude Code × Antigravity 분업 가이드

## 두 시스템의 특성 비교

| 항목 | Claude Code (Anthropic) | Antigravity (Google) |
|------|------------------------|---------------------|
| 모델 | Claude Sonnet/Opus | Gemini 3.5 Flash |
| 강점 | 코드 작성, 파일 편집, 리서치, 멀티에이전트 스폰 | 지속 실행, 관리형 샌드박스, SDK 기반 파이프라인 |
| 인스트럭션 파일 | `CLAUDE.md` | `AGENTS.md` |
| Skills | `.claude/` 디렉토리 | `.agents/skills/` 디렉토리 |
| 장기 실행 | 세션 내 백그라운드 에이전트 | Managed Execution (API 기반) |
| 웹 검색 | WebSearch 툴 | Gemini 네이티브 검색 |

---

## 역할 분담 원칙

### 기본 원칙: 브랜치로 충돌 방지

두 시스템이 같은 파일을 동시에 건드리지 않도록 **브랜치를 기준으로 분리**한다.

```
main ← 둘 다 머지 가능. 직접 커밋 금지.
│
├── agent/data        ← 어느 쪽이든 담당 지정
├── agent/model       ← 어느 쪽이든 담당 지정
├── agent/generation  ← 어느 쪽이든 담당 지정
├── agent/evaluation  ← 어느 쪽이든 담당 지정
└── agent/research    ← 어느 쪽이든 담당 지정
```

동시에 같은 브랜치에서 작업하면 충돌. **한 시점에 한 브랜치는 한 시스템만.**

---

## 권장 분업 패턴

### 패턴 A — 영역 고정 분담 (가장 단순)

각 시스템이 고정된 영역을 전담:

| 브랜치 | 담당 |
|--------|------|
| `agent/data` | Antigravity |
| `agent/model` | Claude Code |
| `agent/generation` | Claude Code |
| `agent/evaluation` | Antigravity |
| `agent/research` | Claude Code |

**장점:** 충돌 없음, 단순  
**단점:** 특정 작업에 더 적합한 시스템이 있어도 고정됨

---

### 패턴 B — 작업 유형 기반 분담 (권장)

무엇을 하느냐에 따라 시스템을 선택:

**Claude Code가 더 적합한 작업:**
- 아이디어 발전 및 개념 문서 작성 (지금처럼)
- 멀티 에이전트를 스폰해서 병렬 리서치
- 코드 스니펫, pseudo-code 설계
- 웹 검색 → OKF 문서 보강
- 기존 파일의 문제점 캐치 및 수정

**Antigravity가 더 적합한 작업:**
- 실제 데이터 처리 파이프라인 코드 실행 (OCR, 정제 스크립트)
- 장기 실행 배치 작업 (2,000권 스캔 처리)
- Gemini API 연동 코드 (합성 데이터 생성 파이프라인)
- 모델 학습 스크립트 실행 및 모니터링
- 반복적인 데이터 변환 자동화

---

### 패턴 C — 순차 핸드오프 (세밀한 협업)

한 시스템이 설계하면, 다른 시스템이 구현:

```
1. Claude Code: 창작 노트 역추론 프롬프트 설계 → cot_training_data_design.md 작성
                                    ↓
2. Antigravity: 그 프롬프트로 실제 합성 데이터 2만 건 생성 → data/synthetic/ 저장
                                    ↓
3. Claude Code: 생성된 데이터 품질 샘플링 → 문제점 보고 → 프롬프트 개선
                                    ↓
4. Antigravity: 개선된 프롬프트로 재생성
```

**이 패턴의 핵심:** OKF 번들이 핸드오프 메모 역할을 한다.
Claude Code가 설계 문서를 쓰면 Antigravity가 `AGENTS.md`를 통해 그 설계를 읽고 구현.

---

## 실제 사용 방법

### Claude Code에서 Antigravity에게 작업 넘기기

1. Claude Code가 OKF 문서에 작업 지시를 기록
2. `git commit` 및 `git push`
3. Antigravity에서 해당 브랜치 pull
4. `AGENTS.md`가 자동 로드되어 컨텍스트 파악
5. 담당 디렉토리 파일의 `## 미결 사항` 확인 후 작업 시작

### Antigravity에서 Claude Code에게 작업 넘기기

반대 방향도 동일. OKF 문서의 `## 미결 사항`이 핸드오프 노트.

---

## 충돌 방지 체크리스트

작업 시작 전 확인:
- [ ] 현재 어느 브랜치에 있는가?
- [ ] 이 브랜치에 다른 시스템의 미커밋 변경이 없는가? (`git status`)
- [ ] 작업하려는 파일을 다른 시스템이 동시에 편집 중이 아닌가?

작업 완료 후:
- [ ] `log.md`에 변경 이력 추가
- [ ] `## 미결 사항` 업데이트 (다음 작업자를 위해)
- [ ] 커밋 후 `main`에 머지 요청 (또는 직접 머지)

---

## OKF 번들이 협업의 핵심

두 시스템은 모델도 다르고 환경도 다르지만,
**OKF 번들(마크다운 파일들)이 공통 언어** 역할을 한다.

- 어떤 시스템이 작업해도 같은 형식(frontmatter + 마크다운)
- `## 미결 사항`이 자연스러운 핸드오프 노트
- git이 버전 관리 및 충돌 해소를 담당
- `CLAUDE.md` / `AGENTS.md`가 각 시스템의 온보딩 담당
