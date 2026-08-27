---
title: "[AI 보안관제 구축] 1-3. 표적 서버 만들기와 첫 공격 — Ubuntu + Juice Shop, 무방비 상태 확인"
date: 2027-06-21 09:00:00 +0900
categories:
  - 1.응용강의
  - AI보안관제구축
tags:
  - 보안관제
  - Ubuntu
  - Docker
  - JuiceShop
  - SQLInjection
pin: false
math: false
mermaid: false
---

> 지난 시간 도달 상태: 공격자 VM(Kali · `attacker` · 192.168.59.10) 준비 완료.
> 오늘의 목표: **표적 서버(VM4 · `victim`)** 를 세우고 취약한 웹앱(Juice Shop)을 올린 뒤, Kali에서 **첫 공격**을 날려 방어 장치가 하나도 없을 때 공격이 그대로 통하는 것을 확인합니다.
> 오늘 켜는 VM: **VM1(Kali) + VM4(victim)** 두 대.
{: .prompt-info }

## 0. 오늘의 그림

지금은 방화벽·WAF가 아직 없습니다. 공격자와 표적이 **같은 노출망(`soc-dmz`)** 에 나란히 있어, 공격이 아무 저항 없이 표적에 닿습니다.

```text
[ Kali 공격자 ]  192.168.59.10
        │   (soc-dmz 노출망, 방어장치 없음)
        ▼
[ Ubuntu 표적 ]  192.168.59.20  ─ Juice Shop (취약 웹앱, 포트 3000)
```

이 **무방비 상태**가 앞으로 모든 방어 단계의 "before" 기준입니다. 2단계에서 방화벽을 끼우면 같은 공격이 어떻게 막히는지 비교하게 됩니다.

---

## 1. 따라 하기: Ubuntu Server 설치용 VM 만들기

> ### 따라 하기 1-1. 설치 이미지(ISO) 내려받기
>
> **목적** 표적 서버의 운영체제 이미지를 준비합니다.
{: .prompt-tip }

**1단계.** 브라우저에서 Ubuntu 서버 이미지를 받습니다.

```text
https://ubuntu.com/download/server
```

**Ubuntu Server 24.04 LTS** 의 `.iso` 파일을 내려받습니다.

> ### 따라 하기 1-2. 빈 가상머신 생성
>
> **목적** Ubuntu를 설치할 빈 VM을 만듭니다.
{: .prompt-tip }

**2단계.** VirtualBox 관리자에서 **새로 만들기(New)** 를 누르고 다음과 같이 설정합니다.

| 항목                                       | 값                          |
| ---------------------------------------- | -------------------------- |
| 이름(Name)                                 | **`victim`**               |
| ISO 이미지                                  | 앞서 받은 Ubuntu Server `.iso` |
| 종류(Type)                                 | Linux                      |
| 버전(Version)                              | Ubuntu (64-bit)            |
| 자동 설치 건너뛰기(Skip Unattended Installation) | **체크**                     |


![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787580696813.png)

![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787580778302.png)

**3단계.** 하드웨어를 설정합니다.

| 항목 | 값 |
| --- | --- |
| 메모리(RAM) | **`2048` MB** |
| 프로세서(CPU) | **`2`** |
| 디스크 | **`25` GB** (동적 할당) |
![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787580724028.png)

![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787580733050.png)


**4단계.** **네트워크 어댑터 2개**를 지정합니다. VM을 선택하고 **설정(Settings) → 네트워크**:

- **어댑터 1**: 다음에 연결됨 = **NAT** (설치·패키지 다운로드용)
- **어댑터 2**: 다음에 연결됨 = **내부 네트워크**, 이름 = **`soc-dmz`**



> 📷 **화면 캡처 위치** — `victim` VM의 네트워크 설정에서 어댑터 1=NAT, 어댑터 2=내부 네트워크 `soc-dmz` 로 지정된 화면.
{: .prompt-tip }

---

## 2. 따라 하기: Ubuntu Server 설치

> ### 따라 하기 2-1. 설치 마법사 진행
>
> **목적** 최소 구성으로 Ubuntu Server를 설치합니다.
{: .prompt-tip }

**1단계.** `victim` VM을 **시작(Start)** 하면 설치 화면이 뜹니다. 다음 순서로 진행합니다. 특별히 언급하지 않은 항목은 **기본값(Done)** 으로 넘어갑니다.

