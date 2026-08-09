---
title: "[6G AI-RAN] 01. 6G 비전과 AI-Native RAN 패러다임"
date: 2026-07-30 09:10:00 +0900
categories:
  - 2.미래기술
  - 6G AI RAN
  - Part I 기초·아키텍처
tags:
  - 6G
  - AI-RAN
  - AI-Native
  - AI-RAN-Alliance
  - Digital-Twin
math: true
mermaid: false
---

# 6G 비전과 AI-Native RAN 패러다임

## 들어가며 — "AI를 붙인 RAN"과 "AI로 다시 설계한 RAN"의 차이

5G에서도 이미 AI(Artificial Intelligence)[^a-ai]/ML(Machine Learning)[^a-ml]은 쓰였습니다. 그러나 그것은 **Add-On**이었습니다. ETRI(Electronics and Telecommunications Research Institute)[^a-etri] 권동승·나지현[^r28]은 4G·5G의 AI 도입 방식을 이렇게 정리합니다.

> 4G·5G에서 AI는 **Add-On 기능**으로 구현되어, AI 모델 훈련·평가는 RAN 외부에서 수행되고 훈련된 모델이나 정책만 RAN에 전달되었다. 이 방식에는 두 가지 피할 수 없는 문제가 있다.
> ① 대규모의 실시간·세밀한 메트릭을 RAN 밖의 AI 엔터티로 전송해야 하므로 **TN·RAN 용량에 부담**이 생기고, 벤더 간 상호운용성 문제가 추가된다.
> ② **효과적인 추론은 효과적인 지표에 크게 좌우**되는데, 외부로 나간 지표는 이미 시간·해상도 손실을 겪는다.
{: .prompt-info }

**Native AI**(AI-native RAN)는 이 구조를 반대로 뒤집습니다. 대규모 메트릭을 RAN(Radio Access Network)[^a-ran] 밖으로 보내지 않고, **AI 훈련·추론에 필요한 메트릭을 RAN의 제어 플로우·데이터 플로우 안으로 수렴**시킵니다[^r28]. 즉 AI가 RAN의 부가기능이 아니라 **RAN 프로토콜 스택과 제어 루프의 구성요소**가 됩니다.

이 장에서 다루는 내용:

1. 6G가 AI-native를 요구하는 이유 (아키텍처 원칙)
2. RAN 아키텍처 진화의 맥락 — D-RAN(Distributed RAN)[^a-d-ran] → C-RAN(Centralized RAN)[^a-c-ran] → vRAN(virtualized RAN)[^a-vran]/O-RAN(Open Radio Access Network)[^a-o-ran] → AI-RAN(AI Radio Access Network)[^a-ai-ran]
3. **AI-RAN Alliance의 세 기둥** — AI for RAN / AI and RAN / AI on RAN
4. AI-RAN 핵심 인에이블러 — 디지털 트윈, IRS(Intelligent Reflecting Surface)[^a-irs], GenAI, 블록체인
5. 6G AI-RAN 유즈케이스와, 각 유즈케이스가 만드는 보안 요구

---

## 1. 6G 아키텍처 원칙과 AI의 위치

O-RAN Alliance의 6G 작업반 **nGRG**(next Generation Research Group)[^a-ngrg]는 2023년 설문을 통해 6G RAN 연구 방향을 조사했습니다[^r28]. 관심 아키텍처 영역과 우선순위는 다음과 같았습니다.

| 구분 | nGRG 설문 결과 |
|---|---|
| **6G 아키텍처 관심 영역** | TN[^a-tn], CN[^a-cn], RAN, SMO[^a-smo], **DT(Digital Twin)[^a-dt]**, **AI**, **보안과 프라이버시**, SBA[^a-sba], 에너지 효율성 |
| **유즈케이스 우선순위** | ① 네트워크 **모든 부분**에 AI/ML 적용 → ② 실시간 분석 → ③ 소프트웨어화·분리 → ④ B5G 엣지 중심 → ⑤ **프라이버시와 보안** → ⑥ 자동화/오토노믹 |
| **O-RAN 아키텍처 업무 영역** | 네트워크 자동화, 공통 SMO, **AI/ML Native 아키텍처**, 새로운 기능·인터페이스 |
| **6G RAN 3대 연구 방향** | **Native AI RAN**, Cloud-friendly RAN, Service-based RAN |

