---
title: "[AI 보안 자동화 Lab] 10. 자동 리포트 생성과 MCP 패키징 (시리즈 마무리)"
date: 2026-06-08 19:00:00 +0900
categories:
  - 웹보안
  - AI자동화
tags:
  - 리포트
  - CVSS
  - MCP
  - AI-Agent
  - 자동화
pin: false
math: false
mermaid: true
---

# 자동 리포트 생성과 MCP 패키징 (시리즈 마무리)

09강에서 자율 점검 에이전트를 완성했다. 마지막으로, 점검의 **결과물**을 완성한다.  
모의해킹의 가치는 "취약점을 찾았다"가 아니라 **"무엇이, 왜 위험하고, 어떻게 고치는가를 전달하는 리포트"**에 있다.  
그리고 우리가 만든 에이전트를 **MCP 서버**로 패키징해 Claude 같은 다른 도구에서도 재사용한다.

> **🎯 우리가 지금 왜 이걸 하나요?**  
> 보안 점검의 진짜 가치는 "취약점을 찾았다"가 아니라, **"무엇이 왜 위험하고 어떻게 고치는지를 잘 전달하는 것"**에 있어요. 아무리 잘 찾아도 정리해서 전달하지 못하면 아무것도 바뀌지 않으니까요. 이번 강의에서는 AI가 발견 내용을 **읽기 좋은 리포트**로 정리하게 만들어, 점검의 마지막 퍼즐을 맞춰요. 그리고 우리 도구를 다른 곳에서도 재사용할 수 있게 포장(MCP)하는 법도 배웁니다.
{: .prompt-info }

다룰 내용:

1. 좋은 취약점 리포트의 구성
2. AI로 Markdown 리포트 자동 생성
3. CVSS 점수 자동 부여
4. MCP 서버로 패키징
5. 자동화의 한계와 실전 주의점 (시리즈 총정리)

---

## 1. 좋은 취약점 리포트의 구성

전문 모의해킹 리포트의 각 발견 항목은 보통 다음을 담는다.

| 항목 | 내용 |
|---|---|
| 제목 | 취약점 이름 (예: SQL Injection - id 파라미터) |
| 심각도 | Critical/High/Medium/Low + CVSS 점수 |
| 설명 | 무엇이 문제인가 |
| 재현 절차 | 어떻게 확인하는가 (요청 예시) |
| 영향 | 악용 시 무슨 일이 일어나는가 |
| 대응 방안 | 어떻게 고치는가 |

---

## 2. AI로 Markdown 리포트 자동 생성

09강 에이전트의 점검 결과(`messages`)를 입력으로, 리포트를 생성한다. `report_gen.py`를 만든다.

```python
# report_gen.py — 점검 결과를 Markdown 리포트로
import ollama

MODEL = "qwen2.5:7b"

REPORT_PROMPT = """너는 모의해킹 리포트 작성자다. 아래 점검 결과를 바탕으로
한국어 Markdown 취약점 리포트를 작성하라. 각 취약점마다 다음 구조를 지켜라:

## [심각도] 취약점 제목
- **CVSS**: 점수 (벡터)
- **설명**: ...
- **재현 절차**: 번호 목록
- **영향**: ...
- **대응 방안**: ...

마지막에 '## 요약 표'로 전체 취약점을 표로 정리하라.
근거 없는 내용은 지어내지 마라."""

def generate_report(findings_text: str) -> str:
    resp = ollama.chat(model=MODEL, messages=[
        {"role": "system", "content": REPORT_PROMPT},
        {"role": "user", "content": findings_text},
    ])
    return resp["message"]["content"]

if __name__ == "__main__":
    # 09강 에이전트의 최종 요약을 그대로 넣는다 (예시)
    findings = """
    - 포트 22/80/3306 개방, 3306(MySQL) 외부 노출
    - Apache/2.4.58 + DVWA, 보안 헤더(X-Content-Type-Options 등) 누락
    - SQL Injection: id 파라미터 취약, DB(dvwa) 노출
    - Reflected XSS: name 파라미터 반사됨 (Low)
    - Command Injection: ip 파라미터로 id 명령 실행됨
    """
    md = generate_report(findings)
    with open("report.md", "w") as f:
        f.write(md)
    print("[+] report.md 생성 완료\n")
    print(md)
```

