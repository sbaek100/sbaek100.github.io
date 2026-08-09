---
title: 리눅스 기초 3강 종합문제 - 연구소 계정 개편과 보안 점검 ★핵심
date: 2026-09-08 09:30:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - 종합문제
  - 복습
  - 사용자계정
  - chmod
  - chage
  - umask
  - SetGID
  - StickyBit
  - 보안점검
pin:
mermaid: false
---

> **종합문제 안내**
> 1. 본 글은 **제3강. 사용자 계정 관리와 접근 권한**의 복습용 종합문제이다. 본 과정에서 **출제 비중과 실무 중요도가 가장 높은 영역**이므로 반드시 완주할 것을 권장한다.
> 2. 제3강의 실습 결과에 의존하지 않는다. 필요한 계정·그룹·디렉터리를 여기서 직접 생성하므로 **독립적으로 수행**할 수 있다.
> 3. 각 문제는 먼저 스스로 작성해 본 뒤 접혀 있는 정답을 펼쳐 확인한다.
> 4. **문제 7은 의도적으로 취약한 설정을 만들어 두고 이를 찾아내는 실습**이다. 반드시 마지막 정리 절차까지 수행하여야 한다.
{: .prompt-info }

> **실습 환경에 관한 경고**
> 본 종합문제는 계정을 생성·삭제하고 파일 권한을 변경하며, **일시적으로 취약한 설정을 재현**한다. 반드시 제0강에서 구축한 **학습용 가상 머신에서만** 수행하여야 하며, 운영 중인 서버에서 실행해서는 안 된다.
>
> 취약 설정 재현은 "공격 방법"을 익히기 위한 것이 아니라, **관리자가 무엇을 점검해야 하는지**를 체득하기 위한 것이다. 각 항목은 탐지 후 즉시 시정한다.
{: .prompt-danger }

| 문제 | 주제 | 대응 절 | 배정 시간 |
|---|---|---|---|
| 준비 | 실습 환경 구성 | — | 5분 |
| 문제 1 | 계정과 그룹 생성 | 3.1 ~ 3.4 | 15분 |
| 문제 2 | 비밀번호 정책 | 3.5 | 10분 |
| 문제 3 | 팀 전용 디렉터리(SetGID) | 5.4 · 6.1 | 15분 |
| 문제 4 | 공용 자료실(스티키 비트) | 6.1 | 10분 |
| 문제 5 | umask 계산 검증 | 5.6 | 10분 |
| 문제 6 | 접근 장애 진단 | 5.2 | 15분 |
| 문제 7 | 보안 점검 5종 | 1.2 · 2.1 · 6.2 | 25분 |
| 문제 8 | sudo 최소 권한 위임 | 4.2 | 10분 |
| 문제 9 | 점검 보고서 작성 | 5.1 ~ 6.3 | 10분 |
| 마무리 | 자가 채점과 정리 | — | 15분 |

---
---

# 시나리오

---

학습자는 중견 제조기업 **한빛정밀**의 신임 시스템 담당자이다. 연구소 조직 개편에 따라 서버의 계정 체계를 새로 구성하는 업무를 맡았다.

전임 담당자는 인수인계 문서를 남기지 않고 퇴사하였으며, 정보보호 담당 부서는 다음과 같이 요구하였다.

> *"연구소 개편에 맞추어 계정과 공유 공간을 새로 구성해 주십시오. 요구사항은 다섯 가지입니다.*
> *첫째, 연구팀과 품질팀의 자료는 서로 열람할 수 없어야 합니다.*
> *둘째, 같은 팀끼리는 상대의 파일을 수정할 수 있어야 합니다.*
> *셋째, 공용 자료실은 누구나 자료를 올릴 수 있되 남의 자료를 지울 수는 없어야 합니다.*
> *넷째, 모든 계정에 90일 비밀번호 변경 정책을 적용해 주십시오.*
> *다섯째, 전임자가 남긴 설정 중 위험한 항목이 있는지 함께 점검하여 보고해 주십시오."*

다섯 번째 요구사항이 본 종합문제의 핵심이다. **전임자가 남긴 것으로 가정한 취약 설정을 준비 단계에서 재현**해 두고, 학습자가 이를 스스로 찾아내어 시정한다.

---
---

# 준비 단계. 실습 환경 구성

---

아래 블록을 **한 번에 복사하여 실행**한다. 작업 디렉터리와 함께 "전임자가 남긴 설정"이 생성된다.

```bash
mkdir -p ~/review03 && cd ~/review03
sudo mkdir -p /srv/hanbit/legacy

# (1) 전임자가 남긴 설정 파일 — 권한이 과도하게 열려 있다
echo "DB_HOST=10.0.0.11" | sudo tee /srv/hanbit/legacy/app.conf > /dev/null
sudo chmod 666 /srv/hanbit/legacy/app.conf

# (2) 전임자가 만들어 둔 관리 도구 — SetUID가 설정되어 있다
sudo cp /usr/bin/find /srv/hanbit/legacy/oldtool
sudo chmod 4755 /srv/hanbit/legacy/oldtool

# (3) 전임자가 임시로 만든 관리용 계정
sudo useradd -o -u 0 -g 0 -M -s /bin/bash svcbackup

# (4) 서비스 전용 계정인데 로그인 셸이 부여되어 있다
sudo useradd -r -M -s /bin/bash legacysvc

# (5) 인수인계용으로 만든 계정 — 비밀번호가 비어 있다
sudo useradd -m -s /bin/bash tempadmin
sudo passwd -d tempadmin

echo "== 준비 완료 =="
ls -l /srv/hanbit/legacy
```

