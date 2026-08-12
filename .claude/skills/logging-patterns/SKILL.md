---
name: logging-patterns
description: "Comprehensive guidance for logging and observability patterns including structured logging, log levels, correlation IDs, context propagation, log aggregation, metrics design, sampling, and language-specific patterns for Python, Java, and TypeScript. Use when implementing structured logging, correlation IDs, log aggregation, metrics design, or observability in any application. Keywords: structured logging, add logging, log aggregation, correlation ID, ELK stack, observability setup, log format."
allowed-tools: Read,Glob,Grep,Bash,Edit,Write
user-invocable: true
---


<!-- generated-by: claude-global-library/tools/project_sync -->
<!-- source: skills/logging-patterns/SKILL.md -- edit the library, then re-run sync_project.py -->

# Logging and Observability Patterns Skill Instructions

### Skill Name
logging-patterns

### Description
This skill provides comprehensive guidance for building observable production systems through effective logging, metrics, and distributed tracing. It covers structured logging with JSON format, log level selection, correlation ID patterns, context propagation across service boundaries, log aggregation (ELK, Loki), metrics design, sampling strategies for high-volume systems, and language-specific implementations for Python, Java, and TypeScript.

This skill applies to backend services, APIs, data pipelines, and any system that needs reliable observability in production.

---

## Mathematical Foundations

### M1: Log Volume and Sampling Math
```
Log ingestion cost:
  log_volume = request_rate × avg_logs_per_request × avg_log_size
  Example: 1000 RPS × 5 logs/request × 500B = 2.5 MB/s = 216 GB/day

Log sampling strategies:
  Head-based sampling:
    P(log sampled) = sample_rate  (decided at request start)
    Consistent: all logs in same trace either all sampled or all dropped
    Memory: O(1) per decision

  Tail-based sampling:
    Sample after full trace completes: always keep errors, sample success
    P(keep) = P(error) × 1.0 + P(success) × sample_rate
    Memory: O(in_flight_traces × avg_trace_size) — buffer all spans until decision

Sampling rate impact on debugging:
  At 1% sampling: P(sampled | specific_error) = 1% per occurrence
  Need ~100 occurrences to have 63% chance of capturing one sample
  P(captured at least once after n occurrences) = 1 - (1-sample_rate)^n
  For n=100, p=0.01: P = 1 - (0.99)^100 = 63.3%

Cardinality explosion:
  High cardinality field (user_id, request_id): O(n_unique_values) time series
  For 1M users × 10 metrics = 10M time series → Prometheus memory: ~10M × 1KB = 10GB
  Rule: never use high-cardinality values as metric labels
```

### M2: Correlation ID Entropy Requirements
```
Correlation ID collision probability (birthday paradox):
  P(collision) ≈ n² / (2 × id_space_size)

  For n = concurrent_requests × retention_period:
    100 RPS × 86400s × 7 days = 60.5M requests in 7-day window
    UUID v4 (122 random bits, space=2^122): P(collision) ≈ (60.5M)² / (2×2^122) ≈ 10^-22

  Minimum entropy: 64 bits is safe for most systems
    64-bit ID, 1M requests/day: P(collision/day) = 10^-6

Trace context propagation overhead:
  W3C Trace Context: 2 HTTP headers (~100 bytes total)
  Headers per request: constant overhead regardless of payload size
  Overhead ratio: 100B / (1KB avg payload) = 10% → negligible for most APIs
```

### M3: Log Level Performance Impact
```
Log level filtering overhead:
  Logger.isEnabled(level) check: ~10 ns (bitwise comparison)
  String formatting (if level enabled): ~1-10 µs (string concat, JSON serialization)
  I/O write (async logger): ~0 ns at call site (buffered to queue)
  I/O write (sync logger): ~10-100 µs (syscall, disk/network)

Async logger buffering:
  Queue size = buffer_capacity × avg_log_entry_size
  Overflow behavior: drop (lose logs) or block (back-pressure on app)
  Recommended: NEVER block application for logging (drop with counter)

Log level cost hierarchy (per log call):
  TRACE/DEBUG (if disabled): ~10 ns (isEnabled check only)
  INFO/WARN/ERROR (async):   ~200-500 ns (serialize + queue enqueue)
  ERROR (sync with stack trace): ~100 µs (stack capture + serialize + I/O)

Structured JSON logging vs plain text:
  JSON serialization: ~5-50× slower than plain string concat
  JSON parsing at query time: O(1) per field vs O(n) regex scan for plain text
  Trade-off: slightly slower writes → much faster queries
```

### M4: Distributed Trace Overhead Math
```
Span overhead per service call:
  Span creation: ~1-5 µs (allocate span object, set start time)
  Context propagation: ~100 ns (read/write from/to carrier)
  Span export (async): ~0 ns at call site (buffered)
  Span export (sync): ~100 µs per batch

Sampling math for tracing budget:
  Target trace volume: storage_budget / avg_trace_size / retention_days
  Example: 10 GB/day storage, 10 KB avg trace, 1-day retention:
    10 GB / 10 KB = 1,000,000 traces/day = 11.6 traces/second

  At 1000 RPS: sample_rate = 11.6 / 1000 = 1.16% → round to 1%

Head sampling vs adaptive:
  Fixed 1% head sampling: poor for low-traffic endpoints (may miss all errors)
  Adaptive per-endpoint: high-traffic = 0.1%, low-traffic = 100%
  P(capture error | adaptive) >> P(capture error | fixed rate) for low-traffic errors
```

### M5: Log Storage Cost Optimization Mathematics

