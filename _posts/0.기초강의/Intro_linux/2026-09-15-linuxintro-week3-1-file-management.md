---
title: 리눅스 기초 3주차-1. 파일과 디렉터리 관리
date: 2026-09-15 09:00:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - touch
  - mkdir
  - cp
  - mv
  - rm
  - cat
  - find
  - 링크
pin:
mermaid: false
---

> **학습 목표**
> 1. `touch`로 빈 파일을, `mkdir`로 디렉터리를 생성할 수 있다.
> 2. `cp`·`mv`·`rm`·`rmdir`로 파일과 디렉터리를 복사·이동·삭제할 수 있다.
> 3. 와일드카드를 사용하여 여러 파일을 한 번에 지정할 수 있다.
> 4. `cat`·`less`·`head`·`tail`을 상황에 맞게 선택하여 파일 내용을 조회할 수 있다.
> 5. `find`로 조건에 맞는 파일을 검색할 수 있다.
> 6. 하드 링크와 심볼릭 링크의 개념을 구분하여 설명할 수 있다.
{: .prompt-info }

Ubuntu Desktop 24.04 LTS에서 마우스로 수행하던 작업 — 폴더 만들기, 파일 복사, 이름 변경, 삭제 — 을 터미널에서는 명령어로 수행한다. 바탕화면 좌측의 활동 표시줄에서 **파일(Files)** 앱을 열면 그래픽 환경으로도 동일한 작업을 할 수 있으나, 명령어에는 결정적인 장점이 있다. **하나의 명령으로 수천 개의 파일을 동시에 처리할 수 있고, 그 명령을 저장해 두면 자동으로 반복 실행할 수 있다.**

본 강의의 모든 실습은 터미널에서 수행한다. 터미널은 `Ctrl + Alt + T`를 누르거나 활동 표시줄에서 **터미널(Terminal)**을 선택하여 연다. 프롬프트는 다음과 같은 형태로 표시된다.

```
student@ubuntu-y1:~$
```

여기서 `student`는 로그인한 사용자, `ubuntu-y1`은 호스트(컴퓨터) 이름, `~`는 현재 위치가 홈 디렉터리(`/home/student`)임을 나타낸다.


---
---

# 제1절. 파일과 디렉터리 만들기

---

## 1.1 빈 파일 생성 — `touch`

`touch`는 본래 파일의 시각 정보를 갱신하는 명령이나, 대상 파일이 존재하지 않으면 **빈 파일을 새로 생성**한다. 그러므로 실습용 파일을 준비할 때 널리 사용한다.

| 명령 | 기능 |
|---|---|
| `touch 파일명` | 빈 파일 생성. 파일이 이미 있으면 최종 수정 시각만 갱신 |
| `touch a.txt b.txt c.txt` | 여러 파일을 한 번에 생성 |

```bash
touch memo.txt
touch report1.txt report2.txt report3.txt
```

내용을 포함한 파일을 만들 때에는 `echo`와 리다이렉션 기호를 사용한다.

```bash
echo "안녕하세요" > hello.txt
```

> **`>`와 `>>`의 구분 — 매우 중요**
> - **`>`** : 기존 내용을 **모두 삭제하고** 새로 기록한다(덮어쓰기).
> - **`>>`** : 기존 내용 **뒤에 이어서** 기록한다(추가).
>
> 중요한 파일에 `>`를 잘못 사용하면 내용이 즉시 소실된다. 이 기호는 명령을 실행하기 **전에** 파일을 먼저 비우기 때문이다.
{: .prompt-warning }

---

## 1.2 디렉터리 생성 — `mkdir`

`mkdir`은 **m**a**k**e **dir**ectory(디렉터리를 만들다)의 약어이다.

| 명령어 | 약어 풀이 | 기능 | 주요 옵션 |
|---|---|---|---|
| `mkdir` | make directory | 디렉터리 생성 | `-p` 상위 디렉터리까지 함께 생성 |

```bash
mkdir project
mkdir -p project/src/main
```

`-p`는 **p**arents(부모)의 약어로, 중간 디렉터리가 존재하지 않아도 한 번에 생성한다. `-p` 없이 `mkdir project/src/main`을 실행하면 상위 디렉터리 `project`가 없어 오류가 발생한다.

중괄호 확장을 사용하면 여러 디렉터리를 한 번에 생성할 수 있다.

