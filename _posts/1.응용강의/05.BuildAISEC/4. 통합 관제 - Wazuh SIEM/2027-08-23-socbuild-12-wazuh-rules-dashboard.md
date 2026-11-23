---
title: "[AI 보안관제 구축] 4-3. 규칙과 대시보드 — 흩어진 로그를 하나의 사건으로"
date: 2027-08-23 09:00:00 +0900
categories:
  - 1.응용강의
  - AI보안관제구축
tags:
  - 보안관제
  - SIEM
  - Wazuh
  - 상관분석
  - 대시보드
pin: false
math: false
mermaid: false
---

> 지난 시간 도달 상태: OPNsense와 waf의 로그가 관제 서버로 올라옵니다. Suricata 경보는 화면에 뜨지만, **ModSecurity 차단 기록은 디코딩까지만 되고 경보가 되지 못했습니다.**
> 오늘의 목표: 그 마지막 관문인 **규칙**을 우리 손으로 씁니다. 나아가 **"같은 IP가 1분에 다섯 번 차단되면 한 건의 공격"** 이라는 **상관 규칙**을 만들고, 결과를 한눈에 보는 **대시보드**를 꾸며 4단계를 마칩니다.
> 오늘 켜는 VM: **다섯 대 전부**.
{: .prompt-info }

> **이 편의 명령이 기준으로 삼은 버전**
>
> | 항목 | 기준 | 비고 |
> | --- | --- | --- |
> | Wazuh | **4.14** | 규칙 파일 경로 `/var/ossec/etc/rules/local_rules.xml`, 도구 `/var/ossec/bin/wazuh-logtest` |
> | 대시보드 메뉴 | **4.8 이상**: `Threat Intelligence → Threat Hunting` | 4.7 이하는 `Security events` |
> | 규칙 문법 | `frequency`·`timeframe`·`if_matched_sid` 는 오래전부터 지원 / **`same_field` 는 비교적 최근 4.x** | 오류가 나면 그 줄만 빼도 실습은 됩니다(4-1 참고) |
> | 사용자 규칙 번호 | **100000 이상** | 그 아래는 Wazuh 기본 규칙이 씁니다 |
>
> **필드 이름은 버전이 아니라 로그 형식이 정합니다.** 그래서 이 편은 규칙을 쓰기 전에 `wazuh-logtest` 로 **실제 필드 이름을 눈으로 확인하는 단계**를 먼저 둡니다.
{: .prompt-tip }

## 1. 이론 — 디코더와 규칙, 그리고 레벨

### 1.1 둘의 역할이 다릅니다

| | 디코더(decoder) | 규칙(rule) |
| --- | --- | --- |
| 하는 일 | 로그에서 **값을 뽑아냄** | 뽑힌 값을 보고 **판정** |
| 질문 | "출발지 IP가 어디 적혀 있나?" | "이 값이면 경보인가? 몇 점짜리인가?" |
| 우리 경우 | JSON은 **자동**(4-2에서 확인) | **없음 → 오늘 만듭니다** |

우리는 3-2에서 감사 로그를 JSON으로 맞춰 두었기 때문에 **디코더를 만들 필요가 없습니다.** 오늘 할 일은 규칙뿐입니다.

### 1.2 경보 레벨 — 숫자에 뜻이 있습니다

Wazuh의 모든 경보에는 0~15의 **레벨**이 붙습니다.

| 레벨 | 뜻 | 예 |
| --- | --- | --- |
| 0 | 기록만 하고 경보로 세지 않음 | 분류용 기본 규칙 |
| 3~5 | 정보·낮음 | 로그인 성공, 웹 4xx |
| 6~9 | 중간 | 정책 위반, 반복 실패 |
| **10~12** | **높음** | **공격 시도, 차단 발생** |
| 13~15 | 심각 | 침해 정황, 다중 공격 |

> **레벨은 "누가 언제 볼 것인가"를 정하는 숫자입니다.** 5단계에서 AI에게 "레벨 10 이상만 판단하라"고 지시하게 되므로, **오늘 매기는 점수가 곧 5단계의 필터**가 됩니다. 아무 데나 12를 주면 AI가 온종일 헛일을 하고, 너무 짜게 주면 진짜 공격을 놓칩니다.
{: .prompt-tip }

