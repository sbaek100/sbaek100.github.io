---
title: "[AI 보안관제 구축] 3-1. 웹 방화벽(WAF) 이해와 리버스 프록시 세우기"
date: 2027-07-19 09:00:00 +0900
categories:
  - 1.응용강의
  - AI보안관제구축
tags:
  - 보안관제
  - WAF
  - Nginx
  - 리버스프록시
  - 웹보안
pin: false
math: false
mermaid: false
---

> 지난 시간 도달 상태: 경계에 OPNsense 방화벽이 서서 **포트**를 막고, Suricata가 허용된 포트로 들어오는 **공격 패턴을 탐지**합니다. 그래도 SQL 인젝션은 표적까지 **도달**합니다.
> 오늘의 목표: 표적 서버 **바로 앞**에 웹 전용 방어 지점이 될 **WAF 서버(VM3 · `waf` · 192.168.59.30)** 를 세우고, Nginx **리버스 프록시**로 모든 웹 요청이 이 서버를 거쳐 가게 만듭니다. 그리고 **프록시만으로는 공격이 막히지 않는다**는 사실을 눈으로 확인합니다.
> 오늘 켜는 VM: **OPNsense + Kali + victim + waf** 네 대.
{: .prompt-info }

> **이 편의 명령이 기준으로 삼은 버전**
>
> | 구성요소 | 버전 | 확인 명령 |
> | --- | --- | --- |
> | WAF 서버 OS | **Ubuntu Server 24.04 LTS**(noble) | `lsb_release -a` |
> | 웹 서버 | **Nginx 1.24.0** (우분투 기본 저장소) | `nginx -v` |
> | 방화벽 | **OPNsense 26.1 이상** (2-1편에서 설치) | Lobby → Dashboard |
> | 표적 웹앱 | OWASP Juice Shop (Docker `bkimminich/juice-shop`) | victim에서 `sudo docker ps` |
>
> 네트워크 설정은 **netplan**(Ubuntu 18.04 이후 방식) 기준이고, 방화벽 화면은 OPNsense 26.x 기준입니다. 다른 배포판·버전에서는 명령이나 메뉴 이름이 달라질 수 있습니다.
{: .prompt-tip }

## 0. 오늘의 그림

지금까지 웹 요청은 방화벽을 지나 **표적 서버로 곧장** 갔습니다.

```text
[ Kali ] 192.168.57.10 ──▶ [ OPNsense ] ──▶ [ victim ] 192.168.59.20:3000
   공격자망 soc-wan          방화벽·IDS            표적(Juice Shop)
```

오늘 이 경로를 한 번 꺾습니다. 표적 앞에 **WAF 서버**를 세우고, 바깥에서 들어오는 웹은 **오직 WAF만** 만날 수 있게 만듭니다.

```text
[ Kali ] 192.168.57.10 ──▶ [ OPNsense ] ──▶ [ waf ] 192.168.59.30:80
   공격자망 soc-wan          방화벽·IDS         Nginx 리버스 프록시
                                                    │  (노출망 soc-dmz 내부)
                                                    ▼
                                             [ victim ] 192.168.59.20:3000
                                                    표적(Juice Shop)
```

**표적으로 가는 문(3000)은 방화벽에서 닫습니다.** 그러면 공격자가 표적을 직접 만질 방법이 없어지고, 모든 웹 요청은 반드시 WAF를 통과해야 합니다. 이것이 WAF가 "검사할 기회"를 얻는 유일한 방법입니다.

> **오늘은 아직 검사하지 않습니다.** 오늘 세우는 것은 **길목**뿐입니다. 그 길목에 검사기(ModSecurity)를 얹는 것은 다음 시간(3-2)입니다. 순서를 나눈 이유는, 길목이 제대로 서기 전에 검사기를 얹으면 "웹이 안 되는 것"이 프록시 탓인지 검사기 탓인지 가릴 수 없기 때문입니다.
{: .prompt-tip }

---

## 1. 이론 ① — 방화벽·IDS가 있는데 왜 또 WAF인가

2단계를 마친 지금, 우리에게는 이미 두 겹의 방어가 있습니다. 그런데도 SQL 인젝션은 표적에 닿았습니다. 각 장비가 **보는 것이 다르기** 때문입니다.

