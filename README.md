# Prism

**AI-native, agentic creator-affiliate matching and revenue-attribution engine for micro/nano creators.**

Named for the idea: one product, refracted through the AI matcher, splits into dozens of narrow, high-intent creator partnerships instead of one broad influencer bet.

## Problem

Brands want to run performance-based (affiliate) deals with hundreds of micro/nano creators (<10K followers) because they convert better than celebrity posts — but finding the right creators, onboarding them, and tracking who actually drove a sale is a manual, spreadsheet-driven mess. Existing affiliate trackers (Impact, Partnero) handle the *tracking* half but do nothing for *discovery* — brands still have to find relevant creators themselves, usually by hashtag search, which surfaces creators who use a keyword, not creators whose audience actually has the problem the product solves.

## Solution

A brand pastes a product description. Prism embeds it, semantically matches it against a database of creator content (not creator bios or hashtags), ranks candidates by fit, and — once a deal is accepted — a set of agents take over: they draft outreach, generate a creator-specific creative brief, issue a trackable promo code, attribute resulting sales via storefront webhooks, watch for fraud and disclosure compliance, and settle commission. One platform, mostly run by agents, covers discovery → outreach → tracking → compliance → payout.

## Core Features

| # | Feature | What makes it non-trivial |
|---|---|---|
| 1 | **Semantic creator matching** | Matches on embedded *content transcripts*, not hashtags/keywords — a creator who talks about postpartum nutrition surfaces for a "postpartum protein powder" query even if they've never used #fitness. |
| 2 | **Predictive ROI scoring** | Given a creator's historical engagement + content style + a target price point, estimate expected conversions before the brand commits budget. |
| 3 | **Promo-code-first tracking** | Cookie tracking is dying (iOS ITP, privacy browsers). Primary attribution path is a unique per-creator discount code redeemed at checkout — survives cookie clearing, works cross-device. Click-based cookie tracking is a secondary, lower-confidence signal. |
| 4 | **AI creative brief generator** | On deal acceptance, generate a creator-specific brief (hook, angle, format) derived from that creator's own historically top-performing content — not a generic brand brief. |
| 5 | **Real-time performance dashboard** | Clicks, conversions, CPA, and owed payouts per creator/deal, updated as webhooks land. |
| 6 | **Conversational reporting** | Brand asks "which 3 creators drove the most revenue from TikTok last week?" in plain English; answered via a constrained NL→SQL agent, not a BI dashboard the brand has to learn. |
| 7 | **Lookalike expansion** | "Find more creators like this one that already converted" — a second matching pass seeded by a winning deal instead of the original product description. |
| 8 | **Creator self-serve marketplace** | Creators browse open briefs and apply directly, cutting the brand's outbound-matching load. |
| 9 | **Refund/chargeback clawback** | Shopify `refunds/create` webhook reverses the linked `ConversionEvent` and any unpaid commission — the original spec had no refund path. |
| 10 | **Cohort/campaign analytics** | Roll-up performance by campaign or niche, not just per-creator. |

## Agentic AI Layer

Everything above the raw CRUD layer is agent-driven — not single-shot LLM calls, but tool-calling loops that can chain steps and act on their own within guardrails. The agents below are wired together as a **LangGraph** state graph (so handoffs like matching → brief → outreach are explicit edges, not ad-hoc function calls), and every run is traced in **LangSmith** — with 6+ agents, "why did it pick this creator" or "why did compliance flag this" needs to be inspectable, not a black box.

| Agent | What it does | Guardrail |
|---|---|---|
| **Matching agent** | Iterates on the semantic search: re-queries if the first pass is thin, filters by platform/price-tier/availability, explains *why* it picked each creator | Read-only tools; never creates a `Deal` itself |
| **Outreach agent** | Drafts and sends creator outreach, auto-follows-up after N days of silence | Brand approves the first message template before it's used |
| **Negotiation agent** | Handles commission counter-offers within brand-set bounds | Escalates to a human the moment a counter falls outside those bounds |
| **Brief-generation agent** | Builds a creator-specific creative brief from that creator's top-performing `ContentPiece` rows | — |
| **Compliance agent** | Scans posted creator content for FTC/ASCI disclosure language (`#ad`, `#sponsored`) | Flags for human review; never auto-penalizes a creator |
| **Fraud agent** | Flags click fraud and fake-engagement/bot-follower patterns before a deal is funded | Flags, doesn't auto-reject |
| **Reporting agent** | NL→SQL over Neon for the brand chat interface | Query is read-only and scoped to that brand's own rows only |

