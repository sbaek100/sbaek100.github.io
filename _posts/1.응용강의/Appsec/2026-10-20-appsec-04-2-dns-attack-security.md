---
title: "[애플리케이션 보안] 04-2. DNS 공격 유형과 보안 대책"
date: 2026-10-20 10:00:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - DNS보안
tags:
  - ZoneTransfer
  - DNSSpoofing
  - CachePoisoning
  - Pharming
  - DNSAmplification
  - DNSSEC
  - BIND
math: false
mermaid: false
---

> **04-1** 에서 만든 DNS 서버(BIND, 존 `lab.test`)가 동작 중이어야 합니다.
{: .prompt-info }

## 0. 실습 환경

| 장비 | IP | 역할 |
|---|---|---|
| Kali | **192.168.0.10** | 공격자·DNS 질의자 |
| Ubuntu | **192.168.0.30** | DNS 서버(BIND) |

> 명령과 예상 화면은 **Ubuntu 24.04.4 LTS(BIND 9.18.39)** 와 **Kali Rolling(DiG 9.20.23)** 에서 실행해 확인했습니다. Transaction ID·시각·프로세스 번호와 인터넷 도메인의 IP 주소는 화면과 다를 수 있습니다. 3.4·3.6·4.1절은 **인터넷 연결**이 필요합니다.
{: .prompt-warning }

이 글의 실습은 모두 **취약한 상태에서 공격이 통하는 것을 먼저 보고, 설정을 바꿔 막히는 것을 확인**하는 순서입니다.

| 절 | Kali에서 할 일 | Ubuntu에서 할 일 |
|---|---|---|
| 3.2 존 전송 | 존 전체 복사: 성공(취약) → 실패(안전) | `allow-transfer { none; }` 적용, 거부 로그 확인 |
| 3.3 버전 노출 | BIND 버전 조회: 보임(취약) → 숨김(안전) | `version` 옵션 적용 |
| 3.4 hosts 변조 | hosts 파일을 바꿔 가짜 IP로 보내기 → 원상 복구 | — |
| 3.5 캐시 포이즈닝 | (이론) | (이론) |
| 3.6 재귀 개방·증폭 | 외부 이름 조회·크기 비교: 허용(취약) → 거부(안전) | `allow-recursion` 적용 |
| 4.1 DNSSEC | — | 서명이 있는 도메인과 없는 도메인 비교 |

> 3.2·3.3·3.6절은 같은 설정 파일 `/etc/bind/named.conf.options` 에 옵션을 **한 줄씩 더해 가며** 고칩니다. 각 단계의 명령은 그때까지의 옵션을 모두 담은 **파일 전체**를 다시 저장하므로, 순서대로 진행합니다.
{: .prompt-info }

---

# 3. DNS 공격 유형

## 3.1 공격 유형 한눈에 보기

### [이론]

DNS는 **인증 없이** 질의와 응답을 주고받도록 만들어졌습니다. 그래서 응답이 정말 진짜 서버에서 왔는지, 중간에 바뀌지 않았는지 확인하지 않습니다. DNS 위협은 보안의 세 요소로 나누면 다음과 같습니다.

| 위협 | 깨지는 보안 요소 | 공격 | 대책 |
|---|---|---|---|
| **응답 엿보기** | 기밀성 | 평문 UDP 53을 도청해 사용자가 어떤 사이트를 찾는지 알아냄 | DoH·DoT(4.2절) |
| **응답 변조·위조** | 무결성 | **DNS 스푸핑**, **캐시 포이즈닝**, **hosts 변조·파밍** → 가짜 사이트로 유도 | **DNSSEC**, hosts 파일 보호 |
| **서비스 거부** | 가용성 | **DNS 증폭(반사) 공격**, DNS 질의 폭주(Query Flooding) | 재귀 제한, 응답 크기·속도 제한, Anycast |
| **정보 노출** | (정찰에 이용) | **존 전송**으로 서버 목록 수집, **버전** 정보 수집 | 존 전송 제한, 버전 숨김 |

> **중계(relay) 취약점은 DNS가 아니라 메일 서버**의 취약점입니다(03-2 3.3절). DNS 취약점 보기에 섞여 나와도 구분할 수 있어야 합니다.
{: .prompt-warning }

| 공격 이름 | 뜻 |
|---|---|
| **DNS 스푸핑**(Spoofing) | 진짜 DNS 서버보다 먼저 **가짜 응답**을 보내 사용자를 속이는 공격의 통칭 |
| **캐시 포이즈닝**(Cache Poisoning) | DNS **서버의 캐시**에 가짜 정보를 넣어, 그 서버를 쓰는 **모든 사용자**를 가짜 사이트로 보냄 |
| **파밍**(Pharming) | DNS·hosts를 조작해, 사용자가 **진짜 주소를 정확히 입력해도** 가짜 사이트에 접속하게 만드는 공격 |

---

## 3.2 존 전송으로 서버 목록 수집

### [이론]

**존 전송(AXFR)** 은 원래 Secondary DNS 서버가 Primary의 존을 복사해 가는 기능입니다(04-1 2.1절). 이를 **아무에게나 허용하면**, 공격자가 명령 한 줄로 도메인 안의 **모든 서버 이름과 IP**를 가져갑니다. 어떤 서버가 있는지 지도를 그려 주는 셈이라, 공격 전 **정보 수집(정찰)** 에 쓰입니다.

