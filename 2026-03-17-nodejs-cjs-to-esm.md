---
date: 2026-03-17
topic: nodejs-cjs-to-esm
personas: [Full-Stack Developer, Library/Package Maintainer, Frontend Developer]
konveyor-components: [Kantra, Kai, Rulesets, Hub, Platform Awareness]
migration-type: modernization
---

# Migration Stories: Node.js CommonJS to ES Modules

## Migration Landscape

### What is changing?

The JavaScript ecosystem is completing its shift from CommonJS (`require()` / `module.exports`) to ES Modules (`import` / `export`). Node.js 24 — now LTS through April 2028 — makes ESM the default module system. Node.js 22 introduced `require(esm)`, allowing CJS code to load ESM modules, removing the last major interop barrier. Major packages (chalk, got, node-fetch, execa, globby, nanoid) have gone ESM-only, and frameworks (Next.js, Remix, Astro, Nuxt, SvelteKit) are ESM-first. 2026 is widely called "the year of full ESM adoption."

### Who is affected?

Every Node.js developer. The npm registry has 2.5M+ packages. As of late 2024, ~25.8% of npm packages shipped ESM (up from 7.8% in 2021) — meaning ~74% of packages still need migration. Enterprise backends at Netflix, PayPal, LinkedIn, Uber, and thousands of mid-market companies run large CJS codebases. Library maintainers face dual-publish complexity or pressure to go ESM-only.

### What is the timeline?

| Date | Event |
|------|-------|
| Sep 2025 | Node.js 18 EOL |
| Apr 2026 | Node.js 20 EOL |
| Oct 2025 | Node.js 24 released |
| Jan 2026 | Node.js 24 enters LTS |
| Apr 2028 | Node.js 24 LTS ends |

There is no single hard deadline (unlike .NET Framework EOL). Instead, it's an ecosystem tide: ESM-only dependencies force upstream consumers to migrate. The pressure is bottom-up (popular packages going ESM-only) and top-down (frameworks requiring ESM).

### What are the migration paths?

1. **Full ESM conversion** — Set `"type": "module"` in package.json, convert all `require()` to `import`, `module.exports` to `export`, add file extensions, replace `__dirname`/`__filename` with `import.meta.dirname`/`import.meta.filename`. Clean but high-effort for large codebases.

2. **Dual-package publishing** — Use conditional exports in package.json to ship both CJS and ESM builds. Common for library authors who need backward compatibility. Adds build complexity (two output formats, potential dual-package hazard).

3. **Gradual migration with interop** — Leverage Node.js 22+'s `require(esm)` to incrementally adopt ESM modules from CJS code. Lower risk, but leaves the codebase in a hybrid state.

### What does Konveyor already offer for this?

Konveyor has strong existing Node.js/TypeScript infrastructure:

- **Analyzer (Kantra)**: Node.js provider using TypeScript Language Server, supports `nodejs.referenced` condition for detecting symbol references across `.ts`, `.tsx`, `.js`, `.jsx` files. Already filters `node_modules`.
- **Preview rulesets**: Existing rules for PatternFly v5-v6, Angular migrations, Astro v4-v5 — proving the Node.js rule authoring pipeline works.
- **Rule generator**: Auto-generates Node.js/TypeScript rules using LLMs (OpenAI, Claude, Gemini). Supports combo rules (`nodejs.referenced` + `builtin.filecontent`) to reduce false positives.
- **Kai**: Supports TypeScript for AI-assisted refactoring.
- **VS Code extension**: JavaScript/TypeScript language support already integrated.

**No CJS-to-ESM rulesets exist yet.** This is a greenfield opportunity built on proven infrastructure.

## Personas

### Persona 2 — Full-Stack Developer

**Background:** IC developer working across the stack — React frontend (already ESM via bundler) and Node.js backend services (still CJS). Does the actual file-by-file migration work. Writes and maintains tests.

**Technical depth:** Intermediate — comfortable with both `require()` and `import`, but hasn't dealt with edge cases like dynamic requires, conditional imports, or `require.resolve()` in ESM.

**Migration authority:** Implementer — executes the migration plan, files PRs, deals with breakage.

**What keeps them up at night:** Tedious, repetitive work. Converting hundreds of files by hand, fixing each broken test one at a time, and discovering edge cases that the "simple find-and-replace" didn't handle.

### Persona 3 — Library / Package Maintainer

**Background:** Maintains 1-5 popular open-source npm packages (1K-100K weekly downloads). Has a day job and maintains packages in spare time. Gets GitHub issues demanding ESM support weekly.

**Technical depth:** Expert with JavaScript modules, but overwhelmed by the dual-publish complexity — conditional exports, build toolchains, testing both CJS and ESM outputs.

**Migration authority:** Decision maker for their packages, but constrained by time and the need to not break downstream consumers.

**What keeps them up at night:** Breaking changes for users. If they go ESM-only, they'll get a flood of issues from CJS consumers. If they dual-publish, the build/test matrix doubles. Either way, it's unpaid work.

