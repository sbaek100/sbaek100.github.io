---
title: "[6G AI-RAN] 03. RAN 내 AI/ML 워크플로우 및 데이터 파이프라인"
date: 2026-07-30 09:30:00 +0900
categories:
  - 2.미래기술
  - 6G AI RAN
  - Part I 기초·아키텍처
tags:
  - AI-ML-Workflow
  - O-RAN
  - xApp
  - rApp
  - Federated-Learning
  - KPM
math: true
mermaid: false
---

# RAN 내 AI/ML 워크플로우 및 데이터 파이프라인

## 들어가며 — 공격은 "파이프라인 단계"를 겨냥한다

Part II에서 다룰 위협들은 대부분 **"어느 단계를 겨냥하는가"** 로 분류됩니다. 학습 단계를 노리면 오염(poisoning), 추론 단계를 노리면 회피(evasion), 배포 경로를 노리면 공급망 공격입니다. 따라서 위협을 이해하려면 **파이프라인을 먼저 알아야 합니다.**

이 장에서 다루는 내용:

1. O-RAN(Open Radio Access Network)[^a-o-ran] 표준 AI(Artificial Intelligence)[^a-ai]/ML(Machine Learning)[^a-ml] 워크플로우 6단계
2. 데이터는 어디서 오는가 — E2/A1/O1과 KPM(Key Performance Measurement)[^a-kpm]
3. AI/ML 모델의 배포 옵션 (어디에 얹히는가)
4. 파이프라인 구성요소의 배치 유연성과 그 대가
5. 타이밍 예산 — 파이프라인이 실시간 제어를 만날 때
6. 데이터 파편화와 컨텍스트 공유 문제
7. 오설정(misconfiguration) — 가장 흔하고 가장 과소평가된 위험

---

## 1. O-RAN AI/ML 워크플로우 6단계

O-RAN Alliance가 규정한 표준 AI/ML 워크플로우입니다[^r5].

![O-RAN AI/ML 워크플로우 (출처: Benzaïd 등[^r5], Fig. 3)](/assets/img/posts/6g-ai-ran/aisurvey-fig3-p7.png)
_그림 3-1. **O-RAN AI/ML 워크플로우**. O-RAN 인프라(O-RAN 노드)에서 E2/A1/O1을 통해 Raw Data가 수집되고 → Data Preparation → AI/ML Training → AI/ML Model Management(Validation → Certification → On-boarding → Deployment) → AI/ML Inference → AI/ML Continuous Operations의 순환 구조입니다. 검증 실패 모델은 Retrain 루프로 돌아가고, 추론 결과는 Actions(E2/A1/O1)로 인프라에 반영되며, Model Performance Feedback이 다시 데이터 수집으로 돌아옵니다. 출처: [^r5], Fig. 3._

| 단계 | 하는 일 | 입출력 | 이 단계를 노리는 공격 (Ch6) |
|---|---|---|---|
| **① Data Collection** | E2/A1/O1에서 네트워크 사용량·지연·채널 품질 등 원시 데이터 수집[^r5] | → Raw Data | 데이터 오염, 재밍을 통한 물리적 데이터 왜곡 |
| **② Data Preparation** | 특정 AI/ML 모델에 맞게 데이터 정제·변환 (학습용/추론용 분기)[^r5] | Raw Data → Training Data / Inference Data | 라벨 플리핑, 백도어 트리거 삽입 |
| **③ AI/ML Training** | 지도·비지도·**RL**[^a-rl]·**FL**[^a-fl] 방식으로 학습[^r5] | Training Data → 모델 | 모델 오염, FL 클라이언트 악성화 |
| **④ AI/ML Model Management** | **Validation → Certification → On-boarding → Deployment**. 검증 통과 모델은 SMO[^a-smo]/Non-RT RIC[^a-ric] 카탈로그에 게시, 실패 모델은 **Retrain**[^r5] | 모델 → 배포된 모델 | 검증 우회, 카탈로그 위조, **ML 공급망 공격** |
| **⑤ AI/ML Inference (Serving)** | 배포된 모델의 추론을 AI/ML 지원 솔루션이 활용 — **A1/E2로 정책 관리, E2로 제어 액션, O1로 설정 관리**[^r5] | Inference Data → Actions | 회피 공격, 모델 탈취, 재프로그래밍 |
| **⑥ AI/ML Continuous Operations** | 배포 모델의 성능 지표 모니터링 → 성능 저하 시 **모델 재선택 또는 재학습 트리거**[^r5] | 성능 피드백 | **자원 고갈로 모니터링 지연 유도**, 재학습 루프 악용 |

> **④ Model Management가 보안의 관문입니다.** Validation·Certification·On-boarding이라는 4단 검증 게이트를 통과한 모델만 카탈로그에 올라가고 배포됩니다. 반대로 말하면 **이 게이트를 우회하거나, 게이트 이전의 학습 데이터를 오염시키면 "검증된 악성 모델"이 만들어집니다.** Ch7의 AIBOM(AI Bill of Materials)[^a-aibom]이 이 문제를 정면으로 다룹니다.
{: .prompt-danger }

> **⑥의 재학습 루프는 양날의 검입니다.** 성능 저하 시 자동 재학습은 복원력을 주지만, 공격자가 의도적으로 성능을 떨어뜨려 **재학습을 유발하고 그 시점에 오염 데이터를 주입**하는 2단 공격이 가능합니다.
{: .prompt-warning }

---

## 2. 데이터는 어디서 오는가 — 인터페이스와 KPM