| 장비 | 보는 것 | 판단 기준 | 못 보는 것 |
| --- | --- | --- | --- |
| **방화벽**(OPNsense) | IP 주소와 **포트** | "이 문을 열어 둘 것인가" | 열어 둔 문으로 무엇이 들어오는지 |
| **IDS/IPS**(Suricata) | 지나가는 **패킷의 내용** | "알려진 공격 문자열이 있는가" | 그 요청이 **웹에서 무슨 의미**인지 |
| **WAF**(ModSecurity) | **HTTP 요청 자체** | "이 요청이 웹 애플리케이션에 어떤 짓을 하는가" | 웹이 아닌 트래픽(관심 밖) |

Suricata도 분명 SQL 인젝션을 잡아냈습니다. 그렇다면 WAF는 무엇을 더 하는 걸까요? 네 가지가 결정적으로 다릅니다.

**첫째, 요청을 "해석"한 뒤 검사합니다.** Suricata는 지나가는 바이트를 규칙 문자열과 대조합니다. WAF는 HTTP 요청을 **파싱**해서 "이건 URL, 이건 파라미터 `q` 의 값, 이건 쿠키, 이건 JSON 본문의 `email` 필드" 로 나눈 다음, 각 조각을 따로 검사합니다. 그래서 **"파라미터 `q` 에서만 이 규칙을 끄자"** 같은 정밀한 운영이 가능합니다. 3-3에서 이 성질을 그대로 활용합니다.

**둘째, 요청 본문(POST body)을 봅니다.** 2단계에서 우리가 켠 SQL 규칙은 전부 **URL(HTTP URI)** 을 보는 규칙이었습니다. 그런데 Juice Shop의 로그인은 `POST /rest/user/login` 의 **JSON 본문**에 아이디와 비밀번호를 담아 보냅니다. 1-3편에서 성공했던 `' OR 1=1;--` 로그인 우회가 바로 그것입니다. **URL만 보는 눈에는 이 공격이 보이지 않습니다.** WAF는 본문을 열어 보므로 잡습니다. (3-2에서 직접 확인합니다.)

**셋째, 정규화(normalization)를 합니다.** 공격자는 같은 공격을 여러 모양으로 씁니다. `UNION` 을 `%55NION` 으로 쓰거나, 두 번 인코딩하거나, 대소문자를 섞거나, 주석을 끼워 넣습니다. WAF는 검사 전에 **디코딩·소문자 변환·공백 정리 같은 변환(transformation)** 을 먼저 거칩니다. 시그니처 하나가 수십 가지 변형을 함께 커버하는 이유입니다.

**넷째, 위치가 다릅니다.** IDS는 경계에 있고 WAF는 **웹 서버 바로 앞**에 있습니다. 나중에 HTTPS를 쓰게 되면 경계의 IDS는 암호화된 내용을 볼 수 없지만, WAF는 **복호화가 끝난 자리**에 서 있어 그대로 검사할 수 있습니다.

> **그렇다고 Suricata가 필요 없어지는 것은 아닙니다.** Suricata는 웹이 아닌 공격(포트 스캔·SSH 무차별 대입 등)과 **모든 망**의 트래픽을 봅니다. WAF는 웹 하나만 깊이 봅니다. 계층 방어(defense in depth)란 **서로 다른 눈을 겹쳐 두는 것**이고, 4단계 SIEM에서 이 두 눈의 기록을 한 화면에 모을 때 그 가치가 드러납니다.
{: .prompt-info }

---

## 2. 이론 ② — WAF의 역사와 배치 방식

### 2.1 어디서 왔는가

| 시기 | 사건 | 의미 |
| --- | --- | --- |
| 1990년대 후반 | 웹 애플리케이션 급증, SQL 인젝션·XSS 공개 | 방화벽이 못 막는 공격이 주류가 됨 |
| **2002년** | **ModSecurity** 공개 (Ivan Ristić, Apache 모듈) | 오픈소스 WAF의 출발점 |
| 2006년 | **OWASP Core Rule Set(CRS)** 시작 | "규칙은 공용 자산" — 누구나 쓰는 표준 규칙 모음 |
| 2006~2008년 | **PCI DSS 6.6** 요구사항 | 카드 결제 시스템은 소스 코드 검토 **또는 WAF** 중 하나를 반드시 — WAF가 업계 표준이 된 결정적 계기 |
| 2017년 | **libmodsecurity(ModSecurity v3)** | 엔진을 라이브러리로 분리 → Apache 밖(**Nginx** 등)에서도 사용 가능 |
| 2020년대 | 클라우드 WAF 보편화, CRS 3.x 성숙 | 규칙을 직접 굴리는 대신 서비스로 사는 선택지도 생김 |
| 2024년 | ModSecurity 엔진의 유지보수를 **OWASP CRS 프로젝트**가 이어받음 | 엔진과 규칙이 한 우산 아래로 |

