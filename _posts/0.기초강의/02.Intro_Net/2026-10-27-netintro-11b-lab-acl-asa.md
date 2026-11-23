---
title: 네트워크 실무 11강 - 확장 ACL과 ASA 방화벽 (따라하기 실습)
date: 2026-10-27 10:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - PacketTracer
  - ACL
  - ASA
  - 방화벽
pin:
mermaid: false
---

> **이 실습에서 하는 일**
> 먼저 확장 ACL로 특정 트래픽만 골라 차단해 봅니다. 이어서 ASA 방화벽에 INSIDE·DMZ·OUTSIDE 세 존을 구성해, 방화벽의 기본 동작(안→밖 허용, 밖→안 차단)을 확인하고, 정적 NAT로 DMZ의 웹 서버를 외부에 공개합니다.
{: .prompt-info }

## 실습 11-1. 확장 ACL로 트래픽 필터링

> **실습 목표** 확장 ACL로 특정 네트워크에서 오는 특정 트래픽(예: 웹만 허용)을 제어한다.
{: .prompt-tip }

라우터에서 “내부로 들어오는 트래픽 중 웹(80)만 허용하고 나머지는 막는” 확장 ACL을 만들어 봅니다.

```text
R1(config)# access-list 110 permit tcp any host 192.168.10.100 eq 80
R1(config)# access-list 110 deny ip any any
R1(config)# interface g0/0
R1(config-if)# ip access-group 110 in
```

`access-list 110`은 확장 ACL 번호, `permit tcp ... eq 80`은 웹만 허용, 마지막 `deny ip any any`는 (사실 생략해도 암묵적으로 있지만) 명시적으로 나머지를 차단합니다. `ip access-group 110 in`으로 인터페이스에 적용해야 동작합니다.

**확인.** 허용된 웹 접속은 되고, ping(ICMP)이나 다른 포트는 막히는지 확인합니다. `show access-lists`로 규칙별 매치 횟수를 봅니다.

## 실습 11-2. ASA 방화벽 3존 구성

> **실습 목표** ASA에 INSIDE·DMZ·OUTSIDE 존을 구성하고, 방화벽 기본 동작과 웹 서버 공개(정적 NAT)를 확인한다.
> **주소 계획** INSIDE `172.16.0.0/24`(ASA .2), DMZ `192.168.10.0/24`(ASA .254, 웹서버 .100), OUTSIDE `100.0.0.0/24`(ASA .10, 게이트웨이 .1)
{: .prompt-tip }

**1단계 — 장비 배치.** ASA(5506-X 또는 5505) 1대를 놓고, INSIDE 쪽에 내부 PC, DMZ 쪽에 웹 서버, OUTSIDE 쪽에 외부 PC를 각각 스위치를 통해 연결합니다.

**2단계 — 세 인터페이스에 존·보안레벨·IP 설정.** ASA `[CLI]`에서 입력합니다.

```text
ciscoasa> enable
ciscoasa# configure terminal
ciscoasa(config)# hostname ASA
ASA(config)# interface g1/1
ASA(config-if)# nameif inside
ASA(config-if)# security-level 100
ASA(config-if)# ip address 172.16.0.2 255.255.255.0
ASA(config-if)# no shutdown
ASA(config-if)# exit
ASA(config)# interface g1/2
ASA(config-if)# nameif dmz
ASA(config-if)# security-level 50
ASA(config-if)# ip address 192.168.10.254 255.255.255.0
ASA(config-if)# no shutdown
ASA(config-if)# exit
ASA(config)# interface g1/3
ASA(config-if)# nameif outside
ASA(config-if)# security-level 0
ASA(config-if)# ip address 100.0.0.10 255.255.255.0
ASA(config-if)# no shutdown
```

**3단계 — 외부로 나가는 기본 경로.** ASA가 인터넷 방향으로 나가는 디폴트 라우트를 넣습니다.

```text
ASA(config)# route outside 0.0.0.0 0.0.0.0 100.0.0.1
```

**4단계 — 기본 동작 확인.** 내부 PC의 게이트웨이를 ASA의 inside IP(172.16.0.2)로 두고, 내부(100)에서 외부(0)로 ping/웹이 나가는지 확인합니다. 반대로 외부에서 내부로 먼저 접속하면 **기본 차단**됨을 확인합니다.

> **[화면 삽입]** PT 9.0 — ASA에서 `show nameif`로 inside(100)·dmz(50)·outside(0) 세 존이 구성된 화면.
> _(그림 파일 예정: `netintro-11b-01.png`)_

**5단계 — 웹 서버 공개(정적 NAT + ACL).** DMZ의 웹 서버(192.168.10.100)를 외부에서 접속할 수 있도록, ASA 공인 IP의 80번을 서버로 넘깁니다.

```text
ASA(config)# object network WEB
ASA(config-network-object)# host 192.168.10.100
ASA(config-network-object)# nat (dmz,outside) static interface service tcp www www
ASA(config-network-object)# exit
ASA(config)# access-list OUT-IN extended permit tcp any host 192.168.10.100 eq www
ASA(config)# access-group OUT-IN in interface outside
```

이 설정의 뜻: “외부에서 ASA 공인 IP(100.0.0.10)의 80번으로 오는 웹 요청을 DMZ 웹 서버로 넘기고(정적 NAT), 그 통신을 ACL로 허용한다”입니다.

**6단계 — 확인.** 외부 PC의 웹 브라우저에서 `http://100.0.0.10`으로 접속하면 DMZ 웹 서버의 페이지가 열리는지 확인합니다.

> **결과 확인** 내부→외부는 자유롭게 나가고, 외부→내부는 기본 차단되며, 웹(80)만 정적 NAT로 열려 외부에서 서버에 접속되면 완료입니다.
{: .prompt-tip }

> **참고 — PT 버전 차이** ASA의 NAT·ACL 문법은 ASA 소프트웨어 버전에 따라 조금씩 다릅니다(특히 8.3 이후 NAT 문법). 패킷 트레이서 9.0에 내장된 ASA 버전에 맞춰 위 명령을 적용하고, 오류가 나면 `?` 도움말로 해당 버전의 문법을 확인하세요.
{: .prompt-info }

> **생각해 보기 — 왜 딱 80번만 열까?**
> 5단계에서 웹(80)만 열고 나머지는 막았습니다. 관리가 편하도록 “외부에서 이 서버로 다 열어 주면” 안 될까요? 서버가 웹 서비스만 제공한다면, 열려 있는 다른 포트(예: 원격 접속·데이터베이스)는 공격자에게 무엇이 될까요? ‘필요한 것만 최소로 연다’는 원칙(10강의 공격 표면 줄이기)이 방화벽에서 어떻게 적용되는지 생각해 보세요.
{: .prompt-info }

---

이 실습으로 확장 ACL과 ASA 방화벽의 핵심(존·보안레벨·정적 NAT)을 직접 구성했습니다. 다음 편([11강 실습 문제])에서 대회 본사 방화벽과 유사한 과제에 도전합니다.
