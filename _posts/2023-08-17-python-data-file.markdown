---
title: "MySQL 데이터 복구" #Article title.
date: 2023-08-17
category: [DB, MYSQL] #One, more categories or no at all.
tag: [rollback, mysql]
---

# MySQL 데이터 복구하기

MySQL은 DDL, DML과 같이 DB에 변경을 주는 이벤트를 바이너리 형태로 기록하는 바이너리 로그가 있다
변경을 주는 이벤트만 남기기 때문에 ,show select 같은 쿼리문은 찾을 수 없다.

이전에 고객사에서 사원들 계정 관련 대규모 변경 건이 있었고, 해당 발생 사항들을 모두 DB에 저장을 해두고 있었다.
그런데 2,3일 후 보니 하드디스크 용량이 부족해서 서버가 다운 되었는데, 고작 몇일만에 30GB가 가득 찬 것!

어디가 가장 데이터가 많나 찾아보니 mysql 이 바이너리 로그를 저장하는 dir 이었다.
갑자기 몇 십만 건의 insert 가 생겨서 바이너리 로그도 급증한 것으로 예상된다.

이 때 처음으로 바이너리 로그에 대해 알게 됬는데 이 후 내가 실수로 데이터를 삭제해야해서 복구를 해야했다.

## 바이너리 로그 경로 설정

```
[mysqld]

log-bin=/data/base_name
```

기본 설정은 /data로 되어있고, base_name을 따로 지정해주지 않으면 시스템이름으로 바이너리 로그가 만들어진다.( ex.sdde22300001 ) 
나는 나중에 원할한 복구를 위해 backup dir을 따로 설정 해두었다

혹시 바이너리 로그 생성을 중지하고 싶다면 `SET sql_log_bin=OFF`, 근데 그냥 혹시를 위해서 켜두는게 좋을듯

## 바이너리 로그 check

`SHOW BINARYY LOGS;`

위 명령어로 현재 저장되어 있는 바이너리 로그를 모두 확인 가능

현재 작성중인 로그 파일을 현재시점에서 기록을 멈추고 새로운 복구 파일을 만드려면

`FLUSH LOGS;`

이렇게 해서 복구 시점을 명확히 할 수 있음


## 바이너리 로그 변환


바이너리로 되어 있기 때문에, 복구를 위해서는 텍스트로 변환해야한다.
mysql에서 이 때 사용할 수 있는 `mysqlbinlog` 라는 툴을 지원한다.

`mysqlbinlog /var/lib/mysql/binlog.0* > backup_log.sql`

모든 바이너리 로그가 아니라 원하는 것들만 변환하고 싶다면

`mysqlbinlog /var/lib/mysql/binlog.03 /var/lib/mysql/binlog.06 > backup_log.sql`

이렇게 변환 후 변한 된 파일을 확인해보니 내가 실수로 삭제한 truncate 가 있더라!
나는 삭제한 데이터가 3개 정도라 다시 보면서 내가 직접 생성해줫지만 아래 명령어와 같이 할 수도 있다.

`mysql -u root -p -f < backup_log.sql`

`-f`옵션으로 강제해줘야함! cuz 안에 create table과 같은 쿼리도 있기 때문!
