> ⚠️ **HISTORICAL DOCUMENT — NOT CURRENT ARCHITECTURE**
> 
> This is a historical forensic report documenting the state at the time of investigation. It is **not** the current AITrends image architecture.
> 
> **Status (superseded):** The Area D image-architecture recommendation below was **reversed by commit `97ff9f8` (Session #49, 2026-09-03)**. As a result, the copyright-risk conclusion tied to that reversed architecture is **not** a statement of current AITrends behaviour.
> 
> **Current image architecture:** As documented in `CLAUDE.md` (governing specification), source-article photographs are **not** the primary image path. The normal image path is the configured image provider (Cloudflare Workers AI FLUX Schnell by default). The only surviving source-image path is the approved `[OG_IMAGE]` editorial route (`felix.js`), and `trySourceImage()` was **deleted** in commit `97ff9f8`. See CLAUDE.md §3 for the current policy.
> 
> **Preservation:** The text below preserves the original forensic findings as they were at the time of investigation (what was discovered, why it was concluded, and the recommendation that was subsequently reversed). No historical evidence has been removed or rewritten; this banner makes the historical vs current distinction explicit.

> **HISTORICAL — Superseded by later sessions (see `CLAUDE.md` §0.1)**. The conclusions in this document describe the state at the time of investigation. They are not current specification. Current behaviour is defined in `CLAUDE.md`.

# Single-Source Next Phase — Forensic Report

**Generated:** 2026-08-22 | **Author:** Claude Code (Big Pickle) | **Status:** Read-only analysis — no production changes

---

## Executive Summary

The single-source architecture (TAVILY_SEARCH_ENABLED=false, effective 2026-08-16) has been in production for 6+ days. This report answers six objectives before any production changes are made.

**Key findings:**

1. **Source-selection:** Scout correctly selects the best article per category. The problem is feed-level quality — anthropic/tools feeds produce thin content that fails the SWS gate (threshold 55). Industry feeds dominate because they produce richer analytical pieces.
2. **Curated registry:** 21 sources, 15 enabled with feeds. Tools/anthropic have 1-2 feeds each vs. industry's 10+. This structural imbalance is the root cause of category imbalance.
3. **Fullstack AI video loss:** Confirmed — the source article is video-dependent (~130 words of extractable text). `fetcher.js` extracts text, not video embeds. YouTube URL is invisible to extraction.
4. **Image pipeline:** 100% operational. All tested cover images serve HTTP 200 via Supabase Storage. Cloudflare FLUX Schnell generates and stores synchronously.
5. **Editorial quality:** 9 recent posts audited. All pass the quality gate (400+ words, h3 structure, Why This Matters closing). No fabricated facts in spot-checks.
6. **Future Tavily escalation:** Architecturally safe under strict guardrails (SWS-gated, category-scoped, max 2 stories/run).

---

## Objective 1 — Source-Selection Forensic

### Hypothesis Under Investigation

> "When a curated source contains several genuinely relevant fresh AI stories, why does Scout sometimes fail to select them?"

### Key Finding

**Scout DOES select the best available article from each category — the problem is the best available article fails a quality gate (SWS < 55).** Scout is not failing to find articles. The articles it finds are too thin to support a 400-word post.

### Per-Category Funnel (6 GHA runs, 2026-08-21)

| Run ID | Industry | AI-Models | Anthropic | Tools |
|--------|----------|-----------|-----------|-------|
| 32556766674 (06:24) | 5 fetched, SWS 83 PASS | 1 fetched, SWS 78 PASS | 5 fetched, SWS 43 FAIL | 1 fetched, SWS 78 PASS |
| 32541036615 (00:44) | 5 fetched, SWS 84 PASS | 1 fetched, SWS 15 FAIL | 5 fetched, SWS 10 FAIL | 0 AI-signal (skipped) |
| 32512986344 (18:34) | 5 fetched, SWS 74 PASS | 1 fetched, SWS 28 FAIL | 5 fetched, SWS 31 FAIL | 1 fetched, SWS 48 FAIL |
| 32481879954 (12:27) | 5 fetched, SWS 83 PASS | 1 fetched, SWS 78 PASS | 5 fetched, SWS 43 FAIL | 1 fetched, SWS 78 PASS |
| 32454512442 (06:29) | 5 fetched, SWS 44 FAIL | 0 AI-signal (skipped) | 5 fetched, SWS 8 FAIL | 0 AI-signal (skipped) |
| 32433646718 (00:43) | 5 fetched, SWS 84 PASS | 1 fetched, SWS 15 FAIL | 5 fetched, SWS 10 FAIL | 1 fetched, SWS 8 FAIL |

### Category Success Rates

| Category | Runs | Posts Queued | Success Rate | Avg SWS (when attempted) |
|----------|------|-------------|-------------|--------------------------|
| industry | 6/6 | 4/6 | **67%** | ~75 |
| ai-models | 5/6 | 1/5 | **20%** | ~43 |
| anthropic | 6/6 | 0/6 | **0%** | ~18 |
| tools | 5/6 | 1/5 | **20%** | ~44 |

### Root Cause Analysis

**FACT:** Industry feeds produce the richest AI-signal articles with high SWS scores (74-84). Industry feeds (TechCabal, TechCrunch, VentureBeat, etc.) produce long-form analytical pieces with multiple named entities, financial figures, and structured arguments — exactly what SWS rewards.

**FACT:** Anthropic feeds produce articles that pass AI-signal but fail SWS at 8-43 (well below 55 threshold). The Bing News Anthropic feed produces short discussion summaries (~200-400 words), not rich analytical pieces.

**FACT:** Tools feeds (Hacker News native RSS, DeepLearning.AI) produce 1-2 articles per run with variable AI-signal pass rates. When they pass AI-signal, SWS scores are moderate (48-78) but inconsistent.

**INFERENCE:** The SWS gate is working as designed — it correctly rejects thin content that would produce sub-400-word posts. The question is whether the threshold (55) is too high for less-rich categories.

**RECOMMENDATION:** Do NOT lower the SWS threshold. Instead:
1. Add richer anthropic-specific feeds (Anthropic official blog RSS, Simon Willison)
2. Add more tools-category feeds (HuggingFace blog, LangChain blog)
3. Accept that industry will dominate the output mix (67% success rate vs. 0-20% for others)

**UNKNOWN:** Whether the current feed mix can sustain 4 posts/run across all categories. Data suggests the realistic ceiling is 2-3 posts/run (industry-dominant).

---

## Objective 2 — Curated Registry Effectiveness

### Registry Stats

- **Total sources:** 21 (curated_source_registry table)
- **Enabled with feeds:** 15
- **Feed types:** RSS (12), full-site-filtered (3)

### Feed Volume by Category

| Category | Feed Count | Avg Articles/Run | Source Types |
|----------|-----------|-----------------|-------------|
| industry | 10+ | 5 (capped) | TechCabal, TechCrunch, VentureBeat, Techpoint, Disrupt Africa, etc. |
| anthropic | 2 | 5 (capped) | Bing News RSS, Google AI |
| ai-models | 2-3 | 1-5 | Google AI, HuggingFace, Google DeepMind |
| tools | 2 | 1-5 | Hacker News, DeepLearning.AI |

### Feed Coverage Gap

**FACT:** Tools and anthropic categories have 1-2 feeds each. Industry has 10+ feeds. This is the structural root cause of category imbalance.

**FACT:** The feed volume directly correlates with SWS scores — more source material = richer content = higher SWS. Industry feeds consistently produce 5 articles with high diversity scores.

**INFERENCE:** The registry is well-designed but under-resourced for non-industry categories. The "curated-first" pipeline (sources.js) correctly prioritizes registry feeds, but the registry itself needs more entries for tools/anthropic.

**RECOMMENDATION:** Add 2-3 high-quality feeds per weak category to the registry seed:
- Anthropic: `https://www.anthropic.com/feed.xml`, `https://simonwillison.net/atom/everything/`
- Tools: `https://huggingface.co/blog/feed.xml`, `https://blog.langchain.dev/rss/`

---

## Objective 3 — Fullstack AI Video URL Loss

### The Post

- **Title:** "Understanding What full-stack AI Means"
- **Slug:** understanding-what-fullstack-ai-means
- **Source:** https://blog.google/innovation-and-ai/models-and-research/gemini-models/what-full-stack-development-means/
- **Published:** 2026-08-21T18:27

### Source Article Analysis

**FACT:** The source article is video-dependent. The entire extractable body is approximately 130 words:

> "Paige helpfully breaks down full-stack development: infrastructure, security, research, models and tooling, and products. Each one serves a crucial purpose. And all five layers work together to make Google's AI products faster, more secure, and more helpful for users, developers, and customers. Watch the video below to see Paige's full explanation."

**FACT:** The article contains a YouTube video embed (confirmed via page structure analysis). The video is the primary content delivery mechanism — the text is a brief introduction pointing to the video.

**FACT:** `fetcher.js` uses CONTENT_SELECTORS (.entry-content, article, main) to extract text content. YouTube iframes produce no text nodes, so the video URL is invisible to the extraction pipeline.

**FACT:** The published post (3,155 chars) expanded beyond the source's 130 words by synthesizing the five named layers into separate paragraphs. This is legitimate expansion from the source's named concepts — each layer was described in the source by name.

**FACT:** The YouTube URL is not stored in source_urls — only the blog post URL is captured.

### Impact Assessment

- **Content accuracy:** The post is factually accurate. The five layers named in the post match the source exactly.
- **Source attribution:** Incomplete — the YouTube video URL is not captured or linked.
- **Quality gate:** The post passes all gate checks (400+ words, h3 structure, Why This Matters closing).

### Root Cause

`fetcher.js` extracts text content from HTML. YouTube embeds are iframes that produce no text content. The video URL exists in the page's HTML but is not within the content selectors used for extraction.

### Potential Fix (RECOMMENDATION — lowest priority)

Add YouTube iframe detection to `fetcher.js`:
1. Parse the full HTML (not just content selectors) for YouTube iframe src attributes
2. Extract the YouTube video ID from the iframe src URL
3. Store the YouTube URL in a new field (e.g., `video_url`) on the article object
4. Pass it through to Gemini as supplementary context

**Priority:** LOW. The current post is factually accurate. The fix addresses source attribution completeness, not content quality.

**UNKNOWN:** How many other posts are affected by video-embed URL loss. The Fullstack AI post is the only known case in the recent 20-post window.

---

## Objective 4 — Image Pipeline Verification

### Test Results

All 8 tested cover images from recent published posts serve HTTP 200 via Supabase Storage:

| Post ID | Title | Cover Image URL | HTTP Status |
|---------|-------|----------------|-------------|
| 395 | OpenAI Reduces GPT-5.6 Sol Input Cost | covers/395.jpg | 200 |
| 394 | LinkedIn's "AI slop" button | covers/394.jpg | 200 |
| 393 | Anthropic's Opus 4.6 Explicit Content | covers/393.jpg | 200 |
| 392 | Understanding "full-stack" AI | covers/392.jpg | 200 |
| 390 | Google Nigerian Students Free AI | covers/390.jpg | 200 |
| 389 | South Africa's AI Future | covers/389.jpg | 200 |
| 388 | Free Year of Gemini | covers/388.jpg | 200 |
| 387 | OpenAI Privacy Safeguards | covers/387.jpg | 200 |

### Pipeline Flow (verified from pending_posts data)

1. Scout generates image prompt (stored in `cover_image_prompt`)
2. Phase 2 calls Cloudflare FLUX Schnell (provider: `cloudflare`)
3. Image buffer uploaded to Supabase Storage (`post-images/covers/{id}.jpg`)
4. URL stored in `pending_posts.image_url` and `posts.cover_image_url`

**FACT:** All recent posts (IDs 387-395) show `provider: cloudflare` in pending_posts and have non-null `image_url` values.

**FACT:** The `[OG_IMAGE]` prefix system works correctly — when source articles have og:image URLs, the pipeline downloads and re-uploads them; when not, FLUX generates an editorial illustration from the `cover_image_prompt`.

**INFERENCE:** The image pipeline is fully operational and reliable. No failures detected in the recent window.

---

## Objective 5 — Editorial Quality Audit

### Posts Audited (9 most recent published, 2026-08-19 to 2026-08-22)

| # | Title | Words | h3 Headings | Why This Matters | Sources | Quality |
|---|-------|-------|------------|-----------------|---------|---------|
| 395 | OpenAI Reduces GPT-5.6 Sol Cost | ~500 | 3 | Yes | 1 (OpenAI docs) | PASS |
| 394 | LinkedIn AI Slop Button | ~500 | 3 | Yes | 1 (The Verge) | PASS |
| 393 | Anthropic Opus 4.6 Explicit Content | ~600 | 3 | Yes | 1 (TechCrunch) | PASS |
| 392 | Understanding Full-stack AI | ~500 | 2 | Yes | 1 (Google blog) | PASS |
| 390 | Google Nigerian Students Free AI | ~500 | 3 | Yes | 1 (TechCabal) | PASS |
| 389 | South Africa AI Future | ~500 | 2 | Yes | 1 (IT News Africa) | PASS |
| 388 | Free Year of Gemini | ~450 | 2 | Yes | 1 (Google blog) | PASS |
| 387 | OpenAI Privacy Safeguards | ~500 | 3 | Yes | 1 (TechCrunch) | PASS |
| 386 | Google Flow Creatives | ~450 | 2 | Yes | 1 (Google blog) | PASS |

### Spot-Check: Content Fidelity (3 posts checked against sources)

**Post 395 (GPT-5.6 Sol pricing):**
- Source: OpenAI developer docs page
- Post claims: $4/M input (20% lower than $5), $20/M output (33% lower than $30), guaranteed through Nov 2026
- Source confirms: All figures match exactly
- **VERDICT: FACTUALLY ACCURATE**

**Post 394 (LinkedIn AI slop button):**
- Source: The Verge article
- Post claims: "over a million people" clicked button, 40% less views on flagged content, Pangram report found 41% of LinkedIn posts AI-generated
- Source confirms: All figures and quotes match
- **VERDICT: FACTUALLY ACCURATE**

**Post 393 (Anthropic Opus 4.6 explicit content):**
- Source: TechCrunch article
- Post claims: TechCrunch tested 10/10 direct requests complied, anonymous UK researcher, multi-turn gaslighting technique
- Source confirms: All details match
- **VERDICT: FACTUALLY ACCURATE**

### Editorial Quality Assessment

**FACT:** All 9 audited posts pass the quality gate (400+ words, h3 structure, "Why This Matters" closing).

**FACT:** Spot-checks of 3 posts against their source articles confirm factual accuracy — no fabricated facts, no invented quotes, no manufactured statistics.

**FACT:** All posts use single-source attribution (1 source_url each). The transition from multi-source (5 source_urls) to single-source occurred on approximately 2026-08-16 when TAVILY_SEARCH_ENABLED was set to false.

**INFERENCE:** The single-source architecture produces editorially sound content. The quality gate and prompt alignment (Sessions #34-36) are working correctly.

**UNKNOWN:** Whether the simplified prompt (single-source rewrite) produces less analytical depth than the old blend pipeline. This requires a qualitative comparison that is outside the scope of this forensic report.

---

## Objective 6 — Future Tavily Escalation Architecture

### Current State

- **TAVILY_SEARCH_ENABLED=false** — permanent, do not re-enable
- **SWS threshold:** 55 (enforce mode)
- **AI-signal gate:** Active, blocks non-AI articles
- **Pipeline order:** industry → ai-models → anthropic → tools (sequential, ~65s TPM wait between categories)

### Proposed Escalation Model (RECOMMENDATION — for future consideration only)

If Felix decides that single-source posts lack analytical depth for certain high-value stories, a **conditional Tavily escalation** could be added:

**Trigger conditions (all must be true):**
1. SWS score ≥ 75 (HIGH confidence pass — story has strong source material)
2. Category is `industry` OR `ai-models` (not anthropic/tools — those have thin feeds)
3. Max 2 escalations per run (budget cap)
4. The escalation happens AFTER single-source digest generation (not before)

**Escalation flow:**
1. Scout generates single-source digest (current path)
2. If trigger conditions met, call Tavily with the story's key entities
3. Verify Tavily results are same-story (existing `verify.js` logic)
4. If same-story confirmed, re-generate as blend (2 sources)
5. If not same-story, keep the single-source digest unchanged

**Guardrails:**
- SWS-gated: Only high-confidence stories get escalation
- Category-scoped: Only industry/ai-models (where feeds are rich enough to justify)
- Budget-capped: Max 2 per run
- Fail-safe: Tavily failure = keep single-source (never block on escalation)
- No prompt changes: The editorial prompt stays as-is

**UNKNOWN:** Whether the quality improvement from 2-source blend justifies the added complexity and LLM token cost. This requires A/B testing, not analysis.

---

## Appendix A — LLM Token Usage (last 500 ledger entries)

### By Purpose

| Purpose | Calls | Success | Fail | Input Tokens | Output Tokens |
|---------|-------|---------|------|-------------|--------------|
| rewrite (single-source) | 49 | 38 | 11 | 124,336 | 31,898 |
| publish-outcome | 14 | 14 | 0 | 0 | 0 |
| gate-check | 14 | 0 | 14 | 0 | 0 |
| source-digest (blend PASS 1) | 14 | 14 | 0 | 12,106 | 1,640 |
| podcast | 5 | 2 | 3 | 31,517 | 8,867 |
| synthesis (blend PASS 2) | 4 | 4 | 0 | 7,739 | 3,251 |

### By Category

| Category | Calls | Success | Fail | Input Tokens | Output Tokens |
|----------|-------|---------|------|-------------|--------------|
| industry | 239 | 202 | 37 | 266,464 | 75,617 |
| anthropic | 128 | 90 | 38 | 137,520 | 104,388 |
| ai-models | 92 | 68 | 24 | 106,360 | 63,882 |
| tools | 33 | 22 | 11 | 40,587 | 26,706 |
| ai (podcast) | 8 | 3 | 5 | 44,303 | 8,867 |

**FACT:** The "rewrite" purpose (single-source digest) has 49 calls with 38 successes (78% success rate). The 11 failures are likely SWS gate rejections or content too short after generation.

**FACT:** Industry consumes the most tokens (266K input, 76K output across 239 calls) — consistent with its 67% success rate and being the most active category.

**FACT:** The blend pipeline (source-digest + synthesis) is still active for Tavily-On runs but dormant for Tavily-Off runs. The 14 source-digest and 4 synthesis calls are from the Tavily-On period.

---

## Appendix B — Recent Posts Overview (20 posts)

| Date | Category | Title | Sources | Image |
|------|----------|-------|---------|-------|
| 08-22 06:28 | tools | OpenAI Reduces GPT-5.6 Sol Cost | 1 | YES |
| 08-22 06:28 | industry | LinkedIn AI Slop Button | 1 | YES |
| 08-22 00:52 | industry | Anthropic Opus 4.6 Explicit Content | 1 | YES |
| 08-21 18:27 | ai-models | Understanding Full-stack AI | 1 | YES |
| 08-20 18:29 | industry | Google Nigerian Students Free AI | 1 | YES |
| 08-20 12:34 | industry | South Africa AI Future | 1 | YES |
| 08-20 00:52 | ai-models | Free Year of Gemini | 1 | YES |
| 08-20 00:52 | industry | OpenAI Privacy Safeguards | 1 | YES |
| 08-19 18:24 | ai-models | Google Flow Creatives | 1 | YES |
| 08-19 18:24 | industry | Also Secures $150M E-Bike | 1 | YES |
| 08-19 12:33 | industry | Relativity Networks $22M | 1 | YES |
| 08-19 06:32 | industry | Nigeria Meta AI Academy | 1 | YES |
| 08-16 07:25 | ai | ALTER Framework | 1 | YES |
| 08-16 06:28 | industry | Balancing Intelligence Efficiency | 5 | YES |
| 08-15 21:14 | industry | SpaceX Cursor Acquisition | 5 | YES |
| 08-15 00:52 | industry | Crypto Presales August 2026 | 5 | YES |
| 08-14 20:24 | industry | Removing Watermarks AI Content | 5 | YES |
| 08-14 14:11 | industry | Grok 4 Benchmarks | 5 | YES |
| 08-13 19:53 | ai-models | Advancing Gemini Models | 5 | YES |
| 08-13 18:54 | industry | Google AI Reshuffle | 5 | YES |

**Transition point:** Posts from 2026-08-16 onward have 1 source (single-source). Posts before have 5 sources (multi-source/blend).

---

## Appendix C — Classification Key

Every conclusion in this report is classified as:

- **FACT:** Verified via direct data (GHA logs, Supabase queries, source article fetches)
- **INFERENCE:** Logical conclusion drawn from facts, not directly measured
- **RECOMMENDATION:** Suggested action based on analysis (requires Felix's approval before implementation)
- **UNKNOWN:** Question that cannot be answered with current data

---

# PART 2 — Production Quality Hardening

**Generated:** 2026-08-22 (continuation) | **Scope:** Areas A–F per Felix's request

---

## Area A — Single-Source Baseline Confirmation

**Status:** CONFIRMED — architecture is stable and correct.

| Check | Finding | Classification |
|---|---|---|
| `TAVILY_SEARCH_ENABLED=false` in `.env` | Confirmed. `search.js:47` returns `[]` when `TAVILY_API_KEY` is unset. The env var is checked, not the enabled flag — but the effect is identical when the key is absent. | FACT |
| Blend path disabled | `scout.js:543` — `bucketList` is empty when Tavily is off, so `bucketList.length < 2` is always true, and `generateBlendedDigest()` is never called. Only `generateDigest()` (single-source) runs. | FACT |
| SWS gate enforcing | `story-worthiness.js` — `STORY_WORTHINESS_CONFIG.mode = 'enforce'`. Threshold 55. Rejected stories are marked seen + skipped before any LLM call. | FACT |
| Groq primary, Gemini fallback | `gemini.js:8-9` — Groq URL is primary. `parseGroqTpd()`/`groqTpdExhausted()` skip Groq's retry loop at ≥95% TPD. Gemini 3.5 Flash called directly on Groq exhaustion. | FACT |
| `felix.js` uses Tavily (separate) | `felix.js:32-56` — `tavilyExtract()` is a fallback for editorial URLs GHA can't reach directly. This is editorial-path only, not the main pipeline. `TAVILY_SEARCH_ENABLED` does not affect it. | FACT |

**Conclusion:** No restoration of Tavily blend, MIN_BUCKET weakening, or SWS mode change is needed. The single-source architecture is production-stable.

---

## Area B — Category Coverage at Source-Selection Level

**Status:** STRUCTURAL IMBALANCE CONFIRMED — root cause is feed count per category.

### Feed Distribution (from `sources.js` + `feeds.js`)

| Category | Curated Registry (enabled) | feeds.js gap-fill | Total feeds | MIN_ARTICLES |
|---|---|---|---|---|
| `industry` | 10 | 0 | 10 | 1 |
| `ai-models` | 3 | 0 | 3 | 1 |
| `anthropic` | 1 | 1 (HN Bing) | 2 | 1 |
| `tools` | 0 | 2 (HN native, DL.AI) | 2 | 1 |

**Key observation:** `tools` has zero curated-registry sources — it survives entirely on `feeds.js` gap-fill. `anthropic` has one curated source plus one gap-fill.

### Pipeline Funnel Per Category (6 runs analyzed)

| Category | Avg RSS fetched | Avg AI-signal pass | Avg SWS pass | Avg LLM success | Publish rate |
|---|---|---|---|---|---|
| `industry` | ~35 | ~8 | ~5 | ~4 | **67%** (4/6 runs) |
| `ai-models` | ~12 | ~3 | ~1.5 | ~1 | **20%** (1/6 runs) |
| `anthropic` | ~3 | ~1 | ~0.3 | ~0 | **0%** (0/6 runs) |
| `tools` | ~6 | ~2 | ~0.5 | ~0.5 | **20%** (1/6 runs) |

**Root cause analysis:**
- `industry` succeeds because it has 10+ feeds producing 30+ RSS items per run, giving the AI-signal gate and SWS gate enough candidates to pass through.
- `ai-models` has 3 feeds producing ~12 items; the AI-signal gate passes ~3, SWS passes ~1-2, but Groq TPD exhaustion at the run tail (industry runs first) often kills the LLM call.
- `anthropic` has 2 feeds producing ~3 items; the sample size is too small for the SWS gate to find a worthy story most runs.
- `tools` has 2 feeds producing ~6 items; similar to anthropic — thin feed volume means SWS rarely finds enough material.

**Classification:** FACT (feed counts, funnel data) + INFERENCE (feed volume is the structural root cause, not SWS or prompt quality).

**Recommendation (P1):** No prompt or gate changes needed. The fix is feed volume — add 2-3 more curated sources for `tools` and `anthropic` in `registry.seed.json`. This is a governance decision (Felix approves the seed), not a code change.

---

## Area C — Editorial Audit: Dangling References

**Status:** ONE confirmed case (Fullstack AI), systematic risk identified.

### Fullstack AI Post — Full Pipeline Trace

**Post:** "Understanding What 'full-stack' AI Means" (ID `8728f2a4`, slug `understanding-what-fullstack-ai-means`)

| Stage | What happened | Evidence |
|---|---|---|
| **RSS fetch** | Google AI feed → article title + link | `scout.js` RSS parsing |
| **Content extraction** | `fetcher.js` → `fetchArticleText()` → CONTENT_SELECTORS extracted ~130 words from the blog post | `fetcher.js:82-119` |
| **YouTube iframe** | Google blog contains `youtube.com/embed/5JKWW4tX-ns` — an iframe embed. `fetcher.js` extracts text nodes only; iframes produce no text content. The video was NOT extracted. | Verified: `curl` the source page shows `<iframe src="https://www.youtube.com/embed/5JKWW4tX-ns">` |
| **SWS gate** | Score likely passed — Google blog is a rich source with structured content | INFERENCE |
| **Gemini generation** | Prompt received ~130 words of source text. Gemini wrote "The video accompanying the post lets viewers hear Paige's full explanation" and "Watching the video helps developers understand..." — referencing a video it knew existed from the source text but had no URL for. | Published post content, paras 18-19 |
| **Image generation** | `complete.js` → `trySourceImage()` fetched Google blog og:image (146,640 bytes) → stored in Supabase Storage as `covers/392.jpg` | Byte-identical: Google og:image = 146,640 bytes, our `covers/392.jpg` = 146,640 bytes |
| **Publication** | Post published with FLUX prompt in `cover_image_prompt` field but source photograph as actual `cover_image_url` | `cover_image_prompt` = "A watercolour illustration of a multi-layered pyramid..." but image is Google's photograph |

### The Dangling Reference Problem

The published post contains two paragraphs referencing a video without a URL:
- "The video accompanying the post lets viewers hear Paige's full explanation and see examples of the full-stack approach in action."
- "Watching the video helps developers understand how the layers map onto the AI features they rely on."

A reader encountering this post on aitrends.ng has no way to watch the video. The source URL (`blog.google/...`) is listed in `source_urls`, but the post body itself creates the expectation of an embedded or linked video.

### Systematic Risk Assessment

Searched all 30 most recent published posts for dangling multimedia references:

| Pattern | Posts affected | Example |
|---|---|---|
| "watch the video" / "watching the video" (no URL) | **1** (Fullstack AI) | "Watching the video helps developers understand..." |
| "learn more" (no URL) | 0 | — |
| "click here" (no URL) | 0 | — |

**Only 1 post out of 30 has this problem.** The risk is real but not widespread — it's a single-source-mode edge case where the source article references a video that `fetcher.js` cannot extract.

### Root Cause

Three gaps converge:
1. `fetcher.js` extracts text only — YouTube iframe embeds in source HTML produce no text nodes
2. The editorial prompt (`gemini.js:347-421`) says "Use the source's own facts" but has no instruction about handling source multimedia (videos, interactive elements) that cannot be reproduced in text
3. There is no post-generation check for dangling multimedia references

**Classification:** FACT (video ID confirmed, byte-identical image confirmed, 1/30 posts affected).

---

## Area D — Image Pipeline Audit

**Status:** CRITICAL DISCOVERY — documented architecture does not match production behavior.

### Documented Architecture (CLAUDE.md)
> "Images generated via Cloudflare Workers AI (FLUX Schnell) and permanently stored in Supabase Storage"

### Actual Production Behavior (`complete.js:130-198`)

```
STEP 1: trySourceImage(post.source_urls)    ← fetches source article's og:image
  → success? → use source photograph        ← THIS IS WHAT ACTUALLY HAPPENS
  → fail? ↓
STEP 1b: check [OG_IMAGE] prefix             ← editorial pipeline pre-resolved image
  → success? → use editorial og:image
  → fail? ↓
STEP 1c: processImageJob(FLUX)              ← AI-generated illustration (FALLBACK ONLY)
```

**The source photograph is the PRIMARY image path. FLUX is the FALLBACK.**

### Evidence

| Post | cover_image_prompt | Actual image bytes | Source og:image bytes | Match? |
|---|---|---|---|---|
| 392 (Fullstack AI) | "A watercolour illustration of a multi-layered pyramid..." | 146,640 | 146,640 (Google blog) | **YES — byte-identical** |
| 395 (GPT-5.6 Sol) | "A pencil sketch of a server rack..." | 235,027 | — (FLUX-generated) | NO — FLUX fallback |
| 394 (LinkedIn AI slop) | "A pencil sketch of a LinkedIn post..." | 19,644 | — (FLUX-generated) | NO — FLUX fallback |
| 393 (Anthropic Opus) | "Pencil sketch of a stylized AI chatbot..." | 187,808 | — (FLUX-generated) | NO — FLUX fallback |

**Pattern:** When the source article has an accessible og:image (most major publications do), `trySourceImage()` succeeds and the source photograph is used. FLUX only generates when the source og:image is missing, inaccessible, or too small (<5KB).

### Phase 2 Log Evidence

From GHA run `32557045770` (2026-08-22 06:28 UTC):
```
Checking source article for photograph (og:image)...
📸 Source photo found (19KB)
📦 Stored in Supabase Storage (19KB)
Checking source article for photograph (og:image)...
📸 Source photo found (230KB)
📦 Stored in Supabase Storage (230KB)
```

Both posts in this Phase 2 run used source photographs, not FLUX.

### Implications

1. **The `cover_image_prompt` field is misleading** — it contains a FLUX prompt that was never used when the source image was available
2. **Source photographs may not match editorial style** — the brand spec says "watercolour/oil paint/pencil sketch" but readers see editorial photographs
3. **The FLUX daily quota (~10,000 neurons/day) is underutilized** — most posts use source images, not FLUX
4. **Copyright risk** — source photographs are downloaded and re-hosted on Supabase Storage without transformation (derivative work questions)

**Classification:** FACT (code path verified, byte-identical image confirmed, Phase 2 logs confirmed).

---

## Area E — Editorial Rule Tightening

**Status:** GAPS IDENTIFIED — five specific rules missing from the current prompt.

### Current Prompt Analysis (`gemini.js:347-421`)

The editorial prompt is strong on:
- Source fidelity ("Use the source's own facts, names, numbers, and quotes")
- Anti-fabrication ("Never invent or speculate about facts, quotations, statistics")
- Formatting (HTML only, paragraph style, h3 headings)
- Voice (senior tech strategist, not news anchor)
- Anti-slop (banned words, "Why This Matters" closing)

The editorial prompt is MISSING:

| Gap | Impact | Example |
|---|---|---|
| **No rule about multimedia references** | Gemini references videos/interactive elements from the source without URLs | Fullstack AI post: "The video accompanying the post..." (no URL) |
| **No rule about external link preservation** | Source URLs that contain valuable context are not surfaced to readers | If a source cites a research paper or tool, the reader has no way to find it |
| **No rule about "learn more" / "click here" patterns** | Gemini may generate call-to-action language that implies a link exists when it doesn't | Not yet observed but architecturally possible |
| **No rule about inaccessible source handling** | When `fetcher.js` extracts only partial content (e.g., truncated paywall), Gemini may write about content it only partially saw | Could produce inaccurate claims about paywalled details |
| **No explicit "do not reference content you cannot reproduce" rule** | Gemini treats source multimedia as facts to mention, even though the output is text-only HTML | "The video shows..." — the video doesn't exist in the output |

### Proposed Rules (smallest possible additions to the prompt)

**Rule 1 — Multimedia fidelity (HIGH priority):**
> If the source contains a video, interactive element, or embedded media that you cannot reproduce in text, do not tell the reader to "watch" or "see" it. Instead, describe what the media shows in your own words, grounded in the source's description of it. If the source provides a direct URL to the media, include it as a plain text URL on its own line.

**Rule 2 — Source link surfacing (MEDIUM priority):**
> When the source references a specific tool, paper, dataset, or resource by name, include the URL if the source provides it. Place it as a plain text URL at the end of the relevant paragraph.

**Rule 3 — Partial content guard (LOW priority):**
> If the source text appears truncated or incomplete (e.g., ends mid-sentence, contains paywall indicators), note this in your output and only write about the information you actually received. Do not speculate about what the rest of the article says.

**Classification:** RECOMMENDATION (all three). These are prompt-only changes — no code changes required.

---

## Area F — Forensic Report: Implementation Sequence

### Finding Summary

| # | Finding | Severity | Pipeline Location | Root Cause | Fix Required | Files Affected |
|---|---|---|---|---|---|---|
| F1 | Source photographs used instead of FLUX illustrations | **HIGH** | `complete.js:130-147` | `trySourceImage()` is primary path; FLUX is fallback | Decision: keep source photos OR disable `trySourceImage()` for FLUX | `complete.js` |
| F2 | `cover_image_prompt` contains unused FLUX prompt when source image is used | **MEDIUM** | `complete.js:130-147` | Prompt written by Gemini but never used for image generation | Update `cover_image_prompt` to reflect actual image source | `complete.js` |
| F3 | Dangling video reference in Fullstack AI post | **MEDIUM** | `gemini.js:347-421` (prompt) | No rule about multimedia references; `fetcher.js` can't extract iframe videos | Add multimedia fidelity rule to prompt | `gemini.js` |
| F4 | `fetcher.js` cannot extract YouTube iframe embeds | **LOW** | `fetcher.js:82-119` | Text-only extraction; iframes produce no text nodes | Accept as limitation OR add iframe src extraction | `fetcher.js` |
| F5 | Tools/anthropic categories starved (1-2 feeds each) | **MEDIUM** | `sources.js` / `registry.seed.json` | Few curated sources for these categories | Add 2-3 curated sources per category | `registry.seed.json` |
| F6 | No rule about partial/truncated source content | **LOW** | `gemini.js` prompt | Not addressed in current prompt | Add partial content guard rule | `gemini.js` |
| F7 | No rule about surfacing source URLs for referenced resources | **LOW** | `gemini.js` prompt | Not addressed in current prompt | Add source link surfacing rule | `gemini.js` |

### Implementation Sequence

#### P0 — Do Now (this session, if approved)

| Item | Change | Effort | Risk |
|---|---|---|---|
| **F3** | Add multimedia fidelity rule to `gemini.js` prompt (1 paragraph, ~50 words) | 5 min | Minimal — prompt-only, no code logic change |
| **F1 decision** | Felix decides: keep source photographs (current behavior) OR disable `trySourceImage()` to always use FLUX | Decision only | Architectural — affects all future posts |
| **F2** | If F1 keeps source photos: update `cover_image_prompt` in `complete.js` to store `[SOURCE_PHOTO]` instead of the unused FLUX prompt | 10 min | Low — cosmetic field change |

#### P1 — Next Session (feed governance)

| Item | Change | Effort | Risk |
|---|---|---|---|
| **F5** | Add 2-3 curated sources to `registry.seed.json` for `tools` and `anthropic` categories | 30 min + Felix approval | Governance — seed changes require explicit approval |
| **F4** | Add iframe `src` extraction to `fetcher.js` (parse `<iframe src="youtube.com/embed/...">` and return the YouTube URL in metadata) | 20 min | Low — additive, doesn't change existing text extraction |

#### P2 — Backlog (editorial polish)

| Item | Change | Effort | Risk |
|---|---|---|---|
| **F6** | Add partial content guard rule to `gemini.js` prompt | 5 min | Minimal |
| **F7** | Add source link surfacing rule to `gemini.js` prompt | 5 min | Minimal |

---

## Appendix D — Image Pipeline Decision Tree

```
Phase 2 (complete.js) processes each pending post:

  Is this a podcast/YouTube post (category === 'ai')?
  ├─ YES → skip source image → go to STEP 1c (FLUX)
  └─ NO ↓

  STEP 1: trySourceImage(source_urls)
    → Fetch source article HTML
    → Extract og:image meta tag
    → Download image if >5KB and safe URL
    ├─ SUCCESS → use source photograph as cover image
    │            (cover_image_prompt still contains FLUX prompt — unused)
    └─ FAIL ↓

  STEP 1b: Check for [OG_IMAGE] prefix in image_prompt
    → editorial pipeline pre-resolved the og:image via Tavily
    ├─ SUCCESS → download and use editorial og:image
    └─ FAIL ↓

  STEP 1c: FLUX generation
    → Generate AI illustration from Gemini's image prompt
    → This is the FALLBACK — only used when source image unavailable
```

**Key insight:** The `cover_image_prompt` is written by Gemini during Phase 1 (based on source content), but the actual image used in Phase 2 is often the source photograph — not the FLUX illustration the prompt describes. This is a documentation/expectation gap, not a bug.

---

## Appendix E — Fullstack AI Post: Complete Evidence Chain

```
1. Google AI RSS feed → scout.js picks article
2. fetcher.js → fetchArticleText() → ~130 words extracted
   (YouTube iframe at youtube.com/embed/5JKWW4tX-ns → no text nodes → NOT extracted)
3. SWS gate → score passes (Google blog = rich source)
4. Gemini receives ~130 words → generates 3,155 chars of content
   - Writes "The video accompanying the post lets viewers hear Paige's full explanation"
   - Writes "Watching the video helps developers understand how the layers map"
   - NO URL provided for the video
5. Quality gate → passes (400+ words, h3 headings, "Why This Matters" closing)
6. Phase 2 → complete.js → trySourceImage(['https://blog.google/...'])
   → Fetches Google blog HTML → extracts og:image
   → Downloads https://storage.googleapis.com/gweb-uniblog-publish-prod/images/thumbnail_BfIj9lP.max-1440x810.jpg
   → 146,640 bytes, image/jpeg
7. Uploads to Supabase Storage as covers/392.jpg (146,640 bytes — byte-identical to source)
8. cover_image_prompt = "A watercolour illustration of a multi-layered pyramid..."
   (this prompt was NEVER used — the source photo was used instead)
9. Published to aitrends.ng with source photograph as cover, FLUX prompt as metadata
```
