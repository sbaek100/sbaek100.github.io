---
title: "보안 실험을 위한 VM 환경 구축(칼리리눅스: 공격자)"
date: 2026-03-06 08:00:00 +0900
categories:
  - 강의
  - 보안시스템구축실습
tags:
  - 칼리리눅스
  - 설치
  - Virtualbox
pin: false
---
---

# 보안 실험을 위한 Kali Linux VM 환경 구축

> **완성 목표:** 한글 입력 가능 + NAT 네트워크(192.168.0.10) + Ubuntu(192.168.0.30) ping 통신

---

## 전체 흐름

```
[1단계] Kali Linux VM 이미지 다운로드 (VirtualBox용 .7z)
    ↓
[2단계] VirtualBox 설치 + VM 이미지 임포트(Import)
    ↓
[3단계] root 비밀번호 설정
    ↓
[4단계] apt update & upgrade + 시간 동기화
    ↓
[5단계] 한글 입력 환경 설정 (fcitx5-hangul)
    ↓
[6단계] NAT 네트워크 설정 + 고정 IP (192.168.0.10)
    ↓
[7단계] Ubuntu(192.168.0.30) ping 테스트
    ↓
[8단계] 스냅샷(Snapshot) 저장
```

> 💡 **ISO 설치 방식 vs VM 이미지 방식 차이**  
> ISO 방식은 OS를 직접 설치해야 하고 20~40분 소요됨.  
> VM 이미지 방식은 압축 해제 + 임포트만 하면 되므로 **5분 이내** 완료 가능.

---

## 1단계 — Kali Linux VM 이미지 다운로드

### 1-1. 공식 사이트 접속

