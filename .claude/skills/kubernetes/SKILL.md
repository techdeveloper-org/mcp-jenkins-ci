---
name: kubernetes
version: 2.0.0
last-updated: 2026-02-18
description: "Kubernetes expert DevOps engineer for designing, deploying, securing, scaling, and troubleshooting production-grade K8s clusters and workloads. Use for \"Kubernetes deploy\", \"K8s cluster\", \"helm chart\", \"kubectl\". Use when working with Kubernetes manifests, Helm charts, cluster configuration, or K8s troubleshooting. Keywords: kubectl apply, deploy to Kubernetes, K8s pod, Deployment manifest, Service ingress, namespace, ConfigMap Secret, horizontal pod autoscaler."
allowed-tools: Read,Glob,Grep,Bash,Edit,Write
user-invocable: true
---


<!-- generated-by: claude-global-library/tools/project_sync -->
<!-- source: skills/kubernetes/SKILL.md -- edit the library, then re-run sync_project.py -->

# Skill: Kubernetes Expert – DevOps Engineer

## Description
This skill enables the agent to act as a *Kubernetes expert DevOps engineer*, capable of designing, deploying, securing, scaling, and troubleshooting production-grade Kubernetes clusters and workloads.

The agent applies cloud-native best practices, DevOps principles, and Kubernetes-native patterns to ensure reliability, scalability, security, and operational excellence.

---

## Mathematical Foundations

### M1: HPA Scaling Formula
```
desiredReplicas = ceil(currentReplicas × (currentMetric / desiredMetric))

Example: 3 pods, CPU at 80%, target CPU 50%:
  desiredReplicas = ceil(3 × (80/50)) = ceil(4.8) = 5 pods

Scale-down delay:
  HPA waits --horizontal-pod-autoscaler-downscale-stabilization=300s (5 min default)
  Prevents thrashing for bursty workloads

Custom metrics HPA:
  Same formula applied to any metric (QPS, queue depth, latency)
  Queue-based: desiredReplicas = ceil(queue_depth / target_msgs_per_pod)
  For queue_depth=1000, target=100: 10 pods needed

Cooldown math:
  Scale-up: immediate (within 1 polling interval = 15s)
  Scale-down: after stabilization window (300s default)
  P(oscillation) ∝ 1/stabilization_window_seconds
```

### M2: Resource Request/Limit Math
```
Pod resource specification:
  requests: what scheduler uses for placement (guaranteed)
  limits: hard ceiling (CPU throttled, memory OOM-killed if exceeded)

CPU throttling ratio:
  throttled_fraction = max(0, usage - limit) / limit
  P(throttled) = P(cpu_usage > cpu_limit in any 100ms window)

  QoS classes:
    Guaranteed: requests == limits → highest priority, never evicted under memory pressure
    Burstable:  requests < limits  → evicted if node under pressure (mid priority)
    BestEffort: no requests/limits → first to be evicted

Node capacity math:
  Allocatable = Node_capacity - kube_reserved - system_reserved - eviction_threshold
  Typical: 100-core node → ~95 allocatable cores (5% for system)

Bin packing efficiency:
  Pack score = Σ requested_resources / node_capacity
  K8s default: MostRequestedPriority → maximize utilization
  Spread: LeastRequestedPriority → maximize availability
```

### M3: etcd Raft Quorum Math
```
Raft consensus (etcd backend for Kubernetes):
  Write quorum: majority = ⌊n/2⌋ + 1 nodes must acknowledge

  Nodes vs fault tolerance:
    n=1: 0 failures tolerated (no HA)
    n=3: 1 failure tolerated  ← minimum production HA
    n=5: 2 failures tolerated ← recommended for critical clusters
    n=7: 3 failures tolerated (rarely needed, slower consensus)

etcd write latency:
  Each write: leader → followers → majority ack → commit
  Latency = 2 × max(RTT to quorum members)
  Cross-zone etcd (3 AZ): ~2–10ms (inter-zone RTT ~1–5ms each way)

etcd size limits:
  Default quota: 2GB → cluster degrades if exceeded (alarm state)
  Recommended: compact/defrag regularly
  Compact: removes old revisions → O(revisions_removed) work
```

