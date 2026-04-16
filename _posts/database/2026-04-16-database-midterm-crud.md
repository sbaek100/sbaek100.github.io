---
layout: post
title: 2026-1학기 중간고사준비(재직자반) - CRUD 및 AND/OR 실습 20문제
date: 2026-04-16 10:00:00 +0900
categories:
  - 강의
  - 데이터베이스보안
tags:
  - mysql
  - 중간고사
  - crud
  - insert
  - select
  - update
  - delete
  - where
  - and
  - or
  - beginner
mermaid: false
pin: true
description: 재직자 과정 중간고사 대비 CRUD(INSERT, SELECT, UPDATE, DELETE)와 AND/OR 조건 필터링을 집중 연습하는 20문제. 직접 DB를 만들고, 데이터를 넣고, 조회·수정·삭제하는 전 과정을 단계별로 익힌다.
---

# 2026-1학기 중간고사(재직자반)


---

## 시험 전 준비 사항 체크리스트

시험 전에 아래 항목을 반드시 확인한다.

| 순번 | 확인 항목 | 비고 |
|------|----------|------|
| 1 | MySQL Workbench 설치 및 정상 접속 확인 | localhost / root 계정 |
| 2 | `CREATE DATABASE` → `USE` → `CREATE TABLE` 흐름 숙지 | 매 시험 시작 시 필요 |
| 3 | `INSERT INTO` 문법 — VALUES 순서와 컬럼 지정 방식 모두 연습 | 컬럼 생략 시 전체 컬럼 순서대로 |
| 4 | `SELECT * FROM` 기본 조회 | 전체 조회, 특정 컬럼 조회 |
| 5 | `WHERE` 절 — 비교 연산자(`=`, `>=`, `<`, `<>`) 사용법 | 문자열은 작은따옴표 필수 |
| 6 | `AND` / `OR` 조건 조합 및 괄호 우선순위 | 괄호 없으면 AND가 OR보다 먼저 |
| 7 | `UPDATE ... SET ... WHERE` 문법 | WHERE 빠뜨리면 전체 행 변경 주의 |
| 8 | `DELETE FROM ... WHERE` 문법 | WHERE 빠뜨리면 전체 행 삭제 주의 |
| 9 | `IN`, `BETWEEN`, `LIKE`, `NOT` 키워드 | OR 여러 개 대신 IN 사용 |
| 10 | `ORDER BY` 정렬 — ASC(오름차순), DESC(내림차순) | 기본값은 ASC |
| 11 | Safe Update Mode 확인 | Workbench 설정에서 OFF 또는 PK 조건 사용 |
| 12 | 세미콜론(`;`) 빠뜨리지 않기 | 문장 끝에 반드시 세미콜론 |

> **핵심 암기:** `UPDATE`와 `DELETE`에서 `WHERE`를 빠뜨리면 **전체 행**이 변경되거나 삭제된다. 항상 `WHERE` 조건을 먼저 확인하는 습관을 들인다.
{: .prompt-warning }

---

## 사전 실습 1. 데이터베이스 생성

MySQL Workbench Query 창에 아래 SQL을 입력하고 실행(`Ctrl+Enter` 또는 번개 버튼)한다.

```sql
DROP DATABASE IF EXISTS crud_db;
CREATE DATABASE crud_db;
USE crud_db;
```

> `USE crud_db;`를 반드시 실행해야 이후 쿼리가 이 데이터베이스에 적용된다.
{: .prompt-tip }

---

## 사전 실습 2. 테이블 생성 및 데이터 입력

### 부서 테이블 생성

```sql
CREATE TABLE departments (
    dept_id   INT         PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL,
    location  VARCHAR(50)
);
```

### 직원 테이블 생성

```sql
CREATE TABLE employees (
    emp_id    INT          PRIMARY KEY AUTO_INCREMENT,
    name      VARCHAR(50)  NOT NULL,
    age       INT          NOT NULL,
    city      VARCHAR(50),
    salary    INT,
    dept_id   INT,
    hire_year INT
);
```

### 데이터 입력

