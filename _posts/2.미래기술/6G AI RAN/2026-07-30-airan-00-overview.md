---
title: "[6G AI-RAN] 00. 시리즈 개요 — 6G AI-RAN과 보안 지형도"
date: 2026-07-30 09:00:00 +0900
categories:
  - 2.미래보안
  - 6G AI RAN
  - 개요
tags:
  - 6G
  - AI-RAN
  - O-RAN
  - RIC
  - Zero-Trust
  - Adversarial-ML
pin: true
math: true
mermaid: false
---

# 6G AI-RAN 시리즈 개요

## 들어가며 — 왜 지금 "AI-RAN 보안"인가

이 시리즈는 **6G AI-RAN(AI Radio Access Network)[^a-ai-ran]의 개념·아키텍처와, 지금까지 학계·표준화 단체에서 제기된 보안 이슈 전체**를 한 흐름으로 정리합니다.

핵심 문제의식은 하나입니다.

> **AI는 RAN 보안의 "해결책"이자 동시에 "새로운 공격 표면"이다.**
> Benzaïd 등[^r5]은 이를 *double-edged sword*(양날의 검)라고 표현합니다. AI/ML은 near-real-time 위협 탐지·자율 대응을 가능하게 하지만, 그 AI 모델 자체가 오염(poisoning)·회피(evasion)·모델 탈취·공급망 침해의 표적이 됩니다.
{: .prompt-warning }

전통적 RAN(Radio Access Network)[^a-ran]에서는 이런 고민이 필요 없었습니다. 기지국은 단일 벤더의 폐쇄된 하드웨어였고, 제어 논리는 규칙 기반이었으며, 외부에서 코드를 주입할 통로가 없었습니다. 그러나 6G를 향한 RAN은 세 가지가 동시에 벌어집니다.

| 변화 | 무엇이 열리는가 | 새로 생기는 위험 |
|---|---|---|
| **분해(Disaggregation)** | 기지국이 RU[^a-ru]/DU[^a-du]/CU[^a-cu]로 쪼개지고 벤더를 섞을 수 있음 | 개방 인터페이스(O-FH[^a-o-fh], F1, E1, E2, A1, O1, O2)가 모두 공격 표면 |
| **클라우드화(Cloudification)** | 범용 서버·컨테이너 위에서 RAN 기능이 동작 | 컨테이너 탈출, 멀티테넌시 격리 실패, 자원 고갈 |
| **지능화(Intelligence)** | xApp/rApp/dApp이 RAN을 실시간 제어 | **제3자 AI[^a-ai] 코드가 무선 자원을 직접 제어** — 오염된 모델 = 오염된 네트워크 |

여기에 2024년 이후 **AI-RAN Alliance**(SoftBank·NVIDIA 주도, 회원사 100여 개)[^r30]가 등장하면서, RAN은 "AI로 최적화되는 망"을 넘어 **"AI 워크로드를 실행하는 분산 컴퓨팅 인프라"** 로까지 확장되었습니다. 즉 통신 사업자의 기지국 서버가 외부 기업의 AI(Artificial Intelligence) 연산을 대신 돌리는 구조입니다. 보안 관점에서 이것은 **신뢰 경계(trust boundary)가 근본적으로 바뀌는 사건**입니다.

---

## 1. 이 시리즈의 구성

