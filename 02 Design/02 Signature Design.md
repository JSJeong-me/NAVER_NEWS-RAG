위 내용으로 '구체적인 모듈/클래스 시그니처 설계'를 진행해 주세요.



아까 설계했던 아키텍처를 그대로 쓰면서, 이번엔 **“Python에서 실제로 만들 클래스/함수의 인터페이스를 어떻게 잡을지”**에 집중해서 정리해볼게요.

> ⚠️ 주의: 아래는 **시그니처/인터페이스 설계**만이고, 내부 로직 구현은 전혀 안 들어갑니다.
> (type hint, 인자/리턴 구조, 모듈 이름·클래스 이름, DTO 구조 위주)

---

## 0. 공통 Domain 모델 (DTO) 설계

먼저 여러 레이어에서 함께 쓰는 **데이터 구조**부터 잡고 갈게요.

### 0.1 `news_regime.domain.models`

```python
class NewsArticleMeta:
    """
    Metadata of a news article as extracted from mainnews list page.
    """
    id: str | None              # 내부 생성용 임시 ID (DB 저장 전에는 None일 수 있음)
    title: str
    url: str
    publisher: str | None
    category: str | None
    summary: str | None
    published_at: "datetime"
    crawled_at: "datetime"


class NewsArticleDetail:
    """
    Full article content including body text.
    """
    meta: NewsArticleMeta
    body_text: str
    reporter: str | None
    tags: list[str] | None


class SymbolMatch:
    """
    Mapping between article and one symbol.
    """
    news_id: str | None          # DB ID or temporary ID
    symbol: str
    match_score: float


class ArticleSentiment:
    """
    Sentiment and risk/impact analysis result for one article.
    """
    news_id: str
    sentiment_score: float       # e.g. -1.0 ~ 1.0
    regulatory_risk_flag: bool
    earnings_risk_flag: bool
    credit_risk_flag: bool
    impact_score: float


class SymbolAggregatedStats:
    """
    Aggregated stats for one symbol over a time window.
    """
    symbol: str
    window_start: "datetime"
    window_end: "datetime"
    num_articles: int

    avg_sentiment: float
    max_sentiment: float
    avg_impact: float
    max_impact: float

    regulatory_risk_ratio: float
    earnings_risk_ratio: float
    credit_risk_ratio: float

    last_article_time: "datetime | None"


class RecommendationItem:
    """
    One buy candidate with scores and explanation.
    """
    symbol: str
    created_at: "datetime"
    sentiment_score: float
    impact_score: float
    composite_score: float
    risk_summary: dict[str, float | bool]
    status: str                   # "ACTIVE" | "WITHDRAWN" | ...
    reason_summary: str


class RegimeStatus:
    """
    Market regime classification result at a given timestamp.
    """
    timestamp: "datetime"
    regime_label: str             # "Bullish" | "Bearish" | ...
    confidence: float
    explanation: str


class GlobalNewsStats:
    """
    Aggregated statistics over all news in a time window for regime decision.
    """
    window_start: "datetime"
    window_end: "datetime"

    num_articles: int
    avg_sentiment: float
    positive_ratio: float
    negative_ratio: float

    regulatory_risk_ratio: float
    earnings_risk_ratio: float
    credit_risk_ratio: float

    macro_news_ratio: float
    sector_news_ratio: float
    single_stock_news_ratio: float
```

---

## 1. Config Layer

### 1.1 `news_regime.config.models`

