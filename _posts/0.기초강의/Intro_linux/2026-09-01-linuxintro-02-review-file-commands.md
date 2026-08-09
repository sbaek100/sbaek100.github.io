---
title: 리눅스 기초 2강 종합문제 - 학과 자료실 정비
date: 2026-09-01 09:30:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - 종합문제
  - 복습
  - ls
  - find
  - grep
  - tar
  - 와일드카드
pin:
mermaid: false
---

> **종합문제 안내**
> 1. 본 글은 **제2강. 디렉터리 구조와 기본 명령어**의 복습용 종합문제이다.
> 2. 제2강의 실습 결과(`~/lab02`)에 의존하지 않는다. **필요한 자료를 여기서 직접 생성**하므로 독립적으로 수행할 수 있다.
> 3. 각 문제는 **먼저 스스로 작성해 본 뒤** 정답을 확인하는 방식으로 진행한다. 정답은 접혀 있으므로 눌러서 펼친다.
> 4. 모든 명령은 실제로 실행하여야 한다. **끝까지 수행하면 하나의 자료 정비 작업이 완성된다.**
{: .prompt-info }

| 문제 | 주제 | 대응 절 | 배정 시간 |
|---|---|---|---|
| 준비 | 실습 자료 생성 | — | 5분 |
| 문제 1 | 현황 파악과 목록 조회 | 1.3 · 1.4 | 10분 |
| 문제 2 | 절대 경로와 상대 경로 | 1.2 | 5분 |
| 문제 3 | 분류 디렉터리 생성과 이동 | 2.1 ~ 2.4 | 10분 |
| 문제 4 | 숨김 파일과 오래된 파일 처리 | 1.3 · 4.1 | 10분 |
| 문제 5 | 대용량 파일 탐지 | 4.1 | 5분 |
| 문제 6 | 로그 분석 | 4.2 · 4.3 | 15분 |
| 문제 7 | 백업과 복원 검증 | 4.4 | 10분 |
| 문제 8 | 처리 보고서 작성 | 2.2 · 4.3 | 10분 |
| 마무리 | 자가 채점과 정리 | — | 10분 |

---
---

# 시나리오

---

학습자는 대학 학과 사무실의 **전산 자료 담당 조교**이다. 학과에서 사용하는 자료 서버의 접수 폴더에는 각 담당자가 제출한 파일이 아무런 규칙 없이 쌓여 있다.

행정실장은 다음과 같이 지시하였다.

> *"학기가 끝나기 전에 접수 폴더를 정리해 주십시오. 요구사항은 네 가지입니다. 첫째, 확장자별로 분류할 것. 둘째, 1년이 지난 자료와 용량이 큰 자료를 따로 모을 것. 셋째, 서버 점검 기록에서 오류가 몇 건이나 발생했는지 집계할 것. 넷째, 정리 결과를 압축 보관하고 복원이 가능한지까지 확인할 것. 처리 내역은 보고서로 제출해 주십시오."*

본 종합문제는 이 지시를 **제2강에서 학습한 명령만으로** 수행한다.

> **작업 원칙**
> 파일을 이동하거나 삭제하는 명령을 실행하기 전에는 **반드시 동일한 대상을 `ls` 또는 `find`로 먼저 확인**한다. 제2강 2.3절에서 강조한 작업 습관이며, 본 종합문제의 채점 기준에도 포함된다.
{: .prompt-warning }

---
---

# 준비 단계. 실습 자료 생성

---

아래 블록을 **한 번에 복사하여 실행**한다. 정비 대상이 되는 자료가 생성된다.

