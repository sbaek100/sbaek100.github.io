---
title: "[AI 보안관제 구축] 4-2. 에이전트로 로그 모으기 — 방화벽·WAF의 기록을 한곳으로"
date: 2027-08-16 09:00:00 +0900
categories:
  - 1.응용강의
  - AI보안관제구축
tags:
  - 보안관제
  - SIEM
  - Wazuh
  - 에이전트
  - 로그수집
pin: false
math: false
mermaid: false
---

> 지난 시간 도달 상태: 관제망(`soc-lan`)이 열리고 Wazuh SIEM이 돌고 있습니다. 다만 **Agents 화면이 비어 있어** 아무 로그도 들어오지 않습니다.
> 오늘의 목표: **waf 서버**와 **OPNsense 방화벽**에 Wazuh 에이전트를 붙여, 지금까지 만든 로그를 관제 서버로 밀어 올립니다. 그리고 같은 공격을 다시 보내 **한 화면에 두 계층의 기록이 나란히 올라오는 것**을 확인합니다.
> 오늘 켜는 VM: **다섯 대 전부** (soc · OPNsense · waf · victim · Kali).
{: .prompt-info }

> **이 편의 명령이 기준으로 삼은 버전**
>
> | 구성요소 | 버전 | 어디서 정해지는가 |
> | --- | --- | --- |
> | Wazuh 매니저 | **4.14** (4-1에서 설치) | 설치 스크립트 주소 |
> | 우분투용 에이전트 | **4.x 최신** | apt 저장소 주소가 `.../4.x/apt` 라 **매니저와 같은 계열의 최신**이 깔립니다 |
> | OPNsense 에이전트 | 플러그인 **`os-wazuh-agent`** 가 주는 버전 | OPNsense 저장소가 정하므로 매니저와 다를 수 있습니다 |
> | waf 서버 OS | Ubuntu Server 24.04 LTS | 3-1에서 설치 |
>
> **매니저와 에이전트는 같은 4.x 계열로 맞추는 것이 안전합니다.** 특히 OPNsense 쪽은 우리가 버전을 고를 수 없으므로, 설치 뒤 아래로 확인해 두세요.
>
> ```bash
> /var/ossec/bin/wazuh-control info      # 에이전트(OPNsense 콘솔 또는 waf 서버)
> ```
>
> 매니저(4.14)보다 **한참 낮은** 버전이면 일부 필드나 기능이 다르게 보일 수 있습니다. 그럴 때는 **에이전트가 보낸 원본 로그**(4-2 마지막의 `wazuh-logtest`)로 확인하는 것이 가장 확실합니다.
{: .prompt-tip }

## 0. 오늘의 그림

오늘 만드는 것은 **로그가 흐르는 길**입니다. 공격은 왼쪽에서 오른쪽으로 흐르고, **로그는 위로 올라갑니다.**

```text
[ Kali 57.10 ] ──공격──▶ [ OPNsense ] ──▶ [ waf 59.30 ] ──▶ [ victim 59.20 ]
                              │                  │
                       eve.json│           modsec_audit.log
                     (Suricata)│           juice_access.log
                              │                  │
                              └──── 에이전트 ────┘
                                       │  1514/tcp
                                       ▼
                              [ soc 58.100 ]  Wazuh SIEM
```

**로그는 한 방향으로만 흐릅니다.** 감시 대상이 관제 서버로 **밀어 올릴** 뿐, 관제 서버가 감시 대상에 들어가지 않습니다(4-1의 OPT2 차단 규칙이 그 뜻입니다).

---

## 1. 이론 — 로그 한 줄이 경보가 되기까지

### 1.1 다섯 개의 관문

로그 한 줄은 아래 다섯 단계를 모두 통과해야 화면에 나타납니다. **어느 한 곳에서 막히면 화면은 조용합니다.**

| 단계 | 하는 일 | 오늘 우리가 하는 것 | 막히면 |
| --- | --- | --- | --- |
| ① **수집** | 에이전트가 파일을 읽음 | `localfile` 로 파일 지정 | 아무 데이터도 안 감 |
| ② **전송** | 관제 서버로 보냄 | 1514/tcp 문 열기 | 에이전트가 `Disconnected` |
| ③ **디코딩** | 로그에서 필드를 뽑음 | JSON은 자동, 나머지는 디코더 | 필드가 비어 있음 |
| ④ **규칙** | "이건 경보다" 판정 | Suricata는 **기본 제공**, ModSecurity는 **직접 작성**(4-3) | **경보가 안 생김** |
| ⑤ **인덱싱** | 검색 가능하게 저장 | 자동 | 화면에서 검색 안 됨 |

