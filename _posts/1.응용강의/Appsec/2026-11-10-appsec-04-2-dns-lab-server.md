---
title: "[애플리케이션 보안] 04-2. 실습 — BIND9로 DNS 서버 구축하기"
date: 2026-11-10 10:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - DNS보안
tags:
  - BIND9
  - DNS
  - Zone파일
  - dig
  - 실습
math: false
mermaid: true
---

> 이 글은 **실습** 입니다. 개념은 앞 글(**04-1. DNS 보안 이론**)을 먼저 읽으세요.  
> 목표: **Ubuntu(192.168.56.30)** 에 DNS 서버(BIND9)를 올리고, `lab.local` 도메인을 만들어 **Kali(192.168.56.10)** 에서 조회한다.
{: .prompt-info }

## 0. 목표 그림

```mermaid
flowchart LR
    K["Kali 192.168.56.10<br/>(질의)"] -- "dig www.lab.local" --> U["Ubuntu 192.168.56.30<br/>(BIND9 권한 서버)"]
    U -- "192.168.56.30 응답" --> K
```

우리가 만들 `lab.local` 도메인의 레코드:

| 이름 | 종류 | 값 |
|---|---|---|
| `ns.lab.local` | A | 192.168.56.30 |
| `www.lab.local` | A | 192.168.56.30 |
| `ubuntu.lab.local` | A | 192.168.56.30 |
| `kali.lab.local` | A | 192.168.56.10 |
| `mail.lab.local` | A | 192.168.56.30 |
| `lab.local` | MX | mail.lab.local |

---

## 1. (Ubuntu) BIND9 설치

```bash
# === Ubuntu에서 실행 ===
sudo apt update
sudo apt install -y bind9 bind9-utils bind9-dnsutils

# 서비스 확인 (active (running) 이면 정상, q로 종료)
sudo systemctl status named
```

> BIND9의 서비스 이름은 **`named`** 입니다. (`bind9` 로도 동작하지만 본 실습은 `named` 로 통일)
{: .prompt-tip }

---

## 2. (Ubuntu) Zone 등록 — named.conf.local

DNS 서버에 "`lab.local` 도메인을 내가 책임진다"고 알려 줍니다. `/etc/bind/named.conf.local` 을 편집합니다.

```bash
# === Ubuntu에서 실행 ===
sudo nano /etc/bind/named.conf.local
```

파일 맨 아래에 다음을 추가합니다.

```text
zone "lab.local" {
    type master;
    file "/etc/bind/db.lab.local";
    allow-transfer { any; };   # (주의) 다음 실습 04-3에서 '취약 설정'으로 다룸
};
```

> `allow-transfer { any; }` 는 일부러 **취약하게** 둔 설정입니다. 04-3에서 이게 왜 위험한지 직접 확인하고 고칩니다.
{: .prompt-warning }

---

## 3. (Ubuntu) Zone 파일 작성 — db.lab.local

이제 실제 레코드를 담은 Zone 파일을 만듭니다.

```bash
# === Ubuntu에서 실행 ===
sudo nano /etc/bind/db.lab.local
```

아래 내용을 그대로 입력합니다. (SOA가 맨 위, 그 아래 NS·A·MX 레코드)

```text
$TTL    604800
@       IN      SOA     ns.lab.local. admin.lab.local. (
                          2         ; Serial (수정할 때마다 1 올림)
                     604800         ; Refresh
                      86400         ; Retry
                    2419200         ; Expire
                     604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.lab.local.
ns      IN      A       192.168.56.30
www     IN      A       192.168.56.30
ubuntu  IN      A       192.168.56.30
kali    IN      A       192.168.56.10
mail    IN      A       192.168.56.30
@       IN      MX 10   mail.lab.local.
```

> `@` 는 도메인 자기 자신(`lab.local`)을 뜻합니다. 이름 끝의 **점(`.`)** 은 "완전한 이름"이라는 표시이니 빠뜨리지 마세요.  
> Zone 파일을 고칠 때는 항상 **Serial 번호를 1 올려야** 변경이 반영됩니다.
{: .prompt-warning }

---

## 4. (Ubuntu) 설정 문법 검사 후 반영

DNS는 문법 오류가 있으면 서비스가 통째로 죽습니다. **반드시 검사 먼저** 합니다.

```bash
# === Ubuntu에서 실행 ===
# 전체 설정 문법 검사 (출력이 없으면 정상)
sudo named-checkconf

# Zone 파일 검사 (OK 가 나오면 정상)
sudo named-checkzone lab.local /etc/bind/db.lab.local

# 이상 없으면 반영
sudo systemctl restart named
```

`named-checkzone` 결과가 `zone lab.local/IN: loaded serial 2 / OK` 면 성공입니다.

---

## 5. (Kali) dig·nslookup으로 조회

이제 **Kali** 에서 우리 DNS 서버(`192.168.56.30`)에게 직접 물어봅니다.

```bash
# === Kali에서 실행 ===
# A 레코드 조회
dig @192.168.56.30 www.lab.local

# 짧게 결과만
dig @192.168.56.30 www.lab.local +short      # → 192.168.56.30

dig @192.168.56.30 kali.lab.local +short     # → 192.168.56.10

# MX(메일 서버) 레코드
dig @192.168.56.30 lab.local MX +short       # → 10 mail.lab.local.

# nslookup으로도 가능
nslookup www.lab.local 192.168.56.30
```

- `@192.168.56.30` 은 "이 DNS 서버에게 물어봐" 라는 뜻입니다.
- `www.lab.local` 이 `192.168.56.30` 으로 응답되면 **우리 DNS 서버가 정상 동작** 하는 것입니다.

---

## 6. 체크리스트

- [ ] (Ubuntu) `systemctl status named` 가 running
- [ ] (Ubuntu) `named-checkconf` 오류 없음, `named-checkzone` 가 **OK**
- [ ] (Kali) `dig @192.168.56.30 www.lab.local +short` → `192.168.56.30`
- [ ] (Kali) `dig @192.168.56.30 lab.local MX +short` → `mail.lab.local`

> ⭐ 잘 안 되면: Serial 올렸는지, 이름 끝 점(`.`) 빠뜨리지 않았는지, `named-checkzone` 결과를 다시 확인하세요.
{: .prompt-tip }

---

## 7. 다음 글

DNS 서버가 동작합니다. 다음 글 **04-3. Zone Transfer 취약점과 방어** 에서, 지금 `allow-transfer { any; }` 로 열어 둔 설정 때문에 **Kali가 전체 Zone을 통째로 가져갈 수 있는 것**을 직접 확인하고, 이를 막아 봅니다.
