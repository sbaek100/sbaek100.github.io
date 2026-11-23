---
title: C 프로그래밍 기초 7강 - 포인터와 배열 (2)
date: 2026-10-05 09:00:00 +0900
categories:
  - 0.기초강의
  - C 프로그래밍 기초
tags:
  - C언어
  - KnR
  - 포인터배열
  - 다차원배열
  - 명령행인자
  - argv
  - 함수포인터
  - 선언읽기
  - 버퍼오버플로
  - 보안
pin:
mermaid: false
---

> **학습 목표**
> 1. 포인터 배열을 선언·초기화하고, 문자열 목록을 다룰 수 있다.
> 2. 포인터 배열을 이용해 **줄들을 정렬**할 수 있고, 그 방식이 왜 효율적인지 설명할 수 있다.
> 3. 2차원 배열을 선언·초기화하고 함수에 넘길 수 있다.
> 4. **포인터 배열과 2차원 배열의 차이**를 설명하고 상황에 맞게 고를 수 있다.
> 5. `argc`·`argv`로 명령행 인자를 받아 처리할 수 있다.
> 6. `-x`·`-n` 같은 **옵션을 해석하는** 프로그램을 작성할 수 있다.
> 7. 함수 포인터를 선언·전달·호출할 수 있고, 이를 이용해 **동작을 바꿔 끼우는** 함수를 작성할 수 있다.
> 8. 복잡한 선언을 안에서 바깥으로 읽는 규칙을 적용할 수 있다.
> 9. **버퍼 오버플로가 인접한 값을 어떻게 망가뜨리는지** 직접 확인하고 설명할 수 있다.
> 10. 크기를 함께 넘기는 안전한 함수로 같은 코드를 고쳐 쓸 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

6강에서 포인터의 기본을 익혔습니다. 이제 그것을 **실제로 쓸모 있는 도구**로 만듭니다.

| 하고 싶은 일 | 필요한 것 |
|---|---|
| 길이가 제각각인 문자열 여러 개를 정렬 | **포인터 배열**(제1절) |
| 표(달력, 성적표) 다루기 | **다차원 배열**(제2절) |
| 프로그램에 값을 넘겨 실행 | **명령행 인자**(제4절) |
| 정렬 기준을 바꿔 끼우기 | **함수 포인터**(제5절) |

그리고 이번 강의 마지막에서, **2강부터 다섯 번에 걸쳐 예고해 온 버퍼 오버플로를 직접 재현**합니다. 지금까지는 "위험하다"고만 했지만, 이제 포인터와 명령행 인자를 알았으므로 **실제로 값이 어떻게 망가지는지 눈으로 확인**할 수 있습니다.

> **참고 문헌**
> 이번 강의는 다음 책의 **제5장 5.6 ~ 5.12**를 바탕으로 재구성하고, 마지막에 보안 실습 한 절을 더한 것입니다.
> Brian W. Kernighan, Dennis M. Ritchie, *The C Programming Language*, 2nd Edition, Prentice Hall, 1988.
{: .prompt-info }

| K&R | 원서 절 제목 | 이번 강의 |
|---|---|---|
| 5.6 | Pointer Arrays; Pointers to Pointers | 제1절 |
| 5.7 | Multi-dimensional Arrays | 제2절 |
| 5.8 | Initialization of Pointer Arrays | 제3절 |
| 5.9 | Pointers vs. Multi-dimensional Arrays | 제3절 |
| 5.10 | Command-line Arguments | 제4절 |
| 5.11 | Pointers to Functions | 제5절 |
| 5.12 | Complicated Declarations | 제6절 |
| — | (우리 추가) 버퍼 오버플로 실증 | 제7절 |