### 2.1 인터페이스별 데이터 역할

| 인터페이스 | 수집 방향 (Raw Data) | 작용 방향 (Actions) |
|---|---|---|
| **E2** | E2 노드(O-DU[^a-du], O-CU-CP[^a-cu], O-CU-UP)의 KPM — 셀·UE[^a-ue] 단위 성능 지표 | **제어 액션(control action)** — 무선자원 제어 |
| **A1** | Non-RT RIC → Near-RT RIC의 Enrichment Information | **정책(policy)** 관리 |
| **O1** | 관리 요소의 FCAPS[^a-fcaps] 데이터 | **설정 관리(configuration)** |
| **Y1** | Near-RT RIC → 인증된 소비자에게 RAN[^a-ran] 분석 데이터 제공[^r5] | (소비 전용) |

`E2SM-KPM`(E2 Service Model — Key Performance Measurement)[^a-e2sm]이 KPM 수집의 표준 서비스 모델이며, ETRI(Electronics and Telecommunications Research Institute)[^a-etri] 정리[^r28]에서도 프로그래머블 RAN(Radio Access Network)을 실현하는 핵심 인터페이스로 지목됩니다.

### 2.2 실제 연구에서 사용된 특징(feature) 예시

각 연구가 어떤 계층의 어떤 지표를 썼는지 보면, 데이터 파이프라인의 실체가 분명해집니다.

| 연구 | 계층 | 사용 특징 |
|---|---|---|
| O-DU 수준 DoS[^a-dos] 조기 탐지[^r26] | PHY[^a-phy], MAC[^a-mac] | **SINR[^a-sinr], CQI[^a-cqi], bitrate, 패킷 수** |
| Near-RT RIC DDoS[^a-ddos] 탐지[^r13] | E2 노드 KPM | 다변량 시계열 KPM (UE별 다운링크 비트레이트 등) |
| 재밍 탐지 (Bayesian 지원) | PHY, MAC, RLC[^a-rlc], PDCP[^a-pdcp] | **크로스 레이어 KPI[^a-kpi]**[^r5] |
| 이상 UE 탐지 (O-RAN SC[^a-oransc] xApp) | 무선 신호 지표 | **RSRP[^a-rsrp], RSRQ[^a-rsrq], NR-RSSI[^a-rssi], throughput, PRB[^a-prb] 사용률**[^r5] |
| Det-RAN[^r11] | PHY + 상위 계층 | **DMRS[^a-dmrs] 파일럿 시퀀스**로 UE 지문 생성 + 크로스 레이어 특징 |
| SpotLight[^r12] | 프론트홀·스레드·전송 | FH[^a-fh] 처리량, FH 관련 스레드 런타임, DL[^a-dl] TCP[^a-tcp] 처리량, SINR/SNR[^a-snr] |

> **주목**: Det-RAN[^r11]은 **RRC(Radio Resource Control)[^a-rrc] 계층에 무결성 보호가 없다**는 근본 문제에 대응하기 위해, 상위 계층 신뢰가 불가능한 상황에서 **물리계층 DMRS 파일럿을 지문으로 사용**합니다. "데이터 소스의 신뢰성"이 파이프라인 설계를 결정한 사례입니다. (Ch8에서 상세히)
{: .prompt-tip }

---

## 3. AI/ML 모델의 배포 옵션 — 어디에 얹히는가

Yungaicela-Naula 등[^r7]은 O-RAN에서 AI/ML 모델의 배포 옵션을 **단일(Single)** 과 **분산(Distributed)** 으로 나눕니다.

| 대분류 | 배포 옵션 | 위치 | 연구 성숙도[^r7] |
|---|---|---|---|
| **Single**<br/>(단일 배치) | RIC function | RIC 플랫폼 기능 자체 | **최소 탐구 또는 미탐구** |
| | rApp | Non-RT RIC | 활발 |
| | xApp | Near-RT RIC | 활발 |
| | CU 또는 DU (dApp 포함) | O-CU / O-DU 내부 | 진행 중 |
| **Distributed**<br/>(분산 배치) | Coordinating apps | rApp ↔ xApp 협력 | 진행 중 |
| | Model splitting | 노드 간 모델 분할 | **최소 탐구 또는 미탐구** |
| | Model sharing | RIC → UE 등 | **최소 탐구 또는 미탐구** |
| | Federated learning | Near-RT RIC(에이전트) + Non-RT RIC(집계) | **최소 탐구 또는 미탐구** |

_표 3-1. AI/ML 배포 옵션 분류. Yungaicela-Naula 등[^r7]의 Fig. 2를 표로 재구성. 원 논문은 **RIC 플랫폼 기능 내 AI/ML, model splitting, model sharing, FL**을 "최소 탐구 또는 미탐구 영역(gray boxes)"으로 표시합니다._

### 3.1 배포 옵션별 특성과 위험

