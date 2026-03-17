---
date: 2026-03-17
topic: dotnet-framework-to-dotnet8
personas: [Enterprise Architect, Engineering Manager / Director]
konveyor-components: [Kantra, Kai, Rulesets, Hub, Platform Awareness]
migration-type: modernization
---

# Migration Stories: .NET Framework → .NET 8+

## Migration Landscape

### What is changing?

.NET Framework 4.8 is in permanent maintenance mode — security patches only, no new features, ever. The ecosystem is moving on.

Three technologies central to enterprise .NET Framework apps are dead with no forward path:
- **ASP.NET Web Forms** (`System.Web.UI`) — not supported in modern .NET. Requires complete UI rewrite to Blazor or Razor Pages.
- **WCF server-side** (`System.ServiceModel`) — not supported. Must be replaced with gRPC or CoreWCF.
- **ASMX Web Services** — must be replaced with ASP.NET Core Web API.

The entire `System.Web` namespace — the foundation of ASP.NET Framework — is gone. `HttpContext.Current` (static, available anywhere) becomes `IHttpContextAccessor` (dependency-injected). `Global.asax` becomes `Program.cs` middleware pipeline. `Web.config` becomes `appsettings.json`. HTTP Modules become Middleware. This is a paradigm shift, not an upgrade.

### Who is affected?

- **46,000+ companies** currently use Microsoft .NET (6sense, 2026), with 50% in the United States
- **Millions of applications** still run on .NET Framework 4.x in enterprise
- The .NET Development Services market is valued at **$1.41B** (2023), growing to **$2.51B by 2031**
- The broader Legacy Modernization market is **$29.4B in 2026**, growing at 17.6% CAGR
- **62% of U.S. firms still rely on outdated software** in 2026, with maintenance consuming up to 80% of IT budgets
- Enterprises report **$100K–$400K+ per application** for manual .NET Framework refactoring
- Large migration waves commonly allocate **$1–3M** depending on app count and refactor depth

### What is the timeline?

| Date | Event |
|------|-------|
| Nov 2023 | .NET 8 LTS released |
| Nov 2024 | .NET 6 LTS end of support |
| Nov 2025 | .NET 10 LTS released |
| **Jan 12, 2027** | **Windows Server 2016 extended support ends** |
| 2027–2030 | ESU costs escalate 75–125% annually |
| Nov 2028 | .NET 8 LTS end of support |

**The forcing functions:**
1. **Windows Server 2016 EOL (Jan 2027)** — Many .NET Framework apps run on Server 2016. After EOL: no security patches, PCI DSS / HIPAA / SOX audit failures, cyber insurance coverage voided, ESU costs exceed the price of new licenses within 2 years.
2. **.NET 6 already EOL (Nov 2024)** — Teams that migrated partially to .NET 6 now need to move again.
3. **Talent drain** — Senior .NET engineers in 2026 expect .NET 8+, containers, cloud-native. Staying on Framework narrows the hiring pipeline.

### What are the migration paths?

**Path 1: Big-bang rewrite to ASP.NET Core on .NET 8/10**
- Highest payoff (30–50% faster, 40–60% less memory, cross-platform, containers)
- Highest risk — months of effort, breaking changes everywhere
- Best for: smaller apps, greenfield-adjacent rewrites

**Path 2: Incremental migration using System.Web Adapters**
- Microsoft's `dotnet/systemweb-adapters` library bridges ASP.NET Framework and ASP.NET Core
- Run both stacks side-by-side, migrate endpoints incrementally
- Best for: large enterprise apps that can't afford downtime

**Path 3: Lift-and-shift to Azure (buy time)**
- Azure VMs get free Extended Security Updates for Windows Server 2016
- Buys 3 years but doesn't modernize the code — just delays the reckoning
- Best for: apps approaching EOL that don't justify refactoring investment

### What does Konveyor already offer for this?

Konveyor has significant C# migration infrastructure:

**Analyzers:**
- `konveyor/c-sharp-analyzer-provider` — Rust-based gRPC service using tree-sitter and stack-graphs for semantic C# analysis
- `konveyor/analyzer-dotnet-provider` — OmniSharp/Roslyn-based Language Server Protocol implementation

