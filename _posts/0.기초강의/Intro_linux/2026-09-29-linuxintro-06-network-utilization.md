---
title: 리눅스 기초 6강 - 네트워크 설정과 리눅스의 활용
date: 2026-09-29 09:00:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - 네트워크
  - ip명령
  - ss
  - netplan
  - ufw
  - SSH
  - X윈도
  - 컨테이너
pin:
mermaid: false
---

> **학습 목표**
> 1. IP 주소·포트·게이트웨이·DNS의 개념과 상호 관계를 설명할 수 있다.
> 2. `ip`, `ping`, `ss`, `dig`로 네트워크 상태를 단계적으로 진단할 수 있다.
> 3. netplan의 설정 구조를 이해하고 안전하게 적용할 수 있다.
> 4. `ufw`로 방화벽을 구성하고 필요한 포트만 개방할 수 있다.
> 5. SSH로 원격 접속하고 키 기반 인증을 구성할 수 있다.
> 6. X 윈도의 구조를 설명하고 서버에서 그래픽 환경을 배제하는 이유를 논할 수 있다.
> 7. 리눅스의 주요 활용 분야와 컨테이너 기술의 기반을 설명할 수 있다.
{: .prompt-info }

지금까지의 강의는 시스템 내부의 작업을 다루었다. 본 강의에서는 시스템을 **외부 네트워크와 연결**하는 방법을 학습한다.

서버는 네트워크에 연결되어야 비로소 서비스를 제공할 수 있으며, 동시에 외부 공격에 노출되기도 한다. 따라서 네트워크 설정과 방화벽 구성은 함께 다루어야 하는 주제이다.

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 네트워크 기본 개념 | 30분 |
| 제2절 | 네트워크 상태 진단과 설정 | 40분 |
| 제3절 | 방화벽과 원격 접속 | 40분 |
| 제4절 | X 윈도와 리눅스의 활용 분야 | 25분 |
| 제5절 | 종합 실습 | 35분 |
| 제6·7절 | 오류 대응 및 이론 평가 | 10분 |

---
---

# 제1절. 네트워크 기본 개념

---

## 1.1 주소 체계

네트워크에서 데이터를 주고받으려면 송신처와 수신처를 식별할 수 있어야 한다.

| 개념 | 정의 | 비유 | 예시 |
|---|---|---|---|
| **IP 주소** | 네트워크상 호스트를 식별하는 주소 | 건물의 주소 | `192.168.56.30` |
| **포트 번호** | 호스트 내부의 서비스를 식별하는 번호 | 건물 내 호실 번호 | `22`, `80` |
| **게이트웨이** | 외부 네트워크로 나가는 출구 | 단지의 정문 | `10.0.2.2` |
| **DNS 서버** | 도메인 이름을 IP 주소로 변환하는 서버 | 전화번호부 | `8.8.8.8` |
| **서브넷 마스크** | 네트워크 부분과 호스트 부분의 경계 | 동일 단지의 범위 | `255.255.255.0`(=`/24`) |
| **MAC 주소** | 네트워크 인터페이스의 물리 주소 | 장비 고유 번호 | 48비트 |
| **루프백** | 자기 자신을 가리키는 주소 | — | `127.0.0.1` |

> **본 과정 실습 환경의 주소 구성**
> 제0강에서 구성한 환경은 네트워크 어댑터가 두 개이므로, 각 어댑터가 서로 다른 대역을 사용한다.
>
> | 어댑터 | 연결 방식 | 주소 | 용도 |
> |---|---|---|---|
> | 어댑터 1 | NAT | `10.0.2.x`(자동) | 인터넷 연결. **기본 게이트웨이는 `10.0.2.2`** |
> | 어댑터 2 | 호스트 전용 | **`192.168.56.30`**(고정) | 실습 통신, 호스트에서의 접속 |
>
> 따라서 본 강의의 실습에서 `ip route`로 확인하는 기본 게이트웨이는 `10.0.2.2`이고, 윈도우 호스트에서 접속할 때 사용하는 주소는 `192.168.56.30`이다.
{: .prompt-info }

하나의 서버(IP 주소 1개)에서 웹 서비스와 원격 접속 서비스를 동시에 제공할 수 있는 이유가 바로 **포트**이다. 동일한 건물이되 호실이 다른 것에 해당한다.

사설 IP 주소 대역은 다음과 같다.

| 클래스 | 대역 | CIDR 표기 |
|---|---|---|
| A | 10.0.0.0 ~ 10.255.255.255 | `10.0.0.0/8` |
| B | 172.16.0.0 ~ 172.31.255.255 | `172.16.0.0/12` |
| **C** | **192.168.0.0 ~ 192.168.255.255** | **`192.168.0.0/16`** |

본 과정의 실습 환경이 사용하는 `192.168.56.30` 역시 이 C 클래스 사설 대역에 속한다.

---

## 1.2 포트 번호와 주요 서비스

포트 번호는 16비트 값이므로 0부터 65535까지의 범위를 갖는다.

| 구간 | 명칭 | 특징 |
|---|---|---|
| **0 ~ 1023** | **잘 알려진 포트(Well-known)** | 표준 서비스용. **바인딩에 root 권한 필요** |
| 1024 ~ 49151 | 등록된 포트(Registered) | 응용 프로그램 등록 대역 |
| 49152 ~ 65535 | 동적·사설 포트 | 클라이언트의 임시 포트 |

주요 서비스의 포트 번호는 반드시 숙지하여야 한다.

| 포트 | 프로토콜 | 서비스 |
|---|---|---|
| 20 / 21 | TCP | FTP 데이터 / 제어 |
| **22** | TCP | **SSH, SCP, SFTP** |
| 23 | TCP | Telnet(평문 전송, 사용 지양) |
| 25 | TCP | SMTP(메일 발송) |
| **53** | TCP/UDP | **DNS** |
| 67 / 68 | UDP | DHCP 서버 / 클라이언트 |
| **80** | TCP | **HTTP** |
| 110 | TCP | POP3 |
| 143 | TCP | IMAP |
| **443** | TCP | **HTTPS** |
| 445 | TCP | SMB(Samba) |
| 2049 | TCP/UDP | NFS |
| 3306 | TCP | MySQL·MariaDB |
| 5432 | TCP | PostgreSQL |

이 대응 관계는 `/etc/services` 파일에 정의되어 있다.

> **1023번 이하 포트에 root 권한이 필요한 이유**
> 표준 서비스 포트를 임의의 사용자가 점유할 수 있다면, 악의적인 사용자가 가짜 서비스를 구동하여 다른 사용자의 자격 증명을 탈취할 수 있다. 이러한 위험을 방지하기 위해 잘 알려진 포트의 바인딩은 관리자 권한으로 제한되어 있다.
{: .prompt-tip }

---

## 1.3 네트워크 관련 설정 파일

| 파일 | 역할 |
|---|---|
| **`/etc/netplan/*.yaml`** | **우분투의 네트워크 설정**(IP·게이트웨이·DNS) |
| **`/etc/hosts`** | 로컬 이름 해석 표. **DNS보다 먼저 참조된다** |
| `/etc/resolv.conf` | DNS 서버 목록(우분투에서는 자동 관리) |
| `/etc/nsswitch.conf` | 이름 해석 순서 정의 |
| `/etc/services` | 포트와 서비스 이름의 대응표 |
| `/etc/hostname` | 호스트 이름 |

---
---

# 제2절. 네트워크 상태 진단과 설정

---

## 2.1 구형 명령과 신형 명령

Ubuntu 24.04에는 `net-tools` 패키지의 구형 명령이 기본 설치되어 있지 않다. 신형 `iproute2` 계열 명령을 표준으로 익혀야 한다.

| 구형(net-tools) | **신형(iproute2)** | 용도 |
|---|---|---|
| `ifconfig` | **`ip addr`** | 인터페이스와 IP 주소 |
| `ifconfig eth0 up/down` | **`ip link set eth0 up/down`** | 인터페이스 활성화·비활성화 |
| `route -n` | **`ip route`** | 라우팅 테이블 |
| `netstat -tulnp` | **`ss -tulnp`** | 소켓·포트 상태 |
| `arp -a` | **`ip neigh`** | ARP 캐시 |

구형 명령은 `sudo apt install net-tools`로 설치할 수 있다. 시험에는 양쪽 모두 출제되므로 대응 관계를 익혀 두어야 한다.

---

## 2.2 주요 진단 명령

