---
title: "[Oracle] PL/SQL" #Article title.
date: 2023-12-26
category: [DB, ORACLE] #One, more categories or no at all.
tag: [db, oracle]
---

Oracle pipeline을 개발하다 보니  INSERT, UPDATE 등 익숙한 opernation외에 `PL/SQL`이라는 것을 발견해서 찾아보게 되었다. 

mysql에서는 이와 비슷한 'SQL/PSM' 이 있고 MsSQL의 트랜잭션과 비슷하지만 PL/SQL은 실제 프로그래밍 언어처럼 사용할 수 있다는 점에서 차이가 있다.

## PL/SQL?

PL/SQL은 Oracle에서 사용되는 절차적 언어로, SQL을 확장해서 더 다양한 기능을 제공한다. 

원래 SQL은 선언적 언어기 때문에 데이터를 어떻게 얻을지 기술하는 방식이었지만, PL/SQL은 절차적 언어이기 때문에 변수,조건문, 반복문 등을 이용해서 데이터를 어떻게 처리할지 정의할 수 있다. 그리고 선언부, 실행부, 예외처리부 와 같이 명확히 분리된 블럭 구조로 되어 있다. 


### PL/SQL 구조

- `DECLARE` : 선언부, 사용되는 변수나 상수를 선언하는 부분
- `BEGIN` : 실행부, 절차적 형식으로 SQL문을 실행할 수 있도록 제어문,반복문, 함수 등을 정의하는 부분
- `EXCEPTION` : 예외 처리부, PL/SQL 문이 실행되는 중에 발생할 수 있는 에러의 예외처리를 기술하는 부분
- 'END' : PL/SQL 종료

#### 변수 선언 방법

```sql
DECLARE
    변수명 데이터타입 [:= 초기값];
```

아래는 예시

```sql
-- 단순 변수 선언
DECLARE
    emp_count NUMBER;

-- 초기 값 할당
DECLARE
    emp_name VARCHAR2(100) := 'John Doe';

/* 복잡한 데이터 타입
start_date는 날짜 유형으로 오늘 날짜로 초기화되며, total_sales는 소수점 두 자리까지 표현 가능한 숫자 유형
*/
DECLARE
    start_date DATE := SYSDATE;
    total_sales NUMBER(10,2);

```

- 변수명은 문자로 시작해야하고 특수문자와 공백 포함 X
- 대소문자를 구분하지 않지만 변수명은 주로 소문자나 스네이크 케이스로 작성

##### SELECT INTO

변수에 `:=` 로 단일 값을 대입할 수도 있지만 SELECT INTO 문을 이용할 수도 있다.

```sql
SELECT 컬럼명
INTO 변수명
FROM 테이블명
WHERE 조건;

/* 단일 컬럼 값 대입
employee_id가 101인 직원의 salary를 조회하여 emp_salary 변수에 저장
*/
DECLARE
    emp_salary NUMBER;
BEGIN
    SELECT salary INTO emp_salary FROM employees WHERE employee_id = 101;
    DBMS_OUTPUT.PUT_LINE('Employee salary: ' || emp_salary);
END;

/* 여러 컬럼 값 대입
employee_id가 102인 직원의 이름(first_name)과 부서명(department_name)을 조회하여 각각 emp_name, emp_department 변수에 저장
*/
DECLARE
    emp_name VARCHAR2(100);
    emp_department VARCHAR2(50);
BEGIN
    SELECT first_name, department_name
    INTO emp_name, emp_department
    FROM employees JOIN departments USING (department_id)
    WHERE employee_id = 102;
    DBMS_OUTPUT.PUT_LINE('Employee: ' || emp_name || ', Department: ' || emp_department);
END;
```

SELECT INTO 문은 정확히 하나의 행을 반환해야 하고 그렇지 않으면 에러 발생

### PL/SQL 종류

#### 익명 블록

말 그대로 이름 없는 PL/SQL 문으로 일회성 스크립트나 테스트 목적으로 사용된다.

```sql
DECLARE
    emp_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO emp_count FROM employees WHERE department_id = 10;
    DBMS_OUTPUT.PUT_LINE('Department 10 has ' || emp_count || ' employees.');
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('No employees found in Department 10.');
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error occurred: ' || SQLERRM);
END;
```

부서 10의 직우너 수를 계산하고 결과를 출력하는 익명 블록이고 데이터를 찾을 수 없거나 다른 오류에 대한 예외처리도 포함되어 있다.

#### 저장 프로시저

 DB에 저장되며, 입력 매개변수를 받을 수 있고 리턴 값을 하나 이상 가질 수 있는 프로그램

```sql
CREATE OR REPLACE PROCEDURE UpdateSalary(emp_id IN NUMBER, new_salary IN NUMBER) IS
BEGIN
    UPDATE employees SET salary = new_salary WHERE employee_id = emp_id;
    IF SQL%ROWCOUNT = 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Employee not found');
    END IF;
END UpdateSalary;
```

이 프로시저는 특정 직원의 급여를 업데이트하고, 오류가 있을 시에 사용자 정의 오류를 발생시킨다.

#### 함수

 저장 프로시저와 유사하지만 항상 값을 반환해야하고 다른 SQL문 내에서 호출 가능

```sql
CREATE OR REPLACE FUNCTION GetEmployeeName(emp_id IN NUMBER) RETURN VARCHAR2 IS
    emp_name VARCHAR2(100);
BEGIN
    SELECT first_name || ' ' || last_name INTO emp_name FROM employees WHERE employee_id = emp_id;
    RETURN emp_name;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN NULL;
END GetEmployeeName;

```

이 함수는 주어진 직원 ID에 해당하는 직원의 이름을 반환하고 존재하지 않을 경우 NULL 반환

#### 트리거

데이터베이스 테이블 변화를 주는 특정 이벤트(INSERT,UPDATE 등)에 자동으로 반응해서 실행되는 PL/SQL

```sql
CREATE OR REPLACE TRIGGER LogEmployeeUpdate
AFTER UPDATE ON employees
FOR EACH ROW
BEGIN
    INSERT INTO employee_audit (employee_id, audit_date)
    VALUES (:NEW.employee_id, SYSDATE);
END LogEmployeeUpdate;

```

이 트리거는 employees 테이블에 대한 갱신 후 각 행에 대해 감사 로그를 기록한다.

#### 패키지

관련 프로시저, 함수, 변수, 타입 등을 그룹화한 컨테이너, 복잡한 프로그램을 모둘화하고 재사용 가능하게 해준다.


저장 프로시저와 함수는 비슷해보이지만 주요 차이점은 아래와 같다.

1. 저장 프로시저는 데이터 삽입, 삭제, 업데이트 등 복잡한 비즈니스 로직을 처리하고 함수는 계산을 수행하고 단일 값을 반환하는데 사용된다.
2. 저장 프로시저는 리턴 값이 있어도 되고 없어도 되지만, 함수는 반드시 리턴 값을 반환해야 한다.
3. 저장 프로시저는 독립적으로 호출되고, 함수는 SQL 쿼리 내에서 다른 함수와 함께 사용될수 있다. 