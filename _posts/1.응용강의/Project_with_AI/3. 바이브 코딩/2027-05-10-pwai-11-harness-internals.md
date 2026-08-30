---
title: 11강. 하네스 심화 I - 4개 레이어와 실행 엔진
date: 2027-05-10 09:00:00 +0900
categories:
  - 1.응용강의
  - AI와 함께하는 프로젝트
tags:
  - 하네스
  - ClaudeCode
  - 심화
  - 헤드리스모드
  - PRD
pin:
mermaid: false
---

> **학습 목표**
> 1. 하네스 프레임워크의 4개 레이어 구조를 원본 자료 기준으로 설명할 수 있다.
> 2. `/harness` 명령의 워크플로우(A~E)와 phases 파일 구조를 이해한다.
> 3. 실행 엔진(`execute.py`)의 핵심 동작 — 가드레일 주입, 헤드리스 실행, 재시도 — 를 코드 수준에서 설명할 수 있다.
{: .prompt-info }

## 0. 심화 과정의 목적

앞선 구현 강의에서는 완성 속도를 우선하여, 원본 자료의 세부 구조는 AI에게 맡기고 넘어갔다. 심화 과정(11~12강)에서는 원본 자료 세 가지를 직접 열어 확인한다.

| 자료 | 내용 |
|---|---|
| [원저자 영상](https://www.youtube.com/watch?v=AQOvNx87Urs) (실밸개발자, 41:38) | 하네스를 처음부터 만드는 과정의 시연 |
| [Notion 튜토리얼](https://raspy-roll-970.notion.site/340f7725c9d98176b68bd31c823c7540) | 영상 내용을 글로 정리한 공식 안내서 |
| [GitHub 저장소](https://github.com/jha0313/harness_framework) | 실제로 클론하여 쓰는 원본 코드 |

본 강의의 모든 인용은 위 자료를 실제로 열어 확인한 내용이다.

---

## 1. 4개 레이어 구조

Notion 튜토리얼은 하네스를 4개 레이어로 설명한다.

| 레이어 | 구성 | 역할 |
|---|---|---|
| **Layer 1** | `docs/` (PRD·ARCHITECTURE·ADR·UI_GUIDE) | 프로젝트의 두뇌 — 무엇을 왜 어떻게 만드는가 |
| **Layer 2** | `CLAUDE.md` | 프로젝트의 헌법 — AI가 가장 먼저 읽는 규칙 |
| **Layer 3** | `/harness` + `execute.py` | 실행 엔진 — 문서를 읽고 단계별로 자동 실행 |
| **Layer 4** | Hooks (`settings.json`) | 자동 검증 장치 — 규칙 위반을 기계적으로 차단 |

### 1.1 실제 저장소 확인 — 문서와 실물의 차이

저장소를 클론하여 확인한 실제 구조는 다음과 같다.

```text
harness_framework/
├── .claude/
│   ├── commands/harness.md, review.md
│   └── settings.json
├── docs/PRD.md, ARCHITECTURE.md, ADR.md, UI_GUIDE.md
├── scripts/execute.py, test_execute.py
├── .gitignore
└── CLAUDE.md
```

주의할 점이 두 가지 있다.

1. **`phases/` 폴더는 저장소에 없다.** `/harness`를 처음 실행할 때 생성된다.
2. **`package.json`이 없다.** Notion의 빠른 시작 가이드는 `npm install`부터 시작하지만, 저장소에는 해당 파일이 없어 그 명령은 실패한다. 이 저장소는 완성된 앱이 아니라 **문서 템플릿과 실행 엔진만 담긴 뼈대**이며, 실제 앱 코드는 `/harness` 실행 시 AI가 생성한다.

문서(Notion)와 실물(저장소)이 다를 수 있다는 것 자체가 오픈소스를 다룰 때의 중요한 교훈이다 — **항상 실물을 직접 확인한다.**

### 1.2 Layer 1 — docs/의 실제 템플릿

`docs/PRD.md`의 원문 구조는 다음과 같다.

```markdown
PRD: {프로젝트명}
목표: {해결하려는 문제를 한 줄로}
사용자: {누가 쓰는지}
핵심 기능: {기능 1~3}
MVP 제외 사항: {안 만들 것 1~3}
디자인: {디자인 방향}
```

기획 챕터에서 작성한 PRD의 **축약판**임을 알 수 있다 — 목표=개요·목표, 사용자=대상 사용자, 핵심 기능=In Scope, MVP 제외 사항=Out of Scope에 대응하며, 성공 기준·시나리오·제약사항 같은 항목이 빠진 최소 형태이다. 원저자의 5항목이 실무 PRD의 뼈대만 남긴 것이라는 사실은, 역으로 **어떤 항목까지 채우느냐가 프로젝트의 규모와 목적에 따라 조절 가능**함을 보여 준다.

원저자는 "MVP 제외 사항"에 대해 "이걸 안 써놓으면 AI가 '이 기능도 추가할까요?' 하면서 범위가 끝없이 늘어난다"라고 설명한다. `docs/ADR.md`(Architecture Decision Records)는 "무엇을 선택했고, 왜 선택했고, 무엇을 포기했는지"를 기록하며, **트레이드오프가 핵심**이다 — 포기한 것이 없으면 결정이 아니다.

`docs/UI_GUIDE.md`에는 "AI 슬롭(slop) 안티패턴" 표가 포함되어 있다. AI가 만든 화면임을 드러내는 상투적 패턴들을 사전에 금지하는 목록으로, 일부를 옮기면 다음과 같다.

```text
금지 사항                          이유
backdrop-filter: blur()           AI 템플릿의 가장 흔한 징후
gradient-text                     AI 생성 랜딩페이지의 1번 특징
"Powered by AI" 배지               기능이 아니라 장식
보라/인디고 브랜드 색상               "AI = 보라색" 클리셰
```

화면 설계 강의에서 작성한 "하지 말 것" 목록과 같은 발상이며, 웹 프론트엔드 맥락에서 더 세밀하게 발전시킨 것이다.

### 1.3 Layer 2 — CLAUDE.md의 두 가지 장치

`CLAUDE.md` 원문에서 주목할 장치는 두 가지이다.

1. **`CRITICAL` 키워드** — AI가 우선순위 신호로 인식하여 일반 규칙보다 강하게 따른다.
2. **TDD 규칙** — "새 기능 구현 시 반드시 테스트를 먼저 작성"이라는 한 줄만으로도 코드 품질이 크게 오른다.

구현 강의의 지시문에서 `CRITICAL:` 접두사와 TDD 규칙을 사용한 것은 이 원본 설계를 그대로 따른 것이다.

---

## 2. Layer 3 — `/harness` 명령과 phases 구조

### 2.1 워크플로우 A~E

`.claude/commands/harness.md` 원문은 `/harness` 실행 시의 절차를 다섯 단계로 정의한다.

| 단계 | 이름 | 내용 |
|---|---|---|
| A | 탐색 | `docs/` 문서를 읽고 기획·설계 의도 파악 |
| B | 논의 | 결정이 필요한 사항을 사용자와 논의 |
| C | Step 설계 | 여러 step으로 나눈 구현 계획 초안 작성, 피드백 요청 |
| D | 파일 생성 | 승인 시 `phases/` 아래 파일 생성 |
| E | 실행 | `execute.py` 실행 |

구현 강의에서 `/harness` 입력 후 "5단계로 나누겠습니다. 이대로 만들까요?"라는 대화가 나온 것이 바로 C단계이다.

### 2.2 Step 설계 원칙 (원문 요약)

```text
- Scope 최소화     : 하나의 step은 하나의 레이어/모듈만 다룬다
- 자기완결성       : 각 step은 독립된 세션에서 실행된다.
                    "이전 대화에서 논의한 바와 같이" 같은 외부 참조 금지
- AC는 실행 가능한 커맨드 : "동작해야 한다"가 아니라 실제 명령(pytest 등)
- 주의사항은 구체적으로  : "조심해라" 대신 "X를 하지 마라. 이유: Y"
```

**자기완결성**이 특히 중요하다. 각 step은 완전히 새로운 AI 세션에서 실행되므로 이전 대화 기록이 전혀 없다. 필요한 정보는 전부 step 파일 안에 있어야 한다.

### 2.3 phases 파일 구조

`/harness`의 D단계가 생성하는 파일은 세 종류이다.

```json
// phases/index.json — 전체 현황
{ "phases": [ { "dir": "0-mvp", "status": "pending" } ] }
```

```json
// phases/0-mvp/index.json — 작업 상세
{
  "project": "pcapeek",
  "phase": "0-mvp",
  "steps": [
    { "step": 0, "name": "project-setup", "status": "pending" },
    { "step": 1, "name": "reader", "status": "pending" }
  ]
}
```

```markdown
<!-- phases/0-mvp/step1.md — step마다 하나 -->
# Step 1: reader
## 읽어야 할 파일: /docs/ARCHITECTURE.md, /docs/ADR.md
## 작업: {구체적 구현 지시}
## Acceptance Criteria: pytest tests/test_reader.py
## 금지사항: {X를 하지 마라. 이유: Y}
```

상태 값(`pending`·`completed`·`error`·`blocked`)은 사람이 아니라 실행 엔진이 자동으로 갱신한다.

---

## 3. 실행 엔진 — `execute.py` 코드 분석

`scripts/execute.py`(417줄)는 `StepExecutor` 클래스 하나로 구성된다. 전체 흐름은 `run()` 메서드 여섯 줄에 압축되어 있다.

```python
def run(self):
    self._print_header()
    self._check_blockers()      # 이전에 error/blocked 된 step이 있으면 중단
    self._checkout_branch()     # feat-{phase} 브랜치로 전환
    guardrails = self._load_guardrails()   # CLAUDE.md + docs/*.md 읽기
    self._ensure_created_at()
    self._execute_all_steps(guardrails)    # pending step을 순서대로 실행
    self._finalize()
```

### 3.1 가드레일 주입 — 하네스의 정체

```python
def _load_guardrails(self) -> str:
    sections = []
    claude_md = ROOT / "CLAUDE.md"
    if claude_md.exists():
        sections.append(f"## 프로젝트 규칙 (CLAUDE.md)\n\n{claude_md.read_text()}")
    docs_dir = ROOT / "docs"
    if docs_dir.is_dir():
        for doc in sorted(docs_dir.glob("*.md")):
            sections.append(f"## {doc.stem}\n\n{doc.read_text()}")
    return "\n\n---\n\n".join(sections)
```

이 함수가 하는 일은 단순하다 — `CLAUDE.md`와 `docs/`의 모든 문서를 **통째로 읽어 이어 붙인다**. 즉 가드레일의 정체는 "매 step마다 모든 문서를 프롬프트 앞부분에 다시 붙여 넣는 것"이다. 마법이 아니다. 문서가 얕으면 이 함수가 주입하는 내용도 얕아지므로, "하네스에 무엇을 넣느냐가 곧 결과의 품질"이라는 원저자의 명제가 코드 한 함수로 구현되어 있는 셈이다.

### 3.2 헤드리스 실행

```python
result = subprocess.run(
    ["claude", "-p", "--dangerously-skip-permissions",
     "--output-format", "json", prompt],
    cwd=self._root, capture_output=True, text=True, timeout=1800,
)
```

| 모드 | 방식 | 용도 |
|---|---|---|
| 대화형 모드 (`claude`) | 사람이 채팅으로 주고받으며 매번 승인 | 일상적 사용 |
| **헤드리스 모드 (`claude -p`)** | 프롬프트를 넘기면 끝까지 자동 실행 | 자동화 |

step마다 **새로운 AI 세션**이 호출된다. 이전 step의 세션은 종료되고, 다음 step은 백지 상태의 AI가 가드레일 + step 파일을 받아 시작한다. 각 step의 작업 범위가 문서로 제한되어 있으므로 AI가 범위 밖의 일을 하지 않는다.

> `--dangerously-skip-permissions` 플래그는 이름 그대로 위험하다 — 파일 삭제·수정 등 모든 작업의 승인 절차를 건너뛴다. 실행 엔진이 이 플래그를 쓸 수 있는 이유는, 각 step의 범위가 문서로 제한되고 **훅(12강)이 위험한 명령을 대신 차단하는 것을 전제**하기 때문이다. **훅 없이 이 플래그만 따로 사용해서는 안 된다.**
{: .prompt-danger }

### 3.3 재시도와 자가 교정

실행 엔진은 step 실패 시 **최대 3회**(`MAX_RETRIES = 3`) 재시도하며, 실패할 때마다 이전 에러 메시지를 다음 프롬프트에 포함시켜 같은 실수를 반복하지 않게 한다. 3회 모두 실패하면 `error` 상태로 기록하고 중단하며, API 키 입력처럼 사람의 개입이 필요한 경우는 `blocked` 상태로 즉시 중단한다.

### 3.4 2단계 커밋

코드 변경(`feat(...)`)과 진행 상태 기록(`chore(...)`)을 커밋 두 개로 분리한다. 이렇게 하면 `git log`에서 실제 기능 변경과 상태 파일 갱신이 섞이지 않는다.

### 3.5 실행 결과 예시 (Notion 튜토리얼 원문)

```text
python3 scripts/execute.py mvp

==================================================
  Harness Executor
  Task: mvp | Phases: 5 | Pending: 5
==================================================
  ✓ Phase 1: 프로젝트-초기화 [180s]
  ✓ Phase 2: 타입-+-유틸리티 [300s]
  ✓ Phase 3: api-라우트 [240s]
  ✓ Phase 4: ui-컴포넌트 [300s]
  ✓ Phase 5: 메인-페이지-통합 [150s]
==================================================
```

5단계 합계 1,170초, 약 19.5분이다. 구현 강의에서 언급한 "17분"과 같은 규모임을 확인할 수 있다.

---

## 실습

1. 저장소를 클론하여(또는 구현 강의에서 만든 프로젝트에서) `docs/PRD.md`와 `.claude/commands/harness.md`를 직접 열어 원문을 확인하라.
2. 기획 단계에서 완성한 자신의 PRD를 원본 템플릿의 5개 항목(목표·사용자·핵심 기능·MVP 제외 사항·디자인)으로 **압축**하라. 무엇을 버렸고 왜 버려도 되는지(또는 버리면 안 되는지) 한 줄씩 기록한다.
3. pcapeek의 step 하나를 골라 2.3절의 step 파일 형식(읽어야 할 파일·작업·AC·금지사항)으로 직접 작성하라. AC는 반드시 실행 가능한 명령으로 적는다.

---

## 핵심 용어 정리

| 용어 | 설명 |
|---|---|
| 레이어 | 하네스를 구성하는 4개 층 (docs / CLAUDE.md / 실행엔진 / Hooks) |
| 가드레일 주입 | 매 step마다 모든 문서를 프롬프트에 포함시키는 것 |
| 헤드리스 모드 | `claude -p` — 사람 없이 자동 실행되는 모드 |
| 자기완결성 | 각 step 파일이 독립 세션에서 실행 가능하도록 완결된 것 |
| AC | Acceptance Criteria — 실행 가능한 명령으로 적는 완료 판정 기준 |

---

## 다음 강의

12강에서는 Layer 4(Hooks)를 다룬다. Notion이 설명하는 훅 3종이 실제 저장소에는 존재하지 않는다는 사실을 확인하고, 현행 규격에 맞춰 직접 만들어 본다. 마지막으로 전체 워크플로우를 정리하고 해커톤 실전 체크리스트로 과정을 마무리한다.

## 출처

| 자료 | 확인 내용 |
|---|---|
| [harness.md 원문](https://github.com/jha0313/harness_framework/blob/main/.claude/commands/harness.md) | 워크플로우 A~E, step 설계 원칙, phases 구조 |
| [execute.py 원문](https://github.com/jha0313/harness_framework/blob/main/scripts/execute.py) | 417줄 전체 — 가드레일·헤드리스·재시도·커밋 로직 |
| [Notion 튜토리얼](https://raspy-roll-970.notion.site/340f7725c9d98176b68bd31c823c7540) | 4레이어 설명, 실행 예시 원문 |

실행 예시(3.5절)는 Notion에 실린 원저자의 실행 결과를 옮긴 것이며, 본 강의 환경에서 재실행하지는 않았다.
