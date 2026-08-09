---
title: 리눅스 기초 부록 종합문제 - 시스템 보안 감사
date: 2026-10-06 09:30:00 +0900
categories:
  - 0.기초강의
  - 리눅스 기초
tags:
  - 리눅스
  - Ubuntu2404
  - 종합문제
  - 복습
  - 보안점검
  - 감사
  - SetUID
  - 로그분석
  - Metasploitable
pin:
mermaid: false
---

> **종합문제 안내**
> 1. 본 글은 **리눅스 기초 과정 전체(제1강 ~ 제6강)** 를 하나의 점검 체계로 통합하는 마무리 종합문제이다.
> 2. **Metasploitable 2가 없어도 수행할 수 있다.** 취약 설정을 격리된 실습 영역에 직접 재현하여 탐지·시정하는 방식으로 진행하며, Metasploitable 2를 구축한 경우에는 문제 8에서 대조 감사를 추가로 수행한다.
> 3. 지금까지의 복습이 "명령을 익히는" 과정이었다면, 본 문제는 **"무엇을 점검해야 하는지 스스로 판단하는"** 훈련이다.
> 4. 정리 절차를 반드시 완료하여야 한다.
{: .prompt-info }

> **실습 환경에 관한 경고**
> 본 종합문제는 계정과 파일 권한을 일시적으로 취약하게 만든다. 반드시 **학습용 가상 머신에서만** 수행한다.
>
> 여기서 다루는 것은 **관리자가 자신의 시스템을 점검하는 방법**이며, 타인의 시스템을 대상으로 하는 행위는 명시적 허가 없이는 법적 책임을 수반한다.
{: .prompt-danger }

| 문제 | 주제 | 대응 강의 | 배정 시간 |
|---|---|---|---|
| 준비 | 감사 대상 환경 구성 | — | 10분 |
| 문제 1 | 감사 항목 설계 | 전 범위 | 10분 |
| 문제 2 | 계정 감사 | 3강 | 20분 |
| 문제 3 | 파일 권한 감사 | 3강 | 20분 |
| 문제 4 | 서비스와 포트 감사 | 5강 · 6강 | 15분 |
| 문제 5 | 접근 통제 감사 | 6강 | 15분 |
| 문제 6 | 로그 감사 | 2강 · 5강 | 15분 |
| 문제 7 | 통합 감사 도구 작성 | 4강 | 25분 |
| 문제 8 | 시정 조치와 재감사 | 전 범위 | 15분 |
| 문제 9 | Metasploitable 2 대조(선택) | 부록 | 20분 |
| 마무리 | 자가 채점과 정리 | — | 15분 |

---
---

# 시나리오

---

학습자는 정보보호팀의 **분기 보안 감사 담당자**로 지정되었다. 감사 대상은 운영 부서가 관리해 온 리눅스 서버 한 대이다.

정보보호 책임자는 다음과 같이 지시하였다.

> *"이번 감사는 도구를 사서 돌리는 방식이 아니라, **표준 명령만으로 무엇을 확인할 수 있는지** 정리하는 데 목적이 있습니다.*
> *계정·권한·서비스·접근통제·로그의 다섯 영역을 점검하고, 각 항목마다 '무엇을 근거로 정상이라고 판단했는지'를 함께 기록해 주십시오.*
> *마지막에는 같은 점검을 매 분기 반복할 수 있도록 자동화 도구로 남겨 주십시오."*

**보안은 별도의 도구가 아니라 기본 관리의 결과이다.** 본 종합문제는 그 사실을 스스로 확인하는 과정이다.

---
---

# 준비 단계. 감사 대상 환경 구성

---

아래 블록을 **한 번에 복사하여 실행**한다. 감사 과정에서 발견되어야 할 취약 설정이 구성된다.

```bash
mkdir -p ~/review07 && cd ~/review07
sudo mkdir -p /srv/audit_lab/shared

# (1) 설정 파일의 권한이 과도하게 열려 있다
echo "API_KEY=demo-1234" | sudo tee /srv/audit_lab/service.conf > /dev/null
sudo chmod 666 /srv/audit_lab/service.conf

# (2) 관리 도구에 SetUID가 설정되어 있다
sudo cp /usr/bin/head /srv/audit_lab/peek
sudo chmod 4755 /srv/audit_lab/peek

# (3) 공용 디렉터리에 스티키 비트가 없다
sudo chmod 777 /srv/audit_lab/shared

# (4) 이름을 위장한 관리자 권한 계정
sudo useradd -o -u 0 -g 0 -M -s /bin/bash sysmon

# (5) 서비스 계정인데 로그인 셸이 부여되어 있다
sudo useradd -r -M -s /bin/bash appworker

# (6) 비밀번호가 비어 있는 계정
sudo useradd -m -s /bin/bash guestuser
sudo passwd -d guestuser

# (7) 비밀번호 정책이 적용되지 않은 계정
sudo useradd -m -s /bin/bash devtest
echo "devtest:Audit#2026" | sudo chpasswd

echo "== 감사 대상 환경 구성 완료 =="
ls -l /srv/audit_lab
```

> 일곱 항목이 각각 어떤 문제인지는 **밝히지 않는다.** 문제 2와 문제 3에서 스스로 찾아내는 것이 본 종합문제의 목적이다.

---
---

# 문제 1. 감사 항목 설계

---

> **상황**
> 명령을 실행하기 전에 **무엇을 점검할 것인지** 먼저 정의하여야 한다. 항목 없이 명령부터 실행하면 감사가 아니라 탐색에 그친다.
>
> **요구사항**
> 다섯 영역에 대하여 각각 두 개 이상의 점검 항목과, 그 항목을 확인할 명령을 표로 정리한다.
>
> | 영역 | 점검 항목 | 확인 명령 | 정상 기준 |
> |---|---|---|---|
> | 계정 | ? | ? | ? |
> | 권한 | ? | ? | ? |
> | 서비스 | ? | ? | ? |
> | 접근 통제 | ? | ? | ? |
> | 로그 | ? | ? | ? |
{: .prompt-info }