**Step 1 — Log compression ratio formula (LZ4 vs gzip).**
Log data is highly compressible; choosing the right algorithm balances CPU vs storage cost:
```
Compression ratio (CR) = compressed_size / original_size

Empirical log compression ratios (for structured JSON logs):
  No compression:    CR = 1.00, throughput = ∞ (CPU-bound: 0% overhead)
  LZ4 compression:   CR = 0.25–0.35, throughput = 400–600 MB/s (very fast)
  Snappy:            CR = 0.30–0.40, throughput = 250–400 MB/s
  gzip level 1:      CR = 0.15–0.25, throughput = 100–200 MB/s
  gzip level 6:      CR = 0.12–0.20, throughput = 30–60 MB/s
  gzip level 9:      CR = 0.10–0.18, throughput = 10–20 MB/s
  Zstandard (zstd):  CR = 0.10–0.22, throughput = 300–500 MB/s (best ratio+speed)

Storage cost formula with compression:
  storage_GB_per_day = raw_GB_per_day × CR_algorithm
  For 216 GB/day raw (1000 RPS × 5 logs × 500B):
    LZ4:  216 × 0.30 = 64.8 GB/day
    gzip: 216 × 0.18 = 38.9 GB/day
    zstd: 216 × 0.15 = 32.4 GB/day

CPU overhead formula:
  CPU_overhead_fraction = log_volume_MB_per_s × (1/throughput_MB_per_s) / n_cpu_cores
  For LZ4, 2.5 MB/s logs, 400 MB/s throughput, 4 cores:
    CPU_overhead = 2.5/400 / 4 = 0.16%  (negligible)
  For gzip-6, 2.5 MB/s, 50 MB/s, 4 cores:
    CPU_overhead = 2.5/50 / 4 = 1.25%   (acceptable)
  Recommendation: use LZ4 for hot tier (fast queries), gzip/zstd for cold tier (storage)
```

**Step 2 — ELK stack retention cost formula.**
Total Elasticsearch storage cost with tiered retention:
```
storage_total = Σ_tier (daily_compressed_volume_GB × retention_days_tier × replication_factor_tier)

Typical tiered ELK configuration:
  Hot tier (0–7 days):    replication_factor = 1.5 (primary + 0.5 replica)
  Warm tier (7–30 days):  replication_factor = 1.0 (primary only after shrink)
  Cold tier (30–90 days): replication_factor = 0 (S3/GCS snapshot, no Elasticsearch)

Storage_monthly = (compressed_GB_per_day × 7 × 1.5)   [hot]
               + (compressed_GB_per_day × 23 × 1.0)   [warm]
               + (compressed_GB_per_day × 60 × 0)     [cold: flat S3 cost]

Cost formula:
  ES_storage_cost = storage_TB × cost_per_TB_per_month_USD
  AWS EBS gp3: $0.08/GB/month → $82/TB/month
  S3 cold: $0.023/GB/month → $23.5/TB/month

  For 64.8 GB/day LZ4 compressed logs:
    Hot (7 days): 64.8 × 7 × 1.5 = 680 GB = $54.4/month
    Warm (23 days): 64.8 × 23 = 1490 GB = $119/month
    Cold (60 days): 64.8 × 60 × 0.023 = $89.5/month (S3)
    Total: $263/month for 90-day retention at 1000 RPS
```

**Step 3 — Log sampling rate impact on storage formula.**
Sampling reduces storage cost at the expense of observability coverage:
```
storage_sampled = storage_full × sampling_rate + fixed_overhead

  Fixed overhead: ERROR and WARN logs (always kept, typically 1–5% of volume)
  Variable: INFO and DEBUG logs (candidates for sampling)

  For 95% INFO, 4% WARN, 1% ERROR distribution:
    storage_sampled = V × (0.01×1.0 + 0.04×1.0 + 0.95×s)
                    = V × (0.05 + 0.95s)
    At s=0.10 (10% INFO sampling): storage_sampled = V × (0.05 + 0.095) = 0.145V
    Storage reduction = 1 - 0.145 = 85.5% reduction (vs 100% retention)

P(missing error signal | sampling rate s):
  P(miss) = P(error preceded only by INFO logs) × (1 - s)
  For correlated sampling (keep all logs with error correlation_id):
    P(miss) = 0 for errors (all error-correlated logs retained)
    P(miss) for INFO-only events = 1 - s

Cost-observability tradeoff:
  Observability_score(s) = 1 - P(miss_relevant_debug_info | incident)
                         ≈ s × P(debug_in_INFO) + 1 × P(debug_in_WARN_or_ERROR)
  For typical incident: 70% of debug info in INFO, 30% in WARN/ERROR:
    O(s=0.10) = 0.10 × 0.70 + 1.0 × 0.30 = 0.37 (37% debug coverage)
    O(s=0.50) = 0.50 × 0.70 + 1.0 × 0.30 = 0.65 (65% debug coverage)
    O(s=1.00) = 1.00 × 0.70 + 1.0 × 0.30 = 1.00 (100% debug coverage)
```

---

### M6: India CERT-In Log Retention and DPDP 2023 Compliance Mathematics

**Step 1 — CERT-In mandatory log retention formula (180 days).**
CERT-In April 2022 circular mandates 180-day log retention for all ICT systems in India:
```
Minimum_retention_days = 180 days (CERT-In 2022 — all entities)

Storage for 180-day retention:
  S_180 = compressed_GB_per_day × 180 × replication_factor

  For 64.8 GB/day compressed:
    S_180 (no replication) = 64.8 × 180 = 11,664 GB ≈ 11.4 TB
    S_180 (1× replica) = 11,664 × 2 = 23.3 TB

CERT-In mandated log categories (all must be retained for 180 days):
  [MANDATORY] Access logs: all inbound/outbound connections (with source IP, timestamp)
  [MANDATORY] Authentication logs: all login attempts (success + failure)
  [MANDATORY] System event logs: OS-level events (services start/stop, reboots)
  [MANDATORY] Application logs: all application-level events
  [MANDATORY] Database query logs: for financial/healthcare applications
  [MANDATORY] Network flow logs: for cloud deployments (VPC flow logs, firewall logs)

Sector-specific retention above 180-day baseline:
  RBI regulated (banks, NBFCs):        5 years (IT Master Direction §8.3)
  SEBI regulated (brokers, exchanges): 5 years (SEBI Cyber Security circular)
  IRDAI regulated (insurance):         5 years (IRDAI Information Security Guidelines)
  DPDP Significant Data Fiduciaries:   5 years post-relationship end
  Healthcare (DISHA proposed):         8 years
```

