# AITrends Project — Master Specification
**Last Updated:** 2026-06-04
**Owner:** Felix Okon
**Maintained by:** FAIT (Felicota Audio Infotech), Lagos

> This is the single source of truth for the entire AITrends.ng project.
> Both sub-projects (aitrends.ng and scout-agent) are governed by this file.
> Read this at the start of EVERY Claude Code session on either sub-project.

---

## Project Root Structure

```
WEBSITE_AI_Generated_Xai/
  └── aitrends-project/
        ├── CLAUDE.md                     ← THIS FILE — master spec for the whole project
        ├── SESSION_LOG.html              ← Master session log (all sessions, both projects)
        ├── TRAINING_MANUAL.html          ← Vibe-Coding training manual (12 chapters, 77KB)
        ├── aitrends.ng/                  ← Blog frontend (Next.js · Vercel)
        │     ├── AITRENDS_NG_SPEC.md    ← Detailed frontend spec (reference, not primary)
        │     └── SESSION_LOG_AITRENDS.html ← Archived aitrends.ng-only sessions
        └── scout-agent/                  ← Automation pipeline (Node.js · GitHub Actions)
              ├── SCOUT_AGENT_SPEC.md    ← Detailed scout spec (reference, not primary)
              └── SESSION_LOG_SCOUT.html ← Archived scout-agent-only sessions
```

**GitHub repos:**
- Blog frontend: https://github.com/FAIT-Blog/aitrends-ng
- Scout pipeline: https://github.com/FAIT-Blog/scout-agent

**Live site:** https://aitrends-ng.vercel.app
**Custom domain (pending DNS):** https://aitrends.ng

---

## 1. MISSION

AITrends.ng is a **fully autonomous AI news and blog platform** that captures the latest news,
events, happenings, innovations, and trends in Artificial Intelligence — with a **primary focus on
the Africa region, and Nigeria / West Africa in particular**. No daily human intervention. The
system runs itself automatically.

> **Content stance — Africa First:**
> Every post must be written FROM an African perspective. Global AI news is covered only when it
> has a specific, clear implication for African developers, founders, or Nigeria/West Africa's tech
> ecosystem. Posts must be written FROM an African lens — not global tech news with Africa
> mentioned at the end.

**Brand tagline:** "Africa's autonomous briefing on AI — news, trends, and what it means for builders here."
**Footer line:** "Today and the next AI trends"
**Powered by:** FAIT (small footer attribution — "by FAIT")
**Voice:** Punchy, opinionated, like a smart senior tech analyst briefing you — Africa-first.
**Audience:** Developers, founders, and AI practitioners in Nigeria, Ghana, Kenya, South Africa, and West Africa.

---

## 2. HOW THE TWO PARTS CONNECT

```
Africa RSS Feeds (TechCabal, Techpoint, Disrupt Africa, Ventureburn)  ←── PRIMARY
Global RSS Feeds (TechCrunch, VentureBeat, OpenAI, Google AI, HuggingFace, etc.)
        │
        ▼
  scout-agent/                         ← Node.js cron on GitHub Actions (every 6 hours)
    1. Fetches #scout-editor Slack channel (Felix's editorial inputs)
    2. Loads evergreen vocabulary from Supabase (40 baseline + learned terms)
    3. Fetches all RSS feeds
    4. Deduplicates against Supabase scout_memory table
    5. Scores articles by evergreen potential
    6. Groups by category, caps at MAX_ARTICLES=5
    7. Generates 800-word Africa-first digest via Gemini 3.5 Flash (ONE STORY ONLY)
    8. Submits image job to image provider (HF: returns null jobId; Fal/AI Horde: returns request_id)
    9. Saves to pending_posts table (status: pending_image) — nothing published yet
   10. Saves trending terms to evergreen_vocab in Supabase
   11. Marks articles as seen in scout_memory
   12. EXITS — Phase 2 (complete.yml, every 5 min) picks up pending posts:
       → Calls image provider → uploads to Supabase Storage → publishes → notifies Slack
        │
        ▼
  aitrends.ng/                         ← Next.js 16 on Vercel (auto-deploys from main)
    /api/posts/create                  ← Validates API key, writes to Supabase
        │
        ▼
  Supabase PostgreSQL                  ← stores posts, scout_memory, evergreen_vocab
        │
        ▼
  aitrends-ng.vercel.app               ← Live site, auto-served from Supabase
        │
        ▼
  Slack #aitrends-feed                 ← Success notification with title + cover image
```

