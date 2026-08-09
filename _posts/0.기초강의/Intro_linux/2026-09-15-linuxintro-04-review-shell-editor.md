---
title: 리눅스 기초 4강 종합문제 - 점검 도구 복구와 자동화
date: 2026-09-15 09:30:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - 종합문제
  - 복습
  - bash
  - 환경변수
  - PATH
  - 리다이렉션
  - 파이프
  - vi
  - nano
pin:
mermaid: false
---

> **종합문제 안내**
> 1. 본 글은 **제4강. 셸의 이해와 텍스트 편집기**의 복습용 종합문제이다.
> 2. 제4강의 실습 결과에 의존하지 않는다. 필요한 파일을 여기서 직접 생성하므로 **독립적으로 수행**할 수 있다.
> 3. 문제 6은 **고장난 스크립트를 진단하여 고치는 형식**이다. 오류 메시지를 읽고 원인을 추론하는 훈련이므로, 정답을 보기 전에 반드시 직접 실행해 볼 것을 권장한다.
> 4. 문제 8은 `~/.bashrc`를 수정한다. **시작 전에 백업본을 만들며, 마지막에 복원 절차를 수행**한다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 | 배정 시간 |
|---|---|---|---|
| 준비 | 실습 파일 생성 | — | 5분 |
| 문제 1 | 셸의 확인과 중첩 | 1.2 · 1.3 | 10분 |
| 문제 2 | 변수의 상속 범위 | 2.2 | 10분 |
| 문제 3 | `PATH`와 개인 명령 | 2.4 | 10분 |
| 문제 4 | 표준 출력과 표준 오류의 분리 | 3.1 · 3.2 | 15분 |
| 문제 5 | 파이프와 필터 조합 | 3.3 | 15분 |
| 문제 6 | 고장난 스크립트 3종 진단 | 1.2 · 2.1 · 4.2 | 25분 |
| 문제 7 | `vi`로 도구 작성 | 4.3 | 15분 |
| 문제 8 | 별칭과 초기화 파일 | 2.6 · 2.7 | 10분 |
| 마무리 | 자가 채점과 복원 | — | 10분 |

---
---

# 시나리오

---

학습자는 시스템 운영팀에 배치된 신입 담당자이다. 인수인계 과정에서 선임이 작성한 **일일 점검 스크립트 세 개**를 넘겨받았다.

팀장은 다음과 같이 지시하였다.

> *"넘겨받은 스크립트가 셋 다 동작하지 않습니다. 오류 메시지를 보고 원인을 찾아 고쳐 주십시오. 그리고 고친 뒤에는 매번 긴 경로를 입력하지 않고 어느 위치에서든 한 단어로 실행할 수 있게 정리해 주십시오. 마지막으로 세 도구의 결과를 하나로 모으는 요약 도구를 직접 작성하기 바랍니다."*

세 스크립트에는 **초심자가 가장 흔히 범하는 오류**가 하나씩 심어져 있다. 오류 메시지를 정확히 읽는 훈련이 본 종합문제의 핵심이다.

---
---

# 준비 단계. 실습 파일 생성

---

아래 블록을 **한 번에 복사하여 실행**한다. 고장난 스크립트 세 개가 생성된다.

```bash
mkdir -p ~/review04/tools ~/review04/logs && cd ~/review04/tools

cat > check_disk.sh << 'EOF'
#!/bin/basj
echo "[디스크 점검]"
df -h / | tail -1
EOF

cat > check_mem.sh << 'EOF'
#!/bin/bash
THRESHOLD = 80
echo "[메모리 점검]"
free -h | awk '/^Mem/ {print "  전체 " $2 " 중 " $3 " 사용"}'
echo "점검 시각 $(date '+%T')" > "$HOME/review04/logs/mem.log"
EOF

cat > check_user.sh << 'EOF'
#!/bin/sh
COUNT=$(awk -F: '$3>=1000 && $3<65534' /etc/passwd | wc -l)
echo "[계정 점검]"
if [[ $COUNT -gt 0 ]]; then
  echo "  일반 사용자 $COUNT 명"
fi
EOF

echo "== 준비 완료 =="
ls -l
```

