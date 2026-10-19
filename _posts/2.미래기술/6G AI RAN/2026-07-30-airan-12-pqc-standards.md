---
title: "[6G AI-RAN] 12. 양자 암호(PQC), 표준화 및 미래 연구 과제"
date: 2026-07-30 11:00:00 +0900
categories:
  - 2.미래보안
  - 6G AI RAN
  - Part IV 검증·미래
tags:
  - PQC
  - Quantum-ML
  - 3GPP
  - O-RAN-WG11
  - ETSI
  - EU-AI-Act
  - Agentic-AI
math: true
mermaid: false
---

# 양자 암호(PQC), 표준화 및 미래 연구 과제

## 들어가며 — 지금 암호화한 데이터는 언제까지 안전한가

6G 상용화 목표 시점은 **2030년경**입니다(Ch1 그림 0-2). 그리고 오늘 O-RAN(Open Radio Access Network)[^a-oran] 인터페이스를 흐르는 데이터는 TLS(Transport Layer Security)[^a-tls]·DTLS(Datagram Transport Layer Security)[^a-dtls]·IPsec(Internet Protocol Security)[^a-ipsec]으로 보호되며, 그 밑에는 **RSA(Rivest–Shamir–Adleman)[^a-rsa]·ECDH(Elliptic Curve Diffie–Hellman)[^a-ecdh]·ECDSA(Elliptic Curve Digital Signature Algorithm)[^a-ecdsa]** 같은 고전 공개키 암호가 있습니다.

> **Harvest-now, decrypt-later.** 공격자는 지금 트래픽을 수집해 저장해 두고, 충분히 큰 양자컴퓨터가 등장한 뒤에 복호화할 수 있습니다. 가입자 식별자·위치 이력처럼 **수십 년간 민감성이 유지되는 데이터**를 다루는 RAN(Radio Access Network)[^a-ran]에서, 이것은 미래의 문제가 아니라 **지금의 문제**입니다.
{: .prompt-danger }

이 장의 구조:

1. PQC — 무엇을 언제 어디에 적용하는가
2. **양자 ML(QML) 보안** — 6G 인에이블러의 새 공격면
3. 표준화 지형 — 3GPP · O-RAN WG11 · ETSI · NIST · EU AI Act
4. 미래 연구 과제 — 체인 모델, QML, LLM·에이전틱 AI
5. 종합 로드맵과 시리즈 마무리

> **범위 안내**: **3GPP[^a-3gpp] TS[^a-ts] 33.501, O-RAN WG11[^a-wg] 규격(Threat Modeling·SecReq·Shared O-RU[^a-oru]), ETSI[^a-etsi] PAS[^a-pas](TR[^a-tr] 104 106 / TS 104 104)** 는 참조 자료 폴더의 원문을 근거로 서술했습니다(§3, 그림 12-1~12-7). 반면 **NIST[^a-nist] FIPS[^a-fips] 203/204/205와 IR[^a-ir] 8547(PQC[^a-pqc])** 은 원문을 보유하지 않아 **문서 수준 인용**입니다 — **PQC 전환 일정과 알고리즘 목록은 표준화 진행에 따라 갱신되므로 실제 적용 시 NIST 원문을 확인**하시기 바랍니다.
{: .prompt-info }

---

## 1. PQC — 무엇을 언제 어디에 적용하는가

### 1.1 표준화된 알고리즘

| 표준 | 알고리즘 | 용도 | 기반 |
|---|---|---|---|
| **FIPS 203** | **ML-KEM**[^a-ml-kem] (CRYSTALS-Kyber 기반) | **키 캡슐화(KEM[^a-kem])** — 키 교환 | 격자(모듈 격자) |
| **FIPS 204** | **ML-DSA**[^a-ml-dsa] (CRYSTALS-Dilithium 기반) | **디지털 서명** (주력) | 격자 |
| **FIPS 205** | **SLH-DSA**[^a-slh-dsa] (SPHINCS+ 기반) | **디지털 서명** (해시 기반, 보수적 백업) | 해시 |
| (추가 선정) | **HQC**[^a-hqc] | KEM 백업 (ML-KEM과 다른 수학적 가정) | 코드 기반 |

_표 12-1. NIST PQC 표준. **표준 참조** — 원문은 참조 폴더에 미포함._

### 1.2 O-RAN에서 PQC를 적용해야 할 지점

Ch7 §2에서 정리한 보안 통제를 PQC(Post-Quantum Cryptography) 관점으로 다시 읽으면, 교체 대상이 명확해집니다.

| 대상 | 현재 사용 암호 | PQC 전환 필요 | 근거(본 시리즈) |
|---|---|---|---|
| **E2 인터페이스** — IPsec 터널 모드, DTLS, TLS | 키 교환(ECDH), 서명(ECDSA/RSA) | **ML-KEM + ML-DSA** | Ch7 §2.5[^r6] |
| **A1 / O1 / O2 / F1 / E1** — TLS·IPsec | 동일 | 동일 | Ch7 §2.5, Ch3 §7.2[^r7] |
| **O-FH(Open Fronthaul, 프론트홀)[^a-o-fh]** — MACsec[^a-macsec] 등 | 대칭키는 상대적으로 안전, **키 수립**이 문제 | KEM 교체 | Ch2 §2.3 |
| **Root CA[^a-ca] 인증서 + xApp 인증서** | RSA/ECDSA 서명 | **ML-DSA** (장수명 인증서는 우선순위 높음) | Ch7 §2.1[^r6] |
| **OAuth[^a-oauth] 2.0 액세스 토큰 서명 (JWT[^a-jwt])** | RSA/ECDSA | **ML-DSA** | Ch7 §2.2[^r6] |
| **xApp 패키지 서명** (개발자 서명 → 사업자 서명) | RSA/ECDSA | **ML-DSA / SLH-DSA** | Ch7 §2.1[^r6] |
| **AI[^a-ai] 모델·컨테이너 서명 (AIBOM[^a-aibom])** | 코드 서명 | **ML-DSA / SLH-DSA** — **장기 무결성이 필요** | Ch7 §5.2[^r5] |
| **SUCI**[^a-suci] **생성** (SUPI[^a-supi] 은닉) | ECIES[^a-ecies] 계열 | KEM 기반 재설계 | Ch7 §6.3[^r6] |
| **ZKP[^a-zkp]·HE[^a-he]·SMPC[^a-smpc] 등 PETs[^a-pets]** | 각 스킴의 가정 | **격자 기반 HE는 상대적으로 양자 저항** / ZKP 기반 가정 재검토 필요 | Ch9 §3, §6[^r5] |

> **우선순위 판단 기준**: ① **데이터 수명이 긴 것 먼저** (가입자 식별자·위치 → SUCI, UE-NIB[^a-ue-nib] 관련 통신) ② **서명 수명이 긴 것 먼저** (Root CA, 모델 서명 — 한 번 서명하면 수년간 신뢰됨) ③ **교체가 어려운 것 먼저** (하드웨어에 박힌 키, 장기 배치 O-RU).
{: .prompt-tip }

### 1.3 전환 전략 — 암호 민첩성(crypto agility)

PQC 전환의 실무 원칙은 "알고리즘을 바꾸는 것"이 아니라 **"알고리즘을 바꿀 수 있게 만드는 것"** 입니다.

| 전략 | 내용 | O-RAN 적용 |
|---|---|---|
| **하이브리드 키 교환** | 고전 알고리즘 + PQC를 함께 사용해, 어느 한쪽이 깨져도 안전 | TLS 1.3 하이브리드 그룹 |
| **암호 민첩성** | 알고리즘을 **설정으로 교체 가능**하게 설계, 하드코딩 금지 | Ch3 §7.2의 경고와 직결 — *"보안 프로토콜의 복잡성이 오설정 취약성을 만든다"*[^r7] |
| **인벤토리 먼저** | 어디서 어떤 암호를 쓰는지 목록화 (CBOM[^a-cbom], Cryptographic BOM) | **AIBOM과 함께 관리**(Ch7 §5.2) |
| **키 수명주기** | 생성·저장·**순환(rotation)**·**폐기(revocation)** 전 과정 | Ch7 §1.2에서 지목한 미해결 과제[^r6] |

