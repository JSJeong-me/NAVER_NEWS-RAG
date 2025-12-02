다음 단계에서 바로

LlmClient → 실제 OpenAI Python SDK 호출 구현,

NewsSummarizer/NewsClassifier → prompt 포맷에 맞춰 JSON 파싱 구현,

을 차례대로 넣어가면서 실제 LLM 통합 버전으로 작성해 주세요.


아래는 **지금까지 만든 design skeleton에 LLM 통합을 실제로 넣은 버전**입니다.

* `llm.client` → OpenAI Python SDK 실제 호출 + JSON 파싱 헬퍼
* `llm.prompts` → Summarizer / Classifier용 JSON Schema 포함 프롬프트
* `llm.summarizer` / `llm.classifier` → 프롬프트에 맞춰 LLM 호출 + JSON 파싱하여 dataclass로 변환

이번 단계에서는 **Chat Completions + Structured Outputs (`response_format.json_schema`)** 조합으로 구현했습니다.
(공식 문서 기준으로 `json_schema`는 `gpt-4o-mini`, `gpt-4o-2024-08-06` 등에서 지원됩니다. ([OpenAI Platform][1]))

> ⚙️ 운영 가정
>
> * Python 3.11/3.13 (Jetson AGX Orin)
> * `pip install openai` (공식 Python SDK) ([GitHub][2])
> * 환경변수 `OPENAI_API_KEY` 로 키 설정

---

## 1. `llm/client.py` – OpenAI SDK 연동 + JSON 파서

````python
# file: llm/client.py
from __future__ import annotations

import json
import logging
from dataclasses import dataclass
from typing import Any, Dict, List, Optional, Sequence

from openai import OpenAI
from openai import APIError, APIConnectionError, APIStatusError  # type hints용

logger = logging.getLogger(__name__)


# -----------------------------
# 기본 타입 정의
# -----------------------------
@dataclass
class LlmConfig:
    """
    LLM 호출 공통 설정.

    - model_name: Chat Completions용 모델 이름 (예: 'gpt-4o-mini')
    - temperature: 샘플링 온도. 요약/분류는 0.0~0.3 정도 추천.
    - max_tokens: LLM이 생성할 최대 토큰 수
    - request_timeout: seconds 단위 요청 타임아웃
    """
    model_name: str = "gpt-4o-mini"
    temperature: float = 0.2
    max_tokens: int = 800
    request_timeout: float = 15.0


@dataclass
class LlmMessage:
    """
    Chat 스타일 메시지 한 개.

    role: 'system' | 'user' | 'assistant'
    content: 순수 텍스트 (멀티모달/툴콜은 이 레이어에서는 사용하지 않음)
    """
    role: str
    content: str


@dataclass
class LlmResult:
    """
    LlmClient.generate() 의 표준 결과.

    - raw_text: 모델이 생성한 원본 텍스트
    - parsed_json: JSON 파싱이 성공했을 때 dict / list, 실패시 None
    - usage: OpenAI가 반환하는 토큰 사용량 정보 (있으면 dict, 없으면 None)
    - request_id: OpenAI request id (디버깅용)
    - raw_response: SDK가 반환한 원본 response 객체 (필요시 직접 사용)
    """
    raw_text: str
    parsed_json: Any | None
    usage: Dict[str, Any] | None
    request_id: Optional[str] = None
    raw_response: Any | None = None


