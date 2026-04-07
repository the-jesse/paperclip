# CLAUDE.md — Paperclip Session Context

This file is loaded automatically at the start of every Claude Code session.
It captures a full research review of this repo so future sessions don't start cold.

---

## What Is This App

**Paperclip** is an open-source AI-agent company orchestration platform.
Tagline: *"If OpenClaw is an employee, Paperclip is the company."*

It's a Node.js/React control plane that orchestrates teams of AI agents to run autonomous businesses:
- Org charts, hierarchies, reporting lines
- Task/issue ticketing with atomic checkout semantics
- Heartbeat-driven agent execution with session persistence
- Monthly budgets per agent with hard-stop enforcement
- Board governance (approval gates, audit log, pause/terminate)
- Multi-company/multi-tenant in a single deployment
- Mobile-accessible dashboard

**GitHub**: https://github.com/paperclipai/paperclip  
**License**: MIT  
**Version**: 0.3.1 (as of review)  
**Discord**: https://discord.gg/m4HZY7xNG3

---

## Quick Start (When Installing on Mac/PC)

```bash
# One-command install
npx paperclipai onboard --yes

# Or manual
git clone https://github.com/paperclipai/paperclip.git
cd paperclip
pnpm install
pnpm dev
```

Requirements: Node.js 20+, pnpm 9.15+  
API server: `http://localhost:3100`  
Embedded PostgreSQL auto-created — no DB setup needed.

```bash
# Verify it's running
curl http://localhost:3100/api/health
curl http://localhost:3100/api/companies
```

---

## Stack

| Layer | Tech |
|---|---|
| Backend | Node.js + TypeScript 5.7, Express 5.1 |
| Frontend | React 19, Vite 6, React Router 7, TanStack Query 5 |
| Database | PostgreSQL 17 via Drizzle ORM 0.38; embedded PGlite for zero-config dev |
| Auth (humans) | BetterAuth 1.4 (sessions) |
| Auth (agents) | Hashed bearer API keys + custom JWT |
| Storage | Local disk or S3-compatible |
| Secrets | Local encrypted (AES, key at `~/.paperclip/instances/default/secrets-key.txt`) |
| Real-time | WebSocket (`ws`) for live UI events |
| Testing | Vitest 3 (unit), Playwright 1.58 (e2e) |
| Package manager | pnpm 9.15 workspaces (monorepo) |
| CSS | Tailwind CSS 4, Radix UI |

---

## Monorepo Structure

```
server/              Express REST API + orchestration services
ui/                  React board UI (served by API server)
packages/
  db/                Drizzle schema (64 tables), migrations, DB clients
  shared/            Shared TypeScript types, Zod validators, constants
  adapters/          Built-in agent adapters (one package per adapter)
  adapter-utils/     Shared adapter utilities
  plugins/           Plugin system SDK + examples
cli/                 paperclipai CLI (commander.js)
skills/              Runtime skills injected into agents at heartbeat time
tests/
  e2e/               Playwright end-to-end tests
  release-smoke/     Release validation tests
doc/                 Deep product + spec docs (read these before changing things)
scripts/             Dev runner, release, Docker entrypoint, migration helpers
```

---

## Key Docs to Read Before Changing Anything

In order per AGENTS.md:
1. `doc/GOAL.md`
2. `doc/PRODUCT.md`
3. `doc/SPEC-implementation.md` — the concrete V1 build contract
4. `doc/DEVELOPING.md`
5. `doc/DATABASE.md`

`doc/SPEC.md` = long-horizon product vision  
`doc/DEPLOYMENT-MODES.md` = auth/deployment mode reference

---

## Dev Commands

```bash
pnpm dev              # Full dev (API + UI, watch mode) — port 3100
pnpm dev:once         # Full dev without watch
pnpm dev:server       # Server only
pnpm build            # Build everything
pnpm typecheck        # TypeScript check across all packages
pnpm test:run         # Run tests once
pnpm db:generate      # Generate DB migration after schema change
pnpm db:migrate       # Apply pending migrations
pnpm db:backup        # Backup embedded Postgres
pnpm paperclipai      # CLI tool
```

Before handing off any change:
```bash
pnpm -r typecheck && pnpm test:run && pnpm build
```

---

## Built-in Agent Adapters

