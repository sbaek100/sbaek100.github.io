---
title: 리눅스 기초 5주차-2. 원격 접속과 서비스 관리
date: 2026-09-29 11:00:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - SSH
  - scp
  - systemd
  - systemctl
  - ufw
pin:
mermaid: false
---

> **학습 목표**
> 1. SSH의 개념을 이해하고, `ssh` 명령으로 다른 컴퓨터에 원격 접속할 수 있다.
> 2. Ubuntu Desktop에 SSH 서버(openssh-server)를 설치하여 원격 접속을 받을 수 있다.
> 3. `scp`로 컴퓨터 사이에 파일을 전송할 수 있다.
> 4. systemd와 `systemctl`의 개념을 이해하고 서비스를 조회·시작·정지·자동 시작 등록할 수 있다.
> 5. `ufw`로 방화벽을 구성하고, 서비스 포트의 개념과 개방 순서를 설명할 수 있다.
{: .prompt-info }

앞 강의에서 네트워크 상태를 확인하는 방법을 익혔다. 본 강의에서는 한 걸음 나아가, **다른 컴퓨터에 원격으로 접속**하고 **컴퓨터에서 동작하는 서비스를 관리**하는 방법을 학습한다.

본 강의의 실습은 **Ubuntu Desktop 24.04 LTS** 환경을 전제로 한다. 실습 계정은 `student`, 호스트 이름은 `ubuntu-y1`, 명령 프롬프트는 `student@ubuntu-y1:~$`이며, 실습 파일은 `~/lab05` 디렉터리에 둔다. 데스크톱 판은 **SSH 서버가 기본으로 설치되어 있지 않으므로**, 원격 접속을 받으려면 먼저 서버 프로그램을 설치하는 절차가 필요하다.


---
---

# 제1절. SSH 원격 접속

---

## 1.1 SSH란 무엇인가

**SSH(secure shell)** 는 네트워크를 통해 다른 컴퓨터에 접속하여 그 컴퓨터를 명령으로 조작할 수 있게 하는 프로토콜이다. 통신 구간 전체를 **암호화**하는 것이 가장 큰 특징이다. 과거에 사용되던 Telnet은 비밀번호를 포함한 모든 내용을 평문으로 주고받았으므로 현재는 사용하지 않는다.

원격 접속에는 두 컴퓨터가 필요하다.

| 역할 | 설명 | 필요한 프로그램 |
|---|---|---|
| **SSH 클라이언트** | 접속을 **거는** 쪽 | `ssh` (Ubuntu에 기본 설치) |
| **SSH 서버** | 접속을 **받는** 쪽 | `openssh-server` (Desktop 판은 별도 설치) |

Ubuntu Desktop에는 접속을 거는 `ssh` 명령은 기본으로 들어 있으나, **접속을 받는 서버 프로그램은 설치되어 있지 않다.** 따라서 다른 컴퓨터가 나에게 접속하도록 하려면 서버를 설치하여야 한다.

**[그림 5-4] SSH 클라이언트와 서버의 연결 구조 — 개념도**

```text
  ┌───────────────────────────┐              ┌───────────────────────────┐
  │   SSH 클라이언트           │              │   SSH 서버                 │
  │   (접속을 거는 쪽)         │              │   (접속을 받는 쪽)         │
  │                           │              │                           │
  │   프로그램 : ssh          │              │   프로그램 : sshd         │
  │   주소     : 192.168.0.30 │              │   주소     : 192.168.0.50 │
  └────────────┬──────────────┘              └──────────────┬────────────┘
               │                                            │
               │                                      22/tcp 에서 대기
               │                                            │
               │  ①  ssh student@192.168.0.50               │
               │ ─────────────────────────────────────────► │
               │                                            │
               │  ②  서버가 자신의 호스트 키 지문을 제시     │
               │ ◄───────────────────────────────────────── │
               │     (최초 1회만 yes 로 승인 →              │
               │      ~/.ssh/known_hosts 에 저장)            │
               │                                            │
               │  ③  비밀번호 또는 공개 키로 사용자 인증     │
               │ ─────────────────────────────────────────► │
               │                                            │
               │  ④  암호화된 통로로 원격 셸 세션 연결       │
               │ ◄════════════════════════════════════════► │
               └────────────────────────────────────────────┘
                     ══ 구간 전체가 암호화됨 ══
                     (Telnet은 이 구간이 평문이므로 사용하지 않음)
```