> 주목할 점: 설문 응답에서 **보안과 프라이버시가 유즈케이스 우선순위 5위**로 이미 들어와 있고, "AI/ML은 시스템 성능뿐 아니라 관리·오케스트레이션에서도 전체 시스템을 효율화하는 핵심 기술"로 규정됩니다. 또한 **6G 네트워크 복원력(resilience)** 항목에서 "AI/ML이 여러 네트워크 KPI를 모니터링하고 네트워크 설정을 조정해 복원력을 향상시켜야 한다"고 명시합니다[^r28]. 즉 **AI는 6G에서 보안·복원력의 구현 수단으로 표준화 논의에 포함되어 있습니다.**
{: .prompt-tip }

### 1.1 Native AI 아키텍처가 갖춰야 할 세 가지 기술 특징

ETRI 정리[^r28]에 따르면 O-RAN의 Native AI 아키텍처는 다음을 만족해야 합니다.

| # | 요구 특징 | 의미 | 보안 함의 (Ch4~Ch6 연결) |
|---|---|---|---|
| 1 | **계층화된 분산 AI 배치** | AI 기능이 중앙집중되지 않고 O-RAN 논리 노드에 유연하게 분산 배치 | 신뢰 경계가 다수로 쪼개짐 → 노드별 검증 필요 |
| 2 | **Cross-Domain AI 협업** | RAN-UE[^a-ue] 간, RAN-CN 간 등 도메인을 넘는 AI 작업 공동 수행 | 도메인 간 데이터 이동 → 프라이버시·모델 탈취 위험 (Ch9) |
| 3 | **AI 서비스 품질 보장** | AI가 UE·네트워크 요소에 제공되는 **서비스**이므로 QoS[^a-qos] 보장 대상 | 자원 고갈 공격(EDoS[^a-edos])이 곧 SLA[^a-sla] 위반 (Ch6) |

### 1.2 프로그래머블 RAN — Native AI로 가는 통로

Native AI를 실현하는 실용적 접근은 **프로그래머블 RAN**입니다. `E2SM-KPM`(E2 Service Model — Key Performance Measurement)[^a-e2sm-kpm] 같은 인터페이스로 RAN에서 데이터를 수집·상호작용하고, 세 가지 구성요소를 프로그래밍 가능하게 만듭니다[^r28].

| 구성요소 | 내용 |
|---|---|
| **파라미터(Parameter)** | 개방형 인터페이스를 통해 AI 모델과 SW 정의 RAN 기능 간 파라미터를 원활히 적응 |
| **데이터(Data)** | AI 모델 훈련용 데이터셋 구성 + RAN 기능 내 데이터 관계 탐색 |
| **동작(Behaviour)** | 다양한 AI 모델을 활용해 RAN 동작을 수정, **모델을 동적으로 업데이트·교체** |

> **보안 관점의 경고**: "모델을 동적으로 업데이트·교체할 수 있다"는 것은 곧 **모델 교체 경로가 공격 경로**라는 뜻입니다. Ch6의 ML 공급망 공격, Ch7의 AIBOM(AI Bill of Materials)[^a-aibom]이 바로 이 지점을 다룹니다.
{: .prompt-danger }

### 1.3 중앙집중형 AI vs 분산형 AI

| 구분 | 중앙집중형 AI | 분산형 AI |
|---|---|---|
| 위치 | 디바이스/클라우드의 중앙 시스템 | 6G 네트워크 어디에나 배치 |
| 강점 | 데이터·자원 통제, 중앙집중 의사결정 이점 | 대역폭 효율, 낮은 지연, **데이터 프라이버시 해결** |
| 약점 | 지연 민감 서비스 부적합 | 협업 조율 복잡, 부분 관측 |
| 선택 기준 | 애플리케이션·데이터 구축·프라이버시 요구사항에 따라 결정 (공존도 가능) | 동일 |

여기에 **DTN**(Digital Twin Network)[^a-dtn]을 결합해 분산형 Native AI + 분산형 DT 시스템을 구성하는 방향이 제시됩니다. DT는 Native AI를 실행할 인프라를 제공하고, AI 컴퓨팅·처리를 시뮬레이션 및 온라인 모두에서 DT 도메인에서 수행할 수 있게 합니다[^r28]. 이 개념은 Ch10(디지털 트윈 기반 검증)과 Ch11(테스트베드)에서 다시 등장합니다.

---

## 2. RAN 아키텍처 진화의 맥락

![D-RAN에서 O-RAN으로의 아키텍처 전환 (출처: Rathakrishnan 등[^r1], Fig. 2)](/assets/img/posts/6g-ai-ran/airan6g-fig2.png)
_그림 1-1. D-RAN → O-RAN 아키텍처 전환. 셀 사이트에 모든 기능이 있던 구조에서, 기능이 분리되어 클라우드로 이동합니다. 출처: [^r1], Fig. 2._

