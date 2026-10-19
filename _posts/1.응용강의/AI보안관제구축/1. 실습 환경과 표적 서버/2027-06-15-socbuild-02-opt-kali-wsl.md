---
title: "[AI 보안관제 구축] 1-2 부록(선택). WSL로 Kali Linux 설치하기 — VM 없이 윈도우 안에 공격 도구 갖추기"
date: 2027-06-14 18:00:00 +0900
categories:
  - 1.응용강의
  - AI보안관제구축
tags:
  - 보안관제
  - Kali
  - WSL
  - nmap
  - 선택부록
pin: false
math: false
mermaid: false
---

> **이 편은 선택(부록)입니다.** 건너뛰어도 1-3 이후 진행에 전혀 지장이 없습니다.
> 전제 상태: [1-1. 전체 설계도와 실습망 준비](/posts/socbuild-01-overview-network/) 를 끝낸 상태. (1-2의 Kali VM은 **있어도 되고 없어도 됩니다.**)
> 오늘의 목표: 윈도우 11 안에 **WSL(Windows Subsystem for Linux)** 로 Kali Linux를 설치하고, `nmap`·`nikto`·`sqlmap` 이 동작하는 것을 확인한 뒤, **같은 네트워크에 있는 표적 서버**를 공격해 봅니다.
{: .prompt-info }

> **★ 시작 전 확인 — WSL 2와 VirtualBox는 같은 컴퓨터에서 함께 쓸 수 있지만, VirtualBox가 느려질 수 있습니다.**
>
> WSL 2는 윈도우의 **Hyper-V 가상화 기능**을 켭니다. 이 상태에서 VirtualBox는 자체 엔진 대신 Hyper-V 위에서 도는 **호환 모드**로 전환되고, VM 창 오른쪽 아래에 **거북이 아이콘**이 보이면서 속도가 떨어질 수 있습니다. VirtualBox 7.1 이상에서는 많이 개선되었지만, 무거운 VM을 켰을 때 **눈에 띄게 느리면** 이 부록의 WSL을 끄거나(10장) 제거하는 것이 해결책입니다.
>
> 지금 설치된 VirtualBox 버전은 관리자 화면의 **도움말 → VirtualBox 정보** 에서 확인합니다. **7.1 미만이면 먼저 최신 버전으로 올린 뒤** 이 부록을 진행하세요.
{: .prompt-danger }

> ### 시작 전 점검 목록
>
> - [ ] **Windows 11** (또는 Windows 10 버전 2004 이상)이다
> - [ ] BIOS/UEFI에서 **가상화(VT-x / AMD-V)** 가 켜져 있다 ← 1-1에서 VirtualBox VM이 부팅됐다면 켜져 있는 것입니다
> - [ ] 관리자 권한으로 PowerShell을 열 수 있다
> - [ ] 인터넷이 연결되어 있다 (Kali 도구 내려받기 약 2 GB)
{: .prompt-warning }

---

## 1. 이론: WSL이란 무엇인가

**WSL(Windows Subsystem for Linux)** 은 윈도우 안에서 리눅스를 실행하는 마이크로소프트의 공식 기능입니다. 1-2에서 만든 Kali **VM**은 메모리 3 GB를 차지하고 화면 하나를 통째로 쓰지만, WSL은 리눅스를 **터미널 창 하나로** 띄우므로 훨씬 가볍습니다.

WSL에는 두 세대가 있습니다.

| | WSL 1 | **WSL 2** (우리가 쓰는 것) |
| --- | --- | --- |
| 원리 | 리눅스 명령을 윈도우 명령으로 **번역** | 아주 가벼운 **진짜 리눅스 커널**을 Hyper-V로 실행 |
| 네트워크 원시 소켓 | X (**nmap이 제대로 안 됨**) | O |
| Docker·systemd | X | O |
| 시작 속도 | 빠름 | 빠름 (1~2초) |

nmap 같은 공격 도구는 패킷을 직접 조립하는 **원시 소켓(raw socket)** 을 쓰기 때문에 **반드시 WSL 2** 여야 합니다. 요즘 윈도우 11에서는 기본값이 WSL 2이므로, 특별히 손댈 것은 없습니다.

