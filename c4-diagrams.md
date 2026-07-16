# C4 Model Diagrams — Jolly

Architecture diagrams in Unicode box-drawing style. Same format as the system context
diagram in `README.md` Section 3.1 — renders in any terminal, no tools needed.

---

## System Context (C4 Level 1)

See `README.md` Section 3.1. The diagram there shows the flow:

```
Customer's Agent → Jolly CLI → Saleor Cloud, Saleor GraphQL, Git, pnpm, Vercel, Stripe
```

For a complete listing of every external system Jolly interacts with:

```
┌──────────────┬──────────────────────────────┬──────────────────────────────┐
│ System       │ Interface                    │ Purpose                      │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ Saleor Cloud │ cloud.saleor.io/platform/api │ Auth, org/project/environmen │
│ API          │ HTTPS REST                   │ CRUD                         │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ Saleor Auth  │ auth.saleor.io/realms/       │ OAuth2 device authorization  │
│ (Keycloak)   │ saleor-cloud                 │ grant + token refresh        │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ Saleor Store │ *.saleor.cloud/graphql/      │ GraphQL: stock seeding,      │
│ GraphQL      │ HTTPS GraphQL                │ appInstall, purchasability   │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ GitHub       │ github.com                   │ git clone saleor/storefront  │
│              │ git                          │                              │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ Vercel CLI   │ npx vercel                   │ spawn subprocess: deploy,    │
│              │                              │ login, whoami, env vars      │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ @saleor/     │ npx @saleor/configurator     │ spawn subprocess: deploy     │
│ configurator │                              │ starter recipe               │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ Stripe App   │ stripe-v2.saleor.app         │ installed via Saleor GraphQL │
│              │ (via appInstall)             │ appInstall; Dashboard config │
├──────────────┼──────────────────────────────┼──────────────────────────────┤
│ mcp.saleor   │ mcp.saleor.app               │ read-only store data access  │
│ .app (MCP)   │ (informational, not contacted│ after setup (MCP server)     │
│              │ by Jolly's own code)         │                              │
└──────────────┴──────────────────────────────┴──────────────────────────────┘
```

Jolly spawns official CLIs — it does not reimplement their APIs. Each spawned CLI
uses its own auth (Vercel session, git credentials, configurator token). Jolly's own
request code contacts only: cloud.saleor.io, auth.saleor.io, *.saleor.cloud, github.com.

---

## Container View (C4 Level 2)

Jolly ships four containers. The CLI is the primary; the others are supporting assets that ship with or alongside it.

```
┌──────────────┬──────────────┬────────────────────────────────┬───────────────────────┐
│ Container    │ Technology   │ What it does                   │ Delivered how         │
├──────────────┼──────────────┼────────────────────────────────┼───────────────────────┤
│ Jolly CLI    │ TypeScript   │ CLI entry, command handlers,   │ Published npm package │
│              │ esbuild      │ 7-stage pipeline, all output   │ @dk/jolly             │
│              │ dist/index.js│ formats (--json, --quiet, TTY) │ npx-installable       │
├──────────────┼──────────────┼────────────────────────────────┼───────────────────────┤
│ Message      │ JSON file    │ All human-facing copy resolved │ Bundled in npm package│
│ Catalog      │              │ by key at runtime.             │ assets/messages/      │
│              │              │ Placeholders filled per run.   │ cli.json              │
├──────────────┼──────────────┼────────────────────────────────┼───────────────────────┤
│ Jolly Skill  │ Markdown     │ Agent playbook for supervising │ Bundled in npm package│
│              │ SKILL.md     │ jolly start. Covers auth, risk │ assets/skills/jolly/  │
│              │              │ approval, gate handling.       │                       │
├──────────────┼──────────────┼────────────────────────────────┼───────────────────────┤
│ Jolly        │ HTML, CSS    │ jolly.cool — homepage with     │ Deployed to Vercel as │
│ Homepage     │ static files │ setup instructions, agent      │ separate project from │
│              │              │ handoff copy box.              │ assets/homepage/      │
└──────────────┴──────────────┴────────────────────────────────┴───────────────────────┘
```

The CLI container is decomposed further in the Component View below. The Skill and Homepage
are human-authored assets (Shipshape rules in AGENTS.md). The Message Catalog is the single
source for all human-facing interactive copy (feature 006 and 027 guard this).

---

## Component View (C4 Level 3)

CLI internals. The entry point dispatches to one of four modules; all modules share the utilities layer.

Dispatch flow: `CLI Entry (parser)` → `Auth/Cloud`, `Stage Orchestrator`, `Doctor`, or `Output` → all use `Shared Utilities`