![RAN의 세대별 진화 (출처: Salmi 등[^r4], Fig. 3)](/assets/img/posts/6g-ai-ran/ainative-fig3.png)
_그림 1-2. RAN 진화 — 폐쇄형 단일 기지국에서 개방·지능형 RAN까지. 출처: [^r4], Fig. 3._

각 단계가 무엇을 해결하고 무엇을 남겼는지 정리하면 다음과 같습니다. (구조적 세부는 Ch2에서 상세히 다룹니다.)

| 단계 | 해결한 문제 | 남긴 문제 |
|---|---|---|
| **D-RAN** (분산형) | 커버리지 확대 | RF[^a-rf]-안테나 간 동축케이블 비용, 사이트별 중복 투자 |
| **C-RAN** (BBU 풀) | BBU[^a-bbu] 중앙화로 처리자원 공유, CAPEX[^a-capex] 절감 | 프론트홀 대역폭 폭증(CPRI[^a-cpri]), 벤더 종속 |
| **vRAN / SD-RAN[^a-sd-ran]** | 범용 서버·가상화, 제어/데이터 평면 분리 | 벤더 통제 AI, 폐쇄 인터페이스 → 상호운용성 제약 |
| **O-RAN** | 개방 인터페이스, 멀티벤더, RIC(RAN Intelligent Controller)[^a-ric] 기반 지능 | **공격 표면 확대**, 제3자 xApp 신뢰 문제 |
| **AI-RAN** | AI가 RAN 성능·자원·수익 모델을 재정의 | **AI 자체가 표적** — 이 시리즈 Part II·III의 주제 |

---

## 3. AI-RAN Alliance의 세 기둥 — AI for / and / on RAN

2024년 SoftBank와 NVIDIA가 결성한 **AI-RAN Alliance**는 회원사 100여 개 규모로 성장했고 국내 이통 3사도 참여하고 있습니다[^r30]. Alliance는 표준을 직접 만들지 않고 3GPP(3rd Generation Partnership Project)[^a-3gpp]·O-RAN Alliance 규격에 영향을 주는 방식으로 작동하며[^r2], 세 개의 작업 축(working group)을 둡니다.

### 3.1 세 기둥 비교

Netmanias 손장우[^r30]의 정리와 Polese 등[^r2]의 기술적 정의를 통합했습니다.

| 구분 | **AI for RAN** | **AI and RAN** | **AI on RAN** |
|---|---|---|---|
| **한 줄 정의** | AI/ML로 RAN 성능을 개선·최적화 | RAN 워크로드와 AI 워크로드가 하나의 컴퓨팅 자원을 공유 | RAN 인프라에서 AI 응용 서비스를 실행해 판매 |
| **무엇을 바꾸나** | 규칙 기반 알고리즘 → AI/ML 모델을 기지국에 내장 | 자원 할당 정책 (비탄력 RAN + 탄력 AI 공존) | RAN을 분산 엣지 AI 플랫폼으로 확장 |
| **주요 예제** | AI 채널 추정·보간, 간섭 제어, MAC[^a-mac] 스케줄링, 빔포밍 최적화, 전력 제어 | 심야·유휴 시간대 RAN 서버 GPU를 외부 AI 학습/추론에 대여 | 실시간 영상 분석, 자율주행, 원격 수술 보조, 실시간 로봇 제어, AR[^a-ar], LLM[^a-llm], 기업용 RAG[^a-rag], Agentic AI |
| **효과** | UE 통신 품질 향상, 투자비·운용비 절감 | 유휴 자원 유연 분배로 **TCO[^a-tco] 절감** + 신규 수익 | RAN 접속 단말 대상 **초저지연 AI 서비스** |
| **워크로드 특성**[^r2] | RAN DSP[^a-dsp] = **비탄력(non-elastic)**, 고정·예측 가능한 자원, 타이밍 제약 필수 | 이종 워크로드 공존 = 성능 격리(isolation)가 핵심 | AI 추론/학습 = **탄력적(elastic)**, 자원만 충분하면 유연 |

![AI-RAN 6G의 핵심 특징 (출처: Khan & Schmid[^r3], Fig. 2)](/assets/img/posts/6g-ai-ran/airansota-fig2.png)
_그림 1-3. AI-RAN 6G의 핵심 특징. 출처: [^r3], Fig. 2._

### 3.2 실증 사례로 본 세 기둥

