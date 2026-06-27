---
type: Design
title: 행갈이/연갈이 특수 토큰 설계 및 전처리 파이프라인
description: 시의 형식적 핵심인 행갈이와 연갈이를 명시적으로 학습시키기 위한 전처리 및 후처리 변환 파이프라인 설계.
tags: [preprocessing, tokenization, line-break, stanza-break, pipeline]
timestamp: 2026-06-27T00:00:00Z
---

# 행갈이/연갈이 특수 토큰 설계 및 전처리 파이프라인

시의 형식적 미학을 구성하는 가장 중요한 핵심 요소는 **행갈이(line break)**와 **연갈이(stanza break)**입니다. 일반적인 산문 텍스트와 달리, 시에서의 개행은 단순한 줄바꿈 문자가 아닌 시적 긴장감, 호흡의 속도 제어, 리듬감, 그리고 독창적인 이미지 배치를 결정짓는 고도의 의도적 장치입니다.

기존의 일반 거대 언어 모델(LLM)은 개행 문자(`\n`, `\n\n`)를 공백의 연장선으로 취급하여 시적 호흡에 대한 정교한 통제가 불가능했습니다. 본 설계는 시적 구조를 명시적으로 통제하고 학습하기 위해 `<시작>`, `<행갈이>`, `<연갈이>`, `<끝>`이라는 특수 토큰(Special Tokens)을 도입하고, 이를 처리하기 위한 견고한 전처리 및 후처리 파이프라인을 제시합니다.

---

## 1. 특수 토큰 사양 및 미학적 의의

| 특수 토큰 | 명칭 | 기능 설명 | 미학적·기술적 의의 |
|:---|:---|:---|:---|
| `<시작>` | BOS (Beginning of Sequence) | 시퀀스의 시작점 정의 | 모델에게 새로운 시 창작 태스크의 시작을 선언하고 초기 컨텍스트를 초기화합니다. |
| `<행갈이>` | Line Break | 시행(Line)의 끝을 정의 | 같은 연 내부에서 한 행이 끝나고 다음 행으로 넘어감을 표시합니다. 호흡의 미세한 전환과 시적 리듬 단위를 지정합니다. |
| `<연갈이>` | Stanza Break | 연(Stanza)의 끝을 정의 | 연과 연 사이의 구분(빈 줄)을 나타냅니다. 큰 흐름의 변화, 시공간적 전환, 호흡의 깊은 멈춤을 명시화합니다. |
| `<끝>` | EOS (End of Sequence) | 시퀀스의 종료점 정의 | 시의 유기적 마침표를 나타냅니다. 모델이 스스로 시의 완결성을 인지하고 생성을 멈추도록 유도합니다. |

### 토크나이저 통합 시 주의사항
- 특수 토큰들은 토크나이저(Tokenizer)에 고유한 스페셜 토큰으로 추가되어 단일 ID를 부여받아야 합니다.
- 만약 일반 텍스트로 처리되어 여러 서브워드(Subwords)로 쪼개진다면(예: `<` + `행` + `갈` + `이` + `>`), 모델이 줄바꿈의 구조적 경계를 온전히 인지하기 어렵고 불필요한 연산 낭비가 발생합니다.

---