**Critical link:** The `SCOUT_API_KEY` must match in three places simultaneously:
1. `scout-agent/.env` (local) → `SCOUT_API_KEY`
2. GitHub Actions Secrets → `SCOUT_API_KEY`
3. Vercel environment variables → `SCOUT_API_KEY`
Rotate all three together if you ever rotate the key.

---

## 3. BRAND IDENTITY

**Visual Style:**
- Background: `#0a0a0f` (near black, slight blue tint)
- Primary accent: `#2563eb` (electric blue)
- Secondary accent: `#f59e0b` (gold — nods to Nigerian colours)
- Text: `#e5e7eb`
- Muted text: `#6b7280`
- Font — headings: Sora (Google Fonts)
- Font — body: Inter (Google Fonts)
- Font — code: JetBrains Mono (Google Fonts)
- Border radius: 8px standard, 12px cards
- Card style: Dark surface (`#111827`), 1px border (`#1f2937`), subtle hover lift

**Cover Image Style (HuggingFace FLUX.1-schnell → Supabase Storage):**
- Images generated via HF FLUX.1-schnell and permanently stored in Supabase Storage (not lazy URLs)
- Africa-first: images must depict the specific story — not a generic "tech in Africa" scene
- Art styles rotate per post: watercolour painting, vector flat illustration, editorial ink sketch, bold risograph print
- BANNED compositions: person-at-laptop-near-window-with-city-view — explicitly prohibited in Gemini prompt
- Scene variety encouraged: financial flows, product shots, event scenes, maps, infrastructure, abstract concepts
- Every prompt ends with: "No text, no logos, cinematic composition."
- Images served from: `https://tixagzzcaeqdohuyrngl.supabase.co/storage/v1/object/public/post-images/covers/{id}.jpg`
- Use plain `<img>` tags for these images — never `<Image>` from next/image (no optimization proxy)

---

## 4. DATABASE SCHEMA (Supabase)

**Supabase project URL:** `https://tixagzzcaeqdohuyrngl.supabase.co`

### Table: `posts`
| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key, auto |
| title | text | post headline |
| slug | text | unique, auto-generated from title |
| content | text | full digest HTML |
| excerpt | text | 2-sentence summary for cards |
| category | text | `ai-models` · `anthropic` · `industry` · `tools` |
| tags | text[] | array e.g. `["claude", "anthropic", "llm"]` |
| cover_image_url | text | Pollinations.ai generated URL |
| cover_image_prompt | text | the prompt used to generate the image |
| source_urls | text[] | original RSS article URLs (attribution) |
| status | text | `published` · `draft` |
| auto_generated | boolean | true if Scout posted it |
| created_at | timestamptz | auto |
| published_at | timestamptz | set on publish |

### Table: `scout_memory`
| Column | Type | Notes |
|---|---|---|
| id | bigint | primary key, identity |
| feed_url | text | the RSS feed source |
| item_guid | text | RSS item unique ID |
| item_title | text | original article title |
| post_id | text | ID of published post (nullable) |
| created_at | timestamptz | auto |

Unique constraint on `(feed_url, item_guid)` — prevents Scout from re-publishing seen articles.

### Table: `evergreen_vocab`
| Column | Type | Notes |
|---|---|---|
| id | bigint | primary key, identity |
| term | text | unique, the vocab term |
| source | text | `auto` (Gemini-extracted) or `manual` |
| added_at | timestamptz | auto |

Self-updating vocabulary — Gemini extracts `TRENDING_TERMS` from each digest and saves them here. Scout loads these on the next run and merges with the 40-term hardcoded baseline to score articles.

---

## 5. AITRENDS.NG — BLOG FRONTEND

**Stack:** Next.js 16.2.6 · React 18.3.1 · TypeScript · Supabase · Vercel
**Local dev:** `npm run dev` → http://localhost:3000
**GitHub:** https://github.com/FAIT-Blog/aitrends-ng
**Deployed:** https://aitrends-ng.vercel.app

