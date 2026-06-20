# 90-Day Build Roadmap — Questionsfactory 2.0

## Phase Overview

| Phase | Weeks | Focus | Deliverable |
|---|---|---|---|
| 1 | 1-3 | Foundation | Monorepo + Supabase + auth working |
| 2 | 4-6 | Core Product | Scoring engine + training loop + dashboard |
| 3 | 7-9 | Extension | Chrome Extension live on Chrome Web Store |
| 4 | 10-12 | Team Features | Analytics + pilot readiness |

---

## Phase 1: Foundation (Weeks 1-3)

### Week 1 — Monorepo & Supabase

**Goal**: `pnpm dev` runs web app with auth working.

Tasks:
- [ ] Scaffold Turborepo with `apps/web`, `apps/extension`, `packages/scoring`, `packages/ui`
- [ ] Create Supabase project (free tier initially)
- [ ] Run migration 001-005 from `docs/04-database-schema.md`
- [ ] Configure Supabase Auth (email/password + Google OAuth)
- [ ] Set up `apps/web` with Next.js 15 App Router
- [ ] Implement auth guard in `app/(app)/layout.tsx`
- [ ] Deploy to Vercel (preview URL)

**Done when**: User can sign up, log in, and see an empty dashboard.

### Week 2 — Org Onboarding

Tasks:
- [ ] Org creation flow (name, slug, invite link)
- [ ] Invite link → join org flow
- [ ] User profile setup (display name)
- [ ] Basic team creation UI (managers only)
- [ ] Seed 20 global lessons (5 per difficulty × 4 dimensions to start)

**Done when**: A manager can create an org, invite 2 teammates, assign them to a team.

### Week 3 — Scoring Engine Package

Tasks:
- [ ] Implement `packages/scoring/src/rule-based.ts` (all 6 dimensions)
- [ ] Write unit tests for rule-based scorer (target: 20+ test cases)
- [ ] Implement `packages/scoring/src/llm-scoring.ts` (calls Edge Function)
- [ ] Set up `supabase/functions/score-prompt/` Edge Function
- [ ] Integration test: submit a prompt → receive scores

**Done when**: `scoreRuleBased("Help me write code")` returns scores for all 6 dimensions in < 10ms.

---

## Phase 2: Core Product (Weeks 4-6)

### Week 4 — Training System