<details markdown="1">
<summary><b>정답 예시 및 해설 보기</b></summary>

| 영역 | 점검 항목 | 확인 명령 | 정상 기준 |
|---|---|---|---|
| **계정** | UID 0 계정 | `awk -F: '$3==0' /etc/passwd` | `root` 하나뿐 |
| | 비밀번호 없는 계정 | `sudo awk -F: '$2==""' /etc/shadow` | 없음 |
| | 셸이 부여된 시스템 계정 | `awk -F: '$3<1000 && $3!=0 && $7 ~ /bash$/' /etc/passwd` | 없음 |
| | 비밀번호 정책 | `sudo chage -l 계정` | 최대 사용 기간이 설정됨 |
| **권한** | 누구나 쓸 수 있는 파일 | `sudo find /srv /etc -type f -perm -0002` | 없음 |
| | 비표준 SetUID | `sudo find / -perm -4000 -type f` | `/usr`·`/snap` 외에 없음 |
| | 공유 디렉터리의 스티키 비트 | `[ -k 경로 ]` | 설정됨 |
| | 홈 디렉터리 권한 | `ls -ld /home/*` | `750` 이하 |
| **서비스** | 대기 중인 포트 | `ss -tuln` | 업무에 필요한 포트만 |
| | 자동 시작 서비스 | `systemctl list-unit-files --state=enabled` | 불필요한 항목 없음 |
| **접근 통제** | 방화벽 상태 | `sudo ufw status` | `active` |
| | SSH 설정 | `grep -iE "PermitRootLogin\|PasswordAuthentication" /etc/ssh/sshd_config` | root 로그인 차단 |
| **로그** | 인증 실패 기록 | `sudo grep "Failed password" /var/log/auth.log` | 비정상적 반복 없음 |
| | 권한 상승 기록 | `sudo grep "sudo:" /var/log/auth.log` | 승인된 계정만 |

**해설**

- 감사의 출발점은 **"정상 기준"을 먼저 정의하는 것**이다. 기준이 없으면 출력을 보고도 정상인지 판단할 수 없다.
- 다섯 영역은 각각 다음 질문에 대응한다.
  - 계정 — **누가** 접근할 수 있는가
  - 권한 — **무엇에** 접근할 수 있는가
  - 서비스 — **어디로** 들어올 수 있는가
  - 접근 통제 — **어떻게** 차단하고 있는가
  - 로그 — **무슨 일이** 있었는가
- 이 다섯 질문은 규모와 무관하게 동일하게 적용된다.

</details>

---
---

# 문제 2. 계정 감사

---

> **상황**
> 첫 번째 영역을 점검한다. 준비 단계에서 구성된 계정 관련 문제 **네 가지**를 모두 찾아내야 한다.
>
> **요구사항**
> 1. UID가 0인 계정을 조회한다.
> 2. 비밀번호가 설정되지 않은 계정을 조회한다.
> 3. 로그인 셸이 부여된 시스템 계정을 조회한다.
> 4. 일반 사용자 계정의 **비밀번호 정책 적용 여부**를 조회한다.
> 5. 계정 잠금 상태를 확인한다.
> 6. 각 항목이 왜 위험한지 한 문장으로 설명한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
echo "--- ① UID 0 계정 ---"
awk -F: '$3 == 0 {print $1, $3, $7}' /etc/passwd

echo "--- ② 비밀번호 없는 계정 ---"
sudo awk -F: '($2 == "") {print $1}' /etc/shadow

echo "--- ③ 셸이 부여된 시스템 계정 ---"
awk -F: '$3 < 1000 && $3 != 0 && $7 ~ /(bash|zsh|ksh)$/ {print $1, $3, $7}' /etc/passwd

echo "--- ④ 일반 사용자와 비밀번호 정책 ---"
for u in $(awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd); do
  echo "[$u] $(sudo chage -l "$u" | grep -i 'Maximum number')"
done

echo "--- ⑤ 계정 상태 ---"
for u in $(awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd); do
  sudo passwd -S "$u"
done
```

**발견되어야 할 항목**

| 발견 | 계정 | 위험성 |
|---|---|---|
| ① | **`sysmon`** | 커널은 **이름이 아니라 UID로 권한을 판정**하므로, UID 0인 계정은 이름과 무관하게 root와 동일한 전권을 갖는다. |
| ② | **`guestuser`** | 비밀번호 필드가 비어 있으면 **아무 입력 없이 인증이 통과**될 수 있다. |
| ③ | **`appworker`** | 서비스 계정에 셸이 있으면 계정 탈취 시 곧바로 명령을 실행할 발판이 된다. |
| ④ | **`devtest`** | `Maximum number of days`가 `99999`이면 **비밀번호를 영구히 바꾸지 않아도 되는 상태**이다. |

**해설**

- `passwd -S`의 두 번째 항목은 계정 상태를 나타낸다. **`P`는 정상 설정, `L`은 잠김, `NP`는 비밀번호 없음**이다.
- `chpasswd`는 비밀번호를 일괄 설정하는 명령으로, `계정:비밀번호` 형식을 입력받는다. 스크립트에서 사용하지만 **명령 이력에 비밀번호가 남으므로** 실무에서는 파일 입력이나 대화식 `passwd`를 사용한다.
- 이 네 항목은 **로그인 시도 없이 파일 조회만으로** 확인된다. 감사에서 조회 명령이 강력한 도구가 되는 이유이다.

</details>

**완료 기준** — `sysmon`·`guestuser`·`appworker`·`devtest` 네 계정의 문제를 모두 식별하였다.

---
---

# 문제 3. 파일 권한 감사

---

> **상황**
> 두 번째 영역을 점검한다. 준비 단계에서 구성된 권한 관련 문제 **세 가지**를 찾아내야 한다.
>
> **요구사항**
> 1. `/srv`와 `/etc` 아래에서 **누구나 쓸 수 있는 일반 파일**을 조회한다.
> 2. 시스템 전체에서 **표준 경로 밖의 SetUID·SetGID 파일**을 조회한다.
> 3. 모든 사용자가 쓸 수 있는 디렉터리 중 **스티키 비트가 없는 것**을 조회한다.
> 4. 일반 사용자 홈 디렉터리의 권한을 점검한다.
> 5. 소유자가 존재하지 않는 파일이 있는지 확인한다.
{: .prompt-info }

**힌트** — `find`의 `-perm -0002`, `-perm -4000`, `-perm -1000`, `-nouser` 조건을 사용한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
echo "--- ① 누구나 쓸 수 있는 파일 ---"
sudo find /srv /etc -type f -perm -0002 -exec ls -l {} +

echo "--- ② 표준 경로 밖의 SetUID / SetGID ---"
sudo find / -perm -4000 -type f 2>/dev/null | grep -vE "^/usr|^/snap"
sudo find / -perm -2000 -type f 2>/dev/null | grep -vE "^/usr|^/snap"

echo "--- ③ 스티키 비트가 없는 쓰기 개방 디렉터리 ---"
sudo find /srv /tmp /var/tmp -type d -perm -0002 ! -perm -1000 2>/dev/null

echo "--- ④ 홈 디렉터리 권한 ---"
ls -ld /home/*

echo "--- ⑤ 소유자 없는 파일 ---"
sudo find / -xdev -nouser 2>/dev/null | head
```

