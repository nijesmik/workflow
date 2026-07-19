# pr-review-finding-validator — receiving-code-review를 검증자로 고친 것

이 에이전트(`pr-review-finding-validator`)는 **`pr-review` 파이프라인**(우리 스킬)의 검증 단계다. 자동
코드리뷰 *도구* `pr-review-toolkit:review-pr`(이하 **review-pr** — 파이프라인 `pr-review`와 다른 별개
의존성)가 낸 지적(이하 **finding**)을 받아, **코드를 수정하지 않고** 각 finding이 진짜인지 판정한다.
그 판정을 파이프라인의 Fix·Comment 단계가 소비한다.

이 프롬프트는 `superpowers:receiving-code-review`(리뷰 피드백을 받은 **구현자**가 어떻게 대응할지
규율하는 원본 프롬프트)를 출발점으로, **코드를 수정하지 않는 독립 검증자** 역할로 고쳐 만들었다.
원본은 이미 다듬어진 검증 프롬프트라 그 자산을 살리고, 역할을 반전한 뒤, 우리 파이프라인에 필요한
결정 모델을 얹었다. 아래는 그 변형을 **역할 반전 → 가져온 것 → 더한 것 → 버린 것** 순으로 정리한 것이다.

## 역할 반전 — 구현자에서 검증자로

원본 receiving-code-review는 **"리뷰 피드백을 받아 직접 고치는 구현자"** 시점이다.
이 에이전트는 정반대로 **"코드를 수정하지 않는 독립 검증자"**다. 작성자 편향을 피하려고
별도 서브에이전트로 띄우고, 코드를 직접 로드해 판정만 한다. 이 반전이 나머지 변경의 출발점이다.

| | receiving-code-review | pr-review-finding-validator |
|---|---|---|
| 주체 | 메인 에이전트 본인 | 새로 띄운 서브에이전트(이전 맥락 없음 → 편향 차단) |
| 대상 | 사람/리뷰어 피드백 | `pr-review-toolkit:review-pr` findings |
| 권한 | 구현(Edit/Write) | read-only (Read/Grep/Glob/Bash 조회만) |
| 종료 | 수용/반박 + 구현 | 구조화된 verdict (구현 없음) |

## 그대로 가져온 검증 원칙

원본의 검증 원칙은 이미 검증된 자산이라 최대한 보존했다 — 빈 동의를 막고, 코드 대조를 강제하고,
외부 제안에 회의적이게 한다. 검증자 역할에 맞춰 문구만 손봤다. (아래 표의 **"원본 대응" 열**은
receiving-code-review의 섹션명 — 원본을 아는 사람을 위한 추적용이고, 모르면 "어떻게 고쳤나" 열만
읽어도 된다.)

