> ⚠️ **HISTORICAL DOCUMENT — INCIDENT RESOLVED**
> 
> This is a historical forensic investigation. The production-starvation incident described in this report **has been resolved**.
> 
> **Incident root cause (documented at the time):** GitHub Actions free-tier minutes exhaustion on the private `FAIT-Blog/scout-agent` repository, which caused workflow runs to be created but receive no runner (`runner_id: 0`) from 2026-08-23 21:15:02 UTC onward.
> 
> This document records the evidence, analysis, and recommendations as they existed at the time of the investigation. It is preserved for historical reference and must not be treated as a statement of current production status. See `CLAUDE.md` §0.1 for the document authority hierarchy.
> 
> **HISTORICAL — Superseded by later sessions (see `CLAUDE.md` §0.1)**. The conclusions in this document describe the state at the time of investigation. They are not current specification. Current behaviour is defined in `CLAUDE.md`.

# PRODUCTION STARVATION FORENSIC

**Repo:** `FAIT-Blog/scout-agent` (private, User account, created 2026-05-31)
**Investigated:** 2026-08-27
**Status:** COMPLETE — evidence-based, zero code/config changes made.

---

## A. Executive Finding

**GitHub Actions free-tier minutes exhaustion on the private `FAIT-Blog/scout-agent` repo**
stopped the entire pipeline at **21:15:02 UTC on 2026-08-23**.

From that instant onward, **every** workflow run on the repo — Scout Agent (Phase 1),
Complete Posts (Phase 2), Homepage Monitor, and Podcast — fails in ~3–5 seconds with no
steps executed and **no runner allocated** (`runner_id: 0`, `runner_group_id: 0`,
`steps: []`, "log not found"). This is the exact same signature as the July 24, 2026
minutes-exhaustion event documented in Session #33, when the account's free-tier private-repo
minutes (2,000/month) were consumed.

The blog therefore published its last post on **2026-08-23 18:19:43 UTC**
("Guillaume Meyer's open-source tool to strip Anthropic's AI watermark…"). Post publishing
cadence was an uninterrupted **4/4/1/4/4 per day** across Aug 19–23, then **zero** from Aug 24.

The cutover is knife-edge: the **103 runs** from Aug 20 → Aug 23 20:14 all succeeded; the very
next scheduled run (21:15) and every run since has failed. The `head_sha` is identical
(`77e9c946`) on both sides of the cutover — **no code change correlates**. All four workflows
killed simultaneously = an account-level resource stop, not a code/feed/provider defect.

**Secondary finding (NOT the cause):** cron-job.org has dispatched **zero** `workflow_dispatch`
events since `2026-08-11T21:00:08Z` — the external triggers that supplied Phase 1/2/podcast still
appear dead, but native `schedule` crons carried the pipeline until the minutes cutover, so cron-job.org
is not the cause. It is a separate reliability gap that must be re-checked once runners return.

---

## B. Run-by-Run Timeline (the cutover)

| Run ID | Workflow | Created (UTC) | Conclusion | Signature |
|---|---|---|---|---|
| 32663806208 | Complete Posts | **2026-08-23 20:14:34** | ✅ success | last success of any kind |
| 32666934651 | Complete Posts | **2026-08-23 21:15:02** | ❌ failure | runner_id 0, steps [], no log — **first failure** |
| 32671402197 | Scout Agent | 2026-08-23 22:14 | ❌ failure | runner_id 0, steps [], no log |
| 32673716394 | Complete Posts | 2026-08-23 23:12 | ❌ failure | runner_id 0, steps [], no log |
| 32677445459 | Scout Agent | 2026-08-24 00:41 | ❌ failure | runner_id 0, steps [], no log |
| 32678089720 | Complete Posts | 2026-08-24 00:54 | ❌ failure | runner_id 0, steps [], no log |
| … | all four workflows | every scheduled slot to 2026-08-27 | ❌ failure | runner_id 0, steps [], no log |