| 순서 | 장비 | 할 일 | 상태 |
|---|---|---|---|
| 실습 1 | Kali | 존 전체 복사 시도 | **취약** — 성공 |
| 실습 2 | Ubuntu | 존 전송을 막는 옵션 추가 | **안전**하게 바꿈 |
| 실습 3 | Kali | 다시 시도 | 실패, 개별 조회는 정상 |
| 실습 4 | Ubuntu | 로그에서 거부 기록 확인 | |

### [실습 1] (Kali) 존 전체 복사 — 성공

```bash
# === Kali ===
dig @192.168.0.30 lab.test AXFR
```

```

; <<>> DiG 9.20.23-1-Debian <<>> @192.168.0.30 lab.test AXFR
; (1 server found)
;; global options: +cmd
lab.test.		3600	IN	SOA	ns.lab.test. admin.lab.test. 2026111001 3600 900 604800 3600
lab.test.		3600	IN	NS	ns.lab.test.
lab.test.		3600	IN	MX	10 mail.lab.test.
lab.test.		3600	IN	TXT	"v=spf1 ip4:192.168.0.30 -all"
ftp.lab.test.		3600	IN	CNAME	www.lab.test.
kali.lab.test.		3600	IN	A	192.168.0.10
mail.lab.test.		3600	IN	A	192.168.0.30
ns.lab.test.		3600	IN	A	192.168.0.30
ubuntu.lab.test.	3600	IN	A	192.168.0.30
www.lab.test.		3600	IN	A	192.168.0.30
lab.test.		3600	IN	SOA	ns.lab.test. admin.lab.test. 2026111001 3600 900 604800 3600
;; Query time: 4 msec
;; SERVER: 192.168.0.30#53(192.168.0.30) (TCP)
;; WHEN: Tue Oct 20 11:02:10 KST 2026
;; XFR size: 11 records (messages 1, bytes 336)

```

| 명령·옵션 | 설명 |
|---|---|
| `AXFR` | 레코드 종류 자리에 `AXFR` 을 쓰면 **존 전체 전송**을 요청 |

| 화면 | 의미 |
|---|---|
| SOA부터 SOA까지 11줄 | `lab.test` 존 파일의 **모든 레코드**. 존 전송은 SOA로 시작해 SOA로 끝남 |
| `(TCP)` | 존 전송은 **TCP 53** 을 씀(04-1 1.4절) |
| `XFR size: 11 records` | 레코드 11개를 받음 |

Kali는 Secondary 서버가 아닌데도 존 전체를 받았습니다. BIND 9.18은 `allow-transfer` 를 따로 정하지 않으면 **누구에게나 존 전송을 허용**합니다. 기본값이 취약한 설정입니다.

### [실습 2] (Ubuntu) 존 전송 막기

`named.conf.options` 를 다음 내용으로 바꿉니다. 원래 파일의 설명(주석)을 빼고 옵션만 남긴 뒤, 마지막에 `allow-transfer` 한 줄을 더한 것입니다.

```bash
# === Ubuntu ===
sudo tee /etc/bind/named.conf.options > /dev/null <<'EOF'
options {
    directory "/var/cache/bind";
    dnssec-validation auto;
    listen-on-v6 { any; };
    allow-transfer { none; };
};
EOF
sudo named-checkconf
sudo rndc reload
```

`tee` 와 `named-checkconf` 는 아무것도 출력하지 않습니다.

```
server reload successful
```

| 설정 | 설명 |
|---|---|
| `directory "/var/cache/bind";` | 작업 파일을 두는 곳(원래 값) |
| `dnssec-validation auto;` | 다른 도메인의 DNSSEC 서명을 검증(원래 값, 4.1절) |
| `listen-on-v6 { any; };` | IPv6 주소에서도 질의를 받음(원래 값) |
| **`allow-transfer { none; };`** | 존 전송을 **아무에게도** 허용하지 않음 |

> 실제 운영에서는 `none` 대신 **Secondary 서버의 IP만** 적습니다. 예: `allow-transfer { 192.168.0.31; };`
{: .prompt-tip }

### [실습 3] (Kali) 다시 존 전체 복사 시도 — 실패

```bash
# === Kali ===
dig @192.168.0.30 lab.test AXFR
dig @192.168.0.30 www.lab.test +short
```

```

; <<>> DiG 9.20.23-1-Debian <<>> @192.168.0.30 lab.test AXFR
; (1 server found)
;; global options: +cmd
; Transfer failed.
192.168.0.30
```

| 화면 | 의미 |
|---|---|
| `; Transfer failed.` | 존 전송이 **거부**됨 |
| `192.168.0.30` | 이름 하나씩 묻는 **일반 조회는 정상**. 존 "전체 복사"만 막은 것 |

### [실습 4] (Ubuntu) 거부 기록 확인

```bash
# === Ubuntu ===
sudo journalctl -u named --no-pager | grep denied
```

```
Oct 20 11:04:31 ubuntu named[2468]: client @0x7f3b2c006248 192.168.0.10#41890 (lab.test): zone transfer 'lab.test/AXFR/IN' denied
```

| 명령·옵션 | 설명 |
|---|---|
| `journalctl -u named` | `named` 서비스가 남긴 로그 |
| `--no-pager` | 한 화면씩 끊지 않고 한 번에 출력 |
| <code>&#124; grep denied</code> | `denied`(거부)가 들어 있는 줄만 골라냄 |

로그에 **누가(`192.168.0.10`) 어떤 존(`lab.test`)을 전송하려다 거부되었는지** 남습니다. 존 전송 시도는 공격 전 정찰 신호이므로, 운영 중에는 이런 기록을 눈여겨봅니다.

### 정리

