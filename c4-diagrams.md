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

| Container | Technology | What it does | Delivered how |
|-----------|------------|--------------|---------------|
| Jolly CLI | TypeScript, esbuild, dist/index.js | CLI entry, command handlers, 7-stage pipeline | Published npm package @dk/jolly, npx-installable |
| Message Catalog | JSON file | All human-facing copy resolved by key at runtime | Bundled in npm package, assets/messages/cli.json |
| Jolly Skill | Markdown SKILL.md | Agent playbook for supervising jolly start | Bundled in npm package, assets/skills/jolly/ |
| Jolly Homepage | HTML/CSS | jolly.cool — setup instructions, agent handoff page | Deployed to Vercel as separate project from assets/homepage/ |

The CLI container is decomposed in the Component View. The Skill and Homepage are human-authored assets (Shipshape rules in AGENTS.md).

---

## Component View (C4 Level 3)

CLI internals — each component is a logical module in `src/`. The dispatch flow is:

`CLI Entry (parser)` → dispatches to → `Auth/Cloud`, `Stage Orchestrator`, `Doctor`, or `Output` → all use → `Shared Utilities`

| Module | Responsibilities |
|--------|------------------|
| CLI Entry | @bomb.sh/args parser. Single parser seam for all flags (--json, --quiet, --yes). Dispatches to command handler. Handles --help, --dry-run, unknown commands. |
| Auth / Cloud | OAuth2 device grant (Keycloak). Token storage + refresh. .env read/write. Cloud API: org list, project CRUD, environment create with task polling. Commands: login, logout, auth status, create store. |
| Stage Orchestrator | jolly start — 7-stage pipeline with concurrency (store + storefront overlap). Idempotency gates, resumability detection from .env/local dirs/Vercel state. Emits riskContext per high-risk stage. Spawns official CLIs. |
| Doctor Module | Diagnostic checks per group: skills, init, saleor, storefront, deployment, stripe. Runs vercel whoami, Cloud API organizations probe, GraphQL purchasability probe. Reports per-check status. |
| Output Module | JSON envelope builder (command, status, summary, data, checks, nextSteps, errors). Human TTY renderer with colour/emoji/TTY detection. --quiet filter. In-place stderr progress. |
| Shared Utils | HTTP retry (cloudFetchRetry). Host allowlist (NON_FIRST_PARTY_HOST gate). .env I/O (mode 600, POSIX quoting). Risk context builder. Skill installer (npx skills add). MCP config writer (.mcp.json merge). |

---

## Deployment View

The Jolly CLI runs on the customer's machine and calls real services. It spawns official CLIs rather than reimplementing their APIs.

- **Customer Machine**: Node.js >= 20.12.0, runs `npx @dk/jolly` (dist/index.js, esbuild). Contacts cloud.saleor.io (REST), auth.saleor.io (OAuth2), github.com (git clone). Spawns npx vercel, npx @saleor/configurator, npx pnpm.
- **Saleor Cloud**: Three services — Cloud API (REST at cloud.saleor.io/platform/api, org/project/env CRUD), Auth (Keycloak at auth.saleor.io/realms/saleor-cloud, device grant + token refresh), Store GraphQL (*.saleor.cloud/graphql/, stock seeding, appInstall, purchasability).
- **npm Registry**: registry.npmjs.org — npx resolves and downloads @dk/jolly package.
- **Vercel Edge**: Deployed storefront (Next.js 16 / Paper), npx vercel deploy, env vars, disable deploy protection.
- **GitHub**: github.com — git clone saleor/storefront (Paper template), npx skills add refs.

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
