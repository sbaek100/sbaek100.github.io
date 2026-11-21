---
title: "[애플리케이션 보안] 06-1. SSL/TLS 개념과 HTTPS 서버 구성"
date: 2026-10-22 09:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - SSL-TLS보안
tags:
  - SSL
  - TLS
  - HTTPS
  - 인증서
  - CA
  - Handshake
  - openssl
  - Apache
math: false
mermaid: false
---


## 0. 실습 환경

| 장비 | IP | 역할 |
|---|---|---|
| Kali | **192.168.0.10** | 웹 브라우저·`curl`·`openssl` 로 접속하는 쪽(클라이언트), Wireshark 캡처 |
| Ubuntu | **192.168.0.30** | 웹 서버(Apache), 실습용 인증기관(CA) |

> 코드 블록 첫 줄의 `# === Ubuntu ===` / `# === Kali ===` 를 보고 **어느 VM에서 칠지** 확인한 뒤 붙여 넣습니다.
>
> 명령과 예상 화면은 **Ubuntu 24.04.4 LTS(Apache 2.4.58, OpenSSL 3.0.13)** 와 **Kali Rolling(OpenSSL 3.6.2, curl 8.21.0, Wireshark 4.6.6)** 에서 실행해 확인했습니다. 날짜·시각, 인증서 일련번호와 지문, 키를 만들 때 나오는 점(`.`)과 `+` 기호, 프로세스 번호, 웹 응답의 `ETag` 값은 실행할 때마다 다릅니다.
{: .prompt-warning }

> 이 글의 실습은 **모두 무료 오픈소스**만 씁니다. 웹 서버는 **Apache**, 인증서는 **OpenSSL** 로 직접 만든 **실습용 인증기관(Lab Root CA)** 이 발급합니다. 돈을 내고 인증서를 사거나, 인터넷에 공개된 도메인을 준비할 필요가 없습니다(2.1절 끝의 "인증서를 얻는 방법" 참고).
{: .prompt-info }

이 글에서 Kali와 Ubuntu가 하는 일은 다음과 같습니다.

| 절 | Kali에서 할 일 | Ubuntu에서 할 일 |
|---|---|---|
| 1.1 평문 HTTP의 문제 | HTTP로 로그인 정보 보내고 Wireshark로 엿보기 | Apache 설치, 방화벽 허용 |
| 1.2 SSL/TLS의 위치와 포트 | — | TLS를 쓰는 서비스 포트 확인 |
| 2.1 인증서와 CA | 미리 설치된 루트 인증서 확인 | — |
| 2.2 인증서 발급 | — | 실습용 CA 만들기, 서버 인증서 발급 |
| 2.3 HTTPS 적용 | — | Apache에 인증서 연결, 443 포트 열기 |
| 2.4 인증서 신뢰 | 경고 확인 → CA 인증서 설치 → 경고 사라짐 | — |
| 2.5 Handshake | `curl` 과 Wireshark로 Handshake 관찰, 비밀번호가 숨는 것 확인 | — |

---

# 1. SSL/TLS 개념

## 1.1 평문 HTTP의 문제

### [이론]

웹(HTTP, 80번 포트)은 요청과 응답을 **평문**으로 주고받습니다. 02장의 FTP, 03장의 SMTP와 같은 문제가 있어서, 경로 위의 도청자는 **로그인할 때 보내는 아이디와 비밀번호까지** 그대로 볼 수 있습니다.

웹 서버가 아이디와 비밀번호를 요구하는 가장 단순한 방식이 **HTTP 기본 인증(Basic Authentication)** 입니다. 브라우저는 `아이디:비밀번호` 를 **Base64** 로 바꿔 `Authorization` 헤더에 담아 보냅니다.

| 원래 글자 | Base64로 바꾼 글자 |
|---|---|
| `student:Summer2026!` | `c3R1ZGVudDpTdW1tZXIyMDI2IQ==` |

> **Base64는 암호화가 아닙니다.** 이메일 첨부파일처럼 글자가 아닌 데이터를 글자로 옮기는 **표현 방식(인코딩)** 일 뿐이라, 키 없이 누구나 원래 글자로 되돌립니다. 그래서 HTTP 기본 인증은 반드시 **HTTPS(HTTP + TLS)** 와 함께 써야 합니다.
{: .prompt-danger }

### [실습 1] (Ubuntu) Apache 설치와 방화벽

```bash
# === Ubuntu ===
sudo apt update && sudo apt install -y apache2
systemctl is-active apache2
apache2 -v
```

| 명령·옵션 | 설명 |
|---|---|
| `apt install -y apache2` | 웹 서버 **Apache** 설치. 설치가 끝나면 바로 80번 포트로 동작함 |
| `systemctl is-active apache2` | Apache가 동작 중인지 한 단어로 표시 |
| `apache2 -v` | 설치된 Apache의 버전 표시 |

`apt` 는 설치 과정을 길게 출력합니다. 끝부분은 다음과 같습니다(설치 진행 줄 수십 개는 `...` 로 생략).

```
...
Setting up apache2 (2.4.58-1ubuntu8.19) ...
Enabling module mpm_event.
Enabling module authz_core.
...
Enabling module status.
Enabling module reqtimeout.
Enabling conf charset.
Enabling conf localized-error-pages.
Enabling conf other-vhosts-access-log.
Enabling conf security.
Enabling conf serve-cgi-bin.
Enabling site 000-default.
Created symlink /etc/systemd/system/multi-user.target.wants/apache2.service → /usr/lib/systemd/system/apache2.service.
Created symlink /etc/systemd/system/multi-user.target.wants/apache-htcacheclean.service → /usr/lib/systemd/system/apache-htcacheclean.service.
...
```

나머지 두 명령의 결과는 다음과 같습니다.

```
active
Server version: Apache/2.4.58 (Ubuntu)
Server built:   2026-11-12T12:58:56
```

> 패키지 버전 숫자(`8.19`)와 빌드 날짜는 업데이트 시점에 따라 다를 수 있습니다. 01장 환경 점검에서 nginx를 지우지 않았다면 80번 포트가 겹쳐 Apache가 시작되지 않으니, 먼저 nginx를 제거합니다.
{: .prompt-info }

Ubuntu 방화벽(ufw)에서 Kali가 웹(80·443)에 접속할 수 있도록 허용합니다.

```bash
# === Ubuntu ===
sudo ufw status
sudo ufw allow from 192.168.0.10 to any port 80,443 proto tcp
```

01강에서 방화벽을 켜고 실습망(`192.168.0.0/24`)을 허용해 둔 경우의 화면입니다.

```
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
Anywhere                   ALLOW       192.168.0.0/24            
OpenSSH (v6)               ALLOW       Anywhere (v6)             

Rule added
```

| 화면 | 의미 |
|---|---|
| `Anywhere ALLOW 192.168.0.0/24` | 01강에서 넣은 규칙. 이것만으로도 Kali의 웹 접속은 이미 허용됨 |
| `Rule added` | 이번에 Kali(`192.168.0.10`)의 80·443번만 허용하는 규칙을 하나 더 넣음. 겹쳐도 문제없음 |

> 규칙 목록은 이전 실습에서 무엇을 넣었는지에 따라 다를 수 있습니다(예: `OpenSSH` 대신 `22/tcp`). 방화벽이 꺼져 있으면 `Status: inactive` 다음에 `Rules updated` 가 나오고, 같은 규칙을 전에 넣은 적이 있으면 `Skipping adding existing rule` 이 나옵니다. 어느 경우든 그대로 진행합니다.
{: .prompt-info }

| 명령·옵션 | 설명 |
|---|---|
| `ufw allow from 192.168.0.10 to any port 80,443 proto tcp` | Kali에서 이 서버의 80·443번 포트로 오는 TCP 접속을 허용. 포트를 쉼표로 여러 개 적을 때는 `proto tcp` 를 꼭 붙임 |

### [실습 2] (Kali) HTTP로 로그인 정보 보내고 엿보기

03장 1.4절처럼 Wireshark를 켜고 캡처를 시작합니다.

```bash
# === Kali ===
wireshark &
```

| 순서 | 할 일 |
|---|---|
| ① | 위쪽 **캡처 필터** 칸에 `tcp port 80 or tcp port 443` 입력 → 웹 패킷만 모음 |
| ② | 인터페이스 목록에서 **`eth0`**(192.168.0.10이 붙은 카드)을 더블클릭 → 캡처 시작 |
| ③ | **디스플레이 필터** 칸에 `http` 입력 후 `Enter` |

캡처를 켠 채로, 기본 인증으로 아이디 `student`, 비밀번호 `Summer2026!` 를 보내며 웹 페이지를 요청합니다.

```bash
# === Kali ===
curl -s -u 'student:Summer2026!' -o /dev/null -w '%{http_code}\n' http://192.168.0.30/
```

```
200
```

| 명령·옵션 | 설명 |
|---|---|
| `curl` | 명령줄 웹 클라이언트. 브라우저 대신 웹 요청을 보내고 응답을 받음 |
| `-u 'student:Summer2026!'` | HTTP 기본 인증으로 아이디·비밀번호를 보냄. `!` 가 셸에서 특별한 뜻이 있으므로 작은따옴표로 감쌈 |
| `-s` | 진행 상황 표시를 끔 |
| `-o /dev/null` | 받은 웹 페이지 내용은 버림(화면에 출력하지 않음) |
| `-w '%{http_code}\n'` | 응답 상태 코드만 출력. `200` 은 정상 응답 |

> 이 페이지는 로그인을 요구하지 않으므로 서버는 그냥 기본 페이지를 돌려줍니다(`200`). 그래도 `curl` 은 비밀번호를 **이미 요청에 실어 보냈습니다.** 실제 로그인 페이지에서 일어나는 일과 같습니다.
{: .prompt-info }

Wireshark 위쪽 빨간 네모 버튼(**Stop**)으로 캡처를 멈춥니다. 패킷 목록의 Source·Destination·Protocol·Info 열은 다음과 같습니다.

```
Source        Destination   Protocol  Info
192.168.0.10  192.168.0.30  HTTP      GET / HTTP/1.1
192.168.0.30  192.168.0.10  HTTP      HTTP/1.1 200 OK  (text/html)
```

