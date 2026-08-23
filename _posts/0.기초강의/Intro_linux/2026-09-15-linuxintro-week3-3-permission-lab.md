---
title: 리눅스 기초 3주차-3. 접근 권한 시나리오 실습
date: 2026-09-15 13:00:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - 접근권한
  - 시나리오실습
  - 보안
  - chmod
  - SetUID
pin:
mermaid: false
---

> **학습 목표**
> 1. 요구사항에 맞추어 팀 협업 디렉터리의 권한을 설계·구현할 수 있다.
> 2. 취약하게 설정된 시스템의 권한 문제를 진단하고 바로잡을 수 있다.
> 3. 특수 권한(SetUID·Sticky Bit)의 위험 요소를 점검할 수 있다.
{: .prompt-info }

본 실습은 3주차-2에서 학습한 사용자 계정과 접근 권한을 실제 상황에 적용하는 시간이다. 세 가지 시나리오를 순서대로 수행한다. 각 시나리오는 독립적이며, 실습에 필요한 계정과 데이터를 **직접 생성**하고, 실습이 끝나면 **반드시 정리**한다.

> **실습 준비 안내**
> 모든 실습은 Ubuntu 데스크톱(`ubuntu-y1`)의 터미널에서 수행한다. 관리자 권한이 필요한 명령에는 `sudo`가 붙으며, 비밀번호를 물으면 로그인 비밀번호를 입력한다. 실습 계정을 만들고 삭제하므로, 시작 전 스냅샷을 찍어 두면 언제든 되돌릴 수 있다.
{: .prompt-tip }

---
---

# 시나리오 1. 개발팀 협업 디렉터리 구축

---

## 상황

학습자는 시스템 관리자이다. 개발팀 두 사람(`alice`, `bob`)이 함께 사용할 협업 디렉터리를 구축하여야 한다. 요구사항은 다음과 같다.

| 번호 | 요구사항 |
|---|---|
| 1 | 개발팀 전용 디렉터리는 개발팀 구성원만 접근할 수 있어야 한다. |
| 2 | 팀에 속하지 않은 사용자는 접근할 수 없어야 한다. |
| 3 | 같은 팀 구성원이 만든 파일은 자동으로 팀 그룹 소유가 되어, 서로 읽고 쓸 수 있어야 한다. |

## 단계 1. 실습 계정과 그룹 준비

```bash
sudo groupadd devteam
```

```bash
sudo useradd -m -s /bin/bash alice
sudo useradd -m -s /bin/bash bob
```

```bash
sudo usermod -aG devteam alice
sudo usermod -aG devteam bob
```

> `-aG`는 기존 그룹을 유지하며(**a**ppend) 새 그룹(**G**roup)에 추가한다. `-a`를 빠뜨리면 기존 그룹에서 제외되므로 주의한다.

```bash
id alice
```

> `alice`의 소속 그룹 목록에 `devteam`이 표시되어야 한다.

## 단계 2. 협업 디렉터리 생성과 그룹 지정

```bash
sudo mkdir -p /srv/teamwork
```

```bash
sudo chown root:devteam /srv/teamwork
```

> `/srv`는 서비스용 자료를 두는 표준 위치이다. 소유자는 root, 소유 그룹은 `devteam`으로 지정하였다.

## 단계 3. 권한 설정 — 요구사항 1·2의 구현

```bash
sudo chmod 2770 /srv/teamwork
```

```bash
ls -ld /srv/teamwork
```

> `2770`의 각 자리를 해석하면 다음과 같다.
>
> | 자리 | 값 | 의미 |
> |---|---|---|
> | 특수 | 2 | **SetGID** — 새 파일이 `devteam` 그룹을 상속(요구사항 3) |
> | 소유자 | 7 | root: 읽기·쓰기·실행 |
> | 그룹 | 7 | devteam: 읽기·쓰기·실행 |
> | 기타 | 0 | 그 외 사용자: 접근 불가(요구사항 2) |
>
> 출력에서 그룹 권한 자리에 `s`(예: `drwxrws---`)가 보이면 SetGID가 정상 설정된 것이다.

## 단계 4. 요구사항 3 검증 — 그룹 소유권 상속

```bash
sudo -u alice touch /srv/teamwork/alice_note.txt
```

```bash
ls -l /srv/teamwork/alice_note.txt
```