**Existing rulesets** (in `c-sharp-analyzer-provider/rulesets/dotnet-core-migration/`):

| Ruleset File | Coverage | Active Rules |
|---|---|---|
| `01-web-framework-migration.yaml` | System.Web, HttpContext.Current, Global.asax, MVC controllers, routing, Web API, bundling, areas, Web Forms, HttpRuntime | 10 |
| `02-authentication-security-migration.yaml` | WebSecurity, OAuth, Membership, Forms Auth, DotNetOpenAuth, anti-forgery, MachineKey → Data Protection | 8 |
| `03-entity-framework-migration.yaml` | EF6 → EF Core, DbContext, initialization, DbSet API, spatial data, lazy loading | 5 |
| `04-configuration-migration.yaml` | Web.config → appsettings.json, ConfigurationManager, connection strings, session state, GlobalConfiguration, routing, bundling | 10 |
| `05-runtime-utilities-migration.yaml` | Thread.Abort, BinaryFormatter, AppDomain, .NET Remoting, WCF/ServiceModel, Code Access Security | 11 |
| `06-advanced-runtime-migration.yaml` | XSLT scripts, AssemblyBuilder, multi-module assemblies, System.Drawing, WinForms | 6 |
| **Total** | | **50 active rules** |

Additionally:
- `rulesets/hack/dotnet/DOTNET_RULES.csv` — 1.7MB comprehensive database of .NET Framework types and their compatibility status with .NET Core
- Test data: NerdDinner (ASP.NET MVC 4) for validation

## Personas

### Persona 1 — Enterprise Architect

**Background:** Owns the technology strategy and reference architecture across the organization's application portfolio. Evaluates migration paths, sets standards, and presents business cases to leadership. Responsible for 50–200+ .NET applications across multiple teams.

**Technical depth:** Expert with .NET ecosystem, intermediate with cloud-native/container patterns

**Migration authority:** Decision maker — chooses the migration path and prioritizes which apps move first

**What keeps them up at night:** "We have 150 .NET Framework apps. I don't even know which ones use WCF, which ones are WebForms, and which ones are straightforward MVC. How do I prioritize when I can't even assess the scope?"

### Persona 4 — Engineering Manager / Director

**Background:** Manages 2–4 teams of developers. Owns delivery timelines and headcount allocation. Must balance migration work against feature development and business commitments. Reports migration progress to VP/CTO.

**Technical depth:** Intermediate — understands the technical challenges at a high level but doesn't write migration code

**Migration authority:** Decision maker — allocates team capacity, sets sprint priorities, approves migration timelines

**What keeps them up at night:** "Leadership wants new features AND the migration done by Q4. I can't staff both. If I pull developers onto migration, feature delivery slips. If I delay migration, we fail the compliance audit. I need tooling that makes the migration faster so my team can do both."

## Empathy Maps

### Persona 1 — Enterprise Architect — Empathy Map

**Thinks:**
- "We have 150 apps but I can only justify migrating maybe 40 this year. Which 40? I need data, not gut feel."
- "If I pick the wrong migration path for a WebForms app and we're six months in before realizing it needs a full rewrite, that's my credibility gone."
- "Existing tooling handles project files and package references well, but nobody covers the full journey — assessment, code transformation, and validation — in one flow."
- "The compliance team is already asking about our Windows Server 2016 remediation plan. I need to show them a portfolio-level view, not app-by-app spreadsheets."

**Feels:**
- Overwhelmed by the sheer scale — 150 apps, each with unknown dependency depth
- Frustrated that there's no single tool that gives a portfolio-wide migration assessment
- Anxious about making irreversible architectural decisions (gRPC vs CoreWCF, Blazor vs Razor Pages) that lock teams in
- Cautiously optimistic that AI-assisted tooling could compress timelines

**Says:**
- "We need an application-level inventory before we can commit to any timeline."
- "Give me a heat map — which apps are high-effort, which are straightforward MVC lifts?"
- "I'm not signing off on a migration plan until I can see the blast radius for each app."
- "Can we get an automated assessment across the whole portfolio? I can't ask 12 teams to do this manually."

