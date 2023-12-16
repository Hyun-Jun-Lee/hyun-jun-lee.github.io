---
title: "[Oracle] ReDo Log & ASH" #Article title.
date: 2023-12-14
category: [DB, ORACLE] #One, more categories or no at all.
tag: [db, oracle]
---

# Redo Log?

redo 로그는 oracle 에서 데이터 복구 작업에 주로 쓰이는 로그로 데이터베이스에 발생하는 모든 변경 사항을 저장하는 로그이다. (실제 데이터 변경 전 redo log에 먼저 기록한 후 변경)
oralce 데이터베이스의 각 인스턴스는 에러 발생 시 데이터베이스를 보호하기 위해 연관된 redo 로그를 가지고 있다.
redo log 관련 설정을 하거나 접근하려면 sys 권한이 필요하다.

- redo log buffer
    - DDL, DML에 의해 데이터베이스에 변경이 생겼을 때, 이런 정보를 모두 저장하는 메모리 영역
    - 해당 영역에 저장되는 정보들은 commit 되는 순간 redo log에 저장되며,  메모리에 저장된 데이터를 디스크에 저장한다.

# How Oracle Write To The Redo Log 

redo 로그는 다른 하나가 아카이빙 되는 동안 항상 쓰기에 사용할 수 있도록 최소 2개 이상의 파일로 구성된다.(ARCHIVELOG 모드가 설정된 경우)

LGWR이 redo 로그를 작성하는 역활을 하는데, 현재 redo 로그의 용량이 다차면 다음 redo 로그를 작성하고, 마지막 redo 로그를 다 작성하면 다시 처음의 redo 로그로 돌아와 작성하는
순환 방식으로 로그를 작성한다.

우리가 데이터에 변경을 주는 update, insert, delete 등의 쿼리를 실행시키고 commit을 하면 redo log buffer에 작성되어 있던 redo log record가 LGWR에 의해서 redo log 파일에 기록된다. 

## Redo Log Status

oracle은 한 번에 하나의 redo 로그 파일을 사용해서 redo log buffer에 작성된 redoce를 저장한다. 

- CURRENT : 현재 LGWR이 작성하고 있는 상태
- ACTIVE : 데이터 복구에 필요한 redo log 파일
- INACTIVE : 더 이상 데이터 복구에 필요하지 않은 redo log 파일

oracle에서 데이터 복구를 위해 사용되는 redo log file은 ACTIVE 상태인 redo log이다.

실제 회사에서 redo log 중복이 일어나는 경우가 있어서 확인해보았을 때, Inactive 상태의 redo 로그가 굉장히 많았다. 아마 로그마이너 설정 중 중복 설정되서 발생한 에러라고 예상된다.

그리고 Redo log 파일을 삭제하려고 할 때, 데이터베이스 상에서 INACTIVE 상태로 만들고 삭제를 해야하고, os 상에서 삭제해버리면 데이터 동기화에 오류가 생긴다고한다!

## Log Switch & Log Sequnce Numbers
Log switch는 LOWR이 하나의 redo 로그 파일에서의 쓰기를 중단하고, 다른 redo 로그 파일이 쓰기를 시작하는 지점이다. 즉, 현재 redo 로그 파일이 완전이 채워져 다름 redo 로그 파일로 이동하여 저장할 때 발생한다.

사용자가 설정한다면 아직 redo 로그파일이 다채워지지 않았어도 log switch가 발생하도록 할 수 있다. 

`SQL> alter system switch logfile;`

log switch가 발생하면 checkpoint가 생성되는데, 이 checkpoint는 commit된 데이터가 redo log file에 어디까지 저장되었는지 기록하는 역활을 한다. 
이 checkpoint에는 여러 종류가 있다.