> **오늘 ④에서 일부러 한 번 막힙니다.** Suricata의 `eve.json` 은 Wazuh가 **이미 아는 형식**이라 규칙이 기본으로 들어 있습니다. 반면 ModSecurity 감사 로그는 **우리가 만든 형식**이라 규칙이 없습니다. 그래서 오늘 로그를 올려도 화면에는 나타나지 않습니다. 그 경험이 4-3(규칙 작성)의 출발점입니다.
>
> 2단계에서 배운 것과 같은 교훈입니다 — **"로그가 들어왔다 ≠ 경보가 뜬다."** 규칙이 있어야 뜻이 생깁니다.
{: .prompt-tip }

### 1.2 에이전트 방식 vs 무에이전트(syslog) 방식

| | 에이전트 | 무에이전트(syslog 전송) |
| --- | --- | --- |
| 설치 | 대상에 프로그램을 깔아야 함 | 설치 불필요 |
| 전송 | **암호화**, 유실 시 재전송 | 평문 UDP가 흔함, 유실 가능 |
| 형식 | 파일마다 **형식 지정 가능**(JSON 등) | syslog 형식으로 눌러 담김 → **디코더를 직접 만들어야 하는 경우가 많음** |
| 상태 | 연결 상태·버전을 관제에서 확인 | 알 수 없음 |
| 쓸 곳 | 서버·방화벽 등 설치 가능한 곳 | 스위치·프린터 등 설치 불가능한 장비 |

**이 강의는 양쪽 다 에이전트를 씁니다.** OPNsense에도 **공식 플러그인**이 있어서, 방화벽에 GUI로 에이전트를 붙일 수 있기 때문입니다.

### 1.3 쓰는 포트 두 개

| 포트 | 용도 |
| --- | --- |
| **1515/tcp** | **등록(enrollment)** — 에이전트가 처음 자기를 관제 서버에 신고할 때 |
| **1514/tcp** | **평상시 통신** — 로그를 밀어 올리는 길 |

4-1에서 만들어 둔 `wazuh_ports` 별칭이 이 둘입니다.

---

## 2. 따라 하기: 관제망으로 가는 문 열기

waf 서버(59.30, OPT1)에서 관제 서버(58.100, OPT2)로 가려면 **방화벽을 지나야 합니다.** 지금은 그 길이 없습니다.

> ### 따라 하기 2-1. OPT1에 로그 전송 규칙 추가
{: .prompt-tip }

**1단계.** OPNsense에서 **Firewall → Rules → OPT1** 로 갑니다. 2-2편에서 만든 규칙 네 개가 있습니다.

**2단계.** **+ Add** 로 규칙을 만듭니다.

| 항목 | 값 |
| --- | --- |
| Action | **Pass** |
| Interface | OPT1 |
| Direction | in |
| Protocol | **TCP** |
| Source | **OPT1 net** |
| Destination | **`soc_host`** |
| Destination Port | **`wazuh_ports`** |
| Description | `dmz logs to soc` |

**3단계.** 이 규칙을 **목록 맨 위**로 옮기고 **Apply** 를 누릅니다.

**4단계.** waf 서버에서 길이 뚫렸는지 확인합니다.

```bash
nc -zv 192.168.58.100 1514
```

`succeeded` 또는 `open` 이 나오면 됩니다. (`nc` 가 없으면 `sudo apt install -y netcat-openbsd` 로 설치하세요.)

> **연결이 안 되면 순서를 확인하세요.** 2-2편의 차단 규칙보다 **아래**에 있으면 걸리지 않습니다. 그리고 관제망 방향은 열되, **DMZ에서 관제망의 다른 포트로는 가지 못한다**는 점도 확인해 두면 좋습니다.
>
> ```bash
> nc -zv 192.168.58.100 22
> ```
>
> 이쪽은 **실패**해야 정상입니다. 로그를 올리는 문 하나만 열었기 때문입니다.
{: .prompt-warning }

