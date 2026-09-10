---
title: "[AI 보안관제 구축] 2-3 부록(선택). Suricata IPS 전환 — 탐지를 차단으로"
date: 2027-07-12 18:00:00 +0900
categories:
  - 1.응용강의
  - AI보안관제구축
tags:
  - 보안관제
  - Suricata
  - IPS
  - 침입차단
  - 선택부록
pin: false
math: false
mermaid: false
---

> **이 편은 선택(부록)입니다.** 건너뛰어도 3단계 이후 진행에 전혀 지장이 없습니다.
> 전제 상태: [2-3. Suricata IDS](/posts/socbuild-06-suricata-ids/) 를 끝내고, **Alerts 탭에 스캔·SQLi 경보가 뜨는 것을 눈으로 확인**한 상태.
> 오늘의 목표: 같은 엔진·같은 규칙의 동작을 `alert` 에서 **`drop`** 으로 바꿔, 공격 패킷이 **표적에 닿지 못하게** 만듭니다.
> 오늘 켜는 VM: **OPNsense + Kali + victim**.
{: .prompt-info }

## 1. 왜 이것만 따로 떼어 놓았는가

IDS와 IPS는 **같은 엔진, 같은 규칙**을 씁니다. 다른 것은 마지막 동작 한 가지뿐입니다.

| | IDS | IPS |
| --- | --- | --- |
| 검사 | 같음 | 같음 |
| 규칙 | 같음(ET Open) | 같음 |
| 공격을 만나면 | **`alert`** — 기록하고 흘려보냄 | **`drop`** — 기록하고 버림 |
| 트래픽 위치 | 경로 **옆**에서 복사본 관찰 | 경로 **한가운데**(인라인) |

그런데 "옆에서 보기"를 "한가운데 서기"로 바꾸는 일이 실제로는 만만치 않습니다. **아래 다섯 가지가 모두 맞아야 차단이 일어나고, 하나만 어긋나도 "탐지는 되는데 차단은 안 되는" 상태가 됩니다.**

| # | 조건 | 빠뜨리면 생기는 증상 | 이 강의의 단계 |
| --- | --- | --- | --- |
| ① | 하드웨어 오프로딩 해제 (+재부팅) | 경보가 들쭉날쭉, 차단도 안 됨 | 2-3의 **2-0** |
| ② | `Capture mode` 를 `Divert (IPS)` 로 | 절대 안 막힘 | **3장** |
| ③ | `Divert-to` 방화벽 규칙 생성 | **경보조차 새로 안 쌓임** | **4장** |
| ④ | Policy로 규칙 동작을 `Drop` 으로 | Alerts에 `allowed` 만 남음 | **6장** |
| ⑤ | 기존 연결 기록(state) 초기화 | 설정은 맞는데 안 막힘 | 4·7장 |

> **그래서 본 강의(2-3)는 IDS까지만 다룹니다.** 관제의 목표인 "**공격 → 탐지 → AI 판단 → 자동 차단**" 폐루프에서 Suricata가 맡은 몫은 **탐지와 경보 기록(`eve.json`)** 이고, 차단은 **5단계에서 AI가 OPNsense 방화벽 API로 공격자 IP를 막는 방식**으로 완성되기 때문입니다. 즉 **이 부록을 하지 않아도 폐루프는 닫힙니다.**
>
> 실무의 순서도 같습니다. 인라인 차단은 **오탐 한 건이 곧 서비스 장애**가 되므로, 보통 한동안 IDS로 운영하며 오탐을 걷어낸 뒤 검증된 규칙만 골라 차단으로 올립니다. 이 부록은 그 "올리는 과정"을 실습으로 미리 보는 것입니다.
{: .prompt-tip }

> **★ 시작 전 확인 — OPNsense `26.1` 이상이어야 합니다.**
>
> 3장에서 쓸 **`Capture mode`** 드롭다운(PCAP / Netmap / **Divert**)은 **26.1에서 처음 들어갔습니다.** 그 이전 버전의 Settings 탭에는 드롭다운 대신 **`IPS mode` 체크박스 하나만** 있고, 그것은 **Netmap 전용**입니다. VirtualBox 가상 랜카드에서 Netmap은 통신이 통째로 끊기기 쉬워 따라갈 수 없습니다.
>
> **Lobby → Dashboard** 에서 버전을 확인하세요. `26.1` 미만이면 **System → Firmware → Updates** 로 먼저 업데이트한 뒤 진행합니다.
{: .prompt-danger }

