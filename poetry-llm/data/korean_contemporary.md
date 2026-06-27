---
type: Data Source
title: 한국 현대시 데이터
description: 1945년 이후 한국 현대시집 약 2,000권. 프로젝트의 핵심 학습 도메인.
tags: [data, korean, contemporary-poetry, scan]
timestamp: 2026-06-27T00:00:00Z
---

# 한국 현대시 데이터

## 개요

한국 현대시는 이 프로젝트의 **1차 목표 도메인**이다.
모델이 최종적으로 생성해야 할 시의 형식, 감각, 언어가 여기에 있다.

## 수집 목표

| 항목 | 목표량 | 비고 |
|------|--------|------|
| 현대시집 (스캔) | ~2,000권 | 1945년 이후 |
| 신춘문예 당선작 | 전량 | 주요 일간지 |
| 메이저 문예지 수록시 | 가능 범위 | 창작과비평, 문학동네, 현대문학 등 |

## 주요 스캔 대상 시집 (예시)

> 탐색중

### 한국 현대시 정전 (1945~1980)
- 김수영 전집 — 언어 실험, 산문시, 참여시
- 김춘수 — 무의미시, 꽃 연작
- 박재삼, 서정주, 박목월 — 서정시 전통
- 신동엽 — 민족서사시

### 1980년대~2000년대
- 최승자, 기형도 — 도시, 어둠, 해체적 언어
- 이성복 — 난해시, 이미지의 비약
- 황지우 — 형식 실험, 타이포그래피 시
- 박상순 — 포스트모던 실험

### 2000년대 이후
- 최정례, 김혜순 — 여성시, 신체 언어
- 이제니, 강성은 — 새로운 감각
- 황인찬, 최승자 — 동시대 감각

## 데이터 형식

```
<시집 메타>
제목: ...
시인: ...
출판연도: ...
출판사: ...

<시>
제목: ...
본문:
[원문 텍스트, 행갈이/연갈이 보존]
</시>
```

## OCR 품질 보증 파이프라인 (OCR QA Pipeline)

스캔된 현대 시집의 디지털 변환 과정에서 시의 호흡인 띄어쓰기, 행갈이, 특수 기호 오인식을 방지하기 위한 이중 검증 및 정량적 품질 보증 파이프라인을 운영한다.

```mermaid
graph TD
    A[원본 시집 스캔 이미지] --> B[Layout Parser: 레이아웃 검출]
    B --> C1[Engine A: Clova OCR]
    B --> C2[Engine B: Google Cloud Vision]
    C1 --> D[이중 엔진 교차 검증 및 정렬]
    C2 --> D
    D --> E{CER < 임계치 및 행 일치?}
    E -- Yes --> F[자동 정제 및 특수 토큰화]
    E -- No --> G[Human-in-the-Loop 검수]
    G --> F
    F --> H[최종 코퍼스 데이터베이스]
```

### 1. 이중 OCR 엔진 교차 검증 및 CER 계산
두 개의 독립적인 OCR 엔진의 출력 텍스트를 정렬(Sequence Alignment)한 뒤, 문자 수준의 오인식률(Character Error Rate, CER)을 계산한다. CER이 임계값(예: 2%)을 초과하거나 두 엔진 간 행(Line) 개수가 다를 경우, 수동 검수 대상(Flagged)으로 분류한다.

```python
# python pseudocode
from dataclasses import dataclass
from typing import List, Tuple

@dataclass
class BoundingBox:
    x_min: float
    y_min: float
    x_max: float
    y_max: float

@dataclass
class OCRToken:
    text: str
    bbox: BoundingBox
    confidence: float

@dataclass
class PoetryLine:
    tokens: List[OCRToken]
    line_number: int

class OCRQAPipeline:
    def __init__(self, cer_threshold: float = 0.02):
        self.cer_threshold = cer_threshold

    def calculate_cer(self, text_a: str, text_b: str) -> float:
        """Levenshtein Distance 기반 Character Error Rate 계산"""
        import editdistance
        return editdistance.eval(text_a, text_b) / max(len(text_a), len(text_b), 1)

    def verify_and_align(self, engine_a_out: List[PoetryLine], engine_b_out: List[PoetryLine]) -> Tuple[List[PoetryLine], bool]:
        """
        이중 OCR 엔진 출력을 교차 검증하여, 레이아웃/텍스트 불일치 발생 시 플래그를 설정합니다.
        """
        is_flagged = False
        aligned_poem = []
        
        # 행 개수 차이 발생 시 레이아웃 무결성 문제로 수동 검수 플래그
        if len(engine_a_out) != len(engine_b_out):
            is_flagged = True
            base_engine = engine_a_out if len(engine_a_out) >= len(engine_b_out) else engine_b_out
        else:
            base_engine = engine_a_out
            
        for i, line in enumerate(base_engine):
            if i < len(engine_b_out):
                text_a = "".join(t.text for t in line.tokens)
                text_b = "".join(t.text for t in engine_b_out[i].tokens)
                
                cer = self.calculate_cer(text_a, text_b)
                if cer > self.cer_threshold:
                    is_flagged = True
            aligned_poem.append(line)
            
        return aligned_poem, is_flagged
```

