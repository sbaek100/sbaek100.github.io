---
title: 네트워크 기초 3장 - 라우팅 ① 개념·정적·동적 라우팅·NAT
date: 2026-03-09 13:00:00 +0900
categories:
  - 0.기초강의
  - 네트워크
tags:
  - 라우팅
  - 정적라우팅
  - RIP
  - OSPF
  - BGP
  - NAT
pin:
mermaid: true
---

> **학습목표 (3장 전체)**
> 1. 라우팅의 기본 개념과 라우팅 테이블·홉·메트릭의 의미를 설명할 수 있다.
> 2. 정적 라우팅과 동적 라우팅의 차이와 장단점을 비교할 수 있다.
> 3. AS, IGP, EGP의 관계와 Routed 프로토콜·Routing 프로토콜의 차이를 구분할 수 있다.
> 4. 거리 벡터·링크 상태·경로 벡터 라우팅 알고리즘의 원리와 차이를 설명할 수 있다.
> 5. RIP·OSPF·BGP 등 주요 라우팅 프로토콜의 특징을 설명할 수 있다.
{: .prompt-info }

2장에서 IP 주소로 목적지를 ‘지정’하는 법을 배웠다면, 3장에서는 그 목적지까지 패킷을 실제로 ‘어떻게 보내는가’, 즉 **라우팅**을 다룹니다. 수많은 네트워크가 얽힌 인터넷에서 라우터가 최적의 경로를 찾아내는 원리와, 이를 구현한 대표 프로토콜을 살펴봅니다.

## 1. 라우팅의 기본 개념

라우팅(Routing)이란 출발지에서 목적지까지 패킷이 지나갈 경로를 결정하는 과정입니다. 이 일을 담당하는 장비가 라우터(Router)이며, 라우터는 자신이 가진 **라우팅 테이블**(경로표)을 참고해 패킷을 다음 라우터, 즉 **넥스트 홉**(Next Hop)으로 넘깁니다. 패킷이 라우터를 하나 지날 때마다 홉(Hop)이 하나씩 늘어납니다. 라우터가 여러 경로 가운데 하나를 고를 때 기준이 되는 값을 **메트릭**(Metric)이라 하며, 홉 수·대역폭·지연 등을 사용해 값이 작을수록 선호합니다.

| 항목 | 의미 |
|---|---|
| 목적지 네트워크 | 패킷이 향하는 목적지 네트워크 주소 |
| 서브넷 마스크/프리픽스 | 네트워크 부분의 길이 |
| 넥스트 홉 | 목적지로 가기 위해 패킷을 넘길 다음 라우터 |
| 출력 인터페이스 | 패킷을 내보낼 라우터의 포트 |
| 메트릭 | 이 경로의 비용(작을수록 선호) |

_표 3-1. 라우팅 테이블의 주요 구성 항목_

> **참고 — 최장 일치(Longest Prefix Match)와 기본 경로** 하나의 목적지가 라우팅 테이블의 여러 항목과 동시에 겹칠 수 있습니다. 이때 라우터는 프리픽스가 가장 긴(가장 구체적인) 항목을 선택합니다. 예컨대 10.1.1.0/24와 10.0.0.0/8이 모두 있으면 /24를 따릅니다. 어느 항목에도 맞지 않으면 기본 경로(Default Route, 0.0.0.0/0)로 보내고, 그마저 없으면 패킷을 폐기합니다.
{: .prompt-info }

## 2. 정적 라우팅과 동적 라우팅

라우팅 테이블을 채우는 방식은 두 가지입니다. **정적 라우팅**은 관리자가 경로를 하나하나 수동으로 입력하는 방식으로, 설정이 명확하고 추가 트래픽이 없어 소규모·단순한 망에 적합하지만 회선이 끊기거나 구조가 바뀌어도 스스로 대응하지 못합니다. **동적 라우팅**은 라우터들이 라우팅 프로토콜로 경로 정보를 자동으로 주고받으며 테이블을 갱신하는 방식으로, 망이 크거나 변화가 잦아도 스스로 최적 경로를 다시 찾지만 정보 교환에 따른 부담이 있습니다.

