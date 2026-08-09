---
title: 리눅스 기초 5강 종합문제 - 야간 장애 대응
date: 2026-09-22 09:30:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - 종합문제
  - 복습
  - 프로세스
  - 시그널
  - systemd
  - cron
  - apt
pin:
mermaid: false
---

> **종합문제 안내**
> 1. 본 글은 **제5강. 프로세스 관리와 소프트웨어 설치**의 복습용 종합문제이다.
> 2. 제5강의 실습 결과에 의존하지 않는다. 부하 상황과 서비스를 여기서 직접 만들어 내므로 **독립적으로 수행**할 수 있다.
> 3. 문제 4·5는 **의도적으로 CPU 부하를 발생**시킨다. 실습용 가상 머신에서만 수행하고, 마지막 정리 절차를 반드시 완료한다.
> 4. 문제 6은 systemd 유닛을 등록하고 문제 7은 cron에 작업을 등록한다. 두 항목 모두 정리 절차에서 제거한다.
{: .prompt-info }

| 문제 | 주제 | 대응 절 | 배정 시간 |
|---|---|---|---|
| 준비 | 실습 도구 생성 | — | 5분 |
| 문제 1 | 프로세스 조회와 계층 | 1.4 · 1.5 | 10분 |
| 문제 2 | 작업 제어 | 2.3 | 10분 |
| 문제 3 | 시그널의 차이 | 2.1 · 2.2 | 15분 |
| 문제 4 | 우선순위 조정 | 2.4 | 10분 |
| 문제 5 | 장애 원인 식별과 조치 | 1.4 · 2.2 | 15분 |
| 문제 6 | 서비스 등록과 자동 시작 | 3.1 · 3.2 | 20분 |
| 문제 7 | 작업 예약과 `PATH` | 3.3 | 15분 |
| 문제 8 | 패키지 관리 | 4.2 ~ 4.4 | 15분 |
| 마무리 | 자가 채점과 정리 | — | 10분 |

---
---

# 시나리오

---

학습자는 데이터센터 운영팀의 **야간 당직 담당자**이다. 새벽 두 시, 관제 시스템에서 다음과 같은 알림이 접수되었다.

> *"업무 시스템의 응답이 느립니다. 웹 화면이 열리는 데 10초 이상 걸린다는 신고가 세 건 접수되었습니다."*

당직 지침에 따르면 처리 절차는 다음과 같다.

| 단계 | 조치 |
|---|---|
| ① | 부하 상태를 수치로 확인한다 |
| ② | 원인 프로세스를 식별한다 |
| ③ | **즉시 종료하지 말고** 우선순위를 낮추어 서비스를 회복시킨다 |
| ④ | 정상 종료(SIGTERM)를 시도하고, 실패한 경우에만 강제 종료(SIGKILL)한다 |
| ⑤ | 재발 방지를 위한 감시 체계를 구성한다 |
| ⑥ | 처리 내역을 인수인계 보고서로 남긴다 |

본 종합문제는 이 절차를 그대로 따라간다.

---
---

# 준비 단계. 실습 도구 생성

---

아래 블록을 **한 번에 복사하여 실행**한다. 장애 상황을 재현할 도구와 시그널 실험용 스크립트가 생성된다.

```bash
mkdir -p ~/review05 && cd ~/review05

cat > cpu_hog.sh << 'EOF'
#!/bin/bash
# 실습용 CPU 부하 발생 도구
while true; do
  echo "$RANDOM" > /dev/null
done
EOF

cat > stubborn.sh << 'EOF'
#!/bin/bash
# SIGTERM을 의도적으로 무시하는 프로세스
trap 'echo "[$(date +%T)] SIGTERM 수신 - 무시함" >> "$HOME/review05/signal.log"' TERM
echo "[$(date +%T)] 시작 PID=$$" >> "$HOME/review05/signal.log"
while true; do sleep 1; done
EOF

chmod +x cpu_hog.sh stubborn.sh
echo "== 준비 완료 =="
ls -l
```