| Part | 장 | 핵심 질문 |
|---|---|---|
| **I. 기초·아키텍처** | Ch1 | 6G는 왜 AI-native여야 하는가? AI for/and/on RAN은 각각 무엇인가? |
| | Ch2 | RU·DU·CU는 정확히 무엇을 하는가? O-RAN[^a-o-ran]이 AI-RAN으로 어떻게 확장되는가? |
| | Ch3 | RAN 안에서 AI 모델은 어떤 파이프라인을 타는가? 데이터는 어디서 오는가? |
| **II. 위협** | Ch4 | 공격자 모델을 어떤 차원으로 기술하는가? 표면은 어디까지인가? |
| | Ch5 | 개방 인터페이스와 xApp/LLM[^a-llm] 에이전트 제어 루프는 어떻게 뚫리는가? |
| | Ch6 | RAN AI 모델을 직접 겨냥한 공격은 실제로 얼마나 위험한가? |
| **III. 방어** | Ch7 | RAN에 Zero Trust를 어떻게 적용하는가? ZT-AI Shield[^a-zt]란? |
| | Ch8 | AI로 공격을 탐지·자율 치유하는 실제 시스템은 어떤 것이 있는가? |
| | Ch9 | 프라이버시를 지키면서 학습하려면? 하드웨어 신뢰 기반은? |
| **IV. 검증·미래** | Ch10 | 자율 결정을 어떻게 "검증"하는가? 컴플라이언스 자동화는? |
| | Ch11 | 어떤 테스트베드·데이터셋으로 재현·비교하는가? |
| | Ch12 | 양자내성암호, 3GPP/O-RAN 표준, 남은 연구 과제는? |

---

## 2. 한눈에 보는 6G AI-RAN 보안 지형도

Benzaïd 등[^r5]의 분류를 축으로, 이 시리즈가 다루는 위협·방어를 O-RAN(Open Radio Access Network) ML(Machine Learning)[^a-ml] 파이프라인 단계에 매핑하면 다음과 같습니다.

| ML 파이프라인 단계 | 대표 위협 | 대표 방어 | 다루는 장 |
|---|---|---|---|
| Data Collection | 데이터 오염, 센서/무선 채널 조작(재밍) | 데이터 출처 검증, 이상 데이터 필터링 | Ch3, Ch6, Ch8 |
| Pre-processing | 라벨 플리핑, 백도어 트리거 삽입 | 데이터 정제, 재라벨링 | Ch6 |
| Training | 모델 오염, FL(Federated Learning)[^a-fl] 클라이언트 악성화 | 강건 집계, 적대적 학습, 증류(distillation) | Ch6, Ch9 |
| Testing | 검증 우회, 테스트 커버리지 부족 | 정형 검증, LLM 기반 컴플라이언스 검사 | Ch10, Ch11 |
| Serving(Inference) | 회피 공격, 모델 탈취, 재프로그래밍 | MTD[^a-mtd], 입력 정화, XAI[^a-xai] 기반 판단 근거 검사 | Ch6, Ch7, Ch8 |
| Monitoring | 자원 고갈(EDoS[^a-edos]), 탐지 지연 유도 | 지속 모니터링·테스트, AIBOM[^a-aibom] | Ch7, Ch12 |
| 전 단계 공통 | ML 공급망 침해, 하드웨어 트로이목마 | **ZT-AI Shield**(MTD+AIBOM+XAI+PETs[^a-pets]+GAN[^a-gan]+Mon&Test) | Ch7, Ch9 |

![ZT-AI Shield: O-RAN ML 파이프라인 전 단계를 대상으로 한 다층 방어 프레임워크 (출처: Benzaïd 등[^r5], Fig. 31)](/assets/img/posts/6g-ai-ran/aisurvey-fig31.png)
_그림 0-1. **ZT-AI Shield** — 파이프라인 단계별로 위협(빨강)과 대응 인에이블러(초록)를 매핑한 프레임워크. 이 시리즈 Part III의 골격이 되는 그림입니다. 출처: [^r5], Fig. 31._

---

## 3. RAN 진화의 큰 그림

AI 도입 관점에서 본 이동통신 세대별 진화입니다. 1G~3G는 AI가 없고, 4G에서 최소한의 ML, 5G에서 부분적 AI, 6G에서 **완전한 AI 지원(AI-RAN)** 으로 이동합니다.

![이동통신 세대와 RAN 아키텍처 진화를 AI 도입 관점에서 정리한 그림 (출처: Rathakrishnan 등[^r1], Fig. 1)](/assets/img/posts/6g-ai-ran/airan6g-fig1.png)
_그림 0-2. 세대별 RAN 아키텍처(D-RAN → C-RAN → vRAN/O-RAN → AI-RAN)와 AI 지원 수준의 상관관계. 6G(2030 예상)에서 Large-Scale AI 기반 AI-RAN에 도달합니다. 출처: [^r1], Fig. 1._

