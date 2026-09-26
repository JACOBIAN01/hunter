# Hunter Architecture — Batch Filter (Jev) → One Live Session (Claude for Chrome)

Two stages, two different jobs, run by two different systems:

- **Stage 1 — Jev, offline, batch.** Given a pile of links (tens to hundreds), score each one's fit against your portfolio before anyone spends time on it. Runs entirely inside Claude Code, before you ever touch a browser. Jev never sees a live form and never decides what goes in a specific field — its whole contribution ends the moment the shortlist exists.
- **Stage 2 — Claude for Chrome, live, one continuous session.** Given the shortlist, work through it end to end: reads your portfolio, fills every field itself (common and creative alike — there's no live hand-off to Jev here, it can't be wired into Claude for Chrome's session), pauses once per job for your quick confirm, submits, moves on.

This corrects the earlier version of this document, which had Jev deciding individual form fields live, in-session. That's not actually possible — there's no hook for an outside model to inject decisions into Claude for Chrome's live behavior. Jev's real, usable role is entirely upstream: batch triage before the browser ever opens.

## Stage 1 — Batch intake & filter (offline, fully automatic)

```
   50–100+ job links                    profile/portfolio.json
         │                                (your live portfolio URL
         ▼                                 + a cached summary)
   /add-jobs <all links>                          │
         │                                          │
         ▼                                          │
  dedup against queue.json ──▶ skip repeats          │
         │                                          │
         ▼                                          │
  fetch + extract each new posting                   │
  (company, role, requirements)                      │
         │                                          │
         ▼                                          │
  ┌─────────────────────────────────────────────────┘
  │   call Jev: "given this portfolio summary and
  │              this job's requirements — fit score?"
  ▼
  score ≥ fit_threshold (e.g. 0.7)?
    │                        │
   yes                       no
    │                        │
    ▼                        ▼
 status: shortlisted   status: low-fit
    │                  (parked, not deleted —
    ▼                   can be manually promoted later)
 /draft-job --all-shortlisted
 (pre-writes the creative answers
  only for shortlisted jobs)
```

Output of Stage 1: a `shortlist` — a short list of jobs worth your time, each with pre-written creative answers already sitting in `jobs/<id>/draft-answers.md`. Zero manual touches inside this stage beyond kicking off `/add-jobs` once.

## Stage 2 — One live Claude-for-Chrome session (interactive, human present throughout)

```
  /shortlist  ──▶  prints: portfolio URL + shortlisted links + scores
       │            (you paste this whole block into Claude for Chrome, once)
       ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │             Claude for Chrome — one continuous session             │
  │                                                                     │
  │   for each job in the shortlist, in order:                         │
  │                                                                     │
  │      open portfolio tab + this job's tab                           │
  │                    │                                                │
  │                    ▼                                                │
  │      read portfolio → fill common fields directly                  │
  │      (name, email, phone, work auth, salary, notice period, etc.)  │
  │                    │                                                │
  │                    ▼                                                │
  │      use the pre-drafted answer for creative questions,            │
  │      or write one fresh if a question wasn't anticipated            │
  │                    │                                                │
  │                    ▼                                                │
  │      form fully filled — NOT submitted yet                         │
  │                    │                                                │
  │                    ▼                                                │
  │      ── quick confirm summary shown to you ──                      │
  │      ("filled X, Y, Z — essay answer: [preview] — submit?")        │
  │                    │                                                │
  │                    ▼                                                │
  │              you say "yes" → it submits → logs it → next job       │
  │                                                                     │
  └───────────────────────────────────────────────────────────────────┘
       │
       ▼
  end-of-session log: which jobs were submitted
       │
       ▼
  /set-status <id1> <id2> ... submitted   (one bulk command closes the loop)
```

Manual touches inside Stage 2: **1** to start the session (paste the shortlist), **1 quick "yes"** per shortlisted job before it submits, **1 bulk command** at the end. Nothing else.

## Division of labor

| Question | Answered by | When |
|---|---|---|
| "Is this job even worth pursuing?" | Jev — fit score vs. portfolio | Offline, before the browser opens |
| "What's the candidate's email/phone/salary/notice period?" | Claude for Chrome — reads it straight from the portfolio, live | During the live session |
| "Why do you want to work here?" | Claude — either pre-drafted by Claude Code, or written fresh live if unanticipated | Offline (preferred) or live (fallback) |
| Should this be submitted? | **You — always, once per job, no exceptions** | During the live session, right before each submit |

## Why it's split this way

- Jev never touches a browser and never writes prose — it only scores, which is what makes triaging 50+ postings a few seconds of work instead of an hour of reading or a slow, expensive pile of LLM calls.
- Claude for Chrome never needs Jev mid-session because there's no way to give it that hook today — so it just does the whole live fill itself, common fields included. That's not a compromise, it's the only thing that's actually buildable right now.
- The human touch is concentrated at exactly one point per job — the confirm-before-submit — instead of being spread across "start a session," "review a draft file," and "mark status," which is what the original design had before this round of simplification.

## Current status

Stage 1 (Jev batch scoring) is blocked on Jev API access being wired into `/add-jobs` — a real but small integration once TypeSafe AI's actual API is available to call. Stage 2 (the one-continuous-session flow) should be tested first with a small batch (3–4 links) to confirm Claude for Chrome can actually hold context across multiple tabs/applications in one sitting before relying on it for a full shortlist — this is an assumption, not a confirmed capability.