**발견되어야 할 항목**

| 발견 | 대상 | 위험성 |
|---|---|---|
| ① | **`/srv/audit_lab/service.conf`**(666) | 임의의 사용자가 설정을 변조할 수 있다. 접속 정보를 바꾸어 통신을 가로채는 공격이 가능하다. |
| ② | **`/srv/audit_lab/peek`**(4755) | 파일 내용을 출력하는 도구가 **root 권한으로 동작**하므로, 일반 사용자가 시스템의 모든 파일을 열람할 수 있다. |
| ③ | **`/srv/audit_lab/shared`**(777) | 스티키 비트가 없으므로 **누구나 남의 파일을 삭제**할 수 있다. |

**직접 확인해 본다.**

```bash
sudo -u guestuser /srv/audit_lab/peek -2 /etc/shadow
```

일반 사용자가 `/etc/shadow`의 내용을 열람할 수 있다. `peek`은 정상적인 프로그램이고 취약한 코드도 아니지만, **root 소유 + SetUID라는 조합만으로** 시스템의 가장 중요한 파일이 노출된다.

**해설**

- `-perm -0002`의 앞선 `-`는 "**해당 비트를 포함하는**"이라는 의미이다. 부호 없는 `-perm 0002`는 권한이 정확히 일치하는 경우만 찾으므로 실무에서는 쓰이지 않는다.
- `! -perm -1000`은 "스티키 비트를 **포함하지 않는**"이라는 조건이다. `!`는 조건을 뒤집는다.
- 우분투는 홈 디렉터리를 기본 `750`으로 생성한다(`/etc/login.defs`의 `HOME_MODE`). `755`나 `777`로 되어 있으면 타인이 개인 자료를 열람할 수 있다.
- `-nouser`는 계정이 삭제되었으나 파일이 남아 소유자를 찾을 수 없는 상태를 찾는다. 이후 같은 UID가 다른 사용자에게 배정되면 **그 사용자가 이전 사용자의 파일을 그대로 소유**하게 되므로, 계정 회수 시 반드시 점검한다.

</details>

**완료 기준** — 세 항목을 모두 식별하고, SetUID의 위험성을 실제 실행으로 확인하였다.

---
---

# 문제 4. 서비스와 포트 감사

---

> **상황**
> 세 번째 영역을 점검한다. **공격 표면**을 파악하는 단계이다.
>
> **요구사항**
> 1. 대기 중인 포트를 조회하고 개수를 센다.
> 2. 각 포트를 점유한 프로그램을 확인한다.
> 3. 부팅 시 자동 시작되도록 등록된 서비스 목록을 조회한다.
> 4. 실행 중인 서비스의 개수를 확인한다.
> 5. 조회된 포트가 각각 어떤 서비스인지 `/etc/services`와 대조한다.
> 6. **공격 표면**의 개념으로 결과를 해석한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
ss -tuln
ss -tlnH | wc -l
sudo ss -tulnp

systemctl list-unit-files --type=service --state=enabled | head -20
systemctl list-units --type=service --state=running | wc -l

