---
type: Section Index
title: 모델 설계
description: 베이스 모델 선정, 파인튜닝 전략, 아키텍처 고려사항.
tags: [model, fine-tuning, architecture]
timestamp: 2026-06-27T00:00:00Z
---

# 모델 설계

| 문서 | 내용 |
|------|------|
| [model_selection.md](model_selection.md) | 20~40B 후보 모델 비교 및 선정 기준 |
| [finetuning_strategy.md](finetuning_strategy.md) | 풀 파인튜닝 vs LoRA 결정 및 학습 전략 |
| [special_tokens.md](special_tokens.md) | 특수 토큰 추가 방법 및 임베딩 확장 |
| [continual_pretraining.md](continual_pretraining.md) | 시/문학 코퍼스 CPT → CoT SFT 2단계 학습 전략과 커리큘럼 학습과의 관계 |
| [training_infrastructure.md](training_infrastructure.md) | 분산 학습(FSDP vs DeepSpeed ZeRO-3) 설정, 비용 추정, 모니터링 지표 |