> 세 파일 모두 **실행 권한이 없는 상태**로 생성되었다는 점에 유의한다. 이것이 첫 번째 관문이다.
>
> `<< 'EOF'`처럼 종료 표시를 작은따옴표로 감싸면 내부의 `$` 등이 **확장되지 않고 그대로 기록**된다. 스크립트 파일을 만들 때 반드시 사용하는 형태이다.

---
---

# 문제 1. 셸의 확인과 중첩

---

> **상황**
> 작업 전에 현재 환경이 어떤 셸인지 확인하여야 한다.
>
> **요구사항**
> 1. **로그인 셸**과 **현재 실행 중인 셸**을 각각 다른 명령으로 조회하고, 두 개념의 차이를 설명한다.
> 2. 로그인 셸로 지정할 수 있는 경로 목록이 담긴 파일을 확인한다.
> 3. `/bin/sh`의 실체가 무엇인지 확인한다.
> 4. 새 셸을 실행하여 **중첩 깊이가 증가**하는 것을 확인한 뒤 빠져나온다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
echo $SHELL
ps -p $$
cat /etc/shells
ls -l /bin/sh

echo $SHLVL
bash
echo $SHLVL
exit
echo $SHLVL
```

**해설**

- `$SHELL`은 `/etc/passwd` 7번째 필드에 기록된 **로그인 셸**을 담고 있을 뿐, **지금 실행 중인 셸을 보장하지 않는다.** 현재 셸은 `ps -p $$`로 확인한다. `$$`는 현재 셸의 PID를 담은 특수 변수이다.
- `chsh`로 로그인 셸을 바꿀 때 참조되는 파일이 **`/etc/shells`** 이다. 이 목록에 없는 경로는 지정할 수 없다.
- 우분투의 `/bin/sh`는 bash가 아니라 **dash**로 연결되어 있다. 부팅 스크립트를 빠르게 실행하기 위한 선택이며, 이 사실이 문제 6의 세 번째 스크립트와 직결된다.
- `SHLVL`은 셸의 중첩 깊이이다. `bash`를 실행하면 1 증가하고 `exit`하면 되돌아온다.

</details>

**완료 기준** — `/bin/sh -> dash`를 확인하였고, `SHLVL`이 증가했다가 되돌아온다.

---
---

# 문제 2. 변수의 상속 범위

---

> **상황**
> 스크립트에서 변수가 전달되지 않는 문제가 자주 발생한다. 그 경계를 실험으로 확인한다.
>
> **요구사항**
> 1. 변수 `SITE`에 `hanbit-plant`를 저장하고 화면에 출력한다.
> 2. **자식 셸에서 이 변수가 보이는지** 확인한다.
> 3. 환경 변수로 승격한 뒤 다시 확인한다.
> 4. `env`로 등록 여부를 조회한다.
> 5. 변수를 해제하고 사라졌는지 확인한다.
> 6. 등호 앞뒤에 공백을 넣었을 때 어떤 오류가 발생하는지 확인한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
SITE="hanbit-plant"
echo $SITE

bash -c 'echo "자식 셸에서 본 SITE = [$SITE]"'

export SITE
bash -c 'echo "자식 셸에서 본 SITE = [$SITE]"'

env | grep SITE

unset SITE
echo "해제 후 = [$SITE]"

SITE = "hanbit-plant"
```

**해설**

- 사용자 변수는 **현재 셸에서만** 유효하므로 자식 셸에서는 빈 값으로 보인다. `export`로 승격하여야 자식 프로세스까지 전달된다. **`export`가 그 경계를 결정한다.**
- 조회 명령도 구분된다. `set`은 사용자 변수를 포함한 전체를, `env`(또는 `printenv`)는 **환경 변수만** 출력한다.
- 마지막 명령은 `SITE: command not found` 오류를 낸다. 셸이 등호 앞의 `SITE`를 **명령어 이름으로 해석**하기 때문이다. 변수 대입에는 등호 앞뒤에 공백을 두지 않는다. 문제 6에서 같은 오류를 다시 만나게 된다.

</details>

**완료 기준** — `export` 전에는 빈 값, 후에는 값이 전달되는 것을 확인하였다.

---
---

# 문제 3. `PATH`와 개인 명령

---

