---
title: "[애플리케이션 보안] 03-2. 이메일 공격 유형과 보안 대책"
date: 2026-10-19 10:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - 메일보안
tags:
  - EmailSpoofing
  - OpenRelay
  - ActiveContents
  - SpamFilter
  - SPF
  - DMARC
  - Postfix
math: false
mermaid: false
---

> **03-1** 에서 만든 메일 서버(Postfix, 수신 계정 `mailuser`)가 동작 중이어야 합니다.
{: .prompt-info }

## 0. 실습 환경

| 장비 | IP | 역할 |
|---|---|---|
| Kali | **192.168.0.10** | 공격자·메일 발송자 |
| Ubuntu | **192.168.0.30** | 메일 서버(Postfix) |

> 명령과 예상 화면은 **Ubuntu 24.04.4 LTS(Postfix 3.8.6)** 와 **Kali Rolling** 에서 실행해 확인했습니다. 큐 번호, 시각, 로그의 호스트 이름처럼 환경마다 다른 값은 화면과 다를 수 있습니다. 4.2절의 SPF 조회는 **Kali가 인터넷에 연결**되어 있어야 합니다.
{: .prompt-warning }

절마다 Kali와 Ubuntu에서 하는 일은 다음과 같습니다. Kali는 **공격하거나 시험 메일을 보내는 쪽**, Ubuntu는 **결과를 확인하고 막는 쪽**입니다.

| 절 | Kali에서 할 일 | Ubuntu에서 할 일 |
|---|---|---|
| 3.2 발신자 위조 | `admin@bank.com` 으로 위장한 메일 보내기 | 받은 메일의 `Received` 에서 실제 IP 찾기 |
| 3.3 오픈 릴레이 | 외부 도메인으로 보내기: 성공(취약) → 거부(안전) | `mynetworks` 를 넓혀 릴레이를 열었다가 되돌림 |
| 3.4 VRFY | 계정 확인: 구별됨(취약) → 막힘(안전) | `disable_vrfy_command = yes` 로 차단 |
| 3.5 Active Contents | (이론만) | (이론만) |
| 4.1 첨부파일 필터링 | `.exe` 첨부: 접수(취약) → 거부(안전) | 메일함 확인, 차단 규칙 만들기, 로그 확인 |
| 4.2 SPF·DMARC | 실제 도메인의 레코드 조회 | — |
| 4.3 STARTTLS | 평문 메일: 접수(취약) → 거부(안전) | 암호화 필수(`encrypt`)로 바꿨다가 되돌림 |

> 3.3·4.1·4.3·3.4는 모두 **취약한 상태에서 공격이 통하는 것을 먼저 보고, 설정을 바꿔 막히는 것을 확인**하는 순서입니다. 3.3과 4.3은 실습 끝에 설정을 **원래대로 되돌리는 단계**가 있으니 빠뜨리지 않습니다.
{: .prompt-info }

---

# 3. 이메일 공격 유형

## 3.1 공격 유형 한눈에 보기

### [이론]

메일 공격은 **누구를 노리느냐**에 따라 두 갈래로 나눕니다. 어느 쪽에 속하는지 구분하는 것이 중요합니다.

| 대상                | 공격                     | 원리                                                       | 대책                      |
| ----------------- | ---------------------- | -------------------------------------------------------- | ----------------------- |
| **메일 클라이언트**(사용자) | **Active Contents 공격** | 메일을 열 때 HTML을 해석하는 메일 프로그램·웹 브라우저에서 **스크립트가 실행**되어 정보 유출 | 메일 프로그램의 **스크립트 기능을 끔** |
|                   | **트로이 목마·악성 첨부**       | 정상 파일처럼 보이는 첨부파일을 열면 악성코드 실행                             | 첨부파일 필터링, 백신 검사         |
|                   | 피싱                     | 위조한 발신자로 속여 비밀번호·금융정보 입력을 유도                             | SPF·DMARC, 사용자 교육       |
| **메일 서버**         | **버퍼 오버플로우 공격**        | Sendmail 같은 **메일 서버 프로그램의 취약점**을 이용해 서버 권한 획득            | 최신 버전·패치                |
|                   | **서비스 거부(DoS) 공격**     | 대량 메일(메일 폭탄)·대량 접속으로 서버를 마비                              | 스팸 필터, 접속·크기 제한         |
|                   | 스팸 릴레이 악용              | 외부인이 서버를 **스팸 중계소**로 사용                                  | 릴레이 제한                  |
|                   | 계정 정보 수집               | `VRFY`·`EXPN` 명령으로 있는 계정을 알아냄                            | 명령 비활성화                 |

> **Active Contents 공격과 트로이 목마 공격은 클라이언트**를 노리는 공격이고, **버퍼 오버플로우와 서비스 거부 공격은 서버**를 노리는 공격입니다. 바이러스 메일은 메일 서버를 공격하는 것이 아니라 **사용자**를 노립니다.
{: .prompt-warning }

> **피싱 대책 "SPF·DMARC, 사용자 교육"의 뜻**
>
> - **SPF·DMARC**: 받는 서버가 위조 메일을 **자동으로 걸러 내는** 기술입니다. SPF는 도메인 주인이 "우리 메일은 이 IP에서만 나간다"는 목록을 DNS에 공개해, 목록에 없는 곳에서 온 메일을 가려냅니다. DMARC는 검사에 실패한 메일을 보고만 할지(`none`), 스팸함에 넣을지(`quarantine`), 거부할지(`reject`) 정합니다(4.2절).
> - **사용자 교육**: 서버를 빠져나온 메일에 **사람이 속지 않게** 하는 대책입니다. 보낸 사람 주소를 끝까지 확인하고, 링크에 마우스를 올려 실제 주소를 보고, 메일로 비밀번호를 요구하거나 급하게 재촉하면 의심하며, 의심스러우면 공식 홈페이지·전화로 따로 확인하도록 가르칩니다. 회사에서는 가짜 피싱 메일을 보내 보는 **모의 피싱 훈련**으로 연습합니다.
>
> SPF·DMARC는 도메인 주인이 **미리 등록해야** 작동하고, `bank-security.com` 처럼 **비슷한 진짜 도메인**을 사서 보내면 통과합니다. 그래서 두 대책을 함께 씁니다.
{: .prompt-info }

---

## 3.2 발신자 위조(스푸핑)

### [이론]

03-1에서 본 것처럼 SMTP는 **보내는 사람을 확인하지 않습니다.** `MAIL FROM` 과 `From:` 에 어떤 주소를 적어도 그대로 받아 줍니다. 공격자는 이를 이용해 은행·관리자·거래처로 **위장한 메일**을 보내고, 받은 사람이 링크를 누르거나 비밀번호를 입력하도록 속입니다.

다만 `Received` 헤더에는 서버가 직접 확인한 **실제 접속 IP**가 남습니다. 위조 메일을 가려낼 때 `Received` 를 보는 이유입니다.

### [실습 1] (Kali) 은행 관리자로 위장한 메일 보내기

```bash
# === Kali ===
nc -C 192.168.0.30 25
```

`nc -C` 는 03-1에서 쓴 것과 같은 명령입니다. Kali에서 Ubuntu의 25번 포트로 TCP 연결을 열고, 입력한 SMTP 명령을 그대로 보냅니다. `-C` 는 줄 끝을 SMTP 규칙(CR+LF)에 맞추는 옵션입니다.

> `nc` 로 접속한 상태에서는 **프롬프트(`$`)가 나오지 않고** 커서만 깜빡입니다. 정상이며, 그대로 명령을 입력하면 됩니다. 오타를 냈거나 `QUIT` 뒤에도 프롬프트가 돌아오지 않으면 `Ctrl+C` 로 빠져나와 다시 접속합니다. 이 글의 다른 `nc` 실습도 같습니다.
{: .prompt-info }

아래를 한 줄씩 입력합니다. 공격자가 `bank.com` 이라는 은행의 관리자를 사칭하는 상황입니다. Kali는 `bank.com` 과 아무 관계가 없습니다.

```
EHLO bank.com
MAIL FROM:<admin@bank.com>
RCPT TO:<mailuser@ubuntu.lab.local>
DATA
From: Bank Admin <admin@bank.com>
To: mailuser@ubuntu.lab.local
Subject: Check your account

Please log in to confirm your password.
.
QUIT
```

터미널에는 입력한 줄과 서버 응답이 번갈아 나타납니다. 전체 화면은 다음과 같습니다.

