---
name: github-actions-ci
description: "Comprehensive guidance for GitHub Actions CI/CD workflows including syntax, triggers, jobs, secrets, caching, artifacts, Docker actions, reusable workflows, and security best practices. Use when setting up GitHub Actions workflows, CI/CD pipelines, automated testing, Docker builds, or reusable workflow patterns. Keywords: GitHub Actions workflow, CI pipeline, run tests on push, automate deployment, GitHub workflow YAML, on pull_request trigger."
allowed-tools: Read,Glob,Grep,Bash,Edit,Write
user-invocable: true
---


<!-- generated-by: claude-global-library/tools/project_sync -->
<!-- source: skills/github-actions-ci/SKILL.md -- edit the library, then re-run sync_project.py -->

# GitHub Actions CI/CD Skill Instructions

### Skill Name
github-actions-ci

### Description
This skill provides comprehensive guidance for designing and implementing GitHub Actions CI/CD pipelines. It covers workflow syntax, trigger configuration, job orchestration, secrets management, dependency caching, artifact handling, Docker-based actions, reusable workflows, and security hardening.

This skill applies to any project hosted on GitHub that needs automated testing, building, deployment, or release automation.

---

## Mathematical Foundations

### M1: Workflow Concurrency and Matrix Strategy
```
Matrix strategy expansion:
  matrix: {os: [ubuntu, windows, macos], node: [18, 20, 22]}
  Total jobs = Π(dimension sizes) = 3 × 3 = 9 concurrent jobs

  Billing impact: matrix jobs run concurrently (consume multiple runner slots)
  Cost: 9 parallel jobs × job_duration = 9× concurrent minute consumption

Concurrency control:
  concurrency: group: ${{ github.ref }}; cancel-in-progress: true
  Effect: only 1 workflow per branch runs (new push cancels old)
  Savings: branch with 5 commits in 1 minute → 5 workflow runs vs 1
  P(wasted_compute) = P(commit_pushed_before_previous_completes) ≈ high in active PRs

Runner parallelism limits:
  GitHub Free: 20 concurrent jobs
  GitHub Pro: 40 concurrent jobs
  GitHub Enterprise: 500+ concurrent jobs
```

### M2: Cache Hit Rate and Bandwidth Math
```
actions/cache key strategy:
  cache-key: runner.os-deps-${{ hashFiles('package-lock.json') }}
  P(cache_hit) = P(lockfile_unchanged since last cached run)

  Typical hit rates:
    Lock file hash:    70–90% (changes with dep updates)
    OS + lang version: 95%+ (rarely changes)
    Code hash:        ~0% (changes every commit)

Cache size limits:
  GitHub: 10 GB per repository cache storage
  LRU eviction: least recently used cache entries evicted when limit reached
  Optimization: cache per branch independently (each branch needs its own deps)

Bandwidth savings:
  Without cache: download_time = deps_size / runner_bandwidth
    npm: ~50–500 MB at ~100 MB/s = 0.5–5s
    Maven: ~100–1000 MB at ~100 MB/s = 1–10s
  With cache (cache hit): restore_time ≈ 0.1 × download_time (local SSD read)
```

### M3: OIDC Token Security Math
```
OIDC vs long-lived secrets:
  Long-lived secret: P(exposure) grows over time (rotation neglect, log leaks)
  OIDC token: max lifetime = job duration (minutes), P(replay_attack) ≈ 0

  OIDC token claims for cloud auth:
    iss: token issuer (GitHub's OIDC endpoint)
    sub: repo:owner/repo:ref:refs/heads/main (precise claim)
    aud: cloud provider (e.g., sts.amazonaws.com)

  AWS role trust policy entropy:
    subject_claim_filter = "repo:org/repo:ref:refs/heads/main"
    Only exact branch/repo can assume role → blast radius: 1 workflow

Token expiration:
  OIDC token issued per job step → expires after job completes
  No rotation needed → eliminates secret rotation complexity
```

### M4: Artifact Retention and Storage Cost
```
Artifact retention:
  Default: 90 days
  Storage: $0.25/GB/month for GitHub Enterprise (beyond free tier)

  Artifact size estimate:
    Build output (compiled binary): 10–500 MB
    Test reports: 1–10 MB
    Docker images: 50–1000 MB (avoid storing in Actions artifacts)

  Cleanup math:
    Daily artifacts × 90 days × avg_size = total artifact storage
    For 10 builds/day, 100MB avg: 10 × 90 × 100MB = 90 GB

Self-hosted runner cost:
  GitHub-hosted: ~$0.008/minute (Linux, 2-core)
  Self-hosted: fixed infra cost (EC2 t3.medium: ~$0.0416/hr = $0.0007/min)
  Break-even: self-hosted cheaper when > 5000 minutes/month per runner
```

