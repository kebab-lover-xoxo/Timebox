# System Architecture

**Chore-Optimised Calendar · Monte Carlo Scheduler · AI Timeboxing · Gantt View**
*Build order: top = first, bottom = last*

-----

## PostgreSQL 16

> Relational store. Source of truth. Holds all user, event, chore, and schedule data. Soft delete, 14-day hard purge.

- Store all structured data via SQLAlchemy 2.0 async models
- All schema changes through Alembic migrations — no manual ALTER TABLE ever
- UUID v7 primary keys on every table
- Soft delete only — `deleted_at` column, never hard delete application data
- Partial indexes on every `WHERE deleted_at IS NULL` query path
- PgBouncer transaction mode in prod — cap real connections at 20–50
- One write primary, one read replica — route reads explicitly
- `pg_cron` nightly hard purge: `DELETE WHERE deleted_at < NOW() - INTERVAL '14 days'`
- Purge logged to `purge_audit` before execution — fact of deletion kept 90 days
- **Constraint:** No raw SQL strings — ORM or parameterised queries only (SQL injection)
- **Constraint:** No schema changes without committed migration file first
- **Constraint:** Max function length 60 lines — split query builders if longer (NASA rule 1)

> **Python libs:** `asyncpg`, `sqlalchemy[asyncio]`, `alembic`, `pgbouncer`

-----

## ChromaDB

> Vector store. Holds embeddings for chore patterns, calendar context, feedback signals. Client/server prod, embedded local.

- Separate StatefulSet pod in prod — never embedded alongside API process
- One client wrapper `chroma_client.py` — no direct SDK calls in business logic
- Collections: `user_chore_patterns`, `calendar_context`, `user_feedback_signals`
- Configurable embedding model — default `all-MiniLM-L6-v2`, swappable via config
- Local dev: embedded mode, path from `~/.appname/config.yaml`
- **Constraint:** All calls through wrapper — zero raw client calls in routers
- **Constraint:** Collection names are constants, never runtime strings (injection risk)

> **Python libs:** `chromadb`, `sentence-transformers`

-----

## Redis

> Fast ephemeral store. Sessions, cache, event bus, Celery broker, rate limits. Five jobs, one process.

- Sessions: refresh tokens keyed `sess:{userId}:{tokenId}`, TTL 30 days
- Cache: API responses keyed `cache:{route}:{paramHash}`, TTL 60s–5min
- Event bus: Redis Streams as `EventBus` abstraction — Kafka-ready interface
- Celery broker: job queue for Monte Carlo runs and long data tasks
- Rate limiting: counters keyed `rl:{ip}:{action}`, TTL 15 min
- WebSocket message rate limiting: per-connection counter via Redis
- **Constraint:** All key patterns defined as constants — no freeform string keys
- **Constraint:** TTL on every key — nothing lives forever
- **Constraint:** EventBus interface only — no direct `redis.xadd` outside abstraction

> **Python libs:** `redis[asyncio]`, `celery[redis]`

-----

## k3s Cluster

> Kubernetes, lightweight. Runs everything. Same manifests local and prod. Self-healing by default.

- Namespaces: `prod`, `staging`, `monitoring`, `infra`
- All workloads: liveness + readiness probes required, no exceptions
- RollingUpdate: `maxUnavailable: 0`, `maxSurge: 1` — zero-downtime deploys
- TopologySpreadConstraints on all Deployments — pods spread across nodes
- StatefulSets for PostgreSQL, ChromaDB, Redis — persistent volume claims per pod
- Network policies: pods talk only to explicitly whitelisted peers
- All pods: non-root user, read-only root filesystem, dropped capabilities
- **Constraint:** No `latest` image tags — always pin to git SHA
- **Constraint:** Resource `requests` and `limits` on every container (NASA rule 2)
- **Constraint:** No `kubectl apply` in prod manually — Helm only

-----

## Vault

> Secrets management. Lease-based rotation. No secret lives forever. All services pull credentials at startup, not from env files.

