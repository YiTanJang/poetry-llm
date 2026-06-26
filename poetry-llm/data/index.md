---
type: Section Index
title: 학습 데이터
description: 시, 시론, 시평론, 보조 예술 데이터의 수집 전략 및 구성.
tags: [data, corpus, collection]
timestamp: 2026-06-27
---

# 학습 데이터

| 문서 | 내용 |
|------|------|
| [korean_contemporary.md](korean_contemporary.md) | 한국 현대시 데이터 (~2,000권) |
| [korean_classical.md](korean_classical.md) | 저작권 만료 한국 시 |
| [poetry_criticism.md](poetry_criticism.md) | 시론 및 시평론 |
| [foreign_poetry.md](foreign_poetry.md) | 외국어 시 데이터 (~500권) |
| [other_arts.md](other_arts.md) | 보조 예술 데이터 (미술, 소설, 극) |
| [acquisition.md](acquisition.md) | 데이터 수집 방법 및 저작권 전략 |
| [quality_and_cleaning.md](quality_and_cleaning.md) | 데이터 정제 및 품질 필터링 가이드라인 |
| [token_budget.md](token_budget.md) | 토큰 예산 및 데이터 규모 계산 |
| [augmentation.md](augmentation.md) | 시 데이터 증강 전략 |

## 데이터 구성 목표

```
총 학습 데이터 구성 (추정)
─────────────────────────────────────────────
한국 현대시          ~2,000권 ── 핵심 도메인
시론/시평론          최대한    ── 메타 지식
저작권 만료 한국시   전량      ── 자유 사용
신춘문예/문예지      가능 범위 ── 현대 규범
외국어 시            ~500권   ── 교차 미학
외국어 시론/평론     최대한    ── 보편 시학
보조 예술 데이터     적정 비율 ── 감각 확장
─────────────────────────────────────────────
```

## 핵심 원칙

시론/시평론은 단순 해설이 아니다 — 시인이 **왜 이 언어를 선택했는지**를 설명하는
가장 밀도 높은 메타데이터다. 이 데이터가 모델이 시의 "논리"를 배우는 핵심 경로다.