```bash
mkdir -p ~/review02/inbox/notes && cd ~/review02/inbox

printf 'name,score\nkim,88\nlee,92\npark,79\n' > grade_2026spring.csv
printf 'name,dept\nkim,network\nlee,security\n' > member_2026.csv
printf 'item,count\nchair,40\ndesk,38\n' > asset_2025.csv

printf 'INFO: check started\nERROR: disk usage 91%%\nINFO: backup done\nERROR: smtp timeout\n' > srv_check_01.log
printf 'INFO: check started\nWARN: high memory\nINFO: check finished\n' > srv_check_02.log
printf 'ERROR: service not responding\nINFO: restarted\nERROR: disk usage 93%%\n' > srv_check_03.log

printf 'PORT=8080\nDEBUG=false\n' > webapp.conf
printf 'TIMEOUT=30\nRETRY=3\n' > agent.conf

echo "학과 자료실 이용 안내" > notice.txt
echo "장비 반납 요령" > notes/return_guide.md
echo "임시 저장본" > .draft_memo.txt
echo "이전 설정 백업" > .agent.conf.bak

touch -d "2025-01-20" old_notice.txt
touch -d "2025-02-11" asset_2025.csv
dd if=/dev/zero of=camera_dump.log bs=1M count=3 status=none

echo "== 준비 완료 =="
ls -la
```

> `dd`는 지정한 크기의 자료를 만들어 내는 명령으로, 여기서는 **용량이 큰 파일을 흉내 내기 위해서만** 사용하였다. 제2강의 학습 범위는 아니므로 의미만 이해하고 넘어간다.
>
> `touch -d "날짜"`는 파일의 수정 시각을 지정한 날짜로 바꾼다. 문제 4에서 "오래된 파일"을 검색하기 위한 준비이다.

---
---

# 문제 1. 현황 파악과 목록 조회

---

> **상황**
> 정비를 시작하기 전에 접수 폴더의 규모를 파악하여야 한다.
>
> **요구사항**
> 1. 현재 위치의 **숨김 파일을 포함한** 전체 목록을 상세 형식으로 출력한다.
> 2. `inbox` 아래에 존재하는 **일반 파일의 총 개수**를 구한다.
> 3. `inbox` 아래에 존재하는 **디렉터리의 개수**를 구한다.
> 4. 각 파일의 용량을 **사람이 읽기 쉬운 단위**로, **최근에 수정된 것이 아래에 오도록** 정렬하여 출력한다.
{: .prompt-info }

**힌트** — `ls`의 `-l` `-a` `-h` `-t` `-r` 옵션은 하나로 묶어 쓸 수 있다. 개수는 `find`와 `wc -l`을 파이프로 연결한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
ls -la
ls -lah
find . -type f | wc -l
find . -type d | wc -l
ls -lhtr
```

**해설**

- `ls -la`의 `-a`가 없으면 `.draft_memo.txt`와 `.agent.conf.bak`이 보이지 않는다. 리눅스에서 **마침표로 시작하는 이름은 숨김 파일**이다.
- 일반 파일은 **14개**(숨김 파일 2개 포함), 디렉터리는 `.`(자기 자신)과 `notes`를 합하여 **2개**이다. `find`는 숨김 파일도 함께 세므로 `ls`로 눈으로 센 개수와 다를 수 있다는 점에 유의한다.
- `find . -type d`는 시작 지점인 현재 디렉터리도 포함하여 세므로, 하위 디렉터리만 세려면 `find . -mindepth 1 -type d | wc -l`을 사용한다.
- `-lhtr`는 상세(`l`) + 읽기 쉬운 단위(`h`) + 시각순(`t`) + 역순(`r`)의 조합이다. 가장 최근 파일이 화면 아래에 오므로 긴 목록에서 유용하다.

</details>

**완료 기준** — 일반 파일 14개, 디렉터리 2개가 확인되고 `camera_dump.log`가 `3.0M`로 표시된다.

---
---

# 문제 2. 절대 경로와 상대 경로

---

> **상황**
> 정비 도중 여러 위치를 오가야 한다. 경로 지정 방식을 정확히 구분하여야 한다.
>
> **요구사항**
> 1. **절대 경로**를 사용하여 `/etc/ssh` 디렉터리로 이동한 뒤 현재 위치를 출력한다.
> 2. **상대 경로**만 사용하여 상위 디렉터리로 한 단계 올라간 뒤 현재 위치를 출력한다.
> 3. **직전에 있던 디렉터리**로 되돌아간다.
> 4. 한 번의 명령으로 자신의 홈 디렉터리로 이동한 뒤, `inbox`의 하위 `notes` 디렉터리로 상대 경로를 사용하여 이동한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cd /etc/ssh
pwd
cd ..
pwd
cd -
pwd
cd
cd review02/inbox/notes
pwd
```