## 2. 전처리 파이프라인 (Preprocessing Pipeline)

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
    원시 시 데이터를 정제하고 행갈이, 연갈이, 시작, 끝 등의 특수 토큰을 삽입하는 전처리 클래스
    """
    def __init__(self, start_token="<시작>", end_token="<끝>", line_token="<행갈이>", stanza_token="<연갈이>"):
        self.start_token = start_token
        self.end_token = end_token
        self.line_token = line_token
        self.stanza_token = stanza_token

    def preprocess(self, text: str) -> str:
        if not text or not text.strip():
            return f"{self.start_token}{self.end_token}"
        
        # 1. 개행 문자 규격화 (\r\n -> \n)
        normalized_text = text.replace("\r\n", "\n").replace("\r", "\n")
        
        # 2. 라인별 분할
        lines = normalized_text.split("\n")
        
        # 3. 우측 공백 제거 (Trailing whitespace cleanup)
        cleaned_lines = [line.rstrip() for line in lines]
        
        # 4. 연(Stanza) 단위 그룹화 (빈 줄을 기준으로 분리)
        stanzas = []
        current_stanza = []
        
        for line in cleaned_lines:
            if line == "":
                if current_stanza:
                    stanzas.append(current_stanza)
                    current_stanza = []
            else:
                current_stanza.append(line)
        if current_stanza:
            stanzas.append(current_stanza)
            
        # 예외 처리: 유효한 연이 전혀 없는 경우
        if not stanzas:
            return f"{self.start_token}{self.end_token}"
            
        # 5. 특수 토큰이 조립된 문자열 빌드
        processed_parts = [self.start_token]
        
        num_stanzas = len(stanzas)
        for i, stanza in enumerate(stanzas):
            num_lines = len(stanza)
            for j, line in enumerate(stanza):
                processed_parts.append(line)
                
                # 시 전체의 마지막 행인 경우
                if i == num_stanzas - 1 and j == num_lines - 1:
                    processed_parts.append(self.end_token)
                # 연의 마지막 행인 경우 (단, 시 전체의 마지막 연이 아닐 때)
                elif j == num_lines - 1:
                    processed_parts.append(self.stanza_token)
                # 동일 연 내부의 행갈이인 경우
                else:
                    processed_parts.append(self.line_token)
                    
        return "".join(processed_parts)
```

---

## 3. 후처리 및 디코딩 파이프라인 (Post-processing Decoding Pipeline)

모델이 생성해낸 특수 토큰 포함 텍스트를 사용자가 읽을 수 있는 일반적인 시 형태(줄바꿈 및 빈 줄 반영)로 역변환합니다. 토크나이저 디코딩 시 토큰 앞뒤로 예기치 않게 추가된 공백(예: `단어 <행갈이>` 또는 `<행갈이> 단어`)을 정밀하게 보정하는 정규식 필터링을 포함합니다.

### 파이썬 후처리 구현 코드

```python
class PoetryPostprocessor:
    """
    특수 토큰이 포함된 모델 출력 시퀀스를 복원하고 공백을 미학적으로 정제하는 후처리 클래스
    """
    def __init__(self, start_token="<시작>", end_token="<끝>", line_token="<행갈이>", stanza_token="<연갈이>"):
        self.start_token = start_token
        self.end_token = end_token
        self.line_token = line_token
        self.stanza_token = stanza_token

    def decode(self, token_text: str) -> str:
        if not token_text:
            return ""
            
        # 1. EOS(끝) 토큰 이후로 생성된 불필요한 텍스트 및 반복 잘라내기
        if self.end_token in token_text:
            token_text = token_text.split(self.end_token)[0]
            
        # 2. 시작 토큰 제거
        token_text = token_text.replace(self.start_token, "")
        
        # 3. 토크나이저 디코딩 과정에서 특수 토큰 주변에 붙은 불필요한 공백 문자 정제
        # 예: "말을 한다 <행갈이> 뒤를 이어서" -> "말을 한다<행갈이>뒤를 이어서"
        # 단, 행 안의 단어 결합을 방지하기 위해 특수 토큰 앞뒤의 공백을 제거함
        token_text = re.sub(r'\s*' + re.escape(self.line_token) + r'\s*', self.line_token, token_text)
        token_text = re.sub(r'\s*' + re.escape(self.stanza_token) + r'\s*', self.stanza_token, token_text)
        
        # 4. 특수 토큰을 표준 개행 및 빈 줄로 치환
        # <행갈이> -> \n
        # <연갈이> -> \n\n
        decoded_text = token_text.replace(self.line_token, "\n").replace(self.stanza_token, "\n\n")
        
        # 5. 최종 문자열의 좌우 불필요한 여백 및 개행 스트립
        return decoded_text.strip()
```

---

## 4. 검증 및 데이터 정합성 테스트 (Validation & Integrity Check)

전처리 및 후처리 과정에서 시의 원래 줄(Line) 수와 연(Stanza) 수가 변질되지 않았는지, 그리고 특수 토큰의 규칙적 분포가 보장되는지 검증하는 엔진입니다.

### 정합성 검증 스키마
1. **행/연 개수 대조**: 원본 텍스트의 유효 시행/연 수와 디코딩된 결과의 시행/연 수가 완전히 동일한가?
2. **토큰 분포 정밀 검사**:
   - `<시작>` 토큰으로 유일하게 시작하며, 스트림 전체에서 단 1회만 발견되는가?
   - `<끝>` 토큰으로 유일하게 종결하며, 스트림 전체에서 단 1회만 발견되는가?
   - 총 행 경계 문자(행갈이 개수 + 연갈이 개수)는 `총 시행 수 - 1`과 일치하는가?
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
        """
        if not text or not text.strip():
            return 0, 0
        
        normalized = text.replace("\r\n", "\n").replace("\r", "\n").strip()
        lines = [line.rstrip() for line in normalized.split("\n")]
        
        total_lines = 0
        stanzas = 0
        in_stanza = False
        
        for line in lines:
            if line != "":
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
        stanza_token_count = processed_text.count("<연갈이>")
        
        raw_lines, raw_stanzas = cls.count_lines_and_stanzas(raw_text)
        dec_lines, dec_stanzas = cls.count_lines_and_stanzas(decoded_text)
        
        expected_total_breaks = max(0, raw_lines - 1)
        actual_total_breaks = line_token_count + stanza_token_count
        
        # 통합 정합성 판정
        is_consistent = (
            raw_lines == dec_lines and
            raw_stanzas == dec_stanzas and
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
            "decoded_lines": dec_lines,
            "decoded_stanzas": dec_stanzas,
            "starts_correct": starts_correct,
            "ends_correct": ends_correct,
            "start_count": start_count,
            "end_count": end_count,
            "line_token_count": line_token_count,
            "stanza_token_count": stanza_token_count
        }
