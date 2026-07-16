# C4 Model Diagrams — Jolly

Architecture diagrams in Unicode box-drawing style. Same format as the system context
diagram in `README.md` Section 3.1 — renders in any terminal, no tools needed.

---

## System Context (C4 Level 1)

```
                     ┌───────────────────────┐
                     │    Customer's Agent    │── CLI (npx, --json envelope)
                     └───────────┬───────────┘
                                 │
                                 ▼
          ┌──────────────────────────────────────────────────────────┐
          │                        Jolly CLI                         │
          │  @dk/jolly  (npx-installable, TypeScript → esbuild)      │
          │                                                          │
          │  Subcommands: login, logout, auth status, init, start,   │
          │  doctor, upgrade, skills, create store, completion       │
          ├────────┬────────┬────────┬────────┬────────┬────────────┤
          │        │        │        │        │        │            │
          ▼        ▼        ▼        ▼        ▼        ▼
     ┌────────┐┌───────┐┌──────┐┌──────┐┌──────┐┌────────┐
     │ Saleor ││ Saleor││ Git  ││ pnpm ││Vercel││ Stripe │
     │ Cloud  ││ Graph ││ Hub  ││      ││ CLI  ││ (via   │
     │ API    ││ QL    ││      ││      ││      ││ Saleor │
     │cloud.sa││*.sale││github││      ││      ││ appIn- │
     │leor.io ││or.clo││.com  ││      ││      ││stall)  │
     └────────┘└───────┘└──────┘└──────┘└──────┘└────────┘
          │        │        │        │        │        │
          │ REST   │ Graph  │ git   │ spawn │ spawn  │ Saleor
          │ CRUD   │ QL     │ clone │ pnpm  │ npx    │ GraphQL
          │        │        │       │ inst. │ vercel │ appInstall
          ▼        ▼        ▼       ▼       ▼        ▼
```

---

## Container View (C4 Level 2)

```
                     ┌─────────────────────────────────────┐
                     │         Jolly Homepage               │
                     │  jolly.cool  (Vercel-deployed)       │
                     │  /setup page for agent handoff       │
                     └────────────────┬────────────────────┘
                                      │ customer reads setup
                                      ▼
┌───────────────────────────────────────────────────────────────────────┐
│                              Jolly CLI                                │
│  @dk/jolly  —  dist/index.js  (esbuild bundle, Node.js)               │
│                                                                       │
│  ┌──────────────────────────┐     ┌────────────────────────────────┐  │
│  │    Message Catalog        │     │        Jolly Skill             │  │
│  │  assets/messages/cli.json │     │  assets/skills/jolly/SKILL.md  │  │
│  │  All human-facing copy    │     │  Bundled agent playbook        │  │
│  │  resolved by key at runtim│     │  Installed via npx skills add  │  │
│  └──────────────────────────┘     └────────────────────────────────┘  │
│                                                                       │
│  Spawns official CLIs for mechanical work:                            │
│    git clone  →  pnpm install  →  @saleor/configurator  →  npx vercel│
└───────────────────────────────────────────────────────────────────────┘
                               │
                               │ npx @dk/jolly <command>
                               ▼
                    ┌─────────────────────┐
                    │  Customer's Agent   │
                    └─────────────────────┘

  External systems Jolly talks to:
  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌────────┐  ┌──────────┐
  │ Saleor   │  │ Saleor   │  │ Saleor Store │  │ GitHub │  │ Vercel   │
  │ Cloud API│  │ Auth     │  │ GraphQL      │  │        │  │ CLI      │
  │cloud.sale│  │auth.saleo│  │*.saleor.cloud│  │github.c│  │(spawned) │
  │or.io     │  │r.io      │  │/graphql/     │  │om      │  │          │
  └──────────┘  └──────────┘  └──────────────┘  └────────┘  └──────────┘
```

---

## Component View (C4 Level 3)

CLI internals — each component is a logical module in `src/`.

