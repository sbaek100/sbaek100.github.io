---
title: "[AI 보안 자동화 Lab] 11. SQL Injection 자동 점검 — sqlmap과 CWE-89"
date: 2026-06-10 19:00:00 +0900
categories:
  - 1.응용강의
  - AI자동화
tags:
  - SQL-Injection
  - sqlmap
  - CWE-89
  - DVWA
  - 취약점점검
  - 자동화
pin: false
math: false
mermaid: true
---

# SQL Injection 자동 점검 — sqlmap과 CWE-89

10강까지는 정찰과 스캐닝으로 "의심 목록"을 만들었습니다. 이번 강의부터는 **실제로 취약점을 확정**합니다.  
첫 대상은 웹 취약점의 대표주자 **SQL Injection(SQLi)**입니다. 도구는 **`sqlmap`**, 대상은 DVWA의 SQLi 페이지입니다.

> 이 강의는 손으로 했던 SQL Injection 점검을 **AI 에이전트로 자동화**하는 버전입니다. 수동 실습을 먼저 해 보면 이해가 훨씬 쉽습니다.
{: .prompt-tip }

> **🎯 우리가 지금 왜 이걸 하나요?**  
> 지금까지는 "여기가 취약할 것 같다"는 **의심 목록**만 만들었습니다. 이제 그 의심이 진짜인지 **직접 확인**할 차례입니다. SQL Injection은 한 번 뚫리면 데이터베이스 전체(회원 정보·비밀번호까지)가 통째로 새어 나갈 수 있는, 가장 위험하고 가장 흔한 취약점입니다. 그래서 제일 먼저 확인합니다. 도구(sqlmap)도 잘 갖춰져 있어 자동화 연습으로도 적합합니다. 03강에서 배운 공통 언어로 말하면, 이 취약점의 정식 이름은 **CWE-89**이며 **OWASP Top 10의 A03:Injection** 범주에 속합니다.
{: .prompt-info }

다룰 내용:

1. SQL Injection의 정식 이름 — CWE-89와 A03:Injection
2. 인증 쿠키 준비 (DVWA는 로그인이 필요합니다)
3. sqlmap을 에이전트에 통합
4. AI가 취약 파라미터 식별 → DB 추출
5. DVWA 보안 레벨(Low/Medium)별 차이
6. 점검 인사이트 — SQL Injection의 원리와 근본 대응

---

## 1. 이 취약점의 정식 이름 — CWE-89와 A03:Injection

본격적인 점검에 들어가기 전에, 우리가 찾으려는 대상을 **03강에서 배운 공통 언어**로 다시 불러 봅니다.

- **CWE-89**: *Improper Neutralization of Special Elements used in an SQL Command*  
  "SQL 명령에 사용되는 특수 문자를 제대로 무력화(중화)하지 못함"이라는 뜻입니다. 즉 사용자가 입력한 따옴표(`'`)나 키워드(`OR`, `UNION` 등)가 데이터로 처리되지 않고 **SQL 명령의 일부로 해석**되는 약점의 표준 분류입니다.
- **OWASP Top 10 A03:2021 — Injection**: OWASP는 SQL Injection을 포함한 여러 주입 결함을 **A03:Injection** 범주로 묶습니다. SQLi는 이 범주의 가장 대표적인 사례입니다.

> 03강에서 강조했듯이, AI 에이전트가 "여기 뭔가 이상합니다"라고만 말하면 쓸모가 없습니다. **"이것은 CWE-89(SQL 인젝션) 유형이고, OWASP A03:Injection에 해당합니다"**처럼 공통 언어로 출력해야 분류·우선순위 산정·공유가 가능합니다. 이번 강의의 AI 분석가 프롬프트에도 이 표준 용어를 그대로 출력하도록 지시합니다.
{: .prompt-info }

---

## 2. 먼저: DVWA는 로그인이 필요합니다