```
┌──────────────────────┬─────────────────────────────────────────────────────────────────────┐
│ Module               │ Responsibilities                                                    │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ CLI Entry            │ @bomb.sh/args parser. Single parser seam for GLOBAL_BOOLEAN_FLAGS   │
│                      │ (--json, --quiet, --yes). Dispatches to handler by command name.    │
│                      │ Handles --help, --dry-run, unknown commands/flags with error codes. │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ Auth / Cloud Module  │ OAuth2 device authorization grant (Keycloak, client_id=jolly).      │
│                      │ Token storage + refresh in .env. Cloud API: orgs/projects/envs.     │
│                      │ Commands: login, logout, auth status, create store.                 │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ Stage Orchestrator   │ jolly start — runs 7 stages in order with concurrency where         │
│                      │ independent (store + storefront overlap). Idempotency gates,        │
│                      │ resumability detection from .env/local dirs/Vercel state.           │
│                      │ Emits riskContext per high-risk stage for agent approval.           │
│                      │ Spawns official CLIs for mechanical work (git, pnpm, configurator,  │
│                      │ vercel). Concurrency is observable in reported per-stage timing.    │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ Doctor Module        │ Diagnostic checks per group: skills, init, saleor, storefront,      │
│                      │ deployment, stripe. Runs vercel whoami, Cloud API organizations     │
│                      │ probe, GraphQL purchasability probe. Reports per-check status:      │
│                      │ pass/warning/fail/skipped/unknown. Never fabricates checks.         │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ Output Module        │ JSON envelope builder — { command, status, summary, data, checks,   │
│                      │ nextSteps, errors }. Human TTY renderer with colour/emoji/TTY       │
│                      │ detection. --quiet filter (only non-pass to stderr). In-place       │
│                      │ stderr progress for interactive (Bombshell).                        │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ Shared Utilities     │ HTTP retry with backoff (cloudFetchRetry). First-party host         │
│                      │ allowlist (NON_FIRST_PARTY_HOST gate on all outbound requests).     │
│                      │ .env reader/writer (mode 600, POSIX-safe quoting). Risk context     │
│                      │ builder. Skill installer (npx skills add). MCP config writer        │
│                      │ (.mcp.json merge without overwrite).                                │
└──────────────────────┴─────────────────────────────────────────────────────────────────────┘
```

---

## Deployment View

The CLI runs on the customer machine and reaches out to real services. It never reimplements a service API — it spawns the service's own CLI.

```
┌──────────────────────────────────────────────────┐
│                  npm Registry                    │
│  registry.npmjs.org — @dk/jolly package          │
│  npx resolve & download                          │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│              Customer Machine                    │
│  Node.js >= 20.12.0                              │
│  npx @dk/jolly — dist/index.js (esbuild)         │
│                                                  │
│  HTTPS outbound: cloud.saleor.io (REST)          │
│  HTTPS outbound: auth.saleor.io (OAuth2)         │
│  git clone: github.com (saleor/storefront)       │
│  spawn: npx vercel (deploy, login, whoami)       │
│  spawn: npx @saleor/configurator (deploy recipe) │
│  spawn: npx pnpm (storefront deps)               │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│                 Saleor Cloud                     │
│                                                  │
│  Cloud API — cloud.saleor.io/platform/api        │
│    REST: orgs, projects, environments (CRUD)     │
│    Auth: Authorization: Token <staff-token>      │
│                                                  │
│  Auth (Keycloak) — auth.saleor.io/realms/        │
│    saleor-cloud                                  │
│    OAuth2 device grant + token refresh           │
│                                                  │
│  Store GraphQL — *.saleor.cloud/graphql/         │
│    Stock seeding, appInstall, purchasability     │
│    Auth: Authorization: Bearer <access-token>    │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│                  Vercel Edge                     │
│                                                  │
│  Deployed storefront — Next.js 16 / Saleor Paper │
│  npx vercel deploy                               │
│  Environment vars: NEXT_PUBLIC_SALEOR_API_URL    │
│  NEXT_PUBLIC_DEFAULT_CHANNEL                     │
│  Disable Vercel Deployment Protection (SSO)      │
│  Deployed URL: *.vercel.app                      │
└──────────────────────────────────────────────────┘
```

---

## `jolly start` — Stage Pipeline

```
                    ┌──────────┐
                    │ 0. Boot  │
                    │ init     │
                    │ skills   │
                    │ .mcp.json│
                    └────┬─────┘
                         │
                         ▼
                    ┌──────────┐
                    │ 1. Auth  │
                    │ device   │
                    │ grant    │
                    └────┬─────┘
                         │
                         ▼
                 ┌───────────────┐
                 │ 2. Store Prov │
                 │ Cloud API     │──────┐
                 │ env creation  │      │
                 └───────┬───────┘      │
                         │              │
                         ▼              │
                 ┌───────────────┐      │
                 │ 3. Recipe     │      │   ┌──────────────┐
                 │ Configurator  │      ├───│ 5. Storefront│
                 │ deploy        │      │   │ git clone    │
                 │ confirm store │      │   │ pnpm install │
                 └───────┬───────┘      │   └──────┬───────┘
                         │              │          │
                         ▼              │          │
                 ┌───────────────┐      │          │
                 │ 4. Stock Seed │      │          │
                 │ Saleor GQL    │      │          │
                 │ concurrent    │      │          │
                 │ mutations     │      │          │
                 └───────┬───────┘      │          │
                         │              │          │
                         └──────────────┼──────────┘
                                        │
                                        ▼
                                ┌──────────────┐
                                │ 6. Deploy    │
                                │ Vercel CLI   │
                                │ spawn npx    │
                                └──────┬───────┘
                                       │
                                       ▼
                                ┌──────────────┐
                                │ 7. Stripe    │
                                │ appInstall   │
                                │ Saleor GQL   │
                                │ human gate   │
                                └──────────────┘

  Concurrency: stages 2 + 5 overlap (credential-independent storefront
  clone/install runs while the store provisions). Stage 4 uses concurrent
  Saleor GraphQL mutations across all recipe variants.

  Gates: stages 2, 3, 5, 6 require agent approval (feature 021 riskContext).
  --yes pre-approves all.
```
