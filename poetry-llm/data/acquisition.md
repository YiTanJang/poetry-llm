---
type: Data Source
title: 데이터 수집 파이프라인
description: 스캔/PDF 입수, OCR, 정제, 토크나이징까지의 실행 파이프라인.
tags: [data, acquisition, ocr, pipeline]
timestamp: 2026-06-28T00:00:00Z
---

# 데이터 수집 파이프라인

## 입수 소스

> 탐색중

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
- 텍스트 추출 후 행갈이 마크업 보존 여부 확인 (EPUB 구조에 따라 다름)
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

## 웹 스크래핑 아키텍처 (공개 디지털 아카이브 수집)

공개 도메인(Project Gutenberg, 위키문헌, Internet Archive 등)으로부터의 안전하고 효율적인 데이터 수집을 위한 분산형 스크래핑 아키텍처의 설계는 다음과 같다.

### 1. 시스템 아키텍처 구성

```
[수집 스케줄러 (Scheduler)]
       │ (수집 대상 URL/태스크 분배)
       ▼
[비동기 태스크 큐 (Task Queue: Redis / asyncio.Queue)]
       │
       ├─────────────────────────┬─────────────────────────┐
       ▼                         ▼                         ▼
 [크롤러 작업자 1]         [크롤러 작업자 2]         [크롤러 작업자 N]
 (User-Agent/Proxy 로테이션, robots.txt 준수, 지연 시간 제어)
       │                         │                         │
       └─────────────────────────┼─────────────────────────┘
                                 ▼
                    [중복 필터링 (Bloom Filter)]
                                 │ (신규 데이터 판별)
                                 ▼
                     [파서 엔진 (Parser Engine)]
                 (HTML -> 시 구조화 & 메타데이터 추출)
                                 │
                                 ▼
                   [검증 및 저장 (Validator & Storage)]
              (Raw HTML 백업 & OKF 표준 메타데이터 포맷 저장)
```

- **Scheduler & Task Queue**: 수집 주기 및 대상 URL 목록을 관리하며, 중복 수집을 방지하기 위한 체크포인트를 관리한다.
- **Fetcher Layer**: `robots.txt`를 엄격히 준수한다. 각 사이트별 Rate Limit(정중한 요청 대기 시간)을 동적으로 조정하고, 무작위 User-Agent 및 프록시 로테이션을 통해 서버 부하 분산 및 IP 블로킹을 우회한다.
- **Deduplication Engine**: Bloom Filter 또는 Redis Set을 활용하여 이미 수집되거나 파싱된 시(Poem)의 중복 처리를 실시간으로 필터링한다.
- **Parser Engine**: BeautifulSoup/lxml 및 정규표현식을 조합하여 사이트마다 다른 HTML 태그 구조에서 실제 시의 제목, 시인, 본문(행/연 구분), 창작 연도 등을 추출한다.
- **Storage Layer**: 감사(Audit) 및 재처리를 위해 원본 HTML을 압축 보관(Raw)하고, 파싱이 완료된 정제 데이터는 표준 OKF JSON/YAML 파일 형태로 보존한다.

### 2. 크롤러 아키텍처 코드 스켈레톤 (Python Asyncio & HTTPX)

