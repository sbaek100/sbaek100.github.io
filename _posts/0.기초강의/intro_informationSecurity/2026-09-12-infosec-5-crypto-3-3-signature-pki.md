---
title: "4장 ⑧ 현대 암호 - 전자서명과 PKI"
date: 2026-09-12 12:40:00 +0900
categories:
  - 0.기초강의
  - 정보보안
tags:
  - 전자서명
  - PKI
  - 인증서
  - CA
  - 부인방지
  - GPG
  - Kleopatra
pin: false
math: true
mermaid: true
---

# 현대 암호 #2-3: 전자서명과 PKI

> 앞 글 **[공개키 암호 일반(DH·RSA)](/posts/infosec-5-crypto-3-1-publickey/)**에서 만든 "개인키/공개키"를 이용해, **누가 보냈는지(인증)·안 바뀌었는지(무결성)·부인 못 하게(부인방지)** 보장하는 것이 전자서명이고, 공개키의 **주인을 신뢰**하게 해 주는 체계가 PKI다.  
> 메시지 무결성의 기반인 해시·MAC/HMAC은 **[해시 함수와 응용](/posts/infosec-5-crypto-3-2-hash/)** 참고.
{: .prompt-info }

---

## 1. 전자서명의 목적

1. 송신자 신원 확인(인증)
2. 메시지 변조 검출(무결성)
3. 사후 부인 방지(부인방지)

서명 기본 흐름:

1. 메시지 해시 `H(M)` 계산
2. 개인키로 해시에 서명
3. 수신자는 공개키로 서명 검증 + 메시지 해시 재계산 비교

전자서명은 "내용을 숨기는 것"이 아니라 "누가 보냈는지와 내용이 안 바뀌었는지"를 확인하는 기술이다.

예시:

1. 학교가 PDF 공지문을 배포한다.
2. 학교는 공지문 자체를 암호화하는 것이 아니라, 공지문 해시에 개인키로 서명한다.
3. 학생이나 학부모는 학교 공개키로 서명을 검증한다.
4. 검증이 성공하면 "정말 학교가 보낸 문서이고, 중간에 변조되지 않았구나"라고 판단할 수 있다.

여기서 중요한 오해 정리:

- 전자서명 = 암호화 자체가 아니다.
- 전자서명 = 인증 + 무결성 + 부인방지에 초점이 있다.
- 기밀성이 필요하면 별도로 암호화를 함께 써야 한다.

즉, **암호화와 전자서명은 역할이 다르며 같이 쓰일 수도 있다.**

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779146882725.png)

### 1.1 전자서명의 수학적 원리 (RSA 기준)

전자서명은 **공개키 암호의 연산을 거꾸로** 쓴 것이다. 암호화가 "공개키로 잠그고 개인키로 연다"면, 서명은 "**개인키로 잠그고(서명) 공개키로 연다(검증)**".

$$
\text{서명: } S = H(M)^{d} \bmod n \qquad \text{검증: } H(M) \stackrel{?}{=} S^{e} \bmod n
$$

- 오직 **개인키 $d$를 가진 사람만** 올바른 $S$를 만들 수 있음 → **인증·부인방지**.
- 누구나 **공개키 $(e,n)$로** $S^e$를 계산해 $H(M)$과 비교 → 변조 시 불일치 → **무결성**.
- $e \times d \equiv 1 \pmod{\varphi(n)}$ 관계로 검증이 성립하는 수학적 근거는 **[⑥ 수학 배경 — 오일러 정리](/posts/infosec-5-crypto-3-6-math/#a7-오일러-정리)** 참고. (타원곡선 기반 서명은 ECDSA)

> 메시지 전체가 아니라 **해시 $H(M)$에 서명**하는 이유는 ① 속도(긴 문서 대신 고정 길이 요약), ② 안전성 때문이다. 해시의 성질은 **[해시 함수와 응용](/posts/infosec-5-crypto-3-2-hash/)** 참고.
{: .prompt-tip }

---

## 2. 전자 서명 (그림)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147382979.png)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147392547.png)

---

## 3. PKI(Public Key Infrastructure)

공개키를 신뢰하려면 “이 키가 정말 그 사람의 키인가?”를 검증해야 한다.

PKI는 이를 위한 기반구조다.

- CA(인증기관): 인증서 발급/폐기
- RA(등록기관): 신원 확인 절차 지원
- 인증서/CRL/체인 검증: 신뢰 경로 확인

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779146427576.png)
![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779146444778.png)

핵심 문제는 이것이다.  
공개키 암호에서는 공개키를 남에게 보여줘도 되지만, **그 공개키가 진짜 누구 것인지** 는 별도로 확인해야 한다.