### Critical Rules — Frontend
- **Never upgrade React to 19** — stay on React 18.3.1 (tested, stable)
- **Never use `pages/` router** — App Router (`app/`) only
- **Never use `<form>` tags** — controlled React state and `onClick` handlers
- **Never use `npm audit fix --force`** — silently breaks dependencies
- **Never delete `/api/posts/create`** — Scout depends on it
- **Never make the API endpoint publicly writable** — always check `x-api-key` header
- **Never log real secrets in SESSION_LOG** — use `[REDACTED]`
- **Always use `unoptimized` on `<Image>` components** that render Pollinations URLs

### Pages & Routes
| Route | Description |
|---|---|
| `/` | Homepage — HeroPost + category filter tabs + PostGrid + Sidebar |
| `/category/[slug]` | Category-filtered feed |
| `/post/[slug]` | Individual post — cover image, body HTML, sources, share buttons, related posts |
| `/about` | What AITrends.ng is, how it works, who it's for |
| `/api/posts/create` | POST — Scout publishes here (x-api-key protected) |
| `/api/posts/draft` | POST — saves draft (x-api-key protected) |
| `/api/health` | GET — returns `{ status: "ok" }` |
| `/sitemap.xml` | Dynamic sitemap from all published posts |
| `/feed.xml` | RSS 2.0 feed — latest 20 posts |
| `/robots.txt` | Allows all crawlers, references sitemap |

### Key Files
```
aitrends.ng/
  app/
    page.tsx                   ← Homepage (force-dynamic, fetches from Supabase)
    layout.tsx                 ← Root layout — NavBar + Footer + global metadata
    about/page.tsx             ← About page
    post/[slug]/page.tsx       ← Individual post page (force-dynamic)
    category/[slug]/page.tsx   ← Category page (force-dynamic)
    sitemap.ts                 ← Dynamic sitemap (force-dynamic)
    feed.xml/route.ts          ← RSS feed
    robots.ts                  ← robots.txt
    globals.css                ← Brand CSS vars + Google Fonts import
    api/
      posts/create/route.ts    ← Scout's publish endpoint
      posts/draft/route.ts     ← Draft endpoint
      health/route.ts          ← Health check
  components/
    NavBar.tsx                 ← Site nav with active link highlight
    Footer.tsx                 ← FAIT attribution + "Today and the next AI trends"
    Sidebar.tsx                ← About blurb + category counts + latest posts
    HeroPost.tsx               ← Large featured post (two-column, unoptimized image)
    PostCard.tsx               ← Grid card (unoptimized image)
    PostGrid.tsx               ← 9-per-page grid with Load More button
    CategoryBadge.tsx          ← Coloured pill per category
  lib/
    supabase.ts                ← Lazy Proxy Supabase client (avoids build-time init)
    createPost.ts              ← Shared post creation with unique slug loop
    slugify.ts                 ← Slug generation utility
    types.ts                   ← Post TypeScript interface
  next.config.ts               ← Allows image.pollinations.ai + *.supabase.co
  supabase/schema.sql          ← Full DB schema + RLS policies
```

### Environment Variables — Frontend (Vercel + .env.local)
```env
NEXT_PUBLIC_SUPABASE_URL=https://tixagzzcaeqdohuyrngl.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=[REDACTED]
SUPABASE_SECRET_KEY=[REDACTED]
SCOUT_API_KEY=[REDACTED]           ← Must match scout-agent and GitHub Secret
SLACK_WEBHOOK_URL=[REDACTED]
NEXT_PUBLIC_SITE_URL=https://aitrends-ng.vercel.app
```

### What Has Been Built ✅
- Homepage, category pages, post page, about page
- HeroPost + PostCard + PostGrid + Sidebar + NavBar + Footer
- Dynamic metadata + OpenGraph + Twitter Card per post
- Sitemap.xml + feed.xml (RSS) + robots.txt
- `/api/posts/create` + `/api/posts/draft` + `/api/health`
- All images use `unoptimized` — Pollinations images now load correctly
- Footer updated: "Today and the next AI trends"
- About page updated: Gemini 3.5 Flash, real feed list, Africa-first mission

### Known Issues
- 🔴 Individual post pages (`/post/[slug]`) returning HTTP 404 on Vercel despite route file existing — under investigation (Round 2 task)
- 🟡 OG meta tags cannot be verified until post 404 is resolved

