---
title: "[애플리케이션 보안] 02-2. FTP 공격 유형과 보안기술"
date: 2026-10-18 10:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - FTP보안
tags:
  - FTP
  - BounceAttack
  - AnonymousFTP
  - TFTP
  - SFTP
  - FTPS
  - Wireshark
  - 정보보안산업기사
math: false
mermaid: false
---

> **02-1** 에서 만든 FTP 서버(`ftpuser` / `ftppass123`)가 동작 중이어야 합니다. 
{: .prompt-info }

## 0. 실습 환경

| 장비 | IP | 역할 |
|---|---|---|
| Kali | **192.168.0.10** | FTP·SFTP 클라이언트, 와이어샤크 |
| Ubuntu | **192.168.0.30** | vsftpd(21번), OpenSSH(22번) |

> 명령과 예상 화면은 **Ubuntu 24.04.4 LTS** 와 **Kali Rolling** 에서 실행해 확인했습니다.
>
> 패킷 수집은 **본인이 구성한 실습망에서만** 합니다. 허가 없이 다른 사람의 통신을 수집하는 것은 불법입니다.
{: .prompt-danger }

---

# 3. FTP 공격 유형

이 장은 **이론**입니다. 각 공격이 **무엇을 악용하는지**와 **대책**을 구분할 수 있으면 됩니다.

## 3.1 공격 유형 한눈에 보기

| 공격                               | 악용하는 약점                                | 결과                                             | 대책                                     |
| -------------------------------- | -------------------------------------- | ---------------------------------------------- | -------------------------------------- |
| **Anonymous FTP 공격**             | 인증 없이 접속을 허용하는 익명 FTP                  | 익명 사용자에게 **쓰기 권한**이 있으면 **악성코드 업로드·유포**        | 익명 FTP 사용 금지, 쓰기 권한 제거, 디렉터리 소유자·권한 관리 |
| **FTP Bounce 공격**                | 서버가 `PORT` 명령의 **목적지를 검사하지 않는** 설계상 문제 | FTP 서버를 거쳐 **내부망 포트 스캔**, 공격자 출발지 은폐, 위조 메일 전송 | `PORT` 목적지 검증(현대 서버의 기본값), 익명 FTP 금지   |
| **TFTP 공격** (File-Name Transfer) | TFTP에 **인증이 없음**                       | 파일 이름만 알면 누구나 **설정 파일 등을 가져감**                 | TFTP 미사용, 꼭 쓰면 **secure 모드**·접근 제한     |
| **무차별 대입(Brute Force)**          | 비밀번호를 반복 대입                            | 약한 비밀번호 계정 탈취                                  | 강한 비밀번호, 로그인 실패 감시, 접속 IP 제한           |
| **스니핑(평문 노출)**                   | FTP가 **암호화하지 않음**                      | 아이디·비밀번호·파일 내용 노출                              | **SFTP·FTPS** 사용 (4장 실습)               |

> **함정 주의**: **ARP 스푸핑**은 FTP 서비스에서 발생하는 공격이 아닙니다. 같은 네트워크에서 MAC 주소를 속이는 **스니핑 기법**입니다. "FTP 공격이 아닌 것"을 고르는 문제에 자주 나옵니다.
{: .prompt-warning }

## 3.2 Anonymous FTP 공격

| 항목 | 내용 |
|---|---|
| 개요 | 보안 절차를 거치지 않은 **익명 사용자**에게 FTP 서버 접근을 허용 |
| 위험 | 익명 사용자가 서버에 **쓰기 권한**이 있으면 **악성코드 생성·업로드**가 가능 |
| 대책 | ① 반드시 쓸 경우가 아니면 서비스 제거 ② 익명 사용자의 루트·`bin`·`etc`·`pub` 디렉터리의 **소유자와 퍼미션 관리** ③ 익명용 `passwd` 에서 불필요한 항목 제거 ④ 익명 사용자에게 **쓰기 권한을 주지 않음** |

> 02-1의 2.2절에서 `anonymous_enable=NO` 로 익명 접속이 막혀 있는 것을 확인했습니다.
{: .prompt-info }

## 3.3 FTP Bounce 공격

Active 모드에서 클라이언트는 `PORT` 명령을 통해 데이터 연결의 목적지 주소와 포트 번호를 서버에 통지한다(02-1 참조). 프로토콜의 설계 의도상 이 목적지는 명령을 보낸 **클라이언트 자신의 주소**여야 한다.