### 1.3 규칙의 뼈대

```xml
<rule id="100201" level="10">
  <if_sid>100200</if_sid>
  <field name="transaction.response.http_code">^403$</field>
  <description>WAF 차단</description>
  <group>attack,waf_block,</group>
</rule>
```

| 요소 | 뜻 |
| --- | --- |
| `id` | 규칙 번호. **사용자 규칙은 100000 이상**을 씁니다(기본 규칙과 충돌 방지) |
| `level` | 경보 레벨 |
| `if_sid` | **부모 규칙** — "이 규칙이 먼저 걸린 로그 중에서" 라는 뜻 |
| `field` | 디코딩된 필드의 값 조건 |
| `group` | 분류 꼬리표. 대시보드 검색에 씁니다 |

여기에 **상관분석**용 요소가 더 있습니다.

| 요소 | 뜻 |
| --- | --- |
| `frequency` | 몇 번 이상 발생하면 |
| `timeframe` | 몇 초 안에 |
| `if_matched_sid` | 어떤 규칙이 반복되었을 때 |
| `same_field` | **같은 값**(예: 같은 출발지 IP)일 때만 |

> **이것이 SIEM과 단순 로그 뷰어의 결정적 차이입니다.** 로그 한 줄만 보면 "403 한 건"일 뿐이지만, **1분에 다섯 건이 같은 IP에서 나왔다**는 사실은 로그를 모아 본 사람만 알 수 있습니다. 사건은 낱개가 아니라 **묶음**으로 드러납니다.
{: .prompt-info }

---

## 2. 따라 하기: 필드 이름부터 확인한다

규칙을 쓰기 전에 **필드 이름을 정확히 알아야 합니다.** 추측해서 쓰면 조용히 안 걸립니다.

> ### 따라 하기 2-1. 시험용 로그 한 줄 만들기
{: .prompt-tip }

**1단계.** Kali에서 공격을 한 번 보내 새 기록을 만듭니다.

```bash
curl -s -o /dev/null -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"
```

**2단계.** waf 서버에서 방금 줄을 복사합니다.

```bash
sudo tail -n 1 /var/log/nginx/modsec_audit.log
```

> ### 따라 하기 2-2. `wazuh-logtest` 로 필드 확인
{: .prompt-tip }

**3단계.** soc 서버에서 시험 도구를 띄우고, 복사한 줄을 붙여 넣습니다.

```bash
sudo /var/ossec/bin/wazuh-logtest
```

**4단계.** **Phase 2** 에 나오는 이름들을 그대로 적어 둡니다. 우리가 쓸 것은 세 개입니다.

```text
**Phase 2: Completed decoding.
        name: 'json'
        transaction.id: '17XXXXXXX...'
        transaction.client_ip: '192.168.57.10'
        transaction.response.http_code: '403'
        ...
```

| 쓸 필드 | 뜻 |
| --- | --- |
| `transaction.id` | 이 로그가 ModSecurity 감사 로그임을 알아보는 표식 |
| `transaction.client_ip` | **공격자 IP** (3-2에서 진짜 IP가 찍히는 것을 확인했습니다) |
| `transaction.response.http_code` | 응답 코드 — `403` 이면 차단된 것 |

> **★ 화면에 보이는 이름을 그대로 쓰세요.** 아래 예시는 위 세 이름을 기준으로 썼습니다. 여러분의 출력에 다른 이름이 보인다면(버전에 따라 다를 수 있습니다) **본 것을 우선**하세요. 규칙이 안 걸리는 원인의 대부분이 이 이름 불일치입니다.
{: .prompt-danger }

---

## 3. 따라 하기: 규칙 작성

> ### 따라 하기 3-1. `local_rules.xml` 에 규칙 넣기
>
> **목적** WAF 차단 기록을 경보로 만듭니다.
{: .prompt-tip }

**1단계.** soc 서버에서 사용자 규칙 파일을 엽니다. 이 파일이 **우리 규칙의 지정석**입니다.

