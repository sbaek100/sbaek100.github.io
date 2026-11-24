---
title: "[애플리케이션 보안] 05-1. 데이터베이스 개념과 접근통제"
date: 2026-10-21 09:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - DB보안
tags:
  - Database
  - MariaDB
  - SQL
  - GRANT
  - REVOKE
  - View
  - 접근통제
math: false
mermaid: false
---


## 0. 실습 환경

| 장비 | IP | 역할 |
|---|---|---|
| Ubuntu | **192.168.0.30** | DB 서버(MariaDB). 이 글의 실습은 **모두 Ubuntu** 에서 합니다 |
| Kali | **192.168.0.10** | 이 글에서는 쓰지 않음(05-2 4.4절에서 원격 접속 시험에 사용) |

> 코드 블록 첫 줄의 `# === Ubuntu ===` 는 **Ubuntu 터미널**에서, `-- === Ubuntu: MariaDB console ... ===` 는 **MariaDB 콘솔**(`MariaDB [...]>` 프롬프트)에서 입력한다는 뜻입니다.
>
> 명령과 예상 화면은 **Ubuntu 24.04.4 LTS(MariaDB 10.11.14)** 에서 실행해 확인했습니다. 실행 시간(`(0.001 sec)`), 접속 번호(`connection id`), 프로세스 번호는 실행할 때마다 다릅니다. 콘솔에 들어갈 때 나오는 안내문(`Welcome to the MariaDB monitor ...`)은 1.1절에서 한 번 보여 주고, 이후 관리자 콘솔 화면에서는 생략합니다.
{: .prompt-warning }

> DB 서버는 보통 **웹 서버 뒤쪽 내부망**에 두고, 인터넷이나 사용자 PC에서 직접 접속하지 못하게 합니다. 그래서 이 글의 실습도 DB 서버(Ubuntu) **안에서만** 접속합니다. 05-2에서 바깥(Kali)에서는 접속이 안 되는 것을 확인합니다.
{: .prompt-info }

실습에서 만드는 것은 다음과 같습니다.

| 이름 | 종류 | 내용 |
|---|---|---|
| `companydb` | 데이터베이스 | 회사의 부서·직원 정보 |
| `departments` | 테이블 | 부서 번호, 부서 이름 |
| `employees` | 테이블 | 직원 번호, 이름, 이메일, 부서 번호, **급여(민감 정보)** |
| `appuser` / `apppass123` | DB 계정 | 웹 애플리케이션이 쓰는 계정. **최소 권한만** 줌 |
| `emp_public` | 뷰 | 급여를 뺀 직원 목록 |
| `hr_kim`, `hr_reader`, `hr_writer` | 계정·역할 | 2.5절 실습용. 실습 끝에 **지움** |

| 절 | Ubuntu에서 할 일 |
|---|---|
| 1.1 DB와 DBMS | MariaDB 설치, 콘솔 접속 |
| 1.2 키 | 테이블 만들기, 키 확인, 중복·참조 오류 확인 |
| 1.3 SQL의 종류 | 트랜잭션 취소(ROLLBACK) |
| 2.1 계정과 인증 | 기본 계정과 root 인증 방식 확인 |
| 2.2 권한 부여·회수 | 최소 권한 계정 만들기, 권한 밖 작업 거부 확인 |
| 2.3 뷰 | 급여를 숨긴 뷰만 보여 주기 |
| 2.4 로그인 실패 잠금 | 비밀번호 3회 실패 시 계정 잠금과 해제 |
| 2.5 관리자 권한 제한·암호 정책·역할 | 관리자급 계정 점검, 약한 암호 거부, 역할(Role)로 권한 묶어 주기 |

---

# 1. 데이터베이스 개념

## 1.1 데이터베이스와 DBMS

### [이론]

**데이터베이스(Database)** 는 여러 사람이 함께 쓰려고 **체계적으로 모아 둔 데이터**입니다. 데이터베이스를 만들고, 저장하고, 찾고, 지키는 프로그램을 **DBMS(Database Management System)** 라고 합니다.

| 구분 | 예 |
|---|---|
| 관계형 DBMS(RDBMS) | **Oracle**, **MS-SQL**(SQL Server), **MySQL**, **MariaDB**, PostgreSQL |
| 비관계형(NoSQL) | MongoDB, Redis |

관계형 DB는 데이터를 **표(테이블)** 로 저장합니다.

| 일상 용어 | 관계형 DB 용어 | 예(`employees` 테이블) |
|---|---|---|
| 표 | **테이블**(릴레이션) | `employees` |
| 가로줄 | **행**(튜플, 레코드) | 직원 한 명의 정보 |
| 세로줄 | **열**(속성, 컬럼) | `name`, `salary` |
| 표의 설계도 | **스키마** | 어떤 열이 있고 각 열에 어떤 형식의 값이 들어가는지 |

> **MariaDB** 는 MySQL을 만든 개발자들이 MySQL에서 갈라져 나와 만든 **무료 오픈소스** DBMS입니다. 명령과 사용법이 MySQL과 거의 같습니다. Ubuntu에서는 `mariadb` 와 `mysql` 두 명령 이름을 모두 쓸 수 있습니다.
{: .prompt-info }

#### DB 보안이 중요한 이유

DB에는 개인정보, 급여, 카드번호 같은 **회사의 가장 중요한 정보**가 모여 있습니다. 웹 서버가 뚫려도 DB가 지켜지면 피해가 줄고, DB가 뚫리면 정보가 한꺼번에 새어 나갑니다.

| DB 보안 요구사항 | 뜻 | 이 과정에서 다루는 곳 |
|---|---|---|
| **부적절한 접근 방지** | 허가된 사용자만, 허가된 범위에서 접근 | 2.2·2.3·2.5절 |
| **사용자 인증** | 접속하는 사람이 누구인지 확인 | 2.1절 |
| **무결성 보장** | 잘못된 값, 허가 없는 변경을 막음 | 1.2절(키), 1.3절(트랜잭션) |
| **추론 방지** | 공개된 값으로 비밀 값을 알아내지 못하게 함 | 05-2 3.3절 |
| **기밀 데이터 보호** | 민감한 값은 암호화 | 05-2 4.2절 |
| **감사(Audit)** | 누가 언제 무엇을 했는지 기록 | 05-2 4.3절 |

### [실습] (Ubuntu) MariaDB 설치와 콘솔 접속

```bash
# === Ubuntu ===
sudo apt update && sudo apt install -y mariadb-server
systemctl is-active mariadb
mariadb --version
sudo ss -tlnp | grep 3306
```

| 명령·옵션 | 설명 |
|---|---|
| `apt install -y mariadb-server` | MariaDB 서버와 콘솔 프로그램(클라이언트)을 설치. 설치가 끝나면 바로 실행됨 |
| `systemctl is-active mariadb` | DB 서버가 동작 중인지 한 단어로 표시 |
| `mariadb --version` | 설치된 버전 표시 |
| `ss -tlnp` 와 `grep 3306` | DB 서버가 기다리는 **3306번 포트**를 찾음 |

`apt` 는 설치 과정을 길게 출력합니다. 끝부분은 다음과 같습니다(설치 진행 줄 수십 개는 `...` 로 생략).

```
...
Setting up mariadb-client (1:10.11.14-0ubuntu0.24.04.1) ...
...
Setting up mariadb-server (1:10.11.14-0ubuntu0.24.04.1) ...
Created symlink /etc/systemd/system/multi-user.target.wants/mariadb.service → /usr/lib/systemd/system/mariadb.service.
...
Processing triggers for mariadb-server (1:10.11.14-0ubuntu0.24.04.1) ...
```

나머지 세 명령의 결과는 다음과 같습니다.

```
active
mariadb  Ver 15.1 Distrib 10.11.14-MariaDB, for debian-linux-gnu (x86_64) using  EditLine wrapper
LISTEN 0      80                           127.0.0.1:3306      0.0.0.0:*    users:(("mariadbd",pid=1517,fd=24))
```

| 화면 | 의미 |
|---|---|
| `10.11.14-MariaDB` | 설치된 MariaDB 버전 |
| `127.0.0.1:3306` | DB 서버가 **자기 자신(127.0.0.1)** 의 3306번 포트에서만 기다림. 다른 PC에서는 접속할 수 없음(05-2 4.4절) |

> 패키지 버전 숫자는 업데이트 시점에 따라 다를 수 있습니다.
{: .prompt-info }

DB 관리자(root)로 콘솔에 들어갑니다.

```bash
# === Ubuntu ===
sudo mariadb
```