ss -tlnH | awk '{print $4}' | awk -F: '{print $NF}' | sort -u
grep -E "^(ssh|http|https|domain)[[:space:]]" /etc/services
```

**해설**

- **개방된 포트 하나하나가 외부에서 접근할 수 있는 통로**이며, 이 통로의 총합을 **공격 표면(Attack Surface)** 이라 한다. 통로가 많을수록 그중 하나에 취약점이 존재할 확률이 높아진다.
- 기본 설치 상태의 Ubuntu Server는 대기 포트가 매우 적다. `22`(SSH)와 `53`(systemd-resolved의 로컬 스텁) 정도가 일반적이다. 이는 **필요한 것만 설치하고 나머지는 두지 않는다**는 원칙이 지켜진 결과이다.
- 22번 포트의 점유 프로세스가 `sshd`가 아니라 `systemd`로 표시될 수 있다. **소켓 활성화** 방식이기 때문이다.
- `systemctl list-unit-files --state=enabled`는 **부팅 시 자동 시작**되는 목록이다. 여기에 업무와 무관한 서비스가 있으면 `systemctl disable`로 해제한다. **"사용하지 않는 서비스는 중지하고 필요한 포트만 개방한다"** 는 원칙이 서버 강화의 기본이다.
- 감사에서는 **개수 자체보다 "목록의 모든 항목을 설명할 수 있는가"** 가 중요하다. 설명하지 못하는 포트가 있다면 그것이 곧 점검 대상이다.

</details>

**완료 기준** — 대기 중인 모든 포트에 대해 어떤 서비스인지 설명할 수 있다.

---
---

# 문제 5. 접근 통제 감사

---

> **상황**
> 네 번째 영역을 점검한다.
>
> **요구사항**
> 1. 방화벽의 활성화 여부와 기본 정책을 확인한다.
> 2. 등록된 규칙을 번호와 함께 조회한다.
> 3. SSH 서버 설정에서 **root 직접 로그인**과 **비밀번호 인증** 항목을 확인한다.
> 4. SSH 설정의 문법을 검사한다.
> 5. `sudo` 권한을 가진 계정과 위임 규칙을 확인한다.
> 6. 각 항목의 권장 설정과 그 근거를 정리한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
sudo ufw status verbose
sudo ufw status numbered

grep -iE "^#?(PermitRootLogin|PasswordAuthentication|PubkeyAuthentication|Port|AllowUsers)" /etc/ssh/sshd_config
sudo sshd -t && echo "SSH 설정 문법 정상"

getent group sudo
sudo grep -vE "^#|^$" /etc/sudoers
ls -l /etc/sudoers.d/
```

**권장 설정과 근거**

| 항목 | 권장값 | 근거 |
|---|---|---|
| 방화벽 기본 정책 | `deny (incoming)` | 명시적으로 허용한 것만 통과시킨다(화이트리스트 방식) |
| `PermitRootLogin` | `no` | root는 모든 시스템에 존재하는 이름이므로 **무차별 대입의 첫 번째 표적**이 된다 |
| `PasswordAuthentication` | `no`(키 인증 구성 후) | 비밀번호 추측 공격 자체를 차단한다 |
| `AllowUsers` | 필요한 계정만 나열 | 접근 주체를 최소화한다 |
| `sudo` 그룹 | 필요한 인원만 | 권한 상승 경로를 최소화한다 |

**해설**

- **`PasswordAuthentication no`로 바꾸기 전에는 반드시 별도의 세션에서 키 인증이 동작하는지 확인**하여야 한다. 확인 없이 적용하고 세션을 종료하면 접속 수단을 완전히 상실한다.
- 설정 변경 후에는 재시작 전에 **`sudo sshd -t`** 로 문법을 검사한다. 오류가 있는 설정으로 재시작하면 SSH 서비스가 기동되지 않아 원격 복구가 불가능해진다.
- `sudo`는 `su`와 달리 **자기 자신의 비밀번호로 인증하고, 허용 범위를 지정할 수 있으며, 실행 기록이 `/var/log/auth.log`에 남는다.** 감사 관점에서 `su`보다 `sudo`가 권장되는 이유가 이 **추적 가능성**이다.
- 개별 위임 규칙은 `/etc/sudoers.d/` 아래 파일로 분리하여 관리하며, 편집은 반드시 `visudo`로 수행한다.

</details>

**완료 기준** — 방화벽 상태와 SSH 설정 항목을 근거와 함께 설명할 수 있다.

---
---

# 문제 6. 로그 감사

---

> **상황**
> 다섯 번째 영역이다. **무슨 일이 있었는지**를 기록에서 확인한다.
>
> **요구사항**
> 1. 인증 실패 기록을 **의도적으로 발생**시킨다(존재하지 않는 계정으로 접속을 시도한다).
> 2. `/var/log/auth.log`에서 인증 실패 기록을 찾는다.
> 3. 권한 상승(`sudo`) 사용 기록을 조회한다.
> 4. 로그인 실패 이력을 전용 명령으로 조회한다.
> 5. 시스템 전체의 오류 수준 로그를 조회한다.
> 6. 로그를 **실시간으로 추적**하는 방법을 확인한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
# ① 인증 실패를 의도적으로 발생시킨다 (아무 문자열이나 세 번 입력한 뒤 Ctrl+C 로 중단)
ssh nosuchuser@localhost

# ② 인증 실패 기록
sudo grep -E "Failed password|Invalid user" /var/log/auth.log | tail -5
sudo grep -cE "Failed password|Invalid user" /var/log/auth.log

# ③ 권한 상승 기록
sudo grep "sudo:" /var/log/auth.log | tail -5

# ④ 로그인 실패 이력
sudo lastb | head -5
last | head -5

# ⑤ 오류 수준 로그
sudo journalctl -p err -b --no-pager | tail -10

