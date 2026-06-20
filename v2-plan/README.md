# Questionsfactory 2.0

> Train your team to ask better questions. Prove it works.

A B2B SaaS platform that makes AI prompt quality **measurable**, **trainable**, and **provably correlated** with workplace productivity.

---

## Vision

Enterprises pay $30-100/user/month for Copilot and ChatGPT Enterprise, then see disappointing ROI because their teams don't know how to prompt effectively. Questionsfactory 2.0 fixes this with a quantified training loop:

```
Prompt → Score (6 dimensions) → Train → Better Prompts → Better AI Output → Measurable ROI
```

## Three Core Differentiators

1. **Multi-dimension scoring** — Not just "is this a good prompt?" but WHY: Purpose / Context / Boundary / Depth / Action / Assumption, each scored 0-100
2. **Adaptive training** — Personalized exercises targeting each user's weakest dimensions
3. **Real-time workplace integration** — Chrome Extension scores prompts inside ChatGPT/Claude/Copilot as users type, with zero friction

No competitor has all three simultaneously.

---

## Documentation

| File | Contents |
|---|---|
| [CLAUDE.md](./CLAUDE.md) | Session guide for Claude Code — start here |
| [docs/01-market-research.md](./docs/01-market-research.md) | Market validation, competitor analysis |
| [docs/02-business-plan.md](./docs/02-business-plan.md) | Business model, GTM, pricing, ROI scenarios |
| [docs/03-architecture.md](./docs/03-architecture.md) | System architecture, component design |
| [docs/04-database-schema.md](./docs/04-database-schema.md) | Full PostgreSQL schema + RLS policies |
| [docs/05-roadmap.md](./docs/05-roadmap.md) | 90-day build plan, milestones |

---

## Tech Stack

Turborepo monorepo · Next.js 15 (App Router) · TypeScript · Tailwind + shadcn/ui · Supabase (Postgres + Auth + Realtime) · Chrome Extension MV3 · Vercel

---

## Status

Planning complete. Ready to build.

V1 (single-file HTML prototype): https://github.com/iks9245/Questionsfactory
