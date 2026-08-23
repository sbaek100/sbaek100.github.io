---
title: 네트워크 실무 4강 - 스위치 CLI와 포트 보안 (따라하기 실습)
date: 2026-09-08 10:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - PacketTracer
  - CLI
  - 스위치
  - 포트보안
pin:
mermaid: false
---
> **이 실습에서 하는 일**
> 처음으로 장비의 명령어 창(CLI)에 접속해, 스위치가 MAC 주소를 학습하는 과정을 눈으로 확인합니다. 이어서 포트 보안을 설정해, 허가되지 않은 장비가 연결되면 포트가 차단되도록 만듭니다.
{: .prompt-info }

## 1. 장비 CLI 첫걸음

라우터와 스위치는 PC처럼 화면에 클릭 메뉴가 아니라, **명령어를 직접 입력하는 CLI**(Command Line Interface)로 설정합니다. 장치를 클릭한 뒤 `[CLI]` 탭을 열면 검은 명령어 창이 나옵니다. CLI에는 권한이 다른 몇 가지 ‘모드’가 있고, 프롬프트 모양으로 지금 어느 모드인지 알 수 있습니다.

| 모드 | 프롬프트 | 진입 명령 | 하는 일 |
|---|---|---|---|
| 사용자 모드 | `Switch>` | (접속 직후) | 상태 조회 등 제한된 명령 |
| 특권 모드 | `Switch#` | `enable` | 모든 조회·설정 파일 조작 |
| 전역 설정 모드 | `Switch(config)#` | `configure terminal` | 이름·포트·보안 등 설정의 시작점 |

_표 4-5. CLI의 기본 모드_

가장 먼저 익힐 것은 모드 이동과 이름 변경, 그리고 **설정 저장**입니다. 아래는 스위치 이름을 `SW1`로 바꾸는 예입니다. 한 줄씩 입력해 보세요.

```text
Switch> enable
Switch# configure terminal
Switch(config)# hostname SW1
SW1(config)# exit
SW1# write memory
```

> **꼭 기억할 것 — 저장** `hostname`을 바꾸면 프롬프트가 즉시 `SW1`로 바뀝니다. 하지만 설정은 아직 전원이 꺼지면 사라지는 임시 메모리(RAM)에 있습니다. `write memory`(또는 `copy running-config startup-config`)를 실행해야 전원이 꺼져도 남는 곳(NVRAM)에 저장됩니다. **대회에서 저장을 잊으면 그동안의 작업이 모두 사라집니다.**
{: .prompt-warning }

## 2. 실습 4-1. 스위치의 MAC 주소 학습 관찰

> **실습 목표** 스위치가 프레임의 출발지 MAC을 학습해 주소 테이블을 채우는 과정을 확인한다.
> **주소 계획** PC0 `192.168.10.1`, PC1 `192.168.10.2`, PC2 `192.168.10.3` (모두 /24)
{: .prompt-tip }

**실습 절차**

**1단계.** 스위치(2960) 1대에 PC 3대를 연결하고 IP를 위 계획대로 설정합니다. 스위치 이름을 위 방법으로 `SW1`로 바꿉니다.

**2단계.** 스위치 `[CLI]`에서 `show mac-address-table`을 입력해 테이블이 비어 있는지 확인합니다.

```text
SW1# show mac-address-table
          Mac Address Table
-------------------------------------------
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
(비어 있음)
```

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786429816778.png)


**3단계.** PC0에서 `ping 192.168.10.2`로 PC1에 통신을 보냅니다.

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786429890161.png)

**4단계.** 다시 `show mac-address-table`을 입력해, 학습된 MAC 주소와 연결 포트를 확인합니다.

```text
SW1# show mac-address-table
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0001.6350.1a01    DYNAMIC     Fa0/1
   1    000c.851f.2b02    DYNAMIC     Fa0/2
```


![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786429963520.png)
> **결과 확인** 통신 전에는 비어 있던 MAC 테이블에, 통신 후 각 PC의 MAC과 포트가 나타나면 완료입니다. 스위치는 IP가 아니라 **MAC(2계층 주소)** 으로 판단한다는 것을 직접 확인한 것입니다.
{: .prompt-tip }


---

## 3. 실습 4-2. 포트 보안 설정