**Does:**
- Builds spreadsheets manually categorizing apps by framework (MVC, WebForms, WCF, Web API)
- Runs proof-of-concept migrations on 1–2 small apps to validate approach
- Researches tooling options
- Lobbies leadership for dedicated migration budget and headcount
- Creates migration tiers: Tier 1 (straightforward MVC), Tier 2 (MVC + WCF), Tier 3 (WebForms — full rewrite)

**Pain Points:**
1. **No portfolio-level visibility** — Can't answer "how big is our migration?" without weeks of manual auditing across teams
2. **Effort estimation is guesswork** — Each app is a black box until someone digs into the codebase. A "simple MVC app" might have 200 `HttpContext.Current` calls buried in utility classes
3. **Decision paralysis on dead-end technologies** — WebForms apps have no incremental path. Do you rewrite to Blazor (new, less mature) or Razor Pages (proven but different paradigm)? Wrong call wastes months
4. **End-to-end tooling gap** — Available tools cover pieces of the migration (project files, NuGet updates, some code patterns) but no single solution spans portfolio assessment → deep code transformation → validation. Teams stitch together multiple tools and still do significant manual work

**Goals:**
1. Deliver a prioritized migration roadmap with defensible effort estimates that leadership will fund
2. Reduce per-app migration cost from $100K–$400K to something that makes the business case self-evident

**Unspoken Needs:**
- Confidence that the assessment data is accurate — they can't afford to present wrong estimates to the CTO
- A way to show progress to leadership without waiting for the first app to fully migrate
- Validation that other enterprises are taking the same approach — "are we doing this right?"

### Persona 4 — Engineering Manager / Director — Empathy Map

**Thinks:**
- "My team has 3 sprints of feature work committed. If I pull people onto migration, I break promises to the business. If I don't, we fail the security audit."
- "The architects say 'migrate to .NET 8' like it's a sprint task. They don't understand that my team has never written ASP.NET Core code. There's a learning curve."
- "If tooling can automate 60–80% of the mechanical work, my senior devs only need to handle the hard 20%. That changes the staffing math completely."
- "I need to show the VP that migration isn't just cost — it's performance gains, reduced infrastructure spend, faster hiring. But I need numbers."

**Feels:**
- Squeezed between competing priorities — leadership wants features AND migration AND no production incidents
- Cautious about tooling promises — wants to see concrete before/after results on their codebase before committing the team
- Protective of the team — doesn't want to burn out developers with a year-long migration death march
- Relieved when tooling actually works — anything that reduces manual effort is a win they can feel

**Says:**
- "What's the developer-hours estimate? I need to plan sprints."
- "Can we do this incrementally? I can't freeze feature development for six months."
- "Show me what the tool actually does — don't tell me it 'automates migration,' show me the before and after."
- "I need a dashboard my VP can look at. How many apps done, how many remaining, what's the projected completion date."

**Does:**
- Negotiates with product managers for "migration sprints" carved out of the roadmap
- Assigns one senior developer to evaluate migration tooling and run a pilot
- Sets up weekly migration standups to track progress across apps
- Escalates blockers (WCF apps, WebForms apps) to the architect when the team gets stuck
- Tracks migration velocity — apps per sprint, lines changed, test coverage — to forecast completion

**Pain Points:**
1. **Capacity allocation is zero-sum** — Every developer-day on migration is a developer-day not on features. There's no slack in the schedule. Tooling that saves 40 hours per app is the difference between "possible" and "impossible"
2. **Team skill gap** — Developers know .NET Framework deeply but ASP.NET Core is a different mental model. The learning curve compounds the migration effort. They need guardrails, not just documentation
3. **No way to measure progress at portfolio level** — "We migrated 3 apps" means nothing to the VP without context. Need: "3 of 40 Tier-1 apps done, 85% of straightforward patterns automated, Tier-2 starts Q3"
4. **Risk of regression** — Every migrated app needs testing. Without automated validation that the migrated code is functionally equivalent, each app is a potential production incident

**Goals:**
1. Complete the migration without sacrificing feature delivery or team morale
2. Demonstrate ROI to leadership — show that migration investment pays back in infrastructure savings, performance, and hiring velocity

