---
title: 리눅스 기초 5강 - 프로세스 관리와 소프트웨어 설치
date: 2026-09-22 09:00:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - 프로세스
  - ps
  - top
  - kill
  - 시그널
  - nice
  - systemd
  - systemctl
  - cron
  - apt
  - dpkg
  - rpm
pin:
mermaid: false
---

> **학습 목표**
> 1. 프로세스의 개념과 유형을 구분하고 상태 값을 해석할 수 있다.
> 2. `ps`, `top`, `pstree`로 프로세스를 조회할 수 있다.
> 3. 시그널의 종류를 이해하고 프로세스를 적절히 종료할 수 있다.
> 4. 포그라운드·백그라운드 작업을 전환하고 제어할 수 있다.
> 5. `nice` 값으로 프로세스 우선순위를 조정할 수 있다.
> 6. `systemctl`로 서비스를 제어하고 자동 시작을 설정할 수 있다.
> 7. `cron`과 `at`으로 작업을 예약할 수 있다.
> 8. `apt`와 `dpkg`로 패키지를 관리하고, RPM 계열 명령과 대응시킬 수 있다.
{: .prompt-info }

셸에 명령을 입력하면 프로그램이 메모리에 적재되어 실행 상태가 된다. 이 실행 중인 프로그램의 인스턴스를 **프로세스**라 한다.

시스템 관리 업무의 상당 부분은 "어떤 프로세스가 자원을 얼마나 사용하고 있으며, 중단된 서비스를 어떻게 복구할 것인가"라는 문제를 해결하는 일이다. 아울러 필요한 소프트웨어를 안전하게 설치·갱신·제거하는 패키지 관리 역시 운영의 기본기에 해당한다.

| 절 | 내용 | 배정 시간 |
|---|---|---|
| 제1절 | 프로세스의 이해와 조회 | 35분 |
| 제2절 | 시그널과 작업 제어 | 35분 |
| 제3절 | 서비스 관리와 작업 예약 | 40분 |
| 제4절 | 패키지 관리 | 30분 |
| 제5절 | 종합 실습 | 30분 |
| 제6·7절 | 오류 대응 및 이론 평가 | 10분 |

---
---

# 제1절. 프로세스의 이해와 조회

---

## 1.1 프로그램과 프로세스의 구분

| 용어 | 정의 | 비유 |
|---|---|---|
| **프로그램** | 디스크에 저장된 실행 파일(정적) | 조리법(문서) |
| **프로세스** | 메모리에 적재되어 실행 중인 프로그램의 인스턴스(동적) | 실제로 조리하는 행위 |
| **스레드** | 프로세스 내부에서 자원을 공유하며 동작하는 실행 흐름 | 한 조리 과정 내의 병행 작업 |
| **데몬(Daemon)** | 배경에서 상주하며 서비스를 제공하는 프로세스 | 상시 가동되는 설비 |

동일한 조리법으로 세 차례 조리하면 결과물이 세 개 생기듯, 동일한 프로그램을 세 번 실행하면 세 개의 프로세스가 생성된다.

---

## 1.2 프로세스의 속성

| 속성 | 의미 |
|---|---|
| **PID** | **P**rocess **ID**entifier. 프로세스 식별 번호 |
| **PPID** | **P**arent **PID**. 부모 프로세스의 식별 번호 |
| UID / EUID | 실행 사용자 / 유효 사용자(SetUID 적용 결과가 반영됨) |
| **STAT** | 현재 상태 |
| NI / PRI | nice 값 / 우선순위 |
| TTY | 연결된 터미널(`?`이면 데몬) |

**STAT(상태) 값의 의미**

| 값 | 의미 |
|---|---|
| **R** | **R**unning. 실행 중 |
| **S** | **S**leeping. 대기 중(대부분의 프로세스가 이 상태) |
| D | 중단할 수 없는 대기(디스크 입출력 중) |
| **T** | s**T**opped. 정지됨 |
| **Z** | **Z**ombie. 좀비 |

---

## 1.3 프로세스의 계층 구조

리눅스에서 새로운 프로세스는 **`fork()`로 부모를 복제한 뒤 `exec()`로 다른 프로그램으로 교체**하는 방식으로 생성된다. 이러한 구조로 인해 모든 프로세스는 부모를 가지며, 최상위 부모는 **PID 1번의 `systemd`** 이다.

```
 systemd (PID 1)             모든 프로세스의 최상위 부모
   ├── sshd                  원격 접속 관리
   │     └── sshd: student   사용자 접속 세션
   │            └── bash     사용자의 셸
   │                  └── ps 사용자가 실행한 명령
   ├── cron                  예약 작업 관리
   └── nginx                 웹 서버
         ├── nginx: worker
         └── nginx: worker
```

| 특수 상태 | 정의 | 대응 방법 |
|---|---|---|
| **좀비(Zombie)** | 종료되었으나 부모가 종료 상태를 회수하지 않아 프로세스 표에 잔존 | 부모를 종료하면 systemd가 입양하여 정리한다. |
| 고아(Orphan) | 부모가 먼저 종료된 프로세스 | systemd가 자동으로 입양하므로 문제되지 않는다. |

> **자주 출제되는 함정**
> "좀비 프로세스는 `kill -9`로 제거할 수 있다"는 진술은 **거짓**이다.
>
> 좀비는 이미 종료된 프로세스이므로 시그널이 의미를 갖지 않는다. 부모가 상태를 회수하거나 부모가 종료되어야만 소멸한다.
{: .prompt-danger }

---

## 1.4 프로세스 조회 — `ps`

`ps`는 **p**rocess **s**tatus의 약어이며, 역사적 이유로 두 계열의 옵션 문법을 함께 지원한다.

| 형식 | 명령 | 특징 |
|---|---|---|
| BSD 스타일(하이픈 없음) | **`ps aux`** | `a` 모든 터미널, `u` 사용자 중심 형식, `x` 터미널 없는 프로세스 포함. **CPU·메모리 사용률 표시** |
| System V 스타일(하이픈 사용) | **`ps -ef`** | `-e` 모든 프로세스, `-f` 전체 형식. **PPID 표시** |

| 명령 | 용도 |
|---|---|
| `ps` | 현재 터미널의 프로세스만 |
| `ps aux` | 전체 프로세스(사용률 포함) |
| `ps -ef` | 전체 프로세스(부모 번호 포함) |
| `ps -u alice` | 특정 사용자의 프로세스 |
| `ps aux --sort=-%mem` | 메모리 사용량 내림차순 정렬 |
| `pgrep -a nginx` | 이름으로 PID 검색 |
| `pstree -p` | 트리 형태로 표시 |

---

## 1.5 실시간 감시 — `top`

`top`은 시스템 자원 사용 현황을 실시간으로 갱신하여 표시한다.