| 상태 | `allow-transfer` | Kali의 `dig ... AXFR` |
|---|---|---|
| **취약** (실습 1) | 설정 없음 (BIND 9.18 기본값 = 모두 허용) | 존 전체 11개 레코드 노출 |
| **안전** (실습 2~4) | `{ none; }` (또는 Secondary IP만) | `Transfer failed.`, 로그에 `denied` |

---

## 3.3 버전 정보 노출과 DNS 서버 운영 보안

### [이론]

BIND는 `version.bind` 라는 특별한 이름으로 질의하면 **자기 버전**을 알려 줍니다. 공격자는 이 버전으로 **알려진 취약점**을 찾아 공격합니다.

| 순서 | 장비 | 할 일 | 상태 |
|---|---|---|---|
| 실습 1 | Kali | 버전 조회 | **취약** — 정확한 버전 노출 |
| 실습 2 | Ubuntu | `version` 옵션 추가 | **안전**하게 바꿈 |
| 실습 3 | Kali | 다시 조회 | 가짜 문구만 보임 |

### [실습 1] (Kali) BIND 버전 조회 — 노출

```bash
# === Kali ===
dig @192.168.0.30 version.bind chaos txt +short
```

```
"9.18.39-0ubuntu0.24.04.8-Ubuntu"
```

| 명령·옵션 | 설명 |
|---|---|
| `version.bind` | BIND가 자기 버전을 담아 두는 특별한 이름 |
| `chaos` | 클래스. 보통의 `IN`(인터넷)이 아니라 서버 정보용 **CHAOS** 클래스로 질의 |
| `txt` | 글자(TXT) 형식으로 달라는 뜻 |

### [실습 2] (Ubuntu) 버전 숨기기

3.2절 옵션에 `version` 한 줄을 더합니다.

```bash
# === Ubuntu ===
sudo tee /etc/bind/named.conf.options > /dev/null <<'EOF'
options {
    directory "/var/cache/bind";
    dnssec-validation auto;
    listen-on-v6 { any; };
    allow-transfer { none; };
    version "not disclosed";
};
EOF
sudo named-checkconf
sudo rndc reload
```

`tee` 와 `named-checkconf` 는 아무것도 출력하지 않습니다.

```
server reload successful
```

| 설정 | 설명 |
|---|---|
| **`version "not disclosed";`** | `version.bind` 질의에 진짜 버전 대신 **따옴표 안의 문구**로 답함 |

### [실습 3] (Kali) 다시 버전 조회 — 숨김

```bash
# === Kali ===
dig @192.168.0.30 version.bind chaos txt +short
```

```
"not disclosed"
```

#### DNS 서버 운영 보안

버전을 숨기는 것은 정보 노출을 줄일 뿐, 취약점 자체를 없애지는 않습니다. DNS 서버는 다음 원칙으로 운영합니다.

| 원칙 | 내용 | 이 실습에서 |
|---|---|---|
| **최신 버전 유지** | 알려진 취약점이 고쳐진 버전으로 계속 업데이트 | `sudo apt upgrade` |
| **root가 아닌 계정으로 실행** | 취약점이 공격당해도 공격자가 얻는 권한을 제한. **root로 실행하는 것은 잘못된 운영** | `named -u bind` (04-1 2.2절) |
| **chroot 환경** | 서버가 접근할 수 있는 디렉터리를 특정 폴더 안으로 가둠. 침입당해도 시스템 전체 파일에 접근하지 못함 | (이론) |
| **불필요하면 제거** | DNS 서비스가 필요 없는 서버에서는 BIND를 지움 | — |
| **정보 최소화** | 버전 숨김, 존 전송 제한, HINFO 레코드 등록하지 않기 | 3.2·3.3절 |
| **DMZ 배치** | 외부에 공개하는 DNS 서버는 내부망이 아닌 **DMZ**에 둠 | — |

> 이름이 비슷한 **root 실행**과 **chroot** 를 혼동하지 않습니다. root로 실행하는 것은 위험한 운영이고, chroot는 피해 범위를 가두는 보안 대책입니다.
{: .prompt-warning }

### 정리

| 상태 | `version` | `version.bind` 조회 결과 |
|---|---|---|
| **취약** (실습 1) | 설정 없음 | `"9.18.39-0ubuntu0.24.04.8-Ubuntu"` — 정확한 버전 |
| **안전** (실습 2·3) | `"not disclosed"` | `"not disclosed"` |

---

## 3.4 hosts 파일 변조와 파밍

### [이론]

04-1 1.2절에서 본 것처럼 PC는 이름을 찾을 때 **hosts 파일을 DNS보다 먼저** 봅니다. 악성코드가 hosts 파일에 `가짜IP  은행주소` 한 줄을 넣으면, 사용자가 은행 주소를 **정확히 입력해도** 가짜 사이트로 접속합니다. 이것이 **파밍**의 대표적인 수법입니다.

| 순서 | 장비 | 할 일 | 상태 |
|---|---|---|---|
| 실습 1 | Kali | 원래 IP 확인, hosts 파일에 가짜 줄 추가 | **취약** — 가짜 IP로 바뀜 |
| 실습 2 | Kali | 가짜 줄 삭제, 파일 권한 확인 | **안전**하게 되돌림 |

### [실습 1] (Kali) hosts 파일에 가짜 줄 넣기

먼저 `www.naver.com` 이 원래 어떤 IP로 바뀌는지 봅니다.

```bash
# === Kali ===
getent hosts www.naver.com
```

```
23.218.185.12   e6030.a.akamaiedge.net www.naver.com www.naver.com.nheos.com www.naver.com.edgekey.net
```

