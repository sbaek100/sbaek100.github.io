---
title: 중급 C 프로그래밍 5강 - 시그널
date: 2026-12-21 09:00:00 +0900
categories:
  - 1.응용강의
  - 중급 C 프로그래밍
tags:
  - C언어
  - 시스템프로그래밍
  - 리눅스
  - 시그널
  - sigaction
  - SIGCHLD
  - EINTR
  - 비동기
  - 우아한종료
pin:
mermaid: false
---

> **학습 목표**
> 1. 시그널이 **비동기 알림**이라는 것을 설명하고 주요 시그널을 구분할 수 있다.
> 2. `kill` 명령과 `kill()` 시스템 호출로 시그널을 보낼 수 있다.
> 3. **`sigaction`** 으로 처리기를 등록할 수 있고, `signal`보다 나은 이유를 설명할 수 있다.
> 4. 처리기 안에서 **해도 되는 일과 안 되는 일**을 구분할 수 있다.
> 5. `volatile sig_atomic_t` 깃발 방식으로 **우아한 종료**를 구현할 수 있다.
> 6. **`SIGCHLD`로 좀비를 자동으로 거둘 수 있다.**
> 7. **`EINTR`** 이 왜 생기는지 설명하고 `SA_RESTART`와 재시도로 대응할 수 있다.
> 8. `sigprocmask`로 시그널을 잠시 막아 임계 구역을 지킬 수 있다.
> 9. `alarm`으로 시간 제한을 걸 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

지금까지 프로그램은 **자기가 부른 것만** 처리했습니다. 이번 강의에서는 **밖에서 끼어드는 사건**을 다룹니다.

두 가지 빚을 갚습니다.

| 미뤄 둔 것 | 어디서 |
|---|---|
| **`EINTR`이면 다시 시도한다** — 왜? | 2강 4.2절 `write_full` |
| **좀비를 안 남기려면?** | 4강 8.3절 |

그리고 실용적인 질문에 답합니다.

- Ctrl+C를 누르면 실제로 무슨 일이 일어나는가
- `kill -9`는 왜 언제나 통하는가
- 서버가 "정리하고 나서" 끝나게 하려면

**기준 교재: APUE 3판 10장(Signals).**