DVWA의 취약점 페이지는 로그인 후에만 접근됩니다. 그래서 sqlmap에 **세션 쿠키**를 넘겨야 합니다.  
브라우저(또는 앞선 강의처럼 코드)로 로그인한 뒤 쿠키 두 개가 필요합니다: `PHPSESSID`, `security`.

```bash
# 수동 확인: 브라우저 개발자도구 → Application → Cookies 에서 복사합니다.
# 예시 형태
#   PHPSESSID=abcdef123456;  security=low
```

> DVWA의 SQLi 실습 URL 형태:
> ```text
> http://192.168.57.30/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit
> ```
> 여기서 `id` 파라미터가 점검 대상입니다.
{: .prompt-info }

---

## 3. sqlmap을 에이전트에 통합

`sqli_agent.py`를 만듭니다. 핵심은 sqlmap 함수입니다.

```python
# sqli_agent.py (1/2) — sqlmap 도구
import subprocess

ALLOWED_TARGETS = {"192.168.57.30"}
def _guard(t): return any(x in t for x in ALLOWED_TARGETS)

def run_sqlmap(url: str, cookie: str) -> str:
    """주어진 URL의 파라미터에 SQL Injection이 있는지 점검하고,
    있으면 데이터베이스 목록을 추출합니다."""
    if not _guard(url):
        return f"거부됨: {url} 은 허용 대상이 아닙니다."
    cmd = [
        "sqlmap", "-u", url,
        "--cookie", cookie,
        "--batch",          # 질문에 기본값으로 자동 응답
        "--level", "1", "--risk", "1",
        "--dbs",            # 취약하면 DB 목록 추출
        "--flush-session",  # 이전 결과 캐시 무시
    ]
    r = subprocess.run(cmd, capture_output=True, text=True, timeout=300)
    return r.stdout[-6000:]   # 출력이 길어서 뒷부분만 전달
```

> **파괴적 옵션 주의**: `--os-shell`, `--dump-all`, `--passwords` 같은 옵션은 강력하고 위험합니다. 자동화 초기 단계에서는 **탐지(`--dbs`)까지만** 합니다. 더 깊은 추출은 자율 에이전트·가드레일 강의에서 **사람 승인(human-in-the-loop)** 단계를 거쳐 수행합니다.
{: .prompt-warning }

---

## 4. AI가 점검을 지휘하게 하기

```python
# sqli_agent.py (2/2) — AI가 sqlmap 결과를 해석합니다.
import ollama

MODEL = "qwen2.5:7b"

TARGET = "http://192.168.57.30/DVWA/vulnerabilities/sqli/?id=1&Submit=Submit"
COOKIE = "PHPSESSID=여기에_본인_세션값; security=low"   # 2단계에서 복사한 값

ANALYST = """너는 SQL Injection 점검 분석가다. 아래는 sqlmap 실행 결과다.
다음을 한국어로 요약하라:
1) 취약 여부(취약/안전)와 근거가 된 sqlmap의 핵심 문구
2) 표준 분류(CWE-89, OWASP A03:Injection) 명시
3) 탐지된 주입 기법(예: boolean-based, error-based, UNION 등)
4) 추출된 데이터베이스 이름들
5) 권장 대응(예: Prepared Statement / Parameterized Query 사용)"""

if __name__ == "__main__":
    print("[*] sqlmap 실행 중... (시간이 걸립니다)")
    raw = run_sqlmap(TARGET, COOKIE)

    resp = ollama.chat(model=MODEL, messages=[
        {"role": "system", "content": ANALYST},
        {"role": "user", "content": raw},
    ])
    print("[*] 점검 결과 요약:\n" + resp["message"]["content"])
```

