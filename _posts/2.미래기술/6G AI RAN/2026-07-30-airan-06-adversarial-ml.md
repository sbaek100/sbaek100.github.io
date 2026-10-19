---
title: "[6G AI-RAN] 06. RAN AI/ML 모델 대상 적대적 공격 (Adversarial ML)"
date: 2026-07-30 10:00:00 +0900
categories:
  - 2.미래보안
  - 6G AI RAN
  - Part II 위협·공격벡터
tags:
  - Adversarial-ML
  - FGSM
  - PGD
  - Poisoning
  - Model-Stealing
  - Supply-Chain
  - Hardware-Trojan
math: true
mermaid: false
---

# RAN AI/ML 모델 대상 적대적 공격

## 들어가며 — 정확도 100%에서 0%까지, 필요한 것은 악성 xApp 하나

Chiejina 등[^r8]의 실험 결과 한 줄이 이 장 전체를 요약합니다.

> **InterClass-Spec 모델의 정확도가 0%로 떨어진다.**[^r8]
{: .prompt-danger }

이것은 시뮬레이션 숫자가 아니라 **실제 O-RAN(Open Radio Access Network)[^a-o-ran] 테스트베드(USRP[^a-usrp] 기반 RAN[^a-ran]·UE[^a-ue]·재머 + 랙 서버)** 에서 측정한 값입니다. 공격에 필요한 것은 **같은 near-RT RIC(RAN Intelligent Controller)[^a-ric]에 공존하는 악성 xApp 하나** — O-RAN의 개방성 때문에 동일 데이터베이스에 접근할 수 있다는 가정뿐입니다.

이 장에서 다루는 내용:

1. 공격 시나리오의 원형 — 악성 xApp과 공유 데이터베이스
2. FGSM[^a-fgsm] / PGD[^a-pgd] — 수식과 O-RAN에서의 의미
3. Benzaïd 등의 ML(Machine Learning)[^a-ml] 위협 9범주 상세 (오염 → 회피 → 프라이버시 3종 → 탈취 → 재프로그래밍 → 공급망 → 자원고갈)
4. 시스템 수준 영향 — 처리량·BLER·SLA(Service Level Agreement)[^a-sla]
5. 방어 — 적대적 학습, 증류, SmoothAdv
6. 5G RAN 문헌에서 실제로 무엇이 연구되었나 (통계)

---

## 1. 공격 시나리오의 원형 — 악성 xApp

### 1.1 표적: InterClass xApp

Chiejina 등[^r8]은 **재머 탐지용 InterClass xApp** 두 종류를 구현했습니다.

| xApp | 입력 데이터 | 파이프라인 |
|---|---|---|
| **InterClass-Spec** | **스펙트로그램** | ① 버퍼에서 최근 **10 ms I/Q[^a-iq] 데이터** 수집(LTE[^a-lte]/5G 1 프레임 길이) → ② 보조 마이크로서비스가 I/Q → 스펙트로그램 변환 → ③ **RIC 데이터베이스에 저장** → ④ xApp이 DB 조회해 간섭 여부 판정 → ⑤ 정책 컨트롤러로 결정 전달 → ⑥ RAN에 제어 메시지 |
| **InterClass-KPM** | **KPM**(Key Performance Measurement)[^a-kpm] | 동일 흐름. ①에서 I/Q 대신 KPM을 indication request로 수신, ②에서 KPM 전처리 후 DB 저장 |

제어 액션은 구체적입니다: 재머가 없으면 **달성 가능한 최고 MCS[^a-mcs] 사용**, 재머가 탐지되면 완화 조치(적응형 MCS, 반송파 주파수 변경, 자원블록 기반 지능형 스케줄링). 논문은 **적응형 MCS**를 완화 방식으로 채택합니다[^r8].

![간섭 분류 xApp 개요 (악성 xApp 포함) (출처: Chiejina 등[^r8], Fig. 2)](/assets/img/posts/6g-ai-ran/sysadv-fig2.png)
_그림 6-1. **InterClass xApp과 악성 xApp의 공존**. 정상 xApp과 악성 xApp이 **같은 near-RT RIC, 같은 RIC 데이터베이스**를 공유합니다. 출처: [^r8], Fig. 2._

![단순화된 O-RAN 아키텍처 (출처: Chiejina 등[^r8], Fig. 1)](/assets/img/posts/6g-ai-ran/sysadv-fig1.png)
_그림 6-2. 실험에 사용된 단순화 O-RAN 아키텍처. 출처: [^r8], Fig. 1._

![KPM 모델 아키텍처 (출처: Chiejina 등[^r8], Fig. 3)](/assets/img/posts/6g-ai-ran/sysadv-fig3.png)
_그림 6-3. InterClass-KPM 모델 아키텍처. 출처: [^r8], Fig. 3._

### 1.2 위협 모델

| 항목 | 내용 |
|---|---|
| **표적** | near-RT RIC 내 xApp의 **ML 모델이 추론에 사용하는 테스트 데이터**[^r8] |
| **공격자** | 공유 near-RT RIC에 공존하는 **악성 xApp**. O-RAN의 개방성으로 **동일 DB 접근 가정** |
| **행동** | 테스트 시점 또는 **실시간**으로 공유 DB의 데이터를 조작 |
| **지식** | **White-box** — 모델에 대한 완전한 지식·무제한 접근. 논문은 이것이 *"O-RAN 아키텍처에 내재한 취약성과 개방성에 부합하는 매우 실용적인 접근"* 이라고 명시[^r8] |
| **목표** | 재머 신호가 있어도 **SOI(Signal of Interest)[^a-soi]로 분류**하게 만들기 (targeted attack) |
| **영향 지표** | **처리량(throughput)** 과 **BLER**(Block Error Rate)[^a-bler] |

> **"White-box가 실용적"이라는 주장에 주의하십시오.** 일반 ML 보안에서 white-box는 최악 가정이지만, O-RAN에서는 ① 모델이 오픈소스 xApp일 수 있고, ② 규격이 공개이며, ③ 같은 RIC에 악성 xApp이 존재할 수 있으므로 **현실적 가정**입니다. Ch4의 "표준의 개방성이 gray-box 지식을 무료로 제공한다"의 극단 사례입니다.
{: .prompt-danger }

---

## 2. FGSM과 PGD — 수식과 의미

### 2.1 FGSM (Fast Gradient Sign Method)