> `sudo -u alice`는 alice의 권한으로 명령을 실행한다. 생성된 파일의 **그룹이 `devteam`** 으로 표시되면, SetGID에 의한 그룹 상속이 동작한 것이다. 이제 같은 팀의 bob도 이 파일에 접근할 수 있다.

```bash
sudo -u bob cat /srv/teamwork/alice_note.txt
```

> bob이 alice의 파일을 읽을 수 있음을 확인한다. 두 사용자 모두 `devteam` 그룹에 속하기 때문이다.

## 단계 5. 요구사항 2 검증 — 비구성원 차단

```bash
sudo useradd -m -s /bin/bash outsider
```

```bash
sudo -u outsider ls /srv/teamwork
```

> `Permission denied`로 차단된다. `outsider`는 `devteam`에 속하지 않아 기타(others) 권한(`0`)이 적용되기 때문이다.

## 단계 6. 정리 — 반드시 수행

```bash
sudo rm -rf /srv/teamwork
sudo userdel -r alice
sudo userdel -r bob
sudo userdel -r outsider
sudo groupdel devteam
```

> **완료 기준**
> 1. 그룹 구성원(alice, bob)은 협업 디렉터리에 접근하고, 서로의 파일을 공유할 수 있다.
> 2. SetGID로 새 파일의 그룹이 자동 상속된다.
> 3. 비구성원(outsider)은 접근이 차단된다.
> 4. 실습 계정과 디렉터리를 모두 정리하였다.
{: .prompt-tip }

---
---

# 시나리오 2. 취약한 권한 진단과 시정

---

## 상황

학습자는 보안 담당자로서, 동료가 급하게 설정해 둔 웹 서비스 디렉터리를 점검하게 되었다. 여러 파일의 권한이 부적절하게 설정되어 있다. 취약한 부분을 **진단**하고 **바로잡는다.**

## 단계 1. 취약한 환경 구성(점검 대상 준비)

실습을 위하여, 의도적으로 취약하게 설정된 디렉터리를 만든다.

```bash
mkdir -p ~/lab03/audit && cd ~/lab03/audit
```

```bash
echo "DB_PASSWORD=secret123" > config.env
echo "#!/bin/bash" > deploy.sh
echo "echo deploying" >> deploy.sh
touch note.bak
```

```bash
chmod 777 config.env      # (취약) 설정 파일에 전권 부여
chmod 666 deploy.sh       # (취약) 실행 스크립트가 실행 불가·쓰기 개방
chmod 644 note.bak        # 백업 파일 노출
```

## 단계 2. 진단 — 현재 권한 조사

```bash
ls -l
```

> 다음 세 가지 문제를 확인한다.
>
> | 파일 | 현재 권한 | 문제점 |
> |---|---|---|
> | `config.env` | `-rwxrwxrwx`(777) | 비밀번호가 담긴 설정 파일을 누구나 읽고 수정할 수 있다. |
> | `deploy.sh` | `-rw-rw-rw-`(666) | 실행 권한이 없어 동작하지 않고, 누구나 내용을 바꿀 수 있다. |
> | `note.bak` | `-rw-r--r--`(644) | 백업 파일이 타인에게 노출되어 있다. |

## 단계 3. 시정 — 최소 권한으로 조정

```bash
chmod 600 config.env
```

> 민감한 설정 파일은 소유자만 읽고 쓰도록 `600`으로 제한한다.

```bash
chmod 700 deploy.sh
```

> 실행 스크립트는 소유자만 읽고·쓰고·실행하도록 `700`으로 설정한다.

```bash
chmod 600 note.bak
```

> 백업 파일도 소유자 전용으로 제한한다. 불필요한 백업 파일은 삭제하는 것이 더 바람직하다.

## 단계 4. 시정 결과 확인

```bash
ls -l
```

> `config.env`가 `-rw-------`, `deploy.sh`가 `-rwx------`, `note.bak`가 `-rw-------`로 바뀌었는지 확인한다. 이제 소유자 외에는 접근할 수 없다.

## 단계 5. 정리

```bash
cd ~ && rm -rf ~/lab03/audit
```

> **완료 기준**
> 1. 세 파일의 권한 문제를 각각 진단하였다.
> 2. `config.env` 600, `deploy.sh` 700, `note.bak` 600으로 시정하였다.
> 3. "최소 권한 원칙"에 따라, 각 파일에 필요한 최소한의 권한만 남겼다.
{: .prompt-tip }

