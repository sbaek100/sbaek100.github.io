---
title: C 프로그래밍 기초 9강 - 입출력과 파일
date: 2026-10-19 09:00:00 +0900
categories:
  - 0.기초강의
  - C 프로그래밍 기초
tags:
  - C언어
  - KnR
  - 입출력
  - printf
  - scanf
  - 파일
  - fopen
  - fgets
  - stderr
  - exit
  - 가변인자
  - 표준라이브러리
pin:
mermaid: false
---

> **학습 목표**
> 1. 표준 입력·출력·오류의 세 흐름을 구분하고 리다이렉션·파이프로 연결할 수 있다.
> 2. `printf`의 서식을 폭·정밀도까지 정확히 지정할 수 있고, 반환값의 뜻을 설명할 수 있다.
> 3. `sprintf`의 위험과 `snprintf`의 안전한 사용법을 설명할 수 있다.
> 4. 가변 인자 함수를 `<stdarg.h>`로 작성할 수 있다.
> 5. `scanf`·`sscanf`의 동작과 반환값을 정확히 이해하고 안전하게 사용할 수 있다.
> 6. `fopen`·`fclose`로 파일을 열고 닫으며, 모드(`"r"`·`"w"`·`"a"`)를 구분할 수 있다.
> 7. `getc`·`putc`·`fprintf`·`fscanf`로 파일을 읽고 쓸 수 있다.
> 8. 오류 메시지를 `stderr`로 보내고 `exit`로 상태를 알릴 수 있다.
> 9. `fgets`·`fputs`로 줄 단위 입출력을 하고, **`gets`를 쓰지 않아야 하는 이유**를 설명할 수 있다.
> 10. 표준 라이브러리의 주요 함수군(문자열·문자·수학·메모리·난수)을 찾아 쓸 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

지금까지 만든 프로그램은 **실행이 끝나면 모든 것을 잃어버립니다.** 8강에서 구조체로 학생 정보를 잘 정리해 두어도, 프로그램이 끝나면 사라집니다. 다음에 다시 쓰려면 사람이 또 입력해야 합니다.

이번 강의에서는 **파일**을 다룹니다. 그러면 프로그램이 만든 결과를 남겨 두었다가 다시 읽어 올 수 있습니다.

동시에 그동안 미뤄 두었던 **입출력의 정확한 규칙**도 정리합니다.

| 미뤄 둔 것 | 나온 곳 |
|---|---|
| `printf`의 폭과 정밀도 전체 | 3강 5절(맛보기) |
| `scanf`의 정식 사용법 | 4강 부록 A(최소한만) |
| `%s` 대신 `fgets`를 쓰라던 이유 | 4강 부록 A |
| 표준 라이브러리에 무엇이 있는가 | 여러 강의에 흩어짐 |

> **참고 문헌**
> 이번 강의는 다음 책의 **제7장 Input and Output** 전체를 바탕으로 재구성한 것입니다.
> Brian W. Kernighan, Dennis M. Ritchie, *The C Programming Language*, 2nd Edition, Prentice Hall, 1988.
> 원서 제8장(UNIX 시스템 인터페이스)은 Windows 실습 환경과 맞지 않아 이 과정에서는 다루지 않습니다.
{: .prompt-info }

| K&R | 원서 절 제목 | 이번 강의 |
|---|---|---|
| 7.1 | Standard Input and Output | 제1절 |
| 7.2 | Formatted Output — printf | 제2절 |
| 7.3 | Variable-length Argument Lists | 제3절 |
| 7.4 | Formatted Input — Scanf | 제4절 |
| 7.5 | File Access | 제5절 |
| 7.6 | Error Handling — Stderr and Exit | 제6절 |
| 7.7 | Line Input and Output | 제7절 |
| 7.8 | Miscellaneous Functions | 제8절 |
| — | (우리 추가) 구조체를 파일로 다루기 | 제9절 |