**각 단계의 의미**

| 단계 | 일어나는 일 | 관련 개념 | 실패 시 나타나는 메시지 |
|---|---|---|---|
| ① | 클라이언트가 서버의 22번 포트로 연결을 시도 | **포트 22**, 방화벽 | `Connection refused`(서버 미설치) / `Connection timed out`(방화벽 차단) |
| ② | **서버가 진짜인지** 확인. 지문이 바뀌면 위조 서버일 가능성을 경고 | 호스트 키, `~/.ssh/known_hosts` | `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` |
| ③ | **사용자가 진짜인지** 확인 | 비밀번호 인증, 키 인증 | `Permission denied (publickey,password)` |
| ④ | 셸 세션이 열려 원격 컴퓨터에 명령을 입력할 수 있게 됨 | 암호화 터널 | — |

② 와 ③ 은 **확인의 방향이 서로 반대**라는 점이 중요하다. ②는 클라이언트가 서버를 검증하는 단계이고, ③은 서버가 사용자를 검증하는 단계이다.

**최초 접속 시의 예상 화면**

```text
student@ubuntu-y1:~$ ssh student@192.168.0.50
The authenticity of host '192.168.0.50 (192.168.0.50)' can't be established.
ED25519 key fingerprint is SHA256:Af4NzqYb7PpIXWyfnQmmc1MvM+tuQqfi+yD/cbGibFQ.
This key is not known by any other names.                          ← ② 호스트 키 확인
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.0.50' (ED25519) to the list of known hosts.
student@192.168.0.50's password:                                   ← ③ 사용자 인증
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-xx-generic x86_64)
 ...
student@ubuntu-y2:~$                                               ← ④ 원격 셸 프롬프트
```

> 마지막 줄의 프롬프트가 `ubuntu-y1`에서 **`ubuntu-y2`로 바뀐 점**이 접속 성공의 가장 확실한 표시이다. 이 상태에서 입력하는 명령은 모두 원격 컴퓨터에서 실행된다. 지문 값과 호스트 이름은 환경마다 다르다.
{: .prompt-tip }

---

## 1.2 주요 명령

| 명령 | 기능 |
|---|---|
| `ssh 사용자@주소` | 원격 접속 |
| `ssh -p 2222 사용자@주소` | 포트를 지정하여 접속 |
| `exit` | 원격 접속 종료 |
| `ssh-keygen -t ed25519` | 인증용 키 쌍 생성 |
| `ssh-copy-id 사용자@주소` | 공개 키를 서버에 등록 |

SSH가 사용하는 기본 포트는 **22번**이다. 접속 대상 컴퓨터에서 이 포트가 열려 있고 SSH 서버가 대기 중이어야 접속이 이루어진다.

---

## 1.3 SSH 서버 설치

Ubuntu Desktop에서 원격 접속을 받으려면 `openssh-server` 패키지를 설치한다. 설치가 끝나면 곧바로 22번 포트가 대기 상태가 되어 다른 컴퓨터의 접속을 받을 수 있다.

```bash
sudo apt update
sudo apt install -y openssh-server
```

설치 후에는 22번 포트가 대기 상태인지 확인한다.

```bash
systemctl status ssh.socket
```

> `active (listening)`으로 표시되면 22번 포트에서 접속을 기다리는 상태이다. 서비스 관리 명령은 제3절에서 자세히 다룬다.