> `trap`은 특정 시그널을 받았을 때 수행할 동작을 지정하는 명령이다. 정상적인 서비스는 SIGTERM을 받으면 정리 작업 후 종료하도록 구현하지만, 여기서는 **학습을 위해 의도적으로 무시**하도록 작성하였다.

---
---

# 문제 1. 프로세스 조회와 계층

---

> **상황**
> 조치에 앞서 시스템의 프로세스 구조를 파악한다.
>
> **요구사항**
> 1. 현재 터미널에 연결된 프로세스만 조회하고, 이어서 **부모 프로세스 번호가 포함된 형식**으로 다시 조회한다.
> 2. 전체 프로세스를 **BSD 문법**과 **System V 문법** 두 가지로 각각 조회하고, 열 구성의 차이를 설명한다.
> 3. **PID 1번** 프로세스가 무엇인지 확인한다.
> 4. 자신의 셸을 기준으로 프로세스 계층을 트리 형태로 확인한다.
> 5. **메모리 사용량 상위 5개** 프로세스를 조회한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
ps
ps -f

ps aux | head -5
ps -ef | head -5

ps -p 1 -o pid,ppid,user,comm

pstree -p $$

ps aux --sort=-%mem | head -6
ps aux --sort=-%cpu | head -6
```

**해설**

- 옵션 없는 `ps`는 **현재 터미널에 연결된 프로세스만** 보여 준다. 통상 `bash`와 `ps` 두 개가 출력된다.
- **`ps aux`(BSD 문법, 하이픈 없음)** 는 `%CPU`·`%MEM`·`STAT` 열을 포함하여 **자원 사용률** 파악에 적합하고, **`ps -ef`(System V 문법, 하이픈 사용)** 는 `PPID` 열을 포함하여 **부모-자식 관계** 파악에 적합하다. 두 문법이 공존하는 것은 유닉스의 두 계보가 합쳐진 역사적 이유 때문이며, 시험에서는 양쪽을 모두 묻는다.
- PID 1번은 **`systemd`** 이며 모든 프로세스의 최상위 부모이다. 부모가 먼저 종료된 고아 프로세스는 systemd가 입양한다.
- `--sort=-%mem`의 앞선 `-`는 **내림차순**을 의미한다.
- `STAT` 열의 값은 `R`(실행), `S`(대기), `D`(중단 불가 대기), `T`(정지), `Z`(좀비)이다. 대부분의 프로세스가 `S` 상태라는 점을 확인한다.

</details>

**완료 기준** — `ps aux`와 `ps -ef`의 열 구성 차이를 설명할 수 있고, PID 1이 `systemd`임을 확인하였다.

---
---

# 문제 2. 작업 제어

---

> **상황**
> 장시간 걸리는 작업을 다루려면 포그라운드와 백그라운드를 전환할 수 있어야 한다.
>
> **요구사항**
> 1. `sleep 300`과 `sleep 400`을 **백그라운드로** 실행한다.
> 2. 작업 목록을 **PID와 함께** 조회한다.
> 3. 1번 작업을 포그라운드로 가져온 뒤 **정지**시킨다.
> 4. 정지된 작업을 다시 **백그라운드에서 재개**한다.
> 5. 두 작업을 작업 번호로 종료한다.
> 6. 로그아웃 후에도 실행을 유지하려면 어떤 명령을 사용하는지 설명한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sleep 300 &
sleep 400 &

jobs -l

fg %1
# 이 상태에서 Ctrl + Z 를 누른다

jobs
bg %1
jobs

kill %1
kill %2
jobs
```

**해설**

- 명령 끝의 `&`는 백그라운드 실행을 의미하며, 터미널을 즉시 반환하므로 다른 작업을 이어서 수행할 수 있다.
- `jobs -l`의 `[1]`, `[2]`는 **작업 번호**, 그 옆의 숫자가 **PID**이다. `kill %1`처럼 작업 번호를 쓸 때는 앞에 `%`를 붙인다.
- `Ctrl + Z`는 SIGTSTP(20)를 보내어 작업을 **정지(`Stopped`)** 시키고, `bg`는 SIGCONT(18)로 백그라운드에서 **재개(`Running`)** 한다.
- `nohup 명령 &`은 **no h**ang**up**의 약어로, 접속이 끊길 때 전달되는 SIGHUP을 무시하게 한다. SSH로 접속하여 장시간 작업을 시작할 때 반드시 사용한다.

