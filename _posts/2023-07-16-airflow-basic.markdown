---
title: "Airflow 기본 구조" #Article title.
date: 2023-07-16
category: [AIRFLOW,AIRFLOW] #One, more categories or no at all.
tag: [airflow, python]
---

# Airflow 기본 구조

AirFlow는 Workflow를 정의하고 실행 가능한 플랫폼으로 반복 된 작업을 자동화 하기위해 사용한다.

주로 주기적으로 실행하는 ETL 작업에 주로 사용한다.

## AirFlow 기본 구조

<img width="626" alt="스크린샷 2023-07-10 오전 12 11 07" src="https://github.com/Hyun-Jun-Lee/hyun-jun-lee.github.io/assets/76996686/3601f911-e01b-48c8-a37b-9b4f5efab863">


- Scheduler : AirFlow의 DAG, Task 들 모니터링 및 실행 순서 관리
- Worker : 스케쥴러의 의해 전달 받은 DAG를 실행, 실제 코드가 실행되는 컨테이너 역활
- database : AirFlow에서 실행할 작업들에 관한 정보들 저장
- WebServer : AirFlow UI
- DAG : AirFlow 에서 실행할 작업들 파이프라인 형태로 저장
    
    ```python
    from airflow import DAG
    
    default_args = {
    	'owner' : hj,
    	'start_date' : datetime(2023,04,29)
    	'end_date' : datetime(2023,05,29)
    	# error 낫을 때 메일
    	'email' : [email]
    	# 재시도 횟수 및 딜레이
    	'retries' : 3
    	'retry_delay' : timedelta(minutes=3)
    	}
    
    test_dag = DAG(
    	id = "test_dag_v1",
    	schedule_interval = "0 * * * *",
    	defailt_args = default_args
    
    t1 = PythonPerator(dag=test_dag, task_id="test_task_1", params={})
    ```
    

Scheduler가 Dag directory에서 작업을 가져와서 Worker에서 실행하는 형태

## AirFlow 활용 예시

<img width="726" alt="스크린샷 2023-04-30 오전 1 21 43" src="https://github.com/Hyun-Jun-Lee/hyun-jun-lee.github.io/assets/76996686/bc73bc26-0292-4073-93e2-4dd65db716a5">

1. 데이터가 별로 크지 않다면 Redshift에 바로 저장
2. 데이터가 많을 경우 효율적인 방식
    1. Dag가 Production DB에서 원하는 데이터 읽어와서 csv, json 등과 같은 파일로 만들어서 S3 업로드
    2. S3에서 저장된 데이터를 Redshift로 한번에 Bulk INSERT
    
    <img width="729" alt="스크린샷 2023-04-30 오전 9 54 57" src="https://github.com/Hyun-Jun-Lee/hyun-jun-lee.github.io/assets/76996686/9864cb5f-307c-442f-b6f4-ab86d7719d4a">

## 데이터 파이프라인 생성시 고려할점

- Python Code로 작성하기 때문에 항상 버그 주의
- 작은 양의 데이터일 경우 매번 전체 복사해서 테이블 만드는게 편함 → Full Refresh
- BackFill을 위해 `catchup=True` 설정 잊지말기
- 데이터 파이프라인 입출력 문서화
- Airflow는 dag_dir_list_interval 옵션에 따라서 dags 폴더를 주기적으로 스캔하는데, Dag 폴더안에 있는 `.py` 파일 중 `from airflow import DAG` 와 같이 임포트  되어있으면 DAG로 인식한다.
    - 그래서 test 할 때 주의!

## Airflow WEB UI에서 디버깅 하는법

- 에러가 있는 task는 빨간색으로 표시되고, 해당 task 클릭하여 `Log` 확인

<img width="474" alt="스크린샷 2023-04-29 오후 11 24 07" src="https://github.com/Hyun-Jun-Lee/hyun-jun-lee.github.io/assets/76996686/29db3a96-d554-4d9d-9597-2fee4fa36718">

<img width="296" alt="스크린샷 2023-04-29 오후 11 25 00" src="https://github.com/Hyun-Jun-Lee/hyun-jun-lee.github.io/assets/76996686/8da4b774-b560-4b01-8a26-81227b7d1ac2">