```python
class DbConfig:
    dsn: str              # e.g. "sqlite:///news_regime.db" or postgres DSN
    echo: bool
    pool_size: int
    timeout: float


class CrawlConfig:
    base_url: str         # "https://finance.naver.com"
    mainnews_path: str    # "/news/mainnews.naver"
    user_agent: str
    request_timeout: float
    max_retries: int
    retry_backoff_sec: float
    min_request_interval_sec: float   # rate limit


class AnalysisConfig:
    sentiment_positive_threshold: float
    sentiment_negative_threshold: float
    min_articles_for_symbol: int
    aggregation_window_hours: int

    # risk thresholds 등 필요 시 추가
    max_regulatory_risk_ratio_for_buy: float
    min_impact_for_buy: float


class RegimeConfig:
    regime_window_hours: int

    bullish_sentiment_threshold: float
    bearish_sentiment_threshold: float
    high_risk_ratio_threshold: float
    volatility_impact_threshold: float


class ArchiveConfig:
    retention_days: int          # 3
    archive_root_dir: str        # "/data/archive"
    file_format: str             # "jsonl" | "parquet"
    batch_size: int              # delete/export batch


class AppConfig:
    db: DbConfig
    crawl: CrawlConfig
    analysis: AnalysisConfig
    regime: RegimeConfig
    archive: ArchiveConfig

    log_level: str               # "INFO" | "DEBUG" | ...
    timezone: str                # "Asia/Seoul"
```

### 1.2 `news_regime.config.loader`

```python
class ConfigLoader:
    def load(self, env: str | None = None) -> AppConfig:
        """
        Load AppConfig from YAML/ENV files depending on environment name.
        """


def get_config(env: str | None = None) -> AppConfig:
    """
    Convenience function to obtain AppConfig singleton for current process.
    """
```

---

## 2. Crawler Layer

### 2.1 `news_regime.crawler.http_client`

```python
class HttpResponse:
    url: str
    status_code: int
    headers: dict[str, str]
    text: str
    elapsed_sec: float


class HttpClient:
    def __init__(self, base_url: str, timeout: float, user_agent: str):
        ...

    def get(self, path: str, params: dict[str, str] | None = None) -> HttpResponse:
        """
        Perform HTTP GET to base_url + path with retry/backoff.
        """
```

### 2.2 `news_regime.crawler.naver_client`

```python
from news_regime.domain.models import NewsArticleMeta, NewsArticleDetail

class NaverNewsClient:
    def __init__(self, http_client: HttpClient, crawl_config: CrawlConfig):
        ...

    def fetch_mainnews_page(self) -> str:
        """
        Fetch raw HTML of mainnews page.
        """

    def fetch_article_page(self, article_url: str) -> str:
        """
        Fetch raw HTML of a specific article page.
        """

    def crawl_mainnews(self) -> list[NewsArticleMeta]:
        """
        High-level: fetch mainnews page and parse it into a list of NewsArticleMeta.
        (내부에서 parser 호출 가능, 또는 별도 파이프라인에서 호출)
        """

    def crawl_article_detail(self, meta: NewsArticleMeta) -> NewsArticleDetail:
        """
        High-level: fetch article page and parse it into NewsArticleDetail.
        """
```

---

## 3. Parser Layer

### 3.1 `news_regime.parser.mainnews_parser`

```python
from news_regime.domain.models import NewsArticleMeta

class MainNewsParser:
    def parse(self, html: str, crawled_at: "datetime") -> list[NewsArticleMeta]:
        """
        Parse mainnews HTML and build list of NewsArticleMeta.
        """
```

### 3.2 `news_regime.parser.article_parser`

```python
from news_regime.domain.models import NewsArticleMeta, NewsArticleDetail

class ArticleParser:
    def parse(self, html: str, meta: NewsArticleMeta) -> NewsArticleDetail:
        """
        Parse article HTML into NewsArticleDetail, including body_text, reporter, tags.
        """
```

---

## 4. Preprocess Layer

### 4.1 `news_regime.preprocess.text_normalizer`

```python
class TextNormalizer:
    def strip_html(self, html: str) -> str:
        """
        Remove HTML tags and return plain text.
        """

    def normalize_whitespace(self, text: str) -> str:
        """
        Normalize whitespace, newlines, etc.
        """

    def normalize_text(self, html_or_text: str) -> str:
        """
        Full pipeline: strip HTML if needed, normalize whitespace, etc.
        """
```

### 4.2 `news_regime.preprocess.time_utils`