# ⑥ 실시간 추적 (Ctrl+C 로 종료)
sudo tail -f /var/log/auth.log
```

**해설**

- `/var/log/auth.log`에는 **인증과 권한 상승에 관한 모든 기록**이 남는다. 침해 대응에서 가장 먼저 확인하는 파일이다.
- **`Failed password`가 짧은 시간에 반복**되면 무차별 대입 시도로 판단한다. 출발지 IP를 함께 확인하여 방화벽에서 차단하거나 `fail2ban`과 같은 도구를 도입한다.
- `last`는 성공한 로그인 이력(`/var/log/wtmp`), **`lastb`는 실패한 로그인 이력(`/var/log/btmp`)** 을 보여 준다. `lastb`는 관리자 권한이 필요하다.
- `journalctl -p err -b`는 **이번 부팅 이후의 오류 이상 수준** 기록만 추린다. `-p`는 우선순위, `-b`는 부팅 기준이다.
- `tail -f`는 새로 기록되는 내용을 즉시 이어서 출력한다. 장애나 공격이 진행 중일 때 가장 먼저 사용하는 형태이다.
- `sudo` 기록에는 **실행한 사용자·시각·작업 디렉터리·명령**이 모두 남는다. 이것이 감사에서 `su`보다 `sudo`를 요구하는 이유이다.

</details>

**완료 기준** — 자신이 발생시킨 인증 실패 기록을 로그에서 찾아냈다.

---
---

# 문제 7. 통합 감사 도구 작성

---

> **상황**
> 마지막 요청("매 분기 반복할 수 있도록 자동화")을 구현한다.
>
> **요구사항**
> 다섯 영역의 점검을 하나의 스크립트로 통합하되, 각 항목마다 **`[정상]` 또는 `[점검필요]`로 판정**하여 출력한다. 판정 근거가 되는 값도 함께 표시한다.
{: .prompt-info }

**힌트** — 특수 권한은 `grep`이 아니라 `[ -k ]`·`[ -u ]`·`[ -g ]`로 검사한다. `sudo`가 필요한 항목은 `sudo -n`과 `if`로 처리한다.

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
cat > ~/review07/audit.sh << 'EOF'
#!/bin/bash
# 리눅스 기초 과정 통합 보안 감사 도구

PASS="[정상]  "
WARN="[점검필요]"
SKIP="[확인불가]"

echo "=========================================="
echo "  시스템 보안 감사 보고"
echo "  점검 일시: $(date '+%Y-%m-%d %H:%M:%S')"
echo "  대상 호스트: $(hostname)"
echo "=========================================="
echo

# 관리자 권한 사용 가능 여부를 먼저 판정한다
if sudo -n true 2>/dev/null; then SUDO_OK=1; else SUDO_OK=0; fi
[ "$SUDO_OK" -eq 0 ] && echo "주의: 관리자 권한 없이 실행되어 일부 항목을 확인할 수 없습니다." && echo

echo "── 1. 계정 ──"
N=$(awk -F: '$3==0' /etc/passwd | wc -l)
[ "$N" -eq 1 ] && echo "$PASS UID 0 계정: root 하나" \
               || echo "$WARN UID 0 계정 $N 개: $(awk -F: '$3==0 {print $1}' /etc/passwd | tr '\n' ' ')"

if [ "$SUDO_OK" -eq 1 ]; then
  EMPTY=$(sudo -n awk -F: '$2=="" {print $1}' /etc/shadow | tr '\n' ' ')
  [ -z "$EMPTY" ] && echo "$PASS 빈 비밀번호 계정 없음" \
                  || echo "$WARN 빈 비밀번호 계정: $EMPTY"
else
  echo "$SKIP 빈 비밀번호 계정(관리자 권한 필요)"
fi

SHELLED=$(awk -F: '$3<1000 && $3!=0 && $7 ~ /(bash|zsh|ksh)$/ {print $1}' /etc/passwd | tr '\n' ' ')
[ -z "$SHELLED" ] && echo "$PASS 셸 보유 시스템 계정 없음" \
                  || echo "$WARN 셸 보유 시스템 계정: $SHELLED"
echo

echo "── 2. 파일 권한 ──"
if [ "$SUDO_OK" -eq 1 ]; then
  WW=$(sudo -n find /srv /etc -type f -perm -0002 2>/dev/null | wc -l)
  [ "$WW" -eq 0 ] && echo "$PASS 쓰기 개방 파일 없음" \
                  || echo "$WARN 쓰기 개방 파일 $WW 건"

  SUID=$(sudo -n find / -perm -4000 -type f 2>/dev/null | grep -vcE '^/usr|^/snap')
  [ "$SUID" -eq 0 ] && echo "$PASS 비표준 SetUID 없음" \
                    || echo "$WARN 비표준 SetUID $SUID 건"
else
  echo "$SKIP 쓰기 개방 파일 / 비표준 SetUID(관리자 권한 필요)"
fi

[ -k /tmp ] && echo "$PASS /tmp 스티키 비트 설정됨" \
            || echo "$WARN /tmp 스티키 비트 없음"
echo

echo "── 3. 서비스와 포트 ──"
echo "        대기 포트 $(ss -tlnH | wc -l) 개 / 실행 서비스 $(systemctl list-units --type=service --state=running --no-legend 2>/dev/null | wc -l) 개"
ss -tlnH | awk '{print "        - " $4}' | sort -u
echo

echo "── 4. 접근 통제 ──"
if FW=$(sudo -n ufw status 2>/dev/null); then
  echo "$FW" | head -1 | grep -q active \
    && echo "$PASS 방화벽 활성화" || echo "$WARN 방화벽 비활성화"
else
  echo "        방화벽 상태: 권한 없음(관리자 권한으로 실행할 것)"
fi

grep -qiE "^PermitRootLogin[[:space:]]+no" /etc/ssh/sshd_config \
  && echo "$PASS SSH root 로그인 차단" \
  || echo "$WARN SSH root 로그인 설정 확인 필요"
echo

echo "── 5. 로그 ──"
if [ "$SUDO_OK" -eq 1 ]; then
  FAIL=$(sudo -n grep -cE "Failed password|Invalid user" /var/log/auth.log 2>/dev/null)
  echo "        인증 실패 기록: ${FAIL:-0} 건"
else
  echo "        인증 실패 기록: 확인 불가(관리자 권한 필요)"
fi
echo "        최근 로그인:"
last -n 3 2>/dev/null | head -3 | sed 's/^/        /'
EOF

chmod 750 ~/review07/audit.sh
~/review07/audit.sh
~/review07/audit.sh > ~/review07/audit_before.txt
```

**해설**