| 시기 | RAN 아키텍처 | 제어 알고리즘 | AI 지원 |
|---|---|---|---|
| 1984~ (1G/2G) | D-RAN[^a-d-ran] (분산형, RF[^a-rf]+BBU[^a-bbu] 일체) | 단순 규칙 기반 | 없음 |
| ~2011 (3G/4G 초기) | C-RAN[^a-c-ran] (BBU 풀 중앙화) | ML/DL[^a-dl] 기반 알고리즘 도입 시작 | 최소 |
| 2019~ (5G) | vRAN[^a-vran], O-RAN | RIC[^a-ric]·xApp 기반 폐루프 제어 | 부분적 |
| 2030 예상 (6G) | **AI-RAN** | Large-Scale AI, LLM/에이전트 | **완전** |

---

## 4. 인용 표기 규칙과 참고문헌 정책

이 시리즈는 **IEEE(Institute of Electrical and Electronics Engineers)[^a-ieee] 인용 양식**을 따릅니다.

- 본문에서는 `[R5]` 형태의 각주 링크로 표기하며, 클릭하면 각 글 하단의 References로 이동합니다.
- 그림은 모두 **원 논문에서 추출**했으며, 캡션에 원출처와 원 그림 번호(`Fig. n`)를 명시했습니다. 학습·연구 목적의 인용이며, 저작권은 각 원저작자에게 있습니다.
- 본 시리즈에서 다루는 표준 문서(3GPP[^a-3gpp] TS[^a-ts] 33.501, O-RAN WG11[^a-wg] 규격, NIST[^a-nist] SP[^a-sp] 800-207, NIST FIPS[^a-fips] 203/204/205 등)는 참조 폴더에 원문이 없는 항목도 있어, 해당 절에서 **"표준 참조"** 로 별도 표기했습니다.

> **그림 추출 방식**: 참조 폴더의 PDF에서 캡션("Fig. n" / "Figure n")을 탐지해 상단 도형 영역을 자동 크롭·렌더링했습니다(190 DPI). 벡터 도형과 래스터 이미지를 함께 포함하도록 영역을 확장하므로 본문 텍스트가 섞여 들어가지 않습니다.
{: .prompt-info }

---

## 5. 전체 참고문헌 (IEEE 양식)

이 시리즈에서 사용하는 핵심 문헌의 통합 목록입니다. 각 장은 해당 장에서 인용한 문헌만 하단 References에 다시 나열합니다.
본문 각주 번호 `[R n]`은 아래 목록의 **Rn**과 대응합니다.

### 아키텍처 · AI-RAN 개념

- **[R1]** M. Rathakrishnan, S. Gayan, R. Singh, A. Kaur, H. Inaltekin, S. Edirisinghe, and H. V. Poor, "Towards AI-driven RANs for 6G and beyond: Architectural advancements and future horizons," *arXiv preprint* arXiv:2506.16070, Jun. 2025.
- **[R2]** M. Polese, N. Mohamadi, S. D'Oro, L. Bonati, and T. Melodia, "Beyond connectivity: An open architecture for AI-RAN convergence in 6G," *arXiv preprint* arXiv:2507.06911v2, Dec. 2025.
- **[R3]** N. A. Khan and S. Schmid, "AI-RAN in 6G networks: State-of-the-art and challenges," *IEEE Open Journal of the Communications Society*, vol. 5, pp. 294–311, 2024, doi: 10.1109/OJCOMS.2023.3343069.
- **[R4]** S. Salmi, M. A. Ouameur, M. Bagaa, G. C. Alexandropoulos, A. Tahenni, D. Massicotte, and A. Ksentini, "AI-native O-RAN architectures for 6G: Towards real-time adaptation, conflict resolution, and efficient resource management," *TechRxiv preprint*, Sep. 2025.
- **[R6]** S. Soltani, A. Amanloo, M. Shojafar, and R. Tafazolli, "Intelligent control in 6G open RAN: Security risk or opportunity?" *IEEE Open Journal of the Communications Society*, 2025.
- **[R28]** 권동승, 나지현, "O-RAN에서 6G RAN 연구 방향," *전자통신동향분석*, vol. 40, no. 5, pp. 101–112, Oct. 2025, doi: 10.22648/ETRI.2025.J.400510.
- **[R29]** 김민건, 김준우, 이훈, 배정숙, 김일규, "6G 무선접속망을 위한 AI/ML 기반 지능형 RAN 기술 동향," *전자통신동향분석*, vol. 41, no. 1, pp. 1–10, Feb. 2026, doi: 10.22648/ETRI.2025.J.410101.
- **[R30]** 손장우, "사례로 알아보는 AI-RAN 개념 — AI for RAN, AI and RAN, AI on RAN," *Netmanias*, Feb. 24, 2026. [Online]. Available: <https://www.netmanias.com/ko/?m=view&id=oneshot&no=16477>