> ### 시작 전 점검 목록
>
> - [ ] 2-3의 **2-0(하드웨어 오프로딩 네 항목 해제 + 재부팅)** 을 했다
> - [ ] Kali에서 `nmap -sA` 와 SQLi curl을 보내면 **Alerts에 경보가 뜬다**
> - [ ] OPNsense 스냅샷 **`suricata-ids`** 를 저장해 두었다 ← **되돌릴 안전망입니다. 없으면 지금 만드세요**
{: .prompt-warning }

---

## 2. 준비: 되돌릴 길을 먼저 확보합니다

IPS는 트래픽 경로 한가운데에 끼어드는 기능이라, 잘못되면 **공격뿐 아니라 정상 통신까지 멈춥니다.** 그래서 **시작하기 전에** 되돌리는 방법을 먼저 알아 둡니다.

**정상 여부를 재는 자**는 이 명령 하나입니다. 앞으로 단계마다 이것으로 확인합니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.59.20:3000
```

`200` 이 나오면 정상 웹은 살아 있는 것입니다. 이 값이 `000`(응답 없음)으로 바뀌면 **정상 통신까지 막힌 것**이므로, 9장의 되돌리기 순서를 따르면 됩니다.

---

## 3. 따라 하기: 가로채기 방식(Capture mode) 고르기

| Capture mode | 성격 | 우리 환경에서 |
| --- | --- | --- |
| **PCAP live mode (IDS)** | 지나가는 트래픽의 **복사본**을 관찰만 합니다. 막지는 못합니다 | 2-3에서 쓴 방식 |
| **Netmap (IPS)** | 랜카드 드라이버 수준에서 패킷을 직접 가로챕니다. 가장 빠르지만 가상 랜카드에서 불안정할 수 있음 | 권장하지 않음 |
| **Divert (IPS)** | 방화벽(pf)이 패킷을 Suricata에게 넘겨주고, 검사 결과에 따라 돌려받습니다 | **오늘 쓸 방식** |

> **왜 Divert인가?** Netmap은 랜카드 드라이버 지원이 필요한데 VirtualBox 가상 랜카드에서는 통신이 끊기거나 느려질 수 있습니다. Divert는 이미 동작 중인 방화벽(pf)이 패킷을 대신 넘겨주므로 **랜카드 종류를 가리지 않습니다.** "경로 한가운데에서 검사하고 공격이면 버린다"는 원리는 똑같습니다.
{: .prompt-tip }

> **하드웨어 오프로딩은 2-3의 2-0에서 이미 껐습니다.** 혹시 건너뛰었다면 지금 **Interfaces → Settings** 로 가서 네 항목(Hardware CRC · TSO · LRO · VLAN Hardware Filtering)을 끄고 **재부팅**한 뒤 이어서 진행하세요. 이것이 안 되어 있으면 아래를 아무리 정확히 해도 차단되지 않습니다.
{: .prompt-warning }

> ### 따라 하기 3-1. IPS로 전환
{: .prompt-tip }

**1단계.** **Services → Intrusion Detection → Administration → Settings** 에서 아래처럼 바꿉니다.

| 항목 | 값 |
| --- | --- |
| Enabled | 체크 (그대로) |
| **Capture mode** | **`Divert (IPS)`** ← **이 칸 하나만 바꿉니다** |
| **Listeners** | **`1`** (Divert를 고르면 새로 나타나는 칸. 기본값 그대로) |
| Home networks | `192.168.59.0/24` (그대로) |

**2단계.** 페이지 맨 아래 **Apply** 를 누릅니다.

> **★ `Interfaces` 칸이 사라졌다고 당황하지 마세요 — 정상입니다. 그리고 이것이 4장이 필요한 이유입니다.**
>
> `Capture mode` 를 `Divert (IPS)` 로 바꾸는 순간 **`Interfaces` 와 `Promiscuous mode` 칸이 화면에서 사라집니다.** OPNsense의 화면 정의 파일에 두 칸은 **pcap·netmap 모드에서만 보이도록** 지정돼 있기 때문입니다.
>
> 왜 없앴을까요? **Divert 모드에는 "어느 인터페이스를 볼지"라는 개념이 아예 없기 때문**입니다. 대신 **"어느 방화벽 규칙에 걸린 트래픽을 볼지"** 로 바뀝니다. 즉 화면이 사라진 그 자리를 **4장에서 만들 `Divert-to` 방화벽 규칙이 대신 채웁니다.**
>
> **그래서 지금 Apply만 누르고 멈추면, Suricata는 검사할 트래픽을 하나도 지정받지 못한 상태입니다.** 반드시 4장까지 하세요.
{: .prompt-danger }

> **`Listeners`** 의 공식 설명은 "보통 CPU 개수만큼"이지만, 실습용 VM에서는 **기본값 1** 로 충분합니다.
{: .prompt-info }

> **Apply를 누르는 순간 관리 화면이 잠깐 끊길 수 있습니다.** Suricata가 다시 뜨면서 생기는 정상적인 순단입니다. 몇 초 기다렸다 새로고침하세요. 오래 끊기면 OPNsense 콘솔에서 메뉴 **`11` (Reload all services)** 을 실행합니다.
{: .prompt-warning }

---

## 4. 따라 하기: (★가장 중요) 검사할 트래픽을 Suricata에게 넘기는 방화벽 규칙

> ### 따라 하기 4-1. `Divert-to` 규칙 만들기
>
> **목적** Divert 모드가 실제로 패킷을 받아 보게 만듭니다.
{: .prompt-tip }

> **★ 여기가 가장 빠뜨리기 쉬운 단계입니다 — 이 규칙이 없으면 Suricata는 패킷을 한 개도 못 봅니다.**
>
> `Capture mode` 를 **`Divert (IPS)`** 로 바꾸는 것만으로는 **아무 일도 일어나지 않습니다.** Netmap과 달리 Divert는 Suricata가 랜카드에 직접 붙는 방식이 아니라, **방화벽이 "이 트래픽을 검사해 줘" 하고 건네줄 때만** 동작하기 때문입니다.
>
> 그 "건네준다"를 지정하는 것이 방화벽 규칙의 **`Divert-to`** 칸입니다. OPNsense 공식 문서도 이렇게 적고 있습니다 — *"Capture mode를 Divert (IPS)로 설정한 뒤, **검사할 트래픽에 맞는 방화벽 규칙을 만들고 그 규칙에 Divert-to를 지정하라**."*
>
> **증상**: 이 단계를 빠뜨리면 IPS로 바꿔도 공격이 그대로 통하고, **Alerts 탭에 경보조차 새로 쌓이지 않습니다.**
{: .prompt-danger }

**1단계.** **Firewall → Rules** 로 갑니다.

> **★ 메뉴 이름이 버전마다 다릅니다.** `Divert-to` 칸은 **새 형식 규칙 화면**에만 있는데, 그 화면의 메뉴 위치가 OPNsense 판올림에 따라 바뀌어 왔습니다. 셋 중 하나로 들어가면 됩니다.
>
> | 메뉴에 이렇게 보이면 | 상황 |
> | --- | --- |
> | **`Firewall → Rules`** 하나만 있음 | 새 화면이 예전 것을 **대체한** 버전(26.x 등). 그대로 들어가면 됩니다 |
> | `Firewall → Rules` 와 **`Rules [new]`** 가 나란히 | 두 화면이 **공존**하는 버전. **`[new]`** 쪽으로 |
> | **`Firewall → Automation → Filter`** | 새 화면이 Automation 아래 있던 버전 |
>
> 셋 다 같은 화면입니다. 들어가서 **`+`** 로 규칙 폼을 열었을 때 **`Divert-to` 칸이 보이면** 제대로 찾은 것입니다. 안 보이면 폼 위쪽의 **advanced mode** 토글을 켜 보세요.
>
> 규칙 목록에서 각 줄 앞에 **`L`** 표시가 붙어 있다면 "예전 형식(legacy)으로 만들어진 규칙"이라는 뜻입니다. 2-2에서 만든 규칙들이 그렇게 보이는 것은 정상이며, **지금 보고 있는 화면이 새 화면이라는 증거**이기도 합니다.
{: .prompt-warning }

**2단계.** **+ Add** 로 규칙을 만듭니다.

| 항목 | 값 |
| --- | --- |
| Enabled | **체크** |
| Action | **Pass** |
| Interface | **LAN** |
| Direction | **in** |
| Protocol | **TCP** |
| Source | **LAN net** |
| Destination | **`victim_host`** (2-2에서 만든 별칭) |
| Destination port | **`web_ports`** (2-2에서 만든 별칭 = 3000) |
| **Divert-to** | **`Intrusion Detection`** ← **오늘의 핵심** |
| Description | `divert web traffic to IDS` |

**3단계.** **Save** → 목록 위쪽 **Apply** 를 누릅니다.

> **`Divert-to` 드롭다운에는 `Intrusion Detection` 하나만 나옵니다.** OPNsense에서 divert 소켓을 여는 서비스가 현재 Suricata뿐이기 때문입니다. 이것이 안 보이면 3장의 `Capture mode` 변경이 **Apply** 되지 않은 것입니다.
{: .prompt-info }

> **2-2에서 만든 LAN 규칙과 충돌하지 않나요?** 걱정하지 않아도 됩니다. OPNsense는 **새 형식 규칙을 예전 형식(목록에 `L` 로 표시되는) 규칙보다 먼저** 처리합니다. 그래서 방금 만든 divert 규칙이 2-2의 "표적 웹 허용" 규칙보다 앞서 걸리고, 3000 포트 트래픽은 **Suricata를 거친 뒤** 통과합니다. 나머지 포트는 2-2의 차단 규칙이 그대로 막습니다.
{: .prompt-tip }

> **★ 규칙을 만든 뒤 반드시 기존 연결 기록(state)을 지우세요.**
>
> 방화벽은 **이미 성립된 연결**은 규칙을 다시 보지 않고 그냥 통과시킵니다(2-1의 상태 기반 원리). 그래서 2-3에서 `curl` 을 이미 실행했다면 그 연결 기록이 남아 있어, **새 divert 규칙이 적용되지 않은 것처럼 보입니다.**
>
> **Firewall → Diagnostics → States** 에서 검색창에 `192.168.59.20` 을 넣고 남아 있는 항목을 지우거나, 화면의 **Reset state table** 을 실행하세요. 이것만으로 "설정은 다 맞는데 안 막힌다"가 풀리는 경우가 많습니다.
{: .prompt-danger }

---

## 5. 따라 하기: (중간 점검) Suricata가 정말 트래픽을 보는지 확인

차단 설정으로 넘어가기 전에, **지금 상태에서 이미 탐지는 되어야 합니다.** Kali에서 SQLi를 한 번 보내고 **Alerts** 에 새 경보가 뜨는지 보세요.

```bash
curl -G --max-time 10 "http://192.168.59.20:3000/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"
```

경보가 뜨면 **Divert 경로가 살아 있는 것**이므로 6장으로 갑니다. (아직 `Action` 은 `allowed` 입니다 — 규칙 동작이 여전히 `alert` 이기 때문이며, 6장에서 바꿉니다.)

> **★ 경보가 안 뜨면 6장으로 넘어가지 마세요.** 4장의 divert 규칙이나 state 초기화를 다시 확인해야 합니다. 여기서 안 되는데 정책만 바꾸면 원인이 두 겹으로 쌓여 훨씬 찾기 어려워집니다.
{: .prompt-danger }

---

## 6. 따라 하기: 규칙을 "차단(drop)"으로 바꾸는 정책 추가

IPS 모드라도 규칙 동작이 `alert` 이면 여전히 경보만 냅니다. **Policy** 로 우리 규칙셋의 동작을 `drop` 으로 바꿉니다.

> **★ Policy는 Administration 안의 탭이 아닙니다.** 지금까지 쓰던 Settings·Download·Rules·Alerts 탭과 달리, Policy는 **왼쪽 메뉴의 별도 항목**입니다. 왼쪽에서 **Services → Intrusion Detection** 을 펼치면 `Administration` 아래에 **`Policy`** 가 따로 있습니다. 그것을 누르세요(주소는 `/ui/ids/policy`).
{: .prompt-warning }

**1단계.** 왼쪽 메뉴 **Services → Intrusion Detection → Policy** 페이지에서 **+ Add**:

| 항목 | 값 |
| --- | --- |
| Enabled | 체크 |
| Rulesets | **`emerging-web_server.rules`** 를 반드시 포함(SQLi 규칙이 여기 있음) + `emerging-scan` · `emerging-web_specific_apps` |
| **Action** | **`Alert`** ← 반드시 고르세요. **비워 두면 안 됩니다** |
| Rules(메타데이터 필터들) | 모두 **`Nothing selected`** 그대로 |
| **New action** | **`Drop`** |
| Description | 스캔·웹공격 차단 |

![](/assets/img/posts/2027-07-12-socbuild-06-suricata-ids-ips-1787816642950.png)

> **★ 여기서 거의 모두가 걸려 넘어집니다 — `Action` 과 `New action` 은 서로 다른 칸입니다.**
>
> - 위쪽 **`Action`** = *"지금 이 규칙이 가진 동작"* 으로 거르는 **필터**입니다. 우리 규칙들은 지금 **`alert`** 상태이므로 여기에 **`Alert`** 를 고릅니다. **여기에 `Drop` 을 고르면 "이미 drop인 규칙"만 찾게 되는데 그런 규칙이 없어서, 정책이 아무것도 안 바꿉니다** — 그러면 IPS로 바꿔도 공격이 그대로 통과합니다.
> - 아래쪽 **`New action`** = 걸린 규칙을 **바꿀 동작**입니다. 여기에 **`Drop`** 을 넣어야 실제로 차단됩니다.
{: .prompt-danger }

> **★ `Action` 을 비워 두지(Clear All) 마세요 — 정상 통신까지 끊깁니다.**
>
> `Action` 을 비우면 "동작을 가리지 않고 전부"라는 뜻이 되어, **지금 꺼져 있는 규칙까지 함께 걸립니다.** 그런데 Policy의 `New action` 을 `Drop` 으로 두면 걸린 규칙을 **켜면서** 동작을 `drop` 으로 바꿉니다. 즉 우리가 내려받은 세 규칙셋에서 **기본으로 꺼져 있던 1,100여 개 규칙이 한꺼번에 "차단"으로 살아납니다.**
>
> 이 규칙들이 기본으로 꺼져 있는 이유는 **오탐이 많아서**입니다. 그것들이 전부 drop이 되면 공격이 아닌 정상 웹 접속까지 끊겨, "웹도 안 되고 공격도 안 된다"는 상태가 됩니다. 반드시 **`Alert`** 를 고르세요.
{: .prompt-danger }

**2단계.** **Save** → **Apply** 를 누릅니다.

---

## 7. 따라 하기: 같은 공격 다시 실행 → 차단 확인

> **먼저 연결 기록을 지우고 시작하세요.** **Firewall → Diagnostics → States** 에서 `192.168.59.20` 항목을 지웁니다. 앞에서 만든 연결이 남아 있으면 그 연결은 새 정책을 타지 않아, 제대로 설정했는데도 "안 막힌다"로 보입니다.
{: .prompt-warning }

**1단계.** Kali에서 SQL 인젝션을 **다시** 보냅니다.

```bash
curl -G --max-time 10 "http://192.168.59.20:3000/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"
```

![](/assets/img/posts/2027-07-12-socbuild-06-suricata-ids-ips-1787817363314.png)

이번엔 **응답이 오지 않거나 연결이 끊깁니다.** 공격 패킷이 Suricata에서 버려졌기 때문입니다.

**2단계.** 정상 웹 요청은 여전히 되는지 확인합니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.59.20:3000
```

