---
title: C 프로그래밍 기초 11강 - 라이브러리 만들기와 빌드 자동화
date: 2026-11-02 09:00:00 +0900
categories:
  - 0.기초강의
  - C 프로그래밍 기초
tags:
  - C언어
  - 라이브러리
  - 정적라이브러리
  - DLL
  - Makefile
  - 빌드
  - 헤더설계
  - pkgconfig
  - MSYS2
pin:
mermaid: false
---

> **학습 목표**
> 1. 소스·목적 파일·라이브러리·실행 파일의 관계를 설명할 수 있다.
> 2. `ar`로 **정적 라이브러리(`.a`)** 를 만들고 링크할 수 있다.
> 3. `-I`·`-L`·`-l` 옵션의 역할을 구분하고, 각각의 오류 메시지에 대응할 수 있다.
> 4. 동적 라이브러리(DLL)의 개념과 배포 시 주의점을 설명할 수 있다.
> 5. 남이 만든 라이브러리를 설치해 사용할 수 있다.
> 6. `pkg-config`로 컴파일 옵션을 얻을 수 있다.
> 7. 공개할 것과 감출 것을 구분하여 헤더를 설계할 수 있다.
> 8. `Makefile`의 규칙·변수·자동 변수·패턴 규칙을 사용할 수 있다.
> 9. 바뀐 파일만 다시 빌드되는 **의존 관계**를 설명할 수 있다.
> 10. 지금까지 만든 함수들을 하나의 라이브러리로 묶어 배포 가능한 형태로 만들 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

5강에서 프로그램을 **여러 파일로 나누는 법**을 배웠습니다. 그런데 거기서 멈추면 아직 부족합니다.

지금까지 만든 함수들을 떠올려 보십시오. `binsearch`(4강), `itoa`·`trim`·`reverse`(4강), `get_line`(2강), `safe_append`(10강)… **모두 다른 프로그램에서도 쓸모 있는 부품**입니다. 그런데 새 프로그램을 만들 때마다 소스를 복사해 붙여 넣는다면, 버그를 하나 고쳐도 **복사본 전부를 찾아 고쳐야** 합니다.

이번 강의에서는 그 부품들을 **라이브러리로 묶고**, 빌드를 **자동화**합니다. 그리고 남이 만든 라이브러리를 가져다 쓰는 법도 익힙니다. 이것이 실제 C 개발의 마지막 조각입니다.