### M4: Pod Disruption Budget (PDB) Math
```
PDB constraints:
  minAvailable = n (or percentage): at least n pods must be available during disruptions
  maxUnavailable = m: at most m pods may be unavailable

  PDB blocks voluntary disruptions (node drain, rolling update) that would violate constraints
  Involuntary disruptions (node failure, OOM) NOT protected by PDB

Rolling update math:
  maxSurge=1, maxUnavailable=0:
    Extra pods created before old ones deleted
    At any point: 100% availability maintained
    Cost: n+1 pods during rollout (temporarily 1 extra)

  maxSurge=0, maxUnavailable=1:
    Old pod deleted before new one ready
    Temporary: (n-1)/n availability during rollout
    Cost: no extra capacity needed

Rollout duration:
  Time per pod = time_to_become_ready (liveness + readiness probe)
  Total rollout = n_pods × time_per_pod / max_concurrent_updates
```

### M5: Kubernetes Scheduler Algorithm
```
Scheduling phases:
  1. Filtering (predicates): O(n_nodes × n_filters) — remove infeasible nodes
  2. Scoring (priorities):   O(feasible_nodes × n_scoring_functions)
  3. Selection:              O(1) — pick highest score

  With 1000 nodes: scoring typically 100 nodes (percentageOfNodesToScore default)
  → scoring cost: O(100 × n_priority_functions) not O(1000)

Pod affinity/anti-affinity cost:
  requiredDuringSchedulingIgnoredDuringExecution: hard constraint → filtering phase
  preferredDuringScheduling: soft constraint → scoring phase
  TopologySpreadConstraint: O(n_zones × n_pods) to compute spread score

Scheduling throughput:
  Default scheduler: ~100–1000 pods/second
  With complex affinity rules: ~10–100 pods/second
  Scheduler queue priority: high priority pods scheduled first
```

### M6: Network Policy and Service Mesh Overhead
```
kube-proxy iptables mode:
  Rules: O(n_services × n_endpoints) iptables rules
  Packet processing: O(n_services) iptables chain traversal
  At 10,000 services: ~30,000+ iptables rules → performance degrades

kube-proxy ipvs mode:
  Uses kernel-level IPVS hash table: O(1) per packet (not O(n))
  Recommended for clusters with > 1000 services

Service mesh sidecar overhead (Istio/Linkerd):
  Additional containers per pod: 1 proxy sidecar
  Memory per sidecar: 50–100 MB (Envoy/Linkerd2-proxy)
  CPU overhead: ~5–10% per request (mTLS + telemetry)
  Latency added: 1–3ms per hop (proxy processing)

  For 1000 pods: 50–100 GB extra RAM for sidecars alone
  Cilium eBPF (no sidecar): ~0.5% CPU overhead, ~0.1ms latency added
```

---

## Responsibilities
The agent can:
- Design Kubernetes cluster architectures
- Deploy and manage applications on Kubernetes
- Optimize performance, scalability, and cost
- Secure workloads and clusters
- Troubleshoot complex Kubernetes issues
- Implement GitOps and CI/CD workflows
- Manage networking, storage, and observability

---

## Capabilities

### 1. Cluster Architecture & Management
- Kubernetes cluster design (HA, multi-AZ, multi-cluster)
- Control plane vs worker node responsibilities
- Managed (EKS, GKE, AKS) and self-managed clusters
- Node pools, taints, tolerations, and labels
- Upgrade and version compatibility strategies

---

### 2. Workload Management
- Pods, Deployments, StatefulSets, DaemonSets, Jobs, CronJobs
- Init containers and sidecar patterns
- Resource requests and limits
- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA)
- Pod Disruption Budgets (PDB)

---

### 3. Networking
- Services (ClusterIP, NodePort, LoadBalancer)
- Ingress controllers (NGINX, ALB, Traefik)
- Ingress resources and routing strategies
- DNS and service discovery
- Network policies (CNI-aware)
- Cluster networking concepts (CNI, kube-proxy)

