---
name: docker
version: 2.0.0
last-updated: 2026-02-18
description: "Docker expert for designing, building, optimizing, securing Dockerfiles, Compose files, CI/CD integration, troubleshooting containers. Use when user mentions Docker, Dockerfile, containerize, optimize image, secure Docker, docker-compose, or related tasks like \"dockerize my app\", \"fix Docker build\", \"multi-stage Dockerfile\". Keywords: write Dockerfile, docker-compose, build container, Docker image, container networking, Docker volume, multi-stage build, docker run."
allowed-tools: Read,Glob,Grep,Bash
user-invocable: true
---


<!-- generated-by: claude-global-library/tools/project_sync -->
<!-- source: skills/docker/SKILL.md -- edit the library, then re-run sync_project.py -->

# Skill: Docker Expert – DevOps Engineer

## Description
This skill enables the agent to act as a *Docker expert DevOps engineer*, capable of designing, building, optimizing, securing, and troubleshooting containerized applications in production-grade environments. The agent applies Docker best practices, DevOps principles, and cloud-native patterns.

It is optimized for:
- Microservices-based architectures
- CI/CD pipelines
- Cloud and on-prem deployments
- High availability and security-focused systems

---

## Mathematical Foundations

### M1: Image Layer Math (Copy-on-Write Overlay FS)
```
Docker image: ordered stack of read-only layers + 1 writable container layer
  Image size = Σ layer_sizes (no deduplication on disk, shared in memory)
  Container overhead = writable layer size (starts at 0, grows with writes)

Layer deduplication (content-addressable):
  SHA256 hash per layer → layers with same content share storage
  Benefit: pulling image with 5 shared layers → only 0 new layers downloaded

  Multi-stage build size reduction:
    Stage 1 (build tools): ~500MB (Go compiler, Maven, npm)
    Stage 2 (runtime only): ~50MB (only compiled binary + runtime)
    Reduction: 90% smaller production image

Layer caching hit rate:
  P(cache_hit) = P(layer_content_unchanged since last build)
  COPY . .:  P(cache_hit) ≈ 0 (any file change invalidates)
  COPY requirements.txt .:  P(cache_hit) ≈ 0.9 (only deps changed)
  Layer order rule: put least-changed layers FIRST for maximum cache hits
```

### M2: cgroup Resource Limits Math
```
CPU limits (--cpus, --cpu-shares):
  --cpus=0.5: container gets 50% of 1 CPU core (enforced by CFS scheduler)
  CPU throttling ratio = (throttled_time / total_time)
    P(throttled) increases when: cpu_usage_per_period > cpu_quota

  CFS parameters:
    cpu.cfs_period_us = 100,000 (100ms)
    cpu.cfs_quota_us  = cpus × 100,000
    For --cpus=2: quota = 200,000µs per 100ms → 2 cores equivalent

Memory limits:
  --memory=512m: hard limit → OOM kill if exceeded
  --memory-reservation=256m: soft limit (throttled, not killed)
  OOM kill score: process with highest oom_score_adj killed first

  Memory overhead per container:
    cgroup metadata: ~4KB
    Network namespace: ~2KB
    PID namespace: ~1KB
    Total container overhead: ~7KB (negligible)
```

### M3: Namespace Isolation Cost
```
Container creates 6 namespaces:
  pid, net, mnt, uts, ipc, user

Namespace creation cost: ~1ms per container (measured)
Process fork inside container: identical to bare-metal fork (~1µs)

Network namespace overhead:
  veth pair (virtual ethernet): adds ~0.5µs per packet vs host networking
  iptables/NAT overhead: ~2µs per packet for bridge networking
  --network=host: 0 overhead (no namespace, shares host network stack)

seccomp syscall filtering:
  Filters ~300+ syscalls → kernel check per syscall
  Overhead: ~1% CPU for syscall-intensive workloads (measured by Docker)
  Default Docker seccomp profile: blocks ~44 syscalls (security vs performance)
```