이 강의는 **3회차 분량**(모두 합쳐 약 415분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제4절 | 포인터 배열·다차원 배열·`argv` | 175분 |
| **2회차** | 제5절 ~ 제8절 | 함수 포인터와 버퍼 오버플로 | 130분 |
| **3회차** | 실습문제 | 스스로 해 보기 | 110분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 포인터 배열과 줄 정렬 | 55분 |
| 제2절 | 다차원 배열 | 40분 |
| 제3절 | 포인터 배열 vs 2차원 배열 | 30분 |
| 제4절 | 명령행 인자 | 50분 |
| 제5절 | 함수 포인터 | 45분 |
| 제6절 | 복잡한 선언 읽기 | 25분 |
| 제7절 | **버퍼 오버플로 실증** | 45분 |
| 제8절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 폴더**는 `C:\c-study\lab07`을 사용합니다.

```powershell
mkdir C:\c-study\lab07
```

---

## 제1절. 포인터 배열과 줄 정렬

### 1.1 길이가 제각각인 자료를 정렬하려면

4강에서 정수 배열을 정렬했고, 5강에서 퀵 정렬을 만들었습니다. 그런데 **텍스트 줄들을 정렬**하려면 새로운 문제가 생깁니다.

- 줄마다 **길이가 다릅니다.**
- 정수처럼 **한 번의 대입으로 맞바꿀 수 없습니다.** 글자를 하나씩 옮겨야 합니다.

긴 줄과 짧은 줄을 통째로 옮기는 것은 느리고 번거롭습니다. **해결책은 줄 자체가 아니라 줄을 가리키는 포인터를 맞바꾸는 것**입니다.

| 정렬 전 | 정렬 후 |
|---|---|
| `lineptr[0]` → `"defghi"` | `lineptr[0]` → `"abc"` |
| `lineptr[1]` → `"jklmnopqrst"` | `lineptr[1]` → `"defghi"` |
| `lineptr[2]` → `"abc"` | `lineptr[2]` → `"jklmnopqrst"` |

**글자들은 제자리에 그대로 있고, 화살표만 바뀝니다.** 아무리 긴 줄이라도 옮기는 비용은 포인터 하나(8바이트)로 같습니다.

### 1.2 포인터 배열의 선언

```c
char *lineptr[MAXLINES];
```

이 선언을 읽는 방법은 6강에서 배운 규칙 그대로입니다. **`lineptr`은 배열이고, 그 원소 하나하나가 `char`를 가리키는 포인터**입니다.

| 표기 | 뜻 |
|---|---|
| `lineptr` | 포인터들의 배열(그 자체는 첫 원소의 주소) |
| `lineptr[i]` | i번째 줄을 가리키는 **포인터** |
| `*lineptr[i]` | i번째 줄의 **첫 글자** |

### 1.3 프로그램

`sortlines.c`입니다. 하는 일은 셋으로 나뉩니다.

> 모든 줄을 읽는다 → 정렬한다 → 순서대로 출력한다

```c
#include <stdio.h>
#include <string.h>

#define MAXLINES 100       /* 정렬할 수 있는 최대 줄 수 */
#define MAXLEN   1000      /* 한 줄의 최대 길이 */
#define ALLOCSIZE 10000    /* 줄들을 담아 두는 공간 */

char *lineptr[MAXLINES];   /* 각 줄을 가리키는 포인터의 배열 */

static char allocbuf[ALLOCSIZE];
static char *allocp = allocbuf;

char *alloc(int n)
{
    if (allocbuf + ALLOCSIZE - allocp >= n) {
        allocp += n;
        return allocp - n;
    }
    return NULL;
}

/* get_line: 한 줄을 읽어 s 에 담고 길이를 돌려준다 (2강에서 만든 것) */
int get_line(char *s, int lim)
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

/* readlines: 입력의 줄들을 읽어 lineptr 에 채운다. 줄 수를 돌려준다 */
int readlines(char *lineptr[], int maxlines)
{
    int len, nlines;
    char *p, line[MAXLEN];

    nlines = 0;
    while ((len = get_line(line, MAXLEN)) > 0) {
        if (nlines >= maxlines || (p = alloc(len)) == NULL)
            return -1;                  /* 자리가 모자란다 */
        line[len - 1] = '\0';           /* 끝의 개행을 지운다 */
        strcpy(p, line);
        lineptr[nlines++] = p;
    }
    return nlines;
}

/* writelines: 줄들을 순서대로 출력한다 */
void writelines(char *lineptr[], int nlines)
{
    int i;

    for (i = 0; i < nlines; i++)
        printf("%s\n", lineptr[i]);
}

void swap(char *v[], int i, int j)
{
    char *temp;

    temp = v[i];
    v[i] = v[j];
    v[j] = temp;
}

/* my_qsort: v[left]...v[right] 를 사전 순으로 정렬한다 */
void my_qsort(char *v[], int left, int right)
{
    int i, last;

    if (left >= right)
        return;
    swap(v, left, (left + right) / 2);
    last = left;
    for (i = left + 1; i <= right; i++)
        if (strcmp(v[i], v[left]) < 0)
            swap(v, ++last, i);
    swap(v, left, last);
    my_qsort(v, left, last - 1);
    my_qsort(v, last + 1, right);
}

int main(void)
{
    int nlines;

    if ((nlines = readlines(lineptr, MAXLINES)) >= 0) {
        my_qsort(lineptr, 0, nlines - 1);
        writelines(lineptr, nlines);
        return 0;
    }
    printf("오류: 입력이 너무 많아 정렬할 수 없습니다\n");
    return 1;
}
```

다음 내용의 `lines.txt`로 실행합니다.

```text
banana
apple
Cherry
date
```

```powershell
.\sortlines.exe < lines.txt
```

```text
Cherry
apple
banana
date
```

> **왜 `Cherry`가 맨 앞에 왔을까요?**
> `strcmp`는 **문자의 값**을 비교하는데, ASCII에서 대문자(`A`=65)가 소문자(`a`=97)보다 작기 때문입니다(3강 3.4절). 사람이 기대하는 "사전 순"과 다르므로, 대소문자를 무시하고 정렬하려면 비교 함수를 바꾸어야 합니다. 실습문제 5에서 직접 만들어 봅니다.
{: .prompt-info }

### 1.4 5강의 퀵 정렬과 무엇이 달라졌는가

| 항목 | 5강(정수 정렬) | 이번(줄 정렬) |
|---|---|---|
| 배열 자료형 | `int v[]` | `char *v[]` |
| 비교 | `v[i] < v[left]` | `strcmp(v[i], v[left]) < 0` |
| 맞바꾸기 | `int temp;` | `char *temp;` |
| 알고리즘 | **동일** | **동일** |

**알고리즘은 한 줄도 바꾸지 않았습니다.** 자료형과 비교 방법만 바뀌었을 뿐입니다. 이 점에 착안하면 "비교 방법 자체를 밖에서 정해 주는" 정렬을 만들 수 있는데, 그것이 제5절의 함수 포인터입니다.

---

## 제2절. 다차원 배열

### 2.1 표를 다루기

날짜 변환 프로그램을 만들어 봅니다. 3월 1일이 그해의 몇 번째 날인지, 반대로 60번째 날이 몇 월 며칠인지 구하는 것입니다.

두 함수 모두 **각 달의 날짜 수**가 필요하고, 그것은 **평년과 윤년이 다릅니다.** 그러므로 표가 두 줄 필요합니다.

```c
static char daytab[2][13] = {
    {0, 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31},
    {0, 31, 29, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31}
};
```

- **`[2][13]`** — 2줄 13칸입니다. 첫 줄은 평년, 둘째 줄은 윤년입니다.
- **맨 앞의 0** — 월 번호를 1~12로 그대로 쓰기 위해 0번 칸을 버립니다. 첨자를 조정하는 것보다 훨씬 읽기 쉽습니다.
- **`char`를 쓴 이유** — 31 이하의 작은 수만 담으므로 1바이트면 충분합니다. 3강에서 배운 "`char`는 작은 정수"의 정당한 활용입니다.

### 2.2 프로그램

`daytab.c`입니다.

```c
/* day_of_year: 월·일을 그해의 몇 번째 날인지로 바꾼다 */
int day_of_year(int year, int month, int day)
{
    int i, leap;

    leap = (year % 4 == 0 && year % 100 != 0) || year % 400 == 0;
    for (i = 1; i < month; i++)
        day += daytab[leap][i];
    return day;
}

/* month_day: 그해의 몇 번째 날을 월·일로 바꾼다 */
void month_day(int year, int yearday, int *pmonth, int *pday)
{
    int i, leap;

    leap = (year % 4 == 0 && year % 100 != 0) || year % 400 == 0;
    for (i = 1; yearday > daytab[leap][i]; i++)
        yearday -= daytab[leap][i];
    *pmonth = i;
    *pday = yearday;
}
```

```text
2026-03-01 은 그해의 60번째 날
2024-03-01 은 그해의 61번째 날 (윤년)
2026-12-31 은 그해의 365번째 날
2026년 60번째 날 = 3월 1일
2024년 60번째 날 = 2월 29일 (윤년)
```

여기에 배울 점이 셋 있습니다.

**① 논리식의 값을 첨자로 씁니다.**

```c
leap = (year % 4 == 0 && year % 100 != 0) || year % 400 == 0;
```

3강에서 배운 대로 **논리식의 값은 참이면 1, 거짓이면 0**입니다. 그래서 `daytab[leap][i]`가 곧 "평년 줄 또는 윤년 줄"을 고르는 셈이 됩니다.

**② `month_day`는 결과가 둘이므로 포인터를 씁니다.**

6강 3.4절에서 배운 방식 그대로입니다. 월과 일을 각각 `*pmonth`, `*pday`로 돌려줍니다.

**③ 첨자는 대괄호를 각각 씁니다.**

```c
daytab[i][j]        /* 올바름 */
daytab[i, j]        /* 잘못 — 다른 언어의 표기 */
```

C에서 2차원 배열은 사실 **"배열의 배열"** 입니다. `daytab`은 "13칸짜리 배열"을 2개 가진 배열이며, `daytab[1]`은 그 자체가 13칸짜리 배열입니다.

### 2.3 2차원 배열을 함수에 넘기기

배열을 넘기면 **첫 원소의 주소**만 전달된다는 것을 6강에서 배웠습니다. 2차원 배열에서 "첫 원소"는 **첫 번째 줄(13칸짜리 배열)** 입니다. 그러므로 함수는 "한 줄이 몇 칸인지"를 반드시 알아야 합니다.

```c
void f(int daytab[2][13]) { ... }    /* 줄 수는 적어도 되고 */
void f(int daytab[][13])  { ... }    /* 생략해도 되지만 */
void f(int (*daytab)[13]) { ... }    /* 칸 수는 반드시 필요하다 */
```

**첫 번째 첨자만 생략할 수 있고 나머지는 반드시 적어야 합니다.** 컴파일러가 `daytab[i][j]`의 위치를 계산하려면 한 줄의 길이를 알아야 하기 때문입니다.

세 번째 표기의 괄호에 주의하십시오.

| 표기 | 뜻 |
|---|---|
| `int (*p)[13]` | **13칸짜리 배열을 가리키는 포인터** |
| `int *p[13]` | **포인터 13개짜리 배열** |

`[]`가 `*`보다 우선순위가 높으므로(3강 12.1절) 괄호가 없으면 뜻이 완전히 달라집니다.

---

## 제3절. 포인터 배열 vs 2차원 배열

### 3.1 두 가지로 문자열 목록 만들기

같은 목록을 두 방법으로 만들 수 있습니다.

```c
const char *name[] = {"잘못된 달", "1월", "2월", "3월"};   /* 포인터 배열 */
char aname[][15]   = {"잘못된 달", "1월", "2월", "3월"};   /* 2차원 배열 */
```

`ptr_vs_2d.c`로 크기를 비교합니다.

```text
name  : 포인터 배열,   sizeof = 32 바이트
aname : 2차원 배열,    sizeof = 60 바이트
포인터 하나의 크기 = 8, 줄 하나의 칸 = 15
name[2]  = 2월
aname[2] = 2월
```

| 구분 | 포인터 배열 `name` | 2차원 배열 `aname` |
|---|---|---|
| 구조 | 주소 4개 + 글자들은 **각자 필요한 만큼** | 15칸짜리 방 4개 |
| 크기 | 32바이트 + 문자열들 | **60바이트 고정** |
| 줄 길이 | **제각각이어도 된다** | 모두 같아야 한다(가장 긴 것 기준) |
| 낭비 | 없음 | 짧은 줄은 **남는 칸이 버려진다** |
| 내용 변경 | 문자열 상수를 가리키면 **불가** | 가능 |

`"2월"`은 7바이트면 되는데 2차원 배열에서는 15칸을 차지합니다. 줄 길이 차이가 클수록 낭비가 커지므로, **문자열 목록에는 대체로 포인터 배열이 유리**합니다.

### 3.2 `month_name` — 포인터 배열의 전형적 활용

```c
/* month_name: n 번째 달의 이름을 돌려준다 */
const char *month_name(int n)
{
    static const char *name[] = {
        "잘못된 달",
        "1월", "2월", "3월",
        "4월", "5월", "6월",
        "7월", "8월", "9월",
        "10월", "11월", "12월"
    };

    return (n < 1 || n > 12) ? name[0] : name[n];
}
```

```text
1월 2월 3월 4월 5월 6월 7월 8월 9월 10월 11월 12월 
month_name(0)  = 잘못된 달
month_name(13) = 잘못된 달
```

- **`static`으로 선언한 이유**(5강 7.2절): 이 표는 호출될 때마다 새로 만들 필요가 없습니다. 한 번 만들어 두고 계속 씁니다.
- **크기를 적지 않았습니다.** 초기값의 개수를 세어 컴파일러가 13으로 정합니다(5강 10.2절).
- **0번 칸을 오류 처리용으로 쓴 것**이 요령입니다. 범위를 벗어나면 `name[0]`을 돌려주므로 호출한 쪽이 `NULL` 검사를 하지 않아도 됩니다.
- 반환 자료형이 **`const char *`** 입니다. 돌려준 문자열을 호출한 쪽이 고치지 못하게 막는 장치입니다(6강 6.2절).

---

## 제4절. 명령행 인자

### 4.1 프로그램에 값을 넘기는 표준 방법

지금까지 우리 프로그램은 값을 **입력으로만** 받았습니다. 그런데 실제 명령들은 실행할 때 값을 함께 받습니다.

```powershell
findstr hello file.txt
```

C 프로그램도 같은 일을 할 수 있습니다. `main`이 인자를 두 개 받도록 쓰면 됩니다.

```c
int main(int argc, char *argv[])
```

| 이름 | 뜻 |
|---|---|
| `argc` | **인자의 개수**(argument count). 프로그램 이름을 포함하므로 최소 1 |
| `argv` | **인자 문자열들을 가리키는 포인터 배열**(argument vector) |

| 요소 | 내용 |
|---|---|
| `argv[0]` | 프로그램 이름 |
| `argv[1]` | 첫 번째 인자 |
| `argv[argc-1]` | 마지막 인자 |
| `argv[argc]` | **언제나 널 포인터** |

`argvinfo.c`로 확인합니다.

```c
#include <stdio.h>

/* argc 와 argv 가 무엇인지 확인한다 */
int main(int argc, char *argv[])
{
    int i;

    printf("argc = %d\n", argc);
    for (i = 0; i < argc; i++)
        printf("argv[%d] = \"%s\"\n", i, argv[i]);
    printf("argv[%d] = %s   <- 마지막은 언제나 널 포인터\n",
           argc, (argv[argc] == NULL) ? "NULL" : "(NULL 아님)");
    return 0;
}
```

```powershell
.\argvinfo.exe hello world
```

```text
argc = 3
argv[0] = "C:\c-study\lab07\argvinfo.exe"
argv[1] = "hello"
argv[2] = "world"
argv[3] = NULL   <- 마지막은 언제나 널 포인터
```

> `argv[0]`에 들어오는 문자열은 환경에 따라 다릅니다. 전체 경로가 올 수도 있고 이름만 올 수도 있으므로, **`argv[0]`은 사용법 안내에 출력하는 용도 정도로만** 쓰는 것이 안전합니다.
{: .prompt-info }

### 4.2 `echo` — 두 가지 표기

인자들을 그대로 출력하는 프로그램입니다. 먼저 **배열처럼 다루는 판**입니다.

```c
/* 1차 판: 배열처럼 다루기 */
int main(int argc, char *argv[])
{
    int i;

    for (i = 1; i < argc; i++)
        printf("%s%s", argv[i], (i < argc - 1) ? " " : "");
    printf("\n");
    return 0;
}
```

다음은 **포인터를 직접 움직이는 판**입니다.

```c
/* 2차 판: 포인터를 움직이기 */
int main(int argc, char *argv[])
{
    while (--argc > 0)
        printf("%s%s", *++argv, (argc > 1) ? " " : "");
    printf("\n");
    return 0;
}
```

```powershell
.\echo2.exe hello, world
```

```text
hello, world
```

`*++argv`를 읽는 순서가 중요합니다.

| 단계 | 하는 일 |
|---|---|
| 1 | `++argv` — 포인터 배열을 가리키는 포인터를 **한 칸 옮긴다**(처음에는 `argv[0]` → `argv[1]`) |
| 2 | `*` — 그 자리에 있는 **문자열 포인터를 꺼낸다** |

`argv`는 **"문자를 가리키는 포인터"들의 배열을 가리키는 포인터**입니다. 이렇게 포인터가 두 겹으로 겹치는 구조가 C에서 문자열 목록을 다루는 표준 방식입니다.

### 4.3 옵션 해석하기

이제 5강의 패턴 찾기 프로그램을 개선합니다. 찾을 패턴을 코드에 박아 두는 대신 **명령행에서 받고**, 옵션도 처리합니다.

| 옵션 | 뜻 |
|---|---|
| `-n` | 줄 번호를 함께 출력 |
| `-x` | 패턴이 **없는** 줄을 출력 |

`findcmd.c`의 핵심 부분입니다.

```c
    while (--argc > 0 && (*++argv)[0] == '-')
        while ((c = *++argv[0]) != '\0')
            switch (c) {
            case 'x':
                except = 1;
                break;
            case 'n':
                number = 1;
                break;
            default:
                printf("find: 알 수 없는 옵션 %c\n", c);
                argc = 0;
                found = -1;
                break;
            }

    if (argc != 1) {
        printf("사용법: find -x -n 패턴\n");
        return 1;
    }

    while (get_line(line, MAXLINE) > 0) {
        lineno++;
        if ((strstr(line, *argv) != NULL) != except) {
            if (number)
                printf("%ld:", lineno);
            printf("%s", line);
            found++;
        }
    }
```

`poem.txt`로 실행해 봅니다.

```text
Ah Love! could you and I with Fate conspire
To grasp this sorry Scheme of Things entire,
Would not we shatter it to bits -- and then
Re-mould it nearer to the Heart's Desire!
```

```powershell
.\findcmd.exe ould < poem.txt
```

```text
Ah Love! could you and I with Fate conspire
Would not we shatter it to bits -- and then
Re-mould it nearer to the Heart's Desire!
```

```powershell
.\findcmd.exe -n ould < poem.txt
```

```text
1:Ah Love! could you and I with Fate conspire
3:Would not we shatter it to bits -- and then
4:Re-mould it nearer to the Heart's Desire!
```

```powershell
.\findcmd.exe -x ould < poem.txt
```

```text
To grasp this sorry Scheme of Things entire,
```

옵션은 **묶어서 쓸 수도** 있습니다. `.\findcmd.exe -nx ould` 는 `-n -x`와 같습니다. 안쪽 `while`이 한 인자 안의 글자들을 계속 훑기 때문입니다.

> **이 두 줄이 이번 절에서 가장 어려운 부분입니다.**
> ```c
> while (--argc > 0 && (*++argv)[0] == '-')
>     while ((c = *++argv[0]) != '\0')
> ```
> | 식 | 뜻 |
> |---|---|
> | `*++argv` | 다음 인자 문자열(포인터) |
> | `(*++argv)[0]` | 그 문자열의 **첫 글자** — `-`인지 검사 |
> | `argv[0]` | 지금 인자 문자열 |
> | `++argv[0]` | 그 **문자열 안에서** 한 글자 전진 |
> | `*++argv[0]` | 전진한 자리의 **글자** |
>
> 괄호가 없으면 `[]`가 `*`보다 우선하므로 뜻이 달라집니다. 실무에서는 이렇게까지 압축하기보다 **두세 단계로 풀어 쓰는 편**이 낫습니다. 다만 이런 코드를 **읽을 수는 있어야** 합니다.
{: .prompt-warning }

`strstr(s, t)`는 표준 라이브러리 함수로, `s` 안에서 `t`가 처음 나타나는 **위치의 포인터**를 돌려주고 없으면 `NULL`을 돌려줍니다. 5강에서 만든 `strindex`의 표준판이며 `<string.h>`에 있습니다.

---

> **▶ 여기서부터 2회차 — 함수 포인터와 버퍼 오버플로**
> 제5절 ~ 제8절, 약 130분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제5절. 함수 포인터

### 5.1 동작을 인자로 넘기기

1.4절에서 확인했듯이, 정렬 알고리즘은 **비교 방법과 무관**합니다. 그렇다면 **비교 방법 자체를 인자로 받으면** 하나의 정렬 함수로 여러 기준을 처리할 수 있습니다.

C에서 함수는 변수가 아니지만, **함수를 가리키는 포인터**는 만들 수 있습니다. 이것을 인자로 넘기면 됩니다.

### 5.2 선언 읽기

```c
int (*comp)(const char *, const char *);
```

| 부분 | 뜻 |
|---|---|
| `(*comp)` | `comp`는 **포인터**이다 |
| `(*comp)(...)` | 그 포인터는 **함수**를 가리킨다 |
| 인자 `(const char *, const char *)` | 그 함수는 문자열 포인터 두 개를 받는다 |
| 맨 앞 `int` | 그 함수는 `int`를 돌려준다 |

**괄호가 반드시 필요합니다.**

```c
int (*comp)(const char *, const char *);   /* 함수를 가리키는 포인터 */
int *comp(const char *, const char *);     /* int 포인터를 돌려주는 함수 — 전혀 다르다! */
```

### 5.3 정렬 기준을 바꿔 끼우기

`fnptr.c`입니다.

```c
/* numcmp: 두 문자열을 앞부분의 수치로 비교한다 */
int numcmp(const char *s1, const char *s2)
{
    double v1, v2;

    v1 = atof(s1);
    v2 = atof(s2);
    if (v1 < v2)
        return -1;
    if (v1 > v2)
        return 1;
    return 0;
}

/* my_sort: 비교 함수를 인자로 받아 정렬한다 */
void my_sort(const char *v[], int left, int right,
             int (*comp)(const char *, const char *))
{
    int i, last;

    if (left >= right)
        return;
    swap(v, left, (left + right) / 2);
    last = left;
    for (i = left + 1; i <= right; i++)
        if ((*comp)(v[i], v[left]) < 0)
            swap(v, ++last, i);
    swap(v, left, last);
    my_sort(v, left, last - 1, comp);
    my_sort(v, last + 1, right, comp);
}
```

```c
    my_sort(data1, 0, n - 1, strcmp_wrap);   /* 사전 순 */
    my_sort(data2, 0, n - 1, numcmp);        /* 수치 순 */
```

```text
원본        : 25 100 7 3 48
사전 순 정렬: 100 25 3 48 7
수치 순 정렬: 3 7 25 48 100
```

**같은 자료를 같은 정렬 함수로 처리했는데 결과가 다릅니다.** 비교 함수 하나만 바꿔 끼웠기 때문입니다.

사전 순에서 `100`이 맨 앞에 오는 이유는 첫 글자 `'1'`이 `'2'`, `'3'`, `'4'`, `'7'`보다 작기 때문입니다. **문자열로 비교하면 숫자의 크기와 무관**하다는 사실을 보여 주는 좋은 예입니다.

### 5.4 함수 이름은 그 자체가 주소입니다

```c
my_sort(data, 0, n - 1, numcmp);      /* & 를 붙이지 않는다 */
```

배열 이름이 첫 원소의 주소이듯, **함수 이름은 그 함수의 주소**입니다. 그래서 `&numcmp`라고 쓰지 않아도 됩니다(써도 같은 뜻입니다).

호출할 때도 두 가지 표기가 모두 됩니다.

```c
(*comp)(a, b);      /* 원래 형태 — 포인터를 풀어서 호출 */
comp(a, b);         /* 줄임 표기 — 요즘은 이 편을 더 많이 쓴다 */
```

### 5.5 어디에 쓰이는가

| 쓰임 | 예 |
|---|---|
| 정렬·검색 기준 지정 | 표준 라이브러리의 `qsort` |
| 이벤트 처리기 등록 | 버튼을 눌렀을 때 부를 함수 |
| 명령 표 만들기 | 명령 이름과 처리 함수를 짝지어 배열에 |
| 콜백 | 라이브러리가 사용자의 함수를 되부르는 구조 |

표준 라이브러리의 `qsort`가 대표적입니다.

```c
void qsort(void *base, size_t n, size_t size,
           int (*compar)(const void *, const void *));
```

자료형에 상관없이 무엇이든 정렬할 수 있는 이유가 바로 **비교 함수를 밖에서 받기** 때문입니다. `void *`는 "어떤 자료형이든 가리킬 수 있는 포인터"로, 자세한 내용은 이 과정의 범위를 넘어섭니다.

---

## 제6절. 복잡한 선언 읽기

### 6.1 안에서 바깥으로 읽습니다

C의 선언은 **"이 변수를 사용하는 모습"** 을 흉내 내어 적기 때문에, 복잡해지면 왼쪽에서 오른쪽으로 읽을 수 없습니다. 규칙은 다음과 같습니다.

> **이름에서 시작해서, 우선순위에 따라 안에서 바깥으로 읽는다.**
> `()`와 `[]`가 `*`보다 우선순위가 높다.

| 선언 | 읽는 순서 | 뜻 |
|---|---|---|
| `int *p;` | `p` → `*` → `int` | `p`는 **int를 가리키는 포인터** |
| `int p[5];` | `p` → `[5]` → `int` | `p`는 **int 5개짜리 배열** |
| `int *p[5];` | `p` → `[5]` → `*` → `int` | `p`는 **int 포인터 5개짜리 배열** |
| `int (*p)[5];` | `p` → `*` → `[5]` → `int` | `p`는 **int 5개짜리 배열을 가리키는 포인터** |
| `int f(void);` | `f` → `()` → `int` | `f`는 **int를 돌려주는 함수** |
| `int *f(void);` | `f` → `()` → `*` → `int` | `f`는 **int 포인터를 돌려주는 함수** |
| `int (*f)(void);` | `f` → `*` → `()` → `int` | `f`는 **int를 돌려주는 함수를 가리키는 포인터** |
| `char **argv;` | `argv` → `*` → `*` → `char` | `argv`는 **char 포인터를 가리키는 포인터** |

**괄호가 있으면 그 안을 먼저 읽는다** — 이 한 가지만 지키면 대부분 해결됩니다.

`decl_read.c`로 실제로 확인합니다.

```c
    int  x = 10;
    int *p = &x;              /* p: int 를 가리키는 포인터 */
    int  a[5] = {1, 2, 3, 4, 5};
    int *pa[3];               /* pa: int 포인터 3개짜리 배열 */
    int (*ap)[5] = &a;        /* ap: int 5개짜리 배열을 가리키는 포인터 */
    int (*pf)(void) = f1;     /* pf: int 를 돌려주는 함수를 가리키는 포인터 */
    int *(*pfp)(void) = f2;   /* pfp: int 포인터를 돌려주는 함수를 가리키는 포인터 */
```

```text
*p        = 10
*pa[0]    = 10
(*ap)[2]  = 3   <- 배열을 가리키는 포인터
pf()      = 1   <- 함수 포인터로 호출
*pfp()    = 42   <- 포인터를 돌려주는 함수
sizeof a  = 20, sizeof *ap = 20
```

`sizeof *ap`가 20인 것에 주목하십시오. `ap`가 **배열 전체를 가리키므로** `*ap`는 `int` 5개(20바이트)입니다.

> **실무 조언: 복잡한 선언은 만들지 마십시오.**
> 읽을 줄은 알아야 하지만, 직접 쓸 때는 **`typedef`로 이름을 붙여** 단계적으로 만드는 편이 훨씬 낫습니다. `typedef`는 8강에서 다룹니다.
> ```c
> typedef int (*CompareFn)(const char *, const char *);
> void my_sort(const char *v[], int left, int right, CompareFn comp);
> ```
{: .prompt-tip }

---

## 제7절. 버퍼 오버플로 실증

이제 2강 10.6절, 3강 2.2절, 5강 11.3절에서 예고해 온 것을 **직접 확인**합니다. 지금까지는 "배열 밖에 쓰면 옆 값이 망가진다"고 말로만 했습니다.

### 7.1 취약한 프로그램

`bof_demo.c`입니다. 이름을 받아 저장하는, 그럴듯해 보이는 프로그램입니다.

```c
#include <stdio.h>
#include <string.h>

struct account {
    char name[8];      /* 이름은 7글자까지 + '\0' */
    int  authorized;   /* 0 이면 권한 없음 */
};

/* 사용법: bof_demo.exe 이름 */
int main(int argc, char *argv[])
{
    struct account a;

    a.authorized = 0;

    if (argc != 2) {
        printf("사용법: %s 이름\n", argv[0]);
        return 1;
    }

    strcpy(a.name, argv[1]);      /* 위험: 길이를 검사하지 않는다 */

    printf("name       = %s\n", a.name);
    printf("authorized = %d (0x%08X)\n", a.authorized, (unsigned) a.authorized);
    if (a.authorized)
        printf(">> 권한 검사를 통과해 버렸습니다!\n");
    else
        printf(">> 권한 없음\n");
    return 0;
}
```

`authorized`는 **어디에서도 1로 바뀌지 않습니다.** 코드를 아무리 읽어도 권한이 생길 방법이 없어 보입니다.

### 7.2 정상 입력

```powershell
.\bof_demo.exe kim
```

```text
name       = kim
authorized = 0 (0x00000000)
>> 권한 없음
```

의도대로 동작합니다.

### 7.3 긴 입력

이번에는 `name`이 담을 수 있는 7글자를 훨씬 넘겨 봅니다.

```powershell
.\bof_demo.exe AAAAAAAAAAAA
```

```text
name       = AAAAAAAAAAAA
authorized = 1094795585 (0x41414141)
>> 권한 검사를 통과해 버렸습니다!
```

**권한 검사를 통과했습니다.** 그러나 `authorized`에 값을 넣는 코드는 프로그램 어디에도 없습니다.

### 7.4 무슨 일이 일어났는가

`struct account`의 메모리 배치를 보면 답이 나옵니다.

| 위치(구조체 시작부터) | 0~7 | 8~11 |
|---|---|---|
| 무엇 | `name[0]` ~ `name[7]` | `authorized`(4바이트) |

`"AAAAAAAAAAAA"`는 12글자에 끝 표시 `'\0'`까지 **13바이트**입니다. `strcpy`는 크기를 검사하지 않으므로 그대로 써 버립니다.

| 바이트 | 어디에 들어갔나 |
|---|---|
| 0~7번째 `A` | `name` 안 (정상) |
| 8~11번째 `A` | **`authorized` 자리를 덮어씀** |
| `'\0'` | 그 뒤까지 침범 |

`'A'`의 값은 65, 16진수로 `0x41`입니다. 네 개가 들어갔으니 `authorized`는 **`0x41414141`**, 10진수로 1094795585가 되었습니다. **0이 아니므로 참**이고, 권한 검사를 통과한 것입니다.

> **이것이 버퍼 오버플로입니다.**
> 공격자가 입력의 길이와 내용을 마음대로 정할 수 있다면, **프로그램의 논리를 건드리지 않고도 판단 결과를 바꿀 수 있습니다.**
>
> 여기서는 옆에 있던 변수 하나가 바뀌었을 뿐이지만, 5강 11.3절에서 본 것처럼 **스택 프레임에는 "돌아갈 주소"도 함께 놓여 있습니다.** 더 많이 넘치면 그 주소까지 덮어써서 프로그램의 흐름 자체를 빼앗을 수 있습니다. 그 방법과 대응은 정보보안 과목의 영역이며, 이 과정에서는 **"왜 위험한가"와 "어떻게 막는가"까지** 다룹니다.
{: .prompt-danger }

### 7.5 컴파일러는 언제 잡아 주는가

2강 실습에서 문자열 상수를 그대로 복사했을 때는 컴파일러가 경고했습니다.

```text
warning: '__builtin_memcpy' writing 13 bytes into a region of size 12 overflows the destination
```

그런데 **이번 프로그램은 경고가 나오지 않습니다.** 복사할 문자열이 `argv[1]`, 즉 **실행할 때 정해지는 값**이라 컴파일 시점에는 길이를 알 수 없기 때문입니다.

**실제 공격에 쓰이는 입력은 언제나 실행 중에 들어옵니다.** 그러므로 컴파일러 경고에만 의존해서는 안 되며, **코드 자체가 크기를 검사**해야 합니다.

### 7.6 고치기

`bof_safe.c`입니다.

```c
    /* 크기를 함께 넘겨 넘치지 않도록 한다 */
    snprintf(a.name, sizeof a.name, "%s", argv[1]);

    printf("name       = %s (%zu자)\n", a.name, strlen(a.name));
    printf("authorized = %d\n", a.authorized);
    if (strlen(argv[1]) >= sizeof a.name)
        printf(">> 입력이 잘렸습니다. 원본 길이 = %zu\n", strlen(argv[1]));
```

```powershell
.\bof_safe.exe AAAAAAAAAAAA
```

```text
name       = AAAAAAA (7자)
authorized = 0
>> 입력이 잘렸습니다. 원본 길이 = 12
```

같은 입력인데 이번에는 **7글자만 저장되고 `authorized`는 0 그대로**입니다.

| 원칙 | 설명 |
|---|---|
| **크기를 함께 넘긴다** | `sizeof a.name`으로 배열 자신의 크기를 사용한다. 숫자 8을 직접 적으면 배열 크기를 바꿀 때 어긋난다 |
| **잘렸는지 확인한다** | 조용히 자르는 것도 버그가 될 수 있다. 필요하면 오류로 알린다 |
| **경계를 지키는 함수를 쓴다** | `strcpy` 대신 `snprintf`, `gets` 대신 `fgets` |

이번 절에서 다룬 것은 **입구 하나**입니다. 어떤 함수들이 위험하고 무엇으로 바꾸어야 하는지는 **10강 「안전한 C 프로그래밍」** 에서 정리합니다.

---

## 제8절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| `int *p[5]`와 `int (*p)[5]`를 혼동 | `[]`가 `*`보다 우선 | 괄호를 확인하고 안에서 바깥으로 읽는다 |
| 2차원 배열을 함수에 넘길 때 오류 | 칸 수를 생략함 | `f(int a[][13])` — 두 번째 첨자는 필수 |
| 문자열 목록의 내용을 고치자 죽음 | 포인터 배열이 문자열 상수를 가리킴 | 고쳐야 하면 2차원 배열이나 복사본 사용 |
| `argv[1]`을 썼는데 죽음 | 인자가 없는데 접근 | `argc`를 먼저 검사 |
| 옵션 해석이 이상함 | `*++argv[0]`의 괄호·우선순위 | 단계를 나누어 쓴다 |
| 함수 포인터 선언이 컴파일 안 됨 | 괄호 누락 | `int (*f)(void)` |
| 정렬 결과가 숫자 크기와 다름 | 문자열로 비교 | 수치 비교 함수를 만들어 넘긴다 |
| 배열보다 긴 입력에 프로그램이 이상해짐 | 경계 검사 없음(버퍼 오버플로) | `snprintf`·`fgets`로 크기를 함께 넘긴다 |

---

> **▶ 여기서부터 3회차 — 스스로 해 보기**
> 실습문제, 약 110분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 실습문제

> **안내**
> 1. 모든 문제는 `C:\c-study\lab07`에서 수행합니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다. **먼저 스스로 작성한 뒤** 확인하시기 바랍니다.
> 3. 명령행 인자를 쓰는 문제는 PowerShell에서 `.\프로그램.exe 인자` 형태로 실행합니다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 | K&R |
|---|---|---|---|
| 문제 1 | 날짜 변환에 오류 검사 넣기 | 2.2 | 5-8 |
| 문제 2 | 날짜 변환의 포인터 판 | 2.2 | 5-9 |
| 문제 3 | `expr` — 명령행 계산기 | 4 | 5-10 |
| 문제 4 | 명령행 인자의 합·평균 | 4 | — |
| 문제 5 | 정렬에 `-r`·`-f` 옵션 넣기 | 1 · 4 · 5 | 5-14 · 5-15 |
| 문제 6 | 함수 포인터로 연산 고르기 | 5 | — |
| 문제 7 | 버퍼 오버플로 재현과 방어 | 7 | — |
| 문제 8 | 선언 읽기 | 6 | — |
| 문제 9 | 문자열 목록 뒤집기 | 1 | — |
| 문제 10 | 2차원 배열로 성적표 만들기 | 2 | — |

---

### 문제 1. 날짜 변환에 오류 검사 넣기

`day_of_year`와 `month_day`는 잘못된 입력을 검사하지 않습니다. 검사를 넣으십시오.

**정답 및 해설**

```c
/* day_of_year: 잘못된 입력이면 -1 을 돌려준다 */
int day_of_year(int year, int month, int day)
{
    int i, leap;

    leap = is_leap(year);
    if (month < 1 || month > 12)
        return -1;
    if (day < 1 || day > daytab[leap][month])
        return -1;

    for (i = 1; i < month; i++)
        day += daytab[leap][i];
    return day;
}

/* month_day: 잘못된 입력이면 *pmonth 를 -1 로 둔다 */
void month_day(int year, int yearday, int *pmonth, int *pday)
{
    int i, leap;

    leap = is_leap(year);
    if (yearday < 1 || yearday > (leap ? 366 : 365)) {
        *pmonth = -1;
        *pday = -1;
        return;
    }
    for (i = 1; yearday > daytab[leap][i]; i++)
        yearday -= daytab[leap][i];
    *pmonth = i;
    *pday = yearday;
}
```

```text
day_of_year(2026, 2, 29)  = -1   <- 평년에는 없는 날
day_of_year(2024, 2, 29)  = 60   <- 윤년에는 있다
day_of_year(2026, 13, 1)  = -1
month_day(2026, 400) -> 월 = -1, 일 = -1
```

- **날짜 검사는 그 달의 일수를 표에서 꺼내 비교**해야 합니다. `day > 31`로 검사하면 2월 30일을 걸러 내지 못합니다.
- 윤년 판정이 두 함수에 똑같이 나오므로 `is_leap` 함수로 뽑아냈습니다. 반복되는 것을 함수로 묶는 것이 5강의 교훈입니다.
- `month_day`는 값을 돌려주지 않으므로 **`*pmonth`에 −1을 넣어 실패를 알립니다.** 호출한 쪽이 반드시 검사해야 합니다.

---

### 문제 2. 날짜 변환의 포인터 판

`daytab[leap][i]` 대신 **포인터**를 사용하도록 고치십시오.

**정답 및 해설**

```c
/* day_of_year 의 포인터 판 */
int day_of_year(int year, int month, int day)
{
    char *p;
    int i, leap;

    leap = (year % 4 == 0 && year % 100 != 0) || year % 400 == 0;
    p = daytab[leap];          /* 그해에 해당하는 줄을 가리킨다 */
    for (i = 1; i < month; i++)
        day += *(p + i);
    return day;
}
```

```text
day_of_year(2026, 3, 1) = 60
month_day(2024, 60) -> 2월 29일
```

- **`daytab[leap]`은 그 자체가 배열**(13칸)이므로, 그 이름은 첫 원소의 주소입니다. 따라서 `char *p`에 그대로 담을 수 있습니다.
- `daytab[leap][i]`와 `*(p + i)`는 정확히 같은 뜻입니다(6강 4.2절).
- 2차원 배열의 한 줄을 포인터로 받는 이 방식은 **표의 한 행만 함수에 넘길 때** 특히 유용합니다.

---

### 문제 3. `expr` — 명령행 계산기

명령행에서 역폴란드 식을 받아 계산하는 프로그램을 작성하십시오.

**정답 및 해설**

```c
/* expr: 명령행에서 역폴란드 식을 계산한다
   예) expr 2 3 4 + '*'   ->  14 */
int main(int argc, char *argv[])
{
    double op2;
    int i;

    for (i = 1; i < argc; i++) {
        const char *s = argv[i];

        if (s[1] == '\0' && (*s == '+' || *s == '-' || *s == '*' || *s == '/')) {
            switch (*s) {
            case '+':
                push(pop() + pop());
                break;
            case '*':
                push(pop() * pop());
                break;
            case '-':
                op2 = pop();
                push(pop() - op2);
                break;
            case '/':
                op2 = pop();
                if (op2 != 0.0)
                    push(pop() / op2);
                else {
                    printf("오류: 0 으로 나눌 수 없습니다\n");
                    return 1;
                }
                break;
            }
        } else {
            push(atof(s));
        }
    }
    printf("%.8g\n", pop());
    return 0;
}
```

```powershell
.\expr.exe 2 3 4 + '*'
```

```text
14
```

- 5강의 계산기와 **알고리즘은 같고 입력원만 바뀌었습니다.** `getop`으로 읽던 것을 `argv`에서 꺼내 올 뿐입니다.
- **`*`는 PowerShell이 특별하게 해석**할 수 있으므로 작은따옴표로 감싸는 편이 안전합니다.
- 연산자인지 판단할 때 **`s[1] == '\0'`을 함께 검사**했습니다. 그러지 않으면 `-5` 같은 음수가 빼기 연산자로 오해됩니다.

---

### 문제 4. 명령행 인자의 합과 평균

명령행으로 받은 수들의 합과 평균을 구하십시오.

**정답 및 해설**

```c
int main(int argc, char *argv[])
{
    double sum = 0.0;
    int i, count = 0;

    if (argc < 2) {
        printf("사용법: %s 수1 수2 ...\n", argv[0]);
        return 1;
    }

    for (i = 1; i < argc; i++) {
        sum += atof(argv[i]);
        count++;
    }

    printf("개수 = %d, 합 = %g, 평균 = %g\n", count, sum, sum / count);
    return 0;
}
```

```powershell
.\argsum.exe 10 20 30.5
```

```text
개수 = 3, 합 = 60.5, 평균 = 20.1667
```

- **`argc < 2` 검사가 반드시 필요합니다.** 인자가 없는데 `argv[1]`을 읽으면 프로그램이 죽습니다.
- 인자는 언제나 **문자열**이므로 `atof`(또는 `atoi`)로 수로 바꾸어야 합니다. `argv[1] + argv[2]`처럼 쓰면 포인터끼리 더하는 셈이라 컴파일 오류입니다.
- `count`가 0이 될 수 없음을 위에서 보장했으므로 0으로 나눌 걱정이 없습니다.

---

### 문제 5. 정렬에 `-r`·`-f` 옵션 넣기

줄 정렬 프로그램에 **역순 정렬(`-r`)** 과 **대소문자 무시(`-f`)** 옵션을 추가하십시오. 두 옵션은 함께 쓸 수 있어야 합니다.

**정답 및 해설**

```c
/* 대소문자를 무시하고 비교한다 (-f) */
static int foldcmp(const char *s1, const char *s2)
{
    for ( ; tolower((unsigned char) *s1) == tolower((unsigned char) *s2); s1++, s2++)
        if (*s1 == '\0')
            return 0;
    return tolower((unsigned char) *s1) - tolower((unsigned char) *s2);
}

/* 정렬 방향까지 적용한 비교 */
static int compare(const char *s1, const char *s2,
                   int (*cmp)(const char *, const char *))
{
    int r = (*cmp)(s1, s2);

    return reverse_order ? -r : r;
}
```

```c
    cmp = fold_case ? foldcmp : strcmp_wrap;
```

`lines.txt`(banana, apple, Cherry, date)로 실행한 결과입니다.

```text
[옵션 없음]      Cherry apple banana date
[-f]             apple banana Cherry date
[-r]             date banana apple Cherry
[-rf]            date Cherry banana apple
```

- **두 옵션의 성격이 다릅니다.** `-f`는 **비교 함수 자체를 바꾸고**, `-r`은 **비교 결과의 부호를 뒤집습니다.** 그래서 `-f`는 함수 포인터로, `-r`은 결과에 `-`를 붙여 처리했습니다. 이렇게 나누면 두 옵션이 자연스럽게 함께 동작합니다.
- 역순을 만들 때 **비교 결과에 `-`를 붙이는 방법**이 가장 간단합니다. 정렬 알고리즘은 손대지 않아도 됩니다.
- `tolower`에 `(unsigned char)` 캐스트를 붙인 이유는 3강 7.4절에서 설명한 `char`의 부호 문제 때문입니다.

---

### 문제 6. 함수 포인터로 연산 고르기

`calcfn 3 + 4` 처럼 명령행에서 연산을 받아 계산하되, **연산자에 맞는 함수를 골라 호출**하도록 작성하십시오.

**정답 및 해설**

```c
static double add(double a, double b) { return a + b; }
static double sub(double a, double b) { return a - b; }
static double mul(double a, double b) { return a * b; }
static double dvd(double a, double b) { return (b != 0.0) ? a / b : 0.0; }

/* 연산자에 맞는 함수를 골라 돌려준다 */
static double (*select_op(char op))(double, double)
{
    switch (op) {
    case '+': return add;
    case '-': return sub;
    case '*': return mul;
    case '/': return dvd;
    default:  return NULL;
    }
}

int main(int argc, char *argv[])
{
    double (*fn)(double, double);

    if (argc != 4) {
        printf("사용법: %s 수1 연산자 수2\n", argv[0]);
        return 1;
    }

    fn = select_op(argv[2][0]);
    if (fn == NULL) {
        printf("알 수 없는 연산자: %s\n", argv[2]);
        return 1;
    }

    printf("%g\n", (*fn)(atof(argv[1]), atof(argv[3])));
    return 0;
}
```

```powershell
.\calcfn.exe 3 + 4
```

```text
7
```

```powershell
.\calcfn.exe 10 / 4
```

```text
2.5
```

- `select_op`의 선언이 이번 강의에서 가장 복잡합니다. 6.1절의 규칙으로 읽으면 **"`select_op`는 `char`를 받아, `double` 두 개를 받아 `double`을 돌려주는 함수를 가리키는 포인터를 돌려주는 함수"** 입니다.
- **`typedef`를 쓰면 훨씬 읽기 쉬워집니다.** 8강에서 배운 뒤 다시 고쳐 보시기 바랍니다.
- `NULL`을 실패 표시로 쓰고 **호출 전에 반드시 검사**했습니다(6강 5.4절). 검사 없이 `(*fn)(...)`을 호출하면 프로그램이 죽습니다.

---

### 문제 7. 버퍼 오버플로 재현과 방어

7절의 `bof_demo.c`를 직접 실행하여 값이 망가지는 것을 확인하고, 다음 두 가지를 조사하십시오.

1. `authorized`를 1로 만들려면 **몇 글자**가 필요한가?
2. `snprintf` 대신 다른 방법으로도 막을 수 있는가?

**정답 및 해설**

**① 필요한 글자 수**

`name`이 8칸이므로 **9글자째부터 `authorized` 영역을 침범**합니다.

| 입력 길이 | `authorized`의 값 |
|---|---|
| 7 이하 | 0 (정상, `'\0'`까지 `name` 안에 들어감) |
| 8 | 0 (`'\0'`이 9번째 칸에 들어가지만 값은 0) |
| 9 | 마지막 글자 한 개만 들어가 작은 값 |
| 12 | `0x41414141` = 1094795585 |

즉 **1로 만들려면** 9글자를 입력하되 마지막(9번째) 글자의 값이 1이어야 합니다. 보통의 키보드 문자로는 값이 1인 글자를 입력하기 어렵지만, **0이 아니기만 하면 조건은 참**이 되므로 9글자만 넣어도 권한 검사는 통과합니다. 이 사실이 더 중요합니다. **정확한 값을 맞출 필요조차 없다는 것**입니다.

**② 다른 방어 방법**

| 방법 | 코드 | 비고 |
|---|---|---|
| 길이를 먼저 검사 | `if (strlen(argv[1]) >= sizeof a.name) { 오류 처리 }` | 가장 명확하다. 자르지 않고 **거부**한다 |
| `snprintf` | `snprintf(a.name, sizeof a.name, "%s", argv[1])` | 조용히 자른다 |
| `strncpy` + 직접 `'\0'` | `strncpy(a.name, argv[1], sizeof a.name - 1); a.name[sizeof a.name - 1] = '\0';` | **끝 표시를 직접 붙여야** 한다(6강 실습문제 5) |

**입력을 자르는 것과 거부하는 것 중 무엇이 옳은지는 상황에 달려 있습니다.** 이름이 잘린 채 저장되는 것도 사고가 될 수 있으므로, 실무에서는 **거부하고 오류를 알리는 편**이 안전한 경우가 많습니다.

---

### 문제 8. 선언 읽기

다음 선언을 우리말로 옮기고, 각각이 무엇을 담을 수 있는지 설명하십시오.

```c
char *a[10];
char (*b)[10];
int (*c)(int, int);
int *d(int);
char **e;
```

**정답 및 해설**

| 선언 | 읽기 | 담는 것 |
|---|---|---|
| `char *a[10];` | `a`는 **char 포인터 10개짜리 배열** | 문자열 10개의 주소 |
| `char (*b)[10];` | `b`는 **char 10칸짜리 배열을 가리키는 포인터** | 2차원 배열의 한 줄 |
| `int (*c)(int, int);` | `c`는 **int 두 개를 받아 int를 돌려주는 함수를 가리키는 포인터** | 함수의 주소 |
| `int *d(int);` | `d`는 **int를 받아 int 포인터를 돌려주는 함수** | (함수 선언) |
| `char **e;` | `e`는 **char 포인터를 가리키는 포인터** | `argv`와 같은 구조 |

- `a`와 `b`, `c`와 `d`는 **괄호 하나로 뜻이 완전히 달라집니다.**
- `e`는 `main`의 `argv`와 같은 자료형입니다. `char *argv[]`와 `char **argv`는 매개변수 자리에서 같은 뜻입니다(6강 4.4절).
- 읽는 요령은 언제나 같습니다. **이름에서 시작해 괄호 안을 먼저, 그다음 `[]`·`()`, 마지막으로 `*`와 자료형**입니다.

---

### 문제 9. 문자열 목록 뒤집기

포인터 배열에 든 문자열들의 **순서를 뒤집는** 함수를 작성하십시오. 문자열 자체는 옮기지 않아야 합니다.

**정답 및 해설**

```c
void reverse_list(const char *v[], int n)
{
    const char *temp;
    int i, j;

    for (i = 0, j = n - 1; i < j; i++, j--) {
        temp = v[i];
        v[i] = v[j];
        v[j] = temp;
    }
}
```

- 4강의 문자열 뒤집기와 **구조가 똑같습니다.** 다만 맞바꾸는 것이 `char`가 아니라 **`const char *`(포인터)** 입니다.
- 그러므로 임시 변수 `temp`의 자료형도 **`const char *`** 여야 합니다.
- **문자열의 글자는 하나도 움직이지 않습니다.** 1.1절에서 말한 "화살표만 바꾼다"가 그대로 적용됩니다. 문자열이 아무리 길어도 비용은 같습니다.

---

### 문제 10. 2차원 배열로 성적표 만들기

학생 4명, 과목 3개의 점수를 2차원 배열에 담고 **학생별 평균과 과목별 평균**을 출력하십시오.

**정답 및 해설**

```c
#define STUDENTS 4
#define SUBJECTS 3

void print_report(int score[][SUBJECTS], int n)
{
    int i, j, sum;

    printf("학생  국어  영어  수학  평균\n");
    for (i = 0; i < n; i++) {
        sum = 0;
        printf("%3d ", i + 1);
        for (j = 0; j < SUBJECTS; j++) {
            printf("%5d ", score[i][j]);
            sum += score[i][j];
        }
        printf("%6.1f\n", (double) sum / SUBJECTS);
    }

    printf("과목평균");
    for (j = 0; j < SUBJECTS; j++) {
        sum = 0;
        for (i = 0; i < n; i++)
            sum += score[i][j];
        printf("%5.1f ", (double) sum / n);
    }
    printf("\n");
}
```

- **매개변수에 `int score[][SUBJECTS]`** 라고 적은 것이 이 문제의 핵심입니다. 2.3절에서 배운 대로 **두 번째 첨자는 반드시 필요**합니다.
- 학생별 평균은 **행을 따라**, 과목별 평균은 **열을 따라** 더합니다. 두 반복의 안팎이 뒤바뀌는 것을 확인하십시오.
- 평균을 구할 때 **`(double)` 캐스트**를 빠뜨리면 정수 나눗셈이 되어 소수점이 사라집니다(3강 7.5절).

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 예제 파일 — `sortlines.c`, `daytab.c`, `month_name.c`, `ptr_vs_2d.c`, `echo1.c`, `echo2.c`, `argvinfo.c`, `findcmd.c`, `fnptr.c`, `decl_read.c` |
| 2 | 실습문제 `ex1.c` ~ `ex10.c` |
| 3 | **버퍼 오버플로 실습 보고서** — `bof_demo.exe`를 정상 입력과 긴 입력으로 각각 실행한 화면, `bof_safe.exe`의 결과, 그리고 왜 값이 바뀌었는지에 대한 설명(그림 포함) |
| 4 | 실행 결과 화면 — 문제 3·5·6 |
| 5 | 짧은 서술 ① 줄을 정렬할 때 문자열이 아니라 포인터를 맞바꾸는 이유 |
| 6 | 짧은 서술 ② 포인터 배열과 2차원 배열을 각각 언제 쓰는 것이 좋은지 |
| 7 | 짧은 서술 ③ `bof_demo.c`에서 컴파일러 경고가 나오지 않은 이유 |

---

## 정리

| 구분 | 내용 |
|---|---|
| 포인터 배열 | `char *v[]` — 길이가 다른 문자열들을 다루는 표준 방법. **포인터만 맞바꾼다** |
| 2차원 배열 | "배열의 배열". `a[i][j]`, 함수에 넘길 때 **두 번째 첨자는 필수** |
| 둘의 차이 | 포인터 배열은 낭비가 없고 길이가 자유롭다 / 2차원 배열은 고정 크기, 내용 수정 가능 |
| 명령행 인자 | `main(int argc, char *argv[])`, `argv[0]`은 프로그램 이름, `argv[argc]`는 `NULL` |
| 옵션 처리 | `while (--argc > 0 && (*++argv)[0] == '-')` — 읽을 줄은 알되 풀어 쓰는 편이 낫다 |
| 함수 포인터 | `int (*comp)(...)` — **괄호 필수**. 동작을 인자로 넘겨 하나의 함수로 여러 기준 처리 |
| 함수 이름 | 그 자체가 주소. `&` 없이 넘기고, `comp(a,b)`로 호출해도 된다 |
| 선언 읽기 | **이름에서 시작해 안에서 바깥으로**. `()`·`[]`가 `*`보다 우선 |
| 버퍼 오버플로 | 배열을 넘겨 쓰면 **옆 변수의 값이 바뀐다**. `0x41414141` = `AAAA` |
| 방어 | 크기를 함께 넘긴다(`snprintf`), 길이를 먼저 검사한다, `sizeof`로 크기를 얻는다 |

포인터를 마쳤습니다. 이제 **여러 자료를 하나로 묶는 방법**이 남았습니다. 학생 한 명의 이름·학번·점수를 따로 관리하는 것은 번거롭고 실수하기 쉽습니다.

**다음 8강에서는 구조체(K&R 제6장)** 를 다룹니다. 서로 다른 자료형을 하나의 이름으로 묶고, 구조체의 배열을 만들고, 함수로 주고받으며, `typedef`로 이름을 붙이는 방법을 배웁니다. 이번 강의에서 복잡했던 함수 포인터 선언도 `typedef`로 훨씬 읽기 쉽게 만들 수 있습니다.

---

## 부록 A. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `sortlines.c` | 포인터 배열로 줄 정렬 | 1.3 |
| `daytab.c` | 2차원 배열로 날짜 변환 | 2.2 |
| `month_name.c` | 포인터 배열의 초기화 | 3.2 |
| `ptr_vs_2d.c` | 두 방식의 크기 비교 | 3.1 |
| `echo1.c` · `echo2.c` | 명령행 인자 출력 | 4.2 |
| `argvinfo.c` | `argc`·`argv` 확인 | 4.1 |
| `findcmd.c` | 옵션 해석 | 4.3 |
| `fnptr.c` | 함수 포인터로 정렬 기준 바꾸기 | 5.3 |
| `decl_read.c` | 복잡한 선언 | 6.1 |
| **`bof_demo.c`** | **버퍼 오버플로 재현** | 7.1 |
| **`bof_safe.c`** | **안전하게 고친 판** | 7.6 |

## 부록 B. 명령행 인자 실행 방법 (PowerShell)

| 하려는 일 | 명령 |
|---|---|
| 인자 넘기기 | `.\prog.exe hello world` |
| 공백이 든 인자 | `.\prog.exe "hello world"` |
| 특수 문자 인자 | `.\prog.exe 2 3 '*'` |
| 인자 + 입력 파일 | `.\findcmd.exe -n ould < poem.txt` |
| 종료 코드 확인 | `.\prog.exe; $LASTEXITCODE` |

> `*`, `>`, `|` 등은 PowerShell이 먼저 해석하므로 **작은따옴표로 감싸야** 프로그램에 그대로 전달됩니다.
{: .prompt-warning }
