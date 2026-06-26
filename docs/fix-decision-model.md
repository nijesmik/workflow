# fix-결정 모델

`pr-review-finding-validator`는 review-pr가 낸 finding마다 중첩된 세 질문으로 판정을 내리고,
그 판정이 `pr-review` Fix·Comment 단계의 동작을 결정한다. 이 문서는 그 모델을 기술한다.

## 세 질문

1. **실재하는가?** — `verdict`: `TP` 또는 `FP`. FP면 사유만 기록하고 종료한다.
2. **(TP) 사람의 정책·설계·계약 판단 없이 올바른 fix를 결정·적용할 수 있는가?** — `auto_fixable`: `Y` 또는 `N`.
   - `Y` — fix가 일의적·기계적이다. 자동으로 수정한다. **파일 위치와 규모는 따지지 않는다.**
   - `N` — fix에 사람의 판단이 필요하다. 추측하지 않고 상세 브리프와 함께 기록(noted)한다.
3. **(`auto_fixable=N`일 때만) 머지를 막는가?** — `blocks_merge`: `Y` 또는 `N`. 미수정 항목 중
   무엇을 머지 전에 반드시 해결해야 하는지를 사람에게 표시한다.

## 산출

- **TP + `auto_fixable=Y`** → 자동 수정. `Fixed in <sha>`.
- **TP + `auto_fixable=N`** → 코드 변경 없이 상세 브리프(issue / decision_needed / options)와 `blocks_merge`를 기록.
- **FP** → 코드 변경 없이 사유만 기록.

## 왜 단일 결정인가

fix를 자동으로 적용할지 가르는 본질적 기준은 "사람의 정책·설계 판단이 필요한가" 하나다.

- 영향(머지 차단 여부)은 수정된 항목에는 의미가 없다 — 수정되면 더는 막지 않는다. 따라서
  머지 차단 여부는 라우팅 기준이 아니라 **미수정 항목의 우선순위 표시**일 뿐이다.
- 파일이 PR diff 안에 있는지, 변경 규모가 큰지는 fix의 정당성과 무관하다. review-pr의 finding은
  애초에 PR diff를 보고 나오므로 PR과 관련돼 있으며, PR의 코드가 의존하는 결함은 그 뿌리가
  다른 파일에 있어도 고치는 것이 옳다.

이 때문에 in-scope·규모 기준, 머지 차단 축, 추측에 기반한 잠정 수정은 모두 두지 않는다.

## 관련 파일

- [agents/pr-review-finding-validator.md](../agents/pr-review-finding-validator.md) — 판정을 산출하는 검증 에이전트
- [skills/pr-review/SKILL.md](../skills/pr-review/SKILL.md) — Fix·Comment 단계가 판정을 소비
