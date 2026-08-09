---
title: "[6G AI-RAN] 11. 컨테이너 기반 에뮬레이션, 테스트베드 및 벤치마크"
date: 2026-07-30 10:50:00 +0900
categories:
  - 2.미래기술
  - 6G AI RAN
  - Part IV 검증·미래
tags:
  - Testbed
  - Kubernetes
  - O-RAN-SC
  - USRP
  - Dataset
  - Benchmark
  - Digital-Twin
math: true
mermaid: false
---

# 컨테이너 기반 에뮬레이션, 테스트베드 및 벤치마크

## 들어가며 — 재현 가능성이 곧 신뢰성이다

Part II~III에서 본 수치들 — 정확도 0%, F1 0.978, 2 ms, 2,268 ms — 은 모두 **특정 테스트베드에서 특정 조건으로** 측정된 값입니다. 그 조건을 모르면 비교할 수 없고, 재현할 수 없으면 신뢰할 수 없습니다.

이 장은 **"어디서, 무엇으로, 어떻게 측정했는가"** 를 정리합니다.

1. 테스트베드 스택 — 컨테이너부터 무선까지
2. 실제 테스트베드 사례 4종
3. 데이터·모델 자산 (DeepMIMO, LibIQ, 디지털 트윈)
4. 보안 데이터셋
5. AI-RAN 공존 실측 — Polese 등의 프로파일링
6. 벤치마크 리포팅 체크리스트
7. 남은 공백 — ML 취약점 스코어링과 재현성

---

## 1. 테스트베드 스택

O-RAN(Open Radio Access Network)[^a-oran] 보안 실험 환경은 대략 4개 계층으로 구성됩니다.

| 계층 | 구성 요소 | 예시 |
|---|---|---|
| **① 무선/물리** | SDR[^a-sdr] 하드웨어, 채널 에뮬레이터, 재머 | **USRP**[^a-usrp](RAN[^a-ran]·UE[^a-ue]·jammer), 대형 채널 에뮬레이터[^r8], [^r11] |
| **② RAN 스택** | O-RU[^a-oru]/O-DU[^a-odu]/O-CU[^a-ocu] 구현 | 오픈소스 RAN 스택, 상용 vRAN[^a-vran] |
| **③ RIC 플랫폼** | Near-RT RIC[^a-ric] + xApp 런타임 | **O-RAN SC(OSC)[^a-osc] near-RT RIC — Kubernetes 클러스터 상의 마이크로서비스**[^r6], [^r9] |
| **④ 오케스트레이션** | 컨테이너 플랫폼, 테넌트 격리 | **Kubernetes / Red Hat OpenShift, 네임스페이스 기반 워크로드 격리**[^r2] |

### 1.1 컨테이너 기반 RIC — O-RAN SC

> O-RAN Alliance는 **O-RAN Software Community(OSC)** 를 통해 **Kubernetes 클러스터 상의 마이크로서비스로 구성된 near-RT RIC**를 배포하여 O-RAN 솔루션을 프로토타이핑한다.[^r6]
{: .prompt-info }

또한 ONF(Open Networking Foundation)[^a-onf]는 O-RAN 아키텍처의 예시 플랫폼으로 **SD-RAN(Software-Defined RAN)[^a-sd-ran] 오픈소스 프로젝트**를 정의했습니다. 3GPP(3rd Generation Partnership Project)[^a-3gpp] 표준과 O-RAN 참조 설계 규격에 부합하며, SDN(Software-Defined Networking)[^a-sdn] 원칙에 맞춰 설계되어 **SDN 컨트롤러와 P4[^a-p4] 프로그래머블 스위치**의 유연성·확장성을 활용합니다. 이 구현에서 **near-RT RIC는 마이크로서비스 기반 ONOS(μONOS)[^a-onos] 컨트롤러로 구현된 오픈소스 SD-RAN 컨트롤러로 동작**합니다[^r6]. 학계에는 **FlexRIC** 같은 다른 RIC 구현도 존재합니다[^r6].

| 구현 | 특징 |
|---|---|
| **O-RAN SC (OSC) near-RT RIC** | Kubernetes 클러스터 상 마이크로서비스. 가장 널리 쓰이는 실험 플랫폼 |
| **ONF SD-RAN (μONOS)** | 마이크로서비스 기반 ONOS. SDN 원칙 정합, P4 스위치 활용 |
| **FlexRIC** | 학계 RIC 구현 |

### 1.2 컨테이너 격리 — 보안 관점

Ch2에서 본 AI(Artificial Intelligence)[^a-ai]-RAN Site의 오케스트레이션 계층이 여기 해당합니다.

> AI-RAN Site는 컴퓨트(GPU 등 AI 가속기), 스토리지, 네트워킹 자원을 갖추며, 이들은 **컨테이너 오케스트레이션 플랫폼(예: Kubernetes 또는 Red Hat OpenShift)** 으로 관리되고, 이 플랫폼은 **네임스페이스를 통한 테넌트 워크로드 격리**도 제공한다.[^r2]
{: .prompt-info }

| 격리 수단 | 막는 것 | 못 막는 것 |
|---|---|---|
| **네임스페이스** | 논리적 자원 분리, 이름 충돌 | **커널 공유** — 컨테이너 탈출 |
| 리소스 쿼터/limits | 자원 고갈(Ch6 §3.7) | 세밀한 GPU[^a-gpu] 공유 채널 |
| **RAN 워크로드 선점(preemption)**[^r2] | AI 학습의 지연 스파이크가 RAN을 훼손하는 것 | — |
| **TEE**[^a-tee](Ch9) | O-Cloud 내부 고권한 공격자 | 메모리·오버헤드 제약 |

