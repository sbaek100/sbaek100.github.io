---
title: "[AI 보안관제 구축] 3-3. WAF 운영 — 오탐 튜닝과 사용자 정의 규칙, 그리고 로그"
date: 2027-08-02 09:00:00 +0900
categories:
  - 1.응용강의
  - AI보안관제구축
tags:
  - 보안관제
  - WAF
  - ModSecurity
  - 오탐튜닝
  - 로그관리
pin: false
math: false
mermaid: false
---

> 지난 시간 도달 상태: WAF가 SQL 인젝션·XSS·경로 탐색·본문 로그인 우회를 모두 **403** 으로 막습니다.
> 오늘의 목표: **정상 손님까지 막히는 문제(오탐)** 를 직접 만들어 보고, 감사 로그에서 원인 규칙을 찾아 **딱 필요한 만큼만 예외를 여는** 방법을 익힙니다. 이어서 우리 서비스만의 **사용자 정의 규칙**을 하나 만들고, 4단계 SIEM이 읽어 갈 **로그 세 가지**를 확정하며 3단계를 마칩니다.
> 오늘 켜는 VM: **OPNsense + Kali + victim + waf** 네 대.
{: .prompt-info }

> **이 편의 명령이 기준으로 삼은 버전**
>
> | 구성요소 | 버전 | 이 편에서 영향받는 것 |
> | --- | --- | --- |
> | WAF 엔진 | **ModSecurity v3**(libmodsecurity 3.0.12) | `SecRuleRemoveById`·`SecRuleUpdateTargetById`·`ctl:` 는 v3에서도 그대로 동작 |
> | 규칙셋 | **OWASP CRS 3.3.5** | **규칙 번호(942100·942190·949110)와 예외 파일 이름** |
> | OS | Ubuntu Server 24.04 LTS | 경로 `/etc/modsecurity/crs/` (데비안 계열 패키지 배치) |
>
> **규칙 번호는 CRS 버전에 딸린 값입니다.** 아래 예시의 942100·942190은 CRS 3.3.5에서 실제로 쓰이는 번호지만, 여러분 로그에 다른 번호가 보이면 **로그에 나온 번호가 정답**입니다. 그래서 이 편은 "번호를 외우는" 대신 **로그에서 번호를 찾아내는 절차**를 익히는 데 초점을 둡니다.
{: .prompt-tip }

## 1. 이론 — WAF 운영의 8할은 오탐과의 싸움

### 1.1 네 가지 결과

| | 실제로 공격 | 실제로 정상 |
| --- | --- | --- |
| **WAF가 막음** | 정탐 (원하는 것) | **오탐(False Positive)** — 손님이 못 씀 |
| **WAF가 통과** | **미탐(False Negative)** — 뚫림 | 정상 (원하는 것) |

규칙을 조이면 미탐이 줄고 오탐이 늘어납니다. 풀면 반대가 됩니다. **둘을 동시에 0으로 만드는 설정은 없습니다.** WAF 운영이란 이 둘 사이에서 우리 서비스에 맞는 지점을 찾아가는 일입니다.

> **실무에서 WAF가 실패하는 방식은 대체로 하나입니다.** 뚫려서가 아니라, **오탐이 너무 많아 결국 꺼 버리는 것**입니다. "일단 켜 두고 나중에 보자"로 시작해 장애 신고가 쌓이면, 가장 빠른 해결책이 전원 차단이 되어 버립니다. 그래서 3-2에서 **DetectionOnly로 먼저 켠 것**이고, 오늘 **튜닝하는 법**을 따로 한 시간 잡은 것입니다.
{: .prompt-warning }

### 1.2 튜닝의 세 가지 원칙