> **각 항목이 무엇을 흉내 낸 것인지는 문제 7에서 밝힌다.** 지금은 내용을 분석하지 말고 다음 문제로 진행한다.
>
> `sudo tee 파일 > /dev/null`은 관리자 권한으로 파일에 내용을 기록하는 표준적인 방법이다. `sudo echo "..." > 파일` 형태는 리다이렉션을 **셸이 먼저 처리**하므로 권한 부족으로 실패한다는 점을 함께 기억한다.

---
---

# 문제 1. 계정과 그룹 생성

---

> **상황**
> 연구팀 2명과 품질팀 1명의 계정을 만들어야 한다.
>
> **요구사항**
>
> | 계정 | 설명(GECOS) | 기본 셸 | 홈 디렉터리 | 보조 그룹 |
> |---|---|---|---|---|
> | `rnd1` | R&D Engineer 1 | `/bin/bash` | 생성 | `rndteam` |
> | `rnd2` | R&D Engineer 2 | `/bin/bash` | 생성 | `rndteam` |
> | `qa1` | QA Engineer 1 | `/bin/bash` | 생성 | `qateam` |
>
> 1. 그룹 `rndteam`과 `qateam`을 먼저 생성한다.
> 2. 위 표대로 계정 3개를 생성한다.
> 3. 세 계정의 비밀번호를 `Hanbit#2026`으로 설정한다.
> 4. 각 계정의 UID와 소속 그룹을 확인한다.
> 5. `rnd2`를 `qateam`에도 **기존 소속을 유지한 채** 추가한다.
{: .prompt-info }

**힌트** — `useradd`에는 `-m`을 반드시 지정한다. 보조 그룹 추가는 `usermod -aG`이며 `-a`를 빠뜨리면 기존 그룹에서 제외된다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sudo groupadd rndteam
sudo groupadd qateam

sudo useradd -m -s /bin/bash -c "R&D Engineer 1" -G rndteam rnd1
sudo useradd -m -s /bin/bash -c "R&D Engineer 2" -G rndteam rnd2
sudo useradd -m -s /bin/bash -c "QA Engineer 1" -G qateam qa1

sudo passwd rnd1
sudo passwd rnd2
sudo passwd qa1

grep -E "^(rnd1|rnd2|qa1):" /etc/passwd
id rnd1
id qa1
getent group rndteam qateam

sudo usermod -aG qateam rnd2
id rnd2
```

**해설**

- **`-m`(make home)** 을 생략하면 홈 디렉터리가 생성되지 않아 로그인 시 작업 위치가 `/`가 되고, `/etc/skel`의 초기 설정 파일도 복사되지 않는다.
- `-s`를 생략하면 `/etc/default/useradd`의 기본값인 **`/bin/sh`** 가 적용된다.
- `-G`는 **보조 그룹**, `-g`는 **기본 그룹**이다. 두 옵션의 대소문자 차이가 의미의 차이이다.
- `usermod -aG`에서 **`-a`를 빠뜨리면 기존 보조 그룹이 모두 제거**된다. 관리자 계정에 실행하면 `sudo` 그룹에서 빠져 관리 권한을 잃는다.
- 우분투는 UPG(User Private Group) 정책을 사용하므로, 계정을 만들면 **같은 이름의 그룹이 함께 생성되어 기본 그룹**이 된다. `id rnd1`의 `gid=`가 이에 해당한다.
- `/etc/group`의 네 번째 필드에는 **보조 그룹 구성원만** 나열된다. 기본 그룹으로 소속된 사용자는 표시되지 않으므로, 전체 소속은 반드시 **`id`** 로 확인한다.

</details>

**완료 기준** — `id rnd2`의 `groups=` 항목에 `rnd2`, `rndteam`, `qateam`이 모두 표시된다.

---
---

# 문제 2. 비밀번호 정책

---

> **상황**
> 정보보호 부서의 네 번째 요구사항을 적용하여야 한다.
>
> **요구사항**
> 1. 세 계정 모두에 **최대 사용 90일**, **최소 사용 7일**, **만료 14일 전 경고**를 설정한다.
> 2. `qa1`은 계약직이므로 **2026년 12월 31일에 계정이 만료**되도록 설정한다.
> 3. `rnd1`은 임시 비밀번호를 발급한 상태이므로 **다음 로그인 시 반드시 비밀번호를 변경**하도록 강제한다.
> 4. 세 계정의 정책을 조회하여 적용 결과를 확인한다.
{: .prompt-info }

**힌트** — `chage`의 옵션은 대소문자를 구분한다. `-M`은 최대, `-m`은 최소, `-W`는 경고, `-E`는 계정 만료일, `-d 0`은 즉시 만료이다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sudo chage -M 90 -m 7 -W 14 rnd1
sudo chage -M 90 -m 7 -W 14 rnd2
sudo chage -M 90 -m 7 -W 14 qa1

sudo chage -E 2026-12-31 qa1
sudo chage -d 0 rnd1

sudo chage -l rnd1
sudo chage -l qa1 | grep -i "expires"
```

**해설**

