---
title: 네트워크 기초 3장 - 라우팅 ② 라우터 보안 (모드·암호·ACL·필터링)
date: 2026-03-09 14:00:00 +0900
categories:
  - 0.기초강의
  - 컴퓨터 네트워크 기초
tags:
  - 라우터보안
  - ACL
  - enablesecret
  - ingress필터링
  - uRPF
pin:
mermaid: true
---

앞 편에서 라우터로 네트워크를 연결하고 경로를 학습시켰습니다. 그런데 라우터는 네트워크의 관문(Gateway)인 동시에, 공격자가 가장 먼저 노리는 장비이기도 합니다. 라우터 자체가 뚫리면 그 뒤의 모든 네트워크가 함께 위험해지므로, 이 편에서는 라우터를 안전하게 설정하고 운영하는 방법을 다룹니다.

## 1. 라우터의 모드

라우터는 상황에 따라 서로 다른 권한을 가진 모드로 동작합니다. 모드를 구분하지 못하면 명령어가 먹히지 않거나(권한 부족), 운영 중인 장비 설정을 실수로 바꿔 버릴 수 있습니다.

| 모드 | 프롬프트 | 설명 |
|---|---|---|
| 유저(User) | `Router>` | 로그인 직후 기본 모드. 상태 조회 등 제한된 명령어만 |
| 특권(Privileged) | `Router#` | `enable`로 진입. 모든 show 명령·설정 파일 조작 가능 |
| 전역 설정(Global Config) | `Router(config)#` | `configure terminal`로 진입. 호스트명·인터페이스·암호 등 설정의 시작점 |
| RXBOOT(ROMMON) | `rommon 1>` | 비밀번호 분실·IOS 손상 시 복구용 모드 |
| Setup | (대화식 질문) | 설정 파일이 없을 때 질문에 답하며 기본 설정을 만드는 모드 |

_표 3-8. 라우터의 동작 모드_

모드는 `enable`(유저→특권), `configure terminal`(특권→전역 설정), `exit`(한 단계 상위) 또는 `Ctrl+Z`(설정 모드에서 한 번에 특권 모드로), `disable`(특권→유저)로 오갑니다. 전역 설정 모드 아래에는 인터페이스별(`config-if`), 회선별(`config-line`), 라우팅 프로토콜별(`config-router`) 설정 모드가 있고, 명령 앞에 `no`를 붙이면 취소됩니다(실습 3-2의 `no ip route …`가 그 예).

## 2. 기본 명령어

- **도움말**: 프롬프트에서 `?`를 입력하면 그 위치에서 쓸 수 있는 명령어 목록을 봅니다.
- **호스트 이름 변경**: `hostname R1` → 프롬프트가 `R1(config)#`로 바뀝니다.
- **배너 설정**: `banner motd # 문구 #` — ‘허가되지 않은 접속을 금지한다’는 경고 문구를 넣는 것이 보안 관례입니다.
- **시간 동기화**: `ntp server 시간서버IP` — 시각을 맞춰 두면 사고 시 여러 장비 로그를 시간순으로 비교할 수 있습니다.
- **설정 저장**: `write memory`(또는 `copy running-config startup-config`) — RAM의 running-config를 전원이 꺼져도 남는 NVRAM(startup-config)에 저장합니다. 실행하지 않으면 재부팅 시 설정이 사라집니다.

## 3. 라우터 암호 설정

라우터에는 접근 경로별로 네 가지 암호를 설정할 수 있습니다.

1. **콘솔 패스워드**: `line console 0 → password 암호 → login → exit`. 콘솔 케이블 직접 연결에도 로그인을 요구.
2. **VTY(텔넷) 패스워드**: `line vty 0 4 → password 암호 → login → exit`. `vty 0 4`는 0~4번, 최대 5개 텔넷 세션을 동시에 받는다는 뜻.
3. **enable password**: 특권 모드 진입 암호. 기본적으로 **평문**으로 저장됩니다.
4. **enable secret**: 같은 역할이지만 **MD5 해시**로 암호화되어 저장되며, 둘이 동시에 있으면 항상 enable secret이 우선합니다.

주의할 점은, enable password·line password는 기본적으로 `show running-config`에 평문 그대로 노출된다는 것입니다. `service password-encryption`을 실행하면 타입 7(가역적 암호화)로 바꿔 화면 노출은 막지만 복호화가 가능해 완전히 안전하지는 않습니다. 반면 enable secret은 타입 5(MD5)로 저장되어 되돌릴 수 없으므로, 특권 모드 암호는 **enable secret**을 쓰는 것이 안전합니다.

## 4. 원격 접속(텔넷) 통제

