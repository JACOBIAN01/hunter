# Hunter — project rules

Job application assistant. Full design: [hunter-plan.md](hunter-plan.md), [hunter-architecture.md](hunter-architecture.md), [hunter-jev.md](hunter-jev.md).

## Non-negotiables

- Never mark a job `approved` or `submitted` automatically. Only `/set-status`, run by the human, changes those.
- Never fabricate facts not present in `profile/portfolio.json`'s cached summary or `profile/preferences.json`. If a question needs info that isn't there, say so in the draft instead of guessing.
- Never mass-edit `profile/qa-bank.md` silently — it's human-curated; propose changes, don't overwrite.
- Never submit an application without a human confirming first, no matter how confident the draft is.
- If the Jev API is unreachable, land jobs as `researched` (unscored) rather than blocking the batch or guessing a score.