`GET / HTTP/1.1` 패킷을 클릭하고, 아래 상세 창에서 **Hypertext Transfer Protocol** 을 펼칩니다.

```
Hypertext Transfer Protocol
    GET / HTTP/1.1\r\n
        Request Method: GET
        Request URI: /
        Request Version: HTTP/1.1
    Host: 192.168.0.30\r\n
    Authorization: Basic c3R1ZGVudDpTdW1tZXIyMDI2IQ==\r\n
        Credentials: student:Summer2026!
    User-Agent: curl/8.21.0\r\n
    Accept: */*\r\n
    \r\n
    [Full request URI: http://192.168.0.30/]
```

> 📷 **캡처**: Wireshark 상세 창에서 `Credentials: student:Summer2026!` 가 보이는 화면.

| 화면 | 의미 |
|---|---|
| `Authorization: Basic c3R1...` | 브라우저(`curl`)가 보낸 로그인 정보. Base64로 바뀌어 있을 뿐 |
| `Credentials: student:Summer2026!` | **Wireshark가 Base64를 자동으로 풀어** 아이디·비밀번호를 그대로 보여 줌 |

Base64를 되돌리는 데는 키가 필요 없다는 것을 터미널에서도 확인합니다.

```bash
# === Kali ===
echo 'c3R1ZGVudDpTdW1tZXIyMDI2IQ==' | base64 -d; echo
```

```
student:Summer2026!
```

| 명령·옵션 | 설명 |
|---|---|
| `base64 -d` | Base64 글자를 원래 글자로 되돌림(`-d`, decode) |
| `; echo` | 결과 끝에 줄바꿈이 없으므로 줄을 바꿔 프롬프트가 붙지 않게 함 |

> 웹 로그인 정보도 FTP·SMTP처럼 **도청만으로 탈취**됩니다. 이 통신을 암호화하는 것이 **SSL/TLS** 이고, HTTP에 TLS를 더한 것이 **HTTPS(443)** 입니다. 2.5절에서 똑같은 요청을 HTTPS로 보내 비교합니다.
{: .prompt-danger }

---

## 1.2 SSL/TLS의 역사와 위치

### [이론]

**SSL(Secure Sockets Layer)** 은 **넷스케이프(Netscape)** 사가 웹 브라우저와 웹 서버 사이의 통신을 보호하려고 만든 프로토콜입니다. 이후 **IETF** 가 SSL 3.0을 바탕으로 표준을 만들면서 이름을 **TLS(Transport Layer Security)** 로 바꾸었습니다.

| 버전 | 발표 | 만든 곳 | 현재 상태 |
|---|---|---|---|
| SSL 2.0 | 1995 | 넷스케이프 | **사용 금지**(취약) |
| SSL 3.0 | 1996 | 넷스케이프 | **사용 금지**(POODLE 공격, 06-2 3.3절) |
| TLS 1.0 | 1999 (RFC 2246) | IETF | 사용 중단. 내부 버전 번호가 **3.1** 이라 "SSL 3.1"이라고도 부름 |
| TLS 1.1 | 2006 | IETF | 사용 중단 |
| **TLS 1.2** | 2008 (RFC 5246) | IETF | **사용** |
| **TLS 1.3** | 2018 (RFC 8446) | IETF | **사용**(가장 빠르고 안전) |

> 지금 쓰는 것은 **TLS 1.2와 1.3** 이지만, 습관적으로 "SSL"이라고 부르는 경우가 많습니다. "SSL 인증서", "SSL VPN"의 SSL은 대부분 TLS를 가리킵니다.
{: .prompt-info }

#### 어느 계층에서 동작하나

SSL/TLS는 **응용 계층(HTTP 등)과 전송 계층(TCP) 사이**에서 동작합니다. 응용 프로그램이 보낸 데이터를 TCP로 넘기기 직전에 암호화하고, 받은 데이터는 응용 프로그램에 넘기기 직전에 복호화합니다. 그래서 HTTP뿐 아니라 메일·FTP·LDAP처럼 **TCP를 쓰는 여러 응용 프로토콜**에 그대로 끼워 쓸 수 있습니다.

| 계층 | 보안 프로토콜 | 보호하는 것 |
|---|---|---|
| 응용 계층 | **S-HTTP**, **PGP**, **S/MIME**, **SET** | 특정 응용의 메시지(웹 문서 하나, 메일 한 통, 결제 정보) |
| 응용과 전송 사이(흔히 "전송 계층 보안") | **SSL/TLS** | TCP 연결 하나의 모든 데이터 |
| 네트워크 계층 | **IPSec** | IP 패킷 전체(07장) |
| 데이터링크 계층 | **PPTP**, **L2TP**, L2F | 두 지점 사이의 링크(터널) |

> **"SSL은 IP(네트워크) 계층에서 동작한다"** 는 설명은 틀립니다. 네트워크 계층 보안은 **IPSec**, VPN 터널링 프로토콜 PPTP·L2TP는 **데이터링크(2계층)** 입니다. 계층과 프로토콜을 짝짓는 문제는 혼동하기 쉬우니 표를 그대로 기억해 둡니다.
{: .prompt-warning }

#### SSL/TLS를 쓰는 서비스와 포트

응용 프로토콜에 TLS를 더하면 이름 끝에 **S(Secure)** 를 붙이고, 보통 **다른 포트 번호**를 씁니다.

| 평문 | 포트 | TLS를 더한 것 | 포트 |
|---|---|---|---|
| HTTP | 80 | **HTTPS** | **443** |
| SMTP(메일 제출) | 25·587 | **SMTPS** | **465** |
| IMAP | 143 | **IMAPS** | **993** |
| POP3 | 110 | **POP3S** | **995** |
| LDAP | 389 | **LDAPS** | **636** |
| FTP | 21 | **FTPS**(암시적) | **990** |
| DNS | 53 | **DoT**(DNS over TLS) | **853** |

> 포트를 바꾸지 않고, 평문으로 연결한 뒤 **`STARTTLS` 명령으로 도중에 TLS로 전환**하는 방식도 있습니다. 03-1에서 SMTP 서버가 `250-STARTTLS` 로 제안하던 것이 이 방식입니다(25·587번 포트).
{: .prompt-info }

> 휴대 단말기용 무선 인터넷(WAP)에서는 TLS를 무선 환경에 맞게 줄인 **WTLS(Wireless TLS)** 와, 무선 환경용 공개키 기반 구조 **WPKI(Wireless PKI)** 를 썼습니다. WPKI는 인증서를 다루는 **체계**이지 데이터를 암호화하는 **알고리즘이 아닙니다.**
{: .prompt-info }

### [실습] (Ubuntu) TLS를 쓰는 서비스 포트 확인

리눅스는 잘 알려진 서비스 이름과 포트 번호를 `/etc/services` 파일에 적어 둡니다.

```bash
# === Ubuntu ===
grep -wE '^(https|submissions|imaps|pop3s|ldaps|ftps|domain-s)' /etc/services
```

```
https		443/tcp				# http protocol over TLS/SSL
https		443/udp				# HTTP/3
submissions	465/tcp		ssmtp smtps urd # Submission over TLS [RFC8314]
ldaps		636/tcp				# LDAP over SSL
ldaps		636/udp
domain-s	853/tcp				# DNS over TLS [RFC7858]
domain-s	853/udp				# DNS over DTLS [RFC8094]
ftps-data	989/tcp				# FTP over SSL (data)
ftps		990/tcp
imaps		993/tcp				# IMAP over SSL
pop3s		995/tcp				# POP-3 over SSL
```

| 명령·옵션 | 설명 |
|---|---|
| `grep -E '^(https...)'` | 줄 맨 앞(`^`)이 괄호 안에 세로 막대로 나열한 이름 중 하나로 시작하는 줄만 표시 |
| `-w` | 단어 전체가 같을 때만 찾음(`https` 를 찾을 때 `httpsxxx` 같은 줄은 제외) |

| 화면 | 의미 |
|---|---|
| `submissions 465/tcp ... smtps` | SMTPS(465)의 정식 이름. 별명 `smtps` 가 함께 적혀 있음 |
| `443/udp # HTTP/3` | 최신 HTTP/3는 TCP 대신 UDP(QUIC)를 쓰며, 그 안에서 TLS 1.3을 씀 |
| `# ... over TLS/SSL` | 이름 끝의 `s` 가 모두 TLS(SSL)를 더했다는 뜻 |

---

## 1.3 SSL/TLS가 제공하는 보안 서비스

### [이론]

| 보안 서비스 | 방법 | 쓰는 알고리즘(예) |
|---|---|---|
| **기밀성** | **대칭키 암호**로 데이터를 암호화 | 현재 **AES**, ChaCha20. 옛날에는 DES, 3DES, RC4, IDEA, 수출용 DES-40·RC4-40 |
| **무결성** | **MAC**(메시지 인증 코드, 주로 HMAC)을 붙여 변조를 탐지 | SHA-256, SHA-384 (옛날 MD5, SHA-1) |
| **인증** | **인증서**(공개키 + CA의 전자서명)로 상대가 진짜인지 확인 | RSA, ECDSA |
| 키 교환 | 대칭키를 만들 재료를 **공개키 암호**로 안전하게 주고받음 | RSA, DH·DHE, **ECDHE** |

TLS는 두 종류의 암호를 섞어 씁니다(**하이브리드 방식**).

| 구분 | 대칭키 암호 | 공개키 암호 |
|---|---|---|
| 키 | 암호화와 복호화에 **같은 키** | **공개키**(누구나)와 **개인키**(주인만) 한 쌍 |
| 속도 | **빠름** | 느림 |
| 약점 | 처음에 키를 어떻게 안전하게 나눠 가질지가 문제 | 느려서 많은 데이터를 암호화하기 어려움 |
| TLS에서 맡는 일 | 실제 데이터(웹 페이지, 비밀번호) 암호화 | **대칭키를 안전하게 나누기**, 서버 인증 |

> 한 줄로 정리하면 **"공개키 암호로 대칭키를 안전하게 나눈 뒤, 빠른 대칭키로 본 데이터를 암호화한다"** 입니다. 03장의 PGP 전자봉투와 같은 아이디어입니다.
{: .prompt-tip }

