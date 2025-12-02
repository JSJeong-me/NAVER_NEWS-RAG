지금까지 작성한 'design skeleton'에 'llm.client, llm.summarizer, llm.classifier에 대한 구체적인 인터페이스/프롬프트 설계를' 포함하여 개선된 'design skeleton' 을 작성해 주세요.



바로 LLM까지 포함해서 한 번에 돌아가는 **개선된 design skeleton**으로 정리해볼게요.
기존 구조는 그대로 두고, 여기에

* `llm.client`
* `llm.summarizer`
* `llm.classifier`

를 추가하고, 기존 모듈(`SentimentAnalyzer`, `RiskTagger`, `RegimeClassifier`, `RecommendationEngine`, `QueryService`)과 **어떻게 연결되는지**까지 포함해서 설계만 합니다. (여전히 구현은 없음)

---

## 0. 상위 아키텍처 변화 요약

추가되는 큰 변화는 세 가지입니다.

1. **새 패키지: `news_regime.llm`**

   * `LlmClient`: OpenAI 등 LLM 호출 래퍼
   * `NewsSummarizer`: 기사/종목/Regime 요약 담당
   * `NewsClassifier`: sentiment + risk LLM 분류 담당

2. **Config 확장**

   * `LlmConfig`: 모델명, temperature, max_tokens, 동시 호출 제한 등

3. **기존 분석/추천/Regime 모듈과의 연결**

   * `SentimentAnalyzer`/`RiskTagger`에 LLM 기반 구현체 추가
   * `RecommendationEngine`에서 LLM 요약을 써서 `reason_summary` 생성 옵션
   * `RegimeClassifier`에서 LLM으로 자연어 explanation 생성 옵션
   * `QueryService`에서 LLM 요약을 활용한 “읽기 좋은 리포트” 생성

---

## 1. Config 확장

### 1.1 `news_regime/config/models.py` – LlmConfig 추가

```python
from dataclasses import dataclass

@dataclass
class LlmConfig:
    """
    LLM 호출 관련 설정.

    - model_name: OpenAI 모델명 (예: "gpt-4.1-mini")
    - temperature: 출력 다양성
    - max_tokens: 응답 최대 토큰
    - request_timeout: HTTP 타임아웃
    - max_concurrent_requests: 동시에 요청할 수 있는 최대 수
    - enable_llm_for: 어떤 기능에 LLM을 쓸지 on/off (요약/분류/설명)
    """
    model_name: str = "gpt-4.1-mini"
    temperature: float = 0.2
    max_tokens: int = 512
    request_timeout: float = 30.0
    max_concurrent_requests: int = 4

    # 기능 플래그
    use_for_article_summary: bool = True
    use_for_symbol_summary: bool = True
    use_for_regime_explanation: bool = True
    use_for_sentiment_risk: bool = False  # 기본은 룰 기반, 필요시 True
```

`AppConfig`에 LlmConfig 추가:

```python
@dataclass
class AppConfig:
    db: DbConfig
    crawl: CrawlConfig
    analysis: AnalysisConfig
    regime: RegimeConfig
    archive: ArchiveConfig
    llm: LlmConfig          # ★ LLM 설정 추가
    log_level: str = "INFO"
    timezone: str = "Asia/Seoul"
```

---

## 2. LLM 도메인/DTO 확장

### 2.1 `news_regime/domain/models.py` – ArticleSummary 추가

```python
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class ArticleSummary:
    """
    단일 뉴스 기사에 대한 LLM 요약 결과.
    """
    news_id: str
    short_summary: str             # 2~3줄 요약
    bullet_points: List[str]       # 핵심 포인트 리스트
```

(그리고 필요한 곳에서 import 해서 사용)

---

## 3. LLM Layer 패키지 설계

새 디렉토리:

```text
news_regime/
  llm/
    __init__.py
    client.py
    prompts.py
    summarizer.py
    classifier.py
```

---

## 3.1 `news_regime/llm/client.py` – LlmClient 설계