---

## 3. 따라 하기: waf 서버에 에이전트 설치

> ### 따라 하기 3-1. 저장소 등록과 설치
{: .prompt-tip }

**1단계.** waf 서버에서 Wazuh 저장소를 등록합니다.

```bash
sudo apt-get install -y gnupg apt-transport-https
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
  sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt-get update
```

**2단계.** 관제 서버 주소를 지정해 설치합니다. **이 한 줄에 등록 정보가 들어갑니다.**

```bash
sudo WAZUH_MANAGER="192.168.58.100" apt-get install -y wazuh-agent
```

**3단계.** 서비스를 켭니다.

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
systemctl is-active wazuh-agent
```

> **`WAZUH_MANAGER` 를 빠뜨리고 설치했다면** 설정 파일에서 직접 고칠 수 있습니다.
>
> ```bash
> sudo sed -i 's|<address>.*</address>|<address>192.168.58.100</address>|' /var/ossec/etc/ossec.conf
> sudo systemctl restart wazuh-agent
> ```
{: .prompt-warning }

> ### 따라 하기 3-2. 등록됐는지 확인
{: .prompt-tip }

**4단계.** Kali 브라우저에서 대시보드(`https://192.168.58.100`) → 왼쪽 메뉴 **Agents**.

`waf` 가 **Active**(초록색)로 보이면 성공입니다. 등록에 30초쯤 걸립니다.

**5단계.** 명령으로도 확인할 수 있습니다. soc 서버에서 실행합니다.

```bash
sudo /var/ossec/bin/agent_control -l
```

> 📷 **필요 이미지 ①** — Wazuh 대시보드 Agents 화면에 `waf` 가 Active로 올라온 모습.
{: .prompt-tip }

> **`Disconnected` 또는 `Never connected` 로 보인다면** — ①~③ 순서로 확인하세요.
>
> | # | 확인 | 방법 |
> | --- | --- | --- |
> | 1 | 방화벽 문(2장) | waf에서 `nc -zv 192.168.58.100 1514` |
> | 2 | 에이전트가 보는 주소 | `grep -A2 "<server>" /var/ossec/etc/ossec.conf` 에 58.100이 있는지 |
> | 3 | 에이전트 자신의 기록 | `sudo tail -n 30 /var/ossec/logs/ossec.log` |
>
> 3번 로그에는 등록 실패 사유가 그대로 적힙니다. **추측하기 전에 이 파일을 먼저 보세요.**
{: .prompt-warning }

---

## 4. 따라 하기: waf가 올릴 로그 지정하기

에이전트는 설치만으로 **시스템 로그(`/var/log/syslog`, 인증 로그 등)** 는 알아서 올립니다. 하지만 3단계에서 우리가 만든 로그는 **직접 지정**해야 합니다.

> ### 따라 하기 4-1. `ossec.conf` 에 파일 두 개 추가
{: .prompt-tip }

**1단계.** 설정 파일의 맨 끝(`</ossec_config>` 앞)에 두 덩어리를 넣습니다. 아래 명령이 안전하게 끼워 넣어 줍니다.

```bash
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak

sudo python3 - <<'EOF'
path = "/var/ossec/etc/ossec.conf"
block = """
  <!-- 3단계에서 만든 WAF 로그 -->
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/nginx/modsec_audit.log</location>
  </localfile>

  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/nginx/juice_access.log</location>
  </localfile>

</ossec_config>"""
text = open(path).read()
text = text.rstrip()
assert text.endswith("</ossec_config>"), "예상과 다른 파일 구조입니다. 직접 편집하세요."
text = text[: text.rfind("</ossec_config>")] + block + "\n"
open(path, "w").write(text)
print("추가 완료")
EOF
```

**2단계.** 제대로 들어갔는지 확인합니다.

```bash
sudo tail -n 15 /var/ossec/etc/ossec.conf
```

**3단계.** 에이전트를 다시 시작합니다.

```bash
sudo systemctl restart wazuh-agent
sudo grep -i "Analyzing file" /var/ossec/logs/ossec.log | tail -n 10
```