**Unspoken Needs:**
- A safety net — confidence that automated transformations won't introduce subtle bugs that surface in production weeks later
- Political cover — data showing this is an industry-wide initiative, not their team being slow
- Something to show in the first two weeks — leadership loses patience with long ramp-ups before visible results

## Konveyor Opportunity Analysis

| Konveyor Capability | Opportunity | Impact |
|---------------------|-------------|--------|
| **Analysis (Kantra)** | Portfolio-wide scanning of .NET Framework apps to classify migration complexity: count `System.Web` dependencies, detect WebForms vs MVC vs Web API vs WCF, flag dead-end patterns. Generate a per-app effort score using the existing 6-category rule model. This is the "heat map" the Enterprise Architect needs. | **High** |
| **AI-Assisted Refactoring (Kai)** | Automate the mechanical code transformations: `System.Web.Mvc.Controller` → `Microsoft.AspNetCore.Mvc.Controller`, `HttpContext.Current` → `IHttpContextAccessor` injection, `Web.config` sections → `appsettings.json`, `Global.asax` event handlers → middleware registration, EF6 patterns → EF Core. Kai's RAG capability can learn from early manual migrations in the org and apply those patterns to remaining apps — the more apps you migrate, the faster each subsequent one goes. | **High** |
| **Rulesets** | 50 active rules exist across 6 categories. Key expansion opportunities: (1) Enable and validate the 6 commented-out rules (COM+, Workflow Foundation, Security Transparency, CAS, Delegate.BeginInvoke, MVC Controller). (2) Add WCF → gRPC transformation rules with concrete replacement patterns, not just detection. (3) Add WebForms → Blazor/Razor Pages rules covering `System.Web.UI` controls → Blazor components. (4) Add post-migration validation rules that verify the target app is clean of Framework dependencies. | **High** |
| **Migration Planning (Hub)** | Portfolio-level dashboard tracking migration across 50–200 apps: which tier each app falls into, migration status per app, aggregate effort remaining, projected completion date. This is the dashboard both personas need — the Enterprise Architect to get funding, the Engineering Manager to keep it. | **High** |
| **Platform Awareness** | Detect deployment model dependencies: IIS bindings in `Web.config`, Windows Service configurations, Windows-specific file paths, registry access, COM interop. Flag apps that can't be containerized without infrastructure changes. Provide guidance on IIS → Kestrel transition and Windows → Linux container readiness. | **Medium** |

### Gaps to close

| Gap | What's needed | Persona served |
|-----|--------------|----------------|
| 6 rules are commented out | Validate and enable advanced runtime rules (COM+, WF, Security Transparency, CAS, Delegate.BeginInvoke, MVC Controller) | Enterprise Architect (complete assessment) |
| WCF → gRPC is detect-only | Add replacement code patterns and Kai training examples for `ServiceContract` → gRPC proto + server implementation | Engineering Manager (automate the hard parts) |
| WebForms → Blazor guidance is minimal | WebForms apps have no incremental path. Even detection + effort estimation + migration option guidance would be valuable | Both |
| No post-migration validation rules | Rules that confirm the migrated app is clean: no `System.Web` references, no .NET Framework-only APIs, no `web.config` dependencies | Engineering Manager (safety net) |
| Hub doesn't have a .NET migration view | Need a portfolio view with tiering, status tracking, and effort aggregation specific to .NET migration | Both |
| Kai hasn't been demonstrated for C# | Kai works for Java and Go — proving it on C# with the existing rulesets would be a showcase | Both |

## Konveyor Product Stories

---

### Story K-1: Portfolio-Wide .NET Framework Assessment

**Persona:** Enterprise Architect
**Empathy context:** They're manually building spreadsheets to categorize 150 apps and can't answer "how big is our migration?" without weeks of auditing across teams.

**Story:**
As an Enterprise Architect, I want to run Kantra across my entire .NET Framework portfolio and get a per-app migration complexity score, so that I can present a prioritized migration roadmap to leadership without weeks of manual auditing.