실행한다.

```bash
python3 report_gen.py
```

**예상 결과** (`report.md` 일부):

```markdown
## [High] SQL Injection - id 파라미터
- **CVSS**: 8.6 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N)
- **설명**: /vulnerabilities/sqli/ 의 id 파라미터가 입력값을 검증 없이
  SQL 쿼리에 연결하여 임의 쿼리 실행이 가능하다.
- **재현 절차**:
  1. 로그인 후 sqli 페이지 접근
  2. id=1' 입력 시 DB 오류 발생 확인
  3. sqlmap --dbs 로 dvwa, information_schema 추출
- **영향**: 데이터베이스 전체 열람·변조 가능
- **대응 방안**: Prepared Statement(파라미터 바인딩) 적용, 최소 권한 계정 사용

## 요약 표
| 취약점 | 심각도 | CVSS |
|---|---|---|
| SQL Injection | High | 8.6 |
| Command Injection | Critical | 9.8 |
| Reflected XSS | Medium | 6.1 |
| 보안 헤더 누락 | Low | 3.1 |
```

> HTML로 바꾸려면 `pip install markdown` 후 `markdown.markdown(md)`로 변환하거나, `pandoc report.md -o report.html`을 쓴다.
{: .prompt-tip }

---

## 3. CVSS 점수 자동 부여

> **점검 인사이트 — CVSS를 AI에게 맡길 때 주의**  
> CVSS(Common Vulnerability Scoring System)는 취약점 심각도를 0~10으로 표준화한 체계다. AI는 그럴듯한 점수를 빠르게 제시하지만, **벡터(AV/AC/PR…)를 잘못 추정**하기도 한다.  
> 예: Command Injection은 보통 Critical(9.8)이지만, **인증이 필요**(DVWA는 로그인 후)하면 `PR:L`이 되어 점수가 내려간다. AI가 매긴 점수는 **검토 대상**이지 확정값이 아니다. 리포트에는 항상 **사람 검토 단계**를 둔다.
{: .prompt-warning }

---

## 4. MCP 서버로 패키징 (재사용)

우리가 만든 도구를 **MCP(Model Context Protocol) 서버**로 감싸면, Claude Desktop·Claude Code 같은 클라이언트가 이 점검 도구를 **직접** 불러 쓸 수 있다.

```bash
pip install fastmcp
```

```python
# mcp_server.py — 점검 도구를 MCP로 노출
from fastmcp import FastMCP
import subprocess

mcp = FastMCP("dvwa-pentest")
ALLOWED = {"192.168.0.30"}

@mcp.tool()
def nmap_scan(target: str) -> str:
    """대상 IP의 포트/서비스를 스캔한다 (실습 랩 전용)."""
    if target not in ALLOWED:
        return "거부됨: 허용 대상 아님"
    return subprocess.run(["nmap", "-sV", target],
        capture_output=True, text=True, timeout=180).stdout

@mcp.tool()
def nikto_scan(url: str) -> str:
    """웹 URL의 취약점을 스캔한다 (실습 랩 전용)."""
    if not any(a in url for a in ALLOWED):
        return "거부됨: 허용 대상 아님"
    return subprocess.run(["nikto", "-h", url, "-maxtime", "90s"],
        capture_output=True, text=True, timeout=180).stdout

if __name__ == "__main__":
    mcp.run()
```

클라이언트(예: Claude Desktop) 설정에 등록한다.