```

---

## 미결 사항

1. **행걸침(Enjambment)과 일반 행갈이의 의미적 구분 방법**:
   전처리 단계에서 문장 성분 의존 구문 분석(Dependency Parsing) 등을 연계하여, 문법적 마디가 끝나지 않고 다음 행으로 이어지는 행걸침을 식별해 별도의 특수 토큰(예: `<행갈이:걸침>`)으로 인코딩해야 하는가? 아니면 일반 `<행갈이>`를 부여하고 이를 모델이 문맥상으로 자연스럽게 학습하게 해야 하는가?

2. **의도된 시적 여백과 타이포그래피 보존 방안**:
   시인에 따라 행 중간에 넓은 간격(예: 3개 이상의 연속 공백)을 두거나, 시행 전체를 들여쓰는 등 타이포그래피 기법을 사용하는 경우가 있다. 이러한 비정형적 여백을 일반 토크나이저의 공백 축소 규칙에 영향받지 않고 보존하기 위해 `<여백:N>` 또는 `<들여쓰기:N>` 같은 특수 제어 토큰을 추가 정의해야 하는가?

3. **기존 다국어 베이스 모델 어휘 사전(Vocabulary)과의 통합 및 임베딩 초기화 최적화**:
   Qwen2.5-32B와 같은 대규모 베이스 모델에 새로운 특수 토큰들을 추가할 때, 기존 어휘 사전에 미치는 영향을 최소화하고 새로 할당된 임베딩 가중치를 단순히 랜덤 값이 아닌 기존 문맥(예: `\n`, `\n\n`)과 의미론적으로 유사한 위치로 안정적이고 빠르게 수렴시키기 위한 가중치 초기화 방법론은 무엇인가?
