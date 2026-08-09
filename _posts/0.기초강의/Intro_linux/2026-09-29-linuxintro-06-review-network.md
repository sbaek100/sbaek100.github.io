---
title: 리눅스 기초 6강 종합문제 - 지사 서버 인수와 서비스 공개
date: 2026-09-29 09:30:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - 종합문제
  - 복습
  - 네트워크
  - ss
  - netplan
  - ufw
  - SSH
  - nginx
pin:
mermaid: false
---

> **종합문제 안내**
> 1. 본 글은 **제6강. 네트워크 설정과 리눅스의 활용**의 복습용 종합문제이다.
> 2. 제6강의 실습 결과에 의존하지 않으나, **제0강에서 구축한 네트워크 환경**(어댑터 1 = NAT, 어댑터 2 = 호스트 전용 `192.168.56.30`)을 전제로 한다.
> 3. 문제 4는 **네트워크 설정을 변경**한다. 접속이 끊길 위험이 있으므로 `netplan try`를 사용하며, 콘솔 접근이 불가능하면 문법 검사까지만 수행한다.
> 4. 문제 5 이후는 방화벽을 활성화한다. **SSH 허용을 먼저 수행**하지 않으면 접속이 단절되므로 순서를 반드시 지킨다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 | 배정 시간 |
|---|---|---|---|
| 준비 | 도구 설치와 현황 기록 | 2.2 | 10분 |
| 문제 1 | 5단계 진단 절차 | 2.3 | 15분 |
| 문제 2 | 포트와 서비스 조회 | 1.2 · 2.2 | 10분 |
| 문제 3 | 이름 해석의 우선순위 | 1.3 | 10분 |
| 문제 4 | netplan 설정과 안전 적용 | 2.4 | 15분 |
| 문제 5 | 방화벽 구성 | 3.1 | 15분 |
| 문제 6 | 웹 서비스 공개 | 3.1 · 4.4 | 20분 |
| 문제 7 | SSH 키 인증과 파일 전송 | 3.2 · 3.3 | 20분 |
| 문제 8 | 점검 도구 작성 | 전 범위 | 15분 |
| 마무리 | 자가 채점과 정리 | — | 10분 |

---
---

# 시나리오

---

학습자는 본사 인프라팀 소속으로, **지사에 설치된 서버 한 대를 원격으로 인수**하게 되었다. 인수인계 문서에는 IP 주소만 적혀 있고 나머지 정보는 없다.

지사 담당자는 다음과 같이 요청하였다.

> *"이 서버에 부서 안내 페이지를 올려 주십시오. 조건이 있습니다.*
> *첫째, 네트워크 상태를 먼저 진단하여 이상이 없는지 확인해 주십시오.*
> *둘째, 외부에서 웹 페이지가 보여야 하지만 불필요한 포트는 열지 말아 주십시오.*
> *셋째, 앞으로는 비밀번호 없이 안전하게 접속할 수 있도록 구성해 주십시오.*
> *넷째, 상태를 한 번에 확인할 수 있는 점검 도구를 남겨 주십시오."*

> **원격 작업의 원칙**
> 본 종합문제의 모든 작업은 **자기 자신의 접속을 끊지 않는 것**을 최우선으로 한다. 네트워크 설정과 방화벽은 잘못 적용하면 즉시 접속이 단절되어 복구가 불가능해지므로, 각 문제에 제시된 **안전 절차와 순서**를 반드시 지킨다.
{: .prompt-danger }

---
---

# 준비 단계. 도구 설치와 현황 기록

---

```bash
mkdir -p ~/review06 && cd ~/review06
```

```bash
sudo apt update
```

```bash
sudo apt install -y bind9-dnsutils traceroute
```

> **`dig`·`host`·`nslookup`·`traceroute`는 Ubuntu Server 24.04에 기본 설치되어 있지 않다.** 설치하지 않고 실행하면 `command not found`가 출력된다. `bind9-dnsutils` 패키지가 DNS 조회 명령 세 개를 함께 제공한다.

작업 전 상태를 기록해 둔다. **변경 작업 전에 원래 상태를 남기는 것은 원격 작업의 기본 수칙이다.**

```bash
{
  echo "===== 작업 전 상태 ($(date '+%F %T')) ====="
  echo "[주소]";      ip -br addr
  echo "[경로]";      ip route
  echo "[개방 포트]"; ss -tuln
  echo "[방화벽]";    sudo ufw status
} > before.txt
cat before.txt
```

---
---

# 문제 1. 5단계 진단 절차

---

