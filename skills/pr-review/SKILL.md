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
- 명령이 **에러로 실패**하면(인증 만료·네트워크·권한) 중단하고 사용자에게 알린다. `gh pr view`는 PR이 없을 때도 비정상 종료하므로, stderr가 "no pull requests found" 류인지로 **"PR 없음"과 "gh 오류"를 구분**한다. 오류면 PR 생성 흐름으로 빠지지 말 것.
- 열린 PR이 **있으면** 번호·head·base를 기억하고 2단계로.
- **없으면** 사용자에게 묻는다: "현재 브랜치에 열린 PR이 없습니다. `commit-commands:commit-push-pr`로 PR을 생성할까요?"
  - 승인 → `commit-commands:commit-push-pr` 스킬을 호출해 커밋·푸시·PR 생성(컨벤션은 그 명령이 처리) → 다시 `gh pr view`로 번호 확보 후 2단계로.
  - 거절 → 중단한다.

## 2단계 — Review

`pr-review-toolkit:review-pr` 스킬을 인자 없이 호출한다(diff 보고 적용 리뷰를 자동 선택).
결과에서 finding을 수집한다: 각 finding의 설명, `file:line`, severity(Critical/Important/Suggestion).
**finding이 0개면** 3·4·5단계를 건너뛰고(수정·검증할 게 없음) 곧장 6단계에서 "findings 없음" 코멘트로 끝낸다.

## 3단계 — Validate (독립 검증)

findings를 **파일/모듈별로 묶어** `pr-review-finding-validator` 서브에이전트에 분배한다(`Task` 도구).
한 그룹이 너무 크면(대략 6개 이상) 파일 경계로 더 쪼갠다. 한 영역에 적게 모이면 단일 validator로
충분하다. 각 validator에 그 그룹의 findings + 해당 파일의 diff를 주고, 코드를 직접 로드해 판정하게 한다.

validator 산출(finding별): `verdict`(TP/FP) + 근거 + `blocks_merge`(Y/N) + `auto_fixable`(Y/N-a/N-b).
validator는 작성자 편향을 피하는 장치다 — 그 판정을 메인이 뒤집지 않는다.

## 4단계 — Fix (두 축 매트릭스, push는 안 함)

**fix 가드(먼저 확인)**: PR head 브랜치가 repo 기본 **또는 보호** 브랜치인지 본다.
```bash
gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
# 보호 룰도 가능하면: gh api repos/{owner}/{repo}/branches/<head>/protection (404면 비보호)
```
head가 기본/보호 브랜치면(release PR 등) 수정 전 사용자에게 확인한다. 피처 브랜치면 무인.

각 TP를 매트릭스대로 처리한다. **수정은 메인이 직접** `Edit`/`Write`로 한다(validator는 안 함). 표는 **TP 전용**:

| blocks_merge | auto_fixable | 동작 | resolution |
|---|---|---|---|
| Y | Y | 자동 수정 | `Fixed in <sha>` |
| Y | N-a | 자동 수정(5단계 재리뷰 강제) | `Fixed in <sha>` |
| Y | N-b | 자동 수정 시도(재리뷰), **해결로 안 침** | `Tentative fix in <sha> — 설계성 추측, 머지 전 사람 확인 필요` |
| N | Y | 자동 수정 | `Fixed in <sha>` |
| N | N-a 또는 N-b | 코드 변경 없이 기록 | `noted, 권장 수정` |

FP는 표 밖 — 코드 변경 없이 테이블에 FP 사유만 기록한다.

각 finding(또는 밀접히 연관된 묶음)을 `commit-commands:commit`으로 **로컬 커밋만** 한다 —
finding↔커밋이 일의적이도록(여러 finding을 한 커밋에 섞지 않는다). resolution의 `<sha>`는 그 커밋 해시.
**아직 push하지 않는다** — push는 5단계 검증 통과 후.

## 5단계 — Verify (push 전 게이트)

4단계 수정은 아직 **로컬 커밋** 상태다. 검증을 통과해야 push한다 — 깨진 커밋이 원격에 먼저 도달하지 않게.

- **항상 실행**: 영향 앱의 `lint / test / typecheck`. 명령은 프로젝트 CLAUDE.md에서 찾고, 없으면
  `package.json` scripts / Makefile / pyproject 등을 확인해 결정한다. 못 찾으면 그 사실을 코멘트에 남긴다.
- **조건부 재리뷰** — 아래 중 하나라도 참이면 수정 영역을 `pr-review-toolkit:review-pr`로 재리뷰:
  1. 수정이 **로직**을 건드림 — 단순 typo·주석·import 순서·포맷은 제외. 단 **사이드이펙트 import 추가·설정값/상수 변경·문자열 리터럴 변경은 로직으로 본다.**
  2. 수정이 원래 PR diff 밖 파일을 건드림
  3. Critical TP를 수정함
  4. N-b 잠정 수정이 있음(항상 재리뷰)

  다 거짓이면 재리뷰 스킵.
- 재리뷰가 **새 TP**를 내면 3→4단계로 돌아간다. **최대 2라운드**(재리뷰 사이클 2회). 초과 시 잔여 TP는
  "unresolved, blocks merge"로 기록하고 루프를 멈춘다.
- **검증 통과 시**: 로컬 커밋들을 push한다. push가 거부되면(non-fast-forward) `git pull --rebase` 후 1회 재시도,
  그래도 실패하면 "push 실패"로 기록하고 6단계로.
- **lint/test/typecheck 실패 시**: **push하지 않는다**(깨진 커밋은 로컬에 남김). ❌로 기록하고 6단계로.

## 6단계 — Comment (모든 종료 경로가 거친다)

정상 완료든 중단(검증 실패 / 2라운드 초과 / push 실패 / gh 오류 외)이든 **항상 이 코멘트를 남긴다.**

**기존 코멘트 갱신**: 코멘트 첫 줄 마커 `## pr-review 결과`로 식별한다. 같은 마커의 코멘트가 이미 있으면
그 코멘트를 edit하고(코멘트 id를 `gh api`로 찾아 PATCH, 또는 러너가 pr-review 코멘트만 단다는 전제하에
`gh pr comment <번호> --edit-last`), 없으면 `gh pr comment <번호> --body ...`로 새로 만든다 → 재실행해도 중복 안 쌓임.

```markdown
## pr-review 결과

<!-- 미해결 머지-블로커가 있으면 테이블 위에 명시 -->
> ⚠️ **머지 블로커 미해결**: <항목들> — 머지 전 사람 확인 필요

| Finding | Severity | Verdict | Resolution |
| --- | --- | --- | --- |
| <finding> (`file:line`) | Critical/Important/Suggestion | TP/FP — 근거 | Fixed in `<sha>` / Tentative fix in `<sha>` / noted / Dismissed |

<!-- Verify 실패 / push 실패 시 -->
> ❌ lint/test/typecheck 실패: <요약> (수정 커밋은 로컬에만 있고 push 안 됨)
```

- blocks_merge TP 중 미해결 + 모든 N-b 잠정 수정은 테이블 **밖**에 ⚠️로 명시한다.
- 종료점은 이 코멘트다. 머지하지 않는다.