```python
import asyncio
import logging
import random
from typing import Dict, Optional
import httpx
from bs4 import BeautifulSoup
from urllib.parse import urlparse

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("PoetryScraper")

class RobustPoetryScraper:
    def __init__(self, base_url: str, delay_range: tuple = (1.0, 3.0)):
        self.base_url = base_url
        self.delay_range = delay_range
        self.client = httpx.AsyncClient(
            headers={"User-Agent": "PoetryLLM-Bot/1.0 (Research Project; contact@example.com)"},
            timeout=10.0
        )
        self.visited_urls = set()

    async def fetch_page(self, url: str) -> Optional[str]:
        """robots.txt 및 Politeness 정책을 준수하며 페이지를 가져옴"""
        if url in self.visited_urls:
            return None
            
        # Politeness Delay 적용 (서버 부하 방지)
        delay = random.uniform(*self.delay_range)
        await asyncio.sleep(delay)
        
        try:
            response = await self.client.get(url)
            if response.status_code == 200:
                self.visited_urls.add(url)
                return response.text
            else:
                logger.warning(f"Failed to fetch {url}: Status {response.status_code}")
        except Exception as e:
            logger.error(f"Error fetching {url}: {e}")
        return None

    def parse_poetry(self, html_content: str, source_url: str) -> Optional[Dict]:
        """HTML을 분석하여 시의 구조와 메타데이터를 추출"""
        soup = BeautifulSoup(html_content, 'lxml')
        
        try:
            # 아카이브 형식에 따른 파싱 규칙 적용 (예시: 위키문헌 구조)
            title = soup.find("h1", id="firstHeading").text.strip()
            poet = soup.find("a", title=True).text.strip() # 실제 구현에서는 정밀 셀렉터 사용
            
            # 시 본문 및 행/연 복원
            body_div = soup.find("div", class_="mw-parser-output")
            paragraphs = body_div.find_all("p")
            
            lines = []
            for p in paragraphs:
                for line in p.text.split("\n"):
                    cleaned_line = line.strip()
                    if cleaned_line:
                        lines.append(cleaned_line)
                lines.append("<연갈이>") # 단락 끝에 연갈이 토큰 삽입
            
            # 마지막 연갈이 제거 후 표준 포맷 구성
            if lines and lines[-1] == "<연갈이>":
                lines.pop()
                
            return {
                "시인": poet,
                "시제목": title,
                "본문": " <행갈이> ".join(lines),
                "출판연도": "공개 도메인",
                "소스": "web-scraping",
                "원본URL": source_url
            }
        except Exception as e:
            logger.error(f"Error parsing content from {source_url}: {e}")
            return None

    async def run_pipeline(self, target_urls: list):
        for url in target_urls:
            logger.info(f"Processing URL: {url}")
            html = await self.fetch_page(url)
            if html:
                poetry_data = self.parse_poetry(html, url)
                if poetry_data:
                    logger.info(f"Successfully scraped: {poetry_data['시제목']} by {poetry_data['시인']}")
                    # TODO: 저장소 파이프라인으로 데이터 전달
```

---

## 저작권 공정이용 (Fair Use) 법적 논리 및 가이드라인

비상업적 연구 목적으로 저작권이 있는 시집 및 현대시 데이터를 모델 학습에 활용할 때 적용 가능한 법적 근거와 데이터 보호 가이드라인은 다음과 같다.

### 1. 대한민국 저작권법상 법적 근거

- **저작권법 제35조의5 (저작물의 공정한 이용)**:
  - 대한민국 저작권법은 보도·비평·교육·연구 등을 위하여 저작물의 통상적인 이용방법과 충돌하지 아니하고 저작권자의 정당한 이익을 부당하게 해치지 아니하는 범위 내에서 저작물을 이용할 수 있도록 공정이용 규정을 두고 있다.
  - **연구 목적성**: 본 프로젝트는 기존에 없는 새로운 미학적 수사와 구조를 탐색하는 **비상업적 학술 연구**로, 공정이용의 목적 요건을 충족한다.
  - **텍스트 및 데이터 마이닝 (TDM) 면책 동향**: 컴퓨터를 통한 대량의 데이터 분석 및 학습 과정에서 발생하는 일시적 복제 및 가공은 저작권 침해로 보기 어렵다는 학계 및 법조계의 다수설을 따르며, 국회에 계류 중인 TDM 면책 조항 법제화 방향과 궤를 같이한다.

### 2. 미국 저작권법 제107조 (Fair Use) 4요소 분석

미국의 저작권법 제107조에 명시된 공정이용 판단 기준 4가지를 기준으로 본 연구의 데이터 수집 정당성을 분석한다.

1. **이용의 목적과 성격 (Purpose and Character of the Use)**:
   - **변형적 이용 (Transformative Use)**: 모델은 원본 시의 감상적/심미적 가치를 그대로 복제하여 대중에게 서비스하지 않는다. 대신, 언어 구조, 문체 패턴, 감정 임베딩 등 추상적 '패턴'을 기계 학습하기 위해 이용한다. 이는 저작물의 본래 목적과 완전히 다른 차원의 변형적 이용이다 (*Authors Guild v. Google, Inc.* 판례 참조).
2. **저작물의 성격 (Nature of Copyrighted Work)**:
   - 시 저작물은 창작성의 정도가 매우 높은 예술 저작물로 보호 범위가 넓으나, 모델 학습은 저작물의 표현 그 자체를 배포하기 위함이 아니라 사실상의 수학적/통계적 자질(feature)을 추출하기 위한 도구적 사용에 한정된다.
