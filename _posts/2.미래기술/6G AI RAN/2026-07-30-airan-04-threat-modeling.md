---
title: "[6G AI-RAN] 04. 6G AI-RAN 위협 모델링 및 공격 표면"
date: 2026-07-30 09:40:00 +0900
categories:
  - 2.미래기술
  - 6G AI RAN
  - Part II 위협·공격벡터
tags:
  - Threat-Modeling
  - Attack-Surface
  - NIST-Taxonomy
  - O-RAN-WG11
  - Adversarial-ML
math: true
mermaid: false
---

# 6G AI-RAN 위협 모델링 및 공격 표면

## 들어가며 — 위협 모델 없이는 방어를 비교할 수 없다

"이 방어가 좋은가?"라는 질문은 **"어떤 공격자를 가정하는가?"** 를 먼저 정하지 않으면 의미가 없습니다. 화이트박스 공격자를 막는 방어와 블랙박스 공격자를 막는 방어는 비교 대상이 아닙니다.

이 장은 6G AI-RAN 위협을 **체계적으로 기술하는 틀**을 세웁니다.

1. 두 층의 공격 표면 — O-RAN(Open Radio Access Network)[^a-o-ran] 인프라 표면 + AI(Artificial Intelligence)[^a-ai]/ML(Machine Learning)[^a-ml] 표면
2. Benzaïd 등의 ML 위협 모델 6차원 (NIST(National Institute of Standards and Technology)[^a-nist] 정렬 + O-RAN 특화)
3. 공격 전략 7분류 (S1~S7)
4. ML 위협 9범주 개요
5. O-RAN 특화 취약점 목록과 **표준 위협 카탈로그**
   - 5.1~5.3 **O-RAN WG11(Working Group 11)[^a-wg] / ETSI(European Telecommunications Standards Institute)[^a-etsi] 위협 모델링 원문** — 인터페이스별 위협, **가상화·클라우드 공격 6종**, 위험 평가 방법론
   - 5.4 **공유 O-RU**(O-RAN Radio Unit)[^a-ru] 멀티테넌트 위협과 PKI(Public Key Infrastructure)[^a-pki]
6. ML 소프트웨어 의존성으로 생기는 표면
7. 위협 모델 작성 실습 템플릿

---

## 1. 두 층의 공격 표면

6G AI-RAN의 공격 표면은 **두 개의 층이 겹쳐 있습니다.**

| 층 | 표면 구성요소 | 대표 침해 결과 |
|---|---|---|
| **Layer 1 — O-RAN 인프라 표면** | 개방 인터페이스 (O-FH[^a-o-fh] · F1 · E1 · E2 · A1 · O1 · O2 · Y1 · R1) | 관측 왜곡 + 제어 탈취 |
| | 가상화·컨테이너 (O-Cloud · Kubernetes) | 컨테이너 탈출, 테넌트 격리 실패 |
| | 제3자 앱 (xApp · rApp · dApp) | 무선자원 무단 제어 |
| | 오픈소스 SW[^a-sw] 스택 | 알려진 취약점 누적 |
| | 멀티벤더 공급망 | rogue 구성요소 도입 |
| **Layer 2 — AI/ML 표면**<br/>(Ch4~Ch6의 주제) | 학습 데이터 | 오염(poisoning) |
| | 모델 파라미터 | 모델 탈취·변조 |
| | 추론 입력 | 회피(evasion) |
| | 모델 배포 경로 / 의존 라이브러리 | ML 공급망 공격 |
| | AI 가속 하드웨어 | 하드웨어 트로이목마 |

_표 4-1. 6G AI-RAN의 2층 공격 표면._

**두 층은 단순히 쌓여 있는 것이 아닙니다.** Layer 1의 침해가 Layer 2의 데이터·모델에 도달하고, 역으로 **Layer 2의 AI 결정이 Layer 1의 인프라를 직접 제어**하므로 — **AI 침해가 곧 인프라 침해**입니다. 이 양방향성이 AI-RAN 위협 모델링의 출발점입니다.

Benzaïd 등[^r5]은 이 중첩을 이렇게 표현합니다.

> O-RAN 아키텍처의 개방성·분해·지능성은 vRAN·C-RAN에는 근본적으로 부재한 고급 보안 능력(예측적 보안 분석, near-RT RIC를 통한 실시간 위협 탐지, 지역화된 분산 엣지 분석, 동적 정책 집행을 갖춘 적응형 방어)을 가능하게 한다. **그러나 O-RAN의 AI 네이티브 지원과 개방 인터페이스는 공격 표면을 확장하여, 벤더 통제 아키텍처에서는 대부분 제약되던 고유한 보안 과제를 만든다. 여기에는 특히 AI/ML 모델을 겨냥한 정교한 적대적 위협과, 제3자 AI 기반 서비스 및 외주 하드웨어 구성요소에서 비롯되는 공급망 위험이 포함된다.**[^r5]
{: .prompt-warning }

### 1.1 개방형 RAN vs 폐쇄형 RAN — AI 보안 관점 비교

| 아키텍처 | AI 통합 수준 | 가능한 AI 보안 기능 | 제약 |
|---|---|---|---|
| **전통 RAN**(Radio Access Network)[^a-ran] (폐쇄) | 최소 | 사실상 없음 — 벤더 독점 정적 보안 통제 | 적응성·투명성 부재 |
| **C-RAN**(Centralized RAN)[^a-c-ran] / **vRAN**(virtualized RAN)[^a-vran] | 기초적 | 기본적 AI/ML 보안 기능 통합 가능 | **벤더 통제 AI 구현 + 독점 인터페이스** → 상호운용성·고급 기능 제약 |
| **O-RAN** | **네이티브** | 예측적 보안 분석, near-RT 위협 탐지, 분산 엣지 분석, 동적 정책 집행 적응형 방어, **제3자 보안 xApp/rApp 통합** | **표면 확장 — 적대적 ML 위협 + 공급망 위험** |

### 1.2 참고: 5G 보안·프라이버시 위협의 전경

AI-RAN 특화 위협 이전에, 5G 자체의 위협 전경을 배경으로 알아 둘 필요가 있습니다.

![5G 보안·프라이버시 위협 개요 (출처: He 등[^r24], Fig. 1)](/assets/img/posts/6g-ai-ran/ai5gsec-fig1.png)
_그림 4-1. 5G 보안·프라이버시 위협 개요. 출처: [^r24], Fig. 1._

---

## 2. ML 위협 모델의 6차원

Benzaïd 등[^r5]은 NIST 적대적 ML 분류체계[^nist]와 정렬하면서, **O-RAN 특화 차원을 추가**한 위협 모델을 제안합니다.

![ML 위협 모델 차원 (출처: Benzaïd 등[^r5], Fig. 4)](/assets/img/posts/6g-ai-ran/aisurvey-fig4.png)
_그림 4-2. **ML 위협 모델 차원**. 좌측: 공격 전략 S1~S7. 중앙: 공격자 능력(C1~C8)과 한계(L1~L3). 우측: 지식 수준(W/G/B/M)과 표적 학습 패러다임(ML/DL[^a-dl], RL[^a-rl]/DRL[^a-drl], FL[^a-fl]). 출처: [^r5], Fig. 4._

### 2.1 차원별 정의

| 차원 | 정의 | 값 |
|---|---|---|
| **Goal (목표)** | 무엇을 깨뜨리려 하는가 | 무결성 / 가용성 / 기밀성(프라이버시) |
| **Knowledge (지식)** | 표적 시스템에 대해 무엇을 아는가 | **W**hite-box / **G**ray-box / **B**lack-box / **M**yopic |
| **Capability (능력)** | 표적 ML 시스템과 어떻게 상호작용할 수 있는가 | **C1~C8** (아래 표) |
| **Limits (한계)** | 외부 제약 | **L1** Label Limit / **L2** Query Limit / **L3** Training Process Limit |
| **Specificity (특정성)** | 표적 지정 여부 | Targeted (특정 샘플) / Indiscriminate (임의 샘플) |
| **Influence (영향 시점)** | ML 파이프라인의 어느 단계 | **Causative**(학습 단계) / **Exploratory**(추론 단계) |
| **Attack Strategy (전략)** | 어떤 기법으로 공격하는가 | **S1~S7** (3절) |

