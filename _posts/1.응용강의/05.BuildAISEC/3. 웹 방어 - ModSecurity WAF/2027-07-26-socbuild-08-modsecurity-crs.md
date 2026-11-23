---
title: "[AI 보안관제 구축] 3-2. ModSecurity + OWASP CRS — 웹 공격을 애플리케이션 계층에서 차단"
date: 2027-07-26 09:00:00 +0900
categories:
  - 1.응용강의
  - AI보안관제구축
tags:
  - 보안관제
  - WAF
  - ModSecurity
  - OWASPCRS
  - 웹보안
pin: false
math: false
mermaid: false
---

> 지난 시간 도달 상태: 표적으로 가는 직통 경로를 닫고, 모든 웹 요청이 **WAF 서버(192.168.59.30)** 의 Nginx를 지나가게 만들었습니다. 다만 검사기가 없어 공격은 그대로 통과했습니다.
> 오늘의 목표: 그 Nginx에 **ModSecurity 엔진**과 **OWASP CRS 규칙**을 얹어, 어제 통과했던 공격을 **403 Forbidden** 으로 막습니다. 2단계와 같은 순서 — **먼저 탐지(DetectionOnly), 확인한 뒤 차단(On)** — 으로 올립니다.
> 오늘 켜는 VM: **OPNsense + Kali + victim + waf** 네 대.
{: .prompt-info }

> **★ 이 편의 명령이 기준으로 삼은 버전 — 여기가 어긋나면 명령이 그대로 듣지 않습니다.**
>
> | 구성요소 | 버전 | 확인 명령 |
> | --- | --- | --- |
> | OS | **Ubuntu Server 24.04 LTS**(noble) | `lsb_release -a` |
> | 웹 서버 | **Nginx 1.24.0**(우분투 기본 저장소) | `nginx -v` |
> | Nginx 커넥터 | **`libnginx-mod-http-modsecurity` 1.0.3** (universe) | `apt policy libnginx-mod-http-modsecurity` |
> | **WAF 엔진** | **libmodsecurity 3.0.12 — 즉 ModSecurity `v3`** | 커넥터가 함께 설치 |
> | 규칙셋 | **OWASP CRS 3.3.5** | `apt policy modsecurity-crs` |
>
> 세 가지를 특히 기억하세요.
>
> 1. **커넥터는 우분투가 배포하는 nginx 1.24.0에 묶여 있습니다**(꾸러미가 `nginx-abi-1.24.0-1` 을 요구). nginx.org 저장소나 소스 빌드 nginx를 쓰면 **모듈이 로드되지 않습니다.**
> 2. **엔진은 v3입니다.** 인터넷 자료의 상당수는 Apache 시절 **v2** 기준이라 지시어가 다릅니다(3장에서 표로 정리합니다).
> 3. **CRS가 4.x로 올라가면** 규칙 번호(942100·949110 등)와 이상 점수 개념은 그대로지만 파일 배치가 달라질 수 있습니다. 그래서 2-2에서 **경로를 눈으로 확인하는 단계**를 먼저 둡니다.
{: .prompt-danger }

## 1. 이론 — 엔진과 규칙, 그리고 점수

### 1.1 ModSecurity는 "규칙 실행기"입니다

WAF를 이루는 것은 두 부분입니다.

| 구성 | 역할 | 우리가 쓸 것 |
| --- | --- | --- |
| **엔진** | HTTP 요청을 파싱하고, 규칙을 하나씩 대조하고, 지시대로 통과·차단·기록 | **ModSecurity v3**(libmodsecurity) + Nginx 커넥터 |
| **규칙** | "무엇이 공격인가"를 적어 둔 목록 | **OWASP CRS 3.3** |

Suricata와 ET Open 규칙의 관계와 똑같습니다. **엔진만 깔면 아무것도 잡지 못하고, 규칙만 있어도 아무 일이 일어나지 않습니다.**

### 1.2 규칙 한 줄 읽기