```
                         ┌──────────────────────────────────────┐
                         │            CLI Entry                 │
                         │  @bomb.sh/args parser                │
                         │  Single parser seam for all flags    │
                         │  (--json, --quiet, --yes)            │
                         │  Dispatches to command handlers      │
                         └──┬────────┬────────┬────────┬────────┘
                            │        │        │        │
          ┌─────────────────┤        │        │        ├────────────────┐
          │                 │        │        │        │                │
          ▼                 ▼        ▼        ▼        ▼                │
   ┌──────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐     │
   │ Auth Module  │  │  Cloud   │  │  Stage   │  │   Doctor     │     │
   │ OAuth2 device│  │  API     │  │Orchestrat│  │   Module     │     │
   │ grant flow   │  │  Client  │  │   or     │  │ Diagnostics  │     │
   │ .env read/   │  │ Org/proj/│  │ jolly    │  │ per group:   │     │
   │ write        │  │ env CRUD │  │ start    │  │ skills, init,│     │
   │ token mgmt   │  │ task     │  │ pipeline │  │ saleor,      │     │
   │ refresh      │  │ polling  │  │ 7 stages │  │ storefront,  │     │
   └──────┬───────┘  └────┬─────┘  └────┬─────┘  │ deployment,  │     │
          │               │             │        │ stripe       │     │
          │               │             │        └──────┬───────┘     │
          │               │             │               │             │
          ▼               ▼             ▼               ▼             │
  ┌────────────────────────────────────────────────────────────┐      │
  │                    Shared Utilities                         │      │
  │  HTTP retry (cloudFetchRetry) | Host allowlist | .env I/O  │      │
  │  POSIX-safe quoting | Mode 600 file permissions            │      │
  └────────────────────────────────────────────────────────────┘      │
                                                                      │
   ┌──────────────────────┐     ┌──────────────────┐                  │
   │   Output Module      │     │  Risk Context    │                  │
   │ Envelope builder     │     │  Builder         │                  │
   │ Human TTY renderer   │     │ Structured risk  │                  │
   │ --quiet filter       │     │ metadata per     │                  │
   │ Colour/emoji/TTY det │     │ high-risk stage  │                  │
   └──────────────────────┘     └──────────────────┘                  │
```

---

## Deployment View

```
                    ┌──────────────────────────────────────┐
                    │        Customer Machine               │
                    │  Node.js >= 20.12.0, any OS           │
                    │                                      │
                    │  npx @dk/jolly  dist/index.js        │
                    │  (esbuild bundle, published npm)     │
                    └──────────┬───────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┬─────────────────┐
          │                    │                    │                 │
          ▼                    ▼                    ▼                 ▼
  ┌───────────────┐  ┌───────────────┐  ┌──────────────┐  ┌──────────────┐
  │  Saleor Cloud │  │  Saleor Auth  │  │   GitHub     │  │  npm Registry│
  │  cloud.saleor │  │  auth.saleor  │  │  github.com  │  │registry.npmjs│
  │  .io/platform │  │  .io/realms/  │  │  git clone   │  │  .org        │
  │  /api         │  │  saleor-cloud │  │  saleor/     │  │  npx resolve │
  │  (REST CRUD)  │  │  (OAuth2)     │  │  storefront  │  │  + download  │
  └───────┬───────┘  └───────┬───────┘  └──────────────┘  └──────────────┘
          │                  │
          │                  │
          ▼                  ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                   Saleor Store Instance                       │
  │              *.saleor.cloud/graphql/                          │
  │                                                               │
  │  GraphQL API: stock seeding, appInstall, purchasability probe │
  │  Dashboard:   *.saleor.cloud/dashboard/                      │
  └──────────────────────────────────────────────────────────────┘

                    ┌────────────────────────────────┐
                    │         Vercel Edge             │
                    │                                │
                    │  Deployed storefront            │
                    │  (Next.js 16 / Paper)           │
                    │  npx vercel deploy               │
                    │                                │
                    │  Set env vars + disable protect │
                    └────────────────────────────────┘
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