```bash
mkdir -p project/{docs,tests,logs}
```

위 명령은 `project` 아래에 `docs`, `tests`, `logs` 세 디렉터리를 동시에 만든다.

> **[그림 3-1] 필요**
> `mkdir -p project/src/main` 실행으로 생성되는 `project → src → main`의 3단계 디렉터리 트리를, 상위에서 하위로 화살표를 그어 계단식으로 표현한 다이어그램.
{: .prompt-tip }

---

> ### 따라 하기 1-1. 실습 디렉터리와 파일 만들기
>
> **목적** 실습용 디렉터리 구조를 생성하고, 파일을 만들어 본다.
{: .prompt-tip }

**1단계.** 실습 디렉터리를 생성하고 그 안으로 이동한다.

```bash
mkdir ~/lab03
```

```bash
cd ~/lab03
```

> `~`는 홈 디렉터리를 의미하므로, `~/lab03`은 `/home/student/lab03`을 가리킨다.

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

> `-R`은 하위 디렉터리까지 재귀적으로 출력한다. `project` 아래에 `docs`, `logs`, `src`, `tests`가 존재하는지 확인한다.

**4단계.** 파일을 생성하고 내용을 기록한다.

```bash
touch project/docs/readme.txt
```

```bash
echo "리눅스 기초 3주차 실습" > project/docs/readme.txt
```

```bash
cat project/docs/readme.txt
```

```bash
printf "apple\nbanana\ncherry\napple\n" > project/docs/fruits.txt
```

> `printf`의 `\n`은 줄바꿈을 의미한다. 위 명령은 네 줄로 이루어진 파일을 만든다.

```bash
cat -n project/docs/fruits.txt
```

> `-n`은 각 행 앞에 행 번호를 붙여 출력한다.

> **확인 사항** `~/lab03/project` 아래에 디렉터리 네 개가 생성되고, `readme.txt`와 `fruits.txt`에 내용이 기록되어 있다면 성공이다.
{: .prompt-tip }

---
---

# 제2절. 복사·이동·삭제

---

## 2.1 세 가지 조작 명령

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
| `-a` | **a**rchive | 보관용 복사(속성 전체 유지) |
| `-v` | **v**erbose | 진행 상황 출력 |
| `-f` | **f**orce | 확인 없이 강제 실행 |

`mv`가 이동과 이름 변경을 동시에 담당하는 이유는, 두 작업이 모두 "디렉터리에 등록된 이름표를 바꾸어 다는 일"로 동일하기 때문이다. 따라서 동일한 저장 장치 안에서는 실제 데이터를 옮기지 않으므로 용량과 무관하게 즉시 완료된다.

> **[그림 3-2] 필요**
> `cp`(원본 유지, 사본 생성), `mv`(원본이 새 위치로 이동), `rm`(대상 제거) 세 명령의 동작을 상자와 화살표로 대비하여 표현한 개념도.
{: .prompt-tip }

---

## 2.2 디렉터리 삭제 — `rmdir`

| 명령어 | 약어 풀이 | 기능 |
|---|---|---|
| `rmdir` | **r**e**m**ove **dir**ectory | **빈** 디렉터리만 삭제 |

`rmdir`이 빈 디렉터리만 삭제하도록 설계된 것은 **의도치 않은 데이터 삭제를 방지하기 위한 안전장치**이다. 내용이 있는 디렉터리를 삭제하려면 `rm -r`을 사용한다.

> **삭제 시 반드시 유의할 사항**
> 리눅스 터미널에는 휴지통 개념이 없다. `rm`으로 삭제한 파일은 **복구가 사실상 불가능하다.** 특히 `sudo rm -rf /경로` 형태의 명령은 시스템 전체를 손상시킬 수 있다.
>
> **권장하는 작업 습관**: 삭제하기 전에 동일한 대상을 `ls`로 먼저 확인한다.
>
> ```
> ls *.txt        ← 대상을 눈으로 확인
> rm *.txt        ← 확인 후 삭제
> ```
{: .prompt-danger }

---

## 2.3 와일드카드(파일명 대치)

여러 파일을 한 번에 지정할 때 사용하는 기호를 **와일드카드**라 한다. 명령을 실행하기 **전에 셸이 먼저** 해당 기호를 실제 파일명 목록으로 확장한다.

