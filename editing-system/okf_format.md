# OKF 편집 규칙

## Frontmatter

모든 `.md` 파일의 맨 위에 YAML frontmatter를 포함한다.

```yaml
---
type: Design
title: 파일 제목
description: 한 줄 요약 — 에이전트가 관련성을 판단하는 기준
tags: [태그1, 태그2]
timestamp: 2026-06-28T00:00:00Z
---
```

### `type` 값

| type | 용도 |
|------|------|
| `Bundle Index` | 번들 루트 또는 섹션 index.md |
| `Vision` | 프로젝트가 왜 존재하는가 |
| `Goals` | 구체적 목표와 비목표 |
| `Roadmap` | 단계별 진행 계획 |
| `Design` | 설계 결정 문서 |
| `Data Source` | 데이터 소스 기술 |
| `Playbook` | 절차·실행 가이드 |
| `Reference` | 관련 연구, 참고 자료 |
| `Open Questions` | 미답 질문 목록 |
| `Bundle Log` | 변경 이력 |

---

## 마커 시스템

### 열린 목록 — `> 탐색중`

섹션 첫 줄에 이 마커가 있으면 항목을 자유롭게 추가할 수 있다. **삭제 금지**.

마커가 없는 섹션 = **확정**. 변경 금지. 변경이 필요하면 `## 미결 사항`에 기록.

### 미결 사항 마커

각 파일 하단의 `## 미결 사항`에서 항목 앞에 붙인다.

| 마커 | 의미 |
|------|------|
| `[TODO]` | 지금 당장 작업 가능 |
| `[Ph1]` | Phase 1 이후 결정 |
| `[Ph2]` | Phase 2 이후 결정 |

### 가설 블록

```markdown
> 가설: 검증되지 않은 주장을 여기에 쓴다.
```

검증 완료 시 블록 제거, 일반 텍스트로 승격.

### Citations

```markdown
# Citations

- 저자 (연도) "논문 제목"
```

아직 읽지 않은 논문은 기재하지 않는다.

---

## 다른 섹션 참조

다른 섹션 파일을 참조할 때는 **링크만** 사용. 직접 수정 금지.

```markdown
→ `model/continual_pretraining.md` 참조
```

---

## 파일 명명

- 소문자 + 언더스코어: `model_selection.md`, `token_budget.md`
- 영어 파일명 (내용은 한국어 무방)

## 커밋 메시지

```
[섹션] 간결한 설명

예:
[data] 한국 현대시 수집 전략 구체화
[model] CPT 스케일링 법칙 인용 추가
[research] Q29-Q35 파이프라인 블로커 추가
```
