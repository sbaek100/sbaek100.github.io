---
title: 12강. 하네스 심화 II - 자동 검증과 해커톤 실전
date: 2027-05-17 09:00:00 +0900
categories:
  - 1.응용강의
  - AI와 함께하는 프로젝트
tags:
  - 하네스
  - Hooks
  - 해커톤
  - 스프린트
  - 심화
pin:
mermaid: false
---

> **학습 목표**
> 1. Hooks의 현행 규격(표준입력 JSON, exit 2)을 이해하고 훅 3종을 직접 만들 수 있다.
> 2. 실전 사례(FeedbackPulse)에서 "코드가 아니라 문서를 고친다"는 원칙을 확인한다.
> 3. 전체 워크플로우를 정리하고, 스프린트와 결합한 해커톤 수행 계획을 세울 수 있다.
{: .prompt-info }

## 1. Layer 4 — Hooks의 현재 상태 확인

### 1.1 문서와 실물의 차이

Notion 튜토리얼은 자동 검증 장치로 훅 3종을 설명한다.

| Hook | 기능 | 문서상 파일 |
|---|---|---|
| TDD Guard | 테스트 없는 구현 파일 수정을 차단 | `scripts/hooks/tdd-guard.sh` |
| Dangerous Command Guard | `rm -rf`, force push 등 위험 명령 차단 | `scripts/hooks/dangerous-cmd-guard.sh` |
| Circuit Breaker | 같은 에러가 60초 안에 5회 반복되면 경고 | `scripts/hooks/circuit-breaker.sh` |

그런데 실제 저장소를 열어 보면 **`scripts/hooks/` 폴더 자체가 존재하지 않는다**. 저장소의 커밋 기록에는 `Revert "feat: hooks 스크립트 추가..."`라는 메시지가 남아 있다 — 한때 훅 파일들을 추가했다가 되돌린 이력이 있으며, 현재의 `.claude/settings.json`은 그 이전의 단순한 버전이다.

이는 원본 자료의 결함이라기보다, 오픈소스 저장소가 변해 가는 과정의 실례이다. 이번 강의에서 할 일은 **Notion이 설명하는 목표 구조를 현행 규격에 맞춰 직접 완성하는 것**이다.

### 1.2 현재 settings.json의 문제 — 원문 분석

저장소에 실제로 들어 있는 훅 설정의 핵심 부분은 다음과 같다.

```json
"PreToolUse": [{
  "matcher": "Bash",
  "hooks": [{
    "type": "command",
    "command": "if echo \"$CLAUDE_TOOL_INPUT\" | grep -qE 'rm\\s+-rf|...'; then echo 'BLOCKED...' >&2; exit 1; fi"
  }]
}]
```

이 코드에는 두 가지 결함이 있다.

| 결함 | 위치 | 문제 |
|---|---|---|
| 구식 환경변수 | `$CLAUDE_TOOL_INPUT` | 현행 규격은 **표준입력 JSON**이다. 이 변수는 비어 있어 grep이 항상 실패한다 |
| 잘못된 종료 코드 | `exit 1` | 현행 규격에서 차단은 **`exit 2`**이다. `exit 1`은 "오류지만 계속 진행"으로 처리된다 |

직접 확인해 보면 결함이 드러난다.

```bash
echo '{"tool_input":{"command":"rm -rf test/"}}' | \
  bash -c 'if echo "$CLAUDE_TOOL_INPUT" | grep -qE "rm\s+-rf"; then echo BLOCKED; exit 1; fi'; echo "종료 코드: $?"
```

`$CLAUDE_TOOL_INPUT`이 비어 있으므로 아무것도 차단하지 못한 채 **종료 코드 0**이 나온다 — 위험한 명령이 그대로 통과한다는 뜻이다.

---

## 2. 훅 3종 직접 만들기

각 훅은 AI에게 작성시키되, 요구 조건을 정확히 지시하고 결과를 직접 검증한다.

### 2.1 Dangerous Command Guard

**지시문**:

