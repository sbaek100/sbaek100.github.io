---
title: "[AI 보안 자동화 Lab] 09. 정찰 자동화 — 여러 도구와 결과 구조화"
date: 2026-06-10 17:00:00 +0900
categories:
  - 1.응용강의
  - AI자동화
tags:
  - Tool-Calling
  - whatweb
  - gobuster
  - 정찰
  - 자동화
pin: false
math: false
mermaid: true
---

# 정찰 자동화 — 여러 도구와 결과 구조화

08강에서는 AI에게 도구(nmap) 하나를 붙였습니다. 이번 강의에서는 도구를 **세 개**로 늘리고,  
AI가 상황에 맞게 **어떤 도구를 쓸지 스스로 선택**하게 합니다. 그리고 결과를 사람과 코드가 모두 쓰기 좋은 **구조화 데이터(JSON)**로 정리합니다.

> **🎯 우리가 지금 왜 이걸 하나요?**  
> 낯선 건물에 들어가기 전에 먼저 **출입구가 몇 개고 어디에 있는지**를 살펴봅니다. 보안 점검도 똑같습니다. 무엇을 어떻게 점검할지 정하려면, 먼저 **"무엇이 거기 있는지"**부터 알아야 합니다. 이 단계를 정찰(Reconnaissance)이라고 부릅니다. 이것은 02강에서 살펴본 **사이버 킬체인의 1단계(정찰, Reconnaissance)**와 직접 연결됩니다. 공격자가 가장 먼저 하는 일을, 이번에는 방어자 관점에서 자동화해 봅니다. 이번 강의에서는 AI가 여러 도구를 스스로 골라 가며 대상의 **지도**를 그리게 만듭니다.
{: .prompt-info }

다룰 내용:

1. 도구 추가 — `whatweb`(기술 식별), `gobuster`(숨은 경로 탐색)
2. 여러 도구를 한 에이전트에 등록
3. AI가 도구를 스스로 고르는 모습 관찰
4. 정찰 결과를 JSON으로 구조화
5. 점검 인사이트 — 웹 핑거프린팅과 디렉터리 탐색

---

## 1. 새 도구 두 개 준비

```bash
# 보통 Kali에 기본 설치되어 있습니다. 없으면 설치합니다.
sudo apt install -y whatweb gobuster
which whatweb gobuster
```

- **whatweb**: 웹 사이트가 어떤 기술(서버, CMS, 프레임워크)로 만들어졌는지 식별합니다.
- **gobuster**: 단어 목록(wordlist)으로 **숨겨진 디렉터리/페이지**를 빠르게 탐색합니다.

---

## 2. 세 도구를 함수로 감싸기

`recon_agent.py`를 만듭니다. 08강의 `run_nmap`에 두 함수를 더합니다.

```python
# recon_agent.py (1/4) — 세 개의 도구 함수
import subprocess

ALLOWED_TARGETS = {"192.168.0.30"}

def _guard(target: str) -> bool:
    # IP 또는 URL 안에 허용 대상이 들어 있는지 확인합니다.
    return any(t in target for t in ALLOWED_TARGETS)

def run_nmap(target: str) -> str:
    """열린 포트와 서비스 버전을 스캔합니다."""
    if not _guard(target):
        return f"거부됨: {target} 은 허용 대상이 아닙니다."
    r = subprocess.run(["nmap", "-sV", "--top-ports", "100", target],
                       capture_output=True, text=True, timeout=180)
    return r.stdout

def run_whatweb(url: str) -> str:
    """웹 기술 스택을 식별합니다."""
    if not _guard(url):
        return f"거부됨: {url} 은 허용 대상이 아닙니다."
    r = subprocess.run(["whatweb", "--color=never", url],
                       capture_output=True, text=True, timeout=120)
    return r.stdout

def run_gobuster(url: str) -> str:
    """숨겨진 디렉터리/페이지를 탐색합니다."""
    if not _guard(url):
        return f"거부됨: {url} 은 허용 대상이 아닙니다."
    r = subprocess.run(
        ["gobuster", "dir", "-u", url, "-q",
         "-w", "/usr/share/wordlists/dirb/common.txt", "-t", "30"],
        capture_output=True, text=True, timeout=300)
    return r.stdout
```

