---
title: "[애플리케이션 보안] 06-2. SSL/TLS 공격 유형과 보안 대책"
date: 2026-10-22 10:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - SSL-TLS보안
tags:
  - MITM
  - Downgrade
  - POODLE
  - Heartbleed
  - SSLStrip
  - HSTS
  - PFS
  - nmap
math: false
mermaid: false
---

> **06-1** 에서 만든 HTTPS 서버(Apache, 실습용 CA가 발급한 인증서)가 동작 중이고, Kali에 **Lab Root CA 인증서가 설치**되어 있어야 합니다.
{: .prompt-info }

## 0. 실습 환경

| 장비 | IP | 역할 |
|---|---|---|
| Kali | **192.168.0.10** | 공격자·점검자, 그리고 피해자 역할의 클라이언트 |
| Ubuntu | **192.168.0.30** | HTTPS 웹 서버(Apache) |

> 명령과 예상 화면은 **Ubuntu 24.04.4 LTS(Apache 2.4.58, OpenSSL 3.0.13)** 와 **Kali Rolling(OpenSSL 3.6.2, curl 8.21.0, Nmap 7.99)** 에서 실행해 확인했습니다. 날짜·시각, 인증서 지문, 웹 응답의 `ETag` 값, 오류 메시지 앞의 16진수 번호(`40E7A4C5...`)는 실행할 때마다 다릅니다.
{: .prompt-warning }

이 글의 실습도 04-2처럼 **취약한 상태를 먼저 보고, 설정을 바꿔 막히는 것을 확인**하는 순서입니다.

| 절 | Kali에서 할 일 | Ubuntu에서 할 일 |
|---|---|---|
| 3.2 중간자 공격 | 가짜 인증서 서버를 띄우고, 인증서 검증 → 차단 / 경고 무시 → 비밀번호 탈취 | — |
| 3.3 다운그레이드 | 옛 버전(TLS 1.0·1.1)으로 접속 시도 → 거부 확인 | — |
| 3.4 약한 암호 | 암호군 점검(nmap), RSA 키 교환 접속: 허용(약함) → 거부(안전) | 암호군·프로토콜 강화 설정 |
| 3.5 Heartbleed | 알려진 취약점 점검 | OpenSSL 버전 확인 |
| 3.6 SSL Strip | HTTP 접속이 HTTPS로 넘어가고 HSTS가 붙는지 확인 | HTTP→HTTPS 이동, HSTS 설정 |

---

# 3. SSL/TLS 공격 유형

## 3.1 공격 유형 한눈에 보기

### [이론]

SSL/TLS는 잘 만든 프로토콜이지만, **설정·사용자·구현** 가운데 약한 곳을 노리는 공격이 있습니다.

| 노리는 곳 | 공격 | 원리 | 대책 |
|---|---|---|---|
| **인증서·사용자** | **중간자 공격**(MITM) | 가짜 인증서로 서버인 척 끼어듦. 사용자가 **경고를 무시**하면 성공 | 공인·사내 CA 인증서, 경고 무시 금지, CA 개인키 보호 |
| **프로토콜 버전** | **다운그레이드**(POODLE), 옛 버전 약점(DROWN, BEAST) | 약한 **옛 버전**(SSL 2.0·3.0, TLS 1.0)으로 협상하게 끌어내리거나, 켜져 있는 옛 버전을 이용해 깨뜨림 | SSL 2.0·3.0, TLS 1.0·1.1 끄기 |
| **암호 알고리즘** | 약한 암호군 공격(FREAK, Logjam, BEAST, Sweet32, RC4 등) | 짧은 키(수출용 40·512비트), 깨진 알고리즘(RC4, DES·3DES), 약한 해시(MD5·SHA-1) | 강한 암호군만 허용, PFS(ECDHE) |
| **압축 기능** | **CRIME** | **TLS 압축** 후 길이의 변화로 쿠키 같은 비밀값을 한 글자씩 추측 | TLS 압축 끄기 |
| **압축 기능** | BREACH | 같은 원리를 **HTTP 응답 압축**(gzip)에 적용. TLS 압축을 꺼도 성공 | 비밀값이 든 페이지의 HTTP 압축 끄기, CSRF 토큰을 요청마다 바꾸기 |
| **구현 버그** | **Heartbleed** | OpenSSL의 버그로 서버 **메모리**가 새어 나감 | 라이브러리 업데이트, 키·인증서 재발급 |
| **HTTPS 우회** | **SSL Strip** | 처음 HTTP 접속을 가로채 HTTPS로 **올라가지 못하게** 함 | HTTP→HTTPS 이동, **HSTS** |

---

## 3.2 인증서 경고 무시와 중간자 공격(MITM)

### [이론]

**중간자 공격(Man-In-The-Middle)** 은 공격자가 클라이언트와 서버 사이에 끼어들어, 클라이언트에게는 서버인 척, 서버에게는 클라이언트인 척하는 공격입니다. 끼어드는 방법은 이미 배운 것들입니다.

| 끼어드는 방법 | 배운 곳 |
|---|---|
| ARP 스푸핑으로 같은 네트워크의 트래픽을 공격자 PC로 돌림 | 네트워크 보안 |
| hosts 파일 변조·DNS 캐시 포이즈닝으로 가짜 IP를 알려 줌 | 04-2 3.4·3.5절 |
| 가짜 무선 AP(Evil Twin)에 접속하게 함 | 무선 보안 |

트래픽을 가로챈 공격자는 진짜 서버의 **개인키가 없으므로** 진짜 인증서로 Handshake를 할 수 없습니다. 그래서 **자기가 만든 가짜 인증서**를 보냅니다.

| 클라이언트의 반응 | 결과 |
|---|---|
| 인증서를 검증함 → 믿는 CA의 서명이 없음 → **연결 거부** | 공격 실패 |
| 경고를 **무시하고 계속**(브라우저의 "위험을 감수하고 계속", `curl -k`) | 공격자와 암호화 연결 → 공격자가 **복호화해서 모두 봄** |

> TLS의 암호화는 **"상대가 진짜일 때"** 만 의미가 있습니다. 가짜 서버와도 암호화는 잘 됩니다. 인증서 경고를 무시하는 습관이 가장 위험한 이유입니다.
{: .prompt-danger }

### [실습 1] (Kali) 가짜 인증서 만들기 — 이름은 똑같이

공격자 입장에서 진짜와 **이름(주체)과 주소(SAN)가 똑같은** 인증서를 만듭니다. 다만 공격자는 Lab Root CA의 개인키가 없으므로 **자체서명**으로 만들 수밖에 없습니다.