#### 정의

FTP Bounce 공격은 공격자가 `PORT` 명령에 자신의 주소 대신 **제3의 호스트 주소**를 기재하여, FTP 서버로 하여금 해당 호스트에 **대리 접속**하도록 유도하는 공격이다. 공격 트래픽이 FTP 서버를 경유하여 반사(bounce)되는 형태로 목표에 도달한다는 점에서 이러한 명칭이 붙었다. 이 공격이 성립하는 근본 원인은 초기 FTP 명세(RFC 959)가 `PORT` 명령에 기재된 목적지의 정당성을 검증하는 절차를 규정하지 않았다는 데 있다.

#### PORT 명령의 형식

`PORT` 명령의 인자는 쉼표로 구분된 6개의 십진수로 구성된다. 앞의 4개는 IPv4 주소를, 뒤의 2개는 포트 번호의 상위 바이트(p1)와 하위 바이트(p2)를 나타내며, 포트 번호는 p1 × 256 + p2로 산출한다.

| 명령 | IP 주소 | 포트 번호 |
|---|---|---|
| `PORT 10,0,0,5,0,25` | `10.0.0.5` | 0 × 256 + 25 = **25** (SMTP) |
| `PORT 10,0,0,5,0,80` | `10.0.0.5` | 0 × 256 + 80 = **80** (HTTP) |

#### 공격 절차

![FTP Bounce 공격 흐름: 공격자가 PORT 명령으로 내부 PC 주소를 적으면 FTP 서버가 대신 접속한다](/assets/img/posts/2026-10-18-appsec-02-2-ftp-bounce.svg){: width="530" height="297" }

| 단계 | 내용 |
|---|---|
| 전제 | 방화벽은 외부에서 내부 호스트(`10.0.0.5`)로 향하는 직접 접속을 **차단**하고, FTP 서버의 21번 포트로 향하는 접속만 허용한다 |
| ① | 공격자는 FTP 서버에 로그인한 후, `PORT 10,0,0,5,0,25` 와 같이 **내부 호스트의 주소와 포트**를 기재한 명령을 전송한다 |
| ② | 목적지를 검증하지 않는 FTP 서버는 데이터 연결을 위해 **내부 호스트의 25번 포트로 접속**을 시도한다. FTP 서버는 방화벽 내부에 위치하므로 이 접속은 차단되지 않는다 |
| ③ | 공격자는 FTP 서버가 반환하는 **응답 코드**를 통해 결과를 판별한다. 연결이 수립되면(150) 해당 포트는 **열림**, 수립되지 않으면(425) **닫힘**으로 판정한다 |

공격자는 포트 번호를 변경하며 ①~③을 반복함으로써, 외부에서는 관측할 수 없는 **내부 호스트의 개방 포트 목록**을 획득한다. 이를 **FTP 서버를 경유한 내부망 포트 스캔**이라 한다. 또한 내부 호스트의 접속 기록에는 공격자가 아닌 **FTP 서버의 주소**가 남으므로, 공격자의 출발지가 은닉되는 효과도 발생한다.

#### 요약

| 항목    | 내용                                                                                      |
| ----- | --------------------------------------------------------------------------------------- |
| 원리    | 서버가 `PORT` 에 기재된 목적지를 **검증하지 않으면**, 클라이언트가 지정한 **제3의 호스트**로 그대로 접속·전송한다                 |
| 결과    | 공격자가 FTP 서버를 **경유**하여 외부에서 직접 접근할 수 없는 **내부망 호스트의 포트를 스캔**하거나, 출발지가 은닉된 위조 메일을 전송할 수 있다 |
| 핵심 용어 | **`PORT` 명령 악용**, **목적지 미검증**, **내부망 포트 스캔**                                            |
| 대책    | 서버가 `PORT` 주소와 실제 클라이언트 주소의 일치 여부를 검증(vsftpd 기본 동작), 익명 FTP 금지                          |

> 시험에서는 "**PORT 명령**을 악용하여 외부에서 직접 접근 불가능한 **내부망** 컴퓨터의 포트에 FTP 서버를 통해 접근"이 정답 문장으로 출제된다.
{: .prompt-tip }

## 3.4 TFTP를 이용한 공격

