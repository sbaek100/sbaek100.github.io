---
title: 중급 C 프로그래밍 15강 - OpenSSL로 TLS 구현
date: 2027-03-01 09:00:00 +0900
categories:
  - 1.응용강의
  - 중급 C 프로그래밍
tags:
  - C언어
  - 시스템프로그래밍
  - 리눅스
  - TLS
  - OpenSSL
  - 인증서검증
  - 네트워크보안
pin:
mermaid: false
---

> **학습 목표**
> 1. `SSL_CTX`와 `SSL`의 역할을 구분할 수 있다.
> 2. **TLS 서버**를 C로 만들 수 있다.
> 3. **TLS 클라이언트**를 C로 만들 수 있다.
> 4. 9강의 소켓 코드를 **어디까지 그대로 쓰는지** 설명할 수 있다.
> 5. **인증서 검증을 켤** 수 있다(`SSL_CTX_set_verify`).
> 6. **이름 검사를 켤** 수 있다(`SSL_set1_host`).
> 7. **검증을 켜지 않은 클라이언트가 가짜 서버에 비밀번호를 넘기는 것**을 실험으로 보일 수 있다.
> 8. `SSL_read`/`SSL_write`의 오류 처리를 바르게 할 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

14강까지 개념과 도구를 갖추었습니다. **오늘 C 코드로 만듭니다.**

| 강의 | 만든 것 |
|---|---|
| 9강 | TCP 서버·클라이언트 — **평문** |
| 12강 | 손으로 만든 암호 채널 |
| 13강 | 키 교환·서명 |
| 14강 | 인증서·자체 CA |
| **15강** | **TLS 서버·클라이언트** |

**14강에서 만든 인증서를 그대로 씁니다.**

> **이번 강의의 결론을 먼저 말씀드립니다.**
> **TLS를 쓴다고 안전해지지 않습니다.** 검증을 켜야 안전해집니다. 그리고 **기본값은 꺼져 있습니다.** 제6절에서 이것을 실험으로 확인합니다.
{: .prompt-danger }

이 강의는 **3회차 분량**(모두 합쳐 약 470분)입니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제4절 | `SSL_CTX`·서버·클라이언트 만들기 | 180분 |
| **2회차** | 제5절 ~ 제9절 | **검증**·**가짜 서버 실험**·오류 처리 | 180분 |
| **3회차** | 실습문제 | 스스로 해 보기 | 110분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 준비 — 두 개의 구조체 | 25분 |
| 제2절 | TLS 서버 만들기 | 55분 |
| 제3절 | TLS 클라이언트 만들기 | 50분 |
| 제4절 | 정상 동작 확인 | 30분 |
| 제5절 | **검증을 켜는 세 줄** | 40분 |
| 제6절 | **검증을 끄면 무슨 일이 일어나는가** | 55분 |
| 제7절 | 이름 검사 | 35분 |
| 제8절 | 오류 처리 | 35분 |
| 제9절 | 자주 나오는 실수 | 20분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 디렉터리**는 `~/cmid/lab15`, **VM 두 대**를 씁니다.

```bash
mkdir -p ~/cmid/lab15 && cd ~/cmid/lab15
cp -r ~/cmid/lab14/ca ~/cmid/lab14/server.crt ~/cmid/lab14/server.key .
```

---

## 제1절. 준비 — 두 개의 구조체

### 1.1 `SSL_CTX`와 `SSL`

| 구조체 | 언제 만드나 | 무엇을 담나 |
|---|---|---|
| **`SSL_CTX`** | **프로그램에 한 번** | 인증서, 키, 신뢰 CA, 설정 |
| **`SSL`** | **연결마다** | 이 연결의 상태 |

```text
   SSL_CTX  (하나)
      ├── SSL  ← 연결 1
      ├── SSL  ← 연결 2
      └── SSL  ← 연결 3
```

> **7강 4절에서 배운 "공유되는 것과 각자의 것"** 과 같은 구분입니다. `SSL_CTX`는 여러 연결이 공유하고, `SSL`은 연결마다 하나씩입니다.
>
> **연결마다 `SSL_CTX`를 만들면** 인증서 파일을 매번 읽고 검증하므로 느려집니다. 흔한 성능 실수입니다.
{: .prompt-tip }

### 1.2 컴파일

```bash
gcc -Wall -Wextra -std=gnu17 tlsserv.c -o tlsserv -lssl -lcrypto
```

### 1.3 OpenSSL 3.0에서는 초기화가 필요 없습니다

옛 자료에는 이런 줄이 있습니다.

```c
SSL_library_init();                 /* 1.1.0 이후 불필요 */
OpenSSL_add_all_algorithms();       /* 불필요 */
```

**OpenSSL 1.1.0부터 자동으로 초기화됩니다.** 옛 코드를 베낄 때 주의하십시오.

---

## 제2절. TLS 서버 만들기

### 2.1 설정 만들기

```c
/* make_ctx: 연결마다가 아니라 프로그램 시작에 한 번만 만든다 */
static SSL_CTX *make_ctx(const char *cert, const char *key)
{
    SSL_CTX *ctx = SSL_CTX_new(TLS_server_method());

    if (ctx == NULL)
        die_ssl("SSL_CTX_new 실패");

    /* TLS 1.2 이하를 아예 받지 않는다 */
    if (SSL_CTX_set_min_proto_version(ctx, TLS1_3_VERSION) != 1)
        die_ssl("최소 판 설정 실패");

    if (SSL_CTX_use_certificate_chain_file(ctx, cert) != 1)
        die_ssl("인증서를 읽을 수 없습니다");
    if (SSL_CTX_use_PrivateKey_file(ctx, key, SSL_FILETYPE_PEM) != 1)
        die_ssl("개인키를 읽을 수 없습니다");
    /* 인증서와 개인키가 짝인지 확인한다 */
    if (SSL_CTX_check_private_key(ctx) != 1)
        die_ssl("인증서와 개인키가 짝이 아닙니다");

    return ctx;
}
```