| 원칙 | 뜻 | 어기면 |
| --- | --- | --- |
| **좁게** | 규칙 전체를 끄지 말고, **어느 경로의 어느 파라미터**인지까지 좁혀서 예외 | 예외 하나가 사이트 전체의 구멍이 됨 |
| **근거를 남기고** | 왜 이 예외를 넣었는지 주석으로 기록 | 반년 뒤 아무도 지우지 못하는 예외가 쌓임 |
| **되돌릴 수 있게** | 예외는 정해진 파일에만, 원본 규칙은 건드리지 않음 | CRS를 업데이트하면 수정이 전부 날아감 |

세 번째가 특히 중요합니다. **`/usr/share/modsecurity-crs/rules/` 안의 파일은 절대 고치지 않습니다.** 그 파일들은 꾸러미가 관리하는 영역이라 업데이트 때 덮어써집니다. 우리가 쓰는 자리는 따로 있습니다.

### 1.3 예외를 쓰는 자리 — 앞과 뒤

3-2에서 `owasp-crs.load` 를 읽어 보았습니다. 불러오는 순서는 이랬습니다.

```text
crs-setup.conf                          (설정)
REQUEST-900-EXCLUSION-RULES-BEFORE-CRS  ← 규칙보다 "앞"
rules/*.conf                            (CRS 본체 — 절대 수정 금지)
RESPONSE-999-EXCLUSION-RULES-AFTER-CRS  ← 규칙보다 "뒤"
```

예외 파일이 **앞뒤로 두 개** 있는 데에는 이유가 있습니다.

| 파일 | 여기에 쓰는 것 | 왜 |
| --- | --- | --- |
| **BEFORE**(900) | 편집증 수준(PL), 이상 점수 임계값 같은 **설정 변수** | CRS가 초기화되기 **전에** 값을 정해야 하기 때문 |
| **AFTER**(999) | `SecRuleRemoveById` · `SecRuleUpdateTargetById` 같은 **규칙 손질** | 손질하려는 규칙이 **이미 읽혀 있어야** 하기 때문 |

> **"없는 규칙은 지울 수 없습니다."** `SecRuleRemoveById 942100` 을 CRS보다 앞에 적으면, 그 시점에는 942100 규칙이 아직 존재하지 않아 아무 일도 일어나지 않습니다(오류가 나기도 합니다). 반대로 PL 설정을 뒤에 적으면 이미 초기화가 끝난 뒤라 늦습니다. **순서가 곧 문법입니다.**
{: .prompt-danger }

---

## 2. 따라 하기: 오탐을 직접 만들어 보기

우리 실습용 쇼핑몰(Juice Shop)이 **보안 서적도 파는 가게**라고 생각해 봅시다. 손님이 `SQL UNION SELECT 강의` 라는 책을 찾으려고 검색창에 그대로 입력합니다. 공격 의도는 전혀 없습니다.

> ### 따라 하기 2-1. 정상 손님이 막히는 장면
{: .prompt-tip }

**1단계.** Kali에서 그 검색을 흉내 냅니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=SQL UNION SELECT 강의"
```

**403** 이 나옵니다. 공격이 아닌데 막혔습니다. **이것이 오탐입니다.**

**2단계.** 비교를 위해 평범한 검색도 해 봅니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=apple juice"
```

이쪽은 **200** 입니다. 검색 기능 자체가 막힌 게 아니라, **특정 단어가 든 검색만** 막히는 것입니다.

> **오탐은 이렇게 생깁니다.** 규칙은 "이 문자열이 위험하다"고 알 뿐, **손님의 의도**는 모릅니다. 보안 서적을 파는 서점, SQL을 가르치는 학원, 개발자 커뮤니티 게시판 — 이런 서비스에서는 공격 문자열이 **정상 콘텐츠**입니다. 실무의 오탐 대부분이 이런 모양입니다.
{: .prompt-info }

> ### 따라 하기 2-2. 원인 규칙 찾기
>
> **목적** "무엇이 막았는가"를 추측하지 않고 로그에서 확인합니다.
{: .prompt-tip }

**3단계.** waf 서버에서 방금 요청의 기록을 봅니다.

