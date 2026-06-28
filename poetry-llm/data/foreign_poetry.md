---
type: Data Source
title: 외국어 시 데이터
description: 영어, 일본어, 중국어, 독일어 등 외국어 시 약 500권. 교차 미학 학습 및 보편 시학 습득용.
tags: [data, foreign, multilingual, open-data]
timestamp: 2026-06-28T00:00:00Z
---

# 외국어 시 데이터

## 역할

외국어 시는 **한국어 시의 직접 모방 대상이 아니다.**
목적은 두 가지:

1. **보편 시학의 습득**: 언어를 초월하는 시의 원리 — 이미지, 리듬, 긴장, 반전
2. **미학적 레퍼토리 확장**: 한국어 시에 없는 기법, 소재, 구조의 감각 흡수

## 오픈 데이터 주요 소스

> 탐색중

### Project Gutenberg
- 영어 고전시 전량 (Whitman, Dickinson, Blake, Keats, Shelley 등)
- 독일어 고전 (Goethe, Rilke, Hölderlin)
- 프랑스어 시 (Baudelaire, Rimbaud, Verlaine, Mallarmé)

### Poetry Foundation / Academy of American Poets
- 현대 영어시 일부 공개

### Poets.org
- 현대 미국 시 선집

### 기타
- Wikisource 다국어 시 섹션
- Internet Archive — 스캔된 시집 다수 공개

## 스캔 목표 (~500권)

| 언어 | 목표 | 우선순위 이유 |
|------|------|-------------|
| 영어 | ~200권 | 현대시 실험의 중심, 자료 풍부 |
| 일본어 | ~150권 | 한국 현대시에 직간접 영향, 한자문화권 |
| 중국어 | ~80권 | 한자 감각, 간결한 이미지 언어 |
| 독일어 | ~40권 | 릴케, 트라클, 브레히트 — 서정 전통 |
| 기타 | ~30권 | 프랑스어, 스페인어, 아랍어 등 |

## 언어별 핵심 작가/작품

> 탐색중

### 영어 (현대)
- William Carlos Williams — 이미지즘, 구어적 직접성
- Wallace Stevens — 철학적 이미지, 관념의 시화
- Sylvia Plath — 고백시, 강렬한 심리적 이미지
- John Ashbery — 난해성, 흐름 의식
- Frank O'Hara — 일상성, 도시적 감각
- Anne Carson — 산문시, 고전과 현대의 융합

### 일본어
- 松尾芭蕉 — 하이쿠, 여백의 시학
- 西脇順三郎 — 일본 초현실주의
- 谷川俊太郎 — 현대적 단순성

### 중국어
- 北島, 顾城 — 朦胧诗(몽롱시), 언어의 비약
- 余秀华 — 농촌, 여성, 직접적 언어

### 독일어
- Rainer Maria Rilke — 사물시(Dinggedicht), 내면성
- Georg Trakl — 표현주의, 몰락의 이미지
- Paul Celan — 홀로코스트 이후 언어, 극한의 압축

## 번역 및 정렬 포맷 규칙 (Translation Formatting Rules)

외국어 원문과 한국어 번역문을 병렬 학습(Parallel Learning)하여 언어 간 시적 감각과 번역적 판단을 모델에 인코딩하기 위해 아래의 포맷 규칙을 적용한다.

### 1. 병렬 블록 구조 (Parallel Block Format)
시의 시각적 형태(행/연 구분)를 온전히 보존하면서 원문과 번역의 대응 관계를 명시하기 위해 다음과 같은 구조화된 포맷을 채택한다.

* **연 단위 병렬(Stanza-by-Stanza Parallel) JSON 구조 (주요 가공 포맷):**
  원문의 문맥과 번역문의 매핑을 명시하기 위해 연 단위로 정렬된 JSON 객체를 사용한다.
  ```json
  {
    "meta": {
      "author": "Rainer Maria Rilke",
      "title": "Der Panther",
      "language": "de",
      "translator": "황현산"
    },
    "parallel_stanzas": [
      {
        "source": "Sein Blick ist vom Vorübergehn der Stäbe<행갈이>so müd geworden, dass er nichts mehr hält.<행갈이>Ihm ist, als ob es tausend Stäbe gäbe<행갈이>und hinter tausend Stäben keine Welt.",
        "target": "그의 눈길은 창살들이 지나가는 것을 보다가<행갈이>너무 피로해져, 아무것도 붙잡지 못한다.<행갈이>그에게는 마치 수천 개의 창살이 있고<행갈이>수천 개의 창살 뒤에는 아무 세상도 없는 듯하다."
      }
    ]
  }
  ```

