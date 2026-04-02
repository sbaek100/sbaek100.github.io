---
title: (7주차) 보안시스템구축실습 7-2 - 네트워크 스캐닝 과제 & 방어 정리
date: 2026-03-26 09:30:00 +0900
categories:
  - 강의
  - 보안시스템구축실습
tags:
  - Nmap
  - Wireshark
  - 네트워크스캐닝
  - 포트관리
  - 방어
  - 셀프체크
mermaid: true
pin: false
description: 7주차 Nmap 스캔 결과 분석 과제와 불필요한 포트 차단 방법, 셀프체크 문제 정리.
---

## 시작 전 확인 (7-1 연결 체크)

7-2를 시작하기 전에 7-1 상태를 확인합니다.

```bash
# Ubuntu 서버에서 실행
# Apache가 실행 중인지 확인
sudo systemctl status apache2
# "active (running)" 이 보여야 합니다

# MySQL이 실행 중인지 확인
sudo systemctl status mysql
# "active (running)" 이 보여야 합니다

# MySQL bind-address 확인 — 반드시 127.0.0.1 이어야 합니다!
sudo grep "bind-address" /etc/mysql/mysql.conf.d/mysqld.cnf
# "bind-address = 127.0.0.1" 이 보여야 정상입니다
# 만약 0.0.0.0 이라면 지금 당장 수정하세요 (7-1 복원 단계 참고)
```

```bash
# Kali에서 현재 상태 스캔으로 확인
nmap -sV -p 22,2222,80,3306 192.168.0.30
# 정상 상태:
# 2222/tcp open  ssh  OpenSSH ...
# 80/tcp   open  http Apache ...
# 3306/tcp closed mysql       ← 이게 보여야 정상!
```

---

## Part 1: 과제 — 스캔 결과 분석 보고서

### 1-1. 스캔 보고서 작성

Nmap 결과를 파일로 저장하고 분석하는 방법입니다.

```bash
# Kali에서 실행
# 스캔 결과를 텍스트 파일로 저장
nmap -sV -O -p- 192.168.0.30 -oN /tmp/scan_report.txt
# -sV: 서비스 버전 탐지
# -O: 운영체제 탐지 (sudo 필요)
# -p-: 전체 포트(1~65535) 스캔 (시간이 걸림)
# -oN: output Normal format, 결과를 일반 텍스트 파일로 저장
# /tmp/scan_report.txt: 저장할 파일 경로

# 저장된 결과 확인
cat /tmp/scan_report.txt
# cat: 파일 내용을 화면에 출력하는 명령어
```

```bash
# 스캔 결과를 XML 형식으로도 저장 (도구 연동에 유용)
nmap -sV -p 22,2222,80,443,3306 192.168.0.30 -oX /tmp/scan_report.xml
# -oX: output XML format, XML 형식으로 저장
# XML은 다른 보안 도구와 연동할 때 편리
```

### 1-2. 스캔 결과 포트 위험도 분류표 작성

아래 표를 채워서 제출하세요.

| 포트 번호 | 서비스 | 상태 | 버전 | 위험도 (상/중/하) | 조치 방안 |
|-----------|--------|------|------|-------------------|-----------|
| 2222 | SSH | open | OpenSSH x.x | 하 | 키 인증 유지 |
| 80 | HTTP | open | Apache x.x | 중 | 불필요한 모듈 비활성화 |
| 3306 | MySQL | closed | - | 하 | 현재 상태 유지 |
| (추가 발견) | | | | | |

### 1-3. Wireshark 캡처 분석 과제

```bash
# Ubuntu 서버에서 Nmap 스캔 패킷 캡처 (인터페이스 이름 확인 필수!)
# ip link show 로 본인 인터페이스 이름 확인 후 ens33 자리에 입력
sudo tcpdump -i ens33 -w /tmp/nmap_capture.pcap -n
# -i: 캡처할 인터페이스 이름 (본인 환경에 맞게 수정)
# -w: 캡처 내용을 파일로 저장
# -n: IP/포트를 숫자로 표시 (빠름)
# Ctrl+C 로 중지
```

