---
title: 네트워크 기초 5장 - VLAN (가상 랜)
date: 2026-03-09 16:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - VLAN
  - 트렁크
  - 802.1Q
  - VLAN간라우팅
  - 브로드캐스트도메인
pin:
mermaid: true
---

> **학습목표**
> 1. VLAN의 개념과 등장 배경을 설명할 수 있다.
> 2. VLAN의 특징과 장단점을 설명할 수 있다.
> 3. VLAN을 나누는 다섯 가지 기준(MAC·네트워크 주소·포트·프로토콜·멀티캐스트 IP)을 구분할 수 있다.
> 4. VLAN이 브로드캐스트 도메인 분리와 보안에 어떤 이점을 주는지 설명할 수 있다.
{: .prompt-info }

4장에서 스위치는 충돌 도메인은 나누지만 브로드캐스트 도메인은 나누지 못한다고 배웠습니다. **VLAN**은 바로 이 한계를 소프트웨어적으로 해결하는 기술입니다. 물리적으로는 하나의 스위치를, 논리적으로는 여러 개의 독립된 네트워크로 나누어 쓰는 방법을 이 장에서 살펴봅니다.

## 1. 개요

**VLAN(Virtual LAN, 가상 랜)** 은 하나의 물리적 스위치를 논리적으로 여러 개의 LAN으로 나누는 기술입니다. 장비가 어디에 물리적으로 연결되어 있느냐와 상관없이, 관리자가 정한 기준에 따라 논리적으로 그룹을 묶습니다. 같은 스위치에 꽂혀 있어도 서로 다른 VLAN에 속하면 직접 통신할 수 없고, 반대로 다른 층·다른 스위치에 있어도 같은 VLAN이면 하나의 LAN처럼 통신할 수 있습니다.

VLAN이 등장한 배경은 **브로드캐스트 도메인 문제**에 있습니다. 일반 스위치는 전체가 하나의 브로드캐스트 도메인이라, 연결된 장비가 많아질수록 브로드캐스트 트래픽이 늘어 성능이 떨어지고 관리·보안도 어려워집니다. VLAN은 이 하나의 브로드캐스트 도메인을 논리적으로 여러 개로 쪼개어 문제를 해결합니다. 단, VLAN으로 나뉜 서로 다른 그룹끼리 통신하려면 3계층 장비인 라우터나 L3 스위치를 거쳐야 하며(VLAN 간 라우팅), 여러 스위치에 걸쳐 같은 VLAN을 확장하려면 프레임에 소속 VLAN을 표시하는 태그를 붙이는 **IEEE 802.1Q** 방식을 쓰고, 이 태그가 실려 여러 VLAN이 함께 지나가는 스위치 간 통로를 **트렁크(Trunk)** 포트라 합니다.

## 2. 특징과 장단점

| 장점 | 설명 |
|---|---|
| 브로드캐스트 트래픽 감소 | 브로드캐스트 도메인이 작아져 불필요한 트래픽이 줄고 성능 향상 |
| 보안 향상 | VLAN 간 통신이 기본 차단되어 민감 부서를 논리적으로 격리 |
| 유연한 관리 | 장비를 물리적으로 옮기지 않고 설정만으로 그룹을 재구성 |
| 비용 절감 | 스위치 한 대로 여러 개의 논리적 LAN을 구성 |

_표 5-1. VLAN의 장점_

3층짜리 회사가 층마다 스위치를 두고 영업·개발·회계 직원이 층에 섞여 앉아 있다고 합시다. VLAN이 없으면 같은 스위치에 꽂힌 사람은 모두 한 네트워크라 회계 자료의 브로드캐스트가 영업팀에도 퍼집니다. VLAN 10(영업)·20(개발)·30(회계)으로 나누면, 자리가 어느 층이든 같은 부서끼리 하나의 논리 네트워크로 묶이고 다른 부서와는 기본적으로 격리됩니다. 부서 이동이 생겨도 케이블을 옮길 필요 없이 포트의 VLAN 설정만 바꾸면 됩니다.

| 단점 | 설명 |
|---|---|
| VLAN 간 통신 장비 필요 | 서로 다른 VLAN이 통신하려면 라우터나 L3 스위치가 추가로 필요 |
| 설정·관리 복잡 | 규모가 커지면 VLAN·트렁크 설계와 관리가 복잡 |
| 설정 오류 시 위험 | 잘못 설정하면 통신 장애나 VLAN 홉핑 같은 보안 허점 발생 가능 |

_표 5-2. VLAN의 단점_

## 3. VLAN의 종류

