---
name: release-versioning-management-core
description: "Semantic versioning (SemVer 2.0.0), Conventional Commits-driven automated version bumping, changelog generation (Keep a Changelog format), monorepo/multi-package release graphs (Changesets/Lerna/Nx), release branching + git tag strategies, release automation pipelines (semantic-release, release-please, goreleaser), API breaking-change detection, and release risk/rollback modeling. Use when setting up automated versioning, generating a CHANGELOG.md, designing a release pipeline, deciding a monorepo version-bump strategy, or auditing whether a release is safe to ship. Keywords: semantic versioning, semver, changelog generation, conventional commits, release automation, version bump, git tagging strategy, monorepo release, breaking change detection, release pipeline, rollback strategy, release notes"
allowed-tools: Read,Glob,Grep,Bash,Edit,Write
user-invocable: true
---


<!-- generated-by: claude-global-library/tools/project_sync -->
<!-- source: skills/release-versioning-management-core/SKILL.md -- edit the library, then re-run sync_project.py -->

# Release & Versioning Management Core Skill Instructions

### Skill Name
release-versioning-management-core

### Description
Comprehensive release engineering knowledge covering the full version-to-ship lifecycle: SemVer 2.0.0 precedence rules, Conventional Commits-driven automated bump classification, changelog generation and formatting, monorepo/multi-package version propagation, git branching and tagging strategy, release automation tooling (semantic-release, release-please, goreleaser, Changesets), API breaking-change detection, and release risk/rollback decision-making. Includes India-specific regulatory layer for change-management audit trails (CERT-In, RBI, SEBI CSCRF, DPDP Act 2023).

---

## 1. Semantic Versioning (SemVer 2.0.0)

### 1.1 Version Format and Precedence

A version is `MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]`:

```
MAJOR — incompatible API changes (breaking change)
MINOR — backward-compatible functionality added
PATCH — backward-compatible bug fixes
PRERELEASE — optional, dot-separated identifiers after a hyphen (e.g. -alpha.1, -rc.2)
BUILD  — optional, dot-separated metadata after a plus (e.g. +20260724.sha.a1b2c3) — MUST be ignored when determining precedence
```

**Precedence rules (SemVer spec §11):**
1. Compare MAJOR, MINOR, PATCH numerically, left to right — first difference wins.
2. A version WITHOUT a pre-release tag has HIGHER precedence than one WITH a pre-release tag, for equal MAJOR.MINOR.PATCH (`1.0.0` > `1.0.0-rc.1`).
3. Pre-release identifiers are compared field-by-field, left to right:
   - Purely numeric identifiers compare numerically.
   - Alphanumeric identifiers compare lexically (ASCII).
   - Numeric identifiers always have lower precedence than alphanumeric identifiers in the same position.
   - A larger set of pre-release fields has higher precedence than a smaller set, if all preceding fields are equal.

Example ordering (lowest to highest):
```
1.0.0-alpha < 1.0.0-alpha.1 < 1.0.0-alpha.beta < 1.0.0-beta
< 1.0.0-beta.2 < 1.0.0-beta.11 < 1.0.0-rc.1 < 1.0.0
```

### 1.2 Version Range Operators

| Operator | Meaning | Example | Matches |
|---|---|---|---|
| `^1.2.3` (caret) | Compatible within MAJOR (or MINOR if MAJOR=0) | `^1.2.3` | `>=1.2.3 <2.0.0` |
| `^0.2.3` | Compatible within MINOR when MAJOR=0 | `^0.2.3` | `>=0.2.3 <0.3.0` |
| `~1.2.3` (tilde) | Compatible within MINOR | `~1.2.3` | `>=1.2.3 <1.3.0` |
| `1.2.x` | Wildcard PATCH | `1.2.x` | `>=1.2.0 <1.3.0` |
| `>=1.2.3 <2.0.0` | Explicit range | — | direct interval |

### 1.3 Zero-Major Convention

`0.y.z` signals the API is **not yet stable** — any change, including breaking ones, MAY be released as a MINOR or PATCH bump. `1.0.0` is the contract commitment point: once published, MAJOR bumps are the only sanctioned vehicle for breaking changes.

---

## 2. Conventional Commits → Automated Version Bumping

### 2.1 Commit Grammar

```
<type>[optional scope][optional !]: <description>

[optional body]

[optional footer(s)]
```

**Type → bump mapping:**

| Type | Bump | Changelog section |
|---|---|---|
| `fix:` | PATCH | Bug Fixes |
| `feat:` | MINOR | Features |
| `feat!:` / `fix!:` / any type with `!` / footer `BREAKING CHANGE:` | MAJOR | ⚠ BREAKING CHANGES |
| `docs:`, `style:`, `refactor:`, `perf:`, `test:`, `build:`, `ci:`, `chore:` | none (unless `!`) | grouped under their own heading or omitted |

`refactor:` and `perf:` MAY be promoted to PATCH if the project's convention treats behavior-preserving-but-risky changes as release-worthy — this is a project-level policy decision, not a Conventional Commits requirement.

### 2.2 Aggregation Rule

Across all commits since the last release, the overall bump is the **maximum** severity found:

```
bump(commits) = max_i( bump(commit_i) )   where MAJOR > MINOR > PATCH > none
```

This is formalized rigorously in **M2** below.

---

## 3. Changelog Generation

### 3.1 Keep a Changelog Format

```markdown
## [1.4.0] - 2026-07-24
### Added
- New `--dry-run` flag for the release CLI (#412)
### Fixed
- Race condition in concurrent tag creation (#418)
### Changed
- Bumped minimum Node.js version to 20 (#420)
### BREAKING CHANGES
- Removed deprecated `--legacy-tag-format` flag (#421)
```