### 보안 · 위협

- **[R5]** C. Benzaïd, T. Taleb, R. Cavalcanti, and A. Sánchez, "AI in Open RAN security: Enabler and threat — A comprehensive survey and future perspectives," *TechRxiv preprint*, Dec. 2025.
- **[R7]** N. M. Yungaicela-Naula, V. Sharma, and S. Scott-Hayward, "Misconfiguration in O-RAN: Analysis of the impact of AI/ML," *Computer Networks*, p. 110455, 2024.
- **[R8]** A. Chiejina, B. Kim, K. Chowdhury, and V. K. Shah, "System-level analysis of adversarial attacks and defenses on intelligence in O-RAN based cellular networks," in *Proc. 17th ACM Conf. on Security and Privacy in Wireless and Mobile Networks (WiSec)*, 2024.
- **[R9]** E. Aizikovich, D. Mimran, E. Grolman, Y. Elovici, and A. Shabtai, "Rogue cell: Adversarial attack and defense in untrusted O-RAN setup exploiting the traffic steering xApp," *arXiv preprint*, 2025.
- **[R10]** D. H. Tashman and S. Cherkaoui, "Adversarial attacks in AI-driven RAN slicing: SLA violations and recovery," *arXiv preprint* arXiv:2604.01049, Apr. 2026.
- **[R24]** H. He, S. Fei, and Z. Yan, "Advancing 5G security and privacy with AI: A survey," *ACM Computing Surveys*, 2025.
- **[R27]** D. Agarwal, "Smooth adversarial training for increasing robustness of O-RAN xApps against PGD attacks," M.S. thesis, Dept. of Electrical and Computer Engineering, Northeastern University, Boston, MA, USA, Dec. 2025.
- **[R33]** M. Akon, M. Toufikuzzaman, and S. R. Hussain, "From control to chaos: A comprehensive formal analysis of 5G's access control," in *Proc. 2025 IEEE Symposium on Security and Privacy (SP)*, 2025, pp. 1043–1062.
- **[R34]** T. Yang, S. M. M. Rashid, A. Ranjbar, G. Tan, and S. R. Hussain, "ORANalyst: Systematic testing framework for open RAN implementations," in *Proc. 33rd USENIX Security Symposium*, Philadelphia, PA, USA, Aug. 2024, pp. 1921–1938.
- **[R38]** W. R. Bezerra, E. A. Galhardo, L. M. Bezerra, and C. B. Westphall, "Threats and AI trends in threat modeling for 5G/6G," in *Proc. 40th Int. Conf. Information Networking (ICOIN)*, 2026, pp. 645–649, doi: 10.1109/ICOIN68469.2026.11480655.

### 탐지 · 방어