VLAN은 ‘무엇을 기준으로 장비를 특정 VLAN에 배정하는가’에 따라 나뉩니다. 포트처럼 관리자가 미리 고정하는 정적 방식과, MAC·주소처럼 장비를 식별해 자동 배정하는 동적 방식이 있습니다.

- **MAC 기반**: MAC 주소를 기준으로 배정하는 동적 방식. 장비를 어느 포트로 옮겨도 VLAN을 유지하는 이동성이 강점이지만, 모든 MAC을 등록·관리해야 하고 MAC 위조 시 다른 VLAN 접근 위험이 있습니다.
- **네트워크 주소 기반**: IP 서브넷을 기준으로 배정. 3계층 정보를 활용해 같은 서브넷이 자연스럽게 묶이지만, IP를 읽어야 해 2계층 처리보다 부담이 큽니다(그룹을 나눌 뿐 라우팅과는 다릅니다).
- **포트 기반**: 물리 포트별로 VLAN을 지정하는 가장 일반적이고 단순한 방식(정적 VLAN). 설정이 쉽지만 장비를 다른 포트로 옮기면 재설정이 필요합니다.
- **프로토콜 기반**: 프레임이 담은 프로토콜(IPv4·IPv6·IPX 등)에 따라 배정. 오늘날은 대부분 IP로 통일되어 활용도가 낮습니다.
- **멀티캐스트 IP 기반**: 멀티캐스트 그룹을 기준으로 동적 구성. IPTV·실시간 스트리밍에 효율적입니다.

| 종류 | 배정 기준 | 방식 | 특징 |
|---|---|---|---|
| MAC 기반 | MAC 주소 | 동적 | 장비 이동에 강함, 관리 부담 |
| 네트워크 주소 기반 | IP 서브넷 | 동적 | 3계층 정보 활용, 처리 부담 |
| 포트 기반 | 물리 포트 | 정적 | 가장 단순·일반적, 이동 시 재설정 |
| 프로토콜 기반 | 프로토콜 종류 | 동적 | 프로토콜별 분리, 활용도 낮음 |
| 멀티캐스트 IP 기반 | 멀티캐스트 그룹 | 동적 | IPTV·스트리밍에 효율적 |

_표 5-3. VLAN 종류의 종합 비교_

> **보안 유의 — VLAN 홉핑 공격** VLAN은 격리로 보안을 높이지만, 잘못 설정하면 공격자가 자신이 속하지 않은 VLAN에 접근하는 **VLAN 홉핑**이 가능합니다. 대표적으로 스위치인 척 트렁크를 협상하는 스위치 스푸핑, 태그를 이중으로 붙여 넘기는 이중 태깅(Double Tagging)이 있습니다. 대응책으로 사용하지 않는 포트 비활성화, 트렁크 자동 협상(DTP) 비활성화, 네이티브 VLAN을 사용하지 않는 별도 VLAN으로 지정하기 등이 있습니다.
{: .prompt-warning }

## 4. VLAN 간 라우팅

서로 다른 VLAN은 2계층에서 완전히 격리되므로, 통신하려면 반드시 3계층 장비를 거쳐야 합니다. 구현 방법은 두 가지입니다. **라우터 온 어 스틱(Router on a Stick)** 은 라우터의 물리 인터페이스 하나를 트렁크로 스위치에 연결하고 그 위에 VLAN별 논리(서브) 인터페이스를 만들어 라우팅하는 방식으로, 장비가 적게 들지만 모든 VLAN 간 트래픽이 링크 하나를 오가 병목이 생길 수 있습니다. **L3 스위치의 SVI**(Switched Virtual Interface)는 L3 스위치 내부에 VLAN별 가상 인터페이스를 만들어 하드웨어로 라우팅하는 방식으로, 속도가 빠르고 배선이 단순해 기업망의 표준입니다.

| 구분 | 라우터 온 어 스틱 | L3 스위치(SVI) |
|---|---|---|
| 처리 주체 | 외부 라우터(소프트웨어) | 스위치 내부(하드웨어) |
| 성능 | 트렁크 링크 병목 가능 | 고속 |
| 적합 규모 | 소규모 | 중·대규모 표준 |

_표 5-4. VLAN 간 라우팅 방식의 비교_

## 5. 트렁크와 802.1Q 태깅

스위치 포트는 하나의 VLAN에만 속하는 **액세스(Access) 포트**와, 여러 VLAN의 프레임을 태그를 달아 함께 나르는 **트렁크(Trunk) 포트**로 나뉩니다. 단말은 액세스 포트에, 스위치·라우터 간 연결은 트렁크 포트로 구성하는 것이 기본입니다.

