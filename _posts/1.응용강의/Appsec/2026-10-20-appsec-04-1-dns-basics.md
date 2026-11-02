---
title: "[애플리케이션 보안] 04-1. DNS 개념과 DNS 서버 운영"
date: 2026-10-20 09:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - DNS보안
tags:
  - DNS
  - BIND
  - Zone
  - DNS레코드
  - dig
  - nslookup
math: false
mermaid: false
---


## 0. 실습 환경

| 장비 | IP | 역할 |
|---|---|---|
| Kali | **192.168.0.10** | DNS에 질의하는 쪽(클라이언트) |
| Ubuntu | **192.168.0.30** | DNS 서버(BIND), 도메인 `lab.test` 관리 |

> 코드 블록 첫 줄의 `# === Ubuntu ===` / `# === Kali ===` 를 보고 **어느 VM에서 칠지** 확인한 뒤 붙여 넣습니다.
>
> 명령과 예상 화면은 **Ubuntu 24.04.4 LTS(BIND 9.18.39)** 와 **Kali Rolling(DiG 9.20.23)** 에서 실행해 확인했습니다. Transaction ID, 시각, 응답 시간처럼 실행할 때마다 바뀌는 값과, 인터넷 도메인의 IP 주소는 화면과 다를 수 있습니다. 1.5절의 실습 5·6은 **Ubuntu가 인터넷에 연결**되어 있어야 합니다.
{: .prompt-warning }

> 실습 도메인은 `lab.test` 입니다. `.test` 는 **시험용으로 예약된 이름**이라 실제 인터넷 도메인과 겹치지 않습니다. 메일 실습에서 쓴 `.local` 은 같은 네트워크 안의 장치 찾기(mDNS) 용도로 예약되어 있어, DNS 실습에 쓰면 `dig` 가 경고를 냅니다.
{: .prompt-info }

이 글에서 Kali와 Ubuntu가 하는 일은 다음과 같습니다.

| 절 | Kali에서 할 일 | Ubuntu에서 할 일 |
|---|---|---|
| 1.2 이름을 찾는 순서 | `/etc/hosts` 내용 확인 | — |
| 2.2 BIND 설치 | — | BIND 설치, 실행 계정 확인 |
| 2.3 존 만들기 | — | 정방향·역방향 존 파일 작성, 문법 검사, 적용 |
| 2.4 조회 | `dig`·`nslookup` 으로 레코드 조회 | — |
| 2.5 캐시·TTL, UDP·TCP | 같은 이름을 두 번 조회, 큰 응답 조회 | (재귀 질의를 대신 처리) |

---

# 1. DNS 개념

## 1.1 DNS와 도메인 이름의 구조

### [이론]

사람은 `www.lab.test` 같은 **이름**을 기억하지만, 컴퓨터는 `192.168.0.30` 같은 **IP 주소**로 통신합니다. **DNS(Domain Name System)** 는 이름을 IP 주소로, 또는 IP 주소를 이름으로 바꿔 주는 시스템입니다. 인터넷의 **전화번호부**라고 생각하면 됩니다.

도메인 이름은 **오른쪽에서 왼쪽으로** 갈수록 범위가 좁아지는 계층 구조입니다.

| 부분 | `www.naver.com.` 에서 | 이름 | 관리 주체 |
|---|---|---|---|
| 맨 오른쪽 `.` | `.` | **루트(root)** | 전 세계 루트 서버 |
| 최상위 도메인 | `com` | **TLD**(Top-Level Domain) | `com` 은 Verisign, `kr` 은 **한국인터넷진흥원(KISA)** |
| 2단계 도메인 | `naver` | 도메인 | 도메인을 등록한 회사 |
| 호스트 | `www` | 호스트(서버) 이름 | 도메인 주인 |

> 보통 `www.naver.com` 처럼 끝의 `.` 을 생략해 쓰지만, DNS 안에서는 항상 루트 `.` 으로 끝납니다. 2.3절 존 파일에서 이 점이 중요해집니다.
{: .prompt-info }

| TLD 종류 | 예 | 뜻 |
|---|---|---|
| gTLD(일반) | `com`, `net`, `org` | 용도별 최상위 도메인 |
| ccTLD(국가) | `kr`, `jp`, `us` | 국가별 최상위 도메인. 우리나라 `.kr` 은 **KISA** 가 관리 |

#### 루트 서버와 Anycast

모든 이름 찾기의 출발점인 **루트 서버**는 `a.root-servers.net` ~ `m.root-servers.net` 의 **13개 이름**으로 운영됩니다. 실제 서버는 수천 대이며, 같은 IP 주소를 전 세계 여러 곳에 두고 **가장 가까운 서버로 연결**하는 **Anycast** 방식을 씁니다.

| 항목 | 내용 |
|---|---|
| Anycast | 같은 IP 주소를 여러 지역의 서버가 함께 쓰고, 라우팅으로 **가장 가까운 곳**에 연결 |
| 사용하는 라우팅 프로토콜 | 인터넷 사업자 간 라우팅인 **BGP** (내부망용 OSPF가 아님) |
| 장점 | 한 곳이 DDoS 공격을 받아도 다른 곳이 응답 → **가용성** 향상 |
| 예 | 루트 서버의 국내 미러, 공개 DNS `1.1.1.1`(Cloudflare), `8.8.8.8`(Google) |