```text
[ 윈도우 11 호스트 ]
 └─ WSL 2 ── kali-linux (터미널)
        │  (윈도우 네트워크를 통해 바깥과 통신)
        ▼
[ 같은 네트워크의 표적 서버 ]  ── Juice Shop (취약 웹앱)
```

> **비유**: WSL은 내 책상 위에 놓인 **작은 리눅스 상자**입니다. 윈도우가 연결된 네트워크를 그대로 빌려 쓰므로, **같은 네트워크에 있는 다른 컴퓨터**에는 그 컴퓨터의 IP로 바로 접근할 수 있습니다.
{: .prompt-tip }

---

## 2. 따라 하기: WSL 켜기

> ### 따라 하기 2-1. WSL 설치 (처음 쓰는 경우)
>
> **목적** 윈도우에 WSL 2 기능을 켜고 리눅스 커널을 내려받습니다.
{: .prompt-tip }

**1단계.** 시작 메뉴에서 `PowerShell` 을 검색하고, **오른쪽 클릭 → 관리자 권한으로 실행** 을 고릅니다.

**2단계.** 다음을 붙여 넣습니다. WSL이 이미 켜져 있으면 "이미 설치됨" 비슷한 메시지가 나오며, 그대로 4단계로 갑니다.

```powershell
wsl --install --no-distribution
```

- `--no-distribution` 은 "리눅스 배포판은 아직 받지 말고 WSL 기능만 켜라"는 뜻입니다. 이 옵션이 없으면 우분투가 자동으로 함께 설치됩니다. (이미 우분투를 쓰고 있어도 상관없습니다.)

**3단계.** 설치가 끝나면 **컴퓨터를 다시 시작**합니다. 재시작 뒤 관리자 PowerShell을 다시 엽니다.

**4단계.** WSL이 정상인지, 기본 버전이 2인지 확인합니다.

```powershell
wsl --version
wsl --set-default-version 2
```

첫 줄 출력에 `WSL 버전: 2.x.x` 와 `커널 버전: 6.x` 가 보이면 됩니다.



> **`wsl --version` 이 "잘못된 옵션"이라고 나온다면** WSL이 오래된 버전입니다. 다음으로 최신으로 올린 뒤 다시 확인하세요.
>
> ```powershell
> wsl --update
> ```
{: .prompt-tip }

---

## 3. 따라 하기: Kali Linux 배포판 설치

> ### 따라 하기 3-1. 설치 가능한 배포판 목록에서 Kali 확인
>
> **목적** WSL이 제공하는 배포판 중 Kali의 정확한 이름을 확인합니다.
{: .prompt-tip }

**1단계.** PowerShell에 붙여 넣습니다.

```powershell
wsl --list --online
```

목록에 다음 줄이 있어야 합니다. 이름은 **`kali-linux`** (소문자, 붙임표) 입니다.

```text
kali-linux                      Kali Linux Rolling
```

> ### 따라 하기 3-2. Kali 설치와 계정 만들기
>
> **목적** Kali를 내려받아 등록하고, 실습 기준표의 계정을 만듭니다.
{: .prompt-tip }

**2단계.** 설치합니다. 몇백 MB를 내려받으므로 잠시 기다립니다.

```powershell
wsl --install -d kali-linux
```

**3단계.** 내려받기가 끝나면 **Kali 창이 자동으로 열리며** 사용자 이름과 비밀번호를 묻습니다. 기준표(1-1강 3.3)의 값을 그대로 씁니다.

| 항목 | 값 |
| --- | --- |
| Enter new UNIX username | `kali` |
| New password | `kali` |
| Retype new password | `kali` |

- 비밀번호를 칠 때 화면에 **아무것도 표시되지 않는 것이 정상**입니다. 그대로 치고 Enter를 누르세요.
- 계정이 만들어지면 `┌──(kali㉿호스트이름)-[~]` 모양의 Kali 프롬프트가 뜹니다. 이 창이 앞으로의 **Kali 터미널**입니다.