| 배포 옵션 | 예시 | 장점 | 보안 위험 |
|---|---|---|---|
| **RIC function** | RIC 플랫폼 기능 자체에 ML 내장 | 플랫폼 차원 최적화 | 침해 시 **모든 xApp에 영향** (연구 부족 영역) |
| **rApp** | 셀 on/off를 RL로 결정 (Rimedo Labs)[^r7] | 장기 정책 최적화 | 정책 오염이 광범위·지속적 |
| **xApp** | 트래픽 스티어링, 이상 탐지 | near-RT 대응 | **제3자 코드가 무선자원 직접 제어** |
| **CU/DU (dApp)** | 물리계층 기반 실시간 탐지[^r11] | 최저 지연 | 프로토콜 스택 내부 침해 = 치명적 |
| **Coordinating apps** | rApp(정책) + xApp(실행) 협력, 에너지 절감 플랫폼[^r7] | 시간 척도 분리 최적화 | **앱 간 상충(conflict)** → 5절/Ch5 |
| **Model splitting** | 모델을 노드별로 분할 | 자원 분산 | **체인 모델 공격** — 한 조각 침해가 연쇄 전파 (Ch12) |
| **Model sharing** | RIC의 rogue BS 탐지 모델을 **UE로 전송**[^r7] | 실시간 탐지, UE 부담↓, 전체 UE 동시 갱신 | **모델이 단말로 배포됨 = 모델 탈취/역공학 표면** |
| **Federated learning** | Near-RT RIC에 FL 에이전트, Non-RT RIC에 집계 서버[^r5] | 프라이버시 보존, 협력 탐지 | **악성 클라이언트 오염**, **집계 서버의 데이터 재구성 공격** (Ch9) |

> Yungaicela-Naula 등[^r7]은 **model sharing** 사례로 흥미로운 구조를 보고합니다. Near-RT RIC의 xApp이 rogue base station(RBS)[^a-rbs] 탐지 모델을 학습한 뒤, 그 모델을 **UE에 전송**해 UE가 직접 RBS를 탐지합니다. 이점은 실시간 탐지·UE 연산부담 감소·전체 UE 동시 업데이트지만, 보안 관점에서는 **탐지 모델이 공격자 손에 들어갈 수 있는 구조**입니다(공격자도 UE를 소유할 수 있으므로). 탐지 회피 설계가 쉬워집니다.
{: .prompt-danger }

### 3.2 실제 xApp/rApp 사례

Yungaicela-Naula 등[^r7]이 정리한 산업계·연구계·O-RAN SC(O-RAN Software Community)의 앱 사례입니다.

| 제공자 | 앱 내용 | 배치 | AI/ML |
|---|---|---|---|
| Rimedo Labs | rApp이 사용자 처리량·전력 소비 기반으로 셀 on/off | Single | **RL** |
| ONF[^a-onf] | rApp이 셀1 부하 감시·차단 결정, xApp이 트래픽을 셀2로 이동 | **Dist** | 미명시 |
| Nokia | xApp이 gNB[^a-gnb]가 다른 gNB 영역을 커버하도록 유도해 셀 종료 가능화 | Single | 미사용 |
| Ericsson | rApp이 라디오 유닛·네트워크를 감시해 비효율 근본원인 파악·권고 | Single | 네트워크 클러스터링·모델링용 AI/ML |

---

## 4. 파이프라인 구성요소의 배치 유연성

O-RAN 아키텍처는 AI/ML 파이프라인 구성요소를 **세 위치 중 하나에 유연하게 배치**할 수 있게 합니다[^r5].

| 배치 위치 | 선택 근거 | 보안 함의 |
|---|---|---|
| **Non-RT RIC 내부** | 지연 요구가 낮고 RAN 데이터 접근이 필요한 학습 | SMO 신뢰 도메인 내 — 상대적으로 안전 |
| **Non-RT RIC 외부, SMO 내부** | 자원이 더 필요하거나 별도 수명주기 관리 | SMO 내부 마이크로서비스 간 인증 필요 |
| **SMO 외부** | 대규모 학습, 외부 데이터 결합 | **신뢰 경계를 벗어남** — 데이터 반출·모델 반입 모두 검증 필요 |

배치 결정 요인은 **학습 태스크, 자원 가용성, 보안 요구, 애플리케이션 지연 요구**입니다[^r5].

> **설계 원칙**: 파이프라인 구성요소가 SMO(Service Management and Orchestration) 외부로 나가는 순간, ① 학습 데이터의 **반출(egress)** 과 ② 학습된 모델의 **반입(ingress)** 이라는 두 개의 새로운 신뢰 경계가 생깁니다. Ch7의 Zero Trust와 Ch9의 PETs(Privacy-Enhancing Technologies)[^a-pets]가 각각을 다룹니다.
{: .prompt-tip }

---

## 5. 타이밍 예산 — 파이프라인이 실시간 제어를 만날 때

AI 파이프라인은 개념적으로 순환 구조지만, **실제 배포에서는 한 바퀴 도는 데 걸리는 시간이 제약**입니다.

![AI 기반 xApp 제어의 타이밍 파이프라인과 지연 문제 (출처: Salmi 등[^r4], Fig. 10)](/assets/img/posts/6g-ai-ran/ainative-fig10.png)
_그림 3-2. AI 기반 xApp 제어의 타이밍 파이프라인. 데이터 수집 → 전달 → 추론 → 제어 액션 적용까지 각 구간이 누적되어 near-RT 예산을 소진합니다. 출처: [^r4], Fig. 10._

| 구간 | 소요 요인 | 최적화 방향 |
|---|---|---|
| 데이터 수집 (E2 보고) | 보고 주기, KPM 개수 | 보고 주기와 정확도 트레이드오프 |
| E2 전달 | 미드홀 지연 | dApp으로 내려 프로토콜 스택 내부 처리[^r11] |
| 추론 | 모델 복잡도 | **경량 모델 선택** — Ch8·Ch11의 지연 실측[^r25] |
| 제어 액션 적용 | E2 제어 요청 처리 | 배치 처리 |

Polese 등[^r2]이 AI-RAN 사이트에서 측정한 **워크로드 배포 지연**은 파이프라인의 또 다른 현실적 제약을 보여줍니다.

