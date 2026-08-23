---
title: "[웹 인프라 구축] 02. 1단계 — 서버 한 대에 전부 넣기 (올인원)"
date: 2026-08-10 09:04:00 +0900
categories:
  - 1.응용강의
  - 웹인프라구축
tags:
  - Flask
  - MariaDB
  - UFW
  - systemd
  - SPOF
math: false
mermaid: true
---

> 이번 글은 **직접 따라 하는 실습**입니다. `web1` 한 대에 웹 애플리케이션(Flask)과 데이터베이스(MariaDB)를 전부 설치해 **동작하는 미니 홈페이지**를 만듭니다. 그리고 이 구조의 문제점을 직접 확인합니다.
{: .prompt-info }

**이번에 켤 VM**: `web1`

## 0. 이번 단계의 그림

```mermaid
flowchart LR
    U["사용자<br/>(호스트 브라우저)"] -- ":8080" --> W["web1<br/>Flask + MariaDB<br/>(전부 한 대)"]
```

모든 서비스의 출발점은 이러한 단일 서버 구성입니다. 단순해 보이지만, 이 안에 "웹 서버란 무엇인가"와 "DB란 무엇인가"의 본질이 모두 들어 있습니다.

| 오늘 배우는 것 | 설명 |
|---|---|
| 웹 애플리케이션 | 요청을 받아 HTML을 만들어 주는 프로그램 (Flask) |
| RDBMS | 데이터를 표(테이블)로 영구 저장 (MariaDB — AWS에서 빌려 쓰면 "RDS") |
| systemd 서비스 | 프로그램을 "서버가 켜지면 자동 실행"으로 등록 |
| 방화벽 규칙 추가 | 새 역할이 생기면 필요한 포트만 연다 |

---

## 1. 소프트웨어 설치

호스트 PC의 PowerShell에서 `web1`에 접속합니다.

```bash
ssh infra@192.168.100.11
```

필요한 것을 전부 우분투 공식 저장소에서 설치합니다. (전부 무료·오픈소스)

```bash
sudo apt update
sudo apt install -y mariadb-server python3-flask python3-pymysql
```

| 패키지 | 정체 |
|---|---|
| `mariadb-server` | 관계형 데이터베이스 서버 |
| `python3-flask` | 파이썬 웹 프레임워크 |
| `python3-pymysql` | 파이썬에서 MariaDB에 접속하는 라이브러리 |

---

## 2. 데이터베이스 준비

### 2.1 기본 보안 설정

MariaDB는 설치 직후 보안 마법사를 한 번 돌려 줍니다.

```bash
sudo mysql_secure_installation
```

질문이 여러 개 나옵니다. 아래처럼 답하세요.

| 질문 | 답 |
|---|---|
| Enter current password for root | 그냥 **Enter** |
| Switch to unix_socket authentication? | `n` |
| Change the root password? | `n` (root는 sudo로만 접속하는 방식 유지) |
| Remove anonymous users? | `Y` |
| Disallow root login remotely? | `Y` ← 중요 |
| Remove test database? | `Y` |
| Reload privilege tables? | `Y` |

### 2.2 홈페이지용 DB와 계정 만들기

```bash
sudo mysql
```

MariaDB 프롬프트(`MariaDB [(none)]>`)가 뜨면 아래를 붙여넣습니다.

```sql
CREATE DATABASE appdb;
CREATE USER 'app'@'localhost' IDENTIFIED BY 'App123!';
GRANT ALL PRIVILEGES ON appdb.* TO 'app'@'localhost';

USE appdb;
CREATE TABLE visits (
  id INT PRIMARY KEY,
  count INT NOT NULL
);
INSERT INTO visits VALUES (1, 0);

FLUSH PRIVILEGES;
EXIT;
```

- `appdb` : 우리 홈페이지 전용 데이터베이스
- `app` 계정: 웹 프로그램이 DB에 접속할 때 쓰는 **전용 계정** (root를 쓰지 않는 것이 기본 보안 수칙)
- `visits` 표: 방문자 수를 저장

---

## 3. 웹 애플리케이션 만들기

### 3.1 코드 작성

```bash
mkdir -p ~/app
nano ~/app/app.py
```

아래 코드를 통째로 붙여넣고 저장합니다.