```python
# news_regime/llm/client.py
from __future__ import annotations
from dataclasses import dataclass
from typing import List, Dict, Any, Optional

from news_regime.config.models import LlmConfig


@dataclass
class LlmMessage:
    """
    LLM chat completion 형식의 메시지 구조.
    """
    role: str      # "system" | "user" | "assistant"
    content: str


@dataclass
class LlmResponse:
    """
    LLM 응답 래퍼.
    """
    raw_text: str                  # 원본 텍스트
    parsed_json: Optional[Dict]    # JSON 응답일 경우 파싱 결과 (없으면 None)


class LlmClient:
    """
    OpenAI 등 LLM API 호출을 래핑하는 공통 클라이언트.

    - 단일 generate
    - 배치 generate_batch
    - JSON 응답을 위한 response_format 지원
    """

    def __init__(self, config: LlmConfig) -> None:
        self.config = config
        # TODO: 실제 OpenAI client 초기화 등

    def generate(
        self,
        messages: List[LlmMessage],
        response_format: Optional[Dict[str, Any]] = None,
    ) -> LlmResponse:
        """
        단일 요청에 대한 LLM 응답 생성.

        :param messages: system/user/assistant 역할의 메시지 리스트
        :param response_format: JSON schema 등 응답 포맷 힌트
        :return: LlmResponse (raw_text, parsed_json)
        """
        # TODO: OpenAI ChatCompletion 호출 + 예외/재시도/타임아웃 처리
        raise NotImplementedError

    def generate_batch(
        self,
        batch_messages: List[List[LlmMessage]],
        response_format: Optional[Dict[str, Any]] = None,
    ) -> List[LlmResponse]:
        """
        여러 요청을 한 번에 보내거나, 순차적으로 처리하는 배치 API.

        - 비용/지연 최적화를 위해 기사 여러 개를 묶어서 호출할 때 사용.
        """
        # TODO: 간단한 for 루프 or async 기반 배치 처리
        raise NotImplementedError
```

---

## 3.2 `news_regime/llm/prompts.py` – Prompt 템플릿 설계

LLM 호출에 쓸 **system/user prompt 템플릿**을 모아두는 곳입니다.
(여기서는 설계/형태만, 실제 문자열은 예시 수준으로)