| 구분 | 정적 라우팅 | 동적 라우팅 |
|---|---|---|
| 경로 설정 | 관리자가 수동 입력 | 프로토콜이 자동 학습·갱신 |
| 변화 대응 | 불가(수동 수정 필요) | 자동 재계산 |
| 부하 | 없음 | 정보 교환 오버헤드 존재 |
| 적합 환경 | 소규모·단순·안정적인 망 | 대규모·변화 잦은 망 |

_표 3-2. 정적 라우팅과 동적 라우팅의 비교_

한 목적지에 대해 서로 다른 출처(정적, RIP, OSPF 등)의 경로가 동시에 존재하면 무엇을 믿어야 할까요? 라우터는 경로 출처마다 신뢰도 순위인 **관리 거리**(AD, Administrative Distance)를 두고 값이 작은 쪽을 테이블에 올립니다(직접 연결=0, 정적=1, eBGP=20, EIGRP=90, OSPF=110, RIP=120).

## 3. 라우팅 프로토콜의 분류

라우팅 프로토콜은 라우터들이 서로 경로 정보를 교환해 테이블을 자동 구성·유지하도록 하는 규칙입니다. **AS**(Autonomous System, 자율 시스템)는 하나의 관리 주체(ISP·대기업)가 통제하는 라우터·네트워크의 집합으로, 전 세계에서 유일한 번호(ASN)를 받습니다. 이 구분에 따라 프로토콜도 AS 내부에서 동작하는 **IGP**(RIP·OSPF·EIGRP)와 AS 사이에서 동작하는 **EGP**(BGP)로 나뉩니다.

이름이 비슷해 자주 혼동되는 것이 Routed 프로토콜과 Routing 프로토콜입니다. **Routed 프로토콜**은 라우터를 건너 운반되는 데이터(IP·IPX) — ‘운반되는 화물’ — 이고, **Routing 프로토콜**은 그 화물이 갈 길을 정하는 내비게이션(RIP·OSPF·BGP)입니다.

> **실습 3-1. 정적 라우팅 — 두 네트워크 연결하기**  
> **실습 목표** 라우터 두 대로 서로 다른 두 네트워크를 잇고, 정적 경로를 입력해 통신시킨다.  
> **주소 계획** LAN1 `192.168.10.0/24`(R1 .254), LAN2 `192.168.20.0/24`(R2 .254), R1–R2 구간 `10.0.0.0/30`(R1 .1, R2 .2)
{: .prompt-tip }

**실습 절차**

1. 장비를 배치한다. 팔레트에서 Router `2911` 2대, Switch `2960` 2대, PC 2대를 드래그해 `PC0–SW1–R1–R2–SW2–PC1` 순서로 놓는다.
2. 케이블로 연결한다. `Copper Straight-Through`로 PC0↔SW1, SW1↔R1(G0/0), R2(G0/0)↔SW2, SW2↔PC1을 연결한다. R1–R2 구간은 라우터 직결이라 원칙은 Cross-Over이지만, 패킷트레이서는 Auto-MDIX를 지원해 Straight-Through로도 인식된다. R1의 G0/1을 R2의 G0/1에 연결한다. 양쪽 포트에 초록 점(링크 업)이 뜨는지 확인한다.
3. PC의 IP를 설정한다. PC0 → `Desktop > IP Configuration` → Static: IP `192.168.10.1`, Mask `255.255.255.0`, Gateway `192.168.10.254`. PC1은 `192.168.20.1 / 255.255.255.0`, 게이트웨이 `192.168.20.254`.
4. R1 콘솔에 들어간다. R1 클릭 → `CLI` 탭 → Enter로 `Router>`를 띄운 뒤 아래를 입력해 전역 설정 모드로 들어간다(설정 저장 메시지가 나오면 No).

    ```text
    enable
    configure terminal
    hostname R1
    ```

5. `R1(config)#`에서 LAN 쪽과 WAN 쪽 인터페이스를 설정하고 상태를 확인한다.

    ```text
    interface g0/0
    ip address 192.168.10.254 255.255.255.0
    no shutdown
    exit
    interface g0/1
    ip address 10.0.0.1 255.255.255.252
    no shutdown
    exit
    ```
    이어서 `show ip interface brief`로 두 인터페이스의 Status·Protocol이 모두 `up`인지 확인한다.