### M5: GitHub Actions Cost Optimization Mathematics

**Step 1 — Minutes consumed formula per workflow run.**
GitHub Actions charges per-minute (rounded up) for GitHub-hosted runners:
```
minutes_consumed_per_run = ceil(T_run_seconds / 60)

T_run = max(T_critical_path_jobs, serial_job_durations_where_sequential)
      = max(job_1_duration_if_parallel, job_2_duration_if_parallel, ...)
      + Σ(sequential_job_durations)

OS multiplier (GitHub billing):
  Linux (ubuntu-*):    1× (baseline)
  Windows (windows-*): 2× (double minutes consumed)
  macOS (macos-*):     10× (ten times minutes consumed!)

Monthly cost formula:
  monthly_cost_USD = Σ_runs minutes_per_run × OS_multiplier × price_per_minute
  Linux price: $0.008/min (GitHub Team/Enterprise)
  Example: 100 runs/day × 10min × 1.0 × $0.008 = $24/day = $720/month

Concurrency group cancellation savings:
  Without cancel-in-progress: N pushes × T_run minutes consumed
  With cancel-in-progress:    1 run × T_run (only latest run completes)
  Savings ratio = (N-1)/N   for N pushes before completion
  For 5 pushes in 1 minute on active PR: 4/5 = 80% savings
```

**Step 2 — Matrix build parallelization speedup via Amdahl's Law.**
Amdahl's Law applied to CI matrix builds:
```
T_sequential = Σ_{i=1}^{N} T_job_i   (all N matrix jobs run in sequence)
T_parallel   = max(T_job_i) + T_setup_overhead   (all N jobs run concurrently)

Speedup S(N) = T_sequential / T_parallel

Matrix efficiency (accounting for setup overhead T_s per job):
  T_parallel = max(T_job_i) + T_s
  T_sequential = N × avg(T_job_i) + N × T_s   (setup overhead multiplied too)

  S(N) = [N × avg(T_job_i) + N × T_s] / [max(T_job_i) + T_s]

Balanced matrix (all jobs equal duration T_job):
  S(N) = N × (T_job + T_s) / (T_job + T_s) = N   [perfect linear speedup]

Imbalanced matrix (one job is 3× others, T_job_i ∈ {10min, 30min, 10min}):
  T_sequential = 50min;  T_parallel = 30min + 1min_setup = 31min
  S = 50/31 = 1.61×  [only 61% speedup despite 3 parallel slots]

Optimal matrix design: shard tests so max(T_job_i) ≈ avg(T_job_i)
  Test sharding into n shards: T_shard = total_test_time / n
  Optimal n = ceil(total_test_time / target_stage_time)
  For 60min total tests, target 10min stage: n = 6 shards → S = 6×
```

**Step 3 — Cache hit rate impact on build time.**
Cache effectiveness formula for dependency restoration:
```
T_build_with_cache = P(cache_hit) × T_restore + P(cache_miss) × T_download
  where P(cache_miss) = 1 - P(cache_hit)

  T_restore ≈ 0.1 × T_download  (local SSD read vs network download)
  T_restore for npm (500MB):   ~5s vs T_download = 50s  (10× faster)
  T_restore for Maven (300MB): ~3s vs T_download = 30s  (10× faster)

Cache hit rate by key strategy:
  P(hit | hash(package-lock.json)) = P(lockfile unchanged) ≈ 0.85 (changes ~15% of commits)
  P(hit | hash(source files))      ≈ 0.0 (source changes every commit)
  P(hit | OS + language_version)   ≈ 0.97 (changes only on version bumps)

Expected build time with optimal caching:
  T_expected = P(hit) × T_restore + P(miss) × (T_download + T_install)
  For P(hit)=0.85, T_restore=5s, T_download=50s, T_install=30s:
    T_expected = 0.85×5 + 0.15×80 = 4.25 + 12 = 16.25s
  vs. no caching: T_always = 80s
  Speedup from cache = 80/16.25 = 4.9× average build time reduction

Cache size budget:
  cache_used = n_branches × n_os × deps_size × safety_factor
  For 10 branches, ubuntu only, 200MB npm: 10 × 1 × 200MB × 1.2 = 2.4GB (well within 10GB limit)
```

---

### M6: India DevSecOps Regulatory Mathematics