```bash
sudo tail -n 1 /var/log/nginx/modsec_audit.log \
  | jq -r '.transaction.messages[]? | "\(.details.ruleId)\t\(.message)"'
```

**출력 예시**(여러분 화면의 번호는 다를 수 있습니다):

```text
942100  SQL Injection Attack Detected via libinjection
942190  Detects MSSQL code execution and information gathering attempts
949110  Inbound Anomaly Score Exceeded (Total Score: 10)
```

**4단계.** 무엇에 걸렸는지, 어느 파라미터였는지까지 봅니다.

```bash
sudo tail -n 1 /var/log/nginx/modsec_audit.log \
  | jq -r '.transaction.messages[]? | "[\(.details.ruleId)] data=\(.details.data)"'
```

`data=` 뒤에 **실제로 규칙에 걸린 조각**이 보입니다. 여기까지 확인하면 "감이 아니라 근거"로 튜닝을 시작할 수 있습니다.

> **읽는 순서를 정해 두세요.**
>
> 1. **`949110` 줄의 Total Score** — 임계값(5)을 얼마나 넘었는가. 6이면 아슬아슬하고, 15면 여러 규칙이 겹친 것입니다
> 2. **949110을 뺀 나머지 번호들** — 실제로 점수를 올린 범인들
> 3. **`data`** — 그 규칙이 문제 삼은 문자열
>
> 949110은 "합계를 계산한 심판"일 뿐이므로, **949110을 예외 처리하면 안 됩니다.** 그것을 끄면 WAF 전체가 차단을 멈춥니다.
{: .prompt-danger }

> 📷 **필요 이미지 ①** — `jq` 출력에서 942100·942190·949110 세 줄과 Total Score가 보이는 터미널 화면.
{: .prompt-tip }

---

## 3. 따라 하기: 예외를 여는 세 가지 방법

원인을 알았으니 예외를 넣습니다. 넓은 것부터 좁은 것 순으로 **직접 비교**해 봅니다.

| 방법 | 문법 | 범위 | 위험 |
| --- | --- | --- | --- |
| ① 규칙 자체를 제거 | `SecRuleRemoveById 942100` | **사이트 전체** | 매우 큼 |
| ② 특정 파라미터만 제외 | `SecRuleUpdateTargetById 942100 "!ARGS:q"` | 모든 경로의 `q` | 중간 |
| ③ 특정 경로의 특정 파라미터만 | `ctl:ruleRemoveTargetById=942100;ARGS:q` | 검색 API의 `q` | 작음 |

> ### 따라 하기 3-1. ① 넓은 예외를 넣고, 무슨 일이 생기는지 본다
{: .prompt-tip }

**1단계.** 뒤쪽 예외 파일에 규칙 제거를 적습니다.

```bash
sudo tee -a /etc/modsecurity/crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf >/dev/null <<'EOF'

# [실습용 · 임시] 서적 검색 오탐 대응 — 나중에 되돌립니다
SecRuleRemoveById 942100
EOF
sudo nginx -t && sudo systemctl reload nginx
```

**2단계.** 정상 검색과 진짜 공격을 **둘 다** 다시 보냅니다.

```bash
# (가) 정상 손님
curl -s -o /dev/null -w "정상검색: %{http_code}\n" -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=SQL UNION SELECT 강의"

# (나) 진짜 공격
curl -s -o /dev/null -w "공격   : %{http_code}\n" -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"
```

**3단계.** 결과를 읽습니다. 둘 중 하나가 나옵니다.

| 결과 | 뜻 | 다음 행동 |
| --- | --- | --- |
| 정상 200 · 공격 **403** | 942100을 뺐어도 **다른 규칙들이 점수를 채워** 공격은 여전히 막힘 | 다행이지만, 방어가 한 겹 얇아진 것은 사실 |
| 정상 200 · 공격 **200/500** | 942100이 **유일한 방어선**이었음 → 예외가 곧 구멍이 됨 | 이 예외는 이대로 두면 안 됨 |
| 정상 **403** · 공격 403 | 942100 말고 **다른 규칙도** 정상 검색을 막고 있음 | 로그를 다시 보고 그 번호도 처리 |