All live in `packages/adapters/<name>/`:

| Adapter ID | Package | What It Runs | Auth |
|---|---|---|---|
| `claude_local` | `claude-local` | Claude Code CLI (Anthropic) | `ANTHROPIC_API_KEY` or subscription login |
| `codex_local` | `codex-local` | OpenAI Codex CLI | OpenAI API key |
| `cursor_local` | `cursor-local` | Cursor | Cursor auth |
| `gemini_local` | `gemini-local` | Google Gemini CLI | `GEMINI_API_KEY` or `GOOGLE_API_KEY` |
| `openclaw_gateway` | `openclaw-gateway` | OpenClaw managed gateway | Gateway URL + API key |
| `opencode_local` | `opencode-local` | SST OpenCode (multi-provider) | Depends on backend model |
| `pi_local` | `pi-local` | Pi by Inflection AI | Pi auth |
| `process` | (built-in) | Any shell command | n/a |
| `http` | (built-in) | Any HTTP endpoint | Custom headers |

Each adapter package has:
- `src/server/execute.ts` — heartbeat invocation logic
- `src/server/parse.ts` — output parsing
- `src/server/quota.ts` — quota/billing detection
- `src/server/skills.ts` — skill injection
- `src/ui/` — config form + output display for the board UI
- `src/cli/` — CLI test/quota probe

### External Adapters (Plugin System)

External adapters can be loaded via `~/.paperclip/adapter-plugins.json`.
No code changes to core needed — pure dynamic loading.

---

## AI Provider Coverage (Owner's Stack)

| Provider/Tool | How to Connect |
|---|---|
| **Claude Code** | `claude_local` adapter — native |
| **OpenClaw** | `openclaw_gateway` adapter — native |
| **Codex** | `codex_local` adapter — native |
| **Google / Gemini** | `gemini_local` adapter — native, `GEMINI_API_KEY` or `GOOGLE_API_KEY` |
| **Cursor** | `cursor_local` adapter — native |
| **Grok (xAI)** | `opencode_local` → OpenCode supports xAI as provider — configure OpenCode with xAI key |
| **Local AI / phone / edge device** | `http` adapter — any device exposing an HTTP endpoint |
| **Pi (Inflection)** | `pi_local` adapter — native |
| **Any other frontier model** | `opencode_local` (OpenCode is multi-provider) or `http` adapter |

**Key insight on Grok**: Not a native adapter, but `opencode_local` routes through SST's OpenCode CLI which itself supports xAI. Configure OpenCode's provider settings to use `OPENAI_API_KEY` pointing at xAI's base URL, or use OpenCode's native xAI provider config.

---

## Deployment Modes

| Mode | Auth | Use Case |
|---|---|---|
| `local_trusted` | No login — localhost only | Local dev, personal use |
| `authenticated` + `private` | Login required | Tailscale/VPN access (phone access) |
| `authenticated` + `public` | Login required + HTTPS | Cloud / internet-facing |

**For phone access**: Set up Tailscale, then run `authenticated/private` mode.

Config lives at `~/.paperclip/instances/default/config.json`.

---

## Database

**64 tables** organized into categories:

| Category | Key Tables |
|---|---|
| Core | `companies`, `agents`, `projects`, `goals`, `issues`, `routines` |
| Work tracking | `issue_comments`, `issue_documents`, `issue_work_products`, `issue_approvals` |
| Financial | `cost_events`, `budget_policies`, `budget_incidents` |
| Execution | `heartbeat_runs`, `heartbeat_run_events`, `execution_workspaces` |
| Plugins | `plugins`, `plugin_config`, `plugin_state`, `plugin_jobs`, `plugin_logs` |
| Auth | `auth_users`, `auth_sessions`, `agent_api_keys` |
| Audit | `activity_log`, `approvals`, `approval_comments` |

**DB change workflow**:
1. Edit `packages/db/src/schema/*.ts`
2. Export from `packages/db/src/schema/index.ts`
3. `pnpm db:generate`
4. `pnpm -r typecheck`

---

## Key Services (Largest / Most Important)