## 타이포그래피 및 실험적 배치 보존 전략

황지우의 해체주의 시나 이상의 띄어쓰기 파괴 등 시각적/공간적 레이아웃이 중요한 경우, 이를 2차원 좌표(Bounding Box)에 기반한 상대적 여백 토큰으로 구조화하여 보존한다.

### 1. 상대적 여백 및 들여쓰기 직렬화
각 단어와 문자 간의 물리적 거리(Gap)를 해당 행의 평균 문자 높이(Height)로 나누어 정규화된 여백 크기를 계산하고, 이를 `<space:N>` 및 `<tab>` 특수 토큰으로 치환하여 모델에 제공한다.

```python
# python pseudocode
class PoetryTypographyPreserver:
    def __init__(self, space_threshold: float = 0.5):
        self.space_threshold = space_threshold

    def serialize_with_layout(self, line: PoetryLine) -> str:
        if not line.tokens:
            return ""
            
        # x_min 기준 정렬
        sorted_tokens = sorted(line.tokens, key=lambda t: t.bbox.x_min)
        
        # 문자 평균 높이를 기준으로 폰트 스케일 계산
        avg_char_height = sum((t.bbox.y_max - t.bbox.y_min) for t in sorted_tokens) / len(sorted_tokens)
        
        serialized_line = []
        
        # 1. 들여쓰기(Indent) 보존
        first_token = sorted_tokens[0]
        if first_token.bbox.x_min > avg_char_height * 1.5:
            tabs = int(first_token.bbox.x_min / (avg_char_height * 1.5))
            serialized_line.append("<tab>" * tabs)
            
        # 2. 단어 간 띄어쓰기 및 광폭 여백 보존
        for i, current in enumerate(sorted_tokens):
            serialized_line.append(current.text)
            
            if i < len(sorted_tokens) - 1:
                next_token = sorted_tokens[i+1]
                gap = next_token.bbox.x_min - current.bbox.x_max
                
                # 여백이 평균 글자 높이 대비 특정 비율 이상인 경우 정밀 토큰화
                if gap > avg_char_height * self.space_threshold:
                    space_scale = round(gap / avg_char_height, 1)
                    serialized_line.append(f"<space:{space_scale}>")
                else:
                    serialized_line.append(" ")
                    
        return "".join(serialized_line)
```

### 2. 실험적 타이포그래피 마크업 스키마
시각적 형태(예: 취소선, 폰트 변화, 삽입 이미지 등)를 표현하기 위한 데이터 모델 스키마이다.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ExperimentalTypographySchema",
  "type": "object",
  "properties": {
    "poem_id": { "type": "string" },
    "has_experimental_layout": { "type": "boolean" },
    "elements": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["strikethrough", "font_style_change", "embedded_graphic", "rotated_text"] },
          "raw_text": { "type": "string" },
          "metadata": {
            "type": "object",
            "properties": {
              "rotation_degrees": { "type": "number" },
              "font_family": { "type": "string" },
              "graphic_description": { "type": "string" }
            }
          }
        },
        "required": ["type"]
      }
    }
  },
  "required": ["poem_id", "has_experimental_layout"]
}
```

# Citations

- 한국문학번역원 (KLTI) — 일부 디지털 자료 제공 가능성 검토
- 국립중앙도서관 디지털 아카이브
- RIDI, 교보문고 전자책 (저작권 협의 필요)

## 미결 사항

- [Ph1] 황지우의 신문 기사 스크랩이나 광고 전단 삽입과 같이 텍스트와 이미지가 결합된 멀티모달 시를 단일 텍스트 토큰화 환경에서 의미론적으로 어떻게 학습시킬 것인가?
- [Ph1] 이상 시의 띄어쓰기 파괴(오감도)와 같이 시인이 의도한 비표준 띄어쓰기를 OCR 엔진이 임의로 띄어쓰기 자동 보정(Spacing Correction) 처리하지 않도록 방지하는 정교한 룰셋의 임계점 정의 문제.
- [Ph1] 수직 쓰기(세로쓰기)로 인쇄된 해방 이전~1970년대 시집의 가로쓰기 자동 변환 시, 다단 레이아웃과 수직 읽기 순서가 교차할 때 발생하는 오정렬의 자동 복구 알고리즘 설계.