> **Ubuntu 24.04의 SSH는 "소켓 활성화" 방식으로 동작한다**
>
> Ubuntu 24.04부터 SSH 서버는 **소켓 활성화(socket activation)** 방식이 기본값이다. `sshd`가 처음부터 상시 떠 있는 것이 아니라, **systemd가 22번 포트를 대신 지키고 있다가 접속이 들어오는 순간에 `sshd`를 기동**한다. 요리사를 온종일 주방에 세워 두는 대신 손님이 들어오면 그때 부르는 식당에 비유할 수 있다.
>
> 그래서 **한 번도 접속이 없었던 상태**에서는 다음과 같이 보인다. 고장이 아니라 **정상**이다.
>
> | 확인 명령 | 24.04에서의 표시 | 의미 |
> |---|---|---|
> | `systemctl is-active ssh` | `inactive` | `sshd`가 아직 기동되지 않음 |
> | `systemctl is-enabled ssh` | `disabled` | `ssh.service`는 직접 등록되지 않음 |
> | **`systemctl is-active ssh.socket`** | **`active`** | **22번 포트를 지키는 주체** |
> | **`systemctl is-enabled ssh.socket`** | **`enabled`** | 부팅 시 자동으로 대기 |
>
> 따라서 **24.04에서는 `ssh`가 아니라 `ssh.socket`을 기준으로 확인**한다. 접속이 한 번 이루어지고 나면 `sshd`가 기동되므로 `systemctl is-active ssh`도 `active`로 바뀐다.
>
> 구형 자료나 타 배포판을 기준으로 한 설명에서는 설치 직후 `systemctl status ssh`가 `active (running)`이라고 서술한다. 24.04에서는 이 서술이 맞지 않는다는 점에 유의한다.
{: .prompt-warning }

---

> ### 따라 하기 1-1. SSH 서버 설치와 접속
>
> **목적** SSH 서버를 설치하고, 자기 자신에게 접속하는 방식으로 원격 접속의 동작 원리를 확인한다.
>
> 별도의 서버가 없어도 자기 자신(`localhost`)에게 접속하는 방식으로 실습할 수 있다. 원리는 원격 접속과 동일하다.
{: .prompt-tip }

**1단계.** 실습 디렉터리를 준비한다.

```bash
mkdir -p ~/lab05 && cd ~/lab05
```

**2단계.** SSH 서버를 설치한다.

```bash
sudo apt update && sudo apt install -y openssh-server
```

**3단계.** SSH 서버가 22번 포트에서 대기 중인지 확인한다.

```bash
sudo ss -tlnp | grep :22
```

> **예상 화면**(아직 한 번도 접속하지 않은 상태)
>
> ```text
> LISTEN 0 4096 0.0.0.0:22 0.0.0.0:* users:(("systemd",pid=1,fd=68))
> LISTEN 0 4096    [::]:22    [::]:* users:(("systemd",pid=1,fd=70))
> ```
>
> 22번 포트를 보유한 프로세스가 `sshd`가 아니라 **`systemd`(PID 1)** 로 표시된다. 앞에서 설명한 소켓 활성화 방식이기 때문이며 정상이다. 다음 단계에서 접속하고 나면 `sshd`도 함께 표시된다. `fd` 번호는 환경마다 다르다.

**4단계.** 자기 자신에게 접속한다.

```bash
ssh localhost
```

> 최초 접속 시 호스트 키 지문을 확인하라는 메시지가 표시된다. `yes`를 입력하고 비밀번호로 로그인한다.
>
> 이 지문은 서버의 신원 정보에 해당하며 `~/.ssh/known_hosts`에 저장된다. 이후 이 값이 바뀌면 위조 서버일 가능성을 경고한다.

**5단계.** 접속된 상태에서 현재 사용자를 확인한 뒤 접속을 종료한다.

```bash
whoami
```

```bash
exit
```

> `exit`를 입력하면 원격 세션이 종료되고 원래의 로컬 프롬프트로 돌아온다.

**6단계.** 접속을 한 번 수행한 뒤 다시 확인한다.

```bash
systemctl is-active ssh
```

```bash
sudo ss -tlnp | grep :22
```

> 3단계에서는 `inactive`였던 `ssh`가 이제 `active`로 바뀌고, 22번 포트에도 `sshd`가 함께 표시된다. **접속이 들어오는 순간 systemd가 `sshd`를 기동**하였기 때문이다. 소켓 활성화의 동작을 눈으로 확인하는 단계이다.

> **확인 사항** `openssh-server`를 설치하여 22번 포트가 열렸고, `ssh localhost`로 접속하였다가 `exit`로 되돌아왔다면 성공이다.
{: .prompt-tip }

---

> ### 따라 하기 1-2. 키 기반 인증
>
> **목적** 비밀번호 대신 키로 인증하도록 구성하고, 비밀번호 없이 접속되는 것을 확인한다.
{: .prompt-tip }

**1단계.** 키 쌍을 생성한다.

```bash
ssh-keygen -t ed25519 -C "linux-lab-$(whoami)" -f ~/.ssh/id_ed25519 -N ""
```

