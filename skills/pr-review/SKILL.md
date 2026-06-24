---
name: pr-review
description: 현재 브랜치의 열린 PR을 리뷰→검증→자동수정→코멘트까지 한 번에 처리한다. pr-review-toolkit으로 리뷰하고 findings를 독립 검증해 안전 범위의 TP만 자동 수정한 뒤 outcome 테이블을 PR 코멘트로 남긴다. Use when PR 리뷰 자동화, 리뷰부터 코멘트까지, "리뷰 돌려줘", pr-review를 요청할 때.
---

# pr-review — PR 리뷰·검증·자동수정·코멘트 파이프라인

현재 브랜치의 **열린 PR**을 입력으로 리뷰 → 검증 → 수정 → 코멘트를 연속 실행한다. PR을
머지 가능 + 감사 가능 상태로 만들고 **코멘트에서 멈춘다**(머지·클린업은 하지 않는다).

PR 생성 컨벤션은 짊어지지 않는다. 커밋/PR 생성은 `commit-commands`에, 리뷰는
`pr-review-toolkit`에 위임한다. base 브랜치 등 컨벤션은 런타임에 발견한다(하드코딩 금지).

## 사람 게이트는 두 곳뿐
1. PR이 없을 때 생성 확인 (1단계)
2. PR head가 기본/보호 브랜치일 때 fix 커밋 확인 (4단계)

일반 피처 흐름(PR 존재 + head=피처 브랜치)에선 둘 다 뜨지 않고 무인 진행한다.

## 1단계 — PR 확보

```bash
gh pr view --json number,headRefName,baseRefName,url
```
- 열린 PR이 **있으면** 번호·head·base를 기억하고 2단계로.
- **없으면** 사용자에게 묻는다: "현재 브랜치에 열린 PR이 없습니다. `commit-commands:commit-push-pr`로 PR을 생성할까요?"
  - 승인 → `commit-commands:commit-push-pr` 스킬을 호출해 커밋·푸시·PR 생성(컨벤션은 그 명령이 처리) → 다시 `gh pr view`로 번호 확보 후 2단계로.
  - 거절 → 중단한다.

## 2단계 — Review

`pr-review-toolkit:review-pr` 스킬을 인자 없이 호출한다(diff 보고 적용 리뷰를 자동 선택).
결과에서 finding을 수집한다: 각 finding의 설명, `file:line`, severity(Critical/Important/Suggestion).
finding이 0개면 5단계 검증만 하고(수정 없음) 6단계에서 "findings 없음" 코멘트로 끝낸다.

## 3단계 — Validate (독립 검증)

findings를 **파일/모듈별로 묶어** `pr-review-finding-validator` 서브에이전트에 분배한다(`Task` 도구).
findings가 적으면(대략 5개 이하 또는 한 영역) 단일 validator로 충분하다. 각 validator에 그
그룹의 findings + 해당 파일의 diff를 주고, 코드를 직접 로드해 판정하게 한다.

validator 산출(finding별): `verdict`(TP/FP) + 근거 + `blocks_merge`(Y/N) + `auto_fixable`(Y/N-a/N-b).
validator는 작성자 편향을 피하는 장치다 — 그 판정을 메인이 뒤집지 않는다.

## 4단계 — Fix (두 축 매트릭스)

**fix 가드(먼저 확인)**: PR head 브랜치가 repo 기본/보호 브랜치인지 본다.
```bash
gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
```
head == 기본 브랜치면(release PR 등) 자동 커밋 전에 사용자에게 확인한다. 그 외 피처 브랜치면 무인.

각 TP를 매트릭스대로 처리한다. **수정은 메인이 직접** `Edit`/`Write`로 한다(validator는 안 함):

| blocks_merge | auto_fixable | 동작 | resolution |
|---|---|---|---|
| Y | Y | 자동 수정 | `Fixed in <sha>` |
| Y | N-a | 자동 수정(+5단계 재리뷰 강제) | `Fixed in <sha>` |
| Y | N-b | 자동 수정 시도(+재리뷰), **해결로 안 침** | `Tentative fix in <sha> — 설계성 추측, 머지 전 사람 확인 필요` |
| N | Y | 자동 수정 | `Fixed in <sha>` |
| N | N(-a/-b) | 코드 변경 없이 기록 | `noted, 권장 수정` |
| FP | — | 변경 없음 | FP 사유 기록 |

각 수정 묶음은 `commit-commands:commit`으로 커밋하고 같은 PR 브랜치에 push한다. resolution의
`<sha>`는 그 커밋 해시.

## 5단계 — Verify

- **항상 실행**: 영향 앱의 `lint / test / typecheck`. 명령은 프로젝트 CLAUDE.md에서 찾고, 없으면
  `package.json` scripts / Makefile / pyproject 등을 확인해 결정한다. 못 찾으면 그 사실을 코멘트에 남긴다.
- **조건부 재리뷰** — 아래 중 하나라도 참이면 수정 영역을 `pr-review-toolkit:review-pr`로 재리뷰:
  1. 수정이 로직을 건드림(단순 typo·주석·import·포맷이 아님)
  2. 수정이 원래 PR diff 밖 파일을 건드림
  3. Critical TP를 수정함
  4. N-b 잠정 수정이 있음(항상 재리뷰)

  다 거짓이면 재리뷰 스킵.
- 재리뷰가 **새 TP**를 내면 3→4단계로 돌아간다. **최대 2라운드.** 초과 시 잔여 TP는
  "unresolved, blocks merge"로 기록하고 멈춘다.
- `lint/test/typecheck`가 깨지면 코멘트에 ❌로 기록하고 거기서 중단한다(루프 계속하지 않음).

## 6단계 — Comment

`gh pr comment <번호> --body <테이블>`로 단일 코멘트를 남긴다(같은 PR에 테이블 하나; 재실행 시 갱신).

```markdown
## pr-review 결과

<!-- 미해결 머지-블로커가 있으면 테이블 위에 명시 -->
> ⚠️ **머지 블로커 미해결**: <항목들> — 머지 전 사람 확인 필요

| Finding | Severity | Verdict | Resolution |
| --- | --- | --- | --- |
| <finding> (`file:line`) | Critical/Important/Suggestion | TP/FP — 근거 | Fixed in `<sha>` / Tentative fix in `<sha>` / noted / Dismissed |

<!-- Verify 실패 시 -->
> ❌ lint/test/typecheck 실패: <요약>
```

- blocks_merge TP 중 미해결 + 모든 N-b 잠정 수정은 테이블 **밖**에 ⚠️로 명시한다.
- 종료점은 이 코멘트다. 머지하지 않는다.