> **실험 설계 시 주의**: 논문이 "격리했다"고 할 때 그것이 **네임스페이스 수준인지, 별도 노드인지, TEE인지**를 확인해야 합니다. Ch6 §1의 공격 원형 — 악성 xApp이 **같은 RIC의 같은 데이터베이스**에 접근 — 은 네임스페이스 격리와 무관하게 성립합니다(SDL(Shared Data Layer)[^a-sdl]이 공유 자산이므로).
{: .prompt-warning }

---

## 2. 실제 테스트베드 사례

### 2.1 적대적 xApp 실험 테스트베드 (Chiejina 등)

![O-RAN 테스트베드 — RAN·UE·재머 USRP와 랙 서버 (출처: Chiejina 등[^r8], Fig. 6)](/assets/img/posts/6g-ai-ran/sysadv-fig6.png)
_그림 11-1. **O-RAN 테스트베드**. 좌측 사진: **RAN, UE, 재머(jammer) USRP**. 우측 사진: 랙 서버(near-RT RIC와 xApp 호스팅). 출처: [^r8], Fig. 6._

| 항목 | 구성 |
|---|---|
| 무선 | USRP 3종 역할 — RAN, UE, **재머** |
| 컴퓨트 | 랙 서버 |
| 데이터 | **I/Q 샘플**[^a-iq](최근 10 ms = LTE[^a-lte]/5G 1 프레임) → 스펙트로그램 변환, 그리고 **KPM**[^a-kpm] |
| 측정 지표 | ML(Machine Learning)[^a-ml] 정확도 + **네트워크 성능(처리량, BLER[^a-bler])** + **O-RAN 타이밍** |

> **이 테스트베드의 가치**: 단순히 "모델 정확도가 떨어졌다"가 아니라 **처리량·BLER 같은 시스템 지표와 O-RAN 타이밍까지 측정**했습니다[^r8]. 보안 연구에서 **ML 지표와 네트워크 지표를 함께 보고하는 것**이 왜 중요한지 보여주는 모범 사례입니다.
{: .prompt-tip }

### 2.2 Rogue Cell / APATE 테스트베드 (Aizikovich 등)

![테스트베드 환경 — OSC near-RT RIC Kubernetes 클러스터와 무선 네트워크 시뮬레이터 (출처: Aizikovich 등[^r9], Fig. 6)](/assets/img/posts/6g-ai-ran/roguecell-fig6.png)
_그림 11-2. **테스트베드 환경**. 좌측: **OSC near-RT RIC Kubernetes 클러스터**. 우측: **무선 네트워크 시뮬레이터**. 출처: [^r9], Fig. 6._

| 항목 | 구성 |
|---|---|
| RIC | **OSC near-RT RIC (Kubernetes 클러스터)** |
| 무선 | **무선 네트워크 시뮬레이터** (실장비 대신) |
| xApp 파이프라인 | **KPIMON → InfluxDB → AD xApp → TS xApp → QP xApp** |
| 공개 | 저자들은 **오픈소스 테스트베드 환경을 공개**했습니다[^r9] |

![정상 시나리오와 MAS의 네트워크 토폴로지 (출처: Aizikovich 등[^r9], Fig. 8)](/assets/img/posts/6g-ai-ran/roguecell-fig8.png)
_그림 11-3. 정상 시나리오 대 MAS(악성 공격 시나리오)의 네트워크 토폴로지. BS1~BS6 기지국. 출처: [^r9], Fig. 8._

![공격에 따른 셀별 UE 분포 패턴 변화 (출처: Aizikovich 등[^r9], Fig. 7)](/assets/img/posts/6g-ai-ran/roguecell-fig7.png)
_그림 11-4. 시간에 따른 셀별 **UE 분포 패턴** 변화로 공격 영향을 시각화. 출처: [^r9], Fig. 7._

> **주목할 기여**: 이 연구는 **오픈소스 테스트베드를 공개**했습니다[^r9]. Ch10 §7.2에서 지적한 "ML 취약점 스코어링 부재" 문제를 완화하려면, 이런 **공개 테스트베드 위에서 공격·방어를 동일 조건으로 비교**하는 관행이 필요합니다.
{: .prompt-tip }

### 2.3 Det-RAN — 에뮬레이션에서 실제 프로토타입까지

Scalingi 등[^r11]의 2단계 검증이 모범적입니다.

| 단계 | 환경 | 측정 |
|---|---|---|
| **1단계: 광범위 에뮬레이션** | 에뮬레이션 환경 | **미관측 테스트 시나리오에서 85% 이상** 공격 예측 정확도 |
| **2단계: 실제 프로토타입** | **대형 채널 에뮬레이터를 갖춘 실물 프로토타입** | **실시간 성능과 비용** 평가 → **2 ms 저지연 실시간 제약 충족** |

![여러 설정에서의 추론 시간 (출처: Scalingi 등[^r11], Fig. 11)](/assets/img/posts/6g-ai-ran/detran-fig11.png)
_그림 11-5. **설정별 추론 시간** — Setup 4에서 Det-RAN이 실시간 요구를 충족합니다. 출처: [^r11], Fig. 11._

일반화 검증 방식도 참고할 만합니다.

![단일 시나리오 학습 후 다중 시나리오 테스트 정확도 (출처: Scalingi 등[^r11], Fig. 7)](/assets/img/posts/6g-ai-ran/detran-fig7.png)
_그림 11-6. **단일 시나리오로 학습 → 다중 시나리오 테스트**. 일반화 성능 평가 방법. 출처: [^r11], Fig. 7._

