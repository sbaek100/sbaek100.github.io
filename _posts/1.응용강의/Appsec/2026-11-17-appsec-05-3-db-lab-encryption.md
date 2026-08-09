---
title: "[애플리케이션 보안] 05-3. 실습 — 컬럼 암호화·비밀번호 해시·감사 로그"
date: 2026-11-17 11:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - DB보안
tags:
  - MariaDB
  - AES암호화
  - 해시
  - 감사로그
  - 실습
math: false
mermaid: true
---

> 이 글은 **실습** 입니다. 개념은 **05-1 이론**의 4장(암호화)·5장(해시·감사)을, 환경은 **05-2**(같은 `companydb`)를 이어서 사용합니다. 전부 **Ubuntu** 에서 진행합니다.
{: .prompt-info }

## A. AES로 민감정보 암호화하기 (API 방식의 핵심)

이론에서 본 **API(응용) 방식** 의 핵심은 "민감한 값을 **암호화해서** DB에 넣는 것"입니다. MariaDB의 `AES_ENCRYPT` / `AES_DECRYPT` 로 직접 해 봅니다.

### A.1 (Ubuntu) 콘솔 접속

```bash
# === Ubuntu에서 실행 ===
sudo mariadb
```

```sql
USE companydb;
```

### A.2 암호화 저장용 테이블

암호문은 바이너리이므로 **`VARBINARY`** 컬럼에 저장합니다.

```sql
CREATE TABLE secret_data (
  id      INT PRIMARY KEY AUTO_INCREMENT,
  owner   VARCHAR(50),
  ssn_enc VARBINARY(255)        -- 암호화된 주민번호(바이너리)
);
```

### A.3 암호화해서 INSERT

키 `MyKey#2025` 로 주민번호를 암호화해 저장합니다.

```sql
INSERT INTO secret_data (owner, ssn_enc) VALUES
  ('홍길동', AES_ENCRYPT('900101-1234567', 'MyKey#2025')),
  ('김철수', AES_ENCRYPT('880202-2345678', 'MyKey#2025'));
```

### A.4 그냥 보면 암호문, 키가 있어야 복호화

```sql
-- 그대로 보면 알아볼 수 없는 바이너리
SELECT owner, ssn_enc FROM secret_data;

-- 16진수로 보면 '암호문'임이 분명히 보임
SELECT owner, HEX(ssn_enc) FROM secret_data;

-- 올바른 키로만 복호화됨
SELECT owner, CAST(AES_DECRYPT(ssn_enc, 'MyKey#2025') AS CHAR) AS ssn
FROM secret_data;

-- 틀린 키 → NULL(못 읽음)
SELECT owner, CAST(AES_DECRYPT(ssn_enc, 'WrongKey') AS CHAR) AS ssn
FROM secret_data;
```

> 🟢 **핵심**: DB 파일이나 백업이 통째로 유출돼도, **키가 없으면 `ssn_enc` 는 의미 없는 바이너리** 입니다. 이것이 컬럼 암호화의 효과입니다.  
> 단, 키를 앱 코드에 그대로 박아 두면 의미가 없으므로 **키 관리(별도 보관)** 가 중요합니다.
{: .prompt-warning }

---

## B. 비밀번호는 암호화가 아니라 "해시"

비밀번호는 **되돌릴 필요가 없으므로** 암호화(AES)가 아니라 **해시(SHA-2)** 로 저장합니다. (이론 5장)

### B.1 해시로 저장

```sql
CREATE TABLE app_users (
  username VARCHAR(50) PRIMARY KEY,
  pw_hash  CHAR(64)          -- SHA-256 결과는 64자리 16진수
);

-- 평문이 아니라 해시를 저장
INSERT INTO app_users VALUES ('admin', SHA2('admin1234', 256));

SELECT * FROM app_users;     -- 비밀번호 원문은 어디에도 없음(해시만)
```

### B.2 로그인 검증 흉내

로그인 시에는 **입력값을 같은 방식으로 해시해서 비교** 합니다. (원문을 저장·비교하지 않음)

```sql
-- 맞는 비밀번호 → admin 행이 반환됨
SELECT username FROM app_users
WHERE username = 'admin' AND pw_hash = SHA2('admin1234', 256);

-- 틀린 비밀번호 → 아무 행도 없음
SELECT username FROM app_users
WHERE username = 'admin' AND pw_hash = SHA2('wrongpw', 256);
```

> 🟢 실제 서비스에서는 무차별 대입을 늦추기 위해 **솔트(salt)+반복 해시**(bcrypt 등)를 씁니다. 원리는 같습니다: **원문을 저장하지 않는다.**
{: .prompt-tip }

---

## C. 감사 로그 — 누가 어떤 쿼리를 실행했나

사고가 났을 때 추적하려면 **쿼리 기록(감사 로그)** 이 필요합니다. 파일 권한 문제를 피하려고 **로그를 테이블에 남기는** 방식을 씁니다.

```sql
-- 로그를 테이블에 기록하도록 설정
SET GLOBAL log_output = 'TABLE';
SET GLOBAL general_log = 'ON';

-- 추적 대상이 될 쿼리 몇 개 실행
SELECT * FROM members;
SELECT owner FROM secret_data;

-- 기록 확인 (최근 5건)
SELECT event_time, user_host, argument
FROM mysql.general_log
ORDER BY event_time DESC
LIMIT 5;

-- 실습 끝나면 로그 끄기(부하 방지)
SET GLOBAL general_log = 'OFF';
```

> `mysql.general_log` 에 **언제·누가·어떤 쿼리** 를 실행했는지 남습니다. 이상 징후(대량 SELECT, 권한 밖 시도)를 사후에 분석할 수 있습니다.
{: .prompt-tip }

콘솔을 나갑니다: `EXIT;`

---

## D. 정리 — DB 보안 한눈에

| 목표 | 기술 | 이번 실습 |
|---|---|---|
| 권한 통제 | 최소 권한 `GRANT`/`REVOKE` | 05-2 |
| 민감정보 보호 | **AES 암호화**(API 방식) | A |
| 비밀번호 보호 | **SHA-2 해시** | B |
| 사후 추적 | **감사 로그** | C |
| 디스크/백업 도난 방어 | **TDE**(저장 레벨 투명 암호화) | 🔴 데모(이론) |

## E. 체크리스트

- [ ] `AES_ENCRYPT` 로 저장 → `HEX()` 로 암호문 확인 → 올바른 키로만 복호화
- [ ] 틀린 키로는 복호화 실패(NULL) 확인
- [ ] 비밀번호를 `SHA2` 해시로 저장하고, 해시 비교로 로그인 검증
- [ ] `mysql.general_log` 에서 실행된 쿼리 기록 확인
- [ ] 암호화(되돌림 가능)와 해시(되돌림 불가)의 차이를 설명할 수 있음

> 이로써 **05. DB 보안** 을 마칩니다. 다음은 **06. SSL/TLS** — 암호화 통신의 핵심인 Handshake 과정(이론)과 인증서 생성·패킷 분석 실습으로 이어집니다.
{: .prompt-info }