- **판정을 사람이 아니라 스크립트가 하도록** 작성하였다. 값만 출력하면 매번 사람이 해석해야 하지만, 기준을 코드로 적어 두면 누가 실행해도 같은 결론이 나온다.
- 특수 권한은 `ls | grep t`처럼 문자열로 판정해서는 안 된다. **경로에 우연히 포함된 문자 때문에 오판**하기 때문이며, `[ -k ]`·`[ -u ]`·`[ -g ]` 전용 검사를 사용한다.
- `sudo -n`은 비밀번호를 묻지 않고 즉시 실패한다. 결과를 **변수에 담은 뒤 `if`로 판정**하는 이유는, 파이프라인의 종료 상태가 마지막 명령의 것이어서 `||`가 의도대로 동작하지 않기 때문이다.
- 스크립트 앞부분에서 **`sudo -n true`로 관리자 권한 사용 가능 여부를 먼저 판정**하는 것이 중요하다. 이 판정이 없으면, 권한이 없어 조회에 실패한 경우에도 결과가 비어 있다는 이유로 **`[정상]`으로 오판**한다. 감사 도구에서 "확인하지 못한 것"과 "확인했더니 문제가 없는 것"은 반드시 구분되어야 한다.
- `grep -vcE`는 일치하지 않는 행의 개수를 센다. 입력이 비어 있으면 `0`을 출력하므로 숫자 비교에 그대로 사용할 수 있다.
- 결과를 `audit_before.txt`로 남겨 두면 **문제 8의 시정 전후 비교**에 사용할 수 있다.

</details>

**완료 기준** — 다섯 영역이 모두 출력되고, 준비 단계에서 만든 취약 항목이 `[점검필요]`로 판정된다.

---
---

# 문제 8. 시정 조치와 재감사

---

> **상황**
> 감사에서 발견한 항목을 모두 시정하고, **동일한 도구로 재감사하여 개선을 입증**하여야 한다.
>
> **요구사항**
> 1. 문제 2·3에서 발견한 일곱 항목을 시정한다.
> 2. 시정 후 감사 도구를 다시 실행하여 결과를 파일로 남긴다.
> 3. **시정 전후의 결과를 비교**한다.
> 4. 감사 결과 보고서를 작성한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

```bash
# ① UID 0 위장 계정 제거
sudo userdel sysmon

# ② 빈 비밀번호 계정 잠금
sudo passwd -l guestuser
sudo passwd -S guestuser

# ③ 시스템 계정의 셸 차단
sudo usermod -s /usr/sbin/nologin appworker
grep "^appworker:" /etc/passwd

# ④ 비밀번호 정책 적용
sudo chage -M 90 -m 7 -W 14 devtest
sudo chage -l devtest | grep -i "maximum"

# ⑤ 쓰기 개방 파일 시정
sudo chmod 640 /srv/audit_lab/service.conf
sudo chown root:root /srv/audit_lab/service.conf
ls -l /srv/audit_lab/service.conf

# ⑥ SetUID 제거
sudo chmod u-s /srv/audit_lab/peek
ls -l /srv/audit_lab/peek

# ⑦ 스티키 비트 설정
sudo chmod 1777 /srv/audit_lab/shared
ls -ld /srv/audit_lab/shared

# 재감사
~/review07/audit.sh > ~/review07/audit_after.txt
diff ~/review07/audit_before.txt ~/review07/audit_after.txt
```

**⑧ SSH root 로그인 차단(선택 — 준비 단계에서 만든 항목이 아니라 기본 강화 항목이다)**

감사 도구의 `4. 접근 통제`에서 `SSH root 로그인 설정 확인 필요`가 표시된다. 우분투는 해당 항목이 **주석 처리된 상태**로 배포되므로, 명시적으로 지정해 두는 것이 권장된다.

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
echo "PermitRootLogin no" | sudo tee -a /etc/ssh/sshd_config > /dev/null
sudo sshd -t && echo "문법 정상"
sudo systemctl reload ssh
grep -i "^PermitRootLogin" /etc/ssh/sshd_config
```

> **주의** — 현재 `student` 계정으로 접속 중이므로 이 설정은 자신의 접속에 영향을 주지 않는다. 다만 **root로 직접 접속하고 있었다면 즉시 차단**되므로, 반드시 `whoami`로 자신의 계정을 먼저 확인한다. 변경 후에는 `sshd -t`로 문법을 검사한 뒤 `reload`한다.
>
> 되돌리려면 `sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config && sudo systemctl reload ssh`를 실행한다.

**보고서 작성**

```bash
{
  echo "===== 분기 보안 감사 결과 보고서 ====="
  echo "감사 일시: $(date '+%Y-%m-%d %H:%M')"
  echo "감사 대상: $(hostname)"
  echo "감사자   : $(whoami)"
  echo
  echo "[1] 발견 및 조치 내역"
  echo "  1. UID 0 위장 계정(sysmon)        → 계정 삭제"
  echo "  2. 빈 비밀번호 계정(guestuser)     → 계정 잠금"
  echo "  3. 셸 보유 시스템 계정(appworker)  → nologin으로 변경"
  echo "  4. 비밀번호 정책 미적용(devtest)   → 90일 정책 적용"
  echo "  5. 쓰기 개방 설정 파일(666)        → 640으로 조정"
  echo "  6. 비표준 SetUID 파일(peek)        → SetUID 제거"
  echo "  7. 스티키 비트 없는 공유 디렉터리  → 1777로 조정"
  echo "  (선택) SSH root 로그인 설정 미지정  → PermitRootLogin no 명시"
  echo
  echo "[2] 재감사 결과"
  cat ~/review07/audit_after.txt
} > ~/review07/audit_report.txt