---

## 1.2 이름을 찾는 순서 — hosts 파일, 캐시, DNS 서버

### [이론]

PC가 `www.naver.com` 의 IP 주소를 찾을 때는 가까운 곳부터 차례로 확인합니다.

| 순서 | 확인하는 곳 | 설명 |
|---|---|---|
| ① | **hosts 파일** | PC 안의 파일. 이름과 IP를 직접 적어 둠. 리눅스 `/etc/hosts`, 윈도우 `C:\Windows\System32\drivers\etc\hosts` |
| ② | **DNS 캐시** | 최근에 찾은 답을 잠시 기억해 둔 것 |
| ③ | **DNS 서버** | PC에 설정된 DNS 서버(보통 통신사나 회사의 DNS 서버)에 질의 |

**hosts 파일이 DNS보다 먼저** 적용됩니다. 그래서 DNS 서버가 고장 났을 때나, 시험용으로 특정 서버에 강제로 접속시킬 때 hosts 파일에 직접 적어 씁니다. 반대로 공격자가 hosts 파일을 바꾸면 사용자를 가짜 사이트로 보낼 수 있습니다(04-2 3.4절).

> **hosts** 는 PC(클라이언트)가 이름을 찾을 때 먼저 보는 파일이고, **named.conf** 는 DNS **서버** 프로그램(BIND)의 설정 파일입니다. 이름이 비슷한 역할로 보여도 쓰는 쪽이 다릅니다.
{: .prompt-warning }

#### 재귀 질의와 반복 질의

③에서 DNS 서버가 답을 모르면, 루트부터 차례로 물어 답을 찾아옵니다.

| 순서 | 묻는 쪽 → 답하는 쪽 | 주고받는 내용 |
|---|---|---|
| ① | PC → 로컬 DNS 서버 | "`www.naver.com` 의 IP를 **끝까지 찾아서** 알려 줘" |
| ② | 로컬 DNS → 루트 서버 | "`com` 은 누가 관리하나?" → "com 서버에게 물어라" |
| ③ | 로컬 DNS → `com` 서버 | "`naver.com` 은?" → "naver.com 서버에게 물어라" |
| ④ | 로컬 DNS → `naver.com` 서버 | "`www.naver.com` 은?" → "IP는 `x.x.x.x`" (**권한 있는 답**) |
| ⑤ | 로컬 DNS → PC | 답을 돌려주고, **캐시에 저장** |

| 구분 | 누가 누구에게 | 뜻 |
|---|---|---|
| **재귀 질의(Recursive)** | PC → 로컬 DNS 서버 (①) | "알아서 끝까지 찾아서 **최종 답만** 달라"고 맡기는 질의 |
| **반복 질의(Iterative)** | 로컬 DNS → 루트·TLD·권한 서버 (②~④) | 각 서버가 "다음은 누구에게 물어라"고만 알려 주고, 묻는 쪽이 **차례로 다시** 묻는 질의 |

| DNS 서버 종류 | 하는 일 |
|---|---|
| **권한 있는(Authoritative) 서버** | 자기 도메인(존)의 원본 정보를 가지고 **직접 답함**. 이 글의 Ubuntu가 `lab.test` 의 권한 서버 |
| **캐시(재귀) 서버** | PC 대신 재귀 질의를 처리하고 결과를 캐시에 저장. 통신사 DNS, `8.8.8.8` 등 |

#### 캐시와 TTL

DNS 서버는 찾은 답을 **TTL(Time To Live, 초)** 동안 캐시에 보관합니다. TTL이 남아 있는 동안 같은 질문이 오면 루트까지 다시 가지 않고 바로 답합니다. 응답이 빨라지는 대신, **캐시에 가짜 정보가 들어가면 TTL 동안 계속 가짜 답을 줍니다.** 이것이 캐시 포이즈닝 공격의 원리입니다(04-2 3.5절).

### [실습] (Kali) hosts 파일 확인

```bash
# === Kali ===
cat /etc/hosts
```

```
127.0.0.1	localhost
127.0.1.1	kali

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

| 화면의 줄 | 의미 |
|---|---|
| `127.0.0.1	localhost` | `localhost` 라는 이름은 DNS에 묻지 않고 바로 `127.0.0.1`(자기 자신)로 바꿈 |
| `127.0.1.1	kali` | 이 PC의 호스트 이름 `kali` 도 자기 자신 |
| `# ...` | 설명(주석) |

> 형식은 **`IP주소  이름`** 한 줄에 하나입니다. 이 파일은 관리자(root)만 고칠 수 있습니다. 04-2에서 이 파일을 바꾸면 어떤 일이 생기는지 확인합니다.
{: .prompt-info }

---

## 1.3 DNS 레코드

### [이론]

DNS 서버는 이름에 대한 정보를 **레코드(Resource Record)** 단위로 저장합니다.

