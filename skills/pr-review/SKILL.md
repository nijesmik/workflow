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

Validator output, per finding (this is the contract Steps 4 and 6 consume):
- `verdict`: `TP` / `FP` / `Unverified` — always, with a `rationale`.
- `auto_fixable`: `Y` / `N` — on a `TP` only.
- `decision_brief` (issue / decision_needed / options) — on an `auto_fixable: N` TP only.
- `blocks_merge`: `Y` / `N` — on an `auto_fixable: N` TP only.

`Unverified` means the validator couldn't confirm the finding from the code (runtime/platform-dependent, or too vague) — surface it to the human, never auto-fix. The validator avoids author bias — the main agent does not overturn its verdicts.

## Step 4 — Fix (no push)

**Fix guard (check first):** determine whether the PR head branch is the repo's default **or a protected** branch.
```bash
gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
# Protected rules too, if possible: gh api repos/{owner}/{repo}/branches/<head>/protection (404 = unprotected)
```
If the head is a default/protected branch (e.g. a release PR): **confirm with the user** — on approval continue below, on decline stop. If it's a feature branch: proceed unattended.

Handle each finding per its verdict. **The main agent makes the edits directly** with `Edit`/`Write` (the validator does not):

- **TP, `auto_fixable: Y`** → apply the fix. **A fix that lands in a file outside the PR diff, or a large fix, is still applied — location and size are not a reason to withhold it.** Commit each finding **right after fixing it** (before touching the next) with `commit-commands:commit`, **locally only** — so finding↔commit stays one-to-one. The `<sha>` in resolution is that commit's hash.
- **TP, `auto_fixable: N`** → no code change. Carry its `decision_brief` and `blocks_merge` to Step 6.
- **FP** → no code change. Carry its rationale to Step 6.
- **Unverified** → no code change (you can't safely fix what you can't confirm). Carry its rationale (what's missing) to Step 6.

**Do not push yet** — push happens in Step 5.

## Step 5 — Verify (pre-push gate)

The Step 4 fixes are still **local commits**. Run 5a → 5b → 5c in order, so a broken commit never reaches the remote first.

### 5a. lint/test/typecheck (always, first)

Find the commands in the project's CLAUDE.md, else from `package.json` scripts / Makefile / pyproject, etc. (if none found, note it in the comment).
- **If they fail:** stop here — do not push (broken commits stay local), record ❌, go to Step 6. Skip 5b (a verification failure takes priority over re-review).

### 5b. Conditional re-review (only after 5a passes)

Re-review the changed code with `pr-review-toolkit:review-pr` **only if a fix changed logic (behaviour)** — otherwise skip 5b. Re-review exists to catch a regression the auto-fix introduced, and only a behaviour change can introduce one.
- **Not logic** (skip): typo, comment, import-ordering, formatting.
- **Counts as logic** (re-review): a side-effecting import; a config value/constant change; a string-literal change; a function signature / interface change; any change to async flow, error handling, or state transitions.

From the re-review, **drop findings already judged** (dedupe) and send **only new TPs** back to Step 3 → 4. **Round counting:** the initial Step 2 review is round 0; each re-review pass is one round. **Stop after 2 re-review rounds** — record any remaining new TPs as "unresolved, blocks merge" and end the loop.

### 5c. Push

Once 5a passes and the loop has ended, push the local commits. **Unresolved TPs do not block the push** — push the fix commits that already passed verification (the remainder is reported in the comment). On a rejected push (non-fast-forward): `git pull --rebase` → **re-run 5a** → push if it passes; if it fails, record "push failed" and go to Step 6.

## Step 6 — Comment

**Every termination path after the PR is secured in Step 1** (success / verification failure / 2-round overflow / push failure) posts this comment. Exception: if Step 1 never obtained a PR (nothing to comment on), tell the user and stop. If the `gh pr comment`/`gh api` call itself fails, report the result to the user directly.

**Post one comment per PR (no duplicates).** Build `$BODY` from the template below, then upsert it — edit the existing comment carrying the marker `## pr-review result` if there is one, else create a new comment:
```bash
id=$(gh api "repos/{owner}/{repo}/issues/<number>/comments" \
       --jq '.[] | select(.body | startswith("## pr-review result")) | .id' | head -1)
if [ -n "$id" ]; then
  gh api --method PATCH "repos/{owner}/{repo}/issues/comments/$id" -f body="$BODY"
else
  gh pr comment <number> --body "$BODY"
fi
```

The `$BODY` template — two sections: items that need you (`auto_fixable: N` TP decisions and `Unverified` confirmations) come first as **detailed briefs**; resolved/dismissed items go below as a terse audit table. Omit a section if it has no rows.

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
- Severity (`Critical`/`Important`/`Suggestion`) is the label from the original review-pr finding (collected in Step 2) — the validator does not re-emit it.
- The terminal point is this comment. Do not merge.
