# Poetry-LLM 프로젝트 — 에이전트 브리핑

## 프로젝트 한 줄 요약

20~40B 언어모델을 풀 파인튜닝하여 **기존에 없는 미학적 언어**를 가진 한국어 시를 생성하는 연구 프로젝트.
단순히 "그럴듯한 시"가 아닌, novelty(새로운 미학 아이디어·수사·소재 조합)를 목표로 한다.

## 지식 번들 구조

모든 프로젝트 지식은 `poetry-llm/` 디렉토리의 OKF 번들에 있다.
작업 전 반드시 `poetry-llm/index.md`를 먼저 읽어라.

```
poetry-llm/
├── index.md              ← 시작점. 전체 구조와 에이전트 역할 분담
├── overview/             ← 비전, 목표, 로드맵
├── data/                 ← 학습 데이터 전략
├── preprocessing/        ← 토큰화, 발음 레이어, 학습 데이터 포맷
├── model/                ← 모델 선정, 파인튜닝 전략
├── generation/           ← CoT 창작 노트, 반복 수정, novelty 필터
├── evaluation/           ← 평가 지표, 인간 평가 설계
└── research/             ← 관련 연구, 미답 질문
```

## 에이전트 브랜치 규칙

각 작업 세션은 **task 브랜치**에서 진행하고 완료 즉시 master로 머지한다.

```bash
# 작업 시작
git checkout master && git pull
git checkout -b task/YYYYMMDD-간략설명

# 작업 완료 후
git checkout master
git merge task/YYYYMMDD-간략설명
git branch -d task/YYYYMMDD-간략설명
```

- 브랜치 수명은 하나의 작업 세션 동안만. 완료 후 즉시 삭제.
- 서브에이전트 여러 개가 같은 task 브랜치를 쓸 경우, **각자 다른 파일**을 담당해야 한다.
- 장기 브랜치(`claude`, `antigravity` 등) 금지 — master 누적 충돌 방지.

작업 완료 후: `poetry-llm/log.md`에 변경 이력 추가

## 작업 지침

1. **읽기 우선**: 담당 섹션의 기존 파일을 모두 읽고 `## 미결 사항`을 파악한 뒤 작업
2. **OKF 형식 준수**: 모든 `.md` 파일은 `type`, `title`, `description`, `tags`, `timestamp` frontmatter 포함
3. **다른 섹션 참조 시**: 직접 수정 금지, 링크로 연결
4. **미결 사항 갱신**: 새로 발견한 미결 사항은 해당 파일 `## 미결 사항`에 추가
5. **근거 있는 주장만**: 추측은 `> 가설:` 블록으로 표시, 검증된 정보는 `# Citations`에 출처 기재
6. **편집 상태 마커**: 섹션 첫 줄 `> 탐색중` = 열린 목록(항목 추가 가능, 삭제 금지). `## 미결 사항` 항목 앞 `[Ph1]`/`[Ph2]` = 해당 Phase 후 결정, `[TODO]` = 지금 작업 가능. 마커 없는 섹션은 확정 → 변경 금지.

## 핵심 설계 결정 (변경 시 overview/ 섹션 업데이트 필요)

- 베이스 모델: **미결** — 현재 주요 검토 후보: Gemma 4 27B/31B. 파일럿은 10B급 예정. `poetry-llm/model/model_selection.md` 참조.
- 파인튜닝: **풀 파인튜닝** (LoRA는 파일럿 단계만)
- 특수 토큰: `<행갈이>`, `<연갈이>`, `<시작>`, `<끝>`
- 생성 파이프라인: **창작 노트(CoT) → 시 초안 → 반복 수정(최대 5회)**
- 목표: **Novelty** — n-gram 독창성 + 임베딩 거리 + 소재 novelty 복합 지표

## 현재 단계

**Phase 0** — 기반 설계 중. 아이디어 구체화 및 미결 사항 탐색이 주 작업.
`poetry-llm/overview/roadmap.md` 참조.