3. **사용된 부분의 양과 질적 중요성 (Amount and Substantiality of the Portion Used)**:
   - 언어모델 학습의 특성상 전체 문맥 정보가 필요하므로 저작물 전량을 학습 데이터로 수집하는 것이 불가피하다. 그러나 최종 학습된 모델 파라미터 내에 원본 텍스트가 직접 복사되지 않으므로 질적으로 침해하지 않는다.
4. **저작물의 시장 가치 및 잠재적 시장에 미치는 영향 (Effect on Market)**:
   - 본 프로젝트의 데이터는 철저히 연구망 내부에서 격리되어 관리되며 외부로 시 텍스트 자체가 배포되지 않는다. 따라서 원본 시집의 판매 시장이나 출판사의 잠재적 상업 시장을 침해하거나 대체하지 않는다.

### 3. 데이터 거버넌스 및 유출 방지 가이드라인 (Data Minimization)

법적 리스크를 최소화하고 공정이용의 요건을 유지하기 위해 다음 가이드라인을 준수한다.

- **망 격리 및 접근 제어**: 수집된 원본 현대시 데이터셋은 외부 접근이 통제된 보안 스토리지에 보관하며, 에이전트 및 연구진 외의 제3자 유출을 원천 차단한다.
- **암기 생성(Memorization) 방지**: 추론(Generation) 단계에서 모델이 특정 원본 시를 그대로 출력하는 현상을 차단하기 위해, 디코딩 파라미터 조정 및 생성물 대상 n-gram 중복도 검사 필터(Novelty Filter)를 적용한다.
- **가중치 배포 제한**: 파인튜닝된 최종 모델은 상업적으로 판매하지 않으며, 연구 목적으로만 공개하되 필요 시 가중치 비공개(Private API 형태)로 운영하여 원본 데이터 복원 리스크를 최소화한다.

---

## 상업 출판사 협력 모델 (Publisher Collaboration Models)

장기적으로 우수한 현대시 데이터를 합법적이고 안정적으로 수집하기 위해, 상업 시 전문 출판사(창비, 문학과지성사, 민음사 등)와 상생할 수 있는 제휴 및 라이선스 모델을 제시한다.

### 1. 보안 트레이닝 샌드박스 (Secure Training Sandbox)

```
[출판사 소유 보안 스토리지] ──── (데이터 제공) ────► [보안 클라우드 학습 환경 (Sandbox)]
                                                              ▲
                                                    [학습 파이프라인 구동]
                                                              │ (데이터 반출 불가)
                                                              ▼
                                                   [체크포인트 파라미터만 추출]
```

- **개요**: 출판사가 민감한 시 텍스트 원본을 연구진에게 직접 제공하는 대신, 보안이 인증된 클라우드 샌드박스(Secure Sandbox VM/Container) 환경 내에 데이터를 업로드한다.
- **통제**: 연구진은 이 보안 환경 내부에서만 모델 파인튜닝 코드를 실행하며, 원본 파일의 복제나 네트워크 외부 전송은 완전히 차단된다. 학습이 완료된 모델 가중치(Weights)만 검증 과정을 거쳐 반출함으로써 데이터 유출 위험을 0%로 통제한다.

### 2. 가치 공유 및 토큰 기반 라이선스 모델 (Token-Based / Value-Sharing Model)

- **토큰 비례 기여도 정산**: 학습에 기여한 출판사별 시집 데이터를 토큰(Token) 수 기준으로 계량화하여 라이선스 풀(Pool)을 구성한다.
- **수익 쉐어 구조**: 향후 본 연구 모델을 기반으로 시인용 창작 지원 도구 상용화 서비스나 AI 시집 출판 등의 부가 수익이 발생할 경우, 사전에 합의된 기여율(학습 데이터 점유 토큰 비율)에 따라 로열티를 배분하는 수익 공유(Revenue Sharing) 계약을 체결한다.

### 3. 작가 지원 도구 공동 개발 (AI Co-Writer Tool Utility)

- **독점적 크리에이티브 툴 제공**: 협력 출판사 소속 시인들이 창작 과정에서 새로운 영감을 얻을 수 있도록, 해당 출판사의 문학적 자산을 보존하면서 협업할 수 있는 전용 AI 창작 어시스턴트(초안 브레인스토밍, 단어 연상 맵 제공 등)를 공동 개발하여 제공한다.
- **피드백 루프**: 현업 작가들의 실제 사용 피드백을 수집하여 시 생성 모델의 미학적 정밀도와 Novelty 지표를 정교화하는 정성적 공동 연구 파트너십을 구축한다.