CRS 규칙은 이런 모양입니다. 문법을 외울 필요는 없고, **어디를 볼지·무엇과 맞출지·어떻게 할지** 세 부분으로 읽으면 됩니다. (아래는 **읽기 쉽게 줄인 것**입니다. CRS 3.3.5의 실제 942100은 파라미터뿐 아니라 쿠키·XML 등 더 많은 곳을 함께 봅니다. 원문은 `/usr/share/modsecurity-crs/rules/REQUEST-942-APPLICATION-ATTACK-SQLI.conf` 에서 `942100` 을 찾아보면 됩니다.)

```text
SecRule ARGS "@detectSQLi" \
    "id:942100, phase:2, block, t:none, t:utf8toUnicode, t:urlDecodeUni, \
     msg:'SQL Injection Attack Detected via libinjection', severity:'CRITICAL', \
     setvar:'tx.anomaly_score_pl1=+%{tx.critical_anomaly_score}'"
```

| 조각 | 뜻 |
| --- | --- |
| `SecRule ARGS` | **어디를 볼지** — 모든 요청 파라미터(쿼리 문자열 + 본문의 필드) |
| `"@detectSQLi"` | **무엇과 맞출지** — libinjection 알고리즘으로 SQL 인젝션 판별 |
| `id:942100` | 규칙 번호. Suricata의 SID와 같은 역할입니다 |
| `phase:2` | 요청 본문까지 다 읽은 시점에 검사 |
| `t:urlDecodeUni` 등 | **정규화** — URL 인코딩을 풀고 비교합니다(`%55NION` → `UNION`) |
| `severity:'CRITICAL'` | 심각도 |
| `setvar:...anomaly_score...` | **점수를 더합니다** ← 오늘의 핵심 |

> **`ARGS` 라는 한 단어가 WAF의 성격을 보여 줍니다.** Suricata는 "패킷 안에 이 글자열이 있는가"를 봤습니다. ModSecurity는 요청을 **파라미터 단위로 분해한 뒤** 각 파라미터를 검사합니다. 그래서 3-3에서 "**파라미터 `q` 에 대해서만** 이 규칙을 빼자" 같은 정밀 조정이 가능합니다.
{: .prompt-tip }

### 1.3 이상 점수(anomaly scoring) — CRS의 심장

CRS 3.x는 **규칙 하나가 걸렸다고 바로 차단하지 않습니다.** 대신 점수를 쌓습니다.

| 심각도 | 더해지는 점수 |
| --- | --- |
| CRITICAL | 5 |
| ERROR | 4 |
| WARNING | 3 |
| NOTICE | 2 |

요청 검사가 끝나면 마지막에 규칙 **`949110`** 이 총점을 임계값과 비교합니다. 기본 임계값은 **5** 이므로, **CRITICAL 규칙 하나만 걸려도 차단**되고, WARNING 두 개(3+3=6)여도 차단됩니다.

> **왜 이렇게 하나요?** 규칙 하나하나는 틀릴 수 있습니다. "따옴표가 있다"는 것만으로 차단하면 `O'Brien` 을 검색한 손님이 막힙니다. 점수제는 **여러 약한 신호가 모였을 때** 결정을 내리게 하고, 임계값이라는 **손잡이 하나**로 전체 민감도를 조절할 수 있게 합니다. 3-3에서 이 손잡이를 직접 돌려 봅니다.
{: .prompt-info }

### 1.4 편집증 수준(Paranoia Level)

CRS의 규칙은 네 단계로 나뉘어 있습니다.

| PL | 성격 | 오탐 |
| --- | --- | --- |
| **PL 1** (기본) | 확실한 공격만 | 적음 — 대부분의 사이트가 여기서 시작 |
| PL 2 | 의심스러운 패턴까지 | 늘어남 |
| PL 3 | 상당히 공격적 | 많음 |
| PL 4 | 거의 화이트리스트에 가까움 | 매우 많음 — 튜닝 없이는 못 씀 |

**오늘은 기본값 PL 1로 진행합니다.** 3-3에서 PL을 올려 보며 "조이면 무슨 일이 생기는가"를 관찰합니다.

---

## 2. 따라 하기: ModSecurity와 CRS 설치

> ### 따라 하기 2-1. 꾸러미 설치
>
> **목적** Nginx용 ModSecurity 모듈과 OWASP CRS 규칙을 받습니다.
{: .prompt-tip }

