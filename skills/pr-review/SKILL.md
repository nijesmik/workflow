---
name: pr-review
description: Reviews the current branch's open PR end to end — review, validate, auto-fix, comment. Runs pr-review-toolkit, independently validates the findings, auto-fixes only the safe true positives, and posts an outcome table as a PR comment. Use to automate PR review from review through comment ("review this PR", "run pr-review").
---

# pr-review — review · validate · auto-fix · comment pipeline

Takes the **open PR** of the current branch as input and runs review → validate → fix → comment in sequence. Brings the PR to a mergeable + auditable state and **stops at the comment** (it does not merge or clean up).

It does not own PR-creation conventions. Commit/PR creation is delegated to `commit-commands`, review to `pr-review-toolkit`. Conventions such as the base branch are discovered at runtime (never hardcoded).

## Only two human gates
1. Confirming PR creation when none exists (Step 1)
2. Confirming a fix commit when the PR head is a default/protected branch (Step 4)

In the normal feature flow (PR exists + head is a feature branch) neither prompt appears; it runs unattended.

## Step 1 — Secure the PR

```bash
gh pr view --json number,headRefName,baseRefName,url
```
- If the command **fails with an error** (expired auth, network, permissions), stop and tell the user. `gh pr view` also exits non-zero when no PR exists, so distinguish **"no PR" from "gh error"** by whether stderr is a "no pull requests found" message. On error, do not fall through to the PR-creation flow.
- If an open PR **exists**, remember its number/head/base and go to Step 2.
- If **none exists**, ask the user: "There is no open PR for the current branch. Create one with `commit-commands:commit-push-pr`?"
  - Approved → invoke the `commit-commands:commit-push-pr` skill to commit, push, and create the PR (that command handles conventions) → re-run `gh pr view` to get the number, then Step 2.
  - Declined → stop.

## Step 2 — Review

Invoke the `pr-review-toolkit:review-pr` skill with no arguments (it picks applicable reviews from the diff). Collect findings from the result: each finding's description, `file:line`, and severity (Critical/Important/Suggestion).
**If there are zero findings**, skip Steps 3–5 (nothing to fix or verify) and go straight to Step 6 with a "no findings" comment.

## Step 3 — Validate (independent)

Group the findings **by file/module** and distribute them to the `pr-review-finding-validator` subagent (`Task` tool). If a group is too large (roughly 6+), split it further on file boundaries. If few findings cluster in one area, a single validator suffices. Give each validator its group's findings + the diff of the relevant files, and have it load the code directly to judge.

Validator output (per finding): `verdict` (`TP`/`FP`/`Unverified`) + rationale. A `TP` also gets `auto_fixable` (Y/N), and an `auto_fixable: N` TP a `decision_brief`. `Unverified` means the validator couldn't confirm the finding from the code (runtime/platform-dependent) — surface it to the human, never auto-fix. The validator is the device that avoids author bias — the main agent does not overturn its verdicts.

## Step 4 — Fix (no push)

**Fix guard (check first):** determine whether the PR head branch is the repo's default **or a protected** branch.
```bash
gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
# Protected rules too, if possible: gh api repos/{owner}/{repo}/branches/<head>/protection (404 = unprotected)
```
If the head is a default/protected branch (e.g. a release PR), confirm with the user before fixing. If it's a feature branch, proceed unattended.

Handle each finding per its verdict. **The main agent makes the edits directly** with `Edit`/`Write` (the validator does not):