Netmanias 정리[^r30]에 실린 주요 데모·실증 사례입니다. 수치는 각 데모 발표 기준입니다.

| 시기 | 주체 | 기둥 | 내용 및 효과 |
|---|---|---|---|
| MWC 2024 | SoftBank / NVIDIA / Fujitsu | AI for RAN | **AI Channel Interpolation** 적용 → 기존 RAN 대비 업링크 처리량 **25% 향상** |
| MWC 2024 | NVIDIA / Radisys / Aarna Networks | AI and RAN | AI와 RAN 리소스 동적 공유 |
| MWC 2024 | NVIDIA / Radisys / Fujitsu | AI on RAN | 5G 카메라 영상의 실시간 객체 검출 |
| MWC 2025 | Samsung | AI for RAN | **셀 경계**에서 스마트폰 업링크 처리량 **30% 이상 향상** |
| MWC 2025 | SoftBank | AI for RAN | 간섭 많은 저 SNR[^a-snr] 환경에서 기존 L1 대비 업링크 처리량 **20~50% 향상** |
| MWC 2025 | DeepSig / NVIDIA | AI for RAN | 신경망 기반 인코더·수신기·디코더가 기존 파일럿/변조 방식을 대체하면서도 **5G-NR[^a-nr] 호환 유지** |
| MWC 2025 | Northeastern University | AI and RAN | **AutoRAN** — AI와 RAN의 공존 실증 |
| MWC 2025 | ARM / Tannera / Effinet / Phluido | AI on RAN | 엣지 객체 검출 |
| 2025 | **뉴젠스 (국내)** | AI on RAN | 김포국제공항 국제선 구역에 국내 최초 **가상화 기지국(AI-RAN) 기반 5G 특화망 실증단지** 구축. 출입제한구역 침입·역행·이상행동을 AI CCTV로 실시간 감지, 경보 시스템 자동 연동. AI 알고리즘이 신호 품질도 개선 |
| 2025.12 | SoftBank / Yaskawa Electric | AI on RAN | **MEC[^a-mec] AI가 빌딩 내 로봇군 관리·제어** (Physical AI) |
| 2026.02 | SoftBank / Ericsson | AI on RAN | **동적 AI 오프로드** 기술 실증 |
| 2024~ | SoftBank | AI on RAN | 초저지연 LLM 기반 클라우드 로봇 제어 / **AITRAS 엣지 AI 서버에서 폐쇄 처리하는 기업용 RAG** |

> **원문 그림 보기**: Netmanias 기사[^r30]에는 위 사례별 구조도가 실려 있습니다. 저작권 보호를 위해 본 시리즈에는 이미지를 재수록하지 않고 원문 링크로 안내합니다.
> AI for RAN 개념 <https://www.netmanias.com/ko/?m=attach&no=44436> · AI and RAN <https://www.netmanias.com/ko/?m=attach&no=44437> · AI on RAN <https://www.netmanias.com/ko/?m=attach&no=44438> · MWC 2024 SoftBank/NVIDIA <https://www.netmanias.com/ko/?m=attach&no=44478> · MWC 2025 Samsung <https://www.netmanias.com/ko/?m=attach&no=44485> · SoftBank Physical AI 실증 구조 <https://www.netmanias.com/ko/?m=attach&no=44780> · 뉴젠스 김포공항 <https://www.netmanias.com/ko/?m=attach&no=44557>
{: .prompt-info }

### 3.3 왜 "AI and RAN"이 보안의 분기점인가

세 기둥 중 보안 지형을 가장 크게 바꾸는 것은 **AI and RAN**입니다.

Polese 등[^r2]은 그 이유를 자원·워크로드 특성 차이로 설명합니다. RAN DSP(Digital Signal Processing)는 채널 코딩/디코딩·스케줄링처럼 **정해진 시간 안에 반드시 끝나야 하는 비탄력 작업**이고, 설정(대역폭·MIMO[^a-mimo] 스트림 수)이 정해지면 필요한 자원량이 고정·예측 가능합니다. 반면 AI 학습·추론은 탄력적입니다. 이 둘을 같은 서버에 얹으면:

| 위험 | 설명 | 관련 장 |
|---|---|---|
| **성능 격리 실패** | 대규모 AI 학습이 지연 스파이크를 만들어 RAN 성능을 훼손[^r2] | Ch6 (자원 고갈), Ch11 (공존 실측) |
| **제3자 코드 유입** | 외부 AI 워크로드의 소프트웨어 스택 취약점이 RAN 안정성에 영향 → 네트워크 혼잡, 서비스 품질 저하, 대규모 장애 가능[^r2] | Ch4, Ch5 |
| **접근·권한 관리** | 오케스트레이터 API[^a-api]와 제어·인프라 관리 파이프라인이 보안 역할·권한을 추적·설명할 수 있어야 함[^r2] | Ch7 (Zero Trust) |