> **⚠️ 실행 전 워드리스트 경로를 먼저 확인하세요**  
> `run_gobuster`가 쓰는 `/usr/share/wordlists/dirb/common.txt`는 **Kali 기본 경로**입니다. 하지만 최소 설치본이나 다른 배포판에서는 이 파일이 없을 수 있고, 그러면 gobuster가 곧바로 오류를 내며 종료합니다. 미리 존재 여부를 확인하고, 없으면 설치하세요.
>
> ```bash
> ls /usr/share/wordlists/dirb/common.txt   # 존재 확인
> sudo apt install -y dirb                   # 없으면 설치
> # 대안: seclists 패키지의 워드리스트 사용 가능
> sudo apt install -y seclists
> ```
>
> 만약 워드리스트가 위 경로와 다른 곳에 있다면, `run_gobuster` 함수의 `-w` 뒤 경로를 **자신의 환경에 맞게** 바꿔 주세요. (예: `/usr/share/seclists/Discovery/Web-Content/common.txt`)
{: .prompt-warning }

---

## 3. 도구 세 개를 LLM에 등록

```python
# recon_agent.py (2/4) — 도구 설명서 3종
def _tool(name, desc, arg, argdesc):
    return {"type": "function", "function": {
        "name": name, "description": desc,
        "parameters": {"type": "object",
            "properties": {arg: {"type": "string", "description": argdesc}},
            "required": [arg]}}}

TOOLS = [
    _tool("run_nmap",    "IP의 열린 포트/서비스 스캔", "target", "대상 IP"),
    _tool("run_whatweb", "URL의 웹 기술 스택 식별",    "url",    "대상 URL"),
    _tool("run_gobuster","URL의 숨은 디렉터리 탐색",   "url",    "대상 URL"),
]

DISPATCH = {"run_nmap": run_nmap, "run_whatweb": run_whatweb, "run_gobuster": run_gobuster}
```

---

## 4. 여러 번 도구를 부르는 루프

정찰은 보통 도구를 **연달아 여러 번** 사용합니다. 그래서 08강의 1회성 루프를 **반복 루프**로 바꿉니다.

```python
# recon_agent.py (3/4) — 다중 도구 호출 루프
import ollama, json

MODEL = "qwen2.5:7b"
MAX_STEPS = 6   # 무한 루프 방지

def run_agent(goal: str):
    messages = [
        {"role": "system", "content":
         "너는 웹 정찰 에이전트다. 대상은 192.168.0.30(DVWA)뿐이다. "
         "필요한 도구를 순서대로 사용해 정찰하라: 먼저 포트, 다음 웹 기술, 다음 디렉터리. "
         "충분히 정보를 모으면 도구 호출을 멈추고 결과를 요약하라."},
        {"role": "user", "content": goal},
    ]

    for step in range(MAX_STEPS):
        resp = ollama.chat(model=MODEL, messages=messages, tools=TOOLS)
        msg = resp["message"]
        messages.append(msg)

        calls = msg.get("tool_calls") or []
        if not calls:                     # 더 부를 도구가 없으면 종료합니다.
            print("[*] 최종 요약:\n" + msg["content"])
            return messages               # 대화 기록을 돌려줍니다 (5절에서 JSON 구조화에 사용)

        for call in calls:
            name = call["function"]["name"]
            args = call["function"]["arguments"]
            arg_val = list(args.values())[0]
            print(f"[step {step+1}] 도구 실행: {name}({arg_val})")
            output = DISPATCH[name](arg_val)
            messages.append({"role": "tool", "name": name,
                             "content": output[:4000]})   # 너무 길면 잘라서 전달합니다.

    print("[!] 최대 단계에 도달했습니다.")
    return messages

if __name__ == "__main__":
    run_agent("192.168.0.30의 DVWA를 정찰해줘. (URL은 http://192.168.0.30/DVWA/)")
```

실행합니다.

```bash
python3 recon_agent.py
```

**예상 결과** (요지):

```text
[step 1] 도구 실행: run_nmap(192.168.0.30)
[step 2] 도구 실행: run_whatweb(http://192.168.0.30/DVWA/)
[step 3] 도구 실행: run_gobuster(http://192.168.0.30/DVWA/)
[*] 최종 요약:
- 포트: 22(ssh), 80(http, Apache 2.4.58), 3306(mysql)
- 웹 기술: PHP, Apache, jQuery 사용. DVWA 확인됨
- 발견 경로: /login.php, /setup.php, /vulnerabilities/, /config/
- 다음 단계: /vulnerabilities/ 하위의 SQLi, XSS 페이지를 점검 권장
```

> **AI가 도구 순서를 스스로 정했습니다.** 우리는 "포트 → 웹기술 → 디렉터리"라는 큰 방향만 system 프롬프트로 주었고, 실제 호출 순서와 인자는 LLM이 판단했습니다. 이것이 08강의 단일 호출과의 결정적 차이입니다.
{: .prompt-info }

