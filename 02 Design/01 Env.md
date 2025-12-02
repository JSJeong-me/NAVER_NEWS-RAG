운영환경은 nvidia jetson agx orin 이고 linux 2x.04, Python 3.1x 환경에서 개발과 운영합니다. 위에 confirmed req. spec 을 python 으로 개발하기 위한 '설계'를 진행하세요. 이때 설계에만 집중하고 code 는 작성하지 마세요.



바로 설계로 가보죠.
지금까지 확정한 **Req Analysis v1.0**을 기반으로, Jetson AGX Orin + Linux 2x.04 + Python 3.1x 환경에서 구현하기 위한 **Python 설계안**만 정리하겠습니다. (※ 함수 정의/코드 구현 X, 설계와 구조만)

---

## 1. 전체 아키텍처 개요

### 1.1 런타임 구조 (v1 기준)

* **형태**: 단일 Python 애플리케이션 + 2개의 배치 엔트리

  * `Hourly Job` : “크롤링 → 분석 → 추천/Regime → DB 저장”
  * `Daily Job` : “3일 지난 데이터 아카이빙 → DB 삭제”
* **스케줄링**

  * Jetson 서버의 **cron 또는 systemd timer**로 2개의 엔트리를 스케줄링

    * 예) 매시 05분: hourly job
    * 매일 02:00: daily archive job
* **확장 방향**

  * 내부 모듈 경계를 명확히 해서 필요 시:

    * Crawler/Analyzer/Recommendation을 별도 마이크로서비스로 분리 가능하게 설계

### 1.2 논리 컴포넌트

1. **Config & Env Layer**
2. **Crawler Layer** (네이버 메인뉴스 페이지 + 상세 페이지 파서)
3. **Preprocessor & Mapper Layer** (텍스트 정제, 종목 매핑)
4. **Analysis Layer** (Sentiment / Risk / Impact / Aggregation)
5. **Recommendation Layer** (매수 종목 후보 선정)
6. **Regime Layer** (시장 Regime 판단)
7. **Persistence Layer** (DB 및 아카이빙 파일 관리)
8. **API / Integration Layer** (다른 Agent/서비스와 연동을 위한 인터페이스)
9. **Ops Layer** (로깅, 헬스체크, 알림)

---

## 2. 처리 플로우 설계

### 2.1 Hourly Pipeline (핵심 플로우)

1. **환경 초기화**

   * 설정 로드 (DB 연결 정보, 크롤링 주기, threshold 등)
   * 로거 초기화

2. **크롤링 단계**

   * 메인 뉴스 페이지 Fetch
   * 목록 파싱 → 기사 메타데이터 리스트 생성
   * 이미 수집된 기사인지 DB 조회를 통해 중복 필터링
   * 새 기사에 대해서만 상세 페이지 Fetch & 파싱

3. **전처리 & 종목 매핑**

   * 제목/본문 텍스트 정제
   * 종목 사전 로드
   * 각 기사 텍스트에서 종목 키워드 매칭 → news-symbol 매핑 결과 생성

4. **분석 단계**

   * 기사별 Sentiment 계산
   * Risk 플래그 태깅
   * Impact 점수 산출
   * 종목별로 최근 N시간 기준 Sentiment/Impact 집계

5. **추천 & Regime**

   * Aggregated 종목 피처 기반으로 매수 후보 종목 선정
   * 최근 N시간 전체 뉴스 통계 기반 Regime 판단
   * 추천/Regime 결과를 DB에 저장

6. **종료 전 정리**

   * 크롤링 및 처리 건수, 에러 요약 로그 기록
   * 필요 시 헬스체크 상태 갱신

### 2.2 Daily Archive Pipeline

1. **환경 초기화**

   * 설정 로드, 로거 초기화

2. **아카이빙 대상 조회**

   * `published_at < (현재 날짜 - 3일)` 조건으로 뉴스/매핑/분석/추천/Regime 레코드 조회

3. **파일로 덤프**

   * 날짜 단위 또는 기간 단위로 JSONL/Parquet 파일에 저장
   * 저장된 파일 경로와 대상 건수 로그 기록

4. **DB 삭제**

   * 덤프 성공한 레코드에 대해 실제 DB 삭제
   * 삭제 건수 로그 기록

5. **실패 처리**

   * 중간 실패 시, 어떤 단계에서 실패했는지 로그 및 알림
   * 재시도 정책은 운영 정책에 맞게 설계

---

## 3. Python 패키지 구조 설계

