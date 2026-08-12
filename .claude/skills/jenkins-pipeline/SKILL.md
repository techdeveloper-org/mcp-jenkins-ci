---
name: jenkins-pipeline
version: 2.0.0
last-updated: 2026-03-23
description: "Jenkins Pipeline Orchestrator for CI/CD pipelines with Docker/K8s integration. Use for \\"Jenkins pipeline\\", \\"Jenkinsfile\\", \\"CI/CD Jenkins\\", \\"deploy Jenkins Docker/K8s\\". Use when designing, writing, or troubleshooting Jenkins CI/CD pipelines. Keywords: write Jenkinsfile, Jenkins declarative pipeline, Jenkins shared library, Jenkins agent, build stage, deploy stage, Jenkins Docker integration."
allowed-tools: Read,Glob,Grep,Bash,Edit,Write
user-invocable: true
---


<!-- generated-by: claude-global-library/tools/project_sync -->
<!-- source: skills/jenkins-pipeline/SKILL.md -- edit the library, then re-run sync_project.py -->

# Skill: Jenkins Pipeline Orchestrator – DevOps Engineer

## Description
This skill enables the agent to act as a **Jenkins Pipeline Orchestrator**, responsible for designing, generating, and maintaining CI/CD pipelines using Jenkins.

The skill **coordinates and invokes**:
* Docker Expert skill (image build, optimization, security)
* Kubernetes Expert skill (deployment, scaling, rollout strategies)

The agent follows an **interactive workflow**, asking the user to select deployment targets, environments, and configurations before finalizing pipelines.

---

## Mathematical Foundations

### M1: Pipeline Parallelism and Critical Path
```
Sequential pipeline duration:
  T_sequential = Σ(stage_duration)

Parallel pipeline duration (Amdahl's Law applied):
  T_parallel = T_critical_path + T_serial_overhead
  Speedup = T_sequential / T_parallel

  Critical path: longest chain of dependent stages
  Example: build(2m) → [test(5m) ∥ lint(1m)] → deploy(1m)
    T_sequential: 2+5+1+1 = 9 min
    T_parallel: 2 + max(5,1) + 1 = 8 min (only 11% speedup → test is bottleneck)
    To improve: split test stage into parallel shards

Test sharding:
  n test shards: T_tests = total_test_time / n (if perfectly balanced)
  Optimal shards: n* = floor(total_test_time / target_stage_time)
  Real-world efficiency: 70–80% (imbalanced tests, setup overhead)
```

### M2: Build Artifact Cache Hit Rate
```
Docker layer cache hit rate in Jenkins:
  P(cache_hit) per layer = P(layer_content_unchanged)
  Average cache hit for code layers: 0.3–0.7 (changes per commit)
  Average cache hit for dep layers: 0.8–0.95 (deps change less often)

Build time with cache:
  T_cached = Σ (P(miss_i) × T_build_layer_i) + Σ (P(hit_i) × T_restore_i)
  T_restore ≈ 0.1 × T_build (cache restore ~10× faster than rebuild)

Workspace cache (node_modules, .gradle, .m2):
  First build: O(download_size / bandwidth)
  Cached build: O(verification_time) ≈ 2–5s vs 60–300s download
  Cache storage: dependency_size × n_concurrent_builds
```

### M3: Executor Slot Allocation Math
```
Jenkins executor utilization:
  utilization = Σ(build_time) / (n_executors × total_time)
  P(queue_wait) increases when utilization > 0.7

Queue wait time (M/M/c queueing model):
  λ = build arrival rate (builds/hour)
  μ = build completion rate (1/avg_build_time)
  ρ = λ / (c × μ)  where c = executors

  P(queue_wait > 0) = (ρ^c / c!) × P0 / (1 - ρ/c)  (Erlang C formula)

  At ρ=0.8 (80% utilization): expected queue time ≈ 4× avg_service_time
  At ρ=0.5 (50% utilization): expected queue time ≈ 0.5× avg_service_time

Recommended: keep executor utilization < 70% for responsive CI
```

### M4: Flakiness Detection Statistics
```
Flaky test identification:
  Run test n times; flag as flaky if: 0 < failure_count < n
  Statistical confidence: binomial test
    P(observe k failures | true_failure_rate=p) = C(n,k) × p^k × (1-p)^(n-k)

  Minimum reruns to detect flakiness at 99% confidence:
    For p=0.1 (10% flaky): need n=44 reruns to detect with 99% confidence
    For p=0.3 (30% flaky): need n=13 reruns

  Practical threshold: flag as flaky if fails > 1 in 10 identical runs

Build failure rate SLO:
  P(false positive alert) = P(build fails due to infra, not code)
  Target: infra_failure_rate < 1% of total build failures
  Track separately: code_failures vs infra_failures vs flaky_test_failures
```