> **상황**
> 점검 도구를 어느 위치에서든 한 단어로 실행할 수 있게 하여야 한다.
>
> **요구사항**
> 1. 현재 `PATH`를 **한 줄에 하나씩** 보기 좋게 출력한다.
> 2. `PATH`에 `.`(현재 디렉터리)이 포함되어 있지 않은지 확인하고, 포함되면 안 되는 이유를 설명한다.
> 3. `~/bin` 디렉터리를 만들고 `sitecheck`라는 이름의 간단한 명령을 배치한다.
> 4. 등록 전에는 실행되지 않고, `PATH`에 추가한 뒤에는 실행되는 것을 확인한다.
> 5. 그 명령이 실제로 어느 파일인지 조회한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
echo $PATH | tr ':' '\n'

mkdir -p ~/bin
printf '#!/bin/bash\necho "한빛정밀 점검 도구 v1"\n' > ~/bin/sitecheck
chmod +x ~/bin/sitecheck

sitecheck
~/bin/sitecheck

export PATH="$HOME/bin:$PATH"
sitecheck
which sitecheck
echo $PATH | tr ':' '\n' | head -3
```

**해설**

- `tr ':' '\n'`은 콜론을 줄바꿈으로 바꾸어 항목을 한 줄씩 보여 준다. 긴 `PATH`를 확인할 때 유용한 관용적 표현이다.
- **`PATH`에 `.`을 포함해서는 안 된다.** 포함되어 있으면 임의의 디렉터리에서 `ls`를 입력했을 때 그 디렉터리에 있는 동명의 파일이 먼저 실행된다. 공격자가 `/tmp` 등에 악성 파일을 두고 관리자가 그곳에서 명령을 입력하기를 기다리는 수법이 실제로 존재한다.
- 등록 전 `sitecheck`는 `command not found`가 되지만 `~/bin/sitecheck`처럼 **경로를 직접 지정하면** 실행된다. 스크립트를 실행할 때 `./script.sh`처럼 `./`를 붙이는 이유가 여기에 있다.
- 우분투의 `~/.profile`은 **`~/bin` 디렉터리가 존재하면 로그인 시 자동으로 `PATH`에 추가**한다. 따라서 다시 로그인하면 `export` 없이도 동작한다.

</details>

**완료 기준** — `which sitecheck`가 `/home/사용자명/bin/sitecheck`를 출력한다.

---
---

# 문제 4. 표준 출력과 표준 오류의 분리

---

> **상황**
> 점검 도구의 결과는 보관하되 오류 메시지는 따로 관리하여야 한다.
>
> **요구사항**
> 존재하는 경로와 존재하지 않는 경로를 함께 지정하여 다음을 각각 수행한다.
>
> | 번호 | 요구 동작 |
> |---|---|
> | ① | 정상 결과만 파일로 보내고 오류는 화면에 남긴다 |
> | ② | 정상 결과와 오류를 **서로 다른 파일**로 분리한다 |
> | ③ | 둘을 **하나의 파일**에 함께 기록한다 |
> | ④ | 오류만 완전히 버린다 |
> | ⑤ | 화면 출력과 파일 저장을 **동시에** 수행한다 |
{: .prompt-info }

**힌트** — 표준 출력은 1번, 표준 오류는 2번이다. `2>&1`은 리다이렉션 지정 **뒤에** 배치하여야 한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cd ~/review04/logs

ls -l /etc /nowhere > out1.txt
cat out1.txt

ls -l /etc /nowhere > out2.txt 2> err2.txt
cat err2.txt

ls -l /etc /nowhere > all3.txt 2>&1
tail -3 all3.txt

ls -l /nowhere 2>/dev/null
echo "종료 상태: $?"

df -h | tee disk.txt | grep "^/dev"
cat disk.txt | head -3
```

**해설**

- `>`는 **표준 출력(1번)만** 처리하므로 ①에서는 오류가 화면에 남는다. 두 통로가 분리되어 있다는 사실이 눈으로 확인되는 대목이다.
- `2>`는 표준 오류만 보낸다. ②처럼 분리해 두면 정상 결과는 보고서로, 오류는 장애 분석용으로 각각 활용할 수 있다.
- `> 파일 2>&1`은 "출력을 파일로 보낸 뒤, 오류도 **출력과 같은 곳**으로 보내라"는 의미이다. 순서를 바꾸어 `2>&1 > 파일`로 쓰면 오류가 복제되는 시점에 출력이 아직 화면을 가리키고 있으므로 **오류는 화면에 남는다.** `2>&1`은 반드시 마지막에 둔다.
- `/dev/null`은 기록된 모든 내용을 버리는 특수 장치이다. 오류를 버려도 **종료 상태(`$?`)는 그대로 남으므로** 성공 여부는 별도로 판정할 수 있다.
- `tee`는 배관의 T자 이음쇠에서 유래한 명칭으로, 흐름을 화면과 파일 두 방향으로 나눈다.

