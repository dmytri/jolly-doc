# C4 Model Diagrams — Jolly

Architecture diagrams derived from spec-driven feature files. All diagrams use Mermaid syntax.

---

## 1. Context Diagram (Level 1)

The Jolly system boundary, its two user roles, and the seven external systems it interacts with.

```mermaid
C4Context
  title System Context — Jolly

  Person(ai_agent, "AI Agent", "Customer's coding agent — primary CLI user")
  Person(human, "Human Customer", "Owns Saleor/Vercel/Stripe accounts, completes interactive gates")

  System_Boundary(jolly, "Jolly") {
    System(jolly_cli, "Jolly CLI", "@dk/jolly — npx-installable CLI")
  }

  System_Ext(saleor_cloud_api, "Saleor Cloud API", "cloud.saleor.io/platform/api — org/project/env CRUD")
  System_Ext(saleor_auth, "Saleor Auth (Keycloak)", "auth.saleor.io — OAuth2 device grant")
  System_Ext(saleor_graphql, "Saleor Store GraphQL", "*.saleor.cloud/graphql/ — store queries, stock, app install")
  System_Ext(github, "GitHub", "github.com — clone Paper storefront, skills")
  System_Ext(vercel_cli, "Vercel CLI", "npx vercel — deploy storefront, env vars")
  System_Ext(configurator, "@saleor/configurator", "npx @saleor/configurator — store config as code")
  System_Ext(stripe_app, "Stripe App (via Saleor)", "Stripe-v2.saleor.app — payment gateway, installed via appInstall")

  Rel(ai_agent, jolly_cli, "Runs npx @dk/jolly <command>", "stdin/stdout JSON envelope")
  Rel(human, jolly_cli, "Runs in interactive terminal", "TTY, stdio passthrough")

  Rel(jolly_cli, saleor_cloud_api, "POST/GET orgs, projects, environments", "HTTPS REST")
  Rel(jolly_cli, saleor_auth, "Device authorization grant, token refresh", "HTTPS OAuth2")
  Rel(jolly_cli, saleor_graphql, "Stock seeding, app install, purchasability probe", "HTTPS GraphQL")
  Rel(jolly_cli, github, "Clone saleor/storefront", "git")
  Rel(jolly_cli, vercel_cli, "Spawn npx vercel deploy, login, whoami", "subprocess")
  Rel(jolly_cli, configurator, "Spawn npx @saleor/configurator deploy", "subprocess")
  Rel(jolly_cli, stripe_app, "Install via Saleor GraphQL appInstall", "via Saleor GraphQL")

  UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

---

## 2. Container Diagram (Level 2)

The Jolly system decomposed into four containers. The CLI is the primary; the homepage, message catalog, and bundled skill are supporting assets.

```mermaid
C4Container
  title Container Diagram — Jolly

  Person(ai_agent, "AI Agent")
  Person(human, "Human Customer")

  System_Boundary(jolly, "Jolly") {
    Container(cli, "Jolly CLI", "TypeScript, esbuild → dist/index.js", "Entry point: @bomb.sh/args parser. Dispatches to command handlers. Emits standard JSON envelope.")
    Container(homepage, "Jolly Homepage", "HTML/CSS, Vercel", "jolly.cool — agent setup instructions, /setup page")
    Container(messages, "Message Catalog", "JSON", "assets/messages/cli.json — all human-facing copy by key")
    Container(skill, "Jolly Skill", "Markdown SKILL.md", "assets/skills/jolly/ — bundled agent playbook")
  }

  System_Ext(saleor_cloud_api, "Saleor Cloud API")
  System_Ext(saleor_auth, "Saleor Auth (Keycloak)")
  System_Ext(saleor_graphql, "Saleor Store GraphQL")
  System_Ext(github, "GitHub")
  System_Ext(vercel_cli, "Vercel CLI")
  System_Ext(configurator, "@saleor/configurator")
  System_Ext(stripe_app, "Stripe App (via Saleor)")
  System_Ext(npm_registry, "npm Registry", "registry.npmjs.org")

  Rel(ai_agent, cli, "npx @dk/jolly <command>")
  Rel(human, cli, "jolly <command> (interactive)")
  Rel(human, homepage, "Opens jolly.cool/setup", "browser")

  Rel(cli, messages, "Reads catalog at startup", "import")
  Rel(cli, skill, "Installs from bundled copy", "file copy")
  Rel(cli, saleor_cloud_api, "Cloud API requests")
  Rel(cli, saleor_auth, "Device grant, token refresh")
  Rel(cli, saleor_graphql, "Store operations")
  Rel(cli, github, "Git clone")
  Rel(cli, vercel_cli, "Spawn subprocess")
  Rel(cli, configurator, "Spawn subprocess")
  Rel(cli, stripe_app, "Install via Saleor GraphQL")

  Rel(homepage, github, "Deploys from assets/homepage/")
  Rel(npm_registry, cli, "npm publish, npx resolves")