1-step 그래디언트 기반 공격입니다. 표적 라벨 $y_{target}$에 대한 손실 $\mathcal{L}(\boldsymbol{\theta}, \boldsymbol{x}, y_{target})$을 **최소화**하는 방향으로 섭동을 만듭니다[^r8].

$$
\boldsymbol{\delta} = \epsilon \cdot \mathrm{sign}\!\big(\nabla_{\boldsymbol{x}} \mathcal{L}(\boldsymbol{\theta}, \boldsymbol{x}, y_{target})\big),
\qquad
\boldsymbol{x}_{adv} = \boldsymbol{x} + \boldsymbol{\delta}
$$

| 기호 | 의미 |
|---|---|
| $\boldsymbol{\theta}$ | 모델 파라미터 |
| $\boldsymbol{x}$ | 모델 입력 (스펙트로그램 또는 KPM 벡터) |
| $y_{target}$ | 공격자가 유도하려는 표적 라벨 (여기서는 "SOI") |
| $\epsilon$ | **공격의 전력 스케일링 인자** — 원본과 유사성을 유지하면서 입력을 얼마나 바꿀 수 있는지 통제 |

### 2.2 PGD (Projected Gradient Descent)

FGSM을 작은 스텝 $\alpha$로 반복하고, 매 스텝마다 원본 주변 $L_\infty$-ball로 투영하는 다단 반복 공격입니다[^r8].

$$
\boldsymbol{x}_0 = \boldsymbol{x}, \qquad
\boldsymbol{x}_n = \mathrm{clip}_{[\boldsymbol{x},\epsilon]}\Big\{\boldsymbol{x}_{n-1} + \alpha\,\mathrm{sign}\big(\nabla_{\boldsymbol{x}_{n-1}} \mathcal{L}(\boldsymbol{\theta}, \boldsymbol{x}_{n-1}, \boldsymbol{y})\big)\Big\},
\qquad
\boldsymbol{x}_{adv} = \boldsymbol{x}_N
$$

$\mathrm{clip}_{[\boldsymbol{x},\epsilon]}(\cdot)$는 원소별로 $[\boldsymbol{x}-\epsilon,\ \boldsymbol{x}+\epsilon]$로 잘라내어 결과가 $\boldsymbol{x}$의 $L_\infty$ $\epsilon$-근방에 머물게 합니다.

> **$\epsilon$이 곧 은밀성입니다.** RAN 문맥에서 $\epsilon$이 작다는 것은 "KPM 보고값이 정상 범위 안에 있다"는 뜻이므로, **범위 검사(range check)** 기반 입력 검증으로는 탐지되지 않습니다. 그래서 Ch8의 XAI(eXplainable AI)[^a-xai] 기반 방어(결정 근거를 검사)가 필요합니다.
{: .prompt-tip }

---

## 3. ML 위협 9범주 상세

Ch4에서 개요를 잡은 9범주를 Benzaïd 등[^r5]의 O-RAN 특화 그림과 함께 봅니다. 각 그림은 **공격 전략 → 필요 능력(Cx/Lx) → 표적 학습 패러다임(ML/DL[^a-dl], RL[^a-rl]/DRL[^a-drl], FL[^a-fl])** 을 매핑한 마인드맵입니다.

### 3.1 ① 오염 공격 (Poisoning)

**Causative** 공격 — (재)학습 단계를 표적으로 무결성 또는 가용성을 침해합니다. 공격자는 학습 데이터나 학습 알고리즘을 변경해 **결정 경계를 자신에게 유리하게 편향**시킵니다[^r5].

![오염 공격 전략 (출처: Benzaïd 등[^r5], Fig. 6)](/assets/img/posts/6g-ai-ran/aisurvey-fig6.png)
_그림 6-4. 오염 공격 전략의 분류. 출처: [^r5], Fig. 6._

| 형태 | 방법 | 한계 |
|---|---|---|
| **Label-flipping** | 소스 클래스 라벨을 표적 클래스로 변경, 특정 클래스 또는 전체의 분류 오류 최대화. 라벨링 함수에 대한 통제 필요(단, 오염 샘플 예산 제한) | **데이터 정제·재라벨링으로 쉽게 필터링** |
| Clean-label | 원본을 바꾸지 않고 정교히 조작된 오염 샘플 주입 | 탐지 난이도 높음 |
| Algorithm corruption | 학습 알고리즘·로직 자체 오염 | C5(SW 통제) 필요 |

**O-RAN 시나리오**: E2 보고를 조작해 이상 탐지 xApp의 재학습 데이터를 오염 → 특정 공격 패턴을 "정상"으로 학습시킴. Ch3에서 언급한 **재학습 트리거 악용**(성능 저하 유발 → 재학습 → 오염 주입)이 결합되면 강력합니다.

![6G 데이터 소스를 이용한 모델 오염 시나리오 (출처: Braeken 등[^r36], Fig. 3)](/assets/img/posts/6g-ai-ran/aisecland-fig3.png)
_그림 6-4b. **6G 데이터 소스에서의 모델 오염 시나리오**. ISAC(Integrated Sensing and Communication)[^a-isac] 센싱·단말(드론·위성·차량·XR)에서 온 원시 데이터에 오염 샘플을 정교히 주입 → 중앙집중(Centralized) 또는 연합(Federated) 학습 경로 모두에서 오류가 전파 → 최종적으로 오염된 모델이 특정 입력에서만 오분류·백도어를 발현합니다. 출처: [^r36], Fig. 3._

Braeken 등[^r36]은 이를 **위협 카드**(공격자/표적/영향/위치) 형식으로 정량화합니다.

| 항목 | 내용 |
|---|---|
| **공격자** | 단말(IoT 센서·자율주행차·UAV·XR 헤드셋·위성)을 침해한 외부 공격자 또는 정당한 접근권을 가진 내부자. **비표적 공격은 FL 클라이언트의 20~30%**, **표적 공격은 1개 클라이언트 이상**을 침해 가능(비잔틴 결함허용 가정과 일치). 클라이언트와 중앙 서버를 동시에 침해할 수는 없음 |
| **결과** | 비표적 → 수렴 실패·성능 저하로 인한 DoS. 표적 → **백도어 트리거**, 특정 입력에서만 오분류, 정상처럼 보이는 오염된 거동 → 신뢰 상실 |
| **위치** | Device 계층(로컬 데이터 조작을 통한 백도어 이식), Edge 계층(FL 의사결정) |