| 단계             | 선택                            |
| -------------- | ----------------------------- |
| 언어             | English                       |
| 키보드            | 기본값                           |
| 설치 종류          | **Ubuntu Server**(일반)         |
| 네트워크           | 그대로 Done (지금은 NAT로 자동 연결됨)    |
| 저장소(디스크)       | 전체 디스크 사용, 기본값                |
| Profile(계정) 설정 | 아래 표대로 입력                     |
| SSH 설치         | **Install OpenSSH server 체크** |
| 추천 스냅          | 아무것도 선택하지 않음                  |

**2단계 — 계정.** Profile 화면에서 기준표(1-1강)대로 입력합니다.

| 항목 | 값 |
| --- | --- |
| Your name | `student` |
| Your server's name | `victim` |
| Pick a username | **`student`** |
| Password | **`student1234`** |

**3단계.** 설치가 끝나면 **Reboot Now** 를 선택합니다. "Please remove the installation medium" 메시지가 나오면 그냥 **Enter** 를 누릅니다. (VirtualBox가 ISO를 자동으로 분리합니다.)

**4단계.** 재부팅 후 로그인 프롬프트에서 `student` / `student1234` 로 로그인합니다.

> 서버판은 그래픽 화면 없이 검은 글자 터미널만 나옵니다. 정상입니다. 앞으로 이 화면에 명령어를 붙여 넣습니다.
{: .prompt-info }

---

## 3. 따라 하기: 표적 서버 고정 IP 지정 (netplan)

Ubuntu Server는 **netplan** 으로 IP를 설정합니다. 어댑터 2(내부망)에 기준표대로 **192.168.59.20** 을 고정합니다.

> ### 따라 하기 3-1. 인터페이스 이름 확인
>
> **목적** 내부망(어댑터 2) 랜카드 이름을 찾습니다.
{: .prompt-tip }

![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787581335929.png)

**1단계.**

```bash
ip -brief address
```

출력 예시:

```text
lo               UNKNOWN   127.0.0.1/8
enp0s3           UP        10.0.2.15/24     ← 어댑터1(NAT). 건드리지 않음
enp0s8           DOWN                        ← 어댑터2(내부망). 여기에 고정IP
```

- `10.0.2.x`가 붙은 것이 **어댑터 1(NAT)**.
- 나머지(`enp0s8` 등)가 **어댑터 2(내부망)** 입니다. 이름을 기억해 둡니다.
![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787581580712.png)


> ### 따라 하기 3-2. netplan 설정 파일 작성
>
> **목적** 어댑터 2에 고정 IP를 부여합니다.
{: .prompt-tip }

**2단계.** 아래 명령을 **통째로 붙여 넣습니다.** 이 명령은 설정 파일을 자동으로 만들어 줍니다.

> ⚠️ 아래에서 `enp0s8` 은 **3-1에서 확인한 실제 이름**으로 바꿔야 합니다. 이름이 다르면 그 이름으로 고쳐 넣으세요.
{: .prompt-warning }

```bash
sudo tee /etc/netplan/99-soc.yaml >/dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: no
      addresses: [192.168.59.20/24]
EOF
```

![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787581699848.png)


**3단계.** 파일 권한을 정리하고 적용합니다.

```bash
sudo chmod 600 /etc/netplan/99-soc.yaml
sudo netplan apply
```

**4단계.** 확인합니다.

```bash
ip -brief address show enp0s8
```

`192.168.59.20/24` 가 보이면 성공입니다.

![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787581812919.png)



> 게이트웨이·DNS는 넣지 않았습니다. 1단계에는 방화벽이 없어 내부망은 인터넷과 무관하고, 인터넷은 어댑터 1(NAT)이 담당합니다.
{: .prompt-info }

---

## 4. 따라 하기: Docker 설치 후 취약 웹앱(Juice Shop) 올리기

취약한 웹앱을 직접 설치하려면 복잡합니다. 대신 **Docker**(컨테이너)로 명령 한 줄에 띄웁니다. OWASP **Juice Shop** 은 교육용으로 만든 대표적인 취약 웹앱입니다.

> ### 따라 하기 4-1. Docker 설치
>
> **목적** 컨테이너 실행 환경을 준비합니다. (NAT 인터넷 필요)
{: .prompt-tip }

