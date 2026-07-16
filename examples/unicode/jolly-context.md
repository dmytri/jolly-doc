# Jolly System Context — Unicode Box Drawing

Renders in any terminal, no tools needed. Uses Unicode box-drawing characters (U+2500–U+257F).

```
┌─────────────────────────────────────────────────────────────┐
│                    Jolly — System Context                     │
└─────────────────────────────────────────────────────────────┘

                    ┌──────────────────────────┐
                    │      AI Agent             │
                    │  (customer's coding       │
                    │   agent, primary user)    │
                    └────────────┬─────────────┘
                                 │ npx @dk/jolly <command>
                                 │ JSON envelope (stdin/stdout)
                                 ▼
  ┌────────────────────────────────────────────────────────────────┐
  │                     Jolly CLI                                   │
  │               @dk/jolly — TypeScript → esbuild bundle           │
  │                                                                  │
  │  login │ logout │ auth status │ init │ start │ doctor           │
  │  upgrade │ skills │ create store │ completion                   │
  └──┬───────┬───────┬──────┬──────┬──────┬────────────────────────┘
     │       │       │      │      │      │
     │       │       │      │      │      │
     ▼       ▼       ▼      ▼      ▼      ▼
  ┌──────┐ ┌─────┐ ┌────┐ ┌────┐ ┌────┐ ┌─────────┐
  │Saleor│ │Sale │ │Git │ │pnpm│ │Ver.│ │@saleor  │
  │Cloud │ │or   │ │Hub │ │    │ │cel │ │Configur.│
  │API   │ │Auth │ │    │ │    │ │CLI │ │         │
  └──────┘ └─────┘ └────┘ └────┘ └────┘ └─────────┘

                    ┌──────────────────────────┐
                    │    Human Customer         │
                    │  (account owner,          │
                    │   interactive gates)      │
                    └──────────────────────────┘


Legend:
  Saleor Cloud API  — cloud.saleor.io          (REST CRUD)
  Saleor Auth       — auth.saleor.io           (OAuth2 device grant)
  GitHub            — github.com               (git clone storefront)
  pnpm              — storefront deps          (spawn subprocess)
  Vercel CLI        — npx vercel               (deploy)
  @saleor/Configur. — npx @saleor/configurator (store config as code)
```

---

## Vercel deploy sequence — Unicode flowchart

Shows a sub-section of the architecture with richer box styles.

```
            Vercel Deploy Flow

  ┌─────────────────────────────────────────────────────┐
  │   jolly start — Deploy Stage                        │
  │                                                     │
  │  1. Check Vercel session                            │
  │     │                                               │
  │     ├── No session → npx vercel login               │
  │     │                 └── Print device URL → human  │
  │     │                 └── Wait for sign-in           │
  │     │                                               │
  │     └── Session OK → skip                           │
  │                                                     │
  │  2. npx vercel --prod  (spawn)                      │
  │     │                                               │
  │     ├── ─ ─ ─ set env vars ─ ─ ─ > Vercel Project   │
  │     │       NEXT_PUBLIC_SALEOR_API_URL              │
  │     │       NEXT_PUBLIC_DEFAULT_CHANNEL             │
  │     │                                               │
  │     └── ─ ─ ─ disable deploy protection ─ ─ > Vercel│
  │                                                     │
  │  3. Real deployed *.vercel.app URL captured          │
  │     from CLI output (never fabricated)              │
  └─────────────────────────────────────────────────────┘
```
