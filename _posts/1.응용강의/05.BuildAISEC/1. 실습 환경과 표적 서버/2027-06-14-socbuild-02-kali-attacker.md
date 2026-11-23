---
title: "[AI 보안관제 구축] 1-2. 공격자 VM 만들기 — Kali Linux 설치·연결"
date: 2027-06-14 09:00:00 +0900
categories:
  - 1.응용강의
  - AI보안관제구축
tags:
  - 보안관제
  - Kali
  - VirtualBox
  - nmap
  - 공격자VM
pin: false
math: false
mermaid: false
---

> 지난 시간 도달 상태: VirtualBox 설치 + 내부 네트워크 이름 3개(`soc-wan`·`soc-lan`·`soc-dmz`) 약속 완료.
> 오늘의 목표: **공격자 역할(VM1 · Kali Linux)** 을 공식 이미지로 세우고 실습망에 연결한 뒤, 공격 도구가 준비됐는지 확인합니다.
> 오늘 켜는 VM: **VM1 한 대**.
{: .prompt-info }

## 0. 왜 Kali부터 만드나요

공격을 재현할 수 있어야 방어의 효과를 확인할 수 있습니다. **Kali Linux**는 nmap·nikto·sqlmap 같은 침투 테스트 도구가 처음부터 설치되어 있는 리눅스 배포판입니다. 우리는 Kali를 "공격자 자리"에 앉혀 두고, 5단계 내내 **똑같은 공격**을 반복하며 방어 장비의 효과를 비교합니다.

> Kali는 직접 OS를 설치하는 대신, 배포처가 제공하는 **VirtualBox용 사전 구성 이미지**를 등록하는 방식이 가장 빠릅니다. 보안 도구가 깔린 상태로 즉시 시작됩니다.
{: .prompt-tip }

---

## 1. 따라 하기: Kali 공식 이미지 내려받기

> ### 따라 하기 1-1. VirtualBox용 Kali 이미지 받기
>
> **목적** 설치 과정 없이 바로 쓸 수 있는 Kali 가상머신 파일을 준비합니다.
{: .prompt-tip }

**1단계.** 브라우저에서 Kali 공식 다운로드 페이지로 이동합니다.

```text
https://www.kali.org/get-kali/#kali-virtual-machines
```

**2단계.** 상단 탭에서 **Virtual Machines** 를 고르고, **VirtualBox** 아이콘 쪽의 **64-bit** 파일을 내려받습니다.

![](/assets/img/posts/2027-06-14-socbuild-02-kali-attacker-1787577506872.png)

![](/assets/img/posts/2027-06-14-socbuild-02-kali-attacker-1787577523683.png)


- 파일 확장자는 `.7z` 입니다. (압축 파일)
- 용량이 크므로(수 GB) 내려받는 데 시간이 걸립니다.

**3단계.** 받은 `.7z` 파일을 압축 해제합니다. 윈도우에 압축 프로그램이 없으면 **7-Zip**(무료, `https://www.7-zip.org`)을 설치해 풉니다.

![](/assets/img/posts/2027-06-14-socbuild-02-kali-attacker-1787579001385.png)

**4단계.** 압축을 풀면 폴더 안에 `.vbox` 파일과 `.vdi`(가상 디스크) 파일이 보입니다. 이 폴더를 앞으로 옮기지 않을 위치(예: `문서\VMs\kali`)에 두세요.

![](/assets/img/posts/2027-06-14-socbuild-02-kali-attacker-1787579158275.png)


> 📷 **화면 캡처 위치** — 압축 해제 후 폴더 안에 `.vbox`, `.vdi` 파일이 보이는 탐색기 화면.
{: .prompt-tip }

---

## 2. 따라 하기: VirtualBox에 Kali 등록하기

> ### 따라 하기 2-1. 기존 가상머신 추가(Add)
>
> **목적** 내려받은 이미지를 VirtualBox 관리자에 등록합니다. (새로 "만들기"가 아니라 "추가"입니다.)
{: .prompt-tip }

**1단계.** VirtualBox 관리자 상단 메뉴에서 **머신(Machine) → 추가(Add)** 를 고릅니다.

**2단계.** 앞서 압축을 푼 폴더 안의 **`.vbox` 파일**을 선택하고 **열기** 를 누릅니다.

**3단계.** 목록에 `kali-linux-...` VM이 나타나면 등록 완료입니다.

