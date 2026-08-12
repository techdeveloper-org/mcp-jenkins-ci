---
name: linux-shell-scripting
version: 2.0.0
last-updated: 2026-03-07
description: "Linux/Unix shell scripting (Bash, sh, zsh) - system automation, file processing, process management, networking, cron jobs, text manipulation, and production-grade script patterns. Use when writing Bash/sh/zsh scripts, automating Linux system tasks, file processing, cron jobs, or production shell tooling. Keywords: bash script, shell script, cron job, Linux command, pipe and redirect, sed awk, shell automation, chmod."
allowed-tools: Read,Glob,Grep,Bash,Edit,Write
user-invocable: true
---


<!-- generated-by: claude-global-library/tools/project_sync -->
<!-- source: skills/linux-shell-scripting/SKILL.md -- edit the library, then re-run sync_project.py -->

# SKILL.md
## Linux Shell Scripting Skill Instructions

### Skill Name
linux-shell-scripting

### Description
Comprehensive skill for writing production-grade Linux/Unix shell scripts. Covers Bash scripting fundamentals, POSIX compliance, system automation, file processing, process management, networking utilities, cron scheduling, text manipulation (awk/sed/grep), error handling, and security best practices.

---

## Mathematical Foundations

### M1: Shell Process & Subshell Performance Math
```
Process spawning cost:
  fork(): ~0.1-1ms (copy-on-write, fast)
  exec(): ~1-5ms (program loading overhead)
  full fork+exec (external command): ~2-10ms total

  Subshell vs function call:
    $(command): spawns subshell + fork + exec = 2-10ms overhead
    function call: O(1) in current process = nanoseconds
    in tight loop: prefer functions over $() for performance
    1000 iterations × 5ms subshell = 5 seconds vs <1ms with functions

  Pipe chain performance:
    cmd1 | cmd2: each command runs in separate subshell concurrently
    pipe buffer: 64KB (Linux default) — if full, cmd1 blocks
    long pipeline: adds latency per stage but enables parallelism
    e.g., cat file | grep | awk | sort | uniq: 5 parallel processes

  Command substitution comparison:
    `backtick` vs $(modern): same behavior, but $() nests better
    $((...)) arithmetic: built-in, no subshell needed
    $((a + b)): O(1), no external process (vs python3 -c "print(a+b)")
```

### M2: Text Processing Complexity Math
```
grep complexity:
  fixed string (-F): O(n×m) with Boyer-Moore-Horspool algorithm
  regex (default): O(n×p) where p = pattern complexity
  n = file size in bytes, p = regex complexity

  awk vs sed vs grep for different tasks:
    grep: O(n) line scan — only for pattern matching/extraction
    sed: O(n) streaming — substitution, deletion, insertion
    awk: O(n) + O(F×L) where F=fields, L=lines — field processing, math

  sort complexity:
    sort: O(n log n) with merge sort
    sort -u: O(n log n) + O(n) dedup
    sort | uniq: same as sort -u (two passes vs one but equivalent)
    uniq without sort: O(1) per line, only removes adjacent duplicates

  wc performance:
    wc -l: O(n) byte scan (counts newlines)
    wc -w: O(n) with whitespace detection
    for large files (>1GB): wc is faster than awk '{count++}END{print count}'

  Text substitution:
    sed 's/a/b/': O(n) per substitution
    sed 's/a/b/g': O(n) per line (global substitution on each line)
    for millions of lines: prefer awk over multiple sed pipes (fewer processes)
```