| 구분 | 액세스 포트 | 트렁크 포트 |
|---|---|---|
| 소속 VLAN | 하나 | 여러 개 |
| 태그 | 없음(일반 프레임) | 802.1Q 태그 사용 |
| 연결 대상 | 단말(PC·프린터·AP) | 스위치·라우터·가상화 서버 |

_표 5-5. 액세스 포트와 트렁크 포트_

IEEE 802.1Q는 2장에서 배운 이더넷 프레임의 출발지 MAC과 타입 필드 사이에 4바이트 태그를 삽입합니다. 태그 안의 VLAN ID 필드가 12비트이므로 이론상 4,094개의 VLAN을 구분할 수 있습니다. 한편 태그 없이 다니도록 예약된 **네이티브 VLAN(Native VLAN)** 이 있는데, 설정이 어긋나면 이중 태깅 공격의 통로가 되므로 사용하지 않는 별도 VLAN으로 바꿔 두는 것이 안전합니다.

## 6. 실습

> **실습 5-1. VLAN 구성과 트렁크**  
> **실습 목표** 한 대의 스위치를 VLAN으로 나누고, 두 스위치를 트렁크로 이어 VLAN을 확장한다.  
> **주소 계획** PC0(SW1) `192.168.10.1`, PC1(SW1) `192.168.20.1`, PC2(SW2) `192.168.10.2`, PC3(SW2) `192.168.20.2`, 모두 /24. PC0·PC2=VLAN10, PC1·PC3=VLAN20
{: .prompt-tip }

**실습 절차**

1. 스위치(2960) 2대와 PC 4대를 놓고, PC0·PC1을 SW1에, PC2·PC3을 SW2에 연결한다. SW1과 SW2를 Copper Straight-Through로 연결한다(예: SW1 Fa0/1 — SW2 Fa0/1). 이 링크가 두 VLAN을 함께 실어 나를 트렁크가 된다.
2. 각 PC의 IP를 주소 계획대로 설정한다(게이트웨이는 실습 5-2에서 넣으므로 비워 둔다).
3. SW1 콘솔에서 VLAN 두 개를 만든다. `name`은 없어도 동작하지만 알아보기 쉽게 붙인다.

    ```text
    enable
    configure terminal
    vlan 10
    name Sales
    exit
    vlan 20
    name Dev
    exit
    ```

4. PC가 물린 액세스 포트를 각 VLAN에 배정한다.

    ```text
    interface fa0/1
    switchport mode access
    switchport access vlan 10
    exit
    interface fa0/2
    switchport mode access
    switchport access vlan 20
    exit
    ```

5. SW2를 잇는 포트를 트렁크로 설정한다: `interface GigabitEthernet0/1 → switchport mode trunk → exit`. 트렁크 포트는 특정 VLAN 하나에 속하지 않고 태그가 붙은 여러 VLAN 프레임을 통과시킨다.
6. SW2에서 3~5번을 반복한다. VLAN 10·20을 만들고, fa0/1(PC2)을 VLAN10, fa0/2(PC3)를 VLAN20 액세스 포트로 배정한 뒤, SW1 방향 포트를 트렁크로 설정한다.
7. `show vlan brief`로 포트 배정을, `show interfaces trunk`로 트렁크 상태(Trunking)를 확인한다.
8. PC0에서 `ping 192.168.10.2`(PC2, 같은 VLAN10·다른 스위치)가 성공하는지 확인한다(트렁크를 타고 전달됨). 이어 PC0에서 `ping 192.168.20.1`(PC1, 같은 스위치·다른 VLAN20)이 실패(Request timed out)하는지 확인한다.
9. (선택) PC1에서 `ping 192.168.20.2`(PC3, 같은 VLAN20)도 실행해 트렁크가 VLAN20도 함께 나르는지 확인한다.

> **결과 확인** 같은 VLAN끼리는(스위치가 달라도) 성공, 다른 VLAN끼리는(같은 스위치라도) 실패하면 격리가 제대로 된 것. 안 되면 ① 포트가 의도한 VLAN에 배정됐는지, ② 양쪽 트렁크 설정, ③ 케이블 연결을 확인한다. 실무에서는 트렁크의 네이티브 VLAN을 기본값(VLAN 1)에서 사용하지 않는 VLAN으로 바꿔 이중 태깅을 예방한다(예: `switchport trunk native vlan 999`).  
> **계층 짚기** VLAN은 하나의 스위치를 논리적으로 나눈다(2계층). 다른 VLAN끼리 통신하려면 3계층 장비가 필요하다 → 다음 실습.
{: .prompt-tip }

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-03-09-netintro-c5-01.png" alt="토폴로지" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-02.png" alt="PC IP 설정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-03.png" alt="VLAN 생성" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-04.png" alt="액세스 포트 배정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-05.png" alt="트렁크 설정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-06.png" alt="show vlan brief" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-07.png" alt="show interfaces trunk" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-08.png" alt="같은 VLAN ping 성공" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-09.png" alt="다른 VLAN ping 실패" style="max-width:24%">
</div>