6. R2도 같은 절차를 반복한다.

    ```text
    enable
    configure terminal
    hostname R2
    interface g0/0
    ip address 192.168.20.254 255.255.255.0
    no shutdown
    exit
    interface g0/1
    ip address 10.0.0.2 255.255.255.252
    no shutdown
    exit
    ```

7. (선택) 정적 경로를 넣기 전, PC0에서 `ping 192.168.20.1`을 하면 `Request timed out`으로 실패한다. R1·R2가 아직 상대편 LAN으로 가는 길을 모르기 때문이다.
8. 각 라우터의 `(config)#`에서 상대편 LAN으로 가는 정적 경로를 넣는다. 명령 구조는 `ip route [목적지 네트워크] [서브넷마스크] [다음 홉 IP]`이다.

    ```text
    R1: ip route 192.168.20.0 255.255.255.0 10.0.0.2
    R2: ip route 192.168.10.0 255.255.255.0 10.0.0.1
    ```

9. `R1#`, `R2#`에서 `show ip route`로 경로가 `S 192.168.20.0/24 [1/0] via 10.0.0.2` 형태로 나타나는지 확인하고, PC0에서 `ping 192.168.20.1`, PC1에서 `ping 192.168.10.1`로 양방향 통신을 확인한다(처음 Request Timed Out은 라우팅에 시간이 걸리기 때문).

> **결과 확인** 정적 경로 전에는 ping이 실패하고, 넣은 뒤에는 4개 응답이 모두 성공하면 완료. 실패 시 ① PC의 IP·마스크·게이트웨이, ② 인터페이스 `no shutdown`(Status up), ③ `ip route`의 목적지·마스크·다음 홉을 순서대로 점검한다.  
> **계층 짚기** 라우터는 3계층(IP)에서 서로 다른 네트워크를 잇는다. 정적 경로는 관리자가 길을 직접 알려 주는 방식이다.
{: .prompt-tip }

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-03-09-netintro-c3-01.png" alt="토폴로지 구성" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-02.png" alt="케이블 연결" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-03.png" alt="PC IP 설정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-04.png" alt="R1 CLI 진입" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-05.png" alt="R1 인터페이스 설정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-06.png" alt="show ip interface brief" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-07.png" alt="R2 설정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-08.png" alt="R2 인터페이스 확인" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-09.png" alt="정적 경로 전 ping 실패" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-10.png" alt="정적 경로 입력" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-11.png" alt="show ip route" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-12.png" alt="ping 성공" style="max-width:24%">
</div>

## 4. 유니캐스트 라우팅

유니캐스트 라우팅은 하나의 출발지에서 하나의 목적지로 패킷을 전달하는, 가장 기본이 되는 라우팅입니다. 라우터는 전체 네트워크를 노드(라우터)와 링크로 이루어진 하나의 그래프로 보고, 각 링크에 매겨진 비용을 바탕으로 목적지까지의 최소 비용 경로를 계산합니다.

![그림 3-3. 유니캐스트 라우팅](/assets/img/posts/2026-03-09-netintro-c3-13.jpeg)
_그림 3-3. 유니캐스트 라우팅_

> **용어 참고 — 전달 방식** 유니캐스트(1:1)와 달리, 하나가 그룹 전체에 보내는 것을 멀티캐스트, 같은 주소를 가진 여러 대상 중 가장 가까운 하나에 보내는 것을 애니캐스트라 합니다.
{: .prompt-info }

### (1) 라우팅 알고리즘 3종

- **거리 벡터(Distance Vector)**: 각 라우터가 ‘목적지까지의 거리와 방향’만 알고, 이웃과 주기적으로 라우팅 테이블 전체를 주고받습니다(벨만-포드 기반). 구현이 단순하지만 전체 지도를 모른 채 이웃의 정보에만 의존하는 ‘소문에 의한 라우팅’이라, 수렴이 느리고 무한 카운트·루프가 생길 수 있습니다(홉 수 제한·스플릿 호라이즌·홀드다운·라우트 포이즈닝으로 완화). 대표: RIP, IGRP.
- **링크 상태(Link State)**: 각 라우터가 자신에 연결된 링크 상태(LSA)를 전체에 퍼뜨려(플러딩) 모두가 동일한 전체 지도를 갖고, 각자 다익스트라 최단 경로(SPF)로 계산합니다. 수렴이 빠르고 루프가 잘 생기지 않지만 계산·메모리 부담이 큽니다. 대표: OSPF, IS-IS.
- **경로 벡터(Path Vector)**: 목적지까지 거쳐 온 AS 경로 전체(AS-Path)를 함께 전달하고, 자기 AS가 이미 들어 있으면 폐기해 루프를 막으며 정책에 따라 경로를 고릅니다. AS 사이 라우팅에 적합. 대표: BGP.

