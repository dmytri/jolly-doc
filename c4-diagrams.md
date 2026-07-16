# C4 Model Diagrams — Jolly

All diagrams follow README.md Section 3.1: fenced code blocks,
Unicode box drawing, splitter bottom, text columns.

---

## System Context (C4 Level 1)

See README.md Section 3.1.

---

## Container View (C4 Level 2)

```
┌──────────────────────────────────────────────────────────┐
│                  Jolly CLI (Container)                   │
│                                                          │
│  @dk/jolly — npm package, npx-installable                │
│  TypeScript, esbuild -> dist/index.js                    │
│                                                          │
│  Artifacts inside:                                       │
│  Message Catalog (JSON) — assets/messages/cli.json       │
│  Jolly Skill (Markdown) — assets/skills/jolly/SKILL.md   │
│  Starter Recipe (YAML) — assets/skills/jolly/recipe.yml  │
│                                                          │
│  Subcommands: login, logout, auth status, init, start,   │
│  doctor, upgrade, skills, create store, completion       │
└────────────────┬─────────────────────────────────────────┘
    AI Agent                  Human Customer             
 (primary user)              (account owner)             
 npx @dk/jolly             interactive terminal          
```

  Jolly Homepage (Container):  jolly.cool
  HTML/CSS, Vercel-deployed from assets/homepage/
  The Human Customer reads the setup guide there.

---

## Component View (C4 Level 3)

```
┌──────────────────────────────────────────────────────┐
│              CLI Entry (@bomb.sh/args)               │
│                                                      │
│  Single parser seam: GLOBAL_BOOLEAN_FLAGS            │
│  (--json, --quiet, --yes)                            │
│  Dispatches: login -> Auth, start -> Orchestrator,   │
│  doctor -> Doctor                                    │
└──────────────────────────────────────────────────────┘

                           │
                           │
                           ▼

┌──────────────────────────────────────────────────────┐
│    Auth/Cloud Module: login, logout, auth status,    │
│                                                      │
│    create store. OAuth2 device grant, .env I/O,      │
│    Cloud API CRUD. -> Saleor Auth, -> Saleor Cloud   │
│                                                      │
│  Stage Orchestrator: jolly start pipeline            │
│    7 stages, concurrency, idempotency,               │
│    riskContext per stage. -> Saleor GQL,             │
│    -> Stripe GQL. Spawns: git, pnpm,                 │
│    configurator, vercel                              │
│                                                      │
│  Doctor Module: diagnostics per group:               │
│    skills, init, saleor, storefront,                 │
│    deploy, stripe. vercel whoami,                    │
│    Cloud API probe, GQL probe.                       │
│    -> Vercel, -> Cloud API, -> Saleor GQL            │
│                                                      │
│  Output Module: JSON envelope builder                │
│    { command, status, summary, data,                 │
│    checks, nextSteps, errors }                       │
│    Human TTY renderer, --quiet filter                │
└──────────────────────────────────────────────────────┘

                           │
                           │
                           ▼

┌────────────────────────────────────────────────────┐
│               Shared Utilities Layer               │
│                                                    │
│  HTTP retry (cloudFetchRetry) | Host allowlist     │
│  .env I/O (mode 600, POSIX) | Risk context         │
│  Skill installer (npx skills) | MCP config writer  │
└────────────────────────────────────────────────────┘
```

---

## Deployment View

```
┌──────────────────────────────────────────────────────────────┐
│            Customer Machine (Node.js >= 20.12.0)             │
│                                                              │
│  npx @dk/jolly — dist/index.js (esbuild)                     │
│                                                              │
│  HTTPS REST -> cloud.saleor.io                               │
│  HTTPS OAuth2 -> auth.saleor.io                              │
│  HTTPS GraphQL -> *.saleor.cloud                             │
│  git clone -> github.com                                     │
│                                                              │
│  Spawns official CLIs:                                       │
│    npx vercel (deploy, login, whoami)                        │
│    npx @saleor/configurator (recipe deploy)                  │
│    npx pnpm (storefront dependencies)                        │
└─────────────────┬────────────────┬────────────┬──────────────┘
    Cloud API          Auth         GitHub       Vercel    
 cloud.saleor.io  auth.saleor.io  github.com  *.vercel.app 
       REST        OAuth2 grant   git clone      deploy    
```

  npm Registry (registry.npmjs.org) publishes @dk/jolly.
  Resolved by npx at runtime.
