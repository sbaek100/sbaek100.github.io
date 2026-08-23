---
title: "정보보안 기초 4장 ③ 현대 암호 - 공개키 암호의 원리 (DH·RSA)"
date: 2026-09-12 11:00:00 +0900
categories:
  - 0.기초강의
  - 정보보안 기초
tags:
  - 공개키암호
  - Diffie-Hellman
  - RSA
  - ECC
  - PQC
  - 모듈러연산
  - 오일러정리
  - 이산로그
  - 하이브리드암호
pin: false
math: true
mermaid: true
---

# 현대 암호 #2-1: 공개키 암호의 원리 (DH·RSA)

> 이 글은 **공개키 암호의 일반 이론**(왜 필요했나 · 기본 개념 · Diffie-Hellman · RSA · 하이브리드)을 다룬다.  
> 이어지는 글: **[④ 타원곡선 암호(ECC)](/posts/infosec-5-crypto-3-4-ecc/)** → **[⑤ 포스트 양자 암호(PQC)](/posts/infosec-5-crypto-3-5-pqc/)** → **[⑥ 수학 배경(부록)](/posts/infosec-5-crypto-3-6-math/)** → **[⑦ 해시 함수와 응용](/posts/infosec-5-crypto-3-2-hash/)** → **[⑧ 전자서명과 PKI](/posts/infosec-5-crypto-3-3-signature-pki/)**
{: .prompt-info }

---

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1773271919026.png)

## 1. 왜 공개키 암호가 필요했는가

### 1.1 대칭키의 핵심 문제: 키 분배

대칭키 암호는 빠르고 강력하지만, 송신자와 수신자가 같은 키를 안전하게 공유해야 한다.

- 통신 전 키를 어떻게 전달할 것인가?
- 전달 과정에서 도청되면 어떻게 할 것인가?
- 사용자 수가 늘어날수록 키 관리가 폭발적으로 어려워진다.

이 문제를 **Key Distribution Problem**이라고 부른다.

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1773271931336.png)

### 1.2 1970년대 환경 변화

![](/assets/img/posts/2026-09-12-infosec-5-crypto-3-1-publickey-1787494266712.png)



**이런 변화 때문에 “서로 미리 키를 공유하지 않고도 안전하게 시작하는 방법”이 필요해졌다.**

---

## 2. 공개키 암호의 기본 개념

공개키 암호(public key encryption)는 서로 다른 두 키를 사용한다.

- 공개키(public key): 누구에게나 공개
- 개인키(private key): 본인만 보관

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1773271950186.png)

핵심 아이디어:

1. 공개키로 암호화한 것은 대응 개인키로만 복호 가능
2. 개인키로 서명한 것은 대응 공개키로 검증 가능

> 공개키는 “열쇠를 나눠주는 방식”을 바꾼 혁신이다.
{: .prompt-tip }

---

## 3. Diffie-Hellman 키 교환

### 3.1 역사적 의미

1976년 Diffie-Hellman은 공개 채널에서 공통 비밀키를 만드는 방법을 제시했다.  
이 논문(`New Directions in Cryptography`)은 현대 공개키 암호학의 출발점이다.

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1773271970982.png)

> **📜 역사 한 토막 — 3년 먼저 발명하고도 말할 수 없었던 사람들**
>
> 사실 "공개 채널로 비밀을 만든다"는 아이디어는 Diffie-Hellman보다 먼저 나왔다. 영국 정보기관 GCHQ의 James Ellis는 **1969년**에 '비(非)비밀 암호([non-secret encryption](https://nsarchive.gwu.edu/sites/default/files/documents/3035765/Document-02.pdf))'라는 개념을 제안했고, 1973년에 갓 입사한 수학자 Clifford Cocks는 이 아이디어를 전해 들은 **그날 저녁 RSA와 사실상 같은 방식을 고안**했다(RSA 발표보다 4년 앞선다). 이듬해 Malcolm Williamson은 DH 키 교환과 같은 방법을 찾아냈다.
> 
> ![|259x361](/assets/img/posts/2026-09-12-infosec-5-crypto-3-1-publickey-1787494392514.png)

![](/assets/img/posts/2026-09-12-infosec-5-crypto-3-1-publickey-1787494444217.png)

> 그러나 이 모든 것은 국가 기밀이었다. 세 사람은 학계의 다른 이들이 같은 발명으로 세계적 명성을 얻는 것을 지켜보면서도 침묵해야 했고, GCHQ가 이 사실을 공개한 것은 **1997년** — Ellis는 공개를 불과 한 달 앞두고 세상을 떠났다. "공개와 검증이 곧 신뢰"라는 현대 암호학의 원칙과 함께, 밀실의 발명이 역사에서 어떤 자리를 얻는지를 동시에 보여주는 일화다.
>
> *근거: 1997년 12월 GCHQ가 기밀 해제한 내부 문서(Ellis 1970년 보고서, Cocks 1973년 노트, Williamson 1974년 노트); Simon Singh, 『The Code Book』(1999) 6장.*
{: .prompt-tip }

