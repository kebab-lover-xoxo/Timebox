# Database Architecture

**Google Calendar-Inspired · Monte Carlo Chore Scheduler · Pomodoro Canvas · Radial Multi-Alarm · Duration Forecasting · Vim/Notion Editor**

-----

## Design Pillars

Five systems, parallel, feeding each other:

```
CALENDAR CORE        CHORE SCHEDULER       POMODORO ENGINE      RADIAL CANVAS        EDITOR ENGINE
─────────────        ───────────────       ───────────────      ─────────────        ─────────────
calendars            chore_definitions     pomodoro_sessions    canvas_sessions      editor_sessions
events               chore_n_history       task_residuals       canvas_items         editor_events
recurrence_          schedule_runs         duration_profiles    canvas_alarms        keystroke_segments
  exceptions         chore_occurrences     chronobiology_       canvas_position_     slash_commands
event_acl            schedule_healing_log    profiles             history             concept_analyses
attention_classes                          session_feedback
event_memory                               residual_prompts
```

-----

## Shared Enums

```sql
CREATE TYPE calendar_color AS ENUM (
  'slate',      -- default
  'rose',       -- meeting
  'amber',      -- task
  'violet',     -- personal
  'emerald',    -- chore
  'sky',        -- homework
  'stone',      -- passive
  'orange'      -- physical
);
-- 7 pastel options + slate default
-- Each color corresponds to an event type by convention, not constraint
-- color NULL means this is the default calendar

CREATE TYPE event_type AS ENUM (
  'meeting',
  'task',
  'personal',
  'chore',          -- renamed from chore_occurrence
  'homework',
  'passive',
  'physical'
);
-- Event type drives default color assignment in application layer
-- No FK constraint — color is a display preference, not enforced by DB

CREATE TYPE attention_class_value AS ENUM (
  'active',    -- requires sustained focus, pomodoro applicable
  'involved',  -- requires presence, no multitasking, no pomodoro (meetings, cooking)
  'passive'    -- awareness only, check-in alarms only (laundry, oven)
);

CREATE TYPE canvas_event_type AS ENUM (
  'passive_multi',   -- n timers, no pomodoro (laundry + oven simultaneously)
  'focus_passive',   -- pomodoro block + passive item(s) running concurrently
  'focus_only',      -- pure pomodoro, no concurrent passive items
  'involved_only'    -- presence required, no pomodoro, check-in alarms only
);
-- canvas_event_type is SET AUTOMATICALLY by the system — see Canvas Event Type section below
```

-----

## Attention Classes

```sql
-- Attention is its own table — extensible, documented, not just an enum value
-- Drives: canvas_event_type assignment, alarm class defaults,
--         residual behavior, pomodoro applicability

CREATE TABLE attention_classes (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  value                 attention_class_value UNIQUE NOT NULL,
  label                 TEXT NOT NULL,
  description           TEXT NOT NULL,
  pomodoro_applicable   BOOLEAN NOT NULL,   -- active=TRUE, involved=FALSE, passive=FALSE
  residual_applicable   BOOLEAN NOT NULL,   -- active=TRUE, involved=FALSE, passive=FALSE
  delay_on_no_complete  BOOLEAN NOT NULL,   -- involved=TRUE (presence implied), others=FALSE
  default_r             NUMERIC(5,4) NOT NULL,  -- default radial position on canvas
  default_alarm_class   TEXT NOT NULL,      -- which alarm_class fires by default
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Seed data (inserted at migration time, not application layer)
INSERT INTO attention_classes VALUES
  (gen_uuid_v7(), 'active',   'Active',   'Sustained focus required. Pomodoro applicable.',
   TRUE,  TRUE,  FALSE, 0.2, 'focus_checkpoint'),
  (gen_uuid_v7(), 'involved', 'Involved', 'Presence required, no multitask. No pomodoro.',
   FALSE, FALSE, TRUE,  0.3, 'passive_check'),
  (gen_uuid_v7(), 'passive',  'Passive',  'Awareness only. Check-in alarms only.',
   FALSE, FALSE, FALSE, 0.8, 'passive_check');
```

-----

## Event Memory

```sql
-- When the same event is created again (matched by normalized title hash + event_type),
-- the system prefills attention_class, canvas_event_type, estimated_minutes,
-- and duration_profile_id from the most recent matching memory entry.
-- User can override. Memory updates on each completion.

CREATE TABLE event_memory (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id               UUID NOT NULL REFERENCES users(id),
  title_hash            TEXT NOT NULL,          -- SHA-256 of lowercase trimmed title
  event_type            event_type NOT NULL,
  attention_class       attention_class_value NOT NULL,
  canvas_event_type     canvas_event_type,
  estimated_minutes     INT,
  duration_profile_id   UUID REFERENCES duration_profiles(id),
  last_seen_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  occurrence_count      INT NOT NULL DEFAULT 1,
  UNIQUE (user_id, title_hash, event_type)
);

CREATE INDEX idx_event_memory_user ON event_memory (user_id, title_hash);
```

-----

## Calendar Core

### calendars

```sql
CREATE TABLE calendars (
  id          UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id     UUID NOT NULL REFERENCES users(id),
  name        TEXT NOT NULL,
  color       calendar_color,                    -- NULL = this is the default calendar
  is_visible  BOOLEAN NOT NULL DEFAULT TRUE,
  -- is_default removed: NULL color implies default
  -- is_visible retained: user may hide a calendar without deleting it
  --   justified for: archived calendars, seasonal calendars (school year),
  --   shared-view calendars the user wants hidden temporarily
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at  TIMESTAMPTZ
);

CREATE UNIQUE INDEX idx_calendars_default_per_user
  ON calendars (user_id) WHERE color IS NULL AND deleted_at IS NULL;
-- Only one default calendar per user enforced at DB level

CREATE INDEX idx_calendars_user ON calendars (user_id) WHERE deleted_at IS NULL;
```