```python
# news_regime/llm/prompts.py
from __future__ import annotations
from dataclasses import dataclass
from typing import Dict, Any


@dataclass
class PromptTemplate:
    """
    프롬프트 템플릿 정의용 데이터 구조.

    - system: 시스템 역할 설명
    - user_template: python format 문자열 (필요한 변수는 dict로 전달)
    - response_format: JSON schema 힌트 (선택)
    """
    system: str
    user_template: str
    response_format: Dict[str, Any] | None = None


# 1) 기사 요약 프롬프트
ARTICLE_SUMMARY_PROMPT = PromptTemplate(
    system=(
        "You are a financial news assistant. "
        "Summarize Korean stock market news articles for traders. "
        "Use concise Korean, suitable for a professional trader."
    ),
    user_template=(
        "제목:\n{title}\n\n"
        "본문:\n{body}\n\n"
        "위 기사의 핵심 내용을 트레이더 관점에서 한글로 요약해 주세요.\n"
        "- 2~3문장 요약(short_summary)\n"
        "- 핵심 포인트 bullet 3~5개(bullet_points)\n"
        "JSON 형식으로만 출력하세요:\n"
        "{{\n"
        '  "short_summary": "....",\n'
        '  "bullet_points": ["...", "..."]\n'
        "}}"
    ),
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "article_summary",
            "schema": {
                "type": "object",
                "properties": {
                    "short_summary": {"type": "string"},
                    "bullet_points": {
                        "type": "array",
                        "items": {"type": "string"},
                    },
                },
                "required": ["short_summary", "bullet_points"],
                "additionalProperties": False,
            },
        },
    },
)


# 2) Sentiment + Risk 분류 프롬프트
SENTIMENT_RISK_PROMPT = PromptTemplate(
    system=(
        "You are a financial news classifier. "
        "You read Korean stock news and classify sentiment and risk flags "
        "for trading decision support."
    ),
    user_template=(
        "제목:\n{title}\n\n"
        "본문:\n{body}\n\n"
        "이 기사를 기반으로 다음 항목을 평가하세요:\n"
        "- sentiment_label: -2, -1, 0, 1, 2 중 하나 (매우부정/부정/중립/긍정/매우긍정)\n"
        "- sentiment_score: -1.0 ~ 1.0 범위 실수\n"
        "- regulatory_risk: 규제/정부정책/수사/제재 등 리스크가 있으면 true, 아니면 false\n"
        "- earnings_risk: 실적악화/가이던스 하향 등 실적 관련 리스크가 있으면 true\n"
        "- credit_risk: 재무건전성/채무불이행 관련 리스크가 있으면 true\n"
        "JSON 형식으로만 답변하세요:\n"
        "{{\n"
        '  "sentiment_label": 0,\n'
        '  "sentiment_score": 0.0,\n'
        '  "regulatory_risk": false,\n'
        '  "earnings_risk": false,\n'
        '  "credit_risk": false\n'
        "}}"
    ),
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "sentiment_risk_classification",
            "schema": {
                "type": "object",
                "properties": {
                    "sentiment_label": {"type": "integer"},
                    "sentiment_score": {"type": "number"},
                    "regulatory_risk": {"type": "boolean"},
                    "earnings_risk": {"type": "boolean"},
                    "credit_risk": {"type": "boolean"},
                },
                "required": [
                    "sentiment_label",
                    "sentiment_score",
                    "regulatory_risk",
                    "earnings_risk",
                    "credit_risk",
                ],
                "additionalProperties": False,
            },
        },
    },
)


# 3) 종목/Regime 요약 프롬프트 (심볼/Regime 설명 공용으로 활용 가능)
SYMBOL_SUMMARY_PROMPT = PromptTemplate(
    system=(
        "You are a senior equity analyst. "
        "You summarize recent news and signals for a given Korean stock symbol."
    ),
    user_template=(
        "종목 코드: {symbol}\n"
        "요약 대상 기간: 최근 {window_hours}시간\n"
        "기사 수: {num_articles}\n"
        "평균 센티먼트: {avg_sentiment}\n"
        "리스크 플래그 비율: 규제={reg_ratio}, 실적={earn_ratio}, 신용={credit_ratio}\n"
        "주요 뉴스 bullet:\n"
        "{top_bullets}\n\n"
        "위 정보를 바탕으로 트레이더 관점에서\n"
        "핵심 포인트를 2~3문장으로 요약해 주세요."
    ),
    response_format=None,  # 자유 텍스트
)
```

※ Regime 설명용 Prompt는 비슷한 방식으로 하나 더 둘 수 있습니다.

---

## 3.3 `news_regime/llm/summarizer.py` – NewsSummarizer 설계