- Runs continue to be **created** on schedule (events land in the runs list), but no hosted runner is
  ever provisioned. That is the documented GitHub behavior when the free-tier minutes pool is exhausted.
- Windows: **Aug 20 00:00 → Aug 23 20:14 = 103 successes / 0 failures**. **Aug 23 21:15 → Aug 27 = 0
  successes / every run failed.** No gradual degradation — a quota cliff.
- Last success per workflow: Scout Agent `32657410318` (18:14:37), Complete Posts `32663806208`
  (20:14:34), Homepage Monitor `32659637695` (18:56:27), Podcast `32623918946` (06:49:13).

**August volume:** 1,042 workflow runs created in August 2026 (event mix: 727 `schedule` +
313 `workflow_dispatch`; the last `workflow_dispatch` was Aug 11 21:00 UTC — cron-job.org).

**Minutes arithmetic (agent-computed):** summing run created→updated deltas, raw August usage ≈
1,527.7 min; with per-job ceil-to-minute rounding ≈ 2,240 min. The free tier for a private repo on
this plan is **2,000 min/month**. Actual billed usage (started→completed) lies between these bounds —
the billing API 404s with this token and the `/timing` endpoint reports `billable.total_ms: 0`
even on successful runs, so an exact line-item figure is not available. The qualitative evidence
(quota-cliff signature, timing ~81% into the month, all workflows at once) is decisive and matches
the Session #33 July precedent where ~2,094 min crossed the same 2,000 cap on ~July 24.

---

## C. Category Funnel under Partial Ops and the Last Successful Production Run

Publishing cadence held until cutover: **Aug 19: 4 · Aug 20: 4 · Aug 21: 1 · Aug 22: 4 · Aug 23: 4**,
then **0** (Aug 24–27 confirmed zero published posts via Supabase).

**Last productive Scout run — `32657410318`, 2026-08-23 18:14:37 UTC:**
- **67 fresh articles** fetched across all sources.
- `industry`: 4 articles processed → **1 post queued** (`pending_posts` id 403, created 18:16:40).
- `anthropic`: 2 articles → **SKIPPED_BEFORE_GENERATION** (SWS 21/100 < 55) — gate working as designed.
- `tools`: 2 articles → both already covered by the event in industry (SWS cross-category dup) — skipped.
- `ai-models`: 0 of 9 articles passed the AI-signal gate — category skipped (authentic, not a defect).
- Result: post 403 → published 18:19:43 UTC. Pipeline was fully healthy at its final breath.

Latest 3 published posts (all pre-cutover): Guillaume Meyer watermark tool (bd48d98a, `industry`,
18:19:43), SkyLens (be0becb0, `industry`, 12:26:09), Nvidia $6B Poolside deal (c260df9a, `industry`, 06:29).

---

## D. P1A Verification (the feature this phase shipped — confirmed working, then starved)

- **Origin registry (`registry.seed.json` @ HEAD `77e9c946`): 24 sources** confirmed via
  `git show origin/main`, including the three P1A additions in both `tools` and `anthropic`:
  - `tools`: **Simon Willison's Weblog** (`simonwillison.net/atom/everything/`) and **Latent Space**
    (`latent.space/feed`) — the category went from 0 → 2 curated sources.
  - `anthropic`: **Anthropic Engineering** (`taobojlen/anthropic-rss-feed` mirror) added to the
    existing **Anthropic News** mirror — 2 curated feeds, **MIN_ARTICLES=2 at the time of this forensic** (current documented value is MIN_ARTICLES=1; see `CLAUDE.md` §6 for current configuration).
- **In production:** the last successful Scout run's log confirms all three sources were fetched
  ("Fetching: Simon Willison's Weblog", "Fetching: Latent Space", "Fetching: Anthropic Engineering") —
  P1A was live and ingesting at the cutover.
- **Live feed health TODAY (2026-08-27):** all three healthy and fresh —
  - Simon Willison's Weblog: HTTP 200, 30 entries, newest **2026-08-26 23:52 UTC**.
  - Latent Space: HTTP 200, 12 items, newest **2026-08-27 01:50 UTC** (2 posts in one hour).
  - Anthropic Engineering: HTTP 200, 25 entries, newest **2026-08-27 08:13 UTC**.