> **세션 쿠키 만료 주의**  
> 코드의 `COOKIE`/`PHPSESSID` 값은 하드코딩되며 일정 시간이 지나면 **만료**됩니다.  
> 스캔이 갑자기 로그인 페이지로 튕기거나 인증 오류가 나면, 브라우저로 DVWA에 다시 로그인한 뒤 **개발자 도구(F12) → Application/Storage → Cookies**에서 새 `PHPSESSID`(및 `security` 쿠키) 값을 복사해 코드에 갱신하세요.  
> 즉, "스캔 실패 = 대개 세션 만료"를 먼저 의심하시기 바랍니다.
{: .prompt-warning }

실행합니다.

```bash
python3 sqli_agent.py
```

**예상 결과** (sqlmap 원본 핵심 + AI 요약):

```text
[*] sqlmap 실행 중...
...
sqlmap: parameter 'id' is vulnerable.
Type: boolean-based blind / error-based / UNION query
available databases [2]:
[*] dvwa
[*] information_schema

[*] 점검 결과 요약:
1) 취약합니다. sqlmap이 'id' 파라미터를 vulnerable로 확정했습니다.
2) 표준 분류: CWE-89, OWASP Top 10 A03:Injection 에 해당합니다.
3) 주입 기법: boolean-based blind, error-based, UNION query 모두 가능
4) 데이터베이스: dvwa, information_schema 가 노출됨
5) 대응: 입력값을 쿼리에 직접 연결하지 말고 Prepared Statement(파라미터 바인딩)
   를 사용하고, 최소 권한 DB 계정을 적용해야 합니다.
```

---

## 5. DVWA 보안 레벨별 차이

쿠키의 `security` 값을 바꿔 AI 에이전트가 어떻게 반응하는지 비교합니다.

| 레벨 | 쿠키 | 결과 | 의미 |
|---|---|---|---|
| **Low** | `security=low` | 즉시 취약 확정 | 입력값 검증이 전혀 없음 |
| **Medium** | `security=medium` | POST 방식 + 약한 필터, sqlmap이 우회 가능 | `mysqli_real_escape_string`만으로는 부족 |
| **High** | `security=high` | 탐지 난이도 급상승 | 별도 세션·토큰 검증으로 자동화가 어려워짐 |

> Medium 레벨은 보통 POST 요청이라, sqlmap에 `-u`(GET) 대신 `--data` 옵션으로 본문 파라미터를 줘야 합니다. AI 에이전트의 system 프롬프트에 "GET이 막히면 POST `--data`로 재시도하라"는 지침을 넣어 **레벨에 따라 전략을 바꾸게** 할 수 있습니다.
{: .prompt-tip }

---

## 6. 점검 인사이트 — SQL Injection의 원리와 근본 대응

> **점검 인사이트 — 왜 주입이 일어나는가**  
> SQLi(SQL Injection: SQL 삽입, CWE-89)의 본질은 **"데이터"와 "명령"의 경계가 무너지는 것**입니다. 코드가 이렇게 짜여 있으면:
> ```php
> $q = "SELECT * FROM users WHERE id = '" . $_GET['id'] . "'";
> ```
> 사용자가 `id=1' OR '1'='1` 을 넣으면, 따옴표가 쿼리 구조를 깨고 사용자의 입력이 **SQL 명령의 일부**가 됩니다. CWE-89의 정의 그대로, 특수 문자(`'`)가 **무력화(중화)되지 못한** 것입니다.
>
> **sqlmap이 자동으로 하는 일**: 참/거짓을 가르는 입력(`' AND 1=1`, `' AND 1=2`)을 보내 응답 차이를 관찰(boolean-based)하거나, DB 오류 메시지를 유도(error-based)하거나, `UNION`으로 다른 테이블을 끌어옵니다.
>
> **근본 대응은 단 하나입니다: Prepared Statement(파라미터 바인딩 / Parameterized Query)**로 데이터와 명령을 분리하는 것입니다. 아래처럼 입력값을 쿼리 문자열에 직접 잇지 않고 자리표시자(placeholder)로 바인딩하면, 입력은 절대 명령으로 해석되지 않습니다.
> ```php
> // 근본 대응 예시 (PHP PDO)
> $stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
> $stmt->execute([$_GET['id']]);   // 입력은 항상 '데이터'로만 처리됩니다.
> ```
> 입력 필터링(escape)은 보조 수단일 뿐이며 Medium 레벨처럼 우회됩니다. AI 에이전트는 탐지뿐 아니라 이 **근본 대응책까지 리포트에 포함**해야 가치가 있습니다.
{: .prompt-info }