```
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 31
Server version: 10.11.14-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

| 화면 | 의미 |
|---|---|
| `MariaDB [(none)]>` | MariaDB **콘솔 프롬프트**. 괄호 안은 지금 쓰고 있는 데이터베이스(아직 없음) |
| `Commands end with ;` | SQL 명령은 **세미콜론(`;`)** 으로 끝내야 실행됨 |

> `sudo mariadb` 는 비밀번호를 묻지 않습니다. Ubuntu의 MariaDB 관리자 계정은 **"리눅스의 root(sudo)로 접속한 사람만 DB의 root로 인정"** 하는 방식(unix_socket)을 쓰기 때문입니다. 2.1절에서 자세히 봅니다. 콘솔에서 나갈 때는 `exit` 를 입력합니다.
{: .prompt-info }

---

## 1.2 키(Key)와 무결성

### [이론]

**키(Key)** 는 테이블에서 **행 하나를 정확히 가려내는 열(또는 열의 묶음)** 입니다. 이 글의 `employees` 테이블로 설명합니다.

| 키 | 뜻 | `employees` 에서 |
|---|---|---|
| **후보키**(Candidate Key) | 행을 하나로 가려낼 수 있는(**유일성**) 최소한의 열(**최소성**) | `emp_id`, `email` |
| **기본키**(Primary Key) | 후보키 가운데 **대표로 고른 하나**. 빈 값(NULL)과 중복을 허용하지 않음 | `emp_id` |
| **대체키**(Alternate Key) | 후보키 가운데 **기본키로 뽑히고 남은** 키 | `email` |
| **슈퍼키**(Super Key) | 유일성만 만족하면 됨(최소성은 없어도 됨). 키에 열을 더 붙인 묶음 | `{emp_id, name}`, `{email, salary}` |
| **외래키**(Foreign Key) | **다른 테이블의 기본키**를 가리키는 열 | `dept_id` → `departments.dept_id` |

> 기본키로 `{학번}` 을 골랐다면 남은 후보키 `{이름, 학과}` 는 **대체키**입니다. 후보키·대체키·슈퍼키의 차이는 혼동하기 쉬우니, **"유일성+최소성 = 후보키, 그중 대표 = 기본키, 나머지 = 대체키, 유일성만 = 슈퍼키"** 로 정리해 둡니다.
{: .prompt-warning }

키는 데이터가 잘못 들어가지 않게 지키는 **무결성 규칙**과 연결됩니다.

| 무결성 | 규칙 | 지키는 키 |
|---|---|---|
| **개체 무결성** | 기본키는 비어 있거나(NULL) 중복될 수 없음 | 기본키 |
| **참조 무결성** | 외래키 값은 참조하는 테이블에 **실제로 있는 값**이어야 함 | 외래키 |

### [실습 1] (Ubuntu) 데이터베이스와 테이블 만들기

`sudo mariadb` 로 들어간 콘솔에서 아래를 입력합니다. 여러 줄을 한꺼번에 붙여 넣어도 됩니다.

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
CREATE DATABASE companydb;
USE companydb;
CREATE TABLE departments (
  dept_id   INT PRIMARY KEY,
  dept_name VARCHAR(30) NOT NULL UNIQUE
);
CREATE TABLE employees (
  emp_id  INT PRIMARY KEY,
  name    VARCHAR(30) NOT NULL,
  email   VARCHAR(50) NOT NULL UNIQUE,
  dept_id INT,
  salary  INT,
  FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);
INSERT INTO departments VALUES (10,'Sales'), (20,'Dev'), (30,'Security');
INSERT INTO employees VALUES
  (1,'Kim','kim@lab.test',10,4200),
  (2,'Lee','lee@lab.test',10,3900),
  (3,'Park','park@lab.test',20,5100),
  (4,'Choi','choi@lab.test',20,4800),
  (5,'Jung','jung@lab.test',20,4500),
  (6,'Kang','kang@lab.test',30,6300);
SELECT * FROM employees;
```

```
MariaDB [(none)]> CREATE DATABASE companydb;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> USE companydb;
Database changed
MariaDB [companydb]> CREATE TABLE departments (
    ->   dept_id   INT PRIMARY KEY,
    ->   dept_name VARCHAR(30) NOT NULL UNIQUE
    -> );
Query OK, 0 rows affected (0.007 sec)

MariaDB [companydb]> CREATE TABLE employees (
    ->   emp_id  INT PRIMARY KEY,
    ->   name    VARCHAR(30) NOT NULL,
    ->   email   VARCHAR(50) NOT NULL UNIQUE,
    ->   dept_id INT,
    ->   salary  INT,
    ->   FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
    -> );
Query OK, 0 rows affected (0.006 sec)

MariaDB [companydb]> INSERT INTO departments VALUES (10,'Sales'), (20,'Dev'), (30,'Security');
Query OK, 3 rows affected (0.001 sec)
Records: 3  Duplicates: 0  Warnings: 0

MariaDB [companydb]> INSERT INTO employees VALUES
    ->   (1,'Kim','kim@lab.test',10,4200),
    ->   (2,'Lee','lee@lab.test',10,3900),
    ->   (3,'Park','park@lab.test',20,5100),
    ->   (4,'Choi','choi@lab.test',20,4800),
    ->   (5,'Jung','jung@lab.test',20,4500),
    ->   (6,'Kang','kang@lab.test',30,6300);
Query OK, 6 rows affected (0.002 sec)
Records: 6  Duplicates: 0  Warnings: 0

MariaDB [companydb]> SELECT * FROM employees;
+--------+------+---------------+---------+--------+
| emp_id | name | email         | dept_id | salary |
+--------+------+---------------+---------+--------+
|      1 | Kim  | kim@lab.test  |      10 |   4200 |
|      2 | Lee  | lee@lab.test  |      10 |   3900 |
|      3 | Park | park@lab.test |      20 |   5100 |
|      4 | Choi | choi@lab.test |      20 |   4800 |
|      5 | Jung | jung@lab.test |      20 |   4500 |
|      6 | Kang | kang@lab.test |      30 |   6300 |
+--------+------+---------------+---------+--------+
6 rows in set (0.000 sec)
```

| 명령 | 설명 |
|---|---|
| `CREATE DATABASE companydb;` | 데이터베이스 `companydb` 를 만듦 |
| `USE companydb;` | 앞으로 `companydb` 를 쓰겠다고 지정. 프롬프트가 `MariaDB [companydb]>` 로 바뀜 |
| `CREATE TABLE ... ( ... );` | 테이블을 만듦. 괄호 안에 `열이름 형식 조건` 을 쉼표로 나열 |
| `INT`, `VARCHAR(30)` | 정수, 최대 30글자 문자열 |
| `PRIMARY KEY` | 기본키 지정 |
| `NOT NULL UNIQUE` | 비어 있으면 안 되고(NOT NULL) 중복도 안 됨(UNIQUE) → `email` 이 **대체키** 역할 |
| `FOREIGN KEY (dept_id) REFERENCES departments(dept_id)` | `dept_id` 를 `departments` 테이블의 기본키를 가리키는 **외래키**로 지정 |
| `INSERT INTO ... VALUES (...), (...);` | 행을 여러 개 한 번에 넣음 |
| `SELECT * FROM employees;` | 모든 열(`*`)을 조회 |

| 화면 | 의미 |
|---|---|
| `->` | 명령이 `;` 로 끝나지 않아 **다음 줄을 기다리는 중**이라는 표시 |
| `Query OK, 6 rows affected` | 6개 행이 들어감 |

> 직원 6명 가운데 **Security 부서는 Kang 한 명뿐**입니다. 이 점은 05-2의 추론 공격 실습에서 중요해집니다.
{: .prompt-info }

### [실습 2] (Ubuntu) 키 확인과 무결성 오류

같은 콘솔에서 이어서 입력합니다.

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
DESC employees;
INSERT INTO employees VALUES (7,'Yoon','kim@lab.test',10,4000);
INSERT INTO employees VALUES (7,'Yoon','yoon@lab.test',99,4000);
```

```
MariaDB [companydb]> DESC employees;
+---------+-------------+------+-----+---------+-------+
| Field   | Type        | Null | Key | Default | Extra |
+---------+-------------+------+-----+---------+-------+
| emp_id  | int(11)     | NO   | PRI | NULL    |       |
| name    | varchar(30) | NO   |     | NULL    |       |
| email   | varchar(50) | NO   | UNI | NULL    |       |
| dept_id | int(11)     | YES  | MUL | NULL    |       |
| salary  | int(11)     | YES  |     | NULL    |       |
+---------+-------------+------+-----+---------+-------+
5 rows in set (0.001 sec)

