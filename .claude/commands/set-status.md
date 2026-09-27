Record a status change for one or more jobs: $ARGUMENTS

Full design reference: `hunter-plan.md`.

Expected form: one or more job ids, followed by the target status (`shortlisted`, `low-fit`, `drafted`, `approved`, `rejected`, `skipped`, `submitted`), optionally followed by a quoted reason (used for `rejected`/`skipped`).

Legal transitions only:
- `researched` → `shortlisted` or `low-fit` (normally set by `/add-jobs`, not this command)
- `shortlisted` → `drafted` (normally set by `/draft-job`, not this command)
- `drafted` → `submitted`
- any active status → `rejected` or `skipped` (requires a reason)
- `needs-input` → `researched` (once content has been supplied)

For each job id given:
1. Read `jobs/<id>/status.json`. If the requested transition isn't in the legal list above, refuse and say what the valid next statuses are instead.
2. Otherwise, append a history entry (`at`, `from`, `to`, `by: "set-status"`, and `reason` if given), update `status` and `blocked_reason` (if rejecting/skipping) in `status.json`.
3. Mirror the new `status` and `status_updated_at` into that job's row in `queue/queue.json`. If transitioning to `submitted`, also stamp `submitted_at`.

Print a one-line result per job id (updated, or refused-with-reason).
