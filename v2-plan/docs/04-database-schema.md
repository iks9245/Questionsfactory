# Database Schema — Questionsfactory 2.0

Full PostgreSQL DDL for Supabase. Run these migrations in order.

---

## Migration 001 — Core Tables

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- =====================================================
-- ORGANIZATIONS
-- =====================================================
CREATE TABLE organizations (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name            TEXT NOT NULL,
  slug            TEXT UNIQUE NOT NULL,          -- used in URLs
  plan            TEXT NOT NULL DEFAULT 'pilot'  -- pilot | team | enterprise
                  CHECK (plan IN ('pilot', 'team', 'enterprise')),
  max_members     INT NOT NULL DEFAULT 10,
  pilot_ends_at   TIMESTAMPTZ,                   -- null = not in pilot
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- USERS
-- =====================================================
-- Extends Supabase auth.users
CREATE TABLE users (
  id              UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  org_id          UUID REFERENCES organizations(id) ON DELETE SET NULL,
  email           TEXT NOT NULL,
  display_name    TEXT,
  role            TEXT NOT NULL DEFAULT 'member'
                  CHECK (role IN ('member', 'manager', 'admin')),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- TEAMS
-- =====================================================
CREATE TABLE teams (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name            TEXT NOT NULL,
  manager_id      UUID REFERENCES users(id) ON DELETE SET NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE team_members (
  team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (team_id, user_id)
);

-- =====================================================
-- ABILITY PROFILES
-- Current rolling average of each user's dimension scores
-- =====================================================
CREATE TABLE ability_profiles (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  org_id          UUID NOT NULL REFERENCES organizations(id),

  -- 6-dimension scores (rolling 30-day average)
  purpose_score   SMALLINT NOT NULL DEFAULT 50 CHECK (purpose_score BETWEEN 0 AND 100),
  context_score   SMALLINT NOT NULL DEFAULT 50 CHECK (context_score BETWEEN 0 AND 100),
  boundary_score  SMALLINT NOT NULL DEFAULT 50 CHECK (boundary_score BETWEEN 0 AND 100),
  depth_score     SMALLINT NOT NULL DEFAULT 50 CHECK (depth_score BETWEEN 0 AND 100),
  action_score    SMALLINT NOT NULL DEFAULT 50 CHECK (action_score BETWEEN 0 AND 100),
  assumption_score SMALLINT NOT NULL DEFAULT 50 CHECK (assumption_score BETWEEN 0 AND 100),

  overall_score   SMALLINT GENERATED ALWAYS AS (
    (purpose_score + context_score + boundary_score +
     depth_score + action_score + assumption_score) / 6
  ) STORED,

  total_sessions  INT NOT NULL DEFAULT 0,
  streak_days     INT NOT NULL DEFAULT 0,
  last_trained_at TIMESTAMPTZ,
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Migration 002 — Training System

```sql
-- =====================================================
-- LESSONS
-- Content library of training exercises
-- =====================================================
CREATE TABLE lessons (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id          UUID REFERENCES organizations(id),  -- NULL = global lesson
  title           TEXT NOT NULL,
  description     TEXT,
  target_dimension TEXT NOT NULL
                  CHECK (target_dimension IN (
                    'purpose','context','boundary','depth','action','assumption','overall'
                  )),
  difficulty      SMALLINT NOT NULL DEFAULT 1 CHECK (difficulty BETWEEN 1 AND 5),
  content         JSONB NOT NULL,  -- {scenario, question, rubric, example_answers}
  is_active       BOOLEAN NOT NULL DEFAULT TRUE,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- TRAINING SESSIONS
-- One session = one question attempt
-- =====================================================
CREATE TABLE training_sessions (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  org_id          UUID NOT NULL REFERENCES organizations(id),
  lesson_id       UUID REFERENCES lessons(id) ON DELETE SET NULL,

  user_answer     TEXT NOT NULL,  -- the prompt/question the user wrote

  -- Scores from LLM evaluation
  purpose_score   SMALLINT CHECK (purpose_score BETWEEN 0 AND 100),
  context_score   SMALLINT CHECK (context_score BETWEEN 0 AND 100),
  boundary_score  SMALLINT CHECK (boundary_score BETWEEN 0 AND 100),
  depth_score     SMALLINT CHECK (depth_score BETWEEN 0 AND 100),
  action_score    SMALLINT CHECK (action_score BETWEEN 0 AND 100),
  assumption_score SMALLINT CHECK (assumption_score BETWEEN 0 AND 100),
  overall_score   SMALLINT CHECK (overall_score BETWEEN 0 AND 100),

  feedback        TEXT,           -- LLM-generated coaching feedback
  confidence      REAL,           -- scorer's confidence 0.0-1.0

  scored_at       TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- MISTAKE RECORDS
-- Tracks patterns in low-scoring dimensions for adaptive training
-- =====================================================
CREATE TABLE mistake_records (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  session_id      UUID NOT NULL REFERENCES training_sessions(id) ON DELETE CASCADE,
  dimension       TEXT NOT NULL
                  CHECK (dimension IN ('purpose','context','boundary','depth','action','assumption')),
  score           SMALLINT NOT NULL,
  mistake_type    TEXT,           -- e.g. 'missing_context', 'vague_action'
  reviewed        BOOLEAN NOT NULL DEFAULT FALSE,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Migration 003 — Coach & Prompt Logs

```sql
-- =====================================================
-- COACH RECORDS
-- AI coach conversation history
-- =====================================================
CREATE TABLE coach_records (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  org_id          UUID NOT NULL REFERENCES organizations(id),
  role            TEXT NOT NULL CHECK (role IN ('user', 'assistant')),
  content         TEXT NOT NULL,
  related_session UUID REFERENCES training_sessions(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- PROMPT LOGS
-- Real-time data from Chrome Extension
-- CRITICAL: Raw prompt text is NEVER stored here.
--           Only metadata + scores + satisfaction rating.
-- =====================================================
CREATE TABLE prompt_logs (
  id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  org_id              UUID NOT NULL REFERENCES organizations(id),

  platform            TEXT NOT NULL
                      CHECK (platform IN ('chatgpt', 'claude', 'copilot', 'gemini', 'other')),
  prompt_length       INT NOT NULL,             -- character count only

  -- 6-dimension scores from rule-based engine (local scoring)
  purpose_score       SMALLINT CHECK (purpose_score BETWEEN 0 AND 100),
  context_score       SMALLINT CHECK (context_score BETWEEN 0 AND 100),
  boundary_score      SMALLINT CHECK (boundary_score BETWEEN 0 AND 100),
  depth_score         SMALLINT CHECK (depth_score BETWEEN 0 AND 100),
  action_score        SMALLINT CHECK (action_score BETWEEN 0 AND 100),
  assumption_score    SMALLINT CHECK (assumption_score BETWEEN 0 AND 100),
  overall_score       SMALLINT CHECK (overall_score BETWEEN 0 AND 100),

  -- User-rated satisfaction with AI response (captured 30s after response)
  satisfaction_rating SMALLINT CHECK (satisfaction_rating BETWEEN 1 AND 5),

  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Migration 004 — Analytics Views & Indexes

```sql
-- =====================================================
-- MATERIALIZED VIEW: Team Weekly Stats
-- Includes Pearson correlation coefficient calculation
-- =====================================================
CREATE MATERIALIZED VIEW team_weekly_stats AS
SELECT
  t.org_id,
  t.id AS team_id,
  t.name AS team_name,
  DATE_TRUNC('week', pl.created_at) AS week,
  COUNT(pl.id) AS prompt_count,
  AVG(pl.overall_score)::NUMERIC(5,2) AS avg_prompt_score,
  AVG(pl.satisfaction_rating)::NUMERIC(5,2) AS avg_satisfaction,
  CORR(pl.overall_score, pl.satisfaction_rating)::NUMERIC(5,4) AS correlation_r,
  COUNT(pl.id) FILTER (WHERE pl.satisfaction_rating IS NOT NULL) AS rated_count
FROM teams t
JOIN team_members tm ON tm.team_id = t.id
JOIN prompt_logs pl ON pl.user_id = tm.user_id
GROUP BY t.org_id, t.id, t.name, DATE_TRUNC('week', pl.created_at);

-- Refresh weekly (via pg_cron or Supabase scheduled function)
CREATE UNIQUE INDEX ON team_weekly_stats (team_id, week);

-- =====================================================
-- PERFORMANCE INDEXES
-- =====================================================
CREATE INDEX idx_prompt_logs_user_id ON prompt_logs (user_id);
CREATE INDEX idx_prompt_logs_org_id ON prompt_logs (org_id);
CREATE INDEX idx_prompt_logs_created_at ON prompt_logs (created_at DESC);
CREATE INDEX idx_prompt_logs_user_created ON prompt_logs (user_id, created_at DESC);

CREATE INDEX idx_training_sessions_user_id ON training_sessions (user_id);
CREATE INDEX idx_training_sessions_created_at ON training_sessions (created_at DESC);

CREATE INDEX idx_mistake_records_user_dim ON mistake_records (user_id, dimension);
```

---

## Migration 005 — Row Level Security

```sql
-- Enable RLS on all tables
ALTER TABLE organizations     ENABLE ROW LEVEL SECURITY;
ALTER TABLE users             ENABLE ROW LEVEL SECURITY;
ALTER TABLE teams             ENABLE ROW LEVEL SECURITY;
ALTER TABLE team_members      ENABLE ROW LEVEL SECURITY;
ALTER TABLE ability_profiles  ENABLE ROW LEVEL SECURITY;
ALTER TABLE lessons           ENABLE ROW LEVEL SECURITY;
ALTER TABLE training_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE mistake_records   ENABLE ROW LEVEL SECURITY;
ALTER TABLE coach_records     ENABLE ROW LEVEL SECURITY;
ALTER TABLE prompt_logs       ENABLE ROW LEVEL SECURITY;

-- =====================================================
-- ORGANIZATIONS: members can read their own org
-- =====================================================
CREATE POLICY "org_read_own" ON organizations
  FOR SELECT USING (
    id = (SELECT org_id FROM users WHERE id = auth.uid())
  );

-- =====================================================
-- USERS: can only see teammates (same org)
-- =====================================================
CREATE POLICY "users_org_isolation" ON users
  FOR ALL USING (
    org_id = (SELECT org_id FROM users WHERE id = auth.uid())
  );

-- =====================================================
-- TRAINING SESSIONS: see own, managers see team
-- =====================================================
CREATE POLICY "training_sessions_self" ON training_sessions
  FOR SELECT USING (user_id = auth.uid());

CREATE POLICY "training_sessions_manager" ON training_sessions
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM teams t
      JOIN team_members tm ON tm.team_id = t.id
      WHERE t.manager_id = auth.uid()
        AND tm.user_id = training_sessions.user_id
    )
  );

CREATE POLICY "training_sessions_insert_self" ON training_sessions
  FOR INSERT WITH CHECK (user_id = auth.uid());

-- =====================================================
-- PROMPT LOGS: self only (max privacy)
-- Managers see aggregated stats via materialized view, not raw logs
-- =====================================================
CREATE POLICY "prompt_logs_self_only" ON prompt_logs
  FOR ALL USING (user_id = auth.uid());

-- =====================================================
-- LESSONS: global lessons + org lessons
-- =====================================================
CREATE POLICY "lessons_global_and_org" ON lessons
  FOR SELECT USING (
    org_id IS NULL  -- global lessons
    OR org_id = (SELECT org_id FROM users WHERE id = auth.uid())
  );

-- =====================================================
-- ABILITY PROFILES: self + manager
-- =====================================================
CREATE POLICY "ability_profiles_self" ON ability_profiles
  FOR ALL USING (user_id = auth.uid());

CREATE POLICY "ability_profiles_manager" ON ability_profiles
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM teams t
      JOIN team_members tm ON tm.team_id = t.id
      WHERE t.manager_id = auth.uid()
        AND tm.user_id = ability_profiles.user_id
    )
  );
```

---

## Design Decisions

| Decision | Reason |
|---|---|
| `prompt_logs` does NOT store raw prompt text | Privacy-first design enables enterprise sales; reduces GDPR/CCPA liability |
| `confidence` field on training_sessions | Allows UI to show "estimated" vs "high-confidence" scores; important for LLM scoring |
| `org_id IS NULL` for global lessons | Allows platform-wide content library without requiring per-org duplication |
| `overall_score` as GENERATED ALWAYS on ability_profiles | Single source of truth; prevents score drift from manual updates |
| Materialized view with CORR() | PostgreSQL's built-in Pearson correlation; refreshed weekly for performance |
| `prompt_logs` self-only RLS | Managers only see aggregated team stats (via view), never individual raw logs |
