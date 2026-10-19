---
title: "[AI 보안관제 구축] 2-1. 경계 방화벽 세우기 — 방화벽·IDS·IPS 이해와 OPNsense 설치"
date: 2027-06-28 09:00:00 +0900
categories:
  - 1.응용강의
  - AI보안관제구축
tags:
  - 보안관제
  - OPNsense
  - 방화벽
  - IDS
  - IPS
pin: false
math: false
mermaid: false
---

> 지난 단계 도달 상태: 공격자(Kali 59.10)와 표적(victim 59.20)이 같은 노출망에 있고, 공격이 **무방비로** 통했습니다.
> 오늘의 목표: 먼저 **방화벽·IDS·IPS가 무엇인지** 제대로 이해한 뒤, 두 VM 사이에 방화벽 **OPNsense** 를 끼워 넣어 망을 분리하고, **바깥으로 나가는 길을 방화벽 하나로 모읍니다.**
> 오늘 켜는 VM: **OPNsense(신규) + Kali + victim** 세 대.
{: .prompt-info }

## 0. 2단계에서 할 일

1단계는 방어 장치가 없었습니다. 2단계에서는 네트워크의 **경계(Perimeter)** 를 세웁니다. 세 시간에 걸쳐 이렇게 진행합니다.

| 강 | 내용 | 마치면 |
| --- | --- | --- |
| **2-1 (오늘)** | 개념 이해 + OPNsense 설치·망 배선 | 트래픽이 방화벽을 통과하게 됨 |
| 2-2 | 방화벽 규칙(최소 허용) | 필요한 포트만 열리고 나머지 스캔은 막힘 |
| 2-3 | Suricata IDS(탐지) | 허용된 포트로 들어오는 **공격 패턴**을 탐지·기록 |
| 2-3 부록 *(선택)* | Suricata IPS(차단) 전환 | 탐지된 공격을 그 자리에서 **차단(drop)** — 건너뛰어도 됨 |

> 오늘은 앞부분(1~3장)이 **이론**입니다. 조금 길지만, 이 개념을 알아야 뒤의 규칙 설정이 "왜 그렇게 하는지" 이해됩니다. 실습만 급하다면 4장부터 따라 해도 되지만, 이론을 먼저 읽기를 권합니다.
{: .prompt-tip }

---

## 1. 이론 ① — 방화벽(Firewall)이란 무엇인가

### 1.1 개념: 네트워크의 현관 경비실

아파트 현관에는 **경비실**이 있습니다. 드나드는 사람을 보고 "누구세요? 몇 호 가세요?"를 확인해, 허락된 사람만 들여보냅니다. 네트워크에서 이 역할을 하는 것이 **방화벽(Firewall)** 입니다.

방화벽은 미리 정해 둔 **규칙(Rule)** 에 따라, 오가는 데이터(트래픽)를 검사해 **허용(Allow)** 하거나 **차단(Block)** 합니다. 판단의 기준은 주로 다음 네 가지입니다.

| 기준 | 예시 | 현관 비유 |
| --- | --- | --- |
| 출발지 IP | "192.168.57.10 에서 온 것" | 누가 왔는가 |
| 목적지 IP | "192.168.59.20 으로 가는 것" | 몇 호를 찾는가 |
| 포트(Port) | "3000번(웹) 문으로" | 어느 문으로 들어가려는가 |
| 방향 | "바깥 → 안쪽" | 들어오는가 나가는가 |

> **포트(Port)** 란 한 컴퓨터 안의 "번호가 붙은 문"입니다. 웹은 보통 80·443·(우리 실습은 3000), SSH 원격접속은 22번 문을 씁니다. 방화벽은 "어느 문을 열어 둘지"를 정하는 장치이기도 합니다.
{: .prompt-tip }

### 1.2 역사: 왜 방화벽이 태어났는가

인터넷 초창기(1980년대)에는 방화벽이 없었습니다. 서로 믿는 연구자들의 네트워크였기 때문입니다. 그 순진한 시대를 끝낸 사건이 있습니다.

- **1988년 — 모리스 웜(Morris Worm)**: 대학원생이 만든 프로그램이 스스로 퍼지며 당시 인터넷에 연결된 컴퓨터 약 6만 대 중 **10%가량(약 6천 대)** 을 마비시켰습니다. 최초의 대규모 인터넷 사고였고, "네트워크의 경계를 지켜야 한다"는 인식을 심었습니다. (이 사건을 계기로 미국에 최초의 사이버 침해대응조직 **CERT** 가 만들어졌습니다.)

이후 방화벽은 세대를 거치며 똑똑해졌습니다.

| 세대 | 시기 | 방식 | 한계/특징 |
| --- | --- | --- | --- |
| **1세대 · 패킷 필터** | 1980년대 말 | 패킷 하나하나의 IP·포트만 보고 허용/차단 | 연결의 앞뒤 맥락을 모름 |
| **2세대 · 상태 기반(Stateful)** | 1990년대 (Check Point FireWall-1, 1994) | "연결 상태"를 기억 → 내가 보낸 요청의 **응답**은 자동 허용 | 훨씬 똑똑하고 안전. 오늘날 기본 |
| **3세대 · 차세대(NGFW)** | 2000년대~ | 애플리케이션·사용자 인식 + IDS/IPS 통합 | 우리가 쓸 OPNsense가 여기 해당 |

우리가 세울 OPNsense는 **상태 기반 방화벽**이면서, 플러그인으로 **IDS/IPS까지 통합**한 3세대형입니다.

> 🖼 **필요 이미지 ①** — 방화벽 3세대 발전 타임라인 인포그래픽(패킷필터 1980s → 상태기반 1994 → NGFW 2000s). 각 세대 아이콘과 한 줄 설명 포함.
{: .prompt-info }

---

## 2. 이론 ② — IDS와 IPS란 무엇인가

### 2.1 방화벽만으로는 부족한 이유

방화벽은 "**어느 문으로** 들어오는가"를 봅니다. 하지만 우리가 정상적으로 열어 둔 **웹 문(3000번)** 으로 공격 문장을 실어 보내면, 방화벽은 그것이 정상 손님인지 도둑인지 구분하지 못합니다. 문은 열려 있고, 그 문으로 들어온 사람의 **행동**은 방화벽이 보지 않기 때문입니다.

그래서 필요한 것이 **감시 카메라와 경비원**에 해당하는 장치입니다.