앞의 실습 4-1에서 스위치는 어떤 MAC이 어느 포트에 연결됐는지 스스로 학습한다는 것을 보았습니다. 그런데 스위치는 기본적으로 **아무 장비나 꽂으면 그대로 받아들입니다.** 회의실 벽의 네트워크 포트에서 회사 PC를 뽑고 외부인이 자기 노트북을 꽂아도, 스위치는 그 노트북을 순순히 네트워크에 들여보냅니다. **포트 보안(Port Security)** 은 바로 이런 상황을 막기 위해 ‘이 포트에는 정해진 장비(MAC)만 연결할 수 있다’는 규칙을 스위치에 심는 기능입니다.

> **실습 목표** 스위치 포트에 연결 가능한 MAC 수를 제한하고, 위반 시 포트를 차단해 비인가 단말의 연결을 막는다.
> **주소 계획** 정상 PC `192.168.10.10`, 대상 PC(통신 상대) `192.168.10.20`, 침입 PC `192.168.10.99` (모두 /24)
{: .prompt-tip }

### 포트 보안은 무엇을 정하는가

포트 보안을 설정할 때 우리는 세 가지를 정합니다. 이 셋만 이해하면 뒤의 명령이 술술 읽힙니다.

**① 몇 개까지 허용할까? (maximum)** — 한 포트에 붙을 수 있는 MAC 주소의 최대 개수입니다. PC 한 대만 꽂는 포트라면 `1`이 적당합니다. (IP 전화기 뒤에 PC를 다는 포트처럼 2대가 정상인 경우엔 `2`로 둡니다.)

**② 허용할 MAC을 어떻게 지정할까? (mac-address)** — 세 가지 방식이 있습니다.

| 방식 | 지정 방법 | 특징 |
|---|---|---|
| 정적(static) | 관리자가 MAC을 직접 입력 | 정확하지만 일일이 조사·입력해야 해 번거로움 |
| 동적(dynamic) | 연결된 MAC을 자동 학습(기본) | 편하지만 재부팅하면 잊어버림 |
| 스티키(sticky) | 자동 학습한 MAC을 설정 파일에 저장 | 편하면서 재부팅에도 유지 — 실무에서 가장 많이 씀 |

_표 4-6. 보안 MAC 주소 지정 방식_

**③ 규칙을 어기면 어떻게 할까? (violation)** — 허용되지 않은 MAC이 들어왔을 때의 대응입니다.

| 모드 | 동작 | 특징 |
|---|---|---|
| protect | 위반 프레임만 조용히 버림 | 관리자에게 알림 없음 |
| restrict | 버리고 로그·카운터에 기록 | 포트는 살아 있음, 기록 남김 |
| shutdown | 포트 자체를 즉시 차단 | 가장 강력(기본값), 관리자가 직접 복구해야 함 |

_표 4-7. 위반(violation) 대응 모드_

우리는 가장 확실하게 막는 **shutdown**을 씁니다. 침입이 감지되면 포트를 통째로 잠가, 관리자가 확인하기 전까지는 그 포트를 아무도 못 쓰게 합니다.

**실습 절차**

**1단계 — 배치와 포트 보안 설정.** 스위치(2960)에 정상 PC를 `Fa0/1`에, 통신 상대가 될 대상 PC를 `Fa0/2`에 연결하고 각 IP를 설정합니다(정상 PC `.10`, 대상 PC `.20`). 그런 다음 정상 PC가 물린 `Fa0/1`을 액세스 모드로 두고 포트 보안을 설정합니다. 아래를 순서대로 입력합니다.

```text
SW1# configure terminal
SW1(config)# interface fa0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security violation shutdown
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# exit
```

각 명령을 한 줄씩 뜯어보면, 앞에서 정한 세 가지(개수·MAC 지정·위반 대응)가 그대로 명령이 된 것을 알 수 있습니다.

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430268951.png)

| 명령 | 의미 |
|---|---|
| `switchport mode access` | 이 포트를 단말용(액세스) 포트로 고정. 포트 보안은 액세스 포트에서만 동작하므로 **반드시 먼저** |
| `switchport port-security` | 이 포트에 포트 보안 기능을 켬 |
| `switchport port-security maximum 1` | 허용 MAC은 1개(①) |
| `switchport port-security violation shutdown` | 위반 시 포트 차단(③) |
| `switchport port-security mac-address sticky` | 정상 연결된 MAC을 자동 학습해 고정(②) |

_표 4-8. 포트 보안 명령의 의미_

