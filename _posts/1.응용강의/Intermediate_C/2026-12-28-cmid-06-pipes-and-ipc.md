---
title: 중급 C 프로그래밍 6강 - 파이프와 프로세스 간 통신
date: 2026-12-28 09:00:00 +0900
categories:
  - 1.응용강의
  - 중급 C 프로그래밍
tags:
  - C언어
  - 시스템프로그래밍
  - 리눅스
  - 파이프
  - dup2
  - 재지정
  - 셸
  - FIFO
  - mmap
  - IPC
pin:
mermaid: false
---

> **학습 목표**
> 1. 파이프가 **커널 안의 버퍼**임을 설명하고 `pipe`로 만들 수 있다.
> 2. **안 쓰는 쪽을 닫아야 하는 이유**를 실험으로 보일 수 있다.
> 3. 파이프의 용량과 원자적 쓰기 한계를 확인할 수 있다.
> 4. **`dup2`로 재지정을 구현**할 수 있다.
> 5. 미니 셸에 **`>`·`<`·`>>`** 를 넣을 수 있다.
> 6. 미니 셸에 **`|`** 를 넣어 여러 명령을 이을 수 있다.
> 7. `popen`·`pclose`로 간단한 경우를 처리할 수 있다.
> 8. **FIFO**로 관계없는 프로세스끼리 통신할 수 있다.
> 9. **`mmap`으로 메모리를 공유**할 수 있고, `fork`의 복사와 대비해 설명할 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

4강에서 프로세스를 만들었고, 5강에서 시그널로 **아주 짧은 알림**을 주고받았습니다. 그러나 시그널로는 **자료를 보낼 수 없습니다.**

이번 강의에서 프로세스 사이에 **자료가 흐르는 관**을 놓습니다.

두 가지 준비물이 여기서 쓰입니다.

| 준비해 둔 것 | 어디서 | 여기서 어떻게 |
|---|---|---|
| **커널은 가장 작은 빈 번호를 준다** | 2강 9.1절 | 재지정의 옛 방식 |
| **`fork`는 서술자를 물려준다**(위치까지 공유) | 4강 3.3절 | **파이프가 이어지는 원리** |
| `exec` 후에도 서술자는 살아남는다 | 4강 5.4절 | 새 프로그램이 파이프를 물려받는다 |

그리고 **미니 셸을 완성합니다.** 4강에서 뼈대를, 5강에서 Ctrl+C 처리를 만들었고, 오늘 재지정과 파이프를 넣으면 실제로 쓸 만해집니다.

**기준 교재: APUE 3판 15장(Interprocess Communication)·17장(Advanced IPC).**

