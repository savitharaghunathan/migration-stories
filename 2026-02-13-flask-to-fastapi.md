---
date: 2026-02-13
topic: flask-to-fastapi
personas: [Platform / SRE, Backend Developer, Engineering Manager]
konveyor-components: [Kantra, Custom Rulesets, Kai, Hub]
migration-type: modernization
---

# Migration Stories: Flask to FastAPI on Kubernetes

## Migration Landscape

The Python web ecosystem is undergoing a structural shift. **Flask** and **Django**, the dominant frameworks for a decade, are losing ground to **FastAPI** for API-centric, cloud-native workloads. This isn't a deprecation — Flask and Django remain maintained — but a **fitness mismatch**. Cloud-native architectures demand async I/O, auto-generated API schemas, native container readiness, and sub-100ms response times. Flask's synchronous WSGI model creates friction in Kubernetes environments where services are small, independently deployable, and I/O-bound (calling LLMs, vector DBs, external APIs).

FastAPI adoption grew **40% year-over-year** (JetBrains 2025 survey), with 38% of Python developers now using it. Real-world migrations report **64% latency reduction** and **2x throughput** gains.

### Who is affected?

Any organization running Python APIs on Kubernetes — particularly teams building AI/ML platforms, data pipelines, or microservices. The AI boom has made this urgent: RAG backends, LLM orchestration layers, and streaming endpoints all expose Flask's synchronous bottleneck.

### Timeline

No hard sunset date, but compounding pressure:
- Python 3.10 EOL in **October 2026** forces version upgrades anyway
- Django 4.x EOL **April 2026** pushes Django shops to reconsider
- FastAPI is the default for new Python projects in cloud-native orgs — legacy Flask/Django APIs become the "old thing" that slows teams down

### Migration Paths

| Path | Description | Trade-off |
|------|-------------|-----------|
| **Flask to FastAPI** | Rewrite sync Flask routes to async FastAPI endpoints with Pydantic models | Highest performance gain, but requires async expertise and rethinking data access patterns |
| **Django to FastAPI + separate frontend** | Extract Django REST APIs into FastAPI, keep Django for admin/ORM or replace with SQLModel | Biggest scope — full decoupling — but unlocks true microservices |
| **Incremental (strangler fig)** | Run FastAPI alongside Flask/Django behind an API gateway, migrate route-by-route | Lowest risk, longest timeline, requires routing infrastructure |

### What does Konveyor offer today?

**Kantra/Analyzer** can run against Python codebases using custom rulesets. Flask-to-FastAPI detection rules (e.g., flagging `@app.route`, `flask.Flask`, synchronous request handlers, `requirements.txt` with `flask`) are buildable today as custom rules.

**Kai** does not yet support Python — this migration story becomes a compelling reason to add Python support.

**Custom rules path** has a low barrier. Teams can start writing detection rules now without waiting for first-party support.

## Personas

### Persona 1 — Platform / SRE

**Background:** Runs the internal Kubernetes platform for a mid-size SaaS company. Manages shared infrastructure, CI/CD pipelines, service mesh, observability, and security posture. Owns Helm charts, ArgoCD pipelines, container image scanning, and dependency audits. 15 microservices, 6 of which are Flask apps.
**Technical depth:** Expert with Kubernetes/containers/security tooling, intermediate with Python application internals
**Migration authority:** Decision maker and gatekeeper — decides what runs on the platform, sets framework standards, and can block deployments that fail security or compliance review
**What keeps them up at night:** Flask services hitting concurrency limits under AI workloads. New framework means new dependency trees to vet, new Dockerfiles, new health checks, new dashboards. Nobody budgets for their work during a migration. Last to know, first to get paged, and accountable for CVEs they didn't introduce.

### Persona 2 — Backend Developer

**Background:** Senior Python dev on the product team. Owns three Flask APIs that power the core product. Writes Python daily, knows Flask inside-out.
**Technical depth:** Expert with Flask/Python, novice with FastAPI and async patterns
**Migration authority:** Implementer — they'll do the actual rewrite
**What keeps them up at night:** Being asked to rewrite working code on a deadline. They've heard async is "better" but haven't internalized `await` patterns, and their SQLAlchemy queries are all synchronous. Breaking production scares them more than slow performance.

### Persona 3 — Engineering Manager