콘솔·VTY 패스워드만으로는 ‘아는 사람만 들어온다’는 최소한의 방어일 뿐입니다. 누가·어디서 접속하는지까지 통제하려면 다음을 함께 씁니다.

1. **계정 기반(ID/PW) 로그인**: `username 이름 password 암호`로 계정을 만들고 `line vty 0 4 → login local`로 지정하면, 접속 시 계정 아이디·비밀번호를 요구해 사용자별 접속 기록을 남길 수 있습니다.
2. **IP 주소 필터링**: 표준 ACL과 `access-class`로 VTY에 접속할 수 있는 출발지 IP를 제한합니다(실습 3-6에서 직접 구성).
3. **사용자별 권한 레벨**: 0(가장 낮음)~15(특권 모드와 동일)로 나눠, `username 이름 privilege 레벨 password 암호`로 계정마다 명령어 범위를 다르게 줍니다.

## 5. 불필요한 프로토콜·서비스 제거

라우터가 기본으로 켜 놓은 서비스 중에는 실무에서 거의 쓰이지 않으면서 공격에 악용될 수 있는 것이 있습니다. 이를 꺼 두는 것이 **공격 표면(Attack Surface)** 을 줄이는 기본입니다. `no service udp-small-servers`(echo·discard 등 스몰 서비스), `no service pad`(X.25의 PAD 서비스), `no service finger`(접속 사용자 정보 노출)를 각각 차단합니다.

## 6. 액세스 리스트(ACL)

**ACL**(Access Control List)은 출발지·목적지 IP, 포트, 프로토콜 등을 기준으로 패킷을 허용(permit)·차단(deny)하는 규칙 목록입니다.

- **Standard ACL(1~99)**: `access-list 번호 {permit|deny} {source [wildcard]|host 주소|any}` 형식으로 출발지 IP만 봅니다. wildcard는 서브넷 마스크를 반전한 값입니다(255.255.255.0 → `0.0.0.255`).
- **Extended ACL(100~199)**: `access-list 번호 {permit|deny} 프로토콜 출발지 wildcard 목적지 wildcard [eq 포트]` 형식으로, 출발지·목적지·프로토콜·포트까지 검사해 훨씬 정교합니다.
- **인터페이스 적용**: `interface … → ip access-group 번호 {in|out}`으로 연결해야 동작합니다(in=수신, out=송신).

ACL을 다룰 때 꼭 기억할 네 가지 규칙이 있습니다. **규칙 1(순차 검사)** 맨 위부터 검사해 처음 일치하는 규칙에서 즉시 결정됩니다. **규칙 2(암묵적 전체 차단)** 맨 끝에 보이지 않는 `deny any`가 붙어 있어, 마지막에 `permit any`가 없으면 일치하지 않는 트래픽은 모두 차단됩니다. **규칙 3(끝에만 추가)** 새 규칙은 항상 맨 뒤에 붙고 중간 줄만 수정할 수 없어, 바꾸려면 전체를 지우고 다시 만들어야 합니다. **규칙 4(미적용 시 전체 허용)** 인터페이스에 연결하지 않으면 아무 효과가 없어 모든 트래픽이 허용됩니다(규칙 2는 ‘맞는 줄이 없는 경우’, 규칙 4는 ‘ACL 자체가 연결 안 된 경우’). 상태는 `show access-lists`·`show ip interface`·`show running-config`로 확인합니다.

## 7. 트래픽 필터링 기법

ACL을 응용하면 위조·비정상 트래픽을 걸러내는 표준 기법을 구현할 수 있습니다.

- **ingress 필터링**: 내부로 유입되는 패킷의 출발지 IP를 검사해, 있을 수 없는 주소(사설·예약 IP)나 내부 IP를 사칭한(스푸핑) 패킷을 차단합니다.
- **egress 필터링**: 내부에서 외부로 나가는 패킷 중 우리 대역이 아닌 IP가 출발지로 찍힌(스푸핑) 패킷을 차단해, 우리 망이 DDoS 반사 공격의 발판이 되는 것을 막습니다.
- **Null routing(블랙홀 필터링)**: 특정 IP·대역으로 가는 트래픽을 존재하지 않는 Null 인터페이스로 보내 버려 차단합니다(`ip route 차단할IP 마스크 null 0`). DDoS 트래픽을 걸러낼 때 쓰며, `no ip unreachables`로 불필요한 ICMP 도달 불가 메시지를 막습니다.
- **Unicast RPF**: 들어온 패킷의 출발지 IP를 라우팅 테이블에서 거꾸로 조회해, 그 IP로 가는 경로가 실제로 이 인터페이스 방향이 맞는지 확인합니다. 일치하면 통과, 아니면 위조 패킷으로 보고 차단(`ip verify unicast reverse-path`).

## 8. 라우팅 프로토콜 보안과 리소스 점검