> **상황**
> 인수 대상 서버의 네트워크가 정상인지 확인하여야 한다.
>
> **요구사항** — 다음 다섯 단계를 **순서대로** 수행하고, 각 단계가 실패했을 때 어느 구간의 문제인지 설명한다.
>
> | 순서 | 확인 대상 | 실패 시 원인 구간 |
> |---|---|---|
> | ① | 루프백 | ? |
> | ② | 자신의 IP 주소 | ? |
> | ③ | 기본 게이트웨이 | ? |
> | ④ | 외부 IP(`8.8.8.8`) | ? |
> | ⑤ | 도메인 이름 | ? |
>
> 게이트웨이 주소는 **직접 입력하지 말고 명령으로 추출**하여 사용한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
ping -c 2 127.0.0.1

ip -br addr
ip route

ping -c 2 $(ip route | awk '/default/ {print $3}')

ping -c 2 8.8.8.8

ping -c 2 www.google.com
resolvectl query www.google.com
```

**해설**

| 순서 | 실패 시 원인 구간 |
|---|---|
| ① 루프백 | **TCP/IP 스택 자체**의 문제 |
| ② IP 주소 | **주소 설정**(netplan) 문제 |
| ③ 게이트웨이 | **케이블·스위치 등 로컬 네트워크** 문제 |
| ④ 외부 IP | **라우팅 또는 외부 회선** 문제 |
| ⑤ 도메인 | **DNS** 문제 |

- 이 순서는 **가까운 곳에서 먼 곳으로** 범위를 넓히며 원인 구간을 좁혀 나가는 방식이다. 어떤 장애든 이 절차를 적용하면 원인을 특정할 수 있다.
- `ip route | awk '/default/ {print $3}'`은 기본 게이트웨이 주소만 추출한다. 본 과정의 환경에서는 어댑터 1(NAT)의 **`10.0.2.2`** 가 출력된다. 호스트 전용 어댑터에는 게이트웨이를 지정하지 않았기 때문이다.
- **"IP로는 되는데 도메인으로만 실패한다"는 증상은 DNS 문제**이다. ④는 성공하고 ⑤만 실패하는 경우가 이에 해당한다.
- 인터페이스 이름이 `eth0`이 아닌 `enp0s3` 형태인 것은 systemd의 **예측 가능한 인터페이스 이름** 규칙 때문이며, 카드 순서가 바뀌어도 이름이 유지되도록 한 것이다.

</details>

**완료 기준** — 다섯 단계가 모두 성공하고, 각 단계의 의미를 설명할 수 있다.

---
---

# 문제 2. 포트와 서비스 조회

---

> **상황**
> 인수 시점에 어떤 서비스가 외부에 열려 있는지 파악하여야 한다.
>
> **요구사항**
> 1. **대기 중인** TCP·UDP 포트를 번호 형식으로 조회한다.
> 2. 각 포트를 **어떤 프로그램이 점유**하고 있는지 함께 조회한다.
> 3. 22번 포트의 대기 상태만 추출한다.
> 4. 포트 번호와 서비스 이름의 대응이 정의된 파일에서 `ssh`·`http`·`https`·`domain` 항목을 조회한다.
> 5. `netstat`을 사용하던 관행을 신형 명령으로 대응시켜 정리한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
ss -tuln
sudo ss -tulnp
sudo ss -tulnp | grep ":22"
grep -E "^(ssh|http|https|domain)[[:space:]]" /etc/services
```

**해설**

- `ss -tulnp`의 각 문자는 **t**cp, **u**dp, **l**isten(대기 중만), **n**umeric(이름 대신 번호), **p**rocess(점유 프로그램)를 의미한다. `-p`는 관리자 권한이 필요하다.
- 22번 포트를 점유한 프로세스가 `sshd`가 아니라 **`systemd`(PID 1)** 로 표시될 수 있다. Ubuntu 22.10 이후의 OpenSSH는 **소켓 활성화** 방식을 사용하여, 평소에는 `ssh.socket`이 포트를 지키다가 접속이 들어오는 순간 `ssh.service`를 기동하기 때문이다.
- 구형·신형 명령의 대응은 다음과 같으며 시험에서는 양쪽 모두 출제된다.
>
> | 구형(net-tools) | 신형(iproute2) |
> |---|---|
> | `ifconfig` | `ip addr` |
> | `route -n` | `ip route` |
> | `netstat -tulnp` | **`ss -tulnp`** |
> | `arp -a` | `ip neigh` |
>
- Ubuntu 24.04에는 `net-tools`가 기본 설치되어 있지 않다. 신형 명령을 표준으로 익혀야 한다.

</details>

**완료 기준** — 22번 포트가 대기 중임을 확인하였고, `netstat`과 `ss`의 대응 관계를 설명할 수 있다.

---
---

# 문제 3. 이름 해석의 우선순위

---