</details>

**완료 기준** — 작업 상태가 `Running → Stopped → Running`으로 변하는 것을 확인하였다.

---
---

# 문제 3. 시그널의 차이

---

> **상황**
> "종료 명령을 보냈는데 프로세스가 죽지 않는다"는 상황을 실험으로 확인한다.
>
> **요구사항**
> 1. `stubborn.sh`를 백그라운드로 실행하고 PID를 확인한다.
> 2. **기본 종료 시그널**을 전송한 뒤 프로세스가 살아 있는지 확인한다.
> 3. `signal.log`를 열어 프로세스가 시그널을 **받고도 무시했음**을 확인한다.
> 4. **강제 종료 시그널**을 전송하고 결과를 확인한다.
> 5. 로그에 아무것도 추가되지 않은 이유를 설명한다.
> 6. 시그널 번호와 이름의 목록을 조회한다.
{: .prompt-info }

**힌트** — 기본 시그널은 15번, 강제 종료는 9번이다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cd ~/review05
./stubborn.sh &
PID=$! && echo "PID = $PID"

kill $PID
sleep 2
ps -p $PID -o pid,stat,comm

cat signal.log

kill -9 $PID
sleep 1
ps -p $PID -o pid,stat,comm || echo "프로세스가 종료되었습니다"

cat signal.log
kill -l | head -5
```

**해설**

| 구분 | SIGTERM(15) | SIGKILL(9) |
|---|---|---|
| 의미 | 정리 후 종료할 것을 **요청** | 즉시 **강제 종료** |
| 프로세스의 대응 | 임시 파일 삭제, 버퍼 기록 등 정리 작업 수행 가능 | **아무 작업도 수행하지 못함** |
| 가로채기 | `trap`으로 가능 | **불가능** |
| 결과 | 정상 종료 | 데이터 손상 가능 |

- `kill`은 옵션이 없으면 **SIGTERM(15)** 을 보낸다. 이는 "종료해 달라"는 **요청**이며 강제력이 없다. 본 실습의 스크립트는 `trap`으로 이를 가로채어 무시하였다.
- **SIGKILL(9)은 커널이 직접 처리**하므로 프로세스가 반응할 기회조차 없다. 그래서 로그에 아무것도 남지 않는다. 데이터베이스처럼 쓰기 작업 중인 프로세스를 SIGKILL로 종료하면 자료가 손상될 수 있다.
- 따라서 **먼저 `kill`(SIGTERM)을 보내고, 일정 시간이 지나도 응답이 없을 때에만 `kill -9`** 를 사용한다.
- `$!`는 **가장 최근에 백그라운드로 실행한 프로세스의 PID**를 담은 특수 변수이다.
- 참고로 **좀비 프로세스는 `kill -9`로 제거할 수 없다.** 이미 종료된 프로세스이므로 시그널이 의미를 갖지 않으며, 부모가 상태를 회수하거나 부모가 종료되어야 사라진다.

</details>

**완료 기준** — SIGTERM 후에는 프로세스가 살아 있고 로그에 "무시함"이 기록되며, SIGKILL 후에는 프로세스가 사라지고 로그에 추가 기록이 없다.

---
---

# 문제 4. 우선순위 조정

---

> **상황**
> 당직 지침 ③에 따라, 종료하기 전에 우선순위를 낮추어 다른 작업에 자원을 양보하도록 한다.
>
> **요구사항**
> 1. `sleep 300`을 실행하고 기본 nice 값을 확인한다.
> 2. nice 값 `10`으로 새 프로세스를 시작하여 값이 반영되었는지 확인한다.
> 3. nice 값 `-5`로 시작을 시도하고 **무슨 일이 일어나는지** 확인한다.
> 4. 실행 중인 프로세스의 nice 값을 `15`로 변경한다.
> 5. 변경한 값을 다시 `5`로 **낮추려고 시도**하고 결과를 확인한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sleep 300 &
ps -o pid,ni,comm -p $!

nice -n 10 sleep 300 &
ps -o pid,ni,comm -p $!

nice -n -5 sleep 300 &
ps -o pid,ni,comm -p $!

sleep 400 &
PID3=$! && renice -n 15 -p $PID3
ps -o pid,ni,comm -p $PID3

renice -n 5 -p $PID3
```