### M3: File Operations Performance
```
File size and I/O:
  cat file: O(n) bytes read + stdout write
  cp: O(n) bytes (sequential read/write)
  mv (same filesystem): O(1) inode rename (no data copy!)
  mv (cross-filesystem): O(n) data copy + delete (same as cp + rm)

  find performance:
    find /path -name "*.txt": O(D×F) where D=dirs, F=files
    find with -maxdepth: reduces D dramatically
    locate (mlocate): O(log N) database lookup (pre-indexed, stale by up to 24h)

  Directory listing:
    ls: O(n) directory entries + sort
    ls -l: O(n) + O(n) stat() calls (one per file)
    ls -la --color: slower due to color computation
    for scripts: use for f in /path/*; do (no ls parsing needed)

  stat() call cost:
    local filesystem: ~0.01ms per call
    NFS/network filesystem: ~1-100ms per call (network latency)
    avoid stat() in tight loops on network filesystems

  File descriptor limits:
    ulimit -n: default 1024 (soft), 65536 (hard) on most Linux
    increase: ulimit -n 65536 (for many open files)
    each open file: ~1KB kernel memory overhead
```

### M4: Process Management Math
```
Process priority (nice values):
  nice value: -20 (highest priority) to +19 (lowest priority)
  default: 0 (normal priority)
  CFS scheduler weight: weight = 1024 / (nice_to_weight[nice+20])
  nice +10 vs nice 0: approximately 10× less CPU time allocation

  Background job control:
    cmd &: runs in background, same session
    nohup cmd &: immune to SIGHUP (session close)
    disown %1: detach from shell (no HUP on exit)
    setsid cmd: new session leader (fully detached)

  Signal handling:
    kill -0 PID: check if process exists (no actual signal, O(1))
    kill -SIGTERM PID: graceful termination request
    kill -SIGKILL PID: force terminate (cannot be caught)
    trap 'cleanup' EXIT: run cleanup function on script exit

  Wait and polling:
    wait $PID: blocks until process completes (efficient)
    while kill -0 $PID 2>/dev/null; do sleep 1; done: polling (1s granularity)
    inotifywait: event-driven file monitoring (0ms latency vs polling)

  Process substitution:
    diff <(cmd1) <(cmd2): runs both commands concurrently
    tee >(cmd1) >(cmd2): fan-out to multiple commands concurrently
```

### M5: Cron & Script Timing Math
```
Cron expression parsing:
  * * * * *: minute(0-59) hour(0-23) day(1-31) month(1-12) weekday(0-6)
  resolution: 1 minute minimum (use sleep for sub-minute)
  combined: overlap prevention with flock

  Script execution budget:
    cron job: must complete before next scheduled run
    if job takes 55min, runs every hour: 5min margin (risky)
    use flock for single-instance guarantee:
    flock -n /tmp/job.lock -c "script.sh" || echo "Already running"

  Sleep precision:
    sleep 1: ~1 second (OS timer resolution ~10ms)
    sleep 0.1: 100ms (bash supports fractional seconds)
    for precise timing: use date +%s%N for nanosecond timestamps

  Log rotation sizing:
    log_retention = log_rate_per_day × retention_days
    daily log rate: 1MB-1GB depending on verbosity
    rotate at: 100MB or daily (whichever comes first)
    compress older logs: gzip saves ~70-90% space

  Disk usage estimation:
    du -sh /path: O(D×F) directory traversal
    df -h: O(1) filesystem stats (from kernel)
    inotifywait for real-time: 0ms latency, vs 1s polling with watch
```

### M6: India CERT-In Linux Hardening and Compliance Score Mathematics

India's CERT-In (Indian Computer Emergency Response Team) issued mandatory Linux hardening guidelines in 2022 (CERT-In Directions under IT Act Section 70B) with specific requirements for government and critical infrastructure Linux systems. STQC (Standardisation Testing and Quality Certification) conducts formal security audits against CIS Benchmark criteria.

**CIS Benchmark Compliance Score Formula:**
The Center for Internet Security (CIS) Linux Benchmark Level 1/Level 2 compliance is scored by STQC as:
```
compliance_score = (passed_controls / total_controls) × 100

CIS Level 1 (essential security): 230 controls
CIS Level 2 (high-security): 280 controls

weighted_score = sum(control_i_weight × pass_i) / sum(all_weights)
  Critical controls weight: 3.0 (authentication, privilege escalation)
  High controls weight:     2.0 (network services, logging)
  Medium controls weight:   1.0 (file permissions, time sync)

STQC pass threshold: weighted_score ≥ 85% for government systems
NIC (National Informatics Centre) threshold: weighted_score ≥ 90%
```