</details>

**완료 기준** — `err2.txt`에는 오류만, `out2.txt`에는 정상 결과만 들어 있다.

---
---

# 문제 5. 파이프와 필터 조합

---

> **상황**
> 서버의 계정 현황을 통계로 제출하여야 한다. 전용 프로그램 없이 명령 조합만으로 처리한다.
>
> **요구사항**
> 1. `/etc/passwd`에서 **로그인 셸의 종류별 계정 수**를 많은 순으로 집계한다.
> 2. **UID가 1000 이상 65534 미만인 일반 사용자**의 이름만 출력한다.
> 3. 홈 디렉터리가 `/home` 아래에 있는 계정의 수를 센다.
> 4. `sudo` 그룹 구성원을 **한 줄에 하나씩** 출력한다.
> 5. 위 결과를 하나의 요약 파일로 저장한다.
{: .prompt-info }

**힌트** — `cut -d: -f필드번호`로 항목을 잘라 내고, `sort | uniq -c | sort -rn`으로 빈도를 집계한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn

awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd

grep -c ":/home/" /etc/passwd

grep "^sudo:" /etc/group | cut -d: -f4 | tr ',' '\n'

{
  echo "===== 계정 현황 요약 ====="
  echo "생성 시각: $(date '+%F %T')"
  echo
  echo "[1] 로그인 셸별 계정 수"
  cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn | sed 's/^/  /'
  echo
  echo "[2] 일반 사용자"
  awk -F: '$3 >= 1000 && $3 < 65534 {print "  - " $1 " (UID " $3 ")"}' /etc/passwd
  echo
  echo "[3] sudo 그룹 구성원"
  grep "^sudo:" /etc/group | cut -d: -f4 | tr ',' '\n' | sed 's/^/  - /'
} > ~/review04/logs/account_summary.txt

cat ~/review04/logs/account_summary.txt
```

**해설**

- `uniq`는 **인접한** 중복만 제거하므로 반드시 `sort`를 먼저 거쳐야 한다. `sort | uniq -c | sort -rn`은 빈도 집계의 표준 관용구이다.
- `sort -rn`의 `-n`은 숫자 정렬, `-r`은 역순이다. `-n` 없이 정렬하면 `10`이 `9`보다 앞에 오는 문자열 정렬이 되어 결과가 왜곡된다.
- 결과에서 **`/usr/sbin/nologin` 계정이 압도적으로 많다**는 점을 확인한다. 서비스마다 최소 권한의 전용 계정을 부여하고 대화형 로그인을 차단한 결과이다.
- `awk -F:`의 `$3`은 UID 필드이다. `65534`(nobody)를 제외하는 조건이 관례적으로 함께 쓰인다.

</details>

**완료 기준** — `account_summary.txt`에 세 항목이 모두 기록되어 있다.

---
---

# 문제 6. 고장난 스크립트 3종 진단 ★핵심

---

> **상황**
> 인수인계받은 세 스크립트가 모두 동작하지 않는다. **하나씩 실행하여 오류 메시지를 확인하고, 원인을 진단한 뒤 수정**하여야 한다.
>
> **요구사항** — 각 스크립트마다 다음 세 가지를 수행한다.
> 1. 실행하여 오류 메시지를 그대로 기록한다.
> 2. 원인을 한 문장으로 설명한다.
> 3. `nano` 또는 `vi`로 수정하고 정상 동작을 확인한다.
{: .prompt-info }

먼저 세 스크립트를 실행해 본다.

```bash
cd ~/review04/tools
./check_disk.sh
./check_mem.sh
./check_user.sh
```

**힌트** — 첫 번째 관문은 세 파일에 **공통으로** 존재한다. 그 다음부터 개별 원인이 드러난다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

**공통 관문 — 실행 권한**

```bash
ls -l
```

세 파일 모두 `-rw-rw-r--`로 **실행 권한(`x`)이 없다.** 따라서 첫 실행에서는 모두 `Permission denied`가 출력된다.

```bash
chmod +x check_disk.sh check_mem.sh check_user.sh
ls -l
```

**① `check_disk.sh` — 셔뱅 오류**

```
bash: ./check_disk.sh: /bin/basj: bad interpreter: No such file or directory
```

첫 행의 `#!/bin/basj`에서 **인터프리터 경로에 오타**가 있다. `bash`를 `basj`로 잘못 적은 것이다. 셔뱅은 이 파일을 어떤 프로그램으로 해석할지 지정하는 행이므로, 경로가 잘못되면 실행 자체가 불가능하다.