**해설**

- **nice 값의 범위는 `-20 ~ 19`** 이며 기본값은 `0`이다. **값이 작을수록 우선순위가 높다.** 이 범위는 시험에 반복 출제된다.
- 명칭은 '양보하다'라는 의미에서 유래하였다. 값이 클수록 다른 프로세스에 CPU를 잘 양보한다는 뜻이므로, **값이 크면 우선순위가 낮다.**
- `nice -n -5`는 일반 사용자에게 거부된다. 경고가 출력되고 프로세스는 **nice 값 0으로 실행**된다. 마지막 `renice -n 5`도 실패한다.
- 즉 **일반 사용자는 우선순위를 낮추는 방향(값을 높이는 방향)으로만 조정할 수 있다.** 특정 사용자가 자원을 독점하는 것을 막기 위한 제약이며, 우선순위를 높이려면 `sudo`가 필요하다.

</details>

**완료 기준** — `nice -n 10`은 적용되고 `-5`는 거부되며, `renice`로 값을 낮추는 시도가 실패한다.

---
---

# 문제 5. 장애 원인 식별과 조치

---

> **상황**
> 이제 실제 장애 상황을 재현하여 당직 절차 ①~④를 수행한다.
>
> **요구사항**
> 1. 부하 발생 도구를 **두 개** 백그라운드로 실행한다(접속이 끊겨도 유지되도록 실행한다).
> 2. **1분 이상 기다린 뒤** 부하 평균을 확인한다.
> 3. CPU를 점유하는 상위 프로세스를 조회하여 원인을 식별한다.
> 4. 원인 프로세스의 **실행 사용자·시작 시각·부모 프로세스**를 확인한다.
> 5. 우선순위를 최하로 낮춘다(조치 ③).
> 6. 정상 종료를 시도하고, 실패한 경우에만 강제 종료한다(조치 ④).
> 7. 처리 후 부하가 회복되는지 확인한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cd ~/review05
nohup ./cpu_hog.sh > /dev/null 2>&1 &
nohup ./cpu_hog.sh > /dev/null 2>&1 &

uptime
sleep 70
uptime

top -b -n 1 | head -12
ps aux --sort=-%cpu | head -5
pgrep -a cpu_hog
ps -ef | grep cpu_hog | grep -v grep

for pid in $(pgrep -f cpu_hog); do sudo renice -n 19 -p $pid; done
ps -eo pid,ni,comm | grep cpu_hog

pkill -f cpu_hog
sleep 2
pgrep -a cpu_hog || echo "정상 종료되었습니다"

