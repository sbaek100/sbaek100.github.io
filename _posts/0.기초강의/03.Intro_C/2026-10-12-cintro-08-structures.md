---
title: C 프로그래밍 기초 8강 - 구조체
date: 2026-10-12 09:00:00 +0900
categories:
  - 0.기초강의
  - C 프로그래밍 기초
tags:
  - C언어
  - KnR
  - 구조체
  - struct
  - 구조체포인터
  - typedef
  - union
  - 비트필드
  - malloc
  - 연결리스트
  - 이진트리
  - 해시테이블
pin:
mermaid: false
---

> **학습 목표**
> 1. 구조체를 선언하고 멤버에 접근할 수 있으며, 구조체를 중첩할 수 있다.
> 2. 구조체를 함수에 넘기고 돌려받을 수 있고, **값 전달과 주소 전달**의 차이를 설명할 수 있다.
> 3. `->` 연산자를 사용할 수 있고 `(*p).x`와의 관계를 설명할 수 있다.
> 4. 구조체 배열을 선언·초기화하고 검색할 수 있다.
> 5. `sizeof`가 **멤버 크기의 단순 합이 아닌 이유**(패딩)를 설명할 수 있다.
> 6. `malloc`·`free`로 필요한 만큼 메모리를 빌리고 돌려줄 수 있다.
> 7. **자기 자신을 가리키는 구조체**로 연결 리스트와 이진 트리를 만들 수 있다.
> 8. 해시 테이블의 원리를 설명하고 간단히 구현할 수 있다.
> 9. `typedef`로 자료형에 이름을 붙여 선언을 읽기 쉽게 만들 수 있다.
> 10. 공용체(`union`)와 비트 필드가 필요한 상황을 설명하고 사용할 수 있다.
{: .prompt-info }

## 0. 이번 강의의 위치

지금까지 우리가 다룬 자료는 **같은 종류끼리만** 묶을 수 있었습니다. 배열은 `int`면 `int`만, `char`면 `char`만 담습니다. 그런데 실제 자료는 그렇지 않습니다.

> 학생 한 명 = **학번(정수) + 이름(문자열) + 점수(정수 배열) + 평균(실수)**

이것을 지금까지 배운 것만으로 다루려면 배열 네 개를 따로 만들고 **첨자가 어긋나지 않도록 사람이 신경 써야** 합니다. 한 학생을 함수에 넘기려면 인자를 네 개 적어야 하고, 학생을 하나 지우면 배열 네 개를 모두 고쳐야 합니다.

**구조체는 서로 다른 자료형을 하나의 이름으로 묶는 장치**입니다. 이번 강의를 마치면 자료를 "덩어리"로 다룰 수 있게 되며, 이는 곧 프로그램을 자료의 구조에 맞추어 설계할 수 있게 된다는 뜻입니다.

> **참고 문헌**
> 이번 강의는 다음 책의 **제6장 Structures** 전체를 바탕으로 재구성한 것입니다.
> Brian W. Kernighan, Dennis M. Ritchie, *The C Programming Language*, 2nd Edition, Prentice Hall, 1988.
{: .prompt-info }

| K&R | 원서 절 제목 | 이번 강의 |
|---|---|---|
| 6.1 | Basics of Structures | 제1절 |
| 6.2 | Structures and Functions | 제2절 · 제3절 |
| 6.3 | Arrays of Structures | 제4절 |
| 6.4 | Pointers to Structures | 제5절 |
| 6.5 | Self-referential Structures | 제6절 · 제7절 |
| 6.6 | Table Lookup | 제8절 |
| 6.7 | Typedef | 제9절 |
| 6.8 | Unions | 제10절 |
| 6.9 | Bit-fields | 제11절 |