### 2.2 지식 수준 — 4가지 위협 모델

Benzaïd 등[^r5]의 Table III를 재구성했습니다. `M`=모델, `F`=특징집합, `f ⊂ F`=특징 부분집합, `M(F_x)`=모델 출력.

| 위협 모델 | 가용 지식 | 모델 출력 관측 | 현실성 |
|---|---|---|---|
| **White-box** | `M, F` (모델 구조·파라미터 + 전체 특징) | ✓ | 최악 가정. 내부자·오픈소스 모델 |
| **Gray-box** | `f` (특징 부분집합) | ✓ | 현실적. 문서·규격에서 특징 추론 가능 |
| **Black-box** | ✗ | ✓ | 가장 현실적. 쿼리만 가능 |
| **Myopic** | `f` (특징 부분집합) | **✗** | **특징 일부는 알지만 모델·학습데이터·출력을 모름** |

> **Myopic 모델이 O-RAN에서 특히 중요한 이유**: 공격자가 E2 KPM(Key Performance Measurement)[^a-kpm] 규격(공개 표준!)으로 특징 집합의 일부를 알 수 있지만, xApp의 모델 출력은 관측할 수 없는 상황이 매우 흔합니다. **표준의 개방성이 곧 gray-box/myopic 지식을 무료로 제공합니다.**
{: .prompt-danger }

### 2.3 능력(Capability) — O-RAN 특화 확장

Benzaïd 등[^r5]이 NIST의 "capability" 차원을 **적대적 통제(control)** 와 **운영적 한계(limit)** 로 분리하고, 차세대 네트워크 특화 능력을 추가한 것이 이 분류의 기여입니다.

| 코드 | 능력 | O-RAN에서의 구체적 의미 |
|---|---|---|
| **C1** | Training Data Control | 학습 데이터 오염 — E2 보고 조작, 데이터 레이크 침해 |
| **C2** | Inference Data Control | 추론 입력 조작 — 실시간 KPM 왜곡 |
| **C3** | Model Control | 모델 파라미터 직접 변경 — 카탈로그/배포 경로 침해 |
| **C4** | Query Access | 모델을 오라클로 사용 — Y1 분석 데이터 소비, xApp API[^a-api] |
| **C5** | Software Control | ML 알고리즘 소스코드 통제 — **오픈소스 의존성** |
| **C6** | **Hardware Control** | **AI 가속기(GPU[^a-gpu]) 하드웨어 통제** — 하드웨어 트로이목마 |
| **C7** | **Comm. Channel Control** | **무선 채널 통제 — 재밍, 신호 조작** |
| **C8** | **Behavior Access** | **행동 관측·모방 — guessing 공격** |
| L1 | Label Limit | 라벨 오염 예산 제한 |
| L2 | Query Limit | 쿼리 횟수 제한 |
| L3 | Training Process Limit | 학습 프로세스 개입 제한 |

> **C6·C7·C8이 이 분류의 핵심 기여**입니다. 일반 ML 보안 분류에는 없는 차원인데, 무선 네트워크에서는 결정적입니다. 재밍(C7)은 학습·추론 데이터를 **물리적으로** 조작하는 능력이고, 이는 소프트웨어 방어로 막을 수 없습니다.
{: .prompt-tip }

---

## 3. 공격 전략 7분류 (S1~S7)

Benzaïd 등[^r5]의 **"method-based, domain-aware"** 접근의 핵심입니다. "무엇을 깨뜨리는가"가 아니라 **"어떻게 공격을 성립시키는가"** 로 분류합니다.

![ML 시스템 공격 전략의 분류 (출처: Benzaïd 등[^r5], Fig. 5)](/assets/img/posts/6g-ai-ran/aisurvey-fig5.png)
_그림 4-3. 공격 전략 분류 — 각 전략에 라벨(S1~S7)이 부여됩니다. 출처: [^r5], Fig. 5._

| 코드 | 전략 | 설명 | O-RAN 예시 |
|---|---|---|---|
| **S1** | **Data/Model Manipulation** | 데이터 또는 모델 파라미터를 직접 조작 | 라벨 플리핑, E2 보고 위조 |
| **S2** | **Optimization-driven Methods** | 최적화로 최소 섭동 계산 (FGSM[^a-fgsm], PGD[^a-pgd], C&W[^a-cw]) | xApp 입력 KPM에 최적 섭동 주입 |
| **S3** | **Model-Probing Exploitation** | 모델을 반복 질의해 정보 추출 | 모델 탈취, 멤버십 추론 |
| **S4** | **Learning-based Methods** | 대리 모델(surrogate)/GAN[^a-gan]을 학습해 공격 생성 | GAN으로 정상처럼 보이는 공격 트래픽 합성 |
| **S5** | **Out-of-Band Info. Exploitation** | 대역 외 정보 활용 | 사이드채널, 전력·타이밍 관측 |
| **S6** | **Temporal/Strategic Methods** | 타이밍을 전략적으로 선택 | **재학습 시점을 노린 오염**, 임계값 서서히 이동 |
| **S7** | **Software/Hardware Compromise** | SW/HW[^a-hw] 침해 | 의존 라이브러리 트로이목마, 하드웨어 트로이목마 |

> **S6(시간적/전략적 방법)** 은 실무에서 가장 놓치기 쉽습니다. Soltani 등[^r6]이 보고한 SDN(Software-Defined Networking)[^a-sdn]/RIC(RAN Intelligent Controller)[^a-ric] 방어 우회 사례가 정확히 이 유형입니다 — 공격자가 LLDP(Link Layer Discovery Protocol)[^a-lldp] 에이전트를 **장기간 과부하시켜 지연 임계값을 서서히 올려놓고**(수 시간 준비), 그 다음 대역 외 채널 지연을 임계값 아래로 숨깁니다. **정적 임계값 기반 방어는 S6에 구조적으로 취약합니다.**
{: .prompt-danger }

---

## 4. ML 위협 9범주 개요

Benzaïd 등[^r5]은 O-RAN에 유의미한 ML 위협을 **9개 범주**로 정리합니다. 각 범주의 상세와 O-RAN 시나리오는 **Ch6**에서 다루고, 여기서는 전체 지형을 잡습니다.

| # | 위협 범주 | Goal | Influence | 대표 전략 | 상세 |
|---|---|---|---|---|---|
| 1 | **Poisoning** (오염) | 무결성 / 가용성 | Causative (재)학습 | S1, S2, S4 | Ch6 §2 |
| 2 | **Evasion** (회피) | 무결성 | Exploratory 추론 | S1, S2, S4 | Ch6 §3 |
| 3 | **Membership Inference** (멤버십 추론) | 기밀성 | Exploratory | S3, S4 | Ch6 §4, Ch9 |
| 4 | **Data Property Inference** (데이터 속성 추론) | 기밀성 | Exploratory | S3 | Ch6 §4, Ch9 |
| 5 | **Data Reconstruction** (데이터 재구성) | 기밀성 | Exploratory | S3, S4 | Ch6 §4, **Ch9** |
| 6 | **Model Stealing** (모델 탈취) | 기밀성 | Exploratory | S3, S4 | Ch6 §5 |
| 7 | **Model Reprogramming** (모델 재프로그래밍) | 무결성 | Exploratory | S1, S2 | Ch6 §5 |
| 8 | **ML Supply Chain Attacks** (ML 공급망) | 전부 | 전 단계 | **S7** | Ch6 §6, Ch7 AIBOM(AI Bill of Materials)[^a-aibom] |
| 9 | **Resource Exhaustion** (자원 고갈) | 가용성 | Exploratory | S1, S6 | Ch6 §7 |