| 명령·옵션 | 설명 |
|---|---|
| `getent hosts 이름` | 웹 브라우저·`ping` 같은 프로그램과 **같은 순서**(hosts 파일 → DNS)로 이름을 IP로 바꿔 보여 줌 |

> IP 주소와 뒤의 별명(CNAME) 목록은 네이버의 설정과 접속 위치에 따라 다르게 나옵니다.
{: .prompt-info }

이제 공격자가 hosts 파일 끝에 가짜 줄을 넣은 상황을 만듭니다. `192.168.0.10`(Kali 자신)을 공격자의 가짜 사이트라고 가정합니다.

```bash
# === Kali ===
echo "192.168.0.10    www.naver.com" | sudo tee -a /etc/hosts
getent hosts www.naver.com
dig www.naver.com +short
```

```
192.168.0.10    www.naver.com
192.168.0.10    www.naver.com
www.naver.com.nheos.com.
www.naver.com.edgekey.net.
e6030.a.akamaiedge.net.
23.218.185.12
```

| 명령·옵션 | 설명 |
|---|---|
| <code>echo "..." &#124; sudo tee -a /etc/hosts</code> | 따옴표 안의 줄을 hosts 파일 끝에 덧붙임. 관리자 권한이 필요해 `sudo tee` 를 씀. `tee` 가 덧붙인 줄을 화면에도 한 번 보여 줌 |

| 화면 | 의미 |
|---|---|
| 첫 줄 `192.168.0.10 www.naver.com` | `tee` 가 파일에 덧붙인 내용 |
| 둘째 줄 (`getent`) | 이제 `www.naver.com` 이 **가짜 IP `192.168.0.10`** 으로 바뀜. 브라우저도 이 IP로 접속함 |
| 나머지 (`dig`) | `dig` 는 hosts 파일을 보지 않고 **DNS 서버에 직접** 물으므로 진짜 IP가 나옴 |

DNS 서버는 진짜 답을 주고 있는데도, PC 안의 hosts 파일 한 줄 때문에 사용자는 가짜 사이트로 갑니다. **`dig` 결과와 `getent`(실제 접속) 결과가 다르면 hosts 파일 변조를 의심**합니다.

### [실습 2] (Kali) 가짜 줄 지우고 권한 확인

```bash
# === Kali ===
sudo sed -i '/www.naver.com/d' /etc/hosts
getent hosts www.naver.com
ls -l /etc/hosts
```

```
23.218.185.12   e6030.a.akamaiedge.net www.naver.com www.naver.com.nheos.com www.naver.com.edgekey.net
-rw-r--r-- 1 root root 195 Oct 20 11:20 /etc/hosts
```

| 명령·옵션 | 설명 |
|---|---|
| `sed -i '/www.naver.com/d' 파일` | `www.naver.com` 이 들어 있는 줄을 **지움**(`d`, delete). `-i` 는 파일을 직접 고침 |
| `ls -l /etc/hosts` | 파일의 권한과 소유자 확인 |

| 화면 | 의미 |
|---|---|
| `getent` 결과 | 다시 **진짜 IP**로 돌아옴 |
| `-rw-r--r-- 1 root root` | 소유자 **root만 쓸 수 있고**(`rw-`), 나머지는 읽기만 가능(`r--`) |

> 파일 크기(`195`)와 시각은 환경마다 다릅니다.
{: .prompt-info }

hosts 파일은 root만 고칠 수 있으므로, 공격자가 이 파일을 바꿨다는 것은 이미 **관리자 권한을 가진 악성코드**가 실행되었다는 뜻입니다.

### 정리

| 상태 | hosts 파일 | `www.naver.com` 접속 IP |
|---|---|---|
| **취약** (실습 1) | 가짜 줄 `192.168.0.10 www.naver.com` | `192.168.0.10`(가짜) — DNS 답과 다름 |
| **안전** (실습 2) | 가짜 줄 삭제, root만 쓰기 가능 | 진짜 IP |

> 대책은 hosts 파일 **권한 관리**와 **변경 감시**, 백신으로 hosts를 바꾸는 악성코드 차단입니다. 윈도우 hosts 파일은 `C:\Windows\System32\drivers\etc\hosts` 입니다.
{: .prompt-tip }

---

## 3.5 DNS 스푸핑과 캐시 포이즈닝

### [이론]

DNS 질의는 대부분 **UDP**로 오가고 인증이 없습니다. 질의를 보낸 쪽은 **먼저 도착한 응답**이 아래 조건만 맞으면 진짜로 믿습니다.

| 응답을 받아들이는 조건 | 크기 | 설명 |
|---|---|---|
| **Transaction ID** 가 같음 | 16비트 = 0~65535 | 04-1 2.4절 `dig` 화면의 `id: 61995` |
| **출발지 포트**가 맞음 | 16비트 | 질의를 보낸 UDP 포트로 응답이 와야 함 |
| 질문 내용이 같음 | — | 물어본 이름과 종류 |

공격자는 진짜 응답보다 **먼저** 위조 응답을 보내려고 합니다.

| 공격 | 방법 | 피해 범위 |
|---|---|---|
| **DNS 스푸핑**(같은 네트워크) | ARP 스푸핑 등으로 질의를 **엿본 뒤** ID와 포트를 알고 위조 응답을 먼저 보냄. `dsniff` 도구 모음의 `dnsspoof` 가 대표적 | 그 네트워크의 사용자 |
| **캐시 포이즈닝**(원격) | 질의를 볼 수 없으므로 DNS 서버가 재귀 질의를 하는 순간에 **ID를 바꿔 가며 위조 응답을 대량으로** 보내 하나라도 맞히기를 노림. 2008년 **카민스키(Kaminsky)** 가 이 공격을 매우 효과적으로 만드는 방법을 발표 | 그 DNS 서버를 쓰는 **모든 사용자**, TTL 동안 계속 |