`Analyzing file: '/var/log/nginx/modsec_audit.log'` 같은 줄이 보이면 **①수집 관문을 통과한 것**입니다. (재시작 직후에는 몇 초 걸릴 수 있으니, 안 보이면 잠시 뒤 다시 실행하세요.)

> **두 파일의 `log_format` 이 다른 이유**
>
> | 파일 | 형식 | 왜 |
> | --- | --- | --- |
> | `modsec_audit.log` | **`json`** | 3-2에서 `SecAuditLogFormat JSON` 으로 맞춰 두었습니다. JSON은 Wazuh가 **필드를 자동으로 분해**합니다 |
> | `juice_access.log` | **`syslog`** | Wazuh에서 `syslog` 는 "평범한 한 줄짜리 텍스트"라는 뜻입니다. 웹 접근 로그는 내장 디코더가 알아봅니다 |
>
> **로그 형식을 미리 JSON으로 맞춰 둔 보람이 여기서 나옵니다.** 형식이 제각각이면 ③디코딩 관문에서 전부 막혔을 것입니다.
{: .prompt-info }

> **권한 문제로 파일을 못 읽을 수 있습니다.** 에이전트는 `wazuh` 사용자로 돌기 때문에, `/var/log/nginx/` 디렉터리를 읽을 수 있어야 합니다. `ossec.log` 에 `Permission denied` 가 보이면 아래로 해결합니다.
>
> ```bash
> sudo usermod -a -G adm wazuh
> sudo chmod 755 /var/log/nginx
> sudo systemctl restart wazuh-agent
> ```
{: .prompt-warning }

---

## 5. 따라 하기: OPNsense에 에이전트 붙이기

방화벽에는 **공식 플러그인**이 있습니다. 화면에서 설치하고 주소만 적으면 됩니다.

> ### 따라 하기 5-1. 플러그인 설치
{: .prompt-tip }

**1단계.** OPNsense → **System → Firmware → Plugins** 로 갑니다.

**2단계.** 검색창에 **`os-wazuh-agent`** 를 입력하고 오른쪽 **`+`** 를 눌러 설치합니다. 잠시 뒤 왼쪽 메뉴에 **Services → Wazuh Agent** 가 생깁니다.

> 📷 **필요 이미지 ②** — Plugins 목록에서 `os-wazuh-agent` 를 설치하는 화면.
{: .prompt-tip }

> ### 따라 하기 5-2. 관제 서버 지정
{: .prompt-tip }

**3단계.** **Services → Wazuh Agent → Settings** 에서 설정합니다.

| 항목 | 값 |
| --- | --- |
| **Enabled** | **체크** |
| **Manager hostname** | **`192.168.58.100`** |
| Password | (비워 둠 — 기본 등록 방식을 씁니다) |
| Intrusion detection events | **체크 해제** (아래 설명 참고) |

**4단계.** **Save** 를 누릅니다. 서비스가 시작되고, 잠시 뒤 관제 서버에 자기를 등록합니다.

**5단계.** 대시보드 **Agents** 에서 방화벽이 **Active** 로 올라왔는지 확인합니다.

> **★ `Intrusion detection events` 를 켜지 않는 이유**
>
> 이 항목을 켜면 Suricata 경보가 **syslog 형식(RFC3164)** 으로 포장되어 올라갑니다. 그러면 Wazuh가 가진 **Suricata 전용 규칙이 걸리지 않아**, 결국 디코더를 직접 만들어야 합니다.
>
> 우리는 대신 **원본 `eve.json` 을 JSON 그대로** 올릴 것입니다(5-3). 그러면 Wazuh의 기본 제공 Suricata 규칙이 그대로 동작합니다. **형식을 바꾸지 않고 원본을 보내는 것**이 언제나 유리합니다.
{: .prompt-danger }

> ### 따라 하기 5-3. `eve.json` 을 JSON 그대로 올리기
{: .prompt-tip }

**6단계.** 먼저 **파일이 실제로 어디 있는지** 확인합니다. OPNsense 콘솔에서 메뉴 **`8`(Shell)** 을 고르고 실행합니다.

```bash
ls -l /var/log/suricata/
```

`eve.json` 이 보이면 그 경로를 씁니다. 인터페이스별 하위 폴더 안에 있다면 그 경로를 그대로 적으면 됩니다.