| 레코드 | 이름 | 담는 내용 | 이 글의 예 |
|---|---|---|---|
| **A** | Address | 이름 → **IPv4 주소** | `www.lab.test` → `192.168.0.30` |
| **AAAA** | — | 이름 → **IPv6 주소** | |
| **CNAME** | Canonical Name | 이름의 **별명**(다른 이름을 가리킴) | `ftp.lab.test` → `www.lab.test` |
| **MX** | Mail eXchanger | 이 도메인의 **메일을 받는 서버** | `lab.test` → `mail.lab.test` |
| **NS** | Name Server | 이 도메인을 맡은 **DNS 서버** | `lab.test` → `ns.lab.test` |
| **PTR** | Pointer | **IP 주소 → 이름**(역방향 조회) | `192.168.0.10` → `kali.lab.test` |
| **SOA** | Start Of Authority | 존(zone)의 **관리 정보**(대표 서버, 관리자, 일련번호, 갱신 주기) | 존 파일 맨 앞에 하나 |
| **TXT** | Text | 자유 글자. **SPF** 같은 메일 보안 정책 | `v=spf1 ...` (03-2 4.2절) |
| **HINFO** | Host Info | 서버의 CPU·OS 정보 | 공격자의 OS 정보 수집에 쓰일 수 있어 보통 등록하지 않음 |

> 메일 서버를 지정하는 레코드는 **MX**, DNS 서버를 지정하는 레코드는 **NS**, 존의 관리 정보는 **SOA** 입니다. 서로 바꿔 쓰지 않도록 구분하는 것이 중요합니다.
{: .prompt-warning }

---

## 1.4 DNS가 쓰는 포트 — UDP 53과 TCP 53

### [이론]

| 프로토콜 | 포트 | 쓰는 경우 |
|---|---|---|
| **UDP** | **53** | 대부분의 일반 질의·응답. 연결 과정이 없어 빠름 |
| **TCP** | **53** | ① 응답이 **512바이트를 넘을 때**(UDP 응답에 잘림 표시 `tc` 가 오면 TCP로 다시 질의) ② **존 전송**(2.1절) |

> 원래 DNS의 UDP 응답은 512바이트까지였습니다. 지금은 **EDNS** 라는 확장으로 UDP로도 더 큰 응답(보통 1232바이트 안팎)을 주고받지만, 그보다 크면 여전히 TCP를 씁니다. 그래서 방화벽에서 DNS를 허용할 때는 **UDP 53과 TCP 53을 모두** 생각해야 합니다. 2.5절 실습 6에서 직접 확인합니다.
{: .prompt-info }

---

# 2. DNS 서버 운영 — BIND

## 2.1 BIND와 존

### [이론]

**BIND(Berkeley Internet Name Domain)** 는 리눅스에서 가장 많이 쓰는 DNS 서버 프로그램입니다. 실행 프로그램 이름은 **`named`**(name daemon)입니다.

**존(zone)** 은 하나의 DNS 서버가 책임지고 관리하는 범위입니다. 이 글에서 Ubuntu는 `lab.test` 존을 맡습니다. 존의 레코드를 적어 둔 파일이 **존 파일**입니다.

| 파일(Ubuntu) | 하는 일 |
|---|---|
| `/etc/bind/named.conf` | 주 설정 파일. 아래 세 파일을 불러오기(include)만 함 |
| `/etc/bind/named.conf.options` | 서버 전체에 적용하는 **옵션**(재귀 허용 범위, 존 전송 허용 범위 등. 04-2에서 수정) |
| `/etc/bind/named.conf.local` | 내가 관리할 **존 목록**(존 이름과 존 파일 위치) |
| `/etc/bind/db.이름` | **존 파일**. 실제 레코드가 들어 있음 |
| `/var/cache/bind` | 캐시 등 작업 파일을 두는 곳 |

#### Primary·Secondary와 존 전송

DNS 서버가 하나뿐이면 고장 났을 때 도메인 전체가 멈춥니다. 그래서 보통 **두 대 이상**을 둡니다.

| 구분 | 하는 일 |
|---|---|
| **Primary**(주, 예전 이름 Master) | 존 파일 **원본**을 가진 서버 |
| **Secondary**(보조, 예전 이름 Slave) | Primary에서 존을 **복사해 와서** 똑같이 답하는 예비 서버 |
| **존 전송(Zone Transfer)** | Secondary가 Primary의 존을 복사해 가는 것. 전체 복사는 **AXFR**, 바뀐 부분만 복사는 **IXFR**. **TCP 53** 사용 |

> 존 전송을 **아무에게나 허용하면**, 공격자도 존 전체를 복사해 가서 도메인 안의 서버 이름과 IP 목록을 한 번에 알아냅니다. 04-2 3.2절에서 이 공격과 대책을 실습합니다.
{: .prompt-warning }

---

## 2.2 BIND 설치

### [실습] (Ubuntu) BIND 설치와 실행 확인

```bash
# === Ubuntu ===
sudo apt update && sudo apt install -y bind9 bind9-dnsutils
systemctl is-active named
ps -o user,pid,cmd -C named
named -v
```

| 명령·옵션 | 설명 |
|---|---|
| `apt install -y bind9 bind9-dnsutils` | DNS 서버 **BIND** 와 조회 도구 모음(`dig`, `nslookup` 등)을 설치 |
| `systemctl is-active named` | DNS 서버(`named`)가 동작 중인지 한 단어로 표시 |
| `ps -o user,pid,cmd -C named` | 실행 중인 프로그램 가운데 `named` 만 골라(`-C`), **실행 계정·프로세스 번호·실행 명령**(`-o user,pid,cmd`)을 표시 |
| `named -v` | 설치된 BIND의 버전 표시 |

`apt` 는 설치 과정을 길게 출력합니다. 끝부분은 다음과 같습니다(설치 진행 줄 수십 개는 `...` 로 생략).