| 순서 | 캐시 포이즈닝 진행 |
|---|---|
| ① | 공격자가 피해 DNS 서버에 `bank.test` 의 IP를 묻게 만듦 |
| ② | 피해 DNS 서버가 `bank.test` 의 권한 서버에 반복 질의를 보냄 |
| ③ | 공격자가 진짜 응답보다 먼저, ID를 바꿔 가며 "`bank.test` = 공격자 IP" 라는 위조 응답을 대량으로 보냄 |
| ④ | ID가 맞은 위조 응답이 먼저 도착하면 DNS 서버가 이를 **캐시에 저장** |
| ⑤ | TTL 동안 이 서버를 쓰는 사용자는 모두 **공격자 IP**를 받음 → 주소창에 진짜 주소를 쳐도 가짜 사이트 |

| 대책 | 내용 |
|---|---|
| **DNSSEC** | 응답에 **전자서명**을 붙여 위조 응답을 검증으로 걸러냄(4.1절). 가장 근본적인 대책 |
| **출발지 포트 무작위화** | ID(65536가지)에 포트(약 65536가지)까지 맞혀야 하므로 맞힐 확률이 크게 낮아짐. 현재 BIND는 기본 적용 |
| **최신 버전 BIND** | 알려진 포이즈닝 취약점이 고쳐진 버전 사용 |
| **재귀 질의 제한** | 아무나 재귀 질의를 시키지 못하게 하면 ①단계가 어려워짐(3.6절) |

> 실제 캐시 포이즈닝과 ARP 스푸핑은 같은 네트워크의 다른 사용자에게 피해를 줄 수 있어 이 실습에서는 하지 않습니다. 결과는 3.4절의 hosts 변조와 같습니다. 사용자는 **진짜 주소를 입력했는데 가짜 IP**를 받습니다.
{: .prompt-danger }

---

## 3.6 재귀 개방(Open Resolver)과 DNS 증폭 공격

### [이론]

**재귀 질의를 아무에게나 처리해 주는 DNS 서버**를 **오픈 리졸버(Open Resolver)** 라고 합니다. 오픈 리졸버는 **DNS 증폭(반사) 공격**에 악용됩니다.

| 순서 | DNS 증폭 공격(DRDoS) |
|---|---|
| ① | 공격자가 출발지 IP를 **피해자 IP로 위조**해서 오픈 리졸버에 질의를 보냄 |
| ② | 질의는 작지만(약 50바이트), 응답이 큰 레코드(TXT, `ANY` 종류 등)를 묻음 |
| ③ | 오픈 리졸버는 위조된 출발지, 즉 **피해자에게** 큰 응답(수천 바이트)을 보냄 |
| ④ | 수많은 오픈 리졸버를 이용하면 피해자의 회선이 큰 응답으로 가득 차 마비됨 |

| 구분 | DNS 증폭(반사) 공격 | DNS 질의 폭주(Query Flooding) |
|---|---|---|
| 노리는 것 | 피해자의 **회선 대역폭** | **DNS 서버 자원**(CPU·메모리) |
| 방법 | 작은 위조 질의 → 큰 응답을 피해자에게 반사 | DNS 서버에 질의를 대량으로 보냄 |

| 순서 | 장비 | 할 일 | 상태 |
|---|---|---|---|
| 실습 1 | Kali | 인터넷 이름 재귀 질의, 질의·응답 크기 비교 | **취약** — 허용, 응답이 약 24배 |
| 실습 2 | Ubuntu | `allow-recursion` 추가 | **안전**하게 바꿈 |
| 실습 3 | Kali | 다시 질의 | `REFUSED`, 자기 존은 정상 응답 |

### [실습 1] (Kali) 재귀 질의와 증폭 크기 — 허용

```bash
# === Kali ===
dig @192.168.0.30 www.google.com +short
dig @192.168.0.30 google.com TXT +qr | grep -E 'QUERY SIZE|MSG SIZE'
```

```
142.251.155.119
142.251.157.119
142.251.150.119
142.251.154.119
142.251.152.119
142.251.153.119
142.251.151.119
142.251.156.119
;; QUERY SIZE: 51
;; MSG SIZE  rcvd: 1215
```

| 명령·옵션 | 설명 |
|---|---|
| `+qr` | 응답뿐 아니라 **내가 보낸 질의**도 함께 출력. `QUERY SIZE` 가 질의 크기 |

| 화면 | 의미 |
|---|---|
| `www.google.com` 의 IP 목록 | Ubuntu가 Kali 대신 **재귀 질의를 처리**해 줌 |
| `QUERY SIZE: 51` / `MSG SIZE rcvd: 1215` | **51바이트** 질의에 **1215바이트** 응답 → 약 **24배**로 커짐 |

> IP 주소와 크기는 구글의 설정에 따라 다릅니다. Ubuntu의 BIND는 기본적으로 **서버와 같은 네트워크**(`192.168.0.0/24`)에 재귀를 허용하므로 Kali의 질의가 처리됩니다. 이 서버가 인터넷에 공개되어 **누구에게나** 이렇게 답하면 오픈 리졸버가 됩니다.
{: .prompt-info }

