---
title: "[애플리케이션 보안] 10-2. 실습 — 서버 하드닝 종합 점검"
date: 2026-06-05 20:28:00 +0900
categories:
  - 강의
  - 애플리케이션보안
  - 종합정리
tags:
  - 하드닝
  - 방화벽
  - ufw
  - 보안점검
  - 실습
math: false
mermaid: true
---

> 이 과정의 **마지막 실습** 입니다. 지금까지 배운 방어를 **Ubuntu(192.168.0.30)** 한 서버에 모아 **하드닝(보안 강화) 점검** 으로 마무리합니다.
{: .prompt-info }

> ⚠️ 방화벽을 켜기 전 **SSH(22) 허용을 먼저** 합니다. 작업 중 **VirtualBox 콘솔 창** 을 열어 두면 안전합니다.
{: .prompt-warning }

## 1. 시스템 최신화

알려진 취약점의 상당수는 **패치만 해도** 막힙니다.

```bash
# === Ubuntu에서 실행 ===
sudo apt update && sudo apt upgrade -y
```

---

## 2. 열린 포트·서비스 점검 (공격 표면 줄이기)

쓰지 않는 서비스는 **공격 표면** 입니다. 무엇이 열려 있는지 먼저 봅니다.

```bash
# === Ubuntu에서 실행 ===
# 현재 열려서 대기 중인 포트
sudo ss -tlnp

# 실행 중인 서비스 목록
systemctl list-units --type=service --state=running
```

- 우리 실습 서버에는 FTP(21)·DNS(53)·HTTP/HTTPS(80/443)·SSH(22)·DB(3306)·메일(25) 등이 보일 수 있습니다.
- **쓰지 않는 서비스는 끕니다.** 예: 메일 실습이 끝났다면

  ```bash
  sudo systemctl disable --now postfix
  ```

---

## 3. 방화벽(ufw) — 필요한 포트만 열기

```bash
# === Ubuntu에서 실행 ===
sudo apt install -y ufw

# 기본 정책: 들어오는 것은 막고, 나가는 것은 허용
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 꼭 필요한 포트만 허용 (SSH를 먼저!)
sudo ufw allow 22/tcp       # SSH
sudo ufw allow 80/tcp       # HTTP
sudo ufw allow 443/tcp      # HTTPS
sudo ufw allow 53           # DNS

# 방화벽 켜기
sudo ufw enable
sudo ufw status verbose
```

> 🟢 이제 허용한 포트 외의 접근은 **모두 차단** 됩니다. 최소한의 통로만 여는 것이 하드닝의 기본입니다.
{: .prompt-tip }

---

## 4. 계정·권한 점검

```bash
# === Ubuntu에서 실행 ===
# 관리자(sudo) 권한을 가진 사용자 확인
getent group sudo

# 로그인 가능한 계정 목록(셸이 있는 계정)
grep -vE "nologin|false" /etc/passwd
```

- 쓰지 않는 계정은 잠그거나 삭제합니다: `sudo usermod -L 계정명` (잠금)
- **최소 권한 원칙**(05장): 꼭 필요한 사람에게만 sudo·DB 권한을 줍니다.

---

## 5. 서비스별 보안 설정 복습 점검

이 과정에서 적용한 방어들이 살아 있는지 확인합니다.

| 항목 | 점검 명령 / 확인 | 기대 상태 |
|---|---|---|
| **SSH 루트 로그인 차단** | `sudo sshd -T \| grep permitrootlogin` | `no` 권장 |
| **SSH 2단계 인증**(07) | `/etc/pam.d/sshd` 에 google_authenticator | 적용됨 |
| **FTP 평문 지양**(02) | 가능하면 **SFTP** 사용, `/etc/ftpusers` 에 root | 설정됨 |
| **DNS Zone Transfer 제한**(04) | `named.conf.local` 의 `allow-transfer` | `none`/지정IP |
| **DB 최소권한·로컬바인드**(05) | `appuser` 권한, DB는 내부에서만 | 제한됨 |
| **TLS 버전**(06) | `testssl.sh 192.168.0.30:443` | TLS1.2+ |

> SSH 루트 차단이 안 돼 있으면 `/etc/ssh/sshd_config` 에서 `PermitRootLogin no` 로 바꾸고 `sudo systemctl restart ssh`.
{: .prompt-tip }

---

## 6. 로그 점검 (사고 대비)

```bash
# === Ubuntu에서 실행 ===
# 인증 관련 로그 (로그인 시도·실패)
sudo tail -n 20 /var/log/auth.log
```

- 반복되는 로그인 실패는 **무차별 대입 시도** 의 흔적일 수 있습니다.
- 자동 차단 도구 **fail2ban** 을 더하면 좋습니다: `sudo apt install -y fail2ban`

---

## 7. 최종 하드닝 체크리스트

- [ ] 시스템 최신 패치(`apt upgrade`)
- [ ] 불필요 서비스 중지, 열린 포트 최소화(`ss -tlnp`)
- [ ] **방화벽(ufw)** 으로 필요한 포트만 허용(SSH 먼저)
- [ ] 불필요 계정 잠금, **최소 권한** 적용
- [ ] SSH 루트 차단 + **2단계 인증**
- [ ] 서비스별 보안설정(FTP·DNS·DB·TLS) 유지 확인
- [ ] 인증 로그 점검(+fail2ban)

---

## 8. 과정을 마치며

> 하나의 가상 실습실에서 **만들고(서버 구축) → 공격하고(스니핑·AXFR·명령삽입) → 막는(암호화·최소권한·2FA·방화벽)** 전 과정을 직접 경험했습니다.  
> 보안은 한 번의 설정이 아니라 **계속 점검하고 강화하는 습관** 입니다. 수고하셨습니다. 🎉
{: .prompt-tip }

이로써 **애플리케이션 보안** 전체 과정을 마칩니다.
