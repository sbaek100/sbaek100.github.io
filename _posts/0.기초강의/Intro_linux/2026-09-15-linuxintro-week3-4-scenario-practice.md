---
title: 리눅스 기초 3주차-4. 시나리오 종합 실습 — 신입 서버 관리자의 첫 주
date: 2026-09-15 15:00:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - 파일관리
  - 사용자계정
  - 접근권한
  - chmod
  - chown
  - SetGID
  - StickyBit
  - 종합실습
pin:
mermaid: false
---

> **학습 목표**
> 1. 3주차에 학습한 파일 관리·계정 관리·접근 권한 명령을 실제 상황과 유사한 시나리오 속에서 스스로 선택하여 사용할 수 있다.
> 2. 뒤섞인 자료를 와일드카드와 `cp`·`mv`·`rm`·`find`로 정리하고, 링크와 조회 명령으로 자료를 다룰 수 있다.
> 3. 요구사항에 맞추어 계정과 그룹을 발급하고, 등록 정보를 시스템 파일에서 검증할 수 있다.
> 4. 그룹 기반 접근 통제와 특수 권한(SetGID·Sticky Bit)으로 팀 공유 공간을 설계·구현·검증할 수 있다.
> 5. 취약하게 설정된 권한을 진단하여 최소 권한으로 시정하고, `umask`·`chattr`·SetUID 점검으로 시스템을 보호할 수 있다.
{: .prompt-info }

본 강의는 3주차-1(파일과 디렉터리 관리), 3주차-2(사용자 계정과 접근 권한), 3주차-3(접근 권한 시나리오 실습)에서 학습한 내용만으로 구성된 **종합 실습**이다. 새로운 명령어는 등장하지 않으며, 이미 배운 명령어를 **시나리오 속에서 스스로 선택하여 사용하는 능력**을 기르는 데 목적이 있다.

실습 계정은 `student`, 호스트 이름은 `ubuntu-y1`이며, 실습 디렉터리로 `~/lab03/mission`을 사용한다.

> **실습 환경에 관한 주의**
> 본 실습은 실제로 계정을 생성하고 시스템 영역(`/srv`)의 권한을 변경한다. 반드시 **학습용 가상 머신**에서만 수행하고, 시작 전에 스냅샷을 찍어 둔다. 마지막 임무의 정리 절차를 반드시 수행한다.
{: .prompt-danger }

---
---

# 제1절. 실습 시나리오와 진행 방법

---

## 1.1 시나리오

> **실습 시나리오 — 신입 서버 관리자의 첫 주**
>
> 2주차의 실습실 도우미 임무를 무사히 마친 학습자는, 이번 주부터 학과 **프로젝트 서버의 신입 관리자**로 배정되었다. 관리 선생님은 다음 다섯 가지 임무가 적힌 업무 지시서를 건넸다.
>
> 1. **임무 1.** 학생들이 제출한 뒤섞인 자료를 종류별로 정리하고, 로그를 점검하라.
> 2. **임무 2.** 웹 프로젝트팀에 새로 합류한 두 학생의 계정을 발급하라.
> 3. **임무 3.** 웹 프로젝트팀 전용 공유 공간과, 전체 학생용 제출함을 구축하라.
> 4. **임무 4.** 전임자가 급하게 설정해 둔 웹 서비스 디렉터리의 보안을 점검하고 시정하라.
> 5. **임무 5.** 점검 결과를 보고서로 제출하고, 실습 환경을 원상 복구하라.
>
> 관리 선생님은 "관리자는 마우스가 아니라 **터미널 명령으로** 일한다"고 재차 강조하였다.
{: .prompt-info }

---

## 1.2 진행 방법

총 **25문제**이며, 임무별 문제 수는 다음과 같다.

| 임무 | 주제 | 문제 | 관련 강의 |
|---|---|---|---|
| 임무 1 | 자료 정리와 로그 점검 | 1~6 | 3주차-1 |
| 임무 2 | 계정과 그룹 발급 | 7~11 | 3주차-2 제1·2절 |
| 임무 3 | 팀 공유 공간 구축 | 12~16 | 3주차-2 제4절, 3주차-3 시나리오 1 |
| 임무 4 | 보안 점검과 시정 | 17~22 | 3주차-2 제3~6절, 3주차-3 시나리오 2·3 |
| 임무 5 | 보고서 제출과 원상 복구 | 23~25 | 3주차 전체 |

각 문제는 다음 순서로 진행한다.

| 순서 | 내용 |
|---|---|
| ① 상황 | 시나리오 속에서 주어지는 상황 설명 |
| ② 과제 | 수행해야 할 일 |
| ③ 힌트 | 사용할 명령어의 실마리 |
| ④ 풀이 | 정답 명령과 해설, 예상 결과 |

**먼저 힌트까지만 읽고 스스로 명령을 입력해 본 뒤, 풀이를 열어 확인하는 방식을 권장한다.** 막히는 경우에는 풀이의 명령을 그대로 따라 입력해도 충분한 학습 효과가 있다.

> **유의 사항**
> 1. 파괴적인 명령(`rm`, `mv`)을 실행하기 전에는 **같은 와일드카드를 `ls`에 먼저 적용**하여 대상을 확인한다.
> 2. 관리자 권한이 필요한 명령에는 `sudo`를 붙이고, 비밀번호를 물으면 자신의 로그인 비밀번호를 입력한다.
> 3. 다른 사용자의 입장에서 동작을 검증할 때는 `sudo -u 사용자 명령` 형태를 사용한다.
> 4. 실습 계정의 비밀번호는 모두 `Linux#2026`으로 통일한다.
{: .prompt-warning }

---
---

# 제2절. 임무 1 — 제출 자료 정리와 로그 점검

---

> **상황**
> 서버의 제출함 디렉터리에 학생들이 낸 C 소스 파일, 보고서 PDF, 제출 기록 로그, 안내문이 한데 뒤섞여 있다. 관리 선생님은 "종류별로 나누어 보관하고, 제출 기록 로그를 살펴 이상이 없는지 확인하라"고 지시하였다.
{: .prompt-info }

> **준비 명령 — 그대로 입력**
> 뒤섞인 제출함을 재현하기 위하여 다음을 순서대로 입력한다. 이 부분은 문제가 아니다.
>
> ```bash
> mkdir -p ~/lab03/mission/inbox && cd ~/lab03/mission/inbox
> touch hw1_mina.c hw1_jun.c hw2_mina.c hw10_mina.c
> touch report_mina.pdf report_jun.pdf
> seq -f "제출기록 %g번: 정상 접수" 1 30 > submit.log
> echo "과제 제출 안내: 마감은 금요일 18시" > notice.txt
> ls
> ```
>
> `seq`는 준비 단계에서 30행짜리 로그를 만들기 위한 보조 명령이며, 본 실습의 학습 대상은 아니다.
{: .prompt-tip }

---

> ### 문제 1. 분류 보관함 구축
>
> **과제** 제출함과 나란히 `~/lab03/mission/sorted` 디렉터리를 만들고, 그 아래에 `c`, `pdf`, `log` 세 디렉터리를 **한 번의 명령으로** 생성한다. 생성 결과를 하위 디렉터리까지 한꺼번에 확인한다.
>
> **힌트** 상위 디렉터리까지 함께 만드는 옵션과, 여러 이름을 한 번에 나열하는 중괄호 확장을 조합한다. 하위까지 조회하는 `ls` 옵션은 대문자 `R`이다.
{: .prompt-tip }

**풀이.**

```bash
mkdir -p ~/lab03/mission/sorted/{c,pdf,log}
```

```bash
ls -R ~/lab03/mission/sorted
```

