---
title: "[애플리케이션 보안] 05-2. 데이터베이스 보안 위협과 대책"
date: 2026-10-21 10:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - DB보안
tags:
  - Aggregation
  - Inference
  - Polyinstantiation
  - DB암호화
  - DB감사
  - MariaDB
math: false
mermaid: false
---

> **05-1** 에서 만든 `companydb`(테이블 `departments`·`employees`, 뷰 `emp_public`)와 계정 `appuser` 가 있어야 합니다.
{: .prompt-info }

## 0. 실습 환경

| 장비 | IP | 역할 |
|---|---|---|
| Ubuntu | **192.168.0.30** | DB 서버(MariaDB) |
| Kali | **192.168.0.10** | 4.4절에서 DB 서버 포트에 원격 접속을 시도 |

> 명령과 예상 화면은 **Ubuntu 24.04.4 LTS(MariaDB 10.11.14)** 와 **Kali Rolling** 에서 실행해 확인했습니다. 실행 시간(`(0.001 sec)`), 접속 번호, 감사 로그의 날짜·시각·번호는 실행할 때마다 다릅니다. 관리자 콘솔(`sudo mariadb`)의 안내문(`Welcome to ...`)은 생략합니다.
{: .prompt-warning }

이 글의 실습은 05-1처럼 **허용된 범위 안에서 정보가 새는 것을 먼저 보고, 설정을 바꿔 막는** 순서입니다.

| 절 | 할 일 |
|---|---|
| 3.2 집성 | 지점별 매출(공개)을 더해 회사 전체 매출(기밀)을 알아냄 |
| 3.3 추론 | 부서별 평균 급여(공개)로 한 사람의 급여를 알아냄 → 질의 결과 크기 제한으로 막음 |
| 4.2 암호화 | 카드번호를 AES로 암호화해 저장, 비밀번호는 해시로 저장 |
| 4.3 감사 | 감사 플러그인으로 로그인 실패·권한 밖 조회를 기록 |
| 4.4 네트워크 접근 제한 | Kali에서 DB 포트(3306)에 접속이 거부되는 것 확인 |

---

# 3. DB 보안 위협

## 3.1 DB 보안 통제와 위협 한눈에 보기

### [이론]

DB를 지키는 **보안 통제**는 크게 세 가지입니다.

| 보안 통제 | 막는 것 | 방법 |
|---|---|---|
| **접근 통제**(Access Control) | 허가받지 않은 사용자의 접근 | 인증, `GRANT`·`REVOKE`, 뷰(05-1) |
| **추론 통제**(Inference Control) | 공개된 값으로 비밀 값을 **간접적으로 알아내는 것** | 질의 결과 크기 제한, 다중 인스턴스화 등(3.3절) |
| **흐름 통제**(Flow Control) | 정보가 **높은 보안 등급에서 낮은 등급으로** 흘러가는 것 | 보안 등급 사이의 정보 이동 검사(3.4절) |

DB를 노리는 위협은 다음과 같습니다.

| 위협 | 내용 |
|---|---|
| **집성**(Aggregation, 집합성) | 하나하나는 공개된 정보를 **모아서** 더 높은 등급의 정보를 만듦(3.2절) |
| **추론**(Inference) | 공개된 정보나 통계로부터 **비밀 정보를 추측**(3.3절) |
| 비인가 접근·권한 남용 | 기본 계정, 약한 비밀번호, 필요 이상의 권한(05-1) |
| **SQL 인젝션** | 웹 입력값에 SQL을 끼워 넣어 DB를 조작(3.5절) |
| 정보 유출·도청 | 백업 파일·DB 파일 탈취, 평문 통신 도청(4장) |

> **집성(Aggregation)** 은 DB의 **보안 위협**이지 보안 통제(유형)가 아닙니다. "접근 제어, 추론 제어, 흐름 제어" 와 섞어 묻는 경우가 많습니다.
{: .prompt-warning }

---

## 3.2 집성(Aggregation)

### [이론]

**집성**은 낮은 보안 등급의 정보를 **여러 개 모아서** 높은 등급의 정보를 알아내는 것입니다. 각 정보 하나하나에는 접근 권한이 있지만, 그것을 **합친 정보**에는 접근 권한이 없는 경우에 생깁니다.

| 예 | 하나하나(공개) | 모은 것(기밀) |
|---|---|---|
| 매출 | 지점별 매출 | **회사 전체 매출** |
| 경쟁사 | 각 지사의 영업 실적 | 경쟁사의 **총 매출액** |
| 군 | 부대 하나의 위치 | 전체 **배치 현황** |

> 개별 데이터 항목보다 **종합한 정보의 보안 등급이 더 높은** 것이 핵심입니다. 반대로 개별 정보에서 숨긴 값을 **추측**하는 것은 3.3절의 **추론**입니다.
{: .prompt-tip }

### [실습] (Ubuntu) 지점별 매출을 더해 전체 매출 알아내기

관리자가 지점별 매출 테이블을 만들고, 지점 담당자들이 보도록 `appuser` 에 읽기 권한을 줍니다. 회사 전체 매출은 **경영진만 아는 기밀**이라고 가정합니다.

```bash
# === Ubuntu ===
sudo mariadb companydb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
CREATE TABLE branch_sales (
  branch VARCHAR(20) PRIMARY KEY,
  amount INT
);
INSERT INTO branch_sales VALUES ('Seoul',120), ('Busan',80), ('Daegu',50), ('Gwangju',40);
GRANT SELECT ON companydb.branch_sales TO 'appuser'@'localhost';
exit
```

```
MariaDB [companydb]> CREATE TABLE branch_sales (
    ->   branch VARCHAR(20) PRIMARY KEY,
    ->   amount INT
    -> );
Query OK, 0 rows affected (0.004 sec)

MariaDB [companydb]> INSERT INTO branch_sales VALUES ('Seoul',120), ('Busan',80), ('Daegu',50), ('Gwangju',40);
Query OK, 4 rows affected (0.001 sec)
Records: 4  Duplicates: 0  Warnings: 0

MariaDB [companydb]> GRANT SELECT ON companydb.branch_sales TO 'appuser'@'localhost';
Query OK, 0 rows affected (0.001 sec)

MariaDB [companydb]> exit
Bye
```

이제 `appuser` 로 접속해 지점별 매출을 보고, 그것을 더해 봅니다. 비밀번호는 `apppass123` 입니다.

```bash
# === Ubuntu ===
mariadb -u appuser -p companydb
```