### Persona 5 — Frontend Developer

**Background:** Builds React/Vue/Svelte applications using Vite or webpack. Writes `import`/`export` daily, but the bundler has been silently papering over CJS/ESM differences. Maintains shared packages consumed by both frontend and backend teams. Runs into module issues when using SSR (Next.js, Nuxt) or when a dependency goes ESM-only and breaks the build.

**Technical depth:** Intermediate — fluent with `import`/`export` syntax but has a false sense of security. Doesn't understand the difference between "ESM syntax transpiled to CJS by a bundler" and "actual native ESM." Gets blindsided by issues in Node.js contexts (SSR, scripts, CLI tools, Jest).

**Migration authority:** Implementer — owns shared UI packages and build configuration. Often the first person to hit ESM breakage when upgrading a dependency.

**What keeps them up at night:** `jest` and `node_modules` hell. Their test suite uses Jest (which historically ran in CJS mode), their shared component library is consumed by both CJS and ESM consumers, and every time they upgrade a dependency they play whack-a-mole with `ERR_REQUIRE_ESM` or `SyntaxError: Cannot use import statement outside a module`.

## Empathy Maps

### Persona 2 — Full-Stack Developer — Empathy Map

**Thinks:**
- "This is going to be 400 files of mechanical changes. There has to be a tool for this."
- "The simple cases are easy — but what about the 30 files with dynamic `require()` and conditional imports?"
- "If I miss one file, the whole service crashes at runtime, not at build time. CJS won't give me a compile error."
- "Am I supposed to add `.js` extensions to every import? Even for TypeScript files? This feels wrong."

**Feels:**
- Resigned — knows it needs to happen, dreads the tedium
- Anxious — ESM errors are runtime errors, not caught by the compiler
- Frustrated — existing tools (cjstoesm) handle 80% of cases but choke on the hard 20%
- Alone — the tech lead made the decision, but the developer does the work

**Says:**
- "I converted 50 files yesterday and the tests still don't pass."
- "Can we do this incrementally or does it have to be a big bang?"
- "The `__dirname` replacement is fine, but what do I do with `require.resolve()`?"
- "I just spent 2 hours figuring out that one dependency doesn't have a proper ESM export."

**Does:**
- Starts with leaf modules (utils, helpers) and works inward toward entry points
- Runs the test suite after every batch of changes — red/green/red/green cycle
- Googles error messages constantly: `ERR_REQUIRE_ESM`, `ERR_UNKNOWN_FILE_EXTENSION`, `ReferenceError: __dirname is not defined`
- Writes custom regex find-and-replace scripts that handle 70% of conversions, then manually fixes the rest
- Keeps a spreadsheet of files migrated vs. remaining

**Pain Points:**
1. **Volume**: Hundreds of files need identical-but-not-quite-identical changes. Too many for manual editing, too varied for simple find-and-replace.
2. **Async infection (hardest technical challenge)**: Dynamic `require()` converted to `await import()` forces every caller up the chain to become async. A single utility function conversion can cascade through 10+ callers. No automated tool handles this.
3. **Test breakage**: Jest configuration needs to change (or switch to Vitest). Mocking patterns that relied on CJS module caching break under ESM. `jest.mock()` works differently with ESM.
4. **No compiler safety net**: A missed `require()` in a rarely-hit code path won't surface until production. ESM conversion errors are runtime, not build-time.
5. **Dependency compatibility**: Some `node_modules` don't export ESM properly. `require()` of a CJS dependency from ESM works, but the reverse only works on Node 22+.

**Goals:**
1. Complete the migration without introducing regressions — every test passes, every endpoint works
2. Finish it in days, not weeks — minimize the tedium-to-progress ratio

**Unspoken Needs:**
- A tool that handles the boring 80% so they can focus their expertise on the hard 20%
- Confidence that nothing was missed — a way to verify "all CJS patterns have been converted"
- Permission to push back on the timeline if edge cases pile up

### Persona 3 — Library / Package Maintainer — Empathy Map

**Thinks:**
- "If I go ESM-only, I'll break half my users. If I dual-publish, I double my build complexity for zero new features."
- "I maintain this for free. The migration is pure cost with no user-visible benefit."
- "Node 22's `require(esm)` should mean I can just go ESM-only... but what about users still on Node 18?"
- "I've been putting this off for two years. The GitHub issues are getting aggressive."

**Feels:**
- Resentful — the ecosystem is forcing unpaid work on volunteers
- Paralyzed by choice — ESM-only vs. dual-publish vs. status quo, each has painful trade-offs
- Guilty — knows they're blocking downstream users who need ESM
- Exhausted — this is maintenance burden, not creative work

**Says:**
- "I'll accept a PR if someone else does the work."
- "Can someone explain conditional exports to me like I'm five?"
- "Which build tool do I even use? tsup? unbuild? rollup? esbuild? They all do it slightly differently."
- "I just want to publish a package, not become a module system expert."