pkill -9 -f cpu_hog 2>/dev/null; echo "처리 완료"
uptime
```

**해설**

- **부하 평균은 즉시 오르지 않는다.** `uptime`이 보여 주는 세 값은 최근 **1분·5분·15분** 동안 실행을 대기한 프로세스의 평균 개수이므로, 부하를 시작한 직후에는 거의 변하지 않는다. 최소 1분은 기다린 뒤 판단하여야 한다.
- **CPU 코어 수보다 큰 부하 평균이 지속되면 과부하**로 판정한다. 코어가 2개인데 부하 평균이 4를 넘으면 작업이 적체되고 있는 것이다.
- `top -b -n 1`의 `-b`(batch)는 대화형 화면 대신 텍스트로 한 번만 출력하므로 파일 저장과 파이프 연결에 사용한다.
- `ps -eo pid,ni,comm | grep cpu_hog`에서 `comm` 열에는 인터프리터인 `bash`가 아니라 **스크립트 파일명**이 표시된다.
- `pkill`은 기본적으로 SIGTERM을 보낸다. `-f`는 명령행 전체에서 패턴을 찾으라는 의미이며, 스크립트처럼 인터프리터로 실행되는 프로세스를 잡을 때 필요하다.
- **종료 순서를 지키는 것이 핵심이다.** 우선순위 하향으로 서비스를 먼저 회복시키고, 정상 종료를 시도한 뒤, 그래도 남아 있을 때만 강제 종료한다.
- 부하 평균은 종료 후에도 **서서히** 내려간다. 평균값이므로 즉시 0이 되지 않는다.

</details>

**완료 기준** — `pgrep -a cpu_hog`가 아무것도 출력하지 않는다.

---
---

# 문제 6. 서비스 등록과 자동 시작

---

> **상황**
> 재발 방지를 위해 상태를 주기적으로 기록하는 감시 서비스를 등록하여야 한다.
>
> **요구사항**
> 1. 30초마다 부하 평균을 기록하는 서비스 유닛 `lab-watch.service`를 작성한다.
> 2. systemd가 새 유닛을 인식하도록 한다.
> 3. **등록 전 자동 시작 여부**를 확인한다.
> 4. 서비스를 **지금 시작**한 뒤, 실행 여부와 자동 시작 여부를 각각 조회하여 **두 값이 다름**을 확인한다.
> 5. 로그를 조회한다.
> 6. 자동 시작을 등록하고, `enable`의 실제 동작이 무엇인지 확인한다.
{: .prompt-info }

**힌트** — `is-active`와 `is-enabled`는 서로 다른 것을 조회한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sudo tee /etc/systemd/system/lab-watch.service > /dev/null << 'EOF'
[Unit]
Description=Lab Load Average Watcher

[Service]
Type=simple
ExecStart=/bin/bash -c 'while true; do echo "load:$(cut -d" " -f1-3 /proc/loadavg)"; sleep 30; done'
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload

systemctl is-enabled lab-watch
sudo systemctl start lab-watch
systemctl is-active lab-watch
systemctl is-enabled lab-watch

journalctl -u lab-watch -n 5 --no-pager

sudo systemctl enable lab-watch
systemctl is-enabled lab-watch
ls -l /etc/systemd/system/multi-user.target.wants/ | grep lab-watch
```

**해설**

- 유닛 파일은 세 절로 구성된다. `[Unit]`은 설명과 의존성, `[Service]`는 실행 방법, `[Install]`은 자동 시작 대상을 정의한다.
- 유닛 파일을 새로 만들거나 수정한 뒤에는 반드시 **`systemctl daemon-reload`** 로 systemd에 알려야 한다.
- **`start`와 `enable`은 서로 독립적이다.**
>
> | 구분 | `start` | `enable` |
> |---|---|---|
> | 의미 | **현재 시점에 실행** | **다음 부팅부터 자동 실행되도록 등록** |
> | 조회 | `is-active` | `is-enabled` |
> | 재부팅 후 | 중지 상태 | 자동 실행 |
>
- 따라서 `start`만 하고 `enable`을 누락하면 **재부팅 후 서비스가 기동되지 않는다.** 실무에서 매우 빈번하게 발생하는 실수이며 시험에도 자주 출제된다.
- `enable`의 실제 동작은 `[Install]` 절이 지정한 대상 디렉터리(`multi-user.target.wants`)에 **심볼릭 링크를 생성하는 것**이다. 목록을 조회하면 링크를 직접 확인할 수 있다.
- 서비스의 출력은 화면이 아니라 **저널**에 기록되므로 `journalctl -u 유닛명`으로 조회한다.

</details>

**완료 기준** — `is-active`가 `active`, `is-enabled`가 `enabled`이고 `multi-user.target.wants`에 링크가 존재한다.