```

---

## 3. Component Diagram (Level 3)

The CLI container's internal components.

```mermaid
C4Component
  title Component Diagram — Jolly CLI

  Person(ai_agent, "AI Agent")
  Person(human, "Human Customer")

  System_Boundary(cli, "Jolly CLI") {
    Component(entry, "CLI Entry", "@bomb.sh/args parser", "Parses argv, dispatches to command handler. Single parser seam for GLOBAL_BOOLEAN_FLAGS (--json, --quiet, --yes).")
    Component(auth, "Auth Module", "OAuth2 device grant, .env read/write", "login/logout/auth status flows. Token storage and refresh.")
    Component(cloud_api, "Cloud API Client", "HTTPS REST", "Org list, project create/reuse, environment create with async task polling.")
    Component(orchestrator, "Stage Orchestrator", "jolly start pipeline", "Chains stages: store, storefront, recipe, stock, deploy, stripe. Concurrency, idempotency gates, resumability.")
    Component(doctor, "Doctor Module", "Diagnostic check groups", "Per-group runners: skills, init, saleor, storefront, deployment, stripe. Vercel whoami, Saleor probe, purchasability check.")
    Component(output, "Output Module", "Envelope builder + human renderer", "JSON envelope builder, human TTY renderer, --quiet filter. Colour/emoji/TTY detection.")
    Component(skill_installer, "Skill Installer", "npx skills add orchestration", "Installs Jolly skill + Saleor agent-skills concurrently. .mcp.json merge, AGENTS.md marker section.")
    Component(risk, "Risk Context Builder", "Structured metadata", "Builds riskContext for each high-risk stage: action, target, riskLevel, categories, reversible, sideEffects, dryRunAvailable.")
    Component(shared, "Shared Utilities", "HTTP retry, host allowlist, env helpers", "cloudFetchRetry, first-party host predicate, .env parser/writer (mode 600, POSIX-safe).")

    Rel(ai_agent, entry, "npx @dk/jolly <command>")
    Rel(human, entry, "jolly <command> (interactive)")

    Rel(entry, auth, "Dispatches login/logout/auth status")
    Rel(entry, cloud_api, "Dispatches create store")
    Rel(entry, orchestrator, "Dispatches start")
    Rel(entry, doctor, "Dispatches doctor")
    Rel(entry, skill_installer, "Dispatches init, skills install/update")
    Rel(entry, output, "Writes result through output module")
    Rel(entry, risk, "Builds riskContext for preview/execution")
    Rel(orchestrator, output, "Emits per-stage progress and envelope")
    Rel(orchestrator, risk, "Emits riskContext for each high-risk stage")
    Rel(auth, shared, "Uses env helpers for .env read/write")
    Rel(cloud_api, shared, "Uses HTTP retry, host allowlist")
    Rel(doctor, shared, "Uses host allowlist, env helpers")

    Rel(auth, saleor_auth, "Device grant, token refresh")
    Rel(cloud_api, saleor_cloud_api, "Org/project/env CRUD")
    Rel(orchestrator, saleor_graphql, "Stock seeding, app install")
    Rel(orchestrator, github, "Git clone storefront")
    Rel(orchestrator, vercel_cli, "Spawn vercel deploy/login")
    Rel(orchestrator, configurator, "Spawn configurator deploy")
    Rel(doctor, vercel_cli, "Spawn vercel whoami")
    Rel(doctor, saleor_graphql, "Purchasability probe")
    Rel(skill_installer, github, "Clone skills via npx skills add")
  }

  System_Ext(saleor_auth, "Saleor Auth (Keycloak)")
  System_Ext(saleor_cloud_api, "Saleor Cloud API")
  System_Ext(saleor_graphql, "Saleor Store GraphQL")
  System_Ext(github, "GitHub")
  System_Ext(vercel_cli, "Vercel CLI")
  System_Ext(configurator, "@saleor/configurator")

  UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