## Architecture

```
                        ┌─────────────────────┐
                        │   Brand Portal       │  React (Next.js, Vercel)
                        │  match/swipe UI,     │
                        │  dashboard, NL chat  │
                        └──────────┬───────────┘
                                   │ REST/JSON
┌─────────────────────┐           │           ┌─────────────────────┐
│  Creator Portal      │──────────┼──────────▶│   Django API         │
│  links, earnings,    │  REST    │           │  (DRF, on Render)     │
│  payout requests     │          │           │  - auth                │
└─────────────────────┘          │           │  - deal lifecycle       │
                                  │           │  - commission engine    │
                                  │           │  - agent orchestration  │
                                  │           └──────────┬──────────────┘
                                  │                      │
                     ┌────────────┴───────┐    ┌─────────┴──────────┐
                     │ Tracking pixel      │    │ Webhook listener    │
                     │ /track?ref=creator  │    │ Shopify orders/     │
                     │ → cookie, redirect  │    │ refunds → attribute │
                     └────────────┬────────┘    └─────────┬───────────┘
                                  │                        │
                                  ▼                        ▼
                        ┌────────────────────────────────────┐
                        │        Neon Postgres + pgvector      │  ← free tier
                        │  brands · creators · content_pieces  │
                        │  (+ embeddings) · deals · clicks ·   │
                        │  conversions · promo_codes · payouts │
                        └───────────────┬───────────────────┘
                                        │
                        ┌───────────────┴───────────────────┐
                        │  LangGraph agent layer (traced       │
                        │  in LangSmith)                       │
                        │  - Groq (Llama 3.3 70B): all agent    │
                        │    reasoning + brief/outreach text    │
                        │  - Groq Whisper large-v3: transcribe  │
                        │    IG/TikTok audio (no captions API)  │
                        │  - OpenAI text-embedding-3-small:     │
                        │    the ONE paid call in the system    │
                        └──────────────────────────────────────┘

  Cloudflare Turnstile guards the tracking-pixel endpoint (bot/click-fraud filter)
  Sentry watches both Django and Next.js for errors (esp. cold-start timeouts)
  Telegram bot pushes real-time ops alerts: new deal, fraud flag, failed webhook
```

## Tech Stack — what's used where, and what it costs

Total out-of-pocket budget: **under $5, and only OpenAI gets paid.** Everything else runs on a free tier. That constraint drove several choices below away from the "obvious" pick.

| Layer | Tech | Responsibility | Cost |
|---|---|---|---|
| Brand Portal | React (Next.js), **shadcn/ui**, Tailwind, Recharts, Framer Motion | Matching flow, swipe review, dashboard, NL chat | Free (Vercel) |
| Creator Portal | React (Next.js), shadcn/ui, mobile-first | Links/codes, earnings, payout requests | Free (Vercel) |
| Backend API | Django + Django REST Framework | Auth, CRUD, commission calc, agent orchestration | Free (Render web service) |
| Auth | **Clerk** | Hosted login/signup UI for both portals, session tokens verified on the Django side via `clerk-backend-api` | Free tier (10k MAU) |
| Agent orchestration | **LangGraph** | Wires the agents below into an explicit state graph (matching → brief → outreach → negotiation) instead of ad-hoc chains | Free (open source) |
| Agent observability | **LangSmith** | Traces every agent run — which tools fired, why a creator was picked, why compliance flagged something | Free tier |
| Agent reasoning/generation | **Groq** (Llama 3.3 70B) | All agent text: matching rationale, outreach drafts, briefs, compliance scan, NL→SQL | Free tier |
| Audio transcription | **Groq Whisper large-v3** | Transcribes IG Reels/TikTok audio (no captions API on those platforms) | Free tier |
| Embeddings | **OpenAI `text-embedding-3-small`** | Vectorizes product descriptions and creator transcripts | **~$0.02 / 1M tokens — the only paid line item** |
| Vector + relational store | **Neon Postgres + `pgvector`**, queried via **Django ORM** (`pgvector.django.VectorField` + `CosineDistance`, not raw SQL) | Single DB for structured data *and* embeddings; branching to test commission logic safely | Free tier (0.5 GB) |
| Cache / rate-limit | **Upstash Redis** (REST API, no persistent connection) | Dedup clicks, rate-limit the tracking pixel | Free tier (10k commands/day) |
| API docs | **drf-spectacular** | Auto-generates OpenAPI/Swagger docs straight from the DRF views — zero-config | Free (open source) |
| Scheduled/background work | **Django + APScheduler**, in-process (same pattern as Nexus) + **GitHub Actions** scheduled workflow to trigger heavier batch jobs and ping the app to prevent cold sleep | No separate Celery worker dyno needed | Free |
| Click-fraud filter | **Cloudflare Turnstile** | CAPTCHA-alternative challenge on the tracking-pixel endpoint, feeds the fraud agent | Free |
| Error monitoring | **Sentry** | Catches backend + frontend errors — especially cold-start timeouts on the pixel/webhook endpoints | Free tier |
| Ops alerting | **Telegram Bot API** | Real-time push alerts — new deal, fraud flag, failed webhook — straight to a Telegram channel | Free (plain HTTP calls, no SDK needed) |
| Outreach email | **Brevo** | Transactional email for the outreach agent's actual sends | Free tier (300 emails/day) |
| Promo-code issuance | Django service → Shopify Admin API | Unique per-creator discount code on deal acceptance | Free (dev store / partner account) |
| Payouts | Stripe Connect | Moves commission to a creator's account | Free to integrate — fees are a % of real transferred money, not a fixed cost, and test mode is $0 |
| Hosting | Vercel (both React apps) + Render free web service (Django) | | Free |

