---
layout: post
title: "2026-1학기 중간고사준비 - MySQL 실습 환경 20문제"
date: 2026-04-16 10:00:00 +0900
categories:
  - 강의
  - 데이터베이스보안
tags:
  - mysql
  - 중간고사
  - crud
  - where
  - aggregate
  - groupby
  - orderby
  - having
  - primary-key
  - foreign-key
  - beginner
mermaid: false
pin: true
description: "2~6주차 핵심 SQL을 복습하는 중간고사 대비 실습 20문제. midterm_db 환경을 직접 만들고 CRUD, 필터링, 집계함수, GROUP BY, 키/제약조건을 단계별로 연습한다."
---

# 2026-1학기 중간고사준비 — MySQL 실습 환경 20문제

이 자료는 2주차부터 6주차까지 배운 핵심 SQL을 하나의 실습 데이터베이스에서 복습하는 중간고사 대비 문제집이다.

MySQL Workbench에서 `midterm_db`를 직접 만들고, 아래 20문제를 순서대로 풀면서 각 주차 내용을 정리한다.

다루는 범위는 다음과 같다.

| 주차 | 핵심 주제 |
|------|-----------|
| 2주차 | CRUD 기초 — CREATE TABLE, INSERT, SELECT, UPDATE, DELETE |
| 3주차 | 데이터 필터링 — WHERE, AND, OR, NOT, IN, BETWEEN, LIKE |
| 4주차 | 자료형 — INT, VARCHAR, DECIMAL, DATE, TEXT |
| 5주차 | 집계 함수 — COUNT, SUM, AVG, MAX, MIN, GROUP BY, ORDER BY, HAVING |
| 6주차 | 키와 제약조건 — PRIMARY KEY, FOREIGN KEY, NOT NULL, UNIQUE, DEFAULT, AUTO_INCREMENT |

> 주의
>
> 아래 SQL은 MySQL 8.x 기준이다.
> MySQL Workbench를 열고 로컬 서버에 연결한 상태에서 시작한다.
{: .prompt-warning }

---

## 사전 실습 1. 데이터베이스 생성

MySQL Workbench Query 창에 아래 SQL을 입력하고 실행(`Ctrl+Enter` 또는 번개 버튼)한다.

```sql
DROP DATABASE IF EXISTS midterm_db;
CREATE DATABASE midterm_db;
USE midterm_db;
```

> `USE midterm_db;`를 반드시 실행해야 이후 쿼리가 이 데이터베이스에 적용된다.
{: .prompt-tip }

---

## 사전 실습 2. 테이블 생성 및 데이터 입력

### 학과 테이블 생성

```sql
CREATE TABLE departments (
    dept_id   INT         PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL,
    budget    INT         DEFAULT 0
);
```

### 학생 테이블 생성

```sql
CREATE TABLE students (
    student_id INT          PRIMARY KEY AUTO_INCREMENT,
    name       VARCHAR(50)  NOT NULL,
    grade      INT          NOT NULL,
    city       VARCHAR(50),
    score      INT,
    dept_id    INT
);
```

### 데이터 입력

```sql
INSERT INTO departments VALUES
(1, '컴퓨터공학과', 5000000),
(2, '전자공학과',   4500000),
(3, '경영학과',     3000000),
(4, '수학과',       2000000);

INSERT INTO students (name, grade, city, score, dept_id) VALUES
('김민준', 1, '서울',  85, 1),
('이서연', 2, '부산',  92, 1),
('박지호', 1, '서울',  70, 2),
('최수아', 3, '인천',  88, 2),
('정우진', 2, '서울',  63, 3),
('한예린', 3, '부산',  95, 3),
('오시현', 1, '대전',  77, 1),
('윤도현', 2, '서울',  55, 4),
('임지아', 3, '인천',  82, 2),
('강태양', 1, '부산',  91, 3);
```

### 입력 확인

```sql
SELECT * FROM departments;
SELECT * FROM students;
```

**departments 결과:**

