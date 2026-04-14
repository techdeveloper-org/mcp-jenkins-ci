# mcp-jenkins-ci

![Python 3.8+](https://img.shields.io/badge/Python-3.8%2B-blue)
![License: MIT](https://img.shields.io/badge/License-MIT-green)
![Part of claude-workflow-engine](https://img.shields.io/badge/Part%20of-claude--workflow--engine-orange)

A FastMCP server that gives Claude direct access to Jenkins CI/CD via the Jenkins REST API. Trigger builds, stream console output, manage the build queue, poll build status, and abort running builds — all over stdio JSON-RPC with no external Python dependencies beyond FastMCP itself.

This server is one of 13 MCP servers in the [claude-workflow-engine](https://github.com/techdeveloper-org/claude-workflow-engine) ecosystem. When enabled, it gates pull request merges in Step 11 of the pipeline on a successful Jenkins build.

---

## Features

- Trigger parameterized and parameterless Jenkins builds
- Get build status for any build number or the latest build
- Stream console output (last 10,000 characters) for any build
- List jobs at the top level or within a folder
- Get detailed job info including health score, recent builds, and queue state
- List recent builds for a job with result and duration
- Abort a running build by job name and build number
- Inspect the live build queue across the entire Jenkins instance
- Poll a build until it completes or a timeout is reached
- Verify Jenkins connectivity and retrieve server metadata
- Supports folder paths (`folder/subfolder/job`) and URL-encodes all path components
- SSL verification can be disabled for self-signed certificates

---

## Tool Reference

| Tool | Description | Key Parameters |
|------|-------------|----------------|
| `jenkins_trigger_build` | Trigger a build for a job. Uses `/buildWithParameters` when parameters are provided, `/build` otherwise. | `job_name` (str), `parameters` (optional str, e.g. `"BRANCH=main&ENV=staging"`) |
| `jenkins_get_build_status` | Get status of a specific build: result, building flag, duration, URL, timestamp. | `job_name` (str), `build_number` (int; use `0` for last build) |
| `jenkins_get_console_output` | Fetch the console log for a build. Truncates to last 10,000 characters if larger. | `job_name` (str), `build_number` (int; use `0` for last build) |
| `jenkins_list_jobs` | List all jobs. Optionally filter to a specific folder. | `folder` (optional str; omit for top-level jobs) |
| `jenkins_get_job_info` | Get detailed job info: description, buildable flag, health score, last build numbers, recent builds. | `job_name` (str) |
| `jenkins_list_builds` | List recent builds for a job with result, duration, and timestamp. | `job_name` (str), `max_builds` (int, default `10`, max `50`) |
| `jenkins_abort_build` | Stop a running build immediately. | `job_name` (str), `build_number` (int; use `0` for last build) |
| `jenkins_get_queue_info` | Get the current build queue: item IDs, job names, blocked status, wait reason. | _(no parameters)_ |
| `jenkins_wait_for_build` | Poll a build every 5 seconds until it finishes or the timeout is reached. | `job_name` (str), `build_number` (int; use `0` for last build), `timeout_seconds` (int, default `300`) |
| `jenkins_health_check` | Verify Jenkins is reachable. Returns version, executor count, mode, and node name. | _(no parameters)_ |

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/techdeveloper-org/mcp-jenkins-ci.git
cd mcp-jenkins-ci
```

### 2. Install dependencies

```bash
pip install fastmcp
```

The server uses Python's standard library (`urllib`, `ssl`, `base64`, `json`) for all Jenkins API calls. No additional HTTP client libraries are required.

### 3. Register in Claude Code

Add the server to your `~/.claude/settings.json` under `mcpServers`:

```json
{
  "mcpServers": {
    "jenkins-ci": {
      "command": "python",
      "args": ["/path/to/mcp-jenkins-ci/server.py"],
      "env": {
        "JENKINS_URL": "https://jenkins.your-company.com",
        "JENKINS_USER": "your-username",
        "JENKINS_API_TOKEN": "your-api-token"
      }
    }
  }
}
```

Restart Claude Code after saving.

---

## Configuration

The server reads four environment variables at startup. Set them in the `env` block of your `settings.json` or export them before running the server manually.

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `JENKINS_URL` | Yes | — | Base URL of your Jenkins instance, e.g. `https://jenkins.company.com` |
| `JENKINS_USER` | Yes | — | Jenkins username for authentication |
| `JENKINS_API_TOKEN` | Yes | — | Jenkins API token (not your password). Tokens bypass CSRF crumb requirements. Generate one at `<JENKINS_URL>/user/<username>/configure`. |
| `JENKINS_VERIFY_SSL` | No | `true` | Set to `false` to skip SSL verification for self-signed certificates |

To enable this server within the claude-workflow-engine pipeline, also set:

```bash
ENABLE_JENKINS=1
```

---

## Usage Examples

### Trigger a build

```
Trigger a build for the "backend-api" job with BRANCH=main and ENV=staging.
```

Claude calls `jenkins_trigger_build` with:
- `job_name`: `"backend-api"`
- `parameters`: `"BRANCH=main&ENV=staging"`

Returns the queue URL where the build is waiting to be scheduled.

---

### Check build status

```
What is the status of build 47 for the "backend-api" job?
```

Claude calls `jenkins_get_build_status` with:
- `job_name`: `"backend-api"`
- `build_number`: `47`

Returns `result` (`SUCCESS`, `FAILURE`, `ABORTED`, or `null` if still running), `building`, `duration_ms`, and the build URL.

---

### Stream console output

```
Show me the console output for the last build of "infra/deploy-prod".
```

Claude calls `jenkins_get_console_output` with:
- `job_name`: `"infra/deploy-prod"`
- `build_number`: `0`

Returns the last 10,000 characters of the build log. If the log was truncated, `truncated: true` and `total_chars` indicate the full size.

---

### Wait for a build to finish

```
Wait for build 52 of "backend-api" to complete. Timeout after 10 minutes.
```

Claude calls `jenkins_wait_for_build` with:
- `job_name`: `"backend-api"`
- `build_number`: `52`
- `timeout_seconds`: `600`

Polls every 5 seconds. Returns `result`, `timed_out`, and `elapsed_seconds` when the build finishes or the timeout is reached.

---

## Pipeline Integration

When `ENABLE_JENKINS=1`, this server participates in **Step 11** of the claude-workflow-engine pipeline (PR creation and code review).

**Step 11 flow with Jenkins enabled:**

1. Implementation is complete (Step 10). A pull request is opened against the target branch.
2. Step 11 calls `jenkins_trigger_build` to start the CI build for the PR branch.
3. Step 11 calls `jenkins_wait_for_build` to block until the build completes.
4. If `result == "SUCCESS"`, the pipeline proceeds to post the build status on the PR and moves toward merge.
5. If `result != "SUCCESS"`, the pipeline surfaces the failure and halts automatic merge, prompting the user to inspect the console output or abort.

This pattern ensures no PR is merged without a passing Jenkins build. The build result is also linked back to the associated Jira issue when `ENABLE_JIRA=1` is set alongside `ENABLE_JENKINS=1`.

---

## Part of the claude-workflow-engine Ecosystem

This server is one of 13 MCP servers used by [claude-workflow-engine](https://github.com/techdeveloper-org/claude-workflow-engine), a LangGraph orchestration pipeline for automating Claude Code development workflows.

| # | Server | Purpose |
|---|--------|---------|
| 1 | mcp-session-mgr | Session lifecycle management |
| 2 | mcp-git-ops | Git branch, commit, push, diff |
| 3 | mcp-github-api | PR creation, merge, issue management |
| 4 | mcp-policy-enforcement | Policy compliance and system health |
| 5 | mcp-token-optimizer | Token reduction (AST navigation, smart read) |
| 6 | mcp-pre-tool-gate | Pre-tool validation and policy checks |
| 7 | mcp-post-tool-tracker | Post-tool progress tracking |
| 8 | mcp-standards-loader | Coding standards detection and loading |
| 9 | mcp-uml-diagram | UML diagram generation (13 types) |
| 10 | mcp-drawio-diagram | Draw.io editable diagram generation |
| 11 | mcp-jira-api | Jira issue create, transition, link |
| **12** | **mcp-jenkins-ci** | **Jenkins CI/CD (this server)** |
| 13 | mcp-figma | Figma design tokens and component extraction |

All servers share the `mcp-base` package for consistent response formatting, error handling, and tool decoration.

---

## Contributing

Contributions are welcome. Please open an issue to discuss the change before submitting a pull request. All changes must include:

- Type hints on all function signatures
- Docstrings for any new tool or helper
- ASCII-only source (cp1252 compatible for Windows)
- No new external dependencies — stdlib only for Jenkins API calls

---

## License

MIT License. See [LICENSE](LICENSE) for details.