| 항목  | 내용                                                                                                                                              |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| 원리  | TFTP는 **인증 절차를 요구하지 않음** → 설정이 잘못되어 있으면 누구나 접속해 파일을 가져갈 수 있음                                                                                    |
| 대책  | ① 쓰지 않으면 서비스 제거(시스템 설정 파일에서 주석 처리) ② 꼭 쓰면 **secure 모드(`-s`)** 로 운영해 지정한 디렉터리만 접근 ③ **TCP Wrapper** 로 허용한 IP에서만 접근 ④ **chroot** 로 접근 가능한 디렉터리 제한 |

> secure 모드는 TFTP 서버를 특정 디렉터리 안에서만 동작하게 하는 옵션입니다. Ubuntu의 TFTP 서버(`tftpd-hpa`)는 설정 파일 `/etc/default/tftpd-hpa` 에 `TFTP_OPTIONS="--secure"` 가 기본으로 들어 있습니다.
{: .prompt-info }

## 3.5 무차별 대입 공격과 로그 확인

비밀번호를 하나씩 바꿔 가며 계속 로그인을 시도하는 공격입니다. 서버는 **로그인 실패 기록**을 남기므로, 짧은 시간에 실패가 반복되면 공격을 의심할 수 있습니다.

### [실습] 로그인 실패가 기록되는지 확인

Kali에서 **틀린 비밀번호**로 한 번 로그인해 봅니다.

```bash
# === Kali ===
ftp 192.168.0.30
```

```
Name (192.168.0.30:kali): ftpuser
331 Please specify the password.
Password:                       <- type a wrong password
530 Login incorrect.
ftp: Login failed
ftp> bye
221 Goodbye.
```

Ubuntu에서 vsftpd 로그를 확인합니다.

```bash
# === Ubuntu ===
sudo grep -E "OK LOGIN|FAIL LOGIN" /var/log/vsftpd.log | tail -n 5
```

```
Sun Oct 18 16:58:39 2026 [pid 1191] [ftpuser] OK LOGIN: Client "::ffff:192.168.0.10"
Sun Oct 18 17:00:47 2026 [pid 1366] [ftpuser] FAIL LOGIN: Client "::ffff:192.168.0.10"
```

📷 **캡처**: `FAIL LOGIN` 이 기록된 화면

| 항목 | 의미 |
|---|---|
| `[ftpuser]` | 로그인을 시도한 계정 |
| `FAIL LOGIN` / `OK LOGIN` | 로그인 실패 / 성공 |
| `Client "::ffff:192.168.0.10"` | 접속한 곳(Kali). `::ffff:` 는 IPv4 주소를 IPv6 형식으로 표시한 것 |

> `No such file or directory` 가 나오면 vsftpd 로그 기록이 꺼져 있는 것입니다. `grep -n '^xferlog_enable' /etc/vsftpd.conf` 결과가 비어 있다면 02-1 [실습 1]의 `xferlog_enable=YES` 를 추가하고 vsftpd를 재시작한 뒤, Kali에서 다시 틀린 비밀번호로 로그인합니다.
{: .prompt-warning }

#### 보조 확인: 시스템 인증 로그

vsftpd는 로그인 인증을 리눅스의 **PAM(인증 모듈)** 에 맡깁니다. 그래서 vsftpd 로그와 별개로, 인증 실패가 시스템 로그(journal)에도 기록됩니다.

```bash
# === Ubuntu ===
sudo journalctl -u vsftpd --since today | grep "authentication failure"
```

```
Oct 18 23:26:35 ubuntu-y1 vsftpd[4066]: pam_unix(vsftpd:auth): authentication failure; logname= uid=0 euid=0 tty=ftp ruser=ftpuser rhost=192.168.0.10  user=ftpuser
```

| 항목 | 의미 |
|---|---|
| `pam_unix(vsftpd:auth)` | vsftpd의 로그인 요청을 PAM이 인증함 |
| `authentication failure` | 인증 실패 |
| `rhost=192.168.0.10` | 접속한 곳(Kali) |
| `user=ftpuser` | 로그인을 시도한 계정 |

### [실습] 반복되는 실패를 탐지하기

실패 한 건만으로는 공격인지 알 수 없습니다. **짧은 시간에 같은 IP에서 실패가 여러 번** 반복되는 것이 무차별 대입 공격의 특징입니다. 실패를 일부러 여러 번 만든 뒤, 그 패턴이 로그에 어떻게 남는지 확인합니다.

Kali에서 **틀린 비밀번호**로 다섯 번 접속을 시도합니다. `curl` 은 비밀번호를 한 줄로 넣을 수 있어 반복에 편리합니다.