> **PCI DSS 6.6 이야기를 기억해 두세요.** "코드를 다 고치든지, 아니면 앞에 WAF를 세우든지"라는 요구는 WAF의 존재 이유를 정확히 설명합니다. **이미 돌아가는 애플리케이션의 코드를 당장 못 고칠 때, 앞에서 대신 막아 주는 장치**가 WAF입니다. 이것을 **가상 패치(virtual patching)** 라고 부릅니다.
>
> 우리 실습이 딱 그 상황입니다. Juice Shop은 일부러 취약하게 만든 앱이라 **고칠 수 없습니다.** 대신 앞에 WAF를 세워 공격이 닿지 못하게 합니다.
{: .prompt-tip }

### 2.2 어디에 놓는가 — 배치 방식 네 가지

| 방식 | 설명 | 장단점 |
| --- | --- | --- |
| **리버스 프록시** | WAF가 웹 서버인 척 요청을 먼저 받고, 검사 후 뒤쪽 진짜 서버로 넘김 | 애플리케이션을 전혀 건드리지 않음 · 별도 서버가 필요 |
| 서버 내장 모듈 | 웹 서버(Apache/Nginx) 안에 모듈로 얹음 | 서버가 하나면 간단 · 웹 서버마다 설치해야 함 |
| 인라인 브리지 | 네트워크 경로에 투명하게 끼워 넣음 | IP 구조 변경 없음 · 장비/구성이 까다로움 |
| 클라우드 WAF | CDN 사업자가 앞단에서 대신 검사 | 설치가 없음 · 트래픽을 외부에 맡겨야 함 |

**이 강의는 리버스 프록시를 씁니다.** 이유는 세 가지입니다.

1. **표적을 전혀 건드리지 않습니다.** victim 서버의 Juice Shop 컨테이너는 1-3편 그대로 둡니다. "앞에 세우는 방어"라는 개념이 눈에 보입니다.
2. **실무에서 가장 흔한 모양입니다.** 웹 서버가 여러 대라도 앞에 한 대만 세우면 전부 보호됩니다.
3. **다음 단계와 이어집니다.** WAF가 남기는 로그가 한곳(59.30)에 모이므로, 4단계에서 SIEM 에이전트를 한 대에만 붙이면 됩니다.

> 🖼 **필요 이미지 ①** — 리버스 프록시 배치 개념도. 왼쪽에 공격자, 가운데 WAF(검사 후 통과/차단), 오른쪽에 웹 서버. "표적으로 가는 직통 경로는 닫혀 있다"는 점이 드러나면 좋습니다.
{: .prompt-tip }

### 2.3 리버스 프록시가 곧 방어는 아닙니다

여기서 흔한 오해를 미리 끊어 둡니다. **프록시를 세운다고 안전해지지 않습니다.** 프록시는 요청을 **전달**할 뿐입니다. 검사기를 얹기 전까지는, 공격도 정직하게 잘 전달합니다. 오늘 마지막 장에서 같은 SQL 인젝션을 다시 보내 **그대로 통하는 것**을 확인하는 이유가 이것입니다.

다만 프록시는 그 자체로 두 가지 이득을 줍니다. **표적을 직접 노출하지 않는 것**(공격 표면 축소), 그리고 **모든 요청이 한 곳을 지나가게 만드는 것**(검사와 로그의 단일 지점). 이 두 번째가 오늘의 진짜 목표입니다.

---

## 3. 따라 하기: WAF 서버(VM3) 만들기

기준표(1-1편)의 **VM3 `waf`** 를 만듭니다. 1-3편의 victim과 같은 방식이라 익숙할 것입니다.

| 항목 | 값 |
| --- | --- |
| 이름 | **`waf`** |
| 운영체제 | Ubuntu Server 24.04 LTS |
| 메모리 | **1536 MB** |
| 프로세서 | 1 |
| 디스크 | 20 GB (동적 할당) |
| IP | **192.168.59.30** (`soc-dmz`) |
| 계정 | `student` / `student1234` |

> ### 따라 하기 3-1. 빈 가상머신 생성
>
> **목적** Ubuntu를 설치할 VM을 만듭니다.
{: .prompt-tip }

**1단계.** VirtualBox 관리자에서 **새로 만들기(New)** 를 누릅니다. 1-3편에서 받아 둔 **Ubuntu Server 24.04 LTS `.iso`** 를 그대로 씁니다.

