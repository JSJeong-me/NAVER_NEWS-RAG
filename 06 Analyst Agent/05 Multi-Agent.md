지금은 'Analyst Agent' 만 구현하지만
다음 단계로 'Trader Agent'와 연동하는 multi-agent 로 확장을 설계에 반영해야 합니다.
이 내용을 '요구사항'에 반영하여 Framework 을 비교 분석하고 선택해 주세요.  

지금 조건을 다시 정리하면:

* **현재 단계**:

  * `/home/ubuntu/news_regime` 안의 CLI (`run_crawler.sh`, `query_db.sh`, `list_news.sh`)를 30분 간격으로 실행 →
    Naver 금융 뉴스 기반 **Regime & 종목 추천 리포트**를 생성하는 **Analyst Agent**만 구현.

* **다음 단계(확장)**:

  * 이 Analyst Agent 결과를 입력으로 쓰는 **Trader Agent** (Universe 필터링, 포지션 사이징, 진입/청산 조건 판단 등)를 붙여
    **Multi-Agent 구조 (Analyst ↔ Trader ↔ Risk 등)** 로 확장해야 함.

이제 이 확장 요구를 포함해서 **프레임워크 요구사항** → **후보 비교** → **최종 선택** 순서로 정리하겠습니다.

---

## 1. 통합 요구사항 (Analyst + 향후 Trader Agent)

### 1.1 공통 요구사항

1. **툴 호출**

   * `/home/ubuntu/news_regime` 내부 CLI를 **툴로 래핑**해서 사용:

     * `run_crawler.sh`
     * `query_db.sh all/news/analysis/recommendations/regime ...`
     * (추가) Trader Agent 단계에서는:

       * KIS/OpenAPI 래퍼
       * 주문/포지션 조회/체결 이력 조회 등 Python 함수 or CLI
   * 툴 콜 결과(stdout/JSON)를 Agent들이 공유/전달할 수 있어야 함.

2. **멀티에이전트 구조**

   * 최소 구조:

     * **Analyst Agent**: 뉴스/DB 기반으로 Regime·추천·리스크 레이더 리포트 생산.
     * **Trader Agent**: Analyst 결과 + 주문/포트폴리오 상태를 바탕으로,

       * “어떤 종목을 어떤 크기/조건으로 다루어야 할지” 제안/계획 생성.
   * 이후 **Risk Agent**, **Execution Agent**까지 추가될 수 있음:

     * Risk Agent: 레버리지/익스포저/섹터 비중 등 검토.
     * Execution Agent: 실제 주문 라우팅/슬리피지 최소화 전략 등.

3. **Jetson / 로컬 환경**

   * Nvidia Jetson AGX Orin + Ubuntu + Python 3.1x.
   * 가볍고, 의존성 과하지 않은 스택 선호.

4. **실행 패턴**

   * **배치형 + (부분)대화형** 둘 다 가능해야 함:

     * 배치: 30분마다 Analyst Agent 단독 실행 (리포트 생성).
     * 대화형: 향후에는

       * “지금 Regime과 추천 종목 기준으로 Trader Agent 의견 보여줘”
       * “이 포지션들에 대해 Risk Agent 검토해줘”
         같은 interactive 멀티에이전트 플로우도 가능.

5. **오케스트레이션**

   * 현재는 단순:

     * [Cron] → Analyst Agent → 리포트
   * 향후는:

     * [Scheduler] → Analyst Agent → Trader Agent → Risk Agent → Execution Agent
   * 이 흐름을 하나의 “workflow/graph”로 표현/관리할 수 있으면 베스트.

---

## 2. 프레임워크 후보 재비교 (멀티에이전트 확장 고려)

### 2.1 OpenAI Agents SDK (Python 기반 경량 Agent SDK)

> **키워드**: 경량, tools, multi-agent 구성 가능, OpenAI 스택 친화적

* 특징

  * “Agent + Tool” 기본 추상화 제공.
  * 여러 Agent를 정의하고, 코드로 직접 orchestration 로직을 짜는 방식 (프레임워크 자체가 “multi-agent runtime”이라기보단 “agent building kit”에 가깝다).
  * OpenAI 툴 호출(Function calling), Responses API와 자연스럽게 연계.

* Analyst + Trader 시나리오에서의 사용 모습

  ```text
  AnalystAgent:
    - tools: run_crawler, query_db_all, parse_db_output
    - output: AnalystReport (Regime + 추천 + 리스크 레이더 JSON)

  TraderAgent:
    - tools: get_positions, get_universe, price_data, send_order(dry-run)
    - input: AnalystReport + 계좌 상태
    - output: TradePlan (종목/사이즈/조건)

  Orchestrator (Python 코드):
    - call AnalystAgent
    - pass result to TraderAgent
    - optional: pass to RiskAgent, ExecutionAgent 등
  ```