### M5: Jenkins Build Queue Optimization Mathematics

**Step 1 — Little's Law for Jenkins build throughput.**
Little's Law (L = λW) applied to the Jenkins build queue:
```
L = λ × W

  L = average number of builds in system (queue + executing)
  λ = average build arrival rate (builds/hour)
  W = average time a build spends in system (queue_wait + execution_time)

Decomposition:
  L = L_q + L_s  (queue + in-service)
  W = W_q + E[S] (waiting time + service time)

  L_q = λ × W_q  (Little's Law for queue portion only)

Example capacity planning:
  λ = 30 builds/hour (one build per 2 minutes average)
  E[S] = 10 minutes average build time
  Target W ≤ 15 minutes (queue_wait ≤ 5 min, build time = 10 min)

  L_target = 30/60 × 15 = 7.5 builds in system at any time
  Minimum executors: c ≥ ceil(λ × E[S]) = ceil(30/60 × 10) = 5 executors
```

**Step 2 — Executor utilization formula (M/M/c model).**
Jenkins executor pool modeled as M/M/c queue:
```
Traffic intensity per server: ρ = λ / (c × μ)
  where λ = arrival rate, μ = 1/E[S] = service rate, c = executor count

System utilization: ρ_system = λ / (c × μ) < 1 (stability condition)

Erlang C formula for P(queue_wait > 0):
  P_0 = [Σ_{k=0}^{c-1} (cρ)^k/k! + (cρ)^c / (c! × (1-ρ))]^{-1}
  P(wait) = [(cρ)^c × P_0] / [c! × (1-ρ)]   (Erlang C formula)

  Expected queue wait: W_q = P(wait) / (c × μ × (1-ρ))

Utilization vs queue wait relationship:
  ρ = 0.50: W_q ≈ 0.5 × E[S] / c   (low contention)
  ρ = 0.70: W_q ≈ 1.2 × E[S] / c   (moderate — recommended max)
  ρ = 0.80: W_q ≈ 2.0 × E[S] / c   (noticeable queue buildup)
  ρ = 0.90: W_q ≈ 4.5 × E[S] / c   (heavy contention — avoid)
  ρ → 1.0:  W_q → ∞                 (queue grows unboundedly)

Recommended: maintain ρ_system ≤ 0.70 for responsive CI
  Required executors: c ≥ ceil(λ / (0.70 × μ)) = ceil(λ × E[S] / 0.70)
  For λ=30/hr, E[S]=10min: c ≥ ceil(30 × 10/60 / 0.70) = ceil(7.14) = 8 executors
```

**Step 3 — Pipeline stage parallelization efficiency formula.**
Jenkins `parallel` block efficiency with stage dependencies:
```
T_pipeline_parallel = T_critical_path + T_sequential_stages + T_agent_overhead

  T_critical_path = max(duration of longest parallel branch)
  T_agent_overhead: agent startup time per parallel branch
    Docker agent: 5–30s (image pull if not cached + container start)
    Static agent: 0–2s (already running, just assignment overhead)

Parallelization efficiency:
  η = T_sequential / (T_parallel × n_parallel_branches)
  Perfect: η = 1.0 (no overhead, perfectly balanced)
  Typical: η = 0.65–0.85 (agent overhead + imbalanced test distribution)

Test stage sharding efficiency:
  T_test_sharded = total_test_time / n_shards + T_shard_setup
  Optimal n_shards = ceil(total_test_time / (target_stage_time - T_shard_setup))

  For total_test_time=60min, target=10min, T_shard_setup=1min:
    n_optimal = ceil(60/(10-1)) = ceil(6.67) = 7 shards
    T_actual = 60/7 + 1 = 9.57min ≈ target ✓

Pipeline speedup vs sequential:
  S = T_sequential / T_parallel = (sum all stages) / (critical path + overhead)
  For build(2m) → [test_shard×7(10m each) ∥ lint(3m)] → deploy(1m):
    T_sequential = 2 + 7×10 + 3 + 1 = 76 min
    T_parallel   = 2 + max(10, 3) + 1 = 13 min
    S = 76/13 = 5.85× speedup via parallelization
```

---

### M6: India CERT-In CI/CD Security Requirements and Compliance Mathematics