_표 6-0. 오염 공격 위협 카드. 출처: Braeken 등[^r36]을 재구성._

### 3.2 ② 회피 공격 (Evasion)

**Exploratory** 공격 — 추론 단계에서 입력을 조작해 오분류를 유발합니다. §1~§2의 InterClass 공격, Ch5의 APATE가 모두 이 범주입니다.

![회피 공격 전략 (출처: Benzaïd 등[^r5], Fig. 7)](/assets/img/posts/6g-ai-ran/aisurvey-fig7.png)
_그림 6-5. 회피 공격 전략의 분류. 출처: [^r5], Fig. 7._

### 3.3 ③④⑤ 프라이버시 3종 — 멤버십 추론 · 속성 추론 · 데이터 재구성

기밀성을 표적으로 하며, **Ch9(프라이버시 보존 AI)** 의 직접적 동기입니다.

![멤버십 추론 공격 전략 (출처: Benzaïd 등[^r5], Fig. 8)](/assets/img/posts/6g-ai-ran/aisurvey-fig8.png)
_그림 6-6. 멤버십 추론 공격 전략 — "이 샘플이 학습 데이터에 포함되었는가?"를 판정. 출처: [^r5], Fig. 8._

![데이터 속성 추론 공격 전략 (출처: Benzaïd 등[^r5], Fig. 9)](/assets/img/posts/6g-ai-ran/aisurvey-fig9.png)
_그림 6-7. 데이터 속성 추론 공격 전략. 출처: [^r5], Fig. 9._

![정직하지만-호기심 많은 FL 서버가 데이터를 노출하는 데이터 재구성 공격 예시 (출처: Benzaïd 등[^r5], Fig. 10)](/assets/img/posts/6g-ai-ran/aisurvey-fig10.png)
_그림 6-8. **데이터 재구성 공격** — *honest-but-curious* FL 서버가 클라이언트 데이터를 복원하는 시나리오. Ch3에서 본 "Near-RT RIC에 FL 에이전트, Non-RT RIC에 집계 서버" 구조가 정확히 이 위협에 노출됩니다. 출처: [^r5], Fig. 10._

![데이터 재구성 공격 전략 (출처: Benzaïd 등[^r5], Fig. 11)](/assets/img/posts/6g-ai-ran/aisurvey-fig11.png)
_그림 6-9. 데이터 재구성 공격 전략의 분류. 출처: [^r5], Fig. 11._

> **UE-NIB와 결합하면 심각합니다.** Ch2에서 확인했듯 UE-NIB(UE Network Information Base)[^a-ue-nib]는 개별 UE 식별자를 보관하고, Soltani 등[^r6]의 **V-02**는 "UE 식별자 비인가 접근"을 심각도 High로 분류합니다. 여기에 데이터 재구성 공격이 성공하면 **가입자 위치 이력까지 복원**될 수 있습니다.
{: .prompt-danger }

Braeken 등[^r36]은 멤버십 추론의 표적을 NWDAF(Network Data Analytics Function)[^a-nwdaf](NEF(Network Exposure Function)[^a-nef] 경유), 엣지 러닝(EL), SemCom(Semantic Communication)[^a-semcom], FL, LLM, GNN(Graph Neural Network)[^a-gnn](네트워크 슬라이싱·디지털 트윈·매시브 IoT)까지 넓히고, 4가지 대응책을 제시합니다 — **신뢰 점수 마스킹**(Confidence Score Masking, 모델이 반환하는 실제 신뢰 점수를 가림), **정규화**(데이터 증강·L1/L2·드롭아웃·조기 종료로 과적합 감소), **지식 증류**(교사 모델의 출력만으로 학생 모델을 학습시켜 학습 데이터 암기 방지), **차분 프라이버시(DP)**(이론적 프라이버시 보장으로 특정 사용 세부사항을 기억하지 못하게 함).

### 3.4 ⑥ 모델 탈취 (Model Stealing)

![모델 탈취 공격 전략 (출처: Benzaïd 등[^r5], Fig. 12)](/assets/img/posts/6g-ai-ran/aisurvey-fig12.png)
_그림 6-10. 모델 탈취 공격 전략. 출처: [^r5], Fig. 12._

**O-RAN 특유의 위험**: Ch3에서 본 **model sharing** — RIC의 rogue BS(Base Station)[^a-bs] 탐지 모델을 UE로 배포하는 구조[^r7] — 는 공격자에게 모델을 직접 건네줍니다. 또한 APATE[^r9]의 1단계(대리 모델 학습)는 모델 탈취의 실용적 변형입니다.

Braeken 등[^r36]은 공격자를 **쿼리 기반**(정보 추출을 위한 반복 질의)과 **사이드채널 기반**(타이밍·전자기 신호, 엣지 GPU 파라미터 유출)으로 나누고, 표적을 MLaaS(Machine Learning as a Service) 플랫폼·O-RAN의 xApp/rApp·엣지 GPU까지 확장합니다. 대응책은 **선택적 오정보**(분포 밖 쿼리에 의도적으로 잘못된 예측을 반환해 쿼리 기반 탈취를 무력화), **기만적 섭동**(출력 확률을 미세하게 왜곡), **ADD**(Account-Aware Distribution Discrepancy — 계정별 쿼리 패턴의 지역적 의존성을 근거로 공격 쿼리를 식별)입니다.

### 3.5 ⑦ 모델 재프로그래밍 (Model Reprogramming)

가장 O-RAN 문맥에서 인상적인 위협입니다 — **모델의 용도 자체를 바꿔버립니다.**

![모델 재프로그래밍 공격 전략 (출처: Benzaïd 등[^r5], Fig. 13)](/assets/img/posts/6g-ai-ran/aisurvey-fig13.png)
_그림 6-11. 모델 재프로그래밍 공격 전략. 출처: [^r5], Fig. 13._

![간섭 분류기를 다른 용도로 재활용하는 신경망 재프로그래밍 공격 예시 (출처: Benzaïd 등[^r5], Fig. 14)](/assets/img/posts/6g-ai-ran/aisurvey-fig14.png)
_그림 6-12. **신경망 재프로그래밍 공격** — 간섭 분류기(interference classifier)를 **공격자가 원하는 다른 작업으로 용도 변경**하는 예시. 즉 사업자 인프라의 모델·연산자원을 공격자가 무단 전용합니다. 출처: [^r5], Fig. 14._

### 3.6 ⑧ ML 공급망 공격 (ML Supply Chain)