---

## 6. SCOUT AGENT — AUTOMATION PIPELINE

**Stack:** Node.js 24 · ES Modules · GitHub Actions (cron)
**Schedule:** `0 */6 * * *` — 00:00, 06:00, 12:00, 18:00 UTC every day
**GitHub:** https://github.com/FAIT-Blog/scout-agent
**Local path:** `aitrends-project/scout-agent/`

### Critical Rules — Scout
- **Never hardcode API keys** — all secrets from `.env` locally, GitHub Secrets in Actions
- **`SCOUT_API_KEY` must stay in sync** across three places: `.env`, GH Secret, Vercel
- **Always use ES modules** — `package.json` has `"type": "module"`. Never use `require()`
- **`index.js` is NOT used by GitHub Actions** — GHA calls `scout.js` directly. `index.js` is for VPS/server mode only
- **Never remove the Gemini fallback or retry logic** — `gemini-3.5-flash` gets 503 under load; 3-attempt retry with fallback to `gemini-2.5-flash` must stay
- **Feed categories must match exactly:** `anthropic`, `industry`, `ai-models`, `tools` — must match `VALID_CATEGORIES` in aitrends.ng `/api/posts/create`
- **`hasSeen()` throws on Supabase error** — intentional. A DB outage must halt the run, not cause duplicate posts
- **Keep `process.exit(0)` in the GHA run command** — Supabase realtime WebSocket keeps Node alive indefinitely without it
- **`maxOutputTokens` must stay at 8192 or higher** — 800-word content + markup + FAQ + all 5 fields exceeds 4000 tokens

### Pipeline (13 Steps)
```
0. Fetch Felix's editorial inputs from #scout-editor Slack (last 8h)
1. Load evergreen vocabulary from Supabase evergreen_vocab table
   (merged with 40-term hardcoded baseline)
2. Fetch all RSS feeds (17 feeds across 4 categories)
3. Deduplicate against scout_memory — skip already-seen articles
4. Score each article by evergreen potential (vocab match count)
5. Group by category, sort by score, cap at MAX_ARTICLES=5
6. Skip categories with < MIN_ARTICLES=2 fresh articles
7. Generate Africa-first 800-word digest via Gemini 3.5 Flash
   (fallback: gemini-2.5-flash on 503; retry up to 3 times, 12s delay)
8. Validate output: title + content >= 400 chars before proceeding
9. Build cover image URL via Pollinations.ai Flux model (fast generation ~2s)
10. Publish to aitrends.ng /api/posts/create
11. Save TRENDING_TERMS to evergreen_vocab in Supabase
12. Mark articles as seen in scout_memory (2-attempt retry on failure)
13. Notify Slack #aitrends-feed on success
```

### File Structure
```
scout-agent/
  scout.js       ← Core pipeline. Exports runScout(). Called by GHA.
                   Contains EVERGREEN_ENTITIES baseline (40 terms) + evergreenScore().
  gemini.js      ← Gemini API integration. Africa-first prompt. 3-attempt retry.
                   Primary: gemini-3.5-flash | Fallback: gemini-2.5-flash
                   Returns: { title, content, excerpt, imagePrompt, trendingTerms }
  feeds.js       ← All 17 RSS feeds (4 Africa-specific + 13 global)
  felix.js       ← Reads #scout-editor Slack channel (last 8h). Parses URLs + text.
  fetcher.js     ← Extracts content from URLs (YouTube transcripts or article body).
  memory.js      ← Supabase ops: hasSeen(), markSeen(), loadEvergreens(), saveEvergreens()
  publisher.js   ← POSTs to aitrends.ng /api/posts/create with SCOUT_API_KEY
  slack.js       ← Sends Block Kit notification to #aitrends-feed
  image.js       ← Builds Pollinations.ai URL with Flux model
  index.js       ← Long-running cron daemon (VPS deployment only — NOT used by GHA)
  .github/
    workflows/
      scout.yml  ← GHA workflow: cron + workflow_dispatch, Node 24, timeout 15min
```

### RSS Feeds (17 total)
**Africa-specific (primary — mission-critical):**
- TechCabal → `industry`
- Techpoint Africa → `industry`
- Disrupt Africa → `industry`
- Ventureburn → `industry`