**Step 1 — CERT-In CI/CD pipeline audit score formula.**
CERT-In (Indian CERT) April 2022 circular mandates cybersecurity controls for all ICT systems:
```
CERT-In_CI_CD_score = Σ_control weight_i × compliance_i

Critical security controls (weight = 15 each):
  [+15] No hardcoded credentials in Jenkinsfile (scanned by trufflehog/gitleaks)
  [+15] Jenkins master accessed via HTTPS + mTLS (encrypted management plane)
  [+15] Build artifacts signed with verified PKI certificates (supply chain integrity)

High security controls (weight = 10 each):
  [+10] Jenkins plugins updated monthly (CERT-In patching SLA: CRITICAL within 3 days)
  [+10] Build logs retained ≥ 180 days (CERT-In 2022 mandatory log retention)
  [+10] RBAC on Jenkins (least-privilege for pipeline execution vs configuration)

Medium controls (weight = 5 each):
  [+5]  Secrets in Jenkins Credentials Store (not environment variables)
  [+5]  Build agent isolation (separate agent per project, no shared workspace)
  [+5]  SAST scan in pipeline (SonarQube or equivalent — CERT-In recommends)
  [+5]  Container image scan (trivy/Clair) before push to registry

Score calculation:
  Max_score = 3×15 + 3×10 + 4×5 = 45 + 30 + 20 = 95
  CERT-In compliance gate: CERT-In_CI_CD_score ≥ 76 (80% of max_score)
```

**Step 2 — STQC software audit pipeline standards.**
STQC Directorate (under MeitY) evaluates software quality for government projects:
```
STQC pipeline gate formula:
  All stages must be present and pass for STQC clearance:

  Stage 1: Static Analysis (SCA + SAST)
    SonarQube quality gate: code_coverage ≥ 80%, no CRITICAL bugs, tech_debt ≤ 5h
    SAST_gate = (critical_issues = 0) AND (coverage ≥ coverage_threshold)

  Stage 2: Automated Testing
    test_pass_rate = passed_tests / total_tests ≥ 0.98
    regression_coverage = regression_test_count / feature_count ≥ 1.0 (≥1 test per feature)

  Stage 3: DAST (Dynamic Application Security Testing)
    OWASP_ZAP_scan_gate = HIGH_vulns = 0 AND MEDIUM_vulns ≤ acceptable_threshold
    STQC_DAST_threshold: MEDIUM ≤ 5 with documented acceptance

  Stage 4: Performance Testing
    load_test_SLA: P95_response ≤ SLA_threshold under target_load TPS
    STQC_perf_gate = P95 ≤ SLA AND error_rate ≤ 0.01 (1%)

  STQC_pipeline_pass = Stage1_pass AND Stage2_pass AND Stage3_pass AND Stage4_pass
  Certification issued: STQC Test Report valid for 12 months (mandatory renewal)
```

**Step 3 — India banking sector Jenkins deployment compliance formula.**
RBI IT Master Directions (Jan 2024) for banks and NBFCs:
```
RBI_deployment_compliance = Σ_control compliance_i / total_controls

Mandatory controls for production Jenkins deployments in banking:
  [MANDATORY] Dual authorization: ≥2 approvers for production pipeline (Four-Eyes Principle)
    Jenkins implementation: input { message "Approve?"; submitter "approver1,approver2" }
    Gate: approval_count ≥ 2 before deploy stage executes

  [MANDATORY] Change Management integration: Jenkins → ITSM ticket (ServiceNow/Jira)
    Each production build must reference a valid approved CR (Change Request) number
    gate: CR_status == "APPROVED" AND CR_planned_time includes current_time

  [MANDATORY] Rollback capability: automated rollback pipeline defined
    rollback_time_SLA ≤ 30 minutes (RBI IT Circular §7.4 — recovery time objective)
    Jenkins rollback: deploy previous image tag if health check fails post-deploy

  [MANDATORY] Deployment window: production deploys only during approved maintenance windows
    Non-emergency: weekend 02:00–05:00 IST or pre-approved weekday window
    Emergency: P1 incident declared → expedited approval (single approver allowed)

  [MONITORING] Post-deployment validation: automated smoke test within 5 minutes
    smoke_test_pass_rate = passed_checks / total_smoke_checks ≥ 0.95
    Auto-rollback trigger: smoke_test_pass_rate < 0.95 OR error_rate > 0.02 within 10min

RBI_compliance_score = satisfied_mandatory_controls / 4 × 100
Minimum: RBI_compliance_score = 100% (all mandatory controls must be satisfied)
  Non-compliance: RBI can issue regulatory action under Banking Regulation Act §35A
```