* 장점

  * 이미 OpenAI API를 쓰고 있고, Jetson 환경에서도 **의존성 가볍게 유지** 가능.
  * Analyst Agent → Trader Agent로의 **데이터 전달/합성 로직을 코드로 깔끔하게 표현** 가능.
  * 멀티에이전트 확장 시에도,

    * 새로운 Agent를 class/function으로 정의하고 orchestrator에서 순차/조건 실행 구현.
  * “뉴스 기반 Regime 판단 + Trader Agent와의 연동”처럼 **도메인 맞춤 플로우를 직접 설계하기 좋음**.

* 단점

  * LangGraph처럼 “시각적인 그래프/상태 머신”은 제공하지 않기 때문에,

    * 아주 복잡한 장기 워크플로/재시도/중단 후 재개까지 필요해지면
    * orchestration 코드를 직접 정리해야 함.

---

### 2.2 LangGraph (+ LangChain)

> **키워드**: 그래프형 워크플로, 멀티에이전트 오케스트레이션, 상태 관리

* 특징

  * 노드와 엣지로 **에이전트/툴 호출 흐름을 명시적으로 모델링**.
  * 장기 실행, durable execution, human-in-the-loop 등을 내장 레벨에서 고려. ([LangChain Docs][1])
  * 여러 Agent (Analyst, Trader, Risk, Execution)를 각각 노드로 등록하고,
    “Analyst 결과가 특정 조건을 만족하면 Trader로 넘어가고, Risk를 거쳐 Execution으로” 같은 구조를 그래프로 표현.

* Analyst + Trader 시나리오 예

  ```text
  Node A: AnalystAgentNode
    - tools: query_db, run_crawler
    - output: AnalystReport

  Node B: TraderAgentNode
    - tools: positions, universe, order_simulator
    - input: AnalystReport
    - output: TradeCandidates

  Node C: RiskAgentNode
    - input: TradeCandidates + current exposure
    - output: RiskFilteredPlan

  Node D: ExecutionAgentNode
    - input: RiskFilteredPlan
    - output: ExecutionCommands

  Graph: A → B → C → D
  ```

* 장점

  * 지금은 Analyst Agent 하나지만,

    * Trader/Risk/Execution을 붙이는 순간 **그래프형 파이프라인**의 장점이 크게 살아남.
  * 각 노드(Agent)별 상태/로그/실패 재시도 등 관리가 쉬움.
  * 장기적으로 “리스크 오버라이드 발생 시 다시 Analyst로 되돌아가 재요약” 같은 **루프 구조**도 표현 가능.

* 단점

  * LangChain/LangGraph 의존성으로 Jetson 환경에 **프레임워크 무게가 커짐**.
  * 처음 세팅/학습 비용이 Agents SDK보다 큼.
  * “단일 30분 배치에서 간단히 리포트만 뽑는 단계”에서는 과한 느낌이 있을 수 있음.

---

### 2.3 AutoGen / Microsoft Agent Framework

> **키워드**: 대화형 multi-agent, 코드·툴·리뷰 협업

* Analyst/Trader 시나리오

  * AnalystAgent와 TraderAgent가 “대화”하면서:

    * Analyst: “현재 Regime은 risk_on → sideways입니다. 추천 종목은 005930, 035420입니다.”
    * Trader: “그럼 최대 레버리지 x까지, 삼성전자 비중을 y%까지 가져가도 될까요?”
    * RiskAgent: “섹터 집중도가 너무 높으니 줄이자” … 이런 다자간 협의 구조.

* 장점

  * “에이전트가 서로 역할 분담하고 토론하는 구조”에는 강함. ([GitHub][2])
  * 텍스트 베이스 human-in-the-loop를 자연스럽게 삽입 가능.

* 단점

  * 지금 목표는 **뉴스 기반 배치 리포트 + Trader Agent 연동**이고,

    * 에이전트 간 “토론/합의”가 핵심은 아님.
  * 프레임워크가 비교적 heavy 하고 추상화 레이어가 많아,

    * Jetson 기준으로 유지보수/튜닝 비용이 커질 수 있음.

---

### 2.4 CrewAI

> **키워드**: 여러 Agent를 “Crew로 묶어서” 한 번에 태스크 수행

* 장점

  * YAML/설정 기반으로 Analyst/Trader/Risk/Execution 역할을 나누고 협업시키기 쉬움. ([CrewAI Documentation][3])
  * 실험/프로토타입을 빠르게 만들기 좋음.

