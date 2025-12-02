다음은 '코드 스켈레톤(파일 구조 + class/def + pass)' 과 '상세한 Comment' 를 넣어 작성해 주세요.



아래는 앞에서 설계한 구조를 그대로 반영한 **코드 스켈레톤**입니다.

* 실제 로직은 전혀 없고,
* **파일 구조 + class/def + pass**,
* 각 위치에서 어떤 일을 해야 하는지 **상세한 주석**만 넣어두었습니다.

필요하면 나중에 파일별로 쪼개서 붙여넣기만 하면 됩니다.

---

### 0. 디렉토리 구조 개요

```text
news_regime/
  __init__.py

  config/
    __init__.py
    models.py
    loader.py

  domain/
    __init__.py
    models.py

  crawler/
    __init__.py
    http_client.py
    naver_client.py

  parser/
    __init__.py
    mainnews_parser.py
    article_parser.py

  preprocess/
    __init__.py
    text_normalizer.py
    time_utils.py

  mapping/
    __init__.py
    symbol_dictionary.py
    matcher.py

  analysis/
    __init__.py
    sentiment.py
    risk.py
    impact.py
    aggregator.py

  recommend/
    __init__.py
    policy.py
    engine.py

  regime/
    __init__.py
    classifier.py

  storage/
    __init__.py
    db.py
    repositories/
      __init__.py
      news_repo.py
      mapping_repo.py
      analysis_repo.py
      recommendation_repo.py
      regime_repo.py

  archive/
    __init__.py
    planner.py
    exporter.py
    cleaner.py
    job.py

  api/
    __init__.py
    service.py

  cli/
    __init__.py
    hourly_job.py
    daily_archive_job.py
    main.py

  common/
    __init__.py
    logging.py
    time_utils.py
    exceptions.py
```

---

## 1. 최상위 패키지

```python
# news_regime/__init__.py
"""
news_regime 패키지의 루트 모듈.

- 네이버 금융 주요뉴스 크롤링
- 뉴스 기반 매수 종목 추천
- 뉴스 기반 시장 Regime 판단
- DB 저장 및 아카이빙

을 담당하는 하위 모듈들을 포함한다.
"""
```

---

## 2. config

```python
# news_regime/config/__init__.py
"""
설정 관련 모듈 패키지.

- models.py: AppConfig, DbConfig, CrawlConfig 등 설정 데이터 구조
- loader.py: YAML/ENV 로부터 설정을 로딩하는 기능
"""
```

```python
# news_regime/config/models.py
from __future__ import annotations
from dataclasses import dataclass


@dataclass
class DbConfig:
    """DB 연결 관련 설정."""
    dsn: str
    echo: bool = False
    pool_size: int = 5
    timeout: float = 30.0


@dataclass
class CrawlConfig:
    """크롤링 관련 설정 (네이버 URL, 타임아웃, 재시도 등)."""
    base_url: str
    mainnews_path: str = "/news/mainnews.naver"
    user_agent: str = "news_regime_crawler"
    request_timeout: float = 10.0
    max_retries: int = 3
    retry_backoff_sec: float = 1.0
    min_request_interval_sec: float = 0.5


@dataclass
class AnalysisConfig:
    """Sentiment/리스크/추천 정책 관련 설정."""
    sentiment_positive_threshold: float = 0.2
    sentiment_negative_threshold: float = -0.2
    min_articles_for_symbol: int = 1
    aggregation_window_hours: int = 24
    max_regulatory_risk_ratio_for_buy: float = 0.1
    min_impact_for_buy: float = 0.1


@dataclass
class RegimeConfig:
    """시장 Regime 판단 규칙 관련 설정."""
    regime_window_hours: int = 24
    bullish_sentiment_threshold: float = 0.2
    bearish_sentiment_threshold: float = -0.2
    high_risk_ratio_threshold: float = 0.3
    volatility_impact_threshold: float = 0.5


@dataclass
class ArchiveConfig:
    """아카이빙(3일 경과 데이터 파일 저장) 관련 설정."""
    retention_days: int = 3
    archive_root_dir: str = "./archive"
    file_format: str = "parquet"  # 또는 "jsonl"
    batch_size: int = 1000


@dataclass
class AppConfig:
    """애플리케이션 전체 설정을 묶는 상위 설정 객체."""
    db: DbConfig
    crawl: CrawlConfig
    analysis: AnalysisConfig
    regime: RegimeConfig
    archive: ArchiveConfig
    log_level: str = "INFO"
    timezone: str = "Asia/Seoul"
```

```python
# news_regime/config/loader.py
from __future__ import annotations
from typing import Optional
from .models import AppConfig, DbConfig, CrawlConfig, AnalysisConfig, RegimeConfig, ArchiveConfig


class ConfigLoader:
    """
    설정 로더.

    - 실제 구현에서는 YAML 파일 또는 환경변수를 읽어서 AppConfig 인스턴스를 만든다.
    - 여기서는 시그니처만 정의하고 내부는 pass 처리.
    """

    def __init__(self, env: Optional[str] = None) -> None:
        """
        :param env: "dev", "prod" 등 환경 이름. None 이면 기본값 사용.
        """
        self.env = env

    def load(self) -> AppConfig:
        """
        설정을 로드해서 AppConfig 를 반환한다.
        실제 구현에서는 YAML/ENV 파싱 로직이 들어간다.
        """
        # TODO: YAML/ENV 로딩 로직 구현
        raise NotImplementedError


def get_config(env: Optional[str] = None) -> AppConfig:
    """
    전역에서 편하게 호출할 수 있는 편의 함수.

    - 내부적으로 ConfigLoader 를 생성하고 load() 를 호출.
    - (실제 구현에서는 캐싱/싱글톤 처리 가능)
    """
    loader = ConfigLoader(env=env)
    return loader.load()
```

