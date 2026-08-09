---
title: 리눅스 기초 3강 - 사용자 계정 관리와 접근 권한 ★핵심
date: 2026-09-08 09:00:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - 사용자계정
  - passwd
  - shadow
  - useradd
  - chage
  - su
  - sudo
  - chmod
  - chown
  - umask
  - SetUID
  - SetGID
  - StickyBit
pin:
mermaid: false
---

> **학습 목표**
> 1. 다중 사용자 운영체제의 개념과 UID·GID 체계를 설명할 수 있다.
> 2. `/etc/passwd`, `/etc/shadow`, `/etc/group` 파일의 각 필드를 해석할 수 있다.
> 3. 계정을 생성·수정·삭제하고 비밀번호 정책을 설정할 수 있다.
> 4. `su`와 `sudo`의 차이를 이해하고 권한 위임을 수행할 수 있다.
> 5. **파일과 디렉터리에서 `rwx` 권한이 갖는 서로 다른 의미**를 설명할 수 있다.
> 6. `chmod`를 8진수와 기호 두 방식으로 사용할 수 있다.
> 7. `umask` 값으로부터 새로 생성되는 파일과 디렉터리의 권한을 계산할 수 있다.
> 8. SetUID·SetGID·스티키 비트의 동작 원리와 보안상 의미를 설명할 수 있다.
{: .prompt-info }

본 강의는 이 과정에서 가장 중요한 내용을 다룬다.

리눅스는 다중 사용자 운영체제이므로, 시스템에서 발생하는 보안 사고의 대부분은 "누가 무엇에 접근할 수 있는가"라는 설정이 잘못된 데에서 비롯된다. 침해 사고에서 공격자가 가장 먼저 시도하는 행위가 계정 탈취와 파일 권한 변조이며, 반대로 시스템을 방어하는 관리자가 가장 먼저 점검하는 항목 역시 계정과 권한이다.

리눅스마스터 2급 시험에서도 본 강의의 내용이 가장 높은 출제 비중을 차지한다. 이러한 중요성을 반영하여 본 강의는 다른 강의보다 실습 분량을 크게 확대하여 구성하였다.

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 사용자와 그룹의 개념 | 20분 |
| 제2절 | 계정 정보 파일의 구조 | 30분 |
| 제3절 | 계정 관리 명령 | 30분 |
| 제4절 | 권한 위임 — `su`와 `sudo` | 20분 |
| 제5절 | 파일 접근 권한 | 45분 |
| 제6절 | 특수 권한 | 25분 |
| 제7절 | 종합 실습 | 45분 |
| 제8·9절 | 오류 대응 및 이론 평가 | 15분 |

> **실습 환경에 관한 주의**
> 본 강의의 실습은 실제로 계정을 생성하고 파일 권한을 변경한다. 반드시 **학습용 시스템**에서만 수행하여야 하며, 운영 중인 서버에서 실행해서는 안 된다. 특히 제6절의 특수 권한 실습은 의도적으로 취약한 상태를 구성하므로, 각 실습 말미의 정리 절차를 반드시 수행하여야 한다.
{: .prompt-danger }

---
---

# 제1절. 사용자와 그룹의 개념

---

## 1.1 다중 사용자 운영체제의 이해

리눅스는 여러 사용자가 동시에 하나의 시스템을 이용할 수 있도록 설계되었다. 이러한 환경에서는 각 사용자의 자료를 서로 격리하고, 시스템의 중요 파일을 일반 사용자로부터 보호하는 장치가 필요하다.

이를 공동주택에 비유하면 다음과 같이 대응시킬 수 있다.

| 공동주택 | 리눅스 시스템 |
|---|---|
| 입주민 | 사용자(user) |
| 세대 열쇠 | 비밀번호 |
| 각 세대 | 홈 디렉터리(`/home/사용자명`) |
| 관리사무소장 | 관리자(root) |
| 동별 자치회 | 그룹(group) |
| 공용 공간 출입 권한 | 그룹 권한 |
| 세대별 출입 통제 | 파일 접근 권한 |

이러한 구조가 제대로 설정되지 않으면 다음과 같은 문제가 발생한다.

- 다른 사용자가 타인의 파일을 열람한다.
- 일반 사용자가 관리자 권한을 획득한다.
- 공용 디렉터리에 보관한 자료를 타인이 삭제한다.

본 강의에서는 위 세 가지 상황을 **실제로 재현한 뒤 이를 방지하는 방법**까지 실습한다.

---

## 1.2 UID와 GID

사용자는 `student`, `alice`와 같은 이름으로 식별되지만, 커널 내부에서는 **숫자**로 관리된다.

| 용어 | 약어 풀이 | 의미 |
|---|---|---|
| **UID** | **U**ser **ID**entifier | 사용자 식별 번호 |
| **GID** | **G**roup **ID**entifier | 그룹 식별 번호 |

파일의 소유자 정보 역시 i-노드에는 숫자로 저장되며, `ls -l`이 표시하는 이름은 `/etc/passwd`를 조회하여 변환한 결과이다.

우분투에서 UID는 다음과 같이 구획되어 있다.

| UID 범위 | 구분 | 예시 |
|---|---|---|
| **0** | **관리자(root)** | root |
| 1 ~ 99 | 배포판이 예약한 정적 시스템 계정 | daemon, bin, sys |
| 100 ~ 999 | 패키지 설치 시 생성되는 동적 시스템 계정 | www-data, sshd |
| **1000 ~ 60000** | **일반 사용자 계정** | student, alice |
| 65534 | 권한이 없는 특수 계정 | nobody |

이 경계값은 `/etc/login.defs`의 `UID_MIN`, `UID_MAX` 항목에 정의되어 있다.

> **매우 중요한 보안 개념**
> 리눅스 커널은 **UID가 0이면 계정명과 무관하게 관리자 권한을 부여한다.** 계정 이름이 `root`가 아니어도 마찬가지이다.
>
> 이 특성 때문에 침해 사고에서 공격자는 평범한 이름으로 UID 0인 계정을 은닉해 두는 수법을 사용한다. 이를 **백도어(backdoor)** 라 한다.
>
> 따라서 시스템 점검 시 "UID가 0인 계정이 root 하나뿐인가"를 확인하는 것은 필수 항목이다.
{: .prompt-danger }

---

## 1.3 기본 그룹과 보조 그룹

동일한 권한을 여러 사용자에게 부여할 때, 사용자별로 개별 설정하는 것은 비효율적이며 인원 변동 시마다 전면 수정이 필요하다. 이러한 문제를 해결하기 위해 **그룹**에 권한을 부여하고, 사용자를 그룹에 소속시키는 방식을 사용한다.

| 구분 | 정의 | 개수 |
|---|---|---|
| **기본 그룹(primary)** | `/etc/passwd`의 4번째 필드에 기록된다. 해당 사용자가 생성한 파일의 소유 그룹이 된다. | **1개** |
| **보조 그룹(secondary)** | `/etc/group`의 4번째 필드에 사용자명이 나열된다. | **여러 개** |

우분투는 **UPG(User Private Group)** 정책을 채택하여, 사용자 `alice`를 생성하면 동명의 그룹 `alice`를 함께 생성하고 이를 기본 그룹으로 지정한다.

---

> ### 따라 하기 3-1. 사용자와 그룹 정보 확인
>
> **목적** 현재 시스템에 어떤 사용자와 그룹이 존재하는지 확인하고, UID 체계를 실제 데이터로 검증한다.
{: .prompt-tip }

**1단계.** 자신의 UID와 소속 그룹을 확인한다.

```bash
id
```

> 출력에서 `uid=`(사용자 번호), `gid=`(기본 그룹), `groups=`(전체 소속 그룹)를 각각 확인한다.

```bash
groups
```

> 소속 그룹의 이름만 간략히 출력한다.

**2단계.** UID가 1000 이상인 일반 사용자를 조회한다.

```bash
awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3}' /etc/passwd
```

> `awk`는 필드 단위로 데이터를 처리하는 도구이다. `-F:`는 콜론을 구분자로 사용하라는 의미이며, `$3`은 3번째 필드(UID)를 가리킨다.

**3단계.** UID가 0인 계정을 확인한다. **보안 점검의 필수 항목이다.**

```bash
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

> **`root` 하나만 출력되어야 정상이다.** 다른 이름이 함께 출력된다면 백도어 계정을 의심하여야 한다.

**4단계.** 시스템 계정의 개수를 확인한다.

```bash
awk -F: '$3 < 1000 {print $1}' /etc/passwd | wc -l
```

> 사람이 사용하는 계정보다 시스템 계정이 훨씬 많다는 사실을 확인한다.

**5단계.** UID 경계값 설정을 확인한다.

```bash
grep -E "^(UID_MIN|UID_MAX|GID_MIN|GID_MAX)" /etc/login.defs
```

> `UID_MIN 1000`, `UID_MAX 60000`이 출력된다. 일반 사용자에게 배정할 번호의 범위를 정의한 값이다.

```bash
grep -E "SYS_UID_MAX|SYS_GID_MAX" /etc/login.defs
```

> 시스템 계정의 상한(`999`)을 정의하는 항목이다. 우분투에서는 이 두 행이 **`#`으로 시작하는 주석 상태**로 배포되며, 주석 처리된 값이 곧 기본값이다. 따라서 앞의 명령처럼 `^`(행의 시작)를 붙이면 검색되지 않으므로, 여기서는 `^`를 제외하고 조회하였다.

> **확인 사항** 자신의 UID가 1000 이상이고, UID 0인 계정이 root 하나뿐임을 확인하였다면 성공이다.
{: .prompt-tip }

---
---

# 제2절. 계정 정보 파일의 구조

---

리눅스의 계정 정보는 세 개의 텍스트 파일에 저장된다. 이 세 파일을 해석할 수 있으면 계정 관리의 절반은 이해한 것이다.

---

## 2.1 `/etc/passwd` — 계정 기본 정보

모든 사용자가 읽을 수 있으며(권한 `644`), 콜론(`:`)으로 구분된 **7개 필드**로 구성된다.

```
 student : x : 1000 : 1000 : Student Kim : /home/student : /bin/bash
    ①      ②    ③      ④        ⑤              ⑥              ⑦
```

| 번호 | 필드명 | 설명 |
|---|---|---|
| ① | 로그인명 | 사용자 계정 이름 |
| ② | 비밀번호 표시자 | **`x`** 는 실제 비밀번호가 `/etc/shadow`에 있음을 의미한다. |
| ③ | **UID** | 사용자 식별 번호 |
| ④ | **GID** | **기본 그룹**의 식별 번호 |
| ⑤ | GECOS | 실명·연락처 등 주석 정보 |
| ⑥ | 홈 디렉터리 | 로그인 직후의 작업 위치 |
| ⑦ | **로그인 셸** | 로그인 시 실행할 프로그램 |

