---
title: "[Web Security Lab] 05. Authentication Attacks"
date: 2026-06-01 14:00:00 +0900
categories:
  - 강의
  - 웹보안
  - 인증공격
tags:
  - Authentication
  - BruteForce
  - DVWA
  - Hydra
  - Burp-Suite
  - 웹모의해킹
pin: false
math: false
mermaid: true
---

## 실습 환경

| 항목 | 내용 |
|---|---|
| 공격 머신 | Kali Linux — `192.168.0.10` |
| 대상 서버 | Ubuntu + DVWA — `192.168.0.30/DVWA/` |
| 주요 도구 | Hydra, Burp Suite, SQLmap |

---

## OWASP Top 10 매핑

| 항목 | 내용 |
|---|---|
| OWASP 카테고리 | **A07:2025 – Authentication Failures** (2021: Identification and Authentication Failures) |
| CWE | CWE-307 (인증 시도 횟수 제한 미흡) · CWE-521 (취약한 비밀번호 요구사항) |
| 영향 | 무차별 대입·사전 공격으로 계정 탈취, 관리자 계정 점유 시 전체 시스템 장악 |
| 한 줄 핵심 | 로그인 시도 제한·강력한 자격증명·다단계 인증이 없어 **자격증명을 추측·대입으로 뚫을 수 있음** |

> A07은 2017 "Broken Authentication" → 2021 "Identification and Authentication Failures" → **2025 "Authentication Failures"** 로 명칭이 정리되어 온 항목입니다(번호는 줄곧 A07).  
> 무차별 대입(Brute Force)·기본 계정·약한 비밀번호 정책·세션 관리 미흡(06장)이 모두 여기에 속합니다.
{: .prompt-info }

---

## 공격의 역사와 주요 사건

### 유래와 발견

무차별 대입(Brute Force)은 **암호의 역사만큼 오래된** 가장 원초적인 공격입니다. "맞을 때까지 전부 대입한다"는 단순함 때문에 늘 마지막 수단으로 살아남았습니다. 인터넷 시대에 들어 이 공격이 진화한 형태가 **크리덴셜 스터핑(Credential Stuffing)** 인데, 이 용어는 2011년경 보안 기업 Shape Security의 공동창업자 **Sumit Agarwal**이 만든 것으로 알려져 있습니다. "한 사이트에서 유출된 아이디·비밀번호 쌍을 **다른 사이트에 그대로 욱여넣는다(stuffing)**"는 의미입니다.

실습에서 쓰는 단어 목록 **`rockyou.txt`** 자체가 역사의 산물입니다. 2009년 소셜 앱 업체 **RockYou가 3,200만 개의 비밀번호를 평문 그대로** 저장하다 통째로 유출당했고, 이 목록이 "사람들이 실제로 쓰는 비밀번호 사전"으로 지금까지 사용됩니다.

### 주요 침해 사건

| 연도 | 사건 | 내용 |
|---|---|---|
| 2009 | **RockYou** | 3,200만 평문 비밀번호 유출 → 사전 공격용 `rockyou.txt`의 기원 |
| 2012 | **LinkedIn** | 솔트 없는 SHA-1 해시 대량 유출(이후 1억 1,700만 건으로 확인) → 재사용 계정을 노린 **크리덴셜 스터핑**의 연료가 됨 |
| 2014 | **iCloud "Celebgate"** | Find My iPhone API에 **로그인 시도 횟수 제한이 없어** 무차별 대입 도구(iBrute)가 통함 + 피싱 → 유명인 사진 유출. 사건 후 Apple이 **Rate Limit** 도입 |
| 2016 | **Mirai 봇넷** | IoT 기기의 **기본 계정(admin/admin 등)** 을 사전 대입으로 장악 → 사상 최대급 DDoS(Dyn 마비) |

> iCloud·Mirai 사건의 공통 원인은 화려한 기법이 아니라 **"시도 횟수 제한(Rate Limit)·기본 계정 변경"이라는 기초 통제의 부재**였습니다. 인증 공격은 대개 기본기로 막힙니다.
{: .prompt-warning }