> **상황**
> 지사 서버를 `branch-web`이라는 별칭으로 부르기로 하였다. DNS에 등록하지 않고 이 서버에서만 인식되도록 설정한다.
>
> **요구사항**
> 1. 이름 해석의 **순서를 정의하는 파일**에서 `hosts` 항목을 확인한다.
> 2. `branch-web`이라는 이름이 자신의 실습망 주소(`192.168.56.30`)로 해석되도록 등록한다.
> 3. 등록한 이름이 실제로 해석되는지 확인한다.
> 4. 해당 이름으로 통신이 되는지 확인한다.
> 5. 실습이 끝나면 등록을 제거한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
grep hosts /etc/nsswitch.conf
cat /etc/hosts

echo "192.168.56.30 branch-web" | sudo tee -a /etc/hosts

getent hosts branch-web
ping -c 2 branch-web

sudo sed -i '/branch-web/d' /etc/hosts
getent hosts branch-web || echo "등록이 제거되었습니다"
```

**해설**

- `/etc/nsswitch.conf`의 `hosts: files dns` 항목에서 **`files`(즉 `/etc/hosts`)가 `dns`보다 앞에** 있다. 따라서 DNS 질의보다 `/etc/hosts`가 **먼저 참조**된다.
- 이 특성은 개발·시험 환경에서 유용하다. 아직 DNS에 등록되지 않은 이름을 미리 사용해 보거나, 특정 도메인을 임시로 다른 주소로 돌릴 때 사용한다.
- 동시에 **악성 코드가 이 파일을 조작하여 정상 사이트를 위조 사이트로 연결하는 공격**에도 이용된다. 침해 대응 시 `/etc/hosts`의 무단 변경 여부를 점검하는 이유이다.
- 확인에는 `ping`보다 **`getent hosts`** 가 적합하다. `ping`은 이름 해석과 통신을 동시에 시도하므로, 상대가 ICMP에 응답하지 않으면 해석이 성공했는지 판단하기 어렵다. `getent`는 **이름 해석만** 수행한다.

</details>

**완료 기준** — `getent hosts branch-web`이 `192.168.56.30`을 반환하고, 제거 후에는 아무것도 반환하지 않는다.

---
---

# 문제 4. netplan 설정과 안전 적용

---

> **상황**
> 지사 담당자가 "실습망 주소를 `.31`로 바꾸어 달라"고 요청하였다. 원격 접속 중이므로 신중하게 진행하여야 한다.
>
> **요구사항**
> 1. 현재 설정 파일을 확인하고 **백업본을 만든다.**
> 2. 인터페이스 이름을 명령으로 추출한다.
> 3. 어댑터 2의 주소만 `192.168.56.31/24`로 바꾸는 설정 파일을 작성한다(어댑터 1은 건드리지 않는다).
> 4. 파일 권한을 `600`으로 조정한다.
> 5. **적용하지 않고 문법만 검사**한다.
> 6. 안전장치가 있는 명령으로 적용을 시도한다.
> 7. 원래 설정으로 복원하고, 이전 주소가 남아 있지 않은지 확인한다.
{: .prompt-info }

> **콘솔 접근이 불가능한 경우 5단계까지만 수행한다.** 주소를 바꾸면 현재의 SSH 세션이 끊어지기 때문이다.
{: .prompt-warning }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
ls -l /etc/netplan/
sudo cat /etc/netplan/*.yaml
sudo cp -a /etc/netplan /root/netplan.review06.bak

IF_NAT=$(ls /sys/class/net | grep -v lo | sed -n '1p')
IF_LAB=$(ls /sys/class/net | grep -v lo | sed -n '2p')
echo "어댑터1(NAT): $IF_NAT / 어댑터2(호스트 전용): $IF_LAB"

sudo tee /etc/netplan/99-review06.yaml > /dev/null << EOF
network:
  version: 2
  ethernets:
    ${IF_NAT}:
      dhcp4: true
    ${IF_LAB}:
      dhcp4: false
      addresses: [192.168.56.31/24]
EOF

sudo chmod 600 /etc/netplan/99-review06.yaml
sudo cat /etc/netplan/99-review06.yaml

sudo netplan generate
sudo netplan get

# 아래는 콘솔 접근이 가능한 경우에만 수행한다
sudo netplan try

# 복원
sudo rm -f /etc/netplan/99-review06.yaml
sudo netplan apply
ip -br addr
```

**해설**