| 항목 | 값 |
| --- | --- |
| 이름(Name) | **`waf`** |
| ISO 이미지 | 1-3편에서 받은 Ubuntu Server `.iso` |
| 종류(Type) | Linux |
| 버전(Version) | Ubuntu (64-bit) |
| 자동 설치 건너뛰기(Skip Unattended Installation) | **체크** |

**2단계.** 하드웨어를 설정합니다.

| 항목 | 값 |
| --- | --- |
| 메모리(RAM) | **`1536` MB** |
| 프로세서(CPU) | **`1`** |
| 디스크 | **`20` GB** (동적 할당) |

> **메모리를 victim보다 작게 잡는 이유** — WAF 서버는 Docker도 데이터베이스도 돌리지 않고 Nginx 하나만 띄웁니다. 오늘부터는 VM을 **네 대** 동시에 켜야 하므로(방화벽 2048 + Kali 3072 + victim 2048 + waf 1536 ≒ 8.7 GB), 호스트 메모리를 아껴야 합니다. 호스트가 8 GB 이하라면 Kali를 2048 MB로 낮추고 진행하세요.
{: .prompt-warning }

**3단계.** VM을 선택하고 **설정(Settings) → 네트워크** 에서 어댑터 두 개를 지정합니다.

- **어댑터 1**: 다음에 연결됨 = **NAT** (설치·패키지 다운로드용 — 오늘 안에 끕니다)
- **어댑터 2**: 다음에 연결됨 = **내부 네트워크**, 이름 = **`soc-dmz`**

> 📷 **필요 이미지 ②** — `waf` VM의 네트워크 설정에서 어댑터 1=NAT, 어댑터 2=내부 네트워크 `soc-dmz` 로 지정된 화면.
{: .prompt-tip }

> ### 따라 하기 3-2. Ubuntu Server 설치
>
> **목적** 최소 구성으로 운영체제를 올립니다.
{: .prompt-tip }

**4단계.** VM을 **시작(Start)** 하고 설치 마법사를 진행합니다. 1-3편과 동일하며, 언급하지 않은 항목은 **기본값(Done)** 입니다.

| 단계 | 선택 |
| --- | --- |
| 언어 / 키보드 | English / 기본값 |
| 설치 종류 | **Ubuntu Server**(일반) |
| 네트워크 | 그대로 Done (지금은 NAT로 자동 연결됨) |
| 저장소(디스크) | 전체 디스크 사용, 기본값 |
| Profile(계정) | 아래 표대로 |
| SSH 설치 | **Install OpenSSH server 체크** |
| 추천 스냅 | 아무것도 선택하지 않음 |

| 항목 | 값 |
| --- | --- |
| Your name | `student` |
| Your server's name | **`waf`** |
| Pick a username | **`student`** |
| Password | **`student1234`** |

**5단계.** 설치가 끝나면 **Reboot Now** 를 고르고, 재부팅 후 `student` / `student1234` 로 로그인합니다.

---

## 4. 따라 하기: Nginx 설치와 고정 IP

### 4.1 먼저 Nginx를 받아 둡니다

> ### 따라 하기 4-1. Nginx 설치 (NAT가 켜져 있는 지금)
>
> **목적** 인터넷이 확실히 되는 동안 필요한 꾸러미를 받아 둡니다.
{: .prompt-tip }

**1단계.** waf 서버 터미널에서 아래를 실행합니다.

```bash
sudo apt update
sudo apt install -y nginx
```

**2단계.** 잘 올라왔는지 확인합니다.

```bash
systemctl is-active nginx
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1/
```

`active` 와 `200` 이 나오면 됩니다. (기본 페이지 "Welcome to nginx!" 가 응답한 것입니다.)

> **왜 지금 설치하나요?** 잠시 뒤 NAT 어댑터를 끄면, 이 서버의 인터넷은 **OPNsense 방화벽을 통해서만** 열립니다. 2-2편에서 표적망(OPT1)에 열어 둔 것은 **DNS(53)와 80·443** 뿐이라 `apt` 는 계속 되지만, 순서를 이렇게 잡아 두면 문제가 생겨도 원인을 가르기 쉽습니다.
{: .prompt-info }

### 4.2 고정 IP를 주고 NAT 다리를 끊습니다

> ### 따라 하기 4-2. NAT 어댑터 끄기
>
> **목적** 바깥으로 나가는 길을 방화벽 하나로 모읍니다.
{: .prompt-tip }

**3단계.** waf 서버를 종료합니다.

