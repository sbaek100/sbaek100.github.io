---
title: "[6G AI-RAN] 08. AI-Driven 자율 보안 탐지 및 Self-Healing RAN"
date: 2026-07-30 10:20:00 +0900
categories:
  - 2.미래기술
  - 6G AI RAN
  - Part III 방어·복원력
tags:
  - Intrusion-Detection
  - Anomaly-Detection
  - XAI
  - LIME
  - SHAP
  - Federated-Learning
  - Self-Healing
math: true
mermaid: false
---

# AI-Driven 자율 보안 탐지 및 Self-Healing RAN

## 들어가며 — 탐지 정확도보다 먼저 물어야 할 것

이 장은 "AI(Artificial Intelligence)[^a-ai]로 공격을 탐지한다"는 연구들을 다룹니다. 그런데 논문의 F1 스코어를 읽기 전에 반드시 확인해야 하는 것이 있습니다.

> **① 그 모델은 어느 시간 예산 안에서 동작하는가?**
> Near-RT RIC(RAN Intelligent Controller)[^a-ric]는 **10 ms ~ 1 s**, O-DU(Distributed Unit)[^a-du]는 그보다 더 엄격합니다[^r25]. F1 0.99인 LSTM(Long Short-Term Memory)[^a-lstm]이 추론에 2초 걸리면 **배포 불가**입니다.
> **② 그 탐지는 무엇을 근거로 판단했는지 설명할 수 있는가?**
> Ch6에서 본 대로 적대적 입력은 정상 범위 안에 있습니다. 근거를 검사할 수 없으면 조작된 결정을 걸러낼 수 없습니다.
{: .prompt-warning }

이 장의 구조:

1. 탐지 응용 지형 — 무엇이 연구되었나
2. **Det-RAN** — dApp 기반 실시간 크로스레이어 탐지
3. **SpotLight** — 엣지·클라우드 분산 이상탐지와 설명가능성
4. **XAI[^a-xai] + LLM[^a-llm]** — 설명을 사람의 언어로
5. **경량 IDS(Intrusion Detection System)[^a-ids] 벤치마크** — 지연 실측이 말해주는 진실
6. 조기 탐지 — DU에서 막기
7. 분산·연합 학습 기반 협력 탐지
8. Self-Healing — 탐지에서 자율 완화까지

---

## 1. 탐지 응용 지형

Benzaïd 등[^r5]이 정리한 O-RAN(Open Radio Access Network)[^a-o-ran] AI 보안 응용을 두 축으로 나눕니다.

| 축 | 세부 응용 | 대표 기법 |
|---|---|---|
| **A. 보안 조치 강화** (Enhancing Security Measures) | DoS[^a-dos]/DDoS[^a-ddos] 탐지 | 얕은/심층 학습, **FL**[^a-fl], FCL[^a-fcl], DRL[^a-drl] 기반 xApp 스케일링 |
| | 시그널링 스톰 탐지 | **DBSCAN**[^a-dbscan](비지도 클러스터링), RL[^a-rl]·분산 AI |
| | 재밍 탐지 | Hoeffding 결정트리, **Bayesian 추론 지원 ML**[^a-ml], ESN[^a-esn], RL, FL+DRL |
| | 이상 탐지 | Isolation Forest, Random Forest, MLP[^a-mlp]-AE[^a-ae], CNN[^a-cnn]-VAE[^a-vae], LSTM-GAN[^a-gan] |
| **B. 전통 평가 도구 고도화** (Elevating Assessment Tools) | 취약점 테스팅 | **BERT**[^a-bert] 기반 CWE[^a-cwe]/CAPEC[^a-capec] 자동 매핑, **LSTM 기반 RRC[^a-rrc] 퍼징** |
| | API[^a-api] 보안 | 유효 API 테스트케이스 예측, GAN 기반 few-shot 이상탐지, LSTM+AE+DBSCAN |
| | 상충 완화 | Team learning, Knowledge transfer (Ch5) |
| | Zero Trust | i-ZTA, DRL 정책엔진, Transformer (Ch7) |

### 1.1 DoS/DDoS 탐지 — 왜 O-RAN에서 특별한가

> O-RAN Alliance가 정리한 O-RAN 위협 표면을 보면 **식별된 보안 위험 대부분이 (D)DoS 공격과 관련**됩니다. 이 공격들은 제어 평면 또는 사용자 평면을 표적으로 하며, **시그널링 스톰, 자원 고갈, 볼류메트릭 네트워크 계층 공격, 은밀한 애플리케이션 계층 공격** 등 다양한 형태로 나타납니다.[^r5]
{: .prompt-info }

| 연구 | 배치 위치 | 기법 | 결과 |
|---|---|---|---|
| O-DU 수준 DoS 조기 탐지[^r26] | **O-DU** | RF[^a-rf] / SVM[^a-svm] / k-NN[^a-knn] / DT[^a-dt] / AdaBoost / MLP 비교 (PHY[^a-phy]·MAC[^a-mac] 특징: SINR[^a-sinr], CQI[^a-cqi], bitrate, 패킷 수) | **Random Forest가 정확도-추론시간 균형 최적** |
| Open-FH[^a-o-fh] DDoS 탐지[^r5] | xApp | **CNN + LSTM** (시공간 의존성 포착), **축소된 특징 공간**으로 S-Plane 시간 제약 충족 | — |
| 크로스도메인 DDoS[^r5] | 트래픽 분류 xApp | **전송망의 ML 트래픽 분류기 피드백**과 RAN[^a-ran] 데이터 결합해 xApp 모델을 지속 정련 | — |
| **FCL 협력 탐지**[^r5] | Near-RT RIC(FL 에이전트) + Non-RT RIC(집계 서버) | **FL + Continual Learning** — 프라이버시 보존 지식 공유 + **파괴적 망각(catastrophic forgetting) 완화** | 이전 지식의 **98.8% 이상 유지**, 탐지 정확도 상당 향상 |
| DRL 기반 IDS xApp 스케일링[^r5] | Near-RT RIC | **A2C**(Advantage Actor-Critic)[^a-a2c] 에이전트가 IDS xApp 수를 네트워크 조건에 동적 적응 | **URLLC[^a-urllc]의 엄격한 지연·가용성 요구를 지키면서 분석 트래픽량 최대화** |

> **FCL(Federated Continual Learning)이 중요한 이유**: 순차적 태스크 학습에서 새 공격을 학습하면 옛 공격을 잊는 **파괴적 망각**이 발생합니다. 공격 유형이 계속 추가되는 보안 도메인에서 이는 치명적입니다. FCL은 FL(프라이버시)과 CL(망각 방지)을 결합해 **98.8% 이상의 지식 보존**을 달성했습니다[^r5].
{: .prompt-tip }

### 1.2 재밍 탐지

| 연구 | 기법 | 특징 |
|---|---|---|
| NG-RAN[^a-ng-ran] 실시간 재밍 탐지[^r5] | **Hoeffding 결정 트리** | 스트리밍 데이터에 적합 |
| O-RAN 즉시/near-RT 재밍 탐지[^r5] | 즉시: k-NN, AdaBoost, **RF(최고 정확도)** / near-RT: **ESN**(Echo State Network) | **크로스레이어 KPI**[^a-kpi](PHY, MAC, RLC[^a-rlc], PDCP[^a-pdcp]). **BNM 기반 인과추론**으로 학습데이터 부족 문제 해결 |
| 다중 UAV[^a-uav] 스웜 재밍 소스 예측[^r5] | **RL** | 실시간 적응성 |
| 협력적 재밍 완화[^r5] | **FL + DRL** — DRL 에이전트를 Near-RT RIC에 배치, E2로 E2 노드와 상호작용하며 국소 학습, Non-RT RIC에서 FL 집계 | 피드백 루프로 재밍 진화에 지속 대응 |

### 1.3 이상 탐지

| 연구 | 모델 | 대상 |
|---|---|---|
| O-RAN SC[^a-sc] anomaly detection xApp[^r5] | **Isolation Forest** | 이상 UE[^a-ue] (RSRP[^a-rsrp], RSRQ[^a-rsrq], NR[^a-nr]-RSSI[^a-rssi], throughput, PRB[^a-prb] 사용률) |
| 벤치마크 연구[^r5] | **Random Forest** | **O-RAN SC의 Isolation Forest보다 정확도·학습시간 모두 우수** |
| MLP-AutoEncoder[^r5] | AE | 이상 UE |
| CNN-VAE / LSTM-GAN[^r5] | 생성 모델 | 셀 이상 |
| GAN·AE 비지도 모델[^r5] | 생성 모델 | **변조 분류(modulation classification)** 이상 → 비인가 신호·송신기 스푸핑·재밍 식별 |
| ML/DL 기반[^r5] | — | **FBS**(False Base Station)[^a-fbs] 존재 인식 |
| P2P[^a-p2p] MLP-FL / + Transfer Learning[^r5] | FL | 분산 이상 탐지 (전이학습으로 통신비용 절감) |