- **TP, `auto_fixable: Y`** → apply the fix. **File location and size are irrelevant.** Commit each finding **right after fixing it** (before touching the next) with `commit-commands:commit`, **locally only** — so finding↔commit stays one-to-one. The `<sha>` in resolution is that commit's hash.
- **TP, `auto_fixable: N`** → no code change. Carry its `decision_brief` and `blocks_merge` to Step 6.
- **FP** → no code change. Carry its rationale to Step 6.
- **Unverified** → no code change (you can't safely fix what you can't confirm). Carry its rationale (what's missing) to Step 6.

**Do not push yet** — push happens in Step 5.

## Step 5 — Verify (pre-push gate)

The Step 4 fixes are still **local commits**. Proceed **in this order** so a broken commit never reaches the remote first.

**(1) lint/test/typecheck — always, and first.** Find the commands in the project's CLAUDE.md, else from `package.json` scripts / Makefile / pyproject, etc. (if none found, note it in the comment).
- **If they fail, stop here**: do not push (broken commits stay local), record ❌, go to Step 6. **Do not evaluate re-review** (verification failure takes priority over re-review).

**(2) Conditional re-review, only after verification passes.** If any of these is true, re-review with `pr-review-toolkit:review-pr`:
  1. A fix touched **logic** — typo/comment/import-ordering/formatting are excluded. But adding a side-effecting import, changing a config value/constant, or changing a string literal counts as logic.
  2. A fix touched a file outside the original PR diff
  3. A Critical TP was fixed

  If all are false, skip re-review.
- From the re-review result, **exclude findings already judged** (dedupe) and send **only new TPs** back to Steps 3→4. **Max 2 rounds.** Beyond that, record remaining new TPs as "unresolved, blocks merge" and stop the loop.

**(3) push.** Once verification passes and the loop ends, push the local commits. **Even if unresolved TPs remain, push the fix commits that already passed verification** (the remainder is reported in the comment and does not block the push). On a rejected push (non-fast-forward), `git pull --rebase` → **re-run lint/test/typecheck** → push if it passes; if it fails, record "push failed" and go to Step 6.

## Step 6 — Comment

**Every termination path after the PR is secured in Step 1** (success / verification failure / 2-round overflow / push failure) posts this comment. Exception: if Step 1 never obtained a PR (nothing to comment on), tell the user and stop. If the `gh pr comment`/`gh api` call itself fails, report the result to the user directly.

**Update the existing comment (avoid duplicates):** find the existing comment by the marker `## pr-review result` and edit it:
```bash
id=$(gh api "repos/{owner}/{repo}/issues/<number>/comments" \
       --jq '.[] | select(.body | startswith("## pr-review result")) | .id' | head -1)
if [ -n "$id" ]; then
  gh api --method PATCH "repos/{owner}/{repo}/issues/comments/$id" -f body="$BODY"
else
  gh pr comment <number> --body "$BODY"
fi
```

Two sections: items that need you — `auto_fixable: N` TP decisions and `Unverified` confirmations — come first as **detailed briefs**; resolved/dismissed items go below as a terse audit table. Omit a section if it has no rows.

```markdown
## pr-review result

<!-- Only if any auto_fixable:N finding has blocks_merge:Y -->
> ⚠️ **Unresolved merge blockers**: <items> — must be resolved before merge

### Needs your attention
<!-- auto_fixable:N TPs (a decision is needed) and Unverified findings (a confirmation is needed). Omit if none. -->
#### <short> (`file:line`) — <Critical/Important/Suggestion>, blocks merge: <yes/no>
- **Issue:** <what is wrong, where, with code context>
- **Decision needed:** <the question to resolve>
- **Options:** <candidate approaches and the trade-off of each>

#### <short> (`file:line`) — Unverified
- **Couldn't verify:** <what is missing — "Cannot verify without [X]">
- **Please confirm:** whether this is a real issue.

### Resolved & dismissed
<!-- Auto-fixed TPs and FPs. Omit this section if none. -->
| Finding | Severity | Verdict | Resolution |
| --- | --- | --- | --- |
| <finding> (`file:line`) | Critical/Important/Suggestion | TP/FP — rationale | Fixed in `<sha>` / Dismissed |

<!-- On verify failure / push failure -->
> ❌ lint/test/typecheck failed: <summary> (fix commits are local only, not pushed)
```

- "Needs your attention" holds (a) every `auto_fixable: N` TP as a decision brief and (b) every `Unverified` finding as a confirm-request, so the human can act quickly. `blocks_merge: Y` TPs are also called out above with ⚠️.
- "Resolved & dismissed" holds auto-fixed TPs and FPs — the terse closed record.
- The terminal point is this comment. Do not merge.
