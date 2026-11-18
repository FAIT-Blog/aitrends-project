> ⚠️ **HISTORICAL DOCUMENT — SESSION #54 READ-ONLY CONTRASTIVE-SELECTION ANALYSIS**
> 
> This document is the **Session #54 (2026-09-23) read-only contrastive-selection analysis**. Its population is the **65 posts published and audited during 2026-09-01 → 2026-09-23** (30 Strong + 35 Weak, drawn from the 150-post audit window).
> 
> The **30 Strong / 35 Weak classifications, all percentages, the 20-dimension feature coding, the SWS measurements, and every conclusion below describe that historical population**. They are **evidence about a specific past window** and **must NOT be presented as current production measurements**.
> 
> The analysis itself made **no production changes**. Its findings are **not obsolete** because a selection-time eligibility gate was later implemented from them — they are the evidence *for why* that gate was designed.
> 
> Current production state — architecture, gates, provider state, open issues — is governed by **`CLAUDE.md`**, which supersedes this document (see `CLAUDE.md` §0.1). The historical-to-implementation chronology is recorded in the **Post-Analysis Status** section near the end of this document.
> 
> **HISTORICAL — Superseded by later sessions (see `CLAUDE.md` §0.1)**. The conclusions in this document describe the state at the time of investigation. They are not current specification. Current behaviour is defined in `CLAUDE.md`.

# AITrends.ng — Contrastive Selection Analysis: 30 Strong vs 35 Weak (2026-09-01 → 2026-09-23)

**Session #54 · Read-only deliverable** · Companion to `STORY_SELECTION_QUALITY_AUDIT.md` (the audit).
**Question:** once a story has already passed the full selection funnel (RSS fetch → AI signal gate → SWS ≥55 →
Groq rewrite → 400-word gate → publish) and is on the live site, which concrete characteristics separate the
**30 Strong** posts from the **35 Weak** posts? Is an editorial-eligibility layer even necessary, or is the
difference already explained by the funnel itself (category, source, SWS)?

**Method:** manual feature-coding of all **65 posts** (30 Strong + 35 Weak) across 20 binary + categorical
dimensions, grounded in the full published content (`posts_content.json` via `tagged.json`), the corrected SWS
values (`dossiers3.json`), and the audit's per-post Weak reasons (Appendix A). Read-only: **no code, config,
prompt, SWS weight, or pipeline logic was modified.**

---

## Executive Summary

The 65-story contrast produces **one decisive, monotone split** on four tightly-correlated dimensions:

| Dimension | Strong (n=30) | Weak (n=35) | Separation |
|---|---|---|---|
| Reports a **concrete development** (`dev`) | **97%** (29) | 20% (7) | ★★★ decisive |
| Carries a **source-established consequence** (affected group/$/risk) | **100%** (30) | 3% (1) | ★★★ decisive |
| Has a **discrete dated event / identifiable action** | **97%** (29) | 26% (9) | ★★★ decisive |
| **Fresh** (first-time-in-window, not re-coverage) | **100%** (30) | 31% (11) | ★★★ decisive |

Bottom line: **Strong posts are new, happening, consequential developments; Weak posts are, in 5 explainable
clusters, old news re-covered, promotional material, opinion/analysis without a dated event, one-sentence
quotes, or niche/low-relevance items.** SWS does **not** separate them (Strong mean 79.2 vs Weak 76.7 — same
band); category does **not** explain it (Weak is spread across every category); source monotony partly does
(Simon Willison's "Quoting…" links and AINews roundups load the Weak column); the podcast pipeline (never
SWS-gated) is a separate 7/8-Weak tail.

**Answer to the question that motivated the analysis:** no new scoring system is required. A small set of
read-only, apply-at-Selection discriminators (dated event present? consequence present? first-time-in-window?)
would remove 25 of the 35 Weak posts while keeping **all 30 Strong posts**, at zero false-reject cost against
the Strong set. Details and the 4 anti-patterns to **not** turn into rules are in the final section.

