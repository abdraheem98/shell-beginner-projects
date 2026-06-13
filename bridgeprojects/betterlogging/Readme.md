# Bridge 1 — Better Logging

> **Session:** Bridge — Intermediate to Advanced  
> **Duration:** ~20 minutes  
> **Prerequisite:** Intermediate Topic 8 (Mini Project) complete  
> **File to upgrade:** `server-monitor/lib/common.sh` + `monitor.sh`

---

## Table of Contents

- [What We Are Improving](#what-we-are-improving)
- [The Problem With Current Logging](#the-problem-with-current-logging)
- [Upgrade 1 — Add Log Levels to Output](#upgrade-1--add-log-levels-to-output)
- [Upgrade 2 — Write to a Proper Log File](#upgrade-2--write-to-a-proper-log-file)
- [Upgrade 3 — Add Log Size Check](#upgrade-3--add-log-size-check)
- [Upgrade 4 — Separate stdout and stderr](#upgrade-4--separate-stdout-and-stderr)
- [Final Upgraded common.sh](#final-upgraded-commonsh)
- [What Changed in monitor.sh](#what-changed-in-monitorsh)
- [Before vs After](#before-vs-after)
- [Key Takeaways](#key-takeaways)

---

## What We Are Improving

Open `lib/common.sh` from the mini project.
The current `log()` function looks like this:

```bash
log() {
  local level=$1; shift
  local message=$*
  local entry
  entry="[$(date '+%Y-%m-%d %H:%M:%S')] [$(printf '%-5s' "$level")] $message"
  echo "$entry"
  echo "$entry" >> "$LOG_FILE"
}
```

This works — but there are three real problems with it.

---

## The Problem With Current Logging

### Problem 1 — No level filtering

Every message prints — even DEBUG messages in production.
When running on a server with 1000 log lines per day,
debug noise makes it impossible to find real errors.

```bash
# Current output — everything prints
[2024-06-12 09:15:32] [DEBUG] Config file loaded
[2024-06-12 09:15:32] [DEBUG] Services array has 3 items
[2024-06-12 09:15:32] [INFO ] nginx — running ✓
[2024-06-12 09:15:32] [DEBUG] Disk check starting for /
[2024-06-12 09:15:32] [INFO ] / — 42% used ✓
```

**What we want:** only show INFO and above normally,
show DEBUG only when explicitly asked.

---

### Problem 2 — No log file size check

The script appends to `monitor.log` every 5 minutes.
After 30 days that file can be 500MB+ with no warning.

```bash
# After 30 days of cron
ls -lh reports/monitor.log
# -rw-r--r-- 1 ubuntu ubuntu 487M Jun 12 09:15 monitor.log
# disk fills up → cron job fails → nobody knows
```

---

### Problem 3 — Errors and info go to the same place

When you pipe output or redirect it,
errors and normal messages get mixed together.
This makes automated processing unreliable.

```bash
# Current — errors and info both go to stdout
./monitor.sh > output.txt    # errors and info both in output.txt

# What we want — separate them
./monitor.sh > info.txt 2> errors.txt
```

---

## Upgrade 1 — Add Log Levels to Output

Replace the current `log()` function with a level-aware version.

**Old version:**
```bash
log() {
  local level=$1; shift
  local message=$*
  local entry
  entry="[$(date '+%Y-%m-%d %H:%M:%S')] [$(printf '%-5s' "$level")] $message"
  echo "$entry"
  echo "$entry" >> "$LOG_FILE"
}
```

**New version — with level filtering:**
```bash
# At the top of common.sh — configurable from outside
LOG_LEVEL="${LOG_LEVEL:-INFO}"

# Weight system — DEBUG=0 INFO=1 WARN=2 ERROR=3
get_level_weight() {
  case $1 in
    DEBUG) echo 0 ;;
    INFO)  echo 1 ;;
    WARN)  echo 2 ;;
    ERROR) echo 3 ;;
    *)     echo 1 ;;
  esac
}

log() {
  local level=$1; shift
  local message=$*

  # Check if this level should be shown
  local msg_weight current_weight
  msg_weight=$(get_level_weight "$level")
  current_weight=$(get_level_weight "$LOG_LEVEL")

  # Skip if message level is lower than configured level
  [ "$msg_weight" -ge "$current_weight" ] || return 0

  local entry
  entry="[$(date '+%Y-%m-%d %H:%M:%S')] [$(printf '%-5s' "$level")] $message"

  echo "$entry"
  echo "$entry" >> "$LOG_FILE"
}
```

**Test it — run with different log levels:**

```bash
# Normal run — only INFO and above
./monitor.sh
```

```
[2024-06-12 09:15:32] [INFO ] nginx — running ✓
[2024-06-12 09:15:32] [INFO ] / — 42% used ✓
[2024-06-12 09:15:32] [ERROR] cron — STOPPED ✗
```

```bash
# Debug run — see everything
LOG_LEVEL=DEBUG ./monitor.sh
```

```
[2024-06-12 09:15:32] [DEBUG] Config file loaded
[2024-06-12 09:15:32] [DEBUG] Services array has 3 items
[2024-06-12 09:15:32] [INFO ] nginx — running ✓
[2024-06-12 09:15:32] [DEBUG] Disk check starting for /
[2024-06-12 09:15:32] [INFO ] / — 42% used ✓
[2024-06-12 09:15:32] [ERROR] cron — STOPPED ✗
```

```bash
# Errors only — useful in CI/CD
LOG_LEVEL=ERROR ./monitor.sh
```

```
[2024-06-12 09:15:32] [ERROR] cron — STOPPED ✗
```

> ✅ Same script — different verbosity controlled from outside.
> No code changes needed.

---

## Upgrade 2 — Write to a Proper Log File

The current script writes everything to one file forever.
Add automatic size checking before every write:

```bash
# Add this to common.sh — runs before each log write
MAX_LOG_SIZE_MB="${MAX_LOG_SIZE_MB:-50}"

check_log_size() {
  [ -f "$LOG_FILE" ] || return 0

  local size_mb
  size_mb=$(du -m "$LOG_FILE" 2>/dev/null | cut -f1)

  if [ "${size_mb:-0}" -ge "$MAX_LOG_SIZE_MB" ]; then
    # Rotate — rename current log with timestamp
    local archive="${LOG_FILE}.$(date '+%Y%m%d-%H%M%S')"
    mv "$LOG_FILE" "$archive"
    gzip "$archive"

    # Start fresh
    touch "$LOG_FILE"

    # Write rotation notice to new log
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO ] Log rotated — archived: $(basename "$archive").gz" \
      >> "$LOG_FILE"
  fi
}
```

**Update `log()` to call it before writing:**

```bash
log() {
  local level=$1; shift
  local message=$*

  local msg_weight current_weight
  msg_weight=$(get_level_weight "$level")
  current_weight=$(get_level_weight "$LOG_LEVEL")
  [ "$msg_weight" -ge "$current_weight" ] || return 0

  # Check log size before writing
  check_log_size

  local entry
  entry="[$(date '+%Y-%m-%d %H:%M:%S')] [$(printf '%-5s' "$level")] $message"

  echo "$entry"
  echo "$entry" >> "$LOG_FILE"
}
```

**Test it — force a rotation:**

```bash
# Create a large fake log to trigger rotation
dd if=/dev/zero bs=1M count=51 > reports/monitor.log 2>/dev/null

# Run monitor — it will rotate automatically
./monitor.sh

# Check what happened
ls -lh reports/
```

```
-rw-r--r-- monitor.log                  (new, small)
-rw-r--r-- monitor.log.20240612-091532.gz  (archived)
```

> ✅ Log never grows out of control.
> No manual cleanup needed.

---

## Upgrade 3 — Add Log Size Check

Add a `warn_log_size()` function that warns when the log
is getting large but not yet at the rotation threshold:

```bash
# Add to common.sh
WARN_LOG_SIZE_MB="${WARN_LOG_SIZE_MB:-40}"

warn_log_size() {
  [ -f "$LOG_FILE" ] || return 0

  local size_mb
  size_mb=$(du -m "$LOG_FILE" 2>/dev/null | cut -f1)

  if [ "${size_mb:-0}" -ge "$WARN_LOG_SIZE_MB" ]; then
    log "WARN" "Log file is ${size_mb}MB — approaching rotation limit (${MAX_LOG_SIZE_MB}MB)"
  fi
}
```

**Call it at the start of monitor.sh:**

```bash
# In monitor.sh — after sourcing common.sh
warn_log_size
```

**Output when log is large:**
```
[2024-06-12 09:15:32] [WARN ] Log file is 43MB — approaching rotation limit (50MB)
```

> ✅ Early warning before the disk fills up.

---

## Upgrade 4 — Separate stdout and stderr

Right now errors and info both go to `stdout`.
Send ERROR level messages to `stderr` instead:

```bash
# Updated log() function — final version
log() {
  local level=$1; shift
  local message=$*

  local msg_weight current_weight
  msg_weight=$(get_level_weight "$level")
  current_weight=$(get_level_weight "$LOG_LEVEL")
  [ "$msg_weight" -ge "$current_weight" ] || return 0

  check_log_size

  local entry
  entry="[$(date '+%Y-%m-%d %H:%M:%S')] [$(printf '%-5s' "$level")] $message"

  # ERROR and WARN → stderr
  # INFO and DEBUG → stdout
  if [ "$level" = "ERROR" ] || [ "$level" = "WARN" ]; then
    echo "$entry" >&2
  else
    echo "$entry"
  fi

  # Always write everything to log file
  echo "$entry" >> "$LOG_FILE"
}
```

**Now you can separate streams:**

```bash
# Only capture errors
./monitor.sh 2> errors-only.txt

# Only capture normal output
./monitor.sh > info-only.txt

# Capture both separately
./monitor.sh > info.txt 2> errors.txt

# See both in terminal but save errors to file
./monitor.sh 2> errors.txt
```

> ✅ Works with standard Unix redirection patterns.
> CI/CD tools can now filter output cleanly.

---

## Final Upgraded `common.sh`

Replace the entire `lib/common.sh` with this:

```bash
# lib/common.sh — upgraded logging with level filtering,
# size rotation, and stderr separation

# ── Log configuration ──────────────────────────────────
LOG_LEVEL="${LOG_LEVEL:-INFO}"
LOG_FILE="${LOG_FILE:-/tmp/monitor.log}"
MAX_LOG_SIZE_MB="${MAX_LOG_SIZE_MB:-50}"
WARN_LOG_SIZE_MB="${WARN_LOG_SIZE_MB:-40}"

# ── Level weights ─────────────────────────────────────
get_level_weight() {
  case $1 in
    DEBUG) echo 0 ;;
    INFO)  echo 1 ;;
    WARN)  echo 2 ;;
    ERROR) echo 3 ;;
    *)     echo 1 ;;
  esac
}

# ── Log rotation ──────────────────────────────────────
check_log_size() {
  [ -f "$LOG_FILE" ] || return 0

  local size_mb
  size_mb=$(du -m "$LOG_FILE" 2>/dev/null | cut -f1)

  if [ "${size_mb:-0}" -ge "$MAX_LOG_SIZE_MB" ]; then
    local archive="${LOG_FILE}.$(date '+%Y%m%d-%H%M%S')"
    mv "$LOG_FILE" "$archive"
    gzip "$archive" &    # compress in background
    touch "$LOG_FILE"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO ] Log rotated: $(basename "$archive").gz" \
      >> "$LOG_FILE"
  fi
}

# ── Log size warning ──────────────────────────────────
warn_log_size() {
  [ -f "$LOG_FILE" ] || return 0
  local size_mb
  size_mb=$(du -m "$LOG_FILE" 2>/dev/null | cut -f1)
  if [ "${size_mb:-0}" -ge "$WARN_LOG_SIZE_MB" ]; then
    log "WARN" "Log is ${size_mb}MB — approaching limit (${MAX_LOG_SIZE_MB}MB)"
  fi
}

# ── Core log function ─────────────────────────────────
log() {
  local level=$1; shift
  local message=$*

  local msg_weight current_weight
  msg_weight=$(get_level_weight "$level")
  current_weight=$(get_level_weight "$LOG_LEVEL")
  [ "$msg_weight" -ge "$current_weight" ] || return 0

  check_log_size

  local entry
  entry="[$(date '+%Y-%m-%d %H:%M:%S')] [$(printf '%-5s' "$level")] $message"

  # Errors and warnings → stderr | Info and debug → stdout
  if [ "$level" = "ERROR" ] || [ "$level" = "WARN" ]; then
    echo "$entry" >&2
  else
    echo "$entry"
  fi

  echo "$entry" >> "$LOG_FILE"
}

# ── Helpers ───────────────────────────────────────────
require_binary() {
  local bin=$1
  command -v "$bin" > /dev/null 2>&1 || {
    log "ERROR" "Required binary not found: $bin"
    exit 1
  }
}

separator() {
  local line="────────────────────────────────────────"
  echo "$line"
  echo "$line" >> "$LOG_FILE"
}
```

---

## What Changed in `monitor.sh`

Add two lines to `monitor.sh` — right after sourcing `common.sh`:

```bash
# In monitor.sh — after source "$SCRIPT_DIR/lib/common.sh"

# Warn if log is getting large
warn_log_size

# Add debug log entries to help trace issues
log "DEBUG" "Config file : $CONFIG_FILE"
log "DEBUG" "Report dir  : $REPORT_DIR"
log "DEBUG" "Log level   : $LOG_LEVEL"
```

No other changes needed in `monitor.sh`.
The upgraded `log()` in `common.sh` handles everything automatically.

---

## Before vs After

### Before

```bash
# All messages look the same
# DEBUG clutters production output
# Log file grows forever
# Errors and info mixed on stdout

[2024-06-12 09:15:32] [DEBUG] Config file loaded
[2024-06-12 09:15:32] [INFO ] nginx — running ✓
[2024-06-12 09:15:32] [ERROR] cron — STOPPED ✗
```

### After

```bash
# Normal run — clean INFO output
./monitor.sh

[2024-06-12 09:15:32] [INFO ] nginx — running ✓
[2024-06-12 09:15:32] [INFO ] / — 42% used ✓
# ↑ stdout
# ERROR and WARN go to stderr (separate stream)

# Debug run — full detail
LOG_LEVEL=DEBUG ./monitor.sh

[2024-06-12 09:15:32] [DEBUG] Config file : /opt/server-monitor/config/monitor.conf
[2024-06-12 09:15:32] [DEBUG] Log level   : DEBUG
[2024-06-12 09:15:32] [INFO ] nginx — running ✓
[2024-06-12 09:15:32] [INFO ] / — 42% used ✓

# Errors only
LOG_LEVEL=ERROR ./monitor.sh 2>&1

[2024-06-12 09:15:32] [ERROR] cron — STOPPED ✗
```

---

## Key Takeaways

| What Changed | Why It Matters |
|---|---|
| `LOG_LEVEL` filtering | No debug noise in production — flip a switch to debug |
| `get_level_weight()` | Compare levels as numbers — clean and extensible |
| `check_log_size()` | Log never fills the disk — rotates automatically |
| `warn_log_size()` | Early warning before hitting the limit |
| ERROR → stderr | Works with standard Unix redirection |
| `LOG_LEVEL` from env | Control verbosity without changing any code |

---

## Connection to the Advanced Series

This upgrade is exactly **Concept 1 — Structured Logging** from the advanced series.

When you get there, you will recognize every piece:

```
Bridge 1 log()           →   Advanced Concept 1 log()
Level filtering          →   Same pattern, same weight system
check_log_size()         →   Same rotation logic
ERROR to stderr          →   Same separation
LOG_LEVEL env var        →   Same external control
```

> The difference between intermediate and advanced logging
> is not the concept — it is the completeness and robustness.
> You have already built the foundation.

---

## What's Next

**Bridge 2 — Config Validation**  
Add a validation gate that checks every value in
`monitor.conf` before any checks run — so the script
fails clearly at startup instead of crashing
halfway through with a confusing error.
This is the bridge to Advanced Concept 4 (Config Management).