```bash
# === Kali ===
cd ~
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout fake.key -out fake.crt \
  -subj "/C=KR/O=LabOrg/CN=ubuntu.lab.local" \
  -addext "subjectAltName=DNS:ubuntu.lab.local,IP:192.168.0.30"
```

키를 만드는 점(`.`)과 `+` 줄이 나오고 `-----` 로 끝납니다(06-1 2.2절과 같음).

| 명령·옵션 | 설명 |
|---|---|
| `-x509` | 자체서명 인증서를 바로 만듦 |
| `-addext "subjectAltName=..."` | SAN 확장을 명령에서 바로 추가(06-1에서는 파일로 넣었음) |

진짜 서버가 보내는 인증서와 가짜 인증서를 비교합니다.

```bash
# === Kali ===
openssl s_client -connect 192.168.0.30:443 < /dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -fingerprint -sha256
openssl x509 -in fake.crt -noout -subject -issuer -fingerprint -sha256
```

```
subject=C=KR, O=LabOrg, CN=ubuntu.lab.local
issuer=C=KR, O=LabOrg, CN=Lab Root CA
sha256 Fingerprint=E4:17:63:9A:D3:1E:77:7D:A7:E2:62:1C:D3:00:26:A7:82:5A:46:78:CB:3A:53:62:C3:8F:11:3C:CD:AF:3E:CC
subject=C=KR, O=LabOrg, CN=ubuntu.lab.local
issuer=C=KR, O=LabOrg, CN=ubuntu.lab.local
sha256 Fingerprint=48:70:83:96:91:BF:34:A0:85:B0:F6:A3:B3:4C:89:6B:B7:E6:FB:83:71:88:69:C9:D7:BB:C6:12:C4:30:F8:F3
```

| 명령·옵션 | 설명 |
|---|---|
| `openssl s_client ... 2>/dev/null` | 서버와 Handshake를 하고 받은 인증서를 출력. 접속 과정 메시지는 버림 |
| 세로 막대 뒤의 `openssl x509 -noout ...` | 앞 명령이 넘겨준 인증서에서 주체·발급자·지문만 뽑아냄 |

| 비교 | 진짜(서버) | 가짜(공격자) |
|---|---|---|
| `subject` | `CN=ubuntu.lab.local` | `CN=ubuntu.lab.local` — **똑같음** |
| `issuer` | `Lab Root CA` | `ubuntu.lab.local` — **자체서명** |
| 지문 | `E4:17:...` | `48:70:...` — **다름** |

> 이름은 누구나 똑같이 적을 수 있습니다. 진짜와 가짜를 가르는 것은 **믿는 CA의 서명이 있는가** 입니다.
{: .prompt-tip }

### [실습 2] (Kali) 가짜 서버 띄우고 피해자 접속 — 인증서를 검증하면 차단

Kali에서 터미널 창을 **두 개** 엽니다. **터미널 A** 는 공격자, **터미널 B** 는 피해자입니다.

터미널 A에서 가짜 인증서로 TLS 서버를 띄웁니다. 이 서버는 받은 데이터를 **복호화해서 화면에 출력**합니다.

```bash
# === Kali (Terminal A: attacker) ===
cd ~
openssl s_server -quiet -accept 4433 -cert fake.crt -key fake.key
```

출력 없이 대기합니다. 터미널 B에서 피해자가 `https://192.168.0.30/` 에 로그인 정보를 보내는데, 공격자가 이미 트래픽을 가로챘다고 가정합니다.

```bash
# === Kali (Terminal B: victim) ===
curl --connect-to 192.168.0.30:443:192.168.0.10:4433 -u 'student:Summer2026!' https://192.168.0.30/
```

```
curl: (60) SSL certificate OpenSSL verify result: self-signed certificate (18)
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the webpage mentioned above.
```

| 명령·옵션 | 설명 |
|---|---|
| `openssl s_server` | 시험용 TLS 서버 |
| `-quiet` | 접속 정보는 줄이고, 받은 데이터만 출력 |
| `-accept 4433` | 4433번 포트에서 대기 |
| `-cert fake.crt -key fake.key` | 가짜 인증서와 그 개인키 사용 |
| `--connect-to 192.168.0.30:443:192.168.0.10:4433` | 주소창(URL)은 `192.168.0.30:443` 그대로 두고, 실제 연결만 `192.168.0.10:4433`(공격자)으로 보냄. ARP 스푸핑·DNS 변조로 **트래픽을 가로챈 상황을 흉내** 냄 |

| 화면 | 의미 |
|---|---|
| `self-signed certificate (18)` | 받은 인증서가 Lab Root CA가 아닌 **자체서명**이라 검증 실패 |
| `could not establish a secure connection` | `curl` 이 **연결을 끊음** → 비밀번호를 보내지 않음. **공격 실패** |

> 터미널 A에는 연결이 끊겼다는 `unexpected eof while reading` 같은 오류 줄이 나올 수 있습니다. 비밀번호는 나오지 않습니다.
{: .prompt-info }

### [실습 3] (Kali) 경고를 무시하면 — 비밀번호 탈취

이번에는 피해자가 경고를 무시합니다. `curl` 의 `-k` 는 브라우저의 **Accept the Risk and Continue** 와 같습니다.

```bash
# === Kali (Terminal B: victim) ===
curl -k --max-time 3 --connect-to 192.168.0.30:443:192.168.0.10:4433 -u 'student:Summer2026!' https://192.168.0.30/
```

```
curl: (28) Operation timed out after 3000 milliseconds with 0 bytes received
```

| 명령·옵션 | 설명 |
|---|---|
| `-k` (`--insecure`) | 인증서 검증을 **하지 않음** |
| `--max-time 3` | 3초 안에 끝나지 않으면 포기. 가짜 서버는 응답을 보내지 않으므로 기다리지 않게 함 |

피해자 쪽에서는 "응답이 없다"로만 보입니다. 터미널 A(공격자)를 봅니다.

```
GET / HTTP/1.1
Host: 192.168.0.30
Authorization: Basic c3R1ZGVudDpTdW1tZXIyMDI2IQ==
User-Agent: curl/8.21.0
Accept: */*

4017439511790000:error:0A000126:SSL routines::unexpected eof while reading:../ssl/record/rec_layer_s3.c:698:
```

> 📷 **캡처**: 터미널 A에 피해자의 `Authorization` 헤더가 복호화되어 출력된 화면.

