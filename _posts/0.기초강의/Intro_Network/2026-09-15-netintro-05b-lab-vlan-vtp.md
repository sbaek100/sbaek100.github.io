---
title: 네트워크 실무 5강 - VLAN·VTP·VLAN간 라우팅 (따라하기 실습)
date: 2026-09-15 10:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - PacketTracer
  - VLAN
  - VTP
  - 트렁크
pin:
mermaid: false
---

> **이 실습에서 하는 일**
> 두 스위치에 VLAN을 만들고 VTP로 정보를 자동 전파한 뒤, 트렁크로 연결해 같은 VLAN은 통하고 다른 VLAN은 막히는 것을 확인합니다. 마지막으로 라우터 온 어 스틱으로 VLAN 간 통신을 이어 줍니다.
{: .prompt-info }

## 실습 5-1. VLAN·VTP·트렁크 구성

> **실습 목표** VTP 서버·클라이언트로 VLAN을 자동 전파하고, 트렁크로 두 스위치를 이어 VLAN을 확장한다.
> **주소 계획** VLAN 10(Sales)·VLAN 20(Dev). PC0·PC2=VLAN10, PC1·PC3=VLAN20. VTP 도메인 `secu.com`.
>
> | PC | VLAN | IP |
> |---|---|---|
> | PC0(SW1) | 10 | 192.168.10.1 /24 |
> | PC1(SW1) | 20 | 192.168.20.1 /24 |
> | PC2(SW2) | 10 | 192.168.10.2 /24 |
> | PC3(SW2) | 20 | 192.168.20.2 /24 |
{: .prompt-tip }

**실습 절차**

**1단계 — 배치·연결.** 스위치(2960) 2대와 PC 4대를 놓습니다. PC0·PC1을 SW1에, PC2·PC3을 SW2에 연결하고, SW1–SW2를 케이블로 잇습니다. 각 PC의 IP를 표대로 설정합니다.

**2단계 — VTP 도메인·모드 설정.** SW1을 VTP 서버로, SW2를 클라이언트로 지정합니다.

```text
SW1(config)# vtp domain secu.com
SW1(config)# vtp mode server
```
```text
SW2(config)# vtp domain secu.com
SW2(config)# vtp mode client
```

**3단계 — 서버에서 VLAN 생성.** VTP 서버인 SW1에서만 VLAN을 만듭니다. (클라이언트 SW2에서는 만들 수 없습니다.)

```text
SW1(config)# vlan 10
SW1(config-vlan)# name Sales
SW1(config-vlan)# exit
SW1(config)# vlan 20
SW1(config-vlan)# name Dev
SW1(config-vlan)# exit
```

**4단계 — 트렁크 설정.** 두 스위치를 잇는 포트를 트렁크로 만듭니다. VTP 정보는 트렁크를 통해서만 전파됩니다.

```text
SW1(config)# interface gigabitEthernet0/1
SW1(config-if)# switchport mode trunk
```
SW2에서도 SW1 방향 포트에 같은 `switchport mode trunk`를 입력합니다.

**5단계 — 전파 확인.** SW2(클라이언트)에서 VLAN을 만들지 않았는데도 VLAN 10·20이 보이는지 확인합니다.

```text
SW2# show vlan brief
VLAN Name       Status    Ports
---- ---------- --------- -------------------------------
10   Sales      active
20   Dev        active
```

> **[화면 삽입]** PT 9.0 — 클라이언트 SW2에서 `show vlan brief`에 서버가 전파한 VLAN 10(Sales)·20(Dev)이 나타난 화면.
> _(그림 파일 예정: `netintro-05b-01.png`)_

**6단계 — 액세스 포트 배정.** 각 PC가 물린 포트를 해당 VLAN의 액세스 포트로 지정합니다(양쪽 스위치 모두).

```text
SW1(config)# interface fa0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit
SW1(config)# interface fa0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
```