> `-t ed25519`는 현재 권장되는 암호 알고리즘이고, `-N ""`은 키 비밀번호를 설정하지 않는다는 의미이다. 실무에서는 반드시 키 비밀번호를 설정하여야 하나, 본 실습에서는 편의를 위해 생략한다.
>
> 같은 이름의 키가 이미 있으면 `Overwrite (y/n)?`이라고 묻는다. 기존 키를 다른 곳에 쓰고 있지 않다면 `y`를 입력하고, 보존해야 한다면 `n`을 입력한 뒤 다음 단계로 넘어간다.

**2단계.** 생성된 키를 확인한다.

```bash
ls -l ~/.ssh/
```

> `id_ed25519`(개인 키, 유출 금지)와 `id_ed25519.pub`(공개 키, 서버에 등록)이 생성되었다.

**3단계.** 공개 키를 서버(자기 자신)에 등록한다.

```bash
ssh-copy-id localhost
```

**4단계.** 등록된 키의 권한을 확인한다.

```bash
ls -ld ~/.ssh && ls -l ~/.ssh/authorized_keys
```

> `~/.ssh`가 `700`, `authorized_keys`가 `600`이어야 한다. 이 값이 느슨하면 SSH가 보안상 키를 무시하고 다시 비밀번호를 요구한다.

**5단계.** 비밀번호 없이 접속되는지 확인한다.

```bash
ssh localhost "hostname; whoami; date"
```

> 비밀번호를 요구하지 않으며, 접속과 동시에 명령을 실행하여 결과만 반환한다. 이 형태가 여러 서버를 자동으로 관리하는 스크립트의 기본 구조이다.

> **확인 사항** 키를 등록한 뒤 비밀번호 없이 접속되었다면 성공이다.
{: .prompt-tip }

---
---

# 제2절. 파일 전송

---

## 2.1 scp — 안전한 파일 복사

**scp(secure copy)** 는 SSH를 이용하여 컴퓨터 사이에 파일을 안전하게 복사하는 명령이다. SSH 위에서 동작하므로 전송 구간이 암호화되며, SSH 접속이 가능한 상대라면 그대로 사용할 수 있다.

| 명령 | 기능 |
|---|---|
| `scp 파일 사용자@주소:/경로` | 로컬 파일을 원격으로 전송 |
| `scp 사용자@주소:/경로/파일 .` | 원격 파일을 로컬로 가져오기 |
| `scp -r 디렉터리 사용자@주소:/경로` | 디렉터리 통째로 전송 |

전송 방향은 명령에서 **먼저 쓴 쪽이 원본, 나중에 쓴 쪽이 대상**이다.

---

> ### 따라 하기 2-1. scp로 파일 전송
>
> **목적** 파일을 만들어 `scp`로 전송하고, 전송이 완료되었는지 확인한다.
{: .prompt-tip }

**1단계.** 전송할 파일을 만든다.

```bash
echo "scp 전송 시험 파일" > ~/lab05/transfer.txt
```

**2단계.** 받을 디렉터리를 만든다.

```bash
mkdir -p ~/lab05/received
```

**3단계.** 파일을 전송한다.

```bash
scp ~/lab05/transfer.txt localhost:~/lab05/received/
```

**4단계.** 전송 결과를 확인한다.

```bash
ls -l ~/lab05/received/
```

> `transfer.txt`가 `received` 디렉터리에 복사되었음을 확인한다. 자기 자신에게 전송하였으나, 원격 컴퓨터에 전송하는 방식도 주소만 바뀔 뿐 동일하다.

> **확인 사항** `scp`로 파일을 전송하고, 대상 디렉터리에서 파일을 확인하였다면 성공이다.
{: .prompt-tip }

---
---

# 제3절. systemd 서비스 관리

---

## 3.1 systemd와 서비스

리눅스가 부팅될 때부터 백그라운드에서 계속 동작하며 특정 기능을 제공하는 프로그램을 **서비스(service)** 또는 **데몬(daemon)** 이라고 한다. 웹 서버, 데이터베이스, SSH 서버가 모두 서비스이다.

Ubuntu 24.04는 이러한 서비스들을 **systemd**라는 관리 체계로 통제한다. 관리자는 `systemctl` 명령으로 서비스를 조회하고 제어한다.