| 에이전트 섹션 | 원본 대응 | 어떻게 고쳤나 |
|---|---|---|
| Overview / Core principle | Overview | "Verify before **judging**"(원본은 implementing), 사회적 편안함보다 기술적 정확성은 그대로 |
| The Validation Pattern | The Response Pattern | READ/UNDERSTAND/VERIFY/EVALUATE 보존, **IMPLEMENT 제거**하고 JUDGE로 |
| Forbidden Responses | Forbidden Responses | "You're absolutely right!" 등 빈 동의 금지 그대로, "단정 말고 읽은 코드로 근거 대라" 한 줄 보강, 구현 관련 항목만 검증용으로 |
| Verifying Each Finding (5점 체크리스트) | Source-Specific Handling → External Reviewers | 거의 그대로. "Be skeptical, but check carefully" + 기능 깨짐/현재 구현 이유/플랫폼/전체 맥락. 단 "체크는 **결함 실재**를 의심할 근거이지 수정안 품질 평가가 아니다"라는 전제를 앞에 붙임 (원본은 구현자라 수정안 실행 여부 판단이 맞지만, 검증자의 판정 대상은 결함 실재) |
| "Cannot verify without [X]" | 동 섹션의 can't-verify 가이드 | 원문은 "사람에게 물어봐라"(investigate/ask/proceed). 비대화형이라 **`Unverified`로 표기해 코멘트로 사람에게 surface** — 불확실성을 사람에게 넘긴다는 원문 의도 유지 |
| Handling Unclear Feedback | Handling Unclear Feedback | finding 뜻 자체가 모호하면 일부 판정 말고 `Unverified`로 (can't-verify와 합쳐 처리) |
| YAGNI Check | YAGNI Check for "Professional" Features | grep으로 사용처 확인 → 안 쓰이면 FP, 그대로 |
| When a Finding Is a False Positive | When To Push Back | "push back"을 "label FP"로. 원본 6개 트리거(아키텍처/컨벤션 충돌 포함) 반영 — 단 "기능 깨짐" 트리거는 **실재 결함이 없을 때**로 좁힘(수정안이 나쁜 실재 결함은 FP가 아니라 TP `auto_fixable: N`). **FP 경계 문단은 새로 더함** (아래 결정 모델) |
| Examples | Real Examples | 판정 예시(FP / Unverified / TP-decision / TP-scope / TP-fix)로 번역 |
| The Bottom Line | The Bottom Line | "suggestions to evaluate, not orders to follow" 그대로 |

## 더한 것 — 결정 모델

원본은 "피드백을 고칠지 / 반박할지"까지만 다루지, finding을 **자동으로 고칠지** 가르는 로직이
없다. 그래서 결정 모델을 새로 얹었다. 이것이 원본과의 가장 큰 차이다.

review-pr가 낸 finding마다 중첩된 네 질문으로 판정하고, 그 판정이 `pr-review`의 Fix·Comment
단계 동작을 결정한다.

### 네 질문

1. **실재하는가?** — `verdict`: `TP`(실재 확인) / `FP`(문제없음 확인) / `Unverified`(코드만으로는 확정 불가 — 런타임·플랫폼·상태 의존, 또는 finding이 너무 모호해 평가 불가). FP는 사유만 기록하고 종료. Unverified는 cited 코드에 revert 테스트를 적용해 pre-existing이면 `pre_existing: Y`를 함께 기록하고 종료(결함 실재와 무관하게 코드의 base 존재 여부는 기계 판정 가능).
2. **(TP) 사람의 정책·설계·계약 판단 없이 올바른 fix를 결정·적용할 수 있는가?** — `auto_fixable`: `Y` 또는 `N`.
   - `Y` — fix가 일의적·기계적이다. 자동으로 수정한다. **파일 위치와 규모는 따지지 않는다.**
   - `N` — fix에 사람의 판단이 필요하다. 추측하지 않고 질문 3으로.
3. **(auto_fixable=N) 결함이 이 PR 이전부터 있었고 PR 범위 밖인가?** — **결함-정체성 기준 revert 테스트**: 이 PR의 diff를 되돌린 코드베이스에 동일 결함이 존재하는가(위치가 달라도). 아니면 → `deferred: decision`(이 PR의 문제). 맞아도 이 PR이 그 결함에 **의존**(새 코드의 정확성이 결함 동작에 좌우)하거나 **악화**(발생 조건을 새로 충족 — "최초 점화" — 하거나 영향 범위를 질적으로 확장; 단순 호출 추가는 제외)시키면 → `deferred: decision`으로 승격. 그 외 → `deferred: scope` — `blocks_merge`를 내보내지 않는다(부재 = N).
4. **(`deferred: decision`일 때만) 머지를 막는가?** — `blocks_merge`: `Y` 또는 `N`. 미해결 결정 중 무엇을 머지 전에 반드시 풀어야 하는지를 사람에게 표시한다.

### 산출

- **TP + `auto_fixable=Y`** → 자동 수정. `Fixed in <sha>`.
- **TP + `deferred: decision`** → 코드 변경 없이 상세 브리프(issue / impact / decision_needed / options / recommendation)와 `blocks_merge`를 기록. recommendation은 옵션 간 트레이드오프가 **코드 근거로 판정 가능할 때만** 내고, 순수 제품·정책 선택이면 `none` — validator는 코드에서 근거를 댈 수 있는 것만 말한다.
- **TP + `deferred: scope`** → 코드 변경 없이 `followup_brief`(issue / why_out_of_scope / impact / fix_sketch)를 기록. 이슈 등록 권고는 **의도적으로 없다** — 등록 가치는 팀 우선순위에 달린 가치 판단이라 코드 근거가 없고, "등록 권장" 앵커링은 후속 이슈 발산을 재생산한다. 등록 여부는 코멘트의 이슈 게이트에서 사람이 고른다.
- **FP** → 코드 변경 없이 사유만 기록.
- **Unverified** → 코드 변경 없이 "무엇이 없어 확정 못 했는지"를 기록. pre-existing이 아니면 코멘트의 **Decide in this PR**에 확인 요청으로, `pre_existing: Y`면 **Pre-existing defects — issue candidates**에 `unconfirmed` 라벨로 올라간다.

위와 함께 더한 것:
- **Output Format 스키마** — verdict + (TP면) auto_fixable + (N이면) deferred, decision엔 decision_brief, scope엔 followup_brief를 구조화 출력. 두 브리프 모두 "읽는 사람이 코드를 열지 않고 결정할 수 있게" 쓴다.
- **FP 경계 문단** (When a Finding Is a False Positive 섹션 내) — "FP는 코드가 실제로 문제없을 때만; 실재 결함은 계약 미정이면 `deferred: decision`, 이 PR과 무관한 pre-existing이면 `deferred: scope`".

### auto_fixable이 유일한 기준인 이유

자동 수정 여부는 **`auto_fixable` 하나로** 가른다 — 즉 "사람의 정책·설계 판단 없이 올바른 fix를
정할 수 있는가". 그 외 속성은 기준에 넣지 않으며, 이유는 다음과 같다.

- **머지 차단(`blocks_merge`)은 기준이 아니다.** 자동으로 고쳐지면 그 항목은 더 이상 머지를
  막지 않으므로, "고칠지"와 머지 차단 여부는 무관하다. 그래서 `blocks_merge`는 **`deferred: decision`
  항목의 우선순위 표시**로만 쓴다 (`deferred: scope`는 pre-existing이라 merge를 막지 않으며 필드 자체가 없다).
- **파일 위치·변경 규모도 자동 수정의 기준이 아니다.** fix가 일의적이면 위치·규모 무관하게 고친다.
  scope는 **고치지 못한 항목의 라우팅**에만 등장한다 — 이 PR에서 결정할 것(`decision`)과 사람이
  가져갈지 말지 고르는 것(`scope`)을 가르는 기준이지, 자동 수정을 막는 기준이 아니다.
  수정이 안전한지는 파이프라인의 Verify 단계(lint/test/typecheck)가 검증한다.
- **추측하지 않는다.** 올바른 fix가 일의적이지 않으면(`auto_fixable: N`) 자동으로 손대지 않고
  사람에게 넘긴다.

## 버린 것

원본에 있었으나 검증자 역할엔 무관해 들어낸 것들이다(왼쪽은 receiving-code-review의 섹션명).

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
