# Jolly arc42 Architecture Documentation

Derived from spec-driven feature files at `/home/exedev/jolly/features/`.

- [1. Introduction and Goals](#1-introduction-and-goals)
- [2. Architecture Constraints](#2-architecture-constraints)
- [3. Context and Scope](#3-context-and-scope)
- [4. Solution Strategy](#4-solution-strategy)
- [5. Building Block View](#5-building-block-view)
- [6. Runtime View](#6-runtime-view)
- [7. Deployment View](#7-deployment-view)
- [8. Cross-cutting Concepts](#8-cross-cutting-concepts)
- [9. Architecture Decisions](#9-architecture-decisions)
- [10. Quality Requirements](#10-quality-requirements)
- [11. Risks and Technical Debt](#11-risks-and-technical-debt)
- [13. C4 Model Diagrams](#13-c4-model-diagrams)
- [12. Glossary](#12-glossary)

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

Jolly is an npm CLI package (`@dk/jolly`) that helps AI coding agents set up a complete Saleor Cloud e-commerce storefront with Vercel deployment and Stripe payment processing. The system is **agent-first**: its primary user is an AI agent, not a human directly. The human is the agent's customer who provides account credentials and completes irreducible interactive gates.

The tool bootstraps a project with agent skills, provisions a Saleor Cloud store, deploys a Paper storefront to Vercel, installs the Stripe payment app, and guides the remaining human steps.

### 1.2 Quality Goals

| Goal | Description |
|---|---|
| Agent-first | CLI output is structured JSON for programmatic consumption; human-friendly output is secondary |
| No fabricated success | Every claim in output is backed by a real operation; never simulate or guess |
| Honest gating | Stop at gates the agent or human cannot pass; report exact state |
| Idempotent | Re-runnable without duplicates; detect existing work and resume |
| Harmless by design | Tests and eval create namespaced resources with best-effort teardown |
| Live-by-design verification | BDD suite uses real services; no mocks, fakes, or dummy credentials |
| Orchestrate, don't reimplement | Jolly spawns official CLIs (`git`, `pnpm`, `@saleor/configurator`, `vercel`) rather than wrapping their APIs |

### 1.3 Stakeholders

- **Customer's AI agent**: Primary user; drives the CLI, parses structured output, makes approval decisions
- **Human customer**: Owns accounts (Saleor Cloud, Vercel, Stripe); completes interactive sign-in gates
- **Jolly maintainer**: Authors specs, verification, and implementation via Shipshape spec-driven workflow

## 2. Architecture Constraints

### 2.1 Technical Constraints

| Constraint | Details |
|---|---|
| Runtime | Node.js >= 20.12.0; dev runtime >= 23 |
| Package manager | npm (for Jolly itself); pnpm (for the Paper storefront) |
| Distribution | Published as `@dk/jolly` on npm; run via `npx @dk/jolly` |
| Language | TypeScript, ES modules, compiled to JavaScript for publish |
| Build | esbuild bundle to `dist/index.js` |

### 2.2 Organizational Constraints

- Jolly is a third-party tool by Dmytri Kleiner — not an official Saleor, Vercel, or Stripe product
- Spec-driven development via Shipshape (Captain → Quartermaster → Crew → Boatswain)
- Verification uses **real services only** — fakes and mocks are forbidden

### 2.3 Domain Constraints

- Saleor Cloud only (never self-hosted Saleor)
- Stripe test mode only (v1)
- Vercel is the only deployment target (v1)
- Storefront baseline is Saleor Paper (`saleor/storefront`)
- Saleor Configurator (`@saleor/configurator`) for store configuration as code

## 3. Context and Scope

### 3.1 System Context (External interfaces)

```
┌─────────────────────────┐
│    Customer's Agent     │── CLI (npx, --json envelope)
└─────────┬───────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────┐
│                        Jolly CLI                         │
│  @dk/jolly  (npx-installable, TypeScript → esbuild)      │
│                                                          │
│  Subcommands: login, logout, auth status, init, start,   │
│  doctor, upgrade, skills, create store, completion       │
└────┬──────┬──────┬──────┬──────┬──────┬──────────────────┘
     │      │      │      │      │      │
     │      │      │      │      │      │
     ▼      ▼      ▼      ▼      ▼      ▼
  Saleor  Saleor  Git     pnpm   Vercel  Stripe (via
  Cloud   GraphQL                CLI    Saleor GraphQL
  API     (store)                       appInstall)
```

### 3.2 External Systems

| System | Interface | Purpose |
|---|---|---|
| Saleor Cloud API | HTTPS REST (`cloud.saleor.io/platform/api`) | Auth, org/project/environment CRUD |
| Saleor Auth (Keycloak) | HTTPS OAuth2 (`auth.saleor.io`) | Device authorization grant |
| Saleor Store GraphQL | HTTPS GraphQL (`*.saleor.cloud/graphql/`) | Store queries, stock seeding, app install |
| GitHub | HTTPS git | Clone `saleor/storefront` (Paper), install skills |
| Vercel CLI | Spawned `npx vercel` | Deploy storefront, manage env vars |
| `@saleor/configurator` | Spawned `npx @saleor/configurator` | Deploy starter recipe as store configuration |
| `npx skills` | Spawned `npx skills add` | Install agent skills |

### 3.3 Business Context

Jolly bridges the gap between a blank Saleor Cloud account and a working, deployed e-commerce storefront. It reduces the setup from a multi-hour manual process spanning multiple web dashboards to a guided agent-driven workflow with minimal human intervention (account creation, Vercel sign-in, Stripe key paste).

## 4. Solution Strategy

### 4.1 Agent-first CLI design

The CLI speaks JSON to agents. Every command emits a consistent envelope: `{ command, status, summary, data, checks, nextSteps, errors }`. The agent branches on `status` and stable error codes, consuming `nextSteps` and `remediation` to self-correct without falling back to `--help`.

### 4.2 Orchestration, not reimplementation

Jolly spawns the official CLIs for mechanical work:
- `git clone` for the Paper storefront
- `pnpm install` for dependencies
- `@saleor/configurator deploy` for store configuration
- `npx vercel` for deployment

Jolly's own code handles only plumbing: auth flows, Cloud API calls, `.env` management, skill installation, and diagnostics.

### 4.3 Verification with real services

Every test tier (`@logic`, `@sandbox`, `@eval`) exercises real behavior against the real integrated test environment. The only admissible doubles are `@exceptional-double` scenarios for conditions the real environment cannot produce (org at limit, unreachable service).

### 4.4 Spec-driven development

All behavior is specified in Gherkin `.feature` files. Implementation (`src/`) is disposable — built by Shipshape Crew to satisfy failing scenarios. Human-authored durable material lives in `assets/`.

## 5. Building Block View

### 5.1 CLI Command Surface (Level 1)

```
@dk/jolly CLI
├── login          — Saleor Cloud device authorization grant
├── logout         — Remove Jolly-managed auth from .env
├── auth status    — Report authentication state
├── init           — Bootstrap agent skills and guidance
├── start          — End-to-end setup orchestration
│   ├── store      — Provision or connect Saleor Cloud store
│   ├── storefront — Clone and install Paper storefront
│   ├── recipe     — Deploy starter recipe via @saleor/configurator
│   ├── stock      — Seed product stock via Saleor GraphQL
│   ├── deploy     — Deploy storefront to Vercel
│   └── stripe     — Install Stripe app via Saleor GraphQL
├── doctor         — Diagnostics (groups: skills, init, saleor, storefront, deployment, stripe)
├── upgrade        — Re-verify skills, report Paper baseline
├── skills         — Skill management (install, update)
├── create store   — Provision or connect a Saleor Cloud store
└── completion     — Shell completion script
```

### 5.2 White-Box: Core Modules

```
src/index.ts           — CLI entry, Bombshell parser, command dispatch
├── auth/              — OAuth2 device grant, token management, .env read/write
├── cloud/             — Saleor Cloud API client (orgs, projects, environments)
├── storefront/        — Paper clone, pnpm install, Vercel deploy orchestration
├── recipe/            — Configurator spawn, stock seeding
├── stripe/            — Stripe app install via Saleor GraphQL
├── skills/            — Skill installation via npx skills add
├── doctor/            — Diagnostic checks (per-group runners)
├── output/            — Envelope builder, human renderer, TTY detection
├── risk/              — Structured risk context builder
└── util/              — Shared: HTTP retry, host allowlist, env helpers
```

### 5.3 Verification Layer (Test Suite)

```
features/
├── *.feature                     — Gherkin specs (30 feature files)
├── step_definitions/*.steps.ts   — Cucumber step implementations
├── support/
│   ├── hooks.ts                  — BeforeAll/AfterAll (reclaim, teardown)
│   ├── world.ts                  — Shared Cucumber world
│   ├── provision.ts              — Saleor Cloud store provisioning
│   ├── sandbox.ts                — Sandbox harness (env creation, teardown)
│   ├── pty.ts                    — PTY loopback for interactive tests
│   └── plank-conformance.ts      — Structural conformance checker
tests/
├── logic.test.ts                 — @logic tier unit tests
└── sandbox.test.ts               — @sandbox tier integration tests
```

## 6. Runtime View

### 6.1 `jolly start` — End-to-End Setup Flow

```
Step 0: Bootstrap
  jolly init → install skills (npx skills add)
             → write .mcp.json
             → scaffold .env, AGENTS.md

Step 1: Auth
  If no token → Saleor device authorization grant
              → print verification URL → wait for human approval
              → store JOLLY_SALEOR_ACCESS_TOKEN, JOLLY_SALEOR_REFRESH_TOKEN

Step 2: Store Provisioning  [high-risk → agent approval]
  If no SALEOR_URL → POST /platform/api/organizations/{org}/environments/
                   → poll task-status until SUCCEEDED
                   → write SALEOR_URL, SALEOR_TOKEN, NEXT_PUBLIC_SALEOR_API_URL to .env

Step 3: Recipe Deploy  [high-risk → agent approval]
  Spawn: npx @saleor/configurator@latest deploy
         --url $SALEOR_URL --token $SALEOR_TOKEN --config recipe.yml
  Verify: read store back, confirm catalog entities exist

Step 4: Stock Seed
  For each recipe variant → Saleor GraphQL productVariantStocksCreate
  Concurrent per-variant mutations

Step 5: Storefront  [high-risk → agent approval]
  Spawn: git clone saleor/storefront
  Spawn: pnpm install
  Approve native build scripts (sharp, esbuild)

Step 6: Deploy  [high-risk → agent approval]
  Spawn: npx vercel@latest login (if no session)
  Spawn: npx vercel@latest --prod
  Set env vars on Vercel project
  Disable Vercel Deployment Protection

Step 7: Stripe Install
  Saleor GraphQL appInstall(manifestUrl: "https://stripe-v2.saleor.app/api/manifest")
  Announce keys + channel-mapping human gate
```

### 6.2 Concurrency

The storefront preparation (git clone + pnpm install) runs **concurrently** with Saleor Cloud store provisioning and recipe deploy, since they have no dependency on each other. The deploy stage waits for storefront readiness. Stock seeding uses **concurrent** per-variant Saleor GraphQL mutations.

### 6.3 Idempotency and Resumability

- `jolly start` detects already-completed stages from observable artifacts (`.env` values, storefront directory, Vercel deployment)
- Skips satisfied stages; resumes from the first outstanding one
- `--dry-run` previews the exact plan without side effects
- `--yes` pre-approves all high-risk stages

## 7. Deployment View

### 7.1 Published Package

```
@dk/jolly (npm)
├── bin/jolly              — Node.js launcher
├── dist/index.js          — esbuild bundle (published)
├── assets/messages/cli.json  — Message catalog
├── assets/skills/         — Bundled Jolly skill
└── README.md
```

### 7.2 Runtime Environment

- **Developer**: Node.js >= 23, npm, runs raw TypeScript via native type stripping
- **Consumer**: `npx @dk/jolly` — Node.js >= 20.12.0, compiled JS
- **CI/Test runner**: Node.js >= 23, `.env` with `JOLLY_SALEOR_CLOUD_TOKEN`, Vercel CLI session
- **Homepage**: `assets/homepage/` deployed to Vercel as `jolly.cool`

### 7.3 Saleor Cloud Environment Topology

```
Saleor Cloud Organization
├── Project (dev plan)
│   └── Environment (blank template)
│       ├── GraphQL API (*.saleor.cloud/graphql/)
│       ├── Dashboard (*.saleor.cloud/dashboard/)
│       ├── Stripe app (installed via appInstall)
│       └── Starter recipe (via @saleor/configurator)
└── Multiple jolly-cannon-fodder-* test environments
```

## 8. Cross-cutting Concepts

### 8.1 Output Contract

Every command emits one consistent JSON envelope on `--json`:

```json
{
  "command": "jolly doctor",
  "status": "success",
  "summary": "All checks passed",
  "data": { /* command-specific */ },
  "checks": [
    { "id": "saleor-cloud-token", "status": "pass", "message": "Authenticated as acme-co" }
  ],
  "nextSteps": [
    { "description": "Run jolly start to continue", "command": "jolly start" }
  ],
  "errors": []
}
```

Fields: camelCase. `status`: `success` | `warning` | `error`. Checks: `pass` | `warning` | `fail` | `skipped` | `unknown`.

Default output (no `--json`) is human-friendly with colour, restrained emoji, in-place progress on stderr. `--quiet` prints only non-pass items to stderr.

### 8.2 Security and Secrets

- Secrets are **never printed** — referenced by name only
- Secrets travel only to their own service (Saleor token → cloud.saleor.io, Vercel token → Vercel CLI only)
- `.env` files are written with mode 600 (owner-only)
- No `--token`, `--token-file`, or `--token-stdin` flags — secrets come from env or device grant
- First-party host allowlist enforced pre-flight (`NON_FIRST_PARTY_HOST` error)

### 8.3 Auth Architecture

Two auth paths:
1. **Staff token** (`JOLLY_SALEOR_CLOUD_TOKEN`): long-lived, for CI/tests, sent as `Authorization: Token`
2. **Device grant** (OAuth2): interactive, produces `JOLLY_SALEOR_ACCESS_TOKEN` (Bearer JWT, short-lived) + `JOLLY_SALEOR_REFRESH_TOKEN` (longer-lived)

Token priority: access token > cloud token. Refresh happens automatically on expiry.

### 8.4 Risk Context

High-risk actions carry structured risk context in the envelope:

```json
{
  "action": "create store",
  "target": "Saleor Cloud environment in acme-co",
  "riskLevel": "high",
  "categories": ["destructive operations", "billing"],
  "reversible": false,
  "sideEffects": ["Creates a Saleor Cloud environment"],
  "dryRunAvailable": true
}
```

Jolly never self-approves; it surfaces the context and pauses for agent approval (`--yes` pre-approves).

### 8.5 Verification Philosophy

- **Real services always** — no mocks, fakes, dummy credentials, or `.invalid` endpoints
- **Tiered**: `@logic` (fast, parallel) → `@sandbox` (real services, heavy/light split) → `@eval` (live baseline agent)
- **Harmless by design**: test resources namespaced (`jolly-cannon-fodder-*`), best-effort teardown, never touch what we didn't create
- **Self-enforcing**: `@property` conformance scenarios detect forbidden doubles, missing recovery, structural violations

### 8.6 Message Catalog

All human-facing interactive copy resolves through `assets/messages/cli.json` by key. No inline string literals on the interactive path. Placeholders (`{org}`, `{url}`) are substituted at render time.

### 8.7 Bombshell as Single CLI Plumbing

- `@bomb.sh/args` — single argument parser
- `@clack/prompts` — single interactive prompt mechanism
- `@bomb.sh/tab` — single shell completion generator

No redundant hand-rolled implementations.

## 9. Architecture Decisions

### 9.1 Spawn CLIs, Don't Wrap APIs

- **Decision**: Jolly spawns `git`, `pnpm`, `@saleor/configurator`, and `npx vercel` as child processes rather than calling their APIs directly
- **Rationale**: Avoids reimplementing each tool's logic, auth, and error handling. Each CLI owns its auth (Vercel CLI session, git credentials). Jolly stays thin and never holds a Vercel token.
- **Consequence**: Installation of these tools is a precondition; Jolly runs them via `npx` so no global install needed for Node-based ones.

### 9.2 Device Authorization Grant for Auth

- **Decision**: Saleor Cloud sign-in uses OAuth2 device authorization grant (public client `jolly`, no secret)
- **Rationale**: Works for both interactive (terminal with URL) and non-interactive (agent relays URL to human) contexts. No token-paste UX.
- **Consequence**: Short-lived access tokens (~5 min) require automatic refresh; persisted refresh token enables re-auth without re-prompt.

### 9.3 Agent-first Output Contract

- **Decision**: All commands emit one consistent JSON envelope under `--json`
- **Rationale**: Enables generic agent parsing: branch on `status`, consume `nextSteps`/`remediation`, chain commands
- **Consequence**: Human output is strictly secondary; `--json` is the canonical interface

### 9.4 No Fabricated Success

- **Decision**: Never claim work that wasn't performed and confirmed
- **Rationale**: Trust. An agent that trusts Jolly's output will act on it; fabricated success causes downstream failures far from the CLI.
- **Enforcement**: Feature 020 scenarios, property checks, eval affordance measurement

### 9.5 Blank Environment Template

- **Decision**: Provisioned Saleor environments use `database_population: null` (blank template)
- **Rationale**: The starter recipe is a complete declarative config; sample data would be deleted on first deploy
- **Consequence**: No demo data — the recipe is the first and only catalog config

### 9.6 Live-by-Design Verification

- **Decision**: Test suite uses real services; fakes and mocks are forbidden
- **Rationale**: A green suite with fakes proves only that the fakes are consistent, not that the real system works. Real-service testing catches integration failures, auth changes, API drift.
- **Enforcement**: Feature 026's `@property` scan for forbidden doubles; `@exceptional-double` tag for genuine edge cases

## 10. Quality Requirements

### 10.1 Test Coverage by Tier

| Tier | Tag | Parallelism | Scope | Count |
|---|---|---|---|---|
| Logic | `@logic` | Parallel | Fast assertions, output shape, redaction, host enumeration | 187 scenarios |
| Sandbox light | `@sandbox` (not `@heavy`) | Parallel | Real-service queries and checks | ~15 scenarios |
| Sandbox heavy | `@sandbox @heavy` | Serial | Full end-to-end runs (`jolly start`, deploys) | ~41 scenarios |
| Eval | `@eval` | Serial | Live baseline agent affordance measurement | 3 scenarios |

### 10.2 Key Quality Properties

- **Idempotency**: Re-running any command is safe and detects existing work
- **Resumability**: `jolly start` resumes from the first incomplete stage
- **Honesty**: Every output claim is backed by a real operation; never simulate
- **Resilience**: Transient failures (429, 5xx, connection drops) are retried before reporting blocked
- **Security**: Secrets never printed, never travel to non-first-party hosts
- **Determinism**: Same input + same environment → same output

## 11. Risks and Technical Debt

### 11.1 Known Risks

| Risk | Mitigation |
|---|---|
| Saleor Cloud API changes | Live-by-design verification catches drift immediately |
| Vercel CLI API changes | Spawned CLI, not raw API; CLI version pinned via `@latest` |
| Org environment limit (2 envs) | Pre-run reclamation of `jolly-cannon-fodder` environments; ENVIRONMENT_LIMIT_REACHED error |
| Test VM resource saturation | Heavy/serial profile; `--max-old-space-size=4096` |
| Shared store transient death (free tier) | Self-heal: delete dead marker store, provision fresh one; one sanctioned retry |

### 11.2 Technical Debt

- `src/index.ts` at ~5,900 lines / ~98 functions
- 16 orphaned step definitions (unused verification support)
- `happy-dom` unused devDependency
- `report.html` / `report.json` tracked in git (should be `.gitignore`d)
- Configurator spurious exit 5 ("partial") on bootstrap requires report-based completion detection
- No formal envelope schema versioning
- No `@planks` annotations in production code (pending Shipwright harbour pass)

## 13. C4 Model Diagrams

Architecture visualisations at all four C4 levels — context, container, component, deployment — plus a sequence diagram for `jolly start` and the test infrastructure layout.

See [C4 Model Diagrams](c4-diagrams.md).

---

## 12. Glossary

| Term | Definition |
|---|---|
| **Saleor Cloud** | Managed e-commerce platform providing GraphQL API and Dashboard |
| **Paper** | Saleor's official storefront template (`saleor/storefront`), Next.js-based |
| **Configurator** | `@saleor/configurator` — Commerce as Code CLI for Saleor store config |
| **Starter recipe** | Jolly's bundled `recipe.yml` — a complete `@saleor/configurator` config with pirate-themed demo catalog |
| **Device authorization grant** | OAuth2 flow where the user authorizes via a verification URL on a separate device |
| **Risk context** | Structured metadata describing an action's impact, enabling agent-decided approval |
| **Envelope** | The standard JSON output shape: `{ command, status, summary, data, checks, nextSteps, errors }` |
| **`jolly-cannon-fodder`** | Namespace prefix for all test-created Saleor Cloud environments and Vercel projects, enabling safe reclamation |
| **Shipshape** | Spec-driven agent workflow (Captain → QM → Crew → Boatswain) used to develop Jolly |
| **Bombshell** | CLI toolkit (`@bomb.sh/args`, `@clack/prompts`, `@bomb.sh/tab`) providing single-source argument parsing, prompts, and completions |
| **Scantling** | A mechanical schema or shape specification, referenced from a `@contract` scenario, defining a non-behavioural constraint |
| **Perturbation** | A temporary `throw new Error(...)` planted in production code to draw attention to a seam needing review; removed when the concern is fixed |