**Step 1 — CERT-In CI/CD pipeline audit requirements (April 2022 circular).**
CERT-In mandates that any CI/CD system processing Indian user data must meet:
```
Audit_log_coverage = logged_pipeline_events / required_pipeline_events

Required events for CERT-In CI/CD compliance:
  [+1] Code commit events (WHO committed WHAT and WHEN)
  [+1] Secrets access events (which secrets were accessed per build)
  [+1] Dependency installation (packages, versions — supply chain audit)
  [+1] Build artifact signatures (SBOM generation + signing)
  [+1] Deployment authorization (WHO approved deployment to production)
  [+1] Infrastructure access (credentials used for cloud deployment)
  Total: 6 event categories

Audit trail retention: ≥ 180 days (CERT-In 2022)
  For 100 builds/day × 180 days × 50KB/build log = 900 MB (trivial)

CERT-In VAPT (Vulnerability Assessment & Penetration Testing) cycle:
  Mandatory frequency for government/regulated entities: ≥ 1 VAPT per 6 months
  VAPT finding severities in CI pipeline:
    CRITICAL: hardcoded secrets in workflow YAML → fix within 24 hours
    HIGH: unpinned action versions (supply chain risk) → fix within 7 days
    MEDIUM: missing SBOM generation → fix within 30 days
```

**Step 2 — STQC certification gate formula.**
STQC (Standardisation Testing and Quality Certification) is mandatory for government software:
```
STQC_certification_score = 0.40 × functional_testing_score
                         + 0.30 × security_testing_score
                         + 0.20 × performance_testing_score
                         + 0.10 × documentation_score

Each component scored 0–100:
  functional_testing_score: % of functional requirements with passing test cases
  security_testing_score:   100 - (10 × CRITICAL_vulns) - (5 × HIGH_vulns) - (1 × MED_vulns)
  performance_testing_score: score based on meeting performance SLAs under load

STQC_gate_pass = STQC_certification_score ≥ 75 (government applications)
  AND CRITICAL_vulnerabilities = 0
  AND HIGH_vulnerabilities ≤ 5 (with documented remediation plan)

Pipeline integration for STQC gate:
  GitHub Actions step: trivy scan → SARIF report → STQC_score calculation
  Gate condition: if security_score < 70 → fail workflow (block deployment)
```

**Step 3 — India NIC government CI/CD standards and compliance formula.**
NIC (National Informatics Centre) mandates for government projects hosted on GI Cloud:
```
NIC CI/CD compliance checklist score (binary checks):
  [+10] Source code in NIC GitLab (not GitHub for classified projects)
  [+10] Container images pulled from NICR (NIC Container Registry) only
  [+10] SBOM generated and stored per build (supply chain traceability)
  [+10] Build artifacts signed with NIC-approved PKI certificates
  [+10] Secrets managed via HashiCorp Vault (NIC instance) — no GitHub Secrets
  [+10] Pipeline runs on NIC-hosted Jenkins/GitLab CI (not GitHub Actions for Confidential)
  [+10] Deployment requires dual-authorization (two NIC-approved signatories)
  [+10] STQC report attached before production deployment
  [+10] Penetration test clearance < 6 months old
  [+10] IT Act 2000 §69 logging enabled (government interception readiness)

NIC_compliance_score = Σ satisfied_checks × 10
Minimum for production deployment on GI Cloud: NIC_compliance_score ≥ 80

For commercial projects (non-government) using GitHub Actions:
  SEBI Tech Risk Framework (2023): pipeline must log all production deployments
  RBI IT Master Direction (2024): MFA required for all production deployment approvals
  Data residency: build logs must not leave India if processing Indian SPD (sensitive personal data)
```

---

## Workflow Fundamentals

### 1. File Location and Naming
- Workflow files live in `.github/workflows/` directory in your repository
- File extension must be `.yml` or `.yaml`
- File name becomes the workflow name displayed in the Actions tab (unless `name:` is set)
- Multiple workflow files can coexist; each runs independently
- Use descriptive names: `ci.yml`, `deploy-production.yml`, `release.yml`

### 2. Workflow Structure
- Top-level keys: `name`, `on` (triggers), `env`, `permissions`, `concurrency`, `jobs`
- `name`: human-readable label shown in the GitHub Actions UI
- `on`: defines what events trigger the workflow (required)
- `jobs`: map of one or more jobs to execute (required)
- `env`: environment variables available to all jobs in the workflow
- `concurrency`: limits simultaneous runs of the same workflow or job

### 3. YAML Syntax Rules
- Indentation is significant; use 2 spaces (never tabs)
- Strings with special characters need quoting: `"${{ github.sha }}"` or single quotes for literals
- Multiline strings use `|` (literal block) or `>` (folded block)
- Comments start with `#`
- Boolean values: `true`/`false` (lowercase); `yes`/`no` are also valid YAML booleans
- Expression syntax: `${{ expression }}` for GitHub Actions expressions inside strings

---

## Triggers

