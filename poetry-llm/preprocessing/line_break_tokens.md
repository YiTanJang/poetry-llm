---
type: Design
title: 행갈이/연갈이 특수 토큰 설계 및 전처리 파이프라인
description: 시의 형식적 핵심인 행갈이와 연갈이를 명시적으로 학습시키기 위한 전처리 및 후처리 변환 파이프라인 설계.
tags: [preprocessing, tokenization, line-break, stanza-break, pipeline]
timestamp: 2026-06-27T00:00:00Z
---

# 행갈이/연갈이 특수 토큰 설계 및 전처리 파이프라인

시의 형식적 미학을 구성하는 가장 중요한 핵심 요소는 **행갈이(line break)**와 **연갈이(stanza break)**입니다. 일반적인 산문 텍스트와 달리, 시에서의 개행은 단순한 줄바꿈 문자가 아닌 시적 긴장감, 호흡의 속도 제어, 리듬감, 그리고 독창적인 이미지 배치를 결정짓는 고도의 의도적 장치입니다.

기존의 일반 거대 언어 모델(LLM)은 개행 문자(`\n`, `\n\n`)를 공백의 연장선으로 취급하여 시적 호흡에 대한 정교한 통제가 불가능했습니다. 본 설계는 시적 구조를 명시적으로 통제하고 학습하기 위해 `<시작>`, `<행갈이>`, `<행갈이:걸침>`, `<연갈이>`, `<여백:N>`, `<끝>`이라는 특수 토큰(Special Tokens)을 도입하고, 이를 처리하기 위한 견고한 전처리 및 후처리 파이프라인을 제시합니다.

---

## 1. 특수 토큰 사양 및 미학적 의의

| 특수 토큰 | 명칭 | 기능 설명 | 미학적·기술적 의의 |
|:---|:---|:---|:---|
| `<시작>` | BOS (Beginning of Sequence) | 시퀀스의 시작점 정의 | 모델에게 새로운 시 창작 태스크의 시작을 선언하고 초기 컨텍스트를 초기화합니다. |
| `<행갈이>` | Line Break | 일반 시행(Line)의 끝을 정의 | 문장 성분 또는 문법적 단위가 종결되며 끝나는 행갈이를 명시합니다. |
| `<행갈이:걸침>` | Enjambment | 행걸침(Enjambment)의 끝을 정의 | 문법/의존 관계가 끊기지 않고 다음 행으로 이어지는 행갈이를 명시하여 시적 리듬과 긴장감을 형성합니다. |
| `<연갈이>` | Stanza Break | 연(Stanza)의 끝을 정의 | 연과 연 사이의 구분(빈 줄)을 나타냅니다. 시공간적 전환과 호흡의 깊은 멈춤을 명시화합니다. |
| `<여백:N>` | Spacing Control | 연속된 N개의 공백 문자 정의 | 시인의 의도적인 타이포그래피적 여백(들여쓰기 및 넓은 공백)을 토크나이저의 왜곡 없이 정확히 보존합니다. |
| `<끝>` | EOS (End of Sequence) | 시퀀스의 종료점 정의 | 시의 유기적 마침표를 나타냅니다. 모델이 스스로 시의 완결성을 인지하고 생성을 멈추도록 유도합니다. |

### 토크나이저 통합 시 주의사항
- 특수 토큰들은 토크나이저(Tokenizer)에 고유한 스페셜 토큰으로 추가되어 단일 ID를 부여받아야 합니다.
- 만약 일반 텍스트로 처리되어 여러 서브워드(Subwords)로 쪼개진다면(예: `<` + `행` + `갈` + `이` + `>`), 모델이 줄바꿈의 구조적 경계를 온전히 인지하기 어렵고 불필요한 연산 낭비가 발생합니다.

---

## 2. 행걸침 및 의도적 여백 식별 규칙 (Rules for Enjambment & Spacing Control)

시의 형식미와 미학적 정교함을 유지하기 위해, 전처리 엔진은 다음 세부 식별 규칙 및 정규식을 기반으로 원본 텍스트에서 행걸침과 여백을 자동으로 탐지합니다.

### 2.1. 행걸침 (`<행갈이:걸침>`) 식별 규칙

행걸침은 문법적으로 연결된 한 문장의 마디가 행의 경계를 넘나드는 현상입니다. 형태소 분석과 문장 부호를 결합한 휴리스틱 알고리즘을 사용합니다.

