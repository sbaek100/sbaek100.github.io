---
title: 네트워크 기초 8장 - 네트워크 기반 프로그램 활용 (진단 명령어)
date: 2026-03-09 21:00:00 +0900
categories:
  - 0.기초강의
  - 네트워크
tags:
  - ping
  - traceroute
  - netstat
  - tcpdump
  - 네트워크진단
pin:
mermaid: true
---

> **학습목표**
> 1. ping·tracert/traceroute로 연결성과 경로를 진단할 수 있다.
> 2. netstat·route로 연결 상태와 라우팅 테이블을 확인할 수 있다.
> 3. tcpdump로 패킷을 캡처하고, ifconfig/ipconfig로 인터페이스를 확인·설정할 수 있다.
> 4. Windows와 Linux에서 같은 목적의 명령이 어떻게 다른지 구분할 수 있다.
{: .prompt-info }

앞 장들에서 배운 개념 — IP·라우팅·포트·ARP — 은 명령어 도구를 통해 눈으로 확인하고 문제를 진단할 수 있습니다. 이 장은 이론보다 명령어와 문법 실습에 초점을 둡니다. 각 명령의 쓰임과 대표 옵션, 출력을 읽는 법을 익힌 뒤 마지막 실습에서 앞서 만든 네트워크에 직접 적용해 봅니다.

> **참고 — 관리자 권한과 운영체제** 일부 명령(tcpdump, route 변경, ifconfig 설정 등)은 관리자 권한이 필요합니다. Linux에서는 앞에 `sudo`를 붙이고, Windows에서는 관리자 권한 명령 프롬프트를 사용합니다. 명령 이름과 옵션은 운영체제마다 다르므로, 이 장에서는 Windows와 Linux를 함께 표기합니다.
{: .prompt-info }

## 1. ping

**ping**은 상대 호스트가 살아 있는지, 얼마나 빨리 응답하는지를 확인하는 가장 기본적인 도구입니다. ICMP 에코 요청을 보내고 에코 응답을 받아 왕복 시간(RTT)과 손실률을 알려 주며, 응답의 TTL 값으로 중간에 몇 개의 라우터를 거쳤는지 짐작할 수 있습니다.

```bash
# Windows: 4번 보내기, 크기 64바이트
ping -n 4 -l 64 192.168.20.1
# Linux: 4번 보내기, 크기 64바이트
ping -c 4 -s 64 192.168.20.1
# 결과 예시
Reply from 192.168.20.1: bytes=32 time=1ms TTL=254
```

| 기능 | Windows | Linux | 설명 |
|---|---|---|---|
| 횟수 지정 | `-n 4` | `-c 4` | 요청을 몇 번 보낼지 |
| 연속 전송 | `-t` | (기본) | 중지할 때까지 계속 전송 |
| 크기 지정 | `-l 64` | `-s 64` | 보낼 데이터 크기(바이트) |
| TTL 지정 | `-i 64` | `-t 64` | 패킷 수명(홉 한계) |

_표 8-1. ping의 주요 옵션_

ping의 출력은 한 줄 한 줄이 정보입니다. `Reply from ...`은 상대가 응답했다는 뜻, `time=1ms`는 왕복 시간(RTT)으로 작을수록 빠릅니다. `TTL=254`는 남은 패킷 수명으로, 출발 시 기본값(흔히 255·128·64)에서 거친 라우터 수만큼 줄어들어 도착한 값입니다(예: 254면 라우터를 한 번 거침). 끝의 `Lost = 0 (0% loss)`는 손실률로, 0%가 아니면 회선 품질·혼잡을 의심합니다. 응답이 없으면 `Request timed out`이 나옵니다.

## 2. tracert / traceroute

**tracert**(Windows)는 목적지까지 패킷이 거쳐 가는 경로(라우터 목록)를 추적합니다. TTL을 1부터 하나씩 늘려 보내면 각 라우터가 TTL 만료(Time Exceeded) 메시지를 돌려주는데, 이를 이용해 홉을 순서대로 알아냅니다. 어디서 통신이 끊기는지 찾을 때 유용합니다. **traceroute**(Linux/Unix)는 같은 원리이지만 기본 프로토콜이 다릅니다 — Windows tracert가 ICMP를 쓰는 반면, Linux traceroute는 기본적으로 UDP를 쓰고 옵션으로 ICMP(`-I`)나 TCP(`-T`)를 선택합니다.