이 강의는 **3회차 분량**(모두 합쳐 약 480분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제5절 | 시그널의 기본과 안전한 처리기 | 170분 |
| **2회차** | 제6절 ~ 제11절 | 우아한 종료·`EINTR`·`SIGCHLD` | 200분 |
| **3회차** | 실습문제 | 스스로 해 보기 | 110분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 시그널이란 | 30분 |
| 제2절 | 주요 시그널과 기본 동작 | 30분 |
| 제3절 | 시그널 보내기 | 25분 |
| 제4절 | `sigaction`으로 붙잡기 | 45분 |
| 제5절 | 처리기 안에서 할 수 있는 일 | 40분 |
| 제6절 | 우아한 종료 | 35분 |
| 제7절 | `EINTR` — 2강의 그 처리 | 45분 |
| 제8절 | **`SIGCHLD`로 좀비 없애기** | 40분 |
| 제9절 | 시그널 막기 | 35분 |
| 제10절 | `alarm`과 시간 제한 | 30분 |
| 제11절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 디렉터리**는 `~/cmid/lab05`를 사용합니다. 이번 강의는 **VM 한 대**(`c-srv`)에서 진행합니다.

```bash
mkdir -p ~/cmid/lab05 && cd ~/cmid/lab05
```

---

## 제1절. 시그널이란

### 1.1 소프트웨어 인터럽트

> **시그널은 프로세스에게 "무슨 일이 일어났다"고 알리는 아주 짧은 통보입니다.**

| 특징 | 설명 |
|---|---|
| **비동기** | 언제 올지 모른다. 코드의 **어느 줄에서든** 끼어든다 |
| **정보가 없다** | 번호 하나뿐. "무엇이" 일어났는지만 알 수 있다 |
| **쌓이지 않는다** | 같은 시그널이 여러 번 와도 **한 번으로 합쳐질 수 있다** |
| 커널이 전달한다 | 프로세스끼리 직접 주고받는 것이 아니다 |

**"정보가 없다"와 "쌓이지 않는다"** 는 시그널의 근본적인 한계이며, 이번 강의의 여러 규칙이 여기서 나옵니다.

### 1.2 누가 보내는가

| 보내는 쪽 | 예 |
|---|---|
| **사용자** | Ctrl+C → `SIGINT`, Ctrl+Z → `SIGTSTP` |
| **커널** | 잘못된 메모리 접근 → `SIGSEGV`, 0으로 나눔 → `SIGFPE` |
| **다른 프로세스** | `kill` 명령, `kill()` 호출 |
| **자기 자신** | `alarm()` → `SIGALRM`, `abort()` → `SIGABRT` |
| **자식의 상태 변화** | 자식이 끝남 → **`SIGCHLD`** |

### 1.3 받으면 무슨 일이 일어나는가

프로세스는 시그널마다 **셋 중 하나**로 반응합니다.

| 반응 | 설명 |
|---|---|
| **기본 동작** | 시그널마다 미리 정해져 있다(제2절) |
| **무시** | `SIG_IGN` |
| **처리기 실행** | 우리가 등록한 함수를 부른다(제4절) |

> **`SIGKILL`(9)과 `SIGSTOP`(19)은 예외입니다.**
> 이 둘은 **붙잡을 수도, 무시할 수도, 막을 수도 없습니다.** 그래서 `kill -9`가 언제나 통합니다. 반대로 말하면, `kill -9`로 죽은 프로그램은 **정리할 기회를 전혀 얻지 못합니다.** 열린 파일의 버퍼도, 임시 파일도 그대로 남습니다.
{: .prompt-warning }

---

## 제2절. 주요 시그널과 기본 동작

### 2.1 이 시스템의 시그널 번호

```c
    printf("%-10s %3s  %s\n", "이름", "번호", "설명");
    for (i = 0; i < sizeof list / sizeof list[0]; i++)
        printf("%-10s %3d  %s\n", names[i], list[i], strsignal(list[i]));
```

```bash
gcc -Wall -Wextra -std=gnu17 signums.c -o signums
```

```bash
./signums
```

```text
이름     번호  설명
--------------------------------------------------
SIGHUP       1  Hangup
SIGINT       2  Interrupt
SIGQUIT      3  Quit
SIGILL       4  Illegal instruction
SIGABRT      6  Aborted
SIGFPE       8  Floating point exception
SIGKILL      9  Killed
SIGUSR1     10  User defined signal 1
SIGSEGV     11  Segmentation fault
SIGUSR2     12  User defined signal 2
SIGPIPE     13  Broken pipe
SIGALRM     14  Alarm clock
SIGTERM     15  Terminated
SIGCHLD     17  Child exited
SIGCONT     18  Continued
SIGSTOP     19  Stopped (signal)
SIGTSTP     20  Stopped
```

셸로도 볼 수 있습니다.

```bash
kill -l | head -3
```

```text
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
```

> **번호는 아키텍처마다 다를 수 있습니다.** 위 값은 x86-64 리눅스 기준입니다. 코드에는 **반드시 이름**을 쓰십시오. 1강 10.1절에서 `errno`에 대해 말한 것과 같은 원칙입니다.
{: .prompt-warning }

### 2.2 알아야 할 것들

| 시그널 | 기본 동작 | 언제 |
|---|---|---|
| `SIGINT` (2) | 종료 | **Ctrl+C** |
| `SIGQUIT` (3) | 종료 + 코어 | Ctrl+\\ |
| `SIGKILL` (9) | **종료 — 막을 수 없음** | `kill -9` |
| `SIGSEGV` (11) | 종료 + 코어 | **잘못된 메모리 접근** |
| `SIGPIPE` (13) | **종료** | **받는 쪽이 없는 파이프에 쓸 때** |
| `SIGALRM` (14) | 종료 | `alarm()` 만료 |
| `SIGTERM` (15) | 종료 | **`kill` 기본값** — 정중한 종료 요청 |
| `SIGCHLD` (17) | **무시** | **자식의 상태 변화** |
| `SIGSTOP` (19) | 멈춤 — **막을 수 없음** | — |
| `SIGTSTP` (20) | 멈춤 | Ctrl+Z |
| `SIGCONT` (18) | 계속 | `fg`, `bg` |
| `SIGUSR1`/`SIGUSR2` | 종료 | **우리 마음대로 쓰라고 비워 둔 것** |

전체 목록과 기본 동작은 [`signal(7)`](https://man7.org/linux/man-pages/man7/signal.7.html)에 있습니다.

### 2.3 두 가지를 기억하십시오

**① `SIGCHLD`의 기본 동작은 "무시"입니다.**

그래서 4강에서 자식이 끝나도 부모는 아무것도 느끼지 못했고, 좀비가 쌓였습니다. 제8절에서 이것을 뒤집습니다.

**② `SIGPIPE`의 기본 동작은 "종료"입니다.**

받는 쪽이 사라진 파이프나 소켓에 쓰면 **프로그램이 그냥 죽습니다.** 서버 프로그램에서는 대부분 이것을 무시하도록 설정하고 `write`의 반환값(`EPIPE`)으로 판단합니다. 9강에서 다시 만납니다.

```c
signal(SIGPIPE, SIG_IGN);       /* 서버에서 흔히 하는 설정 */
```

---

## 제3절. 시그널 보내기

### 3.1 명령으로

```bash
kill -TERM 1234          # 이름으로
kill -15 1234            # 번호로
kill 1234                # 기본값은 SIGTERM
kill -9 1234             # SIGKILL — 최후의 수단
```

| 명령 | 대상 |
|---|---|
| `kill PID` | 그 프로세스 하나 |
| `killall 이름` | 이름이 같은 모든 프로세스 |
| `pkill -f 무늬` | 명령줄이 무늬와 맞는 것 |
| `kill -0 PID` | **보내지 않고 살아 있는지만 확인** |

`kill -0`이 유용합니다. 시그널을 보내지 않고 **존재 여부와 권한만 검사**합니다.

### 3.2 함수로

```c
#include <signal.h>

int kill(pid_t pid, int sig);
int raise(int sig);              /* 자기 자신에게 */
```

| `pid` | 대상 |
|---|---|
| 양수 | 그 PID |
| 0 | **자기 프로세스 그룹 전체** |
| −1 | 보낼 수 있는 모든 프로세스 |
| 음수 | 그 프로세스 그룹 |

> **Ctrl+C는 프로세스 하나가 아니라 프로세스 그룹 전체에 갑니다.**
> 그래서 `프로그램A | 프로그램B`를 실행 중에 Ctrl+C를 누르면 **둘 다** 죽습니다. 터미널이 **전면 프로세스 그룹**에 `SIGINT`를 보내기 때문입니다.
{: .prompt-info }

### 3.3 이름과 번호 사이

```c
strsignal(SIGINT)       /* "Interrupt" — 사람이 읽는 설명 */
```

4강 7.2절에서 `WTERMSIG(status)`를 `strsignal`에 넘겨 쓴 것이 이것입니다.

---

## 제4절. `sigaction`으로 붙잡기

### 4.1 `signal`을 쓰지 마십시오

옛 교재에는 `signal()`이 나옵니다. **쓰지 마십시오.**

| 문제 | 설명 |
|---|---|
| **동작이 시스템마다 다르다** | 처리기 실행 후 기본 동작으로 되돌아가는 판이 있다 |
| 끊긴 시스템 호출의 처리를 고를 수 없다 | `SA_RESTART`를 지정할 수 없다 |
| 처리기가 도는 동안의 차단을 고를 수 없다 | |

[`signal(2)`](https://man7.org/linux/man-pages/man2/signal.2.html)의 man 페이지도 **이식성 있는 코드에서는 `sigaction`을 쓰라**고 권합니다.

### 4.2 `sigaction`

```c
#include <signal.h>

int sigaction(int signum, const struct sigaction *act,
              struct sigaction *oldact);
```

```c
struct sigaction {
    void     (*sa_handler)(int);        /* 처리기 함수 */
    sigset_t   sa_mask;                 /* 처리기가 도는 동안 추가로 막을 것 */
    int        sa_flags;                /* 동작 조절 */
    ...
};
```

| `sa_flags` | 하는 일 |
|---|---|
| **`SA_RESTART`** | **끊긴 시스템 호출을 자동으로 다시 시작**(제7절) |
| `SA_NOCLDSTOP` | 자식이 **멈출 때는** `SIGCHLD`를 보내지 않는다 |
| `SA_NOCLDWAIT` | 자식을 자동으로 거둔다 — 좀비가 생기지 않는다 |
| `SA_SIGINFO` | 더 자세한 정보를 받는 처리기를 쓴다 |

### 4.3 등록하는 정형

```c
    memset(&sa, 0, sizeof sa);
    sa.sa_handler = on_int;
    sigemptyset(&sa.sa_mask);       /* 처리기가 도는 동안 더 막을 시그널 없음 */
    sa.sa_flags = SA_RESTART;       /* 끊긴 시스템 호출을 자동으로 다시 시작 */

    if (sigaction(SIGINT, &sa, NULL) == -1) {
        fprintf(stderr, "sigaction: %s\n", strerror(errno));
        return 1;
    }
```

**`memset`으로 통째로 0을 채우고 시작하는 것**이 중요합니다. `struct sigaction`에는 표준에 없는 항목이 더 있을 수 있어, 초기화하지 않으면 쓰레기 값이 들어갑니다.

> **처리기가 도는 동안 같은 시그널은 자동으로 막힙니다.** `sa_mask`에 넣지 않아도 그렇습니다. 그래서 처리기가 자기 자신에게 다시 불려 겹치는 일은 없습니다.
{: .prompt-tip }

### 4.4 확인

```c
static volatile sig_atomic_t got = 0;

static void on_int(int signo)
{
    (void) signo;
    got++;                          /* 여기서는 이것만 한다 */
}
```

```c
    while (got < limit) {
        if (got != last) {          /* 출력은 주 흐름에서 한다 */
            printf("SIGINT 를 %d번 받았습니다\n", (int) got);
            fflush(stdout);
            last = got;
        }
        pause();                    /* 시그널이 올 때까지 잔다 */
    }
```

```bash
./catch_int 3
```

다른 터미널에서 `kill -INT <PID>`를 세 번 보내거나, 같은 터미널에서 Ctrl+C를 세 번 누르십시오.

```text
PID 466 — Ctrl+C 를 3번 누르면 끝냅니다
SIGINT 를 1번 받았습니다
SIGINT 를 2번 받았습니다
3번 받았으므로 정상적으로 끝냅니다
```

**Ctrl+C를 눌렀는데 프로그램이 죽지 않습니다.** 기본 동작(종료)을 우리 처리기가 대신했기 때문입니다.

> `pause()`는 **어떤 시그널이든 올 때까지** 잠듭니다. 시그널이 오면 `-1`과 `EINTR`을 돌려주며 깨어납니다. 바쁘게 도는 반복문 대신 이것을 쓰면 CPU를 쓰지 않고 기다릴 수 있습니다.
{: .prompt-info }

---

## 제5절. 처리기 안에서 할 수 있는 일

**이 절이 이번 강의에서 가장 중요합니다.**

### 5.1 왜 조심해야 하는가

처리기는 **주 흐름의 어느 줄에서든** 끼어듭니다. 하필 `printf`가 버퍼를 고치는 도중일 수도 있습니다. 그 상태에서 처리기가 또 `printf`를 부르면 **버퍼가 망가집니다.**

[`signal-safety(7)`](https://man7.org/linux/man-pages/man7/signal-safety.7.html)은 이렇게 설명합니다.

> "the *stdio* library, all of whose functions are not async-signal-safe"

그리고 `printf`가 `printf`를 끼어들면 **"unpredictable results"** 가 된다고 못 박습니다.

### 5.2 안전한 함수 목록

> **비동기 시그널 안전(async-signal-safe)** 함수만 처리기에서 부를 수 있습니다.

| 안전한 것 (일부) | 안전하지 않은 것 |
|---|---|
| **`write`** | **`printf`·`fprintf`·`puts`** — stdio 전부 |
| **`_exit`** | `exit` |
| **`waitpid`** | `malloc`·`free` |
| `kill`·`signal`·`sigaction` | `strerror` |
| `open`·`read`·`close` | `getpwuid`·`getaddrinfo` |
| `time`·`alarm`·`pause` | 대부분의 라이브러리 함수 |

전체 목록은 [`signal-safety(7)`](https://man7.org/linux/man-pages/man7/signal-safety.7.html)에 표로 실려 있습니다. **외울 필요는 없고, 처리기를 쓸 때 확인하는 습관**이 필요합니다.

### 5.3 그래서 어떻게 쓰는가

> **처리기는 깃발만 세우고, 일은 주 흐름이 한다.**

```c
static volatile sig_atomic_t got = 0;

static void on_int(int signo)
{
    (void) signo;
    got++;              /* 이것뿐 */
}
```

| 요소 | 왜 필요한가 |
|---|---|
| **`volatile`** | 컴파일러가 "이 값은 안 바뀐다"고 판단해 최적화하는 것을 막는다 |
| **`sig_atomic_t`** | 읽고 쓰는 것이 **나뉘지 않는** 자료형 |
| `static` | 처리기와 주 흐름이 함께 본다 |

> **`int`가 아니라 `sig_atomic_t`여야 합니다.**
> 크기가 큰 자료형은 여러 번에 나누어 읽고 쓸 수 있어, 그 중간에 시그널이 끼어들면 **반쪽짜리 값**을 보게 됩니다. `sig_atomic_t`는 그런 일이 없도록 표준이 보장하는 자료형입니다.
{: .prompt-danger }

**`volatile`을 빠뜨리면** 다음 반복문이 영원히 끝나지 않을 수 있습니다.

```c
while (!stop)       /* stop 이 volatile 이 아니면 컴파일러가 "늘 참"으로 최적화할 수 있다 */
    ;
```

### 5.4 꼭 출력해야 한다면

`write`는 안전합니다.

```c
static void on_int(int signo)
{
    (void) signo;
    write(STDERR_FILENO, "\n중단 요청을 받았습니다\n", 34);
}
```

**길이를 직접 세어 넣어야 하는 것**이 불편하지만, `strlen`은 안전 목록에 있으므로 함께 쓸 수 있습니다. `printf`는 절대 쓰지 마십시오.

### 5.5 `errno`를 지켜 주십시오

처리기가 부른 함수가 `errno`를 바꾸면, **주 흐름이 보고 있던 `errno`가 오염됩니다.**

```c
static void on_chld(int signo)
{
    int saved = errno;              /* 처리기는 errno 를 건드리면 안 된다 */
    pid_t pid;

    while ((pid = waitpid(-1, NULL, WNOHANG)) > 0)
        reaped++;

    (void) signo;
    errno = saved;                  /* 원래대로 되돌려 놓는다 */
}
```

1강 10.2절에서 "실패를 확인한 직후에 `errno`를 보라"고 한 이유가 여기에도 있습니다. **시그널은 그 사이에도 끼어들 수 있습니다.**

---

> **▶ 여기서부터 2회차 — 우아한 종료·`EINTR`·`SIGCHLD`**
> 제6절 ~ 제11절, 약 200분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제6절. 우아한 종료

### 6.1 문제

서버가 `SIGTERM`을 받으면 곧바로 죽습니다. 그러면 다음이 남습니다.

| 남는 것 | 결과 |
|---|---|
| 비우지 않은 버퍼 | 로그가 잘린다 |
| 임시 파일 | 쓰레기가 쌓인다 |
| 처리 중이던 요청 | 클라이언트가 끊긴다 |

### 6.2 깃발 방식

```c
static volatile sig_atomic_t stop = 0;

static void on_term(int signo)
{
    stop = signo;                   /* 어떤 시그널이었는지도 남긴다 */
}
```

```c
    while (!stop) {
        char line[64];
        int len = snprintf(line, sizeof line, "%d 번째 작업\n", ++round);

        if (len > 0)
            write(fd, line, (size_t) len);
        usleep(200000);
    }

    /* 여기서 안전하게 정리한다 — 처리기 안이 아니다 */
    printf("시그널 %d(%s) 를 받아 정리합니다\n", (int) stop, strsignal(stop));
    printf("%d 번째 작업까지 마쳤습니다\n", round);

    if (fsync(fd) == -1)
        fprintf(stderr, "fsync: %s\n", strerror(errno));
    if (close(fd) == -1)
        fprintf(stderr, "close: %s\n", strerror(errno));
```

```bash
./graceful &
sleep 1
kill -TERM $!
```

```text
PID 471 — 일하는 중입니다. SIGTERM 이나 SIGINT 를 보내 보십시오.
시그널 15(Terminated) 를 받아 정리합니다
6 번째 작업까지 마쳤습니다
정리를 마치고 끝냅니다
```

```bash
wc -l work.log
```

```text
6 work.log
```

**정리가 처리기 밖에서 이루어졌습니다.** `printf`·`strsignal`·`fsync`는 모두 처리기에서는 쓸 수 없는 것들인데, 주 흐름이므로 마음껏 쓸 수 있습니다.

### 6.3 이 구조가 표준입니다

| 층 | 하는 일 |
|---|---|
| 처리기 | **깃발 하나만 세운다** |
| 주 흐름 | 반복문 조건에서 깃발을 확인 |
| 반복문 밖 | **자원을 정리하고 정상 종료** |

이 방식이 3강 10절의 원자적 파일 갱신, 2강 5절의 서술자 정리와 자연스럽게 이어집니다. 16강 종합 과제의 서버도 이 구조를 씁니다.

> **`SIGKILL`에는 이 방법이 통하지 않습니다.** 그래서 서비스를 멈출 때는 먼저 `SIGTERM`을 보내고, 일정 시간을 기다린 뒤에도 살아 있으면 그때 `SIGKILL`을 씁니다. `systemd`가 정확히 그렇게 동작합니다.
{: .prompt-warning }

---

## 제7절. `EINTR` — 2강의 그 처리

### 7.1 무슨 일이 일어나는가

`read`가 자료를 기다리며 막혀 있는데 시그널이 도착하면, 커널은 **처리기를 부르기 위해 `read`를 중단**시킵니다. 처리기가 끝난 뒤 두 가지 중 하나가 일어납니다.

| `SA_RESTART` | 결과 |
|---|---|
| **없음** | `read`가 **`-1`과 `EINTR`** 로 실패한다 |
| **있음** | `read`를 **자동으로 다시 시작**한다 |

### 7.2 직접 확인

`eintr_demo.c`는 자료가 오지 않는 파이프에서 읽으며 1초 뒤 `SIGALRM`을 맞습니다. 처리기는 파이프에 1바이트를 넣습니다.

```c
/* 처리기: 파이프에 1바이트를 넣는다. write 는 async-signal-safe 다 */
static void on_alarm(int signo)
{
    (void) signo;
    write(pipefd[1], "X", 1);
}
```

```c
    sa.sa_flags = restart ? SA_RESTART : 0;
    ...
    alarm(1);
    errno = 0;
    n = read(pipefd[0], buf, sizeof buf);   /* 자료가 없어 여기서 막힌다 */

    if (n == -1)
        printf("read 결과: -1, errno=%d (%s)\n", errno, strerror(errno));
    else
        printf("read 결과: %zd 바이트 — 다시 시작되어 성공했습니다\n", n);
```

```bash
./eintr_demo norestart
```

```text
SA_RESTART 없음 — 1초 뒤 SIGALRM 이 옵니다
read 결과: -1, errno=4 (Interrupted system call)
```

```bash
./eintr_demo restart
```

```text
SA_RESTART 있음 — 1초 뒤 SIGALRM 이 옵니다
read 결과: 1 바이트 — 다시 시작되어 성공했습니다
```

**`errno=4`가 `EINTR`입니다.** 같은 코드가 플래그 하나로 전혀 다르게 동작했습니다.

### 7.3 그래서 2강의 그 코드였습니다

```c
        ssize_t w = write(fd, buf + done, n - done);
        if (w < 0) {
            if (errno == EINTR)
                continue;              /* 시그널에 끊겼을 뿐이다 (5강) */
            return -1;
        }
```

**`EINTR`은 오류가 아닙니다.** "아직 아무 일도 하지 않았으니 다시 부르라"는 뜻입니다. 그냥 실패로 처리하면, **시그널이 올 때마다 파일 복사가 중단되는** 프로그램이 됩니다.

> **`SA_RESTART`를 쓰면 되지 않나요?**
> 대체로 그렇지만 **완전하지 않습니다.** `SA_RESTART`가 있어도 다시 시작되지 않는 것들이 있습니다.
>
> | 다시 시작되지 않는 것 | |
> |---|---|
> | `sleep`·`nanosleep` | 잔 시간만큼 줄어든 채 돌아온다 |
> | `select`·`poll`·`epoll_wait` | 9·10강에서 중요해진다 |
> | 시간 제한을 건 소켓 연산 | |
>
> **그래서 `EINTR` 재시도 코드는 여전히 필요합니다.** 자세한 목록은 [`signal(7)`](https://man7.org/linux/man-pages/man7/signal.7.html)의 "Interruption of system calls" 절에 있습니다.
{: .prompt-warning }

### 7.4 `sleep`은 다시 시작되지 않습니다

직접 확인해 보십시오.

```c
    sa.sa_flags = SA_RESTART;
    sigaction(SIGALRM, &sa, NULL);
    alarm(1);
    left = sleep(5);
    printf("sleep(5) 반환값 = %u  <- 남은 초. SA_RESTART 여도 다시 자지 않는다\n", left);
```

```text
sleep(5) 반환값 = 4  <- 남은 초. SA_RESTART 여도 다시 자지 않는다
```

**5초를 자려 했는데 1초 만에 깨어났습니다.** 반환값 4가 "남은 시간"입니다.

더 고약한 것은 **`sleep`이 초 단위**라는 점입니다. 0.9초가 남았어도 0을 돌려주므로, 다음처럼 되풀이해도 시간이 모자랍니다.

```c
while ((left = sleep(left)) > 0)    /* 밀리초 단위 잔여 시간을 잃는다 */
    ;
```

**정확하게 하려면 `nanosleep`을 씁니다.**

```c
/* sleep_full: 시그널에 끊겨도 정해진 시간을 채운다 */
static void sleep_full(time_t sec)
{
    struct timespec req, rem;

    req.tv_sec = sec;
    req.tv_nsec = 0;

    /* nanosleep 은 남은 시간을 나노초까지 알려 준다 — sleep 과 다르다 */
    while (nanosleep(&req, &rem) == -1 && errno == EINTR)
        req = rem;
}
```

---

## 제8절. `SIGCHLD`로 좀비 없애기

**4강에서 남긴 숙제를 풉니다.**

### 8.1 자식이 끝나면 알려 준다

`SIGCHLD`의 기본 동작은 "무시"이지만, **처리기를 등록하면 자식이 끝날 때마다 알림을 받습니다.**

```c
static void on_chld(int signo)
{
    int saved = errno;
    pid_t pid;

    /* 여러 자식이 동시에 끝났을 수 있으므로 되풀이한다 */
    while ((pid = waitpid(-1, NULL, WNOHANG)) > 0)
        reaped++;

    (void) signo;
    errno = saved;
}
```

**세 가지가 모두 필요합니다.**

| 요소 | 왜 |
|---|---|
| **`while` 반복문** | **시그널은 쌓이지 않는다.** 셋이 동시에 끝나도 `SIGCHLD`는 한 번일 수 있다 |
| **`WNOHANG`** | 거둘 자식이 없을 때 **막히면 안 된다** |
| **`errno` 보존** | 5.5절 |

> **`while` 없이 `waitpid`를 한 번만 부르면 좀비가 남습니다.**
> 이것이 가장 흔한 실수입니다. 시그널이 합쳐지는 성질(1.1절)을 잊었기 때문에 생깁니다.
{: .prompt-danger }

### 8.2 확인

```bash
./sigchld 5
```

```text
자식 5개를 만들었습니다. wait 을 부르지 않고 기다립니다.
처리기가 거둔 자식 수: 5
남은 좀비: 0개
```

**부모는 `wait`을 한 번도 부르지 않았는데 좀비가 없습니다.** 처리기가 대신 거두었습니다.

처리기를 등록하지 않으면 어떻게 되는지 비교해 보십시오.

```bash
./nochld
```

```text
남은 좀비: 5개
```

**같은 프로그램에서 처리기만 뺐을 뿐인데 좀비 5개가 남습니다.**

### 8.3 더 간단한 방법

자식의 종료 상태가 필요 없다면 **커널에게 맡길 수 있습니다.**

```c
struct sigaction sa;

memset(&sa, 0, sizeof sa);
sa.sa_handler = SIG_IGN;            /* SIGCHLD 를 명시적으로 무시 */
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;
sigaction(SIGCHLD, &sa, NULL);
```

> **"기본 동작이 무시"인 것과 "명시적으로 `SIG_IGN`을 설정"하는 것은 다릅니다.**
> 앞은 좀비가 남고, 뒤는 **커널이 자동으로 거두어 좀비가 생기지 않습니다.** 이름은 같지만 결과가 정반대인, 시그널에서 가장 헷갈리는 부분입니다.
{: .prompt-warning }

`SA_NOCLDWAIT` 플래그도 같은 효과를 냅니다.

### 8.4 방법 비교

| 방법 | 종료 상태를 알 수 있나 | 복잡도 |
|---|---|---|
| `wait`으로 직접 | 예 | 막힌다 |
| `WNOHANG`으로 주기적 확인 | 예 | 잊기 쉽다 |
| **`SIGCHLD` 처리기** | **예** | 보통 — **가장 널리 쓰인다** |
| `SIG_IGN` / `SA_NOCLDWAIT` | **아니오** | 가장 간단 |
| 두 번 `fork` | 아니오 | 4강 문제 8 |

---

## 제9절. 시그널 막기

### 9.1 왜 막는가

여러 줄에 걸친 자료 갱신 도중에 처리기가 끼어들면, **반쯤 고쳐진 상태**를 보게 됩니다. 그 구간에서는 시그널을 잠시 막아야 합니다.

```c
#include <signal.h>

int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```

| `how` | 하는 일 |
|---|---|
| `SIG_BLOCK` | 목록에 있는 것을 **추가로 막는다** |
| `SIG_UNBLOCK` | 막기를 푼다 |
| `SIG_SETMASK` | 목록으로 **통째로 교체** |

시그널 집합은 전용 함수로 다룹니다.

| 함수 | 하는 일 |
|---|---|
| `sigemptyset` | 비운다 |
| `sigfillset` | 모두 채운다 |
| `sigaddset` / `sigdelset` | 하나 넣기 / 빼기 |
| `sigismember` | 들어 있는지 확인 |

### 9.2 막힌 시그널은 사라지지 않습니다

```c
    sigemptyset(&block);
    sigaddset(&block, SIGINT);
    ...
    if (sigprocmask(SIG_BLOCK, &block, &old) == -1) { ... }

    for (i = 3; i > 0; i--) {
        printf("  임계 구역 %d... (받은 횟수 %d)\n", i, (int) count);
        fflush(stdout);
        sleep(1);
    }

    /* 막혀 있는 동안 도착한 시그널이 있는지 확인한다 */
    if (sigpending(&pending) == 0 && sigismember(&pending, SIGINT))
        printf("--- 막혀 있는 동안 SIGINT 가 도착해 대기 중입니다 ---\n");
    ...
    if (sigprocmask(SIG_SETMASK, &old, NULL) == -1) { ... }
```

```bash
./block_demo &
sleep 1
kill -INT $!
```

```text
PID 478
--- 지금부터 3초간 SIGINT 를 막습니다 ---
  임계 구역 3... (받은 횟수 0)
  임계 구역 2... (받은 횟수 0)
  임계 구역 1... (받은 횟수 0)
--- 막혀 있는 동안 SIGINT 가 도착해 대기 중입니다 ---
--- 막기를 풀었습니다. 받은 횟수 1 ---
```

**세 가지를 확인하십시오.**

| 관찰 | 뜻 |
|---|---|
| 막혀 있는 동안 `count`가 0 | 처리기가 불리지 않았다 |
| `sigpending`이 참 | **버려지지 않고 대기 중이다** |
| 막기를 푼 직후 `count`가 1 | **그때 배달되었다** |

> **막는 것(block)과 무시하는 것(ignore)은 다릅니다.**
>
> | | 막기 | 무시 |
> |---|---|---|
> | 시그널이 | **기다린다** | **버려진다** |
> | 풀면 | **그때 배달된다** | 되돌아오지 않는다 |
{: .prompt-tip }

### 9.3 여러 번 온 것은 한 번이 됩니다

막혀 있는 동안 `SIGINT`가 열 번 와도, 풀었을 때 **한 번만** 배달됩니다. 1.1절의 "쌓이지 않는다"가 이것입니다.

**그래서 8.1절에서 `while` 반복문이 필요했습니다.** "몇 번 왔는가"를 시그널로 세면 안 되고, **"할 일이 남았는가"를 직접 확인**해야 합니다.

### 9.4 `fork`·`exec`과 마스크

| 동작 | 시그널 마스크 | 처리기 |
|---|---|---|
| `fork` | **물려받는다** | **물려받는다** |
| `exec` | **물려받는다** | **기본 동작으로 되돌아간다** |

**`exec` 후에도 마스크는 남는다**는 점에 주의하십시오. 시그널을 막아 둔 채 다른 프로그램을 실행하면, 그 프로그램이 영문도 모른 채 시그널을 못 받습니다. **`exec` 전에 마스크를 되돌려 놓으십시오.**

---

## 제10절. `alarm`과 시간 제한

### 10.1 스스로에게 시그널 예약하기

```c
unsigned int alarm(unsigned int seconds);
```

`seconds` 뒤에 자신에게 `SIGALRM`을 보냅니다. `alarm(0)`은 예약을 취소합니다.

| 특징 |
|---|
| 프로세스당 **알람은 하나뿐** — 새로 걸면 이전 것이 취소된다 |
| 반환값은 **이전 알람의 남은 시간** |

### 10.2 읽기에 시간 제한 걸기

```c
    sa.sa_flags = 0;                /* SA_RESTART 를 주면 안 된다! */
    ...
    alarm(sec);
    errno = 0;
    n = read(STDIN_FILENO, buf, sizeof buf - 1);
    alarm(0);                       /* 남은 알람을 끈다 */

    if (n == -1 && errno == EINTR) {
        printf("\n시간이 지났습니다.\n");
        return 1;
    }
```

```bash
( sleep 5 ) | ./timeout 2
```

```text
2초 안에 한 줄을 입력하십시오: 
시간이 지났습니다.
```

```bash
printf '안녕하세요\n' | ./timeout 3
```

```text
3초 안에 한 줄을 입력하십시오: 받았습니다: 안녕하세요
```

> **여기서는 `SA_RESTART`를 주면 안 됩니다.**
> 주면 `read`가 자동으로 다시 시작되어 **영원히 기다립니다.** 시간 제한이 아예 동작하지 않습니다. 제7절에서 "`SA_RESTART`가 편하다"고 했지만, **끊기는 것 자체가 목적일 때는 정반대**입니다.
{: .prompt-danger }

### 10.3 한계

| 문제 | 설명 |
|---|---|
| 초 단위뿐 | 더 정밀하게 하려면 `setitimer`·`timer_create` |
| 알람이 하나뿐 | 라이브러리와 충돌할 수 있다 |
| 경쟁 조건 | `alarm`과 `read` 사이에 시그널이 오면 놓친다 |

**실무에서는 `select`·`poll`의 시간 제한 기능을 씁니다.** 10강에서 배웁니다. `alarm`은 개념을 익히기에 좋고, 간단한 프로그램에는 여전히 쓸 만합니다.

---

## 제11절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| 처리기 안 `printf`가 이상하다 | **async-signal-safe 아님** | `write` 또는 깃발 방식 |
| 반복문이 안 끝난다 | 깃발에 `volatile` 없음 | `volatile sig_atomic_t` |
| 값이 이상하게 읽힌다 | `sig_atomic_t` 아님 | 자료형 바꾸기 |
| 좀비가 남는다 | 처리기에서 `waitpid` **한 번만** | **`while` 반복문 + `WNOHANG`** |
| `SIGCHLD` 처리기가 안 불린다 | 등록 전에 자식이 끝남 | **등록을 `fork`보다 먼저** |
| `read`가 자꾸 실패한다 | `EINTR` | 재시도 또는 `SA_RESTART` |
| 시간 제한이 안 걸린다 | `SA_RESTART`를 줌 | **그 시그널만 빼기**(10.2절) |
| `sleep`이 일찍 깬다 | 시그널에 끊김 | `nanosleep` 재시도(7.4절) |
| `kill -9`인데 정리가 안 됐다 | `SIGKILL`은 못 잡는다 | `SIGTERM`을 먼저 |
| 파이프에 쓰는데 죽는다 | `SIGPIPE` 기본 동작 | `SIG_IGN` 후 `EPIPE` 확인 |
| `exec` 한 프로그램이 시그널을 못 받는다 | 마스크가 물려짐 | `exec` 전에 되돌리기 |
| `errno`가 엉뚱하다 | 처리기가 덮어씀 | 처리기에서 저장·복원 |
| `signal()`이 시스템마다 다르다 | 옛 함수 | **`sigaction`** |

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
| 문제 1 | `SIGINT` 붙잡기 | 4 |
| 문제 2 | 안전한 처리기 만들기 | 5 |
| 문제 3 | 우아한 종료 | 6 |
| 문제 4 | `EINTR` 재현 | 7 |
| 문제 5 | **`SIGCHLD`로 좀비 없애기** | 8 |
| 문제 6 | 시그널 막기 | 9 |
| 문제 7 | 시간 제한 읽기 | 10 |
| 문제 8 | `SIGUSR1`으로 상태 보고 | 5 · 6 |
| 문제 9 | 미니 셸의 Ctrl+C 처리 | 4 · 4강 6절 |
| 문제 10 | `SIGPIPE` 실험 | 2.3 |

---

### 문제 1. `SIGINT` 붙잡기

Ctrl+C를 세 번 눌러야 끝나는 프로그램을 만드십시오.

**정답 및 해설**

4.3·4.4절의 코드가 답입니다.

```bash
./catch_int 3 &
CPID=$!
for i in 1 2 3; do kill -INT $CPID; sleep 0.4; done
wait $CPID ; echo $?
```

```text
0
```

- **`sigaction`을 쓰십시오.** `signal`은 판마다 동작이 다릅니다(4.1절).
- 처리기에서 `got++`만 한 것이 핵심입니다. 출력은 주 흐름에서 합니다.
- `pause()`로 기다린 것에 주목하십시오. `while (got < limit) ;`처럼 바쁘게 돌면 CPU를 100% 씁니다.
- **종료 상태가 0**인지 확인하십시오. 시그널로 죽었다면 4강 7.2절의 `WIFSIGNALED`가 참이 됩니다.

---

### 문제 2. 안전한 처리기 만들기

처리기에서 `printf` 대신 `write`로 메시지를 내보내십시오.

**정답 및 해설**

```c
static void on_int(int signo)
{
    static const char msg[] = "\n중단 요청을 받았습니다\n";

    (void) signo;
    write(STDERR_FILENO, msg, sizeof msg - 1);
}
```

- **`sizeof msg - 1`** 로 길이를 구했습니다. `sizeof`는 `'\0'`까지 세므로 1을 빼야 합니다. 1부 10강의 "크기를 아는 자가 안전하다"가 그대로 적용됩니다.
- `static const`로 둔 이유는 처리기가 불릴 때마다 스택에 만들지 않기 위해서입니다.
- **`printf`로 바꿔 보아도 대개는 잘 동작합니다.** 그것이 이 문제의 함정입니다. 문제가 드러나는 것은 **하필 주 흐름이 `printf` 도중일 때**뿐이며, 그런 일은 드물게 일어납니다. **드물게 일어나는 버그가 가장 잡기 어렵습니다.**
- 확인하려면 주 흐름에서 `printf`를 쉼 없이 돌리며 시그널을 반복해서 보내 보십시오.

---

### 문제 3. 우아한 종료

`SIGTERM`을 받으면 하던 일을 마무리하고 정리한 뒤 끝나는 프로그램을 만드십시오.

**정답 및 해설**

6.2절의 코드와 결과가 답입니다.

| 확인 | 방법 |
|---|---|
| 정리 메시지가 나오는가 | 출력 확인 |
| 파일이 온전한가 | `wc -l work.log` |
| 종료 상태가 0인가 | `echo $?` |

- **`kill -9`로도 시험해 보십시오.** 정리 메시지가 나오지 않고 종료 상태도 다릅니다. 이 대비가 `SIGKILL`을 잡을 수 없다는 사실을 실감하게 해 줍니다.
- `stop`에 시그널 번호를 넣어 둔 덕분에 **무엇 때문에 끝났는지** 로그에 남길 수 있습니다.
- 진짜 서버는 여기에 "새 요청은 안 받고, 처리 중인 것만 마친다"는 단계를 더합니다.

---

### 문제 4. `EINTR` 재현

`SA_RESTART`가 있을 때와 없을 때 `read`의 결과가 어떻게 달라지는지 확인하십시오.

**정답 및 해설**

7.2절의 결과가 답입니다.

| 설정 | 결과 |
|---|---|
| `SA_RESTART` 없음 | `-1`, `errno=4`(`EINTR`) |
| `SA_RESTART` 있음 | `1` 바이트 — 다시 시작됨 |

- **처리기가 파이프에 1바이트를 넣은 것**이 요령입니다. 그러지 않으면 `SA_RESTART` 쪽이 영원히 기다립니다.
- `errno=4`가 `EINTR`입니다. 숫자 대신 이름으로 비교해야 한다는 원칙(2.1절)을 지키십시오.
- 2강에서 만든 `write_full`·`read_full`의 `EINTR` 처리를 지우고 이 상황을 만들면, **자료가 잘리는 것**을 볼 수 있습니다.

---

### 문제 5. `SIGCHLD`로 좀비 없애기

`wait`을 부르지 않고도 좀비가 남지 않는 프로그램을 만드십시오.

**정답 및 해설**

8.1·8.2절의 코드와 결과가 답입니다.

```text
자식 5개를 만들었습니다. wait 을 부르지 않고 기다립니다.
처리기가 거둔 자식 수: 5
남은 좀비: 0개
```

- **`while (waitpid(-1, NULL, WNOHANG) > 0)`** 의 세 요소가 모두 필요합니다(8.1절).
- **`while`을 `if`로 바꿔 보십시오.** 자식이 동시에 끝나는 경우 좀비가 남습니다. 자식들의 `usleep` 간격을 없애면 재현하기 쉽습니다.
- 이 프로그램에서 `sleep` 대신 `nanosleep` 재시도(`sleep_full`)를 쓴 이유를 설명할 수 있어야 합니다. `sleep`은 첫 `SIGCHLD`에 깨어난 뒤 **초 단위 절단 때문에 남은 시간을 잃어** 일찍 끝나 버립니다(7.4절).
- 종료 상태가 필요 없다면 `SIG_IGN`이 더 간단합니다(8.3절).

---

### 문제 6. 시그널 막기

임계 구역에서 `SIGINT`를 막고, 막힌 동안 도착한 시그널이 나중에 배달되는지 확인하십시오.

**정답 및 해설**

9.2절의 코드와 결과가 답입니다.

- **`sigpending`으로 대기 중인지 확인한 것**이 핵심입니다. 막기와 무시의 차이를 눈으로 보여 줍니다.
- 원래 마스크를 `old`에 받아 두었다가 `SIG_SETMASK`로 되돌린 것에 주목하십시오. `SIG_UNBLOCK`을 쓰면 **원래 막혀 있던 것까지 풀어** 버릴 수 있습니다.
- 막혀 있는 동안 `SIGINT`를 **여러 번** 보내 보십시오. 풀었을 때 `count`는 여전히 1입니다(9.3절).

---

### 문제 7. 시간 제한 읽기

주어진 시간 안에 입력이 없으면 포기하는 프로그램을 만드십시오.

**정답 및 해설**

10.2절의 코드와 결과가 답입니다.

| 시험 | 결과 |
|---|---|
| `( sleep 5 ) \| ./timeout 2` | 시간 초과, 종료 상태 1 |
| `printf '안녕하세요\n' \| ./timeout 3` | 정상 수신, 종료 상태 0 |
| `./timeout 2 < /dev/null` | **EOF**(`read`가 0을 돌려줌) |

- **`/dev/null`로 시험하면 시간 초과가 재현되지 않습니다.** 곧바로 EOF가 되기 때문입니다. `sleep`을 파이프로 물려 **막히는 입력**을 만들어야 합니다. 시험 설계가 잘못되면 검증이 무의미해지는 좋은 예입니다.
- `alarm(0)`으로 남은 알람을 끄는 것을 잊지 마십시오. 그러지 않으면 나중에 엉뚱한 곳에서 `SIGALRM`을 맞습니다.
- 세 가지 결과(`-1`+`EINTR`, `0`, 양수)를 모두 구분해 처리했는지 확인하십시오. 2강 3.1절의 반환값 규약 그대로입니다.

---

### 문제 8. `SIGUSR1`으로 상태 보고

`SIGUSR1`을 받으면 지금까지의 진행 상황을 출력하고, 계속 일하는 프로그램을 만드십시오.

**정답 및 해설**

```c
static volatile sig_atomic_t report = 0;
static volatile sig_atomic_t stop   = 0;

static void on_usr1(int signo) { (void) signo; report = 1; }
static void on_term(int signo) { stop = signo; }
```

```c
    while (!stop) {
        do_one_unit_of_work();
        done++;

        if (report) {               /* 출력은 주 흐름에서 */
            report = 0;
            printf("진행 상황: %ld건 처리\n", done);
            fflush(stdout);
        }
    }
```

```bash
./worker &
WPID=$!
sleep 0.6; kill -USR1 $WPID
sleep 0.6; kill -USR1 $WPID
sleep 0.4; kill -TERM $WPID
wait $WPID ; echo "종료 상태: $?"
```

```text
PID 396 — SIGUSR1 로 상태를 물어보십시오
진행 상황: 7건 처리
진행 상황: 14건 처리
시그널 15 로 종료. 모두 19건 처리했습니다
종료 상태: 0
```

- **`SIGUSR1`·`SIGUSR2`는 우리가 뜻을 정해 쓰라고 비워 둔 시그널**입니다(2.2절).
- 처리기는 `report = 1`만 하고 출력은 주 흐름이 합니다. 5.3절의 원칙 그대로입니다.
- 실무에서 널리 쓰이는 방식입니다. `nginx`는 `SIGUSR1`로 로그 파일을 다시 열고, `SIGHUP`으로 설정을 다시 읽습니다. **프로그램을 멈추지 않고 조작하는 방법**입니다.
- 깃발을 `0`으로 되돌리는 위치에 주의하십시오. 출력 **전에** 내려야 그사이 온 시그널을 놓치지 않습니다.

---

### 문제 9. 미니 셸의 Ctrl+C 처리

4강의 미니 셸에서 Ctrl+C를 누르면 **셸은 살아 있고 실행 중인 명령만** 죽게 하십시오.

**정답 및 해설**

**셸 자신은 `SIGINT`를 무시하고, 자식은 기본 동작으로 되돌립니다.**

```c
    /* 셸 자신은 무시 */
    memset(&sa, 0, sizeof sa);
    sa.sa_handler = SIG_IGN;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGINT, &sa, NULL);
    sigaction(SIGQUIT, &sa, NULL);
```

```c
        if (pid == 0) {
            struct sigaction dfl;

            /* 자식은 기본 동작으로 되돌린다 — exec 이 처리기는 되돌리지만
               SIG_IGN 은 그대로 물려받으므로 직접 풀어야 한다 */
            memset(&dfl, 0, sizeof dfl);
            dfl.sa_handler = SIG_DFL;
            sigemptyset(&dfl.sa_mask);
            dfl.sa_flags = 0;
            sigaction(SIGINT, &dfl, NULL);
            sigaction(SIGQUIT, &dfl, NULL);

            execvp(args[0], args);
            ...
        }
```

> **`exec`은 처리기를 기본 동작으로 되돌리지만, `SIG_IGN`은 그대로 물려줍니다**(9.4절). 그래서 자식에서 **직접 `SIG_DFL`로 되돌려야** 합니다. 이것을 빠뜨리면 자식도 Ctrl+C에 반응하지 않습니다.
{: .prompt-danger }

실제로 확인한 결과입니다.

```text
minish$ echo 시작
시작
minish$ sleep 30
[시그널 2(Interrupt) 로 종료]
minish$ echo 셸은 살아 있습니다
셸은 살아 있습니다
minish$ exit
minish 를 마칩니다.
```

- **`sleep 30`만 죽고 셸은 살아남았습니다.** `WIFSIGNALED`가 참이라 시그널 번호 2를 보고했습니다(4강 7.2절).
- 진짜 셸은 **프로세스 그룹**을 만들어 터미널의 전면 그룹을 바꿉니다(`setpgid`·`tcsetpgrp`). 그래야 배경 작업이 Ctrl+C의 영향을 받지 않습니다. 여기서는 거기까지 가지 않습니다.

---

### 문제 10. `SIGPIPE` 실험

받는 쪽이 사라진 파이프에 쓰면 어떻게 되는지 확인하고, 죽지 않게 고치십시오.

**정답 및 해설**

```bash
yes | head -1
```

`head`가 한 줄만 읽고 끝나면 `yes`는 `SIGPIPE`로 죽습니다. 직접 만들어 확인해 보십시오.

```c
    int fd[2];
    pipe(fd);
    close(fd[0]);                   /* 읽는 쪽을 닫아 버린다 */

    if (ignore) {                   /* 무시하도록 설정하면 */
        sa.sa_handler = SIG_IGN;
        sigaction(SIGPIPE, &sa, NULL);
    }

    errno = 0;
    n = write(fd[1], "hello", 5);
    if (n == -1)
        printf("write 실패: %s (errno=%d)\n", strerror(errno), errno);
    else
        printf("write 성공: %zd\n", n);
```

```bash
./pipetest ; echo "종료 상태: $?"
```

```text
SIGPIPE 기본 설정으로 씁니다
종료 상태: 141
```

```bash
./pipetest ignore ; echo "종료 상태: $?"
```

```text
SIGPIPE 무시 설정으로 씁니다
write 실패: Broken pipe (errno=32)
종료 상태: 0
```

| 설정 | 결과 |
|---|---|
| 기본 | **프로그램이 `SIGPIPE`로 죽는다** — `write` 결과 줄에 도달하지 못한다 |
| `SIG_IGN` | `write`가 `-1`과 **`EPIPE`(32)** 를 돌려준다 |

- **종료 상태 141에 주목하십시오.** 4강 7.4절의 표대로 **128 + 13 = 141**, 즉 `SIGPIPE`(13)로 죽었다는 셸의 표기입니다. 출력 한 줄이 통째로 사라진 것이 "정리할 기회를 못 얻는다"는 말의 뜻입니다.

- **서버에서는 거의 언제나 `SIG_IGN`이 옳습니다.** 클라이언트가 먼저 끊는 일은 늘 일어나며, 그때마다 서버가 죽으면 안 됩니다.
- 죽었는지 확인하려면 4강 7.2절의 방법을 쓰십시오. `WIFSIGNALED`가 참이고 `WTERMSIG`가 13(`SIGPIPE`)입니다.
- 9강에서 소켓을 다룰 때 이 문제를 다시 만납니다. 소켓에도 똑같이 적용됩니다.

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 소스 — `signums.c`, `catch_int.c`, `eintr_demo.c`, `sigchld.c`, `graceful.c`, `block_demo.c`, `timeout.c`, `worker.c`, `pipetest.c` |
| 2 | 이 시스템의 시그널 번호표(문제 1 전) |
| 3 | **`EINTR` 두 화면** — `SA_RESTART` 있을 때와 없을 때(문제 4) |
| 4 | **좀비 비교** — `SIGCHLD` 처리기 있을 때 0개, 없을 때 5개(문제 5) |
| 5 | 우아한 종료 화면과 `work.log` 줄 수(문제 3) |
| 6 | 시그널 막기 — `sigpending` 확인 화면(문제 6) |
| 7 | 시간 제한 — 초과·정상 두 경우(문제 7) |
| 8 | 미니 셸의 Ctrl+C 동작 화면(문제 9) |
| 9 | 짧은 서술 ① 처리기에서 `printf`를 쓰면 안 되는 이유 |
| 10 | 짧은 서술 ② 막기(block)와 무시(ignore)의 차이 |

---

## 정리

| 구분 | 핵심 |
|---|---|
| 시그널 | **비동기·정보 없음·쌓이지 않음** |
| 못 잡는 것 | **`SIGKILL`(9)·`SIGSTOP`(19)** |
| 등록 | **`sigaction`**. `signal`은 쓰지 않는다 |
| 처리기 | **깃발만 세운다.** `volatile sig_atomic_t` |
| 안전한 함수 | `write`·`_exit`·`waitpid`. **`printf`는 안 된다** |
| `errno` | 처리기에서 **저장하고 복원** |
| 우아한 종료 | 깃발 → 반복문 탈출 → **주 흐름에서 정리** |
| `EINTR` | 오류가 아니다. **재시도**하거나 `SA_RESTART` |
| `SA_RESTART`의 한계 | `sleep`·`select`·`poll`은 **다시 시작되지 않는다** |
| 좀비 | **`SIGCHLD` + `while` + `WNOHANG`** |
| `SIG_IGN` | 기본 무시와 **다르다** — 커널이 자동으로 거둔다 |
| 막기 | 대기했다가 **풀면 배달**. 무시는 버려진다 |
| 시간 제한 | `alarm` + **`SA_RESTART` 없이** |

---

## 다음 강의 예고

**6강 「파이프와 프로세스 간 통신」**(APUE 15·17장)에서는 프로세스끼리 **자료를 주고받습니다.**

- `pipe`로 프로세스 사이에 관을 놓는다
- **`dup2`로 재지정을 구현한다** — 4강 3.3절의 서술자 공유가 여기서 쓰인다
- **미니 셸에 `>`·`<`·`|`를 넣어 쓸 만한 셸로 만든다**
- FIFO — 이름이 있는 파이프(3강 3.1절의 `S_ISFIFO`)
- 공유 메모리와 `mmap`

2강 9.1절에서 "커널은 가장 작은 빈 번호를 준다"고 한 규칙이 **재지정의 열쇠**로 쓰입니다.

---

## 부록 A. 이번 강의 명령·함수 요약

| 하려는 일 | 함수 / 명령 |
|---|---|
| 처리기 등록 | `sigaction(signo, &sa, NULL)` |
| 시그널 보내기 | `kill(pid, sig)` · `raise(sig)` |
| 명령으로 보내기 | `kill -TERM PID` · `pkill -f 무늬` |
| 살아 있는지 확인 | `kill -0 PID` |
| 시그널 목록 | `kill -l` |
| 이름 얻기 | `strsignal(signo)` |
| 시그널 기다리기 | `pause()` |
| 집합 만들기 | `sigemptyset` · `sigaddset` · `sigfillset` |
| 막기·풀기 | `sigprocmask(SIG_BLOCK/SIG_SETMASK, ...)` |
| 대기 중 확인 | `sigpending(&set)` · `sigismember` |
| 알람 | `alarm(sec)` · `alarm(0)`으로 취소 |
| 정확한 잠자기 | `nanosleep(&req, &rem)` |
| 자식 거두기(처리기) | `while (waitpid(-1, NULL, WNOHANG) > 0)` |
| 좀비 자동 방지 | `SIGCHLD`를 `SIG_IGN` 또는 `SA_NOCLDWAIT` |

## 부록 B. 표준 문서와 출처

**시스템 호출과 개념**

| 항목 | 리눅스 man | POSIX 표준 |
|---|---|---|
| `sigaction` | [`sigaction(2)`](https://man7.org/linux/man-pages/man2/sigaction.2.html) | [`sigaction()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/sigaction.html) |
| `kill` | [`kill(2)`](https://man7.org/linux/man-pages/man2/kill.2.html) | [`kill()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/kill.html) |
| `sigprocmask` | [`sigprocmask(2)`](https://man7.org/linux/man-pages/man2/sigprocmask.2.html) | [`sigprocmask()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/sigprocmask.html) |
| `alarm` | [`alarm(2)`](https://man7.org/linux/man-pages/man2/alarm.2.html) | [`alarm()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/alarm.html) |
| `nanosleep` | [`nanosleep(2)`](https://man7.org/linux/man-pages/man2/nanosleep.2.html) | [`nanosleep()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/nanosleep.html) |
| `pause` | [`pause(2)`](https://man7.org/linux/man-pages/man2/pause.2.html) | [`pause()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/pause.html) |
| `signal`(권장하지 않음) | [`signal(2)`](https://man7.org/linux/man-pages/man2/signal.2.html) | — |
| **시그널 전반·기본 동작** | [`signal(7)`](https://man7.org/linux/man-pages/man7/signal.7.html) | — |
| **처리기에서 쓸 수 있는 함수** | [`signal-safety(7)`](https://man7.org/linux/man-pages/man7/signal-safety.7.html) | — |

**본문의 주장과 근거**

| 주장 | 근거 |
|---|---|
| 이 시스템의 시그널 번호(2.1절) | `signums` 실행 + `kill -l` |
| `printf`는 처리기에서 안전하지 않다(5.1절) | `signal-safety(7)` 원문 인용 — stdio 전체가 비안전, `write`·`_exit`·`waitpid`는 안전 목록에 있음 |
| **`SA_RESTART` 유무로 `read` 결과가 달라진다**(7.2절) | `eintr_demo` 실행 — 없으면 `-1`/`errno=4`, 있으면 1바이트 |
| **`sleep`은 `SA_RESTART`로도 다시 시작되지 않는다**(7.4절) | `sleep(5)`를 1초 뒤 끊으니 반환값 **4** |
| **`SIGCHLD` 처리기로 좀비 0개**(8.2절) | `sigchld 5` → 거둔 자식 5, 좀비 0 (3회 반복 동일) |
| 처리기가 없으면 좀비 5개(8.2절) | 같은 구조에서 처리기만 뺀 `nochld` 실행 |
| 막힌 시그널은 대기했다가 배달된다(9.2절) | `block_demo` — 막힌 동안 `count` 0, `sigpending` 참, 푼 뒤 `count` 1 |
| 우아한 종료가 동작한다(6.2절) | `graceful`에 `SIGTERM` → 정리 메시지 출력, `work.log` 6줄, 종료 상태 0 |
| 시간 제한이 동작한다(10.2절) | `( sleep 5 ) \| ./timeout 2` → 시간 초과, 종료 상태 1 |
| `SIGPIPE` 기본은 죽고, `SIG_IGN`이면 `EPIPE`(문제 10) | `pipetest` 종료 상태 **141**(=128+13) vs `EPIPE` 32 후 정상 종료 |
| `SIGUSR1` 상태 보고가 동작한다(문제 8) | `worker`에 `SIGUSR1` 2회 → 7건·14건 보고, `SIGTERM`에 19건으로 정상 종료 |
| 미니 셸에서 자식만 Ctrl+C로 죽는다(문제 9) | `sleep 30`이 시그널 2로 종료, 셸은 다음 명령을 계속 받음 |

> 값은 모두 **Ubuntu 24.04에서 실제로 실행한 결과**입니다. PID와 처리 건수는 실행마다 달라지므로 **관계와 대비**를 보십시오. `sigchld`의 좀비 0개는 3회 반복해 같은 결과를 확인했습니다.
{: .prompt-tip }

## 부록 C. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `signums.c` | 시그널 번호와 뜻 | 2 |
| `catch_int.c` | `sigaction`으로 `SIGINT` 붙잡기 | 4 |
| `eintr_demo.c` | **`SA_RESTART`와 `EINTR`** | 7 |
| `sigchld.c` | **좀비 자동 수거** | 8 |
| `graceful.c` | 우아한 종료 | 6 |
| `block_demo.c` | 시그널 막기와 `sigpending` | 9 |
| `timeout.c` | `alarm`으로 시간 제한 | 10 |
| `worker.c` | `SIGUSR1` 상태 보고 | 문제 8 |
| `pipetest.c` | `SIGPIPE` 실험 | 문제 10 |