```text
scripts/hooks/dangerous-cmd-guard.sh 를 만들어 줘.
- 트리거: PreToolUse, matcher는 Bash
- 표준입력 JSON에서 .tool_input.command 를 읽는다
- rm -rf, rm -r, git push --force, git reset --hard 가 포함되면 차단(exit 2)
- rm 은 -r 이나 -f 가 붙은 것만 막는다. rm sample.pcap 같은
  단일 파일 삭제까지 막으면 안 된다
- 차단 사유를 표준 오류(>&2)로 출력하고, 어떤 패턴에 걸렸는지 알려 줘
- JSON 파싱은 python3 + try/except. chmod +x 까지 해 줘
```

**검증** — 다섯 가지 명령을 넣어 차단(2)·통과(0)를 확인한다.

```bash
for cmd in "rm -rf build/" "rm -fr build/" "rm -r build/" "rm sample.pcap" "git push --force"; do
  printf '{"tool_input":{"command":"%s"}}' "$cmd" | bash scripts/hooks/dangerous-cmd-guard.sh >/dev/null 2>&1
  echo "$cmd → 종료 코드 $?"
done
```

`rm sample.pcap`만 `0`, 나머지는 `2`가 나와야 한다. 참고로 원본 정규식은 `rm\s+-rf` 하나만 검사하므로 `rm -fr`(순서 변경)이나 `rm -r`은 통과한다 — 위험한 명령의 변형까지 고려해야 한다는 교훈이다.

### 2.2 TDD Guard

**설계**: 수정하려는 파일이 `src/` 아래 구현 파일인데 대응하는 `tests/test_{모듈}.py`가 없으면 차단한다. `__init__.py`와 문서 파일은 검사에서 제외한다.

**지시문**:

```text
scripts/hooks/tdd-guard.sh 를 만들어 줘.
- 트리거: PreToolUse, matcher는 Write|Edit
- 표준입력 JSON의 .tool_input.file_path 를 읽는다
- src/{패키지}/*.py 인데 대응하는 tests/test_{모듈}.py 가 없으면 차단(exit 2)
- __init__.py, docs/*.md, CLAUDE.md 는 검사하지 않는다
- 차단 사유는 표준 오류로. chmod +x 까지.
```

**검증**:

```bash
echo '{"tool_input":{"file_path":"/project/src/pcapeek/newmod.py"}}' \
  | bash scripts/hooks/tdd-guard.sh; echo "종료 코드: $?"
```

`tests/test_newmod.py`가 없으므로 `2`가 나와야 하며, 테스트 파일 생성 후 재실행하면 `0`으로 바뀌어야 한다.

### 2.3 Circuit Breaker

앞의 두 훅과 달리, 이 훅은 **시간에 걸친 반복을 기억**해야 하므로 상태 파일이 필요하다.

**설계**:

```text
트리거: PostToolUseFailure (도구 실행 실패 시)
상태 파일: .claude/.circuit-breaker.json (실패 시각과 메시지 기록)
로직: 60초 이내 기록만 유지 → 같은 도구가 5회 이상 실패하면 경고 출력
주의: 차단(exit 2)이 아니라 경고만 한다 (exit 0 유지)
```

차단이 아니라 경고인 이유 — TDD Guard와 Dangerous Command Guard는 행동 하나하나가 위험함을 사전에 판정할 수 있으나, Circuit Breaker가 감지하는 것은 "같은 실수의 반복"이라는 패턴이다. 개별 행동은 위험하지 않을 수 있으므로 차단하면 AI가 아예 움직이지 못하게 된다. 경고로 알리고 판단은 다음 재시도에 맡긴다. 상태 파일은 `.gitignore`에 추가하여 커밋되지 않게 한다.

### 2.4 통합 등록

```text
.claude/settings.json 을 갱신해 줘.
- PreToolUse(Write|Edit) → scripts/hooks/tdd-guard.sh
- PreToolUse(Bash) → scripts/hooks/dangerous-cmd-guard.sh
- PostToolUseFailure → scripts/hooks/circuit-breaker.sh
- Stop → pytest 실행 (시험이 없으면 통과)
- 경로는 ${CLAUDE_PROJECT_DIR} 사용
- 갱신 후 각 훅에 표준입력 JSON을 넣어 차단/통과 여부를 표로 보여 줘
```

---

## 3. 실전 사례 — FeedbackPulse