1. **어미/조사 종결 검사 (Morphological Suffix Check)**:
   - 다음 행이 존재하고 비어있지 않으며, 현재 행의 마지막 단어(어절)가 아래의 한국어 어미/조사(의존 형태소)로 끝날 때 행걸침 후보로 분류합니다.
     - **조사**: 主格 `-이/가`, 目的格 `-을/를`, 屬格 `-의`, 處格 `-에/에서`, 共同格 `-와/과`, 補助事 `-은/는`, `-도`, `-만`, `-조차`, `-마저`, `-부터`, `-까지` 등
     - **어미**: 관형사형 어미(단어를 수식하는 `-은/ㄴ`, `-는`, `-을/ㄹ`, `-던`), 연결 어미(다음 구절을 잇는 `-고`, `-며`, `-아/어`, `-게`, `-지`, `-면서`, `-고서` 등)
2. **문장 부호 및 종결 어미 기반 필터링 (Sentence-Closing Filter)**:
   - 현재 행의 마지막 어절이 종결형 문장 부호(`,`, `.`, `!`, `?`, `~`, `"` 등)로 끝나거나, 한국어 문장 종결 어미(예: `-다`, `-요`, `-죠`, `-오`, `-네`, `-도다`, `-리라`, `-습니다` 등)로 끝나는 경우에는 문법적으로 마디가 마무리된 것으로 보아 일반 `<행갈이>`로 처리합니다.
3. **구문 의존 구문 분석과의 연계 (Dependency Parsing Rule)**:
   - 전처리 단계에서 구문 분석기를 탑재한 경우, 현재 행의 마지막 어절의 지배소(Head)가 다음 행의 첫 번째 또는 두 번째 어절 내에 존재하며, 문법적 관계가 주어-술어, 관형어-체언, 부사어-용언인 경우 높은 우선순위로 `<행갈이:걸침>`을 부여합니다.

### 2.2. 의도적 여백 (`<여백:N>`) 식별 규칙

공백 압축으로 인해 시인의 들여쓰기 및 단어 간 정교한 레이아웃이 유실되는 문제를 방지하기 위해 다음과 같은 규칙으로 여백을 치환합니다.

1. **다중 공백 정규식 매칭 (Regex Rule)**:
   - 연속된 2개 이상의 스페이스(` `)를 탐지합니다.
   - **탐지 정규식**: `(?P<spaces> {2,})`
   - 매칭된 공백의 길이(N)를 계산하여 `<여백:N>` 토큰으로 변환합니다.
2. **탭 문자 변환 규칙**:
   - 탭 문자(`\t`)는 시적 가독성을 일관되게 처리하기 위해 공백 4개(`    `)로 정규화한 후, `<여백:4>`로 인코딩합니다.
3. **시행 첫머리 들여쓰기(Indentation) 규칙**:
   - 시행의 가장 앞에 등장하는 모든 공백(1개 이상 포함)은 시인의 의도적인 정렬 효과로 보아 `<여백:N>`으로 변환합니다.
   - 단, 단어 사이에 단 1개만 존재하는 일반 공백은 특수 토큰으로 변환하지 않고 문자 그대로 둡니다.

---

## 3. 전처리 파이프라인 (Preprocessing Pipeline)

원시 시 데이터(Raw Poetry Text)는 작성 기기, 웹 스크래핑 경로 등에 따라 공백 문자가 지저분하게 섞여 있거나 개행 문자가 일관되지 않은 경우가 많습니다. 전처리 엔진은 다음 에지 케이스들을 엄격하게 처리하여 일관된 토큰 스트림으로 변환합니다.

### 처리 대상 에지 케이스
1. **혼합 개행 문자**: Windows 스타일의 `\r\n`과 Unix 스타일의 `\n`, 혹은 오래된 시스템의 `\r`을 `\n`으로 단일 통일합니다.
2. **행 끝 불필요 공백(Trailing Whitespace)**: 시행 끝에 남아 있는 무의미한 띄어쓰기나 탭 문자는 개행 전에 모두 제거합니다.
3. **다중 빈 줄(Multiple Blank Lines)**: 사용자가 실수로 남긴 2개 이상의 연속된 빈 줄(double/triple blank lines)은 하나의 의미적인 연갈이로 병합합니다.
4. **문두/문미 공백 및 개행**: 시 텍스트 전체의 맨 앞이나 맨 뒤에 붙은 불필요한 개행이나 공백을 완전히 제거하여 불필요한 빈 연(Stanza) 생성을 방지합니다.

### 파이썬 전처리 구현 코드