```
220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
EHLO bank.com
250-ubuntu.lab.local
250-PIPELINING
250-SIZE 10240000
250-VRFY
250-ETRN
250-STARTTLS
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING
MAIL FROM:<admin@bank.com>
250 2.1.0 Ok
RCPT TO:<mailuser@ubuntu.lab.local>
250 2.1.5 Ok
DATA
354 End data with <CR><LF>.<CR><LF>
From: Bank Admin <admin@bank.com>
To: mailuser@ubuntu.lab.local
Subject: Check your account

Please log in to confirm your password.
.
250 2.0.0 Ok: queued as 2DCF37AB
QUIT
221 2.0.0 Bye
```

남의 주소를 적었는데도 **아무 확인 없이 `250 Ok` 로 접수**되었습니다.

### [실습 2] (Ubuntu) 받은 메일에서 위조 흔적 찾기

```bash
# === Ubuntu ===
sudo tail -n 15 /var/mail/mailuser
```

> `tail -n 15` 는 파일의 **마지막 15줄**만 보여 주는 명령입니다. 메일함 파일에는 메일이 도착한 순서대로 쌓이므로, 끝부분을 보면 방금 받은 메일만 확인할 수 있습니다.
{: .prompt-info }

```
From admin@bank.com  Mon Oct 19 22:45:26 2026
Return-Path: <admin@bank.com>
X-Original-To: mailuser@ubuntu.lab.local
Delivered-To: mailuser@ubuntu.lab.local
Received: from bank.com (unknown [192.168.0.10])
	by ubuntu.lab.local (Postfix) with ESMTP id 2DCF37AB
	for <mailuser@ubuntu.lab.local>; Mon, 19 Oct 2026 22:45:23 +0900 (KST)
From: Bank Admin <admin@bank.com>
To: mailuser@ubuntu.lab.local
Subject: Check your account
Message-Id: <20261019134524.2DCF37AB@ubuntu.lab.local>
Date: Mon, 19 Oct 2026 22:45:23 +0900 (KST)

Please log in to confirm your password.
```

| 헤더                                    | 값                   | 판단                          |
| ------------------------------------- | ------------------- | --------------------------- |
| `From`, `Return-Path`                 | `admin@bank.com`    | 공격자가 **적은 대로** 보임 → 믿을 수 없음 |
| `Received` 의 `from bank.com`          | `EHLO` 로 공격자가 적은 이름 | 믿을 수 없음                     |
| `Received` 의 `unknown [192.168.0.10]` | 서버가 **직접 확인한 IP**   | **Kali에서 왔다는 증거**           |

> 메일 프로그램은 보통 `From` 만 보여 주므로 사용자는 진짜 은행 메일로 착각합니다. 이런 위조를 **수신 서버가 자동으로 가려내게** 하는 기술이 4.2절의 SPF입니다.
{: .prompt-info }

### 정리

| 장비 | 한 일 | 결과 |
|---|---|---|
| Kali | `MAIL FROM`·`From:` 에 남의 주소(`admin@bank.com`)를 적어 발송 | 아무 확인 없이 `250 Ok` 로 접수 |
| Ubuntu | 메일함에서 헤더 확인 | `From` 은 위조 주소, `Received` 에는 실제 IP `192.168.0.10` |

> - SMTP는 보내는 사람이 **누구인지 확인하지 않으므로** `From` 과 `Return-Path` 는 얼마든지 위조됩니다.
> - `Received` 의 `from` 뒤 이름도 `EHLO` 로 공격자가 적은 값이라 믿을 수 없습니다. 믿을 수 있는 것은 **대괄호 안의 IP** 뿐입니다.
> - 대책은 받는 서버가 발신 도메인을 확인하는 **SPF·DKIM·DMARC**(4.2절)와, 사용자가 링크·첨부를 함부로 열지 않는 습관입니다.
{: .prompt-tip }

---

## 3.3 스팸 릴레이(Open Relay)

### [이론]

**릴레이(relay)** 는 메일 서버가 자기 도메인이 아닌 **다른 도메인으로 가는 메일을 대신 전달**하는 것입니다. 아무나 릴레이를 쓸 수 있는 서버를 **오픈 릴레이(Open Relay)** 라고 합니다.

| 오픈 릴레이일 때 | 결과 |
|---|---|
| 스패머가 내 서버를 거쳐 대량 스팸 발송 | 내 서버가 **스팸 발송지**로 찍힘 |
| 스팸 차단 목록(블랙리스트)에 등록 | 정상 메일까지 상대 서버에서 거부됨 |
| 서버 자원 소모 | 서비스 거부 상태 |

그래서 메일 서버는 **허용한 네트워크(내부 사용자)나 인증한 사용자에게만** 릴레이를 허용합니다.

이 절은 **설정을 잘못 넣어 오픈 릴레이를 직접 만든 뒤, 다시 닫는** 순서로 진행합니다.

| 순서 | 장비 | 할 일 | 상태 |
|---|---|---|---|
| 실습 1 | Ubuntu | 릴레이 허용 범위에 실습망 전체를 넣음 | **취약**하게 만듦 |
| 실습 2 | Kali | 외부 도메인으로 메일 보내기 | 접수됨 = 오픈 릴레이 |
| 실습 3 | Ubuntu | 허용 범위를 원래대로 되돌림 | **안전**하게 되돌림 |
| 실습 4 | Kali | 같은 메일을 다시 보내기 | 거부됨 |

### [실습 1] (Ubuntu) 릴레이 허용 범위를 넓혀 오픈 릴레이 만들기

사내 PC에서 메일이 나가지 않는다는 문의를 받은 관리자가, 원인을 찾지 않고 **실습망 전체(`192.168.0.0/24`)를 릴레이 허용 범위에 넣은** 상황을 흉내 냅니다.

먼저 현재 설정을 봅니다.

```bash
# === Ubuntu ===
postconf mynetworks smtpd_relay_restrictions
```

> `postconf 항목이름` 은 Postfix 설정의 **현재 값을 조회**합니다. 03-1에서 쓴 `postconf -e` 는 값을 **수정**하는 옵션이고, `-e` 없이 쓰면 보기만 합니다. 여러 항목을 띄어 쓰면 한 번에 조회할 수 있습니다.
{: .prompt-info }

```
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
```

| 설정 | 의미 |
|---|---|
| `mynetworks = 127.0.0.0/8 ...` | 릴레이를 허용할 네트워크. `127.0.0.0/8` 은 서버 자신(localhost), 뒤의 `[::ffff:127.0.0.0]/104`, `[::1]/128` 은 같은 서버 자신을 IPv6 형식으로 적은 것 → **서버 자신뿐** |
| `permit_mynetworks` | `mynetworks` 에서 온 메일은 릴레이 허용 |
| `permit_sasl_authenticated` | 아이디·비밀번호로 **인증한 사용자**는 릴레이 허용 |
| `defer_unauth_destination` | 그 밖의 사람이 **남의 도메인**으로 보내려 하면 **일시적 실패(4xx)** 로 거부 |

이제 `mynetworks` 끝에 `192.168.0.0/24` 를 덧붙입니다. 앞부분은 원래 값을 그대로 적고 **맨 뒤에만 한 칸 띄어** 추가합니다.

```bash
# === Ubuntu ===
sudo postconf -e 'mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.0.0/24'
sudo systemctl reload postfix
postconf mynetworks
```

`postconf -e` 와 `systemctl reload` 는 아무것도 출력하지 않습니다. 마지막 조회 결과입니다.

```
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.0.0/24
```

| 명령·옵션 | 설명 |
|---|---|
| `192.168.0.0/24` | `192.168.0.0` ~ `192.168.0.255` 전체. **Kali(192.168.0.10)도 여기에 들어감** |
| `systemctl reload postfix` | 서비스를 멈추지 않고 **바뀐 설정만 다시 읽음**. `restart` 는 서비스를 껐다 켜므로 그 사이 접속이 잠깐 끊기지만, `reload` 는 끊기지 않음 |

### [실습 2] (Kali) 외부 도메인으로 메일 보내기 — 릴레이 성공

Kali에서 이 서버가 담당하지 않는 외부 도메인(`example.com`)으로 메일을 보내 봅니다.

```bash
# === Kali ===
nc -C 192.168.0.30 25
```

접속한 뒤 아래를 한 줄씩 입력합니다. `RCPT TO` 의 응답만 확인하고 `DATA` 는 보내지 않으므로 **실제 메일은 나가지 않습니다.**

```
EHLO kali
MAIL FROM:<kali@kali.lab.local>
RCPT TO:<someone@example.com>
QUIT
```

