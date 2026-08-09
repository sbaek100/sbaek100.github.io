---
title: 리눅스 기초 2강 - 디렉터리 구조와 기본 명령어
date: 2026-09-01 09:00:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - FHS
  - 절대경로
  - 상대경로
  - ls
  - cp
  - mv
  - rm
  - find
  - grep
  - tar
pin:
mermaid: false
---

> **학습 목표**
> 1. FHS 표준에 따른 주요 디렉터리의 용도를 설명할 수 있다.
> 2. 절대 경로와 상대 경로를 구분하고 원하는 위치로 이동할 수 있다.
> 3. `ls -l` 출력의 각 항목을 해석할 수 있다.
> 4. 파일과 디렉터리를 생성·복사·이동·삭제할 수 있다.
> 5. 파일 내용을 상황에 맞는 방법으로 조회할 수 있다.
> 6. `find`와 `grep`을 사용하여 파일과 문자열을 검색할 수 있다.
> 7. `tar`로 여러 파일을 묶고 압축할 수 있다.
{: .prompt-info }

윈도우에서 마우스로 수행하던 작업 — 폴더 열기, 파일 복사, 이름 변경, 삭제 — 을 리눅스에서는 명령어로 수행한다. 처음에는 마우스가 더 편리하게 느껴질 수 있으나, 명령어에는 결정적인 장점이 있다. **하나의 명령으로 수천 개의 파일을 동시에 처리할 수 있고, 그 명령을 저장해 두면 자동으로 반복 실행할 수 있다.**

본 강의에서 학습하는 명령어들은 이후 모든 실습의 기초가 된다.

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 디렉터리 구조와 경로 | 40분 |
| 제2절 | 파일과 디렉터리 조작 | 40분 |
| 제3절 | 파일 내용 조회 | 25분 |
| 제4절 | 검색과 아카이브 | 35분 |
| 제5절 | 종합 실습 | 30분 |
| 제6·7절 | 오류 대응 및 이론 평가 | 10분 |

---
---

# 제1절. 디렉터리 구조와 경로

---

## 1.1 FHS와 주요 디렉터리

리눅스의 디렉터리 구조는 **FHS(Filesystem Hierarchy Standard)** 라는 표준으로 규정되어 있다. 어느 배포판을 사용하더라도 동일한 위치에 동일한 성격의 파일이 존재하므로, 한 번 익혀 두면 계속 활용할 수 있다.

```
 /                        루트. 모든 것의 시작점
 ├── bin  → usr/bin       일반 사용자 명령(ls, cp 등)
 ├── boot                 커널 이미지, GRUB 설정
 ├── dev                  장치 파일
 ├── etc                  ★ 시스템 전역 설정 파일
 ├── home                 ★ 일반 사용자의 홈 디렉터리
 │    └── student              사용자의 개인 작업 공간(~)
 ├── lib  → usr/lib       공유 라이브러리
 ├── media                이동식 매체 자동 마운트 지점
 ├── mnt                  임시 마운트 지점
 ├── opt                  서드파티 응용 프로그램
 ├── proc                 커널·프로세스 정보(가상)
 ├── root                 ※ 관리자의 홈 디렉터리
 ├── sbin → usr/sbin      관리자용 명령
 ├── srv                  서비스 제공 데이터
 ├── sys                  장치·드라이버 정보(가상)
 ├── tmp                  임시 파일
 ├── usr                  설치된 프로그램과 문서
 └── var                  ★ 로그·스풀 등 가변 데이터
      └── log                  시스템 로그
```

실무에서 특히 자주 접근하는 디렉터리는 다음과 같다.

| 디렉터리 | 용도 | 주요 내용 |
|---|---|---|
| **`/etc`** | 설정 파일 보관소. **실행 파일을 두지 않는다.** | `passwd`, `shadow`, `fstab`, `ssh/`, `apt/` |
| **`/home`** | 사용자별 개인 작업 공간 | `/home/student` |
| **`/var/log`** | 시스템 로그 | `syslog`, `auth.log`, `dpkg.log` |
| `/tmp` | 임시 파일. 모든 사용자가 사용 가능 | 재부팅 시 삭제 |
| `/usr/local` | 관리자가 직접 설치한 프로그램 | 패키지 관리자가 관여하지 않는 영역 |
| `/root` | 관리자 전용 홈 디렉터리 | 일반 사용자 접근 불가 |

> **혼동하기 쉬운 개념**
> `/`(루트 디렉터리)와 `/root`(관리자 홈 디렉터리)는 명칭이 유사하나 전혀 다른 대상이다. 전자는 파일 트리의 최상위이고, 후자는 관리자 계정의 개인 디렉터리이다. 시험에서 이를 바꾸어 묻는 문항이 자주 출제된다.
{: .prompt-warning }

---

## 1.2 절대 경로와 상대 경로

파일의 위치를 지정하는 방법은 두 가지이다.

| 구분 | 정의 | 예시 |
|---|---|---|
| **절대 경로** | 루트(`/`)에서 시작하는 완전한 경로 | `/home/student/docs/report.txt` |
| **상대 경로** | **현재 작업 디렉터리**를 기준으로 하는 경로 | `docs/report.txt` |

일상적인 표현에 비유하면 절대 경로는 "서울특별시 강남구 테헤란로 123"과 같이 어디서 출발하든 찾아갈 수 있는 주소이고, 상대 경로는 "여기서 오른쪽으로 두 블록"과 같이 현재 위치를 알아야 의미가 성립하는 안내이다.

**구분 방법은 간단하다. 맨 앞에 `/`가 있으면 절대 경로, 없으면 상대 경로이다.**