```python
import re

class PoetryPreprocessor:
    """
    원시 시 데이터를 정제하고 행갈이, 행걸침, 연갈이, 여백, 시작, 끝 등의 특수 토큰을 삽입하는 전처리 클래스
    """
    def __init__(self, start_token="<시작>", end_token="<끝>", 
                 line_token="<행갈이>", enjamb_token="<행갈이:걸침>", 
                 stanza_token="<연갈이>"):
        self.start_token = start_token
        self.end_token = end_token
        self.line_token = line_token
        self.enjamb_token = enjamb_token
        self.stanza_token = stanza_token

    def is_enjambment(self, current_line: str, next_line: str) -> bool:
        """
        형태소적 특징 및 규칙을 바탕으로 현재 행과 다음 행 사이가 행걸침(Enjambment)인지 판별합니다.
        """
        # 1. 예외 처리: 다음 행이 없거나 비어있는 경우 행걸침 불가
        if not next_line or not next_line.strip():
            return False
            
        # 2. 현재 행의 마지막 어절 추출 (우측 공백 제거 후)
        line_stripped = current_line.rstrip()
        if not line_stripped:
            return False
            
        # 3. 문장 부호 확인: 종결을 의미하는 부호가 있으면 행걸침 아님
        if re.search(r'[.!?~"]\s*$', line_stripped):
            return False
            
        # 4. 한국어 조사 및 어미 결합에 의한 행걸침 휴리스틱 매칭
        # - 조사: 이, 가, 을, 를, 의, 에, 에서, 에게, 와, 과, 로, 으로, 은, 는, 도, 만, 조차, 마저, 부터, 까지, 하고
        # - 어미 (연결 및 관형사형): 고, 며, 게, 지, 아, 어, 고서, 면서, 은, 는, 던, 을, ㄹ, ㄴ
        enjambment_pattern = (
            r'(이|가|을|를|의|에|에게|에서|와|과|로|으로|고|며|고서|은|는|도|만|조차|마저|부터|까지|하고|이랑)$|'
            r'([가-힣](은|는|던|을|ㄹ|ㄴ|어|아|게|지|면서))$'
        )
        
        # - 종결 어미인 경우 명시적 배제
        closing_pattern = r'(다|요|네|오|군|죠|ㅂ니다|습니다|는다|ㄴ다|도다|리라|거라|어라|아라)\s*$'
        
        if re.search(closing_pattern, line_stripped):
            return False
            
        if re.search(enjambment_pattern, line_stripped):
            return True
            
        return False

    def process_spacing(self, line: str) -> str:
        """
        라인 내부의 탭 및 다중 공백, 시행 첫머리 공백을 <여백:N>으로 변환합니다.
        """
        # 1. 탭 문자를 공백 4개로 통일
        line = line.replace('\t', '    ')
        
        # 2. 시행 첫머리 공백(들여쓰기) 감지 및 변환
        leading_spaces = re.match(r'^( +)', line)
        if leading_spaces:
            space_count = len(leading_spaces.group(1))
            line = f"<여백:{space_count}>" + line[space_count:]
            
        # 3. 시행 중간의 2개 이상 연속된 다중 공백 감지 및 변환
        # 시행 첫머리 토큰화 후 중간 공백들을 변환
        def replacer(match):
            return f"<여백:{len(match.group(1))}>"
            
        # 단어 사이의 2개 이상 공백 매칭
        line = re.sub(r'( {2,})', replacer, line)
        return line

    def preprocess(self, text: str) -> str:
        if not text or not text.strip():
            return f"{self.start_token}{self.end_token}"
        
        # 1. 개행 문자 규격화
        normalized_text = text.replace("\r\n", "\n").replace("\r", "\n")
        lines = normalized_text.split("\n")
        
        # 2. 연(Stanza) 단위 그룹화 (빈 줄을 기준으로 분할)
        stanzas = []
        current_stanza = []
        for line in lines:
            # 여백을 제외한 빈 줄 확인 (strip 후 판단)
            if line.strip() == "":
                if current_stanza:
                    stanzas.append(current_stanza)
                    current_stanza = []
            else:
                current_stanza.append(line)
        if current_stanza:
            stanzas.append(current_stanza)
            
        if not stanzas:
            return f"{self.start_token}{self.end_token}"
            
        # 3. 특수 토큰 조립
        processed_parts = [self.start_token]
        num_stanzas = len(stanzas)
        
        for i, stanza in enumerate(stanzas):
            num_lines = len(stanza)
            for j, line in enumerate(stanza):
                # 여백 처리 적용
                processed_line = self.process_spacing(line.rstrip())
                processed_parts.append(processed_line)
                
                # 시 전체의 마지막 행
                if i == num_stanzas - 1 and j == num_lines - 1:
                    processed_parts.append(self.end_token)
                # 연의 마지막 행 (단, 시 전체의 마지막 연이 아님)
                elif j == num_lines - 1:
                    processed_parts.append(self.stanza_token)
                # 동일 연 내부의 행갈이 / 행걸침
                else:
                    next_line = stanza[j + 1]
                    if self.is_enjambment(line, next_line):
                        processed_parts.append(self.enjamb_token)
                    else:
                        processed_parts.append(self.line_token)
                        
        return "".join(processed_parts)
```