터미널에는 입력한 줄과 서버 응답이 번갈아 나타납니다. 전체 화면은 다음과 같습니다.

```
220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
EHLO kali
250-ubuntu.lab.local
250-PIPELINING
250-SIZE 10240000
250-VRFY
250-ETRN
250-STARTTLS
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING
MAIL FROM:<kali@kali.lab.local>
250 2.1.0 Ok
RCPT TO:<someone@example.com>
250 2.1.5 Ok
QUIT
221 2.0.0 Bye
```

`RCPT TO:<someone@example.com>` 에 **`250 2.1.5 Ok`** 가 나왔습니다. 내 서버가 남의 도메인으로 가는 메일을 대신 전달하겠다고 받아 준 것이므로 **오픈 릴레이 상태**입니다. 이 상태에서는 실습망 안의 아무 PC나, 예를 들어 악성코드에 감염된 PC가 이 서버를 통해 스팸을 보낼 수 있습니다.

> 인터넷에 연결된 서버에서 `mynetworks` 에 `0.0.0.0/0`(모든 IP)을 넣으면, 전 세계 누구나 쓰는 오픈 릴레이가 됩니다. 스패머는 이런 서버를 자동으로 찾아다닙니다.
{: .prompt-danger }

### [실습 3] (Ubuntu) 릴레이 허용 범위를 원래대로 되돌리기

`mynetworks` 를 실습 1에서 처음 본 값(서버 자신만)으로 되돌립니다.

```bash
# === Ubuntu ===
sudo postconf -e 'mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128'
sudo systemctl reload postfix
postconf mynetworks
```

`postconf -e` 와 `systemctl reload` 는 아무것도 출력하지 않습니다. 마지막 조회 결과입니다.

```
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
```

### [실습 4] (Kali) 같은 메일을 다시 보내기 — 릴레이 거부

실습 2와 똑같이 접속해 같은 내용을 입력합니다.

```bash
# === Kali ===
nc -C 192.168.0.30 25
```

```
EHLO kali
MAIL FROM:<kali@kali.lab.local>
RCPT TO:<someone@example.com>
QUIT
```

전체 화면은 다음과 같습니다.

```
220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
EHLO kali
250-ubuntu.lab.local
250-PIPELINING
250-SIZE 10240000
250-VRFY
250-ETRN
250-STARTTLS
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING
MAIL FROM:<kali@kali.lab.local>
250 2.1.0 Ok
RCPT TO:<someone@example.com>
454 4.7.1 <someone@example.com>: Relay access denied
QUIT
221 2.0.0 Bye
```

이번에는 `RCPT TO` 단계에서 **`Relay access denied`** 로 거부되었습니다. 반면 3.2절처럼 **이 서버가 받는 주소**(`@ubuntu.lab.local`)로 보내는 메일은 여전히 정상적으로 받습니다.

> 응답 코드가 `5xx` 가 아니라 `454`(일시적 실패)인 것은 `defer_unauth_destination` 의 `defer`(미룸) 때문입니다. 설정이 잘못된 경우에 대비해 영구 거부 대신 "나중에 다시 시도하라"고 답하는 Postfix의 기본 동작입니다.
{: .prompt-info }

### 정리

| 상태 | `mynetworks` | Kali → `someone@example.com` |
|---|---|---|
| **취약** (실습 1·2) | `... 192.168.0.0/24` (실습망 전체) | `250 2.1.5 Ok` — 릴레이 허용 |
| **안전** (실습 3·4) | `127.0.0.0/8 ...` (서버 자신만, 기본값) | `454 ... Relay access denied` |

> - **오픈 릴레이**는 아무나 내 서버를 거쳐 남의 도메인으로 메일을 보낼 수 있는 상태입니다. 스팸 발송지로 찍혀 블랙리스트에 오르고, 정상 메일까지 거부됩니다.
> - Postfix의 기본값은 안전합니다. 위험은 관리자가 `mynetworks` 를 **필요 이상으로 넓힐 때** 생깁니다.
> - 메일 서버는 **자기 도메인으로 오는 메일은 누구에게서나 받고**, **다른 도메인으로 보내는 메일(릴레이)은 허용한 네트워크나 인증한 사용자에게만** 허용해야 합니다.
{: .prompt-tip }

---

## 3.4 계정 정보 수집 — VRFY와 EXPN

### [이론]

| 명령 | 기능 | 공격에 쓰이는 방법 |
|---|---|---|
| `VRFY 계정` | 그 계정이 서버에 있는지 확인 | 이름을 하나씩 넣어 **실제 있는 계정 목록**을 알아냄 |
| `EXPN 목록` | 메일링 목록에 들어 있는 실제 주소를 펼쳐 보여 줌 | 내부 사용자 주소를 한꺼번에 수집 |

알아낸 계정은 **비밀번호 무차별 대입**이나 **스팸 발송 대상**으로 쓰입니다. 그래서 메일 서버에서는 이 명령들을 **비활성화**합니다.

### [실습 1] (Kali) VRFY로 계정 확인하기

```bash
# === Kali ===
nc -C 192.168.0.30 25
```

접속한 뒤 아래를 한 줄씩 입력합니다.

```
EHLO kali
VRFY root
VRFY mailuser
VRFY nobodyhere
QUIT
```

터미널에는 입력한 줄과 서버 응답이 번갈아 나타납니다. 전체 화면은 다음과 같습니다.

```
220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
EHLO kali
250-ubuntu.lab.local
250-PIPELINING
250-SIZE 10240000
250-VRFY
250-ETRN
250-STARTTLS
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING
VRFY root
252 2.0.0 root
VRFY mailuser
252 2.0.0 mailuser
VRFY nobodyhere
550 5.1.1 <nobodyhere>: Recipient address rejected: User unknown in local recipient table
QUIT
221 2.0.0 Bye
```

| 입력 | 응답 | 공격자가 알게 되는 것 |
|---|---|---|
| `VRFY root` | `252` | `root` 계정이 **있다** |
| `VRFY mailuser` | `252` | `mailuser` 계정이 **있다** |
| `VRFY nobodyhere` | `550 ... User unknown` | 이 계정은 **없다** |

응답 코드가 다르기 때문에 공격자는 계정이 있는지 없는지 구별할 수 있습니다.

> `252` 는 원래 "확인해 줄 수는 없지만 이 주소로 오는 메일은 받겠다"는 뜻입니다. Postfix는 **있는 계정에만** `252` 를, 없는 계정에는 `550` 을 돌려주므로 결과적으로 계정 유무가 드러납니다.
{: .prompt-info }

### [실습 2] (Ubuntu) VRFY 비활성화

```bash
# === Ubuntu ===
postconf disable_vrfy_command
sudo postconf -e 'disable_vrfy_command = yes'
sudo systemctl reload postfix
postconf disable_vrfy_command
```

| 명령·옵션 | 설명 |
|---|---|
| `postconf disable_vrfy_command` | 현재 값 조회(처음에는 `no` = VRFY 허용) |
| `postconf -e 'disable_vrfy_command = yes'` | 값을 `yes`(VRFY 막기)로 수정 |
| `systemctl reload postfix` | 바뀐 설정을 다시 읽어 적용(3.3절 실습 1) |

`postconf -e` 와 `systemctl reload` 는 아무것도 출력하지 않습니다. 첫 줄은 수정 전, 둘째 줄은 수정 후 조회 결과입니다.

```
disable_vrfy_command = no
disable_vrfy_command = yes
```

### [실습 3] (Kali) VRFY가 막혔는지 다시 확인

```bash
# === Kali ===
nc -C 192.168.0.30 25
```

접속한 뒤 아래를 한 줄씩 입력합니다.

```
EHLO kali
VRFY root
QUIT
```

터미널에는 입력한 줄과 서버 응답이 번갈아 나타납니다. 전체 화면은 다음과 같습니다.

```
220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
EHLO kali
250-ubuntu.lab.local
250-PIPELINING
250-SIZE 10240000
250-ETRN
250-STARTTLS
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING
VRFY root
502 5.5.1 VRFY command is disabled
QUIT
221 2.0.0 Bye
```

| 확인 | 의미 |
|---|---|
| `EHLO` 기능 목록에서 **`250-VRFY` 줄이 사라짐** | 서버가 더 이상 VRFY 기능을 알리지 않음 |
| `VRFY root` → **`502 ... disabled`** | 계정이 있는지 알려 주지 않음 |

> Postfix는 `EXPN` 명령을 처음부터 지원하지 않습니다. Sendmail 같은 다른 메일 서버는 `EXPN` 도 따로 막아야 합니다.
{: .prompt-info }