### 4. Push and Pull Request Triggers
- `push`: fires on direct pushes to matching branches or tags
- `pull_request`: fires when PRs are opened, synchronized, or reopened against matching branches
- `pull_request_target`: runs in the context of the base branch; use for PRs from forks with secrets access
- Branch and tag filters use glob patterns: `main`, `release/*`, `v[0-9]+.*`
- Path filters limit trigger to specific file changes: `paths: ['src/**', '!docs/**']`
- `paths-ignore` excludes paths; `branches-ignore` excludes branches

```yaml
on:
  push:
    branches: [main, 'release/*']
    tags: ['v*.*.*']
  pull_request:
    branches: [main]
    paths: ['src/**', 'tests/**']
```

### 5. Schedule Trigger
- `schedule` uses cron syntax (UTC timezone)
- Minimum interval is every 5 minutes; GitHub may delay scheduled runs under heavy load
- Useful for nightly builds, dependency audits, and periodic cleanup jobs
- Scheduled workflows run on the default branch of the repository

```yaml
on:
  schedule:
    - cron: '0 2 * * 1-5'  # 2 AM UTC Monday through Friday
    - cron: '0 6 * * 0'    # 6 AM UTC on Sunday
```

### 6. Workflow Dispatch Trigger
- `workflow_dispatch` allows manual triggering from the GitHub UI or API
- Define `inputs` to accept parameters when manually triggering
- Input types: `string`, `choice`, `boolean`, `number`, `environment`
- Access inputs via `${{ github.event.inputs.input_name }}` or `${{ inputs.input_name }}`
- Useful for deployment workflows that need manual approval and parameter selection

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [staging, production]
      deploy_version:
        description: 'Version tag to deploy'
        required: true
        type: string
      dry_run:
        description: 'Perform dry run only'
        type: boolean
        default: false
```

### 7. Other Useful Triggers
- `workflow_call`: makes the workflow callable as a reusable workflow
- `release`: fires on release events (published, created, edited, deleted, prereleased)
- `issues`: fires on issue events (opened, closed, labeled, etc.)
- `issue_comment`: fires when a comment is posted on an issue or pull request
- `repository_dispatch`: fires via an external HTTP POST to the GitHub API (external system integration)
- `create` / `delete`: fires when a branch or tag is created or deleted
- Multiple triggers can be combined in a single workflow file

---

## Jobs

### 8. Job Definition
- Each job runs on a separate virtual machine (runner)
- Jobs run in parallel by default unless dependencies are specified with `needs`
- `runs-on` specifies the runner environment (required for each job)
- GitHub-hosted runners: `ubuntu-latest`, `windows-latest`, `macos-latest`
- Specific versions: `ubuntu-22.04`, `ubuntu-20.04`, `macos-13`, `windows-2022`
- Self-hosted runners: `runs-on: [self-hosted, linux, x64]`

```yaml
jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test

  build:
    name: Build Application
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: npm run build
```

### 9. Job Dependencies and Ordering
- `needs: job_id` or `needs: [job_a, job_b]` declares dependencies
- A job only starts when all its dependencies complete successfully
- Use `if: always()` or `if: failure()` to run a job even if dependencies failed
- `needs.job_id.outputs.key` accesses outputs from a dependency
- `needs.job_id.result` checks the result of a dependency (`success`, `failure`, `cancelled`, `skipped`)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    steps:
      - id: meta
        run: echo "tags=myimage:latest" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.image_tag }}"

  notify:
    needs: [build, deploy]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: echo "Pipeline finished"
```

### 10. Matrix Builds
- `strategy.matrix` runs the same job multiple times with different combinations of values
- Define matrix variables as lists; GitHub creates a job for each combination
- `include` adds extra matrix entries or augments existing ones
- `exclude` removes specific combinations from the matrix
- `strategy.fail-fast: false` prevents cancelling remaining matrix jobs when one fails
- `strategy.max-parallel` limits concurrent matrix jobs to control resource usage

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      max-parallel: 4
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.10', '3.11', '3.12']
        include:
          - os: ubuntu-latest
            python-version: '3.12'
            experimental: true
        exclude:
          - os: windows-latest
            python-version: '3.10'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: python -m pytest
```

### 11. Conditional Execution
- `if:` at job or step level controls whether execution occurs
- Expressions evaluate to boolean; the step or job is skipped if false
- Context objects: `github`, `env`, `vars`, `steps`, `jobs`, `runner`, `secrets`, `inputs`
- Functions: `contains()`, `startsWith()`, `endsWith()`, `format()`, `join()`, `toJSON()`, `fromJSON()`
- Status functions: `success()`, `failure()`, `cancelled()`, `always()`
- Default status check is `success()` when no `if:` is specified

```yaml
steps:
  - name: Deploy to production
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: ./deploy.sh production

  - name: Run only on PR from same repo
    if: github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name == github.repository
    run: ./integration-test.sh

  - name: Always clean up
    if: always()
    run: ./cleanup.sh