- `chage`는 **ch**ange **age**(기간을 변경하다)의 약어이며, 실제 값은 `/etc/shadow`의 4~8번 필드에 기록된다.
- `-m 7`(최소 사용 기간)은 사용자가 정책을 우회하기 위해 **비밀번호를 연속으로 바꾸어 원래 값으로 되돌리는 행위**를 막는 장치이다.
- `-E`는 **계정 자체의 만료일**이고 `-M`은 **비밀번호의 유효 기간**이다. 계약 종료가 정해진 인원에게는 `-E`를 함께 적용하여 회수 누락을 방지한다.
- `-d 0`은 최종 변경일을 1970-01-01로 지정하여 즉시 만료 상태로 만든다. `chage -l` 결과에 `password must be changed`가 표시된다. 임시 비밀번호 발급 시의 표준 조치이다.

</details>

**완료 기준** — `chage -l rnd1`에 `Maximum number of days between password change` 값이 `90`으로 표시된다.

---
---

# 문제 3. 팀 전용 디렉터리 — SetGID

---

> **상황**
> 첫 번째와 두 번째 요구사항("팀 간 분리", "같은 팀끼리는 수정 가능")을 구현하여야 한다.
>
> **요구사항**
> 1. `/srv/hanbit/rnd_area`를 만들고 소유를 `root:rndteam`으로 지정한다.
> 2. 연구팀만 접근할 수 있고 **다른 사용자는 목록조차 볼 수 없도록** 권한을 설정한다.
> 3. 이 디렉터리 안에 생성되는 파일이 **자동으로 `rndteam` 그룹을 갖도록** 설정한다.
> 4. `qa_area`도 동일한 방식으로 `qateam` 전용으로 구성한다.
> 5. 다음 세 가지를 실제로 검증한다.
>    - `rnd1`이 `rnd_area`에 파일을 만들 수 있는가
>    - 그 파일의 **소유 그룹이 `rndteam`인가**
>    - `qa1`이 `rnd_area`에 접근할 수 **없는가**
{: .prompt-info }

**힌트** — 특수 권한을 포함한 네 자리 8진수를 사용한다. SetGID는 `2000`이다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sudo mkdir -p /srv/hanbit/rnd_area /srv/hanbit/qa_area

sudo chown root:rndteam /srv/hanbit/rnd_area
sudo chown root:qateam  /srv/hanbit/qa_area

sudo chmod 2770 /srv/hanbit/rnd_area
sudo chmod 2770 /srv/hanbit/qa_area

ls -ld /srv/hanbit/rnd_area /srv/hanbit/qa_area

sudo -u rnd1 touch /srv/hanbit/rnd_area/design_v1.txt
ls -l /srv/hanbit/rnd_area

sudo -u qa1 ls /srv/hanbit/rnd_area

sudo chmod g+w /srv/hanbit/rnd_area/design_v1.txt
sudo -u rnd2 sh -c 'echo "# rnd2 검토 의견" >> /srv/hanbit/rnd_area/design_v1.txt'
cat /srv/hanbit/rnd_area/design_v1.txt
```

**해설**

- `2770`의 각 자리는 다음을 의미한다.
>
> | 자리 | 값 | 의미 |
> |---|---|---|
> | 특수 | 2 | **SetGID** — 내부에 생성되는 항목이 디렉터리의 그룹을 상속 |
> | 소유자 | 7 | root : 읽기·쓰기·실행 |
> | 그룹 | 7 | 팀 구성원 : 읽기·쓰기·실행 |
> | 기타 | 0 | 그 외 : 접근 불가 |
>
- SetGID가 없으면 `rnd1`이 만든 파일의 소유 그룹은 **개인 그룹 `rnd1`** 이 되어, 같은 팀의 `rnd2`가 수정할 수 없다. `ls -ld` 결과의 `drwxrws---`에서 **그룹 실행 자리의 `s`** 가 SetGID의 표시이다.
- `qa1`의 접근 시도는 기타 권한이 `0`이므로 `Permission denied`로 차단된다. 이것이 요구사항 2("팀 간 분리")의 구현이다.
- 파일에 `g+w`를 부여하여야 같은 팀원이 **내용을 수정**할 수 있다. 그룹 상속은 "그룹이 무엇인가"를 정할 뿐, 쓰기 권한까지 자동으로 부여하지는 않는다. 이 구분이 중요하다.

</details>

**완료 기준** — `design_v1.txt`의 소유 그룹이 `rndteam`이고, `qa1`의 접근이 차단되며, `rnd2`가 추가한 행이 파일에 기록되어 있다.

---
---

# 문제 4. 공용 자료실 — 스티키 비트

---

> **상황**
> 세 번째 요구사항("누구나 올릴 수 있되 남의 자료는 지울 수 없다")을 구현하여야 한다.
>
> **요구사항**
> 1. `/srv/hanbit/share`를 만들고 **모든 사용자가 쓸 수 있도록** 설정한다.
> 2. **먼저 스티키 비트 없이** 구성한 뒤, `qa1`이 `rnd1`의 파일을 삭제할 수 있음을 확인한다(문제 상황 재현).
> 3. 스티키 비트를 설정한 뒤 동일한 시도가 **차단되는지** 확인한다.
> 4. 자신의 파일은 여전히 삭제할 수 있음을 확인한다.
> 5. 같은 설정을 사용하는 시스템 디렉터리를 찾아 권한을 비교한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sudo mkdir -p /srv/hanbit/share
sudo chmod 777 /srv/hanbit/share

sudo -u rnd1 sh -c 'echo "연구 자료" > /srv/hanbit/share/rnd1_doc.txt'
sudo -u qa1 rm -f /srv/hanbit/share/rnd1_doc.txt
ls -l /srv/hanbit/share

sudo chmod 1777 /srv/hanbit/share
ls -ld /srv/hanbit/share

sudo -u rnd1 sh -c 'echo "연구 자료" > /srv/hanbit/share/rnd1_doc.txt'
sudo -u qa1 rm -f /srv/hanbit/share/rnd1_doc.txt

sudo -u qa1 sh -c 'echo "품질 자료" > /srv/hanbit/share/qa1_doc.txt'
sudo -u qa1 rm -f /srv/hanbit/share/qa1_doc.txt

ls -ld /tmp
```