```python
# news_regime/llm/summarizer.py
from __future__ import annotations
from typing import List
from datetime import datetime

from news_regime.domain.models import (
    NewsArticleDetail,
    ArticleSummary,
    SymbolAggregatedStats,
    GlobalNewsStats,
    RegimeStatus,
)
from news_regime.llm.client import LlmClient, LlmMessage
from news_regime.llm.prompts import (
    ARTICLE_SUMMARY_PROMPT,
    SYMBOL_SUMMARY_PROMPT,
)


class NewsSummarizer:
    """
    LLM을 사용하여:
    - 단일 기사 요약
    - 종목 단위 요약 (symbol summary)
    - Regime 설명 텍스트 생성

    을 담당하는 고수준 유틸.
    """

    def __init__(self, client: LlmClient) -> None:
        self.client = client

    # 1) 단일 기사 요약
    def summarize_article(self, detail: NewsArticleDetail) -> ArticleSummary:
        """
        단일 뉴스 기사에 대해 short_summary + bullet_points 를 생성.

        - 고 impact 기사 또는 사용자가 요청한 기사에만 호출.
        """
        # TODO: ARTICLE_SUMMARY_PROMPT 사용해 LLM 호출 및 ArticleSummary 생성
        raise NotImplementedError

    def summarize_articles_batch(
        self,
        details: List[NewsArticleDetail],
    ) -> List[ArticleSummary]:
        """
        여러 기사를 한 번에 요약하는 배치 버전.

        - 성능/비용 최적화를 위해 batch_generate 사용 가능.
        """
        # TODO: generate_batch 사용, 응답 JSON parsing
        raise NotImplementedError

    # 2) 종목 단위 요약 (symbol summary)
    def summarize_symbol(
        self,
        symbol_stats: SymbolAggregatedStats,
        article_summaries: List[ArticleSummary],
    ) -> str:
        """
        종목 단위 요약 텍스트 생성.

        - AggregatedStats + 대표 기사 요약들을 사용해서
          트레이더에게 보여줄 짧은 설명을 만든다.
        """
        # TODO: SYMBOL_SUMMARY_PROMPT 사용해 자연어 요약 생성
        raise NotImplementedError

    # 3) Regime 설명 요약
    def summarize_regime(
        self,
        stats: GlobalNewsStats,
        base_regime: RegimeStatus,
    ) -> str:
        """
        룰 기반으로 결정된 RegimeStatus 를 바탕으로,
        사람이 읽기 좋은 한/두 문장 설명을 LLM으로 생성.

        - 별도 Regime 설명용 PromptTemplate 를 정의해서 사용.
        """
        # TODO: Regime용 PromptTemplate 정의 후 호출
        raise NotImplementedError
```

---

## 3.4 `news_regime/llm/classifier.py` – NewsClassifier 설계

```python
# news_regime/llm/classifier.py
from __future__ import annotations
from dataclasses import dataclass
from typing import List

from news_regime.domain.models import NewsArticleDetail
from news_regime.analysis.risk import RiskFlags
from news_regime.llm.client import LlmClient, LlmMessage, LlmResponse
from news_regime.llm.prompts import SENTIMENT_RISK_PROMPT


@dataclass
class LlmSentimentRiskResult:
    """
    LLM을 통해 얻은 sentiment + risk 분류 결과.
    """
    sentiment_label: int      # -2, -1, 0, 1, 2
    sentiment_score: float    # -1.0 ~ 1.0
    risk_flags: RiskFlags


class NewsClassifier:
    """
    LLM 기반 뉴스 분류기.

    - 단일 기사에 대해 sentiment + risk 플래그를 추론
    - 배치 처리도 지원
    """

    def __init__(self, client: LlmClient) -> None:
        self.client = client

    def classify_one(self, article: NewsArticleDetail) -> LlmSentimentRiskResult:
        """
        단일 기사에 대한 sentiment/risk 분류.

        - SENTIMENT_RISK_PROMPT 를 사용해 LLM 호출
        - JSON 응답을 파싱하여 LlmSentimentRiskResult 로 변환
        """
        # TODO: LlmClient.generate 호출 + JSON parsing
        raise NotImplementedError

    def classify_batch(
        self,
        articles: List[NewsArticleDetail],
    ) -> List[LlmSentimentRiskResult]:
        """
        여러 기사에 대해 sentiment/risk 분류를 수행하는 배치 버전.

        - LlmClient.generate_batch 활용
        - 실패한 기사에 대해 fallback 로직(룰 기반 등) 적용 가능
        """
        # TODO: batch 처리 구현
        raise NotImplementedError
```

---

## 4. 기존 Analysis Layer와 LLM의 연결

LLM 통합을 위해 기존 `SentimentAnalyzer` / `RiskTagger` 를 **interface + 구현체** 구조로 조금 확장하면 좋습니다.

### 4.1 `analysis/sentiment.py` – LLM 어댑터 추가