- **IDS(Intrusion Detection System, 침입 탐지 시스템)**: 지나가는 트래픽의 **내용**을 들여다보고, 알려진 공격 패턴이면 **경보(alert)** 를 울립니다. 트래픽 자체는 그냥 흘려보냅니다. → **감시 카메라**(보고 기록하지만 막지는 않음)
- **IPS(Intrusion Prevention System, 침입 방지 시스템)**: 같은 검사를 하되, 공격이라고 판단하면 **그 자리에서 차단**합니다. → **경비원**(수상하면 즉시 제지)

| 구분 | 방화벽 | IDS | IPS |
| --- | --- | --- | --- |
| 무엇을 보나 | IP·포트(겉봉) | 내용(패턴) | 내용(패턴) |
| 공격 시 | 규칙에 없으면 차단 | **경보만** | **경보 + 차단** |
| 위치 | 경로 위 | 보통 트래픽 **복제**를 관찰 | 트래픽 **경로 위(인라인)** |
| 비유 | 현관 경비실 | 감시 카메라 | 경비원 |

### 2.2 역사: 탐지에서 방지로

- **1980년 — James P. Anderson 보고서**: "컴퓨터 보안 위협 감시" 보고서에서 로그를 분석해 침입을 찾아내자는 아이디어를 처음 제시했습니다. 침입탐지의 출발점입니다.
- **1987년 — Dorothy Denning의 침입탐지 모델**: "평소와 다른 행동"을 통계로 잡아내는 이론(IDES)을 정립했습니다. 오늘날 이상행위 탐지의 뿌리입니다.
- **1998년 — Snort 공개(Martin Roesch)**: 누구나 쓸 수 있는 오픈소스 IDS가 등장해 시그니처 기반 탐지가 대중화됐습니다. Snort의 규칙 문법은 사실상 업계 표준이 됩니다.
- **2000년대 초 — IPS의 부상**: IDS는 "탐지만" 하다 보니 공격이 이미 도달한 뒤에야 알게 되는 한계가 있었습니다. 그래서 트래픽 경로 위에 놓고 **실시간으로 끊는** IPS가 확산됩니다. 즉 IPS는 IDS의 발전형입니다.

### 2.3 탐지 방식 두 가지

| 방식 | 원리 | 장점 | 단점 |
| --- | --- | --- | --- |
| **시그니처 기반** | 알려진 공격의 "지문(패턴)"과 대조 | 정확, 오탐 적음 | 새(변형) 공격은 놓칠 수 있음 |
| **이상행위 기반** | 평소 트래픽과 다르면 의심 | 미지의 공격도 포착 | 오탐 많을 수 있음 |

우리 실습의 Suricata는 **시그니처 기반**을 씁니다. (평소와 다른 "행동"을 잡는 이상탐지는 5단계에서 **AI**가 맡습니다 — 여기서 미리 씨앗을 뿌려 둡니다.)

> 🖼 **필요 이미지 ②** — IDS와 IPS의 위치 차이 그림. IDS는 스위치 미러(복제) 포트로 트래픽을 "관찰만", IPS는 트래픽 경로 한가운데(인라인)에서 "차단"하는 모습을 나란히 비교.
{: .prompt-info }

---

## 3. 이론 ③ — OPNsense란 무엇인가

### 3.1 개념

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787611301267.png)

**OPNsense**는 **FreeBSD**(리눅스와 사촌뻘인 안정적인 서버 운영체제) 위에서 도는 **오픈소스 방화벽·라우터 운영체제**입니다. 한 대의 장비(또는 VM)에 설치하면, 웹 브라우저 화면(GUI)만으로 다음을 모두 다룰 수 있습니다.

- 방화벽 규칙 (허용/차단)
- 라우팅·NAT (망과 망을 잇고 인터넷으로 내보내기)
- **IDS/IPS** (Suricata 내장)
- VPN·프록시·DNS 등 부가 기능

즉 OPNsense는 "**방화벽 + IDS/IPS + 라우터를 하나로 합친 소프트웨어 장비**"입니다. 내부적으로는 트래픽을 걸러 내는 엔진으로 **pf**(원래 OpenBSD에서 만든 강력한 패킷 필터)를 씁니다.

### 3.2 계보(역사)

OPNsense는 하늘에서 뚝 떨어진 것이 아니라, 오픈소스 방화벽의 계보를 잇습니다.

```text
m0n0wall (2003)  →  pfSense (2006)  →  OPNsense (2015)
  최초의 임베디드      pfSense가              Deciso사가 pfSense를
  BSD 방화벽          가장 유명해짐           포크(분기)해 새로 시작
```

OPNsense는 2015년 pfSense에서 갈라져 나와, **순수 오픈소스**와 **빠른 보안 업데이트**, **깔끔한 웹 화면**을 앞세워 성장했습니다.

### 3.3 왜 이 강의에서 OPNsense를 쓰는가

| 이유 | 설명 |
| --- | --- |
| **무료·오픈소스** | 비용 없이 상용 방화벽 수준의 기능을 배울 수 있습니다 |
| **초보 친화 GUI** | 명령어를 몰라도 마우스로 규칙을 만들 수 있습니다 |
| **올인원** | 방화벽·IDS/IPS(Suricata)·VPN을 한 대로 — 우리 실습에 딱 맞습니다 |
| **REST API 제공** | ★ 5단계에서 **AI가 자동으로 차단 규칙을 넣을 때** 바로 이 API를 씁니다. 우리 프로젝트의 최종 목표(자동 대응)를 가능하게 하는 핵심 이유입니다 |
| **실무 축소판** | 기업이 쓰는 방화벽 어플라이언스의 동작 원리를 그대로 경험합니다 |

> **왜 상용 방화벽이 아니라 OPNsense인가?** 상용 장비는 비싸고 자동화 API가 닫혀 있는 경우가 많습니다. OPNsense는 무료이면서 **API가 열려 있어**, 우리가 만들 "AI가 스스로 차단하는" 파이프라인을 학생이 직접 완성할 수 있습니다.
{: .prompt-tip }

> 🖼 **필요 이미지 ③** — OPNsense 계보도(m0n0wall→pfSense→OPNsense) 또는 OPNsense가 방화벽·IDS/IPS·라우터를 하나로 합친 개념도.
{: .prompt-info }

---

## 4. 오늘의 배선 — 무엇이 어떻게 바뀌나

1단계에서는 Kali와 victim이 **같은 망(soc-dmz)** 에 나란히 있었습니다. 오늘은 그 사이에 OPNsense를 넣고, **Kali를 바깥 공격망(soc-wan)으로 이사**시킵니다. victim의 IP(**59.20**)는 그대로입니다.

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787611572897.png)