```bash
# === Kali ===
for i in $(seq 1 5); do
  curl -s -u ftpuser:wrongpass ftp://192.168.0.30/ -o /dev/null
done
echo "done"
```

```
done
```

> 이 반복은 **본인이 만든 실습 서버의 실습 계정**에만 사용합니다. 허가 없이 다른 서버에 비밀번호를 반복 시도하는 것은 불법입니다.
{: .prompt-danger }

Ubuntu에서 실패 횟수를 세어 봅니다.

```bash
# === Ubuntu ===
sudo grep -c "FAIL LOGIN" /var/log/vsftpd.log
sudo grep "FAIL LOGIN" /var/log/vsftpd.log | tail -n 5
```

```
5
Sun Oct 18 17:10:01 2026 [pid 1401] [ftpuser] FAIL LOGIN: Client "::ffff:192.168.0.10"
Sun Oct 18 17:10:01 2026 [pid 1402] [ftpuser] FAIL LOGIN: Client "::ffff:192.168.0.10"
Sun Oct 18 17:10:02 2026 [pid 1403] [ftpuser] FAIL LOGIN: Client "::ffff:192.168.0.10"
Sun Oct 18 17:10:02 2026 [pid 1404] [ftpuser] FAIL LOGIN: Client "::ffff:192.168.0.10"
Sun Oct 18 17:10:02 2026 [pid 1405] [ftpuser] FAIL LOGIN: Client "::ffff:192.168.0.10"
```

📷 **캡처**: `FAIL LOGIN` 이 같은 IP·같은 시각에 반복된 화면

| 관찰 포인트 | 의미 |
|---|---|
| `grep -c` 결과가 **5** | 실패가 다섯 번 기록됨 |
| **같은 `Client`** 가 반복 | 한 곳(Kali)에서 집중적으로 시도함 |
| **시각(초)이 거의 같음** | 사람이 손으로 친 속도가 아니라 **자동 반복** → 공격 신호 |

> 수동으로 로그를 세는 대신, 실무에서는 **fail2ban** 같은 도구가 이 패턴(짧은 시간에 같은 IP의 실패 N회)을 자동으로 찾아 해당 IP를 차단합니다. 이 글의 범위에서는 "반복 실패가 로그에 남고, 이것으로 공격을 탐지한다"까지만 확인합니다.
{: .prompt-info }

> 이런 실패 기록이 같은 IP에서 수십 번 반복되면 무차별 대입 공격입니다. 대책은 **강한 비밀번호**, **허용 IP 제한**(02-1의 ufw 규칙), 그리고 근본적으로 **암호화된 SFTP 사용**입니다.
{: .prompt-tip }

---

# 4. FTP 보안기술

## 4.1 왜 암호화가 필요한가 — 평문 전송

### [이론]

FTP는 아이디, 비밀번호, 파일 내용을 **암호화하지 않고** 그대로 보냅니다. 02-1에서 익명 차단, ftpusers, chroot를 적용했지만, 이 설정들은 **누가 들어오는가**를 통제할 뿐 **오가는 내용이 보이는가**는 막지 못합니다.

### [실습] (Kali) 와이어샤크로 FTP 비밀번호 확인

**① Kali의 네트워크 인터페이스 이름 확인**

```bash
# === Kali ===
ip -br addr | grep 192.168.0.10
```

```
eth0             UP             192.168.0.10/24 fe80::.../64
```

맨 앞의 이름(예: `eth0`)을 기억합니다.

**② 와이어샤크 실행과 캡처 시작**

```bash
# === Kali ===
sudo wireshark &
```

1. 첫 화면의 인터페이스 목록에서 ①에서 확인한 이름(`eth0`)을 **더블클릭**하면 캡처가 시작됩니다.
2. 위쪽 **표시 필터** 칸에 `ftp` 를 입력하고 Enter를 누릅니다.

**③ 새 터미널에서 FTP 로그인**

```bash
# === Kali (new terminal) ===
ftp 192.168.0.30
```

`ftpuser` / `ftppass123` 으로 로그인한 뒤 `bye` 로 나옵니다.

**④ 와이어샤크에서 확인**

패킷 목록에 다음과 같은 줄이 나타납니다.