### Why Clerk, but not Celery
Both are "should we use the default pick" calls, but they resolve differently because the constraint isn't "is the vendor free" — it's "does this specific plan avoid paying for compute we'd otherwise need."

- **Using Clerk** — its free tier (10k MAU) is genuinely free at this project's scale, and it removes the cost of *building and maintaining* auth screens, password reset, session handling, and social login by hand. There's no hidden compute cost lurking in "free tier" here — it's a hosted service you call, not a process you host.
- **Not using Celery** — Celery itself is free (open source), but running it requires a **persistent worker process**. Free web-service tiers (Render, Railway, Fly) are built around request-driven dynos that sleep when idle; a Celery worker needs to stay alive continuously to poll its queue, which usually means paying for a dedicated worker dyno even when the rest of the stack is free. Swapping in **in-process APScheduler** (same pattern already used in Nexus) plus **GitHub Actions scheduled workflows** for heavier batch jobs gets the same periodic/async behavior without ever needing an always-on process. If Prism outgrows this (real click volume, real async load), Celery is the correct next step — it's the right tool once there's revenue to pay for the worker it needs.
- **Not using OpenAI for generation/Whisper** — Groq serves Llama 3.3 70B and Whisper large-v3 on a free tier; OpenAI is used *only* where nothing free is good enough (embeddings).

## Repo Structure

```
prism/
├── README.md
├── .gitignore
├── backend/
│   ├── requirements.txt
│   └── .env.example
└── frontend/
    ├── package.json
    └── .env.local.example
```

`backend/` is the Django + DRF API (deployed to Render). `frontend/` is a single Next.js app serving both the Brand Portal and Creator Portal as route groups behind Clerk auth, deployed to Vercel.

## Environment Variables

**`backend/.env`** (see `backend/.env.example`)