---

## 4. Dynamic Diagram — `jolly start` Sequence

The end-to-end `jolly start --json --yes` call sequence across all seven stages, showing which component invokes which external system.

```mermaid
sequenceDiagram
  actor Agent as AI Agent
  participant Entry as CLI Entry
  participant Auth as Auth Module
  participant Cloud as Cloud API Client
  participant Stage as Stage Orchestrator
  participant Doctor as Doctor Module
  participant Skill as Skill Installer
  participant Shared as Shared Utilities
  participant SaleorA as Saleor Auth
  participant SaleorC as Saleor Cloud API
  participant SaleorG as Saleor GraphQL
  participant GitHub as GitHub
  participant Vercel as Vercel CLI
  participant Conf as @saleor/configurator

  Agent->>Entry: npx @dk/jolly start --json --yes
  Entry->>Stage: Dispatch start
  Entry->>Entry: Parse flags, set --yes pre-approval
  Entry->>Output: Init envelope

  Note over Stage,Skill: Stage 0 — Bootstrap (init)
  Stage->>Skill: Install Jolly skill + Saleor agent-skills
  Skill->>GitHub: npx skills add <refs> (concurrent)
  GitHub-->>Skill: Skills installed
  Skill->>Shared: Merge .mcp.json, update AGENTS.md
  Stage-->>Stage: Bootstrap complete

  Note over Stage,SaleorA: Stage 1 — Auth
  alt No token configured
    Stage->>Auth: Device authorization grant
    Auth->>SaleorA: POST device/code (client_id=jolly)
    SaleorA-->>Auth: device_code, user_code, verification_uri
    Auth-->>Agent: Print verification URL on stderr
    Agent-->>Agent: Relay URL to human, wait for approval
    Auth->>SaleorA: Poll token endpoint
    SaleorA-->>Auth: access_token, refresh_token
    Auth->>Shared: Write JOLLY_SALEOR_ACCESS_TOKEN, REFRESH_TOKEN to .env
  else Token present
    Auth->>SaleorC: GET organizations/ (verify token)
    SaleorC-->>Auth: 200 OK, org list
  end

  Note over Stage,SaleorC: Stage 2 — Store Provisioning
  alt No SALEOR_URL configured
    Stage->>Cloud: Create environment
    Cloud->>SaleorC: POST /organizations/{org}/environments/
    SaleorC-->>Cloud: task_id
    Cloud->>SaleorC: Poll GET task-status/{task_id}
    SaleorC-->>Cloud: SUCCEEDED → domain URL
    Cloud->>Shared: Write SALEOR_URL, SALEOR_TOKEN, NEXT_PUBLIC_SALEOR_API_URL to .env
  else Already configured
    Stage-->>Stage: Detect existing, skip
  end

  Note over Stage,Conf: Stage 3 — Recipe Deploy
  Stage->>Conf: Spawn npx @saleor/configurator deploy --config recipe.yml
  Conf-->>Stage: exit 0 → deploy report
  Stage->>SaleorG: GET store back, verify catalog entities (e.g. featured-products collection)
  SaleorG-->>Stage: Entities confirmed

  Note over Stage,SaleorG: Stage 4 — Stock Seed
  Stage->>SaleorG: productVariantStocksCreate (concurrent per variant)
  SaleorG-->>Stage: Stock entries created
  Stage->>SaleorG: collectionAssignProduct (concurrent per collection)
  SaleorG-->>Stage: Collection assignments done

  Note over Stage,GitHub: Stage 5 — Storefront
  Stage->>GitHub: git clone saleor/storefront (main)
  GitHub-->>Stage: Storefront cloned
  Stage->>Stage: Remove upstream .git, git init
  Stage->>Stage: Spawn pnpm install
  Stage->>Stage: Approve native build scripts (sharp, esbuild)

  Note over Stage,Vercel: Stage 6 — Deploy
  alt No Vercel session
    Stage->>Vercel: Spawn npx vercel login
    Vercel-->>Stage: Device URL for human sign-in
    Stage->>Shared: Wait for sign-in completion
  end
  Stage->>Vercel: Spawn npx vercel --prod
  Vercel-->>Stage: Deployed *.vercel.app URL
  Stage->>Vercel: Set env vars (NEXT_PUBLIC_SALEOR_API_URL, etc.)
  Stage->>Vercel: Disable Deployment Protection

  Note over Stage,SaleorG: Stage 7 — Stripe Install
  Stage->>SaleorG: appInstall(manifestUrl, HANDLE_PAYMENTS)
  SaleorG-->>Stage: App installed
  Stage-->>Agent: Announce keys + channel-mapping human gate in nextSteps

  Stage->>Doctor: Run jolly doctor automatically
  Doctor->>Doctor: Run all check groups
  Stage->>Output: Build final envelope with all stage results + doctor checks
  Output-->>Agent: JSON envelope (stdout, single line)
```

