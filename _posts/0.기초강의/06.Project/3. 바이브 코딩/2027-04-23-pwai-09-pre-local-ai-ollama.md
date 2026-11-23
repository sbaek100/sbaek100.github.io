---
title: "[기술 매뉴얼] 로컬 AI(Ollama) 구축과 ngrok 외부 접속 구성"
date: 2027-04-23 09:00:00 +0900
categories:
  - 1.응용강의
  - AI와 함께하는 프로젝트
tags:
  - Ollama
  - EXAONE
  - ngrok
  - OpenWebUI
  - 로컬AI
  - 바이브코딩
pin:
mermaid: false
---

> **문서 개요**
> 1. CPU만 있는 환경에서 한국어 처리에 강한 **LG EXAONE 3.0(경량 양자화 버전)**을 Ollama로 설치·구동할 수 있다.
> 2. 방화벽을 열지 않고 **ngrok 터널**로 로컬 AI를 외부 네트워크에 노출할 수 있다.
> 3. 용도에 따라 **API 연동 방식**과 **웹 화면(Open WebUI) 방식** 중 하나를 골라 구성할 수 있다.
{: .prompt-info }


![](/assets/img/posts/2027-04-23-pwai-09-pre-local-ai-ollama-1789947627954.png)

## 0. 이 문서의 범위

본 문서는 **내 PC(또는 내 서버)에서 돌아가는 AI**를 만들고, 그것을 밖에서도 쓸 수 있게 여는 절차를 정리한 기술 매뉴얼이다. 별도의 GPU 없이 CPU만으로 동작하는 구성을 전제로 한다.

설정이 갈리는 지점은 단 하나, **어떤 리눅스 환경에서 실행하는가**이다. 따라서 2절만 환경별로 나뉘고 나머지 절차는 공통이다.

| 환경 | 특징 | 해당 절차 |
|---|---|---|
| **네이티브 리눅스** | Ubuntu 등을 직접 설치한 PC·서버. systemd가 Ollama를 서비스로 관리한다 | 2절 [방법 A] |
| **Windows 내 WSL2** | 윈도우 안의 리눅스. systemd가 꺼져 있는 경우가 많다 | 2절 [방법 B] |

**준비물**:

| 항목 | 비고 |
|---|---|
| 리눅스 환경(네이티브 또는 WSL2) | |
| 여유 메모리 8GB 이상, 디스크 10GB 이상 | 모델 파일만 약 4.5GB |
| 인터넷 연결 | |
| ngrok 계정 | 무료 요금제로 충분하다. [ngrok.com](https://ngrok.com) 가입 후 인증 토큰을 발급받는다 |
| (운영 모드 B만) Docker | WSL이라면 윈도우에 **Docker Desktop**을 설치하고 WSL 연동 옵션을 켜 두어야 한다 |

**이 문서에 나오는 포트**: 혼동하기 쉬우므로 먼저 정리한다.

| 포트 | 정체 | 누가 쓰는가 |
|---|---|---|
| **11434** | Ollama의 기본 포트 | 운영 모드 A에서 이 포트를 터널로 연다 |
| **8080** | Open WebUI **컨테이너 안쪽** 포트 | 밖에서 직접 쓰지 않는다. 3000번으로 연결만 한다 |
| **3000** | Open WebUI를 내 컴퓨터에서 여는 포트 | 운영 모드 B에서 이 포트를 터널로 연다 |

ngrok 자체에는 정해진 포트가 없다. `ngrok http <포트>`의 숫자는 **내 컴퓨터에서 이미 돌아가고 있는 프로그램의 포트**를 가리키며, 외부 주소는 실행할 때마다 ngrok이 발급한다.

---

## 1. Ollama 엔진 설치와 모델 구동 (공통)

네이티브 리눅스와 WSL 모두 동일한 스크립트로 AI 추론 엔진을 설치하고 모델을 구동한다.

> **Ollama**란 AI 모델을 내 컴퓨터에서 실행해 주는 프로그램이다. 모델 내려받기, 메모리 적재, 질문·답변 처리를 모두 담당하며, 다른 프로그램이 사용할 수 있도록 **11434번 포트**에 API 창구를 열어 둔다.

### 1.1 Ollama 설치 스크립트 실행

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### 1.2 EXAONE 3.0 한국어 양자화 모델 내려받기와 구동

Hugging Face에 등록된 EXAONE 3.0 7.8B 인스트럭트 모델의 **Q4_K_M 양자화 버전**(약 4.5GB, CPU 환경에 적합)을 Ollama의 직통 내려받기 기능으로 즉시 호출한다.

```bash
ollama run hf.co/bingsu/EXAONE-3.0-7.8B-Instruct-GGUF:Q4_K_M
```

> **양자화(Quantization)**란 모델 내부 숫자의 정밀도를 낮추어 파일 크기와 메모리 사용량을 줄이는 기법이다. `Q4_K_M`은 4비트로 줄인 방식으로, 품질 손실을 억제하면서 CPU만으로도 돌아갈 정도로 가볍다.

명령을 실행하면 내려받기가 진행되며, 완료 후 프롬프트 창이 나타나면 `Ctrl + D`를 눌러 빠져나온다. 프롬프트를 빠져나와도 **서버는 백그라운드에서 계속 실행**된다.

여기까지 수행하면 로컬에서는 AI가 동작한다. 다음 절부터는 이것을 **밖에서도 쓸 수 있게** 만드는 작업이다.

---

## 2. 외부 접속을 위한 설정 변경 (환경별 분기)

Ollama는 기본적으로 **로컬(127.0.0.1) 요청만** 받아들인다. ngrok을 통해 들어오는 외부 트래픽을 수용하려면 두 가지 환경 변수를 변경해야 한다.

| 환경 변수 | 하는 일 |
|---|---|
| `OLLAMA_HOST=0.0.0.0:11434` | 모든 네트워크 인터페이스에서 요청을 받는다. 기본값은 `127.0.0.1:11434`이므로 자기 자신의 요청만 처리한다 |
| `OLLAMA_ORIGINS=*` | 어느 주소에서 온 웹 요청이든 허용한다. 기본값은 `127.0.0.1`과 `0.0.0.0`뿐이다 |

> 두 설정은 **접근 제한을 푸는 조치**이다. 뒤의 6절(보안 주의사항)을 반드시 함께 읽고, 실습이 끝나면 터널을 닫는 습관을 들인다.
{: .prompt-warning }

### 2.1 [방법 A] 네이티브 리눅스 환경 — systemd 기반

부팅 시 자동으로 백그라운드 실행을 관리하는 systemd 데몬의 환경 변수를 수정한다.

**1단계. 서비스 설정 파일(override) 생성**

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo tee /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_ORIGINS=*"
EOF
```

> 포트를 생략하고 `OLLAMA_HOST=0.0.0.0`으로 적어도 기본 포트 11434가 적용된다. 다만 공식 문서가 제시하는 형태는 포트를 함께 적는 `0.0.0.0:11434`이므로 이쪽을 사용한다.
{: .prompt-tip }

**2단계. 데몬 재구성과 서비스 재시작**

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### 2.2 [방법 B] WSL2 환경 — 사용자 프로필 기반

WSL2는 기본적으로 systemd가 비활성화되어 있는 경우가 많으므로, 사용자 계정의 `.bashrc`에 환경 변수를 직접 주입하여 실행한다.

**1단계. 환경 변수 영구 등록과 적용**

```bash
echo 'export OLLAMA_HOST="0.0.0.0:11434"' >> ~/.bashrc
echo 'export OLLAMA_ORIGINS="*"' >> ~/.bashrc
source ~/.bashrc
```

**2단계. Ollama 서버 백그라운드 실행**

```bash
# WSL 터미널을 닫아도 백그라운드에서 계속 실행되도록 데몬화
nohup ollama serve > ollama.log 2>&1 &
```

> `nohup`으로 띄운 프로세스는 WSL **인스턴스 자체가 살아 있는 동안만** 유지된다. 윈도우에서 WSL 터미널 창을 모두 닫으면 잠시 후 인스턴스가 종료되면서 서버도 함께 내려간다. 실습 중에는 터미널 창을 하나 열어 둔다.
{: .prompt-tip }

---

## 3. ngrok 터널링 환경 구축 (공통)

> **ngrok**이란 내 컴퓨터의 특정 포트를 인터넷 주소로 바꾸어 주는 터널링 도구이다. 공유기나 방화벽의 인바운드 포트를 여는 대신, 내 쪽에서 밖으로 나가는(아웃바운드) 연결을 만들어 그 통로로 외부 요청을 받는 방식이다. 고정 IP가 없어도 되고 공유기 설정을 건드리지 않아도 된다는 것이 장점이다.

### 3.1 ngrok 에이전트 설치

```bash
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com bookworm main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install ngrok
```

> 저장소 주소 끝의 `bookworm`은 데비안 배포판 코드명이다. 오래된 자료에는 `buster`로 적힌 경우가 있는데, 현재 ngrok 공식 설치 안내는 `bookworm`을 사용한다. Ubuntu에서도 그대로 적는다.
{: .prompt-tip }

### 3.2 인증 토큰 등록 (최초 1회)

ngrok 대시보드에서 발급받은 본인의 토큰을 등록한다.

```bash
ngrok config add-authtoken <본인의_ngrok_토큰>
```

여기까지가 공통 준비이다. 이제 **무엇을 외부에 열 것인가**에 따라 두 가지 운영 모드 중 하나를 선택한다.

| 운영 모드 | 여는 대상 | 사용하는 상황 |
|---|---|---|
| **A. API 방식** | Ollama 포트(11434) | 파이썬 스크립트·AI 에이전트·외부 앱의 백엔드와 연동 |
| **B. Web 방식** | Open WebUI 포트(3000) | 사람이 브라우저로 직접 대화. 문서 업로드(RAG)도 가능 |

---

## 4. [운영 모드 A] API 기반 외부 연동

파이썬 스크립트, 자율형 AI 에이전트, 또는 외부 애플리케이션의 백엔드와 연동할 때 사용하는 방식이다.

### 4.1 터널링 실행 (Ollama 기본 포트)

```bash
ngrok http 11434
```

실행하면 화면에 `Forwarding  https://xxxx-xx-xx.ngrok-free.dev -> http://localhost:11434` 형태의 주소가 표시된다. 이 주소가 외부에서 접속할 창구이다.

### 4.2 외부 접속 테스트 (cURL)

발급된 주소를 사용하여 외부망에서 요청을 전송한다. 모델명에는 내려받을 때 사용한 **전체 주소를 그대로** 기입한다.

```bash
curl -H "ngrok-skip-browser-warning: true" \
     https://<발급된_ngrok_주소>/api/generate \
     -d '{"model": "hf.co/bingsu/EXAONE-3.0-7.8B-Instruct-GGUF:Q4_K_M", "prompt": "EXAONE 모델의 주요 장점을 한국어로 요약해 주세요.", "stream": false}'
```

| 요청 요소 | 의미 |
|---|---|
| `ngrok-skip-browser-warning: true` | ngrok 무료 요금제의 경고 페이지를 건너뛰고 API 응답을 직접 받기 위한 헤더 |
| `/api/generate` | Ollama의 단발 질의 창구 |
| `"stream": false` | 답변을 조각으로 흘려보내지 않고 완성된 한 덩어리로 받는다 |

---

## 5. [운영 모드 B] Web(Open WebUI) 기반 외부 연동

ChatGPT와 유사한 그래픽 화면을 통해 AI와 대화하고, 로컬 문서를 업로드(RAG)할 수 있도록 구성하는 방식이다.

> WSL 환경이라면 윈도우에 **Docker Desktop**이 설치되어 있고 WSL 연동 옵션이 켜져 있어야 한다.
{: .prompt-warning }

### 5.1 Open WebUI 컨테이너 실행

포트 충돌을 피하기 위해 컨테이너의 8080 포트를 호스트의 **3000번 포트**로 연결하여 백그라운드(`-d`)에서 실행한다.

```bash
docker run -d -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

| 옵션 | 의미 |
|---|---|
| `-p 3000:8080` | 컨테이너 안의 8080 포트를 내 컴퓨터의 3000번 포트로 연결한다 |
| `--add-host=host.docker.internal:host-gateway` | 컨테이너 안에서 **호스트에 떠 있는 Ollama**를 찾아갈 수 있게 하는 통로이다 |
| `-v open-webui:/app/backend/data` | 대화 기록·계정 정보를 컨테이너 밖에 보존한다 |
| `--restart always` | 재부팅 후에도 자동으로 실행한다 |

### 5.2 터널링 실행 (WebUI 매핑 포트)

기존에 실행 중이던 ngrok 터미널을 `Ctrl + C`로 종료하고, 웹 서버가 작동 중인 포트로 다시 개방한다.

```bash
ngrok http 3000
```

### 5.3 외부 접속과 초기 설정

1. 외부 기기(스마트폰·태블릿 등)의 웹 브라우저에서 ngrok이 발급한 URL로 접속한다.
2. ngrok 경고 화면이 나타나면 `Visit Site` 버튼을 클릭하여 통과한다.
3. 로그인 화면 하단의 **Sign Up**을 클릭하여 최초 최고 관리자(Admin) 계정을 생성한다.
4. 로그인 후 상단 모델 선택 메뉴에서 `hf.co/bingsu/EXAONE-3.0-7.8B-Instruct-GGUF:Q4_K_M`을 지정하고 한국어 대화를 진행한다.

> **가장 먼저 가입한 계정이 관리자**가 된다. 터널 주소를 다른 사람에게 알려 주기 전에 본인 계정부터 만들어 두어야 한다.
{: .prompt-tip }

---

## 6. 보안 주의사항

이 구성은 **인증 없이 열린 AI 창구를 인터넷에 노출**하는 것이다. 주소가 무작위 문자열이라 쉽게 발견되지 않을 뿐, 주소를 아는 사람은 누구나 접근할 수 있다.

| 위험 | 대응 |
|---|---|
| 주소가 유출되면 타인이 내 컴퓨터 자원으로 AI를 사용한다 | 실습이 끝나면 ngrok 터미널을 `Ctrl + C`로 반드시 종료한다 |
| 운영 모드 A는 API에 인증이 전혀 없다 | 장기 운영 시 ngrok의 접근 제어 기능이나 별도 인증 수단을 앞단에 둔다 |
| 운영 모드 B는 먼저 가입한 사람이 관리자가 된다 | 터널을 열자마자 관리자 계정부터 생성하고, 관리자 설정에서 신규 가입을 차단한다 |
| 업로드한 문서가 컨테이너에 남는다 | 민감한 자료는 올리지 않는다 |

> `OLLAMA_HOST=0.0.0.0:11434`과 `OLLAMA_ORIGINS=*`는 학습·시험 목적의 설정이다. 상시 운영하는 서비스에 그대로 사용하지 않는다.
{: .prompt-danger }

---

## 자주 발생하는 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| 외부에서 접속하면 연결이 거부된다 | `OLLAMA_HOST` 설정이 적용되지 않았다 | `ss -tlnp` 출력에서 11434번 포트가 `0.0.0.0`에 묶여 있는지 확인한다. `127.0.0.1`이면 2절을 다시 수행한다 |
| cURL 응답으로 HTML 경고 페이지가 온다 | ngrok 무료 요금제의 경고 화면이다 | 요청에 `ngrok-skip-browser-warning: true` 헤더를 추가한다 |
| Open WebUI에 모델 목록이 비어 있다 | 컨테이너가 호스트의 Ollama를 찾지 못한다 | 관리자 설정의 Ollama 주소를 `http://host.docker.internal:11434`로 지정한다 |
| WSL 터미널을 닫았더니 응답이 없다 | WSL 인스턴스가 종료되며 서버도 내려갔다 | 터미널을 다시 열고 `nohup ollama serve > ollama.log 2>&1 &`를 재실행한다 |
| 답변이 매우 느리다 | CPU 추론의 특성이다 | 짧은 질문으로 시험하고, 다른 무거운 프로그램을 종료하여 메모리 여유를 확보한다 |
| ngrok 주소가 실행할 때마다 바뀐다 | 무료 요금제는 임의 주소를 발급한다 | 실행할 때마다 화면에 표시되는 주소를 확인하여 사용한다 |

## 출처

| 내용 | 근거 |
|---|---|
| 기본 바인딩 `127.0.0.1:11434`, systemd로 `OLLAMA_HOST` 변경, `OLLAMA_ORIGINS`의 기본 허용 범위 | [Ollama 공식 FAQ](https://docs.ollama.com/faq) |
| EXAONE 3.0 7.8B GGUF 양자화 모델 | [Hugging Face 모델 카드](https://huggingface.co/bingsu/EXAONE-3.0-7.8B-Instruct-GGUF) |
| apt 저장소 코드명 `bookworm`, GPG 키 경로 | [ngrok 공식 리눅스 설치 안내](https://ngrok.com/download/linux) |
| `ngrok http [address:port \| port]` 구문 | [ngrok 공식 CLI 문서](https://ngrok.com/docs/agent/cli/) |
| `-p 3000:8080`, `--add-host=host.docker.internal:host-gateway`, 컨테이너의 Ollama 탐색 주소 | [Open WebUI 공식 Quick Start](https://docs.openwebui.com/getting-started/quick-start) |

> 위 항목은 2026년 9월 기준 공식 문서로 대조하였다. 실제 설치·구동 실측은 수행하지 않았으므로, 진행 중 출력이 다르면 각 공식 문서를 우선한다.
{: .prompt-info }