한편 2계층에서 여러 스위치가 이중 연결되어 루프가 생기면 브로드캐스트 폭풍이 발생하는데, **스패닝 트리(STP)** 가 논리적으로 하나의 트리만 남기고 나머지를 차단해 루프를 없앱니다. 경로 벡터가 3계층 AS 루프를 막듯, 스패닝 트리는 2계층 스위치 루프를 막습니다.

| 구분 | 거리 벡터 | 링크 상태 | 경로 벡터 |
|---|---|---|---|
| 아는 범위 | 이웃까지의 거리·방향 | 전체 네트워크 지도 | 목적지까지의 AS 경로 |
| 교환 내용 | 라우팅 테이블 전체 | 링크 상태(LSA) | 경로(AS-Path)와 속성 |
| 알고리즘 | 벨만-포드 | 다익스트라(SPF) | 정책 기반 선택 |
| 수렴 속도 | 느림 | 빠름 | 느리지만 안정적 |
| 대표 예 | RIP | OSPF | BGP |

_표 3-6. 라우팅 알고리즘 3종의 비교_

### (2) 주요 라우팅 프로토콜

**IGP(AS 내부)** — **RIP**는 가장 오래된 거리 벡터로 홉 수만 메트릭으로 쓰고 30초마다 테이블 전체를 브로드캐스트하며(UDP 520), 최대 홉이 15로 제한되어 소규모 망에만 적합합니다. **RIPv2**는 서브넷 마스크를 함께 전달해 VLSM·CIDR을 지원하고 멀티캐스트(224.0.0.9)·인증을 추가했습니다. **IGRP/EIGRP**는 시스코 계열로 대역폭·지연을 조합한 복합 메트릭을 쓰며, 특히 EIGRP는 DUAL 알고리즘으로 빠르게 수렴하고 변화 부분만 갱신합니다. **OSPF**는 대표적 링크 상태 프로토콜로 다익스트라 SPF와 대역폭 기반 비용(Cost)을 쓰고, 네트워크를 영역(Area)으로 나눠 확장성이 뛰어나며 멀티캐스트(224.0.0.5/6)·IP 프로토콜 89로 동작합니다.

| 프로토콜 | 알고리즘 | 메트릭 | 특징 |
|---|---|---|---|
| RIP / RIPv2 | 거리 벡터 | 홉 수(최대 15) | 단순, 소규모망 |
| IGRP / EIGRP | 거리 벡터(하이브리드) | 복합(대역폭·지연 등) | 시스코 계열, 빠른 수렴 |
| OSPF | 링크 상태 | 비용(대역폭 기반) | 표준, 대규모망, 영역 구성 |

_표 3-7. 주요 인트라 도메인(IGP) 프로토콜 비교_

**EGP(AS 사이)** — **BGP4**는 서로 다른 AS를 잇는 인터 도메인 라우팅의 사실상 표준으로, 오늘날 인터넷 전체를 지탱하는 경로 벡터 프로토콜입니다. 신뢰성을 위해 TCP 179번 위에서 동작하고, AS-Path로 루프를 방지하며 여러 경로 속성을 바탕으로 정책적으로 경로를 고릅니다(최적보다 안정성·정책 우선).

> **보안 유의 — BGP 하이재킹** BGP는 기본적으로 상대가 광고하는 경로를 신뢰합니다. 이를 악용해 공격자가 특정 대역을 자기 것이라 거짓 광고하면 전 세계 트래픽이 엉뚱한 곳으로 흘러가는 BGP 하이재킹(경로 탈취)이 일어납니다. 대응책으로 경로 원본을 검증하는 RPKI, 필터링, 피어 인증 등이 쓰입니다.
{: .prompt-warning }