---
---

# 시나리오 3. 특수 권한 점검

---

## 상황

시스템에 불필요한 SetUID 파일이 있으면 권한 상승 공격의 통로가 된다. 학습자는 시스템의 SetUID 파일 목록을 점검하고, 위험한 예를 재현하여 그 원리를 이해한다.

## 단계 1. 정상 SetUID 파일 확인

```bash
find /usr/bin -perm -4000 -type f
```

> `-perm -4000`은 SetUID가 설정된 파일을 찾는 조건이다. `passwd`, `sudo` 등 **정상적으로** 관리자 권한이 필요한 명령이 출력된다.

```bash
ls -l /usr/bin/passwd
```

> 소유자 권한 자리에 `s`가 있음을 다시 확인한다.

## 단계 2. 위험한 SetUID의 재현(원리 이해)

관리자 권한으로 동작하는 파일에 함부로 SetUID를 부여하면 왜 위험한지, 안전한 실습 환경에서 재현한다.

```bash
mkdir -p ~/lab03/setuid && cd ~/lab03/setuid
```

```bash
cp /bin/cat ./mycat
```

```bash
ls -l mycat
```

> 일반 사용자 소유의 복사본이다. 아직 특수 권한이 없다.

```bash
chmod u+s mycat
ls -l mycat
```

> 소유자 권한 자리가 `-rwsr-xr-x`로 바뀌었다. 만약 이 파일의 소유자가 root였다면, 누구나 이 `mycat`으로 root 소유의 어떤 파일이든 읽을 수 있게 된다. **불필요한 SetUID가 위험한 이유가 바로 이것이다.**

## 단계 3. 시정 — 불필요한 SetUID 제거

```bash
chmod u-s mycat
ls -l mycat
```

> `s`가 사라지고 `-rwxr-xr-x`로 돌아왔다. 점검 중 정체불명의 SetUID 파일을 발견하면 이처럼 `u-s`로 특수 권한을 제거하거나 파일을 삭제하여야 한다.

## 단계 4. 정리

```bash
cd ~ && rm -rf ~/lab03/setuid
```

> **완료 기준**
> 1. `find -perm -4000`으로 시스템의 SetUID 파일 목록을 조회하였다.
> 2. SetUID 부여(`u+s`)와 제거(`u-s`)를 실습하였다.
> 3. 불필요한 SetUID가 왜 위험한지 자신의 표현으로 설명할 수 있다.
{: .prompt-tip }

---
---

# 자주 발생하는 오류와 대응 방법

---

| 화면에 출력된 메시지 | 원인 | 대응 방법 |
|---|---|---|
| `Permission denied` | 권한 부족 또는 소속 그룹 불일치 | `ls -l`로 권한을, `id`로 소속 그룹을 확인한다. |
| SetGID 후에도 그룹이 상속되지 않음 | 디렉터리에 `chmod g+s`(또는 2xxx)를 적용하지 않음 | `ls -ld`로 그룹 자리에 `s`가 있는지 확인한다. |
| `useradd: user already exists` | 이전 실습 계정이 남아 있음 | `sudo userdel -r 계정명`으로 삭제 후 다시 생성한다. |
| `sudo -u alice` 실행 시 홈 관련 경고 | 정상 동작 | 실습에는 영향이 없다. |
| 정리 후에도 파일이 남음 | `sudo` 누락 | 시스템 영역(`/srv` 등)의 삭제에는 `sudo`가 필요하다. |

---
---

# 요약

---

| 시나리오 | 핵심 학습 내용 |
|---|---|
| 1. 협업 디렉터리 | 그룹 기반 접근 통제와 SetGID를 이용한 그룹 소유권 상속 |
| 2. 취약 권한 시정 | 과도한 권한(`777`, `666`)의 진단과 최소 권한(`600`, `700`)으로의 시정 |
| 3. 특수 권한 점검 | `find -perm`으로 SetUID 점검, 불필요한 특수 권한의 위험과 제거 |

세 시나리오를 통하여, 접근 권한이 단순한 명령의 나열이 아니라 **요구사항을 만족시키면서 위험을 최소화하는 설계 활동**임을 확인하였다. "각 주체에게 필요한 최소한의 권한만 부여한다"는 최소 권한 원칙이 모든 시나리오를 관통하는 기준이다.

이번 실습으로 3주차의 사용자 계정과 접근 권한 학습을 마친다.