```json
{
  "mcpServers": {
    "dvwa-pentest": {
      "command": "python3",
      "args": ["/home/kali/ai-pentest-lab/mcp_server.py"]
    }
  }
}
```

> 이제 로컬 Ollama 모델뿐 아니라, MCP를 지원하는 어떤 AI 클라이언트에서도 "192.168.0.30 점검해줘"라고 하면 우리 도구가 호출된다. **도구와 두뇌가 분리**되어, 두뇌(모델/클라이언트)를 자유롭게 바꿀 수 있다 — 시리즈 내내 강조한 "모델은 부품" 원칙의 완성이다.
{: .prompt-info }

---

## 5. 자동화의 한계와 실전 주의점 (시리즈 총정리)

> **점검 인사이트 — 무엇을 자동화했고, 무엇은 못 하는가**  
> **잘하는 것**: 반복적 정찰·스캔, 결과 1차 분류, 알려진 취약점(SQLi 등) 탐지, 리포트 초안 작성.  
> **못/덜 하는 것**:  
> - **비즈니스 로직 취약점** (권한 우회, 결제 조작 등) — 맥락 이해가 필요해 자동화가 어렵다.  
> - **환각(hallucination)** — 모델이 없는 취약점을 "발견했다"고 지어낼 수 있다. **반드시 도구의 실제 출력으로 교차 검증**해야 한다.  
> - **오탐/미탐** — 자동 결과는 "초안"이며, 최종 판단은 사람의 몫이다.  
>
> 즉 우리가 만든 것은 **사람을 대체하는 도구가 아니라, 사람의 1차 작업을 덜어 주는 보조 도구**다. AI가 빠르게 넓게 훑고, 사람이 깊게 검증하는 분업이 가장 현실적이다.
{: .prompt-info }

---

## 6. 여기까지의 회고 — 그리고 남은 절반

```mermaid
graph LR
    A["01~02</br>환경+LLM"] --> B["03~04</br>말 걸기+도구"]
    B --> C["05~06</br>정찰+스캔"]
    C --> D["07~08</br>SQLi+XSS/CMDi"]
    D --> E["09~10</br>자율+리포트"]
    E --> F["11~13</br>오케스트레이션·로그·Red Team"]
```

작은 부품(요청 보내기)에서 출발해, 도구를 하나씩 붙이고, 마침내 **스스로 점검하고 리포트까지 쓰는 자율 에이전트**를 만들었다.  
그리고 처음부터 끝까지 **"작게 검증하고, 모델은 부품처럼 교체한다"**는 원칙과 **"점검은 자동, 악용은 사람 승인"**이라는 안전 원칙을 지켰다.

여기까지가 **공격(점검) 자동화**의 완성이다. 하지만 진짜 보안은 여기서 끝나지 않는다.  
**이 자동 공격이 로그에 어떻게 남고, 방어자는 어떻게 탐지하며, 실전에서 이 도구는 어디에 위치하는가** — 남은 11~13강이 그 절반을 채운다.

---

## 7. 더 나아가기

- **도구 확장**: `gobuster`(05) + `ffuf`, `wpscan`, `nuclei` 같은 도구 추가
- **모델 업그레이드**: `qwen2.5:7b` → 더 큰 모델 또는 보안 특화 모델로 교체
- **RAG 결합**: CVE 데이터베이스를 모델에 연결해 버전 기반 취약점 매칭
- **다른 대상**: 본인이 만든 또 다른 취약 앱(예: Juice Shop, WebGoat)으로 확장

> 그리고 잊지 말 것 — 이 모든 것은 **본인이 소유하거나 명시적 허가를 받은 격리 환경**에서만. 도구가 강력해질수록, 그것을 쓰는 사람의 책임도 커진다.
{: .prompt-danger }

---

## 8. 다음 강의 예고

다음 **11강**에서는 흩어진 단계를 하나의 **오케스트레이션 파이프라인**으로 묶고, 모든 실행을 **점검 로그**로 남긴다.