```
/home/student/lab03/mission/sorted:
c  log  pdf

/home/student/lab03/mission/sorted/c:

/home/student/lab03/mission/sorted/log:

/home/student/lab03/mission/sorted/pdf:
```

> `-p`는 **p**arents의 약어로 `sorted`가 아직 없어도 함께 생성한다. 중괄호 `{c,pdf,log}`는 셸이 명령 실행 전에 `sorted/c`, `sorted/pdf`, `sorted/log` 세 경로로 확장한다. `-R`은 **R**ecursive의 약어로 하위 디렉터리까지 재귀적으로 출력한다.

---

> ### 문제 2. 와일드카드로 대상 확인 후 분류
>
> **과제** 제출함(`inbox`)에서 ① `mina`의 C 파일만 조회하고, ② 과제 번호가 **한 자리**인 `mina`의 C 파일만 조회하여 `hw10_mina.c`가 빠지는지 관찰한다. ③ 이어서 C 파일 전체를 `sorted/c`로, PDF 파일 전체를 `sorted/pdf`로, 로그 파일을 `sorted/log`로 이동한다. 이동 전에 반드시 `ls`로 대상을 확인한다.
>
> **힌트** 임의의 문자 0개 이상은 `*`, 정확히 한 글자는 `?`이다. 이동 명령은 `mv`이며, 목적지는 `../sorted/…`처럼 상대 경로로 지정할 수 있다.
{: .prompt-tip }

**풀이.**

```bash
cd ~/lab03/mission/inbox
```

```bash
ls *_mina.c
```

```
hw10_mina.c  hw1_mina.c  hw2_mina.c
```

```bash
ls hw?_mina.c
```

```
hw1_mina.c  hw2_mina.c
```

> `?`는 **정확히 한 글자**와 대응하므로 `hw` 뒤에 두 글자(`10`)가 오는 `hw10_mina.c`는 제외된다. `*`는 글자 수 제한이 없어 세 파일 모두와 대응한다.

```bash
ls *.c
```

```bash
mv *.c ../sorted/c/
```

```bash
ls *.pdf
```

```bash
mv *.pdf ../sorted/pdf/
```

```bash
mv *.log ../sorted/log/
```

```bash
ls -R ../sorted
```

> `mv`를 실행하기 전에 동일한 와일드카드를 `ls`에 먼저 적용하여 대상을 확인하는 것이 안전한 작업 습관이다. `..`는 상위 디렉터리(`mission`)이므로 `../sorted/c/`는 `inbox`의 형제 디렉터리 아래를 가리킨다. 이동 후 `inbox`에는 `notice.txt`만 남는다.

---

> ### 문제 3. 제출 기록 로그 점검
>
> **과제** `sorted/log/submit.log`에 대하여 ① 전체 행 수를 계산하고, ② 앞 3행과 뒤 3행을 각각 조회한 뒤, ③ 페이지 단위로 열어 `25번`이 포함된 위치를 검색하고 종료한다.
>
> **힌트** 행 수는 `wc`의 `-l`, 앞·뒤는 `head`와 `tail`에 `-3`을 붙인다. 페이지 단위 조회 명령 안에서는 `/문자열`로 검색하고 `q`로 종료한다.
{: .prompt-tip }

**풀이.**

```bash
cd ~/lab03/mission/sorted/log
```

```bash
wc -l submit.log
```

```
30 submit.log
```

```bash
head -3 submit.log
```

```
제출기록 1번: 정상 접수
제출기록 2번: 정상 접수
제출기록 3번: 정상 접수
```

```bash
tail -3 submit.log
```

```
제출기록 28번: 정상 접수
제출기록 29번: 정상 접수
제출기록 30번: 정상 접수
```

```bash
less submit.log
```

> 화면 안에서 `/25번`을 입력하고 `Enter`를 누르면 해당 행으로 이동한다. `G`로 문서 끝, `g`로 처음으로 이동할 수 있으며, **`q`** 로 종료한다. 짧은 파일은 `cat`, 긴 파일은 `less`, 일부만 볼 때는 `head`·`tail`을 선택하는 것이 조회 명령의 기본 원칙이다.

---

> ### 문제 4. 분류 결과 백업
>
> **과제** 정리된 `sorted` 디렉터리 전체를 `sorted_backup`이라는 이름으로 복사한다. 먼저 옵션 없이 복사를 시도하여 어떤 메시지가 나오는지 관찰한 뒤, 올바른 옵션으로 다시 복사하고 결과를 확인한다.
>
> **힌트** 디렉터리를 복사할 때 필수인 옵션은 **r**ecursive의 약어이다.
{: .prompt-tip }

**풀이.**

```bash
cd ~/lab03/mission
```

```bash
cp sorted sorted_backup
```

```
cp: -r not specified; omitting directory 'sorted'
```

> 디렉터리를 `-r` 없이 복사하면 `omitting directory` 메시지와 함께 건너뛴다. 디렉터리 안의 내용 전체를 다루려면 재귀 옵션이 필요하다.

```bash
cp -r sorted sorted_backup
```

```bash
ls -R sorted_backup
```

> `sorted_backup` 아래에 `c`, `log`, `pdf`와 그 안의 파일이 그대로 복사되어 있으면 성공이다.

---

> ### 문제 5. 이름 변경과 안전한 삭제
>
> **과제** ① 제출함에 남은 `notice.txt`를 `notice_old.txt`로 이름을 바꾼다. ② 이제 비어 있지 않은 `inbox`를 빈 디렉터리 삭제 명령으로 지워 보고 어떤 메시지가 나오는지 관찰한다. ③ `notice_old.txt`를 **확인 질의**를 받으며 삭제한 뒤, 빈 디렉터리가 된 `inbox`를 삭제한다. ④ 문제 4에서 만든 `sorted_backup`은 내용째 삭제한다.
>
> **힌트** 이름 변경은 이동 명령과 같다. 빈 디렉터리 삭제는 `rmdir`, 확인 질의 옵션은 **i**nteractive, 내용째 삭제는 `rm`에 `-r`이다.
{: .prompt-tip }

**풀이.**

```bash
mv inbox/notice.txt inbox/notice_old.txt
```

> `mv`는 같은 디렉터리 안에서 사용하면 이름 변경, 다른 디렉터리를 지정하면 이동이 된다. 두 작업 모두 "디렉터리에 등록된 이름표를 바꾸어 다는 일"이기 때문이다.

```bash
rmdir inbox
```

```
rmdir: failed to remove 'inbox': Directory not empty
```

> `rmdir`은 **빈 디렉터리만** 삭제한다. 의도치 않은 데이터 삭제를 막는 안전장치이다.

```bash
rm -i inbox/notice_old.txt
```

```
rm: remove regular file 'inbox/notice_old.txt'? y
```

```bash
rmdir inbox
```

```bash
ls sorted_backup
```

```bash
rm -r sorted_backup
```

```bash
ls
```

```
sorted
```

> 리눅스 터미널에는 휴지통이 없으므로 `rm`으로 삭제한 파일은 복구할 수 없다. 삭제 전 `ls`로 대상을 확인하는 습관을 유지한다.

---

> ### 문제 6. 파일 검색과 바로가기 만들기
>
> **과제** ① `~/lab03/mission` 아래에서 이름에 `mina`가 들어가는 파일을 모두 찾는다. ② 같은 위치에서 **디렉터리만** 찾는다. ③ 관리 선생님이 자주 확인하는 제출 로그를 `~/lab03/mission/latest.log`라는 **바로가기**로 연결하고, 바로가기를 통해 로그의 앞 2행을 조회한다. ④ `ls -l`로 바로가기의 표시 형태를 확인한다.
>
> **힌트** 검색은 `find 시작경로 조건` 형태이며, 이름 조건은 `-name "패턴"`, 종류 조건은 `-type d`이다. 바로가기(심볼릭 링크)는 `ln`에 **s**ymbolic 옵션을 붙여 `ln -s 원본 링크명`으로 만든다.
{: .prompt-tip }

