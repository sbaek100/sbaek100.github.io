---
title: "[애플리케이션 보안] 07-3. 실습 — SSH 로그인에 OTP 2단계 인증 붙이기"
date: 2026-06-05 20:12:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - 전자상거래보안
tags:
  - OTP
  - 2FA
  - SSH
  - PAM
  - 실습
math: false
mermaid: true
---

> 이 글은 **실습(🟡 약간 난이도 있음)** 입니다. 앞 글(**07-2**)에서 OTP 원리를 익힌 뒤 진행하세요.  
> 목표: **Ubuntu(192.168.0.30)** 의 SSH 로그인을 **비밀번호 + 일회용 코드(OTP)** 2단계로 바꾸고, **Kali(192.168.0.10)** 에서 접속해 본다.
{: .prompt-info }

> ⚠️ **설정 실수로 SSH가 막힐 수 있습니다.** 작업 내내 **VirtualBox 콘솔 창(Ubuntu 화면)을 열어 두세요.** SSH가 안 되면 콘솔로 들어가 설정을 되돌리면 됩니다(7장 복구 방법 참고).
{: .prompt-warning }

## 0. 목표 그림

```mermaid
flowchart LR
    K["Kali"] -- "ssh dvwa@192.168.0.30" --> U["Ubuntu"]
    U --> P1["① 비밀번호 확인"]
    P1 --> P2["② OTP 6자리 확인"]
    P2 --> OK["로그인 성공"]
```

---

## 1. (Ubuntu) Google Authenticator PAM 모듈 설치

```bash
# === Ubuntu에서 실행 ===
sudo apt update
sudo apt install -y libpam-google-authenticator
```

---

## 2. (Ubuntu) dvwa 계정의 OTP 키 만들기

OTP를 적용할 **본인 계정(dvwa)** 으로 설정 마법사를 실행합니다. (`sudo` 없이!)

```bash
# === Ubuntu에서, dvwa 계정으로 실행 ===
google-authenticator
```

질문에 다음과 같이 답합니다.

- `Do you want authentication tokens to be time-based (y/n)` → **y** (시간 기반 TOTP)
- 화면에 **QR 코드 + 비밀키(secret) + 비상 코드(emergency scratch codes)** 가 표시됩니다.
  - 스마트폰 **Google Authenticator** 로 QR을 스캔하거나, 비밀키를 수동 입력한다.
  - **비상 코드 5개는 따로 적어 둔다.** (휴대폰을 못 쓸 때 로그인용)
- `Update your "~/.google_authenticator" file (y/n)` → **y**
- `Disallow multiple uses of the same token (y/n)` → **y** (재사용 방지)
- `increase the original generation time window (y/n)` → **n**
- `enable rate-limiting (y/n)` → **y** (무차별 대입 완화)

> 휴대폰 앱이 없으면, 표시된 **비밀키** 를 07-2의 `oathtool --totp -b "<비밀키>"` 에 넣어 코드를 만들어도 됩니다.
{: .prompt-tip }

---

## 3. (Ubuntu) SSH가 OTP를 묻도록 설정

### 3.1 PAM 설정에 OTP 추가

```bash
# === Ubuntu에서 실행 ===
sudo nano /etc/pam.d/sshd
```

`@include common-auth` 줄을 찾아 **그 아래에** 다음 한 줄을 추가합니다. (비밀번호 다음에 OTP를 묻게 됨)

```text
auth required pam_google_authenticator.so
```

### 3.2 sshd 설정 변경

```bash
# === Ubuntu에서 실행 ===
sudo nano /etc/ssh/sshd_config
```

다음 항목들을 찾아 아래 값으로 맞춥니다. (없으면 추가)

```text
KbdInteractiveAuthentication yes
PasswordAuthentication no
UsePAM yes
AuthenticationMethods keyboard-interactive
```

> 구버전에서는 `KbdInteractiveAuthentication` 대신 `ChallengeResponseAuthentication` 일 수 있습니다. 같은 의미이니 있는 쪽을 `yes` 로 둡니다.
{: .prompt-tip }

### 3.3 SSH 재시작

```bash
# === Ubuntu에서 실행 ===
sudo systemctl restart ssh
```

> ⚠️ **지금 열려 있는 SSH·콘솔 세션은 닫지 마세요.** 새 접속이 정상 동작하는 것을 확인한 뒤 닫습니다.
{: .prompt-warning }

---

## 4. (Kali) 2단계 인증으로 접속

```bash
# === Kali에서 실행 ===
ssh dvwa@192.168.0.30
```

이제 **두 단계** 를 차례로 묻습니다.

```
Password:               ← dvwa 비밀번호
Verification code:      ← 스마트폰 앱(또는 oathtool)의 6자리 OTP
```

둘 다 맞아야 로그인됩니다.

> 🟢 **의미**: 비밀번호가 유출돼도, **그 순간의 OTP가 없으면 로그인 불가** 입니다. 이것이 2단계 인증(2FA)이 계정 탈취를 크게 줄이는 이유입니다.
{: .prompt-tip }

---

## 5. 체크리스트

- [ ] `libpam-google-authenticator` 설치, dvwa로 `google-authenticator` 키 생성
- [ ] 비상 코드 5개 별도 보관
- [ ] `/etc/pam.d/sshd` 에 `pam_google_authenticator.so` 추가
- [ ] `sshd_config` 의 `KbdInteractiveAuthentication yes` + `AuthenticationMethods keyboard-interactive`
- [ ] Kali에서 **비밀번호 + OTP** 2단계로 로그인 성공

---

## 6. (중요) 원래대로 되돌리기 / 잠겼을 때 복구

OTP를 끄고 싶거나 잠겼을 때는 **VirtualBox 콘솔(Ubuntu 화면)** 에서:

```bash
# === Ubuntu 콘솔에서 실행 ===
# 1) PAM에서 OTP 줄 제거 (또는 맨 앞에 # 추가해 주석 처리)
sudo nano /etc/pam.d/sshd        # auth required pam_google_authenticator.so 줄 삭제/주석

# 2) sshd 설정 원복
sudo nano /etc/ssh/sshd_config   # PasswordAuthentication yes 로, AuthenticationMethods 줄 제거

# 3) 재시작
sudo systemctl restart ssh
```

> 콘솔 로그인은 OTP 설정과 무관하게 항상 가능하므로, **콘솔만 있으면 언제든 복구** 할 수 있습니다.
{: .prompt-tip }

---

## 7. 다음 카테고리

이로써 **07. IPSEC·SET·OTP** 를 마칩니다. 다음은 **08. 개발보안 ① — 방법론과 시큐어코딩** — 안전한 소프트웨어를 만드는 절차(이론)와 취약 코드를 고치고 정적분석으로 점검하는 실습으로 이어집니다.
