# C4 Model Diagrams — Jolly

Architecture diagrams in Unicode box-drawing style. All boxes are exactly
66 chars wide with centered connectors. No alignment issues.

---

## System Context (C4 Level 1)

See README.md Section 3.1.

---

## Container View (C4 Level 2)

  ┌────────────────────────────────────────────────────────────────┐
  │                     Customer's Agent (AI)                      │
  │                        Primary CLI user                        │
  │      Reads JSON envelope, approves stages via riskContext      │
  └────────────────────────────────────────────────────────────────┘

                                  │
                                  ▼

  ┌────────────────────────────────────────────────────────────────┐
  │                     Jolly CLI (Container)                      │
  │              TypeScript, esbuild -> dist/index.js              │
  │             npm publish @dk/jolly, npx-installable             │
  │                                                                │
  │             Data artifacts inside this container:              │
  │       Message Catalog (JSON) - assets/messages/cli.json        │
  │     Jolly Skill (Markdown) - assets/skills/jolly/SKILL.md      │
  │     Starter Recipe (YAML) - assets/skills/jolly/recipe.yml     │
  └────────────────────────────────────────────────────────────────┘

    |-- Spawns official CLIs (never reimplements APIs):
    |   git clone -> pnpm install -> @saleor/configurator -> npx vercel
    |-- Own code contacts (first-party only):
    |   cloud.saleor.io  |  auth.saleor.io  |  *.saleor.cloud
    |-- Human Customer (TTY) and Jolly Homepage (jolly.cool)

  The Homepage is a separate container (HTML/CSS, deployed to Vercel).
  The human opens jolly.cool/setup in a browser for agent handoff.

---

## Component View (C4 Level 3)

  ┌────────────────────────────────────────────────────────────────┐
  │                           CLI Entry                            │
  │                      @bomb.sh/args parser                      │
  │                 Flags: --json, --quiet, --yes                  │
  │                   Dispatches by command name                   │
  └────────────────────────────────────────────────────────────────┘

                                  │
                                  ▼

  Dispatches to one of four modules:

  ┌────────────────────────────────────────────────────────────────┐
  │                       Auth/Cloud Module                        │
  │              Commands: login, logout, auth status              │
  │                          create store                          │
  │             OAuth2 device grant + token management             │
  │                   .env read/write (mode 600)                   │
  │              Cloud API: orgs, projects, env CRUD               │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │                       Stage Orchestrator                       │
  │                 jolly start: 7-stage pipeline                  │
  │             Concurrency: store+storefront overlap              │
  │                   Idempotency + resumability                   │
  │                riskContext per high-risk stage                 │
  │            Spawns: git, pnpm, configurator, vercel             │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │                         Doctor Module                          │
  │              Diagnostics per group: skills, init,              │
  │             saleor, storefront, deployment, stripe             │
  │                vercel whoami, Cloud API probe,                 │
  │                  GraphQL purchasability probe                  │
  └────────────────────────────────────────────────────────────────┘

  All modules use shared utilities below:

  ┌────────────────────────────────────────────────────────────────┐
  │                     Shared Utilities Layer                     │
  │         HTTP retry (cloudFetchRetry) | Host allowlist          │
  │       .env I/O (mode 600, POSIX) | Risk context builder        │
  │        Skill installer (npx skills) | MCP config writer        │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │             Output Module (called by all modules)              │
  │         JSON envelope: command, status, summary, data,         │
  │                   checks, nextSteps, errors                    │
  │              Human TTY renderer | --quiet filter               │
  └────────────────────────────────────────────────────────────────┘

---

## Deployment View

  ┌────────────────────────────────────────────────────────────────┐
  │                          npm Registry                          │
  │                       registry.npmjs.org                       │
  │                       @dk/jolly package                        │
  │                     npx resolve & download                     │
  └────────────────────────────────────────────────────────────────┘

                                  │
                                  ▼

  ┌────────────────────────────────────────────────────────────────┐
  │                        Customer Machine                        │
  │                       Node.js >= 20.12.0                       │
  │                 npx @dk/jolly (esbuild bundle)                 │
  │                                                                │
  │                 HTTPS REST -> cloud.saleor.io                  │
  │                 HTTPS OAuth2 -> auth.saleor.io                 │
  │                HTTPS GraphQL -> *.saleor.cloud                 │
  │                    git clone -> github.com                     │
  │                      spawn -> npx vercel                       │
  │               spawn -> npx @saleor/configurator                │
  │                       spawn -> npx pnpm                        │
  └────────────────────────────────────────────────────────────────┘

                                  │
                                  ▼

  ┌────────────────────────────────────────────────────────────────┐
  │                          Saleor Cloud                          │
  │            Cloud API (cloud.saleor.io/platform/api)            │
  │           Auth (auth.saleor.io/realms/saleor-cloud)            │
  │              Store GQL (*.saleor.cloud/graphql/)               │
  └────────────────────────────────────────────────────────────────┘

                                  │
                                  ▼

  ┌────────────────────────────────────────────────────────────────┐
  │                          Vercel Edge                           │
  │            Deployed storefront (Next.js 16, Paper)             │
  │                       npx vercel deploy                        │
  │              Env vars + disable deploy protection              │
  │                          *.vercel.app                          │
  └────────────────────────────────────────────────────────────────┘