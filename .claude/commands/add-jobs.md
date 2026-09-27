Add one or more job links to the queue: $ARGUMENTS

Full design reference: `hunter-plan.md`.

For each URL given:
1. Normalize it (strip tracking params like `utm_*`, `gh_src`, `lever-source`, `ref`, trailing slash, casing in scheme/host) and compute a hash of the normalized URL.
2. Read `queue/queue.json`. If this hash matches an existing job's `url_hash`, report it as a duplicate (with that job's id and status) and skip — do not fetch it.
3. Otherwise, fetch the posting (WebFetch). If the fetch fails, is a login wall, or returns clearly blocked/short content: create the job with `fetch_status: "login-wall"` (or `"blocked"`/`"error"` as appropriate), a best-effort company/role guess from the URL, and status `needs-input`. Do not fabricate requirements. Move to the next URL.
4. On a good fetch: extract company, role, location, seniority, requirements, the ATS platform if identifiable, posted salary/deadline if shown, and any visible application questions.
5. Generate a job id as `<YYYY-MM-DD>_<company-slug>_<role-slug>` (append `-2`, `-3`... on collision). Create `jobs/<id>/`, write `posting.md` (frontmatter: job_id, url, fetched_at, company, role, location, ats, salary_posted, deadline, fetch_status + extracted requirements + a cleaned excerpt) and `questions.json` (`{ detected, source, questions: [{id, text, matched_bank_id}] }`, matching questions against ids in `profile/qa-bank.md` where reasonable).
6. If `profile/portfolio.json`'s `summary_cache` is missing or older than `profile/preferences.json`'s `stale_after_days`, fetch the portfolio URL and refresh the cached summary first.
7. For jobs with `fetch_status: "ok"`: call the Jev API with the portfolio summary and this job's requirements to get a fit score (0–1). If the Jev API is unreachable, leave the job at status `researched` (unscored) instead of blocking — don't guess a score.
8. Set status: `shortlisted` if `fit_score >= profile/preferences.json`'s `fit_threshold`, else `low-fit`.
9. Append/update the job's row in `queue/queue.json` (id, url, url_normalized, url_hash, company, role, status, fit_score, added_at, status_updated_at, folder). Write the job's `status.json` with an initial history entry.
10. Print a summary table: added (shortlisted / low-fit / needs-input / unscored) vs. duplicate, per URL.