### 4. 선점형 비상업 라이선스 및 상용화 콜옵션 계약

- **단계별 라이선싱**: 초기 단계에서는 비상업적 연구(Research-Only) 목적으로 무상 혹은 최소 비용의 라이선스 계약을 체결한다.
- **콜옵션 조항**: 연구 모델이 상용 수준의 성능을 발휘하여 실제 서비스화될 경우, 협력 출판사가 사전에 지정된 고정 요율이나 로열티 조건으로 우선 상용 라이선스를 체결할 수 있는 콜옵션(Call Option) 조항을 삽입하여 출판사의 미래 권리를 보장한다.

---

## 활동 작가(현역 시인) 협력 및 라이선싱 모델 (Living Poets Collaboration & Licensing)

본 프로젝트는 저작권법상의 공정이용을 넘어, 생존해 있는 현역 시인(Living Poets)의 지적재산권을 존중하고 창작 생태계와 상생하기 위한 자발적 협력 및 보상 모델을 구축한다.

### 1. 활동 작가 참여 및 투고 가이드라인 (Submission Guidelines)
- **자발적 투고 및 동의**: 현역 시인은 본인의 시집 또는 미발표작을 프로젝트 참여 신청서와 함께 제출할 수 있다.
- **제출 포맷 및 검증**: 디지털 텍스트(Markdown, HWP, DOCX 등) 또는 출판된 시집의 PDF/스캔본 형태로 제출할 수 있으며, 연구단은 시인의 신원 및 저작권 보유 여부를 확인하는 검증 단계를 거친다.
- **선택적 동의 (Granular Consent)**: 작가는 제출 시 자신의 데이터가 활용될 단계를 선택할 수 있다.
  - 사전학습(Pre-training) 혹은 풀 파인튜닝(SFT) 데이터셋 포함 여부
  - 미발표작의 보안 임베딩/토큰화 활용 여부
  - 생성 모델의 스타일 참조(Few-shot 또는 기여도 산정) 허용 여부

### 2. 저작권 보호 및 보안 샌드박스 (Secure Sandbox for Living Poets)
작가들의 미발표작이나 민감한 저작물이 모델 외부로 유출되거나 원문 그대로 복원되는 것을 방지하기 위해 특수 설계된 보안 샌드박스 환경을 구축한다.
- **안전한 토큰화 및 임베딩화 (Raw Text-Free Storage)**:
  - 시인이 제출한 원문 텍스트는 보안 샌드박스 내에 진입하는 즉시 로컬에서 토큰 ID 또는 고차원 벡터 임베딩(Vector Embeddings)으로 변환된다.
  - 변환이 완료된 원문(Raw Text)은 디스크에서 즉시 완전 삭제(Shredding)되며, 학습 서버에는 오직 수치화된 토큰 시퀀스와 임베딩 가중치만 저장된다.
  - 이를 통해 외부 침입이나 모델 가중치 역추적을 통한 원문 복구가 수학적으로 불가능하도록 제어한다.
- **추론(Generation) 시 원문 유출 차단**:
  - 해당 작가의 데이터를 활용하여 미학적 피드백을 주거나 개별 학습을 진행하는 경우, 디코더 레벨에서 원문과 동일한 n-gram 패턴의 생성을 강제 차단하는 동적 차단(Dynamic Generation Block) 메커니즘을 샌드박스 내부 추론 API에 기본 적용한다.

### 3. 상생형 로열티 및 라이선스료 산정 체계 (Collaborative Royalty Schemes)
현역 시인의 기여를 공정하게 평가하기 위해 다음과 같은 다차원 로열티 산정 메커니즘을 제안한다. 정량적 매개변수를 기반으로 한 기여도 배분 모델을 적용하며, 구체적인 상수 및 비율은 미결 사항으로 남겨둔다.

- **데이터 토큰 볼륨 기준 (Token-based Contribution)**:
  - 학습 데이터셋에 최종 포함된 해당 작가 시의 총 토큰 수($T_{poet}$)가 전체 학습 데이터셋 내에서 차지하는 비율에 비례하여 기본 라이선스료를 산정한다.
  - 식: $W_{base} = f(T_{poet} / T_{total})$ (단, $T_{total}$은 전체 데이터셋의 총 토큰 수)