---

## 3. domain

```python
# news_regime/domain/__init__.py
"""
도메인 계층: 크롤링/분석/추천/Regime 판단에 공통으로 사용되는 데이터 모델 정의.
"""
```

```python
# news_regime/domain/models.py
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional, List, Dict


@dataclass
class NewsArticleMeta:
    """
    메인뉴스 목록에서 얻는 기사 메타 정보.
    DB 저장 전에는 id 가 None 일 수 있다.
    """
    id: Optional[str]
    title: str
    url: str
    publisher: Optional[str]
    category: Optional[str]
    summary: Optional[str]
    published_at: datetime
    crawled_at: datetime


@dataclass
class NewsArticleDetail:
    """
    상세 기사 내용(본문 텍스트 포함).
    """
    meta: NewsArticleMeta
    body_text: str
    reporter: Optional[str] = None
    tags: Optional[List[str]] = None


@dataclass
class SymbolMatch:
    """
    기사와 종목간 매핑 결과.
    """
    news_id: Optional[str]
    symbol: str
    match_score: float


@dataclass
class ArticleSentiment:
    """
    단일 기사에 대한 Sentiment / Risk / Impact 분석 결과.
    """
    news_id: str
    sentiment_score: float
    regulatory_risk_flag: bool
    earnings_risk_flag: bool
    credit_risk_flag: bool
    impact_score: float


@dataclass
class SymbolAggregatedStats:
    """
    지정된 시간 윈도우에서 한 종목에 대해 집계된 뉴스 통계.
    """
    symbol: str
    window_start: datetime
    window_end: datetime
    num_articles: int
    avg_sentiment: float
    max_sentiment: float
    avg_impact: float
    max_impact: float
    regulatory_risk_ratio: float
    earnings_risk_ratio: float
    credit_risk_ratio: float
    last_article_time: Optional[datetime]


@dataclass
class RecommendationItem:
    """
    매수 후보 종목 하나에 대한 추천 정보.
    """
    symbol: str
    created_at: datetime
    sentiment_score: float
    impact_score: float
    composite_score: float
    risk_summary: Dict[str, float | bool] = field(default_factory=dict)
    status: str = "ACTIVE"  # "ACTIVE", "WITHDRAWN" 등
    reason_summary: str = ""


@dataclass
class RegimeStatus:
    """
    특정 시점의 시장 Regime 판단 결과.
    """
    timestamp: datetime
    regime_label: str
    confidence: float
    explanation: str


@dataclass
class GlobalNewsStats:
    """
    Regime 판단을 위한 전체 뉴스 기준 집계 통계.
    """
    window_start: datetime
    window_end: datetime
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

## 4. common

```python
# news_regime/common/__init__.py
"""
공통 유틸리티 모듈.
"""
```

```python
# news_regime/common/exceptions.py
"""
커스텀 예외 타입 정의.
"""


class NewsRegimeError(Exception):
    """기본 예외 베이스 클래스."""


class CrawlError(NewsRegimeError):
    """크롤링 단계에서 발생하는 예외."""


class ParseError(NewsRegimeError):
    """HTML 파싱 단계에서 발생하는 예외."""


class AnalysisError(NewsRegimeError):
    """분석 단계에서 발생하는 예외."""
```

```python
# news_regime/common/logging.py
import logging


def setup_logging(level: str = "INFO") -> None:
    """
    애플리케이션 전역 로깅 설정.

    실제 구현에서는:
    - 로그 포맷 설정
    - 파일 핸들러 추가
    - 모듈별 logger 설정
    등이 들어간다.
    """
    # TODO: logging.basicConfig(...) 등 실제 설정 구현
    pass
```

```python
# news_regime/common/time_utils.py
from __future__ import annotations
from datetime import datetime, timezone
from zoneinfo import ZoneInfo


def now_in_timezone(tz_name: str) -> datetime:
    """
    지정된 타임존 기준 현재 시각을 반환.

    - Asia/Seoul 등 tz_name 을 받아 zoneinfo 사용.
    """
    # TODO: 실제 구현에서는 ZoneInfo(tz_name) 사용
    return datetime.now(timezone.utc)
```

---

## 5. crawler

```python
# news_regime/crawler/__init__.py
"""
크롤러 계층.

- http_client: HTTP 요청 + 재시도 + rate limit
- naver_client: 네이버 금융 메인뉴스/기사 HTML 가져오기
"""
```

```python
# news_regime/crawler/http_client.py
from __future__ import annotations
from dataclasses import dataclass
from typing import Optional, Dict


@dataclass
class HttpResponse:
    """HTTP 응답 래퍼."""
    url: str
    status_code: int
    headers: Dict[str, str]
    text: str
    elapsed_sec: float