### 4.1 Poisoning의 하위 형태 (예시)

9범주 각각이 다시 여러 형태로 갈라집니다. 예를 들어 오염 공격은[^r5]:

| 형태 | 내용 | 한계 |
|---|---|---|
| **Label-flipping** | 원래 소스 클래스 라벨을 표적 클래스로 변경, 분류 오류 최대화 | **데이터 정제·재라벨링으로 쉽게 필터링됨** |
| Clean-label poisoning | 원본 샘플을 바꾸지 않고 정교하게 조작한 오염 샘플을 주입 | 탐지 난이도 높음 |
| Learning-algorithm corruption | 학습 알고리즘 또는 학습 로직 자체를 오염 | C5(SW 통제) 필요 |

---

## 5. O-RAN 특화 취약점 목록 (V-01 ~ V-07)

Soltani 등[^r6]은 RIC 취약점을 **출처(Open RAN 고유 / SDN 상속)**, **영향 자산**, **근본 원인**, **잠재 영향**, **심각도**, **발생 가능성**으로 구조화합니다. 이 표는 실무 위협 모델링의 출발점으로 그대로 쓸 수 있습니다.

| ID | 취약점 | 출처 | 영향 자산 | 근본 원인 | 잠재 영향 | 심각도 | 가능성 |
|---|---|---|---|---|---|---|---|
| **V-01** | 무선자원에 대한 **미인증 접근** | Open RAN 고유 | 가입자 데이터, 네트워크 데이터, gNB[^a-gnb]-DU[^a-du], gNB-CU[^a-cu] | 침해된 xApp / 부실 설계 xApp / **악성 중첩(nested) xApp** / 비보안 API / 오픈소스 SW | 서비스 중단, 성능 저하 | **High** | **High** |
| **V-02** | UE[^a-ue] 식별자에 대한 **비인가 접근** | Open RAN 고유 | 가입자 데이터, 가입자 위치정보 | **악성 xApp을 통한 스니핑**, 침해된 RIC | 정보 무결성·기밀성 침해 | **High** | Medium |
| **V-03** | 무선 접속 정책의 **상충과 불일치** | Open RAN 고유 | 네트워크 정책, gNB-DU, gNB-CU | RIC와 O-gNB 간 상충, **다수 xApp에서 발생하는 상충** | 서비스 중단, 성능 저하 | **High** | Medium |
| **V-04** | 네트워크 **오설정** | Open RAN 고유 | 데이터 트래픽, gNB | **다수 공급사 통합**, 하드웨어·소프트웨어 분리 | 서비스 중단, 성능 저하 | Medium | Medium |
| **V-05** | **오염된 AI/ML RAN 기능** | Open RAN 고유 | 네트워크 데이터, gNB, 데이터 트래픽 | **개방 인터페이스**, **멀티벤더 배포** | 서비스 중단, 성능 저하, 기밀성 침해 | **High** | Medium |
| **V-06** | RIC의 **불완전·불충분한 하드닝** | Open RAN 고유 | 데이터 트래픽, gNB | OS[^a-os]/SW 취약점, **부적절한 암호키 관리**, 패치 관리 프로세스 부재, 부적절한 로그·감사 | 서비스 중단, 기밀성·무결성 침해, 성능 저하 | Medium | **High** |
| **V-07** | 데이터 평면의 **부정확한 토폴로지 발견** | **SDN 상속** | 데이터 트래픽, gNB-DU, gNB-CU-UP | **비인가 LLDP 중계**, 침해된 LLDP, **Port amnesia** 기법, 잘못 계산된 링크 지연 | 서비스 중단, 정보 무결성, 성능 문제 | **High** | Medium |

_표 4-2. RIC 취약점 종합. 출처: Soltani 등[^r6], Table IV를 재구성._

> **읽는 법**:
> - **V-01·V-06이 "가능성 High"** 입니다. 즉 가장 먼저 대응해야 할 항목은 화려한 적대적 ML이 아니라 **xApp 접근 통제(V-01)와 기본 하드닝·패치·키 관리(V-06)** 입니다.
> - **V-05(오염된 AI/ML RAN 기능)** 의 근본 원인이 "개방 인터페이스 + 멀티벤더"로 명시됩니다. 즉 **O-RAN의 설계 목표 자체가 이 취약점의 원인**입니다 — 제거할 수 없고 관리해야 합니다.
> - **V-07만 "SDN 상속"** 입니다. Ch5에서 이 계열(LLDP 위조, Port Amnesia, Link Latency Attack, Bearer Migration Poisoning)을 상세히 다룹니다.
{: .prompt-tip }

### 5.1 NIST 온톨로지 기반 O-RAN 위협 분석 예시

Aizikovich 등[^r9]은 O-RAN 위협을 NIST 온톨로지에 따라 분석합니다.

![NIST 온톨로지 기반 위협 분석 (출처: Aizikovich 등[^r9], Fig. 3)](/assets/img/posts/6g-ai-ran/roguecell-fig3.png)
_그림 4-4. NIST 온톨로지에 기반한 위협 분석 구조. 출처: [^r9], Fig. 3._

### 5.2 O-RAN WG11 위협 모델링 원문 — 표준이 규정한 위협 카탈로그

앞의 V-01~V-07은 학술 논문의 정리입니다. **표준 문서 자체**는 어떻게 위협을 규정하는지 보겠습니다. O-RAN Alliance WG11의 *O-RAN Security Threat Modeling and Risk Assessment*[^wg11threat]는 ETSI를 통해 **TR(Technical Report) 104 106**[^a-tr][^etsithreat]으로도 공개됩니다(동일 내용의 PAS(Publicly Available Specification)[^a-pas]).

![Logical Architecture of O-RAN system (출처: O-RAN WG11[^wg11threat] / ETSI TR 104 106[^etsithreat], Fig. 4-1)](/assets/img/posts/6g-ai-ran/etsithreat-fig4.png)
_그림 4-5. **위협 모델링의 기준 아키텍처**. 모든 위협은 이 논리 아키텍처의 구성요소·인터페이스에 매핑됩니다. 출처: [^etsithreat], Fig. 4-1._

#### 인터페이스·구성요소별 위협 지점

![Threats and Vulnerabilities for O-RAN LLS 7-2x (출처: ETSI TR 104 106[^etsithreat], Fig. 7-1)](/assets/img/posts/6g-ai-ran/etsithreat-fig7.png)
_그림 4-6. **O-RAN LLS(Lower Layer Split)[^a-lls] 7-2x(Open Fronthaul)의 위협과 취약점**. Ch2 §2.2에서 본 Option 7-2x 분할 지점이 그대로 위협 지점이 됩니다. 출처: [^etsithreat], Fig. 7-1._

![UE Identification in Near-RT-RIC (출처: ETSI TR 104 106[^etsithreat], Fig. 7-2)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p37.png)
_그림 4-7. **Near-RT RIC에서의 UE 식별**. Ch4의 **V-02**(UE 식별자 비인가 접근, 심각도 High)와 Ch9의 프라이버시 논의가 이 그림에서 출발합니다. 출처: [^etsithreat], Fig. 7-2._

![Near-RT-RIC and xApps conflict with gNB (출처: ETSI TR 104 106[^etsithreat], Fig. 7-3)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p39.png)
_그림 4-8. **Near-RT RIC·xApp과 gNB의 상충**. Ch3 §7.3과 Ch5 §4의 상충 문제가 **표준 위협 문서에 명시된 위협**임을 보여줍니다 — V-03의 근거. 출처: [^etsithreat], Fig. 7-3._

![A1 interface between the Non-RT RIC and the Near-RT RIC (출처: ETSI TR 104 106[^etsithreat], Fig. 7-4)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p43.png)
_그림 4-9. **A1 인터페이스** 위협 지점. Ch7 §5.5의 "A1 정책을 자동 신뢰하지 말라"의 근거입니다. 출처: [^etsithreat], Fig. 7-4._