> Polese 등[^r2]은 이를 명시적으로 지적합니다: *"제3자 AI 워크로드 통합은 보안·프라이버시·접근 관리에 도전 과제를 제시한다. … 소프트웨어 스택에 취약점이 있는 AI 모델/워크로드 배포는 RAN 안정성에 영향을 줄 수 있고, 그런 공격은 네트워크 혼잡, 서비스 품질 저하, 대규모 장애로 이어질 수 있다."*
{: .prompt-warning }

---

## 4. AI-RAN 핵심 인에이블러

Rathakrishnan 등[^r1]은 AI-RAN을 가능케 하는 네 가지 기술 인에이블러를 제시합니다.

| 인에이블러 | 역할 | 보안적 이면 |
|---|---|---|
| **디지털 트윈(DT)** | 물리 RAN의 가상 복제본에서 정책·모델을 사전 검증 | 트윈 자체가 민감 데이터 집합체 / 트윈-실물 불일치 악용 |
| **IRS**(Intelligent Reflecting Surface) | 무선 환경 자체를 재구성해 적응적 최적화 | 물리계층 조작 표면 확대 (재밍·스푸핑) |
| **대규모 생성 AI(GenAI/LLM)** | 의도(intent) → 정책 변환, 운영자 대화 인터페이스 | 프롬프트 인젝션·탈옥·환각 → 잘못된 망 제어 (Ch5) |
| **블록체인(BC)[^a-bc]** | 다자간 자원 거래·감사 추적 신뢰 | 온체인 지연·비용이 실시간 제어와 상충 |

![RAN-LAM 프레임워크와 AI-RAN을 위한 블록체인 워크플로우/유즈케이스 (출처: Rathakrishnan 등[^r1], Fig. 3)](/assets/img/posts/6g-ai-ran/airan6g-fig3.png)
_그림 1-4. RAN-LAM(Large AI Model) 프레임워크와 블록체인 워크플로우. 출처: [^r1], Fig. 3._

![제안된 6G AI-RAN 아키텍처 (출처: Rathakrishnan 등[^r1], Fig. 4)](/assets/img/posts/6g-ai-ran/airan6g-fig4.png)
_그림 1-5. Rathakrishnan 등이 제안한 6G AI-RAN 아키텍처. 운영자 상호작용 계층(Operator Interaction Layer)이 최상단에 위치하는 점에 주목하십시오 — 이 계층이 Ch5에서 다루는 LLM 에이전트 공격면입니다. 출처: [^r1], Fig. 4._

![OrchestRAN 기반 제안 아키텍처와 전통 스케줄링 방식의 성능 비교 (출처: Rathakrishnan 등[^r1], Fig. 5)](/assets/img/posts/6g-ai-ran/airan6g-fig5.png)
_그림 1-6. 지능형 오케스트레이션·자원 최적화 시나리오에서의 성능 비교. 출처: [^r1], Fig. 5._

---

## 5. 6G AI-RAN 유즈케이스와 보안 요구

![6G AI-RAN 유즈케이스 (출처: Khan & Schmid[^r3], Fig. 4)](/assets/img/posts/6g-ai-ran/airansota-fig4.png)
_그림 1-7. 6G AI-RAN 유즈케이스 전경. 출처: [^r3], Fig. 4._

Khan & Schmid[^r3]는 AI-RAN 6G의 연구 이슈를 **스펙트럼 할당 · 네트워크 아키텍처 · 자원 관리** 세 축으로 정리하고, RF 계획/최적화, 용량·커버리지, 빔포밍, 모빌리티/핸드오버를 AI-RAN 관점에서 재검토합니다.

유즈케이스별로 보안 요구가 어떻게 달라지는지 정리하면 다음과 같습니다.