**4단계.** 어떤 경우든 로그를 다시 확인해 근거를 잡습니다.

```bash
sudo tail -n 2 /var/log/nginx/modsec_audit.log \
  | jq -r '"[\(.transaction.response.http_code)] " + ([.transaction.messages[]?.details.ruleId] | join(","))'
```

> **★ 여기서 배울 것 — 예외는 언제나 "그 자리를 약하게" 만듭니다.**
>
> 위 세 결과 중 무엇이 나오든 결론은 같습니다. 예외를 넣은 자리에서 그 규칙은 더 이상 일하지 않습니다. CRS가 **여러 규칙의 점수를 합산**하는 구조라 한 겹이 빠져도 버티는 경우가 많을 뿐입니다.
>
> 그래서 순서를 이렇게 잡아야 합니다. **(1) 정말 예외가 필요한가**(앱을 고칠 수 있으면 그쪽이 정답입니다) → **(2) 가장 좁은 예외는 무엇인가** → **(3) 넣은 뒤 공격이 여전히 막히는지 반드시 재시험.** 3단계를 건너뛴 튜닝은 튜닝이 아니라 방어 해제입니다.
{: .prompt-danger }

> ### 따라 하기 3-2. ③ 가장 좁은 예외로 바꾸기
{: .prompt-tip }

**5단계.** 방금 넣은 넓은 예외를 지웁니다.

```bash
sudo sed -i '/SecRuleRemoveById 942100/d; /서적 검색 오탐 대응/d' \
  /etc/modsecurity/crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
sudo tail -n 3 /etc/modsecurity/crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
```

두 줄이 사라졌는지 눈으로 확인하고 넘어갑니다.

**6단계.** 앞쪽 예외 파일에 **경로와 파라미터를 못박은** 예외를 넣습니다.

```bash
sudo tee -a /etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf >/dev/null <<'EOF'

# ── 오탐 대응 (2027-08-02, 담당: student)
# 사유: 보안 서적 판매로 인해 검색어에 SQL 키워드가 정상적으로 들어옴
# 범위: /rest/products/search 의 q 파라미터에 한해 942100 미적용
# 재검토: 검색 API에 서버측 입력 검증이 들어가면 이 예외를 제거할 것
SecRule REQUEST_URI "@beginsWith /rest/products/search" \
    "id:1000020, phase:1, pass, nolog, \
     ctl:ruleRemoveTargetById=942100;ARGS:q"
EOF
sudo nginx -t && sudo systemctl reload nginx
```

**7단계.** 세 가지를 확인합니다.

```bash
curl -s -o /dev/null -w "검색 오탐   : %{http_code}\n" -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=SQL UNION SELECT 강의"

curl -s -o /dev/null -w "검색 공격   : %{http_code}\n" -G "http://192.168.59.30/rest/products/search" \
  --data-urlencode "q=x')) UNION SELECT * FROM information_schema.tables;--"

curl -s -o /dev/null -w "로그인 공격 : %{http_code}\n" -X POST "http://192.168.59.30/rest/user/login" \
  -H "Content-Type: application/json" -d "{\"email\":\"' OR 1=1;--\",\"password\":\"x\"}"
```

**세 번째 줄이 핵심입니다.** 로그인 경로는 예외 범위 밖이므로 **여전히 403** 이어야 합니다. 예외를 좁게 만든다는 것은 바로 이런 뜻입니다 — **검색창 하나 때문에 로그인 창의 방어까지 풀지 않는 것.**