**Acceptance Criteria:**
- [ ] Kantra scans a directory of .NET Framework solutions and produces a per-app report
- [ ] Each app is classified by type: MVC, Web API, WebForms, WCF, mixed
- [ ] Each app gets an aggregate effort score based on rules fired (using the existing 6-category model)
- [ ] Output includes a portfolio summary: count by tier, total effort estimate, recommended migration order

**Konveyor component:** Kantra + Hub
**Priority signal:** This is the entry point — no enterprise starts migrating without assessing first. Unlocks every subsequent story.

---

### Story K-2: AI-Assisted System.Web → ASP.NET Core Code Transformation

**Persona:** Engineering Manager / Director
**Empathy context:** Every developer-day on migration is a developer-day not on features. Tooling that saves 40 hours per app is the difference between "possible" and "impossible."

**Story:**
As an Engineering Manager, I want Kai to automatically transform common System.Web patterns (HttpContext.Current, Global.asax event handlers, ConfigurationManager, routing) to their ASP.NET Core equivalents, so that my developers focus on the hard 20% instead of the mechanical 80%.

**Acceptance Criteria:**
- [ ] Kai transforms `HttpContext.Current` usages to `IHttpContextAccessor` with proper DI registration
- [ ] Kai transforms `Global.asax` event handlers to `Program.cs` middleware pipeline registrations
- [ ] Kai transforms `ConfigurationManager.AppSettings["key"]` to `IConfiguration["key"]` with DI injection
- [ ] Kai transforms `System.Web.Mvc.Controller` usages to `Microsoft.AspNetCore.Mvc.Controller`
- [ ] Each transformation produces compilable code with before/after diff visible to the developer

**Konveyor component:** Kai
**Priority signal:** These 4 patterns cover the most common transformations in any MVC app. High volume, high automation potential.

---

### Story K-3: WCF Service Migration Detection and Guidance

**Persona:** Enterprise Architect
**Empathy context:** WCF apps are the highest-risk tier. Wrong architectural decision (gRPC vs CoreWCF vs REST) wastes months. They need data to make the call.

**Story:**
As an Enterprise Architect, I want Kantra to detect WCF service contracts, bindings, and behaviors and recommend a migration target (gRPC, CoreWCF, or REST API) based on the service characteristics, so that I can make informed architectural decisions per service rather than guessing.

**Acceptance Criteria:**
- [ ] Detects `[ServiceContract]`, `[OperationContract]`, and binding configurations
- [ ] Classifies services by complexity: simple request/response, duplex, streaming, message queuing
- [ ] Recommends gRPC for high-performance RPC, CoreWCF for complex bindings, REST API for simple CRUD
- [ ] Provides effort estimate per service based on contract complexity

**Konveyor component:** Kantra + Rulesets
**Priority signal:** WCF migration is consistently described as the hardest part of .NET modernization. Decision support here prevents expensive wrong turns.

---

### Story K-4: Migration Progress Dashboard

**Persona:** Engineering Manager / Director
**Empathy context:** "I need a dashboard my VP can look at. How many apps done, how many remaining, what's the projected completion date." Without this, they can't demonstrate ROI or secure continued funding.

**Story:**
As an Engineering Manager, I want a Hub dashboard showing migration status across my app portfolio — apps by tier, completed vs in-progress vs not-started, effort burned vs remaining — so that I can report progress to leadership and forecast completion.

**Acceptance Criteria:**
- [ ] Dashboard shows apps grouped by migration tier (Tier 1: MVC, Tier 2: MVC+WCF, Tier 3: WebForms)
- [ ] Each app shows status: not started, in progress, migrated, validated
- [ ] Aggregate view: total apps, % complete, effort remaining in story points
- [ ] Trend line showing migration velocity (apps completed per sprint/month)

**Konveyor component:** Hub
**Priority signal:** Without visibility, migration projects lose funding. This is how the Engineering Manager keeps the lights on.

---

### Story K-5: Entity Framework 6 → EF Core Transformation

**Persona:** Engineering Manager / Director
**Empathy context:** Team has deep EF6 knowledge but EF Core has different defaults (lazy loading opt-in, client evaluation exceptions). Subtle behavioral differences cause production bugs that erode confidence.