---

## 5. Deployment Diagram

Node architecture: where each container runs and how they connect.

```mermaid
C4Deployment
  title Deployment Diagram — Jolly

  Deployment_Node(customer_machine, "Customer Machine", "Node.js >= 20.12.0, any OS") {
    Deployment_Node(npx_env, "npx Execution Environment", "") {
      Container(jolly_cli, "Jolly CLI", "TypeScript → esbuild", "dist/index.js at @dk/jolly")
    }
  }

  Deployment_Node(ci_runner, "Test/CI Runner", "Node.js >= 23, Linux") {
    Deployment_Node(test_env, "Test Environment", "") {
      Container(test_suite, "BDD Test Suite", "Cucumber.js, TypeScript", "features/, step_definitions/, support/")
      Container(eval_agent, "Eval Baseline Agent", "pi-coding-agent", "npx pi --model <model>")
    }
  }

  Deployment_Node(saleor_cloud, "Saleor Cloud", "Saleor-managed infrastructure") {
    Deployment_Node(saleor_api, "Cloud API", "") {
      ContainerDb(cloud_api_db, "Cloud API DB", "Organizations, Projects, Environments")
    }
    Deployment_Node(saleor_auth_node, "Auth Service", "") {
      ContainerDb(auth_db, "Keycloak Realm DB", "saleor-cloud realm, device codes, sessions")
    }
    Deployment_Node(saleor_store, "Store Instance", "*.saleor.cloud") {
      ContainerDb(store_graphql, "Store GraphQL API", "Products, orders, customers, channels")
    }
  }

  Deployment_Node(vercel_edge, "Vercel Edge", "") {
    Deployment_Node(vercel_project, "Vercel Project", "") {
      Container(storefront, "Storefront (Paper)", "Next.js 16, deployed from Paper clone")
    }
    Deployment_Node(homepage_node, "Vercel Project", "") {
      Container(homepage, "Homepage", "jolly.cool — HTML/CSS setup page")
    }
  }

  Deployment_Node(github_node, "GitHub", "") {
    Container(git_repos, "Repositories", "saleor/storefront, dmytri/jolly, saleor/agent-skills")
  }

  Deployment_Node(npm_registry_node, "npm Registry", "registry.npmjs.org") {
      Container(npm_package, "@dk/jolly package", "Published tarball")
  }

  Rel(jolly_cli, npm_package, "npx resolves and downloads")
  Rel(jolly_cli, cloud_api_db, "HTTPS REST")
  Rel(jolly_cli, auth_db, "HTTPS OAuth2 device grant")
  Rel(jolly_cli, store_graphql, "HTTPS GraphQL queries")
  Rel(jolly_cli, git_repos, "git clone")
  Rel(jolly_cli, vercel_project, "Spawned npx vercel deploy")
  Rel(jolly_cli, storefront, "Spawned npx vercel targets Paper")

  Rel(test_suite, jolly_cli, "Invokes in subprocess during @sandbox/@logic tests")
  Rel(eval_agent, jolly_cli, "Invokes during @eval affordance measurement")

  Rel(storefront, store_graphql, "Data queries during runtime")
```

