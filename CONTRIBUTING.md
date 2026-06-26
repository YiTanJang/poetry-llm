# 기여 가이드 — Poetry-LLM OKF 번들

이 저장소는 여러 에이전트(및 인간)가 병렬로 기여하는 OKF 지식 번들입니다.

## 브랜치 구조

```
main                    ← 안정적 버전. 직접 커밋 금지.
│
├── agent/data          ← 데이터 에이전트 전담: poetry-llm/data/
├── agent/model         ← 모델 에이전트 전담: poetry-llm/model/, poetry-llm/preprocessing/
├── agent/generation    ← 생성 에이전트 전담: poetry-llm/generation/
├── agent/evaluation    ← 평가 에이전트 전담: poetry-llm/evaluation/
└── agent/research      ← 리서치 에이전트 전담: poetry-llm/research/
```

## 에이전트별 작업 규칙

### 1. 브랜치 선택
각 에이전트는 담당 브랜치에서만 작업합니다.
```bash
git checkout agent/data   # 데이터 에이전트
git checkout agent/model  # 모델 에이전트
# 등
```

### 2. 파일 작업 규칙
- 자신의 담당 디렉토리 파일만 수정
- `index.md`, `log.md`는 각 에이전트가 자유롭게 수정 가능
- 다른 섹션의 파일에 링크를 추가하는 것은 허용
- 다른 에이전트 담당 파일 직접 수정 금지 (제안은 Issue 또는 주석으로)

### 3. OKF 문서 형식

모든 `.md` 파일은 다음 frontmatter를 포함해야 합니다:

```yaml
---
type: <문서 유형>      # REQUIRED
title: <제목>
description: <한 줄 요약>
tags: [태그1, 태그2]
timestamp: <ISO 8601>  # 최종 수정 시각
---
```

**type 값 가이드**:
| type | 용도 |
|------|------|
| `Section Index` | 섹션 인덱스 파일 |
| `Vision` | 비전/방향 문서 |
| `Goals` | 목표 정의 |
| `Roadmap` | 로드맵 |
| `Data Source` | 데이터 소스 기술 |
| `Design` | 설계 결정 문서 |
| `Playbook` | 실행 가이드 |
| `Reference` | 참고 자료 |
| `Bundle Log` | 변경 이력 |
| `Bundle Index` | 번들 루트 인덱스 |

### 4. 커밋 메시지 형식

```
[영역] 간결한 설명

예:
[data] 한국 현대시 수집 전략 구체화
[model] Qwen2.5-32B 파인튜닝 요구사항 추가
[research] 관련 논문 3편 추가
```

### 5. main 머지 절차

1. 브랜치 작업 완료
2. `log.md`에 변경 이력 추가
3. Pull Request 생성 → 사람 또는 오케스트레이터 에이전트가 검토
4. 승인 후 main으로 머지

## 파일 명명 규칙

- 소문자, 언더스코어 구분
- 영어 파일명 (내용은 한국어 가능)
- 예: `model_selection.md`, `training_data_formats.md`

## 미결 사항 표시

문서 내 미결 사항은 `## 미결 사항` 섹션에 목록으로 작성:
```markdown
## 미결 사항
- 학습률 최적값 미결정
- 외국어 데이터 혼합 비율 탐색 필요
```

이를 통해 다른 에이전트가 미결 사항을 발견하고 기여할 수 있습니다.