경로 표기에 사용되는 특수 기호는 다음과 같다.

| 기호 | 의미 | 사용 예 |
|---|---|---|
| `.` (점 하나) | **현재** 디렉터리 | `./run.sh` |
| `..` (점 두 개) | **상위(부모)** 디렉터리 | `cd ..` |
| `~` (물결) | **현재 사용자의 홈** 디렉터리 | `cd ~` |
| `-` (하이픈) | **직전에 있던** 디렉터리 | `cd -` |
| `/` | 루트 디렉터리 또는 경로 구분자 | `/etc/ssh` |

---

## 1.3 이동과 목록 조회 명령

| 명령어 | 약어 풀이 | 기능 |
|---|---|---|
| `pwd` | **p**rint **w**orking **d**irectory | 현재 디렉터리 출력 |
| `cd` | **c**hange **d**irectory | 디렉터리 이동 |
| `ls` | **l**i**s**t | 디렉터리 내용 목록 출력 |

`ls`는 가장 빈번하게 사용하는 명령이므로 주요 옵션을 숙지하여야 한다.

| 옵션 | 약어 풀이 | 기능 |
|---|---|---|
| `-l` | **l**ong | 상세 정보 표시 |
| `-a` | **a**ll | 숨김 파일 포함 |
| `-h` | **h**uman-readable | 크기를 KB/MB/GB 단위로 |
| `-t` | **t**ime | 수정 시각순 정렬 |
| `-r` | **r**everse | 역순 정렬 |
| `-R` | **R**ecursive | 하위 디렉터리까지 재귀 출력 |
| `-d` | **d**irectory | 디렉터리 자체 정보만 출력 |
| `-i` | **i**node | i-노드 번호 표시 |
| `-S` | **S**ize | 크기순 정렬 |

> **리눅스의 숨김 파일**
> 윈도우는 파일 속성으로 숨김 여부를 설정하지만, 리눅스는 **파일명 앞에 마침표(`.`)를 붙이면 숨김 파일이 된다.** `.bashrc`, `.ssh` 등이 이에 해당하며 `ls -a`로만 확인할 수 있다.
{: .prompt-tip }

---

## 1.4 `ls -l` 출력의 해석

`ls -l`의 출력은 9개 항목으로 구성되며, 이는 3강에서 학습할 권한 체계의 전제가 되므로 정확히 익혀야 한다.

```
 -rw-r--r--   1  student  student   1024  Mar 16 10:24  report.txt
 └───┬────┘   │  └──┬──┘  └──┬──┘  └─┬─┘  └────┬─────┘  └────┬───┘
     │        │     │        │       │         │             │
     │        │     │        │       │         │             └ 파일명
     │        │     │        │       │         └─────────────── 최종 수정 시각
     │        │     │        │       └───────────────────────── 크기(바이트)
     │        │     │        └───────────────────────────────── 소유 그룹
     │        │     └────────────────────────────────────────── 소유자
     │        └──────────────────────────────────────────────── 링크 수
     └───────────────────────────────────────────────────────── 파일 종류 + 권한
```

첫 글자는 파일의 종류를 나타낸다.

| 기호 | 종류 |
|---|---|
| `-` | 일반 파일 |
| **`d`** | **d**irectory(디렉터리) |
| `l` | **l**ink(심볼릭 링크) |
| `b` | 블록 장치(디스크) |
| `c` | 문자 장치(터미널 등) |

이어지는 아홉 글자가 권한을 나타내며, 상세한 내용은 3강에서 다룬다.

---

> ### 따라 하기 2-1. 경로 이동과 목록 조회
>
> **목적** 절대 경로와 상대 경로를 사용하여 이동하고, `ls` 옵션의 차이를 확인한다.
{: .prompt-tip }

**1단계.** 현재 위치를 확인한다.

```bash
pwd
```

**2단계.** 절대 경로로 이동한다.

```bash
cd /etc
```

```bash
pwd
```

> 맨 앞에 `/`가 있으므로 절대 경로이다. 프롬프트도 `:/etc$`로 변경된 것을 확인한다.

**3단계.** 상위 디렉터리로 이동한 뒤 상대 경로로 다시 내려간다.

```bash
cd ..
```

```bash
pwd
```

> `/`가 출력된다. `/etc`의 상위는 루트이기 때문이다.

```bash
cd etc/ssh
```

```bash
pwd
```

> 맨 앞에 `/`가 없으므로 상대 경로이다. 현재 위치인 `/`에서 출발하여 `etc` → `ssh`로 이동하였다.

**4단계.** 홈 디렉터리로 복귀한다.

```bash
cd
```

```bash
pwd
```

> `cd`를 인자 없이 실행하면 항상 홈 디렉터리로 이동한다. **경로를 잃었을 때 사용하는 방법이다.**

**5단계.** 직전 디렉터리를 왕복한다.

```bash
cd -
```

```bash
cd -
```

**6단계.** `ls` 옵션의 차이를 비교한다.

```bash
ls /etc | head -10
```

> 이름만 출력된다.

```bash
ls -l /etc | head -10
```

> 권한·소유자·크기·시각이 함께 출력된다.

```bash
ls ~
```

```bash
ls -a ~
```

> 두 결과를 비교한다. `-a`를 붙이면 `.bashrc`, `.profile` 등 마침표로 시작하는 숨김 파일이 나타난다.

```bash
ls -lhtr /var/log | tail -10
```

> `-lhtr`는 상세(l) + 읽기 쉬운 크기(h) + 시각순(t) + 역순(r)의 조합이다. 결과적으로 가장 최근에 수정된 파일이 화면 하단에 위치하므로 로그 확인에 유용하다.

