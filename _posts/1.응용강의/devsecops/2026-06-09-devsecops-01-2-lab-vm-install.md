---
title: "[안전한 자동화 인프라] 01-2. 실습 — VirtualBox에 Ubuntu VM 3대 만들기"
date: 2026-06-09 09:04:00 +0900
categories:
  - 1.응용강의
  - 안전한자동화인프라
  - 실습환경
tags:
  - VirtualBox
  - Ubuntu
  - VM복제
  - 고정IP
  - netplan
math: false
mermaid: true
---

> 이번 글은 **직접 따라 하는 실습**입니다. VirtualBox를 설치하고 Ubuntu Server VM 1대를 만든 뒤, **복제로 3대(build·dev·prod)** 를 구성합니다. 모든 명령은 복사-붙여넣기로 따라 할 수 있습니다.
{: .prompt-info }

## 0. 이번 실습의 목표

```mermaid
flowchart LR
    A["VirtualBox 설치"] --> B["Ubuntu VM 1대 설치"]
    B --> C["복제로 3대 만들기"]
    C --> D["어댑터 2개 + 고정 IP"]
```

| VM | 내부 IP | 메모리 권장 |
|---|---|---|
| **build** | 192.168.56.10 | 2048 MB |
| **dev** | 192.168.56.20 | 1024 MB |
| **prod** | 192.168.56.30 | 1024 MB |

---

## 1. VirtualBox 설치 (호스트 = 윈도우)

1. 브라우저에서 **virtualbox.org → Downloads** 로 이동
2. **Windows hosts** 설치 파일을 내려받아 실행 → 기본값으로 "Next" 진행 → 설치 완료
3. 함께 제공되는 **Oracle VM VirtualBox Extension Pack** 도 받아 설치(선택, USB·확장 기능용)

> 설치 중 네트워크가 잠깐 끊길 수 있다는 경고가 나오면 그대로 진행해도 됩니다.
{: .prompt-tip }

---

## 2. Ubuntu Server 설치 이미지(ISO) 내려받기

1. **ubuntu.com → Download → Ubuntu Server** 로 이동
2. **Ubuntu Server 24.04 LTS** ISO 파일을 내려받습니다. (LTS = 장기지원판)

> ISO는 "설치 CD를 파일 하나로 만든 것"입니다. VM에 이 CD를 넣고 부팅해 설치합니다.
{: .prompt-info }

---

## 3. 호스트 전용 네트워크 먼저 만들기

VM을 만들기 전에, VM들이 내부에서 통신할 **호스트 전용 네트워크**를 준비합니다.

1. VirtualBox 메인 화면 상단 메뉴 → **도구(Tools) → 네트워크(Network)**
2. **호스트 전용 네트워크(Host-only Networks)** 탭 → **만들기(Create)**
3. 생성된 어댑터(보통 `VirtualBox Host-Only Ethernet Adapter`)를 선택 → **속성**에서 아래 확인/설정
   - IPv4 주소: `192.168.56.1`
   - 넷마스크: `255.255.255.0`
   - **DHCP 서버: 사용 안 함(끄기)** — 우리는 고정 IP를 직접 줄 것이므로 끕니다.

> 이 네트워크의 이름(예: `vboxnet0` 또는 어댑터 이름)을 기억해 두세요. VM마다 이 네트워크에 연결합니다.
{: .prompt-tip }

---

## 4. 첫 VM 만들기 (이 한 대를 "원본"으로 복제할 것)

### 4.1 새 VM 생성

1. VirtualBox → **새로 만들기(New)**
2. 이름: `ubuntu-base` / 종류: Linux / 버전: Ubuntu (64-bit)
3. ISO 선택: 방금 받은 Ubuntu Server ISO 지정
4. 메모리: **2048 MB**, CPU: **2개**
5. 디스크: **15 GB**(동적 할당)로 생성 → 완료

### 4.2 어댑터 2개 설정 (부팅 전에)

생성된 `ubuntu-base` 선택 → **설정(Settings) → 네트워크(Network)**

| 어댑터 | 설정 |
|---|---|
| **어댑터 1** | "다음에 연결됨" = **NAT** |
| **어댑터 2** | "어댑터 2 사용하기" 체크 → "다음에 연결됨" = **호스트 전용 어댑터** → 3장에서 만든 네트워크 선택 |

### 4.3 Ubuntu 설치 진행