### 정리

| 장비 | 한 일 | 결과 |
|---|---|---|
| Kali | `VRFY root`·`VRFY mailuser`·`VRFY nobodyhere` | 있는 계정 `252`, 없는 계정 `550` → 계정 목록 수집 가능 |
| Ubuntu | `disable_vrfy_command = yes` 적용 후 `reload` | VRFY 비활성화 |
| Kali | 다시 `VRFY root` | `502 ... disabled`, EHLO 목록에서 `VRFY` 사라짐 |

> - `VRFY`·`EXPN` 은 서버에 **실제 있는 계정**을 알려 주는 명령입니다. 알아낸 계정은 비밀번호 무차별 대입과 스팸 발송 대상이 됩니다.
> - 응답 코드가 계정 유무에 따라 **다르게** 나오는 것 자체가 정보 유출입니다. 그래서 명령을 꺼서 아무것도 알려 주지 않게 합니다.
{: .prompt-tip }

---

## 3.5 Active Contents 공격

### [이론]

요즘 메일은 글자만 있는 것이 아니라 **HTML 문서**로 꾸며져 있습니다. 메일 프로그램이나 웹 브라우저는 이 HTML을 해석해서 보여 줍니다. 이때 메일 안에 **자바스크립트, 비주얼베이직 스크립트** 같은 실행 코드가 들어 있으면 메일을 **열기만 해도** 스크립트가 실행될 수 있습니다.

![](/assets/img/posts/2026-10-19-appsec-03-2-mail-attack-security-1793602644444.png)

| 항목 | 내용 |
|---|---|
| 대상 | 메일을 열 때 **HTML 기능이 있는 메일 클라이언트나 웹 브라우저**를 쓰는 사용자 |
| 수법 | 자바스크립트, 비주얼베이직 스크립트 등을 메일에 넣어 실행 |
| 피해 | 사용자 컴퓨터의 정보 유출, 악성코드 설치 |
| 대책 | 메일 프로그램에서 **스크립트 기능을 사용하지 않도록 설정** |

> 웹메일 서비스는 메일 본문의 `<script>` 를 지우고 보여 주며, Outlook 같은 메일 프로그램도 기본으로 스크립트 실행과 외부 이미지 자동 표시를 막습니다. 이 설정을 함부로 풀지 않는 것이 중요합니다.
{: .prompt-tip }

### 정리

이 절은 Kali·Ubuntu 실습 없이 이론으로만 다룹니다.

> - Active Contents 공격은 메일 **서버가 아니라 메일을 여는 사용자(클라이언트)** 를 노립니다.
> - 메일을 **열기만 해도** 스크립트가 실행될 수 있다는 점이 첨부파일 공격과 다릅니다.
> - 대책은 메일 프로그램의 **스크립트 기능을 끄는 것**입니다.
{: .prompt-tip }

---

# 4. 이메일 보안 대책

## 4.1 스팸 필터 솔루션과 첨부파일 필터링

### [이론]

**스팸 필터 솔루션** 은 메일 서버 앞에서 들어오고 나가는 메일을 검사하는 장비·소프트웨어입니다.

| 항목 | 내용 |
|---|---|
| 위치 | **메일 서버 앞단**에 위치하며 **프록시 메일 서버**로 동작 |
| 막는 공격 | SMTP 프로토콜을 이용한 **DoS 공격**, **폭탄 메일**, **스팸 메일** |
| 필터링 범위 | **헤더**와 **제목**뿐 아니라 **본문**까지 검색해 필터링 |
| 첨부파일 | 특정 **확장자**를 가진 첨부파일을 차단하거나 허용한 것만 전송 |
| 바이러스 | 첨부파일의 바이러스 검사 |
| 내부 유출 방지 | 내부에서 **밖으로 나가는 메일**의 본문을 검색해 중요 정보 유출 차단 |

> 스팸 필터는 메일의 **본문까지** 검사합니다. "헤더와 제목만 필터링하고 본문은 하지 못한다"는 설명은 틀립니다.
{: .prompt-warning }

이 절은 **차단 규칙이 없을 때 실행파일이 그대로 들어오는 것을 먼저 확인한 뒤, 규칙을 넣어 막는** 순서로 진행합니다.

| 순서 | 장비 | 할 일 | 상태 |
|---|---|---|---|
| 실습 1 | Kali | 규칙이 없는 서버로 `.exe` 첨부 메일 보내기 | **취약** — 접수됨 |
| 실습 2 | Ubuntu | 실행파일이 메일함까지 들어왔는지 확인 | 사용자에게 도착 |
| 실습 3 | Ubuntu | 실행파일 첨부 차단 규칙 만들기 | **안전**하게 바꿈 |
| 실습 4 | Kali | `.exe`, `.txt` 첨부 메일 다시 보내기 | `.exe` 만 거부됨 |
| 실습 5 | Ubuntu | 메일 로그에서 차단 기록 확인 | 기록 확인 |

### [실습 1] (Kali) 차단 규칙이 없을 때 실행파일 첨부 보내기

첨부파일이 있는 메일은 `nc` 로 손으로 입력하기 어려우므로 메일 테스트 도구 **swaks** 를 씁니다. 설치하고, 시험용 파일 두 개를 만든 뒤, `.exe` 를 붙여 보냅니다.

```bash
# === Kali ===
sudo apt install -y swaks
cd ~ && echo "AAAA" > invoice.exe && echo "hello" > note.txt

swaks --server 192.168.0.30 --from kali@kali.lab.local --to mailuser@ubuntu.lab.local \
  --header "Subject: Invoice" --body "See attached invoice." --attach @invoice.exe --suppress-data
```

| 명령·옵션 | 설명 |
|---|---|
| `apt install -y swaks` | **swaks**(Swiss Army Knife for SMTP) 설치. SMTP 명령을 대신 입력해 주는 메일 테스트 도구 |
| `cd ~ && echo ... > invoice.exe` | 홈 디렉터리로 이동한 뒤 시험용 파일 두 개를 만듦. 내용은 의미 없는 글자이고, **확장자만** 실행파일(`.exe`)과 문서(`.txt`)로 다름. `&&` 는 앞 명령이 성공하면 이어서 실행하라는 뜻 |
| `--server 192.168.0.30` | 접속할 메일 서버 |
| `--from` / `--to` | 보내는 사람 / 받는 사람 주소 (`MAIL FROM` / `RCPT TO` 에 해당) |
| `--header "Subject: ..."` | 메일 헤더 추가(여기서는 제목) |
| `--body "..."` | 메일 본문 |
| `--attach @invoice.exe` | 파일을 첨부. `@` 는 "이 이름의 파일 내용을 읽어서 붙이라"는 뜻 |
| `--suppress-data` | 보낸 메일 내용(헤더·본문·첨부를 인코딩한 글자) 수십 줄을 화면에 다 찍지 않고 `25 lines sent` 한 줄로 줄여 표시 |
| 줄 끝의 `\` | 명령이 길어서 다음 줄에 이어 쓴다는 표시. 두 줄을 **함께** 복사해 붙여 넣음 |

`apt install` 은 설치 과정을 출력합니다. 패키지 크기와 디스크 용량 줄은 환경마다 달라서 `...` 로 줄였습니다. swaks가 이미 설치되어 있으면 `swaks is already the newest version` 이 나옵니다.

```
Installing:
  swaks

Suggested packages:
  libauthen-ntlm-perl  perl-doc

Recommended packages:
  libio-socket-inet6-perl

Summary:
  Upgrading: 0, Installing: 1, Removing: 0, Not Upgrading: ...
...

Selecting previously unselected package swaks.
Preparing to unpack …/swaks_20240103.0-2_all.deb…
Unpacking swaks (20240103.0-2)…
Setting up swaks (20240103.0-2)…
```

> `Unable to locate package swaks` 나 `404 Not Found` 가 나오면 패키지 목록이 오래된 것입니다. `sudo apt update` 를 먼저 실행한 뒤 다시 설치합니다.
{: .prompt-warning }

`cd` 와 `echo` 는 아무것도 출력하지 않습니다. `swaks` 의 출력은 다음과 같습니다.

```
=== Trying 192.168.0.30:25...
=== Connected to 192.168.0.30.
<-  220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
 -> EHLO kali
<-  250-ubuntu.lab.local
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> MAIL FROM:<kali@kali.lab.local>
<-  250 2.1.0 Ok
 -> RCPT TO:<mailuser@ubuntu.lab.local>
<-  250 2.1.5 Ok
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> 25 lines sent
<-  250 2.0.0 Ok: queued as 56D9D7AB
 -> QUIT