- **`netplan generate`는 문법 검사와 백엔드 설정 생성만 수행하며 적용하지 않는다.** 오류가 없으면 아무 메시지도 출력되지 않는다. 원격 작업에서 가장 먼저 실행하는 안전한 검증 수단이다.
- **`netplan try`** 는 적용 후 120초 동안 확인 입력을 대기하며, 입력이 없으면 **자동으로 이전 설정을 복원**한다. 주소가 바뀌어 세션이 끊기면 확인할 수 없으므로, 그대로 기다리면 원래 주소로 되살아난다. 원격 작업에서는 `apply` 대신 반드시 이 명령을 사용한다.
- 파일 이름이 `99-`로 시작하는 이유는 netplan이 파일을 **사전순으로 읽고 나중 파일이 앞의 설정을 덮어쓰기** 때문이다.
- YAML은 **탭 문자를 허용하지 않으며 공백 들여쓰기만** 인정한다. Ubuntu 24.04는 설정 파일의 권한이 `600`이 아니면 경고를 출력한다.
- 어댑터 1(NAT)을 그대로 두는 이유는, 여기에 고정 주소를 지정하면 **인터넷 연결이 끊겨 패키지 설치가 불가능해지기** 때문이다.
- 복원 후 `ip -br addr`에 **`.31`이 남아 있는 경우가 있다.** `netplan apply`는 새 설정을 적용할 뿐 이전 주소를 항상 회수하지는 않기 때문이다. 이때는 다음과 같이 직접 제거하거나 재부팅한다.
>
> ```bash
> sudo ip addr del 192.168.56.31/24 dev $IF_LAB
> ```
>
> **"설정 파일의 내용"과 "현재 인터페이스의 상태"는 서로 다른 대상**이라는 점을 기억한다.

</details>

**완료 기준** — `netplan generate`가 오류 없이 통과하고, 복원 후 주소가 `192.168.56.30`이다.

---
---

# 문제 5. 방화벽 구성

---

> **상황**
> 두 번째 요청("불필요한 포트는 열지 말 것")을 구현한다.
>
> **요구사항**
> 1. 현재 방화벽 상태와 사용 가능한 애플리케이션 프로파일을 확인한다.
> 2. **올바른 순서로** 방화벽을 구성한다.
> 3. 활성화 후에도 현재 SSH 세션이 유지되는지 확인한다.
> 4. 데이터베이스 포트(3306)를 **실습망 대역에서만** 접근할 수 있도록 규칙을 추가한다.
> 5. 규칙에 번호를 붙여 조회하고, 4번에서 추가한 규칙을 삭제한다.
{: .prompt-info }

**힌트** — 순서를 틀리면 접속이 끊긴다. 무엇을 가장 먼저 해야 하는가.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sudo ufw status verbose
sudo ufw app list

# ① SSH를 먼저 허용한다
sudo ufw allow OpenSSH
sudo ufw status

# ② 기본 정책을 설정한다
sudo ufw default deny incoming
sudo ufw default allow outgoing

# ③ 활성화한다
sudo ufw enable
sudo ufw status verbose

# 출발지를 제한한 규칙
sudo ufw allow from 192.168.56.0/24 to any port 3306 proto tcp
sudo ufw status numbered

sudo ufw delete allow from 192.168.56.0/24 to any port 3306 proto tcp
sudo ufw status numbered
```

**해설**

- **순서가 전부이다.** `default deny incoming`을 설정한 뒤 SSH 허용 없이 `enable`하면 **그 즉시 원격 접속이 끊어지고 콘솔로만 복구할 수 있다.** 실무에서 매우 빈번하게 발생하는 사고이다.
>
> ```
> ① sudo ufw allow OpenSSH          ← SSH를 먼저 허용
> ② sudo ufw default deny incoming  ← 기본 정책
> ③ sudo ufw enable                 ← 활성화
> ```
>
- `ufw`는 **U**ncomplicated **F**ire**w**all의 약어로, 커널의 `nftables`/`iptables`를 간결한 명령으로 조작한다.
- `allow OpenSSH`처럼 **애플리케이션 프로파일**을 사용하면 포트 번호를 외우지 않아도 되고, 서비스가 여러 포트를 쓰는 경우에도 한 번에 처리된다. 사용 가능한 목록은 `ufw app list`로 확인한다.
- 데이터베이스 포트를 전체 대역에 개방하는 것은 매우 위험하다. **`from 대역`으로 출발지를 제한**하는 것이 원칙이며, 이것이 최소 권한 원칙의 네트워크 적용 사례이다.
- 규칙 삭제는 추가할 때와 **동일한 문구**를 `delete` 뒤에 그대로 쓰거나, `ufw status numbered`로 번호를 확인한 뒤 `sudo ufw delete 번호`로 지운다.

</details>

**완료 기준** — `ufw status`가 `Status: active`이고 OpenSSH 규칙이 존재하며, 현재 접속이 유지되고 있다.

---
---

# 문제 6. 웹 서비스 공개

---

> **상황**
> 부서 안내 페이지를 올려야 한다. 서비스가 동작하는 것과 외부에서 접근되는 것은 별개의 문제이다.
>
> **요구사항**
> 1. 웹 서버를 설치하고 **실행 여부와 자동 시작 여부**를 각각 확인한다.
> 2. 웹 서버 프로세스의 **실행 계정**을 확인하고, master와 worker의 계정이 다른 이유를 설명한다.
> 3. 안내 페이지를 작성한다.
> 4. 콘텐츠 파일의 소유와 권한을 **`root:www-data` · `640`** 으로 설정하고 그 근거를 설명한다.
> 5. 로컬에서 접속을 확인한다.
> 6. 방화벽에 HTTP를 개방한다.
> 7. **윈도우 호스트의 브라우저에서** 접속할 주소를 정확히 판단한다.
> 8. 서비스를 중지·시작·재적재하며 `restart`와 `reload`의 차이를 설명한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sudo apt install -y nginx
systemctl is-active nginx
systemctl is-enabled nginx

ps -eo user,pid,comm | grep nginx
grep "^www-data" /etc/passwd

sudo tee /var/www/html/index.html > /dev/null << 'EOF'
<!doctype html>
<html lang="ko">
<head><meta charset="utf-8"><title>지사 안내</title></head>
<body style="font-family: sans-serif; max-width: 640px; margin: 60px auto;">
  <h1>지사 부서 안내</h1>
  <p>Ubuntu Server 24.04 LTS / nginx</p>
  <hr>
  <p>리눅스 기초 6강 종합문제로 구축한 페이지입니다.</p>
</body>
</html>
EOF

sudo chown root:www-data /var/www/html/index.html
sudo chmod 640 /var/www/html/index.html
ls -l /var/www/html/

curl -I http://localhost
curl -s http://localhost | head -5

sudo ufw allow 'Nginx HTTP'
sudo ufw status numbered

hostname -I
hostname -I | tr ' ' '\n' | grep "^192.168.56."

sudo systemctl stop nginx
curl -I http://localhost 2>&1 | head -2
sudo systemctl start nginx
sudo nginx -t
sudo systemctl reload nginx
```