```sql
-- === Ubuntu: MariaDB console (appuser) ===
SELECT * FROM branch_sales;
SELECT SUM(amount) AS company_total FROM branch_sales;
exit
```

```
Enter password: 
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 54
Server version: 10.11.14-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [companydb]> SELECT * FROM branch_sales;
+---------+--------+
| branch  | amount |
+---------+--------+
| Busan   |     80 |
| Daegu   |     50 |
| Gwangju |     40 |
| Seoul   |    120 |
+---------+--------+
4 rows in set (0.000 sec)

MariaDB [companydb]> SELECT SUM(amount) AS company_total FROM branch_sales;
+---------------+
| company_total |
+---------------+
|           290 |
+---------------+
1 row in set (0.000 sec)

MariaDB [companydb]> exit
Bye
```

| 명령 | 설명 |
|---|---|
| `SUM(amount)` | `amount` 열을 모두 더함(집계 함수) |
| `AS company_total` | 결과 열에 이름을 붙임 |

| 화면 | 의미 |
|---|---|
| 지점별 4개 행 | 각각은 **허가된(공개) 정보** |
| `company_total` 290 | 모아서 더한 결과는 **기밀(회사 전체 매출)**. 권한 검사는 모두 통과했지만 정보가 샘 → **집성** |

> 집성은 권한(`GRANT`)만으로는 막기 어렵습니다. 낮은 등급 사용자가 볼 수 있는 항목의 **범위를 처음부터 줄이고**(필요한 지점만), 대량 조회를 감사(4.3절)로 지켜보는 방법을 함께 씁니다.
{: .prompt-info }

---

## 3.3 추론(Inference)

### [이론]

**추론**은 보통의 공개된 데이터나 **통계 값**으로부터 **비밀 정보를 알아내는 것**입니다. 사용자가 통계(합계·평균) 질의는 할 수 있지만 개별 값은 볼 수 없게 해 두어도, 질의를 잘 고르면 개별 값이 드러납니다.

| 예 | 공개된 것 | 알아낸 것 |
|---|---|---|
| 급여 | 부서별 평균 급여, 부서 인원 | 인원이 **1명인 부서**의 평균 = 그 사람의 급여 |
| 성적 | 반 평균, 한 명 빼고 나머지 학생 점수 | 빠진 학생의 점수 |

추론을 막는 방법(**추론 통제**)은 다음과 같습니다.

| 대책 | 내용 |
|---|---|
| **질의 결과 크기 제한** | 결과에 포함되는 행이 **너무 적은**(예: 3명 미만) 통계 질의는 거부 |
| **다중 인스턴스화**(Polyinstantiation, 다중 사례화) | 같은 키의 데이터를 **보안 등급별로 여러 버전**으로 두어, 낮은 등급 사용자에게는 **가짜 정보(Cover Story)** 를 보여 줌 |
| **분할**(Partition) | DB를 보안 등급별로 나누어 따로 관리 |
| **셀 은폐**(Cell Suppression) | 민감한 값이 들어 있는 칸(셀)을 결과에서 숨김 |
| **잡음·섭동**(Noise, Perturbation) | 통계 결과에 작은 오차를 섞어 정확한 값을 거꾸로 계산하지 못하게 함 |

#### 다중 인스턴스화가 필요한 이유

낮은 등급 사용자가 `emp_id = 9` 인 행을 넣으려는데, 이미 **비밀 등급**으로 같은 번호의 행이 있다고 합시다. DB가 "이미 있는 번호입니다(키 중복)" 오류를 내면, 사용자는 **보이지 않는 비밀 행이 있다는 사실**을 추론합니다. 다중 인스턴스화는 낮은 등급용 행을 따로 하나 더 만들어 이런 **입력 오류를 통한 추론**을 막습니다.

> 추론을 막는 대표 기술로 **다중 인스턴스화(Polyinstantiation)** 를 기억해 둡니다. **DAC**(05-1의 GRANT)나 **암호화**는 허가된 사용자가 통계를 통해 알아내는 것을 막지 못합니다.
{: .prompt-warning }

### [실습 1] (Ubuntu) 부서별 평균 급여로 개인 급여 알아내기

인사팀이 "개인 급여는 숨기고 **부서별 통계만** 공개"하려고 뷰를 만들어 `appuser` 에게 줍니다.

```bash
# === Ubuntu ===
sudo mariadb companydb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
CREATE VIEW dept_stats AS
  SELECT d.dept_name, COUNT(*) AS headcount, AVG(e.salary) AS avg_salary
  FROM employees e JOIN departments d ON e.dept_id = d.dept_id
  GROUP BY d.dept_name;
GRANT SELECT ON companydb.dept_stats TO 'appuser'@'localhost';
exit
```

```
MariaDB [companydb]> CREATE VIEW dept_stats AS
    ->   SELECT d.dept_name, COUNT(*) AS headcount, AVG(e.salary) AS avg_salary
    ->   FROM employees e JOIN departments d ON e.dept_id = d.dept_id
    ->   GROUP BY d.dept_name;
Query OK, 0 rows affected (0.003 sec)

MariaDB [companydb]> GRANT SELECT ON companydb.dept_stats TO 'appuser'@'localhost';
Query OK, 0 rows affected (0.001 sec)

MariaDB [companydb]> exit
Bye
```

| 명령 | 설명 |
|---|---|
| `COUNT(*)`, `AVG(e.salary)` | 행의 수(인원), 급여 평균 |
| `GROUP BY d.dept_name` | 부서 이름별로 묶어서 계산 |

`appuser` 로 통계 뷰와 05-1의 직원 목록 뷰를 함께 봅니다.

```bash
# === Ubuntu ===
mariadb -u appuser -p companydb
```

```sql
-- === Ubuntu: MariaDB console (appuser) ===
SELECT * FROM dept_stats;
SELECT name, dept_name FROM emp_public WHERE dept_name = 'Security';
exit
```

```
Enter password: 
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 54
Server version: 10.11.14-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [companydb]> SELECT * FROM dept_stats;
+-----------+-----------+------------+
| dept_name | headcount | avg_salary |
+-----------+-----------+------------+
| Dev       |         3 |  4800.0000 |
| Sales     |         2 |  4050.0000 |
| Security  |         1 |  6300.0000 |
+-----------+-----------+------------+
3 rows in set (0.001 sec)

MariaDB [companydb]> SELECT name, dept_name FROM emp_public WHERE dept_name = 'Security';
+------+-----------+
| name | dept_name |
+------+-----------+
| Kang | Security  |
+------+-----------+
1 row in set (0.000 sec)

MariaDB [companydb]> exit
Bye
```