> **7번째 필드의 의미**
> - `/bin/bash` → **로그인이 가능한 계정**
> - `/usr/sbin/nologin` 또는 `/bin/false` → **로그인이 차단된 계정**
>
> `www-data`(웹 서버), `sshd`(원격 접속)와 같은 서비스 전용 계정은 프로그램을 구동하는 데만 사용되고 대화형 로그인이 불필요하므로 `nologin`으로 설정한다.
>
> 침해 대응 시 이러한 시스템 계정의 셸이 `/bin/bash`로 변경되어 있다면 강력한 이상 징후로 판단한다.
{: .prompt-warning }

---

## 2.2 `/etc/shadow` — 비밀번호와 정책

비밀번호 해시가 저장되므로 **root만 읽을 수 있다**(권한 `640`, 소유 그룹 `shadow`). 콜론으로 구분된 **9개 필드**로 구성된다.

```
 student : $y$j9T$...해시... : 20180 : 0 : 99999 : 7 : : :
    ①              ②            ③     ④    ⑤     ⑥ ⑦ ⑧ ⑨
```

| 번호 | 필드명 | 설명 |
|---|---|---|
| ① | 로그인명 | `/etc/passwd`와 연결되는 키 |
| ② | **암호화된 비밀번호** | `!` 또는 `*`로 시작하면 **잠긴 계정** |
| ③ | 최종 변경일 | 1970-01-01부터 경과한 일수 |
| ④ | 최소 사용 기간 | 변경 후 이 일수 동안 재변경 불가 |
| ⑤ | 최대 사용 기간 | 이 일수가 경과하면 변경 필수 |
| ⑥ | 경고 일수 | 만료 며칠 전부터 경고 |
| ⑦ | 비활성 기간 | 만료 후 이 일수 경과 시 계정 비활성화 |
| ⑧ | 계정 만료일 | 이 날 이후 로그인 불가 |
| ⑨ | 예약 필드 | 사용하지 않음 |

비밀번호 해시의 접두사는 사용된 알고리즘을 나타낸다.

| 접두사 | 알고리즘 |
|---|---|
| `$1$` | MD5(취약하므로 사용 금지) |
| `$5$` | SHA-256 |
| `$6$` | SHA-512 |
| **`$y$`** | **yescrypt (Ubuntu 22.04 이후 기본값)** |

> **비밀번호를 별도 파일로 분리한 이유**
> 초기 유닉스는 `/etc/passwd`에 비밀번호 해시를 함께 저장하였다. 그러나 이 파일은 모든 사용자가 읽을 수 있으므로, 공격자가 파일을 복사하여 시간 제약 없이 해시를 분석하는 **오프라인 사전 공격**이 가능하였다.
>
> 이를 방지하기 위해 해시만 root 전용 파일로 분리하였으며, 그 파일에 '그림자'를 뜻하는 `shadow`라는 명칭을 부여하였다.
{: .prompt-info }

---

## 2.3 `/etc/group` — 그룹 정보

콜론으로 구분된 **4개 필드**로 구성된다.

```
 sudo : x : 27 : student,alice
   ①    ②   ③        ④
```

| 번호 | 필드명 | 설명 |
|---|---|---|
| ① | 그룹명 | |
| ② | 비밀번호 표시자 | `x`(실제 값은 `/etc/gshadow`) |
| ③ | GID | 그룹 식별 번호 |
| ④ | 구성원 목록 | 이 그룹을 **보조 그룹**으로 갖는 사용자 |

> **유의 사항**
> ④번 필드에는 **보조 그룹 구성원만** 나열된다. 해당 그룹을 기본 그룹으로 사용하는 사용자는 여기에 표시되지 않는데, 기본 그룹은 `/etc/passwd`의 4번째 필드로만 지정되기 때문이다.
>
> 따라서 특정 사용자가 속한 그룹 전체를 정확히 확인하려면 반드시 **`id`** 명령을 사용하여야 한다.
{: .prompt-warning }

---

## 2.4 계정 생성 정책 파일

| 파일 | 역할 |
|---|---|
| `/etc/login.defs` | UID·GID 범위, 비밀번호 기간 기본값, 기본 `UMASK` |
| `/etc/default/useradd` | `useradd`의 기본 셸·홈 경로·기본 그룹 |
| **`/etc/skel/`** | **새 계정 생성 시 홈 디렉터리로 복사되는 원본 파일** |
| `/etc/shells` | 로그인 셸로 허용되는 경로 목록 |
| `/etc/sudoers` | `sudo` 권한 위임 규칙 |

`/etc/skel`은 '뼈대'를 뜻하는 skeleton에서 유래한 명칭이다. 조직에서 신규 계정에 표준 설정을 배포하려면 이 디렉터리에 해당 파일을 배치하면 된다.

---

> ### 따라 하기 3-2. 계정 정보 파일 해석
>
> **목적** 세 파일의 필드를 직접 해석하고, 파일 권한이 서로 다르게 설정된 이유를 확인한다.
{: .prompt-tip }

**1단계.** 자신의 계정 정보를 조회한다.

```bash
grep "^$(whoami):" /etc/passwd
```

> `^`는 행의 시작을 의미하며, `$(whoami)`는 자신의 계정명으로 치환된다.
>
> 출력된 7개 필드를 노트에 옮겨 적고 각각의 의미를 기재해 본다.

**2단계.** 필요한 필드만 추출한다.

```bash
cut -d: -f1,3,4,7 /etc/passwd | head -20
```

> `cut`은 필드를 잘라내는 명령이다. `-d:`는 구분자를 콜론으로, `-f1,3,4,7`은 1·3·4·7번 필드를 지정한다. 각각 계정명·UID·GID·로그인 셸이다.

**3단계.** 로그인 가능 여부에 따라 계정을 분류한다.

```bash
grep -c "nologin\|/bin/false" /etc/passwd
```

```bash
grep "bash$" /etc/passwd
```

> 대부분의 계정이 로그인 차단 상태이며, 사람이 사용하는 계정은 소수임을 확인한다. 이는 "서비스는 각각 최소 권한의 전용 계정으로 실행한다"는 원칙의 결과이다.

**4단계.** 두 파일의 권한 차이를 확인한다.

```bash
ls -l /etc/passwd /etc/shadow /etc/group
```

> `/etc/passwd`는 `644`, `/etc/shadow`는 `640`이며 소유 그룹이 `shadow`임을 확인한다.

**5단계.** 일반 사용자로 `/etc/shadow` 접근을 시도한다.

```bash
cat /etc/shadow
```

> `Permission denied` 오류가 발생한다. **이것이 정상 동작이다.**

```bash
sudo grep "^$(whoami):" /etc/shadow
```

> 관리자 권한으로는 조회된다. 두 번째 필드가 `$y$`로 시작하는 것을 확인한다.

**6단계.** 그룹 정보를 확인한다.

```bash
grep "^sudo:" /etc/group
```

> `sudo` 그룹의 구성원이 관리자 명령을 사용할 수 있는 사용자이다.

**7단계.** 계정 생성 정책 파일을 확인한다.

```bash
grep -E "^(PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_WARN_AGE|UMASK)" /etc/login.defs
```

```bash
ls -la /etc/skel
```

> 새 계정을 생성하면 여기의 파일이 홈 디렉터리로 복사된다.

> **확인 사항** `/etc/passwd`의 7개 필드를 설명할 수 있고, `/etc/shadow`가 root 전용인 이유를 설명할 수 있다면 성공이다.
{: .prompt-tip }

---
---

# 제3절. 계정 관리 명령

---

## 3.1 계정 생성 — `useradd`

```bash
sudo useradd -m -s /bin/bash alice
sudo passwd alice
```

| 옵션 | 약어 풀이 | 기능 |
|---|---|---|
| **`-m`** | **m**ake home | **홈 디렉터리 생성 (필수적으로 지정)** |
| **`-s`** | **s**hell | 로그인 셸 지정 |
| `-d` | **d**irectory | 홈 디렉터리 경로 지정 |
| `-c` | **c**omment | GECOS(설명) 필드 |
| `-g` | **g**roup | **기본 그룹** 지정 |
| `-G` | **G**roups | **보조 그룹** 지정 |
| `-u` | **u**id | UID 직접 지정 |
| `-e` | **e**xpire | 계정 만료일 지정 |

> **`-m` 옵션을 생략하는 경우의 문제**
> 홈 디렉터리가 생성되지 않으며, 해당 계정으로 로그인하면 작업 위치가 `/`가 되어 정상적인 작업이 어렵다. 또한 `/etc/skel`의 초기 설정 파일도 복사되지 않는다. **`useradd`에는 항상 `-m`을 지정하여야 한다.**
{: .prompt-warning }

> **`useradd`와 `adduser`의 비교**
>
> | 항목 | `useradd` | `adduser` |
> |---|---|---|
> | 성격 | POSIX 표준 저수준 명령 | 데비안·우분투 계열의 대화형 상위 도구 |
> | 홈 디렉터리 | `-m` 지정 필요 | 자동 생성 |
> | 비밀번호 | `passwd`로 별도 설정 | 대화식으로 즉시 입력 |
> | 활용 | **시험에 주로 출제됨** | 실무에서 편리 |
>
> 본 강의에서는 표준 명령인 `useradd`로 학습한다.
{: .prompt-info }

---

## 3.2 계정 수정 — `usermod`

| 명령 | 기능 |
|---|---|
| `usermod -l 새이름 기존이름` | 로그인명 변경 |
| `usermod -d /new/home -m 사용자` | 홈 디렉터리 변경 및 이동 |
| `usermod -s /usr/sbin/nologin 사용자` | 로그인 셸 변경(로그인 차단) |
| **`usermod -aG 그룹 사용자`** | **보조 그룹 추가** |
| `usermod -L 사용자` / `-U 사용자` | 계정 잠금 / 해제 |
| `usermod -e 2026-12-31 사용자` | 계정 만료일 설정 |

> **`-aG`에서 `-a`를 생략할 경우의 위험**
>
> - `usermod -aG devteam alice` → 기존 보조 그룹을 **유지**하면서 devteam 추가
> - `usermod -G devteam alice` → 기존 보조 그룹을 **모두 제거**하고 devteam만 남김
>
> 관리자 계정에 `-a` 없이 실행하면 `sudo` 그룹에서 제외되어 **관리 권한을 완전히 상실한다.**
>
> `-a`는 **a**ppend(추가)의 약어이다. 그룹을 추가할 때는 **반드시 `-aG` 형태**로 사용하여야 한다.
{: .prompt-danger }

---

## 3.3 계정 삭제 — `userdel`