```text
[이전 · 1단계]
  Kali(59.10) ──────── victim(59.20)       같은 망, 방어 없음
     │                    │
   NAT(인터넷)          NAT(인터넷)          각자 따로 바깥에 나감

[이후 · 2단계]
                     인터넷(NAT)
                         ╎  규칙 업데이트용
  Kali(57.10) ── soc-wan ──[ OPNsense ]── soc-dmz ── victim(59.20)
                         바깥으로 나가는 길은 이 한 곳뿐
```

위 그림에서 Kali와 victim에는 **인터넷으로 직접 나가는 선이 없습니다.** 오늘 두 VM의 NAT 어댑터를 모두 끄고, 바깥과 통하는 길을 **방화벽 한 곳으로 모으는** 것이 배선 작업의 핵심입니다. 그래야 "모든 트래픽은 방화벽을 지나간다"는 전제가 성립하고, 2-3편에서 Suricata가 그 길목에 서서 공격을 읽어 낼 수 있습니다. 인터넷 쪽 다리(NAT)는 방화벽만 가지며, Suricata 규칙을 내려받는 용도로만 씁니다.

### ★ 중요: VirtualBox 망 이름과 OPNsense 인터페이스 역할은 별개입니다

OPNsense는 인터페이스마다 **WAN/LAN/OPT1** 이라는 역할 이름을 붙입니다. 이건 VirtualBox 내부망 이름(`soc-wan` 등)과 다른 개념입니다. 아래 매핑을 그대로 따르세요.

| OPNsense 인터페이스 | 연결할 VirtualBox 어댑터        | IP               | 여기 있는 장비                     |
| -------------- | ------------------------- | ---------------- | ---------------------------- |
| **WAN**        | 어댑터 1 = **NAT**           | 10.0.2.15 (자동)   | (없음) 인터넷 — Suricata 규칙 다운로드용 |
| **LAN**        | 어댑터 2 = 내부망 **`soc-wan`** | **192.168.57.1** | Kali(57.10) — 우리의 관리·공격 콘솔   |
| **OPT1**       | 어댑터 3 = 내부망 **`soc-dmz`** | **192.168.59.1** | victim(59.20) — 표적 서버        |

### 잠깐 — 공격망인 `soc-wan`이 왜 하필 "LAN"인가

여기서 한 번은 걸리고 넘어가는 대목입니다. `soc-wan`은 바깥을 흉내 낸 공격망인데, 방화벽에서는 안쪽을 뜻할 것 같은 **LAN** 자리에 앉힙니다. 이유는 이렇습니다.

**첫째, OPNsense의 WAN·LAN·OPT1은 "신뢰도 등급"이 아니라 그냥 역할 슬롯 이름입니다.** 어느 망을 어느 슬롯에 넣을지는 우리가 정하는 것이고, 슬롯 이름이 그 망을 안전하게 만들어 주지도, 위험하게 만들어 주지도 않습니다.

**둘째, 배정 기준은 딱 하나 — 관리자가 어느 망에 앉아 있는가입니다.** OPNsense는 설치 직후 ① `Default allow LAN to any` 규칙과 ② **잠금 방지(anti-lockout) 규칙**을 **LAN에만** 자동으로 만들어 줍니다. 나머지 인터페이스는 규칙이 하나도 없는 상태, 즉 **전부 차단**입니다. 우리는 Kali의 브라우저로 방화벽을 관리하므로, Kali가 있는 `soc-wan`을 LAN 슬롯에 두는 것입니다.

**셋째, 반대로 하면 스스로 잠깁니다.** Kali 망을 WAN 슬롯에 물리면 `https://192.168.57.1` 접속부터 기본 차단입니다. 그런데 그 차단을 풀 규칙을 넣으려면 바로 그 관리 화면이 필요하고, 콘솔 메뉴에는 규칙 편집기가 없습니다. 초보자가 가장 자주 만나는 "방화벽에 스스로 갇히는" 상황이 이것입니다.

진짜 WAN 슬롯은 VirtualBox NAT에 주어, **Suricata 규칙을 내려받는 인터넷 통로**로만 씁니다.

> 대신 이 배치에는 되짚어 볼 점이 하나 있습니다. 공격망이 LAN 자리에 앉으면서 **기본 허용**을 받는다는 점, 즉 **지금 방화벽은 공격자를 신뢰하고 있다**는 점입니다. 2-2에서 할 일이 바로 이 기본 허용을 걷어내고 필요한 것만 남기는 것입니다. 표적 서버는 처음부터 별도 구간(OPT1)에 두어 규칙으로 통제합니다.
{: .prompt-warning }

> 방화벽이 각 망에서 쓰는 IP(공격망 57.1 · 표적망 59.1)는 1-1 기준표와 같습니다. **역할 이름(WAN/LAN/OPT1)만 OPNsense 방식으로 새로 붙는 것**이니, 기준표의 IP는 그대로 외우시면 됩니다.
{: .prompt-tip }

> **관제망(`soc-lan`, 192.168.58.0/24)은 오늘 연결하지 않습니다.** 여기에 들어갈 SIEM 서버(58.100)는 4단계에서 만들므로, 그때 이 방화벽에 **어댑터 4(OPT2 = 192.168.58.1)** 를 하나 더 붙입니다. 오늘은 다리 세 개만 놓습니다.
{: .prompt-info }

---

## 5. 따라 하기: OPNsense 설치 이미지 준비

> ### 따라 하기 5-1. OPNsense ISO 내려받기
>
> **목적** 방화벽 운영체제 설치 파일을 준비합니다.
{: .prompt-tip }

**1단계.** (호스트 PC 브라우저에서) 공식 다운로드 페이지로 이동합니다.

```text
https://opnsense.org/download/
```

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787611608481.png)

**2단계.** 다음과 같이 고른 뒤 내려받습니다.

| 항목 | 선택 |
| --- | --- |
| Architecture | **amd64** |
| Image type | **dvd** |
| Mirror | 아무 곳(가까운 곳) |

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787611631154.png)

**3단계.** 받은 파일은 `.iso.bz2`(압축)입니다. **7-Zip** 등으로 압축을 풀어 `.iso` 파일로 만듭니다.

---

## 6. 따라 하기: OPNsense VM 만들기 (어댑터 3개)

> ### 따라 하기 6-1. 빈 VM 생성
>
> **목적** 방화벽용 가상머신을 만듭니다.
{: .prompt-tip }

**1단계.** VirtualBox → **새로 만들기(New)**.