| Variable | Used for |
|---|---|
| `DJANGO_SECRET_KEY` | Django's own cryptographic signing |
| `DJANGO_DEBUG`, `ALLOWED_HOSTS`, `CORS_ALLOWED_ORIGINS` | Environment/deploy config |
| `DATABASE_URL` | Neon Postgres connection string (`pgvector` extension enabled on that DB) |
| `GROQ_API_KEY` | All agent reasoning/generation + Whisper transcription |
| `OPENAI_API_KEY` | Embeddings only — the one paid credential |
| `LANGCHAIN_TRACING_V2`, `LANGCHAIN_API_KEY`, `LANGCHAIN_PROJECT` | LangSmith tracing for the LangGraph agent runs |
| `CLERK_SECRET_KEY`, `CLERK_PUBLISHABLE_KEY` | Verifying the session token Clerk issues to the frontend |
| `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN` | Click dedup / rate-limiting cache |
| `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET`, `SHOPIFY_APP_URL`, `SHOPIFY_WEBHOOK_SECRET` | Shopify app OAuth + verifying incoming order/refund webhooks |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` | Payout transfers via Stripe Connect |
| `SENTRY_DSN` | Backend error reporting |
| `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` | Ops alerts (new deal, fraud flag, failed webhook) pushed via plain HTTP call to the Telegram Bot API |
| `BREVO_API_KEY` | Outreach agent's transactional email sends |
| `CLOUDFLARE_TURNSTILE_SECRET_KEY` | Server-side verification of the Turnstile token from the tracking pixel |
| `YOUTUBE_API_KEY` | Phase 2 — creator caption ingestion, not needed for the MVP |

**`frontend/.env.local`** (see `frontend/.env.local.example`)

| Variable | Used for |
|---|---|
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk's client-side SDK (safe to expose) |
| `CLERK_SECRET_KEY` | Clerk's Next.js server-side middleware |
| `NEXT_PUBLIC_CLERK_SIGN_IN_URL`, `NEXT_PUBLIC_CLERK_SIGN_UP_URL` | Clerk hosted auth page routes |
| `NEXT_PUBLIC_API_BASE_URL` | Where the frontend calls the Django API |
| `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | Cloudflare Turnstile widget (public site key; secret stays backend-only) |
| `NEXT_PUBLIC_SENTRY_DSN` | Frontend error reporting |

> **shadcn/ui note**: it isn't installed via `package.json` — components are added on demand with `npx shadcn@latest add <component>`, which copies the component source directly into `frontend/`. `tailwindcss-animate` (already in `package.json`) is its one real dependency.

## Where data lives

**One database, two kinds of data, no separate vector store:**

- **Neon Postgres** holds everything — brands, creators, deals, clicks, conversions, promo codes, payouts (structured rows) *and* the `pgvector` embedding column on `ContentPiece` (unstructured/semantic data). Semantic search is one SQL query (`ORDER BY embedding <=> query_embedding`) against the same database that holds the people/company records — no sync job between a relational DB and a separate vector DB (Qdrant/Pinecone) to keep consistent or pay for.
- **Upstash Redis** holds only ephemeral/cache data — click dedup keys, rate-limit counters. Nothing there is a system of record; losing it costs you nothing but a cache warm-up.
- **No blob/object storage.** Downloaded creator audio is transcribed in-memory/temp-disk and discarded — only the resulting transcript text + embedding is persisted. This sidesteps needing S3/R2 for the MVP and keeps the "who's storing my content" surface area small, which also simplifies any future privacy conversation with creators.
- **PII** (brand contact info, creator email, payout details) lives in the same Neon Postgres — payout *bank details specifically* should go through Stripe Connect's own onboarding (Stripe holds the sensitive banking data, you only store their Stripe account ID), so Prism itself never custodies bank account numbers.

## Data Model (core tables)

- `Brand` — account, storefront platform + API credentials
- `Creator` — profile, connected platforms, Stripe Connect account ID
- `ContentPiece` — one creator video/post + its transcript + embedding
- `Deal` — brand ↔ creator agreement, commission terms, status
- `PromoCode` — one per deal, maps to a Shopify discount code
- `ClickEvent` — pixel hit: ref, ip, user_agent, timestamp
- `ConversionEvent` — attributed sale: deal, order value, attribution method (promo_code | cookie), refund status
- `Payout` — batched commission owed/paid to a creator

## Key Flows

1. **Matching** — brand pastes product description → embedded (OpenAI) → Django ORM query (`ContentPiece.objects.order_by(CosineDistance("embedding", query_embedding))`) over `pgvector` → matching agent (Groq) refines and explains the ranked, deduped-by-creator results.
2. **Deal acceptance** — creator accepts → Django mints a `PromoCode` via Shopify Admin API → brief-generation agent (Groq) writes a creative brief from that creator's top-performing `ContentPiece` rows.
3. **Click** — follower clicks creator's link → tracking pixel logs `ClickEvent` (Upstash-backed dedup), sets cookie, redirects to store.
4. **Attribution** — Shopify `orders/create` webhook fires → Django checks the order for a matching promo code first, cookie second → writes `ConversionEvent` → commission computed and queued for payout.
5. **Refund** — Shopify `refunds/create` webhook fires → linked `ConversionEvent` and any unpaid commission are reversed.
6. **Conversational reporting** — brand asks a question in the portal chat → reporting agent (Groq) generates a constrained, read-only, brand-scoped SQL query against Neon → result rendered as an answer.

