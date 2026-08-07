# mcp-jenkins-ci — Claude Project Context

**Type:** FastMCP Server
**Transport:** stdio
**Python:** 3.8+

---

## What This Server Does

Jenkins CI/CD integration via REST API. Supports Basic auth with API tokens (CSRF-exempt endpoints), parameterized builds, build status polling, console log streaming, and queue management. Pure stdlib urllib — no jenkinsapi dependency.

---

## Entry Point

```
server.py
```

Run via `python server.py` — communicates over stdio using the MCP protocol.

---

## Available Tools

- `jenkins_trigger_build` — Trigger a build (with optional parameters dict)
- `jenkins_get_build_status` — Get build result: SUCCESS/FAILURE/ABORTED/IN_PROGRESS
- `jenkins_get_console_output` — Stream console log for a build (tail N lines)
- `jenkins_list_jobs` — List all jobs in a folder or root
- `jenkins_get_job_info` — Get job config: description, parameters, last build
- `jenkins_list_builds` — List recent builds with timestamps and durations
- `jenkins_abort_build` — Abort a running build
- `jenkins_get_queue_info` — Get queue items (blocked, pending, why-blocked reason)
- `jenkins_wait_for_build` — Poll until build completes (configurable timeout/interval)
- `jenkins_health_check` — Verify Jenkins connectivity and API token validity

---

## Shared Utilities (in this repo)

- `base/` — Shared MCP infrastructure package (response builder, decorators, persistence, clients)
- `mcp_errors.py` — Structured error response helpers
- `input_validator.py` — Null-byte strip, length limits, prompt injection detection
- `rate_limiter.py` — Token bucket rate limiter (enable via ENABLE_RATE_LIMITING=1)
- `idempotency.py` — Atomic (O_CREAT|O_EXCL) idempotency memo backing the optional
                       `idempotency_key` parameter on non-idempotent create/trigger tools

---

## Environment Variables

- `JENKINS_URL` — Jenkins server URL (required)
- `JENKINS_USER` — Jenkins username for Basic auth
- `JENKINS_TOKEN` — Jenkins API token (preferred over password)
- `JENKINS_CRUMB_DISABLED` — Set to 1 if CSRF crumbs are disabled on server

---

## Development

### Running locally

```bash
# Install deps
pip install -r requirements.txt

# Run the MCP server (stdio mode)
python server.py
```

### Testing a tool call manually

```python
import subprocess, json

proc = subprocess.Popen(
    ["python", "server.py"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
)
# Send MCP initialize + tool call via stdin
```

### File structure

```
mcp-jenkins-ci/
+-- server.py          # Main FastMCP server (entry point)
+-- base/              # Shared base package (response, decorators, persistence, clients)
+-- mcp_errors.py      # Error helpers
+-- input_validator.py # Input validation
+-- rate_limiter.py    # Rate limiting
+-- requirements.txt
+-- .gitignore
+-- README.md
+-- CLAUDE.md
```

---

## Key Rules

1. Do NOT edit `base/` directly — it is a copy from `mcp-base` repo
2. To update shared utilities, edit in `mcp-base` and re-copy
3. Keep `server.py` as the single entry point
4. All tool handlers must use `@mcp_tool_handler` decorator for consistent error handling
5. All responses must use `success()` / `error()` / `MCPResponse` builder from `base.response`

---

**Last Updated:** 2026-03-31