| 화면 | 의미 |
|---|---|
| 마지막 `unexpected eof while reading` | 피해자가 3초 만에 포기하고 연결을 끊었다는 뜻. 그 전에 요청은 이미 공격자에게 넘어감 |
| `GET / HTTP/1.1 ...` | 피해자의 HTTPS 요청이 **공격자 손에서 복호화됨** |
| `Authorization: Basic c3R1...` | 06-1 1.1절처럼 `base64 -d` 하면 `student:Summer2026!` |

실습이 끝나면 터미널 A에서 `Ctrl + C` 로 가짜 서버를 끄고, 가짜 키를 지웁니다.

```bash
# === Kali ===
rm -f ~/fake.key ~/fake.crt
```

출력 없음.

#### 정리

| 피해자의 행동 | 결과 |
|---|---|
| 인증서를 검증(기본값) | 가짜 인증서 거부 → **안전** |
| 경고 무시(`-k`, "위험을 감수하고 계속") | 공격자가 **TLS를 풀어 비밀번호를 읽음** |

| 대책 | 내용 |
|---|---|
| 인증서 경고를 **절대 무시하지 않기** | 사용자 교육. 경고가 뜨면 접속을 멈추고 관리자에게 알림 |
| 공인 CA·사내 CA 인증서 사용 | 정상 사이트에서 평소 경고가 뜨지 않아야, 사용자가 경고를 "이상한 일"로 인식함 |
| **CA 개인키 보호** | 공격자가 `ca.key` 를 훔치면 **경고 없는 가짜 인증서**를 만들 수 있음. 실제로 CA가 해킹되어 가짜 인증서가 발급된 사고(2011년 DigiNotar)가 있었고, 그 CA는 모든 브라우저에서 퇴출됨 |
| 앱은 인증서 검증을 끄지 않기 | 개발 중 `-k`, `verify=False` 처럼 검증을 끈 코드를 그대로 배포하면 모든 사용자가 MITM에 노출 |

---

## 3.3 다운그레이드 공격과 옛 버전

### [이론]

클라이언트와 서버는 Handshake에서 **둘 다 지원하는 가장 높은 버전**을 고릅니다. 공격자가 중간에서 ClientHello를 바꾸거나 연결을 일부러 끊어 클라이언트가 낮은 버전으로 다시 시도하게 만들면, **약한 옛 버전**으로 통신하게 할 수 있습니다. 이것이 **다운그레이드 공격**입니다.

| 공격 | 노리는 것 | 내용 |
|---|---|---|
| **POODLE**(2014) | **SSL 3.0** | SSL 3.0으로 끌어내린 뒤, CBC 모드 **패딩** 검사의 허점으로 쿠키를 한 바이트씩 알아냄 |
| **DROWN**(2016) | **SSL 2.0** | 같은 키를 쓰는 서버가 SSL 2.0을 켜 두면, 그것을 이용해 TLS 통신까지 복호화. 연결을 끌어내리는 대신 **다른 프로토콜(SSL 2.0)을 이용**하는 교차 프로토콜 공격 |
| **FREAK**, **Logjam**(2015) | 수출용 약한 암호 | 512비트 RSA·DH 같은 **수출용(EXPORT) 암호군**으로 끌어내린 뒤 키를 계산해 냄 |
| **BEAST**(2011) | TLS 1.0 | TLS 1.0의 CBC 모드 초기화 벡터(IV) 약점으로 쿠키를 알아냄 |

| 대책 | 내용 |
|---|---|
| **옛 버전 끄기** | SSL 2.0·3.0은 물론 TLS 1.0·1.1도 끄고 **TLS 1.2·1.3만** 허용 |
| 다운그레이드 탐지 | TLS 1.3 서버는 1.2 이하로 협상하게 되면 ServerHello의 랜덤 값 끝에 `DOWNGRD` 라는 표시를 넣어, 1.3을 아는 클라이언트가 공격을 알아채게 함 |

### [실습] (Kali) 버전별로 접속해 보기

`openssl s_client` 에 버전을 하나씩 지정해 접속합니다.

```bash
# === Kali ===
openssl s_client -connect 192.168.0.30:443 -brief -tls1 < /dev/null
openssl s_client -connect 192.168.0.30:443 -brief -tls1_1 < /dev/null
```

```
Connecting to 192.168.0.30
40E7A4C56F720000:error:0A00042E:SSL routines:ssl3_read_bytes:tlsv1 alert protocol version:../ssl/record/rec_layer_s3.c:918:SSL alert number 70
40E7A4C56F720000:error:0A000197:SSL routines:SSL_shutdown:shutdown while in init:../ssl/ssl_lib.c:2804:
Connecting to 192.168.0.30
40F7A30634740000:error:0A00042E:SSL routines:ssl3_read_bytes:tlsv1 alert protocol version:../ssl/record/rec_layer_s3.c:918:SSL alert number 70
40F7A30634740000:error:0A000197:SSL routines:SSL_shutdown:shutdown while in init:../ssl/ssl_lib.c:2804:
```

```bash
# === Kali ===
openssl s_client -connect 192.168.0.30:443 -brief -tls1_2 < /dev/null
openssl s_client -connect 192.168.0.30:443 -brief -tls1_3 < /dev/null
```

