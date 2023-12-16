---
title: "WINDOW func" #Article title.
date: 2023-08-07
category: [DB, SQL] #One, more categories or no at all.
tag: [sql,db]
---

# WINDOW func

WINDOW 함수란 행과 행간의 비교, 연산을 정의하기 위한 함수이고 분석함수, 순위함수 라고도 부른다.

- 기본 문법

```sql
SELECT WINDOW_FUNCTION (ARGUMENTS) OVER([PARTITION BY 컬럼] [ORDER BY 컬럼] [WINDOWING 절])
FROM 테이블명;
```

## WINDOW function 종류

- 그룹 내 순위 관련 함수 : RANK, DENSE_RANK, ROW_NUMBER
- 그룹 내 집계 관련 함수 : SUM, MAX, MIN, AVG, COUNT
- 그룹 내 행 순서 관련 함수 : FIRST_VALUE, LAST_VALUE, LAG, LEAD(oracle)
- 그룹 내 비율 관련 함수 : CUME_DIST, PERCENT_RANK, NTILE, RATIO_TO_REPORT

### 1. 그룹 내 순위 관련 함수

- RANK
    - ORDER BY를 포함한 쿼리문에서 특정 컬럼의 순위를 구하는 함수
    - 특점 범위(PARTITION) 내에서 순위를 구할 수도 있고 전체 데이터에서 순위를 구할수도 있음
    - 동일한 값에 대한 순위는 1,1,3,4 와 같이 중간 순위를 비워둠
    
    ```sql
    SELECT JOB, ENAME, SAL, 
    			 -- JOB 별로 SAL 높은 순 정렬
           RANK() OVER (PARTITION BY JOB ORDER BY SAL DESC) JOB_RANK 
      FROM EMP;
    
    JOB       ENAME             SAL   JOB_RANK
    --------- ---------- ---------- ----------
    ANALYST   FORD             3000          1
    ANALYST   SCOTT            3000          1
    CLERK     MILLER           1300          1
    CLERK     ADAMS            1300          1
    CLERK     JAMES             950          3
    CLERK     SMITH             800          4
    ```
    
- DENSE_RANK
    - RANK 함수와 작동 방식이 같으나 동일한 값에 대해 같은 순위 부여
    
    ```sql
    SELECT JOB, ENAME, SAL, 
           DENSE_RANK() OVER (PARTITION BY JOB ORDER BY SAL DESC) JOB_RANK 
      FROM EMP;
    
    JOB       ENAME             SAL   JOB_RANK
    --------- ---------- ---------- ----------
    ANALYST   FORD             3000          1
    ANALYST   SCOTT            3000          1
    CLERK     MILLER           1300          1
    CLERK     ADAMS            1300          1
    CLERK     JAMES             950          2
    CLERK     SMITH             800          3
    ```
    

- ROW_NUMBER
    - RANK, DENSE_RANK 와 달리 동일한 값에 고유한 순위를 부여함
    - 동일한 값에 순서를 정하고 싶다면 ORDER BY 같이 활용
    
    ```sql
    SELECT JOB, ENAME, SAL, 
           ROW_NUMBER() OVER (PARTITION BY JOB ORDER BY SAL DESC) JOB_RANK 
      FROM EMP;
    
    JOB       ENAME             SAL   JOB_RANK
    --------- ---------- ---------- ----------
    ANALYST   FORD             3000          1
    ANALYST   SCOTT            3000          2
    CLERK     MILLER           1300          1
    CLERK     ADAMS            1300          2
    CLERK     JAMES             950          3
    CLERK     SMITH             800          4
    ```
    

### 2. 그룹 내 행 순서 관련 함수

- FIRST_VALUE, LAST_VALUE ( sql server X)
    - 파티션별로 가장 먼저 나온 값 / 나중에 나온 값
    - 동일한 값에 대한 처리 없이 첫번째 값 리턴하기 때문에 따로 처리해줘야함
    
    ```sql
    -- 부서별 연봉 높은 순으로 정렬 후 가장 먼저 나온 값 출력
    SELECT  DEPTNO, ENAME, SAL,
            FIRST_VALUE(ENAME) OVER (PARTITION BY DEPTNO ORDER BY SAL DESC ENAME ASC
            ROWS UNBOUNDED PRECEDING) DEPT_RICH
    FROM    EMP
    ```
    
    ```sql
    -- 부서별 연봉 높은 순으로 정렬 후 가장 나중에 나온 값 출력
    SELECT  DEPTNO, ENAME, SAL,
            LAST_VALUE(ENAME) OVER (PARTITION BY DEPTNO ORDER BY SAL DESC, ENAME ASC
            ROWS UNBOUNDED PRECEDING) DEPT_RICH
    FROM    EMP ;
    ```
    