![AI-RAN 사이트에서의 워크로드 배포 지연 (출처: Polese 등[^r2], Fig. 6)](/assets/img/posts/6g-ai-ran/beyondconn-fig6.png)
_그림 3-3. LLM(Large Language Model)[^a-llm] 모델·파라미터 규모별 순차 배포 지연. 모델이 커지면 배포 자체가 지연 요인이 됩니다 — 즉 "모델을 즉시 교체해 방어한다"는 전략에는 물리적 하한이 있습니다. 출처: [^r2], Fig. 6._

---

## 6. 데이터 파편화와 컨텍스트 공유 문제

Salmi 등[^r4]은 분산 배치의 구조적 결함을 지적합니다: **CU들에 분산된 dApp들이 서로의 컨텍스트를 공유하지 못한다**는 점입니다.

![CU 간 분산 dApp의 컨텍스트 공유 부재 (출처: Salmi 등[^r4], Fig. 11)](/assets/img/posts/6g-ai-ran/ainative-fig11.png)
_그림 3-4. 분산된 dApp들이 컨텍스트를 공유하지 못하는 상황. 각 dApp은 부분 관측만으로 결정을 내립니다. 출처: [^r4], Fig. 11._

이 문제의 해법으로 **CSL**(Context Sharing Layer)[^a-csl]이 제안됩니다.

![제안된 컨텍스트 공유 계층(CSL) 아키텍처 (출처: Salmi 등[^r4], Fig. 12)](/assets/img/posts/6g-ai-ran/ainative-fig12.png)
_그림 3-5. **CSL 아키텍처** — 다중 소스 텔레메트리를 동기화하여 컨텍스트 일관성 있는(context-coherent) 제어를 달성합니다. 출처: [^r4], Fig. 12._

| 관점 | 컨텍스트 공유 부재의 결과 | 보안적 의미 |
|---|---|---|
| 성능 | 부분 관측 기반 결정 → 국소 최적, 전역 비효율 | — |
| 탐지 | **각 dApp/xApp이 공격의 일부만 관측** | **분산 공격이 탐지 임계값 아래로 숨을 수 있음** |
| 대응 | 대응 조치 간 상충 | 공격자가 상충을 유발해 자기 상쇄 유도 |

> **탐지 관점의 핵심**: 컨텍스트 공유가 없으면 "각 노드에서는 정상, 전체로는 공격"인 패턴을 놓칩니다. 반대로 CSL 같은 공유 계층은 **탐지 능력을 높이지만 동시에 새로운 고가치 표적**(모든 컨텍스트가 모이는 곳)이 됩니다.
{: .prompt-warning }

---

## 7. 오설정 — 가장 흔하고 가장 과소평가된 위험

Yungaicela-Naula 등[^r7]은 O-RAN 오설정을 **통합·운영 / 인에이블링 기술(SDN(Software-Defined Networking)[^a-sdn]·NFV(Network Functions Virtualization)[^a-nfv]) / AI/ML** 세 범주로 분석합니다. 오설정은 두 경로로 해를 끼칩니다.

| 경로 | 결과 |
|---|---|
| **직접(direct)** | O-RAN 성능에 즉시 영향 (병목, 자원 저활용) |
| **간접(indirect)** | **보안 위협에 대한 취약성 증가** — 즉 오설정이 공격의 전제조건이 됨 |

### 7.1 통합·운영 오설정

O-RAN은 다수 제조사, 다수 RAT(Radio Access Technology)[^a-rat](WiFi/NR(New Radio)[^a-nr]), 다양한 UE(차량·IoT(Internet of Things)[^a-iot]), 소프트웨어 버전(E2SM), 애플리케이션(eMBB(enhanced Mobile Broadband)[^a-embb]/URLLC(Ultra-Reliable Low-Latency Communication)[^a-urllc])이 섞여 **통합·운영 자체가 극도로 어렵습니다**[^r7].

| 유형 | 구체 예 |
|---|---|
| 표준 절차 미준수 | Near-RT RIC의 **표준 xApp 발견·등록·구독 절차 미준수** → 자동 xApp 배포 실패 |
| SA[^a-sa]/NSA[^a-nsa] 혼재 | LTE[^a-lte]·5G를 NSA에 통합하는 복잡한 과정에서 병목·자원 저활용 발생 |
| 불필요·비보안 요소 | 불필요한 포트·서비스·계정·권한, 기본 설정 의존 → **공격자에 노출 + 성능 저하** |

### 7.2 보안 기능 오설정 — 특히 위험

효과적인 O-RAN 보호에는 세 가지가 필요합니다[^r7]: ① 모든 인터페이스의 통신 보호, ② 통신 종단의 **신뢰 기반 인증**, ③ 신뢰된 인증기관(CA, Certificate Authority)[^a-ca]을 통한 신원 프로비저닝.

3GPP(3rd Generation Partnership Project)[^a-3gpp]와 O-RAN Alliance는 백홀·미드홀(F1)·FH·O1·E2·A1·O2·E1·Xn 인터페이스에 대한 보안 보증 표준을 발표했고, **SSHv2(Secure Shell v2)[^a-ssh], TLS(Transport Layer Security)[^a-tls], DTLS(Datagram Transport Layer Security)[^a-dtls], IPsec(Internet Protocol Security)[^a-ipsec], MACsec(Media Access Control Security)[^a-macsec]** 같은 검증된 프로토콜을 채택했습니다[^r7].