- Self-hosted HashiCorp Vault in `infra` namespace, HA mode with Raft storage
- All DB passwords, API keys, and tokens issued as leases with TTL — auto-rotated on expiry
- Services use Vault Agent sidecar — injects secrets as environment variables at pod start
- External Secrets Operator syncs Vault secrets into K8s Secrets for Helm compatibility
- Dynamic secrets for PostgreSQL: Vault generates a unique DB user per pod on startup, revokes on shutdown
- Audit log enabled — every secret access logged to Loki
- **Constraint:** No static long-lived secrets anywhere in the cluster — lease TTL enforced
- **Constraint:** Vault seal key never stored in cluster — unsealed via cloud KMS (AWS KMS / GCP CKMS)
- **Constraint:** Secret access logged with `trace_id` — every credential use is attributable

-----

## Linkerd (mTLS)

> Service mesh. Encrypts all pod-to-pod traffic. Mutual TLS enforced inside cluster. Compromised pod cannot sniff peers.

- Linkerd installed in `infra` namespace — lightweight, Rust data plane, < 10ms latency overhead
- mTLS automatic for all meshed pods — no application code changes required
- Traffic policy: deny by default, explicit allow per service pair via `AuthorizationPolicy`
- Observability: Linkerd Viz exposes golden metrics (success rate, RPS, latency) per route — feeds Prometheus
- Tap: live traffic inspection per pod for debugging — requires RBAC, disabled in prod by default
- **Constraint:** All application pods annotated `linkerd.io/inject: enabled` — no unmeshed pods in prod
- **Constraint:** `AuthorizationPolicy` required for every cross-service route — no implicit allow
- **Constraint:** Linkerd control plane certificates rotated automatically via cert-manager

-----

## Falco

> Runtime container security. Watches syscalls inside running pods. Detects anomalous behavior post-deploy.

- Deployed as DaemonSet in `infra` namespace — one Falco pod per node
- Rules: shell spawn inside container, unexpected outbound connection, privilege escalation attempt, `/etc/passwd` read, unexpected file write to sensitive paths
- Alerts routed to Alertmanager → Slack (warning) / PagerDuty (critical)
- Custom rules added for application-specific anomalies: unexpected Postgres connection from non-API pod, Redis access from outside allowed services
- FalcoSidekick forwards events to Loki for correlation with application logs via `trace_id`
- **Constraint:** Default ruleset enabled plus custom rules — no Falco with empty ruleset
- **Constraint:** Falco alerts treated as incident triggers — every alert has a runbook (see IR Playbook)
- **Constraint:** Rule changes version-controlled and reviewed — no ad hoc rule edits in prod

-----

## kube-bench

> CIS Kubernetes Benchmark scanner. Audits cluster configuration against hardening standard. Runs on schedule.

- Runs as a Job in `infra` namespace on weekly schedule via `pg_cron` equivalent (K8s CronJob)
- Checks: API server flags, etcd security, kubelet config, RBAC posture, network policies, pod security standards
- Results stored as K8s ConfigMap and forwarded to Loki
- Failures above threshold trigger Alertmanager warning
- **Constraint:** Run after every k3s version upgrade — not just on schedule
- **Constraint:** FAIL results on critical checks block next prod deploy via CI gate
- **Constraint:** Benchmark results reviewed in post-mortem if a security incident occurs

-----

## Ollama

> Local LLM runtime. Developer machine only. Managed via CLI. Never in prod cluster.

- Runs as local daemon at `http://localhost:11434`
- Model path set in `~/.appname/config.yaml`
- CLI: `appname models list`, `pull`, `set`, `remove`
- LLM service points to Ollama base URL in local env — zero code difference from prod
- **Constraint:** Never runs inside k3s in prod — vLLM only on GPU nodes
- **Constraint:** No model names hardcoded — always read from config

-----

## vLLM

> Prod LLM runtime. GPU node pool. OpenAI-compatible interface. Handles concurrent inference load.

- Dedicated GPU node pool — tainted, only LLM service pods scheduled there
- Exposes `/v1/chat/completions` — same interface as Ollama
- Request batching enabled — higher GPU utilisation, lower per-request latency
- Scale-to-zero on GPU nodes when idle
- **Constraint:** LLM service calls vLLM through provider abstraction only
- **Constraint:** Inference timeout enforced at 120s (NASA rule 2)
- **Constraint:** Model validated via health check before pod reaches Ready

> **Python libs:** `vllm`, `torch`

-----

## PyTorch

> Inference runtime backing vLLM. Available in data service for custom embeddings and fine-tuned models.

