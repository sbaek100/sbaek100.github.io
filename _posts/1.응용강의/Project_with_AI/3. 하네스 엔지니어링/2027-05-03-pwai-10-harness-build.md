---
title: 10강. 하네스로 프로젝트 완성 - pcapeek 구현
date: 2027-05-03 09:00:00 +0900
categories:
  - 1.응용강의
  - AI와 함께하는 프로젝트
tags:
  - 하네스
  - ClaudeCode
  - 바이브코딩
  - 프롬프트
  - pcap
pin:
mermaid: false
---

> **학습 목표**
> 1. 오픈소스 하네스 프레임워크를 클론하여 자신의 프로젝트로 초기화할 수 있다.
> 2. AI에게 하네스를 프로젝트에 맞게 수정시키고, 그 결과를 직접 검증할 수 있다.
> 3. 기획 문서를 채우고 자동 실행으로 pcapeek 웹 리포트를 완성할 수 있다.
{: .prompt-info }

## 0. 이번 강의의 흐름

```text
① 하네스 클론                    (명령 한 줄)
② AI가 프로젝트에 맞게 수정        (지시문 하나)
③ 훅이 실제로 막는지 직접 확인      (명령 한 줄)
④ 만들 것을 문서로 확정            (지시문 하나)
⑤ 자동 실행                      (명령 한 줄)
⑥ 완성물 검증                    (실행·교차 확인)
```

사용자가 하는 일은 지시문을 붙여 넣고 결과를 확인하는 것이며, **코드는 전부 AI가 작성한다**. 완성 목표물은 기획 챕터에서 설계한 pcapeek — `.pcap` 통신 기록을 읽어 `report.html` 웹 리포트를 생성하고, 암호화되지 않은 통신(HTTP·Telnet)을 경고하는 프로그램이다.

---

## 1. 하네스 클론

Ubuntu 터미널에서 실행한다.

```bash
cd ~
git clone https://github.com/jha0313/harness_framework.git pcapeek
cd pcapeek
rm -rf .git && git init
```

| 줄 | 기능 |
|---|---|
| 2행 | 오픈소스 하네스를 `pcapeek`이라는 이름으로 내려받는다 |
| 4행 | 원저장소의 이력을 지우고 새 프로젝트로 시작한다 |

내려받은 내용을 확인한다.

```bash
ls docs/ .claude/
```

```text
docs/:      ADR.md  ARCHITECTURE.md  PRD.md  UI_GUIDE.md
.claude/:   commands  settings.json
```

`docs/`의 마크다운 문서를 채우는 것이 이 방식의 핵심이다. 원저자는 이를 "클론 받아서 마크다운만 채우면 바로 사용할 수 있다. 비개발자도 가능하다"라고 설명한다.

작업 전에 git 사용자 정보를 설정하고 초기 상태를 커밋한다. 커밋해 두면 AI가 파일을 잘못 수정해도 언제든 되돌릴 수 있다.

```bash
git config user.name "myname"
git config user.email "me@example.com"
git add -A && git commit -m "start"
```

---

## 2. AI에게 하네스 수정시키기

### 2.1 클론한 그대로는 사용할 수 없다

원본 하네스는 웹(npm/Next.js) 프로젝트를 전제로 하며, 자동 검증 장치(Hook)가 구식 규격이라 그대로 두면 아무것도 차단하지 못한다. 직접 고치지 말고 AI에게 수정시킨다.

### 2.2 수정 지시문

`claude`를 실행하고(`cwd`가 `.../pcapeek`인지 확인), 다음 지시문 전체를 붙여 넣는다.

````text
이 폴더는 harness_framework 를 클론한 거야. 내 프로젝트에 맞게 고쳐 줘.

## 내 프로젝트
- 이름: pcapeek
- 하는 일: .pcap 파일을 읽어 웹 리포트(report.html)를 만든다
- 언어: Python 3.10 이상, 표준 라이브러리만 (pip install 금지)
- 시험 도구: pytest
- 실행: python -m pcapeek.cli <파일>  (report.html 을 만들고 브라우저로 연다)