**7단계.** 디렉터리 자체 정보만 조회한다.

```bash
ls -l /etc/ssh
```

```bash
ls -ld /etc/ssh
```

> `-d`가 없으면 디렉터리 **내용**을, 있으면 디렉터리 **자체**의 정보를 출력한다. 3강에서 디렉터리 권한을 확인할 때 반드시 필요한 옵션이다.

> **확인 사항** 절대 경로와 상대 경로로 각각 이동할 수 있고, `-a`와 `-d` 옵션의 효과를 설명할 수 있다면 성공이다.
{: .prompt-tip }

---
---

# 제2절. 파일과 디렉터리 조작

---

## 2.1 디렉터리 생성과 삭제

| 명령어 | 약어 풀이 | 기능 | 주요 옵션 |
|---|---|---|---|
| `mkdir` | **m**a**k**e **dir**ectory | 디렉터리 생성 | `-p` 상위 디렉터리까지 생성 |
| `rmdir` | **r**e**m**ove **dir**ectory | **빈** 디렉터리만 삭제 | `-p` 상위까지 삭제 |

```bash
mkdir project
mkdir -p project/src/main
```

`-p`는 **p**arents(부모)의 약어로, 중간 디렉터리가 존재하지 않아도 한 번에 생성한다.

`rmdir`이 빈 디렉터리만 삭제하도록 설계된 것은 **의도치 않은 데이터 삭제를 방지하기 위한 안전장치**이다.

---

## 2.2 파일 생성

| 명령 | 기능 |
|---|---|
| `touch 파일명` | 빈 파일 생성. 파일이 이미 있으면 시각만 갱신 |
| `echo "내용" > 파일명` | 내용을 포함한 파일 생성 |

```bash
touch memo.txt
echo "안녕하세요" > hello.txt
```

> **`>`와 `>>`의 구분 — 매우 중요**
> - **`>`** : 기존 내용을 **모두 삭제하고** 새로 기록한다(덮어쓰기).
> - **`>>`** : 기존 내용 **뒤에 이어서** 기록한다(추가).
>
> 중요한 파일에 `>`를 잘못 사용하면 내용이 즉시 소실된다. 이 기호는 명령을 실행하기 **전에** 파일을 먼저 비우기 때문이다.
{: .prompt-warning }

---

## 2.3 복사·이동·삭제

| 명령어 | 약어 풀이 | 기능 |
|---|---|---|
| `cp` | **c**o**p**y | 복사 |
| `mv` | **m**o**v**e | 이동 또는 이름 변경 |
| `rm` | **r**e**m**ove | 삭제 |

공통으로 사용하는 주요 옵션은 다음과 같다.

| 옵션 | 약어 풀이 | 기능 |
|---|---|---|
| **`-r`** | **r**ecursive | 디렉터리와 그 내용 전체. **디렉터리를 다룰 때 필수** |
| `-i` | **i**nteractive | 실행 전 확인 질의 |
| `-p` | **p**reserve | 원본의 속성(권한·시각) 유지 |
| `-a` | **a**rchive | 보관용 복사(`-dR --preserve=all`과 동일) |
| `-v` | **v**erbose | 진행 상황 출력 |
| `-f` | **f**orce | 확인 없이 강제 실행 |

`mv`가 이동과 이름 변경을 동시에 담당하는 이유는, 두 작업이 모두 "디렉터리에 등록된 이름표를 바꾸어 다는 일"로 동일하기 때문이다. 따라서 동일 파일 시스템 내에서는 실제 데이터를 옮기지 않으므로 용량과 무관하게 즉시 완료된다.

> **삭제 시 반드시 유의할 사항**
> 리눅스에는 휴지통 개념이 없다. `rm`으로 삭제한 파일은 **복구가 사실상 불가능하다.** 특히 `sudo rm -rf /경로` 형태의 명령은 시스템 전체를 손상시킬 수 있다.
>
> **권장하는 작업 습관**: 삭제하기 전에 동일한 대상을 `ls`로 먼저 확인한다.
>
> ```
> ls *.txt        ← 대상을 눈으로 확인
> rm *.txt        ← 확인 후 삭제
> ```
{: .prompt-danger }

---

## 2.4 와일드카드(파일명 대치)

여러 파일을 한 번에 지정할 때 사용하는 기호를 **와일드카드** 또는 **globbing**이라 한다. 명령을 실행하기 **전에 셸이 먼저** 해당 기호를 실제 파일명 목록으로 확장한다.

| 기호 | 의미 | 사용 예 |
|---|---|---|
| `*` | 임의의 문자 **0개 이상** | `*.txt` → 모든 txt 파일 |
| `?` | 임의의 문자 **정확히 1개** | `file?.txt` → file1.txt, fileA.txt |
| `[abc]` | 나열된 문자 중 하나 | `file[12].txt` |
| `[a-z]` | 범위 내 문자 하나 | `[a-c]*.conf` |
| `[!abc]` | 나열된 문자가 **아닌** 하나 | `[!0-9]*` |
| `{a,b}` | 나열한 각각으로 확장 | `mkdir {src,docs,test}` |

---

> ### 따라 하기 2-2. 디렉터리·파일 생성과 조작
>
> **목적** 실습용 디렉터리 구조를 생성하고, 복사·이동·삭제 명령을 사용한다.
{: .prompt-tip }

**1단계.** 실습 디렉터리를 생성하고 이동한다.

```bash
mkdir ~/lab02
```

```bash
cd ~/lab02
```

**2단계.** 중간 디렉터리가 없을 때의 동작을 확인한다.

```bash
mkdir project/src/main
```

> 오류가 발생한다. 상위 디렉터리인 `project`가 존재하지 않기 때문이다.