---

## 2. Det-RAN — dApp 기반 실시간 크로스레이어 탐지

Scalingi 등[^r11]은 **RRC 계층에 무결성 보호가 없다**는 근본 문제에 정면으로 대응합니다.

### 2.1 문제 정의

> 5G NR에서 **Control PDU의 계층 2 메시지 무결성 보호 부재**는 대체로 미해결 문제이며, 현재 가용한 해법들은 문제를 부분적으로만 해결하는 낡은 절차에 의존한다. 이는 5G 네트워크를 RRC 계층 등 5G 시스템의 가장 중요한 구성요소에 대한 여러 보안 위협에 노출시킨다. RRC는 무선 자원 관리 수행에 필수적이며, **그 무결성에 대한 어떤 공격도 통신과 신뢰성을 심각히 교란**할 수 있다.[^r11]
{: .prompt-danger }

문제를 더 악화시키는 것은 **RRC Inactive State**입니다 — 배터리 효율과 저지연을 절충하려 도입되었지만 오버헤드 감소와 함께 공격 표면을 만듭니다[^r11].

![계층 2 메시지 무결성 보호 부재로 5G가 노출되는 보안 위협 (출처: Scalingi 등[^r11], Fig. 2)](/assets/img/posts/6g-ai-ran/detran-fig2.png)
_그림 8-1. 계층 2 무결성 보호 부재가 만드는 위협. 출처: [^r11], Fig. 2._

### 2.2 접근 — 물리계층 지문

상위 계층을 신뢰할 수 없으므로 **물리계층에서 신뢰의 근거를 만듭니다.**

| 요소 | 내용 |
|---|---|
| **PHY 지문** | 신뢰 가능한 지문을 생성할 수 있는 물리계층 특징 식별 |
| **DMRS 파일럿** | **RRC 패킷의 subframe 2의 DMRS**(Demodulation Reference Signal)[^a-dmrs] 파일럿 시퀀스 활용 |
| **도착 시각 추론** | 무결성 보호가 없는 **상향 패킷의 도착 시각을 새로운 방식으로 추론** |
| **크로스레이어 특징** | PHY + 상위 계층 특징을 함께 처리 |

![업링크 데이터의 물리계층과 자원 그리드 (출처: Scalingi 등[^r11], Fig. 1)](/assets/img/posts/6g-ai-ran/detran-fig1.png)
_그림 8-2. 업링크 데이터의 물리계층 및 자원 그리드 구조. 출처: [^r11], Fig. 1._

![RRC 패킷 subframe 2의 DMRS 파일럿 시퀀스 활용 (출처: Scalingi 등[^r11], Fig. 3)](/assets/img/posts/6g-ai-ran/detran-fig3.png)
_그림 8-3. **DMRS 파일럿 시퀀스**를 빠른 지문 생성에 활용. 출처: [^r11], Fig. 3._

### 2.3 dApp 설계

![dApp 설계의 상위 수준 구성 — 3개 주요 모듈 (출처: Scalingi 등[^r11], Fig. 5)](/assets/img/posts/6g-ai-ran/detran-fig5.png)
_그림 8-4. **Det-RAN의 dApp 설계** — 3개 주요 모듈과 상호작용. dApp은 O-DU/O-CU 내부에서 동작하므로 Near-RT RIC보다 훨씬 낮은 지연으로 물리계층 데이터에 접근할 수 있습니다. 출처: [^r11], Fig. 5._

![제안 프레임워크의 AI 모듈 (출처: Scalingi 등[^r11], Fig. 4)](/assets/img/posts/6g-ai-ran/detran-fig4.png)
_그림 8-5. Det-RAN의 AI 모듈. 출처: [^r11], Fig. 4._

### 2.4 결과

| 지표 | 값 |
|---|---|
| 미관측 테스트 시나리오 공격 예측 정확도 | **85% 이상**[^r11] |
| 실시간 제약 충족 | **2 ms**[^r11] |
| 검증 방식 | 광범위 에뮬레이션 → **대형 채널 에뮬레이터를 갖춘 실제 프로토타입**으로 실시간 성능·비용 평가 |

![단일 시나리오로 학습해 여러 시나리오에서 테스트한 NN 아키텍처별 정확도 (출처: Scalingi 등[^r11], Fig. 7)](/assets/img/posts/6g-ai-ran/detran-fig7.png)
_그림 8-6. NN[^a-nn] 아키텍처별 테스트 정확도 — 단일 시나리오 학습 후 일반화 성능. 출처: [^r11], Fig. 7._

![다중 시나리오 학습 후 단일 시나리오 테스트 결과 (출처: Scalingi 등[^r11], Fig. 8)](/assets/img/posts/6g-ai-ran/detran-fig8.png)
_그림 8-7. 다중 시나리오 학습 시 일반화 향상. 출처: [^r11], Fig. 8._

![다양한 설정에서의 추론 시간 — Setup 4에서 Det-RAN이 실시간 요구를 충족 (출처: Scalingi 등[^r11], Fig. 11)](/assets/img/posts/6g-ai-ran/detran-fig11.png)
_그림 8-8. **추론 시간** — Setup 4에서 Det-RAN이 실시간 요구를 충족합니다. 출처: [^r11], Fig. 11._

![gNB에서 수신한 RRC 메시지가 동일 UE에서 온 것인지 판별하는 체비셰프 확률 (출처: Scalingi 등[^r11], Fig. 9)](/assets/img/posts/6g-ai-ran/detran-fig9.png)
_그림 8-9. 체비셰프 부등식 기반 확률로 RRC 메시지의 동일 UE 여부를 판별. 출처: [^r11], Fig. 9._

> **Det-RAN의 교훈**: 상위 계층에 무결성 보호가 없다면 **더 아래로 내려가서 신뢰의 근거를 찾습니다.** 그리고 그 처리는 near-RT RIC가 아니라 **dApp**(프로토콜 스택 내부)에 두어야 2 ms 예산에 맞습니다. Ch2의 제어 루프 시간 척도 논의가 여기서 설계 결정으로 이어집니다.
{: .prompt-tip }

---

## 3. SpotLight — 엣지·클라우드 분산 이상탐지와 설명가능성

Sun 등[^r12]은 다른 각도의 문제를 풉니다: **멀티벤더 분해로 인한 운영 복잡성** 때문에 성능 문제·장애의 원인을 찾기 어렵다는 것입니다.

![Open RAN은 다양한 개방 인터페이스를 정의하고 여러 공급자의 HW·SW 구성요소로 이루어진다 (출처: Sun 등[^r12], Fig. 1)](/assets/img/posts/6g-ai-ran/spotlight-fig1.png)
_그림 8-10. Open RAN의 멀티벤더 구성. 출처: [^r12], Fig. 1._

### 3.1 문제의 실체 — 계층을 타고 번지는 이상

![프론트홀 링크 처리량과 관련 스레드 런타임, DL TCP 처리량과 SINR/SNR (출처: Sun 등[^r12], Fig. 2)](/assets/img/posts/6g-ai-ran/spotlight-fig2.png)
_그림 8-11. **FH 링크 처리량 + 관련 collocated 스레드 런타임**(상단), **DL TCP[^a-tcp] 처리량 + SINR/SNR[^a-snr]**(하단). 하나의 근본 원인이 여러 계층 지표에 동시에 나타나는 것이 근본 원인 식별을 어렵게 만듭니다. 출처: [^r12], Fig. 2._

> Sun 등[^r12]이 설명가능성을 요구사항으로 못 박은 이유입니다: *"vRAN의 물리(L1) 계층 이상은 L1의 예상치 못한 상향 트래픽 변동으로 나타날 수 있다. **그러나 이것은 MAC, RLC, PDCP 계층의 상향 트래픽 변동도 유발**하며, 우리는 이 메트릭들도 이상으로 표시할 가능성이 있다. 따라서 방법은 **KPI 간 의존성을 학습·추적하여 최소한의 관련 KPI 집합으로 올바른 근본 원인을 찾도록** 필터링해야 한다."*
{: .prompt-warning }

### 3.2 시스템 아키텍처 — 엣지와 클라우드 분업

![SpotLight 시스템 아키텍처 개요 (출처: Sun 등[^r12], Fig. 3)](/assets/img/posts/6g-ai-ran/spotlight-fig3.png)
_그림 8-12. **SpotLight 시스템 아키텍처**. OOD[^a-ood] = Out of Distribution. 출처: [^r12], Fig. 3._