**CERT-In 2022 Mandatory Requirements (shell scripting impact):**
CERT-In Directions (April 2022) require:
```
log_retention_requirement = 180 days minimum (CERT-In Section 2(ii))
log_collection_overhead = log_rate_bytes/sec × 180 × 86400 / compression_ratio
  example: 10MB/day raw logs × 180 days / 10× gzip = 180MB storage minimum
  with 3× replication (HA): 540MB per server

NTP synchronisation: all systems must sync to NIC NTP servers:
  time.nis.gov.in (primary) or time.stdtime.gov.in (secondary)
  drift_tolerance = ±5ms (CERT-In requirement vs ±128ms NTP default)
  shell check: chronyc tracking | grep "System time" | awk '{print $4}'
```

**India NIC Shell Scripting Standards:**
NIC mandates POSIX-compliant scripts for interoperability across RHEL/Ubuntu/SLES used in government clouds:
```
portability_score = (posix_compliant_constructs / total_constructs) × 100
bash_specific_penalty: -5 points per bash-ism (arrays, [[ ]], $RANDOM)
target: portability_score ≥ 95% for NIC-deployed scripts

script_complexity_limit = cyclomatic_complexity ≤ 10 per function (NIC coding standard)
functions_per_script ≤ 20 (maintainability requirement)
```

**STQC EMI/Security Audit Math:**
```
vulnerability_density = critical_CVEs_found / KLOC (thousands of lines of code)
STQC acceptable: vulnerability_density ≤ 0.5 per KLOC for production scripts
remediation_SLA: Critical CVE → 6 hours, High → 24 hours, Medium → 72 hours (CERT-In 2022)
```

---

## Core Rules (NON-NEGOTIABLE)

### Rule 1: Shebang and Shell Selection
```bash
#!/usr/bin/env bash
# For maximum portability use env lookup
# Use #!/bin/bash only when bash-specific features are required
# Use #!/bin/sh for POSIX-only scripts (maximum compatibility)
```

### Rule 2: Strict Mode (MANDATORY for all scripts)
```bash
set -euo pipefail
# -e: Exit on error
# -u: Treat unset variables as errors
# -o pipefail: Pipe fails if any command in pipeline fails

# Optional: Debug mode
# set -x  # Print each command before execution (debug only)
```

### Rule 3: Quoting (ALWAYS quote variables)
```bash
# CORRECT
echo "$variable"
cp "$source" "$destination"
if [[ -f "$file" ]]; then

# WRONG - never do this
echo $variable
cp $source $destination
if [ -f $file ]; then
```

### Rule 4: Use [[ ]] Over [ ]
```bash
# CORRECT (bash)
[[ -f "$file" ]]
[[ "$str" == "value" ]]
[[ "$num" -gt 5 ]]
[[ "$str" =~ ^[0-9]+$ ]]  # regex support

# AVOID (POSIX [ ] has quirks)
[ -f "$file" ]
```

---

## Script Template (Production-Grade)