```sql
INSERT INTO departments VALUES (10, '개발팀', '서울');
INSERT INTO departments VALUES (20, '영업팀', '부산');
INSERT INTO departments VALUES (30, '인사팀', '서울');
INSERT INTO departments VALUES (40, '기획팀', '대전');

INSERT INTO employees (name, age, city, salary, dept_id, hire_year) VALUES
('김철수', 35, '서울',  4500000, 10, 2018),
('이영희', 28, '부산',  3200000, 20, 2021),
('박민수', 42, '서울',  5800000, 10, 2015),
('최지영', 31, '인천',  3800000, 30, 2020),
('정대호', 27, '서울',  3000000, 10, 2023),
('한소연', 38, '부산',  4200000, 20, 2017),
('오준혁', 45, '대전',  6000000, 40, 2012),
('윤미래', 29, '서울',  3500000, 30, 2022),
('임태균', 33, '인천',  4000000, 20, 2019),
('강하늘', 26, '부산',  2800000, 40, 2024);
```

### 입력 확인

```sql
SELECT * FROM departments;
SELECT * FROM employees;
```

**departments 결과:**

| dept_id | dept_name | location |
|---------|-----------|----------|
| 10 | 개발팀 | 서울 |
| 20 | 영업팀 | 부산 |
| 30 | 인사팀 | 서울 |
| 40 | 기획팀 | 대전 |

**employees 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 1 | 김철수 | 35 | 서울 | 4500000 | 10 | 2018 |
| 2 | 이영희 | 28 | 부산 | 3200000 | 20 | 2021 |
| 3 | 박민수 | 42 | 서울 | 5800000 | 10 | 2015 |
| 4 | 최지영 | 31 | 인천 | 3800000 | 30 | 2020 |
| 5 | 정대호 | 27 | 서울 | 3000000 | 10 | 2023 |
| 6 | 한소연 | 38 | 부산 | 4200000 | 20 | 2017 |
| 7 | 오준혁 | 45 | 대전 | 6000000 | 40 | 2012 |
| 8 | 윤미래 | 29 | 서울 | 3500000 | 30 | 2022 |
| 9 | 임태균 | 33 | 인천 | 4000000 | 20 | 2019 |
| 10 | 강하늘 | 26 | 부산 | 2800000 | 40 | 2024 |

---

## 문제 1 — SELECT 전체 조회

`employees` 테이블의 모든 컬럼과 모든 행을 조회하라.

**힌트:** `SELECT *`를 사용한다.

**답:**

```sql
SELECT * FROM employees;
```

**예상 결과:** 위 사전 실습의 employees 결과와 동일 (10행)

---

## 문제 2 — 특정 컬럼 조회

`employees` 테이블에서 `name`과 `salary` 컬럼만 조회하라.

**힌트:** SELECT 뒤에 원하는 컬럼 이름을 쉼표로 나열한다.

**답:**

```sql
SELECT name, salary FROM employees;
```

**예상 결과:**

| name | salary |
|------|--------|
| 김철수 | 4500000 |
| 이영희 | 3200000 |
| 박민수 | 5800000 |
| 최지영 | 3800000 |
| 정대호 | 3000000 |
| 한소연 | 4200000 |
| 오준혁 | 6000000 |
| 윤미래 | 3500000 |
| 임태균 | 4000000 |
| 강하늘 | 2800000 |

---

## 문제 3 — WHERE 단일 조건

`employees` 테이블에서 `city`가 `'서울'`인 직원만 조회하라.

**힌트:** `WHERE 컬럼 = 값` 형식을 사용한다. 문자열은 작은따옴표로 감싼다.

**답:**

```sql
SELECT * FROM employees WHERE city = '서울';
```

**예상 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 1 | 김철수 | 35 | 서울 | 4500000 | 10 | 2018 |
| 3 | 박민수 | 42 | 서울 | 5800000 | 10 | 2015 |
| 5 | 정대호 | 27 | 서울 | 3000000 | 10 | 2023 |
| 8 | 윤미래 | 29 | 서울 | 3500000 | 30 | 2022 |

---

## 문제 4 — 비교 연산자

`employees` 테이블에서 `salary`가 `4000000` 이상인 직원의 이름과 급여를 조회하라.

**힌트:** 비교 연산자 `>=`를 사용한다.

**답:**