```bash
mkdir -p project/src/main
```

> `-p`를 붙이면 중간 디렉터리까지 한 번에 생성된다.

**3단계.** 여러 디렉터리를 한 번에 생성한다.

```bash
mkdir -p project/{docs,tests,logs}
```

```bash
ls -R project
```

> `-R`은 하위 디렉터리까지 재귀적으로 출력한다.

**4단계.** 파일을 생성한다.

```bash
touch project/docs/readme.txt
```

```bash
echo "리눅스 기초 2강 실습" > project/docs/readme.txt
```

```bash
cat project/docs/readme.txt
```

```bash
printf "apple\nbanana\ncherry\napple\n" > project/docs/fruits.txt
```

> `\n`은 줄바꿈을 의미한다.

```bash
cat -n project/docs/fruits.txt
```

> `-n`은 행 번호를 함께 출력한다.

**5단계.** 빈 디렉터리와 그렇지 않은 디렉터리의 삭제 차이를 확인한다.

```bash
rmdir project/logs
```

> 성공한다. `logs`는 비어 있기 때문이다.

```bash
rmdir project/docs
```

> `Directory not empty` 오류가 발생한다. `rmdir`은 빈 디렉터리만 삭제한다.

**6단계.** 여러 파일을 생성하고 와일드카드로 조작한다.

```bash
touch file1.txt file2.txt file3.txt data1.log data2.log note.md
```

```bash
ls
```

```bash
ls *.txt
```

> **파괴적인 명령을 실행하기 전에 동일한 와일드카드를 `ls`에 먼저 적용하여 대상을 확인하는 것이 안전한 작업 습관이다.**

```bash
mkdir backup
```

```bash
cp *.txt backup/
```

```bash
ls backup
```

**7단계.** 디렉터리 복사 시 `-r` 옵션의 필요성을 확인한다.

```bash
cp project project_copy
```

> `omitting directory` 경고와 함께 실패한다.

```bash
cp -r project project_copy
```

```bash
ls project_copy
```

**8단계.** 속성을 보존하며 복사한다.

```bash
cp -a project project_archive
```

```bash
ls -ld project project_archive
```

> `-a`는 권한·소유자·시각을 그대로 유지하므로 백업 용도에 적합하다.

**9단계.** 이름을 변경하고 이동한다.

```bash
mv file3.txt report.txt
```

```bash
mv note.md project/docs/
```

```bash
ls
```

**10단계.** 확인 옵션을 사용하여 삭제한다.

```bash
rm -i data1.log
```

> `y`를 입력하여 삭제한다. `-i`는 초보 학습자에게 유용한 안전장치이다.

```bash
rm -r project_copy
```

```bash
ls
```

> **확인 사항** `backup` 디렉터리에 txt 파일 3개가 복사되어 있고, `project_copy`가 삭제되었다면 성공이다.
{: .prompt-tip }

---
---

# 제3절. 파일 내용 조회

---

## 3.1 조회 명령의 종류와 선택 기준

| 명령어 | 약어 풀이 | 적합한 상황 |
|---|---|---|
| `cat` | con**cat**enate | 짧은 파일 전체를 한 번에 확인 |
| `less` | — | 긴 파일을 페이지 단위로 확인 |
| `head` | head(머리) | 앞부분만 확인 |
| `tail` | tail(꼬리) | 뒷부분 또는 **실시간 로그** 확인 |
| `wc` | **w**ord **c**ount | 행·단어·문자 수 계산 |
| `file` | — | 파일의 종류 판별 |
| `stat` | status | i-노드·크기·권한 등 상세 정보 |

```bash
cat /etc/hostname          # 전체 출력
less /etc/services         # 페이지 단위 (q로 종료)
head -5 /etc/passwd        # 앞 5행
tail -5 /etc/passwd        # 뒤 5행
tail -f /var/log/syslog    # 실시간 추적
wc -l /etc/passwd          # 행 수
```

---

## 3.2 `less`의 조작 방법

`less`는 대용량 파일을 메모리에 모두 적재하지 않고 필요한 부분만 읽어 표시하므로, 수 기가바이트 파일도 즉시 열 수 있다.

| 키 | 동작 |
|---|---|
| `Space` | 다음 화면 |
| `b` | 이전 화면 |
| `↑` `↓` | 한 행 이동 |
| `/문자열` | 아래 방향 검색 |
| `n` / `N` | 다음 / 이전 검색 결과 |
| `g` / `G` | 문서 처음 / 끝 |
| **`q`** | **종료** |

> **초심자가 자주 겪는 상황**
> `less`나 `man` 화면에서 빠져나오지 못해 당황하는 경우가 많다. **`q`** 를 누르면 종료된다.
{: .prompt-tip }

---

## 3.3 `tail -f`의 활용

`-f`는 **f**ollow(따라가기)의 약어로, 파일에 새로운 내용이 기록되는 즉시 화면에 이어서 출력한다. 서비스 장애를 추적할 때 가장 먼저 사용하는 형태이므로 반드시 익혀 두어야 한다. 종료는 `Ctrl + C`로 한다.

---

> ### 따라 하기 2-3. 파일 내용 조회
>
> **목적** 상황에 따라 적절한 조회 명령을 선택하여 사용한다.
{: .prompt-tip }

**1단계.** 짧은 파일을 전체 출력한다.

```bash
cat /etc/hostname
```

```bash
cat -n ~/lab02/project/docs/fruits.txt
```

**2단계.** 긴 파일을 페이지 단위로 조회한다.

```bash
less /etc/services
```