**1단계.** waf 서버(192.168.59.30)에서 실행합니다.

```bash
sudo apt update
sudo apt install -y libnginx-mod-http-modsecurity modsecurity-crs jq
```

| 꾸러미 | 무엇 |
| --- | --- |
| `libnginx-mod-http-modsecurity` | Nginx에 붙는 **ModSecurity 커넥터**(엔진 `libmodsecurity3` 를 함께 끌고 옵니다) |
| `modsecurity-crs` | **OWASP CRS 3.3** 규칙 모음 |
| `jq` | JSON 로그를 사람이 읽게 정리해 주는 도구 |

> **인터넷이 안 된다면** 3-1의 4-4를 다시 보세요. 이 서버의 인터넷은 **OPNsense를 통해서만** 열려 있습니다. `sudo apt update` 가 실패하면 OPNsense의 OPT1 규칙(DNS 허용·업데이트 허용)부터 확인하고, 급할 때는 VirtualBox에서 `waf` 의 NAT 어댑터를 잠시 켜서 받으세요.
{: .prompt-warning }

> ### 따라 하기 2-2. 설치된 것 확인하기
>
> **목적** 어떤 파일이 어디에 놓였는지 눈으로 익힙니다. 다음 장에서 전부 쓰게 됩니다.
{: .prompt-tip }

**2단계.** 모듈이 Nginx에 등록됐는지 봅니다.

```bash
ls /etc/nginx/modules-enabled/
```

`50-mod-http-modsecurity.conf` 가 보이면 됩니다. 이 파일 안에는 모듈을 불러오는 한 줄이 들어 있습니다.

```bash
cat /etc/nginx/modules-enabled/50-mod-http-modsecurity.conf
```

**3단계.** 규칙과 기본 설정 파일의 위치를 확인합니다.

```bash
ls /etc/nginx/modsecurity.conf /etc/nginx/unicode.mapping
ls /usr/share/modsecurity-crs/owasp-crs.load
ls /etc/modsecurity/crs/
```

| 경로 | 무엇 |
| --- | --- |
| `/etc/nginx/modsecurity.conf` | 엔진 **기본 동작** 설정(꾸러미가 준 것) |
| `/etc/nginx/unicode.mapping` | 유니코드 정규화용 매핑표 |
| `/usr/share/modsecurity-crs/owasp-crs.load` | **CRS 전체를 불러오는 입구 파일** |
| `/etc/modsecurity/crs/crs-setup.conf` | CRS의 동작 설정(임계값·PL 등) |
| `/etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf` | 규칙 **앞**에서 넣는 예외 자리 |
| `/etc/modsecurity/crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf` | 규칙 **뒤**에서 넣는 예외 자리 |

**4단계.** 입구 파일이 무엇을 불러오는지 직접 읽어 봅니다.

```bash
cat /usr/share/modsecurity-crs/owasp-crs.load
```

`crs-setup.conf` → 예외 파일(앞) → `rules/*.conf` → 예외 파일(뒤) 순서로 불러오는 것이 보입니다. **이 순서가 3-3에서 예외를 어디에 쓸지 결정합니다.**

> **★ 출력에 예외 파일 두 개가 안 보인다면**(CRS 버전이 다르면 그럴 수 있습니다), 3-3에서 쓸 예외가 통째로 무시됩니다. 그때는 다음 장에서 만들 `main.conf` 의 **CRS Include 앞뒤에** 아래 두 줄을 직접 넣으면 같은 구조가 됩니다.
>
> ```text
> Include /etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf   ← CRS Include 앞
> Include /etc/modsecurity/crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf   ← CRS Include 뒤
> ```
{: .prompt-warning }

> **인터넷 문서와 다를 수 있습니다.** 많은 안내 글이 `modsecurity.conf-recommended` 파일을 복사하라고 합니다. 그 파일은 **Apache용 꾸러미**에 들어 있어, 우리처럼 Nginx만 설치한 환경에는 **없습니다.** 대신 이 꾸러미가 `/etc/nginx/modsecurity.conf` 를 이미 놓아 주었고, 우리는 다음 장에서 **필요한 값을 우리 파일에 직접 못박아** 어떤 기본값이 들어 있든 결과가 같도록 만들 것입니다.
{: .prompt-warning }

