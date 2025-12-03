가장 일반적인 방법은 **cron**에 등록해서 15분마다 실행하는 겁니다.

(환경: conda env = base, 프로젝트: /home/ubuntu/news_regime, python = /home/ubuntu/miniconda3/bin/python)

1. 실행용 wrapper 스크립트 만들기

프로젝트 디렉터리에서:

cd /home/ubuntu/news_regime

cat > run_analyst_agent.sh << 'EOF'
#!/usr/bin/env bash

# 1) 에러 나면 바로 종료하도록 설정 (옵션)
set -e

# 2) conda 초기화 스크립트 로드
#    miniconda3 설치 경로에 맞게 수정 (지금 정보 기준으로 아래가 맞음)
source ~/miniconda3/etc/profile.d/conda.sh

# 3) base 환경 활성화
conda activate base

# 4) OpenAI API 키 설정
#    실제 키로 바꾸세요. (또는 별도 파일에서 source 해도 됨)
export OPENAI_API_KEY="sk-xxx"

# 5) 프로젝트 디렉터리로 이동
cd /home/ubuntu/news_regime

# 6) (옵션) 크롤러 먼저 실행 – config에서 이미 돌리고 있다면 생략 가능
# ./run_crawler.sh

# 7) Analyst Agent 1회 실행
python -m analyst_agent.run_once
EOF

실행 권한 부여:

chmod +x /home/ubuntu/news_regime/run_analyst_agent.sh


🔐 보안상 OPENAI_API_KEY를 스크립트에 직접 쓰기 싫다면:

echo 'export OPENAI_API_KEY="sk-xxx"' > ~/.openai_env
chmod 600 ~/.openai_env


그리고 run_analyst_agent.sh 안에서:

source ~/.openai_env


3. crontab에 15분 주기 실행 등록

crontab 편집:

crontab -e


열린 파일에 맨 아래 한 줄 추가:

*/15 * * * * /home/ubuntu/news_regime/run_analyst_agent.sh 2>&1


4. 수동 테스트 한 번 해보기 (권장)

cron에 맡기기 전에 수동 한 번:

cd /home/ubuntu/news_regime
./run_analyst_agent.sh


에러 없이 돌아가면 OK

실패하면 터미널에서 에러 메시지 보고 수정 → 이후 cron이 동일하게 실행

이렇게 설정해 두면,

cron → wrapper script 실행

wrapper script → conda base 활성화 + OPENAI_API_KEY 설정 + python -m analyst_agent.run_once
가 15분마다 자동으로 돌아가게 됩니다.







예를 들어, `/home/ubuntu/news_regime`에서 아래 명령을 실행한다고 가정하면:

```bash
cd /home/ubuntu/news_regime
python -m analyst_agent.run_once
```

이를 15분마다 돌리는 cron 설정은 다음과 같이 쓸 수 있습니다.

```bash
*/15 * * * * cd /home/ubuntu/news_regime && /usr/bin/python -m analyst_agent.run_once >> /home/ubuntu/news_regime/logs/analyst_agent_cron.log 2>&1
```

사용 방법:

```bash
crontab -e
```

열린 편집기에 위 한 줄을 추가 후 저장하면 됩니다.

* `*/15 * * * *` : 15분마다
* `cd /home/ubuntu/news_regime` : 프로젝트 디렉터리로 이동
* `/usr/bin/python3` : 시스템 python3 경로 (필요시 `which python3`로 확인해서 수정)
* `>> ...log 2>&1` : stdout/stderr를 로그 파일에 누적 저장


crontab 에서 실행할때 'export OPENAI_API_KEY=sk-xxx' 와 'conda activate base' 와 같은 환경 설정은 어떻게 하나요?


좋은 질문이에요. cron은 **로그인 쉘이 아니라서**

* `conda activate base`
* `export OPENAI_API_KEY=...`

같은 걸 자동으로 안 읽어옵니다. 그래서 **명시적으로 환경을 세팅**해줘야 합니다.
방법은 2~3가지가 있는데, 현실적으로 가장 안전한 패턴 두 개를 정리해 드릴게요.

---

## ✅ 방법 1: “환경이 다 들어있는” 쉘 스크립트를 만들고 cron에서 그 스크립트만 실행

### 1) 실행용 스크립트 작성

`/home/ubuntu/news_regime/run_analyst_agent.sh` 같은 파일을 하나 만듭니다.