> 화면 내에서 다음을 순서대로 수행한다.
> 1. `/http` 입력 후 `Enter` — http가 포함된 위치로 이동
> 2. `n` — 다음 검색 결과로 이동
> 3. `G` — 문서 끝으로 이동
> 4. `g` — 문서 처음으로 이동
> 5. `q` — 종료

**3단계.** 앞부분과 뒷부분을 조회한다.

```bash
head -5 /etc/passwd
```

```bash
tail -5 /etc/passwd
```

**4단계.** 로그를 실시간으로 추적한다.

터미널 두 개가 필요하다. SSH 세션을 하나 더 열거나, 물리 콘솔에서 `Ctrl+Alt+F2`로 다른 콘솔을 사용한다.

**첫 번째 터미널**에서 다음을 실행한다.

```bash
sudo tail -f /var/log/auth.log
```

**두 번째 터미널**에서 다음을 실행한다.

```bash
sudo -k
```

```bash
sudo whoami
```

> 첫 번째 터미널에 새로운 행이 실시간으로 추가되는 것을 관찰한다. 관찰이 끝나면 `Ctrl + C`로 종료한다.
>
> `sudo -k`는 이전에 인증된 자격 증명을 무효화하여 비밀번호를 다시 묻게 한다. `/var/log/auth.log`에는 인증 및 권한 상승 기록이 저장되며, 3강의 보안 감사 주제와 직결된다.

**5단계.** 행 수를 계산하고 파일 종류를 판별한다.

```bash
wc -l /etc/passwd
```

```bash
file /etc/passwd /bin/ls /etc
```

> 각각 텍스트 파일, 실행 파일, 디렉터리로 구분하여 출력한다.

> **확인 사항** `less`에서 검색과 종료를 수행하였고, `tail -f`로 실시간 로그를 관찰하였다면 성공이다.
{: .prompt-tip }

---
---

# 제4절. 검색과 아카이브

---

## 4.1 파일 검색 — `find`

`find`는 디렉터리 트리를 실시간으로 순회하며 조건에 맞는 항목을 찾는다.

```
 find  [검색 시작 경로]  [조건]  [수행할 동작]
 find         .        -name "*.txt"   -print
```

| 조건 | 의미 |
|---|---|
| `-name "패턴"` | 이름이 일치(대소문자 구분) |
| `-iname "패턴"` | 이름이 일치(대소문자 무시) |
| `-type f` / `-type d` | 일반 파일 / 디렉터리 |
| `-size +10M` | 크기가 10MB 초과 |
| `-mtime -7` | 최근 7일 이내 수정 |
| `-user 사용자` | 특정 사용자 소유 |
| `-perm 644` | 권한이 일치 |
| `-maxdepth n` | 탐색 깊이 제한 |

| 동작 | 의미 |
|---|---|
| `-print` | 경로 출력(기본값) |
| `-delete` | 삭제 |
| `-exec 명령 {} \;` | 찾은 항목마다 명령 실행 |
| `-exec 명령 {} +` | 여러 항목을 한 번에 전달(효율적) |

---

## 4.2 문자열 검색 — `grep`

`grep`은 파일 **내부의 내용**에서 특정 문자열이 포함된 행을 찾는다.

```
 grep  [옵션]  "패턴"  파일...
```

| 옵션 | 약어 풀이 | 기능 |
|---|---|---|
| `-n` | **n**umber | 행 번호 표시 |
| `-i` | **i**gnore case | 대소문자 무시 |
| `-v` | in**v**ert | 일치하지 **않는** 행 출력 |
| `-c` | **c**ount | 일치 행의 개수만 출력 |
| `-r` | **r**ecursive | 디렉터리 재귀 검색 |
| `-l` | **l**ist | 일치하는 파일명만 출력 |
| `-w` | **w**ord | 단어 단위로 일치 |

기초적인 정규 표현식 기호는 다음과 같다.

| 기호 | 의미 |
|---|---|
| `^` | 행의 시작 |
| `$` | 행의 끝 |
| `.` | 임의의 한 문자 |
| `*` | 직전 문자의 0회 이상 반복 |

> **`find`와 `grep`의 구분**
> - **`find`** : **파일 자체**를 찾는다(파일 이름·크기·시각 기준).
> - **`grep`** : **파일 내부의 문자열**을 찾는다.
>
> "파일이 어디 있는지 모를 때"는 `find`, "이 단어가 어느 파일에 있는지 알고 싶을 때"는 `grep`을 사용한다.
>
> 또한 셸의 와일드카드와 `grep`의 정규 표현식은 서로 다른 체계이다. 셸의 `*`는 "임의의 문자열"이지만, 정규 표현식의 `*`는 "직전 문자의 반복"을 의미한다.
{: .prompt-warning }

---

## 4.3 파이프 — 명령의 연결

`|`(파이프) 기호는 앞 명령의 출력을 뒤 명령의 입력으로 전달한다. 이를 통해 단순한 명령들을 조합하여 복잡한 작업을 수행할 수 있으며, 이는 유닉스 철학의 핵심에 해당한다.

```bash
ls -l /etc | head -10
ps aux | grep nginx
cat /etc/passwd | wc -l
```

---

## 4.4 아카이브와 압축 — `tar`

리눅스에서는 **묶기(archive)** 와 **압축(compression)** 이 개념적으로 분리되어 있다.

| 구분 | 설명 | 대표 명령 |
|---|---|---|
| 묶기 | 여러 파일을 하나의 파일로 결합. 크기는 줄어들지 않는다. | `tar` |
| 압축 | 파일의 크기를 줄인다. | `gzip`, `bzip2`, `xz` |

`tar`는 본래 묶기만 수행하는 명령이나, 옵션을 지정하면 압축까지 함께 수행한다.