- Conclusion: the sources are producing fresh material daily. The pipeline is starved of a runner,
  not of stories.

---

## E. Provider & Infrastructure Health (all healthy at cutover)

| Layer | State at cutover | Now |
|---|---|---|
| Supabase `pending_posts` | 366 published / 7 failed (all pre-cutover cloudflare image-queue failures, attempts=8, created Aug 15–23) | unchanged — queue empty, nothing stuck |
| LLM providers (Groq GPT-OSS 120B / Gemini fallback) | last profitable run queued + published successfully | N/A (no runs) |
| Image provider (Cloudflare) | publishing normally through Aug 23 | N/A (no runs) |
| Blog frontend (Vercel) | serving normally | unchanged |
| cron-job.org workflow_dispatch | dead since **Aug 11 21:00 UTC** (secondary gap, see A) | still dead |

---

## F. First Confirmed Failure

Run **`32666934651`** — Complete Posts (Phase 2), created **2026-08-23T21:15:02Z**, exactly one
hour after the last success. Job metadata shows the runner was **never allocated**: `runner_id: 0`,
`runner_name: ""`, `runner_group_id: 0`, `runner_group_name: ""`, `steps: []`, ~3-second duration,
conclusion `failure`, job log "log not found". GitHub's documented behavior when a run exists but no
hosted runner can be assigned (free-tier minutes pool empty).

The same exact signature has now repeated on every one of the ~100+ runs since. This is the same
marker that was forensically documented in Session #33 for the July 24 event ("`runner_id: 0` — no
runner allocated").

---

## G. What Is NOT Causing It (ruled out)

1. **Code changes** — `head_sha` (`77e9c946`) identical on the last success and first failure; P1A
   (f6a1523) and P1B (77e9c94) were already deployed and functioning.
2. **Supabase / database** — repo-/table-side queries steady; 366 published rows, clean queue, zero
   stuck `pending_image`; pipeline wrote posts normally right up to the cutover.
3. **LLM providers** — last run queued and published successfully; no provider errors at cutover.
4. **Image generation** — the 7 pre-cutover failed rows are old cloudflare retry-exhaustions
   (Aug 15–23) that never blocked publishing cadence; publishing continued 4/day around them.
5. **Feed breakage / P1A** — all three new sources verified live and fresh TODAY; P1A ingested
   successfully on the last run.
6. **SWS / quality-gate / scoring** — the last run's skips (anthropic SWS 21/100, ai-models 0/9
   AI-signal, tools dup) are the exact designed behavior; a post still queued and published.
7. **cron-job.org** — dead since Aug 11, but the native schedule crons alone kept all workflows
   green until the minutes cutover; if the stop were cron-job.org, Phase 2 would have kept
   publishing (it did until 20:14 Aug 23).
8. **Repo/account Actions being disabled** — runs are still being **created** on schedule; a disabled
   workflow/Actions state would stop run creation entirely. Only runner provisioning stopped.

---

## Closure

```
FORENSIC STATUS: COMPLETE (HISTORICAL)
CODE CHANGES: NONE
CONFIGURATION CHANGES: NONE
RECOMMENDED NEXT ACTION (as recorded at the time of investigation): Restore GitHub Actions
capacity on FAIT-Blog/scout-agent — check Settings → Billing & plans (the free-minutes progress
bar), add a payment method / paid minutes OR wait for the monthly free-tier reset (expected
2026-09-01), then manually dispatch one Scout Agent run and confirm Phase 1 → Phase 2 → publish
completes end-to-end. Also verify the cron-job.org triggers (dead since 2026-08-11) before trusting
them again.

**HISTORICAL CLOSURE NOTE:** The action above is preserved as it was recommended at the time.
This incident was subsequently resolved; the current production status is documented in
`CLAUDE.md` §7A. The recommendation is now historical and superseded by later production evidence;
it is retained here for provenance only.
```