# mcp-jenkins-ci

A FastMCP server providing **Jenkins Ci** capabilities for Claude Code workflows.

---

## Overview

Jenkins CI/CD integration via REST API. Supports Basic auth with API tokens (CSRF-exempt endpoints), parameterized builds, build status polling, console log streaming, and queue management. Pure stdlib urllib — no jenkinsapi dependency.

---

## Tools

| Tool | Description |
|------|-------------|
| `jenkins_trigger_build` | Trigger a build (with optional parameters dict) |
| `jenkins_get_build_status` | Get build result: SUCCESS/FAILURE/ABORTED/IN_PROGRESS |
| `jenkins_get_console_output` | Stream console log for a build (tail N lines) |
| `jenkins_list_jobs` | List all jobs in a folder or root |
| `jenkins_get_job_info` | Get job config: description, parameters, last build |
| `jenkins_list_builds` | List recent builds with timestamps and durations |
| `jenkins_abort_build` | Abort a running build |
| `jenkins_get_queue_info` | Get queue items (blocked, pending, why-blocked reason) |
| `jenkins_wait_for_build` | Poll until build completes (configurable timeout/interval) |
| `jenkins_health_check` | Verify Jenkins connectivity and API token validity |

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/techdeveloper-org/mcp-jenkins-ci.git
cd mcp-jenkins-ci
```

### 2. Install dependencies

```bash
pip install mcp fastmcp
```

### 3. Configure environment

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

---

## Configuration

### Environment Variables

| Variable | Description |
|----------|-------------|
| `JENKINS_URL` | Jenkins server URL (required) |
| `JENKINS_USER` | Jenkins username for Basic auth |
| `JENKINS_TOKEN` | Jenkins API token (preferred over password) |
| `JENKINS_CRUMB_DISABLED` | Set to 1 if CSRF crumbs are disabled on server |

---

## Usage in Claude Code

Add to your `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "jenkins-ci": {
      "command": "python",
      "args": [
        "/path/to/mcp-jenkins-ci/server.py"
      ],
      "env": {}
    }
  }
}
```

---

## Benefits

- Pipeline-native CI integration: trigger build after PR, wait, validate before merge
- Console log streaming helps debug pipeline failures without opening Jenkins UI
- Queue visibility explains why builds are blocked (agent availability, etc.)
- Pure stdlib — no heavy jenkinsapi package, works in restricted environments

---

## Requirements

- Python 3.8+
- `mcp fastmcp`
- See `requirements.txt` for pinned versions

---

## Project Context

This MCP server is part of the **Claude Workflow Engine** ecosystem — a LangGraph-based
orchestration pipeline for automating Claude Code development workflows.

Related repos:
- [`claude-workflow-engine`](https://github.com/techdeveloper-org/claude-workflow-engine) — Main pipeline
- [`mcp-base`](https://github.com/techdeveloper-org/mcp-base) — Shared base utilities used by all MCP servers

---

## License

Private — techdeveloper-org