| 항목 | 값 |
| --- | --- |
| 이름 | **`firewall`** |
| ISO 이미지 | 앞서 준비한 OPNsense `.iso` |
| 종류 | BSD |
| 버전 | FreeBSD (64-bit) |
| 자동 설치 건너뛰기 | 체크 |
![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787612221909.png)


**2단계.** 하드웨어.

| 항목 | 값 |
| --- | --- |
| 메모리 | **`3072` MB** |
| 프로세서 | **`2`** |
| 디스크 | **`20` GB** |

> **메모리를 3072MB로 잡는 두 가지 이유가 있습니다.**
>
> 첫째, **설치가 막힙니다.** 2048MB로 두면 설치 화면에서 `The installer detected only 2047MB of RAM ... requires at least 3000MB` 경고가 뜹니다. 설치 이미지를 디스크로 통째로 복사하는 작업이라 메모리가 필요합니다.
>
> 둘째, 2-3편에서 **Suricata가 탐지 규칙 8,000여 개를 메모리에 올립니다.** 2048MB로도 동작은 하지만 빠듯합니다.
>
> 프로세서를 2로 주는 것도 Suricata 때문입니다. **멀티스레드**로 동작하므로 1개면 IPS 모드에서 웹 응답이 눈에 띄게 느려집니다. CPU 개수는 메모리와 달리 호스트 자원을 미리 잡아 두지 않으므로 2로 두어도 부담이 없습니다.
{: .prompt-tip }

> ### 따라 하기 6-2. 네트워크 어댑터 3개 지정
>
> **목적** 위 매핑표(4장)대로 방화벽의 세 다리를 연결합니다.
{: .prompt-tip }

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787612249356.png)

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787612264799.png)


**3단계.** `firewall` VM → **설정 → 네트워크**에서 어댑터 3개를 각각 켜고 설정합니다. **순서가 중요합니다.**

| 탭 | 다음에 연결됨 | 이름 |
| --- | --- | --- |
| 어댑터 1 | **NAT** | — |
| 어댑터 2 | 내부 네트워크 | **`soc-wan`** |
| 어댑터 3 | 내부 네트워크 | **`soc-dmz`** |
![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787612321049.png)

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787612329657.png)


---

## 7. 따라 하기: OPNsense 설치

> ### 따라 하기 7-1. 라이브 부팅 후 설치 시작
>
> **목적** 디스크에 OPNsense를 설치합니다.
{: .prompt-tip }

**1단계.** `firewall` VM을 **시작**합니다. 부팅이 끝나면 로그인 프롬프트가 나옵니다. 설치 계정으로 로그인합니다.

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787612458027.png)

| 항목 | 값 |
| --- | --- |
| login | `installer` |
| password | `opnsense` |

**2단계.** 설치 마법사가 뜹니다. 다음 순서로 진행합니다.

| 화면 | 선택 |
| --- | --- |
| 키맵(Keymap) | 기본값 → **Continue** |
| 설치 방식 | **`Install (UFS)`** |
| 디스크 선택 | 20GB 디스크 선택 → OK |
| (확인 경고) | Yes |

> 설치 메뉴에는 `Install (ZFS)`·`Install (UFS)`·`Import Config`·`Password Reset` 등이 함께 나옵니다. **`Install (UFS)`** 를 고르세요. ZFS는 메모리를 더 많이 쓰는 파일 시스템이라 실습용 VM에는 맞지 않습니다.
>
> 여기서 `The installer detected only 2047MB of RAM` 경고가 뜬다면 VM 메모리가 2048MB로 되어 있는 것입니다. **`Cancel` → `Force Halt`** 로 끄고, VirtualBox 설정에서 메모리를 **3072MB** 로 올린 뒤 다시 시작하세요. `Proceed anyway` 로 밀어붙이면 복사 도중 실패해 반쯤 설치된 상태가 될 수 있습니다.
{: .prompt-warning }

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787612504705.png)

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787612536489.png)

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787612640336.png)

**3단계.** 설치가 끝나면 **root 비밀번호 설정** 화면이 나옵니다. 기준표대로 입력합니다.

| 항목 | 값 |
| --- | --- |
| root password | **`opnsense123`** |
==> 직접 설정해도 무방함함

**4단계.** **Complete Install → Reboot** 를 선택합니다. 재부팅이 시작되면 VirtualBox 창의 **장치 → 광학 드라이브 → 가상 드라이브에서 디스크 꺼내기**로 설치 ISO를 빼 둡니다.

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787805812817.png)

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787806292421.png)


> **★ 여기서 반드시 확인하세요 — 설치가 정말 됐는지** ISO를 빼지 않으면 디스크가 아니라 **ISO로 다시 부팅**되어, 겉보기에는 똑같이 동작하지만 실제로는 **라이브 모드(live media mode)** 입니다. 라이브 모드에서는 인터페이스 할당·IP·방화벽 규칙이 **전부 메모리에만 저장되어 재부팅하면 모두 사라집니다.** 오늘 작업을 통째로 다시 해야 하므로, 다음 두 가지로 꼭 확인하세요.
>
> - 콘솔 로그인 계정이 `installer` 로 되면 **아직 설치 전**입니다. 설치가 끝났다면 `root` 로만 로그인됩니다.
> - 11장에서 웹 관리 화면에 들어갔을 때 위쪽에 파란 배너로 **"You are currently running in live media mode"** 가 보이면 **설치가 안 된 것**입니다. VM을 끄고 광학 드라이브에서 ISO를 제거한 뒤, 이 장(7장)을 처음부터 다시 하세요.
{: .prompt-danger }

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787612849138.png)

---

## 8. 따라 하기: 콘솔에서 인터페이스·IP 지정

재부팅되면 OPNsense 콘솔 메뉴가 나옵니다. 여기서 세 인터페이스의 역할과 IP를 정합니다. 콘솔에 로그인합니다: `root` / **`opnsense123`** — 7-1 3단계에서 직접 정한 비밀번호입니다.

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787613198224.png)

> 설치 직후에는 OPNsense가 임의로 **`LAN (em0) -> 192.168.1.1/24`**, **`WAN (em1)`** 처럼 배정해 둡니다. 위 화면이 그렇게 보이는 것이 정상이며, 지금부터 이것을 우리 기준표대로 다시 배정합니다.
{: .prompt-info }

> ### 따라 하기 8-1. 인터페이스 할당 (메뉴 1)
>
> **목적** 어느 랜카드가 WAN/LAN/OPT1인지 정합니다.
{: .prompt-tip }

**1단계.** 메뉴에서 **`1`** (Assign interfaces)을 입력하고 Enter.