**풀이.**

```bash
find ~/lab03/mission -name "*mina*"
```

```
/home/student/lab03/mission/sorted/c/hw10_mina.c
/home/student/lab03/mission/sorted/c/hw1_mina.c
/home/student/lab03/mission/sorted/c/hw2_mina.c
/home/student/lab03/mission/sorted/pdf/report_mina.pdf
```

```bash
find ~/lab03/mission -type d
```

> `sorted`와 그 아래 `c`, `pdf`, `log`가 출력된다. `find`는 시작 경로부터 트리를 순회하며 조건에 맞는 항목을 찾는다.

```bash
ln -s ~/lab03/mission/sorted/log/submit.log ~/lab03/mission/latest.log
```

```bash
head -2 ~/lab03/mission/latest.log
```

```
제출기록 1번: 정상 접수
제출기록 2번: 정상 접수
```

```bash
ls -l ~/lab03/mission/latest.log
```

```
lrwxrwxrwx 1 student student 47 ... latest.log -> /home/student/lab03/mission/sorted/log/submit.log
```

> 맨 앞 글자 `l`이 심볼릭 링크임을 나타내고, `링크명 -> 원본경로` 형태로 표시된다. 심볼릭 링크는 원본의 **경로**를 가리키는 바로가기이므로 원본이 삭제되면 링크가 끊어진다.

> **임무 1 완료 기준** `sorted` 아래에 C·PDF·로그 파일이 종류별로 분류되어 있고, `inbox`와 `sorted_backup`이 삭제되었으며, `latest.log`로 로그를 조회할 수 있으면 임무 1 완수이다.
{: .prompt-tip }

---
---

# 제3절. 임무 2 — 계정과 그룹 발급

---

> **상황**
> 웹 프로젝트팀에 `mina`(박미나)와 `jun`(최준) 두 학생이 새로 합류하였다. 관리 선생님은 "두 사람의 계정을 만들고 팀 그룹 `webteam`에 넣되, 나중에 검증할 수 있도록 외부 방문자용 계정 `visitor`도 하나 만들어 두라"고 지시하였다. 계정 발급 전에 자신이 관리자 권한을 가지고 있는지부터 확인해야 한다.
{: .prompt-info }

---

> ### 문제 7. 관리 권한 확인
>
> **과제** ① 자신의 UID와 소속 그룹을 확인하여 일반 사용자 범위(1000 이상)에 있는지, `sudo` 그룹에 속해 있는지 확인한다. ② 자신에게 허용된 `sudo` 명령의 범위를 조회한다. ③ 이 시스템에서 사람이 로그인하여 사용하는 계정(로그인 셸이 `bash`인 계정)이 몇 개인지 계정 목록 파일에서 찾는다.
>
> **힌트** 소속 확인은 두 글자 명령, `sudo` 권한 조회는 `sudo -l`이다. 계정 목록 파일은 `/etc/passwd`이며, 행 끝이 `bash`인 행은 `grep "bash$"`로 고른다.
{: .prompt-tip }

**풀이.**

```bash
id
```

```
uid=1000(student) gid=1000(student) groups=1000(student),4(adm),27(sudo),...
```

> `uid=1000`이므로 일반 사용자 범위이며, `groups`에 `27(sudo)`가 있어 `sudo`를 사용할 수 있다. 리눅스는 이름이 아니라 **UID 숫자**로 사용자를 식별한다.

```bash
sudo -l
```

> 처음 `sudo`를 사용할 때 자신의 비밀번호를 입력한다. `(ALL : ALL) ALL`이 출력되면 모든 명령을 관리자 권한으로 실행할 수 있다는 뜻이다.

```bash
grep "bash$" /etc/passwd
```

```
root:x:0:0:root:/root:/bin/bash
student:x:1000:1000:student,,,:/home/student:/bin/bash
```

> `$`는 행의 끝을 뜻하므로 `bash$`는 "bash로 끝나는 행"이다. 아직 실습 계정을 만들기 전이므로 `root`와 `student` 두 계정만 출력된다.

---

> ### 문제 8. 팀 그룹 생성
>
> **과제** 웹 프로젝트팀 그룹 `webteam`을 생성하고, 그룹 정보 파일에서 생성 결과를 확인한다.
>
> **힌트** 그룹 생성 명령은 group + add이며 관리자 권한이 필요하다. 그룹 정보 파일은 `/etc/group`이다.
{: .prompt-tip }

**풀이.**

```bash
sudo groupadd webteam
```

```bash
grep webteam /etc/group
```

```
webteam:x:1001:
```

> `그룹명:x:GID:구성원목록` 형식이다. 아직 구성원이 없으므로 마지막 자리가 비어 있다. 여러 사용자에게 같은 권한을 줄 때는 사용자마다 설정하지 않고 **그룹**에 권한을 부여한 뒤 사용자를 그룹에 넣는다.

---

> ### 문제 9. 계정 생성과 그룹 배정
>
> **과제** ① `mina` 계정을 홈 디렉터리·로그인 셸(`/bin/bash`)·실명(`Mina Park`)·보조 그룹(`webteam`)을 **모두 지정하여** 한 번에 생성한다. ② `jun` 계정은 실명 `Jun Choi`로 만들되 **그룹 지정을 빠뜨리고** 생성한 뒤, 나중에 그룹을 **추가**하는 명령으로 `webteam`에 넣는다. ③ `visitor` 계정은 홈 디렉터리와 셸만 지정하여 생성한다. ④ 세 계정의 소속을 각각 확인한다.
>
> **힌트** 계정 생성 옵션은 `-m`(홈), `-s`(셸), `-c`(설명), `-G`(보조 그룹)이다. 이미 있는 계정에 그룹을 추가할 때는 `usermod`에 **반드시 `-aG`** 를 붙인다.
{: .prompt-tip }

**풀이.**

```bash
sudo useradd -m -s /bin/bash -c "Mina Park" -G webteam mina
```

```bash
sudo useradd -m -s /bin/bash -c "Jun Choi" jun
```

```bash
sudo usermod -aG webteam jun
```

> `-a`는 **a**ppend의 약어이다. `-a` 없이 `-G webteam`만 쓰면 기존 보조 그룹이 **모두 제거**되고 `webteam`만 남는다. 그룹을 추가할 때는 반드시 `-aG` 형태로 사용한다.

```bash
sudo useradd -m -s /bin/bash visitor
```

```bash
id mina
```

```
uid=1001(mina) gid=1002(mina) groups=1002(mina),1001(webteam)
```

```bash
id jun
```

```
uid=1002(jun) gid=1003(jun) groups=1003(jun),1001(webteam)
```

```bash
id visitor
```

```
uid=1003(visitor) gid=1004(visitor) groups=1004(visitor)
```

> 우분투는 계정을 만들면 동명의 **기본 그룹**을 함께 만든다(`gid=1002(mina)`). `webteam`은 이와 별도의 **보조 그룹**이다. `visitor`는 `webteam`에 속하지 않는다. `-m`을 생략하면 홈 디렉터리가 만들어지지 않아 로그인 후 정상 작업이 어려우므로, `useradd`에는 항상 `-m`을 지정한다.

---