- **[R11]** A. Scalingi, S. D'Oro, F. Restuccia, T. Melodia, and D. Giustiniano, "Det-RAN: Data-driven cross-layer real-time attack detection in 5G open RANs," in *Proc. IEEE INFOCOM*, 2024, pp. 41–50.
- **[R12]** C. Sun, U. Pawar, M. Khoja, X. Foukas, M. K. Marina, and B. Radunovic, "SpotLight: Accurate, explainable and efficient anomaly detection for open RAN," in *Proc. 30th Annual Int. Conf. on Mobile Computing and Networking (MobiCom)*, 2024, pp. 923–937.
- **[R13]** S. Chatzimiltis, M. Shojafar, M. B. Mashhadi, and R. Tafazolli, "Interpretable anomaly-based DDoS detection in AI-RAN with XAI and LLMs," *arXiv preprint* arXiv:2507.21193, Jul. 2025.
- **[R25]** S. Ben Khalifa, R. Taheri, and Z. Pooranian, "Lightweight intrusion detection baselines for Open RAN xApps," in *Proc. IEEE ICC Workshops*, 2026, doi: 10.1109/ICCWorkshops63917.2026.11586479.
- **[R26]** B. M. Xavier, M. Dzaferagic, D. Collins, G. Comarela, M. Martinello, and M. Ruffini, "Machine learning-based early attack detection using Open RAN intelligent controller," in *Proc. IEEE Int. Conf. on Communications (ICC)*, 2023.
- **[R31]** M. Hashem Eiza, B. Akwirry, A. Raschella, M. Mackay, and M. K. Maheshwari, "A hybrid zero trust deployment model for securing O-RAN architecture in 6G networks," *Future Internet*, vol. 17, no. 8, p. 372, Aug. 2025.
- **[R32]** A. S. Abdalla, J. Moore, N. Adhikari, and V. Marojevic, "ZTRAN: Prototyping zero trust security xApps for open radio access network deployments," *IEEE Wireless Communications Magazine*, vol. 31, no. 2, pp. 66–73, Apr. 2024.
- **[R35]** J. Wang, Y. Wang, A. Li, Y. Xiao, R. Zhang, W. Lou, Y. T. Hou, and N. Zhang, "ARI: Attestation of real-time mission execution integrity," in *Proc. 32nd USENIX Security Symposium*, Anaheim, CA, USA, Aug. 2023.

### 종합 서베이 · 표준 프레임워크

- **[R36]** A. Braeken, D. Deac, T. L. Nguyen, G. Gür, Q.-V. Pham, C. Yapa, P. G. Vinueza-Naranjo, H. Carvajal Mora, C. Moremada, and M. Liyanage, "6G AI security: From fundamentals to offensive and defensive landscape in 6G," *IEEE Communications Surveys & Tutorials*, vol. 28, 2026, doi: 10.1109/COMST.2026.3659793.

### LLM · 에이전트 · 가드레일