### 왜 이 공격이 통하는가 — 근본 원리

로그인은 본질적으로 **"맞으면 통과, 틀리면 거부"** 라는 단순 판정입니다. 그래서 공격자가 **무한히, 빠르게, 자동으로** 후보를 던질 수 있다면 — 시간만 충분하면 — 언젠가는 맞습니다. 이 공격을 가능하게 하는 조건은 세 가지입니다.

1. **시도 제한이 없다**: 계정 잠금·Rate Limit이 없으면 Hydra·Burp Intruder가 초당 수백 건을 던질 수 있습니다.
2. **비밀번호가 약하거나 재사용된다**: 사람은 기억하기 쉬운(=추측하기 쉬운) 비밀번호를 쓰고, 여러 사이트에 같은 걸 돌려씁니다 → 사전 공격·크리덴셜 스터핑이 성립.
3. **단서가 새어 나간다**: "아이디가 없습니다" vs "비밀번호가 틀렸습니다"처럼 **응답이 다르면**, 공격자는 유효한 아이디부터 추려낼 수 있습니다(사용자 열거, User Enumeration).

```
시도 제한 없음 + 약한/재사용 비밀번호 + 응답 차이
        → 자동화 도구가 "맞을 때까지" 던지면 결국 뚫림
```

그래서 방어는 **"한 번의 추측을 강하게"(긴 비밀번호·MFA)** 만들거나, **"추측 횟수를 제한"(계정 잠금·Rate Limit·CAPTCHA)** 하거나, **"단서를 없애"(응답 메시지 통일)** 는 세 방향으로 갑니다. 가장 강력한 것은 비밀번호가 뚫려도 막아 주는 **MFA(다단계 인증)** 입니다.

---

## 1. 웹 인증 취약점 개요

**인증(Authentication)** 은 사용자가 주장하는 신원을 확인하는 과정입니다.  
웹 서비스에서 로그인은 가장 흔한 인증 지점이며, 동시에 공격자가 가장 먼저 노리는 지점이기도 합니다.

### 1.1 인증 공격의 주요 유형

| 유형 | 설명 | 특징 |
|---|---|---|
| **Brute Force** | 가능한 모든 조합을 시도 | 느리지만 확실 |
| **Dictionary Attack** | 자주 쓰이는 패스워드 목록으로 시도 | 빠름, 현실적으로 많이 쓰임 |
| **Credential Stuffing** | 다른 곳에서 유출된 계정 재사용 | 이미 검증된 쌍을 사용 |
| **Default Credentials** | 기본 계정(admin/admin 등) 테스트 | 설정 실수를 노림 |
| **Password Spraying** | 흔한 패스워드 하나로 여러 계정 시도 | 계정 잠금 우회 목적 |

---

## 2. Brute Force vs Dictionary Attack

두 공격은 "모르는 패스워드를 찾는다"는 목표는 같지만 접근 방식이 다릅니다.

### 2.1 Brute Force (무차별 대입)

- 가능한 **모든 문자 조합**을 처음부터 끝까지 순차적으로 시도합니다.
- 예: `a`, `b`, ..., `z`, `aa`, `ab`, ..., `zz`, `aaa`, ...
- 시간이 매우 오래 걸리지만, 충분한 시간만 있다면 **반드시** 패스워드를 찾아냅니다.

### 2.2 Dictionary Attack (사전 공격)

- 실제로 자주 쓰이는 패스워드 목록(wordlist)을 이용해 시도합니다.
- 예: `password`, `123456`, `qwerty`, `iloveyou` ...
- Brute Force보다 훨씬 빠르며, 실전에서 성공률이 높습니다.

### 2.3 혼합 공격 (Hybrid Attack)

- 사전 단어에 숫자·특수문자를 조합합니다.
- 예: `password` → `Password1`, `p@ssw0rd`, `password2024`
- Hashcat, John the Ripper 같은 도구의 Rule-based 기능이 이를 자동화합니다.