MariaDB [companydb]> INSERT INTO employees VALUES (7,'Yoon','kim@lab.test',10,4000);
ERROR 1062 (23000): Duplicate entry 'kim@lab.test' for key 'email'
MariaDB [companydb]> INSERT INTO employees VALUES (7,'Yoon','yoon@lab.test',99,4000);
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`companydb`.`employees`, CONSTRAINT `employees_ibfk_1` FOREIGN KEY (`dept_id`) REFERENCES `departments` (`dept_id`))
```

| 명령 | 설명 |
|---|---|
| `DESC employees;` | 테이블 구조(열, 형식, 키)를 보여 줌(DESCribe) |

| 화면 | 의미 |
|---|---|
| `emp_id ... PRI` | **기본키**(PRImary) |
| `email ... UNI` | **유일**(UNIque) 조건 → 대체키 |
| `dept_id ... MUL` | 같은 값이 여러 번 나올 수 있는 색인. 여기서는 **외래키** |
| `ERROR 1062 ... Duplicate entry 'kim@lab.test'` | 대체키(`email`)에 이미 있는 값을 넣으려 해서 **거부** |
| `ERROR 1452 ... foreign key constraint fails` | 없는 부서(`99`)를 가리키려 해서 **참조 무결성** 위반으로 거부 |

> 두 INSERT가 모두 거부되었으므로 `employees` 테이블은 여전히 6명입니다. DBMS가 **잘못된 데이터가 들어오는 것 자체를 막아** 무결성을 지킵니다.
{: .prompt-tip }

---

## 1.3 SQL의 종류 — DDL·DML·DCL

### [이론]

**SQL(Structured Query Language)** 은 관계형 DB를 다루는 표준 언어입니다. 하는 일에 따라 나눕니다.

| 분류 | 이름 | 하는 일 | 명령 |
|---|---|---|---|
| **DDL** | 데이터 정의어(Definition) | DB·테이블·뷰 같은 **구조**를 만들고 바꾸고 지움 | **CREATE**, **ALTER**, **DROP**, TRUNCATE |
| **DML** | 데이터 조작어(Manipulation) | 테이블 안의 **데이터**를 조회·추가·수정·삭제 | **SELECT**, **INSERT**, **UPDATE**, **DELETE** |
| **DCL** | 데이터 제어어(Control) | 사용자의 **권한**을 주고 빼앗음 | **GRANT**, **REVOKE** |
| TCL | 트랜잭션 제어어 | 여러 작업을 한 묶음으로 **확정**하거나 **취소** | COMMIT, ROLLBACK |

> `CREATE` 는 DDL, `SELECT`·`INSERT`·`UPDATE` 는 DML, `GRANT`·`REVOKE` 는 DCL입니다. 책에 따라 TCL을 DCL에 포함하기도 하고, `SELECT` 만 따로 DQL(질의어)로 나누기도 합니다.
{: .prompt-warning }

**트랜잭션(Transaction)** 은 "모두 성공하거나 모두 취소되어야 하는" 작업의 묶음입니다. 계좌 이체에서 출금만 되고 입금이 안 되면 안 되는 것과 같습니다. `ROLLBACK` 으로 취소하면 묶음 안의 작업이 **없던 일**이 되어 데이터 무결성이 지켜집니다.

### [실습] (Ubuntu) 지운 데이터를 ROLLBACK으로 되돌리기

같은 콘솔에서 이어서 입력합니다.

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
START TRANSACTION;
DELETE FROM employees WHERE emp_id = 1;
SELECT COUNT(*) FROM employees;
ROLLBACK;
SELECT COUNT(*) FROM employees;
exit
```

```
MariaDB [companydb]> START TRANSACTION;
Query OK, 0 rows affected (0.000 sec)

MariaDB [companydb]> DELETE FROM employees WHERE emp_id = 1;
Query OK, 1 row affected (0.000 sec)

MariaDB [companydb]> SELECT COUNT(*) FROM employees;
+----------+
| COUNT(*) |
+----------+
|        5 |
+----------+
1 row in set (0.000 sec)

MariaDB [companydb]> ROLLBACK;
Query OK, 0 rows affected (0.000 sec)

MariaDB [companydb]> SELECT COUNT(*) FROM employees;
+----------+
| COUNT(*) |
+----------+
|        6 |
+----------+
1 row in set (0.000 sec)

MariaDB [companydb]> exit
Bye
```

| 명령 | 분류 | 설명 |
|---|---|---|
| `START TRANSACTION;` | TCL | 트랜잭션 시작. 이후 작업은 확정 전까지 임시 상태 |
| `DELETE FROM employees WHERE emp_id = 1;` | DML | 직원 번호 1번(Kim)을 지움. `WHERE` 로 지울 행을 고름 |
| `SELECT COUNT(*) FROM employees;` | DML | 행의 개수를 셈 |
| `ROLLBACK;` | TCL | 트랜잭션 안의 작업을 **모두 취소** |
| `exit` | — | 콘솔에서 나감 |

| 화면 | 의미 |
|---|---|
| `5` → `6` | 지운 것처럼 보였던 Kim이 `ROLLBACK` 으로 **되살아남** |

> `ROLLBACK` 대신 `COMMIT;` 을 입력하면 작업이 **확정**되어 되돌릴 수 없습니다. `WHERE` 없이 `DELETE FROM employees;` 를 실행하면 **모든 행**이 지워지므로 특히 조심합니다.
{: .prompt-danger }

---

# 2. DB 접근통제

## 2.1 계정과 인증 — 기본 계정과 관리자 인증

### [이론]

DBMS는 설치할 때 **관리자 계정**과 **예제·관리용 계정**을 미리 만들어 둡니다. 이 계정의 이름과 비밀번호는 설명서에 공개되어 있어, 바꾸지 않으면 누구나 관리자로 들어올 수 있습니다.

| DBMS | 기본 계정 | 알려진 기본 비밀번호(옛 버전) |
|---|---|---|
| **Oracle** | `sys` | `change_on_install` |
| | `system` | **`manager`** |
| | `scott`(예제 계정) | `tiger` |
| | `dbsnmp`(모니터링 계정) | `dbsnmp` |
| **MS-SQL** | `sa`(system administrator) | 빈 비밀번호로 설치되던 시절이 있었음 |
| **MySQL/MariaDB** | `root` | 옛 버전은 빈 비밀번호 |

> Oracle의 `system` 계정 기본 비밀번호는 **`manager`** 입니다(`manage` 가 아님). 기본 계정은 **비밀번호를 바꾸거나, 쓰지 않으면 잠그거나 지웁니다.** 예제 계정(`scott`)은 운영 서버에 남기지 않습니다.
{: .prompt-warning }

계정 보안을 위해 DBMS마다 비슷한 기능을 제공합니다.

| 하는 일 | Oracle | MS-SQL | MariaDB(이 실습) |
|---|---|---|---|
| 비밀번호 변경 | `ALTER USER system IDENTIFIED BY ...` | `ALTER LOGIN sa WITH PASSWORD = '...'` | `ALTER USER ... IDENTIFIED BY '...'` |
| 관리자 권한 빼기 | `REVOKE DBA FROM user01` | `EXEC sp_dropsrvrolemember 'user01', 'sysadmin'` | `REVOKE ... FROM ...` |
| 로그인 실패 시 잠금 | `CREATE PROFILE ... LIMIT FAILED_LOGIN_ATTEMPTS 3 PASSWORD_LOCK_TIME 1` | 계정 잠금 정책 | `max_password_errors` |

| 자주 나오는 명령 | 정확한 뜻 |
|---|---|
| `EXEC sp_dropsrvrolemember 'user01', 'sysadmin'` | `user01` 을 **sysadmin 역할에서 뺌**. 계정을 지우는 것이 아님 |
| `FAILED_LOGIN_ATTEMPTS 3` | 비밀번호를 **3번 틀리면** 계정을 잠금 |
| `PASSWORD_LOCK_TIME 1` | 잠긴 계정을 **1일 동안** 잠가 둠 |

#### Ubuntu MariaDB의 root — unix_socket 인증

Ubuntu에서 설치한 MariaDB의 `root` 는 비밀번호로는 로그인할 수 없고, **리눅스의 root 권한(sudo)으로 실행한 경우에만** DB의 root로 인정합니다. 이것을 **unix_socket 인증**이라고 합니다. 원격에서 root 비밀번호를 대입하는 공격이 통하지 않습니다.

### [실습 1] (Ubuntu) 기본 계정과 root 인증 방식 확인

```bash
# === Ubuntu ===
sudo mariadb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
SELECT User, Host, plugin FROM mysql.user;
SHOW CREATE USER 'root'@'localhost';
exit
```

```
MariaDB [(none)]> SELECT User, Host, plugin FROM mysql.user;
+-------------+-----------+-----------------------+
| User        | Host      | plugin                |
+-------------+-----------+-----------------------+
| mariadb.sys | localhost | mysql_native_password |
| root        | localhost | mysql_native_password |
| mysql       | localhost | mysql_native_password |
+-------------+-----------+-----------------------+
3 rows in set (0.001 sec)

MariaDB [(none)]> SHOW CREATE USER 'root'@'localhost';
+----------------------------------------------------------------------------------------------------+
| CREATE USER for root@localhost                                                                     |
+----------------------------------------------------------------------------------------------------+
| CREATE USER `root`@`localhost` IDENTIFIED VIA mysql_native_password USING 'invalid' OR unix_socket |
+----------------------------------------------------------------------------------------------------+
1 row in set (0.000 sec)

MariaDB [(none)]> exit
Bye
```

| 명령 | 설명 |
|---|---|
| `SELECT User, Host, plugin FROM mysql.user;` | DB 계정 목록. `Host` 는 그 계정이 **어디서 접속할 수 있는지** |
| `SHOW CREATE USER '계정'@'호스트';` | 계정이 어떤 인증 방식으로 만들어졌는지 표시 |