- LAG, LELAD ( sql server X)
    - 이전/이후 몇 번째 행의 값을 가져오는 함수
    - 두번째 인자는 몇 번째 앞의 행을 가져올지 결정(default=1)
    - 세번째 인자는 두번째 인자가 없을 경우의 default 값을 지정
    
    ```sql
    --HIREDATE를 기준으로 정렬하고 본인보다 입사일자가 하나 더 앞선 사원의 급여를 출력
    SELECT ENAME, HIREDATE, SAL
         , LAG(SAL) OVER (ORDER BY HIREDATE) as PREV_SAL 
      FROM EMP 
     WHERE JOB = 'SALESMAN';
    
    ENAME      HIREDATE         SAL   PREV_SAL
    ---------- --------- ---------- ----------
    ALLEN      20-FEB-81       1600
    WARD       22-FEB-81       1250       1600
    TURNER     08-SEP-81       1500       1250
    MARTIN     28-SEP-81       1250       1500
    
    --HIREDATE를 기준으로 정렬하고 본인보다 입사일자가 두 개 더 앞선 사원의 급여를 출력
    --두 개 더 앞선 사원이 없을 경우 0을 출력
    SELECT ENAME, HIREDATE, SAL
         , LAG(SAL, 2, 0) OVER (ORDER BY HIREDATE) as PREV_SAL 
      FROM EMP 
     WHERE JOB = 'SALESMAN' ;
    
    ENAME      HIREDATE         SAL   PREV_SAL
    ---------- --------- ---------- ----------
    ALLEN      20-FEB-81       1600          0
    WARD       22-FEB-81       1250          0
    TURNER     08-SEP-81       1500       1600
    MARTIN     28-SEP-81       1250       1250
    ```
    
    ```sql
    --HIREDATE를 기준으로 정렬하고 본인보다 HIREDATE가 하나 더 뒤인 날짜를 출력
    --없는 경우 NULL
    SELECT ENAME, HIREDATE
         , LEAD(HIREDATE, 1) OVER (ORDER BY HIREDATE) as "NEXTHIRED" 
      FROM EMP;
    
    ENAME      HIREDATE  NEXTHIRED
    ---------- --------- ---------
    SMITH      17-DEC-80 20-FEB-81
    ALLEN      20-FEB-81 22-FEB-81
    WARD       22-FEB-81 02-APR-81
    JONES      02-APR-81 01-MAY-81
    BLAKE      01-MAY-81 09-JUN-81
    CLARK      09-JUN-81
    ```
    

### 3. 그룹 내 비율 함수

- RATIO_TO_REPORT (SQL server X)
    - 파티션 내 전체 SUM 값에 대한 행별 컬럼 값의 백분율을 소수점으로 출력
    - 각 비율의 합은 1
    
    ```sql
    --전체 급여에서 각각이 차지하는 비율 출력
    SELECT ENAME, SAL
         , ROUND(RATIO_TO_REPORT(SAL) OVER (), 2) as R_R 
      FROM EMP 
     WHERE JOB = 'SALESMAN'; 
    
    ENAME             SAL        R_R
    ---------- ---------- ----------
    ALLEN            1600        .29
    WARD             1250        .22
    MARTIN           1250        .22
    TURNER           1500        .27
    ```
    
- PERCENT_RANK
    - 파티션 별로 가장 먼저 나오는 값을 0, 나중에 나오는값을 1로 해서 행의 순서별 백분율
    
    ```sql
    -- 같은 부서 사람들 중 본인 급여가 몇 분위에 있는지 확인
    SELECT DEPTNO, ENAME, SAL
         , PERCENT_RANK() OVER (PARTITION BY DEPTNO ORDER BY SAL DESC) as P_R 
      FROM EMP; 
    
        DEPTNO ENAME             SAL        P_R
    ---------- ---------- ---------- ----------
            10 KING             5000          0
            10 CLARK            2450         .5
            10 MILLER           1300          1
            20 SCOTT            3000          0
            20 FORD             3000          0
            20 JONES            2975         .5
            20 ADAMS            1100        .75
            20 SMITH             800          1
            30 BLAKE            2850          0
            30 ALLEN            1600         .2
            30 TURNER           1500         .4
            30 MARTIN           1250         .6
            30 WARD             1250         .6
            30 JAMES             950          1
    ```