**해설**

- **master 프로세스는 `root`, worker 프로세스는 `www-data`** 로 실행된다. 1023번 이하인 80번 포트를 바인딩하려면 관리자 권한이 필요하지만, 실제 요청 처리는 **낮은 권한의 전용 계정**이 맡는다. 웹 서버가 침해되더라도 공격자가 얻는 권한은 `www-data`에 한정되는 구조이며, **최소 권한 원칙**의 대표적 구현이다.
- `www-data`의 로그인 셸은 `/usr/sbin/nologin`이다. 서비스 전용 계정에는 대화형 로그인이 불필요하기 때문이다.
- 콘텐츠 권한 `640`(`root:www-data`)의 근거는 다음과 같다.
>
> | 대상 | 권한 | 근거 |
> |---|---|---|
> | 소유자 root | 6 | 관리자가 콘텐츠를 수정한다 |
> | 그룹 www-data | 4 | **웹 서버는 읽기만 하면 충분하다** |
> | 기타 | 0 | 그 외에는 접근할 필요가 없다 |
>
> 웹 서버에 쓰기 권한을 주면 취약점을 통한 파일 업로드·변조가 가능해진다.
- **접속 주소에 주의한다.** 이 서버에는 어댑터가 두 개 있고 `hostname -I`는 어댑터 1(NAT)의 `10.0.2.15`를 **먼저** 출력한다. NAT 주소는 가상 머신 내부에서만 유효하므로 윈도우 호스트에서는 접속되지 않는다. 외부에서 사용할 주소는 **호스트 전용 어댑터의 `192.168.56.30`** 이다.
- **`restart`와 `reload`의 차이**
>
> | 명령 | 동작 | 서비스 중단 |
> |---|---|---|
> | `restart` | 프로세스를 종료 후 재기동 | **순간적으로 발생** |
> | `reload` | 설정만 다시 읽음 | **없음(무중단)** |
>
> 운영 중인 서버에서는 `nginx -t`로 설정 문법을 검증한 뒤 `reload`를 사용하는 것이 원칙이다.

</details>

**완료 기준** — 윈도우 브라우저에서 `http://192.168.56.30/`으로 페이지가 표시된다.

---
---

# 문제 7. SSH 키 인증과 파일 전송

---

> **상황**
> 세 번째 요청("비밀번호 없이 안전하게 접속")을 구현한다. 별도 장비가 없으므로 **자기 자신에게 접속하는 방식**으로 실습한다.
>
> **요구사항**
> 1. SSH 서비스의 상태를 확인한다.
> 2. 키 쌍을 생성한다(알고리즘은 `ed25519`, 실습 편의상 키 비밀번호는 생략한다).
> 3. 공개 키를 등록하고 `~/.ssh`와 `authorized_keys`의 권한을 확인한다.
> 4. **비밀번호 없이** 접속되는지 확인하고, 접속과 동시에 명령을 실행해 본다.
> 5. `scp`로 파일을 전송한다.
> 6. `rsync`로 디렉터리를 동기화한 뒤, 파일 하나를 수정하고 **다시 동기화하여 전송량의 차이**를 관찰한다.
> 7. SSH 서버의 보안 설정 항목을 조회하고 문법을 검사한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
systemctl is-active ssh
systemctl is-active ssh.socket
sudo ss -tlnp | grep ":22"