class HttpClient:
    """
    기본 HTTP 클라이언트 추상화.

    - base_url 기준으로 path 를 받아 GET 요청 수행
    - 재시도/백오프/타임아웃/헤더 설정 등은 실제 구현에서 처리
    """

    def __init__(self, base_url: str, timeout: float, user_agent: str) -> None:
        self.base_url = base_url
        self.timeout = timeout
        self.user_agent = user_agent

    def get(self, path: str, params: Optional[Dict[str, str]] = None) -> HttpResponse:
        """
        base_url + path 로 GET 요청을 보내고 HttpResponse 반환.

        실제 구현에서는:
        - requests 또는 httpx 사용
        - 에러 처리/재시도 로직 포함
        """
        # TODO: HTTP GET 구현
        raise NotImplementedError
```

```python
# news_regime/crawler/naver_client.py
from __future__ import annotations
from typing import List
from datetime import datetime

from news_regime.config.models import CrawlConfig
from news_regime.domain.models import NewsArticleMeta, NewsArticleDetail
from news_regime.crawler.http_client import HttpClient
from news_regime.parser.mainnews_parser import MainNewsParser
from news_regime.parser.article_parser import ArticleParser


class NaverNewsClient:
    """
    네이버 금융 메인뉴스/상세 기사 HTML을 가져오는 클라이언트.

    - HTTP 요청은 HttpClient 사용
    - HTML 파싱은 Parser 계층에 위임
    """

    def __init__(
        self,
        http_client: HttpClient,
        crawl_config: CrawlConfig,
        mainnews_parser: MainNewsParser,
        article_parser: ArticleParser,
    ) -> None:
        self.http_client = http_client
        self.crawl_config = crawl_config
        self.mainnews_parser = mainnews_parser
        self.article_parser = article_parser

    def fetch_mainnews_page(self) -> str:
        """
        메인뉴스 페이지 HTML 문자열을 받아온다.
        """
        # TODO: http_client.get(...) 호출 및 에러 처리
        raise NotImplementedError

    def fetch_article_page(self, article_url: str) -> str:
        """
        개별 기사 페이지 HTML 문자열을 받아온다.
        """
        # TODO: http_client.get(...) 호출 및 에러 처리
        raise NotImplementedError

    def crawl_mainnews(self, crawled_at: datetime) -> List[NewsArticleMeta]:
        """
        메인뉴스 페이지를 크롤링하고 NewsArticleMeta 리스트를 생성한다.
        """
        # TODO: fetch_mainnews_page -> mainnews_parser.parse
        raise NotImplementedError

    def crawl_article_detail(self, meta: NewsArticleMeta) -> NewsArticleDetail:
        """
        개별 기사의 상세 내용을 크롤링하여 NewsArticleDetail 생성.
        """
        # TODO: fetch_article_page -> article_parser.parse
        raise NotImplementedError
```

---

## 6. parser

```python
# news_regime/parser/__init__.py
"""
HTML 파싱 계층.

- mainnews_parser: 메인뉴스 목록 파싱
- article_parser: 개별 기사 상세 파싱
"""
```

```python
# news_regime/parser/mainnews_parser.py
from __future__ import annotations
from typing import List
from datetime import datetime
from news_regime.domain.models import NewsArticleMeta


class MainNewsParser:
    """
    메인뉴스 페이지 HTML을 파싱하여 NewsArticleMeta 리스트를 만드는 클래스.
    """

    def parse(self, html: str, crawled_at: datetime) -> List[NewsArticleMeta]:
        """
        :param html: 메인뉴스 HTML
        :param crawled_at: 실제 크롤링 시각
        :return: 파싱된 기사 메타 정보 리스트
        """
        # TODO: BeautifulSoup 등으로 HTML 파싱 구현
        raise NotImplementedError
```

```python
# news_regime/parser/article_parser.py
from __future__ import annotations
from news_regime.domain.models import NewsArticleMeta, NewsArticleDetail


class ArticleParser:
    """
    개별 기사 페이지 HTML을 파싱하여 NewsArticleDetail 을 만드는 클래스.
    """

    def parse(self, html: str, meta: NewsArticleMeta) -> NewsArticleDetail:
        """
        :param html: 기사 상세 HTML
        :param meta: 메인뉴스 목록에서 얻은 메타 정보
        :return: 본문 텍스트 및 기자명/태그가 포함된 NewsArticleDetail
        """
        # TODO: HTML 파싱, 본문 추출, 기자명/태그 추출 구현
        raise NotImplementedError
```

---

## 7. preprocess

```python
# news_regime/preprocess/__init__.py
"""
전처리 계층.

- text_normalizer: HTML/텍스트 정규화
- time_utils: 네이버 시간 문자열 파싱
"""
```

```python
# news_regime/preprocess/text_normalizer.py
from __future__ import annotations


