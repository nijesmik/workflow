---
name: pr-review
description: Reviews the current branch's open PR end to end — review, validate, auto-fix, comment, issue gate. Runs pr-review-toolkit, independently validates the findings, auto-fixes only the safe true positives, posts an outcome comment, and lets the user pick which pre-existing defects become issues. Use to automate PR review from review through comment ("review this PR", "run pr-review").
---

# pr-review — review · validate · auto-fix · comment · issue-gate pipeline

Takes the **open PR** of the current branch as input and runs review → validate → fix → comment → issue gate in sequence. Brings the PR to a mergeable + auditable state; after the comment, an interactive run asks which pre-existing defects to file as issues (Step 7), then **stops** (it does not merge or clean up).

## Human gates
Two **blocking** gates before merge-relevant actions:
1. Confirming PR creation when none exists (Step 1)
2. Confirming a fix commit when the PR head is a default/protected branch (Step 4)

Plus one **terminal, non-blocking** question: which pre-existing defects to register as issues (Step 7) — asked after push and comment, only when candidates exist, skipped on non-interactive runs.

In the normal feature flow (PR exists + head is a feature branch) it runs unattended through push and comment; only the Step 7 selection can appear at the end.

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

Validator output, per finding (this is the contract Steps 4–7 consume):
- `verdict`: `TP` / `FP` / `Unverified` — always, with a `rationale`.
- `pre_existing`: `Y` — on an `Unverified` finding whose cited code predates this PR (revert test).
- `auto_fixable`: `Y` / `N` — on a `TP` only.
- `deferred`: `decision` / `scope` — on an `auto_fixable: N` TP only. `decision` = this PR's own problem, needs a human call; `scope` = pre-existing defect this PR neither depends on nor aggravates.
- `decision_brief` (issue / impact / decision_needed / options / recommendation) — on a `deferred: decision` TP only.
- `blocks_merge`: `Y` / `N` — on a `deferred: decision` TP only. A `deferred: scope` TP never blocks merge and carries **no** `blocks_merge` field (absent = N).
- `followup_brief` (issue / why_out_of_scope / impact / fix_sketch) — on a `deferred: scope` TP only.

`Unverified` means the validator couldn't confirm the finding from the code (runtime/platform-dependent, or too vague) — surface it to the human, never auto-fix. The main agent does not overturn the validator's verdicts.

## Step 4 — Fix (no push)

**Fix guard (check first):** determine whether the PR head branch is the repo's default **or a protected** branch.
```bash
gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
# Protected rules too, if possible: gh api repos/{owner}/{repo}/branches/<head>/protection (404 = unprotected)
```
If the head is a default/protected branch (e.g. a release PR): **confirm with the user** — on approval continue below, on decline stop. If it's a feature branch: proceed unattended.

Handle each finding per its verdict. **The main agent makes the edits directly** with `Edit`/`Write` (the validator does not):