* **행 단위 정렬 및 인터리빙(Interleaving) 구조:**
  원문과 번역을 행 단위로 번갈아 배치하여 직접적인 시행 대응을 유도하는 시퀀스 형태이다. 특수 태그 `<source_start>`, `<source_end>`, `<target_start>`, `<target_end>`를 정의하여 결합한다.
  ```
  <시작>
  <source_start>Sein Blick ist vom Vorübergehn der Stäbe<source_end>
  <target_start>그의 눈길은 창살들이 지나가는 것을 보다가<target_end><행갈이>
  <source_start>so müd geworden, dass er nichts mehr hält.<source_end>
  <target_start>너무 피로해져, 아무것도 붙잡지 못한다.<target_end><행갈이>
  ...
  <연갈이>
  <끝>
  ```

### 2. SFT 학습 데이터 구조화 포맷 (SFT Training Data Formats)
지도 미세 조정(SFT) 단계에서는 번역문과 원문 간의 정합 수준 및 스타일 학습 목적에 따라 두 가지 형태의 prompt-response 템플릿을 다르게 사용한다.

* **블록 분리형 포맷 (Block-Separated Format):**
  원문 전체를 컨텍스트로 먼저 입력한 뒤, 이에 매핑되는 번역문 전체를 한 번에 출력하도록 유도하는 포맷이다. 시 전체의 구조적 일관성, 연(Stanza)과 연 사이의 문맥적 비약, 전체적인 미학적 분위기(Tone & Manner)를 보존하고 번역하기에 적합하다.
  ```json
  {
    "instruction": "다음 외국어 시를 원작의 미학적 분위기를 살려 한국어로 번역하시오.",
    "input": "[Original Poem]\nSein Blick ist vom Vorübergehn der Stäbe...\n[Stanza 2]...",
    "output": "[Translated Poem]\n그의 눈길은 창살들이 지나가는 것을 보다가...\n[Stanza 2]..."
  }
  ```

* **시행/연 교차(Interleaved) 포맷:**
  행 단위 혹은 연 단위로 원문과 번역문을 번갈아 배치하는 포맷이다. 번역 과정에서 발생하는 미시적인 단어 대응, 소리(음운)의 전이, 혹은 한 행 단위의 시각적 형태 대칭성을 엄격하게 제어하여 학습해야 할 때 사용한다.
  ```json
  {
    "instruction": "다음 시의 각 행(Line)을 대칭적으로 번역하고 정렬된 태그 형식으로 출력하시오.",
    "input": "Sein Blick ist vom Vorübergehn der Stäbe...",
    "output": "<source_start>Sein Blick ist vom Vorübergehn der Stäbe<source_end>\n<target_start>그의 눈길은 창살들이 지나가는 것을 보다가<target_end>"
  }
  ```

### 3. 교차 언어 정렬 및 감정 매핑 메타데이터 (Cross-lingual Alignment & Sentiment Metadata)
학습 데이터 구성 시 원문과 한국어 번역문 간의 미학적 정합도를 정량화하고, 각 언어권 특유의 감정 및 문예적 사조를 인코딩하기 위해 다음과 같은 메타데이터를 어노테이션한다.

* **의미론적 거리 점수 (Semantic Distance Scores):**
  - **다국어 임베딩 유사도**: LaBSE 또는 mUSE와 같은 다국어 문장 임베딩 모델을 활용하여 원문 행/연과 한국어 번역 행/연 간의 코사인 유사도를 측정하고 `semantic_similarity` 점수로 기록한다.
  - **시적 도약도 인덱스**: 단순 직역일 경우 유사도가 매우 높게 나타나지만, 시인 번역자의 '의도적 오역'이나 '시적 변용'이 일어날 경우 유사도가 낮아진다. 이 차이를 통해 '직역(High similarity)'과 '시적 창조(Low similarity)'의 학습 가중치를 조정할 수 있도록 메타데이터로 제공한다.