| 화면 | 의미 |
|---|---|
| `Security ... 1 ... 6300.0000` | Security 부서는 **1명**이고 평균 급여가 6300 |
| `Kang ... Security` | 그 한 명이 Kang → **Kang의 급여는 6300**. `employees` 를 볼 권한이 없는데도 알아냄 → **추론** |
| `Sales ... 2 ... 4050` | 2명뿐인 부서도 위험. 한 사람(예: 본인)의 급여를 알면 나머지 한 사람의 급여가 `4050 × 2 − 본인` 으로 계산됨 |

### [실습 2] (Ubuntu) 인원이 적은 부서의 통계 숨기기

뷰에 **질의 결과 크기 제한**을 넣습니다. 인원이 3명 이상인 부서만 통계를 보여 줍니다.

```bash
# === Ubuntu ===
sudo mariadb companydb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
CREATE OR REPLACE VIEW dept_stats AS
  SELECT d.dept_name, COUNT(*) AS headcount, AVG(e.salary) AS avg_salary
  FROM employees e JOIN departments d ON e.dept_id = d.dept_id
  GROUP BY d.dept_name
  HAVING COUNT(*) >= 3;
exit
```

```
MariaDB [companydb]> CREATE OR REPLACE VIEW dept_stats AS
    ->   SELECT d.dept_name, COUNT(*) AS headcount, AVG(e.salary) AS avg_salary
    ->   FROM employees e JOIN departments d ON e.dept_id = d.dept_id
    ->   GROUP BY d.dept_name
    ->   HAVING COUNT(*) >= 3;
Query OK, 0 rows affected (0.004 sec)

MariaDB [companydb]> exit
Bye
```

| 명령 | 설명 |
|---|---|
| `CREATE OR REPLACE VIEW` | 같은 이름의 뷰가 있으면 새 내용으로 바꿈. 이미 준 권한은 그대로 유지됨 |
| `HAVING COUNT(*) >= 3` | 묶은 결과(부서) 가운데 **인원이 3명 이상인 것만** 남김 |

```bash
# === Ubuntu ===
mariadb -u appuser -p companydb
```

```sql
-- === Ubuntu: MariaDB console (appuser) ===
SELECT * FROM dept_stats;
exit
```

```
Enter password: 
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 56
Server version: 10.11.14-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [companydb]> SELECT * FROM dept_stats;
+-----------+-----------+------------+
| dept_name | headcount | avg_salary |
+-----------+-----------+------------+
| Dev       |         3 |  4800.0000 |
+-----------+-----------+------------+
1 row in set (0.000 sec)

MariaDB [companydb]> exit
Bye
```

| 화면 | 의미 |
|---|---|
| `Dev` 만 남음 | 1명(Security)·2명(Sales) 부서의 통계가 **숨겨짐** → 개인 급여를 거꾸로 계산할 수 없음 |

> 이 방법도 완벽하지는 않습니다. 공격자는 "전체 평균"과 "특정 부서를 뺀 평균"처럼 **큰 집합 두 개의 차이**로 작은 집합을 알아내기도 합니다(추적자 공격). 그래서 실제로는 셀 은폐, 잡음 추가 같은 방법을 함께 씁니다.
{: .prompt-info }

---

## 3.4 흐름 통제

### [이론]

**흐름 통제**는 정보가 **명시적으로든 암묵적으로든** 높은 보호 수준의 객체에서 **낮은 보호 수준의 객체로 옮겨 가는 것**을 검사해 막는 것입니다.

| 예 | 흐름 |
|---|---|
| 비밀 등급 사용자가 비밀 테이블의 값을 **공개 테이블에 복사** | 명시적 흐름 |
| 프로그램이 비밀 값이 크면 A, 작으면 B를 공개 테이블에 기록 | 암묵적 흐름(값이 직접 옮겨지지 않아도 정보가 샘) |

| 통제 | 한 줄 정의 |
|---|---|
| 접근 통제 | **누가** 무엇에 접근할 수 있나 |
| 추론 통제 | 허용된 질의로 **간접적으로** 비밀을 알아내지 못하게 |
| **흐름 통제** | 정보가 **보호 수준이 낮은 곳으로 이동**하지 못하게 |

> "간접적으로 노출된 데이터로 다른 데이터를 추론", "대규모 통계 분석에서 많이 발생", "허용 가능한 질의를 제한" 은 모두 **추론 통제**에 대한 설명이고, "정보가 낮은 보호 수준의 객체로 이동하는 것을 검사" 는 **흐름 통제**입니다.
{: .prompt-warning }

---

## 3.5 SQL 인젝션

### [이론]

**SQL 인젝션**은 웹 페이지의 입력값에 SQL 조각을 넣어, 애플리케이션이 만든 SQL 문의 뜻을 바꾸는 공격입니다. 자세한 원리와 실습은 웹 보안 과정에서 다루고, 여기서는 DB 보안과 연결되는 점만 봅니다.

| 공격 유형 | 결과 |
|---|---|
| **인증 우회** | 비밀번호 없이 로그인(`' OR '1'='1`) |
| **데이터 노출** | 다른 테이블의 데이터까지 조회(UNION) |
| **원격 명령 실행** | DBMS 기능(예: MS-SQL `xp_cmdshell`)으로 서버 명령 실행 |

> SQL 인젝션의 대표 유형은 **인증 우회, 데이터 노출, 원격 명령 실행**입니다. 서비스 거부는 SQL 인젝션의 유형으로 분류하지 않습니다. DB 쪽 대책은 05-1의 **최소 권한**(웹 계정에 DROP·관리 권한 금지)과 4.3절의 **감사**이며, 근본 대책은 애플리케이션에서 **입력값 검증과 준비된 문장(Prepared Statement)** 을 쓰는 것입니다.
{: .prompt-tip }

---

# 4. DB 보안 대책

## 4.1 DB 보안 솔루션

### [이론]

| 솔루션 | 하는 일 |
|---|---|
| **DB 접근제어** | 허가되지 않은 사용자·PC·프로그램의 DB 접속과 SQL 실행을 **차단**하고, 모든 접근을 기록 |
| **DB 감사** | DB에 대한 모든 접근과 SQL 실행을 **기록**해 추적성을 확보(사후 분석) |
| **DB 암호화** | 주민등록번호·카드번호 같은 민감 데이터를 **암호화해 저장**. 조회할 때 복호화 |