Rules: newest release first (reverse-chronological), one `##` heading per version + ISO 8601 date, sub-headings limited to `Added / Changed / Deprecated / Removed / Fixed / Security` (Keep a Changelog v1.1.0 canonical set), every entry links back to its PR/issue for auditability, an `[Unreleased]` section always sits at the top for in-flight work.

### 3.2 Auto-Generation from Commit History

```
1. git log <last_tag>..HEAD --pretty=format:"%H|%s|%b"
2. Parse each subject line against the Conventional Commits grammar (§2.1)
3. Partition into buckets by type (feat/fix/breaking/other)
4. Deduplicate near-identical entries (see M3 similarity clustering)
5. Order within each bucket (see M3 topological ordering by scope)
6. Render using the project's template (Keep a Changelog / GitHub auto-notes / custom)
```

### 3.3 Release Notes vs Changelog

A **changelog** is a durable, append-only engineering record (every entry, forever). **Release notes** are a curated, audience-facing subset — written for end users, may omit `chore:`/`ci:` noise, may group multiple commits into one narrative bullet. Never conflate the two: generate the changelog mechanically, curate release notes editorially (or via an LLM summarization pass over the mechanical changelog — never the reverse).

---

## 4. Monorepo & Multi-Package Versioning

### 4.1 Independent vs Fixed (Lockstep) Versioning

| Strategy | Every package has its own version | All packages share one version |
|---|---|---|
| Tooling | Changesets (independent mode), Lerna `--independent`, Nx per-project release | Lerna fixed mode, Nx `--fixedGroup`, monorepo single-version pattern |
| Pro | Minimal diff noise; a patch to one package doesn't force a bump everywhere | Simple mental model; one version number to communicate externally |
| Con | Consumers must track N version numbers | A one-line fix in a leaf package forces a MAJOR release across the whole repo if any package has a breaking change |

### 4.2 Changesets Workflow

```
1. Contributor runs `changeset add` → writes a markdown file describing bump-level + affected packages
2. On merge to main, changesets accumulate in .changeset/
3. Release job runs `changeset version` → bumps every affected package.json + generates CHANGELOG.md per package
4. Dependency graph propagation (see M4) bumps downstream packages that depend on a bumped package
5. Release job runs `changeset publish` → npm publish + git tag per package
```

### 4.3 Cascading Bump Propagation

When package `A` bumps MAJOR, every package that depends on `A` via a range that no longer matches (e.g. `^1.0.0` when A goes to `2.0.0`) must itself receive at minimum a PATCH bump (to update its lockfile/manifest pointer), formalized in **M4**.

---

## 5. Release Branching & Git Tagging Strategy

### 5.1 Trunk-Based Release Trains

```
main ──●──●──●──●──●──●──●──●──▶
        │        │        │
      v1.2.0   v1.3.0   v1.4.0   (tags cut directly from main, no long-lived release branch)
```

Best for: continuous deployment, weekly/daily release cadence, single supported version in production.

### 5.2 Release-Branch Model (for maintained majors)

```
main ────●────●────●────●──▶
          \
release/1.x ●──●──●──▶  (backport-only: fix: cherry-picked from main)
              v1.4.1  v1.4.2
```

Best for: libraries/SDKs with multiple concurrently supported MAJOR versions, LTS commitments.

### 5.3 Tag Signing and Immutability

- Tags MUST be annotated (`git tag -a vX.Y.Z -m "..."`), never lightweight — annotated tags carry tagger identity + timestamp + message, required for supply-chain provenance (SLSA §2 — see `devsecops-core`).
- Sign tags with `git tag -s` (GPG) or keyless `cosign` (sigstore) in CI — a release tag is a security-relevant artifact, not just a label.
- **Never** move or re-push an existing tag (`git tag -f`). A published version is immutable by SemVer contract (§1.3) — republish as a new PATCH instead.

---

## 6. Release Automation Pipelines

### 6.1 semantic-release (Node ecosystem)

```
1. Analyze commits since last tag (§2.2 aggregation)
2. Determine next version
3. Generate release notes
4. Create GitHub/GitLab release + git tag
5. Publish to registry (npm, Docker Hub, etc.)
6. Update CHANGELOG.md and commit back to main
```
Fully automated — no human-authored version number ever touches `package.json`. Runs on every merge to the release branch; a `none` bump is a no-op (skips publish).

### 6.2 release-please (Google, language-agnostic)

Maintains a standing "release PR" that accumulates changes and a computed next-version + changelog diff. A human merges the release PR to actually cut the release — same commit analysis as semantic-release, but with an explicit human approval gate. Preferred when release timing must be deliberate (regulated environments, App Store review windows) rather than continuous.

### 6.3 goreleaser (Go, cross-compiled binaries)

Handles the artifact side of release automation: cross-compilation matrix, checksums, SBOM attachment, container image + signing, and GitHub Release asset upload — driven off an existing git tag rather than deciding the version itself (commonly paired with a bump tool from §6.1/§6.2 upstream).

### 6.4 CI/CD Release Gate Placement

```
Phase D (QA: 100% coverage, DRE=1.0) → Phase F (Security: 0 findings) → Phase E (RS=1.0)
   → Release Gate: version bump computed (§2.2) → changelog generated (§3.2)
   → tag created + signed (§5.3) → artifact published → GitHub/GitLab Release created
   → deployment triggered (canary/blue-green — see §8)
```
The release gate MUST run strictly after Phase E (Reliability Gate) — a version number is a promise to consumers; never mint one for an artifact that has not cleared RS = 1.0.

