> ⚠️ **HISTORICAL DOCUMENT — NOT CURRENT STATE**
> 
> This is a historical forensic investigation of the Phase 1 story-selection funnel based on 2026-09-01 → 2026-09-23 production evidence. It is retained for historical reference and **is not a description of the current production implementation**.
> 
> For the current production architecture and behavior, see `CLAUDE.md` (governing specification). Where this report documents recommendations, their status is explicitly noted in the “Proposed Actions” section below.
> 
> **HISTORICAL — Superseded by later sessions (see `CLAUDE.md` §0.1)**. The conclusions in this document describe the state at the time of investigation. They are not current specification. Current behaviour is defined in `CLAUDE.md`.

# Selection-Funnel Forensic Report

**Generated:** 2026-09-23 | **Author:** Claude Code (Big Pickle) | **Status:** Read-only audit — no code/config changes, no deploys, no commits

---

## Executive Summary

Audit of the full story-selection funnel of `scout-agent` Phase 1 (2026-09-01 → 2026-09-23, 23 days) against
actual production data: GitHub Actions run logs (90 successful Scout runs), Supabase `scout_memory` (1,175
seen-items), `llm_request_ledger` (783 rows exact-reconciled), `pending_posts` (154 created, 150 published),
and `posts` (150 published).

**Bottom-line yield:** 90 runs × 4 categories = **360 category-slots**. Of those, 267 reached the SWS gate →
228 passed → 145 RSS posts queued (+9 podcast) → **150 published** (0 Phase-2 publish failures). Raw fresh-item
events were 7,962 across the window (avg 88/run) but these over-state supply: AI-signal-rejected items are
**never `markSeen`'d**, so the same non-AI items re-enter the fresh pool every run.

**The dominant defect is upstream of generation, in feed quality — not in the LLM, the gate, or SWS tuning:**

1. **bing.com/news Anthropic feed (added Session #23) is almost unusable.** It produced **376 of 1,175 (32%)**
   of all `scout_memory` marks, yet `fetchContent()` returns only **~19 chars** for its articles (73 of 90 runs
   show `Bing News - Anthropic: 19 chars`). This starves the anthropic category at the SWS gate: **33 of 39 SWS
   rejects across the window are anthropic**, and **22 of 39 reject exactly 8/100** with `primary 18 chars
   (full text)` — the SWS scores are a faithful response to near-empty source text, not a mis-calibration.
2. **The quality gate's `"Why this matters"` check is the #1 post-generation loss.** 132 gate-check events:
   **70 (53%) are the literal missing-"Why this matters" closing** (alone), plus ~40 more combined with
   too-short. 76 of 228 generation attempts (33%) hard-stopped at the gate permanently. All 132 gate first-fails
   happened **after a successful Groq rewrite** — 56 recovered on the one expansion retry, 76 did not.
3. **Cross-run re-processing of the same story wastes generation spend.** The "Pentagon maintains Anthropic
   supply-chain designation" story alone consumed **11 rewrite rows across 3 separate runs** (8 OK / 3 rate_limit
   fails) before being duplicate-skipped post-generation. 8 of the 83 not-queued (run,cat) pairs were
   recent-duplicate skips that had **already burned a full generation** (the dup check runs after generation).
4. **Groq rate-limiting is the second-most-copied failure:** 130 of 490 rewrite rows (27%) failed as
   `rate_limit` before an OK; every rewritten story eventually got one OK, so nothing published was lost, but
   each failure costs 65s of wait + token spend.
5. **Category outcomes reflect source richness, not gate tuning:** industry (richest feeds) queued 60/90,
   tools 36/90, ai-models 29/90, anthropic 20/90. ai-models additionally lost 44 slots to AI-signal skips
   (the Google DeepMind/AI "Proactive cyber defense" item alone recurred in 29 runs).

---

## 1. Source Supply (top of funnel)