DB 접근제어 솔루션은 설치 방식에 따라 장단점이 다릅니다.

| 방식 | 구성 | 장점 | 단점 |
|---|---|---|---|
| **게이트웨이**(Proxy) | 사용자와 DB 사이에 **독립된 서버**를 두고 모든 접속이 거쳐 가게 함 | 여러 DB(다중 인스턴스)를 한곳에서 통제, 접근통제와 로그를 통합 관리 | 장애 대비 **이중화** 필요, 게이트웨이를 **우회**하는 접속(예: DB 서버에 직접 Telnet·SSH로 들어가 SQL 실행)은 통제하기 어려움 |
| 스니핑 | 네트워크 트래픽을 복사해 분석 | 성능 영향 없음 | 실시간 차단이 어려움 |
| **에이전트** | **DB 서버에 접근제어·로깅 프로그램(에이전트)을 설치** | DB에 직접 접속하는 전용 클라이언트까지 **모든 접근 경로**를 통제하는 **가장 강력한** 방식 | DB 서버 부하, 서버마다 설치·관리 |

> "독립된 서버로 구축, 이중화 필요, 우회 접속에 취약, 다중 인스턴스(여러 DB) 통제 가능" 은 **게이트웨이 방식 DB 접근제어 솔루션**의 특징입니다. 반대로 "전용 클라이언트를 포함한 **모든 접근 경로**를 통제하는 가장 강력한 방법" 은 **에이전트 방식**입니다. 둘을 섞지 않도록 주의합니다.
{: .prompt-tip }

---

## 4.2 DB 암호화

### [이론]

| 방식 | 어디서 암·복호화 | 장점 | 단점 |
|---|---|---|---|
| **Plug-in** | **DB 서버 안**에 설치한 암호 모듈 | 애플리케이션 **수정이 적음** | 암·복호화에 **DB 서버 CPU**를 써서 부하. 처리량이 많지 않은 시스템에 적합 |
| **API** | **애플리케이션 서버**에서 암호 모듈(API)을 호출 | DB 서버 부하 없음 | **애플리케이션 수정** 필요. 새로 개발하는 시스템에 적합 |
| **TDE**(Transparent Data Encryption) | **DBMS 커널**이 파일에 쓸 때 자동 암호화 | 애플리케이션 수정 없음, 성능 좋음 | DBMS가 지원해야 함. DB에 접속한 사용자에게는 평문으로 보임 |
| **파일 암호화** | 운영체제에서 DB **파일 전체**를 암호화 | 적용이 쉬움 | 파일 단위라 세밀한 통제가 어려움 |
| **하이브리드** | Plug-in과 API를 섞어 씀 | 시스템 특성에 맞게 선택 | 구성이 복잡 |

> "애플리케이션의 수정이 필요하다" 는 **API 방식**, "DB 서버의 CPU를 사용해 부하가 생긴다" 는 **Plug-in 방식**의 특징입니다.
{: .prompt-warning }

#### 암호화와 해시

| 구분 | 암호화(AES 등) | 해시(SHA-2 등) |
|---|---|---|
| 되돌릴 수 있나 | **키가 있으면 복호화 가능** | **되돌릴 수 없음**(일방향) |
| 쓰는 곳 | 다시 원래 값을 봐야 하는 정보: 주민등록번호, **카드번호** | 원래 값을 볼 필요가 없는 정보: **비밀번호** |
| 확인 방법 | 복호화해서 봄 | 입력값을 같은 방식으로 해시해 **저장된 해시와 비교** |

> 비밀번호를 해시할 때는 사용자마다 다른 임의의 값(**솔트, salt**)을 붙입니다. 솔트가 없으면 같은 비밀번호는 같은 해시가 되어, 미리 계산해 둔 해시 표(레인보우 테이블)로 쉽게 찾아집니다.
{: .prompt-info }

### [실습 1] (Ubuntu) 카드번호를 AES로 암호화해 저장

MariaDB의 `AES_ENCRYPT`·`AES_DECRYPT` 함수로 열 하나를 암호화합니다. 실제 API 방식에서는 이 일을 애플리케이션이 하지만, 원리는 같습니다.

```bash
# === Ubuntu ===
sudo mariadb companydb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
CREATE TABLE customers (
  id       INT PRIMARY KEY,
  name     VARCHAR(30),
  card_enc VARBINARY(64)
);
SET @k = 'Lab#Key2026';
INSERT INTO customers VALUES
  (1, 'Kim', AES_ENCRYPT('4111-1111-1111-1111', @k)),
  (2, 'Lee', AES_ENCRYPT('5500-0000-0000-0004', @k));
SELECT id, name, HEX(card_enc) FROM customers;
SELECT id, name, CAST(AES_DECRYPT(card_enc, @k) AS CHAR) AS card FROM customers;
SELECT id, name, CAST(AES_DECRYPT(card_enc, 'WrongKey') AS CHAR) AS card FROM customers;
```

```
MariaDB [companydb]> CREATE TABLE customers (
    ->   id       INT PRIMARY KEY,
    ->   name     VARCHAR(30),
    ->   card_enc VARBINARY(64)
    -> );
Query OK, 0 rows affected (0.003 sec)

MariaDB [companydb]> SET @k = 'Lab#Key2026';
Query OK, 0 rows affected (0.000 sec)

MariaDB [companydb]> INSERT INTO customers VALUES
    ->   (1, 'Kim', AES_ENCRYPT('4111-1111-1111-1111', @k)),
    ->   (2, 'Lee', AES_ENCRYPT('5500-0000-0000-0004', @k));
Query OK, 2 rows affected (0.001 sec)
Records: 2  Duplicates: 0  Warnings: 0

MariaDB [companydb]> SELECT id, name, HEX(card_enc) FROM customers;
+----+------+------------------------------------------------------------------+
| id | name | HEX(card_enc)                                                    |
+----+------+------------------------------------------------------------------+
|  1 | Kim  | 8DEB0B43A947805F22D76AA67CCA6C85834E36AD53D0BD51D55059F5AB82C82E |
|  2 | Lee  | 96D17ABDEF2BAC968906E37617F25144B5EE79D16A2E2C0DFCFC60BBC99E8CC3 |
+----+------+------------------------------------------------------------------+
2 rows in set (0.000 sec)

MariaDB [companydb]> SELECT id, name, CAST(AES_DECRYPT(card_enc, @k) AS CHAR) AS card FROM customers;
+----+------+---------------------+
| id | name | card                |
+----+------+---------------------+
|  1 | Kim  | 4111-1111-1111-1111 |
|  2 | Lee  | 5500-0000-0000-0004 |
+----+------+---------------------+
2 rows in set (0.000 sec)

MariaDB [companydb]> SELECT id, name, CAST(AES_DECRYPT(card_enc, 'WrongKey') AS CHAR) AS card FROM customers;
+----+------+------+
| id | name | card |
+----+------+------+
|  1 | Kim  | NULL |
|  2 | Lee  | NULL |
+----+------+------+
2 rows in set (0.000 sec)
```