```

---

## Steps and Actions

### 12. Steps Structure
- Steps execute sequentially within a job on the same runner
- `uses: owner/repo@ref` runs a community or official action
- `run: command` runs a shell command directly
- `name:` provides a label for the step in the UI
- `id:` enables referencing the step's outputs and outcome later
- `env:` sets environment variables scoped to the step
- `with:` passes inputs to an action
- `continue-on-error: true` allows the workflow to continue when this step fails
- `timeout-minutes:` limits step execution time

### 13. Common Official Actions
- `actions/checkout@v4`: checks out the repository code
- `actions/setup-node@v4`: installs Node.js; supports multiple versions and caching
- `actions/setup-python@v5`: installs Python with optional pip cache
- `actions/setup-java@v4`: installs JDK with Maven/Gradle cache support
- `actions/setup-go@v5`: installs Go and manages module cache
- `actions/cache@v4`: caches files and directories between runs
- `actions/upload-artifact@v4`: uploads build artifacts for later download
- `actions/download-artifact@v4`: downloads artifacts uploaded in the same run
- `actions/github-script@v7`: runs JavaScript using the GitHub API client

### 14. Shell Commands
- Default shell on Ubuntu/macOS: `bash`; on Windows: `pwsh` (PowerShell)
- Override shell: `shell: bash`, `shell: python`, `shell: cmd`, `shell: pwsh`
- Multi-line commands use `|`:
  ```yaml
  run: |
    echo "Step 1"
    echo "Step 2"
    make build
  ```
- Environment variables set with `echo "VAR=value" >> $GITHUB_ENV`
- Outputs set with `echo "key=value" >> $GITHUB_OUTPUT`
- Step summary set with `echo "## Result" >> $GITHUB_STEP_SUMMARY`

---

## Secrets Management

### 15. Configuring Secrets
- Secrets are stored encrypted in GitHub at the repository, environment, or organization level
- Repository secrets: Settings > Secrets and variables > Actions > New repository secret
- Environment secrets: scoped to a specific deployment environment with optional protection rules
- Organization secrets: shared across multiple repositories in an organization
- Secrets are masked in logs; they never appear in plaintext output
- Maximum secret size: 64 KB; for larger secrets, store encrypted in the repo and decrypt at runtime

### 16. Accessing Secrets in Workflows
- Reference with `${{ secrets.SECRET_NAME }}`
- Pass to steps via `env:` or `with:` inputs
- Never echo secrets directly; they are masked but the practice is still a bad habit
- `GITHUB_TOKEN` is automatically available; it has repository-scoped permissions by default
- Environment secrets require the job to target the environment: `environment: production`

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy
        env:
          API_KEY: ${{ secrets.DEPLOY_API_KEY }}
        run: ./deploy.sh
```

### 17. Environment Variables
- `env:` at workflow, job, or step level sets environment variables
- `vars:` context accesses repository/environment/organization variables (non-secret config)
- `GITHUB_*` built-in variables: `GITHUB_SHA`, `GITHUB_REF`, `GITHUB_REPOSITORY`, `GITHUB_ACTOR`
- Set dynamic variables with `echo "VAR=value" >> $GITHUB_ENV` for use in subsequent steps
- Scoping: workflow-level `env` is available everywhere; job-level only in that job; step-level only in that step

```yaml
env:
  NODE_ENV: production
  APP_VERSION: ${{ github.sha }}

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      BUILD_DIR: dist
    steps:
      - name: Set computed variable
        run: echo "RELEASE_TAG=v$(cat VERSION)" >> $GITHUB_ENV
      - name: Use computed variable
        run: echo "Releasing ${{ env.RELEASE_TAG }}"
```

---

## Caching Dependencies

### 18. actions/cache Usage
- Cache key determines when to use an existing cache vs create a new one
- Key should include OS, dependency file hash, and any other differentiators
- `restore-keys` provides fallback keys for partial cache matches (ordered from most to least specific)
- Cache size limit: 10 GB per repository; oldest caches are evicted when the limit is reached
- Cache entries expire after 7 days of no access

```yaml
- name: Cache Node.js dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

- name: Cache Python pip packages
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-

- name: Cache Maven dependencies
  uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-maven-
```