| 유즈케이스 | 지연/신뢰성 요구 | 지배적 보안 요구 | 관련 장 |
|---|---|---|---|
| 자율주행 · 커넥티드 카 | 초저지연, 초고신뢰 | **안전성(safety)** — 잘못된 제어 결정이 물리적 사고로 | Ch5, Ch10 |
| 원격 수술 보조 · 텔레메디신 | 초저지연, 무결성 | 무결성 + 프라이버시(의료정보) | Ch9, Ch10 |
| 산업 자동화 · 로봇군 제어 | 결정론적 지연 | 가용성, 제어 루프 위조 방지 | Ch5, Ch8 |
| XR[^a-xr] / 홀로그래픽 통신 | 고대역폭 | 자원 고갈 방어, 슬라이스 SLA 보호 | Ch6 |
| 대규모 IoT[^a-iot] | 대량 연결 | **시그널링 스톰**·DDoS[^a-ddos] 방어 | Ch5, Ch8 |
| 엣지 GenAI / 기업용 RAG | 지연 + 기밀성 | 데이터 유출 방지, 테넌트 격리 | Ch9 |
| 자율 네트워크 운영(Zero-touch) | — | **에이전트 결정의 가드레일** | Ch5, Ch10 |

![커버리지와 용량 (출처: Khan & Schmid[^r3], Fig. 3)](/assets/img/posts/6g-ai-ran/airansota-fig3.png)
_그림 1-8. AI-RAN 6G에서의 커버리지·용량 관계. 출처: [^r3], Fig. 3._

---

## 6. 이 장의 요약

- **Add-On AI ≠ Native AI.** 4G/5G는 RAN 밖에서 학습한 모델을 밀어 넣는 방식이었고, 이는 메트릭 전송 부담과 추론 품질 저하를 낳았습니다. 6G Native AI는 메트릭을 RAN 제어·데이터 플로우 안으로 수렴시킵니다[^r28].
- **nGRG 설문**에서 6G RAN 3대 연구 방향은 Native AI RAN, Cloud-friendly RAN, Service-based RAN이며, 보안·프라이버시가 유즈케이스 우선순위에 이미 포함되어 있습니다[^r28].
- **AI-RAN은 세 기둥**입니다: AI **for** RAN(성능), AI **and** RAN(자원 공유), AI **on** RAN(신규 서비스)[^r30], [^r2].
- 이 중 **AI and RAN**이 보안의 분기점입니다. 비탄력 RAN 워크로드와 탄력 AI 워크로드가 같은 인프라를 공유하면서, 성능 격리·제3자 코드 신뢰·권한 추적이 새로운 필수 요구가 됩니다[^r2].
- 인에이블러(DT, IRS, GenAI, BC)는 모두 **효용과 공격면을 동시에** 가져옵니다[^r1].

### 확인 체크리스트

- [ ] Add-On AI와 Native AI의 구조적 차이를 두 문장으로 설명할 수 있는가
- [ ] AI for / and / on RAN을 각각의 목적·효과와 함께 구분할 수 있는가
- [ ] AI and RAN이 왜 성능 격리를 필수 보안 요구로 만드는지 설명할 수 있는가
- [ ] 프로그래머블 RAN의 세 구성요소(파라미터·데이터·동작)를 말할 수 있는가
- [ ] 자신의 관심 유즈케이스에서 지배적 보안 요구가 무엇인지 지목할 수 있는가

**다음 장**: [02. O-RAN 기반 6G AI-RAN 아키텍처 진화](/posts/airan-02-oran-architecture/) — RU·DU·CU의 정확한 역할과 기능 분할(functional split), RIC 내부 구조, 그리고 O-RAN이 AI-RAN으로 확장되는 방식을 다룹니다.

---

### 약어