```
192.168.0.30  192.168.0.10  FTP  Response: 220 (vsFTPd 3.0.5)
192.168.0.10  192.168.0.30  FTP  Request: USER ftpuser
192.168.0.30  192.168.0.10  FTP  Response: 331 Please specify the password.
192.168.0.10  192.168.0.30  FTP  Request: PASS ftppass123
192.168.0.30  192.168.0.10  FTP  Response: 230 Login successful.
```

![](/assets/img/posts/2026-10-18-appsec-02-2-ftp-attack-security-1792366403902.png)

비밀번호만 골라 보려면 표시 필터를 다음으로 바꿉니다.

```
ftp.request.command == "PASS"
```

**⑤ 대화 전체 보기 — Follow TCP Stream**

`PASS` 패킷을 **우클릭 → Follow → TCP Stream** 을 선택하면 로그인 대화가 한 화면에 보입니다.

```
220 (vsFTPd 3.0.5)
USER ftpuser
331 Please specify the password.
PASS ftppass123
230 Login successful.
```


> **결론**: 같은 네트워크에서 패킷을 볼 수 있는 사람은 FTP 비밀번호를 **그대로 읽을 수 있습니다.** 이것이 평문 FTP를 쓰지 말아야 하는 이유입니다.
{: .prompt-warning }

---

## 4.2 SFTP와 FTPS

### [이론]

평문 문제를 해결하는 암호화 파일 전송 방식은 두 가지입니다. 이름이 비슷하지만 **완전히 다른 기술**입니다.

| 구분 | **SFTP** (Secure FTP) | **FTPS** (FTP over SSL/TLS) |
|---|---|---|
| 기반 기술 | **SSH** | **SSL/TLS** |
| 포트 | **22** | 21(Explicit) / 990(Implicit) |
| 관계 | SSH로 파일을 주고받는 **별개의 프로토콜** | 기존 FTP에 **암호화를 덧씌운 것** |
| 준비물 | SSH 서버만 있으면 됨 | 서버 인증서 필요 |

> **시험 빈출**: "secure FTP 프로그램에서 사용하는 보안 프로토콜은?" → **SSH**. SFTP는 FTP의 취약점을 보완하기 위한 **SSH 기반** 파일 전송 프로토콜입니다.
{: .prompt-tip }

### [실습] (Kali) SFTP로 접속해 비교하기

SFTP는 Ubuntu에 설치된 **OpenSSH 서버**가 담당하므로 서버 쪽 추가 설정이 필요 없습니다.

**① 와이어샤크 표시 필터 변경**

와이어샤크 표시 필터를 SSH 포트로 바꿉니다.

```
tcp.port == 22
```

**② 새 터미널에서 SFTP 접속**

```bash
# === Kali (new terminal) ===
sftp ftpuser@192.168.0.30
```

```
The authenticity of host '192.168.0.30 (192.168.0.30)' can't be established.
ED25519 key fingerprint is SHA256:....
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
ftpuser@192.168.0.30's password:
Connected to 192.168.0.30.
sftp> ls
sftp> bye
```

처음 접속할 때 서버 확인 질문에 `yes` 를 입력하고, 비밀번호 `ftppass123` 을 넣습니다. `ls` 로 목록을 본 뒤 `bye` 로 나옵니다.

**③ 와이어샤크에서 확인**

```
192.168.0.10  192.168.0.30  SSH    Client: Protocol (SSH-2.0-OpenSSH_10.3p1 Debian-4)
192.168.0.30  192.168.0.10  SSHv2  Server: Protocol (SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.20)
192.168.0.10  192.168.0.30  SSHv2  Client: Key Exchange Init
192.168.0.30  192.168.0.10  SSHv2  Server: Key Exchange Init
192.168.0.10  192.168.0.30  SSHv2  Client: Diffie-Hellman Key Exchange Init
192.168.0.30  192.168.0.10  SSHv2  Server: Diffie-Hellman Key Exchange Reply, New Keys
192.168.0.10  192.168.0.30  SSHv2  Client: Encrypted packet (len=...)
...
```


- 처음에는 SSH 버전 정보와 **키 교환(Key Exchange)** 과정만 보입니다.
- 그 뒤는 모두 **Encrypted packet(암호화된 패킷)** 으로 표시되어, 아이디·비밀번호·파일 내용을 읽을 수 없습니다.
- 아무 패킷이나 Follow → TCP Stream 해도 알아볼 수 없는 문자만 나옵니다.

### 결과 비교