```bash
sudo poweroff
```

**4단계.** VirtualBox에서 `waf` **설정 → 네트워크 → 어댑터 1** 의 **"네트워크 어댑터 사용하기" 체크를 해제**합니다. 어댑터 2(`soc-dmz`)는 그대로 둡니다. 그리고 VM을 다시 시작합니다.

> ### 따라 하기 4-3. netplan으로 192.168.59.30 고정
>
> **목적** 기준표대로 주소를 못박고, 게이트웨이를 방화벽(59.1)으로 잡습니다.
{: .prompt-tip }

**5단계.** 내부망 랜카드 이름을 확인합니다.

```bash
ip -brief address
```

NAT를 껐으므로 `10.0.2.x` 주소는 더 이상 보이지 않습니다. `lo` 를 뺀 나머지 하나(대개 **`enp0s8`**)가 어댑터 2입니다.

> **이름이 `enp0s3` 로 보일 수도 있습니다.** Ubuntu Server는 랜카드가 꽂힌 **슬롯 번호**로 이름을 짓기 때문에 보통 어댑터 2는 `enp0s8` 그대로입니다. 다르게 보인다면 **보이는 그 이름**을 아래 명령에 넣으세요.
{: .prompt-warning }

**6단계.** 아래를 통째로 붙여 넣습니다. (`enp0s8` 은 5단계에서 확인한 실제 이름으로.)

```bash
sudo tee /etc/netplan/99-soc.yaml >/dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: no
      addresses: [192.168.59.30/24]
      routes:
        - to: default
          via: 192.168.59.1
      nameservers:
        addresses: [192.168.59.1]
EOF
sudo chmod 600 /etc/netplan/99-soc.yaml
sudo netplan apply
```

**7단계.** 주소와 경로를 확인합니다.

```bash
ip -brief address show enp0s8
ip route
```

`192.168.59.30/24` 와 `default via 192.168.59.1` 이 보이면 성공입니다.

> ### 따라 하기 4-4. 방화벽을 통한 인터넷 확인
>
> **목적** 2-2편에서 만든 규칙이 이 서버에도 그대로 적용되는지 봅니다.
{: .prompt-tip }

**8단계.** 두 가지를 확인합니다.

```bash
ping -c 3 8.8.8.8
sudo apt update
```

| 명령 | 기대 결과 | 이유 |
| --- | --- | --- |
| `ping 8.8.8.8` | **실패(100% packet loss)** | 2-2편 규칙은 80·443·53만 열어 두었습니다. ICMP는 허용 대상이 아닙니다 |
| `sudo apt update` | **성공** | `dmz_out_ports`(80·443) 허용 규칙과 DNS(53) 허용 규칙이 살아 있기 때문입니다 |

**이 두 결과가 함께 나오는 것이 정상입니다.** "핑은 안 되는데 apt는 된다"는 것은 최소 허용 규칙이 의도대로 동작한다는 뜻입니다.

> **`apt update` 가 실패한다면** — 2-2편 4장의 OPT1 규칙 중 **DNS 허용(`Resolve Target Name`)** 과 **업데이트 허용(`update https/http`)** 두 개가 살아 있는지 OPNsense에서 확인하세요. 그래도 급하면, VirtualBox에서 `waf` 의 **어댑터 1(NAT)을 잠시 다시 켜서** 패키지를 받고 끄는 방법으로 우회할 수 있습니다. 다만 원인은 나중에라도 찾아 두어야 3-2에서 같은 곳에서 막히지 않습니다.
{: .prompt-warning }

---

## 5. 따라 하기: Nginx 리버스 프록시 구성

이제 Nginx가 **자기 기본 페이지 대신** 표적 서버의 Juice Shop을 대신 내주도록 바꿉니다.

> ### 따라 하기 5-1. 프록시 설정 파일 작성
>
> **목적** 들어온 웹 요청을 표적(59.20:3000)으로 넘기게 합니다.
{: .prompt-tip }

**1단계.** 설정 파일을 만듭니다. 통째로 붙여 넣으세요.

```bash
sudo tee /etc/nginx/sites-available/juice-proxy.conf >/dev/null <<'EOF'
# WebSocket 업그레이드 요청을 그대로 전달하기 위한 준비
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/juice_access.log;
    error_log  /var/log/nginx/juice_error.log;

    location / {
        proxy_pass http://192.168.59.20:3000;

        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        $connection_upgrade;

        proxy_read_timeout 300s;
    }
}
EOF
```