**Anthropic:**
- Anthropic official → `anthropic`
- Hacker News Anthropic/Claude filter → `anthropic`

**Global AI Industry:**
- TechCrunch AI → `industry`
- VentureBeat AI → `industry`
- MIT Technology Review → `industry`

**AI Models:**
- OpenAI → `ai-models`
- Google AI → `ai-models`
- HuggingFace → `ai-models`
- Google DeepMind → `ai-models`
- Mistral AI → `ai-models`

**Tools & Dev:**
- Hacker News (AI/LLM filter) → `tools`
- Simon Willison → `tools`
- DeepLearning.AI The Batch → `tools`

### Environment Variables — Scout (.env + GitHub Secrets)
```env
SUPABASE_URL=https://tixagzzcaeqdohuyrngl.supabase.co
SUPABASE_SECRET_KEY=[REDACTED]
GEMINI_API_KEY=[REDACTED]
BLOG_API_URL=https://aitrends-ng.vercel.app/api/posts/create
SCOUT_API_KEY=[REDACTED]          ← Must match Vercel and aitrends.ng .env.local
SLACK_WEBHOOK_URL=[REDACTED]      ← Outgoing webhook to #aitrends-feed
SLACK_BOT_TOKEN=[REDACTED]        ← Bot token to READ #scout-editor
```

**GitHub Actions Secrets required:**
`SUPABASE_URL`, `SUPABASE_SECRET_KEY`, `GEMINI_API_KEY`, `BLOG_API_URL`, `SCOUT_API_KEY`, `SLACK_WEBHOOK_URL`, `SLACK_BOT_TOKEN`

**Slack app scopes required for `SLACK_BOT_TOKEN`:**
`channels:read`, `channels:history`, `groups:read`, `groups:history`

### Gemini Prompt — Africa-First Mandate
The editorial prompt in `gemini.js` enforces:
- Posts written FROM an African perspective — not global news with Africa at the end
- Lead paragraphs establish Africa/Nigeria/West Africa relevance FIRST
- Every section includes specific African implications woven throughout
- Bottom line reads: **"Bottom line for African builders:"**
- `TRENDING_TERMS` restricted to proper nouns only (company names, model names, product names) — no phrases, no verbs, no generic words

### How to Use #scout-editor
Post messages to the `#scout-editor` Slack channel. Scout reads it at the start of every run (last 8h window).

| What to post | What Scout does |
|---|---|
| YouTube URL | Fetches transcript (may be blocked on GHA — use manual paste) |
| Article URL | Fetches body text, feeds as source to Gemini |
| URL + note in same message | Extracts content + passes your note as editor instruction |
| Plain text | Added to Gemini prompt as editorial topic priority |

### How to Run Locally
```bash
# One-off run (same as GitHub Actions)
node -e "import('./scout.js').then(m => m.runScout()).then(() => process.exit(0)).catch(e => { console.error(e); process.exit(1) })"

# Long-running daemon (VPS only)
npm start   # runs index.js
```

### What Has Been Built ✅
- RSS fetching from 17 feeds (10 original + 7 added in Round 1)
- Supabase deduplication (scout_memory)
- Evergreen scoring — 40-term baseline + self-updating DB vocabulary
- Self-updating vocabulary — TRENDING_TERMS saved to evergreen_vocab after each run
- Felix editorial input via #scout-editor Slack channel
- YouTube transcript auto-fetch (fallback: manual paste)
- Article body text extraction via node-html-parser
- Africa-first Gemini prompt (rewritten Round 1)
- Story-first cover images via Pollinations.ai Flux model
- Gemini retry: 3 attempts, 12s delay, primary→fallback
- process.exit(0) — clean 78–122s runs
- markSeen() 2-attempt retry on Supabase failure
- Slack success notification + GHA failure alert

---

## 7. MASTER TO-DO LIST
*Updated: 2026-06-05*
*Status key: 🔴 Critical | 🟠 High | 🟡 Medium | 🔵 Low / Later*