Ch4 §6에서 표면을 짚었고, 여기서는 전략과 구체 시나리오를 봅니다.

![ML 공급망 공격 전략 (출처: Benzaïd 등[^r5], Fig. 15)](/assets/img/posts/6g-ai-ran/aisurvey-fig15.png)
_그림 6-13. ML 공급망 공격 전략의 분류. 출처: [^r5], Fig. 15._

![트로이목마 임베딩 위협 예시 (출처: Benzaïd 등[^r5], Fig. 16)](/assets/img/posts/6g-ai-ran/aisurvey-fig16.png)
_그림 6-14. 트로이목마 임베딩 위협. 출처: [^r5], Fig. 16._

![침해된 의존성을 통한 트로이목마 임베딩 — 시그널링 스톰 탐지를 속이는 예시 (출처: Benzaïd 등[^r5], Fig. 19)](/assets/img/posts/6g-ai-ran/aisurvey-fig19.png)
_그림 6-15. **침해된 의존성을 통한 트로이목마 임베딩**으로 시그널링 스톰 탐지를 무력화하는 시나리오. Ch5 §3의 SSA(Signaling Storm Attack)[^a-ssa]와 결합하면 "탐지기가 침묵하는 동안 진행되는 제어 평면 DDoS(Distributed Denial of Service)[^a-ddos]"가 됩니다. 출처: [^r5], Fig. 19._

![AI 가속 하드웨어의 비신뢰 주체를 통한 하드웨어 트로이목마 임베딩 예시 (출처: Benzaïd 등[^r5], Fig. 20)](/assets/img/posts/6g-ai-ran/aisurvey-fig20.png)
_그림 6-16. **하드웨어 트로이목마** — AI(Artificial Intelligence)[^a-ai] 가속 하드웨어(GPU[^a-gpu] 등) 공급 경로의 비신뢰 주체를 통한 임베딩. Ch4의 능력 **C6**(Hardware Control)에 해당하며, 소프트웨어 방어로 막을 수 없습니다 → Ch9의 TEE(Trusted Execution Environment)[^a-tee]·원격 검증(attestation)이 필요한 이유. 출처: [^r5], Fig. 20._

![6G에서의 AI 공급망 공격 (출처: Braeken 등[^r36], Fig. 4)](/assets/img/posts/6g-ai-ran/aisecland-fig4.png)
_그림 6-16b. **AI 공급망 공격 파이프라인**. 데이터셋(텔레메트리·CSI 오염, 허위 KPI 주입) → ML 모델 학습(오염되거나 백도어가 통합된 사전학습 xApp/rApp) → 모델 허브 배포(변조된 RIC·ML 파이썬 라이브러리, 침해된 O-RAN 모델 허브·SDK) → ML 프로덕션 모델(RAN 엣지의 부실 검증 모델) → 모델 추론(DoS, QoS/KPI 위반, 채널 데이터 오분류, IDS의 프라이버시·무결성 손실로 귀결). 출처: [^r36], Fig. 4._

Braeken 등[^r36]은 **Hugging Face** 플랫폼을 구체적으로 평가한 연구를 인용하며, 안전하지 않은 직렬화 기법 때문에 다수 모델이 **객체 주입(object injection) 취약점**에 노출되어 있음을 지적합니다. 사전학습 모델에 심긴 백도어는 **파인튜닝 이후에도 AI 공급망을 통해 전파**될 수 있어, Ch10의 SMO(Service Management and Orchestration)[^a-smo] 역시 공급망 공격의 표적이 될 수 있다고 경고합니다. 대응책은 데이터부터 프로덕션 모델까지 **AI 모델 의존성 추적**, 아티팩트 무결성 검증, 모델 허브에 GPG 키 등 **추가 보안 계층 통합**입니다.

### 3.7 ⑨ 자원 고갈 (Resource Exhaustion)

![자원 고갈 공격 전략 (출처: Benzaïd 등[^r5], Fig. 21)](/assets/img/posts/6g-ai-ran/aisurvey-fig21.png)
_그림 6-17. 자원 고갈 공격 전략. 출처: [^r5], Fig. 21._

![간섭의 적시 탐지·완화를 방해하는 자원 고갈 공격 예시 (출처: Benzaïd 등[^r5], Fig. 22)](/assets/img/posts/6g-ai-ran/aisurvey-fig22.png)
_그림 6-18. **자원 고갈 공격이 간섭의 적시 탐지·완화를 방해**하는 시나리오. 탐지기를 무력화하지 않고 **느리게** 만드는 것으로도 near-RT 예산(10ms~1s)을 넘기면 방어는 실패합니다. 출처: [^r5], Fig. 22._

> **AI-RAN에서 자원 고갈이 특별히 위험한 이유**: Ch2에서 본 **AI and RAN 공존** 구조에서는 AI 워크로드와 RAN 워크로드가 GPU를 공유합니다. 공격자가 정당한 AI 워크로드로 위장해 가속기를 점유하면 ① 탐지 xApp의 추론 지연, ② RAN DSP(Digital Signal Processing)[^a-dsp]의 타이밍 위반, ③ **EDoS**[^a-edos](Economical DoS[^a-dos] — 과금·전력 소모) 를 동시에 달성합니다[^r5], [^r2].
{: .prompt-danger }

### 3.8 ⑩ LLM 오남용과 환각 (Cross-Layer LLM Misuse and Hallucinations)

Braeken 등[^r36]은 생성형 AI의 하위 범주로 **LLM 오남용**을 별도 카테고리로 다룹니다. O-RAN xApp/rApp에 LLM이 통합되는 사례가 이미 등장했습니다 — 오프라인 학습 없이 네트워크 슬라이스 전반의 적응형 무선자원관리를 수행하는 LLM, 동일 목적의 rApp으로서의 LLM이 그것입니다.

| 항목 | 내용 |
|---|---|
| **공격자** | LLM 보안 메커니즘을 탈옥(jailbreak)시키거나 악성 프롬프트를 주입하는 공격자, 자원고갈 DoS를 유발하는 공격자, 환각을 유발하는 입력을 제공하는 사용자·시스템 |
| **표적** | 자원할당·정책생성용 O-RAN xApp/rApp에 통합된 LLM, 네트워크 슬라이싱 인프라, LLM 기반 human-in-the-loop 시스템, 자율 의사결정 |
| **결과** | 네트워크 기능·서비스 붕괴, 슬라이스 QoS 저하, **환각 응답에서 비롯된 잘못된 human-in-the-loop 조치**, 자율 의사결정 훼손, 연산 고갈로 인한 모델 가용성 저하 |
| **위치** | RAN·Edge·Application 계층, O-RAN 인프라 |