| 명령 | 기능 |
|---|---|
| `ip addr` (또는 `ip a`) | 인터페이스별 IP 주소 조회 |
| `ip -br addr` | 간략 형식(**br**ief) |
| `ip route` | 라우팅 테이블과 기본 게이트웨이 |
| `ping -c 4 대상` | 도달 가능성 확인 |
| `traceroute 대상` | 경로 추적(별도 설치 필요) |
| **`ss -tulnp`** | **개방된 포트 조회** |
| `dig 도메인` | DNS 조회(상세, 별도 설치 필요) |
| `nslookup 도메인` | DNS 조회(간략, 별도 설치 필요) |
| `resolvectl query 도메인` | DNS 조회(기본 설치됨) |
| `curl -I URL` | HTTP 응답 헤더 확인 |
| `hostnamectl set-hostname 이름` | 호스트 이름 변경 |

> **`dig`·`host`·`nslookup`·`traceroute`는 기본 설치되어 있지 않다**
> Ubuntu Server 24.04의 기본 설치본에는 이 명령들이 포함되어 있지 않으므로, 그대로 실행하면 `command not found`가 출력된다. 다음 명령으로 먼저 설치한다.
>
> ```bash
> sudo apt install -y bind9-dnsutils traceroute
> ```
>
> `bind9-dnsutils` 패키지가 `dig`·`host`·`nslookup` 세 명령을 함께 제공한다(구 명칭은 `dnsutils`이다).
>
> 설치가 곤란한 환경에서는 기본 포함된 **`resolvectl query 도메인`** 또는 **`getent hosts 도메인`** 으로 대체할 수 있다.
{: .prompt-warning }

`ss` 옵션의 의미는 다음과 같다.

```
 ss -tulnp
     │││││
     ││││└─ p : process   어떤 프로그램인지 표시
     │││└── n : numeric   이름 대신 번호로 표시
     ││└─── l : listen    대기 중인 소켓만
     │└──── u : UDP
     └───── t : TCP
```

---

## 2.3 단계적 진단 절차

네트워크 장애의 원인을 파악하는 표준 절차는 다음과 같다. **이 순서를 숙지하면 어떠한 장애도 원인 구간을 좁혀 나갈 수 있다.**

```
 ① ping 127.0.0.1        실패 → TCP/IP 스택 자체의 문제
          ↓ 성공
 ② ip addr               IP 없음 → 주소 설정 문제
          ↓ 주소 존재
 ③ ping 게이트웨이         실패 → 케이블·스위치·로컬 네트워크 문제
          ↓ 성공
 ④ ping 8.8.8.8          실패 → 라우팅·외부 회선 문제
          ↓ 성공
 ⑤ ping google.com       실패 → DNS 문제
```

> **가장 흔한 증상의 해석**
> "IP 주소로는 통신이 되는데 도메인 이름으로만 실패한다"는 증상은 **DNS 문제**이다. 이름을 IP로 변환하는 단계에서만 실패하고 있기 때문이다.
{: .prompt-tip }

---

## 2.4 netplan을 이용한 네트워크 설정

우분투는 **netplan**이라는 추상화 계층을 통해 네트워크를 설정한다. 관리자가 YAML 형식의 설정 파일을 작성하면, netplan이 이를 백엔드(서버는 `systemd-networkd`)용 설정으로 변환한다.

본 과정의 실습 환경(어댑터 두 개)에 적용된 설정은 다음과 같은 형태이다.

```yaml
network:
  version: 2
  ethernets:
    enp0s3:                              # 어댑터 1 : NAT
      dhcp4: true                        #   주소를 자동으로 받는다
    enp0s8:                              # 어댑터 2 : 호스트 전용
      dhcp4: false
      addresses: [192.168.56.30/24]      #   고정 주소만 지정
```

기본 게이트웨이와 DNS 서버까지 직접 지정해야 하는 경우에는 다음과 같이 작성한다. 다만 본 과정의 환경에서는 어댑터 1이 자동으로 처리하므로 별도로 지정하지 않는다.

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.56.30/24
      routes:
        - to: default
          via: 192.168.56.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

| 명령 | 기능 |
|---|---|
| `sudo netplan get` | 현재 설정을 정규화하여 출력 |
| `sudo netplan generate` | 문법 검사 및 백엔드 설정 생성(**적용하지 않음**) |
| **`sudo netplan try`** | **적용 후 120초 내에 확인이 없으면 자동 복원** |
| `sudo netplan apply` | 즉시 적용 |

> **원격 작업 시 반드시 준수하여야 할 사항**
> SSH로 접속한 상태에서 `netplan apply`로 IP 주소를 변경하면 **즉시 접속이 단절되어 복구가 불가능해진다.** 서버가 원격지에 있다면 심각한 문제가 된다.
>
> `netplan try`는 적용 후 120초 동안 확인 입력을 대기하며, 접속이 단절되어 확인이 이루어지지 않으면 **자동으로 이전 설정으로 복원한다.** 원격 작업에서는 반드시 이 명령을 사용하여야 한다.
>
> 또한 YAML 형식은 **탭 문자를 허용하지 않으며 공백 들여쓰기만 인정한다.** Ubuntu 24.04에서는 설정 파일의 권한이 `600`이 아니면 경고를 출력한다.
{: .prompt-danger }

---

> ### 따라 하기 6-1. 네트워크 상태 진단
>
> **목적** 5단계 진단 절차를 수행하고, 개방된 포트를 조회한다.
{: .prompt-tip }

**1단계.** 실습 디렉터리를 생성한다.

```bash
mkdir -p ~/lab06 && cd ~/lab06
```

**2단계.** 인터페이스와 IP 주소를 확인한다.

```bash
ip -br link
```

```bash
ip -br addr
```

> `lo`는 루프백이고, `ens33`·`enp0s3` 등이 실제 네트워크 인터페이스이다.
>
> 인터페이스 이름이 `eth0`이 아닌 이유는 systemd의 **예측 가능한 인터페이스 이름(Predictable Network Interface Names)** 규칙 때문이다. 카드를 여러 개 장착했을 때 부팅 순서에 따라 이름이 바뀌는 문제를 방지하기 위한 것이다.

```bash
ip addr show
```

**3단계.** 라우팅 테이블과 기본 게이트웨이를 확인한다.

```bash
ip route
```

> `default via 10.0.2.2 dev enp0s3` 형태의 행이 기본 게이트웨이이다. 본 과정의 환경에서는 인터넷 경로를 어댑터 1(NAT)이 담당하므로 `10.0.2.2`가 표시된다. 호스트 전용 어댑터에는 게이트웨이가 지정되어 있지 않다.

```bash
ip route get 8.8.8.8
```

> 특정 목적지로 갈 때 실제로 선택되는 경로를 표시한다.

**4단계.** 5단계 진단 절차를 순서대로 수행한다.

```bash
ping -c 3 127.0.0.1
```

> ① 루프백. 실패하면 TCP/IP 스택 자체의 문제이다.

```bash
ping -c 3 $(ip route | awk '/default/ {print $3}')
```

> ② 게이트웨이. 실패하면 로컬 네트워크 문제이다.

```bash
ping -c 3 8.8.8.8
```

> ③ 외부 IP. 실패하면 라우팅 또는 회선 문제이다.

```bash
ping -c 3 www.google.com
```

> ④ 도메인. 여기서만 실패하면 DNS 문제이다.

**5단계.** DNS 설정과 조회를 확인한다.

```bash
resolvectl status | head -20
```

```bash
ls -l /etc/resolv.conf
```

> 이 파일이 다른 위치로 연결된 심볼릭 링크임을 확인한다. **직접 수정하여도 재부팅 시 덮어써지므로**, DNS는 netplan에서 지정하여야 한다.

```bash
sudo apt install -y bind9-dnsutils
```

> `dig`·`host`·`nslookup`은 기본 설치되어 있지 않으므로 먼저 설치한다. 이미 설치되어 있으면 그대로 진행된다.

```bash
dig www.example.com +short
```

```bash
host www.example.com
```

```bash
resolvectl query www.example.com
```

> 마지막 명령은 별도 설치 없이 사용할 수 있는 대체 수단이다. 어떤 DNS 서버가 응답하였는지까지 함께 표시한다.

**6단계.** `/etc/hosts`가 DNS보다 먼저 참조됨을 확인한다.

```bash
cat /etc/hosts
```

```bash
grep hosts /etc/nsswitch.conf
```

> `hosts: files dns` 형태로 `files`(즉 `/etc/hosts`)가 `dns`보다 앞에 있음을 확인한다.

```bash
echo "127.0.0.1 mytest.local" | sudo tee -a /etc/hosts
```

```bash
getent hosts mytest.local
```

> DNS에 존재하지 않는 이름임에도 `127.0.0.1`로 해석된다. `/etc/hosts`가 DNS보다 먼저 참조되었기 때문이다.