**Step 2 — DPDP 2023 PII log anonymization math.**
Digital Personal Data Protection Act 2023 requires minimizing PII in logs:
```
PII exposure score (risk metric):
  PII_exposure = n_PII_fields_in_logs × avg_log_volume_per_day × retention_days

Anonymization techniques and their effectiveness:
  Masking: replace middle chars with *** → risk reduction = 0.90 (partial reversibility)
    mobile: 9876543210 → 98765***10 (last 4 visible for debugging)
    email:  user@example.com → u***@example.com
  Pseudonymization: replace with consistent token → risk reduction = 0.70
    user_id: "usr-123456" → sha256(usr-123456 + secret_salt)[:16]
  Tokenization: store mapping separately → risk reduction = 0.95 (token useless alone)
  Full removal: PII field not logged at all → risk reduction = 1.00 (no debug value)

DPDP compliance score for logs:
  compliance_score = 1 - (PII_fields_logged_raw / total_PII_fields_accessed_in_code)

  Target: compliance_score ≥ 0.95 (≤5% of PII fields logged without anonymization)
  For financial apps: compliance_score = 1.00 (zero raw PII in logs — RBI mandate)

Anonymization pipeline design:
  Log → anonymization_filter → sanitized_log → storage
  anonymization_filter applies regex substitutions:
    credit_card: r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b' → 'XXXX-XXXX-XXXX-****'
    aadhaar_number: r'\b\d{4}\s?\d{4}\s?\d{4}\b' → '****-****-XXXX'
    pan_number: r'\b[A-Z]{5}\d{4}[A-Z]\b' → '**PAN**'
    mobile: r'\b[6-9]\d{9}\b' → 'XXXX-XX****'
```

**Step 3 — RBI/SEBI audit trail requirements formula.**
```
RBI IT Master Direction 2024 §8.3 — Audit trail completeness formula:
  AT_completeness = logged_events / required_event_types

  Required event types for RBI-regulated entities:
    [+1] All financial transactions (debit/credit entries)
    [+1] User authentication events (login/logout + session tokens)
    [+1] Privileged access events (admin, DBA, super-user)
    [+1] Configuration changes (application + infrastructure)
    [+1] Data access on sensitive tables (customer PII, account data)
    [+1] Export/download events (data exfiltration risk indicator)
    [+1] Failed access attempts (security incident precursor)
    [+1] API access logs (third-party integrations: fintech, payment gateways)

  AT_completeness = covered_event_types / 8
  RBI minimum: AT_completeness = 1.0 (all 8 event types must be logged)

SEBI cybersecurity framework (August 2023 circular) — log monitoring SLA:
  Critical alerts: investigated within 1 hour of log event
  High alerts: investigated within 4 hours
  Medium alerts: investigated within 24 hours
  Log SIEM correlation: automated alerting for:
    - Failed login ≥ 5 in 60 seconds → account lockout + alert
    - Privilege escalation outside business hours → immediate alert
    - Data export > threshold_MB → alert + human review required
    - API error_rate > 5% for 5 consecutive minutes → incident declaration

Cost of non-compliance (IT Act 2000 §43A + DPDP §66):
  Missing CERT-In 180-day logs: penalty up to ₹5 crore per incident
  Missing RBI audit trail: regulatory action under Banking Regulation Act §35A
  DPDP breach without proper logs: penalty up to ₹250 crore (SDF) or ₹50 crore (others)
```

---

## Observability Fundamentals

### 1. The Three Pillars of Observability
- Logs: timestamped, discrete records of events that occurred in the system
  - Best for: detailed debugging, audit trails, event sequences, error diagnosis
  - Cost: high storage, high CPU for ingestion and parsing
- Metrics: numeric measurements aggregated over time
  - Best for: dashboards, alerting, capacity planning, performance trends
  - Cost: low storage (pre-aggregated), efficient querying
- Traces: records of request flow across services with timing and causal relationships
  - Best for: latency diagnosis, dependency mapping, bottleneck identification
  - Cost: medium storage; sampling reduces volume
- All three are needed in production; they complement rather than replace each other

### 2. Observability vs Monitoring
- Monitoring: checking known-bad conditions you anticipated (alert when CPU > 90%)
  - Answers: is the system working as expected?
  - Limitation: cannot diagnose novel failures not anticipated by the alerts
- Observability: the ability to understand the internal state of a system from its outputs
  - Answers: why is the system behaving unexpectedly?
  - Achieved through: rich structured logs, high-cardinality metrics, distributed traces
- High observability enables debugging production issues without accessing production servers

### 3. Log vs Metric vs Trace - When to Use Each
- Use a log when: an event occurred, an error was raised, a decision was made, a user action happened
- Use a metric when: you need to track rates, counts, latency distributions, or resource usage over time
- Use a trace when: you need to understand latency across service calls or identify which service is slow
- Avoid: logging every function call (verbose, expensive), metrics without context (hard to diagnose)
- Rule of thumb: start with structured logs; add metrics for known key performance indicators; add traces for service call paths

---

## Structured Logging

### 4. JSON Format Benefits
- Machine-parseable: log aggregation tools (ELK, Loki, Splunk) can index and query fields directly
- No regex needed: field extraction is automatic when logs are already structured
- Consistent schema: every log entry has the same fields; queries are predictable
- Tooling support: all major logging frameworks support JSON output out of the box
- Human readable with formatting: `jq . log.json` or browser DevTools format JSON nicely

### 5. Standard Log Fields
- Every log entry should include a consistent set of standard fields:

```json
{
  "timestamp": "2026-03-15T10:30:00.123Z",
  "level": "INFO",
  "service": "order-service",
  "version": "2.4.1",
  "environment": "production",
  "message": "Order processed successfully",
  "correlation_id": "req-abc123",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "order_id": "ord-789",
  "user_id": "usr-456",
  "duration_ms": 145,
  "host": "order-service-pod-xyz"
}
```

- `timestamp`: ISO 8601 UTC with millisecond precision; never use local time in production logs
- `level`: log severity (DEBUG, INFO, WARN, ERROR, FATAL)
- `service`: name of the service emitting the log
- `version`: service version for correlating logs with deployments
- `correlation_id`: ties together all log entries for a single request across services
- `trace_id` / `span_id`: OpenTelemetry trace context for linking logs to traces
- `message`: human-readable description of the event (keep concise)
- Business context fields: entity IDs relevant to the operation

### 6. What to Include and Exclude
- Include:
  - Entity IDs (user ID, order ID, session ID)
  - Operation name and outcome
  - Durations and counts for performance-sensitive operations
  - Correlation and trace IDs
  - Non-sensitive contextual metadata
- Exclude (never log these):
  - Passwords, tokens, API keys, credentials
  - Full credit card numbers or financial account numbers
  - Social security numbers, passport numbers
  - Personal health information (PHI)
  - Personally identifiable information (PII) unless required by audit rules and properly protected
  - Full request/response bodies that may contain secrets (log only safe subset)

### 7. Log Enrichment
- Add context fields automatically via middleware or interceptors rather than in every log call
- Web frameworks: add request ID, user ID, path, and HTTP method at the request scope
- Message queue consumers: add message ID, topic, and consumer group at the consumption scope
- Background jobs: add job ID, job type, and schedule at the job scope
- Enrichment ensures every log within a context automatically includes the relevant metadata

---

## Log Levels

### 8. Level Semantics
- DEBUG: detailed diagnostic information for developers; typically disabled in production
  - When to use: variable values during algorithm execution, loop iterations, decision points
  - Examples: "Calculated discount: 15% (rule: first-time-customer)"
- INFO: significant events in normal operation; always enabled in production
  - When to use: service start/stop, user actions, successful key operations, state transitions
  - Examples: "Order ord-123 confirmed", "User logged in", "Scheduled job started"
- WARN: something unexpected happened but the system recovered or is degraded; always enabled
  - When to use: deprecated API usage, retried operation, fallback used, resource nearing limit
  - Examples: "Retry 2/3 for payment gateway", "Cache miss; falling back to database"
- ERROR: an operation failed; the system could not complete the requested action
  - When to use: unhandled exceptions, failed external calls, data integrity issues
  - Examples: "Failed to process payment", "Database query failed"
- FATAL / CRITICAL: the service is unable to continue operating; restart may be required
  - When to use: database connection pool exhausted, critical configuration missing, corruption detected
  - Use sparingly; most errors are not fatal at the service level

### 9. Common Level Mistakes
- Do not log expected not-found results as ERROR: a user fetching a non-existent resource is INFO or WARN
- Do not log validation failures as ERROR: invalid user input is expected; use WARN at most
- Do not log DEBUG in production without dynamic log level control; it will flood aggregation systems
- Do not use WARN for errors that require immediate action; those are ERROR
- Do not set the root logger to DEBUG globally in production; only enable DEBUG for specific packages during investigation

### 10. Dynamic Log Level Configuration
- Configure log levels without restarting the service
- Java Spring Boot: `/actuator/loggers/{logger}` endpoint to change levels at runtime
- Python: update `logging.getLogger('module').setLevel(logging.DEBUG)` via admin endpoint or feature flag
- Feature flags: tie log level to a feature flag that can be toggled remotely
- Always revert elevated log levels after investigation to avoid excessive log volume

---

## Correlation IDs

### 11. Correlation ID Design
- A correlation ID is a unique identifier that ties together all log entries for a single user request or business transaction
- Format: UUID v4 or short prefix + random: `req-a1b2c3d4`, `txn-9x8y7z`
- Generate at the entry point: API gateway, load balancer, or the first service to receive the request
- Propagate through all downstream service calls: HTTP headers, message metadata, async job context
- Standard header name: `X-Correlation-ID`, `X-Request-ID`, or `traceparent` (W3C Trace Context)
- Log it on every log entry within the request scope

### 12. Propagation via HTTP Headers

```python
# FastAPI middleware for correlation ID propagation
import uuid
from fastapi import Request
from contextvars import ContextVar

correlation_id_var: ContextVar[str] = ContextVar('correlation_id', default='')

class CorrelationIdMiddleware:
    async def __call__(self, request: Request, call_next):
        correlation_id = (
            request.headers.get('X-Correlation-ID')
            or request.headers.get('X-Request-ID')
            or f"req-{uuid.uuid4().hex[:12]}"
        )
        correlation_id_var.set(correlation_id)

        response = await call_next(request)
        response.headers['X-Correlation-ID'] = correlation_id
        return response

# Outgoing HTTP calls: propagate to downstream services
async def call_inventory_service(order_id: str):
    async with httpx.AsyncClient() as client:
        return await client.get(
            f"http://inventory-service/stock/{order_id}",
            headers={'X-Correlation-ID': correlation_id_var.get()}
        )
```

### 13. Propagation via Message Queues
- Include the correlation ID in message headers or metadata, not in the message body
- Kafka: message headers (`record.headers().add("correlation-id", correlationId.getBytes())`)
- RabbitMQ: `BasicProperties.Builder.correlationId(correlationId)`
- AWS SQS: message attributes (`MessageAttribute` with `DataType: String`)
- Consumer extracts the correlation ID from headers and sets it in the execution context

```java
// Kafka consumer: extract correlation ID from message headers
@KafkaListener(topics = "orders")
public void handleOrderEvent(ConsumerRecord<String, OrderEvent> record) {
    String correlationId = Optional.ofNullable(record.headers().lastHeader("correlation-id"))
        .map(Header::value)
        .map(String::new)
        .orElse("unknown-" + UUID.randomUUID());

    MDC.put("correlationId", correlationId);
    try {
        processOrder(record.value());
    } finally {
        MDC.clear();
    }
}
```