class TextNormalizer:
    """
    기사 텍스트 전처리를 담당하는 클래스.

    - HTML 태그 제거
    - 공백/줄바꿈 정규화
    """

    def strip_html(self, html: str) -> str:
        """HTML 태그를 제거하고 텍스트만 남긴다."""
        # TODO: HTML 파서/정규표현식 사용
        raise NotImplementedError

    def normalize_whitespace(self, text: str) -> str:
        """공백, 줄바꿈 등을 정규화한다."""
        # TODO: 여러 공백을 하나로 합치고, 불필요한 줄바꿈 제거
        raise NotImplementedError

    def normalize_text(self, html_or_text: str) -> str:
        """
        전체 전처리 파이프라인.
        - HTML 입력일 수도 있고, 이미 텍스트일 수도 있음.
        """
        # TODO: strip_html + normalize_whitespace 를 적절히 조합
        raise NotImplementedError
```

```python
# news_regime/preprocess/time_utils.py
from __future__ import annotations
from datetime import datetime
from zoneinfo import ZoneInfo


class TimeParser:
    """
    네이버 표기 시간 문자열을 datetime(Asia/Seoul 등)으로 변환하는 유틸.
    """

    def __init__(self, timezone: str) -> None:
        self.timezone = timezone

    def parse_published_time(self, raw_time_str: str) -> datetime:
        """
        :param raw_time_str: 네이버에 표기된 시간 문자열 (예: '2025.12.02 10:30')
        :return: 지정된 타임존의 datetime
        """
        # TODO: datetime.strptime + ZoneInfo 사용
        raise NotImplementedError
```

---

## 8. mapping

```python
# news_regime/mapping/__init__.py
"""
기사 텍스트 → 종목(Symbol) 매핑 계층.
"""
```

```python
# news_regime/mapping/symbol_dictionary.py
from __future__ import annotations
from dataclasses import dataclass
from typing import List, Optional


@dataclass
class SymbolInfo:
    """
    종목 정보 데이터 구조.
    """
    symbol: str
    name_kr: str
    name_en: Optional[str] = None
    aliases: List[str] = None  # 별칭/약어


class SymbolDictionary:
    """
    종목 리스트를 관리하는 사전 클래스.

    - 파일/DB/외부 API 에서 종목 리스트를 로드할 수 있도록 설계
    """

    def __init__(self) -> None:
        self._symbols: List[SymbolInfo] = []

    def load(self) -> None:
        """
        종목 리스트를 로드하여 self._symbols 에 채운다.
        실제 구현에서는 KRX 파일/DB 등에서 데이터를 가져온다.
        """
        # TODO: 실제 로딩 로직 구현
        raise NotImplementedError

    def get_all(self) -> List[SymbolInfo]:
        """전체 종목 리스트를 반환."""
        return self._symbols

    def find_by_symbol(self, symbol: str) -> Optional[SymbolInfo]:
        """심볼 문자열로 SymbolInfo 검색."""
        # TODO: 간단한 검색 구현
        raise NotImplementedError
```

```python
# news_regime/mapping/matcher.py
from __future__ import annotations
from typing import List
from news_regime.domain.models import NewsArticleDetail, SymbolMatch
from .symbol_dictionary import SymbolDictionary, SymbolInfo


class SymbolMatcher:
    """
    기사 텍스트에서 종목명을 찾아내는 매칭 엔진.

    - 심볼/한글명/별칭/약어 등을 고려해서 match_score 계산
    """

    def __init__(self, symbol_dict: SymbolDictionary) -> None:
        self.symbol_dict = symbol_dict

    def match_symbols(self, article: NewsArticleDetail) -> List[SymbolMatch]:
        """
        :param article: 기사 상세 정보
        :return: 매칭된 종목 리스트 (심볼, 점수)
        """
        # TODO: 문자열 검색/토큰화/스코어링 로직 구현
        raise NotImplementedError
```

---

## 9. analysis

```python
# news_regime/analysis/__init__.py
"""
뉴스 분석 계층.

- sentiment: 감성 분석
- risk: 리스크 키워드 태깅
- impact: 영향도 점수 계산
- aggregator: 기사/종목/글로벌 수준 통계 집계
"""
```

```python
# news_regime/analysis/sentiment.py
from __future__ import annotations
from news_regime.domain.models import NewsArticleDetail


class SentimentAnalyzer:
    """
    기사 텍스트에 대한 Sentiment Score 를 계산하는 클래스.

    v1: 룰/사전 기반
    v2: ML/LLM 으로 교체 가능
    """

    def analyze(self, article: NewsArticleDetail) -> float:
        """
        :return: -1.0 ~ 1.0 범위의 점수
        """
        # TODO: 키워드/사전 기반 감성 분석 구현
        raise NotImplementedError
```

```python
# news_regime/analysis/risk.py
from __future__ import annotations
from dataclasses import dataclass
from news_regime.domain.models import NewsArticleDetail


@dataclass
class RiskFlags:
    """기사에 대한 리스크 플래그."""
    regulatory_risk: bool
    earnings_risk: bool
    credit_risk: bool


class RiskTagger:
    """
    기사 텍스트에서 리스크 관련 키워드를 탐지하여 플래그를 설정하는 클래스.
    """

    def tag(self, article: NewsArticleDetail) -> RiskFlags:
        """
        :return: 리스크 플래그 구조체
        """
        # TODO: 규제/실적/신용 관련 키워드/패턴 매칭 구현
        raise NotImplementedError
```

```python
# news_regime/analysis/impact.py
from __future__ import annotations
from news_regime.domain.models import NewsArticleDetail