> ### 문제 10. 비밀번호 설정과 홈 디렉터리 확인
>
> **과제** ① 세 계정에 실습용 비밀번호 `Linux#2026`을 설정한다. ② `mina`의 홈 디렉터리가 생성되었는지, 그 권한이 무엇인지 확인하고, 다른 일반 사용자가 그 안을 볼 수 있는지 판단한다.
>
> **힌트** 비밀번호 설정은 `passwd`이며 다른 사용자의 것은 `sudo`가 필요하다. 디렉터리 **자체**의 정보는 `ls -l`에 `-d`를 함께 붙인다.
{: .prompt-tip }

**풀이.**

```bash
sudo passwd mina
```

```bash
sudo passwd jun
```

```bash
sudo passwd visitor
```

> 각각 `Linux#2026`을 두 번 입력한다. 입력 중 화면에 아무것도 표시되지 않는 것은 정상이다. 계정 생성 직후에는 비밀번호가 없어 로그인할 수 없으므로 반드시 설정한다.

```bash
ls -ld /home/mina
```

```
drwxr-x--- 2 mina mina 4096 ... /home/mina
```

> `-d`가 없으면 디렉터리 **안의 목록**이 나오므로, 디렉터리 자체의 권한을 볼 때는 `-d`를 붙인다. 권한 `rwxr-x---`(750)은 소유자 `mina`는 전권, 그룹 `mina`는 읽기·통과, **기타 사용자는 접근 불가**를 뜻한다. 따라서 `jun`이나 `visitor`는 `/home/mina` 안을 볼 수 없다.

---

> ### 문제 11. 등록 정보 검증과 관리자 계정 점검
>
> **과제** ① 계정 목록 파일에서 `mina`의 행을 찾아 일곱 개 항목이 각각 무엇을 뜻하는지 설명한다. ② 그룹 정보 파일에서 `webteam`의 구성원이 두 명으로 늘었는지 확인한다. ③ 이 시스템에 **UID가 0인 계정이 `root` 하나뿐인지** 점검한다.
>
> **힌트** `grep "^mina:"`는 `mina:`로 시작하는 행만 고른다. `/etc/passwd`에서 UID 0인 계정은 두 번째 항목 `x` 다음에 `0`이 오므로 `":x:0:"` 패턴으로 찾을 수 있다.
{: .prompt-tip }

**풀이.**

```bash
grep "^mina:" /etc/passwd
```

```
mina:x:1001:1001:Mina Park:/home/mina:/bin/bash
```

| 항목 | 값 | 의미 |
|---|---|---|
| 1 | `mina` | 계정 이름 |
| 2 | `x` | 비밀번호 자리(실제 값은 `/etc/shadow`에 보관) |
| 3 | `1001` | UID |
| 4 | `1001` | 기본 그룹의 GID |
| 5 | `Mina Park` | 설명(`-c`로 지정한 실명) |
| 6 | `/home/mina` | 홈 디렉터리(`-m`으로 생성) |
| 7 | `/bin/bash` | 로그인 셸(`-s`로 지정) |

> 문제 9의 `useradd` 옵션이 각각 어느 항목에 기록되었는지 대응시켜 이해한다.

```bash
grep webteam /etc/group
```

```
webteam:x:1001:mina,jun
```

> 마지막 항목에 `mina,jun`이 나열되어 보조 그룹 구성원이 두 명임을 확인할 수 있다.

```bash
grep ":x:0:" /etc/passwd
```

```
root:x:0:0:root:/root:/bin/bash
```

> **리눅스는 UID가 0이면 계정 이름과 무관하게 관리자 권한을 부여한다.** 출력이 `root` 한 행뿐이면 정상이다. 만약 다른 이름의 계정이 UID 0으로 등록되어 있다면 침해를 의심하여야 하며, 이 점검은 기본적인 보안 점검 항목이다.

> **임무 2 완료 기준** `id mina`와 `id jun`에 `webteam`이 표시되고, `id visitor`에는 표시되지 않으며, 세 계정 모두 비밀번호가 설정되어 있고, UID 0인 계정이 `root`뿐임을 확인하였으면 임무 2 완수이다.
{: .prompt-tip }

---
---

# 제4절. 임무 3 — 팀 공유 공간과 제출함 구축

---

> **상황**
> 관리 선생님은 두 종류의 공유 공간을 요구하였다.
>
> | 공간 | 경로 | 요구사항 |
> |---|---|---|
> | 팀 작업실 | `/srv/webproj` | ① `webteam` 구성원만 접근·읽기·쓰기 가능 ② 비구성원은 목록 조회조차 불가 ③ 구성원이 만든 파일은 자동으로 `webteam` 그룹 소유가 되어 서로 편집 가능 |
> | 전체 제출함 | `/srv/dropbox` | ④ 모든 사용자가 파일을 넣을 수 있음 ⑤ 자기 파일은 지울 수 있으나 **남의 파일은 지울 수 없음** |
{: .prompt-info }

---

> ### 문제 12. 팀 작업실 디렉터리 생성과 소유권 지정
>
> **과제** `/srv/webproj` 디렉터리를 만들고, 소유자는 `root`, 소유 그룹은 `webteam`으로 지정한 뒤 결과를 확인한다.
>
> **힌트** `/srv`는 시스템 영역이므로 생성과 소유권 변경 모두 `sudo`가 필요하다. 소유자와 그룹을 동시에 바꾸는 명령은 `chown 소유자:그룹 대상`이다.
{: .prompt-tip }

**풀이.**

```bash
sudo mkdir -p /srv/webproj
```

```bash
sudo chown root:webteam /srv/webproj
```

```bash
ls -ld /srv/webproj
```

```
drwxr-xr-x 2 root webteam 4096 ... /srv/webproj
```

> 소유자 `root`, 그룹 `webteam`으로 바뀌었다. `chown`은 **ch**ange **own**er의 약어이며, 일반 사용자가 파일을 타인에게 마음대로 넘길 수 없도록 관리자만 실행할 수 있다. 아직 권한이 기본값 755이므로 요구사항은 충족되지 않았다.

---

> ### 문제 13. 요구사항 ①②③을 만족하는 권한 설계
>
> **과제** 요구사항 ①②③을 **네 자리 숫자 하나**로 설계하여 `/srv/webproj`에 적용하고, 각 자리가 어느 요구사항을 만족시키는지 표로 설명한다. 적용 후 `ls -ld` 출력에서 특수 권한이 표시된 위치를 찾는다.
>
> **힌트** 그룹 소유권 상속은 SetGID(2000)이다. 소유자·그룹은 전권, 기타는 0이다.
{: .prompt-tip }

**풀이.**

```bash
sudo chmod 2770 /srv/webproj
```

```bash
ls -ld /srv/webproj
```

```
drwxrws--- 2 root webteam 4096 ... /srv/webproj
```

| 자리 | 값 | 의미 | 요구사항 |
|---|---|---|---|
| 특수 | **2** | SetGID: 새 파일이 디렉터리의 그룹 `webteam`을 상속 | ③ |
| 소유자 | 7 | root: 읽기·쓰기·통과 | — |
| 그룹 | 7 | webteam: 읽기·쓰기·통과 | ① |
| 기타 | **0** | 그 외 사용자: 목록 조회·통과·쓰기 모두 불가 | ② |

> 그룹 권한 자리의 `x` 위치에 **`s`** 가 표시되면(`rws`) SetGID가 설정된 것이다. SetGID는 실행 파일에 걸면 그룹의 권한으로 실행되게 하고, **디렉터리에 걸면 그 안에서 생성되는 파일이 디렉터리의 그룹을 상속**하게 한다.

---

> ### 문제 14. 팀 작업실 검증
>
> **과제** ① `mina`의 권한으로 `/srv/webproj/plan.txt`를 만들고, 그 파일의 소유 그룹이 무엇인지 확인한다. ② `jun`의 권한으로 그 파일을 읽을 수 있는지, 같은 곳에 `jun_note.txt`를 만들 수 있는지 확인한다. ③ `visitor`의 권한으로 목록 조회를 시도하여 차단되는지 확인한다.
>
> **힌트** 다른 사용자의 권한으로 명령을 실행할 때는 `sudo -u 사용자 명령`을 사용한다.
{: .prompt-tip }

