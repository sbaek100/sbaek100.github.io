---
title: C 프로그래밍 기초 5강 - 함수와 프로그램 구조
date: 2026-09-21 09:00:00 +0900
categories:
  - 0.기초강의
  - C 프로그래밍 기초
tags:
  - C언어
  - KnR
  - 함수
  - 분할컴파일
  - 헤더파일
  - extern
  - static
  - 유효범위
  - 재귀
  - 전처리기
  - 매크로
  - 스택프레임
pin:
mermaid: false
---

> **학습 목표**
> 1. 함수 정의의 형식을 설명하고 `return`이 하는 일을 정확히 말할 수 있다.
> 2. `int`가 아닌 값을 돌려주는 함수를 만들 수 있고, 원형 선언이 없을 때의 위험을 설명할 수 있다.
> 3. 프로그램을 **여러 소스 파일로 나누어 따로 컴파일**하고 하나로 링크할 수 있다.
> 4. 헤더 파일을 작성하고 **중복 포함 방지**(`#ifndef`)를 적용할 수 있다.
> 5. 선언과 정의를 구분하고 `extern`이 필요한 자리를 판단할 수 있다.
> 6. `static`으로 파일 밖에서 보이지 않게 감출 수 있고, 함수 안 `static` 변수의 성질을 설명할 수 있다.
> 7. 블록 안 선언이 바깥 이름을 가리는 현상을 설명할 수 있다.
> 8. 변수와 배열의 초기화 규칙을 설명할 수 있다.
> 9. 재귀 함수를 작성할 수 있고, 호출마다 지역 변수가 새로 생기는 이유를 **스택 프레임**으로 설명할 수 있다.
> 10. 전처리기의 `#include`·매크로·조건부 컴파일을 사용할 수 있고, 매크로의 함정을 피할 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

4강까지 우리는 **파일 하나짜리 프로그램**만 만들었습니다. `main` 아래에 함수를 몇 개 붙이는 방식입니다. 그러나 프로그램이 커지면 이 방식은 곧 한계에 부딪힙니다.

- 파일 하나가 수천 줄이 되어 원하는 곳을 찾기 어렵다.
- 한 글자만 고쳐도 **전체를 다시 컴파일**해야 한다.
- 여러 사람이 동시에 작업할 수 없다.
- 4강에서 만든 `binsearch`·`itoa`·`trim`을 **다른 프로그램에서 다시 쓸 수 없다.**

이번 강의에서는 프로그램을 **여러 파일로 나누어 조립하는 방법**을 배웁니다. 이것이 실무 C 프로그래밍의 실제 모습이며, 11강에서 배울 라이브러리 만들기의 바탕이기도 합니다.

> **참고 문헌**
> 이번 강의는 다음 책의 **제4장 Functions and Program Structure** 전체를 바탕으로 재구성하고, 여기에 **Windows·GCC 환경에서의 실제 빌드 방법**을 더한 것입니다.
> Brian W. Kernighan, Dennis M. Ritchie, *The C Programming Language*, 2nd Edition, Prentice Hall, 1988.
{: .prompt-info }

| K&R | 원서 절 제목 | 이번 강의 |
|---|---|---|
| 4.1 | Basics of Functions | 제1절 |
| 4.2 | Functions Returning Non-integers | 제2절 |
| — | (우리 추가) 분할 컴파일 | 제3절 |
| 4.3 | External Variables | 제4절 |
| 4.4 | Scope Rules | 제5절 |
| 4.5 | Header Files | 제6절 |
| 4.6 | Static Variables | 제7절 |
| 4.7 | Register Variables | 제8절 |
| 4.8 | Block Structure | 제9절 |
| 4.9 | Initialization | 제10절 |
| 4.10 | Recursion | 제11절 |
| 4.11 | The C Preprocessor | 제12절 |

