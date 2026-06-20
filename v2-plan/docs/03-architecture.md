# System Architecture — Questionsfactory 2.0

## High-Level Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    User's Browser                           │
│  ┌──────────────────────┐    ┌──────────────────────────┐   │
│  │   Chrome Extension   │    │    Next.js Web App       │   │
│  │                      │    │    (app.qfactory.ai)     │   │
│  │  Content Script      │    │                          │   │
│  │  ├─ Score overlay    │    │  ├─ Landing/pricing      │   │
│  │  ├─ Send intercept   │    │  ├─ Personal dashboard   │   │
│  │  └─ Satisfaction UI  │    │  ├─ Training module      │   │
│  │                      │    │  ├─ Team analytics       │   │
│  │  Background (SW)     │    │  └─ Coach interface      │   │
│  │  └─ Supabase sync    │    │                          │   │
│  └──────────┬───────────┘    └───────────┬──────────────┘   │
└─────────────│───────────────────────────│──────────────────┘
              │ (metadata only)            │ (all app data)
              ▼                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Supabase (hosted)                        │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Auth        │  │  PostgreSQL  │  │  Edge Functions  │  │
│  │              │  │  + RLS       │  │                  │  │
│  │  JWT tokens  │  │              │  │  score-prompt    │  │
│  │  OAuth2      │  │  org data    │  │  (LLM scoring)   │  │
│  │  SSO/SAML    │  │  user data   │  │                  │  │
│  │  (Enterprise)│  │  scores      │  │  generate-report │  │
│  └──────────────┘  │  logs        │  │  (weekly digest) │  │
│                    └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
              │
              ▼ (LLM API calls)
┌─────────────────────────┐
│  OpenAI / Anthropic API  │
│  (for precise scoring)   │
└─────────────────────────┘
```

---

## Next.js App Structure

### Route Groups

```
app/
├── (marketing)/              # Public, no auth, static/ISR
│   ├── page.tsx              # Landing page
│   ├── pricing/page.tsx      # Pricing table
│   └── layout.tsx            # Marketing nav + footer
│
├── (app)/                    # Authenticated, auth guard in layout
│   ├── layout.tsx            # Sidebar + auth check
│   ├── dashboard/
│   │   └── page.tsx          # Personal ability radar chart + streak
│   ├── training/
│   │   ├── page.tsx          # Training session list
│   │   └── [sessionId]/
│   │       └── page.tsx      # Active training session
│   ├── coach/
│   │   └── page.tsx          # AI coach conversation + mistake review
│   ├── team/
│   │   └── page.tsx          # Team leaderboard + manager view
│   └── analytics/
│       └── page.tsx          # Correlation charts + scorecard
│
└── api/
    ├── auth/[...supabase]/
    │   └── route.ts           # Supabase Auth callback
    └── og/
        └── route.tsx          # OG image generation for score cards