### ✅ Completed — Round 1 (4 June 2026, morning)
- ✅ Africa-first Gemini prompt — editorial mandate rewritten; Africa lens is now the primary frame, not an afterthought
- ✅ TRENDING_TERMS quality — prompt tightened to proper nouns only (company names, model names, product names). No phrases, no verbs, no generic words.
- ✅ RSS feeds expanded 10 → 17 — added TechCabal, Techpoint Africa, Disrupt Africa, Ventureburn (Africa-specific industry); Hacker News Anthropic filter, Google DeepMind, Mistral AI (fixes skipped anthropic/ai-models categories)
- ✅ markSeen() retry — 2-attempt retry with 2s delay protects against duplicate posts on Supabase hiccup
- ✅ Footer updated — "Today and the next AI trends" (Footer.tsx line 13)
- ✅ All 4 tagline locations updated — Footer.tsx, app/layout.tsx, components/Sidebar.tsx, app/about/page.tsx
- ✅ About page accuracy — Gemini 1.5 Flash → 3.5 Flash; fake Reddit feeds removed; real 17-feed list; "Who it's for" and "What it is" rewritten for Africa-first mission; tagline updated
- ✅ Project restructured into `aitrends-project/` — both sub-projects moved and renamed; unified CLAUDE.md + merged SESSION_LOG.html written; individual specs renamed; both repos committed and pushed

### ✅ Completed — Combined Session #2 (4 June 2026, evening)
- ✅ **Pollinations root cause diagnosed** — `model=flux` added in Round 1 returned HTTP 402 (paid feature). `unoptimized` prop caused browser to fire 9 concurrent requests; Pollinations rate-limits to 1 per IP → 8 of 9 images failed. PostCard and HeroPost replaced with plain `<img>` tags + `onError` fallback to gradient placeholder.
- ✅ **Pollinations confirmed fully broken (x402 standard)** — Pollinations has adopted the x402 micro-payment protocol. All requests return 402 regardless of IP or model parameter. `migrate-images.js` run from GHA: 26 posts, 0 migrated, 26 failed — Pollinations unreachable server-side too.
- ✅ **Two-phase async image pipeline built** — Scout split into Phase 1 (generate + queue) and Phase 2 (image + publish). Nothing published until image is permanently stored. Three providers integrated behind one env var switch: `IMAGE_PROVIDER=huggingface|fal|aihorde`
  - `image-providers/huggingface.js` — HF Inference API, FLUX.1-schnell, synchronous (runs in Phase 2). Free $0.10/month renewing credits.
  - `image-providers/fal.js` — Fal.ai async queue, FLUX Schnell, Phase 1 submits + Phase 2 polls. $0.003/image.
  - `image-providers/aihorde.js` — AI Horde crowdsourced, SDXL, async queue, truly free forever.
  - `image-providers/index.js` — unified interface. Switch providers by changing one secret.
- ✅ **`complete.js` (Phase 2 worker)** — queries `pending_posts`, calls provider, uploads to Supabase Storage (`post-images` bucket), publishes to blog, notifies Slack. Stuck-post guard resets posts stuck in `generating` >15 min.
- ✅ **`complete.yml` (GHA workflow)** — runs every 5 minutes. Decoupled from Phase 1. Processes each pending post independently.
- ✅ **`scout.js` rewritten as Phase 1 only** — generates digest, submits image job (or notes HF runs in Phase 2), saves to `pending_posts`, marks articles seen, exits. No publish, no image download.
- ✅ **`memory.js`** — added `savePendingPost()`, `getPendingPosts()` (with stuck-guard), `updatePendingPost()`
- ✅ **Supabase Storage `post-images` bucket** — created and verified public. Images stored as `covers/{post_id}.jpg`
- ✅ **`pending_posts` Supabase table** — created with all columns including `updated_at` for stuck-guard
- ✅ **HF token configured** — `hf_[REDACTED]` (write role). Tested: HTTP 200 from HF FLUX.1-schnell API. FLUX model terms accepted on HuggingFace.
- ✅ **Live end-to-end test passed** — Phase 1 queued "Funding the Future: How Core DAO, Talstack, and Apple-Approved AI Agents are Driving AI Innovation in Africa". Phase 2 generated image via HF in 3s, uploaded 94KB JPEG to Supabase Storage, published to aitrends.ng, notified Slack. Total Phase 2 time: 14 seconds.
- ✅ **TechCabal Africa content confirmed working** — first article scored and selected: "Talstack Supports Nigerian Founders" — exactly the Africa-first content the mission requires.