---

## 4. 후처리 및 디코딩 파이프라인 (Post-processing Decoding Pipeline)

모델이 생성해낸 특수 토큰 포함 텍스트를 사용자가 읽을 수 있는 일반적인 시 형태(줄바꿈 및 빈 줄 반영)로 역변환합니다. 토크나이저 디코딩 시 토큰 앞뒤로 예기치 않게 추가된 공백(예: `단어 <행갈이>` 또는 `<행갈이> 단어`)을 정밀하게 보정하는 정규식 필터링을 포함합니다.

### 파이썬 후처리 구현 코드

```python
class PoetryPostprocessor:
    """
    특수 토큰이 포함된 모델 출력 시퀀스를 복원하고 공백을 미학적으로 정제하는 후처리 클래스
    """
    def __init__(self, start_token="<시작>", end_token="<끝>", 
                 line_token="<행갈이>", enjamb_token="<행갈이:걸침>", 
                 stanza_token="<연갈이>"):
        self.start_token = start_token
        self.end_token = end_token
        self.line_token = line_token
        self.enjamb_token = enjamb_token
        self.stanza_token = stanza_token

    def decode(self, token_text: str) -> str:
        if not token_text:
            return ""
            
        # 1. EOS(끝) 토큰 이후 잘라내기
        if self.end_token in token_text:
            token_text = token_text.split(self.end_token)[0]
            
        # 2. 시작 토큰 제거
        token_text = token_text.replace(self.start_token, "")
        
        # 3. 토크나이저 디코딩 과정에서 행/연 구분 토큰 주변에 붙은 불필요한 공백 문자 제거
        token_text = re.sub(r'\s*' + re.escape(self.line_token) + r'\s*', self.line_token, token_text)
        token_text = re.sub(r'\s*' + re.escape(self.enjamb_token) + r'\s*', self.enjamb_token, token_text)
        token_text = re.sub(r'\s*' + re.escape(self.stanza_token) + r'\s*', self.stanza_token, token_text)
        
        # 4. 여백 복원: <여백:N> -> N개의 스페이스
        # 토크나이저 공백 정제 이후 안전하게 복원
        token_text = re.sub(r'<여백:(\d+)>', lambda m: ' ' * int(m.group(1)), token_text)
        
        # 5. 특수 토큰을 표준 개행 및 빈 줄로 치환
        # <행갈이> -> \n
        # <행갈이:걸침> -> \n
        # <연갈이> -> \n\n
        decoded_text = token_text.replace(self.line_token, "\n") \
                                 .replace(self.enjamb_token, "\n") \
                                 .replace(self.stanza_token, "\n\n")
        
        # 6. 최종 문자열의 좌우 불필요한 여백 및 개행 제거
        return decoded_text.strip()
```

---

## 5. 검증 및 데이터 정합성 테스트 (Validation & Integrity Check)

전처리 및 후처리 과정에서 시의 원래 줄(Line) 수와 연(Stanza) 수가 변질되지 않았는지, 그리고 특수 토큰의 규칙적 분포가 보장되는지 검증하는 엔진입니다.

### 정합성 검증 스키마
1. **행/연 개수 대조**: 원본 텍스트의 유효 시행/연 수와 디코딩된 결과의 시행/연 수가 완전히 동일한가?
2. **토큰 분포 정밀 검사**:
   - `<시작>` 토큰으로 유일하게 시작하며, 스트림 전체에서 단 1회만 발견되는가?
   - `<끝>` 토큰으로 유일하게 종결하며, 스트림 전체에서 단 1회만 발견되는가?
   - 총 행 경계 문자(행갈이 개수 + 행걸침 개수 + 연갈이 개수)는 `총 시행 수 - 1`과 일치하는가?
   - 연갈이 토큰 개수는 `총 연 수 - 1`과 일치하는가?

### 파이썬 검증 및 테스트 코드

