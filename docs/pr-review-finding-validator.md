# pr-review-finding-validator — 출처와 설계

`agents/pr-review-finding-validator.md`는 두 출처의 합성이다:

1. **`superpowers:receiving-code-review`** — 검증 원칙(어떻게 검증하고 회의할지)
2. **우리 설계 (brainstorming)** — 두 축 판정 모델(판정 결과를 `pr-review`가 어떻게 소비할지)

이 문서는 어느 줄이 어디서 왔는지를 명시한다.

## 역할 리프레이밍 (원본 → 우리)

원본 receiving-code-review는 **"리뷰 피드백을 받아 직접 고치는 구현자"** 시점이다.
이 에이전트는 정반대로 **"코드를 수정하지 않는 독립 검증자"**다. 작성자 편향을 피하려고
별도 서브에이전트로 띄우고, 코드를 직접 로드해 판정만 한다.

| | receiving-code-review | pr-review-finding-validator |
|---|---|---|
| 주체 | 메인 에이전트 본인 | fresh 서브에이전트 (편향 차단) |
| 대상 | 사람/리뷰어 피드백 | `pr-review-toolkit:review-pr` findings |
| 권한 | 구현(Edit/Write) | read-only (Read/Grep/Glob/Bash 조회만) |
| 종료 | 수용/반박 + 구현 | 구조화된 verdict (구현 없음) |

## 섹션별 출처

### receiving-code-review 원문에서 가져온 것 (검증 원칙)

| 에이전트 섹션 | 원본 대응 | 변경점 |
|---|---|---|
| Overview / Core principle | Overview | "Verify before **judging**"(원본은 implementing), 사회적 편안함보다 기술적 정확성은 그대로 |
| The Validation Pattern | The Response Pattern | READ/UNDERSTAND/VERIFY/EVALUATE 보존, **IMPLEMENT 제거**하고 JUDGE로 |
| Forbidden Responses | Forbidden Responses | "You're absolutely right!" 등 빈 동의 금지 그대로, 구현 관련 항목만 검증용으로 |
| Verifying Each Finding (5점 체크리스트) | Source-Specific Handling → External Reviewers | 거의 그대로. "Be skeptical, but check carefully" + 기능 깨짐/현재 구현 이유/플랫폼/전체 맥락 |
| "Cannot verify without [X]" | 동 섹션의 can't-verify 가이드 | "proceed?" 대신 "추측 금지, rationale에 한계 명시" |
| YAGNI Check | YAGNI Check for "Professional" Features | grep으로 사용처 확인 → 안 쓰이면 FP, 그대로 |
| When a Finding Is a False Positive | When To Push Back | "push back"을 "label FP"로 |
| The Bottom Line | The Bottom Line | "suggestions to evaluate, not orders to follow" 그대로 |

### 원본에 **없던** 우리 추가 (두 축 판정)

| 에이전트 섹션 | 비고 |
|---|---|
| **The Two Axes** | 원본에 없음. pr-review의 fix 결정 로직을 위해 brainstorming에서 설계 |
| `blocks_merge` (축 A) | 머지를 막는가 — 심각도·영향 |
| `auto_fixable` (축 B) | 안전하게 자동수정 가능한가 — in-scope · low-risk · bounded. 셋 다 충족 시 `Y` |
| `N-a` / `N-b` 구분 | not-auto-fixable 사유: 범위·크기(N-a) vs 모호·설계(N-b). N-b는 잠정수정 안전판의 트리거 |
| **Output Format** 스키마 | 원본에 없음. TP/FP + rationale + blocks_merge + auto_fixable을 구조화 출력 |

### 원본에서 **버린** 것

| 원본 섹션 | 버린 이유 |
|---|---|
| Implementation Order | 검증자는 구현하지 않음 |
| Acknowledging Correct Feedback / Gracefully Correcting Your Pushback | 구현자의 사회적 응답 가이드 — 검증자엔 무관 |
| GitHub Thread Replies | 인라인 댓글 응답 기법 — 검증자엔 무관 |
| "From your human partner"(신뢰) vs External 구분 | 검증자는 **모든** finding을 외부=검증 대상으로 동일 취급 |

## 왜 두 출처를 나눴나

- 원본의 검증 원칙은 **이미 검증된 자산**이라 그대로 쓸수록 좋다 — 빈 동의를 막고, 코드 대조를 강제하고, 외부 제안에 회의적이게 한다.
- 두 축은 우리 파이프라인 고유의 **소비 계약**이다 — `pr-review` 4단계 Fix가 이 verdict를 그대로 받아 매트릭스대로 처리한다.
- 둘을 섞지 않고 출처를 명시해두면, 원본 receiving-code-review가 업데이트될 때 무엇을 따라 갱신할지(검증 원칙)와 우리가 자체 관리할지(두 축)가 분명해진다.

## 관련 파일

- [agents/pr-review-finding-validator.md](../agents/pr-review-finding-validator.md) — 에이전트 본문
- [skills/pr-review/SKILL.md](../skills/pr-review/SKILL.md) — 3단계 Validate에서 이 에이전트를 dispatch, 4단계 Fix가 산출을 소비