> **Ch7 §1.2를 다시 읽어야 할 지점입니다.** Soltani 등[^r6]은 O-RAN WG11 권고의 남은 과제로 *"키 생성·저장·순환·폐기를 포함하는 **보호된 키 관리 암호체계**"* 를 명시했습니다. **키 관리가 안 되는 조직은 PQC 전환도 할 수 없습니다** — 알고리즘을 바꾸려면 키를 순환시킬 수 있어야 하기 때문입니다.
{: .prompt-danger }

### 1.4 전환 일정 (표준 참조)

NIST(National Institute of Standards and Technology)는 고전 공개키 암호의 단계적 폐지 일정을 제시했습니다 — **112비트 보안 강도 알고리즘을 2030년까지 deprecated, 2035년까지 disallowed** 로 하는 방향입니다(NIST IR 8547). 6G 상용화 목표(2030년경)와 겹치므로, **6G AI-RAN은 처음부터 PQC를 전제로 설계되어야 합니다.**

| 시점 | 사건 |
|---|---|
| 2024 | **NIST FIPS 203/204/205 공표** |
| 2025 | **HQC 추가 선정** · 하이브리드 키 교환 배포 시작 |
| 2026~2029 | **인벤토리(CBOM) 작성 · 암호 민첩성 확보 · 장수명 서명 우선 전환** |
| **2030** | **6G 상용화 목표** · 112비트 고전 알고리즘 **deprecated** |
| **2035** | 112비트 고전 알고리즘 **disallowed** |

_표 12-2. PQC 전환과 6G 타임라인. **표준 참조** — 실제 적용 시 원문 확인 필요._

---

## 2. 양자 ML(QML) 보안 — 6G 인에이블러의 새 공격면

PQC가 "양자컴퓨터로부터 암호를 지키는" 문제라면, QML(Quantum Machine Learning)[^a-qml] 보안은 **"양자 ML(Machine Learning)[^a-ml] 자체를 지키는"** 문제입니다. Benzaïd 등[^r5]은 이를 미래 연구 방향의 한 축으로 제시합니다.

### 2.1 QML의 약속

> **QML**(Quantum Machine Learning)은 양자 컴퓨팅의 계산력과 ML의 예측 능력을 결합한 것으로, **핵심 6G 인에이블러**로 간주되며 무선 통신을 전례 없는 지능·보안 수준으로 발전시킬 수 있다. 이 약속은 QML이 **양자 병렬성과 얽힘(entanglement)** 을 활용하여 **학습 과정의 지수적 가속**, 방대하고 복잡한 데이터셋의 효과적 처리, 고전 ML의 능력을 넘는 성능을 가능하게 하는 데 기인한다.[^r5]
{: .prompt-info }

### 2.2 O-RAN 보안에 QML을 적용할 유망 방향

Benzaïd 등[^r5]이 제시한 방향:

| 방향 | 내용 |
|---|---|
| **실시간 침입 탐지** | 악성 트래픽 패턴 식별 |
| **행동 이상 탐지** | **고차원 네트워크 텔레메트리의 미묘한 편차** 발견 |
| **체계적 취약점 테스팅** | O-RAN 구성요소(xApp·rApp)와 통신 프로토콜의 보안 약점 식별 |

**제약 조건**: 제안 솔루션은 현재 **NISQ**(Noisy Intermediate-Scale Quantum)[^a-nisq] 장치의 **잡음 취약성**을 다뤄야 합니다[^r5].

> **Quantum Federated Learning이 유망한 이유**[^r5]: *"**축소된 회로 깊이(circuit depth)와 분산 학습 구조** 덕분에 NISQ 장치에서 향상된 수렴 속도, 통신 효율, 학습 정확도, **내재적 잡음 복원력**을 달성하는 실행 가능한 접근으로 인식된다."*
> Ch9의 FL(Federated Learning)[^a-fl] 프라이버시 논의(§2)와 결합하면, **Quantum FL은 프라이버시 + NISQ 잡음 대응을 동시에** 노리는 방향입니다.
{: .prompt-tip }

### 2.3 QML도 적대적 공격에 취약하다

> **고전 ML과 마찬가지로 QML 패러다임도 적대적 공격에 취약**하다. 관심이 커지고 있지만 **QML 보안 연구는 아직 초기 단계(in its infancy)** 다. 따라서 QML 특화 적대적 공격과 그것이 O-RAN 환경에서 야기할 수 있는 위험에 대한 심층적 이해를 개발하기 위한 추가 연구가 필요하다.[^r5]
{: .prompt-warning }

**이미 실증된 공격**:

> 최근 연구는 이 방향의 첫 단계로, **QML 기반 간섭 분류기 xApp이 양자 특화(quantum-tailored) 그래디언트 기반 오염 공격에 의해 속을 수 있음**을 보였다.[^r5]
{: .prompt-danger }

> **Ch6 §1과 정확히 같은 표적입니다.** 고전 InterClass xApp이 FGSM(Fast Gradient Sign Method)[^a-fgsm]/PGD(Projected Gradient Descent)[^a-pgd]로 정확도 0%가 된 것(Ch6 §4.1)과, QML 기반 간섭 분류기 xApp이 양자 특화 오염 공격에 속는 것은 **동일한 구조의 문제**입니다. **알고리즘을 양자로 바꾸는 것은 적대적 취약성을 해결하지 않습니다.**
{: .prompt-danger }

### 2.4 양자 방어 기법

Benzaïd 등[^r5]이 지목한 대응 후보:

| 기법 | 고전 대응물 (Ch9) |
|---|---|
| **Quantum Differential Privacy** | DP[^a-dp] |
| **Quantum Homomorphic Encryption** | HE |
| **Blind Quantum Computation** | TEE[^a-tee] / Confidential Computing |

즉 **Ch9의 PETs 지형이 양자 버전으로 그대로 재현**됩니다. 이는 Ch9 §8의 결론 — "은탄환은 없고 다면적 접근이 필요하다" — 이 양자 시대에도 유효하다는 뜻입니다.

---

## 3. 표준화 지형

### 3.1 기구별 역할 정리

| 기구 | 관련 산출물 | 역할 |
|---|---|---|
| **3GPP** | **TS 33.501**[^ts33501] (5G 시스템 보안 아키텍처·절차), **SCAS**(Security Assurance Specifications)[^a-scas], **IAADE 모델**[^a-iaade] | 셀룰러 보안 기반 표준. SUPI/SUCI 등 식별자 보호[^r6]. **§3.1.1에서 원문 그림으로 상세** |
| **O-RAN Alliance WG11** | **Threat Modeling & Risk Assessment**[^wg11threat], **Security Requirements & Controls Spec**[^wg11secreq], Security Test Specifications, **ZTA for Secure O-RAN**[^oranzt2], **Shared O-RU Security Analysis**[^wg11oru], AI/ML 위협 모델·위험 평가 | O-RAN 특화 보안. **RIC[^a-ric]·xApp의 인증·암호화 절차 의무화**[^r6], AI/ML 위협 모델 제공[^r5] |
| **ETSI** | **TR 104 106**[^etsithreat2] (= O-RAN WG11 Threat Modeling PAS), **TS 104 104**[^etsisecreq2] (= O-RAN WG11 SecReqSpecs PAS) | O-RAN 규격을 **PAS**로 공개 배포 |
| **NIST** | **SP[^a-sp] 800-207**[^nistzt2] (Zero Trust Architecture), **AI 100-2** (적대적 ML 분류체계), **FIPS 203/204/205** (PQC), **IR 8547** (PQC 전환) | ZTA[^a-zta]·적대적 ML·PQC의 참조 프레임 |
| **EU**[^a-eu] | **EU AI Act** | 통신 등 핵심 인프라 AI를 **고위험**으로 분류, **강건성·투명성·설명가능성·프라이버시 보존** 요구[^r5] |
| **AI-RAN Alliance** | (규격 미제작) | 3GPP·O-RAN 규격에 **영향**을 주는 방식으로 작동[^r2] |
| **O-RAN nGRG(next Generation Research Group)[^a-ngrg]** | 6G 연구 방향 설문 보고서 | **Native AI RAN / Cloud-friendly RAN / Service-based RAN** 3대 방향 도출[^r28] |