```bash
# Kali에서 Wireshark로 캡처 파일 분석
wireshark /tmp/nmap_capture.pcap &
# &: 백그라운드로 실행 (터미널 계속 사용 가능)
```

Wireshark에서 확인할 내용:
- `tcp.flags.syn == 1 && tcp.flags.ack == 0` 필터 → SYN 패킷만 표시
- `tcp.flags.rst == 1` 필터 → 닫힌 포트 응답(RST) 확인
- 패킷 수로 몇 개 포트가 스캔됐는지 확인

---

## Part 2: 방어 — 불필요한 포트 닫기

### 2-1. 현재 열린 포트 전체 확인

```bash
# Ubuntu 서버에서 실행
# ss: 소켓 상태 확인 명령어
# -tlnp: TCP(-t), Listen(-l), 숫자(-n), 프로세스(-p) 옵션 조합
sudo ss -tlnp
# 출력 예시:
# State  Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
# LISTEN 0       128     0.0.0.0:80          0.0.0.0:*      users:apache2
# LISTEN 0       4096    127.0.0.1:3306      0.0.0.0:*      users:mysqld
# LISTEN 0       128     0.0.0.0:2222        0.0.0.0:*      users:sshd
```

`Local Address:Port` 열 해석:
- `0.0.0.0:80` → 모든 외부 주소에서 접속 가능 (공개)
- `127.0.0.1:3306` → 로컬에서만 접속 가능 (내부 전용)

### 2-2. MySQL bind-address 복원 확인 및 수정

> **이 단계가 8주차로 이어지는 중요한 설정입니다!**

```bash
# Ubuntu에서 현재 MySQL bind-address 설정 확인
sudo grep "bind-address" /etc/mysql/mysql.conf.d/mysqld.cnf
```

만약 `bind-address = 0.0.0.0` 이라면 즉시 수정하세요:

```bash
# MySQL 설정 파일 열기
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
# bind-address = 0.0.0.0  →  bind-address = 127.0.0.1  로 수정
# nano 편집기 사용법:
#   Ctrl+W: 검색 (bind-address 입력 후 Enter)
#   화살표 키로 이동하여 수정
#   Ctrl+O: 저장
#   Ctrl+X: 나오기
```

```bash
# 설정 적용을 위해 MySQL 재시작
# restart: 서비스를 완전히 멈추고 다시 시작
sudo systemctl restart mysql

# 복원이 제대로 됐는지 확인
sudo ss -tlnp | grep mysql
# "127.0.0.1:3306" 이 보여야 정상
# "0.0.0.0:3306" 이 보이면 다시 수정 필요
```

```bash
# Kali에서 최종 확인 스캔
nmap -p 3306 192.168.0.30
# "3306/tcp closed mysql" 이 보여야 복원 성공!
```

### 2-3. 불필요한 서비스 중지 방법 (참고)

```bash
# 특정 서비스를 중지하고 자동 시작도 해제하는 방법
# (실습에서는 실행하지 않아도 됩니다 — 참고용)
sudo systemctl stop 서비스이름
# stop: 서비스 즉시 중지

sudo systemctl disable 서비스이름
# disable: 부팅 시 자동 시작 해제 (지금 당장 중지는 안 함)
```

---

## Part 3: 스캔 탐지하기

### 3-1. auth.log에서 스캔 흔적 확인

Nmap이 포트를 스캔할 때 SSH 포트에 연결 시도를 하면 로그가 남습니다.

```bash
# Ubuntu에서 최근 auth.log 확인
sudo tail -n 100 /var/log/auth.log | grep "192.168.0.10"
# tail -n 100: 마지막 100줄 출력
# grep "192.168.0.10": Kali IP에서 온 기록만 필터
```

```bash
# 스캔 관련 메시지 확인
sudo grep "Did not receive identification" /var/log/auth.log
# Nmap이 SSH 포트(2222)에 연결했다가 즉시 끊으면 이 메시지가 남음
```

### 3-2. syslog에서 연결 기록 확인