`200` 이 나옵니다. **정상 트래픽은 통과, 공격만 차단** 됩니다.

**3단계.** GUI → **Alerts** 에서 방금 요청의 **Action 이 `drop`** 으로 표시되는지 확인합니다.

> 📷 **필요 이미지** — Alerts 목록에서 같은 SQLi 규칙의 `Action` 이 `allowed`(2-3)에서 `drop`(지금)으로 바뀐 화면.
{: .prompt-tip }

---

## 8. 차단이 안 될 때 — 증상으로 원인 가르기

> **★ 응답(에러 포함)이 돌아온다면** 요청이 표적까지 도달한 것이므로 아직 drop이 안 되는 것입니다. **Alerts 탭의 `Action` 열**로 원인을 나눕니다.
>
> | Alerts에 보이는 것 | 의미 | 할 일 |
> | --- | --- | --- |
> | 경보 자체가 없음 | Suricata가 패킷을 아예 못 봄 | 아래 ①을 확인 (**가장 흔함**) |
> | Action = **`allowed`** | 탐지는 되나 차단 설정이 안 먹음 | 아래 ②③을 확인 |
> | Action = **`drop`** 인데 응답이 옴 | 예전 연결이 살아 있음 | 아래 ④를 확인 |
>
> ① **4장의 divert 방화벽 규칙이 있는가** — `Capture mode` 만 `Divert (IPS)` 로 바꾸고 **Divert-to 규칙을 안 만들면 Suricata에게 패킷이 한 개도 가지 않습니다.** 그래서 차단은커녕 경보조차 새로 안 쌓입니다. **Firewall → Rules** 에서 `Divert-to = Intrusion Detection` 규칙이 **Enabled** 이고 **Apply** 되었는지 보세요.
> ② **Capture mode가 진짜 `Divert (IPS)`인가** — Settings에서 확인. `PCAP live mode (IDS)`로 남아 있으면 절대 안 막힙니다. 바꾸고 **Apply**.
> ③ **Policy가 웹 규칙셋을 Drop으로 덮는가** — SQLi 규칙(`Information Schema`)은 **`emerging-web_server.rules`** 에 있습니다. Policy의 **Rulesets에 이 파일이 포함**되고 `Action`=**Alert**, `New action`=**Drop**, Enabled 체크, **Apply** 되었는지 보세요. `emerging-scan` 만 골랐다면 웹 SQLi 규칙은 여전히 `alert` 로 남습니다.
> ④ **예전 연결 기록(state)을 지웠는가** — **Firewall → Diagnostics → States** 에서 `192.168.59.20` 을 찾아 지운 뒤 다시 시도하세요. 이미 성립된 연결은 새 규칙을 타지 않습니다.
{: .prompt-warning }