```bash
sed -i '1s|.*|#!/bin/bash|' check_disk.sh
head -1 check_disk.sh
./check_disk.sh
```

> `sed -i '1s|...|...|'`은 1행을 통째로 치환하라는 의미이다. 편집기로 수정하여도 무방하다.

**② `check_mem.sh` — 변수 대입의 공백**

```
./check_mem.sh: line 2: THRESHOLD: command not found
```

`THRESHOLD = 80`에서 **등호 앞뒤에 공백**이 있어 셸이 `THRESHOLD`를 명령어로 해석하였다. 문제 2에서 확인한 것과 동일한 오류이다.

또한 이 스크립트에는 **두 번째 결함**이 있다. 마지막 행이 `>`로 로그를 기록하므로 **실행할 때마다 이전 기록이 지워진다.** 점검 이력을 남기려면 `>>`를 사용하여야 한다.

```bash
sed -i 's/^THRESHOLD = 80/THRESHOLD=80/' check_mem.sh
sed -i 's|> "$HOME/review04/logs/mem.log"|>> "$HOME/review04/logs/mem.log"|' check_mem.sh
./check_mem.sh
./check_mem.sh
cat ~/review04/logs/mem.log
```

수정 전에는 실행할 때마다 로그가 한 행으로 초기화되지만, 수정 후에는 실행 횟수만큼 행이 늘어난다. **행이 두 개 이상 누적되어 있으면** 수정이 완료된 것이다.

**③ `check_user.sh` — 셸 종류의 불일치**

```
./check_user.sh: 4: [[: not found
```

첫 행이 `#!/bin/sh`인데 본문에서 **bash 전용 문법인 `[[ ]]`** 를 사용하였다. 우분투의 `/bin/sh`는 dash이므로 이 문법을 지원하지 않는다(문제 1에서 확인한 사실이다).

> **이 오류가 특히 위험한 이유**
> dash는 `[[`를 알 수 없는 명령으로 취급하고 **오류를 출력한 뒤 그대로 다음 행으로 진행**한다. `if` 조건이 거짓으로 판정되어 안쪽 내용이 실행되지 않을 뿐, 스크립트 자체는 **종료 상태 0(성공)으로 끝난다.**
>
> 즉 `[계정 점검]`이라는 머리글만 출력되고 정작 필요한 계정 수는 나오지 않는데도, 자동화 도구는 이를 "정상 종료"로 판단한다. **오류 메시지를 읽지 않고 종료 상태만 확인하면 놓치게 되는 유형**이다.
{: .prompt-danger }

해결 방법은 두 가지이며, 어느 쪽을 선택하여도 무방하다.

```bash
# 방법 1 — 셔뱅을 bash로 바꾼다(가장 간단하다)
sed -i '1s|.*|#!/bin/bash|' check_user.sh
./check_user.sh
```

```bash
# 방법 2 — POSIX 표준 문법인 [ ] 로 바꾼다(sh에서도 동작한다)
# sed -i 's|\[\[ \$COUNT -gt 0 \]\]|[ "$COUNT" -gt 0 ]|' check_user.sh
```

**종합 해설**

| 오류 메시지 | 원인 | 조치 |
|---|---|---|
| `Permission denied` | 실행 권한 없음 | `chmod +x 파일` |
| `bad interpreter: No such file or directory` | 셔뱅의 경로 오타 | 첫 행을 `#!/bin/bash`로 수정 |
| `이름: command not found` | 변수 대입에 공백 사용 | `이름=값` 형태로 수정 |
| `[[: not found` | `#!/bin/sh`에서 bash 전용 문법 사용 | 셔뱅을 bash로 바꾸거나 `[ ]`로 수정 |
| 로그가 매번 초기화됨 | `>`(덮어쓰기) 사용 | `>>`(이어쓰기)로 수정 |