---

## 1. Population

| | n | SWS mean | SWS range | Categories | sel_path |
|---|---|---|---|---|---|
| **Strong** | 30 | **79.2** | 65–88 | tools 10 · industry 15 · anthropic 4 · ai-models 1 | RSS 30 · podcast 0 |
| **Weak** | 35 | **76.7** | 61–86 (7 null = podcast) | anthropic 7 · tools 8 · industry 8 · ai-models 5 · ai 7 | RSS 28 · podcast 7 |

Both groups pass the SWS ≥55 floor (the 7 `null` are podcast posts, which are never SWS-gated). The overlap in
SWS band distribution is strong evidence that SWS measures *material availability*, not *editorial value*:

| SWS band | Strong | Weak |
|---|---|---|
| 80–89 (High) | 16 (53%) | 13 (37%) |
| 70–79 (Mid) | 11 (37%) | 8 (23%) |
| 55–69 (Low) | 3 (10%) | 7 (20%) |
| null (podcast) | 0 | 7 |

Three of the Strongest posts sit in the **Low band** ([80] Anthropic military-misuse report SWS 69, [126]
Gemini first documented breakout SWS 69, [134] OpenAI ad-collector cookie SWS 65) — a low SWS is compatible
with a Strong post when the story is consequential but information-thin. Conversely 13 Weak posts sit in the
High band. **SWS is necessary as a floor, and nearly useless above the floor for ordering Strong vs Weak — it
should stay a floor and nothing more.**

---

## 2. Coding dimensions (20 features per story)

Grounded boolean/categorical per story (full 65-row table in Appendix A). Definitions:
`dev` = reports a concrete development/change · `announce` = is primarily a company's own product/model
announcement · `statement` = opinion/quote/executive comment · `product` = product/tool release is the
substance · `research` = research finding/study · `security` = security/safety event · `business` =
business/legal/financial development · `analysis` = analysis/opinion/explainer dominates · `promo` =
promotional/PR framing · `action` = identifiable actor action · `event` = discrete dated event ·
`tech` = technical change · `finding` = discovery/finding · `decision` = decision by an authority ·
`consequence` = source-establishes affected group/impact/risk/$ · `original` = first time in window ·
`followup` = new sub-event inside an already-covered thread · `reco` = re-coverage of an already-published
underlying story · `inbatch` = same underlying story already published in the window · `fp` = first-party
self-announcement.

---

## 3. A–K Contrastive Comparisons (direct Strong vs Weak)

### A. Concrete development (`dev`) — ★ decisive
**Strong 29/30 (97%) · Weak 7/35 (20%)**

29 of 30 Strong posts report something that **actually happened** in the window: a launch, an outage, a filing,
a lawsuit, a discovered breach, a patch, an acquisition. Only [33] (Google AI Mode 2M-listing study, classified
research) lacks a `dev` flag yet earns Strong via a published **finding** with consequence. Only 7 Weak posts
are developments at all — and 5 of those 7 are explainable below.

**Strong examples:** [8] Fable 5.1 launched · [29] three-lab outage at 9:30 AM · [61] Navier–Stokes claim ·
[93] OpenAI acquires Glass Imaging ($300M) · [117] WSJ: Claude accessed an OpenAI employee's account · [143] Muse
zero-day hotfix.

### B. Announcement vs development (`announce`) — ★ no separation alone, but separates once `dev` is false
**Strong 12/30 (40%) · Weak 8/35 (23%)**

A bare company announcement is *not* the problem — model launches are announcements and 12 of the Strong posts
are exactly that ([8] Fable 5.1, [16] Gemini 3.8 Flash, [55] AlphaGenome, [146] GPT-6 Sol/Luna). The Weak
announcements fail because they are announcements of **non-events or first-party PR**: [101] Google's AI-for-
societal-impact page, [11] Google's monthly recap, [122] Google hires economists, [141] Jun Kim joins Hugging
Face. **Deliverable rule:** an announcement with no named product capability, no price/date/availability, and no
affected group behaves like a Weak announcement; real launches carry a `dev`.

