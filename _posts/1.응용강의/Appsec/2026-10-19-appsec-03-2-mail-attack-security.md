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

---

# 3. 이메일 공격 유형

## 3.1 공격 유형 한눈에 보기

### [이론]

메일 공격은 **누구를 노리느냐**에 따라 두 갈래로 나눕니다. 어느 쪽에 속하는지 구분하는 것이 중요합니다.

| 대상 | 공격 | 원리 | 대책 |
|---|---|---|---|
| **메일 클라이언트**(사용자) | **Active Contents 공격** | 메일을 열 때 HTML을 해석하는 메일 프로그램·웹 브라우저에서 **스크립트가 실행**되어 정보 유출 | 메일 프로그램의 **스크립트 기능을 끔** |
| | **트로이 목마·악성 첨부** | 정상 파일처럼 보이는 첨부파일을 열면 악성코드 실행 | 첨부파일 필터링, 백신 검사 |
| | 피싱 | 위조한 발신자로 속여 비밀번호·금융정보 입력을 유도 | SPF·DMARC, 사용자 교육 |
| **메일 서버** | **버퍼 오버플로우 공격** | Sendmail 같은 **메일 서버 프로그램의 취약점**을 이용해 서버 권한 획득 | 최신 버전·패치 |
| | **서비스 거부(DoS) 공격** | 대량 메일(메일 폭탄)·대량 접속으로 서버를 마비 | 스팸 필터, 접속·크기 제한 |
| | 스팸 릴레이 악용 | 외부인이 서버를 **스팸 중계소**로 사용 | 릴레이 제한 |
| | 계정 정보 수집 | `VRFY`·`EXPN` 명령으로 있는 계정을 알아냄 | 명령 비활성화 |

> **Active Contents 공격과 트로이 목마 공격은 클라이언트**를 노리는 공격이고, **버퍼 오버플로우와 서비스 거부 공격은 서버**를 노리는 공격입니다. 바이러스 메일은 메일 서버를 공격하는 것이 아니라 **사용자**를 노립니다.
{: .prompt-warning }

---

## 3.2 발신자 위조(스푸핑)

### [이론]

03-1에서 본 것처럼 SMTP는 **보내는 사람을 확인하지 않습니다.** `MAIL FROM` 과 `From:` 에 어떤 주소를 적어도 그대로 받아 줍니다. 공격자는 이를 이용해 은행·관리자·거래처로 **위장한 메일**을 보내고, 받은 사람이 링크를 누르거나 비밀번호를 입력하도록 속입니다.

다만 `Received` 헤더에는 서버가 직접 확인한 **실제 접속 IP**가 남습니다. 위조 메일을 가려낼 때 `Received` 를 보는 이유입니다.

### [실습] (Kali) 은행 관리자로 위장한 메일 보내기

```bash
# === Kali ===
nc -C 192.168.0.30 25
```

`nc -C` 는 03-1에서 쓴 것과 같은 명령입니다. Kali에서 Ubuntu의 25번 포트로 TCP 연결을 열고, 입력한 SMTP 명령을 그대로 보냅니다. `-C` 는 줄 끝을 SMTP 규칙(CR+LF)에 맞추는 옵션입니다.

아래를 한 줄씩 입력합니다. 실제로는 존재하지 않는 `bank.com` 관리자를 사칭합니다.

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

### [확인] (Ubuntu) 받은 메일에서 위조 흔적 찾기

```bash
# === Ubuntu ===
sudo tail -15 /var/mail/mailuser
```

> `tail -15` 는 파일의 **마지막 15줄**만 보여 주는 명령입니다. 메일함 파일에는 메일이 도착한 순서대로 쌓이므로, 끝부분을 보면 방금 받은 메일만 확인할 수 있습니다.
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

📷 **캡처**: `From: Bank Admin <admin@bank.com>` 과 `Received: ... [192.168.0.10]` 이 함께 보이는 화면

| 헤더 | 값 | 판단 |
|---|---|---|
| `From`, `Return-Path` | `admin@bank.com` | 공격자가 **적은 대로** 보임 → 믿을 수 없음 |
| `Received` 의 `from bank.com` | `EHLO` 로 공격자가 적은 이름 | 믿을 수 없음 |
| `Received` 의 `unknown [192.168.0.10]` | 서버가 **직접 확인한 IP** | **Kali에서 왔다는 증거** |