```python
# 기존 SentimentAnalyzer 는 interface 느낌으로 유지
class SentimentAnalyzer:
    def analyze(self, article: NewsArticleDetail) -> float:
        raise NotImplementedError


class RuleBasedSentimentAnalyzer(SentimentAnalyzer):
    """
    기존 룰/사전 기반 구현체.
    """
    def analyze(self, article: NewsArticleDetail) -> float:
        # TODO: 기존 규칙 기반 구현
        raise NotImplementedError


class LlmSentimentAnalyzer(SentimentAnalyzer):
    """
    LLM 기반 구현체.

    - 내부적으로 NewsClassifier.classify_one 을 호출하고
    - 반환된 sentiment_score 를 그대로 전달하거나 보정해서 사용.
    """

    def __init__(self, classifier: "NewsClassifier") -> None:
        self.classifier = classifier

    def analyze(self, article: NewsArticleDetail) -> float:
        # TODO: classifier.classify_one(article).sentiment_score 사용
        raise NotImplementedError
```

### 4.2 `analysis/risk.py` – LLM 기반 RiskTagger 어댑터

```python
class RiskTagger:
    def tag(self, article: NewsArticleDetail) -> RiskFlags:
        raise NotImplementedError


class RuleBasedRiskTagger(RiskTagger):
    """
    키워드 기반 리스크 태깅 구현체.
    """
    def tag(self, article: NewsArticleDetail) -> RiskFlags:
        raise NotImplementedError


class LlmRiskTagger(RiskTagger):
    """
    LLM 결과에서 RiskFlags 를 그대로 가져오는 구현체.

    - NewsClassifier 를 사용.
    """

    def __init__(self, classifier: "NewsClassifier") -> None:
        self.classifier = classifier

    def tag(self, article: NewsArticleDetail) -> RiskFlags:
        # TODO: classifier.classify_one(article).risk_flags 사용
        raise NotImplementedError
```

`HourlyJob`에서 어떤 구현체를 쓸지 선택:

* `config.llm.use_for_sentiment_risk == False` → Rule-based
* `True` → LLM 기반 (또는 Hybrid 로직)

---

## 5. Recommendation & Regime & API 레이어에서 LLM 활용 설계

### 5.1 RecommendationEngine 에서 reason_summary LLM 적용

`recommend/engine.py` 안에 LLM summarizer를 선택적으로 연결:

```python
class RecommendationEngine:
    def __init__(
        self,
        policy: RecommendationPolicy,
        summarizer: "NewsSummarizer | None" = None,
    ) -> None:
        self.policy = policy
        self.summarizer = summarizer   # 없으면 rule 기반 문장 조합

    def generate_recommendations(
        self,
        aggregated_stats: List[SymbolAggregatedStats],
        as_of: datetime,
        top_n: int | None = None,
    ) -> List[RecommendationItem]:
        """
        - 후보 종목 선별 + composite_score 산출
        - summarizer 가 있으면 symbol summary 를 생성해 reason_summary 에 넣는다.
        """
        # TODO: policy 사용해 필터링 + 점수 산출
        # TODO: summarizer.summarize_symbol(...) 호출해서 reason_summary 채우기
        raise NotImplementedError
```

### 5.2 RegimeClassifier 에서 설명만 LLM으로

```python
class RegimeClassifier:
    def __init__(
        self,
        config: RegimeConfig,
        summarizer: "NewsSummarizer | None" = None,
    ) -> None:
        self.config = config
        self.summarizer = summarizer

    def classify(self, stats: GlobalNewsStats, as_of: datetime) -> RegimeStatus:
        """
        1) 룰 기반으로 regime_label + confidence 계산
        2) summarizer 가 있으면 LLM으로 explanation 생성
        3) 아니면 rule 기반으로 간단한 설명 문자열 생성
        """
        # TODO: 1) rule 기반 라벨 결정
        # TODO: 2) summarizer.summarize_regime(...) 호출 (옵션)
        raise NotImplementedError
```

### 5.3 QueryService 에서 심볼 요약 리포트 제공

