---
title: "[애플리케이션 보안] 04-3. 실습 — Zone Transfer 취약점과 DNS 스푸핑"
date: 2026-06-05 18:00:00 +0900
categories:
  - 애플리케이션보안
  - DNS보안
tags:
  - ZoneTransfer
  - AXFR
  - DNS스푸핑
  - dig
  - 실습
math: false
mermaid: true
---

> 이 글은 **실습** 입니다. 앞 글(**04-2**)에서 만든 BIND9 서버(`lab.local`)가 동작 중이어야 합니다.  
> 목표: ① **Zone Transfer(AXFR)** 로 전체 도메인 정보를 통째로 빼내고, ② 이를 **막고**, ③ **DNS 스푸핑(리다이렉션)** 효과를 데모로 본다.
{: .prompt-info }

## A. Zone Transfer 공격 — 전체 도메인 정보 탈취

### A.1 (Kali) AXFR로 Zone 전체 덤프

04-2에서 일부러 `allow-transfer { any; }` 로 열어 뒀습니다. 그래서 **누구나** Zone을 통째로 복제할 수 있습니다.

```bash
# === Kali에서 실행 ===
dig axfr lab.local @192.168.0.30
```

출력에 **모든 레코드가 그대로** 나옵니다.

```
lab.local.        IN  SOA  ns.lab.local. admin.lab.local. 2 ...
lab.local.        IN  NS   ns.lab.local.
lab.local.        IN  MX   10 mail.lab.local.
kali.lab.local.   IN  A    192.168.0.10
mail.lab.local.   IN  A    192.168.0.30
ns.lab.local.     IN  A    192.168.0.30
ubuntu.lab.local. IN  A    192.168.0.30
www.lab.local.    IN  A    192.168.0.30
```

> 🔴 **무엇이 문제인가?** 공격자는 이 한 줄로 **내부 서버 이름·IP 목록 전체** 를 손에 넣습니다. 어디를 공격할지 지도를 그려 주는 셈입니다. 이것이 **정찰(정보수집)** 단계의 대표 기법입니다.
{: .prompt-warning }

---

## B. 방어 — Zone Transfer 제한

### B.1 (Ubuntu) 허용 대상을 제한

Zone Transfer는 원래 **지정된 Secondary 서버에게만** 허용해야 합니다. 우리 실습엔 Secondary가 없으니 아예 막습니다.

```bash
# === Ubuntu에서 실행 ===
sudo nano /etc/bind/named.conf.local
```

`lab.local` 존의 `allow-transfer` 를 다음과 같이 바꿉니다.

```text
zone "lab.local" {
    type master;
    file "/etc/bind/db.lab.local";
    allow-transfer { none; };   # any → none 으로 변경 (외부 복제 차단)
};
```

> 실제 운영에서는 `none` 대신 **Secondary의 IP만** 적습니다. 예: `allow-transfer { 192.168.0.31; };`
{: .prompt-tip }

문법 검사 후 반영합니다.

```bash
# === Ubuntu에서 실행 ===
sudo named-checkconf
sudo systemctl restart named
```

### B.2 (Kali) 다시 시도 → 차단 확인

```bash
# === Kali에서 실행 ===
dig axfr lab.local @192.168.0.30
```

이번에는 레코드가 안 나오고 **`; Transfer failed.`** 가 출력됩니다. → 방어 성공!

> 단, 개별 조회(`dig @192.168.0.30 www.lab.local`)는 여전히 정상 동작합니다. **"전체 복제(AXFR)만" 막은 것**이지 DNS 기능 자체를 끈 게 아닙니다.
{: .prompt-tip }

---

## C. DNS 스푸핑(리다이렉션) 데모 — "악성 DNS를 믿으면 생기는 일"

실제 캐시 포이즈닝은 패킷 주입이 필요해 1학년 실습엔 과합니다(🔴). 대신 **"악성 DNS 서버가 거짓 답을 주면 어떻게 되는지"** 그 효과만 안전하게 데모합니다.

### C.1 (Ubuntu) 가짜 도메인 레코드 심기

우리 BIND에 가짜 은행 도메인 `examplebank.com` 을 만들고, **공격자(Kali) IP** 로 향하게 합니다.

```bash
# === Ubuntu에서 실행 ===
sudo nano /etc/bind/named.conf.local
```

아래 존을 추가합니다.

```text
zone "examplebank.com" {
    type master;
    file "/etc/bind/db.examplebank.com";
};
```

가짜 Zone 파일을 만듭니다.

```bash
# === Ubuntu에서 실행 ===
sudo nano /etc/bind/db.examplebank.com
```

```text
$TTL    604800
@       IN      SOA     ns.lab.local. admin.lab.local. ( 1 604800 86400 2419200 604800 )
@       IN      NS      ns.lab.local.
@       IN      A       192.168.0.10
www     IN      A       192.168.0.10
```

> `www.examplebank.com` 을 **공격자 IP 192.168.0.10** 으로 응답하도록 거짓 정보를 넣은 것입니다.
{: .prompt-warning }

검사 후 반영합니다.

```bash
# === Ubuntu에서 실행 ===
sudo named-checkconf
sudo named-checkzone examplebank.com /etc/bind/db.examplebank.com
sudo systemctl restart named
```

### C.2 (Kali) 악성 DNS에 물었을 때 결과

```bash
# === Kali에서 실행 ===
dig @192.168.0.30 www.examplebank.com +short
```

결과는 **`192.168.0.10`**(공격자) 입니다.

> 🔴 **의미**: 사용자가 주소창에 진짜 `www.examplebank.com` 을 쳐도, **신뢰하는 DNS가 거짓 답을 주면 공격자 서버로 끌려갑니다.** 이것이 DNS 스푸핑/캐시 포이즈닝이 피싱으로 이어지는 원리입니다.  
> **방어**: 응답에 서명을 붙여 위조를 막는 **DNSSEC**(이론 6장, 🔴 데모)와, 신뢰할 수 있는 DNS 서버 사용.
{: .prompt-warning }

---

## D. 방어 정리

| 위협 | 방어 |
|---|---|
| Zone Transfer 정보수집 | `allow-transfer` 를 **지정 Secondary로 제한**(또는 none) |
| 캐시 포이즈닝/스푸핑 | **DNSSEC**(응답 서명), 신뢰 DNS 사용, 재귀 질의 제한 |
| DNS 증폭 DDoS | 외부 재귀 차단, 응답 속도 제한(RRL) |

## E. 체크리스트

- [ ] `allow-transfer any` 상태에서 `dig axfr` 로 전체 Zone 덤프 확인
- [ ] `allow-transfer none` 으로 변경 후 `dig axfr` 가 **Transfer failed** 로 차단됨
- [ ] 개별 조회(`www.lab.local`)는 여전히 정상인 것 확인
- [ ] 스푸핑 데모: `www.examplebank.com` 이 공격자 IP로 응답되는 것 확인
- [ ] Zone Transfer 제한과 DNSSEC의 역할을 설명할 수 있음

> 이로써 **04. DNS 보안** 을 마칩니다. 다음은 **05. DB 보안** — 데이터베이스 공격 유형(이론)과 MariaDB 권한·암호화 실습으로 이어집니다.
{: .prompt-info }