원저자는 영상 후반부(18:56~)에서 FeedbackPulse라는 프로젝트를 라이브로 만들며 두 가지 문제를 겪었고, 둘 다 **코드가 아니라 문서를 고쳐** 해결하였다.

| 문제 | 원인 | 해결 |
|---|---|---|
| UI 품질이 낮다 | UI 디자인 가이드가 없었음 | `docs/UI_GUIDE.md` 추가 (안티슬롭 패턴 포함) |
| API 응답 파싱 에러 | JSON 응답의 코드블록 감싸기 미처리 | ADR-009 추가: 코드블록 제거 규칙 |

원저자의 결론은 다음과 같다 — "같은 프레임워크, 같은 구조에서 docs에 가이드 하나를 추가했을 뿐인데 결과물이 확연히 달라졌다."

두 번째 사례를 자세히 보면 ADR의 용도가 드러난다. AI API에 JSON을 요청하면 가끔 응답이 마크다운 코드블록(```json ... ```)으로 감싸져 와서 그대로 파싱하면 오류가 난다. 이를 코드 곳곳에서 예외 처리로 땜질하는 대신, ADR에 규칙으로 기록하였다.

```text
ADR-009: API 응답 파싱
결정: 응답 문자열에서 코드블록 마커를 먼저 제거한 뒤 파싱한다
이유: API가 종종 JSON을 마크다운 코드블록으로 감싸서 반환한다
트레이드오프: 응답 형식이 근본적으로 바뀌면 이 규칙도 재작성해야 한다
```

이렇게 기록하면 이후 모든 step에서 AI가 이 규칙을 가드레일로 함께 받는다. 한 곳의 코드를 고치는 것이 아니라, **앞으로 작성될 모든 코드가 같은 실수를 하지 않게 만드는 것**이다.

### `/review` 명령

`.claude/commands/review.md`에 정의된 `/review`는 다음 네 가지를 자동 점검한다 — ARCHITECTURE.md의 폴더 구조 준수, ADR의 기술 스택 준수, 테스트 작성 여부, CLAUDE.md의 CRITICAL 규칙 준수. 완성한 프로젝트에서 실행하면 점검 결과가 표로 출력되고, 위반 시 수정 방안까지 제시된다.

---

## 4. 전체 워크플로우 정리

원저자가 정리한 하네스 기반 프로젝트의 전체 흐름은 다섯 단계이며, 본 과정의 강의 구성과 다음과 같이 대응된다.

| 단계 | 수행 주체 | 내용 | 본 과정의 해당 부분 |
|---|---|---|---|
| 뼈대 | 사람 | 프레임워크 클론 | 10강 |
| 기획 | 사람 + AI | 비즈니스 모델, 요구사항, 화면 설계 → `docs/` | 3~8강 |
| 하네스 설계 | 사람 + AI | CLAUDE.md 규칙 + Hooks | 10강·12강 |
| 실행 | AI | `/harness` → 자동 실행 | 10강·11강 |
| 리뷰 | 사람 + AI | 결과 확인 → docs 보강 → 재실행 | 10강 7절·12강 |

---

## 5. 해커톤 실전 — 스프린트와 하네스의 결합

과정의 마무리로, 배운 것 전부를 해커톤(또는 새 팀 프로젝트) 상황에 적용하는 절차를 정리한다. 1강의 스프린트가 **시간 관리 틀**이라면 하네스는 **품질 관리 틀**이며, 둘을 결합하면 짧은 기간에도 완성도 있는 결과물이 나온다.

### 5.1 해커톤 24시간 운영 예시

