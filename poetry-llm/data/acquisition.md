---
type: Playbook
title: 데이터 수집 파이프라인
description: 스캔/PDF 입수, OCR, 정제, 토크나이징까지의 실행 파이프라인.
tags: [data, acquisition, ocr, pipeline]
timestamp: 2026-06-27T00:00:00Z
---

# 데이터 수집 파이프라인

## 입수 소스

| 유형 | 소스 | 포맷 |
|------|------|------|
| 한국 현대시집 | 스캔본 / PDF | 이미지 PDF |
| 한국 시론/평론 | 스캔본 / PDF | 이미지 PDF |
| 외국어 시/시론 | 스캔본 / PDF / 전자책 | 이미지 PDF / EPUB |
| 공개 도메인 | Project Gutenberg, 위키문헌, Internet Archive | 텍스트 |
| 보조 예술 데이터 | 스캔본 / 공개 소스 | 혼합 |

## 스캔 파이프라인

```
1. PDF/이미지 입수

2. 스캔 품질 확인
   └── 300 DPI 이상 권장
   └── 행/연 구분이 명확하게 보이는지 확인
   └── 저해상도 → 업스케일링 (Real-ESRGAN 등) 시도

3. OCR
   └── 1차: Naver Clova OCR (한국어 정확도 최상)
   └── 대안: ABBYY FineReader, Tesseract + 한국어 학습 모델
   └── 외국어: 언어별 특화 OCR 또는 Tesseract

4. 후처리
   └── 행갈이 정규화 (OCR이 붙여쓴 시행 분리)
   └── 연갈이 복원 (사라진 빈 줄 복구)
   └── 이상 문자 제거 (OCR 오류 패턴)
   └── 메타데이터 삽입

5. 품질 검증
   └── 샘플 수동 검토 (100편당 5편)
   └── 원본 대조
   └── 오류율 측정 → 기준 초과 시 재처리
```

## OCR 주의 사항

### 한국어 시 특수성
- **행갈이**: OCR이 한 시행을 두 줄로 읽는 오류 빈발 → 후처리 필수
- **연갈이**: 빈 줄이 OCR 처리 과정에서 사라질 수 있음 → 원본 레이아웃 분석
- **들여쓰기**: 일부 시는 들여쓰기로 리듬을 표현 → 보존 필요
- **타이포그래피 실험** (황지우, 이상 등): 수동 처리 필수
- **한자 혼용**: 300 DPI 미만에서 오류 급증
- **세로쓰기**: 별도 OCR 엔진 설정 필요

### EPUB/전자책 처리
- DRM 해제 후 텍스트 추출
- 행갈이 마크업 보존 여부 확인 (EPUB 구조에 따라 다름)
- pandoc 또는 calibre로 텍스트 추출 후 정제

## 메타데이터 표준

모든 시에 다음 메타데이터 첨부:

```yaml
시인: 홍길동
시집: 시집 제목
출판연도: 1990
시제목: 시 제목
원고지면: p.45
언어: ko
장르: 현대시
소스: scan | epub | plaintext
```

## 정제 파이프라인 (코드 수준)

```python
# 행갈이/연갈이 복원 의사 코드
def restore_structure(ocr_text, layout_json):
    """
    OCR 결과와 레이아웃 분석 결과를 결합하여
    행/연 구조를 복원한다.
    """
    lines = split_by_layout(ocr_text, layout_json)
    stanzas = detect_stanza_breaks(layout_json)
    return insert_tokens(lines, stanzas,
                         line_token="<행갈이>",
                         stanza_token="<연갈이>")
```

## 우선순위 행동 목록

1. Naver Clova OCR API 셋업 및 파일럿 테스트 (100편)
2. 행갈이/연갈이 복원 스크립트 작성 및 검증
3. 메타데이터 파이프라인 구축
4. 배치 처리 자동화 (2,000권 스케일)
5. 품질 검증 샘플링 루틴 구축