```
Connecting to 192.168.0.30
Can't use SSL_get_servername
CONNECTION ESTABLISHED
Protocol version: TLSv1.2
Ciphersuite: ECDHE-RSA-AES256-GCM-SHA384
Peer certificate: C=KR, O=LabOrg, CN=ubuntu.lab.local
Hash used: SHA256
Signature type: rsa_pss_rsae_sha256
Verification: OK
Supported Elliptic Curve Point Formats: uncompressed:ansiX962_compressed_prime:ansiX962_compressed_char2
Peer Temp Key: X25519, 253 bits
DONE
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
| `-tls1` / `-tls1_1` / `-tls1_2` / `-tls1_3` | 그 버전**만** 써서 접속 |

| 화면 | 의미 |
|---|---|
| `tlsv1 alert protocol version` ... `SSL alert number 70` | 서버가 **Alert 프로토콜**(06-1 1.4절)로 "그 버전은 안 받는다(protocol_version, 70번)"고 응답 → **TLS 1.0·1.1 거부** |
| `Protocol version: TLSv1.2` / `TLSv1.3` | 1.2와 1.3만 연결됨 |

> Ubuntu 24.04의 Apache 설정에는 `SSLProtocol all -SSLv3` 라고 되어 있지만, 그 아래의 OpenSSL 3이 보안 수준(security level 2)에 따라 TLS 1.0·1.1을 막습니다. 다음 절에서는 이 기본값에 기대지 않고, 허용할 버전을 **설정 파일에 직접** 적습니다.
{: .prompt-info }

---

## 3.4 약한 암호군과 키 교환

### [이론]

| 약한 요소 | 문제 | 예 |
|---|---|---|
| 짧은 키 | 계산으로 키를 찾아낼 수 있음 | **수출용 40비트**(DES-40, RC4-40), 512비트 RSA·DH |
| 깨진 대칭 암호 | 통계적 약점, 블록이 작아 충돌 | **RC4**, **DES**, **3DES**(Sweet32 공격) |
| 약한 해시 | 충돌을 만들 수 있음 | **MD5**, SHA-1 |
| CBC 모드 | 패딩·IV 약점(POODLE, BEAST, Lucky13) | `..._CBC_SHA` 암호군 |
| **RSA 키 교환** | **PFS 없음**: 서버 개인키가 유출되면 녹화해 둔 과거 통신까지 복호화 | `TLS_RSA_WITH_...` 암호군 |
| TLS 압축 | **CRIME**: 압축 후 길이 변화로 비밀값 추측 | 압축 사용 |

> 안전한 조합은 **키 교환 ECDHE(PFS) + 대칭 암호 AES-GCM 또는 ChaCha20(AEAD) + SHA-256 이상**입니다. TLS 1.3은 대칭 암호로 AEAD(AES-GCM·AES-CCM·ChaCha20)만 허용하고, 전체 Handshake에서는 항상 (EC)DHE로 키를 교환합니다.
{: .prompt-tip }

### [실습 1] (Kali) 서버의 암호군 점검 — 기본 설정

`nmap` 의 `ssl-enum-ciphers` 스크립트는 서버가 허용하는 TLS 버전과 암호군을 모두 시험해 보고, 강도를 A~F로 표시합니다.

```bash
# === Kali ===
nmap -Pn -p 443 --script ssl-enum-ciphers 192.168.0.30
```

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-16 17:01 +0900
Nmap scan report for 192.168.0.30
Host is up (0.000049s latency).

PORT    STATE SERVICE
443/tcp open  https
| ssl-enum-ciphers: 
|   TLSv1.2: 
|     ciphers: 
|       TLS_DHE_RSA_WITH_AES_128_CBC_SHA (dh 2048) - A
|       TLS_DHE_RSA_WITH_AES_128_CBC_SHA256 (dh 2048) - A
|       TLS_DHE_RSA_WITH_AES_128_CCM (dh 2048) - A
|       TLS_DHE_RSA_WITH_AES_128_CCM_8 (dh 2048) - A
|       TLS_DHE_RSA_WITH_AES_128_GCM_SHA256 (dh 2048) - A
|       TLS_DHE_RSA_WITH_AES_256_CBC_SHA (dh 2048) - A
|       TLS_DHE_RSA_WITH_AES_256_CBC_SHA256 (dh 2048) - A
|       TLS_DHE_RSA_WITH_AES_256_CCM (dh 2048) - A
|       TLS_DHE_RSA_WITH_AES_256_CCM_8 (dh 2048) - A
|       TLS_DHE_RSA_WITH_AES_256_GCM_SHA384 (dh 2048) - A
|       TLS_DHE_RSA_WITH_ARIA_128_GCM_SHA256 (dh 2048) - A
|       TLS_DHE_RSA_WITH_ARIA_256_GCM_SHA384 (dh 2048) - A
|       TLS_DHE_RSA_WITH_CAMELLIA_128_CBC_SHA (dh 2048) - A
|       TLS_DHE_RSA_WITH_CAMELLIA_128_CBC_SHA256 (dh 2048) - A
|       TLS_DHE_RSA_WITH_CAMELLIA_256_CBC_SHA (dh 2048) - A
|       TLS_DHE_RSA_WITH_CAMELLIA_256_CBC_SHA256 (dh 2048) - A
|       TLS_DHE_RSA_WITH_CHACHA20_POLY1305_SHA256 (dh 2048) - A
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_ARIA_128_GCM_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_ARIA_256_GCM_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CAMELLIA_128_CBC_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CAMELLIA_256_CBC_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256 (secp256r1) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_CCM (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_CCM_8 (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_GCM_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CCM (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CCM_8 (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_GCM_SHA384 (rsa 2048) - A
|       TLS_RSA_WITH_ARIA_128_GCM_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_ARIA_256_GCM_SHA384 (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA256 (rsa 2048) - A
|     compressors: 
|       NULL
|     cipher preference: client
|   TLSv1.3: 
|     ciphers: 
|       TLS_AKE_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
|       TLS_AKE_WITH_AES_256_GCM_SHA384 (ecdh_x25519) - A
|       TLS_AKE_WITH_CHACHA20_POLY1305_SHA256 (ecdh_x25519) - A
|     cipher preference: client
|_  least strength: A

Nmap done: 1 IP address (1 host up) scanned in 0.74 seconds
```

| 명령·옵션 | 설명 |
|---|---|
| `-Pn` | ping으로 살아 있는지 확인하는 단계를 건너뜀 |
| `-p 443` | 443번 포트만 검사 |
| `--script ssl-enum-ciphers` | TLS 버전과 암호군을 하나씩 시도해 목록과 강도를 표시하는 스크립트 |

| 화면 | 의미 |
|---|---|
| `TLSv1.2:` 와 `TLSv1.3:` 만 있음 | SSL 3.0, TLS 1.0·1.1은 받지 않음(3.3절 결과와 같음) |
| TLS 1.2 암호군 44개 | 기본 설정은 **많은 암호군**을 허용 |
| `TLS_RSA_WITH_...` 16개 | **RSA 키 교환** 암호군. 등급은 A지만 **PFS가 없음** |
| `..._CBC_...` | CBC 모드 암호군. 옛 공격(BEAST, Lucky13)의 대상이 되었던 방식 |
| `compressors: NULL` | **TLS 압축을 쓰지 않음** → CRIME 공격 대상 아님 |
| `cipher preference: client` | 암호군을 **클라이언트가 원하는 순서**로 고름. 클라이언트가 약한 것을 먼저 내밀면 그것이 선택될 수 있음 |
| `least strength: A` | 가장 약한 것도 A등급. RC4·DES·수출용 같은 **명백히 약한 암호군은 이미 없음** |

> 시간·응답 시간(`latency`)은 실행할 때마다 다릅니다. 같은 점검을 더 자세히 하는 오픈소스 도구로 **testssl.sh** 가 있습니다(`sudo apt install -y testssl.sh`). 검사 항목이 많아 몇 분이 걸립니다.
{: .prompt-info }

### [실습 2] (Kali) RSA 키 교환으로 접속 — 허용됨