> 메일 프로그램은 보통 `From` 만 보여 주므로 사용자는 진짜 은행 메일로 착각합니다. 이런 위조를 **수신 서버가 자동으로 가려내게** 하는 기술이 4.2절의 SPF입니다.
{: .prompt-info }

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

### [실습] 외부 도메인으로 릴레이가 막히는지 확인

먼저 Ubuntu에서 릴레이를 허용하는 네트워크를 봅니다.

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
| `mynetworks = 127.0.0.0/8 ...` | 릴레이를 허용할 네트워크는 **서버 자신(localhost)뿐** |
| `permit_mynetworks` | `mynetworks` 에서 온 메일은 릴레이 허용 |
| `permit_sasl_authenticated` | 아이디·비밀번호로 **인증한 사용자**는 릴레이 허용 |
| `defer_unauth_destination` | 그 밖의 사람이 **남의 도메인**으로 보내려 하면 거부 |

Kali는 `mynetworks` 에 없습니다. Kali에서 외부 도메인(`example.com`)으로 보내 봅니다.

```bash
# === Kali ===
nc -C 192.168.0.30 25
```

접속한 뒤 아래를 한 줄씩 입력합니다.

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
454 4.7.1 <someone@example.com>: Relay access denied
QUIT
221 2.0.0 Bye
```

📷 **캡처**: `Relay access denied` 화면

`RCPT TO` 단계에서 **`Relay access denied`** 로 거부되었습니다. 이 서버는 오픈 릴레이가 아닙니다. 반면 3.2절처럼 **이 서버가 받는 주소**(`@ubuntu.lab.local`)로 보내는 메일은 정상적으로 받습니다.

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
| `systemctl reload postfix` | 서비스를 멈추지 않고 **설정만 다시 읽음**. `restart` 는 서비스를 껐다 켜므로 그 사이 접속이 잠깐 끊기지만, `reload` 는 끊기지 않음 |

`postconf -e` 와 `systemctl reload` 는 아무것도 출력하지 않습니다. 첫 줄은 수정 전, 둘째 줄은 수정 후 조회 결과입니다.

```
disable_vrfy_command = no
disable_vrfy_command = yes
```

### [확인] (Kali) VRFY가 막혔는지 확인

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

📷 **캡처**: `502 5.5.1 VRFY command is disabled` 화면

| 확인 | 의미 |
|---|---|
| `EHLO` 기능 목록에서 **`250-VRFY` 줄이 사라짐** | 서버가 더 이상 VRFY 기능을 알리지 않음 |
| `VRFY root` → **`502 ... disabled`** | 계정이 있는지 알려 주지 않음 |

> Postfix는 `EXPN` 명령을 처음부터 지원하지 않습니다. Sendmail 같은 다른 메일 서버는 `EXPN` 도 따로 막아야 합니다.
{: .prompt-info }

---

## 3.5 Active Contents 공격

### [이론]

요즘 메일은 글자만 있는 것이 아니라 **HTML 문서**로 꾸며져 있습니다. 메일 프로그램이나 웹 브라우저는 이 HTML을 해석해서 보여 줍니다. 이때 메일 안에 **자바스크립트, 비주얼베이직 스크립트** 같은 실행 코드가 들어 있으면 메일을 **열기만 해도** 스크립트가 실행될 수 있습니다.

| 항목 | 내용 |
|---|---|
| 대상 | 메일을 열 때 **HTML 기능이 있는 메일 클라이언트나 웹 브라우저**를 쓰는 사용자 |
| 수법 | 자바스크립트, 비주얼베이직 스크립트 등을 메일에 넣어 실행 |
| 피해 | 사용자 컴퓨터의 정보 유출, 악성코드 설치 |
| 대책 | 메일 프로그램에서 **스크립트 기능을 사용하지 않도록 설정** |

> 웹메일 서비스는 메일 본문의 `<script>` 를 지우고 보여 주며, Outlook 같은 메일 프로그램도 기본으로 스크립트 실행과 외부 이미지 자동 표시를 막습니다. 이 설정을 함부로 풀지 않는 것이 중요합니다.
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

### [실습 1] (Ubuntu) 실행파일 첨부 차단 규칙 만들기

Postfix의 **`mime_header_checks`** 는 첨부파일의 헤더를 검사하는 기능입니다. 첨부파일 이름이 `.exe`, `.bat`, `.scr`, `.js`, `.vbs` 로 끝나면 거부하는 규칙을 만듭니다.

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

`tee`, `postconf -e`, `systemctl reload` 는 아무것도 출력하지 않습니다. 마지막 조회 명령의 결과만 나옵니다.

```
mime_header_checks = regexp:/etc/postfix/mime_header_checks
```

| 명령·옵션 | 설명 |
|---|---|
| `tee 파일 > /dev/null <<'EOF' ... EOF` | `EOF` 두 줄 사이의 내용을 파일로 저장. `sudo tee` 를 써야 관리자만 쓸 수 있는 `/etc` 아래에 저장 가능. `> /dev/null` 은 같은 내용이 화면에 한 번 더 나오지 않게 버림 |
| `postconf -e 'mime_header_checks = regexp:파일'` | 첨부파일 헤더 검사 규칙으로 이 파일을 사용. `regexp:` 는 파일 안의 규칙이 **정규표현식**(문자 패턴)이라는 뜻 |
| `systemctl reload postfix` | 설정을 다시 읽어 규칙 적용 |

규칙 한 줄의 뜻은 다음과 같습니다.

| 부분 | 의미 |
|---|---|
| `/name="?.*\.(...)"?$/` | 첨부파일 이름(`name=`)이 괄호 안의 확장자(`exe`, `bat`, `scr`, `js`, `vbs`) 가운데 하나로 끝나는 줄 |
| `REJECT Attachment type not allowed` | 그런 메일은 **거부**하고 이 문구를 돌려줌 |

### [실습 2] (Kali) 첨부파일을 붙여 보내기

첨부파일이 있는 메일은 손으로 입력하기 어려우므로 메일 테스트 도구 **swaks** 를 씁니다.

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
| `cd ~ && echo ... > invoice.exe` | 홈 디렉터리로 이동한 뒤 시험용 파일 두 개를 만듦. 내용은 의미 없는 글자이고, **확장자만** 실행파일(`.exe`)과 문서(`.txt`)로 다름 |
| `--server 192.168.0.30` | 접속할 메일 서버 |
| `--from` / `--to` | 보내는 사람 / 받는 사람 주소 (`MAIL FROM` / `RCPT TO` 에 해당) |
| `--header "Subject: ..."` | 메일 헤더 추가(여기서는 제목) |
| `--body "..."` | 메일 본문 |
| `--attach @invoice.exe` | 파일을 첨부. `@` 는 "이 이름의 파일 내용을 읽어서 붙이라"는 뜻 |
| `--suppress-data` | 보낸 메일 내용(헤더·본문·첨부를 인코딩한 글자) 수십 줄을 화면에 다 찍지 않고 `25 lines sent` 한 줄로 줄여 표시 |
| 줄 끝의 `\` | 명령이 길어서 다음 줄에 이어 쓴다는 표시 |

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
<** 550 5.7.1 Attachment type not allowed
 -> QUIT
<-  221 2.0.0 Bye
=== Connection closed with remote host.
```