```bash
# Windows: 이름 조회 없이(-d) 최대 15홉까지
tracert -d -h 15 192.168.20.1
# Linux: 이름 조회 없이(-n) 최대 15홉, ICMP 사용(-I)
traceroute -n -m 15 -I 192.168.20.1
```

| 구분 | tracert (Windows) | traceroute (Linux) |
|---|---|---|
| 기본 프로토콜 | ICMP Echo | UDP(옵션으로 ICMP·TCP) |
| 최대 홉 지정 | `-h` | `-m` |
| 이름 조회 생략 | `-d` | `-n` |

_표 8-2. tracert와 traceroute의 비교_

## 3. netstat

**netstat**(network statistics)는 현재 열려 있는 연결과 포트, 라우팅 테이블, 프로토콜 통계를 보여 줍니다. 어떤 프로그램이 어떤 포트로 통신하고 있는지 확인해 비정상 연결이나 백도어를 찾는 데도 쓰입니다.

```bash
# 활성 연결을 숫자로(-n) 모두(-a) 표시
netstat -an
# Windows: 연결을 사용하는 프로세스 번호(PID)까지
netstat -ano
# Linux: TCP·UDP 리스닝 포트와 프로그램
netstat -tulnp
```

| 옵션 | 의미 |
|---|---|
| `-a` | 모든 연결과 리스닝 포트를 표시 |
| `-n` | 이름 대신 숫자(IP·포트)로 표시(빠름) |
| `-r` | 라우팅 테이블을 표시(route와 유사) |
| `-s` | 프로토콜별 통계를 표시 |
| `-o`(Windows) / `-p`(Linux) | 연결을 사용하는 프로세스(PID·이름)를 함께 표시 |

_표 8-3. netstat의 주요 옵션_

## 4. route

**route**는 호스트의 라우팅 테이블을 확인하고 정적 경로를 직접 추가·삭제하는 명령입니다. 3장에서 배운 라우팅 테이블을 PC 수준에서 들여다보고 조작하는 도구입니다.

```bash
# 라우팅 테이블 보기
route print          # Windows
route -n             # Linux (또는 ip route)
# 정적 경로 추가 (192.168.30.0/24 -> 게이트웨이 192.168.10.254)
route add 192.168.30.0 mask 255.255.255.0 192.168.10.254   # Windows
sudo ip route add 192.168.30.0/24 via 192.168.10.254        # Linux
```

## 5. tcpdump (Linux)

**tcpdump**는 네트워크 인터페이스를 지나는 패킷을 캡처해 화면에 출력하는 명령줄 패킷 분석 도구입니다. 필터 표현식으로 원하는 트래픽만 골라 볼 수 있어 문제 진단·보안 분석에 널리 쓰이며, 그래픽 도구인 와이어샤크(Wireshark)와 같은 일을 터미널에서 합니다.

```bash
# eth0에서 호스트 192.168.20.1의 트래픽만, 이름 조회 없이(-n)
sudo tcpdump -i eth0 -n host 192.168.20.1
# 80번 포트(HTTP) 트래픽만 파일로 저장(-w)
sudo tcpdump -i eth0 tcp port 80 -w web.pcap
```

| 필터 표현식 | 의미 |
|---|---|
| `host 192.168.20.1` | 특정 호스트와 주고받는 트래픽 |
| `net 192.168.10.0/24` | 특정 네트워크 대역의 트래픽 |
| `tcp port 80` | TCP 80번(HTTP) 트래픽 |
| `src host A and dst port 53` | 출발지 A이면서 목적지 포트 53(조건 결합) |

_표 8-4. tcpdump 필터 표현식 예_

> **보안 유의 — 패킷 캡처와 권한** 패킷 캡처는 다른 사용자의 통신 내용까지 들여다볼 수 있으므로 관리자 권한이 필요하고, 함부로 쓰면 도청이 됩니다. 자신이 관리 권한을 가진 네트워크에서 진단·분석 목적으로만 사용해야 합니다.
{: .prompt-warning }