`AES256-SHA` 는 OpenSSL에서 `TLS_RSA_WITH_AES_256_CBC_SHA`(RSA 키 교환 + CBC) 암호군의 이름입니다. 이 암호군만 내밀어 접속해 봅니다.

```bash
# === Kali ===
openssl s_client -connect 192.168.0.30:443 -brief -tls1_2 -cipher AES256-SHA < /dev/null
```

```
Connecting to 192.168.0.30
Can't use SSL_get_servername
CONNECTION ESTABLISHED
Protocol version: TLSv1.2
Ciphersuite: AES256-SHA
Peer certificate: C=KR, O=LabOrg, CN=ubuntu.lab.local
Verification: OK
DONE
```

| 명령·옵션 | 설명 |
|---|---|
| `-cipher AES256-SHA` | 이 암호군**만** 제안 |

| 화면 | 의미 |
|---|---|
| `Ciphersuite: AES256-SHA` | 서버가 **RSA 키 교환** 암호군으로 연결해 줌 |
| `Peer Temp Key` 줄이 없음 | 06-1의 연결과 달리 **임시 키(ECDHE)를 쓰지 않음** → PFS 없음 |

### [실습 3] (Ubuntu) TLS 설정 강화

허용할 버전과 암호군을 적은 설정 파일을 새로 만들고 켭니다.

```bash
# === Ubuntu ===
sudo tee /etc/apache2/conf-available/tls-hardening.conf << 'EOF'
# TLS hardening for the lab web server
SSLProtocol -all +TLSv1.2 +TLSv1.3
SSLCipherSuite ECDHE+AESGCM:ECDHE+CHACHA20
SSLHonorCipherOrder on
EOF
sudo a2enconf tls-hardening
sudo apache2ctl configtest
sudo systemctl reload apache2
```

```
# TLS hardening for the lab web server
SSLProtocol -all +TLSv1.2 +TLSv1.3
SSLCipherSuite ECDHE+AESGCM:ECDHE+CHACHA20
SSLHonorCipherOrder on
Enabling conf tls-hardening.
To activate the new configuration, you need to run:
  systemctl reload apache2
Syntax OK
```

| 명령·설정 | 설명 |
|---|---|
| `tee 파일 << 'EOF' ... EOF` | `EOF` 사이의 내용을 파일로 저장하고 화면에도 출력 |
| `SSLProtocol -all +TLSv1.2 +TLSv1.3` | 모든 버전을 끄고(`-all`) **TLS 1.2·1.3만** 켬 |
| `SSLCipherSuite ECDHE+AESGCM:ECDHE+CHACHA20` | TLS 1.2 암호군을 **ECDHE 키 교환 + AES-GCM 또는 ChaCha20** 으로 제한. RSA 키 교환·CBC는 모두 빠짐 |
| `SSLHonorCipherOrder on` | 클라이언트가 아니라 **서버가 정한 순서**대로 암호군을 고름 |
| `a2enconf tls-hardening` | `conf-available` 의 설정을 켬. 이 파일은 기본 `ssl.conf` 보다 **나중에 읽혀서** 같은 항목을 덮어씀 |
| `systemctl reload apache2` | 접속을 끊지 않고 설정만 다시 읽음(출력 없음) |

### [실습 4] (Kali) 다시 점검 — 약한 키 교환 거부

```bash
# === Kali ===
nmap -Pn -p 443 --script ssl-enum-ciphers 192.168.0.30
```

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-16 17:02 +0900
Nmap scan report for 192.168.0.30
Host is up (0.000087s latency).

PORT    STATE SERVICE
443/tcp open  https
| ssl-enum-ciphers: 
|   TLSv1.2: 
|     ciphers: 
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
|       TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256 (ecdh_x25519) - A
|     compressors: 
|       NULL
|     cipher preference: server
|   TLSv1.3: 
|     ciphers: 
|       TLS_AKE_WITH_AES_256_GCM_SHA384 (ecdh_x25519) - A
|       TLS_AKE_WITH_CHACHA20_POLY1305_SHA256 (ecdh_x25519) - A
|       TLS_AKE_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
|     cipher preference: server
|_  least strength: A

Nmap done: 1 IP address (1 host up) scanned in 0.65 seconds
```

```bash
# === Kali ===
openssl s_client -connect 192.168.0.30:443 -brief -tls1_2 -cipher AES256-SHA < /dev/null
```

```
Connecting to 192.168.0.30
40274EA450730000:error:0A000410:SSL routines:ssl3_read_bytes:ssl/tls alert handshake failure:../ssl/record/rec_layer_s3.c:918:SSL alert number 40
40274EA450730000:error:0A000197:SSL routines:SSL_shutdown:shutdown while in init:../ssl/ssl_lib.c:2804:
```

| 화면 | 의미 |
|---|---|
| TLS 1.2 암호군 44개 → **3개** | ECDHE + GCM/ChaCha20만 남음. 모두 **PFS** |
| `cipher preference: server` | 서버가 정한 순서(강한 것 먼저)로 선택 |
| `alert handshake failure` ... `SSL alert number 40` | 서버가 **Alert**(handshake_failure, 40번)로 "그 암호군으로는 협상할 수 없다"고 거부 |

| 구분 | 기본 설정 | 강화 후 |
|---|---|---|
| 허용 버전 | TLS 1.2·1.3(OpenSSL 기본값에 의존) | TLS 1.2·1.3(**설정에 명시**) |
| TLS 1.2 암호군 | 44개(RSA 키 교환, CBC 포함) | 3개(ECDHE + AEAD) |
| `AES256-SHA` 접속 | **성공**(PFS 없음) | **거부**(alert 40) |
| 암호군 선택 | 클라이언트 순서 | 서버 순서 |

> 암호군을 너무 줄이면 아주 오래된 클라이언트(예: 윈도우 XP의 IE)는 접속하지 못합니다. 실제 서비스에서는 사용자 환경을 보고 정하며, Mozilla가 공개하는 **SSL Configuration Generator** 의 권장값(Intermediate)을 많이 참고합니다.
{: .prompt-info }

---

## 3.5 구현 취약점 — Heartbleed

### [이론]

**Heartbleed**(2014, CVE-2014-0160)는 TLS 프로토콜이 아니라 **OpenSSL 라이브러리의 프로그래밍 실수**입니다.

| 항목 | 내용 |
|---|---|
| 대상 | **OpenSSL 1.0.1 ~ 1.0.1f** |
| 원인 | 연결이 살아 있는지 확인하는 **하트비트(Heartbeat) 확장**에서, 클라이언트가 "보낸 데이터 길이"를 실제보다 크게 적어도 **경계 검사를 하지 않음** |
| 결과 | 서버가 실제 데이터보다 많은, 최대 **64KB의 서버 메모리**를 응답에 담아 돌려줌. 반복하면 계속 새어 나감 |
| 새어 나갈 수 있는 것 | 서버 **개인키**, 세션 키, 다른 사용자의 쿠키·아이디·비밀번호 |
| 흔적 | 정상 TLS 메시지라 서버 로그에 거의 남지 않음 |

| 대책 | 이유 |
|---|---|
| OpenSSL **1.0.1g 이상**으로 업데이트(또는 하트비트 기능 끄기) | 버그 제거 |
| **인증서 폐지·재발급**, 새 개인키 생성 | 개인키가 이미 새어 나갔을 수 있음 |
| 사용자 **비밀번호 변경** 안내, 세션 초기화 | 메모리에 있던 비밀번호·쿠키가 새어 나갔을 수 있음 |

> 암호 알고리즘이 아무리 강해도 **구현(프로그램)** 에 버그가 있으면 소용없습니다. 그래서 보안 라이브러리는 **항상 최신 버전**으로 유지합니다.
{: .prompt-warning }

### [실습] (Ubuntu·Kali) 버전 확인과 취약점 점검

```bash
# === Ubuntu ===
openssl version
```

```
OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024)
```

| 화면 | 의미 |
|---|---|
| `OpenSSL 3.0.13` | Heartbleed 대상(1.0.1~1.0.1f)이 아님. Ubuntu는 보안 패치를 패키지 업데이트(`apt upgrade`)로 배포 |

Kali에서 `nmap` 의 취약점 점검 스크립트로 Heartbleed, POODLE, CCS 주입(OpenSSL의 또 다른 옛 버그)을 한 번에 검사합니다.

```bash
# === Kali ===
nmap -Pn -p 443 --script ssl-heartbleed,ssl-poodle,ssl-ccs-injection 192.168.0.30
```

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-16 17:03 +0900
Nmap scan report for 192.168.0.30
Host is up (0.000058s latency).

PORT    STATE SERVICE
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 0.66 seconds
```