**Does:**
- Reads blog posts and examples for hours trying to understand the "right" way to set up `exports` in package.json
- Looks at how popular packages (e.g., zod, commander) structured their dual exports and tries to copy it
- Tests manually by creating a CJS consumer and an ESM consumer project, running both
- Delays the migration until a particularly loud GitHub issue or a broken downstream user forces their hand

**Pain Points:**
1. **Conditional exports complexity**: The `package.json` `"exports"` field with `"import"` / `"require"` conditions, type declarations, subpath exports — it's a mini-language with footguns (dual-package hazard).
2. **Build toolchain explosion**: Choosing and configuring a build tool to output both CJS and ESM is harder than it should be. Every tool has different defaults and edge cases.
3. **Testing both outputs**: Need to verify the package works when `require()`'d and when `import`'d. No standard way to do this — most maintainers hack together manual smoke tests.
4. **TypeScript declaration files**: `.d.ts` vs `.d.mts` vs `.d.cts`. Getting type declarations to resolve correctly for both CJS and ESM consumers is a nightmare.
5. **Unpaid labor**: Every hour spent on module migration is an hour not spent on features or bug fixes. Users demand ESM but don't contribute PRs.

**Goals:**
1. Ship a working ESM version with minimal breakage for existing users
2. Spend as few hours as possible — this is volunteer time

**Unspoken Needs:**
- A recipe, not a reference manual — "do exactly these 7 steps for a package like yours"
- Automated validation that both CJS and ESM consumers can use the published package correctly
- Social proof that their chosen approach won't blow up ("show me 10 packages my size that did it this way")

### Persona 5 — Frontend Developer — Empathy Map

**Thinks:**
- "I already write `import`/`export` everywhere. Why is this migration even my problem?"
- "Oh. My shared component library is consumed by the backend team's SSR service, and it's all CJS there."
- "Jest just broke because `chalk` v5 is ESM-only and Jest runs in CJS mode by default. Great."
- "I thought Vite handled all this? Why is my build failing?"

**Feels:**
- Confused — they thought they were already using ESM, but they were using ESM-syntax-transpiled-to-CJS
- Annoyed — module errors feel like plumbing problems, not real engineering work
- Defensive — "this is a backend/Node.js problem, not a frontend problem"
- Overwhelmed when SSR enters the picture — suddenly Node.js module resolution matters

**Says:**
- "It works in the browser, why does it break in SSR?"
- "Jest is the problem, not my code. Can we just switch to Vitest?"
- "I updated one dependency and now 40 tests are red."
- "Why do I need to add `.js` extensions to my `.ts` imports? That makes no sense."

**Does:**
- Upgrades a dependency, hits `ERR_REQUIRE_ESM`, and spends half a day figuring out the fix
- Pins dependency versions to avoid ESM-only breakage, creating tech debt
- Configures Jest `transformIgnorePatterns` to transpile ESM-only dependencies back to CJS (a bandaid)
- Advocates loudly for switching from Jest to Vitest, which handles ESM natively
- Maintains a shared UI component library with an awkward CJS build step just for the backend team

**Pain Points:**
1. **Jest + ESM incompatibility (validated March 2026)**: Jest 30's ESM support is still experimental, still requires `--experimental-vm-modules`, and mocking uses `jest.unstable_mockModule`. The underlying blocker is Node.js's `vm` module ESM APIs remaining unstable.
2. **SSR module resolution**: Frontend code that works fine in the browser (bundled by Vite) breaks when run on the server because Node.js resolves modules differently.
3. **Shared package publishing**: Their component library or utility package is consumed by both frontend (ESM via bundler) and backend (CJS in Node.js). They have to deal with dual-package concerns they didn't sign up for.
4. **Transitive dependency breakage**: They don't control when `chalk` or `execa` goes ESM-only, but they bear the cost when it breaks their pipeline.
5. **False confidence**: Years of writing `import`/`export` syntax created a blind spot — they don't understand native ESM semantics (mandatory extensions, async loading, no `__dirname`) because the bundler hid it.

**Goals:**
1. Tests pass, builds ship, dependencies upgrade without half-day debugging sessions
2. Stop being the person who gets paged when the SSR build breaks because of a module issue

**Unspoken Needs:**
- A clear explanation of *why* their bundled ESM isn't the same as native ESM — the mental model gap
- A migration path for Jest to Vitest that doesn't require rewriting every mock
- Someone to handle the shared package's dual-publish problem so they can go back to building UI

## Validated Pain Points

Each concern was checked against real-world evidence (March 2026):