| File | Lines | Purpose |
|---|---|---|
| `server/src/services/heartbeat.ts` | 4,234 | Agent execution engine — checkout, budget, session, cost |
| `server/src/services/company-portability.ts` | 4,319 | Export/import entire orgs (templates, migration, backup) |
| `server/src/services/workspace-runtime.ts` | 2,127 | Docker/native execution environment setup |
| `server/src/services/issues.ts` | 2,098 | Task state machine — transitions, checkout, documents |
| `server/src/services/company-skills.ts` | 2,371 | Skill manifest loading and injection |
| `server/src/services/plugin-loader.ts` | 1,954 | Dynamic plugin loading from npm/local/GitHub |
| `server/src/services/routines.ts` | 1,481 | Scheduled task execution (cron, concurrency, catch-up) |
| `server/src/services/budgets.ts` | 958 | Cost enforcement, quota windows, incident tracking |

---

## Auth Summary

- **Humans (Board)**: BetterAuth sessions, httpOnly cookies
- **Agents**: Hashed bearer API keys stored in `agent_api_keys`; JWT for locally-spawned agents
- **Company scoping**: Every route enforces `companyId` from actor context
- **Board guards**: `board-mutation-guard.ts` middleware — hire, fire, strategic decisions require board actor
- **Activity log**: Immutable, every mutation logged with actor attribution

---

## Environment Variables Reference

```bash
DATABASE_URL=postgres://...           # Unset = embedded Postgres
PORT=3100
HOST=127.0.0.1
SERVE_UI=true
PAPERCLIP_DEPLOYMENT_MODE=local_trusted   # or authenticated
PAPERCLIP_DEPLOYMENT_EXPOSURE=private     # or public
BETTER_AUTH_SECRET=<signing-secret>
PAPERCLIP_AGENT_JWT_SECRET=<secret>
PAPERCLIP_TELEMETRY_DISABLED=1        # Opt out of telemetry
DO_NOT_TRACK=1                        # Standard telemetry opt-out
PAPERCLIP_STORAGE_PROVIDER=local_disk # or s3
PAPERCLIP_SECRETS_PROVIDER=local_encrypted
```

---

## Plugin System

- Plugins run in isolated Node.js child processes
- Load from: npm package, local `file:` path, or GitHub
- Register via: `~/.paperclip/adapter-plugins.json`
- Capabilities: custom entities, tool registration, job scheduling, webhooks, IFrame UI
- Built-in examples: `para-memory-files` (persistent agent memory), `paperclip-create-plugin`
- 14 plugin service files, 10K+ LOC in the plugin subsystem

---

## CI/CD

GitHub Actions workflows:
- **pr.yml** — typecheck + test + build + e2e on every PR
- **release.yml** — tag `v*` triggers npm publish + Docker image + GitHub release
- **docker.yml** — Docker image build
- **refresh-lockfile.yml** — weekly automated `pnpm-lock.yaml` refresh

---

## Core Invariants (Don't Break These)

Per `AGENTS.md`:
1. Every entity is company-scoped — enforce in all routes/services
2. Single-assignee task model — no shared ownership
3. Atomic issue checkout semantics
4. Approval gates for governed actions (hire, CEO strategy)
5. Budget hard-stop auto-pause behavior
6. Activity logging for ALL mutating actions
7. Contracts synced across `db` / `shared` / `server` / `ui` when schema changes

---

## PR Requirements

Every PR must use `.github/PULL_REQUEST_TEMPLATE.md` with all sections filled:
- Thinking Path
- What Changed
- Verification
- Risks
- Model Used (include provider, model ID — e.g. `claude-sonnet-4-6`)
- Checklist

---

## Fork Notes (this-jesse/paperclip)

This is on branch `claude/review-new-repo-0OYJb`.
Upstream is `paperclipai/paperclip`.

The `AGENTS.md` also references a fork `HenkDz/paperclip` with:
- External Hermes adapter (not built-in, loaded via plugin manager)
- Port 3101+ (auto-detects if 3100 taken)
- QoL UI patches (stderr_group accordion, tool_group accordion, dashboard excerpt)

---

## Research Notes

Full deep-dive completed 2026-04-06 via 123-tool automated exploration.
Files reviewed: README, AGENTS.md, SPEC.md, SPEC-implementation.md, DEPLOYMENT-MODES.md,
DATABASE.md, all adapter execute.ts files, package.json, vitest.config.ts,
Dockerfile, .github/workflows, server/src/config.ts, db schema overview.