| 줄 | 왜 |
|---|---|
| `TLS_server_method()` | **`TLSv1_2_method()` 같은 것을 쓰지 마십시오.** 판은 아래 줄에서 정합니다 |
| `set_min_proto_version` | **옛 판을 아예 받지 않는다** |
| **`use_certificate_chain_file`** | `use_certificate_file`이 아닙니다 — **중간 CA까지** 보냅니다 |
| `check_private_key` | 짝이 맞는지 미리 확인 |

> **`use_certificate_chain_file`을 쓰십시오.**
> `SSL_CTX_use_certificate_file`은 **첫 인증서만** 보냅니다. 중간 CA가 있으면 클라이언트가 사슬을 잇지 못해 14강 7.1절의 오류 21이 납니다. **"우리는 되는데 저쪽만 안 된다"** 의 흔한 원인입니다.
{: .prompt-warning }

### 2.2 소켓은 9강 그대로입니다

```c
/* listen_on: 9강에서 만든 것과 같다. TLS 와 무관하다 */
static int listen_on(const char *port)
{
    ...
        setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);
        if (bind(fd, p->ai_addr, p->ai_addrlen) == 0 && listen(fd, 16) == 0)
            break;
    ...
}
```

**`socket`·`bind`·`listen`·`accept`는 하나도 달라지지 않습니다.** TLS는 **연결된 다음에** 시작합니다.

### 2.3 연결마다 하는 일

```c
        /* 연결마다 SSL 객체를 하나 만든다 */
        ssl = SSL_new(ctx);
        if (ssl == NULL) {
            close(cfd);
            continue;
        }
        SSL_set_fd(ssl, cfd);

        if (SSL_accept(ssl) != 1) {
            fprintf(stderr, "  핸드셰이크 실패\n");
            ERR_print_errors_fp(stderr);
        } else {
```

| 함수 | 하는 일 |
|---|---|
| `SSL_new(ctx)` | 이 연결의 상태를 만든다 |
| `SSL_set_fd(ssl, cfd)` | **이미 연결된 서술자**를 붙인다 |
| **`SSL_accept`** | **핸드셰이크를 수행한다**(14강 6.2절) |

**`SSL_accept`가 14강에서 관찰한 그 핸드셰이크입니다.** `ServerHello`·`Certificate`·`CertificateVerify`·`Finished`가 이 한 줄 안에서 오갑니다.

### 2.4 주고받기

```c
            while ((n = SSL_read(ssl, buf, sizeof buf - 1)) > 0) {
                buf[n] = '\0';
                printf("  받음: %s", buf);
                fflush(stdout);
                if (SSL_write(ssl, buf, n) <= 0)
                    break;
            }
            SSL_shutdown(ssl);
```

**`read`/`write`가 `SSL_read`/`SSL_write`로 바뀐 것뿐**입니다. 9강의 에코 서버와 구조가 같습니다.

### 2.5 정리

```c
        SSL_free(ssl);
        close(cfd);
```

**`SSL_free`를 먼저, `close`를 나중에** 합니다. 순서를 지키십시오.

---

## 제3절. TLS 클라이언트 만들기

### 3.1 연결도 9강 그대로입니다

```c
/* connect_to: 9강에서 만든 것과 같다. TLS 와 무관하다 */
static int connect_to(const char *host, const char *port)
{
    ...
        if (connect(fd, p->ai_addr, p->ai_addrlen) == 0)
            break;
    ...
}
```

### 3.2 SNI — 어느 이름으로 접속하는지 알린다

```c
    ssl = SSL_new(ctx);
    SSL_set_fd(ssl, fd);
    /* SNI: 어느 이름으로 접속하는지 알린다 (14강 -servername) */
    SSL_set_tlsext_host_name(ssl, host);
```

**14강의 `-servername`이 이것**입니다. 한 IP에 여러 사이트가 있을 때, 서버가 **어느 인증서를 보내야 하는지** 알려 줍니다.

**이것을 빠뜨리면** 서버가 엉뚱한 인증서를 보내 검증이 실패합니다.

### 3.3 핸드셰이크

```c
    if (SSL_connect(ssl) != 1) {
        long v = SSL_get_verify_result(ssl);

        fprintf(stderr, "핸드셰이크 실패\n");
        fprintf(stderr, "  검증 결과: %ld — %s\n", v,
                X509_verify_cert_error_string(v));
        ERR_print_errors_fp(stderr);
        ...
    }
```

**`X509_verify_cert_error_string`** 이 14강 7.4절의 오류 번호를 사람이 읽을 말로 바꿔 줍니다.

### 3.4 상대 인증서 확인

```c
        X509 *cert = SSL_get1_peer_certificate(ssl);
        long v = SSL_get_verify_result(ssl);

        if (cert != NULL) {
            X509_NAME_oneline(X509_get_subject_name(cert), name, sizeof name);
            printf("  상대 인증서 주인: %s\n", name);
            X509_NAME_oneline(X509_get_issuer_name(cert), name, sizeof name);
            printf("  발급자           : %s\n", name);
            X509_free(cert);
        }
```

> **`SSL_get1_peer_certificate`의 `1`** 은 "참조 횟수를 하나 올린다"는 뜻입니다. 그래서 **`X509_free`로 돌려주어야** 합니다. OpenSSL에는 `_get0_`(빌려 쓰기, 반납 불필요)과 `_get1_`(가져오기, 반납 필요)이 짝으로 있습니다.
>
> **OpenSSL 3.0에서 이름이 바뀌었습니다.** 옛 이름은 `SSL_get_peer_certificate`였습니다. 옛 코드를 볼 때 참고하십시오.
{: .prompt-info }