> **여전히 검색 오탐이 403이라면**, 2-2에서 본 로그에 942100 말고 다른 번호(942190 등)도 있었다는 뜻입니다. 같은 방식으로 그 번호를 예외에 추가하세요.
>
> ```text
> ctl:ruleRemoveTargetById=942100;ARGS:q, \
> ctl:ruleRemoveTargetById=942190;ARGS:q
> ```
>
> **한 번에 하나씩 추가하고, 추가할 때마다 위 세 명령을 다시 돌리세요.** 여러 개를 한꺼번에 넣으면 어느 것이 효과가 있었는지 알 수 없게 되고, 필요 이상으로 넓은 예외가 남습니다.
{: .prompt-warning }

---

## 4. 따라 하기: 손잡이 두 개 — 편집증 수준과 임계값

예외를 하나하나 다는 것 말고, **전체 민감도**를 조절하는 방법도 있습니다.

> ### 따라 하기 4-1. 조여 보기 (PL 2)
>
> **목적** "조이면 무슨 일이 생기는가"를 실제로 관찰합니다.
{: .prompt-tip }

**1단계.** 앞쪽 예외 파일에 설정 변수를 적습니다.

```bash
sudo tee -a /etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf >/dev/null <<'EOF'

# [실습] 편집증 수준을 2로 올려 본다 — 관찰이 끝나면 지웁니다
SecAction "id:1000010, phase:1, nolog, pass, t:none, setvar:tx.paranoia_level=2"
EOF
sudo nginx -t && sudo systemctl reload nginx
```

**2단계.** Kali 브라우저로 `http://192.168.59.30` 에 접속해 상품을 둘러보고, 장바구니에 담고, 회원 가입을 시도해 봅니다. 화면이 군데군데 깨지거나 요청이 403으로 돌아오는 것을 보게 됩니다.

**3단계.** 무엇이 막혔는지 확인합니다.

```bash
sudo tail -n 20 /var/log/nginx/modsec_audit.log \
  | jq -r '"\(.transaction.request.uri)\t" + ([.transaction.messages[]?.details.ruleId] | join(","))' \
  | sort | uniq -c | sort -rn | head
```

가장 자주 걸린 URL과 규칙 번호가 순서대로 나옵니다. **운영에서 튜닝 대상을 고르는 방식이 정확히 이것입니다** — 많이 걸리는 것부터 처리합니다.

**4단계.** 관찰이 끝났으면 되돌립니다.

```bash
sudo sed -i '/id:1000010/d; /편집증 수준을 2로/d' \
  /etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
sudo nginx -t && sudo systemctl reload nginx
```

> ### 따라 하기 4-2. 풀어 보기 (임계값)
{: .prompt-tip }

**5단계.** 이번엔 임계값을 올려 봅니다. 기본 5를 10으로 바꾸면, **CRITICAL 규칙 하나(5점)로는 차단되지 않고 두 개는 되어야** 막힙니다.

```bash
sudo tee -a /etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf >/dev/null <<'EOF'

# [실습] 이상 점수 임계값을 10으로 — 관찰이 끝나면 지웁니다
SecAction "id:1000011, phase:1, nolog, pass, t:none, setvar:tx.inbound_anomaly_score_threshold=10"
EOF
sudo nginx -t && sudo systemctl reload nginx
```

**6단계.** 오탐 검색과 진짜 공격을 다시 보내 비교합니다. 점수가 5를 겨우 넘던 요청은 이제 통과하고, 점수가 크게 쌓인 요청은 여전히 막힙니다. **임계값은 "몇 점부터 범인으로 볼 것인가"를 정하는 손잡이**입니다.

**7단계.** 관찰이 끝나면 **반드시 되돌립니다.** (임계값을 올려 둔 채 두면 방어가 그만큼 느슨해집니다.)

```bash
sudo sed -i '/id:1000011/d; /이상 점수 임계값을 10으로/d' \
  /etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
sudo nginx -t && sudo systemctl reload nginx
grep -c "1000011" /etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
```

마지막 명령이 **`0`** 을 돌려주면 깨끗이 지워진 것입니다.