| 명령 | 기능 |
|---|---|
| `userdel 사용자` | 계정만 삭제(홈 디렉터리 잔존) |
| **`userdel -r 사용자`** | **홈 디렉터리와 메일 스풀까지 삭제** |

> **`-r`을 생략할 경우의 문제**
> 홈 디렉터리가 소유자 없는 상태로 남는다. 이후 다른 사용자에게 동일한 UID가 배정되면, **그 사용자가 이전 사용자의 파일을 그대로 소유하게 된다.** 계정 회수 시 반드시 확인하여야 할 사항이다.
{: .prompt-warning }

---

## 3.4 그룹 관리

| 명령 | 기능 |
|---|---|
| `groupadd 그룹명` | 그룹 생성 |
| `groupmod -n 새이름 기존이름` | 그룹명 변경 |
| `groupdel 그룹명` | 그룹 삭제(누군가의 기본 그룹이면 삭제 불가) |
| `gpasswd -a 사용자 그룹` | 그룹에 사용자 추가 |
| `gpasswd -d 사용자 그룹` | 그룹에서 사용자 제거 |

---

## 3.5 비밀번호와 정책 관리

| 명령 | 기능 |
|---|---|
| `passwd` | 자신의 비밀번호 변경 |
| `sudo passwd 사용자` | 타 사용자의 비밀번호 변경 |
| `sudo passwd -l 사용자` | 계정 잠금(**l**ock) |
| `sudo passwd -u 사용자` | 잠금 해제(**u**nlock) |
| `passwd -S 사용자` | 계정 상태 조회(**S**tatus) |
| `sudo chage -l 사용자` | 비밀번호 정책 조회(**l**ist) |
| `sudo chage -M 90 사용자` | 최대 사용 기간 90일(**M**ax) |
| `sudo chage -m 7 사용자` | 최소 사용 기간 7일(**m**in) |
| `sudo chage -W 14 사용자` | 만료 14일 전부터 경고(**W**arn) |
| `sudo chage -I 30 사용자` | 만료 후 30일 경과 시 비활성화(**I**nactive) |
| `sudo chage -E 2026-12-31 사용자` | 계정 만료일(**E**xpire) |
| `sudo chage -d 0 사용자` | 다음 로그인 시 비밀번호 변경 강제 |

`chage`는 **ch**ange **age**(기간을 변경하다)의 약어이다.

---

> ### 따라 하기 3-3. 계정 생성과 그룹 배정
>
> **목적** `useradd`로 계정을 생성하고, 생성 결과가 각 파일에 어떻게 반영되는지 추적한다.
{: .prompt-tip }

**1단계.** 실습용 그룹을 생성한다.

```bash
sudo groupadd devteam
```

```bash
grep devteam /etc/group
```

**2단계.** `-m` 옵션 없이 계정을 생성하여 문제를 확인한다.

```bash
sudo useradd testuser1
```

```bash
grep testuser1 /etc/passwd
```

```bash
ls -ld /home/testuser1
```

> 홈 디렉터리가 생성되지 않았음을 확인한다. 또한 로그인 셸이 `/bin/sh`로 지정된 것도 확인한다.

**3단계.** 옵션을 갖추어 계정을 생성한다.

```bash
sudo useradd -m -s /bin/bash -c "Alice Kim" -G devteam alice
```

```bash
sudo useradd -m -s /bin/bash -c "Bob Lee" -G devteam bob
```

```bash
sudo useradd -m -s /bin/bash -c "Carol Park" carol
```

**4단계.** 생성 결과를 확인한다.

```bash
grep -E "^(alice|bob|carol):" /etc/passwd
```

```bash
id alice
```

```bash
grep devteam /etc/group
```

> `devteam:x:1001:alice,bob` 형태로 출력되며, carol은 아직 포함되지 않았음을 확인한다.

**5단계.** 홈 디렉터리의 내용과 권한을 확인한다.

```bash
ls -la /home/alice
```

> `.bashrc`, `.profile` 등이 존재한다. 이는 `/etc/skel`에서 복사된 파일이다.

```bash
ls -ld /home/alice /home/bob /home/carol
```

> 권한이 `750`으로 설정되어 다른 일반 사용자가 내용을 조회할 수 없음을 확인한다.

**6단계.** 비밀번호를 설정한다.

```bash
sudo passwd alice
```

```bash
sudo passwd bob
```

```bash
sudo passwd carol
```

> 실습용 비밀번호로 `Linux#2026`을 사용한다. 입력 시 화면에 문자가 표시되지 않는 것은 정상이다.

**7단계.** 비밀번호 설정 여부에 따른 상태를 비교한다.

```bash
sudo passwd -S testuser1
```

```bash
sudo passwd -S alice
```

> 두 번째 항목이 `L`이면 잠김, `P`이면 정상 설정, `NP`이면 비밀번호 없음을 의미한다.

**8단계.** 보조 그룹을 추가한다.

```bash
sudo usermod -aG devteam carol
```

```bash
id carol
```

> **`-a`를 반드시 포함하였는지 확인한다.**

> **확인 사항** `id alice`에 `devteam`이 표시되고, `/home/alice`에 `/etc/skel`의 파일이 복사되어 있다면 성공이다.
{: .prompt-tip }

---

> ### 따라 하기 3-4. 비밀번호 정책과 계정 잠금
>
> **목적** `chage`로 정책을 설정하고, 계정 잠금이 `/etc/shadow`에 반영되는 형태를 확인한다.
{: .prompt-tip }

**1단계.** 현재 정책을 조회한다.

```bash
sudo chage -l alice
```

**2단계.** 비밀번호 정책을 설정한다.

```bash
sudo chage -M 90 -m 7 -W 14 alice
```

```bash
sudo chage -l alice
```

> `-M 90`은 90일마다 변경, `-m 7`은 변경 후 7일간 재변경 금지(비밀번호 이력 우회 방지), `-W 14`는 14일 전부터 경고를 의미한다.

**3단계.** 계정 만료일을 설정한다.

```bash
sudo chage -E 2026-12-31 bob
```

```bash
sudo chage -l bob | grep -i "expires"
```

> 계약직·인턴 등 종료 시점이 정해진 계정에 적용하면 회수 누락을 방지할 수 있다.

**4단계.** 다음 로그인 시 비밀번호 변경을 강제한다.

```bash
sudo chage -d 0 carol
```

```bash
sudo chage -l carol | head -3
```

> 최종 변경일을 0으로 설정하면 즉시 만료 상태가 되어, 다음 로그인 시 반드시 새 비밀번호를 설정하게 된다. 임시 비밀번호 발급 시 함께 적용하는 표준 조치이다.

**5단계.** 계정 잠금 전후의 `/etc/shadow`를 비교한다.

```bash
sudo grep "^bob:" /etc/shadow | cut -c1-25
```

```bash
sudo passwd -l bob
```

```bash
sudo grep "^bob:" /etc/shadow | cut -c1-25
```

> 비밀번호 해시 앞에 **`!`** 가 추가된 것을 확인한다. 어떤 입력을 해시하더라도 `!`로 시작하는 값은 생성될 수 없으므로 비밀번호 인증이 불가능해진다. 원본 해시는 보존되므로 해제가 가능하다.

```bash
sudo passwd -S bob
```

**6단계.** 잠금을 해제한다.

```bash
sudo passwd -u bob
```

```bash
sudo passwd -S bob
```

> **참고**: `passwd -l`은 비밀번호 인증만 차단한다. SSH 공개 키 인증은 여전히 동작할 수 있으므로, 완전한 차단이 필요하면 `usermod -s /usr/sbin/nologin`으로 셸까지 함께 제한하여야 한다.

> **확인 사항** `chage -l`의 출력이 설정값과 일치하고, 잠금 시 해시 앞에 `!`가 추가되었다가 해제 시 제거되는 것을 확인하였다면 성공이다.
{: .prompt-tip }

---
---

# 제4절. 권한 위임 — `su`와 `sudo`

---

## 4.1 두 명령의 비교

| 항목 | `su` | `sudo` |
|---|---|---|
| 약어 풀이 | **s**ubstitute **u**ser | **s**uper**u**ser **do** |
| 필요한 비밀번호 | **대상 계정(root)의 비밀번호** | **자기 자신의 비밀번호** |
| 권한 범위 | 전권 | 규칙으로 세밀하게 제한 가능 |
| 감사 기록 | 제한적 | **실행 명령이 `/var/log/auth.log`에 기록** |
| 비밀번호 공유 | root 비밀번호를 여러 명이 공유하게 됨 | 공유 불필요 |

```bash
su - alice           # alice로 전환(로그인 환경 적용)
su alice             # alice로 전환(현재 환경 유지)
sudo apt update      # 해당 명령만 관리자 권한으로 실행
sudo -i              # 관리자 셸 획득
sudo -l              # 자신에게 허용된 명령 조회
sudo -k              # 캐시된 인증 무효화
```

> **`su`와 `su -`의 차이 — 자주 출제되는 개념**
> - `su alice` → 대상 사용자로 전환하되 **현재의 작업 디렉터리와 환경 변수를 그대로 유지**한다.
> - `su - alice` → 대상 사용자의 **홈 디렉터리로 이동하고 환경 변수도 대상 사용자 기준으로 초기화**한다.
>
> 하이픈 하나의 차이지만 결과가 크게 다르다. 관리 작업 시에는 `/usr/sbin` 등의 경로가 `PATH`에 포함되어야 하므로 반드시 **`su -`** 형태를 사용하여야 한다.
{: .prompt-warning }

우분투는 보안 정책상 root 계정의 비밀번호를 설정하지 않고 잠가 두므로 `su -`가 동작하지 않는다. 이는 결함이 아니라 의도된 설계로, 권한 상승 경로를 `sudo` 하나로 일원화하여 감사를 용이하게 하기 위함이다.

---

## 4.2 `/etc/sudoers`와 `visudo`

`sudo` 권한은 `/etc/sudoers` 파일에 정의된다. 규칙의 형식은 다음과 같다.

```
 사용자   호스트=(실행권한자:그룹)   허용명령
 root     ALL=(ALL:ALL)             ALL
 %sudo    ALL=(ALL:ALL)             ALL
 backup   ALL=(root)  NOPASSWD: /usr/bin/rsync
```

- `%`로 시작하면 **그룹**을 의미한다. 우분투에서는 `sudo` 그룹 소속으로 관리 권한을 부여한다(레드햇 계열은 `wheel` 그룹).
- `NOPASSWD:`는 비밀번호 입력을 생략하게 한다. 자동화 스크립트에 한정하여, 허용 명령을 최소한으로 제한하여 사용하여야 한다.