| 명령 | 기능 |
|---|---|
| `systemctl status 서비스` | 서비스 상태 조회 |
| `sudo systemctl start 서비스` | 서비스 시작 |
| `sudo systemctl stop 서비스` | 서비스 정지 |
| `sudo systemctl restart 서비스` | 서비스 재시작 |
| `sudo systemctl enable 서비스` | 부팅 시 자동 시작 등록 |
| `sudo systemctl disable 서비스` | 자동 시작 해제 |
| `systemctl is-active 서비스` | 현재 실행 여부만 간략히 표시 |
| `systemctl is-enabled 서비스` | 자동 시작 등록 여부만 간략히 표시 |

---

## 3.2 "실행 중"과 "자동 시작"의 구분

서비스에는 서로 다른 두 가지 상태가 있으며, 이를 혼동하지 않아야 한다.

| 상태 | 의미 | 확인 명령 |
|---|---|---|
| **active(실행 중)** | 지금 이 순간 동작하고 있는가 | `systemctl is-active` |
| **enabled(자동 시작)** | 다음 부팅 때 스스로 켜지는가 | `systemctl is-enabled` |

`start`로 시작한 서비스는 지금은 동작하지만, `enable`하지 않으면 재부팅 후에는 다시 꺼진다. 반대로 `enable`만 하고 `start`하지 않으면 지금은 꺼져 있으나 다음 부팅부터 켜진다. **두 상태는 별개이므로 각각 확인하여야 한다.**

---

## 3.3 서비스(`.service`)와 소켓(`.socket`)

`systemctl`이 다루는 대상을 **유닛(unit)** 이라 하며, 유닛에는 여러 종류가 있다. 본 강의에서 다루는 것은 두 가지이다.

| 유닛 종류 | 표기 | 역할 | 활성 상태 표시 |
|---|---|---|---|
| **서비스** | `ssh.service`(줄여서 `ssh`) | 실제로 프로그램을 실행함 | `active (running)` |
| **소켓** | `ssh.socket` | **포트만 대신 지키고 있다가** 접속이 오면 서비스를 기동함 | `active (listening)` |

제1.3절에서 설명한 것처럼 Ubuntu 24.04의 SSH는 `ssh.socket`이 22번 포트를 지키는 구조이다. 따라서 **"SSH가 접속을 받을 수 있는 상태인가"를 확인할 때는 `ssh.socket`을, "지금 `sshd`가 떠 있는가"를 확인할 때는 `ssh`를 본다.**

> **정지할 때 주의할 점**
>
> 소켓이 살아 있으면 서비스를 정지해도 다음 접속에서 다시 기동된다. 따라서 `sudo systemctl stop ssh` 한 줄로는 22번 포트가 닫히지 않는다. 완전히 닫으려면 소켓까지 함께 정지하여야 한다.
>
> ```bash
> sudo systemctl stop ssh.socket ssh.service
> ```
{: .prompt-warning }

---

> ### 따라 하기 3-1. 서비스 상태 조회와 제어
>
> **목적** SSH 서비스를 대상으로 상태를 조회하고, 정지·시작·자동 시작 등록을 실습한다.
{: .prompt-tip }

**1단계.** SSH 서비스의 상태를 조회한다.

```bash
systemctl status ssh
```

> **예상 화면**
>
> ```text
> ○ ssh.service - OpenBSD Secure Shell server
>      Loaded: loaded (/usr/lib/systemd/system/ssh.service; disabled; preset: enabled)
>      Active: inactive (dead)
> TriggeredBy: ● ssh.socket
>        Docs: man:sshd(8)
>              man:sshd_config(5)
> ```
>
> `TriggeredBy: ● ssh.socket` 행이 **"이 서비스는 `ssh.socket`이 기동시킨다"** 는 뜻이다. 접속 이력이 없으면 `inactive (dead)`로 표시되며 정상이다. `q`를 눌러 조회 화면을 빠져나온다.

**2단계.** 22번 포트를 지키는 소켓의 상태를 확인한다.

```bash
systemctl status ssh.socket
```

> `active (listening)`으로 표시되면 접속을 받을 수 있는 상태이다. `q`로 빠져나온다.

**3단계.** 실행 여부와 자동 시작 여부를 간략히 확인한다.

```bash
systemctl is-active ssh.socket
```

```bash
systemctl is-enabled ssh.socket
```