## 고칠 것
1. npm/Next.js 전제를 전부 파이썬(pytest)으로 바꿔 줘.
   CLAUDE.md, .claude/commands, .claude/settings.json 안의 npm 명령들.
2. docs/UI_GUIDE.md 는 docs/OUTPUT_GUIDE.md 로 바꾸고,
   웹 리포트 출력 규칙(외부 CDN·폰트 금지, 색은 아이콘과 함께,
   숫자에 단위, 단일 HTML 파일)으로 내용을 바꿔 줘.
3. docs/ 와 CLAUDE.md 의 빈칸 { } 는 그대로 둬. 내가 곧 채운다.

## 자동 검증(Hook)을 현행 규격으로 다시 만들어 줘
scripts/hooks/ 아래 셸 스크립트 네 개를 만들고 .claude/settings.json 에 등록해 줘.
- dangerous-cmd-guard.sh  위험 명령(rm -rf 등) 차단   PreToolUse(Bash)
- tdd-guard.sh            테스트 없으면 차단          PreToolUse(Write|Edit)
- run-tests.sh            시험 자동 실행              Stop
- circuit-breaker.sh      반복 실패 경고             PostToolUseFailure

**반드시 지킬 것 (안 지키면 훅이 아무것도 안 막아)**
- 입력은 표준 입력의 JSON 이다. 환경 변수 $CLAUDE_TOOL_INPUT 을 쓰지 마라.
  명령은 .tool_input.command, 파일은 .tool_input.file_path 에 있다.
- 차단은 exit 2 다. exit 1 은 안 막힌다.
- 차단 사유는 표준 오류(>&2)로 써라.
- JSON 파싱은 python3 로 하고 try/except 로 감싸라 (jq 쓰지 마).
- 경로는 ${CLAUDE_PROJECT_DIR} 를 써라. 만든 스크립트에 chmod +x 해라.

**훅별 주의**
- dangerous-cmd-guard: rm 은 -r 이나 -f 붙은 것만 막아라.
  rm sample.pcap 같은 파일 하나 삭제까지 막으면 안 된다.