class ImpactScorer:
    """
    뉴스의 영향도(Impact Score)를 계산하는 클래스.

    - 언론사, 제목의 강도, 범위(전시장/섹터/종목) 등을 고려
    """

    def score(self, article: NewsArticleDetail) -> float:
        """
        :return: 0~1 또는 0~100 스케일의 점수
        """
        # TODO: 임의 규칙/가중치를 기반으로 점수 계산
        raise NotImplementedError
```

```python
# news_regime/analysis/aggregator.py
from __future__ import annotations
from typing import List
from datetime import datetime

from news_regime.domain.models import (
    ArticleSentiment,
    SymbolAggregatedStats,
    GlobalNewsStats,
    SymbolMatch,
)


class AggregationService:
    """
    기사 레벨 분석 결과를 종목/전체 수준으로 집계하는 서비스.

    - aggregate_by_symbol: 종목별 집계
    - aggregate_global: 전체 뉴스 기준 집계
    """

    def aggregate_by_symbol(
        self,
        sentiments: List[ArticleSentiment],
        symbol_map: List[SymbolMatch],
        window_start: datetime,
        window_end: datetime,
    ) -> List[SymbolAggregatedStats]:
        """
        기사 단위 분석 결과와 news-symbol 매핑을 이용해 종목별 통계를 계산.

        - 각 symbol 에 대해:
          - 기사 수
          - avg/max sentiment, impact
          - 리스크 플래그 비율
          - 마지막 기사 시각
        """
        # TODO: 종목별 그룹핑 및 통계 계산 구현
        raise NotImplementedError

    def aggregate_global(
        self,
        sentiments: List[ArticleSentiment],
        window_start: datetime,
        window_end: datetime,
    ) -> GlobalNewsStats:
        """
        전체 뉴스 기준으로 Regime 판단에 필요한 통계를 계산.

        - 전체 기사 수
        - 평균 감성 점수
        - 긍정/부정 비율
        - 리스크 플래그 비율
        - macro/sector/single_stock 비율(카테고리 기반)
        """
        # TODO: 전체 집계 통계 계산 구현
        raise NotImplementedError
```

---

## 10. recommend

```python
# news_regime/recommend/__init__.py
"""
매수 종목 추천 계층.

- policy: 룰 기반 추천 정책
- engine: AggregatedStats → RecommendationItem 생성
"""
```

```python
# news_regime/recommend/policy.py
from __future__ import annotations
from news_regime.config.models import AnalysisConfig
from news_regime.domain.models import SymbolAggregatedStats


class RecommendationPolicy:
    """
    룰 기반 매수 후보 판단 정책.

    - Sentiment threshold
    - Impact 최소값
    - 리스크 플래그 허용 범위 등
    """

    def __init__(self, config: AnalysisConfig) -> None:
        self.config = config

    def is_buy_candidate(self, stats: SymbolAggregatedStats) -> bool:
        """
        종목이 매수 후보인지 여부를 리턴.
        """
        # TODO: config 기반 조건식 구현
        raise NotImplementedError

    def compute_composite_score(self, stats: SymbolAggregatedStats) -> float:
        """
        종합 점수(순위용)를 계산.
        """
        # TODO: sentiment/impact/뉴스 수 등을 조합한 점수 계산
        raise NotImplementedError
```

```python
# news_regime/recommend/engine.py
from __future__ import annotations
from typing import List
from datetime import datetime

from news_regime.domain.models import SymbolAggregatedStats, RecommendationItem
from .policy import RecommendationPolicy


class RecommendationEngine:
    """
    AggregatedStats를 입력으로 매수 후보 RecommendationItem 리스트를 생성하는 엔진.
    """

    def __init__(self, policy: RecommendationPolicy) -> None:
        self.policy = policy

    def generate_recommendations(
        self,
        aggregated_stats: List[SymbolAggregatedStats],
        as_of: datetime,
        top_n: int | None = None,
    ) -> List[RecommendationItem]:
        """
        - aggregated_stats 를 순회하며 정책을 적용해 후보 종목을 선별하고,
        - composite_score 기준 정렬 후 상위 top_n 개를 반환.
        """
        # TODO: is_buy_candidate + compute_composite_score 사용해서 리스트 생성
        raise NotImplementedError
```

---

## 11. regime

```python
# news_regime/regime/__init__.py
"""
시장 Regime 판단 계층.
"""
```

```python
# news_regime/regime/classifier.py
from __future__ import annotations
from datetime import datetime
from news_regime.config.models import RegimeConfig
from news_regime.domain.models import GlobalNewsStats, RegimeStatus


class RegimeClassifier:
    """
    전체 뉴스 통계(GlobalNewsStats) 기반으로
    시장 Regime(Bullish/Bearish/Risk-off 등)을 결정하는 클래스.
    """

    def __init__(self, config: RegimeConfig) -> None:
        self.config = config

    def classify(self, stats: GlobalNewsStats, as_of: datetime) -> RegimeStatus:
        """
        룰 기반으로 Regime 라벨/신뢰도/설명을 채워 RegimeStatus 를 반환.
        """
        # TODO: RegimeConfig 기반 조건식 구현
        raise NotImplementedError
