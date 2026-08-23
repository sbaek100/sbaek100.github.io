---
title: 네트워크 실무 3강 - 서비스와 진단 명령어 (따라하기 실습)
date: 2026-09-01 10:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - PacketTracer
  - DHCP
  - DNS
  - 진단명령어
pin:
mermaid: false
---

> **이 실습에서 하는 일**
> 서버 한 대에 DHCP·DNS·HTTP 서비스를 켜서, PC가 IP를 자동으로 받고(DHCP) 이름만으로 웹에 접속하는(DNS) 모습을 확인합니다. 이어서 ping·tracert·netstat·nslookup 같은 진단 명령어로 통신 상태를 직접 들여다봅니다.
{: .prompt-info }

## 실습 3-1. DHCP·DNS·웹 서비스 구성하기

> **실습 목표** 서버에 DHCP·DNS·HTTP를 구성해, PC가 IP를 자동으로 받고 도메인 이름으로 웹에 접속하게 한다.
> **주소 계획** 서버 `192.168.10.100 / 255.255.255.0`, 도메인 이름 `www.class.com`
{: .prompt-tip }

**실습 절차**

**1단계 — 장치 배치.** 스위치(2960) 1대에 서버(Server) 1대와 PC 2대를 연결합니다. 서버를 클릭 → `[Desktop] > [IP Configuration]`에서 고정 IP `192.168.10.100`, 마스크 `255.255.255.0`을 설정합니다. (서버는 자동으로 주소를 받으면 안 되므로 반드시 고정으로 둡니다.)

![](/assets/img/posts/2026-09-01-netintro-03b-lab-services-diagnostics-1786427894413.png)

**2단계 — DHCP 서비스 켜기.** 서버 → `[Services] > [DHCP]`로 이동합니다. `Service`를 `On`으로 바꾸고, 아래처럼 주소 풀을 설정한 뒤 `Save`(또는 `Add`)를 누릅니다.

| 항목 | 값 |
|---|---|
| Default Gateway | 192.168.10.1 |
| DNS Server | 192.168.10.100 |
| Start IP Address | 192.168.10.101 |
| Subnet Mask | 255.255.255.0 |

![](/assets/img/posts/2026-09-01-netintro-03b-lab-services-diagnostics-1786427937102.png)


**3단계 — DNS 서비스 켜기.** `[Services] > [DNS]`로 이동해 `Service`를 `On`으로 바꿉니다. `Name` 칸에 `www.class.com`, `Address` 칸에 `192.168.10.100`을 입력하고 `Add`를 누릅니다. 이렇게 등록한 한 줄을 **A 레코드**라고 합니다(이름 → IP 연결).

![](/assets/img/posts/2026-09-01-netintro-03b-lab-services-diagnostics-1786428001348.png)


**4단계 — HTTP 서비스 확인.** `[Services] > [HTTP]`가 기본으로 `On`인지 확인합니다. 기본 웹 페이지를 그대로 둡니다.

![](/assets/img/posts/2026-09-01-netintro-03b-lab-services-diagnostics-1786428032205.png)

**5단계 — PC를 DHCP로 전환.** PC0을 클릭 → `[Desktop] > [IP Configuration]`에서 `[Static]` 대신 **`[DHCP]`** 를 선택합니다. 잠시 뒤 `DHCP request successful`이 뜨고 IP가 자동으로 채워지면 성공입니다. PC1도 같은 방법으로 DHCP로 바꿉니다.

![](/assets/img/posts/2026-09-01-netintro-03b-lab-services-diagnostics-1786428064976.png)

```text
IP Address......: 192.168.10.101
Subnet Mask.....: 255.255.255.0
Default Gateway.: 192.168.10.1
DNS Server......: 192.168.10.100
```

**6단계 — 이름으로 웹 접속.** PC0 → `[Desktop] > [Web Browser]`에서 주소창에 `http://www.class.com`을 입력합니다. DNS가 이름을 IP(192.168.10.100)로 바꿔 주어 서버의 웹 페이지가 열리면 성공입니다.

![](/assets/img/posts/2026-09-01-netintro-03b-lab-services-diagnostics-1786428158388.png)