```bash
sudo cp /var/ossec/etc/rules/local_rules.xml /var/ossec/etc/rules/local_rules.xml.bak
sudo nano /var/ossec/etc/rules/local_rules.xml
```

**2단계.** 파일 맨 끝에 아래를 붙여 넣습니다.

```xml
<group name="modsecurity,waf,">

  <!-- (1) ModSecurity 감사 로그를 알아보는 기본 규칙 (레벨 0 = 경보 아님) -->
  <rule id="100200" level="0">
    <decoded_as>json</decoded_as>
    <field name="transaction.id">\.+</field>
    <description>ModSecurity 감사 로그</description>
  </rule>

  <!-- (2) 차단된 요청 -->
  <rule id="100201" level="10">
    <if_sid>100200</if_sid>
    <field name="transaction.response.http_code">^403$</field>
    <description>WAF 차단 - 웹 공격이 표적에 닿지 못함</description>
    <group>attack,waf_block,</group>
  </rule>

  <!-- (3) 규칙에 걸렸지만 통과한 요청(탐지만) -->
  <rule id="100202" level="5">
    <if_sid>100200</if_sid>
    <description>WAF 탐지 - 규칙에 걸렸으나 통과함</description>
    <group>waf_detect,</group>
  </rule>

</group>
```

**3단계.** 저장하고(`Ctrl+O` → Enter → `Ctrl+X`), **문법이 맞는지 바로 확인**합니다.

```bash
sudo /var/ossec/bin/wazuh-logtest
```

> **이 명령이 문법 검사기 역할을 합니다.** 규칙 파일에 오류가 있으면 도구가 뜨지 않고 **어느 줄이 잘못됐는지** 알려 줍니다. 매니저를 재시작하기 **전에** 여기서 걸러 내세요. 매니저는 규칙 오류가 있으면 아예 뜨지 않습니다.
{: .prompt-warning }

**4단계.** 도구가 떴다면 2장에서 복사한 로그 줄을 다시 붙여 넣습니다. 이번에는 **Phase 3** 가 채워집니다.

```text
**Phase 3: Completed filtering (rules).
        id: '100201'
        level: '10'
        description: 'WAF 차단 - 웹 공격이 표적에 닿지 못함'
        groups: '['modsecurity', 'waf', 'attack', 'waf_block']'
**Alert to be generated.
```

**마지막 줄 `Alert to be generated.` 가 오늘의 목표였습니다.**

**5단계.** 매니저에 적용합니다.

```bash
sudo systemctl restart wazuh-manager
systemctl is-active wazuh-manager
```

> ### 따라 하기 3-2. 대시보드에서 확인
{: .prompt-tip }

**6단계.** Kali에서 공격을 두어 번 보냅니다.

```bash
curl -s -o /dev/null -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=<script>alert(1)</script>"
```

**7단계.** 대시보드 → **Threat Intelligence → Threat Hunting**(4.7 이하는 `Security events`) → 검색창에 넣어 봅니다.

```text
rule.groups:waf_block
```

이제 경보가 보입니다. 하나를 펼치면 `data.transaction.client_ip` 에 **192.168.57.10** 이 들어 있습니다.

> 📷 **필요 이미지 ①** — `rule.groups:waf_block` 검색 결과와, 펼친 경보 상세에 공격자 IP가 보이는 화면.
{: .prompt-tip }

> **`level 0` 규칙(100200)은 왜 필요한가요?** 두 가지 이유입니다.
>
> 1. **분류를 한 번만 합니다.** "이 로그는 ModSecurity 감사 로그다"를 100200이 판정해 두면, 그 아래 규칙들은 `if_sid` 로 물려받기만 하면 됩니다
> 2. **경보를 만들지 않습니다.** 레벨 0은 "판정은 했지만 알리지는 않는다"는 뜻이라, 조용한 로그가 화면을 채우지 않습니다
>
> 실무 규칙셋이 대부분 이 모양입니다 — **넓은 분류 규칙 하나 + 좁은 판정 규칙 여럿.**
{: .prompt-info }

---