ssh-keygen -t ed25519 -C "review06-$(whoami)" -f ~/.ssh/id_ed25519 -N ""
ls -l ~/.ssh/
cat ~/.ssh/id_ed25519.pub

ssh-copy-id localhost
ls -ld ~/.ssh && ls -l ~/.ssh/authorized_keys

ssh localhost "hostname; whoami; date"

echo "전송 시험 파일" > ~/review06/transfer.txt
mkdir -p ~/review06/received
scp ~/review06/transfer.txt localhost:~/review06/received/
ls -l ~/review06/received/

rsync -avz ~/review06/ localhost:/tmp/review06_sync/
echo "추가한 내용" >> ~/review06/transfer.txt
rsync -avz ~/review06/ localhost:/tmp/review06_sync/

grep -iE "PermitRootLogin|PasswordAuthentication|Port " /etc/ssh/sshd_config
sudo sshd -t && echo "설정 문법에 이상이 없습니다"

rm -rf /tmp/review06_sync ~/review06/received
```

**해설**

- `systemctl is-active ssh`가 `inactive`로 나와도 정상일 수 있다. Ubuntu 22.10 이후에는 **소켓 활성화**가 기본이므로, 접속이 없으면 `ssh.service`는 내려가 있고 `ssh.socket`만 대기한다. 다만 지금은 SSH로 접속한 상태이므로 둘 다 `active`이다.
- 키 쌍은 **개인 키(`id_ed25519`)와 공개 키(`id_ed25519.pub`)** 로 이루어진다. 개인 키는 절대 유출해서는 안 되며, 공개 키만 서버의 `~/.ssh/authorized_keys`에 등록한다.
- **권한이 핵심이다.** SSH는 보안상 `~/.ssh`가 `700`이 아니거나 `authorized_keys`가 `600`이 아니면 해당 키를 **무시한다.** "키를 등록했는데도 비밀번호를 계속 묻는" 문제의 대부분이 이 원인이다.
>
> | 대상 | 필수 권한 |
> |---|---|
> | `~/.ssh` | **700** |
> | `~/.ssh/id_ed25519`(개인 키) | **600** |
> | `~/.ssh/authorized_keys` | **600** |
>
- `ssh 호스트 "명령"` 형태는 접속과 동시에 명령을 실행하고 결과만 반환한다. **다수의 서버를 자동으로 관리하는 스크립트의 기본 구조**이다.
- `scp`는 매번 전체를 복사하지만 **`rsync`는 변경된 부분만 전송**한다. 두 번째 실행에서 전송 목록이 현저히 줄어드는 것을 확인할 수 있으며, 이 때문에 백업과 배포에는 `rsync`가 표준으로 사용된다.
- 실무에서는 키에 **반드시 비밀번호(passphrase)를 설정**하고 `ssh-agent`로 관리한다. 본 실습에서는 편의상 `-N ""`으로 생략하였다.
- 서버 측 보안 강화 항목은 `PermitRootLogin no`, `PasswordAuthentication no`(키 인증 구성 후), `AllowUsers` 제한이다. **`PasswordAuthentication no`로 바꾸기 전에는 반드시 별도 세션에서 키 인증이 동작하는지 확인**하여야 하며, 변경 후에는 `sudo sshd -t`로 문법을 검사한 뒤 재시작한다.

</details>

**완료 기준** — `ssh localhost "hostname"`이 비밀번호 없이 결과를 반환한다.

---
---

# 문제 8. 점검 도구 작성

---

> **상황**
> 네 번째 요청("상태를 한 번에 확인할 수 있는 점검 도구")을 구현한다.
>
> **요구사항** — 다음 여섯 항목을 출력하는 `netcheck.sh`를 작성한다.
>
> | 항목 | 내용 |
> |---|---|
> | [1] | 인터페이스별 주소와 기본 게이트웨이 |
> | [2] | 인터넷 연결과 DNS 조회의 성공 여부 |
> | [3] | 대기 중인 포트 목록 |
> | [4] | 방화벽 활성화 여부 |
> | [5] | 웹 서비스의 실행·자동 시작·HTTP 응답 코드 |
> | [6] | 콘텐츠 파일의 권한과 소유 |
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cat > ~/review06/netcheck.sh << 'EOF'
#!/bin/bash
# 지사 서버 네트워크·서비스 점검 도구

echo "=========================================="
echo "  지사 서버 점검 보고"
echo "  점검 일시: $(date '+%Y-%m-%d %H:%M:%S')"
echo "  대상 호스트: $(hostname)"
echo "=========================================="
echo

echo "[1] 네트워크 구성"
ip -br addr | grep -v "^lo" | awk '{print "  인터페이스: " $1 "  주소: " $3}'
echo "  게이트웨이: $(ip route | awk '/default/ {print $3}')"
echo

echo "[2] 외부 통신"
ping -c 1 -W 2 8.8.8.8 > /dev/null 2>&1 \
  && echo "  인터넷 연결: 정상" || echo "  인터넷 연결: 실패"
getent hosts www.example.com > /dev/null 2>&1 \
  && echo "  DNS 조회   : 정상" || echo "  DNS 조회   : 실패"
echo

echo "[3] 대기 중인 포트"
ss -tlnH | awk '{print "  " $4}' | sort -u
echo

echo "[4] 방화벽"
if FW=$(sudo -n ufw status 2>/dev/null); then
  echo "$FW" | head -1 | sed 's/^/  /'
else
  echo "  (권한 없음 — 관리자 권한으로 실행하면 표시됨)"
fi
echo

echo "[5] 웹 서비스"
echo "  실행 여부  : $(systemctl is-active nginx)"
echo "  자동 시작  : $(systemctl is-enabled nginx)"
curl -s -o /dev/null -w "  HTTP 응답  : %{http_code}\n" http://localhost
echo

echo "[6] 콘텐츠 파일"
echo "  $(stat -c '%A %U:%G %n' /var/www/html/index.html 2>/dev/null)"
EOF

chmod 750 ~/review06/netcheck.sh
~/review06/netcheck.sh
~/review06/netcheck.sh > ~/review06/after.txt
```