**해설**

- 맨 앞에 `/`가 있으면 **절대 경로**, 없으면 **상대 경로**이다. `cd /etc/ssh`는 어느 위치에서 실행하더라도 같은 곳에 도달한다.
- `cd ..`는 상위 디렉터리, `cd -`는 **직전 디렉터리**로 이동하며 이동한 경로를 화면에 출력한다.
- 인자 없는 `cd`는 언제나 홈 디렉터리로 이동한다. **현재 위치를 잃었을 때 가장 먼저 사용하는 명령**이다.
- 마지막 `cd review02/inbox/notes`에는 `/`가 앞에 없으므로 홈 디렉터리를 기준으로 해석된다.

</details>

**완료 기준** — 마지막 `pwd`의 결과가 `/home/사용자명/review02/inbox/notes`이다.

---
---

# 문제 3. 분류 디렉터리 생성과 이동

---

> **상황**
> 확장자별로 자료를 분류하여야 한다.
>
> **요구사항**
> 1. `inbox` 아래에 `sorted` 디렉터리를 만들고, 그 안에 `csv`·`log`·`conf`·`doc` 네 개의 하위 디렉터리를 **한 줄의 명령으로** 생성한다.
> 2. 각 확장자의 파일을 옮기되, **이동 전에 반드시 대상 목록을 먼저 확인**한다.
> 3. `.txt`와 `.md` 파일은 `doc`으로 옮긴다.
> 4. 분류 결과를 트리 형태로 확인한다.
{: .prompt-info }

**힌트** — 중괄호 확장 `{a,b,c}`를 사용하면 여러 이름을 한 번에 지정할 수 있다. 숨김 파일은 `*`로 지정되지 않는다는 점에 유의한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cd ~/review02/inbox
mkdir -p sorted/{csv,log,conf,doc}

ls *.csv
mv *.csv sorted/csv/

ls *.log
mv *.log sorted/log/

ls *.conf
mv *.conf sorted/conf/

ls *.txt
mv *.txt sorted/doc/