이 강의는 **4회차 분량**(모두 합쳐 약 535분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제4절 | 함수의 기본과 역폴란드 계산기 | 160분 |
| **2회차** | 제5절 ~ 제10절 | 유효 범위·헤더·`static`·초기화 | 160분 |
| **3회차** | 제11절 ~ 제13절 | 재귀와 전처리기 | 105분 |
| **4회차** | 실습문제 | 스스로 해 보기 | 110분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 함수의 기본 | 30분 |
| 제2절 | `int`가 아닌 값을 돌려주는 함수 | 30분 |
| 제3절 | 프로그램을 여러 파일로 나누기 | 40분 |
| 제4절 | 역폴란드 계산기 — 큰 예제 | 60분 |
| 제5절 | 유효 범위와 `extern` | 30분 |
| 제6절 | 헤더 파일과 실제 빌드 | 40분 |
| 제7절 | `static` | 35분 |
| 제8절 | `register` | 10분 |
| 제9절 | 블록 구조 | 20분 |
| 제10절 | 초기화 | 25분 |
| 제11절 | 재귀와 스택 프레임 | 45분 |
| 제12절 | 전처리기 | 45분 |
| 제13절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 폴더**는 `C:\c-study\lab05`를 사용합니다.

```powershell
mkdir C:\c-study\lab05
```

---

## 제1절. 함수의 기본

### 1.1 함수는 큰일을 작게 나눈다

함수는 **큰 계산을 작은 조각으로 나누고, 남이 이미 해 놓은 것 위에 쌓아 올릴 수 있게** 해 줍니다. 잘 만든 함수는 내부가 어떻게 돌아가는지 몰라도 **무엇을 하는지만 알면** 쓸 수 있습니다.

C는 함수를 만들기 쉽고 효율적으로 설계되었기 때문에, C 프로그램은 대개 **거대한 함수 몇 개가 아니라 작은 함수 여러 개**로 이루어집니다.

### 1.2 예제 — 특정 문자열이 든 줄 찾기

입력에서 `"ould"`라는 글자가 들어 있는 줄만 출력하는 프로그램을 만듭니다. 유닉스 `grep` 명령의 아주 단순한 형태입니다. 하려는 일을 우리말로 적으면 셋으로 나뉩니다.

```text
읽을 줄이 남아 있는 동안
    그 줄에 패턴이 들어 있으면
        출력한다
```

- "읽을 줄이 남아 있는 동안" → 2강에서 만든 `get_line`
- "출력한다" → 이미 있는 `printf`
- 남는 것은 **"패턴이 들어 있는가"** 하나뿐입니다. 이를 `strindex`로 만듭니다.

`find.c`입니다.

```c
#include <stdio.h>

#define MAXLINE 1000

int get_line(char line[], int max);
int strindex(const char source[], const char searchfor[]);

char pattern[] = "ould";   /* 찾을 문자열 */

/* 패턴이 들어 있는 줄을 모두 출력한다 */
int main(void)
{
    char line[MAXLINE];
    int found = 0;

    while (get_line(line, MAXLINE) > 0)
        if (strindex(line, pattern) >= 0) {
            printf("%s", line);
            found++;
        }
    printf("-- %d개 줄에서 찾았습니다.\n", found);
    return found;
}

/* get_line: 한 줄을 s 로 읽어 들이고 길이를 돌려준다 */
int get_line(char s[], int lim)
{
    int c, i;

    i = 0;
    while (--lim > 0 && (c = getchar()) != EOF && c != '\n')
        s[i++] = c;
    if (c == '\n')
        s[i++] = c;
    s[i] = '\0';
    return i;
}

/* strindex: s 안에서 t 가 처음 나타나는 위치. 없으면 -1 */
int strindex(const char s[], const char t[])
{
    int i, j, k;

    for (i = 0; s[i] != '\0'; i++) {
        for (j = i, k = 0; t[k] != '\0' && s[j] == t[k]; j++, k++)
            ;
        if (k > 0 && t[k] == '\0')
            return i;
    }
    return -1;
}
```

다음 내용을 담은 `poem.txt`로 실행합니다.

```text
Ah Love! could you and I with Fate conspire
To grasp this sorry Scheme of Things entire,
Would not we shatter it to bits -- and then
Re-mould it nearer to the Heart's Desire!
```

```powershell
.\find.exe < poem.txt
```

```text
Ah Love! could you and I with Fate conspire
Would not we shatter it to bits -- and then
Re-mould it nearer to the Heart's Desire!
-- 3개 줄에서 찾았습니다.
```

둘째 줄에는 `ould`가 없으므로 빠졌습니다. `could`·`Would`·`mould` 세 곳이 걸린 것입니다.

### 1.3 함수 정의의 형식

```text
반환자료형 함수이름(매개변수 선언 목록)
{
    선언
    문장
}
```

각 부분은 생략될 수 있습니다. 아무 일도 하지 않는 최소한의 함수는 다음과 같습니다.

```c
void dummy(void) { }
```

이런 빈 함수는 **개발 중에 자리만 잡아 두는 용도**로 쓸모가 있습니다. 나중에 채울 자리를 미리 만들어 두고, 전체 구조부터 완성하는 방식입니다.

### 1.4 `return`이 하는 일

```c
return 식;
```

식의 값을 호출한 쪽으로 돌려주고 함수를 즉시 끝냅니다. 여기서 세 가지를 기억해야 합니다.

| 사실 | 설명 |
|---|---|
| 자동 변환된다 | 식의 값은 **함수의 반환 자료형으로 변환**된 뒤 전달된다 |
| 무시할 수 있다 | 호출한 쪽은 돌려받은 값을 쓰지 않아도 된다 (`printf`의 반환값을 늘 무시하듯이) |
| 값 없이 끝날 수도 있다 | `return;` 만 쓰거나 마지막 `}`에 닿아도 된다 |

> **어떤 경로에서는 값을 돌려주고 어떤 경로에서는 돌려주지 않는 함수**는 문법 오류는 아니지만 거의 언제나 버그입니다. 값을 돌려주지 못한 채 끝나면 호출한 쪽은 **쓰레기 값**을 받습니다. `-Wall`을 켜면 컴파일러가 경고해 줍니다.
{: .prompt-danger }

`main`도 함수이므로 값을 돌려줍니다. 위 프로그램은 **찾은 줄의 개수**를 돌려주며, 이 값은 프로그램을 실행한 쪽(운영체제나 다른 프로그램)이 확인할 수 있습니다.

```powershell
.\find.exe < poem.txt; $LASTEXITCODE
```

---

## 제2절. `int`가 아닌 값을 돌려주는 함수

### 2.1 반환 자료형을 정확히 밝혀야 합니다

지금까지 만든 함수는 `void`이거나 `int`를 돌려주었습니다. 그러나 `sqrt`처럼 `double`을 돌려주는 함수도 필요합니다. 3강의 `my_atoi`(문자열 → 정수)를 실수까지 다루도록 확장한 `my_atof`를 만들어 봅니다.

`atof_demo.c`입니다.

```c
#include <stdio.h>
#include <ctype.h>

#define MAXLINE 100

double my_atof(const char s[]);
int get_line(char line[], int max);

/* 한 줄에 하나씩 입력된 수를 더해 나가는 아주 단순한 계산기 */
int main(void)
{
    double sum;
    char line[MAXLINE];

    sum = 0.0;
    while (get_line(line, MAXLINE) > 0)
        printf("\t%g\n", sum += my_atof(line));
    return 0;
}

/* my_atof: 문자열 s 를 double 로 바꾼다 */
double my_atof(const char s[])
{
    double val, power;
    int i, sign;

    for (i = 0; isspace((unsigned char) s[i]); i++)   /* 공백 건너뛰기 */
        ;
    sign = (s[i] == '-') ? -1 : 1;
    if (s[i] == '+' || s[i] == '-')
        i++;
    for (val = 0.0; isdigit((unsigned char) s[i]); i++)   /* 정수부 */
        val = 10.0 * val + (s[i] - '0');
    if (s[i] == '.')
        i++;
    for (power = 1.0; isdigit((unsigned char) s[i]); i++) {   /* 소수부 */
        val = 10.0 * val + (s[i] - '0');
        power *= 10.0;
    }
    return sign * val / power;
}
```

입력이 다음과 같을 때

```text
1.5
-2.25
10
```

```text
	1.5
	-0.75
	9.25
```

**소수부를 처리하는 방식**이 이 함수의 요령입니다. 소수점 아래 숫자도 정수처럼 계속 이어 붙이되, 자릿수만큼 `power`를 10배씩 키워 두었다가 마지막에 한 번 나눕니다. `123.45`라면 `val`은 12345, `power`는 100이 되어 결과가 123.45가 됩니다.

### 2.2 원형 선언이 없으면 무슨 일이 벌어지는가

`main`에서 `my_atof`를 쓰기 전에 다음 줄이 있었습니다.

```c
double my_atof(const char s[]);
```

이 줄이 **없으면** 어떻게 될까요? 옛 C에서는 처음 보는 이름이 괄호와 함께 나타나면 **"`int`를 돌려주는 함수"로 마음대로 가정**했습니다. 그러면 `my_atof`가 실제로는 `double`을 돌려주는데 호출한 쪽은 `int`로 받으므로 **완전히 엉뚱한 값**이 나옵니다.

더 나쁜 것은, 두 파일을 따로 컴파일하면 **컴파일러가 이 불일치를 발견조차 못 한다**는 점입니다. 같은 파일 안이라면 잡히지만, 파일이 나뉘면 각각을 따로 보기 때문입니다.

> **그래서 함수는 반드시 원형(prototype)을 선언하고 써야 합니다.**
> 현대 C 컴파일러는 원형 없는 호출에 경고를 내며, C99 이후로는 오류로 다루는 경우도 많습니다. 3절부터 배울 **헤더 파일**이 바로 이 원형들을 한곳에 모아 두는 장치입니다.
>
> 참고로 `int f();` 처럼 **괄호를 비워 두는 것**과 `int f(void);` 는 뜻이 다릅니다. 앞의 것은 "매개변수에 대해 아무것도 약속하지 않는다"는 옛 표기여서 검사가 꺼집니다. 인자가 없으면 반드시 **`void`** 라고 적으십시오.
{: .prompt-danger }

---

## 제3절. 프로그램을 여러 파일로 나누기

### 3.1 왜 나누는가

| 이유 | 설명 |
|---|---|
| 다시 쓰기 위해 | `binsearch`를 다른 프로그램에서도 쓰려면 독립된 파일이어야 한다 |
| 빨리 컴파일하려면 | 고친 파일만 다시 컴파일하고 나머지는 그대로 쓴다 |
| 함께 일하려면 | 사람마다 다른 파일을 맡을 수 있다 |
| 감추기 위해 | 내부에서만 쓰는 함수와 변수를 밖에서 보이지 않게 할 수 있다(제7절) |

### 3.2 1강에서 배운 컴파일 4단계를 다시 봅니다

1강에서 `gcc hello.c -o hello.exe` 한 줄이 사실은 네 단계였다고 하였습니다. 그중 **3단계까지를 파일마다 따로 하고, 마지막 링크에서 합치는 것**이 분할 컴파일입니다.

| 파일 | 무엇인가 | 만드는 명령 |
|---|---|---|
| `main.c` | 사람이 쓴 소스 | (편집기로 작성) |
| `main.o` | 기계어로 번역된 **목적 파일** | `gcc -c main.c -o main.o` |
| `calc.exe` | 목적 파일들을 이어 붙인 **실행 파일** | `gcc main.o stack.o ... -o calc.exe` |

**`-c` 옵션이 핵심입니다.** "링크하지 말고 목적 파일까지만 만들라"는 뜻이며, 이 덕분에 파일 하나만 고쳐도 그 파일만 다시 번역하면 됩니다.

```powershell
gcc -Wall -Wextra -std=c17 -c main.c -o main.o
```

```powershell
gcc main.o stack.o getop.o getch.o -o calc.exe
```

> **한 번에 하는 방법도 있습니다.**
> ```powershell
> gcc -Wall -Wextra -std=c17 main.c stack.c getop.c getch.c -o calc.exe
> ```
> 파일이 몇 개뿐일 때는 이 편이 간단합니다. 다만 매번 **전부** 다시 컴파일하므로, 파일이 많아지면 `-c`로 나누어 두고 바뀐 것만 다시 만드는 편이 빠릅니다. 이 과정을 자동으로 관리해 주는 도구가 11강에서 배울 **Makefile**입니다.
{: .prompt-tip }

**함수 하나를 여러 파일에 쪼개어 담을 수는 없습니다.** 나누는 단위는 언제나 함수 이상입니다.

---

## 제4절. 역폴란드 계산기 — 큰 예제

이제 K&R 제4장의 중심 예제인 **계산기**를 만듭니다. 지금까지 배운 것이 거의 모두 들어가며, 다음 절부터 이 프로그램을 재료 삼아 유효 범위·헤더·`static`을 설명합니다.

### 4.1 역폴란드 표기

우리가 쓰는 `(1 - 2) * (4 + 5)` 같은 표기는 괄호가 필요합니다. 그런데 **연산자를 피연산자 뒤에 적으면** 괄호가 필요 없어집니다.

| 보통 표기 | 역폴란드 표기 |
|---|---|
| `(1 - 2) * (4 + 5)` | `1 2 - 4 5 + *` |
| `3 + 4` | `3 4 +` |

이 방식은 **스택**만 있으면 간단히 계산됩니다.

1. 숫자가 나오면 **스택에 넣는다.**
2. 연산자가 나오면 필요한 개수만큼 **꺼내서 계산하고 결과를 다시 넣는다.**
3. 줄이 끝나면 맨 위 값을 꺼내 **출력한다.**

`1 2 - 4 5 + *`를 따라가 보면 다음과 같습니다.

| 읽은 것 | 스택(왼쪽이 아래) | 설명 |
|---|---|---|
| `1` | 1 | 넣는다 |
| `2` | 1 2 | 넣는다 |
| `-` | -1 | 2와 1을 꺼내 1−2를 넣는다 |
| `4` | -1 4 | |
| `5` | -1 4 5 | |
| `+` | -1 9 | 4+5 |
| `*` | -9 | −1×9 |
| 줄 끝 | (빔) | 꺼내어 출력: **−9** |

### 4.2 프로그램의 구조

이 프로그램은 **네 가지 일**로 나뉩니다.

| 할 일 | 담당 | 파일 |
|---|---|---|
| 전체 흐름 제어 | `main` | `main.c` |
| 값을 넣고 꺼내기 | `push`, `pop` | `stack.c` |
| 다음 연산자·피연산자 읽기 | `getop` | `getop.c` |
| 글자 하나 읽기·되돌리기 | `getch`, `ungetch` | `getch.c` |

여기서 중요한 **설계 판단**이 하나 있습니다. **스택을 어디에 둘 것인가?**

`main`이 스택을 들고 있다가 `push`·`pop`에 넘겨주는 방법도 있습니다. 그러나 `main`은 스택이 어떻게 생겼는지 알 필요가 없습니다. 그저 "넣어라", "꺼내라"만 시키면 됩니다. 그래서 **스택은 `stack.c` 안에 숨겨 두고, `push`와 `pop`만 밖으로 내놓습니다.**

### 4.3 `main.c` — 전체 흐름

```c
#include <stdio.h>
#include <stdlib.h>     /* atof */
#include "calc.h"

#define MAXOP 100       /* 피연산자·연산자의 최대 길이 */

/* 역폴란드 표기 계산기 */
int main(void)
{
    int type;
    double op2;
    char s[MAXOP];

    while ((type = getop(s)) != EOF) {
        switch (type) {
        case NUMBER:
            push(atof(s));
            break;
        case '+':
            push(pop() + pop());
            break;
        case '*':
            push(pop() * pop());
            break;
        case '-':
            op2 = pop();            /* 순서가 중요하므로 먼저 꺼내 둔다 */
            push(pop() - op2);
            break;
        case '/':
            op2 = pop();
            if (op2 != 0.0)
                push(pop() / op2);
            else
                printf("오류: 0 으로 나눌 수 없습니다\n");
            break;
        case '\n':
            printf("\t%.8g\n", pop());
            break;
        default:
            printf("오류: 알 수 없는 명령 %s\n", s);
            break;
        }
    }
    return 0;
}
```

4강에서 배운 `switch`의 **가장 전형적인 쓰임**입니다.

> **`-`와 `/`에서만 임시 변수를 쓴 이유**
> ```c
> push(pop() - pop());   /* 잘못된 코드! */
> ```
> `+`와 `*`는 순서가 바뀌어도 결과가 같지만, `-`와 `/`는 다릅니다. 그런데 3강 12.3절에서 배웠듯이 **함수 인자의 평가 순서는 정해져 있지 않으므로**, 두 `pop()` 중 어느 것이 먼저 실행될지 알 수 없습니다. 그래서 반드시 하나를 먼저 꺼내 임시 변수에 담아야 합니다.
{: .prompt-danger }

### 4.4 `stack.c` — 스택과 그 조작

```c
#include <stdio.h>
#include "calc.h"

#define MAXVAL 100      /* 스택의 최대 깊이 */

static int sp = 0;             /* 다음에 값을 넣을 자리 */
static double val[MAXVAL];     /* 값을 쌓아 두는 스택 */

/* push: 값 f 를 스택에 넣는다 */
void push(double f)
{
    if (sp < MAXVAL)
        val[sp++] = f;
    else
        printf("오류: 스택이 가득 찼습니다. %g 를 넣지 못했습니다\n", f);
}

/* pop: 스택의 맨 위 값을 꺼내 돌려준다 */
double pop(void)
{
    if (sp > 0)
        return val[--sp];
    printf("오류: 스택이 비어 있습니다\n");
    return 0.0;
}
```

`sp`와 `val`은 **두 함수가 함께 써야 하고, 호출이 끝나도 값이 남아 있어야 하므로** 함수 바깥에 두었습니다. 앞에 붙은 `static`의 뜻은 제7절에서 설명합니다.

**넘침 검사**(`sp < MAXVAL`)와 **빈 스택 검사**(`sp > 0`)를 반드시 넣어야 한다는 점에 주의하십시오. 2강에서 강조한 배열 경계 지키기가 여기서도 그대로 적용됩니다.

### 4.5 `getop.c` — 한 덩어리씩 읽기

```c
#include <stdio.h>
#include <ctype.h>
#include "calc.h"

/* getop: 다음 연산자나 피연산자를 읽어 온다 */
int getop(char s[])
{
    int i, c;

    while ((s[0] = c = getch()) == ' ' || c == '\t')   /* 공백·탭 건너뛰기 */
        ;
    s[1] = '\0';
    if (!isdigit(c) && c != '.')
        return c;                    /* 숫자가 아니면 그대로 돌려준다 */

    i = 0;
    if (isdigit(c))                  /* 정수부를 모은다 */
        while (isdigit(s[++i] = c = getch()))
            ;
    if (c == '.')                    /* 소수부를 모은다 */
        while (isdigit(s[++i] = c = getch()))
            ;
    s[i] = '\0';
    if (c != EOF)
        ungetch(c);                  /* 한 글자 더 읽었으므로 되돌린다 */
    return NUMBER;
}
```

### 4.6 `getch.c` — 읽은 글자를 되돌리기

여기에 이 예제에서 가장 배울 점이 많은 부분이 있습니다.

**숫자가 어디서 끝나는지 알려면 숫자가 아닌 글자를 하나 읽어 봐야 합니다.** 그런데 그 글자는 다음 차례에 쓸 것이므로 버리면 안 됩니다. 그래서 **"읽은 글자를 입력으로 되돌리는"** 기능을 직접 만듭니다.

```c
#include <stdio.h>
#include "calc.h"

#define BUFSIZE 100

static char buf[BUFSIZE];   /* ungetch 로 되돌린 글자를 담아 두는 곳 */
static int bufp = 0;        /* buf 의 다음 빈 자리 */

/* getch: 되돌린 글자가 있으면 그것을, 없으면 새 글자를 읽는다 */
int getch(void)
{
    return (bufp > 0) ? buf[--bufp] : getchar();
}

/* ungetch: 읽은 글자를 입력으로 되돌린다 */
void ungetch(int c)
{
    if (bufp >= BUFSIZE)
        printf("오류: 되돌릴 수 있는 글자 수를 넘었습니다\n");
    else
        buf[bufp++] = c;
}
```

`getch`와 `ungetch`는 **버퍼를 함께 쓰는 한 쌍**이며, 그 버퍼는 호출과 호출 사이에 값이 남아 있어야 하므로 함수 바깥에 있습니다. 그러나 **바깥에서는 보일 필요가 없으므로** `static`으로 감춥니다.

> 표준 라이브러리에도 한 글자를 되돌리는 `ungetc` 함수가 있습니다(9강에서 다룹니다). 여기서는 원리를 이해하기 위해 직접 만들어 봅니다.
{: .prompt-info }

---

> **▶ 여기서부터 2회차 — 유효 범위·헤더·`static`·초기화**
> 제5절 ~ 제10절, 약 160분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제5절. 유효 범위와 `extern`

### 5.1 이름이 보이는 범위

**유효 범위(scope)** 란 어떤 이름을 쓸 수 있는 프로그램의 영역입니다.

| 대상 | 유효 범위 |
|---|---|
| 함수 안에서 선언한 변수 | **그 함수 안**(정확히는 선언된 블록 안) |
| 매개변수 | 그 함수 안 |
| 함수 바깥에서 선언한 변수(외부 변수) | **선언된 지점부터 그 파일의 끝까지** |

세 번째 줄이 중요합니다. 외부 변수라 해도 **선언보다 앞에 있는 함수에서는 보이지 않습니다.**

### 5.2 선언과 정의

2강에서 잠깐 다룬 구분을 이제 정확히 합니다.

| 용어 | 하는 일 | 예 |
|---|---|---|
| **정의(definition)** | 변수를 실제로 만들고 **저장 공간을 잡는다** | `int sp;` |
| **선언(declaration)** | "이런 이름이 어딘가에 있다"고 **알리기만 한다** | `extern int sp;` |

규칙은 다음과 같습니다.

- 하나의 프로그램 전체에서 외부 변수의 **정의는 정확히 한 번**만 있어야 한다.
- 그 변수를 쓰는 다른 파일에서는 **`extern` 선언**을 해야 한다.
- **초기값은 정의에만** 붙일 수 있다.
- 배열의 크기는 **정의에는 반드시** 적어야 하고, `extern` 선언에서는 생략할 수 있다.

```c
/* file1.c — 정의 */
int sp = 0;
double val[MAXVAL];
```

```c
/* file2.c — 선언 */
extern int sp;
extern double val[];
```

같은 파일 안에서 정의가 사용보다 **앞에** 있으면 `extern`은 필요 없습니다. 그래서 실무에서는 **외부 변수의 정의를 파일 맨 앞에 모아 두는** 것이 관례입니다.

> **함수는 기본적으로 외부(external)입니다.** C는 함수 안에 함수를 정의할 수 없으므로, 모든 함수는 파일 바깥 수준에 있으며 다른 파일에서도 이름으로 부를 수 있습니다. 그래서 함수 원형에는 `extern`을 붙이지 않아도 됩니다(붙여도 무방합니다).
{: .prompt-info }

---

## 제6절. 헤더 파일과 실제 빌드

### 6.1 공유할 것을 한곳에 모읍니다

계산기를 네 파일로 나누면 문제가 하나 생깁니다. `main.c`는 `push`·`pop`·`getop`의 원형을 알아야 하고, `getop.c`는 `getch`·`ungetch`의 원형을 알아야 합니다. 이것을 각 파일에 따로 적으면 **어느 하나를 고칠 때 나머지를 함께 고쳐야** 합니다.

그래서 **여러 파일이 공유하는 선언을 헤더 파일 하나에 모으고**, 필요한 파일에서 `#include`합니다.

`calc.h`입니다.

```c
#ifndef CALC_H          /* 중복 포함 방지 */
#define CALC_H

#define NUMBER '0'      /* 숫자를 찾았다는 표시 */

void   push(double f);      /* 값을 스택에 넣는다 */
double pop(void);           /* 스택에서 값을 꺼낸다 */
int    getop(char s[]);     /* 다음 연산자나 피연산자를 읽는다 */
int    getch(void);         /* 한 글자를 읽는다(되돌린 글자 우선) */
void   ungetch(int c);      /* 읽은 글자를 입력으로 되돌린다 */

#endif   /* CALC_H */
```

### 6.2 중복 포함 방지 — `#ifndef`

헤더가 여러 경로로 두 번 포함되면 같은 선언이 두 번 나타나 오류가 날 수 있습니다. 이를 막는 관용구가 위의 세 줄입니다.

| 줄 | 뜻 |
|---|---|
| `#ifndef CALC_H` | `CALC_H`가 아직 정의되지 않았다면 아래를 처리하라 |
| `#define CALC_H` | 이제 정의한다(다음번에는 건너뛴다) |
| `#endif` | 여기까지 |

**모든 헤더 파일에 반드시 넣어야 하는 습관**이며, 이름은 파일 이름을 대문자로 바꾼 형태를 씁니다. 자세한 문법은 제12절에서 다룹니다.

### 6.3 `#include "..."` 와 `#include <...>`

| 표기 | 찾는 곳 |
|---|---|
| `#include "calc.h"` | **내 소스가 있는 폴더부터** 찾고, 없으면 표준 경로 |
| `#include <stdio.h>` | 컴파일러가 정한 **표준 경로**에서만 찾음 |

즉 **내가 만든 헤더는 큰따옴표, 시스템이 제공하는 헤더는 꺾쇠**입니다.

### 6.4 전체 파일 구성

| 파일 | 포함하는 것 | 담고 있는 것 |
|---|---|---|
| `calc.h` | — | `NUMBER`, 함수 원형 5개 |
| `main.c` | `<stdio.h>` `<stdlib.h>` `"calc.h"` | `main`, `MAXOP` |
| `stack.c` | `<stdio.h>` `"calc.h"` | `sp`, `val`, `push`, `pop`, `MAXVAL` |
| `getop.c` | `<stdio.h>` `<ctype.h>` `"calc.h"` | `getop` |
| `getch.c` | `<stdio.h>` `"calc.h"` | `buf`, `bufp`, `getch`, `ungetch`, `BUFSIZE` |

각 파일이 **자기 일에 필요한 것만** 포함하고 있음을 확인하십시오. `main.c`는 스택이 배열이라는 사실조차 모릅니다.

### 6.5 빌드하고 실행하기

```powershell
gcc -Wall -Wextra -std=c17 -c main.c  -o main.o
```

```powershell
gcc -Wall -Wextra -std=c17 -c stack.c -o stack.o
```

```powershell
gcc -Wall -Wextra -std=c17 -c getop.c -o getop.o
```

```powershell
gcc -Wall -Wextra -std=c17 -c getch.c -o getch.o
```

```powershell
gcc main.o stack.o getop.o getch.o -o calc.exe
```

입력 파일 `calc_in.txt`를 만들어 실행합니다.

```text
1 2 - 4 5 + *
3 4 +
10 2 /
```

```powershell
.\calc.exe < calc_in.txt
```

```text
	-9
	7
	5
```

4.1절에서 손으로 따라간 `1 2 - 4 5 + *`의 결과 **−9**가 그대로 나왔습니다.

> **자주 만나는 링크 오류**
>
> | 메시지 | 원인 | 해결 |
> |---|---|---|
> | `undefined reference to 'push'` | `stack.o`를 링크에 넣지 않음 | 링크 명령에 빠진 `.o`를 추가 |
> | `multiple definition of 'sp'` | 같은 이름을 두 파일에서 **정의** | 한쪽만 정의하고 다른 쪽은 `extern`, 또는 `static`으로 감춘다 |
> | `calc.h: No such file or directory` | 헤더 경로 문제 | 같은 폴더에 두거나 `-I경로` 지정(11강) |
{: .prompt-warning }

---

## 제7절. `static`

`static`은 위치에 따라 **두 가지 전혀 다른 뜻**을 가집니다. 이 구분이 이 절의 전부입니다.

### 7.1 파일 바깥에서 감추기

함수 바깥에 있는 변수나 함수 앞에 `static`을 붙이면, 그 이름은 **그 파일 안에서만** 보입니다.

```c
static int sp = 0;             /* 다른 파일에서는 이 이름을 쓸 수 없다 */
static double val[MAXVAL];
```

계산기의 `sp`·`val`·`buf`·`bufp`가 모두 이 형태입니다. 얻는 것이 두 가지입니다.

1. **다른 파일이 실수로 건드릴 수 없습니다.** `main.c`가 `sp`를 직접 바꾸는 일이 원천적으로 불가능해집니다.
2. **이름 충돌이 사라집니다.** 다른 파일에 우연히 같은 이름의 변수가 있어도 서로 아무 관계가 없습니다.

함수에도 붙일 수 있습니다.

```c
static void helper(void) { ... }   /* 이 파일 안에서만 부를 수 있는 함수 */
```

> **원칙: 밖에서 쓸 필요가 없는 것은 모두 `static`으로 감추십시오.**
> 헤더에 원형을 적은 것만 공개하고 나머지는 감추는 것이 좋은 파일 설계입니다. 2강 11.4절에서 외부 변수를 남용하지 말라고 한 이야기의 실천 방법이기도 합니다.
{: .prompt-tip }

### 7.2 함수 안에서 값을 기억하기

함수 **안**에 선언한 변수에 `static`을 붙이면 뜻이 달라집니다. 그 변수는 여전히 그 함수만의 것이지만, **함수가 끝나도 사라지지 않고 값을 유지**합니다.

`static_demo.c`입니다.

```c
#include <stdio.h>

/* 함수 안의 static 변수는 호출이 끝나도 값을 잃지 않는다 */
int call_count(void)
{
    static int count = 0;   /* 처음 한 번만 0 이 되고, 이후에는 값이 유지된다 */
    int normal = 0;         /* 호출될 때마다 새로 만들어진다 */

    count++;
    normal++;
    printf("static count = %d, 보통 변수 normal = %d\n", count, normal);
    return count;
}

int main(void)
{
    int i;

    for (i = 0; i < 4; i++)
        call_count();
    return 0;
}
```

```text
static count = 1, 보통 변수 normal = 1
static count = 2, 보통 변수 normal = 1
static count = 3, 보통 변수 normal = 1
static count = 4, 보통 변수 normal = 1
```

`count`는 계속 늘어나는데 `normal`은 언제나 1입니다. **보통의 지역 변수는 호출될 때마다 새로 만들어지고 끝나면 사라지지만, `static` 지역 변수는 프로그램이 시작될 때 한 번 만들어져 끝날 때까지 남아 있기** 때문입니다.

| 구분 | 만들어지는 시점 | 사라지는 시점 | 초기값 |
|---|---|---|---|
| 보통 지역 변수 | 함수가 호출될 때 | 함수가 끝날 때 | 지정하지 않으면 **쓰레기 값** |
| `static` 지역 변수 | 프로그램 시작 시 한 번 | 프로그램 종료 시 | 지정하지 않으면 **0** |
| 외부 변수 | 프로그램 시작 시 | 프로그램 종료 시 | 지정하지 않으면 **0** |

---

## 제8절. `register`

변수 앞에 `register`를 붙이면 "이 변수는 매우 자주 쓰이니 되도록 **CPU 레지스터**에 두라"고 컴파일러에 권고하는 뜻이 됩니다.

```c
register int i;
```

다만 이는 **권고일 뿐이며 컴파일러는 무시할 수 있습니다.** 그리고 `register` 변수는 **주소를 얻을 수 없습니다**(6강에서 배울 `&` 연산자를 쓸 수 없습니다).

> 오늘날의 컴파일러는 어떤 변수를 레지스터에 둘지 사람보다 훨씬 잘 판단합니다. 그래서 **`register`는 실무에서 거의 쓰이지 않습니다.** 옛 코드를 읽을 때 알아볼 수 있으면 충분합니다.
{: .prompt-info }

---

## 제9절. 블록 구조

C는 함수 안에 함수를 정의할 수 없습니다. 그러나 **변수는 어떤 블록에서든 선언할 수 있으며**, 그 블록 안에서만 유효합니다. 그리고 **바깥의 같은 이름을 가립니다.**

`scope_demo.c`입니다.

```c
#include <stdio.h>

int x = 100;      /* 외부 변수 */

void f(double x)  /* 매개변수가 외부 변수 x 를 가린다 */
{
    printf("f 안의 x = %.1f  (double, 매개변수)\n", x);
}

int main(void)
{
    printf("main 의 x = %d  (외부 변수)\n", x);
    f(3.5);

    {
        int x = -7;    /* 블록 안에서 새로 선언 — 바깥 x 를 가린다 */
        printf("블록 안의 x = %d\n", x);
    }

    printf("블록을 벗어난 x = %d\n", x);
    return 0;
}
```

```text
main 의 x = 100  (외부 변수)
f 안의 x = 3.5  (double, 매개변수)
블록 안의 x = -7
블록을 벗어난 x = 100
```

블록을 벗어나자 다시 외부 변수 `x`가 보입니다. 블록 안의 `x`는 **완전히 다른 변수**였던 것입니다.

> **바깥 이름을 가리는 변수는 되도록 만들지 마십시오.** 문법적으로는 허용되지만, 읽는 사람이 어느 `x`인지 헷갈리게 만들어 버그의 원인이 됩니다. `-Wshadow` 옵션을 붙이면 컴파일러가 이런 경우를 경고해 줍니다.
{: .prompt-warning }

---

## 제10절. 초기화

### 10.1 초기값의 규칙

| 대상 | 명시하지 않았을 때 | 초기값으로 쓸 수 있는 것 |
|---|---|---|
| 외부 변수·`static` 변수 | **0** | 상수식만 |
| 지역(자동) 변수 | **쓰레기 값** | 어떤 식이든(다른 변수, 함수 호출도 가능) |

지역 변수의 초기화는 사실상 **대입문의 줄임 표기**입니다. 그래서 함수가 호출될 때마다 다시 실행됩니다.

```c
int binsearch(int x, const int v[], int n)
{
    int low = 0;
    int high = n - 1;      /* 매개변수를 이용한 초기화도 된다 */
    int mid;
    ...
}
```

### 10.2 배열의 초기화

`init_demo.c`입니다.

```c
#include <stdio.h>

int g_int;                 /* 외부 변수 — 자동으로 0 */
int g_arr[5];              /* 외부 배열 — 모두 0 */

int main(void)
{
    int days[] = {31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
    int part[5] = {1, 2};              /* 나머지는 0 으로 채워진다 */
    char pattern[] = "ould";           /* '\0' 까지 5칸 */
    int i, n;

    n = (int) (sizeof days / sizeof days[0]);
    printf("days 원소 개수 = %d\n", n);
    printf("days =");
    for (i = 0; i < n; i++)
        printf(" %d", days[i]);
    printf("\n");

    printf("part =");
    for (i = 0; i < 5; i++)
        printf(" %d", part[i]);
    printf("   <- 지정하지 않은 자리는 0\n");

    printf("pattern = \"%s\", sizeof = %zu\n", pattern, sizeof pattern);
    printf("외부 변수 g_int = %d, g_arr[3] = %d   <- 자동으로 0\n", g_int, g_arr[3]);
    return 0;
}
```

```text
days 원소 개수 = 12
days = 31 28 31 30 31 30 31 31 30 31 30 31
part = 1 2 0 0 0   <- 지정하지 않은 자리는 0
pattern = "ould", sizeof = 5
외부 변수 g_int = 0, g_arr[3] = 0   <- 자동으로 0
```

여기서 배울 것이 네 가지입니다.

1. **크기를 생략하면** 초기값의 개수를 세어 컴파일러가 크기를 정합니다. `days`는 12가 되었습니다.
2. **초기값이 모자라면** 나머지는 0으로 채워집니다.
3. **초기값이 남으면** 오류입니다.
4. **문자 배열은 문자열로 초기화**할 수 있습니다. `"ould"`는 `{'o','u','l','d','\0'}`와 같으므로 크기가 **5**입니다.

---

> **▶ 여기서부터 3회차 — 재귀와 전처리기**
> 제11절 ~ 제13절, 약 105분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제11절. 재귀와 스택 프레임

### 11.1 자기 자신을 부르는 함수

C 함수는 **자기 자신을 부를 수 있습니다.** 이를 **재귀 호출(recursion)** 이라 합니다.

4강의 `itoa`를 떠올려 보십시오. 자릿수를 뽑으면 거꾸로 나오므로 배열에 담았다가 뒤집어야 했습니다. 그런데 **앞자리를 먼저 출력하도록 자기 자신을 부르면** 뒤집을 필요가 없습니다.

`printd.c`입니다.

```c
#include <stdio.h>

/* printd: n 을 10진수로 출력한다 (재귀) */
void printd(int n)
{
    if (n < 0) {
        putchar('-');
        n = -n;
    }
    if (n / 10)
        printd(n / 10);      /* 앞자리를 먼저 출력하도록 자기 자신을 부른다 */
    putchar(n % 10 + '0');
}

int main(void)
{
    printd(123);   putchar('\n');
    printd(-4567); putchar('\n');
    printd(0);     putchar('\n');
    return 0;
}
```

```text
123
-4567
0
```

`printd(123)`이 어떻게 동작하는지 따라가 봅니다.

| 단계 | 하는 일 |
|---|---|
| `printd(123)` | 12가 0이 아니므로 `printd(12)`를 부른다 |
| `printd(12)` | 1이 0이 아니므로 `printd(1)`을 부른다 |
| `printd(1)` | 0이므로 더 부르지 않고 `1`을 출력하고 돌아간다 |
| `printd(12)`로 복귀 | `2`를 출력하고 돌아간다 |
| `printd(123)`으로 복귀 | `3`을 출력하고 끝난다 |

결과는 `123`입니다. **먼저 부른 쪽이 나중에 출력한다**는 점이 재귀의 묘미입니다.

### 11.2 퀵 정렬

재귀가 진가를 발휘하는 예가 **퀵 정렬**입니다. 배열에서 기준값 하나를 고르고, 그보다 작은 것과 크거나 같은 것으로 나눈 뒤, **각각에 같은 방법을 다시 적용**합니다.

`qsort_demo.c`입니다.

```c
/* my_qsort: v[left]...v[right] 를 오름차순으로 정렬한다 */
void my_qsort(int v[], int left, int right)
{
    int i, last;

    if (left >= right)             /* 원소가 두 개 미만이면 할 일이 없다 */
        return;
    swap(v, left, (left + right) / 2);   /* 기준값을 맨 앞으로 */
    last = left;
    for (i = left + 1; i <= right; i++)  /* 기준보다 작은 것을 앞으로 모은다 */
        if (v[i] < v[left])
            swap(v, ++last, i);
    swap(v, left, last);                 /* 기준값을 제자리에 놓는다 */
    my_qsort(v, left, last - 1);         /* 왼쪽 부분을 정렬 */
    my_qsort(v, last + 1, right);        /* 오른쪽 부분을 정렬 */
}

/* swap: v[i] 와 v[j] 를 맞바꾼다 */
void swap(int v[], int i, int j)
{
    int temp;

    temp = v[i];
    v[i] = v[j];
    v[j] = temp;
}
```

```text
정렬 전: 38 5 91 2 56 12 72 23 16 8
정렬 후: 2 5 8 12 16 23 38 56 72 91
```

4강의 셸 정렬과 같은 배열을 정렬했습니다. **재귀를 멈추는 조건**(`left >= right`)이 반드시 있어야 한다는 점에 유의하십시오. 없으면 영원히 자기를 부르다가 프로그램이 죽습니다.

`swap`을 따로 만든 이유는 같은 동작이 **세 번** 나오기 때문입니다. 반복되는 것을 함수로 묶는 것이 곧 좋은 설계입니다.

### 11.3 호출마다 지역 변수가 새로 생깁니다 — 스택 프레임

재귀가 가능한 이유는 **함수가 호출될 때마다 지역 변수 한 벌이 새로 만들어지기** 때문입니다. `printd(123)`의 `n`과 `printd(12)`의 `n`은 이름만 같을 뿐 서로 다른 변수입니다.

이 "한 벌"이 놓이는 자리를 **스택 프레임(stack frame)** 이라 하며, 함수가 호출될 때마다 차곡차곡 쌓였다가 돌아갈 때 하나씩 걷힙니다. 프레임에는 다음이 함께 들어갑니다.

| 프레임에 들어가는 것 | 설명 |
|---|---|
| 지역 변수와 배열 | 함수가 끝나면 사라진다 |
| 매개변수 | 값에 의한 호출로 복사된 것 |
| **돌아갈 주소** | 함수가 끝난 뒤 **어디로 복귀할지** 적어 둔 것 |

`frame_demo.c`로 눈으로 확인할 수 있습니다.

```c
#include <stdio.h>

/* 재귀 호출마다 지역 변수가 새로 생기는 것을 주소로 확인한다 */
void depth(int n)
{
    char buf[16];      /* 이 함수의 지역 배열 */
    int  marker = n;

    printf("깊이 %d: buf 주소 = %p, marker 주소 = %p\n",
           n, (void *) buf, (void *) &marker);
    if (n > 1)
        depth(n - 1);
}

int main(void)
{
    depth(4);
    return 0;
}
```

실행하면 다음과 같은 모양이 나옵니다. **구체적인 주소 값은 실행할 때마다 달라지므로** 숫자 자체가 아니라 **규칙**을 보시기 바랍니다.

```text
깊이 4: buf 주소 = 000000000061FDF0, marker 주소 = 000000000061FE0C
깊이 3: buf 주소 = 000000000061FDC0, marker 주소 = 000000000061FDDC
깊이 2: buf 주소 = 000000000061FD90, marker 주소 = 000000000061FDAC
깊이 1: buf 주소 = 000000000061FD60, marker 주소 = 000000000061FD7C
```

깊이 들어갈수록 주소가 **일정한 간격으로 낮아집니다.** 호출할 때마다 프레임이 하나씩 새로 쌓이고 있다는 뜻입니다. (`%p`는 주소를 출력하는 서식 지정자이며, 6강에서 자세히 다룹니다.)

> **여기가 2강에서 예고한 버퍼 오버플로의 원리입니다.**
> 위 그림에서 `buf`는 16칸짜리 배열이고, 그 **바로 위쪽에 `marker`가, 더 위쪽에는 돌아갈 주소가** 놓여 있습니다. 만약 `buf`에 16칸을 넘겨 쓰면 어떻게 될까요?
>
> 1. 먼저 옆에 있는 다른 지역 변수(`marker`)의 값이 조용히 바뀝니다.
> 2. 더 많이 넘치면 **돌아갈 주소까지 덮어씁니다.**
> 3. 그러면 함수가 끝날 때 프로그램은 **원래 자리가 아닌 엉뚱한 곳으로 돌아갑니다.**
>
> 공격자가 그 "엉뚱한 곳"을 마음대로 정할 수 있다면 프로그램의 흐름을 통째로 빼앗을 수 있습니다. 이것이 수십 년간 가장 많은 보안 사고를 일으킨 **스택 버퍼 오버플로**입니다.
>
> 그래서 배열에 쓸 때는 **언제나 크기를 함께 넘기고 검사해야** 합니다. 계산기의 `push`가 `sp < MAXVAL`을 검사한 것, `get_line`이 `lim - 1`을 지킨 것이 모두 같은 이유입니다. 실제로 값이 바뀌는 모습은 **7강**에서 확인하고, 막는 방법은 **10강**에서 다룹니다.
{: .prompt-danger }

### 11.4 재귀의 장단점

| 장점 | 단점 |
|---|---|
| 코드가 짧고 문제의 정의를 그대로 옮길 수 있다 | 호출마다 프레임이 쌓이므로 **메모리를 더 쓴다** |
| 트리처럼 재귀적으로 정의된 자료에 잘 맞는다 | 함수 호출 비용이 있어 **더 빠르지도 않다** |
| — | 멈추는 조건을 빠뜨리면 **스택이 넘쳐 프로그램이 죽는다** |

---

## 제12절. 전처리기

전처리기는 **컴파일 이전에 소스를 글자 수준에서 손보는 단계**입니다(1강의 컴파일 4단계 중 1단계). 세 가지 기능을 봅니다.

### 12.1 파일 포함 `#include`

```c
#include "파일이름"      /* 내 폴더부터 찾는다 */
#include <파일이름>      /* 표준 경로에서 찾는다 */
```

그 자리에 **파일의 내용이 통째로 들어옵니다.** 포함된 파일 안에 또 `#include`가 있어도 됩니다.

큰 프로그램에서 선언을 묶는 **가장 좋은 방법**이 `#include`입니다. 모든 소스 파일이 같은 선언을 보게 되므로, 선언이 어긋나서 생기는 고약한 버그가 사라집니다. 다만 **헤더를 고치면 그것을 포함한 모든 파일을 다시 컴파일해야 한다**는 점을 기억하십시오.

### 12.2 매크로 치환 `#define`

```c
#define 이름 대체할내용
```

2강에서 상수에 이름을 붙일 때 이미 썼습니다. 여기에 **인자를 받는 형태**도 있습니다.

```c
#define MAX(A, B)   ((A) > (B) ? (A) : (B))
```

함수 호출처럼 보이지만 **함수가 아니라 글자 치환**입니다. `x = MAX(p + q, r + s);` 는 다음으로 펼쳐집니다.

```c
x = ((p + q) > (r + s) ? (p + q) : (r + s));
```

자료형에 상관없이 쓸 수 있다는 것이 장점입니다. 그러나 **함정이 두 가지** 있습니다.

`macro_demo.c`로 확인합니다.

```c
#include <stdio.h>

#define SQUARE_BAD(x)   x * x                 /* 괄호가 없어 위험하다 */
#define SQUARE_OK(x)    ((x) * (x))

#define MAX_OK(A, B)    ((A) > (B) ? (A) : (B))

#define DPRINT(expr)    printf(#expr " = %g\n", (double)(expr))

int main(void)
{
    int z = 3;
    int i, j, m;

    printf("SQUARE_BAD(z + 1) = %d   <- z + 1 * z + 1 로 펼쳐진다\n", SQUARE_BAD(z + 1));
    printf("SQUARE_OK (z + 1) = %d\n", SQUARE_OK(z + 1));

    i = 5;
    j = 2;
    printf("MAX_OK(i, j) = %d\n", MAX_OK(i, j));

    /* 부작용이 있는 인자는 두 번 평가된다 */
    i = 5;
    j = 2;
    m = MAX_OK(i++, j++);          /* 결과를 먼저 변수에 담는다 */
    printf("m = MAX_OK(i++, j++) -> m = %d, i = %d, j = %d   <- i 가 두 번 증가\n",
           m, i, j);

    i = 7;
    DPRINT(i / 2.0);
    return 0;
}
```

```text
SQUARE_BAD(z + 1) = 7   <- z + 1 * z + 1 로 펼쳐진다
SQUARE_OK (z + 1) = 16
MAX_OK(i, j) = 5
m = MAX_OK(i++, j++) -> m = 6, i = 7, j = 3   <- i 가 두 번 증가
i / 2.0 = 3.5
```

**함정 ① 괄호를 빠뜨리면 순서가 어긋납니다.**

`SQUARE_BAD(z + 1)`은 `z + 1 * z + 1`로 펼쳐집니다. `z`가 3이면 `3 + 3 + 1 = 7`이 되어 16과 전혀 다릅니다. 그래서 **매크로의 인자와 전체를 모두 괄호로 감싸야** 합니다.

**함정 ② 인자가 두 번 평가됩니다.**

`MAX_OK(i++, j++)`에서 큰 쪽의 `i++`가 **두 번** 실행되어 `i`가 5에서 7이 되었습니다. 매크로 인자에는 **부작용이 있는 식(`++`, 함수 호출, 입출력)을 넣지 마십시오.**

**`#` 연산자** — 매크로 인자 앞에 `#`를 붙이면 **그 인자를 글자 그대로 문자열로** 만듭니다. 위의 `DPRINT(i / 2.0)`이 `"i / 2.0"`이라는 문자열과 값을 함께 출력한 것이 그 예이며, 디버깅용 출력에 유용합니다.

**`##` 연산자**는 두 인자를 이어 붙여 하나의 이름을 만듭니다. 쓸 일이 드물지만 라이브러리 코드에서 종종 보입니다.

**`#undef`** 로 정의를 취소할 수 있습니다.

> **매크로와 함수 중 무엇을 쓸 것인가**
> 오늘날에는 **웬만하면 함수를 쓰십시오.** 컴파일러가 짧은 함수를 알아서 인라인으로 펼쳐 주므로 속도 차이가 거의 없고, 함수는 자료형 검사를 받으며 위의 두 함정도 없습니다. 매크로는 **상수 정의**와 **조건부 컴파일**에 주로 사용하십시오.
{: .prompt-tip }

### 12.3 조건부 컴파일

특정 조건일 때만 코드를 컴파일하도록 지시할 수 있습니다.

`cond_include.c`입니다.

```c
#include <stdio.h>

#define DEBUG 1

int main(void)
{
#ifdef DEBUG
    printf("[디버그] 이 줄은 DEBUG 가 정의되어 있을 때만 컴파일됩니다.\n");
#endif

#if DEBUG >= 1
    printf("[디버그] DEBUG 값 = %d\n", DEBUG);
#else
    printf("배포판입니다.\n");
#endif

#ifndef RELEASE
    printf("RELEASE 는 정의되어 있지 않습니다.\n");
#endif

    printf("컴파일한 파일: %s, 줄 번호: %d\n", __FILE__, __LINE__);
    return 0;
}
```

```text
[디버그] 이 줄은 DEBUG 가 정의되어 있을 때만 컴파일됩니다.
[디버그] DEBUG 값 = 1
RELEASE 는 정의되어 있지 않습니다.
컴파일한 파일: cond_include.c, 줄 번호: 21
```

| 지시문 | 뜻 |
|---|---|
| `#if 식` | 상수식이 참이면 포함 |
| `#ifdef 이름` | 그 이름이 **정의되어 있으면** 포함 |
| `#ifndef 이름` | 정의되어 **있지 않으면** 포함 |
| `#elif` / `#else` / `#endif` | `else if` / `else` / 끝 |

`__FILE__`과 `__LINE__`은 컴파일러가 미리 정의해 둔 매크로로, 각각 **파일 이름**과 **줄 번호**로 치환됩니다. 오류 메시지를 만들 때 유용합니다.

**컴파일할 때 값을 넘길 수도 있습니다.**

```powershell
gcc -Wall -Wextra -std=c17 -DDEBUG=0 cond_include.c -o cond.exe
```

`-D이름=값` 옵션은 소스를 고치지 않고 `#define`을 추가한 것과 같은 효과를 냅니다. 디버그판과 배포판을 같은 소스로 만들 때 쓰는 방법입니다.

그리고 6.2절의 **중복 포함 방지**가 바로 이 기능의 응용이었습니다.

```c
#ifndef CALC_H
#define CALC_H
   ... 헤더의 내용 ...
#endif
```

---

## 제13절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| `undefined reference to '함수이름'` | 링크에 `.o` 파일을 빠뜨림 | 링크 명령에 모든 목적 파일을 넣는다 |
| `multiple definition of '변수'` | 헤더에 변수를 **정의**해 여러 파일에 퍼짐 | 헤더에는 `extern` 선언만, 정의는 `.c` 한 곳에 |
| 헤더를 고쳤는데 반영되지 않음 | 그 헤더를 쓰는 파일을 다시 컴파일하지 않음 | 관련 `.o`를 지우고 다시 빌드 |
| 같은 헤더가 두 번 포함되어 오류 | 중복 포함 방지 누락 | `#ifndef`·`#define`·`#endif` 추가 |
| 함수 반환값이 이상함 | 원형 선언 없이 호출 | 헤더에 원형을 두고 `#include` |
| 매크로 결과가 예상과 다름 | 괄호 누락 | 인자와 전체를 모두 괄호로 감싼다 |
| 매크로 인자의 값이 두 번 바뀜 | 인자에 `++` 등 부작용 | 매크로 인자에 부작용을 넣지 않는다 |
| 재귀가 끝나지 않고 죽음 | 멈추는 조건 누락 | 종료 조건을 먼저 작성한다 |
| 지역 변수 값이 매번 초기화됨 | `static`을 빠뜨림 | 값을 유지하려면 `static` |
| 다른 파일에서 내부 변수를 건드림 | `static`으로 감추지 않음 | 공개할 것만 헤더에, 나머지는 `static` |

---

> **▶ 여기서부터 4회차 — 스스로 해 보기**
> 실습문제, 약 110분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 실습문제

> **안내**
> 1. 모든 문제는 `C:\c-study\lab05`에서 수행합니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다. **먼저 스스로 작성한 뒤** 확인하시기 바랍니다.
> 3. 컴파일은 `gcc -Wall -Wextra -std=c17 …` 형식으로 하며 **경고 0개**를 목표로 합니다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 | K&R |
|---|---|---|---|
| 문제 1 | `strrindex` — 가장 오른쪽 위치 | 1.2 | 4-1 |
| 문제 2 | `atof`에 지수 표기 추가 | 2.1 | 4-2 |
| 문제 3 | 계산기에 `%`와 음수 넣기 | 4 | 4-3 |
| 문제 4 | 계산기에 스택 명령 추가 | 4 · 7 | 4-4 |
| 문제 5 | `ungetch` 없이 `getop` 만들기 | 7.2 | 4-11 |
| 문제 6 | 재귀 `itoa` | 11.1 | 4-12 |
| 문제 7 | 재귀 `reverse` | 11.1 | 4-13 |
| 문제 8 | `SWAP` 매크로 | 12.2 | 4-14 |
| 문제 9 | 4강 함수들을 모듈로 분리 | 3 · 6 | — |
| 문제 10 | `static`으로 호출 횟수 세기 | 7 | — |

---

### 문제 1. `strrindex`

`strindex`는 패턴이 **처음** 나타나는 위치를 돌려줍니다. **가장 오른쪽**에 나타나는 위치를 돌려주는 `strrindex`를 작성하십시오.

**정답 및 해설**

```c
#include <stdio.h>

/* strrindex: s 안에서 t 가 나타나는 가장 오른쪽 위치. 없으면 -1 */
int strrindex(const char s[], const char t[])
{
    int i, j, k, pos;

    pos = -1;
    for (i = 0; s[i] != '\0'; i++) {
        for (j = i, k = 0; t[k] != '\0' && s[j] == t[k]; j++, k++)
            ;
        if (k > 0 && t[k] == '\0')
            pos = i;              /* 찾을 때마다 갱신 — 마지막 것이 남는다 */
    }
    return pos;
}

int main(void)
{
    const char line[] = "would you could you should you";

    printf("strrindex(\"%s\", \"ould\") = %d\n", line, strrindex(line, "ould"));
    printf("strrindex(\"%s\", \"xyz\")  = %d\n", line, strrindex(line, "xyz"));
    return 0;
}
```

```text
strrindex("would you could you should you", "ould") = 22
strrindex("would you could you should you", "xyz")  = -1
```

- **찾자마자 돌려주지 않고 위치만 기억해 두는 것**이 요령입니다. 끝까지 훑으면 마지막으로 기억된 값이 곧 가장 오른쪽 위치입니다.
- `would`(1), `could`(11), `should`(22) 세 곳에 `ould`가 있으므로 답은 **22**입니다.
- 뒤에서부터 훑으면 첫 번째로 찾은 것이 답이므로 더 빠릅니다. 그렇게 고쳐 보는 것도 좋은 연습입니다.

---

### 문제 2. `atof`에 지수 표기 추가

`123.45e-6` 같은 지수 표기를 처리하도록 `my_atof`를 확장하십시오.

**정답 및 해설**

```c
    val = sign * val / power;

    if (s[i] == 'e' || s[i] == 'E') {       /* 지수부 */
        i++;
        exp_sign = (s[i] == '-') ? -1 : 1;
        if (s[i] == '+' || s[i] == '-')
            i++;
        for (exponent = 0; isdigit((unsigned char) s[i]); i++)
            exponent = 10 * exponent + (s[i] - '0');
        while (exponent-- > 0) {
            if (exp_sign > 0)
                val *= 10.0;
            else
                val /= 10.0;
        }
    }
    return val;
```

```text
atof_exp("123.45e-6") = 0.00012345
atof_exp("-2.5E3")    = -2500
atof_exp("3.14")      = 3.14
```

- 앞부분(공백·부호·정수부·소수부)은 그대로 두고 **뒤에 지수부 처리만 덧붙이면** 됩니다. 기존 코드를 건드리지 않는 것이 좋은 확장 방식입니다.
- `e`나 `E`가 없으면 `if`가 통째로 건너뛰어지므로 기존 동작이 그대로 유지됩니다.
- 지수만큼 10을 곱하거나 나눕니다. `while (exponent-- > 0)` 는 후위 감소를 이용한 관용구입니다(3강 8절).

---

### 문제 3. 계산기에 `%`와 음수 넣기

계산기가 나머지 연산(`%`)과 **음수 입력**을 처리하도록 확장하십시오.

**정답 및 해설**

`main.c`에 `case`를 추가합니다.

```c
        case '%':                       /* 추가: 나머지 (실수이므로 fmod) */
            op2 = pop();
            if (op2 != 0.0)
                push(fmod(pop(), op2));
            else
                printf("오류: 0 으로 나눌 수 없습니다\n");
            break;
```

`getop.c`는 **`-`가 빼기인지 음수 부호인지** 구분해야 합니다.

```c
/* getop: 음수도 처리하는 판 — 부호 뒤에 숫자가 오면 음수로 본다 */
int getop(char s[])
{
    int i, c, next;

    while ((s[0] = c = getch()) == ' ' || c == '\t')
        ;
    s[1] = '\0';
    i = 0;

    if (c == '-') {                  /* 빼기인지 음수 부호인지 구분한다 */
        next = getch();
        if (!isdigit(next) && next != '.') {
            ungetch(next);
            return c;                /* 뒤에 숫자가 없으면 빼기 연산자 */
        }
        s[++i] = c = next;           /* 음수의 시작 */
    }

    if (!isdigit(c) && c != '.')
        return c;
    ...
}
```

- **한 글자를 미리 읽어 보고 판단**하는 방식입니다. 아니면 `ungetch`로 되돌립니다. 4.6절에서 만든 되돌리기 기능이 여기서 그대로 쓰였습니다.
- `%`는 실수에 쓸 수 없으므로(3강 5절) `<math.h>`의 `fmod`를 사용하고, 컴파일할 때 `-lm`이 필요할 수 있습니다.
- 이 확장은 **`main.c`와 `getop.c`만** 고치면 되고 `stack.c`·`getch.c`는 손대지 않아도 됩니다. 파일을 잘 나누어 둔 덕분입니다.

---

### 문제 4. 계산기에 스택 명령 추가

맨 위 값 보기(`p`), 복제(`d`), 두 값 교환(`s`), 비우기(`c`) 명령을 추가하십시오.

**정답 및 해설**

`stack.c`에 함수를 넷 추가하고 헤더에 원형을 넣습니다.

```c
/* peek: 맨 위 값을 꺼내지 않고 확인한다 */
double peek(void)
{
    if (sp > 0)
        return val[sp - 1];
    printf("오류: 스택이 비어 있습니다\n");
    return 0.0;
}

/* duplicate: 맨 위 값을 하나 더 쌓는다 */
void duplicate(void)
{
    double top = peek();

    push(top);
}

/* swap_top: 맨 위 두 값을 맞바꾼다 */
void swap_top(void)
{
    double a, b;

    if (sp < 2) {
        printf("오류: 값이 두 개 이상 있어야 합니다\n");
        return;
    }
    a = pop();
    b = pop();
    push(a);
    push(b);
}

/* clear_stack: 스택을 비운다 */
void clear_stack(void)
{
    sp = 0;
}
```

- **네 함수 모두 `stack.c` 안에 있어야 합니다.** `sp`와 `val`이 `static`이라 다른 파일에서는 볼 수 없기 때문입니다. 이것이 바로 7.1절에서 말한 "감추기"의 효과이며, 스택을 다루는 코드가 한곳에 모이게 만듭니다.
- `swap_top`은 두 번 꺼내 **순서를 바꾸어 다시 넣습니다.** `push(a); push(b);` 의 순서를 반대로 하면 아무 일도 일어나지 않으므로 주의하십시오.
- `clear_stack`은 값을 지우지 않고 `sp`만 0으로 만듭니다. 남은 값은 다음에 덮어써지므로 문제가 없습니다.

---

### 문제 5. `ungetch` 없이 `getop` 만들기

되돌린 글자를 배열이 아니라 **함수 안의 `static` 변수 하나**로 기억하도록 `getop`을 고치십시오.

**정답 및 해설**

```c
/* getop_static: 되돌린 글자를 static 변수 하나로 기억한다 (ungetch 불필요) */
int getop_static(char s[])
{
    static int buf = 0;      /* 되돌린 글자. 0 이면 없음 */
    int i, c;

    if (buf != 0) {          /* 지난번에 남겨 둔 글자가 있으면 먼저 쓴다 */
        c = buf;
        buf = 0;
    } else {
        c = getchar();
    }

    while (c == ' ' || c == '\t')
        c = getchar();

    s[0] = (char) c;
    s[1] = '\0';
    if (!isdigit(c) && c != '.')
        return c;

    i = 0;
    if (isdigit(c))
        while (isdigit(s[++i] = c = getchar()))
            ;
    if (c == '.')
        while (isdigit(s[++i] = c = getchar()))
            ;
    s[i] = '\0';
    if (c != EOF)
        buf = c;             /* 한 글자 더 읽었으므로 기억해 둔다 */
    return NUMBER;
}
```

- **함수 안 `static` 변수가 "호출 사이에 값을 기억한다"는 성질**을 그대로 활용한 예입니다(7.2절).
- 파일 두 개(`getch.c`)와 함수 두 개가 사라져 프로그램이 단순해졌습니다.
- 다만 **되돌릴 수 있는 글자가 하나뿐**입니다. 여러 글자를 되돌려야 하면 원래대로 배열이 필요합니다. 필요한 만큼만 만드는 것이 좋은 설계입니다.
- `buf = 0`을 "없음"으로 쓴 것은 값이 0인 문자(`'\0'`)가 입력에 나오지 않는다는 가정입니다. 엄밀하게 하려면 별도의 표시 변수를 두어야 합니다.

---

### 문제 6·7. 재귀 `itoa`와 재귀 `reverse`

`printd`의 방식으로 **재귀** `itoa`를, 그리고 **재귀** `reverse`를 작성하십시오.

**정답 및 해설**

```c
/* itoa_rec: 재귀로 정수를 문자열로 바꾼다 */
static int itoa_rec_helper(int n, char s[], int i)
{
    if (n / 10)
        i = itoa_rec_helper(n / 10, s, i);
    s[i++] = n % 10 + '0';
    return i;
}

void itoa_rec(int n, char s[])
{
    int i = 0;

    if (n < 0) {
        s[i++] = '-';
        n = -n;
    }
    i = itoa_rec_helper(n, s, i);
    s[i] = '\0';
}

/* reverse_rec: 재귀로 문자열을 뒤집는다 */
static void reverse_helper(char s[], int i, int j)
{
    char temp;

    if (i >= j)
        return;
    temp = s[i];
    s[i] = s[j];
    s[j] = temp;
    reverse_helper(s, i + 1, j - 1);
}

void reverse_rec(char s[])
{
    reverse_helper(s, 0, (int) strlen(s) - 1);
}
```

```text
reverse_rec = gnimmargorp
itoa_rec(12345) = 12345
itoa_rec(-987)  = -987
itoa_rec(0)     = 0
```

- 두 함수 모두 **도우미 함수**를 따로 두었습니다. 재귀에는 "지금 어디까지 했는가"를 담을 인자가 필요한데, 그것을 사용자에게 요구할 수는 없기 때문입니다. 도우미를 `static`으로 감추면 밖에서는 깔끔한 함수 하나만 보입니다.
- `itoa_rec`는 4강처럼 **뒤집는 과정이 필요 없습니다.** 재귀가 자릿수를 자연스럽게 앞에서부터 채워 주기 때문입니다.
- `reverse_helper`는 재귀 호출이 **함수의 마지막 문장**입니다. 이런 형태를 꼬리 재귀라 하며, 반복문으로 바꾸기가 쉽습니다.

---

### 문제 8. `SWAP` 매크로

자료형 `t`의 두 변수를 맞바꾸는 매크로 `SWAP(t, x, y)`를 정의하십시오.

**정답 및 해설**

```c
/* SWAP: 자료형 t 의 두 변수를 맞바꾼다 */
#define SWAP(t, x, y)  do { t _tmp = (x); (x) = (y); (y) = _tmp; } while (0)
```

```text
a = 2, b = 1
p = 9.5, q = 1.5
c1 = Z, c2 = A
```

- **자료형을 인자로 받는 것**이 요령입니다. 임시 변수를 만들어야 하는데 그 자료형을 매크로가 알 수 없기 때문입니다.
- **블록 `{ }` 안에 임시 변수를 선언**했습니다. 9절의 블록 구조 덕분에 매크로 밖의 이름과 충돌하지 않습니다.
- **`do { ... } while (0)` 으로 감싼 이유**가 중요합니다. 그냥 `{ ... }`로 두면 다음 코드에서 문제가 생깁니다.

```c
if (a > b)
    SWAP(int, a, b);   /* 세미콜론이 빈 문장이 되어 else 와 짝이 어긋난다 */
else
    ...
```

`do { } while (0)` 은 **세미콜론을 붙여야 완성되는 하나의 문장**이 되므로 이 문제가 사라집니다. 여러 문장을 담은 매크로에서 널리 쓰이는 관용구입니다.

---

### 문제 9. 4강 함수들을 모듈로 분리하기

4강에서 만든 `binsearch`·`my_itoa`·`reverse`·`trim`을 **재사용 가능한 모듈**로 분리하십시오. `mylib.h`와 `mylib.c`를 만들고, 시험용 `mylib_main.c`에서 사용합니다.

**정답 및 해설**

`mylib.h` — 공개할 것만 적습니다.

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

`mylib_main.c` — 사용하는 쪽입니다.

```c
#include <stdio.h>
#include "mylib.h"

int main(void)
{
    int v[] = {2, 5, 8, 12, 16, 23, 38, 56, 72, 91};
    int n = (int) (sizeof v / sizeof v[0]);
    char s[64] = "  hello   ";
    char num[32];
    int len;

    printf("binsearch(23) = %d\n", binsearch(23, v, n));
    printf("binsearch(40) = %d\n", binsearch(40, v, n));

    my_itoa(-4567, num);
    printf("my_itoa(-4567) = \"%s\"\n", num);

    reverse(num);
    printf("reverse 적용   = \"%s\"\n", num);

    printf("trim 전 = \"%s\"\n", s);
    len = trim(s);
    printf("trim 후 = \"%s\" (길이 %d)\n", s, len);
    return 0;
}
```

```powershell
gcc -Wall -Wextra -std=c17 -c mylib.c -o mylib.o
```

```powershell
gcc -Wall -Wextra -std=c17 mylib_main.c mylib.o -o mylib_main.exe
```

```text
binsearch(23) = 5
binsearch(40) = -1
my_itoa(-4567) = "-4567"
reverse 적용   = "7654-"
trim 전 = "  hello   "
trim 후 = "  hello" (길이 7)
```

- **헤더에는 원형과 주석만** 두고 구현은 `.c`에 넣었습니다. 사용하는 쪽은 헤더만 읽으면 됩니다.
- `my_itoa`가 내부에서 `reverse`를 호출합니다. 같은 파일 안이므로 그대로 쓸 수 있으며, `reverse`도 밖에서 쓸모가 있으므로 함께 공개했습니다.
- `trim`은 **앞쪽 공백은 건드리지 않습니다.** 그래서 `"  hello"`가 남았습니다. 함수의 주석에 "끝의"라고 분명히 적어 둔 이유입니다.
- 이 `mylib.o`는 **11강에서 정적 라이브러리(`libmylib.a`)로 묶게 됩니다.**

---

### 문제 10. `static`으로 호출 횟수 세기

함수가 몇 번 호출되었는지 스스로 세도록 만들고, **전역 변수를 쓰는 것보다 나은 이유**를 설명하십시오.

**정답 및 해설**

```c
#include <stdio.h>

/* 이 파일 안에서만 보이는 함수와 변수 */
static int total_calls = 0;      /* 다른 파일에서는 이 이름을 볼 수 없다 */

/* 호출될 때마다 자기 호출 횟수를 세는 함수 */
int work(int x)
{
    static int calls = 0;        /* 이 함수만의 개인 기억 장소 */

    calls++;
    total_calls++;
    printf("work(%d) — 이 함수 호출 %d번째, 파일 전체 호출 %d번째\n",
           x, calls, total_calls);
    return x * 2;
}

static void helper(void)         /* static 함수: 이 파일 밖에서 부를 수 없다 */
{
    total_calls++;
    printf("helper() — 파일 전체 호출 %d번째\n", total_calls);
}

int main(void)
{
    work(1);
    work(2);
    helper();
    work(3);
    printf("합계 호출 횟수 = %d\n", total_calls);
    return 0;
}
```

```text
work(1) — 이 함수 호출 1번째, 파일 전체 호출 1번째
work(2) — 이 함수 호출 2번째, 파일 전체 호출 2번째
helper() — 파일 전체 호출 3번째
work(3) — 이 함수 호출 3번째, 파일 전체 호출 4번째
합계 호출 횟수 = 4
```

**전역 변수보다 나은 이유**

| 방식 | 문제점 |
|---|---|
| 그냥 전역 변수 `int calls;` | 어느 파일에서든 값을 바꿀 수 있어 **누가 망쳤는지 찾기 어렵다.** 다른 파일에 같은 이름이 있으면 충돌한다 |
| `static` 전역 변수 | 그 파일 안에서만 보이므로 **의심할 범위가 파일 하나로 줄어든다** |
| **함수 안 `static` 변수** | **그 함수 말고는 아무도 건드릴 수 없다.** 값을 잘못 바꾼 범인은 오직 그 함수뿐이다 |

세 번째가 가장 좁은 범위입니다. **필요한 만큼만 공개하고 나머지는 감춘다** — 이것이 이번 강의 전체를 관통하는 원칙이며, 2강 11.4절에서 외부 변수를 남용하지 말라고 한 이야기의 실천법입니다.

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 계산기 4개 파일(`calc.h`, `main.c`, `stack.c`, `getop.c`, `getch.c`)과 빌드·실행 화면 |
| 2 | 예제 파일 — `find.c`, `atof_demo.c`, `static_demo.c`, `scope_demo.c`, `init_demo.c`, `printd.c`, `qsort_demo.c`, `macro_demo.c`, `cond_include.c` |
| 3 | 실습문제 정답 파일 전체 |
| 4 | 실행 결과 화면 — 문제 3·4·9·10 |
| 5 | 짧은 서술 ① `stack.c`의 `sp`·`val`에 `static`을 붙이면 무엇이 좋아지는지 설명 |
| 6 | 짧은 서술 ② `push(pop() - pop());` 이 왜 잘못된 코드인지 설명 |
| 7 | 짧은 서술 ③ 재귀 호출에서 지역 변수가 서로 섞이지 않는 이유를 스택 프레임으로 설명 |

---

## 정리

| 구분 | 내용 |
|---|---|
| 함수 | 큰일을 작게 나누는 장치. `return`의 값은 반환 자료형으로 변환된다 |
| 원형 선언 | 반드시 필요하다. 없으면 **파일이 나뉘었을 때 오류가 발견되지 않는다** |
| 분할 컴파일 | `gcc -c`로 `.o`를 만들고 마지막에 링크. 고친 파일만 다시 컴파일 |
| 헤더 파일 | 공유할 선언을 모아 두는 곳. **`#ifndef` 중복 포함 방지 필수**, 내 헤더는 `""`, 시스템 헤더는 `<>` |
| 선언과 정의 | 정의는 **한 번만**, 다른 파일에서는 `extern` 선언 |
| `static`(파일 수준) | 그 파일 밖에서 **보이지 않게 감춘다** |
| `static`(함수 안) | 호출이 끝나도 **값을 유지**한다. 초기값은 0 |
| 블록 구조 | 안쪽 선언이 바깥 이름을 **가린다**. 되도록 피할 것 |
| 초기화 | 외부·`static`은 0, 지역은 쓰레기 값. 배열은 모자라면 0으로 채움 |
| 재귀 | 호출마다 **스택 프레임**이 새로 쌓인다. 종료 조건 필수 |
| 스택 프레임 | 지역 변수 + 매개변수 + **돌아갈 주소** → 버퍼 오버플로가 위험한 이유 |
| 전처리기 | `#include`, 매크로(괄호·부작용 주의), 조건부 컴파일, `-D` 옵션 |

프로그램을 여러 파일로 나누고 필요한 것만 공개하는 방법을 익혔습니다. 그런데 아직 **함수가 호출한 쪽의 변수를 직접 바꾸는 방법**을 모릅니다. 2강에서 "값에 의한 호출뿐"이라고 배웠고, 배열만 예외처럼 보였습니다.

**다음 6강에서는 포인터(K&R 제5장)** 를 다룹니다. 변수의 **주소**를 담는 변수가 포인터이며, 이것을 알면 지금까지 미뤄 둔 물음들이 한꺼번에 풀립니다. `scanf`에 왜 `&`를 붙이는지, 배열 이름이 무엇인지, 함수가 어떻게 호출한 쪽의 값을 바꿀 수 있는지, 그리고 문자열이 실제로 어떻게 다루어지는지를 모두 이해하게 됩니다.

---

## 부록 A. 여러 파일 빌드 명령 요약

| 하려는 일 | 명령 |
|---|---|
| 한 파일만 목적 파일로 | `gcc -Wall -Wextra -std=c17 -c stack.c -o stack.o` |
| 목적 파일들을 링크 | `gcc main.o stack.o getop.o getch.o -o calc.exe` |
| 소스에서 한 번에 | `gcc -Wall -Wextra -std=c17 main.c stack.c getop.c getch.c -o calc.exe` |
| 헤더 경로 지정 | `gcc -Iinclude -c main.c -o main.o` |
| 매크로 정의하며 컴파일 | `gcc -DDEBUG=1 -c main.c -o main.o` |
| 목적 파일 정리 | `Remove-Item *.o` |

> **한 파일만 고쳤을 때**는 그 파일의 `.o`만 다시 만들고 링크만 하면 됩니다.
> ```powershell
> gcc -Wall -Wextra -std=c17 -c getop.c -o getop.o
> ```
> ```powershell
> gcc main.o stack.o getop.o getch.o -o calc.exe
> ```
{: .prompt-tip }

## 부록 B. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `find.c` | 패턴이 든 줄 찾기 | 1.2 |
| `atof_demo.c` | `double`을 돌려주는 함수 | 2.1 |
| `calc.h`·`main.c`·`stack.c`·`getop.c`·`getch.c` | 역폴란드 계산기 | 4 · 6 |
| `static_demo.c` | 함수 안 `static` 변수 | 7.2 |
| `scope_demo.c` | 블록 구조와 이름 가리기 | 9 |
| `init_demo.c` | 초기화 규칙 | 10.2 |
| `printd.c` | 재귀 출력 | 11.1 |
| `qsort_demo.c` | 퀵 정렬 | 11.2 |
| `frame_demo.c` | 스택 프레임 확인 | 11.3 |
| `macro_demo.c` | 매크로와 그 함정 | 12.2 |
| `cond_include.c` | 조건부 컴파일 | 12.3 |
| `mylib.h`·`mylib.c` | 재사용 모듈(실습문제 9) | 실습 |