이 강의는 **3회차 분량**(모두 합쳐 약 495분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제5절 | 구조체의 기본과 패딩 | 175분 |
| **2회차** | 제6절 ~ 제8절 | 동적 메모리와 자료 구조 | 125분 |
| **3회차** | 제9절 ~ 실습문제 | `typedef`·`union`·비트 필드와 실습 | 195분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 구조체의 기본 | 35분 |
| 제2절 | 구조체와 함수 | 35분 |
| 제3절 | 구조체 포인터와 `->` | 35분 |
| 제4절 | 구조체 배열 | 40분 |
| 제5절 | 구조체 포인터로 다루기와 패딩 | 30분 |
| 제6절 | `malloc`·`free` | 30분 |
| 제7절 | 자기 참조 구조체 — 리스트와 트리 | 55분 |
| 제8절 | 해시 테이블 | 40분 |
| 제9절 | `typedef` | 25분 |
| 제10절 | 공용체 `union` | 25분 |
| 제11절 | 비트 필드 | 20분 |
| 제12절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 110분 |

**작업 폴더**는 `C:\c-study\lab08`을 사용합니다.

```powershell
mkdir C:\c-study\lab08
```

---

## 제1절. 구조체의 기본

### 1.1 선언

**구조체(structure)** 는 하나 이상의 변수를 하나의 이름으로 묶은 것입니다. 묶인 변수 하나하나를 **멤버(member)** 라고 합니다.

```c
struct point {          /* 점: x, y 좌표 */
    int x;
    int y;
};
```

| 부분 | 이름 | 설명 |
|---|---|---|
| `struct` | 키워드 | 구조체 선언의 시작 |
| `point` | **구조체 태그** | 이 모양에 붙인 이름. 이후 `struct point`로 쓸 수 있다 |
| `{ ... }` | 멤버 목록 | 묶을 변수들 |
| `;` | — | **반드시 필요하다.** 잊기 쉬운 세미콜론이다 |

이 선언만으로는 **저장 공간이 생기지 않습니다.** "이런 모양이 있다"고 알릴 뿐이며, 변수를 만들려면 따로 정의해야 합니다.

```c
struct point pt;                 /* 변수 하나 */
struct point pt2 = {4, 3};       /* 초기화 */
```

### 1.2 멤버 접근과 중첩

멤버는 **점(`.`) 연산자**로 접근합니다.

```c
printf("%d, %d", pt.x, pt.y);
```

구조체 안에 구조체를 넣을 수도 있습니다.

```c
struct rect {           /* 사각형: 두 점 */
    struct point pt1;
    struct point pt2;
};
```

```c
screen.pt1.x        /* screen 의 pt1 의 x */
```

`point.c`로 확인합니다.

```c
int main(void)
{
    struct point pt = {4, 3};       /* 초기화 */
    struct point origin;
    struct rect screen;

    origin.x = 0;                   /* 멤버 접근 */
    origin.y = 0;

    screen.pt1 = origin;            /* 구조체끼리 통째로 대입 */
    screen.pt2 = pt;

    printf("pt        = (%d, %d)\n", pt.x, pt.y);
    printf("origin    = (%d, %d)\n", origin.x, origin.y);
    printf("screen.pt2.x = %d   <- 중첩된 멤버\n", screen.pt2.x);

    printf("sizeof(struct point) = %zu\n", sizeof(struct point));
    printf("sizeof(struct rect)  = %zu\n", sizeof(struct rect));
    return 0;
}
```

```text
pt        = (4, 3)
origin    = (0, 0)
screen.pt2.x = 4   <- 중첩된 멤버
sizeof(struct point) = 8
sizeof(struct rect)  = 16
```

### 1.3 구조체에 할 수 있는 일

| 할 수 있는 것 | 예 |
|---|---|
| 통째로 **대입·복사** | `screen.pt1 = origin;` |
| 함수에 **넘기고 돌려받기** | `sum = addpoint(a, b);` |
| **주소** 얻기 | `&origin` |
| **멤버 접근** | `origin.x` |

**할 수 없는 것도 분명합니다. 구조체끼리 비교할 수 없습니다.**

```c
if (a == b)        /* 오류! 구조체는 == 로 비교할 수 없다 */
```

같은지 알아보려면 **멤버를 하나씩 비교하는 함수를 직접 만들어야** 합니다. 구조체 안에 빈 자리(제5절의 패딩)가 있을 수 있어 메모리를 통째로 비교하는 것도 안전하지 않습니다.

> 멤버 이름은 **다른 구조체의 멤버 이름이나 보통 변수 이름과 겹쳐도 됩니다.** 문맥으로 구분되기 때문입니다. 오히려 관련된 것끼리 같은 이름을 쓰는 편이 읽기 좋습니다.
{: .prompt-info }

---

## 제2절. 구조체와 함수

### 2.1 구조체를 주고받는 함수

`pointfn.c`입니다.

```c
/* makepoint: 좌표 두 개로 점을 만든다 */
struct point makepoint(int x, int y)
{
    struct point temp;

    temp.x = x;
    temp.y = y;
    return temp;
}

/* addpoint: 두 점을 더한다 */
struct point addpoint(struct point p1, struct point p2)
{
    p1.x += p2.x;          /* 매개변수는 복사본이므로 마음껏 고쳐도 된다 */
    p1.y += p2.y;
    return p1;
}

/* ptinrect: 점 p 가 사각형 r 안에 있으면 1 */
int ptinrect(struct point p, struct rect r)
{
    return p.x >= r.pt1.x && p.x < r.pt2.x
        && p.y >= r.pt1.y && p.y < r.pt2.y;
}
```

```text
a + b = (7, 4)
호출 후 a = (2, 3)   <- 원본은 그대로
(2,3) 이 사각형 안에 있는가? 1
(20,3) 이 사각형 안에 있는가? 0
정리 후: (0,0) ~ (10,8)
```

**`addpoint`가 `p1`을 직접 고쳤는데도 호출한 쪽의 `a`는 그대로입니다.** 구조체도 다른 값과 마찬가지로 **값에 의한 호출**로 전달되기 때문입니다(2강 9절). 매개변수는 이미 복사본이므로 임시 변수를 따로 만들 필요가 없습니다.

### 2.2 매개변수 이름과 멤버 이름이 같아도 됩니다

`makepoint(int x, int y)`의 `x`, `y`는 `struct point`의 멤버 `x`, `y`와 이름이 같습니다. **충돌하지 않으며**, 오히려 둘의 관계를 드러내 주므로 좋은 이름 짓기입니다.

### 2.3 매크로와 함께 쓰기

```c
#define min(a, b)  ((a) < (b) ? (a) : (b))
#define max(a, b)  ((a) > (b) ? (a) : (b))

/* canonrect: pt1 이 왼쪽 아래가 되도록 정리한다 */
struct rect canonrect(struct rect r)
{
    struct rect temp;

    temp.pt1.x = min(r.pt1.x, r.pt2.x);
    temp.pt1.y = min(r.pt1.y, r.pt2.y);
    temp.pt2.x = max(r.pt1.x, r.pt2.x);
    temp.pt2.y = max(r.pt1.y, r.pt2.y);
    return temp;
}
```

5강 12.2절에서 배운 대로 **인자와 전체를 괄호로 감쌌습니다.** 그리고 인자에 `++` 같은 부작용이 없으므로 안전하게 쓸 수 있습니다.

---

## 제3절. 구조체 포인터와 `->`

### 3.1 큰 구조체는 주소로 넘깁니다

구조체가 커지면 통째로 복사하는 비용이 커집니다. 그럴 때는 **포인터를 넘깁니다.** 그러면 원본을 고칠 수도 있습니다.

```c
struct point *pp;      /* 구조체를 가리키는 포인터 */
```

`*pp`가 구조체이므로 멤버는 `(*pp).x`로 접근합니다. **괄호가 반드시 필요합니다.**

```c
(*pp).x        /* 올바름 */
*pp.x          /* 잘못 — *(pp.x) 로 해석된다 */
```

`.`가 `*`보다 우선순위가 높기 때문입니다(3강 12.1절).

### 3.2 `->` — 훨씬 편한 표기

이 형태가 워낙 자주 쓰이므로 전용 연산자가 있습니다.

```c
pp->x          /* (*pp).x 와 완전히 같다 */
```

`ptrstruct.c`로 확인합니다.

```c
/* 구조체 포인터를 받아 원본을 옮긴다 */
void move(struct point *pp, int dx, int dy)
{
    pp->x += dx;          /* (*pp).x 와 같다 */
    pp->y += dy;
}
```

```text
(*pp).x = 1, pp->x = 1   <- 같은 뜻
move 후 origin = (4, 6)   <- 원본이 바뀌었다
r.pt1.x = 0, rp->pt1.x = 0, (r.pt1).x = 0, (rp->pt1).x = 0
++pp->x 는 pp 가 아니라 x 를 증가시킨다: origin.x = 5
```

### 3.3 우선순위 — `.`과 `->`는 가장 세다

`.`과 `->`는 `()`·`[]`와 함께 **우선순위 최상위**입니다. 그래서 다음이 성립합니다.

| 식 | 실제 해석 | 무엇이 증가하는가 |
|---|---|---|
| `++p->len` | `++(p->len)` | **멤버 `len`** |
| `(++p)->len` | 괄호가 우선 | 포인터 `p`를 먼저 옮긴 뒤 멤버 접근 |
| `(p++)->len` | 멤버 접근 후 | 포인터 `p` |
| `*p->str` | `*(p->str)` | `str`이 가리키는 값 |
| `*p->str++` | `*((p->str)++)` | 값을 쓴 뒤 **`str`** 이 전진 |

> 이런 식은 **읽을 줄만 알면 충분합니다.** 직접 쓸 때는 두 줄로 나누는 편이 안전합니다.
{: .prompt-tip }

---

## 제4절. 구조체 배열

### 4.1 나란한 배열 대신 구조체 배열

C 예약어가 소스에 몇 번 나오는지 세는 프로그램을 만듭니다. 예약어 목록과 횟수가 필요한데, 배열 두 개를 나란히 두는 방법은 좋지 않습니다.

```c
char *keyword[NKEYS];      /* 첨자가 어긋나면 그대로 버그 */
int   keycount[NKEYS];
```

**둘이 짝을 이룬다는 사실 자체가 "구조체로 묶으라"는 신호**입니다.

```c
struct key {
    const char *word;
    int count;
} keytab[] = {              /* 반드시 사전 순으로 정렬되어 있어야 한다 */
    {"break", 0},
    {"case", 0},
    {"char", 0},
    /* ... */
    {"while", 0}
};
```

- 구조체 선언과 배열 정의·초기화를 **한 번에** 했습니다.
- 초기값은 **구조체마다 중괄호로 묶어** 적는 것이 읽기 좋습니다.
- 크기를 비워 두면 초기값의 개수를 세어 컴파일러가 정합니다.

### 4.2 원소 개수 구하기

```c
#define NKEYS  (int) (sizeof keytab / sizeof keytab[0])
```

**전체 크기 ÷ 원소 하나의 크기**입니다(4강 3.2절). 손으로 세는 것보다 안전하며, 표에 항목을 더해도 저절로 맞습니다.

> `sizeof keytab / sizeof(struct key)`라고 써도 되지만, **`keytab[0]`으로 나누는 편**이 낫습니다. 나중에 자료형이 바뀌어도 고칠 필요가 없기 때문입니다.
{: .prompt-tip }

### 4.3 프로그램

`keycount.c`의 핵심입니다.

```c
int main(void)
{
    int n;
    char word[MAXWORD];

    while (getword(word, MAXWORD) != EOF)
        if (isalpha((unsigned char) word[0]))
            if ((n = binsearch(word, keytab, NKEYS)) >= 0)
                keytab[n].count++;

    for (n = 0; n < NKEYS; n++)
        if (keytab[n].count > 0)
            printf("%4d %s\n", keytab[n].count, keytab[n].word);
    return 0;
}

/* binsearch: tab[0]...tab[n-1] 에서 word 를 찾는다 */
int binsearch(const char *word, struct key tab[], int n)
{
    int cond, low, high, mid;

    low = 0;
    high = n - 1;
    while (low <= high) {
        mid = (low + high) / 2;
        if ((cond = strcmp(word, tab[mid].word)) < 0)
            high = mid - 1;
        else if (cond > 0)
            low = mid + 1;
        else
            return mid;
    }
    return -1;
}
```

다음 내용의 `sample.txt`로 실행합니다.

```text
int main(void)
{
    int i;
    for (i = 0; i < 10; i++)
        if (i % 2 == 0)
            continue;
    return 0;
}
```

```powershell
.\keycount.exe < sample.txt
```

```text
   1 continue
   1 for
   1 if
   2 int
   1 return
   1 void
```

4강의 이진 탐색과 구조가 같고, **비교 대상이 `tab[mid]`가 아니라 `tab[mid].word`** 라는 점만 다릅니다.

`getword`는 다음 단어를 읽어 오는 함수로, 6강에서 배운 `ungetc`로 **한 글자 미리 읽은 것을 되돌립니다.**

```c
    for ( ; --lim > 0; w++)
        if (!isalnum(*w = (char) getchar())) {
            ungetc(*w, stdin);
            break;
        }
```

---

## 제5절. 구조체 포인터로 다루기와 패딩

### 4.3절의 프로그램을 포인터로 다시 씁니다

`keycount_ptr.c`입니다. 배열 첨자 대신 포인터를 사용합니다.

```c
int main(void)
{
    char word[MAXWORD];
    struct key *p;

    while (getword(word, MAXWORD) != EOF)
        if (isalpha((unsigned char) word[0]))
            if ((p = binsearch(word, keytab, NKEYS)) != NULL)
                p->count++;

    for (p = keytab; p < keytab + NKEYS; p++)
        if (p->count > 0)
            printf("%4d %s\n", p->count, p->word);
    return 0;
}

/* binsearch: 찾으면 그 자리의 포인터, 못 찾으면 NULL */
struct key *binsearch(const char *word, struct key *tab, int n)
{
    int cond;
    struct key *low = &tab[0];
    struct key *high = &tab[n];
    struct key *mid;

    while (low < high) {
        mid = low + (high - low) / 2;      /* 포인터끼리는 더할 수 없다 */
        if ((cond = strcmp(word, mid->word)) < 0)
            high = mid;
        else if (cond > 0)
            low = mid + 1;
        else
            return mid;
    }
    return NULL;
}
```

같은 입력으로 실행하면(표에 예약어 6개만 담은 판입니다):

```text
   2 int
   1 return
   1 void
```

바뀐 점이 네 가지입니다.

| 항목 | 배열판 | 포인터판 |
|---|---|---|
| 반환값 | 첨자(`int`), 실패는 `-1` | **포인터**, 실패는 `NULL` |
| 반복 | `for (n = 0; n < NKEYS; n++)` | `for (p = keytab; p < keytab + NKEYS; p++)` |
| 중간값 | `mid = (low + high) / 2` | **`mid = low + (high - low) / 2`** |
| 접근 | `tab[mid].word` | `mid->word` |

> **`mid = (low + high) / 2` 는 포인터에서는 쓸 수 없습니다.**
> 6강 5.1절에서 배운 대로 **포인터끼리 더하는 것은 허용되지 않습니다.** 뺄셈은 되므로 `low + (high - low) / 2`로 씁니다.
>
> 또 하나, `&tab[n]`은 **배열의 끝 바로 다음**을 가리킵니다. 이 자리는 **계산과 비교에만 쓸 수 있고 값을 읽으면 안 됩니다**(6강 5.5절). 그래서 조건이 `low <= high`가 아니라 `low < high`입니다.
{: .prompt-warning }

**`p++`는 구조체 하나만큼 건너뜁니다.** 6강에서 배운 대로 포인터 연산은 가리키는 자료형의 크기를 자동으로 반영합니다.

### 5.2 `sizeof`는 멤버 크기의 합이 아닙니다

여기서 매우 중요한 사실이 나옵니다. `padding.c`로 확인합니다.

```c
struct bad {            /* 배치가 나쁜 예 */
    char  c;            /* 1바이트 */
    int   i;            /* 4바이트 — 4의 배수 자리에 놓여야 한다 */
    char  d;            /* 1바이트 */
};

struct good {           /* 큰 것부터 모아 둔 예 */
    int   i;
    char  c;
    char  d;
};
```

```text
멤버 크기의 단순 합 = 6
sizeof(struct bad)  = 12   <- 빈 자리(패딩)가 생긴다
sizeof(struct good) = 8   <- 순서만 바꿔도 줄어든다
```

멤버를 다 합치면 6바이트인데 실제로는 12바이트입니다. 왜 그럴까요?

**대부분의 CPU는 자료형을 아무 주소에나 놓지 못합니다.** `int`는 대개 4의 배수 주소에 놓여야 빠르게 읽힙니다. 그래서 컴파일러가 **중간에 빈 자리를 끼워 넣는데**, 이를 **패딩(padding)** 이라 합니다.

| `struct bad`의 배치 | 0 | 1~3 | 4~7 | 8 | 9~11 |
|---|---|---|---|---|---|
| 내용 | `c` | **빈 자리** | `i` | `d` | **빈 자리** |

여기서 실무 규칙 두 가지가 나옵니다.

1. **크기가 큰 멤버부터 선언하면** 빈 자리가 줄어듭니다. 위에서 순서만 바꾸었더니 12바이트가 8바이트가 되었습니다.
2. **구조체의 크기는 반드시 `sizeof`로 구하십시오.** 멤버를 더해 계산하면 틀립니다. 파일에 저장하거나 통신으로 보낼 때 특히 중요합니다.

---

> **▶ 여기서부터 2회차 — 동적 메모리와 자료 구조**
> 제6절 ~ 제8절, 약 125분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제6절. `malloc`과 `free`

### 6.1 필요한 만큼 빌려 쓰기

6강에서 만든 `alloc`은 미리 잡아 둔 큰 배열을 잘라 주는 방식이었고, **되돌려 주는 순서도 제한**되어 있었습니다. 표준 라이브러리에는 훨씬 제대로 된 것이 있습니다.

```c
#include <stdlib.h>

void *malloc(size_t size);   /* size 바이트를 빌린다 */
void  free(void *p);         /* 빌린 것을 돌려준다 */
```

| 함수 | 하는 일 | 실패하면 |
|---|---|---|
| `malloc(n)` | `n`바이트를 확보하고 **그 주소**를 돌려준다 | **`NULL`** |
| `free(p)` | `malloc`으로 받은 자리를 반납한다 | — |

`malloc`이 돌려주는 것은 **`void *`(어떤 자료형이든 가리킬 수 있는 포인터)** 이므로, 쓰려는 자료형으로 캐스트합니다.

```c
struct tnode *p = (struct tnode *) malloc(sizeof(struct tnode));
```

**크기는 반드시 `sizeof`로 구하십시오**(5.2절). 그리고 **결과가 `NULL`인지 반드시 검사해야 합니다.**

### 6.2 지켜야 할 규칙

| 규칙 | 어기면 |
|---|---|
| `malloc`의 결과를 **`NULL` 검사** | 메모리가 부족할 때 프로그램이 죽는다 |
| 다 쓴 것은 **`free`** | **메모리 누수** — 프로그램이 점점 메모리를 잡아먹는다 |
| `free` 한 것을 **다시 쓰지 않는다** | 이미 남에게 넘어간 자리를 건드린다(심각한 보안 문제) |
| **두 번 `free` 하지 않는다** | 프로그램이 죽거나 메모리 관리가 망가진다 |
| `malloc`으로 받은 주소만 `free` | 그 밖의 주소를 넘기면 무슨 일이 생길지 알 수 없다 |

> 문자열을 복사해 보관하는 함수를 만들 때가 많습니다. 6강 실습문제 9에서 만든 것과 같은 일을 `malloc`으로 합니다.
> ```c
> static char *my_strdup(const char *s)
> {
>     char *p = (char *) malloc(strlen(s) + 1);   /* '\0' 자리까지 */
>
>     if (p != NULL)
>         strcpy(p, s);
>     return p;
> }
> ```
> **`+ 1`을 잊으면** 끝 표시를 쓸 자리가 없어 바로 옆 메모리를 침범합니다. 7강에서 본 버퍼 오버플로와 같은 종류의 사고입니다.
{: .prompt-danger }

---

## 제7절. 자기 참조 구조체 — 리스트와 트리

### 7.1 자기 자신을 가리키는 구조체

구조체는 **자기 자신을 멤버로 담을 수는 없지만, 자기 자신을 가리키는 포인터는 담을 수 있습니다.**

```c
struct tnode {              /* 트리의 마디 */
    char *word;             /* 단어를 가리킨다 */
    int count;              /* 나온 횟수 */
    struct tnode *left;     /* 왼쪽 자식 */
    struct tnode *right;    /* 오른쪽 자식 */
};
```

`struct tnode left;`라고 쓰면 크기가 무한이 되어 불가능하지만, **포인터는 크기가 정해져 있으므로** 문제가 없습니다. 이 성질 덕분에 **원소를 필요할 때마다 이어 붙이는 자료 구조**를 만들 수 있습니다.

### 7.2 이진 트리로 단어 세기

입력에 나온 **모든 단어**의 빈도를 세려면 어떻게 해야 할까요? 4절처럼 미리 정해진 표가 없으므로 이진 탐색을 쓸 수 없습니다. 배열에 넣고 매번 처음부터 찾으면 단어가 많아질수록 급격히 느려집니다.

**이진 트리**를 쓰면 됩니다. 마디마다 단어 하나를 두고 **왼쪽에는 작은 단어, 오른쪽에는 큰 단어**를 놓습니다.

```text
                now
             /       \
           is         the
          /  \       /    \
        for  men    of     time
       /   \       /      /    \
     all   good  party  their   to
    /   \
  aid   come
```

찾거나 넣는 방법은 **재귀**입니다. 지금 마디와 비교해서 작으면 왼쪽, 크면 오른쪽으로 내려가고, **빈 자리(`NULL`)를 만나면 거기가 새 단어의 자리**입니다.

`tree.c`의 핵심입니다.

```c
/* addtree: w 를 p 아래에 넣는다 */
struct tnode *addtree(struct tnode *p, const char *w)
{
    int cond;

    if (p == NULL) {                 /* 처음 보는 단어 */
        p = talloc();
        if (p == NULL)
            return NULL;
        p->word = my_strdup(w);
        p->count = 1;
        p->left = p->right = NULL;
    } else if ((cond = strcmp(w, p->word)) == 0) {
        p->count++;                  /* 이미 있는 단어 */
    } else if (cond < 0) {
        p->left = addtree(p->left, w);
    } else {
        p->right = addtree(p->right, w);
    }
    return p;
}

/* treeprint: 사전 순으로 출력한다 */
void treeprint(const struct tnode *p)
{
    if (p != NULL) {
        treeprint(p->left);
        printf("%4d %s\n", p->count, p->word);
        treeprint(p->right);
    }
}

/* treefree: 빌린 메모리를 모두 돌려준다 */
void treefree(struct tnode *p)
{
    if (p != NULL) {
        treefree(p->left);
        treefree(p->right);
        free(p->word);
        free(p);
    }
}
```

다음 내용의 `words.txt`로 실행합니다.

```text
now is the time for all good men to come to the aid of their party
```

```powershell
.\tree.exe < words.txt
```

```text
   1 aid
   1 all
   1 come
   1 for
   1 good
   1 is
   1 men
   1 now
   1 of
   1 party
   2 the
   1 their
   1 time
   2 to
```

**정렬하지 않았는데 사전 순으로 나왔습니다.** `treeprint`가 **왼쪽 → 자기 자신 → 오른쪽** 순으로 방문하기 때문이며, 트리의 구조 자체가 이미 정렬을 담고 있는 것입니다.

| 함수 | 하는 일 | 재귀 |
|---|---|---|
| `addtree` | 알맞은 자리에 넣거나 횟수를 증가 | 자식으로 내려간다 |
| `treeprint` | 왼쪽 → 자기 → 오른쪽 순으로 출력 | 양쪽 자식 |
| `treefree` | **자식을 먼저** 반납한 뒤 자기를 반납 | 양쪽 자식 |

> **`treefree`의 순서가 중요합니다.** 자기를 먼저 `free` 하면 자식의 주소를 잃어버려 반납할 수 없게 됩니다. 그러면 그 메모리는 프로그램이 끝날 때까지 낭비됩니다(**메모리 누수**).
{: .prompt-warning }

> 트리가 **한쪽으로만 자라면**(입력이 이미 정렬되어 있을 때) 사실상 일렬로 늘어서서 느려집니다. 이를 막는 균형 트리가 있으나 이 과정의 범위를 넘어섭니다.
{: .prompt-info }

---

## 제8절. 해시 테이블

### 8.1 이름으로 값을 찾는 표

`#define IN 1` 같은 정의를 기억해 두었다가 나중에 `IN`이 나오면 `1`로 바꾸어야 한다고 합시다. 이런 **이름 → 값** 표는 컴파일러와 매크로 처리기의 핵심 부품입니다.

이름을 하나하나 비교하면 느립니다. **해시(hash)** 는 이름을 계산해 **작은 정수(자리 번호)** 로 바꾸고, 그 자리부터 찾는 방법입니다.

```c
/* hash: 문자열에서 자리 번호를 만든다 */
static unsigned hash(const char *s)
{
    unsigned hashval;

    for (hashval = 0; *s != '\0'; s++)
        hashval = (unsigned char) *s + 31 * hashval;
    return hashval % HASHSIZE;
}
```

**`unsigned`로 계산하는 이유**는 넘침이 일어나도 결과가 음수가 되지 않도록 하기 위해서입니다(3강 2.2절). 첨자에 음수가 들어가면 배열 밖을 건드립니다.

### 8.2 같은 자리에 여러 이름이 오면

서로 다른 이름이 같은 자리 번호를 받을 수 있습니다. 이를 **충돌**이라 하며, 해결책은 **그 자리에 목록을 매다는 것**입니다.

```c
struct nlist {              /* 표의 한 칸 */
    struct nlist *next;     /* 같은 자리에 걸린 다음 칸 */
    char *name;
    char *defn;
};

static struct nlist *hashtab[HASHSIZE];
```

`next`가 다시 `struct nlist`를 가리킵니다. **연결 리스트**이며, 7.1절의 자기 참조 구조체가 여기서도 쓰입니다.

### 8.3 찾기와 넣기

```c
/* lookup: 이름을 찾는다. 없으면 NULL */
struct nlist *lookup(const char *s)
{
    struct nlist *np;

    for (np = hashtab[hash(s)]; np != NULL; np = np->next)
        if (strcmp(s, np->name) == 0)
            return np;
    return NULL;
}
```

이 `for` 문이 **연결 리스트를 따라가는 표준 관용구**입니다.

```c
for (ptr = head; ptr != NULL; ptr = ptr->next)
```

```c
/* install: (이름, 정의)를 표에 넣는다 */
struct nlist *install(const char *name, const char *defn)
{
    struct nlist *np;
    unsigned hashval;

    if ((np = lookup(name)) == NULL) {          /* 새 이름 */
        np = (struct nlist *) malloc(sizeof(*np));
        if (np == NULL || (np->name = my_strdup(name)) == NULL)
            return NULL;
        hashval = hash(name);
        np->next = hashtab[hashval];            /* 맨 앞에 매단다 */
        hashtab[hashval] = np;
    } else {                                    /* 이미 있는 이름 */
        free(np->defn);                         /* 옛 정의를 버린다 */
    }
    if ((np->defn = my_strdup(defn)) == NULL)
        return NULL;
    return np;
}
```

```text
lookup("OUT")     -> 0
다시 정의한 뒤     -> 9
lookup("NOTHING") -> (없음)
hash("IN") = 18, hash("OUT") = 60
```

- **새 칸을 맨 앞에 매답니다.** 끝을 찾아갈 필요가 없어 간단하고 빠릅니다.
- 이미 있는 이름이면 **옛 정의를 `free` 하고** 새 정의를 넣습니다. 이것을 빠뜨리면 메모리 누수입니다.
- `sizeof(*np)`라고 쓴 것에 주목하십시오. `sizeof(struct nlist)`와 같지만, **자료형이 바뀌어도 고칠 필요가 없습니다.**

---

> **▶ 여기서부터 3회차 — `typedef`·`union`·비트 필드와 실습**
> 제9절 ~ 실습문제, 약 195분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제9절. `typedef`

### 9.1 자료형에 이름을 붙입니다

```c
typedef int Length;               /* Length 는 int 의 다른 이름 */
typedef char *String;             /* String 은 char * 의 다른 이름 */
```

이제 `Length len;`은 `int len;`과 완전히 같습니다. **새로운 자료형이 생기는 것이 아니라 기존 자료형에 별명이 붙는 것**입니다.

새 이름은 `typedef` 바로 뒤가 아니라 **변수 이름이 올 자리**에 적습니다. 관례적으로 **첫 글자를 대문자**로 씁니다.

### 9.2 구조체와 함수 포인터에 쓰기

```c
typedef struct point {            /* 구조체에 짧은 이름을 붙인다 */
    int x;
    int y;
} Point;

typedef int (*CompareFn)(const char *, const char *);   /* 함수 포인터 */
```

이제 `struct point pt;` 대신 **`Point pt;`** 로 쓸 수 있습니다.

두 번째 줄이 이번 절의 핵심입니다. 7강 5절에서 함수 포인터 선언이 복잡하다고 했는데, `typedef`를 쓰면 이렇게 달라집니다.

```c
/* 7강의 표기 */
const char *pick(const char *a, const char *b,
                 int (*cmp)(const char *, const char *));
```

```c
/* typedef 를 쓴 표기 */
const char *pick(const char *a, const char *b, CompareFn cmp);
```

```text
Length len = 5
String s   = hello
Point p    = (3, 4)
길이 기준으로 고르면: fig
첫 글자 기준으로 고르면: apple
```

### 9.3 왜 쓰는가

| 이유 | 설명 |
|---|---|
| **이식성** | 환경마다 다른 자료형을 `typedef` 한 줄만 고쳐 대응할 수 있다. 표준의 `size_t`·`ptrdiff_t`가 그 예 |
| **가독성** | `Treeptr`이 "복잡한 구조체를 가리키는 포인터"보다 읽기 쉽다 |
| **복잡한 선언 정리** | 함수 포인터·배열 포인터를 단계적으로 만들 수 있다 |

> `typedef`는 `#define`과 달리 **컴파일러가 처리**합니다. 그래서 단순한 글자 치환으로는 불가능한 것(함수 포인터 등)도 다룰 수 있습니다.
{: .prompt-info }

---

## 제10절. 공용체 `union`

### 10.1 같은 자리를 돌려 쓰기

**공용체(union)** 는 여러 자료형 중 **한 번에 하나만** 담는 변수입니다. 문법은 구조체와 같지만 성질이 다릅니다.

| 구분 | 구조체 | 공용체 |
|---|---|---|
| 멤버 배치 | 나란히 놓인다 | **모두 같은 자리에서 시작** |
| 크기 | 멤버 크기의 합(+패딩) | **가장 큰 멤버의 크기** |
| 동시에 담기 | 모든 멤버를 함께 | **하나만** |

값이 정수일 수도 실수일 수도 문자열일 수도 있는 자료를 하나의 표에 담을 때 씁니다.

```c
enum vtype { INT_VAL, FLOAT_VAL, STR_VAL };

struct value {
    enum vtype type;        /* 지금 무엇이 들어 있는지 기억한다 */
    union {
        int ival;
        float fval;
        const char *sval;
    } u;
};
```

```text
정수  : 42
실수  : 3.5
문자열: hello
sizeof(int)=4, sizeof(float)=4, sizeof(char *)=8
sizeof(공용체) = 8   <- 가장 큰 멤버에 맞춘다
sizeof(struct value) = 16
```

### 10.2 무엇이 들어 있는지는 스스로 기억해야 합니다

공용체는 **지금 어떤 멤버가 유효한지 알려 주지 않습니다.** 정수를 넣고 실수로 꺼내면 아무 의미 없는 값이 나옵니다.

그래서 위 예제처럼 **무엇이 들어 있는지 기록하는 멤버(태그)를 함께 두는 것**이 표준적인 방법입니다. `enum`(3강 3.8절)이 그 역할에 잘 맞습니다.

> 공용체를 초기화할 때는 **첫 번째 멤버의 자료형**으로만 할 수 있습니다. 나머지는 대입으로 넣어야 합니다.
{: .prompt-warning }

---

## 제11절. 비트 필드

### 11.1 한 비트씩 쪼개 쓰기

권한 플래그처럼 **참/거짓 몇 개**를 담는 데 `int`를 여러 개 쓰는 것은 낭비입니다. 3강 9절에서는 마스크와 비트 연산으로 처리했습니다.

```c
#define PERM_READ   01
#define PERM_WRITE  02
#define PERM_EXEC   04

flags |= PERM_READ | PERM_WRITE;      /* 켜기 */
flags &= ~PERM_WRITE;                 /* 끄기 */
if ((flags & PERM_READ) != 0)         /* 검사 */
```

**비트 필드**를 쓰면 같은 일을 보통의 멤버처럼 할 수 있습니다.

```c
struct perm_bits {
    unsigned int read  : 1;
    unsigned int write : 1;
    unsigned int exec  : 1;
};
```

콜론 뒤의 숫자가 **그 멤버가 차지할 비트 수**입니다.

```text
[마스크] 읽기=1 쓰기=1 실행=0 (값 = 3)
[마스크] 쓰기를 끈 뒤 값 = 1
[비트필드] 읽기=1 쓰기=1 실행=0
[비트필드] 쓰기를 끈 뒤 쓰기=0
sizeof(struct perm_bits) = 4
```

```c
p.read = 1;         /* 켜기 — 비트 연산이 필요 없다 */
p.write = 0;        /* 끄기 */
if (p.read)         /* 검사 */
```

### 11.2 주의할 점

| 주의 | 설명 |
|---|---|
| 배치가 **환경마다 다르다** | 왼쪽부터 채우는 컴퓨터도, 오른쪽부터 채우는 컴퓨터도 있다 |
| **주소를 얻을 수 없다** | `&p.read`는 불가능하다 |
| 배열로 만들 수 없다 | — |
| 부호를 명시할 것 | 이식성을 위해 `signed`·`unsigned`를 밝힌다 |

> **파일 형식이나 통신 규격처럼 밖에서 정해진 자료를 해석할 때는 비트 필드를 쓰지 마십시오.** 배치가 환경마다 달라 다른 컴퓨터에서 깨집니다. 그럴 때는 3강에서 배운 **마스크와 시프트**를 쓰는 것이 안전합니다. 비트 필드는 **프로그램 내부에서만 쓰는 플래그**에 적합합니다.
{: .prompt-danger }

---

## 제12절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| 구조체 선언에서 컴파일 오류 | 닫는 중괄호 뒤 **세미콜론 누락** | `};` |
| `*p.x` 에서 오류 | `.`이 `*`보다 우선 | `(*p).x` 또는 `p->x` |
| `if (a == b)` 오류 | 구조체는 비교할 수 없다 | 멤버를 하나씩 비교하는 함수를 만든다 |
| 함수에서 고친 값이 반영 안 됨 | 구조체도 **값 전달** | 포인터를 넘기고 `->`로 접근 |
| `sizeof` 결과가 예상과 다름 | **패딩** | 계산하지 말고 `sizeof`로 구한다 |
| 프로그램이 점점 느려지고 메모리를 먹음 | `free` 누락(**메모리 누수**) | 다 쓴 것은 반납, 트리는 자식부터 |
| `free` 후 죽음 | 반납한 자리를 다시 사용 | 반납 후에는 그 포인터를 쓰지 않는다 |
| `malloc` 후 죽음 | `NULL` 검사 누락 | 반드시 검사 |
| 문자열 복사 후 이상한 글자 | `malloc(strlen(s))` — `+1` 누락 | `strlen(s) + 1` |
| 공용체 값이 이상함 | 넣은 멤버와 다른 멤버로 꺼냄 | 태그를 두어 스스로 기록 |
| 다른 컴퓨터에서 비트 필드가 깨짐 | 배치가 환경 의존 | 외부 자료 형식에는 마스크·시프트 |

---

## 실습문제

> **안내**
> 1. 모든 문제는 `C:\c-study\lab08`에서 수행합니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다. **먼저 스스로 작성한 뒤** 확인하시기 바랍니다.
> 3. `malloc`을 쓰는 문제는 **`NULL` 검사와 `free`** 를 반드시 포함해야 합니다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 | K&R |
|---|---|---|---|
| 문제 1 | 구조체로 성적 관리 | 1 · 3 | — |
| 문제 2 | 구조체 배열 정렬 | 4 · 9 | — |
| 문제 3 | 해시 테이블에서 지우기(`undef`) | 8 | 6-5 |
| 문제 4 | 연결 리스트 만들기 | 7 | — |
| 문제 5 | 구조체 크기와 패딩 조사 | 5.2 | — |
| 문제 6 | 단어 빈도 내림차순 출력 | 7 | 6-4 |
| 문제 7 | `typedef`로 선언 정리 | 9 | — |
| 문제 8 | 공용체로 값 저장 | 10 | — |
| 문제 9 | 비트 필드로 권한 관리 | 11 | — |
| 문제 10 | 구조체 비교 함수 만들기 | 1.3 | — |

---

### 문제 1. 구조체로 성적 관리

학생의 학번·이름·세 과목 점수·평균을 담는 구조체를 만들고, 평균을 계산해 표로 출력하십시오.

**정답 및 해설**

```c
#define NAMELEN  20
#define SUBJECTS 3

struct student {
    int  id;
    char name[NAMELEN];
    int  score[SUBJECTS];
    double average;
};

/* 평균을 계산해 채워 넣는다 */
static void set_average(struct student *s)
{
    int i, sum = 0;

    for (i = 0; i < SUBJECTS; i++)
        sum += s->score[i];
    s->average = (double) sum / SUBJECTS;
}

static void print_student(const struct student *s)
{
    int i;

    printf("%4d %-10s", s->id, s->name);
    for (i = 0; i < SUBJECTS; i++)
        printf("%5d", s->score[i]);
    printf("%8.1f\n", s->average);
}
```

```text
학번 이름       국어  영어  수학    평균
   1 kim           90   85  100    91.7
   2 lee           70   95   80    81.7
   3 park          88   76   92    85.3
```

- **`set_average`는 포인터를 받습니다.** 원본을 고쳐야 하기 때문입니다. 값으로 받으면 복사본의 `average`만 바뀌고 끝납니다.
- **`print_student`는 `const` 포인터**를 받습니다. 고치지 않겠다는 약속이며, 실수로 고치는 코드를 컴파일러가 막아 줍니다(3강 4.3절).
- 이름은 `char name[NAMELEN]`처럼 **배열로 두었습니다.** `char *`로 두면 문자열의 저장 공간을 따로 마련해야 합니다(6강 6.2절).
- 이름을 영문으로 쓴 이유는 한글이 터미널에서 두 칸을 차지해 `%-10s` 정렬이 어긋나기 때문입니다.

---

### 문제 2. 구조체 배열 정렬

학생 배열을 **학번 순·평균 내림차순·이름 순**으로 정렬하십시오. 정렬 함수는 하나만 만들고 **비교 함수를 바꿔 끼우는** 방식으로 작성합니다.

**정답 및 해설**

```c
typedef int (*StudentCmp)(const struct student *, const struct student *);

static int by_id(const struct student *a, const struct student *b)
{
    return a->id - b->id;
}

static int by_average_desc(const struct student *a, const struct student *b)
{
    if (a->average < b->average) return 1;      /* 내림차순 */
    if (a->average > b->average) return -1;
    return 0;
}

/* 단순 선택 정렬 — 비교 함수를 바꿔 끼운다 */
static void sort_students(struct student v[], int n, StudentCmp cmp)
{
    int i, j;
    struct student temp;

    for (i = 0; i < n - 1; i++)
        for (j = i + 1; j < n; j++)
            if (cmp(&v[j], &v[i]) < 0) {
                temp = v[i];         /* 구조체는 통째로 대입할 수 있다 */
                v[i] = v[j];
                v[j] = temp;
            }
}
```

```text
[학번 순]
  1 kim      91.7
  2 lee      78.5
  3 park     85.3
[평균 내림차순]
  1 kim      91.7
  3 park     85.3
  2 lee      78.5
[이름 순]
  1 kim      91.7
  2 lee      78.5
  3 park     85.3
```

- 7강의 함수 포인터를 **`typedef`로 정리**했습니다. 선언이 훨씬 읽기 쉬워졌습니다.
- **구조체를 통째로 맞바꿉니다.** `temp = v[i];` 한 줄로 모든 멤버가 복사됩니다. 멤버가 아무리 많아도 코드는 그대로입니다.
- **내림차순은 비교 결과의 부호를 뒤집어** 만듭니다(7강 실습문제 5와 같은 요령).
- 실수를 비교할 때 `a->average - b->average`를 `int`로 돌려주면 **0.5 같은 차이가 0으로 잘립니다.** 그래서 크기를 따져 `1`·`-1`·`0`을 직접 돌려주었습니다.

---

### 문제 3. 해시 테이블에서 지우기

`install`·`lookup`에 짝이 되는 `undef(name)`을 작성하십시오. 표에서 그 이름을 지우고 메모리를 반납해야 합니다.

**정답 및 해설**

```c
/* undef: 표에서 이름을 지운다 */
int undef(const char *name)
{
    unsigned hashval = hash(name);
    struct nlist *np, *prev;

    prev = NULL;
    for (np = hashtab[hashval]; np != NULL; np = np->next) {
        if (strcmp(name, np->name) == 0) {
            if (prev == NULL)
                hashtab[hashval] = np->next;   /* 첫 칸이었다 */
            else
                prev->next = np->next;         /* 앞 칸이 다음 칸을 가리키게 */
            free(np->name);
            free(np->defn);
            free(np);
            return 1;                          /* 지웠다 */
        }
        prev = np;
    }
    return 0;                                  /* 없었다 */
}
```

```text
undef("OUT")     = 1
undef("OUT") 재시도 = 0
lookup("OUT")    -> (없음)
lookup("IN")     -> 1
```

- **연결 리스트에서 원소를 지우려면 앞 칸을 알아야 합니다.** 그래서 `prev`를 함께 따라가게 했습니다.
- **첫 칸을 지우는 경우가 특별합니다.** 앞 칸이 없으므로 `hashtab[hashval]` 자체를 고쳐야 합니다. 이 경우를 빠뜨리는 것이 가장 흔한 실수입니다.
- **`free`는 세 번** 필요합니다. `name`, `defn`, 그리고 칸 자체입니다. 하나라도 빠뜨리면 메모리 누수입니다.
- 반납 **순서**에 주의하십시오. `free(np)`를 먼저 하면 `np->name`을 읽을 수 없게 됩니다.

---

### 문제 4. 연결 리스트 만들기

정수를 담는 연결 리스트를 만들고 **앞에 넣기·값으로 지우기·전체 출력·전체 반납**을 구현하십시오.

**정답 및 해설**

```c
struct node {
    int value;
    struct node *next;
};

/* 맨 앞에 새 마디를 붙인다 */
struct node *push_front(struct node *head, int value)
{
    struct node *p = (struct node *) malloc(sizeof(struct node));

    if (p == NULL)
        return head;                /* 자리가 없으면 그대로 */
    p->value = value;
    p->next = head;
    return p;                       /* 새 머리를 돌려준다 */
}

/* 값이 v 인 첫 마디를 지운다 */
struct node *remove_value(struct node *head, int v)
{
    struct node *p, *prev = NULL;

    for (p = head; p != NULL; prev = p, p = p->next) {
        if (p->value == v) {
            if (prev == NULL)
                head = p->next;
            else
                prev->next = p->next;
            free(p);
            break;
        }
    }
    return head;
}

void free_list(struct node *head)
{
    struct node *p, *next;

    for (p = head; p != NULL; p = next) {
        next = p->next;             /* 지우기 전에 다음을 기억해 둔다 */
        free(p);
    }
}
```

```text
목록: 1 2 3
목록: 1 3
목록: 1 3
```

- **머리(`head`)가 바뀔 수 있으므로 새 머리를 돌려줍니다.** 이 방식이 초심자에게 가장 안전합니다(포인터를 가리키는 포인터를 쓰는 방법도 있습니다).
- **`free_list`에서 `next`를 미리 저장하는 것**이 핵심입니다. `free(p)` 후에는 `p->next`를 읽을 수 없습니다. 이 실수는 대개 **프로그램이 죽어야 발견됩니다.**
- 없는 값을 지우려 해도 아무 일이 일어나지 않아야 합니다. 마지막 출력이 그대로인 것이 그 확인입니다.

---

### 문제 5. 구조체 크기와 패딩 조사

멤버 순서를 바꾸어 가며 `sizeof`를 조사하고, 가장 작은 배치를 찾으십시오.

**정답 및 해설**

```c
struct bad  { char c; int i; char d; };     /* 12바이트 */
struct good { int i; char c; char d; };     /*  8바이트 */
```

```text
멤버 크기의 단순 합 = 6
sizeof(struct bad)  = 12   <- 빈 자리(패딩)가 생긴다
sizeof(struct good) = 8   <- 순서만 바꿔도 줄어든다
```

| `struct bad`의 배치 | 0 | 1~3 | 4~7 | 8 | 9~11 |
|---|---|---|---|---|---|
| 내용 | `c` | 빈 자리 | `i` | `d` | 빈 자리 |

| `struct good`의 배치 | 0~3 | 4 | 5 | 6~7 |
|---|---|---|---|---|
| 내용 | `i` | `c` | `d` | 빈 자리 |

- **규칙은 "큰 것부터 선언한다"** 입니다. 정렬 요구가 큰 멤버를 앞에 두면 중간에 생기는 빈 자리가 줄어듭니다.
- 끝에 남는 빈 자리는 **배열로 만들었을 때 다음 원소도 정렬되도록** 하기 위한 것입니다. 그래서 `struct good`도 6이 아니라 8입니다.
- 메모리가 넉넉한 환경에서는 신경 쓸 필요가 없지만, **구조체를 수만 개 만들거나 임베디드 환경**이라면 큰 차이가 됩니다.

---

### 문제 6. 단어 빈도 내림차순 출력

`tree.c`는 사전 순으로 출력합니다. **빈도가 높은 순서**로 출력하도록 고치십시오.

**정답 및 해설**

트리를 순회하면서 **배열에 모은 뒤 정렬**하는 것이 가장 간단합니다.

```c
struct entry {
    const char *word;
    int count;
};

static struct entry list[MAXENTRY];
static int nentry = 0;

/* 트리를 훑어 배열에 모은다 */
static void collect(const struct tnode *p)
{
    if (p == NULL || nentry >= MAXENTRY)
        return;
    collect(p->left);
    if (nentry < MAXENTRY) {
        list[nentry].word = p->word;
        list[nentry].count = p->count;
        nentry++;
    }
    collect(p->right);
}

static int by_count_desc(const void *a, const void *b)
{
    const struct entry *x = (const struct entry *) a;
    const struct entry *y = (const struct entry *) b;

    return y->count - x->count;       /* 내림차순 */
}
```

그다음 표준 라이브러리의 `qsort`로 정렬합니다.

```c
    collect(root);
    qsort(list, (size_t) nentry, sizeof list[0], by_count_desc);
```

- **트리는 빈도 순으로 정렬되어 있지 않습니다.** 트리의 순서는 단어의 사전 순이므로, 다른 기준으로 보려면 다시 정렬해야 합니다.
- `qsort`는 7강 5.5절에서 소개한 표준 함수입니다. 비교 함수의 인자가 **`const void *`** 이므로 안에서 캐스트해야 합니다.
- 단어 문자열은 **복사하지 않고 포인터만 담았습니다**(7강 1.1절). 트리를 `free` 하기 전에 출력해야 한다는 점에 주의하십시오.

---

### 문제 7. `typedef`로 선언 정리

7강 실습문제 6의 `select_op` 선언을 `typedef`로 정리하십시오.

**정답 및 해설**

```c
/* 7강의 표기 — 읽기 어렵다 */
static double (*select_op(char op))(double, double);
```

```c
/* typedef 를 쓴 표기 */
typedef double (*BinaryOp)(double, double);

static BinaryOp select_op(char op);
```

- **"두 `double`을 받아 `double`을 돌려주는 함수를 가리키는 포인터"** 에 `BinaryOp`라는 이름을 붙였습니다.
- 그러자 `select_op`의 선언이 **"`char`를 받아 `BinaryOp`를 돌려주는 함수"** 라는 한 줄로 읽힙니다.
- 이것이 9.3절에서 말한 **가독성**의 실제 예입니다. 함수 포인터를 쓸 때는 거의 언제나 `typedef`를 함께 쓰십시오.

---

### 문제 8. 공용체로 값 저장

정수·실수·문자열 중 하나를 담을 수 있는 자료형을 만들고, 무엇이 들어 있는지에 따라 알맞게 출력하십시오.

**정답 및 해설**

```c
enum vtype { INT_VAL, FLOAT_VAL, STR_VAL };

struct value {
    enum vtype type;        /* 지금 무엇이 들어 있는지 기억한다 */
    union {
        int ival;
        float fval;
        const char *sval;
    } u;
};

static void print_value(struct value v)
{
    switch (v.type) {
    case INT_VAL:
        printf("정수  : %d\n", v.u.ival);
        break;
    case FLOAT_VAL:
        printf("실수  : %g\n", (double) v.u.fval);
        break;
    case STR_VAL:
        printf("문자열: %s\n", v.u.sval);
        break;
    }
}
```

```text
정수  : 42
실수  : 3.5
문자열: hello
sizeof(공용체) = 8   <- 가장 큰 멤버에 맞춘다
sizeof(struct value) = 16
```

- **태그(`type`)를 반드시 함께 두어야 합니다.** 공용체 자신은 무엇이 들어 있는지 모릅니다.
- 값을 넣을 때 **태그도 함께 갱신**해야 합니다. 한쪽만 바꾸면 그때부터 프로그램이 잘못된 해석을 하게 됩니다.
- `struct value`가 16바이트인 것은 `enum`(4) + 패딩(4) + 공용체(8)이기 때문입니다. 5.2절의 패딩이 여기서도 나타납니다.

---

### 문제 9. 비트 필드로 권한 관리

읽기·쓰기·실행 권한을 비트 필드로 표현하고, 마스크 방식과 비교하십시오.

**정답 및 해설**

```c
struct perm_bits {
    unsigned int read  : 1;
    unsigned int write : 1;
    unsigned int exec  : 1;
};
```

| 하려는 일 | 마스크 방식 | 비트 필드 방식 |
|---|---|---|
| 켜기 | `flags \|= PERM_READ;` | `p.read = 1;` |
| 끄기 | `flags &= ~PERM_WRITE;` | `p.write = 0;` |
| 검사 | `if (flags & PERM_READ)` | `if (p.read)` |
| 여러 개 한꺼번에 | `flags \|= A \| B;` | 한 줄씩 |
| 파일·통신에 저장 | **안전** | **위험**(배치가 환경 의존) |

```text
[마스크] 읽기=1 쓰기=1 실행=0 (값 = 3)
[마스크] 쓰기를 끈 뒤 값 = 1
[비트필드] 읽기=1 쓰기=1 실행=0
[비트필드] 쓰기를 끈 뒤 쓰기=0
sizeof(struct perm_bits) = 4
```

- **비트 필드는 읽기 쉽고, 마스크는 이식성이 좋습니다.**
- 세 개의 1비트 필드인데 크기가 4바이트인 이유는, 컴파일러가 이들을 **하나의 `unsigned int` 안에 담기** 때문입니다.
- 리눅스의 파일 권한(`rwxr-xr-x`)이 바로 이런 비트 묶음이며, 여러분이 이미 리눅스 강의에서 다룬 개념입니다.

---

### 문제 10. 구조체 비교 함수 만들기

`struct point` 두 개가 같은지 비교하는 함수를 작성하고, **왜 `==`를 쓸 수 없는지** 설명하십시오.

**정답 및 해설**

```c
int point_equal(struct point a, struct point b)
{
    return a.x == b.x && a.y == b.y;
}
```

**`==`를 쓸 수 없는 이유**

1. **C 언어가 허용하지 않습니다.** 구조체에 `==`를 쓰면 컴파일 오류입니다.
2. **메모리를 통째로 비교하는 것도 안전하지 않습니다.** 5.2절에서 본 **패딩** 때문입니다. 빈 자리에는 예전에 쓰던 값이 남아 있을 수 있어, 멤버 값이 모두 같아도 바이트가 다를 수 있습니다.

```c
if (memcmp(&a, &b, sizeof a) == 0)     /* 위험한 방법 */
```

- 그러므로 **멤버를 하나씩 비교하는 함수를 만드는 것이 정답**입니다.
- 멤버에 문자열(`char *`)이 있다면 `strcmp`로 비교해야 합니다. `==`로 비교하면 **주소를 비교**하게 됩니다(6강 6.4절).

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 예제 파일 — `point.c`, `pointfn.c`, `ptrstruct.c`, `keycount.c`, `keycount_ptr.c`, `padding.c`, `tree.c`, `hashtab.c`, `typedef_demo.c`, `union_demo.c`, `bitfield_demo.c` |
| 2 | 실습문제 `ex1.c` ~ `ex10.c` |
| 3 | 실행 결과 화면 — 문제 1·3·4·5 |
| 4 | 짧은 서술 ① 구조체를 함수에 **값으로 넘길 때와 포인터로 넘길 때**의 차이와 각각을 언제 쓰는지 |
| 5 | 짧은 서술 ② `sizeof(struct bad)`가 6이 아니라 12인 이유를 그림과 함께 설명 |
| 6 | 짧은 서술 ③ `treefree`에서 자식을 먼저 반납해야 하는 이유 |

---

## 정리

| 구분 | 내용 |
|---|---|
| 구조체 | 서로 다른 자료형을 하나로 묶는다. 선언 끝의 **세미콜론 필수** |
| 접근 | `s.member`, 포인터면 `p->member`(= `(*p).member`) |
| 할 수 있는 일 | 대입·복사, 함수에 전달·반환, 주소 얻기, 멤버 접근 |
| 할 수 없는 일 | **비교(`==`)** — 멤버별 비교 함수를 만든다 |
| 전달 방식 | 값 전달이 기본. 크거나 고쳐야 하면 **포인터** |
| 구조체 배열 | 짝을 이루는 자료는 나란한 배열 대신 구조체 배열로 |
| 크기 | **`sizeof`로 구한다.** 멤버 합 ≠ 구조체 크기(**패딩**) |
| 메모리 | `malloc`으로 빌리고 **`NULL` 검사**, 다 쓰면 **`free`** |
| 자기 참조 | `struct node *next;` — 리스트와 트리의 바탕 |
| 트리 | 넣기·출력·반납이 모두 **재귀**. 반납은 **자식부터** |
| 해시 | 이름 → 자리 번호, 충돌은 **연결 리스트**로 해결 |
| `typedef` | 자료형에 별명. 함수 포인터 선언을 읽기 쉽게 만든다 |
| `union` | 같은 자리를 돌려 쓴다. **태그를 함께 두어야** 한다 |
| 비트 필드 | 내부 플래그에 적합. **외부 자료 형식에는 쓰지 말 것** |

이제 자료를 원하는 모양으로 묶고, 필요한 만큼 메모리를 빌려 쓰며, 리스트와 트리 같은 자료 구조를 만들 수 있습니다. 그런데 지금까지 만든 프로그램은 **실행이 끝나면 모든 것을 잃어버립니다.**

**다음 9강에서는 입출력과 파일(K&R 제7장)** 을 다룹니다. `printf`·`scanf`의 서식을 정확히 익히고, **파일을 열어 읽고 쓰는 방법**, 표준 입출력과 오류 출력의 구분, 그리고 7강에서 배운 명령행 인자를 이용해 **여러 파일을 처리하는 프로그램**을 만듭니다. 이번 강의에서 만든 구조체를 파일에 저장했다가 다시 읽어 오는 일도 그때 할 수 있게 됩니다.

---

## 부록 A. 구조체 표기 요약

| 표기 | 뜻 |
|---|---|
| `struct tag { ... };` | 구조체 모양 선언(저장 공간 없음) |
| `struct tag v;` | 변수 정의 |
| `struct tag v = {1, 2};` | 정의와 초기화 |
| `v.member` | 멤버 접근 |
| `p->member` | 포인터를 통한 멤버 접근 |
| `(*p).member` | 위와 같은 뜻(괄호 필수) |
| `sizeof(struct tag)` | 크기(패딩 포함) |
| `sizeof(*p)` | 가리키는 구조체의 크기 |
| `typedef struct tag { ... } Name;` | 별명 붙이기 |
| `struct node *next;` | 자기 참조(리스트·트리) |

## 부록 B. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `point.c` | 구조체 선언·초기화·중첩 | 1.2 |
| `pointfn.c` | 구조체를 주고받는 함수 | 2.1 |
| `ptrstruct.c` | 포인터와 `->` | 3.2 |
| `keycount.c` | 구조체 배열 + 이진 탐색 | 4.3 |
| `keycount_ptr.c` | 같은 프로그램의 포인터 판 | 5.1 |
| `padding.c` | 패딩과 멤버 순서 | 5.2 |
| `tree.c` | 이진 트리로 단어 세기 | 7.2 |
| `hashtab.c` | 해시 테이블 | 8.3 |
| `typedef_demo.c` | `typedef` | 9.2 |
| `union_demo.c` | 공용체와 태그 | 10.1 |
| `bitfield_demo.c` | 비트 필드와 마스크 비교 | 11.1 |