```mermaid
flowchart TD
    A[공격자] --> B{공격 유형 선택}
    B --> C[Brute Force</br>모든 조합 대입]
    B --> D[Dictionary Attack</br>사전 파일 대입]
    B --> E[Hybrid Attack</br>사전 + 변형 규칙]
    C --> F{패스워드 일치?}
    D --> F
    E --> F
    F -->|Yes| G[로그인 성공</br>계정 탈취]
    F -->|No| H[다음 후보 시도]
    H --> F
```

---

## 3. 취약한 인증 사례

아래와 같은 설계 결함이 있으면 인증 공격이 쉬워집니다.

| 취약점 | 설명 |
|---|---|
| **계정 잠금 정책 부재** | 실패 횟수 제한 없이 무한 시도 가능 |
| **짧은 비밀번호 허용** | 4자리 이하 비밀번호는 순식간에 크래킹 |
| **기본 비밀번호 미변경** | `admin/admin`, `admin/password` 그대로 운영 |
| **Rate Limiting 없음** | 초당 수천 건의 요청도 차단 없이 처리 |
| **에러 메시지 과다 노출** | "아이디가 틀렸습니다" vs "비밀번호가 틀렸습니다" — 계정 존재 여부 노출 |

> **주의**: Rate Limiting이 없다면 Hydra 같은 도구가 초당 수백 건의 로그인을 시도해 패스워드를 빠르게 찾아낼 수 있습니다.
{: .prompt-warning }

---

## 4. DVWA 실습 — Security Level: Low

### 4.0 레벨별 공격·방어 한눈에 (Brute Force 모듈)

| 레벨 | 서버 측 방어 | 공격(우회) 방안 | 그 레벨의 방어 한계 |
|---|---|---|---|
| **Low** | 없음 (시도 제한·지연·토큰 모두 없음) | Hydra·Burp Intruder로 초고속 자동 대입, SQLi 로그인 우회(`' or 1=1 -- `) | 자동화에 무방비 |
| **Medium** | 실패 시 `sleep(2)` 고정 지연 | 스레드를 늘려 **병렬 요청**으로 지연 상쇄, 시간만 더 들 뿐 가능 | 지연은 대입 자체를 막지 못함 |
| **High** | 요청마다 **Anti-CSRF 토큰(user_token)** + 랜덤 지연 | 매 요청 응답에서 토큰 파싱 후 다음 요청에 첨부(Burp `Recursive grep`/`csrftoken` 옵션) | 자동화를 늦출 뿐 차단은 아님 |
| **Impossible** | **PDO Prepared Statement + 계정 잠금(실패 누적 시 차단) + 토큰** | 잠금으로 일정 횟수 후 차단되어 대입 불가 | — (근본 방어) |

> 핵심: 지연(Medium)·토큰(High)은 자동화를 **늦출 뿐** 막지 못합니다.  
> Impossible의 **계정 잠금·Rate Limiting**(7장)이 무차별 대입을 실제로 차단합니다. SQLi 우회까지 막으려면 **Prepared Statement**도 필수입니다.
{: .prompt-tip }

### 4.1 수동 테스트로 취약점 확인

1. 브라우저에서 `http://192.168.0.30/DVWA/` 접속 후 로그인합니다.
2. 좌측 메뉴에서 **Brute Force** 를 클릭합니다.
3. 잘못된 계정(`test / test`)을 입력하면 다음과 같은 메시지가 출력됩니다.

   ```
   Username and/or password incorrect.
   ```

4. 에러 메시지가 아이디·패스워드 구분 없이 통합 메시지를 출력하는지 확인합니다.  
   (구분하면 계정 존재 여부가 노출되는 취약점)

---

### 4.2 실습 1: Burp Suite Intruder로 자동화

Burp Suite의 **Intruder** 기능으로 패스워드 목록 대입 공격을 수행합니다.