```python
from flask import Flask
import pymysql, socket

app = Flask(__name__)

DB = dict(host="localhost", user="app", password="App123!", database="appdb")

def db_conn():
    return pymysql.connect(**DB)

@app.route("/")
def home():
    conn = db_conn()
    with conn.cursor() as cur:
        cur.execute("UPDATE visits SET count = count + 1 WHERE id = 1")
        conn.commit()
        cur.execute("SELECT count FROM visits WHERE id = 1")
        count = cur.fetchone()[0]
    conn.close()
    return (f"<h1>미니 홈페이지</h1>"
            f"<p>응답한 서버: <b>{socket.gethostname()}</b></p>"
            f"<p>당신은 <b>{count}</b>번째 방문자입니다.</p>")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

코드가 하는 일은 세 줄로 요약됩니다: ① 누가 접속하면 → ② DB의 방문자 수를 1 올리고 → ③ 서버 이름과 함께 HTML로 보여준다. **"응답한 서버 이름"** 은 나중에 서버가 여러 대가 됐을 때 누가 응답했는지 확인하는 핵심 장치입니다.

### 3.2 우선 수동으로 실행해 보기

```bash
python3 ~/app/app.py
```

`Running on http://0.0.0.0:8080` 이 나오면 실행 중입니다. **그런데 호스트 브라우저에서 `http://192.168.100.11:8080` 을 열면 — 접속이 안 됩니다!**

왜일까요? 우리가 1편에서 **방화벽을 "기본 차단"으로 켜 두었기 때문**입니다. 프로그램이 떠 있어도 방화벽이 막으면 못 들어옵니다. 이것이 정상 동작입니다.

`Ctrl+C` 로 일단 종료합니다.

### 3.3 방화벽에 8080 허용 추가

`web1`의 새 역할(웹 서버)에 맞는 문을 엽니다.

```bash
sudo ufw allow 8080/tcp
sudo ufw status
```

> 지금은 "모든 곳에서 8080 허용"이지만, 4편에서 리버스 프록시가 생기면 **"프록시에서 오는 8080만 허용"** 으로 좁힙니다. 인프라가 성장할수록 허용 범위는 좁아집니다.
{: .prompt-tip }

> **★ 꼭 알아 두어야 할 개념 — 포트(Port) 번호**: 포트는 "한 서버 안의 여러 프로그램을 구분하는 문 번호"입니다. 주요 포트는 국제적으로 약속되어 있어(웰노운 포트, Well-known Port) 반드시 암기해 둘 가치가 있습니다: **22**(SSH), **80**(HTTP), **443**(HTTPS), **3306**(MySQL/MariaDB), **6379**(Redis). 8080은 "HTTP의 보조·내부용"으로 관례적으로 쓰이는 번호입니다. 방화벽 규칙, 오류 진단, 보안 점검이 모두 이 번호들을 중심으로 이루어집니다.
{: .prompt-tip }

### 3.4 서비스로 등록 (자동 실행)

손으로 실행하면 SSH 창을 닫는 순간 죽습니다. **systemd 서비스**로 등록해 "재부팅해도 자동으로 뜨는" 진짜 서버 프로그램으로 만듭니다.

```bash
sudo tee /etc/systemd/system/webapp.service > /dev/null <<'EOF'
[Unit]
Description=Mini web application
After=network.target mariadb.service

[Service]
User=infra
WorkingDirectory=/home/infra/app
ExecStart=/usr/bin/python3 /home/infra/app/app.py
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now webapp
systemctl status webapp --no-pager
```

`Active: active (running)` 이 보이면 성공입니다.

---

## 4. 동작 확인

호스트 PC 브라우저에서:

```
http://192.168.100.11:8080
```

**"미니 홈페이지 / 응답한 서버: web1 / 당신은 N번째 방문자입니다"** 가 보이고, 새로고침할 때마다 숫자가 올라가면 완성입니다. 축하합니다 — 여러분은 방금 **웹 서버 + DB + 방화벽 + 자동 실행**을 갖춘 서비스를 만들었습니다.

재부팅 생존 테스트도 해 봅니다.

```bash
sudo reboot
```

1분 뒤 브라우저 새로고침 → 다시 표시되면 완전한 성공입니다. (`Restart=always`와 `enable` 덕분에 서비스가 부팅 시 자동 기동됩니다.)