### 14. W3C Trace Context Header
- `traceparent` header is the standardized format for propagating trace context across services
- Format: `00-{trace-id}-{parent-span-id}-{trace-flags}`
- Example: `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`
  - `00` = version, `4bf9...` = 32-hex trace ID (equivalent to correlation ID), `00f0...` = 16-hex span ID, `01` = sampled flag
- When using OpenTelemetry, the SDK handles propagation automatically via `W3CTraceContextPropagator`
- Prefer `traceparent` for new services; support both `X-Correlation-ID` and `traceparent` for compatibility

---

## Context Propagation

### 15. Thread-Local and Context Variables
- Store request-scoped context (correlation ID, user ID) in a thread-local or async context variable
- Python: `contextvars.ContextVar` (async-safe; propagated across `await` calls)
- Java: `ThreadLocal<T>` for synchronous; `MDC` (Mapped Diagnostic Context) for logging libraries
- Node.js: `AsyncLocalStorage` (async-safe; propagated across Promise chains)
- Context variables are automatically isolated per request; no manual passing through every function

```python
# Python: using contextvars for request context
from contextvars import ContextVar
from dataclasses import dataclass

@dataclass
class RequestContext:
    correlation_id: str
    user_id: str | None
    service_name: str

request_context_var: ContextVar[RequestContext | None] = ContextVar(
    'request_context', default=None
)

def get_log_context() -> dict:
    ctx = request_context_var.get()
    if ctx is None:
        return {}
    return {
        'correlation_id': ctx.correlation_id,
        'user_id': ctx.user_id,
        'service': ctx.service_name,
    }
```

### 16. Java MDC for Logging Context
- MDC (Mapped Diagnostic Context) in SLF4J/Logback stores key-value pairs per thread
- Log patterns include MDC fields: `%X{correlationId}` in pattern layout
- Always clear MDC after request processing to prevent context leakage across requests
- Use try-finally or `MDC.putCloseable()` for automatic cleanup

```java
@Component
public class CorrelationIdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String correlationId = Optional
            .ofNullable(httpRequest.getHeader("X-Correlation-ID"))
            .orElse(UUID.randomUUID().toString());

        MDC.put("correlationId", correlationId);
        MDC.put("path", httpRequest.getRequestURI());

        try {
            ((HttpServletResponse) response).setHeader("X-Correlation-ID", correlationId);
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

### 17. Node.js AsyncLocalStorage
```typescript
import { AsyncLocalStorage } from 'async_hooks';

interface RequestContext {
  correlationId: string;
  userId?: string;
}

const requestStorage = new AsyncLocalStorage<RequestContext>();

export function withRequestContext<T>(
  context: RequestContext,
  fn: () => T | Promise<T>
): T | Promise<T> {
  return requestStorage.run(context, fn);
}

export function getRequestContext(): RequestContext | undefined {
  return requestStorage.getStore();
}

// Express middleware
app.use((req, res, next) => {
  const correlationId = req.headers['x-correlation-id'] as string
    || `req-${randomUUID().replace(/-/g, '').slice(0, 12)}`;
  res.setHeader('X-Correlation-ID', correlationId);
  requestStorage.run({ correlationId, userId: req.user?.id }, next);
});
```

---

## Log Aggregation

### 18. ELK Stack (Elasticsearch, Logstash, Kibana)
- Elasticsearch: distributed search and analytics engine; stores and indexes log documents
- Logstash: log collection, parsing, transformation, and routing pipeline
- Kibana: visualization and dashboarding for Elasticsearch data
- Filebeat: lightweight agent that ships logs from hosts to Logstash or Elasticsearch
- Index template: define field mappings (keyword vs text, date formats) for efficient querying
- Index lifecycle management (ILM): automatically roll over, shrink, and delete indices based on age/size
- Typical pipeline: App -> JSON stdout -> Filebeat -> Logstash (enrich/filter) -> Elasticsearch -> Kibana

### 19. Loki + Grafana (Cloud-Native Alternative)
- Loki: log aggregation system designed for cloud-native environments
- Unlike Elasticsearch, Loki does not index log content; only indexes labels (service, environment, level)
- Lower resource usage than ELK; excellent choice for Kubernetes-native deployments
- LogQL: query language for filtering and aggregating log streams
- Promtail: log collection agent that ships logs to Loki
- Grafana: unified dashboarding for Loki logs, Prometheus metrics, and Jaeger traces in one UI
- Log-derived metrics in Loki: extract numeric metrics from log patterns with `rate()` queries

### 20. Log Query Patterns
- Filter by correlation ID to trace a single request: `{service="order-service"} |= "req-abc123"`
- Count errors per service over 5 minutes: `sum by (service) (rate({level="ERROR"}[5m]))`
- Find slow requests: `{service="api"} | json | duration_ms > 1000 | line_format "{{.message}} {{.duration_ms}}ms"`
- Find recent errors with order context: `{level="ERROR"} | json | order_id != ""`
- Build log-based metrics: derive numeric metrics from log patterns without separate instrumentation

### 21. Log Retention and Cost Management
- Hot tier: last 7-30 days for active investigation and dashboards
- Warm tier: 30-90 days for compliance and post-incident review
- Cold tier (S3 / GCS): 1-7 years for regulatory compliance and audit
- Tiered storage reduces costs; keep recent logs hot, older logs on cheap object storage
- Log sampling at INFO level for health check requests: 1% vs 100% for business events
- Filter out internal polling and heartbeat logs at the agent level before shipping to aggregation

---

## Metrics Design

### 22. Metric Types
- Counter: monotonically increasing number; reset on service restart
  - Use for: request count, error count, events processed, bytes transferred
  - Query: `rate(http_requests_total[5m])` for requests per second
- Gauge: value that can increase or decrease
  - Use for: current queue depth, active connections, memory usage
  - Query: `queue_depth` directly; `delta(queue_depth[1m])` for rate of change
- Histogram: samples observations and counts them in configurable buckets
  - Use for: request latency, response size, payload size
  - Query: `histogram_quantile(0.95, rate(http_duration_seconds_bucket[5m]))` for p95 latency
- Summary: similar to histogram but quantiles calculated on the client; cannot aggregate across instances
  - Prefer histograms in almost all cases; summaries cannot be federated or aggregated in Prometheus

### 23. Metric Naming Conventions
- Follow Prometheus naming conventions for cross-tool compatibility
- Format: `namespace_subsystem_name_unit`
- Examples:
  - `http_requests_total` (counter)
  - `http_request_duration_seconds` (histogram)
  - `db_connections_active` (gauge)
  - `queue_messages_pending` (gauge)
  - `cache_hits_total` (counter)
- Units in the metric name: `_seconds`, `_bytes`, `_total`, `_ratio`
- Use `_total` suffix for all counters per Prometheus convention
- Never abbreviate in ways that lose meaning: `req_dur_s` is worse than `request_duration_seconds`

### 24. Label Cardinality
- Labels add dimensions to metrics; each unique label value combination creates a new time series
- Low-cardinality labels are safe: HTTP status code (200, 400, 500), method (GET, POST), service name, environment
- High-cardinality labels are dangerous and must be avoided:
  - DO NOT label on: user ID, order ID, request ID, email address, IP address
  - High cardinality explodes storage and query performance; this is a common production disaster
- Example budget: 5 services x 10 endpoints x 6 status codes = 300 time series (safe)
- Example disaster: 5 services x 1,000,000 user IDs = 5,000,000 time series (catastrophic)

```python
# Prometheus Python client - correct label usage
from prometheus_client import Counter, Histogram

REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['service', 'method', 'endpoint', 'status']  # All low-cardinality
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['service', 'method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

# Usage in a request handler
REQUEST_COUNT.labels(
    service='order-service',
    method='POST',
    endpoint='/orders',
    status='201'
).inc()

with REQUEST_DURATION.labels(
    service='order-service',
    method='POST',
    endpoint='/orders'
).time():
    result = process_order(order_data)
```

### 25. Four Golden Signals
- Latency: time to service a request; track p50, p95, p99 not just averages
  - Averages hide long tails; a p99 of 5s with a p50 of 10ms indicates a serious tail latency problem
  - Alert on: p99 latency exceeding your SLO threshold
- Traffic: request rate (RPS), events per second, messages processed per second
  - Alert on: unexpected drops (service down or routing issue) or unexpected spikes (attack or bug)
- Errors: error rate as a percentage of total traffic
  - Track 4xx (client errors) and 5xx (server errors) separately
  - Alert on: 5xx error rate exceeding your SLO threshold (e.g., > 0.1%)
- Saturation: how full the service is; utilization approaching capacity
  - Track: CPU percentage, memory percentage, queue depth, connection pool utilization
  - Alert on: saturation > 80% sustained; this predicts imminent failure

---

## Sampling Strategies

### 26. Why Sampling
- High-traffic systems may generate millions of requests per minute
- Tracing every request at 100% is cost-prohibitive at scale
- Logging every request at DEBUG level floods the aggregation pipeline
- Sampling reduces volume while preserving the ability to diagnose issues
- Goal: keep enough data to answer questions, discard data that will never be queried

### 27. Head-Based Sampling
- Sampling decision made at the start of the request, before any processing occurs
- The decision propagates through all downstream service calls (all or nothing per trace)
- Simple to implement; produces consistent full traces when a request is sampled
- Limitation: cannot selectively keep traces based on what happened (errors, slow spans)
- Techniques:
  - Fixed rate: sample 1% of all requests uniformly
  - Consistent sampling by ID: hash user or session ID to keep all requests for a sampled entity
  - Reservoir sampling: guarantee N samples per time period regardless of traffic volume

### 28. Tail-Based Sampling
- Sampling decision made after the request completes, based on what happened
- Keep all: error traces, traces exceeding latency thresholds (p99), traces with new error types
- Discard most: successful fast requests at a high rate (e.g., keep 0.1%)
- Requires a buffer to hold all spans until the trace completes (typically 30-60 seconds)
- More complex to implement but produces a much more actionable sample set
- Tools: OpenTelemetry Collector with tail-sampling processor, Grafana Tempo

### 29. Sampling by Log Level
- Always keep: ERROR and FATAL entries (low volume, highest diagnostic value)
- Conditionally keep: WARN entries (moderate volume; keep all or sample at 10-100%)
- Sample aggressively: INFO entries for high-frequency operations (health checks at 0.1%)
- Gate entirely: DEBUG entries in production behind a feature flag; never unconditionally log DEBUG

```python
import random
import logging

class AdaptiveSamplingFilter(logging.Filter):
    """Samples INFO logs at a configured rate; always passes WARN and above."""

    SAMPLE_RATES = {
        'health_check': 0.001,  # 0.1% of health checks
        'cache_hit': 0.01,       # 1% of cache hits
        'default': 0.10,         # 10% of other INFO messages
    }

    def filter(self, record: logging.LogRecord) -> bool:
        if record.levelno >= logging.WARNING:
            return True  # Always keep WARN and above

        operation = getattr(record, 'operation', 'default')
        rate = self.SAMPLE_RATES.get(operation, self.SAMPLE_RATES['default'])
        return random.random() < rate
```

---

## Language-Specific: Python

### 30. Python Logging Module Setup with JSON
```python
import logging
import json
import sys
from contextvars import ContextVar

correlation_id_var: ContextVar[str] = ContextVar('correlation_id', default='')

class JSONFormatter(logging.Formatter):
    """Formats log records as structured JSON with context fields."""

    def format(self, record: logging.LogRecord) -> str:
        log_data = {
            'timestamp': self.formatTime(record, '%Y-%m-%dT%H:%M:%S') + 'Z',
            'level': record.levelname,
            'service': 'my-service',
            'version': '1.0.0',
            'message': record.getMessage(),
            'logger': record.name,
            'correlation_id': correlation_id_var.get(''),
        }
        # Include extra fields passed via logger.info(..., extra={...})
        reserved = logging.LogRecord.__dict__.keys()
        for key, value in record.__dict__.items():
            if key not in reserved and not key.startswith('_'):
                log_data[key] = value

        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)

        return json.dumps(log_data)