---
---

# 문제 7. 작업 예약과 `PATH`

---

> **상황**
> 매 분 상태를 기록하는 예약 작업을 등록하고, "수동으로는 되는데 cron에서는 안 된다"는 현상의 원인을 직접 확인한다.
>
> **요구사항**
> 1. 실행 시각과 `PATH`를 기록하는 스크립트를 작성한다.
> 2. **수동으로 한 번 실행**하여 기록되는 `PATH`를 확인한다.
> 3. 매 분 실행되도록 crontab에 등록한다(기존 등록 항목을 지우지 않도록 유의한다).
> 4. 2분 이상 기다린 뒤 로그를 확인하고, **수동 실행 시와 cron 실행 시의 `PATH`가 어떻게 다른지** 비교한다.
> 5. cron의 실행 기록을 시스템 로그에서 확인한다.
> 6. 예약을 삭제한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cat > ~/review05/watch_log.sh << 'EOF'
#!/bin/bash
echo "[$(date '+%F %T')] PATH=$PATH" >> "$HOME/review05/cron.log"
EOF
chmod +x ~/review05/watch_log.sh

~/review05/watch_log.sh
cat ~/review05/cron.log

(crontab -l 2>/dev/null; echo "* * * * * $HOME/review05/watch_log.sh") | crontab -
crontab -l

sleep 130
cat ~/review05/cron.log

grep CRON /var/log/syslog | tail -5

crontab -r
crontab -l 2>/dev/null || echo "예약 삭제 완료"
```

**해설**

- **`(crontab -l 2>/dev/null; echo "새 항목") | crontab -`** 은 기존 등록 내용을 보존한 채 한 줄을 추가하는 관용적 표현이다. `crontab -e`로 직접 편집하여도 무방하지만, 이 형태는 스크립트로 자동화할 수 있다.
- 시간 필드는 앞에서부터 **분·시·일·월·요일** 순이다. `* * * * *`은 매 분을 의미한다.
- 로그를 비교하면 **cron이 사용하는 `PATH`가 수동 실행 시보다 현저히 짧다는 사실**이 드러난다. cron은 로그인 셸을 거치지 않으므로 `~/.profile`과 `~/.bashrc`가 적용되지 않기 때문이다.
- 이것이 **"수동으로는 되는데 cron에 등록하면 동작하지 않는다"는 문제의 가장 흔한 원인**이다. 해결책은 두 가지이다.
  - 스크립트 안에서 명령을 **절대 경로**로 지정한다.
  - crontab 상단에 `PATH=...`를 명시한다.
- `crontab -r`은 해당 사용자의 crontab을 **전부** 삭제한다. 일부만 지우려면 `crontab -e`로 해당 행만 제거한다.

</details>

**완료 기준** — `cron.log`에 수동 실행분과 cron 실행분이 함께 기록되어 있고 두 `PATH`가 서로 다르다.

---
---

# 문제 8. 패키지 관리

---

> **상황**
> 감시 도구를 보강하기 위해 패키지를 설치하고, 불필요한 패키지를 정리하여야 한다.
>
> **요구사항**
> 1. 저장소 목록을 갱신하고 갱신 가능한 패키지 수를 확인한다.
> 2. `tree`와 `at` 두 패키지를 설치한다.
> 3. `tree`가 설치한 파일 목록과, `tree` 실행 파일이 **어느 패키지에 속하는지**를 각각 조회한다.
> 4. 두 패키지 중 **설정 파일을 가진 쪽이 무엇인지** 확인한다.
> 5. `at`을 `remove`한 뒤 상태 코드를 확인하고, 이어서 `purge`한 뒤 다시 확인한다.
> 6. 레드햇 계열에서 같은 작업을 하려면 어떤 명령을 쓰는지 대응시킨다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sudo apt update
apt list --upgradable 2>/dev/null | head -5

sudo apt install -y tree at

dpkg -L tree | head -10
which tree
dpkg -S $(which tree)

dpkg -L tree | grep "^/etc" || echo "tree : 설정 파일 없음"
dpkg -L at   | grep "^/etc"

sudo apt remove -y at
dpkg -l | awk '$2 == "at"'
dpkg -s at | grep -E "^(Package|Status)"
ls -l /etc/at.deny

sudo apt purge -y at
dpkg -s at 2>/dev/null | grep -E "^(Package|Status)" || echo "완전히 제거되었습니다"
```

