# 결정: 두 축 판정 모델 (blocks_merge × auto_fixable)

`pr-review-finding-validator`가 각 TP에 다는 두 판정 — `blocks_merge`와 `auto_fixable` —
의 설계 근거를 기록한다. (ADR 성격)

> **상태**: 채택됨 (추론 기반 설계). 실사용 검증은 미완 → [#1](https://github.com/nijesmik/workflow/issues/1)

## 맥락 — 심각도 단일 축이 틀렸다

처음 fix 범위를 **심각도(Critical/Important/Suggestion)로 갈랐다** — Critical/Important는
자동수정, Suggestion은 기록. 이게 양 끝에서 틀린다:

- **Suggestion인데 1줄짜리 명백·안전한 개선** → 심각도 기준이면 버려짐 (놓치면 아까운 공짜 개선)
- **Critical인데 재설계가 필요** → 심각도 기준이면 "고쳐라"인데, 블라인드 자동수정이 가장 위험한 지점

즉 **심각도는 "고칠 가치/안전성"의 축이 아니다.** 한 축으로 두 개의 다른 질문에 답하려 한 게 오류.

## 결정 — 두 질문은 직교하므로 두 축으로 분리

| 축 | 답하는 질문 | 입력 |
|---|---|---|
| **A — `blocks_merge`** | 이게 머지를 막아야 하나? | 영향·심각도 |
| **B — `auto_fixable`** | 사람 판단 없이 지금 안전하게 고칠 수 있나? | 수정의 성격 (in-scope · low-risk · bounded) |

두 질문은 상관이 없다(직교). `auto_fixable`은 **셋을 다 충족**해야 `Y`:
1. **In-scope** — 수정이 이미 PR diff에 있는 파일 안에 머문다.
2. **Low-risk / 명백** — 올바른 수정이 일의적. 설계 트레이드오프·계약 변경·재설계 없음.
3. **Bounded** — 작고 국소적.

## 분리의 핵심 이득 — 위험 사분면이 드러난다

| blocks_merge | auto_fixable | 단일(심각도) 축의 오류 | 두 축의 처리 |
|---|---|---|---|
| Y | Y | — | 자동수정 |
| **Y** | **N** | "Critical이니 자동수정" → 추측 수정 위험 | **잠정수정 + 사람 확인 (안전판)** |
| N | Y | "Suggestion이니 버림" → 공짜 안전개선 놓침 | 자동수정(줍는다) |
| N | N | — | 기록만 |

단일 축이면 **blocks_merge=Y & auto_fixable=N** 사분면이 "Critical 자동수정"으로 묻힌다.
분리하니 거기만 따로 안전판(잠정 라벨 + 머지 게이트)을 걸 수 있다.

## not-auto-fixable 세분 (N-a / N-b) — 기준은 "재리뷰가 안전망이 되는가"

`blocks_merge=Y & auto_fixable=N`을 자동수정할지 결정하려면 이유를 쪼개야 했다:

- **N-a (범위·크기)** — 수정이 기계적으로 명백한데 단지 크거나 PR 밖 파일을 건드림.
  → 재리뷰가 "맞게 고쳤나"를 검증할 수 있음 → **자동수정 + 재리뷰**.
- **N-b (모호·설계·계약)** — 올바른 수정이 일의적이지 않음(트레이드오프/재설계).
  → 재리뷰는 "에이전트가 고른 설계가 의도에 맞나"를 검증 **못 함** → **잠정수정 + 사람 확인**.

## 결과 (consequences)

- validator는 TP마다 `blocks_merge`(Y/N) + `auto_fixable`(Y/N-a/N-b)을 산출한다.
- `pr-review` 4단계 Fix가 이 두 값으로 [매트릭스](../skills/pr-review/SKILL.md)를 따라 동작한다.
- N-b는 "Tentative fix" 라벨 + blocks-merge 유지의 트리거다.

## 한계 / 미검증

이건 **추론으로 도출한 설계 휴리스틱**이지 데이터로 검증한 모델이 아니다. in-scope·low-risk·
bounded의 경계는 validator의 판단에 맡긴 주관적 선이고, 실제 PR들에 돌려보기 전엔 두 축이
충분히 갈라주는지(특히 "low-risk"의 회색지대, N-a/N-b 구분의 일관성) 확신할 수 없다.

→ 실사용 검증은 후속 이슈로 분리: **[#1 두 축 판정 모델 실사용 검증](https://github.com/nijesmik/workflow/issues/1)**

## 관련 파일
- [agents/pr-review-finding-validator.md](../agents/pr-review-finding-validator.md) — The Two Axes 섹션
- [docs/pr-review-finding-validator.md](./pr-review-finding-validator.md) — 출처(원본 vs 우리 추가) 구분