> **결과 확인** PC가 DHCP로 IP를 자동으로 받고(192.168.10.101~), 도메인 이름만으로 웹 페이지가 열리면 완료입니다. 사용자는 이름만 알면 되고, 그 뒤에서 DHCP·DNS·HTTP가 협력하는 것 — 이것이 응용 계층의 모습입니다.
{: .prompt-tip }

> **생각해 보기 — DNS가 없으면 어떻게 될까?**
> 6단계에서 웹이 열린 뒤, PC0에서 이번에는 주소창에 이름 대신 `http://192.168.10.100`을 직접 입력해 보세요. 똑같이 열릴 것입니다. 그렇다면 DNS는 왜 필요할까요? IP 주소 대신 이름을 쓰면 무엇이 편리한지, 그리고 서버의 IP가 나중에 바뀌었을 때 이름을 쓰던 사람과 IP를 외우던 사람 중 누가 더 편할지 생각해 보세요.
{: .prompt-info }

![](/assets/img/posts/2026-09-01-netintro-03b-lab-services-diagnostics-1786428211436.png)


---
## 실습 3-2. 진단 명령어로 상태 확인하기

> **실습 목표** 대표 진단 명령으로 자신의 설정·연결성·경로·이름 조회를 직접 확인한다.
{: .prompt-tip }

PC0의 `[Desktop] > [Command Prompt]`에서 아래 명령을 하나씩 실행하며 출력을 읽어 봅니다.

**1단계 — 내 설정 확인 (ipconfig).** 자신의 IP·마스크·게이트웨이를 봅니다. `/all`을 붙이면 MAC 주소와 DNS 서버까지 나옵니다.

```text
PC0> ipconfig /all

   Physical Address......: 0060.7005.1AA1
   IP Address............: 192.168.10.101
   Subnet Mask...........: 255.255.255.0
   Default Gateway.......: 192.168.10.1
   DNS Servers...........: 192.168.10.100
```

**2단계 — 연결성 확인 (ping).** 서버까지 통신되는지 확인합니다. 응답의 `time`(왕복 시간)과 `TTL`을 읽어 봅니다.

```text
PC0> ping 192.168.10.100
Reply from 192.168.10.100: bytes=32 time<1ms TTL=128
```

**3단계 — 이름 조회 확인 (nslookup).** 이름이 어떤 IP로 바뀌는지 DNS 서버에 직접 물어봅니다.

```text
PC0> nslookup www.class.com
Name:    www.class.com
Address: 192.168.10.100
```

**4단계 — 연결 목록 확인 (netstat).** 웹 브라우저로 서버에 접속한 상태에서 실행하면, 현재 열린 연결과 포트(웹=80)를 볼 수 있습니다.

```text
PC0> netstat -n
   Proto  Local Address        Foreign Address      State
   TCP    192.168.10.101:1025  192.168.10.100:80    ESTABLISHED
```

> **결과 확인** 각 명령의 출력에서 내 IP·게이트웨이(ipconfig), 왕복 시간(ping), 이름→IP 변환(nslookup), 열린 연결과 포트(netstat)를 읽어낼 수 있으면 완료입니다. 명령 하나하나가 특정 계층을 들여다보는 ‘창’입니다.
{: .prompt-tip }

## 잘 안 될 때 점검 순서

| 증상 | 점검할 것 |
|---|---|
| PC가 DHCP로 IP를 못 받음 | 서버 IP가 고정(192.168.10.100)인지, DHCP Service가 `On`인지, PC-서버가 같은 스위치에 연결됐는지 |
| 이름으로 웹이 안 열림 | DNS Service가 `On`이고 A 레코드(www.class.com→100)가 등록됐는지, PC의 DNS 서버가 192.168.10.100인지 |
| nslookup 실패 | PC의 DNS 서버 주소가 비어 있지 않은지(ipconfig /all로 확인) |

---

이 실습으로 서버가 제공하는 핵심 서비스(DHCP·DNS·HTTP)를 직접 켜 보고, 진단 명령으로 통신을 확인했습니다. 다음 편([3강 실습 문제])에서는 대회 문제지 형식으로 서비스 구성 과제에 도전합니다.