```

---

## 12. storage

```python
# news_regime/storage/__init__.py
"""
DB 및 저장소 계층.
"""
```

```python
# news_regime/storage/db.py
from __future__ import annotations
from typing import Any
from news_regime.config.models import DbConfig


class DbSessionManager:
    """
    DB 세션/커넥션을 관리하는 클래스.

    - SQLAlchemy 또는 sqlite3/raw 커넥션을 감싸는 역할
    """

    def __init__(self, config: DbConfig) -> None:
        self.config = config
        # TODO: 실제 엔진/커넥션 초기화

    def create_session(self) -> Any:
        """
        DB 세션 또는 커넥션 객체를 생성/반환.
        """
        # TODO: ORM 또는 raw connection 반환
        raise NotImplementedError
```

```python
# news_regime/storage/repositories/__init__.py
"""
Repository 패키지.

엔티티 단위로 DB 접근을 캡슐화.
"""
```

```python
# news_regime/storage/repositories/news_repo.py
from __future__ import annotations
from typing import List
from datetime import datetime
from news_regime.domain.models import NewsArticleMeta, NewsArticleDetail, ArticleSentiment


class NewsRepository:
    """
    뉴스 메타/본문을 관리하는 Repository.

    - exists_by_url
    - insert_news_meta
    - insert_news_detail
    - get_news_in_window
    """

    def exists_by_url(self, session, url: str) -> bool:
        # TODO: SELECT COUNT(*) ... 구현
        raise NotImplementedError

    def insert_news_meta(self, session, meta: NewsArticleMeta) -> str:
        # TODO: INSERT 후 생성된 news_id 반환
        raise NotImplementedError

    def insert_news_detail(self, session, detail: NewsArticleDetail, news_id: str) -> None:
        # TODO: 본문/기자명/태그 등을 저장
        raise NotImplementedError

    def get_news_in_window(
        self,
        session,
        start: datetime,
        end: datetime,
    ) -> List[NewsArticleMeta]:
        # TODO: published_at BETWEEN start AND end
        raise NotImplementedError
```

```python
# news_regime/storage/repositories/mapping_repo.py
from __future__ import annotations
from typing import List
from news_regime.domain.models import SymbolMatch


class SymbolMapRepository:
    """
    news-symbol 매핑 테이블 접근을 담당하는 Repository.
    """

    def insert_mappings(self, session, mappings: List[SymbolMatch]) -> None:
        # TODO: bulk insert 구현
        raise NotImplementedError

    def get_mappings_for_news_ids(
        self,
        session,
        news_ids: List[str],
    ) -> List[SymbolMatch]:
        # TODO: SELECT ... WHERE news_id IN (...)
        raise NotImplementedError
```

```python
# news_regime/storage/repositories/analysis_repo.py
from __future__ import annotations
from typing import List
from datetime import datetime
from news_regime.domain.models import ArticleSentiment


class AnalysisRepository:
    """
    기사 단위 분석 결과(ArticleSentiment)를 관리하는 Repository.
    """

    def insert_article_analysis(
        self,
        session,
        analyses: List[ArticleSentiment],
    ) -> None:
        # TODO: bulk insert 구현
        raise NotImplementedError

    def get_article_analysis_in_window(
        self,
        session,
        start: datetime,
        end: datetime,
    ) -> List[ArticleSentiment]:
        # TODO: published_at / analysis timestamp 기준 조회
        raise NotImplementedError
```

```python
# news_regime/storage/repositories/recommendation_repo.py
from __future__ import annotations
from typing import List
from news_regime.domain.models import RecommendationItem


class RecommendationRepository:
    """
    매수 추천 결과를 관리하는 Repository.
    """

    def insert_recommendations(
        self,
        session,
        items: List[RecommendationItem],
    ) -> None:
        # TODO: bulk insert 구현
        raise NotImplementedError

    def get_latest_recommendations(self, session, limit: int = 50) -> List[RecommendationItem]:
        # TODO: created_at DESC LIMIT N
        raise NotImplementedError
```

```python
# news_regime/storage/repositories/regime_repo.py
from __future__ import annotations
from typing import List
from news_regime.domain.models import RegimeStatus


class RegimeRepository:
    """
    Regime 히스토리를 관리하는 Repository.
    """

    def insert_regime(self, session, regime: RegimeStatus) -> None:
        # TODO: INSERT 구현
        raise NotImplementedError

    def get_recent_regimes(
        self,
        session,
        limit: int = 50,
    ) -> List[RegimeStatus]:
        # TODO: ORDER BY timestamp DESC LIMIT N
        raise NotImplementedError
```

---

## 13. archive

```python
# news_regime/archive/__init__.py
"""
3일 경과 데이터 아카이빙 계층.
"""
```

```python
# news_regime/archive/planner.py
from __future__ import annotations
from datetime import datetime, timedelta
from news_regime.config.models import ArchiveConfig


class ArchivePlanner:
    """
    아카이빙 대상 기준 시점을 계산하는 Planner.
    """

    def __init__(self, config: ArchiveConfig) -> None:
        self.config = config

    def compute_cutoff_datetime(self, now: datetime) -> datetime:
        """
        retention_days 기준으로 cutoff datetime 을 계산.
        """
        # TODO: now - retention_days
        raise NotImplementedError