- URL: [https://www.kali.org/get-kali/](https://www.kali.org/get-kali/)
- 화면 상단에서 **"Virtual Machines"** 카드 클릭 (Recommended 표시된 항목)
![](/assets/img/posts/2026-03-06-server_installation%202-1772767598239.png)

![](/assets/img/posts/2026-03-06-server_installation%202-1772767630605.png)


### 1-2. VirtualBox 용 이미지 선택

|항목|선택 기준|
|---|---|
|플랫폼|**VirtualBox** (VMware 아님)|
|아키텍처|**64-bit (amd64)**|
|파일 형식|`.7z` (압축 파일)|
|파일 크기|약 3~4 GB|

> 💡 `.7z` 는 7-Zip 압축 형식. 다운로드 후 반드시 압축 해제 필요.

### 1-3. 7-Zip 설치 (압축 해제 도구)

- 7-Zip 미설치 시: [https://www.7-zip.org/](https://www.7-zip.org/) 에서 설치
- Windows 64-bit 기준: `7z2301-x64.exe` 다운로드 후 설치

### 1-4. 압축 해제

1. 다운로드된 `.7z` 파일 우클릭
2. **7-Zip** → **여기에 압축 풀기** 클릭
3. 압축 해제 완료 후 폴더 내 파일 확인

압축 해제 후 생성되는 파일:

|파일명|설명|
|---|---|
|`kali-linux-xxxx-virtualbox-amd64.vbox`|VirtualBox VM 설정 파일|
|`kali-linux-xxxx-virtualbox-amd64.vdi`|가상 디스크 이미지|

> ⚠️ **두 파일이 같은 폴더에 있어야** 정상 동작함. 파일을 분리하지 말 것.

---

## 2단계 — VirtualBox 설치 + VM 이미지 임포트

### 2-1. VirtualBox 설치 (미설치 시)

- 공식 사이트: [https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads)
- **Windows hosts** 선택 후 다운로드
- 설치는 기본값(Next → Next → Install)으로 진행

### 2-2. VM 임포트 방법 (대안: VirtualBox 메뉴 이용)

1. VirtualBox 실행 → 상단 메뉴 **[머신]** → **[OPEN]**
   (압출을 푼 폴더의 vbox을 클릭한다.)
   ![](/assets/img/posts/2026-03-06-server_installation%202-1772768212095.png)
       
2. 1단계 압축 해제 폴더에서 `.vbox` 파일 선택   
3. **[열기]** 클릭 → VM 목록에 등록 확인   

### 2-3. VM 하드웨어 설정 조정

임포트된 VM 선택 → **[설정]** → 아래 항목 수정

**시스템 > 마더보드**

|항목|권장값|최솟값|
|---|---|---|
|메모리(RAM)|**4096 MB**|2048 MB|

![](/assets/img/posts/2026-03-06-server_installation%202-1772768269892.png)


**시스템 > 프로세서**

| 항목  | 권장값    |
| --- | ------ |
| CPU | **2개** |
![](/assets/img/posts/2026-03-06-server_installation%202-1772768280356.png)


**디스플레이**

|항목|값|
|---|---|
|비디오 메모리|**128 MB**|
|그래픽 컨트롤러|`VMSVGA`|

### 2-4. VM 최초 실행

1. VM 선택 → **[시작]** 클릭
2. 로그인 화면이 나타나면 임포트 성공

**기본 로그인 정보** (Kali 공식 VM 이미지 기준):

|항목|값|
|---|---|
|사용자명|`kali`|
|비밀번호|`kali`|

> ⚠️ 기본 비밀번호는 보안상 매우 취약함 — 반드시 다음 단계에서 변경

![](/assets/img/posts/2026-03-06-server_installation%202-1772768361333.png)

---

## 3단계 — root 비밀번호 설정

> VM 이미지의 기본 비밀번호(`kali`/`kali`)는 보안상 위험. 반드시 변경해야 함.

### 3-1. 터미널 열기

- 로그인 후 화면 좌측 하단 터미널 아이콘 클릭
- 또는 우클릭 → **Open Terminal Here**

### 3-2. kali 사용자 비밀번호 변경

```bash
passwd
```

```
Current password: kali          ← 현재 비밀번호 입력
New password:                   ← 새 비밀번호 입력
Retype new password:            ← 재입력
passwd: password updated successfully
```

### 3-3. root 비밀번호 설정

```bash
sudo passwd root
```

```
New password:          ← root 에 설정할 비밀번호 입력
Retype new password:   ← 재입력
passwd: password updated successfully
```

![](/assets/img/posts/2026-03-06-server_installation%202-1772768427268.png)

### 3-4. root 계정으로 전환 확인

```bash
su -
# root 비밀번호 입력
```

프롬프트가 `kali@kali:~$` → `root@kali:~#` 으로 바뀌면 성공

![](/assets/img/posts/2026-03-06-server_installation%202-1772768485467.png)

> 💡 이후 실습은 **root 계정**으로 진행 권장 (보안 도구 실행 시 권한 문제 방지)

---

## 4단계 — apt update & upgrade + 시간 동기화

> 터미널(Terminal)을 열고 진행 (root 계정으로 실행 권장)

### 4-1. 터미널 열기

- 화면 좌측 상단 메뉴 또는 우클릭 → **Terminal Emulator**

### 4-2. root 전환

```bash
su -
# root 비밀번호 입력
```

### 4-3. 설치된 패키지 업그레이드

```bash
apt upgrade -y
```

> `-y` 옵션: 중간에 "계속하시겠습니까?" 질문에 자동으로 **yes** 응답  
> ⏱ 소요 시간: 네트워크 속도에 따라 5~20분

### 4-4. 불필요한 패키지 정리

```bash
apt autoremove -y
apt autoclean
```

### 4-5. 시간 동기화 설정

#### 현재 시간 확인

```bash
date
timedatectl status
```

#### 타임존 설정 (한국 시간)

```bash
timedatectl set-timezone Asia/Seoul
```

#### NTP(Network Time Protocol) 동기화 활성화

```bash
timedatectl set-ntp true
```

#### 동기화 확인

```bash
timedatectl status
```

출력 예시:

![](/assets/img/posts/2026-03-06-server_installation%202-1772770471850.png)

---

## 5단계 — 한글 입력 환경 설정 (fcitx5-hangul)

### 5-1. 한글 입력기 설치

```bash
apt install -y fcitx5 fcitx5-hangul fcitx5-config-qt fonts-nanum -y
```

### 5-2. 환경변수 등록

```bash
nano /etc/environment
```

아래 내용 추가 (파일 끝에 붙여넣기):

```
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
```

저장: `Ctrl + O` → `Enter` → `Ctrl + X`

### 5-3. 자동 시작 등록

```bash
# 1. autostart 디렉터리를 생성 (이미 있다면 무시됨)
mkdir -p ~/.config/autostart/

# 2. 파일 다시 복사
cp /usr/share/applications/org.fcitx.Fcitx5.desktop ~/.config/autostart/
```

### 5-4. 재부팅

```bash
reboot
```

- **시스템 종료:** `poweroff` 또는 `shutdown -h now`    
- **시스템 재부팅:** `reboot`
### 5-5. fcitx5 설정 (재부팅 후)

1. 우측 하단 작업표시줄에 **키보드 아이콘** 확인
2. 아이콘 우클릭 → **Configure** 클릭
3. **Input Method** 탭 → `+` 클릭
4. 검색창에 `hangul` 입력 → **Hangul** 선택 → **OK**

### 5-6. 한/영 전환 키 설정

- fcitx5 설정 → **Global Options** → **Toggle Input Method**
- `한/영` 키 또는 `Shift + Space` 로 지정

![](/assets/img/posts/2026-03-06-server_installation%202-1772770835300.png)

### 5-7. 한글 입력 테스트

- 텍스트 에디터 열기 → `한/영` 키 눌러서 한글 입력 확인

![](/assets/img/posts/2026-03-06-server_installation%202-1772770941811.png)

---

## 6단계 — NAT 네트워크 설정 + 고정 IP (192.168.0.10)

### 6-1. VirtualBox NAT 네트워크 생성

VirtualBox 메인 화면에서:

1. 상단 메뉴 **[파일]** → **[도구]** → **[네트워크 관리자]**
2. **NAT 네트워크** 탭 → **[만들기]** 클릭
3. 아래와 같이 설정

|항목|값|
|---|---|
|이름|`NatNetwork`|
|IPv4 접두사|`192.168.0.0/24`|
|DHCP 활성화|**체크 해제** (고정 IP 사용)|

4. **[적용]** 클릭

### 6-2. Kali VM 네트워크 어댑터 변경

1. Kali VM 선택 → **[설정]** → **[네트워크]**
2. 어댑터 1 설정

|항목|값|
|---|---|
|네트워크 어댑터 활성화|**체크**|
|다음에 연결됨|**NAT 네트워크**|
|이름|`NatNetwork`|

3. **[확인]** 클릭

### 6-3. Kali 내부에서 고정 IP 설정

Kali 부팅 후 터미널에서:

#### 네트워크 인터페이스 이름 확인

```bash
ip a
```

출력 예시에서 `eth0` 또는 `enp0s3` 형태의 인터페이스 이름 확인

#### 네트워크 설정 파일 편집

1. `Advanced Network Configuration` 을 클릭
     
 ![](/assets/img/posts/2026-03-06-server_installation%202-1772771474584.png)

2.  ### 이더넷 연결 선택

- 목록에서 현재 사용 중인 연결(보통 **Wired connection 1** 또는 **eth0**)을 선택한 후, 하단의 **톱니바퀴 아이콘(Edit)**을 클릭합니다.
    

3.  IPv4 설정 변경

- 상단 탭에서 **IPv4 Settings**로 이동합니다.    
- **Method** 항목이 기본값인 `Automatic (DHCP)`로 되어 있을 텐데, 이를 **`Manual`**로 변경합니다.
    

4. IP 주소 입력

- **Addresses** 섹션 우측의 **Add** 버튼을 누르고 다음 값들을 입력합니다.
    
    - **Address**: `192.168.0.10`        
    - **Netmask**: `24` (또는 `255.255.255.0`)        
    - **Gateway**: `192.168.0.1` (일반적인 공유기/방화벽 주소)

#### 네트워크 재시작

```bash
systemctl restart NetworkManager
```

#### IP 적용 확인

```bash
ip a
```

출력에서 `192.168.0.10` 확인:

```
2: eth0: ...
    inet 192.168.0.10/24 brd 192.168.0.255 scope global eth0
```


![](/assets/img/posts/2026-03-06-server_installation%202-1772771667248.png)


---

## 7단계 — Ubuntu(192.168.0.30) ping 테스트

> Ubuntu VM 도 동일한 NAT 네트워크(`NatNetwork`)에 연결되어 있어야 함

### 7-1. Ubuntu VM 네트워크 확인

Ubuntu VM 에서 터미널 열고:

```bash
ip a
# 192.168.0.30 이 설정되어 있는지 확인
```

### 7-2. 두 VM 동시 실행

- VirtualBox 에서 Kali VM, Ubuntu VM **둘 다 시작**

### 7-3. Kali → Ubuntu ping 테스트

Kali 터미널에서:

```bash
ping -c 4 192.168.0.30
```

성공 시 출력 예시:

![](/assets/img/posts/2026-03-06-server_installation%202-1772771704808.png)


### 7-4. Ubuntu → Kali ping 테스트 (역방향)

Ubuntu 터미널에서:

```bash
ping -c 4 192.168.0.10
```

![](/assets/img/posts/2026-03-06-server_installation%202-1772771727130.png)

### 7-5. 문제 해결 (ping 안 될 경우)

|증상|확인 사항|
|---|---|
|`Destination Host Unreachable`|두 VM이 같은 NAT 네트워크에 연결되어 있는지 확인|
|`Request timeout`|Ubuntu 방화벽에서 ICMP 차단 여부 확인|
|응답 없음|`ip a` 로 IP가 올바르게 설정됐는지 재확인|

Ubuntu 방화벽 임시 해제 (테스트 목적):

```bash
# Ubuntu 에서 실행
sudo ufw disable
```

---