<-  221 2.0.0 Bye
=== Connection closed with remote host.
```

| 표시 | 의미 |
|---|---|
| `===` | swaks 자신의 안내 메시지 |
| ` ->` | Kali(swaks)가 **보낸** 명령 |
| `<-` | 서버의 **정상 응답** |

> ` -> EHLO kali` 의 `kali` 자리에는 Kali의 호스트 이름이 나옵니다. 큐 번호(`56D9D7AB`)는 실행할 때마다 다릅니다.
{: .prompt-info }

마지막에 **`250 2.0.0 Ok: queued as ...`** 가 나왔습니다. 실행파일이 붙은 메일을 서버가 **아무 검사 없이 받아 준 것**입니다.

### [실습 2] (Ubuntu) 실행파일이 메일함까지 들어왔는지 확인

```bash
# === Ubuntu ===
sudo grep 'name="invoice.exe"' /var/mail/mailuser
```

```
Content-Type: application/octet-stream; name="invoice.exe"
Content-Disposition: attachment; filename="invoice.exe"
```

| 명령·옵션 | 설명 |
|---|---|
| `grep '찾을글자' 파일` | 파일에서 찾을 글자가 들어 있는 줄만 출력 |
| `'name="invoice.exe"'` | 큰따옴표까지 찾을 글자에 포함하려고 바깥을 작은따옴표(`'`)로 감쌈 |

| 화면의 줄 | 의미 |
|---|---|
| `Content-Type: application/octet-stream; name="invoice.exe"` | 첨부파일의 형식과 **파일 이름** |
| `Content-Disposition: attachment; filename="invoice.exe"` | 이 부분이 **첨부파일**이라는 표시 |

`invoice.exe` 가 **사용자 메일함까지 들어왔습니다.** 사용자가 메일 프로그램에서 이 첨부를 열면 악성코드가 실행될 수 있는 상태입니다. 이 두 줄이 다음 실습에서 차단 규칙이 검사할 **첨부파일 헤더**입니다.

### [실습 3] (Ubuntu) 실행파일 첨부 차단 규칙 만들기

Postfix의 **`mime_header_checks`** 는 첨부파일의 헤더(실습 2에서 본 줄)를 검사하는 기능입니다. 첨부파일 이름이 `.exe`, `.bat`, `.scr`, `.js`, `.vbs` 로 끝나면 거부하는 규칙을 만듭니다.

```bash
# === Ubuntu ===
sudo tee /etc/postfix/mime_header_checks > /dev/null <<'EOF'
# block dangerous attachment extensions
/name="?.*\.(exe|bat|scr|js|vbs)"?$/   REJECT Attachment type not allowed
EOF

sudo postconf -e 'mime_header_checks = regexp:/etc/postfix/mime_header_checks'
sudo systemctl reload postfix
postconf mime_header_checks
```

> 첫 줄 `sudo tee ...` 부터 `EOF` 줄까지는 **하나의 명령**입니다. 블록 전체를 한 번에 복사해 붙여 넣습니다. `EOF` 줄 앞뒤에 공백이 있으면 끝을 알아보지 못해 `>` 표시만 계속 나오므로, 그때는 `Ctrl+C` 로 빠져나와 다시 붙여 넣습니다.
{: .prompt-warning }

`tee`, `postconf -e`, `systemctl reload` 는 아무것도 출력하지 않습니다. 마지막 조회 명령의 결과만 나옵니다.

```
mime_header_checks = regexp:/etc/postfix/mime_header_checks
```

| 명령·옵션 | 설명 |
|---|---|
| `tee 파일 > /dev/null <<'EOF' ... EOF` | `EOF` 두 줄 사이의 내용을 파일로 저장. `sudo tee` 를 써야 관리자만 쓸 수 있는 `/etc` 아래에 저장 가능. `> /dev/null` 은 같은 내용이 화면에 한 번 더 나오지 않게 버림 |
| `postconf -e 'mime_header_checks = regexp:파일'` | 첨부파일 헤더 검사 규칙으로 이 파일을 사용. `regexp:` 는 파일 안의 규칙이 **정규표현식**(글자 패턴)이라는 뜻 |
| `systemctl reload postfix` | 설정을 다시 읽어 규칙 적용 |

파일에 저장한 두 줄의 뜻은 다음과 같습니다.

| 부분 | 의미 |
|---|---|
| `# block ...` (첫 줄) | 설명(주석). `#` 으로 시작하는 줄은 Postfix가 읽지 않음 |
| `/ ... /` | 규칙(패턴)의 시작과 끝 |
| `name=` | 실습 2에서 본 첨부파일 헤더의 **파일 이름** 부분 |
| `"?` | 큰따옴표(`"`)가 있어도 되고 없어도 됨. `?` 는 바로 앞 글자가 **없거나 하나**라는 뜻 |
| `.*` | 아무 글자나 몇 개든(파일 이름의 앞부분, 예: `invoice`) |
| `\.` | 진짜 점(`.`). `\` 없이 `.` 만 쓰면 "아무 글자 하나"라는 뜻이 됨 |
| <code>(exe&#124;bat&#124;scr&#124;js&#124;vbs)</code> | 괄호 안의 확장자 가운데 하나. <code>&#124;</code> 는 "또는" |
| `$` | 줄의 끝. 확장자가 이름의 **맨 끝**에 있을 때만 해당 |
| `REJECT Attachment type not allowed` | 패턴에 맞으면 메일을 **거부**하고 이 문구를 돌려줌 |

> Postfix의 정규표현식 규칙은 **대소문자를 구별하지 않습니다.** `INVOICE.EXE` 처럼 대문자로 바꿔도 걸립니다.
{: .prompt-info }

### [실습 4] (Kali) 같은 첨부를 다시 보내기 — 실행파일 거부

실습 1과 같은 명령으로 `.exe` 를 다시 보냅니다.

```bash
# === Kali ===
cd ~
swaks --server 192.168.0.30 --from kali@kali.lab.local --to mailuser@ubuntu.lab.local \
  --header "Subject: Invoice" --body "See attached invoice." --attach @invoice.exe --suppress-data
```

`cd` 는 출력이 없고, `swaks` 의 출력은 다음과 같습니다.

```
=== Trying 192.168.0.30:25...
=== Connected to 192.168.0.30.
<-  220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
 -> EHLO kali
<-  250-ubuntu.lab.local
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> MAIL FROM:<kali@kali.lab.local>
<-  250 2.1.0 Ok
 -> RCPT TO:<mailuser@ubuntu.lab.local>
<-  250 2.1.5 Ok
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> 25 lines sent
<** 550 5.7.1 Attachment type not allowed
 -> QUIT
<-  221 2.0.0 Bye
=== Connection closed with remote host.
```

| 표시 | 의미 |
|---|---|
| `<**` | 서버의 **오류 응답**. 실습 1의 `<-  250 ... queued` 자리에 `<** 550` 이 나옴 |

규칙이 정상 파일까지 막지는 않는지 `.txt` 파일을 붙여 보냅니다.

```bash
# === Kali ===
cd ~
swaks --server 192.168.0.30 --from kali@kali.lab.local --to mailuser@ubuntu.lab.local \
  --header "Subject: Note" --body "See attached note." --attach @note.txt --suppress-data
```

`cd` 는 출력이 없고, `swaks` 의 출력은 다음과 같습니다.

```
=== Trying 192.168.0.30:25...
=== Connected to 192.168.0.30.
<-  220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
 -> EHLO kali
<-  250-ubuntu.lab.local
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> MAIL FROM:<kali@kali.lab.local>
<-  250 2.1.0 Ok
 -> RCPT TO:<mailuser@ubuntu.lab.local>
<-  250 2.1.5 Ok
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> 25 lines sent
<-  250 2.0.0 Ok: queued as CE9C57AB
 -> QUIT