| 기호 | 의미 | 사용 예 |
|---|---|---|
| `*` | 임의의 문자 **0개 이상** | `*.txt` → 모든 txt 파일 |
| `?` | 임의의 문자 **정확히 1개** | `file?.txt` → file1.txt, fileA.txt |
| `[abc]` | 나열된 문자 중 하나 | `file[12].txt` |
| `[a-z]` | 범위 내 문자 하나 | `[a-c]*.conf` |
| `{a,b}` | 나열한 각각으로 확장 | `mkdir {src,docs,test}` |

---

> ### 따라 하기 1-2. 복사·이동·삭제
>
> **목적** 와일드카드로 여러 파일을 지정하고, 복사·이동·삭제 명령을 사용한다.
{: .prompt-tip }

**1단계.** 실습 위치로 이동하고 여러 파일을 생성한다.

```bash
cd ~/lab03
```

```bash
touch file1.txt file2.txt file3.txt data1.log data2.log note.md
```

```bash
ls
```

**2단계.** 파괴적 명령을 실행하기 전에 와일드카드로 대상을 먼저 확인한다.

```bash
ls *.txt
```

> **파괴적인 명령을 실행하기 전에 동일한 와일드카드를 `ls`에 먼저 적용하여 대상을 확인하는 것이 안전한 작업 습관이다.**

**3단계.** 파일을 복사한다.

```bash
mkdir backup
```

```bash
cp *.txt backup/
```

```bash
ls backup
```

> `backup` 디렉터리에 txt 파일 세 개가 복사되었는지 확인한다.

**4단계.** 디렉터리 복사 시 `-r` 옵션의 필요성을 확인한다.

```bash
cp project project_copy
```

> `-r` 옵션이 없어 `omitting directory` 경고와 함께 실패한다.

```bash
cp -r project project_copy
```

```bash
ls project_copy
```

**5단계.** 이름을 변경하고 다른 위치로 이동한다.

```bash
mv file3.txt report.txt
```

> 원본이 있던 위치에서 이름만 `report.txt`로 바뀐다.

```bash
mv note.md project/docs/
```

> `note.md`가 `project/docs/`로 이동한다.

```bash
ls
```

**6단계.** 빈 디렉터리와 그렇지 않은 디렉터리의 삭제 차이를 확인한다.

```bash
rmdir project/logs
```

> 성공한다. `logs`는 비어 있기 때문이다.

```bash
rmdir project/docs
```

> `Directory not empty` 오류가 발생한다. `rmdir`은 빈 디렉터리만 삭제한다.

**7단계.** 확인 옵션을 사용하여 파일을 삭제하고, 디렉터리는 `-r`로 삭제한다.

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

> **확인 사항** `backup`에 txt 파일 세 개가 복사되어 있고, `project_copy`가 삭제되었다면 성공이다.
{: .prompt-tip }

---
---

# 제3절. 파일 내용 조회

---

## 3.1 조회 명령의 선택 기준

| 명령어 | 약어 풀이 | 적합한 상황 |
|---|---|---|
| `cat` | con**cat**enate | 짧은 파일 전체를 한 번에 확인 |
| `less` | — | 긴 파일을 페이지 단위로 확인 |
| `head` | head(머리) | 파일의 **앞부분**만 확인 |
| `tail` | tail(꼬리) | 파일의 **뒷부분** 또는 실시간 로그 확인 |

```bash
cat /etc/hostname          # 전체 출력
less /etc/services         # 페이지 단위 (q로 종료)
head -5 /etc/passwd        # 앞 5행
tail -5 /etc/passwd        # 뒤 5행
```

`head`와 `tail`의 `-n` 숫자(또는 `-5`처럼 축약)로 출력할 행 수를 지정한다.

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

> ### 따라 하기 1-3. 파일 내용 조회
>
> **목적** 상황에 따라 적절한 조회 명령을 선택하여 사용한다.
{: .prompt-tip }

**1단계.** 짧은 파일을 전체 출력한다.

```bash
cat /etc/hostname
```

