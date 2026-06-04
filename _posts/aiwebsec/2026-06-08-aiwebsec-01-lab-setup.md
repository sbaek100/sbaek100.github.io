---
title: "[AI 보안 자동화 Lab] 01. 랩 준비 & 파이썬 작업 환경 만들기"
date: 2026-06-08 10:00:00 +0900
categories:
  - 웹보안
  - AI자동화
tags:
  - Kali-Linux
  - Python
  - DVWA
  - 실습환경
  - 자동화
pin: false
math: false
mermaid: true
---

# 랩 준비 & 파이썬 작업 환경 만들기

코드를 짜기 전에, 집을 짓기 위한 **터를 다지는** 단계다.  
이번 강의에서는 두 가지만 확실히 한다.

1. Kali(192.168.0.10)와 Ubuntu DVWA(192.168.0.30)가 **서로 통신**되는지 확인
2. 앞으로 코드를 작성할 **Python 작업 폴더와 가상환경** 준비

> 이 강의는 명령어를 그대로 따라 치기만 하면 된다. 결과가 예시와 같은지 한 줄씩 확인하며 진행하자.
{: .prompt-tip }

---

## 1. 두 머신이 살아 있는지 확인

먼저 **Kali Linux**를 켜고 터미널을 연다. 그리고 대상 서버가 응답하는지 본다.

```bash
# Kali(192.168.0.10)에서 실행
ping -c 3 192.168.0.30
```

다음과 비슷하게 `0% packet loss`가 나오면 정상이다.

```text
64 bytes from 192.168.0.30: icmp_seq=1 ttl=64 time=0.42 ms
...
3 packets transmitted, 3 received, 0% packet loss
```

> **응답이 없다면?**  
> VirtualBox에서 두 가상머신의 네트워크가 같은 **호스트 전용(Host-Only)** 또는 **내부 네트워크**로 설정됐는지 확인한다.  
> (이전 [Web Security Lab] 00강에서 구성한 그대로여야 한다.)
{: .prompt-warning }

---

## 2. DVWA가 열려 있는지 확인

대상 서버의 웹이 살아 있는지 명령줄에서 빠르게 확인한다.

```bash
# Kali에서 실행 — DVWA 로그인 페이지의 응답 헤더만 확인
curl -I http://192.168.0.30/DVWA/login.php
```

`HTTP/1.1 200 OK` 또는 `302 Found`(로그인 리다이렉트)가 보이면 웹 서버가 정상 동작 중이다.

```text
HTTP/1.1 302 Found
Server: Apache/2.4.58 (Ubuntu)
...
```

> 여기서 보이는 `Server: Apache/2.4.58 (Ubuntu)` 같은 정보가 나중에 AI가 **정찰 단계에서 읽어 들일** 바로 그 정보다. 지금은 우리가 직접 보지만, 05강부터는 AI가 이걸 읽고 판단한다.
{: .prompt-info }

---

## 3. 파이썬 준비 확인

Kali에는 보통 Python 3가 이미 설치돼 있다. 버전을 확인하자.

```bash
python3 --version
```

```text
Python 3.12.x
```

`3.10` 이상이면 충분하다. 만약 없다면 설치한다.

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
```

---

## 4. 작업 폴더 만들기

이 시리즈에서 만들 모든 코드를 담을 폴더를 하나 만든다.

```bash
# 홈 디렉터리 아래에 프로젝트 폴더 생성
mkdir -p ~/ai-pentest-lab
cd ~/ai-pentest-lab
```

앞으로 모든 명령은 이 `~/ai-pentest-lab` 폴더 안에서 실행한다고 가정한다.

---

## 5. 가상환경(venv) 만들기 — 왜 필요한가

파이썬 라이브러리를 시스템 전체에 막 설치하면, 나중에 충돌이 생기기 쉽다.  
**가상환경(virtual environment)**은 이 프로젝트만의 깨끗한 라이브러리 공간을 만들어 준다.  
작업마다 폴더를 따로 나눠 서로 간섭하지 않게 하는 것과 같은 개념이다.

```bash
# 'venv'라는 이름의 가상환경 생성
python3 -m venv venv

# 가상환경 활성화 (활성화되면 프롬프트 앞에 (venv)가 붙는다)
source venv/bin/activate
```

활성화에 성공하면 프롬프트가 이렇게 바뀐다.

```text
(venv) ┌──(kali㉿kali)-[~/ai-pentest-lab]
└─$
```

> **앞으로 코딩할 때마다** 이 폴더에서 `source venv/bin/activate`를 먼저 실행해 `(venv)`가 보이는 상태로 작업한다.  
> 가상환경을 빠져나오려면 `deactivate`를 입력한다.
{: .prompt-tip }

---

## 6. 첫 라이브러리 설치 — 동작 확인용

다음 강의부터 본격적으로 쓸 라이브러리 중, 가장 기본인 `requests`(HTTP 요청)를 미리 설치해 본다.  
설치가 잘 되는지 확인하는 목적이다.

```bash
# (venv) 상태에서 실행
pip install requests
```

설치가 끝나면 잘 깔렸는지 한 줄로 확인한다.

```bash
python3 -c "import requests; print('requests OK:', requests.__version__)"
```

```text
requests OK: 2.32.x
```

---

## 7. 파이썬으로 DVWA에 인사하기

마지막으로, 파이썬 코드가 실제로 대상 서버와 통신하는지 확인하는 아주 작은 스크립트를 작성한다.  
편집기로 `check_target.py` 파일을 만든다.

```bash
nano check_target.py
```

아래 내용을 입력하고 저장한다(`Ctrl+O` → `Enter` → `Ctrl+X`).

```python
# check_target.py — 대상 서버가 살아 있는지 파이썬으로 확인
import requests

TARGET = "http://192.168.0.30/DVWA/login.php"

try:
    resp = requests.get(TARGET, timeout=5)
    print(f"[+] 응답 코드: {resp.status_code}")
    print(f"[+] 서버 헤더: {resp.headers.get('Server', '(없음)')}")
    print(f"[+] 응답 크기: {len(resp.text)} bytes")
except requests.exceptions.RequestException as e:
    print(f"[-] 접속 실패: {e}")
```

실행한다.

```bash
python3 check_target.py
```

다음과 비슷하게 나오면 **이번 강의는 성공**이다.

```text
[+] 응답 코드: 200
[+] 서버 헤더: Apache/2.4.58 (Ubuntu)
[+] 응답 크기: 1523 bytes
```

> 방금 만든 이 작은 스크립트가 바로 우리 에이전트의 **첫 번째 부품**이다.  
> "파이썬으로 대상에게 요청을 보내고 응답을 받는다" — 이것이 모든 자동화의 출발점이다.
{: .prompt-info }

---

## 8. 다음 강의 예고

다음 **02강**에서는 로컬 LLM 실행기 **Ollama**를 Kali에 설치하고, 가장 작은 모델로 동작을 검증한다.