**1단계.**

```bash
sudo apt update
sudo apt install -y docker.io
```

**2단계.** Docker 서비스를 켜고 부팅 시 자동 실행되도록 합니다.

```bash
sudo systemctl enable --now docker
```

**3단계.** `student` 계정이 `sudo` 없이 docker를 쓸 수 있게 그룹에 추가합니다.

```bash
sudo usermod -aG docker student
```

> ⚠️ 위 그룹 변경은 **다시 로그인해야** 적용됩니다. 지금은 우선 `sudo`를 붙여 진행하고, 다음 로그인부터 `sudo` 없이 쓰면 됩니다.
{: .prompt-warning }

> ### 따라 하기 4-2. Juice Shop 컨테이너 실행
>
> **목적** 취약 웹앱을 포트 3000으로 띄웁니다.
{: .prompt-tip }

**4단계.** 다음 한 줄로 이미지를 내려받아 실행합니다.

```bash
sudo docker run -d --restart unless-stopped -p 3000:3000 --name juice-shop bkimminich/juice-shop
```

**5단계.** 정상 실행 여부를 확인합니다.

```bash
sudo docker ps
```

`juice-shop` 이 `Up ...` 상태로 보이면 성공입니다.

![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787581974107.png)



**6단계.** 서버 자신에게서 응답이 오는지 확인합니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000
```

`200` 이 출력되면 웹앱이 살아 있는 것입니다.

![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787582029100.png)

> **DVWA로 하고 싶다면(선택)**: Juice Shop 대신 `sudo docker run -d --restart unless-stopped -p 80:80 --name dvwa vulnerables/web-dvwa` 로 올릴 수 있습니다. 본 강의는 Juice Shop(포트 3000) 기준으로 진행합니다.
{: .prompt-tip }


![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787582085656.png)


---

## 5. 따라 하기: Kali에서 첫 공격 — 무방비 확인

이제 공격자(Kali)로 돌아가, 방어 장치가 없는 표적을 공격해 봅니다. **Kali VM과 victim VM을 모두 켠 상태**여야 합니다.

> ### 따라 하기 5-1. 표적이 살아 있는지 확인 (ping)
>
> **목적** 두 VM이 같은 내부망에서 서로 통하는지 봅니다.
{: .prompt-tip }

**1단계.** Kali 터미널에서:

```bash
ping -c 3 192.168.59.20
```

응답이 오면 두 VM이 연결된 것입니다. (응답이 없으면 두 VM 모두 어댑터 2가 `soc-dmz`인지, IP가 각각 59.10 / 59.20인지 확인하세요.)

![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787582114878.png)

> ### 따라 하기 5-2. 포트 스캔 (nmap)
>
> **목적** 표적에 어떤 서비스가 열려 있는지 정찰합니다. 공격의 첫 단계입니다.
{: .prompt-tip }

**2단계.** Kali 터미널에서:

```bash
nmap -sV -T4 192.168.59.20
```

결과에 **22/tcp (ssh)** 와 **3000/tcp** 가 `open` 으로 보입니다. 방어 장치가 없으니 스캔이 **아무 방해 없이** 표적의 열린 문을 그대로 알려 줍니다.

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu
3000/tcp open  ppp?
```

> **왜 3000이 `http` 가 아니라 `ppp?` 로 나올까 — 사실 3000은 HTTP가 맞습니다.**
>
> nmap의 `SERVICE` 칸은 대개 **포트 번호로 짐작한 이름**입니다. nmap의 포트 목록에 3000번은 관례상 `ppp` 로 등록돼 있어 그렇게 표시됩니다. `-sV` 로 실제 응답까지 받아 봤지만, Juice Shop(Node.js)이 nmap의 버전 지문 목록에 없어 "확실치 않다"는 뜻의 물음표(`?`)가 붙은 것입니다.
>
> 그런데 같은 화면 아래에 nmap이 받아 온 응답 원문(`SF-Port3000...` 으로 시작하는 긴 덩어리)을 보면 `HTTP/1.1 200 OK`, `Content-Type: text/html`, `OWASP Juice Shop` 같은 글자가 그대로 들어 있습니다. **즉 3000은 명백히 HTTP(웹 서버)이고**, nmap의 이름표만 애매했을 뿐입니다. 확실한 확인법은 브라우저로 `http://192.168.59.20:3000` 에 접속해 보는 것입니다(바로 다음 단계).
>
> 이 점은 뒤(2-3편 Suricata)에서 중요합니다. **Suricata는 포트 번호나 nmap의 이름표가 아니라 트래픽 내용으로 HTTP를 알아보므로**, 3000 포트로 들어오는 웹 공격도 그대로 검사합니다.
{: .prompt-info }