---

## Responsibilities
The agent can:
* Design declarative Jenkins pipelines
* Orchestrate Docker and Kubernetes workflows
* Automate build, test, image, and deploy stages
* Ask and validate deployment decisions
* Enforce CI/CD best practices
* Support production-grade pipelines

---

## Core Workflow (Mandatory Interaction)

Before generating or modifying a pipeline, the agent MUST ask:

### 1️⃣ Deployment Target
Ask the user to choose one:
* **Docker (Standalone / VM / ECS / Swarm)**
* **Kubernetes (EKS / GKE / AKS / On-Prem)**

---

### 2️⃣ If Kubernetes is selected, ask:
* **Kubeconfig location**
  - Example: `~/.kube/config`
  - Or Jenkins credential ID
* **Kubernetes context / cluster name**
* **Namespace**
* **Deployment type**
  - Deployment
  - StatefulSet
  - DaemonSet
  - Job / CronJob
* **Deployment strategy**
  - RollingUpdate
  - Blue-Green
  - Canary

---

### 3️⃣ If Docker is selected, ask:
* Target environment:
  - VM
  - Bare metal
  - Cloud service
* Container runtime host access method:
  - SSH
  - Docker socket
* Image registry details

---

## Capabilities

### 1. Jenkins Pipeline Authoring
* Declarative `Jenkinsfile`
* Scripted pipeline (when required)
* Shared libraries usage
* Multi-branch pipelines
* Parameterized builds

---

### 2. Docker Integration (via Docker Expert Skill)
* Build Docker images using best practices
* Multi-stage builds
* Secure image scanning
* Image tagging and versioning
* Push to container registries

> Delegates Docker-specific logic to **Docker Expert skill**

---

### 3. Kubernetes Integration (via Kubernetes Expert Skill)
* Apply Kubernetes manifests
* Helm-based deployments
* Kustomize overlays
* Rollout monitoring
* Automated rollback handling

> Delegates cluster and workload logic to **Kubernetes Expert skill**

---

### 4. CI/CD Stages Supported
* Code checkout
* Static code analysis
* Unit and integration tests
* Docker image build
* Image security scan
* Deployment
* Post-deployment validation

---

### 5. Credentials & Secrets Management
* Jenkins credentials binding
* Secure handling of kubeconfig
* Registry authentication
* No hardcoded secrets in pipelines

---

### 6. Environment Awareness
* Dev / QA / UAT / Prod separation
* Environment-specific configs
* Approval gates for production
* Promotion-based deployments

---

## Inputs
The agent can accept:
* Source code repository
* Existing Jenkinsfiles
* Deployment requirements
* Target environment details
* Dockerfiles
* Kubernetes manifests or Helm charts

---

## Outputs
The agent may produce:
* Jenkinsfile (declarative)
* Pipeline stage breakdowns
* Credential setup instructions
* Deployment automation logic
* Rollback and recovery steps

---

## Tools & Technologies
* Jenkins
* Jenkins Shared Libraries
* Docker CLI
* kubectl
* Helm
* Git
* Container registries

---

## Constraints
* Always ask deployment target before pipeline generation
* Never assume Kubernetes or Docker implicitly
* Do not embed secrets in Jenkinsfiles
* Prefer declarative pipelines
* Pipelines must be reproducible and auditable

---

## Best Practices Followed
* CI/CD as code
* Least privilege access
* Immutable artifacts
* Shift-left security
* Fail-fast pipeline design

---

## Example Use Cases
* “Create a Jenkins pipeline for Spring Boot app”
* “Deploy this app using Docker”
* “Deploy to Kubernetes using Helm”
* “Ask me which cluster and namespace to deploy to”
* “Add blue-green deployment strategy”
* “Integrate Docker build and Kubernetes deploy”

---

## Skill Level
Expert

---

## Notes
* Docker-related tasks are delegated to **Docker Expert skill**
* Kubernetes-related tasks are delegated to **Kubernetes Expert skill**
* This skill acts as the **CI/CD control plane**

---

## 🔧 GLOBAL JENKINSFILE TEMPLATE (MANDATORY)

**Location:** `~/.claude/memory/templates/Jenkinsfile.spring-boot.template`

**CRITICAL:** ALWAYS use this template when creating Jenkinsfiles for Spring Boot microservices!