VM을 **시작(Start)** 하면 Ubuntu 설치 화면이 뜹니다. 키보드로 진행합니다.

1. 언어: English (또는 한국어)
2. 키보드: 기본값
3. 네트워크: 화면에 어댑터 2개가 보입니다. 그대로 진행(IP는 나중에 직접 설정)
4. 저장소: 디스크 전체 사용 기본값
5. **Profile setup** (중요 — 모든 VM 공통 계정):
   - 이름: `devops`
   - 서버 이름(hostname): 일단 `ubuntu-base`
   - 사용자명: **`devops`**
   - 비밀번호: **`devops123`** (실습용 — 실제 서버에선 강한 비밀번호 사용)
6. **Install OpenSSH server** 항목을 **반드시 체크** (원격 접속에 필요)
7. 설치 완료 후 **재부팅** → 로그인 프롬프트가 나오면 성공

> 공통 계정 **`devops` / `devops123`** 와 **OpenSSH 설치**는 이후 모든 자동화의 토대입니다. 꼭 동일하게 설정하세요.
{: .prompt-warning }

---

## 5. 원본 VM을 3대로 복제

이제 `ubuntu-base`를 끄고(`sudo poweroff`), 이걸 복제해 3대를 만듭니다.

각 VM에 대해 VirtualBox에서:

1. `ubuntu-base` 우클릭 → **복제(Clone)**
2. 이름: 차례로 `build`, `dev`, `prod`
3. **"모든 네트워크 카드의 MAC 주소 새로 생성"** 선택 (중요 — 안 하면 IP 충돌)
4. **완전한 복제(Full Clone)** 선택 → 복제

세 번 반복해 **build · dev · prod** 3대를 만듭니다. 메모리는 dev·prod를 1024 MB로 낮춰도 됩니다(설정 → 시스템).

> 원본 `ubuntu-base`는 끈 채로 보관하세요. 나중에 VM이 필요하면 또 복제하면 됩니다.
{: .prompt-tip }

---

## 6. 각 VM에 고정 IP 주기 (netplan)

복제한 3대를 각각 켜고 로그인(`devops`/`devops123`)한 뒤, **호스트 전용 어댑터(어댑터 2)** 에 고정 IP를 줍니다.

### 6.1 어댑터 이름 확인

```bash
ip a
```

출력에서 어댑터가 보통 `enp0s3`(NAT), `enp0s8`(호스트 전용) 두 개로 나옵니다. **두 번째(enp0s8)** 에 고정 IP를 줍니다. 이름이 다르면 본인 화면 기준으로 바꾸세요.

### 6.2 netplan 설정 파일 수정

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

아래처럼 작성합니다. **`build`는 192.168.56.10**, dev는 `.20`, prod는 `.30` 으로 VM마다 주소만 바꿉니다.

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.10/24
```

저장(Ctrl+O, Enter) 후 종료(Ctrl+X). 권한 경고를 막기 위해 파일 권한을 조정하고 적용합니다.

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

### 6.3 hostname 바꾸기 (헷갈림 방지)

각 VM의 이름을 역할에 맞게 바꿉니다.

```bash
# build VM에서
sudo hostnamectl set-hostname build
# dev VM에서는 dev, prod VM에서는 prod
```

다시 로그인하면 프롬프트가 `devops@build:~$` 처럼 바뀝니다.

### 6.4 IP 확인

```bash
ip a show enp0s8
```

`inet 192.168.56.10/24` (build 기준)가 보이면 성공입니다. dev·prod도 같은 방식으로 `.20`, `.30` 확인.

---

## 7. 체크포인트

세 VM에서 각각 아래가 맞는지 확인하세요.

| 항목 | 확인 명령 | 기대 결과 |
|---|---|---|
| 호스트명 | `hostname` | build / dev / prod |
| 고정 IP | `ip a show enp0s8` | .10 / .20 / .30 |
| 인터넷 | `ping -c2 8.8.8.8` | 응답 옴 |
| SSH 데몬 | `systemctl is-active ssh` | active |

> 인터넷(ping 8.8.8.8)이 안 되면 어댑터 1(NAT) 설정을, 내부 IP가 없으면 netplan(어댑터 2)을 다시 확인하세요.
{: .prompt-warning }

다음 글 **01-3** 에서 VM 3대가 서로 통신되는지(ping·SSH) 확인하고, 스냅샷을 찍어 안전벨트를 만듭니다.