**7단계 — 통신 확인.** PC0(VLAN10)에서 PC2(VLAN10, 다른 스위치)로 `ping 192.168.10.2` → **성공**(트렁크를 타고 전달). PC0에서 PC1(VLAN20, 같은 스위치)로 `ping 192.168.20.1` → **실패**(VLAN이 다름).

> **결과 확인** 같은 VLAN끼리는(스위치가 달라도) 성공, 다른 VLAN끼리는(같은 스위치라도) 실패하면 격리가 제대로 된 것입니다. VTP로 SW2에 VLAN이 자동 생성된 것도 확인하세요.
{: .prompt-tip }

> **생각해 보기 — VTP 클라이언트에서 VLAN을 만들려 하면?**
> 5단계에서 클라이언트 SW2에 접속해 `vlan 30`을 만들려고 시도해 보세요. 거부될 것입니다. 왜 클라이언트는 VLAN을 직접 만들 수 없게 막아 두었을까요? 만약 모든 스위치가 자유롭게 VLAN을 만들 수 있다면 큰 회사에서 어떤 혼란이 생길지, VTP가 ‘한 곳에서 관리’를 강제하는 이유를 생각해 보세요.
{: .prompt-info }

## 실습 5-2. VLAN 간 라우팅 (라우터 온 어 스틱)

> **실습 목표** 라우터의 서브 인터페이스로 VLAN 10·20을 통신시킨다.
> **주소 계획** 게이트웨이 g0/0.10=192.168.10.254(VLAN10), g0/0.20=192.168.20.254(VLAN20)
{: .prompt-tip }

**실습 절차**

**1단계.** 실습 5-1의 SW1에 라우터(2911)를 추가로 연결합니다(예: SW1 G0/2 ↔ 라우터 G0/0). SW1의 그 포트를 트렁크로 설정합니다: `interface g0/2` → `switchport mode trunk`.

**2단계.** 라우터의 물리 인터페이스를 먼저 켭니다. IP를 주지 않더라도 반드시 `no shutdown` 해야 서브 인터페이스가 동작합니다(잊기 쉬운 함정).

```text
R1(config)# interface g0/0
R1(config-if)# no shutdown
R1(config-if)# exit
```

**3단계.** VLAN마다 서브 인터페이스를 만들어 게이트웨이 IP를 줍니다. `encapsulation dot1Q 10`의 번호가 어느 VLAN과 연결되는지를 정합니다.

```text
R1(config)# interface g0/0.10
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 192.168.10.254 255.255.255.0
R1(config-subif)# exit
R1(config)# interface g0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.20.254 255.255.255.0
```

**4단계.** 각 PC의 기본 게이트웨이를 자기 VLAN의 라우터 IP로 설정합니다(VLAN10 → 192.168.10.254, VLAN20 → 192.168.20.254).

**5단계.** 이제 PC0(VLAN10)에서 PC1(VLAN20)로 `ping 192.168.20.1`을 실행하면, 실습 5-1에서 실패했던 통신이 **성공**합니다.

> **결과 확인** VLAN 간 ping이 성공하면 완료입니다. ‘막대기(Stick) 하나(트렁크 링크)에 여러 VLAN이 꽂혀 라우터로 들어간다’고 해서 라우터 온 어 스틱이라 부릅니다.
{: .prompt-tip }

## 잘 안 될 때 점검 순서

| 증상 | 점검할 것 |
|---|---|
| VTP가 전파 안 됨 | 두 스위치의 도메인 이름이 같은지, 연결 포트가 트렁크인지 |
| 같은 VLAN인데 통신 실패 | 양쪽 포트가 같은 VLAN 액세스로 배정됐는지, 스위치 간 트렁크가 설정됐는지 |
| VLAN 간 라우팅 실패 | 라우터 물리 포트에 `no shutdown` 했는지, `encapsulation dot1Q` 번호가 VLAN과 일치하는지, PC 게이트웨이가 맞는지 |

---

이 실습으로 VLAN·VTP·트렁크와 VLAN 간 라우팅을 모두 구성했습니다. 다음 편([5강 실습 문제])에서 대회 도면과 유사한 VLAN 구성 과제에 도전합니다.