![RAN Analytics Information (RAI) services via Y1 service interface (출처: ETSI TR 104 106[^etsithreat], Fig. 7-5)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p46.png)
_그림 4-10. **Y1 서비스 인터페이스를 통한 RAN 분석 정보(RAI, RAN Analytics Information)[^a-rai] 서비스**. 인증된 소비자에게 분석 데이터를 노출하는 경로 — **데이터 유출 표면**입니다. 출처: [^etsithreat], Fig. 7-5._

![REST Protocol Stack for the A1 and R1 Interfaces (출처: ETSI TR 104 106[^etsithreat], Fig. 7-13)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p62.png)
_그림 4-11. **A1·R1 인터페이스의 REST(Representational State Transfer)[^a-rest] 프로토콜 스택**. 표준이 REST/HTTP(HyperText Transfer Protocol)[^a-http] 기반임을 명시하므로, 웹 API 보안(인증·인가·레이트리밋·입력검증)이 그대로 적용됩니다. 출처: [^etsithreat], Fig. 7-13._

#### 가상화·클라우드 위협 — Layer 1 표면의 구체화

§1의 "가상화·컨테이너" 표면이 표준 문서에서 구체적 공격으로 열거됩니다. **AI-RAN Site(Ch2 §5.3)에서 AI 워크로드와 RAN 워크로드가 같은 O-Cloud를 공유하므로 특히 중요합니다.**

![Illustration of the VM/Container escape attack (출처: ETSI TR 104 106[^etsithreat], Fig. 7-6)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p50.png)
_그림 4-12. **VM(Virtual Machine)[^a-vm]/컨테이너 탈출(escape) 공격**. Ch11 §1.2에서 "네임스페이스 격리가 막을 수 없는 것"으로 지목한 바로 그 공격입니다. 출처: [^etsithreat], Fig. 7-6._

![Illustration of the migration flooding attack (출처: ETSI TR 104 106[^etsithreat], Fig. 7-7)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p51.png)
_그림 4-13. **마이그레이션 플러딩 공격** — VM/컨테이너 이관을 남발해 자원을 소진. 출처: [^etsithreat], Fig. 7-7._

![Illustration of the migration MITM attack (출처: ETSI TR 104 106[^etsithreat], Fig. 7-9)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p52.png)
_그림 4-14. **마이그레이션 MITM(Man-in-the-Middle)[^a-mitm] 공격** — 이관 중 메모리 이미지를 가로챕니다. **Ch9의 "in-use 데이터 보호(TEE)"**[^a-tee] 필요성의 직접적 근거입니다. 출처: [^etsithreat], Fig. 7-9._

![Illustration of the Theft-of-Service/DoS Attack (출처: ETSI TR 104 106[^etsithreat], Fig. 7-10)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p53.png)
_그림 4-15. **서비스 절취(Theft-of-Service)/DoS(Denial of Service)[^a-dos] 공격**. Ch6 §3.7의 자원 고갈과 **EDoS**(Economic Denial of Sustainability)[^a-edos]가 표준 위협으로 등재된 형태입니다. 출처: [^etsithreat], Fig. 7-10._

![Illustration of the VM/Container hyperjacking attack (출처: ETSI TR 104 106[^etsithreat], Fig. 7-11)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p55.png)
_그림 4-16. **하이퍼재킹(hyperjacking)** — 하이퍼바이저 자체를 장악. Ch9 §4의 "O-Cloud 내부 고권한 공격자"가 이것입니다. 출처: [^etsithreat], Fig. 7-11._

![Illustration of a cross VM/Container side channel attack (출처: ETSI TR 104 106[^etsithreat], Fig. 7-12)](/assets/img/posts/6g-ai-ran/etsithreat-fig7-p56.png)
_그림 4-17. **크로스 VM/컨테이너 사이드채널 공격**. Ch4 §2.3의 공격자 능력 **C6(하드웨어 통제)** 및 **S5(대역 외 정보 악용)** 에 해당합니다. **GPU를 공유하는 AI-RAN에서 새로운 사이드채널이 생깁니다.** 출처: [^etsithreat], Fig. 7-12._

| 표준이 규정한 클라우드 위협 | 본 시리즈 대응 |
|---|---|
| VM/컨테이너 **탈출** | Ch11 §1.2 격리 한계, Ch7 §1.6 샌드박싱 |
| 마이그레이션 **플러딩** | Ch6 §3.7 자원 고갈 |
| 마이그레이션 **MITM** | **Ch9 TEE (in-use 보호)** |
| **Theft-of-Service / DoS** | Ch6 §3.7, EDoS |
| **하이퍼재킹** | **Ch9 TEE + 원격 검증** |
| **크로스 VM 사이드채널** | Ch4 C6·S5, Ch9 §4 |

### 5.3 WG11의 위험 평가 방법론

위협 목록만으로는 우선순위를 정할 수 없습니다. WG11 문서는 위험 평가의 개념 틀도 함께 제시합니다.

![Main concepts of risk assessment (출처: ETSI TR 104 106[^etsithreat], Fig. 9-1)](/assets/img/posts/6g-ai-ran/etsithreat-fig9.png)
_그림 4-18. **위험 평가의 주요 개념**. 위협·취약점·자산·영향·가능성의 관계를 규정합니다 — 표 4-2(V-01~V-07)의 "심각도 × 가능성" 평가가 이 틀에서 나옵니다. 출처: [^etsithreat], Fig. 9-1._

> **실무 적용 순서**: ① 그림 4-5의 논리 아키텍처에 자신의 배치를 매핑 → ② 그림 4-6~4-17에서 해당되는 위협을 골라냄 → ③ 그림 4-18의 위험 평가 틀로 심각도·가능성 산정 → ④ §7의 템플릿으로 위협 모델 문서화 → ⑤ Ch7 §5의 통제와 대조.
{: .prompt-tip }

### 5.4 공유 O-RU — 멀티테넌트 특화 위협

AI-RAN의 인프라 공유(Ch1 §3.3)와 직결되는 표준 문서입니다. WG11의 *Shared O-RU Security Analysis*[^wg11oru]는 **하나의 O-RU를 여러 사업자가 공유**할 때의 신뢰 분할을 다룹니다.

![Shared O-RU Tenant/Host Split (출처: O-RAN WG11[^wg11oru], Fig. 4.1-1)](/assets/img/posts/6g-ai-ran/wg11oru-fig4.png)
_그림 4-19. **공유 O-RU의 Tenant/Host 분할**. 어느 기능이 호스트(인프라 소유자)에, 어느 기능이 테넌트(사업자)에 속하는지가 신뢰 경계를 결정합니다. Ch5의 APATE가 가정한 **멀티오퍼레이터 배치**의 표준적 근거입니다. 출처: [^wg11oru], Fig. 4.1-1._

![Security Architecture for Shared O-RU Options 1 and 2 (출처: O-RAN WG11[^wg11oru], Fig. 4.2-1)](/assets/img/posts/6g-ai-ran/wg11oru-fig4-p10.png)
_그림 4-20. 공유 O-RU 옵션 1·2의 보안 아키텍처. 출처: [^wg11oru], Fig. 4.2-1._

![Shared Operator Host PKI issues certificates to each of the Shared Resource Operators (출처: O-RAN WG11[^wg11oru], Fig. 10-1)](/assets/img/posts/6g-ai-ran/wg11oru-fig10.png)
_그림 4-21. **공유 사업자 호스트 PKI가 각 공유자원 사업자에게 인증서를 발급**하는 예. 멀티테넌트 신뢰의 기술적 실체는 **PKI 설계**입니다 — Ch12 §1.2에서 PQC(Post-Quantum Cryptography)[^a-pqc] 전환 우선순위가 높은 이유입니다. 출처: [^wg11oru], Fig. 10-1._

---

## 6. ML 소프트웨어 의존성이 만드는 표면

가장 과소평가되는 표면입니다. xApp/rApp은 **파이썬 ML 스택 위에 올라갑니다** — 즉 수백 개의 전이 의존성(transitive dependency)을 상속합니다.