- vLLM runs on PyTorch — no direct torch calls in LLM service application code
- Data service uses PyTorch for custom embedding generation if needed
- GPU availability detected at startup — graceful CPU fallback with logged warning
- Model weights loaded once at startup, never per request (NASA rule 2)
- **Constraint:** No model loading inside request handlers — startup only
- **Constraint:** CUDA OOM errors caught explicitly — must not crash pod silently
- **Constraint:** Model weights pulled from S3 or model registry — never committed to git

> **Python libs:** `torch`, `torchvision`, `torchaudio`

-----

## Pydantic v2

> Validation layer for all Python. FastAPI depends on it. Standalone for config schemas and domain models.

- All request/response schemas in `schemas/` — separate from ORM models
- `model_validator` for cross-field validation — never validate in route handlers
- Config via `pydantic-settings` — reads env vars, type-checked at startup
- Shared base models for `created_at`, `updated_at`, `id`
- **Constraint:** No raw `dict` between service layers — always a typed Pydantic model
- **Constraint:** `model_config = ConfigDict(strict=True)` on all schemas — no type coercion
- **Constraint:** Never use ORM models as API response types — always map to schema first (data leakage)

> **Python libs:** `pydantic[email]>=2`, `pydantic-settings`

-----

## Uvicorn

> ASGI server. Runs FastAPI. Sits behind Nginx. Worker count and proxy config are security surface.

- Run with `--proxy-headers --forwarded-allow-ips` scoped to Nginx pod CIDR
- Worker count: `(2 × CPU cores) + 1` — set via env var, not hardcoded
- Timeout: 30s standard routes, 120s LLM routes — enforced at Nginx and Uvicorn
- **Constraint:** Never exposed directly to internet — Nginx Ingress only
- **Constraint:** Worker count via environment variable — no hardcoded value in `CMD`
- **Constraint:** `--forwarded-allow-ips` never wildcard outside trusted cluster network

> **Python libs:** `uvicorn[standard]`, `gunicorn` (prod process manager)

-----

## Core API

> Main FastAPI service. Auth, users, calendar, events. Everything the frontend calls first.

- Domain modules: `auth`, `users`, `calendars`, `events`, `chores` — one router per domain
- All inputs validated at router boundary via Pydantic v2 — nothing raw reaches service layer
- JWT access token 15 min TTL, rotating refresh token 30 days in Redis
- Cursor-based pagination everywhere — no `OFFSET` queries
- Presigned S3 URLs for file uploads — files never proxy through API memory
- RBAC via FastAPI dependency injection — role checked before handler executes
- Structured JSON logs every request: `trace_id`, `user_id`, `route`, `duration_ms`, `status`
- **Constraint:** Max function length 60 lines — extract to service layer if longer (NASA rule 1)
- **Constraint:** No bare `except` — catch specific exceptions, log, re-raise (NASA rule 5)
- **Constraint:** No business logic in routers — routers → services → repositories
- **Constraint:** Secrets from env vars only — never hardcoded, never logged
- **Constraint:** `trace_id` on every log line and response header

-----

## Data Service

> FastAPI + pandas. Ingestion, transformation, exports, Monte Carlo trigger, ChromaDB writes.

- Ingestion: validate schema → queue Celery task → immediate response
- Transformation pipelines in `pipelines/` — pure functions, no side effects
- Long exports: Celery tasks → S3, metadata in Postgres
- ChromaDB writes: feedback embeddings, event context, chore pattern vectors
- **Constraint:** Pipelines are pure functions — no I/O inside transform logic (NASA rule 3)
- **Constraint:** No nested loops in data processing — vectorised pandas ops
- **Constraint:** Max pipeline function length 60 lines (NASA rule 1)
- **Constraint:** All Celery tasks idempotent — safe to retry without duplicate side effects
- **Constraint:** Validate input before any processing — reject early, never halfway

> **Python libs:** `pandas`, `numpy`, `celery[redis]`, `boto3`

-----

## Monte Carlo Scheduler

> Pure function pipeline. No I/O. Seeded, reproducible. Runs as Celery task. Streams progress via WebSocket.