**해설**

- **`update`와 `upgrade`의 구분**이 중요하다. `apt update`는 저장소의 **목록 정보만** 갱신하고, `apt upgrade`가 실제로 패키지를 갱신한다. 항상 `update`를 먼저 실행한다.
- `dpkg -L`(패키지 → 파일)과 `dpkg -S`(파일 → 패키지)는 **조회 방향이 서로 반대**이며, 각각 RPM 계열의 `rpm -ql`, `rpm -qf`에 대응한다.
- **`remove`와 `purge`의 차이는 설정 파일을 가진 패키지에서만 관찰된다.** `tree`는 `/etc`에 파일을 두지 않으므로 차이가 드러나지 않지만, `at`은 `/etc/at.deny` 등을 설치하므로 `remove` 후 상태 코드가 **`rc`**(**r**emoved + **c**onfig-files)로 남는다.
- 상태 조회 시 `grep at`이 아니라 **`awk '$2 == "at"'`** 을 사용하는 이유는, `grep`이 다른 패키지의 **설명문에 포함된 `at`** 까지 찾아내기 때문이다. `$2`는 패키지 이름 필드를 정확히 지정한다.
- 계열별 대응은 다음과 같다.
>
> | 작업 | 데비안·우분투 | 레드햇 계열 |
> |---|---|---|
> | 설치 | `apt install` | `dnf install` / `rpm -ivh` |
> | 삭제 | `apt remove` | `dnf remove` / `rpm -e` |
> | 전체 목록 | `dpkg -l` | `rpm -qa` |
> | 패키지의 파일 | `dpkg -L` | `rpm -ql` |
> | 파일의 패키지 | `dpkg -S` | `rpm -qf` |

</details>

**완료 기준** — `at`의 상태가 `ii → rc → 삭제됨`으로 변하는 과정을 확인하였다.

---
---

# 마무리. 자가 채점

---

```bash
echo "===== 자가 채점 결과 ====="

pgrep -f cpu_hog > /dev/null 2>&1 \
  && echo "[문제 5] 미완료 — 부하 프로세스가 아직 남아 있음" \
  || echo "[문제 5] 통과 — 부하 프로세스 정리됨"

grep -q "무시함" ~/review05/signal.log 2>/dev/null \
  && echo "[문제 3-1] 통과 — SIGTERM 무시 기록 확인" || echo "[문제 3-1] 미완료"

[ "$(grep -c '무시함' ~/review05/signal.log 2>/dev/null)" -ge 1 ] \
  && [ "$(tail -1 ~/review05/signal.log 2>/dev/null | grep -c '무시함')" -eq 1 ] \
  && echo "[문제 3-2] 통과 — SIGKILL 이후 추가 기록 없음" || echo "[문제 3-2] 확인 필요"

systemctl is-active lab-watch > /dev/null 2>&1 \
  && echo "[문제 6-1] 통과 — 서비스 실행 중" || echo "[문제 6-1] 미완료"

systemctl is-enabled lab-watch > /dev/null 2>&1 \
  && echo "[문제 6-2] 통과 — 자동 시작 등록" || echo "[문제 6-2] 미완료"

[ "$(wc -l < ~/review05/cron.log 2>/dev/null)" -ge 2 ] \
  && echo "[문제 7] 통과 — cron 실행 기록 누적" || echo "[문제 7] 미완료"

dpkg -l | awk '$2 == "tree"' | grep -q "^ii" \
  && echo "[문제 8] 통과 — 패키지 설치 확인" || echo "[문제 8] 미완료"
```

---

## 이론 점검