```sql
SELECT name, salary FROM employees WHERE salary >= 4000000;
```

**예상 결과:**

| name | salary |
|------|--------|
| 김철수 | 4500000 |
| 박민수 | 5800000 |
| 한소연 | 4200000 |
| 오준혁 | 6000000 |
| 임태균 | 4000000 |

---

## 문제 5 — AND 조건

`employees` 테이블에서 `city`가 `'서울'`이고 `salary`가 `4000000` 이상인 직원을 조회하라.

**힌트:** 두 조건을 모두 만족해야 하므로 `AND`를 사용한다.

**답:**

```sql
SELECT * FROM employees
WHERE city = '서울' AND salary >= 4000000;
```

**예상 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 1 | 김철수 | 35 | 서울 | 4500000 | 10 | 2018 |
| 3 | 박민수 | 42 | 서울 | 5800000 | 10 | 2015 |

---

## 문제 6 — OR 조건

`employees` 테이블에서 `city`가 `'부산'`이거나 `'인천'`인 직원을 조회하라.

**힌트:** 조건 중 하나만 만족해도 되므로 `OR`을 사용한다.

**답:**

```sql
SELECT * FROM employees
WHERE city = '부산' OR city = '인천';
```

**예상 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 2 | 이영희 | 28 | 부산 | 3200000 | 20 | 2021 |
| 4 | 최지영 | 31 | 인천 | 3800000 | 30 | 2020 |
| 6 | 한소연 | 38 | 부산 | 4200000 | 20 | 2017 |
| 9 | 임태균 | 33 | 인천 | 4000000 | 20 | 2019 |
| 10 | 강하늘 | 26 | 부산 | 2800000 | 40 | 2024 |

---

## 문제 7 — AND + OR 괄호 조합

`employees` 테이블에서 `dept_id`가 `10`이면서, `salary`가 `4000000` 이상이거나 `hire_year`가 `2020` 이후인 직원을 조회하라.

**힌트:** OR 조건을 괄호로 묶어야 의도대로 동작한다. 괄호 없이 `AND`와 `OR`을 섞으면 `AND`가 먼저 실행된다.

**답:**

```sql
SELECT * FROM employees
WHERE dept_id = 10
  AND (salary >= 4000000 OR hire_year >= 2020);
```

**예상 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 1 | 김철수 | 35 | 서울 | 4500000 | 10 | 2018 |
| 3 | 박민수 | 42 | 서울 | 5800000 | 10 | 2015 |
| 5 | 정대호 | 27 | 서울 | 3000000 | 10 | 2023 |

---

## 문제 8 — INSERT 단일 행 추가

`employees` 테이블에 아래 직원을 추가하라.

| name | age | city | salary | dept_id | hire_year |
|------|-----|------|--------|---------|-----------|
| 송지원 | 30 | 대전 | 3600000 | 40 | 2021 |

추가 후 `SELECT`로 결과를 확인하라.

**힌트:** `INSERT INTO 테이블 (컬럼목록) VALUES (값목록)` 형식을 사용한다.

**답:**

```sql
INSERT INTO employees (name, age, city, salary, dept_id, hire_year)
VALUES ('송지원', 30, '대전', 3600000, 40, 2021);

SELECT * FROM employees WHERE name = '송지원';
```

**예상 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 11 | 송지원 | 30 | 대전 | 3600000 | 40 | 2021 |

---

## 문제 9 — INSERT 여러 행 추가

`employees` 테이블에 아래 두 직원을 **한 번의 INSERT 문**으로 추가하라.

| name | age | city | salary | dept_id | hire_year |
|------|-----|------|--------|---------|-----------|
| 배수진 | 34 | 서울 | 4100000 | 30 | 2019 |
| 조민호 | 40 | 부산 | 4800000 | 20 | 2016 |

**힌트:** VALUES 뒤에 괄호를 쉼표로 연결하면 여러 행을 한 번에 넣을 수 있다.

**답:**

```sql
INSERT INTO employees (name, age, city, salary, dept_id, hire_year) VALUES
('배수진', 34, '서울', 4100000, 30, 2019),
('조민호', 40, '부산', 4800000, 20, 2016);

SELECT * FROM employees WHERE name IN ('배수진', '조민호');
```