---

## 제4절. 정상 동작 확인

### 4.1 서버 띄우기 (VM1)

```bash
./tlsserv 7401 server.crt server.key
```

```text
TLS 서버가 포트 7401 에서 기다립니다
```

### 4.2 접속하기 (VM2)

```bash
./tlscli 127.0.0.1 7401 ca/ca.crt
```

```text
[검증 켬] CA=ca/ca.crt
연결됨: TLSv1.3 / TLS_AES_256_GCM_SHA384
  상대 인증서 주인: /C=KR/O=cmid-lab/CN=c-srv
  발급자           : /C=KR/O=cmid-lab/CN=cmid-lab Root CA
  검증 결과        : 0 — ok
비밀번호는 admin1234 입니다
echo: 비밀번호는 admin1234 입니다
계좌번호 110-123-456789
echo: 계좌번호 110-123-456789
```

서버 쪽입니다.

```text
접속: 127.0.0.1:41440
  TLSv1.3 / TLS_AES_256_GCM_SHA384
  받음: 비밀번호는 admin1234 입니다
  받음: 계좌번호 110-123-456789
연결 종료
```

### 4.3 확인할 것

| 줄 | 무엇을 뜻하나 |
|---|---|
| `TLSv1.3` | 2.1절에서 최소 판을 지정했다 |
| **`TLS_AES_256_GCM_SHA384`** | **12강에서 손으로 만든 그것** |
| 발급자가 `cmid-lab Root CA` | **14강에서 만든 CA** |
| **`검증 결과: 0 — ok`** | 사슬이 이어졌다 |

### 4.4 `openssl s_client`로도 접속됩니다

```bash
echo | openssl s_client -connect 127.0.0.1:7405 -CAfile ca/ca.crt -brief
```

```text
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: C = KR, O = cmid-lab, CN = c-srv
Hash used: UNDEF
Signature type: Ed25519
Verification: OK
Server Temp Key: X25519, 253 bits
DONE
```

**우리가 만든 서버가 표준 도구와 통합니다.** 1부부터 써 온 "정답이 있는 것과 맞춰 보기" 검증입니다.

---

> **▶ 여기서부터 2회차 — 검증·가짜 서버 실험·오류 처리**
> 제5절 ~ 제9절, 약 180분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제5절. 검증을 켜는 세 줄

**클라이언트에서 반드시 해야 할 세 가지입니다.**

### 5.1 ① 어떤 CA를 믿을 것인가

```c
        /* ① 어떤 CA 를 믿을 것인가 */
        if (SSL_CTX_load_verify_locations(ctx, cafile, NULL) != 1) {
            fprintf(stderr, "CA 파일 %s 를 읽을 수 없습니다\n", cafile);
            return 1;
        }
```

**시스템 신뢰 목록을 쓰려면** 이렇게 합니다.

```c
SSL_CTX_set_default_verify_paths(ctx);
```

### 5.2 ② 검증 실패를 핸드셰이크 실패로

```c
        /* ② 검증에 실패하면 핸드셰이크를 실패시킨다 */
        SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, NULL);
```

> **이 한 줄이 없으면 검증 결과를 알기만 하고 그냥 진행합니다.**
> **기본값이 `SSL_VERIFY_NONE`** 입니다. 다음 절에서 그 결과를 봅니다.
{: .prompt-danger }

### 5.3 ③ 이름 검사

```c
        /* ③ 이름 검사 — 이것을 빠뜨리는 것이 가장 흔한 구멍이다 */
        if (SSL_set1_host(ssl, host) != 1) {
            fprintf(stderr, "이름 설정 실패\n");
            return 1;
        }
```

**①②만 하고 ③을 빠뜨리는 것이 가장 흔합니다.** 사슬은 검증하지만 **이름은 보지 않는** 상태가 됩니다(제7절).

### 5.4 세 줄의 역할

| 줄 | 없으면 |
|---|---|
| ① `load_verify_locations` | 믿을 CA가 없어 **모두 실패** |
| ② **`set_verify(SSL_VERIFY_PEER)`** | **아무 인증서나 통과** |
| ③ **`SSL_set1_host`** | **다른 사이트의 정상 인증서로 통과** |

---

## 제6절. 검증을 끄면 무슨 일이 일어나는가

**이 절이 이번 강의에서 가장 중요합니다.**

### 6.1 공격자가 되어 봅니다

**14강에서 배운 대로 자기 CA를 만들고, `c-srv` 이름의 인증서를 발급합니다.**

```bash
openssl genpkey -algorithm ED25519 -out evil/ca.key
openssl req -x509 -new -key evil/ca.key -days 3650 \
  -subj "/C=KR/O=evil-corp/CN=evil Root CA" -out evil/ca.crt
```

```bash
openssl genpkey -algorithm ED25519 -out evil/fake.key
openssl req -new -key evil/fake.key -subj "/C=KR/O=evil-corp/CN=c-srv" -out evil/fake.csr
openssl x509 -req -in evil/fake.csr -CA evil/ca.crt -CAkey evil/ca.key \
  -CAcreateserial -days 365 -extfile server.ext -out evil/fake.crt
```

**이름은 `c-srv`로 똑같습니다.** SAN도 똑같습니다. **다른 것은 발급자뿐**입니다.

> **누구나 이렇게 할 수 있습니다.** 인증서를 만드는 데는 아무 권한도 필요 없습니다. **"인증서가 있다"는 것 자체는 아무 의미가 없습니다.** 중요한 것은 **누가 발급했는가**입니다.
{: .prompt-warning }

### 6.2 가짜 서버를 띄웁니다