**Story:**
As an Engineering Manager, I want Kai to transform EF6 DbContext, initialization, and query patterns to EF Core equivalents with explicit callouts for behavioral differences, so that my team avoids subtle runtime bugs in migrated data access code.

**Acceptance Criteria:**
- [ ] Transforms `System.Data.Entity.DbContext` to `Microsoft.EntityFrameworkCore.DbContext`
- [ ] Transforms `Database.SetInitializer` to EF Core migration patterns
- [ ] Flags queries that rely on implicit client evaluation (throws in EF Core 3.0+)
- [ ] Adds explicit `UseLazyLoadingProxies()` configuration where lazy loading was previously default
- [ ] Generates warnings for behavioral differences (not just API changes)

**Konveyor component:** Kai + Rulesets
**Priority signal:** Every .NET Framework web app with a database uses EF6. Data access bugs are the hardest to catch and the most damaging in production.

---

### Story K-6: Post-Migration Validation Rules

**Persona:** Engineering Manager / Director
**Empathy context:** They need a safety net — confidence that automated transformations won't introduce subtle bugs. Every migrated app is a potential production incident without validation.

**Story:**
As an Engineering Manager, I want Kantra to run post-migration validation rules that confirm a migrated app has no remaining .NET Framework dependencies, so that my team can deploy with confidence that nothing was missed.

**Acceptance Criteria:**
- [ ] Detects any remaining `System.Web` namespace references in migrated code
- [ ] Detects any remaining `Web.config` dependencies (should be `appsettings.json`)
- [ ] Detects any remaining .NET Framework-only NuGet packages
- [ ] Detects any remaining `Global.asax`, `BundleConfig.cs`, `RouteConfig.cs` files
- [ ] Produces a clean/not-clean report per app

**Konveyor component:** Kantra + Rulesets
**Priority signal:** Validation closes the loop. Without it, "migrated" is a claim, not a fact.

---

### Story K-7: Organizational Learning via Kai RAG

**Persona:** Engineering Manager / Director
**Empathy context:** The more apps the team migrates, the faster each subsequent one should go. But tribal knowledge stays in the heads of whoever did the last migration.

**Story:**
As an Engineering Manager, I want Kai to learn from my team's completed .NET migrations and apply those patterns to subsequent apps, so that migration velocity increases over time instead of staying flat.

**Acceptance Criteria:**
- [ ] Kai indexes completed migration PRs (before/after code pairs) into its RAG knowledge base
- [ ] Subsequent migration suggestions incorporate org-specific patterns (custom base classes, internal libraries, naming conventions)
- [ ] Migration suggestions improve measurably (fewer manual corrections needed) after 3+ completed apps

**Konveyor component:** Kai
**Priority signal:** This is Konveyor's differentiator — no other tool learns from your org's migration history. Turns migration from linear effort into accelerating effort.

---

## End-User Migration Stories

---

### Story M-1: Assess My .NET Framework Portfolio

**Persona:** Enterprise Architect
**Empathy context:** Overwhelmed by scale, needs data before they can commit to any timeline or budget request.

**Story:**
When I'm asked to present a .NET migration roadmap to leadership, I want to scan my entire portfolio and get a complexity report per app, so I can build a defensible business case with real effort estimates.

**Steps with Konveyor:**
1. Point Kantra at the source code repository root containing all .NET solutions
2. Kantra scans each solution, fires rules across all 6 categories, generates per-app effort scores
3. Review the portfolio summary: apps by tier, total effort, recommended migration sequence
4. Export to a format suitable for leadership presentation

**Without Konveyor:** Manually open each solution in Visual Studio, grep for System.Web / WCF / WebForms references, build a spreadsheet by hand. Takes weeks across 150 apps. Error-prone and incomplete.

**Definition of Done:**
- [ ] Portfolio report generated covering all .NET Framework apps in the org
- [ ] Each app has a tier classification and effort estimate
- [ ] Report is presentable to leadership for funding approval

---

### Story M-2: Migrate a Tier 1 MVC Application

**Persona:** Engineering Manager / Director (delegates to team)
**Empathy context:** Needs a quick win to build confidence and show progress. Picks the simplest app first.