# -----------------------------
# LLM 클라이언트 구현
# -----------------------------
class LlmClient:
    """
    OpenAI Python SDK를 감싸는 thin wrapper.

    - Chat Completions API를 사용
    - response_format으로 json_schema/json_object 를 넘기면
      모델이 JSON을 반환하도록 강제
    - generate()는 JSON 파싱까지 수행한 LlmResult를 반환
    """

    def __init__(self, config: LlmConfig) -> None:
        self.config = config

        # OPENAI_API_KEY 는 환경변수로 주입된다고 가정
        # (필요하다면 여기서 api_key, base_url 인자를 직접 넘겨도 됨)
        self._client = OpenAI(timeout=self.config.request_timeout)

    # --------- public API ---------
    def generate(
        self,
        messages: Sequence[LlmMessage],
        response_format: Optional[Dict[str, Any]] = None,
    ) -> LlmResult:
        """
        단일 Chat Completion 호출.

        messages:
            LlmMessage 리스트 (system + user 조합)
        response_format:
            None 이면 일반 텍스트 템플릿
            {"type": "json_schema", "json_schema": {...}} 이면
            Structured Outputs (JSON Schema) 모드로 동작
            {"type": "json_object"} 이면 JSON mode

        반환:
            LlmResult (raw_text + parsed_json 포함)
        """
        api_messages = [
            {"role": m.role, "content": m.content}
            for m in messages
        ]

        try:
            completion = self._client.chat.completions.create(
                model=self.config.model_name,
                messages=api_messages,
                temperature=self.config.temperature,
                max_tokens=self.config.max_tokens,
                response_format=response_format,
            )
        except (APIConnectionError, APIStatusError, APIError) as exc:  # type: ignore[name-defined]
            # 상위 레이어에서 재시도 전략을 짜기 쉽게 예외를 그대로 올림
            logger.exception("LLM 호출 실패: %s", exc)
            raise

        choice = completion.choices[0]
        text = choice.message.content or ""

        usage = None
        if getattr(completion, "usage", None) is not None:
            # pydantic model → dict
            try:
                usage = completion.usage.model_dump()  # type: ignore[attr-defined]
            except Exception:  # pragma: no cover - 방어적
                usage = dict(completion.usage)  # type: ignore[arg-type]

        request_id = getattr(completion, "_request_id", None)

        parsed = self._try_parse_json(text)

        return LlmResult(
            raw_text=text,
            parsed_json=parsed,
            usage=usage,
            request_id=request_id,
            raw_response=completion,
        )

    def generate_batch(
        self,
        batch_messages: Sequence[Sequence[LlmMessage]],
        response_format: Optional[Dict[str, Any]] = None,
    ) -> List[LlmResult]:
        """
        간단한 배치 호출 헬퍼.

        - OpenAI Chat Completions는 여러 '대화'를 한 번에 처리하는
          공식 배치 API가 없으므로
          여기서는 단순 for-loop으로 감싸는 수준에서 구현
        """
        results: List[LlmResult] = []
        for msgs in batch_messages:
            results.append(self.generate(msgs, response_format=response_format))
        return results

    # --------- internal helpers ---------
    @staticmethod
    def _try_parse_json(text: str) -> Any | None:
        """
        LLM이 출력한 텍스트에서 JSON만 뽑아 파싱한다.

        - ```json ... ``` 코드블록이 있는 경우 자동으로 벗겨낸다.
        - 파싱 실패 시 None 반환 (상위에서 fallback 로직 작성 가능)
        """
        cleaned = text.strip()

        # ```json ... ``` or ``` ... ``` 형태 제거
        if cleaned.startswith("```"):
            # 첫 줄과 마지막 줄이 ``` 로 되어 있다고 가정하고 처리
            lines = cleaned.splitlines()
            if len(lines) >= 3 and lines[0].startswith("```"):
                # 첫 줄의 ```json, ``` 등의 언어 태그 제거
                lines = lines[1:]
                # 마지막 줄이 ``` 라면 제거
                if lines[-1].strip().startswith("```"):
                    lines = lines[:-1]
                cleaned = "\n".join(lines).strip()

        # 순수 JSON 파싱 시도
        try:
            return json.loads(cleaned)
        except json.JSONDecodeError:
            logger.debug("LLM JSON 파싱 실패. raw_text=%s", text)
            return None
````

---

## 2. `llm/prompts.py` – 요약 & 분류용 프롬프트 + JSON Schema

뉴스 크롤링 목적에 맞춰 **두 개의 Structured Output 스키마**를 정의했습니다.

* **기사 요약용**: `ARTICLE_SUMMARY_SCHEMA`
* **기사 단위 매수/리스크 판단 + Regime 힌트**: `ARTICLE_CLASSIFICATION_SCHEMA`