**2단계.** 기본 사이트를 내리고 우리 설정을 켭니다.

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/juice-proxy.conf /etc/nginx/sites-enabled/
sudo nginx -t
```

`syntax is ok` · `test is successful` 두 줄이 나와야 합니다.

**3단계.** 적용합니다.

```bash
sudo systemctl reload nginx
```

> **설정 파일의 각 줄이 하는 일**
>
> | 지시어 | 하는 일 |
> | --- | --- |
> | `proxy_pass` | 받은 요청을 표적 서버로 넘깁니다. **오늘의 핵심 한 줄** |
> | `proxy_set_header Host` | 원래 요청의 호스트 이름을 그대로 전달합니다 |
> | `X-Real-IP` · `X-Forwarded-For` | **진짜 요청자의 IP** 를 뒤쪽 앱에 알려 줍니다. 이것이 없으면 Juice Shop 눈에는 모든 손님이 WAF(59.30)로 보입니다 |
> | `Upgrade` · `Connection` | Juice Shop이 쓰는 **WebSocket** 연결을 끊지 않고 넘겨 줍니다 |
> | `map $http_upgrade ...` | WebSocket 요청일 때만 `Connection: upgrade` 를 보내도록 하는 준비 작업입니다 |
{: .prompt-tip }

> ### 따라 하기 5-2. WAF 서버 자신에서 확인
{: .prompt-tip }

**4단계.** waf 서버에서 직접 확인합니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1/
curl -s http://127.0.0.1/ | head -5
```

`200` 이 나오고, HTML 조각에 `OWASP Juice Shop` 같은 글자가 보이면 **프록시가 성공한 것**입니다. Nginx 기본 페이지("Welcome to nginx!")가 나온다면 2단계의 심볼릭 링크 작업을 다시 확인하세요.

> **502 Bad Gateway가 나온다면** 표적 서버 쪽 문제입니다. victim(59.20)에서 `sudo docker ps` 로 `juice-shop` 이 `Up` 인지 확인하고, waf에서 `curl -s -o /dev/null -w "%{http_code}\n" http://192.168.59.20:3000/` 이 `200`을 주는지 보세요. **waf와 victim은 같은 `soc-dmz` 망(59.0/24)에 있어 이 통신은 방화벽을 거치지 않습니다** — 그래서 방화벽 규칙과는 무관한 문제입니다.
{: .prompt-warning }

---

## 6. 따라 하기: 방화벽 규칙 바꾸기 — 표적을 감추고 WAF만 노출

프록시가 서 있어도, 공격자가 표적(59.20:3000)에 **직접** 갈 수 있다면 아무 의미가 없습니다. WAF를 우회하면 그만이니까요. 그래서 방화벽에서 **길을 하나로 강제**합니다.

> **오늘 실습에서 가장 중요한 장입니다.** WAF는 "지나가지 않으면 검사할 수 없는" 장치입니다. 3-2에서 ModSecurity가 아무리 잘 잡아도, 표적으로 가는 옆문이 열려 있으면 방어는 성립하지 않습니다.
{: .prompt-danger }

> ### 따라 하기 6-1. WAF 별칭 만들기
{: .prompt-tip }

**1단계.** Kali 브라우저에서 OPNsense(`https://192.168.57.1`) 로그인 → **Firewall → Aliases** → **+ Add**.

| 항목 | 값 |
| --- | --- |
| Name | **`waf_host`** |
| Type | Host(s) |
| Content | `192.168.59.30` |
| Description | WAF 서버 |

**2단계.** 한 번 더 **+ Add** 로 포트 별칭을 만듭니다.

| 항목 | 값 |
| --- | --- |
| Name | **`waf_ports`** |
| Type | Port(s) |
| Content | `80` |
| Description | WAF 공개 포트 |

**3단계.** 목록 아래쪽 **Apply** 를 눌러 별칭을 적용합니다.

> ### 따라 하기 6-2. LAN 규칙 다시 짜기
>
> **목적** 웹은 WAF로만, 표적으로 가는 직통은 전부 차단합니다.
{: .prompt-tip }

**4단계.** **Firewall → Rules → LAN** 으로 갑니다. 2-2편에서 만든 규칙이 이 순서로 있을 것입니다.

| 현재 순서 | 규칙 | 오늘 할 일 |
| --- | --- | --- |
| 1 | 표적 웹 허용 (Pass → `victim_host` : `web_ports`) | **비활성화** ← 표적 직통을 끊습니다 |
| 2 | 표적 그 외 전부 차단 (Block → `victim_host`) | 그대로 둡니다 |
| 3 | LAN 기본 허용 규칙 | 그대로 둡니다 |

