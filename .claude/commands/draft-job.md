Draft tailored answers for a job (or all shortlisted jobs): $ARGUMENTS

Full design reference: `hunter-plan.md`.

Accepts a specific job id, or `--all-shortlisted` to target every job currently at status `shortlisted`. Also accepts `--refresh` (re-fetch the posting first) and `--force` (allow redrafting a job already `drafted`/`submitted`).

For each target job:
1. If the job's status is `drafted` or `submitted` and `--force` was not passed, refuse and say why.
2. If `--refresh` was passed, or the job's `posting.md` is older than `profile/preferences.json`'s `stale_after_days`, re-fetch the posting first and update `posting.md`.
3. Refuse to draft if `fetch_status` isn't `"ok"` — tell the user to supply the posting content manually first.
4. Read fresh (never cached): the portfolio (fetch `profile/portfolio.json`'s `url` directly, or use `summary_cache` if very recent), `profile/preferences.json`, `profile/qa-bank.md`, and this job's `posting.md`/`questions.json`.
5. For each detected application question, tailor an answer to this specific company/role/requirements — cite `[based on: <qa-bank id>]` or `[based on: portfolio]` provenance on every answer. If no questions were detected, draft the standard anticipated set instead (why-this-role, why-this-company, salary, notice-period, work-authorization), tagged as anticipated.
6. If a previous `draft-answers.md` exists for this job, copy it to `jobs/<id>/history/draft-v<N>.md` before overwriting; increment `draft_version`.
7. Write `jobs/<id>/draft-answers.md` with frontmatter (job_id, status, draft_version, drafted_at, based_on_profile_version) and a blank "Reviewer Notes" section at the end.
8. Set status to `drafted` in both `status.json` (append history entry) and `queue/queue.json`.

Print a summary of which jobs were drafted, skipped (already drafted/submitted, no `--force`), or refused (not fetch-ok).
