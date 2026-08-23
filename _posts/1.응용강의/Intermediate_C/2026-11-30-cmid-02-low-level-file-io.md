---
title: 중급 C 프로그래밍 2강 - 저수준 파일 입출력과 파일 서술자
date: 2026-11-30 09:00:00 +0900
categories:
  - 1.응용강의
  - 중급 C 프로그래밍
tags:
  - C언어
  - 시스템프로그래밍
  - 리눅스
  - 파일서술자
  - open
  - read
  - write
  - lseek
  - 버퍼링
pin:
mermaid: false
---

> **학습 목표**
> 1. `FILE *`와 파일 서술자의 관계를 설명할 수 있다.
> 2. `open`의 플래그와 권한 인자를 상황에 맞게 쓸 수 있다.
> 3. **`read`·`write`가 요청보다 적게 처리할 수 있음**을 알고 대응 함수를 작성할 수 있다.
> 4. `close`를 빠뜨렸을 때 무슨 일이 생기는지 설명할 수 있다.
> 5. `lseek`로 파일 안을 이동하고 크기를 구할 수 있다.
> 6. **버퍼 크기가 성능에 미치는 영향**을 측정할 수 있다.
> 7. `O_EXCL`·`O_APPEND`처럼 `fopen`으로는 할 수 없는 일을 할 수 있다.
> 8. 파일 서술자 표와 `/proc/PID/fd`를 이해할 수 있다.
> 9. 서술자 번호 배정 규칙을 설명할 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

1강에서 **`write`를 직접 불러** 보았습니다. 이번 강의에서는 파일 입출력 전체를 그 층위에서 다시 배웁니다.

1부 9강에서 배운 것과 대응시켜 보십시오.

| 1부 9강 (표준 라이브러리) | 2부 2강 (시스템 호출) |
|---|---|
| `fopen` | `open` |
| `fread`·`fgets`·`fgetc` | `read` |
| `fwrite`·`fprintf`·`fputc` | `write` |
| `fclose` | `close` |
| `fseek`·`ftell` | `lseek` |
| `FILE *` | **`int`(파일 서술자)** |

**아래층이 위층보다 반드시 좋은 것은 아닙니다.** 각각 알맞은 자리가 있으며, 이번 강의의 목표 중 하나는 **언제 어느 쪽을 쓸지 판단하는 것**입니다.