## 4. 따라 하기: 상관 규칙 — 낱개를 사건으로 묶기

403 한 건은 "누가 이상한 요청을 한 번 보냈다"입니다. 그런데 **1분에 다섯 번**이면 이야기가 달라집니다. 사람이 실수로 그럴 수는 없습니다.

> ### 따라 하기 4-1. 반복 공격 규칙 추가
{: .prompt-tip }

**1단계.** `local_rules.xml` 의 `</group>` **바로 앞**에 규칙을 하나 더 넣습니다.

```xml
  <!-- (4) 같은 IP에서 60초 안에 5회 이상 차단 -->
  <rule id="100210" level="12" frequency="5" timeframe="60">
    <if_matched_sid>100201</if_matched_sid>
    <same_field>transaction.client_ip</same_field>
    <description>지속적 웹 공격 - 같은 출발지에서 60초 내 WAF 차단 5회 이상</description>
    <group>attack,waf_burst,</group>
  </rule>
```

**2단계.** 문법을 확인하고 적용합니다.

```bash
sudo /var/ossec/bin/wazuh-logtest   # 떴다가 Ctrl+C 로 나오면 문법 정상
sudo systemctl restart wazuh-manager
```

> **`same_field` 에서 오류가 난다면** 설치된 Wazuh 버전이 그 요소를 지원하지 않는 것입니다. 그 줄만 지우고 진행하세요. 대신 "출발지를 가리지 않고 60초에 5회"라는 조금 느슨한 규칙이 됩니다. 동작 원리를 익히는 데는 충분합니다.
{: .prompt-warning }

> ### 따라 하기 4-2. 재현하기
{: .prompt-tip }

**3단계.** Kali에서 짧은 시간에 여러 번 공격을 보냅니다.

```bash
for i in $(seq 1 8); do
  curl -s -o /dev/null -G "http://192.168.59.30/rest/products/search" \
    --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"
done
```

**4단계.** 대시보드에서 확인합니다.

```text
rule.id:100210
```

레벨 **12** 짜리 경보가 **한 건** 올라옵니다. 개별 차단(100201) 경보는 여덟 건인데, 그것을 묶은 **"지속적 웹 공격" 경보는 한 건**입니다.

> **★ 이 한 건이 4단계의 결론입니다.**
>
> 사람이 봐야 할 것은 403 여덟 줄이 아니라 **"192.168.57.10이 지금 우리 웹을 계속 두드리고 있다"** 는 문장 하나입니다. SIEM이 하는 일이 바로 이 **압축**입니다.
>
> 그리고 5단계에서 AI가 읽을 것도 이 한 건입니다. 낱개 로그 수천 줄을 AI에게 던지면 비용만 들고 판단이 흐려집니다. **잘 압축된 경보 한 건**이 좋은 판단의 재료입니다.
{: .prompt-danger }

**5단계(선택).** `nikto` 로도 확인해 보세요. 수백 개의 요청이 대부분 403이 되므로 상관 규칙이 확실히 발동합니다.

```bash
nikto -h http://192.168.59.30
```

---

## 5. 따라 하기: 사건을 시간순으로 읽기

이제 **세 계층의 기록**이 한 곳에 있습니다. 공격 하나를 처음부터 끝까지 재구성해 봅니다.

> ### 따라 하기 5-1. 공격 시나리오 재현
{: .prompt-tip }

**1단계.** Kali에서 **정찰 → 스캔 → 공격** 순서로 실행합니다. 실제 공격자의 순서입니다.

```bash
# ① 정찰 — 어떤 문이 열려 있나
sudo nmap -sA -Pn 192.168.59.30

# ② 스캔 — 어떤 약점이 있나
nikto -h http://192.168.59.30

# ③ 공격 — 데이터베이스를 노린다
curl -s -o /dev/null -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"
```

**2단계.** 대시보드에서 시간 범위를 **Last 15 minutes** 로 두고, 공격자 IP로 걸러 봅니다.

```text
data.src_ip:192.168.57.10 or data.transaction.client_ip:192.168.57.10
```

**3단계.** 시간 오름차순으로 정렬해 읽습니다. 이런 이야기가 보입니다.