| 부분 | 위치 | 역할 |
|---|---|---|
| **Data collection** | RAN + 플랫폼 | RAN과 플랫폼 양쪽에 상세 계측 도입. **총 600개 이상의 세밀한 메트릭/KPI** 수집 |
| **Data processing at the edge** | far-edge 노드 | 컴퓨트가 제한적이므로 정교한 ML 부적합 → **경량이지만 강건한 정상 케이스 필터링**만 수행 (대다수가 정상). 업링크 대역폭 최소화 |
| **Anomaly detection & identification** | **클라우드** | 무거운 처리를 담당 (컴퓨트·스토리지 풍부) |

> **설계 동기가 명확합니다**: *"중앙 위치(클라우드)가 이 작업에 더 적합하다. 그러나 **수만 개(10,000s) far-edge 사이트에서 대량 데이터를 보내는 것은 금지적으로 비쌀 수 있다.**"*[^r12] → 엣지 필터링이 비용 문제의 해답입니다.
{: .prompt-tip }

### 3.3 데이터 파이프라인과 방법

| 파라미터 | 값 |
|---|---|
| 샘플링 | KPI별 **100 ms**마다 |
| 윈도우 | **연속 64 샘플**을 하나의 윈도우 $W$로 (다변량 시계열) |
| 출력 | 이상 시 **관련 이상 KPI의 필터링된 집합** → 특정 RAN·플랫폼 구성요소를 지시 |

**설계 요구사항 3가지**[^r12]:

| 요구 | 내용 |
|---|---|
| **Accuracy** | 높은 recall + 높은 precision. **준지도(semi-supervised)** — **정상 데이터만으로, 제한된 양으로** 학습해야 함(모든 종류의 이상을 탐지하려면). 미관측 KPI 패턴에도 일반화 |
| **Explainability** | 이상의 원인을 가리키는 **최소 KPI 부분집합**으로 안정적으로 유도 |
| **Efficiency** | far-edge의 제한된 국소 처리자원 효율 사용 + 클라우드 대역폭·비용 최소화 + 원하는 시간척도 내 완료 |

**4단계 파이프라인**: 앞 두 단계는 **윈도우 수준** 이상 탐지·필터링, 뒤 두 단계는 **KPI 수준** 필터링으로 근본 원인 설명[^r12]. 핵심 조합은 **분포 학습(distribution learning) + 시계열 결측 보간(time-series imputation)** 이며, 저자들은 이 조합이 시계열 이상탐지에 사용된 전례가 없다고 밝힙니다.

![JVGAN 아키텍처, MRPI 워크플로우, 학습된 분포의 도해 (출처: Sun 등[^r12], Fig. 4)](/assets/img/posts/6g-ai-ran/spotlight-fig4.png)
_그림 8-13. **(a) JVGAN 아키텍처**, **(b) MRPI 워크플로우**, **(c) 학습된 분포의 도해**. **JVGAN이 엣지에서 실행**되어 학습 시 습득한 정상 데이터 분포로 정상 케이스를 걸러냅니다(= '잠재적' 이상 탐지). 출처: [^r12], Fig. 4._

### 3.4 결과 — 세 축 모두에서 개선

평가 환경: **실내 오피스 빌딩의 엔터프라이즈 규모 5G Open RAN 배치**에서 수집한 실측 메트릭[^r12].

| 축 | 개선 |
|---|---|
| **정확도** | 기준선 대비 **F1 스코어 13% 향상** |
| **설명가능성** | 보고되는 **KPI 개수 2.3 ~ 4배 감소** |
| **효율** | **대역폭 4 ~ 7배 절감** |

![모든 시나리오에 대한 평균 F1 스코어 (출처: Sun 등[^r12], Fig. 6)](/assets/img/posts/6g-ai-ran/spotlight-fig6.png)
_그림 8-14. 전체 시나리오 평균 F1 스코어 비교 (SpotLight 대 TranAD·MADGAN·VAE-LSTM·LSTM-AE·LSTM-PRED 등 기준선). 출처: [^r12], Fig. 6._

![이상 원인 후보로 플래그된 KPI 비율 (출처: Sun 등[^r12], Fig. 8)](/assets/img/posts/6g-ai-ran/spotlight-fig8.png)
_그림 8-15. 전체 고려 KPI 중 이상 원인 후보로 플래그된 비율 — 낮을수록 설명가능성이 좋습니다. 출처: [^r12], Fig. 8._

![데이터 수집 시간 대비 처리 시간 비율 (출처: Sun 등[^r12], Fig. 9)](/assets/img/posts/6g-ai-ran/spotlight-fig9.png)
_그림 8-16. 데이터 수집 시간 대비 처리 시간 비율 — 실시간성 지표. 출처: [^r12], Fig. 9._

---

## 4. XAI + LLM — 설명을 사람의 언어로

Chatzimiltis 등[^r13]은 **Near-RT RIC 안에서** DDoS를 탐지하고, 그 판단 근거를 **비전문가가 읽을 수 있는 자연어로** 변환합니다.

### 4.1 3층 구조

처리 순서는 **E2 노드 KPM[^a-kpm](다변량 시계열) → LSTM 모델(악성 UE 행동 식별) → post-hoc XAI(LIME[^a-lime]·SHAP[^a-shap]로 개별 예측 해석) → LLM(기술적 설명을 자연어 인사이트로 변환) → 비전문가 운영자** 입니다[^r13].

![Open RAN 아키텍처 내 제안 프레임워크의 종단간 워크플로우 (출처: Chatzimiltis 등[^r13], Fig. 1)](/assets/img/posts/6g-ai-ran/xaiddos-fig1.png)
_그림 8-17. **Open RAN 내 종단간 워크플로우**. 출처: [^r13], Fig. 1._

### 4.2 결과

| 지표 | 값 |
|---|---|
| 탐지 성능 | **F1-score > 0.96** (실제 5G 네트워크 KPM 기준)[^r13] |
| 다룬 공격 유형 | **SYN Flood, ICMP[^a-icmp] Flood, UDP[^a-udp] Fragmentation, DNS[^a-dns] Flood, GTP-U[^a-gtp-u] Flood** |
| 산출물 | 실행 가능하고 **해석 가능한** 출력 |

![공격 상태와 정상 상태의 다운링크 비트레이트 KDE 비교 (출처: Chatzimiltis 등[^r13], Fig. 4)](/assets/img/posts/6g-ai-ran/xaiddos-fig4.png)
_그림 8-18. 공격·정상 조건의 다운링크 비트레이트 KDE[^a-kde] 비교. 출처: [^r13], Fig. 4._

![정상 UE(파랑)와 악성 UE(빨강)의 특징 변동 비교 (출처: Chatzimiltis 등[^r13], Fig. 5)](/assets/img/posts/6g-ai-ran/xaiddos-fig5.png)
_그림 8-19. 정상 UE와 악성 UE의 특징 변동 비교. 출처: [^r13], Fig. 5._

### 4.3 설명가능성 — 보안 도구로서의 LIME/SHAP

![TP·TN·FP·FN 인스턴스에 대한 국소 설명 (출처: Chatzimiltis 등[^r13], Fig. 8)](/assets/img/posts/6g-ai-ran/xaiddos-fig8.png)
_그림 8-20. **TP·TN·FP·FN 각 사례의 국소 설명**. 상단 행은 **LIME 상위 10개 특징 기여도**를 보여줍니다. FP/FN 사례의 근거를 보면 왜 틀렸는지 진단할 수 있습니다. 출처: [^r13], Fig. 8._

![전역 SHAP beeswarm 플롯 — 상위 10개 영향 특징 (출처: Chatzimiltis 등[^r13], Fig. 9)](/assets/img/posts/6g-ai-ran/xaiddos-fig9.png)
_그림 8-21. **전역 SHAP beeswarm** — 테스트 인스턴스 전반에서 가장 영향력 있는 상위 10개 특징. 출처: [^r13], Fig. 9._

![평균 절대 SHAP 값 히트맵 (출처: Chatzimiltis 등[^r13], Fig. 10)](/assets/img/posts/6g-ai-ran/xaiddos-fig10.png)
_그림 8-22. 평균 절대 SHAP 값 히트맵 — 각 특징의 전체 중요도. 출처: [^r13], Fig. 10._

![No Ratio 대 Ratio 설정의 F1-Score 비교 (출처: Chatzimiltis 등[^r13], Fig. 6)](/assets/img/posts/6g-ai-ran/xaiddos-fig6.png)
_그림 8-23. 특징 비율(ratio) 사용 여부에 따른 F1 비교. 출처: [^r13], Fig. 6._