```bash
ping -c 1 mytest.local
```

> 루프백 주소로 응답이 온다. 여기서 확인할 것은 **응답 자체가 아니라, 등록하지 않은 이름이 지정한 주소로 변환되었다는 사실**이다.
>
> 외부의 실제 IP 주소를 기재하는 방식으로 실습하지 않는 이유는, 대부분의 외부 서버가 `ping`(ICMP)에 응답하지 않도록 설정되어 있어 이름 해석은 성공하여도 응답이 오지 않기 때문이다.

```bash
sudo sed -i '/mytest.local/d' /etc/hosts
```

> 이 특성은 개발·시험 환경에서 유용하나, 악성 코드가 이 파일을 조작하여 정상 사이트를 위조 사이트로 연결하는 공격에도 이용된다.

**7단계.** 개방된 포트를 조회한다.

```bash
ss -tuln
```

```bash
sudo ss -tulnp
```

> `-p`를 붙이면 각 포트를 점유한 프로그램이 표시되며, 이를 위해서는 관리자 권한이 필요하다.

```bash
sudo ss -tulnp | grep ":22"
```

> SSH가 22번 포트에서 대기 중임을 확인한다.

```bash
grep -E "^(ssh|http|https|domain)\s" /etc/services
```

> **확인 사항** 5단계 진단 절차를 수행하였고, `ss`로 SSH가 22번 포트에서 대기 중임을 확인하였다면 성공이다.
{: .prompt-tip }

---

> ### 따라 하기 6-2. netplan 설정 확인
>
> **목적** netplan 설정 파일의 구조를 확인하고, 문법 검사와 안전한 적용 절차를 익힌다.
>
> **주의**: 원격 접속만으로 작업하는 경우 6단계까지만 수행하고, 콘솔 접근이 가능한 경우에만 7단계 이후를 수행한다.
{: .prompt-tip }

**1단계.** 현재 설정을 확인한다.

```bash
ls -l /etc/netplan/
```

```bash
sudo cat /etc/netplan/*.yaml
```

```bash
sudo netplan get
```

**2단계.** 백업본을 생성한다.

```bash
sudo cp -a /etc/netplan /root/netplan.backup
```

```bash
sudo ls /root/netplan.backup
```

**3단계.** 인터페이스 이름과 현재 설정을 기록한다.

```bash
IF_NAT=$(ls /sys/class/net | grep -v lo | sed -n '1p')
```

```bash
IF_LAB=$(ls /sys/class/net | grep -v lo | sed -n '2p')
```

```bash
echo "어댑터1(NAT): $IF_NAT / 어댑터2(호스트 전용): $IF_LAB"
```

```bash
ip -br addr
```

```bash
ip route | grep default
```

> **현재 값을 별도로 기록해 둔다.** 문제 발생 시 복원에 필요하다.

> **본 실습에서 변경 대상을 어댑터 2로 한정하는 이유**
> 인터넷 경로를 담당하는 **어댑터 1(NAT)의 설정은 변경하지 않는다.** 이 어댑터에 고정 주소를 지정하면 인터넷 연결이 즉시 끊겨 패키지 설치가 불가능해지기 때문이다.
>
> 따라서 본 실습에서는 **어댑터 2(호스트 전용)** 의 주소를 기존 `192.168.56.30`에서 `192.168.56.31`로 변경해 보는 방식으로 진행한다. 잘못 적용되더라도 인터넷은 유지되며, 원격 접속 주소만 바뀌므로 복구가 용이하다.
{: .prompt-warning }

**4단계.** 새 설정 파일을 작성한다.

```bash
sudo tee /etc/netplan/99-lab-static.yaml > /dev/null << EOF
network:
  version: 2
  ethernets:
    ${IF_NAT}:
      dhcp4: true
    ${IF_LAB}:
      dhcp4: false
      addresses: [192.168.56.31/24]
EOF
```

```bash
sudo chmod 600 /etc/netplan/99-lab-static.yaml
```

```bash
sudo cat /etc/netplan/99-lab-static.yaml
```

> 파일명이 `99-`로 시작하는 이유는 netplan이 파일을 사전순으로 읽고 나중에 읽은 설정이 앞의 설정을 덮어쓰기 때문이다.

**5단계.** 문법을 검사한다.

```bash
sudo netplan generate
```

> 오류가 없으면 아무 메시지도 출력되지 않는다. 오류가 있으면 해당 위치를 알려 준다. **적용 없이 안전하게 검증할 수 있는 명령이다.**

```bash
sudo netplan get
```

**6단계.** 안전장치를 사용하여 적용을 시도한다.

```bash
sudo netplan try
```

> 확인 입력을 120초간 대기하며, 입력이 없으면 자동으로 이전 설정으로 복원한다. **원격 작업 시 반드시 사용하여야 할 명령이다.**
>
> 윈도우 터미널에서 `192.168.56.30`으로 접속 중이라면, 적용되는 순간 세션이 끊겨 확인 입력을 할 수 없게 된다. **이때 아무 조작도 하지 않고 120초를 기다리면 자동으로 원래 주소로 복원되어 세션이 되살아난다.** 이것이 `netplan try`의 동작을 체험하는 가장 좋은 방법이다.

**7단계.** 적용을 확정하려는 경우에만 다음을 실행한다.

```bash
sudo netplan apply
```

```bash
ip -br addr
```

```bash
ping -c 2 8.8.8.8
```

> 어댑터 2의 주소가 `192.168.56.31`로 변경되었으나, 어댑터 1은 그대로이므로 **인터넷 연결은 유지된다.**
>
> 윈도우 터미널에서 다시 접속하려면 주소를 `192.168.56.31`로 지정하여야 한다.

**8단계.** 원래 설정으로 복원한다.

```bash
sudo rm /etc/netplan/99-lab-static.yaml
```

```bash
sudo netplan apply
```

```bash
ip -br addr
```

> 어댑터 2의 주소가 `192.168.56.30`으로 되돌아왔는지 확인한다.
>
> **`.31` 주소가 함께 남아 있는 경우가 있다.** `netplan apply`는 새 설정을 적용할 뿐, 이전에 부여된 주소를 항상 회수하지는 않기 때문이다. 이때는 다음 명령으로 직접 제거한다.

```bash
sudo ip addr del 192.168.56.31/24 dev $IF_LAB
```

```bash
ip -br addr
```

> 그래도 정리되지 않으면 `sudo reboot`으로 재부팅한다. 설정 파일을 삭제하였으므로 재부팅 후에는 `192.168.56.30`만 남는다.
>
> 이는 **"설정 파일의 내용"과 "현재 인터페이스의 상태"가 서로 다른 대상**이라는 점을 보여 주는 사례이다. 설정을 되돌렸다고 하여 실행 중인 상태가 자동으로 함께 되돌아가는 것은 아니다.

> **확인 사항** `netplan generate`가 오류 없이 통과하고, `netplan try`가 확인 대기 상태로 진입하며, 복원 후 주소가 `192.168.56.30`으로 되돌아왔다면 성공이다.
{: .prompt-tip }

---
---

# 제3절. 방화벽과 원격 접속

---

## 3.1 방화벽 `ufw`

`ufw`는 **U**ncomplicated **F**ire**w**all의 약어이다. 리눅스 커널의 `nftables`/`iptables`를 간결한 명령으로 조작할 수 있게 하는 도구이다.

| 명령 | 기능 |
|---|---|
| `sudo ufw status verbose` | 상태 조회 |
| `sudo ufw default deny incoming` | 들어오는 연결 기본 차단 |
| `sudo ufw default allow outgoing` | 나가는 연결 기본 허용 |
| `sudo ufw allow 22/tcp` | 포트 개방 |
| `sudo ufw allow OpenSSH` | 애플리케이션 프로파일로 개방 |
| `sudo ufw allow from 192.168.56.0/24 to any port 3306` | 출발지 제한 개방 |
| `sudo ufw delete allow 80/tcp` | 규칙 삭제 |
| `sudo ufw enable` / `disable` | 활성화 / 비활성화 |
| `sudo ufw status numbered` | 번호가 부여된 규칙 목록 |

> **방화벽 설정 시 반드시 준수하여야 할 순서**
>
> **잘못된 순서:**
> ```
> sudo ufw default deny incoming
> sudo ufw enable                 ← 이 시점에 SSH 접속이 단절된다
> ```
>
> **올바른 순서:**
> ```
> sudo ufw allow OpenSSH          ← ① SSH를 먼저 허용
> sudo ufw default deny incoming  ← ② 기본 정책 설정
> sudo ufw enable                 ← ③ 활성화
> ```
>
> **SSH 허용 → 기본 정책 → 활성화**의 순서를 반드시 준수하여야 한다. 실무에서 매우 빈번하게 발생하는 사고이다.
{: .prompt-danger }