> **주의 — 순서가 중요합니다** `switchport mode access`를 먼저 넣지 않으면, 포트가 자동 협상 상태로 남아 포트 보안이 걸리지 않거나 오류가 납니다. ‘액세스로 고정 → 포트 보안 켜기’ 순서를 지키세요.
{: .prompt-warning }

**2단계 — 정상 MAC 학습시키기.** 정상 PC의 `[Desktop] > [Command Prompt]`에서 대상 PC로 `ping 192.168.10.20`을 보냅니다. 

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430383203.png)


왜 통신을 한 번 시켜야 할까요? sticky는 ‘포트로 프레임이 들어와야’ 그 출발지 MAC을 학습하기 때문입니다. 즉 아직 아무 통신도 없으면 스위치는 이 포트에 어떤 장비가 정상인지 모릅니다. 이 ping 한 번으로 스위치가 `Fa0/1`에 연결된 정상 PC의 MAC을 sticky(자동 고정)로 기억합니다. 스위치에서 `show port-security address`를 입력하면, 그 MAC이 `Fa0/1`에 `SecureSticky`로 등록된 것을 볼 수 있습니다.



```text
SW1# show port-security address
          Secure Mac Address Table
Vlan    Mac Address       Type            Ports
----    -----------       ----            -----
   1    0001.6350.1a01    SecureSticky    FastEthernet0/1
```

`show running-config interface fa0/1`을 입력해 보면, 설정 안에 `switchport port-security mac-address sticky 0001.6350.1a01`처럼 방금 학습한 MAC이 **자동으로 적혀 들어간 것**을 확인할 수 있습니다. 이것이 sticky(‘달라붙는다’)라는 이름의 실제 모습입니다.

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430412679.png)


**3단계 — 침입 PC로 교체 (공격 재현).** 이제 앞에서 말한 위협 — 정상 PC를 뽑고 외부인이 자기 장비를 꽂는 상황 — 을 직접 재현합니다. 

![368](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430458856.png)


패킷 트레이서에서 `Fa0/1`에 연결된 케이블을 삭제한 뒤(오른쪽 도구 모음의 삭제 도구(X)를 골라 케이블을 클릭), 침입 PC(`192.168.10.99`)를 같은 `Fa0/1` 포트에 새로 연결합니다. 그런 다음 침입 PC의 Command Prompt에서 `ping 192.168.10.20`으로 통신을 시도합니다.


이 순간 스위치는 ‘등록된 MAC이 아닌 낯선 MAC’이 들어온 것을 감지합니다.

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430495871.png)

**4단계 — 위반 감지 확인.** 스위치에서 포트 상태를 확인합니다. 낯선 MAC이 감지되어 포트가 잠겼는지 봅니다.

```text
SW1# show port-security interface fa0/1
Port Security              : Enabled
Port Status                : Secure-shutdown
Violation Mode             : Shutdown
Last Source Address:Vlan   : 00d0.58c1.9963:1
Security Violation Count   : 1
```

출력의 각 줄이 상황을 말해 줍니다.

| 항목 | 뜻 |
|---|---|
| Port Status : **Secure-shutdown** | 보안 위반으로 포트가 잠긴 상태(정상일 때는 `Secure-up`) |
| Violation Mode : Shutdown | 우리가 정한 위반 대응(포트 차단) |
| Last Source Address | 위반을 일으킨 MAC — 즉 침입 PC의 MAC |
| Security Violation Count : 1 | 위반이 1회 발생 |

이때 작업 화면에서 `Fa0/1`의 케이블이 **빨간색**으로 바뀌고, `show interface fa0/1`을 보면 포트가 `err-disabled`(오류로 비활성화됨) 상태입니다. shutdown 모드는 위반 프레임만 버리는 것이 아니라 **포트 자체를 꺼 버려** 침입 장비를 확실히 격리합니다.

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430524617.png)

**5단계 — 포트 복구.** shutdown 모드로 잠긴(`err-disabled`) 포트는 저절로 살아나지 않습니다. 관리자가 원인을 확인한 뒤 직접 되살려야 합니다. 침입 장비를 떼어 내고 정상 PC를 다시 연결한 다음, 해당 포트를 껐다 켭니다.

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430578238.png)

```text
SW1(config)# interface fa0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
```

