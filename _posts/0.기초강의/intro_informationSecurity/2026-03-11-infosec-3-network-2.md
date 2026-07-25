---
title: "2장 ② Windows 11 네트워크 기초 실습"
date: 2026-03-11 10:30:00 +0900
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
pin: false
math: false
mermaid: true
---

# Windows 11 네트워크 기초 실습 (Step by Step)

이 문서는 `2026-03-11-infosec-2-network.md` 이론을 Windows 11에서 직접 확인하는 실습 가이드임.  
모든 실습은 조회/관찰 중심이며, 외부 서비스에 공격성 트래픽을 보내지 않음.

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
- IP는 되는데 도메인은 실패: DNS 문제일 가능성이 있함.
- 둘 다 실패: 네트워크 연결 또는 방화벽 이슈일 수 있음.

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
Server:  dns.google
Address: 8.8.8.8

Non-authoritative answer:
Name:    www.example.com
Address: 93.184.216.34
```

`Name`과 `Address`가 함께 나오면 이름 해석이 성공한 것임.

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

### Step by Step

1. HTTP 응답 헤더를 확인하면 됨.

```powershell
curl -I http://example.com
```

2. HTTPS 응답 헤더를 확인하면 됨.

```powershell
curl -I https://example.com
```

3. 결과를 비교하면 됨.
- HTTPS는 TLS를 사용하는 안전한 채널 위에서 통신함.
- 사이트에 따라 HTTP는 HTTPS로 리다이렉트될 수 있음.

### 예상 결과(예시)

```text
HTTP/1.1 301 Moved Permanently
Location: https://example.com/
```

```text
HTTP/1.1 200 OK
Content-Type: text/html
```

HTTP는 `301/302` 리다이렉트가 자주 보이고, HTTPS는 최종 `200 OK` 응답이 보이는 경우가 많함.

---

## 한 번에 복습하기 (짧은 루틴)

아래 순서로 10분 복습 루틴을 진행하면 이론 연결이 빠르게 잡힙니다.

1. `ipconfig /all` (L3 주소 체계)
2. `ping 8.8.8.8 -n 4` (L3 연결성)
3. `tracert www.google.com` (L3 경로)
4. `nslookup openai.com` (L7 DNS)
5. `netstat -ano` (L4 포트/세션)
6. `curl -I https://example.com` (L7 HTTP/HTTPS)

---