> 각각 `active`, `enabled`로 표시되는지 확인한다. 같은 명령을 `ssh`(서비스)에 대해 실행하면 접속 이력에 따라 `inactive`/`active`, 그리고 `disabled`로 나오는데, **Ubuntu 24.04에서는 이것이 정상**이다.

```bash
systemctl is-active ssh
```

```bash
systemctl is-enabled ssh
```

**4단계.** 자동 시작을 등록한다(이미 등록되어 있으면 그대로 진행된다).

```bash
sudo systemctl enable ssh.socket
```

> 재부팅 후에도 22번 포트가 스스로 대기 상태가 되도록 등록한다.

> **주의** SSH를 정지하면 원격 접속이 끊어진다. 자기 자신에게만 접속한 실습 환경에서는 안전하나, 실제 원격 서버에서는 SSH를 정지해서는 안 된다. 특히 소켓까지 정지(`sudo systemctl stop ssh.socket`)하면 다시 접속하여 되살릴 수단이 사라지므로, 물리 콘솔에 접근할 수 없는 서버에서는 절대 실행하지 않는다.

> **참고 — 전통적인 상시 구동 방식으로 바꾸려면**
> `sshd`를 부팅 시점부터 계속 띄워 두는 과거 방식이 필요하다면 소켓을 끄고 서비스를 등록한다. 본 실습에서는 기본값 그대로 두어도 충분하므로 실행하지 않아도 된다.
>
> ```bash
> sudo systemctl disable --now ssh.socket
> sudo systemctl enable --now ssh.service
> ```
{: .prompt-info }

> **확인 사항** `ssh.socket`이 `active`이고 `enabled`임을 확인하였고, `ssh`(서비스)가 `inactive`·`disabled`로 나오는 이유를 설명할 수 있다면 성공이다.
{: .prompt-tip }

---
---

# 제4절. 방화벽과 서비스 포트

---

## 4.1 서비스 포트의 개념

서비스는 저마다 정해진 **포트 번호**로 외부의 연결을 기다린다. 웹 서버는 80번(HTTP)과 443번(HTTPS), SSH 서버는 22번을 사용한다. 방화벽은 이 포트 단위로 연결을 허용하거나 차단한다.

중요한 점은 **서비스가 동작하는 것과 외부에서 접근 가능한 것은 별개**라는 사실이다. 서비스가 포트에서 대기하고 있어도, 방화벽이 그 포트를 막고 있으면 외부에서는 접근할 수 없다.

---

## 4.2 방화벽 ufw

**ufw(uncomplicated firewall)** 는 리눅스의 방화벽 기능을 간결한 명령으로 다룰 수 있게 하는 도구이다.

| 명령 | 기능 |
|---|---|
| `sudo ufw status verbose` | 상태 조회 |
| `sudo ufw allow 22/tcp` | 포트 개방 |
| `sudo ufw allow OpenSSH` | 애플리케이션 프로파일로 SSH 개방 |
| `sudo ufw default deny incoming` | 들어오는 연결 기본 차단 |
| `sudo ufw enable` / `disable` | 활성화 / 비활성화 |
| `sudo ufw delete allow 80/tcp` | 규칙 삭제 |

> **방화벽 설정 시 반드시 준수하여야 할 순서**
>
> 원격 접속 중에 방화벽을 켜면서 SSH를 허용하지 않으면, 활성화되는 순간 접속이 끊어진다. 반드시 **SSH를 먼저 허용한 뒤** 기본 정책을 정하고 활성화한다.
>
> ```
> sudo ufw allow OpenSSH          ← ① SSH를 먼저 허용
> sudo ufw default deny incoming  ← ② 기본 정책 설정
> sudo ufw enable                 ← ③ 활성화
> ```
>
> **SSH 허용 → 기본 정책 → 활성화**의 순서를 반드시 지킨다. 실무에서 매우 빈번하게 발생하는 사고이다.
{: .prompt-danger }

---

> ### 따라 하기 4-1. 방화벽 구성
>
> **목적** 올바른 순서로 방화벽을 활성화하고, 포트 개방 전후로 접근 여부가 달라지는 것을 확인한다.
{: .prompt-tip }

**1단계.** 현재 상태와 사용 가능한 프로파일을 확인한다.

```bash
sudo ufw status verbose
```

```bash
sudo ufw app list
```

**2단계.** ① SSH를 먼저 허용한다.