## 6. ifconfig / ipconfig

**ifconfig**(Linux)는 네트워크 인터페이스의 IP·MAC·상태를 확인·설정하는 명령으로, 최신 배포판에서는 더 기능이 풍부한 `ip` 명령(`ip addr`, `ip link`)으로 대체되는 추세입니다.

```bash
ifconfig                       # 모든 인터페이스 확인
sudo ifconfig eth0 up          # eth0 활성화
sudo ifconfig eth0 192.168.10.50 netmask 255.255.255.0   # IP 임시 설정
ip addr show                   # 대체 명령(권장)
```

**ipconfig**(Windows)는 인터페이스의 IP·서브넷·게이트웨이를 확인하는 명령으로 Linux의 ifconfig에 대응합니다. `/all`로 MAC·DNS까지 보고, DHCP 주소를 반납·재요청하거나 DNS 캐시를 비우는 데도 씁니다.

```bash
ipconfig              # 요약 정보
ipconfig /all         # MAC·DNS 등 상세 정보
ipconfig /release     # DHCP 주소 반납
ipconfig /renew       # DHCP 주소 재요청
ipconfig /flushdns    # DNS 캐시 비우기
```

| 기능 | Windows | Linux |
|---|---|---|
| 인터페이스 확인·설정 | `ipconfig` | `ifconfig` / `ip addr` |
| 경로 추적 | `tracert` | `traceroute` |
| DNS 캐시 비우기 | `ipconfig /flushdns` | `systemd-resolve --flush-caches` 등 |
| 패킷 캡처(CLI) | (Wireshark 등) | `tcpdump` |
| 연결·포트 확인 | `netstat -ano` | `netstat -tulnp` / `ss` |

_표 8-5. Windows와 Linux 명령의 대응_

## 7. 실습

> **실습 8-1. 명령어로 네트워크 진단하기**  
> **실습 목표** 앞서 구성한 네트워크에서 대표 진단 명령으로 연결성·경로·상태를 직접 확인한다.  
> **주소 계획** PC0 `192.168.10.1`(LAN1), PC1 `192.168.20.1`(LAN2) — 실습 3-1과 동일
{: .prompt-tip }

**실습 절차**

1. PC0의 `[Desktop] > [Command Prompt]`를 연다.
2. `ipconfig`로 자신의 IP·서브넷·게이트웨이를 확인하고, `ipconfig /all`로 MAC 주소까지 본다.
3. `ping 192.168.20.1`로 반대편 네트워크의 PC1과 연결성을 확인한다(응답의 time·TTL을 읽어 본다).
4. `tracert 192.168.20.1`로 R1 → R2를 거치는 경로(홉)를 확인한다.
5. `arp -a`로 학습된 IP–MAC 매핑을, `netstat -n`으로 활성 연결을 확인한다.

> **결과 확인** 각 명령의 출력에서 게이트웨이·경로(홉)·MAC 매핑을 읽어낼 수 있으면 완료. (패킷 트레이서의 명령 프롬프트는 ipconfig·ping·tracert·arp·netstat·nslookup·ssh·telnet 등을 지원한다.)  
> **계층 짚기** `ipconfig`(인터페이스)·`ping`과 `tracert`(네트워크 계층 진단)·`arp`(2↔3계층)·`netstat`(전송 계층 연결) — 명령 하나하나가 특정 계층을 들여다보는 창이다. 앞서 배운 계층 구조가 진단 명령으로 그대로 이어진다.
{: .prompt-tip }

---

이 장에서는 네트워크를 진단하고 관리하는 대표 명령어들을 문법과 함께 익히고, 앞서 만든 네트워크에 직접 적용해 보았습니다. 이로써 네트워크를 **이해하고(1~2장)**, **연결하고(3~5장)**, **지키고(6장)**, **운영·진단하는(7~8장)** 전 과정을 한 바퀴 돌았습니다. 여기서 다진 기초와 실습 경험은 이후 더 깊은 네트워크 보안과 운영을 배우는 든든한 토대가 될 것입니다.