예시 패키지 이름: `news_regime`

```text
news_regime/
  config/
  crawler/
  parser/
  preprocess/
  mapping/
  analysis/
  recommend/
  regime/
  storage/
  archive/
  api/
  cli/
  common/
```

각 디렉터리의 역할만 설명합니다.

### 3.1 config

* 역할

  * 환경 설정 로딩 (env, yaml/json 등)
  * threshold, 시간 윈도우, 파일 경로 등 공통 파라미터 관리
* 주요 요소

  * AppConfig: 전체 애플리케이션 설정 구조
  * DbConfig, CrawlConfig, AnalysisConfig, ArchiveConfig 등 하위 설정 객체

### 3.2 crawler

* 역할

  * HTTP 클라이언트, Rate limiting, Retry 정책 담당
  * 네이버 메인 뉴스 페이지, 상세 기사 페이지 Fetch
* 주요 컴포넌트

  * HttpClient 추상화

    * Jetson 환경에서 `requests` 또는 `httpx` 사용 (동기/비동기 선택)
  * NaverNewsClient

    * mainnews 페이지 요청
    * 개별 기사 URL 요청
  * CrawlResult 구조

    * 원본 HTML, 응답코드, 응답시간 등 포함

### 3.3 parser

* 역할

  * HTML → 기사 메타/본문 구조화
* 주요 컴포넌트

  * MainNewsParser

    * mainnews 목록에서 기사 제목, URL, 언론사, 시간, 요약, 카테고리 추출
  * ArticleParser

    * 기사 상세 페이지에서 본문, 기자명, 추가 태그 추출
  * HTML 구조 변경에 대응하기 위해

    * CSS selector/xpath 등은 설정/상수로 분리
    * 파싱 실패 시 예외를 던지되, 상위에서 graceful handling

### 3.4 preprocess

* 역할

  * 텍스트 인코딩/정제, 노이즈 제거
* 주요 컴포넌트

  * TextNormalizer

    * HTML 태그 제거
    * 공백, 줄바꿈 처리
    * 특수문자 정규화
  * TimeParser

    * 네이버 표기 시간(예: “2025.12.02 10:30”) → datetime(Asia/Seoul)

### 3.5 mapping

* 역할

  * 기사 텍스트 → 종목(Symbol) 매핑
* 주요 컴포넌트

  * SymbolDictionary

    * 종목 리스트 로드 및 캐싱
    * 코드, 한글명, 영문명, 약어/별칭 관리
  * SymbolMatcher

    * 단일 기사 텍스트에 대해:

      * 후보 종목 리스트 탐색
      * 매칭 기준(완전 일치, 부분 일치, 대체 표기 등)을 적용
      * match_score 산출
  * MappingResult

    * news_id, symbol, match_score 리스트

---

### 3.6 analysis

* 역할

  * Sentiment, Risk 플래그, Impact 점수, 종목별 집계 수행
* 주요 컴포넌트

1. **SentimentAnalyzer**

   * 기사 단위 Sentiment 산출
   * 전략:

     * v1: 룰/사전 기반(“호재/악재 키워드 중심”)
     * v2 이후: ML/LLM 연결 가능하도록 인터페이스만 고정

2. **RiskTagger**

   * 규제, 실적, 신용 등 리스크 키워드/패턴 기반 플래그 설정
   * 각 플래그별 정의와 키워드 세트는 설정으로 관리

3. **ImpactScorer**

   * 언론사, 제목 표현 강도, 기사 범위(전시장/섹터/종목)를 고려해 Impact 점수 산출

4. **AggregationService**

   * 종목(symbol)별로 최근 N시간 뉴스들을 묶어 통계/대표값 계산

     * 평균/최대 Sentiment
     * 평균/최대 Impact
     * 뉴스 개수, 마지막 뉴스 시각
     * 리스크 플래그 비율

---

### 3.7 recommend

* 역할

  * Aggregated 종목 피처 → 매수 후보 종목 리스트 생성
* 주요 컴포넌트

1. **RecommendationPolicy**

   * 룰 기반 정책 정의:

     * Sentiment threshold
     * Impact 최소값
     * 리스크 플래그 허용 조건
     * 뉴스 수 최소 개수 등
   * 정책 파라미터는 `AnalysisConfig`에서 주입