![다중 시나리오 학습 후 단일 시나리오 테스트 (출처: Scalingi 등[^r11], Fig. 8)](/assets/img/posts/6g-ai-ran/detran-fig8.png)
_그림 11-7. **다중 시나리오 학습 → 단일 시나리오 테스트**. 학습 다양성이 일반화를 개선함을 보입니다. 출처: [^r11], Fig. 8._

> **일반화 평가의 원칙**: 보안 모델은 **미관측 공격 시나리오**에서 평가해야 의미가 있습니다. 학습·테스트 시나리오를 교차 설계한 Det-RAN의 방식(그림 11-6·11-7)을 표준 관행으로 삼을 만합니다.
{: .prompt-tip }

### 2.4 SpotLight — 엔터프라이즈 실환경

Sun 등[^r12]은 시뮬레이터가 아닌 **실제 배치**에서 데이터를 얻었습니다.

| 항목 | 구성 |
|---|---|
| 환경 | **실내 오피스 빌딩의 엔터프라이즈 규모 5G Open RAN 배치** |
| 계측 | **RAN + 플랫폼 양쪽에 상세 계측 도입**, **600개 이상의 세밀한 KPI**[^a-kpi] |
| 샘플링 | KPI별 **100 ms**, **64 샘플 윈도우** |
| 아키텍처 | **far-edge 경량 필터(JVGAN) + 클라우드 정밀 분석** |
| 결과 | F1 **+13%**, 보고 KPI **2.3–4× 감소**, 대역폭 **4–7× 절감** |

![SpotLight 시스템 아키텍처 (출처: Sun 등[^r12], Fig. 3)](/assets/img/posts/6g-ai-ran/spotlight-fig3.png)
_그림 11-8. SpotLight의 엣지·클라우드 분산 아키텍처. 출처: [^r12], Fig. 3._

![프론트홀 처리량·스레드 런타임·DL TCP 처리량·SINR의 동시 변동 (출처: Sun 등[^r12], Fig. 2)](/assets/img/posts/6g-ai-ran/spotlight-fig2.png)
_그림 11-9. 실환경 계측 데이터 — 하나의 근본 원인이 여러 계층 지표에 동시에 나타납니다. 출처: [^r12], Fig. 2._

### 2.5 자율 컴플라이언스 — N8N 기반 워크플로 실증

![N8N으로 구현한 정적 컴플라이언스 워크플로 (출처: Chatzimiltis 등[^r14], Fig. 3)](/assets/img/posts/6g-ai-ran/agentic-fig3.png)
_그림 11-10. **N8N 워크플로 자동화 도구**로 구현한 정적 컴플라이언스 케이스 스터디. LLM 에이전트 파이프라인을 코드 없이 구성·재현할 수 있는 접근입니다. 출처: [^r14], Fig. 3._

---

## 3. 데이터·모델 자산

ETRI(Electronics and Telecommunications Research Institute)[^a-etri] 김민건 등[^r29]은 AI-RAN 연구를 뒷받침하는 대표 인프라로 **DeepMIMO, LibIQ, O-RAN 디지털 트윈 플랫폼**을 지목합니다.

### 3.1 DeepMIMO — 채널 데이터셋

| 특징[^r29] | 내용 |
|---|---|
| 생성 방식 | **레이 트레이싱 기반 시뮬레이션** — 실제 환경 기반 3차원 모델 위에서 수행, **현실감 있는 LOS[^a-los]·NLOS[^a-nlos] 채널** 생성 |
| 구성 가능성 | **BS[^a-bs]·UE 위치, 안테나 수, 빔 형성 방식** 등 파라미터를 사용자가 설정 → 다양한 시나리오 구성 |
| 다목적성 | **빔 선택·빔 추정·채널 예측·위치 추정** 등 여러 AI 문제를 **동일 데이터셋에서** 다룰 수 있어, 공통 기반 위에서 알고리즘 비교·평가 용이 |
| 용도 | mmWave[^a-mmwave]·대규모 MIMO[^a-mimo] 환경 AI 모델 연구의 **공용 기초 데이터셋**. 6G 빔포밍·슬라이스 설계에도 기대 |

### 3.2 LibIQ — 실시간 I/Q 데이터와 스펙트럼 인지

| 특징[^r29] | 내용 |
|---|---|
| 역할 | RAN에서 관측되는 **복소 I/Q 샘플을 수집·가공**해 실시간 스펙트럼 분류·간섭 탐지 모델 학습에 활용 |
| 데이터 다양성 | **다양한 변조 방식, 간섭 패턴, SNR**[^a-snr]에서 I/Q 시퀀스 수집, CNN[^a-cnn] 등 딥러닝 학습용으로 구조화 |
| **배치 방식** | 학습된 모델을 **dApp 형태로 O-RU/O-DU 근처에 배치** → 스펙트럼 상태 신속 분류, 간섭 의심 상황 빠른 감지 |
| 폐루프 | **스펙트럼 센싱 → AI 추론 → RAN 파라미터 조정**을 하나의 짧은 폐루프로 묶음. 동적 스펙트럼 공유·간섭 회피에 적합 |

> **보안 연구와의 접점**: Ch6 §1의 InterClass-Spec xApp이 사용한 것이 정확히 **I/Q → 스펙트로그램** 파이프라인입니다[^r8]. 즉 LibIQ류 자산은 **간섭 분류 모델의 적대적 강건성 연구를 위한 공용 기반**이 될 수 있습니다.
{: .prompt-tip }

### 3.3 O-RAN 디지털 트윈 플랫폼

