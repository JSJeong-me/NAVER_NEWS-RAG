'Analyst Agent' 구현에 사용할 agent framework 을 비교 분석하고 추천해 주세요.



일단 정리부터 하면, 지금 만들려는 **Analyst Agent**는:

* Jetson AGX Orin / Ubuntu / Python 3.x
* `/home/ubuntu/news_regime` 안에 있는
  `run_crawler.sh`, `query_db.sh`, `list_news.sh` 를 **도구(tool)** 로 쓰고
* **30분(설정 가능)마다 실행**해서

  * `query_db.sh all --hours 24 --limit N` 결과를 파싱하고
  * Regime 변화·급격한 변동성·추천 종목을 분석해서
  * 사람이 바로 읽을 수 있는 리포트 텍스트를 뽑는 **배치형 에이전트**

입니다.

이 패턴을 기준으로 주요 Agent 프레임워크들을 비교해볼게요.

---

## 1. 이 Analyst Agent에 필요한 프레임워크 요구조건

프레임워크 선택 기준을 먼저 정리하면:

1. **도구 호출이 쉬워야 함**

   * 로컬 쉘 스크립트(`./query_db.sh ...`) / Python 함수 래핑
   * 결과(stdout) 파싱 후 LLM에 넘기거나, LLM이 직접 읽고 판단.

2. **“주기적 배치 + 단일/소수 Agent” 구조**

   * 크론/시스템 타이머가 30분마다 프로세스를 깨우고
   * 에이전트는 1~2회 툴 호출 → 한 번의 리포트 생성 후 종료.
   * 긴 대화형 멀티 턴보다는 **단일 러너 파이프라인에 가깝다**.

3. **Jetson / 경량 환경**

   * 의존성 과한 프레임워크는 피하는 게 좋음.
   * CLI + OpenAI API + 약간의 orchestration 정도가 적당.

4. **향후 확장 여지**

   * 나중에 “Trader Agent / Risk Agent / Execution Agent”랑 연결하거나
   * Regime 판단 로직을 조금 더 복잡하게 그래프/워크플로로 바꾸고 싶을 수 있음.

---

## 2. 주요 후보 프레임워크 비교

### 2.1 OpenAI Agents SDK (Python)

* 개요
  OpenAI가 내놓은 **경량 agent SDK**.
  “아주 작은 primitive로 multi-agent workflow를 구성하는” 걸 목표로 함. ([OpenAI GitHub][1])

* 특징

  * Python 패키지 하나로 **간단한 Agent + Tools** 구조 정의 가능.
  * Tools를 Python 함수로 감싸서, LLM이 필요할 때 호출:

    * 예: `run_query_db_all(hours: int, limit: int) -> str`
      를 `FunctionTool`로 정의해 툴 콜로 사용. ([OpenAI GitHub][2])
  * OpenAI Responses/Chat API 뿐 아니라 다른 LLM도 붙일 수 있는 provider-agnostic 설계. ([GitHub][3])
  * 프레임워크 자체는 **스케줄링을 안 맡고**, 외부에서 `python -m analyst_agent`를 30분마다 호출하는 구조에 잘 어울림.
  * Swarm 실험에서 발전한, 비교적 최신/안정적인 기반. ([OpenAI GitHub][1])

* 장점 (이 use-case 기준)

  * 이미 OpenAI 모델을 쓰고 있고, Jetson에서 **경량으로 굴리기 좋음**.
  * “Agent + Tool” 레이어만 제공해서, 지금처럼

    * 도구: `run_crawler`, `query_db_all` 등
    * 에이전트: “리포트 작성 Analyst”
      로 설계하기 딱 맞음.
  * 나중에 **추가 Agent** (예: “Risk Reviewer Agent”, “Trader Agent”) 붙이기도 쉬움.

* 단점

  * 아주 복잡한 그래프형 워크플로(여러 상태, 브랜치, 재시도, human-in-the-loop)를 설계하려면 LangGraph 같은 것보다 표현력이 덜 풍부함.
  * 내장 스케줄링이나 거대한 오케스트레이션 플랫폼은 아닌, “코어 SDK”에 가깝다.

---

### 2.2 LangGraph (+ LangChain)

* 개요
  LangChain 팀이 만든 **저수준 오케스트레이션 프레임워크**.
  “상태를 가진 장기 실행 에이전트, 복잡한 워크플로, durable execution”에 초점. ([LangChain Docs][4])