### 19. Built-in Caching with Setup Actions
- `actions/setup-node@v4` has built-in caching: `cache: 'npm'` or `cache: 'yarn'` or `cache: 'pnpm'`
- `actions/setup-python@v5`: `cache: 'pip'` or `cache: 'pipenv'` or `cache: 'poetry'`
- `actions/setup-java@v4`: `cache: 'maven'` or `cache: 'gradle'`
- `actions/setup-go@v5`: `cache: true` enables Go module cache automatically
- Built-in caching uses the dependency lockfile hash as the cache key automatically

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'

- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'
```

---

## Artifacts

### 20. Uploading Artifacts
- Artifacts persist files between jobs in the same workflow run or for post-run download
- `upload-artifact@v4` uploads files or directories as a named artifact
- `retention-days` sets how long the artifact is kept (default 90 days, max 400 days)
- Artifacts are useful for: build outputs, test reports, coverage reports, log files
- Artifact names must be unique within a workflow run

```yaml
- name: Build application
  run: npm run build

- name: Upload build artifacts
  uses: actions/upload-artifact@v4
  with:
    name: build-output-${{ github.sha }}
    path: |
      dist/
      !dist/**/*.map
    retention-days: 30

- name: Upload test results
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: test-results
    path: test-results/junit.xml
```

### 21. Downloading Artifacts
- `download-artifact@v4` retrieves artifacts from the same workflow run
- `name:` specifies which artifact to download; omit to download all artifacts
- `path:` sets the destination directory (defaults to the artifact name)
- Use for passing build outputs between jobs: build job uploads, deploy job downloads
- Cross-workflow artifact access requires the GitHub API via `actions/github-script`

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - run: ./deploy.sh dist/
```

---

## Docker Container Actions

### 22. Using Docker in Workflows
- Run Docker containers as steps using `docker run` in a `run:` command
- Use pre-built Docker images for consistent tool versions
- Docker login with `docker/login-action@v3` for private registry access
- Build images with `docker/build-push-action@v6` for standardized builds

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Login to Docker Hub
    uses: docker/login-action@v3
    with:
      username: ${{ secrets.DOCKERHUB_USERNAME }}
      password: ${{ secrets.DOCKERHUB_TOKEN }}

  - name: Set up Docker Buildx
    uses: docker/setup-buildx-action@v3

  - name: Build and push Docker image
    uses: docker/build-push-action@v6
    with:
      context: .
      push: true
      tags: |
        myorg/myapp:latest
        myorg/myapp:${{ github.sha }}
      cache-from: type=gha
      cache-to: type=gha,mode=max
```

### 23. Docker Container as Job Environment
- Specify a Docker container for the entire job: `container: image_name`
- All steps in the job run inside the container
- Container environment variables and options can be configured under `container:`
- Services run alongside the job in separate containers: `services:`
- Services are accessible by their key name as the hostname within the job

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: python:3.12-slim
      env:
        PYTHONUNBUFFERED: 1
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt
      - run: pytest
        env:
          DATABASE_URL: postgresql://postgres:testpassword@postgres:5432/testdb
```

---

## Reusable Workflows

### 24. Creating Reusable Workflows
- A reusable workflow uses `on: workflow_call:` as its trigger
- Define `inputs` for parameters and `secrets` for sensitive values the caller must pass
- Outputs can be passed back to the calling workflow
- File location: any `.github/workflows/*.yml` file in a repository (public or same repo)
- Reusable workflows centralize CI/CD logic and reduce duplication across repositories

```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test Workflow
on:
  workflow_call:
    inputs:
      python-version:
        required: true
        type: string
      run-coverage:
        required: false
        type: boolean
        default: false
    secrets:
      codecov-token:
        required: false
    outputs:
      test-result:
        description: 'Whether tests passed'
        value: ${{ jobs.test.outputs.result }}

jobs:
  test:
    runs-on: ubuntu-latest
    outputs:
      result: ${{ steps.run-tests.outcome }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - id: run-tests
        run: pytest ${{ inputs.run-coverage && '--cov' || '' }}
```

### 25. Calling Reusable Workflows
- Use `uses: org/repo/.github/workflows/file.yml@ref` in a job (not a step)
- Pass inputs with `with:` and secrets with `secrets:` or `secrets: inherit`
- A job that calls a reusable workflow cannot have `steps:`
- `secrets: inherit` passes all caller secrets to the reusable workflow (use with caution)

```yaml
# .github/workflows/ci.yml
jobs:
  run-tests:
    uses: myorg/shared-workflows/.github/workflows/reusable-test.yml@main
    with:
      python-version: '3.12'
      run-coverage: true
    secrets:
      codecov-token: ${{ secrets.CODECOV_TOKEN }}

  deploy:
    needs: run-tests
    runs-on: ubuntu-latest
    steps:
      - run: echo "Tests result: ${{ needs.run-tests.outputs.test-result }}"
```