#### SSL/TLS가 해 주지 않는 것

| 오해 | 실제 |
|---|---|
| HTTPS를 쓰면 **더 빨라진다** | 암호화·복호화와 Handshake가 더해져 HTTP보다 **조금 느립니다**(요즘 장비에서는 체감하기 어려운 수준) |
| 데이터를 **통신 전·중·후 모두** 보호한다 | **통신하는 동안(전송 구간)만** 보호합니다. 서버에 저장된 데이터나 PC 안의 파일은 보호하지 않습니다 |
| 자물쇠가 있으면 **믿을 만한 회사**다 | 자물쇠는 "이 주소의 서버와 **암호화된 연결**을 했다"는 뜻일 뿐입니다. 사기 사이트도 인증서를 받을 수 있습니다 |
| 인증서는 **클라이언트가 공개키가 없어도** 서버를 인증해 준다 | 서버 인증은 **CA의 공개키**(루트 인증서)를 클라이언트가 **미리 가지고 있어야** 가능합니다(2.1절) |
| 쇼핑몰에 카드번호를 보내도 안전하다 | 전송 구간은 안전하지만 **카드번호는 쇼핑몰(상점)에 그대로 전달**됩니다. 이 문제를 해결하려고 만든 것이 07장의 **SET**(이중서명)입니다 |
| 부인방지를 제공한다 | 대칭키 MAC은 양쪽이 같은 키를 가지므로 "누가 보냈는지"를 제3자에게 증명하지 못합니다 |

---

## 1.4 SSL/TLS를 이루는 프로토콜 — Handshake·ChangeCipherSpec·Alert·Record

### [이론]

SSL/TLS는 하나의 프로토콜이 아니라 **4개의 작은 프로토콜**로 이루어져 있습니다.

| 위치 | 프로토콜 | 하는 일 |
|---|---|---|
| 위층 | **Handshake** | 버전·암호 방식(암호군)을 **협상**하고, 서버(필요하면 클라이언트)를 **인증**하고, 대칭키를 만들 재료를 **교환** |
| 위층 | **ChangeCipherSpec** | "지금부터 방금 합의한 암호 방식으로 바꾼다"고 알리는 **1바이트짜리** 메시지 |
| 위층 | **Alert** | 오류나 종료를 알림. 경고(warning)와 치명적(fatal) 두 단계. 예: 버전 불일치, 인증서 오류, 정상 종료(close_notify) |
| 위층 | (응용 데이터) | HTTP 같은 응용 프로그램의 데이터 |
| 아래층 | **Record** | 위의 모든 메시지를 받아 **잘게 나누고, 압축하고, MAC을 붙이고, 암호화해서** TCP로 보냄 |

#### Record 프로토콜의 처리 순서

Record 프로토콜은 보낼 데이터를 다음 순서로 처리합니다.

| 순서 | 단계 | 하는 일 |
|---|---|---|
| ① | **단편화**(Fragmentation) | 데이터를 최대 16KB(2¹⁴바이트) 크기의 조각으로 나눔 |
| ② | **압축**(Compression) | 조각을 압축(선택. 현재는 보안 문제로 **사용하지 않음**, 06-2 3.4절) |
| ③ | **MAC 추가** | 조각에 메시지 인증 코드를 붙임 → **무결성**. 재전송 공격을 막기 위해 일련번호도 MAC 계산에 넣음 |
| ④ | **암호화** | 조각과 MAC을 함께 **대칭키**로 암호화 → **기밀성** |
| ⑤ | **헤더 추가** | 내용 종류(handshake, alert, 응용 데이터 등)·버전·길이를 적은 **Record 헤더**를 붙여 TCP로 보냄 |

> 순서 **"단편화 → 압축 → MAC → 암호화 → 헤더"** 는 그대로 외워 둡니다. 압축을 암호화 **뒤에** 하면 효과가 없고(암호문은 압축되지 않음), MAC은 암호화 **앞에** 붙입니다.
{: .prompt-warning }

> 이 순서는 SSL과 TLS 1.2까지의 기본 구조입니다. 요즘 쓰는 **AES-GCM·ChaCha20** 같은 AEAD 암호는 암호화와 무결성 확인을 한 번에 처리해 MAC 단계가 따로 없고, TLS 1.3에서는 압축 단계도 없어졌습니다.
{: .prompt-info }

| 자주 헷갈리는 점 | 바른 설명 |
|---|---|
| 암호 알고리즘은 어느 프로토콜이 **고르나**? | **Handshake** 가 협상해서 고름. Record는 정해진 대로 **실행**만 함 |
| 암호화 키는 어디서 나오나? | **Handshake** 에서 교환한 재료로 양쪽이 각자 계산 |
| Record가 보내는 것은 응용 데이터뿐인가? | 아님. Handshake·ChangeCipherSpec·Alert 메시지도 모두 Record에 담겨 나감 |

#### 세션과 연결

| 용어 | 뜻 |
|---|---|
| **세션**(Session) | Handshake로 만든 **보안 설정의 묶음**(암호 방식, 마스터 키 등). 한 번 만들면 다시 쓸 수 있음 |
| **연결**(Connection) | 실제 TCP 연결 하나. **여러 연결이 한 세션을 함께 쓸 수 있어** 매번 전체 Handshake를 하지 않아도 됨(세션 재사용) |

---

# 2. 인증서와 HTTPS 서버 구성

## 2.1 인증서와 인증기관(CA)

### [이론]

공개키 암호로 대칭키를 나누려면, 받은 공개키가 **정말 그 서버의 것인지** 알아야 합니다. 공격자가 자기 공개키를 서버의 것이라고 속여 보내면, 클라이언트는 공격자와 키를 나누게 됩니다(**중간자 공격**, 06-2 3.2절).

**인증서(Certificate)** 는 "이 공개키는 이 도메인의 것이다"라는 내용에 **인증기관(CA, Certificate Authority)** 이 **전자서명**을 해 준 전자 신분증입니다. 형식은 **X.509 버전 3** 입니다.

| X.509 인증서의 주요 항목 | 의미 |
|---|---|
| 버전(Version) | 보통 v3 |
| 일련번호(Serial Number) | CA가 붙인 인증서 고유 번호. **폐지할 때 이 번호로 구분** |
| 서명 알고리즘 | CA가 서명에 쓴 알고리즘(예: sha256WithRSAEncryption) |
| 발급자(Issuer) | 서명한 **CA의 이름** |
| 유효기간(Validity) | 시작일(notBefore)·만료일(notAfter) |
| 주체(Subject) | 인증서 주인. 웹 서버는 도메인 이름(CN) |
| 주체 공개키 | **서버의 공개키** |
| 확장(Extensions) | 주체 대체 이름(SAN, 접속 주소 목록), CA 여부(Basic Constraints) 등 |
| CA의 전자서명 | 위 내용 전체를 **CA의 개인키로 서명**한 값 |

#### 인증서를 믿을 수 있는 이유

| 질문 | 답 |
|---|---|
| 공격자가 인증서 내용을 바꾸면? | 서명이 맞지 않게 됨. 서명은 **CA의 개인키**로만 만들 수 있으므로 공격자가 다시 서명할 수 없음 |
| 서명은 어떻게 확인하나? | **CA의 공개키**로 확인. CA의 공개키는 **루트 인증서** 형태로 브라우저·운영체제에 **미리 설치**되어 있음 |
| 루트 인증서는 누가 보증하나? | 아무도 없음. 루트 CA는 **스스로 서명(자체서명)** 함. 그래서 어떤 루트 인증서를 미리 넣어 둘지를 브라우저·OS 회사가 엄격히 관리함 |

> 공개키가 가짜가 아님을 확인할 수 있는 이유는 **"인증서가 위조할 수 없는 CA의 개인키로 서명되어 있기 때문"** 입니다. "인증서가 암호화되어 있어서"나 "안전한 채널로 받아서"가 아닙니다.
{: .prompt-warning }

#### 신뢰의 사슬(체인)

| 단계 | 누가 서명 | 어디에 있나 |
|---|---|---|
| 루트 CA 인증서 | 자기 자신(자체서명) | 브라우저·OS에 미리 설치 |
| ↓ 중간 CA 인증서 | 루트 CA | 웹 서버가 함께 보내 줌 |
| ↓ 서버 인증서 | 중간 CA | 웹 서버가 보내 줌 |

브라우저는 서버 인증서에서 출발해 서명을 따라 올라가다가, **미리 가지고 있는 루트 인증서**에 닿으면 믿습니다. 닿지 못하면 경고를 띄웁니다.

#### 인증서 폐지 확인 — CRL과 OCSP

유효기간이 남았더라도 개인키가 유출되었거나 정보가 바뀌면 CA는 인증서를 **폐지(revoke)** 합니다.

| 방식 | 내용 |
|---|---|
| **CRL**(Certificate Revocation List) | CA가 **폐지한 인증서의 일련번호 목록**을 주기적으로 발행하고 CA가 서명함. 클라이언트가 목록 전체를 내려받아 확인 → 목록이 커지고, 발행 주기 사이에는 최신이 아님 |
| **OCSP**(Online Certificate Status Protocol) | 인증서 하나의 상태를 OCSP 서버에 **실시간**으로 물어봄(RFC 6960). 응답은 **good**(정상)·**revoked**(폐지)·**unknown**(모름) 셋 중 하나. 보통 HTTP로 주고받으며, 응답에는 OCSP 서버의 **전자서명**이 붙음 |
| OCSP 스테이플링 | 웹 서버가 OCSP 응답을 미리 받아 Handshake 때 **함께 보내 줌** → 클라이언트가 따로 묻지 않아도 됨 |

> CRL에 들어 있는 것은 **폐지된 인증서의 일련번호**이지 공개키가 아닙니다. OCSP 응답에 "bad" 같은 상태는 없습니다. OCSP 응답은 **암호화가 아니라 전자서명**으로 보호합니다.
{: .prompt-warning }

#### 인증서를 얻는 방법