ls -R sorted
```

**해설**

- `mkdir -p sorted/{csv,log,conf,doc}`는 `sorted/csv sorted/log sorted/conf sorted/doc` 네 경로로 확장되어 한 번에 생성된다. `-p`는 중간 디렉터리까지 함께 만들라는 의미이다.
- **`ls`로 먼저 확인하고 `mv`를 실행하는 순서**가 핵심이다. 와일드카드는 셸이 먼저 확장하므로, `ls`의 결과가 곧 `mv`가 처리할 대상이다.
- `.md` 파일은 `notes` 디렉터리 안에 있으므로 `mv notes/*.md sorted/doc/`과 같이 경로를 지정하여야 한다. 현재 위치의 `*.md`로는 지정되지 않는다.
- 숨김 파일 두 개(`.draft_memo.txt`, `.agent.conf.bak`)는 `*.txt`·`*.conf`로 **지정되지 않는다.** 와일드카드 `*`는 마침표로 시작하는 이름과 일치하지 않기 때문이며, 이 문제는 문제 4에서 다룬다.

</details>

**완료 기준** — `ls` 결과에 확장자를 가진 파일이 더 이상 남아 있지 않고(숨김 파일 제외), `sorted` 아래 네 디렉터리에 자료가 분산되어 있다.

---
---

# 문제 4. 숨김 파일과 오래된 파일 처리

---

> **상황**
> 문제 3을 마친 뒤 목록을 다시 확인하니 옮겨지지 않은 파일이 남아 있다. 또한 1년 이상 지난 자료를 별도로 보관하여야 한다.
>
> **요구사항**
> 1. 아직 옮겨지지 않은 파일이 무엇인지, 왜 남았는지 확인한다.
> 2. `notes` 디렉터리의 `.md` 파일을 `sorted/doc`으로 옮긴다.
> 3. 숨김 파일 두 개를 `sorted/hidden` 디렉터리를 만들어 옮긴다.
> 4. `sorted` 아래에서 **수정된 지 30일이 지난 파일**을 검색한다.
> 5. 검색된 파일을 `archive_old` 디렉터리로 옮긴다.
{: .prompt-info }

**힌트** — 숨김 파일은 `.*` 또는 `ls -A`로 확인한다. 오래된 파일은 `find`의 `-mtime +30` 조건을 사용한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
ls -A
mv notes/*.md sorted/doc/

mkdir -p sorted/hidden
ls -d .*.txt .*.bak
mv .draft_memo.txt .agent.conf.bak sorted/hidden/

find sorted -type f -mtime +30

mkdir -p archive_old
find sorted -type f -mtime +30 -exec mv {} archive_old/ \;
ls -l archive_old
```

**해설**

- `ls -A`는 `.`과 `..`을 제외한 숨김 파일을 함께 보여 준다. 남아 있던 이유는 **와일드카드 `*`가 마침표로 시작하는 이름을 지정하지 않기 때문**이다.
- `-mtime +30`은 "마지막 수정 시각이 30일보다 이전"이라는 조건이다. 준비 단계에서 `touch -d`로 시각을 과거로 지정한 `old_notice.txt`와 `asset_2025.csv`가 검색된다.
- `-exec 명령 {} \;`에서 `{}`는 찾은 경로로 치환되고 `\;`는 명령의 끝을 표시한다. 여러 항목을 한 번에 넘기려면 `\;` 대신 `+`를 사용하지만, `mv`처럼 **마지막 인자가 목적지여야 하는 명령**에는 `\;` 형태가 안전하다.

</details>

**완료 기준** — `archive_old`에 파일 2개가 존재하고, `inbox` 최상위에는 `sorted`·`archive_old`·`notes` 디렉터리만 남는다.

---
---

# 문제 5. 대용량 파일 탐지

---

> **상황**
> 자료실 용량이 부족하다는 통보를 받았다. 용량을 많이 차지하는 자료를 찾아 별도로 관리하여야 한다.
>
> **요구사항**
> 1. `~/review02` 아래에서 **크기가 1MB를 넘는 일반 파일**을 검색한다.
> 2. 해당 파일의 크기를 사람이 읽기 쉬운 단위로 확인한다.
> 3. `bigfile` 디렉터리를 만들어 옮긴다.
> 4. `~/review02` 전체가 차지하는 용량을 확인한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cd ~/review02/inbox
find ~/review02 -type f -size +1M
find ~/review02 -type f -size +1M -exec ls -lh {} +

mkdir -p bigfile
mv sorted/log/camera_dump.log bigfile/
ls -lh bigfile

du -sh ~/review02
du -sh ~/review02/*
```

**해설**

- `-size +1M`의 `+`는 "초과"를 의미한다. `-1M`은 미만, 부호가 없으면 정확히 일치하는 경우이다.
- `-exec ls -lh {} +`의 `+`는 찾은 항목을 **한 번에 모아서** 명령에 전달하므로 처리 속도가 빠르다.
- 검색으로 위치(`sorted/log/camera_dump.log`)를 확인하였으므로, 이동은 **경로를 명시하여** 수행하였다. 문제 3에서 `*.log`를 옮길 때 이 파일도 함께 `sorted/log`로 이동하였기 때문이다.
- `du`는 **특정 디렉터리**가 차지하는 용량을, `df`는 **파일 시스템 전체**의 사용량을 조회한다. 두 명령의 구분은 시험에 자주 출제된다.

> **주의 — `find`의 검색 범위 안에 목적지를 두면 안 된다**
> 다음과 같이 작성하면 오류가 발생한다.
>
> ```bash
> find ~/review02 -type f -size +1M -exec mv {} bigfile/ \;
> ```
>
> `bigfile` 디렉터리가 검색 범위인 `~/review02` **안에 포함**되어 있으므로, `find`가 이미 옮겨진 파일을 다시 찾아내어 자기 자신을 자기 자신에게 옮기려 시도한다. 그 결과 `mv: '...' and '...' are the same file` 오류가 출력된다.
>
> 검색 범위와 목적지가 겹칠 때는 **범위를 좁히거나**(`find sorted ...`), **목적지를 검색 범위 밖에 두어야** 한다. 실무의 일괄 처리 스크립트에서 자주 발생하는 사고이다.
{: .prompt-warning }

</details>

**완료 기준** — `bigfile/camera_dump.log`가 존재하고 크기가 `3.0M`로 표시된다.

---
---

# 문제 6. 로그 분석

---

> **상황**
> 행정실장이 "서버 점검 기록에서 오류가 몇 건이나 발생했는지"를 요구하였다.
>
> **요구사항**
> 1. `sorted/log`의 로그 파일에서 `ERROR`가 포함된 행을 **파일별 건수**로 집계한다.
> 2. `ERROR`가 기록된 위치를 **행 번호와 함께** 출력한다.
> 3. 전체 로그의 **총 오류 건수**를 하나의 숫자로 구한다.
> 4. `ERROR`가 **포함되지 않은** 행만 출력한다.
> 5. 행이 `INFO`로 **시작하는** 행만 출력한다.
> 6. 오류가 한 건이라도 기록된 **파일의 이름만** 출력한다.
{: .prompt-info }

**힌트** — `grep`의 `-c` `-n` `-v` `-l` 옵션과 정규 표현식의 `^`(행의 시작)를 사용한다. 총합은 파이프로 `wc -l`에 연결한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cd ~/review02/inbox/sorted/log

grep -c "ERROR" *.log
grep -n "ERROR" *.log
cat *.log | grep "ERROR" | wc -l
grep -v "ERROR" *.log
grep "^INFO" *.log
grep -l "ERROR" *.log
```

**해설**

- `grep -c`는 **파일별 건수**를, `cat *.log | grep ERROR | wc -l`은 **전체 합계**를 구한다. 두 결과의 의미가 다르다는 점을 구분하여야 한다. 총 오류는 **4건**이다.
- `-v`는 조건을 뒤집어 일치하지 **않는** 행을 출력한다. 정상 기록만 추려 볼 때 사용한다.
- `^INFO`의 `^`는 **행의 시작**을 의미한다. 행 중간에 있는 `INFO`는 제외된다.
- `-l`(소문자 L)은 내용을 출력하지 않고 **파일 이름만** 알려 준다. 대상 파일이 많을 때 먼저 범위를 좁히는 용도로 사용한다.
- 셸의 와일드카드 `*`와 정규 표현식의 `*`는 **서로 다른 체계**이다. 전자는 "임의의 문자열", 후자는 "직전 문자의 반복"을 의미한다.

</details>

**완료 기준** — 총 오류 건수 `4`가 확인되고, 오류가 포함된 파일이 `srv_check_01.log`와 `srv_check_03.log`임을 설명할 수 있다.

---
---

# 문제 7. 백업과 복원 검증

---

> **상황**
> 정리한 자료를 압축하여 보관하되, **복원이 가능한지까지 확인**하여야 한다.
>
> **요구사항**
> 1. `sorted` 디렉터리를 gzip으로 압축한 아카이브를 만들되, 파일 이름에 **오늘 날짜**가 포함되도록 한다.
> 2. 압축하지 않은 아카이브도 만들어 **두 파일의 크기를 비교**한다.
> 3. 아카이브의 내용 목록을 확인한다.
> 4. `verify` 디렉터리를 만들어 아카이브를 풀고, **원본과 파일 개수가 일치하는지** 확인한다.
{: .prompt-info }

**힌트** — `tar`의 `f` 옵션은 항상 **옵션 나열의 마지막**에 두어야 한다. 날짜는 명령 치환 `$(date +%Y%m%d)`으로 삽입한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cd ~/review02/inbox

tar -cf sorted_plain.tar sorted
tar -czf sorted_$(date +%Y%m%d).tar.gz sorted
ls -lh sorted_plain.tar sorted_*.tar.gz

tar -tzvf sorted_$(date +%Y%m%d).tar.gz | head -10
tar -tzf sorted_$(date +%Y%m%d).tar.gz | grep -v '/$' | wc -l

mkdir -p ~/review02/verify
tar -xzf sorted_$(date +%Y%m%d).tar.gz -C ~/review02/verify

find ~/review02/verify -type f | wc -l
find sorted -type f | wc -l
```

**해설**

- `tar`는 **묶기**, `gzip`은 **압축**으로 서로 다른 작업이다. `z` 옵션이 압축을 담당하므로 `.tar.gz` 쪽이 현저히 작다.
- `tar -cvfz ...`와 같이 작성하면 `f`가 바로 뒤의 `z`를 파일 이름으로 해석하여, `tar: 파일명: Cannot stat: No such file or directory` 오류와 함께 **`z`라는 이름의 엉뚱한 파일이 생성된다.** `f`는 반드시 마지막에 둔다.
- 목록에서 `/`로 끝나는 항목은 디렉터리이다. `grep -v '/$'`로 제외하여야 **일반 파일 개수**를 정확히 셀 수 있다.
- 복원본과 원본의 파일 개수가 **같아야** 백업이 유효하다. 백업은 복원이 가능할 때 비로소 백업으로서 의미를 갖는다.

</details>

**완료 기준** — 복원본과 원본의 파일 개수가 동일하고, `.tar.gz`가 `.tar`보다 작다.

---
---

# 문제 8. 처리 보고서 작성

---

> **상황**
> 마지막으로 처리 내역을 문서로 제출하여야 한다.
>
> **요구사항**
> `~/review02/report.txt` 파일에 다음 항목을 순서대로 기록한다.
>
> | 항목 | 내용 |
> |---|---|
> | 머리글 | 보고서 제목과 작성 일시 |
> | [1] | 분류 결과 — `sorted` 아래 디렉터리별 파일 개수 |
> | [2] | 별도 보관 — 오래된 자료와 대용량 자료의 목록 |
> | [3] | 로그 점검 — 총 오류 건수 |
> | [4] | 백업 — 아카이브 파일명과 크기 |
{: .prompt-info }

**힌트** — 여러 명령의 출력을 하나의 파일로 모을 때는 `{ ... } > 파일` 형태가 간결하다. 날짜는 `$(date '+%Y-%m-%d %H:%M')`으로 삽입한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cd ~/review02/inbox

{
  echo "===== 학과 자료실 정비 처리 보고서 ====="
  echo "작성 일시: $(date '+%Y-%m-%d %H:%M')"
  echo
  echo "[1] 분류 결과"
  for d in sorted/*; do
    echo "  $d : $(find "$d" -type f | wc -l) 개"
  done
  echo
  echo "[2] 별도 보관"
  echo "  오래된 자료:"
  ls -1 archive_old | sed 's/^/    - /'
  echo "  대용량 자료:"
  ls -1 bigfile | sed 's/^/    - /'
  echo
  echo "[3] 로그 점검"
  echo "  총 오류 건수: $(cat sorted/log/*.log | grep -c ERROR) 건"
  echo
  echo "[4] 백업"
  ls -lh sorted_*.tar.gz | awk '{print "  " $9 "  (" $5 ")"}'
} > ~/review02/report.txt

cat ~/review02/report.txt
```

**해설**

- `{ ... } > 파일`은 중괄호 안의 모든 출력을 한 번에 파일로 보낸다. `echo` 하나하나에 `>>`를 붙이는 방식보다 간결하고, **`>`와 `>>`를 혼동하여 내용을 지우는 사고**를 예방한다.
- `for d in sorted/*`는 반복 구문이다. 제2강의 범위를 넘어서므로, 어렵다면 `find sorted/csv -type f | wc -l`을 항목마다 나열하는 방식으로 작성하여도 무방하다.
- `sed 's/^/    - /'`는 각 행의 시작에 들여쓰기와 기호를 붙인다. 제4강에서 다시 다룬다.

</details>

**완료 기준** — `report.txt`에 네 항목이 모두 기록되어 있고, 오류 건수가 `4 건`으로 표시된다.

---
---

# 마무리. 자가 채점

---

다음 블록을 실행하면 각 문제의 완료 여부가 자동으로 판정된다.

```bash
cd ~/review02/inbox
echo "===== 자가 채점 결과 ====="

[ -d sorted/csv ] && [ -d sorted/log ] && [ -d sorted/conf ] && [ -d sorted/doc ] \
  && echo "[문제 3] 통과 — 분류 디렉터리 4개 생성" || echo "[문제 3] 미완료"

[ "$(find sorted/hidden -type f 2>/dev/null | wc -l)" -eq 2 ] \
  && echo "[문제 4-1] 통과 — 숨김 파일 2개 이동" || echo "[문제 4-1] 미완료"

[ "$(find archive_old -type f 2>/dev/null | wc -l)" -eq 2 ] \
  && echo "[문제 4-2] 통과 — 오래된 자료 2개 분리" || echo "[문제 4-2] 미완료"

[ -f bigfile/camera_dump.log ] \
  && echo "[문제 5] 통과 — 대용량 파일 분리" || echo "[문제 5] 미완료"

[ "$(cat sorted/log/*.log 2>/dev/null | grep -c ERROR)" -eq 4 ] \
  && echo "[문제 6] 통과 — 오류 4건 집계" || echo "[문제 6] 미완료"

ls sorted_*.tar.gz > /dev/null 2>&1 \
  && echo "[문제 7-1] 통과 — 압축 아카이브 생성" || echo "[문제 7-1] 미완료"

[ "$(find ~/review02/verify -type f 2>/dev/null | wc -l)" -eq "$(find sorted -type f | wc -l)" ] \
  && echo "[문제 7-2] 통과 — 복원 개수 일치" || echo "[문제 7-2] 미완료"

[ -s ~/review02/report.txt ] \
  && echo "[문제 8] 통과 — 보고서 작성" || echo "[문제 8] 미완료"
```

> **채점 스크립트에 사용된 검사 기호**
>
> | 기호 | 의미 |
> |---|---|
> | `[ -d 경로 ]` | 디렉터리가 존재하는가 |
> | `[ -f 경로 ]` | 파일이 존재하는가 |
> | `[ -s 경로 ]` | 파일이 존재하며 **내용이 비어 있지 않은가** |
> | `[ A -eq B ]` | 두 숫자가 같은가 |
> | `&&` / `\|\|` | 앞 명령이 성공했을 때 / 실패했을 때 실행 |
>
> 이 표기들은 제4강 3.5절에서 상세히 다룬다.
{: .prompt-tip }

---

## 이론 점검

**문항 1.** 숨김 파일까지 포함하여 목록을 상세 형식으로 출력하는 명령은?

<details markdown="1">
<summary><b>정답 보기</b></summary>

`ls -la` — `-a`는 **a**ll(전부)의 약어이며, 리눅스에서는 **파일명이 마침표로 시작하면 숨김 파일**이 된다.

</details>

**문항 2.** 와일드카드 `*.txt`로 `.draft_memo.txt`가 지정되지 않는 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

셸의 와일드카드 `*`는 **마침표로 시작하는 이름과 일치하지 않도록** 설계되어 있다. 숨김 파일이 의도치 않게 일괄 처리되는 사고를 방지하기 위함이다.

</details>

**문항 3.** `find . -size +1M`과 `find . -size -1M`의 차이는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

`+`는 **초과**, `-`는 **미만**을 의미한다. 부호를 붙이지 않으면 해당 크기와 정확히 일치하는 경우만 검색된다.

</details>

**문항 4.** `grep -c "ERROR" *.log`와 `cat *.log | grep -c "ERROR"`의 결과가 다른 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

전자는 **파일마다 건수를 따로** 출력하고, 후자는 모든 파일의 내용을 하나로 이어 붙인 뒤 세므로 **전체 합계 하나**만 출력한다.

</details>

**문항 5.** `tar -cvfz backup.tar.gz data`를 실행하면 어떤 일이 발생하는가?

<details markdown="1">
<summary><b>정답 보기</b></summary>

`f` 바로 뒤의 `z`가 아카이브 파일 이름으로 해석되어 **`z`라는 파일이 생성**되고, `backup.tar.gz`는 압축 대상으로 취급되어 `Cannot stat: No such file or directory` 오류가 발생한다. `f`는 **항상 옵션 나열의 마지막**에 두어야 한다.

</details>

---
---

# 정리와 복습 요약

---

## 정리 절차

실습 결과를 보존하려면 이 단계를 생략한다. 삭제하려면 다음을 실행한다.

```bash
rm -rf ~/review02
```

```bash
ls ~ | grep review02 || echo "정리 완료"
```

> 삭제 명령은 되돌릴 수 없다. `rm -rf`를 실행하기 전에는 **경로를 눈으로 다시 확인**하는 습관을 들여야 한다.
{: .prompt-danger }

---

## 본 종합문제에서 사용한 명령

| 구분 | 명령 |
|---|---|
| 이동·조회 | `cd`, `pwd`, `ls -la`, `ls -lhtr`, `ls -R`, `ls -A` |
| 생성·이동 | `mkdir -p`, `touch`, `mv`, `cp`, `rm -rf` |
| 검색 | `find -type`, `-name`, `-size`, `-mtime`, `-exec` |
| 내용 검색 | `grep -c`, `-n`, `-v`, `-l`, `^`, `$` |
| 용량 | `du -sh`, `wc -l` |
| 아카이브 | `tar -czf`, `-tzf`, `-xzf`, `-C` |
| 조합 | 파이프 `\|`, 리다이렉션 `>`, `{ ... } > 파일`, `$( )` |

---

## 자기 점검

다음 항목을 스스로 설명할 수 있으면 제2강을 복습한 것으로 본다.

```
 [ ] 절대 경로와 상대 경로를 구분하고 그 근거를 말할 수 있다.
 [ ] ls -l 출력의 각 항목이 무엇인지 설명할 수 있다.
 [ ] 숨김 파일이 와일드카드로 지정되지 않는 이유를 설명할 수 있다.
 [ ] find로 이름·크기·시각 조건을 조합하여 검색할 수 있다.
 [ ] find와 grep의 용도 차이를 설명할 수 있다.
 [ ] 묶기와 압축이 별개의 작업임을 설명할 수 있다.
 [ ] 파괴적 명령 전에 ls로 대상을 확인하는 습관을 지켰다.
```

다음 복습은 **제3강 종합문제 — 연구소 계정 개편과 보안 점검**이다. 본 과정에서 출제 비중과 실무 중요도가 가장 높은 영역이므로 반드시 수행할 것을 권장한다.