| 명령·옵션 | 설명 |
|---|---|
| `--script ssl-heartbleed,ssl-poodle,ssl-ccs-injection` | 세 가지 알려진 취약점을 검사하는 스크립트를 쉼표로 이어 실행 |

| 화면 | 의미 |
|---|---|
| `443/tcp open https` 아래에 아무것도 없음 | 세 스크립트 모두 **취약점을 찾지 못함**. 취약하면 `VULNERABLE:` 과 설명이 출력됨 |

---

## 3.6 SSL Strip과 HSTS

### [이론]

사용자는 주소창에 `https://` 를 잘 치지 않습니다. `naver.com` 만 치거나 `http://` 로 시작하는 링크를 누르면, 옛 브라우저나 HTTPS 우선 기능을 끈 브라우저는 먼저 **HTTP(80)** 로 접속하고, 서버가 "HTTPS로 가라"고 알려 주면 그때 HTTPS로 옮겨 갑니다. **SSL Strip**(2009)은 이 **첫 HTTP 요청**을 노립니다.

| 순서 | 일어나는 일 |
|---|---|
| ① | 공격자가 중간자 위치를 차지함(ARP 스푸핑 등) |
| ② | 사용자의 첫 HTTP 요청을 가로채, 공격자가 대신 진짜 서버와 **HTTPS** 로 통신 |
| ③ | 사용자에게는 HTTPS 링크를 모두 **HTTP로 바꾼(strip)** 페이지를 평문으로 돌려줌 |
| ④ | 사용자는 인증서 경고 없이 **HTTP로 로그인** → 공격자가 평문으로 비밀번호를 봄 |

> SSL Strip은 인증서를 위조하지 않으므로 **경고가 뜨지 않습니다.** 주소창에 자물쇠가 없다는 것 말고는 사용자가 눈치채기 어렵습니다.
{: .prompt-danger }

| 대책 | 내용 |
|---|---|
| HTTP → HTTPS 이동(리다이렉트) | 80번으로 오면 `301` 응답으로 HTTPS 주소를 알려 줌. **이것만으로는 첫 요청이 여전히 HTTP라 부족** |
| **HSTS**(HTTP Strict Transport Security) | HTTPS 응답에 `Strict-Transport-Security: max-age=초` 헤더를 붙임. 브라우저는 그 기간 동안 이 사이트에 **HTTP로는 아예 접속하지 않고** 스스로 HTTPS로 바꿔 접속하며, 인증서 경고가 나면 **무시하고 계속하는 버튼도 없앰** |
| HSTS Preload | 브라우저에 미리 내장된 "HTTPS 전용 사이트 목록"에 등록 → **맨 처음 접속**부터 HTTPS |

### [실습 1] (Ubuntu) HTTP→HTTPS 이동과 HSTS 설정

```bash
# === Ubuntu ===
sudo a2enmod headers
grep -q 'Strict-Transport' /etc/apache2/sites-available/default-ssl.conf || \
  sudo sed -i '/SSLEngine on/a Header always set Strict-Transport-Security "max-age=31536000"' \
  /etc/apache2/sites-available/default-ssl.conf
grep -q 'Redirect permanent' /etc/apache2/sites-available/000-default.conf || \
  sudo sed -i '/^\s*DocumentRoot/a Redirect permanent / https://ubuntu.lab.local/' \
  /etc/apache2/sites-available/000-default.conf
grep -h -E 'Strict-Transport|Redirect' /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/000-default.conf
sudo apache2ctl configtest
sudo systemctl restart apache2
```

```
Enabling module headers.
To activate the new configuration, you need to run:
  systemctl restart apache2
Header always set Strict-Transport-Security "max-age=31536000"
Redirect permanent / https://ubuntu.lab.local/
Syntax OK
```

| 명령·설정 | 설명 |
|---|---|
| `a2enmod headers` | 응답 헤더를 추가하는 모듈(`mod_headers`)을 켬 |
| `sed -i '/찾을 글자/a 넣을 줄'` | `찾을 글자` 가 있는 줄 **바로 아래(a, append)** 에 한 줄을 넣음. `^\s*DocumentRoot` 는 주석(`#`)이 아닌, 줄 앞이 공백뿐인 `DocumentRoot` 줄만 찾음 |
| `grep -q '...' 파일` 과 세로 막대 두 개 | 그 줄이 **이미 있으면 sed를 실행하지 않음**. 블록을 실수로 두 번 실행해도 같은 줄이 두 번 들어가지 않음 |
| `Header always set Strict-Transport-Security "max-age=31536000"` | HTTPS 사이트(443)의 모든 응답에 HSTS 헤더 추가. 31536000초 = **1년** |
| `Redirect permanent / https://ubuntu.lab.local/` | HTTP 사이트(80)로 온 요청을 HTTPS로 **영구 이동(301)**. 이동할 주소는 IP가 아니라 **이름**으로 적음(아래 주의 참고) |