### M4: Build Cache Optimization Math
```
Dockerfile instruction cache invalidation:
  Any instruction change → ALL subsequent instructions must re-run
  Cost of rebuild from layer k = Σ time(layer_k to layer_n)

  Optimal Dockerfile ordering (most stable → least stable):
    1. Base image (rarely changes)
    2. System dependencies (apt-get, yum) - changes with security patches
    3. Application dependencies (package.json, requirements.txt) - changes with libs
    4. Application code - changes every commit

  Build time with poor ordering:    n minutes (always rebuilds all layers)
  Build time with optimal ordering: k seconds (rebuilds only changed + after)

BuildKit parallel stages:
  --mount=type=cache: persistent build cache across builds
  Parallel stage execution: O(critical_path_depth) instead of O(total_stages)
```

### M5: Container Startup Time Budget
```
Container startup time formula (total):
  T_start = T_pull + T_extract + T_ns_create + T_cgroup_setup + T_app_startup

  Where:
    T_pull    = image_size_MB / download_bandwidth_MBps   (cold: 0 if cached)
    T_extract = n_layers × T_per_layer_extract             (T_per_layer ≈ 0.8–1.2s for overlayfs)
    T_ns_create = 6 × T_namespace_init ≈ 6ms              (pid/net/mnt/uts/ipc/user namespaces)
    T_cgroup_setup ≈ 1ms                                   (kernel cgroup hierarchy init)
    T_app_startup: language-dependent (see below)

  Cold start (image not in registry cache):
    T_cold = (500MB / 50MBps) + (8 × 1s) + 6ms + 1ms + T_app = 10s + 8s + T_app ≈ 18s + T_app

  Warm start (image cached on node):
    T_warm = 0 + 0 + 6ms + 1ms + T_app ≈ 7ms + T_app

  Application startup times by runtime:
    T_app(JVM default)        = 2,000–5,000ms (class loading + JIT compilation)
    T_app(JVM + CDS)          = 1,200–3,500ms (20–40% reduction via Class Data Sharing)
    T_app(GraalVM native)     = 10–100ms      (AOT compiled to native binary)
    T_app(Node.js)            = 100–2,000ms   (V8 init + module loading)
    T_app(Go binary)          = 5–50ms        (statically linked, no VM overhead)
    T_app(Python)             = 200–800ms     (interpreter + import tree)

  Layer extraction optimization:
    T_extract = n_layers × 0.8s
    Multi-stage build: 5 layers → T_extract = 4s  (vs 20 layers → 16s for single-stage)
    Reduction: n_stage2_layers << n_stage1_layers  (e.g., 5 vs 25 layers → 4× faster extract)

  Kubernetes readiness probe impact:
    T_ready = T_start + probe_initial_delay + ceil(T_app / probe_period) × probe_period
    Optimal: initialDelaySeconds = T_warm + σ(T_app)  (mean + 1 stdev of app startup)
```

### M6: India MeitY GI Cloud Docker Deployment Mathematics

**Step 1 — Container density formula on GI Cloud (NIC MeghRaj).**
GI Cloud (Government of India Cloud, operated by NIC under MeitY) uses commodity hardware:
```
Containers_per_VM = floor(VM_RAM_GB × 1024 / container_memory_MB) × CPU_overcommit_ratio
  Typical GI Cloud VM: 16 vCPU, 32 GB RAM
  For Spring Boot container (512 MB RAM, 0.5 vCPU):
    Containers_per_VM = floor(32768 / 512) × min(16/0.5, overcommit_limit)
                      = 64 × min(32, 4)    [GI Cloud max overcommit ≈ 4×]
                      = 64 × 4 = 256 theoretical max

  Practical density (80% memory utilization target):
    Containers_per_VM_practical = floor(0.80 × 32768 / 512) = 51 containers/VM

Container density efficiency ratio:
  η_density = Containers_actual / Containers_theoretical
  GI Cloud observed: η_density ≈ 0.65–0.75 (OS overhead, monitoring agents, log collectors)
```

