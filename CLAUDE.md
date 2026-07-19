# workflow

Runtime files (CLAUDE.md, `agents/`, `skills/`) are English for token efficiency; `docs/` may be Korean.

## agents/pr-review-finding-validator.md — verbatim first

Derived from the `superpowers:receiving-code-review` skill (implementer → read-only validator). Diff-ability against the original is an asset:

- Original-derived sections keep the original wording. Deviate only for the role inversion or for decision-model touchpoints needed at judgment time — no other rewording or reshuffling, even if a better phrasing comes to mind.
- The Decision and Output Format sections are ours — edit freely, but don't leak their terms (`auto_fixable`, `deferred`, `pre_existing`, …) into original-derived sections beyond existing touchpoints.
- Section mapping and rationale: [docs/pr-review-finding-validator.md](docs/pr-review-finding-validator.md) — update it after editing the agent.