**7단계.** 플러그인이 읽어 갈 **추가 설정 파일**을 만듭니다.

```bash
mkdir -p /usr/local/opnsense/service/templates/OPNsense/WazuhAgent/ossec_config.d

cat > /usr/local/opnsense/service/templates/OPNsense/WazuhAgent/ossec_config.d/099-suricata.conf <<'EOF'
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/eve.json</location>
  </localfile>
EOF
```

**8단계.** 설정을 다시 만들어 적용합니다.

```bash
configctl template reload OPNsense/WazuhAgent
service wazuh-agent restart
```

**9단계.** 실제 설정 파일에 반영됐는지 확인합니다.

```bash
grep -A3 suricata /var/ossec/etc/ossec.conf
```

> **명령이 안 먹는다면** — 서비스 이름이나 템플릿 경로가 버전에 따라 다를 수 있습니다.
>
> - 서비스 이름 확인: `service -e | grep -i wazuh`
> - GUI에서 **Services → Wazuh Agent → Settings** 화면을 열어 **아무 값도 바꾸지 않고 Save** 를 눌러도 설정이 다시 생성됩니다
> - 그래도 `ossec.conf` 에 반영되지 않으면, 7단계에서 만든 파일의 위치를 다시 확인하세요(오타가 가장 흔합니다)
{: .prompt-warning }

---

## 6. 따라 하기: 같은 공격, 한 화면에서 보기

이제 1단계부터 써 온 공격을 다시 보냅니다. 이번에는 **공격 자체가 아니라 화면**을 보는 것이 목적입니다.

> ### 따라 하기 6-1. 공격 발생시키기
{: .prompt-tip }

**1단계.** Kali에서 세 가지를 연달아 실행합니다.

```bash
sudo nmap -sA -Pn 192.168.59.30

curl -s -o /dev/null -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"

nikto -h http://192.168.59.30
```

**2단계.** 1~2분 기다립니다. 로그가 파일에 쓰이고, 에이전트가 읽어 올리고, 규칙이 판정하기까지 시간이 조금 걸립니다.

> ### 따라 하기 6-2. 대시보드에서 확인
{: .prompt-tip }

**3단계.** 대시보드 왼쪽 메뉴 → **Threat Intelligence → Threat Hunting** 으로 들어가 시간 범위를 **Last 15 minutes** 로 맞춥니다.

> **메뉴 이름은 버전에 따라 다릅니다.** **4.8 이상**은 `Threat Intelligence → Threat Hunting`, **4.7 이하**는 `Security events` 입니다. 둘은 같은 화면이며, 이 강의는 **4.14** 기준으로 적었습니다.
{: .prompt-info }

**4단계.** 검색창에 아래를 차례로 넣어 봅니다.

| 검색어 | 무엇이 보이나 |
| --- | --- |
| `rule.groups:suricata` | 방화벽의 Suricata 경보 — **2-3편에서 본 그 경보**가 이제 여기 있습니다 |
| `agent.name:waf` | waf 서버가 올린 모든 기록 |
| `rule.id:31101` | 웹 서버가 4xx를 돌려준 요청 — **WAF가 403으로 막은 흔적**입니다 |

**5단계.** 아무 경보나 하나 눌러 펼쳐 보세요. `agent.name`, `rule.description`, `data.srcip` 같은 항목이 보입니다. **여러 장비의 기록이 같은 모양으로 정리되어 있는 것** — 이것이 ②정규화의 결과입니다.

> 📷 **필요 이미지 ③** — `rule.groups:suricata` 로 필터링한 이벤트 목록과, 그중 하나를 펼친 상세 화면.
{: .prompt-tip }

> **여기까지 왔다면 큰 고비를 넘긴 것입니다.** 2-3편에서는 OPNsense 화면에서, 3-2편에서는 waf 터미널에서 따로 봐야 했던 것이, 지금은 **한 화면에서 시간순으로** 보입니다.
{: .prompt-info }

---

## 7. 따라 하기: 그런데 WAF 차단 기록은 왜 안 보일까

**1단계.** 대시보드 검색창에 아래를 넣어 봅니다.

```text
data.transaction.client_ip:*
```

**아무것도 안 나옵니다.** ModSecurity 감사 로그는 분명히 올렸는데 말입니다.

