---
title: "[AI 보안 자동화 Lab] 09-05. SIEM 실전 관제 실습 ⑥ — 인공지능에게 보안 데이터 가르치기"
date: 2026-12-28 09:00:00 +0900
categories:
  - 1.응용강의
  - AI자동화
tags:
  - 보안관제
  - SOC
  - 인공지능
  - 머신러닝
  - Jupyter
pin:
mermaid: true
---

# 보안관제 실습 — 인공지능(AI)에게 보안 데이터 가르치기

---

## 0. 이번 강의 한눈에

- **지난 강의 도달 상태**: Wazuh 규칙 기반으로 SQL Injection을 탐지·분석 완료
- **오늘의 목표**: 규칙으로는 놓치는 변종 공격을 잡기 위해 **AI(머신러닝)**에 정상/공격 트래픽을 **직접 학습**시키고 정확도를 평가
- **오늘 켜는 VM**: **VM2 한 대만** (메모리 절약 — 다른 VM은 꺼도 됩니다)
- **사용 도구**: Python + Jupyter Notebook + scikit-learn (모두 무료 오픈소스)

> 오늘은 네트워크 연결이 필요 없습니다. VM2에서 Python 코드를 직접 짜고 실행해 AI를 만들어 봅니다.
{: .prompt-info }

---

## 1. 오늘의 이론: 왜 보안관제에 AI가 필요할까?

지금까지는 사람이 만든 **규칙(Rule)**으로 공격을 잡았습니다. 하지만 규칙에는 한계가 있습니다.

1. **변종 미탐**: 해커가 코드를 살짝 비틀면(예: `'` 대신 16진수) 규칙이 못 잡습니다.
2. **오탐 폭주**: 정상 사용자가 이름에 기호를 넣었을 뿐인데 경보가 울립니다.

**머신러닝(Machine Learning)**은 정상 패턴과 공격 데이터를 잔뜩 학습해, 규칙에 딱 맞지 않아도 **"공격일 확률 98%"** 처럼 스스로 판단합니다. AI 모델 개발은 보통 3단계입니다.

1. **데이터 준비**: 학습용 트래픽 데이터를 숫자로 정리
2. **학습(Training)**: 알고리즘에 데이터를 넣어 정상/공격 경계선을 찾게 함
3. **평가(Prediction)**: 새 데이터로 정확도를 채점

> 현장에서는 `CICIDS2017` 같은 수 GB짜리 대형 데이터셋을 씁니다. 오늘은 16GB PC에서도 1초 만에 돌아가도록 **같은 원리의 소형 데이터를 코드로 생성**해 실습합니다.
{: .prompt-tip }

---

## 2. 따라하기 실습 ① : Jupyter 개발환경 직접 만들기 (VM2)

### [단계 1] 파이썬 도구 설치

VM2 터미널(`Ctrl+Alt+T`)에서:
```bash
sudo apt update
sudo apt install -y python3-pip python3-venv
```

**명령어 상세 설명**
- `sudo apt update` : 설치 가능한 프로그램 목록을 최신으로 갱신(설치 전 준비).
- `sudo apt install -y python3-pip python3-venv` : 파이썬 라이브러리 설치 도구 **`pip`**과 격리 환경 도구 **`venv`**를 설치합니다. 이 둘이 있어야 다음 단계에서 AI 라이브러리를 깔 수 있습니다. (`-y`=설치 확인에 자동 yes)

### [단계 2] 실습용 가상환경(venv) 만들기

프로젝트 폴더를 만들고 격리된 파이썬 환경을 구성합니다.
```bash
mkdir -p ~/ai-soc && cd ~/ai-soc
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install jupyter scikit-learn pandas matplotlib
```

**명령어 상세 설명**
- `mkdir -p ~/ai-soc && cd ~/ai-soc` : 홈 폴더 아래에 작업 폴더 `ai-soc`를 만들고(`mkdir`, `-p`=이미 있어도 오류 안 냄) 그 안으로 들어갑니다(`cd`).
- `python3 -m venv venv` : 이 프로젝트만의 **격리된 파이썬 환경**을 `venv`라는 폴더로 만듭니다. 라이브러리를 여기에만 깔면 시스템 전체나 다른 프로젝트와 **충돌하지 않습니다**.
- `source venv/bin/activate` : 방금 만든 가상환경을 **켭니다**. 성공하면 프롬프트 맨 앞에 **`(venv)`**가 붙습니다. 앞으로 이 표시가 보이는 상태에서만 작업합니다.
- `pip install --upgrade pip` : 라이브러리 설치 도구 `pip` 자신을 최신 버전으로 올립니다(설치 오류 예방).
- `pip install jupyter scikit-learn pandas matplotlib` : 실습에 쓸 4개 도구를 한 번에 설치합니다. **jupyter**(코드를 칸 단위로 실행하는 노트), **scikit-learn**(머신러닝), **pandas**(표 데이터 처리), **matplotlib**(그래프 그리기).

### [단계 3] Jupyter Notebook 실행

```bash
jupyter notebook
```

**명령어 상세 설명**
- `jupyter notebook` : 웹 브라우저에서 코드를 한 칸씩 실행하는 **Jupyter 노트북 서버를 띄웁니다**. 실행하면 Firefox가 자동으로 열리며 파일 목록이 나타납니다. (이 터미널은 서버가 도는 동안 **그대로 두세요** — 닫으면 노트북도 꺼집니다.)
- 잠시 후 Firefox가 자동으로 열리며 Jupyter 파일 목록 화면이 나타납니다.
- 우측 상단 **[New] → [Python 3]** 을 눌러 새 노트북을 만듭니다.