| 시각 | 어디서 | 무슨 일 |
| --- | --- | --- |
| 09:01 | 방화벽(Suricata) | `ET SCAN NMAP -sA` — 누군가 문을 두드림 |
| 09:02 | waf(접근 로그) | 수백 건의 요청이 4xx로 응답됨 |
| 09:02 | waf(ModSecurity) | 스캐너 차단 경보 다수 |
| 09:02 | **상관 규칙** | **지속적 웹 공격 (레벨 12)** |
| 09:03 | waf(ModSecurity) | SQL 인젝션 차단 |

> **각 필드 이름은 화면에서 확인하세요.** 경보를 펼치면 실제 필드 이름이 보입니다. Suricata 경보는 `data.src_ip`, ModSecurity 경보는 `data.transaction.client_ip` 에 출발지가 들어 있습니다. **두 계층이 같은 IP를 서로 다른 이름으로 부르는 것** — 이것이 SIEM 운영의 현실이고, 5단계에서 AI에게 자료를 넘길 때 정리해 주어야 할 부분입니다.
{: .prompt-info }

> ### 따라 하기 5-2. 대시보드로 굳히기
{: .prompt-tip }

**4단계.** 매번 검색어를 치지 않도록 화면을 저장합니다. (메뉴 이름은 버전에 따라 조금 다를 수 있습니다.)

1. **Threat Hunting** 에서 `rule.level >= 10` 으로 거릅니다
2. 화면 위쪽 **Save** 로 이름(`고위험 경보`)을 붙여 저장합니다
3. **Visualize → Create visualization** 에서 막대 그래프를 만들고, 축을 아래처럼 잡습니다

| 만들 그림 | X축(버킷) | 쓸모 |
| --- | --- | --- |
| 공격자 순위 | `data.src_ip` 상위 5 | 누가 제일 시끄러운가 |
| 규칙별 건수 | `rule.description` 상위 10 | 무엇이 가장 많이 걸리는가 |
| 시간대별 추이 | 날짜 히스토그램 | 언제 몰렸는가 |

4. **Dashboard → Create dashboard** 로 세 그림을 한 화면에 올리고 **`SOC 개요`** 로 저장합니다

> 📷 **필요 이미지 ②** — 직접 만든 `SOC 개요` 대시보드(공격자 순위·규칙별 건수·시간대별 추이).
{: .prompt-tip }

> **대시보드는 "무엇을 먼저 볼 것인가"를 정하는 일입니다.** 화면을 예쁘게 만드는 것이 목적이 아니라, **아침에 출근해서 30초 안에 이상을 알아채는 것**이 목적입니다. 그림이 많을수록 좋은 대시보드가 아닙니다.
{: .prompt-tip }

---

## 6. 4단계 정리 — 그리고 남은 한 가지

### 6.1 지금까지 만든 것

| 단계 | 만든 것 | 같은 공격에 대한 반응 |
| --- | --- | --- |
| 1 | 표적과 공격자 | 전부 성공 |
| 2 | 방화벽 + Suricata IDS | 포트는 막힘, 공격은 **탐지** |
| 3 | Nginx + ModSecurity WAF | 웹 공격 **차단** |
| **4** | **Wazuh SIEM** | 세 계층의 기록이 **한 화면에**, 반복 공격은 **한 건의 경보로** |

```text
[ Kali ] ─공격─▶ [ 방화벽·IDS ] ─▶ [ WAF ] ─▶ [ 표적 ]
                      │              │
                      └── 로그 ──────┘
                             ▼
                     [ Wazuh SIEM 58.100 ]
                             │
                    "지속적 웹 공격 (레벨 12)"
                             │
                             ▼
                        ??? ← 아직 사람이 봐야 합니다
```

### 6.2 아직 사람이 필요합니다

경보는 잘 만들어졌지만, **그 경보를 보고 판단하고 조치하는 것은 여전히 사람**입니다. 새벽 3시에 레벨 12 경보가 떠도 아침까지 아무 일도 일어나지 않습니다.

여기서 두 가지 길이 있습니다.