> 그러나 Yungaicela-Naula 등[^r7]의 경고는 날카롭습니다:
> *"보안 프로토콜의 복잡성 — 여러 정교한 설정과 세부사항 — 은 이 프로토콜들을 **오설정에 취약**하게 만든다. … 그러나 이런 라이브러리를 **부적절하게 사용하면 rogue RU, DU, CU, 또는 RIC가 도입되는 것에 네트워크가 노출**된다."*
> 즉 **"TLS를 쓴다"가 아니라 "TLS를 올바르게 설정했다"** 가 보안의 실체입니다.
{: .prompt-danger }

### 7.3 AI/ML 오설정

가장 O-RAN 특유의 문제는 **다수 AI/ML 앱 간의 상충**입니다.

![O-RAN 아키텍처 — Near-RT RIC 내 Conflict mitigation 블록 포함 (출처: Yungaicela-Naula 등[^r7], Fig. 1)](/assets/img/posts/6g-ai-ran/misconf-fig1.png)
_그림 3-6. O-RAN Alliance/3GPP 아키텍처. Near-RT RIC 내부에 **Subscription Management, Security, Conflict mitigation, Shared data layer, Messaging Infrastructure**가 명시되어 있습니다. 출처: [^r7], Fig. 1._

상충의 고전적 예는 **MLB**(Mobility Load Balancing)[^a-mlb]와 **MRO**(Mobility Robustness Optimization)[^a-mro]입니다.

![상충하는 MLB와 MRO (출처: Yungaicela-Naula 등[^r7], Fig. 3)](/assets/img/posts/6g-ai-ran/misconf-fig3.png)
_그림 3-7. SON(Self-Organizing Network)[^a-son] 기능인 MLB와 MRO가 동일 파라미터를 반대 방향으로 조정해 상충하는 상황. 출처: [^r7], Fig. 3._

AI/ML 기반 상충 탐지·완화 방법론:

![xApp 간 상충 관리 — AI/ML을 이용한 탐지와 완화 (출처: Yungaicela-Naula 등[^r7], Fig. 4)](/assets/img/posts/6g-ai-ran/misconf-fig4.png)
_그림 3-8. xApp 상충의 탐지·완화 흐름. 음영 구성요소는 AI/ML 기법이 필요한 부분입니다. 출처: [^r7], Fig. 4._

![상충 xApp 탐지·완화 방법 (출처: Yungaicela-Naula 등[^r7], Fig. 5)](/assets/img/posts/6g-ai-ran/misconf-fig5.png)
_그림 3-9. 상충 xApp 탐지·완화 상세 방법. 출처: [^r7], Fig. 5._

Benzaïd 등[^r5]이 정리한 O-RAN 상충 완화 연구 동향:

| 접근 | 내용 |
|---|---|
| Conflict Mitigation 확장 | Near-RT RIC의 Conflict Mitigation 구성요소에 **충돌 탐지·해소 에이전트** 기능 블록 추가, AI/ML 활용 권고 |
| **Team learning** | xApp 간 **계획된 액션(planned actions)을 공유**하여 DRL[^a-drl] 에이전트가 비상충 액션을 선택 |
| **Knowledge transfer** | 여러 사전학습 DRL xApp의 정책 지식을 **증류(distill)** 해, 그들을 대신해 비상충 액션을 취하는 단일 DRL 모델 구축 |

> **보안 관점의 재해석**: 상충 완화는 성능 문제로 보이지만 실제로는 **가용성 보안 문제**입니다. Soltani 등[^r6]의 RIC 취약점 표에서 **V-03 "무선 접속 정책의 상충과 불일치"** 는 심각도 **High**로 분류되며, 원인으로 "RIC와 O-gNB 간 상충" 및 "다수 xApp에서 발생하는 상충", 영향으로 **서비스 중단·성능 저하**를 명시합니다. Ch5에서 이 표 전체를 다룹니다.
{: .prompt-warning }

---

## 8. 이 장의 요약

- O-RAN AI/ML 워크플로우는 **수집 → 준비 → 학습 → 모델관리(검증·인증·온보딩·배포) → 추론 → 지속운영**의 6단계 순환입니다[^r5].
- 데이터는 **E2/A1/O1**으로 들어오고, 액션도 **E2/A1/O1**으로 나갑니다. 즉 **같은 인터페이스가 관측과 제어를 동시에 담당**하므로, 인터페이스 침해는 곧 관측 왜곡 + 제어 탈취입니다.
- 모델 배포는 **Single**(RIC function / rApp / xApp / CU·DU) 또는 **Distributed**(coordinating apps / model splitting / model sharing / FL) 방식이며, 각 방식마다 고유한 위험을 가집니다[^r7].
- 파이프라인 구성요소는 Non-RT RIC 내부·SMO 내부·**SMO 외부**에 배치될 수 있고, SMO 외부 배치는 두 개의 새 신뢰 경계를 만듭니다[^r5].
- **타이밍 예산**이 모델 선택을 지배하고, **컨텍스트 공유 부재**가 분산 공격의 은신처를 만듭니다[^r4].
- **오설정**은 성능 문제를 넘어 rogue RU(Radio Unit)[^a-ru]/DU/CU/RIC 도입의 문을 열 수 있는 **보안 문제**입니다[^r7].

### 확인 체크리스트

