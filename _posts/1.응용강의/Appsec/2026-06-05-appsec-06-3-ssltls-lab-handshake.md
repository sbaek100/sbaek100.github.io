---
title: "[애플리케이션 보안] 06-3. 실습 — TLS Handshake 캡처와 보안 점검"
date: 2026-06-05 20:06:00 +0900
categories:
  - 1.응용강의
  - 애플리케이션보안
  - 전자상거래보안
tags:
  - TLS
  - Wireshark
  - Handshake
  - testssl
  - 실습
math: false
mermaid: true
---

> 이 글은 **실습** 입니다. 앞 글(**06-2**)에서 만든 HTTPS 서버(`https://192.168.0.30`)가 동작 중이어야 합니다.  
> 목표: ① 와이어샤크로 **TLS Handshake** 를 직접 잡고, ② **HTTP(평문)와 HTTPS(암호화)** 를 비교하고, ③ `testssl.sh` 로 서버 TLS를 점검한다.
{: .prompt-info }

## A. TLS Handshake 캡처

### A.1 (Kali) 와이어샤크 캡처 시작

```bash
# === Kali에서 실행 ===
sudo wireshark &
```

1. 내부망 인터페이스(192.168.0.10 트래픽이 흐르는 카드, 예 `eth1` — 02-3에서 확인한 그 카드)를 더블클릭해 캡처 시작.
2. 표시 필터에 입력:

   ```
   tls.handshake
   ```

### A.2 (Kali) HTTPS 접속으로 Handshake 유발

캡처를 켠 채, 브라우저에서 `https://192.168.0.30` 을 **새로고침** 합니다. (이미 경고를 통과해 둔 상태)

### A.3 (Kali) Handshake 단계 찾아보기

와이어샤크에 잡힌 패킷에서 이론(06-1 4장)의 순서를 직접 확인합니다.

| 와이어샤크 표시 | 이론의 단계 |
|---|---|
| `Client Hello` | ① ClientHello (지원 암호목록 + 랜덤) |
| `Server Hello` | ② ServerHello (선택 암호 + 랜덤) |
| `Certificate` | ③ 서버 인증서(공개키) 전송 |
| `Application Data` | 🔒 이후 대칭키로 암호화된 실제 데이터 |

- `Server Hello` 패킷을 펼치면 **선택된 암호군(Cipher Suite)** 과 **TLS 버전** 을 볼 수 있습니다.
- `Certificate` 패킷에서 06-2에서 만든 **`CN=ubuntu.lab.local`** 인증서가 전달되는 것을 확인하세요.
- `Application Data` 를 **Follow → TLS Stream** 해도 **암호문만** 보입니다 — 내용이 보호되고 있다는 증거입니다.

---

## B. HTTP(평문) vs HTTPS(암호화) 비교

차이를 눈으로 확인하기 위해, **평문 HTTP** 도 잡아 봅니다.

### B.1 (Kali) HTTP 캡처

표시 필터를 바꿉니다.

```
http
```

브라우저에서 평문으로 접속합니다.

```
http://192.168.0.30
```

### B.2 결과 비교

| 구분 | HTTP (80) | HTTPS (443) |
|---|---|---|
| 와이어샤크에서 본문 | **그대로 읽힘**(HTML 보임) | 암호문(`Application Data`) |
| Follow Stream | 내용 노출 | 알아볼 수 없음 |
| 자물쇠 | 없음 | 🔒 |

> 🟢 **결론**: 같은 웹서버라도 **HTTP는 내용이 평문으로 노출**되고, **HTTPS(TLS)는 암호화**되어 보호됩니다. 그래서 로그인·결제 같은 페이지는 반드시 HTTPS여야 합니다.
{: .prompt-warning }

---

## C. testssl.sh로 서버 TLS 점검

`testssl.sh` 는 서버가 **어떤 TLS 버전·암호군을 쓰는지, 알려진 취약점이 있는지** 자동으로 점검하는 도구입니다.

### C.1 (Kali) 설치·실행

```bash
# === Kali에서 실행 ===
sudo apt install -y testssl.sh        # 이미 있으면 생략됨

testssl.sh 192.168.0.30:443
```

### C.2 결과에서 볼 것

- **Protocols**: TLS 1.2 / 1.3 지원 여부, 옛 SSL/TLS1.0 비활성 여부
- **Cipher 강도**: 약한 암호군(취약) 사용 여부
- **Certificate**: 주체·발급자·유효기간(자체서명 여부)
- **Vulnerabilities**: Heartbleed 등 알려진 취약점 점검 결과

> 우리 서버는 자체서명·기본 설정이라 경고가 나올 수 있습니다. 실서비스라면 이 리포트가 **모두 통과(녹색)** 가 되도록 **TLS1.2+·강한 cipher·공인 CA 인증서** 로 맞춥니다.
{: .prompt-tip }

---

## D. 정리 — SSL/TLS 한눈에

| 확인한 것 | 의미 |
|---|---|
| ClientHello/ServerHello/Certificate | Handshake로 **암호 합의 + 키 교환** |
| Application Data 암호문 | 대칭키로 **본 통신 암호화**(기밀성) |
| HTTP vs HTTPS | 평문 노출 vs 암호화 보호 |
| testssl 리포트 | 버전·cipher·취약점 **점검** |

## E. 체크리스트

- [ ] `tls.handshake` 필터로 **ClientHello·ServerHello·Certificate** 확인
- [ ] `Certificate` 에서 `ubuntu.lab.local` 자체서명 인증서 확인
- [ ] HTTP는 본문이 보이고 HTTPS는 `Application Data`(암호문)인 것 비교
- [ ] `testssl.sh` 로 프로토콜·cipher·취약점 점검
- [ ] TLS가 기밀성·무결성·인증을 어떻게 제공하는지 설명 가능

> 이로써 **06. SSL/TLS** 를 마칩니다. 다음은 **07. IPSEC·SET·OTP** — VPN과 전자상거래 보안 이론, 그리고 OTP(일회용 비밀번호) 실습으로 이어집니다.
{: .prompt-info }