cat ~/review07/audit_report.txt
```

**해설**

- **감사는 발견으로 끝나지 않는다.** 시정 조치와 **재감사에 의한 확인**까지 이루어져야 하나의 주기가 완성된다.
- `diff`로 전후를 비교하면 무엇이 개선되었는지 한눈에 드러난다. 보고서에 근거를 남기는 가장 간단한 방법이다.
- 계정을 **삭제할지 잠글지**는 상황에 따라 다르다. 위장 계정처럼 존재 자체가 문제인 경우는 삭제하고, 업무상 필요하나 사용을 중지할 계정은 **잠금**하여 이력을 보존한다.
- `usermod -s /usr/sbin/nologin`은 로그인을 차단하지만 서비스 구동에는 영향을 주지 않는다. 서비스 계정 강화의 표준 조치이다.

</details>

**완료 기준** — 재감사 결과에서 **1. 계정**과 **2. 파일 권한** 영역의 `[점검필요]`가 모두 사라진다. `4. 접근 통제`의 SSH 항목은 ⑧을 수행한 경우에만 `[정상]`으로 바뀐다.

---
---

# 문제 9. Metasploitable 2 대조 감사(선택)

---

> **본 문제는 부록에서 Metasploitable 2를 구축한 경우에만 수행한다.** 구축하지 않았다면 건너뛰어도 무방하다.
{: .prompt-warning }

> **상황**
> 같은 감사 항목을 **의도적으로 취약하게 구성된 시스템**에 적용하여, 설정의 차이가 어떤 결과를 만드는지 확인한다.
>
> **요구사항**
> 1. Metasploitable 2에 접속하여 동일한 항목을 조회한다(구형 명령을 사용한다).
> 2. 두 시스템의 수치를 비교표로 정리한다.
> 3. 차이의 원인이 **기능의 차이인지 설정의 차이인지** 판단한다.
{: .prompt-info }

<details markdown="1">
<summary><b>정답 및 해설 보기</b></summary>

Metasploitable 2에서 실행한다.

```bash
echo "전체 계정 수: $(wc -l < /etc/passwd)"
echo "UID 0 계정: $(awk -F: '$3==0 {print $1}' /etc/passwd | tr '\n' ' ')"
echo "SetUID 파일 수: $(sudo find / -perm -4000 -type f 2>/dev/null | wc -l)"
echo "개방 포트 수: $(netstat -tuln | grep -c LISTEN)"
echo "/tmp 권한: $(ls -ld /tmp | awk '{print $1}')"
echo "실행 데몬 수: $(ps aux | wc -l)"
```

Ubuntu Server에서 같은 항목을 조회한다.

```bash
echo "전체 계정 수: $(wc -l < /etc/passwd)"
echo "UID 0 계정: $(awk -F: '$3==0 {print $1}' /etc/passwd | tr '\n' ' ')"
echo "SetUID 파일 수: $(sudo find / -perm -4000 -type f 2>/dev/null | wc -l)"
echo "개방 포트 수: $(ss -tuln | grep -c LISTEN)"
echo "/tmp 권한: $(ls -ld /tmp | awk '{print $1}')"
echo "실행 데몬 수: $(ps aux | wc -l)"
```

| 점검 항목 | Ubuntu 24.04 | Metasploitable 2 | 판단 |
|---|---|---|---|
| 전체 계정 수 | | | |
| SetUID 파일 수 | | | |
| 개방 포트 수 | | | |
| 평문 서비스(Telnet·FTP) | | | |
| `/tmp` 스티키 비트 | | | |

**해설**

- Metasploitable 2에는 `ss`가 없으므로 구형 `netstat`을, `ip` 대신 `ifconfig`를 사용한다. 실무에서는 노후 시스템을 인수하는 경우가 적지 않으므로 **양쪽 세대의 명령을 모두 읽을 수 있어야** 한다.
- 개방 포트 수와 SetUID 파일 수의 차이가 매우 크게 나타난다. 이 차이는 **운영체제의 성능이나 기능 때문이 아니라 설정 때문**이다.
- 즉 제3강의 계정·권한 관리, 제5강의 불필요한 서비스 중지, 제6강의 방화벽 설정을 적용하지 않으면 **어떤 최신 시스템도 유사한 상태가 될 수 있다.**

</details>

---
---

# 마무리. 자가 채점

---

```bash
echo "===== 자가 채점 결과 ====="

[ "$(awk -F: '$3==0' /etc/passwd | wc -l)" -eq 1 ] \
  && echo "[문제 8-①] 통과 — UID 0은 root뿐" || echo "[문제 8-①] 미완료"

[ -z "$(sudo awk -F: '$2=="" {print $1}' /etc/shadow)" ] \
  && echo "[문제 8-②] 통과 — 빈 비밀번호 없음" || echo "[문제 8-②] 미완료"

[ -z "$(awk -F: '$3<1000 && $3!=0 && $7 ~ /(bash|zsh|ksh)$/ {print $1}' /etc/passwd)" ] \
  && echo "[문제 8-③] 통과 — 시스템 계정 셸 차단" || echo "[문제 8-③] 미완료"

sudo chage -l devtest 2>/dev/null | grep -qi "90" \
  && echo "[문제 8-④] 통과 — 비밀번호 정책 적용" || echo "[문제 8-④] 미완료"

[ "$(sudo find /srv /etc -type f -perm -0002 2>/dev/null | wc -l)" -eq 0 ] \
  && echo "[문제 8-⑤] 통과 — 쓰기 개방 파일 없음" || echo "[문제 8-⑤] 미완료"

[ "$(sudo find / -perm -4000 -type f 2>/dev/null | grep -vcE '^/usr|^/snap')" -eq 0 ] \
  && echo "[문제 8-⑥] 통과 — 비표준 SetUID 없음" || echo "[문제 8-⑥] 미완료"

[ -k /srv/audit_lab/shared ] \
  && echo "[문제 8-⑦] 통과 — 스티키 비트 설정" || echo "[문제 8-⑦] 미완료"

[ -x ~/review07/audit.sh ] && [ -s ~/review07/audit_after.txt ] \
  && echo "[문제 7] 통과 — 감사 도구 작성" || echo "[문제 7] 미완료"

[ -s ~/review07/audit_report.txt ] \
  && echo "[문제 8-보고서] 통과 — 보고서 작성" || echo "[문제 8-보고서] 미완료"