- `Do you want to configure LAGGs now?` → **`n`**
- `Do you want to configure VLANs now?` → **`n`**
- **WAN** 으로 쓸 인터페이스 이름 입력 → **`em0`** (어댑터 1 = NAT)
- **LAN** 으로 쓸 인터페이스 이름 입력 → **`em1`** (어댑터 2 = soc-wan)
- **Optional 1(OPT1)** 이름 입력 → **`em2`** (어댑터 3 = soc-dmz)
- 그 다음 Optional은 비워 두고 Enter
- `Do you want to proceed?` → **`y`**


![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787618239193.png)

> em0/em1/em2 순서는 VirtualBox 어댑터 1/2/3 순서와 같습니다. 만약 인터넷(WAN)이 나중에 안 되면 이 할당을 다시 확인하세요.
{: .prompt-warning }


> ### 따라 하기 8-2. LAN IP 설정 (메뉴 2)
>
> **목적** Kali가 접속할 관리망 IP를 정합니다.
{: .prompt-tip }

**2단계.** 메뉴에서 **`2`** (Set interface IP address) 입력 → 목록에서 **LAN** 선택.


| 물음 | 답 |
| --- | --- |
| Configure IPv4 by DHCP? | **`n`** |
| IPv4 address | **`192.168.57.1`** |
| Subnet bit count | **`24`** |
| Upstream gateway (WAN 아님) | (그냥 Enter, 없음) |
| Configure IPv6? | **`n`** |
| Enable DHCP server on LAN? | **`n`** |
| Change web GUI protocol to HTTP? | **`n`** (HTTPS 유지) |
| Generate a new self-signed web GUI certificate? | **`n`** |
| Restore web GUI access defaults? | **`n`** |

> 버전에 따라 위 물음의 개수와 순서가 조금 다를 수 있습니다. **표에 없는 물음이 나오면 모두 `n`(또는 그냥 Enter)** 으로 넘기면 됩니다. 특히 마지막 `Restore web GUI access defaults?` 에 `y` 를 누르면 관리 접속 규칙이 초기화되므로 반드시 `n` 입니다.
{: .prompt-warning }


**3단계.** 다시 **`2`** 입력 → 이번엔 **OPT1** 선택. 같은 방식으로 표적망 IP를 정합니다.

| 물음 | 답 |
| --- | --- |
| Configure IPv4 by DHCP? | **`n`** |
| IPv4 address | **`192.168.59.1`** |
| Subnet bit count | **`24`** |
| Upstream gateway | (Enter, 없음) |
| Configure IPv6? | **`n`** |
| Enable DHCP server? | **`n`** |
| (그 밖의 물음) | 모두 **`n`** |

**4단계.** WAN(em0)은 **NAT에서 자동(DHCP)** 으로 IP를 받으므로 따로 설정하지 않습니다. 콘솔 상단에 `WAN (em0) -> v4/DHCP4: 10.0.2.15/24` 처럼 표시되면 정상입니다.

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787618375416.png)

세 줄이 위 화면처럼 모두 보이면, 방화벽의 다리 세 개가 제자리를 잡은 것입니다.

---

## 9. 따라 하기: Kali를 공격망으로 이사

이제 Kali를 바깥 공격망(soc-wan)으로 옮기고, IP를 57.10으로 바꿉니다. **OPNsense를 통해 인터넷에 나가므로, Kali의 NAT 어댑터는 끕니다.**

> ### 따라 하기 9-1. Kali VM 어댑터 변경 (VM 끈 상태에서)
>
> **목적** Kali를 soc-wan 한 곳에만 연결합니다.
{: .prompt-tip }

**1단계.** Kali가 켜져 있으면 종료합니다. VirtualBox에서 `attacker`(Kali) → **설정 → 네트워크**:

| 탭     | 변경                                |
| ----- | --------------------------------- |
| 어댑터 1 | **"네트워크 어댑터 사용하기" 체크 해제** (NAT 끔) |
| 어댑터 2 | 내부 네트워크, 이름 **`soc-wan`** 으로 변경   |
![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787614804197.png)


![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787614795784.png)


> ### 따라 하기 9-2. Kali 고정 IP·게이트웨이 재설정
>
> **목적** 57.10 주소와 기본 경로(OPNsense)를 지정합니다.
{: .prompt-tip }

**2단계.** Kali를 켜고 로그인한 뒤 터미널에서 인터페이스 이름을 확인합니다.

```bash
ip -brief address
```

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787618500605.png)

**3단계.** 1단계에서 만든 임시 연결을 지우고, 새 연결을 만듭니다. (아래 `eth0`은 확인한 실제 이름으로 바꾸세요. NAT를 껐으므로 내부망 카드가 보통 `eth0`으로 하나만 남습니다.)

```bash
sudo nmcli connection delete soc-dmz 2>/dev/null
sudo nmcli connection add type ethernet ifname eth0 con-name soc-wan \
  ip4 192.168.57.10/24 gw4 192.168.57.1
sudo nmcli connection modify soc-wan ipv4.dns 192.168.57.1 ipv4.method manual
sudo nmcli connection up soc-wan
```

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787614584808.png)

**4단계.** 확인합니다.

```bash
ip -brief address
ping -c 2 192.168.57.1
```

`192.168.57.10/24` 가 보이고 게이트웨이(57.1)에 응답이 오면 성공입니다.

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787614906499.png)

---

## 10. 따라 하기: victim을 방화벽 뒤로 넣기

victim은 지금 바깥과 두 갈래로 통하고 있습니다. 하나는 1-3편에서 설치용으로 붙여 둔 **어댑터 1(NAT)**, 다른 하나는 표적망(`soc-dmz`)입니다. 4장 그림처럼 **바깥으로 나가는 길을 방화벽 하나로 모으려면**, NAT 다리를 끊고 게이트웨이를 59.1로 잡아 주어야 합니다.

> ### 따라 하기 10-1. victim의 NAT 어댑터 끄기 (VM 끈 상태에서)
>
> **목적** victim이 방화벽을 거치지 않고 바깥에 나가는 샛길을 없앱니다.
{: .prompt-tip }

**1단계.** victim이 켜져 있으면 터미널에서 `sudo poweroff` 로 종료합니다.

**2단계.** VirtualBox에서 `victim` → **설정 → 네트워크**:

| 탭 | 변경 |
| --- | --- |
| 어댑터 1 | **"네트워크 어댑터 사용하기" 체크 해제** (NAT 끔) |
| 어댑터 2 | 그대로 둡니다 (내부 네트워크 `soc-dmz`) |

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787618645615.png)

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787618653589.png)