### [실습 2] (Ubuntu) 재귀 질의 제한

Kali를 **외부 공격자**로 보고, 재귀 질의는 서버 자신만 쓸 수 있게 제한합니다. 3.3절 옵션에 `allow-recursion` 한 줄을 더합니다.

```bash
# === Ubuntu ===
sudo tee /etc/bind/named.conf.options > /dev/null <<'EOF'
options {
    directory "/var/cache/bind";
    dnssec-validation auto;
    listen-on-v6 { any; };
    allow-transfer { none; };
    version "not disclosed";
    allow-recursion { 127.0.0.1; };
};
EOF
sudo named-checkconf
sudo rndc reload
```

`tee` 와 `named-checkconf` 는 아무것도 출력하지 않습니다.

```
server reload successful
```

| 설정 | 설명 |
|---|---|
| **`allow-recursion { 127.0.0.1; };`** | 재귀 질의는 **서버 자신(127.0.0.1)** 이 보낸 것만 처리 |

> 실제 운영에서는 `{ 192.168.0.0/24; }` 처럼 **내부 사용자 대역**만 적습니다. 외부에 공개하는 권한 DNS 서버는 재귀를 아예 끄기도 합니다(`recursion no;`).
{: .prompt-tip }

### [실습 3] (Kali) 다시 질의 — 거부

```bash
# === Kali ===
dig @192.168.0.30 www.google.com
dig @192.168.0.30 www.lab.test +short
```

```

; <<>> DiG 9.20.23-1-Debian <<>> @192.168.0.30 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 3336
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: e27a1cbb4388bae2010000006ae84f2b9dfe794d432bad12 (good)
; EDE: 18 (Prohibited)
;; QUESTION SECTION:
;www.google.com.			IN	A

;; Query time: 0 msec
;; SERVER: 192.168.0.30#53(192.168.0.30) (UDP)
;; WHEN: Tue Oct 20 11:35:07 KST 2026
;; MSG SIZE  rcvd: 77

192.168.0.30
```

| 화면 | 의미 |
|---|---|
| `status: REFUSED` | 질의를 **거부**함 |
| `flags: qr rd` (`ra` 없음) | 재귀를 요청했지만(`rd`) 이 서버는 **재귀를 해 주지 않음**(`ra` 없음) |
| `WARNING: recursion requested but not available` | 위와 같은 뜻의 `dig` 안내 |
| `EDE: 18 (Prohibited)` | 확장 오류 코드. 접근 제어 때문에 **금지됨** |
| `MSG SIZE rcvd: 77` | 거부 응답은 작아서 **증폭에 쓸 수 없음** |
| 마지막 `192.168.0.30` | 이 서버가 맡은 `lab.test` 는 **권한 서버로서 계속 답함** |

> 재귀를 제한해도 **자기 존에 대한 질의**에는 답합니다. 권한 서버의 역할(자기 도메인 알려 주기)과 캐시 서버의 역할(남의 도메인 대신 찾아 주기)을 나누어 생각하면 됩니다.
{: .prompt-info }

### 정리

| 상태 | `allow-recursion` | Kali → `www.google.com` | 증폭 |
|---|---|---|---|
| **취약** (실습 1) | 설정 없음 (기본값 = 같은 네트워크 허용, 인터넷 공개 시 `any` 와 같은 효과) | IP 응답 | 51 → 1215바이트(약 24배) |
| **안전** (실습 2·3) | `{ 127.0.0.1; }` (실무는 내부 대역만) | `REFUSED` | 77바이트 거부 응답뿐 |

> - DNS 증폭 공격의 대책은 **내부 사용자에게만 재귀(Recursive) 질의를 허용**하는 것입니다. "반복(Iterative) 질의를 내부에만 허용"은 틀린 표현입니다.
> - 그 밖에 **일정 크기를 넘는 DNS 응답 차단**, **IP별 초당 응답 수 제한**(BIND의 RRL, Response Rate Limiting), `ANY` 질의 응답 최소화를 함께 씁니다.
{: .prompt-warning }

---

# 4. DNS 보안 대책

## 4.1 DNSSEC

### [이론]

**DNSSEC(DNS Security Extensions)** 은 DNS 응답에 **전자서명**을 붙여, 받는 쪽이 "이 응답이 진짜 도메인 주인이 만든 것이고 중간에 바뀌지 않았다"는 것을 확인하게 하는 기술입니다.

| 제공함 | 제공하지 않음 |
|---|---|
| **데이터 출처 인증**(진짜 권한 서버가 만든 응답인가) | **기밀성**(응답을 암호화하지 않음. 도청하면 그대로 보임) |
| **무결성**(중간에 바뀌지 않았나) | **서비스 거부 방지**(오히려 서명 때문에 응답이 커짐) |
| → **캐시 포이즈닝·스푸핑 같은 위·변조 방지** | |

> DNSSEC은 **암호화가 아니라 전자서명**입니다. "DNSSEC은 DNS 메시지에 기밀성을 제공한다"는 설명은 틀립니다. 기밀성은 DoH·DoT(4.2절)가 담당합니다.
{: .prompt-danger }

| 레코드 | 뜻 |
|---|---|
| **RRSIG** | 레코드에 붙는 **전자서명** |
| **DNSKEY** | 서명을 확인할 **공개키** |
| **DS** | 하위 도메인 공개키의 해시. **상위 존**에 등록해 하위 키를 보증 |
| **NSEC / NSEC3** | "그런 이름은 **없다**"는 답도 서명으로 증명 |