---

## 3. 따라 하기: 설정 — 탐지 모드로 먼저 켜기

### 3.1 우리 설정 파일 만들기

> ### 따라 하기 3-1. `main.conf` 작성
>
> **목적** 엔진 기본값 + CRS + 우리가 확정할 값을 한 파일로 묶습니다.
{: .prompt-tip }

**1단계.** 디렉터리와 로그 자리를 준비합니다.

```bash
sudo mkdir -p /etc/nginx/modsec
sudo touch /var/log/nginx/modsec_audit.log
sudo chown www-data:adm /var/log/nginx/modsec_audit.log
```

**2단계.** 설정 파일을 만듭니다. 통째로 붙여 넣으세요.

```bash
sudo tee /etc/nginx/modsec/main.conf >/dev/null <<'EOF'
# ── 1) 꾸러미가 준 엔진 기본 설정
Include /etc/nginx/modsecurity.conf

# ── 2) OWASP CRS 전체
Include /usr/share/modsecurity-crs/owasp-crs.load

# ── 3) 이 강의에서 확정하는 값 (위에서 무엇이 설정됐든 여기서 덮어씁니다)
SecRuleEngine DetectionOnly
SecRequestBodyAccess On
SecRequestBodyLimit 13107200
SecUnicodeMapFile /etc/nginx/unicode.mapping 20127

# 감사 로그 — 4단계 SIEM이 읽어 갈 파일입니다
SecAuditEngine RelevantOnly
SecAuditLogParts ABCFHZ
SecAuditLogType Serial
SecAuditLog /var/log/nginx/modsec_audit.log
SecAuditLogFormat JSON

# JSON 본문을 "필드"로 분해해서 검사하게 만듭니다
SecRule REQUEST_HEADERS:Content-Type "^application/json" \
    "id:1000001, phase:1, t:none, t:lowercase, pass, nolog, \
     ctl:requestBodyProcessor=JSON"
EOF
```

> **왜 우리 값을 맨 아래에 두나요?** ModSecurity의 설정 지시어는 **나중에 읽은 값이 이깁니다.** 그래서 꾸러미와 CRS를 먼저 불러온 뒤 우리 값을 적으면, 배포판 기본값이 무엇이든 **결과가 우리가 적은 대로 확정**됩니다. 설정 파일을 뒤져 가며 한 줄씩 고치는 것보다 안전하고, 나중에 무엇을 바꿨는지 한눈에 보입니다.
{: .prompt-tip }

> **각 지시어가 하는 일**
>
> | 지시어 | 뜻 |
> | --- | --- |
> | `SecRuleEngine DetectionOnly` | **검사는 하되 막지는 않음** — 오늘 먼저 켤 모드 |
> | `SecRequestBodyAccess On` | **요청 본문까지 읽음** — 로그인 POST 공격을 잡으려면 필수 |
> | `SecAuditEngine RelevantOnly` | 규칙에 걸린 요청만 감사 로그에 남김(전부 남기면 디스크가 금방 참) |
> | `SecAuditLogFormat JSON` | 로그를 **JSON 한 줄**로 남김 — Suricata의 `eve.json` 과 같은 이유 |
> | `SecAuditLogParts ABCFHZ` | 로그에 담을 조각 — A(개요) B(요청 헤더) C(요청 본문) F(응답 헤더) **H(걸린 규칙 목록)** Z(끝) |
> | `id:1000001` 규칙 | `Content-Type: application/json` 이면 본문을 JSON으로 파싱하라는 지시 |
{: .prompt-info }