![ML 소프트웨어 의존성 악용으로 생기는 O-RAN 공격 표면 (출처: Benzaïd 등[^r5], Fig. 18)](/assets/img/posts/6g-ai-ran/aisurvey-fig18.png)
_그림 4-22. **ML 소프트웨어 의존성 악용으로 발생하는 O-RAN 공격 표면**. 출처: [^r5], Fig. 18._

![O-RAN 시스템 내 악성코드 주입 위협 시나리오 (출처: Benzaïd 등[^r5], Fig. 17)](/assets/img/posts/6g-ai-ran/aisurvey-fig17.png)
_그림 4-23. O-RAN 시스템 내 전반적 악성코드 주입 위협 시나리오. 출처: [^r5], Fig. 17._

| 표면 | 구체적 위험 |
|---|---|
| ML 프레임워크 (PyTorch/TF 등) | 역직렬화 취약점, 모델 파일에 임베드된 코드 실행 |
| 전이 의존성 | typosquatting, 계정 탈취를 통한 악성 버전 배포 |
| 사전학습 모델(pretrained weights) | **트로이목마 임베딩** |
| 컨테이너 베이스 이미지 | 알려진 CVE(Common Vulnerabilities and Exposures)[^a-cve] 누적, 권한 상승 |
| AI 가속기 드라이버·펌웨어 | **하드웨어 트로이목마**(C6) |

> **규제적 함의**: Benzaïd 등[^r5]은 **EU AI Act**[^a-eu]를 명시적으로 인용합니다. 통신 네트워크 같은 핵심 인프라를 보호·관리하는 데 쓰이는 AI/ML 시스템은 **고위험(high-risk)** 으로 분류되며, **강건성·투명성·설명가능성·프라이버시 보존**에 대한 엄격한 요구가 부과됩니다. 즉 AI-RAN 보안은 기술 선택이 아니라 **컴플라이언스 의무**입니다(Ch10, Ch12).
{: .prompt-warning }

---

## 7. 위협 모델 작성 템플릿

앞의 6차원을 실무 문서로 옮길 때 쓸 수 있는 템플릿입니다.

```text
[위협 ID]      T-XX
[표적 자산]     예: 트래픽 스티어링 xApp의 DRL 정책 모델
[공격자 목표]   무결성 (특정 셀로 UE를 유인)
[지식]         Gray-box (E2SM-KPM 규격 공개 → 특징 부분집합 f 파악)
[능력]         C2 (추론 데이터 통제: 인접 셀 KPM 보고 조작)
              C7 (통신 채널 통제: 특정 대역 재밍)
[한계]         L2 (쿼리 제한: 모델 출력 직접 관측 불가 → Myopic 근접)
[특정성]        Targeted (특정 UE 그룹)
[영향 시점]     Exploratory (추론 단계)
[공격 전략]     S1 (데이터 조작) + S6 (전략적 타이밍: 트래픽 피크 시점)
[영향]         서비스 중단 / 특정 셀 과부하 / SLA 위반
[관련 취약점]   V-01, V-05
[탐지 가능성]   Near-RT RIC KPM 이상탐지로 부분 탐지 가능 (Ch8)
[방어]         입력 정화 + MTD + XAI 기반 결정 근거 검사 (Ch7)
```

### 7.1 O-RAN WG11 위협 모델링 문서와의 관계

O-RAN Alliance Security Work Group(WG11)은 위협 모델링·위험 평가 문서를 발행하며, ETSI를 통해 PAS(Publicly Available Specification)로도 공개됩니다.

| 문서 | 내용 | 본 시리즈에서 다루는 곳 |
|---|---|---|
| **O-RAN.WG11 Threat Modeling and Risk Assessment**[^wg11threat] (= **ETSI TR 104 106**[^etsithreat]) | O-RAN 보안 위협 모델링 및 위험 평가 | **§5.2~§5.3** (그림 4-5~4-18) |
| **O-RAN.WG11 Security Requirements and Controls Specifications**[^wg11secreq] (= **ETSI TS 104 104**[^a-ts][^etsisecreq]) | O-RAN 보안 요구사항 및 통제 규격 | **Ch2 §3.3, Ch7 §5** |
| **O-RAN WG11 Shared O-RU Security Analysis**[^wg11oru] (TR.O-R004) | 공유 O-RU 보안 분석 | **§5.4** (그림 4-19~4-21) |

Benzaïd 등[^r5]도 O-RAN Alliance가 "AI/ML in O-RAN에 대한 위협 모델과 위험 평가를 제공하여, 잠재적 공격 벡터·관련 위험·완화 고려사항을 제시했다"고 정리합니다.

---

## 8. 계층별 공격·완화 매핑과 자동화된 위협 모델링 동향

지금까지는 O-RAN 표준 문서와 ML 위협 분류체계를 중심으로 보았습니다. 이번 절에서는 두 가지를 보완합니다 — **(1)** Braeken 등[^r36]이 정리한 **6G 아키텍처 계층별 공격·완화 매핑**으로 §1의 2층 모델을 실무 체크리스트 수준까지 구체화하고, **(2)** Bezerra 등[^r38]의 문헌 조사를 바탕으로 **RIS(Reconfigurable Intelligent Surface)[^a-ris] 특화 물리계층 위협**과 **생성형 AI 기반 위협 모델링 자동화**라는 두 최신 흐름을 짚습니다.

### 8.1 6G 아키텍처 5계층 공격·완화 매핑

Braeken 등[^r36]은 6G 아키텍처를 **Device·RAN·Edge·Core·Application** 5계층으로 나누고, 계층마다 대표 AI 기반 공격과 그에 대응하는 AI 기반 완화책을 매핑합니다. §1의 "Layer 1/Layer 2" 2층 모델이 **어떤 물리적 위치**에서 실현되는지 보여주는 보완적 시각입니다.

![AI attacks on 6G architecture and their mitigation techniques (출처: Braeken 등[^r36], Fig. 2)](/assets/img/posts/6g-ai-ran/aisecland-fig2.png)
_그림 4-24. **6G 아키텍처 5계층의 AI 공격·완화 매핑**. Device→RAN→Edge→Core로 이어지는 트래픽 경로 위에 계층별 공격(빨강)과 완화책(초록)을 겹쳐 놓았습니다. 출처: [^r36], Fig. 2._

| 계층 | 대표 AI 기반 공격 | 대표 AI 기반 완화 |
|---|---|---|
| **Device** | 약한 암호화, 침해된 노드, 데이터 절취, 도청 | 기기 행동 기반 이상탐지, 분산 보안을 위한 FL |
| **RAN** | 악성 xApp/rApp, UE 추적·스푸핑, 데이터 오염, MITM | AI 기반 침입탐지(A1·X2 모니터링), 적대적 훈련 |
| **Edge** | DDoS·MITM, 모델 조작, 마이크로서비스 공격, 데이터 오염 | AI 기반 마이크로서비스 위협 탐지, 예측적 취약점 탐지, ML 기반 보안 오케스트레이션 |
| **Core** | SDN/VM/VNF 공격, 정보 절취, DoS, 프라이버시 침해 | ML 기반 보안 오케스트레이션, AI 기반 네트워크 슬라이스 격리 |
| **Application** | 적대적 입력, 무단 데이터 접근, 모델 역전, 딥페이크·사회공학 | 설명가능 AI, 강건한 ML 훈련, 생체인증 |

_표 4-3. 6G 아키텍처 계층별 AI 공격·완화 매핑. 출처: Braeken 등[^r36], Table IV를 요약._

> **§1과의 관계**: 표 4-1의 Layer 1(O-RAN 인프라)은 이 표의 RAN·Edge·Core 계층에, Layer 2(AI/ML)는 다섯 계층 모두에 수평으로 걸쳐 있습니다. 즉 표 4-3은 표 4-1을 "어디서"의 축으로 재배열한 것입니다.
{: .prompt-tip }