---

## 6. Test Infrastructure Deployment

How the verification suite is structured and deployed on the CI/test runner.

```mermaid
C4Deployment
  title Deployment — Verification Suite

  Deployment_Node(runner, "Test Runner VM", "7.9 GB RAM, Linux, Node.js >= 23") {
    Deployment_Node(jolly_repo, "Jolly Repository", "git clone — clean tree") {
      Container(features, "Feature Files", "Gherkin .feature files", "30 feature files, ~400 scenarios")
      Container(steps, "Step Definitions", "TypeScript", "Cucumber step implementations")
      Container(support, "Support Code", "TypeScript", "Hooks, world, sandbox provision, PTY, plank conformance")
    }

    Deployment_Node(test_profiles, "Cucumber Profiles", "Parallel vs serial execution") {
      Container(logic_profile, "@logic Profile", "Parallel, 4 workers", "Fast assertions — 187 scenarios")
      Container(sandbox_light_profile, "@sandbox Light Profile", "Parallel, 2 workers", "Real-service queries — ~15 scenarios")
      Container(sandbox_heavy_profile, "@sandbox Heavy Profile", "Serial", "Full jolly start, deploys — ~41 scenarios")
      Container(eval_profile, "@eval Profile", "Serial", "Live baseline agent — 3 scenarios")
    }

    Deployment_Node(harness, "Sandbox Harness", "") {
      Container(provisioner, "Environment Provisioner", "Creates/deletes Cloud envs", "jolly-cannon-fodder-<run-id> naming")
      Container(reclaimer, "Capacity Reclaimer", "Scans for leftovers", "BeforeAll: delete leaked jolly-cannon-fodder envs")
      Container(shared_store, "Shared Store Cache", "Persistent marker file", "One store reused across runs, self-heals on 404")
      Container(teardown, "Best-effort Teardown", "AfterAll: remove created resources", "Retries on transient network failure")
    }

    Container(dot_env, ".env Secrets", "git-ignored", "JOLLY_SALEOR_CLOUD_TOKEN, HARNESS_OPENROUTER_API_KEY")
    Container(vercel_session, "Vercel CLI Session", "~/.vercel", "Fitted manually for sandbox and eval tiers")
  }

  Deployment_Node(saleor_org, "Saleor Cloud Org", "Dedicated test organization") {
    ContainerDb(test_envs, "jolly-cannon-fodder-* Environments", "2-slot capacity", "Namespaced, disposable, reclaimed aggressively")
    ContainerDb(shared_env, "Shared Store Environment", "Persistent", "Cached across runs, never reclaimed by name")
  }

  Rel(logic_profile, features, "Reads and runs @logic-tagged scenarios")
  Rel(logic_profile, dot_env, "Reads credentials for real-service @logic checks")
  Rel(sandbox_light_profile, features, "Runs @sandbox (not @heavy) scenarios")
  Rel(sandbox_light_profile, harvester, "Isolated env per worker")
  Rel(sandbox_heavy_profile, features, "Runs @sandbox @heavy scenarios serially")
  Rel(eval_profile, features, "Runs @eval scenarios")
  Rel(eval_profile, vercel_session, "Spawns npx vercel with provided session")

  Rel(provisioner, test_envs, "Creates & polls environments")
  Rel(reclaimer, test_envs, "Deletes leftover cannon-fodder envs")
  Rel(shared_store, shared_env, "Reuses or self-heals")
  Rel(teardown, test_envs, "Deletes after scenario/scenario outline")
```

---

## Legend

```
Shape         Meaning
─────         ───────
Person        A human or AI actor
System        The Jolly system boundary
Container     A deployable unit within Jolly
Component     A logical module within a container
System_Ext    An external system Jolly depends on
ContainerDb   A database or data store

Arrows
──────
──→           Relationship / data flow
..→           Asynchronous or spawned interaction

Tags
────
$tags=""     Used to colour-code: person (orange), system (green),
             external_system (blue), container (green/grey),
             component (green/light)
```