_표 6-13b. LLM 오남용·환각 위협 카드. 출처: Braeken 등[^r36]을 재구성._

방어책은 Ch5 §6에서 다룬 가드레일과 상보적입니다 — **RAG(Retrieval-Augmented Generation)**[^a-rag]를 신뢰할 수 있는 최신 지식corpus와 결합, 학습·파인튜닝 corpus 정제, 추론 시 **입력 프롬프트 전처리**로 악성 입력 제거, 생성된 정책을 xApp/rApp에 전달하기 전 **사전 정의된 안전 체크리스트와 대조하는 후처리 검증**입니다[^r36]. Ch5 §6.4의 Formal Certification·Digital Twin Verification·LLM-Agent Guards가 바로 이 후처리 검증의 구체적 실현 형태입니다.

---

## 4. 시스템 수준 영향

### 4.1 InterClass xApp — 정확도 vs $\epsilon$

![InterClass-Spec xApp의 공격·방어별 정확도 대 epsilon 비교 (출처: Chiejina 등[^r8], Fig. 7)](/assets/img/posts/6g-ai-ran/sysadv-fig7.png)
_그림 6-19. **InterClass-Spec xApp** — 공격 종류(FGSM/PGD)와 방어별 정확도를 $\epsilon$에 대해 비교. 방어 없이는 정확도가 **0%까지** 떨어집니다. 출처: [^r8], Fig. 7._

![InterClass-KPM xApp의 공격·방어별 정확도 대 epsilon 비교 (출처: Chiejina 등[^r8], Fig. 8)](/assets/img/posts/6g-ai-ran/sysadv-fig8.png)
_그림 6-20. **InterClass-KPM xApp**의 동일 비교. 출처: [^r8], Fig. 8._

| 관측 | 의미 |
|---|---|
| InterClass-Spec 정확도 → **0%** | 재머가 있어도 항상 "간섭 없음"으로 판정 → RAN이 최고 MCS를 유지 → **처리량 붕괴 + BLER 급증** |
| PGD > FGSM | 반복 최적화가 더 작은 $\epsilon$으로 더 큰 피해 |

### 4.2 AI 기반 RAN 슬라이싱 — SLA 위반과 회복

Tashman & Cherkaoui[^r10]는 **예산 제약 공격자(budget-constrained adversary)** 가 슬라이스 전송을 선택적으로 재밍해 **DRL 기반 자원 할당을 편향**시키는 상황을 정량화합니다.

![시스템 모델 (출처: Tashman & Cherkaoui[^r10], Fig. 1)](/assets/img/posts/6g-ai-ran/slicingadv-fig1.png)
_그림 6-21. AI 기반 RAN 슬라이싱 시스템 모델 — eMBB[^a-embb] / mMTC[^a-mmtc] / URLLC[^a-urllc] 슬라이스에 대한 DRL 자원 할당. 출처: [^r10], Fig. 1._

![공격 없음과 DRL 기반 적대적 공격 하의 슬라이스별 평균 SLA 위반율 (출처: Tashman & Cherkaoui[^r10], Fig. 2)](/assets/img/posts/6g-ai-ran/slicingadv-fig2.png)
_그림 6-22. **슬라이스 수준 평균 SLA 위반율** — 무공격 vs DRL 기반 적대적 재밍. 출처: [^r10], Fig. 2._

![RL 기반 적대적 공격 후 슬라이스별 SLA 회복 거동 (출처: Tashman & Cherkaoui[^r10], Fig. 3)](/assets/img/posts/6g-ai-ran/slicingadv-fig3.png)
_그림 6-23. **공격 후 SLA 회복 거동**. DRL 에이전트의 보상이 무공격 기준선으로 수렴하기까지 **무시할 수 없는 회복 기간(recovery period)** 이 필요합니다. 출처: [^r10], Fig. 3._

| 결론[^r10] | 보안적 함의 |
|---|---|
| 예산 제약 적대적 재밍이 **심각하고 슬라이스 의존적인 정상상태 SLA 위반**을 유발 | 공격 예산이 작아도 특정 슬라이스(예: URLLC)를 표적화하면 치명적 |
| 보상이 기준선으로 수렴하기까지 **비무시적 회복 기간** 필요 | **공격이 끝난 뒤에도 피해가 지속** → 반복적 짧은 공격으로 상시 열화 유지 가능 |

> **"회복 기간"이 새로운 공격 벡터입니다.** 학습형 제어기는 공격 후에도 잘못된 정책에서 빠져나오는 데 시간이 걸립니다. 공격자는 **회복 기간보다 짧은 주기로 공격을 반복**하면 최소 비용으로 영구적 성능 저하를 유지할 수 있습니다. 이는 Ch4의 전략 **S6(Temporal/Strategic)** 의 또 다른 사례입니다.
{: .prompt-danger }

---

## 5. 방어

### 5.1 기준선: 적대적 학습 (Adversarial Training)

Chiejina 등[^r8]의 baseline 방어입니다. 학습 시 적대적 샘플을 포함시켜 결정 경계를 강건화합니다. 한계는 잘 알려져 있습니다 — 학습에 사용한 공격 유형·$\epsilon$ 범위 밖에서는 성능이 떨어지고, 정상 정확도(clean accuracy)를 희생합니다.

### 5.2 증류 기반 방어 (Distillation-based Defense)

![증류 기반 방어 개요 (출처: Chiejina 등[^r8], Fig. 5)](/assets/img/posts/6g-ai-ran/sysadv-fig5.png)
_그림 6-24. **증류 기반 방어**. 교사 모델의 소프트 출력을 학생 모델에 전달하여 그래디언트를 평탄화, 그래디언트 기반 공격(FGSM/PGD)의 효과를 줄입니다. 출처: [^r8], Fig. 5._

### 5.3 SmoothAdv — O-RAN xApp의 PGD 강건성

Agarwal[^r27]은 **Smooth Adversarial Training**을 O-RAN xApp에 적용해 PGD 공격에 대한 강건성 증대를 연구했습니다.

> **자료 제약 안내**: 본 시리즈 참조 폴더의 해당 문헌은 **ProQuest PREVIEW 버전**으로 표지·목차·그림목록만 포함되어 있습니다(본문·참고문헌 미수록). 따라서 본 절은 제목·연구 주제 수준의 인용으로 제한합니다.
{: .prompt-info }