**해설**

- 스티키 비트가 없는 `777` 디렉터리에서는 **누구든 남의 파일을 삭제할 수 있다.** 파일 삭제 권한은 파일 자신이 아니라 **상위 디렉터리의 `w` 권한**이 결정하기 때문이다.
- `1777`을 적용하면 `drwxrwxrwt`가 되어 마지막 자리가 `t`로 표시되며, **자신이 소유한 파일만 삭제**할 수 있다.
- `/tmp` 역시 동일하게 `drwxrwxrwt`이다. 모든 사용자가 임시 파일을 만들지만 남의 파일은 지울 수 없어야 하기 때문이다.
- 두 번째 `qa1`의 삭제 시도는 `Operation not permitted`로 실패한다. `rm -f`를 사용하여도 권한 자체가 없으므로 삭제되지 않는다.

</details>

**완료 기준** — `share`가 `drwxrwxrwt`이고, `rnd1_doc.txt`가 삭제되지 않고 남아 있다.

---
---

# 문제 5. umask 계산 검증

---

> **상황**
> 신규 서버의 기본 생성 권한을 조직 기준(`027`)으로 강화하려 한다. 적용 전에 결과를 예측하여야 한다.
>
> **요구사항**
> 1. 현재 umask 값을 8진수와 기호 형식으로 각각 확인한다.
> 2. 다음 표의 빈칸을 **계산으로 먼저 채운 뒤** 실제로 생성하여 대조한다.
>
> | umask | 파일 권한 | 디렉터리 권한 |
> |---|---|---|
> | 022 | ? | ? |
> | 027 | ? | ? |
> | 077 | ? | ? |
>
> 3. 어떤 umask를 적용하여도 **파일에는 실행 권한이 부여되지 않는** 이유를 설명한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
mkdir -p ~/review03/umask_test && cd ~/review03/umask_test

umask
umask -S

umask 022 && touch f022 && mkdir d022
umask 027 && touch f027 && mkdir d027
umask 077 && touch f077 && mkdir d077

ls -l
umask 022
umask
```

**해설**

| umask | 파일(기준 666) | 디렉터리(기준 777) |
|---|---|---|
| 022 | **644** `rw-r--r--` | **755** `rwxr-xr-x` |
| 027 | **640** `rw-r-----` | **750** `rwxr-x---` |
| 077 | **600** `rw-------` | **700** `rwx------` |

- 기준값이 **파일은 666, 디렉터리는 777**로 서로 다르다는 점이 핵심이다. "umask 022일 때 파일 권한은?"의 정답은 `755`가 아니라 **`644`** 이며, 시험에서 반복 출제되는 함정이다.
- 파일의 기준값에 실행 비트(`1`)가 포함되어 있지 않으므로, umask 값과 무관하게 **새로 만든 파일에는 `x`가 부여되지 않는다.** 실행 권한은 만든 사람이 `chmod`로 명시적으로 부여하여야 한다. 이는 내려받은 파일이 즉시 실행되는 것을 막는 안전장치이다.
- 시스템 전역 기본값은 `/etc/login.defs`의 `UMASK` 항목이 결정하며, 사용자별로 강화하려면 `~/.bashrc`에 `umask 027`을 추가한다.

</details>

**완료 기준** — 예측한 표와 `ls -l`의 실제 결과가 모두 일치한다.

---
---

# 문제 6. 접근 장애 진단

---

> **상황**
> 연구팀에서 다음과 같은 문의가 접수되었다.
>
> *"자료실 디렉터리 안에 파일이 있는 것은 목록으로 보이는데, 열어 보려고 하면 권한이 없다고 나옵니다. 파일 권한은 644로 모두 열려 있는데 왜 그렇습니까?"*
>
> **요구사항**
> 1. 아래 블록을 실행하여 동일한 상황을 재현한다.
> 2. `rnd1`의 입장에서 **목록 조회는 되고 파일 열람은 되지 않는** 현상을 확인한다.
> 3. 원인을 진단한다. **파일이 아니라 어디를 보아야 하는가.**
> 4. 최소한의 변경으로 시정한 뒤 정상 동작을 확인한다.
{: .prompt-info }

```bash
sudo mkdir -p /srv/hanbit/trouble
echo "설계 기준서 본문" | sudo tee /srv/hanbit/trouble/spec.txt > /dev/null
sudo chmod 644 /srv/hanbit/trouble/spec.txt
sudo chmod 744 /srv/hanbit/trouble
ls -ld /srv/hanbit/trouble
ls -l /srv/hanbit/trouble
```

**힌트** — 파일과 디렉터리에서 `rwx`의 의미가 서로 다르다는 점을 떠올린다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
# 2. 현상 확인
sudo -u rnd1 ls /srv/hanbit/trouble
sudo -u rnd1 ls -l /srv/hanbit/trouble
sudo -u rnd1 cat /srv/hanbit/trouble/spec.txt

# 3. 원인 진단 — 디렉터리 권한을 본다
ls -ld /srv/hanbit/trouble

# 4. 시정 — 통과 권한(x)을 부여한다
sudo chmod 755 /srv/hanbit/trouble
ls -ld /srv/hanbit/trouble
sudo -u rnd1 cat /srv/hanbit/trouble/spec.txt
```