**오류 메시지는 원인을 이미 알려 주고 있다.** 메시지를 끝까지 읽는 습관이 진단 능력의 출발점이다.

</details>

**완료 기준** — 세 스크립트가 모두 정상 실행되고, `mem.log`에 실행 횟수만큼 행이 누적된다.

---
---

# 문제 7. `vi`로 요약 도구 작성

---

> **상황**
> 세 도구의 결과를 하나로 모으는 요약 도구를 직접 작성하여야 한다.
>
> **요구사항**
> 1. `vi`로 `~/review04/tools/daily_report.sh`를 작성한다.
> 2. 내용은 아래와 같으며, 세 점검 스크립트를 차례로 호출한다.
> 3. 작성 도중 다음 조작을 **반드시 한 번씩 사용**한다.
>    - `i`로 입력 모드 진입, `Esc`로 명령 모드 복귀
>    - `:set nu`로 행 번호 표시
>    - `dd`로 한 행 삭제한 뒤 `u`로 복원
>    - `:%s/기존/변경/g`로 문서 전체 치환
>    - `:wq`로 저장 후 종료
> 4. 실행 권한을 부여하고 실행한다.
{: .prompt-info }

작성할 내용은 다음과 같다.

```bash
#!/bin/bash
# 일일 점검 요약 도구

TOOLS="$HOME/review04/tools"
OUT="$HOME/review04/logs/daily_$(date +%Y%m%d).txt"

{
  echo "=========================================="
  echo "  일일 점검 요약"
  echo "  점검 시각: $(date '+%Y-%m-%d %H:%M:%S')"
  echo "  점검 계정: $(whoami)@$(hostname)"
  echo "=========================================="
  echo
  "$TOOLS/check_disk.sh"
  echo
  "$TOOLS/check_mem.sh"
  echo
  "$TOOLS/check_user.sh"
} > "$OUT" 2>&1

echo "요약을 생성하였습니다: $OUT"
cat "$OUT"
```

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
vi ~/review04/tools/daily_report.sh
```

편집기에서 다음 순서로 조작한다.

| 순서 | 입력할 키 | 동작 |
|---|---|---|
| 1 | `i` | 입력 모드로 전환(하단에 `-- INSERT --` 표시) |
| 2 | (내용 입력) | 위 스크립트를 작성 |
| 3 | `Esc` | 명령 모드로 복귀 |
| 4 | `:set nu` + `Enter` | 행 번호 표시 |
| 5 | `dd` → `u` | 현재 행 삭제 후 복원 |
| 6 | `:%s/일일 점검/일일 시스템 점검/g` + `Enter` | 문서 전체 치환 |
| 7 | `:wq` + `Enter` | 저장 후 종료 |

```bash
chmod +x ~/review04/tools/daily_report.sh
~/review04/tools/daily_report.sh
ls -l ~/review04/logs/
```

**해설**

- `vi`는 실행 직후 **명령 모드**이다. 이 상태에서 문자를 입력하면 텍스트가 아니라 명령으로 해석되므로, 반드시 `i`를 눌러 입력 모드로 전환한 뒤 작성한다. 상태를 알 수 없으면 `Esc`를 눌러 명령 모드로 돌아간다.
- `:%s/기존/변경/g`에서 `%`는 **문서 전체**, `g`는 **각 행의 모든 항목**을 의미한다. `%`가 없으면 현재 행만, `g`가 없으면 각 행의 첫 항목만 바뀐다.
- `u`(undo)를 알고 있으면 vi에서의 어떤 조작도 되돌릴 수 있다. 편집을 크게 그르쳤다면 `:q!`로 저장하지 않고 빠져나온다.
- 스크립트에서 `{ ... } > "$OUT" 2>&1` 형태를 사용하면 **정상 출력과 오류가 모두** 한 파일에 기록된다. 자동 실행되는 도구는 오류도 함께 남겨야 사후에 원인을 추적할 수 있다.
- 경로를 `"$HOME/..."`처럼 큰따옴표로 감싸는 이유는, 경로에 공백이 포함되어도 하나의 인자로 전달되도록 하기 위함이다.

</details>

**완료 기준** — `~/review04/logs/daily_날짜.txt`가 생성되고 세 점검 결과가 모두 담겨 있다.

---
---

# 문제 8. 별칭과 초기화 파일

---

> **상황**
> 자주 쓰는 명령을 짧게 줄이고, 작성한 도구를 영구적으로 등록하여야 한다.
>
> **요구사항**
> 1. **작업 전에 `~/.bashrc`의 백업본을 만든다.**
> 2. 별칭 `ll`(상세 목록), `lt`(시각순 목록), `dr`(요약 도구 실행)을 정의하여 즉시 사용해 본다.
> 3. 별칭이 **자식 셸에서는 동작하지 않는** 것을 확인한다.
> 4. 세 별칭과 `PATH` 설정을 `~/.bashrc`에 기록하여 영구 적용한다.
> 5. 설정을 즉시 반영하고 확인한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cp ~/.bashrc ~/.bashrc.review04.bak

alias ll='ls -alF'
alias lt='ls -lhtr'
alias dr='$HOME/review04/tools/daily_report.sh'

ll ~/review04
bash -c 'll'

cat >> ~/.bashrc << 'EOF'

# ===== 리눅스 기초 4강 종합문제 설정 =====
export PATH="$HOME/bin:$PATH"
alias ll='ls -alF'
alias lt='ls -lhtr'
alias dr='$HOME/review04/tools/daily_report.sh'
EOF

source ~/.bashrc
alias | tail -3
type ll
```

