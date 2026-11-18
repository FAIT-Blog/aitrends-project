> ⚠️ **HISTORICAL DOCUMENT — SESSION #54 READ-ONLY GATE RECONCILIATION**
> 
> This document is the **Session #54 (2026-09-23) read-only editorial-eligibility gate reconciliation**. It was performed **before implementation**.
> 
> Its evidence window (2026-09-01 → 2026-09-23) and its **65-story coded population** (30 Strong + 35 Weak) are **historical**. The classifications, percentages, formulation results, story numbers, tables, and conclusions below are **historical evidence — not current production measurements**.
> 
> **The original reconciliation made no production changes.** Its "READY FOR IMPLEMENTATION" decision meant the evidence supported proceeding; it did **not** mean the gate already existed.
> 
> Current production truth is governed by **`CLAUDE.md`** and live code (see `CLAUDE.md` §0.1). The later implementation (`ddb8231`) and subsequent production verification are documented in the **Post-Reconciliation Status** section at the end of this document.
> 
> **HISTORICAL — Superseded by later sessions (see `CLAUDE.md` §0.1)**. The conclusions in this document describe the state at the time of investigation. They are not current specification. Current behaviour is defined in `CLAUDE.md`.

# Editorial Eligibility Gate — Reconciliation (65-Story Read-Only Re-Test)