**해설**

- **판정은 명령의 종료 상태로 한다.** `ping -c 1 -W 2`는 1회만 보내고 2초 안에 응답이 없으면 실패로 처리하므로, 점검 도구가 오래 멈추지 않는다.
- DNS 확인에 `dig` 대신 **`getent hosts`** 를 사용한 이유는, 기본 설치 환경에 `dig`가 없을 수 있고 `getent`는 시스템의 이름 해석 경로를 그대로 따르기 때문이다.
- `curl -s -o /dev/null -w "%{http_code}"`는 본문을 버리고 **응답 코드만** 얻는 관용적 표현이다. `200`이면 정상이다.
- `[4]`에서 결과를 **변수에 담은 뒤 `if`로 판정**한 이유가 있다. `sudo -n ufw status | head -1 || echo "..."` 형태로 쓰면 파이프라인의 성공 여부를 **마지막 명령(`head`)** 이 결정하므로, `sudo`가 실패해도 대체 메시지가 출력되지 않는다.
- `sudo -n`은 비밀번호를 묻지 않고 즉시 실패하게 한다. 이 도구를 `cron`에 등록하면 터미널이 없어 비밀번호를 입력할 수 없으므로, 이런 형태로 작성해 두어야 나머지 항목이 정상적으로 기록된다.

</details>

**완료 기준** — 여섯 항목이 모두 출력되고 `[5]`의 HTTP 응답이 `200`이다.

---
---

# 마무리. 자가 채점

---

```bash
cd ~/review06
echo "===== 자가 채점 결과 ====="

ping -c 1 -W 2 8.8.8.8 > /dev/null 2>&1 \
  && echo "[문제 1] 통과 — 외부 통신 정상" || echo "[문제 1] 미완료"

ss -tlnH | grep -q ":22" \
  && echo "[문제 2] 통과 — 22번 포트 대기" || echo "[문제 2] 미완료"

sudo ufw status | head -1 | grep -q "active" \
  && echo "[문제 5-1] 통과 — 방화벽 활성화" || echo "[문제 5-1] 미완료"

sudo ufw status | grep -qi "OpenSSH" \
  && echo "[문제 5-2] 통과 — SSH 허용 규칙 존재" || echo "[문제 5-2] 미완료"

[ "$(systemctl is-active nginx)" = "active" ] \
  && echo "[문제 6-1] 통과 — 웹 서비스 실행" || echo "[문제 6-1] 미완료"

[ "$(stat -c '%a' /var/www/html/index.html 2>/dev/null)" = "640" ] \
  && echo "[문제 6-2] 통과 — 콘텐츠 권한 640" || echo "[문제 6-2] 미완료"

[ "$(curl -s -o /dev/null -w '%{http_code}' http://localhost)" = "200" ] \
  && echo "[문제 6-3] 통과 — HTTP 200 응답" || echo "[문제 6-3] 미완료"

[ -f ~/.ssh/authorized_keys ] && [ "$(stat -c '%a' ~/.ssh)" = "700" ] \
  && echo "[문제 7-1] 통과 — 키 등록과 권한" || echo "[문제 7-1] 미완료"

ssh -o BatchMode=yes -o ConnectTimeout=5 localhost "true" 2>/dev/null \
  && echo "[문제 7-2] 통과 — 비밀번호 없이 접속" || echo "[문제 7-2] 미완료"

[ -x ~/review06/netcheck.sh ] && [ -s ~/review06/after.txt ] \
  && echo "[문제 8] 통과 — 점검 도구 작성" || echo "[문제 8] 미완료"
```