**Story:**
When I assign a developer to migrate a straightforward ASP.NET MVC app, I want the mechanical code transformations automated, so the developer completes the migration in days instead of weeks.

**Steps with Konveyor:**
1. Run Kantra on the target app to get a detailed findings report (which rules fire, where)
2. Run Kai to auto-transform System.Web patterns → ASP.NET Core equivalents
3. Developer reviews Kai's changes, handles the remaining manual items
4. Run Kantra post-migration validation to confirm no Framework dependencies remain
5. Developer runs existing test suite against migrated app

**Without Konveyor:** Developer reads Microsoft migration docs, manually finds and replaces every System.Web reference, rewrites Global.asax to Program.cs by hand, updates every configuration access pattern. Each app takes 2–4 weeks of full-time developer effort.

**Definition of Done:**
- [ ] App compiles and runs on .NET 8
- [ ] Post-migration validation shows zero remaining Framework dependencies
- [ ] Existing tests pass

---

### Story M-3: Triage WCF Services Before Migration

**Persona:** Enterprise Architect
**Empathy context:** WCF is the scariest part. They need to know what they're dealing with before committing developers to the hardest migration path.

**Story:**
When I discover WCF services in the portfolio assessment, I want to understand each service's complexity and get a recommended migration target, so I can plan the right approach per service instead of applying one strategy to all.

**Steps with Konveyor:**
1. Kantra identifies all `[ServiceContract]` and `[OperationContract]` declarations
2. Review the service classification: simple request/response vs duplex vs streaming
3. Accept or override the recommended migration target per service
4. Feed the per-service plan into the Hub for tracking

**Without Konveyor:** Senior architect manually reads every WCF service contract, traces bindings through config files, makes judgment calls with incomplete information. High risk of choosing the wrong migration target.

**Definition of Done:**
- [ ] All WCF services catalogued with complexity classification
- [ ] Migration target selected per service (gRPC / CoreWCF / REST)
- [ ] Effort estimates assigned per service

---

### Story M-4: Track Migration Progress Across Teams

**Persona:** Engineering Manager / Director
**Empathy context:** Squeezed between feature delivery and migration deadlines. Needs to show leadership that the investment is paying off.

**Story:**
When multiple teams are migrating apps in parallel, I want a single view of migration status across the portfolio, so I can identify bottlenecks, reallocate developers, and report progress to my VP.

**Steps with Konveyor:**
1. Each team updates app status in Hub as they complete migration phases
2. Hub dashboard shows real-time portfolio status: by tier, by team, by status
3. Review velocity trends to forecast completion date
4. Identify blocked apps (Tier 3 WebForms, complex WCF) for escalation or deprioritization

**Without Konveyor:** Weekly standup across 4 teams, manually aggregated into a slide deck. Status is always 1 week stale. No trend data, no forecasting.

**Definition of Done:**
- [ ] All in-scope apps tracked in Hub with current status
- [ ] Dashboard accessible to leadership without manual report generation
- [ ] Completion forecast updated weekly based on actual velocity

---

### Story M-5: Migrate Authentication from Forms Auth to ASP.NET Core Identity

**Persona:** Engineering Manager / Director (delegates to team)
**Empathy context:** Authentication migration is high-stakes — get it wrong and users can't log in, or worse, security is compromised. Team needs guardrails.

**Story:**
When my team encounters Forms Authentication or ASP.NET Membership in a migrating app, I want guided transformation to ASP.NET Core Identity, so the team doesn't introduce security vulnerabilities during the most sensitive part of the migration.

**Steps with Konveyor:**
1. Kantra flags `FormsAuthentication`, `Membership`, `WebSecurity`, and `MachineKey` usages
2. Kai suggests replacements: `SignInManager`, `UserManager`, `IDataProtector`
3. Developer reviews and validates auth flow end-to-end
4. Post-migration validation confirms no legacy auth references remain

**Without Konveyor:** Developer reads Microsoft identity migration docs, manually traces every auth touchpoint, hopes they didn't miss a `FormsAuthentication.SetAuthCookie` call buried in a utility class.

**Definition of Done:**
- [ ] All auth patterns migrated to ASP.NET Core Identity
- [ ] Login, logout, registration, and password reset flows tested
- [ ] No remaining `System.Web.Security` or `WebMatrix.WebData` references