| 명령 | 설명 |
|---|---|
| `VARBINARY(64)` | 글자가 아닌 **바이트**를 저장하는 형식. 암호문은 바이트이므로 이 형식을 씀 |
| `SET @k = '...';` | 암호 키를 변수 `@k` 에 담음. 실제로는 키를 DB 밖(키 관리 서버, HSM)에 보관 |
| `AES_ENCRYPT(평문, 키)` | AES로 암호화 |
| `HEX(...)` | 바이트를 16진수 글자로 바꿔 보여 줌 |
| `AES_DECRYPT(암호문, 키)` | 복호화. `CAST(... AS CHAR)` 로 글자로 바꿔 표시 |

| 화면 | 의미 |
|---|---|
| `8DEB0B43...` | DB 파일이나 백업이 유출되어도 카드번호는 **암호문**으로만 보임 |
| 맞는 키 → 카드번호 | 키를 가진 쪽만 원래 값을 봄 |
| 틀린 키 → `NULL` | 키가 없으면 복호화 실패 |

> 이 실습의 카드번호는 결제 시스템 시험용으로 공개된 **가짜 번호**입니다. 키를 SQL 문에 그대로 적으면 4.3절의 감사 로그나 명령 기록에 키가 남을 수 있으므로, 실제 시스템에서는 키를 코드·로그에 남기지 않습니다.
{: .prompt-info }

### [실습 2] (Ubuntu) 비밀번호는 솔트를 붙여 해시로 저장

같은 콘솔에서 이어서 입력합니다.

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
CREATE TABLE web_users (
  login   VARCHAR(30) PRIMARY KEY,
  salt    CHAR(8),
  pw_hash CHAR(64)
);
INSERT INTO web_users VALUES
  ('kim', 'a1b2c3d4', SHA2(CONCAT('a1b2c3d4', 'Summer2026!'), 256)),
  ('lee', 'e5f6g7h8', SHA2(CONCAT('e5f6g7h8', 'Summer2026!'), 256));
SELECT * FROM web_users;
SELECT login FROM web_users WHERE login = 'kim' AND pw_hash = SHA2(CONCAT(salt, 'Summer2026!'), 256);
SELECT login FROM web_users WHERE login = 'kim' AND pw_hash = SHA2(CONCAT(salt, 'wrong'), 256);
exit
```

```
MariaDB [companydb]> CREATE TABLE web_users (
    ->   login   VARCHAR(30) PRIMARY KEY,
    ->   salt    CHAR(8),
    ->   pw_hash CHAR(64)
    -> );
Query OK, 0 rows affected (0.003 sec)

MariaDB [companydb]> INSERT INTO web_users VALUES
    ->   ('kim', 'a1b2c3d4', SHA2(CONCAT('a1b2c3d4', 'Summer2026!'), 256)),
    ->   ('lee', 'e5f6g7h8', SHA2(CONCAT('e5f6g7h8', 'Summer2026!'), 256));
Query OK, 2 rows affected (0.001 sec)
Records: 2  Duplicates: 0  Warnings: 0

MariaDB [companydb]> SELECT * FROM web_users;
+-------+----------+------------------------------------------------------------------+
| login | salt     | pw_hash                                                          |
+-------+----------+------------------------------------------------------------------+
| kim   | a1b2c3d4 | a01855a2f2dc393119d1ef5a75ee137947859c96edbcd9957f290874676e4e36 |
| lee   | e5f6g7h8 | 4c27c0091921f08a4f00e90847c1cb607d2750bebdf8f87165962efe298b8bc1 |
+-------+----------+------------------------------------------------------------------+
2 rows in set (0.000 sec)

MariaDB [companydb]> SELECT login FROM web_users WHERE login = 'kim' AND pw_hash = SHA2(CONCAT(salt, 'Summer2026!'), 256);
+-------+
| login |
+-------+
| kim   |
+-------+
1 row in set (0.000 sec)

MariaDB [companydb]> SELECT login FROM web_users WHERE login = 'kim' AND pw_hash = SHA2(CONCAT(salt, 'wrong'), 256);
Empty set (0.000 sec)

MariaDB [companydb]> exit
Bye
```

| 명령 | 설명 |
|---|---|
| `CONCAT(솔트, 비밀번호)` | 솔트와 비밀번호를 이어 붙임 |
| `SHA2(..., 256)` | SHA-256 해시(16진수 64글자) |
| 두 번째·세 번째 `SELECT` | 로그인 확인 흉내. 입력한 비밀번호를 같은 방식으로 해시해 저장값과 비교 |

| 화면 | 의미 |
|---|---|
| `kim` 과 `lee` 의 해시가 다름 | 두 사람의 비밀번호는 **같지만**(`Summer2026!`), 솔트가 달라 해시가 다름 → 한 사람의 비밀번호가 풀려도 다른 사람은 안전 |
| 맞는 비밀번호 → `kim` | 로그인 성공 |
| 틀린 비밀번호 → `Empty set` | 로그인 실패 |

> 실제 서비스에서는 SHA-256을 한 번 돌리는 대신, 일부러 느리게 만든 비밀번호 전용 함수(**bcrypt**, **Argon2**, PBKDF2)를 애플리케이션에서 씁니다.
{: .prompt-info }

---

## 4.3 DB 감사(Audit)

### [이론]

**감사**는 누가, 언제, 어디서, 어떤 SQL을 실행했고 성공했는지를 **기록**하는 것입니다. 기록이 있어야 사고가 났을 때 원인을 찾고(**책임 추적성**), 이상한 행동(대량 조회, 반복된 로그인 실패, 권한 밖 시도)을 미리 알아챌 수 있습니다.

| 기록할 것 | 이유 |
|---|---|
| 로그인 성공·**실패** | 비밀번호 대입 공격 탐지 |
| 권한·계정 변경(GRANT, CREATE USER) | 권한 남용 탐지 |
| 민감 테이블 조회 | 개인정보 유출 추적 |
| **거부된 SQL** | 권한 밖 시도, SQL 인젝션 탐지 |

> 접속 일시, 접속 성공·실패, 접근 IP·MAC, 계정 유형, 열람·변경·삭제 같은 **행위 기록**은 접근을 막는 **접근통제** 기능이 아니라 **감사증적**(기록)을 만드는 기능입니다. 접근통제는 "막는 것", 감사는 "남기는 것"입니다.
{: .prompt-warning }

### [실습] (Ubuntu) 감사 플러그인으로 로그인 실패와 거부된 조회 기록

MariaDB에 들어 있는 감사 플러그인(`server_audit`)을 켭니다.

```bash
# === Ubuntu ===
sudo mariadb
```

```sql
-- === Ubuntu: MariaDB console (sudo mariadb) ===
INSTALL SONAME 'server_audit';
SET GLOBAL server_audit_events = 'CONNECT,QUERY';
SET GLOBAL server_audit_logging = ON;
exit
```

```
MariaDB [(none)]> INSTALL SONAME 'server_audit';
Query OK, 0 rows affected (0.002 sec)