```python
class PoetryValidator:
    """
    전처리 및 후처리 파이프라인의 데이터 정합성을 정량적으로 확인하는 검증 클래스
    """
    @staticmethod
    def count_lines_and_stanzas(text: str) -> tuple[int, int]:
        """
        텍스트 내 유효 시행(Line) 및 연(Stanza) 개수를 추출합니다.
        (이때 여백 토큰이나 공백 여부는 라인 판단에 지장을 주지 않아야 함)
        """
        if not text or not text.strip():
            return 0, 0
        
        normalized = text.replace("\r\n", "\n").replace("\r", "\n").strip()
        lines = [line.rstrip() for line in normalized.split("\n")]
        
        total_lines = 0
        stanzas = 0
        in_stanza = False
        
        for line in lines:
            if line.strip() != "":
                total_lines += 1
                if not in_stanza:
                    stanzas += 1
                    in_stanza = True
            else:
                in_stanza = False
                
        return total_lines, stanzas

    @classmethod
    def validate_integrity(cls, raw_text: str, processed_text: str, decoded_text: str) -> dict:
        """
        전체 변환 사이클의 정밀 무결성을 테스트합니다.
        """
        starts_correct = processed_text.startswith("<시작>")
        ends_correct = processed_text.endswith("<끝>")
        
        start_count = processed_text.count("<시작>")
        end_count = processed_text.count("<끝>")
        line_token_count = processed_text.count("<행갈이>")
        enjamb_token_count = processed_text.count("<행갈이:걸침>")
        stanza_token_count = processed_text.count("<연갈이>")
        
        raw_lines, raw_stanzas = cls.count_lines_and_stanzas(raw_text)
        decoded_lines, decoded_stanzas = cls.count_lines_and_stanzas(decoded_text)
        
        expected_total_breaks = max(0, raw_lines - 1)
        actual_total_breaks = line_token_count + enjamb_token_count + stanza_token_count
        
        # 통합 정합성 판정
        is_consistent = (
            raw_lines == decoded_lines and
            raw_stanzas == decoded_stanzas and
            starts_correct and
            ends_correct and
            start_count == 1 and
            end_count == 1 and
            actual_total_breaks == expected_total_breaks and
            stanza_token_count == max(0, raw_stanzas - 1)
        )
        
        return {
            "is_consistent": is_consistent,
            "raw_lines": raw_lines,
            "raw_stanzas": raw_stanzas,
            "decoded_lines": decoded_lines,
            "decoded_stanzas": decoded_stanzas,
            "starts_correct": starts_correct,
            "ends_correct": ends_correct,
            "start_count": start_count,
            "end_count": end_count,
            "line_token_count": line_token_count,
            "enjamb_token_count": enjamb_token_count,
            "stanza_token_count": stanza_token_count
        }
```

---

## 미결 사항

- 어미/조사 모호성에 따른 오분류 극복 방안: 형태소 및 구문 분석기 성능 한계로 인해 시적 언어(예: 도치법, 생략법, 고어체 또는 신조어)에서 조사/어미가 종결형인지 연결/관형형인지 모호한 경우, `<행갈이:걸침>`과 `<행갈이>`를 오분류할 가능성이 존재한다. 이 경우 데이터 정제 프로세스에서 오분류율을 최소화하기 위한 휴리스틱 보완책이나 인간 검수(Human-in-the-loop) 가이드라인을 어떻게 설정해야 하는가?
- 여백 크기의 최대값 임계치($N_{max}$) 결정: 여백 토큰 `<여백:N>`에서 `N`이 비정상적으로 큰 값(예: 80자 이상의 공백)으로 생성될 경우, 모델의 어휘 사전(Vocabulary)과 시각적 가독성에 심각한 왜곡이 생길 수 있다. 적절한 여백 제어 최대값 $N_{max}$는 몇으로 제한해야 하며, 그 이상의 공백은 어떻게 무력화하거나 처리해야 하는가?
- 학습 시 행걸침 토큰 `<행갈이:걸침>`과 일반 행갈이 토큰 `<행갈이>`의 미학적 학습 기여도 평가: 두 종류의 행갈이 토큰이 미학적 독창성(Novelty) 향상에 기여하는 정도를 정량적으로 평가하기 위한 비교 실험(A/B 테스트 또는 토큰별 생성 확률 분석) 설계 방식과, 특정 시적 사조(예: 미래주의, 다다이즘)에 최적화된 행걸침 유도 프롬프트/파인튜닝 기법은 무엇인가?