| 화면 | 의미 |
|---|---|
| 계정 3개, 모두 `localhost` | 기본 계정은 **DB 서버 자신에서만** 접속 가능. 이름 없는 익명 계정(`''`)도 없음 |
| `mariadb.sys`, `mysql` | DBMS 내부용 계정 |
| `USING 'invalid'` | root의 비밀번호 자리에 **절대 맞을 수 없는 값**이 들어 있음 → 비밀번호로는 로그인 불가 |
| `OR unix_socket` | 리눅스 root(sudo)로 실행했을 때만 로그인 허용 |

일반 사용자(`student`)로 DB의 root에 들어가 봅니다.

```bash
# === Ubuntu ===
mariadb -u root
```

```
ERROR 1698 (28000): Access denied for user 'root'@'localhost'
```

| 명령·옵션 | 설명 |
|---|---|
| `mariadb -u root` | `sudo` 없이, DB 계정 `root` 로 접속 시도(`-u`, user) |

> `sudo` 없이는 DB의 root로 들어갈 수 없습니다. 리눅스 계정 관리(리눅스 기초 3주차)가 그대로 DB 관리자 보호로 이어집니다.
{: .prompt-tip }

---

## 2.2 권한 부여·회수 — GRANT와 REVOKE

### [이론]

**인증**이 "누구인가"를 확인한다면, **인가(권한)** 는 "무엇을 할 수 있는가"를 정합니다. DB에서는 DCL인 `GRANT` 와 `REVOKE` 로 권한을 관리합니다.

```
GRANT  권한 ON 객체 TO 사용자 [WITH GRANT OPTION];
REVOKE [GRANT OPTION FOR] 권한 ON 객체 FROM 사용자 [CASCADE | RESTRICT];
```

| 항목 | 뜻 |
|---|---|
| 권한 | **SELECT**(읽기), **INSERT**(추가), **UPDATE**(수정), **DELETE**(삭제), ALTER·DROP(구조 변경·삭제), ALL |
| 객체 | 권한을 줄 대상. `DB이름.테이블` (예: `companydb.employees`), `DB이름.*`(DB 전체) |
| **WITH GRANT OPTION** | 받은 권한을 **다른 사용자에게 다시 줄 수 있는** 권한까지 함께 줌 |
| GRANT OPTION FOR | 권한 자체는 두고, **다시 줄 수 있는 권한만** 회수 |
| **CASCADE** | 회수할 때, 그 사용자가 **다른 사람에게 준 권한도 연쇄적으로** 회수 |
| **RESTRICT** | 그 사용자가 다른 사람에게 준 권한이 있으면 **회수를 거부**(오류) |

> 예: "직원 `User01` 이 만든 `Student` 테이블의 레코드를 동료 `Staff01` 이 **읽을 수 있게**" 하려면 `GRANT SELECT ON Student TO Staff01;` 입니다. 읽기 권한의 이름은 `READ` 가 아니라 **`SELECT`**, 부여 명령은 `ALLOW` 가 아니라 **`GRANT`** 입니다. 다시 줄 수 있게 하라는 조건이 없으므로 `WITH GRANT OPTION` 은 붙이지 않습니다.
{: .prompt-warning }

#### 최소 권한 원칙

웹 애플리케이션이 쓰는 DB 계정에 모든 권한(`ALL`)을 주면, SQL 인젝션 같은 공격 한 번으로 **테이블 삭제(DROP)까지** 당할 수 있습니다. 그래서 **꼭 필요한 테이블에, 꼭 필요한 권한만** 줍니다.

| 계정 | 하는 일 | 줄 권한 |
|---|---|---|
| 관리자 | DB 설치·백업·계정 관리 | 전체(평소에는 쓰지 않음) |
| 웹 애플리케이션 | 직원 목록 조회 | 필요한 테이블에 **SELECT만** |
| 인사 담당 프로그램 | 직원 추가·수정 | `employees` 에 SELECT·INSERT·UPDATE |

### [실습 1] (Ubuntu) 최소 권한 계정 만들기

```bash
# === Ubuntu ===
sudo mariadb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'apppass123';
GRANT SELECT ON companydb.employees TO 'appuser'@'localhost';
SHOW GRANTS FOR 'appuser'@'localhost';
exit
```

```
MariaDB [(none)]> CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'apppass123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT SELECT ON companydb.employees TO 'appuser'@'localhost';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> SHOW GRANTS FOR 'appuser'@'localhost';
+----------------------------------------------------------------------------------------------------------------+
| Grants for appuser@localhost                                                                                   |
+----------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `appuser`@`localhost` IDENTIFIED BY PASSWORD '*3A57A62328B87F30CEDF03D383E074E57046ACBB' |
| GRANT SELECT ON `companydb`.`employees` TO `appuser`@`localhost`                                               |
+----------------------------------------------------------------------------------------------------------------+
2 rows in set (0.000 sec)

MariaDB [(none)]> exit
Bye
```

| 명령 | 설명 |
|---|---|
| `CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'apppass123';` | DB 서버 자신(`localhost`)에서만 접속할 수 있는 계정 `appuser` 를 비밀번호 `apppass123` 으로 만듦 |
| `GRANT SELECT ON companydb.employees TO ...;` | `employees` 테이블 **읽기만** 허용 |
| `SHOW GRANTS FOR ...;` | 그 계정이 가진 권한 목록 |

| 화면 | 의미 |
|---|---|
| `GRANT USAGE ON *.*` | "접속만 할 수 있음"(아무 권한 없음)이라는 기본 줄 |
| `IDENTIFIED BY PASSWORD '*3A57...'` | 비밀번호는 원문이 아니라 **해시값**으로 저장됨 |
| `GRANT SELECT ON companydb.employees` | 방금 준 권한 |

### [실습 2] (Ubuntu) 권한 안의 작업과 권한 밖의 작업

이번에는 `appuser` 로 접속합니다. 비밀번호를 물으면 `apppass123` 을 입력합니다(화면에 표시되지 않음).

```bash
# === Ubuntu ===
mariadb -u appuser -p companydb
```

```sql
-- === Ubuntu: MariaDB console (appuser) ===
SELECT name, salary FROM employees WHERE dept_id = 20;
INSERT INTO employees VALUES (7,'Yoon','yoon@lab.test',10,4000);
DROP TABLE employees;
exit
```

```
Enter password: 
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 42
Server version: 10.11.14-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [companydb]> SELECT name, salary FROM employees WHERE dept_id = 20;
+------+--------+
| name | salary |
+------+--------+
| Park |   5100 |
| Choi |   4800 |
| Jung |   4500 |
+------+--------+
3 rows in set (0.001 sec)

MariaDB [companydb]> INSERT INTO employees VALUES (7,'Yoon','yoon@lab.test',10,4000);
ERROR 1142 (42000): INSERT command denied to user 'appuser'@'localhost' for table `companydb`.`employees`
MariaDB [companydb]> DROP TABLE employees;
ERROR 1142 (42000): DROP command denied to user 'appuser'@'localhost' for table `companydb`.`employees`
MariaDB [companydb]> exit
Bye
```

| 명령·옵션 | 설명 |
|---|---|
| `-u appuser` | DB 계정 `appuser` 로 접속 |
| `-p` | 비밀번호를 물어보게 함 |
| `companydb` | 접속하자마자 쓸 데이터베이스 |

| 화면 | 의미 |
|---|---|
| `SELECT` 결과 3행 | 받은 권한(SELECT) 안의 작업은 **성공** |
| `INSERT command denied` | 추가 권한이 없어 **거부** |
| `DROP command denied` | 테이블 삭제도 **거부**. 이 계정을 공격자가 빼앗아도 테이블을 지울 수 없음 |
| `Reading table information ...` | 콘솔이 자동 완성용으로 테이블 정보를 읽는다는 안내. 오류 아님 |

> 그런데 `appuser` 는 **급여(`salary`)까지** 볼 수 있습니다. 웹에서 직원 목록만 보여 주면 되는데 급여까지 읽을 수 있는 것은 여전히 권한이 너무 넓습니다. 다음 절에서 **뷰**로 해결합니다.
{: .prompt-warning }

---

## 2.3 뷰(View)로 보여 줄 열만 허용

### [이론]

**뷰(View)** 는 하나 이상의 테이블에서 원하는 열과 행만 골라 만든 **가상 테이블**입니다. 데이터를 따로 저장하지 않고, 조회할 때마다 원래 테이블에서 가져옵니다.

| 뷰의 보안 효과 | 설명 |
|---|---|
| **열 단위 접근 제어** | 급여 같은 민감한 열을 빼고 보여 줌 |
| **행 단위 접근 제어** | `WHERE` 로 자기 부서 직원만 보이게 함 |
| 구조 숨김 | 사용자는 원래 테이블의 이름과 구조를 몰라도 됨 |

> 사용자에게 원래 테이블이 아니라 **뷰에만 권한**을 주면, 사용자는 뷰에 보이는 것만 볼 수 있습니다. "가상 테이블을 만들어 사용자별로 볼 수 있는 열을 정하는 것"이 바로 **뷰**입니다.
{: .prompt-tip }

### [실습 1] (Ubuntu) 급여를 뺀 뷰 만들고 뷰에만 권한 주기

