# Business Plan — Questionsfactory 2.0

## One-Line Pitch

We help companies prove their AI tools are working — by measuring and improving how their employees prompt.

---

## Problem → Solution

**Problem**: Enterprises spend $30-100/user/month on AI tools (Copilot, ChatGPT Enterprise) and see uneven adoption and hard-to-measure ROI.

**Root cause**: Prompt quality varies wildly by individual. Nobody tracks it. Nobody trains it systematically.

**Solution**: Questionsfactory 2.0 measures prompt quality across 6 dimensions, trains employees adaptively, captures real-time data via a Chrome Extension, and produces a correlation report proving that better prompts → better AI output → better work outcomes.

---

## Target Customer

**Ideal Customer Profile (ICP)**:
- Tech company, SaaS company, or consulting firm
- 50–300 employees
- Already subscribed to Copilot for Microsoft 365 OR ChatGPT Enterprise (signals budget and commitment)
- Has an L&D manager, HR director, or VP Engineering who cares about AI ROI
- Located in US, UK, Canada, or Singapore (English-first, high AI spend)

**Buyer persona**:
- **L&D Manager / HR Director**: Wants to show leadership that AI training is working. Needs a measurable outcome to report.
- **CTO / VP Engineering**: Wants engineering teams to use Copilot effectively. Sees prompt quality as a leverage point.
- **CFO** (secondary): Needs to justify AI tool spend at renewal time.

---

## Revenue Model

### Pricing Tiers

| Tier | Price | Target | What's included |
|---|---|---|---|
| **Free Pilot** | $0 for 3 months | First 3-5 enterprise customers | Full platform + Chrome Extension, in exchange for anonymized data partnership |
| **Team** | $25/user/month | 10-99 users | Training platform + Chrome Extension + team analytics |
| **Enterprise** | $40/user/month | 100+ users | All Team features + custom scoring rubric + dedicated CSM + quarterly correlation report + SSO/SAML |

### Unit Economics (Team tier, 50-person company)

```
Monthly revenue:     50 × $25 = $1,250/month
Annual revenue:      $15,000/year
Gross margin:        ~80% (SaaS + Supabase hosting costs ~$250/month)
Customer LTV:        2-year avg → $30,000
CAC target:          < $3,000 (need 3-5 sales conversations)
Payback period:      < 3 months
```

---

## Go-to-Market Strategy

### Phase 1: Free Pilot → Case Study (Months 1-6)

1. Recruit 3-5 pilot companies via personal network, LinkedIn outreach, ProductHunt launch
2. Offer full platform free for 3 months in exchange for:
   - Anonymized prompt quality data (statistical only, no raw text)
   - Access to HR contact for a case study interview
   - Permission to use their logo on the website
3. At month 3: measure correlation between average prompt scores and user-reported AI satisfaction
4. Publish case study: "Company X increased average prompt score 31% in 90 days, AI satisfaction rating went from 3.1 to 4.2/5"

### Phase 2: Paid Conversion + Outbound (Months 6-12)

1. Convert pilot customers to Team or Enterprise tier
2. Use case study as the primary sales asset
3. Outbound: LinkedIn Sales Navigator targeting L&D Managers at Copilot/ChatGPT-paying companies
4. Content marketing: "How to measure AI ROI" → capture L&D decision-maker emails
5. ProductHunt launch after first case study is live

### Phase 3: Channel (Month 12+)

1. Microsoft Partner Network — co-sell alongside Copilot renewals
2. HR tech integrations — integrate into Workday Learning, SAP SuccessFactors
3. Reseller agreements with consulting firms (McKinsey Digital, Accenture AI)

---

## Application Scenarios (B2B Sales Evidence)

### Scenario A: Software Development Team

**Setup**: 30-person engineering team using GitHub Copilot. Manager complains engineers aren't getting good code suggestions.

**With Questionsfactory**:
- Week 1: Baseline assessment. Average dimension scores: Purpose 72, Context 45, Boundary 38 (context and boundary are weak)
- Week 4: After targeted training on context + boundary dimensions, scores rise to Context 68, Boundary 61
- Week 8: Team reports GitHub Copilot suggestions are "much more relevant." Satisfaction up from 2.8 to 4.1/5
- Outcome: Manager has data to justify Copilot renewal AND promote the L&D initiative internally

**Quantified ROI**: If 30 engineers each save 30 min/day from better AI output = 15 person-hours/day = $900/day at $60/hour. $27,000/month ROI on $750/month training cost. **36x ROI**.

### Scenario B: Marketing Team

**Setup**: 15-person marketing team using ChatGPT for copywriting. Output quality is inconsistent; some writers love it, others hate it.

**With Questionsfactory**:
- Discover: high performers have Purpose + Action scores averaging 82+; low performers average 51
- Train low performers specifically on Purpose (clear goal statement) and Action (specify desired output format)
- 6 weeks later: output consistency improves, team lead reports 40% less revision required

**Quantified ROI**: 40% less revision on content = 2 hours/week/person saved = 30 hours/week for 15 people. At $50/hour = $1,500/week = $6,000/month. **8x ROI** on $375/month team plan.

### Scenario C: Consulting Firm

**Setup**: Management consulting firm, 100 consultants using Claude for research and analysis. Partners concerned about quality variation between junior and senior consultants.

**With Questionsfactory**:
- Use as onboarding tool: new consultants must complete Questionsfactory certification before solo AI use
- Track score progression as KPI in quarterly performance reviews
- Generate team scorecard for leadership: "Q3: team average score improved from 61 to 79/100"
- Identify top 10% performers as internal "Prompt Champions" → peer training program

**Quantified ROI**: Faster onboarding (2 weeks faster → $5,000/consultant value). Reduced partner review time. Scorecard as leadership reporting tool.

---

## Correlation Proof — The Core Sales Mechanism

The MVP must prove this relationship: **higher prompt quality score → higher AI satisfaction rating**.

### Data Capture Design

Every time a user submits a prompt via the Chrome Extension, we capture:

```json
{
  "id": "uuid",
  "user_id": "uuid",
  "org_id": "uuid",
  "platform": "chatgpt",
  "scores": {
    "purpose": 78,
    "context": 45,
    "boundary": 60,
    "depth": 72,
    "action": 83,
    "assumption": 55
  },
  "overall_score": 66,
  "prompt_length": 347,
  "satisfaction_rating": null,
  "created_at": "2026-06-19T10:23:00Z"
}
```

30 seconds after the AI responds, a non-blocking modal asks: "How helpful was this AI response?" (1-5 stars). If rated, `satisfaction_rating` is updated.

### Success Criteria

| Metric | Minimum (MVP viable) | Target (sales-ready) |
|---|---|---|
| Correlation coefficient (r) | > 0.5 | > 0.7 |
| Data points per customer | 500+ | 2,000+ |
| Training period | 90 days | 60 days |
| Score improvement | > 15% | > 25% |
| Satisfaction improvement | > 0.5/5 | > 1.0/5 |

If we hit r > 0.7 with n > 500 in a 90-day pilot, we have a publishable proof of concept and a compelling sales story.

---

## MVP Success Metrics

**Technical**:
- Chrome Extension installs and scores prompts with < 100ms latency
- Zero raw prompt text ever leaves the user's browser
- Dashboard loads in < 2 seconds (P95)

**Business**:
- 3 paid pilot customers completed and converted
- At least 1 case study published
- Average pilot correlation r > 0.5

**Product**:
- DAU/MAU > 40% (sticky daily habit)
- Extension enabled for > 70% of active users
- NPS > 40 (promoter score)