```bash
#!/usr/bin/env bash
#
# Script: script-name.sh
# Version: 1.0.0
# Description: Brief description of what this script does
# Usage: ./script-name.sh [options] <arguments>
# Author: Auto-generated
# Date: YYYY-MM-DD
#

set -euo pipefail

# ─── Constants ───────────────────────────────────────────────
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"

# ─── Default Configuration ──────────────────────────────────
VERBOSE=false
DRY_RUN=false

# ─── Functions ───────────────────────────────────────────────

log() {
    local level="$1"
    shift
    local msg="$*"
    local ts
    ts="$(date '+%Y-%m-%d %H:%M:%S')"
    echo "[$ts] [$level] $msg" | tee -a "$LOG_FILE" 2>/dev/null || true
}

log_info()  { log "INFO"  "$@"; }
log_warn()  { log "WARN"  "$@"; }
log_error() { log "ERROR" "$@"; }

die() {
    log_error "$@"
    exit 1
}

usage() {
    cat <<EOF
Usage: $SCRIPT_NAME [OPTIONS] <args>

Options:
  -h, --help       Show this help message
  -v, --verbose    Enable verbose output
  -n, --dry-run    Show what would be done without executing

Examples:
  $SCRIPT_NAME -v /path/to/dir
  $SCRIPT_NAME --dry-run file.txt
EOF
    exit 0
}

cleanup() {
    # Called on EXIT - clean up temp files, processes, etc.
    local exit_code=$?
    if [[ -n "${TEMP_DIR:-}" && -d "$TEMP_DIR" ]]; then
        rm -rf "$TEMP_DIR"
    fi
    exit "$exit_code"
}

# ─── Argument Parsing ───────────────────────────────────────

parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -h|--help)    usage ;;
            -v|--verbose) VERBOSE=true; shift ;;
            -n|--dry-run) DRY_RUN=true; shift ;;
            --)           shift; break ;;
            -*)           die "Unknown option: $1" ;;
            *)            break ;;
        esac
    done

    # Remaining positional args
    ARGS=("$@")
}

# ─── Main Logic ─────────────────────────────────────────────

main() {
    parse_args "$@"
    trap cleanup EXIT

    log_info "Starting $SCRIPT_NAME"

    # ... script logic here ...

    log_info "Completed successfully"
}

# ─── Entry Point ─────────────────────────────────────────────
main "$@"
```

---

## Essential Patterns

### Pattern 1: File Processing
```bash
# Process file line by line (safe - handles spaces, special chars)
while IFS= read -r line; do
    echo "Processing: $line"
done < "$input_file"

# Process with line numbers
local line_num=0
while IFS= read -r line; do
    ((line_num++))
    echo "$line_num: $line"
done < "$input_file"

# Find and process files (safe for filenames with spaces)
find "$dir" -type f -name "*.log" -print0 | while IFS= read -r -d '' file; do
    echo "Found: $file"
done

# CSV parsing
while IFS=',' read -r col1 col2 col3; do
    echo "Column 1: $col1, Column 2: $col2"
done < "$csv_file"
```

### Pattern 2: Text Processing (awk/sed/grep)
```bash
# awk - columnar data extraction
awk '{print $1, $3}' file.txt                    # Print columns 1 and 3
awk -F',' '{sum+=$2} END{print sum}' data.csv     # Sum column 2
awk '/pattern/{print NR": "$0}' file.txt          # Lines matching pattern

# sed - stream editing
sed -i 's/old/new/g' file.txt                     # In-place replacement
sed -n '10,20p' file.txt                          # Print lines 10-20
sed '/^#/d' config.file                           # Remove comment lines
sed -i.bak 's/foo/bar/g' file.txt                 # Replace with backup

# grep - pattern searching
grep -r "pattern" /path/to/dir                    # Recursive search
grep -n "ERROR" log.txt                           # With line numbers
grep -c "pattern" file.txt                        # Count matches
grep -l "TODO" *.py                               # Files containing match
grep -E "^[0-9]{3}-[0-9]{4}$" file.txt            # Extended regex
grep -v "^#" config.file | grep -v "^$"           # Non-comment, non-empty
```

### Pattern 3: Process Management
```bash
# Background process with PID tracking
long_running_command &
local pid=$!
echo "$pid" > /var/run/myapp.pid

# Wait for process
wait "$pid" || die "Process $pid failed"

# Kill process by PID file
kill_by_pidfile() {
    local pidfile="$1"
    if [[ -f "$pidfile" ]]; then
        local pid
        pid="$(cat "$pidfile")"
        if kill -0 "$pid" 2>/dev/null; then
            kill "$pid"
            wait "$pid" 2>/dev/null || true
        fi
        rm -f "$pidfile"
    fi
}

# Timeout execution
timeout 30 some_command || die "Command timed out after 30s"

# Check if command exists
command -v docker &>/dev/null || die "docker is not installed"
```