| 키 | 동작 |
|---|---|
| **`P`** | CPU 사용률순 정렬 |
| **`M`** | 메모리 사용률순 정렬 |
| `k` | 프로세스에 시그널 전송 |
| `u` | 특정 사용자만 표시 |
| `1` | CPU 코어별 사용률 표시 |
| **`q`** | **종료** |

| 옵션 | 기능 |
|---|---|
| `top -d 3` | 갱신 주기 3초 |
| `top -b -n 1` | 대화형이 아닌 텍스트 출력(파일 저장용) |
| `top -u alice` | 특정 사용자 |

> **부하 평균(load average)의 해석**
> `top` 상단에 표시되는 부하 평균은 최근 1분·5분·15분 동안 실행을 대기한 프로세스의 평균 개수를 의미한다.
>
> **CPU 코어 수보다 큰 값이 지속되면 과부하 상태**로 판단한다. 코어가 4개인데 부하 평균이 8이면 작업이 적체되고 있는 것이다.
{: .prompt-tip }

---

> ### 따라 하기 5-1. 프로세스 조회
>
> **목적** 여러 방식으로 프로세스를 조회하고 각 출력 형식의 차이를 파악한다.
{: .prompt-tip }

**1단계.** 실습 디렉터리를 생성한다.

```bash
mkdir -p ~/lab05 && cd ~/lab05
```

**2단계.** 현재 터미널의 프로세스를 조회한다.

```bash
ps
```

> `bash`와 `ps` 두 개만 출력된다. 옵션 없는 `ps`는 현재 터미널에 연결된 프로세스만 표시한다.

```bash
ps -f
```

> PPID 열이 추가된다. **`ps`의 부모가 `bash`임을 확인한다.**

**3단계.** 두 가지 문법으로 전체 프로세스를 조회한다.

```bash
ps aux | head -10
```

> `%CPU`, `%MEM`, `STAT` 열이 포함된다.

```bash
ps -ef | head -10
```

> `PPID` 열이 포함된다.

**4단계.** PID 1번을 확인한다.

```bash
ps -p 1 -o pid,ppid,user,comm
```

> `systemd`가 출력된다. 모든 프로세스의 최상위 부모이다.

**5단계.** 계층 구조를 확인한다.

```bash
pstree -p | head -20
```

```bash
pstree -p $$
```

> 자신의 셸에서 뻗어 나가는 구조를 확인한다. 명령을 실행하면 셸의 자식으로 생성된다는 사실을 시각적으로 파악할 수 있다.

**6단계.** 자원 사용량 기준으로 정렬한다.

```bash
ps aux --sort=-%mem | head -6
```

> 앞의 `-`는 내림차순을 의미한다.

```bash
ps aux --sort=-%cpu | head -6
```

**7단계.** `top`을 실행하여 조작한다.

```bash
top
```

> 화면 내에서 다음을 순서대로 수행한다.
> 1. `P` — CPU 사용률순 정렬
> 2. `M` — 메모리 사용률순 정렬
> 3. `1` — 코어별 표시
> 4. `q` — 종료

```bash
top -b -n 1 | head -12
```

> `-b`(batch) 모드는 대화형이 아닌 텍스트로 한 번만 출력하므로, 파일 저장이나 파이프 연결에 사용한다.

> **확인 사항** `ps aux`와 `ps -ef`의 열 구성 차이를 설명할 수 있고, 메모리 사용량 상위 프로세스를 조회하였다면 성공이다.
{: .prompt-tip }

---
---

# 제2절. 시그널과 작업 제어

---

## 2.1 시그널의 개념과 종류

**시그널(signal)** 은 커널 또는 다른 프로세스가 특정 프로세스에 전달하는 비동기 알림이다.

| 번호 | 이름 | 의미 | 무시 가능 여부 |
|---|---|---|---|
| 1 | SIGHUP | 터미널 연결 종료. 데몬에서는 **설정 재적재** 용도로 관용적으로 사용 | 가능 |
| 2 | SIGINT | 인터럽트(`Ctrl + C`) | 가능 |
| 3 | SIGQUIT | 종료 및 코어 덤프 | 가능 |
| **9** | **SIGKILL** | **강제 종료** | **불가능** |
| **15** | **SIGTERM** | **정상 종료 요청(기본값)** | 가능 |
| 18 | SIGCONT | 정지된 프로세스 재개 | |
| 19 | SIGSTOP | 정지 | 불가능 |
| 20 | SIGTSTP | 정지(`Ctrl + Z`) | 가능 |

---

## 2.2 프로세스 종료 명령

| 명령 | 동작 |
|---|---|
| `kill PID` | SIGTERM(15) 전송 |
| `kill -9 PID` | SIGKILL(9) 전송 |
| `kill -HUP PID` | 설정 재적재 요청 |
| `kill -l` | 시그널 목록 조회 |
| `killall 프로세스명` | 동일한 이름의 모든 프로세스에 전송 |
| `pkill -u alice` | 조건에 해당하는 프로세스에 전송 |
| `pkill -f "패턴"` | 전체 명령행에서 패턴이 일치하는 프로세스 |

> **`kill -9`를 우선 사용해서는 안 되는 이유**
>
> | 구분 | SIGTERM(15) | SIGKILL(9) |
> |---|---|---|
> | 의미 | 정리 후 종료할 것을 요청 | 즉시 강제 종료 |
> | 프로세스의 대응 | 임시 파일 삭제, 버퍼 기록, 잠금 해제 등 정리 작업 수행 | **아무 작업도 수행하지 못함** |
> | 결과 | 정상 종료 | 데이터 손상 가능 |
>
> **먼저 `kill`(SIGTERM)을 전송하고, 일정 시간이 지나도 응답이 없을 때에만 `kill -9`를 사용한다.** 데이터베이스를 SIGKILL로 종료하면 자료가 손상될 수 있다.
{: .prompt-danger }

---

## 2.3 포그라운드와 백그라운드

| 구분 | 정의 |
|---|---|
| **포그라운드(foreground)** | 터미널을 점유하며 실행. 종료될 때까지 다른 명령을 입력할 수 없다. |
| **백그라운드(background)** | 터미널을 반환하고 배경에서 실행. 명령 끝에 `&`를 붙인다. |

| 명령 | 동작 |
|---|---|
| `명령 &` | 백그라운드로 실행 |
| `jobs` | 현재 셸의 작업 목록(`-l`은 PID 포함) |
| `fg %1` | 1번 작업을 포그라운드로 전환 |
| `bg %1` | 정지된 1번 작업을 백그라운드에서 재개 |
| `Ctrl + Z` | 포그라운드 작업을 정지 |
| **`nohup 명령 &`** | **로그아웃 후에도 계속 실행** |