**Session #54 · Read-only deliverable** · Companion to `STORY_SELECTION_QUALITY_AUDIT.md` and
`CONTRASTIVE_SELECTION_ANALYSIS.md`. Re-tests the proposed editorial-eligibility gate before any production
change, reconciles the reported Strong exception (story #33), measures three candidate formulations against
**all 65 coded rows** (30 Strong + 35 Weak), finds the smallest rule the evidence supports, examines the Weak
survivors, and confirms podcast handling.

**Changes to code / configuration / prompts / SWS / pipeline / deployment / commits: NONE.**

---

## Decision

# ✅ READY FOR IMPLEMENTATION

**The proposed gate survives the 65-story re-test in a *narrowed and simplified* form.** The evidence supports
the following structural rule:

> **A post is eligible if ALL three hold:**
> 1. **Consequence** — the *source* establishes a real affected group, magnitude, risk, or dollar/percentage
>    figure (not a generic "this matters").
> 2. **Freshness** — the underlying story has **not already been published** in the production window
>    (topic-signature dedup, not title-token dedup).
> 3. **Development/Finding** — the story reports a concrete development (launch, incident, deal, breach, patch,
>    outage, release) **or** a research finding/measurement (not merely an announcement-of-intent, a statement,
>    commentary, roundup, promo, or one-sentence quote).

**Exact measured result against the 65 coded rows:**

| Formulation | Strong kept | Strong rejected | Weak kept | Weak rejected |
|---|---|---|---|---|
| A — `(event OR action) AND consequence AND fresh` | 29/30 | **33** | 0/35 | 35/35 |
| B — `(dev OR finding OR decision OR event OR action) AND consequence AND fresh` | **30/30** | 0 | 0/35 | 35/35 |
| C — `(dev OR finding OR decision OR event) AND consequence AND fresh` | **30/30** | 0 | 0/35 | 35/35 |
| **C-min — `(dev OR finding) AND consequence AND fresh` (smallest anchor)** | **30/30** | 0 | 0/35 | 35/35 |
| **MIN — `consequence AND fresh` (no anchor)** | **30/30** | 0 | 0/35 | 35/35 |
| `consequence` alone (newtonian probe) | 30/30 | 0 | **1/35 (60 leaks)** | 34/35 |

**Interpretation:** Formulation **A would lose story #33** (the anomaly — see Section 1). Formulations **B, C,
C-min, and the anchor-less MIN all achieve 30/30 Strong retained and 0/35 Weak retained.** Because Weak story
[60] is the only Weak row with a consequence flag (and it is a hard duplicate, so freshness catches it), the
`consequence AND fresh` pair alone already separates the sample perfectly — **the anchor is redundant insurance on
this dataset, not a required separator.** The smallest irrefutable rule supported by the evidence is
`consequence AND fresh`; adding `(dev OR finding)` as the anchor guards a hypothetical future edge (a
statement that asserts a consequence without any development) at zero measured cost.

---

## 1. The exact Strong exception (story #33) — reconciled

| Field | Value |
|---|---|
| Published title | Google's AI Mode lists identical products at an average 21.6% higher price than standard search |
| Original title | Google AI Mode shows same products 21.6% more expensive than traditional search |
| Category / feed | `tools` / Hacker News |
| Source | `productrise.app` (independent SEO-tool analytics site) |
| SWS | 71 (High band) · sub-scores rich 100 · facts 100 · div 40 · pot 33 |
| Classification | **Research finding** — Strong |
| Date | 2026-09-04 · sel_path RSS single-source · gate_retries 0 |

**What it actually reports:** an independent, dated (Aug 9–31, 2026) empirical measurement of Google AI Mode vs
traditional search: 2M+ product listings, 100K+ SERPs, matched-pair price comparison → same product averages
21.6% higher in AI Mode (49% higher across all listings), a 1.28% overlap of visible listings, price
discrepancy in 38.1% of matches (68.4% of those cheaper in standard search), different sellers in 49.6% of
pairings. **The source establishes a concrete finding with real magnitude and an affected group** (shoppers pay
more; retailers are invisible to AI-Mode buyers; merchants may shift bidding/feed strategy).

**Was it correctly classified Strong? Yes.** It is an independently produced, concrete, quantified
finding-with-consequence — exactly the "news, trends" mission value. It is not a statement, not a promo, not
re-coverage, not thin.

**Does the source establish a concrete finding / discovery / measurement? Yes — this is its entire substance.**
Six concrete measured figures, a 23-day study window, and a named affected group. `consequence = 1`, `fresh = 1`,
`finding = 1`.

**Does it have an identifiable event/action under a broader interpretation?** Under the narrow coding used in the
contrastive pass, `event`/`action` were defined as a discrete dated happening with an identifiable actor
(outage at 9:30 AM, a filing, a launch). A research study is a *publication event* — it was released on a date
with a named producer and a bounded window — so a *broader* interpretation (research-publication-as-event) would
indeed admit #33 as an event. The narrow `event` coding therefore **falsely zeroed** a feature that a
publication-release reading would set to 1.

**Was the coding definition too narrow?** In one specific respect, yes — for exactly one story. The `event`
definition was calibrated on incidents and launches and did not name "a research study / report publication" as
an event class. That is precisely why formulation A (event-based only) fails #33 while **B / C / C-min /
MIN do not**: the `finding` feature (set to 1 for #33) restores it. The reconciliation is complete — **no
Strong story is lost once the anchor includes finding (or is omitted in favour of consequence+fresh)**.

---

## 2. Gate re-test — all 65 rows, all candidate formulations

Method: re-computed from the coded 65-row matrix (`/tmp/funnel/code_kw.js`) joined to `tagged.json`. Two
"fresh" variants were tested (f1 = `original OR followup`; f2 = `NOT reco AND NOT inbatch`). They produced
**identical results** on every formulation — reported once for clarity (f2). Full truth table per Weak row
reproduced in Section 4.

| Formulation | Definition | Strong /30 | Weak /35 | Verdict |
|---|---|---|---|---|
| **A** | `event OR action` ∧ consequence ∧ fresh | 29 ✓ 1 ✗ (#33) | 0 ✓ 35 ✗ | ❌ insufficient — loses #33 |
| **B** | `dev OR finding OR decision OR event OR action` ∧ consequence ∧ fresh | **30 ✓** | **0 ✓** | ✅ |
| **C** | `dev OR finding OR decision OR event` ∧ consequence ∧ fresh | **30 ✓** | **0 ✓** | ✅ |
| **C-min** | `dev OR finding` ∧ consequence ∧ fresh | **30 ✓** | **0 ✓** | ✅ |
| **MIN** | consequence ∧ fresh (no anchor) | **30 ✓** | **0 ✓** | ✅ |
| Newton probe | consequence alone | 30 ✓ | 1 ✓ (#60) 34 ✗ | ❌ consequence-only leaks the duplicate |

**Redundancy proofs (why B collapses to C-min):**
- `action` is redundant with `event` — **0 of 65 rows** have `action=1` and `event=0`.
- `decision` is redundant too — all 8 Strong `decision` posts ([14][26][31][82][95][103][110][132]) also have
  `dev=1` **and** `event=1`; removing `decision` from the anchor changes nothing (30/30·0/35 confirmed).
- `event` adds no Strong coverage beyond `dev` either — every Strong post except #33 has `dev=1`, and #33 has
  `finding=1`. So the **smallest anchor that still retains all 30 Strong is `dev OR finding`** (C-min).
- Even C-min is unnecessary for sample separation: **MIN (`consequence ∧ fresh`) alone achieves the same
  complete separation.** Anchor is optional defensive depth; recommend keeping `(dev OR finding)` because it
  encodes the task's stated distinction ("substantive new development/finding … with consequence" vs
  "statement/commentary/roundup/promo/re-coverage without a new consequential development") with zero measured
  cost.

**Category-independence:** retention of all 30 Strong holds category by category (tools 10/10 · industry
15/15 · anthropic 4/4 · ai-models 1/1) — the rule does not depend on category.
**First-party tolerance:** all 4 first-party Strong ([8][16][99][146]) retained — the rule never rejects
first-party *per se*.
**Announcement tolerance:** all 12 Strong announcement posts ([8][16][24][55][57][61][69][80][95][99][145][146])
retained — announcements pass when they are real launched products with consequence.
**No SWS increase:** the 3 lowest-SWS Strong ([80]=69, [126]=69, [134]=65) all carry consequence + freshness +
an event and are retained — no threshold raise is involved.
**No LLM requirement:** all three conditions are deterministic if consequence-detection is expressed as
source-grounded signals (presence of affected-group entities, percentage/currency figures, scope/scale terms);
freshness via topic-signature dedup; anchor via existing event/finding heuristics. Calibration is an
implementation-session concern, not an evidence concern (the rule itself is proven on the gold set).

---

## 3. The smallest general rule supported by the 65-story evidence

> **ELIGIBLE ⟺ (source-establishes consequence) AND (fresh) [AND (development/finding) — optional anchor].**

- Retains **all 30 Strong** (100% recall on the gold set).
- Removes **all 35 Weak** (100% precision on the gold set).
- Category-independent; first-party- and announcement-tolerant; no SWS raise; deterministically implementable.
- The **anchor `(dev OR finding)` is defensible but not load-bearing** on this sample — its only job is to
  formalise the "substantive development" guard for unseen edge cases. If forced to choose the *absolute
  smallest* rule, the evidence unambiguously supports `consequence AND fresh`.

---

## 4. Remaining Weak survivors under the best candidate rule

**Under the best rule (any of B, C, C-min, MIN): 0 of 35 Weak survive.** There are no survivors to list — the
only Weak row carrying `consequence=1` is [60], and it fails the freshness gate (hard duplicate of [59]).

The *useful* residual-risk analysis is therefore the **intended-failure triples** — the Weak that would pass if
any one component of the rule were defective at implementation time (listed from the per-row failing-predicate
table):

| # | Published title (abridged) | Fails | Classification of the residual risk |
|---|---|---|---|
| 60 | Anthropic class-action from power users | fresh only | **Duplicate of [59] (title-keyed dedup hole).** If freshness/dedup is implemented as current title-token matching only, [60] leaks. Topic-signature dedup is the fix (same development, same day, cross-category). |
| 10 | Fable 5.1 claiming 45% lower cost | consequence + fresh | Re-coverage; source-established consequence absent. |
| 23 | Fable 5.1 + Mythos features | consequence + fresh | Re-coverage (3rd); same launch. |
| 42 | Pentagon maintains supply-chain designation | consequence + fresh | Re-coverage of [35]. |
| 79 | OpenAI's Drive to Win Sparks Math Backlash | consequence + fresh | Re-coverage of [61] + opinion framing. |
| 113 | Yegge closes Gas Town; Databricks +60% | consequence + fresh | Roundup conflation (2 unrelated developments); no single consequence. |
| 135 | Pirate Face BitTorrent tracker | **consequence only** | **Niche/relevance** — real dev, fresh, but no affected group; the near-miss the consequence check exists to catch. |
| 136 | Will John Ternus steer Apple…? | **consequence only** | **Relevance/source-quality** — general Apple exec commentary, approximate AI relevance; near-miss caught only by consequence. |
| 141 | Jun Kim joins Hugging Face | anchor + consequence (C-min) | **Routine people-move** (event under anchor C but no dev/finding); SWS 82 High-band — the hardest case, killed only by consequence + minimal anchor. |
| 18,44,63,67,72,78,83,92,101,122,149,150 … | podcast promos / quotes / PR / hiring | anchor + consequence | Non-development content; no consequence; see podcast section. |

**Residual-risk taxonomy:** the only *implementation-sensitive* survivors are **[60]** (freshness/dedup quality),
**[135][136][141]** (consequence-detection quality — the consequence condition is load-bearing precisely for
these niche/relevance spotted-news cases). All 32 remaining Weak are killed by ≥2 gates simultaneously, so a
single sub-optimal detector cannot re-admit them. **Coding-limitation note:** [141] is reaches anchor C but not
C-min — if the implementation uses the *smallest* anchor `(dev OR finding)`, [141] is killed twice (anchor +
consequence); if the maximal anchor is used, it is killed once (consequence). Either way it is excluded.

---

## 5. Podcast — separate eligibility path (CONFIRMED)

**Yes — podcast must be handled as a separate eligibility path, not forced through the RSS rule.**

Evidence (window, audit + 65-row contrast):
- **7 of 8 podcast posts are Weak; 0 of 8 are Strong** (`ai` category). The 7 Weak podcast rows in the 65-set
  ([18][44][63][72][78][149][150]) all fail the consequence gate; 6 of 7 also fail the `(dev OR finding)` anchor.
  None carries a source-established consequence.
- **Forcing the RSS rule onto podcast would reject 100% (7/7) of the sampled podcast Weak and would also fail the
  audit's 1 Reasonable podcast post** (an interview summary with no dated event/consequence) — i.e. it would
  terminate the podcast lane entirely, which is a policy decision, not a selection judgment.
- Podcast content is structurally different: interview/opinion summaries and promotional material, **never**
  SWS-gated (SWS is null for `ai` posts) and never run through the consequence/event funnel by design.
- **Recommended (from the audit, not implemented here):** podcast gets its **own lightweight eligibility check**
  (a SWS-style score or "does it report a concrete, dated claim from the guest/video" pre-queue gate) applied
  inside `scout-podcast.js` before queueing. Do NOT reuse the RSS consequence+event gate for the podcast lane.

---

## Final Answer

The proposed editorial-eligibility gate — in its **narrowed form `consequence AND fresh`** (optionally with the
`dev OR finding` anchor) — is **fully supported by the 65-story evidence**: 30/30 Strong retained and 35/35 Weak
rejected across all four correct formulations, category-independent, first-party- and announcement-tolerant, and
deterministically implementable. The lone Strong anomaly (#33) is reconciled: it is a **research finding** with
source-established consequence, rescued by the `finding` feature in the anchor and by `consequence` itself. The
implementation must (a) use **topic-signature** freshness, not title-token matching, and (b) treat
**consequence detection** as the load-bearing, calibration-critical component. Podcast receives a **separate
path**. No code, config, prompt, SWS, or pipeline change was made in producing this reconciliation.

---

*Read-only deliverable (Session #54, 2026-09-23). Evidence: `tagged.json` (65 joined rows), `code_kw.js`
(20-feature coding), `STORY_SELECTION_QUALITY_AUDIT.md`, `CONTRASTIVE_SELECTION_ANALYSIS.md`; all computations
reproducible and re-run during this session. `SELECTION_FUNNEL_FORENSIC.md` untouched.*

---

## Post-Reconciliation Status — Subsequent Implementation

**Date:** 2026-10-02 (appended by the documentation refresh; nothing above was altered).

This section exists to fix the **chronology** and to separate **historical evidence** from **implementation
history**. It does not revise, correct, or supersede any measurement, story number, formulation result, or
conclusion above, all of which remain the Session #54 historical record.

### STAGE 1 — SESSION #54 RECONCILIATION (this document)

A read-only evidence and reconciliation exercise. It re-tested candidate formulations against **all 65 coded
rows**, reconciled the reported Strong exception (#33), identified the smallest rule the evidence supported,
and confirmed podcast handling. **It changed nothing.**

Its **"✅ READY FOR IMPLEMENTATION"** decision meant: *the evidence supported proceeding with implementation at
that point.* It did **not** mean an implementation existed. No editorial-eligibility gate was running in
production when this document was written.

### STAGE 2 — SESSION #55 IMPLEMENTATION (subsequent)

The gate was subsequently implemented and committed as **`ddb8231`**, introducing
**`scout-agent/editorial-eligibility.js`** and the **STEP 7b** call site in `scout.js`.

The **later shipped deterministic rule** was:

```
ELIGIBLE  ⇔  (development OR finding)
             AND source-established consequence
             AND fresh
```

**This is the later implemented rule. It is not a rewrite of the historical Session #54 recommendation above**,
which was stated in its narrowed form as `consequence AND fresh` *(optionally with the `dev OR finding` anchor)*
and which deliberately kept several formulations open as evidence-derived options rather than one chosen rule.

**Implementation history — not new editorial rules.** The following are recorded as facts about what was
built, not as governing editorial policy:

- **Consequence detection** was ported verbatim from the calibrated gold harness used in the Session #54 work —
  it is the load-bearing, calibration-critical component identified by the historical analysis above.
- **Freshness** uses the Session #38 `storiesOverlap()` **topic-signature** approach (title **and** content),
  **not** title-token matching — consistent with the historical requirement in the Final Answer above.
- The freshness comparison runs against **recent published posts** within a bounded window.
- The gate executes at **STEP 7b**, i.e. **after SWS enforcement and before digest generation**. Ineligible
  stories are **skipped before generation** (capped articles marked seen, then continue).
- The implementation is **fail-open**: a recent-posts query failure or a gate crash treats the story as fresh
  and proceeds, so a database or gate failure cannot become a starvation source.
- **Podcast remains a separate eligibility path.** The RSS consequence gate did **not** become the podcast gate,
  consistent with Section 5 above; a dedicated podcast eligibility design was left as a later open item and is
  not back-filled here.

**Relationship to the historical formulations (kept deliberately distinct).** The historical analysis measured
and preserved several candidate rules as evidence — **C** `(dev OR finding OR decision OR event) AND
consequence AND fresh`, **C-min** `(dev OR finding) AND consequence AND fresh`, and **MIN** `consequence AND
fresh` — and recorded that `consequence AND fresh` was the smallest observational separator on the 65-story
sample, that `dev OR finding` was retained as **defensive structure** rather than as a sample necessity, that
**#33** required the `finding` feature to be rescued under the narrower anchor, that **#60** demonstrated why
freshness mattered (it leaks under a consequence-only probe), that `action` was **redundant** with `event`, and
that `decision` was **redundant** with the other features in this sample.

The later shipped implementation then **narrowed the anchor to `(development OR finding)`**. That narrowing is
**implementation history**, decided after this document was written. It has **not** been back-filled into the
historical tables above, which remain as measured.

**On #33 and #60 specifically.** The later implementation's `finding` feature is **consistent with** this
reconciliation's conclusion that #33 is a **research finding with source-established consequence** — the
implementation realises that conclusion, it does not revise it. Likewise the later implementation's
**topic-signature freshness** is **consistent with** the historical recommendation and with the #60 finding;
the historical test result is unchanged.

### STAGE 3 — SUBSEQUENT VERIFICATION (later, and separate)

Later live/test verification and the current operational state are documented in **`CLAUDE.md` §7A**. They are
**not** asserted by this document.

Three things must not be conflated:

| Layer | What it is |
|---|---|
| **Historical gold-set result** | 30/30 Strong retained and 35/35 Weak rejected on the **65-story coded population**, plus Formulation A = 29/30 (loses #33) — a property of that historical sample |
| **Implementation verification** | Later test-suite and live-production checks performed against the shipped code |
| **Current operational state** | Whatever `CLAUDE.md` §7A and live code currently say, including open questions |

**The historical 30/30 and 35/35 figures must NOT be represented as a current production accuracy guarantee.**
They describe a specific 65-story sample, not a promise about future output, and they do not substitute for
production measurement.

### Anti-pattern warnings preserved, not adopted as rules

The companion analyses warned against mechanical rules that would **reject first-party sources**, **reject
announcements**, **raise the SWS threshold**, or **reject analysis** categorically, and this reconciliation
emphasised that the supported formulations are **category-independent and first-party- and
announcement-tolerant**. Those remain **historical warnings from this window**. The later implementation must
not be described as having adopted them, and they are **not** governing rules. No new SWS threshold, editorial
rule, consequence definition, source rule, image rule, or publishing rule is introduced by this section —
those belong in the governing documentation if and when intentionally established there.
