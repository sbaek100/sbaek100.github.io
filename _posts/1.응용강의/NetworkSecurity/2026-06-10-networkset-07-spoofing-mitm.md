---
title: "[네트워크 보안] 07. 스푸핑과 중간자 공격 - 상대를 속이는 네트워크 공격"
date: 2026-06-10 16:00:00 +0900
categories:
  - 1.응용강의
  - 네트워크보안
  - 스푸핑
tags:
  - ARPSpoofing
  - DNSSpoofing
  - MITM
  - ettercap
  - SSLStripping
math: false
mermaid: true
---

> **학습목표**
> 1. 중간자 공격(MITM)의 개념과 성립 조건을 설명할 수 있다.
> 2. MITM의 주요 종류(ARP·DNS·DHCP 스푸핑, SSL 스트립)를 구분할 수 있다.
> 3. ARP 스푸핑과 DNS 스푸핑을 결합한 공격 흐름을 이해하고 Ettercap으로 실습할 수 있다.
> 4. MITM의 방어 대책을 제시할 수 있다.
{: .prompt-info }

> **이번 실습 대상**: 피해자 Ubuntu `192.168.60.30`, 게이트웨이 `192.168.60.1`, 공격자 Kali `192.168.60.10`.
{: .prompt-info }

지금까지는 특정 호스트의 취약점을 노렸습니다. 이제 표적을 바꿔 두 컴퓨터가 “주고받는 통신 그 자체”를 노립니다. 대화하는 두 사람 사이에 몰래 끼어들어 엿듣거나 말을 바꿔치기하는 공격 — 이것이 **중간자 공격(Man-in-the-Middle, MITM)**입니다.

## 상황

중간자 공격의 핵심은 양쪽 모두 “나는 상대와 직접 통신하고 있다”고 믿게 만드는 데 있습니다. 우체부가 편지를 배달하는 척하면서 몰래 봉투를 열어 읽고, 필요하면 내용을 고쳐 다시 봉해 보내는 상황을 떠올리면 됩니다. 보낸 사람도 받는 사람도 편지가 중간에 열렸다는 사실을 눈치채지 못합니다.

```mermaid
flowchart LR
    U["사용자"] -->|요청| A["공격자<br/>가짜 응답·변조"]
    A -->|전달| S["정상 서버"]
    S -->|응답| A
    A -->|변조 응답| U
```

이런 끼어들기가 어떻게 가능할까요? 많은 네트워크 프로토콜이 “상대가 정말 그 상대가 맞는지”를 확인하지 않도록 설계됐기 때문입니다. 인터넷 초창기에는 참여자들이 서로 신뢰하는 작은 공동체였기에 ARP·DNS 같은 기본 프로토콜에 굳이 인증 장치를 넣지 않았습니다. 편리함을 위해 신뢰를 전제한 이 설계가, 낯선 이들이 뒤섞인 오늘날에는 그대로 MITM의 빌미가 됩니다.

---

## 원리 — 중간자 공격의 종류

| 기법 | 속이는 대상 | 악용하는 취약점 |
|---|---|---|
| ARP 스푸핑 | IP-MAC 매핑 | ARP 무인증 (→ [06장](/posts/networkset-06-arp-sniffing/)) |
| DNS 스푸핑 | 도메인-IP 매핑 | DNS 응답 검증이 Transaction ID뿐 |
| DHCP 스푸핑 | 네트워크 구성(GW·DNS) | DHCP 서버 인증 부재, 빠른 응답 승리 |
| ICMP 리다이렉트 | 라우팅 경로 | ICMP 리다이렉트를 무인증 수용 |
| SSL 스트립 | HTTPS→HTTP 강등 | HSTS 미설정, 사용자가 http로 시작 |
| 세션 하이재킹 | 로그인 세션 | 탈취한 세션 쿠키로 로그인 상태 도용 |

각 기법의 원리는 다음과 같습니다.

- **DNS 스푸핑** — ARP 스푸핑으로 중간자가 된 뒤, 도메인 질의를 가로채 정상 서버보다 **빠르게 가짜 응답**을 보냅니다. 피해자는 위조된 IP(공격자 서버)로 접속합니다.
- **DHCP 스푸핑** — 가짜 DHCP 서버가 진짜보다 빨리 응답해, 클라이언트의 **기본 게이트웨이·DNS를 공격자 IP**로 설정합니다(옵션 코드 3=GW, 6=DNS).
- **ICMP 리다이렉트** — “그 목적지는 나(공격자)를 경유하는 게 더 빠릅니다”라는 위조 ICMP(Type 5)로 라우팅 테이블을 바꿉니다.
- **SSL 스트립** — 서버와는 HTTPS, 클라이언트와는 HTTP로 이어붙여 응답 속 `https://` 링크와 `Secure` 쿠키·HSTS 헤더를 제거해 평문으로 만듭니다.