> **`nohup`의 필요성**
> `nohup`은 **no h**ang**up**의 약어이다. SSH로 접속하여 장시간 소요되는 작업을 시작한 뒤 접속이 종료되면, 해당 작업도 SIGHUP을 받아 함께 종료된다.
>
> `nohup`을 사용하면 이 시그널을 무시하므로 접속이 끊겨도 작업이 계속된다. 백업이나 대용량 처리 작업에 필수적이다.
{: .prompt-tip }

---

## 2.4 우선순위 조정 — `nice`

| 항목 | 범위 | 의미 |
|---|---|---|
| **NI(nice 값)** | **-20 ~ 19** | 값이 작을수록 **우선순위가 높다.** 기본값은 0 |
| PRI | — | 커널이 계산하는 실제 우선순위 |

| 명령 | 동작 |
|---|---|
| `nice -n 10 명령` | nice 값 10으로 시작 |
| `renice -n 5 -p PID` | 실행 중인 프로세스의 nice 값 변경 |
| `renice -n 5 -u alice` | 특정 사용자의 모든 프로세스 |

> **명칭의 유래와 권한 제약**
> nice는 '양보하다'라는 의미로, 값이 클수록 다른 프로세스에 CPU를 잘 양보한다는 뜻이다. 따라서 nice 값이 크면 우선순위가 낮다.
>
> **일반 사용자는 nice 값을 높이는(우선순위를 낮추는) 것만 가능하며, 낮추려면 관리자 권한이 필요하다.** 특정 사용자가 자원을 독점하는 것을 방지하기 위한 제약이다.
>
> nice 값의 범위 `-20 ~ 19`는 시험에 반복 출제된다.
{: .prompt-warning }

---

> ### 따라 하기 5-2. 작업 제어와 프로세스 종료
>
> **목적** 백그라운드 작업을 제어하고, SIGTERM과 SIGKILL의 차이를 실험으로 확인한다.
{: .prompt-tip }

**1단계.** 포그라운드 실행을 확인한다.

```bash
sleep 60
```

> 60초 동안 다른 명령을 입력할 수 없다. `Ctrl + C`로 중단한다.

**2단계.** 백그라운드로 실행한다.

```bash
sleep 300 &
```

```bash
sleep 400 &
```

```bash
jobs -l
```

> `[1]`, `[2]`는 작업 번호이며 그 옆의 숫자가 PID이다.

**3단계.** 포그라운드와 백그라운드를 전환한다.

```bash
fg %1
```

> 1번 작업이 포그라운드로 전환되어 터미널을 점유한다.

이때 **`Ctrl + Z`** 를 누른다.

```bash
jobs
```

> 상태가 `Stopped`로 표시된다.

```bash
bg %1
```

```bash
jobs
```

> 상태가 `Running`으로 복귀한다.

**4단계.** 작업을 종료한다.

```bash
kill %1
```

```bash
kill %2
```

```bash
jobs
```

**5단계.** 시그널을 무시하는 프로그램을 작성한다.

```bash
cat > ~/lab05/stubborn.sh << 'EOF'
#!/bin/bash
trap 'echo "[$(date +%T)] SIGTERM 수신 - 무시함" >> /tmp/stubborn.log' TERM
echo "[$(date +%T)] 시작. PID=$$" >> /tmp/stubborn.log
while true; do sleep 1; done
EOF
```

```bash
chmod +x ~/lab05/stubborn.sh
```

> `trap`은 특정 시그널을 수신했을 때 수행할 동작을 지정하는 명령이다. 정상적인 서비스는 SIGTERM 수신 시 정리 작업 후 종료하도록 구현되어 있으나, 여기서는 학습을 위해 의도적으로 무시하도록 작성하였다.

**6단계.** SIGTERM을 전송한다.

```bash
~/lab05/stubborn.sh &
```

```bash
PID=$(pgrep -f stubborn.sh | head -1) && echo "PID = $PID"
```

```bash
kill $PID
```

```bash
sleep 1 && ps -p $PID -o pid,stat,comm
```

> 프로세스가 여전히 실행 중이다.

```bash
cat /tmp/stubborn.log
```

> "SIGTERM 수신 - 무시함"이 기록되어 있다. **SIGTERM은 종료 요청일 뿐 강제력이 없다.**

**7단계.** SIGKILL을 전송한다.

```bash
kill -9 $PID
```

```bash
sleep 1 && ps -p $PID -o pid,stat,comm || echo "프로세스가 종료되었습니다"
```

```bash
cat /tmp/stubborn.log
```

> 로그에 아무것도 추가되지 않았다. **SIGKILL은 커널이 직접 처리하므로 `trap`으로 가로챌 수 없으며, 프로세스가 반응할 기회조차 없다.** 이것이 `kill -9`가 위험한 이유이다.

**8단계.** 정지와 재개를 확인한다.

```bash
sleep 600 &
```

```bash
PID2=$! && echo "PID2 = $PID2"
```

> `$!`는 가장 최근에 백그라운드로 실행한 프로세스의 PID를 담은 특수 변수이다.

```bash
kill -STOP $PID2 && ps -p $PID2 -o pid,stat,comm
```

> `STAT`가 `T`(정지)로 변경된다.

```bash
kill -CONT $PID2 && ps -p $PID2 -o pid,stat,comm
```

> `S`(대기)로 복귀한다.

```bash
kill $PID2
```

**9단계.** nice 값을 조정한다.

```bash
sleep 300 &
```

```bash
ps -o pid,ni,comm -p $!
```

```bash
nice -n 10 sleep 300 &
```

```bash
ps -o pid,ni,comm -p $!
```

> NI가 10으로 설정되었다.

```bash
nice -n -5 sleep 300 &
```

```bash
ps -o pid,ni,comm -p $!
```

> 경고가 출력되고 NI는 0으로 유지된다. **일반 사용자는 우선순위를 높일 수 없다.**

```bash
sleep 400 &
```

```bash
PID3=$! && renice -n 15 -p $PID3
```

```bash
ps -o pid,ni,comm -p $PID3
```

```bash
renice -n 5 -p $PID3
```

> 실패한다. 일반 사용자는 한 번 높인 nice 값을 다시 낮출 수 없다.

**10단계.** 정리한다.

```bash
pkill -u $(whoami) sleep
```

```bash
sudo pkill sleep 2>/dev/null; rm -f /tmp/stubborn.log
```

```bash
pgrep -a sleep || echo "정리 완료"
```

> **확인 사항** `fg`·`Ctrl+Z`·`bg`로 작업 상태를 전환하였고, SIGTERM은 무시될 수 있으나 SIGKILL은 무시되지 않음을 로그로 확인하였다면 성공이다.
{: .prompt-tip }

---
---

# 제3절. 서비스 관리와 작업 예약

---

## 3.1 systemd와 유닛

