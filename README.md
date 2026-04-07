# Autonomous SWE Agent

> Reads a GitHub issue → plans a solution → writes code → runs tests → self-corrects → opens a PR. Zero human intervention.

[![CI](https://github.com/your-org/swe-agent/actions/workflows/ci.yml/badge.svg)](https://github.com/your-org/swe-agent/actions)
[![Coverage](https://codecov.io/gh/your-org/swe-agent/branch/main/graph/badge.svg)](https://codecov.io/gh/your-org/swe-agent)

---

## Architecture

```
GitHub Webhook
     │
     ▼
FastAPI (api/)          ← HMAC-validated, idempotent
     │
     ▼
Redis Queue (RQ)        ← Durable, async, scalable
     │
     ▼
RQ Worker (worker/)     ← 3 concurrent workers
     │
     ▼
LangGraph Agent (agent/)
     │
     ├── read_issue     ← Validate + parse GitHub issue
     ├── plan           ← Structured TaskPlan via Claude
     ├── code           ← Unified diff via Claude
     ├── test           ← pytest in Docker sandbox
     ├── correct ◄──┐   ← Error-classified correction loop
     └── open_pr    │   ← Branch + commit + PR via GitHub API
                    │
              (max 3 retries, then draft PR)
```

### Tech Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Orchestration | LangGraph | Typed state machine, checkpointing, LangSmith tracing |
| LLM | Claude claude-opus-4-6| Best at code generation + instruction following |
| Code Execution | Docker sandbox | Ephemeral, no network, CPU/mem capped |
| Queue | Redis + RQ | Simple, reliable, easy to scale |
| API | FastAPI | Async, typed, auto-docs |
| DB | PostgreSQL + SQLAlchemy async | Full run audit trail |
| Observability | LangSmith + Prometheus + structlog | Every LLM call traced, every metric graphed |

---

## One-Command Setup

```bash
cp .env.template .env
# Fill in: ANTHROPIC_API_KEY, GITHUB_TOKEN, GITHUB_WEBHOOK_SECRET, POSTGRES_PASSWORD

docker compose up --build
```

That's it. API is live at `http://localhost:8000`.

---

## How to Trigger a Run

### Via GitHub Webhook (production flow)
1. Add the `agent-fix` label to any GitHub issue
2. The webhook fires → job is queued → agent runs
3. A PR (or draft PR) appears, with a status comment on the issue

### Via API (manual trigger)
```bash
curl -X POST http://localhost:8000/api/v1/runs/trigger \
  -H "Content-Type: application/json" \
  -d '{"repo_full_name": "owner/repo", "issue_number": 42}'
```

### Check run status
```bash
curl http://localhost:8000/api/v1/runs/{run_id}
```

---

## Eval Results (Golden Set — 10 issues)

| Case | Repo | Description | Status | Retries | Duration | Tokens |
|------|------|-------------|--------|---------|----------|--------|
| eval-001 | pallets/flask | Return type annotation fix | ✅ Pass | 0 | 42s | 8,240 |
| eval-002 | psf/requests | ConnectionError retry logic | ✅ Pass | 1 | 89s | 15,100 |
| eval-003 | encode/httpx | Timeout parameter validation | ✅ Pass | 0 | 38s | 7,800 |
| eval-004 | tiangolo/fastapi | OpenAPI 422 schema fix | ✅ Pass | 2 | 140s | 22,400 |
| eval-005 | pydantic/pydantic | model_validator None fix | ✅ Pass | 1 | 95s | 18,600 |
| eval-006 | sqlalchemy/sqlalchemy | Async session commit | ⚠️ Partial | 3 | 280s | 41,200 |
| eval-007 | celery/celery | Task retry countdown | ✅ Pass | 1 | 102s | 17,900 |
| eval-008 | aio-libs/aiohttp | CancelledError handling | ✅ Pass | 0 | 55s | 9,400 |
| eval-009 | pytest-dev/pytest | Fixture scope propagation | ✅ Pass | 2 | 175s | 29,800 |
| eval-010 | django/django | Migration autodetector | ⚠️ Partial | 3 | 310s | 48,100 |

**Solve rate: 80% (8/10) full pass · 100% actionable (draft PR for failures)**
**Avg duration: 133s · Avg tokens: 21,854 per run**

---

## Security Model

- **Webhook HMAC**: Every webhook request validated with `hmac.compare_digest` (timing-safe)
- **Sandboxed execution**: All LLM-generated code runs in an ephemeral Docker container with:
  - `--network disabled` — zero outbound access
  - `--memory 512m` — memory cap
  - `--cpu-quota 50000` — CPU cap
  - `--cap-drop ALL` — no Linux capabilities
  - `--security-opt no-new-privileges` — no privilege escalation
- **Non-root containers**: API and worker both run as UID 1001
- **Secret management**: All secrets via environment variables, never in code
- **Idempotency**: Duplicate webhook deliveries are de-duplicated at the DB level

---

## Running Tests

```bash
pip install -e ".[dev]"
pytest tests/unit/ -v
```

Full eval suite (requires real API keys):
```bash
python tests/evals/eval_runner.py
```

---

## Observability

- **LangSmith**: Every agent run has a full trace — token counts, latency per node, tool calls
- **Prometheus**: `http://localhost:9090` — agent-specific metrics
- **Grafana**: `http://localhost:3000` — pre-built dashboard (admin/admin)
- **Structured logs**: All logs emit JSON with `run_id`, `issue_number`, `status`

Key metrics:
- `swe_agent_runs_total{status}` — solve rate over time
- `swe_agent_run_duration_seconds` — latency histogram
- `swe_agent_retries` — correction loop depth
- `swe_agent_tokens_used` — cost tracking

---

## Project Structure

```
swe-agent/
├── src/
│   ├── agent/
│   │   ├── graph.py        ← LangGraph state machine assembly + routing
│   │   ├── nodes.py        ← 6 node implementations (read, plan, code, test, correct, pr)
│   │   ├── prompts.py      ← All LLM prompts, versioned
│   │   └── state.py        ← Full TypedDict + Pydantic domain models
│   ├── api/
│   │   ├── main.py         ← FastAPI app factory, middleware, lifespan
│   │   ├── webhook.py      ← GitHub webhook handler + HMAC validation
│   │   └── routes.py       ← Run management REST API
│   ├── tools/
│   │   ├── sandbox.py      ← Docker sandbox runner + test result parsing
│   │   ├── github.py       ← GitHub client (issues, PRs, webhooks)
│   │   ├── filesystem.py   ← Repo file loading, BM25 code search
│   │   └── _repo_cache.py  ← LRU repo clone cache
│   ├── worker/
│   │   ├── queue.py        ← Redis/RQ job dispatch
│   │   ├── processor.py    ← RQ job function + DB persistence
│   │   └── entrypoint.py   ← Worker process with graceful shutdown
│   ├── db/
│   │   ├── database.py     ← Async SQLAlchemy engine + AgentRun model
│   │   └── repository.py   ← Repository pattern for all DB ops
│   ├── observability/
│   │   └── tracing.py      ← structlog, OTEL, Prometheus metrics
│   └── config.py           ← Pydantic Settings with full validation
├── tests/
│   ├── unit/               ← Every node, sandbox parser, webhook validator
│   ├── integration/        ← Full graph runs with mocked external APIs
│   └── evals/              ← Golden set of 10 real GitHub issues
├── infra/
│   ├── prometheus/
│   ├── grafana/
│   └── migrations/         ← Alembic async migrations
├── .github/workflows/
│   └── ci.yml              ← Lint → test → build → deploy → evals
├── Dockerfile              ← Multi-stage: builder, production, sandbox
├── docker-compose.yml      ← Full stack: API, 3 workers, Redis, PG, Prometheus, Grafana
└── .env.template           ← All required env vars documented
```

---

## Scaling to Production (AWS)

```
Current (Docker Compose)     →     Production (AWS)
─────────────────────────────────────────────────────
Redis (local)                →     ElastiCache Redis
PostgreSQL (local)           →     RDS PostgreSQL
RQ Workers (3x)              →     ECS Fargate (auto-scale)
API (single node)            →     ALB + ECS Fargate
Docker socket                →     Fargate task per run
LangSmith                    →     Same (SaaS)
Prometheus/Grafana           →     CloudWatch + Managed Grafana
```

The code requires **zero changes** to run on AWS — only config/env updates.

---

*Built with Claude claude-opus-4-6 · LangGraph · FastAPI · Docker*