- [ ] AI/ML 워크플로우 6단계를 순서대로 말할 수 있는가
- [ ] Model Management의 4단 게이트(Validation·Certification·On-boarding·Deployment)를 설명할 수 있는가
- [ ] 배포 옵션 중 model sharing이 왜 모델 탈취 위험을 만드는지 설명할 수 있는가
- [ ] 파이프라인이 SMO 외부에 배치될 때 생기는 두 신뢰 경계를 지목할 수 있는가
- [ ] MLB/MRO 상충이 왜 "가용성 보안 문제"인지 설명할 수 있는가

**다음 장**: [04. 6G AI-RAN 위협 모델링 및 공격 표면](/posts/airan-04-threat-modeling/) — Part II 시작

---

### 약어

[^a-o-ran]: **O-RAN**(Open Radio Access Network): 기지국을 표준 개방형 인터페이스로 분해하여 서로 다른 제조사의 장비 간 상호운용을 가능하게 하는 개방형 무선 접속망 아키텍처입니다. O-RAN Alliance가 규격을 제정합니다.
[^a-ai]: **AI**(Artificial Intelligence): 인공지능. 학습·추론·판단 등 인간의 지적 능력을 컴퓨터 시스템으로 구현하는 기술의 총칭입니다.
[^a-ml]: **ML**(Machine Learning): 기계학습. 명시적인 규칙 대신 데이터에서 패턴을 학습하여 예측·분류를 수행하는 AI의 핵심 기술 분야입니다.
[^a-kpm]: **KPM**(Key Performance Measurement): 셀·단말 단위의 처리량·지연 등 RAN 성능을 나타내는 핵심 성능 측정 지표로, E2 인터페이스를 통해 수집됩니다.
[^a-rl]: **RL**(Reinforcement Learning): 강화학습. 에이전트가 환경과 상호작용하며 보상을 최대화하는 행동 정책을 스스로 학습하는 기계학습 방법입니다.
[^a-fl]: **FL**(Federated Learning): 연합학습. 원본 데이터를 중앙으로 모으지 않고 각 참여 노드가 로컬에서 학습한 모델 갱신값만 공유하여 전역 모델을 만드는 분산 학습 방식입니다.
[^a-smo]: **SMO**(Service Management and Orchestration): O-RAN에서 네트워크 기능의 배포·설정·수명주기를 총괄하는 서비스 관리·오케스트레이션 프레임워크로, Non-RT RIC을 내부에 포함합니다.
[^a-ric]: **RIC**(RAN Intelligent Controller): RAN을 지능적으로 제어·최적화하는 O-RAN의 핵심 컨트롤러로, Near-RT RIC(10ms~1s)과 Non-RT RIC(1s 이상)으로 나뉩니다.
[^a-aibom]: **AIBOM**(AI Bill of Materials): 소프트웨어 자재명세서(SBOM) 개념을 AI로 확장한 것으로, 모델·학습 데이터·의존 라이브러리 등 AI 시스템 구성요소의 출처와 이력을 기록한 명세입니다.
[^a-du]: **DU**(Distributed Unit): 기지국에서 실시간성이 요구되는 하위 프로토콜 계층(상위 PHY·MAC·RLC)을 처리하는 분산 장치입니다. O-RAN 규격을 따르는 DU를 O-DU라 부릅니다.
[^a-cu]: **CU**(Central Unit): 기지국의 상위 프로토콜 계층(PDCP·RRC 등)을 처리하는 중앙 장치로, 제어평면의 CU-CP(CU-Control Plane)와 사용자평면의 CU-UP(CU-User Plane)으로 나뉩니다.
[^a-ue]: **UE**(User Equipment): 스마트폰 등 이동통신망에 접속하는 사용자 단말을 가리키는 3GPP 표준 용어입니다.
[^a-fcaps]: **FCAPS**(Fault, Configuration, Accounting, Performance, Security): 장애·구성·과금·성능·보안의 다섯 영역으로 네트워크 관리 기능을 분류하는 고전적인 관리 모델입니다.
[^a-ran]: **RAN**(Radio Access Network): 무선 접속망. 단말과 코어망 사이에서 무선 구간의 연결을 담당하는 기지국 중심의 네트워크 영역입니다.
[^a-e2sm]: **E2SM**(E2 Service Model): E2 인터페이스 위에서 RIC과 E2 노드가 주고받는 기능별 서비스 정의로, KPM 수집용 E2SM-KPM 등이 규격화되어 있습니다.
[^a-etri]: **ETRI**(Electronics and Telecommunications Research Institute): 한국전자통신연구원. 정보통신 분야를 연구하는 대한민국의 정부출연 연구기관입니다.
[^a-dos]: **DoS**(Denial of Service): 서비스 거부 공격. 자원을 고갈시키거나 시스템을 마비시켜 정상 사용자가 서비스를 이용하지 못하게 하는 공격입니다.
[^a-phy]: **PHY**(Physical Layer): 물리계층. 무선 신호의 변조·부호화 등 실제 비트 전송을 담당하는 프로토콜 스택의 최하위 계층입니다.
[^a-mac]: **MAC**(Medium Access Control): 매체 접근 제어 계층. 무선 자원의 스케줄링과 데이터 다중화·재전송을 담당합니다.
[^a-sinr]: **SINR**(Signal-to-Interference-plus-Noise Ratio): 신호 대 간섭·잡음비. 수신 신호의 품질을 나타내는 대표적인 무선 지표입니다.
[^a-cqi]: **CQI**(Channel Quality Indicator): 채널 품질 지표. 단말이 측정하여 기지국에 보고하며, 변조·부호화 방식 선택의 근거가 됩니다.
[^a-ddos]: **DDoS**(Distributed Denial of Service): 분산 서비스 거부 공격. 다수의 분산된 소스에서 동시에 트래픽을 발생시켜 표적을 마비시키는 DoS 공격입니다.
[^a-rlc]: **RLC**(Radio Link Control): 무선 링크 제어 계층. 데이터의 분할·재조립과 재전송 기반 오류 복구를 담당합니다.
[^a-pdcp]: **PDCP**(Packet Data Convergence Protocol): 패킷 데이터 수렴 프로토콜 계층. 헤더 압축과 암호화·무결성 보호를 담당합니다.
[^a-kpi]: **KPI**(Key Performance Indicator): 핵심 성능 지표. 시스템·서비스의 상태를 정량적으로 평가하기 위한 대표 지표입니다.
[^a-oransc]: **O-RAN SC**(O-RAN Software Community): O-RAN Alliance와 Linux Foundation이 함께 운영하는 O-RAN 참조 구현 오픈소스 커뮤니티입니다.
[^a-rsrp]: **RSRP**(Reference Signal Received Power): 기준 신호 수신 전력. 단말이 측정하는 셀 신호 세기의 대표 지표입니다.
[^a-rsrq]: **RSRQ**(Reference Signal Received Quality): 기준 신호 수신 품질. RSRP를 전체 수신 전력 대비로 정규화한 신호 품질 지표입니다.
[^a-rssi]: **RSSI**(Received Signal Strength Indicator): 수신 신호 세기 지표이며, NR-RSSI는 5G NR 대역에서 측정한 RSSI를 뜻합니다.
[^a-prb]: **PRB**(Physical Resource Block): 물리 자원 블록. 주파수·시간 격자에서 무선 자원을 할당하는 기본 단위입니다.
[^a-dmrs]: **DMRS**(Demodulation Reference Signal): 복조 기준 신호. 수신기가 채널을 추정하여 데이터를 복조할 수 있도록 함께 전송되는 기준 신호입니다.
[^a-fh]: **FH**(Fronthaul): 프론트홀. RU와 DU 사이를 연결하는 전송 구간으로, O-RAN에서는 개방형 규격인 O-FH(Open Fronthaul)를 사용합니다.
[^a-dl]: **DL**(Downlink): 하향링크. 기지국에서 단말 방향으로의 전송을 뜻합니다.
[^a-tcp]: **TCP**(Transmission Control Protocol): 신뢰성 있는 연결형 데이터 전송을 제공하는 인터넷 표준 전송계층 프로토콜입니다.
[^a-snr]: **SNR**(Signal-to-Noise Ratio): 신호 대 잡음비. 잡음 대비 신호 세기를 나타내는 기본적인 품질 지표입니다.
[^a-rrc]: **RRC**(Radio Resource Control): 무선 자원 제어 계층. 연결 설정·핸드오버 등 단말과 망 사이의 제어 시그널링을 담당합니다.
[^a-rbs]: **RBS**(Rogue Base Station): 정상 기지국을 가장하여 단말을 유인·공격하는 불법(악성) 기지국입니다.
[^a-onf]: **ONF**(Open Networking Foundation): SDN 등 개방형 네트워킹 기술의 표준화와 오픈소스 개발을 주도해 온 비영리 컨소시엄입니다.
[^a-gnb]: **gNB**(next generation NodeB): 5G NR 기지국을 가리키는 3GPP 표준 명칭입니다.
[^a-pets]: **PETs**(Privacy-Enhancing Technologies): 프라이버시 강화 기술. 차분 프라이버시, 동형암호, 신뢰 실행 환경 등 데이터를 활용하면서도 개인정보 노출을 최소화하는 기술군의 총칭입니다.
[^a-llm]: **LLM**(Large Language Model): 대규모 언어 모델. 방대한 텍스트로 사전학습되어 언어 이해·생성을 수행하는 초대형 신경망 모델입니다.
[^a-csl]: **CSL**(Context Sharing Layer): 컨텍스트 공유 계층. 분산된 dApp/xApp들이 다중 소스 텔레메트리를 동기화하여 일관된 컨텍스트로 제어 결정을 내리도록 하는 제안 아키텍처입니다.
[^a-sdn]: **SDN**(Software-Defined Networking): 제어평면과 데이터평면을 분리하고 소프트웨어 컨트롤러로 네트워크를 중앙에서 제어하는 아키텍처입니다.
[^a-nfv]: **NFV**(Network Functions Virtualization): 전용 하드웨어 장비로 구현되던 네트워크 기능을 범용 서버 위의 소프트웨어로 가상화하는 기술입니다.
[^a-rat]: **RAT**(Radio Access Technology): 무선 접속 기술. WiFi, LTE, 5G NR처럼 단말이 망에 접속할 때 사용하는 무선 기술 방식을 뜻합니다.
[^a-nr]: **NR**(New Radio): 3GPP가 정의한 5G 무선 접속 기술 규격입니다.
[^a-iot]: **IoT**(Internet of Things): 사물인터넷. 각종 사물이 네트워크에 연결되어 데이터를 주고받는 기술과 환경을 뜻합니다.
[^a-embb]: **eMBB**(enhanced Mobile Broadband): 초광대역 이동통신. 대용량·고속 전송에 초점을 둔 5G의 대표 서비스 유형입니다.
[^a-urllc]: **URLLC**(Ultra-Reliable Low-Latency Communication): 초고신뢰·저지연 통신. 자율주행·원격제어처럼 신뢰성과 지연에 민감한 5G 서비스 유형입니다.
[^a-sa]: **SA**(Standalone): 코어망까지 5G 장비로만 구성하는 5G 단독모드 구축 방식입니다.
[^a-nsa]: **NSA**(Non-Standalone): LTE 코어·기지국 인프라에 5G NR 기지국을 결합하여 운용하는 비단독모드 구축 방식입니다.
[^a-lte]: **LTE**(Long Term Evolution): 3GPP가 표준화한 4세대 이동통신 기술입니다.
[^a-ca]: **CA**(Certificate Authority): 인증기관. 공개키 인증서를 발급·관리하여 통신 주체의 신원을 보증하는 신뢰 기관입니다.
[^a-3gpp]: **3GPP**(3rd Generation Partnership Project): LTE·5G 등 이동통신 표준을 제정하는 국제 표준화 협력 기구입니다.
[^a-ssh]: **SSH**(Secure Shell): 원격 시스템에 암호화된 채널로 안전하게 접속·관리하기 위한 프로토콜로, SSHv2는 그 두 번째 버전입니다.
[^a-tls]: **TLS**(Transport Layer Security): 전송계층에서 통신의 암호화·인증·무결성을 제공하는 표준 보안 프로토콜입니다.
[^a-dtls]: **DTLS**(Datagram Transport Layer Security): UDP 등 데이터그램 전송 위에서 TLS 수준의 보안을 제공하는 프로토콜입니다.
[^a-ipsec]: **IPsec**(Internet Protocol Security): IP 계층에서 패킷 단위의 암호화와 인증을 제공하는 보안 프로토콜 집합입니다.
[^a-macsec]: **MACsec**(Media Access Control Security): 이더넷 링크(2계층)에서 프레임 단위 암호화·무결성을 제공하는 IEEE 802.1AE 표준입니다.
[^a-mlb]: **MLB**(Mobility Load Balancing): 셀 간 부하를 분산하도록 핸드오버 파라미터를 조정하는 SON 기능입니다.
[^a-mro]: **MRO**(Mobility Robustness Optimization): 핸드오버 실패를 줄이도록 파라미터를 최적화하는 SON 기능으로, MLB와 동일 파라미터를 반대 방향으로 조정하며 상충할 수 있습니다.
[^a-son]: **SON**(Self-Organizing Network): 자가조직화 네트워크. 네트워크가 스스로 설정·최적화·복구를 수행하도록 하는 자동화 기능군입니다.
[^a-drl]: **DRL**(Deep Reinforcement Learning): 심층 강화학습. 심층 신경망을 함수 근사에 사용하는 강화학습 기법입니다.
[^a-ru]: **RU**(Radio Unit): 무선 신호의 송수신과 하위 물리계층 처리를 담당하는 무선 장치로, O-RAN 규격을 따르는 RU를 O-RU라 부릅니다.