---

## 7. API Compatibility & Breaking-Change Detection

### 7.1 Public API Surface Diffing

```
1. Extract the public API surface at tag T_{n-1} and at HEAD (exported symbols: functions, types,
   REST/GraphQL schema, CLI flags — whatever the project's contract surface is)
2. Diff the two surfaces:
   Removed(T_{n-1} → HEAD)   = surface(T_{n-1}) \ surface(HEAD)   → breaking (removal)
   Changed-signature          = same name, incompatible signature   → breaking (mutation)
   Added(HEAD \ T_{n-1})     = surface(HEAD) \ surface(T_{n-1})   → non-breaking (additive)
3. If Removed ∪ Changed-signature ≠ ∅ → MAJOR is mandatory regardless of what commit messages claimed
```

Tooling examples: `api-extractor` (TypeScript), `revive`/`apidiff` (Go), `japicmp` (Java), `oasdiff` (OpenAPI schemas). This diff is the ground truth; Conventional Commits `!`/`BREAKING CHANGE:` markers are a *hint* a human wrote, not a proof — §M2 quantifies how often the hint and the diff disagree.

### 7.2 Deprecation Policy Before Removal

```
1. Mark deprecated: emit runtime warning + @deprecated annotation + changelog "Deprecated" entry
   (MINOR bump — deprecation itself is non-breaking)
2. Maintain for ≥ 1 full MAJOR cycle (or the project's published deprecation window, e.g. "2 minor releases")
3. Remove only in the next MAJOR, with the removal itself listed under BREAKING CHANGES
```

---

## 8. Release Risk, Canary Rollout, and Rollback

### 8.1 Progressive Delivery Ladder

```
Internal canary (1% traffic, employees only)
  → Canary (5% production traffic, automated health-gate)
  → Staged rollout (25% → 50% → 100%, each stage gated on error-rate/latency SLOs)
  → Full rollout
```
Automated rollback trigger: error-rate or p99-latency regression beyond a statistical threshold (formalized as a sequential test in **M5**) halts promotion and reverts traffic to the previous known-good version.

### 8.2 Rollback Strategy

- **Blue-green**: keep the previous version's full environment warm; rollback = traffic-switch (seconds, zero rebuild).
- **Rolling**: rollback = redeploy previous tag through the same rolling-update mechanism (minutes).
- A rollback is a **deployment** action, never a **version** action — never re-tag or delete the bad release's version number; instead ship the fix as a new PATCH/MAJOR (per §5.3 immutability) and deploy that.

---

## 9. India Regulatory / Compliance Layer

- **CERT-In Directions 2022 §3(v)**: security-incident-triggered rollbacks must be reportable within 6 hours — the release/rollback audit trail (§5.3 signed tags + §3.1 changelog + deployment event log) is the evidentiary record; ensure timestamps are retained and queryable.
- **RBI IT Outsourcing / Cyber Security Framework**: regulated financial entities must maintain a documented change-management trail for every production release — the Keep a Changelog record (§3.1) plus signed git tags (§5.3) satisfy the "documented and traceable" requirement when retained per the entity's records-retention policy.
- **SEBI CSCRF**: SBOM-linked release provenance (see `devsecops-core` §6–7) should be cross-referenced from the release notes for regulated capital-markets systems — the version tag is the join key between the changelog and the SBOM artifact.
- **DPDP Act 2023 §4/§8**: when a release changes what personal data is processed or how, the release notes MUST flag it distinctly (e.g. a `### Data Processing Changes` heading) so it can be routed to the Data Protection Officer / consent-notice update workflow — this is a release-notes authoring rule, not a versioning-mechanics rule.

---

## Deep Mathematical Foundations

### M1: SemVer as a Formal Total Order + Range Constraint Satisfaction

**Precedence as a lexicographic total order.**

Define a version (ignoring build metadata, which is precedence-irrelevant by spec) as a tuple:
```
v = (M, m, p, PR)   where M, m, p ∈ ℕ  (MAJOR, MINOR, PATCH)
                     PR = pre-release identifier sequence, or ⊥ (absent = release)
```

Precedence `≺` is defined lexicographically on `(M, m, p)` first:
```
v₁ ≺ v₂  ⟺  (M₁, m₁, p₁) <_lex (M₂, m₂, p₂)
             ∨ [(M₁,m₁,p₁) = (M₂,m₂,p₂) ∧ PR₁ <_PR PR₂]
```
where `<_PR` breaks the tie using the pre-release comparison in §1.1 rule 3. Because `<_lex` on `ℕ³` is a strict total order (trichotomy + transitivity hold coordinate-wise) and `<_PR` is a strict total order on the fixed-`(M,m,p)` fiber, their lexicographic composition `≺` is itself a strict total order on the set of all versions. **Proof of transitivity:** if `v₁ ≺ v₂` and `v₂ ≺ v₃`, either the `(M,m,p)` triples strictly increase (transitivity of `<_lex` on ℕ³ closes it) or they are equal across all three and `<_PR` transitivity closes it — no case allows `v₃ ≺ v₁`. This total order is exactly what makes "latest version" and "sort releases" well-defined operations with no ties to break arbitrarily.

**Range operators as intervals on this order.**