```

```python
# news_regime/archive/exporter.py
from __future__ import annotations
from typing import List
from datetime import datetime
from news_regime.config.models import ArchiveConfig


class ArchiveExporter:
    """
    DB 레코드를 파일(JSONL/Parquet 등)로 내보내는 Exporter.
    """

    def __init__(self, config: ArchiveConfig) -> None:
        self.config = config

    def export_news(self, session, cutoff: datetime) -> List[str]:
        """
        cutoff 이전 뉴스 데이터를 파일로 내보내고,
        생성된 파일 경로 리스트를 반환.
        """
        # TODO: SELECT → 파일쓰기
        raise NotImplementedError

    def export_analysis(self, session, cutoff: datetime) -> List[str]:
        """
        분석/추천/Regime 등 관련 테이블을 파일로 내보내기.
        """
        # TODO: SELECT → 파일쓰기
        raise NotImplementedError
```

```python
# news_regime/archive/cleaner.py
from __future__ import annotations
from datetime import datetime


class ArchiveCleaner:
    """
    아카이빙 이후 DB에서 오래된 데이터를 삭제하는 모듈.
    """

    def delete_news_before(self, session, cutoff: datetime) -> int:
        """
        cutoff 이전 뉴스 레코드를 삭제하고, 삭제 건수를 반환.
        """
        # TODO: DELETE FROM news WHERE published_at < cutoff
        raise NotImplementedError

    def delete_analysis_before(self, session, cutoff: datetime) -> int:
        """
        cutoff 이전 분석/추천/Regime 레코드를 삭제.
        """
        # TODO: 관련 테이블별 DELETE
        raise NotImplementedError
```

```python
# news_regime/archive/job.py
from __future__ import annotations
from datetime import datetime
from news_regime.storage.db import DbSessionManager
from .planner import ArchivePlanner
from .exporter import ArchiveExporter
from .cleaner import ArchiveCleaner


class ArchiveJob:
    """
    아카이빙 전체 플로우를 실행하는 상위 Job.

    - cutoff 계산
    - export_news / export_analysis 실행
    - delete_news_before / delete_analysis_before 실행
    """

    def __init__(
        self,
        session_manager: DbSessionManager,
        planner: ArchivePlanner,
        exporter: ArchiveExporter,
        cleaner: ArchiveCleaner,
    ) -> None:
        self.session_manager = session_manager
        self.planner = planner
        self.exporter = exporter
        self.cleaner = cleaner

    def run(self, now: datetime) -> None:
        """
        아카이빙 전체 사이클 실행.
        """
        # TODO: 세션 열기 → cutoff 계산 → export → delete → 커밋
        raise NotImplementedError
```

---

## 14. api

```python
# news_regime/api/__init__.py
"""
외부/내부 서비스에서 사용할 조회용 API 계층.

- REST 서버(FastAPI 등)에 붙이거나,
- 내부 모듈에서 직접 호출할 수 있는 서비스 인터페이스 제공.
"""
```

```python
# news_regime/api/service.py
from __future__ import annotations
from typing import List, Dict
from news_regime.storage.db import DbSessionManager
from news_regime.domain.models import RecommendationItem, RegimeStatus


class QueryService:
    """
    매수 추천, Regime, 종목별 요약 조회를 담당하는 서비스.

    - FastAPI 등에서 이 클래스의 메서드를 호출하도록 설계.
    """

    def __init__(self, session_manager: DbSessionManager) -> None:
        self.session_manager = session_manager

    def get_latest_recommendations(self, limit: int = 20) -> List[RecommendationItem]:
        """
        가장 최근 추천 결과를 반환.
        """
        # TODO: RecommendationRepository 이용
        raise NotImplementedError

    def get_current_regime(self) -> RegimeStatus | None:
        """
        가장 최근 Regime 상태를 반환.
        """
        # TODO: RegimeRepository 이용
        raise NotImplementedError

    def get_symbol_news_summary(
        self,
        symbol: str,
        window_hours: int = 24,
    ) -> Dict:
        """
        특정 종목에 대한 최근 뉴스/감성 요약을 반환.

        반환 구조 예시:
        {
          "symbol": "...",
          "window_hours": 24,
          "num_articles": 10,
          "avg_sentiment": 0.3,
          "latest_news": [...],
        }
        """
        # TODO: NewsRepository, AnalysisRepository 조합해 요약 생성
        raise NotImplementedError
```

---

## 15. cli (엔트리 포인트 / 오케스트레이션)

```python
# news_regime/cli/__init__.py
"""
cron/systemd 에서 호출할 엔트리 포인트 모음.
"""
```

```python
# news_regime/cli/hourly_job.py
from __future__ import annotations
from datetime import datetime

from news_regime.config.loader import get_config
from news_regime.common.logging import setup_logging
from news_regime.storage.db import DbSessionManager
from news_regime.crawler.http_client import HttpClient
from news_regime.crawler.naver_client import NaverNewsClient
from news_regime.parser.mainnews_parser import MainNewsParser
from news_regime.parser.article_parser import ArticleParser
from news_regime.preprocess.text_normalizer import TextNormalizer
from news_regime.preprocess.time_utils import TimeParser
from news_regime.mapping.symbol_dictionary import SymbolDictionary
from news_regime.mapping.matcher import SymbolMatcher
from news_regime.analysis.sentiment import SentimentAnalyzer
from news_regime.analysis.risk import RiskTagger
from news_regime.analysis.impact import ImpactScorer
from news_regime.analysis.aggregator import AggregationService
from news_regime.recommend.policy import RecommendationPolicy
from news_regime.recommend.engine import RecommendationEngine
from news_regime.regime.classifier import RegimeClassifier