---

## 공격 실습 — Ettercap DNS 스푸핑

앞의 두 재료를 합쳐 요리를 만듭니다. ARP 스푸핑으로 통신 사이의 “중간 자리”를 잡고([06장](/posts/networkset-06-arp-sniffing/)), DNS의 무인증이라는 약점을 찌릅니다([08장](/posts/networkset-08-dns-dhcp-snmp/)). 그 결과, 피해자가 특정 도메인을 조회할 때 가짜 IP를 돌려주어 위조 사이트로 유도하는 공격을 **Ettercap 하나**로 완성합니다. Ettercap은 ARP 스푸핑·비밀번호 추출·DNS 위조 등 MITM에 필요한 기능을 **플러그인** 방식으로 한자리에 모은 종합 도구입니다.

> **실습 7-1. Ettercap으로 DNS 스푸핑하기**  
> **목표** ARP 스푸핑과 DNS 스푸핑을 동시에 수행해, 피해자의 도메인 조회를 공격자 IP로 위조한다. **대상** Kali(192.168.60.10) / 피해자 Ubuntu(192.168.60.30) / 게이트웨이(192.168.60.1).
{: .prompt-tip }

**1단계 — 환경 준비와 타깃 확인** (Kali)

```bash
sudo sysctl -w net.ipv4.ip_forward=1      # 경유 트래픽을 원래 목적지로 전달
nmap -sn 192.168.60.0/24                    # 피해자·게이트웨이 확인
```

![nmap -sn — 내부망 활성 호스트 확인](/assets/img/posts/2026-06-10-networkset-07-ettercap-01.png)

**2단계 — DNS 위조 규칙 편집**

```bash
sudo cp /etc/ettercap/etter.dns /etc/ettercap/etter.dns.bak
sudo nano /etc/ettercap/etter.dns
# 파일에 아래 규칙 추가
```

```text
google.com    A   192.168.60.10
*.google.com  A   192.168.60.10
naver.com     A   192.168.60.10
*.naver.com   A   192.168.60.10
```

> **왜?** `etter.dns`는 “어떤 도메인을 어떤 IP로 위조할지”를 적어 두는 규칙표입니다. A 레코드(도메인→IPv4)를 공격자 IP로 지정하면, Ettercap은 피해자가 그 도메인을 물을 때 이 가짜 답을 대신 돌려줍니다. 규칙이 없으면 위조할 대상이 없어 아무 일도 일어나지 않습니다.

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-07-ettercap-03.png" alt="etter.dns에 리다이렉션 도메인 추가" style="max-width:48%">
<img src="/assets/img/posts/2026-06-10-networkset-07-ettercap-04.png" alt="설정 확인 및 문법 점검" style="max-width:48%">
</div>

**3단계 — (선택) 가짜 웹서버 켜기**

```bash
sudo systemctl start apache2      # 위조 IP로 접속했을 때 보여 줄 페이지
# 또는 실제 사이트 클론: httrack https://nid.naver.com/nidlogin.login -O /tmp/clone
```

> **왜?** 피해자가 위조 IP로 실제 접속했을 때 무언가 보이게 하려면 공격자 쪽에 웹서버가 떠 있어야 합니다. 유도가 실제로 먹혔는지 눈으로 확인하기 위한 준비입니다.

![httrack으로 복제한 가짜 NAVER 로그인 페이지](/assets/img/posts/2026-06-10-networkset-07-ettercap-02.png)

**4단계 — Ettercap으로 ARP + DNS 스푸핑 실행**

```bash
sudo ettercap -T -i eth0 -M arp:remote -P dns_spoof /192.168.60.1// /192.168.60.30//
#  -T 텍스트모드,  -M arp:remote 양방향 ARP MITM,  -P dns_spoof DNS 위조 플러그인
```

> **왜?** `-M arp:remote`가 먼저 게이트웨이와 피해자를 양쪽으로 속여 중간 자리를 잡고(6장의 두 arpspoof를 한 번에), `-P dns_spoof`가 그 자리에서 지나가는 DNS 질의를 가로채 `etter.dns` 규칙대로 위조 응답을 끼워 넣습니다. 두 타깃(`/192.168.60.1//`, `/192.168.60.30//`)은 “게이트웨이와 피해자 사이”를 가로채라는 지정입니다. GUI(`-G`)에서는 `Hosts scan → 타깃 지정 → MITM: ARP poisoning → Plugins: dns_spoof → Start` 순서로 진행합니다.

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-07-ettercap-05.png" alt="Ettercap GUI 초기 설정" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-07-ettercap-06.png" alt="스캔한 호스트 목록" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-07-ettercap-07.png" alt="MITM → ARP poisoning" style="max-width:32%">
</div>

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-07-ettercap-08.png" alt="Plugins에서 dns_spoof 활성화" style="max-width:48%">
<img src="/assets/img/posts/2026-06-10-networkset-07-ettercap-09.png" alt="dns_spoof 로그 — 도메인이 192.168.60.10으로 위조됨" style="max-width:48%">
</div>