경로 정보 자체가 조작되면 트래픽을 엉뚱한 곳(공격자)으로 유도할 수 있습니다. 정적 라우팅은 관리자가 직접 입력하므로 공격자가 임의로 바꿀 수 없어 가장 안전합니다. 반면 동적 라우팅(RIP·OSPF)은 위조된 라우팅 정보를 흘려보내는 공격에 노출될 수 있어, **인증 기능**(이웃끼리 정해진 키·패스워드가 일치해야만 정보를 받아들임)을 적용해야 합니다.

라우터가 정상 동작하는지, 설정이 반영됐는지는 show 명령으로 확인합니다.

| 명령어 | 확인 내용 |
|---|---|
| `show running-config` | 현재 RAM에서 동작 중인 설정 |
| `show startup-config` | NVRAM에 저장된, 재부팅해도 남는 설정 |
| `show ip interface brief` | 인터페이스의 IP와 Up/Down 상태 |
| `show ip route` | 라우팅 테이블 |
| `show version` | IOS 버전·부팅 시간·메모리 |
| `show access-lists` | ACL 목록과 규칙별 매치 횟수 |

_표 3-9. 라우터 리소스 점검용 show 명령어_

## 9. 실습

> **실습 3-5. 라우터 기본 보안 설정 — 암호와 배너**  
> **실습 목표** 콘솔·VTY 접속에 로그인 절차를 걸고, 특권 모드 진입에 enable secret을 요구하도록 설정한다. 평문 암호의 위험과 service password-encryption의 효과도 확인한다.
{: .prompt-tip }

**실습 절차**

1. R1 콘솔에서 `enable → configure terminal`로 들어간다.
2. 콘솔 패스워드: `line console 0 → password cisco123 → login → exit`.
3. VTY 패스워드: `line vty 0 4 → password cisco123 → login → exit`.
4. 특권 모드 암호: `enable secret class123`.
5. `R1#`에서 `show running-config`로 콘솔·VTY의 `password cisco123`이 평문 그대로 보이는지 확인한다.
6. `service password-encryption`을 입력한 뒤 다시 `show running-config`로 같은 암호가 타입 7 형태(예: `7 0822455D0A16…`)로 바뀐 것을 확인한다.
7. 경고 배너: `banner motd #Unauthorized access is prohibited#`.
8. PC0의 `Command Prompt`에서 `telnet 192.168.10.254`로 배너 → VTY 암호 → (enable 시) enable secret 순으로 요구되는지 확인한다.

> **결과 확인** 콘솔·텔넷 모두 암호 없이 접근 불가, 특권 모드도 enable secret 없이 진입 불가면 완료. 콘솔·VTY 암호는 짧은 type 7, enable secret은 더 긴 type 5(MD5)로 표시되는 차이도 확인한다.  
> **계층 짚기** 콘솔·VTY 암호는 ‘누가 접속할 수 있는가’, enable 암호는 ‘누가 설정을 바꿀 수 있는가’를 나누어 통제한다. 접속 통제와 조작 통제의 분리가 라우터 보안의 첫걸음이다.
{: .prompt-tip }

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-03-09-netintro-c3-27.png" alt="콘솔 패스워드" style="max-width:32%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-28.png" alt="VTY 패스워드" style="max-width:32%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-29.png" alt="enable secret" style="max-width:32%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-30.png" alt="평문 암호 노출" style="max-width:32%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-31.png" alt="service password-encryption" style="max-width:32%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-32.png" alt="telnet 접속 요구" style="max-width:32%">
</div>

> **실습 3-6. ACL로 텔넷 원격 접속 통제하기**  
> **실습 목표** 계정 기반 로그인과 표준 ACL(access-class)로, 정해진 IP에서만 라우터에 텔넷 접속할 수 있도록 제한한다.  
> **주소 계획** PC0 `192.168.10.1/24`(허용), PC2 `192.168.10.2/24`(차단), 게이트웨이 `192.168.10.254`
{: .prompt-tip }

**실습 절차**

1. SW1에 PC2를 추가 연결하고 IP `192.168.10.2`, 마스크 `255.255.255.0`, 게이트웨이 `192.168.10.254`를 설정한다.
2. `R1(config)#`에서 계정을 만든다: `username admin password cisco123`.
3. VTY 로그인을 계정 인증으로 바꾼다: `line vty 0 4 → login local → exit`.
4. 표준 ACL을 만든다: `access-list 10 permit host 192.168.10.1` → `access-list 10 deny any`.
5. ACL을 VTY에 적용한다: `line vty 0 4 → access-class 10 in → exit`.
6. PC0에서 `telnet 192.168.10.254`로 계정 입력 시 정상 접속되는지 확인한다.
7. PC2에서 `telnet 192.168.10.254`로 계정 화면조차 뜨지 않고 연결이 즉시 거부되는지 확인한다.
8. `R1#`에서 `show access-lists 10`으로 `permit host 192.168.10.1` 줄의 match 횟수가 늘었는지 확인한다.

