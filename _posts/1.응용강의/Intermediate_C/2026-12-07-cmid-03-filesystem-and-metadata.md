---
title: 중급 C 프로그래밍 3강 - 파일 시스템과 메타데이터
date: 2026-12-07 09:00:00 +0900
categories:
  - 1.응용강의
  - 중급 C 프로그래밍
tags:
  - C언어
  - 시스템프로그래밍
  - 리눅스
  - stat
  - 아이노드
  - 하드링크
  - 심벌릭링크
  - 디렉터리
  - 파일권한
  - 원자성
pin:
mermaid: false
---

> **학습 목표**
> 1. **아이노드**의 개념을 설명하고, 이름과 실체가 분리되어 있음을 실험으로 보일 수 있다.
> 2. `stat`·`fstat`·`lstat` 셋을 구분해 알맞게 쓸 수 있다.
> 3. 파일의 종류를 `S_IS*` 매크로로 판정할 수 있다.
> 4. 권한과 소유자를 읽고, 권한 검사 순서를 설명할 수 있다.
> 5. `st_size`와 `st_blocks`의 차이를 설명할 수 있다.
> 6. `atime`·`mtime`·`ctime`이 각각 언제 바뀌는지 실험으로 확인할 수 있다.
> 7. **하드 링크와 심벌릭 링크**의 차이를 설명하고 직접 만들 수 있다.
> 8. `unlink`가 실제로 무엇을 지우는지 설명할 수 있다.
> 9. 디렉터리를 재귀로 순회하여 `du`를 만들 수 있다.
> 10. **깨지지 않는 파일 갱신**을 임시 파일과 `rename`으로 구현할 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

2강에서는 파일의 **내용**을 다루었습니다. 이번에는 파일의 **정보**를 다룹니다.

| 2강 | 3강 |
|---|---|
| 무엇이 들어 있는가 | **누구 것인가, 얼마나 큰가, 언제 바뀌었나** |
| `read`·`write` | **`stat`** |
| 파일 하나 | **디렉터리 나무 전체** |

이 강의를 마치면 **`ls -l`·`du`·`find`가 실제로 무엇을 하는지** 알게 되고, 그중 둘을 직접 만들게 됩니다.

그리고 한 가지 오해를 깹니다.

> **파일은 "이름과 내용의 짝"이 아닙니다.**

이 문장이 이번 강의 전체를 관통합니다.

**기준 교재: APUE 3판 4장(Files and Directories).**