| 방법 | 비용 | 브라우저 경고 | 쓰는 곳 |
|---|---|---|---|
| **자체서명 인증서** | 무료 | **항상 경고** | 잠깐 시험할 때 |
| **사설 CA**(직접 만든 CA)로 발급 | 무료(**OpenSSL**, easy-rsa, mkcert 등 오픈소스) | CA 인증서를 **설치한 PC에서는 경고 없음** | 회사 내부망, **이 실습** |
| **Let's Encrypt** 등 무료 공인 CA | 무료(발급 도구 **certbot** 도 오픈소스) | 없음 | **인터넷에 공개된 도메인**이 있는 서버. 인증서 유효기간이 90일(점점 짧아지는 중)이라 자동 갱신 |
| 유료 공인 CA | 유료 | 없음 | 기관 확인(OV)·확장 확인(EV)이 필요한 곳 |

> Let's Encrypt는 무료지만 "이 도메인의 주인이 맞는지"를 **인터넷에서 그 서버(또는 DNS)에 접속해** 확인합니다. 그래서 `192.168.0.30` 같은 **내부 IP나 인터넷에 없는 이름**(`ubuntu.lab.local`)에는 발급하지 않습니다. 내부망에서는 이 글처럼 **사설 CA** 를 만들고, 그 CA의 인증서를 직원 PC에 설치해 씁니다.
{: .prompt-info }

### [실습] (Kali) 미리 설치된 루트 인증서 확인

```bash
# === Kali ===
ls /etc/ssl/certs/*.pem | wc -l
openssl x509 -in /etc/ssl/certs/ISRG_Root_X1.pem -noout -subject -issuer -dates
```

```
121
subject=C=US, O=Internet Security Research Group, CN=ISRG Root X1
issuer=C=US, O=Internet Security Research Group, CN=ISRG Root X1
notBefore=Jun  4 11:04:38 2015 GMT
notAfter=Jun  4 11:04:38 2035 GMT
```

| 명령·옵션 | 설명 |
|---|---|
| `ls /etc/ssl/certs/*.pem` | 운영체제가 믿는 **루트 인증서** 파일 목록 |
| `wc -l` | 줄 수(= 파일 개수)를 셈 |
| `openssl x509 -in 파일` | X.509 인증서 파일을 읽음 |
| `-noout` | 인증서 원본(BEGIN CERTIFICATE ...)은 출력하지 않음 |
| `-subject -issuer -dates` | 주체, 발급자, 유효기간만 출력 |