<-  221 2.0.0 Bye
=== Connection closed with remote host.
```

| 첨부파일 | 규칙 없음 (실습 1) | 규칙 적용 (실습 4) |
|---|---|---|
| `invoice.exe` | `250` 접수 → 메일함 도착 | **`550` 거부** |
| `note.txt` | — | **`250` 접수** — 정상 전달 |

### [실습 5] (Ubuntu) 메일 로그에서 차단 기록 확인

```bash
# === Ubuntu ===
sudo grep -E 'reject:|status=sent' /var/log/mail.log | tail -2
```

| 명령·옵션 | 설명 |
|---|---|
| `/var/log/mail.log` | Postfix가 메일 처리 기록을 남기는 로그 파일 |
| <code>grep -E 'reject:&#124;status=sent'</code> | `reject:`(거부) **또는** `status=sent`(전달 완료)가 들어 있는 줄만 골라냄. `-E` 는 <code>&#124;</code>(또는) 같은 확장 패턴을 쓰겠다는 옵션 |
| <code>&#124; tail -2</code> | 골라낸 줄 가운데 **마지막 2줄**(가장 최근 기록)만 표시 |

```
2026-10-19T22:47:34.749420+09:00 ubuntu postfix/cleanup[907]: B6C217AB: reject: header Content-Type: application/octet-stream; name="invoice.exe" from unknown[192.168.0.10]; from=<kali@kali.lab.local> to=<mailuser@ubuntu.lab.local> proto=ESMTP helo=<kali>: 5.7.1 Attachment type not allowed
2026-10-19T22:47:42.853472+09:00 ubuntu postfix/local[924]: CE9C57AB: to=<mailuser@ubuntu.lab.local>, relay=local, delay=0.01, delays=0/0/0/0, dsn=2.0.0, status=sent (delivered to mailbox)
```

- 첫 줄: `name="invoice.exe"` 첨부가 **`reject`** 되었고, 보낸 곳 IP(`192.168.0.10`)와 주소가 기록되었습니다. 실습 2에서 본 `Content-Type` 줄이 규칙에 걸린 것입니다.
- 둘째 줄: `.txt` 첨부 메일은 **`status=sent (delivered to mailbox)`**, 메일함에 전달되었습니다.

### 정리

| 상태 | `mime_header_checks` | `invoice.exe` 첨부 메일 |
|---|---|---|
| **취약** (실습 1·2) | 없음 (Postfix 기본값) | `250` 접수, 사용자 메일함에 도착 |
| **안전** (실습 3~5) | 실행파일 확장자 차단 규칙 | `550` 거부, 로그에 `reject` 기록 |

> - Postfix는 기본적으로 **첨부파일을 검사하지 않습니다.** 관리자가 규칙을 넣지 않으면 실행파일도 그대로 사용자에게 갑니다.
> - 악성 첨부·트로이 목마는 정상 파일처럼 보이는 첨부로 **사용자**를 노립니다. 서버 앞단에서 **위험한 확장자를 미리 막는 것**이 1차 대책입니다.
> - 확장자 검사는 파일 **이름**만 봅니다. 목록에 없는 확장자, 이름을 바꾸거나 압축한 악성 파일은 통과할 수 있으므로 **백신 검사**를 함께 씁니다.
> - 스팸 필터 솔루션은 헤더·제목뿐 아니라 **본문**까지 검사하고, 내부에서 **나가는** 메일의 정보 유출도 막습니다.
{: .prompt-tip }

---

## 4.2 SPF·DKIM·DMARC — 발신 도메인 확인

### [이론]

3.2절의 위조 메일은 SMTP가 보내는 사람을 확인하지 않아서 생겼습니다. **SPF(Sender Policy Framework)** 는 이를 막기 위해 **도메인 주인이 "우리 메일은 이 IP에서만 나간다"는 목록을 DNS에 공개**하는 기술입니다.

| 단계 | 하는 일 |
|---|---|
| ① 등록 | 도메인 관리자가 **자기 메일 발송 서버의 IP**를 DNS에 **TXT 레코드**로 등록 |
| ② 수신 | 받는 메일 서버가 메일의 발신 도메인(봉투의 `MAIL FROM` 주소에서 `@` 뒤)을 확인 |
| ③ 조회 | 그 도메인의 SPF 레코드를 DNS에서 조회 |
| ④ 비교 | 메일을 실제로 보내온 서버의 IP(`Received` 에 남는 IP)가 목록에 있는지 비교 |
| ⑤ 판정 | 목록에 없으면 **위조로 판단**해 거부하거나 스팸으로 분류 |

SPF는 **허용된 도메인과 IP에서 온 메일인지 확인해 스팸을 차단하는 기술**입니다. 웹 보안이나 VoIP와는 관계가 없습니다.

| 기술 | 확인하는 것 | DNS에 등록하는 것 |
|---|---|---|
| **SPF** | 보낸 서버의 **IP**가 허용 목록에 있는가 | 허용 IP 목록 (TXT) |
| **DKIM** | 메일에 붙은 **전자서명**이 도메인의 공개키로 맞는가 | 서명 확인용 공개키 (TXT) |
| **DMARC** | SPF·DKIM 결과가 화면에 보이는 `From` 도메인과 맞지 않는 메일을 **어떻게 처리할지** | 처리 정책 `none`/`quarantine`/`reject` (TXT) |

### [실습] (Kali) 실제 도메인의 SPF·DMARC 조회

조회하기 전에 세 기술의 이름과, 이 정보가 들어 있는 **DNS 레코드**를 정리합니다.

| 약어 | 풀이 | 우리말 뜻 |
|---|---|---|
| **SPF** | **S**ender **P**olicy **F**ramework | 발신자 정책 프레임워크 — 이 도메인의 메일을 보낼 수 있는 서버 목록 |
| **DKIM** | **D**omain**K**eys **I**dentified **M**ail | 도메인 키로 식별되는 메일 — 전자서명으로 보낸 도메인과 내용의 변조 여부 확인 |
| **DMARC** | **D**omain-based **M**essage **A**uthentication, **R**eporting and **C**onformance | 도메인 기반 메시지 인증·보고·준수 — SPF·DKIM 결과로 처리 정책을 정하고 보고를 받음 |

**DNS** 는 `naver.com` 같은 이름에 대한 정보를 알려 주는 인터넷의 전화번호부입니다. 정보는 **레코드(record)** 라는 단위로 저장되며, 종류마다 담는 내용이 다릅니다.

| 레코드 | 담는 내용 | 예 |
|---|---|---|
| **A** | 도메인의 **IPv4 주소** | `www.example.com` → `192.0.2.10` |
| **AAAA** | 도메인의 **IPv6 주소** | `www.example.com` → `2001:db8::10` |
| **MX** | 이 도메인의 **메일을 받는 서버** (Mail eXchanger) | `naver.com` → `mx1.mail.naver.com` |
| **TXT** | 아무 **글자(text)** 나 적을 수 있는 칸. 메일 보안 정책을 여기에 적음 | `v=spf1 ...`, `v=DMARC1; ...` |
| **CNAME** | 다른 도메인 이름의 **별명** | `www.example.com` → `example.com` |

SPF·DKIM·DMARC는 전용 레코드가 따로 있지 않고 모두 **TXT 레코드**에 정해진 형식으로 적습니다. 그래서 어느 기술의 값인지는 **글자의 시작 부분**과 **조회하는 이름**으로 구분합니다.

| 기술 | 조회하는 이름 | TXT 값의 시작 |
|---|---|---|
| SPF | 도메인 그대로 (`naver.com`) | `v=spf1` |
| DKIM | `선택자._domainkey.도메인` | `v=DKIM1` |
| DMARC | `_dmarc.도메인` (`_dmarc.naver.com`) | `v=DMARC1` |

> DKIM은 앞의 `선택자`(selector) 이름을 메일 헤더를 보고서야 알 수 있어서 이 실습에서는 조회하지 않습니다.
{: .prompt-info }

```bash
# === Kali ===
dig TXT naver.com +short | grep spf
dig TXT google.com +short | grep spf
dig TXT _spf.google.com +short
```

| 명령·옵션                        | 설명                                                  |
| ---------------------------- | --------------------------------------------------- |
| `dig`                        | DNS 서버에 레코드를 물어보는 조회 도구                             |
| `TXT`                        | 조회할 레코드 종류. SPF는 **TXT 레코드**에 들어 있음                 |
| `naver.com`                  | 조회할 도메인                                             |
| `+short`                     | 설명 없이 **결과 값만** 짧게 출력                               |
| <code>&#124; grep spf</code> | 한 도메인에 TXT 레코드가 여러 개 있을 수 있으므로 `spf` 가 들어 있는 줄만 골라냄 |

```
"v=spf1 ip4:111.91.135.0/27 ip4:125.209.208.0/20 ip4:125.209.224.0/19 ip4:210.89.163.112 ip4:210.89.173.104/29 ip4:114.111.35.0/24 ip4:61.247.196.0/24 ip4:61.247.197.0/24 ~all"
"v=spf1 include:_spf.google.com ~all"
"v=spf1 ip4:74.125.0.0/16 ip4:209.85.128.0/17 ip6:2001:4860:4864::/56 ip6:2404:6800:4864::/56 ip6:2607:f8b0:4864::/56 ip6:2800:3f0:4864::/56 ip6:2a00:1450:4864::/56 ip6:2c0f:fb50:4864::/56 ~all"
```

| 부분 | 의미 |
|---|---|
| `v=spf1` | SPF 버전 1 레코드 |
| `ip4:125.209.208.0/20` | 이 IP 대역의 서버는 이 도메인 메일을 **보낼 수 있음** |
| `include:_spf.google.com` | 허용 목록을 **다른 레코드에서 가져옴** (구글은 목록을 `_spf.google.com` 에 둠) |
| `~all` | 목록에 없는 곳에서 온 메일은 **의심(softfail)** — 스팸으로 분류 |
| `-all` | 목록에 없는 곳에서 온 메일은 **거부(fail)** |

DMARC 정책과 메일 수신 서버(MX)도 조회합니다.

```bash
# === Kali ===
dig TXT _dmarc.google.com +short
dig TXT _dmarc.naver.com +short
dig MX naver.com +short
```

- `_dmarc.도메인` → DMARC 정책은 도메인 앞에 `_dmarc.` 를 붙인 이름의 TXT 레코드에 들어 있습니다.
- `MX` → 그 도메인의 메일을 받는 서버를 알려 주는 레코드 종류입니다.

```
"v=DMARC1; p=reject; rua=mailto:mailauth-reports@google.com"
"v=DMARC1; p=none; rua=mailto:dmarc_reports@worksmobile.com"
20 mx4.mail.naver.com.
20 mx5.mail.naver.com.
20 mx6.mail.naver.com.
10 mx1.mail.naver.com.
```

> `MX` 결과 네 줄은 조회할 때마다 **순서가 바뀌어** 나올 수 있습니다. 순서와 관계없이 같은 네 줄이 나오면 정상입니다. 레코드 값은 각 회사가 설정을 바꾸면 달라질 수 있습니다.
{: .prompt-info }

| 결과 | 의미 |
|---|---|
| google.com `p=reject` | SPF·DKIM 검사에 실패한 메일은 **거부**하라 |
| naver.com `p=none` | 실패해도 처리는 하지 않고 `rua` 주소로 **보고만** 받음 |
| `MX` 레코드 | 이 도메인의 **메일을 받는 서버**. 숫자가 작을수록 먼저 사용 |

### 정리

| 장비 | 한 일 | 결과 |
|---|---|---|
| Kali | `dig TXT` 로 naver.com·google.com의 SPF 조회 | 메일을 보낼 수 있는 IP 목록과 `~all` 확인 |
| Kali | `dig TXT _dmarc.도메인`, `dig MX` 조회 | google `p=reject`, naver `p=none`, 메일 수신 서버 목록 확인 |

> - 3.2절의 `admin@bank.com` 위조 메일이 SPF를 확인하는 서버에 도착했다고 합시다. `bank.com` 이 SPF 레코드를 공개해 두었다면 그 목록에 `192.168.0.10` 이 없으므로 **위조(fail·softfail)** 로 판정됩니다.
> - 반대로 도메인 주인이 SPF 레코드를 **등록하지 않으면** 받는 서버는 판정할 근거가 없습니다. SPF·DMARC는 **보내는 쪽 도메인이 등록해야** 효과가 있는 대책입니다.
> - SPF는 보낸 서버의 **IP**, DKIM은 메일의 **전자서명**, DMARC는 두 검사에 실패했을 때의 **처리 정책**을 담당합니다.
{: .prompt-tip }

---

## 4.3 전송 구간 암호화 — STARTTLS

### [이론]

PGP·S/MIME(03-1)은 **메일 내용 자체**를 암호화합니다. 이와 별도로 **메일 서버 사이, 사용자와 서버 사이의 통신 구간**도 암호화해야 SMTP 명령과 로그인 정보가 노출되지 않습니다.

| 방식                    | 내용                                                    |
| --------------------- | ----------------------------------------------------- |
| **STARTTLS**          | 평문으로 접속한 뒤 `STARTTLS` 명령으로 **TLS 암호화로 전환** (25, 587번) |
| SMTPS / POP3S / IMAPS | 처음부터 TLS로 접속 (465 / 995 / 993번)                       |

이 절은 **평문 메일을 받아 주는 기본 상태를 확인한 뒤, 암호화를 필수로 바꿔 평문을 막는** 순서로 진행합니다. 4.1절에서 설치한 **swaks** 를 씁니다.

| 순서 | 장비 | 할 일 | 상태 |
|---|---|---|---|
| 실습 1 | Kali | 암호화 없이 메일 보내기 | **취약** — 평문 메일도 접수됨 |
| 실습 2 | Ubuntu | 암호화를 필수로 설정 | **안전**하게 바꿈 |
| 실습 3 | Kali | 평문으로, 그리고 STARTTLS로 다시 보내기 | 평문은 거부, 암호화는 접수 |
| 실습 4 | Ubuntu | 설정을 원래대로 되돌리기 | 다른 실습을 위해 복구 |

### [실습 1] (Kali) 암호화 없이 메일 보내기 — 평문 접수

```bash
# === Kali ===
swaks --server 192.168.0.30 --from kali@kali.lab.local --to mailuser@ubuntu.lab.local \
  --header "Subject: Plain" --body "This mail is not encrypted." --suppress-data