> Jupyter는 **코드 셀(회색 상자)**을 한 칸씩 실행하는 도구입니다. 셀을 클릭하고 **`Shift + Enter`** 를 누르면 그 칸이 실행됩니다.
{: .prompt-tip }

---

## 3. 따라하기 실습 ② : AI 침입탐지 모델 만들기

아래 **세 개의 셀**을 차례대로 입력하고 각각 `Shift + Enter`로 실행합니다.

### [셀 1] 데이터 준비 — 정상/공격 트래픽 만들기

```python
# 1. 데이터 준비: 네트워크 트래픽 흐름을 숫자 특징(feature)으로 표현
import numpy as np
import pandas as pd

rng = np.random.default_rng(42)   # 항상 같은 결과가 나오도록 고정

# 정상 트래픽 1000건: 패킷속도/연결시간/바이트가 적당한 범위
normal = pd.DataFrame({
    "packet_rate": rng.normal(50, 10, 1000),
    "duration":    rng.normal(30, 8,  1000),
    "bytes":       rng.normal(5000, 800, 1000),
    "failed_conn": rng.normal(1, 0.5, 1000),
    "label": 0    # 0 = 정상
})

# 공격 트래픽 1000건: 패킷속도↑, 연결시간↓, 실패연결↑ (DoS/포트스캔 특성)
attack = pd.DataFrame({
    "packet_rate": rng.normal(300, 40, 1000),
    "duration":    rng.normal(3, 1.5, 1000),
    "bytes":       rng.normal(800, 300, 1000),
    "failed_conn": rng.normal(20, 5, 1000),
    "label": 1    # 1 = 공격
})

data = pd.concat([normal, attack], ignore_index=True)
print("총 데이터 개수:", len(data))
data.sample(5)   # 무작위 5건 미리보기
```
→ 패킷속도·연결시간·바이트·실패연결 등 4개 특징이 숫자로 든 표 5줄이 출력됩니다.

### [셀 2] 머신러닝 모델 학습 (Random Forest)

```python
# 2. 학습: 데이터를 학습용/시험용으로 나누고 Random Forest로 학습
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier

X = data[["packet_rate", "duration", "bytes", "failed_conn"]]
y = data["label"]

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42)

model = RandomForestClassifier(n_estimators=100, random_state=42)
model.fit(X_train, y_train)       # ← AI가 정상/공격 경계선을 스스로 학습
print("학습 완료! 사용한 학습 데이터:", len(X_train), "건")
```
→ 가장 널리 쓰이는 분류 알고리즘 **Random Forest**가 정상/공격을 구분하는 법을 배웁니다.

### [셀 3] 모델 평가 — 성적표 확인

```python
# 3. 평가: 시험용 데이터로 정확도와 혼동행렬 확인
from sklearn.metrics import accuracy_score, confusion_matrix, ConfusionMatrixDisplay
import matplotlib.pyplot as plt

pred = model.predict(X_test)
acc = accuracy_score(y_test, pred)
print(f"Accuracy(정확도): {acc:.3f}  →  {acc*100:.1f}% 확률로 정상/공격 구별")

cm = confusion_matrix(y_test, pred)
ConfusionMatrixDisplay(cm, display_labels=["정상", "공격"]).plot()
plt.title("Confusion Matrix")
plt.show()
```
- **Accuracy**: 정상/공격을 올바르게 맞춘 비율 (보통 0.99 이상 나옵니다).
- **Confusion Matrix**: 정상을 공격으로 착각(오탐), 공격을 놓침(미탐) 건수를 사각형 그리드로 시각화합니다.

---

## 4. 자주 나는 오류

- **`ModuleNotFoundError: sklearn`**: `(venv)`가 활성화 안 된 상태입니다. `cd ~/ai-soc && source venv/bin/activate` 후 다시 실행.
- **Jupyter 브라우저가 안 열림**: 터미널에 출력된 `http://localhost:8888/?token=...` 주소를 복사해 Firefox에 직접 붙여넣으세요.
- **그림이 안 보임**: 셀 맨 위에 `%matplotlib inline` 을 추가하고 다시 실행.

---

## 5. 핵심 요약

- 규칙 기반 탐지는 변종에 약하지만, **머신러닝**은 데이터에서 스스로 정상/이상 경계를 학습합니다.
- **Jupyter + scikit-learn**으로 비전공자도 **데이터 준비 → 학습 → 평가**의 AI 개발 전 과정을 `Shift+Enter`로 체험했습니다.
- 오늘 만든 "확률 판단" 개념은, 34강에서 **로컬 AI(LLM)**가 로그를 읽고 위험도를 판정하는 자동화로 이어집니다.

## 💾 안전망

33강은 VM2에 파이썬 도구만 추가하는 가벼운 작업입니다. 원하면 `Ubuntu-Workstation` 스냅샷을 새로 찍어 두세요. (필수는 아님)

---

## 6. 다음 강의 예고

34강에서는 관제 서버 VM3에 **로컬 AI(Ollama)**와 **자동 대응(SOAR) 파이프라인**을 구축해, 공격 탐지 시 **AI가 위협을 판정하고 방화벽에 자동 차단을 지시**하는 로봇을 만듭니다.