> **★ 왜 이 두 줄은 "앞" 파일에 써야 하나요 — 3-2와 반대인 것 같은데요?**
>
> 3-2에서는 **나중에 쓴 값이 이긴다**고 했고, 여기서는 **먼저 정한 값이 이긴다**고 합니다. 모순이 아니라 **종류가 다릅니다.**
>
> | 종류 | 예 | 규칙 |
> | --- | --- | --- |
> | **설정 지시어** | `SecRuleEngine`, `SecAuditLog` | 파일을 읽는 동안 계속 덮어씀 → **마지막 값이 이김** |
> | **런타임 변수** | `tx.paranoia_level`, `tx.inbound_anomaly_score_threshold` | CRS 초기화 규칙(901번대)이 **"아직 값이 없으면"** 기본값을 넣음 → **먼저 정한 값이 이김** |
>
> 그래서 PL·임계값은 CRS보다 **앞**에서 정해야 하고, 규칙 손질(`SecRuleRemoveById`)은 CRS보다 **뒤**에서 해야 합니다.
{: .prompt-info }

---

## 5. 따라 하기: 우리 서비스만의 규칙 — 가상 패치 한 건

CRS는 "일반적인 공격"을 막습니다. 우리 서비스에만 있는 문제는 우리가 규칙을 씁니다.

Juice Shop에는 `/ftp` 경로에 내부 파일이 그대로 노출되어 있습니다. 애플리케이션 코드를 고칠 수 없다면(3-1의 **가상 패치**), WAF에서 그 경로를 막으면 됩니다.

> ### 따라 하기 5-1. 노출 확인
{: .prompt-tip }

**1단계.** Kali에서 확인합니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://192.168.59.30/ftp/"
```

**2단계.** 브라우저 `http://192.168.59.30/ftp/` 로도 열어 파일 목록이 보이는지 확인합니다.

> ### 따라 하기 5-2. 규칙 작성
{: .prompt-tip }

**3단계.** 우리 설정 파일 맨 아래에 규칙을 붙입니다. (CRS 예외 파일이 아니라 **우리 파일**에 씁니다. CRS와 무관한 우리 정책이기 때문입니다.)

```bash
sudo tee -a /etc/nginx/modsec/main.conf >/dev/null <<'EOF'

# ── 사용자 정의 규칙 (가상 패치)
# 사유: 애플리케이션이 /ftp 경로에 내부 파일을 노출함. 코드 수정 전까지 WAF에서 차단
SecRule REQUEST_URI "@beginsWith /ftp" \
    "id:1000100, phase:1, deny, status:403, log, \
     msg:'Blocked access to exposed /ftp directory', \
     tag:'local-policy', severity:'CRITICAL'"
EOF
sudo nginx -t && sudo systemctl reload nginx
```

**4단계.** 막히는지 확인합니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://192.168.59.30/ftp/"
curl -s -o /dev/null -w "%{http_code}\n" "http://192.168.59.30/"
```

`/ftp/` 는 **403**, 정상 페이지는 **200** 이어야 합니다.

**5단계.** 우리 규칙이 로그에 어떻게 남는지 봅니다.

```bash
sudo tail -n 1 /var/log/nginx/modsec_audit.log \
  | jq -r '.transaction.messages[]? | "\(.details.ruleId)  \(.message)"'
