---
title: "[애플리케이션 보안] 03-3. 실습 — SPF 조회와 GPG 메일 암호화"
date: 2026-06-05 16:00:00 +0900
categories:
  - 강의
  - 애플리케이션보안
  - 메일보안
tags:
  - SPF
  - dig
  - GPG
  - PGP
  - 실습
math: false
mermaid: true
---

> 이 글은 **실습** 입니다. 개념은 **03-1. 메일 보안 이론**의 4장(SPF)·5장(PGP)을 먼저 읽으세요.  
> 이 실습은 **Kali(192.168.0.10)** 에서 진행합니다. (인터넷 연결 필요 — 어댑터1 NAT)
{: .prompt-info }

## A. SPF 레코드 직접 조회하기

발신자 위조를 막는 **SPF** 는 도메인의 DNS에 **TXT 레코드** 형태로 공개되어 있습니다. `dig` 로 직접 조회해 봅니다.

### A.1 도구 준비

```bash
# === Kali에서 실행 ===
dig -v 2>/dev/null || sudo apt install -y dnsutils
```

### A.2 SPF(TXT) 레코드 조회

```bash
# === Kali에서 실행 ===
dig TXT google.com +short
dig TXT github.com +short
```

출력 예시(도메인마다 다름):

```
"v=spf1 include:_spf.google.com ~all"
```

해석:

| 부분 | 의미 |
|---|---|
| `v=spf1` | SPF 버전 1 |
| `include:_spf.google.com` | 이 도메인 메일을 보낼 수 있는 **정당한 서버 목록**을 가리킴 |
| `~all` | 목록에 없는 곳에서 온 메일은 **의심(softfail)** 처리 |

> `~all`(softfail), `-all`(엄격히 거부, hardfail), `+all`(모두 허용 — 위험) 의 차이를 함께 보세요.
{: .prompt-tip }

### A.3 메일 서버(MX) 레코드도 조회

```bash
# === Kali에서 실행 ===
dig MX google.com +short
```

- **MX 레코드** 는 "이 도메인의 메일을 받는 서버가 누구인지"를 알려 줍니다. (DNS 04 카테고리에서 더 자세히)

---

## B. GPG로 메시지 암호화·서명하기

**GPG** 는 PGP의 오픈소스 구현으로, Kali에 기본 설치되어 있습니다. 공개키로 **암호화(기밀성)**, 개인키로 **서명(인증·무결성)** 을 직접 해 봅니다.

### B.1 내 키 한 쌍 만들기

```bash
# === Kali에서 실행 ===
gpg --full-generate-key
```

물어보는 항목에 다음과 같이 답합니다.

- 키 종류: 기본값 **(1) RSA and RSA** → 엔터
- 키 크기: **3072** → 엔터
- 유효기간: **0**(만료 없음) → 엔터 → `y`
- 이름: 본인 이름(예: `student`)
- 이메일: 아무 이메일(예: `student@example.com`)
- 암호문구(passphrase): **기억할 수 있는 값** 입력 (복호화·서명할 때 필요)

생성된 키를 확인합니다.

```bash
# === Kali에서 실행 ===
gpg --list-keys
```

### B.2 비밀 메시지 만들기

```bash
# === Kali에서 실행 ===
echo "이것은 아무도 보면 안 되는 비밀 메시지입니다." > secret.txt
```

### B.3 암호화 (공개키로 잠그기)

자기 자신에게 보내는 셈치고, **본인 이메일(공개키)** 로 암호화합니다.

```bash
# === Kali에서 실행 (이메일은 B.1에서 입력한 값) ===
gpg --encrypt --armor --recipient "student@example.com" --output secret.enc.asc secret.txt

# 암호문 확인 — 알아볼 수 없는 글자만 보인다
cat secret.enc.asc
```

- `--armor` : 암호문을 텍스트(메일에 붙일 수 있는 형태)로 만든다.
- `--recipient` : **받는 사람의 공개키** 를 지정 (여기서는 나 자신).

### B.4 복호화 (개인키로 열기)

```bash
# === Kali에서 실행 ===
gpg --decrypt secret.enc.asc
```

- 암호문구를 입력하면 원래 메시지가 보입니다. → **내 개인키가 있어야만** 열 수 있습니다(기밀성).

### B.5 전자서명과 검증

```bash
# === Kali에서 실행 ===
# 개인키로 서명
gpg --clearsign --output secret.sign.asc secret.txt

# 서명 내용 확인 (원문 + 서명 블록)
cat secret.sign.asc

# 공개키로 서명 검증
gpg --verify secret.sign.asc
```

- `Good signature` 가 나오면 **위조·변조되지 않았고, 서명자가 본인임** 이 확인된 것입니다(인증·무결성).
- 파일을 한 글자라도 고치면 검증이 실패합니다 — 직접 바꿔서 `--verify` 를 다시 해 보세요.

> GUI를 선호하면 Kali의 **Kleopatra** 로도 같은 작업(키 생성·암호화·서명)을 마우스로 할 수 있습니다.
{: .prompt-tip }

---

## C. 정리 — 메일 보안 한눈에

| 위협 | 방어 |
|---|---|
| 발신자 위조 | **SPF / DKIM / DMARC** (도메인이 DNS에 정당 발신 정보 공개) |
| 내용 도청 | **PGP/GPG 암호화**(공개키) + 전송 구간 SSL/TLS |
| 위·변조 | **전자서명**(개인키) |

## D. 체크리스트

- [ ] `dig TXT <도메인>` 로 SPF 레코드 조회·해석
- [ ] `dig MX <도메인>` 로 메일 서버 조회
- [ ] GPG 키 생성 → 암호화 → 복호화 성공
- [ ] 전자서명 → `--verify` 로 `Good signature` 확인
- [ ] 파일 변조 시 검증이 실패하는 것 확인

> 이로써 **03. 메일 보안** 을 마칩니다. 다음은 **04. DNS 보안** — 도메인 이름이 IP로 바뀌는 과정(이론)과 Zone Transfer 취약점 실습으로 이어집니다.
{: .prompt-info }