> **실습 3-2. 동적 라우팅 — RIP로 경로 자동 학습**  
> **실습 목표** 정적 경로를 지우고 RIP를 설정해, 라우터끼리 경로를 자동으로 주고받게 한다.
{: .prompt-tip }

**실습 절차**

1. R1의 CLI에서 실습 3-1의 정적 경로를 지운다. `enable → configure terminal`로 들어간 뒤 `no ip route 192.168.20.0 255.255.255.0 10.0.0.2`. R2도 `no ip route 192.168.10.0 255.255.255.0 10.0.0.1`.
2. PC0에서 `ping 192.168.20.1`이 다시 `Request timed out`으로 실패하는지 확인한다.
3. `R1(config)#`에서 RIP를 설정한다. `network` 뒤에는 마스크가 아니라 그 인터페이스가 속한 주요(major) 네트워크만 적고, `version 2`로 클래스리스 광고를 지정한다.

    ```text
    router rip
    version 2
    network 192.168.10.0
    network 10.0.0.0
    exit
    ```

4. R2도 같은 방식으로 `router rip → version 2 → network 192.168.20.0 → network 10.0.0.0 → exit`.
5. RIP는 기본 30초 주기이므로 30~60초 기다린 뒤 `R1#`에서 `show ip route`로 `R`로 표시된 `192.168.20.0/24`가 학습됐는지, `show ip protocols`로 RIP 동작을 확인한다.
6. PC0에서 `ping 192.168.20.1`, PC1에서 `ping 192.168.10.1`로 양방향 통신을 확인한다.

> **결과 확인** 정적 경로 없이 RIP 학습 경로로 ping이 성공하면 완료. `R` 경로가 안 보이면 ① 두 라우터 모두 `network`에 WAN(10.0.0.0)을 넣었는지, ② 양쪽 `version 2`인지, ③ 정적 경로를 완전히 지웠는지(남아 있으면 AD가 낮은 정적이 우선) 확인한다.  
> **계층 짚기** 동적 라우팅은 망이 바뀌어도 스스로 대응한다. 거리 벡터(RIP)의 실제 모습이다.
{: .prompt-tip }

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-03-09-netintro-c3-14.png" alt="정적 경로 삭제" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-15.png" alt="RIP 설정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-16.png" alt="show ip route R 경로" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-17.png" alt="ping 성공" style="max-width:24%">
</div>

> **실습 3-3. 동적 라우팅 — OSPF로 최단 경로 계산**  
> **실습 목표** RIP를 OSPF로 바꿔, 링크 상태 방식이 대역폭 기반 비용(Cost)으로 최단 경로를 계산하는 것을 확인한다.
{: .prompt-tip }

**실습 절차**

1. R1의 CLI에서 `no router rip`으로 RIP를 지운다. R2도 `no router rip`.
2. PC0에서 `ping 192.168.20.1`이 다시 실패하는지 확인한다.
3. `R1(config)#`에서 OSPF를 설정한다. 마스크가 아니라 **와일드카드 마스크**를 쓰는 데 유의한다(255.255.255.0 → `0.0.0.255`, /30인 255.255.255.252 → `0.0.0.3`).

    ```text
    router ospf 1
    network 192.168.10.0 0.0.0.255 area 0
    network 10.0.0.0 0.0.0.3 area 0
    exit
    ```

4. R2도 같은 방식으로 `router ospf 1 → network 192.168.20.0 0.0.0.255 area 0 → network 10.0.0.0 0.0.0.3 area 0 → exit`. 양쪽 모두 `area 0`(백본)으로 맞춰야 이웃 관계가 맺어진다.
5. `R1#`에서 `show ip ospf neighbor`로 R2와 이웃 관계(State가 `FULL`)를 확인한다.
6. `show ip route`에서 `O`로 표시된 OSPF 경로가 학습됐는지 확인한다.
7. PC0↔PC1 양방향 ping을 확인한다.