* **교차 언어 감정/정서 매핑 (Cross-lingual Sentiment & Emotion Mappings):**
  - **정서 가치 및 각성도(Valence-Arousal-Dominance) 매핑**: 원문에서 느껴지는 감정적 정조와 번역문에서의 감정적 정조가 언어적 차이로 인해 어떻게 변이되는지 어노테이션한다.
  - **문화적 정서 치환 태그**: 영미시의 'Melancholy(우울)'가 한국어 시어의 '한(恨)'이나 '애수(哀愁)'로 치환되어 표현된 경우, 단순 1:1 감정 단어 대응이 아니라 감정적 맥락이 현지화(Localization)된 방식을 메타 정보(`emotion_adaptation_type`)에 기록하여 모델이 자연스러운 정서 전이를 유도한다.

* **미학적 사조 및 스타일 전이 태그 (Aesthetic Concept & Style Transfer Tags):**
  - 각 시집 및 시인 데이터에 미학적 사조 정보를 태깅하여 모델이 특정 미학적 개념을 학습할 수 있게 한다.
    - **영미 현대시/이미지즘(English Modernism/Imagism)**: `imagism`, `sensory_imaging`, `objective_correlative` (구체적 사물 묘사, 주관적 감정 억제)
    - **독일 사물시(German Dinggedicht)**: `dinggedicht`, `non-human_perspective` (인간 중심에서 벗어나 사물의 관점에서 세계를 집요하게 관찰하여 주객을 전도시키는 기법)
    - **일본 하이쿠(Japanese Haiku)**: `kireji` (사유의 도약을 이루는 끊어 읽는 말), `kigo` (계절을 매개하는 사물)
  - 이 태그들은 지도 미세 조정(SFT) 프롬프트 설계 시 조건화 변수(Conditioning Variables)로 인입되어 "독일 사물시(Dinggedicht)의 시선으로 도시의 사물을 묘사하시오"와 같은 지시 수행이 가능하도록 돕는다.