### 26. Composite Actions
- A composite action bundles multiple steps into a single reusable action
- Defined in `action.yml` file in any directory (often `.github/actions/my-action/`)
- Steps in composite actions can use `uses:` and `run:` just like workflow steps
- Inputs and outputs defined in `action.yml` are passed and returned by the caller
- Composite actions run in the context of the calling workflow runner

```yaml
# .github/actions/setup-and-test/action.yml
name: Setup and Test
description: Sets up Python environment and runs tests
inputs:
  python-version:
    description: Python version to use
    required: true
    default: '3.12'
outputs:
  test-outcome:
    description: Test result outcome
    value: ${{ steps.test.outcome }}
runs:
  using: composite
  steps:
    - uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}
    - name: Install dependencies
      shell: bash
      run: pip install -r requirements.txt
    - name: Run tests
      id: test
      shell: bash
      run: pytest --tb=short
```

---

## Security

### 27. GITHUB_TOKEN Permissions
- `GITHUB_TOKEN` is automatically created for each workflow run with scoped permissions
- Default permissions are configured at the repository or organization level
- Explicitly declare minimal permissions needed using `permissions:` at workflow or job level
- Permission scopes: `contents`, `pull-requests`, `issues`, `packages`, `actions`, `id-token`, etc.
- Setting any permission to `write` implicitly sets all others to `none` (best practice)

```yaml
permissions:
  contents: read        # Read repository files
  pull-requests: write  # Comment on PRs

jobs:
  build:
    permissions:
      contents: read
      packages: write   # Push to GitHub Container Registry
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
```

### 28. Third-Party Action Pinning
- Always pin third-party actions to a specific commit SHA, not a mutable tag
- Mutable tags (like `@v4` or `@main`) can be changed by the action author after you reference them
- Full SHA pinning: `uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`
- Audit action permissions and network access before adopting
- Use tools like `actionlint`, Dependabot, or `step-security/harden-runner` for security scanning
- Dependabot can auto-update pinned SHAs when new versions are released

```yaml
steps:
  # Pinned to a specific commit SHA for security
  - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
  - uses: actions/setup-python@0b93645e9fea7318ecaed2b359559ac225c90a2b  # v5.3.0
```

### 29. Preventing Script Injection
- Never interpolate `${{ github.event.*.body }}` or user-controlled inputs directly into `run:` commands
- Untrusted input can contain shell metacharacters that execute arbitrary commands
- Safe pattern: set the value as an environment variable, then reference the variable in the shell
- This prevents shell injection because environment variable expansion happens after shell parsing

```yaml
# UNSAFE - do not do this
- run: echo "${{ github.event.issue.title }}"

# SAFE - set as env var, reference in shell
- name: Process issue title
  env:
    ISSUE_TITLE: ${{ github.event.issue.title }}
  run: echo "$ISSUE_TITLE"
```

### 30. OpenID Connect (OIDC) for Cloud Authentication
- Use OIDC tokens to authenticate to cloud providers without storing long-lived credentials as secrets
- Supported by AWS, Azure, GCP, HashiCorp Vault, and others
- Requires `id-token: write` permission in the workflow
- Cloud provider verifies the OIDC token issued by GitHub; grants temporary credentials
- No secrets to rotate; credentials are automatically short-lived and scoped per workflow run

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
      - run: aws s3 sync dist/ s3://my-bucket/
```

---

## Common Patterns

### 31. Test on Pull Request
```yaml
name: CI
on:
  pull_request:
    branches: [main, develop]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test
      - run: npm run lint
