Show the queue's current state: $ARGUMENTS

Full design reference: `hunter-plan.md`.

Accepts an optional `--filter <status>` to show only jobs in that status.

1. Read `queue/queue.json` (read-only — this command never writes anything).
2. Print counts at the top, in this order: `shortlisted`, `awaiting drafting` (shortlisted but not yet drafted), `drafted`, `low-fit`, `researched (unscored)`, `needs-input`, `submitted`, `rejected`/`skipped`.
3. Print a table: id, company, role, status, fit_score, days-in-current-status (from `status_updated_at`).
4. Flag any job in `researched`, `shortlisted`, or `drafted` longer than `profile/preferences.json`'s `stale_after_days` with a `stale?` marker — this is a heuristic flag only, it doesn't change the stored status.