**Step 2 — Cost per container per hour on India cloud infrastructure (INR).**
Comparing GI Cloud (NIC) vs AWS Mumbai vs Azure India Central vs Google Cloud Mumbai:
```
Cost_per_container_per_hour = VM_cost_per_hour_INR / containers_per_VM

  GI Cloud (NIC MeghRaj — government rate, FY 2024-25):
    VM (16vCPU/32GB): ₹18/hour (subsidized government rate)
    Cost_per_container = ₹18 / 51 = ₹0.35/container/hour

  AWS Mumbai (m6i.4xlarge, 16vCPU/64GB, on-demand):
    VM cost: ≈ ₹780/hour (USD ~9.40 × ₹83)
    Containers: floor(64GB/0.5GB) × 0.80 = 102
    Cost_per_container = ₹780 / 102 = ₹7.65/container/hour

  Reserved (1-year) AWS cost reduction: 30–40%
    Reserved cost_per_container = ₹7.65 × 0.65 = ₹4.97/container/hour

  GI Cloud cost advantage = AWS_cost / GI_cost = 7.65 / 0.35 = 21.9×
  (Primary driver: government procurement vs commercial cloud pricing)
```

**Step 3 — CERT-In container hardening compliance score formula.**
CERT-In (Indian Computer Emergency Response Team) guidelines (Apr 2022 circular + Docker hardening):
```
CERT-In compliance score (0–100):
  Score = Σ_i weight_i × compliance_i

  Critical controls (weight = 10 each):
    [+10] Non-root USER in Dockerfile (USER nonroot)
    [+10] Read-only root filesystem (--read-only or securityContext.readOnlyRootFilesystem)
    [+10] No privileged containers (--privileged=false)
    [+10] CERT-In approved base image (RedHat UBI / Debian stable, patched < 30 days)
    [+10] Log retention ≥ 180 days (RBI/CERT-In mandate for financial containers)

  High controls (weight = 5 each):
    [+5]  Dropped capabilities (--cap-drop=ALL + --cap-add only required)
    [+5]  Seccomp profile applied (docker/default or custom)
    [+5]  No secrets in ENV (verified by trivy secret scan)
    [+5]  Image signed (cosign + India NIC PKI or commercial CA)
    [+5]  CVE scan passed (no CRITICAL vulns, HIGH ≤ 5 with CERT-In accepted exceptions)

  Moderate controls (weight = 2 each):
    [+2]  Health check defined (HEALTHCHECK instruction)
    [+2]  Resource limits set (memory + CPU)
    [+2]  .dockerignore present
    [+2]  Multi-stage build used
    [+2]  No LATEST tag in FROM

  CERT-In threshold: Score ≥ 85 for government deployment approval
  Financial sector (RBI regulated): Score ≥ 95 required (additional IT Act 2000 §43A)
```

**Step 4 — India NIC/MeitY container compliance gate formula.**
MeitY mandates GIGW 2.0 (Guidelines for Indian Government Websites) + STQC certification for production:
```
STQC_gate_pass = (CERT-In_score ≥ 85)
              AND (VAPT_clearance = TRUE)       [Vulnerability Assessment & Penetration Test]
              AND (data_residency = "India")     [MeitY data localization requirement]
              AND (log_audit_trail = TRUE)       [IT Act 2000 §67C — 180 days logs]
              AND (encryption_at_rest = AES256) [MeitY security policy §4.2]

Container registry compliance (GI Cloud):
  Allowed registries: NIC Container Registry, NICR (nicr.gov.in)
  Banned: Docker Hub direct pulls in production (CERT-In supply chain risk advisory)
  Mirror requirement: all images must be mirrored to NICR with signature verification

  Pull policy compliance rate = Pulls_from_NICR / Total_pulls ≥ 0.999 (99.9% target)
```

---

## Responsibilities
The agent can:
- Design efficient Docker architectures
- Write production-grade Dockerfiles
- Optimize image size, build time, and runtime performance
- Secure container images and runtime environments
- Troubleshoot Docker and container-related issues
- Integrate Docker with CI/CD pipelines
- Collaborate with Kubernetes, cloud services, and monitoring tools

---

## Capabilities

