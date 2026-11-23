---
title: 네트워크 실무 12강 - GRE 터널과 IPSec (따라하기 실습)
date: 2026-11-03 13:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - PacketTracer
  - GRE
  - IPSec
  - VPN
pin:
mermaid: false
---

> **이 실습에서 하는 일**
> 인터넷(공인망)으로 떨어진 본사와 지사의 라우터를 GRE 터널로 잇고, 사설망끼리 통신되게 합니다. 그런 다음 그 통로를 IPSec으로 암호화해, 인터넷을 지나는 내부 데이터를 보호합니다.
{: .prompt-info }

> **주소 계획**
> 공인망 `100.0.0.0/24`: 본사 R1(A-GRE) `100.0.0.9`, 지사 R2(A2-RT) `100.0.0.66`(중간 경로는 공인망으로 연결되어 서로 도달 가능하다고 가정). 사설망: 본사 LAN `192.168.1.0/24`(R1 .1), 지사 LAN `192.168.2.0/24`(R2 .1). 터널: 번호 10, `200.0.0.0/30`(R1 200.0.0.1, R2 200.0.0.2). IPSec crypto map 이름 `MAP`.
{: .prompt-tip }

## 실습 12-1. GRE 터널 구성

**1단계 — 공인망 연결 확인.** 두 라우터의 공인 인터페이스에 IP를 설정하고, 서로의 공인 IP로 ping이 되는지 먼저 확인합니다(터널은 공인망이 먼저 통해야 만들어집니다). 각 라우터의 사설 LAN 인터페이스도 설정합니다(R1 g0/0 = 192.168.1.1, R2 g0/0 = 192.168.2.1).

**2단계 — R1에 터널 인터페이스 생성.** 가상의 터널 인터페이스를 만들고, 겉봉투의 출발지·목적지(공인 IP)를 지정합니다.

```text
R1(config)# interface tunnel 10
R1(config-if)# ip address 200.0.0.1 255.255.255.252
R1(config-if)# tunnel source 100.0.0.9
R1(config-if)# tunnel destination 100.0.0.66
R1(config-if)# exit
```

**3단계 — R2에도 터널 생성.** 출발지·목적지를 반대로 지정합니다.

```text
R2(config)# interface tunnel 10
R2(config-if)# ip address 200.0.0.2 255.255.255.252
R2(config-if)# tunnel source 100.0.0.66
R2(config-if)# tunnel destination 100.0.0.9
R2(config-if)# exit
```

**4단계 — 사설망을 터널로 보내는 경로.** 상대 사설망으로 가는 트래픽이 터널을 통하도록 정적 경로를 넣습니다.

```text
R1(config)# ip route 192.168.2.0 255.255.255.0 200.0.0.2
R2(config)# ip route 192.168.1.0 255.255.255.0 200.0.0.1
```

**5단계 — 확인.** 본사 PC(192.168.1.x)에서 지사 PC(192.168.2.x)로 ping이 되는지 확인합니다. `show ip interface brief`에서 Tunnel10이 `up`인지 봅니다.

> **결과 확인** 사설망끼리 ping이 성공하면 GRE 터널이 동작하는 것입니다. 다만 지금은 **암호화가 없어**, 공인망 구간에서 내용이 평문으로 흐릅니다. 다음 실습에서 IPSec으로 보호합니다.
{: .prompt-tip }

## 실습 12-2. IPSec으로 터널 보호

> **실습 목표** GRE 터널이 지나는 공인 구간을 IPSec으로 암호화한다.
{: .prompt-tip }

**1단계 — 1단계 협상 규칙(ISAKMP).** 두 라우터에 똑같이 설정합니다. 대회 조건대로 정책 번호 10, 암호 aes 256, 해시 sha를 씁니다.

```text
R1(config)# crypto isakmp policy 10
R1(config-isakmp)# encryption aes 256
R1(config-isakmp)# hash sha
R1(config-isakmp)# authentication pre-share
R1(config-isakmp)# group 2
R1(config-isakmp)# exit
R1(config)# crypto isakmp key SECUKEY address 100.0.0.66
```

**2단계 — 2단계 데이터 보호(transform-set).**

```text
R1(config)# crypto ipsec transform-set MYSET esp-aes 256 esp-sha-hmac
```

**3단계 — 보호할 트래픽 지정(ACL).** GRE 터널 트래픽(두 공인 IP 사이)을 보호 대상으로 지정합니다.

```text
R1(config)# access-list 110 permit gre host 100.0.0.9 host 100.0.0.66
```

**4단계 — crypto map으로 묶어 인터페이스에 적용.** 상대·알고리즘·대상 트래픽을 묶어 공인 인터페이스에 겁니다.

```text
R1(config)# crypto map MAP 10 ipsec-isakmp
R1(config-crypto-map)# set peer 100.0.0.66
R1(config-crypto-map)# set transform-set MYSET
R1(config-crypto-map)# match address 110
R1(config-crypto-map)# exit
R1(config)# interface g0/1
R1(config-if)# crypto map MAP
```

**5단계 — R2에 대칭으로 설정.** R2에도 같은 정책·transform-set·crypto map을 넣되, peer와 key 주소, ACL의 host 순서를 R1의 반대(상대가 100.0.0.9)로 지정합니다.

**6단계 — 확인.** 본사 PC↔지사 PC로 통신을 몇 번 일으킨 뒤, 라우터에서 암호화 통계를 확인합니다.

```text
R1# show crypto ipsec sa
   ... #pkts encaps: 4, #pkts encrypt: 4, ...
   ... #pkts decaps: 4, #pkts decrypt: 4, ...
```

> **[화면 삽입]** PT 9.0 — `show crypto ipsec sa`에서 encrypt/decrypt 패킷 수가 증가한 화면(암호화 동작 확인).
> _(그림 파일 예정: `netintro-12c-01.png`)_

> **결과 확인** 사설망 통신은 그대로 되면서, `show crypto ipsec sa`의 encrypt/decrypt 카운터가 올라가면 IPSec이 GRE 터널을 보호하고 있는 것입니다.
{: .prompt-tip }

> **생각해 보기 — 양쪽 설정이 하나라도 다르면?**
> IPSec은 양쪽 라우터의 정책(암호=aes256, 해시=sha, 키 등)이 정확히 일치해야만 통로가 열립니다. 만약 한쪽만 `hash sha`이고 다른 쪽이 `hash md5`라면 어떻게 될까요? 두 사람이 서로 다른 암호책으로 편지를 주고받으려는 상황을 떠올려 보세요. IPSec 설정에서 ‘양쪽 대칭’이 왜 그토록 강조되는지, 그리고 문제가 생겼을 때 어디부터 비교해야 할지 생각해 보세요.
{: .prompt-info }

## 잘 안 될 때 점검 순서

| 증상 | 점검할 것 |
|---|---|
| 터널이 up 안 됨 | 공인 IP끼리 먼저 ping이 되는지, tunnel source/destination이 서로 반대로 맞는지 |
| 사설망 통신 실패 | 상대 사설망으로 가는 정적 경로(터널 방향)를 양쪽에 넣었는지 |
| 암호화 카운터가 안 오름 | 양쪽 isakmp 정책·key·transform-set이 일치하는지, crypto map이 공인 인터페이스에 적용됐는지 |

---

이 실습으로 대회의 가장 어려운 구간인 GRE+IPSec VPN을 직접 구성했습니다. 다음 편([12강 실습 문제])에서 본사–지사 터널 과제에 도전합니다. 이로써 개별 기술을 모두 배웠고, 13·14강에서는 이 모두를 합쳐 실제 대회 수준의 종합 과제를 완주합니다.