```bash
# === Ubuntu ===
sudo mariadb companydb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
REVOKE SELECT ON companydb.employees FROM 'appuser'@'localhost';
CREATE VIEW emp_public AS
  SELECT e.emp_id, e.name, e.email, d.dept_name
  FROM employees e JOIN departments d ON e.dept_id = d.dept_id;
GRANT SELECT ON companydb.emp_public TO 'appuser'@'localhost';
SHOW GRANTS FOR 'appuser'@'localhost';
exit
```

```
MariaDB [companydb]> REVOKE SELECT ON companydb.employees FROM 'appuser'@'localhost';
Query OK, 0 rows affected (0.001 sec)

MariaDB [companydb]> CREATE VIEW emp_public AS
    ->   SELECT e.emp_id, e.name, e.email, d.dept_name
    ->   FROM employees e JOIN departments d ON e.dept_id = d.dept_id;
Query OK, 0 rows affected (0.003 sec)

MariaDB [companydb]> GRANT SELECT ON companydb.emp_public TO 'appuser'@'localhost';
Query OK, 0 rows affected (0.001 sec)

MariaDB [companydb]> SHOW GRANTS FOR 'appuser'@'localhost';
+----------------------------------------------------------------------------------------------------------------+
| Grants for appuser@localhost                                                                                   |
+----------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `appuser`@`localhost` IDENTIFIED BY PASSWORD '*3A57A62328B87F30CEDF03D383E074E57046ACBB' |
| GRANT SELECT ON `companydb`.`emp_public` TO `appuser`@`localhost`                                              |
+----------------------------------------------------------------------------------------------------------------+
2 rows in set (0.000 sec)

MariaDB [companydb]> exit
Bye
```

| 명령 | 설명 |
|---|---|
| `sudo mariadb companydb` | 관리자로 접속하면서 바로 `companydb` 를 씀 |
| `REVOKE SELECT ON companydb.employees FROM ...;` | 2.2절에서 준 `employees` 읽기 권한을 **회수** |
| `CREATE VIEW emp_public AS SELECT ...;` | 직원 번호·이름·이메일·부서 이름만 담은 뷰를 만듦. **급여는 뺌**. 뷰 만들기는 DDL |
| `JOIN departments d ON e.dept_id = d.dept_id` | 두 테이블을 부서 번호로 이어 붙여 부서 **이름**을 가져옴. `e`, `d` 는 테이블의 짧은 별명 |
| `GRANT SELECT ON companydb.emp_public TO ...;` | **뷰에만** 읽기 권한을 줌 |

| 화면 | 의미 |
|---|---|
| `GRANT SELECT ON companydb.emp_public` | `employees` 권한은 사라지고 뷰 권한만 남음 |

### [실습 2] (Ubuntu) appuser로 확인

```bash
# === Ubuntu ===
mariadb -u appuser -p companydb
```

```sql
-- === Ubuntu: MariaDB console (appuser) ===
SHOW TABLES;
SELECT * FROM emp_public;
SELECT name, salary FROM employees;
exit
```

```
Enter password: 
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 44
Server version: 10.11.14-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [companydb]> SHOW TABLES;
+---------------------+
| Tables_in_companydb |
+---------------------+
| emp_public          |
+---------------------+
1 row in set (0.000 sec)

MariaDB [companydb]> SELECT * FROM emp_public;
+--------+------+---------------+-----------+
| emp_id | name | email         | dept_name |
+--------+------+---------------+-----------+
|      1 | Kim  | kim@lab.test  | Sales     |
|      2 | Lee  | lee@lab.test  | Sales     |
|      3 | Park | park@lab.test | Dev       |
|      4 | Choi | choi@lab.test | Dev       |
|      5 | Jung | jung@lab.test | Dev       |
|      6 | Kang | kang@lab.test | Security  |
+--------+------+---------------+-----------+
6 rows in set (0.000 sec)

MariaDB [companydb]> SELECT name, salary FROM employees;
ERROR 1142 (42000): SELECT command denied to user 'appuser'@'localhost' for table `companydb`.`employees`
MariaDB [companydb]> exit
Bye
```

| 화면 | 의미 |
|---|---|
| `SHOW TABLES` 에 `emp_public` 만 보임 | 권한이 없는 테이블(`employees`, `departments`)은 **존재 자체가 보이지 않음** |
| `emp_public` 에 급여 열이 없음 | 뷰로 **열 단위 접근 제어** 성공 |
| `SELECT command denied ... employees` | 원래 테이블 직접 조회는 **거부** |

| 단계 | `appuser` 가 볼 수 있는 것 |
|---|---|
| 2.2절 | `employees` 전체(급여 포함) |
| 2.3절 | `emp_public` 뷰(급여 없음) |

---

## 2.4 로그인 실패 잠금

### [실습] (Ubuntu) 비밀번호를 3번 틀리면 잠그기

2.1절의 Oracle `FAILED_LOGIN_ATTEMPTS 3` 과 같은 기능을 MariaDB의 `max_password_errors` 로 시험합니다. 관리자 콘솔에서 값을 바꿉니다.

```bash
# === Ubuntu ===
sudo mariadb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
SHOW VARIABLES LIKE 'max_password_errors';
SET GLOBAL max_password_errors = 3;
exit
```

```
MariaDB [(none)]> SHOW VARIABLES LIKE 'max_password_errors';
+---------------------+------------+
| Variable_name       | Value      |
+---------------------+------------+
| max_password_errors | 4294967295 |
+---------------------+------------+
1 row in set (0.001 sec)

MariaDB [(none)]> SET GLOBAL max_password_errors = 3;
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> exit
Bye
```

| 명령 | 설명 |
|---|---|
| `SHOW VARIABLES LIKE '...';` | 서버 설정값 확인 |
| `SET GLOBAL max_password_errors = 3;` | 비밀번호를 **연속 3번** 틀린 계정을 잠금. 서버를 다시 시작하면 원래 값으로 돌아감 |

| 화면 | 의미 |
|---|---|
| `4294967295` | 기본값. 사실상 **무제한**(잠그지 않음) → 비밀번호 대입 공격을 막지 못함 |

이제 `appuser` 의 비밀번호를 일부러 세 번 틀린 뒤, 네 번째에 맞는 비밀번호를 넣습니다. 한 줄씩 실행합니다.

```bash
# === Ubuntu ===
mariadb -u appuser -pwrongpass -e 'SELECT 1'
mariadb -u appuser -pwrongpass -e 'SELECT 1'
mariadb -u appuser -pwrongpass -e 'SELECT 1'
mariadb -u appuser -papppass123 -e 'SELECT CURRENT_USER()'
```

```
ERROR 1045 (28000): Access denied for user 'appuser'@'localhost' (using password: YES)
ERROR 1045 (28000): Access denied for user 'appuser'@'localhost' (using password: YES)
ERROR 1045 (28000): Access denied for user 'appuser'@'localhost' (using password: YES)
ERROR 4150 (HY000): User is blocked because of too many credential errors; unblock with 'ALTER USER / FLUSH PRIVILEGES'
```

| 명령·옵션 | 설명 |
|---|---|
| `-pwrongpass` | `-p` 바로 뒤에 **띄어쓰기 없이** 비밀번호를 붙여 묻지 않고 바로 접속. 실습용이며, 실제로는 비밀번호가 명령 기록에 남으므로 쓰지 않음 |
| `-e 'SELECT 1'` | 콘솔에 들어가지 않고 SQL 한 줄만 실행하고 끝냄(execute) |

| 화면 | 의미 |
|---|---|
| `ERROR 1045 ... Access denied` × 3 | 비밀번호 틀림 |
| `ERROR 4150 ... User is blocked` | 네 번째에는 **맞는 비밀번호인데도 거부**. 계정이 잠김 |

관리자가 잠금을 풀고, 다시 접속해 봅니다.

```bash
# === Ubuntu ===
sudo mariadb -e 'FLUSH PRIVILEGES'
mariadb -u appuser -papppass123 -e 'SELECT CURRENT_USER()'
```

```
+-------------------+
| CURRENT_USER()    |
+-------------------+
| appuser@localhost |
+-------------------+
```

| 명령 | 설명 |
|---|---|
| `FLUSH PRIVILEGES` | 계정·권한 정보를 다시 읽음. 이때 잠긴 계정이 풀림(첫 줄은 출력 없음) |
| `SELECT CURRENT_USER()` | 지금 접속한 계정 이름 |

실습이 끝났으면 잠금 설정을 원래대로 돌립니다(출력 없음).

```bash
# === Ubuntu ===
sudo mariadb -e 'SET GLOBAL max_password_errors = 4294967295'
```

> 로그인 실패 잠금은 **비밀번호 대입(무차별 대입) 공격**을 막는 대표적인 방법입니다. 대신 공격자가 일부러 틀려서 정상 사용자를 잠그는 **서비스 거부**에 악용될 수 있어, 실제로는 잠금 시간을 정해 두거나 관리자 알림과 함께 씁니다.
{: .prompt-info }

---

## 2.5 관리자 권한 제한, 암호 정책, 역할(RBAC)

### [이론]