> **`/etc/sudoers` 편집 시 반드시 지켜야 할 사항**
> 이 파일은 **반드시 `sudo visudo`로 편집하여야 한다.** `visudo`는 저장 시 문법을 검사하여 오류가 있는 파일이 반영되는 것을 차단한다.
>
> 문법이 손상된 `sudoers` 파일이 저장되면 **시스템의 모든 사용자가 `sudo`를 사용할 수 없게 되어** 복구가 매우 곤란해진다.
>
> 개별 규칙은 `/etc/sudoers.d/` 디렉터리에 파일로 분리하여 관리하는 것이 권장된다.
{: .prompt-danger }

---

> ### 따라 하기 3-5. `su`와 `sudo` 비교
>
> **목적** 두 명령의 동작 차이를 직접 확인하고, `sudo` 사용 기록이 로그에 남는 것을 관찰한다.
{: .prompt-tip }

**1단계.** `su -`로 계정을 전환한다.

```bash
su - alice
```

> alice의 비밀번호(`Linux#2026`)를 입력한다.

```bash
whoami
```

```bash
pwd
```

> `/home/alice`가 출력된다.

```bash
exit
```

**2단계.** 하이픈 없이 전환하여 차이를 확인한다.

```bash
su alice
```

```bash
pwd
```

> **원래 있던 디렉터리가 그대로 유지된다.** 이것이 `su`와 `su -`의 차이이다.

```bash
exit
```

**3단계.** 일반 사용자가 관리 권한을 갖지 못함을 확인한다.

```bash
su - alice
```

```bash
sudo cat /etc/shadow
```

> 다음과 같은 메시지가 출력된다.
>
> ```
> alice is not in the sudoers file.
> This incident has been reported to the administrator.
> ```
>
> alice는 `sudo` 그룹에 속하지 않으므로 거부되었으며, **이 시도 자체가 `/var/log/auth.log`에 기록된다.** 8단계에서 이 기록을 직접 확인한다.

```bash
exit
```

**4단계.** 자신의 sudo 권한을 확인한다.

```bash
sudo -l
```

**5단계.** `sudoers` 파일의 구조를 확인한다.

```bash
sudo grep -vE "^#|^$" /etc/sudoers
```

```bash
ls -l /etc/sudoers.d/
```

**6단계.** 특정 명령만 위임하는 규칙을 작성한다.

```bash
sudo visudo -f /etc/sudoers.d/90-devteam
```

편집기가 열리면 다음 한 행을 입력하고 저장한다.

```
%devteam ALL=(root) /usr/bin/systemctl status *, /usr/bin/journalctl
```

> `visudo -f`는 지정한 파일에 대해서도 문법 검사를 수행한다. 위 규칙은 `devteam` 그룹에 서비스 상태 조회와 로그 열람만 허용하는 것으로, **최소 권한 원칙**을 적용한 예이다.

**7단계.** 위임 결과를 확인한다.

```bash
su - alice
```

```bash
sudo systemctl status ssh
```

> 허용된 명령이므로 실행된다.

```bash
sudo cat /etc/shadow
```

> 허용되지 않은 명령이므로 거부된다. `sudo`가 전권 위임이 아니라 **범위를 지정한 위임**임을 확인할 수 있다.

```bash
exit
```

**8단계.** 인증 로그를 확인한다.

```bash
sudo tail -20 /var/log/auth.log
```

> 실행한 사용자, 시각, 작업 디렉터리, 명령이 모두 기록되어 있다. `su`보다 `sudo`가 권장되는 가장 큰 이유가 이 **감사 추적성**이다.

> **확인 사항** `su -`와 `su`의 작업 디렉터리 차이를 확인하고, 위임 규칙에 따라 일부 명령만 허용되며 그 기록이 `auth.log`에 남는 것을 확인하였다면 성공이다.
{: .prompt-tip }

---
---

# 제5절. 파일 접근 권한

---

## 5.1 권한의 3×3 구조

리눅스의 기본 권한 모델은 **세 주체**에 대하여 **세 종류의 권한**을 부여하는 방식이다.

```
     -     rwx     r-x     r--
     │      │       │       │
     │      │       │       └── others : 기타 사용자
     │      │       └────────── group  : 소유 그룹의 구성원
     │      └────────────────── user   : 소유자
     └───────────────────────── 파일 종류
```

권한 판정은 **위에서부터 한 번만** 수행된다. 접근자가 소유자이면 `user` 권한만 적용되고 그룹·기타 권한은 참조하지 않는다. 소유자가 아니면서 그룹 구성원이면 `group` 권한만 적용된다.

> **이 특성에서 비롯되는 현상**
> `chmod 077 file`과 같이 소유자 권한만 제거하면, **파일의 소유자 본인은 읽지 못하지만 다른 사용자는 읽을 수 있는** 상황이 발생한다. 다만 소유자는 언제든 `chmod`로 권한을 되돌릴 수 있으므로 완전한 차단은 아니다.
{: .prompt-warning }

---

## 5.2 파일과 디렉터리에서 `rwx`의 의미 차이

**본 강의에서 가장 중요한 내용이다.** 권한 학습에서 가장 많이 혼동하는 지점이며, 시험과 실무 양쪽에서 결정적인 개념이다.

| 권한 | 8진수 | **일반 파일**에서의 의미 | **디렉터리**에서의 의미 |
|---|---|---|---|
| `r` (**r**ead) | **4** | 파일 내용을 읽을 수 있다 | **내부 항목의 이름 목록을 조회할 수 있다(`ls`)** |
| `w` (**w**rite) | **2** | 파일 내용을 수정할 수 있다 | **내부에 항목을 생성·삭제·이름 변경할 수 있다** |
| `x` (e**x**ecute) | **1** | 파일을 프로그램으로 실행할 수 있다 | **디렉터리를 통과할 수 있다(`cd`, 내부 파일 접근)** |

디렉터리를 도서관에 비유하면 다음과 같이 이해할 수 있다.

- **`r`** = 장서 목록표를 열람할 권리
- **`x`** = 서가 안으로 들어갈 권리
- **`w`** = 책을 꽂거나 빼낼 권리

목록표만 볼 수 있고 서가에 들어갈 수 없다면 실제로 책을 꺼낼 수 없듯이, **`x` 권한이 없는 디렉터리 내부의 파일에는 접근할 수 없다.**

여기서 도출되는 세 가지 중요한 귀결은 다음과 같다.

| 번호 | 귀결 |
|---|---|
| ① | 디렉터리에 `x`가 없으면 `r`이 있어도 실질적인 활용이 불가능하다. 이름은 조회되나 각 항목의 상세 정보를 얻을 수 없다. |
| ② | 디렉터리에 `x`만 있고 `r`이 없으면, 목록은 조회할 수 없으나 **정확한 이름을 알면 접근할 수 있다.** |
| ③ | **파일의 삭제 가능 여부는 파일 자신이 아니라, 그 파일이 위치한 디렉터리의 `w` 권한이 결정한다.** |

---

## 5.3 8진수 표기법

읽기 4, 쓰기 2, 실행 1을 합산하여 표기한다.

| 기호 | 계산 | 8진수 |
|---|---|---|
| `rwx` | 4+2+1 | **7** |
| `rw-` | 4+2 | **6** |
| `r-x` | 4+1 | **5** |
| `r--` | 4 | **4** |
| `-wx` | 2+1 | 3 |
| `-w-` | 2 | 2 |
| `--x` | 1 | 1 |
| `---` | 0 | 0 |

실무에서 자주 사용하는 조합은 다음과 같다.

| 8진수 | 기호 | 용도 |
|---|---|---|
| **644** | `rw-r--r--` | 일반 문서·설정 파일 |
| **755** | `rwxr-xr-x` | 실행 파일, 공개 디렉터리 |
| **600** | `rw-------` | 개인 키, 민감 정보 |
| **700** | `rwx------` | 개인 전용 디렉터리 |
| 770 | `rwxrwx---` | 그룹 협업 디렉터리 |
| **777** | `rwxrwxrwx` | **사실상 사용 금지** |

---

## 5.4 권한 변경 — `chmod`

`chmod`는 **ch**ange **mod**e(모드를 변경하다)의 약어이다.

**방식 1: 8진수 모드**

```bash
chmod 644 report.txt
chmod 755 script.sh
chmod -R 750 /srv/project
```

**방식 2: 기호 모드**

```
 chmod [대상][연산자][권한] 파일
```

| 구분 | 기호 | 의미 |
|---|---|---|
| 대상 | `u` / `g` / `o` / `a` | user / group / others / all |
| 연산자 | `+` / `-` / `=` | 추가 / 제거 / 지정(나머지 제거) |
| 권한 | `r` / `w` / `x` / `s` / `t` | 읽기 / 쓰기 / 실행 / SetUID·SetGID / 스티키 |

```bash
chmod u+x script.sh          # 소유자에게 실행 권한 추가
chmod go-w shared.txt        # 그룹·기타의 쓰기 권한 제거
chmod a=r public.txt         # 모두에게 읽기 권한만 부여
chmod u=rwx,g=rx,o= dir      # 750과 동일
```

`-R` 옵션은 하위 항목까지 재귀적으로 적용한다. 다만 디렉터리와 파일에 동일한 값을 일괄 적용하면 부적절한 경우가 많으므로, `find`와 조합하는 방식이 권장된다.

```bash
find /srv/project -type d -exec chmod 750 {} +
find /srv/project -type f -exec chmod 640 {} +
```

---

## 5.5 소유권 변경 — `chown`, `chgrp`

| 명령 | 약어 풀이 | 기능 |
|---|---|---|
| `chown` | **ch**ange **own**er | 소유자 변경 |
| `chgrp` | **ch**ange **gr**ou**p** | 소유 그룹 변경 |

```bash
sudo chown alice file              # 소유자 변경
sudo chown alice:devteam file      # 소유자와 그룹 동시 변경
sudo chgrp devteam file            # 그룹만 변경
sudo chown -R alice:devteam /srv/data
```

> **`chown`이 root 전용인 이유**
> 일반 사용자가 자신의 파일을 타인에게 이전할 수 있다면, 디스크 할당량을 회피하거나 악의적인 SetUID 파일을 타인 소유로 넘기는 공격이 가능해진다. 이러한 이유로 `chown`은 관리자만 실행할 수 있다.
{: .prompt-warning }

---

## 5.6 기본 생성 권한 — `umask`

새로 생성되는 파일과 디렉터리의 권한은 **기준값에서 umask 값을 제거**하여 결정된다.

| 대상 | 기준값 | 이유 |
|---|---|---|
| **일반 파일** | **666** (`rw-rw-rw-`) | 생성 시점에 실행 권한을 부여하지 않는다 |
| **디렉터리** | **777** (`rwxrwxrwx`) | 진입을 위해 실행 권한이 필요하다 |

