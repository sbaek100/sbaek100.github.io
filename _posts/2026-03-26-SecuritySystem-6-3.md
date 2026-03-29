---
title: (6주차) 보안시스템구축실습 6-3 - Fail2Ban으로 SSH 공격 자동 방어
date: 2026-03-26 08:50:00 +0900
categories:
  - 강의
  - 보안시스템구축실습
tags:
  - SSH
  - Fail2Ban
  - 로그분석
  - 자동방어
  - BruteForce
mermaid: true
pin: false
description: Ubuntu 서버에서 Fail2Ban을 설치하고, SSH 포트 2222에 대한 반복 로그인 실패를 자동 탐지하여 차단하는 실습.
---

## 실습 환경

| 구분 | 운영체제 | IP 주소 | 역할 |
|------|----------|---------|------|
| 공격자 PC | Kali Linux | 192.168.0.10 | SSH 접속 시도 |
| 서버 | Ubuntu | 192.168.0.30 | Fail2Ban 동작 확인 |

> **6-1, 6-2를 완료한 뒤 진행합니다.**
> 이 문서는 `Port 2222`, `PubkeyAuthentication yes`, `PasswordAuthentication no` 상태를 전제로 합니다.

---

## Fail2Ban이 왜 필요한가?

6-2에서 확인했듯이, SSH 서버는 공개된 순간부터 다양한 접속 시도와 계정 추측 공격을 받는다. `PasswordAuthentication no` 설정만으로도 공격 성공 가능성은 크게 줄지만, 공격 시도 자체가 사라지지는 않는다.

Fail2Ban은 로그 파일을 자동으로 읽다가,

- 일정 횟수 이상 실패한 IP를 발견하면
- 방화벽 규칙을 추가해
- 일정 시간 동안 그 IP를 차단한다.

즉, **로그 분석을 사람이 수동으로 하는 대신, 기본 대응을 자동화하는 도구**라고 보면 된다.

```mermaid
flowchart LR
    A["공격자<br/>반복 로그인 실패"] --> B["auth.log에 실패 기록 누적"]
    B --> C["Fail2Ban이 로그 감시"]
    C --> D["기준 횟수 초과 감지"]
    D --> E["방화벽에 차단 규칙 추가"]
    E --> F["해당 IP 일정 시간 접속 차단"]
```

---

## Part 1. Fail2Ban 설치

### 1-1. 패키지 설치

```bash
sudo apt update
sudo apt install fail2ban -y
```

예상 화면:

```text
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  fail2ban
Setting up fail2ban (...)
```

### 1-2. 서비스 시작 및 자동 시작 등록

```bash
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
sudo systemctl status fail2ban
```

예상 화면:

```text
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/lib/systemd/system/fail2ban.service; enabled)
     Active: active (running)
```

---

## Part 2. SSH용 기본 설정 확인

Fail2Ban은 기본 설정만으로도 `sshd` jail을 사용할 수 있지만, 실습에서는 SSH 포트가 2222이므로 이를 명확히 반영하는 것이 좋다.

### 2-1. 로컬 설정 파일 만들기

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

### 2-2. sshd 항목 수정

`[sshd]` 항목 근처를 찾아 다음과 같이 맞춘다.

```ini
[sshd]
enabled = true
port    = 2222
logpath = %(sshd_log)s
maxretry = 3
findtime = 10m
bantime = 1h
```

설명:

- `enabled = true`: sshd 감시 활성화
- `port = 2222`: SSH 포트 2222 감시
- `maxretry = 3`: 3회 실패 시 차단
- `findtime = 10m`: 10분 안에 실패 횟수를 셈
- `bantime = 1h`: 1시간 차단

### 2-3. 설정 적용

```bash
sudo fail2ban-client reload
sudo fail2ban-client status sshd
```

예상 화면:

```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed: 0
|  `- File list: /var/log/auth.log
`- Actions
   |- Currently banned: 0
   |- Total banned: 0
   `- Banned IP list:
```

---

## Part 3. 동작 확인

### 3-1. 로그 감시 상태 확인

```bash
sudo tail -f /var/log/auth.log
```

다른 터미널에서 Fail2Ban 상태를 본다.

```bash
sudo fail2ban-client status sshd
```

### 3-2. 차단된 IP 확인

```bash
sudo fail2ban-client status sshd
```

예상 화면:

```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 3
|  |- Total failed: 5
`- Actions
   |- Currently banned: 1
   |- Total banned: 1
   `- Banned IP list: 192.168.0.10
```

### 3-3. 방화벽 반영 확인

환경에 따라 `iptables` 또는 `nftables` 기반으로 반영될 수 있다. 간단히 규칙 존재만 확인한다.

```bash
sudo iptables -L -n
```

또는:

```bash
sudo nft list ruleset
```

---

## Part 4. 운영 관점에서 이해할 점

Fail2Ban은 매우 유용하지만 만능은 아니다.

- 공개키 인증을 대체하지 못한다.
- 포트 변경을 대체하지 못한다.
- 취약한 계정 정책을 해결해주지 못한다.
- 로그가 남지 않는 공격은 직접 탐지하지 못한다.

즉, 다음 순서로 이해하면 좋다.

1. SSH 포트를 2222로 변경
2. root 로그인 차단
3. 공개키 인증만 허용
4. 로그 분석
5. Fail2Ban으로 반복 공격 자동 차단

---

## 셀프체크

**Q1.** Fail2Ban의 핵심 역할은 무엇인가?

- SSH 포트를 자동으로 변경한다
- 로그를 감시해 반복 실패 IP를 자동 차단한다
- 공개키를 자동 생성한다
- 암호화를 대신 수행한다

---

**Q2.** 실습에서 SSH 포트가 2222일 때 `jail.local`의 `port` 값은 무엇이어야 하는가?

- 20
- 21
- 22
- 2222

---

**Q3.** 다음 중 Fail2Ban이 대체할 수 없는 것은 무엇인가?

- 공개키 인증 설정
- 반복 실패 IP 차단
- auth.log 감시
- 기본 대응 자동화

---

## 한 줄 정리

**Fail2Ban은 SSH 보안의 출발점이 아니라, 이미 잘 설정된 SSH 서버에 자동 대응 능력을 더해주는 보조 방어 장치다.**
