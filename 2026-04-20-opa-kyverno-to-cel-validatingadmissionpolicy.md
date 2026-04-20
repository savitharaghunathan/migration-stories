---
date: 2026-04-20
topic: opa-kyverno-to-cel-validatingadmissionpolicy
personas: [Platform Engineer, Security / Compliance Lead]
konveyor-components: [Kantra, Rulesets, Kai]
migration-type: modernization
---

# Migration Stories: OPA/Kyverno Policies to Kubernetes-Native CEL (ValidatingAdmissionPolicy)

## Migration Landscape

### What is changing?

Kubernetes now has built-in policy enforcement. ValidatingAdmissionPolicy (VAP) reached GA in Kubernetes 1.30, and its mutation counterpart MutatingAdmissionPolicy (MAP) reaches GA in Kubernetes 1.36 (April 2026). Both use the Common Expression Language (CEL) — no external webhook, no separate policy engine, no Rego.

This creates a migration path away from external policy engines:

- **OPA Gatekeeper** teams can migrate Rego-based ConstraintTemplates to CEL-based ValidatingAdmissionPolicy resources. Gatekeeper v3.22 (February 2026) already bridges both worlds — it supports a `K8sNativeValidation` engine that lets you write CEL inside ConstraintTemplates, and can auto-generate native VAP resources.
- **Kyverno** teams can migrate JMESPath-based ClusterPolicy resources to CEL-based ValidatingPolicy (Kyverno's superset of VAP). Kyverno 1.17 (February 2026) promoted CEL policies to GA and **deprecated** the legacy JMESPath-based ClusterPolicy API, with removal scheduled for v1.20 (October 2026).

### Who is affected?

Any team running OPA Gatekeeper or Kyverno for Kubernetes admission control. Policy definitions live in source repos as YAML files — ConstraintTemplates, Constraints, ClusterPolicies — maintained by platform and security teams. Both OPA Gatekeeper and Kyverno are CNCF Graduated projects with broad enterprise adoption.

The migration has two distinct populations:

1. **Kyverno users** — hard deadline. Legacy ClusterPolicy API is deprecated now, removal in October 2026. They must migrate.
2. **OPA Gatekeeper users** — no hard deadline, but Gatekeeper is actively building bridges to VAP/CEL. The ecosystem momentum is clear.

### What is the timeline?

| Date | Event |
|------|-------|
| Jun 2024 | ValidatingAdmissionPolicy GA in Kubernetes 1.30 |
| Feb 2026 | Gatekeeper v3.22 — `sync-vap-enforcement-scope` enabled by default |
| Feb 2026 | Kyverno 1.17 — CEL policies GA, legacy ClusterPolicy **deprecated** |
| Apr 2026 | Kubernetes 1.36 — MutatingAdmissionPolicy GA |
| Apr 2026 | Kyverno 1.18 — critical fixes only for legacy APIs |
| **Oct 2026** | **Kyverno 1.20 — legacy ClusterPolicy API removed** |

### Migration paths

**Path 1: Migrate simple policies to native VAP, keep complex ones on external engine**
The hybrid approach. Move straightforward validations (required labels, image tag restrictions, resource limits) to native CEL ValidatingAdmissionPolicy. Keep complex policies that need external data, cross-resource reasoning, or resource generation on Gatekeeper/Kyverno. Lowest risk, most common recommendation.

**Path 2: Full migration to Kubernetes-native CEL**
Eliminate the external policy engine entirely. Only viable if your policies are all simple validations without external data dependencies, cross-resource reasoning, or mutation/generation needs that exceed MAP capabilities.

**Path 3: Adopt engine's built-in CEL bridge**
Use Gatekeeper's `K8sNativeValidation` engine or Kyverno's `ValidatingPolicy` to write CEL within the existing framework. Gets CEL benefits while keeping the engine's audit, reporting, and management features. Least disruptive, but doesn't eliminate the external dependency.

### What does Konveyor offer today?

- **yq-external-provider** — analyzes YAML files, understands Kubernetes `apiVersion`/`kind` fields, can validate against K8s API versions. This is the foundation for scanning policy YAML files in source repos.
- **Kubernetes provider enhancement** — designed to use OPA/Rego as its rules engine for analyzing Kubernetes resources. Supports both live cluster analysis and disconnected YAML analysis via `k8s-resource.rego_expr` and `k8s-resource.rego_module` capabilities. Implementation started January 2024.
- **builtin.filecontent** conditions — existing rule conditions that can match patterns in structured files.

Konveyor does not currently have rulesets for detecting or classifying Kubernetes policy resources (ConstraintTemplates, ClusterPolicies, ValidatingAdmissionPolicies). The yq provider and k8s-provider give it the infrastructure to do so.

## Personas

### Persona 1 — Platform Engineer

**Background:** Owns the Kubernetes platform including admission control. Maintains the policy engine (Gatekeeper or Kyverno), writes and deploys policies, manages upgrades. Policy definitions live in Git repos they maintain.
**Technical depth:** Expert with Kubernetes, intermediate with Rego or Kyverno policy syntax, learning CEL
**Migration authority:** Implementer — converts policies and deploys the new resources
**What keeps them up at night:** Kyverno 1.20 removes the API they depend on. Every ClusterPolicy in their repo needs to become a ValidatingPolicy before October 2026. They have 50+ policies and no automated way to assess which ones can move to CEL cleanly.

### Persona 2 — Security / Compliance Lead

**Background:** Defines what policies must exist, reviews policy changes, ensures compliance posture is maintained through the migration. Doesn't write Rego or CEL but needs to verify that migrated policies enforce the same intent.
**Technical depth:** Expert with compliance frameworks, intermediate with policy concepts, novice with CEL/Rego syntax
**Migration authority:** Decision maker — approves policy changes and the migration timeline
**What keeps them up at night:** A policy migration that silently weakens enforcement. If a Rego policy caught edge cases that the CEL equivalent misses, they won't know until an audit fails.

## Empathy Maps

### Persona 1 — Platform Engineer — Empathy Map

**Thinks:**
- "I have 50+ ClusterPolicies. Some are simple label checks, some do complex cross-resource validation. I need to know which ones can migrate to CEL and which can't before I estimate effort."
- "CEL syntax is different from JMESPath and Rego. I need to learn a new expression language while converting policies under a deadline."
- "Gatekeeper's K8sNativeValidation bridge is tempting — I can write CEL inside my existing ConstraintTemplates without changing the deployment model."

**Feels:**
- Pressured — Kyverno's deprecation timeline is real and October 2026 is close
- Cautious — policy migration is high-stakes; a broken policy either blocks legitimate workloads or lets violations through
- Frustrated — no tooling exists to scan a repo of policies and classify them by migration complexity

**Says:**
- "Which of our policies can move to native VAP and which need to stay on the engine?"
- "I need a side-by-side comparison — does this CEL expression catch the same cases as the Rego/JMESPath original?"
- "Can we run both in parallel during the transition?"

**Does:**
- Manually reads each policy to assess whether it uses features CEL can't handle (external data, cross-resource reasoning, generation)
- Converts one simple policy as a proof of concept to learn CEL syntax
- Tests converted policies in audit/dryrun mode before switching to enforce

**Pain Points:**
1. No automated way to inventory policies and classify them by CEL migration feasibility
2. Manual conversion is tedious and error-prone — easy to miss edge cases in the translation
3. No validation that the CEL equivalent is functionally identical to the original policy

**Goals:**
1. Complete the migration before the legacy API removal deadline
2. Zero policy enforcement gaps during the transition

**Unspoken Needs:**
- A classification report they can hand to management showing scope and effort
- Confidence that converted policies are equivalent — not just syntactically valid but semantically identical

### Persona 2 — Security / Compliance Lead — Empathy Map

**Thinks:**
- "I approved these policies based on what they enforce. If the platform team rewrites them in a different language, how do I verify they still enforce the same thing?"
- "We need to maintain policy coverage during the migration. No gap where a policy is removed from the old engine but not yet active in the new one."

**Feels:**
- Anxious about policy equivalence — the migration is a language translation, and translations can lose meaning
- Dependent on the platform engineer to get the conversion right

**Says:**
- "Show me that the migrated policy catches the same violations as the original."
- "Can we run old and new side by side and compare results before switching?"

**Does:**
- Reviews migration plans and asks for test evidence that converted policies catch the same violations
- Requires a parallel-run period where both old and new policies are active in audit mode

**Pain Points:**
1. Can't independently verify policy equivalence without learning CEL
2. No compliance report showing which policies have been migrated and validated vs. which are pending

**Goals:**
1. Policy enforcement posture is maintained or improved through the migration — never weakened

**Unspoken Needs:**
- Human-readable summary of what each converted policy enforces, so review doesn't require reading CEL

## Konveyor Opportunity Analysis

| Konveyor Capability | Opportunity | What Exists Today | Impact |
|---------------------|-------------|-------------------|--------|
| **Analysis (Kantra)** | Scan source repos containing Gatekeeper ConstraintTemplates or Kyverno ClusterPolicies. Detect policy type, classify by CEL migration feasibility, report migration scope. | yq-external-provider can parse YAML with K8s `apiVersion`/`kind`. k8s-provider enhancement uses OPA/Rego for Kubernetes resource analysis. No policy migration rulesets exist. | **High** |
| **Rulesets** | A `kubernetes/policy-migration` ruleset detecting: (1) Kyverno ClusterPolicy resources with deprecated API, (2) Gatekeeper ConstraintTemplates using Rego-only features (external data, `data.inventory` references), (3) policies using features CEL can't express. Classification output: "CEL-ready" vs "keep on engine." | No Kubernetes policy rulesets exist. The yq provider and k8s-provider give the infrastructure to build them. | **High** |
| **AI-Assisted Refactoring (Kai)** | Generate CEL equivalents from Rego or JMESPath policy logic. Kyverno provides a field-by-field migration guide; Gatekeeper provides CEL template examples. Kai could learn from these patterns and auto-generate converted policies. | Kai supports Java, Go, TypeScript, C#. It does not support YAML policy transformation. The pattern (detect then suggest equivalent) is the same as code migration. | **Medium (gap)** |

### Gaps

| Gap | Why It Matters |
|-----|---------------|
| **No Kubernetes policy rulesets** | Can't detect or classify policy resources without rules targeting ConstraintTemplate, Constraint, ClusterPolicy apiVersions |
| **No CEL migration feasibility classifier** | The critical question — "can this policy move to native CEL?" — requires detecting Rego features (external data, `data.inventory`, `http.send`) and Kyverno features (generate, mutate with complex patches, context API calls) that CEL can't express |
| **Kai doesn't support YAML-to-YAML transformation** | Policy conversion is a structured transformation between known formats — same pattern as code migration but in YAML |
| **No policy equivalence validation** | After conversion, need to verify the CEL policy catches the same violations as the original |

## Konveyor Product Stories

---

### Story K-1: Kubernetes Policy Detection Ruleset

**Persona:** Platform Engineer
**Empathy context:** They have 50+ policies in a repo and no automated way to assess migration scope. Manual inventory takes days.

**Story:**
As a Platform Engineer, I want Kantra to scan my policy repo and identify all Gatekeeper ConstraintTemplates, Constraints, Kyverno ClusterPolicies, and existing ValidatingAdmissionPolicies, so that I have a complete inventory of what needs to migrate.

**Acceptance Criteria:**
- [ ] Detects Gatekeeper `ConstraintTemplate` resources (`apiVersion: templates.gatekeeper.sh/v1`)
- [ ] Detects Gatekeeper `Constraint` resources (custom apiGroup matching ConstraintTemplate names)
- [ ] Detects Kyverno `ClusterPolicy` and `Policy` resources (`apiVersion: kyverno.io/v1`) using deprecated JMESPath-based API
- [ ] Detects existing `ValidatingAdmissionPolicy` resources (`apiVersion: admissionregistration.k8s.io/v1`)
- [ ] Summary report: count by type, by engine, migration status

**Konveyor component:** Kantra + Rulesets (using yq provider)
**Priority signal:** This is the entry point. No migration planning without an inventory.
**What exists today:** yq provider can parse YAML with apiVersion/kind. Rules need to be authored targeting these specific resource types.

---

### Story K-2: CEL Migration Feasibility Classifier

**Persona:** Platform Engineer
**Empathy context:** Not all policies can move to native CEL. They need to know which ones can and which ones must stay on the external engine — before they estimate effort.

**Story:**
As a Platform Engineer, I want Kantra to classify each detected policy as "CEL-ready" or "keep on engine" based on whether it uses features that CEL cannot express, so that I can plan the migration scope accurately.

**Acceptance Criteria:**
- [ ] Flags Rego policies using `data.inventory` (cross-resource reasoning — CEL can't do this)
- [ ] Flags Rego policies using `http.send` or external data providers (CEL has no external data access)
- [ ] Flags Kyverno policies using `generate` rules (VAP has no resource generation)
- [ ] Flags Kyverno policies using `context` with API calls or ConfigMap lookups
- [ ] Flags Kyverno policies using complex `mutate` patches that exceed MutatingAdmissionPolicy capabilities
- [ ] Classifies remaining policies as "CEL-ready" with effort estimate (simple / moderate)
- [ ] Summary: N policies CEL-ready, M policies must stay on engine, with reasons

**Konveyor component:** Kantra + Rulesets
**Priority signal:** This is the decision-support story. Without classification, the Platform Engineer either migrates everything (breaks complex policies) or nothing (misses the deadline).
**What exists today:** yq provider can parse the YAML. Rego feature detection requires pattern matching against Rego code blocks inside ConstraintTemplates. The k8s-provider's Rego expertise could inform rule design.

---

### Story K-3: Deprecated Kyverno API Detection

**Persona:** Platform Engineer
**Empathy context:** Kyverno 1.20 removes the legacy ClusterPolicy API in October 2026. Hard deadline with zero ambiguity.

**Story:**
As a Platform Engineer running Kyverno, I want Kantra to flag all policies using the deprecated JMESPath-based ClusterPolicy API, so that I know exactly what must be converted before the October 2026 removal.

**Acceptance Criteria:**
- [ ] Detects `ClusterPolicy` resources using `validate.pattern`, `validate.anyPattern`, or `validate.deny` with JMESPath conditions
- [ ] Detects `CleanupPolicy` resources (also deprecated)
- [ ] Each finding includes the file path, policy name, and specific deprecated fields used
- [ ] Finding message includes a pointer to Kyverno's migration guide (`kyverno.io/docs/guides/migration-to-cel/`)

**Konveyor component:** Kantra + Rulesets
**Priority signal:** Hard deadline — October 2026. Kyverno's own migration guide provides field-by-field mappings that rule messages can reference.
**What exists today:** yq provider can detect the apiVersion and field patterns. Rules need to be authored.

---

### Story K-4: Policy Migration Progress Tracking

**Persona:** Security / Compliance Lead
**Empathy context:** They need to know: how many policies have been migrated and validated, how many are pending, and whether enforcement posture is maintained throughout.

**Story:**
As a Security / Compliance Lead, I want a report showing policy migration status — converted, validated, pending, and not-migratable — so that I can verify compliance posture is maintained throughout the migration.

**Acceptance Criteria:**
- [ ] Report groups policies by migration status: not started, converted (pending validation), validated, not migratable (staying on engine)
- [ ] Shows total policy count and percentage complete
- [ ] Flags any gaps where a policy has been removed from the old engine but no CEL equivalent exists yet

**Konveyor component:** Kantra + Hub
**Priority signal:** The Security Lead's primary concern is coverage gaps during migration. This report is how they verify none exist.
**What exists today:** Kantra can re-scan after each migration batch. Hub could track status if integrated.

---

## End-User Migration Story

### Story M-1: Migrate Kubernetes Policies to CEL

**Persona:** Platform Engineer + Security / Compliance Lead
**Empathy context:** The whole team needs a repeatable process — the platform engineer converts policies, the security lead verifies equivalence, and progress is tracked to completion.

**Story:**
When our team decides to migrate Kubernetes policies from OPA Gatekeeper or Kyverno to native CEL, I want a single Konveyor-driven workflow that assesses scope, guides the conversion, validates equivalence, and tracks progress across the policy portfolio.

**Steps with Konveyor:**
1. **Assess** — Run `kantra analyze` with `kubernetes/policy-migration` ruleset across the policy repo. Platform Engineer gets classification report (CEL-ready vs. keep-on-engine). Security Lead gets migration scope summary.
2. **Convert** — Platform Engineer works through CEL-ready policies using Kyverno's field-by-field migration guide or Gatekeeper's K8sNativeValidation examples. Each converted policy deployed in audit mode alongside original.
3. **Validate** — Run both original and converted policies in parallel. Compare audit results. Security Lead reviews and approves each conversion.
4. **Track** — Re-run `kantra analyze` after each batch. Track portfolio completion: converted, validated, pending, not-migratable.

**Without Konveyor:** Platform Engineer manually reads each policy file, mentally assesses CEL feasibility, converts without tooling, tests by deploying and hoping. Security Lead gets verbal status updates. Migration progress lives in a spreadsheet nobody updates. Risk of missing a policy and discovering it when the API gets removed.

**Definition of Done:**
- [ ] All policies assessed with classification report generated
- [ ] CEL-ready policies converted and validated via parallel-run
- [ ] Non-migratable policies documented with reasons for staying on engine
- [ ] Portfolio-level progress tracked to completion

## Open Questions

- Has the k8s-provider enhancement progressed beyond the initial January 2024 implementation? Is it usable for policy YAML analysis today?
- Can the yq provider detect patterns inside embedded Rego code blocks within ConstraintTemplate YAML, or does it only match YAML-level fields?
- Should the `kubernetes/policy-migration` ruleset target both Gatekeeper and Kyverno in one ruleset, or split into `gatekeeper-to-cel` and `kyverno-to-cel`?
- How would Kai support YAML-to-YAML policy transformation? Is this a natural extension of its code transformation capabilities or a fundamentally different problem?
- What is the intersection with Konveyor's k8s-provider using Rego as its rules engine — does the k8s-provider itself need to migrate from Rego to CEL as the Kubernetes ecosystem converges?

## Next Steps

- [ ] Prioritize Konveyor product stories for backlog grooming
- [ ] Validate that yq provider can detect ConstraintTemplate, ClusterPolicy, and ValidatingAdmissionPolicy resource types
- [ ] Prototype a `kubernetes/policy-migration` ruleset with 3-5 rules targeting the most common deprecated patterns
- [ ] Validate personas with platform teams running Gatekeeper or Kyverno in production
- [ ] Assess whether k8s-provider's Rego engine could be used for policy equivalence testing

## Sources

- [Kubernetes ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [Kubernetes CEL Reference](https://kubernetes.io/docs/reference/using-api/cel/)
- [Kubernetes 1.36 — MutatingAdmissionPolicy GA](https://dev.to/x4nent/complete-guide-to-kubernetes-136-dra-ga-oci-volumesource-mutatingadmissionpolicy-and-2h8b)
- [Gatekeeper ValidatingAdmissionPolicy Integration](https://open-policy-agent.github.io/gatekeeper/website/docs/validating-admission-policy/)
- [Gatekeeper v3.22 Release](https://github.com/open-policy-agent/gatekeeper/releases/tag/v3.22.0)
- [Kyverno Migration to CEL Guide](https://kyverno.io/docs/guides/migration-to-cel/)
- [Kyverno 1.17 Announcement](https://kyverno.io/blog/2026/02/02/announcing-kyverno-release-1.17/)
- [Kyverno CEL Policies GA — Dedico](https://www.dedico.hu/en/posts/kyverno-cel-policies-v1/)
- [Kubescape CEL Admission Library](https://kubescape.io/blog/2024/08/04/cel-and-kubescape/)
- [Standardizing Security Policies with VAP — Getup Cloud](https://getup.io/en/blog/standardizing-enforcement-of-security-policies)
- [OPA/Gatekeeper vs Kyverno Comparison](https://policyascode.dev/blog/opa-gatekeeper-vs-kyverno/)
- [Konveyor k8s-provider Enhancement](https://github.com/konveyor/enhancements/blob/main/enhancements/analyzer-k8s-provider/README.md)
- [Eliminating the Rego Tax — Red Hat Emerging Technologies](https://next.redhat.com/2026/03/20/eliminating-the-rego-tax-how-ai-orchestrators-automate-kubernetes-compliance/)