| Concern | Verdict | Evidence |
|---------|---------|----------|
| Jest + ESM is broken | **Confirmed** | Jest 30 improved ESM but it's still experimental, requires `--experimental-vm-modules`, uses `jest.unstable_mockModule`. Vitest is the recommended default for new projects. |
| Dynamic require async cascade | **Confirmed, genuinely hard** | "Async infection" is a real community term. `require(path)` to `await import(path)` forces every caller async. No tool handles this. |
| `__dirname` replacement | **Simple on Node 24** | `import.meta.dirname` is a drop-in replacement on Node 21.2+. No polyfill needed. `require.resolve()` is the harder case (needs `createRequire()`). |
| Dual-publish complexity | **Confirmed** | `exports` field described as "wild and evolving" with "painful errors." TypeScript can't emit `.d.cts` from `.ts` source. Tools help but don't eliminate complexity. |
| No compiler safety net | **Confirmed** | Missed `require()` in cold code paths crash at runtime. TypeScript `"module": "nodenext"` catches some, but pure JS gets no static checking. |
| `require(esm)` interop | **Mostly resolved** | Stabilized in Node 22.12.0. Node.js 25.8.1 (March 11, 2026) still patching edge cases. 95% solved, not 100%. |
| Tooling gap | **Confirmed** | cjstoesm handles basics. No tool covers dynamic requires + async infection + test migration + config changes + dependency compatibility together. |

## Konveyor Opportunity Analysis

| Konveyor Capability | Opportunity | Impact |
|---------------------|-------------|--------|
| **Analysis (Kantra)** | Detect every CJS pattern: `require()`, `module.exports`, `__dirname`, `__filename`, `require.resolve()`, dynamic `require(variable)`, conditional requires. Scan `package.json` for missing `"type": "module"` and `"exports"`. Kantra's sweet spot — highly pattern-based, every instance is detectable. | **High** |
| **AI-Assisted Refactoring (Kai)** | Handle the hard 20%: dynamic requires needing async refactoring, `require.resolve()` to `createRequire()` conversions, conditional require patterns. Kai's RAG can learn from early manual conversions and apply patterns to remaining files. | **High** |
| **Rulesets** | New CJS-to-ESM ruleset (~15-20 rules): import/export syntax, Node.js globals, dynamic patterns, package.json config, test framework patterns. Community-reusable across every Node.js project — unlike framework-specific rulesets. | **High** |
| **Migration Planning (Hub)** | Track migration across a portfolio of services. Enterprises with 50+ Node.js microservices need visibility: which services are ESM, hybrid, or not started. Surface migration ordering (shared packages before consumers). | **Medium** |
| **Platform Awareness** | Detect Node.js runtime versions. Flag services on Node 20 (EOL April 2026). Surface CJS-only dependency blockers. | **Medium** |

### Gaps Requiring New Capability

| Gap | Description | Severity |
|-----|-------------|----------|
| **Dependency compatibility analysis** | Scan `node_modules` and report ESM/CJS status of each dependency. Critical for migration planning. | **High** |
| **Cross-file async infection tracing** | Trace the call chain from a dynamic `require()` to show how many callers need async conversion. Requires call-graph analysis via TypeScript Language Server. | **High** |
| **Build/config migration** | `tsconfig.json`, `package.json`, and build tool config changes are not source code. Kantra can detect patterns; Kai needs config-file refactoring support. | **Medium** |
| **Migration ordering** | For monorepos/microservices, shared packages must migrate before consumers. Hub needs internal dependency graph awareness. | **Medium** |
| **Post-migration validation** | "CJS-free" verification scan plus CI integration to prevent regression. | **Low** |

### Strategic Opportunity

This migration is unique in Konveyor's portfolio: **universal applicability**. Every other Node.js ruleset (PatternFly, Angular, Astro) serves a specific framework's users. A CJS-to-ESM ruleset serves every Node.js developer. It's the Node.js equivalent of Java EE to Quarkus — foundational, high-volume, and the kind of thing that puts Konveyor on the map for the JavaScript ecosystem. The existing Node.js infrastructure (analyzer, rule generator, Kai TypeScript support, VS Code extension) means this is a "write the rules and validate" effort, not "build from scratch."

## Konveyor Product Stories

---

### Story K-1: CJS-to-ESM Detection Ruleset for Kantra

**Persona:** Full-Stack Developer
**Empathy context:** They face hundreds of files needing conversion and no way to know the full scope before starting. The first thing they need is a clear inventory — not a guess.

**Story:**
As a Full-Stack Developer, I want Kantra to scan my Node.js codebase and report every CJS pattern categorized by type and conversion difficulty, so that I can estimate effort and plan the migration in batches.

**Acceptance Criteria:**
- [ ] Rules detect `require()` calls (static and dynamic separately)
- [ ] Rules detect `module.exports` and `exports.x` assignments
- [ ] Rules detect `__dirname`, `__filename`, `require.resolve()` usage
- [ ] Rules detect missing `"type": "module"` in `package.json`
- [ ] Each finding is categorized: "mechanical" (auto-convertible) vs. "needs review" (dynamic require, conditional import)
- [ ] Output includes file count and finding count summary

**Konveyor component:** Rulesets + Kantra
**Priority signal:** Foundation for everything else. Every subsequent story depends on detection working first. Applies to every Node.js project — maximum reuse.

---

### Story K-2: AI-Assisted Mechanical CJS-to-ESM Conversion via Kai

**Persona:** Full-Stack Developer
**Empathy context:** They dread the tedium of converting 300+ files by hand. The mechanical 80% — `require()` to `import`, `module.exports` to `export` — is repetitive and error-prone at scale. They need the boring work done for them so they can focus on the hard cases.