```bash
sudo ufw allow OpenSSH
```

> 이 단계를 생략하고 방화벽을 켜면 원격 접속이 끊어진다.

**3단계.** ② 기본 정책을 설정한다.

```bash
sudo ufw default deny incoming
```

```bash
sudo ufw default allow outgoing
```

**4단계.** ③ 방화벽을 활성화한다.

```bash
sudo ufw enable
```

> 데스크톱의 터미널에서 실행하면 곧바로 `Firewall is active and enabled on system startup`이 출력된다. 다만 **SSH로 접속한 상태에서 실행하면** `Command may disrupt existing ssh connections. Proceed with operation (y|n)?`이라고 묻는데, 2단계에서 SSH를 이미 허용하였으므로 `y`를 입력하면 된다.

```bash
sudo ufw status verbose
```

> SSH 접속이 유지되고 있음을 확인한다.

**5단계.** 웹 포트를 개방하고 규칙을 확인한다.

```bash
sudo ufw allow 80/tcp
```

```bash
sudo ufw status numbered
```

> 번호가 부여된 규칙 목록에 80번 포트 허용이 추가되었음을 확인한다.

**6단계.** 개방한 규칙을 삭제하여 정리한다.

```bash
sudo ufw delete allow 80/tcp
```

```bash
sudo ufw status
```

> **확인 사항** SSH 허용 → 기본 정책 → 활성화 순서를 지켰고, 포트 규칙을 추가·삭제하였다면 성공이다.
{: .prompt-tip }

---
---

# 제5절. 종합 실습 — 원격 접속 가능한 서버 만들기

---

> **실습 시나리오**
>
> 학습자는 자신의 Ubuntu Desktop을 **다른 컴퓨터가 SSH로 접속할 수 있는 상태**로 구성한다. SSH 서버를 설치하고, 서비스가 자동으로 시작되도록 등록하며, 방화벽에서 필요한 포트만 개방한다.
{: .prompt-info }

**1단계.** 실습 디렉터리로 이동한다.

```bash
cd ~/lab05
```

**2단계.** SSH 서버를 설치한다.

```bash
sudo apt update && sudo apt install -y openssh-server
```

**3단계.** 22번 포트가 대기 상태이고 자동 시작으로 등록되었는지 확인한다.

```bash
systemctl is-active ssh.socket
```

```bash
systemctl is-enabled ssh.socket
```

> 각각 `active`, `enabled`인지 확인한다. `enabled`가 아니면 `sudo systemctl enable --now ssh.socket`으로 등록한다. `ssh`(서비스) 쪽이 `inactive`·`disabled`로 나오는 것은 소켓 활성화 방식에서 정상이다.

**4단계.** SSH가 22번 포트에서 대기 중인지 확인한다.

```bash
sudo ss -tlnp | grep :22
```

> 아직 접속 이력이 없으면 보유 프로세스가 `systemd`로, 접속한 뒤에는 `sshd`도 함께 표시된다.

**5단계.** 방화벽을 순서에 맞게 구성한다.

```bash
sudo ufw allow OpenSSH
```

```bash
sudo ufw default deny incoming
```

```bash
sudo ufw enable
```

```bash
sudo ufw status verbose
```

**6단계.** 자기 자신에게 접속하여 최종 확인한다.

```bash
ssh localhost "echo 원격 접속 성공; hostname"
```

> 다른 컴퓨터에서 접속할 때는 `localhost` 대신 이 컴퓨터의 IP 주소(앞 강의에서 `ip addr`로 확인한 값)와 계정을 지정하여 `ssh student@주소` 형태로 접속한다.

> **종합 실습 완료 기준**
> 1. `openssh-server`를 설치하였다.
> 2. `ssh.socket`이 `active`이고 `enabled`임을 확인하였다.
> 3. 22번 포트가 대기 중임을 확인하였다.
> 4. SSH 허용 → 기본 정책 → 활성화 순서로 방화벽을 구성하였다.
> 5. `ssh localhost`로 접속하여 명령이 실행됨을 확인하였다.
{: .prompt-tip }

---
---

# 제6절. 자주 발생하는 오류와 대응 방법

---