def configure_logging(level: str = 'INFO') -> None:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JSONFormatter())
    root = logging.getLogger()
    root.setLevel(getattr(logging, level.upper()))
    root.handlers = [handler]


# Usage
logger = logging.getLogger(__name__)
logger.info("Order processed", extra={'order_id': 'ord-123', 'duration_ms': 145})
```

### 31. Python Structured Logging with structlog
- `structlog` is a production-grade structured logging library for Python
- Processors pipeline: chain of callables that transform the log event before output
- Integrates with the standard `logging` module for third-party library compatibility

```python
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,  # Merge ContextVar values automatically
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt='iso'),
        structlog.processors.StackInfoRenderer(),
        structlog.dev.ConsoleRenderer() if DEV_MODE else structlog.processors.JSONRenderer(),
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Bind context that appears on all subsequent calls from this logger instance
bound = logger.bind(order_id='ord-123', user_id='usr-456')
bound.info("Processing payment")
bound.info("Payment confirmed", amount=99.99, currency='USD')

# Using contextvars for auto-binding across function calls
structlog.contextvars.bind_contextvars(correlation_id='req-abc')
logger.info("Any log here will include correlation_id automatically")
```

---

## Language-Specific: Java

### 32. Java SLF4J and Logback Configuration
```xml
<!-- logback-spring.xml -->
<configuration>
  <appender name="JSON_STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <customFields>{"service":"order-service","environment":"${ENVIRONMENT:-local}"}</customFields>
      <fieldNames>
        <timestamp>timestamp</timestamp>
        <message>message</message>
        <logger>logger</logger>
        <thread>thread</thread>
        <levelValue>[ignore]</levelValue>
      </fieldNames>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="JSON_STDOUT"/>
  </root>
</configuration>
```

```java
// SLF4J with structured arguments via logstash-logback-encoder
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import static net.logstash.logback.argument.StructuredArguments.*;

public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public Order processOrder(String orderId) {
        long start = System.currentTimeMillis();
        log.info("Processing order",
            keyValue("orderId", orderId),
            keyValue("correlationId", MDC.get("correlationId"))
        );

        try {
            Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));

            log.info("Order processed",
                keyValue("orderId", orderId),
                keyValue("durationMs", System.currentTimeMillis() - start)
            );
            return order;

        } catch (OrderNotFoundException e) {
            log.warn("Order not found", keyValue("orderId", orderId));
            throw e;
        } catch (Exception e) {
            log.error("Failed to process order",
                keyValue("orderId", orderId),
                keyValue("durationMs", System.currentTimeMillis() - start),
                e
            );
            throw e;
        }
    }
}
```

---

## Language-Specific: TypeScript

### 33. TypeScript Structured Logging with pino
- `pino` is the fastest Node.js logging library; outputs JSON natively
- `pino-http` middleware for automatic request/response logging
- Child loggers for adding context to a specific scope without affecting the root logger

```typescript
import pino from 'pino';
import { AsyncLocalStorage } from 'async_hooks';
import { randomUUID } from 'crypto';

const requestStorage = new AsyncLocalStorage<Record<string, string>>();

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  base: {
    service: 'order-service',
    version: process.env.APP_VERSION || 'unknown',
    environment: process.env.NODE_ENV || 'development',
  },
  formatters: {
    level: (label) => ({ level: label }),
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  mixin() {
    return requestStorage.getStore() || {};
  },
});

// Express/Fastify middleware
export function requestContextMiddleware(req: any, res: any, next: () => void) {
  const correlationId = (req.headers['x-correlation-id'] as string)
    || `req-${randomUUID().replace(/-/g, '').slice(0, 12)}`;

  res.setHeader('X-Correlation-ID', correlationId);

  requestStorage.run({ correlationId, userId: req.user?.id || '' }, () => {
    next();
  });
}

// Usage in services
logger.info({ orderId: 'ord-123', amount: 99.99 }, 'Order created');
logger.error({ orderId: 'ord-123', err: error }, 'Payment processing failed');

// Child logger for module-specific context
const orderLogger = logger.child({ module: 'OrderService' });
orderLogger.info({ orderId }, 'Order status updated');
```

### 34. TypeScript Logging Anti-Patterns
```typescript
// BAD: log and throw causes duplicate error entries
try {
  await processPayment(order);
} catch (error) {
  logger.error({ err: error }, 'Payment failed');  // Logged here
  throw error;  // Also logged by the global handler
}