| 화면 | 의미 |
|---|---|
| `121` | Kali가 기본으로 믿는 루트 CA가 121개(업데이트에 따라 달라짐) |
| `subject` 와 `issuer` 가 같음 | **ISRG Root X1**(Let's Encrypt의 루트 CA)이 **스스로 서명한 자체서명 인증서** |
| `notAfter=... 2035` | 루트 인증서는 유효기간이 수십 년으로 김 |

> 루트 인증서는 자체서명인데도 믿을 수 있는 이유는, OS·브라우저 회사가 검증한 뒤 **미리 넣어 두었기 때문**입니다. 2.4절에서는 우리가 만든 실습용 루트 CA를 이 목록에 **직접 추가**합니다.
{: .prompt-tip }

---

## 2.2 실습용 CA를 만들고 서버 인증서 발급하기

### [이론]

인증서는 다음 순서로 발급됩니다. 실제로는 CA가 별도 기관(또는 별도 서버)이지만, 실습에서는 Ubuntu의 `~/lab-ca` 폴더가 CA 역할을 합니다.

| 순서 | 누가 | 하는 일 | 만들어지는 파일 |
|---|---|---|---|
| ① | CA | CA의 개인키와 **자체서명 루트 인증서** 만들기 | `ca.key`, `ca.crt` |
| ② | 서버 | 서버의 개인키와 **인증서 서명 요청(CSR)** 만들기 | `server.key`, `server.csr` |
| ③ | CA | CSR을 확인하고 **CA 개인키로 서명**해 서버 인증서 발급 | `server.crt` |
| ④ | 서버 | 서버 인증서와 서버 개인키를 웹 서버에 설치 | (2.3절) |
| ⑤ | 클라이언트 | CA 인증서(`ca.crt`)를 **신뢰 목록에 설치** | (2.4절) |

> **CSR(Certificate Signing Request)** 은 "이 공개키와 이 이름으로 인증서를 만들어 주세요"라는 신청서입니다. 신청서에는 **공개키만** 들어가고, **개인키는 서버 밖으로 나가지 않습니다.**
{: .prompt-info }

### [실습 1] (Ubuntu) 실습용 루트 CA 만들기

```bash
# === Ubuntu ===
mkdir -p ~/lab-ca && cd ~/lab-ca
openssl req -x509 -newkey rsa:2048 -nodes -days 3650 \
  -keyout ca.key -out ca.crt \
  -subj "/C=KR/O=LabOrg/CN=Lab Root CA"
```

> 이 블록은 **한 번만** 실행합니다. 다시 실행하면 `ca.key`·`ca.crt` 가 **새 CA로 덮어써져**, 이미 발급한 서버 인증서와 Kali에 설치한 CA 인증서가 더 이상 맞지 않습니다. 실수로 다시 실행했다면 실습 2부터 2.4절 실습 4까지를 다시 진행합니다.
{: .prompt-warning }

키를 만드는 동안 점(`.`)과 `+` 가 몇 줄 출력되고 `-----` 로 끝납니다. 내용은 실행할 때마다 다릅니다.

```
....+...+..+.+..+....+........+.+.................+...+++++++++++++++++++++++++++++++++
..+......+.........+..+...+.......+..+++++++++++++++++++++++++++++++++*.+..+.+...+..+...
-----
```

| 명령·옵션 | 설명 |
|---|---|
| `openssl req` | 인증서 요청(CSR)이나 인증서를 만드는 명령 |
| `-x509` | CSR 대신 **자체서명 인증서**를 바로 만듦 |
| `-newkey rsa:2048` | 2048비트 RSA 키 쌍을 새로 만듦 |
| `-nodes` | 개인키 파일에 암호를 걸지 않음(No DES, 실습 편의). 실제 CA 키는 반드시 암호를 걸고 오프라인에 보관 |
| `-days 3650` | 유효기간 10년(루트 CA는 길게) |
| `-keyout ca.key` / `-out ca.crt` | 개인키 / 인증서를 저장할 파일 이름 |
| `-subj "/C=KR/O=LabOrg/CN=Lab Root CA"` | 주체 이름. 국가(C), 조직(O), 이름(CN)을 묻지 않고 바로 지정 |

```bash
# === Ubuntu ===
cd ~/lab-ca
openssl x509 -in ca.crt -noout -subject -issuer -ext basicConstraints
```

```
subject=C = KR, O = LabOrg, CN = Lab Root CA
issuer=C = KR, O = LabOrg, CN = Lab Root CA
X509v3 Basic Constraints: critical
    CA:TRUE
```

| 화면 | 의미 |
|---|---|
| `subject` 와 `issuer` 가 같음 | 루트 CA이므로 **자체서명** |
| `CA:TRUE` | 이 인증서는 **다른 인증서에 서명할 수 있는 CA 인증서**라는 표시 |

### [실습 2] (Ubuntu) 서버 키와 서명 요청(CSR) 만들기

```bash
# === Ubuntu ===
cd ~/lab-ca
openssl req -newkey rsa:2048 -nodes \
  -keyout server.key -out server.csr \
  -subj "/C=KR/O=LabOrg/CN=ubuntu.lab.local"
openssl req -in server.csr -noout -subject
```

키를 만드는 점과 `+` 줄, `-----` 가 실습 1처럼 나온 뒤 마지막 명령의 결과가 나옵니다.

```
....+.+..+...+++++++++++++++++++++++++++++++++*...+..
-----
subject=C = KR, O = LabOrg, CN = ubuntu.lab.local
```

| 명령·옵션 | 설명 |
|---|---|
| `openssl req` (`-x509` 없음) | 인증서가 아니라 **CSR(서명 요청)** 을 만듦 |
| `CN=ubuntu.lab.local` | 웹 서버의 이름 |
| `openssl req -in server.csr -noout -subject` | CSR에 적힌 주체 이름 확인 |

### [실습 3] (Ubuntu) CA가 서명해 서버 인증서 발급

요즘 브라우저는 인증서의 CN이 아니라 **SAN(Subject Alternative Name, 주체 대체 이름)** 에 적힌 주소만 확인합니다. 학생들은 IP 주소로 접속하므로, SAN에 이름과 IP를 함께 적습니다.

```bash
# === Ubuntu ===
cd ~/lab-ca
echo "subjectAltName=DNS:ubuntu.lab.local,IP:192.168.0.30" > server.ext
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 365 -extfile server.ext -out server.crt
openssl verify -CAfile ca.crt server.crt
```

```
Certificate request self-signature ok
subject=C = KR, O = LabOrg, CN = ubuntu.lab.local
server.crt: OK
```

| 명령·옵션 | 설명 |
|---|---|
| `server.ext` | 인증서에 넣을 **확장 항목**을 적은 파일. 여기서는 SAN(이름 `ubuntu.lab.local`, IP `192.168.0.30`) |
| `openssl x509 -req -in server.csr` | CSR을 받아 인증서를 만듦 |
| `-CA ca.crt -CAkey ca.key` | **CA의 인증서와 개인키로 서명** |
| `-CAcreateserial` | 일련번호를 기록할 파일(`ca.srl`)을 만들어 번호를 붙임 |
| `-days 365` | 서버 인증서 유효기간 1년 |
| `openssl verify -CAfile ca.crt server.crt` | `ca.crt` 를 믿는다고 할 때 `server.crt` 의 서명이 올바른지 검사 |

| 화면 | 의미 |
|---|---|
| `Certificate request self-signature ok` | CSR이 서버 개인키로 올바르게 서명되어 있음(신청자가 개인키를 가지고 있다는 증거) |
| `server.crt: OK` | 서버 인증서가 **Lab Root CA의 서명**으로 검증됨 |

발급된 인증서와 폴더의 파일을 확인합니다.

```bash
# === Ubuntu ===
cd ~/lab-ca
openssl x509 -in server.crt -noout -subject -issuer -dates -ext subjectAltName
ls -l
```

```
subject=C = KR, O = LabOrg, CN = ubuntu.lab.local
issuer=C = KR, O = LabOrg, CN = Lab Root CA
notBefore=Nov 16 08:05:52 2026 GMT
notAfter=Nov 16 08:05:52 2027 GMT
X509v3 Subject Alternative Name: 
    DNS:ubuntu.lab.local, IP Address:192.168.0.30
total 28
-rw-r--r-- 1 student student 1200 Nov 16 17:05 ca.crt
-rw------- 1 student student 1704 Nov 16 17:05 ca.key
-rw-r--r-- 1 student student   41 Nov 16 17:05 ca.srl
-rw-r--r-- 1 student student 1233 Nov 16 17:05 server.crt
-rw-r--r-- 1 student student  940 Nov 16 17:05 server.csr
-rw-r--r-- 1 student student   52 Nov 16 17:05 server.ext
-rw------- 1 student student 1704 Nov 16 17:05 server.key
```

| 화면 | 의미 |
|---|---|
| `issuer=... CN = Lab Root CA` | 이번에는 발급자가 자기 자신이 아니라 **CA**. 자체서명이 아님 |
| `notAfter` | 1년 뒤 만료 |
| `DNS:ubuntu.lab.local, IP Address:192.168.0.30` | 이 두 주소로 접속할 때만 인증서가 유효 |
| `ca.key`, `server.key` 가 `-rw-------` | **개인키는 주인만 읽을 수 있음**(600). 인증서(`.crt`)는 공개해도 되므로 644 |
| 파일 크기(`1704` 등) | 키 값에 따라 몇 바이트씩 달라질 수 있음(예: 1700) |

> **`ca.key` 는 이 실습에서 가장 중요한 파일입니다.** 이 키를 가진 사람은 **어떤 사이트의 인증서든** 만들 수 있고, `ca.crt` 를 믿는 모든 PC가 그 가짜 인증서를 믿습니다. 실제 CA는 루트 키를 인터넷과 분리된 장비에 보관합니다.
{: .prompt-danger }

---

## 2.3 Apache에 HTTPS 적용

### [실습] (Ubuntu) 인증서 설치와 SSL 사이트 켜기

인증서와 개인키를 Apache가 읽는 표준 위치에 복사합니다.

```bash
# === Ubuntu ===
sudo install -m 644 ~/lab-ca/server.crt /etc/ssl/certs/lab.crt
sudo install -m 600 ~/lab-ca/server.key /etc/ssl/private/lab.key
sudo ls -l /etc/ssl/certs/lab.crt /etc/ssl/private/lab.key
```

```
-rw-r--r-- 1 root root 1233 Nov 16 17:05 /etc/ssl/certs/lab.crt
-rw------- 1 root root 1704 Nov 16 17:05 /etc/ssl/private/lab.key
```

| 명령·옵션 | 설명 |
|---|---|
| `install -m 644 원본 대상` | 파일을 복사하면서 권한을 함께 지정. 인증서는 누구나 읽어도 됨(644) |
| `install -m 600 ...` | 개인키는 root만 읽고 쓰게(600) |
| `/etc/ssl/private` | 개인키를 두는 폴더. 일반 사용자는 들어갈 수 없음(그래서 `ls` 에도 `sudo` 가 필요) |

SSL 모듈을 켜고, 기본 SSL 사이트 설정(`default-ssl.conf`)이 우리 인증서를 쓰도록 바꿉니다.

```bash
# === Ubuntu ===
sudo a2enmod ssl
sudo sed -i \
  -e 's#/etc/ssl/certs/ssl-cert-snakeoil.pem#/etc/ssl/certs/lab.crt#' \
  -e 's#/etc/ssl/private/ssl-cert-snakeoil.key#/etc/ssl/private/lab.key#' \
  /etc/apache2/sites-available/default-ssl.conf
grep -E '^\s*SSLCertificate(Key)?File' /etc/apache2/sites-available/default-ssl.conf
```

```
Considering dependency mime for ssl:
Module mime already enabled
Considering dependency socache_shmcb for ssl:
Enabling module socache_shmcb.
Enabling module ssl.
See /usr/share/doc/apache2/README.Debian.gz on how to configure SSL and create self-signed certificates.
To activate the new configuration, you need to run:
  systemctl restart apache2
	SSLCertificateFile      /etc/ssl/certs/lab.crt
	SSLCertificateKeyFile   /etc/ssl/private/lab.key
```

| 명령·옵션 | 설명 |
|---|---|
| `a2enmod ssl` | Apache의 **SSL/TLS 모듈**(`mod_ssl`)을 켬 |
| `sed -i -e 's#옛값#새값#'` | 파일 안의 글자를 바꿔 저장(`-i`). 경로에 `/` 가 많아 구분자로 `#` 를 씀 |
| `ssl-cert-snakeoil` | Ubuntu가 설치할 때 자동으로 만들어 둔 **임시 자체서명 인증서**. 이것을 우리 인증서로 바꿈 |
| `SSLCertificateFile` | 서버 **인증서** 파일 위치 |
| `SSLCertificateKeyFile` | 서버 **개인키** 파일 위치 |

SSL 사이트를 켜고, 설정 문법을 검사한 뒤 Apache를 다시 시작합니다.

```bash
# === Ubuntu ===
sudo a2ensite default-ssl
sudo apache2ctl configtest
sudo systemctl restart apache2
sudo ss -tlnp | grep apache2
```

```
Enabling site default-ssl.
To activate the new configuration, you need to run:
  systemctl reload apache2
Syntax OK
LISTEN 0      511                                  *:443             *:*    users:(("apache2",pid=2075,fd=6),("apache2",pid=2073,fd=6),("apache2",pid=2071,fd=6))
LISTEN 0      511                                  *:80              *:*    users:(("apache2",pid=2075,fd=4),("apache2",pid=2073,fd=4),("apache2",pid=2071,fd=4))
```

| 명령·옵션 | 설명 |
|---|---|
| `a2ensite default-ssl` | 443번 포트용 사이트 설정(`default-ssl.conf`)을 켬 |
| `apache2ctl configtest` | 설정 파일 문법 검사. `Syntax OK` 여야 함 |
| `systemctl restart apache2` | 모듈을 새로 켰으므로 다시 시작(출력 없음) |
| `ss -tlnp` | 대기 중인 TCP 포트와 프로그램 표시 |

| 화면 | 의미 |
|---|---|
| `*:443 ... apache2` | **HTTPS(443)** 가 열림 |
| `*:80 ... apache2` | HTTP(80)도 그대로 열려 있음(06-2 3.6절에서 HTTPS로 넘기도록 바꿈) |

> `configtest` 에서 `AH00558: apache2: Could not reliably determine the server's fully qualified domain name` 경고가 함께 나올 수 있습니다. 서버 이름을 정하지 않았다는 안내일 뿐이며 실습에는 영향이 없습니다. 프로세스 번호(`pid=`)는 환경마다 다릅니다.
{: .prompt-info }

---

## 2.4 접속해 보기 — 인증서 경고와 CA 인증서 설치

### [실습 1] (Kali) CA를 믿기 전 — 경고 확인

Kali의 Firefox에서 `https://192.168.0.30` 에 접속합니다.

- **Warning: Potential Security Risk Ahead** 경고 화면이 뜹니다.
- **Advanced...** 를 누르면 오류 코드 **`SEC_ERROR_UNKNOWN_ISSUER`**(발급자를 모름)가 보입니다.

> 📷 **캡처**: Firefox 경고 화면과 `SEC_ERROR_UNKNOWN_ISSUER` 오류 코드.

여기서 **Accept the Risk and Continue** 는 누르지 않습니다. 같은 내용을 `curl` 로 확인합니다.

```bash
# === Kali ===
curl https://192.168.0.30/
```

```
curl: (60) SSL certificate OpenSSL verify result: unable to get local issuer certificate (20)
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the webpage mentioned above.
```

| 화면 | 의미 |
|---|---|
| `unable to get local issuer certificate` | 서버 인증서의 발급자(**Lab Root CA**)가 Kali의 루트 인증서 목록(2.1절의 121개)에 **없음** |
| `could not establish a secure connection` | 서버가 진짜인지 확인할 수 없어 `curl` 이 **접속을 거부** |

> 브라우저 경고는 "암호화가 안 된다"는 뜻이 아니라 **"상대가 진짜인지 확인할 수 없다"** 는 뜻입니다. 이 상태에서 경고를 무시하면, 중간에 끼어든 공격자의 가짜 서버와도 똑같이 연결됩니다(06-2 3.2절에서 직접 확인).
{: .prompt-danger }

### [실습 2] (Kali) CA 인증서 가져오기와 지문 확인

CA 인증서(`ca.crt`)는 공개해도 되는 파일이지만, **가짜로 바꿔치기되면 안 됩니다.** 그래서 비밀번호로 서버를 확인하는 **SSH(scp)** 로 가져옵니다.

```bash
# === Kali ===
scp student@192.168.0.30:lab-ca/ca.crt ~/lab-ca.crt
```

처음 접속할 때는 Ubuntu의 SSH 키를 믿을지 묻습니다. `yes` 를 입력하고, Ubuntu `student` 계정의 비밀번호를 입력합니다.

```
The authenticity of host '192.168.0.30 (192.168.0.30)' can't be established.
ED25519 key fingerprint is SHA256:ZLkc/SLwjKN7lkQhRM6j31JPFH3Pd6SVW9R3OiQQLFU.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.0.30' (ED25519) to the list of known hosts.
student@192.168.0.30's password: 
ca.crt                                        100% 1200     1.1MB/s   00:00
```

> 이전에 Kali에서 Ubuntu로 SSH 접속한 적이 있으면 앞의 다섯 줄은 나오지 않고 비밀번호만 묻습니다. SSH 키 지문과 전송 속도는 환경마다 다릅니다.
{: .prompt-info }

| 명령·옵션 | 설명 |
|---|---|
| `scp 계정@서버:원본 대상` | SSH로 암호화된 연결을 통해 파일을 복사. 원본 경로 `lab-ca/ca.crt` 는 `student` 의 홈 폴더 기준 |

받은 파일이 Ubuntu의 CA 인증서와 **같은지 지문으로 확인**합니다. 두 VM에서 각각 실행해 값을 비교합니다.

```bash
# === Ubuntu ===
openssl x509 -in ~/lab-ca/ca.crt -noout -fingerprint -sha256
```

```bash
# === Kali ===
openssl x509 -in ~/lab-ca.crt -noout -subject -fingerprint -sha256
```

Ubuntu의 결과입니다.

```
sha256 Fingerprint=E3:C6:CC:0A:07:44:6E:64:5D:C7:DB:F2:BD:DE:03:8E:14:D0:D2:E6:70:FE:35:80:80:AC:3F:58:1E:E7:03:EE
```

Kali의 결과입니다.

```
subject=C=KR, O=LabOrg, CN=Lab Root CA
sha256 Fingerprint=E3:C6:CC:0A:07:44:6E:64:5D:C7:DB:F2:BD:DE:03:8E:14:D0:D2:E6:70:FE:35:80:80:AC:3F:58:1E:E7:03:EE
```

| 명령·옵션 | 설명 |
|---|---|
| `-fingerprint -sha256` | 인증서 전체를 SHA-256으로 해시한 값(**지문**). 한 글자만 달라도 완전히 다른 값이 됨 |

> 두 지문이 **완전히 같아야** 합니다. 지문 값은 CA를 만들 때마다 달라지므로 화면의 값과는 다릅니다. Kali의 OpenSSL이 더 최신 버전이라 `=` 앞뒤 띄어쓰기가 Ubuntu와 조금 다르게 표시됩니다.
{: .prompt-tip }

### [실습 3] (Kali) CA 인증서를 운영체제에 설치

```bash
# === Kali ===
sudo cp ~/lab-ca.crt /usr/local/share/ca-certificates/lab-ca.crt
sudo update-ca-certificates
```

```
Updating certificates in /etc/ssl/certs...
rehash: warning: skipping ca-certificates.crt, it does not contain exactly one certificate or CRL
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
```

| 명령·옵션 | 설명 |
|---|---|
| `/usr/local/share/ca-certificates/` | 관리자가 **직접 추가하는 CA 인증서**를 두는 폴더. 파일 이름이 `.crt` 로 끝나야 함 |
| `update-ca-certificates` | 이 폴더의 인증서를 운영체제의 신뢰 목록(`/etc/ssl/certs`)에 반영 |

| 화면 | 의미 |
|---|---|
| `1 added` | Lab Root CA가 신뢰 목록에 **1개 추가**됨 |
| `rehash: warning: skipping ca-certificates.crt ...` | 여러 인증서를 묶은 파일을 건너뛴다는 안내. 무시해도 됨 |

이제 `curl` 과 `openssl` 로 다시 접속합니다.

```bash
# === Kali ===
curl -I https://192.168.0.30/
openssl s_client -connect 192.168.0.30:443 -brief < /dev/null
```

```
HTTP/1.1 200 OK
Date: Mon, 16 Nov 2026 08:06:10 GMT
Server: Apache/2.4.58 (Ubuntu)
Last-Modified: Mon, 16 Nov 2026 07:58:36 GMT
ETag: "29af-660602526f174"
Accept-Ranges: bytes
Content-Length: 10671
Vary: Accept-Encoding
Content-Type: text/html

Connecting to 192.168.0.30
Can't use SSL_get_servername
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: C=KR, O=LabOrg, CN=ubuntu.lab.local
Hash used: SHA256
Signature type: rsa_pss_rsae_sha256
Verification: OK
Peer Temp Key: X25519, 253 bits
DONE
```

| 명령·옵션 | 설명 |
|---|---|
| `curl -I` | 웹 페이지 본문 없이 **응답 헤더만** 받음 |
| `openssl s_client -connect 주소:443` | TLS 클라이언트. 서버와 Handshake를 하고 결과를 보여 줌 |
| `-brief` | 핵심 정보만 짧게 표시 |
| `< /dev/null` | 연결 뒤 보낼 내용이 없으므로 바로 종료 |

| 화면 | 의미 |
|---|---|
| `HTTP/1.1 200 OK` | **옵션 없이** HTTPS 접속 성공(실습 1에서는 거부) |
| `Protocol version: TLSv1.3` | 협상된 TLS 버전 |
| `Ciphersuite: TLS_AES_256_GCM_SHA384` | 협상된 **암호군**(대칭키 AES-256-GCM, 해시 SHA-384) |
| `Peer Temp Key: X25519` | 키 교환에 쓴 **임시 키**(타원곡선 Diffie-Hellman). 2.5절 참고 |
| `Verification: OK` | 서버 인증서를 **Lab Root CA로 검증하는 데 성공** |
| `Can't use SSL_get_servername` | 이름이 아니라 IP로 접속해 서버 이름(SNI)을 보내지 않았다는 안내. 오류 아님 |

### [실습 4] (Kali) Firefox에 CA 인증서 등록

Firefox는 운영체제의 신뢰 목록과 별도로 **자기만의 인증서 저장소**를 씁니다. 그래서 Firefox에도 한 번 등록합니다.

| 순서 | 할 일 |
|---|---|
| ① | Firefox 오른쪽 위 메뉴(☰) → **Settings** |
| ② | 왼쪽 **Privacy & Security** → 아래로 내려 **Certificates** 의 **View Certificates...** |
| ③ | **Authorities** 탭 → **Import...** → 홈 폴더의 `lab-ca.crt` 선택 |
| ④ | **Trust this CA to identify websites.** 에 체크 → **OK** |
| ⑤ | `https://192.168.0.30` 을 다시 열기 |

> 📷 **캡처**: Authorities 탭에서 `LabOrg` 아래 `Lab Root CA` 가 등록된 화면.

> 📷 **캡처**: 경고 없이 자물쇠가 표시된 `https://192.168.0.30` 화면. 자물쇠를 눌러 **Connection secure**, 인증서 발급자 **LabOrg** 가 보이는 화면.

> 실습 1에서 **Accept the Risk and Continue** 를 눌렀다면 Firefox가 그 예외를 기억하고 있습니다. 같은 **View Certificates...** 창의 **Servers** 탭에서 `192.168.0.30:443` 항목을 골라 **Delete...** 로 지운 뒤 다시 확인합니다.
{: .prompt-info }

| 상태 | `curl` | Firefox |
|---|---|---|
| CA 인증서 설치 전 | `unable to get local issuer certificate` | `SEC_ERROR_UNKNOWN_ISSUER` 경고 |
| CA 인증서 설치 후 | `200 OK`, `Verification: OK` | 경고 없이 자물쇠 |

> 회사에서는 이 과정을 직원 PC마다 손으로 하지 않고, 윈도우 그룹 정책(GPO)이나 단말 관리 시스템으로 **회사 루트 CA 인증서를 한꺼번에 배포**합니다.
{: .prompt-tip }

---

## 2.5 TLS Handshake

### [이론]

**Handshake** 는 데이터를 주고받기 전에 클라이언트와 서버가 **① 버전과 암호군을 정하고, ② 서버를 인증하고, ③ 대칭키를 나눠 갖는** 과정입니다. 아래는 TLS 1.2의 전체 Handshake입니다.

| 단계 | 방향 | 메시지 | 하는 일 |
|---|---|---|---|
| **1. 협상** | C → S | **ClientHello** | 지원하는 **최고 버전**, **랜덤 값**, 세션 ID, 지원하는 **암호군 목록**, 압축 방식 |
| | S → C | **ServerHello** | 고른 **버전**과 **암호군 하나**, 서버의 **랜덤 값**, 세션 ID |
| **2. 서버 인증·키 교환** | S → C | **Certificate** | 서버 **인증서**(공개키 포함) |
| | S → C | **ServerKeyExchange** | (DHE·ECDHE일 때) 서버의 **임시 키 교환 값**을 서버 개인키로 서명해 보냄 |
| | S → C | CertificateRequest | (선택) 클라이언트 인증서를 요구할 때 |
| | S → C | **ServerHelloDone** | "내 쪽 Hello 단계 끝" |
| **3. 클라이언트 키 교환** | C → S | Certificate | (선택) 요구받았을 때 클라이언트 인증서 |
| | C → S | **ClientKeyExchange** | 대칭키를 만들 재료(**프리마스터 시크릿**) 전달. RSA 방식이면 **서버 공개키로 암호화**해 보내고, ECDHE 방식이면 클라이언트의 임시 키 교환 값을 보냄 |
| | C → S | CertificateVerify | (선택) 클라이언트 인증서의 개인키를 가졌다는 증명 |
| **4. 종료** | C → S | **ChangeCipherSpec**, **Finished** | "지금부터 암호화" + 지금까지 주고받은 메시지 전체의 요약값(암호화됨) |
| | S → C | **ChangeCipherSpec**, **Finished** | 서버도 같은 확인 |
| | 양쪽 | Application Data | 이후 모든 데이터는 **대칭키로 암호화** |

#### 키가 만들어지는 과정

| 순서 | 값 | 설명 |
|---|---|---|
| ① | **프리마스터 시크릿**(Pre-master secret) | ClientKeyExchange로 양쪽이 같은 값을 갖게 됨 |
| ② | **마스터 시크릿**(Master secret) | 프리마스터 시크릿 + **클라이언트 랜덤 + 서버 랜덤**으로 계산 |
| ③ | **세션 키**(대칭키, MAC 키 등) | 마스터 시크릿에서 계산. 키 자체는 **회선으로 전송되지 않음** |

> **ClientHello의 랜덤 값**은 매 연결마다 달라서, 공격자가 예전 Handshake를 녹화했다가 다시 보내도(**재전송 공격**) 같은 키가 만들어지지 않습니다. SSL 3.0과 TLS 1.2까지는 랜덤 값의 앞 4바이트에 **현재 시각**을 넣도록 정해져 있었습니다.
{: .prompt-info }

#### 암호군(Cipher Suite) 이름 읽기

`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` 를 나누어 읽습니다.

| 부분 | 값 | 뜻 |
|---|---|---|
| 키 교환 | **ECDHE** | 타원곡선 Diffie-Hellman, 연결마다 새 임시 키 |
| 인증 | **RSA** | 서버 인증서의 RSA 키로 서명 |
| 대칭 암호 | **AES_256_GCM** | 256비트 AES, GCM 모드(암호화와 무결성을 함께 처리) |
| 해시 | **SHA384** | 키 계산 등에 쓰는 해시 |

OpenSSL은 같은 암호군을 `ECDHE-RSA-AES256-GCM-SHA384` 처럼 짧게 표시합니다.

#### RSA 키 교환과 (EC)DHE 키 교환

| 구분 | RSA 키 교환 | ECDHE 키 교환 |
|---|---|---|
| 방법 | 클라이언트가 프리마스터 시크릿을 **서버 공개키로 암호화**해 보냄 | 양쪽이 **연결마다 새 임시 키**를 만들어 Diffie-Hellman으로 같은 값을 계산 |
| 서버 개인키가 나중에 유출되면 | 녹화해 둔 **과거 통신까지 모두 복호화** 가능 | 임시 키는 이미 버려져 **과거 통신은 안전** |
| 이 성질 | 없음 | **PFS**(Perfect Forward Secrecy, 전방향 안전성) |
| TLS 1.3 | **제거됨** | 이것만 사용 |

#### TLS 1.3에서 바뀐 점

| 항목 | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Handshake 왕복 | 2번(2-RTT) | **1번(1-RTT)**. 다시 접속할 때는 0-RTT도 가능 |
| 키 교환 | RSA, DHE, ECDHE | **(EC)DHE만**(전체 Handshake는 항상 PFS) |
| 서버 인증서 | **평문**으로 전송(도청자가 볼 수 있음) | ServerHello 직후부터 **암호화**(인증서도 숨김) |
| 암호군 | 수십 가지(약한 것 포함) | **5가지**(AES-GCM, ChaCha20 등 AEAD만) |
| 압축, 재협상 | 있음 | **제거** |

### [실습 1] (Kali) `curl` 로 Handshake 순서 보기

`curl -v` 는 Handshake 메시지를 한 줄씩 보여 줍니다. 먼저 TLS 1.2로 제한해 접속합니다.

```bash
# === Kali ===
curl -sv --tls-max 1.2 -o /dev/null https://192.168.0.30/ 2>&1 | grep -E '^\* (TLS|SSL conn)'
```

```
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384 / x25519 / RSASSA-PSS
```

| 명령·옵션 | 설명 |
|---|---|
| `-v` | 자세한 과정 출력(verbose). 과정은 표준 오류로 나오므로 `2>&1` 로 합쳐 `grep` 에 넘김 |
| `-s` | 진행 상황 표시를 끔 |
| `--tls-max 1.2` | TLS 1.2까지만 쓰도록 제한(서버는 1.3도 지원하지만 일부러 1.2로 협상) |
| `grep -E '^\* (TLS...'` | `*` 로 시작하는 줄 가운데 TLS 메시지와 최종 연결 정보만 남김 |

| 화면 | 이론의 단계 |
|---|---|
| `(OUT)` / `(IN)` | Kali가 **보낸** 메시지 / **받은** 메시지 |
| `Client hello (1)` → `Server hello (2)` | 1단계 협상 |
| `Certificate (11)` → `Server key exchange (12)` → `Server finished (14)` | 2단계 서버 인증·키 교환. `Server finished` 는 **ServerHelloDone** |
| `Client key exchange (16)` | 3단계 클라이언트 키 교환 |
| `Change cipher spec` → `Finished (20)` × 2 | 4단계 종료 |
| `ECDHE-RSA-AES256-GCM-SHA384 / x25519` | 협상된 암호군. 키 교환은 ECDHE(x25519 곡선) |
| 괄호 안 숫자 | 메시지 종류 번호(ClientHello=1, ServerHello=2, Certificate=11 ...) |

이번에는 제한 없이 접속합니다(TLS 1.3).

```bash
# === Kali ===
curl -sv -o /dev/null https://192.168.0.30/ 2>&1 | grep -E '^\* (TLS|SSL conn)'
```

```
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / x25519 / RSASSA-PSS
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
```

| 화면 | 의미 |
|---|---|
| `Server key exchange`, `Client key exchange` 가 없음 | TLS 1.3은 키 교환 값을 **ClientHello·ServerHello 안에** 함께 담아 한 번 왕복으로 끝냄(1-RTT) |
| `Encrypted Extensions`, `Certificate`, `CERT verify` | ServerHello 다음 메시지들. **이미 암호화된 상태**로 옴 |
| `Newsession Ticket` | 다음에 빨리 다시 접속(세션 재사용)할 때 쓸 표 |

> `curl` 은 자기가 복호화한 결과를 보여 주므로 TLS 1.3에서도 `Certificate` 가 보입니다. **도청자에게도 보이는지**는 다음 실습에서 Wireshark로 확인합니다.
{: .prompt-info }

### [실습 2] (Kali) Wireshark로 Handshake 캡처 — 비밀번호가 숨는지 확인

Wireshark 위쪽의 파란 상어 지느러미 버튼(**Start capturing packets**)을 눌러 새 캡처를 시작합니다. 1.1절에서 잡은 패킷을 저장할지 묻는 창(**Unsaved packets...**)이 뜨면 **Continue without Saving** 을 누릅니다. 캡처 필터(`tcp port 80 or tcp port 443`)와 인터페이스(`eth0`)는 1.1절 설정이 그대로 쓰입니다. 캡처가 시작되면 디스플레이 필터에 `tls` 를 입력합니다. 1.1절과 **똑같은 아이디·비밀번호**를 이번에는 HTTPS로, TLS 1.2와 TLS 1.3 각각 한 번씩 보냅니다.

```bash
# === Kali ===
curl -s --tls-max 1.2 -u 'student:Summer2026!' -o /dev/null -w '%{http_code}\n' https://192.168.0.30/
curl -s -u 'student:Summer2026!' -o /dev/null -w '%{http_code}\n' https://192.168.0.30/
```

```
200
200
```

캡처를 멈춥니다. 패킷 목록의 Source·Destination·Protocol·Info 열은 다음과 같습니다. 앞의 9줄이 첫 번째(TLS 1.2), 뒤의 10줄이 두 번째(TLS 1.3) 접속입니다.

```
Source        Destination   Protocol  Info
192.168.0.10  192.168.0.30  TLSv1     Client Hello
192.168.0.30  192.168.0.10  TLSv1.2   Server Hello, Certificate, Server Key Exchange, Server Hello Done
192.168.0.10  192.168.0.30  TLSv1.2   Client Key Exchange, Change Cipher Spec, Encrypted Handshake Message
192.168.0.30  192.168.0.10  TLSv1.2   Change Cipher Spec, Encrypted Handshake Message
192.168.0.10  192.168.0.30  TLSv1.2   Application Data
192.168.0.30  192.168.0.10  TLSv1.2   Application Data
192.168.0.30  192.168.0.10  TLSv1.2   Application Data
192.168.0.10  192.168.0.30  TLSv1.2   Encrypted Alert
192.168.0.30  192.168.0.10  TLSv1.2   Encrypted Alert
192.168.0.10  192.168.0.30  TLSv1.2   Client Hello
192.168.0.30  192.168.0.10  TLSv1.3   Server Hello, Change Cipher Spec, Application Data, Application Data, Application Data, Application Data
192.168.0.10  192.168.0.30  TLSv1.3   Change Cipher Spec, Application Data
192.168.0.10  192.168.0.30  TLSv1.3   Application Data
192.168.0.30  192.168.0.10  TLSv1.3   Application Data
192.168.0.30  192.168.0.10  TLSv1.3   Application Data
192.168.0.30  192.168.0.10  TLSv1.3   Application Data
192.168.0.30  192.168.0.10  TLSv1.3   Application Data
192.168.0.10  192.168.0.30  TLSv1.3   Application Data
192.168.0.30  192.168.0.10  TLSv1.3   Application Data
```

> 📷 **캡처**: `tls` 필터를 적용한 Wireshark 패킷 목록.

> 여러 메시지가 한 패킷에 함께 실리기도 하고 나뉘기도 하므로, 줄 수와 묶음은 환경에 따라 조금 다를 수 있습니다. Handshake 메시지만 보려면 디스플레이 필터를 `tls.handshake` 로 바꿉니다.
{: .prompt-info }

| 화면 | 의미 |
|---|---|
| 첫 `Client Hello` 의 Protocol이 `TLSv1` | 옛 장비와 호환되도록 **레코드 헤더**에는 낮은 버전 번호를 적음. 실제로 원하는 최고 버전은 메시지 안에 따로 있음 |
| `Server Hello, Certificate, Server Key Exchange, Server Hello Done` | TLS 1.2의 2단계 메시지가 **평문으로** 보임 |
| `Encrypted Handshake Message` | `Finished` 메시지. ChangeCipherSpec 다음이라 이미 암호화됨 |
| `Application Data` | **암호화된 HTTP 요청·응답**. 비밀번호는 이 안에 있음 |
| `Encrypted Alert` | 연결을 끝낸다는 **Alert**(close_notify). 이것도 암호화됨 |
| TLS 1.3의 `Server Hello` 다음이 모두 `Application Data` | 인증서까지 **암호화되어** Wireshark도 Certificate인지 알 수 없음 |

TLS 1.2의 `Server Hello, Certificate, ...` 패킷을 클릭하고 **Transport Layer Security** → **Certificate** 를 펼치면 서버 인증서의 내용이 보입니다.

> 📷 **캡처**: TLS 1.2 Certificate 메시지에서 `subject: ... ubuntu.lab.local`, `issuer: ... Lab Root CA` 가 보이는 상세 창.

마지막으로 **비밀번호가 보이는지** 확인합니다. 아무 TLS 패킷이나 오른쪽 클릭 → **Follow → TCP Stream** 을 고르고, 1.1절의 HTTP 화면과 비교합니다.

| 확인할 것 | HTTP(1.1절) | HTTPS(TLS 1.2) | HTTPS(TLS 1.3) |
|---|:---:|:---:|:---:|
| `GET / HTTP/1.1`, `Authorization` 헤더 | **보임** | 안 보임 | 안 보임 |
| `student:Summer2026!` | **보임**(`Credentials`) | 안 보임 | 안 보임 |
| 서버 인증서의 이름(`ubuntu.lab.local`, `LabOrg`) | — | **보임** | 안 보임 |
| 웹 페이지 내용(HTML) | **보임** | 안 보임 | 안 보임 |

Follow TCP Stream 창은 HTTPS의 경우 대부분 **점(`.`)과 알 수 없는 기호**로 채워집니다. TLS 1.2에서는 그 사이에 인증서 속 `LabOrg`, `ubuntu.lab.local` 같은 글자가 섞여 보이고, TLS 1.3에서는 그마저 보이지 않습니다.

> 📷 **캡처**: HTTPS 접속의 Follow TCP Stream 창(암호문). 1.1절 HTTP의 Follow 화면과 나란히.

> **결론**: 같은 서버, 같은 비밀번호라도 **HTTP는 도청만으로 비밀번호가 노출**되고, **HTTPS는 Application Data로 암호화**되어 보호됩니다. 로그인·결제·개인정보 입력 페이지는 반드시 HTTPS여야 하며, 가능하면 인증서까지 숨기는 **TLS 1.3** 을 씁니다.
{: .prompt-tip }

---

# 확인 문제

> 1. 계층과 보안 프로토콜의 연결이 **틀린** 것은?
>    ① 네트워크 계층 – IPSec
>    ② 응용 계층과 전송 계층 사이 – SSL/TLS
>    ③ 응용 계층 – S-HTTP
>    ④ 네트워크 계층 – PPTP
>
> 2. SSL에 대한 설명으로 **틀린** 것은?
>    ① 넷스케이프사가 웹 통신 보호를 위해 개발하였다
>    ② TCP/IP의 IP 계층에서 보안 기능을 수행한다
>    ③ TLS 1.0은 SSL 3.0을 바탕으로 IETF가 표준화하였다
>    ④ Handshake, ChangeCipherSpec, Alert, Record 프로토콜로 구성된다
>
> 3. SSL Record 프로토콜이 데이터를 처리하는 순서로 옳은 것은?
>    ① 압축 → 단편화 → 암호화 → MAC 추가 → 헤더 추가
>    ② 단편화 → 압축 → 암호화 → MAC 추가 → 헤더 추가
>    ③ 압축 → 단편화 → MAC 추가 → 암호화 → 헤더 추가
>    ④ 단편화 → 압축 → MAC 추가 → 암호화 → 헤더 추가
>
> 4. SSL의 각 계층에 대한 설명으로 **틀린** 것은?
>    ① Record 계층은 상위 메시지를 캡슐화한다
>    ② Record 계층에서 사용할 암호 알고리즘을 선택한다
>    ③ Handshake 계층에서 암호화 키를 계산할 재료를 교환한다
>    ④ Handshake 계층에서 서버와 클라이언트를 인증한다
>
> 5. HTTPS의 특징으로 **틀린** 것은?
>    ① 데이터를 SSL/TLS로 암호화하여 주고받는다
>    ② 중간에서 공격자가 내용을 엿보기 어렵다
>    ③ HTTP보다 처리 속도가 빨라진다
>    ④ 전송 중 데이터가 변경되지 않았는지 확인할 수 있다
>
> 6. TLS 1.2 Handshake 메시지를 순서대로 나열한 것은?
>    가. Server Key Exchange  나. Server Hello  다. Client Hello  라. Client Key Exchange  마. Change Cipher Spec
>    ① 다 → 라 → 나 → 가 → 마
>    ② 다 → 나 → 가 → 라 → 마
>    ③ 다 → 나 → 라 → 마 → 가
>    ④ 다 → 라 → 가 → 나 → 마
>
> 7. 다음 Handshake 과정을 순서대로 나열한 것은?
>    (A) 클라이언트가 프리마스터 시크릿을 서버 공개키로 암호화해 보낸다
>    (B) 서버가 프리마스터 시크릿으로 마스터 시크릿과 세션 키를 만든다
>    (C) 클라이언트가 세션 키로 암호 통신을 시작함을 알리고 Handshake를 마친다
>    (D) 클라이언트가 버전, 지원 암호군, 랜덤 값을 보낸다
>    (E) 서버가 버전, 선택한 암호군, 랜덤 값, 서버 인증서를 보낸다
>    (F) 서버가 세션 키로 암호 통신을 시작함을 알리고 Handshake를 마친다
>
> 8. ClientHello 메시지의 랜덤 값을 만들 때 시간 정보를 포함하도록 한 이유는?
>    ① 재전송 공격을 막기 위해
>    ② 시간으로 동기화를 하기 위해
>    ③ 협상 시간을 서버에 알리기 위해
>    ④ 특별한 의미 없음
>
> 9. SSL/TLS에 대한 설명으로 옳은 것은?
>    ① SSL/TLS를 쓰는 기업은 신뢰할 수 있으므로 신용카드 번호를 보내도 된다
>    ② 서버 인증서가 오므로 클라이언트는 아무 공개키가 없어도 서버를 인증할 수 있다
>    ③ TLS 1.3을 사용하여 통신의 기밀성을 확보할 수 있다
>    ④ 통신 전·중·후의 데이터를 모두 보호해 준다
>
> 10. SSL Record 프로토콜에서 데이터 암호화에 사용하는 대칭키 알고리즘이 **아닌** 것은?
>    ① IDEA
>    ② RC4-40
>    ③ DES-40
>    ④ WPKI
>
> 11. CA에게서 철수의 공개키를 받았을 때, 그 공개키가 가짜가 아님을 확신할 수 있는 이유는?
>    ① 철수의 인증서가 위조할 수 없는 CA의 개인키로 서명되어 있기 때문
>    ② 철수의 인증서가 인증기관의 검증을 거쳤기 때문
>    ③ 철수의 인증서가 세션 키로 암호화되어 있기 때문
>    ④ 철수의 인증서가 신뢰받는 센터를 통해 전달되었기 때문
>
> 12. 다음 중 OCSP 서버의 응답 상태가 **아닌** 것은?
>    ① good
>    ② revoked
>    ③ unknown
>    ④ bad
>
> 13. CRL(인증서 폐지 목록)에 들어 있는 정보는?
>    ① 만료된 인증서의 공개키
>    ② 유효기간 안에 폐지된 인증서의 일련번호
>    ③ 유효기간 안에 폐지된 인증서의 공개키
>    ④ 새로 발급된 인증서의 일련번호
{: .prompt-tip }

<details>
<summary>정답 보기</summary>

1. <strong>④</strong> PPTP는 데이터링크(2계층) 터널링 프로토콜이다(1.2절).<br>
2. <strong>②</strong> SSL은 IP 계층이 아니라 응용 계층과 전송 계층 사이에서 동작한다. TLS 1.0은 "SSL 3.1"이라고도 한다(1.2절).<br>
3. <strong>④</strong> 단편화 → 압축 → MAC → 암호화 → 헤더(1.4절).<br>
4. <strong>②</strong> 암호 알고리즘은 Handshake가 협상해 고른다. Record는 정해진 대로 실행만 한다(1.4절).<br>
5. <strong>③</strong> 암호화·복호화와 Handshake가 더해져 HTTP보다 조금 느리다(1.3절).<br>
6. <strong>②</strong> Client Hello → Server Hello → Server Key Exchange → Client Key Exchange → Change Cipher Spec. 2.5절 실습 1의 `curl -v` 화면과 같은 순서다.<br>
7. <strong>D → E → A → B → C → F</strong> (협상 → 서버 인증서 → 키 교환 → 키 계산 → 클라이언트 종료 → 서버 종료)<br>
8. <strong>①</strong> 매번 달라지는 랜덤 값 때문에 녹화한 Handshake를 다시 보내도 같은 키가 나오지 않는다(2.5절).<br>
9. <strong>③</strong> ①은 상점에 카드번호가 노출되는 문제(07장 SET), ②는 CA의 공개키(루트 인증서)가 미리 있어야 하고, ④는 전송 구간만 보호한다(1.3절).<br>
10. <strong>④</strong> WPKI는 무선 환경의 공개키 기반 구조(체계)이지 암호 알고리즘이 아니다(1.2절).<br>
11. <strong>①</strong> (2.1절)<br>
12. <strong>④</strong> OCSP 응답은 good·revoked·unknown 세 가지다(2.1절).<br>
13. <strong>②</strong> (2.1절)

</details>

---

# 체크리스트

- [ ] (Kali) HTTP 기본 인증 요청을 Wireshark로 잡아 `Credentials: student:Summer2026!` 를 확인하고, Base64가 암호화가 아님을 설명 가능
- [ ] SSL과 TLS의 관계(TLS 1.0 = SSL 3.1), 현재 쓰는 버전(TLS 1.2·1.3)을 설명 가능
- [ ] 계층별 보안 프로토콜(S-HTTP·PGP·SET / SSL·TLS / IPSec / PPTP·L2TP)과 TLS 서비스 포트(443·465·993·995·636·990·853)를 구분 가능
- [ ] SSL/TLS의 4개 프로토콜과 Record 처리 순서(단편화→압축→MAC→암호화→헤더)를 설명 가능
- [ ] (Ubuntu) OpenSSL로 실습용 루트 CA를 만들고, CSR을 서명해 SAN이 있는 서버 인증서를 발급, `openssl verify` 에서 `OK` 확인
- [ ] (Ubuntu) Apache에 인증서를 연결해 443 포트가 열린 것을 확인
- [ ] (Kali) CA 설치 전 `unable to get local issuer certificate`, 설치 후 `Verification: OK` 와 Firefox 자물쇠 확인
- [ ] CA 인증서를 믿을 수 있는 이유(CA 개인키 서명, 루트 인증서 사전 설치)와 CRL·OCSP의 차이를 설명 가능
- [ ] (Kali) `curl -v` 와 Wireshark에서 TLS 1.2 Handshake 순서를 찾고, TLS 1.3에서는 인증서까지 암호화되는 것을 확인
- [ ] (Kali) HTTPS에서는 비밀번호가 `Application Data` 안에 숨는 것을 HTTP와 비교해 확인
- [ ] RSA 키 교환과 ECDHE 키 교환의 차이(PFS)를 설명 가능

> 다음 글 **06-2** 에서는 SSL/TLS를 노리는 공격(중간자 공격, 다운그레이드, 약한 암호, Heartbleed, SSL Strip)을 정리하고, 이 HTTPS 서버를 점검·강화합니다. 이 글에서 만든 서버와 Kali의 CA 설치 상태를 **그대로 두세요.**
{: .prompt-info }
