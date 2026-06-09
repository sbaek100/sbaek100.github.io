---
title: 2026-1학기 중간고사준비 - setup-script.sh 실습 환경 20문제
date: 2026-04-15 09:00:00 +0900
categories:
  - 강의
  - 리눅스(Ubuntu)실습
tags:
  - 리눅스
  - Ubuntu
  - 중간고사
  - wget
  - grep
  - awk
  - sed
  - find
  - chmod
  - bash
  - 파이프
mermaid: false
pin: true
description: 2~6주차 핵심 명령어를 복습하는 중간고사 대비 실습 20문제. GitHub에서 setup-script.sh를 내려받아 ~/security 환경을 만들고 문제를 푼다.
---

# 2026-1학기 중간고사준비 — setup-script.sh 실습 환경 20문제

GitHub에 올려둔 보안 실습용 셸 스크립트(`setup-script.sh`)를 내려받아 `~/security` 환경을 만든 뒤, 그 안의 로그·설정·시스템 파일을 대상으로 20문제를 푼다.


> 주의
>
> 이 스크립트는 실서비스 구축용이 아니라 취약점 분석과 로그 분석 연습용이다.
> 개인 Ubuntu VM 또는 실습 서버에서만 실행하고, 운영 서버에서는 실행하지 않는다.
{: .prompt-danger }

---

## 사전 실습 1. `wget` 설치

Ubuntu에 `wget`이 없으면 아래 명령으로 설치한다.

```bash
sudo apt update
sudo apt install -y wget
```

예상 화면:

```bash
Reading package lists... Done
...
wget is already the newest version (1.21-...).
```

> `wget`이 이미 설치되어 있으면 위와 같이 newest version 메시지만 나온다.
{: .prompt-tip }

---

## 사전 실습 2. 스크립트 내려받기

GitHub에서 `setup-script.sh`를 내려받는다.

