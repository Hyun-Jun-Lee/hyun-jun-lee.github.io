---
title: "airflow with mysql docker compose" #Article title.
date: 2023-08-02
category: [AIRFLOW,AIRFLOW] #One, more categories or no at all.
tag: [airflow, python]
---

# Backfill & Clear

## Catch Up

- 공식문서 설명

> Airflow DAG는 start_date, end_date, 그리고 비 데이터셋 일정으로 정의되어 스케줄러가 개별 DAG 실행 요소로 바꾸어 일련의 간격을 실행합니다. 
스케줄러는 기본적으로 마지막 간격 이후 실행되지 않은 모든 데이터 간격에 대한 DAG 실행을 시작합니다. 이 개념을 Catchup이라고 합니다. 
DAG가 Catchup을 처리하도록 작성되지 않은 경우(예: 간격으로 제한되지 않고 현재 시간을 기준으로 한 경우), Catchup을 끄고 싶을 수 있습니다. 
이는 DAG의 catchup=False로 설정하거나 설정 파일의 catchup_by_default=False로 설정하여 수행할 수 있습니다. 
이 기능이 꺼져 있는 경우, 스케줄러는 오직 최신 간격에 대한 DAG 실행만 생성합니다
> 

매주 월요일마다 데이터를 처리하는 DAG가 있을 때, 이 DAG의 `start_date=2023-01-01` , `schedule_interval='0 0 * * 1` (월요일0시)로 설정 했다.

그런데 1월9일 까지 서버 문제로 airflow가 작동하지 않았고 catchup이 설정되어 있는 상태에서 재가동 시키면,

airflow는 월요일마다 서버 에러로 인해 실행되지 않았던 기간(01-02~01-09)에 대해 차례로 DAG를 실행한다.

즉, `catchup=True` 는 지난 기간에 대해 작업을 수행하면서 최신 데이터까지 처리하는 것이다.

### catch_up=True 설정 전 주의사항

- `start_date` 와 `schedule_interval` 다시 확인
    - date range가 크거나 인터벌이 짧은 경우, catchup 프로세스에 시간이 오래 걸리고 airflow 시스템 속도에도 영향을 끼침
- 각 task들 의존성 확인
- 멱등성 보장하는 task인지 확인

## BackFill

Backfill은 DAG가 이미 배포되어 실행 중인데, 해당 DAG를 `start_date` 이전의 데이터를 원할 때나 지금까지 했던 작업들을 다시 실행해야 할때 사용한다.

```python
airflow dags backfill \
    --start-date START_DATE \
    --end-date END_DATE \
    dag_id
```

ex)

- 로직이 변경되어서 전체 데이터에 반영해야 할 때,
- 컬럼이 변경되어 모든 데이터에 append 작업을 해야 할 때

Backfill은 지정한 기간 동안 DAG 재시작, 지정한 기간동안 전체 재시작, 지정한 기간 지정한 상태일 때만 재시작 등등을 지정하여 실행할 수 있다. 

```python
airflow dags backfill [-h] [-c CONF] [--continue-on-failures]
                      [--delay-on-limit DELAY_ON_LIMIT] [--disable-retry] [-x]
                      [-n] [-e END_DATE] [-i] [-I] [-l] [-m] [--pool POOL]
                      [--rerun-failed-tasks] [--reset-dagruns] [-B]
                      [-s START_DATE] [-S SUBDIR] [-t TASK_REGEX]
                      [--treat-dag-as-regex] [-v] [-y]
                      dag_id
```

