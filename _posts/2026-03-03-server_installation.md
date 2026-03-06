---
title: 보안 실험을 위한 VM 환경 구축(Ubuntu)
date: 2026-03-03 15:00:00 +0900
categories:
  - 강의
  - 실습
tags:
  - 리눅스
  - 설치
  - Virtualbox
pin: true
---
---

## Ubuntu 24.04 실습 환경 구축 가이드

### Step 1: 필수 파일 다운로드 및 VirtualBox 설치

1. **Ubuntu ISO 다운로드**: [Ubuntu 공식 사이트](https://ubuntu.com/download/desktop)에서 **Ubuntu 24.04 LTS Desktop** 이미지 파일을 다운로드합니다.
   ![](/assets/img/posts/2026-03-03-server_installation-1772519962771.png)
    
2. **VirtualBox 설치**: [VirtualBox 공식 사이트](https://www.virtualbox.org/)에서 Windows용 플랫폼 패키지를 다운로드하여 설치합니다.
    ![](/assets/img/posts/2026-03-03-server_installation-1772524102977.png)
    - **Extension Pack**: USB 3.0 및 원격 접속 지원을 위해 함께 설치하는 것이 좋습니다.()
        
3. **기본 설정**: `도구` > `환경 설정` > `일반`에서 가상 머신이 저장될 **기본 머신 폴더**를 지정합니다.
    

---

### Step 2: 가상 머신 생성 및 Ubuntu 설치

1. **새로 만들기**: 이름은 `Ubuntu_Defense`로 설정하고, 다운로드한 ISO 파일을 선택합니다.
    ![](/assets/img/posts/2026-03-03-server_installation-1772524655470.png)
2. **사용자 정보**: 설치 시 사용할 ID와 PW를 설정합니다 (예: `학번` / `복잡한패스워드`).
    ![](/assets/img/posts/2026-03-03-server_installation-1772524710063.png)
    
3. **자원 할당**: 기본 메모리 **2048MB(2GB)** 이상, 프로세서 **2개** 이상을 권장합니다. 처음 설치시에는 일단 사용 자원을 최대한 늘려줍니다.
   ![](/assets/img/posts/2026-03-03-server_installation-1772524803592.png)

4. **설치 완료**: GUI 환경의 Ubuntu 설치가 완료될 때까지 기다립니다.
    ![](/assets/img/posts/2026-03-03-server_installation-1772524818571.png)
5. 설치 하는데 시간이 좀 걸립니다. 차분히 기다려 주세요. 
![](/assets/img/posts/2026-03-03-server_installation-1772525245680.png)
---

### Step 3: 시스템 패키지 최신화 및 시간 설정

설치 후 터미널(`Ctrl + Alt + T`)을 열어 다음 명령어를 순서대로 실행합니다.

1. **패키지 최신화 (중요)**:
    
    - `sudo apt update`: 설치 가능한 패키지 리스트를 업데이트합니다.        
    - `sudo apt upgrade -y`: 실제 패키지들을 최신 버전으로 업그레이드합니다.
    ![](/assets/img/posts/2026-03-03-server_installation-1772525583831.png)  
      
        
2. **시간 동기화**:
    
    - `sudo timedatectl set-timezone Asia/Seoul`: 시간대를 서울로 변경합니다.        
    - `sudo systemctl restart systemd-timesyncd`: NTP 서버와 시간을 동기화합니다.
    ![](/assets/img/posts/2026-03-03-server_installation-1772525722178.png)

---

### Step 4: 한글 사용 환경 설정 (GUI)

Ubuntu 24는 기본적으로 한글 지원이 부족하므로 별도 설정이 필요합니다.

1. **한글 패키지 설치**:
    
    - `sudo apt install fcitx5-hangul fonts-nanum* -y`.
        
2. **입력기 설정**:
    
    - `Settings` > `Region & Language` > `Manage Installed Languages` 클릭.
        
    - **Keyboard input method system**을 `fcitx`로 변경한 후 **재부팅**합니다.
      ![](/assets/img/posts/2026-03-03-server_installation-1772526121663.png)
    
3. **한글 추가**:
    
    - 재부팅 후 상단 입력기 아이콘 클릭 > `Configure` > `+` 버튼 클릭.        
    - `Hangul`을 검색하여 추가하고, 전역 설정에서 한/영 전환 키를 확인합니다.
    -![](/assets/img/posts/2026-03-03-server_installation-1772526300112.png)
    ![](/assets/img/posts/2026-03-03-server_installation-1772526338170.png)    

---

### Step 5: NAT 네트워크 및 고정 IP 설정 (192.168.0.30)

여러 VM 간 통신을 위해 VirtualBox와 Ubuntu 내부 설정을 변경합니다.

1. **VirtualBox 설정**:
    
    - `도구` > `네트워크` > `NAT 네트워크` 탭에서 `만들기` 클릭.        
      ![](/assets/img/posts/2026-03-03-server_installation-1772526444846.png)
    - **IPv4 접두사**: `192.168.0.0/24`로 수정합니다.
      ![](/assets/img/posts/2026-03-03-server_installation-1772526519421.png)
    - 설치한 Ubuntu 서버에 들어가서 네트워크 설정을 'NAT네트워크' 로 바꾸어줌.
    ![](/assets/img/posts/2026-03-03-server_installation-1772526601626.png)
        
2. **Ubuntu 고정 IP 설정**:
    
    - Ubuntu 우측 상단 `Settings` > `Network` > 유선 연결의 `설정(톱니바퀴)` 클릭.        
    - **IPv4 탭**: Method를 **Manual(수동)**로 변경.        
    - **Address**: `192.168.0.30` / **Netmask**: `24` / **Gateway**: `192.168.0.1`.       
    - **DNS**: `8.8.8.8` 입력 후 적용.
      ![](/assets/img/posts/2026-03-03-server_installation-1772526837292.png)
        
3. **적용**: `sudo systemctl restart NetworkManager` 실행 후 `ip addr`로 확인합니다.
    ![](/assets/img/posts/2026-03-03-server_installation-1772526887555.png)
4. DNS 접속 테스트(DNS 는 Google DNS 활용)
![](/assets/img/posts/2026-03-03-server_installation-1772526902821.png)
---

### Step 6: 최종 스냅샷 저장

모든 설정이 완료된 시점의 상태를 저장합니다.

![](/assets/img/posts/2026-03-03-server_installation-1772526951013.png)

- **스냅샷 이름**: `네트워크 및 한글 설정 완료`.
      ![](/assets/img/posts/2026-03-03-server_installation-1772526978373.png)
    
- 이를 통해 실습 중 문제가 발생해도 언제든 초기 상태로 복구가 가능합니다.
    
---