---

## 3.2 SSH — 원격 접속의 표준

**SSH(Secure Shell)** 는 통신 구간을 암호화하는 원격 접속 프로토콜이다. 과거에 사용되던 Telnet은 비밀번호를 포함한 모든 내용을 평문으로 전송하므로 현재는 사용하지 않는다.

| 명령 | 기능 |
|---|---|
| `ssh 사용자@주소` | 원격 접속 |
| `ssh -p 2222 사용자@주소` | 포트 지정 접속 |
| `ssh -X 사용자@주소` | X11 포워딩 |
| `scp 파일 사용자@주소:/경로` | 파일 전송 |
| `scp -r 디렉터리 사용자@주소:/경로` | 디렉터리 전송 |
| **`rsync -avz 원본 사용자@주소:/경로`** | **변경분만 전송(효율적)** |
| `ssh-keygen -t ed25519` | 키 쌍 생성 |
| `ssh-copy-id 사용자@주소` | 공개 키 등록 |

---

## 3.3 키 기반 인증

SSH는 비밀번호 대신 **키 쌍**을 이용한 인증을 지원하며, 이 방식이 보안상 더 안전하다.

```
 [클라이언트]                          [서버]
  개인 키 (id_ed25519)                 공개 키 (authorized_keys)
  ※ 절대 유출 금지                     ※ 서버에 등록

        │ ─────── 접속 요청 ──────────→ │
        │ ←────── 검증 요청 ─────────── │
        │ ─────── 개인 키로 증명 ──────→ │  인증 완료
```

| 파일 | 성격 | 필수 권한 |
|---|---|---|
| `~/.ssh/id_ed25519` | **개인 키**(유출 금지) | **600** |
| `~/.ssh/id_ed25519.pub` | 공개 키(서버에 등록) | 644 |
| `~/.ssh/authorized_keys` | 서버에 등록된 공개 키 목록 | **600** |
| `~/.ssh` | 디렉터리 | **700** |

> **키를 등록하였음에도 비밀번호를 계속 요구하는 경우**
> 원인의 대부분은 **권한 설정 문제**이다. SSH는 보안상 `~/.ssh` 디렉터리 또는 `authorized_keys` 파일의 권한이 느슨하면 해당 키를 **무시한다.**
>
> ```
> chmod 700 ~/.ssh
> chmod 600 ~/.ssh/authorized_keys
> ```
{: .prompt-danger }

---

## 3.4 SSH 서버 보안 설정

설정 파일은 `/etc/ssh/sshd_config`이다.

| 항목 | 권장 설정 | 근거 |
|---|---|---|
| `PermitRootLogin` | `no` | root 직접 로그인 차단 |
| `PasswordAuthentication` | `no`(키 인증 구성 후) | 무차별 대입 공격 차단 |
| `Port` | 기본 22에서 변경 고려 | 자동화 스캔 회피(보조 수단) |
| `AllowUsers` | 필요한 계정만 나열 | 접근 주체 최소화 |

> **설정 변경 시 준수 사항**
> `PasswordAuthentication no`로 변경하기 전에 **반드시 별도의 세션에서 키 인증이 정상 동작하는지 확인**하여야 한다. 확인 없이 적용하고 세션을 종료하면 접속 수단을 완전히 상실한다.
>
> 또한 설정 변경 후 재시작 전에 반드시 문법을 검사한다.
> ```
> sudo sshd -t
> ```
{: .prompt-warning }

---

> ### 따라 하기 6-3. 방화벽 구성
>
> **목적** 올바른 순서로 방화벽을 활성화하고 필요한 포트만 개방한다.
{: .prompt-tip }

**1단계.** 현재 상태를 확인한다.

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

```bash
sudo ufw status
```

> **이 단계를 생략하고 방화벽을 활성화하면 원격 접속이 단절된다.**

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

> 접속이 유지되고 있음을 확인한다.

**5단계.** 웹 서버를 설치하여 차단 상태를 확인한다.

```bash
sudo apt update && sudo apt install -y nginx
```

```bash
systemctl is-active nginx
```

```bash
sudo ss -tlnp | grep :80
```

```bash
curl -I http://localhost
```

> 로컬에서는 접속되나 외부에서는 방화벽에 차단된다. **서비스가 동작하는 것과 외부에서 접근 가능한 것은 별개의 문제이다.**

**6단계.** HTTP 포트를 개방한다.

```bash
sudo ufw allow 80/tcp
```

```bash
sudo ufw status numbered
```

**7단계.** 출발지를 제한한 규칙을 추가한다.

```bash
sudo ufw allow from 192.168.56.0/24 to any port 3306 proto tcp
```

```bash
sudo ufw status numbered
```

> 데이터베이스 포트를 전체 대역에 개방하는 것은 매우 위험하므로, 이와 같이 출발지를 제한하는 것이 원칙이다.

**8단계.** 규칙을 삭제하고 정리한다.

```bash
sudo ufw delete allow from 192.168.56.0/24 to any port 3306 proto tcp
```

```bash
sudo ufw delete allow 80/tcp
```

```bash
sudo apt purge -y nginx nginx-common && sudo apt autoremove -y
```

```bash
sudo ufw status
```

> **확인 사항** SSH 허용 → 기본 정책 → 활성화 순서를 준수하였고, 포트 개방 전후로 접근 가능 여부가 달라지는 것을 확인하였다면 성공이다.
{: .prompt-tip }

---

> ### 따라 하기 6-4. SSH 키 인증과 파일 전송
>
> **목적** 키 쌍을 생성하여 비밀번호 없는 접속을 구성하고, `scp`와 `rsync`의 차이를 확인한다.
>
> 별도의 서버가 없어도 자기 자신에게 접속하는 방식으로 실습할 수 있다.
{: .prompt-tip }

**1단계.** SSH 서비스 상태를 확인한다.

```bash
systemctl is-active ssh
```

```bash
systemctl is-active ssh.socket
```

> 현재 윈도우 터미널에서 SSH로 접속한 상태라면 두 명령 모두 `active`이다.
>
> 다만 **한 번도 접속하지 않은 상태에서는 `ssh`가 `inactive`, `ssh.socket`만 `active`** 로 나온다. Ubuntu 22.10 이후의 OpenSSH는 소켓 활성화 방식을 사용하여, 접속이 들어오는 순간에 `ssh.service`를 기동하기 때문이다(제0강 0-3 4.2절).

```bash
sudo ss -tlnp | grep :22
```

> 22번 포트를 점유한 프로세스가 `sshd` 또는 `systemd`로 표시된다. 소켓 활성화 상태에서는 `systemd`가 대신 포트를 지키고 있다.

**2단계.** 윈도우 호스트에서 접속한다.

제0강에서 호스트 전용 어댑터를 구성하였으므로, 윈도우의 터미널에서 직접 접속할 수 있다. **윈도우 터미널 또는 PowerShell**을 열고 다음을 실행한다.

```
ssh student@192.168.56.30
```

> 이것이 실무에서 서버를 관리하는 실제 방식이다. 접속에 성공하면 `student@ubuntu-lab:~$` 프롬프트가 표시되며, 윈도우 창에서 리눅스 명령을 실행하게 된다.
>
> 접속되지 않는다면 제0강 0-3 제5절의 점검 항목을 확인한다.

**3단계.** 자기 자신에게 접속하여 동작 원리를 확인한다.

이후 단계는 별도의 장비 없이 진행할 수 있도록 **자기 자신에게 접속하는 방식**으로 수행한다. 원리는 원격 접속과 동일하다.

```bash
ssh localhost
```

> 최초 접속 시 호스트 키 지문을 확인하라는 메시지가 표시된다. `yes`를 입력하고 비밀번호로 로그인한다.
>
> 이 지문은 서버의 신원 정보에 해당하며 `~/.ssh/known_hosts`에 저장된다. 이후 이 값이 변경되면 중간자 공격 가능성을 경고한다.

```bash
who
```

```bash
exit
```

**4단계.** 키 쌍을 생성한다.

```bash
ssh-keygen -t ed25519 -C "linux-lab-$(whoami)" -f ~/.ssh/id_ed25519 -N ""
```

> `-t ed25519`는 현재 권장되는 암호 알고리즘이며, `-N ""`은 키 비밀번호를 설정하지 않는다는 의미이다.
>
> **실무에서는 반드시 키 비밀번호(passphrase)를 설정하고 `ssh-agent`로 관리하여야 한다.** 본 실습에서는 편의를 위해 생략한다.