```
...
Setting up bind9-utils (1:9.18.39-0ubuntu0.24.04.8) ...
Setting up bind9 (1:9.18.39-0ubuntu0.24.04.8) ...
info: Adding group `bind' (GID 114) ...
info: Adding system user `bind' (UID 108) ...
info: Adding new user `bind' (UID 108) with group `bind' ...
info: Not creating home directory `/var/cache/bind'.
wrote key file "/etc/bind/rndc.key"
Created symlink /etc/systemd/system/bind9.service → /usr/lib/systemd/system/named.service.
Created symlink /etc/systemd/system/multi-user.target.wants/named.service → /usr/lib/systemd/system/named.service.
...
```

나머지 세 명령의 결과는 다음과 같습니다.

```
active
USER         PID CMD
bind        2468 /usr/sbin/named -f -u bind
BIND 9.18.39-0ubuntu0.24.04.8-Ubuntu (Extended Support Version) <id:>
```

> GID·UID 숫자와 프로세스 번호(`2468`)는 환경마다 다릅니다. `bind9-dnsutils` 가 이미 설치되어 있으면 그 패키지 줄은 나오지 않습니다.
{: .prompt-info }

| 화면 | 의미 |
|---|---|
| `Adding system user 'bind'` | 설치할 때 DNS 서버 전용 계정 `bind` 를 만듦 |
| `active` | DNS 서버가 동작 중 |
| `bind ... /usr/sbin/named -f -u bind` | `named` 가 **root가 아닌 `bind` 계정**으로 실행 중. `-u bind` 가 실행 계정을 지정하는 옵션 |
| `BIND 9.18.39-...` | 설치된 버전. 이 정보가 밖으로 노출되면 공격자가 그 버전의 취약점을 찾음(04-2 3.3절) |

> DNS 서버를 **root 권한으로 실행하면** 버퍼 오버플로 같은 취약점이 공격당했을 때 공격자가 곧바로 root 권한을 얻습니다. Ubuntu의 BIND는 처음부터 `bind` 계정으로 실행되도록 설정되어 있습니다.
{: .prompt-warning }

Ubuntu 방화벽(ufw)이 켜져 있다면 Kali에서 오는 DNS를 허용합니다. `Status: inactive` 이면 건너뜁니다.

```bash
# === Ubuntu ===
sudo ufw status
sudo ufw allow from 192.168.0.10 to any port 53
```

방화벽이 꺼져 있는(Ubuntu 기본값) 경우의 화면입니다.

```
Status: inactive
Rules updated
```

| 명령·옵션 | 설명 |
|---|---|
| `ufw allow from 192.168.0.10 to any port 53` | Kali에서 이 서버의 53번 포트로 오는 접속을 허용. `proto` 를 적지 않으면 **UDP와 TCP를 모두** 허용(1.4절) |

---

## 2.3 존 만들기

### [이론] 존 파일 읽는 법

존 파일은 한 줄에 레코드 하나를 적습니다.

```
www     IN  A      192.168.0.30
```

| 칸 | 값 | 뜻 |
|---|---|---|
| 이름 | `www` | 끝에 `.` 이 없으면 존 이름이 자동으로 붙어 `www.lab.test.` 가 됨. `@` 는 존 이름 자체(`lab.test.`) |
| 클래스 | `IN` | 인터넷(Internet). 거의 항상 `IN` |
| 종류 | `A` | 레코드 종류(1.3절) |
| 값 | `192.168.0.30` | 레코드 내용 |

> 값에 이름을 쓸 때는 **끝에 `.` 을 붙여야** 합니다. `ns.lab.test.` 처럼 `.` 으로 끝나면 "완성된 이름"이고, `.` 을 빼고 `ns.lab.test` 라고 쓰면 존 이름이 한 번 더 붙어 `ns.lab.test.lab.test.` 가 되어 버립니다. 존 파일에서 가장 흔한 실수입니다.
{: .prompt-danger }

존 파일 맨 앞의 **SOA 레코드**에는 존의 관리 정보가 들어갑니다.

| 항목 | 이 글의 값 | 뜻 |
|---|---|---|
| 대표 DNS 서버 | `ns.lab.test.` | 이 존의 Primary 서버 |
| 관리자 메일 | `admin.lab.test.` | `admin@lab.test` 를 뜻함(`@` 대신 첫 `.` 을 씀) |
| serial(일련번호) | `2026111001` | 존을 고칠 때마다 **올려야** 함. Secondary는 이 숫자가 커진 것을 보고 존을 다시 복사함. 보통 `연월일+두 자리` |
| refresh | `3600` | Secondary가 Primary에 바뀐 내용이 있는지 확인하는 간격(초) |
| retry | `900` | refresh에 실패했을 때 다시 시도하는 간격(초) |
| expire | `604800` | Primary와 계속 연락이 안 되면 이 시간(7일) 뒤에 Secondary가 답하기를 멈춤 |
| negative cache TTL | `3600` | "그런 이름 없음" 이라는 답을 캐시에 보관하는 시간(초) |

### [실습 1] (Ubuntu) 존 목록에 lab.test 등록

`named.conf.local` 에 **정방향 존**(이름 → IP) `lab.test` 와 **역방향 존**(IP → 이름) `0.168.192.in-addr.arpa` 를 추가합니다.