| 표시 | 의미 |
|---|---|
| `===` | swaks 자신의 안내 메시지 |
| ` ->` | Kali(swaks)가 **보낸** 명령 |
| `<-` | 서버의 **정상 응답** |
| `<**` | 서버의 **오류 응답** |

이번에는 `.txt` 파일을 붙여 보냅니다.

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

> ` -> EHLO kali` 의 `kali` 자리에는 Kali의 호스트 이름이 나옵니다. 큐 번호(`CE9C57AB`)는 실행할 때마다 다릅니다.
{: .prompt-info }

📷 **캡처**: `.exe` 는 `550 ... Attachment type not allowed`, `.txt` 는 `queued as` 로 나오는 두 화면

| 첨부파일 | 결과 |
|---|---|
| `invoice.exe` | **`550` 거부** — 실행파일이라 차단 |
| `note.txt` | **`250` 접수** — 정상 전달 |


### [확인] (Ubuntu) 메일 로그

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

- 첫 줄: `name="invoice.exe"` 첨부가 **`reject`** 되었고, 보낸 곳 IP(`192.168.0.10`)와 주소가 기록되었습니다.
- 둘째 줄: `.txt` 첨부 메일은 **`status=sent (delivered to mailbox)`**, 메일함에 전달되었습니다.

---

## 4.2 SPF·DKIM·DMARC — 발신 도메인 확인

### [이론]

3.2절의 위조 메일은 SMTP가 보내는 사람을 확인하지 않아서 생겼습니다. **SPF(Sender Policy Framework)** 는 이를 막기 위해 **도메인 주인이 "우리 메일은 이 IP에서만 나간다"는 목록을 DNS에 공개**하는 기술입니다.