![평균 FPR(좌)과 FNR(우) (출처: Chatzimiltis 등[^r13], Fig. 7)](/assets/img/posts/6g-ai-ran/xaiddos-fig7.png)
_그림 8-24. 평균 FPR[^a-fpr]·FNR[^a-fnr]. 보안 운영에서는 **FPR이 곧 운영 부담**이므로 F1만 보아서는 안 됩니다. 출처: [^r13], Fig. 7._

> **Ch7의 XAI를 실제로 쓰는 방법이 여기 있습니다.** 적대적 입력은 예측값이 정상처럼 보이지만 **기여 특징이 이상**합니다. LIME/SHAP 상위 특징이 도메인 상식과 어긋나면(예: 재밍 탐지가 무관한 KPI에 의존) 조작을 의심할 수 있습니다.
{: .prompt-tip }

### 4.4 XAI + LLM을 zero-touch 관리로 확장

Mekrache 등[^r23]은 같은 조합을 **ZSM**(Zero-touch Service Management)[^a-zsm] 전반으로 확장합니다.

![LLM 기반 신뢰 가능 ZSM 아키텍처 설계와 유즈케이스 (출처: Mekrache 등[^r23], Fig. 1)](/assets/img/posts/6g-ai-ran/xaizt-fig1.png)
_그림 8-25. **LLM 기반 신뢰 가능 ZSM 아키텍처**와 유즈케이스. 출처: [^r23], Fig. 1._

![마이크로서비스 부하에 따른 동적 CPU·RAM 스케일링 (출처: Mekrache 등[^r23], Fig. 3)](/assets/img/posts/6g-ai-ran/xaizt-fig3.png)
_그림 8-26. 마이크로서비스 부하에 대응한 동적 CPU·RAM 스케일링. 출처: [^r23], Fig. 3._

![CPU·RAM 할당 비교 (출처: Mekrache 등[^r23], Fig. 4)](/assets/img/posts/6g-ai-ran/xaizt-fig4.png)
_그림 8-27. CPU·RAM 할당 비교. 출처: [^r23], Fig. 4._

![동적 스케일링 유즈케이스에서 LLM별 점수와 출력 생성 시간 (출처: Mekrache 등[^r23], Fig. 5)](/assets/img/posts/6g-ai-ran/xaizt-fig5.png)
_그림 8-28. **LLM별 점수와 생성 시간**. 생성 시간이 초 단위이므로 **Non-RT 루프에만 적합**하다는 점을 확인할 수 있습니다 — Near-RT 제어에 LLM을 직접 넣을 수 없는 이유입니다. Mekrache 등[^r23]은 **Green LLMs**(에너지 효율) 문제도 함께 지적합니다. 출처: [^r23], Fig. 5._

---

## 5. 경량 IDS 벤치마크 — 지연 실측이 말해주는 진실

Ben Khalifa 등[^r25]의 기여는 정확도 경쟁이 아니라 **지연 정량화**입니다.

### 5.1 문제 제기

> 최근 문헌은 CNN이나 LSTM 같은 모델을 강조하지만, 이들은 **Near-RT RIC(10 ms ~ 1 s 루프)와 O-DU의 엄격한 지연 예산과 충돌하는 계산 오버헤드**를 수반한다. **보안 메커니즘의 추론 지연을 명시적으로 정량화한 연구는 거의 없다.**[^r25]
{: .prompt-warning }

### 5.2 실험 구성

| 항목 | 내용 |
|---|---|
| 데이터셋 | **NetSLab-5GORAN-IDD** |
| 평가 모델 | **12개 알고리즘** — 경량 ML(Linear SVM, Decision Tree, Naive Bayes) + 앙상블(XGBoost, Random Forest, Extra Trees) + 딥러닝(LSTM 등) |
| 배치 위치 | Near-RT RIC의 **IDS xApp**, 그리고 O-DU 필터링 |

![중요 인터페이스를 강조한 O-RAN 아키텍처 — Near-RT RIC가 IDS xApp을 호스팅 (출처: Ben Khalifa 등[^r25], Fig. 1)](/assets/img/posts/6g-ai-ran/lightids-fig1.png)
_그림 8-29. **Near-RT RIC가 IDS xApp을 호스팅**하여 실시간 탐지를 수행하는 구조. 출처: [^r25], Fig. 1._

### 5.3 결과 — 세 자리 수 차이

![추론 지연 비교 (로그 스케일) — ML과 DL 모델 간 격차 (출처: Ben Khalifa 등[^r25], Fig. 2)](/assets/img/posts/6g-ai-ran/lightids-fig2.png)
_그림 8-30. **추론 지연 비교 (로그 스케일)**. 논문의 캡션은 *"ML과 DL 모델 사이의 막대한 격차에 주목하라"* 고 명시합니다. 출처: [^r25], Fig. 2._

관측된 지연의 대략적 스케일(밀리초, 논문 표 기준)[^r25]:

| 계열 | 모델 | 추론 지연 스케일 |
|---|---|---|
| 경량 ML | Linear SVM | ~0.03 – 0.07 ms |
| | **Decision Tree** | ~0.04 – 0.15 ms |
| | Naive Bayes | ~0.06 – 0.30 ms |
| 앙상블 | XGBoost | ~0.49 – 1.8 ms |
| | Random Forest | ~0.56 – 0.88 ms |
| | Extra Trees | ~2.6 – 3.4 ms |
| **딥러닝** | **LSTM** | **~2,268 – 3,379 ms** |

### 5.4 결론과 설계 지침

| 결론[^r25] | 설계 지침 |
|---|---|
| **앙상블(Extra Trees, XGBoost)이 우수한 정확도** 달성 | Near-RT RIC의 IDS xApp에는 앙상블이 적합 (수 ms 수준) |
| **단일 Decision Tree가 초저지연 O-DU 필터링의 유효 후보** — DL보다 **수 자리(orders of magnitude) 빠름** | O-DU에는 DT 기반 1차 필터, 통과분만 상위로 |
| 확률 모델(Naive Bayes 등)의 특성 | 저지연이지만 정확도 트레이드오프 |

> **2단 방어 설계가 자연스럽게 도출됩니다**: **O-DU에 초저지연 DT 필터** → **Near-RT RIC에 앙상블 IDS xApp** → **Non-RT RIC/클라우드에 무거운 DL(Deep Learning)[^a-dl]·LLM 분석**. 이는 §3의 SpotLight(엣지 경량 필터 + 클라우드 정밀 분석)와 정확히 같은 원리이며, Ch2의 제어 루프 계층과 일치합니다.
{: .prompt-tip }

---

## 6. 조기 탐지 — DU에서 막기

Xavier 등[^r26]의 핵심 주장은 **"악성 트래픽이 CU(Central Unit)[^a-cu]에 들어오기 전에 DU에서 탐지한다"** 입니다.

![데이터 수집과 ML 모델 학습·추론 프레임워크 (출처: Xavier 등[^r26], Fig. 1)](/assets/img/posts/6g-ai-ran/mlearly-fig1.png)
_그림 8-31. 데이터 수집 및 ML 학습·추론 프레임워크. 출처: [^r26], Fig. 1._

![실험 구성 아키텍처 — 모든 구성요소 간 인터페이스와 통신 (출처: Xavier 등[^r26], Fig. 2)](/assets/img/posts/6g-ai-ran/mlearly-fig2.png)
_그림 8-32. 실험 구성 아키텍처. 자체 개발한 near-RT RIC 인터페이스를 포함합니다. 출처: [^r26], Fig. 2._

| 항목 | 내용 |
|---|---|
| 접근 | OpenRAN 프레임워크로 **공중 인터페이스 측정치 수집**(탐지용) + RAN 동작 **동적 제어** |
| 구현 | 자체 **near-RT RIC 인터페이스** 개발 |
| 모델 비교 | 광범위한 ML 알고리즘을 **near-RT RIC가 설정한 정확도·지연 요구를 만족하는지** 기준으로 분석 |
| 결과 | 현실적 테스트베드에서 정상 대 악성 트래픽을 **약 95% 정확도**로 분류. **악성 트래픽이 CU에 진입하기 전에 DU에서 공격 탐지** |

![개별 클래스(좌)와 이진 문제(우)의 혼동 행렬 (출처: Xavier 등[^r26], Fig. 3)](/assets/img/posts/6g-ai-ran/mlearly-fig3.png)
_그림 8-33. 개별 클래스 및 이진 분류 혼동 행렬 (VoIP[^a-voip], DDoS Ripper, DoS Hulk, Slowloris, Benign 등). 출처: [^r26], Fig. 3._

