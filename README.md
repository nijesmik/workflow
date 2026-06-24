# workflow

Claude Code 플러그인. 개발 워크플로우 자동화 스킬 모음.

## 스킬

### `/workflow:pr-review`
현재 브랜치의 열린 PR을 리뷰 → 검증 → 자동수정 → 코멘트까지 처리한다.

- 리뷰: `pr-review-toolkit:review-pr` (의존성, 자동 설치)
- 검증: `pr-review-finding-validator` 에이전트가 findings를 독립 판정(TP/FP · blocks-merge · auto-fixable)
- 수정: 안전 범위(in-scope·low-risk·bounded)의 TP만 자동 수정, 설계성 이슈는 "잠정 수정"으로 표시
- 결과: PR 코멘트 — 사람 손이 필요한 TP("Needs your attention")와 자동수정·FP("Resolved & dismissed")를 두 테이블로 분리. **머지는 하지 않는다**(사람이 머지 게이트).

PR이 없으면 `commit-commands:commit-push-pr`로 생성할지 먼저 묻는다.

## 설치

```
/plugin marketplace add nijesmik/workflow
/plugin install workflow@nijesmik
```

의존성 `pr-review-toolkit`(리뷰)과 `commit-commands`(커밋·PR 생성)가 함께 설치된다.