```bash
# === Ubuntu ===
sudo tee -a /etc/bind/named.conf.local > /dev/null <<'EOF'

zone "lab.test" {
    type primary;
    file "/etc/bind/db.lab.test";
};

zone "0.168.192.in-addr.arpa" {
    type primary;
    file "/etc/bind/db.192.168.0";
};
EOF
```

`tee` 는 아무것도 출력하지 않습니다.

> 첫 줄 `sudo tee ...` 부터 마지막 `EOF` 줄까지는 **하나의 명령**입니다. 블록 전체를 한 번에 복사해 붙여 넣습니다. `>` 표시만 계속 나오면 `Ctrl+C` 로 빠져나와 다시 붙여 넣습니다.
{: .prompt-warning }

| 명령·설정 | 설명 |
|---|---|
| `tee -a 파일` | 내용을 파일 **끝에 덧붙임**(`-a`, append). 원래 들어 있던 내용은 그대로 둠 |
| `zone "lab.test" { ... };` | `lab.test` 존을 이 서버가 맡겠다는 선언 |
| `type primary;` | 이 서버가 존의 **원본**을 가진 Primary 서버 |
| `file "...";` | 존 파일의 위치 |
| `0.168.192.in-addr.arpa` | `192.168.0.x` 대역의 역방향 존 이름. IP를 **거꾸로** 적고 `.in-addr.arpa` 를 붙임 |

### [실습 2] (Ubuntu) 정방향 존 파일 작성

```bash
# === Ubuntu ===
sudo tee /etc/bind/db.lab.test > /dev/null <<'EOF'
$TTL 3600
@       IN  SOA  ns.lab.test. admin.lab.test. (
                 2026111001  ; serial
                 3600        ; refresh
                 900         ; retry
                 604800      ; expire
                 3600 )      ; negative cache TTL
@       IN  NS     ns.lab.test.
@       IN  MX     10 mail.lab.test.
@       IN  TXT    "v=spf1 ip4:192.168.0.30 -all"
ns      IN  A      192.168.0.30
www     IN  A      192.168.0.30
mail    IN  A      192.168.0.30
ubuntu  IN  A      192.168.0.30
kali    IN  A      192.168.0.10
ftp     IN  CNAME  www.lab.test.
EOF
```

`tee` 는 아무것도 출력하지 않습니다.

| 줄 | 뜻 |
|---|---|
| `$TTL 3600` | 이 파일의 레코드를 다른 서버가 캐시에 **3600초(1시간)** 동안 보관해도 된다는 기본값 |
| `@ IN SOA ... ( ... )` | 존의 관리 정보. 괄호 안은 여러 줄에 나눠 적어도 됨. `;` 뒤는 설명(주석) |
| `@ IN NS ns.lab.test.` | `lab.test` 의 DNS 서버는 `ns.lab.test` |
| `@ IN MX 10 mail.lab.test.` | `@lab.test` 로 오는 메일은 `mail.lab.test` 가 받음. `10` 은 우선순위(작을수록 먼저) |
| `@ IN TXT "v=spf1 ..."` | 이 도메인의 메일은 `192.168.0.30` 에서만 나간다는 SPF 정책(03-2 4.2절) |
| `ns`·`www`·`mail`·`ubuntu IN A 192.168.0.30` | 네 이름 모두 Ubuntu의 IP |
| `kali IN A 192.168.0.10` | Kali의 IP |
| `ftp IN CNAME www.lab.test.` | `ftp.lab.test` 는 `www.lab.test` 의 별명 |

### [실습 3] (Ubuntu) 역방향 존 파일 작성

```bash
# === Ubuntu ===
sudo tee /etc/bind/db.192.168.0 > /dev/null <<'EOF'
$TTL 3600
@       IN  SOA  ns.lab.test. admin.lab.test. (
                 2026111001 3600 900 604800 3600 )
@       IN  NS   ns.lab.test.
30      IN  PTR  ubuntu.lab.test.
10      IN  PTR  kali.lab.test.
EOF
```

`tee` 는 아무것도 출력하지 않습니다.

| 줄 | 뜻 |
|---|---|
| SOA | 정방향 존과 같은 값을 한 줄에 적은 것 |
| `30 IN PTR ubuntu.lab.test.` | `30` 에 존 이름이 붙어 `30.0.168.192.in-addr.arpa`, 즉 **`192.168.0.30` → `ubuntu.lab.test`** |
| `10 IN PTR kali.lab.test.` | **`192.168.0.10` → `kali.lab.test`** |

### [실습 4] (Ubuntu) 문법 검사 후 적용

설정을 잘못 쓴 채로 적용하면 DNS 서버가 존을 읽지 못합니다. 적용하기 전에 **반드시 검사**합니다.

```bash
# === Ubuntu ===
sudo named-checkconf
sudo named-checkzone lab.test /etc/bind/db.lab.test
sudo named-checkzone 0.168.192.in-addr.arpa /etc/bind/db.192.168.0
sudo rndc reload
```

`named-checkconf` 는 문제가 없으면 **아무것도 출력하지 않습니다.** 나머지 명령의 결과는 다음과 같습니다.

```
zone lab.test/IN: loaded serial 2026111001
OK
zone 0.168.192.in-addr.arpa/IN: loaded serial 2026111001
OK
server reload successful
```