**예상 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 12 | 배수진 | 34 | 서울 | 4100000 | 30 | 2019 |
| 13 | 조민호 | 40 | 부산 | 4800000 | 20 | 2016 |

---

## 문제 10 — UPDATE 단일 행 수정

`employees` 테이블에서 `강하늘`의 `salary`를 `3200000`으로 변경하라. 변경 후 SELECT로 확인하라.

**힌트:** `UPDATE 테이블 SET 컬럼=값 WHERE 조건`을 사용한다.

**답:**

```sql
UPDATE employees SET salary = 3200000 WHERE name = '강하늘';

SELECT * FROM employees WHERE name = '강하늘';
```

**예상 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 10 | 강하늘 | 26 | 부산 | 3200000 | 40 | 2024 |

> Safe Update Mode가 켜져 있으면 PK를 조건으로 써야 실행된다. `WHERE emp_id = 10`으로 변경하거나, Workbench 메뉴 `Edit > Preferences > SQL Editor`에서 Safe Updates 체크를 해제한다.
{: .prompt-warning }

---

## 문제 11 — UPDATE 여러 행 수정

`employees` 테이블에서 `dept_id`가 `20`인 직원 전체의 `salary`를 `10%` 인상하라. 변경 후 SELECT로 확인하라.

**힌트:** `SET salary = salary * 1.1`처럼 기존 값을 기반으로 계산할 수 있다.

**답:**

```sql
UPDATE employees SET salary = salary * 1.1 WHERE dept_id = 20;

SELECT * FROM employees WHERE dept_id = 20;
```

**예상 결과:** (문제 9에서 조민호를 추가한 뒤 기준)

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 2 | 이영희 | 28 | 부산 | 3520000 | 20 | 2021 |
| 6 | 한소연 | 38 | 부산 | 4620000 | 20 | 2017 |
| 9 | 임태균 | 33 | 인천 | 4400000 | 20 | 2019 |
| 13 | 조민호 | 40 | 부산 | 5280000 | 20 | 2016 |

---

## 문제 12 — DELETE 단일 행 삭제

`employees` 테이블에서 `송지원`을 삭제하라. 삭제 전 SELECT로 대상을 먼저 확인하라.

**힌트:** 삭제 전에 반드시 `SELECT * FROM employees WHERE name = '송지원';`으로 대상을 확인하는 습관을 들인다.

**답:**

```sql
-- 삭제 대상 먼저 확인
SELECT * FROM employees WHERE name = '송지원';

-- 삭제 실행
DELETE FROM employees WHERE name = '송지원';

-- 삭제 확인
SELECT * FROM employees WHERE name = '송지원';
```

**예상 결과 (삭제 후 조회):** 결과 없음 (0행)

---

## 문제 13 — AND 다중 조건

`employees` 테이블에서 `city`가 `'서울'`이고, `age`가 `30` 이상이고, `dept_id`가 `10`인 직원을 조회하라.

**힌트:** AND를 여러 번 이어서 3개 조건을 모두 만족시킨다.

**답:**

```sql
SELECT * FROM employees
WHERE city = '서울' AND age >= 30 AND dept_id = 10;
```

**예상 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 1 | 김철수 | 35 | 서울 | 4500000 | 10 | 2018 |
| 3 | 박민수 | 42 | 서울 | 5800000 | 10 | 2015 |

---

## 문제 14 — OR 다중 조건과 IN

`employees` 테이블에서 `dept_id`가 `10`, `20`, `30` 중 하나인 직원을 조회하라. OR 방식과 IN 방식을 모두 작성하라.

**힌트:** `IN`은 여러 값을 한꺼번에 비교할 때 `OR`보다 간결하다.

**답:**

```sql
-- OR 사용
SELECT * FROM employees
WHERE dept_id = 10 OR dept_id = 20 OR dept_id = 30;

-- IN 사용 (같은 결과)
SELECT * FROM employees
WHERE dept_id IN (10, 20, 30);
```

**예상 결과:** dept_id가 40인 직원(오준혁, 강하늘)을 제외한 나머지 전원

---

## 문제 15 — AND + OR 복합 (괄호 필수)