### ✅ Completed — Combined Session #3 (5 June 2026)
- ✅ **"One story per post" rule enforced** — Gemini prompt rewritten: "ONE STORY ONLY" replaces "Give each article its own h3 section". Gemini now picks the single most Africa-relevant story and writes a focused piece. H3 headings explore angles (technical/cost/opportunity/risk), not separate source articles.
- ✅ **Repetitive image pattern broken** — Explicitly banned in `gemini.js`: "person at laptop near window with city view". Full prohibited-compositions list added. Risograph print added as 4th art style. Style rotation rule added (no same style in consecutive posts). Replaced single GOOD example with 3 diverse examples (naira→data-stream, scales-of-justice, brain-of-African-cities). "Young African professional + laptop + Lagos skyline" explicitly prohibited as lazy default.
- ✅ **Multi-topic post unpublished** — "Funding the Future: How Core DAO, Talstack, and Apple-Approved AI Agents..." set to `draft`. Hidden from site. Had 8 H3 sections covering 3 unrelated topics.
- ✅ **`regenerate-images.js` built** — queue-based: inserts all posts with Pollinations URLs into `pending_posts`. Phase 2 generates HF FLUX images one per 5-min cycle (~140 min total for 28 posts). First attempt used direct HF calls → 429 rate limit → rewritten to queue approach.
- ✅ **`complete.js` updated for image updates** — when `post_id` is already set in `pending_posts`, updates the existing post's `cover_image_url` directly in Supabase instead of creating a new post via `/api/posts/create`.
- ✅ **28 posts queued for image regeneration** — triggered 5 June 02:55 UTC. Phase 2 generating real HF FLUX images every 5 minutes and storing in Supabase Storage permanently.
- ✅ **Training manual created** — `TRAINING_MANUAL.html` (77KB, 1,617 lines, 12 chapters). See documentation section.
- ✅ **`FAIT-Blog/aitrends-project` GitHub repo created** — hosts `CLAUDE.md`, `SESSION_LOG.html`, `TRAINING_MANUAL.html`.

### 🔴 Critical
- [ ] **Fix individual post page 404s** — every Slack "Read Post →" link is broken. Route file `app/post/[slug]/page.tsx` exists and code is correct; Vercel not serving it. Investigate: check Vercel function logs, check slug format stored vs URL expected.
- [ ] **Fix broken RSS feeds** — 5 feeds failing: Ventureburn (415), HuggingFace blog (429), Google DeepMind (404), Mistral AI (404), Hacker News (429). Need correct feed URLs.

### 🟠 High
- [ ] **Monitor image regeneration** — 28 posts queued. Watch `complete.yml` every 5 min. All 28 should have real images within ~2.5 hours of queue trigger (5 June 02:55 UTC).
- [ ] **Verify new prompt produces focused single-topic posts** — next Phase 1 + Phase 2 cycle: is the post about ONE story with h3 angle headings? Is the image prompt avoiding the banned person+laptop+window pattern?
- [ ] **Submit `sitemap.xml` to Google Search Console** — manual task, no code, 10 minutes, unlocks all SEO work immediately

### 🟡 Medium
- [ ] **Verify OG meta tags** render correctly (once post 404 is fixed — `generateMetadata()` is already implemented correctly in `app/post/[slug]/page.tsx`)
- [ ] **Phase 5: Topic clustering** — find stories multiple sources cover simultaneously, produce one deep authoritative post instead of separate category digests

### 🔵 Later (after system is perfected)
- [ ] Email newsletter / subscriber form (Buttondown or Beehiiv — free)
- [ ] Social media auto-posting (Twitter/X, LinkedIn on each Scout publish)
- [ ] Admin dashboard on aitrends.ng (view, delete, manage posts without going into Supabase)
- [ ] Render deployment for Scout (replace unreliable GHA cron — `index.js` daemon already written)
- [ ] YouTube transcript proxy for GHA IP ranges
- [ ] PDF ingestion (manual paste is current fallback)

---

## 8. WHAT CLAUDE CODE MUST NOT DO (Both Projects)