**접근통제**는 사용자가 가진 **접근 권한(읽기, 쓰기 등)에 따라 논리적인 DB를 나누어 통제**하는 것입니다. 2.2~2.3절의 `GRANT`·`REVOKE`·뷰가 모두 접근통제 수단입니다. DBMS를 설치한 뒤에는 다음도 점검합니다.

| 점검 항목 | 방법 |
|---|---|
| **관리자(DBA) 권한 제한** | 불필요한 DBA 권한을 제거합니다. 관리자 권한은 여러 사람에게 **분산**하는 것이 아니라 **제한**합니다 |
| 공용 관리자 계정 | 여러 명이 한 관리자 계정을 같이 쓰지 않고, **사용자별 계정**을 만들어 누가 했는지 추적합니다 |
| 응용 프로그램 계정 | 응용 프로그램이 DBA 계정을 쓰지 않고 **일반 계정 권한**만 갖게 합니다(2.2절) |
| 기본·guest 계정 | 쓰지 않는 기본 계정과 guest 계정은 막거나 지웁니다(2.1절) |
| **복잡한 암호** | 모든 계정에 길이와 문자 종류 조건을 걸어 약한 암호를 거부합니다 |
| 정기 백업 | 장애와 침해에 대비해 데이터를 정기적으로 백업합니다 |

계정마다 권한을 하나씩 주면 사람이 늘수록 관리가 어렵습니다. **역할(Role)** 은 권한 묶음에 이름을 붙여 두고 사용자에게는 역할만 주는 방식이며, 2.6절의 **RBAC**(역할 기반 접근통제)에 해당합니다. MS-SQL의 고정 데이터베이스 역할과 이 실습의 역할은 다음처럼 대응합니다.

| MS-SQL 역할 | 뜻 | 이 실습에서 |
|---|---|---|
| `db_owner` | **해당 DB**의 모든 유지관리·구성 작업 | `GRANT ALL ON companydb.*` 에 해당 |
| `db_datareader` | 해당 DB의 모든 사용자 테이블 **읽기** | `hr_reader` (`SELECT`) |
| `db_datawriter` | **해당 DB의** 모든 사용자 테이블에 데이터 **추가·수정·삭제**(모든 DB가 아님) | `hr_writer` (`INSERT`·`UPDATE`·`DELETE`) |
| `sysadmin`(서버 역할) | 서버 전체 관리 | `SUPER` 같은 전체(`*.*`) 권한 |

> `db_datawriter` 는 **쓰기 역할일 뿐 읽기 권한을 포함하지 않습니다.** 실습 3에서 `hr_writer` 로 `SELECT` 가 거부되는 것을 확인합니다.
{: .prompt-warning }

### [실습 1] (Ubuntu) 관리자급 계정 점검

서버 전체를 관리할 수 있는 `SUPER` 권한을 가진 계정을 찾습니다.

```bash
# === Ubuntu ===
sudo mariadb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
SELECT GRANTEE, PRIVILEGE_TYPE FROM information_schema.USER_PRIVILEGES WHERE PRIVILEGE_TYPE = 'SUPER';
```

```
MariaDB [(none)]> SELECT GRANTEE, PRIVILEGE_TYPE FROM information_schema.USER_PRIVILEGES WHERE PRIVILEGE_TYPE = 'SUPER';
+--------------------+----------------+
| GRANTEE            | PRIVILEGE_TYPE |
+--------------------+----------------+
| 'root'@'localhost' | SUPER          |
+--------------------+----------------+
1 row in set (0.001 sec)
```

| 화면 | 의미 |
|---|---|
| `'root'@'localhost'` 한 줄 | 관리자 권한을 가진 계정이 **root 하나뿐**이고, 그것도 서버 자신(`localhost`)에서만 접속 가능. 1.2~2.4절에서 만든 `appuser` 는 목록에 없음 |

> 이 목록에 **모르는 계정**이나 응용 프로그램 계정이 보이면 관리자 권한을 회수합니다. 같은 서버를 여러 사람이 관리해야 한다면 root를 같이 쓰지 말고 사람마다 계정을 만들어 권한을 나누어 줍니다.
{: .prompt-tip }

### [실습 2] (Ubuntu) 약한 암호 거부

MariaDB에 들어 있는 `simple_password_check` 플러그인으로 암호 조건을 겁니다. 같은 콘솔에서 이어서 입력합니다.

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
INSTALL SONAME 'simple_password_check';
SHOW VARIABLES LIKE 'simple_password_check%';
CREATE USER 'hr_kim'@'localhost' IDENTIFIED BY 'abc123';
CREATE USER 'hr_kim'@'localhost' IDENTIFIED BY 'Summer#2026x';
```

```
MariaDB [(none)]> INSTALL SONAME 'simple_password_check';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> SHOW VARIABLES LIKE 'simple_password_check%';
+-----------------------------------------+-------+
| Variable_name                           | Value |
+-----------------------------------------+-------+
| simple_password_check_digits            | 1     |
| simple_password_check_letters_same_case | 1     |
| simple_password_check_minimal_length    | 8     |
| simple_password_check_other_characters  | 1     |
+-----------------------------------------+-------+
4 rows in set (0.000 sec)

MariaDB [(none)]> CREATE USER 'hr_kim'@'localhost' IDENTIFIED BY 'abc123';
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements (simple_password_check)
MariaDB [(none)]> CREATE USER 'hr_kim'@'localhost' IDENTIFIED BY 'Summer#2026x';
Query OK, 0 rows affected (0.001 sec)
```

| 명령·화면 | 설명 |
|---|---|
| `INSTALL SONAME 'simple_password_check';` | 암호 검사 플러그인을 불러옴 |
| `..._minimal_length 8` | 최소 **8글자** |
| `..._digits 1` | **숫자** 1개 이상 |
| `..._letters_same_case 1` | **대문자와 소문자** 각각 1개 이상 |
| `..._other_characters 1` | **특수문자** 1개 이상 |
| `ERROR 1819 ... policy requirements` | `abc123` 은 짧고 대문자·특수문자가 없어 **거부** |
| `Summer#2026x` 성공 | 네 가지 조건을 모두 채워 **계정 생성** |

> 암호 조건은 계정을 **새로 만들거나 암호를 바꿀 때** 검사합니다. 이미 있는 `appuser` 의 암호(`apppass123`)는 영향을 받지 않습니다.
{: .prompt-info }

### [실습 3] (Ubuntu) 역할로 권한 묶어 주기

읽기 역할 `hr_reader` 와 쓰기 역할 `hr_writer` 를 만들고, `hr_kim` 에게 읽기 역할만 줍니다. 같은 콘솔에서 이어서 입력합니다.

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
CREATE ROLE hr_reader;
CREATE ROLE hr_writer;
GRANT SELECT ON companydb.* TO hr_reader;
GRANT INSERT, UPDATE, DELETE ON companydb.* TO hr_writer;
GRANT hr_reader TO 'hr_kim'@'localhost';
SET DEFAULT ROLE hr_reader FOR 'hr_kim'@'localhost';
SHOW GRANTS FOR 'hr_kim'@'localhost';
exit
```

```
MariaDB [(none)]> CREATE ROLE hr_reader;
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> CREATE ROLE hr_writer;
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT SELECT ON companydb.* TO hr_reader;
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT INSERT, UPDATE, DELETE ON companydb.* TO hr_writer;
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> GRANT hr_reader TO 'hr_kim'@'localhost';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> SET DEFAULT ROLE hr_reader FOR 'hr_kim'@'localhost';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> SHOW GRANTS FOR 'hr_kim'@'localhost';
+---------------------------------------------------------------------------------------------------------------+
| Grants for hr_kim@localhost                                                                                   |
+---------------------------------------------------------------------------------------------------------------+
| GRANT `hr_reader` TO `hr_kim`@`localhost`                                                                     |
| GRANT USAGE ON *.* TO `hr_kim`@`localhost` IDENTIFIED BY PASSWORD '*354E96573F31B9FE9E5C22A601D4B4EA5A77E704' |
| SET DEFAULT ROLE `hr_reader` FOR `hr_kim`@`localhost`                                                         |
+---------------------------------------------------------------------------------------------------------------+
3 rows in set (0.000 sec)

MariaDB [(none)]> exit
Bye
```

| 명령 | 설명 |
|---|---|
| `CREATE ROLE 역할;` | 권한을 담을 **역할**을 만듦(계정과 비슷하지만 로그인은 못 함) |
| `GRANT 권한 ON companydb.* TO 역할;` | 권한을 사용자가 아니라 **역할에** 줌. `companydb.*` 는 `companydb` 의 모든 테이블 |
| `GRANT 역할 TO 사용자;` | 사용자에게 역할을 줌 |
| `SET DEFAULT ROLE ... FOR ...;` | 접속하면 **자동으로 켜질 역할**을 정함 |

`hr_kim` 으로 접속해 읽기와 쓰기를 시험합니다. 암호는 `Summer#2026x` 입니다.

```bash
# === Ubuntu ===
mariadb -u hr_kim -p companydb
```

