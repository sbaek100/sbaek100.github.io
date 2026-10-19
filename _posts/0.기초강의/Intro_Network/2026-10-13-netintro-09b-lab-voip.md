---
title: 네트워크 실무 9강 - CME로 IP 전화 통화 (따라하기 실습)
date: 2026-10-13 10:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - PacketTracer
  - VoIP
  - CME
  - voiceVLAN
pin:
mermaid: false
---

> **이 실습에서 하는 일**
> 라우터를 통화 관리자(CME)로 만들어, IP 전화기 두 대에 전화번호를 자동으로 나눠 주고, 두 전화기가 서로 통화되게 합니다. 음성 VLAN과 option 150이 포함된 DHCP까지 직접 구성합니다.
{: .prompt-info }

> **주소 계획**
> 음성 네트워크 `172.16.1.0/24`, 게이트웨이(라우터 음성 서브인터페이스) `172.16.1.1`. 전화번호 Phone1=`1001`, Phone2=`1002`. 음성 VLAN 150.
{: .prompt-tip }

## 실습 9-1. CME 기본 통화 구성

**1단계 — 장비 배치.** 라우터(2911) 1대, 스위치(2960) 1대, IP 전화기(7960) 2대를 놓습니다. 스위치–라우터, 스위치–전화기 2대를 연결합니다.

> **전화기 전원** IP 전화기는 스위치의 PoE 포트에서 전원을 받거나, 전화기 `[Physical]` 탭에서 전원 어댑터를 연결해야 켜집니다. 화면에 불이 들어오는지 확인하세요.

**2단계 — 스위치에 음성 VLAN 설정.** 전화기가 연결된 포트를 음성 VLAN 150으로 설정합니다. 전화기 뒤에 PC를 달 수 있으므로 데이터 VLAN도 함께 둘 수 있습니다.

```text
SW1(config)# vlan 150
SW1(config-vlan)# name VOICE
SW1(config-vlan)# exit
SW1(config)# interface range fa0/1 - 2
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport voice vlan 150
```
라우터로 가는 포트는 트렁크로 둡니다: `interface g0/1` → `switchport mode trunk`.

**3단계 — 라우터 음성 서브인터페이스.** 라우터가 음성 VLAN의 게이트웨이가 되도록 서브인터페이스를 만듭니다.

```text
R1(config)# interface g0/0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface g0/0.150
R1(config-subif)# encapsulation dot1Q 150
R1(config-subif)# ip address 172.16.1.1 255.255.255.0
```

**4단계 — 음성용 DHCP(option 150 포함).** 전화기가 IP와 설정 서버 주소를 받도록 DHCP 풀을 만듭니다. `option 150`이 핵심입니다.

```text
R1(config)# ip dhcp pool VOICE
R1(dhcp-config)# network 172.16.1.0 255.255.255.0
R1(dhcp-config)# default-router 172.16.1.1
R1(dhcp-config)# option 150 ip 172.16.1.1
```

**5단계 — 통화 서비스(CME) 설정.** 라우터를 통화 관리자로 만듭니다.

```text
R1(config)# telephony-service
R1(config-telephony)# max-ephones 5
R1(config-telephony)# max-dn 5
R1(config-telephony)# ip source-address 172.16.1.1 port 2000
R1(config-telephony)# auto assign 1 to 5
```

**6단계 — 전화번호 만들기.** 각 전화기에 줄 번호를 만듭니다.

```text
R1(config)# ephone-dn 1
R1(config-ephone-dn)# number 1001
R1(config-ephone-dn)# exit
R1(config)# ephone-dn 2
R1(config-ephone-dn)# number 1002
```

> **[화면 삽입]** PT 9.0 — 두 IP 전화기 화면에 각각 내선번호 `1001`, `1002`가 표시된 상태.
> _(그림 파일 예정: `netintro-09b-01.png`)_

**7단계 — 통화 시험.** 잠시 기다리면 두 전화기 화면에 번호(1001·1002)가 표시됩니다. Phone1의 수화기를 들고(`[GUI]`에서 클릭) `1002`를 누르면 Phone2가 울리고, Phone2의 수화기를 들면 통화가 연결됩니다.

> **결과 확인** 두 전화기에 번호가 뜨고 서로 통화가 연결되면 완료입니다. 번호가 안 뜨면 ① option 150이 라우터 자기 IP인지, ② `ip source-address`가 음성 서브인터페이스 IP(172.16.1.1)인지, ③ 스위치 포트의 voice vlan, ④ 라우터로 가는 포트가 트렁크인지 점검합니다.
{: .prompt-tip }

> **생각해 보기 — option 150을 빠뜨리면?**
> 4단계에서 `option 150`을 일부러 빼고 전화기를 켜 보세요. 전화기는 IP는 받지만 번호가 뜨지 않을 것입니다. 왜 그럴까요? 전화기가 IP만으로는 부족하고 ‘설정을 어디서 받을지’를 알아야 하는데, 그 주소를 알려 주는 것이 바로 option 150이기 때문입니다. 데이터 PC는 필요 없는데 IP 전화기에만 이 항목이 필요한 이유를 9강 이론과 연결해 생각해 보세요.
{: .prompt-info }

---

이 실습으로 라우터를 CME로 만들어 IP 전화 통화를 구성했습니다. 다음 편([9강 실습 문제])에서 대회 본사 전화 구성과 유사한 과제에 도전합니다. 다른 지점과의 통화(GRE 터널·dial-peer)는 12강에서 이어집니다.