_표 12-3. 표준화 기구와 산출물._

### 3.1.1 3GPP TS 33.501 — 무엇을 PQC로 바꿔야 하는가의 지도

§1.2에서 "PQC 전환 대상"을 표로 나열했습니다. 그 대상의 **실제 위치**는 3GPP(3rd Generation Partnership Project) TS 33.501[^ts33501]의 보안 아키텍처와 키 계층에 그려져 있습니다.

![Overview of the security architecture (출처: 3GPP TS 33.501[^ts33501], Fig. 4-1)](/assets/img/posts/6g-ai-ran/ts33501-fig4.png)
_그림 12-1. **5G 시스템 보안 아키텍처 개요**. 네트워크 접속 보안·도메인 보안·사용자 도메인 보안 등 계층이 구분됩니다. 출처: [^ts33501], Fig. 4-1._

![Key hierarchy generation in 5GS (출처: 3GPP TS 33.501[^ts33501], Fig. 6.2.1-1)](/assets/img/posts/6g-ai-ran/ts33501-fig6-p60.png)
_그림 12-2. **5GS의 키 계층 생성**. PQC 전환에서 **"어느 키가 어느 키에서 파생되는가"** 를 아는 것이 결정적입니다 — 상위 키의 수립 방식(공개키 기반)이 바뀌면 하위 파생 전체가 영향을 받습니다. 출처: [^ts33501], Fig. 6.2.1-1._

![Key distribution and key derivation scheme for 5G for network nodes (출처: 3GPP TS 33.501[^ts33501], Fig. 6.2.2-1)](/assets/img/posts/6g-ai-ran/ts33501-fig6-p63.png)
_그림 12-3. **네트워크 노드용 5G 키 배포·파생 체계**. gNB[^a-gnb](즉 O-RAN의 O-CU[^a-ocu]/O-DU[^a-odu])까지 키가 내려오는 경로입니다. 출처: [^ts33501], Fig. 6.2.2-1._

![Subscription identifier query (출처: 3GPP TS 33.501[^ts33501], Fig. 6.12.4-1)](/assets/img/posts/6g-ai-ran/ts33501-fig6-p115.png)
_그림 12-4. **가입 식별자 질의**. Ch7 §9.3의 **SUPI/SUCI** 은닉 메커니즘과 연결되며, §1.2에서 **SUCI 생성을 KEM 기반으로 재설계**해야 한다고 지목한 지점입니다. 출처: [^ts33501], Fig. 6.12.4-1._

**서비스 기반 인가 — CoreScan이 검증한 그 절차**

Ch10 §1.3의 CoreScan[^r33]이 정형 분석한 대상이 바로 다음 절차들입니다. 즉 **"표준 원문의 이 그림이 정형 검증되어 5개의 취약점 계열이 나왔다"** 는 관계로 읽어야 합니다.

![NF Service Consumer obtaining access token before NF Service access (출처: 3GPP TS 33.501[^ts33501], Fig. 13.4.1.1.2-1)](/assets/img/posts/6g-ai-ran/ts33501-fig13-p195.png)
_그림 12-5. **NF[^a-nf] 서비스 소비자가 서비스 접근 전에 액세스 토큰을 획득**하는 절차. OAuth 2.0 기반이며, Ch7 §5.2의 RIC OAuth 흐름과 동일 계열입니다. 출처: [^ts33501], Fig. 13.4.1.1.2-1._

![Authorization and service invocation procedure for indirect communication (출처: 3GPP TS 33.501[^ts33501], Fig. 13.4.1.3.1.1-1)](/assets/img/posts/6g-ai-ran/ts33501-fig13-p203.png)
_그림 12-6. **간접 통신에서의 인가·서비스 호출 절차**. CoreScan이 *"기존 과권한 취약점이 **간접 통신과 로밍으로 확장된다**"* 고 밝힌 대상이 이 절차입니다[^r33]. 출처: [^ts33501], Fig. 13.4.1.3.1.1-1._

![Security capability negotiation (출처: 3GPP TS 33.501[^ts33501], Fig. 13.5-1)](/assets/img/posts/6g-ai-ran/ts33501-fig13-p207.png)
_그림 12-7. **보안 능력 협상(security capability negotiation)**. **암호 민첩성(crypto agility, §1.3)이 표준에 이미 반영된 지점**입니다 — PQC 전환은 새 협상 항목을 추가하는 방식으로 진행됩니다. 출처: [^ts33501], Fig. 13.5-1._

> **정리**: PQC 전환은 "TLS 설정 한 줄"이 아닙니다. 그림 12-2~12-7의 **키 계층**, 그림 12-4의 **식별자 은닉**, 그림 12-5~12-10의 **토큰 기반 인가**, 그림 12-7의 **능력 협상** — 이 네 곳을 모두 손대야 합니다. 그리고 그 각각이 Ch7의 실전 통제(§5.1 인증서, §5.2 OAuth, §5.5 인터페이스)와 1:1로 대응합니다.
{: .prompt-tip }

> **PQC 원문에 대한 안내**: NIST **FIPS 203/204/205**와 **IR 8547**은 본 시리즈 참조 자료 폴더에 원문이 없습니다(로컬에 보유하지 않음). 따라서 §1의 알고리즘 목록·전환 일정은 **문서 수준 인용**이며, 실제 적용 시 NIST 원문을 확인해야 합니다. 반면 **§3의 O-RAN·3GPP·ETSI 표준은 원문을 근거로 서술**했습니다(각주 참조).
{: .prompt-info }

### 3.2 규제가 기술 요구로 번역되는 경로

**EU AI Act**는 통신 등 핵심 인프라를 보호·관리하는 AI/ML 시스템을 **고위험(high-risk)** 으로 분류하고, 네 가지 요구를 부과합니다[^r5]. 각 요구는 본 시리즈의 구체적 인에이블러로 번역됩니다.

| 규제 요구 | 기술 인에이블러 | 다루는 장 |
|---|---|---|
| **강건성(robustness)** | MTD[^a-mtd], 적대적 학습, 증류 기반 방어 | Ch6, Ch7 |
| **투명성 · 설명가능성** | **XAI**[^a-xai] (LIME[^a-lime]·SHAP[^a-shap], post-hoc·model-agnostic) | Ch7, Ch8, Ch9 |
| **프라이버시 보존** | **PETs** (DP, HE, SMPC, TEE, ZKP, 언러닝, 합성데이터) | Ch9 |
| (파생) **추적성 · 책임성** | **AIBOM**, 컴플라이언스 감사 로그·이력 | Ch7, Ch10 |

_표 12-4. EU AI Act 요구가 방어 인에이블러로 번역되는 경로._

> **실무적 함의**: Ch10 §2.3의 **Compliance Dashboard**가 *"감사·규제 목적의 상세 컴플라이언스 보고서 다운로드"* 와 *"컴플라이언스 이력 로그"* 를 제공하는 이유가 여기 있습니다[^r14]. **기술 설계가 규제 요구에서 역산됩니다.**
{: .prompt-tip }

### 3.3 3GPP IAADE 모델과 에이전틱 AI

Benzaïd 등[^r5]은 ZT(Zero Trust)[^a-zt]-AI Shield의 최종 형태를 3GPP 모델에 정렬시킵니다.