---

## 7. 왜 2026년에도 SQL Injection을 점검하는가

SQL Injection은 1998년부터 알려진 "고전" 취약점입니다. 그런데도 2026년 OWASP 분류에서 **인젝션(A03)은 여전히 상위**에 있습니다. 왜 사라지지 않을까요?

- **새 코드가 계속 생깁니다.** 프레임워크가 좋아져도, 직접 문자열로 쿼리를 조립하는 코드는 끊임없이 새로 작성됩니다. 특히 빠르게 만든 내부 도구·관리 페이지에서 자주 나옵니다.
- **피해가 즉각적이고 치명적입니다.** 한 번의 SQLi로 **전체 사용자 정보·인증정보 유출**이 가능합니다. 자동화 공격자에게 가장 "수지맞는" 표적입니다.
- **자동 탐지가 쉬워 공격자도 자동화합니다.** sqlmap 같은 도구로 누구나 대량 탐지가 가능합니다. 그래서 방어자도 **같은 도구로 먼저 자신을 점검**해야 합니다.

---

## 8. 방어자의 사전 조건 (Blue Team)

SQLi를 점검·방어하려면 대상 측에 다음이 전제되어야 합니다.

| 사전 조건 | 목적 |
|---|---|
| 애플리케이션 코드의 **Prepared Statement** 적용 기준선 | 근본 차단. 점검은 이 기준선의 누락을 찾는 것입니다 |
| DB 계정 **최소 권한** (앱 계정에 DROP/파일권한 미부여) | 침해 시 피해 범위 축소 |
| **DB 쿼리 로그 / 슬로우 쿼리 로그** 또는 앱 레벨 쿼리 로깅 | `UNION`, `OR 1=1` 등 비정상 쿼리 흔적 확보 |
| 웹서버 **접근 로그**에 쿼리스트링 기록 | `id=1'` 같은 주입 시도가 URL에 남는지 확인 |
| (선택) **WAF(Web Application Firewall: 웹 방화벽)** 규칙 (SQLi 시그니처) | 1차 차단·탐지 |

> **방어자의 사고법**: sqlmap이 보낸 요청은 접근 로그에 `id=1%27...`, `UNION+SELECT` 같은 **명백한 패턴**으로 남습니다(16~17강의 탐지 편에서 직접 확인합니다). 방어자는 이 패턴을 탐지 규칙으로 만들고, 동시에 **근본 원인(문자열 조립 쿼리)**을 코드에서 제거해야 합니다. 탐지(증상)와 수정(원인)을 함께 가야 합니다.
{: .prompt-tip }

---

## 다음 강의 예고

다음 **12강**에서는 전용 도구가 없는 **XSS·Command Injection** 자동 점검을 다룹니다. AI가 직접 페이로드를 생성·주입·판정하며, 범용 모델과 보안 특화 모델을 비교합니다. SQLi(CWE-89)와 마찬가지로 이들도 OWASP A03:Injection 계열의 약점이라는 점에서 자연스럽게 이어집니다.

---

## 참고 자료

- sqlmap — 자동 SQL Injection 점검 도구: <https://sqlmap.org/>
- OWASP — SQL Injection: <https://owasp.org/www-community/attacks/SQL_Injection>
- MITRE CWE-89 — Improper Neutralization of Special Elements used in an SQL Command: <https://cwe.mitre.org/data/definitions/89.html>
- OWASP Top 10 — A03:2021 Injection: <https://owasp.org/Top10/A03_2021-Injection/>