```

### Auth Guard Pattern

```typescript
// app/(app)/layout.tsx
import { createServerComponentClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'
import { redirect } from 'next/navigation'

export default async function AppLayout({ children }) {
  const supabase = createServerComponentClient({ cookies })
  const { data: { session } } = await supabase.auth.getSession()

  if (!session) redirect('/login')

  return (
    <div className="flex h-screen">
      <Sidebar />
      <main className="flex-1 overflow-auto">{children}</main>
    </div>
  )
}
```

---

## State Management

### Zustand (local UI state)

```typescript
// store/training.ts
interface TrainingStore {
  currentSession: TrainingSession | null
  currentQuestion: Question | null
  userAnswer: string
  isScoring: boolean
  setAnswer: (answer: string) => void
  submitAnswer: () => Promise<void>
}
```

### TanStack Query (server state)

```typescript
// hooks/useAbilityProfile.ts
export function useAbilityProfile(userId: string) {
  return useQuery({
    queryKey: ['ability-profile', userId],
    queryFn: () => supabase
      .from('ability_profiles')
      .select('*')
      .eq('user_id', userId)
      .single()
      .then(r => r.data),
    staleTime: 5 * 60 * 1000,  // 5 min cache
  })
}
```

---

## Chrome Extension Architecture

### File Structure

```
apps/extension/
├── manifest.json
├── src/
│   ├── background/
│   │   └── service-worker.ts    # Auth state, Supabase sync queue
│   ├── content/
│   │   ├── chatgpt.ts           # ChatGPT-specific selectors
│   │   ├── claude.ts            # Claude.ai-specific selectors
│   │   ├── copilot.ts           # GitHub Copilot Chat selectors
│   │   ├── gemini.ts            # Gemini-specific selectors
│   │   ├── scorer.ts            # Calls packages/scoring rule-based engine
│   │   ├── overlay.tsx          # Score badge React component
│   │   └── satisfaction.tsx     # 5-star rating modal React component
│   ├── popup/
│   │   └── Popup.tsx            # Extension popup (quick stats)
│   └── options/
│       └── Options.tsx          # Settings: toggle platforms, auth
├── vite.config.ts
└── package.json
```

### Content Script User Flow

```
User opens ChatGPT
    │
    ▼
Content script injected (chatgpt.ts)
    │
    ├── MutationObserver watches for textarea
    │
    ▼
User types a prompt
    │
    ▼
Send button clicked → intercept event
    │
    ├── Extract textarea text
    ├── Run rule-based scorer locally (< 10ms)
    ├── Show score overlay badge
    └── Allow original send to proceed
         │
         ▼
    AI responds (30 second delay)
         │
         ▼
    Show satisfaction modal (non-blocking)
         │
         ├── User rates 1-5 stars
         └── Background SW syncs {scores, rating, metadata} to Supabase
              (NO raw prompt text)
```

### manifest.json (key parts)

```json
{
  "manifest_version": 3,
  "name": "Questionsfactory",
  "permissions": ["storage", "activeTab"],
  "host_permissions": [
    "https://chat.openai.com/*",
    "https://claude.ai/*",
    "https://github.com/*",
    "https://gemini.google.com/*"
  ],
  "content_scripts": [
    {
      "matches": ["https://chat.openai.com/*"],
      "js": ["content/chatgpt.js"],
      "run_at": "document_idle"
    }
  ],
  "background": {
    "service_worker": "background/service-worker.js",
    "type": "module"
  }
}
```

---

## Scoring Engine Package

```typescript
// packages/scoring/src/rule-based.ts

const DIMENSION_PATTERNS = {
  purpose: {
    positive: [/I want to/i, /please help me/i, /my goal is/i, /I need to/i],
    negative: [/^(write|make|do|fix)/i]  // starts with bare imperative
  },
  context: {
    positive: [/I'm working on/i, /the context is/i, /background:/i, /currently/i],
    negative: []
  },
  // ... other dimensions
}

export function scoreRuleBased(prompt: string): ScoringResult {
  const words = prompt.split(/\s+/).length
  const sentences = prompt.split(/[.!?]+/).length

  const dimensions: DimensionScores = {
    purpose: scoreDimension(prompt, 'purpose'),
    context: scoreDimension(prompt, 'context'),
    boundary: scoreDimension(prompt, 'boundary'),
    depth: scoreDimension(prompt, 'depth'),
    action: scoreDimension(prompt, 'action'),
    assumption: scoreDimension(prompt, 'assumption'),
  }

  const overall = Math.round(
    Object.values(dimensions).reduce((a, b) => a + b, 0) / 6
  )

  const weakest = Object.entries(dimensions)
    .filter(([, score]) => score < 60)
    .sort(([, a], [, b]) => a - b)
    .slice(0, 2)
    .map(([dim]) => dim)

  return {
    dimensions,
    overall,
    weakest,
    suggestions: weakest.map(dim => SUGGESTIONS[dim]),
    mode: 'rule-based',
    confidence: 0.6,
  }
}
```

---

## Supabase Edge Functions

| Function | Trigger | Purpose |
|---|---|---|
| `score-prompt` | HTTP POST | LLM-backed precise scoring for training sessions |
| `generate-report` | Cron (weekly) | Generate team weekly digest + correlation stats |
| `process-training-answer` | HTTP POST | Evaluate user's training answer, update ability profile |

### score-prompt function

```typescript
// supabase/functions/score-prompt/index.ts
import Anthropic from 'npm:@anthropic-ai/sdk'

const client = new Anthropic()

Deno.serve(async (req) => {
  const { prompt, userId, sessionId } = await req.json()

  const response = await client.messages.create({
    model: 'claude-haiku-4-5-20251001',  // fast + cheap for scoring
    max_tokens: 1024,
    system: SCORING_SYSTEM_PROMPT,  // 6-dimension rubric
    messages: [{ role: 'user', content: prompt }],
  })

  const scores = parseScoresFromResponse(response.content[0].text)

  // Store in DB
  await supabase.from('training_sessions').update({
    scores,
    scored_at: new Date().toISOString(),
  }).eq('id', sessionId)

  return Response.json(scores)
})
```

---

## Security Model

1. **No raw prompt text stored** — Content script scores locally; only `DimensionScores + prompt_length + platform` sent to server
2. **Row Level Security** — Every table has RLS policies; users can only read their org's data
3. **JWT validation** — Edge Functions verify Supabase JWT before processing
4. **Content Security Policy** — Extension manifest v3 enforces strict CSP; no inline scripts

---

## Turbo Configuration

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["build"]
    },
    "lint": {}
  }
}
```

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```