```python
# file: llm/prompts.py
from __future__ import annotations

from dataclasses import dataclass
from typing import Any, Dict


@dataclass(frozen=True)
class PromptTemplate:
    """
    단순 텍스트 템플릿 + Structured Output 설정을 묶어 둔 구조체.

    - system: system role 메시지 텍스트 (지시사항)
    - user: user role 메시지 텍스트 (format() 로 변수 삽입)
    - response_format: Chat Completions 'response_format' 에 그대로 전달
    """
    id: str
    description: str
    system: str
    user: str
    response_format: Dict[str, Any] | None = None


# -----------------------------
# 1) 기사 요약용 JSON Schema
# -----------------------------
ARTICLE_SUMMARY_SCHEMA: Dict[str, Any] = {
    "type": "object",
    "properties": {
        "headline_normalized": {
            "type": "string",
            "description": "간결하게 정제된 한국어 헤드라인",
        },
        "summary_ko": {
            "type": "string",
            "description": "핵심 내용만 담은 한국어 3~5문장 요약",
        },
        "summary_en": {
            "type": "string",
            "description": "해외 투자자를 위한 영어 요약 (2~4문장)",
        },
        "sentiment": {
            "type": "string",
            "description": "기사의 전체 톤",
            "enum": [
                "very_negative",
                "negative",
                "neutral",
                "positive",
                "very_positive",
            ],
        },
        "impact_direction": {
            "type": "string",
            "description": "KOSPI/KOSDAQ 전체 방향 관점의 영향",
            "enum": ["bullish", "bearish", "mixed", "none", "unknown"],
        },
        "impact_strength": {
            "type": "string",
            "description": "영향 강도 (시장 전체 기준)",
            "enum": ["low", "medium", "high"],
        },
        "regime_hint": {
            "type": "string",
            "description": "시장 Regime 추정 힌트",
            "enum": ["risk_on", "risk_off", "sideways", "volatile", "unknown"],
        },
        "risk_flags": {
            "type": "object",
            "description": "리스크 관련 플래그",
            "properties": {
                "regulation": {"type": "boolean"},
                "macro": {"type": "boolean"},
                "fx": {"type": "boolean"},
                "credit": {"type": "boolean"},
                "liquidity": {"type": "boolean"},
                "geopolitics": {"type": "boolean"},
                "earnings": {"type": "boolean"},
                "valuation": {"type": "boolean"},
            },
            "required": [
                "regulation",
                "macro",
                "fx",
                "credit",
                "liquidity",
                "geopolitics",
                "earnings",
                "valuation",
            ],
            "additionalProperties": False,
        },
        "theme_tags": {
            "type": "array",
            "description": "기사의 주요 테마/키워드 (자유 텍스트)",
            "items": {"type": "string"},
        },
        "mentions_tickers": {
            "type": "array",
            "description": "명시적으로 언급된 종목 코드 또는 기업명 (문자열 리스트)",
            "items": {"type": "string"},
        },
        "is_buy_candidate": {
            "type": "boolean",
            "description": "이 뉴스가 '매수 종목 후보'를 강화하는 내용인지 여부",
        },
        "buy_candidate_reason": {
            "type": "string",
            "description": "is_buy_candidate가 true일 때, 근거 설명 (한국어)",
        },
        "confidence": {
            "type": "number",
            "minimum": 0.0,
            "maximum": 1.0,
            "description": "요약 및 판단에 대한 신뢰도 (0~1)",
        },
    },
    "required": [
        "headline_normalized",
        "summary_ko",
        "sentiment",
        "impact_direction",
        "impact_strength",
        "risk_flags",
        "is_buy_candidate",
        "confidence",
    ],
    "additionalProperties": False,
}


ARTICLE_SUMMARY_PROMPT = PromptTemplate(
    id="news.article_summary.v1",
    description="네이버 금융 주요뉴스 기사 요약 + 매수/리스크 힌트 추출",
    system=(
        "You are a professional Korean equity research analyst.\n"
        "You receive financial/economic news written in Korean.\n"
        "Your tasks:\n"
        "1) Normalize the headline.\n"
        "2) Summarize the article in Korean (3~5 sentences).\n"
        "3) Provide a short English summary (2~4 sentences) for foreign investors.\n"
        "4) Classify sentiment, impact_direction, impact_strength and regime_hint.\n"
        "5) Set detailed risk_flags.\n"
        "6) Indicate whether the news strengthens 'buy candidate' for any stocks.\n\n"
        "IMPORTANT:\n"
        "- Always respond ONLY with a single JSON object.\n"
        "- Do NOT include any explanations, markdown or comments.\n"
        "- If information is missing, use reasonable 'unknown' / empty defaults."
    ),
    user=(
        "다음은 네이버 금융 주요뉴스 기사 본문과 헤드라인입니다.\n\n"
        "[원본 헤드라인]\n"
        "{headline}\n\n"
        "[기사 본문]\n"
        "{body}\n\n"
        "위 기사를 분석하여 지정된 JSON 스키마에 맞추어 정보를 채워 주세요."
    ),
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "news_article_summary",
            "strict": True,
            "schema": ARTICLE_SUMMARY_SCHEMA,
        },
    },
)

# -----------------------------
# 2) 기사 단위 매수/Regime 분류용 JSON Schema
# -----------------------------
ARTICLE_CLASSIFICATION_SCHEMA: Dict[str, Any] = {
    "type": "object",
    "properties": {
        "headline_normalized": {"type": "string"},
        "primary_tickers": {
            "type": "array",
            "description": "이 기사에 가장 직접적으로 연관된 종목 코드(또는 기업명)",
            "items": {"type": "string"},
        },
        "market_regime": {
            "type": "string",
            "enum": ["risk_on", "risk_off", "sideways", "crash", "euphoric", "unknown"],
        },
        "market_regime_confidence": {
            "type": "number",
            "minimum": 0.0,
            "maximum": 1.0,
        },
        "trade_action": {
            "type": "string",
            "description": "이 뉴스만 놓고 볼 때의 기본 Trade 액션 제안",
            "enum": ["BUY", "ADD", "HOLD", "REDUCE", "EXIT", "AVOID", "IGNORE"],
        },
        "trade_horizon": {
            "type": "string",
            "description": "추천이 유효한 시간 스케일",
            "enum": ["intraday", "swing", "position", "unknown"],
        },
        "sizing_hint": {
            "type": "string",
            "enum": ["small", "normal", "large", "avoid"],
        },
        "bullish_factors": {
            "type": "array",
            "description": "매수/긍정 논리 리스트 (한국어 bullet)",
            "items": {"type": "string"},
        },
        "bearish_factors": {
            "type": "array",
            "description": "매도/부정 논리 리스트 (한국어 bullet)",
            "items": {"type": "string"},
        },
        "risk_overrides": {
            "type": "object",
            "description": "리스크 매니저용 override 신호",
            "properties": {
                "block_entry": {"type": "boolean"},
                "reduce_leverage": {"type": "boolean"},
                "block_short": {"type": "boolean"},
            },
            "required": ["block_entry", "reduce_leverage", "block_short"],
            "additionalProperties": False,
        },
        "overall_confidence": {
            "type": "number",
            "minimum": 0.0,
            "maximum": 1.0,
        },
    },
    "required": [
        "headline_normalized",
        "market_regime",
        "trade_action",
        "overall_confidence",
    ],
    "additionalProperties": False,
}

ARTICLE_CLASSIFICATION_PROMPT = PromptTemplate(
    id="news.article_classification.v1",
    description="기사 단위 매수/회피 신호 및 시장 Regime 추론",
    system=(
        "You are a Korean equity news classifier used by an automated trading system.\n"
        "From each article, you must infer:\n"
        "- Primary tickers affected\n"
        "- Market regime signal\n"
        "- Trade action (BUY/ADD/HOLD/REDUCE/EXIT/AVOID/IGNORE)\n"
        "- Time horizon and sizing hint\n"
        "- Bullish and bearish bullet points\n"
        "- Risk overrides for the risk manager.\n\n"
        "IMPORTANT:\n"
        "- Answer as a single JSON object.\n"
        "- No extra explanation, no markdown.\n"
        "- If unsure, choose 'unknown' or empty arrays but keep schema valid."
    ),
    user=(
        "다음은 한국 주식시장 관련 뉴스 기사입니다.\n\n"
        "[원본 헤드라인]\n"
        "{headline}\n\n"
        "[기사 본문]\n"
        "{body}\n\n"
        "이 기사가 의미하는 매수/매도/회피 관점과 시장 Regime 신호를 위 JSON 스키마에 맞추어 반환하세요."
    ),
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "news_article_classification",
            "strict": True,
            "schema": ARTICLE_CLASSIFICATION_SCHEMA,
        },
    },
)
```