```python
class QueryService:
    def __init__(
        self,
        session_manager: DbSessionManager,
        summarizer: "NewsSummarizer | None" = None,
    ) -> None:
        self.session_manager = session_manager
        self.summarizer = summarizer

    def get_symbol_news_summary(
        self,
        symbol: str,
        window_hours: int = 24,
    ) -> Dict:
        """
        기본 통계 + 대표 기사 리스트 + (선택적으로) LLM 요약 텍스트 제공.

        반환 예:
        {
          "symbol": "005930",
          "window_hours": 24,
          "num_articles": ...,
          "avg_sentiment": ...,
          "risk_ratios": {...},
          "articles": [...],
          "llm_summary": "...",   # summarizer 사용 시
        }
        """
        # TODO: DB 조회 -> AggregatedStats/ArticleSummary 구성
        # TODO: summarizer.summarize_symbol(...) 호출 (옵션)
        raise NotImplementedError
```

---

## 6. HourlyJob 에서 LLM 의존성 구성

`cli/hourly_job.py`의 `__init__` 부분 일부를 LLM-aware하게 확장:

```python
from news_regime.llm.client import LlmClient
from news_regime.llm.summarizer import NewsSummarizer
from news_regime.llm.classifier import NewsClassifier
from news_regime.analysis.sentiment import (
    RuleBasedSentimentAnalyzer,
    LlmSentimentAnalyzer,
)
from news_regime.analysis.risk import (
    RuleBasedRiskTagger,
    LlmRiskTagger,
)

class HourlyJob:
    def __init__(self) -> None:
        self.config = get_config()
        setup_logging(self.config.log_level)

        # DB
        self.session_manager = DbSessionManager(self.config.db)

        # LLM 클라이언트 및 유틸
        self.llm_client = LlmClient(self.config.llm)
        self.news_summarizer = NewsSummarizer(self.llm_client)
        self.news_classifier = NewsClassifier(self.llm_client)

        # Sentiment / Risk 구현체 선택
        if self.config.llm.use_for_sentiment_risk:
            self.sentiment_analyzer = LlmSentimentAnalyzer(self.news_classifier)
            self.risk_tagger = LlmRiskTagger(self.news_classifier)
        else:
            self.sentiment_analyzer = RuleBasedSentimentAnalyzer()
            self.risk_tagger = RuleBasedRiskTagger()

        # RegimeClassifier & RecommendationEngine 에 summarizer 주입
        self.aggregation_service = AggregationService()
        self.reco_policy = RecommendationPolicy(self.config.analysis)
        self.reco_engine = RecommendationEngine(
            policy=self.reco_policy,
            summarizer=self.news_summarizer if self.config.llm.use_for_symbol_summary else None,
        )
        self.regime_classifier = RegimeClassifier(
            config=self.config.regime,
            summarizer=self.news_summarizer if self.config.llm.use_for_regime_explanation else None,
        )

        # 나머지(크롤러/파서/매핑 등)는 기존과 동일...
```

---

## 7. 요약

이렇게 개선된 design skeleton에서:

* **LLM Layer (`news_regime.llm`)**

  * `LlmClient` : OpenAI 호출 공통 래퍼
  * `NewsSummarizer` : 기사/종목/Regime 요약 생성
  * `NewsClassifier` : sentiment + risk 분류 수행
  * `prompts.py` : JSON 기반 분류/요약 prompt 템플릿 정의

* **기존 Layer와의 결합**

  * `SentimentAnalyzer`/`RiskTagger`는 interface + 구현체 구조
  * LLM 구현체(`LlmSentimentAnalyzer`, `LlmRiskTagger`)는 `NewsClassifier`를 사용
  * `RecommendationEngine`/`RegimeClassifier`/`QueryService`는 `NewsSummarizer`를 주입받아 자연어 요약/설명 생성
  * `HourlyJob`에서 config 에 따라 Rule 기반 ↔ LLM 기반 ↔ Hybrid 선택 가능

이 상태면,
다음 단계에서 바로

* `LlmClient` → 실제 OpenAI Python SDK 호출 구현,
* `NewsSummarizer`/`NewsClassifier` → prompt 포맷에 맞춰 JSON 파싱 구현,

을 차례대로 넣어가면서 **실제 LLM 통합 버전**으로 발전시킬 수 있습니다.