| umask | 파일 권한 | 디렉터리 권한 | 성격 |
|---|---|---|---|
| **022** | **644** | **755** | 우분투 기본값 |
| 002 | 664 | 775 | 그룹 협업용 |
| 027 | 640 | 750 | 보안 강화 |
| 077 | 600 | 700 | 최고 수준 |

> **자주 출제되는 함정**
> "umask가 022일 때 새로 생성되는 **파일**의 권한은?"의 정답은 **644**이며, **755가 아니다.** 755는 디렉터리의 답이다.
>
> 파일의 기준값이 666이고 디렉터리의 기준값이 777로 서로 다르다는 점이 핵심이다.
{: .prompt-danger }

---

> ### 따라 하기 3-6. `chmod`로 권한 변경하기
>
> **목적** 8진수 방식과 기호 방식으로 각각 권한을 설정하고, 권한 변경이 실제 동작에 미치는 영향을 확인한다.
{: .prompt-tip }

**1단계.** 실습 디렉터리와 파일을 준비한다.

```bash
mkdir -p ~/lab03 && cd ~/lab03
```

```bash
echo "일반 문서입니다." > doc.txt
```

```bash
printf '#!/bin/bash\necho "스크립트 실행 성공"\n' > run.sh
```

```bash
ls -l
```

> 두 파일 모두 실행 권한(`x`)이 없음을 확인한다.

**2단계.** 실행 권한이 없는 스크립트를 실행해 본다.

```bash
./run.sh
```

> `Permission denied` 오류가 발생한다. 내용이 스크립트이더라도 실행 권한이 없으면 실행할 수 없다.

**3단계.** 기호 모드로 실행 권한을 부여한다.

```bash
chmod u+x run.sh
```

```bash
ls -l run.sh
```

> 소유자 자리에 `x`가 추가된 것을 확인한다.

```bash
./run.sh
```

> 정상적으로 실행된다.

**4단계.** 8진수 모드로 동일한 조작을 수행한다.

```bash
chmod 644 run.sh
```

```bash
ls -l run.sh
```

> 실행 권한이 다시 제거되었다.

```bash
chmod 755 run.sh
```

```bash
ls -l run.sh
```

```bash
./run.sh
```

**5단계.** 기호 모드의 여러 형태를 사용한다.

```bash
chmod go-r doc.txt
```

```bash
ls -l doc.txt
```

```bash
chmod a=r doc.txt
```

```bash
ls -l doc.txt
```

```bash
chmod u=rw,g=r,o= doc.txt
```

```bash
ls -l doc.txt
```

> 결과가 `-rw-r-----`, 즉 `640`임을 확인한다. `o=`는 기타 사용자의 모든 권한을 제거하라는 의미이다.

**6단계.** `-R` 옵션의 부작용과 올바른 대안을 확인한다.

```bash
mkdir -p tree/sub && touch tree/a.txt tree/sub/b.txt
```

```bash
chmod -R 755 tree
```

```bash
ls -lR tree
```

> 문서 파일인 `a.txt`, `b.txt`에도 실행 권한이 부여되었다. 이는 바람직하지 않다.

```bash
find tree -type d -exec chmod 750 {} +
```

```bash
find tree -type f -exec chmod 640 {} +
```

```bash
ls -lR tree
```

> 디렉터리와 파일에 서로 다른 권한이 적용되었다.

> **확인 사항** 두 방식으로 동일한 권한을 설정할 수 있고, `find`와 조합하여 디렉터리와 파일을 구분해 적용할 수 있다면 성공이다.
{: .prompt-tip }

---

> ### 따라 하기 3-7. 디렉터리 권한의 의미 확인 ★핵심 실습
>
> **목적** 디렉터리에서 `r`과 `x`가 각각 무엇을 허용하는지 분리하여 실험하고, 파일 삭제 권한의 소재를 확인한다.
>
> **본 실습은 3강에서 가장 중요한 내용이다.** 이 부분을 이해하면 리눅스 권한 체계의 핵심을 파악한 것이다.
{: .prompt-tip }

**1단계.** 실험용 디렉터리와 파일을 준비한다.

```bash
cd ~/lab03
```

```bash
mkdir -p perm_test
```

```bash
echo "비밀 내용" > perm_test/secret.txt
```

```bash
echo "공개 내용" > perm_test/public.txt
```

```bash
chmod 644 perm_test/*.txt
```

**2단계.** 다른 사용자가 접근할 수 있도록 홈 디렉터리에 통과 권한을 부여한다.

```bash
chmod o+x ~
```

```bash
ls -ld ~
```

> 홈 디렉터리에 `x`만 부여하였다. alice는 통과는 할 수 있으나 목록은 조회할 수 없는 상태이다.

---

### 사례 A. 읽기 권한만 있고 실행 권한이 없는 경우

```bash
chmod 744 perm_test
```

```bash
ls -ld perm_test
```

> `drwxr--r--`이다. 기타 사용자에게 `r`만 부여되어 있다.

```bash
sudo -u alice ls ~/lab03/perm_test
```

> **성공한다.** 파일 이름 목록이 출력된다.

```bash
sudo -u alice ls -l ~/lab03/perm_test
```

> **실패하거나 물음표로 표시된다.** 이름은 알 수 있으나 각 항목의 상세 정보에는 접근할 수 없다.

```bash
sudo -u alice cat ~/lab03/perm_test/public.txt
```

> **실패한다.** 파일에 도달하려면 디렉터리를 통과(`x`)해야 하기 때문이다.

```bash
sudo -u alice sh -c "cd $HOME/lab03/perm_test && pwd"
```

> **실패한다.** 디렉터리에 진입하는 행위 자체가 통과 권한을 요구한다.
>
> 참고로 `cd`는 셸 내부 명령이므로 `sudo -u alice cd ...` 형태로는 실행할 수 없으며, 위와 같이 `sh -c`로 셸을 통해 실행하여야 한다.

> **사례 A의 결론**
> 디렉터리의 `r`은 **"이름 목록을 조회할 권리"** 일 뿐이며, 내부 항목에 도달할 권리가 아니다.
{: .prompt-warning }

---

### 사례 B. 실행 권한만 있고 읽기 권한이 없는 경우

```bash
chmod 711 perm_test
```

```bash
ls -ld perm_test
```

> `drwx--x--x`이다.

```bash
sudo -u alice ls ~/lab03/perm_test
```

> **실패한다.** 목록을 조회할 수 없다.

```bash
sudo -u alice cat ~/lab03/perm_test/public.txt
```

> **성공한다.** 파일 이름을 정확히 알고 있으므로 접근할 수 있다.

```bash
sudo -u alice cat ~/lab03/perm_test/secret.txt
```

> 이 역시 성공한다.

> **사례 B의 결론**
> 사례 A와 정확히 반대이다. 목록은 조회할 수 없으나 **정확한 이름을 알면 접근할 수 있다.**
>
> 이 조합(`--x`)은 "내부 구성은 은닉하되 지정한 파일에는 접근을 허용하고자 하는" 경우에 사용하며, 웹 서버의 상위 디렉터리 등에서 활용된다. 다만 파일명을 추측하여 접근할 수 있다는 위험도 함께 존재한다.
{: .prompt-warning }

---

### 사례 C. 읽기와 실행 권한이 모두 있는 경우

```bash
chmod 755 perm_test
```

```bash
sudo -u alice ls -l ~/lab03/perm_test
```

```bash
sudo -u alice cat ~/lab03/perm_test/secret.txt
```

> 목록 조회와 내용 열람이 모두 가능하다. 디렉터리에는 통상 `r`과 `x`를 함께 부여하며, 이것이 `755`와 `750`이 널리 사용되는 이유이다.

---

### 사례 D. 파일 삭제 권한의 소재

```bash
chmod 444 perm_test/public.txt
```

```bash
ls -l perm_test/public.txt
```

> `-r--r--r--`이다. 누구도 쓸 수 없는 읽기 전용 파일이다.

```bash
rm -f perm_test/public.txt
```

```bash
ls -l perm_test
```

> **삭제되었다.** 쓰기 권한이 전혀 없는 파일임에도 삭제된 것이다.

> **사례 D의 결론 — 본 강의에서 가장 중요한 개념**
> 파일을 삭제하는 행위는 "파일 내용을 수정하는 일"이 아니라 **"디렉터리에 등록된 항목을 제거하는 일"** 이다.
>
> 따라서 삭제 가능 여부는 파일 자신의 권한이 아니라 **그 파일이 위치한 디렉터리의 `w` 권한**이 결정한다.
{: .prompt-danger }

**반대 방향도 확인한다.**

```bash
echo "삭제 시험" > perm_test/target.txt
```

```bash
chmod 777 perm_test/target.txt
```

> 파일에 모든 권한을 부여하였다.

```bash
chmod 555 perm_test
```

> 디렉터리에서 쓰기 권한을 제거하였다.

```bash
rm -f perm_test/target.txt
```

> **실패한다.** 파일 권한이 `777`임에도 삭제되지 않는다.

```bash
ls -l perm_test
```

| 상황 | 결과 |
|---|---|
| 파일 444(읽기 전용) + 디렉터리 755(쓰기 가능) | **삭제됨** |
| 파일 777(전체 허용) + 디렉터리 555(쓰기 불가) | **삭제 안 됨** |

**원상 복구한다.**

```bash
chmod 755 perm_test
```

```bash
rm -f perm_test/target.txt
```

```bash
chmod 750 ~
```

> **확인 사항**
> 다음 세 문장을 자신의 표현으로 설명할 수 있다면 성공이다.
> 1. 디렉터리의 `r`은 목록 조회, `x`는 통과, `w`는 항목 생성·삭제를 허용한다.
> 2. `x` 권한이 없는 디렉터리 내부의 파일에는 접근할 수 없다.
> 3. 파일의 삭제 가능 여부는 상위 디렉터리의 `w` 권한이 결정한다.
{: .prompt-tip }

---

> ### 따라 하기 3-8. `umask` 계산 검증
>
> **목적** umask 값으로부터 생성될 권한을 예측한 뒤, 실제 결과와 대조한다.
{: .prompt-tip }

**1단계.** 현재 umask를 확인한다.

```bash
umask
```

```bash
umask -S
```

> `-S`는 **S**ymbolic(기호 형식)의 약어로, `u=rwx,g=rx,o=rx`와 같이 출력한다.

**2단계.** 결과를 먼저 예측한다.

다음 표를 노트에 작성하고 빈칸을 채운다.

| umask | 예상 파일 권한 | 예상 디렉터리 권한 |
|---|---|---|
| 022 | ? | ? |
| 002 | ? | ? |
| 027 | ? | ? |
| 077 | ? | ? |

**3단계.** 실제로 생성하여 확인한다.

```bash
mkdir -p ~/lab03/umask_test && cd ~/lab03/umask_test
```

