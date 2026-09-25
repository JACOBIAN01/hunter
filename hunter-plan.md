# Job Application Assistant (Claude for Chrome workflow)

## Context

The goal: apply to many jobs (tens to hundreds of links) with almost no manual overhead, while keeping exactly one safety net — a quick human glance before anything is actually submitted to a real employer.

The design has three moving pieces:
1. **Your live portfolio** (a website/URL) — the canonical source of your experience, skills, and background. Not a locally-maintained resume file; Claude Code and Claude for Chrome both read it directly.
2. **Jev** (TypeSafe AI's "System One" model — see [hunter-jev.md](hunter-jev.md)) — a fast, cheap, typed-decision model. Its one job here is **batch triage**: given a pile of job links, score each one's fit against your portfolio *before* anyone spends time on it. It does not touch the browser and cannot be wired into Claude for Chrome's live decisions — it runs entirely offline, inside Claude Code, before you ever open a browser.
3. **Claude for Chrome** — the interactive automator. Once a shortlist exists, you open one continuous session, hand it the whole shortlist plus your portfolio, and it works through the list itself: reading your portfolio, filling every field (common and creative alike — it does all of this itself, Jev's involvement already ended at the shortlist), and pausing once per job for your quick confirm before submitting.

Full diagram: [hunter-architecture.md](hunter-architecture.md).

Confirmed constraints:
- No custom bot / unattended scripted automation — Claude for Chrome stays interactive, human present for the whole batch session.
- A human always gets a quick confirm before each submit — never zero-review, even at scale.
- Plain local files only for tracking (JSON/Markdown) — no database, no dashboard.
- Structure shouldn't need a rewrite if volume grows from tens to hundreds of links.

## How it works, end to end (one batch of many links)

1. **`/add-jobs <link1> <link2> ... <linkN>`** — dedup, fetch, extract each posting, **and** score each one's fit against your portfolio via Jev, setting status `shortlisted` (score ≥ `fit_threshold`) or `low-fit` (below it). Fully automatic. **1 manual touch: pasting the links in.**
2. **`/draft-job --all-shortlisted`** — pre-writes the creative/personalized answers (why this role, why this company, etc.) only for the jobs that made the shortlist — no point drafting for the ones Jev scored low. Fully automatic.
3. **`/shortlist`** — read-only, prints the shortlisted jobs (link, company, role, score) formatted so you can paste them straight into Claude for Chrome, alongside your portfolio URL. **1 manual touch: glancing at it.**
4. **One continuous Claude-for-Chrome session** — you paste the shortlist + portfolio URL once and say "go through these one by one." For each job: it opens your portfolio tab + the job tab, fills common fields straight from your portfolio, uses the pre-drafted answer for creative questions (or writes one fresh if a question wasn't anticipated), then pauses and shows a short confirm summary before submitting. **1 manual touch to start the session, then one quick "yes, go ahead" per shortlisted job.**
5. **Close the loop** — at the end, Claude for Chrome hands you a log of what it submitted; **one bulk command** (`/set-status <id1> <id2> ... submitted`) updates the tracker for the whole batch at once, instead of one command per job.

### Touch count, concretely

For 50 links with a `fit_threshold` of 0.7 (say ~15 clear it): **1 (kick off) + 1 (view shortlist) + 1 (start session) + ~15 (quick confirms) + 1 (bulk status update) ≈ 19 touches** — and 15 of those are seconds-long "yes" replies inside one sitting, not separate setup steps. The one thing that can't shrink further without dropping the safety net entirely is the per-job confirm — that's the deliberate floor, not an oversight.

## Directory structure

```
Hunter/
├── CLAUDE.md                     # non-negotiable rules for any session in this project
├── .claude/commands/
│   ├── add-jobs.md                 # /add-jobs
│   ├── draft-job.md                # /draft-job
│   ├── shortlist.md                # /shortlist
│   ├── queue-status.md             # /queue-status
│   └── set-status.md               # /set-status
├── profile/
│   ├── portfolio.json             # your live portfolio URL + a cached fetched summary
│   ├── preferences.json           # job-search logistics: salary, notice period, work auth, target roles, fit_threshold
│   └── qa-bank.md                 # standard Q&A prose templates, each tagged with a stable id
├── queue/
│   └── queue.json                 # lightweight index: one row per job, current status + fit_score
└── jobs/
    └── <job-id>/
        ├── posting.md              # fetched + extracted requirements
        ├── questions.json          # structured application questions, or "none visible"
        ├── draft-answers.md        # tailored creative answers — pre-written for shortlisted jobs only
        ├── status.json             # per-job state machine + audit history
        └── history/                # prior draft versions, created on redraft
```

There's no local `resume.md` anymore — your live portfolio is the one canonical source for experience/skills, fetched directly by both Claude Code (for scoring/drafting) and Claude for Chrome (live, in-browser). `preferences.json` only holds things a portfolio page typically *doesn't* say out loud: salary expectations, notice period, work authorization, and the `fit_threshold` used for shortlisting.

`job-id` format: `<YYYY-MM-DD>_<company-slug>_<role-slug>` — stable, sorts chronologically, never renamed after creation.

## Data schemas

**`profile/portfolio.json`** — `{ url, fetched_at, summary_cache }`. `summary_cache` is a Claude Code-generated structured summary (skills, experience, target roles as expressed on the site) re-fetched when stale, so Jev has something comparable to score against without re-fetching the live page on every single job.

**`profile/preferences.json`** — salary range, notice period, work authorization, target roles, skills-to-years map (if not well captured by the portfolio), remote preference, `stale_after_days`, and **`fit_threshold`** (currently `0.7`) — the cutoff `/add-jobs` uses to decide `shortlisted` vs. `low-fit`.

**`profile/qa-bank.md`** — unchanged: human-edited prose templates for recurring questions, each tagged with a stable id (`<!-- id: why-this-company --&gt;`) that `/draft-job` cites for provenance.

**`queue/queue.json`** — same lightweight index as before, plus a `fit_score` (0–1) field per row, and status now includes `shortlisted` / `low-fit` alongside the existing values.

**`jobs/<id>/posting.md`, `questions.json`, `draft-answers.md`, `status.json`** — unchanged in shape from the original design; `draft-answers.md` is now only generated for jobs that made the shortlist.

## Commands

- **`/add-jobs <url> [url...]`** — dedup (via `url_hash` against `queue.json`) → `WebFetch` each new posting → extract company/role/requirements/questions → **refresh `profile/portfolio.json`'s cache if stale, then call Jev with (portfolio summary, this job's requirements) → fit_score** → status `shortlisted` if `fit_score ≥ fit_threshold`, else `low-fit` (or `needs-input` on a login-walled/blocked fetch, scored later once content is supplied). Prints a summary: added/duplicate/shortlisted/low-fit/needs-input counts. If the Jev API is unreachable, jobs land as `researched` (unscored) instead of blocking the batch, flagged in `/queue-status` for a rescore.

- **`/draft-job <job-id> [--all-shortlisted] [--refresh] [--force]`** — same behavior as before, but the bulk flag now targets `shortlisted` jobs specifically (not all `researched` ones) — no point drafting for jobs Jev already scored low.

- **`/shortlist [--min-score <n>]`** *(new)* — read-only, lists every `shortlisted` job (id, company, role, score, link) sorted by score descending, plus your portfolio URL at the top, formatted so you can paste the whole block straight into a Claude-for-Chrome session.

- **`/queue-status [--filter <status>]`** — as before, plus surfaces shortlisted/low-fit/unscored counts up top.

- **`/set-status <job-id> [<job-id>...] <approved|rejected|skipped|submitted> ["reason"]`** — now accepts **multiple job-ids in one call**, so the end-of-session cleanup after a Claude-for-Chrome batch is one bulk command instead of one per job. Still validates each transition individually and rejects illegal jumps.

`CLAUDE.md` keeps the same non-negotiables: never auto-approve/auto-submit, never fabricate facts absent from the portfolio/`preferences.json`, never mass-edit `qa-bank.md` silently.

## State machine

```
researched ──(Jev scores fit)──▶ shortlisted (score ≥ fit_threshold)
                              └─▶ low-fit (score < fit_threshold)

shortlisted ──(/draft-job)──▶ drafted
drafted ──(Claude-for-Chrome session + quick confirm)──▶ submitted

any active ──▶ skipped / rejected   (manual override, optional, e.g. you disagree with the score)
needs-input ◀── researched          (login-walled/blocked fetch; rescored once content supplied)
```

`/set-status` rejects any transition not in this table.

## Edge cases

- **Duplicate links**: blocked pre-fetch via `url_hash` match; near-duplicates warned, not auto-skipped.
- **Login-walled posting**: `needs-input`, no fabricated requirements, not scored until real content is supplied.
- **No visible application questions**: `/draft-job` still writes the standard "anticipated" answer set for use mid-session.
- **Portfolio unreachable/stale**: `/add-jobs` warns and either uses the cached `summary_cache` (if not too old) or defers scoring rather than fabricating a fit score.
- **Jev API unavailable/rate-limited**: jobs land as `researched` (unscored), not blocked — `/queue-status` flags "N unscored" and a rescore can be re-run later without re-fetching postings.
- **`fit_threshold` too strict/loose**: rescoring against a new threshold reuses the already-fetched postings and cached portfolio summary — no need to re-fetch anything.
- **Re-drafting after portfolio/`qa-bank.md`/`preferences.json` edits**: `/draft-job` always reads these fresh, archives the previous draft, requires `--force` to redraft an already-`drafted`/`submitted` job.

## Files to create

- `Hunter/CLAUDE.md`
- `Hunter/profile/portfolio.json`, `Hunter/profile/preferences.json`, `Hunter/profile/qa-bank.md`
- `Hunter/queue/queue.json` (empty `{ "version": 1, "jobs": [] }`)
- `Hunter/.claude/commands/add-jobs.md`, `draft-job.md`, `shortlist.md`, `queue-status.md`, `set-status.md`

`jobs/<id>/` folders are created at runtime by `/add-jobs`, not scaffolded up front.

## Verification (once built)

1. Set `profile/portfolio.json`'s `url` to a real portfolio site and `profile/preferences.json`'s `fit_threshold` to `0.7`.
2. Run `/add-jobs` with a batch of real job links (mix of clearly-relevant and clearly-irrelevant roles) — confirm the relevant ones land as `shortlisted` and the irrelevant ones as `low-fit`, each with a `fit_score` in `queue.json`.
3. Run `/add-jobs` again with one repeated link — confirm it's reported as a duplicate, no re-fetch or re-score.
4. Run `/draft-job --all-shortlisted` — confirm `draft-answers.md` is written only for shortlisted jobs.
5. Run `/shortlist` — confirm it prints a clean, pasteable list (portfolio URL + shortlisted links + scores).
6. Manually simulate the Claude-for-Chrome session's end-of-batch log, then run `/set-status <id1> <id2> <id3> submitted` — confirm all three update in one call, and `status.json`/`queue.json` both reflect it.
7. Try an illegal transition (e.g. `/set-status <id> submitted` on a job still `researched`) — confirm it's refused.