| dept_id | dept_name | budget |
|---------|-----------|--------|
| 1 | 컴퓨터공학과 | 5000000 |
| 2 | 전자공학과 | 4500000 |
| 3 | 경영학과 | 3000000 |
| 4 | 수학과 | 2000000 |

**students 결과:**

| student_id | name | grade | city | score | dept_id |
|------------|------|-------|------|-------|---------|
| 1 | 김민준 | 1 | 서울 | 85 | 1 |
| 2 | 이서연 | 2 | 부산 | 92 | 1 |
| 3 | 박지호 | 1 | 서울 | 70 | 2 |
| 4 | 최수아 | 3 | 인천 | 88 | 2 |
| 5 | 정우진 | 2 | 서울 | 63 | 3 |
| 6 | 한예린 | 3 | 부산 | 95 | 3 |
| 7 | 오시현 | 1 | 대전 | 77 | 1 |
| 8 | 윤도현 | 2 | 서울 | 55 | 4 |
| 9 | 임지아 | 3 | 인천 | 82 | 2 |
| 10 | 강태양 | 1 | 부산 | 91 | 3 |

---

## 문제 1

`students` 테이블의 모든 컬럼과 모든 행을 조회하라.

**힌트:** `SELECT *`를 사용한다.

**답:**

```sql
SELECT * FROM students;
```

**예상 결과:** 위 사전 실습의 students 결과와 동일 (10행)

---

## 문제 2

`students` 테이블에서 `name`과 `score` 컬럼만 조회하라.

**힌트:** SELECT 뒤에 원하는 컬럼 이름을 쉼표로 나열한다.

**답:**

```sql
SELECT name, score FROM students;
```

**예상 결과:**

| name | score |
|------|-------|
| 김민준 | 85 |
| 이서연 | 92 |
| 박지호 | 70 |
| ... | ... |

---

## 문제 3

`students` 테이블에서 `grade`가 `1`인 학생만 조회하라.

**힌트:** `WHERE 컬럼 = 값` 형식을 사용한다.

**답:**

```sql
SELECT * FROM students WHERE grade = 1;
```

**예상 결과:**

| student_id | name | grade | city | score | dept_id |
|------------|------|-------|------|-------|---------|
| 1 | 김민준 | 1 | 서울 | 85 | 1 |
| 3 | 박지호 | 1 | 서울 | 70 | 2 |
| 7 | 오시현 | 1 | 대전 | 77 | 1 |
| 10 | 강태양 | 1 | 부산 | 91 | 3 |

---

## 문제 4

`students` 테이블에서 `score`가 `80` 이상인 학생의 이름과 점수를 조회하라.

**힌트:** 비교 연산자 `>=`를 사용한다.

**답:**

```sql
SELECT name, score FROM students WHERE score >= 80;
```

**예상 결과:**

| name | score |
|------|-------|
| 김민준 | 85 |
| 이서연 | 92 |
| 최수아 | 88 |
| 한예린 | 95 |
| 임지아 | 82 |
| 강태양 | 91 |

---

## 문제 5

`students` 테이블에서 `city`가 `'서울'`이고 `score`가 `80` 이상인 학생을 조회하라.

**힌트:** 두 조건을 모두 만족해야 하므로 `AND`를 사용한다.

**답:**

```sql
SELECT * FROM students WHERE city = '서울' AND score >= 80;
```

**예상 결과:**

| student_id | name | grade | city | score | dept_id |
|------------|------|-------|------|-------|---------|
| 1 | 김민준 | 1 | 서울 | 85 | 1 |

---

## 문제 6

`students` 테이블에서 `city`가 `'부산'`이거나 `'인천'`인 학생을 조회하라.

**힌트:** 조건 중 하나만 만족해도 되므로 `OR`을 사용한다. `IN`을 사용하면 더 간결하다.

**답:**

```sql
-- OR 사용
SELECT * FROM students WHERE city = '부산' OR city = '인천';

-- IN 사용 (같은 결과)
SELECT * FROM students WHERE city IN ('부산', '인천');
```