**풀이.**

```bash
sudo -u mina touch /srv/webproj/plan.txt
```

```bash
ls -l /srv/webproj/plan.txt
```

```
-rw-rw-r-- 1 mina webteam 0 ... /srv/webproj/plan.txt
```

> 파일을 만든 사람은 `mina`인데 그룹이 `mina`가 아니라 **`webteam`** 이다. SetGID에 의한 그룹 상속이 동작한 것이다(요구사항 ③).

```bash
sudo -u jun cat /srv/webproj/plan.txt
```

```bash
sudo -u jun touch /srv/webproj/jun_note.txt
```

```bash
ls -l /srv/webproj
```

> `jun`은 `webteam`의 구성원이므로 `mina`의 파일을 읽고, 같은 디렉터리에 새 파일도 만들 수 있다(요구사항 ①). `cat`은 빈 파일이라 아무것도 출력하지 않지만 오류가 없으면 읽기에 성공한 것이다.

```bash
sudo -u visitor ls /srv/webproj
```

```
ls: cannot open directory '/srv/webproj': Permission denied
```

> `visitor`는 `webteam`에 속하지 않으므로 기타(others) 권한 `0`이 적용되어 목록 조회조차 차단된다(요구사항 ②).

---

> ### 문제 15. 전체 제출함 구축과 Sticky Bit 검증
>
> **과제** ① `/srv/dropbox`를 만들고 요구사항 ④⑤를 만족하는 네 자리 권한을 적용한 뒤 `ls -ld`에서 특수 권한 표시를 찾는다. ② `mina`의 권한으로 `mina_hw.txt`를 넣고, `jun`의 권한으로 그 파일을 삭제해 보아 차단되는지 확인한다. ③ `mina` 자신은 그 파일을 지울 수 있는지 확인한다. ④ 시스템에서 같은 방식으로 운영되는 대표 디렉터리 하나를 찾아 권한을 비교한다.
>
> **힌트** 모두에게 쓰기를 허용하되 남의 파일 삭제를 막는 특수 권한은 Sticky Bit(1000)이다. 대표 사례는 모든 사용자가 임시 파일을 두는 디렉터리이다.
{: .prompt-tip }

**풀이.**

```bash
sudo mkdir -p /srv/dropbox
```

```bash
sudo chmod 1777 /srv/dropbox
```

```bash
ls -ld /srv/dropbox
```

```
drwxrwxrwt 2 root root 4096 ... /srv/dropbox
```

> 기타 권한 자리의 `x` 위치에 **`t`** 가 표시되면 Sticky Bit가 설정된 것이다. `777`이므로 누구나 파일을 만들 수 있고(요구사항 ④), Sticky Bit 때문에 **파일 소유자와 root만** 그 파일을 삭제할 수 있다(요구사항 ⑤).

```bash
sudo -u mina touch /srv/dropbox/mina_hw.txt
```

```bash
sudo -u jun rm /srv/dropbox/mina_hw.txt
```

```
rm: cannot remove '/srv/dropbox/mina_hw.txt': Operation not permitted
```

> 디렉터리에 `w` 권한이 있음에도 삭제가 거부되었다. "파일 삭제는 상위 디렉터리의 `w`가 결정한다"는 원칙에 Sticky Bit가 **예외**를 만든 것이다.

```bash
sudo -u mina rm /srv/dropbox/mina_hw.txt
```

```bash
ls /srv/dropbox
```

> 소유자 본인은 삭제할 수 있다. 출력이 비어 있으면 성공이다.

```bash
ls -ld /tmp
```

```
drwxrwxrwt 10 root root 4096 ... /tmp
```

> `/tmp`가 바로 같은 방식(`1777`)으로 운영되는 대표 사례이다. 누구나 임시 파일을 만들 수 있지만 타인의 파일은 지울 수 없다.

---

> ### 문제 16. SetGID를 빠뜨렸을 때의 문제 재현과 시정
>
> **과제** 전임자가 팀 작업실에 SetGID 없이 `770`만 적용했다고 가정하고 그 상태를 재현한다. ① `/srv/webproj`를 `770`으로 바꾼 뒤 `mina`의 권한으로 `design.txt`를 만들고 소유 그룹을 확인한다. ② 이 파일의 그룹만 `webteam`으로 바로잡는다. ③ 근본 원인을 해결하기 위해 디렉터리를 다시 `2770`으로 되돌리고 `s` 표시를 확인한다.
>
> **힌트** 그룹만 바꾸는 명령은 **ch**ange **gr**ou**p**의 약어이며, 다른 사람 소유의 파일이므로 `sudo`가 필요하다.
{: .prompt-tip }

**풀이.**

```bash
sudo chmod 770 /srv/webproj
```

```bash
sudo -u mina touch /srv/webproj/design.txt
```

```bash
ls -l /srv/webproj/design.txt
```

```
-rw-rw-r-- 1 mina mina 0 ... /srv/webproj/design.txt
```

> SetGID가 없으면 파일의 그룹이 만든 사람의 **기본 그룹** `mina`가 된다. 이 상태에서는 `jun`이 그룹 권한(`rw`)이 아니라 기타 권한(`r`)만 적용받아 파일을 **수정할 수 없다.** 팀 협업이 깨지는 원인이다.

```bash
sudo chgrp webteam /srv/webproj/design.txt
```

```bash
ls -l /srv/webproj/design.txt
```

> 그룹이 `webteam`으로 바뀌었다. 그러나 이는 파일 하나를 고친 것일 뿐, 앞으로 만들어질 파일마다 같은 문제가 반복된다.

```bash
sudo chmod 2770 /srv/webproj
```

```bash
ls -ld /srv/webproj
```

> 다시 `drwxrws---`가 되었다. 개별 파일의 시정(`chgrp`)과 근본 원인의 시정(SetGID 복구)을 구분하는 것이 관리자의 판단이다.

> **임무 3 완료 기준** `/srv/webproj`가 `drwxrws---`(root:webteam), `/srv/dropbox`가 `drwxrwxrwt`이고, 구성원의 파일 공유·비구성원 차단·타인 파일 삭제 차단을 모두 확인하였으면 임무 3 완수이다.
{: .prompt-tip }

---
---

# 제5절. 임무 4 — 웹 서비스 디렉터리 보안 점검

---

> **상황**
> 전임자가 급하게 설정해 둔 웹 서비스 디렉터리에서 문제가 있다는 제보가 들어왔다. 관리 선생님은 "권한을 하나하나 진단하여 최소 권한으로 바로잡고, 기본 생성 권한·디렉터리 권한·파일 보호·특수 권한까지 점검하라"고 지시하였다.
{: .prompt-info }

> **준비 명령 — 그대로 입력**
> 취약하게 설정된 웹 서비스 디렉터리를 재현한다. 이 부분은 문제가 아니다.
>
> ```bash
> mkdir -p ~/lab03/mission/webapp/uploads && cd ~/lab03/mission/webapp
> echo "DB_PASS=web#2026" > db.conf
> printf '#!/bin/bash\necho "배포를 시작합니다"\n' > deploy.sh
> cp db.conf db.conf.bak
> echo "업로드 자료" > uploads/photo.txt
> chmod 777 db.conf
> chmod 666 deploy.sh
> chmod 644 db.conf.bak
> chmod 777 uploads
> ```
{: .prompt-tip }

---