```bash
cat -n ~/lab03/project/docs/fruits.txt
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

> `/etc/passwd`는 시스템에 등록된 사용자 계정 목록을 담은 파일이다. 각 행의 의미는 3주차-2 강의에서 학습한다.

**4단계.** 행 수를 계산한다.

```bash
wc -l /etc/passwd
```

> `wc`는 **w**ord **c**ount의 약어이며, `-l`은 행(**l**ine) 수를 센다. 시스템에 등록된 계정의 수를 확인할 수 있다.

> **확인 사항** `cat`으로 전체를, `head`·`tail`로 일부를 조회하였고, `less`에서 검색과 종료를 수행하였다면 성공이다.
{: .prompt-tip }

---
---

# 제4절. 파일 검색과 링크

---

## 4.1 파일 검색 — `find`

`find`는 디렉터리 트리를 순회하며 조건에 맞는 항목을 찾는다.

```
 find  [검색 시작 경로]  [조건]
 find         .        -name "*.txt"
```

| 조건 | 의미 |
|---|---|
| `-name "패턴"` | 이름이 일치(대소문자 구분) |
| `-iname "패턴"` | 이름이 일치(대소문자 무시) |
| `-type f` / `-type d` | 일반 파일 / 디렉터리 |
| `-size +10M` | 크기가 10MB 초과 |
| `-mtime -7` | 최근 7일 이내에 수정 |

> **[그림 3-3] 필요**
> `find . -name "*.txt"` 명령을 `find`(도구) / `.`(검색 시작 경로) / `-name "*.txt"`(조건) 세 부분으로 나누어 각 부분의 역할을 화살표로 설명한 구문 분해 다이어그램.
{: .prompt-tip }

---

## 4.2 링크의 개념

하나의 파일에 여러 개의 이름을 부여하거나, 다른 파일을 가리키는 '바로가기'를 만드는 기능을 **링크(link)** 라 한다. 링크에는 두 종류가 있다.

| 구분 | 생성 명령 | 성격 |
|---|---|---|
| **하드 링크(hard link)** | `ln 원본 링크명` | 같은 실체(데이터)에 이름을 하나 더 붙인다. 원본을 삭제해도 데이터가 유지된다. |
| **심볼릭 링크(symbolic link)** | `ln -s 원본 링크명` | 원본의 경로를 가리키는 '바로가기'이다. 원본을 삭제하면 링크가 끊어진다. |

`ln`은 li**n**k의 약어이며, `-s`는 **s**ymbolic(기호적)의 약어이다. 초보 단계에서는 윈도우의 '바로가기'와 유사한 **심볼릭 링크**를 우선 익혀 두면 충분하다. `ls -l`로 조회하면 심볼릭 링크는 맨 앞에 `l`이 표시되고 `링크명 -> 원본경로` 형태로 나타난다.

---

> ### 따라 하기 1-4. 검색과 링크
>
> **목적** `find`로 조건에 맞는 파일을 찾고, 심볼릭 링크를 만들어 동작을 확인한다.
{: .prompt-tip }

**1단계.** 실습 위치로 이동하여 이름으로 파일을 검색한다.

```bash
cd ~/lab03
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

> 디렉터리만 출력한다.

```bash
find . -type f -mtime -1
```

> 최근 하루 이내에 수정된 일반 파일만 출력한다.

**3단계.** 심볼릭 링크를 생성한다.

```bash
echo "원본 문서입니다." > original.txt
```

```bash
ln -s original.txt shortcut.txt
```

```bash
ls -l shortcut.txt
```

> 맨 앞에 `l`이 표시되고 `shortcut.txt -> original.txt` 형태로 출력되는 것을 확인한다.

```bash
cat shortcut.txt
```

> 링크를 통해 원본의 내용이 그대로 조회된다.

**4단계.** 원본을 삭제한 뒤 링크의 상태를 확인한다.

```bash
rm original.txt
```

```bash
cat shortcut.txt
```

> `No such file or directory` 오류가 발생한다. 심볼릭 링크는 원본의 경로만 가리키므로, 원본이 사라지면 링크가 끊어진다.

> **확인 사항** `find`로 조건에 맞는 파일을 찾았고, 심볼릭 링크가 원본을 가리키다가 원본 삭제 후 끊어지는 것을 확인하였다면 성공이다.
{: .prompt-tip }

---
---

# 제5절. 종합 실습 — 자료 정리와 백업

---

> **실습 시나리오**
>
> 학습자는 각 부서에서 제출한 자료가 뒤섞여 있는 상황을 정리하는 업무를 지시받았다.
>
> *"확장자별로 자료를 분류하고, 처리 내역을 보고서로 남기시오."*
>
> 본 실습은 제1절부터 제4절까지 학습한 명령어만으로 수행한다.
{: .prompt-info }

