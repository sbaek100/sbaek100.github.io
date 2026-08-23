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

> **[그림 5-4] 필요**
> SSH 클라이언트(접속을 거는 컴퓨터)와 SSH 서버(접속을 받는 컴퓨터)가 22번 포트를 통해 암호화된 통신으로 연결되는 구조를 표현한 개념도.
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

Ubuntu Desktop에서 원격 접속을 받으려면 `openssh-server` 패키지를 설치한다. 설치가 끝나면 SSH 서비스가 자동으로 시작되어 22번 포트에서 대기한다.

```bash
sudo apt update
sudo apt install -y openssh-server
```

설치 후에는 서비스가 정상 동작하는지 확인한다.

```bash
systemctl status ssh
```

> `active (running)`으로 표시되면 SSH 서버가 동작 중이다. 서비스 관리 명령은 제3절에서 자세히 다룬다.

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

> 22번 포트를 점유한 프로세스가 `sshd`(SSH daemon)로 표시된다.

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

> ### 따라 하기 3-1. 서비스 상태 조회와 제어
>
> **목적** SSH 서비스를 대상으로 상태를 조회하고, 정지·시작·자동 시작 등록을 실습한다.
{: .prompt-tip }

**1단계.** SSH 서비스의 상태를 조회한다.

```bash
systemctl status ssh
```

> `active (running)`이면 실행 중이다. `q`를 눌러 조회 화면을 빠져나온다.

**2단계.** 실행 여부와 자동 시작 여부를 간략히 확인한다.

```bash
systemctl is-active ssh
```

```bash
systemctl is-enabled ssh
```

> 각각 `active`, `enabled`로 표시되는지 확인한다.

**3단계.** 자동 시작을 등록한다(이미 등록되어 있으면 그대로 진행된다).

```bash
sudo systemctl enable ssh
```

> 재부팅 후에도 SSH 서버가 스스로 켜지도록 등록한다.

> **주의** SSH 서비스를 `stop`하면 원격 접속이 끊어진다. 자기 자신에게만 접속한 실습 환경에서는 안전하나, 실제 원격 서버에서는 SSH를 정지해서는 안 된다.

> **확인 사항** SSH 서비스가 `active`이고 `enabled`임을 확인하였다면 성공이다.
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

**3단계.** 서비스가 실행 중이고 자동 시작으로 등록되었는지 확인한다.

```bash
systemctl is-active ssh
```

```bash
systemctl is-enabled ssh
```

> 각각 `active`, `enabled`인지 확인한다. `enabled`가 아니면 `sudo systemctl enable ssh`로 등록한다.

**4단계.** SSH가 22번 포트에서 대기 중인지 확인한다.

```bash
sudo ss -tlnp | grep :22
```

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
> 2. SSH 서비스가 `active`이고 `enabled`임을 확인하였다.
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
| `systemctl` 정지 후 원격 접속 끊김 | SSH 서비스를 `stop`함 | 콘솔에서 `sudo systemctl start ssh`로 다시 시작한다. |
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
| 키 인증 | 비밀번호 대신 키 쌍으로 인증한다. `~/.ssh` 권한이 느슨하면 키가 무시된다. |
| 파일 전송 | `scp`는 SSH 위에서 파일을 암호화하여 복사한다. |
| systemd | `systemctl`로 서비스를 제어한다. "실행 중(active)"과 "자동 시작(enabled)"은 별개이다. |
| 방화벽 | `ufw`로 포트를 관리하며, **SSH 허용 → 기본 정책 → 활성화** 순서를 지킨다. |
| 포트 개념 | 서비스가 동작하는 것과 외부에서 접근 가능한 것은 별개의 문제이다. |

---

## 7.2 본 강의에서 학습한 명령어

| 명령어 | 기능 |
|---|---|
| `ssh` | 원격 접속 |
| `ssh-keygen` / `ssh-copy-id` | 키 생성 / 등록 |
| `scp` | 파일 전송 |
| `systemctl status` / `start` / `stop` | 서비스 상태 조회 / 시작 / 정지 |
| `systemctl enable` / `disable` | 자동 시작 등록 / 해제 |
| `systemctl is-active` / `is-enabled` | 실행 여부 / 자동 시작 여부 확인 |
| `ufw allow` / `enable` / `status` | 방화벽 규칙 / 활성화 / 조회 |
| `ss -tlnp` | 개방된 포트와 프로세스 조회 |