[^a-ai]: **AI**(Artificial Intelligence): 인공지능. 학습·추론 등 지적 작업을 기계가 수행하도록 하는 기술의 총칭입니다.
[^a-ml]: **ML**(Machine Learning): 기계학습. 데이터에서 패턴을 학습해 예측·결정을 수행하는 AI의 핵심 분야입니다.
[^a-etri]: **ETRI**(Electronics and Telecommunications Research Institute): 한국전자통신연구원. 통신·전자 분야의 국내 대표 정부출연 연구기관입니다.
[^a-ran]: **RAN**(Radio Access Network): 단말과 코어망 사이의 무선 접속을 담당하는 무선 접속망으로, 기지국이 핵심 구성요소입니다.
[^a-d-ran]: **D-RAN**(Distributed RAN): 각 셀 사이트에 RF와 베이스밴드 처리 장비를 모두 두는 전통적 분산형 RAN 구조입니다.
[^a-c-ran]: **C-RAN**(Centralized RAN): 여러 기지국의 BBU를 중앙 전산실에 모아 풀(pool)로 운용하는 중앙집중형 RAN 구조입니다.
[^a-vran]: **vRAN**(virtualized RAN): RAN 기능을 전용 하드웨어가 아닌 범용 서버 위의 소프트웨어로 가상화해 구현하는 구조입니다.
[^a-o-ran]: **O-RAN**(Open Radio Access Network): 개방 인터페이스·멀티벤더·지능형 제어를 지향하는 개방형 RAN 아키텍처(및 이를 표준화하는 O-RAN Alliance 규격)입니다.
[^a-ai-ran]: **AI-RAN**(AI Radio Access Network): AI를 무선 접속망의 설계·운용·서비스에 내재화한 차세대 RAN 개념입니다.
[^a-irs]: **IRS**(Intelligent Reflecting Surface): 지능형 반사 표면. 전파를 원하는 방향으로 반사·조정해 무선 환경 자체를 재구성하는 기술입니다.
[^a-ngrg]: **nGRG**(next Generation Research Group): O-RAN Alliance 산하에서 6G 등 차세대 RAN 연구 방향을 다루는 연구 그룹입니다.
[^a-tn]: **TN**(Transport Network): 전송망. RAN·코어망 등 네트워크 구성요소를 연결하는 유선 전송 인프라입니다.
[^a-cn]: **CN**(Core Network): 코어망. 세션 관리·이동성·과금 등 이동통신망의 중추 기능을 담당합니다.
[^a-smo]: **SMO**(Service Management and Orchestration): O-RAN 도메인의 서비스·자원 관리와 오케스트레이션을 총괄하는 관리 계층입니다.
[^a-dt]: **DT**(Digital Twin): 디지털 트윈. 물리 시스템의 가상 복제본을 만들어 시뮬레이션·검증에 활용하는 기술입니다.
[^a-sba]: **SBA**(Service-Based Architecture): 네트워크 기능을 서비스 단위로 분해해 API로 상호작용하게 하는 5G 코어의 서비스 기반 아키텍처입니다.
[^a-ue]: **UE**(User Equipment): 스마트폰 등 이동통신망에 접속하는 사용자 단말을 가리키는 표준 용어입니다.
[^a-qos]: **QoS**(Quality of Service): 지연·대역폭·손실률 등 통신 서비스의 품질을 보장하기 위한 관리 체계입니다.
[^a-edos]: **EDoS**(Economic Denial of Sustainability): 클라우드 자원의 자동 확장·과금 구조를 악용해 비용을 폭증시켜 서비스의 경제적 지속성을 무너뜨리는 공격입니다.
[^a-sla]: **SLA**(Service Level Agreement): 서비스 제공자와 이용자 간에 품질 수준을 계약으로 보장하는 서비스 수준 협약입니다.
[^a-e2sm-kpm]: **E2SM-KPM**(E2 Service Model — Key Performance Measurement): E2 인터페이스에서 RAN 핵심 성능 측정 지표(KPM)를 수집·보고하기 위한 O-RAN 서비스 모델입니다.
[^a-aibom]: **AIBOM**(AI Bill of Materials): AI 모델의 데이터·모델·의존성 구성 명세서로, 소프트웨어의 SBOM 개념을 AI 공급망으로 확장한 것입니다.
[^a-dtn]: **DTN**(Digital Twin Network): 네트워크 전체를 디지털 트윈으로 구성해 AI 실행·검증 인프라로 활용하는 개념입니다.
[^a-rf]: **RF**(Radio Frequency): 무선 주파수. 공중 인터페이스로 신호를 송수신하는 아날로그 무선 처리 영역을 가리킵니다.
[^a-bbu]: **BBU**(Baseband Unit): 기지국에서 디지털 베이스밴드 신호 처리를 담당하는 장치입니다.
[^a-capex]: **CAPEX**(Capital Expenditure): 자본적 지출. 장비·설비 구축 등 초기 투자 비용을 말합니다.
[^a-cpri]: **CPRI**(Common Public Radio Interface): RF부(RRH)와 BBU 사이 프론트홀 구간 전송을 위한 공용 인터페이스 규격입니다.
[^a-sd-ran]: **SD-RAN**(Software-Defined RAN): SDN 원리를 RAN에 적용해 제어 평면과 데이터 평면을 분리한 소프트웨어 정의 RAN입니다.
[^a-ric]: **RIC**(RAN Intelligent Controller): RAN을 지능적으로 제어·최적화하는 O-RAN의 핵심 컨트롤러로, Near-RT RIC(10ms~1s)과 Non-RT RIC(1s 이상)으로 나뉩니다.
[^a-3gpp]: **3GPP**(3rd Generation Partnership Project): 3G 이후의 이동통신 표준(LTE·5G NR 등)을 제정하는 국제 표준화 협력체입니다.
[^a-mac]: **MAC**(Medium Access Control): 무선 자원 스케줄링·다중화를 담당하는 L2 매체 접근 제어 계층입니다.
[^a-ar]: **AR**(Augmented Reality): 증강현실. 현실 공간에 가상 정보를 겹쳐 보여주는 기술입니다.
[^a-llm]: **LLM**(Large Language Model): 대규모 말뭉치로 학습된 초거대 언어 모델로, 자연어 이해·생성뿐 아니라 네트워크 운영 에이전트의 두뇌로도 활용됩니다.
[^a-rag]: **RAG**(Retrieval-Augmented Generation): 검색 증강 생성. LLM이 외부 지식 저장소에서 관련 문서를 검색해 답변 생성에 활용하는 기법입니다.
[^a-tco]: **TCO**(Total Cost of Ownership): 총소유비용. 도입부터 운영·폐기까지 드는 전체 비용입니다.
[^a-dsp]: **DSP**(Digital Signal Processing): 디지털 신호 처리. RAN에서는 채널 코딩·변복조 등 시간 제약이 엄격한 베이스밴드 연산 워크로드를 가리킵니다.
[^a-snr]: **SNR**(Signal-to-Noise Ratio): 신호 대 잡음비. 값이 낮을수록 잡음·간섭이 심한 무선 환경입니다.
[^a-nr]: **NR**(New Radio): 3GPP가 표준화한 5G 무선 접속 기술 규격의 이름입니다.
[^a-mec]: **MEC**(Multi-access Edge Computing): 기지국 등 네트워크 가장자리에 컴퓨팅 자원을 두어 초저지연 서비스를 제공하는 엣지 컴퓨팅 기술입니다.
[^a-mimo]: **MIMO**(Multiple-Input Multiple-Output): 다중 안테나로 여러 데이터 스트림을 동시에 송수신해 용량을 높이는 기술입니다.
[^a-api]: **API**(Application Programming Interface): 소프트웨어 구성요소 간 기능 호출 규약·인터페이스입니다.
[^a-bc]: **BC**(Blockchain): 블록체인. 분산 원장으로 다자간 거래·감사 기록의 무결성을 보장하는 기술입니다.
[^a-xr]: **XR**(eXtended Reality): 확장현실. VR·AR·MR을 아우르는 몰입형 미디어 기술의 총칭입니다.
[^a-iot]: **IoT**(Internet of Things): 사물인터넷. 다양한 사물이 네트워크에 연결되어 데이터를 주고받는 환경입니다.
[^a-ddos]: **DDoS**(Distributed Denial of Service): 다수의 분산된 소스에서 트래픽을 집중시켜 서비스를 마비시키는 분산 서비스 거부 공격입니다.