**Story:**
As a Full-Stack Developer, I want Kai to automatically convert mechanical CJS patterns to ESM equivalents, so that I only spend my time on the edge cases that need human judgment.

**Acceptance Criteria:**
- [ ] Converts `const x = require('y')` to `import x from 'y'`
- [ ] Converts `const { a, b } = require('y')` to `import { a, b } from 'y'`
- [ ] Converts `module.exports = x` to `export default x`
- [ ] Converts `module.exports.foo = x` to `export const foo = x`
- [ ] Converts `__dirname` to `import.meta.dirname` (Node 21.2+)
- [ ] Converts `__filename` to `import.meta.filename`
- [ ] Adds file extensions to relative import paths where missing
- [ ] Preserves comments, formatting, and surrounding code
- [ ] Skips dynamic `require()` and flags them for manual review

**Konveyor component:** Kai
**Priority signal:** Directly addresses the highest-volume pain point. A 500-file codebase could save 2-3 days of developer time per migration.

---

### Story K-3: Dynamic Require and Async Infection Detection

**Persona:** Full-Stack Developer
**Empathy context:** The mechanical changes are manageable. What keeps them up at night is the dynamic `require(variable)` buried in a utility function that, once converted to `await import()`, forces 7 callers to go async. They need to see the blast radius before they touch anything.

**Story:**
As a Full-Stack Developer, I want Kantra to identify dynamic `require()` calls and trace their callers, so that I can assess the async infection risk before converting them.

**Acceptance Criteria:**
- [ ] Detects `require(variable)`, `require(expression)`, and `require()` inside conditionals
- [ ] For each dynamic require, lists the function it's in and that function's callers (using TypeScript Language Server "find references")
- [ ] Flags the depth of the call chain that would need `async` conversion
- [ ] Provides a severity rating: "contained" (1-2 callers) vs. "cascading" (3+ levels)
- [ ] Suggests alternative patterns where applicable (e.g., top-level static import with lazy usage)

**Konveyor component:** Kantra (extended analysis) + Rulesets
**Priority signal:** The #1 technical blocker for large-scale migrations. No existing tool does this. Clear differentiation for Konveyor.

---

### Story K-4: Jest CJS Pattern Detection and Vitest Migration Guidance

**Persona:** Frontend Developer
**Empathy context:** Their tests break every time a dependency goes ESM-only. Jest's ESM support is still experimental in 2026. They're stuck choosing between `--experimental-vm-modules` hacks and a full Vitest migration — and they need data to make the case to their team.

**Story:**
As a Frontend Developer, I want Kantra to detect Jest-specific CJS patterns in my test suite and provide Vitest migration guidance, so that I can plan my test framework migration alongside the module system migration.

**Acceptance Criteria:**
- [ ] Detects `jest.mock()` calls that need `jest.unstable_mockModule` in ESM
- [ ] Detects `require()` calls in test files, setup files, and config files
- [ ] Detects Jest config files (`jest.config.js`, `jest.config.ts`) and flags CJS-specific settings
- [ ] Provides side-by-side mapping: Jest pattern to Vitest equivalent
- [ ] Reports count of test files affected and estimated conversion scope

**Konveyor component:** Rulesets + Kai
**Priority signal:** Jest is the most popular JS test framework. This pain point is validated and current. Addresses a real blocker that no migration tool covers.

---

### Story K-5: Dependency ESM Compatibility Scanner

**Persona:** Full-Stack Developer, Frontend Developer
**Empathy context:** They can't migrate to ESM if half their dependencies don't support it. Before starting, they need to know: "which of my 150 dependencies are CJS-only, which are ESM-only, and which are dual?" Without this, they're flying blind.

**Story:**
As a Full-Stack Developer, I want Kantra to analyze my `node_modules` and report the ESM/CJS compatibility status of each dependency, so that I can identify blockers before I start migrating my own code.

**Acceptance Criteria:**
- [ ] Reads each dependency's `package.json` for `"type"`, `"exports"`, `"module"`, and `"main"` fields
- [ ] Categorizes each dependency: ESM-only, CJS-only, dual-package, or unknown
- [ ] Flags dependencies that will break under `"type": "module"` (CJS-only packages imported without `createRequire`)
- [ ] Highlights transitive dependencies that are CJS-only
- [ ] Outputs a compatibility report sorted by risk level

**Konveyor component:** Kantra (new capability) + Platform Awareness
**Priority signal:** Addresses the "dependency compatibility" gap. Prerequisite information for any migration.

---

### Story K-6: Package.json Exports Validator for Library Authors

**Persona:** Library / Package Maintainer
**Empathy context:** They've read five blog posts about conditional exports and they're still not sure they got it right. They shipped a package where types broke for half their users because `"types"` wasn't first in the condition. They need validation, not documentation.

**Story:**
As a Library Maintainer, I want Kantra to validate my `package.json` `"exports"` configuration against known best practices and common mistakes, so that I can publish with confidence that both CJS and ESM consumers will work.