#### 단계별 진행

**1단계: 트래픽 캡처**

1. Burp Suite를 실행하고 Proxy > Intercept를 **ON**으로 설정합니다.
2. DVWA Brute Force 페이지에서 임의의 계정을 입력하고 **Login** 버튼을 클릭합니다.
3. Burp Suite에 다음과 같은 GET 요청이 캡처됩니다.

   ```http
   GET /DVWA/vulnerabilities/brute/?username=admin&password=test&Login=Login HTTP/1.1
   Host: 192.168.0.30
   Cookie: PHPSESSID=abcdef1234567890; security=low
   ```

**2단계: Intruder 설정**

1. 캡처된 요청을 **우클릭 → Send to Intruder** 합니다.
2. Intruder 탭 > **Positions** 탭으로 이동합니다.
3. Attack type을 **Sniper**로 설정합니다 (패스워드 하나만 변경).
4. `password=§test§` — `test` 부분만 `§`로 감싸서 페이로드 위치로 지정합니다.

**3단계: 페이로드 설정**

1. **Payloads** 탭으로 이동합니다.
2. Payload type: **Simple list**.
3. **Load** 버튼으로 wordlist 파일을 불러옵니다.

   ```
   /usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt
   ```

4. **Start attack** 를 클릭합니다.

**4단계: 결과 분석**

- 응답 길이(Length)가 다른 항목이 패스워드 일치 결과입니다.
- `admin / password` 항목의 응답 크기가 다른 것과 다르면 성공입니다.

> **팁**: Burp Suite Community Edition은 Intruder 속도가 제한됩니다. Cluster Bomb 설정 시 username + password 두 필드를 동시에 변경할 수 있습니다.
{: .prompt-tip }

---

### 4.3 실습 2: Hydra로 Brute Force

**Hydra**는 네트워크 로그인 서비스를 대상으로 하는 강력한 패스워드 크래킹 도구입니다.

#### HTTP GET 방식 (DVWA Low)

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  192.168.0.30 \
  http-get-form \
  "/DVWA/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:Username and/or password incorrect.:H=Cookie: PHPSESSID=<세션ID>; security=low"
```

#### HTTP POST 방식 (DVWA 로그인 페이지)

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  192.168.0.30 \
  http-post-form \
  "/DVWA/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed"
```

#### Hydra 주요 옵션 설명

| 옵션 | 의미 |
|---|---|
| `-l admin` | 고정 사용자명 (소문자 l) |
| `-L userlist.txt` | 사용자명 목록 파일 |
| `-p password` | 고정 패스워드 |
| `-P rockyou.txt` | 패스워드 목록 파일 |
| `-t 4` | 동시 연결 스레드 수 (기본 16) |
| `-V` | 시도 중인 계정/패스워드 상세 출력 |
| `^USER^` | 사용자명 치환 위치 |
| `^PASS^` | 패스워드 치환 위치 |
| 마지막 콜론 뒤 문자열 | 로그인 실패 시 응답에 포함되는 문자열 (실패 판별) |

> **세션 ID 확인**: Kali 브라우저에서 DVWA 로그인 후 개발자 도구(F12) > Application > Cookies에서 `PHPSESSID` 값을 복사합니다.
{: .prompt-tip }

#### 예상 출력

```
[80][http-get-form] host: 192.168.0.30   login: admin   password: password
1 of 1 target successfully completed, 1 valid password found
```

---

### 4.4 실습 3: SQLmap으로 로그인 우회

인증 페이지에 SQL Injection 취약점이 함께 있다면, **SQLmap**으로 로그인을 우회할 수 있습니다.

```bash
sqlmap -u "http://192.168.0.30/DVWA/vulnerabilities/brute/?username=admin&password=test&Login=Login" \
  --cookie="PHPSESSID=<세션ID>; security=low" \
  --data="username=admin&password=test&Login=Login" \
  --batch \
  --dbs
```