**3단계.** victim을 다시 켜고 로그인한 뒤, 랜카드 이름을 확인합니다.

```bash
ip -brief address
```

> **이름은 그대로 `enp0s8` 입니다.** Kali는 NAT를 끄자 이름이 `eth1`에서 `eth0`으로 당겨졌지만(9장), Ubuntu Server는 랜카드가 꽂힌 **슬롯 번호**로 이름을 짓기 때문에 어댑터 1을 꺼도 어댑터 2는 계속 `enp0s8` 입니다. 같은 상황에서 두 배포판의 이름이 다르게 움직이는 이유입니다.
{: .prompt-tip }

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787618742594.png)

> ### 따라 하기 10-2. 게이트웨이 지정
>
> **목적** 바깥으로 나가는 문(59.1)이 어디인지 알려 줍니다.
{: .prompt-tip }

**1단계.** victim 서버 터미널에서 netplan 파일을 다시 씁니다. (인터페이스 이름 `enp0s8`은 방금 확인한 실제 이름으로.)

```bash
sudo tee /etc/netplan/99-soc.yaml >/dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: no
      addresses: [192.168.59.20/24]
      routes:
        - to: default
          via: 192.168.59.1
      nameservers:
        addresses: [192.168.59.1]
EOF
sudo chmod 600 /etc/netplan/99-soc.yaml
sudo netplan apply
```

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787618896276.png)




**2단계.** 경로가 제대로 잡혔는지 확인합니다.

```bash
ip route
```

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787618955511.png)


`default via 192.168.59.1 dev enp0s8 proto static` **한 줄만** 남아 있으면 성공입니다. 이제 victim이 바깥으로 나가는 길은 방화벽뿐이며, 4장 그림과 정확히 같은 모양이 됐습니다.

위 화면은 **NAT 어댑터를 끄기 전**에 찍은 것이라 `default via 10.0.2.2 ... metric 100`(어댑터 1) 이 함께 보입니다. 이렇게 기본 경로가 둘이면 **metric 값이 작은 쪽이 이기므로** 통신 자체는 되지만, 바깥으로 나가는 문이 두 개라 그림과 어긋납니다. 10-1에서 NAT를 껐다면 아래쪽 줄들은 보이지 않아야 합니다.

**3단계.** 게이트웨이에 ping을 해 봅니다.

```bash
ping -c 2 192.168.59.1
```

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619009290.png)

> **여기서 ping이 실패하는 것이 정상입니다.** 원인은 victim이 아니라 방화벽에 있습니다. OPNsense는 설치 직후 "기본 허용" 규칙을 **LAN에만** 만들어 줍니다(관리 화면 잠금 방지용). 방금 IP만 지정한 **OPT1은 규칙이 하나도 없는 상태**이고, 규칙이 없으면 **전부 차단**이 기본값입니다. 방화벽 자신(59.1)에게 가는 ping도 예외가 아닙니다. 이것이 방화벽의 가장 중요한 성질인 **기본 차단(default deny)** 입니다. 11장에서 GUI에 들어가 허용 규칙을 넣고 다시 확인하겠습니다.
{: .prompt-warning }

> **선을 잘못 꽂은 것과 구분하는 법** — victim에서 `ip neigh show 192.168.59.1` 을 해 보면 MAC 주소(`lladdr ...`)가 보입니다. 상대가 같은 망에 살아 있다는 뜻이므로, 망 이름·IP·랜카드는 모두 정상이고 **방화벽이 버리고 있을 뿐**입니다. (주소 확인용 ARP는 방화벽 규칙의 검사 대상이 아니어서 이런 차이가 생깁니다.)
{: .prompt-tip }

> 샛길을 끊었으므로, 이제 victim의 인터넷은 **전적으로 방화벽에 달려 있습니다.** 11장에서 OPT1 허용 규칙을 넣기 전까지는 `apt` 도 이름 풀이도 되지 않습니다. 규칙을 넣으면 OPNsense가 인터넷 쪽으로 대신 내보내 주므로(자동 아웃바운드 NAT) 다시 살아납니다. Juice Shop은 이미 떠 있으니 그때까지 실습에 지장은 없습니다.
{: .prompt-info }


---

## 11. 따라 하기: Kali 브라우저로 OPNsense 관리 화면 접속

> ### 따라 하기 11-1. 웹 GUI 로그인
>
> **목적** 앞으로 규칙을 설정할 관리 화면에 들어갑니다.
{: .prompt-tip }

**1단계.** Kali의 Firefox 주소창에 입력합니다.

```text
https://192.168.57.1
```

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619072793.png)


**2단계.** "안전하지 않음" 경고가 나오면 **고급 → 위험을 감수하고 계속** 을 누릅니다. (실습용 자체 서명 인증서라 정상입니다.)

**3단계.** 로그인: `root` / `opnsense123`.

> 로그인한 뒤 화면 위쪽에 파란 배너로 **"You are currently running in live media mode. A reboot will reset the configuration."** 가 보이면, 7장의 설치가 끝나지 않았거나 ISO로 다시 부팅된 것입니다. 이대로 진행하면 오늘 한 설정이 재부팅과 함께 모두 사라집니다. VM을 끄고 광학 드라이브에서 ISO를 제거한 뒤 **7장부터 다시** 하세요.
{: .prompt-danger }

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619092712.png)


![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787806640220.png)


**4단계.** 처음 로그인하면 설정 마법사(Wizard)가 나옵니다. 대부분 **Next** 로 넘기고, 다음만 확인합니다.

- **General**: Hostname `firewall`, Domain `soc.lab`, **DNS server 1 = `8.8.8.8`**, **DNS server 2 = `1.1.1.1`** ★ 이 칸을 비워 두면 방화벽이 도메인 이름을 풀지 못해, 2-3편에서 Suricata 규칙을 내려받을 때 실패합니다
  
- **Timezone**: **`Asia/Seoul`** ★ 4단계에서 SIEM이 방화벽·Suricata·서버 로그의 **시각을 맞춰 사건을 엮기** 때문에, 지금 정확히 지정해 두어야 합니다
  
  ![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619186255.png)
  
- **WAN 인터페이스**: "Block private networks", "Block bogon networks" 체크 — 지금은 실습이므로 **둘 다 체크 해제** (WAN이 NAT 사설망이라 막으면 인터넷이 끊깁니다.)
  
  ![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619216653.png)
  