서버에서 동작하는 프로그램(웹 서버, 데이터베이스, 원격 접속 등)은 대부분 **서비스(데몬)** 형태로 관리된다. 이를 관리하는 체계가 **systemd**이며, 관리 명령은 **`systemctl`** 이다.

systemd의 관리 대상을 **유닛(unit)** 이라 하며 확장자로 종류를 구분한다.

| 확장자 | 종류 |
|---|---|
| `.service` | 서비스(데몬) |
| `.socket` | 소켓 활성화 |
| `.timer` | 시간 기반 실행(cron 대체) |
| `.mount` | 마운트 지점 |
| `.target` | 유닛의 그룹(런레벨에 대응) |

유닛 파일의 위치는 다음과 같다.

| 경로 | 성격 |
|---|---|
| `/lib/systemd/system/` | 패키지가 설치한 원본 |
| **`/etc/systemd/system/`** | **관리자가 작성·재정의(우선순위 높음)** |

---

## 3.2 `systemctl` 명령

| 명령 | 기능 |
|---|---|
| `systemctl status 서비스` | 상태 조회 |
| `sudo systemctl start 서비스` | **지금 시작** |
| `sudo systemctl stop 서비스` | 중지 |
| `sudo systemctl restart 서비스` | 재시작 |
| `sudo systemctl reload 서비스` | 설정만 재적재(무중단) |
| **`sudo systemctl enable 서비스`** | **부팅 시 자동 시작 등록** |
| `sudo systemctl disable 서비스` | 자동 시작 해제 |
| `sudo systemctl enable --now 서비스` | 등록과 시작을 동시에 |
| `systemctl is-active 서비스` | 실행 여부 조회 |
| `systemctl is-enabled 서비스` | 자동 시작 여부 조회 |
| `sudo systemctl daemon-reload` | 유닛 파일 변경 후 반영 |

> **`start`와 `enable`의 구분 — 반드시 이해하여야 할 개념**
>
> | 구분 | `start` | `enable` |
> |---|---|---|
> | 의미 | **현재 시점에 실행** | **다음 부팅부터 자동 실행되도록 등록** |
> | 재부팅 후 | 중지 상태 | 자동 실행 |
>
> 두 명령은 서로 독립적이다. `start`만 수행하고 `enable`을 누락하면 **재부팅 후 서비스가 기동되지 않는다.** 실무에서 매우 빈번하게 발생하는 실수이며 시험에도 자주 출제된다.
{: .prompt-danger }

로그 조회는 `journalctl`을 사용한다.

| 명령 | 기능 |
|---|---|
| `journalctl -u 서비스` | 특정 유닛의 로그 |
| `journalctl -f` | 실시간 추적 |
| `journalctl -b` | 이번 부팅 이후 |
| `journalctl -p err` | 오류 수준 이상만 |

---

## 3.3 작업 예약 — `cron`과 `at`

| 구분 | `cron` | `at` |
|---|---|---|
| 성격 | **반복** 실행 | **1회** 실행 |
| 등록 | `crontab -e` | `at 시각` |
| 목록 조회 | `crontab -l` | `atq` |
| 삭제 | `crontab -r` | `atrm 번호` |

crontab의 시간 필드는 다음과 같이 구성된다.

```
 ┌───── 분   (0-59)
 │ ┌─── 시   (0-23)
 │ │ ┌─ 일   (1-31)
 │ │ │ ┌─ 월   (1-12)
 │ │ │ │ ┌ 요일 (0-7, 0과 7은 일요일)
 │ │ │ │ │
 * * * * *   실행할 명령
```

| 설정 예시 | 의미 |
|---|---|
| `0 3 * * *` | 매일 03시 00분 |
| `*/10 * * * *` | 10분마다 |
| `0 2 * * 0` | 매주 일요일 02시 00분 |
| `30 4 1 * *` | 매월 1일 04시 30분 |
| `0 9-18 * * 1-5` | 평일 09시부터 18시까지 매시 정각 |

| 관련 파일 | 설명 |
|---|---|
| `/etc/crontab` | 시스템 crontab. **실행 사용자 필드가 추가로 존재** |
| `/etc/cron.d/` | 패키지가 제공하는 작업 |
| `/etc/cron.{hourly,daily,weekly,monthly}/` | 주기별 스크립트 디렉터리 |
| `/var/spool/cron/crontabs/` | 사용자별 crontab 저장 위치 |

> **cron 작업이 동작하지 않는 주된 원인**
> "수동으로 실행하면 동작하는데 cron에 등록하면 동작하지 않는다"는 문제의 대부분은 **`PATH` 때문**이다.
>
> cron이 실행하는 환경은 로그인 셸이 아니므로 `PATH`가 매우 제한적이다(`/usr/bin:/bin` 수준). 따라서 스크립트 내부에서 명령어를 **절대 경로로 지정**하거나, crontab 상단에 `PATH=`를 명시하여야 한다.
{: .prompt-danger }

---

> ### 따라 하기 5-3. 서비스 관리와 작업 예약
>
> **목적** 사용자 정의 서비스를 등록하여 `start`와 `enable`의 차이를 확인하고, cron으로 작업을 예약한다.
{: .prompt-tip }

**1단계.** 현재 동작 중인 서비스를 조회한다.

```bash
systemctl list-units --type=service --state=running | head -15
```

**2단계.** SSH 서비스의 상태를 확인한다.

```bash
systemctl status ssh
```

> 출력에서 두 항목을 확인한다.
> - `Loaded: ... enabled` → 부팅 시 자동 시작 여부
> - `Active: active (running)` → 현재 실행 여부

```bash
systemctl is-active ssh
```

```bash
systemctl is-enabled ssh
```

**3단계.** 사용자 정의 서비스 유닛을 작성한다.