**문항 1.** `kill`을 옵션 없이 실행하면 어떤 시그널이 전송되는가?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**SIGTERM(15)** 이다. "정리한 뒤 종료해 달라"는 요청이며 프로세스가 가로채거나 무시할 수 있다.

</details>

**문항 2.** 좀비 프로세스를 `kill -9`로 제거할 수 있는가?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**제거할 수 없다.** 좀비는 이미 종료된 프로세스이므로 시그널이 의미를 갖지 않는다. 부모가 종료 상태를 회수하거나 부모가 종료되어야 사라진다.

</details>

**문항 3.** nice 값의 범위와, 값이 클 때 우선순위는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

범위는 **-20 ~ 19**이며 **값이 클수록 우선순위가 낮다.** 일반 사용자는 값을 높이는 방향으로만 조정할 수 있다.

</details>

**문항 4.** `systemctl start`만 실행하고 `enable`을 하지 않으면 어떻게 되는가?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**현재는 실행되지만 재부팅 후에는 기동되지 않는다.** `start`는 현재 실행, `enable`은 자동 시작 등록으로 서로 독립적이다.

</details>

**문항 5.** cron에 등록한 스크립트가 수동 실행과 달리 실패하는 가장 흔한 원인은?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**`PATH`의 차이**이다. cron은 로그인 셸을 거치지 않아 `PATH`가 매우 짧으므로, 스크립트 안에서 명령을 절대 경로로 지정하거나 crontab 상단에 `PATH=`를 명시하여야 한다.

</details>

**문항 6.** `apt remove`와 `apt purge`의 차이는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

`remove`는 프로그램만 제거하고 **설정 파일을 남기며**(상태 `rc`), `purge`는 설정 파일까지 완전히 제거한다. 단, 설정 파일이 없는 패키지에서는 차이가 드러나지 않는다.

</details>

---
---

# 정리 절차 — 반드시 수행

---

```bash
pkill -f cpu_hog 2>/dev/null; pkill -f stubborn.sh 2>/dev/null; echo "프로세스 정리"
```

```bash
crontab -r 2>/dev/null; crontab -l 2>/dev/null || echo "예약 작업 정리 완료"
```

```bash
sudo systemctl stop lab-watch
sudo systemctl disable lab-watch
sudo rm -f /etc/systemd/system/lab-watch.service
sudo systemctl daemon-reload
```

```bash
sudo apt purge -y tree at 2>/dev/null; sudo apt autoremove -y
```

```bash
rm -rf ~/review05
```

**정리 결과를 검증한다.**

```bash
pgrep -a cpu_hog || echo "① 부하 프로세스 없음"
```

```bash
systemctl list-unit-files | grep lab-watch || echo "② 실습 서비스 없음"
```

```bash
crontab -l 2>/dev/null || echo "③ 예약 작업 없음"
```

> 세 항목이 모두 "없음"으로 출력되면 정리가 완료된 것이다. 특히 **부하 프로세스를 남겨 두면 가상 머신이 계속 CPU를 소모**하므로 반드시 확인한다.
{: .prompt-danger }

---

## 자기 점검

```
 [ ] ps aux와 ps -ef의 차이를 설명할 수 있다.
 [ ] jobs · fg · bg · Ctrl+Z로 작업 상태를 전환할 수 있다.
 [ ] SIGTERM과 SIGKILL의 차이를 실험 결과로 설명할 수 있다.
 [ ] nice 값의 범위와 일반 사용자의 제약을 설명할 수 있다.
 [ ] 부하 평균이 즉시 반영되지 않는 이유를 설명할 수 있다.
 [ ] start와 enable의 차이를 is-active · is-enabled로 확인하였다.
 [ ] cron의 PATH가 짧다는 사실을 로그로 확인하였다.
 [ ] remove와 purge의 차이가 나타나는 조건을 설명할 수 있다.
 [ ] 정리 절차를 완료하여 부하·서비스·예약을 모두 제거하였다.
```

다음 복습은 **제6강 종합문제 — 지사 서버 인수와 서비스 공개**이다.