### 5.4 방어 선택 가이드

| 위협 범주 | 1차 방어 | 보완 방어 | 상세 |
|---|---|---|---|
| Poisoning | 데이터 정제·재라벨링, 강건 집계(FL) | 데이터 출처 검증, AIBOM(AI Bill of Materials)[^a-aibom] | Ch7 |
| Evasion | 적대적 학습, 증류, **입력 정화** | **MTD**[^a-mtd](모델 다양화), XAI 근거 검사 | Ch7, Ch8 |
| 프라이버시 3종 | **DP[^a-dp], HE[^a-he], FE**(PETs[^a-pets]) | 데이터 최소화, UE-NIB 접근통제 | **Ch9** |
| Model Stealing | 쿼리 레이트 리밋, 출력 라운딩 | 워터마킹, model sharing 재검토 | Ch7 |
| Reprogramming | 입력 도메인 검증 | **MTD, AIBOM** | Ch7 |
| Supply Chain | **AIBOM** | 서명된 모델·컨테이너, 재현 가능 빌드 | **Ch7** |
| Resource Exhaustion | 자원 쿼터·격리(namespace) | 지속 모니터링, RAN 워크로드 선점 | Ch7, Ch11 |
| Hardware Trojan | — (SW로 불가) | **TEE, 원격 검증(attestation)** | **Ch9** |

---

## 6. 5G RAN 문헌에서 실제로 무엇이 연구되었나

Benzaïd 등[^r5]은 5G RAN 도메인의 적대적 ML 연구를 체계적으로 집계했습니다. **연구가 몰린 곳과 비어 있는 곳**을 보여주므로 연구 주제 선정에 유용합니다.

![5G RAN 연구에서의 적대적 위협 분포 (출처: Benzaïd 등[^r5], Fig. 23)](/assets/img/posts/6g-ai-ran/aisurvey-fig23.png)
_그림 6-25. **적대적 위협 유형의 분포**. 출처: [^r5], Fig. 23._

![5G RAN 연구에서 채택된 적대적 공격 전략 분포 (출처: Benzaïd 등[^r5], Fig. 24)](/assets/img/posts/6g-ai-ran/aisurvey-fig24.png)
_그림 6-26. **채택된 공격 전략의 분포**. 출처: [^r5], Fig. 24._

![적대적 위협의 표적이 된 학습 방식 분포 (출처: Benzaïd 등[^r5], Fig. 25)](/assets/img/posts/6g-ai-ran/aisurvey-fig25.png)
_그림 6-27. **표적이 된 학습 방식(ML/DL, RL/DRL, FL)의 분포**. 출처: [^r5], Fig. 25._

Benzaïd 등[^r5]이 5G RAN의 ML 보안 연구를 정리한 응용 도메인은 다음과 같습니다.

| 응용 도메인 | 대표 표적 |
|---|---|
| Wireless Signal Classification | 변조 분류기, 간섭 분류기 |
| Physical-Layer Authentication | RF(Radio Frequency)[^a-rf] 지문 기반 인증 |
| Power Allocation | 전력 제어 DRL |
| Beam Selection | 빔 선택 모델 |
| **Network Slicing** | 슬라이스 자원 할당 DRL[^r10] |
| **Network Security** | IDS(Intrusion Detection System)[^a-ids]·이상탐지 모델 자체 |

> **연구 공백(research gap)**: Benzaïd 등[^r5]은 미래 연구 방향에서 **체인 모델(chained models)의 AI 보안**이 미탐구 영역이라고 지적합니다. O-RAN Alliance는 모듈성·독립 진화·재사용을 위해 **모델 체이닝을 베스트 프랙티스로 권장**하지만(예: RF 신호강도 예측 모델 + 셀 사용률 예측 모델 → QoE(Quality of Experience)[^a-qoe] 예측 모델), *"모델 간 예측 오차 증폭은 잘 알려져 있으나 **보안 함의는 미탐구 상태**이며, 지금까지 적대적 공격은 단일 모델에 대해서만 연구되었다. 그러나 모델 체이닝에서는 한 모델에 대한 공격이 상호연결된 모델들에 **연쇄 효과(cascading effect)** 를 일으킬 수 있다"* 고 명시합니다. Ch12에서 이 주제를 다시 다룹니다.
{: .prompt-tip }

---

## 7. 이 장의 요약

- 공격 원형은 **같은 near-RT RIC의 악성 xApp이 공유 RIC DB를 조작**하는 것입니다. O-RAN의 개방성이 white-box 가정을 현실화합니다[^r8].
- **FGSM**($\boldsymbol{\delta} = \epsilon\,\mathrm{sign}(\nabla_x\mathcal{L})$)과 **PGD**(반복 + $L_\infty$ 투영)로 InterClass-Spec xApp의 정확도가 **0%** 까지 떨어집니다 — 그 결과는 처리량 붕괴와 BLER 급증입니다.
- ML 위협은 **9범주**: 오염 · 회피 · 멤버십추론 · 속성추론 · 데이터재구성 · 모델탈취 · **재프로그래밍** · **공급망** · **자원고갈**[^r5].
- **재프로그래밍**은 사업자 모델·연산자원을 공격자 용도로 전용하고, **하드웨어 트로이목마**는 소프트웨어 방어로 막을 수 없습니다(→ Ch9 TEE).
- RAN 슬라이싱에 대한 **예산 제약 적대적 재밍**은 슬라이스별 SLA 위반을 유발하고, **회복 기간이 새로운 공격 벡터**가 됩니다[^r10].
- 방어는 단일 기법이 아니라 위협 범주별 조합이며, 그 통합 프레임이 **ZT-AI Shield**(Ch7)입니다.
- Braeken 등[^r36]은 위협을 **공격자/표적/영향/위치의 "위협 카드"** 형식으로 정량화합니다 — 오염 공격은 FL 클라이언트의 **20~30%**(비표적) 또는 **1개 이상**(표적) 침해를 전제하며, 멤버십 추론·모델 탈취에는 각각 신뢰점수 마스킹·지식증류·DP, 선택적 오정보·ADD라는 구체적 대응책이 있습니다.
- **LLM 오남용·환각**은 Braeken 등[^r36]이 추가한 10번째 위협 범주로, O-RAN xApp/rApp에 통합된 LLM의 탈옥·프롬프트 인젝션·자원고갈 DoS·환각을 다루며 RAG·corpus 정제·프롬프트 전후처리로 방어합니다.

