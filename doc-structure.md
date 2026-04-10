# Documentation Structure — `wbe-onecart-ui-refactor`

This directory contains structured documentation to help developers **understand, onboard, and refactor** the codebase efficiently.

---

## 📂 Folder Structure

```text
docs/
├── README.md                        # Docs index / table of contents
│
├── 01-getting-started/
│   ├── onboarding.md                # Day-1 setup guide for new devs
│   ├── prerequisites.md             # Node, npm, IDE, tools, access
│   ├── local-setup.md               # Clone, install, env vars, proxy.conf
│   ├── running-the-app.md           # npm scripts, dev server, debugging
│   └── troubleshooting.md           # Common errors & fixes
│
├── 02-architecture/
│   ├── overview.md                  # High-level architecture diagram
│   ├── tech-stack.md                # Angular 17.3.8, RxJS, libs used
│   ├── folder-structure.md          # src/ tree explained
│   ├── routing.md                   # App routes & lazy loading
│   ├── state-management.md          # Services, BehaviorSubjects, storage
│   ├── api-layer.md                 # HTTP services, interceptors
│   ├── headless-strategy.md         # src/app/headless explained
│   └── theming-styles.md            # SCSS structure, tenant theming
│
├── 03-modules/                      # One file per feature module
│   ├── activities.md
│   ├── cart.md
│   ├── gift-card.md
│   ├── golf.md
│   ├── headless.md
│   ├── layouts.md
│   ├── room.md
│   ├── seat.md
│   ├── snc.md
│   ├── spa.md
│   ├── tenant-properties.md
│   └── unit.md
│
├── 04-shared/
│   ├── overview.md                  # What lives in src/app/shared
│   ├── services.md                  # Global services
│   ├── components-widgets.md        # Reusable UI widgets
│   ├── directives.md
│   ├── pipes.md
│   ├── interceptors.md              # HTTP interceptors
│   ├── utils.md                     # Helper functions
│   └── constants.md
│
├── 05-api-integration/
│   ├── backend-endpoints.md         # List of APIs consumed
│   ├── service-catalog.md           # Each *.service.ts mapped to endpoints
│   ├── auth-flow.md                 # Tokens, session, tenant auth
│   └── error-handling.md
│
├── 06-build-deploy/
│   ├── build-process.md             # Angular build, webpack customizations
│   ├── docker.md                    # Dockerfile walkthrough
│   ├── nginx.md                     # nginx config
│   ├── ci-cd.md                     # azure-pipelines.yml explained
│   ├── environments.md              # dev/stage/prod configs
│   └── versioning.md                # updateVersion.js, release tags
│
├── 07-development/
│   ├── coding-standards.md          # Naming, file conventions
│   ├── git-workflow.md              # Branching (develop, hotfix/*), PRs
│   ├── commit-conventions.md
│   ├── code-review.md
│   ├── testing.md                   # Unit tests, specs
│   └── adding-a-feature.md          # Step-by-step checklist
│
├── 08-domain-knowledge/
│   ├── glossary.md                  # WBE, OneCart, SNC, tenant, etc.
│   ├── business-flows.md            # Booking flow end-to-end
│   ├── user-journeys.md             # Guest booking, cart, checkout
│   └── tenant-model.md              # Multi-tenant concepts
│
├── 09-reference/
│   ├── scripts.md                   # package.json scripts
│   ├── dependencies.md              # Key npm packages & why
│   ├── environment-variables.md
│   └── faq.md
│
└── assets/
    ├── diagrams/                    # Architecture, sequence diagrams
    └── screenshots/
