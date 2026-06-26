# pr-review-finding-validator — receiving-code-review를 검증자로 고친 것

이 에이전트는 `superpowers:receiving-code-review`를 출발점으로, **코드를 수정하지 않는 독립
검증자**로 고쳐 만들었다. 원본은 이미 다듬어진 검증 프롬프트라 그 자산을 살리고, 역할을 반전한
뒤, 우리 파이프라인에 필요한 결정 모델을 얹었다. 아래는 그 변형을 **역할 반전 → 가져온 것 →
더한 것 → 버린 것** 순으로 정리한 것이다.

## 역할 반전 — 구현자에서 검증자로

원본 receiving-code-review는 **"리뷰 피드백을 받아 직접 고치는 구현자"** 시점이다.
이 에이전트는 정반대로 **"코드를 수정하지 않는 독립 검증자"**다. 작성자 편향을 피하려고
별도 서브에이전트로 띄우고, 코드를 직접 로드해 판정만 한다. 이 반전이 나머지 변경의 출발점이다.

| | receiving-code-review | pr-review-finding-validator |
|---|---|---|
| 주체 | 메인 에이전트 본인 | fresh 서브에이전트 (편향 차단) |
| 대상 | 사람/리뷰어 피드백 | `pr-review-toolkit:review-pr` findings |
| 권한 | 구현(Edit/Write) | read-only (Read/Grep/Glob/Bash 조회만) |
| 종료 | 수용/반박 + 구현 | 구조화된 verdict (구현 없음) |

## 그대로 가져온 검증 원칙

원본의 검증 원칙은 이미 검증된 자산이라 최대한 보존했다 — 빈 동의를 막고, 코드 대조를 강제하고,
외부 제안에 회의적이게 한다. 검증자 역할에 맞춰 문구만 손봤다.

| 에이전트 섹션 | 원본 대응 | 어떻게 고쳤나 |
|---|---|---|
| Overview / Core principle | Overview | "Verify before **judging**"(원본은 implementing), 사회적 편안함보다 기술적 정확성은 그대로 |
| The Validation Pattern | The Response Pattern | READ/UNDERSTAND/VERIFY/EVALUATE 보존, **IMPLEMENT 제거**하고 JUDGE로 |
| Forbidden Responses | Forbidden Responses | "You're absolutely right!" 등 빈 동의 금지 그대로, 구현 관련 항목만 검증용으로 |
| Verifying Each Finding (5점 체크리스트) | Source-Specific Handling → External Reviewers | 거의 그대로. "Be skeptical, but check carefully" + 기능 깨짐/현재 구현 이유/플랫폼/전체 맥락 |
| "Cannot verify without [X]" | 동 섹션의 can't-verify 가이드 | "proceed?" 대신 "추측 금지, rationale에 한계 명시" |
| YAGNI Check | YAGNI Check for "Professional" Features | grep으로 사용처 확인 → 안 쓰이면 FP, 그대로 |
| When a Finding Is a False Positive | When To Push Back | "push back"을 "label FP"로. **FP 경계 문단은 새로 더함** (아래 결정 모델) |
| The Bottom Line | The Bottom Line | "suggestions to evaluate, not orders to follow" 그대로 |

## 더한 것 — 결정 모델

원본은 "피드백을 고칠지 / 반박할지"까지만 다루지, finding을 **자동으로 고칠지** 가르는 로직이
없다. 그래서 결정 모델을 새로 얹었다. 이것이 원본과의 가장 큰 차이다.

review-pr가 낸 finding마다 중첩된 세 질문으로 판정하고, 그 판정이 `pr-review`의 Fix·Comment
단계 동작을 결정한다.

### 세 질문

1. **실재하는가?** — `verdict`: `TP` 또는 `FP`. FP면 사유만 기록하고 종료한다.
2. **(TP) 사람의 정책·설계·계약 판단 없이 올바른 fix를 결정·적용할 수 있는가?** — `auto_fixable`: `Y` 또는 `N`.
   - `Y` — fix가 일의적·기계적이다. 자동으로 수정한다. **파일 위치와 규모는 따지지 않는다.**
   - `N` — fix에 사람의 판단이 필요하다. 추측하지 않고 상세 브리프와 함께 기록(noted)한다.
3. **(`auto_fixable=N`일 때만) 머지를 막는가?** — `blocks_merge`: `Y` 또는 `N`. 미수정 항목 중
   무엇을 머지 전에 반드시 해결해야 하는지를 사람에게 표시한다.

### 산출

- **TP + `auto_fixable=Y`** → 자동 수정. `Fixed in <sha>`.
- **TP + `auto_fixable=N`** → 코드 변경 없이 상세 브리프(issue / decision_needed / options)와 `blocks_merge`를 기록.
- **FP** → 코드 변경 없이 사유만 기록.

위와 함께 더한 것:
- **Output Format 스키마** — verdict + (TP면) auto_fixable, noted엔 decision_brief를 구조화 출력.
- **FP 경계 문단** (When a Finding Is a False Positive 섹션 내) — "FP는 코드가 실제로 문제없을 때만; 실재 결함은 fix가 범위 밖이거나 계약 미정이어도 `auto_fixable: N`(noted)".

### auto_fixable이 유일한 기준인 이유

자동 수정 여부는 **`auto_fixable` 하나로** 가른다 — 즉 "사람의 정책·설계 판단 없이 올바른 fix를
정할 수 있는가". 그 외 속성은 기준에 넣지 않으며, 이유는 다음과 같다.

- **머지 차단(`blocks_merge`)은 기준이 아니다.** 자동으로 고쳐지면 그 항목은 더 이상 머지를
  막지 않으므로, "고칠지"와 머지 차단 여부는 무관하다. 그래서 `blocks_merge`는 고치지 못한
  항목의 **우선순위 표시**로만 쓴다.
- **파일 위치·변경 규모도 기준이 아니다.** review-pr의 finding은 PR diff에서 나오므로 본래
  PR과 관련돼 있고, PR의 코드가 의존하는 결함은 그 뿌리가 다른 파일에 있어도 고치는 것이 옳다.
  규모가 크다는 이유로 막지 않으며, 수정이 안전한지는 Verify 단계가 검증한다.
- **추측하지 않는다.** 올바른 fix가 일의적이지 않으면(`auto_fixable: N`) 자동으로 손대지 않고
  사람에게 넘긴다.

## 버린 것

| 원본 섹션 | 버린 이유 |
|---|---|
| Implementation Order | 검증자는 구현하지 않음 |
| Acknowledging Correct Feedback / Gracefully Correcting Your Pushback | 구현자의 사회적 응답 가이드 — 검증자엔 무관 |
| GitHub Thread Replies | 인라인 댓글 응답 기법 — 검증자엔 무관 |
| "From your human partner"(신뢰) vs External 구분 | 검증자는 **모든** finding을 외부=검증 대상으로 동일 취급 |

## 유지보수 노트

가져온 검증 원칙과 더한 결정 모델을 섞지 않고 구분해두면, 원본 receiving-code-review가
업데이트될 때 **검증 원칙은 원본을 따라 갱신**하고 **결정 모델은 우리가 자체 관리**하면 된다는 게
분명해진다.

## 관련 파일

- [agents/pr-review-finding-validator.md](../agents/pr-review-finding-validator.md) — 판정을 산출하는 검증 에이전트 본문
- [skills/pr-review/SKILL.md](../skills/pr-review/SKILL.md) — Validate 단계가 이 에이전트를 dispatch, Fix·Comment 단계가 산출을 소비
