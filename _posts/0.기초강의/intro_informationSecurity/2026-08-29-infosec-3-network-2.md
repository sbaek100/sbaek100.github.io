---
title: "2장 ② Windows 11 네트워크 기초 실습"
date: 2026-08-29 10:00:00 +0900
categories:
  - 0.기초강의
  - 정보보안
tags:
  - 네트워크실습
  - Windows11
  - ipconfig
  - ping
  - tracert
  - nslookup
  - netstat
  - 방화벽
  - WindowsDefenderFirewall
  - ICMP
pin: false
math: false
mermaid: false
---

# Windows 11 네트워크 기초 실습 (Step by Step)

이 문서는 앞 절 **「컴퓨터 네트워크 기초」** 의 이론을 Windows 11에서 직접 확인하는 실습 가이드임.  
실습 1~6은 조회/관찰 중심이며, 외부 서비스에 공격성 트래픽을 보내지 않음. 실습 7은 **같은 강의실 안의 옆 사람 PC**를 대상으로 하며, 반드시 상대의 동의를 얻고 진행해야 함.

---

## 실습 0. 시작 전 준비

### 이 실습이 연결되는 이론

- Part 2: LAN/WAN
- Part 3: OSI 7계층

### Step by Step

1. `Windows + S`를 누르고 `PowerShell`을 입력해 실행하면 됨.
2. 아래 명령어를 입력해 네트워크가 연결되어 있는지 먼저 확인하면 됨.

```powershell
ipconfig
```

3. 출력에서 아래 항목이 보이면 준비 완료임.
- `IPv4 Address`
- `Subnet Mask`
- `Default Gateway`

### 예상 결과(예시)

```text
IPv4 Address . . . . . . . . . . : 192.168.0.25
Subnet Mask . . . . . . . . . . : 255.255.255.0
Default Gateway . . . . . . . . : 192.168.0.1
```

`IPv4 Address`와 `Default Gateway`가 비어 있지 않으면 정상임.

---

## 실습 1. 내 PC의 네트워크 정보 확인 (L3 기본)

### 이 실습이 연결되는 이론

- Part 3.2 L3(Network): IP, 서브넷 마스크, 기본 게이트웨이

### Step by Step

1. 아래 명령어를 입력하면 됨.

```powershell
ipconfig /all
```

2. 출력에서 다음 항목을 찾으면 됨.
- `IPv4 Address`
- `Subnet Mask`
- `Default Gateway`
- `DNS Servers`

3. 왜 중요한지 이해해 보면 됨.
- `IPv4 Address`: 내 장비를 식별함.
- `Default Gateway`: 다른 네트워크로 나갈 때 첫 번째 경로임.
- `DNS Servers`: 도메인 이름 해석을 담당함.

### 예상 결과(예시)

```text
Wireless LAN adapter Wi-Fi:
   IPv4 Address. . . . . . . . . . . : 192.168.0.25
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.0.1
   DNS Servers . . . . . . . . . . . : 8.8.8.8
                                       1.1.1.1
```

---

## 실습 2. 통신 가능 여부와 지연 확인 (L3 + ICMP)

### 이 실습이 연결되는 이론

- Part 3.2 L3(Network)
- Part 4: ICMP 기반 진단 관점

### Step by Step

1. 기본 연결 테스트를 하면 됨.

```powershell
ping 8.8.8.8 -n 4
```

2. 도메인 기반 테스트를 하면 됨.

```powershell
ping www.example.com -n 4
```

3. 결과를 비교하면 됨.
- 둘 다 응답: 인터넷 + DNS가 정상 동작할 가능성이 높음.
- IP는 되는데 도메인은 실패: DNS 문제일 가능성이 있음.
- 둘 다 실패: 네트워크 연결 또는 방화벽 이슈일 수 있음.

> **ping이 실패해도 상대가 죽은 것은 아님.** 많은 서버가 보안상 ICMP 응답을 차단하도록 설정되어 있음. 예를 들어 `ping www.microsoft.com`은 응답하지 않는 경우가 많음. **응답 없음 = 장애**로 단정하면 안 됨. 이 성질은 실습 7에서 다시 다룸.
{: .prompt-warning }

### 예상 결과(예시)

