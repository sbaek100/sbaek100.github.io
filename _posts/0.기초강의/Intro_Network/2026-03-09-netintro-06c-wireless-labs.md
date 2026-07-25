---
title: 네트워크 기초 6장 - 무선통신보안 ③ 무선 네트워크 실습 (연결·WPA2·MAC 필터·SSID 은닉)
date: 2026-03-09 19:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - 무선실습
  - WPA2
  - MAC필터링
  - SSID은닉
  - PacketTracer
pin:
mermaid: true
---

이 편에서는 패킷 트레이서로 무선 네트워크를 직접 구성하며, **연결(Open) → 암호화(WPA2) → 접근제어(MAC 필터) → 노출 최소화(SSID 은닉)** 순으로 보안을 한 겹씩 쌓아 봅니다. 핵심은 암호화가 본질적 방어이고, MAC 필터·SSID 은닉은 이를 돕는 보조 수단이라는 점입니다(앞 편 표 6-5·6-7 참고).

> **실습 6-1. 단순 무선 네트워크 연결**  
> **실습 목표** 무선 공유기와 무선 노트북으로 인증·암호화가 없는 가장 기본적인 무선 네트워크를 구성하고, 개방형(Open) 연결이 어떻게 이루어지는지 확인한다.  
> **주소 계획** SSID = `CLASS-WLAN`, 보안 = 없음(Open), IP = 공유기 DHCP 자동 할당(192.168.0.x)
{: .prompt-tip }

**실습 절차**

1. `[End Devices]`에서 Laptop 1대를, `[Network Devices] > [Wireless Devices]`에서 무선 공유기 `WRT300N` 1대를 작업 공간에 놓는다.
2. Laptop0 클릭 → `[Physical]` 탭에서 노트북 그림의 전원 스위치를 눌러 전원을 끈다.
3. 기본 장착된 유선 모듈을 끌어 슬롯에서 빼내고, 왼쪽 목록의 `WPC300N`(무선 랜카드)을 빈 슬롯에 장착한 뒤 전원을 다시 켠다.
   > **지식 팁** 노트북에는 무선 랜카드가 기본 장착되어 있지 않다. 모듈 교체는 반드시 전원을 끈 상태에서만 가능하며, `WPC300N`을 장착하면 노트북에 `Wireless0` 인터페이스가 생긴다.
4. WRT300N 클릭 → `[GUI]` 탭을 연다. 이 화면이 실제 공유기의 웹 관리 페이지(192.168.0.1)에 해당한다.
5. `[Wireless]` 메뉴에서 `Network Name (SSID)` 칸의 기본값 `Default`를 지우고 `CLASS-WLAN`을 입력한 뒤 맨 아래 `Save Settings`를 누른다.
   > **지식 팁** SSID는 무선 네트워크의 ‘이름표’일 뿐 암호가 아니다. 지금은 Security를 설정하지 않았으므로 누구나 접속할 수 있는 개방형(Open) 망이다.
6. 잠시 기다리면 Laptop0이 자동으로 `CLASS-WLAN`에 연결된다. 노트북과 공유기 사이 무선 연결이 초록색으로 표시되는지 확인한다.
7. Laptop0 → `[Desktop] > [Command Prompt]`를 열고 `ipconfig`를 입력한다. IPv4 Address가 `192.168.0.x`로 표시되면 DHCP로 주소를 받은 것이다.
8. 이어서 `ping 192.168.0.1`로 공유기까지 통신을 확인한다. `Reply from 192.168.0.1`이 나오면 성공이다.

> **결과 확인** 암호 입력 없이 `CLASS-WLAN`에 연결되고, `ipconfig`로 192.168.0.x를 받고, ping에 응답이 오면 성공.  
> **계층 짚기** 개방형 무선망은 편리하지만 트래픽이 평문으로 오가 도청에 그대로 노출된다(표 6-5 ‘무선 트래픽 도청’). 다음 실습에서 이 구멍을 암호화로 막는다.
{: .prompt-tip }

