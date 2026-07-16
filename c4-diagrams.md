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

Jolly ships four containers. The CLI is the primary; the others are supporting assets.

```
┌──────────┬──────────────┬──────────────────────┬──────────────────────┐
│ Container │ Technology   │ What it does         │ Delivered how        │
├──────────┼──────────────┼──────────────────────┼──────────────────────┤
│ Jolly    │ TypeScript   │ CLI entry, command    │ Published npm        │
│ CLI      │ esbuild      │ handlers, 7-stage    │ package @dk/jolly    │
│          │ dist/index.js│ pipeline             │ npx-installable       │
├──────────┼──────────────┼──────────────────────┼──────────────────────┤
│ Message  │ JSON file    │ All human-facing copy│ Bundled in npm        │
│ Catalog  │              │ resolved by key      │ package               │
│          │              │ at runtime           │ assets/messages/     │
├──────────┼──────────────┼──────────────────────┼──────────────────────┤
│ Jolly    │ Markdown     │ Agent playbook for   │ Bundled in npm        │
│ Skill    │ SKILL.md     │ supervising jolly    │ package               │
│          │              │ start                │ assets/skills/       │
├──────────┼──────────────┼──────────────────────┼──────────────────────┤
│ Jolly    │ HTML/CSS     │ jolly.cool — setup   │ Deployed to Vercel   │
│ Homepage │              │ instructions, agent  │ as separate project  │
│          │              │ handoff page         │ from assets/homepage/│
└──────────┴──────────────┴──────────────────────┴──────────────────────┘
```

The CLI container is decomposed further in the Component View below.
The Skill and Homepage are human-authored assets (Shipshape rules in AGENTS.md).

---

## Component View (C4 Level 3)

CLI internals — each component is a logical module in `src/`. The flow is:

```
CLI Entry (parser + dispatch)
    │
    ├── Auth / Cloud Module     (login, logout, auth status, create store)
    ├── Stage Orchestrator      (jolly start — 7-stage pipeline)
    ├── Doctor Module           (diagnostics per group)
    └── Output Module           (envelope builder, TTY renderer, --quiet)
         │
         ▼
    Shared Utilities (across all modules)
    └── HTTP retry, host allowlist, .env I/O, risk context builder,
        skill installer, MCP config writer
```

Detailed module responsibilities:

```
┌──────────────────┬───────────────────────────────────────────────────────┐
│ Module           │ Responsibilities                                      │
├──────────────────┼───────────────────────────────────────────────────────┤
│ CLI Entry        │ @bomb.sh/args parser — single parser seam for all     │
│                  │ flags (--json, --quiet, --yes). Dispatches to command  │
│                  │ handler. Handles --help, --dry-run, unknown commands.  │
├──────────────────┼───────────────────────────────────────────────────────┤
│ Auth / Cloud     │ OAuth2 device authorization grant against Saleor      │
│                  │ Keycloak. Token storage + refresh. .env read/write.   │
│                  │ Cloud API: org list, project create/reuse,            │
│                  │ environment create with async task polling.           │
│                  │ Commands: login, logout, auth status, create store.   │
├──────────────────┼───────────────────────────────────────────────────────┤
│ Stage            │ jolly start — runs 7 stages in order with concurrency │
│ Orchestrator     │ where stages are independent (store+storefront).      │
│                  │ Idempotency gates skip completed stages. Resumability │
│                  │ detection from .env, local dirs, Vercel state.        │
│                  │ Emits riskContext per high-risk stage for agent       │
│                  │ approval. Spawns official CLIs for mechanical work.   │
├──────────────────┼───────────────────────────────────────────────────────┤
│ Doctor Module    │ Diagnostic checks per group: skills, init, saleor,    │
│                  │ storefront, deployment, stripe. Runs vercel whoami,   │
│                  │ Cloud API organizations probe, GraphQL purchasability │
│                  │ probe. Reports per-check status (pass/warning/fail/   │
│                  │ skipped/unknown).                                     │
├──────────────────┼───────────────────────────────────────────────────────┤
│ Output Module    │ JSON envelope builder (command, status, summary, data,│
│                  │ checks, nextSteps, errors). Human TTY renderer with   │
│                  │ colour/emoji/TTY detection. --quiet filter.           │
│                  │ In-place stderr progress for interactive (Bombshell). │
├──────────────────┼───────────────────────────────────────────────────────┤
│ Shared Utils     │ HTTP retry with backoff (cloudFetchRetry). First-party │
│                  │ host allowlist (NON_FIRST_PARTY_HOST gate). .env      │
│                  │ parser/writer (mode 600, POSIX-safe quoting).         │
│                  │ Risk context builder. Skill installer (npx skills     │
│                  │ add). MCP config writer (.mcp.json merge).            │
└──────────────────┴───────────────────────────────────────────────────────┘
```

---

## Deployment View

```
┌──────────────────────────────────────────────────────┐
│                  npm Registry                         │
│  registry.npmjs.org  —  @dk/jolly package            │
│  npx resolve & download                              │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│                 Customer Machine                      │
│  Node.js >= 20.12.0                                   │
│  npx @dk/jolly — dist/index.js (esbuild)             │
│                                                       │
│  Calls real services via HTTPS and spawns CLIs:      │
│                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ HTTPS to     │  │ HTTPS to     │  │ git clone  │  │
│  │ cloud.saleor │  │ auth.saleor  │  │ from       │  │
│  │ .io          │  │ .io          │  │ github.com │  │
│  └──────────────┘  └──────────────┘  └────────────┘  │
│                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ spawn: npx   │  │ spawn: npx   │  │ spawn: npx │  │
│  │ vercel       │  │ @saleor/conf│  │ pnpm       │  │
│  └──────────────┘  └──────────────┘  └────────────┘  │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│                   Saleor Cloud                         │
│                                                       │
│  ┌─────────────────┐  ┌──────────────┐  ┌──────────┐  │
│  │ Cloud API       │  │ Auth         │  │ Store    │  │
│  │ REST: orgs/     │  │ OAuth2       │  │ GraphQL  │  │
│  │ projects/envs   │  │ device grant │  │ stock,   │  │
│  │ cloud.saleor.io │  │ token refresh│  │ appInst. │  │
│  │ /platform/api   │  │ auth.saleor  │  │ *.saleor │  │
│  │                 │  │ .io/realms/  │  │ .cloud/  │  │
│  │                 │  │ saleor-cloud │  │ graphql/ │  │
│  └─────────────────┘  └──────────────┘  └──────────┘  │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│                    Vercel Edge                         │
│                                                       │
│  Deployed storefront (Next.js 16 / Paper)             │
│  npx vercel deploy — env vars — disable protection    │
│  *.vercel.app (public)                                │
└──────────────────────────────────────────────────────┘
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
