# EvoAgent

**EvoAgent is a self-hosted, multi-agent pull-request reviewer.** It reads unified diffs, coordinates a hierarchical team of LLM agents, grounds every high-risk claim in repository evidence, and can post its review back to GitHub or open a verified repair PR.

The project is intentionally built on the Python standard library: the HTTP server, agent runtime, persistence layer, GitHub client, and web dashboard do not require a web framework or an orchestration framework.

---

## Highlights

- **Hierarchical agentic review** — a Lead agent delegates to Security and Correctness/Reliability workers, a blind Critic challenges candidate findings, and the Lead performs final synthesis.
- **Evidence-gated findings** — format, evidence, confidence, and release gates reject unsupported findings; high-severity claims must cite tool evidence or a concrete call chain.
- **Repository-grounded tools** — code search, symbol lookup, AST analysis, Git history, test discovery, project controls, and fixed-command scanners, all exposed through a validated tool registry.
- **Bounded autonomy** — per-role token/time/step budgets, one optional revision round for high-risk reviews, and a single-pass path for ordinary changes.
- **Durable runtime** — checkpointed workflow nodes, task cancellation, resume from the last checkpoint, and explicit state transitions.
- **GitHub integration** — HMAC-SHA256 webhook verification, replay protection, delivery idempotency, comment upsert, and optional verified auto-fixes on draft PRs.
- **Agent Skills** — versioned `SKILL.md` packages selected by the Lead at runtime, with per-role tool permissions and sandboxed text resources.
- **Evolution loop** — confirmed feedback can produce prompt or Skill candidates that are replay-evaluated, gated, activated, and rolled back.
- **Production plumbing** — SQLite or PostgreSQL persistence, in-memory or Redis Streams queuing with leases/retries/dead-letter handling, RBAC and tenant isolation, audit logs, OpenTelemetry traces, Prometheus metrics, and alerts.
- **Web control plane** — a dashboard for reviews, task timelines, reports, skills, feedback, and evolution experiments.

---

## Architecture

```text
HTTP API / GitHub Webhook
          |
          v
    ReviewService
      |        \
      |         TaskStore (SQLite / PostgreSQL)
      |         TaskQueue  (in-memory ACK / Redis Streams + DLQ)
      v
    ReviewHarness
      |  EvoAgent Runtime: budgets, retries, cancellation, checkpoints, resume
      |
      +-- DiffParser
      +-- Deterministic rule scanners
      +-- AgenticReviewer
            |
            +-- Lead Agent
            |     - scans diff, classifies risk (low / normal / high)
            |     - assigns work and Agent Skills
            |     - final finding selection and confidence adjustment
            |
            +-- Security Worker
            |     - injection, authorization, secrets, dangerous call chains
            |
            +-- Correctness/Reliability Worker
            |     - state, errors, concurrency, resources, compatibility, tests
            |
            +-- Critic Worker
            |     - blind review, counterexamples, evidence and severity challenges
            |
            +-- FindingGate (format / evidence / confidence / release)
```

### Review flow

1. A diff arrives from `POST /v1/reviews` or a GitHub `pull_request` webhook.
2. `ReviewHarness` parses added lines and moves the task through `PENDING → PLANNING → EXECUTING → REVIEWING → SUCCESS` (or `FAILED` / `CANCELLED`).
3. Deterministic scanners run first. The Lead then classifies risk:
   - **low / normal** — workers receive a one-shot, tool-free assessment.
   - **high** — workers may use repository tools, and the Lead may request at most one revision round.
4. Security and Correctness/Reliability workers run in parallel and report only to the Lead.
5. The Critic receives candidate findings with source identities removed and challenges them independently.
6. The Lead selects the findings to publish.
7. The `FindingGate` checks location validity, exact-line evidence, strong evidence for high severity, confidence, and fix/test guidance.
8. The report is persisted, exposed as JSON or Markdown, and optionally posted back to the PR.

---

## Quick Start