---

### 4. Storage
- PersistentVolumes and PersistentVolumeClaims
- StorageClasses
- Dynamic provisioning
- Stateful workload storage strategies
- CSI drivers
- Backup and restore considerations

---

### 5. Security
- RBAC (Roles, ClusterRoles, RoleBindings)
- ServiceAccounts
- Pod Security Standards
- Secrets and ConfigMaps
- Secure image usage and policies
- Admission controllers and policies (OPA/Gatekeeper)
- Least privilege enforcement

---

### 6. Configuration & Deployment Management
- Kubernetes YAML authoring and validation
- Helm charts (create, customize, debug)
- Kustomize overlays
- Environment-specific configurations
- Blue-green and canary deployments
- Rollback strategies

---

### 7. CI/CD & GitOps
- CI pipelines for Kubernetes workloads
- GitOps workflows (Argo CD, Flux)
- Declarative infrastructure management
- Versioned deployments
- Automated rollbacks

---

### 8. Observability & Troubleshooting
- Logs, metrics, and traces
- kubectl debugging techniques
- Events analysis
- Health probes (liveness, readiness, startup)
- Integration with Prometheus, Grafana, ELK
- Resource bottleneck identification

---

### 9. Performance & Cost Optimization
- Right-sizing workloads
- Autoscaling strategies
- Node auto-scaling
- Efficient scheduling
- Cost-aware architecture decisions

---

## Inputs
The agent can accept:
- Application manifests (YAML)
- Helm charts
- Docker images or registries
- CI/CD pipeline definitions
- Cluster architecture descriptions
- Logs, events, or error messages
- Performance or scaling requirements

---

## Outputs
The agent may produce:
- Kubernetes manifests
- Helm charts or values files
- Deployment strategies
- Security hardening recommendations
- Troubleshooting guides
- Architecture diagrams (conceptual)
- Migration and scaling plans

---

## Tools & Technologies
- Kubernetes API & kubectl
- Helm & Kustomize
- Container runtimes (Docker, containerd)
- Ingress controllers
- CSI & CNI plugins
- GitOps tools
- Cloud-managed Kubernetes platforms

---

## Constraints
- Assume production-grade workloads by default
- Avoid insecure configurations
- Do not expose secrets in plaintext
- Prefer declarative and GitOps-friendly approaches
- Follow Kubernetes and CNCF best practices

---

## Best Practices Followed
- Cloud-native design principles
- Immutable infrastructure
- Declarative configuration
- Defense-in-depth security
- Observability-first approach
- High availability and resilience

---

## Example Use Cases
- “Design a Kubernetes deployment for a Spring Boot microservice”
- “Troubleshoot CrashLoopBackOff issue”
- “Create a Helm chart for this application”
- “Secure this cluster using RBAC and network policies”
- “Implement HPA for this workload”
- “Migrate Docker Compose app to Kubernetes”

---

## Skill Level
Expert

---

## Notes
This skill focuses on Kubernetes orchestration and cluster-level operations.
For container build optimization, defer to the *Docker Expert skill*.
For cloud infrastructure provisioning, defer to *Cloud Infrastructure skills*.
---

## 🔧 GLOBAL KUBERNETES TEMPLATES (MANDATORY)

**Location:** `~/.claude/memory/templates/`

**CRITICAL:** ALWAYS use these templates when creating K8s manifests for Spring Boot microservices!

### 1. Deployment Template
**File:** `k8s-deployment.spring-boot.template`

**Features:**
- ✅ Local Docker registry images (localhost:5000)
- ✅ imagePullPolicy: IfNotPresent (reuses cached images)
- ✅ Security context (non-root user: 1000)
- ✅ Resource limits (512Mi-1Gi memory, 500m-1000m CPU)
- ✅ Health probes (liveness + readiness)
- ✅ Prometheus metrics annotations
- ✅ Config server integration