> ### 문제 17. 권한 진단
>
> **과제** `webapp` 안의 네 항목(`db.conf`, `deploy.sh`, `db.conf.bak`, `uploads`)의 현재 권한을 조회하고, 각각 **무엇이 왜 문제인지** 표로 정리한다. 배포 스크립트는 실제로 실행해 보아 동작 여부도 확인한다.
>
> **힌트** 상세 조회는 `ls -l`이며, 현재 디렉터리의 스크립트는 `./deploy.sh`로 실행한다.
{: .prompt-tip }

**풀이.**

```bash
cd ~/lab03/mission/webapp
```

```bash
ls -l
```

```
-rwxrwxrwx 1 student student 17 ... db.conf
-rw-r--r-- 1 student student 17 ... db.conf.bak
-rw-rw-rw- 1 student student 40 ... deploy.sh
drwxrwxrwx 2 student student 4096 ... uploads
```

```bash
./deploy.sh
```

```
bash: ./deploy.sh: Permission denied
```

| 항목 | 현재 권한 | 문제점 |
|---|---|---|
| `db.conf` | `-rwxrwxrwx`(777) | 데이터베이스 비밀번호가 담긴 설정 파일을 **누구나 읽고 수정**할 수 있다. 불필요한 실행 권한까지 있다. |
| `deploy.sh` | `-rw-rw-rw-`(666) | 실행 권한이 없어 **동작하지 않고**, 누구나 내용을 바꿀 수 있어 악성 명령이 삽입될 수 있다. |
| `db.conf.bak` | `-rw-r--r--`(644) | 비밀번호 사본인 **백업 파일이 타인에게 읽기 노출**되어 있다. |
| `uploads` | `drwxrwxrwx`(777) | 누구나 파일을 넣고 **남의 파일을 지울 수 있는** 디렉터리이다. |

> 내용이 스크립트이더라도 실행 권한(`x`)이 없으면 실행할 수 없다. `777`처럼 모두에게 전권을 주는 설정은 최소 권한 원칙에 정면으로 어긋난다.

---

> ### 문제 18. 최소 권한으로 시정
>
> **과제** 다음 기준으로 시정하고 결과를 확인한다. ① 설정 파일은 소유자만 읽고 쓸 수 있게 한다. ② 배포 스크립트는 소유자만 읽고·쓰고·실행할 수 있게 한 뒤 실제로 실행한다. ③ 백업 파일은 불필요하므로 삭제한다. ④ 업로드 디렉터리는 소유자만 쓸 수 있고 다른 사용자는 목록 조회와 통과만 가능하게 한다.
>
> **힌트** 숫자 모드로 `600`, `700`, `755`를 사용한다. 기호 모드로 하려면 `u+x`, `go-w` 등을 조합한다.
{: .prompt-tip }

**풀이.**

```bash
chmod 600 db.conf
```

```bash
chmod 700 deploy.sh
```

```bash
./deploy.sh
```

```
배포를 시작합니다
```

```bash
rm db.conf.bak
```

```bash
chmod 755 uploads
```

```bash
ls -l
```

```
-rw------- 1 student student 17 ... db.conf
-rwx------ 1 student student 40 ... deploy.sh
drwxr-xr-x 2 student student 4096 ... uploads
```

> `600`(`rw-------`)은 개인 키·비밀번호 등 민감한 파일의 표준 권한이고, `700`은 개인 전용 실행 파일, `755`는 공개 디렉터리의 표준 권한이다. 비밀번호 사본 같은 불필요한 백업 파일은 권한을 낮추는 것보다 **삭제**하는 편이 더 안전하다.

---

> ### 문제 19. 기본 생성 권한(umask) 검증
>
> **과제** ① 현재 `umask` 값을 확인한다. ② `webapp`에 새 파일 `new.txt`와 새 디렉터리 `newdir`를 만들어 권한이 각각 얼마로 생성되는지 확인하고, 그 숫자가 어떻게 계산되는지 설명한다. ③ 이 터미널에서만 `umask`를 `077`로 바꾼 뒤 `secret.txt`를 만들어 권한을 확인하고, 다시 `022`로 되돌린다.
>
> **힌트** 파일의 기준값은 666, 디렉터리의 기준값은 777이며, 여기서 umask 값을 제거한다.
{: .prompt-tip }

**풀이.**

```bash
umask
```

```
0022
```

```bash
touch new.txt
```

```bash
mkdir newdir
```

```bash
ls -l
```

```
-rw-r--r-- 1 student student 0 ... new.txt
drwxr-xr-x 2 student student 4096 ... newdir
```

| 대상 | 기준값 | umask | 결과 |
|---|---|---|---|
| 파일 `new.txt` | 666 | 022 | **644** (`rw-r--r--`) |
| 디렉터리 `newdir` | 777 | 022 | **755** (`rwxr-xr-x`) |

> 파일은 생성 시점에 실행 권한을 주지 않으므로 기준값이 666이고, 디렉터리는 통과에 실행 권한이 필요하므로 777이다. **"umask 022일 때 새 파일의 권한은 644이지 755가 아니다"** 라는 점이 가장 흔한 혼동이다.

```bash
umask 077
```

```bash
touch secret.txt
```

```bash
ls -l secret.txt
```

```
-rw------- 1 student student 0 ... secret.txt
```

> 666에서 077을 제거하면 600이 된다. 민감한 작업을 할 때 `umask 077`로 두면 만드는 파일마다 자동으로 소유자 전용이 된다. 이 설정은 현재 터미널에만 적용된다.

```bash
umask 022
```

---

> ### 문제 20. 디렉터리의 `r`과 `x`가 허용하는 것
>
> **과제** `visitor`의 입장에서 `uploads` 디렉터리에 접근하는 실험을 한다. ① `visitor`가 홈 디렉터리를 **통과**만 할 수 있도록 기타 사용자에게 `x`만 추가한다. ② `uploads`를 `744`(기타: 읽기만)로 바꾼 뒤 `visitor`로 목록 조회와 파일 내용 조회를 각각 시도한다. ③ `uploads`를 `755`로 바꾼 뒤 같은 두 가지를 다시 시도하고, 결과의 차이를 디렉터리 권한의 의미로 설명한다.
>
> **힌트** 홈 디렉터리는 `~`이며 기호 모드 `o+x`로 통과 권한만 준다. 디렉터리의 `r`은 목록 조회, `x`는 통과(내부 파일 접근)이다.
{: .prompt-tip }

**풀이.**

```bash
chmod o+x ~
```

> 우분투의 홈 디렉터리는 `750`이라 기타 사용자가 통과할 수 없다. `visitor`가 `~/lab03/mission/webapp/uploads`까지 도달하려면 경로상의 모든 디렉터리에 `x`가 있어야 하므로, 홈에 통과 권한만 임시로 준다(실습 후 문제 25에서 되돌린다).

```bash
chmod 744 uploads
```

```bash
sudo -u visitor ls ~/lab03/mission/webapp/uploads
```

```
photo.txt
```

```bash
sudo -u visitor cat ~/lab03/mission/webapp/uploads/photo.txt
```

```
cat: /home/student/lab03/mission/webapp/uploads/photo.txt: Permission denied
```

> 디렉터리의 `r`은 **이름 목록 조회**만 허용한다. 파일 내용에 도달하려면 디렉터리를 **통과**(`x`)해야 하는데 `744`의 기타 권한에는 `x`가 없으므로 실패한다.

```bash
chmod 755 uploads
```

```bash
sudo -u visitor ls ~/lab03/mission/webapp/uploads
```

```bash
sudo -u visitor cat ~/lab03/mission/webapp/uploads/photo.txt
```

```
업로드 자료
```