```bash
ls -l ~/.ssh/
```

```bash
cat ~/.ssh/id_ed25519.pub
```

**5단계.** 공개 키를 등록한다.

```bash
ssh-copy-id localhost
```

```bash
ls -ld ~/.ssh && ls -l ~/.ssh/authorized_keys
```

> 권한이 `700`, `600`인지 확인한다. 이 값이 다르면 SSH가 키를 무시한다.

**6단계.** 비밀번호 없이 접속되는지 확인한다.

```bash
ssh localhost "hostname; whoami; date"
```

> 비밀번호를 요구하지 않으며, 접속과 동시에 명령을 실행하여 결과만 반환한다. 이 형태가 다수의 서버를 자동으로 관리하는 스크립트의 기본 구조이다.

**7단계.** 파일을 전송한다.

```bash
echo "SCP 전송 시험 파일" > ~/lab06/transfer.txt
```

```bash
mkdir -p ~/lab06/received
```

```bash
scp ~/lab06/transfer.txt localhost:~/lab06/received/
```

```bash
ls -l ~/lab06/received/
```

**8단계.** `rsync`로 변경분만 전송한다.

```bash
rsync -avz ~/lab06/ localhost:/tmp/lab06_sync/
```

> 전체가 전송된다.

```bash
echo "추가한 내용" >> ~/lab06/transfer.txt
```

```bash
rsync -avz ~/lab06/ localhost:/tmp/lab06_sync/
```

> **변경된 파일만 전송된다.** `scp`가 매번 전체를 복사하는 것과 달리 `rsync`는 차이만 전송하므로 백업과 배포에 훨씬 효율적이다.

**9단계.** 서버 보안 설정을 확인한다.

```bash
grep -iE "PermitRootLogin|PasswordAuthentication|Port " /etc/ssh/sshd_config
```

```bash
sudo sshd -t && echo "설정 문법에 이상이 없습니다"
```

**10단계.** 정리한다.

```bash
rm -rf /tmp/lab06_sync ~/lab06/received
```

> **확인 사항** 키 등록 후 비밀번호 없이 접속되고, 두 번째 `rsync`가 변경분만 전송하는 것을 확인하였다면 성공이다.
{: .prompt-tip }

---
---

# 제4절. X 윈도와 리눅스의 활용 분야

---

## 4.1 X 윈도의 구조

리눅스의 그래픽 환경은 커널에 내장되어 있지 않고 **응용 프로그램 계층에서 동작**한다. 이 점이 윈도우 운영체제와의 결정적 차이이다. 그래픽 하위 시스템을 제거하여도 운영체제는 완전하게 동작하며, 이것이 서버에서 그래픽 환경을 배제할 수 있는 근거가 된다.

X 윈도 시스템은 **네트워크 투명성**을 전제로 설계된 클라이언트-서버 구조를 갖는다.

| 구성 요소 | 역할 | 위치 |
|---|---|---|
| **X 서버** | **화면·키보드·마우스 등 입출력 장치를 관리** | **사용자 앞의 컴퓨터** |
| **X 클라이언트** | 실제 응용 프로그램(브라우저, 편집기 등) | 원격 호스트일 수 있음 |
| X 프로토콜 | 서버와 클라이언트 간 통신 규약 | — |
| 윈도 매니저 | 창의 테두리·이동·크기 조절 담당 | — |
| 디스플레이 매니저 | 그래픽 로그인 화면 제공(GDM 등) | — |
| 데스크톱 환경 | 윈도 매니저 + 파일 관리자 + 응용 프로그램 묶음 | — |

> **서버와 클라이언트의 위치가 통념과 반대인 이유**
> 일반적으로 '서버'는 원격지의 대형 컴퓨터를 연상시키지만, X 윈도에서는 **사용자 앞의 화면이 X 서버**이고 **프로그램이 X 클라이언트**이다.
>
> X 서버가 "화면이라는 자원을 제공하는 측"이고, 프로그램은 "그 자원의 사용을 요청하는 측"이기 때문이다. 이 개념은 시험에 반복 출제된다.
{: .prompt-danger }

---

## 4.2 데스크톱 환경과 툴킷

| 데스크톱 환경 | 사용 툴킷 | 특징 |
|---|---|---|
| **GNOME** | **GTK** | 우분투 데스크톱 기본값 |
| **KDE Plasma** | **Qt** | 사용자 정의 자유도가 높음 |
| XFCE | GTK | 경량 |
| LXQt | Qt | 초경량 |

> **시험 고정 출제 항목**
> **KDE는 Qt, GNOME은 GTK**를 사용한다. 이 대응 관계는 반드시 암기하여야 한다.
{: .prompt-warning }

---

## 4.3 X 윈도 관련 명령과 환경 변수

| 항목 | 설명 |
|---|---|
| **`DISPLAY`** | 출력할 X 서버를 지정하는 환경 변수(형식: `호스트:디스플레이번호.화면번호`) |
| `startx` | X 세션 시작 |
| `xhost +호스트` | 지정 호스트의 접속 허용(`xhost +`는 전체 허용으로 **매우 위험**) |
| `Ctrl + Alt + Backspace` | X 서버 강제 종료 |
| `Ctrl + Alt + F1~F6` | 가상 콘솔 전환 |
| `ssh -X 호스트` | X11 포워딩으로 원격 GUI 프로그램을 로컬 화면에 표시 |

---

## 4.4 서버에서 그래픽 환경을 배제하는 이유

| 관점 | 근거 |
|---|---|
| 자원 효율 | 그래픽 환경은 수백 MB의 메모리와 상시 CPU를 소비한다. 해당 자원을 서비스에 할당하는 것이 합리적이다. |
| 공격 표면 | 그래픽 스택·디스플레이 매니저·데스크톱 응용 프로그램까지 취약점 관리 대상이 확대된다. |
| 자동화 | 명령행은 스크립트로 기록·재현·배포가 가능하나, 그래픽 조작은 재현이 어렵다. |
| 원격 운영 | SSH는 저대역폭 환경에서도 안정적이나 원격 데스크톱은 대역폭 요구가 크다. |
| 표준화 | 다수의 서버를 동일한 스크립트로 관리하려면 명령행이 전제가 된다. |

---

## 4.5 리눅스의 활용 분야와 기술 동향

| 분야 | 내용 |
|---|---|
| 서버·클라우드 | 웹·데이터베이스·메일 서버. 클라우드 인스턴스의 대다수가 리눅스이다. |
| **컨테이너** | Docker, Podman, Kubernetes |
| 모바일 | **안드로이드**는 리눅스 커널 위에서 동작한다. |
| 임베디드·IoT | 라즈베리파이, 공유기, 차량 인포테인먼트, 산업 제어 |
| 고성능 컴퓨팅 | 세계 슈퍼컴퓨터 상위 500대 전부가 리눅스를 사용한다. |
| 인공지능 | 학습·추론 프레임워크의 표준 실행 환경 |
| 보안 | 모의 침투 도구, 보안 관제 인프라 |

---

## 4.6 컨테이너 기술의 기반

컨테이너는 응용 프로그램을 실행 환경과 함께 포장하여 어느 환경에서든 동일하게 동작하도록 하는 기술이다. 이 기술은 **리눅스 커널의 기능**을 기반으로 구현되어 있다.

| 커널 기능 | 역할 |
|---|---|
| **namespace** | 프로세스·네트워크·파일 시스템 등을 격리하여 독립된 환경처럼 보이게 한다. |
| **cgroups** | CPU·메모리 등 자원 사용량을 제한한다. |
| overlayfs | 이미지 계층을 겹쳐 하나의 파일 시스템으로 제공한다. |

가상 머신과의 차이는 다음과 같다.

| 구분 | 가상 머신(VM) | 컨테이너 |
|---|---|---|
| 방식 | 하드웨어를 가상화하고 게스트 OS를 구동 | **호스트 커널을 공유** |
| 크기 | GB 단위 | MB 단위 |
| 기동 시간 | 수십 초 | 수 초 이내 |
| 격리 수준 | 높음 | 상대적으로 낮음 |

---

> ### 따라 하기 6-5. 그래픽 환경 확인
>
> **목적** 서버 환경에 그래픽 시스템이 없어도 정상 동작함을 확인하고, X 서버와 클라이언트의 위치 관계를 점검한다.
{: .prompt-tip }

**1단계.** 그래픽 환경의 유무를 확인한다.

```bash
systemctl get-default
```

> `multi-user.target`이 출력되면 텍스트 모드이다.

```bash
echo "DISPLAY=[$DISPLAY]"
```

> 값이 비어 있다. 출력할 화면이 지정되어 있지 않다는 의미이다.