![네트워크 트래픽이 초기 구간에서 전이 상태에 있음을 보여주는 CDF (출처: Xavier 등[^r26], Fig. 4)](/assets/img/posts/6g-ai-ran/mlearly-fig4.png)
_그림 8-34. **CDF**[^a-cdf] — 네트워크 트래픽이 초기 몇 초 동안 **전이 상태(transition state)** 에 있음을 보여줍니다. 즉 공격 개시 직후에는 판단을 유보해야 할 구간이 존재합니다. 출처: [^r26], Fig. 4._

---

## 7. Self-Healing — 탐지에서 자율 완화까지

탐지는 절반입니다. 나머지 절반은 **자율 완화(closed-loop automation)** 입니다.

### 7.1 완화 액션 카탈로그

앞선 연구들이 실제로 취한 완화 조치를 정리하면:

| 위협 | 완화 액션 | 실행 지점 | 출처 |
|---|---|---|---|
| 재밍/간섭 | **적응형 MCS[^a-mcs]**, 반송파 주파수 변경, 자원블록 기반 지능형 스케줄링 | E2 제어 → RAN | [^r8] |
| 시그널링 스톰 | **과도한 악성 Attach Request 차단** (RIC가 RRC 연결을 관리하므로 최적 지점) | Near-RT RIC | [^r6] |
| DDoS 부하 변동 | **IDS xApp 인스턴스 수 동적 조정**(A2C DRL) | Near-RT RIC | [^r5] |
| 재밍 진화 | DRL 에이전트 지속 정련 + FL로 에이전트 간 지식 공유 | Near-RT RIC + Non-RT RIC | [^r5] |
| 토폴로지 위조 | **LLDP[^a-lldp] 패킷 폐기** 또는 토폴로지 갱신 (RLV[^a-rlv] 분류 결과 기반) | Near-RT RIC | [^r6] |
| 적대적 입력 | 증류 기반 방어 모델로 전환 | xApp | [^r8] |
| 자원 고갈 | 네임스페이스 쿼터, **RAN 워크로드 선점(preemption)** | AI-RAN Site / AI-SMO[^a-smo] | [^r2] |

### 7.2 폐루프 자동화의 조건

> Soltani 등[^r6]: *"RIC가 DDoS 공격을 자체적으로 식별한 후에는 어떤 형태의 완화 조치가 필요하다. RIC는 RRC 연결을 관리하므로 이 유형의 공격을 다루는 가장 효율적인 Open RAN 구성요소이며, 과도한 악성 Attach Request 방지에 이상적이다. 이런 방식의 탐지·완화는 **내장된 폐루프 자동화**를 보여줄 것이다."*
{: .prompt-info }

폐루프가 안전하게 동작하기 위한 조건:

| 조건 | 왜 | 관련 장 |
|---|---|---|
| **시간 예산 준수** | 예산 초과 시 완화가 무의미 (Ch6의 자원 고갈 공격이 이를 노림) | Ch2, §5 |
| **설명가능성** | 자율 조치의 근거를 사후 감사할 수 있어야 함 | Ch7 XAI, §4 |
| **상충 관리** | 여러 자율 대응이 서로를 상쇄하지 않아야 함 | Ch5 CME |
| **정형 검증 / 가드레일** | 조치가 안전 경계를 벗어나지 않아야 함 | **Ch10** |
| **회복 거동 이해** | 학습형 제어기의 회복 기간을 고려 | Ch6 §4.2 |

> **Self-healing의 역설**: 자율 완화 능력이 강해질수록 **그 능력이 공격자의 무기**가 됩니다. 오탐(false positive)을 유도해 시스템이 스스로 정상 서비스를 차단하게 만드는 것 — Ch10에서 다루는 **가드레일 DoS**[^r19]와 동일한 구조입니다. §4.3의 FPR 지표를 반드시 함께 보아야 하는 이유입니다.
{: .prompt-danger }

---

## 8. 분산·블록체인 기반 AI 보안과 에이전틱 방어 계층

지금까지 본 탐지·자율완화가 **"단일 모델이 무엇을 어떻게 탐지하는가"** 였다면, 이번 절은 두 가지 보완 관점을 더합니다 — **(1)** Braeken 등[^r36]의 **분산 AI 학습(DAIL)·블록체인 기반 AI 보안(BASec)** 프레임워크로 탐지·완화 자체의 신뢰 기반을 분산·검증 가능하게 만들고, **(2)** Feng 등[^r37]의 **에이전틱 방어(Agent-as-Defender)** taxonomy로 §7의 Self-Healing을 계층화된 자율 에이전트 구조로 구체화합니다.

### 8.1 분산 AI 학습(DAIL)과 블록체인 기반 AI 보안(BASec)

| 프레임워크 | 핵심 개념 | 위협·방어 매핑 | 트레이드오프 |
|---|---|---|---|
| **DAIL**(Decentralized AI Learning) | FL(연합학습)의 확장 개념. 학습·추론을 엣지·디바이스 계층으로 분산시켜 복원력·적응성 확보. **보안 집계(secure aggregation)·DP·TEE 기반 원격증명**으로 중앙집중 의존 없이 모델 무결성 유지 | §1.1의 FCL 협력 탐지가 이미 이 원리를 실증 — Ch6의 오염·백도어·드리프트 공격에 직접 대응, 비잔틴 강건 집계·동적 클라이언트 선택으로 이질적 데이터 하의 공격 표면 축소 | 수렴성-복원력 트레이드오프, 통신 비용, 갱신 검증·인센티브 정합, 슬라이스 인지 학습 주기 설계 필요 |
| **BASec**(Blockchain-Based AI Security) | 분산원장으로 FL/AI 관리 전반에 **변조 불가능한 출처 증명·검증 가능 집계·인센티브 메커니즘** 제공. 스마트 컨트랙트가 모델 제출·교차사업자 책임성·보상 분배를 규율 | 출처 변조·리플레이·데이터 유출 위험을 연합 갱신·접근 이벤트의 불변 감사기록으로 완화. **무임승차(free-riding)·표절**은 검증 가능한 기록 앵커링과 proof-of-effort 검증으로 대응 | 처리량·저장 비용, 선택적 온체인 앵커링 + 오프체인 저장, 경량 합의 권장 |

_표 8-0. 분산·블록체인 기반 AI 보안 프레임워크. 출처: Braeken 등[^r36]을 재구성._

> **§1.1 FCL과의 관계**: FCL(Federated Continual Learning)이 이미 "프라이버시 보존 + 파괴적 망각 완화"를 달성했다면, DAIL·BASec은 그 위에 **비잔틴 강건성**(오염된 참여자 배제)과 **감사 가능한 출처 증명**(누가 무엇을 언제 기여했는지)을 추가하는 다음 단계입니다.
{: .prompt-tip }

### 8.2 에이전틱 방어 계층 — Sentinel · Tactical · Strategic

Feng 등[^r37]은 **PRPA(Perception-Reasoning-Planning-Action)**[^a-prpa] 루프를 방어 측에 적용해, 인지 깊이·운영 범위에 따라 방어 에이전트를 3단 계층으로 나눕니다.

| 계층 | 위치 | 초점 | 역할 |
|---|---|---|---|
| **Sentinel Agents** | 극단 엣지 (UE, O-RU) | 지각(Perception) | 경량 뉴로모픽 센싱으로 PHY 계층 이상(재밍 패턴)을 실시간 탐지, 압축된 위협 인텔리전스를 상위로 전달 |
| **Tactical Agents** | Near-RT RIC / MEC 노드 | 행동(Action) | 사전 승인된 정책에 기반해 **밀리초 단위 대응**(빔 널링, 마이크로 세그먼테이션) 실행 — 속도·국소 봉쇄 우선 |
| **Strategic Agents** | Core / SMO | 추론(Reasoning) | LLM과 장기 메모리로 **교차 도메인 캠페인 조율** — APT 근본원인 분석, 전역 보안 정책 적응 |

_표 8-1. 에이전틱 방어 3단 계층. 출처: Feng 등[^r37]을 재구성._

이 계층 구조는 §5의 "O-DU 초저지연 필터 → Near-RT RIC 앙상블 IDS → Non-RT RIC/클라우드 심층분석"이라는 **2~3단 설계**와 정확히 같은 원리를 에이전트 자율성 관점에서 재서술한 것입니다.

### 8.3 RIC 내 에이전틱 감시 — "입력 검증"에서 "의도 검증"으로

Feng 등[^r37]은 RIC에 배치된 감독 에이전트가 **행동 모델링**(시그니처 검사가 아니라)으로 다중 벤더 xApp/rApp의 입력-출력 상관관계를 감시해, 모델 드리프트나 적대적 조작을 나타내는 미묘한 이탈(예: 현재 트래픽과 맞지 않는 스펙트럼을 요청하는 앱)을 탐지한다고 설명합니다. 핵심 함수는 **정책 상충의 자율 해소**입니다 — 에이전트가 반사실적(counterfactual) 추론으로 동시 제어 정책들의 결합 효과를 E2 인터페이스에 커밋하기 전에 시뮬레이션하는, Ch5 §4의 CME(Conflict Management Engine)를 게임이론적으로 확장한 형태입니다.