```bash
umask 022 && touch f022 && mkdir d022
```

```bash
umask 002 && touch f002 && mkdir d002
```

```bash
umask 027 && touch f027 && mkdir d027
```

```bash
umask 077 && touch f077 && mkdir d077
```

```bash
ls -l
```

**4단계.** 정답과 대조한다.

| umask | 파일 | 디렉터리 |
|---|---|---|
| 022 | **644** `rw-r--r--` | **755** `rwxr-xr-x` |
| 002 | **664** `rw-rw-r--` | **775** `rwxrwxr-x` |
| 027 | **640** `rw-r-----` | **750** `rwxr-x---` |
| 077 | **600** `rw-------` | **700** `rwx------` |

> 어떤 umask 값을 사용하더라도 **파일에는 실행 권한이 부여되지 않는다.** 기준값이 666이기 때문이다.

**5단계.** 원래 값으로 복원한다.

```bash
umask 022
```

```bash
umask
```

**6단계.** 영구 적용 방법을 확인한다.

```bash
grep "^UMASK" /etc/login.defs
```

> 사용자별로 강화하려면 `~/.bashrc`에 `umask 027`을 추가하고 재로그인한다. 시스템 전역 기본값은 `/etc/login.defs`의 `UMASK` 항목이 결정한다.

> **확인 사항** 예측한 표와 실제 결과가 모두 일치하면 성공이다. 불일치가 있다면 기준값(파일 666, 디렉터리 777)을 다시 확인한다.
{: .prompt-tip }

---
---

# 제6절. 특수 권한

---

## 6.1 세 가지 특수 비트

기본 9비트 앞에 3비트가 추가로 존재하며, 8진수 네 자리로 표기할 때 맨 앞자리가 이에 해당한다.

| 명칭 | 8진수 | 표시 위치 | 적용 대상 | 동작 |
|---|---|---|---|---|
| **SetUID** | **4000** | 소유자의 `x` 자리 | 실행 파일 | 실행하는 동안 **파일 소유자의 권한**으로 동작 |
| **SetGID** | **2000** | 그룹의 `x` 자리 | 실행 파일 / **디렉터리** | 실행 파일: 소유 그룹 권한으로 동작<br>**디렉터리: 내부 생성 항목이 디렉터리의 그룹을 상속** |
| **스티키 비트** | **1000** | 기타의 `x` 자리 | 디렉터리 | 쓰기 권한이 있어도 **자신의 파일만 삭제 가능** |

```bash
chmod 4755 program      # SetUID  → rwsr-xr-x
chmod 2775 shared_dir   # SetGID  → rwxrwsr-x
chmod 1777 public_dir   # Sticky  → rwxrwxrwt

chmod u+s program       # 기호 방식
chmod g+s shared_dir
chmod o+t public_dir
```

---

## 6.2 SetUID의 용도와 위험

가장 대표적인 SetUID 프로그램은 `/usr/bin/passwd`이다. 사용자가 자신의 비밀번호를 변경하려면 root만 수정할 수 있는 `/etc/shadow`에 접근해야 하는데, 이를 위해 모든 사용자에게 root 비밀번호를 제공할 수는 없다. 따라서 `passwd` 실행 파일에 SetUID를 설정하여, **실행되는 순간에 한하여** root 권한으로 동작하도록 한 것이다.

그러나 동일한 원리 때문에 SetUID는 **권한 상승 공격의 핵심 통로**이기도 하다. root 소유의 SetUID 프로그램에 취약점이 존재하면 공격자는 즉시 관리자 권한을 획득할 수 있다. 따라서 다음 원칙을 준수하여야 한다.

- 필요 최소한의 프로그램에만 SetUID를 부여한다.
- 시스템의 SetUID 파일 목록을 주기적으로 점검하고 기준선과 대조한다.
- 사용자가 쓰기 가능한 파일 시스템은 `nosuid` 옵션으로 마운트하여 효력을 무력화한다.

```bash
sudo find / -perm -4000 -type f 2>/dev/null      # SetUID 목록
sudo find / -perm -2000 -type f 2>/dev/null      # SetGID 목록
```

> **셸 스크립트에는 SetUID가 적용되지 않는다**
> 리눅스 커널은 스크립트에 대한 SetUID를 **의도적으로 무시한다.** 스크립트는 인터프리터가 대신 실행하는 구조이므로 경합 조건(race condition)을 이용한 공격이 가능하기 때문이다.
>
> 따라서 "스크립트에 `chmod 4755`를 적용하면 root 권한으로 동작한다"는 진술은 **거짓**이다. 컴파일된 바이너리에만 적용된다. 시험에 자주 출제되는 함정이다.
{: .prompt-warning }

---

## 6.3 대문자 `S`와 `T`의 의미

표시 문자는 원래의 `x` 권한 유무에 따라 대소문자가 달라진다.

| 상황 | 표시 |
|---|---|
| SetUID + 실행 권한 있음 | `rws` |
| **SetUID + 실행 권한 없음** | **`rwS`** (대문자) |
| 스티키 + 실행 권한 있음 | `rwt` |
| **스티키 + 실행 권한 없음** | **`rwT`** (대문자) |

**대문자 `S`·`T`는 특수 비트는 설정되었으나 해당 자리의 실행 권한이 없어 실제로는 동작하지 않는 상태**를 의미하며, 대부분 설정 오류에 해당한다.

---

> ### 따라 하기 3-9. 특수 권한 실습
>
> **목적** SetUID·SetGID·스티키 비트의 동작을 각각 확인하고, SetUID의 보안 위험을 실증한다.
>
> **주의**: 4단계는 의도적으로 취약한 파일을 생성한다. **6단계 정리 절차를 반드시 수행하여야 한다.**
{: .prompt-tip }

**1단계.** 시스템의 SetUID 프로그램을 조사한다.

```bash
ls -l /usr/bin/passwd
```

> `-rwsr-xr-x`에서 소유자 실행 자리의 `s`를 확인한다.

```bash
sudo find /usr/bin /usr/sbin -perm -4000 -type f 2>/dev/null
```

> 목록이 예상보다 적다는 점에 주목한다. 꼭 필요한 프로그램에만 부여되어 있어야 한다.

**2단계.** SetGID 디렉터리의 그룹 상속을 확인한다.

```bash
sudo mkdir -p /srv/team
```

```bash
sudo chown root:devteam /srv/team
```

```bash
sudo chmod 770 /srv/team
```

```bash
sudo -u alice touch /srv/team/before.txt
```

```bash
ls -l /srv/team
```

> `before.txt`의 소유 그룹이 `alice`(개인 그룹)임을 확인한다. 이 상태에서는 같은 팀의 bob이 파일을 수정할 수 없다.

```bash
sudo chmod 2770 /srv/team
```

```bash
ls -ld /srv/team
```

> `drwxrws---`로 변경되어 그룹 실행 자리에 `s`가 표시된다.

```bash
sudo -u alice touch /srv/team/after.txt
```

```bash
ls -l /srv/team
```

> **`after.txt`의 소유 그룹이 `devteam`이다.** SetGID를 설정한 디렉터리 내부에 생성되는 항목은 디렉터리의 그룹을 상속한다.

**3단계.** 스티키 비트의 효과를 확인한다.

```bash
sudo mkdir /srv/opendir
```

```bash
sudo chmod 777 /srv/opendir
```

```bash
sudo -u alice touch /srv/opendir/alice_note.txt
```

```bash
sudo -u bob rm -f /srv/opendir/alice_note.txt
```

```bash
ls -l /srv/opendir
```

> **bob이 alice의 파일을 삭제하였다.** 디렉터리에 쓰기 권한이 있으면 소유자와 무관하게 삭제가 가능하다는 원리(따라 하기 3-7 사례 D)가 그대로 재현된 것이다.

```bash
sudo chmod 1777 /srv/opendir
```

```bash
ls -ld /srv/opendir
```

> `drwxrwxrwt`로 변경되어 마지막 문자가 `t`가 되었다.

```bash
sudo -u alice touch /srv/opendir/alice_note.txt
```

```bash
sudo -u bob rm -f /srv/opendir/alice_note.txt
```

> **실패한다.** 스티키 비트가 타인의 파일 삭제를 차단한다.

```bash
ls -ld /tmp
```

> `/tmp` 디렉터리가 동일한 설정을 사용하고 있음을 확인한다.

**4단계.** SetUID의 보안 위험을 실증한다.

```bash
sudo cp /bin/cat /srv/mycat
```

```bash
sudo chmod 4755 /srv/mycat
```

```bash
ls -l /srv/mycat
```

```bash
sudo -u alice cat /etc/shadow
```

> 일반 사용자로는 접근이 거부된다.

```bash
sudo -u alice /srv/mycat /etc/shadow
```

> **접근이 허용된다.** 일반 사용자 alice가 시스템의 모든 비밀번호 해시를 열람하였다.

> **본 실험이 시사하는 바**
> `/srv/mycat`은 정상적인 `cat` 프로그램이며 취약한 코드가 아니다. 그럼에도 **root 소유 + SetUID**라는 조합만으로 시스템의 가장 중요한 파일이 노출되었다.
>
> 실제 침해 사고에서는 침입에 성공한 공격자가 이러한 파일을 은닉해 두고 재침입 시 관리자 권한을 획득하는 데 활용한다. 이를 지속성(persistence) 확보라 한다.
>
> 이것이 시스템 점검 시 SetUID 파일 목록을 최우선으로 확인하는 이유이다.
{: .prompt-danger }

**5단계.** 스크립트에는 SetUID가 적용되지 않음을 확인한다.

```bash
sudo sh -c 'printf "#!/bin/bash\nid\n" > /srv/whoisroot.sh'
```

```bash
sudo chmod 4755 /srv/whoisroot.sh
```

```bash
ls -l /srv/whoisroot.sh
```

> 권한 표시에는 `s`가 존재한다.

```bash
sudo -u alice /srv/whoisroot.sh
```

> 출력은 다음과 같은 형태이다.
>
> ```
> uid=1001(alice) gid=1001(alice) groups=1001(alice),1002(devteam)
> ```
>
> **`euid=` 항목이 아예 표시되지 않는다는 점**에 주목한다. `id` 명령은 실제 사용자(uid)와 유효 사용자(euid)가 **서로 다를 때에만** `euid=`를 출력한다. 즉 이 출력은 권한 상승이 전혀 일어나지 않았음을 의미한다.
>
> 4단계에서 SetUID가 적용된 `/srv/mycat`은 root 권한으로 동작하였으나, 동일하게 `chmod 4755`를 적용한 이 스크립트는 그렇지 않다. 커널이 스크립트의 SetUID를 의도적으로 무시하기 때문이다.
>
> 비교를 위해 다음을 실행하면 SetUID가 적용된 경우의 출력을 확인할 수 있다.