- **[R14]** S. Chatzimiltis, M. B. Mashhadi, M. Shojafar, M. Debbah, and R. Tafazolli, "Agentic AI for 6G: A new paradigm for autonomous RAN security compliance," *arXiv preprint* arXiv:2512.12400v2, Apr. 2026.
- **[R37]** H. Feng, T. R. Gadekallu, Y. Xia, Y. Zhao, Z. Wen, J. Cai, P. Bhattacharya, K. Fang, and M. Liyanage, "Agentic AI security in 6G networks: A survey of emerging attack vectors, vulnerabilities, and defenses," *IEEE Open Journal of the Communications Society*, vol. 7, pp. 6334–6370, 2026.
- **[R15]** Y. Tang, M. Zou, W. Guo, and S. A. R. Zaidi, "Guardrailing LLM and agentic decisions for 6G AI-RAN," in *Proc. IEEE 23rd Consumer Communications & Networking Conference (CCNC)*, 2026, doi: 10.1109/CCNC65079.2026.11366609.
- **[R16]** M. Yu, F. Meng, X. Zhou, S. Wang, J. Mao, L. Pang, T. Chen, K. Wang, X. Li, and Y. Zhang, "A survey on trustworthy LLM agents: Threats and countermeasures," *arXiv preprint* arXiv:2503.09648, 2025.
- **[R17]** T. Rebedea, R. Dinu, M. Sreedhar, C. Parisien, and J. Cohen, "NeMo Guardrails: A toolkit for controllable and safe LLM applications with programmable rails," *arXiv preprint* arXiv:2310.10501, Oct. 2023.
- **[R18]** Z. Xiang, L. Zheng, Y. Li, J. Hong, Q. Li, H. Xie, J. Zhang, Z. Xiong, C. Xie, C. Yang, D. Song, and B. Li, "GuardAgent: Safeguard LLM agents via knowledge-enabled reasoning," *arXiv preprint* arXiv:2406.09187v3, May 2025.
- **[R19]** Q. Zhang, Z. Xiong, and Z. M. Mao, "Safeguard is a double-edged sword: Denial-of-service attack on large language models," *arXiv preprint* arXiv:2410.02916, 2024.
- **[R20]** H. Liu, H. Huang, X. Gu, H. Wang, and Y. Wang, "On calibration of LLM-based guard models for reliable content moderation," *arXiv preprint* arXiv:2410.10414, 2024.
- **[R21]** Z. Chu, Y. Wang, L. Li, Z. Wang, Z. Qin, and K. Ren, "A causal explainable guardrails for large language models," in *Proc. 2024 ACM SIGSAC Conf. on Computer and Communications Security (CCS)*, 2024, pp. 1136–1150.
- **[R22]** S. G. Ayyamperumal and L. Ge, "Current state of LLM risks and AI guardrails," *arXiv preprint* arXiv:2406.12934, 2024.
- **[R23]** A. Mekrache, M. Mekki, A. Ksentini, B. Brik, and C. Verikoukis, "On combining XAI and LLMs for trustworthy zero-touch network and service management in 6G," *IEEE Communications Magazine*, 2024.

---

### 약어