| 화면에 출력된 메시지 / 증상 | 원인 | 대응 방법 |
|---|---|---|
| `ssh: connect to host ... Connection refused` | 대상에 SSH 서버가 없음 | 대상 컴퓨터에서 `sudo apt install -y openssh-server`로 설치한다. |
| `Connection timed out` | **방화벽에 차단됨** | `sudo ufw allow OpenSSH`로 22번 포트를 허용한다. |
| SSH 키 등록 후에도 비밀번호를 요구 | `~/.ssh` 권한이 느슨함 | `chmod 700 ~/.ssh`, `chmod 600 ~/.ssh/authorized_keys` |
| `Permission denied (publickey)` | 공개 키 미등록 또는 권한 문제 | `ssh-copy-id`로 키를 다시 등록하고 권한을 확인한다. |
| `scp: ... No such file or directory` | 대상 경로가 존재하지 않음 | 대상 디렉터리를 먼저 만든 뒤 전송한다. |
| `systemctl status ssh`가 `inactive (dead)` | **오류가 아님.** 24.04의 소켓 활성화 방식 | `systemctl status ssh.socket`이 `active (listening)`인지 확인한다. |
| `systemctl is-enabled ssh`가 `disabled` | **오류가 아님.** 등록 주체가 `ssh.socket`임 | `systemctl is-enabled ssh.socket`으로 확인한다. |
| `ss -tlnp`에 `sshd`가 아닌 `systemd` 표시 | **오류가 아님.** systemd가 포트를 대신 지킴 | 한 번 접속하면 `sshd`도 함께 표시된다. |
| `systemctl` 정지 후 원격 접속 끊김 | SSH를 `stop`함 | 콘솔에서 `sudo systemctl start ssh.socket`으로 다시 대기시킨다. |
| 서비스를 `stop`했는데 계속 접속됨 | `ssh.socket`이 살아 있어 다시 기동됨 | `sudo systemctl stop ssh.socket ssh.service`로 소켓까지 정지한다. |
| `ufw enable` 후 접속 단절 | SSH를 허용하지 않고 활성화 | 콘솔에서 `sudo ufw allow OpenSSH`를 실행한다. |
| 재부팅 후 서비스가 꺼짐 | 자동 시작 미등록 | `sudo systemctl enable 서비스`로 등록한다. |

---
---

# 제7절. 요약

---

## 7.1 핵심 개념 정리

| 구분 | 요점 |
|---|---|
| SSH | 통신 구간을 암호화하는 원격 접속 프로토콜이며 기본 포트는 22번이다. |
| SSH 서버 | Desktop 판은 기본 미설치이므로 `openssh-server`를 설치해야 접속을 받는다. |
| 소켓 활성화 | Ubuntu 24.04는 `ssh.socket`이 22번 포트를 지키다가 접속이 오면 `sshd`를 기동한다. 따라서 대기 여부는 `ssh`가 아니라 **`ssh.socket`으로 확인**한다. |
| 키 인증 | 비밀번호 대신 키 쌍으로 인증한다. `~/.ssh` 권한이 느슨하면 키가 무시된다. |
| 파일 전송 | `scp`는 SSH 위에서 파일을 암호화하여 복사한다. |
| systemd | `systemctl`로 유닛을 제어한다. "실행 중(active)"과 "자동 시작(enabled)"은 별개이며, `.service`와 `.socket`도 별개의 유닛이다. |
| 방화벽 | `ufw`로 포트를 관리하며, **SSH 허용 → 기본 정책 → 활성화** 순서를 지킨다. |
| 포트 개념 | 서비스가 동작하는 것과 외부에서 접근 가능한 것은 별개의 문제이다. |

---

## 7.2 본 강의에서 학습한 명령어

| 명령어 | 기능 |
|---|---|
| `ssh` | 원격 접속 |
| `ssh-keygen` / `ssh-copy-id` | 키 생성 / 등록 |
| `scp` | 파일 전송 |
| `systemctl status` / `start` / `stop` | 유닛 상태 조회 / 시작 / 정지 |
| `systemctl enable` / `disable` | 자동 시작 등록 / 해제 |
| `systemctl is-active` / `is-enabled` | 실행 여부 / 자동 시작 여부 확인 |
| `systemctl status ssh.socket` | 22번 포트 대기 여부 확인(Ubuntu 24.04) |
| `ufw allow` / `enable` / `status` | 방화벽 규칙 / 활성화 / 조회 |
| `ss -tlnp` | 개방된 포트와 프로세스 조회 |