> **`is_visible` justification:** A user may have multiple calendars — a work calendar, a school calendar, a chore calendar. Hiding a calendar removes it from the view without deleting it. A seasonal calendar (active only during semester) stays in the system but hidden in summer. An archived calendar (completed project) remains queryable for history. Deletion is destructive; visibility is a display preference.

-----

### events

```sql
CREATE TYPE event_status AS ENUM (
  'scheduled', 'in_progress', 'completed', 'skipped', 'cancelled'
);

CREATE TYPE recurrence_freq AS ENUM ('daily', 'weekly', 'monthly', 'custom');

CREATE TABLE events (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  calendar_id           UUID NOT NULL REFERENCES calendars(id),
  user_id               UUID NOT NULL REFERENCES users(id),
  event_type            event_type NOT NULL,
  attention_class       attention_class_value NOT NULL DEFAULT 'active',
  canvas_event_type     canvas_event_type,       -- set automatically, see Canvas section
  title                 TEXT NOT NULL,
  description           TEXT,
  start_at              TIMESTAMPTZ NOT NULL,
  end_at                TIMESTAMPTZ NOT NULL,
  is_all_day            BOOLEAN NOT NULL DEFAULT FALSE,
  status                event_status NOT NULL DEFAULT 'scheduled',
  location              TEXT,
  -- metadata JSONB removed — all structured data is in typed columns

  -- Recurrence (Google-style sparse — logic trusted as sound)
  is_recurring          BOOLEAN NOT NULL DEFAULT FALSE,
  recurrence_freq       recurrence_freq,
  recurrence_rule       JSONB,                   -- iCal RRULE, Google-compatible subset
  recurrence_end        TIMESTAMPTZ,
  master_event_id       UUID REFERENCES events(id),
  original_start_at     TIMESTAMPTZ,

  -- Duration
  estimated_minutes     INT,
  actual_minutes        INT,
  meaningful_minutes    INT,
  duration_profile_id   UUID REFERENCES duration_profiles(id),

  -- Residual chain
  residual_of           UUID REFERENCES events(id),
  residual_sequence     INT NOT NULL DEFAULT 0,

  -- Cross-system
  chore_occurrence_id   UUID REFERENCES chore_occurrences(id),
  gantt_task_id         UUID REFERENCES gantt_tasks(id),
  ai_session_id         UUID REFERENCES ai_sessions(id),
  editor_session_id     UUID REFERENCES editor_sessions(id),

  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at            TIMESTAMPTZ,

  CONSTRAINT chk_end_after_start CHECK (end_at > start_at),
  CONSTRAINT chk_exception_has_master CHECK (
    original_start_at IS NULL OR master_event_id IS NOT NULL
  )
);

CREATE INDEX idx_events_calendar_range
  ON events (calendar_id, start_at, end_at) WHERE deleted_at IS NULL;
CREATE INDEX idx_events_master
  ON events (master_event_id) WHERE master_event_id IS NOT NULL AND deleted_at IS NULL;
CREATE UNIQUE INDEX idx_events_exception_key
  ON events (master_event_id, original_start_at) WHERE original_start_at IS NOT NULL;
CREATE INDEX idx_events_residual_chain
  ON events (residual_of) WHERE residual_of IS NOT NULL;
CREATE INDEX idx_events_overlap
  ON events (user_id, start_at, end_at) WHERE deleted_at IS NULL;
-- overlap index used by canvas_event_type auto-assignment
```

-----

### recurrence_exceptions

```sql
-- Google recurrence logic trusted as sound. Schema unchanged.
CREATE TYPE exception_type AS ENUM ('cancelled', 'moved', 'modified');

CREATE TABLE recurrence_exceptions (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  master_event_id       UUID NOT NULL REFERENCES events(id),
  original_start_at     TIMESTAMPTZ NOT NULL,
  exception_type        exception_type NOT NULL,
  replacement_event_id  UUID REFERENCES events(id),
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (master_event_id, original_start_at)
);
```

-----

### event_acl

```sql
-- Retained. Argument: the AI agent is a legitimate grantee.
-- The scheduling agent, timebox agent, and conflict resolver all need
-- read access to the user's calendar to function.
-- grantee_type discriminates human users from agent identities.
-- No human-to-human sharing is implemented — agent access only for now.
-- Future: household mode, coach/client, parent/child — ACL is the right primitive.

CREATE TYPE acl_role       AS ENUM ('owner', 'writer', 'reader');
CREATE TYPE acl_scope      AS ENUM ('user', 'agent', 'domain', 'public');
CREATE TYPE acl_grantee_type AS ENUM ('user', 'agent');

CREATE TABLE event_acl (
  id              UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  resource_id     UUID NOT NULL,
  resource_type   TEXT NOT NULL CHECK (resource_type IN ('calendar', 'event')),
  scope           acl_scope NOT NULL,
  role            acl_role NOT NULL,
  grantee_type    acl_grantee_type NOT NULL DEFAULT 'agent',
  grantee_id      UUID REFERENCES users(id),   -- NULL for agent scope
  agent_id        TEXT,                         -- agent identifier string
  granted_by      UUID NOT NULL REFERENCES users(id),
  granted_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at      TIMESTAMPTZ,
  CONSTRAINT chk_grantee_xor_agent CHECK (
    (grantee_type = 'user'  AND grantee_id IS NOT NULL AND agent_id IS NULL) OR
    (grantee_type = 'agent' AND agent_id IS NOT NULL AND grantee_id IS NULL)
  )
);

CREATE INDEX idx_acl_resource ON event_acl (resource_id, resource_type);
CREATE INDEX idx_acl_agent    ON event_acl (agent_id) WHERE agent_id IS NOT NULL;
```

-----

## Canvas Event Type — Documentation

### What it is

`canvas_event_type` describes how an event behaves on the radial canvas. It controls:

- Which alarm classes are available
- Whether pomodoro tracking applies
- Whether focus checkpoints fire
- How the system responds to no-completion

### Values

