# AITrends Project — Master Specification
**Last Updated:** 2026-07-13 (Session #28 — Cooldown timezone bug fixed: UTC-date comparison replaced with 20-hour elapsed-time check, restoring daily self-reset; Session #27 cooldown system built; Session #26 site search shipped + podcast prompt fixed; Session #25 Africa-first mandate removed entirely)
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

> **Reversal note (Session #25, 25 June 2026):** AITrends.ng was Africa-first from launch through
> Session #24. Felix dropped that mandate entirely — see Section 7 Critical and Session #25 in the
> history table for the full reasoning (the mandate itself was the root cause of a recurring
> fabrication problem: when a story had no real Africa angle, Gemini still tried to satisfy the
> gate by manufacturing one). The mission below reflects the **current, general-AI** direction.
> Historical session entries describing the old Africa-first mandate are left as-is per this file's
> own append-only convention — they document what was true at the time, not what's true now.

AITrends.ng is a **fully autonomous AI news and blog platform** that captures the latest news,
events, happenings, innovations, and trends across Artificial Intelligence — models, tools,
companies, and the industry shaping what's next. No daily human intervention. The system runs
itself automatically.

> **Content stance:**
> Every post is grounded in the actual facts of its source — no invented angles, no manufactured
> relevance, no forced framing. Coverage spans model releases, AI companies, developer tools, and
> the broader industry, written with a clear point of view rather than a flat summary.

**Brand tagline:** "Your autonomous briefing on AI — news, trends, and what it means for builders."
**Footer line:** "Today and the next AI trends"
**Powered by:** FAIT (small footer attribution — "by FAIT")
**Voice:** Punchy, opinionated, like a smart senior tech analyst briefing you.
**Audience:** Developers, founders, and AI practitioners who want signal without the noise.

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
    7. Generates 800-word digest via Gemini 3.5 Flash, grounded strictly in the source (ONE STORY ONLY)
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
- Story-specific: images must depict the specific story — not a generic stock-photo tech scene
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

### Table: `newsletter_subscribers`
| Column | Type | Notes |
|---|---|---|
| id | bigint | primary key, identity |
| email | text | unique, not null |
| subscribed_at | timestamptz | auto |
| status | text | `active` · `unsubscribed` |

Written by `/api/subscribe` (upsert on conflict email). View in Supabase Table Editor to see all subscribers.

### Table: `feedback`
| Column | Type | Notes |
|---|---|---|
| id | bigint | primary key, identity |
| name | text | nullable (optional field) |
| email | text | nullable (optional field, validated if provided) |
| message | text | required, max 2000 chars, HTML stripped before storage |
| post_slug | text | nullable — slug of the post the feedback was submitted on |
| created_at | timestamptz | auto |

Written by `/api/feedback`. View in Supabase Table Editor to read visitor comments.

**PostgREST schema cache note:** After creating new tables in Supabase, the PostgREST API layer must reload its schema cache before the tables are accessible via the Supabase client. Run `NOTIFY pgrst, 'reload schema';` in the SQL Editor immediately after creating new tables, or wait ~30 seconds for auto-reload.

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
- **Engagement features (Session #18):**
  - Vercel Analytics — `@vercel/analytics` installed, `<Analytics />` in `layout.tsx`. Page views, unique visitors, top pages — no cookies required
  - `/api/subscribe` — POST, upserts email to `newsletter_subscribers` Supabase table (email validated, timing-safe not needed — public endpoint)
  - `/api/feedback` — POST, inserts name/email/message/post_slug to `feedback` Supabase table (HTML stripped, email validated, 2000 char cap)
  - `NewsletterSignup` component — compact email subscribe form in sidebar on every page
  - `FeedbackForm` component — name/email/message form at the bottom of every post page (below People Also Ask)
- **Security hardened (Session #13):**
  - `sanitize-html` on all AI-generated content before Supabase write (allowlist: h2–h4, p, strong, em, lists, a, blockquote, hr)
  - `crypto.timingSafeEqual()` on API key comparison in `create` and `draft` routes
  - `isSafeUrl()` validation on `cover_image_url` and `source_urls` before storage
  - `getAdminClient()` Supabase singleton in `slack/editorial/route.ts`
- **SEO hardened (Session #14):**
  - `og-default.png` fallback OG image created (1200×630, brand colours) and deployed
  - `unoptimized` removed from post page cover `<Image>` — Next.js now serves WebP automatically for Supabase Storage URLs
  - `cover_image_prompt` used as alt text on all cover images (PostCard, HeroPost, post page) — describes visual content rather than repeating the headline
  - `rel="noopener noreferrer nofollow"` on all external source_urls — prevents PageRank leakage to attribution links
  - About page corrected: Step 01 lists accurate current feeds; Step 03 now says HF FLUX.1-schnell → Supabase Storage (not Pollinations)
  - aitrends.ng root domain live via Whogohost A record → 76.76.21.21

### Known Issues
- 🟡 www.aitrends.ng — Whogohost CNAME for www points to `aitrends.ng` root instead of `cname.vercel-dns.com`; www also not added to Vercel domain list. Fix: change Whogohost CNAME value to `cname.vercel-dns.com`, then add `www.aitrends.ng` to Vercel domains.

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
- **`GEMINI_API_KEY` is tied to a fresh project (free tier)** — old project hit Tier 1 billing after28 days; new key has fresh free-tier quota. Rotate before expiry or create another new project
- **`GROQ_API_KEY` provides cross-provider fallback** — Groq Llama 3.3 70B (14,400 req/day free) is the fallback when Gemini returns non-retryable errors. Different infrastructure, won't go down at the same time

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
7. Generate an accurate digest via Gemini 3.5 Flash
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
  gemini.js      ← Gemini API integration. General-AI editorial prompt. 3-attempt retry.
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

**⚠️ Discrepancy found Session #19 — this list is incomplete.** These 4 scopes only cover *reading* #scout-editor. The bot token also calls `chat.postMessage` (`replyToFelix()` in slack.js, used since Session #9, plus the new `replyFailureToFelix()` added Session #19) — that requires `chat:write` (Slack reports the requirement as `chat:write:bot`), which was never granted. Confirmed live via the response headers on a real `chat.postMessage` call: `x-oauth-scopes: channels:history,channels:read,groups:history,groups:read` — no `chat:write`. **Practical effect: every threaded Slack reply this pipeline has ever attempted to send has been silently failing since Session #9.** Not fixable from code — see Section 7 Critical to-do.

### Gemini Prompt — Editorial Mandate (rewritten Session #25 — Africa-first mandate removed)
The editorial prompt in `gemini.js` enforces:
- Accurate rewrite/synthesis grounded only in the source's own facts — no invented angles, no manufactured relevance
- Closing sentence varies in construction post to post (direct implication, forward-looking note, measured/skeptical beat, or pointed question) — explicitly forbidden from using a fixed lead-in label like "Bottom line:" or "Takeaway:", to avoid the same robotic-uniformity problem the old fixed "Bottom line for African builders:" template caused
- `TRENDING_TERMS` restricted to proper nouns only (company names, model names, product names) — no phrases, no verbs, no generic words
- No geographic relevance gate of any kind — see Section 7 Critical for why this was removed

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
- **YouTube podcast pipeline** (Session #16, commit e0f5d0b) — `scout-podcast.js`, `feeds-podcast.js`, `transcript.js`, `gemini-podcast.js`; 6 channels; SCOUT_MODE=podcast routes from scout.js; `scout-podcast.yml` GHA workflow (daily 06:00 UTC)
- **Gemini daily-quota cooldown** (Session #27, timezone fix Session #28) — `callGeminiWithFallback()` in gemini.js (shared by generateDigest/generateBlendedDigest/generateEditorialDigest/gemini-podcast.js) falls back to gemini-2.5-flash on 429 as well as 503; `gemini_quota_cooldown` Supabase table (single-row, same pattern as `homepage_monitor`) records when both models 429 in the same call; scout.js and scout-podcast.js check it before any RSS/Tavily/Gemini work and skip the entire run if cooldown was set less than 20 hours ago, plus `break` out of remaining categories/channels immediately if exhaustion is hit mid-run; once-per-day Slack alert via `notifyQuotaExhausted()` in slack.js. Cooldown uses elapsed-time staleness (20h threshold) rather than UTC calendar-date comparison — the old UTC-date approach caused the cooldown to re-set itself before midnight PT, permanently blocking every run for an extra day (Session #28 root cause)

### YouTube Podcast Pipeline — Key Details
- **SCOUT_MODE=podcast** routes `scout.js` → `runPodcastScout()` in `scout-podcast.js`
- **6 channels:** Ethan Mollick (UCuEvINOL2TLekZeeCk50Yug — inactive, 0 videos currently), Udacity, Sandeep Swadia | theMITmonk (UC4ZVkG3RQPzvZk7alIVjcCg — corrected ID), The Diary Of A CEO, Silicon Valley Girl, Mo Gawdat
- **Duration gate:** 8 minutes — checks YouTube page JSON-LD (`"duration":"PT..."`); transcript length proxy (6,000 chars) as fallback
- **AI signal gate:** Filters non-AI episodes from mixed channels (Diary of CEO, Udacity, Mo Gawdat philosophy content)
- **Transcript:** 20,000 chars via `youtube-transcript` → Tavily extract fallback → RSS description fallback
- **Thumbnail:** Downloads `maxresdefault.jpg` / `hqdefault.jpg` → base64 → passed to Gemini multimodal
- **Gemini multimodal:** Thumbnail + transcript → generates 6-section post + HF FLUX image prompt
- **6-section structure:** Overview, Key Takeaways, `<div data-youtube=""></div>` embed, Main Ideas (3–6), Notable Quotes, Practical Applications, Final Thoughts, Source attribution (plain text only)
- **YouTube embed:** `<div data-youtube=""></div>` placeholder → replaced at render time by `injectYouTubeEmbed()` in `post/[slug]/page.tsx` using `source_urls[0]` video ID → `youtube-nocookie.com` iframe
- **Cover image:** HF FLUX generates unique editorial illustration from Gemini's prompt (NOT raw thumbnail)
- **Category:** hardcoded `ai` — all podcast posts go here
- **1 post per channel per run** — prevents flooding
- **No Africa gate** — global content
- **complete.js change:** `isYouTubeSource` check skips `trySourceImage` for YouTube posts → HF FLUX path used

### ⚠️ Supabase Migration Required (blocks podcast publishing)
Run this SQL in Supabase console before the first podcast Phase 2 run:
```sql
ALTER TABLE posts DROP CONSTRAINT IF EXISTS posts_category_check;
ALTER TABLE posts ADD CONSTRAINT posts_category_check
  CHECK (category IN ('ai-models','anthropic','industry','tools','ai'));
```

### ⚠️ cron-job.org trigger needed for podcast pipeline
Add a new cron-job.org job:
- URL: `https://api.github.com/repos/FAIT-Blog/scout-agent/actions/workflows/scout-podcast.yml/dispatches`
- Method: POST
- Headers: `Authorization: Bearer {PAT}`, `Content-Type: application/json`
- Body: `{"ref":"main"}`
- Schedule: Once daily (e.g. 07:00 UTC) — or use GHA native cron (scout-podcast.yml already has `0 6 * * *`)

---

## 7. MASTER TO-DO LIST
*Updated: 2026-06-22 (Session #19 — re-verified against live state; see note below)*
*Status key: 🔴 Critical | 🟠 High | 🟡 Medium | 🔵 Low / Later*

**Note (Session #19):** The "Updated" date above had been stale since 19 June — Sessions #16, #17,
and #18 added completed-work entries below without bumping it. More importantly, two items that had
been sitting in this list as outstanding ("🟠 High") were checked directly against the live database
and Vercel-served site during Session #19 and turned out to already be resolved — they just hadn't
been confirmed and removed. Both are marked ✅ below with the verification evidence, rather than
silently deleted, per Felix's instruction to point out discrepancies with comments and reasons.

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

### ✅ Completed — Session #14 (17 June 2026)
- ✅ **Site evaluation completed** — Honest audit of aitrends.ng: og-default.png 404'd, AI signal gate too broad (general business articles passing), Africa GATE fabricating connections via banned patterns, About page had stale Pollinations reference, DNS not configured.
- ✅ **og-default.png created** — 1200×630 PNG generated with Python PIL using brand colours (#0a0a0f background, #2563eb blue, #f59e0b gold). Committed to `aitrends.ng/public/og-default.png` (ae87f97). HTTP 200 verified on live domain.
- ✅ **AI signal gate tightened** — Moved `algorithm`, `automation`, `autonomous`, `digital innovation`, `technology policy`, `tech startup` to WEAK list (require title presence OR 3+ body occurrences). These broad terms were allowing general business restructuring articles to pass. Committed 8301fc3.
- ✅ **Africa GATE hardened** — Added explicit substitution test and forbidden patterns to both `generateDigest()` and `generateBlendedDigest()` in gemini.js: "Could you substitute 'European developers' and the sentence would be equally true? → SKIP." Conditional "What this means for Africa" sentence — only written when source explicitly names an African entity. Committed 8301fc3.
- ✅ **aitrends.ng DNS configured** — A record `@` → `76.76.21.21` set on Whogohost. Root domain HTTP 200 confirmed. Custom domain active on Vercel.
- ✅ **SEO improvements** — Post page cover image `<Image>`: removed `unoptimized` (WebP now served automatically); `cover_image_prompt` used as alt text on all cover images (PostCard, HeroPost, post page); `rel="nofollow"` added to external source_urls; About page updated with accurate feed list and image pipeline description. Build passed cleanly. Committed 5856ea6.
- ⚠️ **www.aitrends.ng broken** — Whogohost CNAME for www points to `aitrends.ng` (root) instead of `cname.vercel-dns.com`; www also not added to Vercel domain list. Manual fix required (Whogohost + Vercel dashboard).

### ✅ Completed — Session #15 (18 June 2026)
- ✅ **Document push strategy locked** — Felix's explicit instruction: SESSION_LOG.html and TRAINING_MANUAL.html are committed locally only, never pushed to GitHub. CLAUDE.md is pushed to GitHub at the end of every session. Documented in Section 9 of CLAUDE.md. Files are 508KB and 257KB — well within GitHub's 100MB limit; decision is strategic, not technical.
- ✅ **HF FLUX.1-schnell confirmed as active image provider** — `image-providers/index.js` default: `process.env.IMAGE_PROVIDER || 'huggingface'`. HuggingFace provider uses FLUX.1-schnell via HF Inference API. Active since Session #2.
- ✅ **www.aitrends.ng verified still broken** — curl confirms no response from www subdomain. Whogohost CNAME for www still points to root domain, not `cname.vercel-dns.com`. www not in Vercel domain list.
- ✅ **updated_at added to Post type and post page** — `updated_at?: string | null` added to `lib/types.ts`; `wasUpdated()` helper shows "Updated [date]" in post meta row when `updated_at` > `published_at` by 1+ day; JSON-LD `dateModified` now uses `updated_at` when available. Build clean. Committed 71550cf (aitrends-ng pushed). **Supabase migration still needed** — the column doesn't exist in the DB yet. Field will always be undefined until migration is run.
- ✅ **S13 backlog explained** — 2 local commits (S13 `439431f`, S14 `e0bbcb0`) not yet on GitHub because container has 403 on aitrends-project repo. Felix must run `git push origin main` from `aitrends-project/` directory to push them. Going forward: split commits (CLAUDE.md → push to GitHub; SESSION_LOG+TRAINING_MANUAL → local only).
- ✅ **www.aitrends.ng live — confirmed via curl** — HTTP/2 200. Vercel screenshot shows: `www.aitrends.ng` Production (✅), `aitrends.ng` 308 → www.aitrends.ng (✅), `aitrends-ng.vercel.app` Production (✅). Correct setup.
- ✅ **Canonical domain updated to www across all URLs** — All hardcoded `https://aitrends.ng` updated to `https://www.aitrends.ng` in: layout.tsx, sitemap.ts, robots.ts, feed.xml/route.ts, post/[slug]/page.tsx, api/posts/create/route.ts. Committed 9463584 and pushed. ~~**One manual step:** update `NEXT_PUBLIC_SITE_URL` in Vercel dashboard~~ — **Found moot, Session #19:** `api/posts/create/route.ts` reads this var but falls back to the identical hardcoded value (`process.env.NEXT_PUBLIC_SITE_URL ?? 'https://www.aitrends.ng'`); every other consumer hardcodes the URL directly and doesn't read the env var at all. Whether the Vercel dashboard value was ever updated has no observable effect. No action needed.

### ✅ Completed — Session #16 (19 June 2026)
- ✅ **YouTube podcast pipeline built and tested** — 4 new scout-agent files: `feeds-podcast.js`, `transcript.js`, `gemini-podcast.js`, `scout-podcast.js`. SCOUT_MODE=podcast routing in scout.js. complete.js YouTube source detection. scout-podcast.yml GHA workflow. All syntax checks pass. Commits e0f5d0b (scout-agent), cdf7c6e + 33bffe6 (aitrends-ng).
- ✅ **theMITmonk channel ID corrected** — Old ID `UC-5yNbJKcYrRbhocskA-6qA` returned 0 items (Sandeep Swadia's old empty channel). Correct ID `UC4ZVkG3RQPzvZk7alIVjcCg` returns 15 videos. Verified via video `IWdvG9Up8Mc` oEmbed.
- ✅ **submitImageJob() bug fixed** — Was storing `{ jobId: null }` object as the jobId. Fixed to destructure `{ jobId }` from the result (consistent with scout.js pattern).
- ✅ **aitrends.ng 'ai' category** — Added to VALID_CATEGORIES in create/route.ts (with sanitize-html div allowance), lib/createPost.ts, sitemap.ts, about page (Step 06), post page (YouTube embed injection). schema.sql CHECK constraint updated in code.
- ✅ **Local test passed** — 3 posts queued: Udacity MSc in AI, Silicon Valley Girl (AI-First Playbook, guest Peter Yang), Mo Gawdat (AI skills). HF FLUX image generation confirmed for 2 of 3. AI signal gate correctly blocked JD Vance, creatine, pyramids, parenting posts from Diary of CEO.
- ✅ **Supabase migration run** — category CHECK constraint updated to include 'ai'. Posts table now accepts podcast posts.
- ✅ **cron-job.org trigger added** — "Scout Podcast Phase 1" job added. Daily trigger for scout-podcast.yml workflow_dispatch.
- ⚠️ **Posts 108, 109, 110** — Were stuck in failed status (Phase 2 retried before migration ran). Reset via SQL UPDATE to pending_image. Should publish on next Phase 2 cycle.
- ⚠️ **Ethan Mollick's channel inactive** — `UCuEvINOL2TLekZeeCk50Yug` returns 0 items. He uploads rarely. Scout skips silently. Update channel ID when Felix provides an active video URL from his channel.

### 🔴 Critical
- ✅ **Africa-first editorial mandate removed entirely (Session #25, 25 June 2026).** Felix's decision, with his own root-cause diagnosis: the Africa relevance gate was itself causing the recurring forced-phrase/fabrication problem (Sessions #7, #8, #14, #23) — when a story had no real Africa angle, Gemini still tried to satisfy the gate by manufacturing one (forced "Lagos, Accra, Nairobi" mentions, bolded phrases to manufacture an impression of relevance). A narrower per-phrase ban doesn't fix this; it just pushes Gemini toward a different fabrication. AITrends.ng is now a general-AI news site. Changes: removed the full "AFRICA RELEVANCE GATE" block (checklist, `SKIP_NO_AFRICA_RELEVANCE` sentinel, the Session #23 ai-models extra-strictness clause — now moot) from both `generateDigest()` and `generateBlendedDigest()` in gemini.js; replaced the old conditional "What this means for Africa" closing sentence with a varied-construction instruction (implication / forward-looking note / skeptical beat / pointed question, explicitly banning fixed labels like "Bottom line:" — mirrors the existing Session #7 "TITLE VARIETY" fix, same mechanism applied to the closing sentence); removed the now-dead sentinel check in scout.js. `generateEditorialDigest()` needed zero changes — it never had an Africa gate by design. Explicitly **not** changed: `feeds.js` (still has the 14 Africa-specific outlets, Felix's call — "keep all current feeds as-is"), `EVERGREEN_ENTITIES`'s Africa-related scoring-boost terms (Felix's call — "leave them in, harmless either way"), `scout-podcast.js`/`gemini-podcast.js` (already fully general). Verified live with one real `runScout()` run (not a batch, given the quota cap): 2 posts queued — one genuinely Africa-relevant (Nigeria's NITDA digital sovereignty strategy, scored on its own merits, not forced) and one purely global with zero Africa framing (Anthropic vs. Alibaba Claude-distillation dispute) — both closing sentences varied in construction, neither used a fixed label, no sentinel leakage. Branding/copy updated to match across CLAUDE.md (this section, tagline, voice, audience in Section 1) and aitrends.ng (layout.tsx titles, about page, Sidebar, NewsletterSignup, feed.xml) — `npm run build` clean afterward. **This is forward-looking only — the 121 existing published posts remain untouched**, per Felix's explicit instruction ("All current live posts are to remain whether relevant or not").
- ✅ **Add second Anthropic feed source** — `hnrss.org/newest?q=Anthropic&points=10` added to feeds.js (Session #6). anthropic category now has 2 feeds; MIN_ARTICLES=2 satisfied. Feed count: 24 → 25.
- ✅ **Assess Gemini capability after simplification** — Audit passed (Session #8 continuation, 12 June): 5 posts reviewed. No asterisks, 1 source URL, title variety confirmed. Gemini accurate-rewrite prompt working as intended.
- ✅ **AI signal gate pushed** — 000c25f pushed to FAIT-Blog/scout-agent in Session #9.
- ✅ **Editorial system redesigned** — Slack polling → Events API push → Supabase queue → Slack threaded reply. (a422cb8 scout-agent, 7d375b3 aitrends-ng). Pending one-time setup: run add-editorial-queue.sql, add SLACK_SIGNING_SECRET to Vercel, enable Events API in Slack app pointing at https://aitrends-ng.vercel.app/api/slack/editorial.
- ✅ **One-time editorial setup complete** — SQL run, Events API enabled, SLACK_SIGNING_SECRET + GITHUB_PAT added to Vercel, cron-job.org PAT updated, Vercel redeployed. Verified: Phase 1 dispatched immediately on editorial input (21:29 UTC, run 27444126665). Editorial rows stay pending until a category queues successfully (expected — Africa GATE correctly skipped that run).
- ✅ **`consumeEditorialRows()` removed from RSS category loop** — Done in Session #12 (commit 5a22b5c). Replaced with permanent comment explaining the invariant. The RSS loop will NEVER touch editorial_queue rows.
- ✅ **Security audit + 8 fixes applied** — Session #13. SEC-01 prompt injection, SEC-02 SSRF, SEC-03 HTML sanitization, SEC-05 timing-safe key compare, SEC-06 URL validation, SEC-07 Supabase singleton, SEC-08 publisher timeout, SEC-09 extended markdown strip. Three commits across both repos.
- ✅ **HN Anthropic feed intermittency fixed (Session #23)** — `hnrss.org/newest?q=Anthropic&points=10` (feeds.js) replaced with Bing News RSS (`bing.com/news/search?q=Anthropic&format=rss`) — confirmed 200 vs hnrss's live 502 at time of testing. Bing wraps links in a click-tracking redirect (`bing.com/news/apiclick.aspx?...&url=...`); added `resolveTrackingLink()` to scout.js (lightweight `redirect: 'manual'` fetch reading the `Location` header, 10s timeout) so the real publisher URL is stored as `source_url`, not the tracking link. Verified end-to-end: 11 fresh items, links resolved to MSN/Yahoo/TheStreet.
- ⚠️ **Old posts retroactive cleanup — IN PROGRESS (Session #23)**, not yet complete:
  - ✅ Markdown asterisks: all 54 affected posts (of 67 pre-Session-8) stripped using the exact same regex chain `publisher.js` already applies to new posts. Verified 0 of 67 posts contain `**bold**` markers afterward. One post (`6adae9f6`) has an unrelated leftover single `*` from an old markdown bullet-list line, not bold syntax — left untouched, flagged as a separate minor cosmetic issue (1 post only).
  - ✅ Irrelevant source_urls: audited all 287 source_urls across 67 posts by fetching each page's title and checking for AI-signal keywords (reusing the same vocabulary `hasAISignal()` uses) rather than comparing to the post's own often-metaphorical title. 87 had a confirmed off-topic title (sports/crime/politics/entertainment/job-listings/directory-pages/unrelated business news — e.g. "Panic as Eriksen collapses", "MOVIE REVIEW: Blood Sisters 2", Binance trading-competition pages). Manually cross-checked against false negatives (cases where the flagged URL was actually the correct primary source, just missed by the keyword check — e.g. "AI-Native" wasn't in the hyphenated-AI list) before removing anything. Removed 68 confirmed-irrelevant URLs across 35 posts; kept 19 genuine sources my keyword check would have wrongly flagged. 32 URLs were unverifiable (fetch errors) and left untouched rather than guessed at. Two posts ended up with `source_urls: []` after removal (all their sources were junk) — consistent with the existing `relevantSourceUrls()` precedent of dropping bad attribution rather than keeping it.
  - ⚠️ Keyword stuffing: detected by scanning for any 4–7 word phrase repeated verbatim 3+ times in a post body (the literal pattern behind the documented "**national AI strategy Africa** ×7" issue, now visible in plain text since the asterisk strip). 36 of 67 posts had a repeated phrase; 25 had a *non-generic* phrase repeated 5+ times and were sent to Gemini for a rewrite (same facts/length/HTML structure, phrase reduced to ≤2 natural mentions). **15 of 25 completed and written** (spot-checked for fidelity before writing — e.g. confirmed a pre-existing Session #7-banned phrase ["For developers in Lagos, Accra, and Nairobi"] in one post was correctly left untouched rather than scope-creeping into an unrelated fix). **10 of 25 still outstanding** — blocked mid-session by hitting Google's Gemini free-tier quota (`GenerateRequestsPerDayPerProjectPerModel-FreeTier`, hard cap of 20 requests/day for `gemini-3.5-flash`). Nothing was written for these 10 — no risk to existing content, the rewrite attempts simply never returned a response. **Open question for Felix:** if this `.env` `GEMINI_API_KEY` is the same key configured in GitHub Actions, today's remaining scheduled Phase 1/podcast runs may also 429 until the quota resets (likely midnight Pacific). Worth checking whether production uses a separate key, or whether the account needs a billing/quota upgrade given normal operation alone (4 Phase 1 runs/day + podcast) may already approach this kind of cap on a free tier.
- ✅ **Google Doc corrections — applied (Session #24, 24 June 2026)**, with two items still quota-blocked. The doc (`docs.google.com/document/d/1nRyGxPExGCFP62CPGv98kj2JZwiGZBDoyji85ZEsCqw`) listed 3 specific corrections: (1) the bolded `**national AI strategy Africa**` stuffing — already resolved by Session #23's asterisk-strip + de-stuffing pass (verified: no bold markers, phrase count reduced 7→2). (2) 3 named-irrelevant source_urls on the Zimbabwe/SA post — the Ruto opinion-piece link was already removed in Session #23's cleanup; the **2 businessday.ng links were not** (false negative in that session's AI-signal keyword check — both titles contained the word "data", which was on the signal list, even though neither is actually about this post's AI-strategy topic). Removed both directly per Felix's explicit naming of the exact URLs. (3) The forced template phrase "For developers in Lagos, Accra, and Nairobi..." (banned going forward since Session #7, but never retroactively fixed) — found in 6 of 121 published posts (not just the one example post Felix linked). Sent all 6 to Gemini for a fact-preserving rewrite of just the affected opening; 4 of 6 completed, spot-checked, and written. **2 of 6 still pending — including Felix's own example post ("The Truth Behind MTN Nigeria Data Consumption...")** — blocked by the same Gemini free-tier daily quota cap hit in Session #23; nothing written for these 2, no content risk.
- [ ] **BusinessDay Nigeria sports feed** — South Africa football post traced to BusinessDay feed. Monitor: does hasAISignal() filter enough, or does BusinessDay need removal? Check next 3–5 industry posts.
- [ ] **Nigerian feeds are general news** — Still influencing article scoring. Monitor whether published topics drift into general business/politics now that hasAISignal() gate is live.
- ✅ **ai-models category-specific Africa GATE tightened (Session #23)** — Added an explicit extra-strictness clause to both `generateDigest()` and `generateBlendedDigest()` in gemini.js, scoped to `category === 'ai-models'`: model/product announcements from OpenAI/Google AI/HuggingFace/DeepMind are now treated as global engineering news by default, even when they claim to "support African languages" or help "underserved regions" — that capability claim alone no longer passes the gate. Requires an explicit named African deployment/partnership/government initiative in the source to count as genuine relevance. Targets the exact documented failure mode (a DeepMind voice-translation/Gemma announcement fabricated into a "Sierra Leone education" story). **Note:** this only prevents *future* fabrications — the existing live Sierra Leone post (`07869fee-10d8-4871-ae17-2f893ff28529`) itself still has the fabricated content and was deliberately left untouched during this session's source_url/keyword-stuffing cleanup (trimming its source or smoothing its prose wouldn't fix the underlying fabricated Africa connection). Needs a separate decision: unpublish (precedent: Session #3 did this for the multi-topic post) or fully rewrite.
  **⚠️ Superseded Session #25 (25 June 2026):** this entire fix is now moot — the Africa GATE it tightened was removed entirely (Felix's call: the gate itself was the root cause of the fabrication pattern, not just the ai-models category's version of it; narrowly patching it was whack-a-mole). The Sierra Leone post's unpublish-or-rewrite decision is still open and separate from this.
- ✅ **Editorial specification implemented** (b473367) — 7 new AI-focused Nigerian/African feeds; 4-section structure; source article og:image as primary; AI illustration as fallback.
- ✅ **Fabrication root cause fixed** (3fd9671) — fetchContent() now called for all RSS articles (4000 chars real text to Gemini). SOURCE RULES added to prompt.
- ✅ **Post quality assessed — Yoco/Dyner post** — non-formulaic title ✅, no forced phrases, 3 relevant source URLs. Passed.
- ✅ **"People Also Ask" correctly implemented** — (21f84bc, 0ed408f)
- ✅ **Full SEO compliance pass completed** — (6817792, 060bc61)
- ✅ **Editorial queue silent-drop bug fixed (Session #19)** — Rows where content extraction failed entirely (blocked YouTube transcript, or long pasted text with no URL) were never added to `items` or properly consumed — they stayed `pending` and silently re-failed on every Phase 1 run, in one case for 6 days, with zero feedback to Felix. Fixed: failed URL extractions now call new `failEditorialRow()` + `replyFailureToFelix()` (memory.js, slack.js) instead of vanishing. Long pasted text (>500 chars, no URL) now becomes its own dedicated post via Step 0.5 instead of a discarded topic hint. felix.js, memory.js, slack.js.
- ✅ **`chat:write` scope added to `SLACK_BOT_TOKEN` — fixed (Session #24, 24 June 2026).** Discovered Session #19, root-caused to the bot token missing `chat:write` (only had `channels:read`, `channels:history`, `groups:read`, `groups:history`) — meant every threaded Slack reply (`replyToFelix()` since Session #9, `replyFailureToFelix()` since Session #19) had been silently failing; only the separate #aitrends-feed webhook notifications were ever arriving. Felix added the scope in api.slack.com and reinstalled the app. Verified two ways: (1) a live `auth.test` call against the new local `.env` token returned `x-oauth-scopes: channels:history,channels:read,groups:history,groups:read,chat:write` and `ok: true` as bot user `scout_editor`; (2) the GitHub Actions `SLACK_BOT_TOKEN` secret was synced directly from the verified local `.env` value via `gh secret set` (not just pasted separately) to guarantee the two are byte-identical, removing the possibility of a copy-paste mismatch between environments. No further action needed — threaded replies should now actually reach Felix going forward.

### 🟠 High
- [x] ~~**Fix www.aitrends.ng**~~ — **Stale, already resolved.** This item predates Session #15, which already noted www live and removed the equivalent item there — it was left duplicated here and never pruned. Re-confirmed Session #19: `curl -s -o /dev/null -w "%{http_code}" https://www.aitrends.ng` → `200`. No action needed.
- [x] ~~**Run Supabase updated_at migration**~~ — **Stale, already resolved.** Re-verified Session #19 with a direct query: `select id, updated_at from posts limit 1` returned a populated timestamp, not null. The column and trigger already exist in the live database. This item had been carried forward unconfirmed since Session #15.
- [x] ~~**Push S13+S14 docs to GitHub**~~ — **Stale, already resolved.** Re-verified Session #19: `git log origin/main --oneline | grep -E "439431f|e0bbcb0"` returns both commits — they are on `origin/main`. `git status` reports the local branch up to date with origin. This item had been carried forward unconfirmed since Session #15.
- ✅ **Submit `sitemap.xml` to Google Search Console** — done (Session #26, 25 June 2026). Felix confirmed completed.
- [ ] **Verify post quality on new posts** — read Yoco/Dyner post body in full. Does it cite verifiable facts? No forced phrases? Varied title structure confirmed from log but body needs human review.
- [ ] **Monitor Nigerian general news feeds** — next 5–10 published posts should be checked: are source_urls now clean? Are post topics genuinely AI/tech or still drifting into general business news?
- ✅ **Tagline + footer text updated (Session #21, 23 June 2026)** — `app/layout.tsx` (description, og:description, twitter:description), `app/about/page.tsx` subtitle, and `components/Footer.tsx` all updated to `"AITrends.ng — insights, and lessons for the future."`. Two corrections made from Felix's literal wording, flagged at the time: "AItrends.ng" → corrected to the established "AITrends.ng" brand spelling; "--" → normalized to "—" matching the site's existing typography. Footer keeps its existing two-tone styled "AITrends.ng" brand element and only swaps the trailing copy (verbatim duplication would have rendered "AITrends.ng — AITrends.ng — insights...", since the new tagline text itself starts with the brand name). `components/Sidebar.tsx` was checked and does not contain this string (has separate, unrelated "About blurb" copy — left untouched, not part of this request). Typecheck + full production build passed clean; verified live afterward — meta description confirmed serving the new text. Commit `2c76eeb` (aitrends-ng).

### 🟡 Medium
- ✅ **Set up a Google Alert for "AITrends.ng" or `site:aitrends.ng` (added Session #20)** — done (Session #26, 25 June 2026). Felix confirmed completed.
- ✅ **Homepage change-detection monitor (Session #22, 23 June 2026)** — Custom-built option chosen over third-party (Felix's call — the off-topic-keyword check needed custom logic regardless). New `scout-agent/homepage-monitor.js`, independent GHA workflow (`homepage-monitor.yml`, every 6h offset from Phase 1), new `homepage_monitor` Supabase table (single-row snapshot, SQL at `aitrends.ng/supabase/add-homepage-monitor.sql`), new `notifyMonitorAlert()` in slack.js using the existing webhook (unaffected by the chat:write scope gap). Checks every run: site `<title>` / meta description changed, nav links added/removed (`nav.site-nav a`), and homepage post titles against an incident-driven off-topic keyword list (MTN, BNPL, football/FIFA — sourced from the documented Session #8 incidents, not a broad guess). Verified live: baseline saved, second run correctly reported no changes, a simulated change (patched directly in Supabase) correctly triggered a Slack alert and the snapshot self-healed back to the real state on the next run. Commits: `e9fc094` (scout-agent), `65665a8` (aitrends.ng SQL reference copy).
- [ ] **Verify OG meta tags** — open a published post in Twitter Card Validator or Facebook Debugger to confirm og:image and title render correctly on social shares.
- [x] ~~**anthropic / ai-models categories rarely publishing**~~ — **Moot, Session #25.** This was attributed to the Africa GATE rejecting otherwise-good Anthropic/ai-models articles for lacking Africa relevance. The gate no longer exists (Session #25 removed it entirely), so the constraint this item was tracking is gone. The HN Anthropic feed intermittency half of this item was separately fixed in Session #23 (Bing News RSS swap).
- ✅ **Issue 2 (SKIP rate)** — Addressed via MIN_ARTICLES 2→1 + expanded AI_SIGNAL (compound forms + domain terms). Monitor whether industry now publishes on runs where previously it was skipping.
- ✅ **Issue 3 (editorial inputs not consumed)** — Fixed via Step 0.5 editorial override in scout.js. Felix's URL submissions now bypass the Africa GATE and always produce a post. **Caveat found Session #19:** this only covered the success path. Rows where extraction *failed entirely* (no URL content, no instruction text) fell through every check silently and were never consumed — see Section 6 felix.js notes and the new Critical item below. Fixed Session #19.
- ✅ **Issue 4 (slug truncation)** — Fixed in slugify.ts. Trims at last word boundary, no trailing hyphens.

### ✅ Completed — Session #18 (21 June 2026)
- ✅ **Vercel Analytics added** — `@vercel/analytics` installed, `<Analytics />` added to `app/layout.tsx`. Tracks page views, unique visitors, top pages — no cookies required. Enabled via Vercel project dashboard → Analytics tab.
- ✅ **Newsletter subscription built** — `/api/subscribe` POST route upserts email to `newsletter_subscribers` Supabase table. `NewsletterSignup` client component added to sidebar (visible on homepage, category pages, all post pages). Email format validated. Duplicate emails handled gracefully via upsert.
- ✅ **Feedback form built** — `/api/feedback` POST route inserts to `feedback` Supabase table. `FeedbackForm` client component added below "People Also Ask" on every post page. Name and email optional, message required (2000 char cap, live counter). HTML stripped before storage.
- ✅ **PostgREST schema cache issue diagnosed and fixed** — After Felix created both tables in Supabase Table Editor, both endpoints returned 500: "Could not find the table in the schema cache". Root cause: PostgREST (the API layer sitting between Supabase client and PostgreSQL) caches the DB schema at startup. New tables are invisible until the cache reloads. Fix: `NOTIFY pgrst, 'reload schema';` in SQL Editor. Both endpoints confirmed working on live site: `{"success":true}` from both. Commit: `8796ed4` (aitrends-ng).

### ✅ Completed — Session #19 (22 June 2026)
- ✅ **Full status audit performed against live state, not against prior session notes** — checked GitHub Actions run history (zero failures, last 50 runs), live Supabase row counts and columns via direct REST queries, live curl checks against www.aitrends.ng, and a real Slack API call to inspect actual granted OAuth scopes. Four discrepancies surfaced that no prior session had caught.
- ✅ **Editorial queue silent-drop bug fixed** — see Critical section above. `failEditorialRow()` added to memory.js; `replyFailureToFelix()` added to slack.js; felix.js rewritten to call both on failed URL extraction instead of silently dropping the row, and to give long pasted text (>500 chars, no URL) its own dedicated post instead of discarding it into a one-line topic hint. All three files pass `node --check`.
- ✅ **4 already-stuck editorial_queue rows resolved** — ids 14, 15 (unrecoverable YouTube URLs, pending since the day of this session and one day prior) marked `failed` directly via the new function. Ids 7, 8 (a ~2,500–10,000 char article Felix pasted across two messages, pending since 16 June) deliberately left `pending` — confirmed via a dry-run of the fixed `fetchFelixInputs()` that they now correctly produce post-ready `items`; they will publish as a real post on the next scheduled Phase 1 run rather than being triggered manually.
- ⚠️ **Slack `chat:write` scope gap discovered — cannot be fixed from code.** See new Critical to-do above. Requires Felix to update the Slack app's OAuth scopes and reissue `SLACK_BOT_TOKEN`.
- ✅ **3 stale to-do items closed out** — "Fix www.aitrends.ng", "Run Supabase updated_at migration", and "Push S13+S14 docs to GitHub" had all been resolved in or before Session #15 but were never removed from the 🟠 High list; each was re-verified live this session and struck through with evidence rather than silently deleted. "Update NEXT_PUBLIC_SITE_URL in Vercel" (Session #15 note) found moot — code already falls back to the same value.
- ✅ **Orphaned file removed** — `command (ypco5u)` (an unexecuted heredoc script accidentally saved as a literal file in the project root by a prior session) deleted after confirming the TRAINING_MANUAL.html edits it described were already applied.
- ✅ **Docs updated chronologically per Felix's explicit instruction** — no past session entries rewritten; every correction added as a new dated note explaining what was found and why, rather than silently edited in place.

### 🔵 Later (after system is perfected)
- [ ] Connect newsletter subscribers to an email sending tool (Buttondown or Beehiiv — free tier for < 1000 subscribers)
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

### Document push strategy — updated Session #15 (18 June 2026)

Felix's explicit instruction: SESSION_LOG.html and TRAINING_MANUAL.html are committed locally but NOT pushed to GitHub until further notice. CLAUDE.md is pushed to GitHub at the end of every session.

**Why not pushed:** The decision is Felix's preference for cloud storage hygiene — not a technical constraint. SESSION_LOG.html (508KB) and TRAINING_MANUAL.html (257KB) are well within GitHub's 100MB-per-file limit. They will never hit a size problem.

**How to commit per session:**
1. First commit — `CLAUDE.md` only: `git add CLAUDE.md && git commit -m "docs: ..."` → push this to GitHub
2. Second commit — session log + manual: `git add SESSION_LOG.html TRAINING_MANUAL.html && git commit -m "docs: ..."` → local only, do NOT push

**GitHub remains the source of truth for CLAUDE.md.** SESSION_LOG.html and TRAINING_MANUAL.html are backed up on Felix's local drive. Felix should keep regular Time Machine or equivalent backups of `aitrends-project/` given this is where the session history lives.

**Current backlog:** Two local commits (S13 docs `439431f`, S14 docs `e0bbcb0`) exist on Felix's Mac but are not on GitHub. Both bundle all three files. Run `git push origin main` from `aitrends-project/` to push them. This is a one-time upload of existing history — from S15 onwards the split-commit rule applies.

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
| S14 | 17 June 2026 | both + docs | Site evaluation + SEO pass: site audit identified 6 issues; og-default.png created and deployed (ae87f97 aitrends-ng); AI signal gate tightened — broad terms (algorithm/automation) moved to WEAK list requiring 3+ body occurrences (8301fc3 scout-agent); Africa GATE hardened with substitution test + conditional "What this means for Africa" sentence (8301fc3); aitrends.ng root domain live via Whogohost A record → 76.76.21.21; SEO improvements: WebP images (removed `unoptimized` from post cover Image), `cover_image_prompt` as descriptive alt text on all cover images, `rel="nofollow"` on external source_urls, About page accuracy fix (correct feed list + HF FLUX image pipeline description) (5856ea6 aitrends-ng); www.aitrends.ng CNAME issue documented (outstanding); docs updated. |
| S15 | 18 June 2026 | aitrends.ng + docs | Document push strategy locked: SESSION_LOG + TRAINING_MANUAL = local-only commits; CLAUDE.md = push to GitHub. HF FLUX.1-schnell confirmed as active default image provider (image-providers/index.js). www.aitrends.ng confirmed still broken via curl (Whogohost CNAME not updated, www not in Vercel). updated_at added: Post interface `updated_at?: string | null`; `wasUpdated()` helper; "Updated [date]" in meta row; JSON-LD dateModified uses updated_at when available; Supabase migration SQL documented in to-do (71550cf aitrends-ng, pushed). |
| S17 | 19 June 2026 | scout-agent + aitrends.ng + docs | Post-deployment bug fixes: isYouTubeSource → category check (29e3641, c627d01); posts 108-110 reset via SQL UPDATE; AI Generated badges + FAIT attribution + Gemini/FLUX brand names removed from site (4d7753d); AI Trends category wired into all 6 frontend locations + purple badge + SEO metadata (48763db); cron-job.org podcast trigger added. |
| S16 | 19 June 2026 | scout-agent + aitrends.ng + docs | YouTube podcast pipeline built and local-tested. New files: feeds-podcast.js (6 channels), transcript.js (20k chars, 8-min gate, Tavily fallback, thumbnail download), gemini-podcast.js (multimodal: thumbnail base64 + transcript → 6-section editorial post + HF FLUX prompt), scout-podcast.js (Phase 1 with AI signal gate, 1 post per channel per run). SCOUT_MODE=podcast routing added to scout.js (3 lines). complete.js: isYouTubeSource check skips og:image for YouTube posts → HF FLUX path used. scout-podcast.yml GHA workflow (daily 06:00 UTC). Fixed: theMITmonk channel ID corrected (UC4ZVkG3RQPzvZk7alIVjcCg). Fixed: submitImageJob() — destructure { jobId } not raw object. aitrends.ng: 'ai' added to VALID_CATEGORIES (create/route.ts, lib/createPost.ts, sitemap.ts, about page, post page YouTube embed). supabase/schema.sql category constraint updated. Blocked: Supabase CHECK constraint on posts.category must be migrated manually before podcast posts can publish. Local test: 3 posts queued (Udacity MSc AI, Silicon Valley Girl AI-First Playbook with Peter Yang, Mo Gawdat on AI). Phase 2 image generation confirmed working. Commits: cdf7c6e + 33bffe6 (aitrends-ng), e0f5d0b (scout-agent). |
| S18 | 21 June 2026 | aitrends.ng + docs | Visitor engagement features: Vercel Analytics (@vercel/analytics, `<Analytics />` in layout.tsx); `/api/subscribe` newsletter endpoint + `NewsletterSignup` component in sidebar; `/api/feedback` per-post comments endpoint + `FeedbackForm` component on post pages. Two Supabase tables created by Felix: `newsletter_subscribers`, `feedback`. PostgREST schema cache issue diagnosed — "Could not find table in schema cache" error on both endpoints; fixed with `NOTIFY pgrst, 'reload schema';` in SQL Editor. Both endpoints verified on live site. Commit: 8796ed4 (aitrends-ng). |
| S19 | 22 June 2026 | scout-agent + docs | Status audit verified against live state instead of prior session notes: GHA run history (0 failures/50 runs), direct Supabase REST queries, live curl checks, real Slack API scope inspection. Found editorial_queue silent-drop bug — rows with failed extraction (blocked YouTube transcripts) or long pasted text with no URL were never consumed, stayed pending up to 6 days, retried and silently re-failed every run. Fixed: `failEditorialRow()` + `replyFailureToFelix()` added (memory.js, slack.js); felix.js now fails loudly + notifies Felix instead of dropping silently, and gives long no-URL text (>500 chars) its own post instead of a discarded topic hint. 4 stuck rows cleaned up live (2 failed, 2 left pending to auto-publish next run). Discovered SLACK_BOT_TOKEN lacks `chat:write` scope — every threaded Slack reply has silently failed since Session #9; flagged as Critical, requires Felix to update Slack app scopes (not fixable from code). Closed out 3 stale to-do items already resolved since Session #15 but never removed (www.aitrends.ng fix, updated_at migration, S13/S14 docs push). Removed orphaned `command (ypco5u)` debris file. |
| S20 | 23 June 2026 | docs only | Felix added 3 items to the to-do list, no code changes. (1) Google Alert for "AITrends.ng"/`site:aitrends.ng` — manual external setup, logged under Medium. (2) Homepage change-detection monitor (alert on title/description change, new nav/category links, non-AI keywords) — logged under Medium with two implementation options noted (third-party service vs. a custom scout-agent script following the project's existing build-it-yourself pattern), decision deferred to implementation time. (3) Tagline + footer rewording — exact old/new text specified by Felix; footer wording confirmed via clarifying question (footer mirrors the new tagline text, not separate wording). Logged under High, not yet executed — Felix said "add to to-do list," not "do it now." |
| S21 | 23 June 2026 | aitrends.ng | Executed the tagline/footer to-do from S20. New text "AITrends.ng — insights, and lessons for the future." applied to layout.tsx (description, og:description, twitter:description), about page subtitle, and Footer.tsx. Corrected Felix's "AItrends.ng" typo to the established brand spelling and normalized "--" to "—"; footer keeps its existing styled brand element and only swaps the trailing copy to avoid the brand name rendering twice in one line. Typecheck + production build clean; confirmed live via curl on the meta description. Commit `2c76eeb`. |
| S22 | 23 June 2026 | scout-agent + aitrends.ng | Built and shipped the homepage change-detection monitor from the S20 to-do. Felix chose the custom-built option over a third-party service (the off-topic-keyword check needed custom logic regardless). New `homepage-monitor.js` (independent of Phase 1/2, own GHA schedule every 6h), new `homepage_monitor` Supabase table (single-row snapshot, SQL written to `aitrends.ng/supabase/add-homepage-monitor.sql`), new `notifyMonitorAlert()` in slack.js reusing the existing webhook (sidesteps the chat:write scope gap from Session #19 entirely). Checks: title/description change, nav link add/remove (`nav.site-nav a`), and homepage post titles against an incident-driven off-topic keyword list (MTN, BNPL, football/FIFA — sourced from documented Session #8 incidents). Hit the same PostgREST schema-cache gotcha from Session #18 when creating the table — walked Felix through the SQL Editor location and the separate `NOTIFY pgrst, 'reload schema';` step explicitly, step by step, since the table wasn't visible via the API immediately after the first attempt. Verified live end-to-end: baseline saved on first run, second run correctly reported no changes, then a simulated change (patched directly in Supabase, not the live site) correctly triggered a Slack alert and the snapshot self-healed back to the real state on the next run. Commits: `e9fc094` (scout-agent), `65665a8` (aitrends.ng SQL reference). |
| S23 | 23 June 2026 | scout-agent + live Supabase data | Worked the 🔴 Critical to-do list. (1) HN Anthropic feed: confirmed hnrss.org 502 live, swapped for Bing News RSS, added `resolveTrackingLink()` to scout.js so Bing's click-tracking redirect resolves to the real publisher URL before storage (not the tracking link) — verified end-to-end against MSN/Yahoo/TheStreet articles. (2) ai-models fabrication risk: added a category-scoped extra-strictness clause to both `generateDigest()` and `generateBlendedDigest()` in gemini.js — model/product announcements from OpenAI/Google/HuggingFace/DeepMind no longer pass the Africa GATE on a bare "supports African languages" capability claim; requires an explicit named African deployment/partnership/initiative. Targets the documented Sierra Leone/DeepMind Gemma fabrication exactly; that existing live post itself was deliberately left untouched (flagged for a separate unpublish-or-rewrite decision, not a cleanup-script fix). (3) Old-post retroactive cleanup, scoped via three rounds of Felix's explicit sign-off before any write: stripped markdown asterisks from all 54 affected posts (same regex `publisher.js` already uses; verified 0/67 remain). Audited all 287 source_urls by fetching page titles and checking AI-signal keywords rather than comparing to the post's own metaphorical title; manually cross-checked for false negatives (URLs that were actually the correct primary source) before removing 68 confirmed-irrelevant URLs across 35 posts — sports/crime/politics/job-listings/directory-pages/unrelated-business-news (e.g. "Panic as Eriksen collapses", a movie review, Binance trading-competition pages); kept 19 genuine sources a naive keyword check would have wrongly flagged; left 32 unverifiable URLs untouched. Detected exact-phrase keyword stuffing (4–7 word phrases repeated 3+ times verbatim — the underlying pattern behind the old "×7" bold-stuffing issue, now visible in plain text) in 36 of 67 posts; sent the 25 with a non-generic phrase repeated 5+ times to Gemini for a fact-preserving rewrite capping repetition at ≤2 mentions. 15 of 25 completed, spot-checked for fidelity, and written; the remaining 10 are still outstanding — mid-session the work hit Google's Gemini free-tier hard daily cap (`GenerateRequestsPerDayPerProjectPerModel-FreeTier`, 20 requests/day for gemini-3.5-flash) from the combined volume of rewrite attempts. Nothing was written for those 10; the failures were silent 429s with no content risk. Flagged for Felix: if this `.env` key is the same one configured in GitHub Actions, today's remaining scheduled Phase 1/podcast runs may also 429 until the quota resets — open question whether production uses a separate key or needs a billing/quota upgrade given normal daily operation alone may already approach this cap. |
| S24 | 24 June 2026 | scout-agent + Slack + live Supabase data | Two threads. (1) Slack `chat:write` fix verified end-to-end: after Felix added the scope and reinstalled the app, a live `auth.test` call confirmed `chat:write` now present in the granted scopes (bot user `scout_editor`); the GitHub Actions `SLACK_BOT_TOKEN` secret was then synced directly from the verified local `.env` value via `gh secret set` (not pasted separately) to guarantee the two are byte-identical. Closes the gap open since Session #9. (2) Felix's long-pending Google Doc corrections finally applied — the same doc Claude Code had wrongly auto-actioned back in Session #7 (the incident that established the "never act on a document without explicit instruction" rule). Read the doc on request first, reported its 3 items and what Session #23 had already incidentally fixed vs. missed, then got Felix's explicit go-ahead per item before writing anything. Found a real gap in Session #23's own cleanup: 2 businessday.ng source_urls on the Zimbabwe/SA post survived because both titles contained "data" — a word that was on the AI-signal allowlist used to judge relevance, even though neither article was about that post's actual topic. Removed both. Also found the banned "For developers in Lagos, Accra, and Nairobi" template phrase (banned going forward since Session #7) still live in 6 of 121 published posts, not just the one example post named in the doc — fixed 4 of 6 via Gemini rewrite (spot-checked, written); the other 2, including Felix's own linked example post, hit the same Session #23 Gemini quota wall and remain outstanding. |
| S25 | 25 June 2026 | scout-agent + aitrends.ng + CLAUDE.md | Felix asked whether the remaining quota-blocked de-stuffing work could be scheduled, and whether checks existed to prevent needing this kind of cleanup again. Checking the second question surfaced a real, currently-live gap: grepped gemini.js for the Session #7 banned-phrase/anti-repetition guards and found zero matches — the Session #8 prompt simplification had silently dropped them, meaning new posts could still reproduce the exact problem just spent two sessions cleaning up. Felix's response reframed the whole problem: a narrow phrase-ban is whack-a-mole, because the forced phrases were never the root cause — the Africa-relevance mandate itself was, since Gemini would manufacture a connection by whatever means necessary whenever a genuinely non-Africa story still had to clear the gate. His decision: drop the Africa-first mandate entirely, make the site general-AI, leave all 121 existing published posts untouched regardless of relevance. Used EnterPlanMode given the scope (two repos, prompt logic, branding copy); two Explore agents mapped every Africa-dependency across scout-agent (gemini.js's gate blocks, scout.js's sentinel handling, feeds.js's 14 Africa + 12 global feed split, confirmed scout-podcast.js already fully general) and every Africa-branding location across aitrends.ng + CLAUDE.md (14+ locations across 9 files); a Plan agent validated the closing-sentence redesign against the existing Session #7 "TITLE VARIETY" precedent before any code was touched. Felix made three explicit scope calls before implementation: leave `EVERGREEN_ENTITIES`'s Africa scoring-boost terms alone (harmless tie-breaker), reject a fixed "Bottom line:" replacement label for the closing sentence ("too obvious, predictable... sometimes come out forced" — correctly identifying it as the same forcing problem in miniature), and keep `feeds.js` exactly as configured. Removed the full Africa GATE block (including the now-moot Session #23 ai-models strictness clause) from `generateDigest()`/`generateBlendedDigest()` in gemini.js; new closing-sentence instruction mandates varied construction (implication/forward-looking/skeptical/question) with fixed labels explicitly banned; removed the dead `SKIP_NO_AFRICA_RELEVANCE` sentinel check in scout.js. Verified with one real `runScout()` run rather than a batch (quota-conscious): 2 posts queued, one genuinely Africa-relevant on its own merits (Nigeria's NITDA digital sovereignty strategy) and one purely global with zero forced framing (Anthropic vs. Alibaba) — both closing sentences varied in construction, no sentinel leakage, no forced phrasing, confirming the gate removal works as intended rather than just assumed. Updated branding/copy across both repos (layout.tsx, about page, Sidebar, NewsletterSignup, feed.xml, CLAUDE.md Section 1 mission/tagline/voice/audience, the Gemini Prompt Mandate subsection) — `npm run build` clean. Marked the now-superseded Session #23 to-do item and a now-moot Medium to-do item with dated annotations rather than rewriting their original text, per this file's own append-only convention. |
| S26 | 25 June 2026 | scout-agent + aitrends.ng + CLAUDE.md | Three threads. (1) Felix confirmed completing the Search Console sitemap submission and Google Alert setup himself — marked both done in the 🟠 High / 🟡 Medium to-do lists. (2) Scheduled the 12 remaining quota-blocked cleanup posts (10 de-stuffing + 2 forced-phrase, from Sessions #23–24) as a one-time cloud routine via the `/schedule` skill for 2026-06-27 08:00 UTC, since the cloud agent runs in an isolated environment with its own git checkout — wrote a self-contained prompt giving it the exact detection methodology (regex/phrase-repetition logic, not a fixed post list, since the live set could shift) rather than relying on it remembering specifics from this conversation; flagged that the cloud environment's Gemini/Supabase secrets weren't confirmed configured. (3) Built site search for aitrends.ng — live dropdown-as-you-type in the NavBar searching title+excerpt+tags, per Felix's explicit UX choices. New `app/api/search/route.ts` (GET, public Supabase client, `.or()` on ilike title/excerpt + `.cs.` exact-match on tags, comma/paren sanitization since those are PostgREST `.or()` syntax characters), new `components/SearchDropdown.tsx` (debounced fetch mirroring the existing `NewsletterSignup.tsx` pattern, click-outside + Escape close, mobile collapses to a tap-to-expand icon rather than hiding search entirely), `NavBar.tsx` + `globals.css` updated to match. Verified the backend live against real Supabase data (title match, tag match via a post whose tags — not title — contained the query, the 1-char guard, comma-injection sanitization, no-match case all correct) and confirmed the component renders server-side; no headless browser tool (chromium-cli/Playwright) was available locally and this is Felix's real Mac rather than a sandboxed container, so rather than silently downloading a fresh Chromium binary for a one-off check, asked Felix to do the interactive click-through himself — he confirmed it works. (4) Felix flagged two formatting issues on the YouTube podcast pipeline's posts: literal "Main Idea 1/2/3..." labels instead of real subheadings, and a separate "Notable Quotes" list instead of quotes woven into the prose naturally. Fixed both in `gemini-podcast.js`'s prompt (the label instruction now explicitly forbids "Main Idea N" prefixes; the quotes section instruction now tells Gemini to weave verbatim quotes into the relevant theme section's prose, reserving `<blockquote>` for the rare standout quote rather than the default). Demonstrated the fix on one real published post ("The AI-First Playbook: How Peter Yang Runs a 140K Newsletter Solo") by manually restructuring its existing content rather than spending a Gemini call — dropped the "Main Idea N:" prefixes from its four section headings and rewove all 5 previously-listed quotes into the body paragraphs they actually support (verified live: zero "Main Idea" or "Notable Quotes" text remains, all section headings now read as natural descriptive subheadings). |
| S27 | 25 June 2026 | scout-agent | Felix asked a detailed operational question: when Gemini's daily quota is exhausted, does the pipeline degrade gracefully, or do cron jobs keep retrying and accumulating failed attempts/error noise until the quota resets? Investigated rather than assumed: confirmed gemini.js's production retry logic doesn't actually retry-storm on 429 (only 503 triggers retry+fallback) — that risk was specific to ad-hoc cleanup scripts, not the live pipeline. But found three real problems: (1) scout.js's RSS fetch + Tavily search + fetchContent all run *before* the Gemini call per category, so the expensive upstream work is wasted every 6h while quota is dead; (2) each category's try/catch swallows the error and never re-throws, so `runScout()` exits 0 (success) even when every category failed — meaning the GHA "Notify Slack on failure" step (`if: failure()`) never fires, so the degradation is completely silent, not just noisy; (3) no quota-state tracking existed anywhere to let a future run skip early. Also discovered the model fallback only triggers on 503, never 429 — meaning `gemini-2.5-flash`'s entirely separate daily quota bucket had never been touched by any of Sessions #23–24's cleanup work despite repeatedly hitting 429 on `gemini-3.5-flash` — a near-free fix on its own. Discussed design before implementing (Felix: "Let's talk through first"): reactive detection chosen over proactive self-counting (a self-counter can't see manual one-off work hitting the same Google-side quota, so it would give false confidence); full-run skip chosen over partial skip (trades a rare missed short-lived RSS item for fully eliminating wasted Tavily/fetch calls); alert once per day, not per skipped run. Built: `gemini.js`'s three duplicated retry blocks (generateDigest, generateBlendedDigest, generateEditorialDigest) extracted into one shared `callGeminiWithFallback()`, now falling back to 2.5-flash on 429 too, only recording a cooldown when *both* models 429 in the same attempt; discovered `gemini-podcast.js` had its own independent copy of the identical retry logic (same shared quota) and refactored it to reuse the new shared helper instead of tripling the fix; `memory.js`/`slack.js` gained `isGeminiQuotaCooledDown()`/`setGeminiQuotaCooldown()` (single-row Supabase state, same pattern as `homepage_monitor`; a cooldown from a previous calendar day is treated as stale rather than computing Google's exact reset time) and a once-per-day Slack alert; `scout.js`/`scout-podcast.js` both check the cooldown before any work and skip the entire run if already exhausted today, and `break` out of remaining categories/channels immediately if exhaustion is hit mid-run rather than continuing to burn calls on items that will fail the same way. SQL migration written to `aitrends.ng/supabase/add-gemini-quota-cooldown.sql`, mirroring `homepage_monitor`'s exact single-row RLS pattern. Felix hit the same "clear the editor" confusion as Session #22 (thought it meant clearing the table, not the SQL Editor's text box) — clarified explicitly. Fully verified live against the real table before shipping: no-cooldown state, new-cooldown-sets-true, same-day-repeat-returns-false (dedup for the alert), and a cooldown timestamped to a previous day correctly treated as stale — test row deleted afterward so the real pipeline starts clean. Closed the loop with one real `runScout()` execution confirming the Session #25 Africa-gate removal and this session's quota protection work correctly together in production code: 2 posts queued (one genuinely Africa-relevant on its own merits, one purely global with zero forced framing), no sentinel leakage, no forced phrasing. |
| S28 | 13 July 2026 | scout-agent | Diagnosed why Scout published zero posts since July 7: `isGeminiQuotaCooledDown()` in memory.js compared UTC calendar dates (`hitDate === today`), but Gemini's quota resets at midnight Pacific, not midnight UTC. The `~01:47 UTC` GHA cron run (6:47 PM PT) always passed the cooldown check (new UTC day), tried Gemini (still exhausted — still previous PT day), hit429, and re-set the cooldown with the new UTC date. This blocked the `05:00 UTC` run (=midnight PT) when the quota actually reset, and every subsequent run that UTC day. The cooldown was self-perpetuating: each day's first run to pass the check would re-set it before midnight PT, blocking the fresh-quota window permanently. Fix: replaced UTC-date comparison with elapsed-time staleness check (`Date.now() - hit_at < 20 hours`). 20-hour threshold is timezone-agnostic, covers worst-case PT-to-reset gap. After fix is live: `DELETE FROM gemini_quota_cooldown WHERE id = 1;` to clear the stuck row. Zero posts published July 7–13; quota was genuinely exhausted but the self-resolve mechanism was broken. |
| S29 | 15 July 2026 | scout-agent + aitrends-project | Root cause confirmed: billing, not timezone. Old Gemini key hit Tier 1 billing after ~28-day free trial — all429s were billing errors. Created new GCP project, new API key. Discovered `gemini-2.5-flash` deprecated for new keys (404). Used `gemini-3.5-flash` instead (200 confirmed). Added Groq Llama 3.3 70B as cross-provider fallback (14,400 req/day free). Rewrote `gemini.js`: `callGroq()` translates Gemini contents→OpenAI messages, `normaliseGroqResponse()` wraps Groq output in Gemini shape, no caller changes needed. Updated GH Actions secrets (GEMINI_API_KEY + GROQ_API_KEY). Cleared cooldown row, triggered manual run — 90 fresh articles, 2 posts queued, 1 published. Pipeline alive after 8 days of silence. Known: 3.5-flash will hit same 28-day billing cliff; fix is same (new project, new key). |