| 명령·옵션 | 설명 |
|---|---|
| `named-checkconf` | `named.conf` 와 그 안에서 불러오는 설정 파일의 **문법** 검사 |
| `named-checkzone 존이름 존파일` | 존 파일의 **문법과 내용** 검사. `OK` 와 함께 읽은 serial을 보여 줌 |
| `rndc reload` | 실행 중인 DNS 서버에 **설정과 존을 다시 읽으라**고 명령. `rndc` 는 BIND를 원격 조종하는 관리 도구 |

> `named-checkzone` 이 오류를 내면 줄 번호가 함께 나옵니다. 가장 흔한 원인은 이름 끝의 `.` 을 빠뜨린 것과 괄호 `( )` 를 닫지 않은 것입니다.
{: .prompt-tip }

---

## 2.4 DNS 조회 — dig와 nslookup

### [실습 1] (Kali) dig로 A 레코드 조회

`dig` 는 DNS 서버에 질의를 보내고 응답을 **자세히** 보여 주는 도구입니다. `@서버` 로 물어볼 DNS 서버를 직접 지정합니다.

```bash
# === Kali ===
dig @192.168.0.30 www.lab.test
```

```

; <<>> DiG 9.20.23-1-Debian <<>> @192.168.0.30 www.lab.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61995
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 621f0fc24586f3f4010000006ae84ec82fcccdcf476801d1 (good)
;; QUESTION SECTION:
;www.lab.test.			IN	A

;; ANSWER SECTION:
www.lab.test.		3600	IN	A	192.168.0.30

;; Query time: 0 msec
;; SERVER: 192.168.0.30#53(192.168.0.30) (UDP)
;; WHEN: Tue Oct 20 10:05:12 KST 2026
;; MSG SIZE  rcvd: 85

```

| 화면 | 의미 |
|---|---|
| `status: NOERROR` | 정상 응답. 없는 이름이면 `NXDOMAIN`, 거부되면 `REFUSED` |
| `id: 61995` | **Transaction ID**. 질의마다 붙이는 번호로, 응답에도 같은 번호가 와야 받아들임(04-2 3.5절에서 중요) |
| `flags: qr` | 이 메시지는 **응답**(query response) |
| `flags: aa` | **권한 있는 답**(authoritative answer). 이 서버가 `lab.test` 의 원본을 가지고 있음 |
| `flags: rd` / `ra` | 재귀를 **요청함**(recursion desired) / 이 서버가 재귀를 **해 줄 수 있음**(recursion available) |
| `QUESTION SECTION` | 내가 물어본 것: `www.lab.test` 의 `A` 레코드 |
| `ANSWER SECTION` | 답: `www.lab.test` 는 `192.168.0.30`. `3600` 은 캐시에 보관해도 되는 **TTL(초)** |
| `SERVER: ...#53 (UDP)` | 답한 서버와 포트, 프로토콜 |
| `MSG SIZE rcvd: 85` | 받은 응답의 크기(바이트) |
| `COOKIE: ...` | 위조 응답을 줄이기 위해 서버와 주고받는 확인 값(EDNS 기능) |

### [실습 2] (Kali) 레코드 종류별 조회

`+short` 를 붙이면 설명 없이 **답만** 출력합니다.

```bash
# === Kali ===
dig @192.168.0.30 lab.test NS +short
dig @192.168.0.30 lab.test MX +short
dig @192.168.0.30 lab.test TXT +short
dig @192.168.0.30 lab.test SOA +short
dig @192.168.0.30 ftp.lab.test +short
dig @192.168.0.30 -x 192.168.0.10 +short
```

```
ns.lab.test.
10 mail.lab.test.
"v=spf1 ip4:192.168.0.30 -all"
ns.lab.test. admin.lab.test. 2026111001 3600 900 604800 3600
www.lab.test.
192.168.0.30
kali.lab.test.
```

| 명령 | 결과 | 의미 |
|---|---|---|
| `lab.test NS` | `ns.lab.test.` | `lab.test` 의 DNS 서버 |
| `lab.test MX` | `10 mail.lab.test.` | 메일 서버와 우선순위 |
| `lab.test TXT` | `"v=spf1 ..."` | SPF 정책 |
| `lab.test SOA` | `ns.lab.test. admin.lab.test. 2026111001 ...` | 대표 서버, 관리자, serial, refresh, retry, expire, negative TTL 순서 |
| `ftp.lab.test` | `www.lab.test.` 다음 줄 `192.168.0.30` | 별명(CNAME)을 따라가 최종 IP까지 알려 줌 |
| `-x 192.168.0.10` | `kali.lab.test.` | **역방향 조회**(PTR). `-x` 가 IP를 `10.0.168.192.in-addr.arpa` 로 바꿔 질의 |

| 명령·옵션 | 설명 |
|---|---|
| `dig @서버 이름 종류` | `서버` 에게 `이름` 의 `종류` 레코드를 질의. 종류를 빼면 `A` |
| `+short` | 답의 값만 짧게 출력 |
| `-x IP주소` | 역방향(IP → 이름) 조회 |

없는 이름을 물으면 어떻게 되는지 봅니다.

```bash
# === Kali ===
dig @192.168.0.30 nobody.lab.test
```