> 이번에는 성공한다. 도서관에 비유하면 `r`은 장서 목록표 열람, `x`는 서가 진입 권리이다. **목록만 볼 수 있고 서가에 들어갈 수 없다면 책을 꺼낼 수 없다.** 디렉터리에 통상 `755`나 `750`을 주는 이유가 여기에 있다.

---

> ### 문제 21. 삭제 권한의 소재와 파일 잠금
>
> **과제** ① `uploads/photo.txt`를 읽기 전용(`444`)으로 바꾼 뒤 소유자인 자신이 `rm -f`로 삭제해 보고, 삭제되는 이유를 설명한다. ② 설정 파일 `db.conf`를 root조차 지울 수 없도록 확장 속성으로 잠근 뒤, 삭제를 시도하여 결과를 확인한다. ③ 설정된 속성을 조회하고, 다음 정리를 위해 잠금을 해제한다.
>
> **힌트** 파일 삭제 가능 여부는 파일 자신이 아니라 **상위 디렉터리의 `w`** 가 결정한다. 확장 속성은 `chattr`의 `+i`(immutable)로 걸고 `-i`로 풀며, 조회는 `lsattr`이다. 모두 `sudo`가 필요하다.
{: .prompt-tip }

**풀이.**

```bash
chmod 444 uploads/photo.txt
```

```bash
ls -l uploads/photo.txt
```

```
-r--r--r-- 1 student student 13 ... uploads/photo.txt
```

```bash
rm -f uploads/photo.txt
```

```bash
ls uploads
```

> **삭제되었다.** 누구도 쓸 수 없는 읽기 전용 파일임에도 지워진 것은, 삭제가 "파일 내용을 수정하는 일"이 아니라 "디렉터리에 등록된 항목을 제거하는 일"이기 때문이다. `uploads`(755)에 소유자의 `w`가 있으므로 삭제가 가능하다. 파일 권한만 믿고 안심해서는 안 된다.

```bash
sudo chattr +i db.conf
```

```bash
rm -f db.conf
```

```
rm: cannot remove 'db.conf': Operation not permitted
```

```bash
sudo rm -f db.conf
```

```
rm: cannot remove 'db.conf': Operation not permitted
```

> `+i`(immutable, 변경 불가) 속성이 걸린 파일은 **root라도** 수정·삭제·이름 변경을 할 수 없다. 상위 디렉터리에 `w`가 있어도 마찬가지이다. 악성코드나 실수로부터 핵심 설정 파일을 지키는 강력한 수단이다.

```bash
lsattr db.conf
```

```
----i---------e------- db.conf
```

```bash
sudo chattr -i db.conf
```

> `i`가 표시되면 속성이 설정된 것이다. 정리 단계에서 삭제할 수 있도록 반드시 `-i`로 해제해 둔다. 해제하지 않으면 문제 25의 `rm -rf`가 실패한다.

---

> ### 문제 22. SetUID 점검
>
> **과제** ① 시스템의 `/usr/bin` 아래에서 SetUID가 설정된 파일 목록을 조회하고, 그중 `passwd`의 권한 표시에서 특수 권한 문자를 찾는다. ② 점검 원리를 이해하기 위해 `webapp`에 `cat`의 복사본 `mycat`을 만들고 SetUID를 부여하여 권한 표시를 확인한다. ③ 같은 점검 명령을 `~/lab03` 아래에 적용하여 방금 만든 `mycat`이 **검출되는지** 확인한다. ④ 불필요한 SetUID를 제거한다.
>
> **힌트** SetUID 검색 조건은 `-perm -4000`이다. 부여는 `chmod u+s`, 제거는 `chmod u-s`이다.
{: .prompt-tip }

**풀이.**

```bash
find /usr/bin -perm -4000 -type f
```

```
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/su
...
```

```bash
ls -l /usr/bin/passwd
```

```
-rwsr-xr-x 1 root root 59976 ... /usr/bin/passwd
```

> 소유자 권한 자리의 `x` 위치에 **`s`** 가 있다. `passwd`는 root 소유이므로, 일반 사용자가 실행하면 **일시적으로 root 권한을 빌려** `/etc/shadow`에 자신의 비밀번호를 기록할 수 있다. 목록의 명령들은 모두 정당한 이유로 SetUID가 필요한 것들이다.

```bash
cp /bin/cat mycat
```

```bash
chmod u+s mycat
```

```bash
ls -l mycat
```

```
-rwsr-xr-x 1 student student 35288 ... mycat
```

> 소유자가 `student`인 복사본이므로 실제 위험은 없다. 그러나 만약 이 파일의 소유자가 root였다면, 누구나 `mycat`으로 root만 읽을 수 있는 파일을 열람할 수 있게 된다. **불필요한 SetUID가 권한 상승의 통로가 되는 원리**이다.

```bash
find ~/lab03 -perm -4000 -type f
```

```
/home/student/lab03/mission/webapp/mycat
```

> 점검 명령이 정체불명의 SetUID 파일을 정확히 검출하였다. 실제 점검에서는 시스템 전체(`/`)를 대상으로 하며, 목록에 낯선 파일이 있으면 침해를 의심한다.

```bash
chmod u-s mycat
```

```bash
ls -l mycat
```

```
-rwxr-xr-x 1 student student 35288 ... mycat
```

> `s`가 사라지고 일반 실행 파일로 돌아왔다. 점검 중 발견한 불필요한 SetUID는 이처럼 `u-s`로 제거하거나 파일 자체를 삭제한다.

> **임무 4 완료 기준** `db.conf` 600, `deploy.sh` 700, `uploads` 755로 시정하였고, umask 022에서 파일 644·디렉터리 755가 생성됨을 확인하였으며, 디렉터리 `r`/`x`의 차이·삭제 권한의 소재·`chattr +i`의 효과·SetUID 검출을 모두 실증하였으면 임무 4 완수이다.
{: .prompt-tip }

---
---

# 제6절. 임무 5 — 보고서 제출과 원상 복구

---

> **상황**
> 관리 선생님은 "점검 결과를 `report.txt` 한 개에 정리하여 전체 제출함에 넣되, 다른 학생이 보고서 내용을 볼 수 없게 하라. 그리고 실습에 쓴 계정과 디렉터리는 남김없이 정리하라"고 지시하였다.
{: .prompt-info }

---

> ### 문제 23. 점검 보고서 작성
>
> **과제** `~/lab03/mission/report.txt`를 새로 만들어 다음 다섯 항목을 구분선과 함께 순서대로 기록하고, 완성된 보고서를 화면에 출력한다.
>
> ① 제목 `===== 3주차 서버 점검 보고서 =====`와 작성 일시 ② `[팀 계정]` — `mina`, `jun`의 소속 정보 ③ `[공유 공간]` — `/srv/webproj`와 `/srv/dropbox` 자체의 상세 정보 ④ `[webapp 권한]` — `webapp` 디렉터리의 상세 목록 ⑤ `[SetUID 파일 수]` — `/usr/bin` 아래 SetUID 파일의 개수
>
> **힌트** 첫 행만 `>`, 나머지는 모두 `>>`이다. 일시는 `date`, 소속은 `id`, 디렉터리 자체 정보는 `ls -ld`, 개수는 `find … | wc -l`이다.
{: .prompt-tip }

**풀이.**

```bash
cd ~/lab03/mission
```

```bash
echo "===== 3주차 서버 점검 보고서 =====" > report.txt
```

```bash
date >> report.txt
```

```bash
echo "" >> report.txt
```

```bash
echo "[팀 계정]" >> report.txt
```

```bash
id mina >> report.txt
```

```bash
id jun >> report.txt
```

```bash
echo "" >> report.txt
```

```bash
echo "[공유 공간]" >> report.txt
```

```bash
ls -ld /srv/webproj /srv/dropbox >> report.txt
```