> **LLM 구동 Agentic AI**를 활용하면 ZT-AI Shield 기능을 제공하는 **전문화된 LLM 기반 에이전트들로 구성된 다중 에이전트 시스템**의 분산 오케스트레이션·관리가 가능해지며, 이들이 **조율된 도구 호출과 응답 메커니즘**을 통해 AI/ML 위협에 대응한다. LLM 구동 Agentic AI를 채택하면 ZT-AI Shield가 **3GPP의 IAADE**(Intent, Awareness, Analysis, Decision, Execution) 모델을 준수하며, **적응형 AI/ML 보안 오케스트레이션을 위한 완전 자율적 지각·추론·계획·실행 능력**을 제공한다.[^r5]
{: .prompt-info }

| IAADE 단계 | ZT-AI Shield에서의 역할 | 본 시리즈 연결 |
|---|---|---|
| **Intent** | 자연어 보안 의도 → 정책 변환 | Ch10 §4 패턴 ① Intent Translator |
| **Awareness** | 위협·상태 지각 | Ch8 탐지 |
| **Analysis** | 위협 분석·근본원인 | Ch8 §3 설명가능성 |
| **Decision** | 대응 결정 | Ch7 정책 엔진 |
| **Execution** | 조치 실행 | Ch8 §7 자율 완화 + **Ch10 가드레일** |

---

## 4. 미래 연구 과제

### 4.1 공백 ①: 체인 모델의 AI 보안 (최우선)

Ch6 §6과 Ch10 §7.1에서 인용한 공백입니다. **O-RAN Alliance가 베스트 프랙티스로 권장하는 설계가 미탐구 취약점을 만듭니다.**

| 항목 | 내용[^r5] |
|---|---|
| 왜 체이닝하는가 | **모듈성, 독립적 진화, 모델 재사용** 촉진. 학습 관심사의 분리 — 각 모델이 특정 학습 태스크 담당 |
| 예시 | **RF[^a-rf] 신호강도 예측 모델 + 셀 사용률 예측 모델 → QoE[^a-qoe] 예측 모델** (현재·인접 셀의 미래 QoE KPI[^a-kpi] 예측) |
| 알려진 문제 | 모델 간 의존에서의 **예측 오차 증폭** |
| **미탐구 문제** | **보안 함의** — 한 모델에 대한 적대적 공격의 **연쇄 효과(cascading effect)** |
| 필요 연구 | ① **danger-risk-consequence 프레임워크 기반 정형 방법론** — 상호의존성 체계적 모델링, 연쇄 효과 분석, **위험 전파 동역학 특성화** ② **인과 귀인 + XAI 시너지** — 체인을 관통하는 **인과 결정 경로 추적**, 종단간 투명성·반사실 분석 |

### 4.2 공백 ②: QML 보안 (§2 참조)

### 4.3 공백 ③: LLM과 O-RAN 보안

Benzaïd 등[^r5]은 **"LLM의 힘을 O-RAN 보안 강화에 활용하는 것은 여전히 미탐구 영역"** 이라고 진단하며, 다음 방향을 제시합니다.

| 방향 | 내용 |
|---|---|
| **적응적·선제적 보안 전략** | 위협이 **현실화되기 전에** 예측·대응 |
| **고급 모니터링** | 특정 위협 벡터에 관련된 보안 데이터를 선별 수집 |
| **자동 취약점 스캐닝** | **xApp/rApp의 소프트웨어·설정 취약점** 자동 스캔 (→ Ch10 자율 컴플라이언스로 일부 실현[^r14]) |
| **컨텍스트 인식 IDS**[^a-ids] | 컨텍스트를 이해하는 이상·침입 탐지 |
| **AI 보안 강화** | **더 현실적이고 O-RAN 특화된 적대적 공격 생성** → 선제적 방어 구축 |
| **AIBOM·XAI 보고서 zero-touch 관리** | 자동·프라이버시 보존적 생성·갱신 + 심층 분석으로 취약점 통찰 추출 |
| **ML 테스트 생성** | **유효하고 의미적으로 일관되며 도메인 맞춤형** 테스트 생성 |
| **LLM(Large Language Model)[^a-llm] + 디지털 트윈 = 가상 AI 레드팀** | 통제된 디지털 복제본 안에서 적대적 시나리오 체계적 생성, 공격 경로 예측, 자동 위협 시뮬레이션으로 방어 지속 검증 |
| **Intent-based AI Security** | **자연어로 표현된 AI 강건성·프라이버시 의도를 실행 가능한 보안 정책·설정으로 자동 번역**. 예: 자원 제약·SLA[^a-sla]에 따라 **최적 PET 선택**, 데이터 특성·모델 구조·프라이버시 요구를 분석해 **PET을 적용할 데이터·모델 파라미터 결정** |
| **Agentic AI 기반 ZT-AI Shield** | §3.3의 IAADE 정렬 |

**그러나 LLM 통합의 과제도 명시됩니다**[^r5]: **설명가능성, 프라이버시 유출, 적대적 공격 취약성, 환각 경향** — 즉 Ch5·Ch10에서 다룬 문제들입니다.

### 4.4 공백 ④: 아키텍처·오케스트레이션 (Polese 등)

Ch2 §6에서 정리한 미해결 과제를 재확인합니다[^r2].

| 범주 | 미해결 질문 |
|---|---|
| 아키텍처 | 효과적 제어·데이터 노출 추상화? 오케스트레이터 호스트? 계층적 접근 필요성? |
| 워크플로 | 오케스트레이터에 적합한 **KPM**[^a-kpm]? O1·O2·E2·A1의 필요 수정? |
| 오케스트레이션 | 이종 가속기·노드 처리능력 차이 → 워크로드 분배 비효율. **대규모 학습의 지연 스파이크** |
| 자원 할당 | 이해관계자 **직교 요구 조화**, 사이트·시간별 **SLA** 반영 |
| 수익화·에너지 | RAN 에너지가 최대 OPEX[^a-opex] → 에너지 효율적 AI 가속. **EDoS**[^a-edos] |
| **보안·프라이버시·접근관리** | 오케스트레이터 API[^a-api]와 파이프라인이 **보안 역할·권한을 추적·설명**할 수 있어야 함 |

### 4.5 공백 ⑤: AI-native RAN 아키텍처 (Salmi 등, ETRI)

| 출처 | 남은 과제 |
|---|---|
| Salmi 등[^r4] | **CME**(상충 관리 엔진)의 강건화, **CSL**(컨텍스트 공유 계층) 구현, **LLM 통합과 결합 최적화 전략 탐구** |
| ETRI[^r28] | Native AI 아키텍처는 **하나로 수렴하지 않고 여러 솔루션이 있을 수 있음**. **Cross-Domain AI 협업**(RAN-UE, RAN-CN[^a-cn]), **AI 서비스 QoS[^a-qos] 보장** |
| ETRI[^r29] | dApp/xApp/rApp 계층 제어, **디지털 트윈·ISAC[^a-isac] 응용**, PHY[^a-phy] 수신기 AI 적용과 **에너지 효율**, **XAI·GenAI[^a-genai]·LLM 기반 RAN 지능화** |

![LLM 기반 의도 주도·적응형 RAN 제어 추론 아키텍처 (출처: Salmi 등[^r4], Fig. 8)](/assets/img/posts/6g-ai-ran/ainative-fig8.png)
_그림 12-8. **LLM 기반 의도 주도 RAN 제어 추론 아키텍처** — 데이터 수집, 메모리, 의미 추론, 출력 전달을 조율합니다. Ch5의 프롬프트 인젝션·메모리 오염 위협이 이 각 단계에 대응합니다. 출처: [^r4], Fig. 8._