- checkpoint 종류 정리 자료 (https://velog.io/@khyup0629/Oracle-Redo-Log-%EA%B4%80%EB%A6%AC%ED%95%98%EA%B8%B0)

# Archived Log

redo log에 대해 공부하다 보면 계속해서 Archived 모드와 Archived 로그에 대해 언급된다.

첫번째 redo log 파일이 모두 작성되면 다음 redo log 파일에 기록하는데, 이미 기록된 데이터가 있다면 새로운 recored들로 데이터가 변경된다.
그래서 redo log의 내용이 덮어씌워지기 전에 백업이 필요한데 그것이 archived redo log이다. 즉 archive 로그는  redo 로그의 복제본이라고 생각할 수 있다.

- archive mode 활성화 방법
    - https://jangkunstory.tistory.com/147

이를 활용하면 에러나 오류로 인해 잠시 DB 모니터링을 못하더라도, 그 시간동안의 DB 변경 사항을 확인 할 수 있기 때문에 Archived 모드를 활성화 하려고 시도했으나 대부분의 고객사에서 해당 설정을 활성화 시키면 DB 부하가 얼마나 생길지 알 수 없는 상황에서 리스크를 만들고 싶지 않아했다.


# Session 매핑 문제

redo 로그를 이용해서 DB의 모든 변경사항을 읽어올 때 SID를 이용해서 V$SESSION 테이블에서 해당 트랜잭션을 실행시킨 Session 정보(ip, host, user)를 join 해서 가져오는데, 이 데이터가 정확하지 않는 문제가 발생했다.

V$SESSION 테이블은 현재 활성화된 세션의 정보만 가지고 있기 때문에라고 생각해서 V$SESSION을 1초마다 select 해서 데이터를 저장하고, 여기서 sid로 매핑하는 방식으로 변경햇는데도 여전히 sessin 매칭이 맞지 않는 문제가 발생했다.

그렇게 조사하던 중 오라클 10g 부터 제공되는 ASH(Active_Session_History)를 발견했다. 

## ASH

ASH는 1초 간격으로 Active Session만 샘플링해서 buffer memory에 보관하고 있다가 가득차게 되면 AWR repository에 내려 쓴다. 자동으로 최근 데이터부터 조회되기 때문에 order by가 불필요하다. 그리고 접속이 끊긴 Session 정보도 저장되어 있기 때문에 좀 더 활용도가 높다.

- AWR(Automatic Workload Repository) : 자동으로 DB에 대한 통계 및 성능자료를 수집해 스냅샷으로 만들어 일정기간 보관하는 기능

```
select 
sample_id, sample_time                                                --① 샘플링이 일어난 시간과 샘플 ID
, session_id, session_serial#, user_id, xid                           --② 세션정보, User명, 트랜잭션ID
, sql_id, sql_child_number, sql_plan_hash_value                       --③ 수행 중 SQL 정보
, session_state                                                       --④ 현재 세션의 상태 정보. 'ON CPU' 또는 'WAITING'
, qc_instance_id, gc_session_id                                       --⑤ 병렬 Slave 세션일 때, 쿼리 코디네이터(QC) 정보를 찾을 수 있게 함
, blocking_session, blocking_session_serial#, blocking_session_status --⑥ 현재 세션의 진행을 막고 있는(=블로킹) 세션 정보
, event, event#, seq#, wait_class, wait_time, time_waited             --⑦ 현재 발생 중인 대기 이벤트 정보
, p1text, p1, p2text, p2, p3text, p3                                  --⑧ 현재 발생 중인 대기 이벤트의 파라미터 정보
, current_obj#, current_file#, current_block#                         --⑨ 해당 세션이 현재 참조하고 있는 오브젝트 정보 / V$session 뷰에  있는 row_wait_obj#, row_wait_file#, row_wait_block# 컬럼을 가져온 것
, program, module, action, client_id                                  --⑩ 애플리케이션 정보
from V$ACTIVE_SESSION_HISTORY;
```

- ASH 테이블 컬럼 정보 및 조회 방법 : https://richtech.tistory.com/121

ASH의 스냅샷들을 저장하고 있는 V$DBA_HIST_ASH_SNAPSHOT 테이블도 있다.

무료로 viewr 프로그램도 제공하고 있는 것 같은데, 실제 적용해본 후 마저 포스팅

### Refernce

- https://docs.oracle.com/
- https://velog.io/@khyup0629/Oracle-Redo-Log-%EA%B4%80%EB%A6%AC%ED%95%98%EA%B8%B0