| 옵션 | 약어 풀이 | 기능 |
|---|---|---|
| `c` | **c**reate | 새로 묶기 |
| `x` | e**x**tract | 풀기 |
| `t` | lis**t** | 내용 목록 확인 |
| `v` | **v**erbose | 진행 상황 출력 |
| **`f`** | **f**ile | 파일명 지정. **항상 옵션 나열의 마지막에 위치** |
| `z` | g**z**ip | gzip 압축 |
| `j` | b**z**ip2 | bzip2 압축 |
| `J` | **J**(xz) | xz 압축 |
| `-C 경로` | **C**hange dir | 지정한 위치에 풀기 |

```bash
tar -cvf backup.tar mydata            # 묶기만
tar -czvf backup.tar.gz mydata        # 묶고 gzip 압축
tar -tzvf backup.tar.gz               # 내용 확인
tar -xzvf backup.tar.gz -C /tmp       # 지정 위치에 풀기
```

> **`f` 옵션의 위치**
> `f` 다음에는 반드시 파일명이 와야 한다. `tar -cvfz data.tar.gz data`와 같이 작성하면 `f`가 바로 뒤의 **`z`를 파일명으로 해석**하여, 다음과 같은 결과가 발생한다.
>
> ```
> tar: data.tar.gz: Cannot stat: No such file or directory
> tar: Exiting with failure status due to previous errors
> ```
>
> 즉 `z`라는 이름의 엉뚱한 아카이브가 생성되고, `data.tar.gz`는 압축 대상 파일로 취급되어 "그런 파일이 없다"는 오류가 출력된다. 이 경우 `rm -f z`로 잘못 생성된 파일을 삭제한다.
>
> **`f`를 옵션 나열의 마지막에 두어야 한다.** 시험에도 자주 출제되는 사항이다.
{: .prompt-warning }

---

> ### 따라 하기 2-4. 검색과 아카이브
>
> **목적** `find`와 `grep`으로 검색하고, `tar`로 파일을 묶고 압축한다.
{: .prompt-tip }

**1단계.** 이름으로 파일을 검색한다.

```bash
cd ~/lab02
```

```bash
find . -name "*.txt"
```

```bash
find . -iname "FILE*"
```

> `-iname`은 대소문자를 무시하므로 소문자 `file1.txt`도 검색된다.

**2단계.** 종류와 조건으로 검색한다.

```bash
find . -type d
```

```bash
find . -type f -mtime -1
```

```bash
sudo find /var/log -type f -size +1M 2>/dev/null
```

> `2>/dev/null`은 권한 부족 등의 오류 메시지를 화면에서 제거한다. 상세한 원리는 4강에서 학습한다.

**3단계.** 검색 결과에 명령을 적용한다.

```bash
find . -name "*.txt" -exec ls -l {} +
```

> `{}`가 찾은 파일 경로로 치환된다. `+`는 여러 항목을 한 번에 전달하여 처리 속도를 높인다.

**4단계.** 파일 내부에서 문자열을 검색한다.

```bash
grep "apple" project/docs/fruits.txt
```

```bash
grep -n "apple" project/docs/fruits.txt
```

```bash
grep -c "apple" project/docs/fruits.txt
```

```bash
grep -v "apple" project/docs/fruits.txt
```

**5단계.** 시스템 파일에서 정보를 추출한다.

```bash
grep "^root" /etc/passwd
```

> `^root`는 행이 `root`로 시작하는 경우를 의미한다.

```bash
grep "bash$" /etc/passwd
```

> `bash$`는 행이 `bash`로 끝나는 경우를 의미한다. 즉 로그인 셸이 bash인 계정을 찾는 실용적인 검색이며, 3강에서 활용한다.

**6단계.** 파이프로 명령을 연결한다.

```bash
ls -l /etc | wc -l
```

```bash
cat /etc/passwd | grep bash | wc -l
```

> "passwd 파일을 읽고 → bash가 포함된 행만 골라 → 행 수를 계산하라"는 세 단계 작업을 하나의 명령으로 처리하였다.

**7단계.** 파일을 묶고 압축한다.

```bash
tar -cvf project.tar project
```

```bash
tar -czvf project.tar.gz project
```

```bash
ls -lh project.tar*
```

> 두 파일의 크기를 비교한다. `.gz`가 붙은 쪽이 현저히 작으며, 이는 `z` 옵션이 압축을 수행하였기 때문이다.

**8단계.** 내용을 확인하고 다른 위치에 푼다.

```bash
tar -tzvf project.tar.gz | head -5
```

```bash
mkdir -p restore
```

```bash
tar -xzvf project.tar.gz -C restore
```

```bash
ls restore/project
```

> **확인 사항** `find`로 조건에 맞는 파일을, `grep`으로 원하는 행을 각각 추출하였고, `tar`로 압축·해제하였다면 성공이다.
{: .prompt-tip }

---
---

# 제5절. 종합 실습 — 로그 자료 정리 및 백업

---

> **실습 시나리오**
>
> 학습자는 시스템 운영 담당자로서 다음 업무를 지시받았다.
>
> *"각 부서에서 제출한 자료가 뒤섞여 있으니, 확장자별로 분류하여 정리하시오. 정리한 결과는 압축하여 백업하고, 처리 내역을 보고서로 작성하시오."*
>
> 본 실습은 제1절부터 제4절까지 학습한 명령어만으로 수행한다.
{: .prompt-info }

---

## 단계 1. 작업 환경 구성

```bash
mkdir -p ~/lab02/workspace
```

```bash
cd ~/lab02/workspace
```

```bash
pwd
```

---

## 단계 2. 정리 대상 자료 생성