```sql
-- === Ubuntu: MariaDB console (hr_kim) ===
SELECT CURRENT_ROLE();
SELECT COUNT(*) FROM employees;
INSERT INTO employees VALUES (9,'Test','test@lab.test',10,1);
SET ROLE hr_writer;
exit
```

```
MariaDB [companydb]> SELECT CURRENT_ROLE();
+----------------+
| CURRENT_ROLE() |
+----------------+
| hr_reader      |
+----------------+
1 row in set (0.000 sec)

MariaDB [companydb]> SELECT COUNT(*) FROM employees;
+----------+
| COUNT(*) |
+----------+
|        6 |
+----------+
1 row in set (0.000 sec)

MariaDB [companydb]> INSERT INTO employees VALUES (9,'Test','test@lab.test',10,1);
ERROR 1142 (42000): INSERT command denied to user 'hr_kim'@'localhost' for table `companydb`.`employees`
MariaDB [companydb]> SET ROLE hr_writer;
ERROR 1959 (OP000): Invalid role specification `hr_writer`
MariaDB [companydb]> exit
Bye
```

| 화면 | 의미 |
|---|---|
| `hr_reader` | 접속하면 읽기 역할이 자동으로 켜짐 |
| `COUNT(*)` 6 | 읽기는 **성공** |
| `INSERT command denied` | 읽기 역할에는 쓰기 권한이 없어 **거부** |
| `Invalid role specification` | `hr_kim` 은 `hr_writer` 를 **받지 못했으므로** 스스로 켤 수 없음 |

관리자가 쓰기 역할을 추가로 줍니다.

```bash
# === Ubuntu ===
sudo mariadb -e "GRANT hr_writer TO 'hr_kim'@'localhost'"
mariadb -u hr_kim -p companydb
```

```sql
-- === Ubuntu: MariaDB console (hr_kim) ===
SET ROLE hr_writer;
SELECT CURRENT_ROLE();
INSERT INTO employees VALUES (9,'Test','test@lab.test',10,1);
SELECT COUNT(*) FROM employees;
SET ROLE hr_reader;
SELECT COUNT(*) FROM employees;
exit
```

```
MariaDB [companydb]> SET ROLE hr_writer;
Query OK, 0 rows affected (0.000 sec)

MariaDB [companydb]> SELECT CURRENT_ROLE();
+----------------+
| CURRENT_ROLE() |
+----------------+
| hr_writer      |
+----------------+
1 row in set (0.000 sec)

MariaDB [companydb]> INSERT INTO employees VALUES (9,'Test','test@lab.test',10,1);
Query OK, 1 row affected (0.001 sec)

MariaDB [companydb]> SELECT COUNT(*) FROM employees;
ERROR 1142 (42000): SELECT command denied to user 'hr_kim'@'localhost' for table `companydb`.`employees`
MariaDB [companydb]> SET ROLE hr_reader;
Query OK, 0 rows affected (0.000 sec)

MariaDB [companydb]> SELECT COUNT(*) FROM employees;
+----------+
| COUNT(*) |
+----------+
|        7 |
+----------+
1 row in set (0.000 sec)

MariaDB [companydb]> exit
Bye
```

| 화면 | 의미 |
|---|---|
| `SET ROLE hr_writer` 성공 | 받은 역할은 사용자가 필요할 때 **전환**해서 씀 |
| `INSERT` 성공 | 쓰기 역할로 **추가 성공** |
| `SELECT command denied` | 쓰기 역할에는 읽기 권한이 없어 **조회 거부**. 한 번에 켜지는 역할은 **하나**이므로 `SET ROLE` 은 앞 역할을 **대체**함 |
| `hr_reader` 로 되돌린 뒤 7 | 읽기 역할로 돌아오면 조회됨. 방금 넣은 행이 보임 |

실습이 끝났으면 **관리자**가 시험 행과 실습용 계정·역할·플러그인을 지웁니다. 05-2의 통계 실습이 6명 기준이므로 시험 행은 반드시 지웁니다.

```bash
# === Ubuntu ===
sudo mariadb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
DELETE FROM companydb.employees WHERE emp_id = 9;
DROP USER 'hr_kim'@'localhost';
DROP ROLE hr_reader, hr_writer;
UNINSTALL SONAME 'simple_password_check';
SELECT COUNT(*) FROM companydb.employees;
exit
```

```
MariaDB [(none)]> DELETE FROM companydb.employees WHERE emp_id = 9;
Query OK, 1 row affected (0.001 sec)

MariaDB [(none)]> DROP USER 'hr_kim'@'localhost';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> DROP ROLE hr_reader, hr_writer;
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> UNINSTALL SONAME 'simple_password_check';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> SELECT COUNT(*) FROM companydb.employees;
+----------+
| COUNT(*) |
+----------+
|        6 |
+----------+
1 row in set (0.000 sec)

MariaDB [(none)]> exit
Bye
```

> 역할을 쓰면 직원이 바뀔 때마다 권한을 일일이 고치지 않고 **역할만 주거나 회수**하면 됩니다.
{: .prompt-tip }

---

## 2.6 DB 보안 정책

### [이론]

DB 보안 정책은 회사 전체 정보보안 정책의 일부입니다. 정책을 만들고 운영할 때 다음을 고려합니다.

| 고려 사항 | 내용 |
|---|---|
| 승인 | **최고경영자의 승인**을 받아야 함 |
| 배포 | 모든 임직원에게 알맞은 방법으로 배포하고, **최신 버전**을 언제든 볼 수 있어야 함 |
| 이해 | 모든 임직원이 목적과 내용을 이해하도록 **교육·홍보** |
| 검토 | **정기적으로, 그리고 필요할 때마다** 검토하고 고침 |
| 책임 | 방향, 책임 할당, 승인 절차 등을 정하는 전담 조직 |

접근통제의 큰 틀은 다음 세 가지 모델로 나눕니다.

| 모델 | 누가 권한을 정하나 | DB에서의 예 |
|---|---|---|
| **DAC**(임의적 접근통제) | **객체 소유자**가 다른 사용자에게 권한을 줌 | `GRANT`·`REVOKE`(이 글의 실습) |
| **MAC**(강제적 접근통제) | 시스템이 **보안 등급**(비밀·대외비 등)에 따라 강제 | 다단계 보안 DB. 05-2의 다중 인스턴스화 |
| **RBAC**(역할 기반 접근통제) | **역할(직무)** 에 권한을 주고, 사용자는 역할을 받음 | MS-SQL `sysadmin` 같은 역할, MariaDB `CREATE ROLE`(2.5절 실습) |

DB 보안 대책은 성격에 따라 세 가지로 나눕니다.

| 구분 | 내용 | 예 |
|---|---|---|
| **관리적** 대책 | 사람과 조직이 지킬 규칙 | **보안 표준지침**, 정책, 교육, 책임 분담 |
| 물리적 대책 | 장비가 있는 장소의 보호 | 서버실 출입통제, 백업 매체 보관 |
| 기술적 대책 | 시스템 기능으로 막기 | 접근통제, 암호화, 방화벽·IDS, 감사 로그 |

> 가장 **강력하고 포괄적**인 방안은 기술적 수단(암호화, 접근통제, 방화벽)이 아니라 **보안 표준지침을 세우는 관리적 대책**입니다. 기술적 수단도 규칙과 책임이 없으면 제대로 운영되지 않기 때문입니다.
{: .prompt-warning }

---

# 확인 문제

> 1. 데이터베이스 사용자 `User01` 이 `Student` 테이블을 만든 뒤, 동료 `Staff01` 에게 그 테이블의 레코드를 **읽을 수 있는** 권한을 주려고 한다. 알맞은 SQL은?
>    ① ALLOW READ ON STUDENT TO STAFF01
>    ② GRANT READ ON STUDENT TO STAFF01
>    ③ ALLOW SELECT ON STUDENT TO STAFF01
>    ④ GRANT SELECT ON STUDENT TO STAFF01
{: .prompt-tip }

**정답:** **④** 읽기 권한은 SELECT이고 권한을 주는 명령은 GRANT이다. ①·③은 ALLOW라는 SQL 명령이 없고, ②는 READ라는 권한 이름이 없어 틀린다. 다시 줄 수 있게 하라는 조건이 없으므로 WITH GRANT OPTION은 붙이지 않는다(2.2절).

> 2. 다음 설명에 해당하는 키는?
>    - 여러 후보키 가운데 기본키로 선정되고 남은 나머지 키
>    - 기본키를 `<학번>` 으로 선정하였다면 `<이름, 학과>` 를 가리킴
>    ① Candidate Key
>    ② Primary Key
>    ③ Alternate Key
>    ④ Foreign Key
{: .prompt-tip }

**정답:** **③** 후보키 중 기본키로 뽑히고 남은 키가 대체키(Alternate Key)이다. 학번이 기본키이면 같은 후보키였던 `<이름, 학과>` 가 대체키가 된다(1.2절).

> 3. 다음 중 DDL(데이터 정의어)에 해당하는 것은?
>    ① CREATE
>    ② SELECT
>    ③ INSERT
>    ④ UPDATE
{: .prompt-tip }