Ch10 §7.4에서 본 **LLM(Large Language Model)[^a-llm] + 디지털 트윈 기반 가상 AI 레드팀**[^r5]의 실행 기반입니다. ETRI 정리[^r28]에서도 6G Native AI 아키텍처의 요소로 **DTN**(Digital Twin Network)[^a-dtn]이 제시되며, *"DT는 Native AI를 실행할 수 있는 인프라를 제공하며, AI 컴퓨팅과 처리는 시뮬레이션과 온라인에서도 DT 도메인으로 실행될 수 있다"* 고 서술됩니다.

| 용도 | 관련 장 |
|---|---|
| 제어 계획 사전 검증 (에이전트 액추에이터) | Ch10 §1, §4 |
| 적대적 시나리오 대량 생성 (가상 레드팀) | Ch10 §7.4 |
| 정책·모델 사전 검증 | Ch1 §4 |

---

## 4. 보안 데이터셋

| 데이터셋 | 용도 | 출처 |
|---|---|---|
| **NetSLab-5GORAN-IDD** | Open RAN xApp 침입탐지 벤치마크 — **12개 알고리즘의 정확도·지연 비교** | [^r25] |
| **5G-NIDD** (5G Network Intrusion Detection Dataset) | 네트워크 침입 탐지. SHAP[^a-shap] 기반 오염 FL[^a-fl] 클라이언트 판별 검증에 사용 | [^r5]가 인용 |
| Det-RAN 수집 데이터 | 크로스레이어(PHY[^a-phy] DMRS[^a-dmrs] + 상위계층) 공격 탐지 | [^r11] |
| SpotLight 실측 KPI | 엔터프라이즈 5G Open RAN의 600+ KPI 시계열 | [^r12] |
| InterClass 실험 데이터 | I/Q → 스펙트로그램, KPM (재머 유무) | [^r8] |
| 조기 탐지 실험 데이터 | VoIP[^a-voip], **DDoS Ripper, DoS Hulk, Slowloris**, Benign 클래스 | [^r26] |
| XAI[^a-xai]-DDoS[^a-ddos] KPM | 실제 5G 네트워크 KPM. **SYN Flood, ICMP[^a-icmp] Flood, UDP[^a-udp] Fragmentation, DNS[^a-dns] Flood, GTP-U[^a-gtp-u] Flood** | [^r13] |

![O-RAN 아키텍처와 IDS xApp의 위치 (출처: Ben Khalifa 등[^r25], Fig. 1)](/assets/img/posts/6g-ai-ran/lightids-fig1.png)
_그림 11-11. IDS[^a-ids] xApp을 Near-RT RIC에 호스팅하는 벤치마크 설정. 출처: [^r25], Fig. 1._

![추론 지연 비교 (로그 스케일) (출처: Ben Khalifa 등[^r25], Fig. 2)](/assets/img/posts/6g-ai-ran/lightids-fig2.png)
_그림 11-12. **12개 모델의 추론 지연 비교 (로그 스케일)**. 이 그림이 이 장의 핵심 메시지입니다 — **정확도만 보고하는 벤치마크는 불완전합니다.** 출처: [^r25], Fig. 2._

---

## 5. AI-RAN 공존 실측 — 프로파일링

Ch2에서 개념으로 본 **AI and RAN 공존**을 Polese 등[^r2]이 실측했습니다. 이것이 Ch6 §3.7 자원 고갈 위협의 정량적 근거가 됩니다.

![RAN 단독(실선) 대 AI-RAN 공존 시의 처리량·CRC 오류율 중앙값 (출처: Polese 등[^r2], Fig. 5)](/assets/img/posts/6g-ai-ran/beyondconn-fig5.png)
_그림 11-13. **RAN 단독(solid bars) 대 AI-RAN 공존** 시의 처리량과 CRC[^a-crc] 오류율 중앙값. **재전송을 포함한 MAC[^a-mac] 계층** 기준입니다. 출처: [^r2], Fig. 5._

![AI-RAN 사이트에서의 워크로드 배포 지연 (출처: Polese 등[^r2], Fig. 6)](/assets/img/posts/6g-ai-ran/beyondconn-fig6.png)
_그림 11-14. **LLM 모델·파라미터 규모별 순차 배포 지연** 프로파일링. 출처: [^r2], Fig. 6._

| 측정 항목 | 왜 보안 연구에 중요한가 |
|---|---|
| RAN 단독 vs 공존 시 **처리량·CRC 오류율** | **AI 워크로드가 RAN 성능에 주는 영향의 기준선(baseline)** — 이 기준선 없이는 "공격에 의한 저하"와 "정상 공존에 의한 저하"를 구분할 수 없습니다 |
| LLM 규모별 **배포 지연** | 모델 교체·롤백 기반 방어(MTD[^a-mtd], Ch7)의 **물리적 하한** |

> **실험 설계 함의**: AI-RAN 보안 실험은 **반드시 "정상 공존" 기준선을 함께 측정**해야 합니다. 그렇지 않으면 공격 효과를 과대평가합니다.
{: .prompt-warning }

---

## 6. 벤치마크 리포팅 체크리스트

앞선 사례들에서 도출한, **AI-RAN 보안 실험을 보고할 때 포함해야 할 항목**입니다.

### 6.1 환경

- [ ] RIC 구현 (**OSC / μONOS(SD-RAN) / FlexRIC** 등)과 버전
- [ ] 컨테이너 플랫폼 및 **격리 수준** (네임스페이스 / 별도 노드 / TEE)
- [ ] 무선 계층 (**실장비 USRP / 채널 에뮬레이터 / 시뮬레이터**)
- [ ] xApp/rApp/dApp 배치 위치 및 **제어 루프 시간 척도**
- [ ] AI 가속기 유무 및 **AI 워크로드 공존 여부**

### 6.2 위협 모델 (Ch4 §7 템플릿 준용)