![운영자-챗봇 상호작용 흐름 (출처: Salmi 등[^r4], Fig. 9)](/assets/img/posts/6g-ai-ran/ainative-fig9.png)
_그림 12-9. LLM 기반 아키텍처가 가능하게 하는 **운영자-챗봇 상호작용 흐름**. Ch10 §4의 패턴 ⑤ Network Status Explainer에 해당하며, **민감정보 전달 역할이므로 프라이버시 가드레일이 필요**합니다. 출처: [^r4], Fig. 9._

![RIS 기반 Open RAN의 폐루프 제어 아키텍처 (출처: Salmi 등[^r4], Fig. 13)](/assets/img/posts/6g-ai-ran/ainative-fig13.png)
_그림 12-10. **DRL[^a-drl] 기반 xApp을 활용한 RIS[^a-ris] 지원 Open RAN 폐루프 제어**. Ch1 §4의 인에이블러 IRS[^a-irs]가 실제 제어 루프에 들어오는 형태이며, **물리계층 조작 표면이 확대**됩니다. 출처: [^r4], Fig. 13._

### 4.6 공백 ⑥ (본 시리즈 관점): 방어 연구의 편중

![문헌에서 채택된 방어 전략 분포 (출처: Benzaïd 등[^r5], Fig. 30)](/assets/img/posts/6g-ai-ran/aisurvey-fig30.png)
_그림 12-11. **5G RAN의 ML 보안 문헌에서 채택된 방어 전략 분포**. 어떤 방어가 과다 연구되고 어떤 것이 비어 있는지 확인할 수 있습니다. 출처: [^r5], Fig. 30._

Benzaïd 등[^r5]의 lessons learned를 종합하면 편중은 다음과 같습니다.

| 과다 연구 | 과소 연구 |
|---|---|
| **사용자 평면** (D)DoS[^a-dos][^a-ddos] 탐지 | **제어 평면** (D)DoS — *"네트워크 관리·시그널링에 파괴적 영향을 줄 수 있는 제어 평면을 위한 고급 AI 기반 메커니즘 개발에 절실한 연구 필요"* |
| **RRC[^a-rrc] 프로토콜** 퍼징 | **HTTP[^a-http]/HTTPS, eCPRI[^a-ecpri], SCTP[^a-sctp]** 등 O-RAN이 지원하는 다른 프로토콜의 AI 기반 퍼징 |
| AI로 **보안 조치 강화** | AI로 **전통 보안 평가 도구 고도화** (취약점 테스팅·API 보안·상충 완화·Zero Trust에서의 AI는 초기 단계) |
| 단일 모델 적대적 공격 | **체인 모델** 연쇄 효과 |
| 모델 **성능** 모니터링 | 모델 **보안·신뢰성** 모니터링 |
| — | **EDoS**(Economical DoS) 완화를 위한 AI/ML |

> **연구 주제를 찾는다면 오른쪽 열을 보십시오.** 특히 **제어 평면 (D)DoS + 체인 모델 연쇄 효과 + 모델 보안 모니터링 + EDoS** 조합은 여러 공백이 겹치는 지점입니다.
{: .prompt-tip }

---

## 5. 종합 로드맵

### 5.1 시급도 × 난이도 매트릭스

| | **낮은 난이도** | **높은 난이도** |
|---|---|---|
| **높은 시급도** | **① 기본기** — xApp 등록·서명 검증, OAuth 2.0 + RBAC[^a-rbac], **키 관리 수명주기**, A1 스키마·범위 검증, 패치 관리 (Ch7 §2 / V-01·V-06) | **② 구조적 방어** — AIBOM 구축, MTD 도입, TEE 기반 FL, 제어평면 (D)DoS 탐지 (Ch7·Ch9) |
| **낮은 시급도** | **③ 관행 정착** — 벤치마크 리포팅 표준화(FPR[^a-fpr] 포함), 테스트베드·데이터셋 공개 (Ch11 §6) | **④ 장기 연구** — 체인 모델 정형 방법론, QML 보안, LLM+디지털 트윈 가상 레드팀, ML 취약점 스코어링 (Ch10 §7, Ch12 §4) |

### 5.2 단계별 실행 순서

| Phase | 이름 | 할 일 | 근거 장 |
|---|---|---|---|
| **1** | **기본기** | **V-01·V-06 대응** — xApp 서명·등록, OAuth 2.0 + RBAC, **키 수명주기(생성·저장·순환·폐기)**, A1 스키마·범위 검증, 패치 관리 | Ch4, Ch7 |
| **2** | **가시성** | 탐지 계층 배치(**O-DU DT[^a-dt] → Near-RT RIC 앙상블 → 클라우드 DL[^a-dl]/LLM**), XAI 근거 검사, **FPR 모니터링**, **정상 AI-RAN 공존 baseline 측정** | Ch8, Ch11 |
| **3** | **AI 파이프라인 Zero Trust** | **AIBOM** 구축, **MTD** 도입, 위협별 **PETs** 선택, **TEE 기반 FL**, **Model Mon&Test** | Ch7, Ch9 |
| **4** | **자율화 + 검증** | 자율 컴플라이언스(SMO[^a-smo] 배치·O1 강행), **가드레일 3층**(정형 인증·디지털 트윈·온라인 가드), **가드레일 DoS·캘리브레이션 관리** | Ch10 |
| **5** | **양자 대비** | **CBOM 인벤토리**, 암호 민첩성 확보, 하이브리드 → PQC 전환, **QML 보안 연구 추적** | Ch12 |

_표 12-5. 6G AI-RAN 보안 실행 로드맵. 본 시리즈 Ch7·Ch8·Ch9·Ch10·Ch12의 결론을 순서화._

> **순서를 지키는 것이 중요합니다.** Ch4 §5에서 확인했듯 **발생 가능성이 High인 것은 V-01(미인증 무선자원 접근)과 V-06(RIC 하드닝 부족)** 입니다. Phase 1을 건너뛰고 Phase 3~5로 가는 것은 **자물쇠를 안 채운 문에 알람을 설치하는 것**과 같습니다.
{: .prompt-danger }

---

## 6. 시리즈 마무리 — 12개 장을 한 문장씩

| 장 | 한 문장 요약 |
|---|---|
| **Ch1** | 6G의 AI는 Add-On이 아니라 Native여야 하고, AI-RAN은 **for / and / on** 세 기둥이며, **AI and RAN(자원 공유)** 이 보안의 분기점이다. |
| **Ch2** | RU[^a-ru]/DU[^a-du]/CU[^a-cu]는 **프로토콜 스택을 어디서 자르는가**의 문제이고(Option 7-2x + Option 2), 인터페이스의 수가 곧 공격 표면의 수다. RIC는 SDN[^a-sdn] 컨트롤러의 후손이다. |
| **Ch3** | AI/ML 워크플로우는 6단계 순환이고, **같은 인터페이스가 관측과 제어를 동시에** 담당하며, 오설정은 rogue 구성요소 도입의 문을 연다. |
| **Ch4** | 위협은 **6차원 + 7전략**으로 기술하고, 표준의 개방성이 gray-box 지식을 무료로 준다. **V-01·V-06이 가장 발생 가능성 높다.** |
| **Ch5** | 전역 뷰를 오염시키면 망이 흔들린다. **BMP는 침해 호스트 2대로** 처리량을 0으로 만들고, LLM 에이전트 제어 루프는 새 위협 계열을 들여온다. |
| **Ch6** | 악성 xApp 하나로 **정확도 0%**가 된다. ML 위협은 9범주이며, **하드웨어 트로이목마는 소프트웨어로 막을 수 없다.** |
| **Ch7** | 위치는 신뢰의 근거가 아니다. **실전 통제 5종**(등록·OAuth·RBAC·SW개발·인터페이스)이 먼저이고, AI 파이프라인에는 **ZT-AI Shield 6 인에이블러**가 필요하다. |
| **Ch8** | 탐지는 **시간 예산과 설명가능성**을 통과해야 의미가 있다. DT는 0.04 ms, LSTM[^a-lstm]은 2,268 ms — 계층 배치가 유일한 해답이다. |
| **Ch9** | FL은 조건부 프라이버시다. **TEE만이 "사용 중" 데이터를 지키고**, DP는 프라이버시를 위해 강건성을 내놓는다. XAI는 양날의 검이다. |
| **Ch10** | 자율 결정에는 **3층 검증**이 필요하다. 자율 컴플라이언스는 실현 가능하지만 정확도 0.75이므로 **인간 폴백이 설계에 포함**되어야 하고, **가드레일 자체가 DoS 표면**이다. |
| **Ch11** | 재현 가능성이 신뢰성이다. **FPR과 두 개의 기준선**(무공격 + 정상 공존)을 반드시 보고해야 하며, ML에는 아직 **CVSS[^a-cvss]가 없다**. |
| **Ch12** | PQC는 **키 관리 역량의 문제**이고, 양자 ML도 적대적 공격에 취약하며, 규제(EU AI Act)가 기술 요구로 역산된다. **기본기 → 가시성 → 파이프라인 ZT → 자율화 → 양자 대비** 순서를 지켜야 한다. |

