---
title: 네트워크 실무 7강 - 프레임릴레이와 HSRP (따라하기 실습)
date: 2026-09-29 10:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - PacketTracer
  - 프레임릴레이
  - HSRP
pin:
mermaid: false
---

> **이 실습에서 하는 일**
> 프레임릴레이 클라우드로 두 라우터를 잇고, DLCI로 통로를 지정해 통신시킵니다. 이어서 두 라우터에 HSRP를 구성해 하나의 가상 게이트웨이를 만들고, 액티브 라우터를 꺼도 통신이 끊기지 않는 것을 확인합니다.
{: .prompt-info }

## 실습 7-1. 프레임릴레이 구성

> **실습 목표** 프레임릴레이 클라우드로 R1·R2를 연결하고 DLCI 매핑으로 통신시킨다.
> **주소 계획** R1 s0/0/0 `10.0.0.1/30`(DLCI 102), R2 s0/0/0 `10.0.0.2/30`(DLCI 201)
{: .prompt-tip }

**실습 절차**

**1단계 — 장비 준비.** 라우터(2911) 2대에 시리얼 모듈(HWIC-2T)을 장착합니다(라우터 `[Physical]` 탭에서 전원을 끄고 모듈을 슬롯에 끼운 뒤 다시 켜기). `[Network Devices] > [WAN Emulation]`에서 **Cloud-PT**를 하나 놓습니다.

**2단계 — 케이블 연결.** 시리얼 케이블(Serial DCE)로 R1 s0/0/0 ↔ Cloud Serial0, R2 s0/0/0 ↔ Cloud Serial1을 연결합니다.

**3단계 — 클라우드에서 프레임릴레이 설정.** Cloud-PT 클릭 → `[Config] > [Serial0]`에서 Frame Relay로 DLCI를 추가하고(예: 102), Serial1에도 DLCI(201)를 추가합니다. 그런 다음 `[Config] > [Frame Relay]`에서 두 포트의 DLCI를 서로 연결(Serial0의 102 ↔ Serial1의 201)하고 `Add`를 누릅니다.

> **[화면 삽입]** PT 9.0 — Cloud-PT `[Config] > [Frame Relay]`에서 Serial0(DLCI 102) ↔ Serial1(DLCI 201)이 연결된 화면.
> _(그림 파일 예정: `netintro-07b-01.png`)_

**4단계 — 라우터 프레임릴레이 설정.** R1의 시리얼 인터페이스에 프레임릴레이 캡슐화와 map을 설정합니다.

```text
R1(config)# interface serial0/0/0
R1(config-if)# encapsulation frame-relay
R1(config-if)# ip address 10.0.0.1 255.255.255.252
R1(config-if)# frame-relay map ip 10.0.0.2 102 broadcast
R1(config-if)# no frame-relay inverse-arp
R1(config-if)# no shutdown
```
R2도 같은 방식으로 설정하되, 자기 DLCI(201)와 상대 IP(10.0.0.1)를 씁니다.

```text
R2(config)# interface serial0/0/0
R2(config-if)# encapsulation frame-relay
R2(config-if)# ip address 10.0.0.2 255.255.255.252
R2(config-if)# frame-relay map ip 10.0.0.1 201 broadcast
R2(config-if)# no frame-relay inverse-arp
R2(config-if)# no shutdown
```

**5단계 — 확인.** R1에서 `ping 10.0.0.2`가 성공하는지 확인합니다. `show frame-relay map`으로 매핑 상태도 볼 수 있습니다.

> **결과 확인** R1↔R2 ping이 성공하면 완료입니다. 실패 시 ① 클라우드의 DLCI 연결, ② 라우터의 `frame-relay map`에서 상대 IP와 자기 DLCI가 맞는지, ③ `no shutdown` 여부를 점검합니다.
{: .prompt-tip }

## 실습 7-2. HSRP 게이트웨이 이중화

> **실습 목표** 두 라우터가 하나의 가상 게이트웨이(100.0.0.1)를 공유하게 하고, 액티브를 꺼도 통신이 유지됨을 확인한다.
> **주소 계획** R1 `100.0.0.11`, R2 `100.0.0.12`, 가상 IP `100.0.0.1`, PC 게이트웨이 = `100.0.0.1`
{: .prompt-tip }

**실습 절차**

**1단계 — 구성.** 스위치 1대에 라우터 2대(R1·R2)와 PC 1대를 연결합니다. R1·R2의 LAN 인터페이스에 각각 `100.0.0.11`, `100.0.0.12`(마스크 255.255.255.0)를 설정하고 켭니다. PC의 게이트웨이는 **가상 IP `100.0.0.1`** 로 지정합니다.

**2단계 — R1을 액티브로 설정.** 우선순위를 높게(110) 주고 선점을 켭니다.

```text
R1(config)# interface g0/0
R1(config-if)# standby 1 ip 100.0.0.1
R1(config-if)# standby 1 priority 110
R1(config-if)# standby 1 preempt
```

**3단계 — R2를 스탠바이로 설정.** 같은 가상 IP를 쓰되 우선순위는 기본값(100)으로 둡니다.

```text
R2(config)# interface g0/0
R2(config-if)# standby 1 ip 100.0.0.1
R2(config-if)# standby 1 preempt
```

**4단계 — 상태 확인.** 각 라우터에서 `show standby brief`로 R1이 `Active`, R2가 `Standby`인지 확인합니다.

```text
R1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110 P Active   local           100.0.0.12      100.0.0.1
```

> **[화면 삽입]** PT 9.0 — R1 `show standby brief`에서 State가 `Active`, Virtual IP가 `100.0.0.1`로 표시된 화면.
> _(그림 파일 예정: `netintro-07b-02.png`)_

**5단계 — 장애 시험.** PC에서 외부(또는 R1 너머)로 지속 ping(`ping -t` 또는 반복)을 걸어 둔 상태에서, **R1의 g0/0을 `shutdown`** 합니다. 잠깐 끊겼다가 R2가 액티브를 넘겨받아 통신이 이어지는지 확인합니다. R1을 다시 `no shutdown`하면 preempt에 의해 R1이 다시 액티브가 됩니다.

> **결과 확인** R1을 꺼도(잠깐의 끊김 후) 통신이 유지되고, R1 복구 시 다시 R1이 액티브가 되면 완료입니다. PC의 게이트웨이 주소는 처음부터 끝까지 `100.0.0.1` 그대로였다는 점에 주목하세요.
{: .prompt-tip }

> **생각해 보기 — preempt를 끄면 어떻게 될까?**
> 5단계에서 R1을 복구했을 때, preempt 덕분에 R1이 다시 액티브를 가져왔습니다. 만약 preempt를 설정하지 않았다면, R1이 살아나도 계속 R2가 액티브로 남습니다. 어느 쪽이 더 나을까요? ‘항상 성능 좋은 R1을 쓰고 싶다’와 ‘불필요한 전환을 줄여 안정적으로 두고 싶다’ 중 상황에 따라 선택이 달라질 수 있습니다. 대회 조건(“복구되면 다시 A-R1로”)은 어느 쪽인지 생각해 보세요.
{: .prompt-info }

---

이 실습으로 WAN(프레임릴레이)과 게이트웨이 이중화(HSRP)를 직접 구성했습니다. 다음 편([7강 실습 문제])에서 HSRP 이중화 과제에 도전합니다.