---

## 3. `llm/summarizer.py` – LLM 기반 기사 요약기 구현

```python
# file: llm/summarizer.py
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional, Tuple

from .client import LlmClient, LlmMessage, LlmResult
from .prompts import ARTICLE_SUMMARY_PROMPT, PromptTemplate


@dataclass
class NewsSummary:
    """
    기사 요약 + 기본 영향 정보를 담는 도메인 모델.

    - raw_dict: LLM이 반환한 JSON 전체 (디버깅/확장용으로 보관)
    """
    headline_normalized: str
    summary_ko: str
    summary_en: str
    sentiment: str
    impact_direction: str
    impact_strength: str
    regime_hint: str
    risk_flags: Dict[str, bool]
    theme_tags: List[str] = field(default_factory=list)
    mentions_tickers: List[str] = field(default_factory=list)
    is_buy_candidate: bool = False
    buy_candidate_reason: str = ""
    confidence: float = 0.0
    raw_dict: Dict[str, Any] | None = None

    @classmethod
    def from_payload(cls, data: Dict[str, Any]) -> "NewsSummary":
        """
        LLM에서 반환된 dict(JSON)을 NewsSummary로 변환.

        - required 필드는 기본값 없이 그대로 읽고
        - optional 필드는 get() + 기본값
        """
        return cls(
            headline_normalized=data.get("headline_normalized", ""),
            summary_ko=data.get("summary_ko", ""),
            summary_en=data.get("summary_en", ""),
            sentiment=data.get("sentiment", "neutral"),
            impact_direction=data.get("impact_direction", "unknown"),
            impact_strength=data.get("impact_strength", "low"),
            regime_hint=data.get("regime_hint", "unknown"),
            risk_flags=data.get("risk_flags", {}) or {},
            theme_tags=data.get("theme_tags", []) or [],
            mentions_tickers=data.get("mentions_tickers", []) or [],
            is_buy_candidate=bool(data.get("is_buy_candidate", False)),
            buy_candidate_reason=data.get("buy_candidate_reason", ""),
            confidence=float(data.get("confidence", 0.0) or 0.0),
            raw_dict=data,
        )


class NewsSummarizer:
    """
    LLM 기반 기사 요약기.

    - LlmClient + PromptTemplate 를 주입 받아서 사용
    - summarize_article() 이 핵심 API
    """

    def __init__(
        self,
        llm_client: LlmClient,
        template: PromptTemplate = ARTICLE_SUMMARY_PROMPT,
    ) -> None:
        self.llm = llm_client
        self.template = template

    def summarize_article(
        self,
        headline: str,
        body: str,
    ) -> Tuple[NewsSummary, LlmResult]:
        """
        단일 기사에 대해 요약 + 영향 정보 추출.

        반환:
            (NewsSummary, LlmResult)
            - NewsSummary: 다운스트림 파이프라인에서 바로 쓰는 도메인 오브젝트
            - LlmResult: raw_text/usage 등을 포함한 내부 디버깅 용도
        """
        user_prompt = self.template.user.format(
            headline=headline.strip(),
            body=body.strip(),
        )

        messages = [
            LlmMessage(role="system", content=self.template.system),
            LlmMessage(role="user", content=user_prompt),
        ]

        llm_result = self.llm.generate(
            messages=messages,
            response_format=self.template.response_format,
        )

        if llm_result.parsed_json is None:
            # Structured Output을 기대했는데 JSON 파싱 실패 시 예외
            raise ValueError(
                "NewsSummarizer: LLM did not return valid JSON. "
                "raw_text=%r" % (llm_result.raw_text[:500],)
            )

        if not isinstance(llm_result.parsed_json, dict):
            raise ValueError(
                "NewsSummarizer: Expected JSON object, got %r"
                % type(llm_result.parsed_json)
            )

        summary = NewsSummary.from_payload(llm_result.parsed_json)
        return summary, llm_result
```