## References

[^r2]: M. Polese, N. Mohamadi, S. D'Oro, L. Bonati, and T. Melodia, "Beyond connectivity: An open architecture for AI-RAN convergence in 6G," *arXiv preprint* arXiv:2507.06911v2, Dec. 2025.
[^r4]: S. Salmi, M. A. Ouameur, M. Bagaa, G. C. Alexandropoulos, A. Tahenni, D. Massicotte, and A. Ksentini, "AI-native O-RAN architectures for 6G: Towards real-time adaptation, conflict resolution, and efficient resource management," *TechRxiv preprint*, Sep. 2025.
[^r5]: C. Benzaïd, T. Taleb, R. Cavalcanti, and A. Sánchez, "AI in Open RAN security: Enabler and threat — A comprehensive survey and future perspectives," *TechRxiv preprint*, Dec. 2025.
[^r6]: S. Soltani, A. Amanloo, M. Shojafar, and R. Tafazolli, "Intelligent control in 6G open RAN: Security risk or opportunity?" *IEEE Open Journal of the Communications Society*, 2025.
[^r7]: N. M. Yungaicela-Naula, V. Sharma, and S. Scott-Hayward, "Misconfiguration in O-RAN: Analysis of the impact of AI/ML," *Computer Networks*, p. 110455, 2024.
[^r11]: A. Scalingi, S. D'Oro, F. Restuccia, T. Melodia, and D. Giustiniano, "Det-RAN: Data-driven cross-layer real-time attack detection in 5G open RANs," in *Proc. IEEE INFOCOM*, 2024, pp. 41–50.
[^r12]: C. Sun, U. Pawar, M. Khoja, X. Foukas, M. K. Marina, and B. Radunovic, "SpotLight: Accurate, explainable and efficient anomaly detection for open RAN," in *Proc. ACM MobiCom*, 2024, pp. 923–937.
[^r13]: S. Chatzimiltis, M. Shojafar, M. B. Mashhadi, and R. Tafazolli, "Interpretable anomaly-based DDoS detection in AI-RAN with XAI and LLMs," *arXiv preprint* arXiv:2507.21193, Jul. 2025.
[^r25]: S. Ben Khalifa, R. Taheri, and Z. Pooranian, "Lightweight intrusion detection baselines for Open RAN xApps," in *Proc. IEEE ICC Workshops*, 2026.
[^r26]: B. M. Xavier, M. Dzaferagic, D. Collins, G. Comarela, M. Martinello, and M. Ruffini, "Machine learning-based early attack detection using Open RAN intelligent controller," in *Proc. IEEE ICC*, 2023.
[^r28]: 권동승, 나지현, "O-RAN에서 6G RAN 연구 방향," *전자통신동향분석*, vol. 40, no. 5, pp. 101–112, Oct. 2025.