```python
class TimeParser:
    def __init__(self, timezone: str):
        ...

    def parse_published_time(self, raw_time_str: str) -> "datetime":
        """
        Convert Naver timestamp string into timezone-aware datetime.
        """
```

---

## 5. Mapping Layer (기사 → 종목)

### 5.1 `news_regime.mapping.symbol_dictionary`

```python
class SymbolInfo:
    symbol: str
    name_kr: str
    name_en: str | None
    aliases: list[str]          # 약어/별칭


class SymbolDictionary:
    def load(self) -> None:
        """
        Load symbol list from file/DB.
        """

    def get_all(self) -> list[SymbolInfo]:
        """
        Return all symbol infos.
        """

    def find_by_symbol(self, symbol: str) -> SymbolInfo | None:
        ...
```

### 5.2 `news_regime.mapping.matcher`

```python
from news_regime.domain.models import NewsArticleDetail, SymbolMatch

class SymbolMatcher:
    def __init__(self, symbol_dict: SymbolDictionary):
        ...

    def match_symbols(self, article: NewsArticleDetail) -> list[SymbolMatch]:
        """
        Analyze article title/body and return list of matching symbols with scores.
        """
```

---

## 6. Analysis Layer

### 6.1 `news_regime.analysis.sentiment`

```python
from news_regime.domain.models import NewsArticleDetail, ArticleSentiment

class SentimentAnalyzer:
    def analyze(self, article: NewsArticleDetail) -> float:
        """
        Return sentiment score for article: -1.0 ~ 1.0.
        """
```

### 6.2 `news_regime.analysis.risk`

```python
from news_regime.domain.models import NewsArticleDetail

class RiskFlags:
    regulatory_risk: bool
    earnings_risk: bool
    credit_risk: bool


class RiskTagger:
    def tag(self, article: NewsArticleDetail) -> RiskFlags:
        """
        Analyze article text and return risk flags.
        """
```

### 6.3 `news_regime.analysis.impact`

```python
from news_regime.domain.models import NewsArticleDetail

class ImpactScorer:
    def score(self, article: NewsArticleDetail) -> float:
        """
        Compute impact score (0~1 or 0~100) for article.
        """
```

### 6.4 `news_regime.analysis.aggregator`

```python
from news_regime.domain.models import (
    ArticleSentiment,
    SymbolAggregatedStats,
    GlobalNewsStats,
)

class AggregationService:
    def aggregate_by_symbol(
        self,
        sentiments: list[ArticleSentiment],
        symbol_map: list["SymbolMatch"],
        window_start: "datetime",
        window_end: "datetime",
    ) -> list[SymbolAggregatedStats]:
        """
        Aggregate article-level analysis into symbol-level stats.
        """

    def aggregate_global(
        self,
        sentiments: list[ArticleSentiment],
        window_start: "datetime",
        window_end: "datetime",
    ) -> GlobalNewsStats:
        """
        Aggregate article-level data into global stats for regime decision.
        """
```

---

## 7. Recommendation Layer

### 7.1 `news_regime.recommend.policy`

```python
from news_regime.domain.models import SymbolAggregatedStats

class RecommendationPolicy:
    def __init__(self, config: AnalysisConfig):
        ...

    def is_buy_candidate(self, stats: SymbolAggregatedStats) -> bool:
        """
        Evaluate if a symbol is a buy candidate according to rules.
        """

    def compute_composite_score(self, stats: SymbolAggregatedStats) -> float:
        """
        Compute a composite score for ranking candidates.
        """
```

### 7.2 `news_regime.recommend.engine`

```python
from news_regime.domain.models import (
    SymbolAggregatedStats,
    RecommendationItem,
)

class RecommendationEngine:
    def __init__(self, policy: RecommendationPolicy):
        ...

    def generate_recommendations(
        self,
        aggregated_stats: list[SymbolAggregatedStats],
        as_of: "datetime",
        top_n: int | None = None,
    ) -> list[RecommendationItem]:
        """
        Generate buy candidate list from symbol aggregated stats.
        """
```

---

## 8. Regime Layer