> **응답에 `SQLITE_ERROR: no such table: information_schema.tables` 가 보인다면 아직 차단이 안 된 것입니다.** Juice Shop의 DB는 **SQLite** 라서 `information_schema` 가 없어 500 에러가 나는데, **에러라도 응답이 왔다는 것은 요청이 표적까지 갔다는 뜻**입니다. IPS+Drop이 제대로 걸리면 응답 자체가 오지 않습니다. (탐지 자체는 DB와 무관합니다 — Suricata는 요청 URL만 봅니다.)
{: .prompt-info }

> **★ 스캔(nmap) 경보가 더는 안 떠도 정상입니다.** Divert로 바꾸면 Suricata는 **방화벽이 통과시킨 트래픽만** 넘겨받습니다. 그런데 2-2의 규칙이 3000 포트 말고는 전부 막으므로, `nmap -sA` 패킷은 애초에 Suricata까지 가지 않습니다. 이상한 일이 아니라 설계가 그런 것입니다 — 스캔은 이미 **방화벽이** 막고 있고, Suricata는 **열어 둔 문으로 들어오는 공격**을 맡습니다.
{: .prompt-info }

---

## 9. 되돌리는 순서 — "웹도 안 되고 공격도 안 된다" 싶을 때

IPS는 경로 한가운데에 끼어들기 때문에, 잘못되면 정상 통신까지 멈춥니다. 아래 순서로 **한 번에 하나씩** 되돌리며 원인을 좁히세요. 각 단계마다 Kali에서 이 명령이 `200` 을 돌려주는지로 확인합니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.59.20:3000
```

| 순서 | 되돌릴 것 | 어디서 |
| --- | --- | --- |
| 1 | **Policy** 를 비활성(Enabled 해제) 또는 삭제 | Services → Intrusion Detection → Policy |
| 2 | **4장의 divert 방화벽 규칙**을 비활성 또는 삭제 | Firewall → Rules |
| 3 | **Capture mode** 를 `PCAP live mode (IDS)` 로 되돌리고 Apply | Administration → Settings |
| 4 | **Suricata Enabled 체크 해제** 후 Apply | Administration → Settings |
| 5 | 여기까지 해도 안 되면 Suricata가 원인이 아닙니다 | victim에서 `ping 192.168.59.1`, Kali에서 `ping 192.168.57.1` → 방화벽·인터페이스·IP 문제 |

> **★ 웹이 통째로 막혔다면 divert 규칙부터 의심하세요.** `Divert-to` 가 걸린 규칙은 **Suricata가 멈춰 있으면 그 트래픽을 전부 버립니다.** OPNsense 공식 문서의 경고 그대로입니다 — *"divert 소켓을 듣는 서비스가 멈추면, 그 방화벽 규칙은 일치하는 패킷을 모두 버린다."* 그래서 **Suricata부터 끄면 웹까지 같이 죽습니다.** 반드시 위 표의 **Policy → divert 규칙 → Capture mode → Suricata** 순서를 지키세요.
{: .prompt-danger }

![](/assets/img/posts/2027-07-12-socbuild-06-suricata-ids-ips-1787816718862.png)

> **★ 설정이 자꾸 사라지거나 안 먹으면 라이브 모드를 의심하세요.** 관리 화면 위쪽에 파란 배너로 `You are currently running in live media mode` 가 보이면, 방화벽이 디스크가 아니라 설치 ISO로 돌고 있는 것입니다. 라이브 모드는 메모리에 만든 임시 디스크를 쓰는데, 규칙 8,000여 개가 그 공간을 채우면 **설정 저장이 통째로 실패**합니다. **Lobby → Dashboard** 에서 **Disk가 100%** 이거나 **Last configuration change 시각이 한참 전**이면 그 이후로 아무것도 저장되지 못한 것입니다. 이때는 **2-1편으로 돌아가 디스크에 설치**부터 끝낸 뒤 다시 시작하세요.
{: .prompt-danger }

> **끝내 안 되면 그대로 두고 넘어가도 됩니다.** IDS 상태(스냅샷 `suricata-ids`)로 되돌린 뒤 3단계로 진행하세요. 앞서 적었듯 **차단은 5단계에서 AI가 방화벽 API로 수행**하므로, 이 부록의 성공 여부가 이후 진도를 막지 않습니다.
{: .prompt-tip }

---

## 10. 정리 — before / after

| 상태 | Alerts의 `Action` | 공격(SQLi) | 정상 웹 |
| --- | --- | --- | --- |
| 2-3 (IDS · PCAP) | `allowed` | 통과 + 경보 | 200 |
| 이 부록 (IPS · Divert + Drop 정책) | **`drop`** | **차단** + 경보 | 200 |

- [ ] `Capture mode` 를 `Divert (IPS)` 로 바꿨다
- [ ] (4장) **Firewall → Rules** 에 `Divert-to = Intrusion Detection` 규칙을 만들고 Apply 했다
- [ ] (5장) 중간 점검에서 **경보가 새로 쌓이는 것**을 먼저 확인했다
- [ ] Policy를 `Action=Alert` → `New action=Drop` 으로 만들었다 (`Action` 을 비우지 않았다)
- [ ] 같은 SQLi가 이제 **drop** 되고, 정상 웹은 `200` 으로 통과한다
- [ ] 잘 안 될 때 **되돌리는 순서(Policy → divert 규칙 → Capture mode → Suricata)** 를 안다

OPNsense 스냅샷을 `suricata-ips` 로 저장해 두면, 이후 실습에서 문제가 생겼을 때 IDS 스냅샷(`suricata-ids`)과 오가며 비교할 수 있습니다.

---

## 다음

부록은 여기까지입니다. 본 과정으로 돌아가 **3단계 — Nginx + ModSecurity + OWASP CRS 웹 방화벽(WAF)** 으로 진행하세요. IPS를 켜 둔 채로 진행해도 되고, IDS로 되돌린 뒤 진행해도 됩니다. **다만 3단계에서 웹 동작이 이상하면, 먼저 이 부록의 설정을 되돌려 원인에서 제외하는 편이 빠릅니다.**