2. **RecommendationEngine**

   * AggregationService 결과를 입력으로:

     * 정책을 적용하여 후보 종목 필터링
     * 각 종목에 대한 추천 점수/신뢰도 계산
   * Recommendation 객체에는:

     * symbol
     * scores (sentiment, impact, composite score)
     * risk_flag 요약
     * 추천 사유 요약 텍스트(간단한 rule-based 텍스트 생성) 포함

3. **RecommendationRepository (storage와 밀접)**

   * 새로 생성된 추천 결과를 DB에 저장
   * 기존 추천과 비교하여 상태(유지/철회) 업데이트

---

### 3.8 regime

* 역할

  * 최근 뉴스 전체 통계를 기반으로 시장 Regime 판단
* 주요 컴포넌트

1. **RegimeConfig**

   * Regime 이름, threshold 정의 (Bullish/Bearish/Risk-off 등)
   * 예: 평균 Sentiment, 부정 뉴스 비율, 규제 리스크 비율, Impact 상위 기사 수

2. **RegimeClassifier**

   * 집계된 글로벌 지표(전체 뉴스 기준)를 입력으로:

     * 룰 기반으로 Regime 라벨 선정
     * confidence 계산
     * 요약 설명 텍스트 생성

3. **RegimeRepository**

   * Regime 판단 결과를 `regime_history` 테이블에 저장
   * 최근 Regime 조회용 메서드 제공

---

### 3.9 storage

* 역할

  * DB 연동, ORM/Query Layer
* 선택

  * Jetson 환경에서 가볍게 시작하려면:

    * SQLite (단일 파일, 매우 간단)
    * 또는 PostgreSQL (Docker 컨테이너로 운용)
* 주요 컴포넌트

1. **DbSessionManager**

   * DB 연결 생성/관리
   * 트랜잭션 컨텍스트 관리

2. **엔티티/테이블 매핑**

   * NewsEntity
   * NewsSymbolMapEntity
   * NewsAnalysisEntity
   * RecommendationEntity
   * RegimeHistoryEntity
   * (필요 시) RawHtmlEntity 등

3. **Repository 계층**

   * NewsRepository

     * 기사 존재 여부 확인(중복 체크)
     * 일괄 Insert/Update
   * SymbolMapRepository
   * AnalysisRepository
   * RecommendationRepository
   * RegimeRepository

---

### 3.10 archive

* 역할

  * 3일 지난 데이터를 파일로 덤프 + DB에서 제거
* 주요 컴포넌트

1. **ArchivePlanner**

   * 어떤 날짜 범위/테이블을 아카이브할지 결정
   * 예: “2025-11-28 이전 데이터 전체”

2. **ArchiveExporter**

   * 각 테이블 데이터를 메모리 스트림/이터레이터로 읽어
   * JSONL 또는 Parquet 포맷 파일로 저장
   * 파일 경로 규칙, 디렉토리 구조 관리

     * 예: `archive/news/2025/12/news_2025-12-02.parquet`

3. **ArchiveCleaner**

   * Export 성공 건에 대한 DB 삭제
   * 삭제 건수 및 타임스탬프 기록

4. **ArchiveJob**

   * ArchivePlanner + Exporter + Cleaner를 순차 실행하는 상위 오케스트레이터

---

### 3.11 api

* 역할

  * 다른 Agent/시스템(Analyst Agent, Trader Agent 등)이 정보를 읽을 수 있는 인터페이스

* v1 선택지

  * (A) 단순 JSON 파일/리포트 생성 (로컬 연동 중심)
  * (B) 경량 FastAPI 기반 REST API (Jetson에서 충분히 가능)

* 제공 엔드포인트 (설계 수준)

  * 최근 매수 후보 종목 목록 조회
  * 현재 Regime 및 최근 N개 Regime 히스토리 조회
  * 특정 종목의 최근 뉴스/요약/Sentiment 조회

---

### 3.12 cli

* 역할

  * cron/systemd에서 호출할 수 있는 진입점
* 주요 엔트리

  * `run_hourly_cycle`

    * “크롤링 → 분석 → 추천/Regime → 저장” 전체 파이프라인 호출
  * `run_daily_archive`

    * “3일 지난 데이터 아카이브 → DB 삭제” 파이프라인 호출
  * (옵션) 디버그/테스트용:

    * 특정 날짜/시간 범위로 회귀 테스트 실행

---

### 3.13 common

* 역할

  * 공통 유틸 (로깅, 시간, 예외 타입 등)
* 주요 요소

  * LoggerFactory
  * TimeUtils (Asia/Seoul 기준 날짜/시간 함수)
  * CustomException 계층 (CrawlError, ParseError, AnalysisError 등)