> **★ 인터넷 예제를 그대로 복사하면 여기서 막힙니다 — v2와 v3는 지시어가 다릅니다.**
>
> 우리가 쓰는 것은 Nginx용 **ModSecurity v3(libmodsecurity 3.0.x)** 입니다. 검색으로 나오는 설정 대부분은 Apache 시절의 **v2** 기준이라, v3에 없는 지시어가 섞여 있습니다. 그런 줄이 하나라도 있으면 **`nginx -t` 가 통째로 실패**합니다.
>
> | 지시어 | v2 | v3 | 이 강의 |
> | --- | --- | --- | --- |
> | `SecTmpDir` · `SecDataDir` | 있음 | **없음(미지원)** | 쓰지 않습니다. v3는 본문 임시 저장을 엔진이 알아서 합니다 |
> | 감사 로그 조각 | A~K 전부 | **A·B·C·E·F·H·J·Z 만 구현** | D·I·G·K가 빠진 **`ABCFHZ`** 를 씁니다 |
> | `SecRuleEngine`·`SecAuditLog*`·`SecRuleRemoveById` | 있음 | 있음(동일) | 그대로 씁니다 |
>
> `SecAuditLogParts` 에 `ABIJDEFHZ` 를 적은 예제를 자주 보게 되는데, 그것은 v2 시절 권장값입니다. 동작하는 경우도 있지만 **구현되지 않은 조각을 요구하는 것**이므로 이 강의는 v3가 실제로 만드는 조각만 적습니다.
{: .prompt-danger }

### 3.2 Nginx에 연결하기

> ### 따라 하기 3-2. 사이트 설정에 두 줄 추가
{: .prompt-tip }

**3단계.** 3-1에서 만든 프록시 설정을 **두 줄이 늘어난 형태**로 다시 씁니다. 아래를 통째로 붙여 넣으세요. (`modsecurity` 로 시작하는 두 줄만 새로 생긴 것이고, 나머지는 3-1과 똑같습니다.)

```bash
sudo tee /etc/nginx/sites-available/juice-proxy.conf >/dev/null <<'EOF'
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name _;

    # ── 오늘 추가하는 두 줄 ──
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;

    access_log /var/log/nginx/juice_access.log;
    error_log  /var/log/nginx/juice_error.log;

    location / {
        proxy_pass http://192.168.59.20:3000;

        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        $connection_upgrade;

        proxy_read_timeout 300s;
    }
}
EOF
```

**4단계.** 두 줄이 제대로 들어갔는지 확인합니다.

```bash
grep -n "modsecurity" /etc/nginx/sites-available/juice-proxy.conf
```

> **두 줄은 반드시 `server { }` 블록 안**에 있어야 합니다. 편집기로 직접 고치고 싶다면 `sudo nano /etc/nginx/sites-available/juice-proxy.conf` 로 열어 `server_name _;` 아래에 적으면 됩니다.
{: .prompt-warning }

**5단계.** 문법을 검사하고 적용합니다.

```bash
sudo nginx -t
sudo systemctl reload nginx
```

`syntax is ok` · `test is successful` 이 나와야 합니다.

> **`nginx -t` 가 실패할 때 — 메시지별 원인**
>
> | 메시지에 보이는 것 | 원인 | 조치 |
> | --- | --- | --- |
> | `unknown directive "modsecurity"` | 모듈이 안 올라옴 | `ls /etc/nginx/modules-enabled/` 에 `50-mod-http-modsecurity.conf` 가 있는지 확인 |
> | `Rules error ... file not found` | Include 경로 오타 | 2-2의 경로 확인 명령을 다시 실행 |
> | `SecAuditLogFormat` 관련 오류 | JSON 지원이 없는 빌드 | 그 줄만 지우고 진행(로그가 옛 형식으로 남습니다) |
> | `Could not open ... audit log` | 로그 파일 권한 | 3-1의 `touch` · `chown` 명령을 다시 실행 |
{: .prompt-warning }

### 3.3 살아 있는지 한 번에 확인

**6단계.** waf 서버 자신에서 일부러 걸릴 만한 요청을 보냅니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://127.0.0.1/?test=<script>alert(1)</script>"
sudo tail -n 3 /var/log/nginx/juice_error.log
```

응답 코드는 **`200`**(아직 DetectionOnly라 막지 않음)이지만, 오류 로그에는 `ModSecurity: Warning. ...` 로 시작하는 줄이 남습니다. **이 줄이 보이면 엔진이 살아 있는 것입니다.**

---

## 4. 따라 하기: 공격 재실행 — 무엇이 걸리는지 먼저 본다

이제 Kali로 돌아가 지금까지 쓰던 공격을 그대로 보냅니다. 아직 **탐지만** 하는 상태이므로 공격은 성공하지만, **로그에는 남습니다.**

> ### 따라 하기 4-1. 세 가지 공격 보내기
{: .prompt-tip }

**1단계.** SQL 인젝션(URL).

```bash
curl -s -o /dev/null -w "%{http_code}\n" -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"
```

**2단계.** 크로스 사이트 스크립팅(XSS).

```bash
curl -s -o /dev/null -w "%{http_code}\n" -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=<script>alert(1)</script>"
```

**3단계.** 경로 탐색(Path Traversal).

```bash
curl -s -o /dev/null -w "%{http_code}\n" --path-as-is \
  "http://192.168.59.30/ftp/../../../../etc/passwd"
