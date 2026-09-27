Print the current shortlist, ready to paste into a Claude for Chrome session: $ARGUMENTS

Full design reference: `hunter-plan.md`.

Accepts an optional `--min-score <n>` to override `profile/preferences.json`'s `fit_threshold` for this listing only (doesn't change stored data).

1. Read `queue/queue.json`. Take every job with status `shortlisted` (or, if `--min-score` given, any job whose `fit_score` clears that value regardless of stored status), sorted by `fit_score` descending.
2. If any of these jobs are not yet `drafted`, note that at the top ("N jobs not yet drafted — run `/draft-job --all-shortlisted` first").
3. Print, in a clean pasteable block:
   - Your portfolio URL (from `profile/portfolio.json`).
   - One line per job: id, company, role, fit_score, url.
4. This is read-only — it must not modify any file.