---

## 4. 데이터 모델 / DB 스키마 설계 (논리 수준)

### 4.1 핵심 엔티티

1. **News**

   * id (PK)
   * title
   * body
   * url (unique)
   * publisher
   * category
   * published_at
   * crawled_at
   * created_at, updated_at

2. **NewsSymbolMap**

   * id (PK)
   * news_id (FK → News)
   * symbol
   * match_score

3. **NewsAnalysis**

   * id (PK)
   * news_id (FK → News)
   * article_sentiment
   * regulatory_risk_flag
   * earnings_risk_flag
   * credit_risk_flag
   * impact_score

4. **Recommendation**

   * id (PK)
   * symbol
   * created_at
   * sentiment_score
   * impact_score
   * risk_summary
   * composite_score
   * status (예: ACTIVE, WITHDRAWN)
   * reason_summary

5. **RegimeHistory**

   * id (PK)
   * timestamp
   * regime_label
   * confidence
   * explanation

### 4.2 인덱스 전략

* News:

  * url (unique index)
  * published_at (index)
* NewsSymbolMap:

  * symbol + news_id (index)
* NewsAnalysis:

  * news_id (unique index)
* Recommendation:

  * symbol + created_at (index)
* RegimeHistory:

  * timestamp (index)

이 구조로 **최근 3일 조회 성능**을 확보하고, 아카이브 시에도 날짜 기준으로 쉽게 필터링 가능.

---

## 5. Jetson AGX Orin / Python 3.1x 환경 고려

1. **의존성 절제**

   * CPU 위주 I/O 작업 (크롤링/텍스트 처리)이 대부분이므로, GPU 라이브러리는 불필요.
   * 가벼운 HTTP/HTML/텍스트·DB 라이브러리 위주 사용.
2. **병렬성**

   * 한 번 크롤링 시 상세 기사 요청은 네트워크 I/O 바운드이므로:

     * Thread 기반 병렬(Future/ThreadPool) 또는 async를 지원하는 구조로 설계
     * 모듈 간 인터페이스는 I/O 비동/동기를 쉽게 전환할 수 있도록 추상화
3. **리소스 관리**

   * 크롤링/분석 후 메모리 해제를 유도할 수 있는 구조 (대용량 리스트 보관 최소화)
   * 아카이브 시에도 스트리밍 방식(배치 사이즈 단위로 select → write → delete)을 고려

---

## 6. 설정/운영 포인트 설계

1. **Config 파일 구조**

   * 환경별 분리: `config.prod.yaml`, `config.dev.yaml` 등
   * 주요 항목:

     * DB 연결 정보
     * 크롤링 주기, 타임아웃, 재시도 횟수
     * Sentiment/Regime threshold
     * 아카이브 보존일(3일), 파일 경로
     * 로깅 레벨/파일 경로

2. **로깅**

   * 파일 rotation 포함 (일별/용량 기준)
   * 모듈별 logger 이름 구분 (`news_regime.crawler`, `news_regime.analysis` 등)

3. **헬스체크 정보**

   * 최근 성공적인 hourly/daily job 실행 시각을 별도 파일 또는 DB 테이블에 기록
   * 외부에서 간단한 스크립트로 “정상 운행 여부”를 확인 가능

---

## 7. 향후 확장 포인트 명시

1. **다중 뉴스 소스**

   * `crawler`와 `parser`를 “소스별 구현체”로 분리 (예: `NaverSource`, `OtherPortalSource`)
2. **LLM / Multi-Agent 연동**

   * Analysis Layer에서 **SentimentAnalyzer**를 LLM 기반 서비스 호출로 교체 가능
   * Recommendation/Regime 결과를 **Analyst Agent**/**Trader Agent** 입력 JSON 포맷으로 직접 제공
3. **실거래/시세 연동**

   * Recommendation 결과에 시세/체결강도 피처를 추가하는 별도 모듈 추가

---

여기까지가 **Req Analysis v1.0을 Python으로 구현하기 위한 설계(Architecture + Module Design)** 입니다.

다음 단계에서는, 원하시면:

* 이 설계를 기준으로 **구체적인 모듈/클래스 시그니처 설계**
* 또는 `run_hourly_cycle` 파이프라인을 중심으로 **시퀀스 다이어그램** 수준의 상세 설계

같은 것부터 차례로 풀어가면서, 그다음에 실제 코드 스켈레톤으로 넘어갈 수 있습니다.
