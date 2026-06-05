---
title: "[애플리케이션 보안] 07-2. 실습 — OTP(일회용 비밀번호) 직접 생성하기"
date: 2026-06-05 20:10:00 +0900
categories:
  - 애플리케이션보안
  - 전자상거래보안
tags:
  - OTP
  - TOTP
  - oathtool
  - 2단계인증
  - 실습
math: false
mermaid: true
---

> 이 글은 **실습** 입니다. 개념은 앞 글(**07-1**)의 4장(OTP)을 먼저 읽으세요.  
> 이 실습은 **Kali(192.168.0.10)** 에서 진행합니다. 스마트폰의 **Google Authenticator**(또는 비슷한 OTP 앱)가 있으면 비교 실습이 가능합니다.
{: .prompt-info }

## 0. 목표

`oathtool` 로 **시간 기반 OTP(TOTP)** 를 직접 만들고, 30초마다 바뀌는 것과 **스마트폰 앱과 같은 코드** 가 나오는 것을 확인합니다.

---

## 1. (Kali) oathtool 설치

```bash
# === Kali에서 실행 ===
sudo apt update
sudo apt install -y oathtool
```

---

## 2. (Kali) 비밀키(시드)로 TOTP 만들기

OTP는 **비밀키(시드)** 와 **현재 시각** 으로 코드를 계산합니다(07-1 4.2). 우선 실습용 고정 비밀키를 씁니다. (Base32 형식)

```bash
# === Kali에서 실행 ===
# 비밀키: JBSWY3DPEHPK3PXP (Base32)
oathtool --totp -b "JBSWY3DPEHPK3PXP"
```

→ 6자리 숫자가 출력됩니다(예: `284172`).

### 2.1 30초마다 바뀌는지 확인

```bash
# === Kali에서 실행 ===
# 2초마다 현재 TOTP를 출력 (Ctrl+C로 종료)
watch -n 2 'oathtool --totp -b "JBSWY3DPEHPK3PXP"'
```

- **30초 경계** 를 지날 때 숫자가 한 번 바뀌는 것을 관찰합니다. → 매번 달라지므로 가로채도 곧 무효가 됩니다.

---

## 3. (선택) 스마트폰 앱과 같은 코드인지 비교

같은 비밀키를 스마트폰 OTP 앱에 넣으면 **똑같은 6자리** 가 나옵니다. (서버·앱이 같은 키+같은 시각을 쓰기 때문)

1. 스마트폰에서 **Google Authenticator** 실행 → **+ → 설정 키 입력(수동 입력)**
2. 계정명: 아무거나(예: `lab`), 키: **`JBSWY3DPEHPK3PXP`**, 유형: **시간 기반**
3. 앱에 표시되는 6자리와, Kali의 `oathtool` 출력이 **같은 시간대(30초)에 동일** 한지 확인

> ⚠️ 숫자가 다르면 대개 **시계가 안 맞아서** 입니다. Kali 시간을 동기화하세요: `sudo systemctl restart systemd-timesyncd` (또는 `sudo ntpdate pool.ntp.org`). TOTP는 **시각이 맞아야** 동작합니다.
{: .prompt-warning }

---

## 4. (참고) 내 비밀키 만들기 / HOTP

### 4.1 무작위 비밀키 생성

```bash
# === Kali에서 실행 ===
head -c 10 /dev/urandom | base32
```

출력된 문자열을 `oathtool --totp -b "<출력값>"` 에 넣으면 나만의 TOTP가 됩니다.

### 4.2 이벤트 기반(HOTP)도 비교

시간이 아니라 **카운터(횟수)** 로 코드를 만드는 방식입니다(07-1 4.1).

```bash
# === Kali에서 실행 ===
oathtool -b --hotp -c 1 "JBSWY3DPEHPK3PXP"    # 카운터 1
oathtool -b --hotp -c 2 "JBSWY3DPEHPK3PXP"    # 카운터 2 (다른 코드)
```

- TOTP는 시간이 지나면, HOTP는 카운터를 올리면 코드가 바뀝니다.

---

## 5. 체크리스트

- [ ] `oathtool --totp` 로 6자리 TOTP 생성
- [ ] 30초마다 코드가 바뀌는 것 관찰
- [ ] (선택) 스마트폰 앱에 같은 키 입력 → 동일 코드 확인
- [ ] HOTP는 카운터에 따라 코드가 바뀌는 것 확인
- [ ] TOTP가 "비밀키 + 현재 시각"으로 만들어진다는 점 설명 가능

---

## 6. 다음 글

OTP를 만들어 봤으니, 다음 글 **07-3. SSH 로그인에 OTP 2단계 인증 붙이기** 에서 실제 로그인에 OTP를 적용해, **비밀번호 + 일회용 코드** 가 둘 다 있어야 접속되는 2단계 인증(2FA)을 만들어 봅니다.