이 강의는 **3회차 분량**(모두 합쳐 약 485분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제6절 | 아이노드·`stat`·권한·시각 | 195분 |
| **2회차** | 제7절 ~ 제11절 | 링크·디렉터리 순회·원자적 갱신 | 180분 |
| **3회차** | 실습문제 | 스스로 해 보기 | 110분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 이름과 실체는 따로 있다 — 아이노드 | 30분 |
| 제2절 | `stat`·`fstat`·`lstat` | 40분 |
| 제3절 | 파일의 종류 | 20분 |
| 제4절 | 권한과 소유자 | 45분 |
| 제5절 | 크기와 블록 | 25분 |
| 제6절 | 세 가지 시각 | 35분 |
| 제7절 | 하드 링크와 심벌릭 링크 | 45분 |
| 제8절 | `unlink`가 지우는 것 | 30분 |
| 제9절 | 디렉터리 순회 — `du` 만들기 | 50분 |
| 제10절 | 깨지지 않는 파일 갱신 | 40분 |
| 제11절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 디렉터리**는 `~/cmid/lab03`을 사용합니다. 이번 강의는 **VM 한 대**(`c-srv`)에서 진행합니다.

```bash
mkdir -p ~/cmid/lab03 && cd ~/cmid/lab03
```

---

## 제1절. 이름과 실체는 따로 있다 — 아이노드

### 1.1 파일 시스템의 세 층

파일 시스템은 세 가지를 따로 관리합니다.

| 층 | 이름 | 무엇을 담는가 |
|---|---|---|
| 1 | **디렉터리 항목** | **이름** → 아이노드 번호 |
| 2 | **아이노드(inode)** | 크기·권한·소유자·시각·**자료가 있는 위치** |
| 3 | 자료 블록 | 실제 내용 |

```text
  디렉터리 "lab03"
  ┌──────────────┬──────────┐
  │ 이름         │ 아이노드 │
  ├──────────────┼──────────┤
  │ hello.txt    │  34421   │──┐
  │ hard.txt     │  34421   │──┤     아이노드 34421
  │ soft.txt     │  35786   │  │   ┌────────────────┐
  └──────────────┴──────────┘  └──▶│ 권한 0644      │
                                   │ 소유자 1000    │
                                   │ 크기 16        │──▶ [자료 블록]
                                   │ 링크 수 2      │
                                   └────────────────┘
```

**핵심은 이것입니다.**

| 사실 | 뜻 |
|---|---|
| 이름은 아이노드를 **가리킬 뿐**이다 | 이름 여러 개가 같은 파일을 가리킬 수 있다 |
| 아이노드에는 **이름이 없다** | 파일 자신은 자기 이름을 모른다 |
| 이름을 지워도 | 아이노드는 **다른 이름이 남아 있으면** 살아 있다 |

### 1.2 아이노드 번호 보기

```bash
ls -li
```

```text
34421 -rw-r--r-- 1 student student 16 Dec  7 09:04 hello.txt
```

맨 앞 숫자가 아이노드 번호입니다.

> 아이노드 번호는 **파일 시스템 안에서만** 유일합니다. 다른 파티션에는 같은 번호가 있을 수 있습니다. 그래서 파일을 정확히 지목하려면 **장치 번호(`st_dev`)와 아이노드 번호(`st_ino`)를 함께** 보아야 합니다.
{: .prompt-info }

개념 문서는 [`inode(7)`](https://man7.org/linux/man-pages/man7/inode.7.html)에 정리되어 있습니다.

---

## 제2절. `stat`·`fstat`·`lstat`

### 2.1 세 가지 형태

```c
#include <sys/stat.h>

int stat(const char *pathname, struct stat *statbuf);
int fstat(int fd, struct stat *statbuf);
int lstat(const char *pathname, struct stat *statbuf);
```

| 함수 | 무엇으로 지목 | 심벌릭 링크를 만나면 |
|---|---|---|
| `stat` | 경로 | **따라간다** — 대상의 정보 |
| `fstat` | **열린 서술자** | (해당 없음) |
| `lstat` | 경로 | **따라가지 않는다** — 링크 자신의 정보 |

**세 가지 구분이 이번 강의에서 가장 자주 실수하는 지점**입니다.

| 하려는 일 | 써야 할 함수 |
|---|---|
| 이미 연 파일의 정보 | **`fstat`** — 경로가 바뀌어도 안전하다 |
| 디렉터리를 훑으며 | **`lstat`** — 링크를 따라가면 같은 파일을 두 번 센다 |
| 링크가 가리키는 대상 | `stat` |

### 2.2 `struct stat`의 주요 항목

| 항목 | 뜻 |
|---|---|
| `st_dev` | 어느 장치에 있나 |
| **`st_ino`** | **아이노드 번호** |
| **`st_mode`** | **종류 + 권한** (한 값에 둘 다 들어 있다) |
| **`st_nlink`** | **하드 링크 수** |
| `st_uid`·`st_gid` | 소유자·그룹 |
| **`st_size`** | **논리적 크기**(바이트) |
| **`st_blocks`** | **실제 블록 수**(512바이트 단위) |
| `st_blksize` | 권장 입출력 크기 |
| `st_atim`·`st_mtim`·`st_ctim` | 세 가지 시각 |

### 2.3 직접 만들어 보기

`statinfo.c`의 핵심 부분입니다.

```c
    for (i = 1; i < argc; i++) {
        /* lstat 은 심벌릭 링크를 따라가지 않는다 */
        if (lstat(argv[i], &st) == -1) {
            fprintf(stderr, "%s: %s\n", argv[i], strerror(errno));
            continue;
        }

        perm_string(st.st_mode, perm);
        pw = getpwuid(st.st_uid);
        gr = getgrgid(st.st_gid);

        printf("%-22s %s\n",    "종류",          type_name(st.st_mode));
        printf("%-22s %04o (%s)\n", "권한",      st.st_mode & 07777, perm);
        printf("%-22s %ld\n",   "아이노드 번호", (long) st.st_ino);
        printf("%-22s %ld\n",   "하드 링크 수",  (long) st.st_nlink);
        ...
    }
```

```bash
gcc -Wall -Wextra -std=gnu17 statinfo.c -o statinfo
```

```bash
printf '안녕하세요\n' > hello.txt
```

```bash
./statinfo hello.txt
```

```text
===== hello.txt =====
종류                 보통 파일
권한                 0644 (rw-r--r--)
아이노드 번호    34421
하드 링크 수      1
소유자 UID          1000 (student)
그룹 GID             1000 (student)
논리적 크기       16 바이트
실제 블록 수      8 (512바이트 단위) = 4096 바이트
권장 입출력 크기 4096
마지막 읽기(atime) 2026-12-07 09:04:27
마지막 수정(mtime) 2026-12-07 09:04:27
정보 변경(ctime)   2026-12-07 09:04:27
```

**"안녕하세요"는 다섯 글자인데 크기가 16바이트**입니다. 한글이 UTF-8에서 글자당 3바이트이고(5 × 3 = 15), 개행 1바이트가 더해졌기 때문입니다. 1강 11.2절에서 확인한 것과 같은 이야기입니다.

### 2.4 UID를 이름으로 바꾸기

`st_uid`는 숫자입니다. 사람이 읽는 이름은 별도로 찾아야 합니다.

```c
#include <pwd.h>
#include <grp.h>

pw = getpwuid(st.st_uid);       /* 실패하면 NULL */
gr = getgrgid(st.st_gid);
```

```c
printf("%s\n", pw ? pw->pw_name : "?");
```

> **`NULL` 검사를 반드시 하십시오.** 계정이 삭제된 뒤 남은 파일은 UID만 있고 이름이 없습니다. `ls -l`이 이름 대신 숫자를 보여 주는 경우가 바로 그것입니다.
{: .prompt-warning }

---

## 제3절. 파일의 종류

### 3.1 일곱 가지

`st_mode`에서 종류를 판정합니다.

```c
static const char *type_name(mode_t m)
{
    if (S_ISREG(m))  return "보통 파일";
    if (S_ISDIR(m))  return "디렉터리";
    if (S_ISLNK(m))  return "심벌릭 링크";
    if (S_ISCHR(m))  return "문자 장치";
    if (S_ISBLK(m))  return "블록 장치";
    if (S_ISFIFO(m)) return "FIFO(이름 있는 파이프)";
    if (S_ISSOCK(m)) return "소켓";
    return "알 수 없음";
}
```

| 매크로 | 종류 | `ls -l`의 첫 글자 | 이 과정에서 |
|---|---|---|---|
| `S_ISREG` | 보통 파일 | `-` | 2강 |
| `S_ISDIR` | 디렉터리 | `d` | 제9절 |
| `S_ISLNK` | 심벌릭 링크 | `l` | 제7절 |
| `S_ISCHR` | 문자 장치 | `c` | `/dev/null` |
| `S_ISBLK` | 블록 장치 | `b` | 디스크 |
| `S_ISFIFO` | FIFO | `p` | **6강** |
| `S_ISSOCK` | 소켓 | `s` | **9강** |

**뒤의 두 가지가 이 과정의 예고편입니다.** 파이프와 소켓도 **파일처럼 다루어진다**는 것이 UNIX의 설계입니다. 6강과 9강에서 그 덕을 보게 됩니다.

### 3.2 확인해 보기

```bash
./statinfo /dev/null /tmp /dev/sda 2>/dev/null | grep 종류
```

```text
종류                 문자 장치
종류                 디렉터리
```

> **`S_ISLNK`는 `lstat`으로 얻은 `st_mode`에서만 참이 됩니다.** `stat`을 쓰면 링크를 따라가 버려 절대 심벌릭 링크로 보이지 않습니다.
{: .prompt-danger }

---

## 제4절. 권한과 소유자

### 4.1 권한 아홉 비트

`st_mode`의 아래쪽에 권한이 들어 있습니다.

| 대상 | 읽기 | 쓰기 | 실행 |
|---|---|---|---|
| 소유자(user) | `S_IRUSR` 0400 | `S_IWUSR` 0200 | `S_IXUSR` 0100 |
| 그룹(group) | `S_IRGRP` 0040 | `S_IWGRP` 0020 | `S_IXGRP` 0010 |
| 그 밖(other) | `S_IROTH` 0004 | `S_IWOTH` 0002 | `S_IXOTH` 0001 |

`ls -l`이 보여 주는 `rw-r--r--` 형태로 바꾸는 코드입니다.

```c
/* perm_string: 0644 같은 값을 rw-r--r-- 로 바꾼다 */
static void perm_string(mode_t m, char *s)
{
    static const char *rwx = "rwxrwxrwx";
    int i;

    for (i = 0; i < 9; i++)
        s[i] = (m & (1 << (8 - i))) ? rwx[i] : '-';

    if (m & S_ISUID) s[2] = (s[2] == 'x') ? 's' : 'S';
    if (m & S_ISGID) s[5] = (s[5] == 'x') ? 's' : 'S';
    if (m & S_ISVTX) s[8] = (s[8] == 'x') ? 't' : 'T';
    s[9] = '\0';
}
```

**비트를 위에서부터 훑는 방식**에 주목하십시오. `1 << (8 - i)`가 0400·0200·0100·0040… 순서로 내려갑니다. 1부 3강에서 배운 비트 연산이 이렇게 쓰입니다.

### 4.2 특수한 세 비트

| 비트 | 값 | 붙는 자리 | 하는 일 |
|---|---|---|---|
| **setuid** | 04000 | 소유자 실행 자리 | 실행할 때 **소유자의 권한**을 얻는다 |
| **setgid** | 02000 | 그룹 실행 자리 | 그룹 권한을 얻는다 / 디렉터리에서는 그룹 상속 |
| **sticky** | 01000 | 그 밖 실행 자리 | 디렉터리에서 **자기 파일만 지울 수 있다** |

실제로 확인해 보십시오.

```bash
stat -c '%A %a %n' /usr/bin/passwd /tmp
```

```text
-rwsr-xr-x 4755 /usr/bin/passwd
drwxrwxrwt 1777 /tmp
```

| 파일 | 왜 그런가 |
|---|---|
| `passwd`의 `s` | 암호를 바꾸려면 `/etc/shadow`를 써야 하는데, 그 파일은 root만 쓸 수 있다 |
| `/tmp`의 `t` | 누구나 쓸 수 있지만 **남의 파일은 지우지 못하게** |

> **setuid는 보안의 핵심 지점입니다.** 일반 사용자가 잠시 root 권한으로 도는 프로그램이므로, 여기에 버그가 있으면 곧바로 권한 상승 취약점이 됩니다. 1부 10강에서 배운 입력 검증이 이런 프로그램에서 가장 중요해집니다.
{: .prompt-warning }

### 4.3 권한 검사의 순서

커널은 다음 순서로 **한 번만** 판정합니다.

| 순서 | 조건 | 결과 |
|---|---|---|
| 1 | 실효 UID가 0(root) | **대부분 통과** |
| 2 | 실효 UID == `st_uid` | **소유자 비트**로 판정하고 끝 |
| 3 | 실효 GID가 그룹과 일치 | **그룹 비트**로 판정하고 끝 |
| 4 | 그 밖 | **other 비트**로 판정 |

> **2번에서 끝난다는 점이 중요합니다.**
> 권한이 `0466`(`r--rw-rw-`)인 파일을 **소유자는 쓸 수 없습니다.** 소유자 비트가 `r--`이므로 거기서 판정이 끝나고, 그룹·other의 `rw-`는 보지도 않습니다. "나는 소유자니까 당연히 되겠지"라는 짐작이 틀리는 경우입니다.
{: .prompt-danger }

경로 해석 규칙 전체는 [`path_resolution(7)`](https://man7.org/linux/man-pages/man7/path_resolution.7.html)에 있습니다. **디렉터리의 실행 권한(`x`)은 "통과할 수 있다"는 뜻**이라는 점도 함께 익혀 두십시오.

### 4.4 `umask`

2강 2.5절에서 본 그대로입니다. **실제로 만들어지는 권한은 `mode & ~umask`** 입니다.

```bash
umask
```

```text
0022
```

```bash
touch p1 ; stat -c '%a %n' p1
```

```text
644 p1
```

`touch`는 0666으로 요청하지만 `umask` 022가 그룹·other의 쓰기 비트를 걷어내 0644가 됩니다.

### 4.5 바꾸기

| 하려는 일 | 함수 |
|---|---|
| 권한 바꾸기 | [`chmod(2)`](https://man7.org/linux/man-pages/man2/chmod.2.html) · `fchmod` |
| 소유자 바꾸기 | [`chown(2)`](https://man7.org/linux/man-pages/man2/chown.2.html) · `fchown` · `lchown` |
| umask 바꾸기 | [`umask(2)`](https://man7.org/linux/man-pages/man2/umask.2.html) |

```c
if (chmod("secret.key", 0600) == -1)
    fprintf(stderr, "chmod: %s\n", strerror(errno));
```

> **개인 키 파일은 반드시 0600이어야 합니다.** 14강에서 TLS 인증서를 만들 때 이 한 줄이 필요합니다. OpenSSH를 비롯한 많은 프로그램은 키 파일 권한이 너무 열려 있으면 **아예 사용을 거부**합니다.
{: .prompt-tip }

---

## 제5절. 크기와 블록

### 5.1 두 가지 크기

2강 6.3절에서 만든 희소 파일을 다시 봅니다.

```bash
printf 'A' > sp.dat ; truncate -s 1M sp.dat
```

```bash
./statinfo sp.dat | grep -E "논리적 크기|실제 블록"
```

```text
논리적 크기       1048576 바이트
실제 블록 수      8 (512바이트 단위) = 4096 바이트
```

```bash
du -h sp.dat
```

```text
4.0K	sp.dat
```

| 항목 | 뜻 | 대응하는 명령 |
|---|---|---|
| `st_size` | **논리적 크기** — 파일이 얼마나 커 보이는가 | `ls -l` |
| `st_blocks × 512` | **실제 디스크 사용량** | **`du`** |

**`du`가 하는 일이 바로 `st_blocks`를 더하는 것입니다.** 제9절에서 직접 만듭니다.

### 5.2 `st_blocks`는 언제나 512바이트 단위

> **`st_blksize`가 4096이라고 해서 `st_blocks`의 단위도 4096인 것은 아닙니다.**
> `st_blocks`의 단위는 **언제나 512바이트**로 고정되어 있습니다. 위 예에서 8 × 512 = 4096바이트가 나온 것이 그 결과입니다. [`stat(2)`](https://man7.org/linux/man-pages/man2/stat.2.html)에 명시되어 있습니다.
{: .prompt-warning }

| 값 | 단위 | 의미 |
|---|---|---|
| `st_blocks` | **512바이트 고정** | 실제로 할당된 블록 수 |
| `st_blksize` | 파일 시스템이 알려 줌 | **입출력에 알맞은 크기**(2강 7절의 그 값) |

### 5.3 작은 파일이 4096바이트를 쓰는 이유

16바이트짜리 `hello.txt`도 `st_blocks`가 8(= 4096바이트)이었습니다. 파일 시스템은 **블록 단위로만 공간을 나누어 주기** 때문입니다. 1바이트를 담아도 블록 하나를 차지합니다.

이것이 "작은 파일이 많으면 디스크가 빨리 찬다"는 말의 근거입니다.

---

## 제6절. 세 가지 시각

### 6.1 무엇이 다른가

| 이름 | 언제 바뀌나 | 흔한 오해 |
|---|---|---|
| `st_atim` (**a**ccess) | **내용을 읽을 때** | — |
| `st_mtim` (**m**odify) | **내용을 고칠 때** | — |
| `st_ctim` (**c**hange) | **아이노드가 바뀔 때** | **"생성 시각"이 아니다** |

> **`ctime`은 creation time이 아닙니다.**
> **c**hange time, 즉 **아이노드의 정보가 바뀐 시각**입니다. 권한·소유자·링크 수가 바뀌어도 갱신됩니다. 리눅스의 전통적인 `stat`에는 **생성 시각이 아예 없습니다**(일부 파일 시스템에서 `statx`로 얻을 수 있습니다).
{: .prompt-danger }

### 6.2 실험으로 확인

`timestamps.c`는 네 가지 동작을 2초 간격으로 수행하며 시각을 찍습니다.

```c
    fd = open(path, O_WRONLY | O_APPEND);
    write(fd, "second\n", 7);
    close(fd);
    snapshot(path, "내용을 쓴 뒤");

    sleep(2);
    fd = open(path, O_RDONLY);
    if (read(fd, buf, sizeof buf) < 0)
        fprintf(stderr, "read: %s\n", strerror(errno));
    close(fd);
    snapshot(path, "읽은 뒤");

    sleep(2);
    if (chmod(path, 0600) == -1)
        fprintf(stderr, "chmod: %s\n", strerror(errno));
    snapshot(path, "권한을 바꾼 뒤");
```

```bash
./timestamps
```

```text
만든 직후    atime 11:04:27   mtime 11:04:27   ctime 11:04:27
내용을 쓴 뒤 atime 11:04:27   mtime 11:04:29   ctime 11:04:29
읽은 뒤       atime 11:04:31   mtime 11:04:29   ctime 11:04:29
권한을 바꾼 뒤 atime 11:04:31   mtime 11:04:29   ctime 11:04:33
```

**한 줄씩 읽어 보십시오.**

| 동작 | atime | mtime | ctime |
|---|---|---|---|
| 쓰기 | 그대로 | **바뀜** | **바뀜** |
| 읽기 | **바뀜** | 그대로 | 그대로 |
| `chmod` | 그대로 | 그대로 | **바뀜** |

- **쓰면 `mtime`과 `ctime`이 함께 바뀝니다.** 내용이 바뀌면 크기 같은 정보도 바뀌므로 아이노드도 갱신되기 때문입니다.
- **`chmod`는 `ctime`만** 바꿉니다. 내용은 건드리지 않았기 때문입니다.
- 이 성질 때문에 **`ctime`은 되돌리기 어렵습니다.** `mtime`은 `touch -d`로 조작할 수 있지만 `ctime`은 그렇지 않아, 침해 사고 조사에서 중요한 단서가 됩니다.

### 6.3 `atime`은 믿기 어렵습니다

읽을 때마다 디스크에 쓰는 것은 낭비이므로, 요즘 리눅스는 기본적으로 **`relatime`** 방식으로 마운트합니다.

| 방식 | 언제 `atime`을 갱신하나 |
|---|---|
| `strictatime` | 읽을 때마다 |
| **`relatime`**(기본값) | `atime`이 `mtime`보다 오래되었거나, 하루가 지났을 때만 |
| `noatime` | 갱신하지 않음 |

```bash
findmnt -no OPTIONS /
```

**그래서 "마지막으로 언제 읽었나"를 `atime`으로 판단하면 안 됩니다.** 위 실험에서 `atime`이 제대로 바뀐 것은 직전에 쓰기가 있어 `relatime` 조건에 걸렸기 때문입니다.

---

> **▶ 여기서부터 2회차 — 링크·디렉터리 순회·원자적 갱신**
> 제7절 ~ 제11절, 약 180분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제7절. 하드 링크와 심벌릭 링크

### 7.1 두 가지 링크

| 구분 | 하드 링크 | 심벌릭 링크 |
|---|---|---|
| 만드는 함수 | [`link(2)`](https://man7.org/linux/man-pages/man2/link.2.html) | [`symlink(2)`](https://man7.org/linux/man-pages/man2/symlink.2.html) |
| 정체 | **같은 아이노드의 또 다른 이름** | **경로 문자열을 담은 별도의 파일** |
| 아이노드 | 원본과 **같다** | **다르다** |
| `st_nlink` | **늘어난다** | 원본은 그대로 |
| 원본을 지우면 | **멀쩡하다** | **끊어진다** |
| 다른 파일 시스템 | **불가** | 가능 |
| 디렉터리 | **불가**(보통) | 가능 |

### 7.2 실험

`links_demo.c`는 파일 하나를 만들고 두 종류의 링크를 건 뒤, 원본 이름을 지웁니다.

```c
    if (link("orig.txt", "hard.txt") == -1)
        fprintf(stderr, "link: %s\n", strerror(errno));
    if (symlink("orig.txt", "soft.txt") == -1)
        fprintf(stderr, "symlink: %s\n", strerror(errno));
```

```bash
./links_demo
```

```text
--- 셋을 만든 직후 ---
orig.txt     lstat: 아이노드 35378    링크수 2  크기 6  종류 보통 파일
hard.txt     lstat: 아이노드 35378    링크수 2  크기 6  종류 보통 파일
soft.txt     lstat: 아이노드 35786    링크수 1  크기 8  종류 심벌릭 링크
              stat: 아이노드 35378    링크수 2  크기 6  <- 대상

--- 원본 이름(orig.txt)을 지운 뒤 ---
orig.txt     (없음)
hard.txt     lstat: 아이노드 35378    링크수 1  크기 6  종류 보통 파일
soft.txt     lstat: 아이노드 35786    링크수 1  크기 8  종류 심벌릭 링크

--- 각 이름으로 읽어 보면 ---
hard.txt -> "hello
"soft.txt -> 열 수 없음: No such file or directory
```

**세 가지를 확인하십시오.**

| 관찰 | 뜻 |
|---|---|
| `orig.txt`와 `hard.txt`의 아이노드가 **35378로 같다** | 이름만 둘이고 파일은 하나다 |
| 링크 수가 **2 → 1**로 줄었다 | 이름 하나가 사라졌을 뿐 |
| `soft.txt`는 아이노드가 **다르고 크기가 8** | 별도의 파일이며, 내용은 `"orig.txt"` 여덟 글자 |
| 원본 삭제 후 `hard.txt`는 **읽히고** `soft.txt`는 **못 읽는다** | 결정적 차이 |

### 7.3 심벌릭 링크의 크기 = 경로 문자열의 길이

`soft.txt`의 크기가 정확히 8이었던 것은 우연이 아닙니다. `"orig.txt"`가 여덟 글자이기 때문입니다.

```bash
ln -sf f1.txt link_to_f1 ; ls -l link_to_f1 | awk '{print $5}'
```

```text
6
```

`"f1.txt"`는 여섯 글자입니다. **심벌릭 링크의 내용은 경로 문자열 그 자체**이며, 읽으려면 [`readlink(2)`](https://man7.org/linux/man-pages/man2/readlink.2.html)를 씁니다.

```c
char target[PATH_MAX];
ssize_t n = readlink("soft.txt", target, sizeof target - 1);

if (n != -1) {
    target[n] = '\0';           /* readlink 는 '\0' 을 붙여 주지 않는다 */
    printf("가리키는 곳: %s\n", target);
}
```

> **`readlink`는 끝에 `'\0'`을 붙여 주지 않습니다.** 1부 10강 4.2절에서 본 `strncpy`의 함정과 같은 유형입니다. 반환값만큼의 자리에 직접 넣어야 합니다.
{: .prompt-danger }

### 7.4 디렉터리의 링크 수

```bash
mkdir -p nl/sub1 nl/sub2 ; stat -c '%h %n' nl nl/sub1
```

```text
4 nl
2 nl/sub1
```

| 디렉터리 | 링크 수 | 왜 |
|---|---|---|
| 하위가 없는 `sub1` | **2** | 자기 이름 + 자기 안의 `.` |
| 하위가 둘인 `nl` | **4** | 자기 이름 + `.` + 하위 둘의 `..` |

> **규칙: 디렉터리의 링크 수 = 2 + 하위 디렉터리 개수**
{: .prompt-tip }

`.`과 `..`이 실제로 **하드 링크**이기 때문에 생기는 결과입니다. 이 사실을 알면 제9절에서 디렉터리를 순회할 때 왜 `.`과 `..`을 반드시 건너뛰어야 하는지 자연히 이해됩니다.

---

## 제8절. `unlink`가 지우는 것

### 8.1 이름을 지우는 것이지 파일을 지우는 것이 아니다

함수 이름이 `delete`가 아니라 [`unlink(2)`](https://man7.org/linux/man-pages/man2/unlink.2.html)인 데는 이유가 있습니다. **디렉터리에서 이름 하나를 떼어 내는 것**이 하는 일의 전부입니다.

아이노드가 실제로 사라지는 조건은 **두 가지가 동시에** 만족될 때입니다.

| 조건 | 확인하는 값 |
|---|---|
| 이름이 하나도 남지 않았다 | `st_nlink == 0` |
| **열어 둔 프로세스가 없다** | 커널이 따로 센다 |

### 8.2 열린 채로 지워 보기

```c
    /* 이름만 지운다 */
    if (unlink("temp.dat") == -1)
        fprintf(stderr, "unlink: %s\n", strerror(errno));

    printf("이름으로 접근: %s\n",
           access("temp.dat", F_OK) == 0 ? "된다" : "안 된다");

    fstat(fd, &st);
    printf("지운 뒤   : 링크 수 %ld, 크기 %lld\n",
           (long) st.st_nlink, (long long) st.st_size);

    /* 서술자로는 여전히 읽고 쓸 수 있다 */
    lseek(fd, 0, SEEK_SET);
    n = read(fd, buf, sizeof buf - 1);
```

```bash
./unlink_open
```

```text
지우기 전 : 링크 수 1, 크기 41
이름으로 접근: 안 된다
지운 뒤   : 링크 수 0, 크기 41
서술자로 읽기: 이 자료는 아직 살아 있습니다
close 하는 순간 자료가 사라집니다.
```

**링크 수가 0인데도 자료를 읽었습니다.** 이름은 사라졌지만 아이노드는 살아 있습니다.

### 8.3 이 성질의 쓸모

| 쓰임 | 어떻게 |
|---|---|
| **안전한 임시 파일** | 만들고 곧바로 `unlink`. 프로그램이 끝나면 **자동으로 사라진다** |
| 남이 못 보게 | 이름이 없으므로 다른 프로세스가 열 수 없다 |
| 정리 누락 방지 | 비정상 종료해도 커널이 회수한다 |

```c
fd = open("tmpfile", O_RDWR | O_CREAT | O_EXCL, 0600);
unlink("tmpfile");          /* 이름은 지우고 서술자만 들고 간다 */
/* 이제 fd 로 마음껏 쓰다가, 끝나면 자동으로 회수된다 */
```

### 8.4 이 성질이 일으키는 사고

**"파일을 지웠는데 디스크 여유가 늘지 않는" 현상**의 원인이기도 합니다.

| 상황 | 결과 |
|---|---|
| 거대한 로그 파일을 `rm` | 서버 프로세스가 **아직 열고 있으면** 공간이 반환되지 않는다 |
| 확인 방법 | `lsof \| grep deleted` |
| 해결 | 프로세스를 다시 시작하거나, `truncate`로 내용을 비운다 |

> **로그 파일은 `rm`이 아니라 비우는 것이 옳습니다.**
> ```bash
> : > /var/log/big.log
> ```
> 이러면 아이노드는 그대로 두고 크기만 0으로 만들므로, 열고 있는 프로세스도 계속 쓸 수 있고 공간도 즉시 반환됩니다.
{: .prompt-tip }

---

## 제9절. 디렉터리 순회 — `du` 만들기

### 9.1 디렉터리를 읽는 함수

디렉터리는 파일이지만 `read`로 읽지 않습니다. 전용 함수를 씁니다.

| 함수 | 하는 일 |
|---|---|
| [`opendir(3)`](https://man7.org/linux/man-pages/man3/opendir.3.html) | 디렉터리를 연다 |
| [`readdir(3)`](https://man7.org/linux/man-pages/man3/readdir.3.html) | 항목을 하나씩 — 없으면 `NULL` |
| `closedir(3)` | 닫는다 |

```c
DIR *dp = opendir(path);
struct dirent *ent;

while ((ent = readdir(dp)) != NULL)
    printf("%s\n", ent->d_name);

closedir(dp);
```

> **`readdir`이 돌려주는 순서는 정해져 있지 않습니다.** 이름순도 아니고 만든 순서도 아닙니다. 정렬이 필요하면 모두 읽어 직접 정렬해야 합니다. `ls`가 정렬해서 보여 주는 것은 `ls`가 하는 일이지 파일 시스템이 하는 일이 아닙니다.
{: .prompt-warning }

### 9.2 `du` 만들기

```c
/* walk: path 를 훑는다. 돌려주는 값은 그 아래 전체의 바이트 수 */
static long long walk(const char *path, int depth)
{
    struct stat st;
    DIR *dp;
    struct dirent *ent;
    char child[PATH_MAX];
    long long sum = 0;

    /* lstat 이어야 한다. stat 이면 심벌릭 링크의 대상을 세어 두 번 센다 */
    if (lstat(path, &st) == -1) {
        fprintf(stderr, "%s: %s\n", path, strerror(errno));
        return 0;
    }

    sum += (long long) st.st_blocks * 512;   /* 실제로 쓰는 디스크 공간 */

    if (S_ISDIR(st.st_mode)) {
        dirs++;
        dp = opendir(path);
        if (dp == NULL) {
            fprintf(stderr, "%s: %s\n", path, strerror(errno));
            return sum;
        }
        while ((ent = readdir(dp)) != NULL) {
            /* . 과 .. 를 건너뛰지 않으면 무한 재귀에 빠진다 */
            if (strcmp(ent->d_name, ".") == 0 || strcmp(ent->d_name, "..") == 0)
                continue;
            if (snprintf(child, sizeof child, "%s/%s", path, ent->d_name)
                    >= (int) sizeof child) {
                fprintf(stderr, "경로가 너무 깁니다: %s/%s\n", path, ent->d_name);
                continue;
            }
            sum += walk(child, depth + 1);
        }
        closedir(dp);

        if (depth <= 1)                      /* du 처럼 위쪽만 보여 준다 */
            printf("%10lld\t%s\n", sum / 1024, path);
    } else if (S_ISREG(st.st_mode)) {
        files++;
    } else {
        others++;
    }

    return sum;
}
```

**네 가지 판단이 들어 있습니다.**

| 판단 | 이유 |
|---|---|
| **`lstat`을 쓴다** | `stat`이면 심벌릭 링크를 따라가 같은 자료를 두 번 센다 |
| **`.`과 `..`을 건너뛴다** | 7.4절에서 본 대로 진짜 하드 링크다. 안 걸러 내면 **무한 재귀** |
| `snprintf` 반환값 검사 | 경로가 잘리면 엉뚱한 파일을 센다(1부 10강 3.2절) |
| `st_blocks × 512` | 논리적 크기가 아니라 **실제 사용량**(제5절) |

### 9.3 진짜 `du`와 비교

**정답이 있는 프로그램은 정답과 맞춰 보는 것이 가장 확실한 검증입니다.**

```bash
mkdir -p tree/a/b
head -c 5000 /dev/zero | tr '\0' 'x' > tree/f1.txt
head -c 100  /dev/zero | tr '\0' 'y' > tree/a/f2.txt
head -c 200  /dev/zero | tr '\0' 'z' > tree/a/b/f3.txt
ln -sf f1.txt tree/link_to_f1
```

```bash
./mydu tree
```

```text
        16	tree/a
        28	tree
---
합계 28672 바이트 (28 KiB)
보통 파일 3개, 디렉터리 3개, 그 밖 1개
```

```bash
du -d 1 tree
```

```text
16	tree/a
28	tree
```

```bash
du -s --block-size=1 tree
```

```text
28672	tree
```

**정확히 일치합니다.** 바이트 단위까지 같습니다.

### 9.4 그런데 하드 링크가 있으면 달라집니다

```bash
ln tree/f1.txt tree/a/hardlink_f1
```

```bash
./mydu tree | tail -3
```

```text
합계 36864 바이트 (36 KiB)
보통 파일 4개, 디렉터리 3개, 그 밖 1개
```

```bash
du -s --block-size=1 tree
```

```text
28672	tree
```

**8192바이트가 어긋났습니다.** `f1.txt`(5000바이트 → 블록 8192바이트)를 **두 번 세었기** 때문입니다.

| 프로그램 | 하드 링크를 |
|---|---|
| 진짜 `du` | **한 번만 센다** — 이미 본 아이노드를 기억한다 |
| 우리 `mydu` | 이름마다 센다 |

`du`는 `(st_dev, st_ino)` 쌍을 기록해 두었다가 같은 것이 다시 나오면 건너뜁니다. **실습문제 8에서 이 기능을 직접 넣습니다.**

> **이것이 "틀린 결과"를 만났을 때의 올바른 태도입니다.**
> 숫자가 다르다는 것을 발견했고, 얼마나 다른지 재었고(8192), 그 값이 무엇인지 설명할 수 있었습니다(f1.txt의 블록 크기). **차이를 설명할 수 있으면 그것은 이미 이해한 것**입니다.
{: .prompt-tip }

### 9.5 더 간단한 방법 — `nftw`

디렉터리 순회는 워낙 흔한 일이라 표준 함수가 있습니다.

```c
#include <ftw.h>

int nftw(const char *dirpath,
         int (*fn)(const char *fpath, const struct stat *sb,
                   int typeflag, struct FTW *ftwbuf),
         int nopenfd, int flags);
```

**함수를 가리키는 포인터를 넘긴다**는 점에 주목하십시오. 1부 7강에서 배우고 12강 `qsort`에서 쓴 그 방식입니다. `nftw`가 나무를 훑으며 파일마다 우리 함수를 불러 줍니다.

| 장점 | 단점 |
|---|---|
| 재귀·경로 조립을 대신해 준다 | 동작을 세밀하게 조절하기 어렵다 |
| 서술자 한계를 관리해 준다 | 전역 변수로 상태를 넘겨야 한다 |

**직접 만들어 본 뒤에 쓰는 것**이 순서입니다. 무엇을 대신해 주는지 알아야 하기 때문입니다.

---

## 제10절. 깨지지 않는 파일 갱신

### 10.1 문제

설정 파일을 고치는 흔한 코드입니다.

```c
fd = open("config.txt", O_WRONLY | O_TRUNC);    /* 내용을 비우고 */
write_full(fd, new_data, len);                  /* 새로 쓴다 */
close(fd);
```

**이 사이에 전원이 나가면 어떻게 됩니까?**

| 시점 | 파일 상태 |
|---|---|
| `O_TRUNC` 직후 | **비어 있다** |
| 쓰는 도중 | **반만 남아 있다** |
| 완료 후 | 정상 |

**설정 파일이 반쪽만 남으면 프로그램은 다음 실행에서 시작조차 못 합니다.** 실제로 서비스 장애의 흔한 원인입니다.

### 10.2 해법 — 임시 파일과 `rename`

> **[`rename(2)`](https://man7.org/linux/man-pages/man2/rename.2.html)은 나눌 수 없는 동작입니다.**
> 같은 파일 시스템 안에서라면, 어느 순간에 보아도 대상 경로에는 **옛 파일 아니면 새 파일**이 있습니다. 중간 상태가 없습니다.

절차는 다섯 단계입니다.

| 단계 | 하는 일 | 왜 |
|---|---|---|
| 1 | **같은 디렉터리**에 임시 파일 생성 | 다른 파일 시스템이면 `rename`이 실패한다 |
| 2 | 새 내용을 모두 쓴다 | |
| 3 | **`fsync`** | 디스크에 실제로 도달했음을 보장 |
| 4 | **`rename`** | 이름을 바꾼다 — 나눌 수 없다 |
| 5 | **디렉터리를 `fsync`** | 이름 변경 자체를 살아남게 한다 |

### 10.3 구현

```c
/* atomic_write: 내용을 통째로 바꾼다. 성공하면 0
   중간에 무슨 일이 생겨도 path 는 '옛 내용' 아니면 '새 내용' 둘 중 하나다 */
static int atomic_write(const char *path, const char *data, size_t len)
{
    char tmp[PATH_MAX];
    char dirbuf[PATH_MAX];
    int fd, dirfd;

    /* 1) 같은 디렉터리에 임시 파일을 만든다 */
    if (snprintf(tmp, sizeof tmp, "%s.tmp.%d", path, (int) getpid())
            >= (int) sizeof tmp) {
        fprintf(stderr, "경로가 너무 깁니다\n");
        return -1;
    }

    fd = open(tmp, O_WRONLY | O_CREAT | O_EXCL, 0644);
    if (fd == -1) {
        fprintf(stderr, "%s: %s\n", tmp, strerror(errno));
        return -1;
    }

    /* 2) 새 내용을 모두 쓴다 */
    if (write_full(fd, data, len) < 0) {
        fprintf(stderr, "write: %s\n", strerror(errno));
        close(fd);
        unlink(tmp);
        return -1;
    }

    /* 3) 디스크에 실제로 기록되었음을 보장한다 */
    if (fsync(fd) == -1) {
        fprintf(stderr, "fsync: %s\n", strerror(errno));
        close(fd);
        unlink(tmp);
        return -1;
    }
    if (close(fd) == -1) {
        fprintf(stderr, "close: %s\n", strerror(errno));
        unlink(tmp);
        return -1;
    }

    /* 4) 이름을 바꾼다 — 이 동작은 나눌 수 없다 */
    if (rename(tmp, path) == -1) {
        fprintf(stderr, "rename: %s\n", strerror(errno));
        unlink(tmp);
        return -1;
    }

    /* 5) 디렉터리 자체도 동기화해야 이름 변경이 살아남는다 */
    snprintf(dirbuf, sizeof dirbuf, "%s", path);
    dirfd = open(dirname(dirbuf), O_RDONLY | O_DIRECTORY);
    if (dirfd != -1) {
        if (fsync(dirfd) == -1)
            fprintf(stderr, "디렉터리 fsync: %s\n", strerror(errno));
        close(dirfd);
    }
    return 0;
}
```

```bash
printf '옛 내용\n' > conf.txt
```

```bash
./atomic_update conf.txt "새 내용입니다"
```

```text
바꾸기 전: 아이노드 43669, 크기 11
바꾼 뒤  : 아이노드 43853, 크기 19
아이노드가 달라졌습니다 -> 파일의 '실체'가 통째로 교체되었습니다
```

**아이노드가 바뀌었다는 것**이 이 방식의 특징입니다. 내용을 고친 것이 아니라 **파일을 통째로 갈아 끼운 것**입니다.

### 10.4 실패 경로마다 정리한다

위 코드에서 실패할 때마다 `unlink(tmp)`를 부른 것에 주목하십시오. 그러지 않으면 **`config.txt.tmp.12345` 같은 찌꺼기가 쌓입니다.** 1부 10강 6.3절에서 배운 "실패 경로의 자원 정리"가 그대로 적용된 것입니다.

### 10.5 부작용도 알아 두십시오

| 부작용 | 설명 |
|---|---|
| **하드 링크가 끊어진다** | 다른 이름은 여전히 옛 아이노드를 가리킨다 |
| 권한·소유자가 초기화된다 | 필요하면 `fstat`으로 읽어 `fchmod`·`fchown`으로 복원 |
| 아이노드가 바뀐다 | 아이노드를 감시하던 프로그램이 놓친다 |

**그래도 대부분의 경우 이 방식이 옳습니다.** 실제로 편집기·패키지 관리자·데이터베이스가 모두 이 방식을 씁니다.

### 10.6 왜 `fsync`가 두 번인가

| 대상 | 보장하는 것 |
|---|---|
| `fsync(fd)` | **파일의 내용**이 디스크에 도달했다 |
| `fsync(dirfd)` | **디렉터리 항목**(= 새 이름)이 디스크에 도달했다 |

내용만 동기화하고 디렉터리를 빠뜨리면, 전원이 나갔을 때 **"내용은 있는데 이름이 옛것"** 인 상태가 될 수 있습니다. 두 번 해야 완결됩니다.

> `fsync`는 느립니다. 실제 디스크에 도달할 때까지 기다리기 때문입니다. **정말로 잃으면 안 되는 자료에만** 쓰십시오. 자세한 보장 범위는 [`fsync(2)`](https://man7.org/linux/man-pages/man2/fsync.2.html)에 있습니다.
{: .prompt-info }

---

## 제11절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| 심벌릭 링크가 절대 안 잡힌다 | `stat`을 씀 | **`lstat`** |
| 디렉터리 순회가 멈추지 않는다 | `.`·`..`을 안 걸렀다 | 반드시 건너뛰기 |
| `du`와 값이 다르다 | 하드 링크 중복 / `st_size` 사용 | `st_blocks × 512`, 아이노드 기록 |
| 크기가 4096의 배수로만 나온다 | 블록 단위 할당 | 정상 |
| `st_blocks`에 `st_blksize`를 곱함 | 단위 혼동 | **`st_blocks`는 언제나 512** |
| `ctime`을 생성 시각으로 씀 | 이름 오해 | **change time** |
| `atime`이 안 바뀐다 | `relatime` | 정상. 6.3절 |
| 소유자인데 쓸 수 없다 | 권한 검사가 **소유자 비트에서 끝남** | 4.3절 |
| `readlink` 결과가 이상하다 | `'\0'`을 안 붙임 | 반환값 위치에 직접 |
| 파일을 지웠는데 용량이 그대로 | 아직 열려 있음 | `lsof \| grep deleted` |
| 설정 파일이 반쪽만 남는다 | 제자리 덮어쓰기 | **임시 파일 + `rename`** |
| `rename`이 `EXDEV`로 실패 | 파일 시스템이 다르다 | 같은 디렉터리에 임시 파일 |
| 권한이 0666이 안 된다 | `umask` | 4.4절 |

---

> **▶ 여기서부터 3회차 — 스스로 해 보기**
> 실습문제, 약 110분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 실습문제

> **안내**
> 1. 컴파일은 **`gcc -Wall -Wextra -std=gnu17`**, **경고 0개**여야 합니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다.
> 3. 결과를 검증할 수 있는 문제는 **`ls`·`du`·`stat`과 비교**하십시오.
{: .prompt-info }

| 문제 | 주제 | 대응 절 |
|---|---|---|
| 문제 1 | `statinfo` 완성 | 2 · 4 |
| 문제 2 | `ls -l` 만들기 | 3 · 4 |
| 문제 3 | 시각 세 가지 관찰 | 6 |
| 문제 4 | 링크 실험 | 7 |
| 문제 5 | 끊어진 심벌릭 링크 찾기 | 7 · 9 |
| 문제 6 | `unlink` 실험 | 8 |
| 문제 7 | `mydu` 완성 | 9 |
| 문제 8 | **하드 링크 중복 제거** | 9.4 |
| 문제 9 | 원자적 갱신 | 10 |
| 문제 10 | `find -name` 흉내 내기 | 9 |

---

### 문제 1. `statinfo` 완성

파일의 모든 메타데이터를 사람이 읽을 수 있게 출력하는 프로그램을 만드십시오. `stat` 명령의 출력과 비교해 검증하십시오.

**정답 및 해설**

2.3절의 코드가 답입니다. 검증은 다음과 같이 합니다.

```bash
./statinfo hello.txt | grep 아이노드
```

```bash
stat -c '아이노드 %i, 링크 %h, 권한 %a, 크기 %s, 블록 %b' hello.txt
```

- **`stat` 명령이 곧 `stat` 시스템 호출입니다.** 우리가 만든 것과 값이 같아야 합니다.
- `st_mode & 07777`로 권한만 뽑은 것에 주의하십시오. `st_mode`에는 종류 비트도 함께 들어 있어 그냥 출력하면 `0100644` 같은 값이 나옵니다.
- `getpwuid`가 `NULL`을 돌려줄 수 있다는 점(2.4절)을 처리했는지 확인하십시오.

---

### 문제 2. `ls -l` 만들기

디렉터리 하나를 받아 `ls -l`처럼 출력하는 `myls`를 만드십시오.

**정답 및 해설**

필요한 조각은 모두 본문에 있습니다.

| 열 | 얻는 곳 |
|---|---|
| 종류 문자 | `S_IS*`(3.1절) |
| 권한 9글자 | `perm_string`(4.1절) |
| 링크 수 | `st_nlink` |
| 소유자·그룹 | `getpwuid`·`getgrgid` |
| 크기 | `st_size` |
| 시각 | `st_mtim` + `strftime` |
| 이름 | `ent->d_name` |

```c
    while ((ent = readdir(dp)) != NULL) {
        if (ent->d_name[0] == '.')          /* 숨김 파일은 건너뛴다 */
            continue;
        snprintf(full, sizeof full, "%s/%s", path, ent->d_name);
        if (lstat(full, &st) == -1)
            continue;
        ...
    }
```

- **`readdir`이 주는 것은 이름뿐**입니다. 나머지는 `lstat`으로 따로 얻어야 하며, 그때 **경로를 조립**해야 합니다. `ent->d_name`만 넘기면 현재 디렉터리에서 찾으므로 다른 디렉터리를 훑을 때 실패합니다.
- 진짜 `ls`는 이름순으로 정렬합니다. `readdir`은 정렬해 주지 않으므로(9.1절) 배열에 모아 `qsort`를 써야 합니다. 1부 12강에서 만든 방식 그대로입니다.
- `ls -l` 맨 위의 `total` 값은 **`st_blocks`의 합을 1024로 나눈 것**입니다. 제5절과 이어집니다.

---

### 문제 3. 시각 세 가지 관찰

읽기·쓰기·`chmod`를 차례로 수행하며 세 시각이 어떻게 변하는지 표로 정리하십시오.

**정답 및 해설**

6.2절의 결과가 답입니다.

| 동작 | atime | mtime | ctime |
|---|---|---|---|
| 쓰기 | 그대로 | **바뀜** | **바뀜** |
| 읽기 | **바뀜** | 그대로 | 그대로 |
| `chmod` | 그대로 | 그대로 | **바뀜** |
| `rename` | 그대로 | 그대로 | **바뀜** |
| 하드 링크 추가 | 그대로 | 그대로 | **바뀜** |

- `sleep(2)`를 넣은 이유는 **초 단위로 비교하기 위해서**입니다. 넣지 않으면 모든 동작이 같은 초에 일어나 구분되지 않습니다.
- `atime`이 기대와 다르게 움직인다면 `relatime` 때문입니다(6.3절). `findmnt -no OPTIONS /`로 확인하십시오.
- **`ctime`을 바꾸는 동작을 모아 보면** 공통점이 보입니다. 모두 **아이노드의 내용**을 건드립니다.

---

### 문제 4. 링크 실험

하드 링크와 심벌릭 링크를 만들고, 원본을 지운 뒤 각각 어떻게 되는지 확인하십시오.

**정답 및 해설**

7.2절의 결과가 답입니다. 반드시 확인할 것은 다음과 같습니다.

| 확인 | 기대 |
|---|---|
| 하드 링크의 아이노드 | **원본과 같다** |
| 링크를 건 뒤 `st_nlink` | **2** |
| 심벌릭 링크의 아이노드 | 다르다 |
| 심벌릭 링크의 크기 | **경로 문자열의 길이** |
| 원본 삭제 후 하드 링크 | 정상 |
| 원본 삭제 후 심벌릭 링크 | **끊어짐** |

- **다른 파일 시스템에 하드 링크를 걸어 보십시오.** `EXDEV`(Invalid cross-device link)로 실패합니다. 아이노드 번호는 파일 시스템 안에서만 유효하기 때문입니다(1.2절).
- 디렉터리에 하드 링크를 걸려고 하면 `EPERM`입니다. 허용하면 파일 시스템에 **순환**이 생겨 순회가 끝나지 않습니다.

---

### 문제 5. 끊어진 심벌릭 링크 찾기

디렉터리를 훑어 **대상이 없는 심벌릭 링크**를 모두 찾는 프로그램을 만드십시오.

**정답 및 해설**

`lstat`과 `stat`을 **둘 다** 쓰는 것이 핵심입니다.

```c
    if (lstat(path, &ls) == -1)
        return;
    if (!S_ISLNK(ls.st_mode))
        return;                     /* 심벌릭 링크가 아니다 */

    if (stat(path, &s) == -1 && errno == ENOENT) {
        char target[PATH_MAX];
        ssize_t n = readlink(path, target, sizeof target - 1);

        if (n != -1) {
            target[n] = '\0';
            printf("끊어짐: %s -> %s\n", path, target);
        }
    }
```

| 함수 | 여기서 알려 주는 것 |
|---|---|
| `lstat` 성공 + `S_ISLNK` | **링크 자체는 존재한다** |
| `stat` 실패 + `ENOENT` | **가리키는 대상이 없다** |

```bash
ln -sf 없는파일 tree/dangling
ln -sf ../없는곳/x tree/a/dangling2
./brokenlink tree
```

```text
끊어짐: tree/dangling -> 없는파일
끊어짐: tree/a/dangling2 -> ../없는곳/x
```

`find . -xtype l` 명령이 같은 일을 합니다. **결과를 비교해 검증하십시오.**

```bash
diff <(./brokenlink tree | sed 's/^끊어짐: //; s/ -> .*//' | sort) <(find tree -xtype l | sort) && echo 일치
```

```text
일치
```

- **두 함수의 차이가 그대로 도구가 되는 좋은 예**입니다. 하나만으로는 판별할 수 없습니다.
- `errno`를 확인한 이유는, `stat` 실패가 `EACCES`(권한 없음)일 수도 있기 때문입니다. 그때는 끊어진 것이 아닙니다. 1강 10.2절에서 배운 대로 **실패의 이유를 구분**해야 합니다.

---

### 문제 6. `unlink` 실험

파일을 연 채로 지우고, 서술자로 계속 읽고 쓸 수 있는지 확인하십시오.

**정답 및 해설**

8.2절의 결과가 답입니다.

```text
지운 뒤   : 링크 수 0, 크기 41
서술자로 읽기: 이 자료는 아직 살아 있습니다
```

- **링크 수가 0인데 자료가 읽힙니다.** 이름과 실체가 분리되어 있다는 제1절의 주장이 여기서 증명됩니다.
- 프로그램이 도는 동안 다른 터미널에서 확인해 보십시오.

```bash
ls -l /proc/$(pgrep unlink_open)/fd
```

- `-> /home/student/cmid/lab03/temp.dat (deleted)`처럼 **`(deleted)` 표시**가 붙습니다. 2강 9.2절에서 본 `/proc/PID/fd`가 여기서도 쓰입니다.
- 실무에서 디스크가 가득 찼는데 `du`로는 이유를 못 찾을 때, 바로 이 상태를 의심합니다(8.4절).

---

### 문제 7. `mydu` 완성

디렉터리 사용량을 재는 `mydu`를 만들고 **`du`와 값이 일치하는지** 확인하십시오.

**정답 및 해설**

9.2절의 코드가 답이며, 검증은 9.3절과 같이 합니다.

```bash
diff <(./mydu tree | head -2 | awk '{print $1, $2}') <(du -d 1 tree | awk '{print $1, $2}')
```

- **`st_size`가 아니라 `st_blocks × 512`** 를 써야 `du`와 맞습니다. `st_size`를 쓰면 `du --apparent-size`와 같아집니다.
- 디렉터리 자신의 블록도 더해야 합니다. 빈 디렉터리도 보통 4096바이트를 차지합니다.
- **일치하지 않는다면 어디서 갈리는지 좁혀 보십시오.** 하위 디렉터리별로 비교하면 금방 찾을 수 있습니다.

---

### 문제 8. 하드 링크 중복 제거

같은 아이노드를 두 번 세지 않도록 `mydu`를 고치십시오.

**정답 및 해설**

**본 아이노드를 기억해 두었다가 건너뜁니다.**

```c
struct seen {
    dev_t dev;
    ino_t ino;
};

static struct seen *seen_list = NULL;
static size_t seen_n = 0, seen_cap = 0;

/* already_seen: 이미 센 파일이면 1, 처음이면 기록하고 0 */
static int already_seen(const struct stat *st)
{
    size_t i;

    if (st->st_nlink <= 1)          /* 링크가 하나뿐이면 중복될 수 없다 */
        return 0;

    for (i = 0; i < seen_n; i++)
        if (seen_list[i].dev == st->st_dev && seen_list[i].ino == st->st_ino)
            return 1;

    if (seen_n == seen_cap) {       /* 1부 12강의 늘어나는 배열 */
        size_t newcap = (seen_cap == 0) ? 64 : seen_cap * 2;
        struct seen *p = realloc(seen_list, newcap * sizeof *p);

        if (p == NULL)
            return 0;               /* 기억하지 못할 뿐, 세기는 계속한다 */
        seen_list = p;
        seen_cap = newcap;
    }
    seen_list[seen_n].dev = st->st_dev;
    seen_list[seen_n].ino = st->st_ino;
    seen_n++;
    return 0;
}
```

`walk` 안에서 더하기 전에 검사합니다.

```c
    if (!already_seen(&st))
        sum += (long long) st.st_blocks * 512;
```

고친 뒤 다시 재면 `du`와 일치합니다.

```bash
./mydu tree3 | tail -2      # 중복 제거 전
```

```text
합계 36864 바이트 (36 KiB)
보통 파일 4개, 디렉터리 3개, 그 밖 1개
```

```bash
./mydu2 tree3 | tail -2     # 중복 제거 후
```

```text
합계 28672 바이트 (28 KiB)
보통 파일 4개, 디렉터리 3개, 그 밖 1개
```

```bash
du -s --block-size=1 tree3
```

```text
28672	tree3
```

디렉터리별 출력도 맞아떨어집니다.

```bash
./mydu2 tree3 | head -2 ; du -d 1 tree3
```

```text
        24	tree3/a
        28	tree3
24	tree3/a
28	tree3
```

- **`st_nlink <= 1`이면 검사를 건너뛴 것**이 중요합니다. 링크가 하나뿐인 파일은 중복될 수 없으므로, 대부분의 파일에서 목록 탐색 비용을 아낍니다.
- **장치 번호까지 비교해야 합니다.** 아이노드 번호는 파일 시스템 안에서만 유일하기 때문입니다(1.2절).
- 지금은 목록을 처음부터 훑으므로 파일이 많으면 느립니다. 1부 8강에서 배운 **해시 표**를 쓰면 훨씬 빨라집니다. 자료 구조 선택이 성능을 결정하는 실제 사례입니다.
- `realloc` 결과를 **임시 포인터로 받은 것**도 1부 12강 5.2절 그대로입니다.

---

### 문제 9. 원자적 갱신

전원이 끊겨도 파일이 깨지지 않도록 갱신하는 함수를 만드십시오.

**정답 및 해설**

10.3절의 코드가 답입니다. 확인할 점은 다음과 같습니다.

| 확인 | 방법 |
|---|---|
| 아이노드가 바뀌는가 | `stat -c %i` 전후 비교 |
| 임시 파일이 남지 않는가 | `ls *.tmp.*` |
| 실패해도 원본이 온전한가 | 일부러 `write`를 실패시켜 확인 |

- **가장 흔한 실수는 임시 파일을 `/tmp`에 만드는 것**입니다. `/tmp`가 별도의 파일 시스템이면 `rename`이 `EXDEV`로 실패합니다. **반드시 대상과 같은 디렉터리**에 만들어야 합니다.
- `O_EXCL`을 쓴 이유는 2강 8.1절과 같습니다. 같은 프로그램이 두 개 돌 때 임시 파일 이름이 겹치지 않게 PID를 붙였지만, 그래도 확인이 필요합니다.
- 권한을 보존하려면 원본을 `stat`으로 읽어 `fchmod`로 옮겨야 합니다(10.5절).

---

### 문제 10. `find -name` 흉내 내기

디렉터리를 재귀로 훑어 **이름이 주어진 무늬와 맞는** 파일을 모두 출력하는 프로그램을 만드십시오.

**정답 및 해설**

무늬 맞추기는 표준 함수 `fnmatch`가 해 줍니다.

```c
#include <fnmatch.h>

    if (fnmatch(pattern, ent->d_name, 0) == 0)
        printf("%s\n", full);
```

```bash
./myfind tree '*.txt'
```

```bash
find tree -name '*.txt'
```

- **두 출력을 정렬해 비교하면 검증이 끝납니다.**

```bash
diff <(./myfind tree '*.txt' | sort) <(find tree -name '*.txt' | sort) && echo 일치
```

```text
일치
```

**디렉터리 이름으로도 시험해 보십시오.** `find`는 파일뿐 아니라 디렉터리도 찾습니다.

```bash
diff <(./myfind tree 'a' | sort) <(find tree -name 'a' | sort) && echo 일치
```

```text
일치
```

- 순회 뼈대는 `mydu`와 **똑같습니다.** 달라지는 것은 "각 파일에서 무엇을 하는가"뿐입니다. 이 구조가 보이면 9.5절의 `nftw`가 왜 그런 모양인지 이해됩니다 — **순회는 공통, 하는 일만 함수로 받는 것**입니다.
- 심벌릭 링크를 따라갈지 말지 정해야 합니다. `find`는 기본적으로 따라가지 않습니다(`-P`). 우리도 `lstat`을 쓰면 같은 동작이 됩니다.
- 읽을 수 없는 디렉터리를 만나면 `find`처럼 **오류를 알리고 계속 진행**해야 합니다.

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 소스 — `statinfo.c`, `myls.c`, `timestamps.c`, `links_demo.c`, `brokenlink.c`, `unlink_open.c`, `mydu.c`, `atomic_update.c`, `myfind.c` |
| 2 | `statinfo`와 `stat` 명령의 출력 비교(문제 1) |
| 3 | **세 가지 시각 변화표**(문제 3) |
| 4 | 링크 실험 결과 — 아이노드·링크 수·삭제 후 상태(문제 4) |
| 5 | `unlink` 실험과 `/proc/PID/fd`의 `(deleted)` 화면(문제 6) |
| 6 | **`mydu`와 `du`의 비교** — 중복 제거 전후 모두(문제 7·8) |
| 7 | 원자적 갱신 전후의 아이노드(문제 9) |
| 8 | `myfind`와 `find`의 `diff` 결과(문제 10) |
| 9 | 짧은 서술 ① 하드 링크와 심벌릭 링크의 차이를 아이노드 관점에서 |
| 10 | 짧은 서술 ② 왜 제자리 덮어쓰기 대신 임시 파일과 `rename`을 쓰는가 |

---

## 정리

| 구분 | 핵심 |
|---|---|
| 아이노드 | **이름과 실체는 따로다.** 디렉터리는 이름 → 아이노드 표 |
| 세 함수 | `stat` 따라감 / `lstat` 안 따라감 / `fstat` 열린 서술자 |
| 종류 | `S_ISREG`·`S_ISDIR`·`S_ISLNK`… **FIFO와 소켓도 파일** |
| 권한 | 소유자→그룹→기타 중 **처음 맞는 것에서 끝난다** |
| 특수 비트 | setuid(4000)·setgid(2000)·sticky(1000) |
| 크기 | `st_size`는 논리, **`st_blocks × 512`는 실제**(단위는 언제나 512) |
| 시각 | `ctime`은 **change**, 생성 시각이 아니다. `atime`은 `relatime` 탓에 믿기 어렵다 |
| 하드 링크 | 같은 아이노드의 다른 이름. 원본을 지워도 살아 있다 |
| 심벌릭 링크 | 경로 문자열을 담은 별도 파일. **크기 = 경로 길이** |
| `unlink` | **이름만 지운다.** 링크 수 0 + 연 사람 없음 → 그때 회수 |
| 순회 | `opendir`/`readdir`, **`.`과 `..`을 반드시 건너뛰기**, 순서 보장 없음 |
| 원자적 갱신 | 임시 파일 → `fsync` → **`rename`** → 디렉터리 `fsync` |

---

## 다음 강의 예고

**4강 「프로세스」**(APUE 8·9장)에서는 프로그램이 **다른 프로그램을 만들어 냅니다.**

- `fork`로 프로세스를 **복제**한다 — 한 번 부르고 두 번 돌아오는 함수
- `exec`로 **다른 프로그램이 된다**
- `wait`로 자식이 끝나기를 기다리고 **종료 상태**를 받는다
- **좀비와 고아** 프로세스가 무엇인지, 왜 생기는지
- 부모가 물려주는 것 — **열린 파일 서술자**(2강 9.3절이 여기서 쓰입니다)

1강 8.6절에서 본 PPID의 정체가 밝혀지고, 셸이 명령을 실행하는 원리를 알게 됩니다.

---

## 부록 A. 이번 강의 명령·함수 요약

| 하려는 일 | 함수 / 명령 |
|---|---|
| 파일 정보 | `stat` · `fstat` · `lstat` |
| 종류 판정 | `S_ISREG` · `S_ISDIR` · `S_ISLNK` … |
| 권한 바꾸기 | `chmod` · `fchmod` |
| 소유자 바꾸기 | `chown` · `fchown` · `lchown` |
| 하드 링크 | `link` |
| 심벌릭 링크 | `symlink` · `readlink` |
| 이름 지우기 | `unlink` |
| 이름 바꾸기 | `rename` |
| 디렉터리 순회 | `opendir` · `readdir` · `closedir` |
| 나무 순회(표준) | `nftw` |
| 무늬 맞추기 | `fnmatch` |
| 디스크 동기화 | `fsync` |
| UID → 이름 | `getpwuid` · `getgrgid` |
| 정보 확인(명령) | `stat -c '%i %h %a %s %b %n'` |
| 사용량 확인(명령) | `du -d 1` · `du -s --block-size=1` |
| 마운트 옵션 | `findmnt -no OPTIONS /` |
| 지워졌지만 열린 파일 | `lsof \| grep deleted` |

## 부록 B. 표준 문서와 출처

**이번 강의에서 다룬 시스템 호출**

| 함수 | 리눅스 man | POSIX 표준 |
|---|---|---|
| `stat` | [`stat(2)`](https://man7.org/linux/man-pages/man2/stat.2.html) | [`stat()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/stat.html) |
| `lstat` | [`lstat(2)`](https://man7.org/linux/man-pages/man2/lstat.2.html) | [`lstat()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/lstat.html) |
| `chmod` | [`chmod(2)`](https://man7.org/linux/man-pages/man2/chmod.2.html) | [`chmod()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/chmod.html) |
| `link` | [`link(2)`](https://man7.org/linux/man-pages/man2/link.2.html) | [`link()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/link.html) |
| `symlink` | [`symlink(2)`](https://man7.org/linux/man-pages/man2/symlink.2.html) | [`symlink()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/symlink.html) |
| `readlink` | [`readlink(2)`](https://man7.org/linux/man-pages/man2/readlink.2.html) | [`readlink()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/readlink.html) |
| `unlink` | [`unlink(2)`](https://man7.org/linux/man-pages/man2/unlink.2.html) | [`unlink()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/unlink.html) |
| `rename` | [`rename(2)`](https://man7.org/linux/man-pages/man2/rename.2.html) | [`rename()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/rename.html) |
| `fsync` | [`fsync(2)`](https://man7.org/linux/man-pages/man2/fsync.2.html) | [`fsync()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/fsync.html) |
| `opendir` | [`opendir(3)`](https://man7.org/linux/man-pages/man3/opendir.3.html) | [`opendir()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/opendir.html) |
| `readdir` | [`readdir(3)`](https://man7.org/linux/man-pages/man3/readdir.3.html) | [`readdir()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/readdir.html) |
| `nftw` | [`nftw(3)`](https://man7.org/linux/man-pages/man3/nftw.3.html) | [`nftw()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/nftw.html) |

**개념 문서**

| 내용 | 문서 |
|---|---|
| 아이노드의 구조와 `st_mode` 비트 | [`inode(7)`](https://man7.org/linux/man-pages/man7/inode.7.html) |
| 경로 해석과 권한 검사 순서(4.3절) | [`path_resolution(7)`](https://man7.org/linux/man-pages/man7/path_resolution.7.html) |
| 심벌릭 링크의 동작 | [`symlink(7)`](https://man7.org/linux/man-pages/man7/symlink.7.html) |

**본문의 주장과 근거**

| 주장 | 근거 |
|---|---|
| `st_blocks`의 단위는 언제나 512(5.2절) | `stat(2)` 명세 + 실측(8 × 512 = 4096) |
| 하드 링크는 아이노드를 공유한다(7.2절) | `links_demo.c` 실행 — 양쪽 모두 아이노드 35378 |
| 심벌릭 링크 크기 = 경로 길이(7.3절) | `ls -l`이 보여 준 6 = `"f1.txt"`의 6글자 |
| 디렉터리 링크 수 = 2 + 하위 개수(7.4절) | 하위 2개인 디렉터리에서 `stat -c '%h'` → 4 |
| 링크 수 0이어도 서술자로 읽힌다(8.2절) | `unlink_open.c` 실행 결과 |
| 쓰기는 mtime+ctime, 읽기는 atime, chmod는 ctime(6.2절) | `timestamps.c` 실행 결과 |
| **`mydu`가 `du`와 정확히 일치**(9.3절) | `du -d 1` → 16·28, `du -s --block-size=1` → 28672 모두 일치 |
| 하드 링크가 있으면 8192바이트 어긋난다(9.4절) | 36864 − 28672 = 8192 = `f1.txt`의 블록 크기 |
| 중복 제거 후 `du`와 다시 일치(문제 8) | `mydu2` 28672 = `du -s --block-size=1`, 디렉터리별 24·28도 `du -d 1`과 일치 |
| `myfind`가 `find -name`과 일치(문제 10) | `diff`로 확인 — 파일·디렉터리 모두 |
| `brokenlink`가 `find -xtype l`과 일치(문제 5) | `diff`로 확인 |
| `/usr/bin/passwd` 4755, `/tmp` 1777(4.2절) | `stat -c '%A %a %n'` |
| `umask` 0022에서 `touch`는 0644(4.4절) | `stat -c '%a'` |

> 표에 실린 값은 모두 **Ubuntu 24.04에서 실제로 실행한 결과**입니다. 아이노드 번호처럼 환경마다 달라지는 값은 **"같다/다르다"의 관계**만 보시면 됩니다.
{: .prompt-tip }

## 부록 C. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `statinfo.c` | 메타데이터 전부 출력 | 2 · 3 · 4 |
| `timestamps.c` | 세 가지 시각의 변화 | 6 |
| `links_demo.c` | 하드 링크와 심벌릭 링크 비교 | 7 |
| `unlink_open.c` | 열린 파일을 지우면 | 8 |
| `mydu.c` | 디렉터리 사용량 — `du` 만들기 | 9 |
| `atomic_update.c` | 임시 파일 + `fsync` + `rename` | 10 |
| `myls.c` | `ls -l` 만들기 | 문제 2 |
| `brokenlink.c` | 끊어진 심벌릭 링크 찾기 | 문제 5 |
| `myfind.c` | `find -name` 만들기 | 문제 10 |