### 8.2 RIS — 물리계층에 특화된 새로운 공격 표면

Bezerra 등[^r38]은 5G/6G 위협 모델링 문헌을 조사하면서, RIS(재구성 가능 지능형 표면)를 **6G가 가져오는 가장 새로운 물리계층 공격 표면**으로 지목합니다. RIS는 다수의 소형 반사 소자로 전파 환경 자체를 프로그래밍 가능하게 만들어 mMIMO(massive MIMO)[^a-mmimo]의 커버리지를 확장하고 AmBC(Ambient Backscatter Communication)[^a-ambc]와 같은 초저전력 통신을 지원합니다. 그러나 이 "프로그래밍 가능한 전파 환경"이라는 속성 자체가 공격자에게도 열려 있습니다.

![6G System Architecture: Devices–C-RAN–Core–Services (출처: Bezerra 등[^r38], Fig. 3)](/assets/img/posts/6g-ai-ran/tmtrend-fig3.png)
_그림 4-25. Bezerra 등이 사용한 **6G 시스템 아키텍처 단순도**. Device→C-RAN→Core→Services로 이어지는 경로 위에 RIS 위협이 얹힙니다. 출처: [^r38], Fig. 3._

| RIS 특화 위협 | 메커니즘 | 근거 문헌 |
|---|---|---|
| **Nulled interference**(무효화 간섭) | RIS 소자를 조작해 특정 방향으로 보강간섭 대신 상쇄간섭을 만들어 데이터 전송을 차단 | Di Renzo 등 |
| **Nulled reflection**(무효화 반사) | 반사 위상을 조작해 의도된 반사 경로 자체를 없앰 | Di Renzo 등 |
| **PL-SKG(Physical-Layer Secret Key Generation) 유출** | 채널 상호성에 기반한 물리계층 키 생성 과정에 RIS가 개입해 키 정보를 유출 | Wei·Guo; Khalid 등; Gao 등 |
| **RIS 재밍** | 정당한 송신에는 유리하고 특정 수신자에는 불리하도록 반사를 조작 — 수신자 입장에서는 재밍 여부를 구분하기 어려움("transparent jamming") | Sena 등 |

_표 4-4. RIS 특화 물리계층 위협. 출처: Bezerra 등[^r38]이 정리한 RIS 문헌을 재구성._

> RIS 공격의 공통점은 **소프트웨어 방어로는 막을 수 없다**는 것입니다 — §2.3에서 본 능력 차원의 **C7(통신 채널 통제)** 이 RIS에서는 "채널을 흐르는 신호"가 아니라 **"채널 그 자체"** 를 겨냥하는 수준으로 확장됩니다. RIS를 도입하는 AI-RAN 배치는 이 표를 위협 모델에 별도 행으로 추가해야 합니다.
{: .prompt-danger }

Bezerra 등[^r38]이 56개 위협을 집계한 문헌 조사(Table I)에서도 **DoS 계열(60%)** 과 **재밍(40%)** 이 가장 빈번히 보고되었고, RIS 관련 위협은 이 중 물리계층 고유 범주로 별도 분류됩니다.

### 8.3 생성형 AI·LLM 기반 위협 모델링 자동화

O-RAN·6G의 공격 표면이 넓어질수록, 사람이 수작업으로 위협을 나열하는 §7의 템플릿 방식만으로는 확장성이 부족합니다. Bezerra 등[^r38]은 이에 대응하는 세 갈래의 자동화 흐름을 정리합니다.

| 접근 | 핵심 아이디어 | 대표 사례 |
|---|---|---|
| **생성형 AI 기반 위협모델 생성** | GAN·GPT 기반·BERT 세 가지 생성형 AI 접근을 비교 평가한 뒤, **GAN 기반 모델과 트랜스포머 기반 모델**을 6G 위협 모델링용으로 제안 | Ferrag, Debbah, Al-Hawawreh |
| **지식베이스 기반 능동 취약점 탐색** | 7단계 방법론으로 위협을 모델링하고 비즈니스 영향까지 정량화. **ThreatFinderAI**는 ENISA·OWASP AI·MITRE ATLAS를 지식베이스로 사용 | Von der Assen 등 |
| **디지털 트윈 기반 취약점 분석** | AI와 디지털 트윈(DT)[^a-dt] 모델로 취약점을 분석. EU 프로젝트의 **HORSE 아키텍처**는 human-centric·오픈소스·그린·지속가능·조정된 프로비저닝을 6G 보안의 핵심 개념으로 제시 | Rodrigues 등 |

_표 4-5. 자동화된 위협 모델링 접근 3갈래. 출처: Bezerra 등[^r38]의 문헌 조사를 재구성._

이 흐름은 §7의 수작업 템플릿을 대체하기보다 **입력을 자동 생성하는 전(前)단계**로 자리 잡을 전망입니다 — IDS(Intrusion Detection System)[^a-ids]·IPS(Intrusion Prevention System)[^a-ips]·SIEM(Security Information and Event Management)[^a-siem]·SOAR(Security Orchestration, Automation, and Response)[^a-soar]·XDR(Extended Detection and Response)[^a-xdr] 같은 위협 헌팅 도구체계에 **LLM 기반 위협 분석과 디지털 트윈 기반 네트워크 보안 평가**가 지식베이스로 결합되는 방향입니다[^r38]. Ch10 §3에서 이 흐름을 컴플라이언스 자동화 관점으로 다시 다룹니다.

---

## 9. 이 장의 요약

- 공격 표면은 **2층 구조**입니다. O-RAN 인프라 표면(인터페이스·컨테이너·제3자 앱·공급망) 위에 AI/ML 표면(데이터·모델·추론입력·배포경로·가속기)이 겹쳐 있고, **AI 결정이 인프라를 제어하므로 AI 침해 = 인프라 침해**입니다.
- ML 위협 모델은 **6차원**(목표·지식·능력·한계·특정성·영향시점) + **공격 전략** 으로 기술합니다[^r5].
- 지식 수준 4단계(**White/Gray/Black/Myopic**)에서, O-RAN은 **표준의 개방성 때문에 gray-box 지식을 공격자에게 무료로 제공**합니다.
- 능력 차원에 **C6(하드웨어), C7(통신채널), C8(행동접근)** 이 추가된 것이 무선 도메인 특화의 핵심입니다.
- 공격 전략 **S1~S7** 중 **S6(전략적 타이밍)** 과 **S7(SW/HW 침해)** 가 실무에서 가장 과소평가됩니다.
- **V-01~V-07** 중 발생 가능성이 High인 것은 **V-01(미인증 무선자원 접근)** 과 **V-06(RIC 하드닝 부족)** 입니다 — 우선순위가 여기 있습니다[^r6].
- **표준 위협 카탈로그**(O-RAN WG11 / ETSI TR 104 106)는 인터페이스별 위협뿐 아니라 **가상화·클라우드 공격 6종** — VM/컨테이너 **탈출**, 마이그레이션 **플러딩**, 마이그레이션 **MITM**, **Theft-of-Service/DoS**, **하이퍼재킹**, **크로스 VM 사이드채널** — 을 명시합니다. AI-RAN이 GPU를 공유하는 구조에서 이 목록은 더 중요해집니다[^etsithreat].
- **공유 O-RU**의 Tenant/Host 분할과 **PKI 설계**가 멀티테넌트 신뢰의 실체입니다[^wg11oru].
- **EU AI Act 하에서 AI-RAN 보안은 컴플라이언스 의무**입니다[^r5].
- **6G 아키텍처 5계층(Device·RAN·Edge·Core·Application) 공격·완화 매핑**은 §1의 2층 모델을 "어디서"의 축으로 재배열한 실무 체크리스트입니다[^r36].
- **RIS**는 소프트웨어 방어로 막을 수 없는 새로운 물리계층 표면을 엽니다 — nulled interference/reflection, PL-SKG 유출, transparent jamming이 대표 위협입니다[^r38].
- 위협 모델링 자체도 **생성형 AI(GAN/트랜스포머 기반 모델 생성) · 지식베이스 기반 능동 탐색(ThreatFinderAI) · 디지털 트윈 기반 분석(HORSE)** 세 갈래로 자동화되는 중입니다[^r38].