---

## 4. `llm/classifier.py` – 기사 단위 매수/Regime 분류기 구현

```python
# file: llm/classifier.py
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Any, Dict, List, Tuple

from .client import LlmClient, LlmMessage, LlmResult
from .prompts import ARTICLE_CLASSIFICATION_PROMPT, PromptTemplate


@dataclass
class ArticleClassification:
    """
    기사 한 건에 대한 Trade/Regime 분류 결과.
    """
    headline_normalized: str
    primary_tickers: List[str]
    market_regime: str
    market_regime_confidence: float
    trade_action: str
    trade_horizon: str
    sizing_hint: str
    bullish_factors: List[str]
    bearish_factors: List[str]
    risk_overrides: Dict[str, bool]
    overall_confidence: float
    raw_dict: Dict[str, Any] | None = None

    @classmethod
    def from_payload(cls, data: Dict[str, Any]) -> "ArticleClassification":
        return cls(
            headline_normalized=data.get("headline_normalized", ""),
            primary_tickers=data.get("primary_tickers", []) or [],
            market_regime=data.get("market_regime", "unknown"),
            market_regime_confidence=float(
                data.get("market_regime_confidence", 0.0) or 0.0
            ),
            trade_action=data.get("trade_action", "IGNORE"),
            trade_horizon=data.get("trade_horizon", "unknown"),
            sizing_hint=data.get("sizing_hint", "small"),
            bullish_factors=data.get("bullish_factors", []) or [],
            bearish_factors=data.get("bearish_factors", []) or [],
            risk_overrides=data.get("risk_overrides", {}) or {},
            overall_confidence=float(data.get("overall_confidence", 0.0) or 0.0),
            raw_dict=data,
        )


class NewsClassifier:
    """
    LLM 기반 기사 단위 Trade/Regime 분류기.

    - summarize 와 별개로, '실제 주문에 쓸 수 있는 신호'만 간결하게 정리하는 역할
    """

    def __init__(
        self,
        llm_client: LlmClient,
        template: PromptTemplate = ARTICLE_CLASSIFICATION_PROMPT,
    ) -> None:
        self.llm = llm_client
        self.template = template

    def classify_article(
        self,
        headline: str,
        body: str,
    ) -> Tuple[ArticleClassification, LlmResult]:
        """
        단일 기사에 대해 Trade/Regime 신호를 추출.

        반환:
            (ArticleClassification, LlmResult)
        """
        user_prompt = self.template.user.format(
            headline=headline.strip(),
            body=body.strip(),
        )

        messages = [
            LlmMessage(role="system", content=self.template.system),
            LlmMessage(role="user", content=user_prompt),
        ]

        llm_result = self.llm.generate(
            messages=messages,
            response_format=self.template.response_format,
        )

        if llm_result.parsed_json is None:
            raise ValueError(
                "NewsClassifier: LLM did not return valid JSON. "
                "raw_text=%r" % (llm_result.raw_text[:500],)
            )

        if not isinstance(llm_result.parsed_json, dict):
            raise ValueError(
                "NewsClassifier: Expected JSON object, got %r"
                % type(llm_result.parsed_json)
            )

        classification = ArticleClassification.from_payload(llm_result.parsed_json)
        return classification, llm_result
```