### Requirements

- Python 3.11+
- An OpenAI Chat Completions-compatible LLM endpoint (DeepSeek, OpenRouter, or any custom endpoint)
- Optional: GitHub token / GitHub App for private repositories, PR comments, or fixes
- Optional: PostgreSQL and Redis for production persistence and queuing

### Install and run

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

export EVOAGENT_LLM_PROVIDER=deepseek
export EVOAGENT_DEEPSEEK_API_KEY="<your-api-key>"

python -m evoagent
```

The service listens on `http://127.0.0.1:8080` by default. Open `/` for the dashboard and `/health` for the health check.

### Configure an LLM

DeepSeek official API:

```bash
export EVOAGENT_LLM_PROVIDER=deepseek
export EVOAGENT_DEEPSEEK_API_KEY="<deepseek-api-key>"
```

OpenRouter free model presets:

```bash
export EVOAGENT_LLM_PROVIDER=openrouter-deepseek-free
export EVOAGENT_OPENROUTER_API_KEY="<openrouter-api-key>"
```

Any OpenAI-compatible endpoint:

```bash
export EVOAGENT_LLM_PROVIDER=custom
export EVOAGENT_LLM_BASE_URL="https://example.com/v1"
export EVOAGENT_LLM_API_KEY="<token>"
export EVOAGENT_LLM_MODEL="<model-name>"
```

The service can start without a model (for health checks and the dashboard), but a configured model is required before reviews can run. Secrets are read from environment variables or a local `.env`; never commit them. A root-level `.env` takes priority over `evoagent/.env`, and real environment variables take priority over both.

### Enable authentication

```bash
export EVOAGENT_AUTH_REQUIRED=true
export EVOAGENT_AUTH_SECRET="<at-least-32-random-bytes>"
export EVOAGENT_BOOTSTRAP_ADMIN_USERNAME=admin
export EVOAGENT_BOOTSTRAP_ADMIN_PASSWORD="<at-least-10-characters>"
python -m evoagent
```

The bootstrap admin is only created if the username does not already exist.

### Create a review

```bash
TOKEN=$(curl -s -X POST http://127.0.0.1:8080/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"<password>"}' | python -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')

curl -s -X POST http://127.0.0.1:8080/v1/reviews \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "repository": "demo/api",
    "pull_request": 12,
    "mode": "agentic",
    "diff": "diff --git a/app.py b/app.py\n--- a/app.py\n+++ b/app.py\n@@ -1 +1,2 @@\n+password = \"secret\"\n+eval(user_input)"
  }'
```

Inspect the task and its Markdown report:

```bash
curl -s -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8080/v1/tasks/<task-id>
curl -s -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8080/v1/tasks/<task-id>/report
```

Add `?async=true` to `POST /v1/reviews` to enqueue the review instead of running it synchronously.

---

## GitHub Integration

EvoAgent accepts GitHub repository webhooks directly; no GitHub App is required for the basic flow.

```text
GitHub Pull request event
        |
        v
https://<public-host>/webhooks/github
        |
        v
http://127.0.0.1:8080/webhooks/github
        |
        v
Asynchronous review task
```

### 1. Configure EvoAgent

```bash
export EVOAGENT_GITHUB_WEBHOOK_SECRET="<random-webhook-secret>"
export EVOAGENT_GITHUB_TOKEN="<fine-grained-PAT>"        # optional
export EVOAGENT_AUTO_POST_REVIEW=true                    # optional
python -m evoagent
```

Supported `pull_request` actions are `opened`, `reopened`, and `synchronize`. Other actions are acknowledged and ignored. The webhook route authenticates with its HMAC signature, not with a dashboard bearer token.

Minimum fine-grained PAT permissions:

| Capability | Permissions |
|---|---|
| Read private PR diffs | Contents: Read, Pull requests: Read |
| Post review comments | Pull requests: Read and write |
| Create fix branches/commits | Contents: Read and write, Pull requests: Read and write |