```bash
sudo tee /etc/systemd/system/lab-heartbeat.service > /dev/null << 'EOF'
[Unit]
Description=Lab Heartbeat Logger

[Service]
Type=simple
ExecStart=/bin/bash -c 'while true; do echo "heartbeat"; sleep 10; done'
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

```bash
cat /etc/systemd/system/lab-heartbeat.service
```

> 유닛 파일은 세 절로 구성된다.
> - `[Unit]` — 설명과 의존성
> - `[Service]` — 실행 방법
> - `[Install]` — 자동 시작 대상

**4단계.** systemd에 새 유닛을 인식시킨다.

```bash
sudo systemctl daemon-reload
```

**5단계.** `start`와 `enable`의 차이를 확인한다.

```bash
systemctl is-enabled lab-heartbeat
```

> `disabled`가 출력된다.

```bash
sudo systemctl start lab-heartbeat
```

```bash
systemctl is-active lab-heartbeat
```

> `active`가 출력된다.

```bash
systemctl is-enabled lab-heartbeat
```

> **여전히 `disabled`이다.** 즉 현재는 실행 중이나 재부팅하면 기동되지 않는다.

**6단계.** 로그를 조회한다.

```bash
journalctl -u lab-heartbeat -n 5 --no-pager
```

```bash
journalctl -u lab-heartbeat -f
```

> 10초마다 새 행이 추가되는 것을 확인한 뒤 `Ctrl + C`로 종료한다.

**7단계.** 자동 시작을 등록한다.

```bash
sudo systemctl enable lab-heartbeat
```

```bash
systemctl is-enabled lab-heartbeat
```

```bash
ls -l /etc/systemd/system/multi-user.target.wants/ | grep lab-heartbeat
```

> `enable`의 실제 동작은 `[Install]` 절의 대상 디렉터리에 **심볼릭 링크를 생성하는 것**임을 확인한다.

**8단계.** 서비스를 정리한다.

```bash
sudo systemctl stop lab-heartbeat
```

```bash
sudo systemctl disable lab-heartbeat
```

```bash
sudo rm /etc/systemd/system/lab-heartbeat.service
```

```bash
sudo systemctl daemon-reload
```

**9단계.** cron으로 반복 작업을 등록한다.

```bash
cat > ~/lab05/logger.sh << 'EOF'
#!/bin/bash
echo "[$(date '+%F %T')] cron 실행 / PATH=$PATH" >> "$HOME/lab05/cron.log"
EOF
```

```bash
chmod +x ~/lab05/logger.sh
```

```bash
~/lab05/logger.sh && cat ~/lab05/cron.log
```

> 수동 실행 시의 `PATH`가 길다는 점을 확인한다.

```bash
(crontab -l 2>/dev/null; echo "* * * * * $HOME/lab05/logger.sh") | crontab -
```

```bash
crontab -l
```

**10단계.** 실행 결과를 확인하고 `PATH`를 비교한다.

```bash
sleep 130 && cat ~/lab05/cron.log
```

> **수동 실행 시와 cron 실행 시의 `PATH`가 다르다는 점을 확인한다.** cron의 `PATH`는 매우 짧으므로, 스크립트에서 명령어를 절대 경로로 지정해야 하는 이유가 여기에 있다.

```bash
grep CRON /var/log/syslog | tail -5
```

**11단계.** 예약을 삭제한다.

```bash
crontab -r
```

```bash
crontab -l 2>/dev/null || echo "삭제 완료"
```

> **확인 사항** `is-active`는 `active`이나 `is-enabled`는 `disabled`인 상태를 직접 확인하였고, cron 실행 시의 `PATH`가 짧다는 점을 관찰하였다면 성공이다.
{: .prompt-tip }

---
---

# 제4절. 패키지 관리

---

## 4.1 패키지 관리자의 역할

윈도우에서 프로그램을 설치하려면 설치 파일을 검색·내려받기·실행하는 절차를 거쳐야 하며, 필요한 구성 요소도 개별적으로 확인해야 한다.

리눅스의 **패키지 관리자**는 이 과정을 자동화한다. 명령 한 줄로 저장소에서 검색하고, 내려받고, 의존 관계에 있는 다른 패키지까지 함께 설치한다.

```bash
sudo apt install tree
```

---

## 4.2 계열별 명령어 대응

| 수행 작업 | **데비안·우분투** | **레드햇 계열** |
|---|---|---|
| 패키지 형식 | `.deb` | `.rpm` |
| 저수준 도구 | **`dpkg`** | **`rpm`** |
| 고수준 도구 | **`apt`** | **`dnf`**(구 `yum`) |
| 설치 | `apt install 이름` | `dnf install 이름` |
| 삭제 | `apt remove 이름` | `dnf remove 이름` |
| 목록 갱신 | `apt update` | `dnf check-update` |
| 전체 갱신 | `apt upgrade` | `dnf upgrade` |
| 설치 목록 조회 | **`dpkg -l`** | **`rpm -qa`** |
| 패키지의 파일 목록 | **`dpkg -L 이름`** | **`rpm -ql 이름`** |
| 파일의 소속 패키지 | **`dpkg -S 파일`** | **`rpm -qf 파일`** |
| 저장소 설정 | `/etc/apt/sources.list*` | `/etc/yum.repos.d/` |

> 시험에서는 양쪽 계열의 명령을 모두 묻는다. 특히 `rpm -qa`(전체 목록), `rpm -e`(삭제), `rpm -ivh`(설치)는 자주 출제된다.
{: .prompt-warning }

---

## 4.3 `apt` 주요 명령

| 명령 | 기능 |
|---|---|
| **`sudo apt update`** | **저장소의 패키지 목록 정보만 갱신** |
| **`sudo apt upgrade`** | **설치된 패키지를 실제로 갱신** |
| `sudo apt full-upgrade` | 의존성 해결을 위해 제거까지 허용하는 갱신 |
| `sudo apt install 이름` | 설치 |
| `sudo apt remove 이름` | 제거(**설정 파일은 잔존**) |
| **`sudo apt purge 이름`** | **설정 파일까지 완전 제거** |
| `sudo apt autoremove` | 불필요한 의존 패키지 정리 |
| `apt search 키워드` | 검색 |
| `apt show 이름` | 상세 정보 |
| `apt list --installed` | 설치된 패키지 목록 |
| `sudo apt clean` | 내려받은 파일 캐시 삭제 |

> **혼동하기 쉬운 두 쌍의 명령**
>
> **① `update`와 `upgrade`**
> - `apt update` : 저장소의 **목록 정보만** 갱신한다. 패키지 자체는 변경되지 않는다.
> - `apt upgrade` : 실제로 패키지를 갱신한다.
>
> 따라서 항상 `update`를 먼저 실행하고 `upgrade`를 수행한다.
>
> **② `remove`와 `purge`**
> - `remove` : 프로그램만 제거하고 **설정 파일은 남긴다.**
> - `purge` : 설정 파일까지 완전히 제거한다.
{: .prompt-danger }

---

## 4.4 `dpkg` 주요 명령

| 명령 | 기능 |
|---|---|
| `sudo dpkg -i 파일.deb` | 설치(**의존성을 자동 해결하지 않음**) |
| `sudo dpkg -r 이름` | 제거 |
| `sudo dpkg -P 이름` | 설정 파일까지 제거 |
| `dpkg -l` | 설치 목록 조회 |
| `dpkg -L 이름` | 해당 패키지가 설치한 파일 목록 |
| `dpkg -S 경로` | 해당 파일이 속한 패키지 조회 |

`dpkg -i` 실행 후 의존성 오류가 발생하면 `sudo apt install -f`로 해결한다.

---

## 4.5 압축과 아카이브(복습)

| 명령 | 기능 | 확장자 |
|---|---|---|
| `tar` | 여러 파일을 하나로 묶기 | `.tar` |
| `gzip` / `gunzip` | 압축 / 해제(속도 우선) | `.gz` |
| `bzip2` / `bunzip2` | 압축률이 더 높음 | `.bz2` |
| `xz` / `unxz` | 압축률이 가장 높음 | `.xz` |
| `zip` / `unzip` | 윈도우 호환 | `.zip` |

일반적으로 압축률은 `xz > bzip2 > gzip` 순이고, 처리 속도는 그 반대이다. 배포용 아카이브에는 `xz`, 로그 순환과 같이 빈번한 작업에는 `gzip`이 주로 사용된다.

---

> ### 따라 하기 5-4. 패키지 관리
>
> **목적** `apt`와 `dpkg`로 패키지를 설치·조회·삭제하고, `remove`와 `purge`의 차이를 확인한다.
{: .prompt-tip }

**1단계.** 저장소 설정을 확인한다.

```bash
cat /etc/apt/sources.list.d/ubuntu.sources 2>/dev/null || cat /etc/apt/sources.list
```

> Ubuntu 24.04부터는 기본 저장소 정의가 `/etc/apt/sources.list.d/ubuntu.sources`로 이동하였다.

**2단계.** 패키지 목록을 갱신한다.

```bash
sudo apt update
```

```bash
apt list --upgradable 2>/dev/null | head -10
```

**3단계.** 패키지를 검색하고 정보를 조회한다.

```bash
apt search "^tree$" 2>/dev/null
```

```bash
apt show tree 2>/dev/null | head -12
```

**4단계.** 패키지를 설치한다.

```bash
sudo apt install -y tree htop
```

```bash
tree -L 2 ~/lab05
```

**5단계.** 설치된 파일과 소속 패키지를 조회한다.

```bash
dpkg -L tree
```

> 해당 패키지가 설치한 파일 목록이다.

```bash
which tree
```

```bash
dpkg -S $(which tree)
```

> 해당 파일이 어느 패키지에 속하는지 조회한다. `-L`과 `-S`는 조회 방향이 반대이며, RPM 계열의 `rpm -ql`, `rpm -qf`와 각각 대응한다.

**6단계.** 설치 목록을 확인한다.

```bash
dpkg -l | grep -E "^ii\s+(tree|htop)"
```

> `ii`는 정상 설치 상태를 의미한다.

```bash
dpkg -l | wc -l
```

**7단계.** `remove`와 `purge`의 차이를 확인한다.

`remove`와 `purge`의 차이는 **설정 파일이 존재하는 패키지**에서만 관찰된다. 따라서 설정 파일(`/etc/at.deny`, `/etc/pam.d/atd`)을 포함하는 `at` 패키지로 확인한다. `at`은 제3절에서 학습한 **1회성 작업 예약** 명령이기도 하다.

```bash
sudo apt install -y at
```

```bash
dpkg -L at | grep "^/etc"
```

> 이 패키지가 `/etc` 아래에 설치한 설정 파일 목록이다. 이 파일들의 존재 여부가 아래 두 명령의 차이를 만든다.

```bash
sudo apt remove -y at
```

```bash
dpkg -l | awk '$2 == "at"'
```

> 상태 코드가 `ii`(정상 설치됨)에서 **`rc`** 로 변경된다. `r`은 **r**emoved(프로그램 제거됨), `c`는 **c**onfig-files(설정 파일 잔존)를 의미한다.
>
> `grep at`이 아니라 `awk '$2 == "at"'`을 사용한 이유는, `grep`이 다른 패키지의 **설명문에 포함된 `at`** 까지 함께 찾아내기 때문이다. `awk`의 `$2`는 두 번째 필드, 즉 패키지 이름만을 정확히 지정한다.

```bash
dpkg -s at | grep -E "^(Package|Status)"
```

> `Status: deinstall ok config-files`로 표시된다. "프로그램은 제거하되 설정 파일은 유지한다"는 상태를 문장으로 확인할 수 있다.

```bash
ls -l /etc/at.deny
```

> 프로그램은 삭제되었으나 **설정 파일은 그대로 남아 있다.** 재설치 시 기존 설정을 그대로 이어서 사용할 수 있도록 보존한 것이다.

```bash
sudo apt purge -y at
```

```bash
dpkg -s at 2>/dev/null | grep -E "^(Package|Status)" || echo "완전히 제거되었습니다"
```

```bash
ls -l /etc/at.deny 2>/dev/null || echo "설정 파일까지 삭제되었습니다"
```

> **참고 — 설정 파일이 없는 패키지**
> 앞서 설치한 `tree`처럼 `/etc`에 설정 파일을 두지 않는 패키지는 보존할 설정 자체가 없으므로, `remove`만 실행하여도 `rc` 상태가 남지 않는다. 다음과 같이 직접 확인할 수 있다.
>
> ```bash
> dpkg -L tree | grep "^/etc" || echo "설정 파일 없음"
> sudo apt remove -y tree
> dpkg -l | awk '$2 == "tree"' | grep . || echo "rc 상태가 남지 않았다"
> ```
>
> 즉 `remove`와 `purge`의 구분은 **설정 파일을 가진 패키지에 한정하여 의미를 갖는다.**
{: .prompt-tip }

**8단계.** 압축 방식별 크기를 비교한다.

```bash
cd ~/lab05
```

```bash
mkdir -p src && cp /etc/services /etc/passwd src/
```

```bash
tar -cvf src.tar src > /dev/null
```

```bash
tar -czvf src.tar.gz src > /dev/null
```

```bash
tar -cjvf src.tar.bz2 src > /dev/null
```

```bash
ls -lh src.tar*
```

> `.tar`는 원본과 크기가 유사하고, 압축한 파일은 현저히 작다. 묶기와 압축이 별개의 작업임을 확인할 수 있다.

**9단계.** 정리한다.

```bash
sudo apt purge -y htop tree at && sudo apt autoremove -y
```

> 이미 제거된 패키지가 포함되어 있어도 오류 없이 진행된다.

```bash
rm -f src.tar src.tar.gz src.tar.bz2
```

```bash
rm -rf src
```

> **확인 사항** 설치 → 파일 조회 → `remove`(설정 파일을 가진 패키지에서 `rc` 상태) → `purge`(완전 삭제)의 흐름을 확인하였다면 성공이다.
{: .prompt-tip }

---
---

# 제5절. 종합 실습 — 서버 성능 저하 원인 분석 및 대응

---

> **실습 시나리오**
>
> 학습자는 서버 운영 담당자이다. 사용자로부터 "서버가 갑자기 느려졌다"는 신고가 접수되었다.
>
> 관리 부서의 지시는 다음과 같다.
>
> *"원인을 파악하여 조치하고, 재발 방지를 위해 자원 사용량을 주기적으로 감시하는 체계를 구성하시오. 처리 결과는 보고서로 제출하시오."*
>
> 본 실습에서는 제1절부터 제4절까지 학습한 내용을 활용하여 이 상황을 처리한다.
{: .prompt-info }

---

## 단계 1. 상황 재현

문제 상황을 재현하기 위해 CPU를 점유하는 프로세스를 생성한다.

```bash
mkdir -p ~/mission05 && cd ~/mission05
```

```bash
cat > cpu_load.sh << 'EOF'
#!/bin/bash
# 실습용 CPU 부하 발생 스크립트
while true; do
  echo "$RANDOM" > /dev/null