- [ ] 공격자 **지식 수준** (White / Gray / Black / **Myopic**)
- [ ] 공격자 **능력** (C1~C8) 및 **한계** (L1~L3)
- [ ] **영향 시점** (Causative / Exploratory)
- [ ] **공격 전략** (S1~S7)
- [ ] 적대적 공격의 경우 **$\epsilon$ 범위**와 노름($L_\infty$ 등)

### 6.3 성능 지표

| 범주 | 반드시 보고 |
|---|---|
| **ML 지표** | Accuracy, **Precision·Recall·F1**, **FPR[^a-fpr]·FNR[^a-fnr]** |
| **지연 지표** | **추론 지연**(모델별), 데이터 수집 지연, 종단간 폐루프 지연 |
| **시스템 지표** | **처리량, BLER/CRC 오류율**, SLA[^a-sla] 위반율 |
| **자원 지표** | CPU[^a-cpu]/GPU 사용률, 메모리, **대역폭** |
| **일반화** | **미관측 시나리오** 성능 (학습·테스트 시나리오 교차 설계) |
| **기준선** | **무공격 baseline** + **정상 AI-RAN 공존 baseline** |

> **FPR을 빼놓지 마십시오.** Ch10 §6.1의 **가드레일 DoS(Denial of Service)[^a-dos]**[^r19]와 Ch8 §7.2의 자율 완화 오탐 문제 때문에, **FPR은 보안 시스템에서 정확도보다 운영상 중요할 수 있습니다.** Chatzimiltis 등[^r13]이 FPR·FNR을 별도 그림으로 보고한 것(Ch8 그림 8-24)은 좋은 관행입니다.
{: .prompt-danger }

### 6.4 재현 가능성

- [ ] 데이터셋 공개 여부 및 접근 방법
- [ ] **테스트베드 코드·구성 공개** (예: Aizikovich 등[^r9]의 오픈소스 테스트베드)
- [ ] 사용한 LLM/모델의 **정확한 버전** (Ch10 §6.2 캘리브레이션 전이성 문제)
- [ ] 난수 시드, 반복 횟수, 신뢰구간

---

## 7. 남은 공백

### 7.1 ML 취약점 스코어링 부재

Ch10 §7.2에서 인용한 공백을 다시 강조합니다.

> **ML 취약점을 성능 영향과 적대적 공격 촉진 위험 양쪽을 반영해 점수화하는 방법론이 아직 고안되지 않았다.** 정확한 취약점 스코어링은 취약점 개선 우선순위 결정에 결정적이다.[^r5]
{: .prompt-warning }

| 소프트웨어 세계 | AI/ML 세계 |
|---|---|
| CVE[^a-cve] / CWE[^a-cwe] / CAPEC[^a-capec] | (부분적으로만 매핑 — BERT[^a-bert] 기반 자동 매핑 연구[^r5]) |
| **CVSS 점수**[^a-cvss] | **없음** |
| SBOM[^a-sbom] | **AIBOM**[^a-aibom] (Ch7, 초기 단계) |
| 퍼징 프레임워크 | **API[^a-api] 수준 퍼징이 모델 수준보다 효과적**이라고 알려짐[^r5] |

### 7.2 테스팅 방법론 공백

Benzaïd 등[^r5]이 지목한 필요 사항:

| 필요 | 내용 |
|---|---|
| **의미 보존 테스트 생성** | O-RAN의 ML 기반 애플리케이션에 대해 **유효하고 의미를 보존하는(semantically preserving)** 테스트를 생성하는 방법 — 기존 방법 적응 또는 신규 개발 |
| **지속적 테스팅 + AI 지원 취약점 탐지** | ML 취약점의 끊임없는 진화·양·복잡도 대응 |
| **모델 헬스 메트릭** | *"O-RAN 애플리케이션에 맞춘 적절한 **모델 헬스 메트릭(model health metrics)** 을 정의할 필요가 있다"*[^r5] |
| **보안·신뢰성 모니터링** | 기존 연구는 **성능 저하 관점의 모델 품질 보증에 집중**했고, **모델의 보안·신뢰성 모니터링은 미탐구** |

### 7.3 자동화·확장성

> **O-RAN에서 예상되는 다양하고 증가하는 수의 ML 모델**을 효과적으로 다루려면 **모델 건강 상태를 모니터링하는 자동화·확장 가능한 메커니즘**이 필수적이다.[^r5]
{: .prompt-info }

수백~수천 개의 xApp/rApp/dApp이 배치되는 6G에서, 수동 검증은 불가능합니다. 이것이 Ch10의 자율 컴플라이언스와 이 장의 벤치마크 자동화가 만나는 지점입니다.

---

## 8. 이 장의 요약