```bash
sudo cp /usr/bin/id /srv/myid && sudo chmod 4755 /srv/myid && sudo -u alice /srv/myid
```

> 이번에는 `euid=0(root)`가 함께 표시된다. **컴파일된 바이너리에만 SetUID가 적용된다**는 사실이 확인된다. 이 파일도 7단계에서 함께 삭제한다.

**6단계.** 대문자 `S` 표시를 확인한다.

```bash
sudo touch /srv/nox
```

```bash
sudo chmod 4644 /srv/nox
```

```bash
ls -l /srv/nox
```

> `-rwSr--r--`로 대문자 `S`가 표시된다. SetUID 비트는 설정되었으나 실행 권한이 없어 효력이 없는 상태이다.

**7단계.** 정리 — **반드시 수행하여야 한다.**

```bash
sudo rm -f /srv/mycat /srv/myid /srv/whoisroot.sh /srv/nox
```

```bash
sudo rm -rf /srv/opendir
```

```bash
sudo find / -perm -4000 -type f 2>/dev/null | grep -vE "^/usr|^/snap" || echo "표준 경로 외 SetUID 파일 없음"
```

> `/usr` 아래는 배포판이 정상적으로 배치한 SetUID 프로그램의 표준 경로이므로 제외한다. `/snap`도 함께 제외하는데, snap 패키지가 설치되어 있으면 `/snap/snapd/*/usr/lib/snapd/snap-confine`과 같은 정상 SetUID 파일이 존재하기 때문이다.
>
> **아무것도 출력되지 않고 "표준 경로 외 SetUID 파일 없음"이 표시되어야 정리가 완료된 것이다.** 다른 경로가 출력된다면 실습 파일이 남아 있는 것이므로 삭제한다.

> **확인 사항** SetGID의 그룹 상속, 스티키 비트의 삭제 차단, SetUID의 권한 상승을 각각 확인하였고, **정리 절차를 완료하였다면** 성공이다.
{: .prompt-tip }

---
---

# 제7절. 종합 실습 — 프로젝트 협업 환경 구축

---

> **실습 시나리오**
>
> 학습자는 소프트웨어 개발 기업 '코드메이커스'의 시스템 관리자이다. 신규 프로젝트 착수에 따라 개발팀 2명(`dev1`, `dev2`)과 디자인팀 1명(`designer1`)이 협업할 환경을 구축하여야 한다.
>
> 관리 부서에서 제시한 요구사항은 다음과 같다.
>
> | 번호 | 요구사항 |
> |---|---|
> | 요구사항 1 | 개발팀 전용 디렉터리는 개발자만 접근할 수 있어야 한다. |
> | 요구사항 2 | 디자이너는 개발팀 디렉터리에 접근할 수 없어야 한다. |
> | 요구사항 3 | 같은 팀 구성원끼리는 서로의 파일을 수정할 수 있어야 한다. |
> | 요구사항 4 | 공용 자료실은 누구나 파일을 등록할 수 있되, **타인의 파일은 삭제할 수 없어야** 한다. |
> | 요구사항 5 | 모든 계정은 90일마다 비밀번호를 변경하여야 한다. |
{: .prompt-info }

---

## 단계 1. 계정 생성

```bash
sudo useradd -m -s /bin/bash -c "개발자1" dev1
```

```bash
sudo useradd -m -s /bin/bash -c "개발자2" dev2
```

```bash
sudo useradd -m -s /bin/bash -c "디자이너1" designer1
```

```bash
sudo passwd dev1
```

```bash
sudo passwd dev2
```

```bash
sudo passwd designer1
```

> 실습용 비밀번호로 `Project#X2026`을 사용한다.

**생성 결과를 확인한다.**

```bash
grep -E "^(dev1|dev2|designer1):" /etc/passwd
```

```bash
id dev1
```

---

## 단계 2. 그룹 생성 및 배정

```bash
sudo groupadd developers
```

```bash
sudo groupadd designers
```

```bash
sudo usermod -aG developers dev1
```

```bash
sudo usermod -aG developers dev2
```

```bash
sudo usermod -aG designers designer1
```

**확인한다.**

```bash
grep -E "^(developers|designers):" /etc/group
```

```bash
groups dev1
```

```bash
groups designer1
```

---

## 단계 3. 요구사항 1·2·3 구현 — 개발팀 전용 디렉터리

```bash
sudo mkdir -p /srv/projectx/dev_area
```

```bash
sudo chown root:developers /srv/projectx/dev_area
```

```bash
sudo chmod 2770 /srv/projectx/dev_area
```

> `2770`의 각 자리는 다음을 의미한다.
>
> | 자리 | 값 | 의미 |
> |---|---|---|
> | 특수 | 2 | SetGID — 내부 생성 항목이 developers 그룹을 상속 |
> | 소유자 | 7 | root: 읽기·쓰기·실행 |
> | 그룹 | 7 | developers: 읽기·쓰기·실행 |
> | 기타 | 0 | 그 외 사용자: 접근 불가 |

```bash
ls -ld /srv/projectx/dev_area
```

**요구사항 1 검증 — 개발자의 접근 가능 여부**

```bash
sudo -u dev1 touch /srv/projectx/dev_area/dev1_code.py
```

```bash
sudo -u dev1 ls -l /srv/projectx/dev_area
```

> 정상적으로 수행된다.

**요구사항 2 검증 — 디자이너의 접근 차단 여부**

```bash
sudo -u designer1 ls /srv/projectx/dev_area
```

> `Permission denied`로 차단된다.

**요구사항 3 검증 — 팀 구성원 간 파일 수정 가능 여부**

```bash
ls -l /srv/projectx/dev_area
```

> 파일의 소유 그룹이 `developers`임을 확인한다. SetGID 설정의 결과이다.

```bash
sudo chmod g+w /srv/projectx/dev_area/dev1_code.py
```

```bash
sudo -u dev2 sh -c 'echo "# dev2가 추가한 주석" >> /srv/projectx/dev_area/dev1_code.py'
```

```bash
cat /srv/projectx/dev_area/dev1_code.py
```

> dev2가 dev1의 파일을 수정하였다.

---

## 단계 4. 디자인팀 디렉터리 구성

```bash
sudo mkdir -p /srv/projectx/design_area
```

```bash
sudo chown root:designers /srv/projectx/design_area
```

```bash
sudo chmod 2770 /srv/projectx/design_area
```

```bash
sudo -u designer1 touch /srv/projectx/design_area/logo.png
```

```bash
sudo -u dev1 ls /srv/projectx/design_area
```

> 이번에는 개발자가 차단된다. 양 팀의 자료가 상호 분리되었다.

---

## 단계 5. 요구사항 4 구현 — 공용 자료실

먼저 스티키 비트 없이 구성하여 문제 상황을 확인한다.

```bash
sudo mkdir -p /srv/projectx/public
```

```bash
sudo chmod 777 /srv/projectx/public
```

```bash
sudo -u dev1 sh -c 'echo "dev1의 자료" > /srv/projectx/public/dev1_doc.txt'
```

```bash
sudo -u designer1 rm -f /srv/projectx/public/dev1_doc.txt
```

```bash
ls -l /srv/projectx/public
```

> designer1이 dev1의 파일을 삭제하였다. **요구사항 4에 위배된다.**

**스티키 비트를 적용한다.**

```bash
sudo chmod 1777 /srv/projectx/public
```

```bash
ls -ld /srv/projectx/public
```

> `drwxrwxrwt`로 변경되었다.

```bash
sudo -u dev1 sh -c 'echo "dev1의 자료" > /srv/projectx/public/dev1_doc.txt'
```

```bash
sudo -u designer1 rm -f /srv/projectx/public/dev1_doc.txt
```

> 삭제가 차단된다. 요구사항 4가 충족되었다.

```bash
sudo -u designer1 sh -c 'echo "designer1의 자료" > /srv/projectx/public/design_doc.txt'
```

```bash
sudo -u designer1 rm -f /srv/projectx/public/design_doc.txt
```

> 자신의 파일은 정상적으로 삭제할 수 있다.

---

## 단계 6. 요구사항 5 구현 — 비밀번호 정책

```bash
sudo chage -M 90 -m 7 -W 14 dev1
```

```bash
sudo chage -M 90 -m 7 -W 14 dev2
```

```bash
sudo chage -M 90 -m 7 -W 14 designer1
```

```bash
sudo chage -l dev1
```

> `Maximum number of days between password change : 90` 항목을 확인한다.

---

## 단계 7. 점검 보고서 작성

```bash
mkdir -p ~/mission03 && cd ~/mission03
```

```bash
{
  echo "===== 프로젝트 협업 환경 점검 보고서 ====="
  echo "점검 일시: $(date '+%Y-%m-%d %H:%M')"
  echo
  echo "[1] UID 0 계정 (root만 존재하여야 함)"
  awk -F: '$3 == 0 {print "  - " $1}' /etc/passwd
  echo
  echo "[2] 프로젝트 계정 (계정명 / UID / 로그인 셸)"
  grep -E "^(dev1|dev2|designer1):" /etc/passwd | cut -d: -f1,3,7 | sed 's/^/  - /'
  echo
  echo "[3] 그룹 구성"
  grep -E "^(developers|designers):" /etc/group | sed 's/^/  - /'
  echo
  echo "[4] 디렉터리 권한"
  ls -ld /srv/projectx/* | sed 's/^/  /'
  echo
  echo "[5] 비밀번호 정책 (dev1)"
  sudo chage -l dev1 | grep -i "maximum" | sed 's/^/  /'
  echo
  echo "[6] 현재 umask"
  echo "  - $(umask)"
} > report.txt
```

```bash
cat report.txt
```

---

## 단계 8. 요구사항 충족 여부 검증표 작성

```bash
{
  echo "===== 요구사항 충족 검증 ====="
  echo
  echo "요구사항 1. 개발팀만 dev_area 접근"
  sudo -u dev1 ls /srv/projectx/dev_area > /dev/null 2>&1 \
    && echo "  충족 (dev1 접근 성공)" || echo "  미충족"
  echo
  echo "요구사항 2. 디자이너는 dev_area 접근 불가"
  sudo -u designer1 ls /srv/projectx/dev_area > /dev/null 2>&1 \
    && echo "  미충족 (접근됨)" || echo "  충족 (접근 차단됨)"
  echo
  echo "요구사항 3. 파일 그룹 상속 (SetGID)"
  ls -l /srv/projectx/dev_area | grep developers > /dev/null \
    && echo "  충족 (developers 그룹 상속)" || echo "  미충족"
  echo
  echo "요구사항 4. 공용 자료실 스티키 비트"
  [ -k /srv/projectx/public ] \
    && echo "  충족 (스티키 비트 설정됨)" || echo "  미충족"
  echo
  echo "요구사항 5. 비밀번호 90일 정책"
  sudo chage -l dev1 | grep -q "90" \
    && echo "  충족 (90일 설정됨)" || echo "  미충족"
} > verify.txt
```