Each range operator from §1.2 denotes a half-open interval `[lo, hi)` under `≺`:
```
^1.2.3  →  [1.2.3, 2.0.0)
^0.2.3  →  [0.2.3, 0.3.0)     (MAJOR=0 collapses the compatible unit to MINOR — spec §9)
~1.2.3  →  [1.2.3, 1.3.0)
1.2.x   →  [1.2.0, 1.3.0)
```
A dependency requirement is therefore a predicate `R(v) = lo ≤ v < hi`. Given a project with dependencies `{d_1, ..., d_k}` each constrained to `R_i`, and a candidate resolution assigning each `d_i` a concrete version `v_i`, the resolution is **valid** iff `∀i: R_i(v_i)` AND every *transitive* constraint (dependency of a dependency) is also satisfied — this is exactly a **constraint-satisfaction problem (CSP)** over the version lattice.

**Version solving as CSP backtracking (PubGrub sketch).**

Naive backtracking over `∏|candidates_i|` is exponential in the worst case (diamond-dependency conflicts). The PubGrub algorithm (used by Cargo, pub, uv) resolves this in practice via **conflict-driven clause learning** borrowed from SAT solving:
```
1. Maintain a partial assignment + a set of "incompatibilities" (minimal sets of
   {package, version-range} pairs that cannot all be true simultaneously)
2. Unit propagation: if all but one term of an incompatibility is satisfied, derive
   the negation of the remaining term (a new constraint) — no branching needed
3. On conflict: instead of blind backtrack, DERIVE a new incompatibility that explains
   the conflict (resolution rule, same as CDCL clause learning) and backjump directly
   to the decision level that caused it
4. Terminate when either a full valid assignment is found, or the root package's own
   requirement becomes provably unsatisfiable (report the derived incompatibility as
   a human-readable "why" trace)
```
Complexity: worst case still exponential (version solving with arbitrary constraints is NP-hard by reduction from 3-SAT — each boolean variable becomes a package with two mutually-exclusive versions, each clause becomes a package required by all three literal-packages), but backjumping + learned incompatibilities make the *practical* case near-linear in the number of packages, because most real dependency graphs are close to a DAG with few genuine conflicts.

### M2: Conventional-Commit Bump Classification as a Monotone Aggregation Function

**Formalizing the bump function.**

Let `Level = {NONE, PATCH, MINOR, MAJOR}` with the total order `NONE < PATCH < MINOR < MAJOR`. Define the per-commit classifier:
```
bump: Commit → Level
bump(c) = MAJOR   if has_bang(c) ∨ has_breaking_footer(c)
        = MINOR   if type(c) = "feat"  (and not MAJOR)
        = PATCH   if type(c) = "fix"   (and not MAJOR)
        = NONE    otherwise
```
The release-level aggregation (§2.2) is:
```
Bump(C) = max_{c ∈ C} bump(c),  C = the commit set since the last release
```
**Proof this is well-defined and monotone.** `(Level, ≤)` is a totally ordered finite set, hence a complete lattice under `max`; `Bump` is therefore the *join* (least upper bound) of `{bump(c) : c ∈ C}` in this lattice — it always exists and is unique regardless of commit ordering (commutativity + associativity of `max`). Monotonicity: adding any commit `c'` to `C` can only raise or hold `Bump(C ∪ {c'}) ≥ Bump(C)`, never lower it — this is the formal guarantee that "you can never accidentally under-bump by adding more commits," only by *missing* a commit's true classification (see below).

**Classifier error and its consequence.**

Let `bump*(c)` be the *true* required bump (as determined by the API-diff ground truth of §7.1) and `bump(c)` the *commit-message-derived* estimate. A commit is **under-classified** when `bump(c) < bump*(c)` — e.g. a contributor wrote `fix:` for a change that in fact removed a public field (true bump = MAJOR, estimated = PATCH). Because `Bump(C) = max bump(c)`, a single under-classified commit silently caps the whole release below its true required level — the aggregation's `max` cannot "average out" one bad label, and no amount of correctly-labeled `feat:`/`fix:` commits recovers a missed `!`. This is why §7.1 mandates API-surface diffing as an independent, mechanical cross-check rather than trusting commit messages alone: model this as two independent estimators `bump_commit(c)` and `bump_diff(c)`; production release gates should compute `Bump_final = max(Bump_commit(C), Bump_diff(C))`, since the true bump can never be *lower* than what either channel independently proves.

**Empirical miss-rate bound.** If commit-message discipline yields a per-commit breaking-change flag recall of `r` (probability a true breaking change is correctly marked `!`/`BREAKING CHANGE:`), then over `n` independent breaking commits in a release, the probability that **at least one** breaking change is silently missed by the message-only channel is:
```
P(≥1 missed) = 1 - r^n
```
For `r = 0.90` (a well-disciplined team) and `n = 3` breaking changes in one release, `P(≥1 missed) = 1 - 0.9³ = 1 - 0.729 = 0.271` — over one in four multi-breaking-change releases will under-bump on commit messages alone. This is the quantitative justification for treating §7.1 (API diffing) as mandatory, not optional tooling, whenever `n` is expected to exceed 1.

### M3: Changelog Generation as Set Partitioning + Similarity Clustering + Topological Ordering

**Partitioning.** The commit set `C` since the last tag is partitioned by `type(c)` into disjoint buckets `C_feat, C_fix, C_breaking, C_other` such that `⋃ C_i = C` and `C_i ∩ C_j = ∅` — a straightforward equivalence-class partition induced by the classifier `type: Commit → {feat, fix, ..., breaking}`.