**정답:** **①** 구조를 만들고 바꾸고 지우는 CREATE·ALTER·DROP이 DDL이다. SELECT·INSERT·UPDATE는 데이터를 다루는 DML이다(1.3절).

> 4. 사용자별로 볼 수 있는 열을 정하기 위해, 하나 이상의 기본 테이블로부터 만들어 사용자에게 보여 주는 가상 테이블은?
>    ① DDL
>    ② 뷰(View)
>    ③ SQL
>    ④ DCL
{: .prompt-tip }

**정답:** **②** 하나 이상의 테이블에서 필요한 열만 골라 보여 주는 가상 테이블은 뷰이다. 사용자마다 다른 뷰를 주면 볼 수 있는 열을 제한할 수 있다. DDL·DCL은 명령의 분류이고, SQL은 언어 이름이다(2.3절).

> 5. Oracle을 설치할 때 생기는 기본 계정과 기본 비밀번호의 연결이 **틀린** 것은?
>    ① sys – change_on_install
>    ② scott – tiger
>    ③ system – manage
>    ④ dbsnmp – dbsnmp
{: .prompt-tip }

**정답:** **③** system의 기본 비밀번호는 manage가 아니라 manager이다. 나머지 sys-change_on_install, scott-tiger, dbsnmp-dbsnmp는 맞는 연결이다. 그래서 설치 직후 기본 비밀번호를 바꿔야 한다(2.1절).

> 6. DB 보안 설정 명령과 그 의미의 연결이 **틀린** 것은?
>    ① `ALTER LOGIN sa WITH PASSWORD = 'test_password'` – sa 계정의 비밀번호 변경
>    ② `EXEC sp_dropsrvrolemember 'user01', 'sysadmin'` – user01 계정과 sysadmin 계정을 삭제
>    ③ `CREATE PROFILE grace_5 LIMIT FAILED_LOGIN_ATTEMPTS 3` – 비밀번호를 3번까지 틀릴 수 있음
>    ④ `CREATE PROFILE grace_5 LIMIT PASSWORD_LOCK_TIME 1` – 잠긴 계정을 1일 동안 잠가 둠
{: .prompt-tip }

**정답:** **②** sp_dropsrvrolemember는 user01을 sysadmin 역할에서 빼는 명령이다. 계정이나 역할을 삭제하지 않으므로 틀린 설명이다. ①은 ALTER LOGIN으로 비밀번호 변경, ③은 실패 허용 횟수 3회, ④는 잠금 유지 기간 1일이라 맞다(2.1절).

> 7. `REVOKE` 로 권한을 회수할 때, 그 사용자가 다른 사용자에게 다시 부여한 권한까지 **연쇄적으로 회수**하는 옵션은?
{: .prompt-tip }

**정답:** **CASCADE** — REVOKE에 CASCADE를 붙이면 그 사용자가 다시 부여한 권한까지 연쇄적으로 회수된다. 붙이지 않으면 그 사용자의 권한만 회수된다(2.2절).

> 8. DB 보안 정책을 작성·운영할 때의 고려 사항으로 옳은 것을 **모두** 고르면?
>    (가) 최고경영자의 승인을 받아야 한다
>    (나) 모든 임직원이 최신 버전의 정책을 자유롭게 볼 수 있어야 한다
>    (다) 모든 임직원이 목적과 내용을 이해하도록 교육·홍보해야 한다
>    (라) 정기적으로 또는 필요할 때 검토하고 갱신해야 한다
{: .prompt-tip }

**정답:** **가, 나, 다, 라 모두** — 보안 정책은 경영진 승인으로 권위를 얻고(가), 최신본을 모두가 볼 수 있어야 하며(나), 이해하도록 교육해야 지켜지고(다), 환경이 바뀌므로 정기 검토·갱신이 필요하다(라)(2.6절).

> 9. 사용자가 가진 접근권한(읽기, 쓰기 등)에 따라 논리적인 DB를 분리하여 통제하는 것은?
>    ① 추론 통제
>    ② 접근통제
>    ③ 흐름 통제
>    ④ 직접 통제
{: .prompt-tip }

**정답:** **②** 접근권한에 따라 논리적 DB를 분리해 통제하는 것이 접근통제이다. 추론 통제는 낱개 정보를 모아 숨긴 정보를 알아내는 것을 막고, 흐름 통제는 정보가 허용되지 않은 곳으로 흘러가는 것을 막는다. 직접 통제는 보안 용어가 아니다(2.5절).

> 10. 다음 중 가장 강력하고 포괄적인 DB 보안 방안은?
>     ① sa 계정에 암호를 할당하고 데이터를 암호화
>     ② DBMS 접근통제
>     ③ 방화벽 및 IDS 운용
>     ④ 보안 표준지침 구축
{: .prompt-tip }

**정답:** **④** DB 보안 방안은 관리적·물리적·기술적으로 나뉜다. 계정 암호·암호화·접근통제·방화벽은 모두 기술적 대책으로 일부분만 막는다. 정책과 절차를 담은 보안 표준지침은 관리적 대책으로 가장 포괄적이다(2.6절).

> 11. 데이터베이스 시스템에서 고려해야 할 보안 방안이 **아닌** 것은?
>     ① 관리자 권한 분산
>     ② 정기적으로 모든 데이터를 백업
>     ③ guest 계정 사용 금지
>     ④ 모든 SQL Server 계정에 복잡한 암호 사용
{: .prompt-tip }

**정답:** **①** 관리자 권한은 분산이 아니라 제한해야 한다. 불필요한 DBA 권한을 없애고, 공용 DBA 계정 대신 사용자별 계정을 쓰며, 응용 프로그램은 일반 계정 권한으로 동작시킨다. ②·③·④는 권장 사항이다(2.5절).

> 12. 다음 중 MS-SQL의 `db_datawriter` 역할에 대한 설명으로 옳은 것은?
>     ① 서버의 모든 데이터베이스에서 데이터를 추가·삭제·변경할 수 있다
>     ② 해당 데이터베이스의 모든 사용자 테이블에서 데이터를 추가·삭제·변경할 수 있다
>     ③ 해당 데이터베이스에서 모든 유지관리와 구성 작업을 수행한다
>     ④ 해당 데이터베이스의 모든 사용자 테이블을 읽을 수 있다
{: .prompt-tip }

**정답:** **②** db_datawriter는 해당 DB의 모든 사용자 테이블에 대해 추가·삭제·변경을 허용한다. ①은 "해당 DB"가 아니라 "모든 DB"라서 틀리고, ③은 db_owner, ④는 db_datareader의 설명이다. 쓰기 역할에는 읽기가 포함되지 않는다(2.5절).

---

# 체크리스트

- [ ] (Ubuntu) MariaDB를 설치하고 3306 포트가 `127.0.0.1` 에서만 열린 것을 확인
- [ ] 후보키·기본키·대체키·슈퍼키·외래키를 `employees` 테이블로 설명 가능
- [ ] (Ubuntu) 중복 이메일(ERROR 1062)과 없는 부서(ERROR 1452)가 거부되는 것을 확인하고 개체·참조 무결성을 설명 가능
- [ ] DDL·DML·DCL·TCL에 속하는 명령을 구분 가능, (Ubuntu) `ROLLBACK` 으로 삭제를 되돌림
- [ ] Oracle·MS-SQL·MariaDB의 기본 계정 위험과, Ubuntu MariaDB root의 unix_socket 인증(`'invalid' OR unix_socket`, ERROR 1698)을 설명 가능
- [ ] (Ubuntu) `appuser` 에 SELECT만 주고 INSERT·DROP이 거부(ERROR 1142)되는 것을 확인
- [ ] `WITH GRANT OPTION`, `CASCADE`, `RESTRICT` 의 뜻을 설명 가능
- [ ] (Ubuntu) 급여를 뺀 뷰 `emp_public` 에만 권한을 주고, 원래 테이블 조회가 거부되는 것을 확인
- [ ] (Ubuntu) `max_password_errors` 로 3회 실패 후 잠금(ERROR 4150)과 `FLUSH PRIVILEGES` 해제를 확인
- [ ] (Ubuntu) `SUPER` 권한을 가진 계정을 조회해 관리자급 계정이 root뿐인 것을 확인, 관리자 권한은 분산이 아니라 제한한다는 점을 설명 가능
- [ ] (Ubuntu) `simple_password_check` 로 약한 암호(ERROR 1819)가 거부되는 것을 확인
- [ ] (Ubuntu) 읽기·쓰기 역할을 만들어 `hr_kim` 에게 주고, 쓰기 역할에는 읽기 권한이 없는 것(ERROR 1142)을 확인한 뒤 실습 계정·역할·시험 행을 정리
- [ ] 관리적·물리적·기술적 대책을 구분하고, 가장 포괄적인 것이 보안 표준지침(관리적)임을 설명 가능

> 다음 글 **05-2** 에서는 이 `companydb` 를 그대로 이어 써서, 권한을 지켜도 정보가 새는 **집성·추론** 공격과, DB 암호화·감사 같은 보안 대책을 실습합니다. 데이터베이스와 `appuser` 계정을 **지우지 마세요.**
{: .prompt-info }