> **핵심 전환**: 문법적으로 유효한 제어 명령이라도, 그 예측된 영향이 망을 불안정하게 만든다면 여전히 악성일 수 있습니다(예: 트래픽 피크 시점의 sleep 명령). 에이전트는 구문이 아니라 **예측된 의도·안정성 영향**을 검증합니다[^r37]. Near-RT RIC의 엄격한 제어 루프(§5의 지연 예산)를 위반하지 않도록, 이 행동 분석은 AI-네이티브 에어 인터페이스·엣지 TPU·의미론적 통신 프로토콜(원시 데이터 대신 의도 요약 교환)로 지연 장벽을 완화합니다.
{: .prompt-tip }

### 8.4 물리계층 방어 — Hyper-Adaptivity

Feng 등[^r37]은 에이전틱 AI가 물리계층 보안(PLS)을 **정적 속성에서 동적 기동으로** 전환한다고 봅니다. RL 에이전트가 재밍 전략을 자율적으로 학습·예측해 주파수·시간·공간 자원에 걸친 다차원 호핑을 실행하며, THz/mmWave 매시브 MIMO에서는 빔포밍 가중치를 동적으로 최적화해 도청자 방향의 사이드로브 누설을 무력화합니다. 핵심 개념은 **Hyper-Adaptivity** — 방어 에이전트가 채널의 코히런스 시간보다, 그리고 공격자의 OODA 루프보다 빠르게 결정을 내리는 것입니다. 구현 과제는 CSI 노후화·추정 오차와 광대역 스펙트럼 스캔의 전력 소모이며, ISAC(레이더 반사로 도청자 위치 추정)과 RIS(채널 자체를 물리적으로 재구성)가 이를 보완합니다.

### 8.5 NDT 기반 방어 — Sim-to-Real 검증과 능동 기만

![NDT 지원 방어 파이프라인 (출처: Feng 등[^r37], Fig. 8)](/assets/img/posts/6g-ai-ran/agentsec-fig8.png)
_그림 8-35. **NDT(Network Digital Twin) 지원 방어 파이프라인**. 좌측 물리 6G망(라이브)에서 인지형 재머가 6G BS를 공격하는 상황을, Sim-to-Real Transfer를 통해 우측 디지털 트윈 에뮬레이션(가상)에서 사전 검증(고신뢰 에뮬레이션 환경 → 위험 평가)한 뒤에만 실제 대응을 적용합니다. 우측 하단 기만 환경(Deception Environment)은 격리된 도메인에서 위상학적으로 동형인 팬텀 슬라이스로 공격 세션을 유도해 포렌식 정보를 추출합니다. 하단 "AGENTIC AI & Continuous Learning" 블록이 지속적 적대적 훈련과 합성 학습 데이터로 두 환경을 공진화시킵니다. 출처: [^r37], Fig. 8._

URLLC처럼 무오류 운영이 요구되는 미션 크리티컬 환경에서는 자율 에이전트가 라이브 물리망에서 시행착오를 할 수 없습니다. NDT(Network Digital Twin)가 고신뢰 결정론적 에뮬레이션 환경을 제공해, 매시브 MIMO 빔포밍을 재구성하기 전에 먼저 트윈에서 실행해 보고 인접 셀 간섭이나 xApp 정책 상충을 유발하지 않는지 정량 평가합니다. 이를 넘어 에이전틱 AI는 NDT를 **능동적 고상호작용 기만 환경**으로도 전환합니다 — 실제 인프라와 위상학적으로 동형인 팬텀 네트워크 슬라이스를 동적으로 오케스트레이션하고, 의심스러운 횡적 이동이 탐지되면 SDN 흐름 테이블을 자율 조작해 공격자의 세션을 이 가상 유인 환경으로 이관시켜 공격자의 툴체인·제로데이 공격 벡터에 대한 포렌식 정보를 추출합니다. NDT는 **안정성-가소성 딜레마**를 해소합니다 — 트윈 안에서는 급진적으로 실험적일 수 있지만(제로데이 해법 탐색), 물리망은 검증된 정책만 수용해 안정을 유지하는 "타임머신 효과"입니다.

### 8.6 자기보증 — 공급망·무결성

Feng 등[^r37]은 사전학습 모델·오염된 데이터셋의 공급망 위협에 대응해, **출처 증명 에이전트(provenance agents)**가 DLT(Distributed Ledger Technology)[^a-dlt]로 AI 구성요소의 전체 계보를 감사하고, 학습 데이터 진본성과 모델 가중치의 암호학적 서명을 자율 검증해 **검증 가능한 AI Bill of Materials**를 확립한다고 제안합니다 — Ch7 §5.2의 AI-BOM을 DLT로 강제하는 구체적 실현입니다. 무결성 보증은 정적 코드 분석을 넘어 **실행 기반 프롬프트 인젝션**에 대한 강건성까지 포괄해야 하며, 유입 계층에 배치된 "살균 에이전트(sanitization agents)"가 운영 데이터에 은닉된 명령을 걸러냅니다. 핵심 원칙은 신뢰가 가정이 아니라 **수학적으로 도출**되어야 한다는 것 — 에이전트가 자신의 추론 상태·도구 호출 행위·정책 드리프트를 런타임에 지속 모니터링하며 이상 감지 시 자율적으로 검증·격리·복구를 촉발합니다.

---

## 9. 이 장의 요약

- 탐지 연구는 **DoS/DDoS · 시그널링 스톰 · 재밍 · 이상탐지** 4대 축과, **취약점 테스팅 · API 보안 · 상충 완화 · Zero Trust** 라는 평가도구 고도화 축으로 나뉩니다[^r5].
- **Det-RAN**: RRC 무결성 보호 부재에 대응해 **DMRS 파일럿 물리계층 지문**을 사용하고, **dApp**에 배치해 **2 ms 실시간 제약**을 충족하며 미관측 시나리오에서 **85% 이상** 정확도를 얻습니다[^r11].
- **SpotLight**: **600+ KPI**를 100 ms로 수집, 64샘플 윈도우, **엣지 JVGAN 경량 필터 + 클라우드 정밀 분석**으로 F1 **+13%**, 보고 KPI **2.3–4배 감소**, 대역폭 **4–7배 절감**[^r12].
- **XAI + LLM**: Near-RT RIC에서 LSTM으로 DDoS를 **F1 > 0.96** 탐지하고, LIME/SHAP → LLM으로 자연어 설명을 생성합니다[^r13]. LLM 생성 시간은 초 단위이므로 **Non-RT 루프 전용**입니다[^r23].
- **경량 IDS 벤치마크**: DT ~0.04 ms vs **LSTM ~2,268 ms** — 세 자리 수 차이. **O-DU에 DT 필터 → Near-RT RIC에 앙상블 → 클라우드에 DL/LLM** 의 2~3단 구조가 유일하게 현실적입니다[^r25].
- **조기 탐지**: **DU에서 CU 진입 전에** 약 95% 정확도로 차단 가능하며, 트래픽 초기 전이 상태를 고려해야 합니다[^r26].
- **Self-healing**은 탐지 + 완화 액션 + 폐루프이며, 안전 조건은 **시간예산·설명가능성·상충관리·정형검증·회복거동**입니다. 그리고 **자율성이 곧 오탐 유도 공격의 표면**임을 잊지 말아야 합니다.
- **DAIL·BASec**은 탐지·완화의 신뢰 기반 자체를 분산·검증 가능하게 만듭니다 — DAIL은 비잔틴 강건 집계로 오염된 참여자를 배제하고, BASec은 블록체인으로 변조 불가능한 출처 증명·인센티브를 제공합니다[^r36].
- **에이전틱 방어 3단 계층**(Sentinel-엣지 지각 / Tactical-RIC 행동 / Strategic-Core 추론)은 §5의 2~3단 지연 설계를 자율 에이전트 구조로 재서술한 것이며, RIC 감시는 **"입력 검증"에서 "의도 검증"** 으로, 물리계층 방어는 **Hyper-Adaptivity**로, 공급망은 **DLT 기반 출처 증명**으로 진화합니다[^r37].
- **NDT 기반 방어**는 사전 검증(sim-to-real)과 능동 기만(팬텀 슬라이스)이라는 두 기능을 모두 제공하며, "안정성-가소성 딜레마"를 해소하는 타임머신 효과를 냅니다[^r37].