> **결과 확인** PC0은 텔넷 성공(계정 인증), PC2는 즉시 거부되면 완료. 안 되면 ① `access-class 10 in`이 `line vty 0 4`에 적용됐는지, ② permit이 deny any보다 위에 있는지, ③ PC2 게이트웨이 설정을 확인한다.  
> **계층 짚기** access-class는 ip access-group과 같은 ACL을 쓰되 인터페이스가 아니라 VTY 회선에 적용한다. ‘어떤 트래픽을 통과시킬까’가 아니라 ‘누가 로그인할 수 있을까’를 통제하는 것이다.
{: .prompt-tip }

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-03-09-netintro-c3-33.png" alt="PC2 추가" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-34.png" alt="계정 생성" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-35.png" alt="login local" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-36.png" alt="표준 ACL" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-37.png" alt="access-class 적용" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-38.png" alt="PC0 접속 성공" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-39.png" alt="PC2 접속 거부" style="max-width:24%">
</div>

> **실습 3-7. Extended ACL로 트래픽 방향별 필터링(ingress 필터링)**  
> **실습 목표** Extended ACL로 특정 네트워크에서 오는 특정 트래픽(ICMP)만 골라 차단하고, 나머지는 그대로 통과시킨다.  
> **주소 계획** LAN1 `192.168.10.0/24`(PC0), LAN2 `192.168.20.0/24`(PC1) — 실습 3-1과 동일
{: .prompt-tip }

**실습 절차**

1. (준비) 실습 3-4에서 NAT/PAT까지 진행했다면 원래 상태로 되돌린다. R1: `no ip nat inside source list 1 interface g0/1 overload → interface g0/1 → no ip nat outside → ip address 10.0.0.1 255.255.255.252 → exit → ip route 192.168.20.0 255.255.255.0 10.0.0.2`. R2도 `interface g0/1 → ip address 10.0.0.2 255.255.255.252 → exit → ip route 192.168.10.0 255.255.255.0 10.0.0.1`로 되돌린 뒤 PC0↔PC1 ping이 되는지 확인한다.
2. PC1에서 `ping 192.168.10.1`이 성공하는 것을 먼저 확인해 둔다(차단 전 상태).
3. `R1(config)#`에서 Extended ACL을 만든다: `access-list 101 deny icmp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 echo` → `access-list 101 permit ip any any`.
4. R1의 외부(WAN, R2 방향) 인터페이스로 들어오는 트래픽에 적용한다: `interface g0/1 → ip access-group 101 in → exit`.
5. PC1에서 `ping 192.168.10.1`이 `Request timed out`으로 실패하는지 확인한다.
6. PC0에서 `ping 192.168.20.1`은 여전히 성공하는지 확인한다(ACL이 LAN2→LAN1 방향의 ping만 막았기 때문).
7. `R1#`에서 `show access-lists 101`로 deny 줄의 match 횟수가 올라갔는지 확인한다.

> **결과 확인** PC1→PC0 ping만 차단되고 PC0→PC1 ping은 성공하면 완료. 양쪽 다 막히면 ACL을 g0/0에 잘못 적용했거나 in/out을 반대로 지정했는지, 아무것도 안 막히면 `permit ip any any`가 deny보다 위에 있는지 확인한다.  
> **계층 짚기** Standard ACL은 ‘누가 보냈는가’만, Extended ACL은 ‘누가·어디로·무엇을’까지 봐서 정교하게 걸러낸다. 앞서 이론에서 다룬 ingress 필터링을 실제로 구현한 예다.
{: .prompt-tip }

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-03-09-netintro-c3-40.png" alt="NAT 원복" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-41.png" alt="차단 전 ping 성공" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-42.png" alt="Extended ACL 작성" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-43.png" alt="ip access-group 적용" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-44.png" alt="PC1 ping 차단" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-45.png" alt="PC0 ping 성공" style="max-width:24%">
<img src="/assets/img/posts/2026-03-09-netintro-c3-46.png" alt="show access-lists" style="max-width:24%">
</div>

---

이 장에서는 라우팅의 기본 개념부터 정적·동적 라우팅, 라우팅 알고리즘, RIP·OSPF·BGP, 그리고 라우터 자체를 지키는 보안 설정까지 살펴보았습니다. 다음 장에서는 이 라우팅을 실제로 수행하는 라우터를 비롯해, 네트워크를 물리적으로 구성하는 **장비들**을 계층별로 살펴봅니다.