### 2. Expose the webhook endpoint

GitHub cannot reach `127.0.0.1`. Forward a public HTTPS URL to the local service, for example with Cloudflare Tunnel or ngrok:

```bash
cloudflared tunnel --url http://127.0.0.1:8080
# or
ngrok http 8080
```

For long-lived deployments, expose only `/webhooks/github` and optionally `/health` through a reverse proxy instead of publishing the entire dashboard.

### 3. Add the webhook in GitHub

Repository **Settings → Webhooks → Add webhook**:

- **Payload URL**: `https://<public-host>/webhooks/github`
- **Content type**: `application/json`
- **Secret**: the same value as `EVOAGENT_GITHUB_WEBHOOK_SECRET`
- **SSL verification**: enabled
- **Events**: select individual events → **Pull requests**
- **Active**: checked

Successful deliveries return `202`. EvoAgent also supports a GitHub App installation flow (`/github/install` and `/github/setup`) when App credentials are configured.

### Verified auto-fixes

When enabled, EvoAgent can create verified repairs on a new `evoagent/fix-pr-*` branch and open a **draft PR**. It never writes directly to the source branch. Repairs are limited to deterministic rules such as debug output, `shell=True`, and hardcoded Python credentials, and are published only after compilation and optional test gates pass.

---

## Agent Roles and Budgets

| Role | Responsibility | Default tools |
|---|---|---|
| Lead | decomposition, risk classification, revision requests, final synthesis | read-only repository/diff discovery |
| Security | untrusted input, authorization, secrets, dangerous data flows | full repository tool suite (high risk only) |
| Correctness/Reliability | state, exceptions, concurrency, resources, compatibility, tests | full repository tool suite (high risk only) |
| Critic | blind challenge, counterexamples, evidence and severity objections | full repository tool suite (high risk only) |

Roles can be restricted with `EVOAGENT_ENABLED_AGENTS` (the Lead is mandatory). Budgets are configurable per role through `EVOAGENT_AGENT_TOKEN_BUDGET`, `EVOAGENT_AGENT_TIME_BUDGET_SECONDS`, and context-window settings.

### Repository tools

Model-facing tools are registered with strict argument schemas and currently include:

`list_repository`, `search_repository`, `search_diff`, `read_file`, `changed_line`, `symbol`, `locate_tests`, `read_project_controls`, `ast_analyze`, `git_context`, `run_scanners`, `run_repository_checks`, and `read_skill_resource`.

Scanners use fixed commands (Semgrep, Bandit, ESLint, mypy, pyright) and repository checks run in an isolated temporary copy with a restricted environment. Tool outputs become evidence with stable `evidence_id` values that findings can cite.

---

## Agent Skills

A Skill is a prompt-time capability package, not a tool connector. Each package contains a `SKILL.md` with YAML frontmatter and a detailed review protocol:

```text
skills/<skill-name>/
├── SKILL.md          # metadata + instructions
└── ...               # optional text resources
```

```markdown
---
name: security-review
description: Review code changes for authorization failures, injection, secret exposure, ...
allowed-tools:
  - changed_line
  - read_file
  - symbol
  - ast_analyze
---

# Security review
...mission, procedure, checklist, evidence/severity, remediation, output contract...
```

Discovery exposes only `name` and `description` to the Lead. Once selected, the full instructions and narrowed tool permissions are injected into the worker context. Resources are readable only through `read_skill_resource` and cannot escape the package directory.

Bundled skills: `api-compatibility`, `code-quality`, `correctness-review`, `database-review`, `observability-review`, `performance-review`, `reliability-review`, `security-review`, and `test-quality`.

Reload disk skills with `POST /v1/skills/reload`. Versioned Skills can be evaluated, activated, and rolled back through `/v1/skill-evolution/*`.

---

## Evolution and Evaluation

EvoAgent closes the loop between review failures and future reviews:

1. Users report `false_positive`, `missed_issue`, `bad_fix`, or `accepted` feedback.
2. Failure cases can produce prompt candidates or versioned `SKILL.md` artifacts.
3. Candidates are replayed against train/validation/holdout data with repository-disjoint splits.
4. Hard gates require improvement without regressing precision, high-risk recall, or fix safety.
5. Only passing candidates are activated; previous versions remain available for rollback.

The evaluation harness computes precision, recall, F1, high-risk recall, clean-diff accuracy, false-positive behavior, and repair verification results. Matching is canonical CWE/path/line based with deterministic one-to-one assignment.

Useful scripts:

```bash
python scripts/run_controlled_experiments.py
python scripts/run_agentic_evaluation.py --help
python scripts/run_accuracy_experiment.py --help
python scripts/run_skill_evolution_experiment.py --help
python scripts/run_prompt_evolution_proof.py
python scripts/run_real_pr_benchmark.py <dataset.jsonl>
```

Some experiments require labelled datasets (see the script arguments). A real-PR benchmark expects at least 300 human-labelled PRs by default.

---

## API Overview

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check and active runtime/provider information |
| `POST` | `/v1/auth/login` | Obtain a tenant-bound bearer token |
| `POST` | `/v1/reviews` | Run a review synchronously |
| `POST` | `/v1/reviews?async=true` | Enqueue a review |
| `GET` | `/v1/tasks/{id}` | Task state, trace, collaboration, and report |
| `GET` | `/v1/tasks/{id}/report` | Markdown report |
| `POST` | `/v1/tasks/{id}/cancel` | Request cancellation |
| `POST` | `/v1/tasks/{id}/resume` | Resume from the latest checkpoint |
| `POST` | `/v1/tasks/{id}/fix` | Create verified repair commits and a draft PR |
| `GET/POST` | `/v1/tasks/{id}/feedback` | Read or record feedback for a completed task |
| `POST` | `/webhooks/github` | GitHub PR webhook |
| `POST` | `/v1/skills/reload` | Reload Skills from disk |
| `POST` | `/v1/evolution/auto` | Generate and evaluate a prompt candidate from failures |
| `POST` | `/v1/evolution/propose` | Evaluate a supplied prompt candidate |
| `GET/POST` | `/v1/evaluation/cases` | List or add versioned evaluation cases |
| `GET` | `/v1/evolution/status` | Model and gate readiness |
| `GET` | `/v1/evolution/runs` | Persisted evolution runs |
| `POST` | `/v1/skills/{name}/versions/{version}/activate` | Activate or roll back a prompt version |
| `POST` | `/v1/skill-evolution/auto` | Generate, replay, and gate a Skill candidate |
| `POST` | `/v1/skill-evolution/propose` | Evaluate a supplied `SKILL.md` artifact |
| `GET` | `/v1/skill-evolution/status` | Skill gate and active version |
| `GET` | `/v1/skill-evolution/runs` | Skill evolution runs |
| `GET` | `/v1/skill-evolution/{name}/versions` | Skill artifact version chain |
| `POST` | `/v1/skill-evolution/{name}/versions/{version}/activate` | Activate or roll back a Skill artifact |
| `GET` | `/metrics` | Prometheus text metrics |
| `GET` | `/api/dashboard` | Dashboard statistics and recent tasks |
| `GET` | `/api/tasks` | Task list |
| `GET` | `/api/skills` | Registered scanners and Agent Skills |
| `GET` | `/api/failures` | Failure cases (audit permission) |
| `GET` | `/api/audit` | Audit log (audit permission) |
| `GET` | `/api/alerts` | Tenant alerts |
| `GET` | `/api/queue/dead-letters` | Dead-letter tasks |
| `POST` | `/v1/queue/dead-letters/replay` | Replay a dead-letter task |
| `GET/POST` | `/api/deployments/llm-review` | Read or configure canary/shadow rollout |

Request limits (defaults): diff ≤ 1 MiB, runtime ≤ 8 steps and 120 seconds per task. See `.env.example` for all knobs.