> DVWA Brute Force 페이지는 SQL Injection 취약점이 별도 모듈로 분리되어 있습니다. 이 명령은 SQLi가 존재하는 경우를 전제로 합니다.
{: .prompt-info }

---

### 4.5 실습 4: 기본 계정 테스트

DVWA에는 기본적으로 다음 계정이 세팅되어 있습니다.  
실제 환경에서도 이런 기본 계정이 변경되지 않은 채 남아 있는 경우가 많습니다.

| 사용자명 | 패스워드 |
|---|---|
| admin | password |
| gordonb | abc123 |
| 1337 | charley |
| pablo | letmein |
| smithy | password |

---

## 5. Medium / High 레벨 분석

### 5.1 Medium 레벨 — 딜레이 추가

Medium 레벨은 로그인 실패 시 **2초 딜레이**를 추가합니다.  
Hydra는 기본적으로 빠른 속도로 요청하므로, 딜레이 없이 그대로 실행하면 타임아웃이 발생할 수 있습니다.

```bash
# -t 1: 스레드를 1개로 줄여 동시 요청을 제한
# -w 5: 응답 대기 시간 5초
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  -t 1 -w 5 \
  192.168.0.30 \
  http-get-form \
  "/DVWA/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:incorrect:H=Cookie: PHPSESSID=<세션ID>; security=medium"
```

### 5.2 High 레벨 — CSRF Token 우회

High 레벨은 매 요청마다 새로운 **CSRF Token**을 삽입합니다.  
단순 반복 공격은 토큰 불일치로 모두 실패합니다.

**Burp Suite Intruder로 우회**:

1. Intruder > **Options** 탭으로 이동합니다.
2. **Grep - Extract** 섹션에서 **Add** 를 클릭합니다.
3. 응답 본문에서 `user_token` 값이 포함된 부분을 선택해 정규식으로 추출하도록 설정합니다.

   ```
   value='([0-9a-f]+)' name='user_token'
   ```

4. Recursive Grep를 활성화하면 이전 응답에서 추출한 토큰을 다음 요청에 자동으로 삽입합니다.

```mermaid
sequenceDiagram
    participant B as Burp Intruder
    participant S as DVWA Server

    B->>S: GET /brute/ (첫 요청)
    S->>B: HTML + user_token=abc123
    Note over B: Grep-Extract로 토큰 추출
    B->>S: GET /brute/?password=test&user_token=abc123
    S->>B: 실패 응답 + user_token=def456
    Note over B: 다음 토큰으로 교체
    B->>S: GET /brute/?password=password&user_token=def456
    S->>B: 로그인 성공!
```

> High의 한계: 토큰은 응답에 노출되므로 Burp가 매번 추출해 첨부하면 자동화가 가능합니다. **속도만 늦출 뿐 차단은 아닙니다.**
{: .prompt-warning }

### 5.3 Impossible 레벨 — 방어 성공

Impossible 레벨은 세 가지를 결합합니다.

1. **PDO Prepared Statement** — 로그인 쿼리의 SQL Injection 우회(`' or 1=1 -- `)를 차단합니다.
2. **계정 잠금(Account Lockout)** — 일정 횟수(예: 3회) 실패 시 일정 시간 계정을 잠가 추가 시도를 막습니다.
3. **Anti-CSRF 토큰** — 외부 자동화 도구의 위조 요청을 차단합니다.

```php
// Impossible 핵심: 실패 누적 시 잠금
if ($failed_login_count >= 3) {
    // 마지막 실패 후 일정 시간 동안 로그인 차단
}
```

무차별 대입은 **시도 자체가 차단**되므로 Hydra·Intruder로도 뚫을 수 없습니다.  
실무에서는 여기에 **Rate Limiting · CAPTCHA · MFA**(7장)를 더해 다층으로 방어합니다.

---

## 6. 공격 흐름 요약

