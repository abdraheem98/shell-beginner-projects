# Intermediate Topic 8 — Real-World Mini Project

> **Session:** Shell Scripting for DevOps — Intermediate Concepts  
> **Duration:** ~35 minutes  
> **Prerequisite:** Topics 1, 1+, 2, 3, 4, 5, 6, 7 must be complete

---

## Table of Contents

- [What We Are Building](#what-we-are-building)
- [What Each Topic Contributes](#what-each-topic-contributes)
- [Project Structure](#project-structure)
- [Step 1 — Create the Project Folder](#step-1--create-the-project-folder)
- [Step 2 — Write the Shared Library](#step-2--write-the-shared-library)
- [Step 3 — Create the Config File](#step-3--create-the-config-file)
- [Step 4 — Build the Monitor Script](#step-4--build-the-monitor-script)
- [Step 5 — Run It and See the Output](#step-5--run-it-and-see-the-output)
- [Step 6 — Add the Report File](#step-6--add-the-report-file)
- [Step 7 — Schedule with Cron](#step-7--schedule-with-cron)
- [Full Final Script](#full-final-script)
- [Key Takeaways](#key-takeaways)

---

## What We Are Building

A **Server Monitor** — a tool that:

- Reads a list of services and disk paths from a config file
- Checks if each service is running
- Checks disk usage on each path
- Checks if the app is responding on a given port
- Writes a structured log for every check
- Saves a summary report to a file
- Can be run manually or on a cron schedule

**This is a tool you could actually use on a real server — today.**

---

## What Each Topic Contributes

| Topic | Used In This Project |
|---|---|
| 1 — Exit Codes | Every check returns 0 or 1 |
| 1+ — Safety Patterns | `set -euo pipefail` + `trap` at the top |
| 2 — String Manipulation | Build file paths, extract disk % |
| 3 — Arrays | List of services and disk paths |
| 4 — Reading Files | Read config file line by line |
| 5 — Command Substitution | Capture disk %, service status, timestamp |
| 6 — Debugging | `DEBUG=1` mode, `trap ERR` |
| 7 — Reusable Scripts | Arguments, usage message, `lib/common.sh` |

---

## Project Structure

```
server-monitor/
├── monitor.sh          ← main script
├── lib/
│   └── common.sh       ← shared logging and helper functions
├── config/
│   └── monitor.conf    ← services, paths, thresholds
└── reports/
    └── (generated)     ← daily report files saved here
```

---

## Step 1 — Create the Project Folder

Run these commands to set up the structure:

```bash
mkdir -p server-monitor/lib
mkdir -p server-monitor/config
mkdir -p server-monitor/reports
cd server-monitor
```

---

## Step 2 — Write the Shared Library

**File: `lib/common.sh`**

This is the shared foundation — used by every script in the project.
Every concept from Topic 1 and Topic 7 lives here.

```bash
# lib/common.sh
# Shared functions for the server-monitor project
# Source this file — do not run it directly

# ── Logging ───────────────────────────────────────────
LOG_LEVEL="${LOG_LEVEL:-INFO}"
LOG_FILE="${LOG_FILE:-/tmp/monitor.log}"

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

  local msg_weight current_weight
  msg_weight=$(get_level_weight "$level")
  current_weight=$(get_level_weight "$LOG_LEVEL")

  # Only log if level is high enough
  [ "$msg_weight" -ge "$current_weight" ] || return 0

  local entry
  entry="[$(date '+%Y-%m-%d %H:%M:%S')] [$(printf '%-5s' "$level")] $message"

  # Print to terminal
  echo "$entry"

  # Also write to log file
  echo "$entry" >> "$LOG_FILE"
}

# ── Binary check ──────────────────────────────────────
require_binary() {
  local bin=$1
  command -v "$bin" > /dev/null 2>&1 || {
    log "ERROR" "Required binary not found: $bin"
    exit 1
  }
}

# ── Separator line ────────────────────────────────────
separator() {
  local line="────────────────────────────────────────"
  echo "$line"
  echo "$line" >> "$LOG_FILE"
}
```

---

## Step 3 — Create the Config File

**File: `config/monitor.conf`**

This separates data from logic — a key principle from Topic 4 and Topic 7.
Anyone can edit this file to add services, paths, or thresholds
without touching the script.

```bash
# config/monitor.conf
# Server Monitor Configuration
# Lines starting with # are comments and are ignored

# ── Services to check ─────────────────────────────────
# Format: SERVICE:<service-name>
SERVICE:nginx
SERVICE:ssh
SERVICE:cron

# ── Disk paths to check ───────────────────────────────
# Format: DISK:<path>:<threshold-percent>
DISK:/,90
DISK:/tmp,80
DISK:/var/log,85

# ── App health check ──────────────────────────────────
# Format: PORT:<port-number>
PORT:3000

# ── Alert settings ────────────────────────────────────
# Format: ALERT_EMAIL:<email>
ALERT_EMAIL:ops@company.com
```

---

## Step 4 — Build the Monitor Script

Now we build `monitor.sh` — one section at a time.
This is where every topic comes together.

**File: `monitor.sh`**

### Section 1 — Header and safety setup (Topics 1+ and 6)

```bash
#!/bin/bash
# monitor.sh — Server monitoring tool
# Usage: ./monitor.sh [--report] [--debug]
# Cron:  */5 * * * * /opt/server-monitor/monitor.sh

set -euo pipefail

# ── Script directory (Topic 7) ────────────────────────
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# ── Source shared library (Topic 7) ───────────────────
source "$SCRIPT_DIR/lib/common.sh"
```

### Section 2 — Configuration (Topics 4 and 7)

```bash
# ── Config ────────────────────────────────────────────
CONFIG_FILE="$SCRIPT_DIR/config/monitor.conf"
REPORT_DIR="$SCRIPT_DIR/reports"
LOG_FILE="$REPORT_DIR/monitor.log"
REPORT_FILE="$REPORT_DIR/report-$(date '+%Y%m%d').txt"
DEBUG="${DEBUG:-0}"
GENERATE_REPORT=false

# Enable trace mode if DEBUG=1 (Topic 6)
[ "$DEBUG" = "1" ] && set -x

# ── Error handler (Topics 1+ and 6) ───────────────────
handle_error() {
  log "ERROR" "Failed at line $1 — command: $BASH_COMMAND"
}
trap 'handle_error $LINENO' ERR
```

### Section 3 — Usage message (Topic 7)

```bash
# ── Usage (Topic 7) ───────────────────────────────────
usage() {
  cat << EOF

  Usage: $(basename "$0") [options]

  Options:
    --report       Save a summary report to $REPORT_DIR
    --debug        Show debug output
    -h, --help     Show this message

  Examples:
    $(basename "$0")
    $(basename "$0") --report
    DEBUG=1 $(basename "$0")

EOF
  exit 0
}

# ── Parse arguments (Topic 7) ─────────────────────────
for arg in "$@"; do
  case $arg in
    --report)       GENERATE_REPORT=true ;;
    --debug)        DEBUG=1; set -x ;;
    -h | --help)    usage ;;
    *)
      log "ERROR" "Unknown option: $arg"
      usage
      ;;
  esac
done
```

### Section 4 — Read config file (Topics 3, 4, 5)

```bash
# ── Load config (Topics 3 and 4) ──────────────────────
SERVICES=()
DISK_CHECKS=()
CHECK_PORT=""
ALERT_EMAIL=""

[ -f "$CONFIG_FILE" ] || {
  log "ERROR" "Config file not found: $CONFIG_FILE"
  exit 1
}

log "INFO" "Loading config: $CONFIG_FILE"

while IFS= read -r line; do
  # Skip blank lines and comments (Topic 4)
  [[ -z "$line" || "$line" =~ ^# ]] && continue

  # Parse each line type (Topics 2 and 4)
  if [[ "$line" =~ ^SERVICE:(.+)$ ]]; then
    SERVICES+=("${BASH_REMATCH[1]}")

  elif [[ "$line" =~ ^DISK:(.+),([0-9]+)$ ]]; then
    DISK_CHECKS+=("${BASH_REMATCH[1]}:${BASH_REMATCH[2]}")

  elif [[ "$line" =~ ^PORT:([0-9]+)$ ]]; then
    CHECK_PORT="${BASH_REMATCH[1]}"

  elif [[ "$line" =~ ^ALERT_EMAIL:(.+)$ ]]; then
    ALERT_EMAIL="${BASH_REMATCH[1]}"
  fi

done < "$CONFIG_FILE"

log "DEBUG" "Services loaded : ${SERVICES[*]:-none}"
log "DEBUG" "Disk checks     : ${DISK_CHECKS[*]:-none}"
log "DEBUG" "Port check      : ${CHECK_PORT:-none}"
```

### Section 5 — Check functions (Topics 1, 3, 5)

```bash
# ── Counters ──────────────────────────────────────────
PASS_COUNT=0
FAIL_COUNT=0
WARN_COUNT=0

# ── Service check (Topics 1 and 5) ────────────────────
check_service() {
  local service=$1

  if systemctl is-active --quiet "$service" 2>/dev/null; then
    log "INFO" "  SERVICE  $service — running ✓"
    PASS_COUNT=$(( PASS_COUNT + 1 ))
    return 0
  else
    log "ERROR" "  SERVICE  $service — STOPPED ✗"
    FAIL_COUNT=$(( FAIL_COUNT + 1 ))
    return 1
  fi
}

# ── Disk check (Topics 2 and 5) ───────────────────────
check_disk() {
  local entry=$1

  # Split path and threshold (Topic 2)
  local path threshold
  IFS=':' read -r path threshold <<< "$entry"

  # Get disk usage as plain number (Topics 2 and 5)
  local usage_pct
  usage_pct=$(df "$path" 2>/dev/null \
    | awk 'NR==2{print $5}' \
    | tr -d '%')

  if [ -z "$usage_pct" ]; then
    log "WARN" "  DISK     $path — could not read usage"
    WARN_COUNT=$(( WARN_COUNT + 1 ))
    return 0
  fi

  if [ "$usage_pct" -ge "$threshold" ]; then
    log "ERROR" "  DISK     $path — ${usage_pct}% used (threshold: ${threshold}%) ✗"
    FAIL_COUNT=$(( FAIL_COUNT + 1 ))
    return 1
  else
    log "INFO" "  DISK     $path — ${usage_pct}% used (threshold: ${threshold}%) ✓"
    PASS_COUNT=$(( PASS_COUNT + 1 ))
    return 0
  fi
}

# ── Port check (Topics 1 and 5) ───────────────────────
check_port() {
  local port=$1

  if ss -tlnp 2>/dev/null | grep -q ":${port} "; then
    log "INFO" "  PORT     $port — listening ✓"
    PASS_COUNT=$(( PASS_COUNT + 1 ))
  else
    log "WARN" "  PORT     $port — nothing listening"
    WARN_COUNT=$(( WARN_COUNT + 1 ))
  fi
}
```

### Section 6 — Run all checks (Topics 3 and 4)

```bash
# ── Run all checks ────────────────────────────────────
run_checks() {
  local hostname
  hostname=$(hostname)
  local check_time
  check_time=$(date '+%Y-%m-%d %H:%M:%S')

  separator
  log "INFO" "Server Monitor — $hostname"
  log "INFO" "Check time — $check_time"
  separator

  # Service checks (Topic 3 — loop over array)
  if [ ${#SERVICES[@]} -gt 0 ]; then
    log "INFO" "Services:"
    for service in "${SERVICES[@]}"; do
      check_service "$service" || true
    done
  fi

  # Disk checks (Topic 3 — loop over array)
  if [ ${#DISK_CHECKS[@]} -gt 0 ]; then
    log "INFO" "Disk:"
    for entry in "${DISK_CHECKS[@]}"; do
      check_disk "$entry" || true
    done
  fi

  # Port check (Topic 5)
  if [ -n "$CHECK_PORT" ]; then
    log "INFO" "Port:"
    check_port "$CHECK_PORT"
  fi

  separator
}
```

### Section 7 — Summary report (Topics 2, 4, 5)

```bash
# ── Generate report (Topics 2 and 5) ──────────────────
generate_report() {
  local total=$(( PASS_COUNT + FAIL_COUNT + WARN_COUNT ))
  local hostname
  hostname=$(hostname)

  log "INFO" "Summary:"
  log "INFO" "  Total checks : $total"
  log "INFO" "  Passed       : $PASS_COUNT ✓"
  log "INFO" "  Warnings     : $WARN_COUNT ⚠"
  log "INFO" "  Failed       : $FAIL_COUNT ✗"
  separator

  # Overall health
  if [ $FAIL_COUNT -gt 0 ]; then
    log "ERROR" "Status: UNHEALTHY — $FAIL_COUNT check(s) failed"
  elif [ $WARN_COUNT -gt 0 ]; then
    log "WARN"  "Status: DEGRADED — $WARN_COUNT warning(s)"
  else
    log "INFO"  "Status: HEALTHY — all checks passed ✓"
  fi

  separator

  # Save to report file if requested (Topic 4 — write to file)
  if $GENERATE_REPORT; then
    mkdir -p "$REPORT_DIR"

    cat > "$REPORT_FILE" << EOF
Server Monitor Report
═════════════════════════════════════════
Host    : $hostname
Date    : $(date '+%Y-%m-%d %H:%M:%S')
═════════════════════════════════════════
Total   : $total
Passed  : $PASS_COUNT
Warnings: $WARN_COUNT
Failed  : $FAIL_COUNT
═════════════════════════════════════════
Status  : $([ $FAIL_COUNT -gt 0 ] && echo "UNHEALTHY" || echo "HEALTHY")
EOF

    log "INFO" "Report saved: $REPORT_FILE"
  fi

  # Exit with failure code if any checks failed (Topic 1)
  [ $FAIL_COUNT -gt 0 ] && exit 1 || exit 0
}
```

### Section 8 — Main entry point

```bash
# ── Main ──────────────────────────────────────────────
mkdir -p "$REPORT_DIR"

require_binary "df"
require_binary "ss"
require_binary "systemctl"

run_checks
generate_report
```

---

## Step 5 — Run It and See the Output

```bash
# Make it executable
chmod +x monitor.sh

# Run normally
./monitor.sh
```

**Output:**
```
[2024-06-12 09:15:32] [INFO ] Loading config: /opt/server-monitor/config/monitor.conf
────────────────────────────────────────
[2024-06-12 09:15:32] [INFO ] Server Monitor — prod-server-01
[2024-06-12 09:15:32] [INFO ] Check time — 2024-06-12 09:15:32
────────────────────────────────────────
[2024-06-12 09:15:32] [INFO ] Services:
[2024-06-12 09:15:32] [INFO ]   SERVICE  nginx — running ✓
[2024-06-12 09:15:32] [INFO ]   SERVICE  ssh — running ✓
[2024-06-12 09:15:32] [ERROR]   SERVICE  cron — STOPPED ✗
[2024-06-12 09:15:32] [INFO ] Disk:
[2024-06-12 09:15:32] [INFO ]   DISK     / — 42% used (threshold: 90%) ✓
[2024-06-12 09:15:32] [INFO ]   DISK     /tmp — 12% used (threshold: 80%) ✓
[2024-06-12 09:15:32] [INFO ]   DISK     /var/log — 67% used (threshold: 85%) ✓
[2024-06-12 09:15:32] [INFO ] Port:
[2024-06-12 09:15:32] [WARN ]   PORT     3000 — nothing listening
────────────────────────────────────────
[2024-06-12 09:15:32] [INFO ] Summary:
[2024-06-12 09:15:32] [INFO ]   Total checks : 6
[2024-06-12 09:15:32] [INFO ]   Passed       : 4 ✓
[2024-06-12 09:15:32] [WARN ]   Warnings     : 1 ⚠
[2024-06-12 09:15:32] [ERROR]   Failed       : 1 ✗
────────────────────────────────────────
[2024-06-12 09:15:32] [ERROR] Status: UNHEALTHY — 1 check(s) failed
────────────────────────────────────────
```

---

## Step 6 — Add the Report File

```bash
# Run with --report to save summary
./monitor.sh --report
```

```bash
# View the saved report
cat reports/report-20240612.txt
```

```
Server Monitor Report
═════════════════════════════════════════
Host    : prod-server-01
Date    : 2024-06-12 09:15:32
═════════════════════════════════════════
Total   : 6
Passed  : 4
Warnings: 1
Failed  : 1
═════════════════════════════════════════
Status  : UNHEALTHY
```

---

## Step 7 — Schedule with Cron

Run the monitor automatically every 5 minutes:

```bash
# Open crontab
crontab -e

# Add this line
*/5 * * * * /opt/server-monitor/monitor.sh --report >> /var/log/monitor-cron.log 2>&1
```

**Other useful schedules:**

```bash
# Every 5 minutes
*/5 * * * * /opt/server-monitor/monitor.sh

# Every hour
0 * * * * /opt/server-monitor/monitor.sh --report

# Daily at 8AM
0 8 * * * /opt/server-monitor/monitor.sh --report

# Debug run manually anytime
DEBUG=1 ./monitor.sh --report
```

---

## Full Final Script

Here is the complete `monitor.sh` — all sections together:

```bash
#!/bin/bash
# monitor.sh — Server monitoring tool
# Usage: ./monitor.sh [--report] [--debug]

set -euo pipefail

# ── Script directory ───────────────────────────────────
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# ── Source shared library ──────────────────────────────
source "$SCRIPT_DIR/lib/common.sh"

# ── Config ────────────────────────────────────────────
CONFIG_FILE="$SCRIPT_DIR/config/monitor.conf"
REPORT_DIR="$SCRIPT_DIR/reports"
LOG_FILE="$REPORT_DIR/monitor.log"
REPORT_FILE="$REPORT_DIR/report-$(date '+%Y%m%d').txt"
DEBUG="${DEBUG:-0}"
GENERATE_REPORT=false

[ "$DEBUG" = "1" ] && set -x

handle_error() {
  log "ERROR" "Failed at line $1 — command: $BASH_COMMAND"
}
trap 'handle_error $LINENO' ERR

# ── Usage ─────────────────────────────────────────────
usage() {
  cat << EOF

  Usage: $(basename "$0") [options]

  Options:
    --report       Save summary to $REPORT_DIR
    --debug        Show debug output
    -h, --help     Show this message

  Examples:
    $(basename "$0")
    $(basename "$0") --report
    DEBUG=1 $(basename "$0")

EOF
  exit 0
}

# ── Parse arguments ────────────────────────────────────
for arg in "$@"; do
  case $arg in
    --report)    GENERATE_REPORT=true ;;
    --debug)     DEBUG=1; set -x ;;
    -h|--help)   usage ;;
    *)
      log "ERROR" "Unknown option: $arg"
      usage
      ;;
  esac
done

# ── Load config ───────────────────────────────────────
SERVICES=()
DISK_CHECKS=()
CHECK_PORT=""
ALERT_EMAIL=""

[ -f "$CONFIG_FILE" ] || {
  log "ERROR" "Config file not found: $CONFIG_FILE"
  exit 1
}

log "INFO" "Loading config: $CONFIG_FILE"

while IFS= read -r line; do
  [[ -z "$line" || "$line" =~ ^# ]] && continue

  if [[ "$line" =~ ^SERVICE:(.+)$ ]]; then
    SERVICES+=("${BASH_REMATCH[1]}")
  elif [[ "$line" =~ ^DISK:(.+),([0-9]+)$ ]]; then
    DISK_CHECKS+=("${BASH_REMATCH[1]}:${BASH_REMATCH[2]}")
  elif [[ "$line" =~ ^PORT:([0-9]+)$ ]]; then
    CHECK_PORT="${BASH_REMATCH[1]}"
  elif [[ "$line" =~ ^ALERT_EMAIL:(.+)$ ]]; then
    ALERT_EMAIL="${BASH_REMATCH[1]}"
  fi

done < "$CONFIG_FILE"

# ── Counters ──────────────────────────────────────────
PASS_COUNT=0
FAIL_COUNT=0
WARN_COUNT=0

# ── Check functions ───────────────────────────────────
check_service() {
  local service=$1
  if systemctl is-active --quiet "$service" 2>/dev/null; then
    log "INFO"  "  SERVICE  $service — running ✓"
    PASS_COUNT=$(( PASS_COUNT + 1 ))
  else
    log "ERROR" "  SERVICE  $service — STOPPED ✗"
    FAIL_COUNT=$(( FAIL_COUNT + 1 ))
  fi
}

check_disk() {
  local entry=$1
  local path threshold
  IFS=':' read -r path threshold <<< "$entry"

  local usage_pct
  usage_pct=$(df "$path" 2>/dev/null \
    | awk 'NR==2{print $5}' \
    | tr -d '%')

  if [ -z "$usage_pct" ]; then
    log "WARN"  "  DISK     $path — could not read usage"
    WARN_COUNT=$(( WARN_COUNT + 1 ))
    return 0
  fi

  if [ "$usage_pct" -ge "$threshold" ]; then
    log "ERROR" "  DISK     $path — ${usage_pct}% used (threshold: ${threshold}%) ✗"
    FAIL_COUNT=$(( FAIL_COUNT + 1 ))
  else
    log "INFO"  "  DISK     $path — ${usage_pct}% used (threshold: ${threshold}%) ✓"
    PASS_COUNT=$(( PASS_COUNT + 1 ))
  fi
}

check_port() {
  local port=$1
  if ss -tlnp 2>/dev/null | grep -q ":${port} "; then
    log "INFO"  "  PORT     $port — listening ✓"
    PASS_COUNT=$(( PASS_COUNT + 1 ))
  else
    log "WARN"  "  PORT     $port — nothing listening"
    WARN_COUNT=$(( WARN_COUNT + 1 ))
  fi
}

# ── Run all checks ────────────────────────────────────
run_checks() {
  separator
  log "INFO" "Server Monitor — $(hostname)"
  log "INFO" "Check time — $(date '+%Y-%m-%d %H:%M:%S')"
  separator

  if [ ${#SERVICES[@]} -gt 0 ]; then
    log "INFO" "Services:"
    for service in "${SERVICES[@]}"; do
      check_service "$service"
    done
  fi

  if [ ${#DISK_CHECKS[@]} -gt 0 ]; then
    log "INFO" "Disk:"
    for entry in "${DISK_CHECKS[@]}"; do
      check_disk "$entry"
    done
  fi

  if [ -n "$CHECK_PORT" ]; then
    log "INFO" "Port:"
    check_port "$CHECK_PORT"
  fi

  separator
}

# ── Summary ───────────────────────────────────────────
generate_report() {
  local total=$(( PASS_COUNT + FAIL_COUNT + WARN_COUNT ))

  log "INFO" "Summary:"
  log "INFO" "  Total checks : $total"
  log "INFO" "  Passed       : $PASS_COUNT ✓"
  log "INFO" "  Warnings     : $WARN_COUNT ⚠"
  log "INFO" "  Failed       : $FAIL_COUNT ✗"
  separator

  if [ $FAIL_COUNT -gt 0 ]; then
    log "ERROR" "Status: UNHEALTHY — $FAIL_COUNT check(s) failed"
  elif [ $WARN_COUNT -gt 0 ]; then
    log "WARN"  "Status: DEGRADED — $WARN_COUNT warning(s)"
  else
    log "INFO"  "Status: HEALTHY — all checks passed ✓"
  fi
  separator

  if $GENERATE_REPORT; then
    mkdir -p "$REPORT_DIR"
    cat > "$REPORT_FILE" << EOF
Server Monitor Report
═════════════════════════════════════════
Host    : $(hostname)
Date    : $(date '+%Y-%m-%d %H:%M:%S')
═════════════════════════════════════════
Total   : $total
Passed  : $PASS_COUNT
Warnings: $WARN_COUNT
Failed  : $FAIL_COUNT
═════════════════════════════════════════
Status  : $([ $FAIL_COUNT -gt 0 ] && echo "UNHEALTHY" || echo "HEALTHY")
EOF
    log "INFO" "Report saved: $REPORT_FILE"
  fi

  [ $FAIL_COUNT -gt 0 ] && exit 1 || exit 0
}

# ── Main ──────────────────────────────────────────────
mkdir -p "$REPORT_DIR"
require_binary "df"
require_binary "ss"
require_binary "systemctl"

run_checks
generate_report
```

---

## Key Takeaways

### What you used from each topic

```
Topic 1  → exit codes on every check — 0 for pass, 1 for fail
Topic 1+ → set -euo pipefail + trap ERR at the top of monitor.sh
Topic 2  → string split for DISK:path:threshold, tr -d '%' for numbers
Topic 3  → SERVICES=() and DISK_CHECKS=() arrays, loops over them
Topic 4  → while IFS= read -r line; done < config to read monitor.conf
Topic 5  → $(hostname), $(date), $(df ...) — command substitution everywhere
Topic 6  → DEBUG=1 enables set -x, trap ERR catches failures
Topic 7  → $SCRIPT_DIR, source lib/common.sh, usage(), argument parsing
```

### The bigger lessons

| Lesson | Where it showed up |
|---|---|
| Separate config from logic | `config/monitor.conf` — change what to check without touching the script |
| Shared functions in one place | `lib/common.sh` — `log()` used everywhere, fixed once |
| Fail loudly with context | `trap ERR` shows line number and command |
| Exit codes matter | `exit 1` when checks fail — CI/CD sees it |
| Build incrementally | Each step added one thing — nothing all at once |
| Reusability | `--report`, `--debug`, `-h` flags make it work for anyone |

---

## What to Try Next

Now that the intermediate series is complete, here are ways to extend the project:

**Easy extensions:**
- Add memory usage check using `free -m`
- Add a check for a specific process using `pgrep`
- Colour the terminal output — green for pass, red for fail

**Medium extensions:**
- Send a Slack alert when a check fails (use `curl` + webhook)
- Add an email alert using `mail` command
- Read multiple config files from a directory

**Hard extensions (bridges into the advanced series):**
- Deploy using `monitor.sh` as a preflight check before running `deploy.sh`
- Integrate with the log parsing from the advanced concepts
- Add check history — compare today's report with yesterday's

---

## You Have Completed the Intermediate Series

```
Topic 1   → Exit Codes & $?            ✅
Topic 1+  → Script Safety Patterns     ✅
Topic 2   → String Manipulation        ✅
Topic 3   → Arrays                     ✅
Topic 4   → Reading Files              ✅
Topic 5   → Command Substitution       ✅
Topic 6   → Debugging Scripts          ✅
Topic 7   → Writing Reusable Scripts   ✅
Topic 8   → Real-World Mini Project    ✅
```

You now have the skills to write production-quality shell scripts —
safe, readable, reusable, and debuggable.

The advanced series builds directly on this foundation —
deploying apps, managing logs, sending notifications,
and automating everything a DevOps engineer does every day.