MariaDB [(none)]> SET GLOBAL server_audit_events = 'CONNECT,QUERY';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> SET GLOBAL server_audit_logging = ON;
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> exit
Bye
```

| 명령 | 설명 |
|---|---|
| `INSTALL SONAME 'server_audit';` | 감사 플러그인을 불러옴. 한 번 설치하면 다시 시작해도 남아 있음 |
| `server_audit_events = 'CONNECT,QUERY'` | **접속**(성공·실패)과 **모든 SQL**을 기록 |
| `server_audit_logging = ON` | 기록 시작. 기록 파일은 `/var/lib/mysql/server_audit.log` |

이제 `appuser` 로 비밀번호를 한 번 틀리고, 다시 바르게 접속해 허용된 조회 하나와 권한 밖 조회 하나를 합니다.

```bash
# === Ubuntu ===
mariadb -u appuser -pwrongpass companydb
```

```
ERROR 1045 (28000): Access denied for user 'appuser'@'localhost' (using password: YES)
```

```bash
# === Ubuntu ===
mariadb -u appuser -p companydb
```

```sql
-- === Ubuntu: MariaDB console (appuser) ===
SELECT name FROM emp_public WHERE emp_id = 1;
SELECT salary FROM employees;
exit
```

```
Enter password: 
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 34
Server version: 10.11.14-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [companydb]> SELECT name FROM emp_public WHERE emp_id = 1;
+------+
| name |
+------+
| Kim  |
+------+
1 row in set (0.000 sec)