**해설**

- 원인은 **디렉터리의 `x`(통과) 권한이 없기 때문**이다. `744`는 `drwxr--r--`로, 기타 사용자에게 `r`만 부여되어 있다.
- 디렉터리에서 각 권한의 의미는 다음과 같다.
>
> | 권한 | 디렉터리에서의 의미 |
> |---|---|
> | `r` | 내부 항목의 **이름 목록**을 조회할 수 있다 |
> | `w` | 내부에 항목을 **생성·삭제**할 수 있다 |
> | `x` | 디렉터리를 **통과**하여 내부 항목에 도달할 수 있다 |
>
- 따라서 `r`만 있으면 **이름은 보이지만 그 안의 파일에는 도달할 수 없다.** `ls -l`을 실행하면 각 항목의 상세 정보를 얻지 못해 `Permission denied` 또는 물음표가 표시된다.
- 파일 권한이 `644`인 것은 문제가 아니었다. **파일에 도달하기 전에 디렉터리에서 막힌 것**이다. 도서관에 비유하면 장서 목록표는 볼 수 있으나 서가 안으로 들어갈 수 없는 상태이다.
- 디렉터리에는 통상 `r`과 `x`를 함께 부여한다. `755`와 `750`이 널리 쓰이는 이유가 여기에 있다.

</details>

**완료 기준** — 시정 후 `rnd1`이 `spec.txt`의 내용을 읽을 수 있다.

---
---

# 문제 7. 보안 점검 5종 ★핵심

---

> **상황**
> 다섯 번째 요구사항이다. 준비 단계에서 생성한 "전임자가 남긴 설정" 다섯 가지에는 각각 보안상 문제가 있다. **무엇이 문제인지 스스로 찾아내어** 시정하여야 한다.
>
> **요구사항** — 다음 다섯 항목을 순서대로 점검하고 시정한다.
>
> | 번호 | 점검 항목 | 판정 기준 |
> |---|---|---|
> | ① | UID가 0인 계정 | `root` 하나만 존재하여야 한다 |
> | ② | 비밀번호가 설정되지 않은 계정 | 하나도 없어야 한다 |
> | ③ | 로그인 셸이 부여된 시스템 계정 | 서비스 계정은 `nologin`이어야 한다 |
> | ④ | 누구나 쓸 수 있는 설정 파일 | `/srv` 아래에 없어야 한다 |
> | ⑤ | 표준 경로 밖의 SetUID 파일 | 없어야 한다 |
{: .prompt-info }

**힌트** — `awk -F:`로 `/etc/passwd`와 `/etc/shadow`의 필드를 조건 검사한다. 파일 점검은 `find`의 `-perm` 조건을 사용한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

**① UID 0 계정 점검**

```bash
awk -F: '$3 == 0 {print $1, $3, $7}' /etc/passwd
```

`root` 외에 **`svcbackup`** 이 함께 출력된다. 리눅스 커널은 **계정 이름과 무관하게 UID가 0이면 관리자 권한을 부여**하므로, 이 계정은 사실상 두 번째 root이다. 평범한 이름으로 위장한 관리자 계정은 침해 사고에서 재침입 통로로 사용되는 대표적 수법이다.