실제 업무 상황을 모의하기 위해 여러 종류의 파일을 생성한다.

```bash
touch sales_2026Q1.csv sales_2026Q2.csv marketing_plan.docx
```

```bash
touch server_01.log server_02.log server_03.log
```

```bash
touch backup_config.conf network.conf
```

```bash
echo "2026년 1분기 매출 보고서" > sales_2026Q1.csv
```

```bash
echo "2026년 2분기 매출 보고서" > sales_2026Q2.csv
```

```bash
printf "ERROR: disk full\nINFO: service started\nERROR: timeout\n" > server_01.log
```

```bash
printf "INFO: backup done\nWARN: high memory\n" > server_02.log
```

```bash
printf "ERROR: connection refused\nINFO: restart\n" > server_03.log
```

```bash
ls -l
```

---

## 단계 3. 분류 디렉터리 생성

```bash
mkdir -p sorted/{csv,log,conf,docs}
```

```bash
ls -R sorted
```

---

## 단계 4. 확장자별 분류

각 이동 명령 전에 대상을 먼저 확인하는 절차를 포함한다.

```bash
ls *.csv
```

```bash
mv *.csv sorted/csv/
```

```bash
ls *.log
```

```bash
mv *.log sorted/log/
```

```bash
ls *.conf
```

```bash
mv *.conf sorted/conf/
```

```bash
ls *.docx
```

```bash
mv *.docx sorted/docs/
```

```bash
ls -R sorted
```

---

## 단계 5. 로그 파일 분석

```bash
grep -c "ERROR" sorted/log/*.log
```

> 각 로그 파일에 `ERROR`가 몇 건 포함되어 있는지 계산한다.

```bash
grep -n "ERROR" sorted/log/*.log
```

> 어느 파일의 몇 번째 행에 오류가 기록되었는지 확인한다.

```bash
cat sorted/log/*.log | grep "ERROR" | wc -l
```

> 전체 오류 건수를 계산한다.

---

## 단계 6. 처리 내역 보고서 작성

```bash
echo "===== 자료 정리 처리 보고서 =====" > summary.txt
```

```bash
date >> summary.txt
```

```bash
echo "" >> summary.txt
```

```bash
echo "[1] 분류 결과 (디렉터리별 파일 수)" >> summary.txt
```

```bash
find sorted -type f | wc -l >> summary.txt
```

```bash
echo "" >> summary.txt
```

```bash
echo "[2] 분류된 파일 목록" >> summary.txt
```

```bash
find sorted -type f >> summary.txt
```

```bash
echo "" >> summary.txt
```

```bash
echo "[3] 로그 오류 건수" >> summary.txt
```

```bash
grep -c "ERROR" sorted/log/*.log >> summary.txt
```

```bash
cat summary.txt
```

---

## 단계 7. 백업 아카이브 생성

```bash
tar -czvf backup_$(date +%Y%m%d).tar.gz sorted summary.txt
```

```bash
ls -lh backup_*.tar.gz
```

```bash
tar -tzvf backup_*.tar.gz
```

---

## 단계 8. 복원 검증

백업이 정상적으로 이루어졌는지 검증하는 절차는 실무에서 필수적이다. **백업은 복원이 가능해야 비로소 백업으로서 의미를 가진다.**

```bash
mkdir -p ~/lab02/verify
```

```bash
tar -xzvf backup_*.tar.gz -C ~/lab02/verify
```

```bash
find ~/lab02/verify -type f | wc -l
```

```bash
find sorted -type f | wc -l
```

> 백업 아카이브에는 `sorted` 디렉터리의 파일들과 **`summary.txt` 한 개**가 함께 들어 있다. 따라서 앞의 숫자(복원본)가 뒤의 숫자(원본 `sorted`)보다 **정확히 1만큼 크면** 복원이 정상적으로 수행된 것이다.
>
> 예를 들어 `sorted`에 9개가 있다면 복원본에서는 10개가 조회된다. 차이가 1이 아니라면 백업 또는 복원 과정에서 누락이 발생한 것이다.

```bash
cat ~/lab02/verify/summary.txt | head -5
```

---

## 단계 9. 정리

```bash
rm -rf ~/lab02/verify
```

```bash
ls ~/lab02/workspace
```

> **종합 실습 완료 기준**
> 1. `sorted` 디렉터리 아래에 확장자별로 파일이 분류되어 있다.
> 2. `summary.txt`에 파일 수와 오류 건수가 기록되어 있다.
> 3. 날짜가 포함된 백업 파일이 생성되었다.
> 4. 백업을 다른 위치에 풀어 파일 수가 `sorted`보다 1개(보고서) 많음을 확인하였다.
>
> 본 실습에서 사용한 명령은 `mkdir`, `cd`, `pwd`, `touch`, `echo`, `printf`, `ls`, `mv`, `find`, `grep`, `wc`, `cat`, `date`, `tar`, `rm`의 15개이다.
{: .prompt-tip }

---
---

# 제6절. 자주 발생하는 오류와 대응 방법

---