**예상 결과:**

| student_id | name | grade | city | score | dept_id |
|------------|------|-------|------|-------|---------|
| 2 | 이서연 | 2 | 부산 | 92 | 1 |
| 4 | 최수아 | 3 | 인천 | 88 | 2 |
| 6 | 한예린 | 3 | 부산 | 95 | 3 |
| 9 | 임지아 | 3 | 인천 | 82 | 2 |
| 10 | 강태양 | 1 | 부산 | 91 | 3 |

---

## 문제 7

`students` 테이블에서 이름이 `'김'`으로 시작하는 학생을 조회하라.

**힌트:** `LIKE '김%'`를 사용한다. `%`는 뒤에 어떤 문자가 와도 된다는 뜻이다.

**답:**

```sql
SELECT * FROM students WHERE name LIKE '김%';
```

**예상 결과:**

| student_id | name | grade | city | score | dept_id |
|------------|------|-------|------|-------|---------|
| 1 | 김민준 | 1 | 서울 | 85 | 1 |

---

## 문제 8

`students` 테이블에서 `score`가 `70` 이상 `90` 이하인 학생을 조회하라.

**힌트:** `BETWEEN 최솟값 AND 최댓값` 형식을 사용한다. 경계값 포함이다.

**답:**

```sql
SELECT * FROM students WHERE score BETWEEN 70 AND 90;
```

**예상 결과:**

| student_id | name | grade | city | score | dept_id |
|------------|------|-------|------|-------|---------|
| 1 | 김민준 | 1 | 서울 | 85 | 1 |
| 3 | 박지호 | 1 | 서울 | 70 | 2 |
| 4 | 최수아 | 3 | 인천 | 88 | 2 |
| 7 | 오시현 | 1 | 대전 | 77 | 1 |
| 9 | 임지아 | 3 | 인천 | 82 | 2 |

---

## 문제 9

`students` 테이블의 전체 행을 `score` 기준 내림차순으로 정렬해서 조회하라.

**힌트:** `ORDER BY 컬럼 DESC`를 사용한다.

**답:**

```sql
SELECT * FROM students ORDER BY score DESC;
```

**예상 결과:**

| student_id | name | grade | city | score | dept_id |
|------------|------|-------|------|-------|---------|
| 6 | 한예린 | 3 | 부산 | 95 | 3 |
| 2 | 이서연 | 2 | 부산 | 92 | 1 |
| 10 | 강태양 | 1 | 부산 | 91 | 3 |
| ... | ... | ... | ... | ... | ... |

---

## 문제 10

`students` 테이블에서 `grade`를 기준으로 오름차순, 같은 학년이면 `score` 기준 내림차순으로 정렬하라.

**힌트:** `ORDER BY` 뒤에 정렬 기준을 쉼표로 여러 개 지정할 수 있다.

**답:**

```sql
SELECT * FROM students ORDER BY grade ASC, score DESC;
```

**예상 결과:**

| student_id | name | grade | city | score | dept_id |
|------------|------|-------|------|-------|---------|
| 10 | 강태양 | 1 | 부산 | 91 | 3 |
| 1 | 김민준 | 1 | 서울 | 85 | 1 |
| 7 | 오시현 | 1 | 대전 | 77 | 1 |
| 3 | 박지호 | 1 | 서울 | 70 | 2 |
| 2 | 이서연 | 2 | 부산 | 92 | 1 |
| ... | ... | ... | ... | ... | ... |

---

## 문제 11

`students` 테이블의 전체 학생 수를 구하라.

**힌트:** `COUNT(*)`는 NULL 포함 전체 행 수를 센다.

**답:**

```sql
SELECT COUNT(*) AS 전체학생수 FROM students;
```

**예상 결과:**

| 전체학생수 |
|-----------|
| 10 |

---

## 문제 12

`students` 테이블에서 `score`의 합계와 평균을 함께 조회하라.

**힌트:** `SUM`과 `AVG`를 하나의 SELECT에 함께 쓸 수 있다. `AS`로 컬럼 별칭을 붙인다.