### 8.1 `news_regime.regime.classifier`

```python
from news_regime.domain.models import GlobalNewsStats, RegimeStatus
from news_regime.config.models import RegimeConfig

class RegimeClassifier:
    def __init__(self, config: RegimeConfig):
        ...

    def classify(self, stats: GlobalNewsStats, as_of: "datetime") -> RegimeStatus:
        """
        Decide regime label/confidence/explanation based on global stats.
        """
```

---

## 9. Storage Layer

(ORM/SQLAlchemy 같은 걸 쓴다고 가정한 **Repository 인터페이스**)

### 9.1 `news_regime.storage.db`

```python
class DbSessionManager:
    def __init__(self, config: DbConfig):
        ...

    def create_session(self):
        """
        Return DB session/connection object (ORM 세션 또는 raw connection).
        """
```

### 9.2 `news_regime.storage.repositories.news_repo`

```python
from news_regime.domain.models import NewsArticleDetail, NewsArticleMeta, ArticleSentiment

class NewsRepository:
    def exists_by_url(self, session, url: str) -> bool:
        """
        Check if a news article already exists by URL.
        """

    def insert_news_meta(self, session, meta: NewsArticleMeta) -> str:
        """
        Insert News meta into DB, return generated news_id.
        """

    def insert_news_detail(self, session, detail: NewsArticleDetail, news_id: str) -> None:
        """
        Insert/update full article body and related fields.
        """

    def get_news_in_window(
        self,
        session,
        start: "datetime",
        end: "datetime",
    ) -> list[NewsArticleMeta]:
        """
        Fetch news in given time window.
        """
```

### 9.3 `news_regime.storage.repositories.mapping_repo`

```python
from news_regime.domain.models import SymbolMatch

class SymbolMapRepository:
    def insert_mappings(self, session, mappings: list[SymbolMatch]) -> None:
        """
        Insert news-symbol mapping rows.
        """

    def get_mappings_for_news_ids(
        self,
        session,
        news_ids: list[str],
    ) -> list[SymbolMatch]:
        """
        Fetch mappings for given news ids.
        """
```

### 9.4 `news_regime.storage.repositories.analysis_repo`

```python
from news_regime.domain.models import ArticleSentiment

class AnalysisRepository:
    def insert_article_analysis(
        self,
        session,
        analyses: list[ArticleSentiment],
    ) -> None:
        """
        Bulk insert article-level analysis.
        """

    def get_article_analysis_in_window(
        self,
        session,
        start: "datetime",
        end: "datetime",
    ) -> list[ArticleSentiment]:
        """
        Fetch analysis results in a time window.
        """
```

### 9.5 `news_regime.storage.repositories.recommendation_repo`

```python
from news_regime.domain.models import RecommendationItem

class RecommendationRepository:
    def insert_recommendations(
        self,
        session,
        items: list[RecommendationItem],
    ) -> None:
        """
        Insert newly generated recommendations.
        """

    def get_latest_recommendations(self, session, limit: int = 50) -> list[RecommendationItem]:
        """
        Fetch latest recommendations.
        """
```

### 9.6 `news_regime.storage.repositories.regime_repo`

```python
from news_regime.domain.models import RegimeStatus

class RegimeRepository:
    def insert_regime(self, session, regime: RegimeStatus) -> None:
        """
        Insert new regime record.
        """

    def get_recent_regimes(
        self,
        session,
        limit: int = 50,
    ) -> list[RegimeStatus]:
        """
        Get recent regime history.
        """
```

---

## 10. Archive Layer

### 10.1 `news_regime.archive.planner`

```python
class ArchivePlanner:
    def __init__(self, config: ArchiveConfig):
        ...

    def compute_cutoff_datetime(self, now: "datetime") -> "datetime":
        """
        Compute the datetime before which records are considered archive targets.
        """
```

### 10.2 `news_regime.archive.exporter`