**Background:** Manages two backend teams (8 developers). Reports to VP Engineering. Accountable for delivery velocity and uptime SLAs.
**Technical depth:** Novice — was a developer 5 years ago, now manages process and people
**Migration authority:** Decision maker — approves sprint priorities and staffing
**What keeps them up at night:** Justifying the migration to leadership. "Why are we spending cycles rewriting something that works?" They need a business case, not a benchmark.

## Empathy Maps

### Platform / SRE — Empathy Map

**Thinks:**
- "If we migrate 6 Flask services to FastAPI, that's 6 new Dockerfiles, 6 new Helm charts, 6 new sets of health check endpoints I need to validate. Who's accounting for my time?"
- "FastAPI uses Uvicorn instead of Gunicorn — do I need to rethink worker scaling, liveness probes, and graceful shutdown behavior?"
- "Starlette, Pydantic v2, Uvicorn — that's a whole new dependency surface. Have I scanned these? Are there CVEs I'm inheriting?"

**Feels:**
- Burdened — migration creates invisible work that nobody estimates or appreciates
- Cautious — burned before by framework migrations that shipped "done" but broke observability
- Skeptical — "developers say it's faster, but faster where? Under what load? Show me the numbers on *our* cluster"

**Says:**
- "I'm fine with the migration, but I need a checklist of what changes on my side before anyone merges anything."
- "Who owns the new container image? Are we changing the base image? Does it still pass our Trivy scans?"
- "Don't tell me about it at deploy time. Tell me at design time."

**Does:**
- Audits the FastAPI dependency tree against CVE databases before approving anything
- Builds a proof-of-concept deployment of one FastAPI service to validate Helm chart patterns, readiness probes, and resource limits
- Creates a platform migration runbook that nobody asked for, because they know nobody else will

**Pain Points:**
1. Framework migrations generate infrastructure work that's invisible in sprint planning
2. New ASGI server (Uvicorn) requires different scaling, process management, and shutdown behavior than WSGI (Gunicorn)
3. Dependency security review for an entirely new framework stack — no institutional knowledge of Starlette/Pydantic CVE history

**Goals:**
1. Migration doesn't degrade observability, security posture, or deployment reliability
2. Reusable platform patterns (base Dockerfile, Helm template, CI pipeline) that other teams can adopt without hand-holding

**Unspoken Needs:**
- Being included early — not consulted as an afterthought when something breaks
- A migration tool that surfaces infrastructure impact, not just code changes

### Backend Developer — Empathy Map

**Thinks:**
- "My Flask app works. It handles 2,000 RPS and nobody complains. Why am I rewriting it?"
- "I don't really understand `async def` vs `def` in FastAPI. If I get it wrong, will requests silently block the event loop?"
- "SQLAlchemy is synchronous. Do I need to switch to async SQLAlchemy? That's not just a framework swap, that's rewriting my entire data layer."

**Feels:**
- Anxious — rewriting production code that currently works feels like unnecessary risk
- Overwhelmed — async Python, Pydantic models, dependency injection, ASGI middleware — it's a lot of new concepts at once
- Quietly excited — they've seen FastAPI's auto-generated docs and type safety and know it's objectively better for API development

**Says:**
- "Can we just do one service first and see how it goes?"
- "I need at least a week to learn async patterns before I can estimate this."
- "What's the rollback plan if the migrated service has issues?"

**Does:**
- Reads FastAPI docs and tutorials on personal time, afraid to admit at work they don't know it yet
- Starts converting the simplest, lowest-traffic Flask service as an unofficial proof of concept
- Copies patterns from Stack Overflow and FastAPI examples without fully understanding async implications
- Keeps the old Flask code around "just in case" even after migration is complete

**Pain Points:**
1. No clear mapping between Flask patterns and FastAPI equivalents (route decorators, request context, error handlers, middleware)
2. Synchronous ORM (SQLAlchemy) doesn't translate cleanly to async — half the performance benefit disappears if you wrap sync calls in `run_in_executor`
3. Testing patterns change completely — Flask's test client vs FastAPI's `TestClient` (httpx-based), pytest-asyncio setup

**Goals:**
1. Migrate without production incidents — their reputation is on the line
2. End up with code that's actually better, not just "newer"

**Unspoken Needs:**
- A pattern guide that says "if your Flask code looks like X, your FastAPI code should look like Y"
- Permission to move slowly and learn rather than sprint through a rewrite
- Automated detection of what needs to change, so they don't miss something buried in a utility module

### Engineering Manager — Empathy Map

