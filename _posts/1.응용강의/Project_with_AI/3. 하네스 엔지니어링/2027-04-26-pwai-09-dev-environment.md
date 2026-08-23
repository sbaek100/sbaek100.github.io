---
title: 9강. 개발 환경 구축 - WSL과 Claude Code
date: 2027-04-26 09:00:00 +0900
categories:
  - 1.응용강의
  - AI와 함께하는 프로젝트
tags:
  - WSL
  - ClaudeCode
  - 바이브코딩
  - 개발환경
  - 비전공자
pin:
mermaid: false
---

> **학습 목표**
> 1. 하네스의 개념을 한 문장으로 설명할 수 있다.
> 2. WSL(리눅스)을 설치하고 기본 명령을 사용할 수 있다.
> 3. Claude Code를 설치하고 AI와 첫 대화를 수행할 수 있다.
{: .prompt-info }

## 1. 하네스란 무엇인가

AI 코딩 도구에 "만들어 달라"고만 지시하면 원하는 결과가 나오지 않는다.

| 단순히 지시하면 | 원인 |
|---|---|
| 라이브러리를 임의로 설치한다 | 프로젝트의 규칙을 모른다 |
| 파일 구조가 실행마다 달라진다 | 이전 작업의 기억이 없다 |
| "이 기능도 넣을까요?"가 끝없이 이어진다 | 어디가 끝인지 모른다 |

> **하네스(Harness)**란 AI가 **프로젝트의 규칙을 따르도록 붙잡아 주는 구조**를 말한다. AI를 통제하여 묶어 두는 것이 아니라, 힘이 엉뚱한 방향으로 가지 않도록 방향을 잡아 주는 안전벨트에 해당한다.

사실 Claude Code를 실행하는 순간 이미 기본적인 하네스(위험 명령 차단, 권한 확인 등)가 작동하고 있다. 본 챕터에서 하는 일은 그 위에 **"내 프로젝트를 아는 층" 하나를 더 올리는 것**이다. 이번 강의에서는 하네스를 만들기 전 단계로, 필요한 도구(리눅스 + AI)를 설치한다.

**준비물**:

| 항목 | 비고 |
|---|---|
| Windows 10(1809 이상) 또는 Windows 11 | |
| Claude 유료 요금제 | Pro·Max·Team 중 하나. **무료 요금제로는 Claude Code를 사용할 수 없다** |
| 인터넷 연결 | |

---

## 2. WSL 설치

> **WSL(Windows Subsystem for Linux)**이란 윈도우 안에서 리눅스를 실행하는 기능이다. 개발 도구 대부분이 리눅스를 전제로 만들어져 있으므로 이를 사용한다.

### 2.1 윈도우 버전 확인

`Win + R` → `winver` 입력. Windows 10은 1809 이상이어야 한다.

### 2.2 관리자 PowerShell에서 설치

시작 버튼 **오른쪽 클릭** → "터미널(관리자)" 선택 → 권한 확인 창에서 "예". 반드시 관리자 권한으로 열어야 하며, 일반 권한으로 열면 설치가 실패한다.

```powershell
wsl --install -d Ubuntu-24.04
```

몇 분 소요된다. 완료되면 **컴퓨터를 재시작**한다. `Ubuntu-24.04`를 찾을 수 없다는 오류가 나오면 `wsl --install`만 입력한다.

### 2.3 재시작 후 계정 생성

Ubuntu 창이 자동으로 열리고 계정 생성을 요구한다.

| 항목 | 규칙 |
|---|---|
| 사용자 이름 | 영문 소문자로 짧게 (예: `student`) |
| 비밀번호 | **입력해도 화면에 표시되지 않는다. 정상이다.** 입력 후 Enter |

비밀번호는 이후 프로그램 설치(`sudo`) 시 필요하므로 반드시 기억한다. 리눅스는 보안상 비밀번호의 길이조차 표시하지 않는다.

### 2.4 설치 확인

PowerShell에서 다음을 실행한다.

```powershell
wsl --list --verbose
```

```text
  NAME              STATE           VERSION
* Ubuntu            Stopped         2
```

`VERSION`이 `2`이면 정상이다. 한글 윈도우에서 출력이 `0���`처럼 깨져 보이면 `$env:WSL_UTF8 = "1"`을 실행한다 — 표시만 깨진 것이며 설치 오류가 아니다.

---

## 3. 리눅스 기본 조작

### 3.1 프롬프트 읽기

시작 메뉴에서 Ubuntu를 실행하면 다음 형태의 줄이 나타난다.

```text
student@DESKTOP-ABC:~$
```

| 부분 | 의미 |
|---|---|
| `student` | 현재 계정 |
| `~` | 현재 위치 (홈 디렉터리) |
| `$` | 명령 입력 대기 표시 |

명령은 `$` 뒤에 입력한다. 인터넷에서 명령을 복사할 때 맨 앞의 `$` 기호는 제외한다.

### 3.2 최초 업데이트

```bash
sudo apt update && sudo apt upgrade -y
```

`sudo`는 "관리자 권한으로 실행"을 뜻하며, 계정 생성 시 정한 비밀번호를 묻는다.

### 3.3 필수 명령 다섯 가지

| 명령 | 기능 |
|---|---|
| `pwd` | 현재 위치 확인 |
| `ls` | 현재 위치의 파일 목록 |
| `cd` | 위치 이동 |
| `mkdir` | 폴더 생성 |
| `cat` | 파일 내용 보기 |

