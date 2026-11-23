---
title: 네트워크 실무 8강 - 메일 서버와 무선 연결 (따라하기 실습)
date: 2026-10-06 10:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - PacketTracer
  - 메일서버
  - 무선
  - WPA2
pin:
mermaid: false
---

> **이 실습에서 하는 일**
> 서버에 메일 서비스(SMTP·POP3)를 구성하고 계정을 만들어, PC 두 대가 실제로 메일을 주고받게 합니다. 이어서 듀얼 밴드 무선 공유기에 WPA2 암호화를 걸고, 노트북은 5GHz, PC는 2.4GHz로 연결합니다.
{: .prompt-info }

## 실습 8-1. 메일 서버 구성과 송수신

> **실습 목표** 서버에 SMTP·POP3 메일 서비스를 만들고, 두 사용자가 메일을 주고받게 한다.
> **주소 계획** 메일 서버 `192.168.10.100/24`, 도메인 `class.com`, 계정 `user1`·`user2`(암호 `cisco`)
{: .prompt-tip }

**실습 절차**

**1단계 — 서버 IP·연결.** 스위치에 서버 1대와 PC 2대를 연결합니다. 서버에 고정 IP `192.168.10.100`을 설정하고, 두 PC에도 같은 대역(`.11`, `.12`)의 IP와 DNS 서버(`192.168.10.100`)를 설정합니다.

**2단계 — 메일 서비스 켜기.** 서버 → `[Services] > [EMAIL]`로 이동합니다. `SMTP Service`와 `POP3 Service`를 모두 `On`으로 바꾸고, `Domain Name` 칸에 `class.com`을 입력한 뒤 `Set`을 누릅니다.

**3단계 — 계정 만들기.** 같은 화면 아래 User Setup에서 `user1`(비밀번호 `cisco`)을 입력하고 `+`로 추가합니다. `user2`도 같은 방법으로 추가합니다.

> **[화면 삽입]** PT 9.0 — 서버 `[Services] > [EMAIL]`에 도메인 `class.com`, 사용자 `user1`·`user2`가 등록된 화면.
> _(그림 파일 예정: `netintro-08b-01.png`)_

**4단계 — PC1의 메일 클라이언트 설정.** PC0 → `[Desktop] > [Email]`을 엽니다. 아래처럼 입력하고 `Save`를 누릅니다.

| 항목 | 값 |
|---|---|
| Your Name | user1 |
| Email Address | user1@class.com |
| Incoming/Outgoing Mail Server | 192.168.10.100 |
| User Name | user1 |
| Password | cisco |

PC1도 같은 방법으로 `user2@class.com`으로 설정합니다.

**5단계 — 메일 보내기.** PC0의 Email 창에서 `Compose`를 눌러, 받는 사람 `user2@class.com`, 제목·본문을 입력하고 `Send`를 누릅니다.

**6단계 — 메일 받기.** PC1의 Email 창에서 `Receive`를 누르면 user2 앞으로 온 메일이 도착합니다.

> **결과 확인** PC0이 보낸 메일이 PC1에 도착하면 완료입니다. 보낼 때 SMTP, 받을 때 POP3가 쓰였습니다. 실패 시 ① 서버의 SMTP·POP3가 모두 On인지, ② 도메인·계정 철자, ③ 클라이언트의 메일 서버 주소를 점검합니다.
{: .prompt-tip }

## 실습 8-2. 무선 연결 (WPA2, 듀얼 밴드)

> **실습 목표** 무선 공유기에 WPA2를 걸고, 노트북은 5GHz, PC는 2.4GHz로 각각 연결한다.
> **주소 계획** 2.4GHz SSID `home2.4`, 5GHz SSID `home5`, 둘 다 WPA2 Personal/AES, 암호 `class2026`
{: .prompt-tip }

**실습 절차**

**1단계 — 장비 배치.** `[Network Devices] > [Wireless Devices]`에서 듀얼 밴드 공유기(Home Router-PT-AC 또는 WRT300N)를 놓습니다. 무선 노트북(Laptop)과 무선 PC를 준비합니다. (노트북·PC는 전원을 끄고 유선 모듈을 빼낸 뒤 무선 랜카드 모듈을 장착해야 합니다.)

**2단계 — 공유기 무선 보안 설정.** 공유기 → `[GUI]` 탭에서 `[Wireless]`로 이동합니다. 2.4GHz 대역의 SSID를 `home2.4`, 5GHz 대역의 SSID를 `home5`로 설정합니다. 각 대역의 `Wireless Security`에서 `WPA2 Personal`, 암호화 `AES`, Passphrase `class2026`을 입력하고 저장합니다.

> **[화면 삽입]** PT 9.0 — 공유기 GUI의 무선 설정. 2.4GHz `home2.4`, 5GHz `home5`, 둘 다 WPA2 Personal/AES 설정 상태.
> _(그림 파일 예정: `netintro-08b-02.png`)_

**3단계 — 노트북을 5GHz에 연결.** 노트북 → `[Config] > [Wireless0]`(또는 `[Desktop] > [PC Wireless]`)에서 SSID `home5`, Authentication `WPA2-PSK`, 암호 `class2026`을 입력합니다. 5GHz 대역에 연결됩니다.

**4단계 — PC를 2.4GHz에 연결.** 무선 PC는 같은 방법으로 SSID `home2.4`, 암호 `class2026`으로 연결합니다.

**5단계 — 확인.** 각 단말의 `[Command Prompt]`에서 `ipconfig`로 공유기 DHCP가 준 IP를 확인하고, `ping [공유기 주소]`로 통신을 확인합니다.

> **결과 확인** 노트북은 `home5`(5GHz), PC는 `home2.4`(2.4GHz)에 각각 WPA2로 연결되고 IP를 받으면 완료입니다. 틀린 암호로는 연결되지 않는 것도 확인해 보세요.
{: .prompt-tip }

> **생각해 보기 — 왜 노트북은 5GHz, PC는 2.4GHz일까?**
> 이 실습에서 노트북은 5GHz, 데스크톱 PC는 2.4GHz에 연결했습니다. 만약 PC가 공유기에서 멀리 떨어진 방에 있다면, 그리고 노트북은 공유기 바로 옆에서 큰 파일을 자주 내려받는다면 이 배치가 왜 합리적일까요? 8강 이론의 두 대역 특성(거리 vs 속도)을 떠올려, 어떤 기기를 어느 대역에 두는 것이 좋을지 생각해 보세요.
{: .prompt-info }

## 잘 안 될 때 점검 순서

| 증상 | 점검할 것 |
|---|---|
| 메일 수신 실패 | 서버 SMTP·POP3가 모두 On인지, 계정 철자·비밀번호, 클라이언트 메일 서버 주소 |
| 무선 연결 안 됨 | 단말에 무선 랜카드 모듈을 장착했는지, SSID·암호가 정확한지, 대역(2.4/5GHz)이 맞는지 |

---

이 실습으로 메일 서버와 무선 연결을 구성했습니다. 다음 편([8강 실습 문제])에서 홈 네트워크(무선+메일) 구성 과제에 도전합니다.