```mermaid
flowchart LR
    A[정보 수집</br>세션ID 획득] --> B[도구 선택]
    B --> C[Hydra</br>HTTP Form]
    B --> D[Burp Suite</br>Intruder]
    B --> E[SQLmap</br>SQLi 취약점]
    C --> F[패스워드 크래킹]
    D --> F
    E --> G[로그인 우회]
    F --> H{성공?}
    G --> H
    H -->|Yes| I[계정 탈취</br>세션 획득]
    H -->|No| J[보안 레벨 분석</br>우회 기법 적용]
    J --> B
```

---

## 7. 방어 방법

인증 공격을 막기 위한 핵심 방어 기법은 다음과 같습니다.

### 7.1 계정 잠금 정책

```
5회 연속 실패 → 30분 잠금
관리자에게 알림 전송
```

- 단, 지나치게 엄격하면 **서비스 거부(DoS)** 로 악용될 수 있습니다.  
  모든 계정을 잠그는 **Password Spraying 방어**도 함께 고려해야 합니다.

### 7.2 Rate Limiting

```
동일 IP에서 분당 5회 이상 로그인 실패 → 임시 차단
nginx / WAF 레벨에서 설정 가능
```

### 7.3 CAPTCHA

- 자동화된 봇 공격을 차단합니다.
- reCAPTCHA v3는 사용자 경험을 해치지 않으면서 봇을 탐지합니다.

### 7.4 MFA (Multi-Factor Authentication)

- 패스워드가 유출되더라도 추가 인증 수단이 없으면 로그인할 수 없습니다.
- TOTP(시간 기반 OTP) 앱, 하드웨어 키(FIDO2) 등을 활용합니다.

### 7.5 강력한 비밀번호 정책

| 항목 | 권장 기준 |
|---|---|
| 최소 길이 | 12자 이상 |
| 복잡도 | 대문자, 소문자, 숫자, 특수문자 혼합 |
| 재사용 제한 | 최근 5개 패스워드 재사용 금지 |
| 유출 패스워드 차단 | HaveIBeenPwned 같은 DB와 비교 |

### 7.6 에러 메시지 통일

```
// 취약한 메시지 (X)
"아이디가 존재하지 않습니다."
"비밀번호가 틀렸습니다."

// 안전한 메시지 (O)
"아이디 또는 비밀번호가 올바르지 않습니다."
```

---

## 8. 핵심 정리

1. **Brute Force**는 모든 조합을, **Dictionary Attack**은 자주 쓰이는 목록을 사용합니다.
2. **Hydra**는 HTTP Form 방식의 로그인 대입 공격에 효과적입니다.
3. **Burp Suite Intruder**는 요청을 세밀하게 제어하며, CSRF 토큰 우회도 가능합니다.
4. DVWA Medium 레벨은 딜레이, High 레벨은 CSRF Token으로 추가 방어합니다.
5. 방어의 핵심은 **계정 잠금 + Rate Limiting + MFA + 강력한 패스워드 정책**의 조합입니다.

---

## 9. 정보보안기사 시험 포인트

| 구분 | 꼭 외울 것 |
|---|---|
| **무차별 대입** | 로그인 시도 횟수 제한(계정 잠금)·Rate Limit이 없을 때 성립 |
| **기본 계정** | `admin`/`administrator`/`manager` 등 기본 계정·기본 비밀번호 점검은 단골 점검 항목 |
| **관리자 페이지 노출** | 추측 쉬운 경로(`/admin`)·포트(8080/8443) 노출, **인증 없이 중간 페이지 직접 접근** 여부 점검 → 경로명을 추측 어렵게 변경 |
| **에러 메시지** | "아이디 없음" vs "비밀번호 틀림"을 구분하면 **사용자 열거** 가능 → 메시지 **통일** |

> **★ 함께 보는 주제**: **불충분한 세션 만료**(세션 타임아웃 미설정/과도)와 **관리자 페이지 노출**은 인증·세션 단원에서 자주 묶여 나옵니다. 상세 이론은 **11번 포스트 3.3~3.4장**, 세션 자체는 **06번 포스트** 참고.
{: .prompt-tip }