```

옵션은 4.1절과 같고, 첨부파일만 없습니다.

```
=== Trying 192.168.0.30:25...
=== Connected to 192.168.0.30.
<-  220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
 -> EHLO kali
<-  250-ubuntu.lab.local
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> MAIL FROM:<kali@kali.lab.local>
<-  250 2.1.0 Ok
 -> RCPT TO:<mailuser@ubuntu.lab.local>
<-  250 2.1.5 Ok
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> 11 lines sent
<-  250 2.0.0 Ok: queued as 1D0317AB
 -> QUIT
<-  221 2.0.0 Bye
=== Connection closed with remote host.
```

| 화면 | 의미 |
|---|---|
| `<-  250-STARTTLS` | 서버는 TLS 암호화로 전환하는 기능을 **제안함** |
| ` -> MAIL FROM:...` 이 바로 이어짐 | 클라이언트가 `STARTTLS` 를 보내지 않고 **평문으로 계속 진행** |
| `250 2.0.0 Ok: queued as ...` | 서버가 **평문 메일을 그대로 받아 줌** |

03-1 1.4절에서 Wireshark로 본 것처럼, 이 메일은 경로 위의 누구나 읽을 수 있습니다. 서버가 STARTTLS를 지원해도 **쓸지 말지를 클라이언트에게 맡기기 때문**입니다.

### [실습 2] (Ubuntu) 암호화를 필수로 설정

먼저 현재 설정을 확인합니다.

```bash
# === Ubuntu ===
postconf smtpd_tls_security_level
```

```
smtpd_tls_security_level = may
```

| 값 | 의미 |
|---|---|
| `may` (Ubuntu 기본값) | 암호화를 **제안만** 함. 클라이언트가 원하지 않으면 평문으로 받음 |
| `encrypt` | 암호화를 **필수**로 함. `STARTTLS` 를 하지 않으면 메일을 받지 않음 |

값을 `encrypt` 로 바꿉니다.

```bash
# === Ubuntu ===
sudo postconf -e 'smtpd_tls_security_level = encrypt'
sudo systemctl reload postfix
postconf smtpd_tls_security_level
```

`postconf -e` 와 `systemctl reload` 는 아무것도 출력하지 않습니다. 마지막 조회 결과입니다.

```
smtpd_tls_security_level = encrypt
```

### [실습 3] (Kali) 평문으로, 그리고 STARTTLS로 다시 보내기

먼저 실습 1과 **똑같은 명령**(암호화 없음)으로 보냅니다.

```bash
# === Kali ===
swaks --server 192.168.0.30 --from kali@kali.lab.local --to mailuser@ubuntu.lab.local \
  --header "Subject: Plain" --body "This mail is not encrypted." --suppress-data
```

```
=== Trying 192.168.0.30:25...
=== Connected to 192.168.0.30.
<-  220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
 -> EHLO kali
<-  250-ubuntu.lab.local
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> MAIL FROM:<kali@kali.lab.local>
<** 530 5.7.0 Must issue a STARTTLS command first
 -> QUIT