`employees` 테이블에서 `city`가 `'서울'`이면서 `dept_id`가 `10`인 직원, 또는 `city`가 `'부산'`이면서 `dept_id`가 `20`인 직원을 조회하라.

**힌트:** 두 묶음을 각각 괄호로 감싸고 OR로 연결한다.

**답:**

```sql
SELECT * FROM employees
WHERE (city = '서울' AND dept_id = 10)
   OR (city = '부산' AND dept_id = 20);
```

**예상 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 1 | 김철수 | 35 | 서울 | 4500000 | 10 | 2018 |
| 3 | 박민수 | 42 | 서울 | 5800000 | 10 | 2015 |
| 5 | 정대호 | 27 | 서울 | 3000000 | 10 | 2023 |
| 2 | 이영희 | 28 | 부산 | 3520000 | 20 | 2021 |
| 6 | 한소연 | 38 | 부산 | 4620000 | 20 | 2017 |
| 13 | 조민호 | 40 | 부산 | 5280000 | 20 | 2016 |

---

## 문제 16 — BETWEEN 범위 조건

`employees` 테이블에서 `salary`가 `3000000` 이상 `4500000` 이하인 직원을 조회하라.

**힌트:** `BETWEEN 최솟값 AND 최댓값`은 경계값을 포함한다.

**답:**

```sql
SELECT * FROM employees
WHERE salary BETWEEN 3000000 AND 4500000;
```

**예상 결과:** salary가 3000000~4500000 범위에 포함되는 직원 (김철수, 이영희(변경 후 3520000), 최지영, 정대호, 윤미래, 임태균(변경 후 4400000), 강하늘(변경 후 3200000), 배수진, ...)

> 앞선 UPDATE 문제의 실행 여부에 따라 결과가 달라질 수 있다.
{: .prompt-info }

---

## 문제 17 — LIKE 패턴 검색

`employees` 테이블에서 이름이 `'김'`으로 시작하는 직원을 조회하라.

**힌트:** `LIKE '김%'`를 사용한다. `%`는 뒤에 어떤 문자가 와도 된다는 뜻이다.

**답:**

```sql
SELECT * FROM employees WHERE name LIKE '김%';
```

**예상 결과:**

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 1 | 김철수 | 35 | 서울 | 4500000 | 10 | 2018 |

---

## 문제 18 — 복합 조건 + ORDER BY

`employees` 테이블에서 (`dept_id`가 `10` 또는 `20`)이면서 `hire_year`가 `2019` 이후인 직원을 조회하되, `salary` 기준 내림차순으로 정렬하라.

**힌트:** `WHERE` + 괄호 조건 + `ORDER BY 컬럼 DESC`를 조합한다.

**답:**

```sql
SELECT * FROM employees
WHERE dept_id IN (10, 20) AND hire_year >= 2019
ORDER BY salary DESC;
```

**예상 결과:** (문제 11 UPDATE 반영 기준)

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 9 | 임태균 | 33 | 인천 | 4400000 | 20 | 2019 |
| 2 | 이영희 | 28 | 부산 | 3520000 | 20 | 2021 |
| 5 | 정대호 | 27 | 서울 | 3000000 | 10 | 2023 |

---

## 문제 19 — DELETE 조건 삭제

`employees` 테이블에서 `hire_year`가 `2024`이고 `salary`가 `3000000` 미만인 직원을 삭제하라. 삭제 전·후 SELECT로 확인하라.

**힌트:** AND 조건으로 삭제 대상을 정확히 지정한다.

**답:**

```sql
-- 삭제 대상 먼저 확인
SELECT * FROM employees
WHERE hire_year = 2024 AND salary < 3000000;

-- 삭제 실행
DELETE FROM employees
WHERE hire_year = 2024 AND salary < 3000000;

-- 삭제 후 확인
SELECT * FROM employees ORDER BY emp_id;
```

**예상 결과 (삭제 대상):** (문제 10 UPDATE 반영 기준)

| emp_id | name | age | city | salary | dept_id | hire_year |
|--------|------|-----|------|--------|---------|-----------|
| 10 | 강하늘 | 26 | 부산 | 3200000 | 40 | 2024 |