**Acceptance Criteria:**
- [ ] Validates `"types"` condition comes before `"import"` and `"require"` in each export entry
- [ ] Checks that referenced files (`.mjs`, `.cjs`, `.d.ts`, `.d.mts`, `.d.cts`) actually exist
- [ ] Warns about dual-package hazard (stateful modules exported in both formats)
- [ ] Validates subpath exports resolve correctly
- [ ] Checks for common mistakes: missing `"default"` condition, wrong file extensions, `"main"` conflicting with `"exports"`

**Konveyor component:** Rulesets + Kantra
**Priority signal:** Library maintainers are the bottleneck for ecosystem-wide ESM adoption. Unblocking them accelerates everything.

---

### Story K-7: Enterprise Migration Dashboard in Hub

**Persona:** Full-Stack Developer (as team proxy), indirectly all personas
**Empathy context:** In an enterprise with 30+ Node.js services, nobody has visibility into overall migration progress. The platform team asks "how far along are we?" and nobody can answer without manually checking each repo.

**Story:**
As a migration lead, I want Konveyor Hub to track CJS-to-ESM migration progress across all services in my organization, so that I can report status, identify stragglers, and coordinate migration ordering.

**Acceptance Criteria:**
- [ ] Each application shows a migration status: Not Started / In Progress / Hybrid / Complete
- [ ] Status is derived from Kantra scan results (ratio of CJS patterns remaining)
- [ ] Dependency graph view shows migration ordering (shared libraries must migrate before consumers)
- [ ] Dashboard shows aggregate progress: "18/32 services fully ESM"
- [ ] Can filter by team, Node.js version, and migration status

**Konveyor component:** Hub
**Priority signal:** Enterprise revenue opportunity. Organizations with large Node.js footprints need portfolio-level visibility.

---

## End-User Migration Stories

---

### Story M-1: Assess Migration Scope

**Persona:** Full-Stack Developer
**Empathy context:** They've been told "migrate to ESM" but have no idea how big the job is. They need numbers before they can negotiate timeline with their tech lead.

**Story:**
When I'm asked to estimate the ESM migration effort, I want to run a single scan that quantifies every CJS pattern in my codebase, so I can give my team lead a realistic timeline.

**Steps with Konveyor:**
1. Run `kantra analyze` with the CJS-to-ESM ruleset against the project
2. Review the categorized report: X mechanical conversions, Y dynamic requires, Z test file changes
3. Use the severity breakdown to estimate: mechanical = 1 min/file, dynamic = 30 min each, test framework = separate workstream
4. Present the report to the tech lead with effort estimates grounded in data

**Without Konveyor:** Manually grep for `require(` across the codebase, eyeball the results, miss patterns like `module.exports` deep in files, and guess at effort. Dynamic requires and test framework issues surface mid-migration as surprises.

**Definition of Done:**
- [ ] Scan completes and produces a categorized report with file counts
- [ ] Developer can articulate scope in a planning meeting using the report

---

### Story M-2: Check Dependency Compatibility Before Starting

**Persona:** Full-Stack Developer
**Empathy context:** Last time they upgraded a dependency, they spent half a day on `ERR_REQUIRE_ESM`. They won't start the migration until they know their dependency tree supports it.

**Story:**
Before I begin converting my code, I want to know which of my dependencies are ESM-compatible, so I can resolve blockers first and avoid mid-migration surprises.

**Steps with Konveyor:**
1. Run the dependency compatibility scan against `package.json` and `node_modules`
2. Review the report: green (ESM-ready), yellow (dual-package), red (CJS-only, blocks ESM migration)
3. For red dependencies: find alternatives, pin to dual-package versions, or plan `createRequire()` wrappers
4. Resolve all red blockers before touching application code

**Without Konveyor:** Manually check each dependency's `package.json`, read their changelogs, or discover incompatibilities at runtime when `import` fails. Often done reactively — you hit the error, then investigate.

**Definition of Done:**
- [ ] All direct dependencies categorized by ESM compatibility
- [ ] Blocking dependencies identified with mitigation plan documented

---

### Story M-3: Batch-Convert Mechanical Patterns

**Persona:** Full-Stack Developer
**Empathy context:** They're staring at 300 files that need the same basic transformation. They know it's automatable but the existing tools choke on edge cases. They need something that handles the simple stuff reliably and flags the rest.

**Story:**
When I begin the conversion phase, I want to auto-convert all mechanical CJS patterns to ESM in a single pass, so I can focus my expertise on the cases that actually need human judgment.

**Steps with Konveyor:**
1. Run Kai with the CJS-to-ESM transformation on the codebase
2. Kai converts all mechanical patterns: `require` to `import`, `module.exports` to `export`, `__dirname` to `import.meta.dirname`, adds file extensions
3. Review Kai's output — diffs for each file, with skipped patterns flagged
4. Run the test suite to verify no regressions in the mechanical conversions
5. Commit the batch conversion, then tackle flagged items individually

