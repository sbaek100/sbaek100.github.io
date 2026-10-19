---
title: "[웹 인프라 구축] 08. 7단계 — DB 복제 (Primary–Standby 이중화)"
date: 2026-08-10 09:16:00 +0900
categories:
  - 1.응용강의
  - 웹인프라구축
tags:
  - MariaDB
  - Replication
  - 바이너리로그
  - mysqldump
  - 고가용성
math: false
mermaid: true
---

> 이번 글은 **직접 따라 하는 실습**입니다. Standby DB `db2`를 만들어 MariaDB **복제(Replication)** 를 겁니다. Primary에 쓴 데이터가 실시간으로 Standby에 따라 써지는 것을 확인하고, Primary 장애 시 **수동 승격**까지 연습합니다.
{: .prompt-info }

**이번에 켤 VM**: `db1`, `db2`(새로 만듦), `web1` (확인용 — 프록시·web2·redis는 오늘 안 켜도 됩니다)

## 0. 이번 단계의 그림

```mermaid
flowchart LR
    W["web1 / web2<br/>(읽기·쓰기)"] --> D1["db1 · Primary<br/>:3306"]
    D1 -- "변경 내역(바이너리 로그)<br/>실시간 전송" --> D2["db2 · Standby<br/>:3306 (읽기 전용)"]
```

| 오늘 배우는 것 | 설명 |
|---|---|
| 바이너리 로그(binlog) | Primary가 모든 데이터 변경을 순서대로 기록하는 파일 (비유: 변경 일기장) |
| 복제(Replication) | Standby가 그 기록을 실시간으로 수신해 동일하게 반영하는 것 |
| `mysqldump` | 복제 시작 전 "현재 상태 통째 복사"용 백업 도구 |
| 승격(Promotion) | Primary 장애 시 Standby를 새 Primary로 올리는 절차 |

**왜 백업 파일로는 부족한가?** 매일 밤 백업하면 "어제까지"만 복구할 수 있습니다. 복제는 **초 단위로 반영되는 살아 있는 사본**이라, 장애 시 데이터 손실이 거의 없습니다. (그리고 복제는 백업을 대체하지 않습니다 — `DELETE` 실수도 그대로 복제되기 때문입니다. 실무에서는 반드시 둘 다 운영합니다.)

> **★ 꼭 알아 두어야 할 개념 — RPO와 RTO**: 장애 대비 설계는 두 수치로 평가합니다. **RPO(Recovery Point Objective, 복구 시점 목표)** 는 "데이터를 최대 얼마 전 시점까지 잃어도 되는가"(하루 1회 백업이면 RPO ≈ 24시간, 실시간 복제면 ≈ 0초), **RTO(Recovery Time Objective, 복구 시간 목표)** 는 "서비스를 얼마 만에 복구해야 하는가"입니다. 오늘의 복제는 **RPO를 0에 가깝게** 만드는 작업이고, 승격 절차의 숙련도가 **RTO**를 결정합니다.
{: .prompt-tip }

---

## 1. db1을 Primary로 설정

`db1`에서 설정 파일을 수정합니다.