> **브라우저는 IP 주소로 접속한 사이트에는 HSTS를 적용하지 않습니다**(HSTS 표준 RFC 6797). 그래서 이 절에서는 인증서의 SAN에 들어 있는 이름 `ubuntu.lab.local` 로 접속합니다.
{: .prompt-warning }

### [실습 2] (Kali) 이름 등록과 확인

Kali가 `ubuntu.lab.local` 을 찾을 수 있도록 hosts 파일에 한 줄을 더합니다(04-1 1.2절).

```bash
# === Kali ===
grep -q 'ubuntu.lab.local' /etc/hosts || echo '192.168.0.30 ubuntu.lab.local' | sudo tee -a /etc/hosts
```

```
192.168.0.30 ubuntu.lab.local
```

| 명령·옵션 | 설명 |
|---|---|
| `sudo tee -a /etc/hosts` | 받은 줄을 hosts 파일 **끝에 덧붙임**(`-a`, append)과 동시에 화면에 출력 |
| `grep -q ... /etc/hosts` 와 세로 막대 두 개 | 이미 그 이름이 hosts 파일에 있으면 덧붙이지 않음(이때는 **출력 없음**) |

```bash
# === Kali ===
curl -I http://ubuntu.lab.local/
curl -I https://ubuntu.lab.local/
```

```
HTTP/1.1 301 Moved Permanently
Date: Mon, 16 Nov 2026 08:17:19 GMT
Server: Apache/2.4.58 (Ubuntu)
Location: https://ubuntu.lab.local/
Content-Type: text/html; charset=iso-8859-1

HTTP/1.1 200 OK
Date: Mon, 16 Nov 2026 08:17:19 GMT
Server: Apache/2.4.58 (Ubuntu)
Strict-Transport-Security: max-age=31536000
Last-Modified: Mon, 16 Nov 2026 07:58:36 GMT
ETag: "29af-660602526f174"
Accept-Ranges: bytes
Content-Length: 10671
Vary: Accept-Encoding
Content-Type: text/html

```

| 화면 | 의미 |
|---|---|
| `301 Moved Permanently` + `Location: https://...` | HTTP로 오면 HTTPS 주소로 보냄 |
| `Strict-Transport-Security: max-age=31536000` | HTTPS 응답에 **HSTS** 가 붙음. 브라우저는 1년 동안 이 사이트를 HTTPS로만 접속 |

Kali의 Firefox 주소창에 `http://ubuntu.lab.local` 을 **`http://` 까지 직접** 입력해 봅니다(이름만 치면 최신 Firefox는 HTTPS부터 시도할 수 있음). 처음에는 서버의 `301` 응답으로 HTTPS로 넘어가고, 그 뒤로는 브라우저가 HSTS를 기억해 **서버에 묻기 전에 스스로** `https://` 로 바꿔 접속합니다.

> 📷 **캡처**: Firefox 개발자 도구(F12) → **Network** 탭을 연 채 `http://ubuntu.lab.local` 을 두 번째로 입력했을 때 `307 Internal Redirect`(브라우저가 HSTS로 스스로 바꾼 이동)와 응답 헤더의 `Strict-Transport-Security` 가 보이는 화면.

> HSTS는 **HTTPS 응답**에 붙어 있을 때만 브라우저가 받아들입니다. HTTP 응답에 넣으면 공격자가 지우거나 바꿀 수 있기 때문입니다. 또 인증서 경고가 있는 사이트에서는 HSTS를 무시하므로, 06-1처럼 CA를 신뢰 목록에 넣어 둔 상태여야 동작합니다.
{: .prompt-info }

---

# 4. SSL/TLS 보안 대책 종합

