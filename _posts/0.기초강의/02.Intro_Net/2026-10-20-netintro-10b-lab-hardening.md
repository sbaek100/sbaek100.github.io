---
title: 네트워크 실무 10강 - 장비 하드닝: SSH·AAA·IOS 복구 (따라하기 실습)
date: 2026-10-20 10:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - PacketTracer
  - SSH
  - AAA
  - IOS복구
  - 하드닝
pin:
mermaid: false
---

> **이 실습에서 하는 일**
> 라우터를 안전하게 잠그는 ‘하드닝’을 단계별로 실습합니다. 암호를 강화하고, 평문 Telnet 대신 암호화된 SSH만 허용하며, 계정 인증(AAA)을 적용하고, 불필요한 서비스를 끕니다. 마지막으로 IOS(운영체제)가 지워졌을 때 복구하는 방법을 익힙니다.
{: .prompt-info }

## 실습 10-1. 암호 강화

**1단계 — 특권 모드 암호와 평문 암호 감추기.** 특권 모드 진입 암호는 되돌릴 수 없는 해시로 저장되는 `enable secret`을 씁니다. 또한 설정 파일에 평문으로 보이는 암호들을 감춥니다.

```text
R1(config)# enable secret class123
R1(config)# service password-encryption
```

**2단계 — 최소 암호 길이 지정.** 대회 지사 라우터 조건처럼 “암호는 최소 10자 이상”을 강제할 수 있습니다.

```text
R1(config)# security passwords min-length 10
```

## 실습 10-2. SSH로 원격 접속 잠그기

> **실습 목표** 평문 Telnet 대신 암호화된 SSHv2만 허용한다.
{: .prompt-tip }

SSH를 켜려면 장비에 이름·도메인이 있어야 하고(암호화 키 생성에 필요), 키를 만든 뒤 VTY 회선에서 SSH만 허용하도록 바꿉니다.

```text
R1(config)# hostname R1
R1(config)# ip domain-name secu.com
R1(config)# crypto key generate rsa
   ! 키 길이(모듈러스)를 물으면 1024 이상(대회 조건은 2048) 입력
R1(config)# ip ssh version 2
R1(config)# username admin secret cisco12345
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
R1(config-line)# login local
```

각 명령의 뜻: `crypto key generate rsa`로 암호화 키를 만들고, `ip ssh version 2`로 더 안전한 SSH2를 지정하며, `transport input ssh`로 Telnet을 막고 SSH만 받고, `login local`로 계정 인증을 씁니다.

**확인.** PC에서 `ssh -l admin 172.16.x.x`로 접속되고, `telnet`은 거부되는지 확인합니다. 라우터에서 `show ip ssh`로 활성 상태를 봅니다.

> **[화면 삽입]** PT 9.0 — PC에서 `ssh -l admin`으로 접속 성공하고 `telnet`은 거부되는 화면.
> _(그림 파일 예정: `netintro-10b-01.png`)_

## 실습 10-3. AAA 인증 적용

> **실습 목표** 로컬 계정 기반 AAA 인증을 켜서, 로그인 시 중앙화된 방식으로 인증하게 한다.
{: .prompt-tip }

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login default local
```

`aaa new-model`을 켜면 AAA 체계가 활성화되고, `aaa authentication login default local`은 “로그인 인증은 로컬 계정으로 하라”는 뜻입니다. 실제 대회처럼 인증 서버(RADIUS)를 쓰려면 `local` 대신 서버 그룹을 지정하고 `radius-server host ...`로 서버를 등록합니다(개념만 알아 둡니다).

## 실습 10-4. 불필요한 서비스 끄기 (공격 표면 줄이기)

10강 이론에서 배운 대로, 안 쓰는 서비스는 공격 표면이 되므로 끕니다.

```text
R1(config)# no ip http server
R1(config)# no cdp run
R1(config)# banner motd #Unauthorized access is prohibited#
```

경고 배너(`banner motd`)는 ‘허가되지 않은 접속 금지’를 알리는 보안 관례입니다.

## 실습 10-5. IOS 복구 (개념과 절차)

라우터의 운영체제인 **IOS** 파일이 지워지면 라우터가 정상 부팅하지 못하고 `rommon>` 모드로 떨어집니다. 이때 네트워크의 다른 서버(TFTP 서버)에서 IOS를 내려받아 복구합니다. 대회 지사 라우터의 “A2-SRV로부터 IOS를 복구하라”가 이것입니다. 절차의 큰 흐름은 다음과 같습니다.

1. 라우터가 `rommon>`으로 부팅되면, 복구용 네트워크 정보를 입력한다(자기 IP, TFTP 서버 IP, IOS 파일 이름).
2. `tftpdnld` 명령으로 TFTP 서버에서 IOS 이미지를 내려받는다.
3. 내려받기가 끝나면 라우터를 재부팅(`reset`)해 새 IOS로 부팅한다.

```text
rommon 1 > IP_ADDRESS=172.16.14.254
rommon 2 > IP_SUBNET_MASK=255.255.255.128
rommon 3 > DEFAULT_GATEWAY=172.16.14.129
rommon 4 > TFTP_SERVER=172.16.14.130
rommon 5 > TFTP_FILE=c2900-universalk9-mz.SPA.bin
rommon 6 > tftpdnld
```

> **참고** 패킷 트레이서에서는 TFTP 서버(`[Services] > [TFTP]`)에 IOS 이미지가 있어야 하며, 라우터에서 IOS를 지운 상태(`erase flash` 후 재부팅)에서 위 절차로 복구합니다. 실제 시험에서는 이 과정을 침착하게 따라 하는 것이 관건입니다.
{: .prompt-info }

> **생각해 보기 — 왜 Telnet을 아예 막을까?**
> 실습 10-2에서 `transport input ssh`로 Telnet을 완전히 막았습니다. ‘SSH도 되고 Telnet도 되게’ 둘 다 열어 두면 더 편하지 않을까요? 그런데 Telnet이 하나라도 열려 있으면 공격자는 그 평문 통로를 노립니다. 보안에서 ‘편의를 위해 열어 둔 뒷문’이 왜 위험한지, 그리고 ‘안 쓰는 것은 막는다’는 원칙(공격 표면 줄이기)과 어떻게 연결되는지 생각해 보세요.
{: .prompt-info }

---

이 실습으로 라우터를 하드닝하는 핵심 기법을 익혔습니다. 다음 편([10강 실습 문제])에서 대회 지사 라우터의 보안 정책과 유사한 과제에 도전합니다.