```bash
ssh infra@192.168.100.21
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

`[mysqld]` 섹션 아래에 (bind-address 근처) 다음을 추가/수정합니다:

```ini
server-id        = 1
log_bin          = /var/log/mysql/mysql-bin.log
binlog_do_db     = appdb
```

```bash
sudo systemctl restart mariadb
```

- `server-id` : 복제에 참여하는 서버의 **고유 번호**. Primary=1, Standby=2로 약속합니다.
- `log_bin` : 바이너리 로그(변경 일기장) 켜기 — 복제의 원천입니다.

### 1.1 복제 전용 계정 만들기

Standby가 일기장을 받아 갈 때 쓸 **최소 권한 계정**입니다.

```bash
sudo mysql
```

```sql
CREATE USER 'repl'@'192.168.100.22' IDENTIFIED BY 'Repl123!';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.100.22';
FLUSH PRIVILEGES;
EXIT;
```

접속 허용 호스트를 **db2의 IP로만** 못박았고, 권한도 복제 수신 하나뿐입니다. "계정마다 꼭 필요한 권한만" — 최소 권한 원칙입니다.

---

## 2. db2 만들기

**1편 7장의 표준 절차**(복제 → 호스트명 → 고정 IP → 확인 → SSH)로: 이름 `db2` / IP `192.168.100.22` / 메모리 1024 MB. 이후:

```bash
ssh infra@192.168.100.22
sudo apt update && sudo apt install -y mariadb-server
sudo mysql_secure_installation    # 답변은 3편과 동일
```

Standby 설정:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

```ini
bind-address = 0.0.0.0
server-id    = 2
read_only    = 1
```

```bash
sudo systemctl restart mariadb
```

- `server-id = 2` — **Primary와 달라야 합니다.** (같으면 복제가 거부됩니다)
- `read_only = 1` — Standby에 실수로 쓰는 것을 막습니다. 사본에 직접 쓰면 두 DB가 어긋납니다.

방화벽도 서로 열어 줍니다:

```bash
# db2에서 — 나중에 승격 대비, web들도 미리 허용
sudo ufw allow from 192.168.100.11 to any port 3306 proto tcp
sudo ufw allow from 192.168.100.12 to any port 3306 proto tcp
```

```bash
# db1에서 — Standby(db2)가 일기장을 받으러 온다
ssh infra@192.168.100.21
sudo ufw allow from 192.168.100.22 to any port 3306 proto tcp
```

### 2.1 db2에도 웹용 계정 만들기 (승격 대비)

한 가지 함정: 잠시 뒤 가져올 백업에는 `appdb` 데이터만 있고 **접속 계정(mysql 시스템 DB)은 포함되지 않습니다.** 계정이 없으면 나중에 db2를 Primary로 승격해도 웹 서버가 로그인하지 못합니다. Standby에는 **Primary와 같은 접속 계정을 미리** 만들어 둡니다.

`db2`에서:

```bash
sudo mysql
```

```sql
CREATE USER 'app'@'192.168.100.%' IDENTIFIED BY 'App123!';
GRANT ALL PRIVILEGES ON appdb.* TO 'app'@'192.168.100.%';
FLUSH PRIVILEGES;
EXIT;
```

---

## 3. 현재 데이터 이사 + 복제 시작

복제는 "지금부터의 변경"만 따라 씁니다. 그래서 먼저 **현재 상태를 통째로 복사**하고, "**일기장의 몇 페이지부터 읽으면 되는지**"를 정확히 맞춰야 합니다.

### 3.1 Primary에서 백업 뜨기 (좌표 포함)

`db1`에서:

```bash
sudo mysqldump --master-data=2 --single-transaction --databases appdb > /tmp/appdb.sql
head -30 /tmp/appdb.sql | grep "CHANGE MASTER"
```

출력 예:

```
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=1234;
```

이 **파일 이름과 숫자(좌표)를 메모**하세요. `--master-data=2` 옵션이 "이 백업이 일기장의 어느 지점 스냅샷인지"를 백업 파일에 적어 준 것입니다.

백업을 db2로 전송:

```bash
scp /tmp/appdb.sql infra@db2:/tmp/
```

### 3.2 Standby에 복원 + 복제 연결

`db2`에서:

```bash
sudo mysql < /tmp/appdb.sql
sudo mysql
```

메모한 좌표를 넣어 복제를 연결합니다. (파일명·숫자는 **본인 것**으로!)

```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.100.21',
  MASTER_USER='repl',
  MASTER_PASSWORD='Repl123!',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=1234;