이 강의는 **4회차 분량**(모두 합쳐 약 500분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제4절 | 파이프의 기본과 규칙 | 140분 |
| **2회차** | 제5절 ~ 제7절 | 재지정과 파이프 — 미니 셸 완성 | 140분 |
| **3회차** | 제8절 ~ 제11절 | `popen`·FIFO·`mmap` | 110분 |
| **4회차** | 실습문제 | 스스로 해 보기 | 110분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 파이프란 | 25분 |
| 제2절 | `pipe`로 관 놓기 | 40분 |
| 제3절 | 반드시 지켜야 할 규칙 | 45분 |
| 제4절 | 파이프의 용량과 원자성 | 30분 |
| 제5절 | `dup2` — 재지정의 원리 | 40분 |
| 제6절 | 미니 셸에 재지정 넣기 | 45분 |
| 제7절 | **미니 셸에 파이프 넣기** | 55분 |
| 제8절 | `popen` — 간편한 방법 | 25분 |
| 제9절 | FIFO — 이름 있는 파이프 | 35분 |
| 제10절 | `mmap` — 메모리 공유 | 35분 |
| 제11절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 디렉터리**는 `~/cmid/lab06`을 사용합니다. 이번 강의는 **VM 한 대**(`c-srv`)에서 진행합니다.

```bash
mkdir -p ~/cmid/lab06 && cd ~/cmid/lab06
```

---

## 제1절. 파이프란

### 1.1 커널 안의 버퍼

```c
#include <unistd.h>

int pipe(int pipefd[2]);
```

`pipe`는 **서술자 두 개를 한꺼번에** 만듭니다.

| 첨자 | 쪽 |
|---|---|
| `pipefd[0]` | **읽는 쪽** |
| `pipefd[1]` | **쓰는 쪽** |

```text
   쓰는 쪽                 커널 안의 버퍼                 읽는 쪽
  pipefd[1] ──write──▶ [ ■ ■ ■ □ □ □ □ ] ──read──▶ pipefd[0]
                        (기본 64 KiB)
```

**외우는 요령**: 표준 입력이 0이고 출력이 1인 것과 같습니다. **0번이 읽기, 1번이 쓰기**입니다.

### 1.2 성질

| 성질 | 설명 |
|---|---|
| **단방향** | 한쪽으로만 흐른다. 양방향이 필요하면 파이프 **두 개** |
| **경계가 없다** | 바이트의 흐름일 뿐. "메시지" 단위가 아니다 |
| 이름이 없다 | 파일 시스템에 나타나지 않는다 → **친척끼리만** 쓸 수 있다 |
| 버퍼가 유한하다 | 가득 차면 쓰는 쪽이 **막힌다** |

> **"경계가 없다"** 는 성질이 11강 프로토콜 설계의 출발점입니다. 세 번 나누어 써도 받는 쪽은 한 번에 받을 수 있고, 그 반대도 마찬가지입니다. 2강 4.1절의 **부분 처리**가 파이프에서 늘 일어납니다.
{: .prompt-warning }

### 1.3 왜 친척끼리만 되는가

파이프는 이름이 없으므로 **서술자를 물려받는 것 말고는 접근할 방법이 없습니다.** 그래서 `fork` 전에 만들어야 하고, 부모–자식이나 형제 사이에서만 쓸 수 있습니다.

관계없는 프로세스끼리 쓰려면 **이름이 필요합니다.** 그것이 제9절의 FIFO입니다.

---

## 제2절. `pipe`로 관 놓기

### 2.1 순서

| 순서 | 하는 일 |
|---|---|
| 1 | **`pipe`로 만든다** — 반드시 `fork`보다 먼저 |
| 2 | `fork` — 자식이 서술자 두 개를 물려받는다 |
| 3 | **각자 안 쓰는 쪽을 닫는다** |
| 4 | 한쪽은 쓰고 한쪽은 읽는다 |
| 5 | 다 쓰면 닫는다 |

### 2.2 예제

```c
    if (pipe(fd) == -1) {
        fprintf(stderr, "pipe: %s\n", strerror(errno));
        return 1;
    }
    printf("파이프 서술자: 읽기 %d, 쓰기 %d\n", fd[0], fd[1]);

    fflush(stdout);                 /* fork 전에 버퍼를 비운다 (4강 4절) */
    pid = fork();
```

```c
    if (pid == 0) {                 /* 자식: 읽는 쪽만 쓴다 */
        close(fd[1]);               /* 쓰는 쪽을 반드시 닫는다 */

        while ((n = read(fd[0], buf, sizeof buf - 1)) > 0) {
            buf[n] = '\0';
            printf("[자식] 받음: %s", buf);
            fflush(stdout);
        }
        if (n == 0)
            printf("[자식] 파일 끝 — 쓰는 쪽이 모두 닫혔습니다\n");
        close(fd[0]);
        _exit(0);
    }

    /* 부모: 쓰는 쪽만 쓴다 */
    close(fd[0]);                   /* 읽는 쪽을 반드시 닫는다 */

    msg = "첫 번째 소식\n";
    write(fd[1], msg, strlen(msg));
    sleep(1);
    msg = "두 번째 소식\n";
    write(fd[1], msg, strlen(msg));

    close(fd[1]);                   /* 닫아야 자식이 EOF 를 본다 */
    wait(NULL);
```

```bash
gcc -Wall -Wextra -std=gnu17 pipe1.c -o pipe1
```

```bash
./pipe1
```

```text
파이프 서술자: 읽기 3, 쓰기 4
[자식] 받음: 첫 번째 소식
[자식] 받음: 두 번째 소식
[부모] 끝
```

**서술자 번호가 3과 4**인 것에 주목하십시오. 0·1·2는 이미 쓰이고 있으므로 **가장 작은 빈 번호**부터 배정됩니다(2강 9.1절).

### 2.3 두 가지 습관

**① `fork` 전에 `fflush`**

4강 4절의 버퍼 복제 함정 그대로입니다. 이것을 빠뜨리면 첫 줄이 **두 번** 나옵니다. 직접 지워 보고 확인하십시오.

**② 길이는 `strlen`으로**

```c
write(fd[1], "첫 번째 소식\n", 19);      /* 손으로 센 값 — 틀리기 쉽다 */
```

위 문자열은 실제로 **18바이트**입니다(한글 5자 × 3 + 공백 2 + 개행 1). 19를 넘기면 문자열 끝의 `'\0'`까지 보내게 됩니다. **1부 10강의 "크기를 아는 자가 안전하다"** 는 여기서도 그대로입니다.

---

## 제3절. 반드시 지켜야 할 규칙

**이 절이 이번 강의에서 가장 중요합니다.**

### 3.1 안 쓰는 쪽을 닫으십시오

> **파이프의 읽기는 "쓰는 쪽이 **모두** 닫혔을 때" 비로소 EOF(0)를 돌려줍니다.**

`fork` 하면 쓰는 쪽 서술자가 **부모와 자식 양쪽에** 생깁니다. 부모가 닫아도 **자식이 들고 있으면** 아직 "모두 닫힘"이 아닙니다.

### 3.2 실험

`pipe_close.c`는 자식이 쓰는 쪽을 닫는 경우와 닫지 않는 경우를 비교합니다. 영원히 기다리지 않도록 5초 알람을 걸어 두었습니다(5강 10절).

```c
static void on_alarm(int signo)
{
    /* 길이를 손으로 세지 말 것 — sizeof 로 얻는다 (1부 10강) */
    static const char msg[] = "  >> 5초가 지나도 EOF 가 오지 않습니다\n";

    (void) signo;
    write(STDERR_FILENO, msg, sizeof msg - 1);
    _exit(2);
}
```

```c
        if (good)
            close(fd[1]);           /* 쓰는 쪽을 닫는다 */
        /* bad 인 경우 일부러 닫지 않는다 */

        while ((n = read(fd[0], buf, sizeof buf - 1)) > 0) {
            buf[n] = '\0';
            printf("  자식이 받음: %s", buf);
            fflush(stdout);
        }
        if (n == 0)
            printf("  자식: EOF 를 받았습니다. 정상 종료합니다\n");
        fflush(stdout);             /* _exit 은 버퍼를 비우지 않는다 (4강 10절) */
        _exit(0);
```

```bash
./pipe_close good
```

```text
good: 부모가 쓰는 쪽을 닫았습니다
  자식이 받음: 한 줄
  자식: EOF 를 받았습니다. 정상 종료합니다
```

```bash
./pipe_close bad
```

```text
bad: 부모가 쓰는 쪽을 닫았습니다
  자식이 받음: 한 줄
  >> 5초가 지나도 EOF 가 오지 않습니다
bad: 자식이 EOF 를 받지 못했습니다
```

**같은 프로그램이 `close` 한 줄 때문에 영원히 멈춥니다.**

> **파이프를 쓰는 프로그램이 멈춰 있다면, 열에 아홉은 서술자를 안 닫은 것입니다.**
> `ls -l /proc/PID/fd`로 확인하십시오(2강 9.2절). 파이프가 남아 있으면 `pipe:[12345]` 형태로 보입니다.
{: .prompt-danger }

### 3.3 네 가지 경우

| 상황 | `read`의 결과 | `write`의 결과 |
|---|---|---|
| 자료가 있다 | 읽는다 | — |
| 비었고 **쓰는 쪽이 열려 있다** | **막힌다** | — |
| 비었고 **쓰는 쪽이 모두 닫혔다** | **0 (EOF)** | — |
| **읽는 쪽이 모두 닫혔다** | — | **`SIGPIPE`** → 죽는다 |
| 가득 찼고 읽는 쪽이 열려 있다 | — | **막힌다** |

마지막에서 둘째 줄이 5강 문제 10에서 본 것입니다. 서버에서는 `SIGPIPE`를 무시하고 `EPIPE`로 판단합니다.

### 3.4 정리

> **각자 안 쓰는 쪽은 즉시 닫는다. 다 쓴 쪽도 즉시 닫는다.**

닫기를 빠뜨리면 세 가지가 일어납니다.

| 결과 | |
|---|---|
| EOF가 오지 않아 **멈춘다** | 3.2절 |
| 서술자가 새어 나간다 | 2강 5.2절 `EMFILE` |
| `SIGPIPE`가 와야 할 때 오지 않는다 | |

---

## 제4절. 파이프의 용량과 원자성

### 4.1 얼마나 담기는가

```c
#define _GNU_SOURCE          /* F_GETPIPE_SZ 는 GNU 확장이다 */
```

```c
    /* 막히지 않게 해 두고 가득 찰 때까지 써 본다 */
    if (fcntl(fd[1], F_SETFL, O_NONBLOCK) == -1) { ... }

    for (;;) {
        n = write(fd[1], chunk, sizeof chunk);
        if (n == -1) {
            if (errno == EAGAIN)
                break;              /* 가득 찼다 */
            ...
        }
        total += n;
    }
```

```bash
./pipe_size
```

```text
파이프에 들어간 바이트 수 : 65536 (64 KiB)
F_GETPIPE_SZ 가 알려 준 값 : 65536
PIPE_BUF (원자적 쓰기 한계): 4096
```

**직접 채워 본 값과 커널이 알려 준 값이 정확히 같습니다.**

### 4.2 `_GNU_SOURCE`가 또 나왔습니다

1강 8.2절에서 `-std=c17`이 POSIX 이름을 감춘다고 배웠고, 그래서 `-std=gnu17`을 쓰기로 했습니다. 그런데 **`F_GETPIPE_SZ`는 `-std=gnu17`로도 보이지 않습니다.**

```bash
gcc -Wall -Wextra -std=gnu17 pipe_size.c -o pipe_size     # _GNU_SOURCE 없이
```

```text
pipe_size.c: In function 'main':
pipe_size.c:41:62: error: 'F_GETPIPE_SZ' undeclared (first use in this function)
```

| 무엇이 필요한가 | 어떻게 |
|---|---|
| POSIX 이름(`PATH_MAX`·`strdup`) | **`-std=gnu17`** 로 충분 |
| **리눅스 고유 확장**(`F_GETPIPE_SZ` 등) | **`#define _GNU_SOURCE`** 가 더 필요 |

**반드시 모든 `#include`보다 앞**에 두어야 합니다([`feature_test_macros(7)`](https://man7.org/linux/man-pages/man7/feature_test_macros.7.html)).

### 4.3 원자적 쓰기 — `PIPE_BUF`

여러 프로세스가 **같은 파이프에 동시에** 쓸 때가 있습니다(로그 수집기 등). 그때 글이 섞이지 않으려면 보장이 필요합니다.

> **`PIPE_BUF`(리눅스에서 4096) 이하의 `write`는 나뉘지 않습니다.**
> 그보다 크면 여러 조각으로 나뉠 수 있고, 그 사이에 다른 프로세스의 자료가 끼어들 수 있습니다.

| 쓰는 크기 | 보장 |
|---|---|
| `PIPE_BUF` 이하 | **통째로 들어간다** — 섞이지 않는다 |
| `PIPE_BUF` 초과 | 나뉠 수 있다 — **섞일 수 있다** |

3강 5.2절에서 본 `st_blocks`의 512처럼, **이런 상수는 반드시 확인하고 쓰는 습관**을 들이십시오.

---

> **▶ 여기서부터 2회차 — 재지정과 파이프 — 미니 셸 완성**
> 제5절 ~ 제7절, 약 140분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제5절. `dup2` — 재지정의 원리

### 5.1 서술자 복제

```c
#include <unistd.h>

int dup(int oldfd);                 /* 가장 작은 빈 번호로 복제 */
int dup2(int oldfd, int newfd);     /* newfd 로 복제 — 열려 있으면 먼저 닫는다 */
```

**`dup2`가 핵심입니다.** "1번을 이 파일로 만들어라"를 한 번에 해 줍니다.

```text
  dup2(fd, 1) 하기 전            dup2(fd, 1) 한 뒤
  ┌────┬──────────────┐          ┌────┬──────────────┐
  │ 1  │──▶ 터미널     │          │ 1  │──┐           │
  │ 3  │──▶ out.txt   │          │ 3  │──┼──▶ out.txt │
  └────┴──────────────┘          └────┴──┘           │
                                  (터미널로 가던 연결은 끊긴다)
```

### 5.2 확인

```c
    saved = dup(STDOUT_FILENO);     /* 원래 표준 출력을 따로 보관해 둔다 */

    if (dup2(fd, STDOUT_FILENO) == -1) {   /* 1번을 파일로 바꾼다 */
        fprintf(stderr, "dup2: %s\n", strerror(errno));
        return 1;
    }
    close(fd);                      /* 원본 서술자는 이제 필요 없다 */

    printf("② 이 줄은 파일로 갑니다\n");   /* 코드는 그대로인데 목적지가 바뀌었다 */
    fflush(stdout);

    if (dup2(saved, STDOUT_FILENO) == -1) { ... }  /* 원래대로 되돌린다 */
    close(saved);
```

```bash
./dup2_demo
```

```text
① 이 줄은 화면으로 나갑니다
열린 파일의 서술자 = 3
③ 다시 화면으로 나갑니다
--- out.txt 의 내용 ---
② 이 줄은 파일로 갑니다
```

**`printf`는 세 번 모두 똑같습니다.** 바뀐 것은 **서술자 1이 어디를 가리키는가**뿐입니다.

1강 8.5절에서 "재지정이 왜 되는지"를 예고했던 답이 이것입니다. 셸은 프로그램을 시작하기 **전에** 자식에서 `dup2`를 해 둡니다.

### 5.3 세 가지 주의

| 주의 | 설명 |
|---|---|
| `dup2` **뒤에 원본을 닫는다** | 안 닫으면 서술자가 샌다 |
| 되돌리려면 **미리 `dup`으로 보관** | 한 번 덮어쓰면 원래 연결은 사라진다 |
| `fflush` **먼저** | 버퍼에 남은 내용이 엉뚱한 곳으로 간다 |

> **옛 방식도 알아 두십시오.**
> ```c
> close(STDOUT_FILENO);        /* 1번을 비운다 */
> fd = open("out.txt", ...);   /* 가장 작은 빈 번호 = 1 을 받는다 */
> ```
> 2강 9.1절의 규칙을 이용한 것입니다. 동작하지만 **그 사이에 시그널 처리기가 파일을 열면 깨집니다.** `dup2`는 이 두 단계를 하나로 묶어 주므로 언제나 `dup2`를 쓰십시오.
{: .prompt-tip }

---

## 제6절. 미니 셸에 재지정 넣기

### 6.1 해석하기

명령줄에서 `<`·`>`·`>>`를 떼어 내고, 나머지를 인자로 삼습니다.

```c
struct command {
    char *argv[MAXARGS];
    int   argc;
    char *infile;               /* <  */
    char *outfile;              /* > 또는 >> */
    int   append;               /* >> 이면 1 */
};
```

```c
    while ((tok = next_token(&pos)) != NULL) {
        if (strcmp(tok, "<") == 0) {
            c->infile = next_token(&pos);
            if (c->infile == NULL) {
                fprintf(stderr, "구문 오류: < 뒤에 파일 이름이 없습니다\n");
                return -1;
            }
        } else if (strcmp(tok, ">") == 0 || strcmp(tok, ">>") == 0) {
            c->append = (tok[1] == '>');
            c->outfile = next_token(&pos);
            ...
        } else {
            c->argv[c->argc++] = tok;
        }
    }
    c->argv[c->argc] = NULL;
```

**재지정 기호와 파일 이름은 `argv`에 들어가지 않습니다.** `ls > out.txt`를 실행하면 `ls`가 받는 인자는 `ls` 하나뿐입니다. 파일로 보내는 일은 **셸이 하고 `ls`는 모릅니다.**

### 6.2 적용하기

```c
/* apply_redirect: 자식 안에서 재지정을 실제로 적용한다 */
static int apply_redirect(const struct command *c)
{
    int fd;

    if (c->infile != NULL) {
        fd = open(c->infile, O_RDONLY);
        if (fd == -1) {
            fprintf(stderr, "%s: %s\n", c->infile, strerror(errno));
            return -1;
        }
        if (dup2(fd, STDIN_FILENO) == -1) { ... }
        close(fd);
    }

    if (c->outfile != NULL) {
        int flags = O_WRONLY | O_CREAT | (c->append ? O_APPEND : O_TRUNC);

        fd = open(c->outfile, flags, 0644);
        if (fd == -1) { ... }
        if (dup2(fd, STDOUT_FILENO) == -1) { ... }
        close(fd);
    }
    return 0;
}
```

**`>`와 `>>`의 차이가 `O_TRUNC`와 `O_APPEND` 한 글자**입니다. 2강 2.4절에서 `fopen`의 `"w"`와 `"a"`를 `open` 플래그로 풀어 본 것이 그대로 쓰입니다.

**반드시 자식 안에서, `exec` 직전에** 해야 합니다. 부모(셸)에서 하면 셸 자신의 출력이 파일로 가 버립니다.

### 6.3 동작 확인

```bash
./minishell
```

```text
minish$ echo 안녕하세요 > o1.txt
minish$ cat o1.txt
안녕하세요
minish$ echo 둘째줄 >> o1.txt
minish$ cat o1.txt
안녕하세요
둘째줄
minish$ wc -l < o1.txt
2
```

**`>`·`>>`·`<`가 모두 동작합니다.**

---

## 제7절. 미니 셸에 파이프 넣기

### 7.1 무엇을 해야 하는가

`ls | sort | head -3`을 실행하려면 다음이 필요합니다.

| 필요한 것 | 개수 |
|---|---|
| 프로세스 | **3개** |
| 파이프 | **2개** |
| `dup2` | 각 자식마다 최대 2번 |

```text
  [ls] ──파이프1──▶ [sort] ──파이프2──▶ [head]
   1번을 파이프1로      0번을 파이프1로       0번을 파이프2로
                       1번을 파이프2로        1번은 그대로(화면)
```

### 7.2 구현

핵심은 **앞 명령의 읽기 쪽을 다음 반복으로 넘기는 것**입니다.

```c
    int prev_read = -1;             /* 앞 명령의 출력을 담은 읽기 쪽 */

    for (i = 0; i < ncmd; i++) {
        int fd[2] = { -1, -1 };

        if (i < ncmd - 1 && pipe(fd) == -1) { ... }

        pids[i] = fork();
        ...
        if (pids[i] == 0) {                     /* 자식 */
            sigaction(SIGINT, &dfl, NULL);      /* 5강 문제 9 */
            sigaction(SIGQUIT, &dfl, NULL);

            if (prev_read != -1) {              /* 앞 명령에서 받아 온다 */
                if (dup2(prev_read, STDIN_FILENO) == -1)
                    _exit(1);
                close(prev_read);
            }
            if (i < ncmd - 1) {                 /* 뒤 명령으로 보낸다 */
                close(fd[0]);                   /* 자식은 읽는 쪽을 안 쓴다 */
                if (dup2(fd[1], STDOUT_FILENO) == -1)
                    _exit(1);
                close(fd[1]);
            }

            /* 파일 재지정은 파이프보다 우선한다 — 셸의 관례 */
            if (apply_redirect(&cmds[i]) == -1)
                _exit(1);

            execvp(cmds[i].argv[0], cmds[i].argv);
            fprintf(stderr, "%s: %s\n", cmds[i].argv[0], strerror(errno));
            _exit(127);
        }

        /* 부모: 자기가 쓰지 않는 쪽을 반드시 닫는다 */
        if (prev_read != -1)
            close(prev_read);
        if (i < ncmd - 1) {
            close(fd[1]);
            prev_read = fd[0];
        }
    }
```

### 7.3 닫기가 전부입니다

**부모의 세 줄이 이 절의 핵심입니다.**

| 부모가 하는 일 | 안 하면 |
|---|---|
| `close(prev_read)` | 서술자가 쌓인다 |
| `close(fd[1])` | **뒤 명령이 EOF를 못 받아 영원히 멈춘다** |
| `prev_read = fd[0]` | 다음 명령이 받을 곳을 기억 |

**부모가 쓰는 쪽을 들고 있으면** 3.1절의 그 문제가 그대로 일어납니다. `ls | wc -l`에서 `wc`가 영원히 기다립니다.

### 7.4 확인 — 진짜 셸과 맞춰 보기

```text
minish$ ls /etc | head -3
PackageKit
X11
adduser.conf
minish$ ls /etc | wc -l
175
```

```bash
ls /etc | wc -l
```

```text
175
```

**같은 값입니다.** 3단 파이프에 재지정까지 섞어도 됩니다.

```text
minish$ ls /etc | sort | head -3 > o2.txt
minish$ cat o2.txt
PackageKit
X11
adduser.conf
```

```bash
ls /etc | sort | head -3
```

```text
PackageKit
X11
adduser.conf
```

### 7.5 오류도 처리합니다

```text
minish$ cat < 없는파일
없는파일: No such file or directory
[종료 상태 1]
minish$ 없는명령 | wc -l
없는명령: No such file or directory
0
minish$ echo a >
구문 오류: > 뒤에 파일 이름이 없습니다
```

- 파이프 중간의 명령이 실패해도 **나머지는 정상적으로 끝납니다.** `wc -l`이 0을 내놓은 것이 그 증거입니다.
- 진짜 셸도 똑같이 동작합니다.

### 7.6 내장 명령은 파이프가 없을 때만

```c
        if (nseg == 1) {                    /* 내장 명령은 파이프가 없을 때만 */
            if (strcmp(cmds[0].argv[0], "exit") == 0)
                break;
            if (strcmp(cmds[0].argv[0], "cd") == 0) { ... }
        }
```

`cd`는 셸 자신이 해야 하는데(4강 6.3절), **파이프 안에서는 자식이 됩니다.** 그래서 파이프가 있으면 내장으로 처리하지 않습니다. 진짜 셸도 `cd /tmp | cat`은 아무 효과가 없습니다.

---

> **▶ 여기서부터 3회차 — `popen`·FIFO·`mmap`**
> 제8절 ~ 제11절, 약 110분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제8절. `popen` — 간편한 방법

### 8.1 세 단계를 한 줄로

```c
#include <stdio.h>

FILE *popen(const char *command, const char *type);
int   pclose(FILE *stream);
```

`pipe` + `fork` + `exec` + `dup2`를 한 번에 해 주고, **`FILE *`로 돌려줍니다.**

| `type` | 뜻 |
|---|---|
| `"r"` | 명령의 **출력을 읽는다** |
| `"w"` | 명령에 **자료를 보낸다** |

```c
    fp = popen("ls -1 /etc | head -5", "r");
    ...
    while (fgets(line, sizeof line, fp) != NULL) {
        printf("  %d: %s", ++n, line);
    }
    status = pclose(fp);
```

```bash
./popen_demo
```

```text
--- /etc 의 앞 5개 ---
  1: PackageKit
  2: X11
  3: adduser.conf
  4: alternatives
  5: apache2
pclose 결과: 0
--- sort -n 에 넘긴 결과 ---
10
20
30
```

**`FILE *`이므로 `fgets`·`fprintf`를 쓸 수 있습니다**(2강 1절). 편리한 대신 대가가 있습니다.

### 8.2 대가

| 문제 | 설명 |
|---|---|
| **셸을 거친다** | `/bin/sh -c "명령"`으로 실행된다 |
| **명령 주입 위험** | 사용자 입력을 그대로 넣으면 **치명적** |
| 한 방향만 | 읽기와 쓰기를 동시에 못 한다 |
| 오류 구분이 어렵다 | `pclose`의 반환값을 `WEXITSTATUS`로 풀어야 한다 |

> **사용자 입력을 `popen`에 넣지 마십시오.**
> ```c
> snprintf(cmd, sizeof cmd, "ls %s", user_input);   /* 위험 */
> popen(cmd, "r");
> ```
> 사용자가 `; rm -rf ~`를 넣으면 그대로 실행됩니다. 셸이 해석하기 때문입니다. **1부 10강의 신뢰 경계**가 여기서도 똑같이 적용됩니다. 안전하게 하려면 `fork`+`execvp`로 직접 만드십시오. 인자를 배열로 넘기면 셸을 거치지 않습니다.
{: .prompt-danger }

실제로 확인해 보십시오. 위 코드에 `/etc; echo 침입성공`을 넘긴 결과입니다.

```text
xml
zsh_command_not_found
침입성공
```

**`침입성공`이 출력되었습니다.** `ls` 목록 뒤에 붙은 저 한 줄이, 공격자가 원하는 아무 명령이나 실행할 수 있다는 증거입니다. 문제 10에서 이것을 막는 방법을 만듭니다.

`pclose`의 반환값은 4강 7.2절의 종료 상태와 같은 형식입니다.

```c
int status = pclose(fp);
if (WIFEXITED(status))
    printf("종료 상태 %d\n", WEXITSTATUS(status));
```

---

## 제9절. FIFO — 이름 있는 파이프

### 9.1 이름이 있으면 남남끼리도 됩니다

```c
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);
```

FIFO는 **파일 시스템에 이름이 있는 파이프**입니다. 3강 3.1절에서 본 `S_ISFIFO`가 이것입니다.

```bash
mkfifo -m 600 /tmp/mypipe
ls -l /tmp/mypipe
```

```text
prw------- 1 student student 0 Dec 28 09:20 /tmp/mypipe
```

**맨 앞 글자가 `p`** 입니다.

### 9.2 써 보기

터미널 두 개를 여십시오.

```bash
cat > /tmp/mypipe
```

```bash
cat < /tmp/mypipe
```

한쪽에 친 글이 다른 쪽에 나타납니다. **두 `cat`은 아무 친척 관계가 아닙니다.**

### 9.3 파이프와 다른 점

| | 파이프 | FIFO |
|---|---|---|
| 이름 | 없다 | **있다** |
| 쓸 수 있는 대상 | 친척끼리만 | **누구나** |
| 만들기 | `pipe()` | `mkfifo()` |
| 열기 | 만들면 바로 | **`open`으로 연다** |
| 없앨 때 | 닫으면 사라진다 | **`unlink` 해야 한다** |

> **`open`이 막힙니다.**
> FIFO를 읽기 전용으로 열면 **쓰는 쪽이 열릴 때까지 `open`이 막힙니다.** 그 반대도 마찬가지입니다. 막히지 않게 하려면 `O_NONBLOCK`을 씁니다. 처음 FIFO를 쓸 때 가장 당황하는 부분입니다.
{: .prompt-warning }

자료가 흐르는 방식(경계 없음, `PIPE_BUF` 원자성, EOF 규칙)은 파이프와 **완전히 같습니다.**

### 9.4 한계

FIFO도 **한 컴퓨터 안에서만** 씁니다. 다른 컴퓨터와 통신하려면 **소켓**이 필요합니다. 9강에서 배웁니다.

---

## 제10절. `mmap` — 메모리 공유

### 10.1 파이프와 다른 접근

파이프는 **자료를 복사해서** 주고받습니다. 큰 자료라면 낭비입니다. **아예 같은 메모리를 함께 보면** 어떨까요?

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int   munmap(void *addr, size_t length);
```

| `prot` | `flags` |
|---|---|
| `PROT_READ` / `PROT_WRITE` | **`MAP_SHARED`** — 바뀐 내용이 공유된다 |
| | `MAP_PRIVATE` — 쓰면 자기만의 복사본이 생긴다 |
| | `MAP_ANONYMOUS` — 파일 없이 메모리만 |

### 10.2 4강의 반례

4강 3.2절에서 **"`fork` 하면 메모리는 복사된다"** 고 배웠습니다. `mmap`에 `MAP_SHARED`를 주면 **그렇지 않습니다.**

```c
    int flags = MAP_ANONYMOUS | (shared ? MAP_SHARED : MAP_PRIVATE);

    counter = mmap(NULL, sizeof *counter, PROT_READ | PROT_WRITE, flags, -1, 0);
    if (counter == MAP_FAILED) { ... }
    *counter = 0;

    pid = fork();
    if (pid == 0) {
        for (i = 0; i < 1000; i++)
            (*counter)++;
        _exit(0);
    }

    wait(NULL);
    printf("MAP_%s : 자식이 1000번 더한 뒤 부모가 본 값 = %ld\n",
           shared ? "SHARED " : "PRIVATE", *counter);
```

```bash
./mmap_share shared ; ./mmap_share private
```

```text
MAP_SHARED  : 자식이 1000번 더한 뒤 부모가 본 값 = 1000
MAP_PRIVATE : 자식이 1000번 더한 뒤 부모가 본 값 = 0
```

**플래그 하나로 결과가 1000과 0으로 갈립니다.**

| 플래그 | 4강에서 배운 것과 |
|---|---|
| `MAP_PRIVATE` | 같다 — 쓸 때 복사(4강 3.4절) |
| **`MAP_SHARED`** | **다르다** — 진짜로 같은 메모리 |

### 10.3 빠른 대신 위험합니다

| 방식 | 속도 | 동기화 |
|---|---|---|
| 파이프 | 복사가 있어 느리다 | **필요 없다** — 커널이 정리해 준다 |
| **공유 메모리** | **가장 빠르다** | **직접 해야 한다** |

> **위 예제는 자식만 값을 바꾸었기 때문에 안전했습니다.**
> 부모와 자식이 **동시에** `(*counter)++`를 하면 값이 어긋납니다. `++`는 "읽고, 더하고, 쓴다" 세 단계이며 그 사이에 상대가 끼어들 수 있기 때문입니다. 이것이 **경쟁 상태**이며, 7강 스레드에서 본격적으로 다룹니다.
{: .prompt-danger }

동기화 수단으로는 세마포어(`sem_init`)나 뮤텍스를 씁니다. 7강에서 배웁니다.

### 10.4 파일을 메모리처럼 다루기

`MAP_ANONYMOUS` 대신 파일 서술자를 주면, **파일을 배열처럼** 쓸 수 있습니다.

```c
fd = open("data.bin", O_RDWR);
p = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
p[100] = 'X';           /* read/write 없이 파일을 고친다 */
```

| 장점 | 단점 |
|---|---|
| `read`·`write` 호출이 없다 | 크기가 바뀌는 파일에 쓰기 어렵다 |
| 큰 파일의 일부만 다루기 좋다 | 오류가 시그널(`SIGBUS`)로 온다 |
| 여러 프로세스가 공유 | 이식성 문제 |

데이터베이스가 널리 쓰는 방식입니다.

---

## 제11절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| **읽기가 영원히 멈춘다** | 쓰는 쪽을 안 닫음 | **모든 쪽에서 `close`**(3.1절) |
| 파이프라인 끝이 안 끝난다 | 부모가 `fd[1]`을 들고 있음 | 7.3절 |
| 쓰다가 프로그램이 죽는다 | `SIGPIPE` | `SIG_IGN` 후 `EPIPE` 확인 |
| 출력이 두 번 나온다 | `fork` 전 버퍼 | **`fflush` 먼저**(2.3절) |
| `pipe`를 `fork` 뒤에 만듦 | 서술자가 안 물려짐 | **`fork`보다 먼저** |
| 자료가 섞인다 | `PIPE_BUF` 초과 | 4096 이하로 나누어 쓰기 |
| `F_GETPIPE_SZ` 없음 | GNU 확장 | **`#define _GNU_SOURCE`**(4.2절) |
| 재지정했는데 셸 출력이 파일로 | 부모에서 `dup2` | **자식에서, `exec` 직전** |
| `dup2` 후 서술자가 샌다 | 원본을 안 닫음 | `dup2` 뒤 `close(fd)` |
| FIFO의 `open`이 멈춘다 | 반대쪽이 아직 안 열림 | 정상. `O_NONBLOCK` |
| `popen`에 사용자 입력 | **명령 주입** | `fork`+`execvp` |
| 공유 메모리 값이 이상하다 | 동기화 없음 | 7강 |
| `mmap` 결과 검사 | `NULL`이 아니라 **`MAP_FAILED`** | 비교 대상을 바르게 |
| `cd`가 파이프에서 안 먹는다 | 자식이 됨 | **정상**(7.6절) |

---

> **▶ 여기서부터 4회차 — 스스로 해 보기**
> 실습문제, 약 110분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 실습문제

> **안내**
> 1. 컴파일은 **`gcc -Wall -Wextra -std=gnu17`**, **경고 0개**여야 합니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다.
> 3. 셸을 만드는 문제는 **진짜 셸의 결과와 비교**해 검증하십시오.
{: .prompt-info }

| 문제 | 주제 | 대응 절 |
|---|---|---|
| 문제 1 | 부모–자식 파이프 | 2 |
| 문제 2 | **닫기를 빠뜨리면** | 3 |
| 문제 3 | 파이프 용량 재기 | 4 |
| 문제 4 | `dup2`로 재지정 | 5 |
| 문제 5 | 미니 셸에 `>`·`<`·`>>` | 6 |
| 문제 6 | **미니 셸에 `\|`** | 7 |
| 문제 7 | 양방향 통신 | 2 · 3 |
| 문제 8 | FIFO로 남남끼리 | 9 |
| 문제 9 | `mmap` 공유 메모리 | 10 |
| 문제 10 | `popen`을 안전하게 바꾸기 | 8 |

---

### 문제 1. 부모–자식 파이프

부모가 보낸 글을 자식이 받아 출력하는 프로그램을 만드십시오.

**정답 및 해설**

2.2절의 코드와 결과가 답입니다.

- **`pipe`를 `fork`보다 먼저** 불러야 합니다. 뒤에 부르면 각자 다른 파이프가 생겨 아무 관계가 없습니다.
- 서술자 번호가 **3과 4**로 나오는지 확인하십시오(2강 9.1절).
- `fork` 전 `fflush`를 빼면 첫 줄이 두 번 나옵니다(4강 4절). 직접 확인해 보십시오.
- 길이는 `strlen`으로 구하십시오. 한글이 섞이면 손으로 세다 반드시 틀립니다(2.3절).

---

### 문제 2. 닫기를 빠뜨리면

쓰는 쪽을 닫지 않았을 때 읽기가 어떻게 되는지 확인하십시오.

**정답 및 해설**

3.2절의 결과가 답입니다.

| 경우 | 결과 |
|---|---|
| 자식이 `close(fd[1])` | EOF를 받고 정상 종료 |
| 닫지 않음 | **영원히 기다린다**(알람이 5초 뒤 끊어 줌) |

- **안전장치로 `alarm`을 건 것**이 좋은 시험 설계입니다(5강 10절). 그러지 않으면 시험 자체가 멈춥니다.
- 처리기에서 `write`와 `_exit`만 쓴 것을 확인하십시오(5강 5.2절). `printf`는 쓸 수 없습니다.
- 멈춘 상태에서 다른 터미널로 확인해 보십시오.

```bash
ls -l /proc/$(pgrep pipe_close)/fd
```

- 파이프가 `pipe:[12345]` 형태로 보입니다. 같은 번호를 여러 프로세스가 들고 있으면 그것이 원인입니다.

---

### 문제 3. 파이프 용량 재기

파이프에 얼마나 담기는지 직접 재고, 커널이 알려 주는 값과 비교하십시오.

**정답 및 해설**

4.1절의 결과가 답입니다.

```text
파이프에 들어간 바이트 수 : 65536 (64 KiB)
F_GETPIPE_SZ 가 알려 준 값 : 65536
PIPE_BUF (원자적 쓰기 한계): 4096
```

- **`O_NONBLOCK`을 걸지 않으면 가득 찬 순간 영원히 멈춥니다.** 재는 방법 자체가 이 문제의 핵심입니다.
- `EAGAIN`으로 "지금은 안 된다"를 판단했습니다(1강 10.1절의 그 `errno`). 9강 논블로킹 서버에서 다시 만납니다.
- **`_GNU_SOURCE`가 필요한 이유**를 설명할 수 있어야 합니다(4.2절).
- `fcntl(fd, F_SETPIPE_SZ, 크기)`로 용량을 바꿀 수도 있습니다. 시도해 보십시오.

---

### 문제 4. `dup2`로 재지정

표준 출력을 파일로 바꾸었다가 되돌리는 프로그램을 만드십시오.

**정답 및 해설**

5.2절의 코드와 결과가 답입니다.

- **`dup`으로 원본을 보관**해야 되돌릴 수 있습니다. 이것을 빠뜨리면 화면으로 돌아갈 방법이 없습니다.
- `dup2` 전에 `fflush`를 하지 않으면, 버퍼에 남아 있던 내용이 **새 목적지로** 나갑니다. 직접 확인해 보십시오.
- `close(fd)`를 빠뜨리면 서술자가 하나 샙니다. `/proc/self/fd`로 확인할 수 있습니다.

---

### 문제 5. 미니 셸에 재지정

`>`·`<`·`>>`를 지원하도록 미니 셸을 고치십시오.

**정답 및 해설**

6.1·6.2절의 코드가 답이며, 확인은 6.3절과 같이 합니다.

| 확인 | 기대 |
|---|---|
| `echo 안녕 > o.txt` | 파일 생성 |
| `echo 둘째 >> o.txt` | 이어 붙음 |
| `wc -l < o.txt` | 2 |
| `cat < 없는파일` | 오류 메시지, 종료 상태 1 |

- **`dup2`를 자식에서, `exec` 직전에** 해야 합니다. 부모에서 하면 셸의 프롬프트까지 파일로 갑니다.
- `open` 실패를 처리했는지 확인하십시오. `< 없는파일`에서 프로그램이 죽으면 안 됩니다.
- 재지정 기호가 `argv`에 들어가지 않는지 확인하십시오. `echo a > f`에서 `echo`가 받는 인자는 `a` 하나입니다.

---

### 문제 6. 미니 셸에 파이프

`|`로 명령 여러 개를 잇도록 고치고, **진짜 셸과 결과를 비교**하십시오.

**정답 및 해설**

7.2절의 코드가 답입니다. 검증은 반드시 비교로 하십시오.

```bash
printf 'ls /etc | wc -l\nexit\n' | ./minishell
```

```bash
ls /etc | wc -l
```

두 값이 같아야 합니다(실행 환경에서는 175였습니다).

- **부모의 `close`가 전부입니다**(7.3절). 하나라도 빠뜨리면 파이프라인이 멈춥니다.
- 자식에서 `close(fd[0])`을 빠뜨리면 그 자식이 자기 출력을 읽을 수 있게 되어, 뒤 명령이 EOF를 받지 못합니다.
- **모든 자식을 `waitpid`로 거두어야** 합니다. 마지막 것만 기다리면 나머지가 좀비가 됩니다(4강 8절).
- 3단 이상도 시험하십시오. `ls | sort | head -3`이 진짜 셸과 같아야 합니다.

---

### 문제 7. 양방향 통신

부모가 자식에게 숫자를 보내면 자식이 제곱해 돌려주는 프로그램을 만드십시오.

**정답 및 해설**

**파이프는 단방향이므로 두 개가 필요합니다.**

```c
    int to_child[2], to_parent[2];

    if (pipe(to_child) == -1 || pipe(to_parent) == -1) { ... }

    pid = fork();
    if (pid == 0) {
        close(to_child[1]);         /* 자식은 to_child 에서 읽기만 */
        close(to_parent[0]);        /* to_parent 에는 쓰기만 */
        ...
        _exit(0);
    }

    close(to_child[0]);             /* 부모는 반대로 */
    close(to_parent[1]);
```

```bash
./twoway
```

```text
2 의 제곱 = 4
3 의 제곱 = 9
4 의 제곱 = 16
5 의 제곱 = 25
```

- **닫아야 할 서술자가 네 개**입니다. 파이프가 늘어날수록 닫기가 복잡해지는 것이 이 방식의 부담입니다.
- **교착 상태에 빠지기 쉽습니다.** 둘 다 상대가 보내기를 기다리면 영원히 멈춥니다. 순서를 명확히 정하십시오.
- 파이프가 가득 차서 생기는 교착도 있습니다. 부모가 64KiB를 다 보내려 하는데 자식이 아직 안 읽으면, 부모는 막히고 자식은 부모의 응답을 기다립니다.
- 실무에서는 이런 경우 **소켓쌍**(`socketpair`)을 씁니다. 하나로 양방향이 됩니다.

---

### 문제 8. FIFO로 남남끼리

FIFO를 만들고, **서로 관계없는 두 프로그램**이 통신하게 하십시오.

**정답 및 해설**

```c
/* fifo_writer.c */
    if (mkfifo(PATH, 0600) == -1 && errno != EEXIST) { ... }

    fd = open(PATH, O_WRONLY);      /* 읽는 쪽이 열릴 때까지 막힌다 */
    ...
    write(fd, msg, strlen(msg));
    close(fd);
```

```c
/* fifo_reader.c */
    fd = open(PATH, O_RDONLY);
    while ((n = read(fd, buf, sizeof buf - 1)) > 0) { ... }
    close(fd);
    unlink(PATH);                   /* 다 쓰면 지운다 */
```

한쪽을 배경으로 돌리고 다른 쪽을 실행한 결과입니다.

```text
파일 종류: FIFO
쓰는 쪽이 열릴 때까지 기다립니다...
ls 로 본 종류: p /tmp/cmid06_fifo
읽는 쪽이 열릴 때까지 기다립니다...
보냈습니다.
받음: 남남끼리 보낸 소식
끝났습니다.
```

- **두 프로그램을 따로 컴파일해 다른 터미널에서 실행**하십시오. `fork`로 만든 관계가 아니라는 점이 핵심입니다.
- **양쪽 모두 "기다립니다"를 출력한 뒤에야 자료가 흘렀습니다.** 9.3절에서 말한 `open`의 막힘이 그대로 보입니다.
- **먼저 실행한 쪽이 멈춥니다.** 반대쪽이 열릴 때까지 `open`이 기다리기 때문입니다(9.3절). 이것을 모르면 "프로그램이 죽었다"고 오해합니다.
- `ls -l`로 확인하면 파일 종류가 **`p`** 입니다(3강 3.1절).
- 다 쓰면 `unlink` 하십시오. 파이프와 달리 **파일 시스템에 남습니다.**

---

### 문제 9. `mmap` 공유 메모리

`MAP_SHARED`와 `MAP_PRIVATE`의 차이를 실험으로 보이십시오.

**정답 및 해설**

10.2절의 코드와 결과가 답입니다.

```text
MAP_SHARED  : 자식이 1000번 더한 뒤 부모가 본 값 = 1000
MAP_PRIVATE : 자식이 1000번 더한 뒤 부모가 본 값 = 0
```

- **4강 3.2절과 나란히 놓고 설명할 수 있어야 합니다.** 보통 변수는 복사되고, `MAP_SHARED` 메모리는 공유됩니다.
- 실패 판정은 `MAP_FAILED`와 비교합니다. `NULL`이 아닙니다.
- **부모도 동시에 더하도록 고쳐 보십시오.** 합이 2000이 되지 않고 실행할 때마다 달라집니다. 이것이 경쟁 상태이며 7강의 주제입니다.
- `munmap`으로 반납했는지 확인하십시오. 3강의 `close`, 1부의 `free`와 같은 책임입니다.

---

### 문제 10. `popen`을 안전하게 바꾸기

사용자 입력을 받아 `ls`를 실행하는 `popen` 코드를 **명령 주입이 불가능하게** 고치십시오.

**정답 및 해설**

**셸을 거치지 않도록 `fork`+`execvp`로 직접 만듭니다.**

```c
/* 위험: 셸이 해석한다
   snprintf(cmd, sizeof cmd, "ls %s", user_input);
   fp = popen(cmd, "r");                                  */

/* 안전: 인자를 배열로 넘기면 셸을 거치지 않는다 */
static FILE *safe_ls(const char *dir)
{
    int fd[2];
    pid_t pid;

    if (pipe(fd) == -1)
        return NULL;

    pid = fork();
    if (pid == -1) {
        close(fd[0]); close(fd[1]);
        return NULL;
    }
    if (pid == 0) {
        char *args[] = { "ls", "-1", NULL, NULL };

        args[2] = (char *) dir;     /* 인자 자리에 그대로 들어간다 */
        close(fd[0]);
        dup2(fd[1], STDOUT_FILENO);
        close(fd[1]);
        execvp("ls", args);
        _exit(127);
    }
    close(fd[1]);
    return fdopen(fd[0], "r");      /* 서술자를 FILE * 로 감싼다 (2강 10절) */
}
```

**두 판을 같은 입력으로 비교하십시오.**

```bash
./safe_exec '/etc; echo 침입성공'
```

```text
ls: cannot access '/etc; echo 침입성공': No such file or directory
자식 종료 상태: 2
```

```bash
./unsafe '/etc; echo 침입성공' | tail -3
```

```text
xml
zsh_command_not_found
침입성공
```

- **`침입성공`이 나오느냐가 갈림길입니다.** `popen` 판에서는 공격이 성공했고, `execvp` 판에서는 `ls`가 그런 이름의 디렉터리를 찾다 실패했을 뿐입니다.
- 셸이 없으므로 `;`·`|`·`$( )`·`&&` 어느 것도 특별한 뜻을 갖지 못합니다. **그저 파일 이름의 일부**입니다.
- `fdopen`으로 `FILE *`을 만들면 `popen`과 같은 편의를 얻습니다(2강 10절).
- 이 방식은 `pclose`가 없으므로 **`fclose` + `waitpid`** 로 직접 정리해야 합니다.
- **1부 10강의 신뢰 경계가 여기서도 그대로입니다.** 바깥에서 온 값을 해석기에 그대로 넘기지 않는 것이 원칙입니다. 11강에서 네트워크 입력을 다룰 때 다시 강조됩니다.

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 소스 — `pipe1.c`, `pipe_close.c`, `pipe_size.c`, `dup2_demo.c`, `minishell.c`, `popen_demo.c`, `fifo_writer.c`/`fifo_reader.c`, `mmap_share.c`, `safe_exec.c` |
| 2 | **닫기 실험** — `good`·`bad` 두 화면(문제 2) |
| 3 | 파이프 용량 측정 결과(문제 3) |
| 4 | **미니 셸 화면** — 재지정·파이프·3단 파이프(문제 5·6) |
| 5 | **진짜 셸과의 비교** — `ls /etc \| wc -l` 두 값(문제 6) |
| 6 | FIFO 두 터미널 화면(문제 8) |
| 7 | `MAP_SHARED`·`MAP_PRIVATE` 비교(문제 9) |
| 8 | 짧은 서술 ① 파이프에서 안 쓰는 쪽을 닫아야 하는 이유 |
| 9 | 짧은 서술 ② `dup2`가 재지정을 가능하게 하는 원리 |
| 10 | 짧은 서술 ③ `popen`이 위험한 이유와 대안 |

---

## 정리

| 구분 | 핵심 |
|---|---|
| 파이프 | 커널 안의 버퍼. **`[0]` 읽기, `[1]` 쓰기**, 단방향, 경계 없음 |
| 순서 | **`pipe` → `fork` → 각자 안 쓰는 쪽 `close`** |
| EOF | **쓰는 쪽이 모두 닫혔을 때만** 온다 |
| 안 닫으면 | **영원히 멈춘다** |
| 읽는 쪽이 없으면 | `SIGPIPE` → `SIG_IGN` 후 `EPIPE` |
| 용량 | 기본 **64 KiB**, 원자적 쓰기는 **`PIPE_BUF`(4096)** 까지 |
| `_GNU_SOURCE` | 리눅스 고유 확장에는 `-std=gnu17`로도 부족 |
| **`dup2`** | 재지정의 전부. **자식에서 `exec` 직전에** |
| 파이프라인 | `prev_read`를 넘기며 잇는다. **부모의 `close` 세 줄이 핵심** |
| `popen` | 편하지만 **셸을 거친다** — 사용자 입력 금지 |
| FIFO | 이름이 있어 **남남끼리** 가능. `open`이 막힌다 |
| `mmap` | **`MAP_SHARED`면 `fork` 후에도 공유** — 가장 빠르지만 동기화는 직접 |

---

## 다음 강의 예고

**7강 「스레드」**(APUE 11장)에서는 **한 프로그램 안의 여러 흐름**을 다룹니다.

- `pthread_create`로 스레드를 만든다 — `fork`와 무엇이 다른가
- **메모리를 처음부터 공유한다** — 오늘 `mmap`으로 힘들게 한 일이 기본값이 된다
- 그래서 생기는 **경쟁 상태**를 직접 재현한다
- 뮤텍스와 조건 변수로 지킨다
- `errno`가 왜 스레드마다 따로였는지 밝혀진다(1강 10.6절)

오늘 10.3절에서 예고한 **"동시에 더하면 값이 어긋난다"** 를 실제로 만들어 보고, 고치는 방법을 배웁니다.

---

## 부록 A. 이번 강의 명령·함수 요약

| 하려는 일 | 함수 / 명령 |
|---|---|
| 파이프 만들기 | `pipe(fd)` — `fd[0]` 읽기, `fd[1]` 쓰기 |
| 용량 확인 | `fcntl(fd, F_GETPIPE_SZ)` (`_GNU_SOURCE` 필요) |
| 용량 바꾸기 | `fcntl(fd, F_SETPIPE_SZ, 크기)` |
| 막히지 않게 | `fcntl(fd, F_SETFL, O_NONBLOCK)` |
| 서술자 복제 | `dup(fd)` · **`dup2(old, new)`** |
| 간편 파이프 | `popen(cmd, "r"/"w")` · `pclose(fp)` |
| 서술자 → `FILE *` | `fdopen(fd, "r")` |
| FIFO 만들기 | `mkfifo(path, 0600)` · 명령은 `mkfifo` |
| 메모리 대응 | `mmap(NULL, len, PROT_*, MAP_*, fd, 0)` · `munmap` |
| 파이프 확인(명령) | `ls -l /proc/PID/fd` |
| 원자적 쓰기 한계 | `PIPE_BUF` (`<limits.h>`) |

## 부록 B. 표준 문서와 출처

**시스템 호출**

| 함수 | 리눅스 man | POSIX 표준 |
|---|---|---|
| `pipe` | [`pipe(2)`](https://man7.org/linux/man-pages/man2/pipe.2.html) | [`pipe()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/pipe.html) |
| `dup`·`dup2` | [`dup(2)`](https://man7.org/linux/man-pages/man2/dup.2.html) | [`dup2()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/dup2.html) |
| `fcntl` | [`fcntl(2)`](https://man7.org/linux/man-pages/man2/fcntl.2.html) | [`fcntl()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/fcntl.html) |
| `mkfifo` | [`mkfifo(3)`](https://man7.org/linux/man-pages/man3/mkfifo.3.html) | [`mkfifo()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/mkfifo.html) |
| `mmap` | [`mmap(2)`](https://man7.org/linux/man-pages/man2/mmap.2.html) | [`mmap()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/mmap.html) |
| `popen` | [`popen(3)`](https://man7.org/linux/man-pages/man3/popen.3.html) | [`popen()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/popen.html) |

**개념 문서**

| 내용 | 문서 |
|---|---|
| 파이프와 FIFO 전반, **`PIPE_BUF` 원자성**·EOF 규칙 | [`pipe(7)`](https://man7.org/linux/man-pages/man7/pipe.7.html) |
| FIFO | [`fifo(7)`](https://man7.org/linux/man-pages/man7/fifo.7.html) |
| 기능 시험 매크로(`_GNU_SOURCE`) | [`feature_test_macros(7)`](https://man7.org/linux/man-pages/man7/feature_test_macros.7.html) |

**본문의 주장과 근거**

| 주장 | 근거 |
|---|---|
| 파이프 서술자는 3·4로 배정된다(2.2절) | `pipe1` 실행 결과 |
| **안 닫으면 EOF가 오지 않는다**(3.2절) | `pipe_close good`은 EOF 수신, `bad`는 5초 알람에 걸림 |
| **파이프 용량 65536, `PIPE_BUF` 4096**(4.1절) | 직접 채운 값 65536 = `F_GETPIPE_SZ` 값, `<limits.h>`의 `PIPE_BUF` |
| `F_GETPIPE_SZ`는 `-std=gnu17`로도 안 보인다(4.2절) | `_GNU_SOURCE` 없이 컴파일 → `undeclared` 오류 |
| `dup2`로 목적지만 바뀐다(5.2절) | `dup2_demo` — ①③은 화면, ②는 `out.txt` |
| **미니 셸이 진짜 셸과 같은 결과**(7.4절) | `ls /etc \| wc -l` → 둘 다 **175**, 3단 파이프 결과도 일치 |
| 파이프 중간 실패에도 나머지는 동작(7.5절) | `없는명령 \| wc -l` → 오류 후 `wc`가 0 출력 |
| **`MAP_SHARED` 1000 vs `MAP_PRIVATE` 0**(10.2절) | `mmap_share` 두 실행 |
| `popen`이 셸을 거친다(8.2절) | `popen(3)` 문서 — `/bin/sh -c` 로 실행 |
| **명령 주입이 실제로 성공한다**(8.2절·문제 10) | 같은 입력 `/etc; echo 침입성공` — `popen` 판은 **`침입성공` 출력**, `execvp` 판은 `ls`가 `No such file` |
| 양방향 통신이 동작한다(문제 7) | `twoway` — 2·3·4·5의 제곱 4·9·16·25 |
| FIFO는 남남끼리 되고 `open`이 막힌다(문제 8) | 양쪽이 "기다립니다"를 출력한 뒤 전달, `ls -l` 첫 글자 **`p`** |

> 값은 모두 **Ubuntu 24.04에서 실제로 실행한 결과**입니다. `ls /etc`의 개수(175)는 설치 상태에 따라 다르므로, **자기 환경에서 두 값이 서로 같은지**를 보시면 됩니다.
{: .prompt-tip }

## 부록 C. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `pipe1.c` | 부모–자식 파이프 기본 | 2 |
| `pipe_close.c` | **닫기를 빠뜨리면** | 3 |
| `pipe_size.c` | 용량과 `PIPE_BUF`, `_GNU_SOURCE` | 4 |
| `dup2_demo.c` | 재지정의 원리 | 5 |
| **`minishell.c`** | **재지정 + 파이프를 갖춘 셸** | 6 · 7 |
| `popen_demo.c` | `popen`·`pclose` | 8 |
| `fifo_writer.c`·`fifo_reader.c` | FIFO | 문제 8 |
| `mmap_share.c` | 공유 메모리 | 10 |
| `safe_exec.c` | `popen`의 안전한 대안 | 문제 10 |