이 강의는 **3회차 분량**(모두 합쳐 약 435분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제5절 | 정적 라이브러리와 `-I`·`-L`·`-l` | 160분 |
| **2회차** | 제6절 ~ 제10절 | 헤더 설계와 `Makefile` | 175분 |
| **3회차** | 실습문제 | 스스로 해 보기 | 100분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 왜 묶는가 — 소스에서 실행 파일까지 | 25분 |
| 제2절 | 정적 라이브러리 만들기 | 40분 |
| 제3절 | `-I`·`-L`·`-l` 세 옵션 | 30분 |
| 제4절 | 동적 라이브러리(DLL) | 30분 |
| 제5절 | 남이 만든 라이브러리 쓰기 | 35분 |
| 제6절 | 헤더 설계 | 30분 |
| 제7절 | `Makefile` 기초 | 55분 |
| 제8절 | 의존 관계와 증분 빌드 | 30분 |
| 제9절 | 실습 — `libcstudy` 만들기 | 45분 |
| 제10절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 100분 |

**작업 폴더**는 `C:\c-study\lab11`을 사용합니다.

```powershell
mkdir C:\c-study\lab11
```

---

## 제1절. 왜 묶는가 — 소스에서 실행 파일까지

### 1.1 다시 보는 컴파일 단계

1강에서 배운 네 단계를 라이브러리까지 넓혀 봅니다.

| 파일 | 무엇인가 | 만드는 명령 |
|---|---|---|
| `mylib.c` | 사람이 쓴 소스 | 편집기 |
| `mylib.o` | 기계어로 번역된 **목적 파일** | `gcc -c mylib.c -o mylib.o` |
| `libmylib.a` | 목적 파일 여러 개를 묶은 **정적 라이브러리** | `ar rcs libmylib.a mylib.o ...` |
| `app.exe` | 링크가 끝난 **실행 파일** | `gcc main.o -L. -lmylib -o app.exe` |

**라이브러리는 목적 파일들을 담은 보관함**입니다. 링커는 그 안에서 **필요한 것만 꺼내** 실행 파일에 넣습니다.

### 1.2 용어 정리 — C에는 "모듈"이 없습니다

| 하는 일 | C의 정확한 용어 | 결과물 |
|---|---|---|
| 소스를 나누어 따로 컴파일 | **분할 컴파일**, 각 `.c`는 **번역 단위** | — |
| 공개할 것을 적어 둠 | **헤더 파일** | `.h` |
| 번역 결과 | **목적 파일** | `.o` |
| 목적 파일 묶음 | **정적 라이브러리**(archive) | `.a`(MSVC는 `.lib`) |
| 실행할 때 불러 쓰는 묶음 | **동적/공유 라이브러리** | `.dll`(윈도우), `.so`(리눅스) |

C 표준에는 module이라는 개념이 없습니다. 관용적으로 "모듈"이라 부르기는 하지만, 정확히는 **"헤더 + 소스 한 쌍"** 이 하나의 단위이고, 그것들을 묶은 것이 라이브러리입니다.

### 1.3 정적과 동적

| 구분 | 정적 라이브러리(`.a`) | 동적 라이브러리(`.dll`) |
|---|---|---|
| 언제 합쳐지나 | **링크할 때** 실행 파일 안으로 | **실행할 때** 따로 불러온다 |
| 배포 | 실행 파일 하나만 주면 된다 | `.dll`도 함께 줘야 한다 |
| 크기 | 실행 파일이 커진다 | 실행 파일은 작다 |
| 갱신 | 다시 빌드해야 한다 | `.dll`만 바꾸면 된다 |
| 여러 프로그램이 쓸 때 | 각자 복사본을 가진다 | **하나를 공유**한다 |

**이 과정에서는 정적 라이브러리를 본편으로 다룹니다.** 배포가 단순해 실습에 적합하기 때문입니다. 동적 라이브러리는 개념과 시연까지만 봅니다.

---

## 제2절. 정적 라이브러리 만들기

### 2.1 재료 — 5강의 모듈

5강 실습문제 9에서 만든 `mylib.h`·`mylib.c`를 그대로 씁니다.

```c
#ifndef MYLIB_H
#define MYLIB_H

/* 정렬된 배열 v 에서 x 를 찾아 첨자를 돌려준다. 없으면 -1 */
int binsearch(int x, const int v[], int n);

/* 정수 n 을 문자열 s 로 바꾼다 */
void my_itoa(int n, char s[]);

/* 문자열 s 를 제자리에서 뒤집는다 */
void reverse(char s[]);

/* 문자열 끝의 공백·탭·개행을 잘라 내고 새 길이를 돌려준다 */
int trim(char s[]);

#endif   /* MYLIB_H */
```

### 2.2 세 단계

**① 목적 파일 만들기**

```powershell
gcc -Wall -Wextra -std=c17 -c mylib.c -o mylib.o
```

**② 라이브러리로 묶기**

```powershell
ar rcs libmylib.a mylib.o
```

| 옵션 | 뜻 |
|---|---|
| `r` | 넣기(이미 있으면 교체) |
| `c` | 없으면 새로 만들되 경고하지 않는다 |
| `s` | **색인**을 만든다(링커가 빨리 찾도록) |

**③ 사용하기**

```powershell
gcc -Wall -Wextra -std=c17 mylib_main.c -L. -lmylib -o app.exe
```

```text
binsearch(23) = 5
binsearch(40) = -1
my_itoa(-4567) = "-4567"
reverse 적용   = "7654-"
trim 전 = "  hello   "
trim 후 = "  hello" (길이 7)
```

### 2.3 이름 규칙 — 가장 많이 막히는 지점

> **`libmylib.a` → `-lmylib`**
> 앞의 `lib`와 뒤의 `.a`를 **떼고** 적습니다.
{: .prompt-warning }

| 파일 이름 | 링크 옵션 |
|---|---|
| `libmylib.a` | `-lmylib` |
| `libm.a`(수학) | `-lm` |
| `libz.a`(압축) | `-lz` |

이 규칙 때문에 라이브러리 파일 이름은 **반드시 `lib`로 시작**해야 합니다. `mylib.a`라고 만들면 `-lmylib`로 찾을 수 없습니다.

### 2.4 안을 들여다보기

```powershell
ar t libmylib.a
```

```text
mylib.o
```

```powershell
nm libmylib.a
```

`nm`은 라이브러리 안에 어떤 이름들이 들어 있는지 보여 줍니다. `T`는 그 라이브러리가 **제공하는** 함수, `U`는 **필요로 하는**(아직 없는) 함수입니다. `undefined reference` 오류를 추적할 때 유용합니다.

### 2.5 링크 순서에 주의

```powershell
gcc mylib_main.c -L. -lmylib -o app.exe      # 올바름
```

```powershell
gcc -L. -lmylib mylib_main.c -o app.exe      # 실패할 수 있음
```

링커는 **왼쪽에서 오른쪽으로** 훑으면서 "아직 못 찾은 이름"을 채웁니다. 그래서 **라이브러리는 그것을 쓰는 파일보다 뒤에** 와야 합니다. 이 규칙을 모르면 "분명히 있는데 못 찾는다"는 상황을 만나게 됩니다.

---

## 제3절. `-I`·`-L`·`-l` 세 옵션

세 옵션은 **역할이 완전히 다릅니다.** 오류 메시지와 짝지어 외우면 헷갈리지 않습니다.

| 옵션 | 알려 주는 것 | 언제 필요한가 | 대응하는 오류 |
|---|---|---|---|
| `-I경로` | **헤더**를 찾을 위치 | `#include "foo.h"`가 다른 폴더에 있을 때 | `fatal error: foo.h: No such file or directory` |
| `-L경로` | **라이브러리 파일**을 찾을 위치 | `.a`/`.dll.a`가 다른 폴더에 있을 때 | `cannot find -lfoo` |
| `-l이름` | **어떤 라이브러리**를 링크할지 | 라이브러리의 함수를 쓸 때 | `undefined reference to 'foo_init'` |

### 3.1 오류별 처방

**① `foo.h: No such file or directory`**

컴파일 단계에서 **헤더를 못 찾았습니다.**

```powershell
gcc -Iinclude -c main.c -o main.o
```

**② `cannot find -lfoo`**

링크 단계에서 **라이브러리 파일 자체를 못 찾았습니다.**

```powershell
gcc main.o -Llib -lfoo -o app.exe
```

**③ `undefined reference to 'foo_init'`**

라이브러리는 찾았지만 **그 함수가 없거나, 아예 링크에 넣지 않았습니다.**

| 원인 | 처방 |
|---|---|
| `-lfoo`를 빠뜨림 | 링크 명령에 추가 |
| 라이브러리를 앞에 씀 | 순서를 바꾼다(2.5절) |
| 함수 이름 철자가 다름 | `nm`으로 실제 이름 확인 |
| 헤더에는 선언했는데 구현이 없음 | `.c`에 정의 추가 |

> **세 오류는 발생 단계가 다릅니다.** ①은 컴파일, ②·③은 링크입니다. 오류 메시지를 보고 **어느 단계에서 났는지 먼저 판단**하면 원인을 훨씬 빨리 좁힐 수 있습니다.
{: .prompt-tip }

---

## 제4절. 동적 라이브러리(DLL)

### 4.1 만들기

```powershell
gcc -shared -o libmylib.dll mylib.o -Wl,--out-implib,libmylib.dll.a
```

결과물이 **두 개**입니다.

| 파일 | 언제 필요한가 |
|---|---|
| `libmylib.dll` | **실행할 때** — 실행 파일 옆이나 PATH에 있어야 한다 |
| `libmylib.dll.a` | **링크할 때** — 가져오기 라이브러리(import library) |

```powershell
gcc -Wall -Wextra -std=c17 mylib_main.c -L. -lmylib -o app_dll.exe
```

### 4.2 의존 관계 확인

```powershell
objdump -p app_dll.exe | findstr "DLL Name"
```

```text
DLL Name: libmylib.dll
DLL Name: KERNEL32.dll
DLL Name: api-ms-win-crt-stdio-l1-1-0.dll
```

실행 파일이 **어떤 DLL을 필요로 하는지** 보여 줍니다.

### 4.3 배포의 함정

> **동적 라이브러리는 만들기보다 배포가 어렵습니다.**
> `app_dll.exe`만 복사해 가면 `libmylib.dll`을 찾지 못해 실행되지 않습니다. 게다가 MSYS2로 빌드한 프로그램은 `libgcc_s_seh-1.dll` 같은 **도구 모음의 DLL**까지 필요할 수 있습니다.
>
> 실습이나 과제 제출에는 **정적 라이브러리**를 쓰십시오. 실행 파일 하나만 주면 됩니다.
{: .prompt-warning }

정적으로 완전히 묶으려면 다음 옵션이 도움이 됩니다.

```powershell
gcc main.c -static -o app.exe
```

---

## 제5절. 남이 만든 라이브러리 쓰기

### 5.1 세 가지 유형

| 유형 | 받는 것 | 하는 일 |
|---|---|---|
| **A. 소스 그대로** | `.h`(+`.c`) | 프로젝트에 넣고 함께 컴파일. `#include "foo.h"` |
| **B. 시스템에 설치** | 표준 경로의 헤더·라이브러리 | 꾸러미 관리자로 설치 후 `-l이름`만 추가 |
| **C. 남이 빌드해 배포** | `.h` + `.a`/`.dll` | `-I`·`-L`·`-l`을 모두 지정, DLL이면 함께 배포 |

### 5.2 유형 B — MSYS2로 설치해 쓰기

MSYS2 UCRT64 터미널에서 설치합니다.

```bash
pacman -S mingw-w64-ucrt-x86_64-zlib
```

설치하면 헤더와 라이브러리가 **표준 경로**에 들어가므로, `-I`·`-L` 없이 `-l`만 붙이면 됩니다.

```c
#include <stdio.h>
#include <string.h>
#include <zlib.h>          /* 남이 만든 라이브러리의 헤더 */

int main(void)
{
    const char *src = "hello hello hello hello hello hello hello hello";
    unsigned char dest[128];
    uLongf destLen = sizeof dest;
    uLong srcLen = (uLong) strlen(src) + 1;

    printf("zlib 버전   : %s\n", zlibVersion());
    if (compress(dest, &destLen, (const Bytef *) src, srcLen) == Z_OK)
        printf("압축 결과   : %lu 바이트 -> %lu 바이트\n",
               (unsigned long) srcLen, (unsigned long) destLen);
    else
        printf("압축 실패\n");
    return 0;
}
```

```powershell
gcc -Wall -Wextra -std=c17 zdemo.c -lz -o zdemo.exe
```

```text
zlib 버전   : 1.3.2
압축 결과   : 48 바이트 -> 18 바이트
```

**한 줄도 짜지 않은 압축 알고리즘을 그대로 썼습니다.** 이것이 라이브러리를 쓰는 이유입니다.

### 5.3 `pkg-config`

라이브러리가 커지면 필요한 옵션이 여러 개가 됩니다. `pkg-config`가 대신 알려 줍니다.

```powershell
pkg-config --cflags --libs zlib
```

```text
-IC:/msys64/ucrt64/bin/../include -LC:/msys64/ucrt64/bin/../lib -lz
```

`Makefile`에서는 이렇게 씁니다.

```makefile
CFLAGS  += $(shell pkg-config --cflags zlib)
LDLIBS  += $(shell pkg-config --libs zlib)
```

---

> **▶ 여기서부터 2회차 — 헤더 설계와 `Makefile`**
> 제6절 ~ 제10절, 약 175분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제6절. 헤더 설계

### 6.1 헤더는 사용 설명서입니다

**헤더에는 밖에서 써야 할 것만** 적습니다. 나머지는 `.c` 안에 `static`으로 감춥니다(5강 7.1절).

| 헤더에 넣을 것 | 헤더에 넣지 말 것 |
|---|---|
| 공개 함수의 **원형** | 함수의 **구현** |
| 공개 자료형(`struct`, `typedef`, `enum`) | 내부에서만 쓰는 자료형 |
| 공개 상수(`#define`, `enum`) | 내부 상수 |
| `extern` 변수 **선언** | 변수 **정의**(중복 정의 오류!) |

### 6.2 반드시 지킬 세 가지

**① 중복 포함 방지**

```c
#ifndef MYLIB_H
#define MYLIB_H
   ...
#endif   /* MYLIB_H */
```

**② 변수를 정의하지 않는다**

```c
int counter;              /* 헤더에 이렇게 쓰면 여러 파일에서 중복 정의 */
extern int counter;       /* 올바름 — 선언만. 정의는 .c 한 곳에 */
```

**③ 문서화 주석**

```c
/* trim: 문자열 끝의 공백·탭·개행을 잘라 내고 새 길이를 돌려준다.
   s 는 수정된다. 앞쪽 공백은 건드리지 않는다. */
int trim(char s[]);
```

**사용하는 사람은 헤더만 읽습니다.** 그러므로 **무엇을 하는지, 인자를 고치는지, 실패하면 무엇을 돌려주는지**를 여기에 적어야 합니다.

### 6.3 좋은 공개 인터페이스의 조건

10강에서 배운 원칙이 그대로 적용됩니다.

| 원칙 | 예 |
|---|---|
| **크기를 함께 받는다** | `int copy_name(char *dst, size_t dstsize, const char *src);` |
| **고치지 않는 인자는 `const`** | `int binsearch(int x, const int v[], int n);` |
| **실패를 알릴 방법을 둔다** | 반환값 0/1, 또는 `NULL` |
| **이름에 접두어를 붙인다** | `cstudy_trim`처럼 — 다른 라이브러리와 충돌을 막는다 |

---

## 제7절. `Makefile` 기초

### 7.1 왜 필요한가

파일이 늘어나면 명령을 손으로 치기 어려워집니다.

```powershell
gcc -Wall -Wextra -std=c17 -c mylib.c -o mylib.o
gcc -Wall -Wextra -std=c17 -c strutil.c -o strutil.o
gcc -Wall -Wextra -std=c17 -c main.c -o main.o
ar rcs libcstudy.a mylib.o strutil.o
gcc main.o -L. -lcstudy -o app.exe
```

`Makefile`은 이 절차를 적어 두고 **바뀐 것만 다시 빌드**해 줍니다.

### 7.2 규칙의 형태

```makefile
목표: 재료
<탭>명령
```

> **명령 앞은 반드시 탭 문자여야 합니다.** 공백 여덟 칸은 안 됩니다. `Makefile`에서 가장 흔한 오류이며, `missing separator` 라는 메시지로 나타납니다. VS Code에서 `Makefile`을 편집할 때 탭이 공백으로 바뀌지 않도록 주의하십시오.
{: .prompt-danger }

### 7.3 가장 단순한 형태

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -std=c17

app.exe: main.o mylib.o
	$(CC) main.o mylib.o -o app.exe

main.o: main.c mylib.h
	$(CC) $(CFLAGS) -c main.c -o main.o

mylib.o: mylib.c mylib.h
	$(CC) $(CFLAGS) -c mylib.c -o mylib.o

clean:
	del /Q *.o app.exe
```

```powershell
make
```

```powershell
make clean
```

### 7.4 자동 변수와 패턴 규칙

같은 내용을 반복하지 않도록 줄일 수 있습니다.

| 자동 변수 | 뜻 |
|---|---|
| `$@` | 목표 이름 |
| `$<` | **첫 번째** 재료 |
| `$^` | 재료 **전체** |

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -std=c17
OBJS    = main.o mylib.o strutil.o
TARGET  = app.exe

$(TARGET): $(OBJS)
	$(CC) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	del /Q *.o $(TARGET)
```

`%.o: %.c`는 **패턴 규칙**입니다. "어떤 `.o`든 같은 이름의 `.c`로부터 이렇게 만들어라"는 뜻이므로, 파일이 늘어도 규칙을 더 쓸 필요가 없습니다.

### 7.5 라이브러리까지 만드는 `Makefile`

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -std=c17
AR      = ar
LIB     = libcstudy.a
LIBOBJS = strutil.o numutil.o
TARGET  = app.exe

all: $(TARGET)

$(TARGET): main.o $(LIB)
	$(CC) main.o -L. -lcstudy -o $@

$(LIB): $(LIBOBJS)
	$(AR) rcs $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	del /Q *.o *.a $(TARGET)

.PHONY: all clean
```

| 요소 | 설명 |
|---|---|
| `all` | 기본 목표. `make`만 치면 이것이 만들어진다 |
| `.PHONY` | **파일이 아니라 이름표**임을 알린다. `clean`이라는 파일이 있어도 동작한다 |
| `$(LIB)` | 라이브러리도 하나의 목표가 된다 |

> Windows에서는 `del /Q`, 리눅스·MSYS2 셸에서는 `rm -f`를 씁니다. PowerShell에서 `make`를 쓰려면 MSYS2에서 `pacman -S make`로 설치한 뒤 `mingw32-make` 또는 `make`를 사용합니다.
{: .prompt-info }

---

## 제8절. 의존 관계와 증분 빌드

### 8.1 `make`는 시각을 비교합니다

`make`의 동작은 단순합니다.

> **목표 파일이 재료보다 오래되었으면** 명령을 실행한다.

```makefile
main.o: main.c mylib.h
```

이 줄은 "`main.o`는 `main.c`와 `mylib.h`에 달려 있다"는 뜻입니다. 그러므로

| 무엇을 고쳤나 | 다시 빌드되는 것 |
|---|---|
| `main.c` | `main.o` → `app.exe` |
| `mylib.c` | `mylib.o` → `libcstudy.a` → `app.exe` |
| **`mylib.h`** | **그 헤더를 쓰는 모든 `.o`** → 전부 |
| 아무것도 안 고침 | `make: 'app.exe' is up to date.` |

### 8.2 헤더 의존을 빠뜨리면

```makefile
main.o: main.c            # mylib.h 를 빠뜨렸다
```

이렇게 쓰면 **헤더를 고쳐도 다시 빌드되지 않습니다.** 그러면 새 헤더와 옛 목적 파일이 섞여 **원인을 알 수 없는 오류**가 납니다. 5강 13절에서 "헤더를 고쳤는데 반영되지 않음"으로 적어 둔 그 문제입니다.

**해결**: 컴파일러가 의존 관계를 만들어 주게 합니다.

```makefile
CFLAGS += -MMD -MP
-include $(OBJS:.o=.d)
```

`-MMD`는 컴파일할 때 `.d` 파일에 의존 관계를 적어 주고, `-include`는 그것을 읽어 들입니다. **헤더 의존을 사람이 관리하지 않아도 됩니다.**

### 8.3 자주 쓰는 명령

| 명령 | 하는 일 |
|---|---|
| `make` | 기본 목표(`all`)를 만든다 |
| `make clean` | 만들어진 것을 지운다 |
| `make app.exe` | 특정 목표만 만든다 |
| `make -n` | **실행하지 않고** 무엇을 할지 보여 준다 |
| `make -B` | 최신이든 아니든 **강제로** 다시 만든다 |

`make -n`은 `Makefile`을 고쳤을 때 **먼저 확인해 보는 습관**으로 삼을 만합니다.

---

## 제9절. 실습 — `libcstudy` 만들기

이번 과정에서 만든 함수들을 하나의 라이브러리로 묶습니다.

### 9.1 구성

| 파일 | 내용 | 출처 |
|---|---|---|
| `cstudy.h` | 공개 원형 전체 | — |
| `strutil.c` | `trim`, `reverse`, `safe_append` | 4강 · 10강 |
| `numutil.c` | `binsearch`, `my_itoa`, `add_checked` | 4강 · 10강 |
| `main.c` | 시험용 프로그램 | — |
| `Makefile` | 빌드 자동화 | 7.5절 |

### 9.2 헤더

```c
#ifndef CSTUDY_H
#define CSTUDY_H

#include <stddef.h>     /* size_t */

/* --- 문자열 --- */

/* 문자열 끝의 공백·탭·개행을 잘라 내고 새 길이를 돌려준다. s 는 수정된다 */
int cstudy_trim(char s[]);

/* 문자열을 제자리에서 뒤집는다 */
void cstudy_reverse(char s[]);

/* dst 에 src 를 이어 붙인다. 자리가 모자라면 0 을 돌려주고 dst 는 그대로 둔다 */
int cstudy_append(char *dst, size_t dstsize, const char *src);

/* --- 수 --- */

/* 오름차순 배열 v 에서 x 를 찾아 첨자를 돌려준다. 없으면 -1 */
int cstudy_binsearch(int x, const int v[], int n);

/* 정수 n 을 문자열 s 로 바꾼다. s 는 최소 12칸이어야 한다 */
void cstudy_itoa(int n, char s[]);

/* 넘치지 않으면 a + b 를 *out 에 넣고 1 을, 넘치면 0 을 돌려준다 */
int cstudy_add_checked(int a, int b, int *out);

#endif   /* CSTUDY_H */
```

**이름 앞에 `cstudy_`를 붙인 것**에 주목하십시오(6.3절). 표준 라이브러리나 다른 라이브러리와 충돌하지 않습니다. 2강에서 `get_line`을 `getline`으로 짓지 않은 것과 같은 이유입니다.

### 9.3 빌드와 사용

```powershell
make
```

```text
gcc -Wall -Wextra -std=c17 -c strutil.c -o strutil.o
gcc -Wall -Wextra -std=c17 -c numutil.c -o numutil.o
ar rcs libcstudy.a strutil.o numutil.o
gcc -Wall -Wextra -std=c17 -c main.c -o main.o
gcc main.o -L. -lcstudy -o app.exe
```

한 파일만 고치고 다시 실행하면

```powershell
make
```

```text
gcc -Wall -Wextra -std=c17 -c strutil.c -o strutil.o
ar rcs libcstudy.a strutil.o numutil.o
gcc main.o -L. -lcstudy -o app.exe
```

**`numutil.c`와 `main.c`는 다시 컴파일되지 않았습니다.** 이것이 증분 빌드입니다.

### 9.4 다른 프로젝트에서 쓰기

라이브러리와 헤더를 옮겨 두고 경로를 지정하면 됩니다.

```powershell
gcc -Wall -Wextra -std=c17 -I..\lab11 newprog.c -L..\lab11 -lcstudy -o newprog.exe
```

**이제 소스를 복사해 붙여 넣을 필요가 없습니다.** 버그를 고치면 라이브러리만 다시 만들어 배포하면 됩니다.

---

## 제10절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| `cannot find -lmylib` | 파일 이름이 `lib`로 시작하지 않음 | `libmylib.a`로 이름 짓기 |
| `undefined reference` | `-l`을 소스보다 앞에 씀 | 라이브러리는 **뒤에** |
| `undefined reference` | 라이브러리에 그 함수가 없음 | `nm`으로 확인 |
| `foo.h: No such file` | 헤더 경로 미지정 | `-I경로` |
| 헤더를 고쳐도 반영 안 됨 | `Makefile`에 헤더 의존 누락 | `-MMD -MP` 사용 |
| `missing separator` | 명령 앞이 **공백** | **탭**으로 |
| `.exe`만 복사했더니 실행 안 됨 | DLL 의존 | 정적 링크 또는 DLL 동봉 |
| 라이브러리를 새로 만들었는데 옛 동작 | `.a`를 다시 만들지 않음 | `make clean` 후 다시 |
| 이름이 충돌함 | 흔한 이름 사용 | 접두어를 붙인다 |
| 헤더에서 중복 정의 오류 | 헤더에 변수를 **정의** | `extern` 선언만 |

---

> **▶ 여기서부터 3회차 — 스스로 해 보기**
> 실습문제, 약 100분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 실습문제

> **안내**
> 1. 모든 문제는 `C:\c-study\lab11`에서 수행합니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다.
> 3. 빌드 명령과 결과 화면을 함께 제출하십시오.
{: .prompt-info }

| 문제 | 주제 | 대응 절 |
|---|---|---|
| 문제 1 | 정적 라이브러리 만들기 | 2 |
| 문제 2 | 세 가지 링크 오류 재현 | 3 |
| 문제 3 | 라이브러리 내용 살펴보기 | 2.4 |
| 문제 4 | 최소 `Makefile` 작성 | 7.3 |
| 문제 5 | 패턴 규칙으로 줄이기 | 7.4 |
| 문제 6 | 헤더 의존 관계 확인 | 8.2 |
| 문제 7 | 헤더 설계 검토 | 6 |
| 문제 8 | 남의 라이브러리 사용 | 5.2 |
| 문제 9 | DLL 만들고 배포 확인 | 4 |
| 문제 10 | `libcstudy` 완성 | 9 |

---

### 문제 1. 정적 라이브러리 만들기

`strutil.c`와 `numutil.c`를 묶어 `libcstudy.a`를 만들고 시험 프로그램에서 사용하십시오.

**정답 및 해설**

```powershell
gcc -Wall -Wextra -std=c17 -c strutil.c -o strutil.o
```

```powershell
gcc -Wall -Wextra -std=c17 -c numutil.c -o numutil.o
```

```powershell
ar rcs libcstudy.a strutil.o numutil.o
```

```powershell
gcc -Wall -Wextra -std=c17 main.c -L. -lcstudy -o app.exe
```

- **`-c`로 목적 파일을 먼저** 만들고, `ar`로 묶고, 마지막에 링크합니다. 이 세 단계가 라이브러리 만들기의 전부입니다.
- `-L.`의 `.`은 **현재 폴더**를 뜻합니다. 이것을 빠뜨리면 표준 경로에서만 찾아 `cannot find -lcstudy`가 납니다.
- 라이브러리를 다시 만든 뒤에는 **실행 파일도 다시 링크**해야 반영됩니다.

---

### 문제 2. 세 가지 링크 오류 재현

`-I`·`-L`·`-l`을 각각 빠뜨려 세 가지 오류를 **일부러 재현**하고, 메시지와 원인을 표로 정리하십시오.

**정답 및 해설**

| 일부러 한 일 | 나타나는 메시지 | 단계 |
|---|---|---|
| 헤더를 다른 폴더에 두고 `-I` 생략 | `fatal error: cstudy.h: No such file or directory` | **컴파일** |
| `-L.` 생략 | `cannot find -lcstudy` | **링크** |
| `-lcstudy` 생략 | `undefined reference to 'cstudy_trim'` | **링크** |

- **오류가 난 단계를 먼저 판단**하는 것이 요령입니다. `fatal error: ... No such file`은 언제나 컴파일 단계이고, `cannot find -l...`과 `undefined reference`는 링크 단계입니다.
- `undefined reference`가 나면 순서를 의심하십시오. `gcc -lcstudy main.c` 처럼 앞에 두면 같은 오류가 납니다(2.5절).
- 이 세 가지를 직접 겪어 두면 실무에서 처음 보는 라이브러리를 붙일 때 훨씬 수월합니다.

---

### 문제 3. 라이브러리 내용 살펴보기

`ar t`와 `nm`으로 `libcstudy.a` 안을 확인하고, `T`와 `U` 표시의 뜻을 설명하십시오.

**정답 및 해설**

```powershell
ar t libcstudy.a
```

```text
strutil.o
numutil.o
```

```powershell
nm libcstudy.a
```

| 표시 | 뜻 |
|---|---|
| `T` | 이 라이브러리가 **제공하는** 함수(text 영역) |
| `U` | 이 라이브러리가 **필요로 하는**(밖에서 와야 하는) 이름 |
| `t` | `static` 함수 — **밖에서 보이지 않는다** |
| `B`/`b` | 초기화되지 않은 전역 변수 |

- 소문자 `t`가 5강 7.1절에서 배운 **감추기(`static`)** 의 결과입니다. 밖에서 링크할 수 없습니다.
- `U`로 나오는 이름은 대개 `printf` 같은 표준 라이브러리 함수입니다. 그것들은 링크할 때 표준 라이브러리에서 채워집니다.
- `undefined reference`가 났을 때 **이름의 철자를 확인**하는 데 `nm`이 가장 빠릅니다.

---

### 문제 4·5. `Makefile` 작성과 개선

먼저 규칙을 하나씩 적은 `Makefile`을 만들고, 그다음 **패턴 규칙과 자동 변수**로 줄이십시오.

**정답 및 해설**

**처음 판(문제 4)**

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -std=c17

app.exe: main.o mylib.o
	$(CC) main.o mylib.o -o app.exe

main.o: main.c mylib.h
	$(CC) $(CFLAGS) -c main.c -o main.o

mylib.o: mylib.c mylib.h
	$(CC) $(CFLAGS) -c mylib.c -o mylib.o

clean:
	del /Q *.o app.exe
```

**줄인 판(문제 5)**

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -std=c17
OBJS    = main.o mylib.o strutil.o
TARGET  = app.exe

$(TARGET): $(OBJS)
	$(CC) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	del /Q *.o $(TARGET)
```

- **`$^`(재료 전체)와 `$@`(목표), `$<`(첫 재료)** 세 개만 알면 대부분의 `Makefile`을 읽을 수 있습니다.
- 패턴 규칙 덕분에 **소스가 늘어도 `OBJS`에 한 줄만 추가**하면 됩니다.
- 다만 줄인 판에는 **헤더 의존이 사라졌습니다.** 문제 6에서 이것을 보완합니다.
- 명령 앞은 반드시 **탭**입니다. 편집기 설정을 확인하십시오.

---

### 문제 6. 헤더 의존 관계 확인

`mylib.h`만 고치고 `make`를 실행해 보십시오. 다시 빌드되지 않는다면 원인을 찾아 고치십시오.

**정답 및 해설**

패턴 규칙만 쓰면 `make`는 **헤더가 바뀐 것을 알지 못합니다.** 확인하는 방법은 간단합니다.

```powershell
make
```

```text
make: 'app.exe' is up to date.
```

헤더를 고쳤는데 이렇게 나오면 문제가 있는 것입니다.

**해결**

```makefile
CFLAGS += -MMD -MP
OBJS    = main.o mylib.o strutil.o

-include $(OBJS:.o=.d)
```

- `-MMD`는 컴파일할 때 `main.d` 같은 파일에 **"main.o는 main.c와 mylib.h에 달려 있다"** 를 자동으로 적어 둡니다.
- `-include`는 그 파일이 없어도 오류를 내지 않고 넘어갑니다. 첫 빌드에서는 `.d`가 아직 없기 때문에 필요합니다.
- `-MP`는 헤더가 삭제되었을 때 생기는 오류를 막아 줍니다.
- 이 세 줄이 5강 13절의 "헤더를 고쳤는데 반영되지 않음" 문제를 **영구히** 해결합니다.

---

### 문제 7. 헤더 설계 검토

다음 헤더의 문제점을 모두 찾아 고치십시오.

```c
#include <stdio.h>

int counter;

int trim(char s[]) {
    /* 구현이 헤더에 들어 있다 */
    return 0;
}

void helper(void);
```

**정답 및 해설**

| 문제 | 이유 | 고침 |
|---|---|---|
| 중복 포함 방지 없음 | 두 번 포함되면 오류 | `#ifndef`·`#define`·`#endif` |
| `int counter;` — **정의** | 여러 파일에 포함되면 중복 정의 | `extern int counter;`(정의는 `.c`에) |
| 함수 **구현**이 헤더에 | 포함하는 파일마다 중복 정의 | 원형만 남기고 구현은 `.c`로 |
| `helper` 공개 | 내부 전용 함수를 노출 | 헤더에서 빼고 `.c`에 `static` |
| 설명 주석 없음 | 쓰는 사람이 동작을 알 수 없다 | 각 원형 위에 주석 |

```c
#ifndef MYLIB_H
#define MYLIB_H

extern int counter;      /* 정의는 mylib.c 에 한 번만 */

/* trim: 문자열 끝의 공백·탭·개행을 잘라 내고 새 길이를 돌려준다.
   s 는 수정된다. 앞쪽 공백은 건드리지 않는다. */
int trim(char s[]);

#endif   /* MYLIB_H */
```

- `#include <stdio.h>`도 **정말 필요할 때만** 둡니다. 헤더가 다른 헤더를 끌어들이면 빌드가 느려지고 의존이 얽힙니다. 여기서는 필요 없으므로 제거했습니다.
- `size_t` 같은 자료형이 필요하면 `<stddef.h>`처럼 **가장 작은 헤더**를 포함합니다.

---

### 문제 8. 남의 라이브러리 사용

`zlib`을 설치하여 문자열을 압축하는 프로그램을 만들고, 압축 전후 크기를 비교하십시오.

**정답 및 해설**

```bash
pacman -S mingw-w64-ucrt-x86_64-zlib
```

```powershell
gcc -Wall -Wextra -std=c17 zdemo.c -lz -o zdemo.exe
```

```text
zlib 버전   : 1.3.2
압축 결과   : 48 바이트 -> 18 바이트
```

- **`-I`·`-L`이 필요 없었던 이유**는 꾸러미 관리자가 헤더와 라이브러리를 **표준 경로**에 설치했기 때문입니다(5.1절 유형 B).
- 같은 문자열이 반복되어 있어 48바이트가 18바이트로 줄었습니다. 압축은 **반복을 찾아 줄이는** 일입니다.
- 이 프로그램에서 우리가 한 일은 **헤더를 읽고 함수를 규칙대로 호출한 것**뿐입니다. 라이브러리를 쓴다는 것은 곧 **남이 쓴 헤더를 읽을 줄 아는 것**입니다.

---

### 문제 9. DLL 만들고 배포 확인

`libcstudy`를 DLL로 만들고, **`.exe`만 다른 폴더로 복사**했을 때 어떻게 되는지 확인하십시오.

**정답 및 해설**

```powershell
gcc -shared -o libcstudy.dll strutil.o numutil.o -Wl,--out-implib,libcstudy.dll.a
```

```powershell
gcc -Wall -Wextra -std=c17 main.c -L. -lcstudy -o app_dll.exe
```

`.exe`만 다른 폴더로 복사해 실행하면 **DLL을 찾을 수 없다는 오류 창**이 뜨며 실행되지 않습니다.

| 확인 방법 | 명령 |
|---|---|
| 무엇을 필요로 하는지 | `objdump -p app_dll.exe \| findstr "DLL Name"` |
| 해결 1 | `.dll`을 `.exe` 옆에 복사 |
| 해결 2 | 정적 라이브러리로 다시 빌드 |
| 해결 3 | `-static`으로 통째로 묶기 |

- **이것이 4.3절에서 말한 배포의 함정**입니다. 개발 PC에서는 잘 돌아가는데 다른 PC에서 실행되지 않는 대표적인 이유입니다.
- 과제 제출에는 정적 라이브러리를 쓰십시오.

---

### 문제 10. `libcstudy` 완성

9절의 구성대로 라이브러리를 완성하고, **다른 폴더의 새 프로그램**에서 사용하십시오.

**정답 및 해설**

제출물은 다음과 같아야 합니다.

| 파일 | 확인할 점 |
|---|---|
| `cstudy.h` | 중복 포함 방지, 접두어, 설명 주석, 크기 인자, `const` |
| `strutil.c`·`numutil.c` | 내부 함수는 `static`, 경고 0개 |
| `Makefile` | 패턴 규칙, `-MMD -MP`, `clean`, `.PHONY` |
| `libcstudy.a` | `ar t`로 목적 파일 두 개 확인 |
| 사용 예 | `-I`·`-L`·`-l`로 다른 폴더에서 빌드 |

```powershell
gcc -Wall -Wextra -std=c17 -I..\lab11 newprog.c -L..\lab11 -lcstudy -o newprog.exe
```

- **여기까지 하면 여러분의 코드는 "재사용 가능한 부품"이 됩니다.** 이 과정에서 만든 모든 함수를 앞으로 계속 쓸 수 있습니다.
- 12강 종합 과제에서 이 라이브러리를 그대로 사용합니다.

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | `libcstudy` 일습 — `cstudy.h`, `strutil.c`, `numutil.c`, `main.c`, `Makefile`, `libcstudy.a` |
| 2 | 빌드 화면 — 처음 `make`와, 한 파일만 고친 뒤의 `make`(증분 빌드가 보이도록) |
| 3 | 문제 2의 **세 가지 오류 재현 화면** |
| 4 | `ar t`·`nm` 출력 |
| 5 | 짧은 서술 ① `-I`·`-L`·`-l`의 역할과 각각의 오류 메시지 |
| 6 | 짧은 서술 ② 정적 라이브러리와 DLL 중 이 과제에 무엇이 적절한지, 이유와 함께 |

---

## 정리

| 구분 | 핵심 |
|---|---|
| 흐름 | `.c` → `.o` → `.a` → `.exe` |
| 만들기 | `gcc -c` → `ar rcs libNAME.a *.o` |
| 이름 규칙 | **`libNAME.a` ↔ `-lNAME`** |
| 링크 순서 | 라이브러리는 **쓰는 파일보다 뒤에** |
| 세 옵션 | `-I` 헤더 / `-L` 라이브러리 위치 / `-l` 라이브러리 이름 |
| 오류 판별 | `No such file`=컴파일, `cannot find -l`·`undefined reference`=링크 |
| DLL | 실행할 때 필요. **배포가 어렵다** |
| 헤더 | 공개할 것만, 중복 포함 방지, 변수는 `extern`, 설명 주석 |
| `Makefile` | 규칙·변수·자동 변수(`$@ $< $^`)·패턴 규칙, 명령 앞은 **탭** |
| 증분 빌드 | 시각 비교로 바뀐 것만. **헤더 의존은 `-MMD -MP`** |

이제 코드를 **부품으로 만들고 자동으로 조립**할 수 있게 되었습니다. 문법(1~9강), 안전(10강), 조립(11강)이 모두 갖추어졌습니다.

**다음 12강은 종합 과제**입니다. 지금까지 배운 모든 것을 하나의 프로그램으로 모읍니다. 구조체로 자료를 담고, 파일로 읽고 쓰며, 안전한 입력 처리를 하고, 라이브러리로 나누어 `Makefile`로 빌드하는 **완결된 프로그램**을 직접 만들게 됩니다.

---

## 부록 A. 명령 요약

| 하려는 일 | 명령 |
|---|---|
| 목적 파일 만들기 | `gcc -Wall -Wextra -std=c17 -c foo.c -o foo.o` |
| 정적 라이브러리 만들기 | `ar rcs libfoo.a foo.o bar.o` |
| 라이브러리 내용 보기 | `ar t libfoo.a` |
| 심벌 보기 | `nm libfoo.a` |
| 정적 링크 | `gcc main.c -L. -lfoo -o app.exe` |
| DLL 만들기 | `gcc -shared -o libfoo.dll foo.o -Wl,--out-implib,libfoo.dll.a` |
| DLL 의존 확인 | `objdump -p app.exe \| findstr "DLL Name"` |
| 통째로 정적 링크 | `gcc main.c -static -o app.exe` |
| 옵션 알아내기 | `pkg-config --cflags --libs zlib` |
| 빌드 | `make` / `make clean` / `make -n` |

## 부록 B. `Makefile` 뼈대 (그대로 쓸 수 있는 형태)

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -std=c17 -MMD -MP
AR      = ar
LIB     = libcstudy.a
LIBOBJS = strutil.o numutil.o
OBJS    = main.o $(LIBOBJS)
TARGET  = app.exe

all: $(TARGET)

$(TARGET): main.o $(LIB)
	$(CC) main.o -L. -lcstudy -o $@

$(LIB): $(LIBOBJS)
	$(AR) rcs $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	del /Q *.o *.d *.a $(TARGET)

-include $(OBJS:.o=.d)

.PHONY: all clean
```