### 1. Docker Image Design & Optimization
- Multi-stage Dockerfile creation
- Layer caching optimization
- Base image selection (alpine, distroless, scratch)
- Reducing image size and attack surface
- Handling JVM, Node.js, Python, Go, and native binaries

### 2. Container Runtime Management
- Docker Engine configuration
- Resource limits (CPU, memory, ulimits)
- Networking (bridge, host, overlay)
- Volumes and bind mounts
- Logging drivers and log rotation

### 3. Security Best Practices
- Non-root containers
- Image vulnerability scanning
- Secrets handling (Docker secrets, env vars, external secret managers)
- Least privilege principles
- Immutable infrastructure patterns

### 4. Docker Compose & Local Orchestration
- docker-compose.yml authoring
- Environment-specific overrides
- Dependency management between services
- Local dev parity with production

### 5. CI/CD Integration
- Docker build & push automation
- BuildKit usage
- Caching strategies in CI pipelines
- Versioning and tagging strategies
- GitHub Actions, GitLab CI, Jenkins compatibility

### 6. Observability & Debugging
- Container logs analysis
- Exec and debugging inside containers
- Health checks
- Metrics exposure
- Integration with Prometheus, Grafana, ELK

### 7. Production Readiness
- Zero-downtime deployments (blue-green, rolling)
- Graceful shutdown handling
- Startup dependency management
- Disaster recovery considerations

---

## Inputs
The agent can accept:
- Application source code
- Existing Dockerfiles
- docker-compose.yml files
- CI/CD pipeline configurations
- Error logs or runtime issues
- Architecture diagrams or descriptions

---

## Outputs
The agent may produce:
- Optimized Dockerfiles
- Secure docker-compose configurations
- CI/CD pipeline snippets
- Troubleshooting steps
- Best practice recommendations
- Architecture explanations
- Migration strategies

---

## Tools & Technologies
- Docker Engine & Docker CLI
- Docker Compose
- BuildKit
- Container registries (Docker Hub, ECR, GCR, ACR)
- Linux container internals
- CI/CD platforms
- Kubernetes (integration-level knowledge)

---

## Constraints
- Avoid insecure defaults
- Prefer minimal, production-safe images
- Do not expose secrets in images or logs
- Follow cloud-native and DevOps best practices
- Assume production-scale workloads unless specified otherwise

---

## Best Practices Followed
- Twelve-Factor App methodology
- Immutable infrastructure
- Infrastructure as Code mindset
- Security by default
- Performance and cost optimization

---

## Example Use Cases
- “Optimize this Dockerfile for a Spring Boot microservice”
- “Fix high memory usage in this container”
- “Create a multi-stage Dockerfile for Node.js”
- “Secure this Docker image for production”
- “Dockerize an existing monolith”
- “Integrate Docker builds into CI pipeline”

---

## Skill Level
Expert

---

## Notes
This skill focuses strictly on Docker and containerization. For orchestration-level automation, defer to Kubernetes or Cloud Infrastructure skills.

---

## 🔧 GLOBAL DOCKERFILE TEMPLATE (MANDATORY)

**Location:** `~/.claude/memory/templates/Dockerfile.spring-boot.template`

**CRITICAL:** ALWAYS use this template when creating Dockerfiles for Spring Boot microservices!

### Template Features:
- ✅ Multi-stage build (build stage + runtime stage)
- ✅ Local Docker registry (localhost:5000) - eliminates Docker Hub rate limiting
- ✅ Maven dependency caching via Docker layers
- ✅ Java 21 with Eclipse Temurin
- ✅ Alpine-based runtime (smaller image size)
- ✅ Optimized layer caching (dependencies separate from source code)

### Variables to Replace:
```
{COMMON_UTIL_VERSION} → Common utility version (e.g., "1.0.0", "2.2.1")
{SERVICE_PORT}        → Port number (e.g., "8087", "8095")
{PROJECT_NAME}        → Project identifier (e.g., "acme-store", "webapp", "api-service")
```

