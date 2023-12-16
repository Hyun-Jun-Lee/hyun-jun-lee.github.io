---
title: "execution_date, start_date, date_interval_start/end" #Article title.
date: 2023-07-19
category: [AIRFLOW,AIRFLOW] #One, more categories or no at all.
tag: [airflow, python]
---

- start_date : Dag가 시작되는 기준 시점
    - `start_date='2023-04-29'` 이고 하루에 한번 실행되는 DAG라면 2023-04-30 일에 처음 실행됨
    - 즉 `start_date`는 시작 날짜가 아니라 읽어와야 하는 날짜로 DAG가 시작되는 기준 시점
    - 현재 시간이 `start_date` 보다 이전 이면 DAG가 실행되지 않는다
        - 하루에 한번 실행되는 DAG일 경우, 생성한 DAG를 당장 실행해보고 싶으면 `start_date` 를 하루 전으로 설정해야한다

- execution_date : task 실행시점이 아닌, Time Window 기준 시점
    - 실제 실행 날짜가 아니고, 처리할 데이터의 시간 범위중 가장 빠른 시점으로 결정되는 값으로 DAG가 실행 될 때 마다 바뀜
    - 하루에 한번 실행되는 DAG이고 `start_date='2023-04-29'` 일 때,
    
    | 작업 실행 시점 | 데이터 처리 범위 | execution_date |
    | --- | --- | --- |
    | 04/30 00:00:00 | 04/29 00:00:00 ~ 04/29 23:59:59 | 04/29 00:00:00 |
    | 05/01 00:00:00 | 04/30 00:00:00 ~ 04/30 23:59:59 | 04/30 00:00:00 |
    | 05/02 00:00:00 | 05/01 00:00:00 ~ 05/01 23:59:59 | 05/01 00:00:00 |

execution 이라는 단어가 혼동을 주기 때문에, AirFlow 2.2 버전 이후 부터는 

`data_interval_start` , `data_interval_end` 

라는 용어를 사용한다.

| 작업 실행 시점 | 데이터 처리 범위 | data_interval_start | data_interval_end |
| --- | --- | --- | --- |
| 04/30 00:00:00 | 04/29 00:00:00 ~ 04/29 23:59:59 | 04/29 00:00:00 | 04/30 00:00:00  |
| 05/01 00:00:00 | 04/30 00:00:00 ~ 04/30 23:59:59 | 04/30 00:00:00 | 05/01 00:00:00 |
| 05/02 00:00:00 | 05/01 00:00:00 ~ 05/01 23:59:59 | 05/01 00:00:00 | 05/02 00:00:00 |

<img width="642" alt="스크린샷 2023-07-10 오전 1 06 42" src="https://github.com/Hyun-Jun-Lee/hyun-jun-lee.github.io/assets/76996686/b4a4ca4f-999e-419d-90ef-84ea784ba2b0">