**2단계.** 정말 안 올라온 것인지, 아니면 다른 이유인지 가려 봅니다. 먼저 waf 서버에서 로그 한 줄을 복사합니다.

```bash
sudo tail -n 1 /var/log/nginx/modsec_audit.log
```

**3단계.** soc 서버에서 **로그 시험 도구**를 띄웁니다.

```bash
sudo /var/ossec/bin/wazuh-logtest
```

**4단계.** 2단계에서 복사한 JSON 한 줄을 붙여 넣고 Enter를 칩니다. 이런 출력이 나옵니다.

```text
**Phase 1: Completed pre-decoding.
**Phase 2: Completed decoding.
        name: 'json'
        transaction.client_ip: '192.168.57.10'
        transaction.response.http_code: '403'
        ...
```

**5단계.** 그런데 그다음이 조용합니다. 규칙 단계가 이렇게 끝납니다.

```text
**Phase 3: Completed filtering (rules).
```

> **★ 진단 완료 — ③디코딩까지는 성공, ④규칙에서 멈췄습니다.**
>
> - **필드는 잘 뽑혔습니다.** `transaction.client_ip` 에 공격자 IP가, `transaction.response.http_code` 에 403이 들어 있습니다
> - **그런데 이 필드를 보고 "이건 경보다"라고 말해 주는 규칙이 없습니다.** 그래서 경보가 만들어지지 않고, 경보가 없으니 화면에도 없습니다
>
> Suricata는 왜 됐을까요? **Wazuh가 Suricata를 미리 알고 있어서** 규칙이 기본으로 들어 있기 때문입니다. ModSecurity의 JSON은 우리가 3-2에서 정한 형식이므로, **규칙은 우리가 씁니다.** 그것이 다음 시간의 주제입니다.
{: .prompt-danger }

> **`wazuh-logtest` 를 꼭 기억해 두세요.** SIEM 작업의 대부분은 "왜 안 뜨지?"를 푸는 일이고, 이 도구가 **다섯 관문 중 어디서 막혔는지**를 정확히 알려 줍니다. 4-3에서 규칙을 만들 때도 이 도구로 시험한 뒤 적용합니다. (빠져나올 때는 `Ctrl+C` 입니다.)
{: .prompt-tip }

---

## 8. 따라 하기: 스냅샷과 점검

**1단계.** 각 서버를 정리하고 스냅샷을 만듭니다.

- `waf` → **`waf-agent`**
- `soc` → **`soc-agents-connected`**
- OPNsense → **`fw-wazuh-agent`**

점검 목록:

- [ ] OPT1에 `dmz logs to soc` 규칙(1514·1515)을 만들고 **맨 위**에 두었다
- [ ] waf에서 `nc -zv 192.168.58.100 1514` 가 성공하고, 22번은 실패한다
- [ ] waf에 에이전트를 설치하고 대시보드에서 **Active** 로 보인다
- [ ] waf의 `ossec.conf` 에 `modsec_audit.log`(json)·`juice_access.log`(syslog)를 추가했다
- [ ] OPNsense에 `os-wazuh-agent` 플러그인을 설치하고 Manager를 58.100으로 지정했다
- [ ] `Intrusion detection events` 는 **끄고**, `099-suricata.conf` 로 **eve.json 원본**을 올렸다
- [ ] 대시보드에서 `rule.groups:suricata` 로 방화벽 경보가 보인다
- [ ] `rule.id:31101` 로 웹 4xx 기록이 보인다
- [ ] `wazuh-logtest` 로 ModSecurity 로그가 **디코딩은 되지만 규칙이 없다**는 것을 확인했다

---

## 다음 시간

로그는 모였지만, 아직 **기계가 읽은 글자**일 뿐입니다. **4-3** 에서는 ModSecurity 로그에 우리 **규칙과 디코더**를 붙여 경보로 만들고, 한 걸음 더 나아가 **"같은 IP가 1분에 다섯 번 차단되면 그것은 한 건의 공격 시도"** 라는 **상관 규칙**을 작성합니다. 그리고 그 결과를 한눈에 보는 **대시보드**를 만들어 4단계를 마칩니다. 이 화면이 곧 5단계에서 **AI가 읽을 자료**가 됩니다.