- **TP, `auto_fixable: Y`** → apply the fix (wherever it lands). Commit each finding **right after fixing it** (before touching the next) with `commit-commands:commit`, **locally only** — so finding↔commit stays one-to-one. The `<sha>` in resolution is that commit's hash.
- **TP, `deferred: decision`** → no code change. Carry its `decision_brief` and `blocks_merge` to Step 6 ("Decide in this PR").
- **TP, `deferred: scope`** → no code change. Carry its `followup_brief` to Step 6 ("Pre-existing defects" — issue candidates for Step 7).
- **FP** → no code change. Carry its rationale to Step 6.
- **Unverified** → no code change (you can't safely fix what you can't confirm). Carry its rationale (what's missing) — and its `pre_existing: Y` flag if set — to Step 6.

**Do not push yet** — push happens in Step 5.

## Step 5 — Verify (pre-push gate)

The Step 4 fixes are still **local commits**. Run 5a → 5b → 5c in order.

### 5a. lint/test/typecheck (always, first)

Find the commands in the project's CLAUDE.md, else from `package.json` scripts / Makefile / pyproject, etc. (if none found, note it in the comment).
- **If they fail:** stop here — do not push (broken commits stay local), record ❌, go to Step 6. Skip 5b (a verification failure takes priority over re-review).

### 5b. Conditional re-review (only after 5a passes)

Re-review the changed code with `pr-review-toolkit:review-pr` **only if a fix changed logic (behaviour)** — otherwise skip 5b. Re-review exists to catch a regression the auto-fix introduced, and only a behaviour change can introduce one.
- **Not logic** (skip): typo, comment, import-ordering, formatting.
- **Counts as logic** (re-review): a side-effecting import; a config value/constant change; a string-literal change; a function signature / interface change; any change to async flow, error handling, or state transitions.

From the re-review, **drop findings already judged** (dedupe) and send **only new TPs** back to Step 3 → 4. **Round counting:** the initial Step 2 review is round 0; each re-review pass is one round. **Stop after 2 re-review rounds** — run any remaining new findings through Step 3 (validation and classification only, no further fixes), then split by track: `deferred: decision` and unfixed `auto_fixable: Y` leftovers are recorded as "unresolved, blocks merge" in Step 6's "Unresolved (round cap reached)" section and named in the ⚠️ banner; `deferred: scope` leftovers go to the Pre-existing defects section (they never block merge); FP/Unverified go to their normal sections. End the loop.

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

The `$BODY` template — four sections: decisions this PR needs (`deferred: decision` briefs and non-pre-existing `Unverified` confirmations) come first as **detailed briefs**; pre-existing defects (issue candidates for Step 7) next; any round-cap leftovers (unresolved blockers from Step 5b) next; resolved/dismissed items last as a terse audit table. Omit a section if it has no rows.

**Carry over Follow-up state before building `$BODY`:** if a marker comment already exists, read each Pre-existing item's `Follow-up` value from it (match items by `file:line` + short title) and keep any issue link or `not registered` verbatim — never reset an already-answered item to `pending`. Items with a carried-over answer are excluded from Step 7.

```markdown
## pr-review result

<!-- Only if any deferred:decision finding has blocks_merge:Y, or the round cap (Step 5b) left deferred:decision or unfixed auto_fixable:Y findings unresolved -->
> ⚠️ **Unresolved merge blockers**: <items> — must be resolved before merge

### Decide in this PR
<!-- deferred:decision TPs (detailed briefs) and Unverified findings without pre_existing (confirmation requests). Omit if none. -->
#### <short> (`file:line`) — <Critical/Important/Suggestion>, blocks merge: <yes/no>
- **What's wrong:** <what is wrong, where, with code context>
- **Impact if unresolved:** <what actually happens if left — user/data perspective>
- **Decision needed:** <the question to resolve>
- **Options:** <candidate approaches and the trade-off of each>
- **Recommendation:** <option — one-line code-grounded reason> / none — not decidable from code evidence

#### <short> (`file:line`) — Unverified
- **Couldn't verify:** <what is missing — "Cannot verify without [X]">
- **Please confirm:** whether this is a real issue.

### Pre-existing defects — issue candidates
<!-- Two item kinds, both pre-existing and never blocking merge. Omit the section if neither kind has rows. -->
<!-- Kind 1: deferred:scope TPs (any severity) — full followup_brief. -->
#### <short> (`file:line`) — <Critical/Important/Suggestion>, pre-existing
- **What's wrong:** <what is wrong, where, with code context>
- **Why out of scope:** <exists on base; this PR neither depends on nor aggravates it>
- **Impact if left:** <what happens + trigger conditions / frequency>
- **Fix sketch:** <direction and rough size>
- **Follow-up:** pending / <issue link> / not registered

<!-- Kind 2: pre_existing Unverified findings — no followup_brief, only a rationale. Label `unconfirmed`. -->
#### <short> (`file:line`) — <Critical/Important/Suggestion>, pre-existing, unconfirmed
- **What's wrong:** <the finding's claim>
- **Couldn't verify:** <rationale — "Cannot verify without [X]">
- **Follow-up:** pending / <issue link> / not registered

### Unresolved (round cap reached)
<!-- Only when Step 5b hits the 2-round cap: deferred:decision leftovers and unfixed auto_fixable:Y leftovers that were classified but never fixed. Both block merge and must also appear in the ⚠️ banner above. Omit if the cap was never reached. -->
| Finding | Severity | Verdict | Why unresolved |
| --- | --- | --- | --- |
| <finding> (`file:line`) | Critical/Important/Suggestion | TP — deferred:decision / TP — auto_fixable:Y | round cap reached before fix |

### Resolved & dismissed
<!-- Auto-fixed TPs and FPs. Omit this section if none. -->
| Finding | Severity | Verdict | Resolution |
| --- | --- | --- | --- |
| <finding> (`file:line`) | Critical/Important/Suggestion | TP/FP — rationale | Fixed in `<sha>` / Dismissed |

<!-- On verify failure / push failure -->
> ❌ lint/test/typecheck failed: <summary> (fix commits are local only, not pushed)
```

- blocks_merge: Y items — `deferred: decision` findings with `blocks_merge:Y`, plus round-cap leftovers from Step 5b (`deferred: decision` and unfixed `auto_fixable: Y`) — are called out above the sections with ⚠️.
- Severity (`Critical`/`Important`/`Suggestion`) comes from the original review-pr finding (Step 2) — the validator does not emit it.
- After the comment, run Step 7 (issue gate) if it applies; otherwise this comment is the terminal point. Do not merge.

## Step 7 — Issue gate (terminal)

Runs after the Step 6 comment when the comment has Pre-existing items with `Follow-up: pending` **and** the session is interactive. Ask the user: **"Which of these pre-existing defects should I register as GitHub issues?"** (multi-select, include a "none" option).

- **Selected items** → `gh issue create` — title from the item's short title; body from its `followup_brief` for a `deferred: scope` item, or from its **rationale** (labeled "unconfirmed — needs human confirmation") for an `unconfirmed` item; plus a link to this PR. Then update the item's `Follow-up` in the comment to the issue link (reuse the Step 6 upsert).
- **Unselected items** (and all items when "none" is chosen) → update `Follow-up` to `not registered`, so a re-run does not re-ask.
- **Non-interactive run** (no one to answer) → skip Step 7 entirely; items stay `pending` and the next interactive run picks them up.
- **`gh issue create` fails** (issues disabled, fork, permissions) → keep that item `pending` and report the failure to the user.
- Step 7 also runs after a verify-failure or push-failure termination (the comment was still posted, and pre-existing defects are independent of this PR's verify result).
- No `pending` items → skip Step 7; the comment is the terminal point. **Never create an issue the user did not select.**