- `-c` : Dagrun의 config 속성 피클화
- `--continue-on-failure` : 일부 작업이 실패해도 backfill 계속 진행
- `delay-on-limit` : dag를 재실행하기 전에 최대dag활성 수에 도달 했을 때 기다리는 시간
- `-i,**--ignore-dependencies**` : 업스트림 작업 건너뛰고 정규표현식에 맞는 작업만 실행(task_regex에서만 설정 가능)
- `--disable-retry` : 설정하면, 작업이 실패해도 재시도하지 않는다.
- `-n` : 각 작업에 대해 모의 실행을 수행합니다. 각 작업에 대한 템플릿 필드만 렌더링하고 다른 작업은 렌더링하지 않습니다
- `-I` : `depends_on_past` 옵션 설정 무시, 즉 이전 작업 실행 실패해도 다음 작업 진행
- `-m` : 작업을 실행하지 않고 성공한 것으로 표시
- `--rerun-failed-tasks` : backfill은 예외를 던져주는 대신 지정한 기간에서 실패한 모든 작업을 자동으로 재실행
- `--reset-dagruns` : 기존 backfill 관련 dag 실행을 모두 삭제하고 새로 시작
- `-y` : 확인하는 프롬프트 실행하지 않음

```python
# 지정한 기간동안 backfill 수행하지 않을 날짜만 수행
airflow dags backfill --start-date {date} --end-date {date} dag_id
# 지정한 기간동안 backfill 모든 재실행
airflow dags backfill --start-date {date} --end-date {date} --reset-dagruns dag_id
# 지정한 기간동안 실패한 task들만 재실행
airflow dags backfill --start-date {date} --end-date {date} --rerun-failed-tasks
```

## Clear

clear는 기존에 실행되었던 DagRun을 지워준다. task 인스턴스를 삭제하는 것이 아니라 `max_tries` 를 초기화하고 현재 작업 상태를 none으로 만들어 설정해서 작업이 재실행 되도록 한다.

```python
airflow tasks clear dag_id \
    --task-regex task_regex \
    --start-date START_DATE \
    --end-date END_DATE
```

- `R, --dag-regex`
    - DAG의 이름을 정확하게 지정하지 않는 대신, DAG_ID를 regex를 이용해 검색할 수 있게 합니다.
- `d, --downstream`
    - 클리어 대상이 되는 태스크의 하위 태스크(현재 태스크에 의존성을 갖고 있는 태스크)를 포함해서 클리어합니다.
- `e END_DATE, --end-date END_DATE`
    - Clear 할 마지막 DagRun의 **data_interval 시작 시각**을 의미합니다.
    - YYYY-MM-DD 형식을 지원하며, 만약 시간을 붙이고 싶다면 `YYYY-MM-DDTHH:mm:SS` 형식으로 나타낼 수 있습니다.
    - timezone을 추가하고 싶다면 `+` 로 나타낼 수 있습니다. 예를 들어 KST는 다음처럼 표현합니다.
        - `2022-09-01T00:10:00+09:00`
- `X, --exclude-parentdag`
    - Clear 된 Task가 어떤 SubDAG의 일부일 때 ParentDAG을 Clear 대상에 포함하지 않습니다.
- `x, --exclude-subdags`
    - SubDAG을 포함하지 않습니다.
- `f, --only-failed`
    - FAILED 상태의 작업에 대해서만 Clear를 수행합니다.
- `r, --only-running`
    - RUNNING 상태의 작업에 대해서만 Clear를 수행합니다.
- `s START_DATE, --start-date START_DATE`
    - Clear 할 시작 DagRun의 **data_interval 시작 시각**을 의미합니다.
    - END_DATE 파라미터와 마찬가지로 YYYY-MM-DD 형식을 지원하며, 만약 시간을 붙이고 싶다면 `YYYY-MM-DDTHH:mm:SS` 형식으로 나타낼 수 있습니다.
- `S SUBDIR, --subdir SUBDIR`
    - DAG 파일이 위치한 디렉토리를 지정해줄 수 있습니다. 기본값으로 `[AIRFLOW_HOME]/dags` 디렉토리가 참조됩니다.
- `t TASK_REGEX, --task-regex TASK_REGEX`
    - DAG 내의 특정 Task에 대해서만 Clear하고자 하는 경우 지정해주면 됩니다.
- `u, --upstream`
    - 클리어 대상이 되는 태스크의 상위 태스크(현재 태스크가 의존성을 갖고 있는 태스크)를 포함해서 클리어합니다.