### 확인 체크리스트

- [ ] White-box / Gray-box / Black-box / Myopic를 "가용 지식"과 "출력 관측" 두 축으로 구분할 수 있는가
- [ ] C6·C7·C8이 왜 무선 도메인에 특별히 필요한지 설명할 수 있는가
- [ ] S6 전략이 정적 임계값 방어를 무력화하는 메커니즘을 설명할 수 있는가
- [ ] V-01~V-07 중 어느 것이 SDN 상속 취약점인지 아는가
- [ ] 자신의 시스템에 대해 §7 템플릿을 채울 수 있는가
- [ ] RIS의 nulled interference/reflection과 PL-SKG 유출이 왜 소프트웨어 방어로 막히지 않는지 설명할 수 있는가
- [ ] 생성형 AI 기반 위협 모델링 자동화의 3갈래(생성·지식베이스·디지털 트윈)를 구분할 수 있는가

**다음 장**: [05. RAN 인터페이스 및 멀티 에이전트 제어 루프 위협](/posts/airan-05-interface-agent-threats/)

---

### 약어

[^a-o-ran]: **O-RAN**(Open Radio Access Network): 기지국을 표준 개방형 인터페이스로 분해하여 서로 다른 제조사의 장비 간 상호운용을 가능하게 하는 개방형 무선 접속망 아키텍처입니다. O-RAN Alliance가 규격을 제정합니다.
[^a-ai]: **AI**(Artificial Intelligence): 인공지능. 학습·추론·판단 등 인간의 지적 능력을 컴퓨터 시스템으로 구현하는 기술의 총칭입니다.
[^a-ml]: **ML**(Machine Learning): 기계학습. 명시적인 규칙 대신 데이터에서 패턴을 학습하여 예측·분류를 수행하는 AI의 핵심 기술 분야입니다.
[^a-nist]: **NIST**(National Institute of Standards and Technology): 미국 국립표준기술연구소. 암호·AI 보안 등 광범위한 분야의 기술 표준과 지침(적대적 ML 분류체계 NIST AI 100-2 등)을 발행합니다.
[^a-wg]: **WG**(Working Group): 표준화 기구 내의 작업반. WG11은 O-RAN Alliance에서 보안을 전담하는 작업반입니다.
[^a-etsi]: **ETSI**(European Telecommunications Standards Institute): 유럽전기통신표준협회. O-RAN 규격을 PAS 형태로도 발행하는 유럽의 통신 표준화 기구입니다.
[^a-ru]: **RU**(Radio Unit): 무선 신호의 송수신과 하위 물리계층 처리를 담당하는 무선 장치이며, O-RU는 O-RAN 규격을 따르는 RU입니다.
[^a-pki]: **PKI**(Public Key Infrastructure): 공개키 기반구조. 인증서의 발급·검증 체계를 통해 통신 주체의 신원과 공개키를 신뢰할 수 있게 관리하는 보안 기반입니다.
[^a-o-fh]: **O-FH**(Open Fronthaul): O-RAN이 표준화한 RU와 DU 사이의 개방형 프론트홀 인터페이스입니다.
[^a-sw]: **SW**(Software): 소프트웨어를 뜻하는 약어입니다.
[^a-ran]: **RAN**(Radio Access Network): 무선 접속망. 단말과 코어망 사이에서 무선 구간의 연결을 담당하는 기지국 중심의 네트워크 영역입니다.
[^a-c-ran]: **C-RAN**(Centralized RAN): 여러 기지국의 기저대역 처리 기능을 중앙에 집중시켜 운용하는 RAN 구조입니다.
[^a-vran]: **vRAN**(virtualized RAN): 기저대역 처리 기능을 범용 서버 위의 소프트웨어로 가상화한 RAN 구조입니다.
[^a-dl]: **DL**(Deep Learning): 딥러닝. 다층 신경망으로 데이터의 표현을 학습하는 기계학습 방법입니다.
[^a-rl]: **RL**(Reinforcement Learning): 강화학습. 에이전트가 환경과 상호작용하며 보상을 최대화하는 행동 정책을 스스로 학습하는 기계학습 방법입니다.
[^a-drl]: **DRL**(Deep Reinforcement Learning): 심층 강화학습. 심층 신경망을 함수 근사에 사용하는 강화학습 기법입니다.
[^a-fl]: **FL**(Federated Learning): 연합학습. 원본 데이터를 중앙으로 모으지 않고 각 참여 노드가 로컬에서 학습한 모델 갱신값만 공유하여 전역 모델을 만드는 분산 학습 방식입니다.
[^a-kpm]: **KPM**(Key Performance Measurement): 셀·단말 단위의 처리량·지연 등 RAN 성능을 나타내는 핵심 성능 측정 지표로, E2 인터페이스를 통해 수집됩니다.
[^a-api]: **API**(Application Programming Interface): 소프트웨어 구성요소들이 서로 기능을 호출·연동할 수 있도록 정의된 인터페이스입니다.
[^a-gpu]: **GPU**(Graphics Processing Unit): 대규모 병렬 연산에 특화된 프로세서로, AI 학습·추론의 핵심 가속 하드웨어입니다.
[^a-fgsm]: **FGSM**(Fast Gradient Sign Method): 손실 함수 기울기의 부호 방향으로 한 번에 섭동을 더해 적대적 예제를 만드는 대표적인 공격 기법입니다.
[^a-pgd]: **PGD**(Projected Gradient Descent): 섭동을 허용 범위 안으로 사영(projection)하며 기울기 기반 공격을 반복 적용하는 강력한 반복형 적대적 공격 기법입니다.
[^a-cw]: **C&W**(Carlini & Wagner): 적대적 예제 생성을 최적화 문제로 정식화하여 최소 크기의 섭동을 찾아내는 강력한 공격 기법입니다.
[^a-gan]: **GAN**(Generative Adversarial Network): 생성적 적대 신경망. 생성자와 판별자가 경쟁하며 학습하여 실제와 유사한 데이터를 만들어내는 모델입니다.
[^a-hw]: **HW**(Hardware): 하드웨어를 뜻하는 약어입니다.
[^a-sdn]: **SDN**(Software-Defined Networking): 제어평면과 데이터평면을 분리하고 소프트웨어 컨트롤러로 네트워크를 중앙에서 제어하는 아키텍처입니다.
[^a-ric]: **RIC**(RAN Intelligent Controller): RAN을 지능적으로 제어·최적화하는 O-RAN의 핵심 컨트롤러로, Near-RT RIC(10ms~1s)과 Non-RT RIC(1s 이상)으로 나뉩니다.
[^a-lldp]: **LLDP**(Link Layer Discovery Protocol): 네트워크 장비가 인접 장비에 자신의 정보를 주기적으로 광고하여 토폴로지 발견을 가능하게 하는 2계층 프로토콜입니다.
[^a-aibom]: **AIBOM**(AI Bill of Materials): 소프트웨어 자재명세서(SBOM) 개념을 AI로 확장한 것으로, 모델·학습 데이터·의존 라이브러리 등 AI 시스템 구성요소의 출처와 이력을 기록한 명세입니다.
[^a-gnb]: **gNB**(next generation NodeB): 5G NR 기지국을 가리키는 3GPP 표준 명칭입니다.
[^a-du]: **DU**(Distributed Unit): 기지국에서 실시간성이 요구되는 하위 프로토콜 계층을 처리하는 분산 장치입니다. O-RAN 규격을 따르는 DU를 O-DU라 부릅니다.
[^a-cu]: **CU**(Central Unit): 기지국의 상위 프로토콜 계층을 처리하는 중앙 장치로, 제어평면의 CU-CP(CU-Control Plane)와 사용자평면의 CU-UP(CU-User Plane)으로 나뉩니다.
[^a-ue]: **UE**(User Equipment): 스마트폰 등 이동통신망에 접속하는 사용자 단말을 가리키는 3GPP 표준 용어입니다.
[^a-os]: **OS**(Operating System): 운영체제. 하드웨어 자원을 관리하고 응용 프로그램의 실행 환경을 제공하는 시스템 소프트웨어입니다.
[^a-tr]: **TR**(Technical Report): 표준화 기구가 발행하는 기술 보고서. 규범적 요구사항보다는 분석·연구 결과를 담는 문서 유형입니다.
[^a-pas]: **PAS**(Publicly Available Specification): 외부 기관이 작성한 규격을 ETSI 등 표준화 기구가 공개 사양 문서로 발행한 것입니다.
[^a-lls]: **LLS**(Lower Layer Split): 물리계층의 하위 기능을 RU에, 상위 기능을 DU에 두는 기지국 기능 분할 방식으로, O-RAN은 Option 7-2x를 채택했습니다.
[^a-rai]: **RAI**(RAN Analytics Information): Near-RT RIC이 Y1 인터페이스를 통해 인증된 소비자에게 제공하는 RAN 분석 정보입니다.
[^a-rest]: **REST**(Representational State Transfer): HTTP 기반으로 자원을 표현하고 조작하는 웹 API 설계 방식입니다.
[^a-http]: **HTTP**(HyperText Transfer Protocol): 웹에서 요청과 응답을 주고받는 표준 응용계층 프로토콜입니다.
[^a-vm]: **VM**(Virtual Machine): 가상머신. 하이퍼바이저 위에서 독립된 물리 서버처럼 동작하도록 가상화된 컴퓨팅 환경입니다.
[^a-mitm]: **MITM**(Man-in-the-Middle): 중간자 공격. 통신 경로의 중간에 끼어들어 데이터를 도청·변조하는 공격입니다.
[^a-tee]: **TEE**(Trusted Execution Environment): 신뢰 실행 환경. 프로세서 수준의 격리 영역을 제공하여 사용 중(in-use)인 데이터와 코드를 보호하는 기술입니다.
[^a-dos]: **DoS**(Denial of Service): 서비스 거부 공격. 자원을 고갈시키거나 시스템을 마비시켜 정상 사용자가 서비스를 이용하지 못하게 하는 공격입니다.
[^a-edos]: **EDoS**(Economic Denial of Sustainability): 클라우드의 종량 과금 구조를 악용해 피해자에게 과도한 비용을 유발함으로써 서비스를 경제적으로 지속 불가능하게 만드는 공격입니다.
[^a-pqc]: **PQC**(Post-Quantum Cryptography): 양자내성암호. 양자컴퓨터로도 풀기 어려운 수학 문제에 기반하여 설계된 차세대 공개키 암호입니다.
[^a-cve]: **CVE**(Common Vulnerabilities and Exposures): 공개적으로 알려진 보안 취약점에 고유 식별자(CVE-연도-번호)를 부여해 관리하는 표준 목록 체계입니다.
[^a-eu]: **EU**(European Union): 유럽연합. AI Act 등 역내 공통 규제를 제정하는 유럽 국가들의 정치·경제 연합입니다.
[^a-ts]: **TS**(Technical Specification): 표준화 기구가 발행하는 기술 규격. 준수해야 할 규범적 요구사항을 담는 문서 유형입니다.
[^a-ris]: **RIS**(Reconfigurable Intelligent Surface): 다수의 소형 반사 소자로 무선 채널 자체를 프로그래밍 가능하게 재구성하는 지능형 표면입니다.
[^a-mmimo]: **mMIMO**(massive MIMO): 매우 많은 수의 안테나를 기지국에 배치해 다수 단말에 동시에 빔을 형성하는 대규모 다중안테나 기술입니다.
[^a-ambc]: **AmBC**(Ambient Backscatter Communication): 별도의 능동 송신기 없이 주변 RF 신호를 반사·변조해 통신하는 초저전력 통신 기법입니다.
[^a-dt]: **DT**(Digital Twin): 물리적 네트워크의 상태를 실시간 반영하는 가상 복제본으로, 정책을 실제 적용 전에 시뮬레이션·검증하는 데 쓰입니다.
[^a-ids]: **IDS**(Intrusion Detection System): 네트워크·시스템의 비정상 행위를 탐지해 경고하는 침입 탐지 시스템입니다.
[^a-ips]: **IPS**(Intrusion Prevention System): 탐지된 침입을 자동으로 차단·저지하는 침입 방지 시스템입니다.
[^a-siem]: **SIEM**(Security Information and Event Management): 여러 시스템의 보안 로그·이벤트를 수집·상관분석하는 보안 정보 이벤트 관리 시스템입니다.
[^a-soar]: **SOAR**(Security Orchestration, Automation, and Response): 보안 대응 절차를 자동화·오케스트레이션하는 체계입니다.
[^a-xdr]: **XDR**(Extended Detection and Response): 엔드포인트·네트워크·클라우드 등 여러 계층의 탐지 데이터를 통합해 대응하는 확장 탐지·대응 체계입니다.