|Value          |Behavior                                          |Alarm classes available                                      |Pomodoro|
|---------------|--------------------------------------------------|-------------------------------------------------------------|--------|
|`passive_multi`|n concurrent timers, no focus (laundry + oven)    |`passive_check`, `custom`                                    |No      |
|`focus_passive`|pomodoro block + passive item(s) running alongside|all                                                          |Yes     |
|`focus_only`   |pure pomodoro, nothing concurrent                 |`focus_checkpoint`, `break_signal`, `refresh_start`, `custom`|Yes     |
|`involved_only`|presence required, no multitask, no pomodoro      |`passive_check`, `break_signal`, `custom`                    |No      |

### User Journey by Type

**`passive_multi` — “Laundry + Oven”**

```
User places laundry on canvas → system reads attention_class=passive → assigns passive_multi
User adds oven item → same session, same type
User drags alarm handles on each item independently on the timeline strip
Items sit at r=0.8 (edge, low urgency)
Alarm fires: "Check laundry" → user taps acknowledge → item stays active
Alarm fires: "Check oven" → user taps → dismissed when done
No pomodoro. No completion flag. No residual.
Canvas session closes when all items dismissed.
```

**`focus_only` — “Homework A”**

```
User creates homework event → system reads attention_class=active → assigns focus_only
Event placed at r=0.2 (near center, high attention)
Pomodoro session starts
Focus checkpoints fire at 25min marks — timer does NOT stop
  → if session exceeds 40min: break = 50% of logged time
  → if session under 40min: break = 5min
Session ends → system prompts: "Done?" → user confirms or sets remaining
If not done: residual_prompt created, user confirms remaining minutes
Residual scheduled into next optimal slot avoiding dead zones
```

**`focus_passive` — “Homework while laundry runs”**

```
Two events overlap in time window → system detects overlap
One is active (homework), one is passive (laundry)
System auto-assigns focus_passive to the canvas item grouping both
Homework pomodoro runs normally
Laundry passive_check alarms fire independently alongside
User sees both on canvas simultaneously
```

**`involved_only` — “Meeting” or “Cooking dinner”**

```
User creates meeting → attention_class=involved → involved_only assigned
Placed at r=0.3 (present but not deep focus)
No pomodoro. Check-in alarm fires at intervals.
No focus checkpoints. No residual.
If session ends without completion logged:
  System prompts: "Did this finish?" → user confirms
  If no response within timeout: flagged for manual review
  Delay recorded but no residual chain started
```

### Automatic Assignment Rules

```
canvas_event_type is NEVER set by the user directly.
Set automatically by the system at event creation and on overlap detection.

Rule 1 — Single event, no overlap:
  attention_class = active   → focus_only
  attention_class = involved → involved_only
  attention_class = passive  → passive_multi

Rule 2 — Overlap detected (two events share a time window for same user):
  active + passive   → focus_passive (active event drives type)
  active + involved  → focus_only + involved_only (two separate canvas items)
  passive + passive  → passive_multi (merged into one canvas item group)
  involved + passive → involved_only (passive alarms run alongside)
  active + active    → two focus_only items (conflict flagged to user)

Rule 3 — Overlap detection query:
  SELECT * FROM events
  WHERE user_id = $1
    AND deleted_at IS NULL
    AND start_at < $end_at
    AND end_at > $start_at
  -- Runs on every event INSERT and UPDATE

Rule 4 — Event memory override:
  If event_memory has a prior canvas_event_type for this title_hash,
  use it as the starting value before overlap rules are applied.
  Overlap rules always win over memory.
```

-----

## Chore Scheduler

### chore_definitions

```sql
CREATE TABLE chore_definitions (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id               UUID NOT NULL REFERENCES users(id),
  name                  TEXT NOT NULL,
  estimated_minutes     INT NOT NULL CHECK (estimated_minutes > 0),
  priority              INT NOT NULL DEFAULT 3 CHECK (priority BETWEEN 1 AND 5),
  attention_class       attention_class_value NOT NULL DEFAULT 'active',
  color                 calendar_color,

  -- Frequency: every nth day — living value, silently adjusted
  n_original            INT NOT NULL CHECK (n_original >= 1),
  n_current             INT NOT NULL CHECK (n_current >= 1),
  n_min                 INT NOT NULL DEFAULT 1,
  n_max                 INT NOT NULL DEFAULT 30,

  preferred_days        INT[] DEFAULT '{}',
  preferred_time_start  TIME,
  preferred_time_end    TIME,
  avoid_days            INT[] DEFAULT '{}',

  last_completed_at     TIMESTAMPTZ,
  next_due_at           TIMESTAMPTZ,
  is_active             BOOLEAN NOT NULL DEFAULT TRUE,
  mc_weight             NUMERIC(6,4) NOT NULL DEFAULT 1.0,

  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at            TIMESTAMPTZ
);

CREATE INDEX idx_chore_def_user
  ON chore_definitions (user_id) WHERE deleted_at IS NULL AND is_active = TRUE;
```

-----

### chore_n_history

```sql
CREATE TYPE n_adjustment_reason AS ENUM (
  'cycle_success',
  'cycle_drift',
  'manual_override',
  'healing_correction',
  'batch_rebalance'     -- new: post-optimization batch improvement pass
);

CREATE TABLE chore_n_history (
  id              UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  chore_id        UUID NOT NULL REFERENCES chore_definitions(id),
  user_id         UUID NOT NULL REFERENCES users(id),
  n_before        INT NOT NULL,
  n_after         INT NOT NULL,
  reason          n_adjustment_reason NOT NULL,
  schedule_run_id UUID REFERENCES schedule_runs(id),
  recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
  -- append-only
);

CREATE INDEX idx_n_history_chore ON chore_n_history (chore_id, recorded_at DESC);
```

-----

### schedule_runs