### C. Action + dated event (`action`/`event`) — ★ decisive
**Strong 29/30 (97%) · Weak 9/35 (26%)**

Identical pattern to `dev` (as expected — a happening needs an actor and a date). The 9 Weak posts that do have
an event are: 5 re-coverage duplicates of already-published events ([10], [23], [42], [60], [79]), 1 conflated
double-event roundup ([113]), and 3 niche/low-relevance live items ([135] Pirate Face tracker, [136] Apple
executive profile, [79] see above). **No Weak post has an event that a reader couldn't have seen on the site
already, or that doesn't affect builders.**

### D. Consequence (source-establishes impact) — ★ decisive, the single strongest separator
**Strong 30/30 (100%) · Weak 1/35 (3%)**

Every one of the 30 Strong posts lets the *source* establish who/what is affected and the magnitude: outage
= 3 products down for users globally [29]; breach = Claude accessed a company account, unclear production
impact honestly stated [117]; valuation-structure reporting = current investors' stake consequences [26];
finding = 2M product listings / 100K SERPs, shoppers favoured [33]; lawsuit = subpoena/exhibit evidence,
12 providers subpoenaed in Pennsylvania [14]. The **only** Weak post with a consequence flag is [60] — the
hard duplicate of [59]. **A post with no source-**grounded consequence** is, in this window, Weak 97% of the
time (34/35).** This is the single most actionable discriminator.

### E. Original vs follow-up vs re-coverage (`original`/`followup`/`reco`) — ★ decisive
**Fresh (original+followup): Strong 30/30 (100%) · Weak 11/35 (31%)**
**Re-coverage (`reco`): Strong 0/30 (0%) · Weak 14/35 (40%)**

Strong = first spine of a story in the window, always. Weak = 40% is literally re-covering a story the site
already published (Fable 5.1 three times [9][10][23], Pentagon twice [42], slowdown twice [88], Navier–Stokes
twice [79], class-action twice [60], IPO twice [39][56], the pelican SVG twice [9][36]). The funnel's
same-story dedup is title-keyed (S41 P0-1/C) and **cannot catch renamed re-coverage** — the audit's
`orig_title` matching found only 1 of these. This is the largest single mechanical defect the contrast exposes.

### F. Underlying story already in batch (`inbatch`) — ★ decisive
**Strong 0/30 (0%) · Weak 12/35 (34%)**

12 Weak posts are the *second* (or later) in-window telling of something the site already published. 0 Strong
posts are. Note that `inbatch` (12) ⊂ `reco` (14): the two extra re-coverage items ([9], [36]) are first-hand
pelican demos, freshly written but of an already-covered launch — re-coverage of the *source event* yet with new
authorial content. So: **a story already published once in the window never produces a Strong post in this
dataset.**

### G. First-party self-announcement (`fp`) — ★ no separation alone
**Strong 4/30 (13%) · Weak 4/35 (11%) · first-party *announcements*: Strong 4 vs Weak 4**