- **SFT(미세조정) 데이터셋 참여 가중치 (SFT Contribution Weight)**:
  - 단순 사전학습이 아닌, 고품질 CoT(창작 노트) 작성을 동반한 SFT 데이터셋이나 RLHF 피드백 데이터에 직접 참여하여 제공한 데이터에 대해서는 높은 질적 가중치($\alpha_{sft}$)를 부여한다.
- **생성물 사용량 비례 로열티 (Generation Usage-based Royalty)**:
  - 향후 시인의 문체를 존중하여 학습된 커스텀 어시스턴트나 창작 지원 도구에서 작가의 참조 벡터/스타일 임베딩이 활성화되거나 선택된 빈도($U_{gen}$)를 측정하여 가변 로열티($R_{usage}$)를 추가 정산한다.

---

## 우선순위 행동 목록

1. Naver Clova OCR API 셋업 및 파일럿 테스트 (100편)
2. 행갈이/연갈이 복원 스크립트 작성 및 검증
3. 메타데이터 파이프라인 구축
4. 배치 처리 자동화 (2,000권 스케일)
5. 품질 검증 샘플링 루틴 구축
6. 공동 아카이브 웹 크롤링 스크립트 및 중복 방지 모듈 개발
7. 주요 시 전문 출판사 대상 보안 샌드박스 기술 검증서(PoC) 및 NDA 협의안 작성
8. 암기 생성(Memorization) 방지 및 novelty 필터링 강도 기준 수립

## 미결 사항

- [Ph1] 동일 시의 판본 차이(예: 개정판 시집에서 일부 자구만 수정된 경우)를 중복으로 판정하여 정제할 것인가, 아니면 변형적 버전으로 모두 학습에 활용할 것인가?
- [Ph2] 생성 결과물에서 원본 시의 연속된 n-gram이 몇 단어/자 이상 일치할 때 이를 암기 생성(Memorization)으로 판단하고 novelty 필터링을 작동시킬 것인지에 대한 정량적 임계치는 어떻게 정할 것인가?
- [TODO] 폐쇄형 클라우드 샌드박스(Secure Sandbox) 학습 환경을 운용할 때 발생하는 클라우드 인프라 비용(GPU 인스턴스, 대용량 보안 스토리지 등)을 연구단과 출판사 간에 어떻게 분담할 것인가?
- [Ph1] 활동 작가가 투고한 미발표작의 보안 임베딩화 과정에서 사용될 토크나이저와 임베딩 모델의 버전이 업데이트될 경우, 기존에 변환되어 보관 중인 벡터 데이터를 무결하게 재변환하고 정합성을 유지하기 위한 샌드박스 내 동적 업데이트 파이프라인은 어떻게 설계해야 하는가?
- [TODO] 현역 작가용 로열티 정산 모델에서 '생성물 사용량 비례 로열티' 산정 시, 생성된 시가 특정 시인의 스타일을 몇 %나 닮았는지를 판별하는 스타일 유사도 평가 지표(Style Similarity Metric)를 어떻게 편향 없이 정량화할 것인가?
- [TODO] [User Review] 저작권 조항 내부 불일치 해소: 이 파일은 저작권법 제35조의5(공정이용)를 근거로 삼고 있으나 `quality_and_cleaning.md`는 제35조의3(교육 목적 예외)을 인용한다. 두 조항은 적용 요건이 다르므로 법률 자문을 통해 단일 조항으로 통일하거나 각 조항의 적용 범위를 명시적으로 분리해야 한다. TDM 면책(section 1에서 언급)은 국회 계류 중 법안이므로 확정 전까지 현행법(제35조의5)만을 공식 근거로 운용해야 한다.
  - 법적 검토가 필요한 서브 질문:
    1. 본 프로젝트의 향후 결과물(모델 또는 생성된 시)이 오픈소스 공개를 넘어 상업적 서비스(API 등)로 활용될 가능성이 있는가? (상업적 성격 추가 시 제35조의5 적용이 매우 엄격해짐)
    2. 국회의 TDM(텍스트 및 데이터 마이닝) 면책 조항 입법이 무산되거나 지연될 경우, 현행 제35조의5만으로 전체 모델 파인튜닝 파이프라인의 법적 리스크를 방어할 수 있는가?
    3. `quality_and_cleaning.md`에 명시된 '교육 목적(제35조의3)' 주장을 폐기하고 프로젝트 전체의 법적 근거를 제35조의5(공정이용)로 일원화할 것인가? 아니면 데이터 처리 단계별로 분리 적용할 것인가?

