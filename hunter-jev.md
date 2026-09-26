# Jev in Hunter — What It Is, Why It Matters, How to Use It

> **Correction from the earlier version of this doc:** Jev was originally described as deciding individual form fields live, during the Claude-for-Chrome session. That turned out not to be feasible — there's no hook for an outside model to inject decisions into Claude for Chrome's live behavior; it makes its own decisions internally and can't be interrupted mid-session by an external API call. Jev's real, usable role in Hunter is **batch triage, run offline by Claude Code, before any browser session starts** — scoring which of many job links are actually worth pursuing. See [hunter-architecture.md](hunter-architecture.md) for the corrected full picture.

## What is Jev

Jev is TypeSafe AI's first "System One" model, released in limited early access on 2026-09-15 alongside a $40M seed round led by DCVC. It is **not** a chat/generative LLM like Claude — it doesn't produce natural-language text at all.

Instead, Jev takes unstructured state plus a **typed question**, and returns a **typed, probabilistic answer**: a classification, a value, a confidence score — in a single ~50ms forward pass with no token-by-token decoding. The name riffs on Kahneman's "System 1" (fast, intuitive judgment) vs. "System 2" (slow, deliberate reasoning) — Jev is built for fast, narrow, repeated decisions, while an LLM like Claude still does the open-ended reading/writing/reasoning.

Reported performance: ~193–200x faster and ~400–445x cheaper than a comparable LLM call on classification-shaped tasks.

## Jev's actual job in Hunter: batch fit-scoring

You hand Hunter a pile of job links — tens to hundreds at once. Before anyone (you or Claude) spends real time on any of them, `/add-jobs` calls Jev once per new posting:

> *"Given this portfolio summary (your skills, experience, target roles) and this job's requirements — what's the fit score?"*

Jev returns a score (0–1) in about 50ms, for a cost that's negligible even across a hundred postings. `/add-jobs` compares that score against `profile/preferences.json`'s `fit_threshold` (currently `0.7`):

- **Score ≥ threshold → `shortlisted`.** This job gets a pre-drafted creative answer and makes it onto the list you eventually hand to Claude for Chrome.
- **Score < threshold → `low-fit`.** Parked, not deleted — you can manually promote one later if you think Jev under-scored it, but by default it doesn't consume any more of your time or Claude's drafting effort.

That's the entire scope of what Jev does in this system. It never sees a live application form, never fills a field, never writes a sentence of prose.

## Why this matters

- **It's the only place your Jev API access is actually usable today.** Claude for Chrome can't call it mid-session (no hook exists) — but Claude Code, running offline before you open a browser, absolutely can call an external API for each of 50–100 postings.
- **Speed and cost at the volume that actually needs it.** Scoring "is this job worth pursuing" for a hundred postings via a full LLM call each would be slow and non-trivial in cost. Doing it via Jev is seconds, for a cost so low it barely registers — this is exactly the classification-shaped, repeated-many-times task Jev is built for.
- **It protects your actual scarce resource: your time in front of Claude for Chrome.** Out of 50 links, maybe 15 are worth a live session. Jev is what turns "read 50 postings yourself" or "wait on 50 slow LLM calls" into "get a ranked shortlist in seconds."

## What Jev does *not* do here (to be explicit, since the earlier version of this doc got this wrong)

- It does not fill any field on any application form.
- It does not write your "why this role" answer — Claude does, offline, ahead of time, for shortlisted jobs.
- It does not run live during your Claude-for-Chrome session at all. By the time you open the browser, Jev's job for that batch is already finished.

## Status

Jev is in **limited early access**, and this document describes the intended integration — the exact request/response shape of its API isn't something I've verified against TypeSafe AI's actual docs (only public/marketing descriptions), so the real `/add-jobs` integration will need to confirm the real request format once you're wiring in your API key. Until then, or if the Jev API is ever unreachable mid-batch, `/add-jobs` should degrade gracefully: land new jobs as `researched` (unscored) rather than blocking the whole batch, so a rescore can be run later without re-fetching anything.

## Sources
- [Jev (AI model) — Wikipedia](https://en.wikipedia.org/wiki/Jev_(AI_model))
- [Introducing System One Models & Jev — TypeSafe AI Blog](https://typesafe.ai/blog/introducing-system-one-models-and-jev)
- [What Is Jev AI? A Practical Guide to System One and Executable Decisions — Hugging Face](https://huggingface.co/blog/sora-2/what-is-jev-ai-a-practical-guide-to-system-one-and)
- [TypeSafe AI's Jev offers an alternative to LLMs — Tom's Hardware](https://www.tomshardware.com/tech-industry/artificial-intelligence/typesafe-ais-jev-offers-an-alternative-to-llms-that-claims-to-be-193x-faster-and-445x-cheaper-system-one-type-model-is-bespoke-for-probabilistic-decision-making)
- [What is Jev? Inside the new AI model built to make software decisions — Business Standard](https://www.business-standard.com/technology/artificial-intelligence/what-is-jev-inside-the-new-ai-model-built-to-make-software-decisions-126092200452_1.html)
- [Jev Explained: Typesafe AI's Non-Autoregressive System-1 Model — MindStudio](https://www.mindstudio.ai/blog/jev-system-one-model-launch)
