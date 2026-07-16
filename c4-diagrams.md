# C4 Model Diagrams — Jolly

All diagrams follow README.md Section 3.1 format: box with splitter
bottom, text columns below. One vertical flow per diagram.

---

## System Context (C4 Level 1)

See README.md Section 3.1.

---

## Container View (C4 Level 2)

Two deployable containers. The CLI is the primary; the Homepage
provides the setup guide. They do not communicate with each other.

┌────────────────────────────────────────────────────────────────┐
│                     Jolly CLI (Container)                      │
│                                                                │
│  @dk/jolly — npm package, npx-installable                      │
│  TypeScript, esbuild -> dist/index.js                          │
│                                                                │
│  Subcommands: login, logout, auth status, init, start,         │
│  doctor, upgrade, skills, create store, completion             │
│                                                                │
│  Artifacts inside this container:                              │
│  Message Catalog  —  assets/messages/cli.json  (JSON)          │
│  Jolly Skill      —  assets/skills/jolly/SKILL.md  (Markdown)  │
│  Starter Recipe   —  assets/skills/jolly/recipe.yml  (YAML)    │
└────────────────┬───────────────────────────────────────────────┘
    AI Agent                     Human Customer                
 (primary user)                 (account owner)                
 npx @dk/jolly                interactive terminal             

  Jolly Homepage (Container):  jolly.cool  —  HTML/CSS, Vercel-deployed
  Ships under assets/homepage/ in the same repo.
  The Human Customer reads the setup guide there.

---

## Component View (C4 Level 3)

The CLI Entry dispatches by command name. All modules use shared utilities.

┌──────────────────────────────────────────────────────────────────────┐
│                      CLI Entry (@bomb.sh/args)                       │
│                                                                      │
│  Single parser seam: GLOBAL_BOOLEAN_FLAGS (--json, --quiet, --yes)   │
│  Dispatches: login -> Auth, start -> Orchestrator, doctor -> Doctor  │
└──────────────────────────────────────────────────────────────────────┘

                                   │
                                   │
                                   ▼

┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│                                                                    │
│                                                                    │
│  Auth/Cloud Module:  login, logout, auth status, create store      │
│    OAuth2 device grant, .env I/O, Cloud API CRUD                   │
│    -> Saleor Auth, -> Saleor Cloud                                 │
│                                                                    │
│  Stage Orchestrator:  jolly start (7-stage pipeline)               │
│    Concurrency, idempotency, riskContext per stage                 │
│    -> Saleor GQL, -> Stripe GQL                                    │
│    Spawns: git, pnpm, configurator, vercel                         │
│                                                                    │
│  Doctor Module:  diagnostics per group                             │
│    skills, init, saleor, storefront, deploy, stripe                │
│    vercel whoami, Cloud API probe, GQL purch. probe                │
│    -> Vercel CLI, -> Cloud API, -> Saleor GQL                      │
│                                                                    │
│  Output Module:  JSON envelope builder                             │
│    { command, status, summary, data, checks, nextSteps, errors }   │
│    Human TTY renderer (colour, emoji, TTY detection)               │
│    --quiet filter, in-place stderr progress                        │
└────────────────────────────────────────────────────────────────────┘

                                  │
                                  │
                                  ▼

┌────────────────────────────────────────────────────────────────────────────┐
│                           Shared Utilities Layer                           │
│                                                                            │
│  HTTP retry (cloudFetchRetry)  |  Host allowlist (NON_FIRST_PARTY_HOST)    │
│  .env I/O (mode 600, POSIX quoting)  |  Risk context builder               │
│  Skill installer (npx skills add)  |  MCP config writer (.mcp.json merge)  │
└────────────────────────────────────────────────────────────────────────────┘

---

## Deployment View

┌──────────────────────────────────────────────────────┐
│        Customer Machine (Node.js >= 20.12.0)         │
│                                                      │
│  npx @dk/jolly  --  dist/index.js  (esbuild bundle)  │
│                                                      │
│  Calls real services:                                │
│    HTTPS REST -> cloud.saleor.io                     │
│    HTTPS OAuth2 -> auth.saleor.io                    │
│    HTTPS GraphQL -> *.saleor.cloud                   │
│    git clone -> github.com                           │
│                                                      │
│  Spawns official CLIs (never reimplements APIs):     │
│    npx vercel  (deploy, login, whoami)               │
│    npx @saleor/configurator  (recipe deploy)         │
│    npx pnpm  (storefront dependencies)               │
└──────────────────┬────────────────┬────────────┬─────┘
 Saleor Cloud API   Saleor Auth      GitHub   Vercel Edge
 cloud.saleor.io   auth.saleor.io  github.com *.vercel.app
    REST CRUD      OAuth2 device   git clone  deploy storefront

  npm Registry (registry.npmjs.org):
    Publishes @dk/jolly. Resolved by npx at runtime.