## How creator & brand data actually gets collected

**Brands — easy:** Shopify OAuth app-install flow. The moment a brand installs the Prism app from the Shopify App Store, you get their catalog and webhook access for free. This is also the acquisition channel — no separate marketing needed to reach brands.

**Creators — the real cold-start problem:**
- **YouTube first.** The YouTube Data API gives you captions directly on OAuth connect — no transcription cost, no ToS risk, no Whisper call needed. This is the cheapest, most reliable content source, so it should be the *only* platform in Phase 1.
- **Instagram/TikTok next.** Neither gives you a spoken transcript via their official APIs. Pull video through the official Graph API / TikTok Creator Marketplace API (never scrape — ToS risk, and account ban risk for the creator), then run the audio through **Groq Whisper** to get transcript text into the same embedding pipeline. This is Phase 2, after YouTube-only has validated the matching quality.
- **Seed one side before the other.** Manually onboard a few hundred creators in a single niche (e.g. wellness) via a lightweight signup + YouTube-connect flow *before* approaching any brand. Go to brands with "we already have 300 vetted creators in your niche," not an empty marketplace.
- **Build vs. buy** is a real option worth considering later: licensing a creator directory (Modash/Upfluence/HypeAuditor public APIs) for base profile/follower data would remove the cold-start problem entirely — Prism would still differentiate on the AI matching/tracking/agent layer on top, just not on data acquisition. Not needed for a $5 MVP, but worth revisiting if this becomes a real product.

## What else to watch on a free-tier stack

Free tiers make the $5 budget possible, but each one has a sharp edge:

- **Render free web service sleeps after ~15 min idle.** A cold start on the *tracking pixel* or the *Shopify webhook* endpoint is the actual risk here — Shopify's webhook timeout is a few seconds, and a sleeping dyno can miss that window, silently dropping a conversion. Mitigate with a free external pinger (UptimeRobot or cron-job.org, both free) hitting the app every 5–10 minutes to keep it warm.
- **Neon free tier auto-suspends compute after inactivity too** — same cold-start risk on the DB side, compounding the above. The same keep-warm ping helps here as a side effect.
- **Upstash Redis free tier caps at 10k commands/day** — fine for a demo, but click-volume at real scale would blow through this; know it's a scaling wall, not a permanent solution.
- **Groq's free tier has request/token-per-minute rate limits** — fine for demo traffic; if multiple agents fire in a burst (e.g. matching + brief-generation back to back), watch for 429s and add basic backoff.
- **Shopify Partner app review has a lead time** — apply for the app listing early, not right before you need webhook/promo-code access live.
- **GitHub Actions free minutes** (2,000/month private, unlimited public) — fine for scheduled cron replacement, but don't lean on it for anything latency-sensitive; it's a scheduler, not a queue.
- **No paid backups.** Neon's free tier doesn't guarantee the backup SLA a paid plan would — for anything you can't afford to lose (payout ledgers), consider a periodic manual export until there's revenue to justify a paid tier.

## Phased Scope

**Phase 1 (MVP)** — YouTube-only creator data, semantic matching (no ROI prediction yet), manual promo-code assignment, Shopify webhook attribution, basic dashboard, matching + reporting agents only.

**Phase 2** — Instagram/TikTok via Groq Whisper transcription, predictive ROI scoring, brief-generation and outreach agents, WooCommerce support, refund clawback.

**Phase 3** — negotiation and compliance/fraud agents, lookalike expansion, creator self-serve marketplace, multi-tier commission logic (tested via Neon branching before merging to prod), Stripe Connect payouts live.

## Open Questions / Risks

- **Cold start on ROI scoring**: needs labeled historical conversion data per creator before the prediction is trustworthy — not viable until Phase 1 has run long enough to generate that data.
- **Commission-branching workflow**: "test on a Neon branch, merge to prod" needs an actual defined process (which branch, how it's validated, how/when it's promoted) — currently just a capability, not a workflow.
- **Compliance agent scope**: flagging FTC/ASCI disclosure language is straightforward text matching to start; genuinely verifying *visual* disclosure (e.g. an on-screen sticker in a Reel) would need vision-model calls, which reintroduces a second paid API — deliberately out of scope until Phase 3 at earliest.