**해설**

- **설정 파일을 수정하기 전에 백업본을 만드는 것은 시스템 관리의 기본 수칙이다.** 복구 수단을 확보한 뒤에 변경한다.
- `bash -c 'll'`은 `command not found`가 된다. 별칭은 **대화형 셸에서만** 적용되며 **스크립트나 자식 셸에서는 동작하지 않는다.** 스크립트에서는 항상 원래 명령을 사용하여야 하는 이유이다.
- 별칭과 프롬프트는 `~/.bashrc`에, 환경 변수는 `~/.profile`에 기재하는 것이 관례이다. 우분투의 `~/.profile`이 앞부분에서 `~/.bashrc`를 호출하므로 로그인 셸에서도 별칭이 적용된다.
- `source ~/.bashrc`(또는 `. ~/.bashrc`)는 새로 로그인하지 않고 설정을 즉시 반영한다.
- 별칭을 무시하고 원래 명령을 실행하려면 앞에 역슬래시를 붙여 `\ls`와 같이 입력한다.

</details>

**완료 기준** — `type ll`이 별칭으로 정의되어 있음을 보여 주고, 새 터미널에서도 `dr`가 동작한다.

---
---

# 마무리. 자가 채점

---

```bash
cd ~/review04
echo "===== 자가 채점 결과 ====="

[ -x tools/check_disk.sh ] && [ -x tools/check_mem.sh ] && [ -x tools/check_user.sh ] \
  && echo "[문제 6-공통] 통과 — 실행 권한 부여" || echo "[문제 6-공통] 미완료"

head -1 tools/check_disk.sh | grep -q "^#!/bin/bash$" \
  && echo "[문제 6-①] 통과 — 셔뱅 수정" || echo "[문제 6-①] 미완료"

grep -q "^THRESHOLD=80$" tools/check_mem.sh \
  && echo "[문제 6-②a] 통과 — 변수 대입 수정" || echo "[문제 6-②a] 미완료"

grep -q ">> \"\$HOME/review04/logs/mem.log\"" tools/check_mem.sh \
  && echo "[문제 6-②b] 통과 — 이어쓰기로 변경" || echo "[문제 6-②b] 미완료"

./tools/check_user.sh 2>/dev/null | grep -q "일반 사용자" \
  && echo "[문제 6-③] 통과 — 계정 점검 정상 실행" || echo "[문제 6-③] 미완료"

[ -x ~/bin/sitecheck ] && command -v sitecheck > /dev/null \
  && echo "[문제 3] 통과 — 개인 명령 등록" || echo "[문제 3] 미완료"

[ -s logs/err2.txt ] && [ -s logs/out2.txt ] \
  && echo "[문제 4] 통과 — 출력·오류 분리" || echo "[문제 4] 미완료"

[ -s logs/account_summary.txt ] \
  && echo "[문제 5] 통과 — 계정 통계 작성" || echo "[문제 5] 미완료"

ls logs/daily_*.txt > /dev/null 2>&1 \
  && echo "[문제 7] 통과 — 요약 도구 작성" || echo "[문제 7] 미완료"

grep -q "alias ll=" ~/.bashrc \
  && echo "[문제 8] 통과 — 별칭 영구 등록" || echo "[문제 8] 미완료"

[ "$(wc -l < logs/mem.log 2>/dev/null)" -ge 2 ] \
  && echo "[추가] 통과 — 로그가 누적되고 있음" || echo "[추가] 미완료 — check_mem.sh의 >> 수정을 확인"
```