**답:**

```sql
SELECT SUM(score) AS 합계, AVG(score) AS 평균 FROM students;
```

**예상 결과:**

| 합계 | 평균 |
|------|------|
| 798 | 79.8000 |

---

## 문제 13

`students` 테이블에서 `score`의 최댓값과 최솟값을 조회하라.

**힌트:** `MAX`와 `MIN`을 사용한다.

**답:**

```sql
SELECT MAX(score) AS 최고점수, MIN(score) AS 최저점수 FROM students;
```

**예상 결과:**

| 최고점수 | 최저점수 |
|---------|---------|
| 95 | 55 |

---

## 문제 14

`students` 테이블에서 `grade`(학년)별로 학생 수와 평균 점수를 구하라.

**힌트:** `GROUP BY grade`로 학년별로 묶은 뒤 집계한다.

**답:**

```sql
SELECT grade, COUNT(*) AS 학생수, AVG(score) AS 평균점수
FROM students
GROUP BY grade;
```

**예상 결과:**

| grade | 학생수 | 평균점수 |
|-------|--------|---------|
| 1 | 4 | 80.7500 |
| 2 | 3 | 70.0000 |
| 3 | 3 | 88.3333 |

---

## 문제 15

`students` 테이블에서 `dept_id`별 평균 점수를 구하되, 평균이 `80` 이상인 학과만 출력하라.

**힌트:** 집계 결과에 조건을 거는 것이므로 `WHERE`가 아니라 `HAVING`을 사용한다.

**답:**

```sql
SELECT dept_id, AVG(score) AS 평균점수
FROM students
GROUP BY dept_id
HAVING AVG(score) >= 80;
```

**예상 결과:**

| dept_id | 평균점수 |
|---------|---------|
| 1 | 84.0000 |
| 2 | 80.0000 |
| 3 | 83.0000 |

> `WHERE`는 GROUP BY **전에** 행을 거르고, `HAVING`은 GROUP BY **후에** 그룹을 거른다.
{: .prompt-tip }

---

## 문제 16

`students` 테이블에서 `city`별 학생 수를 구하고, 학생 수가 많은 순서대로 정렬하라.

**힌트:** `GROUP BY` + `ORDER BY COUNT(*) DESC`를 함께 사용한다.

**답:**

```sql
SELECT city, COUNT(*) AS 학생수
FROM students
GROUP BY city
ORDER BY COUNT(*) DESC;
```

**예상 결과:**

| city | 학생수 |
|------|--------|
| 서울 | 4 |
| 부산 | 3 |
| 인천 | 2 |
| 대전 | 1 |

---

## 문제 17

`students` 테이블에서 `윤도현`의 `score`를 `72`로 변경하라. 변경 후 결과를 SELECT로 확인하라.

**힌트:** `UPDATE 테이블 SET 컬럼=값 WHERE 조건`을 사용한다. WHERE 없이 UPDATE하면 모든 행이 바뀐다.

**답:**

```sql
UPDATE students SET score = 72 WHERE name = '윤도현';
SELECT * FROM students WHERE name = '윤도현';
```

**예상 결과:**

| student_id | name | grade | city | score | dept_id |
|------------|------|-------|------|-------|---------|
| 8 | 윤도현 | 2 | 서울 | 72 | 4 |

> Safe Update Mode가 켜져 있으면 PK를 조건으로 써야 실행된다. Workbench 설정에서 OFF하거나 `WHERE student_id = 8`로 변경한다.
{: .prompt-warning }

---

## 문제 18

`students` 테이블에서 `score`가 `60` 미만인 학생을 삭제하라. 삭제 전 SELECT로 대상을 먼저 확인하라.

**힌트:** 삭제 전 `SELECT * FROM students WHERE score < 60;`으로 대상을 확인하는 습관을 들인다.

**답:**

```sql
-- 삭제 대상 먼저 확인
SELECT * FROM students WHERE score < 60;

-- 삭제 실행
DELETE FROM students WHERE score < 60;
```