MariaDB [companydb]> SELECT salary FROM employees;
ERROR 1142 (42000): SELECT command denied to user 'appuser'@'localhost' for table `companydb`.`employees`
MariaDB [companydb]> exit
Bye
```

감사 기록을 봅니다.

```bash
# === Ubuntu ===
sudo cat /var/lib/mysql/server_audit.log
```

```
20261121 12:15:49,ubuntu-lab,root,localhost,32,62,QUERY,,'SET GLOBAL server_audit_logging = ON',0
20261121 12:15:50,ubuntu-lab,root,localhost,32,0,DISCONNECT,,,0
20261121 12:15:51,ubuntu-lab,appuser,localhost,33,0,FAILED_CONNECT,,,1045
20261121 12:15:51,ubuntu-lab,appuser,localhost,33,0,DISCONNECT,,,0
20261121 12:15:52,ubuntu-lab,appuser,localhost,34,0,CONNECT,companydb,,0
20261121 12:15:52,ubuntu-lab,appuser,localhost,34,64,QUERY,companydb,'show databases',0
20261121 12:15:52,ubuntu-lab,appuser,localhost,34,65,QUERY,companydb,'show tables',0
20261121 12:15:52,ubuntu-lab,appuser,localhost,34,69,QUERY,companydb,'select @@version_comment limit 1',0
20261121 12:15:52,ubuntu-lab,appuser,localhost,34,70,QUERY,companydb,'SELECT name FROM emp_public WHERE emp_id = 1',0
20261121 12:15:53,ubuntu-lab,appuser,localhost,34,71,QUERY,companydb,'SELECT salary FROM employees',1142
20261121 12:15:53,ubuntu-lab,appuser,localhost,34,0,DISCONNECT,companydb,,0
```

| 칸 | 예 | 뜻 |
|---|---|---|
| 1 | `20261121 12:15:51` | 날짜와 시각 |
| 2 | `ubuntu-lab` | DB 서버의 호스트 이름(VM마다 다름) |
| 3, 4 | `appuser`, `localhost` | **누가, 어디서** |
| 5 | `33` | 접속 번호(같은 접속의 기록끼리 같음) |
| 7 | `FAILED_CONNECT`, `CONNECT`, `QUERY`, `DISCONNECT` | 사건 종류 |
| 8 | `companydb` | 사용한 데이터베이스 |
| 9 | `'SELECT salary FROM employees'` | 실행한 SQL |
| 마지막 | `0`, `1045`, `1142` | 결과. `0` 은 성공, 그 밖은 **오류 번호** |

| 찾아낼 것 | 기록 |
|---|---|
| 비밀번호 실패 | `FAILED_CONNECT,,,1045` |
| 권한 밖 조회 시도 | `'SELECT salary FROM employees',1142` |
| `show databases` 등 | 콘솔 프로그램이 접속하면서 자동으로 보낸 SQL |

> 📷 **캡처**: 감사 로그에서 `FAILED_CONNECT` 줄과 `1142` 로 끝나는 줄이 보이는 화면.

> 1~2번째 줄은 감사를 켜는 명령 자체의 기록입니다. 이 실습 전에 다른 접속이 있었다면 그 기록도 함께 보입니다. `SET GLOBAL` 로 바꾼 설정은 서버를 다시 시작하면 꺼지므로, 실제 운영에서는 설정 파일(`/etc/mysql/mariadb.conf.d/`)에 적고, 로그 파일은 **DB 관리자도 고칠 수 없는 별도 서버**로 보냅니다.
{: .prompt-info }

---

## 4.4 네트워크 접근 제한

### [이론]

DB 서버는 웹 서버(애플리케이션 서버)만 접속하면 됩니다. 사용자 PC나 인터넷에서 DB 포트(MySQL·MariaDB **3306**, MS-SQL **1433**, Oracle **1521**)에 직접 접속할 수 있으면, 비밀번호 대입이나 취약점 공격을 바로 받습니다.

| 대책 | 방법 |
|---|---|
| DB 프로그램이 **대기하는 주소** 제한 | MariaDB 설정 `bind-address = 127.0.0.1`(Ubuntu 기본값). 웹 서버가 따로 있으면 내부망 주소만 |
| 계정의 **접속 가능 위치** 제한 | `'appuser'@'localhost'`, `'appuser'@'192.168.0.20'` 처럼 계정마다 호스트 지정. `'%'`(어디서나)는 피함 |
| **방화벽** | DB 포트는 웹 서버 IP에서 오는 것만 허용 |
| DB 접근제어 솔루션 | 4.1절 |

### [실습] (Ubuntu·Kali) DB 포트가 밖으로 열려 있는지 확인

```bash
# === Ubuntu ===
sudo ss -tlnp | grep 3306
```

```
LISTEN 0      80                           127.0.0.1:3306      0.0.0.0:*    users:(("mariadbd",pid=394,fd=24))
```

| 화면 | 의미 |
|---|---|
| `127.0.0.1:3306` | DB 서버는 **자기 자신에게서 오는 접속만** 받음 |

Kali에서 DB 포트와, 비교를 위해 늘 열려 있는 SSH 포트(22)에 접속해 봅니다.

```bash
# === Kali ===
nc -zvn 192.168.0.30 3306
nc -zvn 192.168.0.30 22
```

```
(UNKNOWN) [192.168.0.30] 3306 (mysql) : Connection refused
(UNKNOWN) [192.168.0.30] 22 (ssh) open
```

| 명령·옵션 | 설명 |
|---|---|
| `nc -z` | 데이터를 보내지 않고 **포트가 열려 있는지만** 확인 |
| `-v` | 결과를 자세히 출력 |
| `-n` | IP를 이름으로 바꾸는 조회(DNS)를 하지 않음 |

| 화면 | 의미 |
|---|---|
| `3306 (mysql) : Connection refused` | DB 포트는 밖에서 **접속 불가**. 01강에서 방화벽이 실습망을 허용해 두었어도, DB가 `127.0.0.1` 에서만 기다리므로 막힘 |
| `22 (ssh) open` | 같은 서버라도 밖에서 쓰라고 연 서비스(SSH)는 열려 있음. 네트워크 문제가 아니라 **DB만 막혀 있다**는 비교 |

> 실제 서비스에서는 사용자가 **웹 서버를 통해서만** DB 데이터에 닿게 합니다. DB 포트를 열어야 한다면 `bind-address` 를 바꾸기 전에 **방화벽 규칙과 계정 호스트 제한**을 먼저 준비합니다.
{: .prompt-info }

---

## 4.5 DB 보안 대책 종합

| 번호 | 위협 | 대책 | 이 과정에서 확인한 곳 |
|---|---|---|---|
| 1 | 기본 계정·약한 비밀번호 | 기본 비밀번호 변경, 불필요한 계정 잠금·삭제, unix_socket 인증 | 05-1 2.1 |
| 2 | 비밀번호 대입 | 로그인 실패 잠금 | 05-1 2.4 |
| 3 | 과도한 권한 | 최소 권한(`GRANT SELECT` 만), 뷰로 열 제한 | 05-1 2.2·2.3 |
| 4 | 집성 | 조회 가능한 범위 축소, 대량 조회 감사 | 3.2 |
| 5 | 추론 | 질의 결과 크기 제한, 다중 인스턴스화, 셀 은폐, 잡음 | 3.3 |
| 6 | 정보의 하향 이동 | 흐름 통제 | 3.4 |
| 7 | SQL 인젝션 | 입력값 검증·준비된 문장, 최소 권한, 감사 | 3.5 |
| 8 | DB·백업 파일 유출 | 민감 정보 암호화(Plug-in·API·TDE), 비밀번호는 솔트+해시 | 4.2 |
| 9 | 사후 추적 불가 | 감사 로그(접속 실패, 거부된 SQL), 별도 서버 보관 | 4.3 |
| 10 | DB 포트 노출 | `bind-address`, 계정 호스트 제한, 방화벽, 접근제어 솔루션 | 4.4 |

---

# 확인 문제

> 1. 다음 중 데이터베이스 보안 유형(통제)이 **아닌** 것은?
>    ① 접근 제어
>    ② 추론 제어
>    ③ 흐름 제어
>    ④ 집성(Aggregation)
>
> 2. 다음 설명에 해당하는 데이터베이스 보안 위협 요소는?
>    - 낮은 보안 등급의 정보들을 이용하여 높은 등급의 정보를 알아낸다
>    - 개별 정보는 의미가 부족하나 합치면 중요한 정보를 알 수 있다
>    - 지점별 영업 실적을 조회하여 회사의 전체 영업 실적을 알아낸다
>
>    ① 사용자 인증
>    ② 다중 인스턴스화
>    ③ 집합성(Aggregation)
>    ④ 추론(Inference)
>
> 3. 사용자가 특정 정보에 접근할 권한은 없지만, 그 정보를 이루는 **구성 요소**에는 접근할 권한이 있을 때 생기는 보안 문제는?
>    ① 추론
>    ② 다중 실증
>    ③ 뷰 추출
>    ④ 집합(Aggregation)
>
> 4. 보통의 일반적인 데이터로부터 비밀 정보를 얻어 낼 수 있는 가능성으로, 사용자가 통계 값으로부터 개별 데이터 항목을 추적하지 못하도록 막아야 하는 것은?
>    ① 집합
>    ② 추론
>    ③ 분할
>    ④ 셀 은폐
>
> 5. DB 보안 위협 가운데 추론으로부터 정보 유출을 막기 위한 기술은?
>    ① 집합성
>    ② DAC
>    ③ 암호화
>    ④ 다중 인스턴스화
>
> 6. DB 보안 통제 기법에 대한 설명 가운데 성격이 **다른** 것은?
>    ① 간접적으로 노출된 데이터를 통해 다른 데이터가 공개되는 것을 막음
>    ② 정보가 명시적·암묵적으로 낮은 보호 수준의 객체로 이동하는 것을 검사
>    ③ 빅데이터 같은 대규모 데이터의 통계적 분석에서 많이 발생
>    ④ 허용 가능한 질의를 제한하고 질의의 응답으로 제공되는 데이터를 한정
>
> 7. 다음 특징을 가진 데이터베이스 보안 솔루션은?
>    - 조직의 보안 정책에 따라 사용자 허가와 로깅 여부를 결정
>    - 독립된 서버로 구축되므로 이중화 구성이 필요
>    - 네트워크를 우회하여 접근하는 경우 보안 취약점 발생
>    - 독립된 서버로 다중 인스턴스에 대한 통제가 가능
>
>    ① DB 백업 솔루션
>    ② DB 감사 솔루션
>    ③ DB 암호화 솔루션
>    ④ DB 접근제어 솔루션
>
> 8. DB 암호화 방식에 대한 설명 가운데 **다른 방식**을 설명한 것은?
>    ① 암·복호화 모듈을 DB 서버 안에 설치하고 그곳에서 암·복호화한다
>    ② 데이터 암·복호화 시 DBMS의 부하가 발생할 수 있다
>    ③ 애플리케이션의 수정이 필요하다
>    ④ 성능에 대한 민감성이 낮은 시스템에 저렴하게 구축할 수 있다
>
> 9. 다음 중 SQL 인젝션의 공격 유형이 **아닌** 것은?
>    ① 인증 우회
>    ② 데이터 노출
>    ③ 원격 명령 실행
>    ④ 서비스 거부
>
> 10. 비밀번호를 DB에 저장하는 방법으로 가장 알맞은 것은?
>    ① AES로 암호화하여 키와 함께 저장한다
>    ② 사용자별 솔트를 붙여 일방향 해시값으로 저장한다
>    ③ Base64로 인코딩하여 저장한다
>    ④ 평문으로 저장하고 뷰로 숨긴다
>
> 11. DB에 직접 접근하는 전용 클라이언트를 포함해 **모든 접근 경로**를 제어할 수 있는 가장 강력한 DB 접근제어 구성 방법은?
>     ① 게이트웨이 방법
>     ② 프락시 방법
>     ③ 에이전트 방법
>     ④ 인라인 방법
>
> 12. 데이터 접근통제에 대한 설명으로 **틀린** 것은?
>     ① 시스템 운영에 필요한 데이터나 패턴을 생성·변경·삭제하는 관리 기능을 통제한다
>     ② 서로 다른 역할의 관리자가 데이터를 무단으로 생성·변경·삭제하는 것을 통제한다
>     ③ 비인가 행위를 유발하는 송·수신 데이터를 통제한다
>     ④ 접속 일시, 접속 성공·실패, 접근 IP·MAC, 계정 유형, 열람·변경·삭제 행위를 기록한다
>
> 13. DB 암호화 방식 가운데 **애플리케이션을 수정하지 않고** 쓸 수 있는 것끼리 올바르게 짝지은 것은?
>     ① API 방식 - TDE 방식
>     ② TDE 방식 - 파일 암호화
>     ③ Plug-in 방식 - API 방식
>     ④ Plug-in 방식 - 파일 암호화
{: .prompt-tip }

<details>
<summary>정답 보기</summary>

1. <strong>④</strong> 집성은 보안 위협이다(3.1절).<br>
2. <strong>③</strong> (3.2절)<br>
3. <strong>④</strong> 구성 요소를 모아 권한 없는 정보를 얻는 것은 집합(집성)이다(3.2절).<br>
4. <strong>②</strong> (3.3절)<br>
5. <strong>④</strong> DAC와 암호화는 허가된 사용자의 통계 질의를 통한 추론을 막지 못한다(3.3절).<br>
6. <strong>②</strong> ②는 흐름 통제, 나머지는 추론 통제에 대한 설명이다(3.4절).<br>
7. <strong>④</strong> 게이트웨이 방식 DB 접근제어 솔루션의 특징이다(4.1절).<br>
8. <strong>③</strong> ③은 API 방식, 나머지는 Plug-in 방식의 설명이다(4.2절).<br>
9. <strong>④</strong> (3.5절)<br>
10. <strong>②</strong> 비밀번호는 되돌릴 필요가 없으므로 솔트+해시로 저장한다(4.2절 실습 2).<br>
11. <strong>③</strong> 에이전트 방식은 DB 서버에 접근제어·로깅 기능을 설치해 전용 클라이언트를 포함한 모든 접근 경로를 통제한다. 게이트웨이 방식은 우회 접속에 취약하다(4.1절).<br>
12. <strong>④</strong> ④는 접근통제가 아니라 감사증적을 남기는 기능이다(4.3절).<br>
13. <strong>②</strong> TDE는 DBMS 커널이, 파일 암호화는 운영체제가 처리하므로 애플리케이션 수정이 없다. Plug-in은 경우에 따라 일부 수정이, API는 수정이 필요하다(4.2절).

</details>

---

# 체크리스트

- [ ] 접근·추론·흐름 통제와 집성·추론 위협을 구분해 설명 가능
- [ ] (Ubuntu) 지점별 매출(공개)의 합으로 전체 매출(기밀)을 얻는 집성을 확인
- [ ] (Ubuntu) 부서별 평균 급여 뷰로 1명뿐인 부서원의 급여를 알아내고, `HAVING COUNT(*) >= 3` 으로 막는 것을 확인
- [ ] 다중 인스턴스화가 입력 오류를 통한 추론을 막는 원리를 설명 가능
- [ ] DB 접근제어(게이트웨이 방식)·감사·암호화 솔루션의 특징을 구분 가능
- [ ] Plug-in·API·TDE 방식의 차이(DB 서버 부하, 애플리케이션 수정)를 설명 가능
- [ ] (Ubuntu) AES로 카드번호를 암호화하고 틀린 키는 `NULL` 이 되는 것, 솔트+SHA-256으로 같은 비밀번호도 해시가 다른 것을 확인
- [ ] (Ubuntu) 감사 로그에서 `FAILED_CONNECT ... 1045` 와 거부된 SQL(`1142`)을 찾음, 접속·행위 기록이 접근통제가 아니라 감사증적임을 설명 가능
- [ ] 게이트웨이(우회 접속에 취약)와 에이전트(모든 접근 경로 통제, 가장 강력) 방식의 차이를 설명 가능
- [ ] (Kali) DB 포트 3306은 `Connection refused`, SSH 포트 22는 `open` 인 것을 확인

> 이로써 **05. 데이터베이스 보안**을 마칩니다. 다음은 **06. SSL/TLS** 입니다. 웹 서버와 사용자 사이의 통신을 암호화해, 이 글에서 다룬 DB 데이터가 **오가는 길**까지 보호합니다.
{: .prompt-info }