---

## 5. (선택) 간단한 사용 예시

```python
# file: demo_llm_integration.py
from llm.client import LlmClient, LlmConfig
from llm.summarizer import NewsSummarizer
from llm.classifier import NewsClassifier

if __name__ == "__main__":
    # 1) LLM 클라이언트 준비
    config = LlmConfig(
        model_name="gpt-4o-mini",  # json_schema 지원 모델
        temperature=0.1,
        max_tokens=800,
        request_timeout=20.0,
    )
    llm = LlmClient(config)

    summarizer = NewsSummarizer(llm)
    classifier = NewsClassifier(llm)

    # 예시 기사
    headline = "삼성전자, AI 반도체 투자 확대에 장중 3% 급등"
    body = "..."  # 크롤링된 본문 텍스트

    summary, summary_llm = summarizer.summarize_article(headline, body)
    print("요약:", summary.summary_ko)
    print("매수 후보?:", summary.is_buy_candidate, summary.buy_candidate_reason)

    cls, cls_llm = classifier.classify_article(headline, body)
    print("Regime:", cls.market_regime, cls.market_regime_confidence)
    print("TradeAction:", cls.trade_action, cls.sizing_hint, cls.trade_horizon)
    print("Primary tickers:", cls.primary_tickers)
```

---

## 정리

이 상태에서 바로 다음 작업은:

1. **requirements / 환경 준비**

   * `pip install openai`
   * `export OPENAI_API_KEY=...`

2. **실제 네이버 뉴스 크롤링 파이프라인에 연결**

   * 크롤링된 raw 기사(헤드라인 + 본문)를 `NewsSummarizer`/`NewsClassifier`에 전달
   * 결과 dataclass를 DB에 저장 (또는 매수 후보 랭킹/Regime 집계에 사용)

3. **운영 튜닝 포인트**

   * 모델: `gpt-4o-mini` → 필요시 `gpt-4.1-mini`/`gpt-4o`로 교체
   * temperature / max_tokens / timeout 값 튜닝
   * Schema 필드(예: risk_flags 종류, trade_action enum 등) 세부 조정

원하시면 **다음 단계로는**

* 크롤링 파이프라인에서 이 모듈들을 어떻게 호출할지 (비동기/큐 기반 vs 단순 for-loop)
* LLM 호출 실패/타임아웃 시 fallback 설계
* Regime Aggregator (1시간/일 단위로 기사 결과를 집계해 market regime 결정)
  쪽까지 바로 이어서 설계/코드 작성해 볼 수 있습니다.

[1]: https://platform.openai.com/docs/guides/structured-outputs?utm_source=chatgpt.com "Structured model outputs - OpenAI API"
[2]: https://github.com/openai/openai-python "GitHub - openai/openai-python: The official Python library for the OpenAI API"