**5단계.** 1번 규칙의 **Enabled 체크를 해제**합니다(오른쪽 연필 아이콘 → `Enabled` 해제 → Save). 삭제하지 않고 꺼 두는 이유는, 문제가 생겼을 때 되돌리기 쉽기 때문입니다.

**6단계.** **+ Add** 로 WAF 허용 규칙을 만듭니다.

| 항목 | 값 |
| --- | --- |
| Action | **Pass** |
| Interface | LAN |
| Direction | in |
| Protocol | **TCP** |
| Source | LAN net |
| Destination | **`waf_host`** |
| Destination Port | **`waf_ports`** |
| Description | `allow web to waf` |

**7단계.** 한 번 더 **+ Add** 로, WAF의 나머지 문을 막습니다.

| 항목 | 값 |
| --- | --- |
| Action | **Block** |
| Interface | LAN |
| Direction | in |
| Protocol | any |
| Source | LAN net |
| Destination | **`waf_host`** |
| **Log** | **체크** |
| Description | `block other to waf` |

> **이 규칙이 왜 필요한가요?** 맨 아래 **LAN 기본 허용 규칙** 때문입니다. 이것이 살아 있는 한, 명시적으로 막지 않은 곳은 전부 통합니다. WAF 서버에는 **SSH(22)** 가 열려 있으므로, 막지 않으면 공격자 Kali가 WAF에 SSH로 접속을 시도할 수 있습니다. **방어 장비 자신이 공격 표면이 되어서는 안 됩니다.**
{: .prompt-warning }

**8단계.** 순서를 맞춥니다. 새 규칙은 목록 **맨 아래**에 붙으므로, 드래그해서 아래 순서로 만듭니다.

| 순서 | 규칙 | 동작 |
| --- | --- | --- |
| 1 | `allow web to waf` | Pass — WAF 80만 허용 |
| 2 | `block other to waf` | Block — WAF의 나머지 차단 |
| 3 | 표적 웹 허용 *(비활성)* | (꺼져 있음) |
| 4 | 표적 그 외 전부 차단 | Block — 표적 직통 전면 차단 |
| 5 | LAN 기본 허용 규칙 | 그대로 |

**9단계.** 목록 아래쪽 주황색 **`Apply`** 를 누릅니다.

> 📷 **필요 이미지 ③** — 위 다섯 줄 순서가 보이는 LAN 규칙 목록 화면(3번 줄이 회색 비활성 상태로 보이면 더 좋습니다).
{: .prompt-tip }

> **2-3 부록(IPS)을 진행했다면 한 가지 더.** 그때 만든 `Divert-to` 규칙의 **Destination이 아직 `victim_host` : `web_ports`** 로 되어 있습니다. 이제 웹은 WAF로 들어오므로, 그 규칙의 Destination을 **`waf_host` : `waf_ports`** 로 바꾸고 **Apply** 하세요. 그러지 않으면 Suricata에게 넘어가는 트래픽이 사라져 IPS가 하는 일이 없어집니다.
>
> IDS(PCAP 모드)만 쓰고 있다면 아무것도 바꾸지 않아도 됩니다. Suricata는 LAN 인터페이스를 지나는 트래픽을 전부 보고 있으므로, 목적지가 59.30으로 바뀌어도 계속 탐지합니다.
{: .prompt-warning }

---

## 7. 따라 하기: 같은 공격 다시 실행 — 무엇이 바뀌고 무엇이 그대로인가

이제 1단계부터 써 온 **똑같은 공격**을 다시 보냅니다. 오늘은 **바뀌는 것 하나**와 **안 바뀌는 것 하나**를 동시에 확인하는 것이 목적입니다.

> ### 따라 하기 7-1. 표적이 사라졌는지 확인
{: .prompt-tip }

**1단계.** Kali에서 표적을 직접 스캔합니다.

```bash
sudo nmap -Pn 192.168.59.20
```

3000 포트를 포함해 **전부 `filtered`** 로 나옵니다. 2단계에서는 3000만 `open` 이었는데, 이제 그 문마저 닫혔습니다.