| 신뢰 체인 | 누가 누구를 보증 |
|---|---|
| 루트(`.`) | 모두가 미리 알고 있는 루트 키에서 출발 |
| ↓ TLD(`com`) | 루트가 `com` 의 DS를 서명해 보증 |
| ↓ 도메인(`cloudflare.com`) | `com` 이 `cloudflare.com` 의 DS를 서명해 보증 |
| ↓ 레코드 | `cloudflare.com` 의 키로 서명한 RRSIG로 A 레코드 확인 |

Ubuntu의 BIND는 `dnssec-validation auto;` 설정(3.2절에서 그대로 둔 줄)으로 **다른 도메인의 서명을 검증**합니다. 서명 검증에 실패한 응답은 사용자에게 전달하지 않습니다.

### [실습] (Ubuntu) 서명이 있는 도메인과 없는 도메인 비교

3.6절에서 재귀 질의를 **서버 자신(127.0.0.1)** 에게만 허용했으므로, Ubuntu에서 자기 BIND에 묻습니다. 이 BIND가 서명을 검증해 줍니다.

```bash
# === Ubuntu ===
dig @127.0.0.1 cloudflare.com +dnssec +short
dig @127.0.0.1 naver.com +dnssec +short
```

```
104.16.132.229
104.16.133.229
A 13 2 300 20261103095108 20261101075108 34505 cloudflare.com. RrJFH1Pt3FMK0yOcLi7RspaS+8JBE5CrppY8JmUeIQo0gBiOuB+lfyG8 idoEBiuEur3PWnmS95W3zhqC/56spg==
223.130.200.236
223.130.192.247
223.130.192.248
223.130.200.219
```

| 명령·옵션 | 설명 |
|---|---|
| `+dnssec` | 서명(RRSIG)도 함께 달라고 요청 |

| 화면 | 의미 |
|---|---|
| `cloudflare.com` 의 `A 13 2 300 ...` 줄 | A 레코드에 붙은 **RRSIG(전자서명)**. `13` 은 서명 알고리즘(ECDSA P-256), 뒤의 두 날짜는 서명의 유효기간, 마지막 긴 글자가 서명 값 |
| `naver.com` 은 IP만 | 이 도메인은 **DNSSEC 서명을 하지 않음**. 응답이 위조되어도 DNSSEC으로는 확인할 방법이 없음 |

> IP 주소, 서명 날짜와 값은 조회할 때마다 다릅니다. 각 회사가 DNSSEC 적용 여부를 바꾸면 결과도 달라질 수 있습니다.
{: .prompt-info }

검증 결과는 응답의 플래그로 확인합니다.

```bash
# === Ubuntu ===
dig @127.0.0.1 cloudflare.com +dnssec | grep 'flags:'
dig @127.0.0.1 naver.com +dnssec | grep 'flags:'
```

```
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
; EDNS: version: 0, flags: do; udp: 1232
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1
; EDNS: version: 0, flags: do; udp: 1232
```

| 화면 | 의미 |
|---|---|
| `cloudflare.com` 의 **`ad`** | **Authenticated Data**. BIND가 루트부터 신뢰 체인을 따라 서명을 **검증하는 데 성공**함 |
| `naver.com` 은 `ad` 없음 | 서명이 없어 검증할 것이 없음 |
| `flags: do` | 질의에 **DNSSEC OK** 표시를 붙여 서명을 요청했다는 뜻(`+dnssec` 옵션) |

> 서명 검증에 **실패**한 응답은 BIND가 사용자에게 주지 않고 `SERVFAIL` 로 답합니다. 공격자가 캐시 포이즈닝으로 가짜 응답을 넣으려 해도, 서명을 위조할 수 없으므로 검증 단계에서 걸러집니다.
{: .prompt-tip }

---

## 4.2 그 밖의 대책

### [이론]

| 대책 | 내용 |
|---|---|
| **DoH / DoT** | DNS 질의를 **HTTPS(DoH, 443)** 나 **TLS(DoT, 853)** 로 암호화해 보냄 → DNSSEC이 주지 못하는 **기밀성**(엿보기 방지)을 보완 |
| **DNS 싱크홀**(Sinkhole) | 악성코드가 접속하는 **명령·제어(C&C) 서버 도메인**의 답을 일부러 **싱크홀 서버 IP**로 바꿔 줌 → 감염 PC와 C&C 서버의 연결을 끊고, 싱크홀에 접속하는 PC로 감염 PC를 찾아냄. **공격이 아니라 방어 기술**(KISA가 운영) |
| **Anycast** | 같은 IP의 DNS 서버를 여러 곳에 두어(04-1 1.1절) DDoS를 분산 |
| **방화벽** | DNS 서버에 꼭 필요한 UDP·TCP 53만 허용. 외부에서 들어오는 존 전송·재귀는 차단 |

---

## 4.3 DNS 보안 대책 종합

| 번호 | 위협 | 대책 | 이 글에서 확인한 방법 |
|---|---|---|---|
| 1 | 존 전송으로 서버 목록 수집 | `allow-transfer { none; }` 또는 Secondary IP만 | 3.2 |
| 2 | 버전 정보 노출 | `version "..."` 으로 숨김, 최신 버전 유지 | 3.3 |
| 3 | 서버 침해 시 피해 확대 | root가 아닌 계정 실행, chroot, 불필요하면 제거 | 04-1 2.2, 3.3 |
| 4 | hosts 변조·파밍 | hosts 파일 권한·변경 감시, 백신 | 3.4 |
| 5 | 스푸핑·캐시 포이즈닝 | **DNSSEC**, 포트 무작위화, 최신 BIND, 재귀 제한 | 3.5, 4.1 |
| 6 | DNS 증폭(반사) 공격 | **내부 사용자에게만 재귀 허용**, 응답 크기·속도 제한(RRL) | 3.6 |
| 7 | 응답 엿보기 | DoH·DoT | 4.2 |
| 8 | 악성코드의 C&C 접속 | DNS 싱크홀 | 4.2 |