```

---

## 이론 점검

**문항 1.** 계정 이름이 `sysmon`인데 root와 동일한 권한을 갖는 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

커널은 **UID로 권한을 판정**하며 **UID 0이면 관리자**로 취급하기 때문이다. 계정 이름은 권한과 무관하다.

</details>

**문항 2.** 정상적인 프로그램인 `head`를 복사한 파일이 위험해진 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**root 소유 + SetUID**라는 조합 때문이다. 실행되는 동안 root 권한으로 동작하므로, 파일 내용을 출력하는 기능만으로도 `/etc/shadow` 같은 파일이 노출된다. 프로그램 자체의 결함이 아니라 **설정의 문제**이다.

</details>

**문항 3.** 개방된 포트가 많을수록 위험이 증가하는 것을 설명하는 개념은?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**공격 표면(Attack Surface)의 확대**이다. 통로가 많을수록 그중 하나에 취약점이 존재할 확률이 높아진다.

</details>

**문항 4.** 감사에서 `su`보다 `sudo`가 권장되는 가장 큰 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**추적 가능성**이다. `sudo`는 실행한 사용자·시각·명령이 `/var/log/auth.log`에 기록되며, 허용 범위를 규칙으로 제한할 수도 있다.

</details>

**문항 5.** 로그인 성공 이력과 실패 이력을 각각 조회하는 명령은?

<details markdown="1">
<summary><b>정답 보기</b></summary>

성공은 **`last`**(`/var/log/wtmp`), 실패는 **`lastb`**(`/var/log/btmp`)이다. `lastb`는 관리자 권한이 필요하다.

</details>

**문항 6.** 점검 스크립트에서 스티키 비트를 `ls -ld 경로 | grep t`로 판정하면 안 되는 이유는?

<details markdown="1">
<summary><b>정답 보기</b></summary>

**경로 문자열이나 날짜에 우연히 `t`가 포함되면 항상 참으로 판정**되기 때문이다. `[ -k 경로 ]`처럼 해당 비트만 검사하는 전용 조건을 사용하여야 한다.

</details>

---
---

# 정리 절차 — 반드시 수행

---

```bash
sudo rm -rf /srv/audit_lab
```

```bash
sudo userdel -r guestuser
sudo userdel -r devtest
sudo userdel appworker
sudo userdel sysmon 2>/dev/null
```

```bash
rm -rf ~/review07
```

**정리 결과를 검증한다.**

```bash
grep -E "^(sysmon|appworker|guestuser|devtest):" /etc/passwd || echo "① 실습 계정 전부 삭제 완료"
```

```bash
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

```bash
sudo find / -perm -4000 -type f 2>/dev/null | grep -vE "^/usr|^/snap" || echo "③ 비표준 SetUID 없음"
```

```bash
sudo find / -xdev -nouser 2>/dev/null | head
```

> **검증 기준**
> ① "실습 계정 전부 삭제 완료"가 출력된다.
> ② UID 0 계정이 **`root` 하나뿐**이다.
> ③ "비표준 SetUID 없음"이 출력된다.
> ④ 아무것도 출력되지 않는다(소유자 없는 파일이 남지 않았다).
>
> 문제 8의 ⑧에서 추가한 `PermitRootLogin no` 설정은 **되돌리지 않고 유지할 것을 권장한다.** 보안을 강화하는 설정이며 본 과정의 실습에 아무런 지장을 주지 않는다.
{: .prompt-danger }

---
---

# 과정 마무리

---

## 여섯 강의가 하나로 이어지는 지점

| 감사 영역 | 사용한 강의 내용 |
|---|---|
| 계정 | 제3강 — `/etc/passwd`·`/etc/shadow` 필드 해석, `chage`, `passwd -S` |
| 권한 | 제3강 — `chmod`, 특수 권한, `find -perm` |
| 서비스·포트 | 제5강 `systemctl` · 제6강 `ss` |
| 접근 통제 | 제6강 — `ufw`, `sshd_config` |
| 로그 | 제2강 `grep`·`tail` · 제5강 `journalctl` |
| 자동화 | 제4강 — 변수, 리다이렉션, 파이프, 조건 판정, `vi` |
| 환경 이해 | 제1강 — 부팅, 파일 시스템, 프롬프트 해석 |

**본 과정에서 배운 명령은 그 자체로는 단순한 조회 도구이다. 그러나 무엇을 확인해야 하는지 알고 있을 때 비로소 점검 도구가 된다.** `find`·`awk`·`grep`은 보안 전용 도구가 아니지만, 이 종합문제에서는 시스템의 위험 요소를 식별하는 수단으로 사용되었다.

---

## 자기 점검

```
 [ ] 다섯 영역의 감사 항목과 정상 기준을 스스로 설계할 수 있다.
 [ ] UID 0 계정·빈 비밀번호·셸 보유 시스템 계정을 조회할 수 있다.
 [ ] 쓰기 개방 파일과 비표준 SetUID를 탐지하고 시정할 수 있다.
 [ ] 개방 포트를 조회하고 각각의 용도를 설명할 수 있다.
 [ ] 방화벽과 SSH 설정의 권장값을 근거와 함께 제시할 수 있다.
 [ ] 인증 실패 기록을 로그에서 찾아낼 수 있다.
 [ ] 판정 기준을 코드로 표현한 감사 스크립트를 작성할 수 있다.
 [ ] 시정 후 재감사로 개선을 입증하는 절차를 수행하였다.
 [ ] 정리 절차를 완료하여 시스템을 원래 상태로 되돌렸다.
```

---

## 이후의 학습 방향

| 방향 | 이어서 학습할 내용 |
|---|---|
| 서버 운영 | 웹·데이터베이스 서버 구축, 백업과 복구 체계 |
| 자동화 | 셸 스크립트 심화, 구성 관리 도구 |
| 클라우드 | 컨테이너와 오케스트레이션 |
| **정보 보안** | 취약점 진단, 로그 분석, 침해 대응 |

리눅스마스터 2급을 준비하는 경우, **제3강(계정과 권한)** 의 복습과 본 종합문제의 반복 수행을 권장한다. 출제 비중이 가장 높을 뿐 아니라 이후 모든 학습의 기반이 되기 때문이다.