```text
Reply from 8.8.8.8: bytes=32 time=20ms TTL=117
...
Ping statistics for 8.8.8.8:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

도메인 ping에서도 `Reply from ...`가 보이면 정상임.

---

## 실습 3. 목적지까지의 경로 확인 (LAN/WAN + 라우팅)

### 이 실습이 연결되는 이론

- Part 2.3 LAN/WAN
- Part 3.3 L3 경로 결정

### Step by Step

1. 아래 명령어를 입력하면 됨.

```powershell
tracert www.google.com
```

2. 출력의 `1`, `2`, `3` ... 홉(Hop)을 확인하면 됨.
- 첫 홉: 보통 내 게이트웨이 또는 가까운 라우터임.
- 홉이 늘어날수록 더 먼 네트워크 구간임.

3. 확인 포인트임.
- WAN에서 지연이 커질 수 있는 이유
- 경로가 항상 동일하지 않을 수 있는 이유

### 예상 결과(예시)

```text
  1    <1 ms    <1 ms    <1 ms  192.168.0.1
  2     3 ms     3 ms     2 ms  10.x.x.x
  3    12 ms    11 ms    13 ms  ...
```

여러 홉이 순서대로 출력되면 경로 추적이 정상 동작한 것임.

---

## 실습 4. DNS 이름 해석 확인 (L7 DNS)

### 이 실습이 연결되는 이론

- Part 4.2 DNS(Domain Name System)
- Part 3.2 L7(Application)

### Step by Step

1. 기본 질의를 실행하면 됨.

```powershell
nslookup www.example.com
```

2. 다른 도메인도 조회해 보면 됨.

```powershell
nslookup openai.com
```

3. 출력에서 아래를 읽어보면 됨.
- 어떤 DNS 서버가 응답했는지
- 도메인이 어떤 IP로 해석되는지

4. 이해 포인트임.
- 사용자는 이름을 입력하지만 실제 통신은 IP로 진행됨.
- DNS가 어긋나면 정상 사이트 접속에도 문제가 생길 수 있음.

### 예상 결과(예시)

```text
Non-authoritative answer:
Server:  one.one.one.one
Address:  1.1.1.1

Name:    www.example.com
Addresses:  2606:4700:10::ac42:93f3
          104.20.23.154
          172.66.147.243
```

`Name`과 `Address(es)`가 함께 나오면 이름 해석이 성공한 것임.

> **IP 주소는 교재와 다르게 나오는 것이 정상임.** 큰 사이트는 CDN을 쓰기 때문에 조회 시점·지역·사용하는 DNS 서버에 따라 응답 IP가 달라짐. 또한 `Server:` 줄에는 **내 PC가 사용 중인 DNS 서버**가 표시되므로, 실습 1의 `DNS Servers` 값과 같은지 비교해 보면 좋음. IPv6 주소(`2606:...`)가 함께 나오는 것도 정상임.
{: .prompt-info }

---

## 실습 5. 현재 연결과 포트 확인 (L4 Transport)

### 이 실습이 연결되는 이론

- Part 3.2 L4(Transport): TCP/UDP, 포트
- Part 4.3 TCP 3-Way Handshake 개념

### Step by Step

1. 현재 연결 상태를 확인하면 됨.

```powershell
netstat -ano
```

2. 출력에서 아래 항목을 확인하면 됨.
- `Proto` (TCP/UDP)
- `Local Address` (내 IP:포트)
- `Foreign Address` (상대 IP:포트)
- `State` (ESTABLISHED, LISTENING 등)
- `PID` (프로세스 ID)

3. 특정 포트만 빠르게 보고 싶다면 아래를 사용하면 됨.

```powershell
netstat -ano | findstr :443
```

4. 이해 포인트임.
- L4에서는 어떤 서비스 포트로 통신하는지가 핵심임.
- `ESTABLISHED`는 연결이 성립된 상태를 의미함.

### 예상 결과(예시)

```text
TCP    192.168.0.25:50123   142.250.xxx.xxx:443   ESTABLISHED   12345
TCP    0.0.0.0:135          0.0.0.0:0             LISTENING     980
UDP    0.0.0.0:5353         *:*                                 4567
```

`ESTABLISHED` 또는 `LISTENING` 행이 보이면 정상적으로 상태를 읽고 있는 것임.

---

## 실습 6. HTTP vs HTTPS 응답 헤더 비교 (L7 + 보안)

### 이 실습이 연결되는 이론

- Part 4.5 HTTP vs HTTPS
- Part 3.2 L7(Application)

> **반드시 `curl.exe`라고 확장자까지 입력해야 함.** PowerShell에서 `curl`은 `Invoke-WebRequest`의 **별칭(alias)** 이므로, `curl -I ...`를 그대로 입력하면 다음과 같은 오류가 발생함.
>
> ```text
> Invoke-WebRequest : Cannot process command because of one or more missing mandatory parameters: Uri.
> ```
>
> `curl.exe`로 입력해야 Windows에 내장된 진짜 curl이 실행됨.
{: .prompt-warning }

### Step by Step

1. HTTP 응답 헤더를 확인하면 됨.

```powershell
curl.exe -I http://github.com
```

2. HTTPS 응답 헤더를 확인하면 됨.

```powershell
curl.exe -I https://github.com
```

3. 결과를 비교하면 됨.
- HTTPS는 TLS를 사용하는 안전한 채널 위에서 통신함.
- 많은 사이트가 HTTP로 들어온 요청을 HTTPS로 **강제 이동(리다이렉트)** 시킴.

### 예상 결과(예시)

HTTP 요청 — `301`과 함께 `Location` 헤더로 HTTPS 주소를 알려줌.

```text
HTTP/1.1 301 Moved Permanently
Content-Length: 0
Location: https://github.com/
```

HTTPS 요청 — 최종 `200 OK`.

```text
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
```

`301` 응답의 `Location` 줄이 `https://`로 시작하면, 그 사이트가 평문 통신을 허용하지 않고 암호화된 채널로 옮겨 붙이고 있다는 뜻임.

