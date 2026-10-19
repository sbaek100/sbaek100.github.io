---
title: 중급 C 프로그래밍 1강 - 실습 환경 구축과 시스템 호출의 세계
date: 2026-11-23 09:00:00 +0900
categories:
  - 1.응용강의
  - 중급 C 프로그래밍
tags:
  - C언어
  - 시스템프로그래밍
  - 리눅스
  - Ubuntu2404
  - VirtualBox
  - 시스템호출
  - strace
  - errno
  - netplan
pin:
mermaid: false
---

> **학습 목표**
> 1. 실습용 리눅스 가상 머신 **두 대**를 구성하고 서로 통신시킬 수 있다.
> 2. 고정 IP와 호스트 이름을 설정하고 방화벽을 실습에 맞게 열 수 있다.
> 3. 개발 도구를 설치하고 첫 프로그램을 빌드·실행할 수 있다.
> 4. **사용자 공간과 커널 공간**의 경계를 설명할 수 있다.
> 5. **시스템 호출**과 라이브러리 함수의 차이를 설명할 수 있다.
> 6. `printf` 대신 `write`를 직접 호출할 수 있다.
> 7. `man` 2절을 스스로 찾아 읽을 수 있다.
> 8. 시스템 호출의 실패를 `-1`과 `errno`로 판별하고 보고할 수 있다.
> 9. `strace`로 프로그램이 커널에 무엇을 요청하는지 관찰할 수 있다.
{: .prompt-info }

## 0. 이 과정에 대하여

### 0.1 1부에서 여기까지

「C 프로그래밍 기초」 열두 강의를 마치고 오셨다면, 여러분은 이미 다음을 할 수 있습니다.

| 할 수 있는 것 | 1부의 어디 |
|---|---|
| 포인터와 메모리를 다룬다 | 6·7강 |
| 구조체와 동적 메모리로 자료를 담는다 | 8강 |
| 파일을 읽고 쓴다 | 9강 |
| 바깥 입력을 검증한다 | 10강 |
| 여러 파일로 나누고 라이브러리로 묶는다 | 11강 |
| 완결된 프로그램을 설계한다 | 12강 |

**2부에서 새로 배우는 것은 C 문법이 아닙니다.** 문법은 이미 다 배웠습니다. 이제 배울 것은 **운영체제가 제공하는 인터페이스**입니다.

### 0.2 무엇이 달라지는가

1부의 프로그램은 **혼자 돌아갔습니다.** 화면과 파일 하나를 상대했을 뿐입니다.

| 1부 | 2부 |
|---|---|
| 화면에 출력한다 | **커널에게 요청**해 출력한다 |
| 파일 하나를 읽는다 | 파일 시스템 전체를 다룬다 |
| 함수를 부른다 | **프로세스를 만든다** |
| 프로그램 하나가 돈다 | 여러 프로세스·스레드가 함께 돈다 |
| 입력은 키보드에서 온다 | **입력이 네트워크 저편에서 온다** |
| 내용이 그대로 보인다 | **암호화해서 주고받는다** |

### 0.3 이 과정의 목표

**"프로그램이 실제로 어떻게 동작하는가"를 눈으로 확인하는 것**입니다. 그리고 그 종착점은 **암호화 통신**입니다.

이 과정을 마치면 다음 질문에 스스로 답할 수 있게 됩니다.

- `printf` 한 줄이 화면에 글자를 띄우기까지 무슨 일이 일어나는가
- 터미널에서 명령을 치면 셸은 무엇을 하는가
- 서버 하나가 어떻게 수천 명을 동시에 상대하는가
- **HTTPS의 자물쇠 표시는 정확히 무엇을 보장하는가**
- **암호화된 통신에서 키는 어떻게 서로에게 전달되는가**

마지막 두 질문이 이 과정의 **핵심**입니다. 많은 사람이 "TLS를 쓰면 안전하다"까지는 알지만, **그 안에서 무슨 일이 일어나는지**는 모릅니다. 우리는 패킷을 직접 들여다보고, 암호화 채널을 직접 만들어 보며 확인할 것입니다.

### 0.4 전체 로드맵

1부에서 K&R의 구성을 따랐듯, 2부는 이 분야의 표준 교재인 **APUE**(W. Richard Stevens · Stephen A. Rago, *Advanced Programming in the UNIX Environment*, 3rd Edition, Addison-Wesley, 2013)의 장 구성을 기준으로 삼습니다.

| 부 | 강 | 주제 | APUE 장 | 실습 형태 |
|---|---|---|---|---|
| **1부** | **1강** | **실습 환경 구축과 시스템 호출의 세계** | **1·2장** | **VM 2대** |
| 기초 | 2강 | 저수준 파일 입출력 — 파일 서술자 | 3·5장 | VM 1대 |
| | 3강 | 파일 시스템과 메타데이터 | 4장 | VM 1대 |
| **2부** | 4강 | 프로세스 — `fork`·`exec`·`wait` | 8·9장 | VM 1대 |
| 프로세스 | 5강 | 시그널 | 10장 | VM 1대 |
| | 6강 | 파이프와 프로세스 간 통신 — 작은 셸 만들기 | 15·17장 | VM 1대 |
| **3부** | 7강 | 스레드 — `pthread`와 공유 자료 | 11장 | VM 1대 |
| 동시성 | 8강 | 동시성의 함정 — 교착·재진입·성능 | 12장 | VM 1대 |
| **4부** | 9강 | 소켓 프로그래밍 — 두 대를 잇다 | 16장 | **VM 2대** |
| 네트워크 | 10강 | 다중 접속 서버 — `select`·`poll`·`epoll` | 14·16장 | **VM 2대** |
| | 11강 | UDP와 프로토콜 설계 | 16장 | **VM 2대** |
| **5부** | 12강 | 대칭키 암호 — 직접 만드는 암호 채널 | **—** | **VM 2대** |
| **암호화** | 13강 | 공개키·키 교환·전자서명 | **—** | **VM 2대** |
| **통신** | 14강 | TLS 핸드셰이크 완전 분해 | **—** | **VM 2대** |
| | 15강 | OpenSSL로 TLS 서버·클라이언트 구현 | **—** | **VM 2대** |
| **6부** | 16강 | 커널의 시각 — 관찰 도구 | 7장 | VM 1대 |
| 마무리 | 17강 | 종합 과제 — TLS 동시 접속 서버 | — | **VM 2대** |

**5부 네 강의가 이 과정의 중심**입니다. 앞의 강의들은 모두 그곳에 도달하기 위한 준비입니다.

> **강의 하나는 여러 회차 분량입니다.**
> 한 강의가 대체로 **450~500분**이므로, 각 강의를 **3~4회차**로 나누어 두었습니다. 각 강의 첫머리의 「회차」 표에서 어디까지가 한 번에 할 분량인지 확인하고, 본문의 **▶ 표시**에서 끊어 쉬시기 바랍니다.
> 1부 「C 프로그래밍 기초」도 같은 방식이며 12강이 모두 38회차입니다.
{: .prompt-tip }

### 0.5 교재와 참고 자료