- 테스트베드 스택은 **무선/물리 → RAN 스택 → RIC 플랫폼 → 오케스트레이션** 4층이며, RIC 실험 플랫폼은 **OSC near-RT RIC(Kubernetes)**, **ONF SD-RAN(μONOS)**, **FlexRIC** 등이 있습니다[^r6].
- 컨테이너 격리는 **네임스페이스 → 쿼터 → RAN 선점 → TEE** 순으로 강해지지만, **SDL 같은 공유 자산은 격리로 막을 수 없습니다**.
- 사례: **USRP + 랙 서버**(적대적 xApp, 처리량·BLER·타이밍까지 측정)[^r8], **OSC RIC + 무선 시뮬레이터**(APATE, 오픈소스 공개)[^r9], **에뮬레이션 → 채널 에뮬레이터 프로토타입**(Det-RAN, 2 ms 검증)[^r11], **엔터프라이즈 실환경 5G Open RAN**(SpotLight, 600+ KPI)[^r12], **N8N 워크플로**(자율 컴플라이언스)[^r14].
- 데이터·모델 자산: **DeepMIMO**(레이트레이싱 mmWave/대규모 MIMO 채널, 다목적 벤치마크), **LibIQ**(실시간 I/Q → dApp 배치, 센싱-추론-조정 폐루프), **O-RAN 디지털 트윈 플랫폼**[^r29], [^r28].
- 보안 데이터셋: **NetSLab-5GORAN-IDD**, **5G-NIDD** 등.
- **AI-RAN 공존 프로파일링**(처리량·CRC 오류율, LLM 배포 지연)은 자원 고갈 위협 연구의 **필수 기준선**입니다[^r2].
- 벤치마크 리포팅은 **환경 + 위협모델 + ML/지연/시스템/자원 지표 + 일반화 + 두 개의 기준선 + 재현 정보**를 모두 포함해야 하며, **FPR을 절대 빼놓지 말아야** 합니다.
- 남은 공백: **ML 취약점 스코어링(CVSS 상응물) 부재**, **의미 보존 테스트 생성**, **모델 헬스 메트릭 정의**, **보안·신뢰성 모니터링 연구 부재**, **확장 가능한 자동 검증**[^r5].

### 확인 체크리스트

- [ ] OSC near-RT RIC, ONF SD-RAN(μONOS), FlexRIC를 구분할 수 있는가
- [ ] 네임스페이스 격리가 막을 수 없는 공격을 하나 들 수 있는가
- [ ] Det-RAN의 2단계 검증(에뮬레이션 → 실물 프로토타입) 방식의 장점을 설명할 수 있는가
- [ ] DeepMIMO와 LibIQ의 용도 차이를 말할 수 있는가
- [ ] "정상 AI-RAN 공존 기준선"이 왜 필요한지 설명할 수 있는가
- [ ] 자신의 실험을 §6 체크리스트로 점검할 수 있는가
- [ ] ML 취약점 스코어링 부재가 실무에서 무엇을 어렵게 하는지 설명할 수 있는가

**다음 장**: [12. 양자 암호(PQC), 표준화 및 미래 연구 과제](/posts/airan-12-pqc-standards/)

---

### 약어

