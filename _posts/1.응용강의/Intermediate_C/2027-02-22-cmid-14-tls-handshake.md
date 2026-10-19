---
title: 중급 C 프로그래밍 14강 - TLS 핸드셰이크 완전 분해
date: 2027-02-22 09:00:00 +0900
categories:
  - 1.응용강의
  - 중급 C 프로그래밍
tags:
  - C언어
  - 시스템프로그래밍
  - 리눅스
  - TLS
  - 인증서
  - X.509
  - 인증기관
  - OpenSSL
pin:
mermaid: false
---

> **학습 목표**
> 1. 인증서가 무엇을 해결하는지 설명할 수 있다.
> 2. **자체 인증기관(CA)을 직접 만들** 수 있다.
> 3. 서버 인증서를 발급하고 **신뢰의 사슬**을 검증할 수 있다.
> 4. **TLS 1.3 핸드셰이크의 메시지 순서**를 실제로 관찰할 수 있다.
> 5. 핸드셰이크의 각 메시지가 **12·13강의 무엇에 해당하는지** 짝지을 수 있다.
> 6. **검증 실패 세 가지**(발급자 없음·이름 불일치·기간 만료)를 재현할 수 있다.
> 7. 실제 인터넷 서버의 인증서 사슬을 읽을 수 있다.
> 8. 표준 원문([RFC 9846](https://www.rfc-editor.org/rfc/rfc9846))에서 필요한 절을 찾을 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

13강 6.4절에서 마지막 고리가 남았습니다.

> **"앨리스는 밥의 서명 공개키를 미리 알고 있다"** — 처음 보는 서버와는 어떻게 합니까?

**오늘 그 고리를 채웁니다.** 그리고 12·13강에서 만든 조각들이 **TLS 안에 그대로 들어 있다**는 것을 확인합니다.

| 우리가 만든 것 | TLS에서의 이름 | 오늘 확인할 절 |
|---|---|---|
| 11강 길이 앞머리 프레이밍 | 레코드 계층 | 6.2 |
| 12강 AES-256-GCM | 레코드 보호 | 6.4 |
| 13강 X25519 키 교환 | key_share | **6.4** |
| 13강 Ed25519 서명 | CertificateVerify | **6.4** |
| **오늘: 인증서** | Certificate | 제3절 |

이 강의는 **3회차 분량**(모두 합쳐 약 480분)입니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제4절 | 인증서·**자체 CA 만들기**·사슬 검증 | 185분 |
| **2회차** | 제5절 ~ 제8절 | **핸드셰이크 관찰**·검증 실패·실제 서버 | 185분 |
| **3회차** | 실습문제 | 스스로 해 보기 | 110분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 인증서란 | 30분 |
| 제2절 | X.509 인증서 읽기 | 35분 |
| 제3절 | **자체 CA 만들기** | 60분 |
| 제4절 | 신뢰의 사슬 | 40분 |
| 제5절 | TLS 서버 띄우기 | 30분 |
| 제6절 | **핸드셰이크 완전 분해** | 70분 |
| 제7절 | **검증이 실패하는 세 가지** | 45분 |
| 제8절 | 실제 인터넷 서버 | 25분 |
| 제9절 | 자주 나오는 실수 | 20분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 디렉터리**는 `~/cmid/lab14`, **VM 두 대**를 씁니다.

```bash
mkdir -p ~/cmid/lab14/ca && cd ~/cmid/lab14
```

```bash
openssl version
```

```text
OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024)
```

> 판이 다르면 출력 형식이 조금 다를 수 있습니다. **3.0 이상**이면 이 강의를 그대로 따라갈 수 있습니다.
{: .prompt-tip }

---

## 제1절. 인증서란

### 1.1 문제를 다시 보십시오

13강에서 중간자를 막으려면 **밥의 서명 공개키를 미리 알아야** 했습니다. 상대가 수억 개라면 불가능합니다.

### 1.2 답: 제삼자가 보증한다

> **인증서**: "이 공개키는 이 이름의 주인의 것이다" 라고 **제삼자가 서명한 문서**.

```text
   [인증서]
     이름     : c-srv
     공개키   : (밥의 공개키)
     유효기간 : 2026-08-22 ~ 2027-08-22
     발급자   : cmid-lab Root CA
     ─────────────────────────────
     서명     : (CA 의 개인키로 서명)   ← 13강의 그 서명
```

**서명 자체는 13강에서 이미 배웠습니다.** 새로운 것은 **누가 무엇에 서명하는가**입니다.

### 1.3 문제가 옮겨 갔을 뿐 아닌가

**맞습니다.** 이제 **CA의 공개키를 어떻게 믿는가**가 됩니다.

| 그러나 달라진 것 | |
|---|---|
| 미리 알아야 할 것이 **수억 개 → 수백 개** | CA는 몇백 곳뿐 |
| 운영체제·브라우저가 **미리 담아 둔다** | 설치할 때 함께 온다 |
| 서버가 바뀌어도 | **CA는 그대로** |

**결국 어딘가에서는 "그냥 믿는" 지점이 있어야 합니다.** 그것을 **신뢰 앵커**라 하며, 우리 시스템에서는 여기 있습니다.

```bash
ls /etc/ssl/certs/ | head -5
```

### 1.4 우리가 할 일

**우리가 직접 CA가 되어 봅니다.** CA가 하는 일을 손으로 해 보면 인증서 구조가 몸에 익습니다.

---

## 제2절. X.509 인증서 읽기

### 2.1 형식

| 항목 | |
|---|---|
| 표준 | **X.509 v3** — [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280) |
| 내부 부호화 | DER(이진) |
| 파일 형식 | **PEM**(DER을 Base64로 감싼 텍스트) |

```text
-----BEGIN CERTIFICATE-----
MIIBRTCB+KADAgECAhQ...
-----END CERTIFICATE-----
```

### 2.2 주요 필드

| 필드 | 뜻 |
|---|---|
| **Subject** | 이 인증서의 주인 |
| **Issuer** | 발급한 곳 |
| **Validity** | 유효 기간 |
| **Public Key** | 주인의 공개키 |
| **Subject Alternative Name**(SAN) | **실제로 검사하는 이름 목록** |
| Basic Constraints | **CA인가 아닌가** |
| Extended Key Usage | 용도(서버 인증 등) |
| Signature | 발급자의 서명 |

> **`CN`이 아니라 `SAN`을 봅니다.**
> 예전에는 `Subject`의 `CN`으로 이름을 검사했지만, 오늘날 브라우저와 라이브러리는 **`SAN`만** 봅니다. `SAN`이 없는 인증서는 이름 검증에 실패합니다. [RFC 6125](https://www.rfc-editor.org/rfc/rfc6125)에 정리되어 있습니다.
{: .prompt-warning }

---

## 제3절. 자체 CA 만들기

### 3.1 CA의 키

```bash
openssl genpkey -algorithm ED25519 -out ca/ca.key
```

```bash
chmod 600 ca/ca.key
```

> **CA의 개인키가 이 실습에서 가장 중요한 파일입니다.**
> 이것이 새어 나가면 **누구든 이 CA 이름으로 인증서를 발급**할 수 있습니다. 실제 CA는 인터넷에 연결되지 않은 장비에 보관합니다. 3강 4.5절에서 배운 대로 **0600**이어야 합니다.
{: .prompt-danger }

### 3.2 CA 인증서 (자기 자신에게 서명)

```bash
openssl req -x509 -new -key ca/ca.key -sha256 -days 3650 \
  -subj "/C=KR/O=cmid-lab/CN=cmid-lab Root CA" -out ca/ca.crt
```

```bash
openssl x509 -in ca/ca.crt -noout -subject -issuer -dates
```

```text
subject=C = KR, O = cmid-lab, CN = cmid-lab Root CA
issuer=C = KR, O = cmid-lab, CN = cmid-lab Root CA
notBefore=Aug 22 05:07:40 2026 GMT
notAfter=Aug 19 05:07:40 2036 GMT
```

**`subject`와 `issuer`가 같습니다.** 이것이 **뿌리 인증서**의 특징입니다. 자기가 자기를 보증하므로, **결국 "그냥 믿는" 지점**입니다(1.3절).

```bash
ls -l ca/
```

```text
-rw-r--r-- 1 seungsoo seungsoo 595 Aug 22 14:07 ca.crt
-rw------- 1 seungsoo seungsoo 119 Aug 22 14:07 ca.key
```

**개인키는 119바이트, 인증서는 595바이트**입니다. Ed25519라 이렇게 작습니다. RSA-2048이었다면 각각 열 배가 넘습니다.

### 3.3 서버의 키와 발급 요청서

```bash
openssl genpkey -algorithm ED25519 -out server.key
chmod 600 server.key
```

```bash
openssl req -new -key server.key -subj "/C=KR/O=cmid-lab/CN=c-srv" -out server.csr
```

**CSR**(Certificate Signing Request)은 "이 공개키에 이 이름으로 인증서를 주십시오"라는 요청서입니다. **개인키는 들어 있지 않습니다** — 서버 밖으로 나가지 않습니다.

### 3.4 확장 필드 지정

```bash
cat > server.ext <<'EOF'
basicConstraints = CA:FALSE
keyUsage = digitalSignature
extendedKeyUsage = serverAuth
subjectAltName = DNS:c-srv, DNS:localhost, IP:127.0.0.1
EOF
```

| 줄 | 뜻 |
|---|---|
| `CA:FALSE` | **이 인증서로 다른 인증서를 발급할 수 없다** |
| `serverAuth` | TLS 서버용 |
| `subjectAltName` | **검사할 이름 목록**(2.2절) |

**`CA:FALSE`가 중요합니다.** 이것이 없으면 서버 인증서를 훔친 사람이 **다른 인증서를 마음대로 발급**할 수 있습니다.

### 3.5 CA가 서명

```bash
openssl x509 -req -in server.csr -CA ca/ca.crt -CAkey ca/ca.key \
  -CAcreateserial -days 365 -extfile server.ext -out server.crt
```

```text
Certificate request self-signature ok
subject=C = KR, O = cmid-lab, CN = c-srv
```

```bash
openssl x509 -in server.crt -noout -subject -issuer -dates
```

```text
subject=C = KR, O = cmid-lab, CN = c-srv
issuer=C = KR, O = cmid-lab, CN = cmid-lab Root CA
notBefore=Aug 22 05:07:40 2026 GMT
notAfter=Aug 22 05:07:40 2027 GMT
```

**이번에는 `subject`와 `issuer`가 다릅니다.** CA가 서버를 보증한 것입니다.

```bash
openssl x509 -in server.crt -noout -ext subjectAltName,basicConstraints,extendedKeyUsage
```

```text
X509v3 Basic Constraints: 
    CA:FALSE
X509v3 Extended Key Usage: 
    TLS Web Server Authentication
X509v3 Subject Alternative Name: 
    DNS:c-srv, DNS:localhost, IP Address:127.0.0.1
```

**지정한 확장이 그대로 들어갔습니다.**

---

## 제4절. 신뢰의 사슬

### 4.1 검증

```bash
openssl verify -CAfile ca/ca.crt server.crt
```

```text
server.crt: OK
```

### 4.2 CA를 주지 않으면

```bash
openssl verify server.crt
```

```text
C = KR, O = cmid-lab, CN = c-srv
error 20 at 0 depth lookup: unable to get local issuer certificate
error server.crt: verification failed
```

**오류 20 — 발급자를 찾을 수 없습니다.** 시스템의 신뢰 목록에 `cmid-lab Root CA`가 없기 때문입니다. 당연합니다. 우리가 방금 만든 것이니까요.

### 4.3 사슬이란

```text
   [뿌리 CA]  ← 시스템이 미리 믿는다
       │ 서명
   [중간 CA]  (있을 수도, 없을 수도)
       │ 서명
   [서버 인증서]
```

| 단계 | 검사 |
|---|---|
| 각 서명 | 위쪽 인증서의 공개키로 검증 |
| 각 기간 | 지금이 유효 기간 안인가 |
| 각 `basicConstraints` | 중간 단계는 **CA:TRUE**여야 한다 |
| 마지막 | **뿌리가 신뢰 목록에 있는가** |
| 서버 인증서 | **이름이 맞는가**(SAN) |

**하나라도 실패하면 전체가 실패합니다.**

### 4.4 왜 중간 CA를 두는가

| 이유 | |
|---|---|
| 뿌리 개인키를 **꺼내지 않는다** | 금고에 보관, 중간 CA만 운영 |
| 사고가 나면 | **중간 CA만 폐기** |
| 용도별 분리 | 서버용·코드 서명용 |

제8절에서 실제 사슬을 봅니다.

---

> **▶ 여기서부터 2회차 — 핸드셰이크 관찰·검증 실패·실제 서버**
> 제5절 ~ 제8절, 약 185분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제5절. TLS 서버 띄우기

### 5.1 명령으로 띄우기

```bash
openssl s_server -accept 4433 -cert server.crt -key server.key -www &
```

`-www`는 간단한 웹 응답을 돌려줍니다. **15강에서 이것을 C로 직접 만듭니다.**

### 5.2 접속

```bash
echo | openssl s_client -connect 127.0.0.1:4433 -CAfile ca/ca.crt \
  -servername c-srv -brief
```

```text
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: C = KR, O = cmid-lab, CN = c-srv
Hash used: UNDEF
Signature type: Ed25519
Server Temp Key: X25519, 253 bits
Verification: OK
DONE
```

### 5.3 이 여섯 줄을 자세히 보십시오

| 줄 | 우리가 배운 것 |
|---|---|
| `Protocol version: TLSv1.3` | 오늘의 주제 |
| **`Ciphersuite: TLS_AES_256_GCM_SHA384`** | **12강의 AES-256-GCM** |
| `Peer certificate: ... CN = c-srv` | 제3절에서 만든 인증서 |
| **`Signature type: Ed25519`** | **13강의 Ed25519 서명** |
| **`Server Temp Key: X25519`** | **13강의 X25519 키 교환** |
| `Verification: OK` | 제4절의 사슬 검증 |

> **12·13강에서 만든 것이 그대로 있습니다.**
> `AES-256-GCM`, `X25519`, `Ed25519` — 우리가 손으로 만들어 본 바로 그 세 가지입니다. TLS는 이것들을 **정해진 순서로 조립한 규약**입니다.
>
> `Temp Key`의 **"Temp"** 는 13강 3.5절의 **임시 키 쌍**입니다. 연결마다 새로 만들고 버리므로 순방향 비밀성이 확보됩니다.
{: .prompt-tip }

### 5.4 `Hash used: UNDEF`

Ed25519는 서명 알고리즘 안에 해시가 정해져 있어 따로 고르지 않습니다(13강 5.3절). 그래서 `UNDEF`로 나옵니다. **오류가 아닙니다.**

---

## 제6절. 핸드셰이크 완전 분해

### 6.1 표준 문서

> **TLS 1.3의 현행 표준은 [RFC 9846](https://www.rfc-editor.org/rfc/rfc9846)** 입니다.

| RFC | 상태 |
|---|---|
| RFC 5246 (TLS 1.2) | 구식 |
| RFC 8446 (TLS 1.3, 2018) | **폐기됨** |
| **RFC 9846** | **현행** |

> **RFC 8446을 인용한 자료가 아직 많습니다.**
> [RFC 8446 안내 페이지](https://www.rfc-editor.org/info/rfc8446)에 "This RFC is now obsolete, see RFC 9846." 이라고 적혀 있습니다. 내용의 뼈대는 같지만, **표준을 인용할 때는 현행 번호**를 쓰셔야 합니다.
>
> **표준 문서는 늘 현재 상태를 확인하는 습관**을 들이십시오. 1부부터 강조해 온 "출처를 확인하라"가 여기서도 적용됩니다.
{: .prompt-warning }

### 6.2 메시지 순서 관찰

```bash
echo | openssl s_client -connect 127.0.0.1:4433 -CAfile ca/ca.crt \
  -servername c-srv -msg 2>&1 | grep -E "^(>>>|<<<)"
```

```text
>>> TLS 1.0, RecordHeader [length 0005]
>>> TLS 1.3, Handshake [length 012e], ClientHello
<<< TLS 1.2, RecordHeader [length 0005]
<<< TLS 1.3, Handshake [length 007a], ServerHello
<<< TLS 1.2, RecordHeader [length 0005]
<<< TLS 1.2, RecordHeader [length 0005]
<<< TLS 1.3, InnerContent [length 0001]
<<< TLS 1.3, Handshake [length 0006], EncryptedExtensions
<<< TLS 1.2, RecordHeader [length 0005]
<<< TLS 1.3, InnerContent [length 0001]
<<< TLS 1.3, Handshake [length 01d2], Certificate
<<< TLS 1.2, RecordHeader [length 0005]
<<< TLS 1.3, InnerContent [length 0001]
<<< TLS 1.3, Handshake [length 0048], CertificateVerify
<<< TLS 1.2, RecordHeader [length 0005]
<<< TLS 1.3, InnerContent [length 0001]
<<< TLS 1.3, Handshake [length 0034], Finished
>>> TLS 1.2, RecordHeader [length 0005]
>>> TLS 1.3, ChangeCipherSpec [length 0001]
```

**`>>>`는 보낸 것, `<<<`는 받은 것**입니다.

### 6.3 RecordHeader가 무엇인가

**`RecordHeader [length 0005]` 가 매번 앞에 붙습니다.** 5바이트입니다.

```text
   +------+---------+--------+---------------+
   | 종류1 | 판번호2 | 길이2   | 내용 (길이만큼) |
   +------+---------+--------+---------------+
```

> **11강에서 만든 길이 앞머리 프레이밍과 똑같은 구조**입니다.
> 우리는 4바이트 길이를 앞에 붙였고(`msg_send`), TLS는 종류·판번호·길이 5바이트를 붙입니다. **TCP가 경계를 지켜 주지 않으므로 스스로 표시해야 한다**는 11강 2절의 문제를, TLS도 똑같이 풀고 있습니다.
{: .prompt-tip }

`TLS 1.0`, `TLS 1.2`로 찍히는 판 번호는 **옛 장비와의 호환을 위한 위장**입니다. 실제 판은 확장 필드로 알립니다. RFC 9846의 `legacy_record_version` 항목에 설명되어 있습니다.

### 6.4 각 메시지가 하는 일

| 순서 | 메시지 | 하는 일 | 우리가 배운 것 |
|---|---|---|---|
| 1 | **ClientHello** | 지원 목록 + **내 교환용 공개키** | **13강 3.2절** |
| 2 | **ServerHello** | 선택 결과 + **서버 교환용 공개키** | **13강 3.2절** |
| — | *(여기서 양쪽이 키를 만든다)* | | **13강 3.3절** |
| 3 | EncryptedExtensions | 나머지 설정 — **여기부터 암호화** | 12강 |
| 4 | **Certificate** | **서버 인증서 사슬** | **오늘 제3절** |
| 5 | **CertificateVerify** | **핸드셰이크 전체에 서명** | **13강 제6절** |
| 6 | Finished | 여기까지 틀림없다 | |

### 6.5 결정적인 관찰 — 언제부터 암호화되는가

**`ClientHello`와 `ServerHello`만 평문이고, 그 뒤는 전부 암호화**되어 있습니다.

| 근거 | |
|---|---|
| `EncryptedExtensions` 앞에 **`InnerContent`** 가 붙었다 | 암호화된 레코드를 푼 뒤의 내용이라는 표시 |
| `ClientHello`·`ServerHello`에는 없다 | 평문이다 |

**`Certificate`조차 암호화됩니다.** 도청자는 **어떤 인증서를 쓰는지도 볼 수 없습니다.** TLS 1.2에서는 인증서가 평문이었습니다.

### 6.6 왜 이 순서인가

```text
   ① 키부터 만든다        (ClientHello / ServerHello)
   ② 그 키로 감싼다        (EncryptedExtensions 부터)
   ③ 감싼 안에서 신원 확인  (Certificate / CertificateVerify)
```

**13강 제6절에서 우리가 한 것과 순서가 반대로 보입니다.** 우리는 서명을 먼저 검증하고 키 교환을 했습니다. TLS는 키를 먼저 만들고 그 안에서 신원을 확인합니다.

| | 우리 방식 | TLS 1.3 |
|---|---|---|
| 안전한가 | 그렇다 | **그렇다** |
| 인증서 노출 | — | **감춰진다** |
| 왕복 횟수 | 더 필요 | **1회** |

**어느 쪽이든 "서명으로 키 교환을 묶는다"는 본질은 같습니다.** `CertificateVerify`는 **자기 공개키만이 아니라 핸드셰이크 전체의 해시**에 서명합니다. 그래서 중간에서 무엇 하나라도 바꾸면 서명이 맞지 않습니다.

### 6.7 ChangeCipherSpec은 왜 있나

**TLS 1.3에서는 아무 뜻도 없습니다.** 옛 중간 장비들이 "TLS는 이 메시지를 보내는 것"이라고 굳어 있어, 보내지 않으면 연결을 끊는 경우가 있었습니다. **호환을 위한 빈 메시지**입니다.

> 표준을 만들 때 **기술적 최선보다 현실의 제약**이 앞서는 경우입니다. 2강 8.4절에서 본 "옛 결정이 남긴 흔적"과 같은 이야기입니다.
{: .prompt-info }

### 6.8 눈으로 보는 참고 자료

바이트 하나하나에 설명이 붙은 자료가 있습니다.

| 자료 | 주소 |
|---|---|
| TLS 1.3 핸드셰이크 주석 | [tls13.xargs.org](https://tls13.xargs.org/) |
| TLS 1.2와 비교 | [tls12.xargs.org](https://tls12.xargs.org/) |

**오늘 관찰한 메시지 순서와 나란히 놓고 보십시오.**

---

## 제7절. 검증이 실패하는 세 가지

### 7.1 ① 발급자를 모른다

```bash
echo | openssl s_client -connect 127.0.0.1:4433 -servername c-srv
```

```text
verify error:num=20:unable to get local issuer certificate
verify return:1
verify error:num=21:unable to verify the first certificate
verify return:1
Verification error: unable to verify the first certificate
Verify return code: 21 (unable to verify the first certificate)
```

`-CAfile`을 주지 않았습니다. **우리 CA는 시스템 신뢰 목록에 없습니다.**

| 오류 | 뜻 |
|---|---|
| **20** | 발급자 인증서를 찾을 수 없다 |
| **21** | 첫 인증서를 검증할 수 없다 |

**브라우저에서 보는 "이 연결은 비공개가 아닙니다"가 이것**입니다.

### 7.2 ② 이름이 맞지 않는다

```bash
echo | openssl s_client -connect 127.0.0.1:4433 -CAfile ca/ca.crt \
  -servername wrong-name -verify_hostname wrong-name
```

```text
verify error:num=62:hostname mismatch
Verify return code: 62 (hostname mismatch)
```

**사슬은 완벽합니다.** CA도 신뢰합니다. 그런데 **이름이 SAN 목록에 없습니다**(3.4절에 `c-srv`, `localhost`, `127.0.0.1`만 넣었습니다).

> **이것이 가장 흔히 빠뜨리는 검사입니다.**
> 사슬 검증만 하고 이름 검사를 하지 않으면, **다른 사이트의 정상 인증서**를 들고 온 중간자에게 뚫립니다. 공격자도 자기 도메인의 진짜 인증서는 얼마든지 받을 수 있기 때문입니다.
>
> **15강에서 C 코드로 이 검사를 켜는 법**을 배웁니다. 켜지 않으면 기본값은 **검사하지 않음**입니다.
{: .prompt-danger }

### 7.3 ③ 기간이 지났다

```bash
openssl x509 -req -in server.csr -CA ca/ca.crt -CAkey ca/ca.key \
  -CAcreateserial -days -1 -extfile server.ext -out expired.crt
```

`-days -1`로 **이미 지난 인증서**를 만듭니다.

```bash
openssl verify -CAfile ca/ca.crt expired.crt
```

```text
error 10 at 0 depth lookup: certificate has expired
```

| 오류 | 뜻 |
|---|---|
| **10** | 기간 만료 |
| 9 | 아직 유효하지 않음(시각이 틀린 경우) |

**VM의 시각이 틀리면 정상 인증서도 거절됩니다.** 1강에서 시각 동기를 확인한 이유가 이것입니다.

### 7.4 정리

| 오류 번호 | 뜻 | 흔한 원인 |
|---|---|---|
| 10 | 만료 | 갱신을 잊음, **시각 오차** |
| 18 | 자체 서명 | 자체 인증서를 그냥 씀 |
| **20** | **발급자 없음** | CA를 신뢰 목록에 안 넣음 |
| 21 | 첫 인증서 검증 실패 | 중간 CA를 안 보냄 |
| **62** | **이름 불일치** | SAN 누락, 잘못된 이름 |

> **오류 번호를 외우실 필요는 없습니다.** `openssl errstr`이나 `man 1 verify`로 찾을 수 있습니다. **어떤 검사가 있는지**를 아는 것이 중요합니다.
{: .prompt-tip }

---

## 제8절. 실제 인터넷 서버

### 8.1 사슬 보기

```bash
echo | openssl s_client -connect www.kisa.or.kr:443 -servername www.kisa.or.kr
```

```text
 0 s:C = KR, ST = Jeollanam-do, L = Naju-si, O = Korea Internet and Security Agency, CN = *.kisa.or.kr
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = GeoTrust TLS RSA CA G1
 1 s:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = GeoTrust TLS RSA CA G1
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root G2
 2 s:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root G2
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root G2
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Verify return code: 0 (ok)
```

### 8.2 읽는 법

`s:`는 subject, `i:`는 issuer입니다. **아래에서 위로 이어집니다.**

| 단계 | 주인 | 발급자 |
|---|---|---|
| 2 (**뿌리**) | DigiCert Global Root G2 | **자기 자신** |
| 1 (중간) | GeoTrust TLS RSA CA G1 | ← 2번이 서명 |
| 0 (**서버**) | `*.kisa.or.kr` | ← 1번이 서명 |

| 관찰 | |
|---|---|
| **뿌리의 s와 i가 같다** | 3.2절에서 우리가 만든 것과 같은 구조 |
| **중간 CA가 있다** | 4.4절에서 설명한 그것 |
| `*.kisa.or.kr` | **와일드카드** — 한 단계 하위 이름 전부 |
| `TLS_AES_256_GCM_SHA384` | 우리 서버와 **같은 방식**(5.2절) |
| `Verify return code: 0 (ok)` | 시스템 신뢰 목록에 DigiCert가 있다 |

**우리가 만든 두 단계 사슬이 실제로는 세 단계일 뿐, 구조가 똑같습니다.**

### 8.3 와일드카드

`*.kisa.or.kr`은 `www.kisa.or.kr`에는 맞지만 `a.b.kisa.or.kr`에는 **맞지 않습니다.** **한 단계만** 대신합니다.

### 8.4 몇 군데 더 해 보십시오

```bash
echo | openssl s_client -connect www.google.com:443 -servername www.google.com
```

| 볼 것 | |
|---|---|
| 사슬이 몇 단계인가 | |
| 어떤 CA인가 | |
| 서명 알고리즘 | ECDSA인가 RSA인가 |
| **유효 기간이 얼마나 짧은가** | 요즘은 몇 달짜리가 많다 |

**유효 기간이 짧아지는 이유**는, 폐기 통보가 잘 전달되지 않기 때문입니다. **짧게 발급하고 자주 갱신**하는 편이 안전하다고 판단한 것입니다.

---

## 제9절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| `unable to get local issuer certificate` | CA 미신뢰 | `-CAfile` 또는 신뢰 목록에 설치 |
| **`hostname mismatch`** | SAN에 이름 없음 | 3.4절에 이름 추가 |
| **이름 검사를 안 함** | 기본값이 꺼짐 | **15강에서 켠다** |
| `certificate has expired` | 만료 또는 **시각 오차** | 갱신, 시각 동기 |
| 브라우저만 거부 | SAN 없이 CN만 씀 | **SAN 필수**(2.2절) |
| 중간 CA에서 끊김 | 서버가 사슬을 다 안 보냄 | 서버 인증서 + 중간 CA를 함께 |
| 서버 인증서로 발급이 됨 | `CA:FALSE` 누락 | 3.4절 |
| CA 키 유출 | 권한·보관 | **0600**, 분리 보관 |
| CSR에 개인키가 들어감 | 오해 | CSR에는 **공개키만** |
| 검증 결과를 무시 | 반환값 미검사 | **15강 핵심** |

---

> **▶ 여기서부터 3회차 — 스스로 해 보기**
> 실습문제, 약 110분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 실습문제

> **안내**
> 1. 명령은 그대로 따라 하되, **출력의 각 줄이 무슨 뜻인지** 설명할 수 있어야 합니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 |
|---|---|---|
| 문제 1 | 자체 CA 만들기 | 3 |
| 문제 2 | 서버 인증서 발급 | 3 |
| 문제 3 | **사슬 검증** | 4 |
| 문제 4 | **핸드셰이크 관찰** | 6 |
| 문제 5 | **검증 실패 세 가지** | 7 |
| 문제 6 | 중간 CA 만들기 | 4.4 |
| 문제 7 | CA를 시스템에 설치 | 4 |
| 문제 8 | 실제 서버 사슬 읽기 | 8 |
| 문제 9 | 인증서 전체 내용 읽기 | 2 |
| 문제 10 | RFC 9846에서 찾기 | 6.1 |

---

### 문제 1·2·3. CA·인증서·사슬

자체 CA를 만들고, 서버 인증서를 발급하고, 검증하십시오.

**정답 및 해설**

제3·4절의 명령과 결과가 답입니다.

```text
server.crt: OK
```

- **`subject`와 `issuer`의 관계**를 확인하십시오. CA는 같고, 서버는 다릅니다.
- **`CA:FALSE`를 빼고** 인증서를 만들어, 그것으로 또 다른 인증서를 발급해 보십시오. 됩니다. **이것이 왜 위험한지** 설명할 수 있어야 합니다.
- `ca.key`의 권한을 반드시 `0600`으로 하십시오(3강 4.5절).
- `-CAcreateserial`이 만드는 `.srl` 파일은 **일련번호**를 기억합니다. 같은 일련번호로 두 인증서를 발급하면 안 되기 때문입니다.

---

### 문제 4. 핸드셰이크 관찰

TLS 서버를 띄우고 **메시지 순서**를 관찰해, 각 메시지가 무슨 일을 하는지 적으십시오.

**정답 및 해설**

6.2절의 결과가 답입니다.

```text
>>> ClientHello
<<< ServerHello
<<< EncryptedExtensions      ← 여기부터 암호화
<<< Certificate
<<< CertificateVerify
<<< Finished
```

- **`InnerContent`가 붙은 지점부터 암호화**입니다(6.5절). 이 표시를 찾아내는 것이 이 문제의 핵심입니다.
- `-brief` 출력의 `X25519`·`Ed25519`·`AES_256_GCM`이 **13강·12강에서 만든 것과 같다**는 점을 확인하십시오.
- **`Certificate`가 암호화된다**는 것이 TLS 1.3의 개선점입니다. TLS 1.2와 비교해 보십시오.

```bash
echo | openssl s_client -connect 127.0.0.1:4433 -tls1_2 -CAfile ca/ca.crt -msg 2>&1 | grep -E "^(>>>|<<<)"
```

- 서버가 TLS 1.2를 지원하지 않으면 실패합니다. 그때는 [tls12.xargs.org](https://tls12.xargs.org/)로 비교하십시오.

---

### 문제 5. 검증 실패 세 가지

**발급자 없음·이름 불일치·기간 만료**를 각각 재현하십시오.

**정답 및 해설**

7.1·7.2·7.3절의 결과가 답입니다.

| 경우 | 오류 |
|---|---|
| CA 없이 | **20**, 21 |
| 이름 다름 | **62** |
| 기간 지남 | **10** |

- **②가 가장 중요합니다.** 사슬은 완벽한데 이름만 틀렸습니다. **이 검사를 빼먹은 프로그램이 실무에 아주 많습니다.**
- `-verify_hostname`을 **빼면** 어떻게 되는지도 해 보십시오. `s_client`는 기본적으로 이름을 검사하지 않습니다. **라이브러리도 마찬가지입니다**(15강).
- 오류 번호는 이렇게 찾습니다.

```bash
openssl errstr 20
```

---

### 문제 6. 중간 CA 만들기

**뿌리 → 중간 → 서버**의 세 단계 사슬을 만드십시오.

**정답 및 해설**

```bash
openssl genpkey -algorithm ED25519 -out sub.key
openssl req -new -key sub.key -subj "/C=KR/O=cmid-lab/CN=cmid-lab Sub CA" -out sub.csr
```

```bash
cat > sub.ext <<'EOF'
basicConstraints = critical, CA:TRUE, pathlen:0
keyUsage = critical, keyCertSign, cRLSign
EOF
```

```bash
openssl x509 -req -in sub.csr -CA ca/ca.crt -CAkey ca/ca.key \
  -CAcreateserial -days 1825 -extfile sub.ext -out sub.crt
```

- **`CA:TRUE`** 여야 다른 인증서를 발급할 수 있습니다. 서버 인증서와 반대입니다.
- **`pathlen:0`** 은 "내 아래로 또 다른 CA를 두지 못한다"는 뜻입니다. 사슬 길이를 제한합니다.
- 검증할 때 **중간 CA를 함께 주어야** 합니다.

```bash
openssl verify -CAfile ca/ca.crt -untrusted sub.crt server2.crt
```

- **`-untrusted`** 는 "사슬을 잇는 데 쓰되 신뢰하지는 말라"는 뜻입니다. 신뢰는 뿌리에서만 옵니다.
- 서버는 **자기 인증서와 중간 CA를 함께** 보내야 합니다. 이것을 빼먹는 것이 흔한 실수입니다(9절 표).

---

### 문제 7. CA를 시스템에 설치

우리 CA를 시스템 신뢰 목록에 넣어, `-CAfile` 없이도 검증되게 하십시오.

**정답 및 해설**

```bash
sudo cp ca/ca.crt /usr/local/share/ca-certificates/cmid-lab.crt
sudo update-ca-certificates
```

```text
1 added, 0 removed; done.
```

```bash
echo | openssl s_client -connect 127.0.0.1:4433 -servername c-srv -brief
```

이제 `-CAfile` 없이도 `Verification: OK`가 나옵니다.

- **확장자가 `.crt`여야 합니다.** `.pem`은 무시됩니다.
- 실습이 끝나면 **반드시 지우십시오.**

```bash
sudo rm /usr/local/share/ca-certificates/cmid-lab.crt && sudo update-ca-certificates --fresh
```

> **이것이 회사나 학교의 "보안 검사 장비"가 하는 일**입니다. 자기 CA를 모든 직원 컴퓨터에 설치하면, 그 장비는 **모든 HTTPS 통신을 합법적인 중간자로서** 열어 볼 수 있습니다. 13강 4.5절에서 말한 "중간 검사 장비"의 정체입니다.
>
> 그래서 **신뢰 목록에 무엇이 들어 있는지**가 매우 중요합니다.
{: .prompt-warning }

---

### 문제 8·9. 인증서 읽기

실제 서버의 사슬을 읽고, 인증서 전체 내용을 확인하십시오.

**정답 및 해설**

8.1절의 결과가 답입니다.

```bash
openssl x509 -in server.crt -noout -text
```

```bash
echo | openssl s_client -connect www.google.com:443 -servername www.google.com \
  -showcerts 2>/dev/null | openssl x509 -noout -text | head -30
```

- **`-showcerts`** 는 사슬 전체를 PEM으로 보여 줍니다.
- 볼 것: `Signature Algorithm`, `Validity`, `Subject Alternative Name`, `Basic Constraints`.
- **SAN에 이름이 몇 개나 들어 있는지** 세어 보십시오. 큰 서비스는 수십 개인 경우도 있습니다.
- 인증서에는 **공개키만** 들어 있습니다. `Public-Key:` 항목을 확인하십시오.

---

### 문제 10. RFC 9846에서 찾기

표준 원문에서 다음을 찾아 절 번호를 적으십시오.

1. 핸드셰이크 메시지의 종류 목록
2. `CertificateVerify`가 무엇에 서명하는가
3. `legacy_record_version`

**정답 및 해설**

[RFC 9846](https://www.rfc-editor.org/rfc/rfc9846)을 여시고 목차에서 찾으십시오.

| 찾을 것 | 실마리 |
|---|---|
| 메시지 종류 | `enum { ... } HandshakeType;` 를 검색 |
| CertificateVerify | 4장 Handshake Protocol 아래 |
| legacy_record_version | 5장 Record Protocol 아래 |

- **표준 문서를 두려워하지 마십시오.** 처음부터 읽을 필요가 없습니다. **목차에서 찾아 필요한 절만** 읽는 것이 올바른 사용법입니다.
- RFC는 **본문 검색이 잘 됩니다.** 브라우저의 찾기로 `CertificateVerify`를 검색하면 정의된 곳으로 갑니다.
- **가장 중요한 습관:** RFC 8446을 인용한 자료를 보면 **폐기 여부를 먼저 확인**하십시오(6.1절). `https://www.rfc-editor.org/info/rfc<번호>`에 상태가 나옵니다.

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 명령 기록 — CA 생성부터 검증까지 |
| 2 | CA 인증서 내용 — **subject와 issuer가 같음**(문제 1) |
| 3 | 서버 인증서 내용 — 확장 필드 포함(문제 2) |
| 4 | `openssl verify` 성공·실패 화면(문제 3) |
| 5 | **핸드셰이크 메시지 순서**와 각 메시지의 역할(문제 4) |
| 6 | **검증 실패 세 가지** 화면(문제 5) |
| 7 | 세 단계 사슬 검증 화면(문제 6) |
| 8 | 실제 서버 사슬을 표로 정리(문제 8) |
| 9 | RFC 9846의 절 번호 세 가지(문제 10) |
| 10 | 짧은 서술 ① 인증서가 해결한 문제는 무엇인가 |
| 11 | 짧은 서술 ② TLS 1.3에서 언제부터 암호화되며 왜 그것이 나은가 |
| 12 | 짧은 서술 ③ **이름 검사를 빠뜨리면 왜 위험한가** |

---

## 정리

| 구분 | 핵심 |
|---|---|
| 인증서 | "이 공개키는 이 이름의 것" 을 **제삼자가 서명** |
| 신뢰 앵커 | 결국 **그냥 믿는 지점**이 있어야 한다 |
| X.509 | [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280). PEM은 Base64 텍스트 |
| **SAN** | 이름 검사는 **CN이 아니라 SAN** |
| `CA:FALSE` | 서버 인증서로 발급을 못 하게 |
| 사슬 | 뿌리 → (중간) → 서버. **하나라도 실패하면 전부 실패** |
| CA 개인키 | **가장 중요한 파일.** 유출되면 무엇이든 발급 가능 |
| **TLS 1.3 표준** | **[RFC 9846](https://www.rfc-editor.org/rfc/rfc9846)** (RFC 8446 폐기) |
| 레코드 | 5바이트 앞머리 — **11강 프레이밍과 같은 구조** |
| 메시지 순서 | Hello → **암호화 시작** → Certificate → CertificateVerify → Finished |
| **암호화 시점** | **Hello 직후.** 인증서도 감춰진다 |
| 조립 확인 | `X25519`(13강) + `Ed25519`(13강) + `AES-256-GCM`(12강) |
| 검증 실패 | 20 발급자 없음 · **62 이름 불일치** · 10 만료 |
| 가장 흔한 구멍 | **이름 검사 누락** → 15강 |

---

## 다음 강의 예고

**15강 「OpenSSL로 TLS 구현」** 에서 **C 코드로** TLS 서버와 클라이언트를 만듭니다.

- `SSL_CTX`와 `SSL` — 무엇을 한 번 하고 무엇을 연결마다 하는가
- 9강의 소켓 코드를 **어디까지 그대로 쓰는가**
- **인증서 검증을 켜는 법** — 그리고 켜지 않으면 어떻게 되는가
- **이름 검사를 켜는 법** — 오늘 7.2절의 그 검사
- `SSL_read`/`SSL_write`와 **오류 처리의 함정**
- 오늘 만든 인증서를 그대로 씁니다

**"검증을 켜지 않은 코드"와 "켠 코드"를 나란히 놓고, 13강의 중간자로 시험합니다.**

---

## 부록 A. 이번 강의 명령 요약

| 하려는 일 | 명령 |
|---|---|
| 키 만들기 | `openssl genpkey -algorithm ED25519 -out k.key` |
| **자체 서명 인증서** | `openssl req -x509 -new -key ca.key -days 3650 -subj "/CN=..." -out ca.crt` |
| 발급 요청서 | `openssl req -new -key server.key -subj "/CN=..." -out server.csr` |
| **CA가 서명** | `openssl x509 -req -in s.csr -CA ca.crt -CAkey ca.key -CAcreateserial -extfile s.ext -out s.crt` |
| 인증서 요약 | `openssl x509 -in c.crt -noout -subject -issuer -dates` |
| 확장 필드 | `openssl x509 -in c.crt -noout -ext subjectAltName` |
| 전체 내용 | `openssl x509 -in c.crt -noout -text` |
| **사슬 검증** | `openssl verify -CAfile ca.crt server.crt` |
| 중간 CA 포함 | `openssl verify -CAfile ca.crt -untrusted sub.crt s.crt` |
| TLS 서버 | `openssl s_server -accept 4433 -cert s.crt -key s.key -www` |
| TLS 접속 | `openssl s_client -connect host:443 -CAfile ca.crt -servername name -brief` |
| **메시지 순서** | `openssl s_client ... -msg` |
| 사슬 전체 | `openssl s_client ... -showcerts` |
| 이름 검사 | `openssl s_client ... -verify_hostname name` |
| 오류 번호 해석 | `openssl errstr 20` |
| CA 설치 | `sudo cp ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates` |

## 부록 B. 표준 문서와 출처

**표준**

| 내용 | 문서 |
|---|---|
| **TLS 1.3 (현행)** | **[RFC 9846](https://www.rfc-editor.org/rfc/rfc9846)** |
| TLS 1.3 (2018, **폐기**) | [RFC 8446](https://www.rfc-editor.org/rfc/rfc8446) — [상태 확인](https://www.rfc-editor.org/info/rfc8446) |
| **X.509 인증서** | [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280) |
| 이름 검증 | [RFC 6125](https://www.rfc-editor.org/rfc/rfc6125) |
| CSR 형식 | [RFC 2986](https://www.rfc-editor.org/rfc/rfc2986) |
| PEM 형식 | [RFC 7468](https://www.rfc-editor.org/rfc/rfc7468) |

**OpenSSL 문서**

| 내용 | 주소 |
|---|---|
| `s_client` | [openssl-s_client](https://docs.openssl.org/3.0/man1/openssl-s_client/) |
| `s_server` | [openssl-s_server](https://docs.openssl.org/3.0/man1/openssl-s_server/) |
| `verify` | [openssl-verify](https://docs.openssl.org/3.0/man1/openssl-verify/) |
| `x509` | [openssl-x509](https://docs.openssl.org/3.0/man1/openssl-x509/) |

**보는 자료**

| 내용 | 주소 |
|---|---|
| TLS 1.3 바이트 주석 | [tls13.xargs.org](https://tls13.xargs.org/) |
| TLS 1.2 바이트 주석 | [tls12.xargs.org](https://tls12.xargs.org/) |

**관련 취약점 분류**

| CWE | 내용 |
|---|---|
| [CWE-295](https://cwe.mitre.org/data/definitions/295.html) | **잘못된 인증서 검증** |
| [CWE-297](https://cwe.mitre.org/data/definitions/297.html) | **호스트 이름 불일치 무시** |
| [CWE-296](https://cwe.mitre.org/data/definitions/296.html) | 신뢰 사슬 검증 미흡 |

**본문의 주장과 근거**

| 주장 | 근거 |
|---|---|
| 뿌리 인증서는 subject=issuer(3.2절) | `openssl x509` — 두 줄이 동일 |
| 발급된 인증서는 issuer가 CA(3.5절) | `issuer=... CN = cmid-lab Root CA` |
| 지정한 확장이 들어간다(3.5절) | `CA:FALSE`·`serverAuth`·SAN 3개 |
| **사슬 검증이 성공/실패한다**(4.1·4.2절) | `server.crt: OK` / `error 20` |
| **TLS가 12·13강의 재료를 쓴다**(5.2절) | `AES_256_GCM` · `X25519` · `Ed25519` |
| **핸드셰이크 메시지 순서**(6.2절) | `s_client -msg` 실제 출력 |
| **Hello 이후가 암호화된다**(6.5절) | `InnerContent` 표시가 붙는 지점 |
| **RFC 8446은 폐기, RFC 9846이 현행**(6.1절) | [RFC Editor 안내 페이지](https://www.rfc-editor.org/info/rfc8446) |
| 이름이 다르면 오류 62(7.2절) | `Verify return code: 62 (hostname mismatch)` |
| **실제 사슬은 3단계**(8.1절) | `www.kisa.or.kr` → GeoTrust TLS RSA CA G1 → DigiCert Global Root G2 |

> 인증서의 날짜·일련번호는 만든 시점에 따라 달라집니다. **관계**(subject와 issuer가 이어지는가, 검증이 통과하는가)를 보십시오. **실제 인터넷 서버의 인증서는 갱신되므로** 사슬의 구성이 달라질 수 있습니다. 위 결과는 2026년 8월에 확인한 것입니다. **구조가 같은지**를 보시면 됩니다.
{: .prompt-tip }

## 부록 C. 이번 강의 산출 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `ca/ca.key` | **CA 개인키 — 0600** | 3.1 |
| `ca/ca.crt` | CA 인증서(자체 서명) | 3.2 |
| `server.key` | 서버 개인키 — 0600 | 3.3 |
| `server.csr` | 발급 요청서 | 3.3 |
| `server.ext` | **확장 필드 — SAN 포함** | 3.4 |
| `server.crt` | **서버 인증서** — 15강에서 사용 | 3.5 |
| `sub.key`·`sub.crt` | 중간 CA | 문제 6 |
| `expired.crt` | 만료된 인증서 | 7.3 |