**예상 결과 (삭제 대상):**

삭제 전 기준, score가 60 미만인 행은 없다. (문제 17에서 윤도현을 72로 수정했기 때문)

> 만약 문제 17을 건너뛰었다면 `윤도현 (score=55)`가 삭제 대상이 된다.
{: .prompt-info }

---

## 문제 19

아래 조건으로 `orders` 테이블을 새로 만들어라.

| 컬럼 | 자료형 | 제약조건 |
|------|--------|---------|
| order_id | INT | PRIMARY KEY, AUTO_INCREMENT |
| item_name | VARCHAR(100) | NOT NULL |
| quantity | INT | NOT NULL, DEFAULT 1 |
| price | DECIMAL(10,2) | NOT NULL |
| order_date | DATE | |

**힌트:** CREATE TABLE 문에 각 컬럼의 자료형과 제약조건을 함께 작성한다.

**답:**

```sql
CREATE TABLE orders (
    order_id   INT            PRIMARY KEY AUTO_INCREMENT,
    item_name  VARCHAR(100)   NOT NULL,
    quantity   INT            NOT NULL DEFAULT 1,
    price      DECIMAL(10,2)  NOT NULL,
    order_date DATE
);
```

**확인:**

```sql
DESC orders;
```

**예상 결과:**

| Field | Type | Null | Key | Default | Extra |
|-------|------|------|-----|---------|-------|
| order_id | int | NO | PRI | NULL | auto_increment |
| item_name | varchar(100) | NO | | NULL | |
| quantity | int | NO | | 1 | |
| price | decimal(10,2) | NO | | NULL | |
| order_date | date | YES | | NULL | |

---

## 문제 20

`students` 테이블의 `dept_id` 컬럼이 `departments` 테이블의 `dept_id`를 참조하도록 외래 키(FOREIGN KEY)를 추가하라.

**힌트:** 이미 존재하는 테이블에 FK를 추가하려면 `ALTER TABLE`을 사용한다.

**답:**

```sql
ALTER TABLE students
ADD CONSTRAINT fk_students_dept
FOREIGN KEY (dept_id) REFERENCES departments(dept_id);
```

**확인:**

```sql
-- FK 추가 후 존재하지 않는 dept_id로 INSERT 시도
INSERT INTO students (name, grade, city, score, dept_id)
VALUES ('테스트', 1, '서울', 80, 99);
-- 오류 발생: Cannot add or update a child row: a foreign key constraint fails
```

**예상 결과:** FK가 정상 등록되면 `dept_id = 99`처럼 `departments`에 없는 값은 삽입이 거부된다.

> FK를 추가하려면 참조 대상 테이블(`departments`)이 먼저 존재해야 하고, 현재 `students`에 이미 들어있는 데이터의 `dept_id`가 모두 `departments`에 있어야 한다.
{: .prompt-tip }

---

## 마무리

위 20문제는 2~6주차 핵심 SQL을 실제 테이블에서 직접 작성하며 연습하는 데 초점을 맞췄다. 틀린 문제가 있으면 해당 주차 자료를 다시 읽고 반복 실습하자.

추가로 복습할 때는 아래를 점검하라.

- `WHERE`와 `HAVING`의 차이를 설명할 수 있는가? (`WHERE`는 행 필터, `HAVING`은 그룹 필터)
- `GROUP BY` 절에 없는 컬럼을 `SELECT`에 쓰면 어떤 오류가 나는지 알고 있는가?
- `LIKE '%단어%'`와 `LIKE '단어%'`의 차이를 알고 있는가?
- `COUNT(*)`와 `COUNT(컬럼)`의 차이를 설명할 수 있는가? (NULL 포함 여부)
- `PRIMARY KEY`와 `FOREIGN KEY`의 역할 차이를 설명할 수 있는가?
- `UPDATE`/`DELETE` 실행 전에 `SELECT`로 대상을 먼저 확인하는 습관이 있는가?