- **LAN 인터페이스**: 이미 콘솔에서 잡은 **192.168.57.1 / 24** 가 그대로 보여야 합니다. 값을 바꾸지 말고 Next
  
  ![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619237706.png)
  
- **Root Password**: 비워 두면 기존 값(`opnsense123`) 유지
  
  ![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619266278.png)
  
- 마지막 **Reload** 로 마침(Apply)


> ### 따라 하기 11-2. 차단되고 있는 장면을 눈으로 확인
>
> **목적** 10장에서 ping이 실패한 이유를 방화벽 로그에서 직접 봅니다.
{: .prompt-tip }

**1단계.** GUI 메뉴에서 **Firewall → Log Files → Live View** 로 들어갑니다.

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619339745.png)

**2단계.** 화면을 켜 둔 채, victim 터미널에서 `ping -c 2 192.168.59.1` 을 다시 실행합니다.

**3단계.** 목록에 빨간 표시로 차단 기록이 올라옵니다. 그 줄을 펼치면 적용된 규칙 이름이 **`Default deny / state violation rule`** 로 나옵니다. "허용 규칙이 없어서 기본값(차단)이 적용됐다"는 뜻입니다.



> ### 따라 하기 11-3. OPT1에 허용 규칙 넣기
>
> **목적** 표적망(soc-dmz)이 방화벽을 지나갈 수 있게 문을 엽니다.
{: .prompt-tip }

**1단계.** 좌측 메뉴 **Firewall → Rules** 로 갑니다. 화면 왼쪽 위의 **`All rules`** 드롭다운에서 **`OPT1`** 을 고릅니다.

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619697961.png)

> **처음 보면 헷갈리는 화면입니다.** 규칙 목록은 두 묶음으로 나뉘어 있습니다. 위쪽 **`Automatically generated rules`** 는 OPNsense가 스스로 만든 것이라 **건드리지 않습니다.** 우리가 다루는 것은 아래쪽 **`Interface rules`** 입니다. 지금 여기에는 **LAN 두 줄(IPv4·IPv6)밖에 없습니다.** 그것이 2-1에서 말한 "기본 허용은 LAN에만 있다"는 뜻이고, **OPT1은 한 줄도 없어서 전부 차단**인 것입니다.
{: .prompt-tip }

**2단계.** 목록 **오른쪽 아래**에 있는 주황색 **`+`** 버튼을 누릅니다. (위쪽이 아니라 목록 맨 아래 줄의 오른쪽 끝입니다.)

**3단계.** 열리는 폼에서 다음 항목만 채우고 나머지는 기본값으로 둡니다.

| 항목             | 값                       |
| -------------- | ----------------------- |
| Action         | **Pass**                |
| Interface      | **OPT1**                |
| Direction      | **in**                  |
| TCP/IP Version | IPv4                    |
| Protocol       | **any**                 |
| Source         | **OPT1 net**            |
| Destination    | **any**                 |
| Description    | `OPT1 default allow`    |
![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619871713.png)

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619888041.png)


**4단계.** **Save** 를 누른 뒤, 목록 아래쪽에 나타나는 주황색 **`Apply`** 버튼을 반드시 누릅니다. 이걸 누르지 않으면 규칙이 저장만 되고 **적용되지 않습니다.** 규칙을 넣었는데 아무 변화가 없다면 열에 아홉은 이것을 빠뜨린 경우입니다.

> 지금 넣은 것은 표적망을 **활짝 열어 두는** 규칙입니다. 실무의 DMZ를 이렇게 두면 안 되지만, 오늘의 목표는 "트래픽이 방화벽을 지나가게 만드는 것"이므로 일부러 넓게 열어 둡니다. **2-2에서 필요한 것만 남기고 조입니다.**
{: .prompt-warning }


> ### 따라 하기 11-4. victim에서 다시 확인
>
> **목적** 규칙 한 줄로 막혀 있던 통신이 열리는 것을 확인합니다.
{: .prompt-tip }

**1단계.** victim 터미널에서 10장에서 실패했던 명령을 그대로 다시 실행합니다.

```bash
ping -c 2 192.168.59.1
```

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619920935.png)


이번에는 응답이 옵니다. 달라진 것은 규칙 한 줄뿐입니다.

**2단계.** 바깥으로 나가는 길도 살아났는지 봅니다.

```bash
ping -c 2 8.8.8.8
```

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787619940375.png)

응답이 오면, victim이 **OPNsense를 게이트웨이로 삼아 인터넷까지 나가고 있는 것**입니다. (OPNsense가 WAN 쪽으로 주소를 바꿔 내보내 줍니다.)

**3단계.** 이름 풀이까지 되는지 확인합니다.

```bash
ping -c 2 www.google.com
```

> **여기서 막히면 DNS 설정 문제입니다.** 주소(`8.8.8.8`)는 되는데 이름(`www.google.com`)이 안 된다면, 길은 뚫려 있고 **이름을 물어볼 곳이 없는** 상태입니다. victim은 이름 풀이를 방화벽(59.1)에 맡기고 있으므로, 방화벽이 못 풀면 victim도 못 풉니다. 방화벽 GUI **System → Settings → General** 에서 **DNS server 1 = `8.8.8.8`**, **DNS server 2 = `1.1.1.1`** 을 넣고 **Save** 하세요. 그래도 안 되면 **Services → Unbound DNS → Query Forwarding** 에서 **`Use System Nameservers`** 를 체크하고 Save → Apply 합니다.
>
> 방화벽 자신의 상태만 따로 보려면 **Interfaces → Diagnostics → Ping** 에서 `8.8.8.8` 과 `www.google.com` 을 각각 보내 보세요. 앞의 것만 되면 위 설정이 빠진 것이고, 둘 다 `loss 0.0%` 면 정상입니다. 확인 뒤에는 **Jobs** 탭에서 `■` 로 작업을 멈춰 둡니다.
{: .prompt-warning }

---

## 12. 따라 하기: 아직은 공격이 통하는지 확인

방화벽을 세웠지만, **기본 규칙은 관리망(LAN=Kali)에서 나가는 트래픽을 대부분 허용**하고, 표적망(OPT1)도 방금 넓게 열어 두었습니다. 그래서 지금은 공격이 여전히 통합니다. 이걸 확인하고, 다음 시간(2-2)에 규칙으로 막습니다.

**1단계.** Kali 터미널에서 1단계와 **똑같은 명령**을 실행합니다.

```bash
ping -c 2 192.168.59.20
nmap -sV -T4 192.168.59.20
```

![](/assets/img/posts/2027-06-28-socbuild-04-opnsense-install-1787620083046.png)