| 구분 | 평문 FTP | SFTP |
|---|---|---|
| 포트 | 21 (+ 데이터 포트) | **22** |
| 와이어샤크 표시 | `Request: PASS ftppass123` | `Encrypted packet` |
| 비밀번호 노출 | **보임** | 안 보임 |
| 권장 여부 | ❌ 사용 지양 | ✅ 권장 |

---

## 4.3 FTP 보안 대책 종합

02-1과 02-2에서 다룬 대책을 시험 대비용으로 정리합니다.

| 구분 | 대책 |
|---|---|
| **전송 암호화** | 평문 FTP 대신 **SFTP(SSH)** 또는 **FTPS(SSL/TLS)** 사용 |
| **계정 통제** | `/etc/ftpusers` 에 root 등 시스템 계정 등록, 파일 소유자 root·권한 640 이하 |
| **익명 FTP** | 사용 금지, 쓰기 권한 금지, 루트·`bin`·`etc`·`pub` 소유자·권한 관리, 익명용 `passwd` 불필요 항목 제거 |
| **접근 범위** | chroot로 홈 디렉터리 밖 접근 차단 |
| **운영** | 필요할 때만 서비스 활성화, 중요 파일 저장 금지, 연결 시간(timeout) 제한, 최신 버전·주기적 패치 |
| **Bounce 공격** | `PORT` 목적지 검증, 익명 FTP 금지 |
| **TFTP** | 사용하지 않으면 제거, 쓰면 secure 모드(`-s`)·TCP Wrapper·chroot |
| **무차별 대입** | 강한 비밀번호, 로그인 실패 로그 감시, 접속 IP 제한 |

---

# 확인 문제

> 1. 다음 중 FTP 서비스에서 발생할 수 있는 공격 유형이 **아닌** 것은?
>    ① brute force 공격
>    ② bounce 공격
>    ③ anonymous ftp 공격
>    ④ ARP 스푸핑 공격
>    
> 2. FTP Bounce 공격에 대한 설명으로 옳은 것은?
>    ① 분산 반사 공격(DRDoS)으로 악용된다
>    ② PORT 명령을 악용하여 외부에서 직접 접근 불가능한 내부망 컴퓨터의 포트에 FTP 서버를 통해 접근한다
>    ③ 로그인 응답 속도 차이로 계정 존재 여부를 추측한다
>    ④ active, passive 모드를 임의로 변경한다
>    
> 3. 익명 사용자가 서버에 쓰기 권한이 있을 때 악성코드 생성이 가능한 공격은?
>    
> 4. secure FTP(SFTP)에서 사용하는 보안 프로토콜은? 
>    ① TLS
>    ② SSL
>    ③ SSH
>    ④ RSA
>    
> 5. TFTP를 꼭 사용해야 할 때의 보안 조치로 적당하지 **않은** 것은?
>    ① -s 옵션(secure 모드)으로 데몬을 띄운다
>    ② TCP Wrapper로 허용된 IP에서만 접근하게 한다
>    ③ chroot로 접근 가능한 디렉터리를 제한한다
>    ④ 유추하기 쉬운 패스워드를 사용하지 못하게 제한한다
{: .prompt-tip }

<details>
<summary>정답 보기</summary>

1. <strong>④</strong> ARP 스푸핑은 FTP 공격이 아니라 스니핑 기법이다.<br>
2. <strong>②</strong><br>
3. <strong>Anonymous FTP 공격</strong><br>
4. <strong>③ SSH</strong><br>
5. <strong>④</strong> TFTP에는 패스워드(인증) 자체가 없으므로 패스워드 제한은 대책이 될 수 없다.

</details>

---

# 체크리스트

- [ ] 5가지 공격 유형의 악용 약점과 대책을 표로 설명할 수 있음
- [ ] ARP 스푸핑이 FTP 공격이 아닌 이유를 설명할 수 있음
- [ ] (Ubuntu) `/var/log/vsftpd.log` 에서 `FAIL LOGIN` 확인
- [ ] (Kali) 와이어샤크에서 `Request: PASS ftppass123` 확인
- [ ] (Kali) SFTP 접속 시 `Encrypted packet` 만 보이는 것 확인
- [ ] SFTP(SSH, 22)와 FTPS(SSL/TLS, 21·990)의 차이를 설명할 수 있음

> 이로써 **02. FTP 보안**을 마칩니다. 다음은 **03. 메일 보안**입니다.
{: .prompt-info }