```bash
#!/usr/bin/env bash

# 1) conda 초기화 스크립트 불러오기 (conda 명령어 활성화)
source ~/miniconda3/etc/profile.d/conda.sh   # 경로는 실제 설치 경로 확인

# 2) 사용할 환경 활성화
conda activate base   # 또는 stock, news_env 등 실제 사용하는 env

# 3) OpenAI 키 설정 (실제 키는 여기 넣지 마시고, .env에서 읽어도 됨)
export OPENAI_API_KEY="sk-xxx"

# 4) 프로젝트 디렉터리로 이동
cd /home/ubuntu/news_regime

# 5) 분석 사이클 1회 실행
python -m analyst_agent.run_once
```

> ⚠️ `~/miniconda3/etc/profile.d/conda.sh` 경로는
> `echo $CONDA_EXE` / `which conda` 로 실제 위치 확인해서 수정하세요.

그리고 실행 권한 추가:

```bash
chmod +x /home/ubuntu/news_regime/run_analyst_agent.sh
```

### 2) cron에 등록

```bash
crontab -e
```

열린 곳에 아래 한 줄 추가:

```bash
*/15 * * * * /home/ubuntu/news_regime/run_analyst_agent.sh >> /home/ubuntu/news_regime/logs/analyst_agent_cron.log 2>&1
```

이제 cron은 **15분마다**

* conda env 활성화
* OPENAI_API_KEY 설정
* `python -m analyst_agent.run_once` 실행
  까지 한 번에 잘 돌립니다.

---

## ✅ 방법 2: conda activate 대신 “환경의 python”을 직접 사용 + cron에서 env 설정

이 방법이 더 깔끔할 때도 많습니다.
핵심 아이디어: **굳이 `conda activate` 안 하고, 해당 env의 `python` 경로를 직접 호출**.

### 1) env의 python 경로 확인

먼저 평소에 쓰는 환경을 activate 한 뒤:

```bash
conda activate base      # 또는 실제 사용하는 env
which python
```

예를 들어 이런 결과가 나왔다고 합시다:

```text
/home/ubuntu/miniconda3/envs/base/bin/python
```

이 경로를 cron에서 직접 사용합니다.

### 2) crontab에서 환경 변수 + python 경로 지정

```bash
crontab -e
```

여기에 **위쪽에 환경변수**를 먼저 선언해 두고:

```bash
OPENAI_API_KEY=sk-xxx
```

그 아래에 job을 씁니다:

```bash
*/15 * * * * cd /home/ubuntu/news_regime && /home/ubuntu/miniconda3/envs/base/bin/python -m analyst_agent.run_once >> /home/ubuntu/news_regime/logs/analyst_agent_cron.log 2>&1
```

이렇게 하면:

* `OPENAI_API_KEY`는 cron에서 환경으로 들어가고
* conda activate 없이도 해당 env의 python으로 바로 실행됩니다.

> 🔐 보안적으로는:
>
> * `OPENAI_API_KEY`를 crontab에 직접 넣는 대신
>   `/home/ubuntu/.openai_env` 같은 파일에 `export OPENAI_API_KEY=...` 저장해두고
>   wrapper 스크립트에서 `source ~/.openai_env` 하는 것도 방법입니다.

---

## 정리 – 어떤 방법을 쓰면 좋을까?

* **단순 / 안정성**:
  → ✅ **방법 2 (env의 python 직접 호출)** 추천
  → `conda activate` 없이도 깔끔하게 돌아갑니다.

* **이미 shell 스크립트가 많은 구조**거나
  크롤러/분석 등을 한 스크립트에서 전부 제어하고 싶다면:
  → ✅ **방법 1 (wrapper script + conda activate)**가 관리하기 편합니다.

---

원하시면

* 지금 쓰시는 conda env 이름,
* `which python` 결과,
* 현재 `run_crawler.sh`, `query_db.sh` 위치
  를 알려주시면, 바로 붙여 넣어서 쓸 수 있는 **최종 cron + 스크립트 조합**도 만들어 드릴게요.


  """ 사용중인 conda env는 base (base) ubuntu@ubuntu:~/news_regime$ which python /home/ubuntu/miniconda3/bin/python (base) ubuntu@ubuntu:~/news_regime$ ls -l *.sh -rwxrwxr-x 1 ubuntu ubuntu 201 Dec 3 13:16 list_news.sh -rwxrwxr-x 1 ubuntu ubuntu 204 Dec 3 13:19 query_db.sh -rwxrwxr-x 1 ubuntu ubuntu 3843 Dec 3 17:56 run_crawler.sh (base) ubuntu@ubuntu:~/news_regime$ pwd /home/ubuntu/news_regime """ docstring 의 내용으로 '방법 1 (wrapper script + conda activate)' 실행 방법을 정리해 주세요.




  