```sql
CREATE TYPE schedule_run_type AS ENUM (
  'initialization',
  'periodic',
  'healing',
  'manual',
  'batch_rebalance'   -- post-optimization improvement pass
);

CREATE TYPE schedule_run_status AS ENUM (
  'pending', 'running', 'completed', 'failed', 'superseded'
);

CREATE TABLE schedule_runs (
  id                      UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id                 UUID NOT NULL REFERENCES users(id),
  run_type                schedule_run_type NOT NULL,
  status                  schedule_run_status NOT NULL DEFAULT 'pending',
  window_start            TIMESTAMPTZ NOT NULL,
  window_end              TIMESTAMPTZ NOT NULL,
  window_days             INT NOT NULL DEFAULT 49,
  iterations              INT NOT NULL DEFAULT 50000,
  seed                    BIGINT,
  score                   NUMERIC(8,4),

  -- Distribution quality metrics (used by batch rebalance)
  chores_scheduled        INT,
  mean_daily_load         NUMERIC(6,2),   -- avg chores per day
  load_variance           NUMERIC(8,4),   -- variance of daily load — lower = better balanced
  overloaded_days         INT,            -- days with t+1 or more chores vs mean
  underloaded_days        INT,            -- days with t-1 or fewer chores vs mean
  -- batch_rebalance runs reduce load_variance without full rerun

  parameters              JSONB NOT NULL DEFAULT '{}',
  prior_run_id            UUID REFERENCES schedule_runs(id),
  cycle_completion_rate   NUMERIC(5,4),
  n_adjustments_made      INT NOT NULL DEFAULT 0,
  error_message           TEXT,
  created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at            TIMESTAMPTZ
);

CREATE INDEX idx_schedule_runs_user ON schedule_runs (user_id, created_at DESC);
```

### Chore Distribution Self-Correction — Batch Rebalance

```
After initialization, the MC run has found the optimal distribution.
But optimal by score does not guarantee even load across days.
Some days may have t+1 chores while others have t-1.

Batch rebalance runs AFTER initialization as a separate lightweight pass:

1. Compute daily load from chore_occurrences: count per day across window
2. Compute mean_daily_load t = total_chores / window_days
3. Identify overloaded days (load > t) and underloaded days (load < t)
4. For each overloaded day: find lowest-priority chore whose n_current
   allows a 1-day shift without violating frequency constraint
5. Move it to nearest underloaded day
6. Repeat in batch until load_variance < threshold or no valid moves remain
7. Record as schedule_run_type='batch_rebalance', referencing prior run
8. All moved occurrences updated, chore_n_history logged with reason='batch_rebalance'

Constraints:
  - Never violate n_min or n_max bounds
  - Never move a chore onto an avoid_day
  - Prefer moves that respect preferred_days
  - Max iterations: O(n_chores) — bounded (NASA rule 2)
  - Runs as Celery task, async, does not block calendar
```

-----

### chore_occurrences

```sql
CREATE TYPE occurrence_status AS ENUM (
  'proposed', 'scheduled', 'in_progress', 'completed',
  'missed', 'healed', 'cancelled'
);

CREATE TABLE chore_occurrences (
  id                  UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  schedule_run_id     UUID NOT NULL REFERENCES schedule_runs(id),
  chore_id            UUID NOT NULL REFERENCES chore_definitions(id),
  user_id             UUID NOT NULL REFERENCES users(id),
  proposed_start_at   TIMESTAMPTZ NOT NULL,
  proposed_end_at     TIMESTAMPTZ NOT NULL,
  confidence_score    NUMERIC(5,4) NOT NULL,
  load_score          NUMERIC(5,4) NOT NULL,
  status              occurrence_status NOT NULL DEFAULT 'proposed',
  event_id            UUID REFERENCES events(id),
  healed_from_id      UUID REFERENCES chore_occurrences(id),
  heal_sequence       INT NOT NULL DEFAULT 0,
  completed_at        TIMESTAMPTZ,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_occurrences_chore  ON chore_occurrences (chore_id, proposed_start_at);
CREATE INDEX idx_occurrences_missed ON chore_occurrences (user_id, status)
  WHERE status IN ('missed', 'healed');
```

-----

### schedule_healing_log

```sql
CREATE TYPE heal_trigger AS ENUM (
  'auto_drift_threshold',
  'cycle_end_repair',
  'manual_user_request'
);

CREATE TABLE schedule_healing_log (
  id                  UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id             UUID NOT NULL REFERENCES users(id),
  schedule_run_id     UUID NOT NULL REFERENCES schedule_runs(id),
  trigger             heal_trigger NOT NULL,
  occurrences_missed  INT NOT NULL,
  occurrences_healed  INT NOT NULL,
  n_adjustments       INT NOT NULL DEFAULT 0,
  drift_rate          NUMERIC(5,4),
  healed_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

-----

## Pomodoro Engine

### attention_classes

*(defined above in Shared section)*

### duration_profiles

```sql
CREATE TABLE duration_profiles (
  id                          UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id                     UUID NOT NULL REFERENCES users(id),
  chore_id                    UUID REFERENCES chore_definitions(id),
  master_event_id             UUID REFERENCES events(id),
  label                       TEXT NOT NULL,
  attention_class             attention_class_value NOT NULL,

  -- Total duration Welford
  total_sample_count          INT NOT NULL DEFAULT 0,
  total_mean_minutes          NUMERIC(8,2) NOT NULL DEFAULT 0,
  total_m2                    NUMERIC(10,4) NOT NULL DEFAULT 0,
  total_min_minutes           NUMERIC(8,2),
  total_max_minutes           NUMERIC(8,2),

  -- Meaningful duration Welford (active events only)
  meaningful_sample_count     INT NOT NULL DEFAULT 0,
  meaningful_mean_minutes     NUMERIC(8,2) NOT NULL DEFAULT 0,
  meaningful_m2               NUMERIC(10,4) NOT NULL DEFAULT 0,

  -- Residual statistics
  residual_count              INT NOT NULL DEFAULT 0,
  mean_residual_minutes       NUMERIC(8,2),
  mean_residual_sessions      NUMERIC(4,2),

  -- Day-of-week breakdown (0=Sun..6=Sat)
  dow_total_mean              NUMERIC(8,2)[] DEFAULT ARRAY[NULL,NULL,NULL,NULL,NULL,NULL,NULL]::NUMERIC[],
  dow_meaningful_mean         NUMERIC(8,2)[] DEFAULT ARRAY[NULL,NULL,NULL,NULL,NULL,NULL,NULL]::NUMERIC[],
  dow_sample_count            INT[] DEFAULT ARRAY[0,0,0,0,0,0,0],

  mc_weight                   NUMERIC(6,4) NOT NULL DEFAULT 1.0,
  updated_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT chk_profile_source CHECK (
    chore_id IS NOT NULL OR master_event_id IS NOT NULL
  )
);