> **모든 사이트가 리다이렉트하는 것은 아님.** 예를 들어 `http://example.com`은 `301`이 아니라 `200 OK`를 그대로 반환함. 리다이렉트 여부는 사이트 운영자의 설정에 달린 것이지 프로토콜의 규칙이 아님.
{: .prompt-info }

---

## 실습 7. 옆 사람 PC와 통신 확인하기 (L2 + L3 + 방화벽)

지금까지는 인터넷 반대편의 서버를 대상으로 했음. 이번에는 **같은 강의실, 같은 네트워크 안에 있는 옆 사람의 PC**와 직접 통신해 봄. 2인 1조로 진행하며, 서로를 **A(나)** 와 **B(옆 사람)** 로 부름.

### 이 실습이 연결되는 이론

- Part 2.3 LAN — 같은 LAN 안에서의 통신
- Part 3.2 L2(Data Link): MAC 주소 / L3(Network): IP 주소
- 1장 Part 3.1 기술적 보호대책 — 방화벽
- 1장 Part 5.6 예방통제(Preventive Control)

> **반드시 지킬 것**
> - 상대의 **동의를 얻고** 진행해야 함. 동의 없이 남의 PC를 조사하는 행위는 실습이 아니라 침해임.
> - 대상은 **같은 강의실의 실습용 PC**로 한정함. 공용 Wi-Fi에서 모르는 사람의 IP를 대상으로 삼으면 안 됨.
> - **실습이 끝나면 반드시 원상복구**해야 함(단계 6).
{: .prompt-warning }

### Step 1. 서로 같은 네트워크에 있는지 확인

A와 B가 각각 아래를 실행함.

```powershell
ipconfig | findstr /i "IPv4 게이트웨이 Gateway"
```

두 사람의 결과를 비교함.

- **기본 게이트웨이가 같으면** 같은 네트워크임. 실습을 진행할 수 있음.
- 게이트웨이가 다르면 서로 다른 네트워크에 붙어 있는 것임. 같은 Wi-Fi(또는 같은 유선 스위치)에 연결한 뒤 다시 확인해야 함.

```text
   IPv4 주소 . . . . . . . . . : 192.168.50.63     ← A
   기본 게이트웨이 . . . . . . : 192.168.50.1

   IPv4 주소 . . . . . . . . . : 192.168.50.57     ← B
   기본 게이트웨이 . . . . . . : 192.168.50.1      ← 같으면 OK
```

> **주소가 여러 개 나오는 것은 정상임.** VirtualBox·VMware·WSL·VPN을 설치했다면 가상 어댑터의 주소(`192.168.56.1`, `172.20.x.x` 등)가 함께 표시됨. 이때는 **기본 게이트웨이가 비어 있지 않은 줄**의 IPv4 주소를 골라야 함. 그것이 실제로 외부와 통신하는 어댑터임.
{: .prompt-info }

B는 자신의 **IPv4 주소**를 A에게 알려줌. 이후 이 주소를 `<B의 IP>`로 표기함.

### Step 2. 먼저 실패를 확인 (중요)

A가 B에게 ping을 보냄. **성공하는 것보다 실패하는 것이 정상임.**

```powershell
ping <B의 IP> -n 4
```

대부분 아래처럼 나옴.

```text
Pinging 192.168.50.57 with 32 bytes of data:
Request timed out.
Request timed out.
Request timed out.
Request timed out.

Ping statistics for 192.168.50.57:
    Packets: Sent = 4, Received = 4, Lost = 4 (100% loss),
```