```bash
# syslog: 시스템 일반 로그 파일
# 네트워크 연결 관련 기록도 여기서 확인 가능
sudo grep "192.168.0.10" /var/log/syslog | tail -20
# tail -20: 마지막 20줄만 출력
```

---

## Part 4: 셀프체크

### 객관식 문제 (각 1점)

**Q1.** Nmap에서 서비스 버전 정보를 탐지하는 옵션은?

① `-sS`
② `-sV`
③ `-O`
④ `-p-`

---

**Q2.** MySQL이 `127.0.0.1:3306`에서 대기 중일 때 외부에서 접속 시도 결과는?

① open
② closed 또는 filtered
③ open|filtered
④ unknown

---

**Q3.** Nmap에서 전체 포트(1~65535)를 스캔하는 옵션은?

① `-p 65535`
② `-p all`
③ `-p-`
④ `--full-scan`

---

**Q4.** tcpdump에서 캡처한 패킷을 파일로 저장하는 옵션은?

① `-r`
② `-w`
③ `-f`
④ `-o`

---

### 단답형 문제 (각 2점)

**Q5.** `ss -tlnp` 명령에서 각 옵션(-t, -l, -n, -p)이 의미하는 것을 설명하시오.

**Q6.** MySQL의 `bind-address = 0.0.0.0` 설정이 위험한 이유를 설명하시오.

**Q7.** Nmap의 `-oN` 옵션은 어떤 역할을 하는가?

---

### 정답

| 번호 | 정답 | 해설 |
|------|------|------|
| Q1 | ② | `-sV` (Service Version) 서비스 버전 탐지 |
| Q2 | ② | 127.0.0.1은 로컬에서만 접속 가능, 외부에서 closed 또는 filtered |
| Q3 | ③ | `-p-` 는 "모든 포트"를 의미하는 단축 표기 |
| Q4 | ② | `-w` (write) 캡처 내용을 파일로 저장 |
| Q5 | -t: TCP만, -l: 대기 중인 소켓, -n: 숫자로 표시, -p: 프로세스 이름 |
| Q6 | 0.0.0.0은 모든 네트워크 주소에서 접속을 허용하므로 외부에서 데이터베이스에 직접 접근 가능 → 데이터 유출 위험 |
| Q7 | Normal format으로 스캔 결과를 텍스트 파일로 저장 |

---

## 8주차 준비 사항

8주차에서는 방화벽(iptables, ufw)을 설정합니다.
아래 상태를 유지한 채로 8주차를 시작해야 합니다.

### 상태 확인 체크리스트

```bash
# Ubuntu 서버에서 실행
# 1. MySQL bind-address 확인 — 반드시 127.0.0.1!
sudo grep "bind-address" /etc/mysql/mysql.conf.d/mysqld.cnf
# 예상 출력: bind-address = 127.0.0.1

# 2. Apache 실행 중 확인
sudo systemctl is-active apache2
# 예상 출력: active

# 3. MySQL 실행 중 확인
sudo systemctl is-active mysql
# 예상 출력: active

# 4. SSH 실행 중 확인
sudo systemctl is-active ssh
# 예상 출력: active
```

```bash
# Kali에서 현재 상태 최종 스캔 확인
nmap -sV -p 22,2222,80,443,3306 192.168.0.30
# 8주차 시작 전 정상 상태:
# 2222/tcp open  ssh    OpenSSH ...   ← SSH 열림
# 80/tcp   open  http   Apache ...   ← 웹 서버 열림
# 3306/tcp closed mysql              ← MySQL 외부 차단됨 (중요!)
```

> **8주차 예고: iptables와 ufw로 방화벽 설정**
>
> 7주차에서 Nmap으로 공격자가 서버를 어떻게 탐색하는지 배웠습니다.
> 8주차에서는 이 스캔을 막는 방화벽을 직접 설정합니다.
> - iptables: 리눅스 기본 방화벽 도구 (세밀한 제어)
> - ufw: 더 쉬운 방화벽 관리 도구
> - Kali에서 방화벽 효과를 Nmap으로 직접 검증