**Deduplication via Jaccard similarity.** Two commit descriptions `c_i, c_j` are near-duplicates (e.g. a fix committed, then re-worded in a follow-up commit) when their token sets are highly overlapping:
```
J(c_i, c_j) = |tokens(c_i) ∩ tokens(c_j)| / |tokens(c_i) ∪ tokens(c_j)|
```
Cluster commits into changelog-entry groups using single-linkage clustering with threshold `θ` (typically `θ = 0.6`): two commits merge into the same entry if `J(c_i, c_j) ≥ θ`, and clusters merge transitively. Within each merged cluster, keep the *most recent* description (assume later commits refine/correct earlier ones) or the one referencing the highest-numbered PR (assume it is the final, reviewed wording). This bounds changelog entry count to `|clusters| ≤ |C|` and typically `|clusters| ≪ |C|` for iterative PRs with fixup commits.

**Topological ordering by scope dependency.** When commits carry a `scope` (e.g. `feat(auth): ...`, `fix(db): ...`), and scopes have a known dependency relation (e.g. `auth` depends on `db` — changes to `db` should read before changes to `auth` that build on them), build a scope DAG `G = (Scopes, →)` where `s_1 → s_2` means "s_1 changes should be read before s_2 changes." A valid changelog ordering within a bucket is any topological sort of `G` (Kahn's algorithm, `O(|Scopes| + |edges|)`); if `G` has a cycle, fall back to a total order that is well-defined and stable but semantically arbitrary — e.g. alphabetical by scope — and flag the cycle for the maintainers to resolve (two scopes both claiming to be prerequisites of each other is itself a signal of an architecture issue).

**Changelog information density.** A degenerate changelog generator that emits one entry per raw commit (no clustering) has redundancy proportional to `1 - |clusters|/|C|`; treat the Shannon entropy of the *type* distribution as a quality signal:
```
H(types) = - Σ_t p(t) log₂ p(t),   p(t) = |C_t| / |C|
```
A changelog dominated by one type (e.g. `H ≈ 0`, all `chore:`) is a candidate for suppression entirely (§3.1 rule: `chore`/`ci`/`test` commits are typically omitted from user-facing changelogs) rather than inclusion — low type-entropy plus low reader-relevance is the signal to drop a bucket from the rendered output.

### M4: Monorepo Version Propagation as Dependency-Graph Reachability

**Model.** Represent the monorepo as a directed graph `G = (P, E)` where `P` is the set of packages and `(A, B) ∈ E` means "B depends on A" (edge points from dependency to dependent, i.e. the direction change *propagates*). A changeset assigns each directly-changed package `p` a bump level `Bump(p) ∈ {NONE, PATCH, MINOR, MAJOR}` per M2.

**Propagation rule.** For every edge `(A, B) ∈ E`, if `Bump(A) ≥ MINOR` (A's public surface changed in a way that could affect consumers) **or** `Bump(A) = MAJOR` and `B`'s declared range on `A` no longer contains the new version (per M1's interval check `R_B(v_A')`), then `B` must receive at least a `PATCH` bump (its manifest now points at a different `A`, which is itself a change to `B`'s build reproducibility even if `B`'s own code is untouched):
```
Bump'(B) = max( Bump(B), effective_propagated_bump(A → B) )
effective_propagated_bump(A → B) = PATCH   if range_still_satisfied(R_B, v_A')
                                  = Bump(A) if NOT range_still_satisfied(R_B, v_A')  (range break cascades in full)
```
**Algorithm — cascading closure via reverse-topological BFS.** Because propagation flows strictly along edges (a change to A can affect B, C, ... that depend on it, and transitively their dependents), the full set of packages requiring *some* bump is the forward reachable set from the directly-changed packages `Δ ⊆ P`:
```
Affected = ⋃_{p ∈ Δ} Reach(p),   Reach(p) = {q ∈ P : ∃ path p → q in G}
```
computed by a single multi-source BFS/DFS from `Δ` in `O(|P| + |E|)` — no repeated single-source traversals needed. The *level* each affected package receives is computed by processing `P` in **topological order** (Kahn's algorithm, `O(|P|+|E|)`, requires `G` to be a DAG — a circular inter-package dependency is a build error independent of versioning and must be rejected upstream) so that by the time package `B` is visited, every dependency `A` of `B` has already had its final `Bump'(A)` resolved, making `effective_propagated_bump(A → B)` well-defined without recomputation.

**Fixed (lockstep) versioning as a graph quotient.** Lockstep versioning (§4.1) is equivalent to collapsing the entire package graph into a single super-node before applying M2's aggregation: `Bump_lockstep = max_{p ∈ P} Bump(p)` — every package inherits the release's single global bump regardless of whether it individually changed. This is why lockstep monorepos see MAJOR-bump frequency scale with `|P|` (more packages ⇒ higher probability *some* package has a breaking change in any given release window), a direct consequence of the `max` aggregation over a larger set — independent versioning avoids this scaling because each package's `Bump(p)` is evaluated only over changes that actually touch `p` or its dependency closure.

### M5: Release Risk — Non-Homogeneous Poisson Process (Reliability Growth) + Sequential Rollback Test

**Post-release defect discovery as an NHPP.**

Model the cumulative number of defects discovered by time `t` after release as a non-homogeneous Poisson process with intensity function `λ(t)`. The **Crow-AMSAA (power-law) reliability growth model** is the standard fit:
```
Λ(t) = λ_0 · t^β        (expected cumulative defects by time t)
λ(t) = dΛ/dt = λ_0 · β · t^(β-1)     (instantaneous defect-discovery rate)
```
- `β < 1`: reliability is *improving* over the observation window (discovery rate decelerating — typical of a maturing release as easy bugs are found and fixed).
- `β = 1`: constant-rate (homogeneous) Poisson process — no growth or decay, defects surface uniformly.
- `β > 1`: reliability *degrading* (discovery rate accelerating) — a red flag that the release is unstable and rollback should be strongly weighted.

**Parameter estimation (MLE) from observed defect-discovery times `t_1 < t_2 < ... < t_n` up to test time `T`:**
```
β̂ = n / [ Σ_{i=1}^{n} ln(T / t_i) ]
λ̂_0 = n / T^β̂
```
This is the standard Crow-AMSAA MLE (derivable by maximizing the NHPP log-likelihood `ℓ(λ_0,β) = n ln(λ_0) + n ln(β) + (β-1) Σ ln(t_i) - λ_0 T^β` and solving `∂ℓ/∂β = 0` for `β̂`, then substituting into `∂ℓ/∂λ_0 = 0`).

**Mean Time Between Failures (MTBF) at time t:** `MTBF(t) = 1/λ(t) = t^(1-β) / (λ_0 β)`. A release-health dashboard should track `β̂` on a rolling window post-deploy: a `β̂` trending above 1 within the first canary window is the quantitative trigger for automated rollback, independent of any single incident's severity.

**Canary rollout as a Sequential Probability Ratio Test (SPRT).**

Rather than waiting for a fixed sample size, compare the canary's error rate `p_canary` against the baseline `p_0` (previous version's known-good rate) using Wald's SPRT, which minimizes expected sample size for a given false-accept/false-reject rate:
```
Hypotheses:  H₀: p = p_0 (canary as healthy as baseline)
             H₁: p = p_1 (canary degraded, p_1 > p_0, the minimum regression worth detecting)

Log-likelihood ratio after n requests with k errors:
  Λ_n = k · ln(p_1/p_0) + (n-k) · ln((1-p_1)/(1-p_0))

Decision boundaries (α = false-rollback rate, β_err = false-promote rate):
  A = ln(β_err / (1-α))      B = ln((1-β_err) / α)

  Λ_n ≤ A  →  ACCEPT H₀ → promote canary (stop early, no regression detected)
  Λ_n ≥ B  →  ACCEPT H₁ → automated rollback (stop early, regression confirmed)
  A < Λ_n < B → continue sampling (canary stays at current traffic %, keep collecting)
```
SPRT's key advantage over a fixed-window canary check (e.g. "watch for 30 minutes") is that it reaches a decision in the *minimum expected number of samples* for the target error rates — a badly regressed canary triggers rollback in seconds under high traffic, while a genuinely healthy canary is promoted without waiting out an arbitrary fixed window. For `p_0 = 0.001` (0.1% baseline error rate), `p_1 = 0.005` (5× regression threshold), `α = β_err = 0.05`: `A ≈ ln(0.05/0.95) = -2.944`, `B ≈ ln(0.95/0.05) = 2.944` — symmetric bounds, and each error event contributes `ln(0.005/0.001) = 1.609` toward `B` while each success contributes `ln(0.995/0.999) ≈ -0.004` toward `A`, so roughly 2 error events in a short window already crosses the rollback boundary while thousands of clean requests are needed to *positively* confirm health (the asymmetry is intentional — rollback should be cheap to trigger, promotion should require real evidence).

### M6: Release Cadence & Pipeline Throughput — Queueing Theory + Little's Law

**Release pipeline as an M/M/1 queue.**

Model the release pipeline (build → test → security scan → deploy) as a single-server queue: releases arrive (are triggered) at rate `λ` (releases/day the org *wants* to ship) and are processed at rate `μ` (releases/day the pipeline *can* fully clear, bounded by the slowest mandatory gate). Utilization `ρ = λ/μ` must satisfy `ρ < 1` for a stable (non-diverging) queue. Expected number of releases in the pipeline (queued + in-flight) and expected wait time:
```
L = ρ / (1-ρ)                    (Little's Law: L = λ · W)
W = L/λ = 1 / (μ - λ)            (expected total time a release spends in the pipeline, arrival to ship)
```
As `ρ → 1` (release cadence approaches pipeline capacity), `W → ∞` — this is the formal reason "just ship more often" fails once cadence nears the slowest gate's throughput: queueing delay grows *nonlinearly*, not linearly, as utilization approaches saturation. A pipeline with 4-hour mean clear time (`μ = 6/day`) run at `λ = 5/day` (`ρ = 0.833`) has `W = 1/(6-5) = 1` day mean time-in-pipeline — nearly 5× the raw 4-hour processing time, purely from queueing, well before `ρ` reaches 1.

**Little's Law as the general release-lead-time identity.** Independent of the queueing-discipline assumption above, Little's Law `L = λW` holds for *any* stable system (no distributional assumptions required) and is the correct tool for measuring "release lead time" empirically: count the average number of open release candidates in flight (`L`, directly observable from CI dashboards) and the release arrival rate (`λ`, releases/week merged to the release branch), and lead time falls out as `W = L/λ` — this lets teams estimate lead-time impact of a process change (e.g. adding a mandatory manual QA sign-off) by observing its effect on `L` without needing to model the new step's service-time distribution explicitly.

**Batch size economics (continuous delivery vs. batched releases).**

Trade-off: smaller release batches reduce **per-release risk** (fewer commits = smaller blast radius if something is wrong, faster MTTR since the diff to bisect is smaller) but increase **per-release fixed overhead** (changelog generation, tagging, deployment ceremony, canary wait time — a cost paid per release regardless of batch size). Model total cost per unit time as an Economic-Order-Quantity-style trade-off:
```
Cost(b) = (D/b)·K + (b/2)·H
  D = total change volume per period (commits/week)
  b = batch size (commits per release)
  K = fixed cost per release (pipeline run + ceremony, ~constant)
  H = holding cost per unit of undeployed change per unit time (risk exposure of code sitting unreleased)

Optimal batch size (minimize Cost(b), set dCost/db = 0):
  b* = √(2DK / H)
```
This is the direct analogue of the classical EOQ inventory formula, and it formalizes the intuition behind trunk-based development with small, frequent releases: as `H` (the cost of holding unreleased risk) rises relative to `K` (the fixed ceremony cost of releasing) — which is exactly what release automation (§6) accomplishes, by driving `K → 0` — the optimal batch size `b* → 0`, i.e. release-on-every-merge becomes cost-optimal once the release *ceremony* is fully automated. Teams that still batch into weekly release trains despite full automation are paying `H` unnecessarily; teams that release every commit without any automation are paying `K` unnecessarily. §6's entire premise (semantic-release / release-please) is minimizing `K` so that `b* → 1 commit`.

---

## Anti-Patterns to Avoid

- **Choosing lockstep (fixed) versioning for a monorepo with independent packages**: §4.1's stated con is explicit — a one-line fix in a leaf package forces a MAJOR release across the entire repo if any package has a breaking change, even when most packages are untouched; independent versioning avoids this coupling at the cost of tracking N version numbers.
- **Trusting Conventional Commits' `!`/`BREAKING CHANGE:` marker as proof of a breaking change**: §7.1 states the API-surface diff is ground truth and the commit marker is only a human-written hint the diff can disagree with — computing the version bump from commit messages alone when the actual public API surface hasn't been diffed risks shipping a MAJOR-worthy change under a MINOR tag, or vice versa.
- **Moving or re-pushing an existing git tag**: §5.3 states this explicitly as something to never do — a published version is immutable by SemVer contract; the correct fix for a bad release is a new PATCH, not rewriting history on the old tag.
- **Using lightweight tags for releases instead of annotated tags**: §5.3 requires annotated tags (carrying tagger identity, timestamp, and message) because they're required for supply-chain provenance — a lightweight tag doesn't carry the metadata SLSA-style provenance verification depends on.
- **Cutting the release gate before the reliability gate clears**: §6.4 states the release gate must run strictly after Phase E (RS=1.0) — minting a version number for an artifact that hasn't cleared the reliability gate turns the version into a promise to consumers about something that hasn't actually been verified.
- **Treating a rollback as a version-mechanics action**: §8.2 explicitly forbids re-tagging or deleting a bad release's version number during rollback — rollback is a deployment action (traffic switch or redeploy of the previous tag); the fix ships as a new PATCH/MAJOR, never by erasing the bad version's history.
- **Removing a deprecated API without completing the deprecation window**: §7.2 requires maintaining a deprecated API for at least one full MAJOR cycle (or the project's published window) before removal — skipping straight to removal in the same or next release breaks the deprecation contract consumers were told to expect.
- **Forgetting to cascade a PATCH bump to downstream dependents when an upstream package bumps MAJOR**: §4.3's cascading propagation rule means any package whose dependency range no longer matches the new MAJOR needs at minimum a PATCH bump to update its manifest pointer — skipping this leaves downstream packages referencing a version range that silently excludes the new release.
- **Using fully-automated continuous release (semantic-release) in environments where release timing must be deliberate**: §6.2 notes release-please's human-approval-gate model exists specifically for regulated environments and App Store review windows — defaulting to fully-automated publish-on-merge in a context that needs deliberate timing removes a control the situation actually requires.
- **Burying a personal-data-processing change inside a generic changelog entry**: §9's DPDP Act 2023 §4/§8 rule requires release notes to flag such changes distinctly (e.g. a dedicated "Data Processing Changes" heading) so they route to the DPO/consent-notice workflow — a change described only as a generic "improvement" bullet doesn't trigger the compliance routing it needs.

## Response Rules

1. Always determine the version bump from BOTH the commit-message classifier (§2, M2) AND the API-surface diff (§7.1) — never trust commit messages alone when `n > 1` breaking-candidate commits exist in the release window.
2. Always generate the changelog mechanically from commit history first (§3.2), then curate release notes as a separate, human/LLM-editorial pass (§3.3) — never hand-write the changelog from scratch.
3. For monorepos, always identify whether the project uses independent or lockstep versioning (§4.1) before recommending a bump strategy — the propagation math (M4) differs materially between the two.
4. Recommend annotated + signed tags (§5.3) for every release, never lightweight tags — flag any existing lightweight-tag release history as a supply-chain gap.
5. Never recommend re-tagging or force-pushing an existing version tag — always recommend a new PATCH/MAJOR release instead, per SemVer immutability (§1.3).
6. Place the release gate strictly after the Reliability Gate (RS = 1.0) and Security Gate (0 findings) in any CI/CD pipeline design — cite §6.4's gate ordering.
7. For any release involving a canary or staged rollout, recommend an explicit statistical stopping rule (SPRT, §M5) rather than a fixed time-window "wait and see" — cite the specific `p_0`, `p_1`, `α`, `β_err` the team should set.
8. When a release changes personal-data processing, always flag it as a distinct changelog/release-notes section (§9 DPDP layer) rather than folding it into general "Changed" entries.
9. When recommending release cadence, quantify the trade-off with the batch-size formula (M6) rather than giving an unsupported "release more often" recommendation.
10. Always distinguish "changelog" (durable, mechanical, append-only) from "release notes" (curated, audience-facing) in every recommendation — never suggest they be the same artifact for a public-facing project.

---

## What Not to Do

1. Do NOT recommend manual version-number editing in `package.json`/`Cargo.toml`/`pyproject.toml` when commit-driven automation (§6) is available — manual edits are the primary source of SemVer-precedence bugs.
2. Do NOT recommend lightweight (unannotated, unsigned) git tags for releases — always annotated + signed (§5.3).
3. Do NOT recommend treating `refactor:`/`perf:`/`chore:` commits as automatically bump-worthy without an explicit project policy decision — this is not part of the Conventional Commits spec and silently inflates release frequency.
4. Do NOT recommend skipping the API-surface diff (§7.1) for library/SDK projects, even when commit-message discipline is high — cite M2's quantitative miss-rate bound (`1 - r^n`) as the justification.
5. Do NOT recommend a fixed-duration canary wait ("bake for 30 minutes") when a sequential test (§M5 SPRT) can reach the same confidence with a smaller expected sample and faster rollback on genuine regressions.
6. Do NOT recommend lockstep monorepo versioning without disclosing the MAJOR-bump frequency scaling consequence (M4) — this is a real operational cost, not a stylistic choice.
7. Do NOT recommend suppressing the `[Unreleased]` section of a Keep a Changelog file — it is the audit trail for in-flight work between releases.
8. Do NOT recommend deleting or rewriting changelog history for past releases — the changelog is an append-only record; corrections go in a new entry, not a retroactive edit.
9. Do NOT recommend release automation (semantic-release/release-please) without also wiring the release gate after Security + Reliability gates (§6.4) — automating the *mechanics* of release without automating the *safety checks* first just ships badly-gated releases faster.
10. Do NOT ignore the India regulatory layer (§9) for regulated Indian financial/government systems — an undocumented rollback is a CERT-In §3(v) compliance gap, not just an engineering oversight.

---

## Output Expectations

When responding to release/versioning queries, deliverables should include:

1. **Version Bump Determination** — the computed `Bump(C)` (§2.2/M2) with the specific commits driving MAJOR/MINOR/PATCH, cross-checked against an API-surface diff when applicable (§7.1)
2. **Changelog Entry** — a Keep a Changelog-formatted block (§3.1) ready to prepend to `CHANGELOG.md`, generated from the actual commit range, not hand-written prose
3. **Tagging Command** — the exact `git tag -a -s ...` (or cosign-equivalent) invocation for the computed version
4. **Monorepo Propagation Report** (when applicable) — the affected-package set (M4 `Affected`) with each package's resolved bump level and the reason (direct change vs. propagated)
5. **Release Pipeline Stage Definition** — CI/CD YAML/config for the release gate, placed after Security + Reliability gates (§6.4)
6. **Rollout/Rollback Plan** — canary stage percentages with the specific SPRT parameters (`p_0, p_1, α, β_err`) or blue-green cutover steps (§8)
7. **India Compliance Note** (when applicable) — which regulatory layer (§9) the release trail must satisfy and what evidentiary artifact covers it

---

## Skill Scope

**In Scope:**
- SemVer 2.0.0 precedence, range operators, and version-range constraint satisfaction (version solving)
- Conventional Commits parsing and automated bump-level classification
- Changelog generation (Keep a Changelog format) and release-notes curation
- Monorepo/multi-package independent vs. lockstep versioning and dependency-graph bump propagation
- Git branching, tagging, and tag-signing strategy for releases
- Release automation tooling (semantic-release, release-please, goreleaser, Changesets) and pipeline gate placement
- API/CLI/schema breaking-change detection and deprecation-window policy
- Release risk modeling: reliability growth (Crow-AMSAA/NHPP), canary rollout statistics (SPRT), rollback strategy
- Release cadence and pipeline-throughput economics (queueing theory, Little's Law, batch-size trade-offs)
- India regulatory layer for release/change-management audit trails (CERT-In, RBI, SEBI CSCRF, DPDP Act 2023)

**Out of Scope:**
- Dockerfile/Kubernetes manifest authoring (see `docker`, `kubernetes`)
- CI/CD pipeline syntax mechanics beyond the release stage (see `github-actions-ci`, `jenkins-pipeline`)
- SBOM generation, supply-chain attestation, and container signing internals (see `devsecops-core`)
- Cloud infrastructure provisioning for blue-green/canary environments (see `cloud-security-core`, cloud-engineer)
- General shell/CLI scripting mechanics (see `linux-shell-scripting`, `powershell-scripting`)
- Feature-flag/experimentation platform design (progressive delivery mechanics beyond the canary rollout decision rule in §8/M5)

---

## Version

1.0.1 — Added Anti-Patterns to Avoid: lockstep versioning coupling, commit-marker-trusted-over-API-diff, tag rewriting, lightweight tags, pre-reliability-gate release, rollback-as-version-action, incomplete deprecation window, missed cascading PATCH bump, over-automated release timing in regulated environments, buried DPDP-relevant changelog entries

1.0.0 — Initial release: 9 topic sections covering SemVer 2.0.0, Conventional Commits bump automation, changelog generation (Keep a Changelog), monorepo/multi-package versioning (Changesets/Lerna/Nx), git tagging strategy, release automation (semantic-release/release-please/goreleaser), API breaking-change detection, release risk/rollback, and India regulatory layer (CERT-In/RBI/SEBI CSCRF/DPDP Act 2023). M1-M6 mathematical foundations with full derivations: SemVer total-order + PubGrub version-solving CSP, monotone bump-aggregation lattice + miss-rate bound, changelog set-partitioning + Jaccard clustering + topological ordering, monorepo dependency-graph propagation via topological BFS, Crow-AMSAA NHPP reliability growth + Wald SPRT canary testing, M/M/1 release-pipeline queueing + Little's Law + EOQ batch-size economics.