> **왜 실패하는가**
> Windows Defender 방화벽은 기본적으로 **들어오는(inbound) ICMP Echo Request를 차단**함. 즉 B의 PC가 A의 ping을 받고도 **일부러 응답하지 않는 것**임. 네트워크가 끊긴 것도, 케이블이 빠진 것도 아님.
>
> 여기서 꼭 이해할 점은 **막고 있는 쪽이 B라는 사실**임. A가 자기 PC의 방화벽을 아무리 풀어도 결과는 바뀌지 않음. **B가 자기 방화벽을 열어야** A의 ping이 통함.
{: .prompt-tip }

### Step 3. B의 네트워크 프로필 확인

B가 아래를 실행함.

```powershell
Get-NetConnectionProfile | Select-Object Name, InterfaceAlias, NetworkCategory
```

```text
Name        InterfaceAlias NetworkCategory
----        -------------- ---------------
2401Infosec 2401Infosec    Public
```

- `Public`(공용) — 카페·공항 같은 신뢰할 수 없는 망으로 간주하여 **가장 엄격하게 차단**함. 기본값이 보통 이것임.
- `Private`(개인) — 집·회사 내부망으로 간주하여 일부 통신을 허용함.

프로필에 따라 적용되는 방화벽 규칙이 달라지므로, 어느 프로필인지 먼저 확인해야 함.

### Step 4. B가 방화벽에서 ICMP 인바운드를 허용

세 가지 방법 중 하나를 쓰면 됨. **방법 2를 권장함.**

#### (방법 1) GUI로 설정

1. `Windows + S` → `Windows 보안` 실행
2. **방화벽 및 네트워크 보호** → 아래쪽 **고급 설정** 클릭
3. 왼쪽에서 **인바운드 규칙** 선택
4. 목록에서 **파일 및 프린터 공유(에코 요청 - ICMPv4-In)** 를 찾음
5. 해당 항목을 우클릭 → **규칙 사용** 클릭 (영문판은 *Enable Rule*)
6. 같은 이름이 여러 개 보이면, **프로필 열이 Step 3에서 확인한 값(공용/개인)과 일치하는 것**을 켜야 함

#### (방법 2) PowerShell로 설정 — 권장

**관리자 권한 PowerShell**에서 실행함. (`Windows + S` → `PowerShell` → 우클릭 → **관리자 권한으로 실행**)

```powershell
Get-NetFirewallRule -Name "FPS-ICMP4-ERQ-In*" | Enable-NetFirewallRule
```

> 여기서 쓴 `FPS-ICMP4-ERQ-In`은 화면에 보이는 이름이 아니라 **규칙의 내부 식별자(Name)** 임. 화면 표시 이름(DisplayName)은 한글판에서 "파일 및 프린터 공유(에코 요청 - ICMPv4-In)", 영문판에서 "File and Printer Sharing (Echo Request - ICMPv4-In)"으로 다르지만, **내부 식별자는 언어와 무관하게 동일함**. 그래서 이 방법이 가장 확실함.
>
> 뒤에 `*`를 붙인 이유는 Windows 버전에 따라 `FPS-ICMP4-ERQ-In`, `FPS-ICMP4-ERQ-In-NoScope`, `FPS-ICMP4-ERQ-In-V2`처럼 **프로필별로 이름이 조금씩 다른 규칙이 여러 개 존재**하기 때문임. 와일드카드로 한 번에 처리하면 버전 차이에 영향을 받지 않음.
{: .prompt-info }

적용 여부는 아래로 확인함.

```powershell
Get-NetFirewallRule -Name "FPS-ICMP4-ERQ-In*" | Select-Object Name, Profile, Enabled
```

```text
Name                     Profile         Enabled
----                     -------         -------
FPS-ICMP4-ERQ-In         Private, Public    True
FPS-ICMP4-ERQ-In-V2      Any                True
FPS-ICMP4-ERQ-In-NoScope Domain             True
```

#### (방법 3) netsh로 새 규칙 추가

관리자 권한 PowerShell 또는 명령 프롬프트에서 실행함. 내장 규칙을 건드리지 않고 **실습 전용 규칙을 새로 만드는** 방식이라, 나중에 지우기가 쉬움.

```powershell
netsh advfirewall firewall add rule name="ICMPv4 Echo (Class Lab)" protocol=icmpv4:8,any dir=in action=allow
```

`protocol=icmpv4:8,any`에서 `8`이 **Echo Request(ping 요청)** 의 타입 번호임. 즉 ping 요청만 허용하고 다른 ICMP 유형은 건드리지 않음.

### Step 5. 다시 통신 확인

A가 다시 ping을 보냄.

