---
title: 중급 C 프로그래밍 4강 - 프로세스 fork exec wait
date: 2026-12-14 09:00:00 +0900
categories:
  - 1.응용강의
  - 중급 C 프로그래밍
tags:
  - C언어
  - 시스템프로그래밍
  - 리눅스
  - 프로세스
  - fork
  - exec
  - wait
  - 좀비프로세스
  - 셸
pin:
mermaid: false
---

> **학습 목표**
> 1. 프로그램과 프로세스를 구분해 설명할 수 있다.
> 2. **`fork`가 한 번 불리고 두 번 돌아온다**는 것을 설명하고 반환값으로 갈래를 나눌 수 있다.
> 3. 부모와 자식이 **무엇을 복사하고 무엇을 공유하는지** 실험으로 보일 수 있다.
> 4. `fork`가 **출력 버퍼까지 복제**한다는 함정을 설명하고 피할 수 있다.
> 5. `exec` 계열 함수로 다른 프로그램이 될 수 있다.
> 6. **`fork` + `exec` + `wait`** 로 셸의 동작을 재현할 수 있다.
> 7. 종료 상태를 `WIFEXITED`·`WEXITSTATUS`·`WIFSIGNALED`로 해석할 수 있다.
> 8. **좀비**와 **고아** 프로세스가 무엇인지 설명하고 관찰할 수 있다.
> 9. `exit`와 `_exit`의 차이를 설명하고 알맞게 쓸 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

지금까지는 **프로그램 하나가 혼자** 돌았습니다. 이번 강의부터 달라집니다.

| 지금까지 | 이번 강의 |
|---|---|
| 함수를 부른다 | **프로세스를 만든다** |
| 하나의 실행 흐름 | **여러 개의 프로세스** |
| 파일을 다룬다 | **프로그램을 다룬다** |

1강 8.6절에서 이런 출력을 보았습니다.

```text
프로세스 번호(PID)  : 2841
부모 프로세스(PPID) : 2103
```

**"우리 프로그램에게 부모가 있다"** 는 사실만 확인하고 넘어갔습니다. 이번 강의에서 그 부모가 무엇을 했는지, 그리고 우리가 직접 부모가 되는 방법을 배웁니다.

이 강의를 마치면 **셸이 명령을 실행하는 원리**를 알게 되고, 작은 셸을 직접 만들게 됩니다.

**기준 교재: APUE 3판 8장(Process Control)·9장(Process Relationships).**