class HourlyJob:
    """
    1시간 주기 파이프라인 전체를 실행하는 Job 클래스.

    - 설정 로드
    - 크롤링
    - 전처리/매핑
    - 분석/집계
    - 추천/Regime 판단
    - DB 저장
    """

    def __init__(self) -> None:
        """
        의존성들을 초기화하는 부분.

        실제 구현에서는 DI 컨테이너를 쓰거나 별도 factory 생성 가능.
        """
        self.config = get_config()
        setup_logging(self.config.log_level)

        # DB / 세션
        self.session_manager = DbSessionManager(self.config.db)

        # 크롤러/파서
        http_client = HttpClient(
            base_url=self.config.crawl.base_url,
            timeout=self.config.crawl.request_timeout,
            user_agent=self.config.crawl.user_agent,
        )
        mainnews_parser = MainNewsParser()
        article_parser = ArticleParser()
        self.naver_client = NaverNewsClient(
            http_client=http_client,
            crawl_config=self.config.crawl,
            mainnews_parser=mainnews_parser,
            article_parser=article_parser,
        )

        # 전처리/매핑
        self.text_normalizer = TextNormalizer()
        self.time_parser = TimeParser(self.config.timezone)
        self.symbol_dict = SymbolDictionary()
        self.symbol_matcher = SymbolMatcher(self.symbol_dict)

        # 분석
        self.sentiment_analyzer = SentimentAnalyzer()
        self.risk_tagger = RiskTagger()
        self.impact_scorer = ImpactScorer()
        self.aggregation_service = AggregationService()

        # 추천/Regime
        self.reco_policy = RecommendationPolicy(self.config.analysis)
        self.reco_engine = RecommendationEngine(self.reco_policy)
        self.regime_classifier = RegimeClassifier(self.config.regime)

        # TODO: Repository, 기타 의존성 초기화 (NewsRepository 등)

    def run(self, now: datetime) -> None:
        """
        전체 hourly cycle 을 수행.

        - 1) 메인뉴스 크롤링 및 신규 기사 필터링
        - 2) 상세 기사 크롤링 + 전처리
        - 3) 종목 매핑
        - 4) Sentiment / Risk / Impact 분석
        - 5) 종목/전체 집계
        - 6) 추천 생성 + Regime 판단
        - 7) DB 저장
        """
        # TODO: 위 단계들을 순서대로 호출하는 오케스트레이션 구현
        raise NotImplementedError
```

```python
# news_regime/cli/daily_archive_job.py
from __future__ import annotations
from datetime import datetime

from news_regime.config.loader import get_config
from news_regime.common.logging import setup_logging
from news_regime.storage.db import DbSessionManager
from news_regime.archive.planner import ArchivePlanner
from news_regime.archive.exporter import ArchiveExporter
from news_regime.archive.cleaner import ArchiveCleaner
from news_regime.archive.job import ArchiveJob


class DailyArchiveJob:
    """
    매일 1회 실행되는 아카이빙 Job.
    """

    def __init__(self) -> None:
        self.config = get_config()
        setup_logging(self.config.log_level)
        self.session_manager = DbSessionManager(self.config.db)

        planner = ArchivePlanner(self.config.archive)
        exporter = ArchiveExporter(self.config.archive)
        cleaner = ArchiveCleaner()
        self.archive_job = ArchiveJob(
            session_manager=self.session_manager,
            planner=planner,
            exporter=exporter,
            cleaner=cleaner,
        )

    def run(self, now: datetime) -> None:
        """
        아카이빙 전체 사이클 실행.
        """
        # TODO: ArchiveJob.run 호출, 예외 처리/로깅
        raise NotImplementedError
```

```python
# news_regime/cli/main.py
from __future__ import annotations
from datetime import datetime
import argparse

from .hourly_job import HourlyJob
from .daily_archive_job import DailyArchiveJob


def main() -> None:
    """
    CLI 엔트리 포인트.

    예:
      python -m news_regime.cli.main --job hourly
      python -m news_regime.cli.main --job archive
    """
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--job",
        choices=["hourly", "archive"],
        required=True,
        help="실행할 job 타입 선택",
    )
    args = parser.parse_args()

    now = datetime.now()

    if args.job == "hourly":
        job = HourlyJob()
        job.run(now)
    elif args.job == "archive":
        job = DailyArchiveJob()
        job.run(now)


if __name__ == "__main__":
    main()
```

---

여기까지가 **전체 시스템의 코드 스켈레톤 + 상세 주석**입니다.

다음 단계에서 원하시면,

* 특정 부분부터 실제 구현(예: 크롤러 & 파서 / Sentiment 룰 / Regime 룰),
* 또는 테스트 코드 스켈레톤(pytest 기반) 설계,

중 어디부터 살을 붙일지 골라서 하나씩 같이 구현해 나가면 됩니다.