이렇게 하면 포트가 다시 `Secure-up` 상태로 돌아오고, 등록된 정상 PC는 정상 통신합니다. (참고: 일정 시간이 지나면 자동으로 복구되게 하는 `errdisable recovery` 설정도 있지만, 여기서는 수동 복구로 충분합니다.)

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430611303.png)

> **결과 확인** 등록된 정상 MAC은 통신되고, 낯선 MAC이 연결되면 포트가 `Secure-shutdown`으로 차단되며, 껐다 켜면 복구되면 완료입니다. ‘정해진 장비만 그 자리에 꽂을 수 있다’는 물리적 접근 통제를 스위치가 스스로 해낸 것입니다.
{: .prompt-tip }

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430634871.png)


---

### MAC 스푸핑 — 포트 보안이 뚫리는 지점

포트 보안은 MAC 주소로 ‘정상 장비인지’를 판단합니다. 그런데 여기에 근본적인 약점이 있습니다. **MAC 주소는 위조(스푸핑)할 수 있기 때문입니다.**

2강에서 MAC 주소는 랜카드에 공장에서 새겨지지만 소프트웨어로 얼마든지 바꿔 말할 수 있다고 배웠습니다. 공격자는 이 점을 다음처럼 악용합니다.

1. 정상 PC가 내보내는 프레임을 엿보면, 그 안에 **출발지 MAC이 평문**으로 들어 있습니다(암호화되지 않음). 즉 어떤 MAC이 허용됐는지 쉽게 알아냅니다.
2. 공격자가 자기 장비의 MAC을 그 허용된 MAC과 **똑같이** 바꿉니다(스푸핑).
3. 스위치가 보기엔 ‘등록된 바로 그 MAC’이 다시 온 것이라, 포트 보안을 그대로 통과시킵니다.

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430719877.png)

> **직접 확인해 보기 (선택)** 실습을 복구한 뒤, 침입 PC의 `[Config] > [FastEthernet0]`에서 MAC Address 칸을 정상 PC의 MAC과 똑같이 바꿔 보세요. 그러면 포트 보안이 침입 PC를 정상 장비로 착각해 차단하지 않습니다. MAC 기반 통제가 얼마나 쉽게 우회되는지 눈으로 확인할 수 있습니다.
{: .prompt-tip }

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430793388.png)

![](/assets/img/posts/2026-09-08-netintro-04b-lab-switch-portsecurity-1786430800749.png)

그래서 포트 보안은 ‘실수로 엉뚱한 장비를 꽂는 것’이나 ‘방문객·외부인의 단순한 무단 연결’을 막는 **기본 방어**로는 훌륭하지만, 작정하고 MAC을 위조하는 공격자까지 막지는 못합니다. 이것이 포트 보안처럼 MAC 주소에 기대는 통제가 모두 ‘보조 수단’으로 불리는 이유입니다. 장비의 MAC이 아니라 **사용자를 계정으로 인증**하는 더 강한 방식이 필요한데, 그것이 12강에서 배울 **802.1X**입니다.

> **생각해 보기 — 그래도 포트 보안을 쓰는 이유는?**
> MAC 스푸핑으로 뚫릴 수 있는데도 회사들은 여전히 포트 보안을 씁니다. 왜일까요? 모든 침입자가 MAC을 위조할 만큼 정교하지는 않고, 대부분의 무단 연결(방문객이 회의실 포트에 노트북 꽂기, 몰래 공유기 연결하기)은 포트 보안만으로도 걸러지기 때문입니다. ‘완벽하지 않으니 안 쓴다’와 ‘기본은 막고 그 위에 더 강한 방어를 얹는다(계층 방어)’ 중 무엇이 현실적인지 생각해 보세요.
{: .prompt-info }

## 잘 안 될 때 점검 순서

| 증상 | 점검할 것 |
|---|---|
| MAC 테이블이 계속 비어 있음 | 통신(ping)을 한 번이라도 성공시켰는지, 케이블 링크가 초록색인지 |
| 포트 보안이 동작 안 함 | `switchport mode access`를 먼저 넣었는지, `port-security` 명령들이 해당 인터페이스(config-if)에서 입력됐는지 |
| 포트가 계속 차단 상태 | 정상 PC를 다시 꽂고 `shutdown` → `no shutdown`으로 복구했는지 |

---

이 실습으로 처음 장비 CLI를 다루고, 스위치의 동작과 포트 보안을 직접 확인했습니다. 다음 편([4강 실습 문제])에서 포트 보안 과제에 도전합니다.