**Without Konveyor:** Write custom regex scripts that handle 70% of cases. Manually fix the remaining 30%. Alternate between converting and running tests for hours. Keep a spreadsheet to track progress.

**Definition of Done:**
- [ ] All mechanical `require()`/`module.exports` patterns converted
- [ ] All `__dirname`/`__filename` references updated
- [ ] File extensions added to relative imports
- [ ] Test suite passes after conversion
- [ ] Remaining non-mechanical patterns listed for manual review

---

### Story M-4: Resolve Dynamic Require Patterns

**Persona:** Full-Stack Developer
**Empathy context:** Kai flagged 3 dynamic `require()` calls. One of them is in a plugin loader used by 12 route handlers. Converting it to `await import()` means making the plugin loader async, which means making all 12 route handlers async, which means touching the Express middleware chain. They need to see the blast radius.

**Story:**
When I encounter a dynamic `require()` flagged for manual review, I want to see its full call chain and async infection radius, so I can choose the right refactoring strategy.

**Steps with Konveyor:**
1. Review Kantra's dynamic require report showing the call chain depth
2. For "contained" cases (1-2 callers): convert to `await import()` and update callers
3. For "cascading" cases: consider alternative patterns — top-level static import with lazy usage, or `createRequire()` as an escape hatch
4. Use Kai to suggest the refactoring pattern based on the specific code context
5. Manually review and test each conversion

**Without Konveyor:** Discover the async cascade mid-refactoring. Convert a function to async, run tests, find 4 new failures, trace callers manually, convert those to async, run tests again, find more failures. A 10-minute change becomes a 2-hour yak shave.

**Definition of Done:**
- [ ] All dynamic `require()` calls resolved — converted to `await import()`, `createRequire()`, or static import
- [ ] No remaining `require()` calls in the codebase (verified by Kantra re-scan)

---

### Story M-5: Migrate Test Framework from Jest to Vitest

**Persona:** Frontend Developer
**Empathy context:** Their Jest suite has 400 tests. 30 of them use `jest.mock()` which behaves differently in ESM. They can't keep using `--experimental-vm-modules` forever. They need a structured migration, not a weekend hack.

**Story:**
When my test suite breaks due to ESM incompatibility with Jest, I want a structured migration path to Vitest with clear pattern mappings, so I can migrate without rewriting every test from scratch.

**Steps with Konveyor:**
1. Run Kantra scan against test files to identify Jest-specific CJS patterns
2. Review the mapping: `jest.mock()` to `vi.mock()`, `jest.fn()` to `vi.fn()`, `jest.spyOn()` to `vi.spyOn()`, `jest.config.js` to `vitest.config.ts`
3. Use Kai to auto-convert the mechanical test patterns (most are 1:1 replacements)
4. Manually review the 30 `jest.mock()` calls that used CJS module caching behavior — these may need restructuring under ESM
5. Replace Jest config with Vitest config, remove `--experimental-vm-modules` workaround
6. Run full test suite under Vitest

**Without Konveyor:** Read Vitest migration docs, manually find-and-replace `jest.` to `vi.`, discover that `jest.mock()` hoisting works differently in Vitest, spend a day debugging mock ordering issues. Miss `jest.mock()` calls in shared test utilities.

**Definition of Done:**
- [ ] All test files converted from Jest API to Vitest API
- [ ] Vitest config replaces Jest config
- [ ] All tests pass under Vitest
- [ ] No remaining `jest` imports in test files

---

### Story M-6: Validate Package Exports for Library Publishing

**Persona:** Library / Package Maintainer
**Empathy context:** They're about to publish a breaking change. Their last ESM attempt broke types for half their users. They need a validation step before they hit `npm publish` — not after the bug reports arrive.

**Story:**
Before I publish my library's ESM version, I want to validate that both CJS and ESM consumers can import it correctly with proper type resolution, so I don't break my downstream users.

**Steps with Konveyor:**
1. Run Kantra's package.json exports validator against the package
2. Fix any flagged issues: `"types"` ordering, missing files, dual-package hazard warnings
3. Kantra verifies that the referenced `.mjs`, `.cjs`, `.d.ts`, `.d.mts` files exist and resolve correctly
4. Manually smoke-test with a CJS consumer (`require('my-package')`) and an ESM consumer (`import from 'my-package'`)
5. Publish with confidence

**Without Konveyor:** Copy the `exports` config from a popular package, hope it works, publish, wait for GitHub issues. Discover 2 days later that TypeScript types don't resolve for CJS consumers because `"types"` was in the wrong position.

**Definition of Done:**
- [ ] `package.json` `"exports"` passes all validation rules
- [ ] Package is importable via both `require()` and `import`
- [ ] TypeScript types resolve correctly for both CJS and ESM consumers

---

### Story M-7: Verify Migration Completeness

**Persona:** Full-Stack Developer
**Empathy context:** They've been converting files for 3 days. They think they're done, but a missed `require()` in an error handler that only fires under load will crash production at 2am. They need certainty, not hope.