```

; <<>> DiG 9.20.23-1-Debian <<>> @192.168.0.30 nobody.lab.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 27584
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 66e49ff106b203bc010000006ae84ed4563a4270071ba535 (good)
;; QUESTION SECTION:
;nobody.lab.test.		IN	A

;; AUTHORITY SECTION:
lab.test.		3600	IN	SOA	ns.lab.test. admin.lab.test. 2026111001 3600 900 604800 3600

;; Query time: 0 msec
;; SERVER: 192.168.0.30#53(192.168.0.30) (UDP)
;; WHEN: Tue Oct 20 10:06:40 KST 2026
;; MSG SIZE  rcvd: 117

```

| 화면 | 의미 |
|---|---|
| `status: NXDOMAIN` | 그런 이름이 **없음**(Non-eXistent Domain) |
| `ANSWER: 0` | 답이 없음 |
| `AUTHORITY SECTION` 의 SOA | "없다"는 답의 근거로 존의 SOA를 함께 보냄. 맨 끝 `3600` 동안 이 "없음"을 캐시에 보관 |

### [실습 3] (Kali) nslookup으로 조회

`nslookup` 은 윈도우와 리눅스에 모두 있는 간단한 조회 도구입니다.

```bash
# === Kali ===
nslookup www.lab.test 192.168.0.30
nslookup -type=mx lab.test 192.168.0.30
```

```
Server:		192.168.0.30
Address:	192.168.0.30#53

Name:	www.lab.test
Address: 192.168.0.30

Server:		192.168.0.30
Address:	192.168.0.30#53

lab.test	mail exchanger = 10 mail.lab.test.

```

| 명령·옵션 | 설명 |
|---|---|
| `nslookup 이름 서버` | `서버` 에게 `이름` 을 질의. 서버를 빼면 PC에 설정된 DNS 서버에 질의 |
| `-type=mx` | 조회할 레코드 종류 지정 |
| `Server:` / `Address:` | 질의한 DNS 서버 |
| `Name:` / `Address:` | 답 |

> `dig` 는 응답의 헤더·플래그·TTL까지 모두 보여 주므로 보안 점검에 주로 쓰고, `nslookup` 은 답만 간단히 볼 때 씁니다.
{: .prompt-tip }

---

## 2.5 캐시·TTL과 UDP·TCP 확인

### [실습 5] (Kali) 같은 이름을 두 번 조회해 캐시 확인

Ubuntu의 BIND는 `lab.test` 의 권한 서버이면서, 같은 네트워크의 PC를 위해 **재귀 질의도 대신 처리**합니다. Kali가 인터넷 도메인을 물으면 Ubuntu가 루트부터 찾아와 답하고 캐시에 저장합니다.

```bash
# === Kali ===
dig @192.168.0.30 www.daum.net
```

```

; <<>> DiG 9.20.23-1-Debian <<>> @192.168.0.30 www.daum.net
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12178
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 351f5a844c6521c7010000006ae84f0b39323480b93687a2 (good)
;; QUESTION SECTION:
;www.daum.net.			IN	A

;; ANSWER SECTION:
www.daum.net.		300	IN	CNAME	daum-4vdtymgd.kgslb.com.
daum-4vdtymgd.kgslb.com. 10	IN	A	121.53.105.193

;; Query time: 1422 msec
;; SERVER: 192.168.0.30#53(192.168.0.30) (UDP)
;; WHEN: Tue Oct 20 10:08:23 KST 2026
;; MSG SIZE  rcvd: 122

```

5초쯤 기다렸다가 **같은 명령**을 다시 실행합니다.

```bash
# === Kali ===
dig @192.168.0.30 www.daum.net
```

```

; <<>> DiG 9.20.23-1-Debian <<>> @192.168.0.30 www.daum.net
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38157
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 513ce5e5d592b02d010000006ae84f106edb52b2d31fae64 (good)
;; QUESTION SECTION:
;www.daum.net.			IN	A

;; ANSWER SECTION:
www.daum.net.		295	IN	CNAME	daum-4vdtymgd.kgslb.com.
daum-4vdtymgd.kgslb.com. 5	IN	A	121.53.105.193

;; Query time: 0 msec
;; SERVER: 192.168.0.30#53(192.168.0.30) (UDP)
;; WHEN: Tue Oct 20 10:08:28 KST 2026
;; MSG SIZE  rcvd: 122

```

> CNAME 이름과 IP 주소, TTL 값, 응답 시간은 다음 사이트의 설정과 네트워크 상태에 따라 다르게 나옵니다. 두 번째 조회에서 **응답 시간이 0에 가깝고 TTL이 줄어든 것**만 확인하면 됩니다.
{: .prompt-info }

| 비교 | 첫 번째 | 두 번째 | 의미 |
|---|---|---|---|
| `Query time` | `1422 msec` | `0 msec` | 첫 번째는 루트·TLD·권한 서버를 차례로 찾아감. 두 번째는 **캐시에서 바로** 답함 |
| `www.daum.net` 의 TTL | `300` | `295` | 캐시에 남은 시간. 5초 지나 5만큼 줄어듦. 0이 되면 캐시에서 지우고 다시 찾아감 |
| `flags` 에 `aa` | 없음 | 없음 | Ubuntu는 `daum.net` 의 권한 서버가 아니므로 **권한 없는 답**(캐시에서 가져온 답) |

### [실습 6] (Kali) 큰 응답은 TCP로 — UDP·TCP 53