* 특징

  * 워크플로를 그래프(node/edge + state)로 모델링:

    * Node: LLM 호출, 툴 호출, 파서 등
    * Edge: 다음에 어느 Node로 갈지 결정하는 로직.
  * 장점:

    * 재시도, 중단 후 재개, 히스토리 관리, human-in-the-loop 등.
    * 장기 실행/복잡한 multi-step pipeline에 강함.

* 장점 (이 use-case 기준)

  * Analyst Agent 파이프라인을 **그래프로 명시**하고 싶다면:

    * Node1: `query_db.sh` 호출
    * Node2: 파싱
    * Node3: Regime 분석
    * Node4: 추천 종목 분석
    * Node5: 리포트 생성/저장
  * 향후:

    * 같은 그래프 안에 “다른 Agent” (예: Risk Manager)를 추가하고,
    * 인간 승인 노드(HITL)를 섞는 등 **복잡한 오케스트레이션으로 확장**하기 좋음.

* 단점

  * 지금의 “30분마다 한번 돌고 끝나는 단순 파이프라인”에는 **조금 과한 무게**일 수 있음.
  * LangChain 기본 의존성이 따라와서 Jetson 환경에서는 dependency footprint가 늘어남.

---

### 2.3 Microsoft AutoGen (0.7.x / AG2 계열)

* 개요
  잘 알려진 multi-agent 프레임워크. 여러 에이전트가 서로 대화하며 협업하는 구조. ([GitHub][5])
  최근에는 **Microsoft Agent Framework**로 방향이 옮겨가는 중. ([Microsoft Learn][6])

* 특징

  * “각 Agent는 디지털 동료” 라는 철학으로,

    * ResearchAgent, CriticAgent, CoderAgent 등 **역할 분업**에 최적화.
  * 코드 실행, 툴 호출, 인간 참여까지 섞은 대화형 워크플로에 강함.

* 장점

  * 이미 AutoGen 0.7.x를 다른 프로젝트(트레이딩 등)에 쓰고 계시다면,

    * Analyst Agent도 같은 스택 위에 얹어 multi-agent 구조로 확장 가능.
  * 예:

    * NewsCrawlerAgent (기존 CLI 호출)
    * AnalystAgent (Regime/종목 분석)
    * ReporterAgent (리포트 텍스트 작성)

* 단점 (이 use-case 기준)

  * Analyst Agent는 **단일 러너 + 단일 리포트**에 가깝기 때문에,

    * AutoGen의 “대화형 multi-agent” 강점이 크게 필요하지 않을 수 있음.
  * 프레임워크 자체가 다른 것보다 **무겁고 추상화 레이어가 많음**.

---

### 2.4 CrewAI

* 개요
  독립적인 Python multi-agent 프레임워크. “Crew(팀)” 단위로 여러 Agent를 정의하고 협업시키는 구조. ([CrewAI Documentation][7])

* 특징

  * YAML 기반 설정, high-level abstraction으로 비교적 빠르게 multi-agent set-up 가능.
  * 웹 검색/툴콜/문서 분석 등 예제가 잘 정리되어 있고 커뮤니티도 활발.

* 장점

  * “ResearchCrew + AnalystCrew + ReporterCrew” 같은 구조로 설계할 때는 편리.
  * 향후 Web UI/Studio 등과 연동해 시각화를 하고 싶다면 고려 대상.

* 단점

  * Again, 이번 Analyst Agent는 **하나의 명확한 파이프라인 + 비교적 심플한 역할**이라,

    * CrewAI의 multi-agent 협업 구조를 쓰면 오히려 복잡도가 올라갈 수 있음.
  * Jetson 환경에서 의존성/무게감 측면에서도 굳이 쓸 이유가 크지 않음.

---

### 2.5 “프레임워크 없이 직접 구현” (baseline)

* 사실 이번 Analyst Agent는,

  * `subprocess.run(["./query_db.sh", "all", "--hours", "..."])`
  * * Regime/추천 분석 로직
  * * Markdown 리포트 generator
      만으로 **충분히 구현 가능**합니다.
* 그 위에 LLM 사용도:

  * “LLM = openai.ChatCompletion.create() 호출 + 프롬프트 템플릿” 정도만으로도 문제 없음.

단, 이미 다른 multi-agent 시스템(트레이딩, Risk Manager 등)을 많이 쓰고 있기 때문에:

* 장기적으로 다른 에이전트들과 **같은 스택/모니터링 체계**로 묶고 싶다면
  → 완전 DIY 보다는 “가벼운 agent SDK 위에 얹는 것”이 유지보수에 유리합니다.

---

## 3. 결론: 이 Analyst Agent에 대한 추천

### 3.1 1순위 추천: **OpenAI Agents SDK + 외부 스케줄러**

**이 프로젝트에 가장 잘 맞는 조합은:**

* 외부 스케줄러: `cron` 또는 `systemd timer` (30분마다)
* 내부: **OpenAI Agents SDK** 기반의 단일 Analyst Agent

입니다.

**구체적인 그림**

1. Tools (Python 함수)

   * `run_crawler_tool()`  → `subprocess.run("./run_crawler.sh")`
   * `query_db_all_tool(hours: int, limit: int)` → `subprocess.run("./query_db.sh all --hours ...")`
   * (옵션) `list_news_tool()`

2. Agent

   * Role: “Naver News 기반 시장 Regime/변동성/추천 종목 리포트를 작성하는 Analyst”
   * Loop:

     * 필요 시 `run_crawler_tool` 호출 (설정에 따라)
     * `query_db_all_tool` 호출 → 결과 텍스트를 state에 저장
     * 파싱/요약용 LLM 호출 → [Regime Alert ~ Summary] 리포트 생성

3. 스케줄링

   * Linux cron 예:

     ```cron
     */30 * * * * cd /home/ubuntu/news_regime && python -m analyst_agent.run_once >> logs/analyst_agent.log 2>&1
     ```
   * Agents SDK는 **에이전트 실행/툴 로직에만 집중**, 배치 스케줄링은 OS 수준에서 처리.

이렇게 하면:

* Jetson 환경에 불필요한 무거운 의존성 없이,
* LLM + Tool 콜 구조를 **표준화된 방식으로 관리**할 수 있고,
* 나중에 이 Analyst Agent를 다른 multi-agent 시스템(예: Trading Agent 팀)과 붙일 때도 재사용이 쉽습니다.

---

### 3.2 “복잡한 오케스트레이션으로 확장”을 이미 염두에 두고 있다면

* 만약 장기적으로:

  * Analyst Agent → Risk Manager → Trader Agent까지를
    하나의 **복잡한 상태 그래프/워크플로**로 묶고 싶다
  * 그리고 중간중간 human-in-the-loop(승인·코멘트)까지 넣고 싶다

라면,

* **LangGraph**를 워크플로 오케스트레이션 레이어로 두고,
* 각 Node 내부에서 OpenAI Agents SDK 또는 단순 openai 호출을 사용하는 구조도 좋습니다. ([LangChain Docs][4])

다만, **이번 Analyst Agent 단독 구현만 놓고 보면**:

> “OpenAI Agents SDK + cron”이
> 가장 단순하고, Jetson 환경과도 잘 맞고,
> 기존 OpenAI 스택과 통합도 자연스러운 선택입니다.

---

원하시면 다음 단계에서:

* OpenAI Agents SDK 기준

  * `Tool` 시그니처 설계 (run_crawler, query_db_all 등)
  * Analyst Agent의 system prompt + tool 사용 정책
  * `run_once()` 형태의 실행 엔트리포인트 설계

까지 구체적인 설계 스켈레톤을 짜 드릴게요.

[1]: https://openai.github.io/openai-agents-python/?utm_source=chatgpt.com "OpenAI Agents SDK"
[2]: https://openai.github.io/openai-agents-python/tools/?utm_source=chatgpt.com "Tools - OpenAI Agents SDK"
[3]: https://github.com/openai/openai-agents-python?utm_source=chatgpt.com "openai/openai-agents-python: A lightweight, powerful ..."
[4]: https://docs.langchain.com/oss/python/langgraph/overview?utm_source=chatgpt.com "LangGraph overview - Docs by LangChain"
[5]: https://github.com/microsoft/autogen?utm_source=chatgpt.com "microsoft/autogen: A programming framework for agentic AI"
[6]: https://learn.microsoft.com/en-us/agent-framework/overview/agent-framework-overview?utm_source=chatgpt.com "Introduction to Microsoft Agent Framework"
[7]: https://docs.crewai.com/en/introduction?utm_source=chatgpt.com "Introduction"