**5단계 — 피해자 시스템에서 위조 확인**

```bash
# (피해자 Ubuntu에서)
nslookup google.com          # 응답 IP가 192.168.60.10 으로 나오는지
```

> **왜?** 피해자에서 조회한 IP가 공격자(192.168.60.10)로 나오면 위조가 성공한 것입니다. 동시에 Kali의 Ettercap 화면에는 `dns_spoof … sending spoofed reply` 로그가 찍힙니다 — 공격이 실제로 응답을 바꿔치기하고 있다는 증거입니다. 브라우저로 접속하면 가짜 사이트가 뜨고, HTTPS 사이트는 **인증서 경고**가 나타납니다. 이 경고가 곧 방어의 실마리입니다.

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-07-ettercap-10.png" alt="피해자 nslookup → 192.168.60.10" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-07-ettercap-11.png" alt="피해자 dig → 192.168.60.10" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-07-ettercap-12.png" alt="가짜 사이트와 연결 안전하지 않음 경고" style="max-width:32%">
</div>

**6단계 — 정지와 원상복구**

```bash
# Ettercap 화면에서 q 로 종료 → ARP 캐시가 정상 복구됨
sudo cp /etc/ettercap/etter.dns.bak /etc/ettercap/etter.dns
sudo sysctl -w net.ipv4.ip_forward=0
```

> **왜?** Ettercap은 종료할 때 오염시킨 ARP 캐시를 원래대로 되돌려 줍니다. 실습을 끝내면 반드시 정상 종료해, 피해자 VM의 통신을 온전한 상태로 돌려놓습니다.

---

## 방어

DNS 스푸핑이 무서운 이유는, 피해자가 아무 잘못도 하지 않았는데 당한다는 데 있습니다. 주소를 정확히 입력해도 가짜 사이트로 끌려가 아이디·비밀번호를 넘겨줄 수 있습니다. 근본 방어는 **DNSSEC**(서명이 맞지 않는 위조 응답 거부, [08장](/posts/networkset-08-dns-dhcp-snmp/))이고, **HTTPS**가 더해지면 위조 사이트가 진짜 인증서를 못 가져 브라우저가 경고를 띄웁니다. 애초에 ARP 스푸핑을 막는 **정적 ARP·DAI**는 이 공격의 전제 자체를 무너뜨립니다.

| 방어 | 막는 공격 |
|---|---|
| DNSSEC | DNS 스푸핑·캐시 포이즈닝 |
| HTTPS + HSTS | SSL 스트립, 위조 사이트(인증서 경고) |
| 정적 ARP · DAI | ARP 스푸핑(MITM의 전제) |
| DHCP Snooping | DHCP 스푸핑 |
| VPN | 구간 전체 암호화 |

인증서 경고를 무시하지 않는 **사용자 습관**도 중요한 방어선입니다.

> **윤리·법적 유의** DNS 스푸핑은 타인을 가짜 사이트로 유인하는 심각한 침해이자 사기의 도구가 될 수 있습니다. 반드시 본인이 만든 격리 실습망의 가상 피해자에게만 수행합니다. 실제 네트워크·타인을 대상으로 하면 정보통신망법·형법 위반으로 처벌됩니다.
{: .prompt-danger }

---

## 기출 연결

| 기출 키워드 | 기억할 내용 |
|---|---|
| 스푸핑 | 신분·주소·이름을 속이는 공격 |
| ARP 스푸핑 | IP-MAC 매핑 위조(인증 부재) |
| DNS 스푸핑 | 도메인-IP 응답 위조, 대응은 DNSSEC |
| DHCP 스푸핑 | 가짜 GW·DNS 배포, 대응은 DHCP Snooping |
| SSL 스트립 | HTTPS→HTTP 강등, 대응은 HSTS |
| 세션 하이재킹 | 탈취한 세션 쿠키로 로그인 도용 |

기출에서는 “스니핑 공격기법이 아닌 것”, “DNS 스푸핑이 악용하는 프로토콜 특성”, “중간자 공격 대응 방안”처럼 공격명과 방어기술을 섞어 묻습니다.