---

## 이론 점검

**문항 1.** `echo $SHELL`과 `ps -p $$`의 결과가 다를 수 있는 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

`$SHELL`은 `/etc/passwd`에 기록된 **로그인 셸**을 담고 있을 뿐이며, **현재 실행 중인 셸**을 보장하지 않는다. 다른 셸을 실행한 상태라면 두 값이 달라진다.

</details>

**문항 2.** `#!/bin/sh`로 시작하는 스크립트에서 `[[ ]]`를 사용하면 실패하는 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

우분투의 `/bin/sh`는 bash가 아니라 **dash**로 연결되어 있으며, `[[ ]]`는 **bash 전용 확장 문법**이기 때문이다. bash의 기능이 필요하면 셔뱅을 `#!/bin/bash`로 명시하여야 한다.

</details>

**문항 3.** `cmd > log.txt 2>&1`과 `cmd 2>&1 > log.txt`의 차이는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

리다이렉션은 **왼쪽에서 오른쪽 순서로** 처리된다. 후자는 표준 오류가 복제되는 시점에 표준 출력이 아직 화면을 가리키고 있으므로 **오류가 화면에 남는다.** `2>&1`은 반드시 마지막에 배치한다.

</details>

**문항 4.** 별칭을 정의하였는데 스크립트 안에서 동작하지 않는 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

별칭은 **대화형 셸에서만** 적용된다. 스크립트는 비대화형 셸에서 실행되므로 별칭이 적용되지 않으며, 스크립트에서는 원래 명령을 사용하여야 한다.

</details>

**문항 5.** `PATH`에 `.`을 포함하면 안 되는 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

임의의 디렉터리에서 명령을 입력했을 때 **그 디렉터리에 있는 동명의 파일이 먼저 실행**되기 때문이다. 공격자가 `ls` 등의 이름으로 악성 파일을 배치해 두는 수법에 이용될 수 있다.

</details>

---
---

# 정리와 복원

---

## `~/.bashrc` 복원

문제 8에서 추가한 설정을 유지하려면 이 단계를 생략한다. 원래대로 되돌리려면 다음을 실행한다.

```bash
cp ~/.bashrc.review04.bak ~/.bashrc
```

```bash
source ~/.bashrc
```

```bash
tail -5 ~/.bashrc
```

## 실습 파일 정리

```bash
rm -rf ~/review04
```

```bash
rm -f ~/bin/sitecheck
```

> 백업본 `~/.bashrc.review04.bak`은 남겨 두어도 무방하다. 삭제하려면 `rm -f ~/.bashrc.review04.bak`을 실행한다.
{: .prompt-tip }

---

## 자기 점검

```
 [ ] 로그인 셸과 현재 셸의 차이를 설명할 수 있다.
 [ ] 사용자 변수와 환경 변수의 경계를 export로 설명할 수 있다.
 [ ] PATH의 탐색 원리와 . 을 포함하면 안 되는 이유를 설명할 수 있다.
 [ ] 표준 출력과 표준 오류를 각각 다른 파일로 분리할 수 있다.
 [ ] sort · uniq -c · cut · awk를 조합하여 통계를 낼 수 있다.
 [ ] 네 가지 대표 오류 메시지를 보고 원인을 즉시 지목할 수 있다.
 [ ] vi에서 입력·저장·취소·치환을 수행할 수 있다.
 [ ] ~/.bashrc를 백업한 뒤 수정하고 복원까지 완료하였다.
```

다음 복습은 **제5강 종합문제 — 야간 장애 대응**이다.