---

# 확인 문제

> 1. DNS 서버가 동작하지 않거나 특정 서버로 강제로 접속시키려 할 때, PC에서 이름과 IP를 직접 적어 두는 파일은?
>    ① named.conf
>    ② hosts
>    ③ services
>    ④ inetd.conf
>
> 2. 메일 서버를 지정하는 DNS 레코드와, 그 도메인을 맡은 DNS 서버를 지정하는 레코드를 차례로 쓰시오.
>
> 3. PC가 참조하는 DNS 서버의 캐시에 위조 정보를 넣어 사용자를 위장 사이트로 접속하게 하는 공격은?
>    ① 존 전송 공격
>    ② DNS 캐시 포이즈닝
>    ③ 재귀 질의 폭주
>    ④ 버전 정보 수집
>
> 4. DNSSEC에 대한 설명으로 틀린 것은?
>    ① DNS 응답에 전자서명을 붙인다
>    ② 응답이 위·변조되지 않았음을 확인할 수 있다
>    ③ DNS 메시지를 암호화해 기밀성을 제공한다
>    ④ 상위 존이 하위 존의 키를 보증하는 신뢰 체인을 쓴다
>
> 5. DNS 서버를 안전하게 운영하는 방법이 아닌 것은?
>    ① 최신 버전으로 유지한다
>    ② chroot 환경에서 운영한다
>    ③ root 권한으로 실행한다
>    ④ 사용하지 않으면 서비스를 제거한다
>
> 6. DNS 증폭 공격에 대한 대책으로 틀린 것은?
>    ① 일정 크기 이상의 DNS 응답을 차단한다
>    ② IP별 초당 응답 수를 제한한다
>    ③ 내부 사용자에게만 반복(Iterative) 질의를 허용한다
>    ④ 오픈 리졸버로 동작하지 않게 설정한다
>
> 7. DNS가 TCP 53번 포트를 쓰는 경우 두 가지를 쓰시오.
>
> 8. BIND에서 존 전송을 아무에게도 허용하지 않는 설정을 쓰시오.
>
> 9. DNS 보안 취약점과 거리가 먼 것은?
>    ① 메일 중계(relay) 취약점
>    ② DNS 응답 변조
>    ③ DNS 서비스 거부
>    ④ DNS 응답 엿보기
>
> 10. 악성코드에 감염된 PC가 C&C 서버 도메인을 조회하면 다른 IP를 알려 주어 연결을 끊는 **방어** 기술은?
{: .prompt-tip }

<details>
<summary>정답 보기</summary>

1. <strong>②</strong> hosts 파일은 DNS보다 먼저 적용된다. named.conf는 DNS 서버(BIND)의 설정 파일이다.<br>
2. <strong>MX</strong>, <strong>NS</strong> (존의 관리 정보는 SOA)<br>
3. <strong>②</strong> ①은 서버 목록 수집, ③은 서비스 거부, ④는 정보 수집이다.<br>
4. <strong>③</strong> DNSSEC은 출처 인증과 무결성만 제공하고 기밀성과 서비스 거부 방지는 제공하지 않는다.<br>
5. <strong>③</strong> root로 실행하면 취약점이 공격당했을 때 root 권한을 빼앗긴다. BIND는 bind 계정으로 실행한다.<br>
6. <strong>③</strong> 내부 사용자에게만 <strong>재귀(Recursive)</strong> 질의를 허용해야 한다.<br>
7. 응답이 512바이트를 넘을 때(UDP 응답에 tc 표시), 존 전송(AXFR/IXFR)<br>
8. <code>allow-transfer { none; };</code><br>
9. <strong>①</strong> 중계(relay)는 메일 서버의 취약점이다.<br>
10. <strong>DNS 싱크홀</strong>

</details>

---

# 체크리스트

- [ ] DNS 위협을 엿보기(기밀성)·변조(무결성)·서비스 거부(가용성)·정보 노출로 나누어 설명 가능
- [ ] (Kali·Ubuntu) `dig ... AXFR` 이 기본 설정에서 성공하고, `allow-transfer { none; }` 적용 뒤 `Transfer failed.` 가 나오며 로그에 `denied` 가 남는 것을 확인
- [ ] (Kali·Ubuntu) `version.bind` 조회 결과가 실제 버전에서 `"not disclosed"` 로 바뀌는 것을 확인
- [ ] root 실행이 위험한 이유와 chroot의 의미를 설명 가능
- [ ] (Kali) hosts 파일에 가짜 줄을 넣으면 `getent` 는 가짜 IP, `dig` 는 진짜 IP가 나오는 것을 확인하고 원상 복구
- [ ] Transaction ID와 출발지 포트를 이용해 캐시 포이즈닝의 원리와 대책을 설명 가능
- [ ] (Kali·Ubuntu) 재귀 허용 상태에서 51→1215바이트 증폭을 확인하고, `allow-recursion` 적용 뒤 `REFUSED` 확인
- [ ] (Ubuntu) `+dnssec` 으로 RRSIG·`ad` 가 있는 도메인과 없는 도메인을 비교하고, DNSSEC이 제공하지 않는 것(기밀성, DoS 방지)을 설명 가능