## References

[^r5]: C. Benzaïd, T. Taleb, R. Cavalcanti, and A. Sánchez, "AI in Open RAN security: Enabler and threat — A comprehensive survey and future perspectives," *TechRxiv preprint*, Dec. 2025.
[^r6]: S. Soltani, A. Amanloo, M. Shojafar, and R. Tafazolli, "Intelligent control in 6G open RAN: Security risk or opportunity?" *IEEE Open Journal of the Communications Society*, 2025.
[^r9]: E. Aizikovich, D. Mimran, E. Grolman, Y. Elovici, and A. Shabtai, "Rogue cell: Adversarial attack and defense in untrusted O-RAN setup exploiting the traffic steering xApp," *arXiv preprint*, 2025.
[^r24]: H. He, S. Fei, and Z. Yan, "Advancing 5G security and privacy with AI: A survey," *ACM Computing Surveys*, 2025.
[^wg11threat]: O-RAN ALLIANCE WG11, *O-RAN Security Threat Modeling and Risk Assessment*, Technical Report O-RAN.WG11.Threat-Modeling.O-R003-v03.00.
[^etsithreat]: ETSI, *Publicly Available Specification (PAS); O-RAN Security Threat Modeling and Risk Assessment*, ETSI TR 104 106 V3.0.0, Jun. 2025.
[^wg11secreq]: O-RAN ALLIANCE WG11, *O-RAN Security Requirements and Controls Specifications*, O-RAN.WG11.SecReqSpecs-R003-v09.01.
[^etsisecreq]: ETSI, *Publicly Available Specification (PAS); O-RAN Security Requirements and Controls Specifications*, ETSI TS 104 104 V9.1.0, Jun. 2025.
[^wg11oru]: O-RAN ALLIANCE WG11, *Shared O-RU Security Analysis*, Technical Report TR.O-R004-v06.00, 2025.
[^nist]: National Institute of Standards and Technology, *Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations*, NIST AI 100-2. (표준 참조 — 본 시리즈 참조 폴더에 원문 미포함; Benzaïd 등[^r5]의 인용을 통해 참조)
[^r36]: A. Braeken, D. Deac, T. L. Nguyen, G. Gür, Q.-V. Pham, C. Yapa, P. G. Vinueza-Naranjo, H. Carvajal Mora, C. Moremada, and M. Liyanage, "6G AI security: From fundamentals to offensive and defensive landscape in 6G," *IEEE Communications Surveys & Tutorials*, vol. 28, 2026.
[^r38]: W. R. Bezerra, E. A. Galhardo, L. M. Bezerra, and C. B. Westphall, "Threats and AI trends in threat modeling for 5G/6G," in *Proc. 40th Int. Conf. Information Networking (ICOIN)*, 2026, pp. 645–649, doi: 10.1109/ICOIN68469.2026.11480655.