- Never hardcode API keys, secrets, or webhook URLs in source files
- Never upgrade React to 19 — stay on React 18
- Never use `pages/` router in aitrends.ng — App Router only
- Never use `npm audit fix --force` — breaks dependencies silently
- Never delete `/api/posts/create` — Scout depends on it
- Never make the API endpoint publicly writable without the `x-api-key` check
- Never log real API keys, tokens, or secrets in SESSION_LOG — always `[REDACTED]`
- Never upgrade Prisma to version 7 (applies to BFX project, not AITrends)
- Never use `require()` in scout-agent — ES modules only (`"type": "module"`)
- Never remove the Gemini fallback or retry logic in gemini.js
- Never use Next.js `<Image>` for Pollinations URLs — use plain `<img>` tag with `onError` gradient fallback
- Never lower `maxOutputTokens` below 8192 in gemini.js
- Never call `publishPost()` from `scout.js` — publishing is Phase 2 only (`complete.js`)
- Never store a Pollinations lazy URL as `cover_image_url` in posts — always use permanent Supabase Storage URL
- Never change `IMAGE_PROVIDER` without ensuring the new provider's API key secret is also set

---

## 9. SESSION LOG REQUIREMENT

**Mandatory. Non-negotiable. This is how Felix stays informed.**

The master session log lives at:
```
aitrends-project/SESSION_LOG.html
```

**Every turn must be logged verbatim:**
- Felix's exact words — no paraphrasing
- Every tool call: Read, Write, Edit, Bash — with full output
- Every code change: before AND after in full
- Every error message: verbatim, with root cause and fix
- Every decision and why it was made
- All secrets as `[REDACTED]`

**No summaries. Ever.** If something took 10 tool calls to fix, all 10 are in the log — every command, every output, every file change before and after, every error and its root cause and fix.

**Why this matters — Felix's explicit instruction (4 June 2026):**
> "We need it to teach and encourage people/students to embrace Vibe-coding and to promote how
> efficient Claude Code can be and is."

This session log is a teaching document. It demonstrates what real autonomous Claude Code sessions look like — the thought process, the mistakes, the fixes, the decisions. Show everything. A reader who has never used Claude Code should be able to follow every step.

---

## 10. SESSION HISTORY

| # | Date | Project | Focus |
|---|---|---|---|
| aitrends S1 | 30 May 2026 | aitrends.ng | Full build: scaffold, React downgrade to 18, Supabase lazy Proxy client, all pages + components + API routes, 3 build attempts (TypeScript errors fixed), deploy to Vercel |
| scout S1 | 1 June 2026 | scout-agent | Pipeline audit + fixes: Node 24, remove silent failures, hasSeen() throws on error, Gemini validation, SCOUT_API_KEY rename, DeepLearning.AI feed fix, gemini-3.5-flash upgrade with 2.5 fallback, CLAUDE.md written |
| scout S2 | 2 June 2026 | scout-agent | Editorial upgrade: Africa-sentence prompt + evergreen scoring (40-term vocab) + Felix #scout-editor input channel + URL/YouTube extractor + self-updating evergreen_vocab + process.exit(0) fix (502s → 78s) + maxOutputTokens 4000 → 8192 |
| scout S3 | 3 June 2026 | scout-agent | Full system audit: GHA run history check, site audit (all 4 category pages), post 404 discovery, Africa-focus count (8/26=31%), root cause diagnosis (feeds are global — no Africa content), mission rewrite, tagline retire, master to-do list, image generation guidelines |
| combined S1 | 4 June 2026 AM | both | Round 1: Africa-first Gemini prompt, 17 feeds (+4 Africa), markSeen retry, footer updated, taglines updated, About page fixed, project restructured into aitrends-project/, unified CLAUDE.md + SESSION_LOG.html written |
| combined S2 | 4 June 2026 PM | both | Image pipeline rebuilt: Pollinations confirmed broken (x402 payment standard), plain `<img>` tags with onError fallback, two-phase async pipeline (Phase 1 queue / Phase 2 complete), 3 providers (HF/Fal/AI Horde) behind one env var, Supabase Storage `post-images` bucket, `pending_posts` table, HF FLUX.1-schnell confirmed working (HTTP 200, 3s generation), live end-to-end test passed — first Africa-first post with permanent Supabase image published |