- Input: chore templates, existing events, preferences, feedback weights, time window
- Each iteration: sample free slots → score by preference + conflict + feedback weight
- Weight update rule: `weight += lr * signal` where `completed=+1`, `skipped=−0.5`, `too_early/late=±0.3`
- Weight updates in separate `update_weights()` — never inside the MC loop
- Output: `ScheduledSlot[]` ranked by `confidence_score` (0.0–1.0)
- Seed stored with `ScheduleRun` — every run reproducible from stored parameters
- Progress streamed via `WS /ws/schedule/{run_id}` as `ScheduleProgressPayload`
- **Constraint:** Pure function — no DB calls, no HTTP inside MC loop (NASA rule 3)
- **Constraint:** Serialisable inputs only — snapshot passed in, not live queries
- **Constraint:** Max 10,000 iterations default — configurable via `UserPreferences`, bounded (NASA rule 2)

> **Python libs:** `numpy` (random sampling), `scipy` (optional distribution fitting)

-----

## LLM Service

> FastAPI proxy. One interface. Two backends. Swap provider via config, zero app code change.

- Single router: `/v1/chat/completions` — OpenAI-compatible
- Provider abstraction: `base.py` interface, `ollama.py` and `vllm.py` implementations
- Active provider resolved at startup from config — no runtime switching
- Timebox inference: given task + existing events → propose optimal slot
- Conflict resolution: given overlapping events → propose reordering
- Rate limit queue via Redis — controls concurrent inference per user
- Token stream via `WS /ws/ai/stream/{session_id}` as `AITokenPayload`
- **Constraint:** No provider-specific code outside `providers/` directory
- **Constraint:** Prompt content never logged — only SHA-256 hash stored in `ai_sessions`
- **Constraint:** Provider interface only import in routers — abstraction never bypassed
- **Constraint:** All prompts pass through `PromptSanitiser` middleware before forwarding — no bypass

> **Python libs:** `httpx` (async HTTP to Ollama/vLLM), `tiktoken` (token counting)

-----

## Prompt Injection Middleware

> Sanitisation layer between user input and LLM. Runs before every inference request. No prompt reaches the model without passing this.

- Implemented as FastAPI middleware in `llm-service/middleware/prompt_sanitiser.py`
- Strip control characters: `\x00`–`\x1F` except `\n`, `\t` — removes jailbreak escape sequences
- Enforce max token count: `tiktoken` encodes prompt, rejects if over configured limit (default 4096 tokens)
- Pattern detection: regex patterns for known injection signatures (`ignore previous instructions`, `you are now`, `system:`) — configurable blocklist in `config.yaml`
- System prompt isolation: user input never concatenated directly with system prompt — always inserted into a typed message structure
- Injection attempts logged as security events to Loki with `trace_id` and user_id — never the prompt content
- **Constraint:** Middleware runs before provider dispatch — cannot be short-circuited by route handlers
- **Constraint:** Pattern blocklist version-controlled — changes require PR review
- **Constraint:** Failed sanitisation returns `400` with generic message — no hint about which rule triggered

-----

## WebSockets

> Native FastAPI WS handler server-side. Native browser WebSocket API client-side. No Socket.io.

- FastAPI: `@app.websocket("/ws/{type}/{id}")` — one handler per connection type
- Auth: JWT query param on handshake — validated before upgrade completes, rejected if invalid
- Token re-validation on each inbound message — JWT claims checked against Redis session store
- Explicit re-auth frame: when access token TTL expires mid-connection, server sends `auth_required` frame — client must send fresh token within 30s or connection is closed
- Connection registry in Redis — tracks active connections for fan-out
- Graceful close: server sends close frame, client reconnects with exponential backoff
- Application-layer message rate limiting via Redis counter per connection
- **Constraint:** No unauthenticated connections — reject before upgrade
- **Constraint:** Re-auth frame mandatory on token expiry — no grace period for stale tokens on long sessions
- **Constraint:** Message size limit enforced server-side — disconnect on oversized frame (DoS)
- **Constraint:** All WS payloads typed with Pydantic models — no raw JSON blobs

> **Python libs:** Starlette WebSocket (built into FastAPI), `websockets` for standalone async clients

-----

## GitHub Actions

> CI pipeline. Every PR and push to main. Sequential gates — fail stops everything.

- Trigger: PR open/update + push to `main`
- All stages sequential — failure stops pipeline, no skipping
- Docker build only after all gates pass — never speculative
- **Constraint:** No secrets in workflow YAML — GitHub Actions encrypted secrets only
- **Constraint:** All action versions pinned to SHA — never `@main` or `@latest`
- **Constraint:** Human approval gate on prod deploy only — everything below fully automated