**Story:**
After completing my migration, I want to verify that zero CJS patterns remain in my codebase, so I can deploy with confidence that no runtime `ReferenceError` lurks in a cold code path.

**Steps with Konveyor:**
1. Re-run Kantra with the CJS-to-ESM ruleset against the fully migrated codebase
2. Expect zero findings — if any remain, they're files that were missed
3. Verify `package.json` has `"type": "module"`
4. Verify no `.js` files contain `require()` or `module.exports`
5. Add the Kantra ruleset to CI to prevent CJS regression

**Without Konveyor:** Grep for `require(` and hope the regex catches everything. Miss patterns like `exports.foo =` or `require.resolve()`. Deploy and discover the miss at runtime.

**Definition of Done:**
- [ ] Kantra scan reports zero CJS findings
- [ ] `package.json` contains `"type": "module"`
- [ ] CI pipeline includes CJS-detection scan to prevent regression
- [ ] Application starts and passes integration tests in ESM mode

---

## Open Questions

- **Monorepo support**: How does Kantra handle monorepos with mixed CJS/ESM packages (e.g., Turborepo, Nx workspaces)? Each package may have its own `"type"` field and migration state.
- **TypeScript `moduleResolution` guidance**: Should the ruleset recommend `"module": "nodenext"` in tsconfig.json? This enables compile-time detection of ESM issues but has its own migration cost.
- **`require(esm)` as escape hatch**: Should Konveyor recommend `require(esm)` (Node 22+) as a viable intermediate state, or push for full ESM conversion? The answer may depend on the organization's Node.js version policy.
- **Rule priority and ordering**: When a codebase has 500+ findings, how should Kantra prioritize them? Leaf modules first? Highest-risk dynamic requires first? Most-imported files first?
- **Jest-to-Vitest scope**: Is test framework migration in scope for a "CJS-to-ESM" ruleset, or should it be a separate ruleset? The concerns overlap but aren't identical.
- **Ecosystem adoption metrics**: Can Konveyor track ESM adoption rates across analyzed projects to inform ruleset priorities and demonstrate community impact?

## Next Steps

- [ ] Prioritize Konveyor product stories for backlog grooming (K-1 and K-2 are prerequisites for all others)
- [ ] Validate personas with real Node.js developers and library maintainers via customer interviews
- [ ] Prototype the CJS-to-ESM detection ruleset (K-1) using existing Node.js provider and `builtin.filecontent`
- [ ] Evaluate cross-file async infection tracing feasibility using TypeScript Language Server capabilities
- [ ] Create detailed implementation plans for high-priority stories (K-1, K-2, K-5)
- [ ] Research and benchmark against existing tools (cjstoesm, to-esm, jscodeshift codemods) to identify differentiation points

## Sources

- [Node.js 24 Becomes LTS — NodeSource](https://nodesource.com/blog/nodejs-24-becomes-lts)
- [ESM in 2026: The End of CommonJS — Jeff Bruchado](https://jeffbruchado.com.br/en/blog/esm-2026-end-commonjs-modern-javascript)
- [require(esm): From Experiment to Stability — Joyee Cheung](https://joyeecheung.github.io/blog/2025/12/30/require-esm-in-node-js-from-experiment-to-stability/)
- [Move on to ESM-only — Anthony Fu](https://antfu.me/posts/move-on-to-esm-only)
- [Jest ESM Documentation](https://jestjs.io/docs/ecmascript-modules)
- [Vitest vs Jest 30 in 2026](https://dev.to/dataformathub/vitest-vs-jest-30-why-2026-is-the-year-of-browser-native-testing-2fgb)
- [require(esm) Backported to Node.js 20 — Socket](https://socket.dev/blog/require-esm-backported-to-node-js-20)
- [Node.js 25.8.1 CJS/ESM Fix — LinuxCompatible](https://www.linuxcompatible.org/story/release-corrects-commonjs-module-errors-in-projects)
- [Guide to package.json exports — Hiroki Osame](https://hirok.io/posts/package-json-exports)
- [Publishing dual ESM+CJS packages — Mayank](https://mayank.co/blog/dual-packages/)
- [Migrate 60k LOC TypeScript to ESM — Logto](https://dev.to/logto/migrate-a-60k-loc-typescript-nodejs-repo-to-esm-and-testing-become-4x-faster-12-5f82)
- [Migrating a 1M-Line Backend to TypeScript + ESM — Medium](https://xuanlong43.medium.com/from-commonjs-to-typescript-esm-migrating-a-1m-line-backend-to-typescript-part-2-7e2e232be411)
- [cjstoesm — GitHub](https://github.com/wessberg/cjstoesm)
- [10 Points for CJS-to-ESM Migration — Daniel Jimenez Garcia](https://medium.com/@danieljimgarcia/10-points-to-consider-when-migrating-commonjs-to-es-modules-in-node-abe8ed31ac10)
- [Node.js v22 to v24 Migration Guide](https://nodejs.org/en/blog/migrations/v22-to-v24)