```

`1000100  Blocked access to exposed /ftp directory` 가 보입니다.

> **`id` 번호를 아무거나 쓰면 안 됩니다.** ModSecurity는 같은 번호가 두 번 나오면 설정 자체를 거부합니다. CRS가 쓰는 번호대(900000~999999)를 피해, 우리 규칙은 **1000000번대**로 정해 두었습니다. 이렇게 대역을 나눠 두면 CRS를 업데이트해도 충돌하지 않습니다.
{: .prompt-warning }

> **`phase:1` 로 쓴 이유** — 이 판단에는 요청 본문이 필요 없습니다. URL만 보면 되므로 **가장 이른 단계에서 끊는 것**이 효율적입니다. 반대로 본문을 봐야 하는 규칙은 `phase:2` 여야 합니다.
{: .prompt-info }

---

## 6. 따라 하기: 로그 정리 — 4단계가 읽어 갈 것들

3단계를 마치며, 다음 단계에서 모아 갈 로그를 확정합니다.

**1단계.** 세 파일이 각각 쌓이고 있는지 확인합니다.

```bash
# WAF 서버(59.30)에서
ls -lh /var/log/nginx/modsec_audit.log /var/log/nginx/juice_access.log
sudo wc -l /var/log/nginx/modsec_audit.log
```

| 어디 | 파일 | 무엇을 알려 주는가 |
| --- | --- | --- |
| OPNsense | `/var/log/suricata/eve.json` | 경계에서 **탐지된** 공격 (2-3) |
| waf(59.30) | `/var/log/nginx/modsec_audit.log` | WAF가 **판단하고 차단한** 웹 공격 |
| waf(59.30) | `/var/log/nginx/juice_access.log` | 모든 웹 요청의 **접근 기록**(정상 포함) |

**2단계.** 감사 로그가 무한정 커지지 않도록 관리 설정을 확인합니다.

```bash
grep -n "SecAuditEngine" /etc/nginx/modsec/main.conf
ls /etc/logrotate.d/nginx
```

`SecAuditEngine RelevantOnly` 는 **규칙에 걸린 요청만** 기록하라는 뜻입니다. 이것을 `On` 으로 바꾸면 모든 요청이 JSON으로 쌓여 디스크가 금방 찹니다. 실습 환경에서는 반드시 `RelevantOnly` 로 둡니다.

> **왜 세 개나 필요한가요?** 4단계에서 하나의 사건을 조사할 때 이렇게 쓰입니다.
>
> 1. `eve.json` — "192.168.57.10이 SQL 인젝션을 **시도했다**"
> 2. `modsec_audit.log` — "그 요청은 WAF가 **403으로 막았다**"
> 3. `juice_access.log` — "같은 IP가 그 전후로 **무엇을 더 눌러 봤는가**"
>
> 셋을 합쳐야 "정찰 → 공격 → 차단"이라는 **이야기**가 됩니다. 그리고 5단계의 AI는 이 이야기를 읽고 "이 IP를 차단할 것인가"를 판단하게 됩니다.
{: .prompt-tip }

> **문제가 생겼을 때 되돌리는 순서** — WAF 때문에 서비스가 멈춘 것 같으면 아래 순서로 한 단계씩 풀면서 확인하세요. 확인 명령은 `curl -s -o /dev/null -w "%{http_code}\n" http://192.168.59.30/` 입니다.
>
> | 순서 | 되돌릴 것 | 방법 |
> | --- | --- | --- |
> | 1 | 오늘 추가한 예외·사용자 정의 규칙 | 해당 줄을 주석 처리하고 reload |
> | 2 | 차단을 잠시 멈춤 | `SecRuleEngine DetectionOnly` 로 바꾸고 reload |
> | 3 | WAF 자체를 끔 | 사이트 설정의 `modsecurity on;` → `modsecurity off;` |
> | 4 | 프록시까지 의심 | victim에서 `sudo docker ps` 로 Juice Shop 확인 |
>
> **3-2에서 만든 백업(`main.conf.bak`)이 있으므로 언제든 원상 복구할 수 있습니다.** `sudo cp /etc/nginx/modsec/main.conf.bak /etc/nginx/modsec/main.conf`
{: .prompt-warning }

---

## 7. 3단계 정리 — before / after

같은 공격 네 가지가 단계마다 어떻게 달라졌는지 한눈에 봅니다.