```bash
cat verify.txt
```

> **점검 스크립트를 작성할 때의 유의 사항**
> 요구사항 4의 판정에 `ls -ld ... | grep "t"`와 같은 방식을 사용해서는 안 된다. 경로 문자열인 `/srv/projec**t**x/public`에 이미 `t`가 포함되어 있으므로, 스티키 비트가 없어도 항상 "충족"으로 판정되기 때문이다.
>
> 셸이 제공하는 **`[ -k 경로 ]`** 검사는 스티키 비트의 설정 여부만을 직접 확인하므로 이러한 오판이 발생하지 않는다. 같은 방식으로 `[ -u 경로 ]`는 SetUID를, `[ -g 경로 ]`는 SetGID를 검사한다.
>
> 점검 도구는 "우연히 통과하는 조건"을 배제하도록 작성하여야 한다는 점을 기억한다.
{: .prompt-warning }

---

## 단계 9. 정리 — 반드시 수행

```bash
sudo rm -rf /srv/projectx /srv/team
```

```bash
sudo userdel -r dev1
```

```bash
sudo userdel -r dev2
```

```bash
sudo userdel -r designer1
```

```bash
sudo userdel -r alice
```

```bash
sudo userdel -r bob
```

```bash
sudo userdel -r carol
```

```bash
sudo userdel testuser1
```

```bash
sudo groupdel developers
```

```bash
sudo groupdel designers
```

```bash
sudo groupdel devteam
```

```bash
sudo rm -f /etc/sudoers.d/90-devteam
```

**정리 결과를 확인한다.**

```bash
grep -E "^(dev1|dev2|designer1|alice|bob|carol|testuser1):" /etc/passwd || echo "실습 계정 전부 삭제 완료"
```

```bash
sudo find / -xdev -nouser 2>/dev/null | head
```

> 아무 항목도 출력되지 않아야 정상이다. 출력된다면 소유자 없는 파일이 잔존한 것이다.

> **종합 실습 완료 기준**
> 1. 계정 3개와 그룹 2개가 생성되고 상호 배정되었다.
> 2. SetGID 디렉터리에서 그룹 상속이 확인되었다.
> 3. 스티키 비트로 타인의 파일 삭제가 차단되었다.
> 4. 비밀번호 정책이 적용되었다.
> 5. 검증표에서 요구사항 5개가 모두 '충족'으로 표시되었다.
> 6. **정리 절차를 완료하였다.**
{: .prompt-tip }

---
---

# 제8절. 자주 발생하는 오류와 대응 방법

---

| 화면에 출력된 메시지 | 원인 | 대응 방법 |
|---|---|---|
| `Permission denied` | 권한 부족 | `ls -l`로 권한을 확인한 뒤 `chmod` 또는 `sudo`를 적용한다. |
| `Operation not permitted` | root 전용 작업 시도 | `sudo`를 붙인다(`chown` 등). |
| `is not in the sudoers file` | `sudo` 그룹 미소속 | `sudo usermod -aG sudo 사용자` |
| `user does not exist` | 계정 미존재 | `grep 계정명 /etc/passwd`로 확인한다. |
| `group does not exist` | 그룹 미존재 | `groupadd`로 먼저 생성한다. |
| `cannot remove: Operation not permitted` | 스티키 비트가 삭제를 차단 | 자신의 파일인지 확인한다. |
| 디렉터리 목록은 보이나 파일이 열리지 않음 | 디렉터리에 `x` 권한 없음 | `chmod +x 디렉터리` |
| 그룹에 추가했으나 권한이 적용되지 않음 | 세션에 반영되지 않음 | **로그아웃 후 재로그인**한다. |

> **가장 빈번한 혼란 사례**
> `usermod -aG`로 그룹에 추가하였음에도 권한이 적용되지 않는 경우가 있다. **그룹 소속은 로그인 시점에 결정되므로** 이미 로그인해 있는 세션에는 즉시 반영되지 않는다. 로그아웃 후 재로그인하면 해결된다.
{: .prompt-warning }

---
---

# 제9절. 이론 평가

---

**문항 1.** 리눅스에서 관리자 권한을 갖는 계정의 UID로 옳은 것은?

① **0** ② 1 ③ 100 ④ 1000

---

**문항 2.** `/etc/passwd` 파일의 필드 개수와 4번째 필드의 의미를 바르게 짝지은 것은?

① 5개 – 홈 디렉터리
② **7개 – 기본 그룹의 GID**
③ 7개 – 보조 그룹
④ 9개 – UID

---

**문항 3.** 암호화된 비밀번호가 실제로 저장되는 파일과, 계정이 잠겼음을 나타내는 표시로 옳은 것은?

① `/etc/passwd` – 비밀번호 필드의 `x`
② **`/etc/shadow` – 해시 앞의 `!`**
③ `/etc/group` – 비밀번호 필드의 `*`
④ `/etc/gshadow` – 빈 필드

---

**문항 4.** `useradd`로 계정을 생성하면서 홈 디렉터리를 함께 생성하는 옵션은?

① `-d` ② **`-m`** ③ `-g` ④ `-c`

---

**문항 5.** 사용자를 기존 보조 그룹에서 제외하지 않고 새 그룹만 추가하는 명령은?

① `usermod -G devteam alice`
② **`usermod -aG devteam alice`**
③ `groupmod -a alice devteam`
④ `gpasswd -A alice devteam`

---

**문항 6.** `umask` 값이 `022`일 때 새로 생성되는 **일반 파일**의 권한은?

① **644** ② 664 ③ 755 ④ 777

---

**문항 7.** `chmod 2775 shared` 명령으로 설정되는 특수 권한과 그 효과로 옳은 것은?

① SetUID – 실행 시 소유자 권한으로 동작
② **SetGID – 디렉터리 내 생성 항목이 디렉터리의 그룹을 상속**
③ 스티키 비트 – 자신의 파일만 삭제 가능
④ SetUID + 스티키 비트

---

**문항 8.** `/tmp` 디렉터리에 설정되어, 쓰기 권한이 있어도 자신의 파일만 삭제할 수 있게 하는 특수 권한은?

① SetUID ② SetGID ③ **스티키 비트** ④ ACL

---

**문항 9.** 권한이 `444`인 파일이 권한 `755`인 디렉터리에 존재한다. 해당 디렉터리의 소유자가 이 파일을 삭제할 수 있는가?

① 파일에 쓰기 권한이 없으므로 삭제할 수 없다.
② **디렉터리에 쓰기 권한이 있으므로 삭제할 수 있다.**
③ root만 삭제할 수 있다.
④ 스티키 비트가 설정되어야 삭제할 수 있다.

---

**문항 10.** 디렉터리 권한이 `--x`(실행 권한만 존재)일 때 가능한 동작으로 옳은 것은?

① `ls`로 내부 목록을 조회할 수 있다.
② **정확한 파일명을 알면 해당 파일에 접근할 수 있다.**
③ 새 파일을 생성할 수 있다.
④ 어떠한 동작도 수행할 수 없다.

---
---

# 제10절. 요약

---

## 10.1 핵심 개념 정리

| 구분 | 요점 |
|---|---|
| 사용자 식별 | 커널은 사용자를 UID로 식별하며, **UID 0은 계정명과 무관하게 관리자**이다. |
| 계정 파일 | `/etc/passwd`는 7개 필드, `/etc/shadow`는 9개 필드로 구성된다. 비밀번호 해시를 분리한 것은 오프라인 공격을 방지하기 위함이다. |
| 계정 관리 | 생성은 `useradd -m -s`, 수정은 `usermod`(보조 그룹은 반드시 `-aG`), 삭제는 `userdel -r`로 수행한다. |
| 권한 위임 | `sudo`는 자기 비밀번호로 인증하고 범위를 제한할 수 있으며 기록이 남는다. `sudoers` 편집은 반드시 `visudo`로 한다. |
| **권한의 의미** | **파일과 디렉터리에서 `rwx`의 의미가 다르다.** 디렉터리의 `r`은 목록 조회, `x`는 통과, `w`는 항목 생성·삭제이다. |
| **삭제 권한** | **파일의 삭제 가능 여부는 상위 디렉터리의 `w` 권한이 결정한다.** |
| umask | 기준값은 파일 666, 디렉터리 777이다. umask 022이면 파일 644, 디렉터리 755가 된다. |
| 특수 권한 | SetUID(4000)·SetGID(2000)·스티키(1000). SetUID는 권한 상승의 정당한 수단인 동시에 가장 위험한 공격 통로이며, 셸 스크립트에는 적용되지 않는다. |

---

## 10.2 본 강의에서 학습한 명령어

### 계정 관리

| 명령어 | 약어 풀이 | 기능 |
|---|---|---|
| `useradd -m -s` | user add | 계정 생성 |
| `usermod -aG` | user modify | 계정 수정·그룹 추가 |
| `userdel -r` | user delete | 계정 삭제 |
| `groupadd` / `groupdel` | — | 그룹 생성·삭제 |
| `passwd` | — | 비밀번호 설정·잠금 |
| `chage` | change age | 비밀번호 기간 정책 |
| `id` / `groups` | — | 소속 확인 |
| `su -` / `sudo` | substitute user / superuser do | 권한 전환·위임 |
| `visudo` | — | sudoers 안전 편집 |

### 권한 관리

| 명령어 | 약어 풀이 | 기능 |
|---|---|---|
| `chmod` | change mode | 권한 변경 |
| `chown` | change owner | 소유자 변경 |
| `chgrp` | change group | 소유 그룹 변경 |
| `umask` | — | 기본 생성 권한 설정 |

---

## 10.3 반드시 기억해야 할 다섯 가지

| 번호 | 내용 |
|---|---|
| 1 | **UID 0은 관리자이다.** 계정 이름과 무관하다. |
| 2 | **디렉터리의 `r`은 목록 조회, `x`는 통과이다.** 둘 다 있어야 정상적으로 활용할 수 있다. |
| 3 | **파일 삭제 권한은 상위 디렉터리의 `w`가 결정한다.** |
| 4 | **umask 022 → 파일 644, 디렉터리 755.** 기준값이 서로 다르다. |
| 5 | **SetUID(4)·SetGID(2)·스티키(1).** SetUID는 편리하나 위험하며, 스크립트에는 적용되지 않는다. |

---

## 10.4 다음 강의 예고

제4강에서는 지금까지 사용한 명령들을 실제로 해석하고 실행하는 **셸**의 동작 원리와, 설정 파일을 수정하기 위한 **편집기** 사용법을 학습한다.