이 다섯 가지 외에는 필요할 때 AI에게 물으면 된다. 그것이 본 과정의 방식이다. 추가로 두 가지 요령을 익혀 둔다 — **`Tab` 키**는 이름을 자동 완성하고, **`↑` 키**는 직전 명령을 다시 불러온다.

> 작업은 리눅스 영역(`~/` 아래)에서 수행한다. 윈도우 영역(`/mnt/c`)은 파일 접근이 매우 느리다.
{: .prompt-warning }

---

## 4. Claude Code 설치와 첫 대화

### 4.1 설치

Ubuntu 터미널에서 실행한다.

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

설치 확인:

```bash
claude --version
```

```text
2.1.211 (Claude Code)
```

버전 번호는 달라도 무방하며 `(Claude Code)`가 표시되면 정상이다. `command not found` 오류가 나오면 터미널을 닫았다가 다시 연다.

### 4.2 로그인

```bash
claude
```

최초 실행 시 로그인을 요구한다. 브라우저에서 인증하면 되고, 한 번 인증하면 이후에는 묻지 않는다. 브라우저가 자동으로 열리지 않으면 화면에 표시된 주소를 복사하여 브라우저에 직접 붙여 넣는다.

### 4.3 화면 읽기

```text
 ✻ Welcome to Claude Code!
   /help for help
   cwd: /home/student
```

`cwd`는 현재 작업 폴더이며, **AI는 이 폴더 안에서 작업한다**. 실행할 때마다 확인하는 습관을 들인다.

### 4.4 첫 대화

명령어가 아니라 자연어로 지시한다. 이것이 **바이브 코딩**이다.

```text
안녕이라고 출력하는 파이썬 파일을 hello.py 로 만들어 줘
```

AI는 파일을 만들기 전에 승인을 요청한다.

```text
  Do you want to create hello.py?
  ❯ 1. Yes
    2. Yes, and don't ask again this session
    3. No, and tell Claude what to do differently
```

처음에는 1번(Yes)을 선택하면서 AI가 무엇을 하는지 관찰한다.

### 4.5 기본 조작 네 가지

| 조작 | 기능 |
|---|---|
| `/clear` | 주제가 바뀔 때 대화 내용 초기화 |
| `/exit` | 종료 |
| `Esc` | AI가 엉뚱한 방향으로 갈 때 **즉시 중단** |
| `Shift+Tab` | 권한 모드 전환 (묻기 / 자동 진행 / 계획만) |

실습이 끝나면 `/exit`로 나온 뒤 `rm -f hello.py`로 파일을 정리한다. AI가 만든 결과물이 마음에 들지 않으면 삭제하면 그만이다.

---

## 5. 설치 점검

다음 네 가지가 확인되면 이번 강의의 목표를 달성한 것이다.

```bash
wsl --list --verbose      # (PowerShell) VERSION 이 2
```

```bash
claude --version          # (Ubuntu) (Claude Code) 표시
```

```bash
python3 --version         # Python 3.12.x — 기본 설치되어 있다
```

```text
방금 만든 파일 실행해 줘    # (claude 안에서) 대화가 성립한다
```

---

## 자주 발생하는 문제

| 증상 | 해결 |
|---|---|
| `wsl --install` 인식 안 됨 | 윈도우 업데이트 수행 |
| `WslRegisterDistribution failed: 0x80370102` | BIOS에서 가상화(VT-x/SVM) 활성화 |
| 비밀번호가 표시되지 않음 | 원래 표시되지 않는다. 입력 후 Enter |
| 한글 깨짐(`���`) | PowerShell에서 `$env:WSL_UTF8 = "1"` |
| `claude: command not found` | 터미널을 닫았다가 다시 열기 |
| 로그인 실패 | 유료 요금제(Pro 이상) 필요 |
| 비정상적으로 느리거나 네트워크 불통 | PowerShell에서 `wsl --shutdown` 후 재시작 |

---

## (선택) 편집기 연동

터미널만으로 과정 전체를 진행할 수 있으나, VS Code를 연동하면 AI가 수정한 내용이 색상으로 표시되어 편리하다.

1. [code.visualstudio.com](https://code.visualstudio.com/)에서 VS Code를 윈도우에 설치
2. VS Code에서 `Ctrl+Shift+X` → `WSL` 확장 설치 (Microsoft 제공)
3. Ubuntu 터미널에서 작업 폴더로 이동 후 `code .`
4. 좌측 하단에 `>< WSL: Ubuntu` 표시가 보이면 성공
5. `Ctrl+Shift+X` → `Claude Code` 확장 설치 (Anthropic 제공)

---

## 다음 강의

10강에서는 오픈소스 **하네스 프레임워크를 가져와, 기획 챕터에서 완성한 기획서와 화면 설계를 AI에게 전달하여 pcapeek을 실제로 완성**한다. 코드는 전부 AI가 작성하며, 사용자는 지시문을 붙여 넣고 결과를 확인한다.

## 출처

| 내용 | 근거 |
|---|---|
| WSL 설치·확인 출력 | Windows 11 + WSL 2.6.3.0 실측 |
| 한글 깨짐 해결(`WSL_UTF8=1`) | 실측 |
| Claude Code 설치·요금제 | [공식 설치 문서](https://code.claude.com/docs/en/setup) |
| "이미 하네스를 쓰고 있다" 개념 | [실밸개발자, 하네스 세팅 영상](https://www.youtube.com/watch?v=AQOvNx87Urs) |