### Template Features:
- ✅ Local Docker registry (localhost:5000) - eliminates Docker Hub rate limiting
- ✅ Git SHA versioning for deployments
- ✅ Maven dependency caching (workspace/.m2)
- ✅ Linux-only commands (NO Windows support)
- ✅ Kubernetes deployment automation
- ✅ Namespace auto-creation
- ✅ Rollout status checking

### Variables to Replace:
```
{SERVICE_NAME}     → Full service name (e.g., "acme-store-customer-service")
{SERVICE_PORT}     → Port number (e.g., "8087")
{K8S_NAMESPACE}    → Kubernetes namespace (e.g., "acme-store")
{COMMON_UTIL_COPY} → Project-specific copy command for common utility JAR
```

### Usage:
```bash
# Read template
cat ~/.claude/memory/templates/Jenkinsfile.spring-boot.template

# Replace variables
sed -e 's/{SERVICE_NAME}/acme-store-customer-service/g' \
    -e 's/{SERVICE_PORT}/8087/g' \
    -e 's/{K8S_NAMESPACE}/acme-store/g' \
    ~/.claude/memory/templates/Jenkinsfile.spring-boot.template > Jenkinsfile

# Manually replace {COMMON_UTIL_COPY} based on project
```

### Common Utility Copy Commands by Project:

**Acme Store** (dynamic version from pom.xml):
```groovy
def commonUtilVersion = sh(
    script: "mvn help:evaluate -Dexpression=common-utility.version -q -DforceStdout", 
    returnStdout: true
).trim()
sh "cp ../acme-store-common-utility/target/acme-store-common-utility-${commonUtilVersion}.jar ."
env.COMMON_UTIL_VERSION = commonUtilVersion
```

**WebApp** (fixed version):
```groovy
sh 'cp ../webapp-common-utility/target/webapp-common-utility-1.0.0.jar .'
```

**API Service** (fixed version):
```groovy
sh 'cp ../api-service-common-lib/target/api-service-common-lib-1.0.0.jar .'
```

### NEVER:
- ❌ Use Windows-specific code (bat, isUnix())
- ❌ Use Docker Hub images directly (use localhost:5000)
- ❌ Skip Maven caching
- ❌ Hardcode image tags (use Git SHA)
- ❌ Skip namespace auto-creation

### Complete Guide:
`~/.claude/memory/templates/TEMPLATE-USAGE-GUIDE.md`

---

## Anti-Patterns to Avoid

- **Parallelizing a stage that isn't the critical-path bottleneck**: M1's worked example shows parallelizing test∥lint only achieves an 11% speedup because test dominates the critical path — investing sharding effort in a non-bottleneck stage doesn't move the overall pipeline duration; identify the critical path first, then shard the stage actually on it.
- **Running the executor pool above 70% utilization**: M3/M5 Step 2's Erlang C math shows queue wait grows from ≈1.2×E[S]/c at ρ=0.70 to ≈4.5×E[S]/c at ρ=0.90, and unboundedly as ρ→1.0 — provisioning executors for "average" load instead of the 70%-utilization target leaves no headroom for the queue-wait blowup that arrives well before 100% utilization.
- **Sizing the executor pool from the bare Little's Law minimum instead of the utilization-target formula**: `c ≥ ceil(λ × E[S])` (M5 Step 1) is the stability floor, not the responsive-CI target — the actual recommended count comes from `c ≥ ceil(λ × E[S] / 0.70)` (M5 Step 2), which is meaningfully higher (8 vs 5 executors in the worked example) and is what keeps queue wait from growing unboundedly.
- **Flagging a test as flaky after a single failure**: M4's binomial confidence math requires 44 reruns to detect a 10%-true-failure-rate flaky test at 99% confidence — a one-off failure carries far too little statistical evidence to label a test flaky; the practical threshold in this skill is failing more than 1 in 10 identical reruns.
- **Lumping infrastructure failures, code failures, and flaky-test failures into one build-failure metric**: M4 requires tracking these separately, with a target infra_failure_rate below 1% of total build failures — a combined metric hides whether CI infrastructure itself is degrading versus the code under test actually regressing.
- **Sharding tests without subtracting shard-setup overhead from the target**: M5 Step 3's optimal shard count is `ceil(total_test_time / (target_stage_time - T_shard_setup))`, not a naive division by target_stage_time alone — omitting the setup-overhead term under-shards and produces stage durations that overshoot the actual target.
- **Defaulting to Docker agents for every parallel branch regardless of branch count**: M5 Step 3 notes Docker agent startup is 5-30s versus 0-2s for an already-running static agent — for pipelines with many small parallel branches, the cumulative Docker startup overhead can rival or exceed the actual work being parallelized.
- **Storing secrets as Jenkins environment variables instead of the Credentials Store**: M6 Step 1 explicitly separates this as a distinct compliance control from generic secrets management — environment variables are visible in build logs and process listings in ways the Credentials Store is specifically designed to avoid.
- **Skipping dual authorization on banking-sector production deployments**: RBI's Four-Eyes Principle (M6 Step 3) is a MANDATORY control requiring ≥2 approvers before the deploy stage executes — RBI compliance requires 100% of mandatory controls satisfied, not a partial score; skipping this one control fails the entire compliance gate regardless of how well other controls are met.
- **Deploying to banking production outside an approved maintenance window without a declared P1 emergency**: M6 Step 3's deployment-window control only permits emergency deployment via expedited single-approver approval when a P1 incident is formally declared — an ad hoc "urgent but not P1" deployment outside the window doesn't meet this mandatory control's exception criteria.