| Metric | Value | Evidence |
|---|---|---|
| Successful Scout runs | **90** (4/day Sep 1–22, 2 on Sep 23) | GHA `scout` workflow #286465178 logs |
| Fresh-article events (sum, per run) | **7,962** (avg 88, median 89, min 64, max 127) | `Found N fresh articles across all feeds` lines |
| Unique `scout_memory` marks | **1,175** (all distinct; item re-marks = 0) | Supabase, `processed_at` window |
| `scout_memory` rows by type | 8/8 with post_id-null in window (link unused) | whole-table post_id-not-null = 122 (legacy only) |
| Feed with most marks | **bing.com/news?q=Anthropic = 376 (32%)** | detailed below |
| AI-signal category skips | **74** (ai-models 44, tools 22, anthropic 8, industry 0) | log `Skipping {cat} — only 0 AI-signal` |

**Important observation — the fresh count is inflated:** the AI-signal skip path (scout.js:444-449) does NOT
`markSeen`. Non-AI articles filtered out of `capped` are simply never recorded, so they are reported "fresh"
again on the next run. The Google DeepMind item "Proactive cyber defense for governments and enterprises"
appeared in the sample-rejected line of **29 runs** (58 occurrences) with **zero** `scout_memory` rows. Fresh
events therefore measure fetch traffic, not unique supply.

### Feed-level `scout_memory` marks (top 14)

| Feed | Marks | % of 1,175 |
|---|---|---|
| bing.com/news?q=Anthropic | 376 | 32.0 |
| theverge.com/ai | 89 | 7.6 |
| techcrunch.com/ai | 74 | 6.3 |
| runtimewire.com | 70 | 6.0 |
| news.ycombinator.com/rss | 66 | 5.6 |
| simonwillison.net | 57 | 4.9 |
| openai.com/news/rss | 54 | 4.6 |
| iafrica.com/feed | 47 | 4.0 |
| blog.google/rss | 40 | 3.4 |
| ventureburn.com | 39 | 3.3 |
| aibase.ng | 37 | 3.1 |
| (YouTube podcast channels ×2) | 36 + 20 | 4.8 |
| innovation-village.com | 21 | 1.8 |

---

## 2. Selection Path (per-category slot accounting)

360 category-slots total (90 runs × 4). Full reconciliation:

| Stage | Slots | Notes |
|---|---|---|
| Category-slots | **360** | 90 × industry, ai-models, anthropic, tools |
| AI-signal skip (no AI-signal article) | **74** | items filtered, NOT markSeen, re-enter next run |
| Cross-category same-run dup (pre-search) | **19** | `⏭️ Duplicate of story queued earlier this run` — markSeen ✓ |
| SWS evaluated | **267** | passes 228, rejects 39 |
| Reached generation (≥1 rewrite row) | **228** | every pair had ≥1 OK + 1+ rate_limit retries |
| − Recent-dup skip AFTER generation (8) | | 33527695808/34039740972/33958550596/34025941389/34161161349/34178825768 anthropic + 34339972015/35047752153 tools — dup check runs after the Groq rewrite |
| − Gate hard-stop (retry failed) | | 75 (details §5) |
| Queued to `pending_posts` (RSS) | **145** | industry 60 · tools 36 · ai-models 29 · anthropic 20 |
| Podcast queued (`ai`) | **9** | 23 podcast runs, 23 success |
| Failed pending | **4** | ids 404/407/409 (RSS) + 410 (podcast) — all attempts=8/cloudflare, pre-S49 image bug |
| **Published** | **150** | tools 35 · industry 60 · anthropic 20 · ai-models 27 · ai 8 |

**Consistency check:** ledger gate-check rows (132) = sum of logged gate-retries (54+23+16+39 = 132 ✓).
Queued-from-pending (145 RSS + 9 podcast = 154) = 150 published + 4 failed ✓. Rewrite-pair accounting closes
exactly: 145 queued + 75 gate-hard-stops + 8 post-gen recent-dup skips = **228** ✓. Cross-category same-run dup
skips (19) logged exactly. Ledger rewrite rows that link to a `pending_post_id` map to **145 distinct pending
ids** ✓.

---

## 3. SWS Gate (story-worthiness, enforce mode)