**2단계.** 웹으로도 직접 닿는지 봅니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 http://192.168.59.20:3000/
```

응답이 오지 않고 `000` 이 나옵니다. **공격자 입장에서 표적 서버는 이제 존재하지 않는 것과 같습니다.**

**3단계.** 이번엔 WAF를 스캔합니다.

```bash
sudo nmap -Pn 192.168.59.30
```

**80 포트만 `open`**, 나머지(22 포함)는 `filtered` 로 나옵니다. 노출된 문이 딱 하나로 줄었습니다.

> ### 따라 하기 7-2. 그런데 공격은 여전히 통합니다
{: .prompt-tip }

**4단계.** Kali 브라우저에서 `http://192.168.59.30` 에 접속합니다. Juice Shop이 그대로 열립니다. **주소가 표적(59.20:3000)이 아니라 WAF(59.30)** 라는 점에 주목하세요. 앞으로 모든 실습의 주소는 이것입니다.

**5단계.** 2단계에서 쓰던 SQL 인젝션을 **주소만 바꿔** 보냅니다.

```bash
curl -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"
```

응답이 **그대로 돌아옵니다.** (Juice Shop의 DB가 SQLite라 `SQLITE_ERROR: no such table` 500 응답이 오는 것도 2단계와 똑같습니다.)

**6단계.** 로그인 우회도 다시 해 봅니다. 브라우저에서 `http://192.168.59.30` → **Account → Login** → 이메일 칸에 `' OR 1=1;--`, 비밀번호는 아무 값이나 넣고 로그인하면 **여전히 관리자로 들어가집니다.**

> **오늘의 결론 — 프록시는 길목이지 방어가 아닙니다.**
>
> | 항목 | 2단계까지 | 오늘(3-1) |
> | --- | --- | --- |
> | 표적 직접 접근 | 3000 열림 | **불가(전부 차단)** |
> | 노출된 문 | 표적의 3000 | **WAF의 80 하나** |
> | SQL 인젝션 | 성공 | **여전히 성공** |
> | 로그인 우회 | 성공 | **여전히 성공** |
>
> 바뀐 것은 **경로**뿐입니다. 그러나 이 경로 변경이 없으면 다음 시간의 검사기는 아무 소용이 없습니다. **오늘 우리는 검문소를 세울 자리를 만들었습니다.**
{: .prompt-info }

**7단계(선택).** Suricata를 켜 두었다면 **Services → Intrusion Detection → Administration → Alerts** 를 확인하세요. 목적지가 **192.168.59.30** 으로 바뀐 채 SQLi 경보가 그대로 올라옵니다. 경계의 IDS는 목적지가 바뀌어도 계속 탐지합니다.

---

## 8. 따라 하기: 스냅샷과 점검

**1단계.** waf 서버를 종료합니다.

```bash
sudo poweroff
```

**2단계.** VirtualBox에서 `waf` 선택 → **스냅샷 → 만들기** → 이름 **`waf-proxy`** 로 저장합니다. (OPNsense도 규칙을 바꿨으니 `fw-waf-front` 로 하나 더 저장해 두면 좋습니다.)

점검 목록:

- [ ] `waf` VM이 설치되고 IP가 **192.168.59.30** 으로 고정됐다
- [ ] `waf` 의 NAT 어댑터를 끄고, 게이트웨이가 **192.168.59.1** 이다
- [ ] waf에서 `ping 8.8.8.8` 은 실패하고 `sudo apt update` 는 성공한다
- [ ] waf에서 `curl http://127.0.0.1/` 이 Juice Shop 화면을 돌려준다
- [ ] OPNsense에 `waf_host` · `waf_ports` 별칭을 만들었다
- [ ] LAN 규칙이 **WAF 80 허용 → WAF 그 외 차단 → 표적 전면 차단** 순서로 정리됐다
- [ ] Kali에서 `nmap 192.168.59.20` 이 **전부 filtered** 다
- [ ] Kali에서 `nmap 192.168.59.30` 이 **80만 open** 이다
- [ ] Kali 브라우저 `http://192.168.59.30` 에서 Juice Shop이 열린다
- [ ] 같은 SQL 인젝션이 **아직 통한다**는 것을 확인했다 (다음 시간의 출발점)

---

## 다음 시간

길목은 만들었지만 검사는 아직 하지 않습니다. **3-2** 에서는 이 Nginx에 **ModSecurity 엔진**과 **OWASP CRS(공개 규칙 모음)** 를 얹습니다. 2단계에서 IDS를 먼저 켜고 IPS로 올렸던 것처럼, WAF도 **탐지만 하는 모드(DetectionOnly)로 먼저 켜서 무엇이 걸리는지 본 뒤** 차단으로 올립니다. 오늘 통과했던 그 SQL 인젝션이 **403** 으로 막히는 것을 확인하게 됩니다.