![](/assets/img/posts/2027-06-15-socbuild-02-opt-kali-wsl-1789029752231.png)

**4단계.** (창이 자동으로 안 열렸거나, 나중에 다시 열 때) 시작 메뉴에서 **`Kali Linux`** 를 실행하거나, 아무 PowerShell에서 다음을 칩니다.

```powershell
wsl -d kali-linux
```

> **Kali가 기본 배포판이 아니라면** `wsl` 만 쳤을 때 우분투가 열릴 수 있습니다. 기본을 Kali로 바꾸려면 `wsl --set-default kali-linux` 를 한 번 실행합니다. (이 강의는 항상 `-d kali-linux` 를 붙여 헷갈리지 않게 합니다.)
{: .prompt-tip }

---

## 4. 따라 하기: 공격 도구 설치

WSL용 Kali는 **도구가 거의 없는 최소 상태**로 설치됩니다. VM 이미지와 달리 `nmap` 조차 없습니다. 필요한 도구를 직접 설치합니다.

> ### 따라 하기 4-1. 패키지 목록 갱신과 핵심 도구 3종 설치
>
> **목적** 이 과정에서 쓰는 nmap·nikto·sqlmap과 curl을 설치합니다.
{: .prompt-tip }

**1단계.** Kali 터미널에 붙여 넣습니다. 붙여넣기는 창 안에서 **오른쪽 클릭** 또는 `Ctrl+Shift+V` 입니다.

```bash
sudo apt update
```

`[sudo] password for kali:` 가 나오면 `kali` 를 입력합니다. (역시 화면에 표시되지 않습니다.)

**2단계.** 핵심 도구를 설치합니다. 약 200 MB, 1~3분 걸립니다.

```bash
sudo apt install -y nmap nikto sqlmap curl
```

**3단계.** 1-2강 6장과 같은 방법으로 확인합니다. 세 줄 모두 버전이 출력되면 성공입니다.

```bash
nmap --version
nikto -Version
sqlmap --version
```

> 📷 **화면 캡처 위치** — 세 도구의 버전이 모두 출력된 Kali 터미널.
{: .prompt-tip }

> ### 따라 하기 4-2. (선택) Kali 전체 도구 묶음 설치
>
> **목적** VM 이미지와 같은 수준의 도구 세트를 갖춥니다. **약 2 GB, 10~20분** 걸리므로 필요할 때만 하세요.
{: .prompt-tip }

```bash
sudo apt install -y kali-linux-headless
```

- `kali-linux-headless` 는 화면(GUI) 없이 터미널에서 쓰는 도구만 모은 묶음입니다. 이 과정에서 쓰는 도구는 4-1의 세 개로 충분하므로, 건너뛰어도 됩니다.
- 설치 중 화면 설정을 묻는 보라색 창(`kismet`·`wireshark` 등)이 나오면 **기본값(Enter 또는 No)** 으로 넘깁니다.

---

## 5. 따라 하기: 도구 동작 확인 — 자기 자신 스캔

표적 서버에 손대기 전에, 도구가 제대로 도는지 **자기 자신**을 스캔해 확인합니다. 가장 안전한 확인 방법입니다.

> ### 따라 하기 5-1. 로컬 스캔
>
> **목적** nmap이 WSL 2에서 원시 소켓으로 정상 동작하는지 확인합니다.
{: .prompt-tip }

**1단계.** 기본 스캔:

```bash
nmap 127.0.0.1
```

`Nmap done: 1 IP address (1 host up)` 으로 끝나면 됩니다. 열린 포트가 하나도 없어도 정상입니다.

**2단계.** 관리자 권한이 필요한 **SYN 스캔**(`-sS`)이 되는지 봅니다. 이 스캔이 원시 소켓을 씁니다.

```bash
sudo nmap -sS 127.0.0.1
```

같은 결과가 나오면 **WSL 2에서 원시 소켓이 살아 있는 것**입니다. 1-2 Kali VM과 같은 방식으로 스캔할 수 있습니다.