[^a-oran]: **O-RAN**(Open Radio Access Network): 기지국 구성요소 간 인터페이스를 개방형 표준으로 규정해 다중 벤더 구성을 가능하게 하는 무선 접속망 아키텍처입니다.
[^a-sdr]: **SDR**(Software-Defined Radio): 변조·복조 등 무선 신호 처리를 전용 하드웨어 대신 소프트웨어로 구현하는 무선 기술입니다.
[^a-usrp]: **USRP**(Universal Software Radio Peripheral): SDR 실험에 널리 쓰이는 범용 소프트웨어 무선 하드웨어 플랫폼입니다.
[^a-ran]: **RAN**(Radio Access Network): 단말과 코어망 사이에서 무선 접속을 담당하는 무선 접속망 구간입니다.
[^a-ue]: **UE**(User Equipment): 이동통신망에 접속하는 단말 장치를 가리키는 표준 용어입니다.
[^a-oru]: **O-RU**(O-RAN Radio Unit): O-RAN에서 무선 신호 송수신과 하위 물리계층 처리를 담당하는 무선 장치입니다.
[^a-odu]: **O-DU**(O-RAN Distributed Unit): O-RAN에서 실시간성이 높은 하위 프로토콜 계층 처리를 담당하는 분산 장치입니다.
[^a-ocu]: **O-CU**(O-RAN Central Unit): O-RAN에서 상위 프로토콜 계층 처리를 담당하는 중앙 장치입니다.
[^a-vran]: **vRAN**(virtualized RAN): RAN 기능을 범용 서버 위의 소프트웨어로 가상화해 구현한 형태입니다.
[^a-ric]: **RIC**(RAN Intelligent Controller): RAN을 지능적으로 제어·최적화하는 O-RAN의 컨트롤러로, 제어 주기에 따라 Near-RT(준실시간)와 Non-RT(비실시간)로 나뉩니다.
[^a-osc]: **OSC**(O-RAN Software Community): O-RAN 규격의 오픈소스 참조 구현을 개발하는 커뮤니티입니다.
[^a-onf]: **ONF**(Open Networking Foundation): 개방형 네트워킹 기술의 표준화와 오픈소스화를 추진하는 비영리 컨소시엄입니다.
[^a-sd-ran]: **SD-RAN**(Software-Defined RAN): ONF가 주도한, SDN 원칙에 따른 오픈소스 O-RAN 구현 프로젝트입니다.
[^a-3gpp]: **3GPP**(3rd Generation Partnership Project): 3G부터 5G·6G까지 이동통신 국제 표준을 제정하는 표준화 협력체입니다.
[^a-sdn]: **SDN**(Software-Defined Networking): 네트워크의 제어 평면을 데이터 평면에서 분리해 소프트웨어로 중앙 제어하는 네트워킹 패러다임입니다.
[^a-p4]: **P4**(Programming Protocol-independent Packet Processors): 스위치 등 패킷 처리 장치의 동작을 프로토콜 독립적으로 기술하는 프로그래밍 언어입니다.
[^a-onos]: **ONOS**(Open Network Operating System): 통신 사업자급 SDN 컨트롤러 오픈소스 프로젝트이며, μONOS는 그 마이크로서비스 기반 후속 구현입니다.
[^a-ai]: **AI**(Artificial Intelligence): 인간의 학습·추론 능력을 계산 시스템으로 구현하는 인공지능 기술입니다.
[^a-gpu]: **GPU**(Graphics Processing Unit): 대규모 병렬 연산에 특화된 프로세서로, AI 학습·추론 가속에 널리 사용됩니다.
[^a-tee]: **TEE**(Trusted Execution Environment): 프로세서 안에 격리된 실행 영역을 만들어 사용 중인 코드·데이터를 보호하는 신뢰 실행 환경입니다.
[^a-sdl]: **SDL**(Shared Data Layer): Near-RT RIC 내부에서 xApp들이 공유하는 데이터 저장 계층입니다.
[^a-iq]: **I/Q**(In-phase/Quadrature): 무선 신호를 동상(I)·직교(Q) 두 성분의 복소 샘플로 표현한 것으로, 신호 분석과 ML 입력에 쓰입니다.
[^a-lte]: **LTE**(Long Term Evolution): 4세대 이동통신 표준 기술입니다.
[^a-kpm]: **KPM**(Key Performance Measurement): E2 인터페이스를 통해 수집되는 RAN 핵심 성능 측정 지표(서비스 모델)입니다.
[^a-ml]: **ML**(Machine Learning): 데이터로부터 패턴을 학습해 예측·분류를 수행하는 기계학습 기술입니다.
[^a-bler]: **BLER**(Block Error Rate): 전송 블록 단위의 오류율로, 무선 링크 품질을 나타내는 지표입니다.
[^a-kpi]: **KPI**(Key Performance Indicator): 시스템·네트워크의 성능을 정량적으로 나타내는 핵심 성과 지표입니다.
[^a-etri]: **ETRI**(Electronics and Telecommunications Research Institute): 한국전자통신연구원. 정보통신 분야의 국책 연구기관입니다.
[^a-los]: **LOS**(Line of Sight): 송신기와 수신기 사이에 장애물 없는 직접 경로가 존재하는 가시선 전파 환경입니다.
[^a-nlos]: **NLOS**(Non-Line of Sight): 직접 경로 없이 반사·회절 경로로만 신호가 도달하는 비가시선 전파 환경입니다.
[^a-bs]: **BS**(Base Station): 셀 내 단말과 무선으로 통신하는 기지국입니다.
[^a-mmwave]: **mmWave**(millimeter Wave): 대략 24 GHz 이상의 밀리미터파 대역으로, 5G·6G의 대용량 전송에 활용됩니다.
[^a-mimo]: **MIMO**(Multiple-Input Multiple-Output): 다수의 송·수신 안테나로 용량과 신뢰성을 높이는 다중 안테나 기술입니다.
[^a-snr]: **SNR**(Signal-to-Noise Ratio): 신호 대 잡음 전력의 비율로, 수신 품질을 나타내는 기본 지표입니다.
[^a-cnn]: **CNN**(Convolutional Neural Network): 합성곱 연산으로 공간적 특징을 추출하는 딥러닝 신경망 구조입니다.
[^a-llm]: **LLM**(Large Language Model): 방대한 텍스트로 학습되어 자연어 이해·생성 능력을 제공하는 대규모 언어 모델입니다.
[^a-dtn]: **DTN**(Digital Twin Network): 실제 네트워크를 가상으로 복제해 시뮬레이션·검증에 활용하는 디지털 트윈 네트워크입니다.
[^a-shap]: **SHAP**(SHapley Additive exPlanations): 게임이론의 섀플리 값에 기반해 각 입력 특징의 기여도를 정량화하는 설명가능 AI 기법입니다.
[^a-fl]: **FL**(Federated Learning): 원본 데이터를 중앙에 모으지 않고 각 참여자가 로컬 학습 결과만 공유하는 연합학습 기법입니다.
[^a-phy]: **PHY**(Physical Layer): 프로토콜 스택의 최하위 물리계층으로, 실제 무선 신호의 송수신을 담당합니다.
[^a-dmrs]: **DMRS**(Demodulation Reference Signal): 수신기가 채널을 추정해 신호를 복조하는 데 사용하는 복조 참조 신호입니다.
[^a-voip]: **VoIP**(Voice over IP): IP 네트워크를 통해 음성 통화를 전송하는 기술입니다.
[^a-xai]: **XAI**(eXplainable AI): AI 모델의 판단 근거를 사람이 이해할 수 있게 설명하는 설명가능 인공지능 기술입니다.
[^a-ddos]: **DDoS**(Distributed Denial of Service): 다수의 분산된 호스트가 동시에 트래픽을 보내 서비스를 마비시키는 분산 서비스 거부 공격입니다.
[^a-icmp]: **ICMP**(Internet Control Message Protocol): 네트워크 진단과 오류 보고에 쓰이는 인터넷 제어 메시지 프로토콜입니다.
[^a-udp]: **UDP**(User Datagram Protocol): 연결 설정 없이 데이터그램을 전송하는 비연결형 전송 계층 프로토콜입니다.
[^a-dns]: **DNS**(Domain Name System): 도메인 이름을 IP 주소로 변환하는 인터넷 이름 체계입니다.
[^a-gtp-u]: **GTP-U**(GPRS Tunnelling Protocol – User plane): 이동통신 코어망에서 사용자 평면 트래픽을 터널링하는 프로토콜입니다.
[^a-ids]: **IDS**(Intrusion Detection System): 트래픽과 행위를 분석해 침입을 탐지하는 침입탐지시스템입니다.
[^a-crc]: **CRC**(Cyclic Redundancy Check): 전송 데이터의 오류를 검출하는 순환 중복 검사 기법으로, 무선 링크 품질 지표로도 쓰입니다.
[^a-mac]: **MAC**(Medium Access Control): 무선 자원 접근과 스케줄링을 담당하는 매체 접근 제어 계층입니다.
[^a-mtd]: **MTD**(Moving Target Defense): 시스템 구성이나 모델을 주기적으로 변경해 공격자가 표적을 고정하지 못하게 하는 방어 전략입니다.
[^a-fpr]: **FPR**(False Positive Rate): 정상을 공격으로 잘못 판정한 비율(오탐률)입니다.
[^a-fnr]: **FNR**(False Negative Rate): 공격을 정상으로 잘못 판정한 비율(미탐률)입니다.
[^a-sla]: **SLA**(Service Level Agreement): 서비스 제공자가 보장해야 하는 성능·가용성 수준을 정의한 서비스 수준 협약입니다.
[^a-cpu]: **CPU**(Central Processing Unit): 범용 연산을 수행하는 중앙처리장치입니다.
[^a-dos]: **DoS**(Denial of Service): 자원을 고갈시키거나 처리를 방해해 정상적인 서비스 제공을 막는 서비스 거부 공격입니다.
[^a-cve]: **CVE**(Common Vulnerabilities and Exposures): 공개적으로 알려진 보안 취약점에 고유 식별자를 부여하는 목록 체계입니다.
[^a-cwe]: **CWE**(Common Weakness Enumeration): 소프트웨어·하드웨어의 보안 약점 유형을 분류한 표준 목록입니다.
[^a-capec]: **CAPEC**(Common Attack Pattern Enumeration and Classification): 알려진 공격 패턴을 열거·분류한 지식 베이스입니다.
[^a-bert]: **BERT**(Bidirectional Encoder Representations from Transformers): 트랜스포머 기반의 양방향 사전학습 언어 모델입니다.
[^a-cvss]: **CVSS**(Common Vulnerability Scoring System): 취약점의 심각도를 0~10 점수로 정량화하는 공통 취약점 평가 체계입니다.
[^a-sbom]: **SBOM**(Software Bill of Materials): 소프트웨어를 구성하는 컴포넌트와 의존성의 목록으로, 공급망 보안의 기초 자료입니다.
[^a-aibom]: **AIBOM**(AI Bill of Materials): AI 모델·데이터·의존성의 구성 내역을 기록한 목록으로, SBOM의 AI 확장입니다.
[^a-api]: **API**(Application Programming Interface): 소프트웨어 구성요소 간 기능 호출 규약을 정의한 인터페이스입니다.