> **결과 확인** 라우팅 테이블에 `O` 경로가 나타나고 ping이 성공하면 완료. RIP(홉 수)와 달리 OSPF는 대역폭 기반 비용으로 경로를 고른다. 이웃이 `FULL`이 아니면 ① area 번호 일치, ② 와일드카드 마스크가 인터페이스 대역과 일치, ③ 인터페이스 up 상태를 점검한다.  
> **계층 짚기** OSPF는 링크 상태 라우팅(다익스트라 SPF)의 실제 모습이다.
{: .prompt-tip }

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-03-09-netintro-c3-18.png" alt="RIP 삭제" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-19.png" alt="OSPF 설정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-20.png" alt="show ip ospf neighbor FULL" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-21.png" alt="show ip route O 경로" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-22.png" alt="ping 성공" style="max-width:24%">
</div>

> **실습 3-4. NAT — 사설망을 공인망으로 내보내기 (PAT)**  
> **실습 목표** 사설 IP를 쓰는 내부망이 하나의 공인 IP를 포트로 나눠 쓰며 외부로 나가는 PAT(NAT 과부하)를 구성한다. (NAT 개념은 2장에서 다뤘다.)  
> **주소 계획** 내부(사설) `192.168.10.0/24`(R1 inside .254), 공인 구간 `203.0.113.0/30`(R1 outside .1, R2 .2)
{: .prompt-tip }

**실습 절차**

1. 실습 3-1의 토폴로지를 쓰되, R1의 LAN을 ‘내부(사설)’, R1–R2 구간을 ‘공인 구간’으로 바꾼다. 이전 라우팅 프로토콜을 정리한다: `no router ospf 1`(R1·R2 각각).
2. R1–R2 구간 IP를 공인 대역으로 다시 설정한다.

    ```text
    R1(config)# interface g0/1
    ip address 203.0.113.1 255.255.255.252
    exit
    R2(config)# interface g0/1
    ip address 203.0.113.2 255.255.255.252
    exit
    ```

3. R1에서 남은 정적 경로를 지우고, 외부로 나가는 기본 경로(디폴트 루트)를 넣는다.

    ```text
    no ip route 192.168.20.0 255.255.255.0 10.0.0.2
    ip route 0.0.0.0 0.0.0.0 203.0.113.2
    ```

4. R1 인터페이스에 NAT 역할을 지정한다. 내부망 쪽이 `inside`, 공인망 쪽이 `outside`다.

    ```text
    interface g0/0
    ip nat inside
    exit
    interface g0/1
    ip nat outside
    exit
    ```

5. 변환 대상과 PAT(과부하)를 설정한다. `overload`가 하나의 공인 IP를 포트로 나눠 여러 내부 호스트가 함께 쓴다는 뜻이다.

    ```text
    access-list 1 permit 192.168.10.0 0.0.0.255
    ip nat inside source list 1 interface g0/1 overload
    ```

6. 내부 PC0에서 `ping 203.0.113.2`로 R2까지 확인한 뒤, `ping 192.168.20.1`로 R2 뒤의 외부 호스트까지 NAT를 거쳐 통신되는지 확인한다.
7. `R1#`에서 `show ip nat translations`로 사설 IP가 공인 IP·포트로 바뀐 매핑을 확인한다.

> **결과 확인** 변환 테이블에 `Inside local(192.168.10.x:포트) ↔ Inside global(203.0.113.1:포트)` 매핑이 보이고 외부 통신이 되면 완료. 매핑이 비면 ① access-list가 내부 대역을 포함하는지, ② `ip nat inside/outside`가 올바른 인터페이스에 적용됐는지, ③ overload 명령의 interface 이름이 outside와 일치하는지 확인한다.  
> **계층 짚기** 여러 사설 IP가 하나의 공인 IP를 포트로 나눠 쓰는 것이 PAT다. IPv4 주소 고갈을 늦춘 기술(2장)의 실제 동작을 확인했다.
{: .prompt-tip }

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-03-09-netintro-c3-23.png" alt="공인 구간 IP 설정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-24.png" alt="NAT inside/outside 지정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-25.png" alt="PAT overload 설정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-26.png" alt="show ip nat translations" style="max-width:24%">
</div>

---

여기까지 라우팅의 개념과 정적·동적 라우팅, NAT를 실습으로 확인했습니다. 다음 편에서는 라우터 자체를 지키는 **라우터 보안** — 모드·암호·ACL·불필요 서비스 제거 — 을 다룹니다.