-----

## Ruff Gate

> First gate. Seconds to run. Lint, format, types. Bad code stopped before tests consume time.

- `ruff check` — zero warnings allowed
- `ruff format --check` — format enforced, no debate
- `mypy --strict` — no implicit `Any`
- **Constraint:** `# noqa` suppression requires inline comment — no silent suppression
- **Constraint:** `type: ignore` banned except in third-party shims — exceptions in ADR

-----

## pytest + pytest-cov

> Test gate. Unit and integration. 80% coverage floor. Integration runs real Postgres and Redis.

- Unit: pure functions, mocked I/O — fast, run first
- Integration: Docker Compose spins Postgres + Redis — real connections
- Coverage gate: 80% minimum on application code only
- Report uploaded to SonarQube for trend tracking
- **Constraint:** No sleeping tests — event-driven assertions or mocked timers (NASA rule 2)
- **Constraint:** Every public function has at least one test
- **Constraint:** Tests deterministic — no randomness, no time-dependent assertions without mocking

-----

## SonarQube

> Static analysis. Duplication, smells, hotspots. Hard gate — merge blocked on failure.

- Self-hosted in `monitoring` namespace, dedicated Postgres, ~2GB RAM
- Quality gate: duplication < 3%, zero critical issues, zero unreviewed hotspots
- Results as PR comment — visible before merge
- **Constraint:** Hard CI failure — not advisory
- **Constraint:** Security hotspots reviewed and marked — none left open in main

-----

## Trivy

> Container CVE scan. Every image, every build. CRITICAL = hard fail, image never pushed.

- Scans every Docker image after build, before registry push
- HIGH CVEs: reported, resolved within 7 days
- Scan results stored as CI artefact per run
- **Constraint:** No base images without known provenance
- **Constraint:** Images rebuilt weekly — catches new CVEs in unchanged code

-----

## Gitleaks

> Secrets scanner. Every commit. Credential in code equals immediate pipeline stop.

- Scans all committed files for tokens, keys, credentials
- Custom rules for internal formats in `.gitleaks.toml`
- Pre-commit hook via `pre-commit` — catches before push
- **Constraint:** Suppression requires `.gitleaksignore` entry with ticket reference
- **Constraint:** Pre-commit hook mandatory for all contributors

-----

## pip-audit / npm audit

> Dependency CVE scan. Python and Node. High and critical block the build.

- `pip-audit` on all Python dependency files
- `npm audit --audit-level=high` on all Node packages
- **Constraint:** Pinned versions in all lockfiles — no floating `^` in prod deps
- **Constraint:** Dependabot enabled — weekly automated update PRs

-----

## pip-licenses / license-checker

> Dependency licence scan. Python and Node. GPL contamination caught before it ships.