// GOOD: log OR throw; let the global handler log uncaught errors
try {
  await processPayment(order);
} catch (error) {
  if (error instanceof ValidationError) {
    logger.warn({ err: error }, 'Payment validation failed');  // Handled; log and return
    return { success: false, reason: error.message };
  }
  throw error;  // Unexpected; propagate for global handler to log
}
```

---

## Anti-Patterns to Avoid

### Common Mistakes to Avoid
- Silent swallowing: `except: pass` or `catch (e) {}` without any logging; errors disappear silently
- Log and throw: logging an error AND re-throwing it causes duplicate entries in aggregation
- Over-logging at INFO: logging every cache hit, every DB query; floods the pipeline and hides real events
- Unstructured messages: `log.info(f"User {user_id} placed order {order_id}")` cannot be queried by field
- Missing context: `log.error("Request failed")` with no correlation ID, no entity IDs, no useful context
- Sensitive data: logging passwords, tokens, or PII violates compliance and security requirements
- String concatenation in log calls: `log.debug("Items: " + str(items))` builds the string even if DEBUG is filtered

### Deep Mathematical Anti-Patterns
- **Using high-cardinality fields as metric labels**: attaching `user_id` or `request_id` as a Prometheus label instead of a log field triggers the cardinality explosion in M1 — 1M users × 10 metrics becomes 10M time series at ~1KB each, roughly 10GB of memory pressure for data that belongs in structured logs, not metric dimensions.
- **Under-provisioning correlation-ID entropy**: choosing a short or low-entropy ID scheme instead of the 64-bit-minimum bound from M2's birthday-paradox analysis risks real collisions at scale — at 1M requests/day a 64-bit ID keeps `P(collision/day) ≈ 10^-6`, but a narrower ID space does not.
- **Blocking the application to guarantee log delivery**: configuring a synchronous logger, or an async logger that blocks on queue overflow instead of dropping with a counter, converts M3's ~10-100µs sync I/O cost into application back-pressure — the recommended policy is drop-and-count, never block.
- **Fixed head-sampling regardless of endpoint traffic**: applying one flat sample rate (e.g. 1%) uniformly across high- and low-traffic endpoints, instead of the adaptive per-endpoint scheme in M4, silently drops the only samples a low-traffic endpoint would ever produce — `P(capture error | adaptive) >> P(capture error | fixed rate)` for exactly those endpoints.
- **Sampling INFO logs without correlation-aware retention**: applying a flat sampling rate to all log levels instead of M5's correlated-sampling rule (always keep every log carrying an error's `correlation_id`) turns `P(miss)` for error-adjacent context from 0 into `1 - s` — the storage saving is real, but so is the debugging blind spot it creates.
- **Choosing gzip level 9 for hot-tier logs**: picking the highest compression ratio without checking the M5 Step-1 throughput table trades a 10-20 MB/s write path for a marginal storage gain over LZ4 or zstd (300-600 MB/s) — reserve the slowest, tightest compression for cold tier only.
- **Treating CERT-In's 180-day retention as sector-uniform**: applying the CERT-In 2022 baseline to RBI-, SEBI-, or IRDAI-regulated data instead of the sector-specific 5-year (or 8-year healthcare) floor in M6 under-retains exactly the logs subject to the strictest audit requirement.
- **Logging PII in raw form "for debugging" instead of masking**: skipping the M6 anonymization pipeline (masking, pseudonymization, or tokenization) for fields like mobile numbers, Aadhaar, or PAN pushes the DPDP compliance_score below the 0.95 target — and to 1.00-required for financial apps under the RBI mandate — turning a debugging convenience into a ₹250-crore-exposure compliance gap.

```python
# BAD: unstructured, no context
logger.error(f"Failed to process {order_id}")

# GOOD: structured, with full context
logger.error(
    "Order processing failed",
    extra={
        'order_id': order_id,
        'user_id': user_id,
        'correlation_id': correlation_id_var.get(),
        'error_type': type(e).__name__,
        'duration_ms': elapsed_ms,
    },
    exc_info=True  # Includes stack trace
)
```

---

## Response Rules

- Always recommend JSON structured logging for production systems
- Include correlation ID propagation in every service and async context design
- Specify log levels with clear semantic guidance, not just definitions
- Address high-cardinality label risk whenever metrics are being designed
- Include sampling strategy recommendation when the system is high-volume (>1000 RPS)
- Provide the specific logging library and configuration snippet for the user's stack

---

## What Not to Do

- Do not log sensitive data: passwords, tokens, PII, payment card data, personal health information
- Do not use string concatenation or f-strings in log calls; use structured fields
- Do not set DEBUG level globally in production; it will flood the aggregation pipeline
- Do not label Prometheus metrics with user IDs, order IDs, or request IDs; high cardinality destroys performance
- Do not log without a correlation ID in a distributed system; cross-service debugging becomes impossible
- Do not forget to clear MDC in Java finally blocks; stale MDC context leaks between thread pool requests
- Do not use print statements for logging in production code; use a proper logging framework

---

## Output Expectations

- Complete logger configuration with JSON formatter for the specified language and framework
- Correlation ID middleware implementation with HTTP header and message queue propagation
- Prometheus metrics definition with correct metric type, name, labels, and histogram buckets
- Sampling configuration appropriate for the specified traffic volume
- Logback XML, pino config, or Python logging.dictConfig as applicable
- Log aggregation query examples (Kibana KQL or LogQL) for common debugging scenarios

---

## Skill Scope

In scope:
- Structured logging (JSON format) design and implementation
- Log level semantics and selection guidance for production
- Correlation ID generation and propagation via HTTP headers and message queues
- Context propagation using MDC (Java), contextvars (Python), AsyncLocalStorage (Node.js)
- Log aggregation with ELK Stack and Loki + Grafana
- Metrics design: counters, gauges, histograms; naming conventions; cardinality management
- Sampling strategies: head-based, tail-based, adaptive, level-based sampling
- Language-specific: Python (logging module, structlog), Java (SLF4J, Logback, MDC), TypeScript (pino, winston)

Out of scope:
- Distributed tracing instrumentation beyond correlation ID and trace context propagation
- Alerting rule configuration and on-call rotation management
- Log-based security incident detection (SIEM)
- Infrastructure metrics (host CPU, disk, network) beyond application-level metrics
- Frontend logging and browser telemetry
- Cloud-provider-specific logging service configuration (CloudWatch, Stackdriver pricing)

---

## Version
2.0.1 — Renamed Logging Anti-Patterns to Anti-Patterns to Avoid; added Deep Mathematical Anti-Patterns grounded in M1-M6: cardinality explosion, correlation-ID entropy, blocking log delivery, fixed head-sampling, uncorrelated INFO sampling, wrong compression tier, sector-uniform retention assumption, raw PII logging
2.0.0