```

> **`--path-as-is` 를 붙인 이유** — 이 옵션이 없으면 **curl이 먼저 `../` 를 정리해서** 보내 버립니다. 그러면 서버는 공격 문장을 구경도 못 합니다. 공격 도구가 요청을 손보지 않게 하는 것도 실습의 일부입니다.
{: .prompt-tip }

**4단계.** 로그인 우회(POST **본문**).

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST "http://192.168.59.30/rest/user/login" \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"' OR 1=1;--\",\"password\":\"x\"}"
```

네 개 모두 `200` 또는 `500` 등 **정상적인 응답 코드**가 나옵니다. 아직 아무것도 막지 않았습니다.

> ### 따라 하기 4-2. 감사 로그 읽기
{: .prompt-tip }

**5단계.** waf 서버에서 마지막 기록을 봅니다.

```bash
sudo tail -n 1 /var/log/nginx/modsec_audit.log | jq .
```

JSON 한 덩어리가 예쁘게 펼쳐집니다. 양이 많으니 필요한 것만 뽑아 봅니다.

**6단계.** 걸린 규칙 번호와 메시지만 뽑아 봅니다.

```bash
sudo tail -n 1 /var/log/nginx/modsec_audit.log \
  | jq -r '.transaction.messages[]? | "\(.details.ruleId)  \(.message)"'
```

출력 예시(환경에 따라 규칙 번호는 다를 수 있습니다):

```text
942100  SQL Injection Attack Detected via libinjection
942190  Detects MSSQL code execution and information gathering attempts
949110  Inbound Anomaly Score Exceeded (Total Score: 10)
```

**7단계.** 요청자 IP와 URL도 함께 봅니다.

```bash
sudo tail -n 4 /var/log/nginx/modsec_audit.log \
  | jq -r '"\(.transaction.client_ip)  \(.transaction.request.method) \(.transaction.request.uri)"'
```

> **★ 여기서 반드시 확인할 것 — `client_ip` 가 `192.168.57.10`(Kali) 로 찍힙니다.**
>
> 공격자는 다른 망(`soc-wan`)에 있고 방화벽을 두 번 지나왔는데도, WAF 로그에는 **진짜 공격자 IP** 가 남습니다. OPNsense가 내부 인터페이스 사이에서는 주소를 바꾸지 않고 **경로만 잡아 주기(라우팅)** 때문입니다.
>
> 이것이 **5단계의 전제**입니다. AI가 로그를 읽고 "이 IP를 막아라"라고 판단하려면, 로그에 적힌 IP가 진짜여야 합니다. 만약 여기에 `192.168.59.1` 이나 `192.168.59.30` 이 찍힌다면 자동 차단은 **엉뚱한 대상을 막게 됩니다.**
{: .prompt-danger }

> **마지막 949110 줄이 핵심입니다.** `Inbound Anomaly Score Exceeded (Total Score: 10)` 은 **"점수가 임계값(5)을 넘었다"** 는 뜻입니다. 지금은 DetectionOnly라 넘겨보냈지만, 차단 모드였다면 **여기서 403** 이 나갔을 것입니다. 5장에서 그 스위치를 켭니다.
{: .prompt-info }

> 📷 **필요 이미지 ①** — `jq` 로 뽑아낸 규칙 번호 목록과 `Total Score` 줄이 보이는 터미널 화면.
{: .prompt-tip }

---

## 5. 따라 하기: 차단 모드로 올리기

> ### 따라 하기 5-1. `SecRuleEngine` 을 On으로
{: .prompt-tip }