[^a-ai-ran]: **AI-RAN**(AI Radio Access Network): AI를 무선 접속망의 설계·운용·서비스에 내재화한 차세대 RAN 개념입니다. AI-RAN Alliance가 개념 확산을 주도하고 있습니다.
[^a-ran]: **RAN**(Radio Access Network): 단말과 코어망 사이의 무선 접속을 담당하는 무선 접속망으로, 기지국이 핵심 구성요소입니다.
[^a-ru]: **RU**(Radio Unit): 분해된 기지국에서 안테나 쪽 RF 처리와 하위 물리계층을 담당하는 무선 유닛입니다.
[^a-du]: **DU**(Distributed Unit): 분해된 기지국에서 상위 물리계층·MAC·RLC 등 지연에 민감한 베이스밴드 처리를 담당하는 분산 유닛입니다.
[^a-cu]: **CU**(Central Unit): 분해된 기지국에서 PDCP·RRC 등 상위 계층을 담당하며 여러 DU를 관장하는 중앙 유닛입니다.
[^a-o-fh]: **O-FH**(Open Fronthaul): O-RAN이 표준화한 RU-DU 간 개방형 프론트홀 인터페이스입니다.
[^a-ai]: **AI**(Artificial Intelligence): 인공지능. 학습·추론 등 지적 작업을 기계가 수행하도록 하는 기술의 총칭입니다.
[^a-o-ran]: **O-RAN**(Open Radio Access Network): 개방 인터페이스·멀티벤더·지능형 제어를 지향하는 개방형 RAN 아키텍처(및 이를 표준화하는 O-RAN Alliance 규격)입니다.
[^a-llm]: **LLM**(Large Language Model): 대규모 말뭉치로 학습된 초거대 언어 모델로, 자연어 이해·생성뿐 아니라 네트워크 운영 에이전트의 두뇌로도 활용됩니다.
[^a-zt]: **ZT**(Zero Trust): "아무것도 신뢰하지 않고 항상 검증한다"는 원칙의 보안 모델입니다. ZT-AI Shield는 이 원칙을 AI 파이프라인 방어에 적용한 프레임워크입니다.
[^a-ml]: **ML**(Machine Learning): 기계학습. 데이터에서 패턴을 학습해 예측·결정을 수행하는 AI의 핵심 분야입니다.
[^a-fl]: **FL**(Federated Learning): 연합학습. 원데이터를 중앙에 모으지 않고 각 노드가 로컬 학습한 모델 업데이트만 공유해 공동 모델을 만드는 분산 학습 기법입니다.
[^a-mtd]: **MTD**(Moving Target Defense): 시스템 구성·모델 등을 지속적으로 변화시켜 공격자가 표적을 고정하지 못하게 하는 방어 기법입니다.
[^a-xai]: **XAI**(eXplainable AI): 설명 가능 AI. 모델의 판단 근거를 사람이 이해할 수 있게 제시하는 기술입니다.
[^a-edos]: **EDoS**(Economic Denial of Sustainability): 클라우드 자원의 자동 확장·과금 구조를 악용해 비용을 폭증시켜 서비스의 경제적 지속성을 무너뜨리는 공격입니다.
[^a-aibom]: **AIBOM**(AI Bill of Materials): AI 모델의 데이터·모델·의존성 구성 명세서로, 소프트웨어의 SBOM(Software Bill of Materials) 개념을 AI 공급망으로 확장한 것입니다.
[^a-pets]: **PETs**(Privacy-Enhancing Technologies): 차분 프라이버시·동형암호 등 데이터 활용 과정에서 개인정보 노출을 줄이는 기술군입니다.
[^a-gan]: **GAN**(Generative Adversarial Network): 생성자와 판별자가 경쟁하며 학습하는 생성형 신경망으로, 데이터 증강이나 공격 트래픽 모사 등에 쓰입니다.
[^a-d-ran]: **D-RAN**(Distributed RAN): 각 셀 사이트에 RF와 베이스밴드 처리 장비를 모두 두는 전통적 분산형 RAN 구조입니다.
[^a-rf]: **RF**(Radio Frequency): 무선 주파수. 공중 인터페이스로 신호를 송수신하는 아날로그 무선 처리 영역을 가리킵니다.
[^a-bbu]: **BBU**(Baseband Unit): 기지국에서 디지털 베이스밴드 신호 처리를 담당하는 장치입니다.
[^a-c-ran]: **C-RAN**(Centralized RAN): 여러 기지국의 BBU를 중앙 전산실에 모아 풀(pool)로 운용하는 중앙집중형 RAN 구조입니다.
[^a-dl]: **DL**(Deep Learning): 딥러닝. 다층 신경망으로 복잡한 패턴을 학습하는 ML의 하위 분야입니다.
[^a-vran]: **vRAN**(virtualized RAN): RAN 기능을 전용 하드웨어가 아닌 범용 서버 위의 소프트웨어로 가상화해 구현하는 구조입니다.
[^a-ric]: **RIC**(RAN Intelligent Controller): RAN을 지능적으로 제어·최적화하는 O-RAN의 핵심 컨트롤러로, Near-RT RIC(10ms~1s)과 Non-RT RIC(1s 이상)으로 나뉩니다.
[^a-ieee]: **IEEE**(Institute of Electrical and Electronics Engineers): 전기전자공학 분야의 국제 학술 단체로, 표준 제정과 논문 인용 양식으로도 널리 알려져 있습니다.
[^a-3gpp]: **3GPP**(3rd Generation Partnership Project): 3G 이후의 이동통신 표준(LTE·5G NR 등)을 제정하는 국제 표준화 협력체입니다.
[^a-ts]: **TS**(Technical Specification): 3GPP·ETSI 등 표준화 단체가 발행하는 기술규격 문서 유형입니다.
[^a-wg]: **WG**(Working Group): 표준화 단체 안에서 특정 주제를 담당하는 작업반입니다. O-RAN WG11은 보안 작업반입니다.
[^a-nist]: **NIST**(National Institute of Standards and Technology): 미국 국립표준기술연구소로, SP·FIPS 시리즈 등 보안·암호 표준 문서를 발행합니다.
[^a-sp]: **SP**(Special Publication): NIST가 발행하는 특별간행물 문서 시리즈로, SP 800 시리즈가 보안 지침으로 널리 쓰입니다.
[^a-fips]: **FIPS**(Federal Information Processing Standards): 미국 연방정보처리표준. FIPS 203/204/205는 양자내성암호 표준입니다.
[^a-ris]: **RIS**(Reconfigurable Intelligent Surface): 다수의 소형 반사 소자로 무선 채널 자체를 프로그래밍 가능하게 재구성하는 지능형 표면으로, 6G의 핵심 물리계층 인에이블러 중 하나입니다.
[^a-mmimo]: **mMIMO**(massive Multiple-Input Multiple-Output): 매우 많은 수의 안테나를 기지국에 배치해 다수 단말에 동시에 빔을 형성하는 대규모 다중안테나 기술입니다.
[^a-ambc]: **AmBC**(Ambient Backscatter Communication): 별도의 능동 송신기 없이 주변 RF 신호를 반사·변조해 통신하는 초저전력 통신 기법입니다.
[^a-prpa]: **PRPA**(Perception-Reasoning-Planning-Action): 에이전틱 AI가 관측을 인식하고 추론·계획한 뒤 행동으로 옮기는 폐루프 제어 사이클을 가리키는 이 시리즈의 핵심 개념입니다.
[^a-dlt]: **DLT**(Distributed Ledger Technology): 블록체인으로 대표되는, 다수 노드가 원장을 분산 공유·검증하는 기술로 AI 모델 계보 추적(AIBOM)에 활용됩니다.
[^a-ntn]: **NTN**(Non-Terrestrial Network): 위성·HAPS 등 지상 기지국이 아닌 비지상 플랫폼을 통해 통신을 제공하는 6G 확장 영역입니다.
[^a-marl]: **MARL**(Multi-Agent Reinforcement Learning): 다수의 에이전트가 상호작용하며 각자 또는 공동의 보상을 학습하는 강화학습 기법으로, 공격·방어 스웜 양쪽에 쓰입니다.
[^a-qkd]: **QKD**(Quantum Key Distribution): 양자 상태의 특성을 이용해 도청 시도가 반드시 검출되는 방식으로 암호키를 분배하는 양자통신 기술입니다.
[^a-pqc]: **PQC**(Post-Quantum Cryptography): 양자컴퓨터의 공격에도 안전하도록 설계된 암호 알고리즘군으로, NIST가 ML-KEM(FIPS 203)·ML-DSA(FIPS 204)·SPHINCS+(FIPS 205)를 표준으로 선정했습니다.
[^a-sba]: **SBA**(Service-Based Architecture): 코어망 기능을 독립적인 서비스 단위로 구현하고 API로 상호작용하게 하는 5G/6G 코어망 설계 방식입니다.
[^a-dt]: **DT**(Digital Twin): 물리적 네트워크의 상태를 실시간으로 반영하는 가상 복제본으로, 정책을 실제 적용 전에 시뮬레이션·검증하는 데 쓰입니다.

### 이 글에서 인용한 문헌

[^r1]: M. Rathakrishnan, S. Gayan, R. Singh, A. Kaur, H. Inaltekin, S. Edirisinghe, and H. V. Poor, "Towards AI-driven RANs for 6G and beyond: Architectural advancements and future horizons," *arXiv preprint* arXiv:2506.16070, Jun. 2025.
[^r30]: 손장우, "사례로 알아보는 AI-RAN 개념 — AI for RAN, AI and RAN, AI on RAN," *Netmanias*, Feb. 24, 2026. [Online]. Available: <https://www.netmanias.com/ko/?m=view&id=oneshot&no=16477>
[^r5]: C. Benzaïd, T. Taleb, R. Cavalcanti, and A. Sánchez, "AI in Open RAN security: Enabler and threat — A comprehensive survey and future perspectives," *TechRxiv preprint*, Dec. 2025.