```bash
echo "" >> report.txt
```

```bash
echo "[webapp 권한]" >> report.txt
```

```bash
ls -l webapp >> report.txt
```

```bash
echo "" >> report.txt
```

```bash
echo "[SetUID 파일 수]" >> report.txt
```

```bash
find /usr/bin -perm -4000 -type f | wc -l >> report.txt
```

```bash
cat report.txt
```

```
===== 3주차 서버 점검 보고서 =====
2026. 09. 15. (월) 15:42:10 KST

[팀 계정]
uid=1001(mina) gid=1002(mina) groups=1002(mina),1001(webteam)
uid=1002(jun) gid=1003(jun) groups=1003(jun),1001(webteam)

[공유 공간]
drwxrwxrwt 2 root root    4096 ... /srv/dropbox
drwxrws--- 2 root webteam 4096 ... /srv/webproj

[webapp 권한]
-rw------- 1 student student   17 ... db.conf
-rwx------ 1 student student   40 ... deploy.sh
-rwxr-xr-x 1 student student 35288 ... mycat
-rw-r--r-- 1 student student    0 ... new.txt
drwxr-xr-x 2 student student 4096 ... newdir
-rw------- 1 student student    0 ... secret.txt
drwxr-xr-x 2 student student 4096 ... uploads

[SetUID 파일 수]
12
```

> 첫 행에서만 `>`로 파일을 새로 만들고, 이후에는 `>>`로 이어 붙인다. 중간에 `>`를 쓰면 앞의 내용이 모두 사라진다. SetUID 파일 수는 시스템에 따라 다를 수 있다.

---

> ### 문제 24. 보고서 제출과 열람 제한
>
> **과제** ① 보고서를 전체 제출함에 `report_student.txt`라는 이름으로 복사한다. ② 제출한 사본을 **소유자만 읽을 수 있게** 제한한다. ③ `visitor`의 권한으로 제출함 목록 조회, 사본 내용 조회, 사본 삭제를 각각 시도하여 어디까지 허용되는지 확인한다.
>
> **힌트** 복사 시 목적지에 새 이름을 함께 적으면 이름을 바꾸어 복사된다. 열람 제한은 `600`이다. 제출함은 `1777`이므로 목록은 보이지만 삭제와 열람은 각각 다른 이유로 막힌다.
{: .prompt-tip }

**풀이.**

```bash
cp report.txt /srv/dropbox/report_student.txt
```

```bash
chmod 600 /srv/dropbox/report_student.txt
```

```bash
sudo -u visitor ls -l /srv/dropbox
```

```
-rw------- 1 student student 812 ... report_student.txt
```

> 제출함은 기타 사용자에게 `r`·`x`가 있으므로 목록 조회는 허용된다.

```bash
sudo -u visitor cat /srv/dropbox/report_student.txt
```

```
cat: /srv/dropbox/report_student.txt: Permission denied
```

> 파일 권한이 `600`이므로 소유자가 아닌 `visitor`는 내용을 읽을 수 없다. 이것은 **파일 자신의 권한**이 막은 것이다.

```bash
sudo -u visitor rm /srv/dropbox/report_student.txt
```

```
rm: cannot remove '/srv/dropbox/report_student.txt': Operation not permitted
```

> 제출함에 `w`가 있음에도 삭제가 거부된 것은 **Sticky Bit** 때문이다. 열람은 파일 권한이, 삭제는 디렉터리의 Sticky Bit가 각각 막고 있음을 구분하여 이해한다.

---

> ### 문제 25. 원상 복구
>
> **과제** 다음 순서로 실습 환경을 정리하고, 정리 결과를 검증한다. ① 문제 20에서 변경한 홈 디렉터리 권한을 `750`으로 되돌린다. ② 보고서 원본을 홈 디렉터리로 옮겨 보관한다. ③ `/srv` 아래의 두 공유 공간을 내용째 삭제한다. ④ 실습 계정 세 개를 홈 디렉터리까지 삭제하고, 그룹을 삭제한다. ⑤ `~/lab03/mission`을 내용째 삭제한다. ⑥ 계정과 그룹이 사라졌는지 확인한다.
>
> **힌트** 시스템 영역과 계정 삭제에는 `sudo`가 필요하다. 계정 삭제 시 홈까지 지우는 옵션은 `-r`이며, `userdel`은 한 번에 한 계정만 받는다. 존재하지 않는 계정을 `id`로 조회하면 `no such user`가 나온다.
{: .prompt-tip }

**풀이.**

```bash
chmod 750 ~
```

```bash
mv ~/lab03/mission/report.txt ~/report_week3.txt
```

```bash
sudo rm -rf /srv/webproj /srv/dropbox
```

```bash
sudo userdel -r mina
```

```bash
sudo userdel -r jun
```

```bash
sudo userdel -r visitor
```

```bash
sudo groupdel webteam
```

```bash
rm -rf ~/lab03/mission
```

> 문제 21에서 `chattr -i`로 잠금을 해제해 두었기 때문에 `db.conf`를 포함한 전체가 삭제된다. 만약 `Operation not permitted`가 나온다면 `sudo chattr -i ~/lab03/mission/webapp/db.conf`를 먼저 실행한다.

```bash
id mina
```

```
id: 'mina': no such user
```

```bash
grep webteam /etc/group
```

> 아무것도 출력되지 않으면 그룹이 삭제된 것이다.

```bash
ls -ld ~ /srv
```

> 홈 디렉터리가 `drwxr-x---`로 돌아왔고, `/srv`에 실습 디렉터리가 남아 있지 않으면 정리가 끝난 것이다.

> **임무 5 완료 기준** `~/report_week3.txt`에 다섯 항목이 순서대로 기록되어 있고, 제출함에서 열람과 삭제가 각각 다른 이유로 차단됨을 확인하였으며, 계정·그룹·`/srv` 디렉터리·`mission`이 모두 정리되어 있으면 전체 임무 완수이다.
{: .prompt-tip }

---
---

# 제7절. 자주 발생하는 오류와 대응 방법

---

| 화면에 출력된 메시지 | 원인 | 대응 방법 |
|---|---|---|
| `omitting directory` | `cp`에 `-r` 누락 | `cp -r`로 다시 실행한다. |
| `Directory not empty` | `rmdir`로 비어 있지 않은 디렉터리 삭제 시도 | 내용을 먼저 지우거나 `rm -r`을 사용한다. |
| `useradd: user 'mina' already exists` | 이전 실습 계정이 남아 있음 | `sudo userdel -r mina` 후 다시 생성한다. |
| `usermod` 후 `id`에 그룹이 없음 | `-a` 누락으로 기존 그룹이 대체됨 | `sudo usermod -aG webteam 사용자`로 다시 추가한다. |
| SetGID 후에도 그룹이 상속되지 않음 | `2770`이 아닌 `770` 적용 | `ls -ld`로 그룹 자리에 `s`가 있는지 확인한다. |
| `sudo -u visitor cat`이 `Permission denied` | 경로상의 디렉터리에 `x` 없음 | 홈 디렉터리에 `chmod o+x ~`가 적용되었는지, 대상 디렉터리가 `755`인지 확인한다. |
| `rm -rf`가 `Operation not permitted` | `chattr +i`가 남아 있음 | `sudo chattr -i 파일`로 해제한 뒤 삭제한다. |
| `Operation not permitted` (제출함) | Sticky Bit로 타인 파일 삭제 차단 | 정상 동작이다. 소유자 또는 root만 삭제할 수 있다. |
| `userdel: user mina is currently used by process` | 해당 사용자로 실행 중인 프로세스가 있음 | 터미널을 새로 열거나 잠시 후 다시 시도한다. |

---
---