START SLAVE;
SHOW SLAVE STATUS\G
```

`SHOW SLAVE STATUS\G` 출력에서 이 세 줄만 보면 됩니다:

```
Slave_IO_Running:  Yes     ← 일기장을 받아오는 중
Slave_SQL_Running: Yes     ← 받아온 걸 따라 쓰는 중
Seconds_Behind_Master: 0   ← Primary보다 몇 초 뒤처졌나
```

**Yes / Yes / 0** 이면 복제 완성입니다.

---

## 4. 동작 확인 — 데이터가 따라 써지는 순간

### 4.1 실시간 복제 확인

`db1`(Primary)에서 데이터를 넣고:

```bash
sudo mysql -e "INSERT INTO appdb.users (username, password_hash) VALUES ('repl_test', 'dummy');"
```

**즉시** `db2`(Standby)에서 조회:

```bash
sudo mysql -e "SELECT username FROM appdb.users;"
```

`repl_test`가 이미 반영되어 있습니다. 홈페이지에서 **회원가입을 하나 더** 진행하고 db2에서 조회해도 동일하게 나타납니다. Primary의 모든 변경이 Standby에 실시간 기록되는 것입니다.

### 4.2 Standby는 쓰기 거부 확인

`web1`에서 일반 계정(`app`)으로 db2에 쓰기를 시도하면:

```bash
mariadb -h db2 -u app -p'App123!' appdb -e "INSERT INTO visits VALUES (99, 0);"
```

`--read-only` 오류로 거부됩니다. 복제본의 **일관성(Consistency)** 이 이렇게 보호됩니다. (관리자 계정은 read_only의 예외이므로, 테스트는 일반 계정으로 해야 의미가 있습니다.)

---

## 5. 장애 대응 훈련 — Primary 장애와 Standby 승격

실무 시나리오를 그대로 연습합니다. **db1에 갑작스러운 장애가 발생했다고 가정합니다:**

```bash
ssh infra@192.168.100.21 "sudo poweroff"
```

홈페이지 새로고침 → 500 에러(웹 서버가 DB를 못 찾음). 이제 **db2를 새 Primary로 승격**합니다.

`db2`에서:

```bash
sudo mysql -e "STOP SLAVE; RESET SLAVE ALL;"          # 복제 수신 중단 (더 올 게 없다)
sudo sed -i 's/^read_only.*/read_only = 0/' /etc/mysql/mariadb.conf.d/50-server.cnf
sudo systemctl restart mariadb                         # 쓰기 허용으로 전환
```

웹 서버들이 새 Primary를 바라보게 합니다. 우리는 이름으로 접속하므로 **hosts 파일의 db1을 db2 주소로** 바꿔주면 끝입니다. `web1`(과 web2 켜져 있으면 양쪽)에서:

```bash
sudo sed -i 's/^192.168.100.21  db1/192.168.100.22  db1/' /etc/hosts
sudo systemctl restart webapp
```

홈페이지 새로고침 → **복구!** 로그인·회원가입 모두 됩니다. 이것이 "이름으로 부른다" 약속의 보답입니다 — 코드는 한 줄도 안 고쳤습니다.

> 실습을 마쳤으면 **원상 복구**하세요: web들의 hosts를 원래대로(`192.168.100.21  db1`) 되돌리고, db1을 켠 뒤, db2는 3장 절차로 복제를 다시 연결합니다(재복원 포함). 이 원상 복구 과정 자체가 최고의 복습입니다. 실무에서는 이 전환을 자동화(MHA, Orchestrator 등)하지만, **원리는 지금 손으로 한 것과 동일**합니다.
{: .prompt-tip }

---

## 6. 트러블슈팅

| 증상 | 진단 | 해결 |
|---|---|---|
| `Slave_IO_Running: Connecting` 이 계속됨 | `db2`에서 `mariadb -h db1 -u repl -p'Repl123!' -e "SELECT 1;"` | 접속 자체가 안 되면: db1 방화벽에 `.22` 허용 누락(2장), repl 계정 오타 |
| `Slave_IO_Running: No` + 에러 1236 | `SHOW SLAVE STATUS\G`의 `Last_IO_Error` | 좌표(MASTER_LOG_FILE/POS) 오타 — 3.1 좌표로 `CHANGE MASTER` 다시 |
| 에러: "misconfigured... equal MariaDB server ids" | 양쪽에서 `sudo mysql -e "SELECT @@server_id;"` | **server-id 중복** — VM을 복제로 만들었을 때 자주 발생. db2를 2로 수정 후 재시작 |
| `Slave_SQL_Running: No` | `Last_SQL_Error` 확인 | 복원 전에 Standby에 쓰기를 한 경우가 대부분 — 3장을 처음부터(복원→좌표→START SLAVE) 다시 |
| `Seconds_Behind_Master`가 계속 큼 | — | Standby VM 메모리/CPU 부족. 실습에선 잠시 기다리면 따라잡음 |
| 승격 후에도 웹 500 에러 | `web1`에서 `mariadb -h db1 -u app -p'App123!' appdb -e "SELECT 1;"` | hosts 수정 누락, db2 방화벽에 web 허용 누락(2장에서 미리 했는지) |

---

## 7. 핵심 용어 정리

| 용어 | 영문 | 정의 |
|---|---|---|
| 복제 | Replication | Primary의 변경 사항을 Standby가 실시간으로 수신·반영하는 기술 |
| 바이너리 로그 | Binary Log (binlog) | 모든 데이터 변경을 순서대로 기록한 파일 — 복제의 원천 |
| Primary / Standby | — | 쓰기를 담당하는 원본 DB / 대기 상태의 실시간 사본 DB |
| 승격 | Promotion | 장애 시 Standby를 새로운 Primary로 전환하는 절차 |
| RPO | Recovery Point Objective | 허용 가능한 데이터 손실 시점의 목표치 |
| RTO | Recovery Time Objective | 허용 가능한 서비스 복구 시간의 목표치 |
| 복제 지연 | Replication Lag | Standby가 Primary보다 뒤처진 정도 (`Seconds_Behind_Master`) |
| 논리 백업 | Logical Backup | 데이터를 SQL 문 형태로 추출하는 백업 (`mysqldump`) |

## 8. 마무리 — 스냅샷

전체 종료 후 스냅샷: `db1` → **`stage7-primary`**, `db2` → **`stage7-standby`**

여기까지로 **모든 층의 이중화**가 끝났습니다:

| 층 | 구성 | 장애 시 |
|---|---|---|
| 프록시 | proxy1 + proxy2 (VIP) | 자동 승계 (초 단위) |
| 웹 | web1 + web2 (LB) | 자동 제외 (사용자 무감각) |
| 세션 | redis1 | 재로그인으로 복구 |
| DB | db1 + db2 (복제) | 수동 승격 (데이터 무손실) |

그러나 아직 모든 서버가 **한 네트워크에 평평하게** 놓여 있습니다. 다음 편이 이 과정의 하이라이트 — **방화벽 VM을 세우고 DMZ와 내부망을 물리적으로 분리**합니다.