---

## 5. 결과를 JSON으로 구조화

요약 텍스트는 사람이 읽기에는 좋지만, **다음 강의의 코드가 이어받기에는** 구조가 필요합니다.  
LLM에게 결과를 **정해진 JSON 형식**으로 내놓게 합니다.

```python
# recon_agent.py (4/4) — 정찰 결과를 JSON으로
def summarize_as_json(messages):
    messages.append({"role": "user", "content":
        '지금까지 정찰 결과를 아래 JSON 형식으로만 출력하라. 설명 금지.\n'
        '{"open_ports": [], "web_tech": [], "found_paths": [], "next_steps": []}'})
    resp = ollama.chat(model=MODEL, messages=messages, format="json")
    return resp["message"]["content"]
```

그리고 4절의 `if __name__` 블록을 아래처럼 바꿉니다. `run_agent`가 돌려준 대화 기록(`history`)을 그대로 `summarize_as_json`에 넘깁니다.

```python
# recon_agent.py — 실행 진입점 (4절의 if __name__ 블록을 이걸로 교체)
if __name__ == "__main__":
    history = run_agent("192.168.0.30의 DVWA를 정찰해줘. (URL은 http://192.168.0.30/DVWA/)")
    print("\n[*] 구조화 결과(JSON):")
    print(summarize_as_json(history))
```

다시 실행하면, 텍스트 요약에 이어 다음 JSON이 출력됩니다.

```bash
python3 recon_agent.py
```

**예상 결과:**

```json
{
  "open_ports": ["22/ssh", "80/http", "3306/mysql"],
  "web_tech": ["Apache/2.4.58", "PHP", "jQuery"],
  "found_paths": ["/login.php", "/setup.php", "/vulnerabilities/", "/config/"],
  "next_steps": ["SQL Injection 점검", "XSS 점검", "config 디렉터리 접근 확인"]
}
```

> Ollama의 `format="json"` 옵션은 모델이 **유효한 JSON만** 출력하도록 강제합니다. 구조화 출력의 핵심 도구입니다.
{: .prompt-tip }

---

## 6. 점검 인사이트 — 핑거프린팅과 디렉터리 탐색

> **점검 인사이트 — 정찰의 두 축**  
> **① 핑거프린팅(whatweb)**: 어떤 기술인지 알면 공격 벡터가 좁혀집니다. PHP 앱이면 LFI(Local File Inclusion: 로컬 파일 포함) / RFI(Remote File Inclusion: 원격 파일 포함) 및 파일 업로드 취약점을, WordPress면 플러그인 취약점을 우선 살펴봅니다.  
> **② 콘텐츠 탐색(gobuster)**: 링크로 노출되지 않은 페이지가 종종 가장 위험합니다. `/config/`, `/setup.php`, `/.git/`, 백업 파일(`.bak`) 같은 경로는 **개발자가 잊은 문**입니다. DVWA에서 `/setup.php`가 보이는 것처럼, 실제 서비스에서도 관리/설치 페이지 노출은 흔한 실수입니다.  
>
> 자동화의 이점: 사람은 이 과정을 지루해하지만, AI 에이전트는 **결과를 읽고 "다음에 어디를 팔지"까지 판단**해 줍니다.
{: .prompt-info }

이렇게 수집한 정찰 결과(JSON)는 02강 킬체인의 1단계 산출물에 해당하며, 이후 단계(무기화·전달·취약점 점검)의 입력으로 자연스럽게 이어집니다.

---

## 다음 강의 예고

다음 **10강**에서는 웹 취약점 스캐너 **`nikto`**를 통합하고, 스캔에서 나온 발견 항목을 **CWE**에 매핑해 AI가 위험을 선별·우선순위화하게 만듭니다. (CWE·CVE·CVSS는 03강 참조)

---

## 참고 자료

- WhatWeb — Next generation web scanner. <https://github.com/urbanadventurer/WhatWeb>
- Gobuster — Directory/File, DNS and VHost busting tool. <https://github.com/OJ/gobuster>
- MITRE ATT&CK — Reconnaissance (TA0043). <https://attack.mitre.org/tactics/TA0043/>
- Ollama — Structured outputs (`format` 옵션). <https://github.com/ollama/ollama/blob/main/docs/api.md>
- OWASP Web Security Testing Guide — Information Gathering. <https://owasp.org/www-project-web-security-testing-guide/>