## References

[^r1]: M. Rathakrishnan, S. Gayan, R. Singh, A. Kaur, H. Inaltekin, S. Edirisinghe, and H. V. Poor, "Towards AI-driven RANs for 6G and beyond: Architectural advancements and future horizons," *arXiv preprint* arXiv:2506.16070, Jun. 2025.
[^r2]: M. Polese, N. Mohamadi, S. D'Oro, L. Bonati, and T. Melodia, "Beyond connectivity: An open architecture for AI-RAN convergence in 6G," *arXiv preprint* arXiv:2507.06911v2, Dec. 2025.
[^r3]: N. A. Khan and S. Schmid, "AI-RAN in 6G networks: State-of-the-art and challenges," *IEEE Open Journal of the Communications Society*, vol. 5, pp. 294–311, 2024, doi: 10.1109/OJCOMS.2023.3343069.
[^r4]: S. Salmi, M. A. Ouameur, M. Bagaa, G. C. Alexandropoulos, A. Tahenni, D. Massicotte, and A. Ksentini, "AI-native O-RAN architectures for 6G: Towards real-time adaptation, conflict resolution, and efficient resource management," *TechRxiv preprint*, Sep. 2025.
[^r28]: 권동승, 나지현, "O-RAN에서 6G RAN 연구 방향," *전자통신동향분석*, vol. 40, no. 5, pp. 101–112, Oct. 2025, doi: 10.22648/ETRI.2025.J.400510.
[^r30]: 손장우, "사례로 알아보는 AI-RAN 개념 — AI for RAN, AI and RAN, AI on RAN," *Netmanias*, Feb. 24, 2026. [Online]. Available: <https://www.netmanias.com/ko/?m=view&id=oneshot&no=16477>