| 단계 | 하는 일 |
|---|---|
| ① 등록 | 도메인 관리자가 **자기 메일 발송 서버의 IP**를 DNS에 **TXT 레코드**로 등록 |
| ② 수신 | 받는 메일 서버가 메일의 발신 도메인을 확인 |
| ③ 조회 | 그 도메인의 SPF 레코드를 DNS에서 조회 |
| ④ 비교 | 메일을 실제로 보내온 서버의 IP(`Received` 에 남는 IP)가 목록에 있는지 비교 |
| ⑤ 판정 | 목록에 없으면 **위조로 판단**해 거부하거나 스팸으로 분류 |

SPF는 **허용된 도메인과 IP에서 온 메일인지 확인해 스팸을 차단하는 기술**입니다. 웹 보안이나 VoIP와는 관계가 없습니다.

| 기술 | 확인하는 것 | DNS에 등록하는 것 |
|---|---|---|
| **SPF** | 보낸 서버의 **IP**가 허용 목록에 있는가 | 허용 IP 목록 (TXT) |
| **DKIM** | 메일에 붙은 **전자서명**이 도메인의 공개키로 맞는가 | 서명 확인용 공개키 (TXT) |
| **DMARC** | SPF·DKIM 검사에 실패한 메일을 **어떻게 처리할지** | 처리 정책 `none`/`quarantine`/`reject` (TXT) |

### [실습] (Kali) 실제 도메인의 SPF·DMARC 조회

```bash
# === Kali ===
dig TXT naver.com +short | grep spf
dig TXT google.com +short | grep spf
dig TXT _spf.google.com +short
```

| 명령·옵션 | 설명 |
|---|---|
| `dig` | DNS 서버에 레코드를 물어보는 조회 도구 |
| `TXT` | 조회할 레코드 종류. SPF는 **TXT 레코드**에 들어 있음 |
| `naver.com` | 조회할 도메인 |
| `+short` | 설명 없이 **결과 값만** 짧게 출력 |
| <code>&#124; grep spf</code> | 한 도메인에 TXT 레코드가 여러 개 있을 수 있으므로 `spf` 가 들어 있는 줄만 골라냄 |

```
"v=spf1 ip4:111.91.135.0/27 ip4:125.209.208.0/20 ip4:125.209.224.0/19 ip4:210.89.163.112 ip4:210.89.173.104/29 ip4:114.111.35.0/24 ip4:61.247.196.0/24 ip4:61.247.197.0/24 ~all"
"v=spf1 include:_spf.google.com ~all"
"v=spf1 ip4:74.125.0.0/16 ip4:209.85.128.0/17 ip6:2001:4860:4864::/56 ip6:2404:6800:4864::/56 ip6:2607:f8b0:4864::/56 ip6:2800:3f0:4864::/56 ip6:2a00:1450:4864::/56 ip6:2c0f:fb50:4864::/56 ~all"
```

📷 **캡처**: naver.com의 `v=spf1 ip4:...` 레코드

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

> 3.2절의 `admin@bank.com` 위조 메일이 SPF를 확인하는 서버에 도착하면, `bank.com` 의 SPF 목록에 `192.168.0.10` 이 없으므로 위조로 판정됩니다.
{: .prompt-tip }

---

## 4.3 전송 구간 암호화 — STARTTLS

### [이론]

PGP·S/MIME(03-1)은 **메일 내용 자체**를 암호화합니다. 이와 별도로 **메일 서버 사이, 사용자와 서버 사이의 통신 구간**도 암호화해야 SMTP 명령과 로그인 정보가 노출되지 않습니다.

| 방식                    | 내용                                                    |
| --------------------- | ----------------------------------------------------- |
| **STARTTLS**          | 평문으로 접속한 뒤 `STARTTLS` 명령으로 **TLS 암호화로 전환** (25, 587번) |
| SMTPS / POP3S / IMAPS | 처음부터 TLS로 접속 (465 / 995 / 993번)                       |

### [확인] (Kali) 서버가 STARTTLS를 지원하는지 확인

```bash
# === Kali ===
nc -C 192.168.0.30 25
```

접속한 뒤 아래를 한 줄씩 입력합니다.

```
EHLO kali
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
QUIT
221 2.0.0 Bye
```