> **실습 5-2. VLAN 간 라우팅 (라우터 온 어 스틱)**  
> **실습 목표** 라우터의 서브 인터페이스로 VLAN 간 라우팅을 구성해, 서로 다른 VLAN이 통신하게 한다.  
> **주소 계획** 라우터 서브 인터페이스(게이트웨이): `g0/0.10 = 192.168.10.254`(VLAN10), `g0/0.20 = 192.168.20.254`(VLAN20)
{: .prompt-tip }

**실습 절차**

1. 실습 5-1의 SW1에 라우터(2911) 1대를 추가 연결한다. SW1의 여유 포트(예: G0/2)와 라우터의 G0/0을 연결한다.
2. SW1에서 라우터 쪽 포트를 트렁크로 설정한다: `interface GigabitEthernet0/2 → switchport mode trunk → exit`. 라우터는 VLAN 명령을 모르므로 트렁크 설정은 스위치 쪽에만 한다.
3. 라우터 콘솔에서 물리 인터페이스를 먼저 켠다: `interface GigabitEthernet0/0 → no shutdown → exit`. 서브 인터페이스는 물리 포트가 살아 있어야 동작하므로 IP를 안 주더라도 `no shutdown`은 반드시 한다(잊기 쉬운 함정).
4. VLAN10용 서브 인터페이스를 만든다. 실제로 어느 VLAN과 연결되는지는 `encapsulation dot1Q 10`의 10이 정한다.

    ```text
    interface GigabitEthernet0/0.10
    encapsulation dot1Q 10
    ip address 192.168.10.254 255.255.255.0
    exit
    ```

5. VLAN20용도 같은 방식으로 만든다.

    ```text
    interface GigabitEthernet0/0.20
    encapsulation dot1Q 20
    ip address 192.168.20.254 255.255.255.0
    exit
    ```

6. `show ip interface brief`로 G0/0, G0/0.10, G0/0.20이 모두 up/up인지 확인한다(물리 g0/0은 IP가 없어도 up이면 된다).
7. 각 PC의 기본 게이트웨이를 자기 VLAN의 라우터 IP로 설정한다(VLAN10 → `192.168.10.254`, VLAN20 → `192.168.20.254`).
8. PC0(VLAN10)에서 `ping 192.168.20.1`(PC1, VLAN20)을 실행해 통신을 확인한다. `show ip route`로 두 네트워크가 `C`(직접 연결)로 나타나는지 확인한다.
9. (선택) PC2(VLAN10)에서 PC3(VLAN20)로도 ping을 실행해 4대 전체가 서로 통신되는지 확인한다.

> **결과 확인** 실습 5-1에서 실패했던 VLAN 간 ping이 성공하면 완료. 안 되면 ① 라우터 쪽 포트가 트렁크인지, ② `encapsulation dot1Q` 번호가 VLAN 번호와 일치하는지, ③ 물리 g0/0에 `no shutdown`을 했는지, ④ PC 게이트웨이가 정확한지 확인한다.  
> **계층 짚기** 2계층의 격리를 3계층(라우팅)이 다시 이어 준다. 막대기(Stick) 하나(트렁크 링크 하나)에 여러 VLAN이 꽂혀 라우터로 들어간다고 해서 ‘라우터 온 어 스틱’이라 부른다.
{: .prompt-tip }

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-03-09-netintro-c5-10.png" alt="라우터 추가 연결" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-11.png" alt="라우터 쪽 트렁크" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-12.png" alt="물리 인터페이스 no shutdown" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-13.png" alt="VLAN10 서브 인터페이스" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-14.png" alt="VLAN20 서브 인터페이스" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-15.png" alt="show ip interface brief" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-16.png" alt="PC 게이트웨이 설정" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c5-17.png" alt="VLAN 간 ping 성공" style="max-width:24%">
</div>

---

이 장에서는 물리적 구조에 얽매이지 않고 네트워크를 논리적으로 나누는 VLAN을 살펴보았습니다. VLAN은 성능·관리·보안 모두에 이점을 주며, 보안 단원에서 네트워크 분리(세그먼테이션)의 기본 수단으로 다시 등장합니다. 다음 장에서는 케이블을 벗어나 전파를 매체로 쓰는 **무선 네트워크와 그 보안**을 다룹니다.
