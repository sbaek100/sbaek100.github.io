---
title: "[애플리케이션 보안] 05-2. 실습 — MariaDB 계정·권한과 최소 권한"
date: 2026-11-17 10:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - DB보안
tags:
  - MariaDB
  - 접근통제
  - GRANT
  - 최소권한
  - 실습
math: false
mermaid: true
---

> 이 글은 **실습** 입니다. 개념은 앞 글(**05-1. DB 보안 이론**)의 3장(접근통제)을 먼저 읽으세요.  
> 이 실습은 **Ubuntu(192.168.56.30)** 한 대에서 진행합니다. (DB는 기본적으로 외부에 열지 않으므로 로컬에서 실습)
{: .prompt-info }

## 0. 목표

읽기 전용 계정 `appuser` 를 만들어, **권한 안의 작업(SELECT)은 되고, 권한 밖의 작업(INSERT/DROP)은 거부되는 것**을 직접 확인합니다.

---

## 1. (Ubuntu) MariaDB 설치·접속

```bash
# === Ubuntu에서 실행 ===
sudo apt update
sudo apt install -y mariadb-server

sudo systemctl status mariadb     # active (running) 확인 (q로 종료)

# 관리자(root)로 DB 콘솔 접속 (소켓 인증이라 비번 없이 들어감)
sudo mariadb
```

> 프롬프트가 `MariaDB [(none)]>` 로 바뀌면 DB 콘솔 안입니다. 이제부터는 **SQL 명령** 을 입력합니다. (끝에 세미콜론 `;`)
{: .prompt-tip }

---

## 2. (DB 콘솔) 데이터베이스·테이블·데이터 만들기

```sql
CREATE DATABASE companydb;
USE companydb;

CREATE TABLE members (
  id     INT PRIMARY KEY AUTO_INCREMENT,
  name   VARCHAR(50),
  dept   VARCHAR(50),
  salary INT
);

INSERT INTO members (name, dept, salary) VALUES
  ('홍길동', '영업', 5000),
  ('김철수', '개발', 6000),
  ('이영희', '인사', 5500);

SELECT * FROM members;
```

`SELECT` 결과로 3명이 보이면 데이터 준비 완료입니다.

---

## 3. (DB 콘솔) 최소 권한 계정 만들기

웹앱이 쓸 계정 `appuser` 를 만들되, **`members` 테이블 읽기(SELECT)만** 허용합니다.

```sql
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'apppass123';

-- 최소 권한: companydb.members 에 대해 SELECT 만
GRANT SELECT ON companydb.members TO 'appuser'@'localhost';

FLUSH PRIVILEGES;

-- 부여된 권한 확인
SHOW GRANTS FOR 'appuser'@'localhost';
```

권한을 다 줬으면 콘솔에서 나갑니다.

```sql
EXIT;
```

---

## 4. (Ubuntu) 제한 계정으로 동작 확인

이제 `appuser` 로 접속해, **무엇이 되고 무엇이 막히는지** 봅니다.

```bash
# === Ubuntu에서 실행 ===
mariadb -u appuser -p companydb
# 비밀번호: apppass123
```

DB 콘솔에서 차례로 시도합니다.

```sql
-- ✅ 허용된 작업 (SELECT)
SELECT * FROM members;

-- ❌ 권한 밖 작업 (INSERT) → 거부됨
INSERT INTO members (name, dept, salary) VALUES ('해커', '외부', 9999);

-- ❌ 테이블 삭제(DROP) → 거부됨
DROP TABLE members;
```

INSERT·DROP 시 다음과 비슷한 오류가 납니다.

```
ERROR 1142 (42000): INSERT command denied to user 'appuser'@'localhost' for table `members`
```

> 🟢 **이것이 최소 권한의 힘입니다.** 설령 이 계정이 SQL Injection으로 탈취되어도, 공격자는 **읽기만** 할 수 있을 뿐 데이터를 **지우거나 변조할 수 없습니다.**
{: .prompt-tip }

확인 후 나갑니다: `EXIT;`

---

## 5. (DB 콘솔) 권한 회수(REVOKE)와 정리

권한이 과하게 부여됐을 때 되돌리는 방법입니다.

```bash
# === Ubuntu에서 실행 ===
sudo mariadb
```

```sql
-- 만약 실수로 모든 권한을 줬다면 회수
-- (예시) REVOKE ALL PRIVILEGES ON companydb.* FROM 'appuser'@'localhost';

REVOKE SELECT ON companydb.members FROM 'appuser'@'localhost';
SHOW GRANTS FOR 'appuser'@'localhost';   -- USAGE(접속만 가능)만 남음
FLUSH PRIVILEGES;
EXIT;
```

> 권한이 회수되면 `appuser` 는 접속은 되지만 `SELECT` 조차 거부됩니다. **권한은 "필요한 만큼만, 필요할 때만"** 이 원칙입니다.
{: .prompt-warning }

---

## 6. 체크리스트

- [ ] MariaDB 설치·running, `sudo mariadb` 접속
- [ ] `companydb.members` 테이블·데이터 생성
- [ ] `appuser` 에 **SELECT 권한만** 부여(`SHOW GRANTS` 확인)
- [ ] `appuser` 로 SELECT는 성공, **INSERT·DROP은 거부(ERROR 1142)**
- [ ] `REVOKE` 로 권한 회수 동작 확인

> ⭐ `appuser` / `companydb` 는 다음 실습(05-3)에서도 그대로 사용합니다. 지우지 마세요.
{: .prompt-tip }

---

## 7. 다음 글

접근통제(누가·어디까지)를 익혔습니다. 다음 글 **05-3. 컬럼 암호화와 비밀번호 해시** 에서는 같은 `companydb` 에 **AES로 민감정보를 암호화** 하고, **비밀번호를 해시로 저장** 하며, **쿼리 감사 로그** 를 남겨 봅니다.