- tdd-guard: src/pcapeek/*.py 만 검사, __init__.py 와 make_sample.py 는 예외.
- run-tests: 아직 pytest 도 시험 파일도 없는 첫 단계에서 막히면 안 된다.
  pytest 미설치나 "수집된 시험 없음(종료 코드 5)" 이면 통과(0),
  시험이 실제로 실패했을 때만(종료 코드 1) 차단(2).

## 실행 엔진
scripts/execute.py 는 npm 전제라 지우고, scripts/run_phases.py 로 새로 써 줘.
- phases/<task>/index.json 에서 pending 인 다음 step 을 찾아 순서대로 실행
- claude -p --permission-mode acceptEdits 로 헤드리스 실행, 끝나면 git commit
- --dry-run 은 프롬프트만 출력

## 뼈대
phases/index.json, phases/0-mvp/index.json (step 은 비워 둬),
src/pcapeek/__init__.py, tests/__init__.py

## 다 고친 뒤
각 훅에 시험용 JSON 을 표준 입력으로 넣어서
막을 것은 막고(exit 2) 통과할 것은 통과하는지(exit 0) 확인해서 표로 보여 줘.
````

> 이 지시문에서 가장 중요한 부분은 「반드시 지킬 것」 다섯 줄이다. 이것이 없으면 AI가 인터넷에 퍼진 구식 방식대로 훅을 만들고, 그 결과 **자동 검증 장치가 있는데 아무것도 차단하지 못하는** 상태가 된다. 좋은 지시문은 "무엇을 만들라"보다 **"어떤 함정을 피하라"**를 담는다.
{: .prompt-warning }

수정이 끝나면 `scripts/hooks/`에 `.sh` 파일 네 개가 생성되었는지, 마지막에 훅 시험 결과 표가 출력되었는지 확인한다.

---

## 3. 훅이 실제로 차단하는지 직접 확인

AI가 "완료되었습니다"라고 보고하더라도 한 번은 직접 확인해야 한다. 훅은 동작하지 않아도 오류를 내지 않고 조용히 무력화되기 때문이다. `/exit`로 나온 뒤 실행한다.

```bash
echo '{"tool_input":{"command":"rm -rf test/"}}' \
  | bash scripts/hooks/dangerous-cmd-guard.sh; echo "종료 코드: $?"
```

```text
차단: 되돌릴 수 없는 명령입니다 — 'rm -rf test/'
종료 코드: 2
```

**종료 코드 2가 차단, 0이 통과이다.** `1`이 나오면 구식 방식이므로, AI를 다시 실행하여 "훅이 exit 1을 내는데 차단은 exit 2여야 한다. 표준 입력 JSON 방식으로 고쳐라"라고 지시한다. 안전한 명령이 통과하는지도 확인한다.

```bash
echo '{"tool_input":{"command":"pytest -q"}}' \
  | bash scripts/hooks/dangerous-cmd-guard.sh; echo "종료 코드: $?"
```

`0`이 나와야 한다. 확인이 끝나면 커밋한다 — `git add -A && git commit -m "harness ready"`.

---

## 4. 만들 것을 문서로 확정

여기가 가장 중요한 단계이다. **하네스가 아무리 좋아도 문서가 얕으면 결과도 얕다.** 기획 챕터에서 완성한 기획서(문제 진술문·Must·Won't·Use Case)와 화면 설계(와이어프레임·하지 말 것)가 이 지시문의 재료가 된다. 다시 `claude`를 실행하고 붙여 넣는다.

````text
docs/ 네 문서와 CLAUDE.md 의 { } 빈칸을 아래 내용으로 채워 줘.

## 프로젝트
pcapeek — .pcap 파일을 읽어 무슨 통신이 오갔고 위험한 것은 없는지
웹 리포트(report.html)로 알려 주는 도구. 쓰는 사람은 네트워크를 처음 배우는 1학년.

## 핵심 기능
1. 패킷 수·크기·시간을 요약한다
2. 프로토콜(TCP/UDP/ICMP) 비율을 막대로 보여 준다
3. 가장 많이 오간 상대 IP와 포트를 상위 5개씩
4. 암호화되지 않은 프로토콜(HTTP·Telnet·FTP 등)을 찾아 경고한다
5. 권한 없이 연습하도록 연습용 pcap 을 만들어 주는 명령도 함께
6. 실행하면 report.html 을 만들고, webbrowser 모듈로 기본 브라우저에서 자동으로 연다

## 안 만들 것 (이유까지 적어 줘)
- IPv6 해석 — 건너뛰고 개수만 센다
- 실시간 캡처 — 저장된 파일만. 캡처는 tcpdump 가 한다
- pcapng 형식 — 옛 pcap 만
- 패킷 내용(페이로드) 표시 — 개인정보가 찍힌다
- 실제 웹서버 — report.html 파일 하나를 만드는 것까지만.
  서버를 띄우고 여러 사용자가 접속하는 것은 이번 범위 밖
- 외부 CSS 프레임워크(Bootstrap 등)·CDN·웹폰트 — 표준 라이브러리만 쓰고,
  인터넷 연결 없이도 report.html 이 그대로 열려야 한다

## 구조 (ARCHITECTURE.md)
reader.py → decode.py → stats.py → report.py → cli.py 한 방향으로만 흐른다.
- reader   파일 → 패킷 목록 (바이트만 다룬다)
- decode   패킷 → 출발지·도착지·포트
- stats    여러 패킷 → 요약 통계
- report   요약 → HTML 문자열 생성 (외부 자원 없이 <style> 안에 CSS 전부 포함)
- cli      명령행 진입점. report.py 를 호출해 report.html 로 저장하고
           webbrowser 로 연다. print 는 여기서만
- make_sample.py 는 연습용 pcap 생성기

## 결정 (ADR.md, 트레이드오프까지)
- Python 3.10+, 표준 라이브러리만 (설치가 필요하면 실습실에서 막힌다)
- 이상한 패킷 하나 때문에 멈추지 않는다 — 건너뛰고 'report.html'에
  '건너뜀 N개'를 찍는다
- 연습용 생성기는 누가 언제 돌려도 같은 파일이 나오게
- 평문 판단은 포트 번호로만 (21·23·25·80·110·143)
- 웹 서버를 띄우지 않고, 실행할 때마다 report.html 을 새로 만든다 —
  포트·서버 관리 개념 없이 파일 하나로 결과를 볼 수 있다.
  대신 실시간 갱신은 포기한다(다시 실행해야 최신 결과가 보인다)

## 출력 (OUTPUT_GUIDE.md)
한 페이지 안에. 숫자에 단위. 위험은 빨간 배경 + ⚠ 아이콘을 함께 써서
색맹인 사용자도 알아볼 수 있게. 외부 CDN·폰트·이미지를 불러오지 않는다
(인터넷 없이도 열려야 한다). <style> 로 CSS를 파일 안에 전부 넣는다.
화면 배치는 위에서 아래로: 제목 → 요약 카드 → 프로토콜 막대그래프 →
상대·포트 표 → 경고 카드 순서. (화면 설계에서 그린 와이어프레임 그대로 '형식' 항목에 적어 줘)

## 완료 기준
- python -m pcapeek.make_sample sample.pcap 이 연습 파일을 만든다
- python -m pcapeek.cli sample.pcap 이 report.html 을 만들고
  기본 브라우저로 연다 (자동으로 안 열려도 종료 코드는 0)
- report.html 을 인터넷 연결 없이 열어도 깨지지 않는다
- 잘못된 파일에 죽지 않고 이유를 터미널에 찍고 종료 코드 1
- pytest 가 전부 통과한다

## 이름 규약
파일·폴더·함수·변수 이름은 영문 소문자로. 주석과 화면 출력만 우리말로.

다 채운 뒤 { } 가 하나도 안 남았는지 확인해서 알려 줘.
````

"안 만들 것"에 이유까지 적는 것이 핵심이다. 이유가 없으면 AI가 개발 도중 "이 기능도 추가할까요?"를 반복한다. 완료 후 다음으로 확인한다.

```bash
grep -c '{[^}]*}' docs/*.md CLAUDE.md
```

전부 `0`이면 정상이다. `head -20 docs/PRD.md`로 내용도 직접 읽어 본다 — 지시한 내용이 들어갔는지 확인하는 일은 AI가 대신할 수 없다.

---

## 5. 자동 실행

### 5.1 작업 지시서 생성

`claude` 안에서 `/harness`를 입력하면, AI가 `docs/`를 읽고 구현 계획을 제안한다.

```text
docs/ 를 읽었습니다. 5단계로 나누겠습니다.
  step 0  뼈대       step 1  reader.py
  step 2  decode.py  step 3  stats.py
  step 4  report.py + cli.py
이대로 만들까요?
```

승인하면 단계별 지시서가 생성된다. `/exit`로 나온다.

### 5.2 실행

```bash
python3 scripts/run_phases.py 0-mvp
```

AI가 단계마다 파일을 만들고, 시험을 돌리고, 커밋한다. 20분 안팎이 소요된다(원저자는 유사 규모에서 17분이 걸렸다고 밝혔다). 사람이 지켜보지 않아도 진행되며, 중간에 멈추려면 `Ctrl+C`, 다시 실행하면 완료된 단계는 건너뛰고 이어서 진행한다.

---

## 6. 완성물 검증

### 6.1 시험 실행

자동 실행 과정에서 AI가 시험 도구(pytest)를 담은 가상환경 `.venv`를 만들어 두었다. `.venv` 폴더가 없으면 AI가 다른 방식을 택한 것이므로 `pytest`를 그대로 실행한다.

```bash
./.venv/bin/python -m pytest
```

```text
.............................  [100%]
29 passed
```

### 6.2 실행 및 리포트 확인

```bash
./.venv/bin/python -m pcapeek.make_sample sample.pcap
./.venv/bin/python -m pcapeek.cli sample.pcap
```

```text
sample.pcap 를 만들었습니다 — 패킷 65개, 24,142바이트
report.html 을 만들었습니다: /home/student/pcapeek/report.html
```

터미널에는 두 줄만 출력되고, 결과는 브라우저에서 열린다. 브라우저가 자동으로 열리지 않는 환경에서는 출력된 경로를 직접 브라우저에 붙여 넣는다. 브라우저에 표시되는 화면은 화면 설계 단계에서 그린 와이어프레임과 같은 배치이다 — 제목, 요약 카드, 프로토콜 막대그래프, 상대·포트 표, 그리고 빨간 배경의 경고 카드(HTTP 12개·Telnet 4개).

| 리포트 내용 | 해석 |
|---|---|
| `65개 (해석 60 / 건너뜀 5)` | 5개는 IPv6 등 — "안 만들 것"으로 정한 항목이라 건너뜀 |
| `443 24개` | HTTPS — 암호화됨 |
| `80 12개` | HTTP — 암호화되지 않음 |
| `⚠ HTTP · Telnet` | 제3자가 읽을 수 있는 통신 — 빨간 카드로 강조 |

### 6.3 교차 검증

자체 결과만 믿지 않고 표준 도구로 확인한다.

```bash
tcpdump -r sample.pcap -c 3 -n
```

표준 도구 `tcpdump`가 생성된 파일을 읽으면 올바른 pcap 파일임이 확인된다. (`tcpdump`가 없으면 `sudo apt install -y tcpdump`)

---

## 7. 결과가 마음에 들지 않을 때

> **코드를 고치지 말고, 문서를 고쳐 다시 실행한다.**

`report.py`를 직접 수정하면 다음 실행에서 원래대로 돌아온다. `docs/OUTPUT_GUIDE.md`를 수정하면 영구적으로 반영된다.

```text
docs/OUTPUT_GUIDE.md 를 다시 읽고 report.py 를 거기에 맞춰 줘.
상위 목록을 5개가 아니라 3개만 보여 주게 바꿔 줘.
```

이것이 하네스의 핵심 원리이다 — 코드가 아니라 **AI에게 주는 맥락(문서)의 품질**을 올린다.

---

## 자주 발생하는 문제

| 증상 | 해결 |
|---|---|
| 훅이 종료 코드 1 | 구식 방식이다. 2.2절 지시문의 「반드시 지킬 것」대로 재지시 |
| 훅이 안전한 명령까지 차단 | "rm 은 -r/-f 붙은 것만 막게 해 줘" |
| 시작하자마자 멈춤 | Stop 훅이 시험 없음을 실패로 오판. "시험 없으면 통과(0)시켜 줘" |
| `docs/`에 `{` 잔존 | 해당 파일만 다시 채우도록 지시 |
| 이전 결과가 계속 나옴 | `find . -name __pycache__ -exec rm -rf {} +` |
| 처음부터 다시 하고 싶음 | `git checkout . && git clean -fd` (커밋 지점으로 복원) |
| 브라우저 자동 실행 실패 (`gio: ...` 오류) | 오류가 아니다. 출력된 report.html 경로를 직접 열기 |

---

## 검증 근거

| 본문의 주장 | 근거 |
|---|---|
| "마크다운만 채우면 된다"·17분 | [원저자 영상](https://www.youtube.com/watch?v=AQOvNx87Urs) 설명란 |
| 원본 훅이 현행 규격에서 아무것도 차단하지 못함 | 표준입력 JSON 실측 — 종료 코드 0 |
| `exit 2`가 차단 | [공식 Hooks 문서](https://code.claude.com/docs/en/hooks) |
| report.html 생성·수치(패킷 65, HTTP 12, Telnet 4 등) | Ubuntu 24.04 / Python 3.12.3 실측 |
| 브라우저 자동 실행 실패 시 `gio` 오류 | WSLg 환경 실측 |

AI가 만드는 결과물은 실행마다 조금씩 다르다. 파일 개수·시험 개수가 달라도 무방하며, **pytest가 통과하고 report.html이 요약과 경고를 담고 있으면** 완성이다. 단, `run_phases.py`로 AI가 코드를 생성하는 과정 자체는 검증 환경의 제약으로 실측하지 못했으므로 "20분"은 추정치이다.

---

## 다음 강의

11강부터는 심화 과정이다. 이번 강의에서 "AI에게 맡기고 넘어간" 부분 — 하네스의 4개 레이어 구조와 실행 엔진의 실제 코드 — 를 원본 자료를 직접 열어 확인한다.
