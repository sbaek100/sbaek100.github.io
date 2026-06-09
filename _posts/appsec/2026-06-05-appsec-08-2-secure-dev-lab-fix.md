---
title: "[애플리케이션 보안] 08-2. 실습 — 취약 코드를 안전 코드로 고치기"
date: 2026-06-05 20:16:00 +0900
categories:
  - 강의
  - 애플리케이션보안
  - 개발보안
tags:
  - 시큐어코딩
  - 입력검증
  - 명령어삽입
  - Python
  - 실습
math: false
mermaid: true
---

> 이 글은 **실습** 입니다. 개념은 앞 글(**08-1**)의 3장(시큐어코딩)을 먼저 읽으세요.  
> 이 실습은 **Kali(192.168.0.10)** 에서 진행합니다. (파이썬 기본 설치됨)
{: .prompt-info }

## 0. 목표

흔한 취약점 두 가지(**명령어 삽입**, **비밀번호 하드코딩**)가 있는 파이썬 코드를 만들고, **시큐어코딩 원칙으로 고쳐** 봅니다. (이 코드는 08-3 정적분석 실습에서 그대로 점검합니다.)

---

## 1. (Kali) 취약한 코드 만들기

작업 폴더를 만들고 취약한 스크립트 `vuln.py` 를 작성합니다.

```bash
# === Kali에서 실행 ===
mkdir -p ~/secure-lab && cd ~/secure-lab
nano vuln.py
```

```python
# vuln.py  --- 취약한 예시
import os

# (취약점 1) 비밀번호를 코드에 그대로 박음(하드코딩)
PASSWORD = "admin1234"

host = input("핑을 보낼 호스트: ")

# (취약점 2) 사용자 입력을 셸 명령에 그대로 붙임 → 명령어 삽입
os.system("ping -c 1 " + host)
```

### 1.1 왜 위험한지 직접 확인

```bash
# === Kali에서 실행 ===
python3 vuln.py
# 호스트에 다음을 입력해 본다:
# 127.0.0.1; id
```

입력 `127.0.0.1; id` 를 넣으면 ping 뒤에 **`id` 명령까지 실행** 됩니다. → 입력값이 **명령의 일부** 가 되어 버린 것(08장 이론의 입력 검증 실패).

> 🔴 이것이 **명령어 삽입(Command Injection)** 입니다. `os.system` + 문자열 결합이 원인입니다.
{: .prompt-warning }

---

## 2. (Kali) 안전한 코드로 고치기

같은 기능을, 시큐어코딩 원칙을 적용해 `safe.py` 로 다시 작성합니다.

```bash
# === Kali에서 실행 ===
nano safe.py
```

```python
# safe.py  --- 안전한 예시
import subprocess
import ipaddress
import os

# (수정 1) 비밀번호는 환경변수에서 읽음 (코드에 없음)
PASSWORD = os.environ.get("APP_PASSWORD")

host = input("핑을 보낼 호스트: ")

# (수정 2) 입력 검증: 올바른 IP 형식만 허용(화이트리스트)
try:
    ipaddress.ip_address(host)
except ValueError:
    print("유효한 IP 주소가 아닙니다.")
    raise SystemExit(1)

# (수정 3) 셸을 거치지 않고 리스트 인자로 실행 → 명령어 삽입 불가
subprocess.run(["ping", "-c", "1", host], check=True)
```

### 2.1 고친 코드로 확인

```bash
# === Kali에서 실행 ===
python3 safe.py
# 127.0.0.1; id   → "유효한 IP 주소가 아닙니다" (차단!)
# 127.0.0.1       → 정상 ping
```

이제 `127.0.0.1; id` 같은 입력은 **IP 형식이 아니라서 거부** 되고, 설령 통과해도 `subprocess` 가 셸을 안 쓰므로 **명령이 덧붙지 않습니다.**

---

## 3. 무엇을 고쳤나 — 시큐어코딩 체크

| 취약점 | 원인 | 적용한 시큐어코딩 |
|---|---|---|
| 명령어 삽입 | `os.system("... " + host)` | **입력 검증**(IP만) + **안전한 API**(`subprocess` 리스트 인자) |
| 비밀번호 하드코딩 | 코드에 `"admin1234"` | **환경변수로 분리**(`os.environ`) |

> 핵심 교훈: **① 입력을 믿지 말고 검증한다, ② 위험한 함수(`os.system`) 대신 안전한 API를 쓴다, ③ 비밀은 코드에 넣지 않는다.**
{: .prompt-tip }

---

## 4. 체크리스트

- [ ] `vuln.py` 에서 `127.0.0.1; id` 입력 시 `id` 가 실행되는 것(취약) 확인
- [ ] `safe.py` 에서 같은 입력이 **거부** 되는 것 확인
- [ ] `os.system` → `subprocess.run([...])` 변경의 이유 설명 가능
- [ ] 하드코딩된 비밀번호를 환경변수로 분리

> 두 파일(`vuln.py`, `safe.py`)은 다음 실습(08-3)에서 **정적분석 도구로 자동 점검** 합니다. 지우지 마세요.
{: .prompt-tip }

---

## 5. 다음 글

직접 눈으로 취약점을 고쳤습니다. 다음 글 **08-3. 정적분석 도구로 자동 점검** 에서 `semgrep` 으로 `vuln.py` 의 취약점을 **자동으로 찾아내고**, 고친 `safe.py` 와 비교합니다.