```powershell
ping <B의 IP> -n 4
```

```text
Reply from 192.168.50.57: bytes=32 time<1ms TTL=128
Reply from 192.168.50.57: bytes=32 time<1ms TTL=128

Ping statistics for 192.168.50.57:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
```

`Reply from ...`이 보이면 성공임. 이어서 아래도 확인해 보면 좋음.

```powershell
Test-NetConnection -ComputerName <B의 IP>
```

```text
ComputerName           : 192.168.50.57
PingSucceeded          : True
PingReplyDetails (RTT) : 0 ms
```

마지막으로 **L2까지 내려가 확인**함. ping이 오간 뒤 A의 ARP 테이블에 B의 **MAC 주소**가 남아 있어야 함.

```powershell
arp -a | findstr <B의 IP>
```

```text
  192.168.50.57         20-1e-88-d4-e9-c8     dynamic
```

> 같은 LAN 안에서는 **IP 주소만으로 통신하지 않음.** 실제 프레임을 보내려면 상대의 MAC 주소가 필요하고, 그것을 알아내는 절차가 ARP임. 이 한 줄이 **L3(IP) ↔ L2(MAC) 대응**을 눈으로 보여줌.
{: .prompt-tip }

### Step 6. 원상복구 (반드시 수행)

실습이 끝나면 B는 열어 둔 규칙을 되돌림. 관리자 권한 PowerShell에서 실행함.

방법 2로 설정했다면,

```powershell
Get-NetFirewallRule -Name "FPS-ICMP4-ERQ-In*" | Disable-NetFirewallRule
```

방법 3으로 설정했다면,

```powershell
netsh advfirewall firewall delete rule name="ICMPv4 Echo (Class Lab)"
```

복구되었는지는 A가 다시 ping을 보내 **`Request timed out`으로 돌아오는지**로 확인함.

> **방화벽을 통째로 끄면 안 됨.**
> `netsh advfirewall set allprofiles state off` 같은 명령으로 방화벽 전체를 내리면 ping은 당연히 통하지만, 그 순간 **모든 포트가 함께 열림**. 이 실습에 필요한 것은 ICMP Echo Request 하나뿐임.
>
> 1장에서 배운 **최소 권한(least privilege)** 원칙이 그대로 적용됨. **필요한 것만, 필요한 동안만** 열어야 함. 실습용으로 연 규칙을 그대로 두고 다니는 것도 같은 이유로 좋지 않음.
{: .prompt-warning }

### 문제 해결

| 증상 | 원인 | 조치 |
|---|---|---|
| 방화벽을 열었는데도 `Request timed out` | 켠 규칙의 **프로필**이 현재 네트워크와 다름 | Step 3으로 돌아가 프로필을 확인하고, 해당 프로필의 규칙을 켬 |
| `Enable-NetFirewallRule`에서 접근 거부 오류 | 일반 권한으로 실행함 | PowerShell을 **관리자 권한으로 실행** |
| ping 자체가 `Destination host unreachable` | 서로 다른 네트워크에 있음 | Step 1에서 기본 게이트웨이가 같은지 확인 |
| B의 IP가 실습 도중 바뀜 | DHCP 임대 갱신 | B가 `ipconfig`로 주소를 다시 확인하여 알려줌 |
| 백신·보안 프로그램이 별도로 차단 | Windows 방화벽 외에 제3의 보안 제품이 동작 중 | 해당 제품의 방화벽 설정도 함께 확인 |

### 생각해 볼 점

- 서버 관리자가 ping 응답을 일부러 막는 이유는 무엇인가. (공격자에게 "여기 살아 있는 장비가 있다"는 사실을 알려주지 않기 위함임)
- 그렇다면 ping이 막혀 있을 때, 그 장비가 살아 있는지 확인할 다른 방법은 무엇이 있는가.
- 방화벽은 1장에서 배운 **예방통제**에 해당함. 이 실습에서 **탐지통제**에 해당하는 것은 무엇이겠는가.

---

## 한 번에 복습하기 (짧은 루틴)

아래 순서로 10분 복습 루틴을 진행하면 이론 연결이 빠르게 잡힘.

1. `ipconfig /all` (L3 주소 체계)
2. `ping 8.8.8.8 -n 4` (L3 연결성)
3. `tracert www.google.com` (L3 경로)
4. `nslookup openai.com` (L7 DNS)
5. `netstat -ano` (L4 포트/세션)
6. `curl.exe -I https://github.com` (L7 HTTP/HTTPS)
7. `ping <옆 사람 IP>` → `arp -a` (LAN 내 L3 ↔ L2 대응, 방화벽의 영향)

---