---

## 5. 그런데 — 이 구조의 문제점 (직접 확인)

### 실험 1: 단일 장애점(SPOF)의 확인

```bash
sudo systemctl stop webapp
```

브라우저 새로고침 → **접속 불가.** 서비스 전체가 이 한 대에 의존하고 있습니다. 이처럼 "그 하나가 정지하면 전체가 중단되는 지점"을 **단일 장애점(SPOF, Single Point of Failure)** 이라 합니다.

```bash
sudo systemctl start webapp   # 복구
```

### 실험 2: 동일 서버 안의 자원 경합

```bash
ps aux | grep -E "mariadbd|app.py" | grep -v grep
```

웹 프로그램과 DB가 **같은 컴퓨터의 CPU·메모리를 나눠 쓰고** 있습니다. 접속자가 몰려 웹이 바빠지면 DB도 함께 느려지고, 그 반대도 마찬가지입니다.

### 문제 정리

| # | 문제 | 해결 편 |
|---|---|---|
| 1 | 웹과 DB가 자원을 뺏는다 | **03 (DB 분리)** |
| 2 | 암호화 없는 http, 포트 8080 노출 | 04 (리버스 프록시 + HTTPS) |
| 3 | 서버 1대 = 단일 장애점 | 05~08 (증설·이중화) |
| 4 | 인터넷에서 서버에 직접 닿는다 | 09 (방화벽 + DMZ) |

---

## 6. 트러블슈팅

| 증상 | 진단 명령 | 원인/해결 |
|---|---|---|
| 브라우저 접속 불가 | `sudo ufw status` | 8080 허용 누락 → 3.3 다시 |
| 브라우저 접속 불가 (방화벽 정상) | `systemctl status webapp` | 서비스 중단 → 아래 로그 확인 |
| 서비스가 계속 재시작됨 | `journalctl -u webapp -n 30` | 코드 오타(붙여넣기 누락) 또는 DB 접속 실패 |
| `Access denied for user 'app'` | `sudo mysql` 후 `SELECT user,host FROM mysql.user;` | 2.2의 계정 생성 SQL 다시 실행, 비밀번호 `App123!` 확인 |
| `Unknown database 'appdb'` | 〃 | `CREATE DATABASE appdb;` 누락 |
| 페이지는 뜨는데 500 에러 | `journalctl -u webapp -n 30` | visits 테이블 누락 → 2.2의 `CREATE TABLE` 다시 |
| `ping web1` 은 되는데 이름 접속 안 됨 | `cat /etc/hosts` | 1편 6.3의 hosts 블록 누락 |

> **트러블슈팅의 황금 순서**: ① 방화벽(`ufw status`) → ② 프로세스(`systemctl status`) → ③ 로그(`journalctl`) → ④ 설정 파일. 앞으로 모든 편에서 이 순서로 진단합니다.
{: .prompt-warning }

---

## 7. 핵심 용어 정리

| 용어 | 영문 | 정의 |
|---|---|---|
| 웹 애플리케이션 | Web Application | 요청을 받아 동적으로 응답(HTML 등)을 생성하는 프로그램 |
| 관계형 DBMS | RDBMS | 데이터를 표(테이블) 형태로 저장·관리하는 시스템 (MariaDB, MySQL 등) |
| systemd 서비스 | systemd Service | 리눅스에서 프로그램을 자동 기동·재시작·감시 대상으로 등록하는 표준 방식 |
| 포트 | Port | 한 서버 안에서 프로그램(서비스)을 구분하는 번호 (0~65535) |
| 단일 장애점 | SPOF | 그 하나의 정지가 전체 서비스 중단으로 이어지는 구성 요소 |
| 로그 | Log | 프로그램의 동작·오류 기록. `journalctl`이 systemd 서비스의 로그 열람 도구 |

## 8. 마무리 — 스냅샷

```bash
sudo poweroff
```

VirtualBox에서 `web1` 스냅샷: **`stage1-allinone`**

다음 편에서는 DB를 **별도 서버 `db1`로 분리**하고, 홈페이지에 **회원가입/로그인 기능**을 추가합니다. "서버 두 대가 네트워크로 협력"하는 첫 경험입니다.