```bash
which startx Xorg 2>/dev/null || echo "X 서버가 설치되어 있지 않습니다"
```

> 그래픽 시스템이 전혀 없음에도 서버는 완전하게 동작하고 있음을 확인한다.

**2단계.** 그래픽 환경 설치 시의 자원 소요를 확인한다.

```bash
apt show ubuntu-desktop 2>/dev/null | grep -iE "^(Package|Installed-Size)" | head -2
```

```bash
free -h
```

> **실습 목적으로 `ubuntu-desktop`을 실제로 설치하지 않는다.** 수 GB의 저장 공간을 소요하며 시스템의 기본 타깃까지 변경된다.

**3단계.** X11 포워딩 설정을 확인한다.

```bash
grep -i "X11Forwarding" /etc/ssh/sshd_config
```

```bash
ssh -X localhost 'echo "포워딩된 DISPLAY=[$DISPLAY]"'
```

> **본 실습 환경에서는 다음과 같이 값이 비어 있게 출력되는 것이 정상이다.**
>
> ```
> 포워딩된 DISPLAY=[]
> ```
>
> 이유는 1단계에서 확인한 그대로이다. **포워딩할 X 서버가 어디에도 없기 때문이다.** 현재 세션은 윈도우 터미널에서 접속한 문자 기반 세션이므로 `DISPLAY`가 비어 있고, 전달할 화면이 없으므로 `ssh -X`는 포워딩을 수행하지 않는다. 서버에 `xauth`가 설치되어 있지 않으면 `X11 forwarding request failed`라는 경고가 함께 표시되기도 한다.
>
> **값이 표시되게 하려면** 접속하는 쪽(윈도우)에 X 서버 프로그램(VcXsrv, Xming 등)을 설치하여 실행하고, X 서버를 지원하는 SSH 클라이언트로 접속하여야 한다. 그 경우 `localhost:10.0`과 같은 값이 표시된다.
>
> 이 결과가 곧 본 절의 핵심을 확인시켜 준다. **화면을 담당하는 X 서버는 사용자 측에 있어야 하며**, 프로그램이 실행되는 원격 측은 X 클라이언트에 불과하다. 사용자 측에 X 서버가 없으면 원격에 아무리 프로그램이 있어도 표시할 수 없다.

**4단계.** 가상 콘솔을 확인한다.

```bash
who
```

> `tty1`은 물리 콘솔, `pts/0`은 원격 접속 세션을 의미한다.

```bash
ls /dev/tty[1-6]
```

```bash
systemctl status getty@tty1 --no-pager | head -5
```

> **확인 사항** 서버에 X 서버가 없음을 확인하고, X 서버와 X 클라이언트의 위치 관계를 설명할 수 있다면 성공이다.
{: .prompt-tip }

---
---

# 제5절. 종합 실습 — 웹 서비스 구축과 보안 점검

---

> **실습 시나리오**
>
> 학습자는 서버 운영 담당자로서 다음 업무를 지시받았다.
>
> *"부서 안내 페이지를 서비스할 웹 서버를 구축하시오. 외부에서 접속이 가능하도록 방화벽을 구성하되, 불필요한 포트는 개방하지 않아야 한다. 구축 완료 후 점검 도구를 작성하여 제출하시오."*
>
> 본 실습은 제1강부터 제6강까지 학습한 내용을 종합적으로 적용하는 최종 과제이다.
{: .prompt-info }

---

## 단계 1. 작업 환경 준비 및 현황 파악

```bash
mkdir -p ~/mission06 && cd ~/mission06
```

```bash
ip -br addr | grep -v "^lo"
```

> 서버의 IP 주소를 확인하여 기록한다.

```bash
sudo ss -tuln
```

> 구축 전 개방된 포트 현황을 기록한다.

---

## 단계 2. 웹 서버 설치 (제5강 적용)

```bash
sudo apt update
```

```bash
sudo apt install -y nginx
```

```bash
systemctl is-active nginx
```

```bash
systemctl is-enabled nginx
```

> **`enabled` 상태인지 확인한다.** 재부팅 후에도 자동으로 기동되어야 한다.

---

## 단계 3. 서비스 실행 계정 확인 (제3강 적용)

```bash
ps -eo user,pid,comm | grep nginx
```

> 다음 사항을 확인한다.
> - master 프로세스는 **root**로 실행된다. 1023번 이하인 80번 포트를 바인딩하려면 관리자 권한이 필요하기 때문이다.
> - worker 프로세스는 **www-data**로 실행된다. 실제 요청 처리는 낮은 권한으로 수행한다.

```bash
grep "^www-data" /etc/passwd
```

> 로그인 셸이 `nologin`으로 설정되어 있음을 확인한다. 서비스 전용 계정은 대화형 로그인이 불필요하기 때문이다.

> **최소 권한 원칙의 적용 사례**
> 이러한 구조에서는 웹 서버가 침해되더라도 공격자가 획득하는 권한은 `www-data`에 한정된다. 관리자 권한으로의 확대를 차단하는 설계이다.
{: .prompt-tip }

---

## 단계 4. 콘텐츠 배치 (제2강·제4강 적용)

```bash
sudo tee /var/www/html/index.html > /dev/null << 'EOF'
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <title>리눅스 기초 - 실습 서버</title>
</head>
<body style="font-family: sans-serif; max-width: 640px; margin: 60px auto;">
  <h1>리눅스 기초 실습 서버</h1>
  <p>Ubuntu Server 24.04 LTS / nginx</p>
  <hr>
  <p>본 페이지는 리눅스 기초 6강 종합 실습으로 구축되었습니다.</p>
</body>
</html>
EOF
```

---

## 단계 5. 파일 권한 설정 (제3강 적용)

```bash
sudo chown root:www-data /var/www/html/index.html
```

```bash
sudo chmod 640 /var/www/html/index.html
```

```bash
ls -l /var/www/html/
```

> `640`을 적용한 근거는 다음과 같다.
>
> | 대상 | 권한 | 근거 |
> |---|---|---|
> | 소유자(root) | 6(읽기·쓰기) | 관리자가 콘텐츠를 수정 |
> | 그룹(www-data) | 4(**읽기만**) | 웹 서버는 읽기만 하면 충분 |
> | 기타 | 0(없음) | 그 외에는 접근 불필요 |
>
> 웹 서버에 쓰기 권한을 부여하면 취약점을 통한 파일 업로드·변조가 가능해진다.

---

## 단계 6. 로컬 동작 확인

```bash
curl -I http://localhost
```

```bash
curl -s http://localhost | head -8
```

---

## 단계 7. 방화벽 구성 (순서 준수)

```bash
sudo ufw allow OpenSSH
```

```bash
sudo ufw allow 'Nginx HTTP'
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

---

## 단계 8. 외부 접속 확인

```bash
hostname -I
```

> 이 서버가 보유한 주소가 **모두** 출력된다. `10.0.2.15`(어댑터 1, NAT)와 `192.168.56.30`(어댑터 2, 호스트 전용) 두 개가 표시된다.

```bash
hostname -I | tr ' ' '\n' | grep "^192.168.56."
```

> 제4강에서 학습한 `tr`(문자 치환)과 `grep`을 조합하여, 여러 주소 중 **외부에서 접근할 수 있는 주소만** 골라낸다.

```bash
echo "브라우저에서 http://192.168.56.30/ 로 접속하십시오"
```

> 윈도우 호스트의 브라우저에서 위 주소(`http://192.168.56.30/`)로 접속하여 페이지가 표시되는지 확인한다.
>
> **주의 — `hostname -I`의 첫 번째 주소를 사용해서는 안 된다.** 이 서버에는 어댑터가 두 개 있으며, `hostname -I`는 어댑터 1(NAT)의 `10.0.2.15`를 먼저 출력한다. NAT 주소는 가상 머신 내부에서만 유효하므로 **윈도우 호스트에서는 접속되지 않는다.** 외부에서 접근할 때 사용하는 주소는 항상 호스트 전용 어댑터의 `192.168.56.30`이다.
>
> 서버가 여러 주소를 가질 때 "어느 주소로 서비스하는가"를 구분하는 것은 실무의 기본 점검 항목이다.

---

## 단계 9. 접속 기록 확인 (제2강 적용)

```bash
sudo tail -5 /var/log/nginx/access.log
```

```bash
sudo journalctl -u nginx -n 10 --no-pager
```

---

## 단계 10. 서비스 제어 실습 (제5강 적용)

```bash
sudo systemctl stop nginx
```

```bash
curl -I http://localhost 2>&1 | head -2
```

> 접속이 되지 않는다.

```bash
sudo systemctl start nginx
```