> **실습 6-2. WPA2-PSK 무선 암호화**  
> **실습 목표** 개방형 무선망에 WPA2-PSK(AES) 암호화를 적용하고, 올바른 암호를 가진 단말만 접속됨을 확인한다.  
> **주소 계획** SSID = `CLASS-WLAN`, 보안 = WPA2 Personal / AES, 사전 공유 키 = `Class#2026`
{: .prompt-tip }

**실습 절차**

1. WRT300N → `[GUI] > [Wireless] > [Wireless Security]`로 이동한다.
2. `Security Mode` 드롭다운에서 `WPA2 Personal`을 선택한다.
3. `Encryption`을 `AES`로 두고, `Passphrase` 칸에 `Class#2026`을 입력한 뒤 `Save Settings`를 누른다.
   > **지식 팁** WPA2 암호(Passphrase)는 최소 8자 이상이어야 한다. 이 사전 공유 키(PSK) 하나로 접속하는 모든 단말이 인증되므로, 유출되면 전체가 위험하다(표 6-7 개인용 vs 기업용).
4. 설정을 바꾸는 순간 Laptop0의 무선 연결이 끊어진다(연결선이 빨간색). 노트북이 아직 새 암호를 모르기 때문이다.
5. Laptop0 → `[Config]` 탭 → 왼쪽 `[Wireless0]`을 선택한다.
6. `Authentication`에서 `WPA2-PSK`를 고르고, `Encryption Type`을 `AES`로, `PSK Pass Phrase` 칸에 `Class#2026`을 입력한다.
   > **지식 팁** `[Config]`의 Wireless0 화면은 노트북 무선 랜카드의 설정 창이다. 실제 노트북에서 Wi-Fi 목록을 골라 암호를 넣는 것과 같은 동작이다.
7. 연결선이 다시 초록색으로 살아나면 `[Command Prompt]`에서 `ipconfig`와 `ping 192.168.0.1`로 확인한다.
8. [확인용] Wireless0의 PSK Pass Phrase를 일부러 틀리게(`Class#0000`) 바꿔 본다. 연결이 끊기는지 확인한 뒤 다시 `Class#2026`으로 되돌린다.

> **결과 확인** 올바른 암호로는 연결·IP 획득·ping이 되고, 틀린 암호로는 연결되지 않으면 성공.  
> **계층 짚기** WPA2는 CCMP(AES)로 전파 구간을 암호화하므로 도청해도 내용을 해독할 수 없다. 다만 모두가 같은 암호를 공유하는 PSK 방식이라 암호 유출에 취약하다. 기업용(802.1X/RADIUS)은 앞 편에서 다뤘다.
{: .prompt-tip }

> **실습 6-3. MAC 주소 필터링**  
> **실습 목표** 허용된 MAC 주소를 가진 단말만 접속하도록 필터를 설정하고, MAC 기반 접근 제어의 동작과 한계를 함께 확인한다.  
> **주소 계획** 실습 6-2 설정 유지(WPA2-PSK) + MAC 필터 = 허용 목록에 Laptop0만 등록
{: .prompt-tip }

**실습 절차**

1. Laptop1을 새로 추가하고, 실습 6-1·6-2와 같은 방식으로 `WPC300N`을 장착해 `CLASS-WLAN`(WPA2, `Class#2026`)에 접속시킨다. 두 노트북이 모두 연결됨을 먼저 확인한다.
2. 각 노트북의 무선 MAC 주소를 확인한다. Laptop0 → `[Config] > [Wireless0]` 화면 위쪽의 `MAC Address` 값을 적어 두거나, `ipconfig /all`의 `Physical Address` 항목으로 확인한다. Laptop1도 같은 방법으로 확인한다.
   > **지식 팁** MAC 주소는 랜카드마다 공장에서 부여된 고유 번호(48비트)로, 제조사 식별 3바이트 + 일련번호 3바이트로 이루어진다.