> **`-sS` 만 오류가 나거나 결과가 이상하면** WSL 1로 설치된 것입니다. 관리자 PowerShell에서 다음으로 WSL 2로 바꿉니다. (몇 분 걸립니다.)
>
> ```powershell
> wsl --set-version kali-linux 2
> ```
{: .prompt-warning }

---

## 6. 따라 하기: (선택) 같은 네트워크의 표적 서버 공격

이 장은 **표적 서버(웹앱 Juice Shop)가 준비되고, 그 서버가 이 PC와 같은 네트워크에 있을 때** 합니다. 아직이라면 [1-3](/posts/socbuild-03-victim-server-firstattack/) 을 먼저 마친 뒤 돌아오세요.

WSL Kali는 윈도우가 연결된 네트워크를 그대로 빌려 씁니다. 따라서 **같은 네트워크에 있는 표적 서버에는 그 서버의 IP로 바로** 접근할 수 있습니다. 1-3에서 Kali VM으로 했던 첫 공격을, 여기서는 WSL 터미널에서 그대로 재현합니다.

> 아래에서 `<표적서버IP>` 는 여러분의 표적 서버가 받은 실제 IP로 바꿔 넣으세요. 표적 서버 콘솔에서 `ip -brief address` 로 확인할 수 있습니다.
{: .prompt-warning }

> ### 따라 하기 6-1. 표적이 살아 있는지 확인 (ping)
>
> **목적** WSL Kali에서 표적 서버까지 길이 닿는지 봅니다.
{: .prompt-tip }

**1단계.** Kali 터미널에서:

```bash
ping -c 3 <표적서버IP>
```

응답이 오면 연결된 것입니다.

> ### 따라 하기 6-2. 포트 스캔 (nmap)
>
> **목적** 표적에 어떤 서비스가 열려 있는지 정찰합니다.
{: .prompt-tip }

**2단계.**

```bash
nmap -sV -T4 <표적서버IP>
```

**22/tcp (ssh)** 와 **3000/tcp** 가 `open` 으로 보이면, 방어 장치가 없는 표적의 열린 문이 그대로 드러난 것입니다.

> ### 따라 하기 6-3. 웹 취약점 스캔과 SQL 인젝션
>
> **목적** 대표적인 웹 공격이 실제로 통하는 것을 확인합니다.
{: .prompt-tip }

**3단계.** 응답이 오는지 먼저 봅니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://<표적서버IP>:3000
```

`200` 이면 웹앱이 살아 있는 것입니다.

**4단계.** 웹 취약점 스캐너를 돌립니다.

```bash
nikto -h http://<표적서버IP>:3000
```

**5단계.** 로그인 우회(SQL 인젝션)는 화면이 필요하므로 **윈도우 브라우저**로 `http://<표적서버IP>:3000` 에 접속해, 1-3강 5-4와 똑같이 로그인 칸에 `' OR 1=1;--` 를 넣어 확인합니다. (WSL Kali에는 기본적으로 그래픽 브라우저가 없습니다.)

> 방어 장치가 하나도 없어 스캔·웹 접근·SQL 인젝션이 모두 성공합니다. 이 **무방비 상태**가 이후 방어 단계의 "before" 기준입니다. WSL Kali로 하든 Kali VM으로 하든 결과는 같습니다.
{: .prompt-danger }

---

## 7. 알아 두기: 윈도우 ↔ WSL 파일 오가기

공격 결과를 윈도우에서 보고서로 정리할 때 유용합니다.