- `pip-licenses --format=json` on all Python environments — outputs licence per package
- `license-checker --json` on all Node `package.json` files
- Blocklist: `GPL-2.0`, `GPL-3.0`, `AGPL-3.0`, `LGPL-2.1` — configurable per project
- Allowlist: `MIT`, `Apache-2.0`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`, `0BSD`
- Licence report stored as CI artefact per run — queryable for compliance audit
- Pipeline fails on any package matching blocklist
- **Constraint:** Runs after pip-audit / npm audit — same stage, no additional overhead
- **Constraint:** Licence blocklist version-controlled — changes require legal review comment in PR
- **Constraint:** New dependencies reviewed for licence before merge — automated gate is a safety net, not a substitute

-----

## Docker Build

> Multi-stage. Non-root user. Minimal prod image. Tagged by git SHA. No latest, ever.

- Stages: `base` → `test` → `prod` — test layer never ships
- Non-root user before `CMD`
- `.dockerignore` excludes `.git`, secrets, dev deps
- **Constraint:** No secrets in Dockerfile layers — build args only
- **Constraint:** `COPY` specific paths — never `COPY . .` in prod stage
- **Constraint:** Image size alerted if > 200MB Python / 50MB Node

-----

## syft (SBOM)

> Software Bill of Materials. Generated from every Docker image after build. Inventory of what ships.

- `syft` runs on every built image — outputs SBOM in SPDX and CycloneDX formats
- SBOM stored as CI artefact alongside image SHA — permanent audit record
- Enables rapid response to zero-days: query SBOM to find which images contain a vulnerable package without rebuilding
- SBOM uploaded to container registry as image attestation — verifiable at deploy time
- **Constraint:** SBOM generation is a required CI step — build without SBOM does not push to registry
- **Constraint:** SBOM format: both SPDX (tooling compatibility) and CycloneDX (Dependency-Track ingestion)
- **Constraint:** SBOM retained for the lifetime of the image — not pruned when image is pruned

-----

## Container Registry

> Image storage. SHA-tagged. Helm values reference explicit tag per environment.

- Images pushed only after all gates pass — SBOM attestation attached at push time
- Tags: `{service}:{git-sha}` — no latest, no semver
- Helm `values.yaml` per env references explicit SHA
- Registry scanning enabled — secondary CVE pass on stored images
- **Constraint:** Old images pruned on schedule — SBOM retained separately before prune
- **Constraint:** Pull secrets in K8s Secrets — never in Helm values files

-----

## Nginx Ingress

> Traffic entry. TLS, routing, rate limiting, CORS. Nothing reaches services without passing here.

- TLS 1.3 — cert-manager + Let’s Encrypt, auto-renews
- Routes: `/api/v1/*` → Core API, `/api/data/*` → Data Service, `/api/llm/*` → LLM Service, `/*` → Frontend
- Rate limiting: 100 req/min per IP, configurable per route
- CORS: explicit allowlist — no wildcard `*` in production
- Body limit: 50MB — blocks oversized payload attacks
- **Constraint:** No service reachable from outside cluster except through Nginx
- **Constraint:** Security headers on all responses: `HSTS`, `X-Frame-Options`, `X-Content-Type-Options`, `CSP`
- **Constraint:** Access logs structured JSON — every request traceable by `trace_id`

-----

## Prometheus + Grafana

> Metrics and dashboards. All services scraped. Alertmanager pages on SLO breach.

- `prometheus-fastapi-instrumentator` auto-instruments all FastAPI services
- SLOs alerted: API p99 > 500ms, error rate > 0.1%, LLM p95 > 30s, MC run > 60s
- Alertmanager: Slack (warning), PagerDuty (critical)
- Linkerd Viz metrics ingested alongside application metrics — golden signals per service route
- Falco alert events forwarded to Alertmanager — runtime security on same paging path as SLO breaches
- **Constraint:** No alert without a runbook link
- **Constraint:** Dashboards in version control — never clicked together in UI

-----

## OWASP ZAP (DAST)

> Dynamic application security testing. Runs against live staging after every deploy. Finds runtime issues SAST cannot.

- OWASP ZAP runs as a GitHub Actions step after `deploy-staging` completes
- Passive scan: crawls all API routes, flags information leakage, missing headers, insecure cookies
- Active scan: targeted attack simulation on authenticated endpoints — SQL injection, XSS, path traversal
- Auth: ZAP configured with a test-user JWT — scans authenticated routes, not just public surface
- Results stored as CI artefact (HTML + JSON report) per run
- Pipeline fails if any HIGH or CRITICAL finding is new — existing known findings suppressed via `.zap/false-positives.yaml`
- **Constraint:** ZAP runs against staging only — never prod
- **Constraint:** New HIGH/CRITICAL findings block deploy-prod gate — must be triaged before promotion
- **Constraint:** False-positive suppression file version-controlled — no silent ignores without PR review

-----

## Loki + Promtail

> Log aggregation. Structured JSON only. Every pod, one place, 30 days hot.

- Promtail daemonset scrapes all pod logs automatically
- Required fields: `timestamp`, `level`, `service`, `trace_id`, `message`
- Retention: 30 days hot, 90 days cold on S3
- Falco events and Vault audit logs ingested into Loki alongside application logs
- **Constraint:** No PII in logs — user IDs only, never email, name, payload
- **Constraint:** `trace_id` mandatory — logs without it alerted as misconfigured

-----

## S3 Backup Encryption

> All backups encrypted at rest. SSE-KMS on backup bucket. Restore drill on schedule.

- Dedicated S3 bucket for PostgreSQL daily snapshots — SSE-KMS encryption, AWS KMS or GCP CKMS
- Bucket policy: no public access, no cross-account access without explicit policy
- Versioning enabled — accidental overwrite recoverable
- Lifecycle policy: 14 days standard storage (aligned to retention window), then deleted — no indefinite backup accumulation
- Monthly restore drill: automated job restores latest backup to isolated RDS instance, runs schema validation, reports success/failure to Slack
- **Constraint:** Backup bucket never accessible from application pods — separate IAM role, separate credentials
- **Constraint:** KMS key rotation enabled — annual automatic rotation
- **Constraint:** Restore drill failure triggers PagerDuty alert — backup that cannot restore is not a backup

-----

## Sentry

> Error tracking. All unhandled exceptions, frontend and backend, with full context.

- FastAPI: captures unhandled exceptions with request context and stack trace
- React: captures unhandled JS errors and promise rejections
- `trace_id` on every event — correlates with logs and Prometheus
- `before_send` strips sensitive fields before transmission
- **Constraint:** DSN in K8s Secret / env var — never in source code
- **Constraint:** No passwords, tokens, or PII in payloads — scrubbing configured and tested

-----

## Incident Response Playbook

> Defined response process for every alert. Detection to post-mortem. No on-call engineer improvises.

- Every Alertmanager rule links to a runbook in `docs/runbooks/` — version-controlled, reviewed quarterly
- IR phases:
1. **Detection** — alert fires, on-call paged via PagerDuty, acknowledge within 5 min
1. **Triage** — severity assessed: P1 (service down), P2 (degraded), P3 (non-user-facing)
1. **Containment** — isolate affected pod/service: scale to 0, network policy block, or rollback via Helm
1. **Eradication** — root cause identified, fix applied to staging, gates pass
1. **Recovery** — deploy fix to prod, verify SLOs restored, monitor for 30 min
1. **Post-mortem** — blameless, written within 48 hours, action items tracked in issues
- Falco triggers: immediate containment — affected pod isolated, network policy applied, security team notified
- Vault breach: all leases revoked immediately, dynamic secrets rotated, audit log pulled
- Data breach protocol: legal notified within 24 hours, users notified per jurisdiction requirements
- **Constraint:** Every runbook tested in tabletop exercise quarterly
- **Constraint:** Post-mortem action items tracked to completion — no open items > 30 days
- **Constraint:** IR playbook itself reviewed after every P1 incident — process improves with each event

-----

## Local Config

> One file controls local environment. `appname init` writes it. Users never hand-edit.

- Generated by `appname init` on first run
- Controls: LLM provider, model, base URL, Postgres URL, ChromaDB mode and path
- `appname config set <key> <value>` for all changes
- App validates config at startup — refuses to start on invalid or missing fields
- **Constraint:** `.gitignore` entry required — never committed
- **Constraint:** Credentials in env vars only — config holds paths and URLs, not secrets

-----

## Zod + Client-Side Validation

> Validate before submit. Zod mirrors Pydantic. No request leaves without passing a schema.

- Schemas in `packages/types/schemas/` — generated from FastAPI OpenAPI spec via `openapi-ts`
- React Hook Form `resolver` wired to Zod — errors at field level, not on submit
- Runtime response validation: all API responses parsed through Zod on receipt
- Regex: named `.regex()` refinements in `lib/patterns.ts` — no inline regex in components
- **Constraint:** No regex literals in component files — all patterns named exports
- **Constraint:** `z.any()` banned in production schemas — every field typed explicitly
- **Constraint:** All `.parse()` calls in try/catch — Zod errors to Sentry, generic message to user

-----

## Client-Side Cryptography

> Encrypt before send. ECDH key exchange. PBKDF2 derivation. PQC forward secrecy. SubtleCrypto only.

- **ECDH P-256:** ephemeral key exchange, new keypair per session — `SubtleCrypto.deriveKey`
- **PBKDF2:** 600,000 iterations, SHA-256, 32-byte output — run in Web Worker, never blocks main thread
- **ML-KEM (Kyber):** `ml-kem` WASM lib — `SubtleCrypto` has no native PQC yet
- **Hybrid:** ECDH + ML-KEM shared secrets XOR’d — classical and post-quantum in parallel
- **AES-256-GCM:** authenticated encryption, unique nonce per message
- Keys in memory only — never `localStorage`, never `sessionStorage`, cleared on tab close
- **Constraint:** `SubtleCrypto` only for standard ops — no pure-JS crypto fallbacks
- **Constraint:** PBKDF2 iteration count stored alongside key material — supports future re-derivation
- **Constraint:** All crypto in `lib/crypto/` module — no inline crypto in components
- **Constraint:** Server-side crypto via Python `cryptography` lib — high-level API preferred over `hazmat`

> **Python libs:** `cryptography`

-----

## Tailwind + Design System

> Utility classes for layout. CSS custom properties for all tokens. shadcn/ui owned in repo.

- shadcn/ui components copied into `packages/ui/` — owned and modified, not imported from npm
- CSS custom properties for all tokens: `--radius`, `--color-primary`, `--color-surface`, `--spacing-unit`, `--font-size-base`, `--duration-fast`, `--duration-normal`
- Tailwind config extends tokens — `bg-primary` maps to `var(--color-primary)`, never raw hex
- Dark mode via `class` strategy — `dark:` variants
- Component variants via `cva()` — no conditional string concatenation for class logic
- **Constraint:** No hardcoded hex, rgb, or pixel values in component files — tokens only
- **Constraint:** No arbitrary Tailwind `[]` values for spacing or color — add to token system first
- **Constraint:** `border-radius` always via `--radius` — never `rounded-[6px]`
- **Constraint:** No inline styles — Tailwind or CSS variables only, never `style={{}}`

-----

## React Frontend

> React 19, TypeScript strict, Tailwind, shadcn/ui. Built last. Renders only — no logic, no data.

- Strict TypeScript — no `any`, all API types from OpenAPI spec via `openapi-ts`
- TanStack Query for all server state — no manual `fetch` in components
- Zustand for client-only UI state — never cache server data here
- Code split per route — initial bundle < 150KB gzipped
- Error boundary per route — one crash never downs the app
- TanStack Virtual for all large lists and calendar grids — no DOM thrash
- **Constraint:** No business logic in components — components render, hooks fetch, utils compute
- **Constraint:** All user inputs sanitised before display — never render raw API strings as HTML (XSS)
- **Constraint:** No `console.log` in committed code — remove before merge
- **Constraint:** Accessible by default — keyboard navigable, ARIA labels on icon-only elements
- **Constraint:** No inline styles — Tailwind or CSS variables only

-----

## Postgres Data Retention

> Ephemeral by default. Soft delete on action. Hard purge 14 days later. Recovery window is 14 days exactly.

- Soft delete: `deleted_at = NOW()` — record invisible to application immediately
- Hard purge: `pg_cron` nightly at 03:00 UTC — `DELETE WHERE deleted_at < NOW() - INTERVAL '14 days'`
- Purge logged to `purge_audit(table, record_id, deleted_at, purged_at)` before execution
- User-facing: “data permanently removed after 14 days” stated explicitly in UI
- Backups: daily snapshots retained 14 days — aligned to recovery window
- **Constraint:** Purge job idempotent — safe to re-run
- **Constraint:** Audit log retained 90 days — record gone, fact of deletion is not
- **Constraint:** No hard deletes in application code — forbidden, purge job only
- **Constraint:** Purge job monitored via Prometheus job metric — failure alerts on-call

-----

## DevSecOps & OpSec Gap Analysis

### Covered

|Area                  |Implementation                              |
|----------------------|--------------------------------------------|
|SAST                  |SonarQube + Ruff + mypy                     |
|Container scan (build)|Trivy                                       |
|Dependency CVEs       |pip-audit + npm audit                       |
|Secrets detection     |Gitleaks (CI + pre-commit)                  |
|Supply chain          |Pinned action SHAs                          |
|Auth (HTTP)           |JWT short TTL + rotating refresh            |
|Auth (WebSocket)      |JWT on handshake + per-message re-validation|
|Rate limiting (HTTP)  |Nginx                                       |
|Rate limiting (WS)    |Redis app-layer counter                     |
|Input validation      |Pydantic v2 strict + Zod                    |
|SQL injection         |ORM + parameterised queries only            |
|Client crypto         |ECDH + PBKDF2 + ML-KEM + AES-256-GCM        |
|PII in logs           |Structured logs, no PII policy              |
|Prompt privacy        |Prompt hash only, never stored raw          |

### Coverage Summary

All previously identified air gaps are now first-class sections in this document.