- ping이 응답하고, nmap에 `22`(ssh) 와 `3000`(HTTP·Juice Shop) 이 여전히 `open` 으로 보입니다. (3000의 `SERVICE` 가 `ppp?` 로 나오는 건 nmap이 포트 번호로 붙인 이름표가 애매한 것일 뿐, 1단계 표적 스캔 때 본 대로 실제로는 HTTP입니다.)
- 다른 점: 이제 이 트래픽은 **OPNsense를 통과**하고 있습니다. 다음 시간에 이 길목에서 막습니다.
- 4장 그림의 **빨간 화살표(공격) → 방화벽 → 파란 화살표 → victim** 이 실제로 그렇게 흐르고 있는 것입니다. 2-3편에서는 이 길목에 Suricata를 세워, 화살표 한가운데에서 공격 패턴을 읽어 냅니다.

> 만약 victim에 닿지 않으면: ① Kali gw가 57.1인지, ② victim gw가 59.1인지, ③ OPNsense OPT1(em2, 59.1)이 올라왔는지, ④ 11-3에서 **Apply changes** 를 눌렀는지 순서로 확인하세요.
{: .prompt-warning }

---

## 13. 따라 하기: 설정 백업 — 오늘 한 일을 파일 하나에 담기

방화벽 설정은 **파일 하나로 통째로 저장하고 되돌릴 수 있습니다.** 실무에서 방화벽을 만지기 전에 반드시 하는 일이고, 실습에서도 뭔가 꼬였을 때 처음부터 다시 하지 않게 해 주는 안전장치입니다.

> ### 따라 하기 13-1. 설정 내려받기
>
> **목적** 오늘 만든 설정을 파일로 보관합니다.
{: .prompt-tip }

**1단계.** Kali 브라우저의 OPNsense GUI에서 **System → Configuration → Backups**.

**2단계.** **Download configuration** 을 누릅니다. 암호화 옵션은 체크하지 않습니다.

**3단계.** `config-firewall.xml` 같은 파일이 Kali에 저장됩니다. 이 파일 하나에 **인터페이스 배정·IP·방화벽 규칙·별칭·DNS 설정**이 모두 들어 있습니다.

> 되돌릴 때는 같은 화면의 **Restore configuration** 에서 이 파일을 올리면 됩니다. 방화벽이 재부팅되면서 저장 시점 상태로 돌아갑니다. 단, Suricata 규칙 파일(`.rules`)처럼 용량이 큰 내려받은 자료는 백업에 들어가지 않으므로, 복원 뒤 2-3편의 규칙 다운로드는 다시 해 주어야 합니다.
{: .prompt-info }

> **★ 관리 화면 위쪽에 파란 배너 `You are currently running in live media mode` 가 보인다면**, 지금 방화벽은 **디스크가 아니라 설치 ISO로 돌고 있는 상태**입니다. 재부팅하면 설정이 사라지는 것은 물론, 더 고약한 일이 먼저 벌어집니다. 라이브 모드는 메모리에 만든 **2GB짜리 임시 디스크**로 동작하는데, 2-3편에서 탐지 규칙 8,000여 개를 내려받는 순간 **그 공간이 꽉 차서 설정 저장 자체가 실패**합니다. 화면에서는 바꾼 것처럼 보이는데 실제로는 기록되지 않아, "설정이 안 먹는다"는 증상으로 나타납니다.
>
> **Lobby → Dashboard** 에서 두 가지만 보면 바로 확인됩니다.
>
> | 볼 것 | 정상 | 위험 신호 |
> | --- | --- | --- |
> | **Disk** | 여유 있음 | **100%** — 더 이상 아무것도 저장되지 않음 |
> | **Last configuration change** | 방금 전 시각 | 현재 시각보다 **한참 전** — 그 뒤 설정이 전부 저장 실패 |
>
> 이 경우 순서는 이렇습니다.
>
> 1. **먼저 위 13-1로 설정을 내려받으세요**(가장 중요 — 이걸 건너뛰면 전부 다시 해야 합니다)
> 2. 7장으로 돌아가 **디스크에 설치**하고, 재부팅될 때 **ISO를 반드시 빼세요**
> 3. 8장의 콘솔 작업(인터페이스 배정 + LAN IP `192.168.57.1`)만 다시 해서 GUI에 접속할 수 있게 만듭니다
> 4. **System → Configuration → Backups → Restore configuration** 으로 ①에서 받은 파일을 올립니다. 나머지 설정이 통째로 돌아옵니다
{: .prompt-danger }

---

## 14. 스냅샷과 점검

세 VM을 모두 안전하게 종료한 뒤(Kali·victim은 `sudo poweroff`, OPNsense는 콘솔 메뉴 **`5` Power off system** — `6`은 재부팅이므로 주의하세요), 각 VM에 스냅샷을 만듭니다: OPNsense `fw-installed`, Kali `on-wan`, victim `behind-fw`.

점검 목록:

- [ ] 방화벽·IDS·IPS의 차이를 한 문장씩으로 설명할 수 있다
- [ ] OPNsense가 무엇이고 왜 쓰는지 말할 수 있다
- [ ] OPNsense 콘솔에 WAN(10.0.2.x)·LAN(57.1)·OPT1(59.1)이 표시된다
- [ ] Kali가 **57.10** 으로 바뀌고 57.1에 ping이 된다
- [ ] victim의 NAT 어댑터를 끄고, 게이트웨이가 **59.1** 로 설정됐다
- [ ] victim의 `ip route` 에 기본 경로가 **한 줄만** 남았다
- [ ] Kali 브라우저에서 `https://192.168.57.1` GUI 로그인이 된다
- [ ] "규칙이 없으면 차단(default deny)"이 무슨 뜻인지 설명할 수 있다
- [ ] OPT1에 허용 규칙을 넣은 뒤 victim에서 `ping 192.168.59.1` 이 응답한다
- [ ] Kali에서 `nmap 192.168.59.20` 이 (아직은) 통한다
- [ ] 관리 화면에 **live media mode 배너가 없다**(= 디스크에 설치됨)
- [ ] 설정 백업 파일(`config-*.xml`)을 내려받아 두었다

---

## 다음 시간

트래픽이 방화벽을 통과하게 됐지만 아직 막지는 않았습니다. **2-2** 에서는 방화벽 규칙을 **"필요한 것만 허용, 나머지는 차단"** 으로 바꿔, 오늘 통했던 nmap 스캔에서 **웹 포트(3000) 외에는 모두 막히는** 것을 직접 확인합니다.