| 화면 | 의미 |
|---|---|
| **`250-STARTTLS`** | 이 서버는 평문으로 접속한 뒤 **TLS 암호화로 전환하는 기능**을 지원함 |
| `250-VRFY` 줄이 없음 | 3.4절에서 VRFY를 비활성화했기 때문 |

> Ubuntu의 Postfix는 설치할 때 자체 서명 인증서(`ssl-cert-snakeoil`)를 사용하도록 설정되어 있어서 STARTTLS가 처음부터 켜져 있습니다. 실제 운영 서버에서는 공인 인증기관이 발급한 인증서로 바꾸어야 상대 서버와 사용자가 인증서를 신뢰할 수 있습니다.
{: .prompt-info }

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

# 확인 문제

> 1. 메일 서비스 공격 유형 중 메일 서버에 대한 공격 기법은?
>    ① Active Contents 공격
>    ② 서비스 거부 공격
>    ③ 트로이 목마 공격
>    ④ 바이러스 메일
>
> 2. 이메일을 열람할 때 HTML 기능이 있는 클라이언트나 웹 브라우저 사용자를 대상으로, 자바스크립트 등을 이용해 정보를 유출하는 공격 기법은?
>
> 3. 스팸 필터 솔루션에 대한 설명으로 가장 부적절한 것은?
>    ① 메일 서버 앞단에 위치하며 프록시 메일 서버로 동작한다
>    ② SMTP 프로토콜을 이용한 DoS 공격이나 폭탄 메일, 스팸 메일을 차단한다
>    ③ 메일 헤더와 제목 필터링은 제공하지만 본문에 대한 필터링은 제공하지 못한다
>    ④ 첨부파일 필터링 기능으로 특정 확장자를 가진 파일만 전송되도록 설정할 수 있다
>
> 4. SPF(Sender Policy Framework)에 대한 설명으로 옳은 것은?
>    ① IP 보안 정책을 위한 프레임워크이다
>    ② 웹 서버에 업로드하는 송신자를 통제하는 웹 보안 정책이다
>    ③ VoIP에서 서비스 거부 공격을 완화하기 위해 송신자를 제한한다
>    ④ 메일 송신자가 허용된 도메인이나 IP 주소인지 확인하여 스팸을 통제한다
>
> 5. 메일 서버에서 `VRFY` 명령을 막아야 하는 이유를 쓰시오.
>
> 6. 외부 사용자가 자기 메일 서버를 거쳐 다른 도메인으로 메일을 보낼 수 있는 상태를 무엇이라 하며, 어떤 위험이 있는가?
{: .prompt-tip }

<details>
<summary>정답 보기</summary>

1. <strong>②</strong> Active Contents·트로이 목마는 클라이언트 공격, 바이러스 메일은 메일 서버 공격으로 보지 않는다.<br>
2. <strong>Active Contents 공격</strong> (대책: 스크립트 기능을 사용하지 않도록 설정)<br>
3. <strong>③</strong> 스팸 필터는 본문까지 검색하며, 내부에서 나가는 메일의 정보 유출도 막는다.<br>
4. <strong>④</strong><br>
5. 응답 코드의 차이로 서버에 <strong>실제 있는 계정</strong>을 알아낼 수 있어, 무차별 대입·스팸 대상 수집에 쓰이기 때문이다.<br>
6. <strong>오픈 릴레이(Open Relay)</strong>. 스팸 중계에 악용되어 서버가 블랙리스트에 오르고 정상 메일까지 거부된다.

</details>

---

# 체크리스트

- [ ] 클라이언트 대상 공격(Active Contents, 트로이 목마)과 서버 대상 공격(버퍼 오버플로우, DoS)을 구분
- [ ] (Kali) `admin@bank.com` 위조 메일이 `250` 으로 접수됨을 확인
- [ ] (Ubuntu) 위조 메일의 `Received` 에서 실제 IP `192.168.0.10` 확인
- [ ] (Kali) 외부 도메인 릴레이가 `Relay access denied` 로 막힘을 확인
- [ ] (Ubuntu) `disable_vrfy_command = yes` 적용 후 `502 ... disabled` 확인
- [ ] (Ubuntu·Kali) `.exe` 첨부 `550` 거부, `.txt` 첨부 `250` 접수, 로그의 `reject` 확인
- [ ] (Kali) `dig TXT` 로 SPF·DMARC 레코드를 조회하고 `~all`, `-all`, `p=reject` 를 설명