### 확인 체크리스트

- [ ] Det-RAN이 왜 물리계층 지문을 쓰고 왜 dApp에 배치했는지 설명할 수 있는가
- [ ] SpotLight의 엣지·클라우드 분업 동기(비용)를 설명할 수 있는가
- [ ] SpotLight가 "정상 데이터만으로 학습"해야 하는 이유를 설명할 수 있는가
- [ ] LIME/SHAP를 적대적 입력 탐지에 쓰는 방법을 설명할 수 있는가
- [ ] DT와 LSTM의 지연 차이가 배치 결정에 어떻게 반영되는지 말할 수 있는가
- [ ] 폐루프 자율 완화의 5가지 안전 조건을 나열할 수 있는가
- [ ] DAIL과 BASec이 각각 어떤 공격에 대응하는지 구분할 수 있는가
- [ ] Sentinel/Tactical/Strategic 에이전트의 배치 위치와 역할을 말할 수 있는가
- [ ] "입력 검증"과 "의도 검증"의 차이를 예시로 설명할 수 있는가

**다음 장**: [09. 프라이버시 보존형 AI 및 신뢰 실행 환경(TEE)](/posts/airan-09-privacy-tee/)

---

### 약어

[^a-ai]: **AI**(Artificial Intelligence): 인간의 학습·추론 능력을 컴퓨터로 구현하는 기술(인공지능)을 통칭합니다.
[^a-ric]: **RIC**(RAN Intelligent Controller): RAN을 지능적으로 제어·최적화하는 O-RAN의 핵심 컨트롤러로, Near-RT RIC(10ms~1s)과 Non-RT RIC(1s 이상)으로 나뉩니다.
[^a-du]: **DU**(Distributed Unit): RLC/MAC/상위 PHY 등 하위 프로토콜 스택을 처리하는 분산 장치로, O-RAN에서는 O-DU로 표기합니다.
[^a-lstm]: **LSTM**(Long Short-Term Memory): 장기 의존성을 학습할 수 있는 순환 신경망의 한 종류로, 시계열 분석에 널리 쓰입니다.
[^a-xai]: **XAI**(eXplainable AI): AI 모델의 판단 근거를 사람이 해석할 수 있게 설명하는 기술입니다.
[^a-llm]: **LLM**(Large Language Model): 방대한 텍스트로 학습되어 자연어 이해·생성 능력을 제공하는 대규모 언어 모델입니다.
[^a-ids]: **IDS**(Intrusion Detection System): 트래픽·이벤트를 분석해 침입을 탐지하는 시스템입니다.
[^a-o-ran]: **O-RAN**(Open Radio Access Network): RAN 구성요소를 개방형 표준 인터페이스로 분해해 멀티벤더 구성을 가능하게 하는 개방형 무선 액세스 네트워크 구조입니다.
[^a-dos]: **DoS**(Denial of Service): 자원을 고갈시켜 정상 서비스 제공을 방해하는 서비스 거부 공격입니다.
[^a-ddos]: **DDoS**(Distributed Denial of Service): 다수의 분산된 소스에서 동시에 수행하는 대규모 서비스 거부 공격입니다.
[^a-fl]: **FL**(Federated Learning): 원본 데이터를 공유하지 않고 각 노드가 국소 학습한 모델을 집계해 공동 모델을 만드는 연합 학습 기법입니다.
[^a-fcl]: **FCL**(Federated Continual Learning): 연합 학습에 지속 학습을 결합해 파괴적 망각을 완화하는 기법입니다.
[^a-drl]: **DRL**(Deep Reinforcement Learning): 심층 신경망을 함수 근사에 사용하는 강화학습입니다.
[^a-dbscan]: **DBSCAN**(Density-Based Spatial Clustering of Applications with Noise): 밀도 기반의 비지도 클러스터링 알고리즘입니다.
[^a-rl]: **RL**(Reinforcement Learning): 에이전트가 환경과 상호작용하며 보상을 최대화하는 정책을 학습하는 강화학습 기법입니다.
[^a-ml]: **ML**(Machine Learning): 데이터로부터 패턴을 학습해 예측·판단을 수행하는 AI의 핵심 분야입니다.
[^a-esn]: **ESN**(Echo State Network): 순환 신경망의 일종으로, 스트리밍 시계열 처리에 효율적입니다.
[^a-mlp]: **MLP**(Multi-Layer Perceptron): 완전연결 계층으로 구성된 기본적인 신경망 구조입니다.
[^a-ae]: **AE**(AutoEncoder): 입력을 압축·복원하며 정상 패턴을 학습해 이상 탐지에 쓰이는 신경망입니다.
[^a-cnn]: **CNN**(Convolutional Neural Network): 합성곱 연산으로 공간적 특징을 추출하는 신경망입니다.
[^a-vae]: **VAE**(Variational AutoEncoder): 잠재 분포를 학습하는 확률적 오토인코더 계열 생성 모델입니다.
[^a-gan]: **GAN**(Generative Adversarial Network): 생성자와 판별자가 경쟁하며 학습하는 생성형 신경망입니다.
[^a-bert]: **BERT**(Bidirectional Encoder Representations from Transformers): 양방향 문맥을 학습하는 Transformer 기반 언어 모델입니다.
[^a-cwe]: **CWE**(Common Weakness Enumeration): 소프트웨어 보안 약점의 표준 분류 체계입니다.
[^a-capec]: **CAPEC**(Common Attack Pattern Enumeration and Classification): 알려진 공격 패턴의 표준 분류 체계입니다.
[^a-rrc]: **RRC**(Radio Resource Control): 연결 수립·해제 등 무선 자원 제어를 담당하는 계층 3 프로토콜입니다.
[^a-api]: **API**(Application Programming Interface): 소프트웨어 구성요소 간 기능 호출 규약입니다.
[^a-rf]: **RF**(Random Forest): 다수의 결정 트리를 앙상블해 분류·회귀를 수행하는 머신러닝 알고리즘입니다.
[^a-svm]: **SVM**(Support Vector Machine): 마진을 최대화하는 초평면으로 분류하는 머신러닝 알고리즘입니다.
[^a-knn]: **k-NN**(k-Nearest Neighbors): 가장 가까운 k개 이웃의 다수결로 분류하는 알고리즘입니다.
[^a-dt]: **DT**(Decision Tree): 특징 조건의 분기로 분류하는 결정 트리 알고리즘으로, 추론이 매우 빠릅니다.
[^a-phy]: **PHY**(Physical Layer): 무선 신호의 변조·부호화를 담당하는 물리 계층입니다.
[^a-mac]: **MAC**(Medium Access Control): 무선 자원 스케줄링과 매체 접근을 제어하는 계층 2 프로토콜입니다.
[^a-sinr]: **SINR**(Signal-to-Interference-plus-Noise Ratio): 간섭과 잡음 대비 신호 세기의 비율로, 무선 링크 품질 지표입니다.
[^a-cqi]: **CQI**(Channel Quality Indicator): UE가 기지국에 보고하는 채널 품질 지표입니다.
[^a-o-fh]: **O-FH**(Open Fronthaul): O-DU와 O-RU를 연결하는 O-RAN의 개방형 프론트홀 인터페이스로, 본문에는 Open-FH로 표기되어 있습니다.
[^a-ran]: **RAN**(Radio Access Network): 단말과 코어망 사이에서 무선 접속을 담당하는 무선 액세스 네트워크입니다.
[^a-a2c]: **A2C**(Advantage Actor-Critic): 정책(액터)과 가치(크리틱)를 함께 학습하는 강화학습 알고리즘입니다.
[^a-urllc]: **URLLC**(Ultra-Reliable Low-Latency Communication): 5G의 초신뢰·저지연 통신 서비스 유형입니다.
[^a-ng-ran]: **NG-RAN**(Next Generation RAN): 5G 표준의 무선 액세스 네트워크입니다.
[^a-kpi]: **KPI**(Key Performance Indicator): 네트워크 성능을 나타내는 핵심 성능 지표입니다.
[^a-rlc]: **RLC**(Radio Link Control): 분할·재전송 등 무선 링크 제어를 담당하는 계층 2 프로토콜입니다.
[^a-pdcp]: **PDCP**(Packet Data Convergence Protocol): 헤더 압축·암호화를 담당하는 계층 2 프로토콜입니다.
[^a-uav]: **UAV**(Unmanned Aerial Vehicle): 무인 항공기(드론)입니다.
[^a-sc]: **SC**(Software Community): O-RAN Alliance의 오픈소스 소프트웨어 커뮤니티로, O-RAN 참조 구현을 개발합니다.
[^a-ue]: **UE**(User Equipment): 이동통신 네트워크에 접속하는 단말 장치입니다.
[^a-rsrp]: **RSRP**(Reference Signal Received Power): 기준 신호의 수신 전력으로, 커버리지 품질 지표입니다.
[^a-rsrq]: **RSRQ**(Reference Signal Received Quality): 기준 신호의 수신 품질 지표입니다.
[^a-nr]: **NR**(New Radio): 3GPP가 정의한 5G의 무선 접속 기술 표준입니다.
[^a-rssi]: **RSSI**(Received Signal Strength Indicator): 수신 신호 세기 지표입니다.
[^a-prb]: **PRB**(Physical Resource Block): 무선 자원 할당의 기본 단위입니다.
[^a-fbs]: **FBS**(False Base Station): 정상 기지국을 가장해 단말을 유인하는 가짜 기지국입니다.
[^a-p2p]: **P2P**(Peer-to-Peer): 중앙 서버 없이 노드 간 직접 통신하는 분산 구조입니다.
[^a-dmrs]: **DMRS**(Demodulation Reference Signal): 수신 신호의 복조를 위해 삽입되는 기준 신호입니다.
[^a-nn]: **NN**(Neural Network): 뉴런을 모방한 노드들의 연결로 학습하는 신경망 모델을 통칭합니다.
[^a-tcp]: **TCP**(Transmission Control Protocol): 연결 지향으로 신뢰성 있는 전송을 보장하는 전송 계층 프로토콜입니다.
[^a-snr]: **SNR**(Signal-to-Noise Ratio): 잡음 대비 신호 세기의 비율입니다.
[^a-ood]: **OOD**(Out of Distribution): 학습 데이터의 분포를 벗어난 입력·상태를 뜻합니다.
[^a-kpm]: **KPM**(Key Performance Measurement): E2 인터페이스로 수집되는 RAN 핵심 성능 측정 지표입니다.
[^a-lime]: **LIME**(Local Interpretable Model-agnostic Explanations): 개별 예측의 주변을 국소 근사해 판단 근거를 설명하는 XAI 기법입니다.
[^a-shap]: **SHAP**(SHapley Additive exPlanations): 게임이론의 섀플리 값으로 각 특징의 기여도를 산출하는 XAI 기법입니다.
[^a-icmp]: **ICMP**(Internet Control Message Protocol): 네트워크 진단·오류 보고에 쓰이는 인터넷 제어 메시지 프로토콜입니다.
[^a-udp]: **UDP**(User Datagram Protocol): 연결 설정 없이 데이터그램을 전송하는 전송 계층 프로토콜입니다.
[^a-dns]: **DNS**(Domain Name System): 도메인 이름을 IP 주소로 변환하는 시스템입니다.
[^a-gtp-u]: **GTP-U**(GPRS Tunnelling Protocol – User plane): 이동통신망에서 사용자 데이터를 터널링해 전달하는 프로토콜입니다.
[^a-kde]: **KDE**(Kernel Density Estimation): 데이터의 확률 밀도를 커널 함수로 추정하는 통계 기법입니다.
[^a-fpr]: **FPR**(False Positive Rate): 정상을 공격으로 잘못 판정하는 비율(오탐률)입니다.
[^a-fnr]: **FNR**(False Negative Rate): 공격을 정상으로 놓치는 비율(미탐률)입니다.
[^a-zsm]: **ZSM**(Zero-touch Service Management): 사람의 개입 없이 네트워크·서비스를 자동 관리하는 ETSI의 프레임워크입니다.
[^a-dl]: **DL**(Deep Learning): 다층 신경망으로 표현을 학습하는 심층 학습입니다.
[^a-cu]: **CU**(Central Unit): PDCP 이상의 상위 계층을 처리하는 중앙 장치로, O-RAN에서는 O-CU로 표기합니다.
[^a-voip]: **VoIP**(Voice over IP): IP 네트워크로 음성을 전송하는 기술입니다.
[^a-cdf]: **CDF**(Cumulative Distribution Function): 확률변수가 특정 값 이하일 확률을 나타내는 누적 분포 함수입니다.
[^a-mcs]: **MCS**(Modulation and Coding Scheme): 채널 상태에 따라 변조 방식과 부호율을 정하는 조합입니다.
[^a-lldp]: **LLDP**(Link Layer Discovery Protocol): 인접 장비 정보를 교환해 네트워크 토폴로지를 파악하는 링크 계층 프로토콜입니다.
[^a-rlv]: **RLV**(Real-time Link Verification): 링크 지연 등의 메트릭을 실시간 검증해 위조된 토폴로지 링크를 탐지하는 방어 기법입니다.
[^a-smo]: **SMO**(Service Management and Orchestration): O-RAN 전체의 서비스 관리·오케스트레이션 프레임워크로, Non-RT RIC를 포함합니다.
[^a-prpa]: **PRPA**(Perception-Reasoning-Planning-Action): 에이전틱 AI가 관측을 인식하고 추론·계획한 뒤 행동으로 옮기는 폐루프 제어 사이클입니다.
[^a-dlt]: **DLT**(Distributed Ledger Technology): 블록체인으로 대표되는, 다수 노드가 원장을 분산 공유·검증하는 기술입니다.