```bash
sudo userdel svcbackup
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

**② 비밀번호 없는 계정 점검**

```bash
sudo awk -F: '($2 == "") {print "비밀번호 없음: " $1}' /etc/shadow
```

**`tempadmin`** 이 검출된다. 비밀번호 필드가 비어 있으면 **아무 입력 없이 인증이 통과**될 수 있다. 즉시 잠그거나 비밀번호를 설정하여야 한다.

```bash
sudo passwd -l tempadmin
sudo passwd -S tempadmin
```

**③ 로그인 셸이 부여된 시스템 계정 점검**

```bash
awk -F: '$3 < 1000 && $3 != 0 && $7 ~ /(bash|zsh|ksh)$/ {print $1, $3, $7}' /etc/passwd
```

**`legacysvc`** 가 검출된다. 서비스 전용 계정은 프로그램을 구동하는 데만 사용되므로 대화형 로그인이 불필요하다. 셸을 부여해 두면 해당 계정이 탈취되었을 때 곧바로 명령을 실행할 수 있는 발판이 된다.

```bash
sudo usermod -s /usr/sbin/nologin legacysvc
grep "^legacysvc:" /etc/passwd
```

**④ 누구나 쓸 수 있는 파일 점검**

```bash
sudo find /srv -type f -perm -0002 -exec ls -l {} +
```

**`/srv/hanbit/legacy/app.conf`** 가 `-rw-rw-rw-`(666)로 검출된다. 임의의 사용자가 설정 파일을 수정할 수 있다는 뜻이며, 접속 정보를 조작하여 통신을 가로채는 공격이 가능해진다.

```bash
sudo chmod 640 /srv/hanbit/legacy/app.conf
sudo chown root:root /srv/hanbit/legacy/app.conf
ls -l /srv/hanbit/legacy/app.conf
```

**⑤ 비정상 SetUID 파일 점검**

```bash
sudo find / -perm -4000 -type f 2>/dev/null | grep -vE "^/usr|^/snap"
```

**`/srv/hanbit/legacy/oldtool`** 이 검출된다. root 소유의 SetUID 파일은 **실행되는 동안 root 권한으로 동작**하므로, 파일 탐색·열람 기능을 가진 도구에 이를 부여하면 일반 사용자가 시스템의 모든 파일을 읽을 수 있게 된다.

```bash
sudo chmod u-s /srv/hanbit/legacy/oldtool
ls -l /srv/hanbit/legacy/oldtool
sudo find / -perm -4000 -type f 2>/dev/null | grep -vE "^/usr|^/snap" || echo "표준 경로 외 SetUID 파일 없음"
```

**공통 해설**

- `-perm -0002`의 앞선 `-`는 "**해당 비트를 포함하는**"이라는 의미이다. `-perm 0002`(부호 없음)는 권한이 정확히 `000000010`인 경우만 찾으므로 실무에서는 거의 사용하지 않는다.
- `/usr`와 `/snap`을 제외하는 이유는 배포판이 정상적으로 배치한 SetUID 프로그램(`passwd`, `sudo`, `snap-confine` 등)이 그곳에 있기 때문이다. **점검의 목적은 "표준 목록에 없는 것"을 찾는 데 있다.**
- SetUID 제거는 `chmod u-s` 또는 `chmod 0755`로 수행한다.

</details>

**완료 기준** — 다섯 항목의 재점검 결과가 모두 "이상 없음"으로 나온다.

---
---

# 문제 8. sudo 최소 권한 위임

---

> **상황**
> 연구팀이 "서비스 상태와 로그를 직접 확인하고 싶다"고 요청하였다. 관리 권한 전체를 부여할 수는 없다.
>
> **요구사항**
> 1. `rndteam` 그룹에 **서비스 상태 조회와 로그 열람 명령만** 허용하는 규칙을 작성한다.
> 2. 규칙은 `/etc/sudoers`를 직접 편집하지 말고 **별도 파일로 분리**하여 관리한다.
> 3. 문법 검사를 수행한다.
> 4. `rnd1`로 전환하여 **허용된 명령은 실행되고 허용되지 않은 명령은 거부되는지** 확인한다.
{: .prompt-info }

**힌트** — 규칙 파일은 `/etc/sudoers.d/` 아래에 두며, 편집은 반드시 `visudo`를 사용한다. 그룹은 `%`로 지정한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sudo visudo -f /etc/sudoers.d/95-rndteam
```

편집기가 열리면 다음 한 행을 입력하고 저장한다.

```
%rndteam ALL=(root) /usr/bin/systemctl status *, /usr/bin/journalctl
```

```bash
sudo visudo -c
ls -l /etc/sudoers.d/95-rndteam

su - rnd1
sudo -l
sudo systemctl status ssh --no-pager
sudo cat /etc/shadow
exit
```

**해설**

- `%`로 시작하면 **그룹**을 의미한다. 개별 사용자마다 규칙을 만드는 대신 그룹 단위로 위임하면 인원 변동에 유연하게 대응할 수 있다.
- `/etc/sudoers`를 편집기로 직접 여는 것은 금지된다. **`visudo`는 저장 시 문법을 검사**하여 오류가 있는 파일이 반영되는 것을 차단한다. 문법이 깨진 `sudoers`가 저장되면 시스템의 모든 사용자가 `sudo`를 사용할 수 없게 되어 복구가 매우 곤란해진다.
- 규칙을 `/etc/sudoers.d/` 아래의 개별 파일로 분리하면 추가·회수가 쉽고 변경 이력을 관리하기 좋다.
- `sudo -l`은 **자신에게 허용된 명령 목록**을 보여 준다. 위임 범위를 확인하는 표준 방법이다.
- `sudo cat /etc/shadow`는 규칙에 없으므로 거부된다. 이것이 `sudo`가 **전권 위임이 아니라 범위를 지정한 위임**이라는 증거이며, `su`와의 결정적 차이이다.

</details>

**완료 기준** — `rnd1`이 `sudo systemctl status ssh`는 실행할 수 있고 `sudo cat /etc/shadow`는 거부된다.

---
---

# 문제 9. 점검 보고서 작성

---

> **상황**
> 정보보호 부서에 제출할 보고서를 작성하여야 한다.
>
> **요구사항**
> `~/review03/audit_report.txt`에 다음 항목을 기록한다.
>
> | 항목 | 내용 |
> |---|---|
> | [1] | 신규 계정 현황(계정명 / UID / 로그인 셸) |
> | [2] | 그룹 구성 |
> | [3] | 디렉터리 권한(특수 권한 포함) |
> | [4] | 비밀번호 정책 적용 결과 |
> | [5] | 보안 점검 5종의 재점검 결과 |
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
mkdir -p ~/review03 && cd ~/review03