- GitHub 페이지: [setup-script.sh](https://github.com/sbaek100/security-training/blob/main/setup-script.sh)
- Raw 파일: [raw setup-script.sh](https://raw.githubusercontent.com/sbaek100/security-training/main/setup-script.sh)

```bash
cd ~
wget https://raw.githubusercontent.com/sbaek100/security-training/main/setup-script.sh
ls -l setup-script.sh
```

예상 화면:

```bash
Saving to: 'setup-script.sh'
-rw-r--r-- 1 ubuntu ubuntu ... setup-script.sh
```

---

## 사전 실습 3. 스크립트 확인 후 실행

다운로드한 파일은 바로 실행하지 말고, 먼저 내용을 확인한 뒤 실행한다.

```bash
cat setup-script.sh
chmod +x setup-script.sh
./setup-script.sh
```

예상 화면:

```bash
Creating directories...
Creating access.log...
Creating error.log...
Creating auth.log...
Creating sshd_config...
Creating apache2.conf...
...
Setup complete!
```

---

## 사전 실습 4. 생성된 환경 확인

스크립트 실행이 끝나면 `~/security` 폴더가 생성된다.
이제부터의 20문제는 이 환경이 준비된 상태에서 푼다.

```bash
sudo apt install -y tree
tree ~/security
```

예상 화면:

```bash
security
├── config
│   ├── apache2.conf
│   ├── firewall_rules.txt
│   └── sshd_config
├── logs
│   ├── access.log
│   ├── auth.log
│   └── error.log
├── network_capture_summary.txt
├── reports
├── samples
│   ├── database_dump.sql
│   ├── .htaccess
│   ├── suspicious_script.sh
│   └── vulnerability_scan.txt
├── scripts
└── system
    ├── firewall_status.txt
    ├── installed_packages.txt
    ├── network_config.txt
    ├── passwd
    └── suid_files.txt
```

---

## 문제 1

실습을 시작하기 전에 현재 작업 디렉터리가 어디인지 확인하려고 한다. 현재 경로를 출력하는 명령은 무엇인가?

### 실습

현재 경로를 확인하라.

### 예상 화면

```bash
/home/ubuntu
```

### 답

```bash
pwd
```

---

## 문제 2

`~/security` 디렉터리로 이동한 뒤, 안에 어떤 파일과 폴더가 있는지 확인하라.

### 실습

`~/security`로 이동하고 내용을 확인하라.

### 예상 화면

```bash
config  logs  network_capture_summary.txt  reports  samples  scripts  system
```

### 답

```bash
cd ~/security
ls
```

---

## 문제 3

`~/security/samples` 디렉터리에 숨긴 파일(hidden file)이 있는지 확인하라. 숨긴 파일의 이름은 무엇인가?

### 실습

`samples` 디렉터리의 모든 파일(숨긴 파일 포함)을 확인하라.

### 예상 화면

```bash
.  ..  .htaccess  database_dump.sql  suspicious_script.sh  vulnerability_scan.txt
```

### 답

```bash
ls -la ~/security/samples
```

> `.htaccess`가 숨긴 파일이다. 파일 이름이 `.`(점)으로 시작하면 `ls`만으로는 보이지 않고, `-a` 옵션을 써야 표시된다.
{: .prompt-tip }

---

## 문제 4

SSH 설정 파일의 내용을 확인하려고 한다. `~/security/config/sshd_config` 파일의 전체 내용을 출력하라.

### 실습

`sshd_config` 파일 내용을 출력하라.

### 예상 화면

```bash
# SSH Server Configuration
Port 22
PermitRootLogin yes
PasswordAuthentication yes
MaxAuthTries 6
...
```

### 답

```bash
cat ~/security/config/sshd_config
```

---

## 문제 5

`~/security/samples/suspicious_script.sh` 파일의 권한과 소유자를 확인하라. 이 파일에 실행 권한(`x`)이 있는가?

### 실습

파일의 상세 정보를 확인하라.

### 예상 화면

```bash
-rw-r--r-- 1 ubuntu ubuntu ... suspicious_script.sh
```

### 답

```bash
ls -l ~/security/samples/suspicious_script.sh
```

> 권한이 `-rw-r--r--`이므로 소유자·그룹·기타 모두 실행 권한(`x`)이 없다.
{: .prompt-tip }

---

## 문제 6

앞 문제에서 확인한 `suspicious_script.sh`에 소유자 실행 권한을 추가하라. 추가한 뒤 권한이 바뀌었는지 확인까지 하라.

### 실습

소유자에게 실행 권한을 부여하고 결과를 확인하라.

### 예상 화면

```bash
-rwxr--r-- 1 ubuntu ubuntu ... suspicious_script.sh
```

### 답

```bash
chmod u+x ~/security/samples/suspicious_script.sh
ls -l ~/security/samples/suspicious_script.sh
```

---

## 문제 7

인증 로그(`~/security/logs/auth.log`)에서 `Failed`가 포함된 줄을 모두 찾아라.

### 실습

`auth.log`에서 `Failed` 문자열이 포함된 줄을 출력하라.

### 예상 화면

```bash
Jun 15 03:22:01 webserver sshd[1234]: Failed password for root from 10.0.0.99 port 22 ssh2
Jun 15 03:22:05 webserver sshd[1234]: Failed password for root from 10.0.0.99 port 22 ssh2
Jun 15 03:22:10 webserver sshd[1234]: Failed password for admin from 10.0.0.99 port 22 ssh2
...
```

### 답

```bash
grep "Failed" ~/security/logs/auth.log
```

---

## 문제 8

`auth.log`에서 `Failed`가 포함된 줄이 총 몇 개인지 세어라.

### 실습

`Failed` 문자열이 포함된 줄의 수만 출력하라.

### 예상 화면

```bash
15
```

### 답

```bash
grep -c "Failed" ~/security/logs/auth.log
```

---

## 문제 9

접근 로그(`~/security/logs/access.log`)에서 HTTP 상태 코드 `404`가 포함된 줄이 몇 개인지 구하라.

### 실습

`access.log`에서 `404`가 포함된 줄의 수를 출력하라.

### 예상 화면

```bash
3
```

### 답

```bash
grep "404" ~/security/logs/access.log | wc -l
```

> `grep -c "404"` 도 같은 결과를 낸다. 파이프(`|`)와 `wc -l`을 사용하는 방법도 익혀 두자.
{: .prompt-tip }

---

## 문제 10

`access.log`에서 각 줄의 첫 번째 필드(IP 주소)만 추출하라.

### 실습

`access.log`의 IP 주소(첫 번째 필드)만 출력하라.

### 예상 화면

```bash
192.168.1.100
8.8.8.8
45.33.22.11
123.45.67.89
192.168.1.100
...
```

### 답

```bash
awk '{print $1}' ~/security/logs/access.log
```

---

## 문제 11

`access.log`에서 어떤 IP가 가장 많이 접속했는지 확인하라. IP별 접속 횟수를 내림차순으로 정렬해서 출력하라.

### 실습

IP별 접속 횟수를 구하고 많은 순서대로 출력하라.

### 예상 화면

```bash
      4 45.33.22.11
      4 8.8.8.8
      3 123.45.67.89
      ...
```

### 답

```bash
awk '{print $1}' ~/security/logs/access.log | sort | uniq -c | sort -rn
```

> 이 파이프라인은 보안 분석에서 매우 자주 쓰인다. `awk`로 필드 추출 → `sort`로 정렬 → `uniq -c`로 집계 → `sort -rn`으로 내림차순 정렬.
{: .prompt-tip }

---

## 문제 12

`auth.log`에서 `Failed password` 줄만 골라낸 뒤, 공격을 시도한 IP 주소만 추출하라.

### 실습

로그인 실패 줄에서 IP 주소만 출력하라.

### 예상 화면

```bash
10.0.0.99
10.0.0.99
10.0.0.99
...
```

### 답

```bash
grep "Failed password" ~/security/logs/auth.log | awk '{print $(NF-3)}'
```

> `NF`는 awk에서 "마지막 필드 번호"를 뜻한다. sshd 로그 형식에서 IP는 뒤에서 네 번째 필드에 있으므로 `$(NF-3)`으로 추출한다. 로그 형식에 따라 필드 위치가 달라질 수 있으니 `cat`으로 먼저 줄 구조를 확인하는 습관을 들이자.
{: .prompt-tip }

---

## 문제 13

`~/security/system/passwd` 파일에서 사용자 이름(첫 번째 필드)만 추출하라. 필드 구분자는 `:`이다.

### 실습

`passwd`에서 사용자 이름만 출력하라.

### 예상 화면

```bash
root
daemon
bin
sys
...
```

### 답

```bash
cut -d: -f1 ~/security/system/passwd
```

---

## 문제 14

`~/security` 디렉터리 아래에서 확장자가 `.log`인 파일을 모두 찾아라.

### 실습

`.log` 확장자를 가진 파일을 재귀적으로 검색하라.

### 예상 화면

```bash
/home/ubuntu/security/logs/access.log
/home/ubuntu/security/logs/error.log
/home/ubuntu/security/logs/auth.log
```

### 답

```bash
find ~/security -name "*.log"
```

---

## 문제 15

`~/security` 안의 각 하위 디렉터리가 차지하는 용량을 사람이 읽기 쉬운 단위(K, M 등)로 확인하라.

### 실습

각 하위 디렉터리의 디스크 사용량을 확인하라.

### 예상 화면

```bash
4.0K    /home/ubuntu/security/reports
4.0K    /home/ubuntu/security/scripts
12K     /home/ubuntu/security/logs
8.0K    /home/ubuntu/security/config
16K     /home/ubuntu/security/samples
8.0K    /home/ubuntu/security/system
56K     /home/ubuntu/security
```

### 답

```bash
du -h --max-depth=1 ~/security
```

---

## 문제 16

`~/security/logs` 디렉터리 전체를 `logs_backup.tar.gz`로 압축하라. 압축 파일은 `~/security/reports/`에 저장하라.

### 실습

logs 디렉터리를 tar.gz로 압축하라.

### 예상 화면

```bash
# 출력 없음. 아래 명령으로 확인한다.
ls -l ~/security/reports/logs_backup.tar.gz
-rw-r--r-- 1 ubuntu ubuntu ... logs_backup.tar.gz
```

### 답

```bash
tar -czf ~/security/reports/logs_backup.tar.gz -C ~/security logs
```

> `-c`는 생성(create), `-z`는 gzip 압축, `-f`는 출력 파일 이름 지정이다. `-C ~/security`는 해당 디렉터리로 이동한 뒤 `logs`를 압축하라는 뜻이다.
{: .prompt-tip }

---

## 문제 17

`access.log`에서 `404` 상태 코드가 포함된 줄을 `~/security/reports/404_report.txt`로 저장하라.

### 실습

검색 결과를 파일로 리다이렉션하라.

### 예상 화면

```bash
# 터미널 출력 없음. 아래 명령으로 확인한다.
cat ~/security/reports/404_report.txt
45.33.22.11 - - [15/Jun/2024:03:45:22 +0000] "GET /../../etc/passwd HTTP/1.1" 404 ...
...
```

### 답

```bash
grep "404" ~/security/logs/access.log > ~/security/reports/404_report.txt
```

---

## 문제 18

`~/security/scripts/hello.sh`를 만들어라. 변수 `NAME`에 자신의 이름을 저장하고, `"안녕하세요, [이름]입니다"` 형식으로 출력하는 스크립트를 작성한 뒤 실행하라.

### 실습

변수와 `echo`를 사용하는 Bash 스크립트를 작성하고 실행하라.

### 예상 화면

```bash
안녕하세요, 홍길동입니다
```

### 답

```bash
cat << 'EOF' > ~/security/scripts/hello.sh
#!/bin/bash
NAME="홍길동"
echo "안녕하세요, ${NAME}입니다"
EOF
chmod +x ~/security/scripts/hello.sh
~/security/scripts/hello.sh
```

> `NAME="홍길동"` 부분에 자신의 이름을 넣으면 된다. 변수에 값을 넣을 때 `=` 양쪽에 공백이 있으면 오류가 나니 주의하자.
{: .prompt-tip }

---

## 문제 19

`~/security/scripts/check_log.sh`를 만들어라. `~/security/logs/auth.log` 파일이 존재하면 `"auth.log 파일이 존재합니다"`를 출력하고, 없으면 `"auth.log 파일이 없습니다"`를 출력하는 스크립트를 작성하라.

### 실습

`if` 조건문과 `-f` 테스트를 사용하는 스크립트를 작성하고 실행하라.

### 예상 화면

```bash
auth.log 파일이 존재합니다
```

### 답

```bash
cat << 'EOF' > ~/security/scripts/check_log.sh
#!/bin/bash
if [ -f ~/security/logs/auth.log ]; then
    echo "auth.log 파일이 존재합니다"
else
    echo "auth.log 파일이 없습니다"
fi
EOF
chmod +x ~/security/scripts/check_log.sh
~/security/scripts/check_log.sh
```

> `-f`는 "일반 파일이 존재하는가"를 테스트한다. 디렉터리를 검사하려면 `-d`를 쓴다.
{: .prompt-tip }

---

## 문제 20

`~/security/scripts/count_errors.sh`를 만들어라. `~/security/logs/` 아래의 모든 `.log` 파일을 반복하면서, 각 파일에서 `error`(대소문자 무시)가 포함된 줄 수를 출력하는 스크립트를 작성하라.

### 실습

`for` 반복문, `grep -ic`, 변수를 사용하는 스크립트를 작성하고 실행하라.

### 예상 화면

```bash
access.log: 0건
auth.log: 0건
error.log: 5건
```

### 답

```bash
cat << 'EOF' > ~/security/scripts/count_errors.sh
#!/bin/bash
for FILE in ~/security/logs/*.log; do
    COUNT=$(grep -ic "error" "$FILE")
    echo "$(basename "$FILE"): ${COUNT}건"
done
EOF
chmod +x ~/security/scripts/count_errors.sh
~/security/scripts/count_errors.sh
```

> `grep -i`는 대소문자를 무시하고, `-c`는 일치하는 줄 수만 출력한다. `basename`은 경로에서 파일 이름만 꺼내는 명령이다.
{: .prompt-tip }

---