| 자료 | 쓰임 | 주소 |
|---|---|---|
| **APUE** 3판 (Stevens · Rago, 2013) | **기준 교재.** 1~11·16강의 장 대응은 위 표를 따른다 | — (도서) |
| **TLPI** (M. Kerrisk, *The Linux Programming Interface*, 2010) | 참조용. `epoll` 등 **리눅스 고유 기능**은 이쪽이 정확하다 | [man7.org/tlpi](https://man7.org/tlpi/) |
| **Linux man-pages** | `man` 2·3·7절과 같은 내용을 웹에서 | [man7.org/linux/man-pages](https://man7.org/linux/man-pages/) |
| **POSIX Issue 8** (IEEE Std 1003.1-2024) | **표준 원문.** 리눅스 전용인지 구분할 때 | [pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/9799919799/) |

**5부(12~15강)의 APUE 장이 비어 있는 것에 주목하십시오.** APUE와 TLPI는 **암호화를 다루지 않습니다.** 운영체제의 인터페이스와 암호학은 별개의 영역이기 때문입니다. 그래서 이 과정의 핵심부는 별도의 자료를 씁니다.

| 자료 | 쓰이는 곳 | 주소 |
|---|---|---|
| J. Aumasson, *Serious Cryptography* 2판 (2024) | 12·13강 — 대칭키·공개키·해시의 원리 | — (도서) |
| I. Ristić, *Bulletproof TLS and PKI* 2판 (2022) | 14·15강 — 인증서·PKI·운영 | — (도서) |
| **The Illustrated TLS 1.3 Connection** | **14강 — 핸드셰이크를 바이트 단위로.** 무료 | [tls13.xargs.org](https://tls13.xargs.org/) |
| **RFC 9846** — TLS 1.3 (현행) | 14강 — **표준 원문** | [rfc-editor.org/rfc/rfc9846](https://www.rfc-editor.org/rfc/rfc9846) |
| RFC 8446 — TLS 1.3 (2018, 폐기됨) | 참고. 인터넷 자료 대부분이 이 번호를 가리킨다 | [rfc-editor.org/rfc/rfc8446](https://www.rfc-editor.org/rfc/rfc8446) |
| OpenSSL 3.0 공식 문서 · `man 3ssl` | 15강 — API | [docs.openssl.org/3.0](https://docs.openssl.org/3.0/) |

> **표준 문서의 번호는 바뀝니다.**
> TLS 1.3은 2018년 **RFC 8446**으로 나왔지만, 지금은 **RFC 9846**이 이를 대체한 현행 표준입니다. 인터넷의 설명 글 대부분은 아직 8446을 인용하고 있으므로 둘 다 알아 두어야 합니다.
> **자료를 인용할 때는 그것이 아직 유효한지 반드시 확인하십시오.** RFC 편집자 페이지 맨 위에 "Obsoleted by …"가 붙어 있으면 폐기된 문서입니다. 이 습관 하나가 낡은 방식을 그대로 구현하는 사고를 막아 줍니다.
{: .prompt-warning }

> 교재가 없어도 이 과정을 따라올 수 있도록 본문은 스스로 완결되게 썼습니다. 다만 **더 깊이 알고 싶을 때 어디를 펴야 하는지**는 각 강 서두와 마지막 부록에 밝혀 둡니다. 본문에 실린 명령 출력과 측정값은 모두 **Ubuntu 24.04 환경에서 실제로 실행한 결과**이며, 환경에 따라 달라지는 값은 그때마다 표시해 두었습니다.
{: .prompt-info }

### 0.6 이번 강의의 구성

이 강의는 **3회차 분량**(모두 합쳐 약 450분)입니다. 한 번에 끝내려 하지 마시고 **회차 단위로 끊어** 학습하시기 바랍니다.

| 회차 | 범위 | 주제 | 분량 |
|---|---|---|---|
| **1회차** | 제1절 ~ 제6절 | 실습 환경 구축 — VM 두 대 | 180분 |
| **2회차** | 제7절 ~ 제12절 | 시스템 호출의 세계 | 170분 |
| **3회차** | 실습문제 | 스스로 해 보기 | 100분 |

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 왜 리눅스이고 왜 두 대인가 | 20분 |
| 제2절 | 실습 환경 설계 | 20분 |
| 제3절 | 두 번째 VM 만들기 | 40분 |
| 제4절 | 네트워크 설정 | 45분 |
| 제5절 | 개발 도구 설치 | 25분 |
| 제6절 | 두 대가 통하는지 확인 | 30분 |
| 제7절 | 사용자 공간과 커널 공간 | 30분 |
| 제8절 | 첫 시스템 호출 | 35분 |
| 제9절 | `man`을 읽는 법 | 25분 |
| 제10절 | 실패를 다루는 법 — `errno` | 35분 |
| 제11절 | `strace`로 들여다보기 | 30분 |
| 제12절 | 자주 나오는 실수 | 15분 |
| 실습문제 | 스스로 해 보기 (10문항) | 100분 |

---

## 제1절. 왜 리눅스이고 왜 두 대인가

### 1.1 왜 리눅스인가

| 이유 | 설명 |
|---|---|
| **인터페이스가 표준화되어 있다** | POSIX 규격. 윈도우는 전혀 다른 API를 쓴다 |
| **C와 함께 태어났다** | UNIX를 만들기 위해 C가 만들어졌다. K&R 8장이 UNIX 인터페이스인 이유다 |
| **관찰 도구가 갖추어져 있다** | `strace`·`gdb`·`tcpdump`·`/proc` |
| **실무의 표준** | 서버·클라우드·임베디드 대부분 |
| **커널 소스를 읽을 수 있다** | 궁금하면 직접 확인할 수 있다 |

**1부에서 배운 것은 전부 그대로 통합니다.** `gcc`·`make`·헤더·라이브러리·포인터·구조체 — 바뀌는 것은 없습니다. 오히려 C가 원래 있던 자리로 돌아가는 것에 가깝습니다.

### 1.2 왜 두 대인가

통신 프로그램을 한 대에서만 실습하면 **가장 중요한 것을 못 배웁니다.**

| 한 대에서 (`127.0.0.1`) | 두 대에서 |
|---|---|
| 언제나 성공한다 | **방화벽에 막힌다** |
| 주소를 신경 쓸 필요가 없다 | 어느 주소에 붙일지 정해야 한다 |
| 패킷이 밖으로 나가지 않는다 | **패킷을 잡아서 볼 수 있다** |
| 암호화의 필요를 못 느낀다 | **평문이 그대로 보인다** |

마지막 줄이 결정적입니다. 9강에서 우리는 두 VM 사이에 오가는 문자열을 `tcpdump`로 **그대로 읽어 낼 것**입니다. 그 장면을 본 뒤에야 12강부터의 암호화가 왜 필요한지 몸으로 이해하게 됩니다.

> **"실습은 되는데 실제로는 안 되는" 대부분의 원인이 한 대에서만 시험했기 때문입니다.** 방화벽, 바인딩 주소, 이름 해석 — 모두 두 대가 되어야 드러납니다.
{: .prompt-tip }

---

## 제2절. 실습 환경 설계

### 2.1 전제

이 과정은 다음을 **이미 갖추었다고 가정**합니다.

| 항목 | 상태 |
|---|---|
| VirtualBox | 설치되어 있음 |
| Ubuntu Server 24.04 LTS 가상 머신 | **한 대 설치되어 있음** |
| 기본 리눅스 명령 | 「리눅스 기초」 과정 수준 |

VM 설치 절차 자체는 다루지 않습니다. 아직 준비되지 않았다면 「리눅스 기초」 0강을 먼저 진행하십시오.

### 2.2 만들 환경

**기존 VM 한 대를 복제하여 두 대로 만듭니다.**

| 역할 | 호스트 이름 | 호스트 전용 IP | 하는 일 |
|---|---|---|---|
| **서버** | `c-srv` | **192.168.56.60** | 서버 프로그램을 돌린다 |
| **클라이언트** | `c-cli` | **192.168.56.61** | 접속하는 쪽. 패킷도 여기서 잡는다 |
| 윈도우 호스트 | — | 192.168.56.1 | SSH로 두 대에 접속 |

### 2.3 네트워크 구성

각 VM에 **어댑터를 두 개** 붙입니다.

| 어댑터 | 종류 | 이름(Ubuntu 24.04) | 용도 |
|---|---|---|---|
| 어댑터 1 | **NAT** | `enp0s3` | 인터넷(패키지 설치) |
| 어댑터 2 | **호스트 전용 어댑터** | `enp0s8` | **VM끼리·호스트와 통신** |

```text
   [ Windows 호스트 ]  192.168.56.1
            │
   ┌────────┴─────────────────────┐   호스트 전용 네트워크
   │        192.168.56.0/24       │   (VirtualBox Host-Only)
   │                              │
 [ c-srv ]  .60            [ c-cli ]  .61
   enp0s8                    enp0s8
   enp0s3 ── NAT ── 인터넷    enp0s3 ── NAT ── 인터넷
```

> **NAT만으로는 VM끼리 통신할 수 없습니다.**
> VirtualBox의 NAT는 VM마다 독립된 엔진이라 서로를 볼 수 없습니다. VM 사이 통신에는 **호스트 전용 어댑터**가 반드시 필요합니다. 이 과정의 모든 통신 실습은 `192.168.56.0/24`에서 이루어집니다.
{: .prompt-warning }

### 2.4 왜 192.168.56.x 인가

`192.168.56.0/24`는 VirtualBox가 호스트 전용 어댑터에 쓰는 **기본 대역**입니다. 여기에 더해 두 가지 이유가 있습니다.

| 이유 | 설명 |
|---|---|
| **다른 과정과 어긋나지 않는다** | 「리눅스 기초」 과정도 같은 대역을 쓴다 |
| **호스트 운영체제를 가리지 않는다** | 아래 제한 때문 |

> **호스트 운영체제에 따라 제한이 다릅니다.**
> VirtualBox 설명서는 다음과 같이 밝히고 있습니다. "On Linux, macOS and Solaris Oracle VM VirtualBox will only allow IP addresses in 192.168.56.0/21 range to be assigned to host-only adapters." 즉 **리눅스·macOS·Solaris 호스트에서는 `192.168.56.0/21` 밖의 주소를 쓸 수 없고**(관리자가 `/etc/vbox/networks.conf`로 바꿀 수는 있습니다), **Windows 호스트에는 이 제한이 없습니다.**
> 이 과정은 Windows 호스트를 전제하므로 다른 대역도 쓸 수 있지만, **어느 호스트에서도 그대로 통하는 주소**를 쓰기 위해 `192.168.56.x`를 택했습니다.
> 출처: [VirtualBox Manual, Chapter 6: Virtual Networking](https://www.virtualbox.org/manual/ch06.html)
{: .prompt-info }

또한 기본 DHCP 풀(`.101`~`.254`)과 겹치지 않도록 `.60`·`.61`을 씁니다.

### 2.5 계정

| 항목 | 값 |
|---|---|
| 사용자 | `student` |
| 작업 디렉터리 | `~/cmid/labNN` (강의 번호별) |

---

## 제3절. 두 번째 VM 만들기

### 3.1 복제

**VirtualBox 관리자**에서 기존 Ubuntu VM을 오른쪽 클릭하고 **복제**를 선택합니다.

| 설정 | 값 | 이유 |
|---|---|---|
| 이름 | `c-cli` | 클라이언트 역할 |
| MAC 주소 정책 | **모든 네트워크 어댑터의 새 MAC 주소 생성** | 같은 MAC이면 IP가 충돌한다 |
| 복제 방식 | **완전한 복제** | 원본과 독립적으로 동작 |

원본 VM의 이름도 `c-srv`로 바꾸어 둡니다. 이름표를 붙여 두지 않으면 **어느 창이 어느 VM인지 헷갈려** 실습 내내 혼란을 겪습니다.

> **복제 전에 원본 VM은 반드시 종료 상태여야 합니다.** 실행 중이거나 저장된 상태에서 복제하면 복제본의 파일 시스템이 깨질 수 있습니다.
{: .prompt-danger }

### 3.2 어댑터 확인

두 VM 모두 **설정 → 네트워크**에서 다음을 확인합니다.

| 탭 | 설정 |
|---|---|
| 어댑터 1 | 사용함, **NAT** |
| 어댑터 2 | 사용함, **호스트 전용 어댑터**, 이름 `VirtualBox Host-Only Ethernet Adapter` |

어댑터 2가 없다면 **사용함**에 체크하고 종류를 선택하십시오.

### 3.3 두 대를 켜고 로그인

두 VM을 모두 시작하고 `student` 계정으로 로그인합니다. 이 시점에는 **두 대 모두 호스트 이름과 IP가 같습니다.** 다음 절에서 구분해 줍니다.

---

## 제4절. 네트워크 설정

**이 절은 두 VM에서 각각 수행합니다.** 값만 다릅니다.

### 4.1 현재 상태 확인

```bash
ip -brief address
```

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s3           UP             10.0.2.15/24 ...
enp0s8           UP             192.168.56.103/24 ...
```

| 확인할 것 | 뜻 |
|---|---|
| `enp0s3` | NAT. `10.0.2.15`는 VirtualBox가 준 주소 |
| `enp0s8` | 호스트 전용. **DHCP로 아무 주소나 받았다** |

DHCP 주소는 켤 때마다 바뀔 수 있습니다. 서버 프로그램을 실습하려면 **주소가 고정**되어야 합니다.

> 인터페이스 이름이 `enp0s3`·`enp0s8`이 아니라면 위 명령의 실제 출력을 따르십시오. 이후 설정에서 이름을 그대로 바꾸어 적어야 합니다.
{: .prompt-info }

### 4.2 고정 IP 설정 (netplan)

Ubuntu 24.04는 **netplan**으로 네트워크를 관리합니다. 새 설정 파일을 만듭니다.

```bash
sudo nano /etc/netplan/60-lab.yaml
```

**서버(c-srv)에 넣을 내용**

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.60/24
```

**클라이언트(c-cli)에 넣을 내용**

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.61/24
```

> **YAML은 들여쓰기가 문법입니다.**
> 탭 문자를 쓰면 안 되고 **공백만** 사용해야 합니다. 들여쓰기 칸 수가 어긋나면 적용되지 않습니다. 1부 11강에서 `Makefile`은 반드시 탭이어야 했던 것과 정반대이니 주의하십시오.
{: .prompt-danger }

여기에는 `gateway4`도 `nameservers`도 넣지 않았습니다. **인터넷은 NAT 어댑터(`enp0s3`)가 담당**하고, 이 어댑터는 실습망 전용이기 때문입니다.

### 4.3 권한 설정과 적용

```bash
sudo chmod 600 /etc/netplan/60-lab.yaml
```

```bash
sudo netplan apply
```

권한을 600으로 바꾸지 않으면 **설정 파일이 너무 열려 있다(too open)는 경고**가 나옵니다. 동작은 하지만, 네트워크 설정 파일에는 무선 암호 같은 비밀이 들어갈 수 있기 때문입니다. **경고를 남기지 않는 습관**은 1부에서부터 지켜 온 원칙입니다.

YAML 문법만 미리 시험해 볼 수도 있습니다.

```bash
sudo netplan try
```

이 명령은 설정을 적용한 뒤 120초 안에 확인을 받지 못하면 **원래대로 되돌립니다.** 원격으로 접속해 작업하다 네트워크를 끊어 먹는 사고를 막아 줍니다.

> netplan의 전체 문법과 옵션은 공식 문서에 정리되어 있습니다 — [Netplan documentation](https://netplan.readthedocs.io/en/stable/). `man 5 netplan`으로도 볼 수 있습니다.
{: .prompt-info }

### 4.4 적용 확인

```bash
ip -brief address show enp0s8
```

```text
enp0s8           UP             192.168.56.60/24
```

### 4.5 호스트 이름 바꾸기

**서버에서**

```bash
sudo hostnamectl set-hostname c-srv
```

**클라이언트에서**

```bash
sudo hostnamectl set-hostname c-cli
```

새 셸을 열면 프롬프트가 바뀝니다.

```text
student@c-srv:~$
```

### 4.6 이름으로 서로 부르기

IP를 외우는 대신 이름을 씁니다. **두 VM 모두**에서 `/etc/hosts`를 편집합니다.

```bash
sudo nano /etc/hosts
```

다음 두 줄을 추가합니다.

```text
192.168.56.60   c-srv
192.168.56.61   c-cli
```

이제 `ping c-srv`처럼 이름으로 부를 수 있습니다. **앞으로 모든 실습에서 IP 대신 이름을 씁니다.** 나중에 주소가 바뀌어도 이 파일 한 곳만 고치면 됩니다.

### 4.7 방화벽

Ubuntu의 방화벽은 `ufw`입니다. **실습망 전체를 허용**해 둡니다.

```bash
sudo ufw allow from 192.168.56.0/24
```

```bash
sudo ufw enable
```

```bash
sudo ufw status
```

```text
Status: active

To                         Action      From
--                         ------      ----
Anywhere                   ALLOW       192.168.56.0/24
```

> **방화벽은 통신 실습에서 가장 흔한 함정입니다.**
> 9강 이후 "서버는 켰는데 클라이언트가 붙지 못하는" 상황을 만나면 **가장 먼저 `ufw status`를 확인**하십시오. 증상으로 구분할 수 있습니다.
>
> | 증상 | 원인 |
> |---|---|
> | `Connection refused` (즉시) | 그 포트에 **아무도 듣고 있지 않다** |
> | 응답 없이 한참 뒤 `timed out` | **방화벽이 조용히 버렸다** |
{: .prompt-warning }

---

## 제5절. 개발 도구 설치

**두 VM 모두**에서 실행합니다.

```bash
sudo apt update
```

```bash
sudo apt install -y build-essential gdb make strace ltrace tcpdump \
    net-tools iproute2 manpages-dev manpages-posix-dev \
    openssl libssl-dev pkg-config git vim
```

| 꾸러미 | 왜 필요한가 | 처음 쓰는 곳 |
|---|---|---|
| `build-essential` | `gcc`, `make`, 표준 헤더 | 1강 |
| `gdb` | 디버거 | 4강 |
| `strace` | **시스템 호출 추적** | 1강 |
| `ltrace` | 라이브러리 호출 추적 | 16강 |
| `tcpdump` | **패킷 관찰** | 9강 |
| `manpages-dev`·`manpages-posix-dev` | **`man` 2·3절 문서** | 1강 |
| `openssl` | 명령행 암호 도구 | 12강 |
| **`libssl-dev`** | **OpenSSL 개발 헤더** | 12강 |
| `pkg-config` | 컴파일 옵션 조회 | 12강 |

### 5.1 설치 확인

```bash
gcc --version | head -1
```

```text
gcc (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0
```

```bash
openssl version
```

```text
OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024)
```

```bash
man 2 write | head -5
```

```text
write(2)                   System Calls Manual                  write(2)

NAME
       write - write to a file descriptor
```

**`man 2 write`가 나오지 않으면** `manpages-dev`가 설치되지 않은 것입니다. 이 과정 내내 `man` 2절을 읽게 되므로 반드시 확인하십시오.

### 5.2 작업 디렉터리

```bash
mkdir -p ~/cmid/lab01
```

```bash
cd ~/cmid/lab01
```

---

## 제6절. 두 대가 통하는지 확인

### 6.1 ping

**클라이언트에서**

```bash
ping -c 3 c-srv
```

```text
PING c-srv (192.168.56.60) 56(84) bytes of data.
64 bytes from c-srv (192.168.56.60): icmp_seq=1 ttl=64 time=0.412 ms
64 bytes from c-srv (192.168.56.60): icmp_seq=2 ttl=64 time=0.385 ms
64 bytes from c-srv (192.168.56.60): icmp_seq=3 ttl=64 time=0.401 ms

--- c-srv ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2043ms
```

**서버에서도 반대로** 확인합니다.

```bash
ping -c 3 c-cli
```

### 6.2 SSH

**윈도우 호스트의 터미널**에서 두 대에 각각 접속해 봅니다.

```powershell
ssh student@192.168.56.60
```

```powershell
ssh student@192.168.56.61
```

> **VirtualBox 콘솔 창이 아니라 SSH로 실습하십시오.**
> 콘솔 창에서는 복사·붙여넣기와 스크롤이 되지 않아 오타가 늘고 출력을 놓치게 됩니다. 창 두 개를 나란히 띄워 놓고 **왼쪽은 서버, 오른쪽은 클라이언트**로 쓰면 통신 실습이 훨씬 수월합니다.
{: .prompt-tip }

### 6.3 스냅샷

여기까지 되었다면 **두 VM 모두 스냅샷**을 찍어 두십시오.

| 항목 | 값 |
|---|---|
| 이름 | `cmid-base` |
| 설명 | 네트워크·개발 도구 설치 완료 |

실습 중 무언가 크게 잘못되면 이 지점으로 돌아오면 됩니다.

---

> **▶ 여기서부터 2회차 — 시스템 호출의 세계**
> 제7절 ~ 제12절, 약 170분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 제7절. 사용자 공간과 커널 공간

### 7.1 경계

컴퓨터의 메모리와 권한은 **두 세계로 나뉘어** 있습니다.

| 구분 | 사용자 공간(user space) | 커널 공간(kernel space) |
|---|---|---|
| 무엇이 있나 | 우리가 만든 프로그램 | 운영체제의 핵심 |
| 권한 | 제한됨 | 무엇이든 가능 |
| 하드웨어 접근 | **불가** | 가능 |
| 다른 프로세스 메모리 | **볼 수 없음** | 볼 수 있음 |

**우리 프로그램은 디스크에 직접 쓸 수 없습니다.** 화면에 글자를 찍을 수도, 네트워크로 바이트를 보낼 수도 없습니다. 그런 일은 전부 **커널에게 부탁**해야 합니다.

이 부탁이 **시스템 호출(system call)** 입니다.

### 7.2 왜 이렇게 나누어 두었는가

| 이유 | 설명 |
|---|---|
| **보호** | 프로그램 하나가 잘못되어도 시스템 전체가 무너지지 않는다 |
| **격리** | 다른 프로그램의 메모리를 훔쳐볼 수 없다 |
| **중재** | 여러 프로그램이 하나뿐인 디스크·화면을 나누어 쓴다 |
| **추상화** | SSD든 하드디스크든 `write` 하나로 쓴다 |

1부 10강에서 배운 버퍼 오버플로가 **그 프로그램 안에서만** 문제를 일으켰던 이유가 여기 있습니다. 옆 변수는 덮어쓸 수 있어도, **다른 프로세스의 메모리는 건드릴 수 없습니다.** 그 경계를 커널이 지키고 있기 때문입니다.

### 7.3 `printf` 한 줄의 여정

1부 9강에서 "`printf`는 화면이 아니라 스트림에 쓴다"고 배웠습니다. 그 아래를 마저 봅니다.

| 층 | 무슨 일이 일어나는가 | 어느 공간 |
|---|---|---|
| 1 | `printf("%d\n", x)` 호출 | 사용자 |
| 2 | 서식을 해석해 문자열을 만든다 | 사용자 |
| 3 | `stdout` 버퍼에 쌓는다 | 사용자 |
| 4 | 버퍼가 차거나 개행을 만나면 **`write(1, buf, n)`** | 사용자 → **경계** |
| 5 | CPU가 커널 모드로 전환한다 | **전환** |
| 6 | 커널이 터미널 드라이버에 넘긴다 | 커널 |
| 7 | 하드웨어가 글자를 그린다 | 하드웨어 |
| 8 | 사용자 모드로 돌아오며 **쓴 바이트 수**를 돌려준다 | **전환** |

**1부는 1~3층을 배운 것입니다. 2부는 4층부터 시작합니다.**

### 7.4 시스템 호출과 라이브러리 함수

| 구분 | 라이브러리 함수 | 시스템 호출 |
|---|---|---|
| 예 | `printf`, `fopen`, `malloc`, `strlen` | `write`, `open`, `mmap`, `fork` |
| 어디에 있나 | 사용자 공간(libc) | **커널** |
| 문서 | `man 3` | **`man 2`** |
| 비용 | 함수 호출 한 번 | **모드 전환**이 필요해 훨씬 비싸다 |
| 버퍼링 | 있다 | **없다** |

> **시스템 호출은 비쌉니다.**
> 모드를 전환하고 돌아오는 데 수백~수천 나노초가 듭니다. 그래서 표준 라이브러리가 **버퍼를 두어 횟수를 줄이는** 것입니다. `printf`를 백 번 불러도 `write`는 몇 번만 일어납니다. 2강에서 이 차이를 숫자로 확인합니다.
{: .prompt-info }

### 7.5 경계를 아는 것이 왜 중요한가

| 질문 | 답 |
|---|---|
| 왜 `printf` 뒤에 `fflush`가 필요한가 | 아직 4층에 도달하지 않았기 때문 |
| 왜 프로그램이 죽으면 출력이 사라지는가 | 버퍼에만 있고 커널에 넘기지 못해서 |
| 왜 `stderr`는 즉시 보이는가 | 버퍼링하지 않아 바로 `write` 하기 때문 |
| 왜 파일 복사가 느린가 | `read`/`write` 횟수가 많아서(2강) |

**1부 12강에서 `fclose`의 반환값을 검사한 이유**도 이제 정확히 설명됩니다. `fprintf`가 성공한 것은 3층까지의 성공일 뿐이고, 실제로 디스크에 도달했는지는 **버퍼를 비우는 순간**에야 드러납니다.

---

## 제8절. 첫 시스템 호출

### 8.1 `write`를 직접 부른다

`~/cmid/lab01/hello_sys.c`를 만듭니다.

```c
/* hello_sys.c — 표준 라이브러리를 거치지 않고 커널에 직접 요청한다 */
#include <unistd.h>     /* write, STDOUT_FILENO */
#include <string.h>     /* strlen */

int main(void)
{
    const char *msg = "커널에게 직접 부탁했습니다.\n";
    ssize_t n;

    /* printf 가 아니라 write 시스템 호출을 그대로 쓴다 */
    n = write(STDOUT_FILENO, msg, strlen(msg));

    if (n < 0)
        return 1;
    return 0;
}
```

```bash
gcc -Wall -Wextra -std=gnu17 hello_sys.c -o hello_sys
```

```bash
./hello_sys
```

```text
커널에게 직접 부탁했습니다.
```

### 8.2 컴파일 표준이 바뀝니다

1부에서는 `-std=c17`을 썼습니다. **2부에서는 `-std=gnu17`을 씁니다.**

```bash
gcc -Wall -Wextra -std=gnu17 파일.c -o 실행파일
```

이유를 반드시 이해하고 넘어가십시오. 다음 프로그램을 `-std=c17`로 컴파일해 보십시오.

```c
#include <stdio.h>
#include <limits.h>
#include <unistd.h>

int main(void)
{
    char buf[PATH_MAX];        /* 경로의 최대 길이 */
    if (getcwd(buf, sizeof buf) != NULL)
        printf("%s\n", buf);
    return 0;
}
```

```bash
gcc -Wall -Wextra -std=c17 pathtest.c -o pathtest
```

```text
pathtest.c: In function 'main':
pathtest.c:7:14: error: 'PATH_MAX' undeclared (first use in this function)
    7 |     char buf[PATH_MAX];
      |              ^~~~~~~~
```

**`<limits.h>`를 포함했는데도 `PATH_MAX`가 없습니다.**

| 옵션 | 뜻 | POSIX 상수 |
|---|---|---|
| `-std=c17` | **엄격한 ISO C.** `__STRICT_ANSI__`가 정의된다 | **감춰진다** |
| `-std=gnu17` | ISO C + POSIX·GNU 확장 | 보인다 |

C 표준에는 `PATH_MAX`가 없습니다. 그것은 **POSIX의 것**입니다. 엄격한 ISO 모드에서는 표준 밖의 이름을 숨겨, 표준만 쓰는 이식성 있는 코드인지 검사해 줍니다. 그런데 **2부에서 우리가 쓸 것은 거의 전부 POSIX**입니다.

같은 문제는 앞으로 계속 나타납니다.

| 감춰지는 것 | 어디 소속 | 쓰는 곳 |
|---|---|---|
| `PATH_MAX` | POSIX | 1강 |
| `strdup` | POSIX | 3강 |
| `getline` | POSIX | 6강 |
| `struct sockaddr_in` 일부 필드 | POSIX | 9강 |
| `accept4`, `epoll_*` | 리눅스 | 10강 |

> **또 다른 방법도 있습니다.** `-std=c17`을 유지한 채 `-D_GNU_SOURCE`나 `-D_POSIX_C_SOURCE=200809L`을 붙이는 것입니다. 이것을 **기능 시험 매크로(feature test macro)** 라고 하며 [`feature_test_macros(7)`](https://man7.org/linux/man-pages/man7/feature_test_macros.7.html)에 정리되어 있습니다.
> 이 과정에서는 학습 부담을 줄이기 위해 **`-std=gnu17` 한 가지로 통일**합니다. 다만 남의 코드에서 `#define _GNU_SOURCE`가 파일 맨 위에 있는 것을 보게 되면 같은 이유임을 알아 두십시오. 그 문서는 이렇게 못 박고 있습니다. "In order to be effective, a feature test macro **must be defined before including any header files**." **반드시 모든 `#include`보다 앞**이어야 합니다.
{: .prompt-info }

### 8.3 무엇이 새로운가

| 요소 | 설명 |
|---|---|
| `#include <unistd.h>` | POSIX 표준 헤더. `stdio.h`가 아니다 |
| `STDOUT_FILENO` | **1**. 표준 출력의 파일 서술자 |
| `ssize_t` | 부호 있는 크기. 실패를 `-1`로 알려야 하므로 부호가 필요하다 |
| 반환값 | **실제로 쓴 바이트 수.** 요청한 만큼이 아닐 수 있다 |

`size_t`(부호 없음)와 `ssize_t`(부호 있음)의 구분에 주목하십시오. 1부 10강 5.1절에서 부호 혼용의 위험을 배웠는데, **여기서는 부호가 있어야만 하는 이유**가 있습니다. 실패를 `-1`로 표현해야 하기 때문입니다.

### 8.4 세 가지 서술자

프로그램이 시작될 때 **세 개의 파일 서술자가 이미 열려** 있습니다.

| 번호 | 이름 | 상수 | 1부의 이름 |
|---|---|---|---|
| 0 | 표준 입력 | `STDIN_FILENO` | `stdin` |
| 1 | 표준 출력 | `STDOUT_FILENO` | `stdout` |
| 2 | 표준 오류 | `STDERR_FILENO` | `stderr` |

**1부에서 쓰던 `stdout`은 사실 서술자 1을 감싼 껍데기**였습니다. 2강에서 그 관계를 정확히 밝힙니다.

### 8.5 재지정이 왜 되는지

```bash
./hello_sys > out.txt
```

```bash
cat out.txt
```

```text
커널에게 직접 부탁했습니다.
```

프로그램은 **아무것도 바뀌지 않았습니다.** 여전히 서술자 1에 썼을 뿐입니다. 셸이 프로그램을 시작하기 **전에** 서술자 1이 화면 대신 파일을 가리키도록 바꾸어 둔 것입니다.

**이것이 UNIX 설계의 핵심 아이디어입니다.** 6강에서 `dup2`로 이 재지정을 직접 구현하며 작은 셸을 만들게 됩니다.

### 8.6 커널만이 답할 수 있는 것

```c
/* whoami_sys.c — 커널만이 답할 수 있는 것들을 물어본다 */
#include <stdio.h>
#include <unistd.h>
#include <sys/utsname.h>
#include <limits.h>

int main(void)
{
    struct utsname u;
    char cwd[PATH_MAX];

    printf("프로세스 번호(PID)  : %d\n", (int) getpid());
    printf("부모 프로세스(PPID) : %d\n", (int) getppid());
    printf("실제 사용자(UID)    : %d\n", (int) getuid());
    printf("실효 사용자(EUID)   : %d\n", (int) geteuid());

    if (getcwd(cwd, sizeof cwd) != NULL)
        printf("현재 작업 디렉터리  : %s\n", cwd);

    if (uname(&u) == 0) {
        printf("커널 이름           : %s\n", u.sysname);
        printf("커널 판             : %s\n", u.release);
        printf("하드웨어            : %s\n", u.machine);
    }
    return 0;
}
```

```bash
gcc -Wall -Wextra -std=gnu17 whoami_sys.c -o whoami_sys
```

```bash
./whoami_sys
```

```text
프로세스 번호(PID)  : 2841
부모 프로세스(PPID) : 2103
실제 사용자(UID)    : 1000
실효 사용자(EUID)   : 1000
현재 작업 디렉터리  : /home/student/cmid/lab01
커널 이름           : Linux
커널 판             : 6.8.0-51-generic
하드웨어            : x86_64
```

> PID는 실행할 때마다 달라집니다. 위 값은 **예시**입니다. 커널 판(`uname -r`)도 갱신 상태에 따라 다를 수 있습니다.
{: .prompt-info }

| 정보 | 왜 커널만 아는가 |
|---|---|
| PID | 커널이 프로세스를 만들며 부여했다 |
| PPID | **부모가 있다** — 4강의 주제 |
| UID/EUID | 권한 판정의 근거. 3강에서 다시 |
| 작업 디렉터리 | 프로세스마다 따로 가진 상태 |

`getpid`·`getuid`는 인자도 없고 실패하지도 않습니다. **커널에 저장된 값을 그대로 읽어 오는 것**뿐이기 때문입니다.

여기서 **PPID**를 눈여겨보십시오. 우리 프로그램에는 부모가 있습니다. 셸입니다. 그리고 셸에게도 부모가 있습니다. **프로세스는 나무 구조를 이룹니다.** 4강에서 이 나무를 직접 자라게 할 것입니다.

---

## 제9절. `man`을 읽는 법

2부에서 가장 자주 쓸 도구는 컴파일러가 아니라 **`man`** 입니다.

### 9.1 절 번호

```bash
man 2 write
```

| 절 | 내용 | 예 |
|---|---|---|
| 1 | 사용자 명령 | `man 1 ls` |
| **2** | **시스템 호출** | `man 2 write` |
| **3** | **라이브러리 함수** | `man 3 printf` |
| 5 | 파일 형식 | `man 5 hosts` |
| 7 | 개념·규약 | `man 7 signal`, `man 7 socket` |

**절 번호를 빼면 낮은 번호가 먼저 나옵니다.** `man write`는 1절의 `write` 명령(다른 사용자에게 메시지 보내기)을 보여 주므로, 시스템 호출을 볼 때는 **반드시 `man 2`** 라고 적으십시오.

### 9.2 읽는 순서

`man 2 write`의 구조입니다.

| 항목 | 무엇을 보나 |
|---|---|
| **SYNOPSIS** | 필요한 헤더와 함수 원형. **가장 먼저 본다** |
| DESCRIPTION | 하는 일 |
| **RETURN VALUE** | 성공·실패를 어떻게 알리나. **두 번째로 본다** |
| **ERRORS** | `errno`에 어떤 값이 들어오나 |
| NOTES | 함정과 주의사항. 여기에 중요한 것이 많다 |
| SEE ALSO | 관련 함수 |

```text
SYNOPSIS
       #include <unistd.h>

       ssize_t write(int fd, const void buf[.count], size_t count);

RETURN VALUE
       On success, the number of bytes written is returned.  On error, -1
       is returned, and errno is set to indicate the error.
```

**이 두 부분만 정확히 읽어도 대부분의 시스템 호출을 쓸 수 있습니다.**

### 9.3 반드시 기억할 규약

> **시스템 호출은 실패를 `-1`로, 이유를 `errno`로 알린다.**

| 반환값 | 뜻 |
|---|---|
| `-1` | **실패.** `errno`를 보라 |
| 0 이상 | 성공(의미는 함수마다 다르다) |

예외도 있습니다. `mmap`은 실패 시 `MAP_FAILED`를, `fopen` 같은 라이브러리 함수는 `NULL`을 돌려줍니다. **그래서 `man`의 RETURN VALUE를 매번 확인해야 합니다.**

### 9.4 그 밖의 검색

```bash
man -k socket
```

```bash
man 7 socket
```

`man -k`(= `apropos`)는 이름을 모를 때 유용합니다. `man 7`의 개념 문서들은 특히 네트워크 부분에서 큰 도움이 됩니다.

### 9.5 웹에서 같은 문서 보기

VM 밖에서 찾아볼 때는 다음 두 곳을 쓰십시오. **둘 다 원본 문서이며 내용이 같습니다.**

| 자료 | 성격 | 주소 |
|---|---|---|
| **Linux man-pages** | 리눅스 구현 기준. `man`과 같은 내용 | [man7.org/linux/man-pages](https://man7.org/linux/man-pages/) |
| **POSIX (Issue 8)** | **표준 규격 원문.** 리눅스 고유 동작과 표준을 구분할 때 | [pubs.opengroup.org Issue 8](https://pubs.opengroup.org/onlinepubs/9799919799/) |

예를 들어 `write`는 다음 두 문서를 나란히 볼 수 있습니다.

| 문서 | 주소 |
|---|---|
| `write(2)` — 리눅스 | [man7.org/…/write.2.html](https://man7.org/linux/man-pages/man2/write.2.html) |
| `write()` — POSIX Issue 8 | [pubs.opengroup.org/…/write.html](https://pubs.opengroup.org/onlinepubs/9799919799/functions/write.html) |

> **둘을 구분하는 습관이 중요합니다.**
> POSIX에 있으면 다른 UNIX 계열에서도 통하고, 리눅스 man 페이지에만 있으면 **리눅스 전용**입니다. 10강에서 배울 `epoll`이 대표적인 리눅스 전용 기능입니다. 이식성이 필요한 코드를 쓸 때 이 구분이 곧 판단 근거가 됩니다.
{: .prompt-tip }

---

## 제10절. 실패를 다루는 법 — `errno`

### 10.1 `errno`란

시스템 호출이 실패하면 커널은 `-1`을 돌려주고, **실패한 이유를 `errno`라는 전역 변수에 적어 둡니다.**

```c
#include <errno.h>
```

| 값 | 이름 | `strerror`가 주는 설명 | 나오는 곳 |
|---|---|---|---|
| 2 | `ENOENT` | No such file or directory | 파일이 없다 |
| 13 | `EACCES` | Permission denied | 권한이 없다 |
| 9 | `EBADF` | Bad file descriptor | 잘못된 서술자 |
| 4 | `EINTR` | Interrupted system call | 시그널에 중단됨(5강) |
| 11 | `EAGAIN` | Resource temporarily unavailable | 지금은 안 된다(10강) |
| 17 | `EEXIST` | File exists | 이미 있다(2강) |
| 24 | `EMFILE` | Too many open files | 서술자 한계(2강) |
| 32 | `EPIPE` | Broken pipe | 받는 쪽이 사라졌다(6강) |
| 111 | `ECONNREFUSED` | Connection refused | 연결 거부(9강) |

> **숫자는 시스템마다 다를 수 있습니다.** 위 값은 Linux x86-64 기준이며, 코드에는 반드시 **숫자가 아니라 이름**을 쓰십시오. 전체 목록은 [`errno(3)`](https://man7.org/linux/man-pages/man3/errno.3.html)에 있고, 자기 환경의 값은 `errno -l`(moreutils) 또는 직접 출력해 확인할 수 있습니다.
{: .prompt-warning }

### 10.2 세 가지 규칙

> **① 반환값이 실패를 가리킬 때만 `errno`를 본다.**
> 성공한 호출은 `errno`를 **지우지 않습니다.** 예전 실패의 흔적이 남아 있을 수 있습니다.
>
> **② 실패를 확인한 직후에 본다.**
> 그사이에 다른 함수를 부르면 `errno`가 덮어써질 수 있습니다. `printf`조차 바꿀 수 있습니다.
>
> **③ 필요하면 미리 지운다.**
> `errno = 0;`
{: .prompt-warning }

### 10.3 확인해 보기

```c
/* err_demo.c — 시스템 호출이 실패했을 때 무엇을 보아야 하는가 */
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>          /* errno */
#include <string.h>         /* strerror */
#include <fcntl.h>          /* open */
#include <unistd.h>         /* close */

int main(int argc, char *argv[])
{
    const char *path = (argc > 1) ? argv[1] : "/그런파일은없다";
    int fd;

    errno = 0;                              /* 호출 직전에 반드시 지운다 */
    fd = open(path, O_RDONLY);

    if (fd == -1) {                         /* 실패는 -1 로 알린다 */
        printf("open 실패\n");
        printf("  errno 값     : %d\n", errno);
        printf("  strerror     : %s\n", strerror(errno));
        perror("  perror       ");          /* 앞말 + ": " + 설명 */
        return 1;
    }

    printf("open 성공, 파일 서술자 = %d\n", fd);
    close(fd);
    return 0;
}
```

```bash
gcc -Wall -Wextra -std=gnu17 err_demo.c -o err_demo
```

```bash
./err_demo
```

```text
open 실패
  errno 값     : 2
  strerror     : No such file or directory
  perror       : No such file or directory
```

```bash
./err_demo /etc/hostname
```

```text
open 성공, 파일 서술자 = 3
```

```bash
./err_demo /etc/shadow
```

```text
open 실패
  errno 값     : 13
  strerror     : Permission denied
  perror       : Permission denied
```

**같은 `open`이 상황에 따라 다른 `errno`를 남깁니다.** "열리지 않았다"까지만 아는 것과 "권한이 없어서"까지 아는 것은 전혀 다릅니다.

### 10.4 서술자가 왜 3인가

`open`이 성공했을 때 돌려준 값이 **3**이었습니다. 0·1·2는 이미 표준 입출력이 쓰고 있기 때문입니다.

> **커널은 언제나 "쓸 수 있는 가장 작은 번호"를 줍니다.** 이 규칙은 6강에서 파이프로 재지정을 구현할 때 결정적으로 쓰입니다.
{: .prompt-tip }

### 10.5 보고하는 방법 세 가지

| 방법 | 특징 |
|---|---|
| `perror("앞말")` | 가장 간단. **`stderr`로** 나간다 |
| `strerror(errno)` | 문자열을 얻어 원하는 대로 조합 |
| `fprintf(stderr, "%s: %s\n", path, strerror(errno))` | **가장 실용적** — 무엇이 실패했는지까지 |

실무에서는 세 번째를 씁니다. `perror("open")`만으로는 **어느 파일을 열다 실패했는지** 알 수 없기 때문입니다.

```c
if (fd == -1) {
    fprintf(stderr, "%s: %s\n", path, strerror(errno));
    return 1;
}
```

> **출력 순서가 뒤섞여 보인다면** 버퍼링 때문입니다. `stdout`은 화면에서는 줄 단위, 파일로 재지정하면 블록 단위로 버퍼링되는 반면 `stderr`는 버퍼링하지 않습니다. 그래서 `./err_demo > out.txt` 처럼 재지정하면 `perror`의 출력이 먼저 화면에 나타납니다. 1부 9강에서 배운 버퍼링이 그대로 드러나는 장면입니다.
{: .prompt-info }

### 10.6 `errno`는 사실 변수가 아니다

`errno`는 전역 변수처럼 보이지만, 실제로는 **스레드마다 따로 존재**하도록 매크로로 구현되어 있습니다.

만약 진짜 전역 변수였다면, 7강에서 배울 여러 스레드가 동시에 시스템 호출을 할 때 서로의 오류 번호를 덮어써 버렸을 것입니다. **이 설계 하나가 스레드 안전성의 좋은 예**입니다.

---

## 제11절. `strace`로 들여다보기

### 11.1 프로그램의 부탁을 엿듣는다

`strace`는 프로그램이 **커널에 무엇을 요청하는지** 모두 보여 줍니다.

```bash
strace ./hello_sys
```

```text
execve("./hello_sys", ["./hello_sys"], 0x7ffdf28af6b0 /* 22 vars */) = 0
brk(NULL)                               = 0x60f9c158c000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7b8f484d9000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=19499, ...}) = 0
mmap(NULL, 19499, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7b8f484d4000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220\243\2\0\0\0\0\0"..., 832) = 832
...
write(1, "\354\273\244\353\204\220\354\227\220\352\262\214 \354\247\201\354\240\221 \353\266\200\355\203\201\355\226\210\354\212\265"..., 40) = 40
+++ exited with 0 +++
```

> 주소값(`0x7ffd...`)과 환경 변수 개수는 실행할 때마다 다릅니다. **바뀌지 않는 것은 호출의 종류와 순서**입니다.
{: .prompt-info }

처음 보면 압도적이지만, **읽는 요령이 있습니다.**

| 부분 | 무엇인가 |
|---|---|
| `execve` | 프로그램이 시작되는 순간 |
| `openat`·`mmap` 여러 줄 | **동적 라이브러리(libc)를 불러오는 과정** |
| **가운데** | 우리가 짠 코드가 하는 일 |
| `exit_group` | 종료 |

### 11.2 관심 있는 것만 보기

```bash
strace -e trace=write ./hello_sys
```

```text
write(1, "\354\273\244\353\204\220\354\227\220\352\262\214 \354\247\201\354\240\221 \353\266\200\355\203\201\355\226\210\354\212\265"..., 40커널에게 직접 부탁했습니다.
) = 40
+++ exited with 0 +++
```

**`write`가 정확히 한 번** 일어났고, **40바이트**를 썼으며, 반환값도 40입니다.

처음 보면 출력이 뒤엉켜 보입니다. 두 가지가 섞여 있기 때문입니다.

| 보이는 것 | 정체 |
|---|---|
| `\354\273\244…` | **strace가 보여 주는 바이트**를 8진수로 표기한 것 |
| `커널에게 직접 부탁했습니다.` | **프로그램 자신의 출력**이 같은 화면에 끼어든 것 |

`\354\273\244`는 `커` 한 글자입니다. 직접 확인해 보십시오.

```bash
printf '커널에게 직접 부탁했습니다.\n' | od -b | head -2
```

```text
0000000 354 273 244 353 204 220 354 227 220 352 262 214 040 354 247 201
0000020 354 240 221 040 353 266 200 355 203 201 355 226 210 354 212 265
```

> **한글 한 글자는 UTF-8에서 3바이트입니다.** 눈에 보이는 글자 수(13자)와 바이트 수(40)가 다릅니다. 이 사실은 9강에서 네트워크로 문자열을 보낼 때 결정적으로 중요해집니다. strace가 문자열을 32바이트까지만 보여 주고 `...`로 줄인 것도 확인해 두십시오.
{: .prompt-info }

### 11.3 버퍼링을 세어 보는 법

표준 라이브러리는 출력 대상에 따라 **버퍼링 방식을 스스로 정합니다.**

| 대상 | 버퍼링 방식 | `write`가 일어나는 시점 |
|---|---|---|
| 터미널(화면) | **줄 단위** | 개행을 만날 때마다 |
| 파일·파이프 | **블록 단위**(보통 4096바이트) | 버퍼가 찰 때, 그리고 종료할 때 |
| `stderr` | **버퍼링 없음** | 호출할 때마다 즉시 |

그러므로 같은 프로그램이라도 **재지정 여부에 따라 시스템 호출 횟수가 크게 달라집니다.** 실습문제 6에서 `strace -c`로 직접 세어 확인합니다.

**이 차이가 7.4절의 "시스템 호출은 비싸다"를 눈으로 보여 줍니다.**

### 11.4 자주 쓰는 옵션

| 옵션 | 하는 일 |
|---|---|
| `-e trace=write,read` | 특정 호출만 |
| `-e trace=network` | 네트워크 관련 전부(9강) |
| `-c` | **횟수와 시간을 요약** |
| `-f` | 자식 프로세스까지 추적(4강) |
| `-o 파일` | 파일로 저장 |
| `-p PID` | **이미 돌고 있는 프로세스**에 붙는다 |

```bash
strace -c ./hello_sys
```

```text
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  0.00    0.000000           0         1           read
  0.00    0.000000           0         1           write
  0.00    0.000000           0         2           close
  0.00    0.000000           0         2           fstat
  0.00    0.000000           0         8           mmap
  0.00    0.000000           0         3           mprotect
  0.00    0.000000           0         1           munmap
  0.00    0.000000           0         1           brk
  0.00    0.000000           0         2           pread64
  0.00    0.000000           0         1         1 access
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         2           openat
  0.00    0.000000           0         1           set_robust_list
  0.00    0.000000           0         1           prlimit64
  0.00    0.000000           0         1           rseq
------ ----------- ----------- --------- --------- ----------------
100.00    0.000000           0        30         1 total
```

**세 줄만 읽으면 됩니다.**

| 항목 | 값 | 뜻 |
|---|---|---|
| `write` | **1회** | 우리가 짠 코드가 한 일 |
| `total` | **30회** | 프로그램 전체 |
| `errors` | 1 | `access("/etc/ld.so.preload")` 실패 — **정상이다** |

우리 코드는 `write` **한 번**만 요청했는데 실제로는 30번의 시스템 호출이 일어났습니다. 나머지 29번은 **동적 라이브러리(libc)를 찾아 메모리에 올리는 과정**입니다. `/etc/ld.so.preload`가 없어서 나는 오류 1건도 정상적인 탐색 과정입니다.

> 총 횟수는 배포판·libc 판에 따라 다릅니다. 그러나 **`write`가 1회**라는 것은 우리 코드가 결정한 값이므로 어디서나 같습니다.
{: .prompt-info }

**`strace`는 2부 내내 쓸 도구입니다.** 프로그램이 왜 멈춰 있는지, 왜 파일을 못 찾는지, 어디서 실패했는지 — 대부분 여기서 답이 나옵니다.

---

## 제12절. 자주 나오는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| `netplan apply` 후에도 IP가 그대로 | YAML 들여쓰기 오류 | 공백만 사용, `sudo netplan try`로 확인 |
| `Permissions ... are too open` 경고 | 설정 파일 권한 | `sudo chmod 600` |
| 두 VM의 IP가 같다 | 복제 시 MAC 재생성 안 함 | 어댑터 MAC 새로 생성 후 재설정 |
| `ping c-srv` 실패 | `/etc/hosts` 미등록 | 두 대 모두에 등록 |
| ping은 되는데 접속이 안 됨 | 방화벽 | `sudo ufw status`, 실습망 허용 |
| `man 2 write` 없음 | `manpages-dev` 미설치 | `sudo apt install manpages-dev` |
| `implicit declaration of function 'write'` | `<unistd.h>` 누락 | 헤더 추가 |
| `errno`가 엉뚱한 값 | 성공한 호출 뒤에 읽음 | **실패를 확인한 뒤에만** 읽는다 |
| `perror` 출력이 먼저 나옴 | 버퍼링 차이 | 정상. 10.5절 참고 |
| `write` 반환값이 요청보다 작음 | **정상 동작** | 2강에서 다룬다 |
| `'PATH_MAX' undeclared` | `-std=c17`이 POSIX를 감춤 | **`-std=gnu17`** 사용(8.2절) |
| `implicit declaration of 'strdup'` | 같은 이유 | `-std=gnu17` |

---

> **▶ 여기서부터 3회차 — 스스로 해 보기**
> 실습문제, 약 100분입니다. 앞 회차와 이어지므로, 여기서 쉬었다가 다시 시작해도 좋습니다.
{: .prompt-info }

## 실습문제

> **안내**
> 1. 문제 1~4는 환경 구축, 5~10은 프로그래밍입니다.
> 2. 정답과 해설은 각 문제 바로 아래에 이어집니다.
> 3. 컴파일은 **`gcc -Wall -Wextra -std=gnu17`** 로 하고, **경고 0개**여야 합니다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 |
|---|---|---|
| 문제 1 | VM 2대 구성 확인 | 3 · 4 |
| 문제 2 | 네트워크 확인 보고 | 4 · 6 |
| 문제 3 | 방화벽 동작 확인 | 4.7 |
| 문제 4 | 개발 도구 확인 | 5 |
| 문제 5 | `write`로 출력하기 | 8.1 |
| 문제 6 | `strace`로 `printf`와 비교 | 11.3 |
| 문제 7 | `errno` 세 가지 상황 | 10 |
| 문제 8 | 오류 보고 함수 만들기 | 10.5 |
| 문제 9 | `man` 읽고 요약하기 | 9 |
| 문제 10 | 두 VM에서 같은 프로그램 실행 | 6 · 8 |

---

### 문제 1. VM 2대 구성 확인

두 VM에서 다음을 실행하고 결과를 표로 정리하십시오.

```bash
hostname; ip -brief address show enp0s8; id
```

**정답 및 해설**

| 항목 | c-srv | c-cli |
|---|---|---|
| `hostname` | `c-srv` | `c-cli` |
| `enp0s8` | `192.168.56.60/24` | `192.168.56.61/24` |
| `id` | `uid=1000(student) ...` | `uid=1000(student) ...` |

- **호스트 이름이 둘 다 같다면** `hostnamectl set-hostname`을 하지 않은 것입니다.
- **IP가 `192.168.56.10x`처럼 보인다면** DHCP 주소를 그대로 쓰고 있는 것입니다. netplan 파일이 적용되지 않았습니다.
- `uid`가 두 대 모두 1000인 것은 정상입니다. 복제본이므로 계정이 같습니다. **UID는 시스템 안에서만 유일**하며, 서로 다른 컴퓨터끼리 같아도 문제가 되지 않습니다.

---

### 문제 2. 네트워크 확인 보고

양방향 `ping`과 SSH 접속을 확인하고 결과를 보고하십시오.

**정답 및 해설**

| 방향 | 명령 | 기대 |
|---|---|---|
| c-cli → c-srv | `ping -c 3 c-srv` | 0% packet loss |
| c-srv → c-cli | `ping -c 3 c-cli` | 0% packet loss |
| 호스트 → c-srv | `ssh student@192.168.56.60` | 로그인 |
| 호스트 → c-cli | `ssh student@192.168.56.61` | 로그인 |

- **한쪽 방향만 된다면** 대개 방화벽입니다. `ping`은 ICMP를 쓰므로, `ufw`가 실습망 전체를 허용하고 있는지 확인하십시오.
- **이름으로는 안 되고 IP로는 된다면** `/etc/hosts` 문제입니다. 이 구분을 스스로 할 수 있어야 합니다. 앞으로 "안 된다"는 상황에서 **어느 층에서 막혔는지** 좁히는 것이 곧 실력입니다.

---

### 문제 3. 방화벽 동작 확인

방화벽을 잠시 막아 두고, 통신이 **어떻게 실패하는지** 관찰하십시오.

**정답 및 해설**

서버에서 임시로 규칙을 넣습니다.

```bash
sudo ufw deny from 192.168.56.61
```

클라이언트에서

```bash
ping -c 3 c-srv
```

```text
--- c-srv ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2039ms
```

원상 복구합니다.

```bash
sudo ufw delete deny from 192.168.56.61
```

- **응답이 없는 채로 조용히 실패**하는 것이 방화벽 차단의 특징입니다. "거부되었다"는 메시지조차 오지 않습니다.
- `ufw deny` 대신 `ufw reject`를 쓰면 거부 응답을 보내므로 **즉시 실패**합니다. 이 차이가 9강의 `Connection refused`와 `timed out`의 차이로 이어집니다.
- 실습이 끝나면 **반드시 규칙을 지우십시오.** 남겨 두면 다음 강의에서 원인 모를 고생을 하게 됩니다.

---

### 문제 4. 개발 도구 확인

필요한 도구가 모두 설치되었는지 확인하십시오.

**정답 및 해설**

```bash
gcc --version | head -1
make --version | head -1
strace -V | head -1
openssl version
man 2 write > /dev/null && echo "man 2 OK"
ls /usr/include/openssl/ssl.h
```

```text
gcc (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0
GNU Make 4.3
strace -- version 6.8
OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024)
man 2 OK
/usr/include/openssl/ssl.h
```

> 위 판 번호는 **Ubuntu 24.04.3 LTS 기준**입니다. 갱신 상태에 따라 다를 수 있으므로 **숫자가 아니라 "나오는가"** 를 확인하십시오. `gdb --version`·`tcpdump --version`도 같은 방식으로 확인합니다.
{: .prompt-info }

**개발 헤더까지 되는지**는 직접 컴파일해 보는 것이 확실합니다.

```c
/* ossl_check.c — OpenSSL 개발 환경이 갖추어졌는지 확인한다 */
#include <stdio.h>
#include <openssl/opensslv.h>
#include <openssl/crypto.h>
#include <openssl/evp.h>
#include <openssl/ssl.h>

int main(void)
{
    printf("헤더 판  : %s\n", OPENSSL_VERSION_TEXT);
    printf("라이브러리: %s\n", OpenSSL_version(OPENSSL_VERSION));
    printf("AES-256-GCM 사용 가능: %s\n",
           EVP_get_cipherbyname("aes-256-gcm") != NULL ? "예" : "아니오");
    printf("TLS 메서드 사용 가능  : %s\n",
           TLS_client_method() != NULL ? "예" : "아니오");
    return 0;
}
```

```bash
gcc -Wall -Wextra -std=gnu17 ossl_check.c -o ossl_check -lssl -lcrypto
```

```bash
./ossl_check
```

```text
헤더 판  : OpenSSL 3.0.13 30 Jan 2024
라이브러리: OpenSSL 3.0.13 30 Jan 2024
AES-256-GCM 사용 가능: 예
TLS 메서드 사용 가능  : 예
```

- **`openssl version`이 되는 것과 위 프로그램이 컴파일되는 것은 다릅니다.** 앞은 **명령 도구**, 뒤는 **개발 헤더와 라이브러리**가 있다는 뜻입니다. `libssl-dev`가 없으면 `openssl/ssl.h: No such file or directory`로 실패합니다.
- **지금 확인해 두어야 합니다.** 12강에 가서야 없다는 것을 알면 그때 다시 인터넷 연결부터 손대야 합니다.
- `-lssl -lcrypto`가 붙은 이유는 1부 11강에서 배운 그대로입니다. 헤더는 `-I`, 라이브러리는 `-l`입니다.
- 12강에서 쓸 AES-256-GCM과 15강에서 쓸 TLS 기능이 실제로 들어 있는지까지 확인했습니다.

---

### 문제 5. `write`로 출력하기

`printf`를 전혀 쓰지 않고, 명령행 인자를 한 줄에 하나씩 출력하는 프로그램을 만드십시오.

**정답 및 해설**

```c
/* argwrite.c — printf 없이 인자를 출력한다 */
#include <unistd.h>
#include <string.h>

int main(int argc, char *argv[])
{
    int i;

    for (i = 1; i < argc; i++) {
        write(STDOUT_FILENO, argv[i], strlen(argv[i]));
        write(STDOUT_FILENO, "\n", 1);
    }
    return 0;
}
```

```bash
./argwrite 하나 둘 셋
```

```text
하나
둘
셋
```

- **개행도 직접 써야 합니다.** `write`는 아무것도 덧붙여 주지 않습니다.
- `strlen`으로 길이를 구한 것에 주의하십시오. `write`는 **길이를 반드시 알려 주어야** 합니다. 1부 10강의 "크기를 아는 자가 안전하다"가 여기서도 그대로 적용됩니다.
- `write`를 두 번 부르는 대신 버퍼에 모아 한 번에 쓰면 시스템 호출이 절반으로 줍니다. **그것이 바로 `printf`가 하는 일**입니다.

---

### 문제 6. `strace`로 `printf`와 비교

같은 내용을 1000줄 출력하는 프로그램을 만들고, **출력 대상에 따라 `write` 호출 횟수가 어떻게 달라지는지** 세어 표로 만드십시오.

**정답 및 해설**

```c
/* many_out.c — 버퍼링의 차이를 세어 본다
   사용법: ./many_out [줄수] [err]   (기본 1000줄, 두 번째 인자가 있으면 stderr 로) */
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[])
{
    long n = (argc > 1) ? strtol(argv[1], NULL, 10) : 1000;
    int to_err = (argc > 2);
    long i;

    for (i = 0; i < n; i++) {
        if (to_err)
            fprintf(stderr, "%04ld 번째 줄입니다.\n", i);
        else
            printf("%04ld 번째 줄입니다.\n", i);
    }
    return 0;
}
```

세 가지 경우를 각각 셉니다.

```bash
strace -c -e trace=write ./many_out 1000 2>&1 >/dev/null | tail -4
```

```bash
strace -c -e trace=write -o st.txt ./many_out 1000 > out.txt ; tail -4 st.txt
```

```bash
strace -c -e trace=write -o st2.txt ./many_out 1000 err 2> err.txt ; tail -4 st2.txt
```

먼저 자료의 크기를 확인해 둡니다.

```bash
head -1 out.txt | wc -c ; wc -c < out.txt
```

```text
26
26000
```

실측 결과입니다.

```text
--- (1) printf 를 터미널로 ---
100.00    0.010322          10      1000           write
--- (2) printf 를 파일로 재지정 ---
100.00    0.000081          11         7           write
--- (3) fprintf(stderr, ...) ---
100.00    0.006308           6      1000           write
```

| 경우 | 버퍼링 | `write` 횟수 |
|---|---|---|
| `printf` → **화면** | 줄 단위 | **1000회** (줄마다 한 번) |
| `printf` → **파일 재지정** | 블록 단위(4096) | **7회** (26000 ÷ 4096 ≈ 6.35 → 6회 + 나머지 1회) |
| `fprintf(stderr, …)` | 없음 | **1000회** (호출마다 한 번) |

**7회라는 값을 계산으로 미리 맞힐 수 있었다**는 점이 중요합니다. 전체 26000바이트를 블록 크기 4096으로 나눈 결과입니다. 블록 크기는 파일 시스템이 알려 주는 값이며 직접 확인할 수 있습니다.

```bash
stat -c '%o' out.txt
```

```text
4096
```

- **프로그램은 전혀 바뀌지 않았는데 시스템 호출 횟수가 140배 넘게 차이 납니다.** 표준 라이브러리가 출력 대상이 터미널인지 파일인지 보고 버퍼링 방식을 정하기 때문입니다.
- 한 줄이 26바이트인 것은 한글이 UTF-8에서 글자당 3바이트이기 때문입니다. `0000`(4) + 공백(1) + `번째`(6) + 공백(1) + `줄입니다.`(13) + 개행(1) = 26.
- 블록 크기는 파일 시스템이 알려 주는 값이라 환경에 따라 4096이 아닐 수 있습니다. 그때는 `write` 횟수도 달라집니다. **`stat -c '%o'`로 자기 환경의 값을 확인한 뒤 계산이 맞는지 보십시오.**
- 이것이 1부 9강에서 "출력이 안 보이면 `fflush`"라고 배운 현상의 정확한 원인입니다. 화면일 때는 줄마다 나가므로 문제가 드러나지 않다가, 파일로 재지정하는 순간 드러납니다.

---

### 문제 7. `errno` 세 가지 상황

`open`이 세 가지 다른 `errno`를 남기도록 만들고 표로 정리하십시오.

**정답 및 해설**

| 대상 | `errno` | 이름 | 뜻 |
|---|---|---|---|
| `/없는파일` | 2 | `ENOENT` | 파일이 없다 |
| `/etc/shadow` | 13 | `EACCES` | 권한이 없다 |
| `/etc` (쓰기 모드로) | 21 | `EISDIR` | 디렉터리다 |

```bash
./err_demo /etc
```

디렉터리를 `O_RDONLY`로 여는 것은 **성공합니다**(3강에서 디렉터리를 읽을 때 씁니다). `EISDIR`을 보려면 쓰기 모드로 열어야 합니다.

```c
fd = open("/etc", O_WRONLY);
```

- **`errno` 값 자체(2, 13, 21)를 코드에 적지 마십시오.** 반드시 `ENOENT` 같은 이름을 쓰십시오. 숫자는 시스템마다 다를 수 있습니다.
- 오류를 구분해 처리해야 할 때가 있습니다. 예를 들어 "파일이 없으면 새로 만들고, 권한이 없으면 포기한다"는 판단은 `errno == ENOENT`인지 보아야 가능합니다.

---

### 문제 8. 오류 보고 함수 만들기

시스템 호출 실패를 일관되게 보고하는 함수를 만들고, 앞으로 계속 쓸 수 있도록 `util.h`/`util.c`로 분리하십시오.

**정답 및 해설**

```c
/* util.h — 2부 전체에서 함께 쓸 도구 */
#ifndef UTIL_H
#define UTIL_H

/* die: 오류 메시지를 stderr 로 내고 프로그램을 끝낸다.
   errno 가 0 이 아니면 그 설명을 함께 붙인다. */
void die(const char *fmt, ...);

/* warn: 같은 형식으로 알리기만 하고 계속 진행한다 */
void warn(const char *fmt, ...);

#endif   /* UTIL_H */
```

```c
/* util.c */
#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>
#include <errno.h>
#include <string.h>
#include "util.h"

static void report(const char *fmt, va_list ap)
{
    int saved = errno;              /* 아래 호출들이 errno 를 바꿀 수 있다 */

    vfprintf(stderr, fmt, ap);
    if (saved != 0)
        fprintf(stderr, ": %s", strerror(saved));
    fprintf(stderr, "\n");
}

void die(const char *fmt, ...)
{
    va_list ap;

    va_start(ap, fmt);
    report(fmt, ap);
    va_end(ap);
    exit(EXIT_FAILURE);
}

void warn(const char *fmt, ...)
{
    va_list ap;

    va_start(ap, fmt);
    report(fmt, ap);
    va_end(ap);
}
```

```c
    fd = open(path, O_RDONLY);
    if (fd == -1)
        die("open(\"%s\")", path);
```

```text
open("/없는파일"): No such file or directory
```

- **`errno`를 즉시 저장한 것**이 핵심입니다(`int saved = errno;`). `vfprintf`가 내부적으로 `errno`를 바꿀 수 있기 때문입니다. 10.2절 규칙 ②의 실천입니다.
- `<stdarg.h>`의 가변 인자는 1부에서 다루지 않았지만, `va_list`·`va_start`·`va_end` 세 가지만 알면 충분합니다. `printf`가 인자를 여러 개 받는 원리가 이것입니다.
- **이 파일은 2부 내내 재사용합니다.** 1부 11강에서 만든 `libcstudy`처럼, 여기서부터 `libcmid`를 쌓아 가십시오.

---

### 문제 9. `man` 읽고 요약하기

`man 2 read`를 읽고 다음 표를 채우십시오.

**정답 및 해설**

| 항목 | 내용 |
|---|---|
| 필요한 헤더 | `<unistd.h>` |
| 원형 | `ssize_t read(int fd, void buf[.count], size_t count);` |
| 성공 반환 | **읽은 바이트 수** |
| 0을 돌려줄 때 | **파일 끝(EOF)** |
| 실패 반환 | `-1`, `errno` 설정 |
| 주요 `errno` | `EBADF`(잘못된 서술자), `EINTR`(시그널로 중단), `EAGAIN`, `EFAULT` |
| 주의할 점 | **요청한 것보다 적게 읽을 수 있다** |

- 마지막 줄이 가장 중요합니다. `read(fd, buf, 1000)`이 1000을 돌려줄 것이라고 가정하면 **반드시 언젠가 깨집니다.** 특히 네트워크에서 그렇습니다(9강).
- 반환값 **0은 실패가 아니라 파일 끝**입니다. `-1`과 혼동하지 마십시오.
- `man`에는 `EINTR`이 있습니다. 지금은 넘어가도 되지만, 5강에서 시그널을 배우면 **이 한 줄 때문에 서버가 죽는** 상황을 만나게 됩니다.

---

### 문제 10. 두 VM에서 같은 프로그램 실행

`whoami_sys`를 두 VM에서 각각 실행하고 결과를 비교하십시오.

**정답 및 해설**

| 항목 | c-srv | c-cli | 왜 |
|---|---|---|---|
| PID | 다름 | 다름 | 각자 독립된 커널이 부여 |
| UID | 1000 | 1000 | 복제본이라 계정이 같다 |
| 커널 판 | 같음 | 같음 | 같은 이미지에서 복제 |
| 작업 디렉터리 | 같을 수 있음 | 같을 수 있음 | 경로가 같으므로 |

- **PID는 컴퓨터마다 독립적입니다.** 두 VM에서 우연히 같은 PID가 나올 수도 있지만, 서로 아무 관계가 없습니다.
- 이 실습의 진짜 목적은 **"두 대는 완전히 별개의 컴퓨터"** 임을 확인하는 것입니다. 9강에서 소켓으로 연결하기 전까지, 두 대는 서로에 대해 아무것도 모릅니다.
- 실행 파일을 옮기고 싶다면 `scp`를 쓸 수 있습니다.

```bash
scp whoami_sys student@c-cli:~/cmid/lab01/
```

---

## 과제 제출물

| 번호 | 내용 |
|---|---|
| 1 | 두 VM의 `hostname`·`ip -brief address`·`id` 출력(문제 1) |
| 2 | 양방향 `ping` 결과와 SSH 접속 화면(문제 2) |
| 3 | 방화벽 차단·복구 실험 기록(문제 3) |
| 4 | 개발 도구 확인 출력(문제 4) |
| 5 | 소스 — `argwrite.c`, `many_out.c`, `util.h`/`util.c` |
| 6 | `strace -c -e trace=write` 비교 결과표(문제 6) |
| 7 | `errno` 세 가지 상황 표(문제 7) |
| 8 | `man 2 read` 요약표(문제 9) |
| 9 | 짧은 서술 ① `printf`가 화면에 글자를 띄우기까지의 층 |
| 10 | 짧은 서술 ② 시스템 호출이 라이브러리 함수보다 비싼 이유 |

---

## 정리

| 구분 | 핵심 |
|---|---|
| 환경 | **VM 2대**(`c-srv` .60 / `c-cli` .61), 호스트 전용 어댑터, `/etc/hosts`로 이름 사용 |
| 네트워크 | netplan은 **공백 들여쓰기**, 권한 600, `netplan apply` |
| 방화벽 | 실습망 허용. **`refused`는 없는 것, `timeout`은 막힌 것** |
| 컴파일 | **`-std=gnu17`**. `-std=c17`은 POSIX 이름을 감춘다 |
| 두 세계 | 사용자 공간은 커널에 **부탁**할 수만 있다 |
| 시스템 호출 | 모드 전환이 필요해 **비싸다**. 그래서 버퍼링이 존재한다 |
| 서술자 | 0·1·2는 예약. 새로 열면 **가장 작은 빈 번호** |
| `man` | **2절 = 시스템 호출.** SYNOPSIS와 RETURN VALUE를 먼저 |
| 실패 | **`-1`과 `errno`**. 실패를 확인한 직후에만 읽는다 |
| 도구 | `strace`가 2부 내내 최고의 친구 |

---

## 다음 강의 예고

**2강 「저수준 파일 입출력」** 에서는 `FILE *`의 가면을 벗깁니다.

- `open`·`read`·`write`·`close`로 파일을 직접 다룬다
- `FILE *`와 파일 서술자의 관계를 밝힌다
- **`read`가 요청보다 적게 읽는** 상황을 직접 만들어 본다
- 버퍼 크기에 따른 성능 차이를 측정한다
- `fopen`으로는 할 수 없는 일들을 해 본다

1부 9강에서 배운 파일 입출력이 **실제로 무엇이었는지** 알게 됩니다.

---

## 부록 A. VS Code로 VM 편집하기

VirtualBox 콘솔이나 `vim` 대신, 1부에서 쓰던 VS Code로 VM의 파일을 직접 편집할 수 있습니다.

| 단계 | 내용 |
|---|---|
| 1 | VS Code 확장에서 **Remote - SSH** 설치 |
| 2 | `F1` → `Remote-SSH: Connect to Host...` |
| 3 | `student@192.168.56.60` 입력 |
| 4 | 암호 입력 후 접속. 왼쪽 탐색기에서 `~/cmid` 열기 |
| 5 | 터미널(`` Ctrl+` ``)은 **VM의 셸**이 열린다 |

편집기는 익숙한 것을 쓰고 실행은 VM에서 하는 구성이 됩니다. 다만 **`vim` 기본 조작은 익혀 두십시오.** SSH만 되는 서버에서 설정 파일 한 줄을 고쳐야 하는 상황이 반드시 옵니다.

## 부록 B. 이번 강의 명령 요약

| 하려는 일 | 명령 |
|---|---|
| **컴파일(2부 표준형)** | **`gcc -Wall -Wextra -std=gnu17 파일.c -o 실행파일`** |
| IP 확인 | `ip -brief address` |
| 네트워크 적용 | `sudo netplan apply` |
| 문법만 시험 | `sudo netplan try` |
| 호스트 이름 변경 | `sudo hostnamectl set-hostname c-srv` |
| 방화벽 상태 | `sudo ufw status` |
| 실습망 허용 | `sudo ufw allow from 192.168.56.0/24` |
| 시스템 호출 문서 | `man 2 write` |
| 이름으로 검색 | `man -k socket` |
| 시스템 호출 추적 | `strace ./prog` |
| 특정 호출만 | `strace -e trace=write ./prog` |
| 횟수 요약 | `strace -c ./prog` |
| 파일 전송 | `scp prog student@c-cli:~/` |

## 부록 C. 표준 문서와 출처

이 강의에서 사실로 제시한 내용의 근거입니다. **직접 확인해 보시기 바랍니다.**

**시스템 호출과 표준**

| 내용 | 문서 |
|---|---|
| `write` — 리눅스 | [`write(2)`](https://man7.org/linux/man-pages/man2/write.2.html) |
| `write` — POSIX 표준 | [POSIX Issue 8, `write()`](https://pubs.opengroup.org/onlinepubs/9799919799/functions/write.html) |
| `read` — 리눅스 | [`read(2)`](https://man7.org/linux/man-pages/man2/read.2.html) |
| `open` | [`open(2)`](https://man7.org/linux/man-pages/man2/open.2.html) |
| `errno` 목록 | [`errno(3)`](https://man7.org/linux/man-pages/man3/errno.3.html) |
| `getpid`·`getppid` | [`getpid(2)`](https://man7.org/linux/man-pages/man2/getpid.2.html) |
| `uname` | [`uname(2)`](https://man7.org/linux/man-pages/man2/uname.2.html) |
| **기능 시험 매크로**(8.2절의 `-std` 문제) | [`feature_test_macros(7)`](https://man7.org/linux/man-pages/man7/feature_test_macros.7.html) |
| 전체 목록 | [Linux man-pages](https://man7.org/linux/man-pages/) · [POSIX Issue 8](https://pubs.opengroup.org/onlinepubs/9799919799/) |

**환경 구축**

| 내용 | 문서 |
|---|---|
| 호스트 전용 네트워크와 **IP 대역 제한**(2.4절) | [VirtualBox Manual, Ch. 6 Virtual Networking](https://www.virtualbox.org/manual/ch06.html) |
| netplan YAML 문법(4.2절) | [Netplan documentation](https://netplan.readthedocs.io/en/stable/) · `man 5 netplan` |
| `ufw` | `man 8 ufw` |
| `strace` | [strace.io](https://strace.io/) · `man 1 strace` |

**5부에서 쓸 표준(미리 보기)**

| 내용 | 문서 |
|---|---|
| **TLS 1.3 — 현행 표준** | [**RFC 9846**](https://www.rfc-editor.org/rfc/rfc9846) |
| TLS 1.3 — 2018년 판(폐기됨) | [RFC 8446](https://www.rfc-editor.org/rfc/rfc8446) |
| 핸드셰이크 바이트 단위 해설 | [The Illustrated TLS 1.3 Connection](https://tls13.xargs.org/) |
| OpenSSL 3.0 API | [docs.openssl.org/3.0](https://docs.openssl.org/3.0/) |

**본문 측정값의 출처**

| 값 | 어떻게 얻었나 |
|---|---|
| `gcc` 13.3.0 · `make` 4.3 · `strace` 6.8 · OpenSSL 3.0.13 | Ubuntu 24.04.3 LTS에서 각 도구의 `--version` |
| `write` 40바이트 · 8진 표기 | `od -b`와 `strace -e trace=write` |
| `strace -c` 30회 | 실제 실행 결과 |
| 버퍼링별 `write` 1000 / 7 / 1000회 | `strace -c -e trace=write` 실측(문제 6) |
| 블록 크기 4096 | `stat -c '%o'` |

## 부록 D. 이번 강의 예제 파일

| 파일 | 내용 | 절 |
|---|---|---|
| `hello_sys.c` | `write` 직접 호출 | 8.1 |
| `whoami_sys.c` | `getpid`·`getuid`·`uname` | 8.6 |
| `err_demo.c` | `errno`·`strerror`·`perror` | 10.3 |
| `argwrite.c` | `printf` 없이 인자 출력 | 문제 5 |
| `many_out.c` | 버퍼링 비교 | 문제 6 |
| `util.h`/`util.c` | **2부 공용 오류 처리** | 문제 8 |