### 마지막 확인 체크리스트

- [ ] PQC 전환 대상 중 자신의 시스템에서 **가장 우선순위 높은 항목**을 지목할 수 있는가
- [ ] "PQC 전환은 키 관리 역량의 문제"라는 말의 의미를 설명할 수 있는가
- [ ] QML로 바꿔도 적대적 취약성이 해결되지 않는 이유를 설명할 수 있는가
- [ ] EU AI Act의 4가지 요구가 어떤 기술 인에이블러로 번역되는지 매핑할 수 있는가
- [ ] 자신의 연구·구축 계획을 §5.2 5단계 로드맵 위에 배치할 수 있는가
- [ ] §4.6의 "과소 연구" 열에서 자신이 착수할 주제를 하나 고를 수 있는가

**시리즈 처음으로**: [00. 시리즈 개요 — 6G AI-RAN과 보안 지형도](/posts/airan-00-overview/)

---

### 약어

[^a-oran]: **O-RAN**(Open Radio Access Network): 기지국 구성요소 간 인터페이스를 개방형 표준으로 규정해 다중 벤더 구성을 가능하게 하는 무선 접속망 아키텍처입니다.
[^a-tls]: **TLS**(Transport Layer Security): 통신 구간을 암호화·인증하는 인터넷 표준 보안 프로토콜입니다.
[^a-dtls]: **DTLS**(Datagram Transport Layer Security): TLS를 데이터그램(비연결형) 전송에 맞게 변형한 보안 프로토콜입니다.
[^a-ipsec]: **IPsec**(Internet Protocol Security): IP 계층에서 패킷 단위 암호화·인증을 제공하는 보안 프로토콜 묶음입니다.
[^a-rsa]: **RSA**(Rivest–Shamir–Adleman): 소인수분해의 어려움에 기반한 대표적인 공개키 암호·서명 알고리즘입니다.
[^a-ecdh]: **ECDH**(Elliptic Curve Diffie–Hellman): 타원곡선 이산대수 문제에 기반한 키 합의(키 교환) 알고리즘입니다.
[^a-ecdsa]: **ECDSA**(Elliptic Curve Digital Signature Algorithm): 타원곡선 기반 전자서명 알고리즘입니다.
[^a-ran]: **RAN**(Radio Access Network): 단말과 코어망 사이에서 무선 접속을 담당하는 무선 접속망 구간입니다.
[^a-3gpp]: **3GPP**(3rd Generation Partnership Project): 3G부터 5G·6G까지 이동통신 국제 표준을 제정하는 표준화 협력체입니다.
[^a-ts]: **TS**(Technical Specification): 3GPP·ETSI 등 표준화 기구가 발행하는 기술규격 문서 유형입니다.
[^a-wg]: **WG**(Working Group): 표준화 기구 내부의 작업반을 가리키며, WG11은 O-RAN Alliance의 보안 작업반입니다.
[^a-oru]: **O-RU**(O-RAN Radio Unit): O-RAN에서 무선 신호 송수신과 하위 물리계층 처리를 담당하는 무선 장치입니다.
[^a-etsi]: **ETSI**(European Telecommunications Standards Institute): 유럽전기통신표준협회. 유럽의 정보통신 표준화 기구입니다.
[^a-pas]: **PAS**(Publicly Available Specification): 외부 단체의 규격을 공개 규격 형태로 배포하는 표준 문서 유형입니다.
[^a-tr]: **TR**(Technical Report): 표준화 기구가 발행하는 기술보고서 문서 유형입니다.
[^a-nist]: **NIST**(National Institute of Standards and Technology): 미국 국립표준기술연구소. FIPS·SP 등 암호·보안 표준을 제정합니다.
[^a-fips]: **FIPS**(Federal Information Processing Standards): NIST가 제정하는 미국 연방 정보처리 표준입니다.
[^a-ir]: **IR**(Interagency Report): NIST가 발간하는 기관 간 보고서 문서 유형입니다.
[^a-pqc]: **PQC**(Post-Quantum Cryptography): 양자컴퓨터의 공격에도 안전하도록 설계된 공개키 암호 체계입니다.
[^a-ml-kem]: **ML-KEM**(Module-Lattice-Based Key-Encapsulation Mechanism): 모듈 격자 문제에 기반한 NIST 표준(FIPS 203) 키 캡슐화 알고리즘입니다.
[^a-kem]: **KEM**(Key Encapsulation Mechanism): 공개키로 대칭키를 안전하게 캡슐화해 전달하는 키 수립 방식입니다.
[^a-ml-dsa]: **ML-DSA**(Module-Lattice-Based Digital Signature Algorithm): 모듈 격자 기반의 NIST 표준(FIPS 204) 전자서명 알고리즘입니다.
[^a-slh-dsa]: **SLH-DSA**(Stateless Hash-Based Digital Signature Algorithm): 해시 함수의 안전성에만 의존하는 NIST 표준(FIPS 205) 무상태 전자서명 알고리즘입니다.
[^a-hqc]: **HQC**(Hamming Quasi-Cyclic): 오류정정부호(코드) 문제에 기반한 NIST 선정 백업 KEM 알고리즘입니다.
[^a-o-fh]: **O-FH**(Open Fronthaul): O-RU와 O-DU 사이를 잇는 O-RAN의 개방형 프론트홀 인터페이스입니다.
[^a-macsec]: **MACsec**(MAC Security): 이더넷 링크 계층에서 프레임 단위 암호화·무결성을 제공하는 보안 표준입니다.
[^a-ca]: **CA**(Certificate Authority): 공개키 인증서를 발급·서명하는 신뢰 기관입니다.
[^a-oauth]: **OAuth**(Open Authorization): 자격증명을 직접 공유하지 않고 접근 권한을 위임하는 표준 인가 프레임워크입니다.
[^a-jwt]: **JWT**(JSON Web Token): 권한 정보(클레임)를 서명과 함께 담아 전달하는 토큰 형식입니다.
[^a-ai]: **AI**(Artificial Intelligence): 인간의 학습·추론 능력을 계산 시스템으로 구현하는 인공지능 기술입니다.
[^a-aibom]: **AIBOM**(AI Bill of Materials): AI 모델·데이터·의존성의 구성 내역을 기록한 목록으로, SBOM(Software Bill of Materials)의 AI 확장입니다.
[^a-suci]: **SUCI**(Subscription Concealed Identifier): 가입자 영구 식별자를 암호화해 무선 구간 노출을 막는 은닉 식별자입니다.
[^a-supi]: **SUPI**(Subscription Permanent Identifier): 5G 가입자의 영구 식별자입니다.
[^a-ecies]: **ECIES**(Elliptic Curve Integrated Encryption Scheme): 타원곡선 기반 하이브리드 공개키 암호화 방식으로, SUCI 생성에 사용됩니다.
[^a-zkp]: **ZKP**(Zero-Knowledge Proof): 주장 내용 이외의 정보를 드러내지 않고 그 주장이 참임을 증명하는 영지식 증명 기법입니다.
[^a-he]: **HE**(Homomorphic Encryption): 암호문 상태 그대로 연산할 수 있는 동형암호 기술입니다.
[^a-smpc]: **SMPC**(Secure Multi-Party Computation): 여러 참여자가 각자의 입력을 공개하지 않고 공동 계산을 수행하는 안전한 다자간 계산 기법입니다.
[^a-pets]: **PETs**(Privacy-Enhancing Technologies): 데이터 활용 과정에서 프라이버시를 보호하는 기술들의 총칭입니다.
[^a-ue-nib]: **UE-NIB**(UE Network Information Base): Near-RT RIC이 유지하는 UE(User Equipment, 단말) 관련 네트워크 정보 저장소입니다.
[^a-cbom]: **CBOM**(Cryptographic Bill of Materials): 시스템이 사용하는 암호 알고리즘·키·인증서의 구성 내역 목록입니다.
[^a-qml]: **QML**(Quantum Machine Learning): 양자 컴퓨팅의 계산력을 기계학습에 결합한 양자 기계학습 기술입니다.
[^a-ml]: **ML**(Machine Learning): 데이터로부터 패턴을 학습해 예측·분류를 수행하는 기계학습 기술입니다.
[^a-nisq]: **NISQ**(Noisy Intermediate-Scale Quantum): 오류 보정 없이 동작하는 중간 규모의 현세대 잡음 양자컴퓨터를 가리키는 용어입니다.
[^a-fl]: **FL**(Federated Learning): 원본 데이터를 중앙에 모으지 않고 각 참여자가 로컬 학습 결과만 공유하는 연합학습 기법입니다.
[^a-fgsm]: **FGSM**(Fast Gradient Sign Method): 손실 기울기의 부호 방향으로 입력을 교란하는 대표적인 적대적 예제 생성 기법입니다.
[^a-pgd]: **PGD**(Projected Gradient Descent): FGSM을 반복 적용하면서 교란 범위를 제한하는 더 강력한 적대적 공격 기법입니다.
[^a-dp]: **DP**(Differential Privacy): 개별 데이터의 포함 여부가 분석 결과에 미치는 영향을 수학적으로 제한하는 차분 프라이버시 기법입니다.
[^a-tee]: **TEE**(Trusted Execution Environment): 프로세서 안에 격리된 실행 영역을 만들어 사용 중인 코드·데이터를 보호하는 신뢰 실행 환경입니다.
[^a-scas]: **SCAS**(Security Assurance Specifications): 3GPP의 네트워크 장비 보안 보증 규격입니다.
[^a-iaade]: **IAADE**(Intent, Awareness, Analysis, Decision, Execution): 의도-지각-분석-결정-실행으로 이어지는 3GPP의 자율 네트워크 운영 모델입니다.
[^a-ric]: **RIC**(RAN Intelligent Controller): RAN을 지능적으로 제어·최적화하는 O-RAN의 컨트롤러로, 제어 주기에 따라 Near-RT(준실시간)와 Non-RT(비실시간)로 나뉩니다.
[^a-sp]: **SP**(Special Publication): NIST가 발간하는 특별 간행물 문서 유형입니다.
[^a-zta]: **ZTA**(Zero Trust Architecture): 어떤 주체도 기본적으로 신뢰하지 않고 모든 접근을 검증하는 제로 트러스트 보안 아키텍처입니다.
[^a-eu]: **EU**(European Union): 유럽연합입니다.
[^a-ngrg]: **nGRG**(next Generation Research Group): O-RAN Alliance에서 6G 등 차세대 기술의 연구 방향을 다루는 연구 그룹입니다.
[^a-gnb]: **gNB**(next generation NodeB): 5G 기지국을 가리키는 3GPP 명칭입니다.
[^a-ocu]: **O-CU**(O-RAN Central Unit): O-RAN에서 상위 프로토콜 계층 처리를 담당하는 중앙 장치입니다.
[^a-odu]: **O-DU**(O-RAN Distributed Unit): O-RAN에서 실시간성이 높은 하위 프로토콜 계층 처리를 담당하는 분산 장치입니다.
[^a-nf]: **NF**(Network Function): 5G 코어망을 구성하는 개별 네트워크 기능 단위입니다.
[^a-mtd]: **MTD**(Moving Target Defense): 시스템 구성이나 모델을 주기적으로 변경해 공격자가 표적을 고정하지 못하게 하는 방어 전략입니다.
[^a-xai]: **XAI**(eXplainable AI): AI 모델의 판단 근거를 사람이 이해할 수 있게 설명하는 설명가능 인공지능 기술입니다.
[^a-lime]: **LIME**(Local Interpretable Model-agnostic Explanations): 개별 예측 주변을 국소적으로 근사해 모델 판단을 설명하는 XAI 기법입니다.
[^a-shap]: **SHAP**(SHapley Additive exPlanations): 게임이론의 섀플리 값에 기반해 각 입력 특징의 기여도를 정량화하는 XAI 기법입니다.
[^a-zt]: **ZT**(Zero Trust): 위치나 소속을 근거로 신뢰를 부여하지 않고 모든 접근을 매번 검증하는 보안 원칙입니다.
[^a-rf]: **RF**(Radio Frequency): 무선 통신에 사용되는 무선 주파수 신호·대역을 가리킵니다.
[^a-qoe]: **QoE**(Quality of Experience): 사용자가 체감하는 서비스 품질 지표입니다.
[^a-kpi]: **KPI**(Key Performance Indicator): 시스템·네트워크의 성능을 정량적으로 나타내는 핵심 성과 지표입니다.
[^a-ids]: **IDS**(Intrusion Detection System): 트래픽과 행위를 분석해 침입을 탐지하는 침입탐지시스템입니다.
[^a-llm]: **LLM**(Large Language Model): 방대한 텍스트로 학습되어 자연어 이해·생성 능력을 제공하는 대규모 언어 모델입니다.
[^a-sla]: **SLA**(Service Level Agreement): 서비스 제공자가 보장해야 하는 성능·가용성 수준을 정의한 서비스 수준 협약입니다.
[^a-kpm]: **KPM**(Key Performance Measurement): E2 인터페이스를 통해 수집되는 RAN 핵심 성능 측정 지표(서비스 모델)입니다.
[^a-opex]: **OPEX**(Operating Expenditure): 운영 비용(운영 지출)입니다.
[^a-edos]: **EDoS**(Economical Denial of Service): 종량제 자원을 고갈시켜 비용 폭증을 유발하는 경제적 서비스 거부 공격입니다.
[^a-api]: **API**(Application Programming Interface): 소프트웨어 구성요소 간 기능 호출 규약을 정의한 인터페이스입니다.
[^a-cn]: **CN**(Core Network): RAN 뒤에서 세션·이동성·가입자 관리를 담당하는 코어망입니다.
[^a-qos]: **QoS**(Quality of Service): 네트워크가 보장하는 지연·대역폭 등 전송 품질 지표입니다.
[^a-isac]: **ISAC**(Integrated Sensing and Communication): 통신과 센싱(감지)을 하나의 시스템과 전파 자원으로 통합하는 6G 후보 기술입니다.
[^a-phy]: **PHY**(Physical Layer): 프로토콜 스택의 최하위 물리계층으로, 실제 무선 신호의 송수신을 담당합니다.
[^a-genai]: **GenAI**(Generative AI): 텍스트·이미지 등 새로운 콘텐츠를 생성하는 생성형 인공지능입니다.
[^a-drl]: **DRL**(Deep Reinforcement Learning): 심층 신경망과 강화학습을 결합해 시행착오로 제어 정책을 학습하는 기법입니다.
[^a-ris]: **RIS**(Reconfigurable Intelligent Surface): 전파 반사 특성을 소프트웨어로 제어해 무선 환경을 조정하는 지능형 반사 표면입니다.
[^a-irs]: **IRS**(Intelligent Reflecting Surface): RIS와 같은 개념으로 쓰이는 지능형 반사 표면의 다른 명칭입니다.
[^a-dos]: **DoS**(Denial of Service): 자원을 고갈시키거나 처리를 방해해 정상적인 서비스 제공을 막는 서비스 거부 공격입니다.
[^a-ddos]: **DDoS**(Distributed Denial of Service): 다수의 분산된 호스트가 동시에 트래픽을 보내 서비스를 마비시키는 분산 서비스 거부 공격입니다.
[^a-rrc]: **RRC**(Radio Resource Control): 단말과 기지국 간 연결 설정과 무선 자원 제어를 담당하는 제어 프로토콜입니다.
[^a-http]: **HTTP**(HyperText Transfer Protocol): 웹의 표준 전송 프로토콜이며, HTTPS는 이를 TLS로 암호화한 버전입니다.
[^a-ecpri]: **eCPRI**(enhanced Common Public Radio Interface): 프론트홀 구간의 무선 데이터 전송을 위한 개선된 인터페이스 규격입니다.
[^a-sctp]: **SCTP**(Stream Control Transmission Protocol): 다중 스트림을 지원하는 전송 계층 프로토콜로, 이동통신 시그널링 전송에 널리 쓰입니다.
[^a-rbac]: **RBAC**(Role-Based Access Control): 사용자 개인이 아니라 역할 단위로 권한을 부여하는 역할 기반 접근 제어 모델입니다.
[^a-fpr]: **FPR**(False Positive Rate): 정상을 공격으로 잘못 판정한 비율(오탐률)입니다.
[^a-dt]: **DT**(Decision Tree): 조건 분기 트리로 분류·회귀를 수행하는 경량 기계학습 모델입니다.
[^a-dl]: **DL**(Deep Learning): 다층 신경망으로 데이터의 표현을 학습하는 딥러닝 기술입니다.
[^a-smo]: **SMO**(Service Management and Orchestration): O-RAN에서 RAN 요소의 관리와 오케스트레이션을 총괄하는 프레임워크입니다.
[^a-ru]: **RU**(Radio Unit): 기지국에서 무선 신호 송수신을 담당하는 무선 장치입니다.
[^a-du]: **DU**(Distributed Unit): 기지국에서 실시간성이 높은 하위 프로토콜 계층 처리를 담당하는 분산 장치입니다.
[^a-cu]: **CU**(Central Unit): 기지국에서 상위 프로토콜 계층 처리를 담당하는 중앙 장치입니다.
[^a-sdn]: **SDN**(Software-Defined Networking): 네트워크의 제어 평면을 데이터 평면에서 분리해 소프트웨어로 중앙 제어하는 네트워킹 패러다임입니다.
[^a-lstm]: **LSTM**(Long Short-Term Memory): 장기 의존성을 학습할 수 있는 순환 신경망 구조입니다.
[^a-cvss]: **CVSS**(Common Vulnerability Scoring System): 취약점의 심각도를 0~10 점수로 정량화하는 공통 취약점 평가 체계입니다.