3. WRT300N → `[GUI] > [Wireless] > [Wireless MAC Filter]`로 이동한다.
4. `Wireless MAC Filter`를 `Enabled`로 바꾼다.
5. “Permit PCs listed below to access the wireless network”(목록의 PC만 허용)를 선택한다.
6. MAC Address 목록의 첫 칸에 Laptop0의 MAC만 입력하고 `Save Settings`를 누른다.
7. Laptop0은 연결을 유지하지만, 목록에 없는 Laptop1은 연결이 끊겨 IP를 받지 못한다. Laptop1 → `[Command Prompt]`에서 `ipconfig` 결과가 `169.254.x.x`(자동 사설 주소)이거나 주소를 받지 못하면 차단된 것이다.

> **결과 확인** 허용 목록의 Laptop0만 통신되고, 목록에 없는 Laptop1은 WPA2 암호를 알아도 차단되면 성공.  
> **지식 팁 — 생각해 보기** MAC 주소는 무선 구간에 평문으로 노출된다. Laptop0을 잠시 끈 뒤 Laptop1의 `[Config] > [Wireless0]`에서 MAC Address를 Laptop0 값으로 바꾸면(스푸핑) 필터를 그대로 통과한다. 따라서 MAC 필터는 보조 수단일 뿐, 강한 인증·암호화와 함께 써야 한다(표 6-5 ‘MAC 노출·스푸핑’).  
> **계층 짚기** MAC 필터링은 데이터링크 계층(2계층)의 주소로 접근을 통제하는 방식이다. 손쉽지만 위조에 약하다는 한계가 뚜렷하다.
{: .prompt-tip }

> **실습 6-4. SSID 은닉**  
> **실습 목표** SSID 브로드캐스트를 꺼서 네트워크 이름을 숨기고, 숨긴 망에 수동으로 접속해 본다. ‘숨김’이 곧 보안은 아님을 이해한다.  
> **주소 계획** 실습 6-2·6-3 설정 유지 + SSID Broadcast = Disabled
{: .prompt-tip }

**실습 절차**

1. (준비) 실습 6-3의 MAC 필터는 이번 관찰을 방해하지 않도록 잠시 `Disabled`로 돌려 둔다.
2. WRT300N → `[GUI] > [Wireless] > [Basic Wireless Settings]`로 이동한다.
3. `SSID Broadcast` 항목을 `Disabled`로 바꾸고 `Save Settings`를 누른다.
   > **지식 팁** SSID Broadcast를 끄면 공유기가 비콘(Beacon) 프레임에 이름을 실어 알리지 않는다. 그래서 주변 단말의 Wi-Fi 목록에 나타나지 않는다.
4. Laptop1 → `[Desktop] > [PC Wireless] > [Connect]` 탭을 열어 네트워크 목록에 `CLASS-WLAN`이 더 이상 보이지 않음을 확인한다.
5. 이번에는 `[Config] > [Wireless0]`에서 SSID 칸에 `CLASS-WLAN`을 직접 정확히 입력하고, WPA2 암호(`Class#2026`)를 넣어 수동으로 접속한다.
6. 연결이 초록색으로 살아나면 `[Command Prompt]`에서 `ipconfig`와 `ping 192.168.0.1`로 확인한다.

> **결과 확인** 목록에서 SSID가 사라지지만, 이름을 정확히 입력하면 정상 접속되면 성공.  
> **지식 팁 — 생각해 보기** SSID 은닉은 목록에서 이름을 감출 뿐이다. 단말이 접속을 시도할 때 SSID가 프레임에 실려 나가므로 트래픽 분석으로 쉽게 드러난다. 은닉은 정찰을 늦추는 보조 수단일 뿐이며, 실제 보안은 WPA2 암호화(6-2)와 접근제어(6-3)가 담당한다.  
> **계층 짚기** 네 실습을 거치며 연결(Open) → 암호화(WPA2) → 접근제어(MAC) → 노출 최소화(SSID 은닉) 순으로 보안이 한 겹씩 쌓였다. 핵심은 암호화가 본질적 방어이고, MAC 필터·SSID 은닉은 이를 돕는 보조 수단이라는 점이다.
{: .prompt-tip }

---

이것으로 제1부 네트워크 기초의 무선 파트를 마칩니다. 다음 장에서는 네트워크가 정상적으로 동작하는지 확인하고 문제를 진단하는 **네트워크 관리** 기술을 다룹니다.