> 문제 10에서 강하늘의 salary를 3200000으로 변경했다면 삭제 대상이 아니다(3000000 이상이므로). 문제를 순서대로 풀었는지에 따라 결과가 달라질 수 있다.
{: .prompt-info }

---

## 문제 20 — 종합 문제: 테이블 생성 + INSERT + 조건 조회

아래 조건으로 `projects` 테이블을 새로 만들고, 데이터를 3건 입력한 뒤, 조건 조회를 수행하라.

### (1) 테이블 생성

| 컬럼 | 자료형 | 제약조건 |
|------|--------|---------|
| project_id | INT | PRIMARY KEY, AUTO_INCREMENT |
| project_name | VARCHAR(100) | NOT NULL |
| dept_id | INT | NOT NULL |
| budget | INT | DEFAULT 0 |
| status | VARCHAR(20) | |

### (2) 데이터 입력

| project_name | dept_id | budget | status |
|-------------|---------|--------|--------|
| 홈페이지 리뉴얼 | 10 | 5000000 | 진행중 |
| 신규 영업 시스템 | 20 | 8000000 | 계획 |
| 인사 평가 자동화 | 30 | 3000000 | 진행중 |

### (3) 조건 조회

`status`가 `'진행중'`이고 `budget`이 `4000000` 이상인 프로젝트를 조회하라.

**답:**

```sql
-- (1) 테이블 생성
CREATE TABLE projects (
    project_id   INT          PRIMARY KEY AUTO_INCREMENT,
    project_name VARCHAR(100) NOT NULL,
    dept_id      INT          NOT NULL,
    budget       INT          DEFAULT 0,
    status       VARCHAR(20)
);

-- (2) 데이터 입력
INSERT INTO projects (project_name, dept_id, budget, status) VALUES
('홈페이지 리뉴얼', 10, 5000000, '진행중'),
('신규 영업 시스템', 20, 8000000, '계획'),
('인사 평가 자동화', 30, 3000000, '진행중');

-- 입력 확인
SELECT * FROM projects;
```

**projects 전체 결과:**

| project_id | project_name | dept_id | budget | status |
|------------|-------------|---------|--------|--------|
| 1 | 홈페이지 리뉴얼 | 10 | 5000000 | 진행중 |
| 2 | 신규 영업 시스템 | 20 | 8000000 | 계획 |
| 3 | 인사 평가 자동화 | 30 | 3000000 | 진행중 |

```sql
-- (3) 조건 조회
SELECT * FROM projects
WHERE status = '진행중' AND budget >= 4000000;
```

**조건 조회 결과:**

| project_id | project_name | dept_id | budget | status |
|------------|-------------|---------|--------|--------|
| 1 | 홈페이지 리뉴얼 | 10 | 5000000 | 진행중 |

---

## CRUD 핵심 요약

| 구분 | SQL 문법 | 설명 |
|------|---------|------|
| **C**reate | `INSERT INTO 테이블 (컬럼) VALUES (값)` | 새 행 추가 |
| **R**ead | `SELECT 컬럼 FROM 테이블 WHERE 조건` | 데이터 조회 |
| **U**pdate | `UPDATE 테이블 SET 컬럼=값 WHERE 조건` | 기존 행 수정 |
| **D**elete | `DELETE FROM 테이블 WHERE 조건` | 행 삭제 |

> **UPDATE, DELETE 실행 전에는 항상 같은 WHERE 조건으로 SELECT를 먼저 실행하여 대상을 확인한다.**
{: .prompt-warning }

---

## AND / OR 핵심 정리

| 연산자 | 의미 | 예시 |
|--------|------|------|
| `AND` | 모든 조건을 만족 | `WHERE city = '서울' AND age >= 30` |
| `OR` | 하나라도 만족 | `WHERE city = '서울' OR city = '부산'` |
| `괄호` | 우선순위 지정 | `WHERE dept_id = 10 AND (age >= 30 OR salary >= 4000000)` |

**우선순위:** 괄호 없이 `AND`와 `OR`를 섞으면 `AND`가 먼저 실행된다. 의도와 다른 결과가 나올 수 있으므로 **OR 조건은 반드시 괄호로 감싼다.**

---