Tasks:
- [ ] Training session UI (`app/(app)/training/`)
- [ ] Lesson selector (filtered by user's weakest dimensions)
- [ ] Answer submission → LLM scoring via Edge Function
- [ ] Feedback display (dimension breakdown + suggestions)
- [ ] Mistake tracking (write to `mistake_records`)
- [ ] Ability profile update after each session

**Done when**: User can complete a training session end-to-end and see dimension scores improve.

### Week 5 — Personal Dashboard

Tasks:
- [ ] Radar chart: 6-dimension ability profile (Recharts or Nivo)
- [ ] Score trend line chart (30-day history)
- [ ] Training streak counter
- [ ] "Your weakest dimension" card with suggested next lesson
- [ ] Recent sessions list
- [ ] Score card: shareable PNG export (via `/api/og` route)

**Done when**: Dashboard clearly shows a user's strengths/weaknesses and recommends next action.

### Week 6 — AI Coach

Tasks:
- [ ] Coach interface (`app/(app)/coach/`)
- [ ] Streaming chat UI with Supabase Edge Function
- [ ] Coach has context: user's ability profile + recent mistakes
- [ ] Coach can suggest specific exercises
- [ ] Mistake review queue (lessons targeting past mistakes)

**Done when**: User can have a 5-turn coaching conversation that references their personal data.

---

## Phase 3: Chrome Extension (Weeks 7-9)

### Week 7 — ChatGPT Integration (Primary Target)

Tasks:
- [ ] Set up `apps/extension` with Vite + React + Manifest V3
- [ ] Content script for `chat.openai.com`
  - [ ] MutationObserver to detect textarea
  - [ ] Intercept form submit event
  - [ ] Run rule-based scorer on textarea content
  - [ ] Inject score overlay badge (React portal)
- [ ] Background service worker: auth + Supabase sync queue
- [ ] Extension popup: sign-in + current score summary

**Done when**: Score badge appears when typing in ChatGPT.

### Week 8 — Satisfaction Rating + Sync

Tasks:
- [ ] Satisfaction modal component (5-star, 30s delay after response)
- [ ] Sync `PromptLogEntry` to Supabase (excluding raw text)
- [ ] Retry queue for failed syncs (offline resilience)
- [ ] Privacy notice on first install (explain what's tracked)
- [ ] Options page: toggle tracking per platform

**Done when**: Full loop works: Type → score → send → AI responds → rate → data syncs.

### Week 9 — Multi-Platform + Web Store

Tasks:
- [ ] Claude.ai content script
- [ ] GitHub Copilot Chat content script
- [ ] Gemini content script
- [ ] Polish overlay UI (responsive, doesn't break site layout)
- [ ] Chrome Web Store submission (screenshots, description, privacy policy)
- [ ] Wait for review approval (~3-7 days)

**Done when**: Extension published on Chrome Web Store, installs work for all 4 platforms.

---

## Phase 4: Team Features & Pilot Readiness (Weeks 10-12)

### Week 10 — Team Analytics

Tasks:
- [ ] Team dashboard (`app/(app)/team/`)
- [ ] Refresh `team_weekly_stats` materialized view via scheduled Edge Function
- [ ] Team leaderboard (anonymized by default, manager can enable names)
- [ ] Dimension heatmap: which dimensions are weak across team?
- [ ] Week-over-week trend

**Done when**: Manager can see their team's aggregate prompt quality without seeing individual raw logs.

### Week 11 — Correlation Analytics

Tasks:
- [ ] Analytics page (`app/(app)/analytics/`)
- [ ] Scatter plot: prompt score vs satisfaction rating (per user)
- [ ] Team-level correlation coefficient display (CORR() from materialized view)
- [ ] "Correlation report" PDF export for manager to share with leadership
- [ ] Minimum data threshold UX: "Need 50 more ratings to generate report"

**Done when**: Manager can see r-value and download a one-page correlation report.

### Week 12 — Pilot Readiness

Tasks:
- [ ] Landing page: clear value prop, pricing table, "Request pilot" CTA
- [ ] OG image generation for score cards (`/api/og`)
- [ ] Demo mode: fake data for sales demos (no account required)
- [ ] Admin panel: manage orgs, view pilot status, usage stats
- [ ] SSO/SAML setup docs (Enterprise tier onboarding)
- [ ] Onboarding email sequence (welcome, day 3 tips, week 1 report)

**Done when**: A prospect can visit the landing page, request a pilot, and be onboarded within 24 hours.

---

## Week 13+: Pilot Outreach

- Identify 10 target companies (Copilot or ChatGPT Enterprise customers, 50-200 employees)
- LinkedIn outreach to L&D Managers / CTOs
- Offer: "Free 90-day pilot — we'll build your correlation report at no cost"
- Onboard first 3 pilots
- At day 45: check data, adjust scoring model if needed
- At day 90: generate correlation report, schedule case study interview
- Publish case study → begin paid conversion conversations

---

## Technical Debt Plan (v1 → v2)

| v1 Problem | v2 Solution |
|---|---|
| Single 7,695-line HTML file | Turborepo monorepo, strict separation of concerns |
| LocalStorage only | Supabase PostgreSQL with RLS |
| No user accounts | Supabase Auth + org management |
| API keys entered in browser | Edge Functions proxy; keys never in client |
| No team features | Full multi-tenancy via org_id on every table |
| No real-time data | Supabase Realtime for live dashboard updates |
| Manual question generation | Lesson library + adaptive selection algorithm |
| No mobile | Next.js responsive + PWA potential |

---

## MVP Success Criteria

### Technical (Week 12 checkpoint)
- [ ] Extension installed and working on ChatGPT
- [ ] Training loop: lesson → answer → LLM score → feedback → profile update
- [ ] Dashboard renders radar chart from real data
- [ ] Team analytics: correlation coefficient calculated from 500+ data points
- [ ] Supabase RLS: confirmed no cross-org data leakage (test with 2 separate org accounts)
- [ ] Chrome Web Store: approved and live

### Business (Month 6 checkpoint)
- [ ] 3 pilot companies onboarded (50+ users each)
- [ ] 2,000+ prompt logs collected with satisfaction ratings
- [ ] At least 1 correlation r > 0.5 demonstrated
- [ ] 1 case study drafted and approved by pilot company
- [ ] 1 paid conversion ($25+/user/month)