---

### Story M-6: Validate Migration Completeness Before Go-Live

**Persona:** Engineering Manager / Director
**Empathy context:** The moment before go-live is when anxiety peaks. "Did we miss anything?" is the question that keeps them up at night.

**Story:**
When a migrated app is ready for deployment, I want an automated completeness check that confirms no .NET Framework artifacts remain, so I can approve the release with confidence.

**Steps with Konveyor:**
1. Run Kantra post-migration validation ruleset against the migrated codebase
2. Review any remaining findings — are they false positives or genuine misses?
3. Fix any genuine misses
4. Re-run validation until clean
5. Mark app as "validated" in Hub

**Without Konveyor:** Manual code review hoping to catch leftover System.Web references. Grep for known patterns. Miss the ones you didn't think to grep for. Find out in production.

**Definition of Done:**
- [ ] Kantra validation returns zero findings
- [ ] App deployed to staging and smoke-tested
- [ ] App status updated to "validated" in Hub

---

## Open Questions

- Is the `csharp.referenced` condition in the c-sharp-analyzer-provider fully operational? The 50 hand-crafted rules depend on it.
- Which analyzer provider is the primary path forward — tree-sitter/stack-graphs (`c-sharp-analyzer-provider`) or OmniSharp/Roslyn (`analyzer-dotnet-provider`)?
- Has Kai been tested with C# codebases? What's needed to enable C# support in Kai's RAG pipeline?
- How many real-world .NET Framework apps (beyond NerdDinner) have the existing rules been validated against?
- Should Konveyor target .NET 8 (current LTS) or .NET 10 (latest LTS, Nov 2025) as the migration target?
- What's the relationship between Konveyor's C# rulesets and Azure AppCAT's Konveyor-based rulesets (`Azure/appcat-konveyor-rulesets`)?

## Next Steps

- [ ] Prioritize Konveyor product stories for backlog grooming
- [ ] Validate the 50 existing rules against 2-3 real enterprise .NET Framework codebases
- [ ] Enable and validate the 6 commented-out rules
- [ ] Demonstrate Kai on a C# migration (start with NerdDinner → ASP.NET Core)
- [ ] Validate personas with .NET enterprise customers or Red Hat consulting teams
- [ ] Create detailed implementation plans for K-1 (Portfolio Assessment) and K-2 (System.Web Transformation) as highest-priority stories

## Sources

- [.NET Migration Guide: Framework 4.8 to .NET 10](https://wojciechowski.app/en/articles/dotnet-migration-guide)
- [The 2026 Guide to Enterprise .NET 8 Migration](https://technijian.com/podcast/the-2026-guide-to-enterprise-net-8-migration)
- [What Happened to the .NET Upgrade Assistant?](https://www.gapvelocity.ai/blog/what-happened-to-the-.net-upgrade-assistant)
- [Windows Server 2016 EOL — January 12, 2027](https://hypestkey.com/windows-server-2016-end-of-life/)
- [Windows Server 2016 ESU Costs & Migration Strategies](https://windowsnews.ai/article/windows-server-2016-end-of-life-esu-costs-migration-strategies-for-2025.402954)
- [Migrate from ASP.NET Framework to ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/migration/fx-to-core/?view=aspnetcore-10.0)
- [Incremental ASP.NET to ASP.NET Core Migration](https://devblogs.microsoft.com/dotnet/incremental-asp-net-to-asp-net-core-migration/)
- [Migrate HTTP Modules to ASP.NET Core Middleware](https://learn.microsoft.com/en-us/aspnet/core/migration/fx-to-core/areas/http-modules?view=aspnetcore-9.0)
- [konveyor/c-sharp-analyzer-provider](https://github.com/konveyor/c-sharp-analyzer-provider)
- [konveyor/analyzer-dotnet-provider](https://github.com/konveyor/analyzer-dotnet-provider)
- [konveyor/rulesets](https://github.com/konveyor/rulesets)
- [Legacy Modernization Market Report 2026](https://www.mordorintelligence.com/industry-reports/legacy-modernization-market)
