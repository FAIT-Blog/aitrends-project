# AITrends Project — Master Specification
**Last Updated:** 2026-06-17 (Session #13 — admin dashboard architecture + risks documented, master build prompt written, platform research, full security audit + 9 fixes applied across scout-agent and aitrends.ng)
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
        ├── TRAINING_MANUAL.html          ← Vibe-Coding training manual (12 chapters, 1,867 lines)
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
| cover_image_url | text | Supabase Storage URL (HF FLUX.1-schnell generated image) |
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
SLACK_SIGNING_SECRET=[REDACTED]    ← Slack app Signing Secret — validates /api/slack/editorial webhooks
GITHUB_PAT=[REDACTED]              ← Fine-grained PAT, Actions R/W on FAIT-Blog/scout-agent — dispatches Phase 1 on editorial input
NEXT_PUBLIC_SITE_URL=https://aitrends-ng.vercel.app
```

### What Has Been Built ✅
- Homepage, category pages, post page, about page
- HeroPost + PostCard + PostGrid + Sidebar + NavBar + Footer
- Dynamic metadata + OpenGraph + Twitter Card per post
- Sitemap.xml + feed.xml (RSS) + robots.txt
- `/api/posts/create` + `/api/posts/draft` + `/api/health`
- Plain `<img>` tags with `onError` gradient fallback (Pollinations confirmed broken — x402)
- Footer updated: "Today and the next AI trends"
- About page updated: Gemini 3.5 Flash, real feed list, Africa-first mission
- ShareButtons on post pages: X, WhatsApp, LinkedIn, Telegram, Copy Link (clipboard + "Copied!" feedback)
- Continuous scroll via IntersectionObserver 300px lookahead — no "Load More" button
- **Security hardened (Session #13):**
  - `sanitize-html` on all AI-generated content before Supabase write (allowlist: h2–h4, p, strong, em, lists, a, blockquote, hr)
  - `crypto.timingSafeEqual()` on API key comparison in `create` and `draft` routes
  - `isSafeUrl()` validation on `cover_image_url` and `source_urls` before storage
  - `getAdminClient()` Supabase singleton in `slack/editorial/route.ts`

### Known Issues
- 🟡 OG meta tags — verify in Twitter Card Validator / Facebook Debugger (post 404 issue resolved in Session #5)

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
0. Read Felix's editorial inputs from editorial_queue Supabase table (pending rows)
   (Felix posts to #scout-editor → Slack Events API → /api/slack/editorial → DB → here)
   (Webhook also dispatches Phase 1 immediately via GITHUB_PAT — no 6h wait)
1. Load evergreen vocabulary from Supabase evergreen_vocab table
   (merged with 40-term hardcoded baseline)
2. Fetch all RSS feeds (25 feeds across 4 categories)
3. Deduplicate against scout_memory — skip already-seen articles
4. Score each article by evergreen potential (vocab match count)
5. Group by category, sort by score, cap at MAX_ARTICLES=5
6. Skip categories with < MIN_ARTICLES=1 fresh articles
7. Generate Africa-first 800-word digest via Gemini 3.5 Flash
   (fallback: gemini-2.5-flash on 503; retry up to 3 times, 12s delay)
8. Validate output: title + content >= 400 chars before proceeding
9. Submit image job to active provider (HF FLUX.1-schnell / Fal.ai / AI Horde) — returns jobId or null for HF
10. Save to pending_posts table (status: pending_image) — Phase 2 handles publish
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

### RSS Feeds (25 total)
**Nigeria & Africa AI — AI-dedicated portals (primary):**
- AIBase Nigeria → `industry`
- Africa AI News → `industry`
- iAfrica → `industry`
- AI in Nigeria → `industry`
- Techeconomy Nigeria → `industry`
- CIO Africa → `industry`
- Innovation Village → `industry`

**Africa tech media:**
- TechCabal → `industry`
- Techpoint Africa → `industry`
- Disrupt Africa → `industry`
- Ventureburn → `industry`
- Premium Times Nigeria → `industry`
- BusinessDay Nigeria → `industry`
- The Eagle Online → `industry`

**Anthropic (2 feeds — satisfies MIN_ARTICLES=2):**
- Anthropic official (GitHub mirror) → `anthropic`
- Hacker News Anthropic filter (points≥10) → `anthropic`

**Global AI Industry:**
- TechCrunch AI → `industry`
- VentureBeat AI → `industry`
- MIT Technology Review → `industry`

**AI Models:**
- OpenAI → `ai-models`
- Google AI → `ai-models`
- HuggingFace → `ai-models`
- Google DeepMind → `ai-models`

**Tools & Dev:**
- Hacker News (native RSS) → `tools`
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
TAVILY_API_KEY=[REDACTED]         ← Web search for same-story cross-referencing (tavily.com)
```

**GitHub Actions Secrets required:**
`SUPABASE_URL`, `SUPABASE_SECRET_KEY`, `GEMINI_API_KEY`, `BLOG_API_URL`, `SCOUT_API_KEY`, `SLACK_WEBHOOK_URL`, `SLACK_BOT_TOKEN`, `TAVILY_API_KEY`

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
- RSS fetching from 25 feeds (10 original + 7 Round 1 + 8 Session #5 + 1 Session #6: HN Anthropic)
- Supabase deduplication (scout_memory)
- Evergreen scoring — 40-term baseline + self-updating DB vocabulary
- Self-updating vocabulary — TRENDING_TERMS saved to evergreen_vocab after each run
- Felix editorial input via #scout-editor Slack channel
- YouTube transcript auto-fetch (fallback: manual paste)
- Article body text extraction via node-html-parser
- Africa-first Gemini prompt (rewritten Round 1)
- Story-first cover images via HF FLUX.1-schnell → permanently stored in Supabase Storage
- Gemini retry: 3 attempts, 12s delay, primary→fallback
- process.exit(0) — clean 78–122s runs
- markSeen() 2-attempt retry on Supabase failure
- Slack success notification + GHA failure alert
- AI signal gate (`hasAISignal()`) — blocks non-AI articles before Gemini (commit 000c25f)
- Search-aggregate pipeline — `search.js` (Tavily), `verify.js` (named entity overlap), blend mode (commit 47a0c8c)
- Self-learning priority sources — `source_reputation` table, promote at 5 hits, demote after 7 silent days

---

## 7. MASTER TO-DO LIST
*Updated: 2026-06-17*
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

### ✅ Completed — Combined Session #5 (6 June 2026, early hours)
- ✅ **Phase 1 false-failure fixed** — "Trigger Phase 2" step removed from `scout.yml`. GITHUB_TOKEN on scheduled runs cannot dispatch workflow_dispatch (HTTP 403) regardless of `permissions: actions: write`. cron-job.org already handles Phase 2 every 30 min reliably. False failures were causing GHA to skip Phase 1's own scheduled runs (00:00 UTC June 6 not fired).
- ✅ **3 broken RSS feeds removed** — Mistral AI (no RSS found, all paths 404), hnrss.org Anthropic filter (hnrss.org 502), hnrss.org Tools filter (hnrss.org 502). Feed count: 17 → 15.
- ✅ **Google DeepMind feed fixed** — URL changed from `/discover/blog/rss.xml` → `/blog/rss.xml` (200 confirmed).
- ✅ **Hacker News tools feed replaced** — hnrss.org/frontpage → `news.ycombinator.com/rss` (native HN RSS, GET returns valid XML).
- ✅ **Slack notification URL fixed** — `complete.js` image-regeneration branch was using UUID as post URL (`/post/{uuid}`). Now queries Supabase for slug before building URL. Root cause of "every Slack link broken" report.
- ✅ **Post page 404 confirmed NOT a code bug** — `curl` tests confirm `/post/{slug}` returns HTTP 200 on Vercel. The broken links were all UUID-based URLs from the Session #3 regeneration batch.
- ✅ **Phase 1 manually triggered + verified** — run 27051910295: ✓ success in 1m32s, 3 posts queued (industry, ai-models, tools), all 15 feeds fetched without errors.
- ✅ **cron-job.org Phase 1 trigger configured and confirmed** — run 27053359015: ✓ success 1m10s at 05:06 UTC, triggered by cron-job.org workflow_dispatch. All 15 feeds clean, 2 posts queued. Pipeline is now fully autonomous: both phases driven by cron-job.org (Phase 1 every 6h, Phase 2 every 30 min). GHA native cron is passive fallback only.
- ✅ **Fabricated Africa content fixed** — root cause was gemini.js lines 31+46 ("translate them into African context" + "fill it yourself"). These permitted Gemini to invent Africa connections for non-Africa stories. Replaced with a REJECTION GATE: if no genuine Africa/Nigeria relevance exists, Gemini outputs SKIP_NO_AFRICA_RELEVANCE and Scout silently skips the category. Tested: ai-models skipped correctly (⏭️), industry published genuine Nigeria story (Glovo/Paystack/Moniepoint Lagos summit).
- ✅ **3 Nigerian news feeds added** — Premium Times Nigeria, BusinessDay Nigeria, The Eagle Online (all HTTP 200 confirmed). industry category feeds: 7 → 10. Nigeria now has dedicated news sources so genuine Nigerian AI stories (e.g. AI deepfakes warnings, fintech AI) enter the pipeline.
- ✅ **Simon Willison removed** — personal UK developer blog was source of the WebAssembly/micropython fabricated Africa post.
- ✅ **Topic clustering added to scout.js** — addClusterScores() finds articles from DIFFERENT sources sharing 2+ title keywords and adds cluster bonus (×3 weight). Stories covered by multiple Nigerian outlets float to the top automatically. Log shows [ev:N cl:N] scores per article and 🔗 cluster detected message.

### ✅ Completed — Combined Session #4 (5 June 2026, evening)
- ✅ **Phase 1 → Phase 2 direct trigger** — `scout.yml` now fires `complete.yml` via `workflow_dispatch` immediately after every successful Phase 1 run (commit 8ada833). GHA's 15-min cron was running 0 times — posts were sitting in pending queue for hours. Direct trigger fixes this; posts now publish within minutes of being queued.
- ✅ **`scout.yml` permissions fix** — added `permissions: actions: write` at workflow level (commit adae741). Without it, `GITHUB_TOKEN` returned HTTP 403 when Phase 1 tried to dispatch Phase 2 — Phase 1 marked itself as failed even though content was queued correctly.
- ✅ **cron-job.org external trigger** — Phase 2 now triggered every 30 minutes from cron-job.org as a reliability safety net. PAT scoped to `FAIT-Blog/scout-agent` Actions: write. Confirmed working (HTTP 204). Combined with Phase 1 direct trigger, Phase 2 now has two independent trigger paths.
- ✅ **ShareButtons component** — `components/ShareButtons.tsx` added. Replaced the two inline share links on post pages with five share options: X (Twitter), WhatsApp (critical for Nigerian/West African audience), LinkedIn, Telegram, Copy Link (clipboard copy with 2-second "Copied!" feedback, brand-colour hover). Commit ed4fdd6 on aitrends-ng repo.
- ✅ **Image regeneration complete** — all 28 posts with Pollinations URLs replaced with permanent HF FLUX images stored in Supabase Storage. Phase 2 ran every cycle; all 28 confirmed migrated.

### ✅ Completed — Combined Session #3 (5 June 2026)
- ✅ **"One story per post" rule enforced** — Gemini prompt rewritten: "ONE STORY ONLY" replaces "Give each article its own h3 section". Gemini now picks the single most Africa-relevant story and writes a focused piece. H3 headings explore angles (technical/cost/opportunity/risk), not separate source articles.
- ✅ **Repetitive image pattern broken** — Explicitly banned in `gemini.js`: "person at laptop near window with city view". Full prohibited-compositions list added. Risograph print added as 4th art style. Style rotation rule added (no same style in consecutive posts). Replaced single GOOD example with 3 diverse examples (naira→data-stream, scales-of-justice, brain-of-African-cities). "Young African professional + laptop + Lagos skyline" explicitly prohibited as lazy default.
- ✅ **Multi-topic post unpublished** — "Funding the Future: How Core DAO, Talstack, and Apple-Approved AI Agents..." set to `draft`. Hidden from site. Had 8 H3 sections covering 3 unrelated topics.
- ✅ **`regenerate-images.js` built** — queue-based: inserts all posts with Pollinations URLs into `pending_posts`. Phase 2 generates HF FLUX images one per 5-min cycle (~140 min total for 28 posts). First attempt used direct HF calls → 429 rate limit → rewritten to queue approach.
- ✅ **`complete.js` updated for image updates** — when `post_id` is already set in `pending_posts`, updates the existing post's `cover_image_url` directly in Supabase instead of creating a new post via `/api/posts/create`.
- ✅ **28 posts queued for image regeneration** — triggered 5 June 02:55 UTC. Phase 2 generating real HF FLUX images every 5 minutes and storing in Supabase Storage permanently.
- ✅ **Training manual created** — `TRAINING_MANUAL.html` (12 chapters, now 1,867 lines after Session #4 additions). See documentation section.
- ✅ **`FAIT-Blog/aitrends-project` GitHub repo created** — hosts `CLAUDE.md`, `SESSION_LOG.html`, `TRAINING_MANUAL.html`.

### ✅ Completed — Combined Session #7 (8–9 June 2026)
- ✅ **Irrelevant source_urls fixed** — `relevantSourceUrls()` added to scout.js. Filters articles scoring 0 on evergreen vocab with no cluster signal and no full text extracted. Sports, crime, politics articles no longer appear as source attribution. Confirmed: "3 relevant of 5 fetched" on first run. (d1e2629)
- ✅ **Topic deduplication added** — `getRecentDuplicate()` added to scout.js. Checks Supabase for posts published in last 24h in same category; if 3+ title keywords overlap, new post is skipped and articles marked seen. Fixes 3× MTN story problem. (d1e2629)
- ✅ **Forced phrases banned from Gemini prompt** — BANNED PHRASES block added: "For developers in Lagos, Accra, and Nairobi" and all city-list permutations explicitly prohibited. Was appearing in 10+ posts as an AI template filler. (d1e2629)
- ✅ **Title variety enforced in Gemini prompt** — TITLE VARIETY block: mandates rotation between plain declarative, factual lead, direct statement, question-turned-statement. Ban on starting every title with "Why/How" or ending with "for African Builders". (d1e2629)
- ✅ **Bold keyword stuffing restricted** — BOLD TEXT RULES block: max 3 bold phrases per post, first mention only, never repeat-bold the primary keyword. Eliminates "**national AI strategy Africa** ×7" pattern. (d1e2629)
- ⚠️ **Image recreation (jimp) — failed, reverted in Session #8** — jimp composite() produced solid black images (rgb(1,2,10) on every pixel). JPEG pixels loaded without alpha channel; jimp treated as fully transparent; overlay×0.28 + bg×0 = black. Reverted to raw source photos. (d1a20c3 broken → 9ca0dea reverted)
- ✅ **Phase 1 live test confirmed** — run 27162186006: 60 fresh articles, 3 of 5 source URLs relevant, Yoco/Dyner queued with non-formulaic title, tools SKIP'd correctly.

### ✅ Completed — Combined Session #8 (10 June 2026)
- ✅ **Black images fixed** — `styleWithAI()` removed from complete.js entirely. jimp composite produced rgb(1,2,10) solid black (JPEG alpha=0 bug). Raw source editorial photos now used directly. (9ca0dea)
- ✅ **Asterisks fixed** — `publisher.js` strips `**text**` → `text` before every publish. gemini.js FORMATTING RULES: explicit ban on `**markdown**` in HTML content. (9ca0dea)
- ✅ **Gemini simplified — capability test** — full 150-line analytical digest prompt replaced with a 2-task prompt: (1) rephrase title, (2) accurate rewrite matching source word count. No 800-word minimum. No synthesis. No fabrication. Single article passed to generateDigest(). (9ca0dea)
- ✅ **Source URL reduced to 1** — `sourceUrls = [topArticle[0].link]`. Exactly the article Gemini rewrote from. No more WWDC articles credited to Nedbank stories. (9ca0dea)
- ✅ **Mobile responsive** — HeroPost stacks to single column at ≤640px. NavBar scrolls horizontally on mobile. Media query in globals.css, class names on HeroPost.tsx and NavBar.tsx. (e916384)

### ✅ Completed — Session #11 (13 June 2026)
- ✅ **Editorial queue rows 1 + 2 recovered** — Both rows were incorrectly marked `consumed` by old code (a422cb8, pre-221154e) during the 10:15 UTC Phase 1 run. The old code called `consumeEditorialRows()` inside the RSS category loop after any category success — meaning the deepfake post's title was written into Felix's Substack rows even though the Substack content was never passed to Gemini. Diagnosis confirmed by querying `pending_posts` (only deepfake post ID 80 at 10:15 UTC). PATCH to Supabase reset both rows: `status → pending`, `used_in_post → null`, `consumed_at → null`, `posted_at → null`. All three editorial rows now clean and pending.
- ✅ **Substack content confirmed extractable** — `fetchContent()` tested on all three Substack URLs: all return 4,019 chars of real content. The problem was never Substack blocking — it was `consumeEditorialRows()` being called at the wrong point in the pipeline.
- ✅ **Residual risk identified** — `consumeEditorialRows()` remains inside the RSS category loop as a fallback. If Step 0.5 fails for any reason (network error on URL fetch), editorial rows will be consumed again with the first RSS category post that succeeds. Recommendation: remove the call from category loop entirely.
- ✅ **fetcher.js confirmed working on WordPress SmartMag (aibase.ng)** — The CONTENT_SELECTORS cascade (`.entry-content` first, `article` last, 200-char minimum) is deployed and working. Fixes the 78-char sidebar widget extraction that caused thin posts. Commit 07f3f4b.
- ✅ **TRAINING_MANUAL Chapter 20 added** — Editorial consumption bug mechanics, database repair procedure, residual risk.

### ✅ Completed — Session #10 (13 June 2026)
- ✅ **Slug truncation fixed** — `lib/slugify.ts` now trims at last complete word boundary before 80 chars. No trailing hyphens. Affects all future posts. Commit 1370f92 on aitrends-ng.
- ✅ **Editorial override (Africa GATE exemption)** — `generateEditorialDigest()` added to `gemini.js`. `scout.js` Step 0.5 now generates a dedicated post from Felix's editorial sources BEFORE the RSS loop — Africa GATE suspended, editorial authority applies. If `felixInputs.sources.length > 0`, a post is always queued from Felix's content. Commit 221154e on scout-agent.
- ✅ **AI_SIGNAL expanded** — Added compound forms (`ai-powered`, `ai-driven`, `ai-enabled`, `ai-based`, `ai-generated`, `ai-assisted`) and domain terms (`chatbot`, `natural language`, `computer vision`, `predictive analytics`, `speech recognition`, `language model`, `multimodal`, `robotics`). The hyphenated form gap was the main reason articles like "AI-powered fintech" were failing the gate. Commit 221154e.
- ✅ **MIN_ARTICLES lowered 2 → 1** — The Africa GATE is the quality guard, not article count. Requiring 2 AI-signal articles was too restrictive for categories where dedicated AI portals publish infrequently. 1 strong AI article is sufficient for Gemini to evaluate. Commit 221154e.
- ✅ **TRAINING_MANUAL web session explained** — The web Claude was correct: TRAINING_MANUAL.html lives in `FAIT-Blog/aitrends-project`, not `aitrends-ng`. Web sessions only see the repo they're given. Claude Code (VSCode extension) has full local filesystem access so can always reach it. No fix needed — file is in the right place.

### ✅ Completed — Session #8 Continuation + Session #9 (12 June 2026)
- ✅ **5-post audit passed** — No asterisks, 1 source URL per post, title variety confirmed. South Africa football post (AI in FIFA stadium management) left published for study — traces to BusinessDay Nigeria feed.
- ✅ **3 off-topic posts confirmed by Felix** — MTN (general telecom), BNPL ×2 (buy now pay later fintech — no AI angle). Felix: "The MTN one and the BNPL ones — those are off-topic."
- ✅ **AI signal gate built** — `hasAISignal()` added to `scout.js` (commit 000c25f). Checks title + fetched body against explicit AI keyword list before evergreen scoring. Articles with no AI signal dropped before Gemini call. Implicit filter: fast, free, no API cost.
- ✅ **Push failure diagnosed** — 000c25f committed on Felix's local machine. Remote container had no GitHub credentials → "Repository not found" on git push. MCP push_files also 403. Push completed in Session #9 (this session).
- ✅ **Chapter 15 added to TRAINING_MANUAL.html** — "Content Signal Filtering — Blocking Non-AI Articles". Covers implicit vs explicit filter distinction, hasAISignal() implementation, deployment status.
- ✅ **Sidebar spacing fixed** — Latest Posts titles no longer overlap. `gap: 2` → 0, padding `8px` → `14px`, lineHeight `1.45` → `1.65`. (22b44db)
- ✅ **Hover effects added site-wide** — blue glow on post titles, hero headline, sidebar links, nav links, logo. CSS classes in globals.css + classNames on components. (be48564)
- ✅ **Search-aggregate pipeline built** — `search.js` (Tavily API), `verify.js` (named entity same-story check), `generateBlendedDigest()` in gemini.js, self-learning `source_reputation` table (promote at 5, demote after 7 days). (47a0c8c)
- ✅ **Tavily API key added** — `.env` updated. Key verified: 3 results for MTN/Alipay query confirmed working.
- ✅ **Tavily API key added and confirmed in GH Secrets** — key added to `.env` (Session #9). CONFIRMED by Felix screenshot (Session #12) that it was already in GitHub Actions Secrets.
- ✅ **TAVILY_API_KEY env wire fixed** — Secret existed in GH Secrets but was NOT mapped in scout.yml env block for the "Run Scout" step. Added `TAVILY_API_KEY: ${{ secrets.TAVILY_API_KEY }}`. Commit: `1b5512d` (Session #12).

### ✅ Completed — Session #12 (14 June 2026)
- ✅ **`consumeEditorialRows()` removed from RSS category loop** — Definitively removed. Replaced with permanent explanatory comment: "The RSS loop must never touch editorial_queue rows." The invariant is enforced in code, not just documentation. Commit: `5a22b5c`.
- ✅ **Editorial instruction promoted to primary directive** — `generateEditorialDigest()` in gemini.js rewritten. Felix's notes/instructions now appear as `EDITOR'S INSTRUCTION (this overrides all other directives)` — the very first thing Gemini reads. Africa framing reduced to one optional closing sentence. Felix's editorial authority supersedes all pipeline programming. Commit: `5a22b5c`.
- ✅ **Tavily fallback added to felix.js** — `tavilyExtract(url)` added. When `fetchContent()` fails on GHA-blocked URLs (Substack mobile-share links fail in 80ms), falls back to `https://api.tavily.com/extract`. Content returned as `[Article content]: ...`. Commit: `467a106`.
- ✅ **Diagnostic logging added to scout.js** — When a category is skipped by hasAISignal(), logs the top 5 rejected article titles with source names. Makes future debugging immediate without re-running. Commit: `467a106`.
- ✅ **`TAVILY_API_KEY` mapped in scout.yml env block** — Root cause of "TAVILY not set" across all GHA runs: secret existed in GitHub Secrets but was never in the workflow step's `env:` block. Added `TAVILY_API_KEY: ${{ secrets.TAVILY_API_KEY }}`. Commit: `1b5512d`.
- ✅ **Continuous scroll implemented on site** — `PostGrid.tsx` rewritten. Removed "Load More" button, replaced with IntersectionObserver watching a zero-height sentinel below the card grid. `rootMargin: '300px'` triggers 300px before scroll reaches bottom — seamless on mobile finger-flick scrolling. Observer disconnects when all posts loaded. End-of-list indicator shows post count. Commit: `c2311ae` on aitrends-ng.
- ✅ **Claim verification lesson** — Claude Code made wrong claim "TAVILY_API_KEY not in GitHub Secrets" without running `gh secret list`. Root cause was env block gap, not missing secret. Lesson: before claiming a secret is absent, always run `gh secret list --repo <org>/<repo>` AND read the workflow env block to confirm the secret is actually passed to process.env.

### ✅ Completed — Session #13 (16–17 June 2026)
- ✅ **Admin dashboard architecture documented** — Full architecture, risks, and mitigations written in session. Supabase `scout_config` singleton table, Phase 1 config-read-once pattern, standing instruction injection position, category sync requirement across 3 places, SSRF/gate-off/config-missing risks all documented. No implementation yet — design decision recorded for future build.
- ✅ **Consolidated master to-do list compiled** — 4 categories (Housekeeping, Pipeline Bugs, Admin Dashboard, SEO, Backlog) with priority ordering and suggested execution sequence for next session.
- ✅ **Master build prompt written** — 18-section, ~5,000-word self-contained prompt covering the complete project from scratch to future vision with 99% replication precision. Includes all architecture, schema, file structure, Gemini prompt rules, banned phrases, security rules, bug history, and future roadmap.
- ✅ **Platform research completed** — No Africa-first fully autonomous AI news platform found. Closest global equivalents (The Batch, TLDR, Ben's Bites) are all human-curated. AITrends.ng's specific combination of autonomous publishing + Africa-first mandate + two-phase image pipeline + self-learning vocab is novel.
- ✅ **Full security audit completed** — 9 vulnerabilities identified across 6 files. 3 critical (prompt injection, SSRF, HTML injection), 2 high (no rate limiting, timing attack), 4 medium/low. Full impact assessment written for each fix before implementation.
- ✅ **SEC-01 — Prompt injection defence** — `<SOURCE_CONTENT>` XML boundaries added around all external article text in `generateDigest()`, `generateBlendedDigest()`, `generateEditorialDigest()` in `gemini.js`. Prevents malicious RSS content from hijacking Gemini instructions. Commit: `f6788c7`.
- ✅ **SEC-02 — SSRF guard on image downloads** — `isSafeImageUrl()` added to `complete.js`. Validates URLs before every external image fetch (`downloadDirectImage`, `trySourceImage`, and the og:image parsed from HTML). Blocks AWS/GCP metadata endpoints, localhost, and RFC 1918 private IP ranges. Commit: `f6788c7`.
- ✅ **SEC-03 — HTML sanitization on post content** — `sanitize-html@2.17.5` installed in aitrends.ng. Applied to `create/route.ts` before Supabase write. Allowlist: `h2/h3/h4`, `p/br`, `strong/em`, `ul/ol/li`, `a/blockquote/hr`. Strips `script`, `iframe`, `style`, `on*` event handlers, and `javascript:/data:` URLs. Allowlist confirmed against actual published post tags (`h3`, `p`, `strong`). Commit: `6f2dce5`.
- ✅ **SEC-05 — Timing-safe API key comparison** — `crypto.timingSafeEqual()` replaces `!==` in `create/route.ts` and `draft/route.ts`. Prevents timing-based API key brute-force attacks. Commit: `5fe2a15`.
- ✅ **SEC-06 — URL validation on stored fields** — `isSafeUrl()` helper added to `create/route.ts` and `draft/route.ts`. Validates `cover_image_url` and each `source_urls` entry — rejects `javascript:`, `data:`, and non-http/https schemes before Supabase write. Commit: `5fe2a15`.
- ✅ **SEC-07 — Supabase singleton in Slack webhook** — `getAdminClient()` lazy singleton replaces per-request `createClient()` in `slack/editorial/route.ts`. One connection pool shared across all webhook invocations. Commit: `5fe2a15`.
- ✅ **SEC-08 — Publisher request timeout** — `AbortSignal.timeout(45000)` added to `publishPost()` fetch call in `publisher.js`. Prevents Phase 2 hanging indefinitely on Vercel cold start + Supabase sleep. On timeout, post resets to `pending_image` and retries next cycle. Commit: `f6788c7`.
- ✅ **SEC-09 — Extended markdown stripping** — `publisher.js` now strips `*italic*`, `__bold__`, `_italic_`, `` `code` `` in addition to existing `**bold**`. Commit: `f6788c7`.
- ✅ **SEC-04 deferred** — Rate limiting on `/api/posts/*` not yet implemented. API key is the primary protection at current scale. Revisit when admin dashboard is built. Upstash Redis approach documented for future implementation.

### 🔴 Critical
- ✅ **Add second Anthropic feed source** — `hnrss.org/newest?q=Anthropic&points=10` added to feeds.js (Session #6). anthropic category now has 2 feeds; MIN_ARTICLES=2 satisfied. Feed count: 24 → 25.
- ✅ **Assess Gemini capability after simplification** — Audit passed (Session #8 continuation, 12 June): 5 posts reviewed. No asterisks, 1 source URL, title variety confirmed. Gemini accurate-rewrite prompt working as intended.
- ✅ **AI signal gate pushed** — 000c25f pushed to FAIT-Blog/scout-agent in Session #9.
- ✅ **Editorial system redesigned** — Slack polling → Events API push → Supabase queue → Slack threaded reply. (a422cb8 scout-agent, 7d375b3 aitrends-ng). Pending one-time setup: run add-editorial-queue.sql, add SLACK_SIGNING_SECRET to Vercel, enable Events API in Slack app pointing at https://aitrends-ng.vercel.app/api/slack/editorial.
- ✅ **One-time editorial setup complete** — SQL run, Events API enabled, SLACK_SIGNING_SECRET + GITHUB_PAT added to Vercel, cron-job.org PAT updated, Vercel redeployed. Verified: Phase 1 dispatched immediately on editorial input (21:29 UTC, run 27444126665). Editorial rows stay pending until a category queues successfully (expected — Africa GATE correctly skipped that run).
- ✅ **`consumeEditorialRows()` removed from RSS category loop** — Done in Session #12 (commit 5a22b5c). Replaced with permanent comment explaining the invariant. The RSS loop will NEVER touch editorial_queue rows.
- ✅ **Security audit + 8 fixes applied** — Session #13. SEC-01 prompt injection, SEC-02 SSRF, SEC-03 HTML sanitization, SEC-05 timing-safe key compare, SEC-06 URL validation, SEC-07 Supabase singleton, SEC-08 publisher timeout, SEC-09 extended markdown strip. Three commits across both repos.
- [ ] **HN Anthropic feed intermittency (monitor)** — hnrss.org returning 502 on roughly half of Phase 1 runs. Need a stable second Anthropic source.
- [ ] **Old posts need retroactive cleanup** — posts published before Session #8 still have: markdown asterisks in body (publisher.js strip only applies to future posts), irrelevant source_urls, keyword stuffing. Bulk Supabase content edit needed.
- [ ] **Google Doc corrections pending** — Felix shared a doc with specific post edits. Awaiting explicit go-ahead.
- [ ] **BusinessDay Nigeria sports feed** — South Africa football post traced to BusinessDay feed. Monitor: does hasAISignal() filter enough, or does BusinessDay need removal? Check next 3–5 industry posts.
- [ ] **Nigerian feeds are general news** — Still influencing article scoring. Monitor whether published topics drift into general business/politics now that hasAISignal() gate is live.
- [ ] **Fabrication in ai-models category** — Sierra Leone education post fabricated from DeepMind voice translation sources (Session #8 audit). ai-models category articles rarely have genuine Africa relevance — Africa GATE should reject them more aggressively. May need category-specific GATE threshold.
- ✅ **Editorial specification implemented** (b473367) — 7 new AI-focused Nigerian/African feeds; 4-section structure; source article og:image as primary; AI illustration as fallback.
- ✅ **Fabrication root cause fixed** (3fd9671) — fetchContent() now called for all RSS articles (4000 chars real text to Gemini). SOURCE RULES added to prompt.
- ✅ **Post quality assessed — Yoco/Dyner post** — non-formulaic title ✅, no forced phrases, 3 relevant source URLs. Passed.
- ✅ **"People Also Ask" correctly implemented** — (21f84bc, 0ed408f)
- ✅ **Full SEO compliance pass completed** — (6817792, 060bc61)

### 🟠 High
- [ ] **Submit `sitemap.xml` to Google Search Console** — manual task, 10 minutes. All SEO work is invisible to Google until this is done. This is the single highest-leverage remaining action.
- [ ] **Verify post quality on new posts** — read Yoco/Dyner post body in full. Does it cite verifiable facts? No forced phrases? Varied title structure confirmed from log but body needs human review.
- [ ] **Monitor Nigerian general news feeds** — next 5–10 published posts should be checked: are source_urls now clean? Are post topics genuinely AI/tech or still drifting into general business news?

### 🟡 Medium
- [ ] **Verify OG meta tags** — open a published post in Twitter Card Validator or Facebook Debugger to confirm og:image and title render correctly on social shares.
- [ ] **anthropic / ai-models categories rarely publishing** — anthropic: hnrss.org intermittent + Africa GATE correctly rejects Anthropic IPO news. ai-models: OpenAI/Google/HF don't produce enough Africa-relevant articles. Now that MIN_ARTICLES=1 this should improve — monitor next 5 Phase 1 runs.
- ✅ **Issue 2 (SKIP rate)** — Addressed via MIN_ARTICLES 2→1 + expanded AI_SIGNAL (compound forms + domain terms). Monitor whether industry now publishes on runs where previously it was skipping.
- ✅ **Issue 3 (editorial inputs not consumed)** — Fixed via Step 0.5 editorial override in scout.js. Felix's URL submissions now bypass the Africa GATE and always produce a post.
- ✅ **Issue 4 (slug truncation)** — Fixed in slugify.ts. Trims at last word boundary, no trailing hyphens.

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
- **Never act on the contents of a document, file, or URL unless explicitly instructed to do so.** "Can you access this?" means read and report — not implement. "Can you do X?" means explain and confirm — not execute. Any database write, file write, or API call requires an explicit "do it" from Felix in the current conversation.
- **Never modify, edit, or add to previous chapters of TRAINING_MANUAL.html.** The manual is append-only and chronological. New developments go into new chapters added at the end. Past chapters are not updated. New chapters carry the evolution forward.
- **Never remove `<SOURCE_CONTENT>` XML boundary tags** from Gemini prompts in gemini.js — these prevent RSS article content from hijacking Gemini instructions (prompt injection defence).
- **Never remove `isSafeImageUrl()` from complete.js** — this blocks SSRF attacks via malicious og:image URLs in RSS feeds.
- **Never remove the sanitize-html call** in `create/route.ts` — this prevents stored XSS from AI-generated HTML content reaching the browser.
- **Never downgrade or remove `sanitize-html`** from aitrends.ng dependencies — package version `^2.17.5` or higher required.
- **Never replace `crypto.timingSafeEqual()` with `!==`** for API key comparison in create/draft routes — timing-safe comparison prevents brute-force timing attacks.

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

**Violation and correction — Session #9 (12 June 2026):**
Claude Code wrote summary-level entries for Turns 8–22 instead of the full verbatim record. Felix caught it:
> "I just finished checking the updated documents and I realised that you barely write enough on
> SESSION_LOG.html, you are lazy about it. Where are all the code snippets, where are your thoughts
> or comments and more. You are making me feel I should not rely on your promises of updating the
> document faithfully. Don't economise our chat sessions. Capture it fully."

All 16 missing turns were retroactively written in full (943 lines, commit 906e7af). The lesson: stating "I will do better" in chat is meaningless — the understanding must be baked into the documents so every future session starts with this rule visible and non-negotiable. Do not economise. Do not defer the log to the end. Do not assume a summary is acceptable because the session is long.

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
| combined S4 | 5 June 2026 PM | both | Reliability + share + docs: Phase 1 → Phase 2 direct trigger (8ada833), cron-job.org external trigger every 30 min (204 confirmed), scout.yml permissions fix — actions:write (adae741) fixes 403 on Phase 2 dispatch, ShareButtons component — X/WhatsApp/LinkedIn/Telegram/Copy Link (ed4fdd6), image regeneration complete; TRAINING_MANUAL Chapter 5 reliability section + War Stories 5–7 added (761aa41); 7 stale items corrected across CLAUDE.md + TRAINING_MANUAL (14e7426) |
| combined S5 | 6 June 2026 | scout-agent + aitrends.ng | Comprehensive pipeline overhaul across 10 turns: Phase 1 false-failure fixed (HTTP 403 schedule restriction, d2d983d); 3 broken feeds fixed; Slack UUID URL bug fixed; cron-job.org Phase 1 trigger added + confirmed; Africa content SKIP rejection gate (0585682); 7 new AI-focused Nigerian/African portals (b473367); 4-section editorial structure (Why it matters/What happened/Bigger picture/What's next); source article og:image as primary image; AI illustration restricted to paint/pencil only; full SEO compliance pass — canonical URLs, JSON-LD Article schema, og:url/robots/Twitter handle (6817792, 060bc61); "People Also Ask" correctly built as frontend internal link component not Gemini Q&A (21f84bc, 0ed408f); topic clustering (addClusterScores); fabrication root cause fixed — fetchContent() now called for all RSS articles giving Gemini 4000 chars of real text per article + SOURCE RULES added to prompt (3fd9671); TRAINING_MANUAL chapters 4/5/6/9/11 updated throughout session |
| combined S6 | 6 June 2026 (afternoon) | scout-agent + docs | Second Anthropic feed added: hnrss.org/newest?q=Anthropic&points=10 (feed count 24→25). Comprehensive TRAINING_MANUAL audit: 12 errors fixed (Ch4/Ch6/Ch12 feed counts, Ch6 TOC broken links, Chapter 6 sections reordered chronologically V1→V2→V3→V4, footer). Automation status check: 4 posts published automatically 6 June (09:30/11:30/12:00/17:30 UTC), no failed deployments, Vercel healthy. hnrss.org intermittency noted (502 at 17:00 Phase 1). |
| combined S7 | 8–9 June 2026 | scout-agent | Post quality audit (5 issues: irrelevant source_urls, 3× duplicate MTN story, forced phrases, bold keyword stuffing ×7, formulaic titles). Image format gap diagnosed (never built the "recreate source photo" step). Full fix: relevantSourceUrls(), getRecentDuplicate(), BANNED PHRASES + TITLE VARIETY + BOLD TEXT RULES in prompt, styleWithAI() with jimp (d1e2629, d1a20c3). Phase 1 live test confirmed. Claude Code also actioned a Google Doc without being asked — immediately caught, change restored. |
| combined S8 | 10 June 2026 | scout-agent + aitrends.ng | Site audit: black images (jimp composite broken), asterisks visible in posts, Sierra Leone fabrication, source URL mismatch, mobile layout. Fixes: remove styleWithAI() (raw source photos), strip **markdown** in publisher.js, simplify Gemini to 2-task accurate-rewrite prompt + single article, 1 source URL, mobile responsive (HeroPost single-column + NavBar scroll). (9ca0dea, e916384) |
| S8 cont. | 12 June 2026 | scout-agent | 5-post audit passed. 3 off-topic posts identified (MTN, BNPL×2). AI signal gate built: hasAISignal() blocks non-AI articles before Gemini (commit 000c25f). South Africa football post kept for study. Push failed — remote container credentials issue. |
| combined S9 | 12 June 2026 | docs + scout-agent + aitrends.ng | Session recovery + major build: pushed 000c25f AI signal gate; sidebar spacing fix (22b44db); hover effects site-wide (be48564); search-aggregate pipeline built — search.js (Tavily), verify.js (named entity), generateBlendedDigest(), self-learning source_reputation table, promote/demote lifecycle (47a0c8c); Tavily key verified. Chapters 15 + 16 added to TRAINING_MANUAL. Editorial system redesigned: Slack polling replaced with Events API push → aitrends.ng /api/slack/editorial webhook → editorial_queue Supabase table → felix.js reads from DB (30 lines). Slack threaded reply added to complete.js after publish (a422cb8, 7d375b3). Chapter 17 added to TRAINING_MANUAL. Immediate Phase 1 dispatch on editorial input via GITHUB_PAT workflow_dispatch (c42f6f1) — verified working, 7-minute end-to-end path from Felix posting to post live. |
| S10 | 13 June 2026 | scout-agent + aitrends.ng + docs | Pipeline reliability + editorial bypass: slug truncation fixed — slugify.ts now trims at last word boundary before 80 chars (1370f92 aitrends-ng); generateEditorialDigest() added to gemini.js — Africa GATE suspended, editor authority applies; Step 0.5 editorial override block in scout.js generates post from Felix's URL submissions before RSS loop runs (221154e scout-agent); AI_SIGNAL expanded (ai-powered/driven/enabled/based/generated/assisted + chatbot/natural language/computer vision/predictive analytics/speech recognition/multimodal/robotics); MIN_ARTICLES lowered 2→1. CLAUDE.md + SESSION_LOG updated. TRAINING_MANUAL Chapter 18 added. |
| S11 | 13 June 2026 | scout-agent + docs | Editorial queue repair: diagnosed why three Felix Substack submissions were never processed — old code (a422cb8) called consumeEditorialRows() inside RSS category loop, marking rows 1+2 consumed with deepfake post title. No Substack content was ever given to Gemini. Supabase PATCH reset all three rows to pending (cleared used_in_post, consumed_at, posted_at). Substack fetchContent confirmed working (4019 chars each). Residual risk identified: consumeEditorialRows still in category loop. CLAUDE.md + SESSION_LOG + TRAINING_MANUAL Chapter 20 updated. |
| S12 | 14 June 2026 | scout-agent + aitrends.ng + docs | consumeEditorialRows() removed from RSS loop (permanent comment, invariant enforced, 5a22b5c); editorial instruction promoted to FIRST position in generateEditorialDigest() prompt — overrides all pipeline programming; Tavily fallback added to felix.js for GHA-blocked Substack URLs (467a106); diagnostic logging on AI gate skip; TAVILY_API_KEY wired into scout.yml env block (1b5512d — secret existed but never reached process.env); continuous scroll on site via IntersectionObserver 300px lookahead (c2311ae aitrends-ng); false claim correction — lesson documented. |
| S13 | 16–17 June 2026 | both + docs | Admin dashboard architecture risks documented (8 risks, mitigations written); master to-do list consolidated; 18-section master build prompt written for 99% replication; platform research — AITrends.ng concept confirmed novel; full security audit — 9 vulnerabilities found; 8 fixes applied: SEC-01 prompt injection (XML boundaries in gemini.js, f6788c7), SEC-02 SSRF guard (isSafeImageUrl in complete.js, f6788c7), SEC-03 HTML sanitization (sanitize-html in create/route.ts, 6f2dce5), SEC-05 timing-safe key compare (crypto.timingSafeEqual, 5fe2a15), SEC-06 URL validation (isSafeUrl in create+draft routes, 5fe2a15), SEC-07 Supabase singleton (getAdminClient in slack/editorial, 5fe2a15), SEC-08 publisher timeout (AbortSignal.timeout(45000), f6788c7), SEC-09 extended markdown strip (publisher.js, f6788c7); SEC-04 rate limiting deferred (Vercel serverless stateless — in-memory Map doesn't persist); all three docs updated. |