| 스프린트 | 시간 | 활동 | 산출물 |
|---|---|---|---|
| Sprint 0 | 0~3시간 | 팀 구성, 문제 정의, BMC 축약판(고객·가치·Won't만) | 한 장짜리 기획 |
| Sprint 1 | 3~6시간 | 요구사항(Must만)·와이어프레임, 하네스 클론·docs 작성 | docs/ 완성 |
| Sprint 2 | 6~14시간 | `/harness` 자동 실행, 결과 확인, 문서 보강 후 재실행 | 동작하는 MVP |
| Sprint 3 | 14~20시간 | 검증(테스트·교차 확인), 데모 시나리오 연습 | 검증된 결과물 |
| Sprint 4 | 20~24시간 | 발표 자료(문제 Hook·한 줄 요약·구조도·차별점·수치) | 발표 |

각 스프린트 종료 시 데일리 스크럼 형식의 짧은 점검(무엇을 했는가·다음은 무엇인가·막힌 것은 무엇인가)을 수행한다.

### 5.2 해커톤 체크리스트

```text
[ ] 문제 진술문에 해결책이 섞여 있지 않다
[ ] Won't 목록에 이유가 붙어 있다 (범위 확장 차단)
[ ] 와이어프레임이 docs/OUTPUT_GUIDE.md 에 들어가 있다
[ ] 훅이 실제로 차단하는지 종료 코드로 확인했다 (2=차단, 0=통과)
[ ] 결과물을 표준 도구로 교차 검증했다
[ ] 발표 5요소(Hook·한줄요약·구조도·차별점·수치)가 준비되었다
[ ] 팀원 전원이 시스템 구조를 설명할 수 있다
```

### 5.3 다음 프로젝트로의 이식

하네스의 배관(Layer 3·4)은 재사용하고 문서(Layer 1·2)만 새로 쓰면 된다. 새 폴더에서 AI에게 다음과 같이 지시한다.

```text
~/pcapeek 의 .claude 와 scripts 를 이 폴더로 복사해 줘.
그 안의 'pcapeek' 이라는 이름을 전부 '{새이름}' 으로 바꾸고,
바꾼 뒤 grep 으로 옛 이름이 안 남았는지 확인해 줘.
docs/ 와 CLAUDE.md 는 빈칸 템플릿으로만 만들어 줘.
```

이름 치환 후 반드시 확인시킨다 — 치환이 실패하면 훅이 조용히 무력화된다.

---

## 실습 (종합)

1. 훅 3종을 2절의 지시문으로 작성시키고, 각각의 검증 명령으로 차단·통과 여부를 확인하라.
2. 자신의 프로젝트에서 겪은(또는 겪을 법한) 문제 하나를 ADR 형식(결정·이유·트레이드오프)으로 작성하라.
3. 5.1절의 24시간 운영표를 자신의 팀 상황(팀원 수, 주제)에 맞게 수정하여 해커톤 수행 계획을 완성하라.

---

## 챕터를 마치며

하네스 엔지니어링 챕터를 통해 다음의 전 과정을 수행하였다.

```text
개발 환경 → 하네스 구축 → 자동 구현 → 검증
→ 하네스 내부 구조 이해 → 자동 검증 장치 → 해커톤 적용
```

코드를 직접 작성한 시간은 거의 없었으나, **무엇을 만들지 정의하고, AI에게 정확히 지시하고, 결과를 검증하는** 전 주기를 경험하였다. 이것이 바이브 코딩 시대의 개발 역량이며, 어떤 주제의 해커톤에서도 재사용할 수 있는 틀이다.

---

## 다음 강의

13강에서는 과정의 마지막 단계인 **최종 발표**를 다룬다. 작품 평가의 6개 요소(완성도·독창성·실용성·난이도·기획 및 참여도·기타)를 기준으로 발표 자료를 구성하는 법, 시연 대본과 실패 대비, 복장·어조·질의응답까지 발표의 전 과정을 학습한다.

## 출처

| 자료 | 확인 내용 |
|---|---|
| [settings.json 원문](https://github.com/jha0313/harness_framework/blob/main/.claude/settings.json) | 현재 훅 코드와 Revert 커밋 이력 |
| [Notion 튜토리얼](https://raspy-roll-970.notion.site/340f7725c9d98176b68bd31c823c7540) | 훅 3종 설명, FeedbackPulse 사례, 전체 워크플로우 |
| [공식 Hooks 문서](https://code.claude.com/docs/en/hooks) | 현행 규격 (표준입력 JSON, exit 2) |

`scripts/hooks/*.sh` 3종은 본 강의 시점의 저장소에 존재하지 않으며, Notion의 설명을 근거로 본 강의에서 직접 설계·작성한 것이다. 원저자가 이후 저장소를 갱신하면 실물과 다를 수 있으므로 그 경우 실물을 기준으로 삼는다.