### 확인 체크리스트

- [ ] FGSM과 PGD의 차이를 수식으로 설명할 수 있는가
- [ ] $\epsilon$이 왜 "은밀성"과 직결되는지, 범위 검사가 왜 무력한지 설명할 수 있는가
- [ ] 9범주를 Goal(무결성/가용성/기밀성)로 분류할 수 있는가
- [ ] 모델 재프로그래밍이 다른 공격과 무엇이 다른지 설명할 수 있는가
- [ ] "회복 기간"이 공격 벡터가 되는 논리를 설명할 수 있는가
- [ ] 체인 모델의 연쇄 효과가 왜 미탐구 영역인지 설명할 수 있는가

**다음 장**: [07. 6G RAN을 위한 Zero Trust Architecture](/posts/airan-07-zero-trust/) — Part III 시작

---

## 약어

[^a-o-ran]: **O-RAN**(Open Radio Access Network): RAN을 개방형 인터페이스로 분해해 다중 벤더 구성과 AI 기반 제어를 가능하게 하는 아키텍처입니다.
[^a-usrp]: **USRP**(Universal Software Radio Peripheral): 소프트웨어로 무선 규격을 정의해 송수신하는 SDR 장비로, 대학·연구소의 O-RAN 테스트베드 구축에 널리 쓰입니다.
[^a-ran]: **RAN**(Radio Access Network): 단말과 코어망을 무선으로 연결하는 기지국 계층의 접속망입니다.
[^a-ue]: **UE**(User Equipment): 이동통신망에 접속하는 사용자 단말을 가리키는 3GPP 표준 용어입니다.
[^a-ric]: **RIC**(RAN Intelligent Controller): RAN을 지능적으로 제어·최적화하는 O-RAN의 핵심 컨트롤러로, Near-RT RIC(10ms~1s)과 Non-RT RIC(1s 이상)으로 나뉩니다.
[^a-fgsm]: **FGSM**(Fast Gradient Sign Method): 손실 함수의 그래디언트 부호 방향으로 한 번에 섭동을 더해 오분류를 유도하는 1-step 적대적 공격입니다.
[^a-pgd]: **PGD**(Projected Gradient Descent): FGSM을 작은 보폭으로 반복하고 매 단계마다 원본 주변 $\epsilon$-근방으로 투영하는 다단 반복 공격입니다.
[^a-ml]: **ML**(Machine Learning): 데이터에서 규칙을 학습해 예측·분류를 수행하는 기법의 총칭입니다.
[^a-sla]: **SLA**(Service Level Agreement): 지연·처리량·가용성 등 사업자가 보장하기로 약속한 서비스 수준 합의로, 위반 시 계약상 책임이 발생합니다.
[^a-iq]: **I/Q**(In-phase/Quadrature): 무선 신호를 동상 성분과 직교 성분의 복소 표본으로 표현한 형식으로, 스펙트로그램 등 물리계층 분석의 원자료가 됩니다.
[^a-lte]: **LTE**(Long Term Evolution): 3GPP가 규격화한 4세대 이동통신 기술로, 10 ms 무선 프레임 구조를 사용합니다.
[^a-kpm]: **KPM**(Key Performance Measurement): E2 인터페이스로 RIC에 보고되는 RAN 성능 측정값으로, xApp의 판단 입력이 됩니다.
[^a-mcs]: **MCS**(Modulation and Coding Scheme): 채널 상태에 따라 변조 차수와 부호율을 선택하는 방식으로, 잘못 높이면 오류율이 급증합니다.
[^a-soi]: **SOI**(Signal of Interest): 재밍·간섭이 아닌 정상적으로 수신하려는 대상 신호를 뜻합니다.
[^a-bler]: **BLER**(Block Error Rate): 전송 블록 중 복호에 실패한 비율로, 무선 링크 품질을 나타내는 핵심 지표입니다.
[^a-xai]: **XAI**(eXplainable AI): 모델이 그런 결정을 내린 근거를 사람이 이해할 수 있게 제시하는 설명가능 인공지능 기법입니다.
[^a-dl]: **DL**(Deep Learning): 다층 신경망으로 특징 표현까지 함께 학습하는 기계학습의 하위 분야입니다.
[^a-rl]: **RL**(Reinforcement Learning): 환경과 상호작용하며 받은 보상을 최대화하도록 정책을 학습하는 방법론입니다.
[^a-drl]: **DRL**(Deep Reinforcement Learning): 심층 신경망으로 상태를 표현하고 누적 보상을 최대화하는 정책을 학습하는 강화학습 기법입니다.
[^a-fl]: **FL**(Federated Learning): 원본 데이터를 모으지 않고 각 참여자가 학습한 모델 갱신만 집계해 공동 모델을 만드는 분산 학습 방식입니다.
[^a-ue-nib]: **UE-NIB**(UE Network Information Base): Near-RT RIC이 유지하는 단말 단위 정보 데이터베이스로, UE 식별자를 포함해 프라이버시 민감도가 높습니다.
[^a-bs]: **BS**(Base Station): 단말과 무선으로 접속해 코어망까지 연결해 주는 기지국입니다.
[^a-ssa]: **SSA**(Signaling Storm Attack): 대량의 시그널링 메시지를 유발해 제어 평면(RIC·코어망)의 처리 용량을 소진시키는 공격입니다.
[^a-ddos]: **DDoS**(Distributed Denial of Service): 다수의 분산된 단말이 동시에 요청을 보내 정상 서비스를 마비시키는 분산 서비스 거부 공격입니다.
[^a-ai]: **AI**(Artificial Intelligence): 학습·추론·판단 기능을 기계로 구현하는 기술 분야로, 6G에서는 RAN 제어와 워크로드 양쪽에 걸쳐 사용됩니다.
[^a-gpu]: **GPU**(Graphics Processing Unit): 대규모 병렬 연산에 특화된 가속기로, AI 워크로드와 RAN 신호처리가 함께 공유하는 자원입니다.
[^a-tee]: **TEE**(Trusted Execution Environment): 프로세서가 제공하는 격리 실행 영역으로, 코드·데이터를 상위 권한 소프트웨어로부터도 보호하고 원격 검증을 지원합니다.
[^a-dsp]: **DSP**(Digital Signal Processing): 무선 파형의 변복조·필터링 등을 디지털로 처리하는 연산으로, RAN에서는 엄격한 실시간 마감시간을 지켜야 합니다.
[^a-edos]: **EDoS**(Economic Denial of Sustainability): 서비스를 멈추는 대신 과도한 자원 사용·과금을 유발해 사업자의 경제적 지속가능성을 무너뜨리는 공격입니다.
[^a-dos]: **DoS**(Denial of Service): 자원이나 프로토콜 약점을 소진시켜 정상 사용자의 서비스 이용을 방해하는 서비스 거부 공격입니다.
[^a-embb]: **eMBB**(enhanced Mobile Broadband): 고속 대용량 데이터 전송을 목표로 하는 5G/6G 서비스 범주입니다.
[^a-mmtc]: **mMTC**(massive Machine Type Communications): 저전력 IoT 기기를 대규모로 수용하는 것을 목표로 하는 서비스 범주입니다.
[^a-urllc]: **URLLC**(Ultra-Reliable Low-Latency Communications): 초저지연·초고신뢰를 요구하는 서비스 범주로, 원격제어·자율주행 등이 해당합니다.
[^a-aibom]: **AIBOM**(AI Bill of Materials): 모델·데이터셋·라이브러리 등 AI 시스템의 구성요소와 출처를 명세한 목록으로, SBOM의 AI 확장입니다.
[^a-mtd]: **MTD**(Moving Target Defense): 모델·구성·경로를 계속 바꾸어 공격자가 확보한 사전 지식을 무력화하는 능동 방어 전략입니다.
[^a-dp]: **DP**(Differential Privacy): 통계적 잡음을 더해 개별 데이터의 포함 여부를 알 수 없게 만드는 수학적 프라이버시 보장 기법입니다.
[^a-he]: **HE**(Homomorphic Encryption): 암호문 상태 그대로 연산을 수행해 복호 없이 결과를 얻을 수 있게 하는 암호 기법입니다.
[^a-pets]: **PETs**(Privacy-Enhancing Technologies): 데이터 활용과 프라이버시 보호를 동시에 달성하기 위한 기술군의 총칭입니다.
[^a-rf]: **RF**(Radio Frequency): 무선 주파수 신호를 뜻하며, 송신기 고유의 미세한 하드웨어 특성은 RF 지문 인증에 활용됩니다.
[^a-ids]: **IDS**(Intrusion Detection System): 트래픽·행위를 분석해 침입 징후를 탐지하고 경보를 발생시키는 시스템입니다.
[^a-qoe]: **QoE**(Quality of Experience): 지연·처리량 같은 지표를 사용자가 실제로 체감하는 품질로 환산한 값입니다.
[^a-isac]: **ISAC**(Integrated Sensing and Communication): 통신 신호로 통신과 주변 환경 감지(레이더 유사 기능)를 동시에 수행하는 6G 핵심 기술입니다.
[^a-nwdaf]: **NWDAF**(Network Data Analytics Function): 5G/6G 코어망에서 네트워크 데이터를 수집·분석해 인사이트를 제공하는 기능입니다.
[^a-nef]: **NEF**(Network Exposure Function): 코어망의 능력·데이터를 외부 애플리케이션에 안전하게 노출하는 API 게이트웨이 기능입니다.
[^a-semcom]: **SemCom**(Semantic Communication): 비트 자체가 아니라 정보의 의미를 전달하는 것을 목표로 하는 차세대 통신 패러다임입니다.
[^a-gnn]: **GNN**(Graph Neural Network): 그래프 구조 데이터의 노드·엣지 관계를 학습하는 신경망으로, 네트워크 슬라이싱·토폴로지 분석에 쓰입니다.
[^a-smo]: **SMO**(Service Management and Orchestration): O-RAN에서 슬라이스·자원·정책을 관리·오케스트레이션하는 서비스 관리 프레임워크입니다.
[^a-rag]: **RAG**(Retrieval-Augmented Generation): 외부 지식베이스에서 관련 문서를 검색해 프롬프트에 덧붙여 생성 품질을 높이는 기법입니다.