{
  echo "===== 연구소 계정 개편 및 보안 점검 보고서 ====="
  echo "작성 일시: $(date '+%Y-%m-%d %H:%M')"
  echo "작성 계정: $(whoami)@$(hostname)"
  echo
  echo "[1] 신규 계정 현황"
  grep -E "^(rnd1|rnd2|qa1):" /etc/passwd | cut -d: -f1,3,7 | sed 's/^/  - /'
  echo
  echo "[2] 그룹 구성"
  getent group rndteam qateam | sed 's/^/  - /'
  echo
  echo "[3] 디렉터리 권한"
  ls -ld /srv/hanbit/* | awk '{print "  " $1 "  " $3 ":" $4 "  " $9}'
  echo
  echo "[4] 비밀번호 정책 (rnd1)"
  sudo chage -l rnd1 | grep -iE "maximum|minimum|warning" | sed 's/^/  /'
  echo
  echo "[5] 보안 점검 재점검 결과"
  echo "  ① UID 0 계정      : $(awk -F: '$3==0 {print $1}' /etc/passwd | tr '\n' ' ')"
  echo "  ② 비밀번호 없는 계정: $(sudo awk -F: '$2=="" {print $1}' /etc/shadow | tr '\n' ' ')(비어 있으면 정상)"
  echo "  ③ 셸 보유 시스템계정: $(awk -F: '$3<1000 && $3!=0 && $7 ~ /(bash|zsh|ksh)$/ {print $1}' /etc/passwd | tr '\n' ' ')(비어 있으면 정상)"
  echo "  ④ 쓰기개방 파일     : $(sudo find /srv -type f -perm -0002 2>/dev/null | wc -l) 건"
  echo "  ⑤ 비표준 SetUID    : $(sudo find / -perm -4000 -type f 2>/dev/null | grep -vE '^/usr|^/snap' | wc -l) 건"
} > audit_report.txt

cat audit_report.txt
```

**해설**

- `[5]`의 각 항목은 **점검 명령을 그대로 보고서에 포함**하는 방식으로 작성하였다. 사람이 손으로 옮겨 적으면 오기가 발생하지만, 명령의 출력을 직접 기록하면 근거가 남는다.
- `tr '\n' ' '`는 여러 행의 결과를 한 줄로 이어 붙인다. 보고서의 한 항목에 값을 나열할 때 유용하다.
- `④`와 `⑤`는 건수만 기록하였다. **정상 상태의 기댓값이 `0`** 이므로 숫자만으로 판정이 가능하다.

</details>

**완료 기준** — 보고서의 `[5]`에서 ①이 `root`뿐이고 ②③이 비어 있으며 ④⑤가 `0 건`이다.

---
---

# 마무리. 자가 채점

---

```bash
echo "===== 자가 채점 결과 ====="

id rnd2 2>/dev/null | grep -q rndteam && id rnd2 | grep -q qateam \
  && echo "[문제 1] 통과 — 계정·그룹 구성" || echo "[문제 1] 미완료"

sudo chage -l rnd1 2>/dev/null | grep -q "90" \
  && echo "[문제 2] 통과 — 비밀번호 정책" || echo "[문제 2] 미완료"

[ -g /srv/hanbit/rnd_area ] \
  && echo "[문제 3-1] 통과 — SetGID 설정" || echo "[문제 3-1] 미완료"

[ "$(stat -c '%G' /srv/hanbit/rnd_area/design_v1.txt 2>/dev/null)" = "rndteam" ] \
  && echo "[문제 3-2] 통과 — 그룹 상속 확인" || echo "[문제 3-2] 미완료"

[ -k /srv/hanbit/share ] \
  && echo "[문제 4] 통과 — 스티키 비트" || echo "[문제 4] 미완료"

[ "$(stat -c '%a' /srv/hanbit/trouble 2>/dev/null)" = "755" ] \
  && echo "[문제 6] 통과 — 접근 장애 시정" || echo "[문제 6] 미완료"

[ "$(awk -F: '$3==0' /etc/passwd | wc -l)" -eq 1 ] \
  && echo "[문제 7-①] 통과 — UID 0은 root뿐" || echo "[문제 7-①] 미완료"

[ -z "$(sudo awk -F: '$2=="" {print $1}' /etc/shadow)" ] \
  && echo "[문제 7-②] 통과 — 빈 비밀번호 없음" || echo "[문제 7-②] 미완료"

[ -z "$(awk -F: '$3<1000 && $3!=0 && $7 ~ /(bash|zsh|ksh)$/ {print $1}' /etc/passwd)" ] \
  && echo "[문제 7-③] 통과 — 시스템 계정 셸 차단" || echo "[문제 7-③] 미완료"

[ "$(sudo find /srv -type f -perm -0002 2>/dev/null | wc -l)" -eq 0 ] \
  && echo "[문제 7-④] 통과 — 쓰기개방 파일 없음" || echo "[문제 7-④] 미완료"

[ "$(sudo find / -perm -4000 -type f 2>/dev/null | grep -vcE '^/usr|^/snap')" -eq 0 ] \
  && echo "[문제 7-⑤] 통과 — 비표준 SetUID 없음" || echo "[문제 7-⑤] 미완료"

[ -s ~/review03/audit_report.txt ] \
  && echo "[문제 9] 통과 — 보고서 작성" || echo "[문제 9] 미완료"
```

> **채점에 사용한 검사 기호**
>
> | 기호 | 의미 |
> |---|---|
> | `[ -g 경로 ]` | **SetGID**가 설정되어 있는가 |
> | `[ -u 경로 ]` | **SetUID**가 설정되어 있는가 |
> | `[ -k 경로 ]` | **스티키 비트**가 설정되어 있는가 |
> | `[ -z 문자열 ]` | 문자열이 **비어 있는가** |
> | `stat -c '%a'` | 권한을 8진수로 출력 |
> | `stat -c '%G'` | 소유 그룹 이름을 출력 |
>
> 특수 권한을 `ls` 결과에 `grep`으로 검사하면 **경로 문자열에 우연히 포함된 문자 때문에 오판**할 수 있다. `[ -g ]`·`[ -k ]`와 같은 전용 검사를 사용하여야 한다.
{: .prompt-tip }

---

## 이론 점검

**문항 1.** 계정 이름이 `svcbackup`인데도 관리자와 동일한 권한을 갖는 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

리눅스 커널은 **계정 이름이 아니라 UID로 권한을 판단**하며, **UID가 0이면 관리자**로 취급한다. 따라서 이름과 무관하게 UID 0인 계정은 모두 root와 동일한 권한을 갖는다.

</details>

**문항 2.** `usermod -G qateam rnd2`와 `usermod -aG qateam rnd2`의 차이는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

`-a`(append)가 없으면 **기존 보조 그룹이 모두 제거되고** 지정한 그룹만 남는다. 관리자 계정에 실수로 실행하면 `sudo` 그룹에서 빠져 관리 권한을 상실한다.

</details>

**문항 3.** 권한이 `444`인 파일이 `755` 디렉터리 안에 있다. 디렉터리 소유자가 이 파일을 삭제할 수 있는가?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**삭제할 수 있다.** 파일 삭제는 "파일 내용을 고치는 일"이 아니라 "디렉터리에 등록된 항목을 제거하는 일"이므로, **상위 디렉터리의 `w` 권한**이 결정한다.

</details>

**문항 4.** umask가 `027`일 때 새로 만든 파일과 디렉터리의 권한은?

<details markdown="1">
<summary><b>정답 보기</b></summary>

파일은 **640**(`rw-r-----`), 디렉터리는 **750**(`rwxr-x---`)이다. 기준값이 파일 666, 디렉터리 777로 서로 다르기 때문이다.

</details>

**문항 5.** 팀 공유 디렉터리에 `chmod 2770`을 적용하는 목적은?

<details markdown="1">
<summary><b>정답 보기</b></summary>

앞자리 `2`는 **SetGID**로, 그 디렉터리 안에 생성되는 항목이 **디렉터리의 소유 그룹을 상속**하게 한다. 이를 통해 같은 팀 구성원이 서로의 파일에 그룹 권한으로 접근할 수 있다.

</details>

**문항 6.** 셸 스크립트에 `chmod 4755`를 적용하면 root 권한으로 동작하는가?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**동작하지 않는다.** 리눅스 커널은 스크립트의 SetUID를 의도적으로 무시한다. 인터프리터가 대신 실행하는 구조에서 경합 조건을 이용한 공격이 가능하기 때문이며, SetUID는 **컴파일된 바이너리에만** 적용된다.

</details>

---
---

# 정리 절차 — 반드시 수행

---

본 종합문제는 취약 설정을 재현하였으므로 **정리를 생략해서는 안 된다.**

```bash
sudo rm -rf /srv/hanbit
```

```bash
sudo rm -f /etc/sudoers.d/95-rndteam
```

```bash
sudo userdel -r rnd1
sudo userdel -r rnd2
sudo userdel -r qa1
sudo userdel -r tempadmin
sudo userdel legacysvc
```

```bash
sudo groupdel rndteam
sudo groupdel qateam
```

```bash
rm -rf ~/review03/umask_test
```

**정리 결과를 검증한다.**

```bash
grep -E "^(rnd1|rnd2|qa1|tempadmin|legacysvc|svcbackup):" /etc/passwd || echo "실습 계정 전부 삭제 완료"
```

```bash
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

```bash
sudo find / -xdev -nouser 2>/dev/null | head
```

```bash
sudo visudo -c
```

> **검증 기준**
> - 첫 번째 명령에서 "실습 계정 전부 삭제 완료"가 출력된다.
> - 두 번째 명령의 결과가 **`root` 하나뿐**이다.
> - 세 번째 명령은 아무것도 출력하지 않는다(소유자 없는 파일이 남지 않았다는 의미이다).
> - 네 번째 명령이 `parsed OK`로 끝난다.
>
> `userdel` 실행 시 "currently used by process"라는 오류가 발생하면, 해당 계정으로 열어 둔 세션이 남아 있는 것이다. `exit`으로 모든 세션을 종료한 뒤 다시 시도한다.
{: .prompt-danger }

---

## 자기 점검

```
 [ ] useradd의 -m·-s·-G 옵션의 역할을 각각 설명할 수 있다.
 [ ] usermod -aG에서 -a를 빠뜨렸을 때의 결과를 설명할 수 있다.
 [ ] 디렉터리에서 r·w·x가 각각 무엇을 허용하는지 설명할 수 있다.
 [ ] 파일 삭제 권한이 상위 디렉터리의 w로 결정된다는 사실을 실험으로 확인하였다.
 [ ] umask 값으로부터 파일과 디렉터리의 권한을 각각 계산할 수 있다.
 [ ] SetUID·SetGID·스티키 비트의 8진수 값과 용도를 구분할 수 있다.
 [ ] UID 0 계정, 빈 비밀번호, 비표준 SetUID를 스스로 점검할 수 있다.
 [ ] 정리 절차를 완료하여 시스템을 원래 상태로 되돌렸다.
```

다음 복습은 **제4강 종합문제 — 점검 도구 복구와 자동화**이다.