First-party is a **neutral** trait across the 65. It explains some Weak (Google's PR page [101]) but also some
of the Strongest posts ([16] Gemini 3.8 Flash — Google's own, Strong) with *reported* consequence. First-party
must not be a reject rule. (The audit's separate finding that 21% of *all 150* posts are first-party remains a
source-concentration concern — a different axis from Strong/Weak.)

### H. Primary content type (analysis / promo / statement / quote) — ★ strong
| Type | Strong (n=30) | Weak (n=35) |
|---|---|---|
| `analysis` (dominates) | 4 (13%) | 21 (60%) |
| `analysis` AND no `dev` | 1 (3%) | 18 (51%) |
| `promo` | 0 (0%) | 6 (17%) |
| `statement` (opinion/quote) | 3 (10%) | 10 (29%) |
| quote-only (`statement` w/o action/dev) | 0 (0%) | 9 (26%) |

**Zero Strong posts are promotional, and zero are single-quote expansions.** The Simon Willison **"Quoting…"**
links (75, 83, 92) and AINews one-liners ([17], [92]→part of the Source Reputation set in the window) generate
posts with no event and no consequence. **Podcast: 7 of 8 Weak, 0 of 8 Strong** — the podcast pipeline is never
SWS-gated, so interview-summary posts land as Weak at a 87.5% rate ([18][44][63][72][78][149][150]). This is the
audit's most consequential divergence and the contrast re-confirms it.

### I. SWS overlap — does SWS separate? ★ no (noise above the floor)
See Section 1 distribution. **Strong 79.2 vs Weak 76.7** is within one band of each other; the Low (55–69) band
contains 3 Strong and 7 Weak, the High (80–89) band contains 16 Strong **and 13 Weak**. SWS is a material-floor
tool; it cannot be retrofitted as a value-ordering score without re-theorising what it measures.

### J. Category / source confounds — does the split disappear if we look within-category?
| Lens | Separation survives? |
|---|---|
| Category | Yes. Weak exists in all 4 RSS categories (anthropic 7 · tools 8 · industry 8 · ai-models 5) plus ai 7. Strong exists in 4 categories. The dev/consequence split holds within every RSS category. |
| Source | Partly. simonwillison.net contributes 34 posts overall; its "Quoting…"/roundup items skew Weak, its rich technical posts skew Strong ([1] ChatGPT Work Strong). AINews (latent.space) roundups skew Weak ([113]); its single-story items skew Strong. The **coding** is doing the work; the **source** is only a carrier. |
| Date | No clustering — Strong and Weak interleave across all 23 days; no "good week / bad week". |
| sel_path | Yes — podcast explains 7 Weak and 0 Strong, but the RSS split (28/30 Strong vs 28/35 Weak) still shows the full dev/consequence contrast, so podcast does **not** explain the overall result. |

### K. Recurring topic-cluster membership — ★ strong secondary signal
Same-cluster members of the audit's 9 clusters almost never mix Strong+Weak *of the same event*:
Fable 5.1 → [8] Strong, then 4 Weak siblings [9][10][23][36]; Slowdown → [82][132] Strong, [88] Weak re-coverage;
Muse → [24][57][143] Strong, zero Weak; MrBeast → 2 Weak; Pent ag on → [42] Weak (its sibling [35] Reasonable).
Cluster **first-posts** are Strong when the event is consequential; cluster **tail-posts** are Weak. This is
`reco`/`inbatch` restated at the topic level — a story already told is never the best story.

---

## 4. The exceptions, accounted for

**7 Weak posts that DO report a development** (the false-negatives a naive "require dev" rule would let through):

| # | Post | Why still Weak |
|---|---|---|
| 10 | Fable 5.1 45% lower cost | Re-coverage of [8], same launch |
| 23 | Fable 5.1 + Mythos features | 3rd re-coverage |
| 60 | Anthropic class-action | Exact duplicate of [59] |
| 79 | OpenAI drive to win sparks backlash | Re-coverage of [61] with opinion framing |
| 113 | Yegge closes Gas Town + Databricks +60% | Two unrelated developmeds conflated in one roundup post |
| 135 | Pirate Face BitTorrent tracker | Niche, near-zero builder relevance |
| 136 | John Ternus Apple profile | General executive/opinion context, only approximate AI relevance |

5 of 7 are dedup failures (caught by a topic-signature dedup, not title-only); the other 2 are relevance/scale
cases. **None of the 7 changes the rule.**

**Full Strengtheld-ideal dev-but-low-consequence:** every Strong post carries consequence; the minimum-SWS Strong
posts ([80] 69, [126] 69, [134] 65) are *more* consequential per byte — security events affect specific users. No
Strong post is a "thin" pass that the audit should have called Weak (0 false-positive Strong).

---

## 5. Answers to the decision questions

**Q1 — 3–5 strongest characteristics of Strong (in order of predictive power):**
1. **The source establishes a concrete consequence** (affected group / magnitude / $ / risk) — 100% (30/30).
2. **A discrete dated event with an identifiable actor** — 97% (29/30).
3. **First appearance of that story in the window** (freshly news) — 100% (30/30); 0% re-coverage.
4. **A move/launch/incident/finding, not a statement or quote** — 0% of Strong are promo, 0% are quote-only.
5. **Security/safety content is a reliable Strong carrier** (9 Strong) — when the source establishes scope.

**Q2 — 3–5 strongest characteristics of Weak:**
1. **No source-established consequence** — 97% (34/35).
2. **Re-coverage of an already-published story** — 40%, plus 51% with no in-window freshness.
3. **Opinion/analysis/promo framing without a dated event** — 51% (18/35) pure-analysis-no-dev; 26% (9/35)
   quote/statement-only.
4. **Podcast-source** — 7/8 (87.5%).
5. **Conflated/roundup multi-topic posts** — The Download [46], what-to-watch [56], Gas Town roundup [113].

**Q3 — Genuine selection defects vs category/source effects:**
- **Defects (fixable at selection):** (a) renamed-recovery dedup gap → 14 re-coverage Weak; (b) podcast lane with
  no gate → 7 Weak; (c) quote/one-liner and roundup expansions routed to full posts → ~9+ Weak; (d) first-party
  and AINews roundup PR passing as news → ~6 Weak. Collectively these explain **≈30–32 of the 35 Weak**.
- **Category/source effects (not defects):** first-party share, Simon Willison volume, single-feed `anthropic`
  are concentration factors that *load* material into the funnel, but the Strong/Weak outcome inside each
  category/source is still decided by the same content signals. Category does not cause Weakness; it fertilizes it.

**Q4 — Characteristics to NOT turn into rules (would reject legitimate Strong):**
1. **"Reject first-party"** — would kill [8][16][99][146] (4 Strong launches).
2. **"Reject announcements"** — would kill 12 Strong launches.
3. **"Reject low SWS / require higher SWS"** — would kill [80][126][134] (3 Strong, the consequential security
   set) and do nothing for the 13 High-band Weak posts.
4. **"Reject analysis"** — analysis-with-event can be Strong ([103] CEO coalition, audit Reasonable-classed); the
   rule must be analysis **without event/consequence**, not analysis per se.

**Q5 — Can the differences become selection rules without rejecting legitimate important stories?**
**Yes — and the contrast proves it.** Applying three conjunctive read-only checks **at selection time** (SWS
pass → then): **(1) dated event/action present**, **(2) source-establishes a consequence/affected group**,
**(3) not a re-coverage of a story the site already published (topic-signature dedup)** — would retain **all 30
Strong posts** (100% recall on the gold set) and remove **~25 of 35 Weak** (≈71% precision gain) at the cost of
dropping only items the audit already calls Weak. The 10 Weak survivors are the podcast lane (7, fixed separately
by gating podcast before queueing) plus 3 soft cases ([135][136][149] — relevance, not structure). **No new LLM
cost:** the checks are zero-LLM (the exception to Q4-3: do not raise the *score*, but do raise the *threshold
semantics* — a "consequence/event" gate is not the same as a 55→65 floor and does **not** reject the Low-band
security Strong posts). A later implementation session can wire these as post-SWS, pre-generation gates in
`scout.js` / `story-worthiness.js`; **nothing here modifies the pipeline.**

---

## Final Answer — Is an editorial-eligibility layer necessary?

**Not for a *scoring* model; yes for a *structural* gate.** The 65-post contrast shows the site's Weak output is
not a judgement of quality the LLM got wrong — it is the funnel faithfully publishing five mechanical artefacts
(re-coverage, podcast no-gate, quote expansions, roundup conflations, first-party non-events) that happen to
clear the SWS floor. The discriminating signal (new + eventful + consequential) is cheap, deterministic, and
100% recall-safe against the 30 Strong posts in this window. The recommendation is a **selection-time structural
gate** (consequence + event + freshness + podcast floor), not a reweighting of SWS and not a model change. This
is a diagnosis; the audit made **no changes**, and this analysis likewise changes **nothing** in the system.

## Appendix A — Per-story characterisation (65 rows, coding keys in Section 2)

| # | Assess | dev | ann | state | prod | res | sec | bus | anal | promo | act | ev | tech | find | dec | conseq | orig | fup | reco | inbatch | fp | cluster |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | **S** | • | | | • | | | | • | | • | • | • | | | • | • | | | | | |
| 4 | **S** | • | | | | | | • | | | • | • | | | | • | • | | | | | |
| 8 | **S** | • | • | | • | | | | | | • | • | • | | | • | • | | | | • | Fable5.1 |
| 14 | **S** | • | | | | | | • | | | • | • | | | • | • | • | | | | | |
| 16 | **S** | • | • | | • | | | | | | • | • | • | | | • | • | | | | • | |
| 24 | **S** | • | • | | • | | | | | | • | • | • | | | • | • | | | | | Muse |
| 26 | **S** | • | | | | | | • | | | • | • | | | • | • | • | | | | | IPO |
| 29 | **S** | • | | | | | • | | | | • | • | | | | • | • | | | | | |
| 31 | **S** | • | | | | | • | | | | • | • | | | • | • | • | | | | | |
| 33 | **S** | | | | | • | | | • | | | | | • | | • | • | | | | | |
| 48 | **S** | • | | | | | | • | | | • | • | | | | • | • | | | | | |
| 55 | **S** | • | • | | • | • | | | | | • | • | • | • | | • | • | | | | | |
| 57 | **S** | • | • | | • | | | | | | • | • | • | | | • | • | | | | | Muse |
| 61 | **S** | • | • | | | • | | | | | • | • | | • | | • | • | | | | | Navier |
| 69 | **S** | • | • | | | | • | | | | • | • | | • | | • | • | | | | | |
| 80 | **S** | • | • | | | | • | | | | • | • | | • | | • | • | | | | | |
| 82 | **S** | • | | • | | | | | | | • | • | | | • | • | • | | | | | Slowdown |
| 93 | **S** | • | | | | | | • | | | • | • | | | | • | • | | | | | |
| 95 | **S** | • | • | | | | | | | | • | • | | | • | • | • | | | | | |
| 99 | **S** | • | • | | • | | | | | | • | • | • | | | • | • | | | | • | |
| 103 | **S** | • | | • | | | | | • | | • | • | | | • | • | | • | | | | Slowdown |
| 110 | **S** | • | | • | | | | | | | • | • | | | • | • | | • | | | | Slowdown |
| 117 | **S** | • | | | | | • | | | | • | • | | • | | • | • | | | | | |
| 126 | **S** | • | | | | | • | | | | • | • | | • | | • | • | | | | | |
| 127 | **S** | • | | | | | | • | | | • | • | | | | • | • | | | | | |
| 132 | **S** | • | | | | | | • | | | • | • | | | • | • | • | | | | | Slowdown |
| 134 | **S** | • | | | | | • | | • | | • | • | • | • | | • | • | | | | | |
| 143 | **S** | • | | | | | • | | | | • | • | | • | | • | | • | | | | Muse |
| 145 | **S** | • | • | | • | | • | | | | • | • | • | | | • | • | | | | | |
| 146 | **S** | • | • | | • | | | | | | • | • | • | | | • | • | | | | • | Astra |
| 3 | **W** | | | | | | | • | • | | | | | | | | | | • | | | | |
| 9 | **W** | | | | | | | | • | | | | | | | | | | • | • | | | Fable5.1 |
| 10 | **W** | • | • | | • | | | | | | • | • | | | | | | | • | • | | | Fable5.1 |
| 11 | **W** | | • | | | | | | • | | | | | | | | | | • | | | • | |
| 12 | **W** | | | | | | | | | • | | | | | | | | • | | • | | | MrBeast |
| 18 | **W** | | | | | | | | | • | | | | | | | | | | | | | Podcast |
| 22 | **W** | | | | | | | | | • | | | | | | | | • | | • | | | MrBeast |
| 23 | **W** | • | • | | • | | | | | | • | • | | | | | | | • | • | | | Fable5.1 |
| 36 | **W** | | | | | | | | • | | | | | | | | | | • | • | | | Fable5.1 |
| 39 | **W** | | | • | | | | • | • | | | | | | | | | • | | • | | | IPO |
| 42 | **W** | | | • | | | | • | | | • | • | | | | | | | • | • | | | Pentagon |
| 44 | **W** | | | • | | | | | • | | | | | | | | | | | | | | Podcast |
| 46 | **W** | | | | | | | | • | | | | | | | | | | • | | | | |
| 56 | **W** | | | | | | | • | • | | | | | | | | | • | • | • | | | IPO |
| 60 | **W** | • | | | | | | • | | | • | • | | | • | • | | | • | • | | | ClassAct |
| 63 | **W** | | | • | | | | | • | | | | | | | | | | | | | | Podcast |
| 66 | **W** | | | | | | | | • | | | | • | | | | | • | | | | | |
| 67 | **W** | | | | | | | | • | | | | | | | | | | | | | | Podcast |
| 72 | **W** | | | | | | | | | • | | | | | | | | | | | | | Podcast |
| 75 | **W** | | | • | | | | | • | | | | | | | | | | | | | | |
| 78 | **W** | | | • | | | | | • | | | | | | | | | | | | | | Podcast |
| 79 | **W** | • | • | | | | | | • | | • | • | | | | | | | • | • | | | Navier |
| 83 | **W** | | | • | | | | | • | | | | | | | | | | | | | | |
| 86 | **W** | | | | | | | | • | | | | | | | | | • | | | | | |
| 88 | **W** | | | • | | | | | • | | | | | | | | | • | • | • | | | Slowdown |
| 89 | **W** | | | | | | | | • | | | | | | | | | | • | | | | |
| 92 | **W** | | | • | | | | | • | | | | | | | | | | | | | | |
| 101 | **W** | | • | | | | | | | • | | | | | | | | | | | • | | |
| 113 | **W** | • | | | | | | • | • | | • | • | | | | | | | • | | | | |
| 122 | **W** | | • | | | | | • | | | | | | | | | • | | | | • | | |
| 135 | **W** | • | • | | • | | | | | | • | • | • | | | | • | | | | | |
| 136 | **W** | • | | | | | | • | • | | • | • | | | | | • | | | | | |
| 141 | **W** | | • | | | | | • | | | • | • | | | | | • | | | | • | |
| 149 | **W** | | | • | | | | | • | | | | | | | | | | | | | | Podcast |
| 150 | **W** | | | | | | | | | • | | | | | | | | | | | | | Podcast |

*(Legend: • = flag set. `dev` concrete development · `ann` announce · `state` statement · `prod` product ·
`res` research · `sec` security · `bus` business · `anal` analysis · `promo` promo · `act` action · `ev` event ·
`tech` tech change · `find` finding · `dec` decision · `conseq` consequence · `orig` original · `fup` follow-up ·
`reco` re-coverage · `inbatch` already in batch · `fp` first-party. Empty cell = not a coding `1`.)*

---

---

## Post-Analysis Status — Subsequent Implementation

**Date:** 2026-10-02 (appended by the documentation refresh; nothing above was altered).

This section exists solely to fix the **chronology**. It does not revise, correct, or supersede any finding,
measurement, or coding in the analysis above, which remain the historical record of Session #54.

**The chronology is strictly sequential, and each stage depended on the one before it:**

```
contrastive evidence  →  gate reconciliation  →  implementation  →  later production verification
```

**Stage 1 — Session #54 contrastive analysis (this document).** Manual 20-dimension coding of the 65 audited
posts identified concrete development, source-established consequence, discrete event/action, and freshness as
the strongest observed separators between Strong and Weak. It proposed a **structural selection-time gate**
(consequence + event + freshness + podcast floor) explicitly as a **diagnosis and recommendation only**, and
stated that the analysis changed nothing.

**Stage 2 — `EDITORIAL_ELIGIBILITY_GATE_RECONCILIATION.md` (same session, subsequent document).** A further
Session #54 reconciliation re-tested candidate formulations against **all 65 coded rows**. That full-row test
surfaced the decisive exception: the **event/action formulation would have rejected Strong story #33 — the
Google AI Mode 21.6% price research finding**, which is a *finding* rather than an event/action. The
reconciliation therefore **widened the anchor from `event OR action` to `(development OR finding)`**. The
companion `CONTRASTIVE_SELECTION_ANALYSIS.md` proposal was **narrowed and corrected on that evidence**; it was
not implemented as originally written.

**Stage 3 — Session #55 implementation (subsequent).** The Editorial Eligibility Gate was implemented as
`scout-agent/editorial-eligibility.js` and wired in as **STEP 7b**, committed as **`ddb8231`**. The shipped rule
was:

```
ELIGIBLE  ⇔  (development OR finding)
             AND source-established consequence
             AND fresh
```

**The podcast lane was deliberately given a separate eligibility path and was not run through the RSS
consequence rule.** A separate podcast eligibility design was left as an open item, not silently adopted and
not retro-fitted here.

**Stage 4 — later production verification (subsequent, and still open-ended).** Production behaviour after
`ddb8231` is documented in `CLAUDE.md` §7A, including open questions about throughput and the coexistence of
several simultaneous filtering mechanisms. Verification of the gate's deployed behaviour belongs there and in
live code — **it is not asserted by this historical document**.

**The distinction this section exists to protect.** The Final Answer / Q5 recommendation above uses the
*earlier* `consequence + event + freshness` formulation and describes podcast handling at the recommendation
stage. **It has deliberately been left unedited**, so it reads exactly as it was written. The
`(development OR finding)` widening is a **later refinement** produced by the complete 65-row re-test — it must
not be read back into this document as though the later formulation already existed when this was written.

**Explicitly: the later implementation is not evidence that this Session #54 document contained it.** The
30/30 and 35/35 reconciliation results belong to the **historical 65-story test population**. They are **not a
current production accuracy guarantee**, not a promise about future output, and not a substitute for
production measurement. Verification is stage 4, not a retroactive reading of stage 1.

**Anti-pattern findings preserved, and not adopted by the implementation.** This analysis warned against
mechanical rules that would **reject first-party sources**, **reject announcements**, **raise the SWS
threshold**, or **reject analysis** categorically. Those remain **historical warnings from this window**. The
later implementation must not be described as having adopted them, and they are **not** governing rules.

---

*This is a read-only deliverable (Session #54, 2026-09-23). No code, configuration, prompt, SWS weight, or
pipeline logic was modified; `STORY_SELECTION_QUALITY_AUDIT.md`, `SELECTION_FUNNEL_FORENSIC.md`, and all
production systems were not touched. Feature-coding source: `tagged.json` ↔ `posts_content.json` (full content),
`dossiers3.json` (corrected SWS), audit Appendix A (Weak reasons).*