`google.com` 의 TXT 레코드는 수십 줄이라 응답이 큽니다. `+noedns` 로 EDNS 확장을 끄고 **예전 방식(UDP 최대 512바이트)** 으로 물어봅니다.

```bash
# === Kali ===
dig @192.168.0.30 google.com TXT +noedns +ignore | grep -E 'flags:|SERVER|MSG SIZE'
dig @192.168.0.30 google.com TXT +noedns | grep -E 'Truncated|SERVER|MSG SIZE'
```

```
;; flags: qr tc rd ra ad; QUERY: 1, ANSWER: 8, AUTHORITY: 0, ADDITIONAL: 0
;; SERVER: 192.168.0.30#53(192.168.0.30) (UDP)
;; MSG SIZE  rcvd: 486
;; Truncated, retrying in TCP mode.
;; SERVER: 192.168.0.30#53(192.168.0.30) (TCP)
;; MSG SIZE  rcvd: 1176
```

| 명령·옵션 | 설명 |
|---|---|
| `+noedns` | EDNS 확장을 쓰지 않음 → UDP 응답이 **512바이트**로 제한됨 |
| `+ignore` | 응답이 잘려도 TCP로 다시 묻지 않고 **잘린 응답을 그대로** 보여 줌 |
| <code>grep -E 'flags:&#124;SERVER&#124;MSG SIZE'</code> | 긴 출력 가운데 플래그·서버·크기 줄만 골라냄. `-E` 는 <code>&#124;</code>(또는)를 쓰는 옵션 |

| 화면 | 의미 |
|---|---|
| 첫 명령의 `flags: ... tc` | **tc(TrunCated, 잘림)**. 512바이트에 다 들어가지 않아 일부(`ANSWER: 8`)만 UDP로 옴 |
| `MSG SIZE rcvd: 486` | UDP로 받은 응답은 512바이트 이하 |
| `Truncated, retrying in TCP mode.` | 잘림 표시를 보고 `dig` 가 **TCP로 다시 질의** |
| `(TCP)`, `MSG SIZE rcvd: 1176` | TCP로 1176바이트 전체를 받음 |

> 레코드 개수와 크기는 `google.com` 의 설정에 따라 달라집니다. **UDP 응답에 `tc` 가 붙고, 다시 물으면 `(TCP)` 로 더 큰 응답이 온다**는 흐름만 확인합니다. 그래서 방화벽에서 **TCP 53을 허용하면 512바이트가 넘는 큰 응답도 받을 수 있습니다.**
{: .prompt-tip }

---

# 정리

| 개념 | 핵심 |
|---|---|
| 도메인 구조 | 루트(`.`) → TLD(`com`, `kr`) → 도메인 → 호스트. `.kr` 은 KISA가 관리 |
| 루트 서버 | 13개 이름, Anycast(BGP)로 여러 곳에 미러 |
| 이름 찾는 순서 | **hosts 파일 → DNS 캐시 → DNS 서버** |
| 재귀 / 반복 질의 | PC → 로컬 DNS는 **재귀**(끝까지 찾아 달라), 로컬 DNS → 루트·TLD·권한 서버는 **반복** |
| 캐시·TTL | 찾은 답을 TTL 동안 보관. 빠르지만 가짜 정보가 들어가면 TTL 동안 유지됨 |
| 레코드 | A·AAAA(IP), CNAME(별명), **MX(메일)**, **NS(DNS 서버)**, PTR(역방향), **SOA(존 관리 정보)**, TXT(SPF) |
| 포트 | **UDP 53**(일반), **TCP 53**(512바이트 초과 응답, 존 전송) |
| BIND 설정 | `named.conf.options`(옵션), `named.conf.local`(존 목록), `db.*`(존 파일) |
| 실행 계정 | `named` 는 root가 아닌 **`bind`** 계정으로 실행 |
| 적용 순서 | **`named-checkconf` → `named-checkzone` → `rndc reload`** |

---

# 체크리스트

- [ ] 도메인 이름을 루트·TLD·도메인·호스트로 나누어 설명 가능
- [ ] 재귀 질의와 반복 질의가 각각 누구와 누구 사이의 질의인지 설명 가능
- [ ] (Kali) `/etc/hosts` 의 형식과 역할 확인
- [ ] (Ubuntu) BIND가 `active` 이고 `bind` 계정으로 실행 중임을 확인
- [ ] (Ubuntu) `lab.test` 정방향 존과 `0.168.192.in-addr.arpa` 역방향 존을 만들고 `named-checkzone` 에서 `OK` 확인
- [ ] (Kali) `dig` 의 `status`, `id`, `flags`(`qr aa rd ra`), TTL을 읽을 수 있음
- [ ] (Kali) NS·MX·TXT·SOA·CNAME·PTR을 조회하고 각 레코드의 역할 설명 가능
- [ ] (Kali) 두 번째 조회에서 `Query time` 이 줄고 TTL이 감소하는 캐시 동작 확인
- [ ] (Kali) `tc` 플래그와 TCP 재질의로 UDP·TCP 53의 쓰임 확인

> 다음 글 **04-2** 에서는 이 DNS 서버를 대상으로 존 전송·버전 노출·재귀 개방 같은 **설정 취약점을 직접 확인한 뒤 막고**, hosts 변조·캐시 포이즈닝·증폭 공격과 DNSSEC을 다룹니다.
{: .prompt-info }