* 단점

  * 상업 트레이딩/리스크 시스템처럼 **정교한 제어 + 로컬 CLI/네트워크 툴과 깔끔한 통합**이 중요한 곳보다는,

    * “리서치용, 문서 요약/초안 만들기” 중심에 좀 더 적합.
  * Jetson + CLI 기반 환경에서는 Agents SDK/직접 구현 쪽이 더 직관적일 수 있음.

---

## 3. 결론: 멀티에이전트 확장을 고려한 선택

### 3.1 현실적인 단계별 전략

1. **1단계 (지금)** – Analyst Agent 단독, 배치형 리포트

   * 요구:

     * 30분마다 CLI 호출 + Regime/추천 리포트 생성.
     * Multi-agent는 아직 사용 안 함.
   * 최적:

     * **OpenAI Agents SDK** 또는 **심지어 프레임워크 없이 직접 구현**도 가능.

2. **2단계** – Trader Agent 연동 (멀티에이전트 시작)

   * 요구:

     * Analyst Agent 결과(JSON)를 Trader Agent 입력으로 넘김.
     * Trader Agent는 포지션/유니버스/리스크 제약을 고려해 “Trade Plan” 생성.
     * Orchestrator가 전체 플로우를 관리.
   * 최적:

     * 여전히 **OpenAI Agents SDK + Python orchestrator**로 충분:

       * `run_analyst() -> AnalystReport`
       * `run_trader(AnalystReport, account_state) -> TradePlan`
       * 필요 시 Risk/Execution Agent를 나중에 같은 패턴으로 추가.

3. **3단계** – 복잡한 멀티에이전트/워크플로 (Risk/Execution까지, 루프 구조, human-in-the-loop)

   * 요구:

     * “Regime이 risk_off면 Trader 계획 일부 무효화 → Analyst에게 재검토 요청” 같은 루프.
     * 실패 재시도, 부분 롤백, 상태 저장 등.
   * 이 시점에서 **LangGraph** 같은 그래프 오케스트레이션을 추가 고려.

---

### 3.2 최종 추천

**요구사항(지금 Analyst → 이후 Trader → 멀티에이전트 확장)을 모두 고려했을 때:**

> **1차 구현 및 초기 멀티에이전트(Analyst + Trader) 단계에서는
> OpenAI Agents SDK + Python 기반 Orchestrator**를 사용하는 것이 가장 합리적입니다.

#### 왜 이 선택이 좋은지 요약:

1. **현재 요구와 딱 맞음**

   * 지금은 배치형 Analyst Agent 1개만 필요 → SDK 없이도 되지만,
     곧바로 Trader Agent를 붙일 수 있도록 **Agent/Tool 레이어만 가볍게 도입**하는게 좋음.

2. **좋은 확장 경로**

   * 나중에 Trader/Risk/Execution Agent를 추가할 때,

     * 각 Agent를 하나의 클래스/함수로 정의,
     * Orchestrator에서 순차/조건적인 호출을 구현하면 멀티에이전트 구조 완성.
   * 이미 OpenAI 모델을 쓰고 있으니 **모델/툴 통합이 최소 마찰**.

3. **Jetson/CLI 친화적**

   * 무겁지 않은 의존성, 코드로 직접 CLI 호출·파싱 제어 가능.
   * GPU/환경 설정을 복잡한 프레임워크에 맞추느라 시간을 쓰지 않아도 됨.

4. **LangGraph/AutoGen 등 “더 무거운” 프레임워크는,**

   * 정말로 복잡한 멀티에이전트 오케스트레이션이 필요해질 때(3단계 이후)에만 도입해도 늦지 않음.
   * 그때는 “현재 OpenAI Agents SDK 기반 Agent들을 노드로 감싸서 LangGraph에 올리는” 방식으로 자연스럽게 업그레이드할 수 있음.

---

원하시면 다음 단계에서:

* **OpenAI Agents SDK를 기준으로 한 구체 설계**:

  * `AnalystToolset` (run_crawler, query_db_all 등)
  * `AnalystAgent` system prompt/입출력 스키마
  * `TraderAgent`와 결과 JSON 계약(Interface)을 미리 정의해서,
* 나중에 Trader Agent를 끼워 넣기 쉬운 **설계 스켈레톤**까지 같이 정리해 드릴게요.

[1]: https://docs.langchain.com/oss/python/langgraph/overview?utm_source=chatgpt.com "LangGraph overview - Docs by LangChain"
[2]: https://github.com/microsoft/autogen?utm_source=chatgpt.com "microsoft/autogen: A programming framework for agentic AI"
[3]: https://docs.crewai.com/en/introduction?utm_source=chatgpt.com "Introduction"