**Thinks:**
- "My VP will ask: what's the ROI? I can't answer 'it's faster' — I need dollar amounts or customer-facing metrics."
- "If I commit two sprints to this and we ship zero features, how do I explain that in the quarterly review?"
- "The team is already stretched. Do I pull people off feature work, or hire a contractor who doesn't know our codebase?"

**Feels:**
- Torn — intellectually agrees the migration is right, but can't justify it in a roadmap packed with feature requests
- Exposed — doesn't have deep enough technical knowledge to evaluate whether the migration estimate is realistic
- Pressured — hears "we need to modernize" from platform team and "we need features" from product, simultaneously

**Says:**
- "Give me a one-pager I can take to leadership. Latency numbers, cost savings, risk mitigation."
- "Can we do this incrementally so we don't stop shipping?"
- "What's the blast radius if it goes wrong?"

**Does:**
- Asks the backend developer to timebox a proof of concept to get real numbers from their own stack
- Looks for case studies from similar companies to justify the investment
- Tries to sneak migration work into feature sprints by picking features that touch Flask services anyway — "while you're in there, migrate it"

**Pain Points:**
1. No standardized way to assess migration scope — how many routes, how many Flask-specific patterns, how much test rewrite?
2. Can't quantify business risk of *not* migrating (what's the cost of Flask's concurrency ceiling when traffic doubles?)
3. Migration progress is invisible — no dashboard showing "40% of routes migrated, 60% remaining"

**Goals:**
1. Migration happens without derailing the product roadmap
2. Clear, defensible justification they can present upward

**Unspoken Needs:**
- An automated assessment report that gives them scope and effort numbers without requiring developer time to produce
- Confidence that the team won't be stuck halfway — a migration that's 50% done is worse than not starting

## Konveyor Opportunity Analysis

| Konveyor Capability | Opportunity | Impact |
|---|---|---|
| **Analysis (Kantra)** | Custom rules to detect Flask-specific patterns: `@app.route` decorators, `flask.request` context usage, `Flask-SQLAlchemy` models, synchronous route handlers, WSGI middleware, Flask extensions. Also detect infrastructure artifacts: Dockerfiles with gunicorn, requirements.txt with flask, Helm health check paths. | **High** |
| **Custom Rulesets** | Create a `flask-to-fastapi` ruleset package with pattern equivalence guides in rule messages. Publish as a community ruleset. Additionally, a framework-agnostic `python-cloud-readiness` ruleset for missing Dockerfiles, hardcoded config, missing structured logging. | **High** |
| **AI-Assisted Refactoring (Kai)** | Python support in Kai to auto-generate FastAPI equivalents from detected Flask patterns. Currently blocked — Kai doesn't support Python. This migration story is the strongest product roadmap argument for adding it. | **High** |
| **Migration Planning (Hub)** | Track migration progress across a portfolio of Flask services: how many assessed, how many migrated, which have unresolved findings. Give the Engineering Manager their dashboard. | **Medium** |

### Gaps

| Gap | Why It Matters | Who Needs It |
|---|---|---|
| **Python language support in Kai** | Can't offer AI-assisted refactoring for the fastest-growing migration scenario in cloud-native | Backend Developer |
| **Dependency security surface analysis** | No way to compare CVE profile of source (Flask) vs target (FastAPI/Starlette/Uvicorn/Pydantic) dependency trees | Platform / SRE |
| **Migration effort estimation** | Analysis detects *what* needs to change but doesn't estimate *how much work* it is | Engineering Manager |
| **Infrastructure impact report** | Analysis focuses on application code; no first-class support for flagging Dockerfile, Helm chart, CI pipeline changes | Platform / SRE |

### Strategic Take