예를 들어 공격자가 "이게 은행 공개키입니다"라고 가짜 공개키를 보내면, 사용자는 속아서 공격자 키로 암호화할 수 있다.  
이 문제를 해결하기 위해 등장한 것이 **인증서(certificate)** 와 **PKI** 다. (이는 앞 글의 [DH 한계 — MITM 공격](/posts/infosec-5-crypto-3-1-publickey/#34-한계-mitm-공격)을 막는 장치이기도 하다.)

인증서를 쉽게 말하면:

- "이 공개키는 정말 이 사이트/이 사람의 것이다"라고
- 믿을 수 있는 기관이 전자서명해서 붙여 둔 디지털 증명서

브라우저가 HTTPS 사이트에 접속할 때는 보통 다음을 확인한다.

1. 인증서를 누가 발급했는가(Issuer)
2. 인증서가 지금도 유효한가(유효기간)
3. 접속한 도메인 이름과 인증서의 도메인이 일치하는가
4. 중간 CA와 루트 CA를 따라 올라가는 신뢰 체인이 맞는가
5. 폐기된 인증서가 아닌가

학교 비유:

1. 학생증에 이름과 사진이 적혀 있다.
2. 하지만 그냥 종이에 이름만 적었다고 학생증이 되지는 않는다.
3. 학교가 공식 도장을 찍어 줘야 신뢰할 수 있다.

PKI에서 CA의 역할이 바로 이 "공식 도장"에 가깝다.

따라서 PKI는 단순히 키를 저장하는 곳이 아니라, **공개키의 주인을 믿을 수 있게 만들어 주는 신뢰 체계** 라고 정리하면 된다.

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779146464708.png)

---

## 4. GPG4Win 실습 흐름

여기까지의 개념을 실제 도구로 연결해 보는 단계가 GPG4Win 실습이다.  
PDF 후반부는 GPG4Win 기반의 실습 절차를 안내하며, 앞에서 배운 공개키 암호, 전자서명, 무결성 개념이 실제로 어떻게 쓰이는지 체험하게 한다.

공식 사이트:

- [GPG4Win](https://www.gpg4win.org/)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147475571.png)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147483762.png)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147493033.png)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147502808.png)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147512143.png)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147522371.png)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147536944.png)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147547227.png)

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147556042.png)
![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147565901.png)
![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147574857.png)
![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147583881.png)
![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147592898.png)
![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779147601498.png)

### 4.1 Kleopatra (GUI) 실습 가이드
GPG4Win 패키지에 포함된 핵심 그래픽 관리 도구인 **Kleopatra**를 사용하여 다음 단계를 파트너(동료 학생)와 함께 차근차근 실습해 보세요.

#### 1단계: 개인별 OpenPGP 키 쌍 생성
1. **Kleopatra** 실행 후 상단 메뉴에서 `파일(File) -> 새 키 쌍(New Key Pair)`을 클릭합니다.
2. `개인 OpenPGP 키 쌍 생성(Create a personal OpenPGP key pair)`을 선택합니다.
3. 이름과 이메일을 입력합니다. (실습용이므로 가상의 정보를 입력해도 무방합니다.)
4. `생성(Create)`을 누르면 **비밀번호(Passphrase)** 입력 창이 뜹니다. 개인키를 보호할 비밀번호를 설정하고 잘 기억해 둡니다.

#### 2단계: 나의 공개키 내보내기 (Export) 및 파트너와 공유
1. Kleopatra 메인 화면에서 방금 생성한 키를 우클릭하고 **내보내기(Export...)**를 선택합니다.
2. 파일 이름을 `[내이름]_pubkey.asc`로 하여 저장합니다.
3. 이 `.asc` 파일(텍스트 파일로 열면 `-----BEGIN PGP PUBLIC KEY BLOCK-----`으로 시작함)을 이메일이나 메신저를 통해 파트너에게 전달합니다.

#### 3단계: 파트너의 공개키 가져오기 (Import) 및 인증 (Certify)
1. 파트너에게 받은 `.asc` 파일을 저장합니다.
2. Kleopatra 상단 메뉴의 **가져오기(Import...)**를 클릭하여 파트너의 `.asc` 파일을 선택합니다.
3. 가져오기가 완료되면 해당 키를 우클릭한 뒤 **인증(Certify...)**을 누릅니다. 
   - *교수님의 팁:* "인증"이란 상대방의 신원을 실제로 확인하고 이 공개키가 진짜 상대방의 것임을 나의 서명으로 보증하는 행위입니다.

#### 4단계: 파일 암호화 + 전자서명 (Encrypt & Sign)
파트너에게 보낼 중요한 문서(예: `report.txt`)를 작성합니다.
1. Kleopatra 메인 화면에서 **서명/암호화(Sign/Encrypt...)** 버튼을 클릭합니다.
2. 암호화할 파일을 선택합니다.
3. **타인을 위해 암호화(Encrypt for others)** 옵션을 체크하고, 수신인 목록에서 **파트너의 공개키**를 추가합니다. (이로써 파트너만 복호화할 수 있게 됩니다.)
4. **다음으로 서명(Sign as)** 옵션을 체크하고, 나의 키를 선택합니다. (이로써 내가 보냈음을 증명하게 됩니다.)
5. **서명/암호화(Sign/Encrypt)** 버튼을 누르면 나의 개인키 비밀번호 입력을 요구합니다. 비밀번호를 입력하면 `report.txt.gpg` 파일이 생성됩니다. 이 파일을 파트너에게 전송합니다.

#### 5단계: 수신된 파일 복호화 + 서명 검증 (Decrypt & Verify)
파트너가 보낸 `.gpg` 파일을 받았습니다.
1. 받은 `report.txt.gpg` 파일을 더블클릭하거나 Kleopatra 화면으로 끌어다 놓습니다(Drag & Drop).
2. 나의 개인키 비밀번호(Passphrase)를 묻는 창이 뜨면, **내 키를 만들 때 설정했던 비밀번호**를 입력합니다.
3. 복호화된 결과 화면을 확인합니다:
   - **"Decryption succeeded"** (복호화 성공) 문구와 함께 원본 파일이 복원됩니다.
   - **전자서명 검증 결과**도 함께 표시되며, 파트너의 이름과 함께 서명이 유효한지(Valid Signature) 알려줍니다. 만약 누군가 중간에 암호화된 파일을 임의로 훼손했다면 검증 실패 경고가 발생합니다.

### 4.2 보완 자료: 크로스 플랫폼 CLI GnuPG (`gpg`) 실습 가이드
터미널 환경(Windows PowerShell, macOS Terminal, Linux Bash 등)을 선호하거나 GUI 도구 설치가 어려운 경우, 표준 명령행 도구인 `gpg`를 이용해 동일한 과정을 실습할 수 있습니다. 

이 실습은 명령어를 직접 입력해 보며 공개키 암호학의 내부 매커니즘을 보다 직관적으로 이해하는 데 큰 도움이 됩니다.

#### 1단계: GnuPG 설치 확인
대부분의 리눅스/macOS에는 기본 설치되어 있습니다. 윈도우의 경우 GPG4Win 설치 시 CMD/PowerShell에서 `gpg` 명령어가 자동으로 등록됩니다.
```bash
gpg --version
```

#### 2단계: 키 쌍 생성
터미널에서 아래 명령을 실행하고 안내에 따라 이름, 이메일, 비밀번호(Passphrase)를 입력합니다.
```bash
gpg --generate-key
```

#### 3단계: 내 공개키를 텍스트 형식으로 내보내기
상대방에게 줄 공개키 파일을 텍스트(ASCII Armor) 형식으로 추출합니다.
```bash
gpg --armor --export "your_email@example.com" > my_public_key.asc
```
*(참고: 생성된 `my_public_key.asc` 파일을 상대방에게 전달합니다.)*

#### 4단계: 상대방의 공개키 등록 및 신뢰 설정
받은 상대방의 공개키를 내 시스템에 등록합니다.
```bash
gpg --import partner_public_key.asc
```
상대방 키가 안전한지 검증하고 서명(인증)을 통해 신뢰를 부여합니다.
```bash
gpg --sign-key "partner_email@example.com"
```

#### 5단계: 파일 암호화 및 서명 동시에 수행
파트너의 공개키로 암호화하고, 내 개인키로 서명합니다.
```bash
# --encrypt: 암호화
# --sign: 전자서명 추가
# --recipient (-r): 수신자(상대방)의 이메일 지정
# --local-user (-u): 송신자(나)의 이메일 지정
gpg --encrypt --sign --recipient "partner_email@example.com" --local-user "your_email@example.com" secret.txt
```
*결과물로 `secret.txt.gpg` 이진 파일이 생성됩니다. 이 파일을 상대방에게 보냅니다.*

#### 6단계: 복호화 및 서명 검증
상대방이 보낸 `.gpg` 파일을 받아서 복호화하고 서명을 동시에 검증합니다.
```bash
gpg --decrypt secret.txt.gpg
```
실행하면 내 개인키 비밀번호를 묻고, 비밀번호가 맞으면 다음과 같은 출력과 함께 원문이 표시됩니다.
```text
gpg: encrypted with 2048-bit RSA key, ID 1234ABCD..., created 2026-06-01
      "Your Name <your_email@example.com>"
gpg: Signature made 06/01/26 10:30:00 KST
gpg:                using RSA key 5678EFGH...
gpg: Good signature from "Partner Name <partner_email@example.com>" [full]
```
`Good signature from...` 메시지는 서명이 변조되지 않았으며 파트너가 확실히 작성했음을 증명합니다.

### 4.3 요약 및 관찰 포인트
이 실습을 통해 정보보안 관점에서 다음을 반드시 숙지해 주세요:

| 실습 시나리오 | 사용하는 키 (송신/수신) | 달성하는 보안 목표 |
| :--- | :--- | :--- |
| **암호화 (Encryption)** | 수신자의 **공개키 (Public Key)** | **기밀성 (Confidentiality)**: 오직 수신자만이 자신의 개인키로 풀 수 있음. |
| **전자서명 (Signature)** | 송신자의 **개인키 (Private Key)** | **인증 및 무결성 (Authentication, Integrity)**, **부인방지 (Non-repudiation)**: 송신자 외에는 서명할 수 없고, 조금의 변조도 잡아냄. |
| **암호화 + 서명** | 수신자 공개키 + 송신자 개인키 | **기밀성 + 무결성 + 인증 + 부인방지**를 모두 충족하는 안전한 채널 형성. |