| 길 | 방식 | 한계 |
| --- | --- | --- |
| **자동 대응(Active Response)** | "레벨 12면 무조건 차단" 같은 **고정 규칙**으로 즉시 조치 | 오탐이면 **정상 사용자를 끊습니다.** 조건을 정교하게 쓸수록 관리가 어려워집니다 |
| **AI 판단** | 경보의 **맥락을 읽고** 차단할지 지켜볼지 판단 | 판단 근거를 남기고, 사람이 검토할 수 있어야 합니다 |

Wazuh에도 Active Response 기능이 있어 경보가 뜨면 스크립트를 실행할 수 있습니다. 다만 그것은 **에이전트가 설치된 그 서버의 방화벽**을 만지는 것이라, 우리가 원하는 **경계 방화벽(OPNsense)에서의 차단**과는 다릅니다.

> **그래서 5단계는 이렇게 갑니다.** 관제 서버에 **로컬 AI(Ollama)** 를 올리고, Wazuh가 만든 경보를 읽혀 **"차단할까요, 지켜볼까요"** 를 판단하게 합니다. 차단 결정이 나면 **OPNsense API** 를 호출해 공격자 IP를 경계에서 끊습니다. 4-1에서 **관제망 → 방화벽 443 포트를 미리 열어 둔 것**이 이것을 위해서였습니다.
>
> AI가 읽을 자료도 오늘 준비됐습니다. 경보는 `/var/ossec/logs/alerts/alerts.json` 에 JSON으로 쌓이고, Wazuh API(`https://192.168.58.100:55000`)로도 가져올 수 있습니다.
{: .prompt-tip }

---

## 7. 따라 하기: 스냅샷과 점검

**1단계.** 규칙을 백업합니다.

```bash
sudo cp /var/ossec/etc/rules/local_rules.xml ~/local_rules-$(date +%F).xml
ls -l ~/local_rules-*.xml
```

**2단계.** soc 서버를 종료하고 스냅샷 **`soc-rules`** 를 만듭니다.

```bash
sudo poweroff
```

점검 목록:

- [ ] `wazuh-logtest` 로 **필드 이름을 먼저 확인**한 뒤 규칙을 썼다
- [ ] `local_rules.xml` 에 100200(분류)·100201(차단)·100202(탐지) 규칙을 넣었다
- [ ] `wazuh-logtest` 에서 **`Alert to be generated.`** 를 확인했다
- [ ] 매니저 재시작 후 대시보드에서 `rule.groups:waf_block` 경보가 보인다
- [ ] 상관 규칙(100210)을 넣고, 반복 공격이 **레벨 12 한 건**으로 묶이는 것을 확인했다
- [ ] 공격자 IP로 걸러 **정찰 → 스캔 → 공격** 순서를 시간순으로 읽었다
- [ ] `SOC 개요` 대시보드를 저장했다
- [ ] 경보가 `/var/ossec/logs/alerts/alerts.json` 에 쌓인다는 것을 확인했다
- [ ] 레벨 체계(0·5·10·12)의 뜻을 설명할 수 있다

> **마지막 항목을 직접 확인해 두면 좋습니다.** 5단계에서 AI가 읽을 파일입니다.
>
> ```bash
> sudo tail -n 1 /var/ossec/logs/alerts/alerts.json | jq '{rule: .rule.description, level: .rule.level, agent: .agent.name}'
> ```
{: .prompt-tip }

---

## 다음 단계 예고 (5단계)

이제 마지막입니다. **5단계** 에서는 관제 서버에 **Ollama**로 로컬 AI 모델을 올리고, 오늘 만든 경보를 읽혀 **차단 여부를 스스로 판단**하게 합니다. 판단이 나오면 **OPNsense API** 를 호출해 공격자 IP를 경계 방화벽에서 끊습니다.

그러면 1단계에서 무방비로 성공했던 그 공격이 **탐지 → 차단 → 기록 → 판단 → 자동 차단** 이라는 **폐루프**를 한 바퀴 도는 것을, 처음부터 끝까지 자기 손으로 만든 시스템 위에서 보게 됩니다.