| 화면에 출력된 메시지 | 원인 | 대응 방법 |
|---|---|---|
| `No such file or directory` | 파일 또는 경로가 존재하지 않음 | `ls`로 이름을 확인한다. **대소문자**를 확인한다. |
| `Directory not empty` | `rmdir`로 비어 있지 않은 디렉터리를 삭제 시도 | `rm -r`을 사용한다. |
| `omitting directory` | `cp`에서 `-r` 옵션 누락 | `cp -r`로 다시 실행한다. |
| `cannot remove: Is a directory` | `rm`에 `-r` 옵션 누락 | `rm -r`을 사용한다. |
| `Permission denied` | 권한 부족 | `ls -l`로 권한을 확인한다. 상세 내용은 3강에서 다룬다. |
| `tar: 파일명: Cannot stat: No such file or directory` | `f` 옵션 뒤에 파일명이 오지 않음(`-cvfz` 형태) | `-czvf`와 같이 `f`를 마지막에 배치한다. 이때 **`z`라는 이름의 파일이 잘못 생성되므로 함께 삭제**한다. |
| `rm: cannot remove '*.txt'` | 조건에 맞는 파일이 없어 와일드카드가 확장되지 않음 | `ls *.txt`로 대상 존재 여부를 먼저 확인한다. |
| `less`/`man`에서 나가지 못함 | 페이지 표시 상태 | **`q`** 를 누른다. |

---
---

# 제7절. 이론 평가

---

**문항 1.** 리눅스의 시스템 전역 설정 파일이 저장되는 디렉터리는?

① `/var` ② **`/etc`** ③ `/usr` ④ `/opt`

---

**문항 2.** 현재 디렉터리를 기준으로 **상위 디렉터리**를 나타내는 기호는?

① `.` ② **`..`** ③ `~` ④ `-`

---

**문항 3.** `ls` 명령에서 숨김 파일까지 모두 표시하는 옵션은?

① `-l` ② **`-a`** ③ `-h` ④ `-R`

---

**문항 4.** `mkdir` 명령으로 존재하지 않는 중간 디렉터리까지 한 번에 생성하는 옵션은?

① `-r` ② `-m` ③ **`-p`** ④ `-a`

---

**문항 5.** 디렉터리를 복사할 때 반드시 지정해야 하는 `cp` 옵션은?

① `-i` ② **`-r`** ③ `-v` ④ `-f`

---

**문항 6.** 파일의 기존 내용을 삭제하지 않고 **뒤에 이어서** 기록하는 리다이렉션 기호는?

① `>` ② **`>>`** ③ `<` ④ `|`

---

**문항 7.** 로그 파일에 새로 기록되는 내용을 **실시간으로** 화면에 출력하는 명령은?

① `head -f` ② **`tail -f`** ③ `cat -f` ④ `less -f`

---

**문항 8.** `/etc/passwd` 파일에서 행이 `bash`로 끝나는 행만 출력하는 명령은?

① `grep "^bash" /etc/passwd`
② **`grep "bash$" /etc/passwd`**
③ `grep "*bash" /etc/passwd`
④ `grep -v "bash" /etc/passwd`

---

**문항 9.** `find` 명령으로 현재 디렉터리 아래에서 크기가 10MB를 초과하는 **일반 파일**을 검색하는 명령은?

① `find . -type d -size +10M`
② **`find . -type f -size +10M`**
③ `find . -type f -size 10M`
④ `find . -name "+10M"`

---

**문항 10.** `data` 디렉터리를 gzip으로 압축하여 `data.tar.gz`로 생성하는 올바른 명령은?

① `tar -cvfz data.tar.gz data`
② **`tar -czvf data.tar.gz data`**
③ `tar -xzvf data.tar.gz data`
④ `tar -tzvf data.tar.gz data`

---
---

# 제8절. 요약

---

## 8.1 핵심 개념 정리

| 구분 | 요점 |
|---|---|
| 디렉터리 구조 | FHS 표준에 따라 `/`를 정점으로 하는 단일 트리를 구성한다. `/etc`(설정), `/home`(사용자), `/var/log`(로그)를 우선 기억한다. |
| 경로 표기 | 맨 앞에 `/`가 있으면 절대 경로, 없으면 상대 경로이다. |
| `ls -l` 해석 | 종류·권한·링크 수·소유자·그룹·크기·시각·이름의 순으로 출력되며, 3강 권한 학습의 전제가 된다. |
| 파일 조작 | 디렉터리를 다룰 때는 `-r` 옵션이 필수이며, 삭제 전 `ls`로 대상을 확인하는 습관이 중요하다. |
| 리다이렉션 | `>`는 덮어쓰기, `>>`는 추가이다. |
| 검색 | `find`는 파일 자체를, `grep`은 파일 내부의 문자열을 찾는다. |
| 아카이브 | `tar`는 묶기, `gzip` 계열은 압축이며 별개의 작업이다. `f` 옵션은 항상 마지막에 배치한다. |

---

## 8.2 본 강의에서 학습한 명령어

| 명령어 | 약어 풀이 | 기능 |
|---|---|---|
| `cd` | change directory | 디렉터리 이동 |
| `ls` | list | 목록 조회 |
| `mkdir` | make directory | 디렉터리 생성 |
| `rmdir` | remove directory | 빈 디렉터리 삭제 |
| `touch` | — | 빈 파일 생성 |
| `cp` | copy | 복사 |
| `mv` | move | 이동·이름 변경 |
| `rm` | remove | 삭제 |
| `cat` | concatenate | 파일 내용 출력 |
| `less` | — | 페이지 단위 조회 |
| `head` / `tail` | — | 앞부분 / 뒷부분 조회 |
| `wc` | word count | 행·단어 수 계산 |
| `file` | — | 파일 종류 판별 |
| `find` | — | 파일 검색 |
| `grep` | — | 문자열 검색 |
| `tar` | tape archive | 묶기·압축 |

---

## 8.3 다음 강의 예고

제3강에서는 본 강의에서 생성한 파일들이 **누구의 소유이며 누가 어떤 작업을 수행할 수 있는지**를 결정하는 계정과 권한 체계를 학습한다. 리눅스 시스템 관리에서 가장 중요한 내용이며, 리눅스마스터 2급 시험에서도 출제 비중이 가장 높은 영역이다.