### Pattern 4: Temporary Files and Directories
```bash
# Safe temp file creation
TEMP_FILE="$(mktemp)"
TEMP_DIR="$(mktemp -d)"

# Ensure cleanup on exit
trap 'rm -rf "$TEMP_FILE" "$TEMP_DIR"' EXIT

# Named pipe (FIFO) for IPC
FIFO="$(mktemp -u)"
mkfifo "$FIFO"
trap 'rm -f "$FIFO"' EXIT
```

### Pattern 5: Locking (Prevent Concurrent Execution)
```bash
LOCK_FILE="/var/lock/${SCRIPT_NAME}.lock"

acquire_lock() {
    if ! mkdir "$LOCK_FILE" 2>/dev/null; then
        die "Another instance is running (lock: $LOCK_FILE)"
    fi
    trap 'rm -rf "$LOCK_FILE"' EXIT
}

# Using flock (preferred on Linux)
exec 200>"$LOCK_FILE"
flock -n 200 || die "Another instance is running"
```

### Pattern 6: Networking
```bash
# Check port availability
check_port() {
    local host="$1" port="$2"
    if nc -z "$host" "$port" 2>/dev/null; then
        echo "Port $port is open on $host"
        return 0
    fi
    return 1
}

# Wait for service to be ready
wait_for_service() {
    local host="$1" port="$2" timeout="${3:-30}"
    local elapsed=0
    while ! nc -z "$host" "$port" 2>/dev/null; do
        ((elapsed++))
        if [[ $elapsed -ge $timeout ]]; then
            die "Service $host:$port not ready after ${timeout}s"
        fi
        sleep 1
    done
    log_info "Service $host:$port is ready"
}

# Download with retry
download_with_retry() {
    local url="$1" dest="$2" max_retries="${3:-3}"
    local attempt=0
    while ((attempt < max_retries)); do
        ((attempt++))
        if curl -fsSL -o "$dest" "$url"; then
            return 0
        fi
        log_warn "Download attempt $attempt/$max_retries failed"
        sleep $((attempt * 2))
    done
    return 1
}
```

### Pattern 7: Cron Job Management
```bash
# Add a cron job (idempotent)
add_cron_job() {
    local schedule="$1" command="$2"
    local marker="# managed:${SCRIPT_NAME}"
    (crontab -l 2>/dev/null | grep -v "$marker"; echo "$schedule $command $marker") | crontab -
}

# Remove a cron job
remove_cron_job() {
    local marker="# managed:${SCRIPT_NAME}"
    crontab -l 2>/dev/null | grep -v "$marker" | crontab -
}

# Common cron schedules
# ┌──── minute (0-59)
# │ ┌──── hour (0-23)
# │ │ ┌──── day of month (1-31)
# │ │ │ ┌──── month (1-12)
# │ │ │ │ ┌──── day of week (0-7, 0=7=Sunday)
# * * * * *
# 0 * * * *     Every hour
# */5 * * * *   Every 5 minutes
# 0 2 * * *     Daily at 2 AM
# 0 0 * * 0     Weekly on Sunday
# 0 0 1 * *     Monthly on the 1st
```