## References

[^r2]: M. Polese, N. Mohamadi, S. D'Oro, L. Bonati, and T. Melodia, "Beyond connectivity: An open architecture for AI-RAN convergence in 6G," *arXiv preprint* arXiv:2507.06911v2, Dec. 2025.
[^r5]: C. Benzaïd, T. Taleb, R. Cavalcanti, and A. Sánchez, "AI in Open RAN security: Enabler and threat — A comprehensive survey and future perspectives," *TechRxiv preprint*, Dec. 2025.
[^r6]: S. Soltani, A. Amanloo, M. Shojafar, and R. Tafazolli, "Intelligent control in 6G open RAN: Security risk or opportunity?" *IEEE Open Journal of the Communications Society*, 2025.
[^r8]: A. Chiejina, B. Kim, K. Chowdhury, and V. K. Shah, "System-level analysis of adversarial attacks and defenses on intelligence in O-RAN based cellular networks," in *Proc. ACM WiSec*, 2024.
[^r11]: A. Scalingi, S. D'Oro, F. Restuccia, T. Melodia, and D. Giustiniano, "Det-RAN: Data-driven cross-layer real-time attack detection in 5G open RANs," in *Proc. IEEE INFOCOM*, 2024, pp. 41–50.
[^r12]: C. Sun, U. Pawar, M. Khoja, X. Foukas, M. K. Marina, and B. Radunovic, "SpotLight: Accurate, explainable and efficient anomaly detection for open RAN," in *Proc. 30th Annual Int. Conf. on Mobile Computing and Networking (MobiCom)*, Washington D.C., USA, Nov. 2024, pp. 923–937.
[^r13]: S. Chatzimiltis, M. Shojafar, M. B. Mashhadi, and R. Tafazolli, "Interpretable anomaly-based DDoS detection in AI-RAN with XAI and LLMs," *arXiv preprint* arXiv:2507.21193, Jul. 2025.
[^r19]: Q. Zhang, Z. Xiong, and Z. M. Mao, "Safeguard is a double-edged sword: Denial-of-service attack on large language models," *arXiv preprint* arXiv:2410.02916, 2024.
[^r23]: A. Mekrache, M. Mekki, A. Ksentini, B. Brik, and C. Verikoukis, "On combining XAI and LLMs for trustworthy zero-touch network and service management in 6G," *IEEE Communications Magazine*, 2024.
[^r25]: S. Ben Khalifa, R. Taheri, and Z. Pooranian, "Lightweight intrusion detection baselines for Open RAN xApps," in *Proc. IEEE ICC Workshops*, 2026.
[^r26]: B. M. Xavier, M. Dzaferagic, D. Collins, G. Comarela, M. Martinello, and M. Ruffini, "Machine learning-based early attack detection using Open RAN intelligent controller," in *Proc. IEEE ICC*, 2023.
[^r36]: A. Braeken, D. Deac, T. L. Nguyen, G. Gür, Q.-V. Pham, C. Yapa, P. G. Vinueza-Naranjo, H. Carvajal Mora, C. Moremada, and M. Liyanage, "6G AI security: From fundamentals to offensive and defensive landscape in 6G," *IEEE Communications Surveys & Tutorials*, vol. 28, 2026.
[^r37]: H. Feng, T. R. Gadekallu, Y. Xia, Y. Zhao, Z. Wen, J. Cai, P. Bhattacharya, K. Fang, and M. Liyanage, "Agentic AI security in 6G networks: A survey of emerging attack vectors, vulnerabilities, and defenses," *IEEE Open Journal of the Communications Society*, vol. 7, pp. 6334–6370, 2026.