done
EOF
```

```bash
chmod +x cpu_load.sh
```

```bash
nohup ./cpu_load.sh > /dev/null 2>&1 &
```

```bash
nohup ./cpu_load.sh > /dev/null 2>&1 &
```

> 두 개의 프로세스를 백그라운드로 실행하였다.

---

## 단계 2. 부하 상태 확인

```bash
uptime
```

> 부하 평균이 상승한 것을 확인한다.

```bash
top -b -n 1 | head -12
```

---

## 단계 3. 원인 프로세스 식별

```bash
ps aux --sort=-%cpu | head -5
```

> CPU를 점유하는 프로세스가 `cpu_load.sh`임을 확인한다.

```bash
pgrep -a cpu_load
```

```bash
ps -ef | grep cpu_load | grep -v grep
```

> 실행 사용자(UID), 시작 시각(STIME), 부모 프로세스(PPID)를 확인한다.

---

## 단계 4. 1차 조치 — 우선순위 하향

즉시 종료하기 전에 우선순위를 낮추어 다른 작업에 자원을 양보하도록 한다.

```bash
for pid in $(pgrep -f cpu_load); do sudo renice -n 19 -p $pid; done
```

```bash
ps -eo pid,ni,comm | grep cpu_load
```

> `NI` 열이 `19`로 변경되었음을 확인한다. 스크립트를 실행하면 `comm` 열에는 인터프리터인 `bash`가 아니라 **스크립트 파일명(`cpu_load.sh`)** 이 표시된다.

---

## 단계 5. 2차 조치 — 프로세스 종료

**정상 종료를 먼저 시도하고, 실패한 경우에만 강제 종료한다.**

```bash
pkill -f cpu_load
```

```bash
sleep 2 && pgrep -a cpu_load || echo "정상 종료되었습니다"
```

> 만약 종료되지 않았다면 다음을 실행한다.

```bash
pkill -9 -f cpu_load 2>/dev/null; echo "처리 완료"
```

```bash
uptime
```

> 부하 평균이 점차 하락하는 것을 확인한다.

---

## 단계 6. 감시 도구 작성

```bash
cat > ~/mission05/monitor.sh << 'EOF'
#!/bin/bash
# 시스템 자원 감시 스크립트