```python
class ArchiveExporter:
    def __init__(self, config: ArchiveConfig):
        ...

    def export_news(
        self,
        session,
        cutoff: "datetime",
    ) -> list[str]:
        """
        Export old news to archive files.
        Return list of generated file paths.
        """

    def export_analysis(
        self,
        session,
        cutoff: "datetime",
    ) -> list[str]:
        """
        Export analysis-related tables.
        """
```

### 10.3 `news_regime.archive.cleaner`

```python
class ArchiveCleaner:
    def delete_news_before(self, session, cutoff: "datetime") -> int:
        """
        Delete news before cutoff, return number of deleted rows.
        """

    def delete_analysis_before(self, session, cutoff: "datetime") -> int:
        """
        Delete analysis-related rows before cutoff.
        """
```

### 10.4 `news_regime.archive.job`

```python
class ArchiveJob:
    def __init__(
        self,
        session_manager: DbSessionManager,
        planner: ArchivePlanner,
        exporter: ArchiveExporter,
        cleaner: ArchiveCleaner,
    ):
        ...

    def run(self, now: "datetime") -> None:
        """
        Full archive cycle: plan → export → delete → logging.
        """
```

---

## 11. API / Integration Layer

### 11.1 `news_regime.api.service`

(REST로 가든, 내부 호출로 가든 **비즈니스 서비스 인터페이스**)

```python
from news_regime.domain.models import RecommendationItem, RegimeStatus

class QueryService:
    def __init__(self, session_manager: DbSessionManager):
        ...

    def get_latest_recommendations(self, limit: int = 20) -> list[RecommendationItem]:
        """
        Return latest recommendation items.
        """

    def get_current_regime(self) -> RegimeStatus | None:
        """
        Return most recent regime status.
        """

    def get_symbol_news_summary(
        self,
        symbol: str,
        window_hours: int = 24,
    ) -> dict:
        """
        Return news/sentiment summary for one symbol.
        """
```

(이걸 FastAPI 엔드포인트에서 감싸는 형태로 구현하면 됨)

---

## 12. CLI / Orchestrator Layer

### 12.1 `news_regime.cli.hourly_job`

```python
class HourlyJob:
    def __init__(
        self,
        config: AppConfig,
        session_manager: DbSessionManager,
        naver_client: NaverNewsClient,
        mainnews_parser: MainNewsParser,
        article_parser: ArticleParser,
        text_normalizer: TextNormalizer,
        symbol_dict: SymbolDictionary,
        symbol_matcher: SymbolMatcher,
        sentiment_analyzer: SentimentAnalyzer,
        risk_tagger: RiskTagger,
        impact_scorer: ImpactScorer,
        aggregation_service: AggregationService,
        recommendation_engine: RecommendationEngine,
        regime_classifier: RegimeClassifier,
    ):
        ...

    def run(self, now: "datetime") -> None:
        """
        Execute full hourly pipeline:
        crawl → parse → preprocess → mapping → analysis → recommend → regime → DB.
        """
```

### 12.2 `news_regime.cli.daily_archive_job`

```python
class DailyArchiveJob:
    def __init__(
        self,
        config: AppConfig,
        session_manager: DbSessionManager,
        archive_job: ArchiveJob,
    ):
        ...

    def run(self, now: "datetime") -> None:
        """
        Execute daily archive pipeline.
        """
```

### 12.3 진입 함수 (실제 스크립트에서 호출)

```python
def run_hourly_cycle() -> None:
    """
    Entry point for cron/systemd hourly job.
    """

def run_daily_archive() -> None:
    """
    Entry point for cron/systemd daily archive job.
    """
```

---

여기까지가 **“이 프로젝트에서 실제 Python 모듈/클래스/함수 시그니처를 어떻게 잡을지”에 대한 구체 설계**입니다.

다음 단계로는, 원하시면:

* 이 시그니처를 기반으로 **코드 스켈레톤(파일 구조 + class/def + pass)** 생성
* 또는 특정 부분 (예: `HourlyJob.run` 의 상세 시퀀스, `RecommendationPolicy` 룰 테이블) 을 더 구체화

중 하나를 골라서 이어갈 수 있어요.