### 4. 번역 품질 필터링 및 메타데이터 가이드라인
* **역자 정보(Translator's ID) 필수 기입**: 황현산, 김현, 김춘수 등 시학적 완성도가 검증된 번역을 선별하여 번역자의 이름을 데이터셋에 라벨링한다.
* **직역 vs 시적 변용 레이블**: 원문의 구조를 그대로 살린 '직역(Literal)'과 한국어의 자수율/가락을 살려 변형한 '의역/시적 변용(Poetic Adaptation)'을 레이블로 명시하여, 모델이 필요에 따라 기계적 의미 매핑 또는 예술적 변용 방식을 구분하여 학습하게 한다.
* **역자 주석(Translator's Commentaries) 연계**: 특정 시어 번역의 미학적 이유를 서술한 해설서나 시평이 존재할 경우, 평론 데이터(`poetry_criticism.md`)와 연계하여 메타 지식으로 제공한다.

## 교차 미학적 전이 전략 (Cross-lingual Aesthetic Transfer Strategies)

모델이 서구 및 동아시아 시학의 구조와 수사학적 자산을 내재화하여, 한국어 시 창작 시 이를 유기적으로 융합·변용(Hybridization)하도록 유도하는 구체적인 학습 및 생성 전략이다.

### 1. 영미 현대시의 이미지즘(Imagism) 및 시각적 등가물 전이
* **핵심 미학**: 에즈라 파운드(Ezra Pound)나 윌리엄 카를로스 윌리엄스(William Carlos Williams) 스타일의 고도로 압축된 사물 묘사, 구어적 직접성, 주관적 감정 표출의 억제.
* **전이 메커니즘**:
  - **시각적 등가물(Objective Correlative) 추출**: 훈련 과정에서 영미 이미지즘 시의 명사-형용사 조합 및 묘사 방식(예: "Red wheelbarrow glazed with rain water" -> "빗물에 반짝이는 빨간 외바퀴 손수레")을 한국어의 구어체 및 사물 지향 언어와 결합시킨다.
  - **관념의 구체화(Wallace Stevens 스타일)**: 추상 관념을 현대 도시나 자연의 구체적이고 낯선 시각 이미지로 번역하는 패턴을 학습 데이터 내 시평/평론을 통해 강화한다.

### 2. 일본 하이쿠(Haiku)의 정형 구조 및 여백 전이
* **핵심 미학**: 5-7-5조(음수율)의 압축성, 계어(Kigo, 계절을 지시하는 매개체), 끊어 읽는 말(Kireji, 순간의 침묵과 사유의 도약).
* **전이 메커니즘**:
  - **한국어 음수율 변환**: 일본어의 5-7-5 음절 제약을 한국어 시조의 3-4조나 현대시의 4-4조 음수율에 대응시키거나, 3행 구성(초-중-종장 변형)으로 변환 학습한다.
  - **여백과 순간의 종결성**: 문장 종결어미를 극도로 생략하고 명사형 종결(`~임`, `~것`, 명사 단독 배치) 또는 시행 중간의 강제 단절을 통해 하이쿠 특유의 '정지된 순간의 울림'을 생성 모듈에 이식한다.
  - **현대적 계어(Kigo) 전이**: '편의점 조명', '노이즈 캔슬링', '지하철 스크린도어'와 같은 현대 도시의 소품을 계어(Kigo)처럼 순간의 공간성을 규정하는 매개체로 치환하도록 데이터 증강 과정에서 유도한다.

### 3. 중국 현대시(몽롱시)의 비선형적 비약 전이
* **핵심 미학**: 베이다오(北島), 구청(顾城) 등의 몽롱시(朦胧诗)에 나타나는 비논리적 인과관계, 고도로 상징적인 이미지 비약, 통사적 불연속성.
* **전이 메커니즘**:
  - **통사적 단절 학습**: 주어나 조사를 의도적으로 생략하고 독립된 이미지를 병렬 배치하여 한국어 행/연 사이의 미학적 여백과 다의성을 증폭시킨다.
  - **한자어 감각의 현대적 차용**: 두 개 이상의 이질적 개념(예: "황폐한 시간", "비명 지르는 돌")을 압축적 복합어로 결합해 한국어 시어 조합의 신선함(Novelty)을 유도한다.

### 4. 독일 사물시(Dinggedicht)의 내면화 및 시점 제어
* **핵심 미학**: 릴케(Rainer Maria Rilke) 스타일의 사물을 집요하게 관찰하여 주체와 객체의 경계를 무너뜨리는 시학.
* **전이 메커니즘**:
  - **사물 관점(Non-human Perspective)의 화자 정의**: CoT 창작 노트 단계에서 1인칭 화자를 인간이 아닌 '방 내부의 의자', '시들어가는 장미', '고장 난 환풍기' 등으로 고정한 뒤 대상을 묘사하도록 구조화하여 사물 시학의 내면화를 수행한다.

## 미결 사항

- [Ph1] 번역문-원문 병렬 데이터 수집 시, 기계 번역(NMT)을 활용해 자동 정렬을 수행할 경우 발생할 수 있는 시학적 왜곡(예: 행/연 구조 붕괴)을 방지할 수 있는 정량적 필터링 기준은 무엇인가?
- [TODO] 일본 하이쿠의 끊어 읽는 말(Kireji)이나 계어(Kigo)의 미학적 기능을 한국어 현대시로 이식할 때, 정형적 3행 구조를 강제할 것인가 혹은 한국어 구어체의 호흡을 고려한 가변적 줄바꿈을 권장할 것인가?
  - 특히 파인튜닝 시 도입되는 `<행갈이>` 토큰의 위치가 하이쿠의 순간적인 호흡(Kireji)과 시각적 단절을 재현하는 데 어떤 역할을 하는지 실험적 분석이 필요하다.
  - 현대적 계어(Kigo)가 등장한 직후 의도적으로 `<행갈이>`를 강제하여 시적 긴장감과 여백을 만들어내는 프롬프트(CoT 창작 노트) 설계가 유효한가?
- [Ph1] 다국어 시 텍스트(독일어, 중국어, 일본어 등)의 원문 학습 시, 한국어 중심의 토크나이저 어휘 사전(Vocabulary)에서 발생하는 토큰 파편화(Fragmentation) 문제를 해결하고 임베딩 공간에서 교차 언어적 정렬을 극대화할 수 있는 토큰화 기법은 무엇인가?
- [Ph1] 교차 언어 감정 매핑 메타데이터 구축 시, 번역 과정에서 발생하는 정서의 문화적 변이(예: 원문의 'Melancholy'가 한국어 번역에서 '한(恨)'의 정조로 변용될 때)로 인한 감정 유사도 점수의 왜곡을 완화하고, 다차원적인 감정 보존율을 표현할 수 있는 평가 척도는 무엇인가?
- [Ph2] SFT 학습 데이터에서 블록 분리형 포맷과 교차(Interleaved) 포맷의 최적의 조합 비율은 무엇이며, 두 포맷이 학습 모델의 창작 중 교차 미학적 스타일 전이 능력(예: 한국어로 시를 지을 때 영미 현대시나 독일 사물시 기법을 융합하는 능력)에 어떤 상이한 효과를 미치는가?