**4단계.** VM 이름을 알아보기 쉽게 바꿉니다. 목록에서 VM을 **오른쪽 클릭 → 설정(Settings) → 일반(General)** → 이름을 **`attacker`** 로 변경합니다.

> 📷 **화면 캡처 위치** — VirtualBox 관리자 왼쪽 목록에 `attacker`(Kali) VM이 등록된 화면.
{: .prompt-tip }

---

## 3. 따라 하기: 하드웨어·네트워크 설정

> ### 따라 하기 3-1. 메모리와 어댑터 2개 지정
>
> **목적** 기준표(1-1강)대로 사양과 네트워크를 맞춥니다.
{: .prompt-tip }

VM `attacker`를 선택하고 **설정(Settings)** 을 엽니다.

**1단계 — 메모리.** **시스템(System) → 마더보드** 에서 기본 메모리를 **`3072` MB**로 맞춥니다. (호스트가 8GB면 2048 MB로 낮춰도 됩니다.)

![](/assets/img/posts/2027-06-14-socbuild-02-kali-attacker-1787577341425.png)

**2단계 — 어댑터 1(NAT).** **네트워크(Network) → 어댑터 1** 탭에서:

| 항목 | 값 |
| --- | --- |
| 네트워크 어댑터 사용하기 | 체크 |
| 다음에 연결됨(Attached to) | **NAT** |

![](/assets/img/posts/2027-06-14-socbuild-02-kali-attacker-1787577359584.png)

이 어댑터는 Kali 도구를 업데이트할 때 쓰는 임시 인터넷입니다.

**3단계 — 어댑터 2(내부 네트워크).** **어댑터 2** 탭에서:

| 항목                   | 값                             |
| -------------------- | ----------------------------- |
| 네트워크 어댑터 사용하기        | 체크                            |
| 다음에 연결됨(Attached to) | **내부 네트워크(Internal Network)** |
| 이름(Name)             | **`soc-dmz`** (직접 입력)         |
![](/assets/img/posts/2027-06-14-socbuild-02-kali-attacker-1787577403573.png)


> `soc-dmz` 이름을 처음 입력하는 순간, 지난 시간에 약속한 노출망이 실제로 생성됩니다. 오늘은 방화벽이 없으므로 공격자를 임시로 이 노출망에 함께 둡니다.
{: .prompt-info }

**4단계.** **확인(OK)** 을 눌러 설정을 저장합니다.

> 📷 **화면 캡처 위치** — 어댑터 2가 "내부 네트워크 / soc-dmz"로 설정된 네트워크 설정 창.
{: .prompt-tip }

---

## 4. 따라 하기: Kali 첫 부팅과 로그인

> ### 따라 하기 4-1. 부팅·로그인
>
> **목적** VM을 처음 켜고 로그인합니다.
{: .prompt-tip }

**1단계.** `attacker` VM을 선택하고 **시작(Start)** 을 누릅니다.

**2단계.** 로그인 화면에서 기준표(1-1강)의 계정으로 로그인합니다.

| 항목 | 값 |
| --- | --- |
| 아이디 | `kali` |
| 비밀번호 | `kali` |
![](/assets/img/posts/2027-06-14-socbuild-02-kali-attacker-1787579527547.png)

**3단계.** 바탕화면이 뜨면, 위쪽 또는 옆쪽의 **검은 터미널 아이콘**을 눌러 터미널을 엽니다. 앞으로 모든 명령어는 이 터미널에 붙여 넣습니다.

> 이 강의의 명령어는 복사해서 터미널에 **붙여넣기(Ctrl+Shift+V)** 하면 됩니다. Kali 터미널에서는 보통 `Ctrl+V`가 아니라 **`Ctrl+Shift+V`** 로 붙여 넣습니다.
{: .prompt-tip }

---

## 5. 따라 하기: 실습망 고정 IP 지정

내부 네트워크에는 자동 IP가 없으므로, 기준표대로 **192.168.59.10**(1단계 임시)을 손으로 지정합니다. Kali는 NetworkManager를 쓰므로 `nmcli` 명령으로 간단히 설정합니다.

> ### 따라 하기 5-1. 인터페이스 이름 확인
>
> **목적** 내부 네트워크(어댑터 2)에 해당하는 랜카드 이름을 찾습니다.
{: .prompt-tip }

**1단계.** 터미널에 다음을 붙여 넣습니다.

```bash
ip -brief address
```

출력 예시는 다음과 같습니다.