| 공격 | 1단계(무방비) | 2단계(방화벽+IDS) | **3단계(WAF)** |
| --- | --- | --- | --- |
| 포트 스캔(nmap) | 전부 노출 | 3000만 열림 + 탐지 | **표적은 전부 filtered**, WAF의 80만 열림 |
| SQL 인젝션(URL) | 성공 | 성공 + 탐지 | **403 차단** |
| 로그인 우회(POST 본문) | 성공 | **성공**(URL 규칙이 못 봄) | **403 차단** |
| 스캐너(nikto) | 성공 | 탐지 | **거의 전부 403** |
| 정상 웹 접속 | 성공 | 성공 | 성공(오탐은 튜닝으로 해결) |

3단계를 마치면 방어는 **세 겹**이 됩니다.

```text
[ Kali 공격자 ]
      │
      ▼  ① 방화벽 — 열린 문만 통과 (2-2)
[ OPNsense ] ─ ② Suricata IDS — 지나가는 패턴을 탐지·기록 (2-3)
      │
      ▼  ③ WAF — HTTP 요청을 해석해 공격이면 차단 (3-2 · 3-3)
[  waf 59.30  ]
      │
      ▼
[ victim 59.20 ]  표적은 바깥에서 보이지 않음
```

> **그런데 아직 빠진 것이 있습니다.** 세 겹의 방어가 각자 열심히 일하고 있지만, **기록은 세 군데에 흩어져 있습니다.** 지금 "오늘 우리 시스템에 무슨 일이 있었나"를 알려면 OPNsense 화면, waf 서버 터미널, 그리고 각각의 로그 파일을 따로 뒤져야 합니다. 공격자가 밤에 다녀갔다면 아침까지 아무도 모릅니다. **4단계가 필요한 이유입니다.**
{: .prompt-info }

---

## 8. 따라 하기: 스냅샷과 점검

**1단계.** 지금 설정을 백업합니다.

```bash
sudo tar czf ~/waf-config-$(date +%F).tar.gz \
  /etc/nginx/modsec/main.conf \
  /etc/nginx/sites-available/juice-proxy.conf \
  /etc/modsecurity/crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf \
  /etc/modsecurity/crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
ls -lh ~/waf-config-*.tar.gz
```

**2단계.** 서버를 종료하고 스냅샷을 만듭니다.

```bash
sudo poweroff
```

VirtualBox에서 `waf` 선택 → **스냅샷 → 만들기** → 이름 **`waf-tuned`**.

점검 목록:

- [ ] 정상 검색어가 막히는 **오탐을 재현**했다
- [ ] 감사 로그에서 **원인 규칙 번호와 Total Score** 를 읽었다
- [ ] 넓은 예외(`SecRuleRemoveById`)와 좁은 예외(`ctl:ruleRemoveTargetById`)의 차이를 설명할 수 있다
- [ ] 좁은 예외를 넣은 뒤 **로그인 경로의 방어는 그대로**인 것을 확인했다
- [ ] 예외에 **사유·범위·재검토 조건**을 주석으로 남겼다
- [ ] PL과 임계값이 왜 **BEFORE 파일**에 들어가는지 설명할 수 있다
- [ ] 사용자 정의 규칙(`id:1000100`)으로 `/ftp` 를 막았다
- [ ] 4단계가 읽어 갈 **로그 세 가지**의 위치를 안다
- [ ] 스냅샷 `waf-tuned` 와 설정 백업(`~/waf-config-*.tar.gz`)을 만들었다

---

## 다음 단계 예고 (4단계)

방어는 세 겹이 되었지만, **보는 사람이 없으면 방어는 절반짜리**입니다. **4단계** 에서는 관제망(`soc-lan`)에 **Wazuh SIEM 서버(VM5 · 192.168.58.100)** 를 세우고, 지금까지 만든 로그를 한 곳으로 모읍니다.

- OPNsense의 `eve.json`(Suricata 탐지)
- waf 서버의 `modsec_audit.log`(WAF 차단)와 접근 로그

같은 공격을 다시 한 번 보냈을 때, **여러 계층의 기록이 한 화면에 시간 순서로 늘어서는 것**을 보게 됩니다. 그리고 그 화면이 바로 5단계에서 **AI가 읽을 자료**가 됩니다.