## Response Rules

1. Always use Declarative Pipeline syntax (`pipeline {}`) — never Scripted Pipeline for new work
2. Always use Linux-only commands — never `bat`, `isUnix()`, or Windows-specific syntax
3. Always use `agent { label '...' }` or `agent any` — never leave agent unspecified
4. Always cache Maven dependencies: `MAVEN_OPTS` with workspace `.m2` directory
5. Always use Git SHA for Docker image tags — never use `latest` or branch name as image tag
6. Always create Kubernetes namespaces before deploying: `kubectl create namespace --dry-run -o yaml | kubectl apply -f -`
7. Always check rollout status after deploy: `kubectl rollout status deployment/...`
8. Always use local Docker registry (`localhost:5000`) — never pull directly from Docker Hub
9. Wrap all `sh` commands with proper exit code handling — failures must fail the stage

---

## What Not to Do

- Do not use Scripted Pipeline syntax (`node {}`) for new pipelines
- Do not use Windows-specific commands (`bat`, `cmd`, `isUnix()`)
- Do not use `latest` as Docker image tag — use Git SHA (`env.GIT_COMMIT`)
- Do not pull images directly from Docker Hub — use local registry
- Do not skip Maven dependency caching — cold builds waste 5–10 minutes
- Do not hardcode credentials in Jenkinsfile — use `credentials()` binding
- Do not skip namespace auto-creation — deployment will fail if namespace missing
- Do not ignore `kubectl rollout status` — pipeline must wait for successful rollout
- Do not store secrets in `env` declarations — use Jenkins Credentials Store

---

## Output Expectations

- Complete Jenkinsfile with all stages: Checkout, Build, Test, Docker Build, Docker Push, K8s Deploy
- Maven caching configuration with `MAVEN_OPTS` workspace path
- Docker build commands using Git SHA tagging and local registry
- Kubernetes deploy commands with namespace auto-creation and rollout status check
- Variable replacement instructions for `{SERVICE_NAME}`, `{SERVICE_PORT}`, `{K8S_NAMESPACE}`, `{COMMON_UTIL_COPY}`
- Post-build notifications (success/failure) with build URL and stage that failed
- For multi-branch: branch-specific deployment strategy (main→prod, develop→staging)

---

## Skill Scope

### In Scope
- Declarative Pipeline authoring: stages, steps, post, parameters, environment, options
- Maven build with dependency caching and test reporting
- Docker image build, tag, and push to local or remote registries
- Kubernetes deployment: namespace management, rolling deploy, rollout status verification
- Jenkins Shared Libraries: vars/, src/ structure, `@Library` annotation
- Credentials management: `withCredentials`, `credentials()` binding
- Pipeline triggers: cron, SCM polling, webhooks
- Multi-branch pipelines and branch-specific deploy strategies
- Pipeline debugging: `echo`, `sh returnStdout: true`, `catchError`

### Out of Scope
- Dockerfile authoring (use docker skill)
- Kubernetes manifest writing (use kubernetes skill)
- GitHub Actions workflows (use github-actions-ci skill)
- Cloud infrastructure provisioning

---

## Version
2.0.1 — Added Anti-Patterns to Avoid: non-bottleneck parallelization, over-70%-utilization executor pools, bare-minimum executor sizing, single-failure flaky verdicts, combined failure-rate metrics, unadjusted shard counts, Docker-agent overuse for small branches, env-var secrets, missing dual authorization, out-of-window banking deployments
2.0.0