### Pattern 8: Service/Daemon Script
```bash
# systemd-compatible service script pattern
start_service() {
    log_info "Starting service..."
    nohup "$SCRIPT_DIR/app" >> "$LOG_FILE" 2>&1 &
    echo $! > "$PID_FILE"
    log_info "Service started (PID: $(cat "$PID_FILE"))"
}

stop_service() {
    if [[ -f "$PID_FILE" ]]; then
        local pid
        pid="$(cat "$PID_FILE")"
        if kill -0 "$pid" 2>/dev/null; then
            log_info "Stopping service (PID: $pid)..."
            kill "$pid"
            local count=0
            while kill -0 "$pid" 2>/dev/null && ((count < 10)); do
                sleep 1
                ((count++))
            done
            if kill -0 "$pid" 2>/dev/null; then
                log_warn "Force killing PID $pid"
                kill -9 "$pid"
            fi
        fi
        rm -f "$PID_FILE"
    fi
}

status_service() {
    if [[ -f "$PID_FILE" ]] && kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
        echo "Running (PID: $(cat "$PID_FILE"))"
    else
        echo "Stopped"
    fi
}

case "${1:-}" in
    start)   start_service ;;
    stop)    stop_service ;;
    restart) stop_service; start_service ;;
    status)  status_service ;;
    *)       echo "Usage: $0 {start|stop|restart|status}" ;;
esac
```

---

## Arrays and Data Structures

```bash
# Indexed array
declare -a files=("file1.txt" "file2.txt" "file3.txt")
files+=("file4.txt")                    # Append
echo "${files[0]}"                      # Access by index
echo "${#files[@]}"                     # Length
echo "${files[@]}"                      # All elements

# Associative array (bash 4+)
declare -A config
config[host]="localhost"
config[port]="8080"
config[debug]="true"

for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done

# Array iteration
for item in "${files[@]}"; do
    echo "Processing: $item"
done
```

---

## String Operations

```bash
# Substring
str="Hello, World!"
echo "${str:0:5}"           # Hello
echo "${str:7}"             # World!

# Replacement
echo "${str/World/Bash}"    # Hello, Bash!
echo "${str//l/L}"          # HeLLo, WorLd! (all occurrences)

# Length
echo "${#str}"              # 13

# Default values
echo "${VAR:-default}"      # Use default if VAR is unset/empty
echo "${VAR:=default}"      # Set AND use default if unset/empty

# Trim
echo "${str#Hello, }"       # World! (remove prefix)
echo "${str%!}"             # Hello, World (remove suffix)

# Pattern-based path manipulation
filepath="/home/user/docs/report.tar.gz"
echo "${filepath##*/}"      # report.tar.gz (basename)
echo "${filepath%/*}"       # /home/user/docs (dirname)
echo "${filepath%%.*}"      # /home/user/docs/report (remove all extensions)
echo "${filepath##*.}"      # gz (last extension)
```

---

## Error Handling Patterns

```bash
# Trap for cleanup
trap cleanup EXIT
trap 'die "Interrupted"' INT TERM

# Retry with backoff
retry() {
    local max_attempts="$1"
    shift
    local attempt=1
    while true; do
        if "$@"; then
            return 0
        fi
        if ((attempt >= max_attempts)); then
            log_error "Failed after $max_attempts attempts: $*"
            return 1
        fi
        local delay=$((attempt * 2))
        log_warn "Attempt $attempt/$max_attempts failed, retrying in ${delay}s..."
        sleep "$delay"
        ((attempt++))
    done
}

# Usage: retry 3 curl -fsSL http://example.com

# Assert function
assert() {
    local condition="$1" message="${2:-Assertion failed}"
    if ! eval "$condition"; then
        die "$message"
    fi
}

# Usage: assert '[[ -f "$config_file" ]]' "Config file not found: $config_file"
```

---

## Security Best Practices

1. **Never use eval with user input** - command injection risk
2. **Quote all variables** - prevents word splitting and globbing
3. **Use `--` to end option parsing** - prevents flag injection: `rm -- "$file"`
4. **Validate all inputs** before using them
5. **Restrict PATH** at script start: `export PATH="/usr/local/bin:/usr/bin:/bin"`
6. **Use `mktemp`** for temp files - never hardcode temp paths
7. **Set umask** for file creation: `umask 077`
8. **Avoid storing secrets** in scripts - use env vars or secret managers
9. **Use `readonly`** for constants to prevent accidental modification
10. **Check return codes** - never ignore command failures