LOG="$HOME/mission05/monitor.log"
LIMIT=50   # CPU 사용률 경고 기준(%)

{
  echo "----- $(date '+%F %T') -----"

  echo "부하 평균:$(uptime | awk -F'load average:' '{print $2}')"

  echo "CPU 사용 상위 3개:"
  ps -eo pcpu,pid,user,comm --sort=-pcpu | head -4 | tail -3 | \
    awk '{printf "  %6s%%  PID=%-7s %-10s %s\n", $1, $2, $3, $4}'

  echo "메모리: $(free -h | awk '/^Mem/ {print $3 " / " $2 " 사용"}')"
  echo "디스크: $(df -h / | tail -1 | awk '{print $5 " 사용"}')"

  TOP=$(ps -eo pcpu --sort=-pcpu --no-headers | head -1 | cut -d. -f1)
  if [ "${TOP:-0}" -ge "$LIMIT" ]; then
    echo "  [경고] CPU 사용률이 ${TOP}%에 도달하였습니다."
  fi
} >> "$LOG"

tail -12 "$LOG"
EOF
```

```bash
chmod 750 ~/mission05/monitor.sh
```

```bash
~/mission05/monitor.sh
```

---

## 단계 7. 감시 도구 자동 실행 등록

```bash
(crontab -l 2>/dev/null; echo "*/2 * * * * $HOME/mission05/monitor.sh > /dev/null 2>&1") | crontab -
```

```bash
crontab -l
```

> 2분 주기로 자동 감시하도록 등록하였다.

```bash
sleep 130 && tail -15 ~/mission05/monitor.log
```

---

## 단계 8. 처리 결과 보고서 작성

```bash
{
  echo "===== 서버 성능 저하 처리 보고서 ====="
  echo "작성자: $(whoami)"
  echo "작성일시: $(date '+%Y-%m-%d %H:%M')"
  echo
  echo "[1] 신고 내용"
  echo "  서버 응답 속도 저하"
  echo
  echo "[2] 원인 분석"
  echo "  cpu_load.sh 프로세스 2개가 CPU를 지속 점유"
  echo "  ps aux --sort=-%cpu 명령으로 식별"
  echo
  echo "[3] 조치 내역"
  echo "  1차: renice로 우선순위를 19로 하향 조정"
  echo "  2차: pkill(SIGTERM)로 정상 종료"
  echo
  echo "[4] 재발 방지 대책"
  echo "  monitor.sh 작성 후 cron에 2분 주기로 등록"
  echo
  echo "[5] 조치 후 시스템 상태"
  uptime | sed 's/^/  /'
  free -h | grep Mem | sed 's/^/  /'
  df -h / | tail -1 | sed 's/^/  /'
} > ~/mission05/report.txt
```

```bash
cat ~/mission05/report.txt
```

---

## 단계 9. 정리

```bash
crontab -r
```

```bash
pkill -f cpu_load 2>/dev/null; echo "정리 완료"
```

```bash
ls -l ~/mission05/
```

> **종합 실습 완료 기준**
> 1. 부하 상승을 `uptime`과 `top`으로 확인하였다.
> 2. `ps aux --sort=-%cpu`로 원인 프로세스를 식별하였다.
> 3. `renice`로 우선순위를 조정하고 `pkill`로 정상 종료하였다.
> 4. 감시 스크립트를 작성하여 `cron`에 등록하고 동작을 확인하였다.
> 5. 처리 보고서를 생성하였다.
> 6. 예약 작업과 실습 프로세스를 정리하였다.
{: .prompt-tip }

---
---

# 제6절. 자주 발생하는 오류와 대응 방법

---

| 화면에 출력된 메시지 / 증상 | 원인 | 대응 방법 |
|---|---|---|
| `No such process` | 해당 PID가 존재하지 않음 | `ps` 또는 `pgrep`으로 다시 확인한다. |
| `Operation not permitted` (kill) | 타 사용자의 프로세스 | `sudo kill`을 사용한다. |
| 좀비 프로세스가 사라지지 않음 | 부모가 회수하지 않음 | 부모 프로세스를 종료한다. `kill -9`는 효과가 없다. |
| `Failed to start` (systemctl) | 유닛 파일 오류 | `journalctl -u 서비스명`으로 원인을 확인한다. |
| 재부팅 후 서비스가 기동되지 않음 | `enable` 누락 | `sudo systemctl enable 서비스명` |
| cron 작업이 실행되지 않음 | `PATH` 제약 | 스크립트에서 절대 경로를 사용한다. |
| `Unable to acquire the dpkg lock` | 다른 apt 프로세스가 실행 중 | 종료를 기다린 후 재시도한다. |
| `E: Unable to locate package` | 패키지 목록이 오래됨 | `sudo apt update`를 먼저 실행한다. |
| `tar: 파일명: Cannot stat: No such file or directory` | `f` 옵션 위치 오류(`-cvfz` 형태) | `-czvf`와 같이 `f`를 마지막에 배치한다. `z`라는 이름으로 잘못 생성된 파일도 삭제한다. |

---
---

# 제7절. 이론 평가

---

**문항 1.** 리눅스에서 PID가 1번인 프로세스는?

① init 스크립트 ② **systemd** ③ kthreadd ④ bash

---

**문항 2.** 종료되었으나 부모 프로세스가 종료 상태를 회수하지 않아 프로세스 표에 잔존하는 상태는?

① 고아 프로세스 ② 데몬 ③ **좀비 프로세스** ④ 정지 프로세스

---

**문항 3.** `kill` 명령이 옵션 없이 전송하는 기본 시그널은?

① SIGHUP(1) ② SIGINT(2) ③ SIGKILL(9) ④ **SIGTERM(15)**

---

**문항 4.** 프로세스가 무시하거나 가로챌 수 **없는** 시그널은?

① SIGHUP ② SIGINT ③ SIGTERM ④ **SIGKILL**

---

**문항 5.** nice 값의 범위로 옳은 것은?

① 0 ~ 39 ② **-20 ~ 19** ③ -19 ~ 20 ④ 1 ~ 99

---

**문항 6.** 현재는 실행하지 않고 다음 부팅부터 자동으로 시작되도록 등록하는 명령은?

① `systemctl start 서비스`
② **`systemctl enable 서비스`**
③ `systemctl restart 서비스`
④ `systemctl mask 서비스`

---

**문항 7.** crontab의 시간 필드 순서로 옳은 것은?

① 시 분 일 월 요일
② **분 시 일 월 요일**
③ 분 시 월 일 요일
④ 요일 분 시 일 월

---

**문항 8.** `apt update` 명령이 수행하는 동작으로 옳은 것은?

① 설치된 모든 패키지를 최신 버전으로 갱신한다.
② **저장소의 패키지 목록 정보만 갱신한다.**
③ 사용하지 않는 패키지를 삭제한다.
④ 커널을 업그레이드한다.

---

**문항 9.** 패키지를 제거하면서 설정 파일까지 완전히 삭제하는 명령은?

① `apt remove 이름` ② **`apt purge 이름`** ③ `apt autoremove` ④ `dpkg -r 이름`

---

**문항 10.** 레드햇 계열에서 설치된 모든 패키지 목록을 조회하는 명령은?

① `rpm -ivh` ② **`rpm -qa`** ③ `rpm -e` ④ `rpm -ql`

---
---

# 제8절. 요약

---

## 8.1 핵심 개념 정리

| 구분 | 요점 |
|---|---|
| 프로세스 | 실행 중인 프로그램이며, `fork`/`exec`로 생성되어 부모-자식 계층을 이룬다. 최상위 부모는 PID 1의 `systemd`이다. |
| 조회 | `ps aux`(BSD)와 `ps -ef`(System V) 두 문법을 모두 알아야 하며, 실시간 감시는 `top`을 사용한다. |
| 시그널 | SIGTERM(15)을 먼저 전송하고, 응답이 없을 때에만 SIGKILL(9)을 사용한다. SIGKILL은 무시할 수 없으나 정리 작업의 기회도 제공하지 않는다. |
| 좀비 | 이미 종료된 프로세스이므로 `kill -9`로 제거할 수 없다. |
| 우선순위 | nice 값의 범위는 -20 ~ 19이며, 일반 사용자는 값을 높이는 방향으로만 조정할 수 있다. |
| 서비스 관리 | `start`는 현재 실행, `enable`은 부팅 시 자동 실행 등록으로 서로 독립적이다. |
| 작업 예약 | 반복은 `cron`, 1회는 `at`이다. cron 환경은 `PATH`가 제한적이므로 절대 경로를 사용한다. |
| 패키지 관리 | `update`는 목록 갱신, `upgrade`는 실제 갱신이다. `remove`는 설정 파일을 남기고 `purge`는 완전히 제거한다. |

---

## 8.2 본 강의에서 학습한 명령어

### 프로세스 관리

| 명령어 | 기능 |
|---|---|
| `ps aux` / `ps -ef` | 전체 프로세스 조회 |
| `top` | 실시간 감시 |
| `pstree` | 계층 구조 조회 |
| `pgrep` / `pkill` | 이름으로 검색 / 종료 |
| `kill` / `kill -9` | 정상 종료 / 강제 종료 |
| `jobs` / `fg` / `bg` | 작업 제어 |
| `nohup` | 로그아웃 후에도 유지 |
| `nice` / `renice` | 우선순위 조정 |

### 서비스와 예약

| 명령어 | 기능 |
|---|---|
| `systemctl start` / `stop` | 서비스 시작 / 중지 |
| `systemctl enable` / `disable` | 자동 시작 설정 / 해제 |
| `systemctl status` | 상태 조회 |
| `journalctl -u` | 서비스 로그 조회 |
| `crontab -e` / `-l` / `-r` | 반복 예약 등록 / 조회 / 삭제 |
| `at` / `atq` / `atrm` | 1회 예약 |

### 패키지 관리 대응표

| 데비안·우분투 | 레드햇 계열 | 기능 |
|---|---|---|
| `apt update` | `dnf check-update` | 목록 갱신 |
| `apt install` | `dnf install` | 설치 |
| `apt purge` | `dnf remove` | 완전 삭제 |
| `dpkg -l` | `rpm -qa` | 설치 목록 |
| `dpkg -L` | `rpm -ql` | 패키지의 파일 |
| `dpkg -S` | `rpm -qf` | 파일의 패키지 |

---

## 8.3 다음 강의 예고

제6강에서는 시스템을 네트워크에 연결하고 외부에 서비스를 제공하기 위한 **네트워크 설정, 방화벽, 원격 접속**을 학습하며, 리눅스의 그래픽 환경과 활용 분야를 함께 다룬다.