이 강의는 **3회차 분량**(모두 합쳐 약 490분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제4절 | `fork`와 메모리 | 145분 |
| **2회차** | 제5절 ~ 제8절 | `exec`·셸·`wait`·좀비 | 165분 |
| **3회차** | 제9절 ~ 실습문제 | 고아·종료 방법과 실습 | 180분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 프로그램과 프로세스 | 25분 |
| 제2절 | `fork` — 한 번 부르고 두 번 돌아온다 | 40분 |
| 제3절 | 무엇이 복사되고 무엇이 공유되나 | 50분 |
| 제4절 | `fork`가 복제하는 뜻밖의 것 — 버퍼 | 30분 |
| 제5절 | `exec` — 다른 프로그램이 되기 | 40분 |
| 제6절 | `fork` + `exec` = 셸 | 45분 |
| 제7절 | `wait`과 종료 상태 | 45분 |
| 제8절 | 좀비 | 30분 |
| 제9절 | 고아와 subreaper | 30분 |
| 제10절 | 끝내는 법 — `exit`와 `_exit` | 25분 |
| 제11절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 디렉터리**는 `~/cmid/lab04`를 사용합니다. 이번 강의는 **VM 한 대**(`c-srv`)에서 진행합니다.

```bash
mkdir -p ~/cmid/lab04 && cd ~/cmid/lab04
```

---

## 제1절. 프로그램과 프로세스

### 1.1 둘은 다릅니다

| 구분 | 프로그램 | 프로세스 |
|---|---|---|
| 무엇 | **디스크에 있는 파일** | **메모리에서 실행 중인 것** |
| 개수 | 하나 | 같은 프로그램으로 **여러 개** |
| 상태 | 없음 | PID·메모리·열린 파일·현재 위치 |

`/bin/ls`는 프로그램이고, `ls`를 실행할 때마다 새 프로세스가 생깁니다. 같은 프로그램으로 열 개의 프로세스를 동시에 돌릴 수 있으며, **각자 독립된 메모리**를 가집니다.

### 1.2 프로세스가 가진 것

| 항목 | 확인 방법 |
|---|---|
| PID·PPID | `getpid()`·`getppid()` |
| 메모리 공간 | `/proc/PID/maps` |
| 열린 파일 서술자 | `/proc/PID/fd` (2강 9.2절) |
| 현재 작업 디렉터리 | `/proc/PID/cwd` |
| 환경 변수 | `/proc/PID/environ` |
| 실행 파일 | `/proc/PID/exe` |

```bash
ls -l /proc/self/exe
```

```text
lrwxrwxrwx 1 student student 0 Dec 14 09:10 /proc/self/exe -> /usr/bin/ls
```

**`/proc/self`는 지금 보고 있는 프로세스 자신**을 가리킵니다. 위 결과가 `ls`인 이유는 `ls` 명령이 자기 자신을 본 것이기 때문입니다.

전체 목록은 [`proc(5)`](https://man7.org/linux/man-pages/man5/proc.5.html)에 있습니다.

### 1.3 프로세스는 나무를 이룹니다

```bash
ps -o pid,ppid,comm --forest -e | head -12
```

```text
    PID    PPID COMMAND
      1       0 systemd
     39       1 systemd-journal
     89       1 systemd-udevd
    109      89  \_ (udev-worker)
    274       2 login
    330     274  \_ bash
```

**모든 프로세스에는 부모가 있고**, 뿌리는 PID 1입니다. 이 나무가 어떻게 자라는지가 이번 강의의 주제입니다.

---

## 제2절. `fork` — 한 번 부르고 두 번 돌아온다

### 2.1 세상에서 가장 이상한 함수

```c
#include <unistd.h>

pid_t fork(void);
```

> **`fork`는 한 번 불리지만 두 번 돌아옵니다.**
> 부모에게 한 번, **새로 생긴 자식에게 한 번**.

| 어디로 돌아오나 | 반환값 |
|---|---|
| **부모** | **자식의 PID**(양수) |
| **자식** | **0** |
| 실패 | `-1`(부모에게만) |

**반환값이 다르다는 것이 유일한 구분 수단**입니다. 그 밖의 모든 것은 똑같습니다.

### 2.2 확인해 보기

```c
    pid = fork();

    if (pid == -1) {
        fprintf(stderr, "fork: %s\n", strerror(errno));
        return 1;
    }

    if (pid == 0) {
        /* 자식에게는 0 이 돌아온다 */
        printf("[자식] PID %d, 부모 %d, fork 반환값 %d\n",
               (int) getpid(), (int) getppid(), (int) pid);
    } else {
        /* 부모에게는 자식의 PID 가 돌아온다 */
        printf("[부모] PID %d, 부모 %d, fork 반환값 %d\n",
               (int) getpid(), (int) getppid(), (int) pid);
    }

    printf("이 줄은 둘 다 실행합니다 (PID %d)\n", (int) getpid());
```

```bash
gcc -Wall -Wextra -std=gnu17 fork1.c -o fork1
```

```bash
./fork1
```

```text
갈라지기 전 : PID 435, 부모 433
[부모] PID 435, 부모 433, fork 반환값 436
이 줄은 둘 다 실행합니다 (PID 435)
[자식] PID 436, 부모 435, fork 반환값 0
이 줄은 둘 다 실행합니다 (PID 436)
```

**세 가지를 확인하십시오.**

| 관찰 | 뜻 |
|---|---|
| 부모의 반환값 **436** = 자식의 PID | 부모는 자식을 알아본다 |
| 자식의 반환값 **0**, 자식의 PPID **435** = 부모의 PID | 자식은 자기 부모를 안다 |
| 마지막 줄이 **두 번** 나왔다 | `fork` 이후 코드는 **둘 다** 실행한다 |

### 2.3 자식은 `fork` 다음 줄부터 시작합니다

**자식은 프로그램의 처음부터 시작하지 않습니다.** `fork`가 돌아온 바로 그 지점부터 이어서 실행합니다. 부모가 지금까지 만든 변수 값, 열어 둔 파일, 현재 디렉터리를 모두 그대로 물려받은 상태입니다.

### 2.4 누가 먼저 실행되는가

> **정해져 있지 않습니다.**
> 위 출력에서는 부모가 먼저 나왔지만, 다음 실행에서는 자식이 먼저일 수 있습니다. 어느 쪽을 먼저 돌릴지는 **스케줄러가 정합니다.**
{: .prompt-danger }

**실행 순서에 의존하는 코드를 쓰면 안 됩니다.** 순서가 필요하면 `wait`(제7절)이나 파이프(6강)로 **명시적으로** 맞춰야 합니다. 여러 번 실행해 보면 순서가 바뀌는 것을 볼 수 있습니다.

```bash
for i in 1 2 3 4 5; do ./fork1 | head -2 | tail -1; done
```

---

## 제3절. 무엇이 복사되고 무엇이 공유되나

**이 절이 이번 강의에서 가장 중요합니다.**

### 3.1 실험

`fork_share.c`는 전역·지역·힙 변수와 열린 파일을 두고 갈라진 뒤, 자식이 모두 바꿉니다.

```c
    if (pid == 0) {
        /* 자식이 값을 바꾸고 파일에 쓴다 */
        global = 111;
        local  = 222;
        *heap  = 333;
        write(fd, "AAAAA", 5);
        pos = lseek(fd, 0, SEEK_CUR);

        printf("[자식] global=%d local=%d heap=%d  파일 위치=%lld\n",
               global, local, *heap, (long long) pos);
        fflush(stdout);             /* _exit 은 버퍼를 비우지 않는다 — 제10절 */
        free(heap);
        close(fd);
        _exit(0);
    }

    /* 부모는 자식이 끝나기를 기다린 뒤 확인한다 */
    wait(NULL);

    pos = lseek(fd, 0, SEEK_CUR);
    printf("[부모] global=%d local=%d heap=%d  파일 위치=%lld\n",
           global, local, *heap, (long long) pos);

    write(fd, "BBBBB", 5);
```

```bash
./fork_share
```

```text
[자식] global=111 local=222 heap=333  파일 위치=5
[부모] global=100 local=200 heap=300  파일 위치=5
--- shared.txt 의 내용 ---
AAAAABBBBB
```

### 3.2 결과 해석

| 항목 | 자식이 바꾼 뒤 부모는 | 결론 |
|---|---|---|
| 전역 변수 | 100 (그대로) | **복사됨** |
| 지역 변수 | 200 (그대로) | **복사됨** |
| 힙 | 300 (그대로) | **복사됨** |
| **파일 위치** | **5 (자식이 쓴 만큼 옮겨졌다)** | **공유됨** |

**메모리는 복사되고 파일 위치는 공유됩니다.** 이 비대칭이 핵심입니다.

파일 내용도 확인하십시오. `AAAAABBBBB` — 부모가 나중에 쓴 `BBBBB`가 **덮어쓰지 않고 이어졌습니다.** 부모가 `lseek`를 하지 않았는데도 위치가 5였기 때문입니다.

### 3.3 왜 그런가 — 2강의 세 층 그림

2강 9.3절에서 본 그림이 여기서 쓰입니다.

```text
 부모                        열린 파일 설명            아이노드
 ┌───────────┐
 │ 3 ────────┼──┐
 └───────────┘  ├──────────▶ [위치 5, O_RDWR] ──────▶ shared.txt
 자식           │             ↑
 ┌───────────┐  │             └── 둘이 같은 것을 가리킨다
 │ 3 ────────┼──┘
 └───────────┘
```

`fork`는 **서술자 표를 복사**하지만, 그 표가 가리키는 **열린 파일 설명은 하나**입니다. 그래서 위치가 공유됩니다.

| 층 | `fork` 이후 |
|---|---|
| 서술자 표(번호 → 설명) | **복사된다** |
| 열린 파일 설명(위치·플래그) | **공유된다** |
| 아이노드 | 원래부터 하나 |

> **이 성질이 셸의 재지정을 가능하게 합니다.**
> `ls > out.txt`에서 셸이 자식의 서술자 1을 파일로 바꾸어 주고, 여러 프로그램이 같은 파일에 이어 쓸 수 있는 것이 모두 이 구조 덕분입니다. 6강에서 직접 구현합니다.
{: .prompt-tip }

### 3.4 그래서 복사가 느리지 않은가 — 쓸 때 복사

메모리를 통째로 복사하면 큰 프로그램에서 `fork`가 매우 느릴 것입니다. 실제로는 그렇지 않습니다.

> **쓸 때 복사(copy-on-write)**
> 커널은 처음에 **같은 물리 메모리를 함께 쓰게** 하고 읽기 전용으로 표시해 둡니다. 어느 한쪽이 **쓰려고 할 때** 그 페이지만 복사합니다.

| 결과 | |
|---|---|
| `fork` 자체는 빠르다 | 페이지 표만 만든다 |
| 읽기만 하면 복사가 없다 | 메모리도 아낀다 |
| **프로그램에서는 완전히 복사된 것처럼 보인다** | 위 실험 결과 그대로 |

`fork` 직후 곧바로 `exec`을 하는 흔한 경우에는 거의 아무것도 복사되지 않습니다.

---

## 제4절. `fork`가 복제하는 뜻밖의 것 — 버퍼

### 4.1 고전적인 함정

```c
/* fork_buffer.c — fork 는 버퍼까지 복제한다 */
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    printf("이 줄은 몇 번 나올까요?\n");   /* 개행이 있는데도 위험하다 */

    if (fork() == 0) {
        /* 자식은 아무것도 더 하지 않는다 */
    } else {
        wait(NULL);
    }

    return 0;                              /* 둘 다 여기서 버퍼를 비운다 */
}
```

**한 번만 출력될 것 같습니다.** 확인해 보십시오.

```bash
./fork_buffer
```

```text
이 줄은 몇 번 나올까요?
```

```bash
./fork_buffer > fb.txt ; cat fb.txt
```

```text
이 줄은 몇 번 나올까요?
이 줄은 몇 번 나올까요?
```

**재지정하면 두 번 나옵니다.**

### 4.2 왜 그런가

1강 11.3절과 2강 1.3절에서 배운 버퍼링이 원인입니다.

| 출력 대상 | 버퍼링 | `fork` 시점에 버퍼는 | 결과 |
|---|---|---|---|
| 터미널 | 줄 단위 | **이미 비어 있다** | 한 번 |
| 파일·파이프 | 블록 단위 | **아직 내용이 남아 있다** | **두 번** |

`fork`는 메모리를 복제하고, **`stdout`의 버퍼도 메모리입니다.** 부모와 자식이 각각 자기 버퍼를 비우면서 같은 내용이 두 번 나갑니다.

> **화면에서 시험할 때는 멀쩡하다가 파일로 재지정하는 순간 드러납니다.** 로그를 파일로 남기기 시작한 뒤에야 중복이 발견되는 전형적인 사고입니다.
{: .prompt-danger }

### 4.3 규칙

> **`fork` 하기 전에 `fflush`.**

```c
printf("갈라지기 전\n");
fflush(stdout);          /* 반드시 */
pid = fork();
```

| 방법 | 설명 |
|---|---|
| `fflush(stdout)` | 갈라지기 전에 비운다. **가장 확실** |
| `fflush(NULL)` | 열린 모든 스트림을 비운다 |
| `write`를 쓴다 | 버퍼를 거치지 않는다(1강 8절) |
| 자식에서 `_exit` | 버퍼를 비우지 않고 끝낸다(제10절) |

`fork1.c`에서 `fork` 직전에 `fflush(stdout)`을 넣은 것이 바로 이 때문입니다.

---

> **▶ 여기서부터 2회차 — `exec`·셸·`wait`·좀비**
> 제5절 ~ 제8절, 약 165분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제5절. `exec` — 다른 프로그램이 되기

### 5.1 프로세스는 그대로, 내용물만 바뀐다

`fork`가 **프로세스를 만드는** 일이라면, `exec`은 **지금 프로세스를 다른 프로그램으로 바꾸는** 일입니다.

| | `fork` | `exec` |
|---|---|---|
| 프로세스 개수 | **하나 늘어난다** | 그대로 |
| PID | 새로 생긴다 | **바뀌지 않는다** |
| 메모리 내용 | 복제된다 | **통째로 교체된다** |
| 돌아오는가 | 두 번 | **성공하면 돌아오지 않는다** |

> **`exec`이 성공하면 그다음 줄은 절대 실행되지 않습니다.**
> 그 코드가 메모리에서 사라졌기 때문입니다. 따라서 **`exec` 다음 줄은 곧 "실패했다"는 뜻**입니다.
{: .prompt-warning }

```c
execvp(argv[1], &argv[1]);
/* 여기에 도달했다면 exec 이 실패한 것이다 */
fprintf(stderr, "%s: %s\n", argv[1], strerror(errno));
_exit(127);
```

### 5.2 여섯 가지 이름

이름이 여섯 가지지만 규칙이 있습니다.

| 이름 | `l` / `v` | `p` | `e` |
|---|---|---|---|
| `execl` | 인자를 **나열**(list) | | |
| `execv` | 인자를 **배열**(vector) | | |
| `execlp` | 나열 | **PATH 검색** | |
| `execvp` | 배열 | **PATH 검색** | |
| `execle` | 나열 | | **환경 변수 지정** |
| `execve` | 배열 | | 환경 지정 — **진짜 시스템 호출** |

| 글자 | 뜻 |
|---|---|
| `l` | 인자를 `"ls", "-l", NULL`처럼 하나씩 나열 |
| `v` | 인자를 `char *argv[]` 배열로 |
| `p` | `PATH` 환경 변수에서 프로그램을 찾는다 |
| `e` | 환경 변수를 직접 넘긴다 |

**실제 시스템 호출은 [`execve(2)`](https://man7.org/linux/man-pages/man2/execve.2.html) 하나뿐**이고, 나머지 다섯은 라이브러리 함수입니다([`exec(3)`](https://man7.org/linux/man-pages/man3/exec.3.html)).

### 5.3 쓰는 법

```c
execl("/bin/ls", "ls", "-l", NULL);          /* 경로를 직접 */
execlp("ls", "ls", "-l", NULL);              /* PATH 에서 찾기 */

char *args[] = { "ls", "-l", NULL };
execvp("ls", args);                          /* 배열로 */
```

> **첫 번째 인자와 `argv[0]`이 따로인 것에 주의하십시오.**
> `execlp("ls", "ls", "-l", NULL)`에서 앞의 `"ls"`는 **찾을 프로그램**이고, 뒤의 `"ls"`는 그 프로그램이 받을 **`argv[0]`** 입니다. 같은 값을 두 번 쓰는 것이 관례이지만, 다르게 줄 수도 있습니다. 실제로 어떤 프로그램은 `argv[0]`을 보고 동작을 바꿉니다.
{: .prompt-tip }

**인자 목록은 반드시 `NULL`로 끝나야 합니다.** 빠뜨리면 커널이 어디까지가 인자인지 몰라 미정의 동작이 됩니다.

### 5.4 무엇이 남고 무엇이 사라지나

`exec` 후에도 살아남는 것들이 있습니다.

| 살아남는 것 | 사라지는 것 |
|---|---|
| **PID·PPID** | 메모리 내용(코드·자료·힙·스택) |
| **열린 파일 서술자** | 열려 있던 `FILE *` 버퍼 |
| 현재 작업 디렉터리 | 시그널 처리기(5강) |
| umask·자원 한계 | |
| 환경 변수(기본적으로) | |

**"열린 파일 서술자가 살아남는다"** 는 것이 결정적입니다. 부모가 서술자를 준비해 두고 `exec`을 하면, **새 프로그램이 그것을 그대로 물려받습니다.** 6강의 재지정과 파이프가 모두 이 성질 위에 서 있습니다.

> 물려주고 싶지 않은 서술자에는 2강 8.3절에서 배운 **`O_CLOEXEC`** 를 씁니다. 개인 키 파일처럼 자식이 보면 안 되는 것에 반드시 필요합니다.
{: .prompt-warning }

---

## 제6절. `fork` + `exec` = 셸

### 6.1 셸이 하는 일

터미널에서 `ls -l`을 치면 셸은 다음을 합니다.

| 순서 | 하는 일 |
|---|---|
| 1 | 줄을 읽어 단어로 자른다 |
| 2 | **`fork`** — 자신을 복제한다 |
| 3 | 자식이 **`exec`** — `ls`가 된다 |
| 4 | 부모는 **`wait`** — 끝나기를 기다린다 |
| 5 | 종료 상태를 `$?`에 담고 다음 프롬프트 |

**왜 `fork` 없이 그냥 `exec`을 하지 않을까요?** 그러면 **셸 자신이 `ls`가 되어 버려** 명령이 끝나는 순간 터미널도 끝납니다. 자식을 만들어 그 자식만 희생시키는 것입니다.

### 6.2 작은 셸

```c
        pid = fork();
        if (pid == -1) {
            fprintf(stderr, "fork: %s\n", strerror(errno));
            continue;
        }

        if (pid == 0) {
            execvp(args[0], args);
            fprintf(stderr, "%s: %s\n", args[0], strerror(errno));
            _exit(127);
        }

        if (waitpid(pid, &status, 0) == -1) {
            fprintf(stderr, "waitpid: %s\n", strerror(errno));
            continue;
        }
        if (WIFEXITED(status) && WEXITSTATUS(status) != 0)
            fprintf(stderr, "[종료 상태 %d]\n", WEXITSTATUS(status));
        else if (WIFSIGNALED(status))
            fprintf(stderr, "[시그널 %d 로 종료]\n", WTERMSIG(status));
```

줄을 단어로 자르는 부분입니다.

```c
/* split: 줄을 공백으로 잘라 argv 를 만든다. 인자 개수를 돌려준다 */
static int split(char *line, char *argv[], int max)
{
    int n = 0;
    char *p = line;

    while (*p != '\0' && n < max - 1) {
        while (*p == ' ' || *p == '\t')
            *p++ = '\0';            /* 공백을 끝 표시로 바꾼다 */
        if (*p == '\0')
            break;
        argv[n++] = p;
        while (*p != '\0' && *p != ' ' && *p != '\t')
            p++;
    }
    argv[n] = NULL;                 /* execvp 는 NULL 로 끝나야 한다 */
    return n;
}
```

**새 메모리를 빌리지 않습니다.** 원래 줄에서 공백을 `'\0'`으로 바꾸고 그 자리를 가리키기만 합니다. 1부 12강에서 CSV를 자를 때 쓴 방법과 같습니다.

```bash
gcc -Wall -Wextra -std=gnu17 minishell.c -o minishell
```

```bash
./minishell
```

```text
minish$ echo 안녕하세요
안녕하세요
minish$ pwd
/home/student/cmid/lab04
minish$ false
[종료 상태 1]
minish$ 없는명령
없는명령: No such file or directory
[종료 상태 127]
minish$ exit
minish 를 마칩니다.
```

### 6.3 내장 명령이 필요한 이유

```c
        if (strcmp(args[0], "cd") == 0) {       /* 이것도 내장이어야 한다 */
            const char *dir = (n > 1) ? args[1] : getenv("HOME");
            if (dir == NULL || chdir(dir) == -1)
                fprintf(stderr, "cd: %s\n", strerror(errno));
            continue;
        }
```

> **`cd`를 자식에서 실행하면 아무 소용이 없습니다.**
> 자식의 작업 디렉터리만 바뀌고 자식은 곧 끝나 버립니다. 부모인 셸은 제자리입니다. 그래서 `cd`는 **반드시 셸 자신이 직접** 해야 하며, 이런 명령을 **내장 명령(builtin)** 이라 부릅니다.
{: .prompt-danger }

```bash
which cd
```

**아무것도 나오지 않습니다.** `/bin/cd`라는 프로그램은 존재하지 않습니다. `exit`도 마찬가지입니다.

### 6.4 이 셸이 못 하는 것

| 못 하는 것 | 어떻게 되나 | 언제 배우나 |
|---|---|---|
| 따옴표 | `sh -c "exit 7"`이 세 단어로 잘린다 | 실습문제 9 |
| 재지정 `>` `<` | 그냥 인자로 넘어간다 | **6강** |
| 파이프 `\|` | 마찬가지 | **6강** |
| 배경 실행 `&` | 마찬가지 | 실습문제 10 |
| 변수 확장 `$HOME` | 그대로 넘어간다 | — |
| Ctrl+C 처리 | 셸까지 함께 죽는다 | **5강** |

**지금은 이것으로 충분합니다.** 뼈대가 `fork` + `exec` + `wait`이라는 것을 확인하는 것이 목적입니다. 6강에서 재지정과 파이프를 얹으면 쓸 만한 셸이 됩니다.

---

## 제7절. `wait`과 종료 상태

### 7.1 두 함수

```c
#include <sys/wait.h>

pid_t wait(int *wstatus);
pid_t waitpid(pid_t pid, int *wstatus, int options);
```

| 함수 | 기다리는 대상 |
|---|---|
| `wait` | **아무 자식이나** 하나 |
| `waitpid` | **지정한 자식**(또는 `-1`이면 아무나) |

| `options` | 뜻 |
|---|---|
| 0 | 끝날 때까지 **기다린다** |
| `WNOHANG` | **기다리지 않는다.** 끝난 자식이 없으면 0을 돌려준다 |

자식이 여럿일 때는 `waitpid`가 필요합니다. `wait`으로는 **어느 자식이 끝났는지 고를 수 없기** 때문입니다.

### 7.2 종료 상태는 정수 하나가 아니다

`wstatus`에는 **여러 정보가 비트로 묶여** 들어 있습니다. 직접 해석하지 말고 **반드시 매크로를 쓰십시오.**

| 매크로 | 묻는 것 | 함께 쓰는 매크로 |
|---|---|---|
| `WIFEXITED` | 정상적으로 끝났나 | `WEXITSTATUS` — 종료 상태 |
| `WIFSIGNALED` | **시그널로 죽었나** | `WTERMSIG` — 시그널 번호 |
| `WIFSTOPPED` | 멈췄나 | `WSTOPSIG` |
| `WCOREDUMP` | 코어 덤프가 생겼나 | — |

```c
static void report(int status)
{
    if (WIFEXITED(status)) {
        printf("정상 종료, 종료 상태 %d\n", WEXITSTATUS(status));
    } else if (WIFSIGNALED(status)) {
        printf("시그널로 종료, 시그널 %d (%s)%s\n",
               WTERMSIG(status), strsignal(WTERMSIG(status)),
               WCOREDUMP(status) ? " — 코어 덤프 생성" : "");
    } else if (WIFSTOPPED(status)) {
        printf("멈춤, 시그널 %d\n", WSTOPSIG(status));
    } else {
        printf("알 수 없는 상태 0x%x\n", status);
    }
}
```

### 7.3 네 가지 경우를 직접 확인

```bash
./exitstatus true
```

```text
자식 451: 정상 종료, 종료 상태 0
```

```bash
./exitstatus sh -c "exit 42"
```

```text
자식 453: 정상 종료, 종료 상태 42
```

```bash
./exitstatus sh -c "kill -SEGV \$\$"
```

```text
자식 455: 시그널로 종료, 시그널 11 (Segmentation fault) — 코어 덤프 생성
```

```bash
./exitstatus 없는명령
```

```text
없는명령: No such file or directory
자식 457: 정상 종료, 종료 상태 127
```

**마지막이 중요합니다.** 명령을 찾지 못한 것은 **자식이 `_exit(127)`을 부른 결과**이지 커널이 정해 준 값이 아닙니다. 우리가 그렇게 만든 것입니다.

### 7.4 셸의 `$?`와 같은 값입니다

```bash
sh -c "exit 42"; echo $?
```

```text
42
```

```bash
없는명령 2>/dev/null; echo $?
```

```text
127
```

**우리 프로그램이 얻은 값과 셸이 보여 주는 값이 같습니다.** 셸도 똑같이 `waitpid`로 받아 `WEXITSTATUS`로 꺼내고 있기 때문입니다.

| 종료 상태 | 관례 |
|---|---|
| 0 | 성공 |
| 1~125 | 프로그램이 정한 실패 |
| **126** | 실행 권한이 없음 |
| **127** | **명령을 찾을 수 없음** |
| 128+N | 시그널 N으로 죽음(셸의 표기) |

> **종료 상태는 0~255만 전달됩니다.** `exit(256)`은 0이 되고 `exit(-1)`은 255가 됩니다. 1부 10강에서 배운 자료형 절단이 여기서도 나타납니다.
{: .prompt-warning }

### 7.5 기다리지 않고 확인하기

```c
pid_t w = waitpid(-1, &status, WNOHANG);

if (w == 0) {
    /* 아직 끝난 자식이 없다 — 다른 일을 계속한다 */
} else if (w > 0) {
    /* w 번 자식이 끝났다 */
}
```

서버가 여러 자식을 돌리면서 **막히지 않고** 정리해야 할 때 씁니다. 10강의 다중 접속 서버에서 다시 만납니다.

---

## 제8절. 좀비

### 8.1 왜 생기는가

자식이 끝나도 **커널은 종료 상태를 버리지 않습니다.** 부모가 `wait`으로 가져갈 때까지 보관합니다.

> **좀비(zombie)**: 실행은 끝났지만 **부모가 아직 거두지 않아** 종료 상태만 남아 있는 프로세스.

| 좀비가 가진 것 | 가지지 않은 것 |
|---|---|
| PID, 종료 상태 | 메모리, 열린 파일, CPU 시간 |

**좀비는 메모리를 먹지 않습니다.** 그런데도 문제가 되는 이유는 **PID를 차지하기** 때문입니다. PID는 유한하므로, 좀비가 계속 쌓이면 새 프로세스를 만들 수 없게 됩니다.

### 8.2 관찰

```c
    pid = fork();
    if (pid == 0) {
        printf("[자식 %d] 곧바로 끝납니다\n", (int) getpid());
        _exit(3);
    }

    printf("[부모 %d] 자식 %d 를 거두지 않고 3초 기다립니다\n",
           (int) getpid(), (int) pid);
    ...
    wait(NULL);                     /* 이제 거둔다 */
```

```bash
./zombie
```

```text
[부모 425] 자식 426 를 거두지 않고 3초 기다립니다
--- 거두기 전 ---
    PID    PPID STAT COMMAND
    426     425 Z+   zombie
--- 거둔 뒤 ---
    PID    PPID STAT COMMAND
(그런 프로세스가 없습니다 — 완전히 사라졌습니다)
```

**`STAT` 열의 `Z`가 좀비**입니다. `ps`에서는 `<defunct>`로 표시되기도 합니다.

```bash
ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'
```

### 8.3 막는 방법

| 방법 | 설명 | 배우는 곳 |
|---|---|---|
| **`wait`으로 거둔다** | 가장 기본 | 이번 강의 |
| `WNOHANG`으로 주기적으로 확인 | 막히지 않는다 | 7.5절 |
| **`SIGCHLD` 처리기에서 거둔다** | 자식이 끝날 때 알림을 받는다 | **5강** |
| `SIGCHLD`를 무시하도록 설정 | 커널이 알아서 거둔다 | 5강 |
| **두 번 `fork`** | 손자를 만들어 subreaper에게 맡긴다 | 실습문제 8 |

**서버 프로그램에서 좀비 처리를 빠뜨리는 것은 흔한 사고**입니다. 접속마다 자식을 만드는 구조에서는 반드시 필요합니다.

---

> **▶ 여기서부터 3회차 — 고아·종료 방법과 실습**
> 제9절 ~ 실습문제, 약 180분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제9절. 고아와 subreaper

### 9.1 부모가 먼저 죽으면

```c
    pid_t pid = fork();

    if (pid == 0) {
        printf("[자식 %d] 처음 부모 = %d\n", (int) getpid(), (int) getppid());
        fflush(stdout);
        sleep(2);                   /* 그사이 부모가 먼저 끝난다 */
        printf("[자식 %d] 부모가 죽은 뒤 부모 = %d\n",
               (int) getpid(), (int) getppid());
        return 0;
    }

    printf("[부모 %d] 자식 %d 를 남기고 먼저 끝납니다\n",
           (int) getpid(), (int) pid);
    return 0;
```

```bash
./orphan
```

```text
[부모 353] 자식 354 를 남기고 먼저 끝납니다
[자식 354] 처음 부모 = 353
[자식 354] 부모가 죽은 뒤 부모 = 272
```

**부모가 바뀌었습니다.** 부모 없는 프로세스는 존재할 수 없기 때문에 **누군가가 입양**합니다.

### 9.2 누가 입양하는가 — 교과서와 다를 수 있습니다

오래된 교재는 **"고아는 PID 1(init)이 거둔다"** 고 설명합니다. 지금은 조건부로만 맞습니다.

[`PR_SET_CHILD_SUBREAPER`](https://man7.org/linux/man-pages/man2/PR_SET_CHILD_SUBREAPER.2const.html) 문서는 이렇게 적고 있습니다.

> "When a process becomes orphaned (i.e., its immediate parent terminates), then that process will be reparented to the nearest still living ancestor subreaper."

| 상황 | 새 부모 |
|---|---|
| 조상 중에 **subreaper**가 있다 | **가장 가까운 subreaper** |
| 없다 | PID 1 (init / systemd) |

`prctl(PR_SET_CHILD_SUBREAPER, 1)`로 자신을 subreaper로 등록한 프로세스는 **자기 자손의 고아를 대신 거둡니다.** Linux 3.4(2012)에 들어온 기능이며, 오늘날 `systemd --user`·세션 관리자·컨테이너 런타임이 널리 씁니다.

### 9.3 직접 확인하십시오

**위 실행에서 새 부모는 PID 1이 아니라 272였습니다.** 그것이 무엇인지 확인해 보십시오.

```bash
ps -o pid,ppid,comm -p 272
```

```text
    PID    PPID COMMAND
    272     271 Relay(273)
```

이 결과는 **검증 환경이 WSL이었기 때문**입니다. `Relay`는 WSL이 세션을 중계하려고 띄우는 프로세스이며, subreaper로 등록되어 있습니다.

```bash
systemd-detect-virt
```

```text
wsl
```

> **여러분의 VM에서는 다른 값이 나올 수 있습니다.**
> 그것이 정상입니다. 중요한 것은 **번호가 아니라 확인하는 방법**입니다.
>
> ```bash
> ./orphan ; sleep 3
> ```
> 로 새 PPID를 얻고, `ps -o pid,comm -p <그 번호>`로 정체를 확인하십시오. PID 1이 나올 수도, `systemd --user`가 나올 수도 있습니다. **어느 쪽이든 9.2절의 규칙으로 설명됩니다.**
{: .prompt-warning }

```bash
ps -o pid,comm -p 1
```

```text
    PID COMMAND
      1 systemd
```

### 9.4 고아는 문제가 아닙니다

| | 좀비 | 고아 |
|---|---|---|
| 무엇 | 끝났는데 안 거두어짐 | 부모가 먼저 죽음 |
| 살아 있나 | **아니다** | **그렇다** |
| 자원 | PID만 차지 | 정상적으로 사용 |
| 문제인가 | **쌓이면 문제** | **정상 동작** |

**고아가 되는 것은 오히려 의도적으로 쓰이기도 합니다.** 데몬(배경 서비스)을 만들 때 일부러 부모를 죽여 터미널에서 떼어 냅니다.

---

## 제10절. 끝내는 법 — `exit`와 `_exit`

### 10.1 세 가지 방법

| 방법 | 버퍼를 비우나 | `atexit` 함수를 부르나 |
|---|---|---|
| `return`(main에서) | **예** | **예** |
| [`exit(3)`](https://man7.org/linux/man-pages/man3/exit.3.html) | **예** | **예** |
| [`_exit(2)`](https://man7.org/linux/man-pages/man2/_exit.2.html) | **아니오** | **아니오** |

### 10.2 확인

```c
/* exit_vs_exit.c — exit 와 _exit 은 무엇이 다른가
   사용법: ./exit_vs_exit [under]   ('under' 를 주면 _exit 을 쓴다) */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static void bye(void)
{
    printf("[atexit] 정리 함수가 불렸습니다\n");
}

int main(int argc, char *argv[])
{
    atexit(bye);

    printf("이 줄은 버퍼에 있습니다");   /* 개행이 없다 */

    if (argc > 1 && argv[1][0] == 'u')
        _exit(0);                        /* 버퍼를 비우지 않고 즉시 끝낸다 */

    exit(0);                             /* 버퍼를 비우고 atexit 도 부른다 */
}
```

```bash
./exit_vs_exit
```

```text
이 줄은 버퍼에 있습니다[atexit] 정리 함수가 불렸습니다
```

```bash
./exit_vs_exit under
```

```text
```

**`_exit`을 쓰면 아무것도 출력되지 않습니다.** 버퍼에 있던 내용이 그대로 버려졌습니다.

### 10.3 그런데 왜 `_exit`을 쓰는가

**자식 프로세스에서는 `_exit`이 옳은 경우가 많습니다.**

| 이유 | 설명 |
|---|---|
| **버퍼 중복 출력을 막는다** | 제4절의 함정. 부모가 이미 출력할 내용을 자식이 또 내보내지 않는다 |
| **`atexit` 중복 실행을 막는다** | 부모가 등록한 정리 함수가 자식에서도 불리면 안 된다 |
| `exec` 실패 후 | 어차피 곧 사라질 프로세스다 |

> **규칙**
> - **부모(원래 프로세스)**: `return` 또는 `exit`
> - **`fork`로 만든 자식**: **`_exit`**
{: .prompt-tip }

이 과정의 모든 예제에서 자식이 `_exit`을 쓴 이유가 이것입니다. 다만 자식이 꼭 출력해야 할 것이 있다면 **`_exit` 전에 `fflush`** 를 해야 합니다(3.1절에서 그렇게 했습니다).

---

## 제11절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| 출력이 두 번 나온다 | `fork` 전에 버퍼가 남아 있음 | **`fflush` 후 `fork`** |
| `exec` 다음 줄이 실행된다 | `exec`이 **실패**한 것 | `errno`를 보고 보고 후 `_exit(127)` |
| `exec` 인자가 이상하다 | `NULL` 누락 / `argv[0]` 빠뜨림 | 목록 끝에 `NULL`, 첫 인자는 프로그램 이름 |
| 자식이 안 끝나는데 부모가 먼저 끝난다 | `wait` 누락 | `wait`/`waitpid` |
| `ps`에 `Z`가 쌓인다 | 좀비 | `wait` 또는 `SIGCHLD`(5강) |
| 종료 상태가 이상하다 | `status`를 직접 해석 | **`WEXITSTATUS` 매크로** |
| 종료 상태 256이 0이 된다 | 8비트만 전달 | 0~255만 쓴다 |
| `cd`가 안 먹는다 | 자식에서 실행 | **내장 명령으로** |
| 자식의 출력이 사라진다 | `_exit`이 버퍼를 안 비움 | 앞에 `fflush` |
| 고아의 부모가 1이 아니다 | subreaper | **정상**. 9.2절 |
| `fork` 결과 검사 누락 | `-1`도 0이 아니다 | `if (pid == -1)`을 **먼저** |
| 자식이 부모의 변수를 못 바꾼다 | 메모리는 복사됨 | **정상**. 3.2절 |
| 파일이 덮어써진다 | 서술자를 공유해 위치가 겹침 | `O_APPEND` 또는 따로 `open` |

---

## 실습문제

> **안내**
> 1. 컴파일은 **`gcc -Wall -Wextra -std=gnu17`**, **경고 0개**여야 합니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 |
|---|---|---|
| 문제 1 | `fork` 기본 | 2 |
| 문제 2 | 복사와 공유 확인 | 3 |
| 문제 3 | 버퍼 중복 재현 | 4 |
| 문제 4 | 자식 여러 개 만들기 | 2 · 7 |
| 문제 5 | `exec` 여섯 가지 | 5 |
| 문제 6 | 종료 상태 해석 | 7 |
| 문제 7 | 좀비 만들고 없애기 | 8 |
| 문제 8 | **두 번 `fork`** | 8 · 9 |
| 문제 9 | 미니 셸에 따옴표 넣기 | 6 |
| 문제 10 | 배경 실행 `&` | 6 · 7 |

---

### 문제 1. `fork` 기본

`fork`의 반환값으로 부모와 자식을 나누고, 각자의 PID·PPID를 출력하십시오.

**정답 및 해설**

2.2절의 코드가 답입니다.

- **`pid == -1`을 가장 먼저 검사**해야 합니다. `-1`은 0이 아니므로 `if (pid == 0) … else …`로만 쓰면 **실패를 부모로 착각**합니다.
- 자식의 `getppid()`가 부모의 `getpid()`와 같은지 확인하십시오. 다르다면 부모가 이미 죽어 입양된 것입니다(제9절).
- **여러 번 실행해 출력 순서가 바뀌는지** 보십시오. 순서는 보장되지 않습니다(2.4절).

---

### 문제 2. 복사와 공유 확인

전역·지역·힙 변수와 파일 위치가 각각 복사되는지 공유되는지 실험으로 확인하십시오.

**정답 및 해설**

3.1절의 코드와 결과가 답입니다.

| 항목 | 결과 |
|---|---|
| 전역·지역·힙 | **복사** — 자식이 바꿔도 부모는 그대로 |
| 파일 위치 | **공유** — 자식이 5바이트 쓰면 부모의 위치도 5 |

- 파일 내용이 `AAAAABBBBB`인 것이 공유의 증거입니다. 위치가 따로였다면 부모의 `BBBBB`가 앞을 덮어썼을 것입니다.
- **자식에서 `fflush(stdout)`을 빠뜨리면 자식의 출력이 사라집니다.** `_exit`이 버퍼를 비우지 않기 때문입니다(10.3절). 직접 빼 보고 확인하십시오.
- 부모가 `wait(NULL)`을 부른 이유는 **자식이 먼저 쓰도록 순서를 맞추기** 위해서입니다. 없으면 결과가 실행마다 달라집니다.

---

### 문제 3. 버퍼 중복 재현

`fork` 전 출력이 두 번 나오는 현상을 재현하고, `fflush`로 고치십시오.

**정답 및 해설**

4.1절의 프로그램을 **반드시 파일로 재지정해** 실행해야 합니다.

```bash
./fork_buffer          # 터미널 → 1번
./fork_buffer > fb.txt ; wc -l fb.txt   # 파일 → 2번
```

```text
2 fb.txt
```

`printf` 다음에 `fflush(stdout);`을 넣고 다시 재면 1이 됩니다.

- **화면에서는 재현되지 않습니다.** 터미널은 줄 단위 버퍼링이라 개행에서 이미 비워지기 때문입니다. "내 화면에서는 잘 되는데"의 전형적인 사례입니다.
- 파이프로 보내도 같은 현상이 납니다. `./fork_buffer | cat`으로 확인해 보십시오.
- 1부 9강에서 `fflush`를 배울 때는 "출력이 안 보이는 문제"였는데, 여기서는 **"출력이 두 번 보이는 문제"** 가 되었습니다. 원인은 같습니다.

---

### 문제 4. 자식 여러 개 만들기

자식 세 개를 만들어 각자 다른 일을 시키고, 부모가 **모두** 거두게 하십시오.

**정답 및 해설**

```c
    for (i = 0; i < 3; i++) {
        pid = fork();
        if (pid == -1) {
            fprintf(stderr, "fork: %s\n", strerror(errno));
            break;
        }
        if (pid == 0) {
            printf("[자식 %d] 번호 %d\n", (int) getpid(), i);
            fflush(stdout);
            sleep(i + 1);
            _exit(i);               /* 번호를 종료 상태로 */
        }
    }

    /* 부모만 여기에 온다. 자식은 모두 _exit 으로 빠져나갔다 */
    while ((pid = wait(&status)) > 0)
        printf("자식 %d 거둠, 종료 상태 %d\n",
               (int) pid, WEXITSTATUS(status));
```

```bash
./manykids
```

```text
[자식 407] 번호 0
[자식 408] 번호 1
[자식 409] 번호 2
자식 407 거둠, 종료 상태 0
자식 408 거둠, 종료 상태 1
자식 409 거둠, 종료 상태 2
```

- **자식이 반드시 `_exit`으로 빠져나가야 합니다.** 그러지 않으면 자식도 반복문을 계속 돌아 **손자·증손자가 폭증**합니다. `fork` 폭탄이 되는 흔한 실수입니다.
- 부모의 `while ((pid = wait(&status)) > 0)`은 **거둘 자식이 없어지면** `wait`이 `-1`(`ECHILD`)을 돌려주므로 자연히 끝납니다.
- 끝나는 **순서는 정해져 있지 않습니다.** `sleep(i + 1)`을 주었으므로 대체로 번호순이지만, 그것도 보장은 아닙니다.

---

### 문제 5. `exec` 여섯 가지

같은 명령을 `execl`·`execv`·`execlp`·`execvp`로 각각 실행해 보고 차이를 정리하십시오.

**정답 및 해설**

```c
    switch (which) {
    case 'l':
        execl("/bin/ls", "ls", "-l", NULL);          break;
    case 'v': {
        char *args[] = { "ls", "-l", NULL };
        execv("/bin/ls", args);                      break;
    }
    case 'p':
        execlp("ls", "ls", "-l", NULL);              break;
    case 'q': {
        char *args[] = { "ls", "-l", NULL };
        execvp("ls", args);                          break;
    }
    }
    fprintf(stderr, "exec 실패: %s\n", strerror(errno));
    _exit(127);
```

| 함수 | 경로 | 인자 |
|---|---|---|
| `execl` | **전체 경로 필요** | 나열 |
| `execv` | **전체 경로 필요** | 배열 |
| `execlp` | 이름만으로 OK | 나열 |
| `execvp` | 이름만으로 OK | 배열 |

```bash
for w in l v p q; do printf "%s -> " $w; ./exec_demo $w >/dev/null 2>&1 && echo 성공 || echo 실패; done
```

```text
l -> 성공
v -> 성공
p -> 성공
q -> 성공
```

```bash
./exec_demo x        # execl("ls", ...) — p 가 없는데 이름만 주었다
```

```text
exec 실패: No such file or directory
```

- **`p`가 없는 것에 `"ls"`만 주면 실패합니다.** `ENOENT`가 나옵니다. `PATH`를 찾지 않기 때문입니다.
- 셸을 만들 때는 **`execvp`** 가 알맞습니다. 사용자가 친 이름을 `PATH`에서 찾아야 하고, 인자 개수가 실행 중에 정해지므로 배열이어야 합니다.

**`strace`로 보면 `p`가 하는 일이 그대로 드러납니다.**

```bash
strace -e trace=execve ./exec_demo p 2>&1 >/dev/null | head -2
```

```text
execve("./exec_demo", ["./exec_demo", "p"], 0x7fffb9c9c558 /* 22 vars */) = 0
execve("/usr/local/sbin/ls", ["ls", "-l"], 0x7ffcfdb89180 /* 22 vars */) = -1 ENOENT (No such file or directory)
```

- **두 번째 줄이 `PATH` 탐색의 첫 시도**입니다. `execlp`는 `PATH`의 디렉터리를 앞에서부터 하나씩 붙여 가며 **`execve`를 반복 호출**합니다. `/usr/local/sbin/ls`가 없으니 `ENOENT`, 그다음 `/usr/local/bin/ls`… 이런 식으로 찾다가 `/usr/bin/ls`에서 성공합니다.
- 즉 **`execlp`는 마법이 아니라 `execve`를 여러 번 부르는 라이브러리 함수**입니다. 실제 시스템 호출은 언제나 `execve` 하나뿐이라는 5.2절의 설명이 눈으로 확인됩니다.

---

### 문제 6. 종료 상태 해석

`WIFEXITED`·`WIFSIGNALED`로 자식의 최후를 판별하는 프로그램을 만드십시오.

**정답 및 해설**

7.2절의 `report` 함수가 답이며, 결과는 7.3절과 같습니다.

| 시험 | 결과 |
|---|---|
| `true` | 종료 상태 0 |
| `sh -c "exit 42"` | 종료 상태 42 |
| `sh -c "kill -SEGV $$"` | 시그널 11 |
| 없는 명령 | 종료 상태 127 |

- **`WEXITSTATUS`는 `WIFEXITED`가 참일 때만 의미가 있습니다.** 시그널로 죽은 경우에 꺼내면 쓰레기 값이 나옵니다. 반드시 순서대로 검사하십시오.
- 셸의 `$?`와 비교해 같은지 확인하십시오(7.4절). 우리가 커널에서 직접 받은 값과 셸이 보여 주는 값이 같아야 합니다.
- `strsignal`이 시그널 번호를 사람이 읽는 문자열로 바꾸어 줍니다. 5강에서 시그널을 본격적으로 다룹니다.

---

### 문제 7. 좀비 만들고 없애기

좀비를 만들어 `ps`로 확인한 뒤, `wait`으로 없애십시오.

**정답 및 해설**

8.2절의 결과가 답입니다.

```text
    PID    PPID STAT COMMAND
    426     425 Z+   zombie
```

- **`STAT`이 `Z`** 인지 확인하십시오. `Z+`의 `+`는 전면 프로세스 그룹이라는 뜻입니다.
- 좀비의 `/proc/PID/`는 여전히 존재합니다. 그러나 `/proc/PID/fd`는 비어 있습니다. **자원은 이미 반환되었고 종료 상태만 남았다**는 사실을 눈으로 확인할 수 있습니다.

```bash
ls /proc/<좀비PID>/fd
```

- 부모를 강제로 죽여도 좀비가 사라집니다. 고아가 된 좀비를 subreaper가 즉시 거두기 때문입니다(제9절).

---

### 문제 8. 두 번 `fork`

부모가 `wait`으로 막히지 않으면서도 **좀비를 남기지 않는** 방법을 구현하십시오.

**정답 및 해설**

**손자를 만들고 중간 자식을 즉시 끝냅니다.**

```c
    pid = fork();
    if (pid == -1)
        die("fork");

    if (pid == 0) {                 /* 중간 자식 */
        pid_t grand = fork();

        if (grand == -1)
            _exit(1);
        if (grand > 0)
            _exit(0);               /* 중간 자식은 곧바로 끝난다 */

        /* 여기부터 손자 — 진짜 일을 한다 */
        printf("[손자 %d] 부모 = %d\n", (int) getpid(), (int) getppid());
        sleep(2);
        printf("[손자 %d] 부모 = %d  <- 입양되었다\n",
               (int) getpid(), (int) getppid());
        _exit(0);
    }

    waitpid(pid, NULL, 0);          /* 중간 자식만 거둔다 — 즉시 끝난다 */
    printf("[부모 %d] 중간 자식을 거두었습니다. 손자는 남의 손에 맡겼습니다\n",
           (int) getpid());
```

| 왜 되는가 |
|---|
| 중간 자식이 **즉시** 끝나므로 `waitpid`가 막히지 않는다 |
| 손자는 부모를 잃어 **subreaper에게 입양**된다(제9절) |
| 손자가 끝나면 **그 subreaper가 거둔다** — 좀비가 되지 않는다 |

```bash
./doublefork ; sleep 3
```

```text
[부모 419] 중간 자식을 거두었습니다. 손자는 남의 손에 맡겼습니다
[손자 421] 부모 = 279
[손자 421] 부모 = 279  <- 입양되었다
```

> **첫 출력에서 이미 부모가 279입니다.**
> 중간 자식이 워낙 빨리 끝나 **손자가 첫 줄을 찍기도 전에 입양이 끝났기** 때문입니다. 원래 부모(중간 자식)의 PID를 보고 싶다면 손자에서 `sleep` 없이 곧바로 찍어도 경쟁이 되므로, 중간 자식이 잠시 기다리게 해야 합니다. **순서는 보장되지 않는다**는 2.4절의 원칙이 여기서도 적용됩니다.
{: .prompt-info }

- **데몬을 만드는 고전적인 방법**이며, 여기에 `setsid`와 작업 디렉터리 변경을 더하면 완전한 데몬이 됩니다.
- 오늘날에는 `systemd`가 서비스를 직접 관리하므로 이 기법을 덜 쓰지만, **왜 이렇게 하는지 아는 것**은 여전히 중요합니다.
- 5강에서 배울 `SIGCHLD` 처리기가 더 간단한 대안입니다.

---

### 문제 9. 미니 셸에 따옴표 넣기

`split` 함수를 고쳐 `echo "안녕 하세요"`가 **한 개의 인자**로 넘어가게 하십시오.

**정답 및 해설**

**따옴표 안에서는 공백을 구분자로 보지 않으면 됩니다.**

```c
static int split(char *line, char *argv[], int max)
{
    int n = 0;
    char *p = line, *w;

    while (*p != '\0' && n < max - 1) {
        while (*p == ' ' || *p == '\t')
            p++;
        if (*p == '\0')
            break;

        if (*p == '"' || *p == '\'') {      /* 따옴표로 시작하면 */
            char quote = *p++;
            argv[n++] = w = p;
            while (*p != '\0' && *p != quote)
                *w++ = *p++;                /* 짝을 만날 때까지 그대로 */
            if (*p != '\0')
                p++;                        /* 닫는 따옴표를 건너뛴다 */
            *w = '\0';
        } else {
            argv[n++] = p;
            while (*p != '\0' && *p != ' ' && *p != '\t')
                p++;
            if (*p != '\0')
                *p++ = '\0';
        }
    }
    argv[n] = NULL;
    return n;
}
```

```text
minish$ echo "안녕 하세요"
안녕 하세요
minish$ echo '작은 따옴표'
작은 따옴표
```

- **읽는 자리(`p`)와 쓰는 자리(`w`)를 나눈 것**이 요령입니다. 따옴표를 걷어내면 문자열이 짧아지므로 제자리에서 앞으로 당깁니다. 새 메모리를 빌리지 않는 방식은 그대로입니다.
- 닫는 따옴표가 없는 경우를 처리했는지 확인하십시오. 진짜 셸은 다음 줄을 더 읽지만, 여기서는 줄 끝까지를 인자로 삼으면 충분합니다.
- 진짜 셸은 `"`와 `'`의 동작이 다릅니다(변수 확장 여부). 여기서는 구분하지 않았습니다.

---

### 문제 10. 배경 실행 `&`

명령 끝에 `&`가 있으면 **기다리지 않고** 다음 프롬프트를 내도록 하십시오.

**정답 및 해설**

```c
        /* 마지막 인자가 & 이면 떼어 내고 표시해 둔다 */
        background = 0;
        if (n > 0 && strcmp(args[n - 1], "&") == 0) {
            background = 1;
            args[--n] = NULL;
        }
        ...
        if (background) {
            printf("[배경 실행] PID %d\n", (int) pid);
        } else {
            waitpid(pid, &status, 0);
            ...
        }

        /* 프롬프트를 내기 전에 끝난 배경 자식들을 거둔다 */
        while (waitpid(-1, &status, WNOHANG) > 0)
            ;
```

```text
minish$ sleep 3 &
[배경 실행] PID 512
minish$ 
```

**정말로 좀비가 남지 않는지 확인하십시오.**

```bash
printf 'sleep 1 &\nsleep 2\nsleep 0\nexit\n' | ./minishell2 > /dev/null 2>&1
ps -eo stat | grep -c '^Z'
```

```text
0
```

- **`WNOHANG`으로 주기적으로 거두는 것**이 핵심입니다(7.5절). 그러지 않으면 배경 자식이 모두 좀비가 됩니다. `while (waitpid(-1, &status, WNOHANG) > 0) ;` 줄을 지우고 다시 세어 보면 차이를 볼 수 있습니다.
- 진짜 셸은 자식이 끝나는 **순간** `SIGCHLD`를 받아 `[1]+ Done` 같은 알림을 냅니다. 그것은 5강에서 배웁니다.
- 배경 프로세스는 터미널을 부모와 공유하므로, 출력이 프롬프트와 뒤섞입니다. 진짜 셸은 **프로세스 그룹**으로 이를 관리합니다.

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 소스 — `fork1.c`, `fork_share.c`, `fork_buffer.c`, `exitstatus.c`, `zombie.c`, `orphan.c`, `minishell.c`, `exit_vs_exit.c`, `doublefork.c` |
| 2 | `fork_share` 실행 결과와 **복사/공유 판정표**(문제 2) |
| 3 | **버퍼 중복 재현** — 터미널과 파일 재지정 두 화면(문제 3) |
| 4 | 네 가지 종료 상태 결과와 셸 `$?`와의 비교(문제 6) |
| 5 | 좀비의 `ps` 화면 — 거두기 전후(문제 7) |
| 6 | **고아의 새 부모 PID와 그 정체** — `ps`로 확인한 결과(9.3절) |
| 7 | 미니 셸 실행 화면 — 따옴표·배경 실행 포함(문제 9·10) |
| 8 | 짧은 서술 ① `fork` 후 메모리는 복사되는데 파일 위치는 공유되는 이유 |
| 9 | 짧은 서술 ② 자식에서 `exit` 대신 `_exit`을 쓰는 이유 |

---

## 정리

| 구분 | 핵심 |
|---|---|
| `fork` | **한 번 부르고 두 번 돌아온다.** 자식 0, 부모 자식PID, 실패 −1 |
| 복사 | 전역·지역·힙 **모두 복사**(쓸 때 복사로 효율적) |
| 공유 | **열린 파일 설명 = 파일 위치가 공유된다** |
| 버퍼 | `fork`는 **출력 버퍼도 복제**한다 → **`fflush` 후 `fork`** |
| `exec` | PID 유지, 메모리 교체, **성공하면 돌아오지 않는다** |
| 여섯 변형 | `l` 나열 / `v` 배열 / `p` PATH / `e` 환경. **시스템 호출은 `execve` 하나** |
| 살아남는 것 | **열린 서술자**(6강 재지정의 토대), PID, 작업 디렉터리 |
| 셸 | **`fork` → `exec` → `wait`**. `cd`·`exit`은 내장이어야 한다 |
| 종료 상태 | `WIFEXITED`·`WEXITSTATUS`·`WIFSIGNALED`. **0~255만 전달** |
| 좀비 | 끝났는데 안 거둠. **PID를 차지** → `wait` 필수 |
| 고아 | 부모가 먼저 죽음. **가장 가까운 subreaper**가 입양(없으면 PID 1) |
| 끝내기 | 부모는 `exit`, **자식은 `_exit`** |

---

## 다음 강의 예고

**5강 「시그널」**(APUE 10장)에서는 프로세스에 **비동기적으로 끼어드는** 사건을 다룹니다.

- Ctrl+C를 누르면 실제로 무슨 일이 일어나는가
- `sigaction`으로 시그널을 붙잡는다
- **`SIGCHLD`로 좀비를 자동으로 거둔다** — 이번 강의의 숙제를 푼다
- **`EINTR`** — 2강에서 `_full` 함수에 넣어 둔 그 처리의 정체
- 시그널 처리기 안에서 **해도 되는 일과 안 되는 일**

2강 4.2절에서 "시그널에 끊겼을 뿐이다(5강)"라고 미뤄 둔 이야기를 마무리합니다.

---

## 부록 A. 이번 강의 명령·함수 요약

| 하려는 일 | 함수 / 명령 |
|---|---|
| 프로세스 만들기 | `fork()` |
| 다른 프로그램 되기 | `execl` · `execv` · `execlp` · `execvp` · `execle` · `execve` |
| 자식 기다리기 | `wait(&status)` · `waitpid(pid, &status, 0)` |
| 막히지 않고 확인 | `waitpid(-1, &status, WNOHANG)` |
| 종료 상태 해석 | `WIFEXITED` · `WEXITSTATUS` · `WIFSIGNALED` · `WTERMSIG` |
| 시그널 이름 | `strsignal(WTERMSIG(status))` |
| 끝내기 | `exit()` — 부모 / `_exit()` — 자식 |
| 정리 함수 등록 | `atexit()` |
| 자기 정보 | `getpid()` · `getppid()` |
| 디렉터리 바꾸기 | `chdir()` |
| 프로세스 나무 보기 | `ps -o pid,ppid,stat,comm --forest -e` |
| 좀비 찾기 | `ps -eo pid,ppid,stat,comm \| awk '$3 ~ /^Z/'` |
| 특정 프로세스 | `ps -o pid,ppid,comm -p PID` |
| 자기 실행 파일 | `ls -l /proc/self/exe` |
| 가상화 종류 | `systemd-detect-virt` |

## 부록 B. 표준 문서와 출처

**이번 강의에서 다룬 시스템 호출**

| 함수 | 리눅스 man | POSIX 표준 |
|---|---|---|
| `fork` | [`fork(2)`](https://man7.org/linux/man-pages/man2/fork.2.html) | [`fork()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/fork.html) |
| `execve` | [`execve(2)`](https://man7.org/linux/man-pages/man2/execve.2.html) | [`exec`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/exec.html) |
| `exec` 계열 | [`exec(3)`](https://man7.org/linux/man-pages/man3/exec.3.html) | 위와 같음 |
| `wait`·`waitpid` | [`wait(2)`](https://man7.org/linux/man-pages/man2/wait.2.html) | [`waitpid()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/waitpid.html) |
| `_exit` | [`_exit(2)`](https://man7.org/linux/man-pages/man2/_exit.2.html) | [`_exit()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/_exit.html) |
| `exit`·`atexit` | [`exit(3)`](https://man7.org/linux/man-pages/man3/exit.3.html) · [`atexit(3)`](https://man7.org/linux/man-pages/man3/atexit.3.html) | [`atexit()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/atexit.html) |
| `getpid` | [`getpid(2)`](https://man7.org/linux/man-pages/man2/getpid.2.html) | [`getpid()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/getpid.html) |
| `chdir` | [`chdir(2)`](https://man7.org/linux/man-pages/man2/chdir.2.html) | [`chdir()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/chdir.html) |

**개념 문서**

| 내용 | 문서 |
|---|---|
| **subreaper와 고아 재배정**(9.2절) | [`PR_SET_CHILD_SUBREAPER`](https://man7.org/linux/man-pages/man2/PR_SET_CHILD_SUBREAPER.2const.html) · [`prctl(2)`](https://man7.org/linux/man-pages/man2/prctl.2.html) |
| `/proc`의 프로세스 정보 | [`proc(5)`](https://man7.org/linux/man-pages/man5/proc.5.html) |
| 프로세스 자격 증명 | [`credentials(7)`](https://man7.org/linux/man-pages/man7/credentials.7.html) |

**본문의 주장과 근거**

| 주장 | 근거 |
|---|---|
| 부모는 자식 PID, 자식은 0을 받는다(2.2절) | `fork1` 실행 — 부모 반환값 436 = 자식 PID |
| **메모리는 복사, 파일 위치는 공유**(3.2절) | `fork_share` 실행 — 변수 100/200/300 유지, 위치 양쪽 모두 5, 내용 `AAAAABBBBB` |
| `fork`는 출력 버퍼도 복제(4.1절) | 터미널 1줄 vs 파일 재지정 **2줄**(`wc -l` = 2) |
| 종료 상태 42·127, SIGSEGV 11(7.3절) | `exitstatus` 실행 결과 |
| 우리 값이 셸 `$?`와 같다(7.4절) | `sh -c "exit 42"; echo $?` → 42 |
| 좀비는 `ps`에서 `Z`(8.2절) | `zombie` 실행 — `426 425 Z+ zombie`, `wait` 후 소멸 |
| **고아는 subreaper가 입양**(9.2절) | `PR_SET_CHILD_SUBREAPER` 문서 원문 인용 + `orphan` 실행(새 부모 272 = `Relay`, PID 1은 `systemd`) |
| `_exit`은 버퍼를 비우지 않는다(10.2절) | `exit_vs_exit` 실행 — `exit`은 출력·`atexit` 모두, `_exit`은 아무것도 없음 |
| 미니 셸이 실제로 동작한다(6.2절) | `echo`·`pwd`·`false`(1)·없는명령(127) 실행 결과 |
| `execlp`는 `PATH`를 훑으며 `execve`를 반복한다(문제 5) | `strace -e trace=execve` — `/usr/local/sbin/ls` 시도가 `ENOENT`로 찍힘 |
| `p` 없는 `execl`에 이름만 주면 실패(문제 5) | `./exec_demo x` → `No such file or directory` |
| 자식 셋의 종료 상태 0·1·2를 모두 거둠(문제 4) | `manykids` 실행 결과 |
| 두 번 `fork` 하면 손자가 입양된다(문제 8) | `doublefork` 실행 — 손자의 부모가 279(subreaper) |
| 배경 실행 후 **좀비 0개**(문제 10) | `ps -eo stat \| grep -c '^Z'` → 0 |

> 값은 모두 **Ubuntu 24.04에서 실제로 실행한 결과**입니다. PID는 실행마다 달라지므로 **번호가 아니라 관계**(부모 반환값 = 자식 PID 등)를 보십시오. 9.3절의 고아 입양 결과는 **검증 환경이 WSL이라 여러분의 VM과 다를 수 있으며**, 확인하는 방법을 함께 실었습니다.
{: .prompt-tip }

## 부록 C. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `fork1.c` | `fork`의 두 반환값 | 2 |
| `fork_share.c` | 복사되는 것과 공유되는 것 | 3 |
| `fork_buffer.c` | 버퍼 복제 함정 | 4 |
| `exitstatus.c` | `fork`+`exec`+`waitpid`, 종료 상태 해석 | 5 · 7 |
| `zombie.c` | 좀비 관찰 | 8 |
| `orphan.c` | 고아와 입양 | 9 |
| `exit_vs_exit.c` | `exit`와 `_exit` | 10 |
| `minishell.c` | **작은 셸** | 6 |
| `doublefork.c` | 두 번 `fork` | 문제 8 |