**1단계.** 설정 한 줄만 바꿉니다.

```bash
sudo sed -i 's/^SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/nginx/modsec/main.conf
grep -n "^SecRuleEngine" /etc/nginx/modsec/main.conf
```

`SecRuleEngine On` 한 줄만 보이면 됩니다.

**2단계.** 적용합니다.

```bash
sudo nginx -t && sudo systemctl reload nginx
```

> **IDS에서 IPS로 올리던 것과 똑같은 순간입니다.** 다만 WAF 쪽이 훨씬 간단합니다. 가로채기 방식을 바꿀 필요도, 별도의 전달 규칙을 만들 필요도 없습니다. **WAF는 처음부터 트래픽 경로 위에 서 있기 때문입니다.** 리버스 프록시라는 배치가 주는 이점이 여기서 드러납니다.
{: .prompt-tip }

> ### 따라 하기 5-2. 같은 공격 다시 — 이번엔 막힙니다
{: .prompt-tip }

**3단계.** Kali에서 4-1의 네 명령을 **그대로** 다시 실행합니다. 이번에는 응답이 이렇게 바뀝니다.

| 공격 | 3-1까지 | 지금 |
| --- | --- | --- |
| SQL 인젝션(URL) | 500 (SQLite 오류까지 도달) | **403** |
| XSS | 200 | **403** |
| 경로 탐색 | 200/404 | **403** |
| 로그인 우회(본문) | **200 + 인증 토큰** | **403** |

**4단계.** 응답 본문도 확인해 봅니다.

```bash
curl -i -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--" | head -12
```

`HTTP/1.1 403 Forbidden` 과 함께 Nginx의 기본 403 페이지가 돌아옵니다. **공격 문장은 표적 서버에 닿지 못했습니다.**

**5단계.** 정상 접속은 그대로인지 확인합니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.59.30/
```

`200` 이 나와야 합니다. Kali 브라우저에서 `http://192.168.59.30` 도 열어 보세요.

> **★ 네 번째 줄이 오늘의 하이라이트입니다.**
>
> 로그인 우회(`' OR 1=1;--`)는 **POST 본문**에 담겨 갑니다. 2단계에서 켠 Suricata 규칙은 **URL만** 보았으므로 이 공격을 잡지 못했습니다. 1-3편에서 성공했던 그 로그인 우회가 2단계를 마친 뒤에도 여전히 성공했던 이유가 이것입니다.
>
> WAF는 `SecRequestBodyAccess On` 과 JSON 파서 덕분에 **본문을 필드로 분해해서** 검사합니다. 그래서 막힙니다. **"계층을 겹치는 이유"가 가장 선명하게 드러나는 지점**입니다.
>
> **이 네 번째만 403이 안 나온다면** 본문 검사 쪽 설정 문제입니다. `main.conf` 에서 두 가지를 확인하세요 — ① `SecRequestBodyAccess On` 이 들어 있는가 ② `id:1000001` 의 JSON 파서 규칙이 들어 있는가. 둘 중 하나가 빠지면 본문이 필드로 쪼개지지 않아 SQL 규칙이 볼 것이 없어집니다.
{: .prompt-danger }

> ### 따라 하기 5-3. 스캐너도 막아 봅니다
{: .prompt-tip }

**6단계.** Kali에서 웹 취약점 스캐너를 돌립니다.

```bash
nikto -h http://192.168.59.30
```

거의 모든 항목이 **403** 으로 돌아옵니다. CRS의 스캐너 탐지 규칙(913100 계열)이 `Nikto` 라는 User-Agent를 알아보기 때문입니다.

> **2단계의 Suricata도 같은 스캔을 잡았습니다.** 그때는 `ET SCAN Nikto` 경보가 떴고, 지금은 WAF가 **응답 자체를 거부**합니다. 같은 공격을 두 계층이 각자 다르게 다루는 것입니다 — 하나는 **기록**, 하나는 **차단**. 4단계에서 이 두 기록을 한 화면에 모읍니다.
{: .prompt-info }

---

## 6. 따라 하기: 두 계층의 기록을 나란히 보기

오늘 만든 것은 방어만이 아닙니다. **같은 사건에 대한 두 번째 기록**을 만들었습니다.