```text
lo               UNKNOWN   127.0.0.1/8
eth0             UP        10.0.2.15/24        ← 어댑터1(NAT). 인터넷용
eth1             UP        169.254.x.x/16      ← 어댑터2(내부망). 여기에 고정IP를 줌
```

- `10.0.2.x`가 붙은 것이 **어댑터 1(NAT)** 입니다. 건드리지 않습니다.
- 나머지 하나(`eth1` 등)가 **어댑터 2(내부망)** 입니다. 이름을 확인해 둡니다. (환경에 따라 `eth1`이 아닐 수 있습니다.)

> ⚠️ 아래 명령의 `eth1` 부분은 **여러분 화면에서 확인한 실제 이름**으로 바꿔 넣으세요.
{: .prompt-warning }

> ### 따라 하기 5-2. 고정 IP 부여
>
> **목적** 어댑터 2에 `192.168.59.10/24`를 고정합니다.
{: .prompt-tip }

**2단계.** 다음 세 줄을 붙여 넣습니다. (게이트웨이·DNS는 1단계에는 방화벽이 없으므로 지정하지 않습니다. 인터넷은 어댑터1 NAT가 담당합니다.)

```bash
sudo nmcli connection add type ethernet ifname eth1 con-name soc-dmz ip4 192.168.59.10/24
sudo nmcli connection modify soc-dmz ipv4.method manual
sudo nmcli connection up soc-dmz
```

**3단계.** 제대로 붙었는지 확인합니다.

```bash
ip -brief address show eth1
```

`192.168.59.10/24`가 보이면 성공입니다.

```text
eth1             UP        192.168.59.10/24
```

![](/assets/img/posts/2027-06-14-socbuild-02-kali-attacker-1787579800401.png)

---

## 6. 따라 하기: 공격 도구 점검과 도구 최신화

> ### 따라 하기 6-1. 핵심 도구가 있는지 확인
>
> **목적** 앞으로 쓸 nmap·nikto·sqlmap이 준비됐는지 봅니다.
{: .prompt-tip }

**1단계.** 다음을 한 줄씩 붙여 넣어 버전이 출력되면 설치되어 있는 것입니다.

```bash
nmap --version
nikto -Version
sqlmap --version
```

![](/assets/img/posts/2027-06-14-socbuild-02-kali-attacker-1787579909352.png)

**2단계.** (선택) 도구 목록을 최신으로 맞춥니다. NAT 인터넷이 연결되어 있어야 합니다.

```bash
sudo apt update
```

> 만약 위 세 도구 중 없는 것이 있다면 다음으로 설치합니다: `sudo apt install -y nmap nikto sqlmap`
{: .prompt-tip }

---

## 7. 따라 하기: 스냅샷 저장

지금까지의 깨끗한 상태를 **스냅샷**으로 저장하면, 이후 실습에서 무언가 꼬여도 이 지점으로 즉시 되돌릴 수 있습니다.

> ### 따라 하기 7-1. 스냅샷 만들기
>
> **목적** "설치 직후 정상 상태"를 저장합니다.
{: .prompt-tip }

**1단계.** VM을 종료합니다. 터미널에서:

```bash
sudo poweroff
```

**2단계.** VirtualBox 관리자에서 `attacker` VM 선택 → 오른쪽 위 **≡(메뉴) → 스냅샷(Snapshots) → 만들기(Take)**.

**3단계.** 이름을 **`base-clean`** 으로 저장합니다.

> 📷 **화면 캡처 위치** — 스냅샷 목록에 `base-clean`이 저장된 화면.
{: .prompt-tip }

---

## 8. 오늘의 점검

- [ ] `attacker`(Kali) VM이 등록되고 부팅·로그인된다
- [ ] 어댑터 1 = NAT, 어댑터 2 = 내부 네트워크 `soc-dmz` 로 설정됐다
- [ ] `ip -brief address`에서 어댑터 2가 **192.168.59.10/24** 로 보인다
- [ ] `nmap --version`이 정상 출력된다
- [ ] 스냅샷 `base-clean` 을 저장했다

---

## 다음 시간

다음 시간에는 **표적 서버(VM4 · Ubuntu + Juice Shop)** 를 세워 같은 노출망에 연결하고, Kali에서 첫 공격을 날려 **방어 장치가 하나도 없을 때 공격이 얼마나 쉽게 통하는지**를 눈으로 확인합니다. 이 "무방비 상태"가 이후 모든 방어 단계의 비교 기준이 됩니다.