이 강의는 **3회차 분량**(모두 합쳐 약 470분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제5절 | 파일 서술자와 `read`·`write` | 185분 |
| **2회차** | 제6절 ~ 제11절 | `lseek`·성능·`open`만의 기능 | 175분 |
| **3회차** | 실습문제 | 스스로 해 보기 | 110분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | `FILE *`의 가면을 벗긴다 | 30분 |
| 제2절 | `open` — 파일을 여는 법 | 45분 |
| 제3절 | `read`와 `write` | 45분 |
| 제4절 | 부분 처리에 대응하기 | 40분 |
| 제5절 | `close`와 서술자 누수 | 25분 |
| 제6절 | `lseek`와 파일 안의 자리 | 35분 |
| 제7절 | 버퍼 크기와 성능 | 40분 |
| 제8절 | `fopen`으로는 할 수 없는 일 | 35분 |
| 제9절 | 파일 서술자 표 | 30분 |
| 제10절 | 어느 쪽을 쓸 것인가 | 20분 |
| 제11절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 디렉터리**는 `~/cmid/lab02`를 사용합니다. 이번 강의는 **VM 한 대**(`c-srv`)에서 진행합니다.

```bash
mkdir -p ~/cmid/lab02 && cd ~/cmid/lab02
```

---

## 제1절. `FILE *`의 가면을 벗긴다

### 1.1 `FILE`은 무엇인가

1부에서 `FILE *fp`를 수없이 썼지만, **그 안에 무엇이 들어 있는지는 보지 않았습니다.**

`FILE`은 구조체입니다. 그리고 그 안에는 **파일 서술자 하나와 버퍼**가 들어 있습니다.

| `FILE`이 가진 것 | 하는 일 |
|---|---|
| **파일 서술자(int)** | 커널에게 "어느 파일"인지 알려 주는 번호 |
| 버퍼와 그 크기 | 시스템 호출 횟수를 줄인다 |
| 현재 위치 | 버퍼 안 어디까지 읽었나 |
| 오류·파일 끝 표시 | `ferror`·`feof`가 보는 값 |
| 버퍼링 방식 | 줄 단위인가 블록 단위인가 |

**즉 `FILE`은 파일 서술자를 감싼 껍데기입니다.**

### 1.2 껍데기 속을 들여다보기

`fileno` 함수가 그 안의 서술자를 꺼내 줍니다.

```c
/* fileno_demo.c — FILE * 의 속에는 파일 서술자가 들어 있다 */
#include <stdio.h>
#include <unistd.h>
#include <string.h>

int main(void)
{
    const char *msg = "  [2] write 로 직접\n";

    printf("stdin  의 서술자 = %d\n", fileno(stdin));
    printf("stdout 의 서술자 = %d\n", fileno(stdout));
    printf("stderr 의 서술자 = %d\n", fileno(stderr));

    printf("  [1] printf 로\n");
    write(STDOUT_FILENO, msg, strlen(msg));
    printf("  [3] printf 로\n");

    return 0;
}
```

```bash
gcc -Wall -Wextra -std=gnu17 fileno_demo.c -o fileno_demo
```

```bash
./fileno_demo
```

```text
stdin  의 서술자 = 0
stdout 의 서술자 = 1
stderr 의 서술자 = 2
  [1] printf 로
  [2] write 로 직접
  [3] printf 로
```

**순서대로 나왔습니다.** 화면은 줄 단위 버퍼링이라 각 `printf`가 곧바로 나가기 때문입니다.

### 1.3 섞어 쓰면 안 되는 이유

같은 프로그램을 **파일로 재지정**해 보십시오.

```bash
./fileno_demo > o.txt
```

```bash
cat o.txt
```

```text
  [2] write 로 직접
stdin  의 서술자 = 0
stdout 의 서술자 = 1
stderr 의 서술자 = 2
  [1] printf 로
  [3] printf 로
```

**순서가 무너졌습니다.** `[2]`가 맨 앞으로 나왔습니다.

| 무슨 일이 일어났나 |
|---|
| 파일로 재지정되자 `stdout`이 **블록 단위 버퍼링**으로 바뀌었다 |
| `printf`의 출력은 전부 버퍼에 쌓였다 |
| `write`는 버퍼를 거치지 않고 **곧바로 커널에** 갔다 |
| 프로그램이 끝나며 버퍼가 비워져, 쌓여 있던 것이 **뒤늦게** 나왔다 |

> **같은 대상에 `FILE *`과 파일 서술자를 섞어 쓰지 마십시오.**
> 하나를 골라 쓰거나, 반드시 섞어야 한다면 전환하기 전에 `fflush`로 버퍼를 비우십시오. 이 문제는 화면에서 시험할 때는 드러나지 않다가 **파일로 재지정하는 순간** 나타나기 때문에 특히 고약합니다.
{: .prompt-danger }

### 1.4 두 층의 관계

```text
   [ 우리 프로그램 ]
        │  printf / fprintf / fgets ...
   ┌────▼────────────────┐
   │  FILE (버퍼 + fd)    │   ← 사용자 공간, 표준 라이브러리
   └────┬────────────────┘
        │  write / read  (버퍼가 차거나 비었을 때만)
   ═════▼═════════════════   ← 사용자·커널 경계
   [        커널          ]
        │
   [ 디스크 · 터미널 · 소켓 ]
```

1강 7.3절에서 본 층 그림이 여기서 완성됩니다.

---

## 제2절. `open` — 파일을 여는 법

### 2.1 원형

```c
#include <fcntl.h>

int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

| 반환 | 뜻 |
|---|---|
| 0 이상 | **새 파일 서술자** |
| `-1` | 실패. `errno` 확인 |

**인자가 두 가지 형태**인 것에 주의하십시오. 파일을 **새로 만들 때만**(`O_CREAT`) 세 번째 인자가 필요합니다.

### 2.2 접근 방식 — 반드시 하나

| 플래그 | 뜻 |
|---|---|
| `O_RDONLY` | 읽기만 |
| `O_WRONLY` | 쓰기만 |
| `O_RDWR` | 읽고 쓰기 |

셋 중 **정확히 하나**를 골라야 합니다.

### 2.3 함께 쓰는 플래그

| 플래그 | 하는 일 | `fopen`에 대응 |
|---|---|---|
| `O_CREAT` | 없으면 만든다 | `"w"`, `"a"` |
| `O_TRUNC` | 있으면 **내용을 지운다** | `"w"` |
| `O_APPEND` | 언제나 **끝에** 쓴다 | `"a"` |
| **`O_EXCL`** | `O_CREAT`와 함께 — **이미 있으면 실패** | **없음** |
| `O_NONBLOCK` | 막히지 않게 한다 | **없음** |
| `O_CLOEXEC` | `exec` 할 때 자동으로 닫는다 | **없음** |
| `O_SYNC` | 디스크에 실제로 쓸 때까지 기다린다 | **없음** |

**아래 네 가지는 `fopen`으로 할 수 없습니다.** 제8절에서 왜 중요한지 보게 됩니다.

### 2.4 `fopen` 모드와의 대응

| `fopen` 모드 | 대응하는 `open` |
|---|---|
| `"r"` | `O_RDONLY` |
| `"w"` | `O_WRONLY \| O_CREAT \| O_TRUNC` |
| `"a"` | `O_WRONLY \| O_CREAT \| O_APPEND` |
| `"r+"` | `O_RDWR` |
| `"w+"` | `O_RDWR \| O_CREAT \| O_TRUNC` |

**`"w"`가 기존 내용을 지운다**는 사실이 `O_TRUNC`로 명확히 드러납니다. 1부에서 "왜 파일이 비었지?" 하고 당황했다면 이것이 이유였습니다.

### 2.5 권한 인자

```c
fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
```

`0644`는 **8진수**입니다.

| 자리 | 대상 | 값 | 뜻 |
|---|---|---|---|
| 6 | 소유자 | 4+2 | 읽기+쓰기 |
| 4 | 그룹 | 4 | 읽기 |
| 4 | 그 밖 | 4 | 읽기 |

**실제로 만들어지는 권한은 `mode & ~umask`** 입니다. `umask`가 `022`라면 `0644`를 요청해도 그대로 `0644`가 되지만, `0666`을 요청하면 `0644`가 됩니다.

```bash
umask
```

```text
0022
```

> **`0644`의 앞 `0`을 빠뜨리지 마십시오.** `644`라고 쓰면 8진수가 아니라 10진수 644가 되어 엉뚱한 권한이 됩니다. 실행 파일 권한이 이상하게 붙는 사고의 대표적 원인입니다.
{: .prompt-warning }

---

## 제3절. `read`와 `write`

### 3.1 원형

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

| 반환값 | `read`의 뜻 | `write`의 뜻 |
|---|---|---|
| 양수 | **실제로 읽은** 바이트 수 | **실제로 쓴** 바이트 수 |
| 0 | **파일 끝(EOF)** | 0바이트를 요청했다 |
| `-1` | 실패 | 실패 |

### 3.2 `cat`을 직접 만들어 본다

```c
static int cat_fd(int fd, const char *name)
{
    char buf[BUFSIZE];
    ssize_t n;

    while ((n = read(fd, buf, sizeof buf)) > 0) {
        if (write_full(STDOUT_FILENO, buf, (size_t) n) < 0) {
            fprintf(stderr, "write: %s\n", strerror(errno));
            return -1;
        }
    }
    if (n < 0) {
        fprintf(stderr, "%s: %s\n", name, strerror(errno));
        return -1;
    }
    return 0;                          /* n == 0 이면 파일 끝 */
}
```

```c
    for (i = 1; i < argc; i++) {
        fd = open(argv[i], O_RDONLY);
        if (fd == -1) {
            fprintf(stderr, "%s: %s\n", argv[i], strerror(errno));
            status = 1;
            continue;                  /* 한 파일이 실패해도 나머지는 계속 */
        }
        if (cat_fd(fd, argv[i]) < 0)
            status = 1;
        close(fd);
    }
    return status;
```

```bash
gcc -Wall -Wextra -std=gnu17 mycat.c -o mycat
```

```bash
printf '첫째 줄\n둘째 줄\n' > sample.txt
```

```bash
./mycat sample.txt
```

```text
첫째 줄
둘째 줄
```

```bash
./mycat 없는파일
```

```text
없는파일: No such file or directory
```

```bash
echo $?
```

```text
1
```

### 3.3 눈여겨볼 설계

| 판단 | 이유 |
|---|---|
| 인자가 없으면 **표준 입력**을 읽는다 | `cat` 본래의 동작. 파이프에 쓸 수 있다 |
| 한 파일이 실패해도 **계속 진행** | 진짜 `cat`도 그렇게 동작한다 |
| **종료 상태**를 1로 | 셸이 성공·실패를 알 수 있다 |
| 오류는 `stderr`로 | 출력을 파일로 재지정해도 오류는 화면에 |
| `n == 0`과 `n < 0`을 **구분** | 파일 끝과 오류는 전혀 다른 일이다 |

1부에서 배운 원칙들이 그대로 이어집니다. 달라진 것은 **도구가 한 층 아래로 내려온 것뿐**입니다.

```bash
./mycat sample.txt | ./mycat
```

```text
첫째 줄
둘째 줄
```

인자 없이 부르면 표준 입력을 읽으므로 파이프로 이을 수 있습니다.

---

## 제4절. 부분 처리에 대응하기

### 4.1 이것이 이번 강의에서 가장 중요합니다

> **`read`와 `write`는 요청한 만큼 처리하지 않을 수 있습니다.**

이것은 오류가 아니라 **정상 동작**입니다. 추측이 아니라 **표준 문서에 명시되어 있습니다.**

[`read(2)`](https://man7.org/linux/man-pages/man2/read.2.html)의 RETURN VALUE입니다.

```text
RETURN VALUE
       On success, the number of bytes read is returned (zero
       indicates end of file), and the file position is advanced
       by this number.  It is not an error if this number is
       smaller than the number of bytes requested; this may
       happen for example because fewer bytes are actually
       available right now (maybe because we were close to end-
       of-file, or because we are reading from a pipe, or from a
       terminal), or because read() was interrupted by a signal.
```

[`write(2)`](https://man7.org/linux/man-pages/man2/write.2.html)에도 같은 경고가 있습니다.

```text
       Note that a successful write() may transfer fewer than
       count bytes.
```

**"It is not an error"** 와 **"may transfer fewer than count bytes"** 라는 문장을 기억하십시오. 이 한 줄을 놓쳐서 생기는 사고가 2부 내내 반복됩니다.

| `read`가 적게 읽는 경우 | `write`가 적게 쓰는 경우 |
|---|---|
| 파일 끝에 가까워졌다 | 디스크가 가득 찼다 |
| **터미널에서 한 줄만 들어왔다** | 파이프 버퍼가 찼다 |
| **네트워크로 일부만 도착했다** | **소켓 송신 버퍼가 찼다** |
| 시그널에 중단되었다 | 시그널에 중단되었다 |

**네트워크에서는 부분 처리가 예외가 아니라 일상입니다.** 9강에서 이 사실을 잊으면 곧바로 자료가 깨집니다. 지금 대응 함수를 만들어 두고 2부 내내 씁니다.

### 4.2 `write_full`

```c
/* write_full: n 바이트를 모두 쓸 때까지 되풀이한다. 성공하면 0, 실패하면 -1 */
static int write_full(int fd, const char *buf, size_t n)
{
    size_t done = 0;

    while (done < n) {
        ssize_t w = write(fd, buf + done, n - done);
        if (w < 0) {
            if (errno == EINTR)
                continue;              /* 시그널에 끊겼을 뿐이다 (5강) */
            return -1;
        }
        done += (size_t) w;
    }
    return 0;
}
```

| 요소 | 이유 |
|---|---|
| `done`을 누적 | 쓴 만큼 **앞으로 밀며** 남은 것만 다시 |
| `buf + done` | 1부 6강의 포인터 연산 |
| `EINTR`이면 **다시 시도** | 실패가 아니다. 5강에서 자세히 |
| 그 밖의 오류는 `-1` | 호출한 쪽이 판단한다 |

### 4.3 `read_full`

읽기 쪽은 **파일 끝을 구분**해야 하므로 조금 다릅니다.

```c
/* read_full: n 바이트를 채울 때까지 읽는다.
   돌려주는 값은 실제로 읽은 바이트 수(n 보다 작으면 파일 끝), 실패하면 -1 */
static ssize_t read_full(int fd, char *buf, size_t n)
{
    size_t done = 0;

    while (done < n) {
        ssize_t r = read(fd, buf + done, n - done);
        if (r < 0) {
            if (errno == EINTR)
                continue;
            return -1;
        }
        if (r == 0)
            break;                     /* 파일 끝 — 더 읽을 것이 없다 */
        done += (size_t) r;
    }
    return (ssize_t) done;
}
```

**`r == 0`에서 멈추는 것**이 핵심입니다. 그러지 않으면 파일 끝에서 무한히 돌게 됩니다.

> **`write`에서는 0이 거의 나오지 않지만, `read`에서 0은 반드시 처리해야 합니다.** 소켓에서 0은 **상대가 연결을 닫았다**는 뜻이 됩니다(9강).
{: .prompt-tip }

### 4.4 왜 라이브러리로 만들어 두는가

이 두 함수는 **2부의 거의 모든 강의에서 쓰입니다.** 1강 문제 8에서 만든 `util.c`에 함께 넣어 두십시오.

```c
/* io_util.h */
#ifndef IO_UTIL_H
#define IO_UTIL_H

#include <stddef.h>
#include <sys/types.h>

int     write_full(int fd, const void *buf, size_t n);
ssize_t read_full(int fd, void *buf, size_t n);

#endif   /* IO_UTIL_H */
```

1부 11강에서 만든 라이브러리 방식 그대로입니다.

---

## 제5절. `close`와 서술자 누수

### 5.1 닫아야 하는 이유

```c
int close(int fd);
```

| 닫지 않으면 | 결과 |
|---|---|
| 서술자가 쌓인다 | **한계에 도달하면 `open`이 실패**한다 |
| 커널 자원이 남는다 | 파일이 삭제되어도 공간이 반환되지 않는다 |
| 잠금이 유지된다 | 다른 프로세스가 접근하지 못한다 |

### 5.2 한계 확인

```bash
ulimit -n
```

```text
1024
```

프로세스 하나가 열 수 있는 서술자의 **soft 한계**입니다. 서버 프로그램에서 접속마다 서술자를 하나씩 쓰면서 닫지 않으면 **이 숫자에 도달하는 순간 무너집니다.** 10강에서 다중 접속 서버를 만들 때 반드시 기억해야 할 값입니다.

> 값은 배포판과 설정에 따라 다릅니다. 1024인 곳도, 수만인 곳도 있습니다. **`ulimit -n`으로 자기 환경의 값을 직접 확인**하십시오. `ulimit -Hn`은 관리자가 정한 hard 한계이며, soft 한계는 그 아래에서 스스로 올릴 수 있습니다.
{: .prompt-info }

### 5.3 `close`도 실패한다

```c
if (close(fd) == -1)
    fprintf(stderr, "close: %s\n", strerror(errno));
```

**1부 12강에서 `fclose`의 반환값을 검사한 것과 같은 이유**입니다. 버퍼에 남아 있던 것을 마지막으로 밀어 내는 과정에서 디스크가 가득 찼다는 사실이 드러날 수 있습니다.

> **다만 `close`가 실패해도 서술자는 이미 닫혔습니다.** 다시 `close` 하면 안 됩니다. 실패하면 **보고만 하고 넘어가는 것**이 옳습니다.
{: .prompt-warning }

---

> **▶ 여기서부터 2회차 — `lseek`·성능·`open`만의 기능**
> 제6절 ~ 제11절, 약 175분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제6절. `lseek`와 파일 안의 자리

### 6.1 파일 위치

열린 파일마다 **"다음에 읽고 쓸 자리"** 가 있습니다. `read`·`write`는 이 자리를 자동으로 옮깁니다.

```c
#include <unistd.h>

off_t lseek(int fd, off_t offset, int whence);
```

| `whence` | 기준 |
|---|---|
| `SEEK_SET` | 파일의 처음 |
| `SEEK_CUR` | 현재 자리 |
| `SEEK_END` | 파일의 끝 |

### 6.2 자주 쓰는 두 가지

```c
size = lseek(fd, 0, SEEK_END);      /* 파일 크기를 구한다 */
```

```c
pos  = lseek(fd, 0, SEEK_CUR);      /* 지금 어디인지 묻는다 */
```

### 6.3 파일 끝을 넘어가면

**파일 끝보다 뒤로 이동한 뒤 쓰면, 그 사이는 `0`으로 채워집니다.** 그런데 그 `0`들은 **디스크를 차지하지 않습니다.**

```c
    write(fd, "처음", 6);                       /* 한글 2자 = 6바이트 */
    pos = lseek(fd, 0, SEEK_CUR);
    printf("쓰고 난 뒤 위치      : %lld\n", (long long) pos);

    /* 파일 끝을 훌쩍 넘겨 이동한 뒤 쓰면 '구멍' 이 생긴다 */
    lseek(fd, 1024 * 1024, SEEK_SET);
    write(fd, "끝", 3);

    size = lseek(fd, 0, SEEK_END);
    printf("파일의 논리적 크기   : %lld 바이트\n", (long long) size);
```

```bash
./lseek_demo
```

```text
쓰고 난 뒤 위치      : 6
파일의 논리적 크기   : 1048579 바이트
구멍 속 바이트       : 0 0 0 0
```

```bash
ls -l sparse.dat
```

```text
-rw-r--r-- 1 student student 1048579 Nov 30 09:12 sparse.dat
```

```bash
du -h --apparent-size sparse.dat ; du -h sparse.dat
```

```text
1.1M	sparse.dat
8.0K	sparse.dat
```

**1MB짜리 파일이 디스크는 8KB만 쓰고 있습니다.** 이것을 **희소 파일(sparse file)** 이라 합니다.

| 명령 | 보여 주는 것 |
|---|---|
| `ls -l` | **논리적 크기** — 파일이 얼마나 커 보이는가 |
| `du --apparent-size` | 같은 논리적 크기 |
| `du` | **실제로 쓰는 디스크 공간** |

> **가상 머신 디스크 이미지나 데이터베이스 파일이 이 방식을 씁니다.** 그래서 "100GB 파일을 만들었는데 디스크가 줄지 않는" 현상이 생깁니다. 반대로 그런 파일을 그냥 복사하면 **구멍이 실제 0으로 채워져 진짜 100GB가 되는** 사고가 납니다. `cp --sparse=always`가 필요한 이유입니다.
{: .prompt-info }

### 6.4 `lseek`를 쓸 수 없는 것

| 대상 | `lseek` 가능? |
|---|---|
| 보통 파일 | 가능 |
| 블록 장치 | 가능 |
| **파이프·FIFO** | **불가**(`ESPIPE`) |
| **소켓** | **불가** |
| 터미널 | 불가 |

**스트림은 흘러갈 뿐 되감을 수 없습니다.** 9강에서 소켓을 다룰 때 이 제약이 프로토콜 설계를 좌우합니다.

---

## 제7절. 버퍼 크기와 성능

### 7.1 직접 재어 본다

1강 7.4절에서 "시스템 호출은 비싸다"고 했습니다. **얼마나 비싼지 직접 측정**합니다.

```c
    clock_gettime(CLOCK_MONOTONIC, &t0);

    while ((n = read(in, buf, bufsize)) > 0) {
        ssize_t done = 0;
        calls++;
        while (done < n) {
            ssize_t w = write(out, buf + done, (size_t) (n - done));
            if (w < 0) {
                if (errno == EINTR)
                    continue;
                fprintf(stderr, "write: %s\n", strerror(errno));
                goto fail;
            }
            calls++;
            done += w;
        }
        total += n;
    }
```

```bash
gcc -Wall -Wextra -std=gnu17 copy_buf.c -o copy_buf
```

64MB짜리 시험 파일을 만듭니다.

```bash
dd if=/dev/urandom of=big.dat bs=1M count=64 status=none
```

```bash
for bs in 1 16 256 4096 65536 1048576; do ./copy_buf big.dat copy.dat $bs; done
```

### 7.2 측정 결과

```text
버퍼       1 바이트 | 시스템 호출 134217728회 | 67108864 바이트 | 24.662초
버퍼      16 바이트 | 시스템 호출  8388608회 | 67108864 바이트 | 1.637초
버퍼     256 바이트 | 시스템 호출   524288회 | 67108864 바이트 | 0.123초
버퍼    4096 바이트 | 시스템 호출    32768회 | 67108864 바이트 | 0.028초
버퍼   65536 바이트 | 시스템 호출     2048회 | 67108864 바이트 | 0.026초
버퍼 1048576 바이트 | 시스템 호출      128회 | 67108864 바이트 | 0.025초
```

| 버퍼 크기 | 시스템 호출 | 걸린 시간 | 1바이트 대비 |
|---|---|---|---|
| 1 | 1억 3천만 회 | **24.662초** | 1배 |
| 16 | 838만 회 | 1.637초 | **15배 빠름** |
| 256 | 52만 회 | 0.123초 | **200배** |
| **4096** | 3만 회 | **0.028초** | **880배** |
| 65536 | 2048회 | 0.026초 | 948배 |
| 1048576 | 128회 | 0.025초 | 986배 |

> 절대 시간은 컴퓨터와 디스크에 따라 다릅니다. 그러나 **비율의 모양은 어디서나 같습니다.**
{: .prompt-info }

### 7.3 세 가지 교훈

**① 시스템 호출은 정말로 비쌉니다.**

같은 자료를 옮기는데 **880배 차이**가 났습니다. 자료의 양은 똑같습니다. 다른 것은 **커널을 몇 번 불렀는가**뿐입니다.

**② 그러나 무작정 키운다고 좋아지지 않습니다.**

4096에서 65536으로 열여섯 배 늘렸는데 개선은 7% 남짓입니다. 1MB까지 가도 마찬가지입니다.

| 왜 그런가 |
|---|
| 어느 지점부터는 **디스크 속도**가 병목이 된다 |
| 큰 버퍼는 메모리를 차지하고 캐시 효율을 떨어뜨린다 |
| 파일 시스템의 블록 크기(보통 4096)와 맞추는 것이 효율적이다 |

**③ 그래서 표준 라이브러리가 존재합니다.**

`fopen`이 기본으로 잡는 버퍼 크기는 대개 4096바이트입니다. 위 표에서 가장 좋은 지점입니다. **1부에서 `fgetc`를 한 글자씩 불러도 느리지 않았던 이유**가 이것입니다. 겉보기에는 한 글자씩이지만 실제로는 4096바이트씩 읽고 있었습니다.

### 7.4 알맞은 버퍼 크기 고르기

| 상황 | 권장 |
|---|---|
| 보통 파일 복사 | 4KB ~ 64KB |
| 파일 시스템 블록에 맞추고 싶다 | `stat`의 `st_blksize`(3강) |
| 네트워크 | 4KB ~ 16KB |
| 아주 큰 파일 | 64KB ~ 1MB |

---

## 제8절. `fopen`으로는 할 수 없는 일

### 8.1 `O_EXCL` — 경쟁 없이 만들기

**"파일이 없으면 만든다"** 를 다음처럼 쓰면 안 됩니다.

```c
if (access(path, F_OK) != 0) {       /* 없는지 확인하고 */
    fd = open(path, O_WRONLY | O_CREAT, 0644);   /* 만든다 */
}
```

확인과 생성 **사이의 짧은 순간**에 다른 프로세스가 같은 파일을 만들 수 있습니다. 이것을 **경쟁 조건(race condition)** 이라 하며, 보안 취약점의 고전적 유형입니다.

`O_EXCL`은 이 둘을 **하나의 나눌 수 없는 동작**으로 만듭니다.

```c
    fd = open(lock, O_WRONLY | O_CREAT | O_EXCL, 0644);
    if (fd == -1) {
        if (errno == EEXIST) {
            printf("이미 다른 쪽이 잠갔습니다. 물러납니다.\n");
            return 1;
        }
        fprintf(stderr, "%s: %s\n", lock, strerror(errno));
        return 1;
    }

    printf("잠금을 얻었습니다 (서술자 %d)\n", fd);
```

```bash
./excl_demo
```

```text
잠금을 얻었습니다 (서술자 3)
잠금을 풀었습니다.
```

```bash
touch my.lock ; ./excl_demo ; echo $?
```

```text
이미 다른 쪽이 잠갔습니다. 물러납니다.
1
```

**커널이 보장합니다.** 두 프로세스가 정확히 같은 순간에 시도해도 **성공하는 쪽은 하나뿐**입니다.

> 이 방식은 **잠금 파일(lock file)** 의 기본 원리입니다. 다만 프로그램이 비정상 종료하면 파일이 남아 영원히 잠긴 상태가 되므로, 실무에서는 PID를 함께 기록하거나 `flock`을 씁니다.
{: .prompt-info }

### 8.2 `O_APPEND` — 원자적인 덧붙이기

여러 프로세스가 **같은 로그 파일**에 쓰는 상황을 생각해 보십시오.

| 방식 | 문제 |
|---|---|
| `lseek(fd, 0, SEEK_END)` 후 `write` | 두 동작 **사이에** 다른 쪽이 쓰면 **덮어쓴다** |
| **`O_APPEND`로 열고 `write`** | 커널이 **자리 이동과 쓰기를 하나로** 처리한다 |

```c
fd = open("app.log", O_WRONLY | O_CREAT | O_APPEND, 0644);
```

**여러 프로세스가 같은 파일에 로그를 남길 수 있는 것**이 이 플래그 덕분입니다. 셸의 `>>`도 이 플래그를 씁니다.

### 8.3 `O_CLOEXEC` — 물려주지 않기

4강에서 배울 `exec`는 **열린 서술자를 그대로 물려줍니다.** 그런데 물려주면 안 되는 것들이 있습니다.

| 물려주면 생기는 문제 |
|---|
| 자식이 부모의 비밀 파일을 읽을 수 있다 |
| 자식이 서술자를 붙들고 있어 자원이 반환되지 않는다 |

```c
fd = open("secret.key", O_RDONLY | O_CLOEXEC);
```

**14강에서 개인 키 파일을 열 때 반드시 쓸 플래그**입니다.

### 8.4 `O_NONBLOCK` — 기다리지 않기

기본적으로 `read`는 자료가 올 때까지 **멈춰 서서 기다립니다.** `O_NONBLOCK`을 주면 자료가 없을 때 곧바로 `-1`과 `EAGAIN`을 돌려줍니다.

**10강에서 다중 접속 서버를 만들 때 핵심이 되는 플래그**입니다. 지금은 이런 것이 있다는 정도만 알아 두십시오.

### 8.5 정리

| 하고 싶은 일 | `fopen` | `open` |
|---|---|---|
| 보통 읽고 쓰기 | **편하다** | 번거롭다 |
| 서식 있는 입출력 | **`fprintf`** | 직접 만들어야 |
| 경쟁 없이 만들기 | 불가 | **`O_EXCL`** |
| 원자적 덧붙이기 | 불완전 | **`O_APPEND`** |
| 물려주지 않기 | 불가 | **`O_CLOEXEC`** |
| 막히지 않는 입출력 | 불가 | **`O_NONBLOCK`** |
| 소켓 | **불가** | **필수** |

---

## 제9절. 파일 서술자 표

### 9.1 번호는 어떻게 정해지는가

```c
    a = open("/etc/hostname", O_RDONLY);
    b = open("/etc/hostname", O_RDONLY);
    printf("a = %d, b = %d\n", a, b);

    close(a);                       /* 가운데를 비운다 */
    c = open("/etc/hostname", O_RDONLY);
    printf("a 를 닫은 뒤 새로 연 c = %d   <- 가장 작은 빈 번호\n", c);
```

```bash
./fdtable
```

```text
a = 3, b = 4
a 를 닫은 뒤 새로 연 c = 3   <- 가장 작은 빈 번호

/proc/self/fd 를 들여다봅니다:
total 0
lrwx------ 1 student student 64 Nov 30 09:20 0 -> /dev/pts/0
lrwx------ 1 student student 64 Nov 30 09:20 1 -> /dev/pts/0
lrwx------ 1 student student 64 Nov 30 09:20 2 -> /dev/pts/0
lr-x------ 1 student student 64 Nov 30 09:20 3 -> /etc/hostname
lr-x------ 1 student student 64 Nov 30 09:20 4 -> /etc/hostname
lr-x------ 1 student student 64 Nov 30 09:20 5 -> /proc/1234/fd
```

> **규칙: 커널은 언제나 "쓸 수 있는 가장 작은 번호"를 준다.**
{: .prompt-tip }

이 규칙이 **6강에서 재지정을 구현할 때 결정적**으로 쓰입니다. 서술자 1을 닫고 곧바로 파일을 열면, 그 파일이 **표준 출력이 되기** 때문입니다.

### 9.2 `/proc`으로 들여다보기

| 확인할 것 | 뜻 |
|---|---|
| 0·1·2 → `/dev/pts/0` | **셋 모두 같은 터미널**을 가리킨다 |
| 3·4 → `/etc/hostname` | 같은 파일을 두 번 열었다 |
| 5 → `/proc/1234/fd` | `ls` 명령이 이 디렉터리를 읽는 중 |

**돌고 있는 다른 프로세스도 볼 수 있습니다.**

```bash
ls -l /proc/$(pgrep -n bash)/fd
```

서버가 서술자를 흘리고 있는지 확인할 때 이 방법을 씁니다. 목록이 계속 길어진다면 `close`를 빠뜨린 것입니다.

### 9.3 세 층의 자료 구조

같은 파일을 두 번 열었을 때 커널 안에서 벌어지는 일입니다.

| 층 | 무엇 | 누구의 것 |
|---|---|---|
| 1 | **서술자 표** — 번호 → 열린 파일 | **프로세스마다** |
| 2 | **열린 파일 설명** — 위치, 플래그 | 열 때마다 하나 |
| 3 | **아이노드** — 파일의 실체 | 파일마다 하나 |

```text
 프로세스 A                     열린 파일 설명            아이노드
 ┌───────────┐
 │ 3 ────────┼──────────▶ [위치 0,  O_RDONLY] ──┐
 │ 4 ────────┼──────────▶ [위치 0,  O_RDONLY] ──┼──▶ /etc/hostname
 └───────────┘                                  │
 프로세스 B                                      │
 ┌───────────┐                                  │
 │ 3 ────────┼──────────▶ [위치 50, O_RDONLY] ──┘
 └───────────┘
```

| 결과 |
|---|
| 따로 `open` 하면 **위치가 따로** 움직인다 |
| `dup`으로 복제하면 **위치를 공유**한다(6강) |
| `fork`로 물려받으면 **위치를 공유**한다(4강) |

이 그림이 4강과 6강의 바탕이 됩니다. 지금 이해해 두면 나중이 훨씬 쉽습니다.

---

## 제10절. 어느 쪽을 쓸 것인가

**아래층이 언제나 좋은 것은 아닙니다.**

| 상황 | 권장 | 이유 |
|---|---|---|
| 텍스트를 줄 단위로 | **`FILE *`** | `fgets`가 훨씬 편하다 |
| 서식 있는 출력 | **`FILE *`** | `fprintf` |
| 설정 파일 읽기 | **`FILE *`** | 성능이 문제되지 않는다 |
| 대용량 복사 | **서술자** | 버퍼를 직접 조절 |
| **소켓** | **서술자** | 선택의 여지가 없다 |
| 특수 플래그가 필요 | **서술자** | `O_EXCL` 등 |
| 다중 입출력 감시 | **서술자** | `select`·`epoll`(10강) |
| 정확한 원자성이 필요 | **서술자** | `O_APPEND` |

> **둘을 섞을 수는 있습니다.** `fdopen`으로 서술자를 `FILE *`로 감싸거나, `fileno`로 서술자를 꺼낼 수 있습니다. 다만 1.3절에서 보았듯 **같은 대상에 두 방식으로 동시에 쓰면 순서가 무너집니다.**
{: .prompt-warning }

```c
FILE *fp = fdopen(fd, "r");     /* 서술자를 FILE * 로 감싼다 */
int   fd2 = fileno(fp);         /* FILE * 에서 서술자를 꺼낸다 */
```

`fdopen`은 9강에서 소켓을 줄 단위로 읽고 싶을 때 유용하게 쓰입니다.

---

## 제11절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| 파일이 비어 버렸다 | `O_TRUNC` | 덧붙이려면 `O_APPEND` |
| 권한이 이상하다 | `644`(10진수)로 씀 | **`0644`** |
| `open`이 `EACCES` | 권한 부족 | `ls -l`로 확인 |
| 자료가 잘려 저장됨 | `write` 반환값 무시 | **`write_full`** |
| 파일 끝에서 무한 반복 | `read`의 0을 처리 안 함 | `r == 0`이면 중단 |
| `Too many open files` | 서술자 누수 | 모든 경로에서 `close` |
| 출력 순서가 뒤죽박죽 | `FILE *`와 `write` 혼용 | 하나만 쓰거나 `fflush` |
| 복사가 너무 느림 | 버퍼가 작다 | **4KB 이상** |
| `lseek`가 `ESPIPE` | 파이프·소켓 | 되감을 수 없다 |
| 큰 파일을 복사했더니 용량 폭증 | 희소 파일 | `cp --sparse=always` |
| `PATH_MAX` 없음 | `-std=c17` | **`-std=gnu17`**(1강 8.2절) |

---

> **▶ 여기서부터 3회차 — 스스로 해 보기**
> 실습문제, 약 110분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 실습문제

> **안내**
> 1. 컴파일은 **`gcc -Wall -Wextra -std=gnu17`**, **경고 0개**여야 합니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 |
|---|---|---|
| 문제 1 | `FILE *`와 서술자 혼용 관찰 | 1.3 |
| 문제 2 | `mycat` 완성 | 3.2 |
| 문제 3 | `write_full`·`read_full` | 4 |
| 문제 4 | 버퍼 크기 성능 측정 | 7 |
| 문제 5 | 파일 크기 구하기 | 6.2 |
| 문제 6 | 희소 파일 만들기 | 6.3 |
| 문제 7 | `O_EXCL` 잠금 | 8.1 |
| 문제 8 | `O_APPEND` 확인 | 8.2 |
| 문제 9 | 서술자 누수 재현 | 5.2 |
| 문제 10 | `tail -n` 흉내 내기 | 6 · 4 |

---

### 문제 1. `FILE *`와 서술자 혼용 관찰

`fileno_demo`를 화면과 파일로 각각 실행하여 **출력 순서가 달라지는 것**을 확인하고 이유를 설명하십시오.

**정답 및 해설**

| 실행 | 결과 |
|---|---|
| `./fileno_demo` | `[1] [2] [3]` 순서대로 |
| `./fileno_demo > o.txt` | **`[2]`가 맨 앞으로** |

- 화면은 **줄 단위 버퍼링**이라 `printf`가 개행마다 곧바로 `write`를 부릅니다.
- 파일로 재지정하면 **블록 단위 버퍼링**으로 바뀌어, `printf`의 출력은 모두 버퍼에 머물다가 **종료 시점에** 한꺼번에 나갑니다.
- `write`는 버퍼를 거치지 않으므로 그동안 먼저 도착합니다.
- **해결**: `write` 직전에 `fflush(stdout)`을 넣으면 순서가 유지됩니다. 직접 확인해 보십시오.

---

### 문제 2. `mycat` 완성

`open`·`read`·`write`만으로 `cat`을 구현하십시오. 여러 파일, 표준 입력, 오류 처리를 모두 지원해야 합니다.

**정답 및 해설**

3.2절의 코드가 답입니다. 확인할 점은 다음과 같습니다.

| 확인 | 방법 |
|---|---|
| 여러 파일 | `./mycat a.txt b.txt` |
| 표준 입력 | `echo hi \| ./mycat` |
| 파이프 연결 | `./mycat a.txt \| ./mycat` |
| 없는 파일 | 오류를 내고 **나머지는 계속** |
| 종료 상태 | 실패가 하나라도 있으면 1 |

- **`printf`를 쓰지 않은 것**이 중요합니다. 이 프로그램은 이진 파일도 그대로 통과시킬 수 있어야 합니다. `printf("%s")`를 쓰면 `'\0'`에서 잘립니다.
- 버퍼 크기를 4096으로 잡은 근거는 제7절의 측정 결과입니다. 근거 없이 고른 숫자가 아닙니다.

---

### 문제 3. `write_full`·`read_full`

부분 처리에 대응하는 두 함수를 만들고 `io_util.h`/`io_util.c`로 분리하십시오.

**정답 및 해설**

4.2·4.3절의 코드가 답입니다.

| 함수 | 성공 | 실패 | 특별한 경우 |
|---|---|---|---|
| `write_full` | 0 | -1 | — |
| `read_full` | 읽은 바이트 수 | -1 | **n보다 작으면 파일 끝** |

- 두 함수의 반환 규약이 **다르다**는 점에 주의하십시오. 읽기는 "몇 바이트나 얻었는가"가 중요하고, 쓰기는 "다 썼는가"만 중요합니다.
- `EINTR`을 다시 시도로 처리한 것이 핵심입니다. 5강에서 시그널을 배우면 이 한 줄이 없어 서버가 죽는 사례를 보게 됩니다.
- **부분 처리를 확인하는 방법**: 파이프로 큰 자료를 보내면서 읽는 쪽을 느리게 하면 `write`가 적게 쓰는 것을 볼 수 있습니다. 6강에서 실제로 만들어 봅니다.

---

### 문제 4. 버퍼 크기 성능 측정

64MB 파일을 여러 버퍼 크기로 복사하고 결과표를 만드십시오.

**정답 및 해설**

```bash
dd if=/dev/urandom of=big.dat bs=1M count=64 status=none
for bs in 1 16 256 4096 65536 1048576; do ./copy_buf big.dat copy.dat $bs; done
```

측정 예시입니다.

| 버퍼 | 시스템 호출 | 시간 |
|---|---|---|
| 1 | 134,217,728 | 24.662초 |
| 16 | 8,388,608 | 1.637초 |
| 256 | 524,288 | 0.123초 |
| 4096 | 32,768 | **0.028초** |
| 65536 | 2,048 | 0.026초 |
| 1048576 | 128 | 0.025초 |

- **1바이트 버퍼는 쓰지 마십시오.** 880배 느립니다.
- **4096 이후로는 개선이 거의 없습니다.** 병목이 시스템 호출에서 디스크로 옮겨 갔기 때문입니다.
- 두 번째 실행이 더 빠르게 나올 수 있습니다. **페이지 캐시**에 자료가 남아 있기 때문입니다. 공정한 비교를 하려면 매번 파일을 바꾸거나 캐시를 비워야 합니다.
- 시스템 호출 횟수가 `read`와 `write`를 합한 값임에 주의하십시오. 그래서 64MB ÷ 1바이트 = 6700만의 **두 배**가 나왔습니다.

---

### 문제 5. 파일 크기 구하기

`lseek`로 파일 크기를 구하는 함수를 만들고, `ls -l`의 값과 비교하십시오.

**정답 및 해설**

```c
/* file_size: 파일의 크기를 돌려준다. 실패하면 -1 */
static off_t file_size(int fd)
{
    off_t cur, size;

    cur = lseek(fd, 0, SEEK_CUR);          /* 지금 자리를 기억해 두고 */
    if (cur == (off_t) -1)
        return -1;
    size = lseek(fd, 0, SEEK_END);         /* 끝으로 가서 크기를 얻고 */
    if (size == (off_t) -1)
        return -1;
    if (lseek(fd, cur, SEEK_SET) == (off_t) -1)   /* 원래 자리로 되돌린다 */
        return -1;
    return size;
}
```

- **원래 자리로 되돌리는 것**이 중요합니다. 크기를 재는 함수가 파일 위치를 바꿔 버리면, 그 뒤의 `read`가 엉뚱한 곳에서 시작합니다. **관찰이 대상을 바꾸면 안 됩니다.**
- 실패 판정은 `(off_t) -1`과 비교합니다. `off_t`가 부호 있는 형이기 때문입니다.
- 실무에서는 3강에서 배울 **`fstat`**을 씁니다. 위치를 건드리지 않고, 파이프에도 쓸 수 있으며, 크기 말고도 많은 것을 알려 줍니다.

---

### 문제 6. 희소 파일 만들기

1MB 크기의 희소 파일을 만들고 `ls`와 `du`의 값을 비교하십시오.

**정답 및 해설**

6.3절의 코드로 만든 뒤

```bash
ls -l sparse.dat ; du -h --apparent-size sparse.dat ; du -h sparse.dat
```

```text
-rw-r--r-- 1 student student 1048579 Nov 30 09:12 sparse.dat
1.1M	sparse.dat
8.0K	sparse.dat
```

- **논리적으로는 1MB, 실제로는 8KB**입니다. 구멍은 디스크를 차지하지 않습니다.
- 읽으면 `0`이 나옵니다. 커널이 **그 자리에 아무것도 없음을 알고 0을 만들어 줍니다.**
- 8KB는 실제로 쓴 두 조각(앞의 6바이트, 뒤의 3바이트)이 각각 블록 하나씩을 차지한 결과입니다.
- `cp sparse.dat copy.dat` 후 `du`를 다시 재어 보십시오. 파일 시스템과 `cp` 판에 따라 구멍이 유지되기도 하고 실제 0으로 채워지기도 합니다.

---

### 문제 7. `O_EXCL` 잠금

잠금 파일로 **한 번에 하나만** 실행되는 프로그램을 만드십시오.

**정답 및 해설**

8.1절의 코드가 답입니다. 두 터미널에서 동시에 실행해 확인하십시오.

- **`access` 후 `open`은 틀린 방법**임을 설명할 수 있어야 합니다. 두 동작 사이에 틈이 있어, 그 순간 다른 프로세스가 끼어들 수 있습니다. 이런 취약점을 **TOCTOU**(time-of-check to time-of-use)라 부릅니다.
- 1부 10강에서 배운 **신뢰 경계**와 같은 문제입니다. "확인했으니 괜찮겠지"라는 가정이 무너지는 지점입니다.
- 개선점: 잠금 파일에 PID를 적어 두면, 남겨진 잠금이 진짜 살아 있는 프로세스의 것인지 확인할 수 있습니다.

```c
    char line[32];
    int len = snprintf(line, sizeof line, "%d\n", (int) getpid());
    write(fd, line, (size_t) len);
```

---

### 문제 8. `O_APPEND` 확인

`O_APPEND` 없이 두 프로세스가 같은 파일에 쓰면 어떻게 되는지 확인하고, 있을 때와 비교하십시오.

**정답 및 해설**

| 방식 | 결과 |
|---|---|
| `lseek(SEEK_END)` 후 `write` | 두 프로세스가 **같은 자리에 써서 덮어쓴다** |
| `O_APPEND` | 모든 줄이 온전히 남는다 |

```bash
./appender a & ./appender b & wait ; wc -l app.log
```

- `O_APPEND`가 없으면 줄 수가 기대보다 **적게** 나옵니다. 자리 이동과 쓰기 사이에 다른 프로세스가 끼어들었기 때문입니다.
- `O_APPEND`는 그 둘을 **커널이 하나로 묶어** 처리합니다. 프로그램이 아무리 노력해도 사용자 공간에서는 이 보장을 만들 수 없습니다.
- 셸의 `>>`가 바로 이 플래그입니다. 반면 `>`는 `O_TRUNC`입니다.

---

### 문제 9. 서술자 누수 재현

파일을 열기만 하고 닫지 않는 프로그램을 만들어 **한계에 도달**시키고, 그때의 `errno`를 확인하십시오.

**정답 및 해설**

```c
/* fdleak.c — 서술자를 흘리면 어떻게 되는가 */
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>

int main(void)
{
    int i, fd;

    for (i = 0; ; i++) {
        fd = open("/etc/hostname", O_RDONLY);   /* 닫지 않는다 */
        if (fd == -1) {
            printf("%d 번째에서 실패: %s (errno=%d)\n",
                   i, strerror(errno), errno);
            break;
        }
    }
    return 0;
}
```

```bash
ulimit -n
```

```text
1024
```

```bash
./fdleak
```

```text
1021 번째에서 실패: Too many open files (errno=24)
```

- **멈추는 지점은 `ulimit -n` 값에서 3을 뺀 수**입니다. 0·1·2가 이미 쓰이고 있기 때문입니다. 한계가 10240인 환경에서는 10237에서 멈춥니다. **자신의 환경에서 `ulimit -n`을 먼저 확인하고, 예측한 값과 실제가 맞는지 보십시오.**
- `errno`는 `EMFILE`(24)입니다. **프로세스별 한계**를 넘었다는 뜻입니다. 시스템 전체 한계를 넘으면 `ENFILE`이 됩니다.
- **서버 프로그램에서 가장 흔한 자원 누수**입니다. 접속마다 서술자를 하나씩 쓰는데 닫지 않으면 정확히 이 지점에서 서비스가 멈춥니다.
- 확인 방법: 프로그램을 잠시 멈춘 뒤 다른 터미널에서

```bash
ls /proc/$(pgrep fdleak)/fd | wc -l
```

---

### 문제 10. `tail -n` 흉내 내기

파일의 **마지막 N줄**을 출력하는 프로그램을 만드십시오. 큰 파일에서도 빨라야 합니다.

**정답 및 해설**

**핵심은 파일 전체를 읽지 않는 것**입니다. 끝에서부터 거꾸로 읽습니다.

```c
/* mytail.c — 파일의 마지막 n 줄을 출력한다 (핵심 부분) */
#define CHUNK 4096

    size = lseek(fd, 0, SEEK_END);
    if (size == (off_t) -1) {
        fprintf(stderr, "%s: 되감을 수 없는 파일입니다\n", path);
        close(fd);
        return 1;
    }
    pos = size;

    if (n == 0)                     /* 0 줄을 요청하면 아무것도 내보내지 않는다 */
        goto found;

    /* 끝에서부터 거꾸로 읽으며 개행을 센다 */
    while (pos > 0) {
        size_t want = (pos >= (off_t) CHUNK) ? (size_t) CHUNK : (size_t) pos;

        pos -= (off_t) want;
        if (lseek(fd, pos, SEEK_SET) == (off_t) -1)
            break;
        if (read_full(fd, buf, want) != (ssize_t) want)
            break;

        for (i = (int) want - 1; i >= 0; i--) {
            if (buf[i] != '\n')
                continue;
            if (pos + i + 1 == size)    /* 파일 맨 끝의 개행은 세지 않는다 */
                continue;
            lines++;
            if (lines == n) {           /* n 개째 개행 바로 뒤가 시작 지점 */
                pos += i + 1;
                goto found;
            }
        }
    }

found:
    lseek(fd, pos, SEEK_SET);
    /* 여기서부터 끝까지 그대로 내보낸다 */
```

**정답을 스스로 채점하는 방법**이 있습니다. 진짜 `tail`과 비교하십시오.

```bash
seq 1 100000 > nums.txt
diff <(./mytail 3 nums.txt) <(tail -3 nums.txt) && echo 일치
```

여섯 가지 경우를 모두 시험해야 합니다.

| 경우 | 만드는 법 | 주의할 점 |
|---|---|---|
| 보통 파일 | `seq 1 100000 > nums.txt` | 기본 |
| 요청이 파일보다 많음 | `printf 'a\nb\n' > tiny.txt` | 전체를 낸다 |
| **마지막 줄에 개행이 없음** | `printf 'x\ny\nz' > noeol.txt` | 가장 많이 틀리는 곳 |
| 빈 파일 | `: > empty.txt` | 아무것도 내지 않는다 |
| **0줄 요청** | `./mytail 0 nums.txt` | 아무것도 내지 않는다 |
| 파이프 | `cat nums.txt \| ./mytail 3 /dev/stdin` | **되감을 수 없다** |

- **두 개의 함정이 있습니다.** 첫째, **파일 맨 끝의 개행은 세면 안 됩니다.** 그것은 마지막 줄의 끝일 뿐 새 줄의 시작이 아닙니다. 둘째, **`n`개째 개행에서 멈춰야** 합니다. `lines > n`으로 쓰면 한 줄이 더 나옵니다.
- **`lseek`가 있어야 가능한 알고리즘**입니다. `FILE *`로도 `fseek`가 있으니 가능하지만, **파이프에서는 어느 쪽으로도 불가능**합니다. 그래서 진짜 `tail`은 파이프 입력일 때 전체를 읽어 버퍼에 담는 다른 알고리즘을 씁니다.
- 10만 줄 파일에서 마지막 3줄을 얻는 데 **4096바이트 한 번만 읽으면 됩니다.** 전체를 읽는 방식과 비교하면 압도적으로 빠릅니다.
- 1부 12강에서 배운 시험 계획 네 갈래(정상·경계·오류·자원)를 그대로 적용해 보십시오.

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 소스 — `mycat.c`, `copy_buf.c`, `lseek_demo.c`, `excl_demo.c`, `fdtable.c`, `fdleak.c`, `mytail.c`, `io_util.h`/`io_util.c` |
| 2 | 문제 1의 두 실행 화면 비교와 설명 |
| 3 | **버퍼 크기 성능 측정표**(문제 4) |
| 4 | 희소 파일의 `ls`·`du` 비교(문제 6) |
| 5 | `O_EXCL` 잠금 동작 화면(문제 7) |
| 6 | 서술자 누수의 `errno`와 한계 값(문제 9) |
| 7 | `mytail` 시험 결과표 — 정상·경계·오류(문제 10) |
| 8 | 짧은 서술 ① `FILE *`와 파일 서술자의 관계 |
| 9 | 짧은 서술 ② `read`가 요청보다 적게 읽을 수 있는 상황 네 가지 |

---

## 정리

| 구분 | 핵심 |
|---|---|
| `FILE *` | **파일 서술자를 감싼 껍데기.** 안에 버퍼가 있다 |
| 혼용 | 같은 대상에 두 방식을 섞으면 **순서가 무너진다** |
| `open` | 접근 방식 하나 + 부가 플래그. `O_CREAT`일 때만 권한 인자 |
| 권한 | **`0644`** — 앞의 0을 빠뜨리지 말 것. 실제 권한은 `mode & ~umask` |
| **부분 처리** | `read`·`write`는 **요청보다 적게 처리할 수 있다.** `_full` 함수 필수 |
| `read`의 0 | 오류가 아니라 **파일 끝** |
| `close` | 빠뜨리면 **`EMFILE`**. 실패해도 다시 닫지 않는다 |
| `lseek` | 크기 구하기, 희소 파일. **파이프·소켓에는 불가** |
| 버퍼 크기 | 1바이트와 4KB가 **880배** 차이. 4KB 넘으면 이득이 작다 |
| `open`만의 것 | `O_EXCL`·`O_APPEND`·`O_CLOEXEC`·`O_NONBLOCK` |
| 서술자 배정 | **가장 작은 빈 번호** — 6강 재지정의 열쇠 |

---

## 다음 강의 예고

**3강 「파일 시스템과 메타데이터」** 에서는 파일의 **내용이 아니라 정보**를 다룹니다.

- `stat`으로 크기·권한·시각·아이노드를 읽는다
- 권한 검사가 실제로 어떻게 이루어지는가
- 디렉터리를 순회한다 — `du`를 직접 만든다
- 하드 링크와 심벌릭 링크의 차이를 실험으로 확인한다
- **원자적 파일 갱신** — 임시 파일과 `rename`

파일이 "이름과 내용의 짝"이 아니라는 사실을 알게 됩니다.

---

## 부록 A. 이번 강의 명령·함수 요약

| 하려는 일 | 함수 |
|---|---|
| 파일 열기 | `open(path, flags[, mode])` |
| 읽기 | `read(fd, buf, count)` |
| 쓰기 | `write(fd, buf, count)` |
| 닫기 | `close(fd)` |
| 자리 이동 | `lseek(fd, off, whence)` |
| 크기 구하기 | `lseek(fd, 0, SEEK_END)` |
| `FILE *` → 서술자 | `fileno(fp)` |
| 서술자 → `FILE *` | `fdopen(fd, mode)` |
| 열린 서술자 확인 | `ls -l /proc/PID/fd` |
| 한계 확인 | `ulimit -n` |
| 실제 디스크 사용량 | `du -h` |
| 논리적 크기 | `du -h --apparent-size`, `ls -l` |

## 부록 B. 표준 문서와 출처

**이번 강의에서 다룬 시스템 호출**

| 함수 | 리눅스 man | POSIX 표준 |
|---|---|---|
| `open` | [`open(2)`](https://man7.org/linux/man-pages/man2/open.2.html) | [`open()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/open.html) |
| `read` | [`read(2)`](https://man7.org/linux/man-pages/man2/read.2.html) | [`read()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/read.html) |
| `write` | [`write(2)`](https://man7.org/linux/man-pages/man2/write.2.html) | [`write()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/write.html) |
| `close` | [`close(2)`](https://man7.org/linux/man-pages/man2/close.2.html) | [`close()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/close.html) |
| `lseek` | [`lseek(2)`](https://man7.org/linux/man-pages/man2/lseek.2.html) | [`lseek()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/lseek.html) |
| `fileno` | [`fileno(3)`](https://man7.org/linux/man-pages/man3/fileno.3.html) | [`fileno()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/fileno.html) |
| `fdopen` | [`fdopen(3)`](https://man7.org/linux/man-pages/man3/fdopen.3.html) | [`fdopen()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/fdopen.html) |
| `umask` | [`umask(2)`](https://man7.org/linux/man-pages/man2/umask.2.html) | [`umask()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/umask.html) |
| 서술자 확인 | [`proc_pid_fd(5)`](https://man7.org/linux/man-pages/man5/proc_pid_fd.5.html) | — (리눅스 전용) |

**본문의 주장과 근거**

| 주장 | 근거 |
|---|---|
| `read`·`write`는 적게 처리할 수 있다(4.1절) | `read(2)`·`write(2)`의 RETURN VALUE 원문 |
| 버퍼 1바이트와 4KB가 880배 차이(7.2절) | 64MB 파일 복사 **실측**. `copy_buf.c`로 재현 가능 |
| 희소 파일은 `ls` 1048579 vs `du` 8.0K(6.3절) | `lseek_demo.c` 실행 후 **실측** |
| 서술자는 가장 작은 빈 번호(9.1절) | `fdtable.c` 실행 결과 + `open(2)`의 명세 |
| 블록 크기 4096 | `stat -c '%o'` |
| `EMFILE`은 `ulimit -n` − 3에서(문제 9) | `fdleak.c` 실측. 한계값은 환경마다 다름 |
| `mytail`이 `tail`과 일치(문제 10) | `diff <(./mytail 3 f) <(tail -3 f)` — 6가지 경계 조건 전부 확인 |

> **이 표의 목적은 "믿으라"가 아니라 "확인해 보라"입니다.** 측정값은 모두 재현 절차를 함께 적어 두었습니다. 자기 환경에서 다른 값이 나온다면 그것이 틀린 것이 아니라, **환경이 다르다는 사실을 알아낸 것**입니다. 왜 다른지 설명할 수 있다면 그 편이 더 나은 학습입니다.
{: .prompt-tip }

## 부록 C. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `fileno_demo.c` | `FILE *` 속의 서술자, 혼용의 함정 | 1 |
| `mycat.c` | `cat` 구현, `write_full` | 3 · 4 |
| `copy_buf.c` | 버퍼 크기별 성능 측정 | 7 |
| `lseek_demo.c` | 파일 위치와 희소 파일 | 6 |
| `excl_demo.c` | `O_EXCL` 잠금 | 8.1 |
| `fdtable.c` | 서술자 번호 배정, `/proc/self/fd` | 9 |
| `fdleak.c` | 서술자 누수와 `EMFILE` | 문제 9 |
| `mytail.c` | 마지막 N줄 출력 | 문제 10 |