<-  221 2.0.0 Bye
=== Connection closed with remote host.
```

`MAIL FROM` 단계에서 **`530 5.7.0 Must issue a STARTTLS command first`**("먼저 STARTTLS 명령을 보내라")로 거부되었습니다. 평문으로는 메일을 보낼 수 없습니다.

이번에는 끝에 **`--tls`** 를 붙여 STARTTLS로 암호화한 뒤 보냅니다.

```bash
# === Kali ===
swaks --server 192.168.0.30 --from kali@kali.lab.local --to mailuser@ubuntu.lab.local \
  --header "Subject: TLS" --body "This mail is encrypted." --suppress-data --tls
```

| 명령·옵션 | 설명 |
|---|---|
| `--tls` | `EHLO` 다음에 **`STARTTLS` 명령을 보내 암호화를 시작**한 뒤 메일을 보냄 |

```
=== Trying 192.168.0.30:25...
=== Connected to 192.168.0.30.
<-  220 ubuntu.lab.local ESMTP Postfix (Ubuntu)
 -> EHLO kali
<-  250-ubuntu.lab.local
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> STARTTLS
<-  220 2.0.0 Ready to start TLS
=== TLS started with cipher TLSv1.3:TLS_AES_256_GCM_SHA384:256
=== TLS client certificate not requested and not sent
=== TLS no client certificate set
=== TLS peer[0]   subject=[/CN=ubuntu]
===               commonName=[ubuntu], subjectAltName=[DNS:ubuntu] notAfter=[2036-10-15T07:08:28Z]
=== TLS peer certificate failed CA verification (self-signed certificate), failed host verification (using host 192.168.0.30 to verify)
 ~> EHLO kali
<~  250-ubuntu.lab.local
<~  250-PIPELINING
<~  250-SIZE 10240000
<~  250-ETRN
<~  250-ENHANCEDSTATUSCODES
<~  250-8BITMIME
<~  250-DSN
<~  250-SMTPUTF8
<~  250 CHUNKING
 ~> MAIL FROM:<kali@kali.lab.local>
<~  250 2.1.0 Ok
 ~> RCPT TO:<mailuser@ubuntu.lab.local>
<~  250 2.1.5 Ok
 ~> DATA
<~  354 End data with <CR><LF>.<CR><LF>
 ~> 11 lines sent
<~  250 2.0.0 Ok: queued as 2C5607AB
 ~> QUIT
<~  221 2.0.0 Bye
=== Connection closed with remote host.
```

> `CN=ubuntu` 자리에는 각자의 Ubuntu 호스트 이름이, `notAfter=` 에는 Postfix를 설치한 날로부터 약 10년 뒤 날짜가 나옵니다. 큐 번호도 실행할 때마다 다릅니다.
{: .prompt-info }

| 화면 | 의미 |
|---|---|
| ` -> STARTTLS` / `<-  220 2.0.0 Ready to start TLS` | 클라이언트가 암호화를 요청하고 서버가 수락 |
| `TLS started with cipher TLSv1.3:...` | **TLS 1.3** 으로 암호화 시작 |
| `failed CA verification (self-signed certificate)` | 서버 인증서가 **자체 서명** 인증서라 신뢰할 수 있는 기관이 보증하지 않음. 실습 환경에서는 정상이며 암호화는 됨 |
| `~>` / `<~` | 물결(`~`) 표시는 **암호화된 구간**에서 보내고 받은 내용 |
| `250 2.0.0 Ok: queued as ...` | 암호화한 메일은 정상 접수 |

STARTTLS 뒤에는 `EHLO` 부터 `QUIT` 까지 모든 명령과 메일 내용이 **암호화되어** 오갑니다. 같은 메일을 Wireshark로 캡처하면 `STARTTLS` 이후는 TLS 패킷으로만 보이고 내용은 읽을 수 없습니다.

### [실습 4] (Ubuntu) 설정을 원래대로 되돌리기

```bash
# === Ubuntu ===
sudo postconf -e 'smtpd_tls_security_level = may'
sudo systemctl reload postfix
postconf smtpd_tls_security_level
```

`postconf -e` 와 `systemctl reload` 는 아무것도 출력하지 않습니다. 마지막 조회 결과입니다.

```
smtpd_tls_security_level = may
```

> 25번 포트는 **다른 메일 서버가 메일을 넘겨주는** 포트입니다. 인터넷에는 아직 TLS를 쓰지 못하는 메일 서버도 있어서, 25번에서 `encrypt` 로 강제하면 그런 서버가 보내는 정상 메일까지 받지 못합니다. 그래서 실무에서는 25번은 `may` 로 두고, **사용자가 로그인해 메일을 보내는 587번(Submission)** 에서 암호화를 필수로 합니다.
{: .prompt-warning }

> Ubuntu의 Postfix는 설치할 때 자체 서명 인증서(`ssl-cert-snakeoil`)를 사용하도록 설정되어 있어서 STARTTLS가 처음부터 켜져 있습니다. 실제 운영 서버에서는 공인 인증기관이 발급한 인증서로 바꾸어야 상대 서버와 사용자가 인증서를 신뢰할 수 있습니다.
{: .prompt-info }

### 정리

| 상태 | `smtpd_tls_security_level` | 암호화 없이 보낸 메일 | `--tls`(STARTTLS)로 보낸 메일 |
|---|---|---|---|
| **취약** (실습 1) | `may` (기본값) | `250` 접수 — 평문으로 오감 | 접수 |
| **안전** (실습 2·3) | `encrypt` | **`530` 거부** | `250` 접수 — 암호화되어 오감 |

> - 화면에 `250-STARTTLS` 가 보인다고 **안전한 것은 아닙니다.** 암호화를 쓸 수 있다는 뜻일 뿐, 클라이언트가 요청하지 않으면 평문으로 오갑니다.
> - 암호화를 **필수**로 해야 평문 전송을 막을 수 있습니다. 실무에서는 사용자가 메일을 보내는 587번에 적용합니다.
> - STARTTLS·SMTPS는 **회선 구간**을, PGP·S/MIME은 **메일 내용**을 보호합니다. 서로 대신할 수 없어 함께 씁니다.
{: .prompt-tip }

---

## 4.4 이메일 보안 대책 종합

| 번호 | 위협 | 대책 | 이 글에서 확인한 방법 |
|---|---|---|---|
| 1 | 발신자 위조·피싱 | SPF·DKIM·DMARC, `Received` 헤더 확인 | 3.2, 4.2 |
| 2 | 스팸 릴레이 | 허용 네트워크·인증 사용자만 릴레이 | 3.3 |
| 3 | 계정 정보 수집 | `VRFY`·`EXPN` 비활성화 | 3.4 |
| 4 | Active Contents | 메일 프로그램의 스크립트 기능 끄기 | 3.5 |
| 5 | 악성 첨부·트로이 목마 | 첨부파일 확장자 필터링, 백신 검사 | 4.1 |
| 6 | 스팸·메일 폭탄·DoS | 메일 서버 앞단의 스팸 필터 솔루션 | 4.1 |
| 7 | 버퍼 오버플로우 | 메일 서버 최신 버전·패치 | 3.1 |
| 8 | 내용 도청·변조 | PGP·S/MIME, STARTTLS | 03-1, 4.3 |

---


---

# 체크리스트

- [ ] 클라이언트 대상 공격(Active Contents, 트로이 목마)과 서버 대상 공격(버퍼 오버플로우, DoS)을 구분
- [ ] (Kali) `admin@bank.com` 위조 메일이 `250` 으로 접수되는지 확인
- [ ] (Ubuntu) 위조 메일의 `Received` 에서 실제 IP `192.168.0.10` 확인
- [ ] (Ubuntu·Kali) `mynetworks` 를 넓히면 외부 도메인 릴레이가 `250` 으로 허용되고, 되돌리면 `Relay access denied` 로 막힘을 확인
- [ ] (Ubuntu) `mynetworks` 를 기본값(서버 자신만)으로 되돌렸는지 확인
- [ ] (Ubuntu·Kali) `disable_vrfy_command` 를 `no` → `yes` 로 바꾸면 `252`/`550` 이 `502 ... disabled` 로 바뀜을 확인
- [ ] (Ubuntu·Kali) 규칙이 없을 때 `.exe` 첨부가 메일함까지 들어오고, 규칙 적용 후 `550` 거부·`.txt` 는 `250` 접수, 로그의 `reject` 확인
- [ ] (Kali) `dig TXT` 로 SPF·DMARC 레코드를 조회하고 `~all`, `-all`, `p=reject` 를 설명
- [ ] (Ubuntu·Kali) `smtpd_tls_security_level` 을 `encrypt` 로 바꾸면 평문 메일은 `530`, `--tls` 메일은 `250` 임을 확인하고 `may` 로 되돌림