### 3.2 알고리즘 직관

양측이 공개값 `p`(큰 소수), `g`(원시근)를 공유한다.

1. Alice는 비밀값 `a`를 고르고 `A = g^a mod p`를 보냄
2. Bob은 비밀값 `b`를 고르고 `B = g^b mod p`를 보냄
3. Alice: `K = B^a mod p`
4. Bob: `K = A^b mod p`

두 값은 같아 공통키가 된다.

$$
K = g^{ab} \bmod p
$$

![](/assets/img/posts/2026-09-12-infosec-5-crypto-3-1-publickey-1787494536051.png)

#### 키 배송 문제(Key Distribution Problem)를 해결한 의미

- 공개 채널에서도 안전하게 공통 키를 만들 수 있다.
- 공개키 암호학의 출발점이 되었다.
- 인터넷 보안의 기초 토대를 마련했다.
- 두 사람이 미리 만나지 않아도 비밀을 공유할 수 있게 했다.

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1773272019109.png)

### 3.3 왜 안전한가

공격자는 `p, g, A, B`는 볼 수 있지만, `a` 또는 `b`를 찾기 어렵다.  
이 어려움이 **이산로그 문제(Discrete Logarithm Problem)**에 기반한다. 
(수학적 상세는 **[⑥ 수학 배경 — A.8 Diffie-Hellman](/posts/infosec-5-crypto-3-6-math/#a8-diffie-hellmandh-키-교환)**)

### 3.4 한계: MITM 공격

DH만 단독으로 쓰면 인증이 없다.  
공격자가 중간에서 각각과 별도 키를 맺는 Man-in-the-Middle 공격이 가능하다.

결론:

1. DH는 키 교환에 강력하다.
2. 단독으로는 신원 보장이 부족하다.
3. 인증서/서명/PKI와 결합해야 안전하다. (→ **[전자서명과 PKI](/posts/infosec-5-crypto-3-3-signature-pki/)**)

---

## 4. RSA 공개키 암호

### 4.1 핵심 아이디어

RSA(1977, Rivest-Shamir-Adleman)는 “곱셈은 쉽고 소인수분해는 어렵다”는 비대칭성을 이용한다.

기본 식:

$$
C = M^e \bmod n,\quad M = C^d \bmod n
$$

### 4.2 키 생성 개요

1. 큰 소수 `p, q` 선택
2. `n = pq`
3. `\phi(n) = (p-1)(q-1)`
4. `gcd(e, \phi(n)) = 1`인 `e` 선택
5. `ed \equiv 1 \pmod{\phi(n)}`를 만족하는 `d` 계산

공개키는 `(e, n)`, 개인키는 `(d, n)`이다. (왜 복호화가 되는지의 증명은 **[⑥ 수학 배경 — A.7 오일러 정리](/posts/infosec-5-crypto-3-6-math/#a7-오일러-정리)**)

> **📜 역사 한 토막 — 와인 파티에서 태어난 RSA, 그리고 "4경 년"의 호언**
>
> 1977년 4월, MIT의 세 사람은 유월절 파티에서 돌아온 밤에 결정적 진전을 이뤘다. 소파에 누워 있던 Rivest가 "곱하기는 쉽고 소인수분해는 어렵다"를 이용하는 구성을 떠올려 날이 밝기 전에 논문 초안을 완성한 것이다 — 1년 가까이 40여 차례 시도하고, 그때마다 Adleman이 결함을 찾아 무산시킨 끝이었다. Adleman은 "내 기여는 적으니 이름을 빼 달라"고 했다가 설득 끝에 세 번째 저자로 남았고, 그렇게 **R-S-A**가 되었다.
>
> 그해 8월, 과학 칼럼니스트 Martin Gardner는 Scientific American에 RSA를 소개하며 **129자리 숫자로 암호화한 문장(RSA-129)** 을 현상금 100달러와 함께 실었다. 당시 추정 해독 시간은 "약 4경(4×10¹⁶) 년". 그러나 컴퓨터 성능과 인수분해 알고리즘이 나란히 발전하면서, **17년 뒤인 1994년** 전 세계 자원봉사자 600여 명이 인터넷으로 계산을 나눠 8개월 만에 풀어냈다. 숨어 있던 답은 "**THE MAGIC WORDS ARE SQUEAMISH OSSIFRAGE**"라는 뜻 없는 문장이었다.
>
> 교훈은 명확하다. **"현재 기술로 몇 년"이라는 안전성 평가는 유통기한이 있는 진술**이며, 키 길이는 시대와 함께 자라야 한다(RSA-129는 약 426비트, 오늘날 권장은 2048비트 이상이다).
>
> *근거: M. Gardner, "A New Kind of Cipher That Would Take Millions of Years to Break"(Scientific American, 1977년 8월호); D. Atkins 외, "The Magic Words are Squeamish Ossifrage"(ASIACRYPT '94) — 1994년 해독 팀의 공식 논문; 탄생 비화는 Rivest·Adleman의 회고 인터뷰(예: Adleman의 SIAM News 인터뷰)와 Simon Singh, 『The Code Book』(1999)에 전한다.*
{: .prompt-tip }

### 4.3 RSA 실습 링크

- [RSA 키 생성](https://emn178.github.io/online-tools/rsa/key-generator/)
- [RSA 복호](https://emn178.github.io/online-tools/rsa/decrypt/)


#### 💡 실습 시나리오: Alice와 Bob의 비밀 통신
이해를 돕기 위해 Alice(수신자)와 Bob(송신자)이 RSA 도구를 사용하여 메시지를 안전하게 주고받고 서명하는 구체적인 예제를 따라해 봅시다.

![](/assets/img/posts/2026-09-12-infosec-5-crypto-3-1-publickey-1787494963841.png)

##### 1단계: Alice의 키 쌍 생성 (Key Generation)
1. **[RSA 키 생성]** 링크를 클릭합니다.
2. Key Size를 `1024` 또는 `2048`로 선택하고 **Generate** 버튼을 누릅니다.
3. 화면에 생성된 두 개의 키 박스를 확인합니다:
   - **Public Key (공개키, $e$와 $n$)**: 모든 사람에게 공개되는 키입니다.
   - **Private Key (개인키, $d$와 $n$)**: Alice 자신만 안전하게 보관해야 하는 키입니다.
4. 실습을 위해 각각의 키를 메모장에 따로 복사해 둡니다.

##### 2단계: Bob이 Alice의 공개키로 메시지 암호화 (Encryption)
Bob은 Alice에게 "Meet me at 10 PM"이라는 비밀 메시지를 보내고 싶습니다.
1. **[RSA 암호화]** 도구(또는 복호화 도구 내 암호화 탭)를 엽니다.
2. **Public Key** 입력란에 **Alice의 공개키**를 붙여넣습니다.
3. Plaintext 입력란에 `Meet me at 10 PM`을 입력하고 **Encrypt**를 누릅니다.
4. 출력된 암호문(Base64 또는 Hex 형태의 무작위 문자열)을 복사하여 Alice에게 보냅니다. (이 암호문은 인터넷에 공개되어도 괜찮습니다.)

##### 3단계: Alice가 자신의 개인키로 메시지 복호화 (Decryption)
Alice는 Bob에게서 받은 암호문을 해독합니다.
1. **[RSA 복호]** 링크를 클릭합니다.
2. **Private Key** 입력란에 오직 자신만 알고 있는 **Alice의 개인키**를 붙여넣습니다.
3. Ciphertext 입력란에 Bob에게 받은 **암호문**을 붙여넣고 **Decrypt**를 누릅니다.
4. 원문인 `Meet me at 10 PM`이 정확히 복원되는지 확인합니다.


---

## 5. 공개키 암호의 장단점

장점:

1. 키 분배 문제 완화
2. 사용자 수 증가 시 확장성 우수
3. 전자서명/인증 체계 구축 가능

단점:

1. 대칭키 대비 연산이 느림
2. 구현 복잡도와 인증 인프라 필요

그래서 실무는 보통 하이브리드 방식을 쓴다.

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779146373916.png)

---

## 6. 하이브리드 암호

실무 표준 흐름:

1. 공개키 방식으로 세션키(대칭키) 교환
2. 실제 데이터는 빠른 대칭키로 처리

```mermaid
flowchart LR
P["공개키 알고리즘"] --> K["세션키 안전 교환"]
K --> S["대칭키 알고리즘으로 대용량 데이터 암호화"]
```

여기서 **세션키(session key)** 는 "지금 이 연결에서만 잠깐 쓰는 일회용 대칭키"라고 생각하면 쉽다.  
예를 들어 웹브라우저가 은행 사이트에 접속하면, 처음부터 로그인 정보 전체를 공개키로 느리게 처리하는 것이 아니라:

1. 서버의 공개키 또는 공개키 기반 키교환 절차를 사용해
2. 브라우저와 서버만 아는 세션키를 안전하게 만들고
3. 그 뒤부터는 AES 같은 빠른 대칭키 암호로 데이터를 주고받는다.

왜 이렇게 나눠 쓰는가:

1. 공개키 암호는 안전한 키교환과 인증에 강하다.
2. 대칭키 암호는 속도가 빨라 대용량 데이터 처리에 유리하다.
3. 두 장점을 합치면 "안전하면서도 빠른" 구조가 된다.

생활식 비유:

- 공개키 암호: "비밀번호가 걸린 안전한 열쇠 전달 절차"
- 대칭키 암호: "전달받은 열쇠로 실제 창고 문을 빠르게 여닫는 과정"

즉, **공개키는 '열쇠를 안전하게 주고받는 역할'**, **대칭키는 '실제 내용을 빠르게 지키는 역할'** 을 맡는다.

![](/assets/img/posts/2026-03-11-infosec-5-crypto-3-1779146405648.png)

> **다음 글로**: RSA·DH의 뒤를 잇는 3세대 공개키 암호 → **[④ 타원곡선 암호(ECC)](/posts/infosec-5-crypto-3-4-ecc/)**
{: .prompt-info }

---