---

## 단계 1. 작업 환경 구성

```bash
mkdir -p ~/lab03/workspace
```

```bash
cd ~/lab03/workspace
```

```bash
pwd
```

---

## 단계 2. 정리 대상 자료 생성

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
echo "2026년 1분기 매출 자료" > sales_2026Q1.csv
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
mv *.docx sorted/docs/
```

```bash
ls -R sorted
```

---

## 단계 5. 처리 내역 보고서 작성

```bash
echo "===== 자료 정리 보고서 =====" > summary.txt
```

```bash
date >> summary.txt
```

```bash
echo "" >> summary.txt
```

```bash
echo "[분류된 파일 수]" >> summary.txt
```

```bash
find sorted -type f | wc -l >> summary.txt
```

```bash
echo "" >> summary.txt
```

```bash
echo "[분류된 파일 목록]" >> summary.txt
```

```bash
find sorted -type f >> summary.txt
```

```bash
cat summary.txt
```

> 보고서를 누적 기록할 때는 `>>`(추가)를 사용한다. 첫 행에서만 `>`(덮어쓰기)를 사용하여 이전 내용을 초기화하였다.

---

## 단계 6. 결과 확인

```bash
ls -R sorted
```

```bash
find sorted -type f | wc -l
```

> **종합 실습 완료 기준**
> 1. `sorted` 디렉터리 아래에 확장자별로 파일이 분류되어 있다.
> 2. `summary.txt`에 파일 수와 목록이 기록되어 있다.
>
> 본 실습에서 사용한 명령은 `mkdir`, `cd`, `pwd`, `touch`, `echo`, `ls`, `mv`, `find`, `wc`, `cat`, `date`의 11개이다.
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
| `rm: cannot remove '*.txt'` | 조건에 맞는 파일이 없어 와일드카드가 확장되지 않음 | `ls *.txt`로 대상 존재 여부를 먼저 확인한다. |
| `less`/`man`에서 나가지 못함 | 페이지 표시 상태 | **`q`** 를 누른다. |
| 심볼릭 링크에서 `No such file or directory` | 원본이 삭제되어 링크가 끊어짐 | `ls -l`로 링크가 가리키는 원본 경로를 확인한다. |

---
---

# 제7절. 요약

---

## 7.1 핵심 개념 정리

| 구분 | 요점 |
|---|---|
| 생성 | `touch`는 빈 파일, `mkdir -p`는 중간 경로까지 디렉터리를 생성한다. |
| 리다이렉션 | `>`는 덮어쓰기, `>>`는 추가이다. |
| 복사·이동·삭제 | 디렉터리를 다룰 때는 `-r`이 필수이며, 삭제 전 `ls`로 대상을 확인하는 습관이 중요하다. |
| 삭제 안전 | 터미널에는 휴지통이 없다. `rm`은 복구가 불가능하다. |
| 조회 | 짧은 파일은 `cat`, 긴 파일은 `less`, 일부는 `head`·`tail`을 사용한다. |
| 검색 | `find`는 이름·종류·크기·시각 등의 조건으로 파일을 찾는다. |
| 링크 | 심볼릭 링크는 원본 경로를 가리키는 '바로가기'이며, 원본 삭제 시 끊어진다. |

---

## 7.2 본 강의에서 학습한 명령어

| 명령어 | 약어 풀이 | 기능 |
|---|---|---|
| `touch` | — | 빈 파일 생성 |
| `mkdir` | make directory | 디렉터리 생성 |
| `rmdir` | remove directory | 빈 디렉터리 삭제 |
| `cp` | copy | 복사 |
| `mv` | move | 이동·이름 변경 |
| `rm` | remove | 삭제 |
| `cat` | concatenate | 파일 내용 출력 |
| `less` | — | 페이지 단위 조회 |
| `head` / `tail` | — | 앞부분 / 뒷부분 조회 |
| `wc` | word count | 행·단어 수 계산 |
| `find` | — | 파일 검색 |
| `ln` | link | 링크 생성 |

---

## 7.3 다음 강의 예고

3주차-2 강의에서는 본 강의에서 생성한 파일들이 **누구의 소유이며 누가 어떤 작업을 수행할 수 있는지**를 결정하는 사용자 계정과 접근 권한 체계를 학습한다. 리눅스 시스템 관리에서 가장 중요한 내용에 해당한다.