The biggest opportunity isn't any single rule or feature — it's that Konveyor has **zero Python presence today**. The Flask-to-FastAPI migration is the ideal wedge: high urgency (AI workloads), clear before/after, broad audience (Python is #1 in cloud-native AI/ML), and low barrier to start (custom rules work today, Kai support follows).

## Konveyor Product Stories

### Story K-1: Flask-to-FastAPI Detection Ruleset

**Persona:** All three
**Empathy context:** The developer needs to know what to change, the manager needs scope numbers, the platform/SRE needs to see infrastructure impact. One analysis run serves all three.

**Story:**
As a migration team, I want Konveyor to analyze a Flask codebase and flag every application pattern *and* infrastructure artifact that needs to change for FastAPI, so that all stakeholders see the full migration picture from a single scan.

**Acceptance Criteria:**
- [ ] Detects Flask routes, request context usage, extension imports, sync handlers, WSGI middleware
- [ ] Detects infrastructure: Dockerfile with gunicorn, requirements.txt with flask, Helm health check paths
- [ ] Categorizes findings by complexity (simple / moderate / complex)
- [ ] Summary output with counts suitable for a non-technical audience

**Konveyor component:** Kantra / Custom Rulesets
**Priority signal:** Foundation — nothing else works without this.

### Story K-2: Pattern Equivalence in Rule Messages

**Persona:** Backend Developer
**Empathy context:** Flagging a problem without showing the solution creates anxiety, not progress. They need "your Flask code looks like X, write this FastAPI code instead."

**Story:**
As a backend developer, I want each finding to include a concrete before/after code example, so that I can migrate without guessing at async patterns.

**Acceptance Criteria:**
- [ ] Each rule includes Flask -> FastAPI code example in the message
- [ ] Covers routes, request parsing, error handling, and sync-to-async ORM patterns
- [ ] Warns about common async pitfalls (blocking event loop, shared state)

**Konveyor component:** Custom Rulesets
**Priority signal:** Directly reduces ramp-up time and error rate.

### Story K-3: Python Support in Kai

**Persona:** Backend Developer
**Empathy context:** Manually rewriting 200+ routes is demoralizing. AI-generated boilerplate turns migration from a rewrite into a review task.

**Story:**
As a backend developer, I want Kai to auto-generate FastAPI code from Flask routes using analysis findings, so that I review and refine instead of rewriting from scratch.

**Acceptance Criteria:**
- [ ] Kai accepts Python files with Flask findings from Kantra
- [ ] Generates FastAPI routes preserving business logic
- [ ] Flags unsafe conversions for manual review

**Konveyor component:** Kai
**Priority signal:** Largest investment, highest payoff. Ship K-1 and K-2 first to validate the scenario.

## End-User Migration Story

### Story M-1: Migrate Flask Services to FastAPI

**Persona:** Backend Developer + Platform / SRE + Engineering Manager
**Empathy context:** The whole team needs a single, repeatable process — the developer converts code, the platform/SRE updates infra, the manager tracks progress. Today each of these is ad hoc.

**Story:**
When our team decides to migrate Flask services to FastAPI, I want a single Konveyor-driven workflow that assesses scope, guides the conversion, validates completion, and tracks progress across the portfolio.

**Steps with Konveyor:**
1. **Assess** — Run `kantra analyze` with `flask-to-fastapi` ruleset across all services. Manager gets scope summary with counts and complexity. Developer gets per-file findings.
2. **Migrate** — Developer works through findings route-by-route using pattern equivalence guides. Platform/SRE updates Dockerfile, Helm charts, and CI config using infrastructure findings.
3. **Validate** — Re-run analysis targeting zero findings. Run tests and load benchmarks to confirm functional parity and performance gain.
4. **Repeat** — Move to next service ranked by complexity. Track portfolio completion in Hub.

**Without Konveyor:** Developers grep for Flask patterns manually. Manager gets inconsistent verbal estimates. Infrastructure changes are discovered when pods crash-loop in staging. Migration status lives in a spreadsheet nobody updates.

**Definition of Done:**
- [ ] All services assessed with findings reports generated
- [ ] Each migrated service shows zero findings on re-analysis
- [ ] Tests passing and performance improvement measured per service
- [ ] Portfolio-level progress tracked to completion

## Open Questions

- What is the realistic timeline for adding Python support to Kai?
- Should the `flask-to-fastapi` ruleset be a first-party Konveyor ruleset or a community-contributed one?
- Are there existing Konveyor users running Python workloads who could validate these personas and pain points?
- How does the strangler fig (incremental) migration path interact with Konveyor analysis — can it track partial migrations where Flask and FastAPI coexist?
- Should infrastructure detection rules (Dockerfile, Helm) be part of the `flask-to-fastapi` ruleset or a separate `python-cloud-readiness` ruleset?

## Next Steps

- [ ] Prioritize Konveyor product stories for backlog grooming
- [ ] Validate personas with real users or customer interviews
- [ ] Create detailed plans for high-priority stories
- [ ] Build a proof-of-concept `flask-to-fastapi` custom ruleset to test with a sample Flask app
- [ ] Evaluate Kai Python support as a roadmap item