### Usage:
```bash
sed -e 's/{COMMON_UTIL_VERSION}/2.2.1/g' \
    -e 's/{SERVICE_PORT}/8087/g' \
    -e 's/{PROJECT_NAME}/acme-store/g' \
    ~/.claude/memory/templates/Dockerfile.spring-boot.template > Dockerfile
```

### Project-Specific Configuration:

| Project | PROJECT_NAME | Common Util Version | Typical Ports |
|---------|--------------|---------------------|---------------|
| Acme Store | acme-store | Dynamic (from pom.xml) | 8081-8095 |
| WebApp | webapp | 1.0.0 (fixed) | 8761, 8085, 8888 |
| API Service | api-service | 1.0.0 (fixed) | 9001-9013 |

### Base Images (MUST be in localhost:5000):

Before using template, ensure these images are in local registry:

```bash
# Pull from Docker Hub
docker pull maven:3.9.9-eclipse-temurin-21
docker pull eclipse-temurin:21-jre-alpine

# Tag for local registry
docker tag maven:3.9.9-eclipse-temurin-21 localhost:5000/maven:3.9.9-eclipse-temurin-21
docker tag eclipse-temurin:21-jre-alpine localhost:5000/eclipse-temurin:21-jre-alpine

# Push to local registry
docker push localhost:5000/maven:3.9.9-eclipse-temurin-21
docker push localhost:5000/eclipse-temurin:21-jre-alpine
```

### NEVER:
- ❌ Use Docker Hub images directly (FROM maven:... ❌)
- ❌ Use ALWAYS in imagePullPolicy
- ❌ Skip multi-stage builds
- ❌ Copy entire project at once (breaks layer caching)
- ❌ Use latest tag in production

### Layer Optimization:
1. Copy pom.xml first → Download dependencies (cached)
2. Copy source code last → Only rebuilds when code changes
3. Separate build stage from runtime stage
4. Use .dockerignore to exclude unnecessary files

### Complete Guide:
`~/.claude/memory/templates/TEMPLATE-USAGE-GUIDE.md`

---

## Anti-Patterns to Avoid

- **Running `COPY . .` before copying dependency manifests**: M1's cache-hit math shows `COPY . .` has `P(cache_hit) ≈ 0` (any file change invalidates it) versus `COPY requirements.txt .` at `P(cache_hit) ≈ 0.9` — copying the whole context early forces a full dependency reinstall on every code change instead of only when dependencies actually change.
- **Ordering Dockerfile instructions least-stable-first**: M4's cache-invalidation rule is absolute — any instruction change forces ALL subsequent instructions to re-run — so placing frequently-changing instructions (application code) before rarely-changing ones (base image, system deps) turns every build into a full rebuild instead of the intended incremental one.
- **Setting `--cpus` and assuming smooth throttling**: the CFS scheduler enforces the quota within each 100ms period (M2) — a container that bursts past its quota mid-period gets throttled hard until the next period resets, not smoothly rate-limited; bursty workloads sized only by average CPU usage will see throttling the average doesn't predict.
- **Using only a hard memory limit for bursty workloads**: `--memory` triggers an OOM kill on breach, while `--memory-reservation` only throttles (M2) — a workload with legitimate transient memory spikes needs the reservation/soft-limit distinction, or a hard-limit-only configuration will kill the container for spikes it should have survived.
- **Defaulting to `--network=host` for isolation-sensitive workloads**: host networking eliminates network-namespace overhead entirely (M3) but also eliminates the network isolation containers exist to provide — trading that isolation away for the ~0.5-2µs/packet veth/NAT overhead is rarely the right tradeoff outside of genuinely latency-critical, trusted workloads.
- **Disabling seccomp filtering "for performance"**: the measured overhead is only ~1% CPU for syscall-intensive workloads (M3) — disabling the default profile (which blocks ~44 syscalls) to chase that marginal gain removes a real security control for a performance benefit that's usually not worth it.
- **Setting Kubernetes `initialDelaySeconds` to a guessed fixed value**: M5's optimal formula is `T_warm + σ(T_app)` (mean plus one standard deviation of measured app startup) — a guessed delay either routes traffic before the app is ready (too short) or wastes startup time doing nothing (too long); the delay should come from measured T_app data, not intuition.
- **Using JVM-default startup for latency-sensitive autoscaling or scale-to-zero workloads**: M5's runtime comparison shows JVM default at 2,000-5,000ms versus GraalVM native at 10-100ms — a 20-500× difference that directly determines whether a scale-to-zero or fast-autoscaling architecture is viable; picking the default runtime without checking this delta can make an otherwise-sound autoscaling design fail its latency budget.
- **Pulling images directly from Docker Hub in GI Cloud/government production deployments**: M6 Step 4 explicitly bans this per CERT-In's supply-chain risk advisory — all images must be mirrored to NICR with signature verification; a direct Docker Hub pull in this context is a compliance violation, not just a suboptimal practice.
- **Treating a CERT-In compliance score of 85 as sufficient for RBI-regulated financial containers**: M6 Step 3 sets the general government threshold at ≥85 but the financial-sector threshold at ≥95 — deploying an RBI-regulated container at the general government bar, rather than the stricter financial-sector one, under-clears the actual required compliance level.