```bash
# Input validation
validate_input() {
    local input="$1"
    # Reject dangerous characters
    if [[ "$input" =~ [\'\"\\$\`\;] ]]; then
        die "Invalid characters in input: $input"
    fi
}
```

---

## Performance Tips

```bash
# Prefer built-in over external commands
# SLOW: result=$(echo "$str" | tr '[:upper:]' '[:lower:]')
# FAST:
result="${str,,}"

# SLOW: count=$(echo "$str" | wc -c)
# FAST:
count="${#str}"

# Use printf over echo for portability
printf '%s\n' "$message"

# Parallel execution
parallel_exec() {
    local max_jobs="${1:-4}"
    shift
    local pids=()
    for cmd in "$@"; do
        eval "$cmd" &
        pids+=($!)
        if ((${#pids[@]} >= max_jobs)); then
            wait "${pids[0]}"
            pids=("${pids[@]:1}")
        fi
    done
    wait
}

# Process substitution (avoid temp files)
diff <(sort file1.txt) <(sort file2.txt)
```

---

## Anti-Patterns to Avoid

- **Using `$(command)` inside a tight loop instead of a shell function**: M1's worked example shows 1000 iterations at 5ms subshell overhead each costs 5 seconds versus under 1ms with an in-process function call — spawning a subshell per iteration for logic that doesn't need an external process wastes orders of magnitude of time.
- **Shelling out for arithmetic instead of using `$((...))`**: built-in arithmetic expansion is O(1) with no subshell (M1) — reaching for `python3 -c "print(a+b)"` or `expr` for simple integer math pays a full fork+exec cost (2-10ms) for work the shell can do natively for free.
- **Chaining multiple `sed` pipes for large-line-count text processing**: M2 recommends a single `awk` pass over multiple sed pipe stages for millions of lines — each additional pipe stage spawns another process; consolidating into one awk program avoids the per-stage process-spawn overhead that compounds at scale.
- **Calling `stat()`-backed commands (`ls -l`, per-file `stat`) in a tight loop against a network filesystem**: M3 shows local stat() costs ~0.01ms per call but NFS/network filesystems cost 1-100ms per call — a loop that seemed fast in local testing can become 100-10,000× slower once it runs against network-mounted storage.
- **Parsing `ls` output to iterate over files**: this skill's own recommendation (M3) is `for f in /path/*; do` — parsing `ls` breaks on filenames with spaces/newlines/special characters and adds unnecessary process overhead that globbing avoids entirely.
- **Running `find` without `-maxdepth` when the search depth is actually bounded**: `find`'s cost is O(D×F) (M3) — omitting `-maxdepth` when the target files are known to live within a specific depth searches far more directories than necessary, especially costly on deep trees.
- **Polling process completion with a sleep loop instead of `wait` or event-driven monitoring**: `wait $PID` blocks efficiently until completion, while a `while kill -0 $PID; do sleep 1; done` loop adds up to 1 second of detection latency per check (M4) — for filesystem events specifically, `inotifywait` provides 0ms-latency event-driven detection that polling can't match.
- **Scheduling a cron job without `flock` when its runtime approaches the schedule interval**: M5's example (55-minute job on an hourly schedule, 5-minute margin) is explicitly flagged as risky — without `flock -n /tmp/job.lock`, a slow run can overlap with the next scheduled invocation, corrupting shared state or duplicating work.
- **Relying on default NTP drift tolerance for CERT-In-governed government systems**: CERT-In requires ±5ms drift tolerance via NIC NTP servers (M6), far tighter than the ±128ms default NTP tolerance — a system left on default NTP configuration doesn't meet this mandate even if it appears "synced" by ordinary standards.
- **Writing bash-specific constructs (arrays, `[[ ]]`, `$RANDOM`) for NIC-deployed scripts requiring POSIX portability**: M6's scoring penalizes each bash-ism at -5 points against a ≥95% portability_score target across RHEL/Ubuntu/SLES government cloud deployments — bash-only syntax that works fine in one environment can silently fail portability requirements in this specific deployment context.