이 강의는 **3회차 분량**(모두 합쳐 약 470분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제4절 | 표준 입출력과 서식 | 150분 |
| **2회차** | 제5절 ~ 제8절 | 파일·오류 처리·줄 단위 입출력 | 155분 |
| **3회차** | 제9절 ~ 실습문제 | 구조체 파일 다루기와 실습 | 165분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 표준 입출력과 리다이렉션 | 30분 |
| 제2절 | `printf` 정확히 | 45분 |
| 제3절 | 가변 인자 함수 | 30분 |
| 제4절 | `scanf`와 `sscanf` | 45분 |
| 제5절 | 파일 열고 읽고 쓰기 | 55분 |
| 제6절 | 오류 처리 — `stderr`와 `exit` | 35분 |
| 제7절 | 줄 단위 입출력 | 35분 |
| 제8절 | 표준 라이브러리 둘러보기 | 30분 |
| 제9절 | 구조체를 파일로 다루기 | 40분 |
| 제10절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 폴더**는 `C:\c-study\lab09`를 사용합니다.

```powershell
mkdir C:\c-study\lab09
```

---

## 제1절. 표준 입출력과 리다이렉션

### 1.1 입출력은 언어가 아니라 라이브러리입니다

2강 1.1절에서 이야기한 대로 **C 언어 자체에는 입출력이 없습니다.** 모두 표준 라이브러리가 제공하는 함수이며, 그래서 `#include <stdio.h>`가 필요합니다.

라이브러리가 제공하는 모형은 단순합니다. 자료는 **문자들의 흐름(스트림)** 이며, 각 줄은 개행 문자로 끝납니다. 실제 저장 방식이 환경마다 달라도 **라이브러리가 그 차이를 감춰 줍니다.**

### 1.2 세 개의 흐름

프로그램이 시작되면 **세 개의 흐름이 이미 열려 있습니다.**

| 이름 | 뜻 | 기본 연결 | 용도 |
|---|---|---|---|
| `stdin` | 표준 입력 | 키보드 | 자료를 읽는다 |
| `stdout` | 표준 출력 | 화면 | **결과**를 내보낸다 |
| `stderr` | 표준 오류 | 화면 | **오류 메시지**를 내보낸다 |

`getchar()`는 `stdin`에서 읽고, `printf`와 `putchar`는 `stdout`으로 씁니다.

**`stdout`과 `stderr`가 따로 있는 이유**는 제6절에서 자세히 다룹니다. 요점은 **결과와 오류를 갈라 놓아야 결과만 파일로 저장할 수 있다**는 것입니다.

### 1.3 리다이렉션과 파이프

2강 부록 B에서 이미 써 온 방법을 정리합니다.

| 하려는 일 | PowerShell |
|---|---|
| 파일을 입력으로 | `.\prog.exe < in.txt` |
| 출력을 파일로 | `.\prog.exe > out.txt` |
| 둘 다 | `.\prog.exe < in.txt > out.txt` |
| 앞 프로그램의 출력을 뒤 프로그램의 입력으로 | `.\prog1.exe \| .\prog2.exe` |

**프로그램은 이 사실을 전혀 모릅니다.** `getchar`는 키보드에서 읽는지 파일에서 읽는지 구분하지 않으며, `"< in.txt"` 같은 문자열이 `argv`에 들어오지도 않습니다. 연결은 **셸이 대신 해 줍니다.**

`lower.c`는 이 방식만으로도 쓸모 있는 프로그램이 됩니다.

```c
#include <stdio.h>
#include <ctype.h>

/* lower: 입력을 모두 소문자로 바꾸어 출력한다 */
int main(void)
{
    int c;

    while ((c = getchar()) != EOF)
        putchar(tolower(c));
    return 0;
}
```

```powershell
.\lower.exe < poem.txt > poem_lower.txt
```

---

## 제2절. `printf` 정확히

### 2.1 서식 지정자의 완전한 형태

```text
%[플래그][폭][.정밀도][길이]변환문자
```

| 자리 | 뜻 |
|---|---|
| 플래그 | `-` 왼쪽 정렬, `0` 0으로 채움, `+` 부호 표시 |
| 폭 | **최소** 출력 폭. 모자라면 채우고, 넘치면 무시된다 |
| 정밀도 | 실수는 소수점 이하 자릿수, **문자열은 최대 글자 수** |
| 길이 | `h`(short), `l`(long), `ll`(long long), `z`(size_t) |
| 변환문자 | 아래 표 |

| 변환문자 | 인자 자료형 | 출력 형태 |
|---|---|---|
| `d`, `i` | `int` | 10진 정수 |
| `o` | `int` | 8진수(접두어 없음) |
| `x`, `X` | `int` | 16진수(접두어 없음), 소문자/대문자 |
| `u` | `unsigned` | 부호 없는 10진수 |
| `c` | `int` | 문자 한 개 |
| `s` | `char *` | `'\0'`까지, 또는 정밀도만큼 |
| `f` | `double` | `m.dddddd`(기본 6자리) |
| `e`, `E` | `double` | 지수 표기 |
| `g`, `G` | `double` | 상황에 따라 `%e` 또는 `%f` |
| `p` | `void *` | 주소 |
| `%` | — | 퍼센트 기호 자체 |

### 2.2 문자열의 정밀도

**정밀도가 문자열에서는 "최대 글자 수"를 뜻한다**는 점이 특히 헷갈립니다. `printf_fmt.c`로 확인합니다.

```c
    const char *s = "hello, world";     /* 12글자 */

    printf(":%s:\n", s);
    printf(":%10s:\n", s);
    printf(":%.10s:\n", s);
    printf(":%-10s:\n", s);
    printf(":%.15s:\n", s);
    printf(":%-15s:\n", s);
    printf(":%15.10s:\n", s);
    printf(":%-15.10s:\n", s);
```

```text
:hello, world:
:hello, world:
:hello, wor:
:hello, world:
:hello, world:
:hello, world   :
:     hello, wor:
:hello, wor     :
```

| 서식 | 결과 | 이유 |
|---|---|---|
| `%10s` | 그대로 | 폭 10 < 글자 수 12 → **폭은 무시된다** |
| `%.10s` | 잘림 | 정밀도 10 → **앞 10글자만** |
| `%-15s` | 오른쪽에 공백 | 폭 15, 왼쪽 정렬 |
| `%15.10s` | 잘리고 오른쪽 정렬 | 10글자로 자른 뒤 폭 15 |

**폭은 늘리기만 하고, 정밀도는 자릅니다.**

### 2.3 폭과 정밀도를 인자로 넘기기

`*`를 쓰면 값을 인자로 넘길 수 있습니다.

```c
    printf(":%*.*s:   <- 폭 15, 정밀도 5 를 인자로\n", 15, 5, s);
```

```text
:          hello:   <- 폭 15, 정밀도 5 를 인자로
```

### 2.4 `printf`의 반환값

`printf`는 **출력한 글자(바이트) 수**를 돌려줍니다.

```c
    n = printf("이 줄은 몇 글자일까요?\n");
    printf("방금 printf 가 돌려준 값 = %d\n", n);
```

```text
이 줄은 몇 글자일까요?
방금 printf 가 돌려준 값 = 32
```

한글이 아홉 글자인데 32가 나왔습니다. **UTF-8에서 한글 한 글자는 3바이트**이기 때문입니다(9 × 3 = 27, 여기에 공백 3개와 `?`, 개행 = 5를 더해 32). 2강 6.9절에서 `getchar`가 바이트 단위로 읽는다고 한 것과 같은 이야기입니다.

### 2.5 `sprintf`와 `snprintf`

화면 대신 **문자열에 찍는** 함수도 있습니다.

```c
    /* sprintf: 화면 대신 문자열에 찍는다 (크기 검사 없음 — 위험) */
    sprintf(buf, "%d-%02d-%02d", 2026, 10, 19);

    /* snprintf: 크기를 함께 넘긴다 (안전) */
    n = snprintf(small, sizeof small, "%s", "programming");
```

```text
sprintf 결과 = 2026-10-19
snprintf 결과 = "program"
돌려준 값 = 11   <- 잘리지 않았다면 필요했을 길이
>> 입력이 잘렸습니다 (버퍼 8칸, 필요 12칸)
```

> **`sprintf`는 버퍼 오버플로의 단골입니다.**
> 결과가 배열보다 길어도 **그냥 써 버립니다.** 7강에서 본 것과 똑같은 사고가 납니다. 항상 **`snprintf`에 `sizeof`로 크기를 함께 넘기십시오.**
>
> `snprintf`의 반환값은 **실제로 쓴 길이가 아니라 "잘리지 않았다면 필요했을 길이"** 입니다. 그래서 위처럼 비교하면 잘렸는지 알 수 있습니다.
{: .prompt-danger }

### 2.6 형식 문자열에 사용자 입력을 넣지 마십시오

```c
printf(s);          /* 위험: s 안에 % 가 있으면 오작동한다 */
printf("%s", s);    /* 안전 */
```

`s`에 `%d`가 들어 있으면 `printf`는 있지도 않은 인자를 읽으려 합니다. 사용자 입력이 그대로 들어가면 **프로그램의 메모리를 읽어 내는 공격(형식 문자열 취약점)** 으로 이어집니다. **문자열을 출력할 때는 언제나 `printf("%s", s)`** 로 쓰십시오.

---

## 제3절. 가변 인자 함수

### 3.1 인자의 개수가 정해지지 않은 함수

`printf`는 인자를 몇 개 받을지 미리 정해져 있지 않습니다. 이런 함수를 직접 만들 수도 있습니다.

```c
void minprintf(const char *fmt, ...);
```

`...`는 **"여기부터는 개수와 자료형이 정해지지 않았다"** 는 뜻이며, **반드시 마지막**에 와야 합니다. 그리고 **이름이 있는 인자가 최소 하나** 필요합니다.

### 3.2 `<stdarg.h>`

| 이름 | 하는 일 |
|---|---|
| `va_list ap;` | 인자들을 차례로 가리킬 변수 |
| `va_start(ap, 마지막_이름있는_인자)` | 준비. **쓰기 전에 한 번** |
| `va_arg(ap, 자료형)` | 인자 하나를 꺼내고 다음으로 이동 |
| `va_end(ap)` | 뒷정리. **끝내기 전에 한 번** |

`minprintf.c`입니다.

```c
#include <stdarg.h>

/* minprintf: 아주 단순한 printf (가변 인자 연습) */
void minprintf(const char *fmt, ...)
{
    va_list ap;             /* 이름 없는 인자들을 차례로 가리킨다 */
    const char *p, *sval;
    int ival;
    double dval;

    va_start(ap, fmt);      /* 마지막으로 이름이 있는 인자를 알려 준다 */
    for (p = fmt; *p != '\0'; p++) {
        if (*p != '%') {
            putchar(*p);
            continue;
        }
        switch (*++p) {
        case 'd':
            ival = va_arg(ap, int);
            printf("%d", ival);
            break;
        case 'f':
            dval = va_arg(ap, double);
            printf("%f", dval);
            break;
        case 's':
            for (sval = va_arg(ap, char *); *sval; sval++)
                putchar(*sval);
            break;
        case '%':
            putchar('%');
            break;
        default:
            putchar(*p);
            break;
        }
    }
    va_end(ap);             /* 뒷정리 */
}
```

```text
정수 42, 실수 3.140000, 문자열 hello
100% 완료
```

**`va_arg`에 자료형을 알려 주어야 하는 이유**는, 인자들이 어떤 자료형인지 **함수가 알 방법이 없기 때문**입니다. 형식 문자열이 유일한 단서입니다.

> 그래서 `printf("%d", 3.14)`처럼 서식과 인자가 어긋나면 **엉뚱한 메모리를 읽습니다.** 3강에서 `-Wall`을 켜라고 강조한 이유가 여기에 있습니다. 컴파일러는 `printf` 계열 함수의 형식 문자열을 검사해 줍니다.
{: .prompt-warning }

---

## 제4절. `scanf`와 `sscanf`

### 4.1 `printf`의 반대

```c
int scanf(const char *format, ...);
int sscanf(const char *s, const char *format, ...);
```

`scanf`는 표준 입력에서, `sscanf`는 **주어진 문자열에서** 읽습니다. 나머지 인자는 **모두 포인터여야 합니다**(6강 3.3절).

| 변환문자 | 읽는 것 | 인자 자료형 |
|---|---|---|
| `d` | 10진 정수 | `int *` |
| `i` | 정수(8진·16진 접두어 인식) | `int *` |
| `o`, `x`, `u` | 8진 / 16진 / 부호 없는 10진 | `int *`, `unsigned *` |
| `c` | 문자(**공백도 읽는다**) | `char *` |
| `s` | 공백 전까지의 문자열 | `char *` |
| `f`, `e`, `g` | 실수 | **`float *`** |
| `lf` | 실수 | **`double *`** |
| `%` | 문자 `%` 자체 | — |

### 4.2 반환값이 가장 중요합니다

**`scanf`는 성공적으로 읽어 저장한 항목의 개수를 돌려줍니다.** 입력이 끝나면 `EOF`입니다.

```c
    while (scanf("%lf", &v) == 1)
        printf("\t%.2f\n", sum += v);
```

입력이 `1.5`, `2.25`, `10` 이면

```text
	1.50
	3.75
	13.75
```

**반환값을 검사하지 않으면** 값이 들어오지 않은 변수를 그대로 쓰게 됩니다.

### 4.3 안전한 방법 — 줄로 읽고 `sscanf`로 뜯기

`scanf`를 직접 쓰면 형식이 어긋났을 때 처리가 까다롭습니다(4강 부록 A). **더 나은 방법은 `fgets`로 한 줄을 통째로 읽고, 그 줄을 `sscanf`로 해석하는 것**입니다.

`scanf_date.c`입니다. 두 가지 날짜 형식을 모두 받아들입니다.

```c
    while (fgets(line, sizeof line, stdin) != NULL) {
        if (sscanf(line, "%d %19s %d", &day, monthname, &year) == 3)
            printf("정상(이름형) : %d년 %s %d일\n", year, monthname, day);
        else if (sscanf(line, "%d/%d/%d", &month, &day, &year) == 3)
            printf("정상(숫자형) : %d년 %d월 %d일\n", year, month, day);
        else
            printf("형식 오류    : %s", line);
    }
```

입력이

```text
25 Dec 1988
12/25/1988
hello
```

```text
정상(이름형) : 1988년 Dec 25일
정상(숫자형) : 1988년 12월 25일
형식 오류    : hello
```

이 방식의 장점이 큽니다.

| 장점 | 설명 |
|---|---|
| 여러 형식을 **차례로 시도**할 수 있다 | 실패해도 줄은 그대로 남아 있다 |
| 실패해도 **입력이 꼬이지 않는다** | `scanf`와 달리 남은 글자를 버릴 필요가 없다 |
| 오류 메시지에 **문제의 줄을 그대로** 보여 줄 수 있다 | — |

### 4.4 `%s`의 위험과 폭 제한

```c
sscanf(line, "%s", name);        /* 위험: 길이 제한이 없다 */
sscanf(line, "%19s", name);      /* 안전: 최대 19글자 + '\0' */
```

`%s`는 배열 크기를 모르므로 **긴 입력이 들어오면 배열 밖을 침범합니다**(7강 7절). 반드시 **폭을 지정**하십시오. 배열이 `char name[20]`이면 폭은 **19**입니다. `'\0'` 자리를 남겨야 하기 때문입니다.

### 4.5 `%c`와 공백

`%c`는 **공백류도 읽습니다.** 숫자를 읽은 뒤 문자를 읽으면 앞서 남은 개행이 먼저 잡힙니다.

```c
scanf(" %c", &ch);      /* 앞의 공백이 "공백류를 건너뛰라"는 뜻 */
```

---

> **▶ 여기서부터 2회차 — 파일·오류 처리·줄 단위 입출력**
> 제5절 ~ 제8절, 약 155분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제5절. 파일 열고 읽고 쓰기

### 5.1 파일 포인터

파일을 쓰려면 먼저 **열어야** 합니다.

```c
FILE *fp;

fp = fopen("이름", "모드");
```

`FILE`은 파일에 관한 정보를 담은 구조체이며, 우리는 그 내용을 알 필요 없이 **포인터만** 다룹니다. `fopen`이 실패하면 **`NULL`** 을 돌려주므로 **반드시 검사해야 합니다.**

| 모드 | 뜻 | 파일이 없으면 | 파일이 있으면 |
|---|---|---|---|
| `"r"` | 읽기 | **실패**(`NULL`) | 처음부터 읽는다 |
| `"w"` | 쓰기 | 새로 만든다 | **기존 내용을 모두 지운다** |
| `"a"` | 덧붙이기 | 새로 만든다 | 끝에 이어 쓴다 |
| `"rb"`·`"wb"` | 이진 파일 | — | 텍스트 변환을 하지 않는다 |

> **`"w"`는 기존 파일을 지웁니다.** 실습 중에 소스 파일 이름을 잘못 적어 날리는 사고가 흔합니다. 파일 이름을 넘길 때 특히 조심하십시오.
{: .prompt-danger }

다 쓴 파일은 **`fclose(fp)`** 로 닫습니다. 닫으면 **출력 버퍼가 비워지고**, 동시에 열 수 있는 파일 수의 제한에서도 벗어납니다.

### 5.2 읽고 쓰는 함수들

| 표준 입출력 | 파일 | 하는 일 |
|---|---|---|
| `getchar()` | `getc(fp)` | 한 글자 읽기 |
| `putchar(c)` | `putc(c, fp)` | 한 글자 쓰기 |
| `printf(...)` | `fprintf(fp, ...)` | 서식 출력 |
| `scanf(...)` | `fscanf(fp, ...)` | 서식 입력 |
| — | `fgets(buf, n, fp)` | 한 줄 읽기 |
| — | `fputs(s, fp)` | 문자열 쓰기 |

사실 `getchar()`는 `getc(stdin)`이고 `putchar(c)`는 `putc(c, stdout)`입니다. **우리가 1강부터 써 온 함수들이 파일 함수의 특수한 경우**였던 것입니다.

### 5.3 `cat` — 파일들을 이어서 출력하기

`cat1.c`입니다.

```c
#include <stdio.h>

void filecopy(FILE *ifp, FILE *ofp);

/* cat: 파일들을 이어서 출력한다 (1차 판) */
int main(int argc, char *argv[])
{
    FILE *fp;

    if (argc == 1) {                     /* 인자가 없으면 표준 입력 */
        filecopy(stdin, stdout);
    } else {
        while (--argc > 0) {
            if ((fp = fopen(*++argv, "r")) == NULL) {
                printf("cat: %s 파일을 열 수 없습니다\n", *argv);
                return 1;
            }
            filecopy(fp, stdout);
            fclose(fp);                  /* 다 쓴 파일은 반드시 닫는다 */
        }
    }
    return 0;
}

/* filecopy: ifp 의 내용을 ofp 로 옮긴다 */
void filecopy(FILE *ifp, FILE *ofp)
{
    int c;

    while ((c = getc(ifp)) != EOF)
        putc(c, ofp);
}
```

```powershell
.\cat1.exe a.txt b.txt
```

**설계가 훌륭한 지점이 두 곳** 있습니다.

1. **인자가 없으면 표준 입력을 처리합니다.** 그래서 `.\cat1.exe < in.txt`도 되고 파이프에도 쓸 수 있습니다.
2. **`filecopy`는 어떤 스트림이든 받습니다.** `stdin`·`stdout`도 결국 `FILE *`이므로 파일과 똑같이 다룰 수 있습니다.

### 5.4 파일에 쓰고 다시 읽기

`fileio.c`는 세 가지 모드를 모두 보여 줍니다.

```c
    /* (1) 쓰기 */
    if ((fp = fopen("scores.txt", "w")) == NULL) {
        fprintf(stderr, "scores.txt 를 만들 수 없습니다\n");
        return 1;
    }
    fprintf(fp, "%s %.1f\n", "kim", 91.5);
    ...
    fclose(fp);

    /* (2) 읽기 */
    while ((n = fscanf(fp, "%31s %lf", name, &score)) == 2) {
        i++;
        printf("%d행: %-6s %5.1f\n", i, name, score);
    }

    /* (3) 덧붙이기 */
    fp = fopen("scores.txt", "a");
    fprintf(fp, "%s %.1f\n", "choi", 88.5);
```

```text
scores.txt 에 3줄을 기록했습니다.
1행: kim     91.5
2행: lee     78.0
3행: park    85.5
한 줄을 덧붙였습니다.
```

- `fprintf`는 **`printf`와 완전히 같고 첫 인자로 파일만 더 받습니다.**
- `fscanf`도 마찬가지이며 **반환값을 검사**해야 합니다.
- `%31s`처럼 **폭을 지정**한 것에 주목하십시오(4.4절). 배열이 `char name[32]`이기 때문입니다.

---

## 제6절. 오류 처리 — `stderr`와 `exit`

### 6.1 왜 오류를 따로 내보내는가

`cat1.c`에는 문제가 있습니다. 오류 메시지를 `printf`로 출력하므로 **결과와 섞입니다.** 다음처럼 실행하면 오류 메시지까지 파일에 들어가 버립니다.

```powershell
.\cat1.exe a.txt 없는파일.txt > result.txt
```

**화면에는 아무것도 보이지 않고, 결과 파일이 오염됩니다.** 그래서 오류는 `stderr`로 보내야 합니다. `stderr`는 **출력을 파일로 돌려도 화면에 그대로 나옵니다.**

### 6.2 `cat`의 개선판

`cat2.c`입니다.

```c
#include <stdio.h>
#include <stdlib.h>

void filecopy(FILE *ifp, FILE *ofp);     /* 1차 판과 같은 함수 */

/* cat: 오류를 표준 오류로 내보내는 판 (2차 판) */
int main(int argc, char *argv[])
{
    FILE *fp;
    const char *prog = argv[0];          /* 오류 메시지에 쓸 프로그램 이름 */

    if (argc == 1) {
        filecopy(stdin, stdout);
    } else {
        while (--argc > 0) {
            if ((fp = fopen(*++argv, "r")) == NULL) {
                fprintf(stderr, "%s: %s 파일을 열 수 없습니다\n", prog, *argv);
                exit(1);                 /* 0 이 아닌 값 = 실패 */
            }
            filecopy(fp, stdout);
            fclose(fp);
        }
    }
    if (ferror(stdout)) {                /* 출력 중 오류가 있었는가 */
        fprintf(stderr, "%s: 표준 출력에 쓰는 중 오류\n", prog);
        exit(2);
    }
    exit(0);
}
```

바뀐 점이 셋입니다.

| 항목 | 설명 |
|---|---|
| `fprintf(stderr, ...)` | 오류는 **화면에 그대로** 나온다 |
| **프로그램 이름**을 함께 출력 | 여러 프로그램을 연결해 쓸 때 누가 낸 오류인지 알 수 있다 |
| `exit(값)` | 프로그램을 끝내며 **상태를 알린다** |

### 6.3 `exit`와 종료 상태

```c
exit(0);      /* 정상 종료 */
exit(1);      /* 오류 */
```

`main` 안에서는 `return 값;`과 같습니다. 그러나 **`exit`는 어느 함수에서든 부를 수 있습니다.** 또한 열려 있는 출력 파일을 모두 닫아(버퍼를 비워) 줍니다.

종료 상태는 셸에서 확인할 수 있습니다.

```powershell
.\cat2.exe 없는파일.txt; $LASTEXITCODE
```

**관례상 0은 성공, 0이 아니면 실패**입니다. 이 값으로 다른 프로그램이나 스크립트가 성공 여부를 판단합니다.

### 6.4 `ferror`와 `feof`

| 함수 | 뜻 |
|---|---|
| `ferror(fp)` | 그 스트림에 **오류**가 있었으면 0이 아닌 값 |
| `feof(fp)` | **입력의 끝**에 도달했으면 0이 아닌 값 |

출력 오류는 드물지만 실제로 일어납니다(디스크가 가득 찼을 때 등). 중요한 프로그램은 `ferror(stdout)`을 확인해야 합니다.

> **`while (!feof(fp))` 형태로 반복하지 마십시오.**
> `feof`는 **읽기를 시도해 실패한 뒤에야** 참이 됩니다. 그래서 이 형태로 쓰면 **마지막 자료를 한 번 더 처리**하는 버그가 생깁니다. 언제나 **읽기 함수의 반환값으로 판단**하십시오.
> ```c
> while ((c = getc(fp)) != EOF)              /* 올바름 */
> while (fgets(line, sizeof line, fp) != NULL)  /* 올바름 */
> ```
{: .prompt-danger }

---

## 제7절. 줄 단위 입출력

### 7.1 `fgets`와 `fputs`

```c
char *fgets(char *line, int maxline, FILE *fp);
int   fputs(const char *line, FILE *fp);
```

| 함수 | 하는 일 | 돌려주는 것 |
|---|---|---|
| `fgets` | 최대 `maxline - 1`글자를 읽는다. **개행도 포함**하고 끝에 `'\0'`을 붙인다 | `line`, 끝이거나 오류면 `NULL` |
| `fputs` | 문자열을 쓴다(개행을 붙이지 않는다) | 오류면 `EOF` |

**`fgets`는 크기를 함께 받으므로 안전합니다.** 2강에서 만든 `get_line`을 `fgets`로 다시 쓰면 이렇게 짧아집니다.

```c
/* get_line: fgets 로 만든 우리 판. 길이를 돌려준다 */
int get_line(char *line, int max)
{
    if (fgets(line, max, stdin) == NULL)
        return 0;
    return (int) strlen(line);
}
```

### 7.2 개행 처리

`fgets`는 **개행 문자를 그대로 남겨 둡니다.** 대개는 지워야 합니다.

```c
        if (line[len - 1] == '\n') {
            line[len - 1] = '\0';        /* 끝의 개행을 지운다 */
```

입력이 `hello`, `world!!` 두 줄일 때

```text
 1행 ( 5자): [hello]
 2행 ( 7자): [world!!]
모두 2행
```

> **개행이 없을 수도 있습니다.** 줄이 배열보다 길어 잘렸거나, 파일의 마지막 줄에 개행이 없는 경우입니다. **검사 없이 `line[len-1] = '\0'`을 하면** 마지막 글자를 잘라먹습니다.
{: .prompt-warning }

### 7.3 `gets`는 절대 쓰지 마십시오

옛 코드에는 `gets(line)`이 보입니다. **크기를 받지 않으므로 입력이 길면 반드시 넘칩니다.** 막을 방법이 아예 없습니다.

이 함수는 **C11 표준에서 아예 삭제되었습니다.** 인터넷의 옛 예제에서 보더라도 **언제나 `fgets`로 바꾸어** 쓰십시오.

| 쓰지 말 것 | 대신 쓸 것 |
|---|---|
| `gets(line)` | `fgets(line, sizeof line, stdin)` |
| `scanf("%s", buf)` | `fgets` 또는 `scanf("%19s", buf)` |
| `sprintf(buf, ...)` | `snprintf(buf, sizeof buf, ...)` |
| `strcpy(dst, src)` | 길이 검사 후 복사, 또는 `snprintf` |

이 표는 **10강 「안전한 C 프로그래밍」의 뼈대**이기도 합니다.

---

## 제8절. 표준 라이브러리 둘러보기

지금까지 여러 강의에 흩어져 나온 함수들을 정리합니다. `misc_demo.c`로 한꺼번에 확인할 수 있습니다.

### 8.1 문자열 — `<string.h>`

| 함수 | 하는 일 |
|---|---|
| `strlen(s)` | 길이(`'\0'` 제외) |
| `strcpy(s, t)` / `strncpy(s, t, n)` | 복사 |
| `strcat(s, t)` / `strncat(s, t, n)` | 이어 붙이기 |
| `strcmp(s, t)` / `strncmp(s, t, n)` | 비교(음수·0·양수) |
| `strchr(s, c)` / `strrchr(s, c)` | 문자가 **처음/마지막** 나온 위치의 포인터 |
| `strstr(s, t)` | 문자열이 처음 나온 위치의 포인터 |

### 8.2 문자 — `<ctype.h>`

`isalpha`·`isdigit`·`isalnum`·`isspace`·`isupper`·`islower`·`isprint`, 그리고 `toupper`·`tolower`.

**참일 때 1이 아닐 수 있다**는 점을 잊지 마십시오(3강 7.3절).

### 8.3 수학 — `<math.h>`

`sqrt`·`pow`·`fabs`·`sin`·`cos`·`exp`·`log`·`log10`. 모두 `double`을 받고 `double`을 돌려줍니다.

### 8.4 그 밖 — `<stdlib.h>`

| 함수 | 하는 일 |
|---|---|
| `atoi(s)` / `atof(s)` | 문자열 → 정수 / 실수 |
| `malloc(n)` / `calloc(n, size)` / `free(p)` | 메모리 빌리기(8강 6절). `calloc`은 **0으로 채워** 준다 |
| `qsort(...)` | 어떤 자료형이든 정렬(7강 5.5절) |
| `rand()` / `srand(seed)` | 난수. `RAND_MAX`까지 |
| `exit(status)` | 종료 |
| `system(s)` | 명령 실행 |

```text
strlen        = 16
strchr(,'g')  = gramming in c
strrchr(,'g') = g in c
strstr(,"in") = ing in c
strcmp("abc","abd") = -1
isalpha('x')=1 isdigit('5')=1 isspace(' ')=1
toupper('a')=A tolower('Z')=z
sqrt(2)=1.414214  pow(2,10)=1024  fabs(-3.5)=3.5
log10(1000)=3
atoi("123abc")=123  atof("3.5e2")=350
RAND_MAX = 32767
```

> **`RAND_MAX`가 32767** 인 것에 주목하십시오. 환경마다 다르며, 우리 환경에서는 그리 크지 않습니다. 큰 범위의 난수가 필요하면 그에 맞는 방법을 따로 써야 합니다.
{: .prompt-info }

---

> **▶ 여기서부터 3회차 — 구조체 파일 다루기와 실습**
> 제9절 ~ 실습문제, 약 165분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제9절. 구조체를 파일로 다루기

8강에서 만든 구조체를 파일과 연결하면 **자료를 남길 수 있는 프로그램**이 됩니다. 실무에서 가장 흔한 형태가 **CSV**(쉼표로 구분된 값)입니다.

`students.csv`

```text
1,kim,91.5
2,lee,78.0
3,park,85.5
bad line
```

`csv_read.c`의 핵심입니다.

```c
struct student {
    int  id;
    char name[32];
    double average;
};

/* CSV 한 줄을 뜯어 구조체에 담는다. 성공하면 1 */
static int parse_line(const char *line, struct student *s)
{
    return sscanf(line, "%d,%31[^,],%lf", &s->id, s->name, &s->average) == 3;
}
```

```c
    while (fgets(line, sizeof line, fp) != NULL) {
        lineno++;
        if (n >= MAXSTUDENT) {
            fprintf(stderr, "경고: %d명까지만 읽습니다\n", MAXSTUDENT);
            break;
        }
        if (parse_line(line, &list[n]))
            n++;
        else
            fprintf(stderr, "%d행 형식 오류: %s", lineno, line);
    }
    fclose(fp);

    qsort(list, (size_t) n, sizeof list[0], by_average_desc);
```

```powershell
.\csv_read.exe students.csv
```

```text
평균 내림차순 (3명)
  1 kim        91.5
  3 park       85.5
  2 lee        78.0
```

그리고 화면에는(표준 오류로) 다음이 함께 나옵니다.

```text
4행 형식 오류: bad line
```

이 짧은 프로그램에 이번 과정의 거의 모든 것이 들어 있습니다.

| 요소 | 배운 곳 |
|---|---|
| 명령행 인자로 파일 이름 받기 | 7강 |
| `fopen` 실패 검사, `stderr`로 오류 | 9강 5·6절 |
| `fgets`로 한 줄씩 안전하게 읽기 | 9강 7절 |
| `sscanf`로 뜯어 **구조체**에 담기 | 8강 · 9강 4절 |
| 배열 넘침 검사 | 2강 · 7강 |
| `qsort`와 **비교 함수** | 7강 5절 |

> **`%31[^,]`** 는 "쉼표가 아닌 글자를 최대 31개 읽어라"는 뜻입니다. `%s`는 공백에서 멈추므로 이름에 공백이 있으면 잘립니다. 대괄호 안에 **읽을(또는 `^`로 읽지 않을) 글자 집합**을 적는 이 표기를 알아 두면 CSV 처리가 훨씬 편해집니다.
{: .prompt-tip }

---

## 제10절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| `fopen` 뒤에 프로그램이 죽음 | `NULL` 검사 누락 | 반드시 검사 |
| 파일 내용이 사라짐 | `"w"` 모드로 열었다 | 덧붙이려면 `"a"` |
| 파일에 쓴 내용이 없음 | `fclose` 누락(버퍼가 안 비워짐) | 다 쓰면 닫는다 |
| 마지막 자료가 두 번 처리됨 | `while (!feof(fp))` | 읽기 함수의 반환값으로 판단 |
| `scanf` 후 값이 이상함 | `&` 누락 또는 반환값 미검사 | 포인터 전달, 반환값 검사 |
| 긴 입력에 프로그램이 망가짐 | `gets`·`%s`·`sprintf`·`strcpy` | `fgets`·`%19s`·`snprintf` |
| 줄 끝에 이상한 글자 | `fgets`가 남긴 개행 | 검사 후 `'\0'`으로 바꾼다 |
| 결과 파일에 오류 메시지가 섞임 | 오류를 `printf`로 출력 | `fprintf(stderr, ...)` |
| `%s`로 문자열 출력 시 오작동 | `printf(s)` 형태 | `printf("%s", s)` |
| 실수를 못 읽음 | `scanf`에서 `%f`와 `%lf` 혼동 | `double`은 **`%lf`** |

---

## 실습문제

> **안내**
> 1. 모든 문제는 `C:\c-study\lab09`에서 수행합니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다. **먼저 스스로 작성한 뒤** 확인하시기 바랍니다.
> 3. 파일을 다루는 문제는 **`fopen` 결과 검사와 `fclose`** 를 반드시 포함해야 합니다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 | K&R |
|---|---|---|---|
| 문제 1 | 프로그램 이름에 따라 대소문자 변환 | 1 | 7-1 |
| 문제 2 | 보이지 않는 문자를 8진수로 | 2 | 7-2 |
| 문제 3 | `minprintf` 확장 | 3 | 7-3 |
| 문제 4 | 두 파일 비교 | 5 · 7 | 7-6 |
| 문제 5 | 여러 파일에서 패턴 찾기 | 5 · 6 | 7-7 |
| 문제 6 | 파일별 줄·단어·문자 세기 | 5 | — |
| 문제 7 | 파일에서 수를 읽어 통계 내기 | 5 · 8 | — |
| 문제 8 | 안전한 입력 함수 만들기 | 4 · 7 | — |
| 문제 9 | 성적 파일 읽어 정렬·저장 | 9 | — |
| 문제 10 | 위험한 함수 찾아 고치기 | 7.3 | — |

---

### 문제 1. 프로그램 이름에 따라 대소문자 변환

프로그램 이름이 `upper`를 포함하면 **대문자로**, 아니면 **소문자로** 바꾸어 출력하는 프로그램을 작성하십시오.

**정답 및 해설**

```c
#include <stdio.h>
#include <ctype.h>
#include <string.h>

/* 프로그램 이름이 upper 를 포함하면 대문자로, 아니면 소문자로 */
int main(int argc, char *argv[])
{
    int c;
    int to_upper = 0;
    const char *name = (argc > 0) ? argv[0] : "lower";

    if (strstr(name, "upper") != NULL)
        to_upper = 1;

    while ((c = getchar()) != EOF)
        putchar(to_upper ? toupper(c) : tolower(c));
    return 0;
}
```

같은 소스를 두 이름으로 빌드해 확인합니다.

```powershell
gcc -Wall -Wextra -std=c17 ex1.c -o lower.exe
```

```powershell
gcc -Wall -Wextra -std=c17 ex1.c -o upper.exe
```

```powershell
"Hello World" | .\upper.exe
```

```text
HELLO WORLD
```

- **`argv[0]`이 프로그램 이름**이라는 7강의 내용을 활용했습니다.
- `argc > 0`을 확인한 이유는, 드물지만 `argv[0]`이 비어 있는 환경이 있기 때문입니다.
- 유닉스에서는 같은 실행 파일에 여러 이름을 붙여 **동작을 바꾸는 기법**이 실제로 쓰입니다.

---

### 문제 2. 보이지 않는 문자를 8진수로

입력을 그대로 출력하되 **인쇄할 수 없는 문자는 `\nnn` 형태의 8진수로** 보여 주고, 줄이 너무 길면 접으십시오.

**정답 및 해설**

```c
#define MAXCOL 60

int main(void)
{
    int c, col = 0;

    while ((c = getchar()) != EOF) {
        if (c == '\n') {
            putchar('\n');
            col = 0;
            continue;
        }
        if (isprint(c)) {
            putchar(c);
            col++;
        } else {
            col += printf("\\%03o", c);    /* 8진수로 보이게 */
        }
        if (col >= MAXCOL) {               /* 너무 길면 접는다 */
            printf("\\\n");
            col = 0;
        }
    }
    return 0;
}
```

탭이 들어 있는 `a<탭>b` 를 입력하면

```text
a\011b
```

- **탭의 값은 9**, 8진수로 `011`입니다(3강 3.5절).
- `printf`가 **출력한 글자 수를 돌려준다**는 성질(2.4절)을 이용해 열 위치를 셌습니다. `\011`은 4글자이므로 `col`이 4 늘어납니다.
- 접을 때 줄 끝에 `\`를 붙이는 것은 **"이 줄이 이어진다"** 는 표시로, 실제 도구들이 쓰는 관례입니다.
- `isprint`는 `<ctype.h>`의 함수로, 화면에 보이는 문자인지 검사합니다.

---

### 문제 3. `minprintf` 확장

`minprintf`가 `%x`(16진수)와 `%c`(문자)도 처리하도록 확장하십시오.

**정답 및 해설**

```c
        switch (*++p) {
        case 'd':
            ival = va_arg(ap, int);
            printf("%d", ival);
            break;
        case 'x':                               /* 추가 */
            ival = va_arg(ap, int);
            printf("%x", ival);
            break;
        case 'c':                               /* 추가 */
            ival = va_arg(ap, int);             /* char 가 아니라 int 로 받는다 */
            putchar(ival);
            break;
        ...
```

- **`%c`인데도 `va_arg(ap, int)`로 받는 것**이 핵심입니다. 가변 인자로 넘어온 `char`는 **자동으로 `int`로 승격**되기 때문입니다. `va_arg(ap, char)`라고 쓰면 잘못된 값을 읽습니다.
- 같은 이유로 `float`도 **`double`로 승격**됩니다. 그래서 3.2절의 `minprintf`가 `%f`에서 `va_arg(ap, double)`을 쓴 것입니다.
- 이런 규칙이 있기 때문에 **가변 인자 함수는 자료형 검사를 받지 못합니다.** 서식 문자열과 인자가 어긋나도 컴파일러가 막아 주지 못하는 이유입니다.

---

### 문제 4. 두 파일 비교

두 파일을 비교하여 **처음으로 다른 줄**을 출력하십시오.

**정답 및 해설**

```c
    for ( ; ; ) {
        r1 = fgets(line1, sizeof line1, fp1);
        r2 = fgets(line2, sizeof line2, fp2);
        lineno++;

        if (r1 == NULL && r2 == NULL) {
            printf("두 파일이 같습니다.\n");
            break;
        }
        if (r1 == NULL || r2 == NULL) {
            printf("%ld행: 한쪽 파일이 먼저 끝났습니다.\n", lineno);
            break;
        }
        if (strcmp(line1, line2) != 0) {
            printf("%ld행에서 처음 달라집니다.\n", lineno);
            printf("  %s: %s", argv[1], line1);
            printf("  %s: %s", argv[2], line2);
            break;
        }
    }
```

`a.txt`(hello/world/same)와 `b.txt`(hello/WORLD/same)를 비교하면

```text
2행에서 처음 달라집니다.
  a.txt: world
  b.txt: WORLD
```

- **경우가 셋**입니다. 둘 다 끝(같음), 한쪽만 끝(길이가 다름), 내용이 다름. 이 셋을 모두 처리해야 올바른 프로그램입니다.
- `fgets`는 개행을 남기므로 `printf("%s", line1)`에 `\n`을 붙이지 않았습니다.
- 파일을 두 개 열었으므로 **두 개 모두 닫아야** 합니다. 중간에 실패하면 이미 연 것을 닫고 나가는 것도 잊지 마십시오.

---

### 문제 5. 여러 파일에서 패턴 찾기

7강의 패턴 찾기를 확장하여 **여러 파일**에서 찾고, 파일이 둘 이상이면 **파일 이름도 함께** 출력하십시오.

**정답 및 해설**

```c
static int search(FILE *fp, const char *pattern, const char *fname, int show_name)
{
    char line[MAXLINE];
    long lineno = 0;
    int found = 0;

    while (fgets(line, sizeof line, fp) != NULL) {
        lineno++;
        if (strstr(line, pattern) != NULL) {
            if (show_name)
                printf("%s:%ld: %s", fname, lineno, line);
            else
                printf("%ld: %s", lineno, line);
            found++;
        }
    }
    return found;
}
```

```c
    if (argc == 2) {                       /* 파일 이름이 없으면 표준 입력 */
        total = search(stdin, argv[1], "(표준 입력)", 0);
    } else {
        for (i = 2; i < argc; i++) {
            if ((fp = fopen(argv[i], "r")) == NULL) {
                fprintf(stderr, "%s: %s 를 열 수 없습니다\n", argv[0], argv[i]);
                continue;                  /* 다음 파일로 넘어간다 */
            }
            total += search(fp, argv[1], argv[i], argc > 3);
            fclose(fp);
        }
    }
    return (total > 0) ? 0 : 1;
```

- **파일 하나를 처리하는 함수를 분리**한 것이 요령입니다. `stdin`도 `FILE *`이므로 그대로 넘길 수 있습니다(5.3절).
- 열 수 없는 파일을 만나면 **오류를 알리고 다음 파일로 넘어갑니다.** `exit` 하면 나머지 파일을 처리하지 못합니다. 어느 쪽이 옳은지는 프로그램의 목적에 따라 다릅니다.
- **파일이 둘 이상일 때만 이름을 붙이는 것**은 실제 `grep`의 동작과 같습니다.
- 찾은 것이 있으면 0, 없으면 1을 돌려주어 **다른 프로그램이 결과를 판단**할 수 있게 했습니다.

---

### 문제 6. 파일별 줄·단어·문자 세기

2강에서 만든 `wc`를 **여러 파일**에 대해 동작하도록 확장하고, 파일이 둘 이상이면 합계도 출력하십시오.

**정답 및 해설**

```c
static void count(FILE *fp, const char *name, long *tl, long *tw, long *tc)
{
    int c, in_word = 0;
    long nl = 0, nw = 0, nc = 0;

    while ((c = getc(fp)) != EOF) {
        nc++;
        if (c == '\n')
            nl++;
        if (isspace(c)) {
            in_word = 0;
        } else if (!in_word) {
            in_word = 1;
            nw++;
        }
    }
    printf("%8ld %8ld %8ld  %s\n", nl, nw, nc, name);
    *tl += nl;
    *tw += nw;
    *tc += nc;
}
```

```powershell
.\ex6.exe in.txt
```

```text
      줄     단어     문자  파일
       2        6       28  in.txt
```

- 2강의 상태 기계를 그대로 옮기고 **`getchar` 대신 `getc(fp)`** 만 바꾸었습니다.
- **합계를 포인터로 돌려받습니다**(6강 3.4절). 값을 셋이나 돌려주어야 하기 때문입니다.
- `isspace`를 쓰면 공백·탭·개행을 한 번에 처리할 수 있어 2강 판보다 간결합니다.

---

### 문제 7. 파일에서 수를 읽어 통계 내기

파일에 든 수들을 읽어 **개수·합계·평균·표준편차·최대·최소**를 출력하십시오.

**정답 및 해설**

```c
    while (n < MAXN && fscanf(fp, "%lf", &v[n]) == 1) {
        sum += v[n];
        n++;
    }
    fclose(fp);

    if (n == 0) {
        printf("읽은 수가 없습니다.\n");
        return 1;
    }

    mean = sum / n;
    mx = mn = v[0];
    for (i = 0; i < n; i++) {
        var += (v[i] - mean) * (v[i] - mean);
        if (v[i] > mx) mx = v[i];
        if (v[i] < mn) mn = v[i];
    }
    var /= n;
```

`numbers.txt`에 `10 20 30 40`이 들어 있으면

```text
개수 = 4
합계 = 100.00
평균 = 25.00
표준편차 = 11.18
최대 = 40.00, 최소 = 10.00
```

- **`fscanf`의 반환값이 1인 동안** 반복합니다. 이 조건 하나로 파일 끝과 형식 오류를 모두 처리합니다.
- **배열 크기 검사(`n < MAXN`)를 조건 앞쪽에** 두었습니다. 단축 평가 덕분에 자리가 없으면 읽지도 않습니다(3강 6.2절).
- **평균을 먼저 구해야 분산을 구할 수 있으므로** 자료를 배열에 담아 두 번 훑었습니다.
- `n == 0` 검사가 없으면 **0으로 나누게 됩니다.**

---

### 문제 8. 안전한 입력 함수 만들기

숫자를 안전하게 입력받는 함수를 만드십시오. 숫자가 아닌 값이 들어와도 **무한 반복에 빠지지 않아야** 합니다.

**정답 및 해설**

```c
/* 안전하게 정수 하나를 읽는다. 성공하면 1 */
static int read_int(const char *prompt, int *out)
{
    char line[MAXLINE];

    printf("%s", prompt);
    if (fgets(line, sizeof line, stdin) == NULL)
        return 0;                          /* 입력 끝 */
    if (sscanf(line, "%d", out) != 1) {
        printf("숫자가 아닙니다: %s", line);
        return 0;
    }
    return 1;
}
```

- **4강 부록 A의 문제를 근본적으로 해결한 형태**입니다. `scanf`를 직접 쓰면 실패한 입력이 그대로 남아 무한 반복에 빠졌지만, **줄 단위로 읽으면 실패해도 그 줄은 이미 소비**되었으므로 그런 일이 없습니다.
- 그래서 `clear_input()` 같은 뒤처리 함수가 필요 없습니다.
- 결과는 **포인터로 돌려주고 반환값은 성공 여부**로 씁니다(6강 3.4절).
- 실무에서 숫자를 엄밀하게 읽으려면 `strtol`을 씁니다. `sscanf`는 `"12abc"`도 12로 받아들이기 때문입니다.

---

### 문제 9. 성적 파일 읽어 정렬·저장

CSV 파일을 읽어 구조체 배열에 담고, 평균 내림차순으로 정렬한 뒤 **새 파일로 저장**하십시오.

**정답 및 해설**

9절의 `csv_read.c`에 저장 부분을 더합니다.

```c
    /* 정렬 결과를 새 파일로 저장 */
    if ((out = fopen("sorted.csv", "w")) == NULL) {
        fprintf(stderr, "%s: sorted.csv 를 만들 수 없습니다\n", argv[0]);
        return 1;
    }
    for (i = 0; i < n; i++)
        fprintf(out, "%d,%s,%.1f\n", list[i].id, list[i].name, list[i].average);
    fclose(out);
    printf("sorted.csv 에 %d명을 저장했습니다.\n", n);
```

```text
평균 내림차순 (3명)
  1 kim        91.5
  3 park       85.5
  2 lee        78.0
sorted.csv 에 3명을 저장했습니다.
```

- **읽기와 쓰기의 형식을 맞추어 두면** 저장한 파일을 그대로 다시 읽을 수 있습니다. 프로그램이 자기 결과를 다시 입력으로 쓸 수 있다는 뜻입니다.
- **읽는 파일과 쓰는 파일을 같은 이름으로 하지 마십시오.** `"w"`로 여는 순간 원본이 지워집니다(5.1절).
- 이 프로그램이 8강의 구조체와 9강의 파일을 잇는 지점입니다. 12강 종합 과제의 뼈대가 됩니다.

---

### 문제 10. 위험한 함수 찾아 고치기

다음 코드에서 위험한 부분을 모두 찾아 고치십시오.

```c
char name[8];
char msg[16];

gets(name);
sprintf(msg, "안녕, %s님", name);
printf(msg);
```

**정답 및 해설**

**세 줄 모두 위험합니다.**

| 줄 | 문제 | 고침 |
|---|---|---|
| `gets(name)` | 크기를 받지 않아 **반드시 넘칠 수 있다.** C11에서 삭제됨 | `fgets(name, sizeof name, stdin)` |
| `sprintf(msg, ...)` | 결과가 `msg`보다 길면 **넘친다** | `snprintf(msg, sizeof msg, ...)` |
| `printf(msg)` | 사용자 입력이 **형식 문자열**로 해석된다 | `printf("%s", msg)` |

```c
char name[8];
char msg[64];
size_t len;

if (fgets(name, sizeof name, stdin) != NULL) {
    len = strlen(name);
    if (len > 0 && name[len - 1] == '\n')
        name[len - 1] = '\0';              /* 개행 제거 */
    snprintf(msg, sizeof msg, "안녕, %s님", name);
    printf("%s\n", msg);
}
```

- **셋 다 7강에서 본 버퍼 오버플로와 같은 뿌리**입니다. 크기를 함께 넘기지 않는 함수는 언제나 위험합니다.
- `msg`의 크기도 넉넉히 키웠습니다. 한글은 UTF-8에서 글자당 3바이트이므로 16칸은 너무 작습니다(2.4절).
- `printf(msg)` 형태는 **형식 문자열 취약점**이라 불리며, 실제 보안 사고의 한 유형입니다. 10강에서 다시 다룹니다.

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 예제 파일 — `lower.c`, `printf_fmt.c`, `sprintf_demo.c`, `minprintf.c`, `scanf_date.c`, `cat1.c`, `cat2.c`, `fileio.c`, `fgets_demo.c`, `misc_demo.c`, `csv_read.c` |
| 2 | 실습문제 `ex1.c` ~ `ex10.c` |
| 3 | 실행 결과 화면 — 문제 4·5·7·9 |
| 4 | `cat1.exe`와 `cat2.exe`를 **없는 파일 이름으로** 실행하고, 출력을 파일로 돌렸을 때(`> out.txt`) 무엇이 다른지 화면과 함께 설명 |
| 5 | 짧은 서술 ① `while (!feof(fp))` 로 반복하면 안 되는 이유 |
| 6 | 짧은 서술 ② `scanf("%s", buf)` 대신 `fgets` + `sscanf` 를 쓰는 것이 나은 이유 |

---

## 정리

| 구분 | 내용 |
|---|---|
| 세 흐름 | `stdin`·`stdout`·`stderr`. 오류는 반드시 **`stderr`** 로 |
| 리다이렉션 | `<`, `>`, `\|` — 프로그램은 알지 못한다 |
| `printf` | 폭은 늘리고 **정밀도는 자른다**. 반환값은 출력한 바이트 수 |
| 안전 출력 | `sprintf` 대신 **`snprintf`**, `printf(s)` 대신 **`printf("%s", s)`** |
| 가변 인자 | `<stdarg.h>`의 `va_list`·`va_start`·`va_arg`·`va_end`. `char`·`float`는 승격된다 |
| `scanf` | 인자는 **포인터**, **반환값을 반드시 검사**, `%s`에는 **폭 지정** |
| 권장 패턴 | **`fgets`로 한 줄 → `sscanf`로 해석** |
| 파일 | `fopen`(`NULL` 검사) → `getc`/`putc`/`fprintf`/`fscanf` → **`fclose`** |
| 모드 | `"r"` 읽기, `"w"` **덮어씀**, `"a"` 덧붙임 |
| 오류 처리 | `fprintf(stderr, ...)`, `exit(값)`, `ferror`. **`feof`로 반복 조건 삼지 말 것** |
| 줄 입출력 | `fgets`(개행 포함, 크기 지정) / `fputs`. **`gets`는 절대 금지** |
| 라이브러리 | `<string.h>` `<ctype.h>` `<math.h>` `<stdlib.h>` |

이제 프로그램이 **자료를 남기고 다시 읽을 수 있게** 되었습니다. C 언어의 문법과 표준 라이브러리를 한 바퀴 모두 돌았습니다.

**다음 10강에서는 안전한 C 프로그래밍**을 다룹니다. 2강부터 여러 번 마주친 버퍼 오버플로를 비롯해, **위험한 함수와 안전한 대체 함수**를 체계적으로 정리하고, 정수 넘침·형식 문자열·메모리 관리 실수까지 **실제 사고로 이어지는 유형**을 실습으로 확인합니다. 이번 강의 7.3절의 표가 그 출발점입니다.

---

## 부록 A. 입출력 함수 대조표

| 표준 입출력 | 파일 | 문자열 |
|---|---|---|
| `getchar()` | `getc(fp)` | — |
| `putchar(c)` | `putc(c, fp)` | — |
| `printf(...)` | `fprintf(fp, ...)` | `snprintf(buf, n, ...)` |
| `scanf(...)` | `fscanf(fp, ...)` | `sscanf(s, ...)` |
| `gets(s)` ❌ | `fgets(s, n, fp)` | — |
| `puts(s)` | `fputs(s, fp)` | — |

## 부록 B. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `lower.c` | 표준 입출력과 리다이렉션 | 1.3 |
| `printf_fmt.c` | 폭과 정밀도 | 2.2 |
| `sprintf_demo.c` | `sprintf`와 `snprintf` | 2.5 |
| `minprintf.c` | 가변 인자 함수 | 3.2 |
| `scanf_date.c` | `fgets` + `sscanf` 패턴 | 4.3 |
| `scanf_calc.c` | `scanf` 반환값 | 4.2 |
| `cat1.c` | 파일 열고 읽기 | 5.3 |
| `cat2.c` | `stderr`와 `exit` | 6.2 |
| `fileio.c` | 쓰기·읽기·덧붙이기 | 5.4 |
| `fgets_demo.c` | 줄 단위 입력 | 7.2 |
| `misc_demo.c` | 표준 라이브러리 | 8 |
| `csv_read.c` | 구조체 + 파일 | 9 |