---

## References

[^r2]: M. Polese, N. Mohamadi, S. D'Oro, L. Bonati, and T. Melodia, "Beyond connectivity: An open architecture for AI-RAN convergence in 6G," *arXiv preprint* arXiv:2507.06911v2, Dec. 2025.
[^r5]: C. Benzaïd, T. Taleb, R. Cavalcanti, and A. Sánchez, "AI in Open RAN security: Enabler and threat — A comprehensive survey and future perspectives," *TechRxiv preprint*, Dec. 2025.
[^r6]: S. Soltani, A. Amanloo, M. Shojafar, and R. Tafazolli, "Intelligent control in 6G open RAN: Security risk or opportunity?" *IEEE Open Journal of the Communications Society*, 2025.
[^r7]: N. M. Yungaicela-Naula, V. Sharma, and S. Scott-Hayward, "Misconfiguration in O-RAN: Analysis of the impact of AI/ML," *Computer Networks*, p. 110455, 2024.
[^r8]: A. Chiejina, B. Kim, K. Chowdhury, and V. K. Shah, "System-level analysis of adversarial attacks and defenses on intelligence in O-RAN based cellular networks," in *Proc. 17th ACM Conf. on Security and Privacy in Wireless and Mobile Networks (WiSec)*, Seoul, Korea, May 2024.
[^r9]: E. Aizikovich, D. Mimran, E. Grolman, Y. Elovici, and A. Shabtai, "Rogue cell: Adversarial attack and defense in untrusted O-RAN setup exploiting the traffic steering xApp," *arXiv preprint*, 2025.
[^r10]: D. H. Tashman and S. Cherkaoui, "Adversarial attacks in AI-driven RAN slicing: SLA violations and recovery," *arXiv preprint* arXiv:2604.01049, Apr. 2026.
[^r27]: D. Agarwal, "Smooth adversarial training for increasing robustness of O-RAN xApps against PGD attacks," M.S. thesis, Dept. of Electrical and Computer Engineering, Northeastern University, Boston, MA, USA, Dec. 2025.
[^r36]: A. Braeken, D. Deac, T. L. Nguyen, G. Gür, Q.-V. Pham, C. Yapa, P. G. Vinueza-Naranjo, H. Carvajal Mora, C. Moremada, and M. Liyanage, "6G AI security: From fundamentals to offensive and defensive landscape in 6G," *IEEE Communications Surveys & Tutorials*, vol. 28, 2026.