**1단계.** Kali에서 공격을 한 번 더 보냅니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"
```

**2단계.** 두 곳을 확인합니다.

| 어디서 | 무엇이 보이는가 | 남는 파일 |
| --- | --- | --- |
| OPNsense → Services → Intrusion Detection → **Alerts** | `ET WEB_SERVER ... UNION SELECT ...` 경보 (탐지) | `/var/log/suricata/eve.json` |
| waf 서버 → `modsec_audit.log` | 규칙 942100·949110, **차단됨** | `/var/log/nginx/modsec_audit.log` |

**3단계.** WAF 쪽 기록을 다시 확인합니다.

```bash
sudo tail -n 1 /var/log/nginx/modsec_audit.log \
  | jq -r '"\(.transaction.client_ip)  \(.transaction.response.http_code)  \(.transaction.request.uri)"'
```

`192.168.57.10  403  /rest/products/search?q=...` 처럼 나옵니다.

> **같은 공격, 두 개의 로그, 서로 다른 관점.** 경계의 Suricata는 "누가 무엇을 시도했다"를 알려 주고, WAF는 "그것을 어떻게 처리했다"를 알려 줍니다. 4단계 Wazuh SIEM은 이 두 파일을 함께 읽어 **하나의 사건으로 묶습니다.** 그래서 오늘 로그 형식을 굳이 **JSON** 으로 맞춰 둔 것입니다.
{: .prompt-tip }

> **정상 사용 중에 403이 뜬다면 — 그것이 바로 오탐입니다.** Juice Shop을 브라우저로 이리저리 눌러 보면 일부 화면이 깨지거나 일부 요청이 403으로 돌아올 수 있습니다. **고장이 아니라 CRS가 정상 요청을 공격으로 오해한 것**이고, 이것이 실무 WAF 운영의 8할입니다. **3-3에서 이 문제만 집중적으로 다룹니다.** 오늘은 "그런 일이 생긴다"는 것만 확인하고 넘어가세요.
{: .prompt-warning }

---

## 7. 따라 하기: 스냅샷과 점검

**1단계.** 현재 설정을 백업해 둡니다(다음 시간에 마음껏 고치기 위해서입니다).

```bash
sudo cp /etc/nginx/modsec/main.conf /etc/nginx/modsec/main.conf.bak
```

**2단계.** waf 서버를 종료하고 스냅샷을 만듭니다.

```bash
sudo poweroff
```

VirtualBox에서 `waf` 선택 → **스냅샷 → 만들기** → 이름 **`waf-crs-on`**.

점검 목록:

- [ ] `libnginx-mod-http-modsecurity` · `modsecurity-crs` 를 설치했다
- [ ] `/etc/nginx/modsec/main.conf` 를 만들고 CRS를 불러왔다
- [ ] 사이트 설정에 `modsecurity on;` 두 줄을 넣고 `nginx -t` 가 통과했다
- [ ] DetectionOnly 상태에서 **감사 로그에 규칙 번호가 남는 것**을 확인했다
- [ ] 감사 로그의 `client_ip` 가 **192.168.57.10**(진짜 공격자)로 찍힌다
- [ ] `SecRuleEngine On` 으로 바꾼 뒤 SQLi·XSS·경로 탐색이 **403** 이다
- [ ] **POST 본문**의 로그인 우회(`' OR 1=1;--`)도 403이다
- [ ] 정상 접속(`http://192.168.59.30/`)은 여전히 **200** 이다
- [ ] 이상 점수(anomaly score)와 임계값 5의 뜻을 설명할 수 있다

---

## 다음 시간

공격은 막았지만, **정상 손님도 함께 막고 있을지 모릅니다.** WAF를 켜 놓고 아무도 튜닝하지 않으면, 결국 "너무 시끄러워서 껐다"로 끝납니다. **3-3** 에서는 일부러 오탐을 만들어 보고, 감사 로그에서 원인 규칙을 찾아 **정확히 필요한 만큼만 예외를 여는 방법**을 익힙니다. 그리고 3단계 전체를 정리하며, 4단계 SIEM이 읽어 갈 로그 세 가지를 확정합니다.