```bash
curl -I http://localhost 2>/dev/null | head -2
```

```bash
sudo nginx -t
```

> 설정 파일의 문법을 검사한다.

```bash
sudo systemctl reload nginx
```

> **`restart`와 `reload`의 차이**
> - `restart` : 프로세스를 종료 후 재기동하므로 순간적인 서비스 중단이 발생한다.
> - `reload` : 설정만 재적재하므로 **무중단**이다.
>
> 운영 중인 서버에서는 `nginx -t`로 검증한 뒤 `reload`를 사용하는 것이 원칙이다.

---

## 단계 11. 종합 점검 도구 작성

```bash
cat > ~/mission06/healthcheck.sh << 'EOF'
#!/bin/bash
# 웹 서비스 종합 점검 스크립트
# 리눅스 기초 6강 종합 실습

echo "=========================================="
echo "  서버 상태 점검 보고"
echo "  점검 일시: $(date '+%Y-%m-%d %H:%M:%S')"
echo "=========================================="
echo

echo "[1] 네트워크 구성"
ip -br addr | grep -v "^lo" | awk '{print "  인터페이스: " $1 "  주소: " $3}'
echo "  게이트웨이: $(ip route | awk '/default/ {print $3}')"
echo

echo "[2] 외부 통신 상태"
ping -c 1 -W 2 8.8.8.8 > /dev/null 2>&1 \
  && echo "  인터넷 연결: 정상" || echo "  인터넷 연결: 실패"
getent hosts www.example.com > /dev/null 2>&1 \
  && echo "  DNS 조회   : 정상" || echo "  DNS 조회   : 실패"
echo

echo "[3] 개방된 포트"
ss -tlnH | awk '{print "  " $4}' | sort -u
echo

echo "[4] 방화벽 상태"
if FW=$(sudo -n ufw status 2>/dev/null); then
  echo "$FW" | head -1 | sed 's/^/  /'
else
  echo "  (권한 없음 — 관리자 권한으로 실행하면 표시됨)"
fi
echo

echo "[5] 웹 서비스 상태"
echo "  서비스 실행 : $(systemctl is-active nginx)"
echo "  자동 시작   : $(systemctl is-enabled nginx)"
curl -s -o /dev/null -w "  HTTP 응답   : %{http_code}\n" http://localhost
echo

echo "[6] 자원 사용 현황"
echo "  부하 평균:$(uptime | awk -F'load average:' '{print $2}')"
echo "  메모리   : $(free -h | awk '/^Mem/ {print $3 " / " $2}')"
echo "  디스크   : $(df -h / | tail -1 | awk '{print $5 " 사용"}')"
echo

echo "[7] 보안 점검"
echo "  UID 0 계정  : $(awk -F: '$3==0 {print $1}' /etc/passwd | tr '\n' ' ')"
echo "  콘텐츠 권한 : $(stat -c '%A %U:%G' /var/www/html/index.html 2>/dev/null)"
EOF
```

```bash
chmod 750 ~/mission06/healthcheck.sh
```

```bash
~/mission06/healthcheck.sh
```

---

## 단계 12. 점검 자동화 등록 (제5강 적용)

```bash
mkdir -p ~/mission06/logs
```

```bash
(crontab -l 2>/dev/null; echo "0 9 * * * $HOME/mission06/healthcheck.sh >> $HOME/mission06/logs/daily.log 2>&1") | crontab -
```

```bash
crontab -l
```

> 매일 09시에 자동으로 점검을 수행하고 결과를 기록하도록 등록하였다.

> **cron으로 실행할 때 `sudo`가 동작하지 않는 이유**
> 스크립트의 `[4] 방화벽 상태` 항목은 `sudo`를 사용한다. 그런데 cron은 **터미널 없이** 명령을 실행하므로 비밀번호를 입력할 수단이 없다. 따라서 그대로 두면 로그에 `sudo: a terminal is required to read the password`가 기록된다.
>
> 본 스크립트는 `sudo -n`(**n**on-interactive, 비밀번호를 묻지 않고 즉시 실패)과 대체 메시지를 사용하여, cron에서 실행되더라도 나머지 항목이 정상적으로 기록되도록 작성하였다.
>
> 이때 결과를 **변수에 먼저 담은 뒤 `if`로 판정**한 이유가 있다. `sudo -n ufw status | head -1 || echo "..."` 형태로 작성하면 파이프라인의 성공 여부를 **마지막 명령(`head`)** 이 결정하므로, `sudo`가 실패하여도 `||` 뒤가 실행되지 않아 아무것도 출력되지 않는다. 파이프와 조건부 실행을 함께 쓸 때 자주 발생하는 오류이다.
>
> 방화벽 상태까지 자동으로 기록하려면 해당 명령에 한정하여 비밀번호 없이 실행할 수 있도록 위임 규칙을 추가한다(제3강 4.2절의 최소 권한 위임).
>
> ```bash
> echo "$(whoami) ALL=(root) NOPASSWD: /usr/sbin/ufw status" | sudo tee /etc/sudoers.d/91-ufw-status
> sudo chmod 440 /etc/sudoers.d/91-ufw-status
> sudo visudo -c
> ```
>
> 실습을 마친 뒤에는 `sudo rm -f /etc/sudoers.d/91-ufw-status`로 제거한다.
{: .prompt-warning }

---

## 단계 13. 정리

```bash
crontab -r
```

```bash
sudo ufw delete allow 'Nginx HTTP'
```

```bash
sudo apt purge -y nginx nginx-common && sudo apt autoremove -y
```

```bash
sudo rm -f /etc/sudoers.d/91-ufw-status
```

> 단계 12의 참고 사항에서 위임 규칙을 추가한 경우에만 해당한다. 파일이 없어도 오류 없이 진행된다.

```bash
sudo ufw status
```

> **종합 실습 완료 기준**
> 1. 웹 서버를 설치하고 자동 시작 등록 상태를 확인하였다.
> 2. 서비스 실행 계정의 권한 분리 구조를 확인하였다.
> 3. 콘텐츠 파일에 `640` 권한을 적용하고 그 근거를 설명할 수 있다.
> 4. SSH 허용 → 기본 정책 → 활성화 순서로 방화벽을 구성하였다.
> 5. 점검 스크립트를 작성하여 7개 항목을 모두 출력하였다.
> 6. cron에 자동 점검을 등록하고 정리하였다.
>
> **본 실습에서 활용한 강의별 내용**
>
> | 강의 | 활용 내용 |
> |---|---|
> | 제1강 | 시스템 정보 조회, 서비스 상태 확인 |
> | 제2강 | 파일 생성, 로그 조회 |
> | 제3강 | 계정 권한 구조, 파일 권한 `640` 설정 |
> | 제4강 | 리다이렉션, 파이프, 스크립트 작성 |
> | 제5강 | 프로세스 조회, 서비스 제어, cron 등록 |
> | 제6강 | 네트워크 진단, 방화벽 구성, 웹 서비스 |
{: .prompt-tip }

---
---

# 제6절. 자주 발생하는 오류와 대응 방법

---

| 화면에 출력된 메시지 / 증상 | 원인 | 대응 방법 |
|---|---|---|
| `dig: command not found` | `bind9-dnsutils` 미설치 | `sudo apt install -y bind9-dnsutils` 또는 `resolvectl query`로 대체한다. |
| `traceroute: command not found` | `traceroute` 미설치 | `sudo apt install -y traceroute` |
| `ssh -X` 후 `DISPLAY`가 빈 값 | 접속하는 쪽에 X 서버가 없음 | 문자 기반 세션에서는 정상이다. 윈도우에 VcXsrv 등을 설치해야 값이 표시된다. |
| 외부에서 웹 페이지에 접속되지 않음 | NAT 주소(`10.0.2.x`)로 접속 시도 | 호스트 전용 주소 `192.168.56.30`으로 접속한다. |
| `netplan apply` 후에도 옛 주소가 남음 | 인터페이스 상태가 회수되지 않음 | `sudo ip addr del 옛주소/24 dev 인터페이스` 또는 재부팅한다. |
| `Network is unreachable` | 네트워크 설정이 없음 | `ip addr`로 주소 할당 여부를 확인한다. |
| `Name or service not known` | **DNS 문제** | `resolvectl status`로 DNS 서버를 확인한다. |
| `Connection refused` | 해당 포트에 서비스가 없음 | `ss -tlnp`로 대기 상태를 확인한다. |
| `Connection timed out` | **방화벽에 차단됨** | `ufw status`로 규칙을 확인한다. |
| SSH 키 등록 후에도 비밀번호를 요구 | 권한 설정 문제 | `chmod 700 ~/.ssh`, `chmod 600 authorized_keys` |
| `netplan apply` 후 접속 단절 | 설정 오류 | 콘솔에서 백업본을 복원한다. 이후에는 `netplan try`를 사용한다. |
| YAML 파싱 오류 | 탭 문자 사용 | **공백**으로 들여쓰기한다. |
| `ufw enable` 후 접속 단절 | SSH를 허용하지 않고 활성화 | 콘솔에서 `ufw allow OpenSSH`를 실행한다. |
| 서비스는 동작하나 외부 접속 불가 | 방화벽 미개방 | `sudo ufw allow 포트/tcp` |