> `ssh -o BatchMode=yes`는 **비밀번호를 물어야 하는 상황이면 즉시 실패**하게 한다. 따라서 이 명령이 성공했다는 것은 **키 인증만으로 접속되었다**는 증거가 된다.
{: .prompt-tip }

---

## 이론 점검

**문항 1.** "IP 주소로는 통신되는데 도메인 이름으로만 실패한다"는 증상의 원인은?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**DNS 문제**이다. 이름을 IP로 변환하는 단계에서만 실패하고 있기 때문이다.

</details>

**문항 2.** 원격 접속 중 `ufw`를 활성화할 때 가장 먼저 해야 할 일은?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**SSH를 허용하는 규칙을 먼저 추가**하는 것이다(`sudo ufw allow OpenSSH`). 기본 정책을 deny로 두고 그냥 활성화하면 즉시 접속이 끊긴다.

</details>

**문항 3.** 원격지에서 네트워크 설정을 바꿀 때 `netplan apply` 대신 사용해야 할 명령은?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**`netplan try`** 이다. 적용 후 120초 안에 확인 입력이 없으면 **자동으로 이전 설정을 복원**하므로 접속이 끊겨도 되살아난다.

</details>

**문항 4.** 키를 등록했는데도 비밀번호를 계속 요구받는 가장 흔한 원인은?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**권한 설정 문제**이다. `~/.ssh`가 `700`, `authorized_keys`가 `600`이 아니면 SSH가 해당 키를 무시한다.

</details>

**문항 5.** nginx의 worker 프로세스를 `www-data` 계정으로 실행하는 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**최소 권한 원칙**의 적용이다. 80번 포트 바인딩에는 root 권한이 필요하지만 요청 처리까지 root로 수행하면 침해 시 시스템 전체가 노출되므로, 실제 처리는 권한이 낮은 전용 계정이 맡는다.

</details>

**문항 6.** `hostname -I`의 첫 번째 주소를 외부 접속 주소로 사용하면 안 되는 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

어댑터가 두 개인 환경에서 첫 번째로 출력되는 주소는 **NAT 어댑터의 `10.0.2.x`** 이며, 이는 가상 머신 내부에서만 유효하다. 외부에서 접근할 주소는 호스트 전용 어댑터의 `192.168.56.30`이다.

</details>

---
---

# 정리 절차

---

```bash
sudo ufw delete allow 'Nginx HTTP'
```

```bash
sudo apt purge -y nginx nginx-common && sudo apt autoremove -y
```

```bash
sudo sed -i '/branch-web/d' /etc/hosts
```

```bash
sudo rm -f /etc/netplan/99-review06.yaml
```

```bash
rm -rf ~/review06
```

**정리 결과를 검증한다.**

```bash
sudo ufw status
```

```bash
ip -br addr
```

```bash
sudo netplan generate && echo "netplan 설정 정상"
```

> **방화벽은 활성 상태로 두어도 무방하다.** OpenSSH 규칙이 남아 있으면 접속에 지장이 없으며, 오히려 안전한 상태이다. 완전히 되돌리려면 `sudo ufw disable`을 실행한다.
>
> SSH 키(`~/.ssh/id_ed25519`)는 이후 실습에도 유용하므로 **삭제하지 않고 보존**할 것을 권장한다.
{: .prompt-tip }

---

## 자기 점검

```
 [ ] 5단계 진단 절차를 순서대로 수행하고 각 단계의 의미를 설명할 수 있다.
 [ ] ss -tulnp의 각 옵션이 무엇인지 설명할 수 있다.
 [ ] /etc/hosts가 DNS보다 먼저 참조된다는 사실을 확인하였다.
 [ ] netplan generate와 try의 차이를 설명할 수 있다.
 [ ] 방화벽 구성의 올바른 순서를 지켰다.
 [ ] 서비스 동작과 외부 접근 가능 여부가 별개임을 실험으로 확인하였다.
 [ ] SSH 키 인증을 구성하고 권한 요건을 설명할 수 있다.
 [ ] 두 개의 주소 중 외부 접속에 사용할 주소를 판단할 수 있다.
```

이것으로 제1강부터 제6강까지의 복습을 마쳤다. 마지막 복습은 **부록 종합문제 — 시스템 보안 감사**이며, 여섯 강의의 내용을 하나의 점검 체계로 통합한다.
