# Engineering & Coding Standards Handbook

**NASA-Inspired · Data-Driven Execution · Full-Stack**

-----

## Table of Contents

1. [Core Architecture & Stack](#1-core-architecture--stack)
1. [Naming Conventions](#2-naming-conventions)
1. [Directory Layout](#3-directory-layout)
1. [Execution Philosophy](#4-execution-philosophy)
1. [NASA 10 Laws — Adapted for Web](#5-nasa-10-laws--adapted-for-web)
1. [Pattern Reference](#6-pattern-reference)
1. [Infrastructure & Deployment](#7-infrastructure--deployment)
1. [AI Agent Code Generation Rules](#8-ai-agent-code-generation-rules)

-----

## 1. Core Architecture & Stack

Distributed, containerized, microservices-oriented. Each layer owns one concern.

|Layer             |Technology           |Responsibility                                     |State                      |
|------------------|---------------------|---------------------------------------------------|---------------------------|
|Frontend          |React 19 + TypeScript|UI, routing, client state                          |Zustand, TanStack Query    |
|API Gateway       |Nginx                |Reverse proxy, TLS, rate limiting, routing         |Stateless                  |
|Backend API       |FastAPI (Python)     |Auth, orchestration, validation, routing           |Redis sessions + cache     |
|AI / Service Logic|Python               |Pipelines, ML, agent execution, data transforms    |In-memory queues, transient|
|Relational DB     |PostgreSQL 16        |Structured relational storage, source of truth     |PersistentVolumeClaims     |
|Vector DB         |ChromaDB             |Embeddings, similarity search, RAG context         |Persistent disk indices    |
|Containerization  |Docker               |Environment isolation, multi-stage builds          |Ephemeral + mounted volumes|
|Orchestration     |k3s (Kubernetes)     |Pod scaling, self-healing, rolling updates, secrets|PVC, ConfigMaps, Vault     |

-----

## 2. Naming Conventions

|Asset                |Convention                                |Example                                                 |
|---------------------|------------------------------------------|--------------------------------------------------------|
|File names           |`kebab-case`                              |`user-profile.tsx`, `auth-middleware.py`                |
|Top-level directories|`kebab-case`                              |`my-react-app/src/`                                     |
|Module/subdirectories|`PascalCase`                              |`ComponentUtils/`, `RouteHandlers/`                     |
|Variables & functions|`snake_case`                              |`user_id`, `get_session_data()`                         |
|Classes & types      |`PascalCase`                              |`class DatabaseConnector`, `type UserPayload`           |
|Database tables      |`snake_case`                              |`user_accounts`, `vector_embeddings`                    |
|Environment variables|`UPPER_SNAKE`                             |`DATABASE_URL`, `JWT_SECRET_KEY`                        |
|React component files|`kebab-case` filename, `PascalCase` export|`primary-button.tsx` → `export function PrimaryButton()`|
|TypeScript interfaces|No `I` prefix                             |`UserProfile` not `IUserProfile`                        |
|Boolean variables    |`is_` / `has_` / `can_` prefix            |`is_active`, `has_payload`, `can_submit`                |

-----

## 3. Directory Layout

### Frontend (React + TypeScript)

```
src/
├── Assets/
├── Components/
│   └── PrimaryButton/
│       ├── primary-button.tsx
│       └── primary-button.test.tsx
├── Hooks/
├── Pages/
├── Services/
├── Store/
├── Types/
├── lib/
│   ├── patterns.ts         # All regex — named exports only
│   ├── crypto/             # All crypto operations
│   └── dispatch-maps/      # Lookup tables for data-driven dispatch
├── app.tsx
└── main.tsx
```

### Backend (FastAPI / Python)

```
app/
├── Agents/
├── Core/                   # Config, logging, lifespan, app factory
├── Middleware/              # Auth, validation, prompt sanitiser, logging
├── Models/                 # SQLAlchemy ORM models
├── Pipelines/              # Pure-function data transforms (no I/O)
├── Routers/                # FastAPI routes — thin, delegates to Services
├── Schemas/                # Pydantic v2 schemas
├── Services/               # Business logic layer
├── dispatch_maps/          # Lookup tables for data-driven dispatch
└── main.py
```

### Shared

```
packages/
├── types/                  # Shared TS types generated from OpenAPI spec
└── ui/                     # Owned shadcn/ui component overrides
```

-----

## 4. Execution Philosophy

### Core Principle

> Minimize explicit application-level control-flow divergence.
> Prefer deterministic, data-driven execution over imperative branching.

This is a design preference, not a zero-tolerance ban. The goal is to reduce nested, repeated, and structurally tangled branching — not to make simple code needlessly complex.

-----

### Where data-driven dispatch is mandatory

These layers must use lookup tables, maps, and declarative transforms. Branching here creates maintenance surfaces that grow with every new case:

|Layer                      |Why                                                           |
|---------------------------|--------------------------------------------------------------|
|Dispatch systems           |Every `if (type === x)` will grow into an `if/elif/elif` chain|
|Event routing              |Route tables scale cleanly; chained conditionals do not       |
|State transforms / reducers|Exhaustive maps are statically verifiable                     |
|Rendering pipelines        |Component maps are composable; nested ternaries are not       |
|Orchestration layers       |Declarative config is auditable; imperative logic is not      |

-----

### Where branches are fully permitted

These contexts are where imperative control flow is correct, readable, and appropriate:

|Context             |Why branching is fine                                 |
|--------------------|------------------------------------------------------|
|Basic guards        |`if (!user) return` is clearer than any alternative   |
|Input validation    |Schema error paths are explicit by design             |
|Simple early exits  |Single-condition exits reduce indentation without loss|
|Local business logic|A two-case condition is not a dispatch problem        |
|Error handling      |`try/catch` is the right construct for error surfaces |

-----

### The actual rules

**Avoid:**

- Nested `if/else` chains (more than one level deep)
- Repeated branching on the same variable across multiple functions
- `switch` statements with more than 3 cases — use a dispatch map instead
- Inline ternaries inside JSX beyond a single value substitution
- Early returns that branch on more than one condition

**Prefer:**

- Lookup tables for multi-case dispatch
- Boolean reduction for compound guard conditions
- Functional transforms (`.filter`, `.map`, `.reduce`) over imperative loops with conditionals
- Optional chaining and nullish coalescing for nullable access
- Single exit points in pure functions and service layer methods

**Mandatory regardless:**

- Strict schema validation on all external inputs
- Named booleans before use in conditions
- No inline regex, no inline crypto
- Immutable updates only
- Singleton infrastructure resources

-----

### Enforcement tier

|Practice                                           |Tier                        |
|---------------------------------------------------|----------------------------|
|Strict schema validation (Pydantic / Zod)          |**Mandatory**               |
|Singleton infra resources                          |**Mandatory**               |
|No inline crypto                                   |**Mandatory**               |
|No inline regex                                    |**Mandatory**               |
|Naming conventions                                 |**Mandatory**               |
|Immutable updates                                  |**Mandatory**               |
|CI enforcement (zero warnings)                     |**Mandatory**               |
|Dispatch maps for multi-case routing               |**Strongly preferred**      |
|Boolean reduction for compound guards              |**Strongly preferred**      |
|Functional transforms over imperative loops        |**Strongly preferred**      |
|Single exit point in pure/service functions        |**Preferred**               |
|Avoiding nested if/else                            |**Preferred**               |
|Avoiding ternaries in JSX beyond value substitution|**Preferred**               |
|Early returns for simple guards                    |**Permitted — use judgment**|
|Local if/else for two-case business logic          |**Permitted — use judgment**|

-----

## 5. NASA 10 Laws — Adapted for Web

The original NASA/JPL Power of Ten rules target avionics and constrained embedded systems. The following are adapted for a FastAPI + React + async distributed stack. The spirit is preserved; the letter is applied with judgment.

-----

### Rule 1 — Simple Control Flow

Avoid deep nesting and multi-level recursion. Prefer readable functional methods. When recursion is required, enforce a hard depth ceiling.

```python
MAX_DEPTH: int = 10

def traverse(node: Node, depth: int = 0) -> list[Node]:
    if depth >= MAX_DEPTH:
        return [node]
    if not node.children:
        return [node]
    child_results = [
        item
        for child in node.children
        for item in traverse(child, depth + 1)
    ]
    return [node, *child_results]
```

```typescript
const MAX_DEPTH = 10;

const traverse = (node: Node, depth = 0): Node[] => {
  if (depth >= MAX_DEPTH || node.children.length === 0) return [node];
  return [node, ...node.children.flatMap(c => traverse(c, depth + 1))];
};
```

> Guard clauses at the top of recursive functions are the correct pattern here. This is one of the legitimate uses of early return.

-----

### Rule 2 — Fixed Loop Upper Bounds

Every loop, worker, and agent iteration must declare an explicit maximum.

```python
MAX_AGENT_ITERATIONS: int = 10

def run_agent(task: Task) -> Result:
    results = [execute_step(task, i) for i in range(MAX_AGENT_ITERATIONS)]
    return aggregate_results(results)
```

```typescript
const MAX_AGENT_ITERATIONS = 10;

const run_agent = async (task: Task): Promise<Result> => {
  const results = await Promise.all(
    Array.from({ length: MAX_AGENT_ITERATIONS }, (_, i) => execute_step(task, i))
  );
  return aggregate_results(results);
};
```

-----

### Rule 3 — Memory & Resource Stability Post-Initialization

Instantiate infrastructure singletons exactly once at application boot.

```python
# Core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

engine  = create_async_engine(DATABASE_URL, pool_size=20, max_overflow=0)
Session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
# Import Session everywhere — never call create_async_engine inside a route
```

```typescript
// Services/chroma-client.ts
import { ChromaClient } from 'chromadb';
const chroma_client = new ChromaClient({ path: CHROMA_URL });
export { chroma_client };
// Import this — never `new ChromaClient()` inside a component or route
```

-----

### Rule 4 — 60-Line Function Ceiling

No function, component, or handler exceeds 60 lines. Extract into named helpers.

```typescript
// PROHIBITED — monolithic component
export function UserDashboard() { /* 80 lines */ }

// REQUIRED — composed from named sub-functions under 60 lines each
const render_header   = (user: User)       => <Header user={user} />;
const render_stats    = (stats: Stats)     => <StatGrid stats={stats} />;
const render_calendar = (events: Event[])  => <CalendarView events={events} />;

export function UserDashboard({ user, stats, events }: DashboardProps) {
  return (
    <main>
      {render_header(user)}
      {render_stats(stats)}
      {render_calendar(events)}
    </main>
  );
}
```

-----

### Rule 5 — One Statement Per Line, Low Density

One thing declared per line. Named boolean intermediates before compound conditions.

```typescript
// PROHIBITED
const is_valid = req.body.user && req.body.user.age > 18
  ? req.body.user.role === 'admin' ? true : false : false;

// REQUIRED
const has_payload    = Boolean(req.body.user);
const is_adult       = has_payload && req.body.user.age > 18;
const is_admin       = has_payload && req.body.user.role === 'admin';
const is_valid_admin = is_adult && is_admin;
```

-----

### Rule 6 — Minimal Scope, `const` by Default

Declare variables immediately before use. `var` banned. `const` default. `let` only for index counters with no functional alternative.

```typescript
// PROHIBITED
var user_data = null;
// ... 20 lines ...
user_data = await fetch_user(id);

// REQUIRED
const user_data = await fetch_user(id);
const result    = compute(user_data);
```

-----

### Rule 7 — Zero Ignored Returns, Strict Validation

Every non-void call has its return checked. Every external input parsed through a schema.

```typescript
const parse_user = (raw: unknown) => UserSchema.safeParse(raw);
// Caller always checks .success before using .data
```

```python
class UserCreate(BaseModel):
    model_config = ConfigDict(strict=True)
    email: str
    role:  str
# FastAPI validates at boundary — strict=True on every schema
```

-----

### Rule 8 — Explicit Environmental Execution

No runtime config divergence between local and prod. No `NODE_ENV === 'development'` forks in application logic. Feature flags via config values, not environment string checks.

-----

### Rule 9 — Immutability by Default

Never mutate arguments or shared state directly.

```typescript
// PROHIBITED
const update_user = (user: User) => { user.name = 'New'; return user; };

// REQUIRED
const update_user = (user: User, name: string): User => ({ ...user, name });
```

```python
from dataclasses import replace

def update_user(user: User, name: str) -> User:
    return replace(user, name=name)
```

-----

### Rule 10 — Pedantic Compiler Enforcement

Zero warnings, zero errors in CI.

|Tool                     |Scope              |Gate   |
|-------------------------|-------------------|-------|
|`mypy --strict`          |Python types       |CI fail|
|`ruff check`             |Python lint        |CI fail|
|`ruff format --check`    |Python format      |CI fail|
|`tsc --noEmit`           |TypeScript types   |CI fail|
|`eslint --max-warnings 0`|TS/JS lint         |CI fail|
|`prettier --check`       |TS/JS format       |CI fail|
|SonarQube quality gate   |Duplication, smells|CI fail|
|Trivy                    |Container CVEs     |CI fail|
|Gitleaks                 |Secrets in code    |CI fail|

-----

## 6. Pattern Reference

### Dispatch Maps — Use for multi-case routing (3+ cases)

```typescript
const EVENT_HANDLERS: Record<EventType, (e: CalendarEvent) => void> = {
  meeting:  handle_meeting,
  task:     handle_task,
  chore:    handle_chore,
  homework: handle_homework,
  passive:  handle_passive,
  physical: handle_physical,
};

const handle_event = (event: CalendarEvent): void => {
  EVENT_HANDLERS[event.event_type](event);
};
```

```python
EVENT_HANDLERS: dict[str, Callable[[Event], None]] = {
    "meeting":  handle_meeting,
    "task":     handle_task,
    "chore":    handle_chore,
    "homework": handle_homework,
    "passive":  handle_passive,
    "physical": handle_physical,
}

def handle_event(event: Event) -> None:
    EVENT_HANDLERS[event.event_type](event)
```

> For 1–2 cases, a plain `if` is cleaner. Dispatch maps earn their weight at 3+ cases.

-----

### Attention Class Dispatch

```python
ATTENTION_DEFAULTS: dict[str, dict] = {
    "active":   {"pomodoro": True,  "residual": True,  "default_r": 0.2},
    "involved": {"pomodoro": False, "residual": False, "default_r": 0.3},
    "passive":  {"pomodoro": False, "residual": False, "default_r": 0.8},
}

def get_attention_config(attention_class: str) -> dict:
    return ATTENTION_DEFAULTS[attention_class]
```

-----

### Canvas Event Type Assignment

```python
OVERLAP_TYPE_MAP: dict[frozenset, str] = {
    frozenset({"active"}):             "focus_only",
    frozenset({"involved"}):           "involved_only",
    frozenset({"passive"}):            "passive_multi",
    frozenset({"active", "passive"}):  "focus_passive",
    frozenset({"involved","passive"}): "involved_only",
    frozenset({"active","involved"}):  "focus_only",
}

def assign_canvas_type(attention_classes: list[str]) -> str:
    return OVERLAP_TYPE_MAP.get(frozenset(attention_classes), "focus_only")
```

-----

### Boolean Reduction — Use for compound guards

```typescript
const has_user    = Boolean(user);
const is_active   = has_user && user.active;
const has_role    = has_user && Boolean(user.role);
const can_proceed = is_active && has_role;
```

-----

### Simple Guard — Early return is correct here

```typescript
// Permitted — simple, single-condition guard
const process_user = (user: User | null): string => {
  if (!user) return 'no-user';
  return user.name;
};
```

-----

### Nullable Access

```typescript
const city = user?.address?.city ?? 'Unknown';
const name = user?.name ?? 'Anonymous';
```

-----

### Numeric Bounds

```typescript
const clamp   = (v: number, lo: number, hi: number): number => Math.min(Math.max(v, lo), hi);
const percent = (part: number, total: number): number => clamp((part / total) * 100, 0, 100);
```

-----

### Break Duration Logic — Local business logic, if permitted

```typescript
// Two cases, local to one function — if is appropriate here
const get_break_minutes = (session_minutes: number): number => {
  if (session_minutes >= 40) return Math.round(session_minutes * 0.5);
  return 5;
};
```

> Alternatively as a dispatch map if this rule spreads to multiple callsites:

```typescript
const BREAK_RULES: Array<{ threshold: number; compute: (m: number) => number }> = [
  { threshold: 40, compute: (m) => Math.round(m * 0.5) },
  { threshold: 0,  compute: (_) => 5 },
];

const get_break_minutes = (session_minutes: number): number =>
  (BREAK_RULES.find(r => session_minutes >= r.threshold) ?? BREAK_RULES[1])
    .compute(session_minutes);
```

-----

### Functional Transforms — Prefer over imperative loops with conditionals

```typescript
// DISCOURAGED
const result: string[] = [];
for (const user of users) {
  if (user.active) result.push(user.name);
}

// PREFERRED
const result = users
  .filter(u => u.active)
  .map(u => u.name);
```

-----

## 7. Infrastructure & Deployment

### Docker

- Multi-stage builds: `base` → `test` → `prod` — test layer never ships
- Non-root user before `CMD` in every image
- Image tagged by git SHA — no `latest`
- No secrets in Dockerfile layers — build args only

### k3s / Kubernetes

- `livenessProbe` and `readinessProbe` on every HTTP workload
- Resource `requests` and `limits` on every container
- `maxUnavailable: 0`, `maxSurge: 1` — zero-downtime rolling update
- Vault Agent sidecar injects secrets — no secrets in ConfigMaps or env files
- Linkerd mTLS on all meshed pods — `AuthorizationPolicy` deny-by-default

### Nginx

- All traffic enters through Nginx — no direct pod port exposure
- TLS 1.3 via cert-manager
- Timeout aligned with backend: `proxy_read_timeout` = Uvicorn timeout (120s for LLM routes)
- Security headers: `HSTS`, `X-Frame-Options`, `X-Content-Type-Options`, `CSP`

-----

## 8. AI Agent Code Generation Rules

When generating, modifying, or reviewing code in this codebase:

### Mandatory — no exceptions

1. **`snake_case`** for all variables, functions, object keys, and parameters. `camelCase` is forbidden.
1. **Strict schemas** — Pydantic `ConfigDict(strict=True)` or Zod `.safeParse()` on every external input. Return value always checked.
1. **Immutable updates** — never mutate arguments. Use spread, `replace()`, `.map()`, `.filter()`.
1. **Singleton resources** — infrastructure clients instantiated once at boot, never inside a route or render.
1. **No inline regex** — all patterns are named exports in `lib/patterns.ts` or `patterns.py`.
1. **No inline crypto** — all operations in `lib/crypto/`.
1. **Naming conventions** — files `kebab-case`, module directories `PascalCase`, top-level `kebab-case`, env vars `UPPER_SNAKE`.
1. **60-line ceiling** — no function or component exceeds 60 lines.
1. **One statement per line** — named boolean intermediates before compound conditions.
1. **No placeholders** — no `// TODO`, no `...`, no stubs. Every function complete and valid.
1. **CI clean** — generated code must pass `mypy --strict`, `ruff`, `tsc --noEmit`, `eslint --max-warnings 0`.
1. **TOC update** — after every change, update `TOC.md`: add new files, uncheck “complete” on modified files.

### Strongly preferred

1. **Dispatch maps** for 3+ case routing — live in `lib/dispatch-maps/` (TS) or `dispatch_maps/` (Python).
1. **Boolean reduction** for compound guard conditions.
1. **Functional transforms** (`.filter`, `.map`) over imperative loops with conditionals.
1. **Single exit point** in pure functions and service layer methods.

### Permitted with judgment

1. **Early returns** for simple single-condition guards (`if (!user) return`).
1. **`if/else`** for two-case local business logic where a dispatch map would be over-engineering.
1. **Ternaries** for single value substitution in JSX or simple assignments — not nested, not chained.

### Never acceptable regardless of context

- Nested `if/else` chains beyond one level deep
- `switch` with 3+ cases — use a dispatch map
- Nested ternaries
- Repeated branching on the same variable across multiple functions
- `var` — banned entirely
- Mutation of function arguments
- Inline regex or inline crypto
- `camelCase` variables
