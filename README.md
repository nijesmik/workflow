# workflow

Claude Code 플러그인. 개발 워크플로우 자동화 스킬 모음.

## 스킬

### `/workflow:pr-review`
현재 브랜치의 열린 PR을 리뷰 → 검증 → 자동수정 → 코멘트 → 이슈 등록 선택까지 처리한다.

- 리뷰: `pr-review-toolkit:review-pr` (의존성, 자동 설치)
- 검증: `pr-review-finding-validator` 에이전트가 각 finding을 독립 판정 — `TP`(실재) / `FP`(문제없음) / `Unverified`(코드만으론 확정 불가)
- 수정: 사람의 정책·설계 판단 없이 고칠 수 있는 TP(`auto_fixable`)만 자동 수정·커밋한다. 나머지 TP는 **이 PR 안에서 결정할 것**(`deferred: decision`)과 **PR 이전부터 있던 범위 밖 결함**(`deferred: scope`)으로 갈라 사람에게 넘긴다 — 범위 밖 결함은 merge를 막지 않는다
- 결과: PR 코멘트 — **"Decide in this PR"**(결정 필요 TP·확인 필요 Unverified 상세 브리프) + **"Pre-existing defects"**(범위 밖 결함 전부) + **"Resolved & dismissed"**(자동수정·FP 테이블). 마지막에 범위 밖 결함 중 **사람이 고른 것만** GitHub 이슈로 등록한다(스킬이 임의로 이슈를 만들지 않음). **머지는 하지 않는다**(사람이 머지 게이트).

PR이 없으면 `commit-commands:commit-push-pr`로 생성할지 먼저 묻는다.

## 설치

```
/plugin marketplace add nijesmik/workflow
/plugin install workflow@nijesmik
```

의존성 `pr-review-toolkit`(리뷰)과 `commit-commands`(커밋·PR 생성)가 함께 설치된다.