---

## Security Model

- **Webhook authenticity** — HMAC-SHA256 signature verification with a replay window and delivery-id idempotency.
- **User authentication** — HMAC-signed bearer tokens with configurable TTL; PBKDF2-SHA256 password hashing with 310,000 rounds.
- **RBAC** — `admin`, `maintainer`, and `auditor` roles with `read`, `review`, `fix`, `manage`, and `audit` permissions.
- **Tenant isolation** — tasks, memories, repository allowlists, skills, deployments, alerts, and audit records are tenant-scoped.
- **Untrusted context** — diffs, repository content, tool output, feedback, and recalled memories are explicitly treated as untrusted evidence, not instructions.
- **Bounded execution** — tool schemas, path-escape checks, restricted subprocess environments, isolated verification copies, and per-agent budgets.
- **Skill sandboxing** — no symlinks, no path traversal, size/count limits, and a synthetic `read_skill_resource` tool restricted to selected packages.
- **Auditability** — immutable tenant audit records for reviews, fixes, feedback, evolution, rollouts, and queue operations.

Never expose the dashboard and API publicly with authentication disabled. Use a strong `EVOAGENT_AUTH_SECRET`, a separate webhook secret, and a reverse proxy that publishes only the routes you intend to expose.

---

## Deployment

### Docker Compose (PostgreSQL + Redis)

```bash
cp .env.example .env
# set your LLM keys and secrets in .env
docker compose up --build
```

The Compose stack starts:

- `evoagent` on port `8080`
- PostgreSQL 16 for durable state
- Redis 7 for Streams-based queuing

### Local production mode

Set `EVOAGENT_DATABASE_URL` to a PostgreSQL DSN and `EVOAGENT_REDIS_URL` to a Redis URL. Leave both empty to use SQLite plus the in-process acknowledged queue.

---

## Project Layout

```text
evoagent/
  api.py                 HTTP routing, auth, webhook endpoint, dashboard assets
  service.py             application wiring and use cases
  harness.py             stateful review workflow
  runtime.py             bounded node runtime, tool registry, schemas
  agentic_core.py        Lead / worker / Critic orchestration
  reviewer.py            deterministic and LLM reviewers
  repository_tools.py    evidence-producing repository tools
  gates.py               finding format/evidence/confidence/release gates
  context_manager.py     token-aware diff and observation compression
  memory.py              working, episodic, semantic, procedural memory
  skills.py              SKILL.md discovery, validation, resources
  evolution.py           prompt evaluation and activation
  skill_evolution.py     versioned Agent Skill replay gating
  evaluation_v2.py       product-backed agentic evaluation harness
  task_queue.py          in-memory ACK and Redis Streams queue with DLQ
  store.py               SQLite implementation
  postgres_store.py      PostgreSQL implementation
  github.py              GitHub REST client and App authentication
  fixer.py / patching.py deterministic and LLM patch providers
  verifier.py            compile/CST/test gates for repairs
  auth.py                login, tokens, roles
  telemetry.py           per-review model/tool accounting and traces
  observability.py       OpenTelemetry and alerts
  metrics.py             Prometheus metrics
skills/                  Agent Skill packages
web/                     dependency-free dashboard
tests/                   unit and integration tests
scripts/                 evaluation and evolution experiments
```

---

## Testing

```bash
python -m unittest discover -s tests -v
```

The test suite covers the runtime, diff parser, harness, service, GitHub signature handling, lead/worker collaboration, skills, memory, context management, evaluation, production features, and evolution gates. Some evolution/benchmark tests expect dataset files under `evaluation_data/`; provide those datasets before running the corresponding experiment scripts.

---

## License and Data

No license file is included in this repository. Add one before distributing. Do not commit API keys, PATs, webhook secrets, `.env` files, or real customer source code. Review data and model outputs can contain sensitive information; handle stored tasks and reports accordingly.