## Response Rules

1. Always write multi-stage Dockerfiles — never single-stage for production images
2. Always use specific image tags, never `latest` in Dockerfiles or Compose
3. Always use local registry (`localhost:5000`) when configured — never pull directly from Docker Hub in project pipelines
4. Always add `.dockerignore` alongside every Dockerfile
5. Always set non-root USER in the runtime stage for security
6. Always set resource limits (memory, CPU) in Compose and Kubernetes manifests
7. When troubleshooting, show `docker logs`, `docker inspect`, and `docker stats` commands
8. Never store secrets in Dockerfile ENV instructions — reference external secret managers
9. Validate layer caching strategy: deps copied before source code

---

## What Not to Do

- Do not use `latest` tag for base images in any production Dockerfile
- Do not use `ADD` where `COPY` suffices — `ADD` has implicit behaviors (tar extraction, URL fetch)
- Do not install dev dependencies in runtime stage — build stage only
- Do not run containers as root user in production
- Do not expose unnecessary ports — only the service port
- Do not use `ALWAYS` imagePullPolicy in Kubernetes with local registries
- Do not skip `.dockerignore` — this leaks secrets and inflates image size
- Do not run `docker build` without cache busting strategy when dependencies change
- Do not hard-code environment-specific config in the image — use environment variables

---

## Output Expectations

- Dockerfile with multi-stage build, correct base image, non-root user, health check
- `.dockerignore` file alongside every Dockerfile
- `docker-compose.yml` with named volumes, restart policies, resource limits, and health checks
- Build commands with explicit tagging (`-t name:version`)
- Layer optimization explanation showing caching strategy
- For local registry: push/pull commands and tag commands included
- Security notes flagging any deviations from least-privilege principle

---

## Skill Scope

### In Scope
- Writing and optimizing Dockerfiles (multi-stage, layer caching, ARG/ENV, COPY vs ADD)
- Docker Compose: service definitions, volumes, networks, depends_on, health checks
- Container image optimization: image size reduction, layer deduplication
- Container security: non-root user, read-only filesystem, capability dropping, secret management
- Local and remote Docker registry usage (localhost:5000, Docker Hub, ECR, GCR)
- Debugging containers: logs, exec, inspect, stats, events
- Docker networking: bridge, host, overlay, macvlan

### Out of Scope
- Kubernetes orchestration (use kubernetes skill)
- CI/CD pipeline configuration (use jenkins-pipeline or github-actions-ci skill)
- Cloud infrastructure provisioning (use cloud-engineer agent)
- Container runtime security beyond Docker capabilities (eBPF, Falco)

---

## Version
2.0.1 — Added Anti-Patterns to Avoid: early COPY-all cache busting, unstable-first layer ordering, CFS burst throttling misunderstanding, hard-limit-only OOM risk, default host networking, seccomp disabled for performance, guessed readiness-probe delay, JVM-default on scale-to-zero, direct Docker Hub pulls in GI Cloud, under-cleared financial-sector CERT-In score
2.0.0