- Evaluations: **267** (min 8, max 88). Threshold 55.
- **Pass ≥55: 228 (85%)** · Reject <55: **39 (15%)**
- Confidence: HIGH 224 · MEDIUM 40 · LOW 3 (all 3 low-confidence at 55-59 band, diagnostics only)
- Buckets: <55 = 39 · 55-64 = 10 · 65-74 = 66 · 75-84 = 133 · 85+ = 19

**Reject anatomy (the selection model's own verdict):**

| Reject trait | Count | Meaning |
|---|---|---|
| Total SWS rejects | **39** | anthropic **33** · industry 3 · tools 3 · ai-models 0 |
| Summary-only (richness 0) | **39 / 39** | every reject lacked usable full text |
| Scoring exactly **8/100** | **22** | `primary 18 chars (full text)` — fetch empty/near-empty |
| SWS 8 in GHA logs | 21 runs | signature of Bing-anthropic starvation |
| Runs with `Bing News - Anthropic: 19 chars` fetch | **73 / 90** | the upstream cause |

**Assessment:** SWS is operating as designed (correctly rejecting thin material). Its rejections are
concentrated in anthropic because that category's dominant feed returns ~19-char bodies. The 22 exact-8 rejects
are a *feed* artifact, not a threshold miscalibration — same conclusion as the 2026-08-22 single-source
forensic, now with 23 days of quantified proof.

---

## 4. Generation (Groq openai/gpt-oss-120b — single-source mode)

- Ledger rewrite rows: **490** = 360 OK + **130 rate_limit fail** (26.5% attempt failure rate).
- Distinct `story_title`s in rewrite rows: **216**. Distinct pending-post joins: **145**.
- Avg per OK call (n=360): **3,430** input tokens (est) · **912** output (est) · **5,692 ms**.
- Window totals (est): **1,234,853** input · **328,443** output across successful rewrites only.
- Provider split: groq 501 (490 rewrite + 11 podcast), pipeline pseudo-rows 282 (132 gate-check + 150
  publish-outcome). **Zero Gemini, zero Tavily** — confirmed single-provider, single-source operation.
- Worst cross-run re-processing: Pentagon-Anthropic story **11 rows / 3 runs** (8 OK / 3 rate_limit),
  King Charles 10 rows, Anthropic small-labs 8 rows.

`TAVILY_SEARCH_ENABLED=false` (single-source) confirmed in every log (`🔒 Tavily story-search OFF`).

---

## 5. Quality Gate (post-generation, the largest loss)

- `gate-check` ledger events: **132** (all `success:false`, `retry_attempt:-1` sentinel).
- Unique stories hitting the gate-check first-fail: **123**.
- Outcomes of the 132 gate-fails: **56 recovered by the one expansion retry** (→ queued) · **75 hard-stopped**
  (→ permanently not queued) · **1 recovered but then recent-dup-skipped** (run 34339972015/tools, Navier-Stokes —
  first attempt "no digest produced", retry recovered the digest, then the post-generation dup check killed it).
  75 hard-stop messages logged ("Content still failing quality gate after one retry").

**Reason histogram (all 132):**

| Reason | Count |
|---|---|
| `missing "Why this matters" closing` (alone) | **70** |
| too short (N < 400) + missing closing | ~40 combined variants |
| too short only | ~15 |
| `no digest produced` | 9 |
| banned word ("shocking"/"incredible") | 2 |
| no h3 headings (+ too short) | 2 |

**Assessment:** 58% of successful LLM rewrites (132/228) failed the Phase-1 gate on first attempt, and a third
of all generation attempts (75/228) ended as hard-stops. The dominant single reason is the literal
`why this matters` heading requirement — a repeatable, high-frequency, structurally-preventable loss.

---

## 6. Publish (Phase 2)

- `publish-outcome` events: **150 all success**, 0 fail — the two-phase pipeline published every row that
  survived Phase 1 queueing.
- Published/day: 3–12 (Sep 3=12, Sep 16=10; no zero days in the window).
- Categories: industry 60 · tools 35 · ai-models 27 · anthropic 20 · ai (podcast) 8.
- 4 failed pending rows (404 Polimill ai-models, 407 OpenAI CA bill ai-models, 409 Wrapture tools, 410 Andrew Ng
  ai/podcast): all `attempts=8`, `provider=cloudflare`, updated Sep 2 — died at the image stage pre-S49 fix.
  Correctly failed; the S49 fix is why no post-Sep-3 row has this failure mode.

---

## 7. Selection Defects — Concrete Examples

### Defect A — Bing Anthropic feed yields ~19-char bodies → 22 exact-8 SWS rejects + starved category
- **Evidence:** run 33649729321 (ai-models? → anthropic): all 4 capitols fetched at `✓ Bing News - Anthropic: 19 chars`, then `[SWS enforce] SWS 8/100`, `Richness 0/100 primary 18 chars (full text)`, `Fact Density 7/100 1 unique entit(ies), 18 total chars`, `Source Diversity 30/100 1 source(s)`.
- 73/90 runs show the same 19-char signature; anthropic = 33 of 39 SWS rejects; bing = 32% of all marks.
- **Impact:** the anthropic category (MIN_ARTICLES=2) uses this feed as one of two sources, and it cannot pass SWS.

### Defect B — `"Why this matters"` gate rejection is the single biggest post-generation loss
- **Evidence:** 70/132 gate-check rows are the bare missing-closing reason; 76/228 generation attempts hard-stop.
- **Impact:** ~1/3 of all Groq rewrites produce content the gate rejects at 400-word/heading parity; every such
  attempt costs a full Groq call + 65s wait.

### Defect C — Cross-run re-processing of the same story (post-generation dup check)
- **Evidence:** "Pentagon maintains Anthropic's supply-chain-risk designation" = 11 rewrite rows across 3 runs
  (Sep 17-18); 8 (run,cat) pairs with rewrite rows, no queue, and either no gate-check or a recovered one =
  post-generation recent-dup skips (recent-dup check at scout.js:673 (historical reference: September 2026 working tree). Current implementation location: scout.js:706 (verified 2026-10-02). runs AFTER generation). Concrete list:
  33527695808/33958550596/34025941389/34039740972/34161161349/34178825768 (anthropic: Claude AI Suite, Pentagon
  ×3, Small Labs ×2) + 34339972015/35047752153 (tools: Navier-Stokes after a recovered gate retry, Gemini 3.8 Live).
- **Impact:** wasted generation spend on already-covered stories.

### Defect D — AI-signal-rejected items never `markSeen` → phantom supply + repeat churn
- **Evidence:** "Proactive cyber defense for governments and enterprises" [Google DeepMind/Google AI] appears in
  sample-rejected lines of **29 runs** (58 occurrences), **0** `scout_memory` rows. ai-models lost 44 slots of
  90 to this path.
- **Impact:** inflated fresh counts (7,962 events for ~1,175 unique), extra fetch work each run; low severity
  (no LLM spend on these) but noisy for any future capacity planning.

---

## 8. Editorial Interpretation (concise)

1. **The pipeline is healthy where it is fed well.** Industry → 60/90 queued; 150/154 pending rows published;
   zero publish-stage failures; zero zero-post days. The two-phase design and the SWS+gate combination work.
2. **The ranking decision is correct; the supply is the bottleneck.** Scout selects the top AI-signal article
   per category and SWS gates it fairly. The categories with thin or broken feeds (anthropic via Bing, i.e.
   19-char bodies; ai-models via Google "flyer" items + 44 AI-signal skips) collapse before generation. Fixing
   the **Bing anthropic source** would directly lift anthropic from 20/90 toward the 60-70% of industry/tools.
3. **Genre for the largest wasted spend is structural, not conceptual:** the literal `"Why this matters"` heading
   accounts for over half of all gate first-fails, and a third of generation attempts hard-stop. This is the highest-
   ROI knobs identified: a prompt/heading-robustness or retry-strategy change converts existing Groq spend into
   published posts.
4. **SWS is evidence of feed health, not a tuning variable.** Its 39 rejects (22 at exactly 8/100) trace to
   empty fetch results; lowering the threshold would only admit 18-char summaries that the gate would reject
   downstream. No SWS weight/threshold change recommended from this window.
5. **Rate-limit churn (130/490 = 27%) is absorbed but costs ~130 × 65s ≈ 2.4 hours of run time/day and tokens;**
   it never lost a post. Only matters as the queue grows.
6. **Net actionable vector (ranked): (1) replace/repair the Bing anthropic feed; (2) gate-robustness around
   "Why this matters"; (3) move the recent-duplicate check before generation (or delete it now that SWS dedups
   same-run + cross-category pre-search); (4) optionally `markSeen` AI-signal-rejected items to de-noise fresh
   counts. None were changed — this window is evidence-only.**

### Proposed Actions — verified status as of 2026-10-02

Status was verified against the `scout-agent` working tree and current `CLAUDE.md` §7A. Historical
recommendation status and current defect status are **not** necessarily the same; both are stated.

| # | Proposed action | Recommendation status | Current defect status |
|---|---|---|---|
| 1 | Replace/repair the Bing-News-Anthropic feed (~19-char bodies) | **OPEN — not implemented** | **STILL OPEN.** No commit removed or repaired the feed; `CLAUDE.md` §7A 🔴 records it as "Still open — not fixed", the largest upstream defect (32% of `scout_memory` marks, 33 of 39 SWS rejects). |
| 2 | Gate-robustness around the literal "Why this matters" requirement | **IMPLEMENTED / CLOSED as an action** — commit `7133ebc` (Session #56) tightened the WTM prompt in all 3 digest paths (4 sites) | **STILL OPEN as a production quality issue.** `7133ebc` is deployed but **measured ineffective**: all 5 post-deployment Phase 1 gate rejections still failed on missing "Why this matters" (7/7 when scoped from `ddb8231`). Additionally `CLAUDE.md` §7A records an active prompt↔gate contradiction: `gemini.js:374` tells the model not to force a WTM section while the gates require the literal substring. The gate-contract change is explicitly deferred. |
| 3 | Move/delete the recent-duplicate check that runs after generation | **OPEN — not implemented** | **STILL OPEN.** The check still runs post-generation in the current working tree (see refreshed line reference above). No commit moved or removed it. |
| 4 | Optionally `markSeen` AI-signal-rejected items to de-noise fresh counts | **OPEN — not implemented (optional/deprioritised)** | **STILL OPEN.** The AI-signal skip path still does not `markSeen`; the fresh-count inflation mechanism described in §1 remains in place. |

---

## 9. Data-Quality Caveats

- **Historical evidence strings.** Log strings quoted throughout this report (e.g.
  `Bing News - Anthropic: 19 chars`, `[SWS enforce] SWS 8/100`, `Richness 0/100 primary 18 chars (full text)`,
  `Fact Density 7/100 ...`, `Source Diversity 30/100 ...`) are **historical evidence strings captured from
  September 2026 production run logs**. They are preserved verbatim as evidence and are not current source
  code or current log wording. For reference, the current SWS enforce path in the working tree logs
  `⏭️  SKIPPED_BEFORE_GENERATION — story worthiness ...` (`scout.js:584`, verified 2026-10-02), and the
  Editorial Eligibility Gate (added later, `ddb8231`) logs a second `SKIPPED_BEFORE_GENERATION` variant
  (`scout.js:614`) that did not exist during this forensic window.

- `fresh` counts are per-run fetch numbers, not unique items (Defect D inflation); unique universe ≥ 1,175.
- Token figures are estimates (~4 chars/token) baked into the ledger, not provider-billed.
- `scout_memory.post_id` is NULL for all 1,175 in-window rows; the 122 legacy rows aren't in this window, so
  post attribution was done via the `llm_request_ledger.pending_post_id` join instead.
- SWS sub-scores (Richness/Fact/Div/Potential) are log-only and were captured from 267/267 evaluations in the
  extracted run logs; the 39 rejects' full breakdown lines were verified against source logs for the examples.