## References

[^r2]: M. Polese, N. Mohamadi, S. D'Oro, L. Bonati, and T. Melodia, "Beyond connectivity: An open architecture for AI-RAN convergence in 6G," *arXiv preprint* arXiv:2507.06911v2, Dec. 2025.
[^r5]: C. Benzaïd, T. Taleb, R. Cavalcanti, and A. Sánchez, "AI in Open RAN security: Enabler and threat — A comprehensive survey and future perspectives," *TechRxiv preprint*, Dec. 2025.
[^r6]: S. Soltani, A. Amanloo, M. Shojafar, and R. Tafazolli, "Intelligent control in 6G open RAN: Security risk or opportunity?" *IEEE Open Journal of the Communications Society*, 2025.
[^r8]: A. Chiejina, B. Kim, K. Chowdhury, and V. K. Shah, "System-level analysis of adversarial attacks and defenses on intelligence in O-RAN based cellular networks," in *Proc. ACM WiSec*, 2024.
[^r9]: E. Aizikovich, D. Mimran, E. Grolman, Y. Elovici, and A. Shabtai, "Rogue cell: Adversarial attack and defense in untrusted O-RAN setup exploiting the traffic steering xApp," *arXiv preprint*, 2025.
[^r11]: A. Scalingi, S. D'Oro, F. Restuccia, T. Melodia, and D. Giustiniano, "Det-RAN: Data-driven cross-layer real-time attack detection in 5G open RANs," in *Proc. IEEE INFOCOM*, 2024, pp. 41–50.
[^r12]: C. Sun, U. Pawar, M. Khoja, X. Foukas, M. K. Marina, and B. Radunovic, "SpotLight: Accurate, explainable and efficient anomaly detection for open RAN," in *Proc. ACM MobiCom*, 2024, pp. 923–937.
[^r13]: S. Chatzimiltis, M. Shojafar, M. B. Mashhadi, and R. Tafazolli, "Interpretable anomaly-based DDoS detection in AI-RAN with XAI and LLMs," *arXiv preprint* arXiv:2507.21193, Jul. 2025.
[^r14]: S. Chatzimiltis, M. B. Mashhadi, M. Shojafar, M. Debbah, and R. Tafazolli, "Agentic AI for 6G: A new paradigm for autonomous RAN security compliance," *arXiv preprint* arXiv:2512.12400v2, Apr. 2026.
[^r19]: Q. Zhang, Z. Xiong, and Z. M. Mao, "Safeguard is a double-edged sword: Denial-of-service attack on large language models," *arXiv preprint* arXiv:2410.02916, 2024.
[^r25]: S. Ben Khalifa, R. Taheri, and Z. Pooranian, "Lightweight intrusion detection baselines for Open RAN xApps," in *Proc. IEEE ICC Workshops*, 2026.
[^r26]: B. M. Xavier, M. Dzaferagic, D. Collins, G. Comarela, M. Martinello, and M. Ruffini, "Machine learning-based early attack detection using Open RAN intelligent controller," in *Proc. IEEE ICC*, 2023.
[^r28]: 권동승, 나지현, "O-RAN에서 6G RAN 연구 방향," *전자통신동향분석*, vol. 40, no. 5, pp. 101–112, Oct. 2025.
[^r29]: 김민건, 김준우, 이훈, 배정숙, 김일규, "6G 무선접속망을 위한 AI/ML 기반 지능형 RAN 기술 동향," *전자통신동향분석*, vol. 41, no. 1, pp. 1–10, Feb. 2026.