> ### 따라 하기 5-3. 웹앱 접속
>
> **목적** 표적 웹앱이 공격자에게 그대로 보이는지 확인합니다.
{: .prompt-tip }

![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787582197399.png)

**3단계.** Kali 바탕화면의 웹 브라우저(Firefox)를 열고 주소창에 입력합니다.

```text
http://192.168.59.20:3000
```

Juice Shop 쇼핑몰 화면이 뜹니다. 방화벽·WAF가 없으니 공격자가 표적 웹앱에 자유롭게 접근합니다.

> ### 따라 하기 5-4. SQL 인젝션으로 로그인 우회
>
> **목적** 대표적인 웹 공격이 실제로 통하는 것을 확인합니다.
{: .prompt-tip }

**4단계.** Juice Shop 화면 오른쪽 위 **Account → Login** 으로 이동합니다.

**5단계.** 로그인 칸에 다음을 입력합니다.

| 칸        | 입력값             |
| -------- | --------------- |
| Email    | `' OR 1=1;--`   |
| Password | 아무 값이나 (예: `x`) |
![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787582241358.png)


**6단계.** **Log in** 을 누르면, 비밀번호를 모르는데도 **관리자 계정으로 로그인**됩니다. 화면 위에 사용자 이메일이 표시됩니다.

> 방금 입력한 `' OR 1=1;--` 는 데이터베이스 질의문을 조작해 "항상 참"을 만드는 **SQL 인젝션(SQLi)** 공격입니다. 지금은 이 공격을 걸러 줄 장치가 하나도 없어 그대로 성공했습니다. **3단계(웹 방화벽)** 를 세우면 바로 이 요청이 403으로 차단됩니다.
{: .prompt-danger }

![](/assets/img/posts/2027-06-21-socbuild-03-victim-server-firstattack-1787582253355.png)

---

## 6. 따라 하기: 표적 서버 스냅샷

깨끗한 표적 상태를 저장해 둡니다.

**1단계.** victim 서버 터미널에서 종료:

```bash
sudo poweroff
```

**2단계.** VirtualBox 관리자에서 `victim` 선택 → **스냅샷 → 만들기** → 이름 **`victim-ready`** 로 저장합니다.

---

## 7. 1단계 정리와 점검

1단계를 마치면 다음 상태가 됩니다.

- **Kali 공격자**(59.10)와 **Ubuntu 표적**(59.20)이 같은 노출망에 있음
- 표적에는 취약 웹앱(Juice Shop)이 돌고 있음
- 방어 장치가 **하나도 없어**, 포트 스캔·웹 접근·SQL 인젝션이 **모두 성공**

점검 목록:

- [ ] `victim` 서버가 설치되고 IP가 **192.168.59.20** 으로 고정됐다
- [ ] `sudo docker ps` 에 `juice-shop` 이 `Up` 상태다
- [ ] Kali에서 `ping 192.168.59.20` 이 응답한다
- [ ] `nmap` 으로 3000 포트가 `open` 으로 보인다
- [ ] Juice Shop에서 `' OR 1=1;--` 로 로그인 우회가 성공했다
- [ ] 스냅샷 `victim-ready` 를 저장했다

---

## 다음 단계 예고 (2단계)

지금은 공격이 무방비로 통합니다. **2단계**에서는 공격자와 표적 사이에 **OPNsense 방화벽 + Suricata(IDS/IPS)** 를 끼워 넣습니다. 그러면 오늘 성공했던 **똑같은 nmap 스캔이 탐지·차단**되는 것을 직접 보게 됩니다. 이때 공격자(Kali)는 바깥 공격망(`soc-wan` · 192.168.57.10)으로 옮기고, 표적은 방화벽 뒤 DMZ(**192.168.59.20 그대로**)에 남습니다.