```bash
./tlsserv 7402 evil/fake.crt evil/fake.key
```

### 6.3 검증하는 클라이언트로 접속

```bash
./tlscli 127.0.0.1 7402 ca/ca.crt
```

```text
[검증 켬] CA=ca/ca.crt
핸드셰이크 실패
  검증 결과: 20 — unable to get local issuer certificate
...:SSL routines:tls_post_process_server_certificate:certificate verify failed:...
```

**막혔습니다.** 오류 20 — 14강 7.1절의 그 오류입니다.

서버 쪽에도 흔적이 남습니다.

```text
접속: 127.0.0.1:48428
  핸드셰이크 실패
...:SSL routines:ssl3_read_bytes:tlsv1 alert unknown ca:...:SSL alert number 48
```

**`tlsv1 alert unknown ca`** — 클라이언트가 **"당신 CA를 모릅니다"라는 경고를 보내고** 끊은 것입니다. TLS에는 이런 경고 메시지가 정의되어 있습니다([RFC 9846](https://www.rfc-editor.org/rfc/rfc9846) 6장).

> 출력 순서가 뒤섞여 보일 수 있습니다. **표준 출력은 파이프로 연결되면 블록 버퍼**가 되고 표준 오류는 버퍼가 없기 때문입니다(1부 6강). `fflush(stdout)`을 쓰거나 화면에서 직접 실행하면 순서대로 나옵니다.
{: .prompt-tip }

### 6.4 검증하지 않는 클라이언트로 접속

**`SSL_CTX_set_verify` 한 줄만 빼고** 같은 가짜 서버에 접속합니다.

```bash
./tlscli 127.0.0.1 7403 ca/ca.crt noverify
```

```text
[검증 끔] 어떤 인증서든 받아들입니다 — 절대 이렇게 쓰지 마십시오
연결됨: TLSv1.3 / TLS_AES_256_GCM_SHA384
  상대 인증서 주인: /C=KR/O=evil-corp/CN=c-srv
  발급자           : /C=KR/O=evil-corp/CN=evil Root CA
  검증 결과        : 20 — unable to get local issuer certificate
echo: 비밀번호는 admin1234 입니다
echo: 계좌번호 110-123-456789
```

**가짜 서버가 읽은 것입니다.**

```text
접속: 127.0.0.1:45220
  TLSv1.3 / TLS_AES_256_GCM_SHA384
  받음: 비밀번호는 admin1234 입니다
  받음: 계좌번호 110-123-456789
연결 종료
```

### 6.5 이 화면을 오래 보십시오

| 관찰 | |
|---|---|
| `TLSv1.3` | **최신 판입니다** |
| `TLS_AES_256_GCM_SHA384` | **최강 암호 방식입니다** |
| 발급자 `evil Root CA` | **공격자의 CA입니다** |
| **`검증 결과: 20`** | **실패했다고 나와 있습니다** |
| **그런데 통신이 되었습니다** | |
| **비밀번호가 공격자에게 넘어갔습니다** | |

> **가장 무서운 것은 `검증 결과: 20` 이 찍혀 있다는 점입니다.**
> 프로그램은 **검증이 실패했다는 것을 알고 있었습니다.** 다만 `SSL_VERIFY_PEER`를 켜지 않았으므로 **그 사실을 무시하고 진행**했습니다.
>
> **TLS를 썼습니다. 최신 판입니다. 강한 암호를 씁니다. 그리고 비밀번호를 공격자에게 넘겼습니다.**
{: .prompt-danger }

### 6.6 9강과 비교하면

| | 9강(평문) | 오늘(검증 끈 TLS) |
|---|---|---|
| 도청자가 읽을 수 있나 | **그렇다** | 아니다 |
| **공격자가 서버 행세를 하면** | 읽힌다 | **읽힌다** |
| 겉보기 | 위험해 보인다 | **안전해 보인다** |

**"안전해 보이는데 안전하지 않은 것"이 더 위험합니다.** 아무도 의심하지 않기 때문입니다.

### 6.7 실무에서 왜 이런 일이 생기는가

| 상황 | 결과 |
|---|---|
| 개발 중 자체 인증서라 오류가 남 | **검증을 끄고 넘어감** |
| "일단 되게 하자" | 그대로 배포 |
| 인터넷의 예제를 베낌 | 예제에 검증이 없음 |
| 라이브러리 기본값을 믿음 | **기본값은 꺼져 있음** |

**개발 중 오류가 나면 검증을 끄지 말고, 자체 CA를 신뢰 목록에 넣으십시오**(14강 문제 7).

---

## 제7절. 이름 검사

### 7.1 사슬만 맞으면 되는 것이 아닙니다

`SSL_set1_host`를 켠 클라이언트로 **SAN에 없는 주소**로 접속해 봅니다.

```bash
./tlscli 127.0.0.2 7404 ca/ca.crt
```

```text
[검증 켬] CA=ca/ca.crt
핸드셰이크 실패
  검증 결과: 64 — IP address mismatch
...:certificate verify failed:...
```

**서버는 진짜입니다. CA도 신뢰합니다. 사슬도 완벽합니다.** 그런데 **이름이 맞지 않아** 거절되었습니다.

14강 3.4절에서 SAN에 넣은 것은 `c-srv`, `localhost`, `127.0.0.1` 뿐입니다. `127.0.0.2`는 없습니다.

| 오류 | 언제 |
|---|---|
| **62** | 이름(DNS) 불일치 — 14강 7.2절 |
| **64** | **주소(IP) 불일치** — 오늘 |

### 7.2 IP로 접속할 때

IP 주소를 명시적으로 검사하려면 전용 함수가 있습니다.

```c
X509_VERIFY_PARAM *p = SSL_get0_param(ssl);

X509_VERIFY_PARAM_set1_ip_asc(p, "192.168.56.60");
```

**실무에서는 이름으로 접속하는 것이 정상**입니다. IP로만 접속해야 한다면 인증서 SAN에 그 IP를 넣어야 합니다.

### 7.3 이름 검사를 빠뜨리면

**공격자는 자기 도메인의 진짜 인증서를 얼마든지 받을 수 있습니다.** 무료로 발급해 주는 곳도 많습니다.

```text
   공격자가 evil.example.com 의 진짜 인증서를 받는다
        ↓
   중간에서 bank.example.com 인 척하며 그 인증서를 내민다
        ↓
   사슬 검증: 통과 (진짜 CA 가 발급했으니까)
   이름 검사: 없음
        ↓
   뚫린다
```

**사슬 검증만으로는 부족합니다.** 그래서 세 줄이 모두 필요합니다(5.4절).

### 7.4 서버가 클라이언트를 확인하려면

서버 쪽에서도 같은 것을 할 수 있습니다.

```c
SSL_CTX_load_verify_locations(ctx, "ca/ca.crt", NULL);
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, NULL);
```

**상호 TLS**(mTLS)라 합니다. 클라이언트도 인증서를 내야 합니다. 서버 사이 통신이나 사내 시스템에 씁니다.

---

## 제8절. 오류 처리

### 8.1 `SSL_read`의 반환값

**`read`와 다릅니다.**

| 반환 | 뜻 |
|---|---|
| **> 0** | 읽은 바이트 수 |
| **≤ 0** | **오류가 아닐 수도 있다** — `SSL_get_error`로 물어야 한다 |

```c
    n = SSL_read(ssl, buf, sizeof buf);
    if (n <= 0) {
        int err = SSL_get_error(ssl, n);

        switch (err) {
        case SSL_ERROR_ZERO_RETURN:     /* 상대가 정상 종료했다 */
            break;
        case SSL_ERROR_WANT_READ:       /* 논블로킹 — 다시 부르면 된다 */
        case SSL_ERROR_WANT_WRITE:
            break;
        case SSL_ERROR_SYSCALL:         /* errno 를 보라 */
        default:
            ERR_print_errors_fp(stderr);
        }
    }
```

### 8.2 `SSL_ERROR_WANT_READ`가 중요합니다

**논블로킹 소켓(10강)에서는 `SSL_write` 중에도 `WANT_READ`가 날 수 있습니다.** TLS가 내부적으로 재협상 등을 하기 때문입니다.

> **`SSL_write`가 `WANT_READ`를 돌려주면 다시 부를 때 같은 자료를 같은 주소로** 주어야 합니다. 그것이 기본 동작입니다. 버퍼를 바꾸면 안 됩니다.
{: .prompt-warning }

### 8.3 10강의 다중 접속과 함께 쓰려면

| 할 일 | |
|---|---|
| `SSL_get_fd(ssl)`로 서술자를 얻어 `epoll`에 등록 | |
| `WANT_READ` → 읽기 대기 등록 | |
| `WANT_WRITE` → **쓰기 대기 등록** | |
| 부분 전송 처리 | 11강 3절 |

**17강에서 이것을 합칩니다.**

### 8.4 `SSL_shutdown`

```c
SSL_shutdown(ssl);
```

**정상 종료를 알리는 메시지를 보냅니다.** 이것 없이 소켓을 닫으면 상대가 **"잘린 것인지 끝난 것인지"** 구분하지 못합니다.

| 반환 | 뜻 |
|---|---|
| 1 | 양쪽 모두 종료 완료 |
| 0 | 내 쪽만 보냈다. **다시 부르면** 상대 것을 기다린다 |
| < 0 | 오류 |

**보내고 바로 닫아도 되는 경우가 많지만**, 상대가 자른 것으로 오해하지 않게 하려면 0이 나왔을 때 한 번 더 부르는 것이 정확합니다.

### 8.5 오류 대기열

OpenSSL은 오류를 **스레드마다 대기열**에 쌓습니다.

```c
ERR_print_errors_fp(stderr);        /* 대기열을 비우며 출력한다 */
```

**오류를 확인하지 않고 두면 다음 오류와 섞입니다.** 실패한 자리에서 바로 꺼내십시오.

---

## 제9절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| **가짜 서버에 뚫림** | `SSL_CTX_set_verify` 누락 | **5.2절** |
| **다른 사이트 인증서로 뚫림** | `SSL_set1_host` 누락 | **5.3절** |
| 상대만 검증 실패 | `use_certificate_file` 사용 | **`_chain_file`**(2.1절) |
| 엉뚱한 인증서를 받음 | SNI 미설정 | `SSL_set_tlsext_host_name`(3.2절) |
| `SSL_read`가 0인데 오류 처리 | 반환값 오해 | `SSL_get_error`(8.1절) |
| 논블로킹에서 자료가 깨짐 | `WANT_WRITE` 뒤 버퍼 교체 | **같은 버퍼**(8.2절) |
| 접속이 느림 | 연결마다 `SSL_CTX` | **한 번만**(1.1절) |
| 메모리 누수 | `X509_free` 누락 | `_get1_`은 반납(3.4절) |
| 개인키를 못 읽음 | 권한 | 0600 + 실행 사용자 확인 |
| 옛 함수가 없다는 오류 | 3.0에서 이름 변경 | `SSL_get1_peer_certificate` |
| 상대가 "잘렸다"고 함 | `SSL_shutdown` 누락 | 8.4절 |
| 오류 메시지가 엉뚱함 | 대기열이 섞임 | 바로 출력(8.5절) |

---

> **▶ 여기서부터 3회차 — 스스로 해 보기**
> 실습문제, 약 110분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 실습문제

> **안내**
> 1. 컴파일은 **`gcc -Wall -Wextra -std=gnu17 ... -lssl -lcrypto`**, **경고 0개**여야 합니다.
> 2. 14강에서 만든 인증서를 씁니다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 |
|---|---|---|
| 문제 1 | TLS 서버 | 2 |
| 문제 2 | TLS 클라이언트 | 3 |
| 문제 3 | **검증 세 줄** | 5 |
| 문제 4 | **가짜 서버 실험** | 6 |
| 문제 5 | 이름 검사 | 7 |
| 문제 6 | 오류 처리 | 8 |
| 문제 7 | 상호 TLS | 7.4 |
| 문제 8 | 시스템 CA로 실제 사이트 접속 | 5.1 |
| 문제 9 | 표준 도구와 상호 검증 | 4.4 |
| 문제 10 | 9강 코드와 비교 | 2 · 3 |

---

### 문제 1·2·3. 서버·클라이언트·검증

TLS 서버와 클라이언트를 만들고, **검증 세 줄**을 모두 넣으십시오.

**정답 및 해설**

제2·3·5절의 코드와 4.2절의 결과가 답입니다.

```text
  검증 결과        : 0 — ok
```

- **`0 — ok`가 나와야** 제대로 검증한 것입니다.
- `SSL_CTX_check_private_key`를 일부러 실패시켜 보십시오. 다른 키를 주면 **시작할 때** 잡힙니다. 실행 중에 알게 되는 것보다 훨씬 낫습니다.
- `SSL_get_version`·`SSL_get_cipher`로 실제 협상 결과를 찍으십시오.

---

### 문제 4. 가짜 서버 실험

**자기 CA로 같은 이름의 인증서를 만들어** 가짜 서버를 띄우고, 검증하는 클라이언트와 검증하지 않는 클라이언트로 각각 접속하십시오.

**정답 및 해설**

6.1~6.4절의 명령과 결과가 답입니다.

| 클라이언트 | 결과 |
|---|---|
| **검증 켬** | **오류 20으로 차단** |
| **검증 끔** | **연결됨 — 비밀번호가 넘어감** |

- **검증을 끈 판에서도 `검증 결과: 20`이 찍힌다**는 것을 반드시 확인하십시오. **알고도 진행한 것**입니다(6.5절).
- 서버 쪽 로그의 **`tlsv1 alert unknown ca`** 도 확인하십시오. 클라이언트가 이유를 알려 주고 끊은 것입니다.
- 가짜 인증서와 진짜 인증서를 나란히 놓고 비교하십시오.

```bash
openssl x509 -in server.crt -noout -subject -issuer
openssl x509 -in evil/fake.crt -noout -subject -issuer
```

- **`subject`는 같고 `issuer`만 다릅니다.** 이것이 전부입니다.
- 이 실험을 **직접 해 보는 것**이 이번 강의의 핵심입니다. "검증을 켜세요"라는 말보다 이 화면 하나가 오래 남습니다.

---

### 문제 5. 이름 검사

SAN에 없는 주소로 접속해 **오류 64**를 재현하고, `SSL_set1_host`를 빼면 어떻게 되는지 확인하십시오.

**정답 및 해설**

7.1절의 결과가 답입니다.

```text
  검증 결과: 64 — IP address mismatch
```

- **`SSL_set1_host`를 빼면 통과합니다.** 사슬은 완벽하기 때문입니다.
- 이것이 왜 위험한지 7.3절의 그림으로 설명할 수 있어야 합니다.
- `/etc/hosts`에 이름을 등록해 이름으로도 시험해 보십시오.

```bash
echo "127.0.0.1 c-srv" | sudo tee -a /etc/hosts
```

- 그러면 `c-srv`로는 통과하고, 다른 이름으로는 **오류 62**가 납니다(14강 7.2절).

---

### 문제 6. 오류 처리

`SSL_get_error`로 각 경우를 구분해 처리하십시오.

**정답 및 해설**

8.1절의 코드가 답입니다.

| 경우 | 재현 방법 |
|---|---|
| `SSL_ERROR_ZERO_RETURN` | 상대가 `SSL_shutdown` 후 종료 |
| `SSL_ERROR_SYSCALL` | 상대를 강제 종료(`kill -9`) |
| `SSL_ERROR_WANT_READ` | 논블로킹으로 설정 |

- **`SSL_shutdown` 없이 끊으면** `SSL_ERROR_SYSCALL`이 납니다. `ZERO_RETURN`과 구분되는 것을 보십시오.
- 논블로킹은 10강의 방식대로 `O_NONBLOCK`을 설정합니다.
- **정상 종료와 비정상 종료를 구분하는 것**이 중요합니다. 자료가 잘렸는데 정상으로 처리하면 잘못된 결과를 씁니다.

---

### 문제 7. 상호 TLS

서버도 클라이언트 인증서를 요구하게 만드십시오.

**정답 및 해설**

```bash
openssl genpkey -algorithm ED25519 -out client.key
openssl req -new -key client.key -subj "/C=KR/O=cmid-lab/CN=c-cli" -out client.csr
```

```bash
cat > client.ext <<'EOF'
basicConstraints = CA:FALSE
extendedKeyUsage = clientAuth
EOF
openssl x509 -req -in client.csr -CA ca/ca.crt -CAkey ca/ca.key \
  -CAcreateserial -days 365 -extfile client.ext -out client.crt
```

```c
/* 서버 쪽 */
SSL_CTX_load_verify_locations(ctx, "ca/ca.crt", NULL);
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, NULL);
```

```c
/* 클라이언트 쪽 */
SSL_CTX_use_certificate_chain_file(ctx, "client.crt");
SSL_CTX_use_PrivateKey_file(ctx, "client.key", SSL_FILETYPE_PEM);
```

- **`clientAuth`** 입니다. `serverAuth`가 아닙니다.
- **`SSL_VERIFY_FAIL_IF_NO_PEER_CERT`가 없으면** 인증서를 안 낸 클라이언트가 그냥 통과합니다. `SSL_VERIFY_PEER`만으로는 부족합니다.
- 클라이언트 인증서 없이 접속해 거절되는 것을 확인하십시오.

---

### 문제 8·9. 실제 사이트와 상호 검증

시스템 CA로 실제 사이트에 접속하고, 우리 서버를 표준 도구로 검증하십시오.

**정답 및 해설**

```c
SSL_CTX_set_default_verify_paths(ctx);
```

```bash
./tlscli www.kisa.or.kr 443 system
```

- 실제 사이트에 붙으려면 **HTTP 요청**을 보내야 응답이 옵니다.

```c
const char *req = "GET / HTTP/1.1\r\nHost: www.kisa.or.kr\r\nConnection: close\r\n\r\n";
SSL_write(ssl, req, (int) strlen(req));
```

- 4.4절처럼 **`openssl s_client`로 우리 서버에 붙는 것**도 해 보십시오. `Verification: OK`가 나와야 합니다.
- **양방향으로 통해야** 제대로 만든 것입니다. 한쪽만 되면 어딘가 어긋난 것입니다.

---

### 문제 10. 9강 코드와 비교

9강의 `tcpserv.c`와 오늘의 `tlsserv.c`를 비교해, **달라진 줄만** 뽑으십시오.

**정답 및 해설**

```bash
diff tcpserv.c tlsserv.c
```

| 그대로인 것 | 새로 생긴 것 |
|---|---|
| `socket`·`bind`·`listen`·`accept` | `SSL_CTX_new`·`SSL_new` |
| `getaddrinfo` | `SSL_accept` |
| `SO_REUSEADDR` | `SSL_read`/`SSL_write` |
| `EINTR` 재시도 | `SSL_shutdown`·`SSL_free` |
| 반복 구조 | |

- **소켓 부분은 하나도 안 바뀌었습니다.** TLS는 **연결된 뒤에 얹히는 계층**입니다.
- 12강 8.1절에서 그린 층 구조와 같습니다. 우리가 `securechan`을 얹은 자리에 TLS가 들어간 것입니다.
- **9강부터 15강까지가 하나의 이야기**였다는 것을 확인하십시오.

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 소스 — `tlsserv.c`, `tlscli.c` |
| 2 | 정상 접속 화면 — 서버·클라이언트 양쪽(문제 1) |
| 3 | **검증 세 줄**이 들어간 위치 표시(문제 3) |
| 4 | **가짜 서버 + 검증하는 클라이언트** — 차단 화면(문제 4) |
| 5 | **가짜 서버 + 검증 끈 클라이언트** — 비밀번호가 넘어간 화면(문제 4) |
| 6 | 진짜·가짜 인증서의 `subject`/`issuer` 비교(문제 4) |
| 7 | 이름 불일치 오류 화면(문제 5) |
| 8 | 상호 TLS 동작 화면(문제 7) |
| 9 | `openssl s_client`로 우리 서버 검증(문제 9) |
| 10 | 9강 코드와의 차이 정리(문제 10) |
| 11 | 짧은 서술 ① **왜 TLS를 쓰는 것만으로는 부족한가** |
| 12 | 짧은 서술 ② 검증 결과가 20인데 통신이 된 이유 |
| 13 | 짧은 서술 ③ 이름 검사를 빠뜨리면 어떤 공격이 가능한가 |

---

## 정리

| 구분 | 핵심 |
|---|---|
| `SSL_CTX` | **프로그램에 하나.** 인증서·키·신뢰 CA |
| `SSL` | **연결마다 하나** |
| 소켓 | **9강 그대로.** TLS는 연결된 뒤에 얹힌다 |
| 핸드셰이크 | `SSL_accept` / `SSL_connect` |
| 서버 인증서 | **`use_certificate_chain_file`** (중간 CA 포함) |
| SNI | `SSL_set_tlsext_host_name` — 어느 이름으로 왔는지 |
| **검증 ①** | `SSL_CTX_load_verify_locations` — 믿을 CA |
| **검증 ②** | **`SSL_CTX_set_verify(SSL_VERIFY_PEER)`** — **기본값은 꺼짐** |
| **검증 ③** | **`SSL_set1_host`** — **이름 검사** |
| **핵심 실증** | **검증을 끈 클라이언트가 가짜 서버에 비밀번호를 넘겼다** |
| 무서운 점 | **검증 결과 20을 알고도 진행했다** |
| 오류 처리 | `SSL_get_error` — 0이 오류가 아닐 수 있다 |
| 논블로킹 | `WANT_READ`/`WANT_WRITE`, **같은 버퍼로 재시도** |
| 종료 | `SSL_shutdown` — 잘린 것과 끝난 것을 구분하게 |
| **결론** | **TLS를 쓴다고 안전해지지 않는다. 검증을 켜야 안전해진다** |

---

## 다음 강의 예고

암호 통신 이야기가 끝났습니다. **16강 「커널의 시각」** 에서는 지금까지 만든 프로그램이 **운영체제 안에서 어떻게 보이는지** 봅니다.

- 가상 메모리 — 프로그램마다 자기 주소 공간
- `/proc`으로 **우리 프로그램을 들여다본다**
- 스케줄링 — 왜 순서가 보장되지 않는가
- 페이지 폴트와 메모리 매핑(6강 재방문)
- **1부부터 지금까지의 모든 개념이 커널에서 어떻게 만나는가**

그리고 **17강**에서 9~15강의 모든 것을 합쳐 **TLS 동시 접속 서버**를 만듭니다.

---

## 부록 A. 이번 강의 함수 요약

| 하려는 일 | 함수 |
|---|---|
| 설정 만들기 | `SSL_CTX_new(TLS_server_method())` / `TLS_client_method()` |
| 최소 판 지정 | `SSL_CTX_set_min_proto_version(ctx, TLS1_3_VERSION)` |
| **인증서 넣기** | **`SSL_CTX_use_certificate_chain_file`** |
| 개인키 넣기 | `SSL_CTX_use_PrivateKey_file(ctx, k, SSL_FILETYPE_PEM)` |
| 짝 확인 | `SSL_CTX_check_private_key` |
| **믿을 CA** | `SSL_CTX_load_verify_locations` |
| 시스템 CA | `SSL_CTX_set_default_verify_paths` |
| **검증 켜기** | **`SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, NULL)`** |
| 상호 TLS | `SSL_VERIFY_PEER \| SSL_VERIFY_FAIL_IF_NO_PEER_CERT` |
| 연결 객체 | `SSL_new(ctx)` / `SSL_free` |
| 서술자 붙이기 | `SSL_set_fd(ssl, fd)` |
| SNI | `SSL_set_tlsext_host_name(ssl, host)` |
| **이름 검사** | **`SSL_set1_host(ssl, host)`** |
| IP 검사 | `X509_VERIFY_PARAM_set1_ip_asc(SSL_get0_param(ssl), ip)` |
| 핸드셰이크 | `SSL_accept` / `SSL_connect` |
| 주고받기 | `SSL_read` / `SSL_write` |
| **오류 구분** | **`SSL_get_error(ssl, ret)`** |
| 검증 결과 | `SSL_get_verify_result` + `X509_verify_cert_error_string` |
| 상대 인증서 | `SSL_get1_peer_certificate` → **`X509_free`** |
| 협상 결과 | `SSL_get_version` · `SSL_get_cipher` |
| 종료 | `SSL_shutdown` |
| 오류 출력 | `ERR_print_errors_fp(stderr)` |

## 부록 B. 표준 문서와 출처

**OpenSSL 문서**

| 내용 | 주소 |
|---|---|
| `SSL_CTX_new` | [docs.openssl.org](https://docs.openssl.org/3.0/man3/SSL_CTX_new/) |
| **`SSL_CTX_set_verify`** | [SSL_CTX_set_verify](https://docs.openssl.org/3.0/man3/SSL_CTX_set_verify/) |
| **`SSL_set1_host`** | [SSL_set1_host](https://docs.openssl.org/3.0/man3/SSL_set1_host/) |
| `SSL_get_error` | [SSL_get_error](https://docs.openssl.org/3.0/man3/SSL_get_error/) |
| `SSL_read` | [SSL_read](https://docs.openssl.org/3.0/man3/SSL_read/) |
| `SSL_shutdown` | [SSL_shutdown](https://docs.openssl.org/3.0/man3/SSL_shutdown/) |
| 인증서 검증 개요 | [ossl-guide-tls-client-block](https://docs.openssl.org/3.0/man7/ossl-guide-tls-client-block/) |

**표준**

| 내용 | 문서 |
|---|---|
| **TLS 1.3** | [RFC 9846](https://www.rfc-editor.org/rfc/rfc9846) |
| 이름 검증 | [RFC 6125](https://www.rfc-editor.org/rfc/rfc6125) |
| SNI 등 확장 | [RFC 6066](https://www.rfc-editor.org/rfc/rfc6066) |

**관련 취약점 분류**

| CWE | 내용 |
|---|---|
| [CWE-295](https://cwe.mitre.org/data/definitions/295.html) | **잘못된 인증서 검증** — 6.4절 |
| [CWE-297](https://cwe.mitre.org/data/definitions/297.html) | **호스트 이름 불일치 무시** — 7.3절 |
| [CWE-599](https://cwe.mitre.org/data/definitions/599.html) | 인증서 검증 함수 오용 |

**본문의 주장과 근거**

| 주장 | 근거 |
|---|---|
| TLS 서버·클라이언트가 동작한다(4.2절) | `tlsserv`/`tlscli` — `검증 결과: 0 — ok`, 양방향 에코 |
| 표준 도구와 통한다(4.4절) | `openssl s_client -brief` → `Verification: OK` |
| 12·13강의 재료를 쓴다(4.3절) | `TLS_AES_256_GCM_SHA384` · `X25519` · `Ed25519` |
| **검증을 켜면 가짜 서버가 막힌다**(6.3절) | 오류 20, 서버 쪽 `tlsv1 alert unknown ca` |
| **검증을 끄면 가짜 서버에 비밀번호가 넘어간다**(6.4절) | 가짜 서버가 `받음: 비밀번호는 admin1234 입니다` 출력 |
| **검증 실패를 알고도 진행한다**(6.5절) | 같은 화면에 `검증 결과: 20` 이 찍혀 있다 |
| 이름이 다르면 거절된다(7.1절) | `검증 결과: 64 — IP address mismatch` |

> 포트 번호와 접속 포트는 실행할 때마다 다릅니다. **관계**(검증을 켜면 막히는가, 끄면 통하는가)를 보십시오. 검증 환경은 Ubuntu 24.04 / OpenSSL 3.0.13이며, **두 VM에서도 결과는 같습니다.**
{: .prompt-tip }

## 부록 C. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `tlsserv.c` | **TLS 에코 서버** — 17강에서 발전시킴 | 2 |
| `tlscli.c` | **TLS 클라이언트 — 검증 켬/끔 두 판** | 3 · 5 |
| `evil/ca.crt`·`evil/fake.crt` | **공격자의 CA와 가짜 인증서** | 6.1 |
| `client.crt`·`client.key` | 상호 TLS용 | 문제 7 |