| 하고 싶은 일 | 방법 |
| --- | --- |
| WSL에서 윈도우 폴더 보기 | 윈도우 `C:\` 는 WSL 안에서 `/mnt/c/` 입니다. 예: `ls /mnt/c/Users` |
| 윈도우 탐색기에서 WSL 폴더 보기 | 탐색기 주소창에 `\\wsl$\kali-linux\home\kali` |
| 스캔 결과를 윈도우 바탕화면에 저장 | `nmap <표적서버IP> -oN /mnt/c/Users/<윈도우사용자>/Desktop/scan.txt` |
| Kali 터미널을 새 창으로 | 윈도우 터미널(Windows Terminal) 탭의 **∨ → Kali Linux** |

> `/mnt/c/` 아래의 윈도우 파일은 리눅스 쪽에서 읽고 쓰는 속도가 느립니다. 큰 작업은 WSL 안(`~`)에서 하고, 결과만 윈도우로 옮기는 편이 낫습니다.
{: .prompt-tip }

---

## 8. 스냅샷 대신 — WSL 백업(내보내기)

VM의 스냅샷(1-2강 7장)에 해당하는 기능이 WSL에는 없습니다. 대신 **전체를 파일 하나로 내보내기** 할 수 있습니다. 도구 설치가 끝난 지금 상태를 저장해 두면, 나중에 꼬였을 때 되돌릴 수 있습니다.

> ### 따라 하기 8-1. 내보내기·되돌리기
>
> **목적** 지금의 깨끗한 Kali 상태를 파일로 보관합니다.
{: .prompt-tip }

**1단계.** (PowerShell) 먼저 Kali를 멈추고 내보냅니다. 파일 크기는 도구 설치량에 따라 1~4 GB입니다.

```powershell
wsl --terminate kali-linux
wsl --export kali-linux C:\VMs\kali-wsl-base.tar
```

폴더 `C:\VMs` 가 없으면 먼저 만듭니다.

**2단계.** (되돌려야 할 때) 지금 것을 지우고 파일에서 다시 만듭니다. **`--unregister` 는 되돌릴 수 없으므로** 1단계의 파일이 있는지 먼저 확인하세요.

```powershell
wsl --unregister kali-linux
wsl --import kali-linux C:\VMs\kali-wsl C:\VMs\kali-wsl-base.tar
```

- 되돌린 뒤에는 기본 로그인이 `root` 가 되기도 합니다. 그때는 `wsl -d kali-linux -u kali` 로 열면 됩니다.

---

## 9. 끄기·제거하기

WSL Kali는 창을 닫으면 몇 초 뒤 알아서 멈춥니다. 다만 VirtualBox VM이 느려진다고 느껴질 때는 **WSL 전체를 명시적으로 멈추는 것**이 도움이 됩니다.

| 하고 싶은 일 | PowerShell 명령 |
| --- | --- |
| WSL 전체 멈추기 (메모리 즉시 반환) | `wsl --shutdown` |
| Kali만 멈추기 | `wsl --terminate kali-linux` |
| 상태 보기 | `wsl --list --verbose` |
| Kali 완전 삭제 | `wsl --unregister kali-linux` (8장의 백업 파일이 있으면 언제든 복구 가능) |

> WSL이 VirtualBox 속도에 주는 영향은 **WSL 기능 자체(Hyper-V)** 가 켜져 있는 데서 옵니다. Kali를 멈추거나 지워도 그 영향은 남습니다. VirtualBox 속도가 도저히 안 나오면, 관리자 PowerShell에서 `Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform` 으로 기능을 끄고 재시작하면 원래대로 돌아옵니다. (이후 WSL은 쓸 수 없습니다.)
{: .prompt-tip }

---

## 10. 오늘의 점검

- [ ] `wsl --version` 에서 WSL 2와 리눅스 커널 버전이 보인다
- [ ] `wsl -d kali-linux` 로 Kali 프롬프트가 열리고, 계정은 `kali/kali` 다
- [ ] `nmap --version` · `nikto -Version` · `sqlmap --version` 이 모두 출력된다
- [ ] `sudo nmap -sS 127.0.0.1` 이 오류 없이 끝난다 (= WSL 2 원시 소켓 정상)
- [ ] (표적 준비 시) `nmap -sV <표적서버IP>` 로 3000 포트가 `open` 으로 보인다
- [ ] `C:\VMs\kali-wsl-base.tar` 백업을 만들었다

---

## 다음

부록은 여기까지입니다. 본 과정으로 돌아가 **[1-3. 표적 서버 만들기와 첫 공격](/posts/socbuild-03-victim-server-firstattack/)** 으로 진행하세요.