---
---

# 제7절. 이론 평가

---

**문항 1.** X 윈도 시스템에서 사용자의 화면과 입력 장치를 관리하는 구성 요소는?

① X 클라이언트 ② **X 서버** ③ 윈도 매니저 ④ 디스플레이 매니저

---

**문항 2.** KDE와 GNOME이 각각 사용하는 툴킷을 바르게 짝지은 것은?

① KDE–GTK, GNOME–Qt
② **KDE–Qt, GNOME–GTK**
③ 양쪽 모두 Qt
④ 양쪽 모두 GTK

---

**문항 3.** SSH의 기본 포트 번호는?

① 21 ② **22** ③ 23 ④ 25

---

**문항 4.** 잘 알려진 포트(Well-known port)의 번호 범위는?

① 0 ~ 255 ② **0 ~ 1023** ③ 1024 ~ 49151 ④ 49152 ~ 65535

---

**문항 5.** `netstat -tulnp`를 대체하는 iproute2 계열 명령은?

① `ip route` ② `ip addr` ③ **`ss -tulnp`** ④ `ip link`

---

**문항 6.** Ubuntu 24.04에서 네트워크 설정을 담당하는 도구와 설정 파일의 위치로 옳은 것은?

① ifcfg – `/etc/sysconfig/network-scripts/`
② **netplan – `/etc/netplan/*.yaml`**
③ interfaces – `/etc/network/interfaces`
④ NetworkManager – `/etc/resolv.conf`

---

**문항 7.** 원격 접속 상태에서 네트워크 설정을 안전하게 적용하기 위해, 확인 입력이 없으면 자동으로 이전 설정으로 복원하는 명령은?

① `netplan apply` ② `netplan generate` ③ **`netplan try`** ④ `netplan get`

---

**문항 8.** 도메인 이름을 IP 주소로 변환할 때, DNS 질의보다 **먼저** 참조되는 파일은?

① `/etc/resolv.conf` ② **`/etc/hosts`** ③ `/etc/services` ④ `/etc/nsswitch.conf`

---

**문항 9.** 원격 접속 중에 `ufw`를 활성화할 때 **가장 먼저** 수행하여야 하는 작업은?

① 기본 정책을 allow로 설정한다.
② **SSH 포트를 허용하는 규칙을 추가한다.**
③ 로그 기록을 활성화한다.
④ 인터페이스를 재시작한다.

---

**문항 10.** 컨테이너 기술이 기반으로 삼는 리눅스 커널 기능의 조합으로 옳은 것은?

① systemd와 GRUB
② **namespace와 cgroups**
③ SELinux와 AppArmor
④ LVM과 RAID

---
---

# 제8절. 요약

---

## 8.1 핵심 개념 정리

| 구분 | 요점 |
|---|---|
| 주소 체계 | IP는 호스트를, 포트는 서비스를 식별한다. 0~1023은 잘 알려진 포트로 바인딩에 root 권한이 필요하다. |
| 진단 절차 | 루프백 → IP 주소 → 게이트웨이 → 외부 IP → 도메인 순으로 원인 구간을 좁혀 나간다. |
| DNS 문제의 판별 | IP로는 통신되나 도메인으로만 실패하면 DNS 문제이다. |
| 명령 체계 | 구형 `ifconfig`·`netstat` 대신 신형 `ip`·`ss`를 사용한다. |
| 네트워크 설정 | Ubuntu는 netplan YAML로 관리하며, 원격 작업 시 반드시 `netplan try`를 사용한다. |
| 방화벽 | **SSH 허용 → 기본 정책 → 활성화**의 순서를 준수한다. |
| SSH | 키 기반 인증을 구성하고 root 직접 로그인과 비밀번호 인증을 제한하는 것이 표준이다. `~/.ssh` 권한이 느슨하면 키가 무시된다. |
| X 윈도 | 커널이 아닌 응용 계층에서 동작하며, **사용자 앞의 화면이 X 서버**이다. KDE는 Qt, GNOME은 GTK를 사용한다. |
| 컨테이너 | namespace(격리)와 cgroups(자원 제한)라는 커널 기능 위에 구축되어 있다. |

---

## 8.2 본 강의에서 학습한 명령어

| 명령어 | 기능 |
|---|---|
| `ip -br addr` | IP 주소 조회 |
| `ip route` | 라우팅 테이블 조회 |
| `ping` | 도달 가능성 확인 |
| `ss -tulnp` | 개방된 포트 조회 |
| `dig` / `host` | DNS 조회(`bind9-dnsutils` 설치 필요) |
| `resolvectl query` / `getent hosts` | DNS 조회(기본 설치) |
| `netplan get` / `generate` / `try` | 네트워크 설정 조회 / 검사 / 안전 적용 |
| `ufw allow` / `enable` / `status` | 방화벽 규칙 / 활성화 / 조회 |
| `ssh` | 원격 접속 |
| `scp` / `rsync` | 파일 전송 / 증분 동기화 |
| `ssh-keygen` / `ssh-copy-id` | 키 생성 / 등록 |
| `sshd -t` | SSH 설정 문법 검사 |

---
---

# 제9절. 전체 과정 정리

---

## 9.1 여섯 강의의 구성

| 강의 | 주제 | 핵심 내용 |
|---|---|---|
| 제1강 | 리눅스의 이해와 시스템 구조 | 계층 구조, 라이선스, 배포판, 프롬프트, 부팅 과정, 파일 시스템 |
| 제2강 | 디렉터리 구조와 기본 명령어 | FHS, 경로, `ls -l` 해석, 파일 조작, `find`·`grep`, `tar` |
| **제3강** | **계정 관리와 접근 권한** | **계정 파일, `chmod`, `umask`, 특수 권한** |
| 제4강 | 셸의 이해와 텍스트 편집기 | 환경 변수, 초기화 파일, 리다이렉션·파이프, `vi`·`nano` |
| 제5강 | 프로세스 관리와 소프트웨어 설치 | `ps`·`top`, 시그널, `systemctl`, `cron`, `apt` |
| 제6강 | 네트워크 설정과 리눅스의 활용 | `ip`·`ss`, netplan, `ufw`, SSH, X 윈도, 컨테이너 |
| **부록** | **Metasploitable 2 구축과 시스템 점검** | **취약 시스템 격리 구성, 학습 내용을 적용한 보안 점검** |

---

## 9.2 학습 이후의 방향

| 방향 | 주요 학습 내용 |
|---|---|
| 서버 운영 | 웹 서버, 데이터베이스, 백업·복구 체계 |
| 자동화 | 셸 스크립트 심화, 구성 관리 도구 |
| 클라우드 | 클라우드 인프라, 컨테이너, 오케스트레이션 |
| **정보 보안** | 취약점 진단, 로그 분석, 침해 대응 |

리눅스마스터 2급 자격을 준비하는 경우, 본 과정에서 다룬 명령어의 **옵션 단위 숙지**와 함께 **제3강의 계정·권한 영역**을 반복 학습할 것을 권장한다. 출제 비중이 가장 높을 뿐 아니라, 이후의 서버 운영과 보안 학습 전반의 기반이 되는 내용이기 때문이다.

---

## 9.3 부록 안내

본 과정을 마친 뒤에는 **부록. Metasploitable 2 구축과 시스템 점검 실습**을 수행할 것을 권장한다.

의도적으로 취약하게 구성된 시스템을 격리된 환경에 추가로 구축하고, 제1강부터 제6강까지 학습한 조회 명령만으로 그 시스템의 위험 요소를 식별하는 실습이다. 안전한 설정과 위험한 설정을 직접 비교함으로써 본 과정에서 학습한 내용의 의미를 확인할 수 있다.

---

## 9.4 실습 자료의 보존

본 과정에서 생성한 `~/lab01` ~ `~/lab06`, `~/mission03` ~ `~/mission06` 디렉터리는 학습 기록으로서 가치가 있으므로 삭제하지 않고 보존할 것을 권장한다. 이후 유사한 작업을 수행할 때 참고 자료로 활용할 수 있다.