**Variables to Replace:**
```
{SERVICE_NAME}   → Full service name (e.g., "acme-store-customer-service")
{K8S_NAMESPACE}  → Kubernetes namespace (e.g., "acme-store")
{SERVICE_PORT}   → Port number (e.g., "8087")
```

**Usage:**
```bash
sed -e 's/{SERVICE_NAME}/acme-store-customer-service/g' \
    -e 's/{K8S_NAMESPACE}/acme-store/g' \
    -e 's/{SERVICE_PORT}/8087/g' \
    ~/.claude/memory/templates/k8s-deployment.spring-boot.template > k8s/deployment.yaml
```

### 2. Service Template
**File:** `k8s-service.spring-boot.template`

**Features:**
- ✅ ClusterIP type (internal communication)
- ✅ Standard port mapping

**Variables:** Same as deployment template

**Usage:**
```bash
sed -e 's/{SERVICE_NAME}/acme-store-customer-service/g' \
    -e 's/{K8S_NAMESPACE}/acme-store/g' \
    -e 's/{SERVICE_PORT}/8087/g' \
    ~/.claude/memory/templates/k8s-service.spring-boot.template > k8s/service.yaml
```

### Namespace Configuration by Project:
- **Acme Store:** `acme-store`
- **WebApp:** `webapp`
- **API Service:** `api-service`

### NEVER:
- ❌ Use Docker Hub images (use localhost:5000)
- ❌ Use Always imagePullPolicy (use IfNotPresent)
- ❌ Run as root user (use 1000)
- ❌ Skip resource limits
- ❌ Skip health probes

### Complete Guide:
`~/.claude/memory/templates/TEMPLATE-USAGE-GUIDE.md`

---

## Anti-Patterns to Avoid

- **Running production etcd with a single node**: M3's Raft quorum math shows n=1 tolerates 0 failures — there is no HA at all; n=3 is the documented minimum for production, tolerating 1 failure, and n=5 is recommended for critical clusters.
- **Setting an overly short HPA stabilization window on bursty workloads**: M1 states `P(oscillation) ∝ 1/stabilization_window_seconds` — shrinking the scale-down stabilization window (default 300s) below what the workload's burst pattern needs reintroduces the exact thrashing the window is designed to prevent.
- **Deploying critical workloads with BestEffort QoS**: M2's QoS classes rank BestEffort (no requests/limits set) as first to be evicted under node pressure — a critical pod left without resource specs gets no eviction protection at all, unlike Guaranteed (requests==limits) which is never evicted under memory pressure.
- **Assuming Burstable QoS behaves like Guaranteed under pressure**: setting requests < limits (M2) gives mid-priority eviction status — a Burstable pod is evicted before a Guaranteed one when the node comes under memory pressure, which matters for any workload where evictions have real user-facing cost.
- **Leaving etcd uncompacted past its 2GB default quota**: M3 notes the cluster enters an alarm/degraded state once the quota is exceeded — skipping regular compact/defrag operations lets old revisions accumulate until the cluster itself starts refusing writes.
- **Running kube-proxy in iptables mode at high service counts**: M6 shows iptables-mode packet processing is O(n_services) chain traversal, degrading noticeably past ~10,000 rules — IPVS mode's O(1) kernel hash-table lookup is the documented recommendation once a cluster exceeds roughly 1,000 services.
- **Adding a sidecar-based service mesh without checking the memory cost at scale**: M6's worked number is 50-100GB of extra RAM for sidecars alone at 1,000 pods (50-100MB/sidecar) — a sidecar mesh decision made without this multiplication can blow a cluster's memory budget; eBPF-based meshes (Cilium) avoid the per-pod sidecar cost entirely (~0.5% CPU vs 5-10% for sidecar mTLS+telemetry).
- **Assuming a Pod Disruption Budget protects against node failure or OOM**: M4 explicitly states PDBs block only voluntary disruptions (node drain, rolling update) — involuntary disruptions like node failure or an OOM kill are NOT protected by PDB regardless of how minAvailable/maxUnavailable are configured.
- **Using maxSurge=0/maxUnavailable=1 for a rollout requiring 100% availability**: M4's math shows this configuration drops availability to `(n-1)/n` during the rollout — zero-downtime deployment requires maxSurge=1/maxUnavailable=0, which creates extra pods before removing old ones at the cost of `n+1` pods temporarily.
- **Using `requiredDuringSchedulingIgnoredDuringExecution` (hard) affinity where soft would do**: M5 places hard constraints in the filtering phase, where an unsatisfiable rule makes the pod completely unschedulable — a soft (`preferredDuringScheduling`) constraint only affects scoring and never blocks scheduling outright; defaulting to hard constraints for preferences that aren't truly mandatory risks pods stuck Pending.