## References

[^r2]: M. Polese, N. Mohamadi, S. D'Oro, L. Bonati, and T. Melodia, "Beyond connectivity: An open architecture for AI-RAN convergence in 6G," *arXiv preprint* arXiv:2507.06911v2, Dec. 2025.
[^r4]: S. Salmi, M. A. Ouameur, M. Bagaa, G. C. Alexandropoulos, A. Tahenni, D. Massicotte, and A. Ksentini, "AI-native O-RAN architectures for 6G: Towards real-time adaptation, conflict resolution, and efficient resource management," *TechRxiv preprint*, Sep. 2025.
[^r5]: C. Benzaïd, T. Taleb, R. Cavalcanti, and A. Sánchez, "AI in Open RAN security: Enabler and threat — A comprehensive survey and future perspectives," *TechRxiv preprint*, Dec. 2025.
[^r6]: S. Soltani, A. Amanloo, M. Shojafar, and R. Tafazolli, "Intelligent control in 6G open RAN: Security risk or opportunity?" *IEEE Open Journal of the Communications Society*, 2025.
[^r7]: N. M. Yungaicela-Naula, V. Sharma, and S. Scott-Hayward, "Misconfiguration in O-RAN: Analysis of the impact of AI/ML," *Computer Networks*, p. 110455, 2024.
[^r14]: S. Chatzimiltis, M. B. Mashhadi, M. Shojafar, M. Debbah, and R. Tafazolli, "Agentic AI for 6G: A new paradigm for autonomous RAN security compliance," *arXiv preprint* arXiv:2512.12400v2, Apr. 2026.
[^r28]: 권동승, 나지현, "O-RAN에서 6G RAN 연구 방향," *전자통신동향분석*, vol. 40, no. 5, pp. 101–112, Oct. 2025.
[^r29]: 김민건, 김준우, 이훈, 배정숙, 김일규, "6G 무선접속망을 위한 AI/ML 기반 지능형 RAN 기술 동향," *전자통신동향분석*, vol. 41, no. 1, pp. 1–10, Feb. 2026.