CREATE INDEX idx_duration_profiles_user  ON duration_profiles (user_id);
CREATE INDEX idx_duration_profiles_chore ON duration_profiles (chore_id)
  WHERE chore_id IS NOT NULL;
```

-----

### pomodoro_sessions

```sql
-- Only applicable to active attention_class events.
-- involved and passive events do not create pomodoro_sessions.

CREATE TYPE pomodoro_status AS ENUM (
  'active', 'paused', 'completed', 'abandoned', 'interrupted'
);

CREATE TABLE pomodoro_sessions (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id               UUID NOT NULL REFERENCES users(id),
  event_id              UUID REFERENCES events(id),
  canvas_item_id        UUID REFERENCES canvas_items(id),
  duration_profile_id   UUID REFERENCES duration_profiles(id),
  editor_session_id     UUID REFERENCES editor_sessions(id),  -- linked if editor open

  intended_minutes      INT NOT NULL,
  actual_minutes        INT,
  meaningful_minutes    INT,
  dead_zone_minutes     INT NOT NULL DEFAULT 0,
  interruption_count    INT NOT NULL DEFAULT 0,

  started_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  paused_at             TIMESTAMPTZ,
  resumed_at            TIMESTAMPTZ,
  ended_at              TIMESTAMPTZ,

  status                pomodoro_status NOT NULL DEFAULT 'active',
  completion_flag       BOOLEAN NOT NULL DEFAULT FALSE,
  notes                 TEXT,
  residual_id           UUID REFERENCES task_residuals(id),

  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pomodoro_user  ON pomodoro_sessions (user_id, started_at DESC);
CREATE INDEX idx_pomodoro_event ON pomodoro_sessions (event_id) WHERE event_id IS NOT NULL;
CREATE INDEX idx_pomodoro_open  ON pomodoro_sessions (user_id)
  WHERE completion_flag = FALSE AND status NOT IN ('abandoned', 'interrupted');
```

-----

### task_residuals

```sql
-- Only created for active attention_class events.
-- Involved events use delay flags, not residuals.
-- Created only after user confirms via residual_prompts — never auto-created.

CREATE TYPE residual_status AS ENUM (
  'open', 'scheduled', 'in_progress', 'completed', 'abandoned'
);

CREATE TABLE task_residuals (
  id                        UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id                   UUID NOT NULL REFERENCES users(id),
  origin_event_id           UUID NOT NULL REFERENCES events(id),
  origin_session_id         UUID NOT NULL REFERENCES pomodoro_sessions(id),
  duration_profile_id       UUID REFERENCES duration_profiles(id),
  remaining_minutes         INT NOT NULL,
  total_elapsed_minutes     INT NOT NULL DEFAULT 0,
  total_meaningful_minutes  INT NOT NULL DEFAULT 0,
  session_count             INT NOT NULL DEFAULT 1,
  status                    residual_status NOT NULL DEFAULT 'open',
  next_event_id             UUID REFERENCES events(id),
  next_scheduled_at         TIMESTAMPTZ,
  completion_flag           BOOLEAN NOT NULL DEFAULT FALSE,
  completed_at              TIMESTAMPTZ,
  created_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_residuals_open
  ON task_residuals (user_id, status) WHERE status IN ('open', 'scheduled');
```

-----

### residual_prompts

```sql
-- Created when a session ends without completion_flag.
-- System waits for user response before creating task_residuals.
-- Timeout without response → flagged for manual review.

CREATE TYPE prompt_status AS ENUM (
  'pending',          -- awaiting user response
  'confirmed',        -- user confirmed incomplete → task_residual created
  'completed',        -- user said done → completion_flag set, no residual
  'timed_out',        -- no response → flagged for manual review
  'dismissed'         -- user dismissed without answering
);

CREATE TABLE residual_prompts (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id               UUID NOT NULL REFERENCES users(id),
  pomodoro_session_id   UUID NOT NULL REFERENCES pomodoro_sessions(id),
  event_id              UUID NOT NULL REFERENCES events(id),
  status                prompt_status NOT NULL DEFAULT 'pending',
  prompted_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  responded_at          TIMESTAMPTZ,
  timeout_at            TIMESTAMPTZ NOT NULL,  -- prompted_at + configurable timeout
  user_remaining_minutes INT,                  -- user-entered if confirmed incomplete
  residual_id           UUID REFERENCES task_residuals(id),  -- set if confirmed
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_residual_prompts_pending
  ON residual_prompts (user_id, status) WHERE status = 'pending';
CREATE INDEX idx_residual_prompts_timeout
  ON residual_prompts (timeout_at) WHERE status = 'pending';
-- pg_cron polls timeout_at <= NOW() to flip pending → timed_out
```

-----

### session_feedback

```sql
CREATE TABLE session_feedback (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id               UUID NOT NULL REFERENCES users(id),
  pomodoro_session_id   UUID NOT NULL REFERENCES pomodoro_sessions(id),
  canvas_item_id        UUID REFERENCES canvas_items(id),
  completion_flag       BOOLEAN NOT NULL,
  notes                 TEXT,
  inferred_performance  NUMERIC(5,4),   -- meaningful / actual, set by analysis service
  flagged_dead_zone     BOOLEAN NOT NULL DEFAULT FALSE,
  submitted_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_feedback_session ON session_feedback (pomodoro_session_id);
CREATE INDEX idx_feedback_user    ON session_feedback (user_id, submitted_at DESC);
```

-----

### chronobiology_profiles

```sql
CREATE TABLE chronobiology_profiles (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id               UUID UNIQUE NOT NULL REFERENCES users(id),
  hourly_performance    NUMERIC(5,4)[] DEFAULT ARRAY_FILL(NULL::NUMERIC, ARRAY[24]),
  hourly_sample_count   INT[] DEFAULT ARRAY_FILL(0, ARRAY[24]),
  hourly_m2             NUMERIC(10,4)[] DEFAULT ARRAY_FILL(0::NUMERIC, ARRAY[24]),
  known_dead_zones      INT[],
  known_peak_zones      INT[],
  cortisol_drop_hour    INT,            -- learned per user, never assumed
  last_analyzed_at      TIMESTAMPTZ,
  sample_count          INT NOT NULL DEFAULT 0,
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

-----

## Radial Canvas

### canvas_sessions

```sql
CREATE TABLE canvas_sessions (
  id          UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id     UUID NOT NULL REFERENCES users(id),
  opened_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  closed_at   TIMESTAMPTZ,
  item_count  INT NOT NULL DEFAULT 0
);
CREATE INDEX idx_canvas_sessions_user ON canvas_sessions (user_id, opened_at DESC);
```

-----

### canvas_items

```sql
CREATE TYPE canvas_item_status AS ENUM ('active', 'snoozed', 'dismissed', 'completed');

CREATE TABLE canvas_items (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  canvas_session_id     UUID NOT NULL REFERENCES canvas_sessions(id),
  user_id               UUID NOT NULL REFERENCES users(id),
  event_id              UUID REFERENCES events(id),
  label                 TEXT NOT NULL,
  canvas_event_type     canvas_event_type NOT NULL,   -- auto-assigned, mirrors event
  attention_class       attention_class_value NOT NULL,

  -- Radial position
  r                     NUMERIC(5,4) NOT NULL CHECK (r >= 0 AND r <= 1),
  theta                 NUMERIC(6,2) NOT NULL CHECK (theta >= 0 AND theta < 360),
  attention_weight      NUMERIC(5,4) GENERATED ALWAYS AS (1.0 - r) STORED,

  -- Zoom tracking (A/B testing + self-improvement signal)
  zoom_count            INT NOT NULL DEFAULT 0,          -- how many times user zoomed on this item
  zoom_total_delta      NUMERIC(8,2) NOT NULL DEFAULT 0, -- cumulative zoom delta (+ = zoom in)
  zoom_mean_delta       NUMERIC(6,2),                    -- mean zoom per interaction, updated on close
  -- zoom data feeds canvas layout A/B tests:
  -- items with high zoom_count and positive zoom_mean_delta → place closer to center by default

  status                canvas_item_status NOT NULL DEFAULT 'active',
  placed_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_moved_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  dismissed_at          TIMESTAMPTZ,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_canvas_items_session ON canvas_items (canvas_session_id);
CREATE INDEX idx_canvas_items_r
  ON canvas_items (user_id, r) WHERE status = 'active';
```

-----

### canvas_alarms

```sql
-- alarm_status: 'pending' renamed to 'started' — reflects that
-- the alarm timeline begins when the item is placed, not when it fires.

CREATE TYPE alarm_status AS ENUM (
  'started',        -- was: pending — alarm timeline active
  'fired',
  'acknowledged',
  'dismissed',
  'cancelled',
  'snoozed'
);

CREATE TYPE alarm_class AS ENUM (
  'passive_check',      -- check the laundry, check the oven
  'focus_checkpoint',   -- pomodoro mark — does NOT stop timer if session exceeds 25 min
                        -- break logic: if session >= 40min → break = 50% logged time
                        --              if session < 40min  → break = 5 min
  'break_signal',       -- scheduled break point
  'refresh_start',      -- user began another pomodoro on same task
                        -- implies prior session ended, new session starts on same event
  'custom'
);

CREATE TABLE canvas_alarms (
  id                UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  canvas_item_id    UUID NOT NULL REFERENCES canvas_items(id),
  user_id           UUID NOT NULL REFERENCES users(id),
  alarm_class       alarm_class NOT NULL DEFAULT 'custom',
  label             TEXT,
  offset_minutes    NUMERIC(6,2) NOT NULL,
  fires_at          TIMESTAMPTZ NOT NULL,
  handle_position   NUMERIC(5,4) NOT NULL CHECK (handle_position >= 0 AND handle_position <= 1),
  status            alarm_status NOT NULL DEFAULT 'started',
  fired_at          TIMESTAMPTZ,
  acknowledged_at   TIMESTAMPTZ,
  snooze_minutes    INT,
  snoozed_until     TIMESTAMPTZ,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_canvas_alarms_started
  ON canvas_alarms (fires_at) WHERE status = 'started';
CREATE INDEX idx_canvas_alarms_item ON canvas_alarms (canvas_item_id);
```

> **Focus checkpoint break logic:**
> 
> - Session actual time < 40 min → break = 5 min flat
> - Session actual time ≥ 40 min → break = 50% of logged meaningful time
> - Focus checkpoint alarm does NOT interrupt or stop the timer
> - `refresh_start` alarm fires when user restarts pomodoro on same event
>   — creates a new `pomodoro_session` linked to same `event_id`, `refresh_start` logged

-----

### canvas_position_history

```sql
-- Append-only drag log. DRY note: zoom data lives on canvas_items
-- (aggregated), position history lives here (per-drag raw signal).
-- These are different concerns — no duplication.

CREATE TABLE canvas_position_history (
  id              UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  canvas_item_id  UUID NOT NULL REFERENCES canvas_items(id),
  user_id         UUID NOT NULL REFERENCES users(id),
  r_before        NUMERIC(5,4) NOT NULL,
  theta_before    NUMERIC(6,2) NOT NULL,
  r_after         NUMERIC(5,4) NOT NULL,
  theta_after     NUMERIC(6,2) NOT NULL,
  moved_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_position_history_item
  ON canvas_position_history (canvas_item_id, moved_at DESC);
```

-----

## Editor Engine

### Feature Overview

A Vim/Notion-inspired text editor embedded in the event/task context.
Captures structured session data via `/slash` commands.
Keystroke-level timing feeds bottleneck analysis and concept weakness detection.
`///` marks completion — idempotent, cannot duplicate within one editor session.

### Slash Commands

|Command     |Meaning                    |Data written                                            |
|------------|---------------------------|--------------------------------------------------------|
|`/start`    |Pomodoro begins            |`pomodoro_sessions` start, `slash_commands` entry       |
|`/break`    |Break begins               |`canvas_alarms` break_signal, `slash_commands` entry    |
|`/finished` |Task complete              |`completion_flag = TRUE`, `slash_commands` entry        |
|`/buffering`|User is thinking           |`keystroke_segments` gap logged, `slash_commands` entry |
|`/prompt`   |AI model called            |`ai_sessions` created, `slash_commands` entry           |
|`/bored`    |Behavioral signal          |`slash_commands` entry, `boredom_flag = TRUE` on segment|
|`///`       |Problem completed/started  |Completion marker — idempotent per session              |
|`////`      |Additional completion depth|Allowed — each `///` block is a separate marker         |


> `///` cannot duplicate within one `editor_session`. A second `///` on the same line
> is a no-op. Additional `///` groups after a newline are valid — each is a distinct marker.

### editor_sessions

```sql
CREATE TYPE editor_session_status AS ENUM (
  'active', 'paused', 'completed', 'abandoned'
);

CREATE TABLE editor_sessions (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id               UUID NOT NULL REFERENCES users(id),
  event_id              UUID REFERENCES events(id),
  pomodoro_session_id   UUID REFERENCES pomodoro_sessions(id),
  title                 TEXT,
  status                editor_session_status NOT NULL DEFAULT 'active',

  -- Session-level timing
  started_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  ended_at              TIMESTAMPTZ,
  total_active_ms       BIGINT NOT NULL DEFAULT 0,   -- ms with keystrokes
  total_idle_ms         BIGINT NOT NULL DEFAULT 0,   -- ms without keystrokes
  total_buffering_ms    BIGINT NOT NULL DEFAULT 0,   -- ms in /buffering state

  -- Content metrics
  word_count            INT NOT NULL DEFAULT 0,
  completion_markers    INT NOT NULL DEFAULT 0,       -- count of /// groups
  boredom_flags         INT NOT NULL DEFAULT 0,       -- count of /bored

  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_editor_sessions_user  ON editor_sessions (user_id, started_at DESC);
CREATE INDEX idx_editor_sessions_event ON editor_sessions (event_id) WHERE event_id IS NOT NULL;
```

-----

### keystroke_segments

```sql
-- A segment is a continuous burst of typing activity.
-- A new segment starts after idle_threshold_ms of no keystrokes (default 2000ms).
-- Captures when user started/stopped typing and the gap between bursts.
-- Gaps = thinking time, buffering, distraction — all distinguishable by context.

CREATE TYPE segment_context AS ENUM (
  'typing',       -- active keystroke burst
  'idle',         -- no activity, no slash command active
  'buffering',    -- /buffering command active
  'prompted',     -- /prompt active (waiting on AI)
  'break'         -- /break active
);

CREATE TABLE keystroke_segments (
  id                  UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  editor_session_id   UUID NOT NULL REFERENCES editor_sessions(id),
  user_id             UUID NOT NULL REFERENCES users(id),
  context             segment_context NOT NULL,
  started_at          TIMESTAMPTZ NOT NULL,
  ended_at            TIMESTAMPTZ,
  duration_ms         BIGINT,                     -- computed on segment close
  keystrokes          INT NOT NULL DEFAULT 0,
  words_added         INT NOT NULL DEFAULT 0,
  words_deleted       INT NOT NULL DEFAULT 0,
  boredom_flag        BOOLEAN NOT NULL DEFAULT FALSE,
  -- DRY note: slash_commands table stores the slash event itself;
  -- keystroke_segments stores the time window around it.
  -- No duplication — different granularity.
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_keystroke_segments_session
  ON keystroke_segments (editor_session_id, started_at);
CREATE INDEX idx_keystroke_segments_user
  ON keystroke_segments (user_id, started_at DESC);
```

-----

### slash_commands

```sql
CREATE TYPE slash_command_type AS ENUM (
  'start', 'break', 'finished', 'buffering', 'prompt', 'bored', 'completion_marker'
);

CREATE TABLE slash_commands (
  id                  UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  editor_session_id   UUID NOT NULL REFERENCES editor_sessions(id),
  user_id             UUID NOT NULL REFERENCES users(id),
  command_type        slash_command_type NOT NULL,
  triggered_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  cursor_position     INT,                          -- character offset in document
  line_number         INT,
  context_text        TEXT,                         -- surrounding text at trigger (100 char window)
                                                    -- not the full document
  linked_session_id   UUID REFERENCES pomodoro_sessions(id),   -- for /start, /break, /finished
  linked_ai_session_id UUID REFERENCES ai_sessions(id),        -- for /prompt
  is_duplicate        BOOLEAN NOT NULL DEFAULT FALSE,           -- TRUE if /// fired twice same line
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_slash_commands_session ON slash_commands (editor_session_id, triggered_at);
CREATE UNIQUE INDEX idx_completion_marker_dedupe
  ON slash_commands (editor_session_id, line_number)
  WHERE command_type = 'completion_marker' AND is_duplicate = FALSE;
-- Enforces /// idempotency at DB level per session per line
```

-----

### concept_analyses

```sql
-- AI analysis of editor session data.
-- Identifies concept weaknesses from keystroke patterns,
-- buffering frequency, boredom flags, and completion markers.
-- Input: editor_session summary + keystroke_segments + slash_commands
-- Output: structured weakness signals stored here.

CREATE TABLE concept_analyses (
  id                    UUID PRIMARY KEY DEFAULT gen_uuid_v7(),
  user_id               UUID NOT NULL REFERENCES users(id),
  editor_session_id     UUID NOT NULL REFERENCES editor_sessions(id),
  ai_session_id         UUID REFERENCES ai_sessions(id),

  -- Bottleneck signals identified by model
  high_idle_ratio       BOOLEAN NOT NULL DEFAULT FALSE,  -- idle_ms / total_ms > threshold
  high_buffering_ratio  BOOLEAN NOT NULL DEFAULT FALSE,  -- thinking time unusually high
  frequent_deletions    BOOLEAN NOT NULL DEFAULT FALSE,  -- high words_deleted / words_added
  boredom_detected      BOOLEAN NOT NULL DEFAULT FALSE,
  completion_rate       NUMERIC(5,4),                    -- completion_markers / expected

  -- Concept weakness (from model response)
  weakness_summary      TEXT,                            -- model-generated, < 500 chars
  suggested_topics      TEXT[],                          -- topics to revisit
  confidence            NUMERIC(5,4),                    -- model confidence in analysis

  analyzed_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_concept_analyses_user
  ON concept_analyses (user_id, analyzed_at DESC);
CREATE INDEX idx_concept_analyses_session
  ON concept_analyses (editor_session_id);
```

-----

## Math Interpreter

```
Embedded in the editor. Triggered by:
- Inline expressions: "3 * 25 = " → autocompletes "75"
- Slash context: "/start 25 + 10" → resolves to 35 min session

Implementation:
  - Client-side: mathjs library parses inline expressions
  - Trigger: cursor after "=" with numeric expression to the left
  - Output: autocomplete suggestion, user accepts with Tab
  - No server round-trip for basic arithmetic
  - Server-side: Python mathjs equivalent (sympy) for complex expressions
    via /api/v1/editor/evaluate endpoint

No DB storage needed — math evaluation is stateless.
Results that feed into pomodoro (session duration) are stored via /start command.
```

-----

## DRY Audit

```
Potential duplication reviewed and resolved:

1. canvas_items.zoom_* vs canvas_position_history
   → Different: zoom is aggregated per-item, position is per-drag raw log. No duplication.

2. pomodoro_sessions.actual_minutes vs events.actual_minutes
   → events.actual_minutes = sum across all pomodoro sessions for that event.
      Updated by application layer on session complete. Not duplicated — different scope.

3. session_feedback vs pomodoro_sessions.notes
   → pomodoro_sessions.notes = in-session notes (real-time).
      session_feedback.notes = post-session reflection. Different timing, different purpose.

4. slash_commands vs keystroke_segments
   → slash_commands = discrete events. keystroke_segments = time windows.
      Complementary, not duplicated.

5. editor_sessions.total_active_ms vs SUM(keystroke_segments.duration_ms)
   → editor_sessions totals are running aggregates updated in real-time.
      keystroke_segments are the raw records they aggregate from.
      Denormalized intentionally — avoids expensive SUM on every read.
      Application layer keeps them in sync on segment close.

6. attention_classes table vs attention_class_value enum
   → Enum used as FK-safe type. Table adds semantic metadata (pomodoro_applicable etc).
      Both necessary — enum for DB type safety, table for application logic. Not duplicated.

7. chore_definitions.color vs calendars.color
   → Different entities, both need display color. Shared enum type, not shared data.

8. event_memory.canvas_event_type vs events.canvas_event_type
   → Memory stores the learned default. Event stores the actual assigned value.
      Memory is the prior; event is the result. Not duplicated.
```

-----

## Retention Policy

*(To be defined in separate discussion)*

-----

## ChromaDB Collections

```
canvas_behavior
  Vector: (attention_class, r_mean, theta_mean, alarm_count, canvas_event_type, outcome)
  Feeds canvas layout A/B testing alongside zoom_count signal

pomodoro_patterns
  Vector: (task_label, day_of_week, hour_of_day, intended_minutes, meaningful_minutes)
  Novel task duration estimation via semantic similarity

chronobiology_signal
  Vector: (hour_of_day, day_of_week, inferred_performance, notes_embedding)
  Notes embedding from session_feedback.notes for qualitative dead zone detection

editor_concepts
  Vector: (weakness_summary_embedding, suggested_topics_embedding, event_type)
  Accumulates concept weakness signal across sessions
  Enables: "you consistently buffer on recursion problems" → suggest review
```

-----

## Parallel System Boundaries

```
CALENDAR CORE
  calendars · events · recurrence_exceptions · event_acl · event_memory · attention_classes
  Read-only from all systems below. Written to by chore scheduler (apply) and residuals (next slot).

CHORE SCHEDULER
  chore_definitions · chore_n_history · schedule_runs · chore_occurrences · schedule_healing_log
  Reads calendar for conflict avoidance. Writes chore_occurrences → events on apply.

POMODORO ENGINE
  pomodoro_sessions · task_residuals · residual_prompts · session_feedback
  duration_profiles · chronobiology_profiles
  Reads events. Writes duration_profiles. Creates residual_prompts on overrun.

RADIAL CANVAS
  canvas_sessions · canvas_items · canvas_alarms · canvas_position_history
  Reads events and pomodoro. Writes nothing back to calendar or pomodoro.
  Alarm pipeline: pg_cron → Redis Streams → WebSocket push.

EDITOR ENGINE
  editor_sessions · keystroke_segments · slash_commands · concept_analyses
  Reads pomodoro_sessions (links to active session).
  Writes slash_commands that trigger pomodoro events.
  Writes concept_analyses via AI session.
  Math evaluation is stateless — no DB writes.
```