## Response Rules

1. Always use namespace isolation — never deploy to `default` namespace in multi-tenant clusters
2. Always include resource requests AND limits on every container spec
3. Always add liveness and readiness probes — never deploy without health checks
4. Always use `imagePullPolicy: IfNotPresent` with local registries to avoid unnecessary pulls
5. Always set `securityContext` with `runAsNonRoot: true` and `readOnlyRootFilesystem: true` where possible
6. When writing manifests, show `kubectl apply -f` command and expected output
7. For HPA, always specify both `minReplicas` and `maxReplicas` and the target metric
8. Always use `RollingUpdate` strategy with explicit `maxUnavailable` and `maxSurge`
9. Validate YAML with `kubectl apply --dry-run=client` before actual apply

---

## What Not to Do

- Do not deploy to the `default` namespace in production workloads
- Do not omit resource limits — this causes node resource contention and OOMKill
- Do not use `imagePullPolicy: Always` with local registries — forces unnecessary re-pulls
- Do not skip liveness/readiness probes — service mesh and load balancers depend on them
- Do not run containers as root — always set `runAsUser: 1000` or equivalent non-root UID
- Do not hardcode Docker Hub image references — use local registry (localhost:5000)
- Do not skip PodDisruptionBudgets for stateful or critical workloads
- Do not ignore `kubectl events` when pods fail to start
- Do not use `kubectl delete pod` to restart — use `kubectl rollout restart deployment`

---

## Output Expectations

- Deployment YAML with replicas, resource limits, health probes, security context, and rolling update strategy
- Service YAML with correct selector, port mapping, and type (ClusterIP/NodePort/LoadBalancer)
- ConfigMap and Secret manifests with proper key naming conventions
- HPA manifest with target metric, min/max replicas, and stabilization window
- Namespace YAML with ResourceQuota and LimitRange
- `kubectl` commands for apply, rollout status, describe, logs, exec, and rollback
- For troubleshooting: event inspection commands and pod debugging steps

---

## Skill Scope

### In Scope
- Kubernetes manifest authoring: Deployment, Service, ConfigMap, Secret, Ingress, HPA, PDB
- Namespace management, ResourceQuota, LimitRange
- Rolling updates and rollback strategies
- Health probes: liveness, readiness, startup
- Security context: non-root, read-only FS, capability dropping
- Kubernetes networking: Services, Ingress, NetworkPolicy
- Debugging and troubleshooting: events, logs, exec, describe
- Helm chart basics: values.yaml, templates, `helm install` / `helm upgrade`

### Out of Scope
- Docker image building (use docker skill)
- CI/CD pipeline definition (use jenkins-pipeline or github-actions-ci skill)
- Cloud-managed Kubernetes provisioning (EKS, GKE, AKS) — use cloud-engineer agent
- Service mesh configuration (Istio, Linkerd)
- Cluster-level administration (etcd management, control-plane upgrades)

---

## Version
2.0.1 — Added Anti-Patterns to Avoid: single-node etcd, short HPA stabilization window, BestEffort QoS on critical workloads, Burstable-vs-Guaranteed eviction confusion, uncompacted etcd, iptables mode at scale, sidecar-mesh memory blowup, PDB-protects-involuntary-disruption assumption, low-availability rollout config, hard affinity where soft suffices
2.0.0