| 번호 | 위협 | 대책 | 이 글에서 확인한 방법 |
|---|---|---|---|
| 1 | 평문 통신 도청 | 모든 로그인·개인정보 페이지에 **HTTPS** | 06-1 1.1, 2.5 |
| 2 | 가짜 인증서·중간자 공격 | 공인·사내 CA 인증서, **경고 무시 금지**, CA 개인키 보호, 앱에서 검증 끄지 않기 | 3.2 |
| 3 | 다운그레이드·옛 버전 공격(POODLE, DROWN, BEAST) | SSL 2.0·3.0, TLS 1.0·1.1 끄기(`SSLProtocol -all +TLSv1.2 +TLSv1.3`) | 3.3 |
| 4 | 약한 암호·키 교환 | **ECDHE(PFS) + AES-GCM/ChaCha20**, RC4·DES·3DES·수출용·MD5 금지, 서버 순서 우선 | 3.4 |
| 5 | 압축 공격(CRIME) | TLS 압축 끄기 | 3.4 (`compressors: NULL`) |
| 6 | 구현 버그(Heartbleed) | OpenSSL 등 라이브러리 **최신 유지**, 유출 의심 시 키·인증서 재발급 | 3.5 |
| 7 | SSL Strip | HTTP→HTTPS **301 이동** + **HSTS**(+ Preload) | 3.6 |
| 8 | 인증서 만료·폐지 | 유효기간 관리·자동 갱신(Let's Encrypt는 90일), CRL·OCSP 확인 | 06-1 2.1 |
| 9 | 서버 정보 노출 | 정기 점검(`nmap ssl-enum-ciphers`, testssl.sh) | 3.4 |

> 이 실습에서 바꾼 설정(`tls-hardening.conf`, HSTS, HTTP→HTTPS 이동)과 Kali hosts 파일의 `ubuntu.lab.local` 줄은 그대로 두어도 다음 강의 진행에 문제가 없습니다. 다만 HTTP(80)로 접속하면 이제 HTTPS로 넘어가므로, 이후 강의에서 평문 HTTP 실습이 필요하면 아래 명령으로 되돌립니다.
{: .prompt-info }

```bash
# === Ubuntu ===
sudo a2disconf tls-hardening
sudo sed -i '/Strict-Transport-Security/d' /etc/apache2/sites-available/default-ssl.conf
sudo sed -i '/Redirect permanent/d' /etc/apache2/sites-available/000-default.conf
sudo systemctl restart apache2
```

```
Conf tls-hardening disabled.
To activate the new configuration, you need to run:
  systemctl reload apache2
```

| 명령·옵션 | 설명 |
|---|---|
| `a2disconf tls-hardening` | 3.4절에서 켠 TLS 강화 설정을 끔 |
| `sed -i '/글자/d'` | `글자` 가 들어 있는 줄을 지움(`d`, delete) |

---

# 확인 문제

> 1. 공격자가 서버인 척 가짜 인증서를 보냈을 때 공격이 **성공하는** 경우는?
>    ① 클라이언트가 인증서의 발급자를 신뢰 목록에서 확인한 경우
>    ② 사용자가 브라우저의 인증서 경고를 무시하고 접속을 계속한 경우
>    ③ 서버가 TLS 1.3만 허용한 경우
>    ④ 서버가 HSTS 헤더를 보낸 경우
>
> 2. SSL 3.0으로 협상을 끌어내린 뒤 CBC 모드의 패딩 검사 약점을 이용해 암호문을 해독하는 공격은?
>    ① Heartbleed
>    ② POODLE
>    ③ CRIME
>    ④ SSL Strip
>
> 3. Heartbleed 취약점에 대한 설명으로 **틀린** 것은?
>    ① OpenSSL의 하트비트 확장에서 길이 값의 경계 검사를 하지 않아 발생하였다
>    ② 한 번에 최대 64KB의 서버 메모리 내용이 노출될 수 있다
>    ③ 서버의 개인키가 노출될 수 있으므로 인증서 재발급이 필요하다
>    ④ TLS 프로토콜 설계의 결함이므로 TLS 1.3으로 바꾸어야만 해결된다
>
> 4. 사용자의 첫 HTTP 요청을 가로채 HTTPS 링크를 HTTP로 바꾸어 전달하는 공격과, 이를 막기 위해 서버가 보내는 응답 헤더를 바르게 짝지은 것은?
>    ① SSL Strip – Strict-Transport-Security
>    ② SSL Strip – Content-Security-Policy
>    ③ POODLE – Strict-Transport-Security
>    ④ DROWN – X-Frame-Options
>
> 5. 서버의 개인키가 나중에 유출되더라도 과거에 녹화된 통신 내용을 복호화할 수 없게 하는 성질과, 이를 제공하는 키 교환 방식으로 옳은 것은?
>    ① 부인방지 – RSA 키 교환
>    ② PFS(전방향 안전성) – ECDHE 키 교환
>    ③ 무결성 – HMAC
>    ④ PFS(전방향 안전성) – RSA 키 교환
>
> 6. TLS의 압축 기능을 이용해 압축된 길이의 변화로 쿠키 등 비밀값을 추측하는 공격과 그 대책은?
>
> 7. 웹 서버의 SSL/TLS 보안 설정으로 적절하지 **않은** 것은?
>    ① SSL 2.0, SSL 3.0을 비활성화한다
>    ② ECDHE 키 교환과 AES-GCM 암호군을 우선 사용한다
>    ③ 오래된 클라이언트 호환을 위해 RC4와 수출용(EXPORT) 암호군을 허용한다
>    ④ OpenSSL 등 암호 라이브러리를 최신 버전으로 유지한다
>
> 8. 서버에 TLS 1.0으로 접속을 시도했더니 "alert protocol version" 오류가 발생하였다. 이 오류를 보낸 SSL/TLS의 하위 프로토콜은?
>    ① Record
>    ② Handshake
>    ③ ChangeCipherSpec
>    ④ Alert
{: .prompt-tip }

<details>
<summary>정답 보기</summary>

1. <strong>②</strong> 경고를 무시하면 가짜 서버와 암호화 연결이 맺어져 공격자가 복호화한다(3.2절 실습 3).<br>
2. <strong>②</strong> (3.3절)<br>
3. <strong>④</strong> Heartbleed는 프로토콜 설계가 아니라 OpenSSL 구현의 버그이며, 라이브러리 업데이트(1.0.1g 이상)로 해결한다(3.5절).<br>
4. <strong>①</strong> (3.6절)<br>
5. <strong>②</strong> RSA 키 교환은 서버 개인키로 과거 통신까지 복호화할 수 있다. ECDHE는 연결마다 임시 키를 쓰고 버린다(06-1 2.5절, 3.4절).<br>
6. <strong>CRIME 공격, TLS 압축 기능 끄기</strong> (3.4절, `compressors: NULL`)<br>
7. <strong>③</strong> RC4와 수출용 암호군은 깨진 암호로 허용하면 안 된다(3.4절).<br>
8. <strong>④</strong> 오류·종료를 알리는 것은 Alert 프로토콜이다(protocol_version, 70번)(3.3절).

</details>

---

# 체크리스트

- [ ] SSL/TLS 공격을 인증서·버전·암호·압축·구현·HTTPS 우회로 나누어 원리와 대책을 설명 가능
- [ ] (Kali) 이름이 같은 가짜 인증서를 만들어 진짜와 발급자·지문이 다른 것을 확인
- [ ] (Kali) 인증서를 검증하면 가짜 서버 연결이 `self-signed certificate` 로 거부되고, `-k`(경고 무시)로 접속하면 공격자 화면에 `Authorization` 헤더가 나오는 것을 확인
- [ ] (Kali) TLS 1.0·1.1 접속이 `alert protocol version`(70)으로 거부되는 것을 확인
- [ ] (Kali·Ubuntu) `nmap ssl-enum-ciphers` 로 암호군 44개 → 3개, `AES256-SHA` 접속이 성공 → `handshake failure`(40)로 바뀌는 것을 확인
- [ ] Heartbleed의 대상 버전·원인·피해·대책을 설명하고, (Ubuntu) OpenSSL 버전과 (Kali) nmap 취약점 점검 결과를 확인
- [ ] (Kali) `ubuntu.lab.local` 로 HTTP 접속하면 `301` 로 HTTPS로 이동하고, HTTPS 응답에 `Strict-Transport-Security` 가 붙는 것을 확인(IP 접속에는 HSTS가 적용되지 않는 이유 포함)
- [ ] SSL Strip이 인증서 경고 없이 성공하는 이유와 HSTS가 이를 막는 원리를 설명 가능

> 이로써 **06. SSL/TLS** 를 마칩니다. 다음은 **07. IPSec·SET·OTP** 입니다. 06-1 1.3절에서 남겨 둔 문제, 즉 **SSL로 카드번호를 보내면 상점에 그대로 노출되는 문제**를 SET이 어떻게 해결하는지 살펴봅니다.
{: .prompt-info }