```

### 32. Deploy on Merge to Main
```yaml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - run: ./scripts/deploy.sh
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
```

### 33. Release Automation
```yaml
name: Release
on:
  push:
    tags: ['v*.*.*']
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: dist/*.zip
          generate_release_notes: true
```

---

## Anti-Patterns to Avoid

- **Skipping `cancel-in-progress` concurrency control on active PR branches**: M1's math shows this wastes compute proportional to how often commits land before the previous run finishes — 5 pushes in 1 minute on an active PR without cancellation means 5 full workflow runs instead of 1, and M5's Step 1 formula quantifies the savings ratio as `(N-1)/N` (80% for that example).
- **Estimating CI cost without applying the OS multiplier**: M5's billing formula applies a 2× multiplier for Windows runners and a 10× multiplier for macOS versus the Linux baseline — a cost estimate built purely on job duration and Linux pricing will badly undercount actual spend for any workflow using Windows or macOS runners.
- **Designing an imbalanced test matrix**: M5 Step 2's worked example shows a 3×-imbalanced matrix (10/30/10 minutes) achieves only 1.61× speedup from 3 parallel slots instead of near-3× — the bottleneck job dominates wall-clock time regardless of how many other jobs run in parallel; sharding to balance `max(T_job_i) ≈ avg(T_job_i)` is what actually captures the parallelism.
- **Choosing self-hosted runners below the cost break-even point**: M4 states self-hosted only becomes cheaper than GitHub-hosted runners above roughly 5,000 minutes/month per runner — provisioning self-hosted infrastructure for lower-volume workloads adds operational overhead without the cost benefit that justifies it.
- **Keying dependency caches on source-file hashes instead of lockfile hashes**: M2/M5 Step 3 show `P(cache_hit | hash(source_files)) ≈ 0` since source changes every commit, versus `P(cache_hit | hash(lockfile)) ≈ 0.85-0.9` — a cache key tied to the wrong input effectively disables caching while still paying its bookkeeping overhead.
- **Storing Docker images as GitHub Actions artifacts**: M4 explicitly flags this as something to avoid — container images (50-1000MB) belong in a container registry, not Actions artifact storage, which is priced and structured for build outputs and test reports, not image distribution.
- **Using long-lived cloud credentials instead of OIDC for cloud authentication**: M3 contrasts long-lived secrets (exposure risk growing over time via rotation neglect and log leaks) against OIDC tokens (lifetime capped at job duration, `P(replay_attack) ≈ 0`) — defaulting to static AWS/GCP/Azure keys in GitHub Secrets when OIDC federation is available accepts unnecessary long-term exposure risk.
- **Leaving third-party action versions unpinned**: M6 Step 1 classifies unpinned action versions as a HIGH-severity VAPT finding with a 7-day mandatory remediation window under CERT-In guidance — using a floating tag like `@v2` instead of a pinned SHA is a supply-chain risk with a defined compliance deadline, not just a style preference.
- **Hardcoding secrets directly in workflow YAML**: M6 Step 1 rates this as the single CRITICAL-severity CI/CD finding with a 24-hour fix requirement — it's the most severe category in the VAPT severity table, above unpinned actions and missing SBOMs.
- **Using GitHub Actions and GitHub Secrets for classified or Confidential-tier government projects**: M6 Step 3's NIC compliance checklist mandates NIC-hosted Jenkins/GitLab CI and a NIC Vault instance instead of GitHub Actions/Secrets for this classification tier — for this specific project category, the platform choice itself is a compliance requirement, not a configuration detail within GitHub Actions.

## Response Rules

- Always specify exact action versions (prefer commit SHA pinning for third-party actions)
- Include `permissions:` declarations with minimal required permissions
- Recommend using built-in caching in setup actions when available
- Flag any pattern that could expose secrets or allow script injection
- Suggest OIDC over long-lived credentials for cloud authentication

---

## What Not to Do

- Do not interpolate untrusted input (`github.event.*.body`, PR titles) directly into `run:` commands
- Do not use `secrets: inherit` in reusable workflows unless access is explicitly intended
- Do not pin actions to mutable tags (`@v4`, `@latest`, `@main`) in security-sensitive workflows
- Do not store sensitive values in `env:` at the workflow level if they are only needed in one step
- Do not use `pull_request_target` without understanding its security implications (runs with full repo access)
- Do not skip the `permissions:` declaration; rely on permissive defaults

---

## Output Expectations

- Complete, working workflow YAML with proper indentation and syntax
- Inline comments explaining non-obvious choices
- Security-hardened examples by default (pinned actions, minimal permissions, OIDC where applicable)
- Matrix build configurations tailored to the language and test scope
- Caching configuration that matches the package manager in use

---

## Skill Scope

In scope:
- GitHub Actions workflow syntax and configuration
- Trigger configuration and filtering strategies
- Job orchestration, dependencies, and matrix builds
- Secrets, environment variables, and OIDC authentication
- Dependency caching and artifact management
- Docker container actions and service containers
- Reusable workflows and composite actions
- Security hardening and permission management
- Common CI/CD patterns (test-on-PR, deploy-on-merge, release)

Out of scope:
- GitHub Apps development and webhook handling
- Self-hosted runner infrastructure setup
- Jenkins, GitLab CI, CircleCI, or other CI/CD platforms
- Kubernetes deployment manifests and Helm charts
- Cloud provider-specific deployment services beyond workflow steps
- Repository management and branch protection rules

---

## Version
2.0.1 — Added Anti-Patterns to Avoid: missing cancel-in-progress, ignored OS cost multiplier, imbalanced test matrix, self-hosted-runner break-even miss, source-hash cache keys, Docker images as artifacts, long-lived cloud credentials, unpinned action versions, hardcoded workflow secrets, GitHub Actions for classified government projects
2.0.0