### 표준 문서 — 참조 폴더에 원문 보유

[^ts33501]: 3GPP, *Security Architecture and Procedures for 5G System*, TS 33.501.
[^wg11threat]: O-RAN ALLIANCE WG11, *O-RAN Security Threat Modeling and Risk Assessment*, O-RAN.WG11.Threat-Modeling.O-R003-v03.00.
[^etsithreat2]: ETSI, *Publicly Available Specification (PAS); O-RAN Security Threat Modeling and Risk Assessment*, ETSI TR 104 106 V3.0.0, Jun. 2025.
[^wg11secreq]: O-RAN ALLIANCE WG11, *O-RAN Security Requirements and Controls Specifications*, O-RAN.WG11.SecReqSpecs-R003-v09.01.
[^etsisecreq2]: ETSI, *Publicly Available Specification (PAS); O-RAN Security Requirements and Controls Specifications*, ETSI TS 104 104 V9.1.0, Jun. 2025.
[^oranzt2]: O-RAN ALLIANCE, *Zero Trust Architecture (ZTA) for Secure O-RAN*, White Paper O-RAN.WP.ZTA-for-secure-O-RAN-v1.0, 2024.
[^wg11oru]: O-RAN ALLIANCE WG11, *Shared O-RU Security Analysis*, TR.O-R004-v06.00, 2025.
[^nistzt2]: S. Rose, O. Borchert, S. Mitchell, and S. Connelly, *Zero Trust Architecture*, NIST SP 800-207, Aug. 2020, doi: 10.6028/NIST.SP.800-207.
[^r33]: M. Akon, M. Toufikuzzaman, and S. R. Hussain, "From control to chaos: A comprehensive formal analysis of 5G's access control," in *Proc. 2025 IEEE Symposium on Security and Privacy (SP)*, 2025, pp. 1043–1062.

### 표준 문서 — 원문 미보유 (문서 수준 인용)

- National Institute of Standards and Technology, *Module-Lattice-Based Key-Encapsulation Mechanism Standard*, FIPS 203, Aug. 2024.
- National Institute of Standards and Technology, *Module-Lattice-Based Digital Signature Standard*, FIPS 204, Aug. 2024.
- National Institute of Standards and Technology, *Stateless Hash-Based Digital Signature Standard*, FIPS 205, Aug. 2024.
- National Institute of Standards and Technology, *Transition to Post-Quantum Cryptography Standards*, NIST IR 8547 (draft).
- National Institute of Standards and Technology, *Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations*, NIST AI 100-2.
- European Union, *Artificial Intelligence Act* (Regulation (EU) 2024/1689).