## Response Rules

1. Always start every script with `#!/usr/bin/env bash` and `set -euo pipefail`
2. Always quote variables: `"$var"` — unquoted variables are word-split and glob-expanded
3. Always use `[[ ]]` for tests in bash, never `[ ]` — `[[ ]]` prevents word-splitting and has regex support
4. Always use `mktemp` for temp files and trap for cleanup — never hardcode `/tmp/myfile`
5. Always add `usage()` function and `--help` / `-h` handling
6. When processing multiple files, use `while read -r` or `find ... -print0 | xargs -0`
7. Always validate required arguments before executing main logic
8. Always use meaningful exit codes: 0=success, 1=general error, 2=misuse
9. Always run `shellcheck script.sh` before submitting — zero warnings required

---

## What Not to Do

- Do not use `[ ]` (single brackets) — use `[[ ]]` in bash scripts
- Do not leave variables unquoted — `rm $file` can delete unintended files if `$file` has spaces
- Do not use `cat file | grep pattern` — use `grep pattern file` (useless use of cat)
- Do not ignore exit codes — always check `$?` or use `set -e`
- Do not use `ls` output in scripts — use glob patterns or `find` instead
- Do not hardcode paths like `/home/user/` — use `$HOME` and `$(dirname "$0")`
- Do not use `echo` for binary data — use `printf`
- Do not parse `ls` output for filenames — filenames can contain spaces and newlines
- Do not skip `shellcheck` — it catches the majority of common shell bugs

---

## Output Expectations

- Production-ready script with shebang, strict mode, usage function, argument validation, logging
- Functions: `usage()`, `log()`, `main()` with `main "$@"` call at bottom
- Trap-based cleanup for temp files and lock files
- Meaningful exit codes with error messages to stderr (`>&2`)
- ShellCheck-passing script with zero warnings
- Inline comments explaining non-obvious logic
- Example usage in a comment block at the top

---

## Skill Scope

### In Scope
- Bash/Zsh scripting for Linux/macOS system automation
- File processing: reading, writing, parsing, transforming text files
- Process management: background jobs, signal handling, PID files
- Network operations: curl, wget, SSH remote execution
- Cron job scripts and scheduled automation
- System administration scripts: user management, disk monitoring, log rotation
- Hook scripts for tools (git hooks, Claude Code hooks)
- CI/CD helper scripts in bash

### Out of Scope
- PowerShell scripting (use powershell-scripting skill)
- Python system scripting (use python-system-scripting skill)
- Complex data processing requiring Python/awk (defer to python-system-scripting)
- Kernel module or eBPF programs

---

## Checklist Before Committing Any Shell Script

1. [ ] Shebang line present (`#!/usr/bin/env bash`)
2. [ ] `set -euo pipefail` at the top
3. [ ] All variables quoted (`"$var"`)
4. [ ] `[[ ]]` used instead of `[ ]` (for bash scripts)
5. [ ] Temp files created with `mktemp` and cleaned up via trap
6. [ ] Input validation for user-provided arguments
7. [ ] Meaningful exit codes (0=success, 1=error)
8. [ ] Logging with timestamps
9. [ ] Usage/help function present
10. [ ] ShellCheck passes (`shellcheck script.sh`)

---

## Version

1.0.1 — Added Anti-Patterns to Avoid: subshell-in-loop overhead, shelled-out arithmetic, multi-sed-pipe chains, network-filesystem stat() loops, ls-output parsing, unbounded find depth, sleep-loop polling, flock-less overlapping cron jobs, default NTP drift on CERT-In systems, bash-isms in NIC-portable scripts

1.0.0
