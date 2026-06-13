# Intermediate Topic 5 — Command Substitution & Pipes

> **Session:** Shell Scripting for DevOps — Intermediate Concepts  
> **Duration:** ~20 minutes  
> **Prerequisite:** Topics 1, 1+, 2, 3, 4 must be complete

---

## Table of Contents

- [What Are Command Substitution and Pipes?](#what-are-command-substitution-and-pipes)
- [Why DevOps Engineers Need This](#why-devops-engineers-need-this)
- [Part 1 — Command Substitution](#part-1--command-substitution)
  - [Basic Syntax](#basic-syntax)
  - [Storing Output in Variables](#storing-output-in-variables)
  - [Using Output Inline](#using-output-inline)
  - [Nesting Command Substitution](#nesting-command-substitution)
- [Part 2 — Pipes](#part-2--pipes)
  - [What a Pipe Does](#what-a-pipe-does)
  - [Chaining Commands](#chaining-commands)
  - [Common Pipe Combinations](#common-pipe-combinations)
- [Part 3 — Combining Both](#part-3--combining-both)
- [Real DevOps Examples](#real-devops-examples)
- [Common Mistakes](#common-mistakes)
- [Quick Exercise](#quick-exercise)

---

## What Are Command Substitution and Pipes?

### Command Substitution

Runs a command and **captures its output** as a value you can store
or use directly:

```bash
# Without command substitution — two steps
date '+%Y-%m-%d'         # prints 2024-06-12
TODAY="2024-06-12"       # hardcode it manually — bad

# With command substitution — one step
TODAY=$(date '+%Y-%m-%d')   # run the command, capture output
echo "$TODAY"               # 2024-06-12
```

### Pipes

Sends the **output of one command** directly into the **input of another**:

```bash
# Without pipe — two steps, temp file needed
ls /var/log > /tmp/files.txt
grep ".log" /tmp/files.txt

# With pipe — one step, no temp file
ls /var/log | grep ".log"
```

---

## Why DevOps Engineers Need This

These two are the **backbone of every real DevOps script**:

| Task | How |
|---|---|
| Get today's date for a filename | `$(date '+%Y%m%d')` |
| Count running processes | `$(ps aux \| wc -l)` |
| Get disk usage as a number | `$(df / \| awk 'NR==2{print $5}' \| tr -d '%')` |
| Find the current git branch | `$(git branch --show-current)` |
| Get server's IP address | `$(hostname -I \| awk '{print $1}')` |
| Count lines in a log file | `$(wc -l < app.log)` |
| Find PID of a process | `$(pgrep -f myapp)` |

Without these, you would have to hardcode values or use temp files everywhere.

---

## Part 1 — Command Substitution

### Basic Syntax

There are two syntaxes — always use the modern `$()` style:

```bash
# Old style — backticks (avoid this)
TODAY=`date '+%Y-%m-%d'`

# Modern style — $() — always use this
TODAY=$(date '+%Y-%m-%d')
```

Why `$()` is better:
- Easier to read
- Can be nested
- Works inside double quotes
- No confusion with single quotes

---

### Storing Output in Variables

```bash
#!/bin/bash
set -euo pipefail

# Date and time
TODAY=$(date '+%Y-%m-%d')
NOW=$(date '+%Y-%m-%d %H:%M:%S')
TIMESTAMP=$(date '+%Y%m%d%H%M%S')

echo "Today    : $TODAY"
echo "Now      : $NOW"
echo "Timestamp: $TIMESTAMP"
```

**Output:**
```
Today    : 2024-06-12
Now      : 2024-06-12 09:15:32
Timestamp: 20240612091532
```

```bash
#!/bin/bash
set -euo pipefail

# System information
HOSTNAME=$(hostname)
OS=$(uname -s)
KERNEL=$(uname -r)
CPU_COUNT=$(nproc)
MEMORY_MB=$(free -m | awk 'NR==2{print $2}')
DISK_USAGE=$(df / | awk 'NR==2{print $5}')
CURRENT_USER=$(whoami)
UPTIME=$(uptime -p 2>/dev/null || uptime | awk -F'up ' '{print $2}')

echo "Hostname : $HOSTNAME"
echo "OS       : $OS"
echo "Kernel   : $KERNEL"
echo "CPUs     : $CPU_COUNT"
echo "Memory   : ${MEMORY_MB}MB"
echo "Disk     : $DISK_USAGE used"
echo "User     : $CURRENT_USER"
echo "Uptime   : $UPTIME"
```

**Output:**
```
Hostname : prod-server-01
OS       : Linux
Kernel   : 5.15.0-91-generic
CPUs     : 4
Memory   : 7982MB
Disk     : 42% used
Uptime   : up 14 days, 3 hours
```

---

### Using Output Inline

You do not always need to store the output first —
use it directly inside a string:

```bash
#!/bin/bash
set -euo pipefail

APP_NAME="myapp"
VERSION="1.4.2"

# Use command substitution inline inside strings
LOG_FILE="/var/log/$APP_NAME/deploy-$(date '+%Y%m%d').log"
BACKUP_NAME="${APP_NAME}-$(date '+%Y%m%d%H%M%S').tar.gz"
RELEASE_DIR="/opt/$APP_NAME/releases/$VERSION-$(date '+%Y%m%d')"

echo "Log file    : $LOG_FILE"
echo "Backup name : $BACKUP_NAME"
echo "Release dir : $RELEASE_DIR"
```

**Output:**
```
Log file    : /var/log/myapp/deploy-20240612.log
Backup name : myapp-20240612091532.tar.gz
Release dir : /opt/myapp/releases/1.4.2-20240612
```

---

### Nesting Command Substitution

You can put a `$()` inside another `$()`:

```bash
#!/bin/bash
set -euo pipefail

# Get disk usage of the directory where the script lives
SCRIPT_DIR=$(dirname "$(realpath "$0")")
echo "Script is in: $SCRIPT_DIR"

# Get the latest file in a directory
LATEST_FILE=$(ls -t /var/log/myapp/*.log | head -1)
LATEST_SIZE=$(du -sh "$(ls -t /var/log/myapp/*.log | head -1)" | cut -f1)
echo "Latest log: $LATEST_FILE ($LATEST_SIZE)"
```

> Keep nesting to a maximum of two levels —
> deeper than that becomes hard to read.

---

## Part 2 — Pipes

### What a Pipe Does

A pipe `|` connects two commands:

```
command1 | command2
```

- `command1` runs and produces output
- That output becomes the **input** of `command2`
- `command2` processes it and produces its own output

Think of it like an assembly line:

```
Raw material → Machine 1 → Machine 2 → Machine 3 → Final product
ls /var/log  → grep .log → sort       → head -5   → Top 5 log files
```

---

### Chaining Commands

```bash
#!/bin/bash
set -euo pipefail

# One pipe — filter
ls /var/log | grep "\.log"

# Two pipes — filter then count
ls /var/log | grep "\.log" | wc -l

# Three pipes — filter, sort, show top 5
ps aux | grep "nginx" | sort -k3 -rn | head -5
```

**Build it step by step — always teach this way:**

```bash
# Step 1 — start with basic command
ps aux

# Step 2 — filter by process name
ps aux | grep "nginx"

# Step 3 — sort by CPU usage (column 3)
ps aux | grep "nginx" | sort -k3 -rn

# Step 4 — show only top 5
ps aux | grep "nginx" | sort -k3 -rn | head -5
```

> Always build pipes **one step at a time**.
> Run each step to see its output before adding the next.

---

### Common Pipe Combinations

These are the combinations DevOps engineers use every single day:

```bash
#!/bin/bash
set -euo pipefail

# ── Counting ──────────────────────────────────────────

# Count running processes
ps aux | wc -l

# Count log files
ls /var/log | grep "\.log" | wc -l

# Count error lines in a log
grep "\[ERROR\]" /var/log/myapp/app.log | wc -l

# ── Filtering ─────────────────────────────────────────

# Find all nginx processes
ps aux | grep nginx | grep -v grep

# Find log files larger than a keyword match
ls -lh /var/log | grep "\.log"

# Find lines containing ERROR but not DEBUG
grep "ERROR" app.log | grep -v "DEBUG"

# ── Extracting ────────────────────────────────────────

# Get just the process IDs of nginx
ps aux | grep nginx | grep -v grep | awk '{print $2}'

# Get just the IP addresses from a log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq

# Get disk usage percentage as a plain number
df / | awk 'NR==2{print $5}' | tr -d '%'

# ── Sorting ───────────────────────────────────────────

# Top 5 IPs hitting nginx
awk '{print $1}' /var/log/nginx/access.log \
  | sort \
  | uniq -c \
  | sort -rn \
  | head -5

# Largest files in /var/log
du -sh /var/log/* | sort -rh | head -10
```

---

## Part 3 — Combining Both

The real power comes from using command substitution AND pipes together:

```bash
#!/bin/bash
set -euo pipefail

# Capture piped output into a variable
ERROR_COUNT=$(grep "\[ERROR\]" /var/log/myapp/app.log | wc -l)
echo "Errors found: $ERROR_COUNT"

# Get top IP from nginx log
TOP_IP=$(awk '{print $1}' /var/log/nginx/access.log \
  | sort | uniq -c | sort -rn | head -1 | awk '{print $2}')
echo "Top IP: $TOP_IP"

# Get disk usage as a number (for comparison)
DISK_PCT=$(df / | awk 'NR==2{print $5}' | tr -d '%')
echo "Disk usage: ${DISK_PCT}%"

if [ "$DISK_PCT" -gt 80 ]; then
  echo "WARNING: Disk above 80%"
fi

# Get memory usage
MEM_FREE=$(free -m | awk 'NR==2{print $4}')
MEM_TOTAL=$(free -m | awk 'NR==2{print $2}')
MEM_PCT=$(awk "BEGIN {printf \"%d\", ($MEM_FREE/$MEM_TOTAL)*100}")
echo "Free memory: ${MEM_FREE}MB (${MEM_PCT}%)"

# Count active connections
CONN_COUNT=$(ss -tn state established | wc -l)
echo "Active connections: $CONN_COUNT"

# Get PID of a running service
if APP_PID=$(pgrep -f "node app.js" 2>/dev/null); then
  echo "App is running — PID: $APP_PID"
else
  echo "App is NOT running"
fi
```

---

## Real DevOps Examples

### Example 1 — System health snapshot

```bash
#!/bin/bash
set -euo pipefail

APP_NAME="myapp"

# Collect all metrics using command substitution
HOSTNAME=$(hostname)
DATE=$(date '+%Y-%m-%d %H:%M:%S')
DISK_PCT=$(df / | awk 'NR==2{print $5}' | tr -d '%')
MEM_FREE_MB=$(free -m | awk 'NR==2{print $4}')
CPU_LOAD=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | tr -d ',')
APP_ERRORS=$(grep -c "\[ERROR\]" /var/log/$APP_NAME/app.log 2>/dev/null || echo 0)
NGINX_5XX=$(awk '$9 ~ /^5/' /var/log/nginx/access.log 2>/dev/null | wc -l || echo 0)
OPEN_CONNECTIONS=$(ss -tn state established 2>/dev/null | wc -l || echo 0)

# Print formatted report
echo "════ Health Snapshot: $HOSTNAME ════"
echo "Time             : $DATE"
echo "Disk usage       : ${DISK_PCT}%"
echo "Free memory      : ${MEM_FREE_MB}MB"
echo "CPU load (1min)  : $CPU_LOAD"
echo "App errors       : $APP_ERRORS"
echo "Nginx 5xx        : $NGINX_5XX"
echo "Open connections : $OPEN_CONNECTIONS"
echo "══════════════════════════════════════"

# Alert on thresholds
[ "$DISK_PCT"  -gt 85 ] && echo "⚠️  ALERT: Disk above 85%"
[ "$MEM_FREE_MB" -lt 200 ] && echo "⚠️  ALERT: Low memory (${MEM_FREE_MB}MB free)"
[ "$APP_ERRORS" -gt 50 ] && echo "⚠️  ALERT: High error count ($APP_ERRORS)"
```

**Output:**
```
════ Health Snapshot: prod-server-01 ════
Time             : 2024-06-12 09:15:32
Disk usage       : 42%
Free memory      : 1823MB
CPU load (1min)  : 0.32
App errors       : 7
Nginx 5xx        : 23
Open connections : 148
══════════════════════════════════════
```

---

### Example 2 — Find and kill a zombie process

```bash
#!/bin/bash
set -euo pipefail

PROCESS_NAME="${1:-}"
[ -z "$PROCESS_NAME" ] && { echo "Usage: $0 <process-name>"; exit 1; }

echo "Looking for: $PROCESS_NAME"

# Find all matching PIDs
PIDS=$(pgrep -f "$PROCESS_NAME" 2>/dev/null || echo "")

if [ -z "$PIDS" ]; then
  echo "No process found matching: $PROCESS_NAME"
  exit 0
fi

# Count how many
PID_COUNT=$(echo "$PIDS" | wc -l)
echo "Found $PID_COUNT process(es):"

# Show details for each PID
echo "$PIDS" | while read -r pid; do
  PROCESS_INFO=$(ps -p "$pid" -o pid,ppid,cmd --no-headers 2>/dev/null || echo "gone")
  echo "  $PROCESS_INFO"
done

# Ask to kill (in real scripts this might be automatic)
echo ""
echo "Killing $PID_COUNT process(es)..."
echo "$PIDS" | while read -r pid; do
  kill -TERM "$pid" 2>/dev/null && echo "  Killed PID: $pid" || echo "  Could not kill PID: $pid"
done
```

---

### Example 3 — Build a release filename dynamically

```bash
#!/bin/bash
set -euo pipefail

APP_NAME="myapp"
VERSION="${1:-}"

[ -z "$VERSION" ] && { echo "Usage: $0 <version>"; exit 1; }

# Build all the names dynamically
GIT_HASH=$(git rev-parse --short HEAD 2>/dev/null || echo "nogit")
BUILD_DATE=$(date '+%Y%m%d')
BUILD_TIME=$(date '+%H%M%S')
BUILD_USER=$(whoami)
HOST=$(hostname -s)

ARTIFACT_NAME="${APP_NAME}-${VERSION}-${BUILD_DATE}-${GIT_HASH}.tar.gz"
RELEASE_TAG="${VERSION}-${BUILD_DATE}"
DEPLOY_LOG="deploy-${RELEASE_TAG}-${BUILD_TIME}.log"

echo "Artifact  : $ARTIFACT_NAME"
echo "Tag       : $RELEASE_TAG"
echo "Log file  : $DEPLOY_LOG"
echo "Built by  : $BUILD_USER @ $HOST"
echo "Git hash  : $GIT_HASH"
```

**Output:**
```
Artifact  : myapp-1.4.2-20240612-a3f1b2c.tar.gz
Tag       : 1.4.2-20240612
Log file  : deploy-1.4.2-20240612-091532.log
Built by  : deploy @ prod-server-01
Git hash  : a3f1b2c
```

---

### Example 4 — Check if port is in use

```bash
#!/bin/bash
set -euo pipefail

check_port() {
  local port=$1

  # Capture ss output into a variable
  local result
  result=$(ss -tlnp 2>/dev/null | grep ":${port} " || echo "")

  if [ -n "$result" ]; then
    # Extract process name from the result
    local process
    process=$(echo "$result" | grep -oP 'users:\(\("\K[^"]+' || echo "unknown")
    echo "Port $port is IN USE by: $process"
    return 1
  else
    echo "Port $port is FREE ✓"
    return 0
  fi
}

# Check common ports
PORTS=(80 443 3000 3306 5432 6379 8080)
for port in "${PORTS[@]}"; do
  check_port "$port"
done
```

**Output:**
```
Port 80   is IN USE by: nginx
Port 443  is IN USE by: nginx
Port 3000 is IN USE by: node
Port 3306 is FREE ✓
Port 5432 is FREE ✓
Port 6379 is IN USE by: redis-server
Port 8080 is FREE ✓
```

---

## Common Mistakes

### Mistake 1 — Not quoting command substitution

```bash
# Wrong — breaks if output has spaces
FILENAME=$(ls /var/log | head -1)
cp $FILENAME /backup/    # breaks if filename has spaces

# Correct — always quote
cp "$FILENAME" /backup/
```

---

### Mistake 2 — Ignoring errors in command substitution

```bash
# Wrong — if command fails, variable gets empty string silently
VERSION=$(git describe --tags 2>/dev/null)
echo "Version: $VERSION"    # empty — no error shown

# Correct — check if output is empty
VERSION=$(git describe --tags 2>/dev/null || echo "unknown")
echo "Version: $VERSION"    # unknown — clear fallback
```

---

### Mistake 3 — Using command substitution for large output

```bash
# Wrong — reading a 1GB log file into a variable
CONTENT=$(cat /var/log/huge-file.log)    # runs out of memory

# Correct — use while read for large files
while IFS= read -r line; do
  process "$line"
done < /var/log/huge-file.log
```

---

### Mistake 4 — Forgetting `grep -v grep` in process searches

```bash
# Wrong — the grep command itself appears in results
ps aux | grep "nginx"
# Output includes: "grep nginx" — false positive

# Correct — exclude grep itself
ps aux | grep "nginx" | grep -v grep

# Even better — use pgrep
pgrep -f nginx
```

---

### Mistake 5 — Not handling empty output

```bash
# Wrong — crashes if no process found
PID=$(pgrep myapp)
kill "$PID"    # error if PID is empty

# Correct — check before using
if PID=$(pgrep myapp 2>/dev/null); then
  echo "Killing myapp PID: $PID"
  kill "$PID"
else
  echo "myapp is not running"
fi
```

---

## Quick Exercise

> **Task:** Write a script called `server-stats.sh` that collects
> server information using command substitution and pipes, then
> prints a formatted report.
>
> It should collect and display:
>
> 1. **Hostname** using `hostname`
> 2. **Current date and time** using `date`
> 3. **Disk usage %** on `/` — just the number, no `%` sign
> 4. **Free memory in MB** using `free -m`
> 5. **Number of running processes** using `ps aux | wc -l`
> 6. **Current logged-in user** using `whoami`
> 7. **Top 3 largest files** in `/var/log` using `du` and `sort`
>
> Then check:
> - If disk usage > 80 → print `"WARNING: Disk is high"`
> - If free memory < 500MB → print `"WARNING: Low memory"`
> - Otherwise → print `"System looks healthy"`

**Expected output:**
```
════ Server Stats ════════════════════
Hostname   : prod-server-01
Date/Time  : 2024-06-12 09:15:32
Disk usage : 42%
Free memory: 1823MB
Processes  : 142
User       : ubuntu
══════════════════════════════════════
Top 3 largest files in /var/log:
  2.1G  /var/log/syslog
  450M  /var/log/auth.log
  120M  /var/log/kern.log
══════════════════════════════════════
System looks healthy
```

**Expected solution:**

```bash
#!/bin/bash
set -euo pipefail

# Collect metrics
HOSTNAME=$(hostname)
DATETIME=$(date '+%Y-%m-%d %H:%M:%S')
DISK_PCT=$(df / | awk 'NR==2{print $5}' | tr -d '%')
MEM_FREE=$(free -m | awk 'NR==2{print $4}')
PROCESS_COUNT=$(ps aux | wc -l)
CURRENT_USER=$(whoami)

# Print report
echo "════ Server Stats ════════════════════"
printf "%-11s: %s\n" "Hostname"    "$HOSTNAME"
printf "%-11s: %s\n" "Date/Time"   "$DATETIME"
printf "%-11s: %s%%\n" "Disk usage" "$DISK_PCT"
printf "%-11s: %sMB\n" "Free memory" "$MEM_FREE"
printf "%-11s: %s\n" "Processes"   "$PROCESS_COUNT"
printf "%-11s: %s\n" "User"        "$CURRENT_USER"
echo "══════════════════════════════════════"

# Top 3 largest files
echo "Top 3 largest files in /var/log:"
du -sh /var/log/* 2>/dev/null \
  | sort -rh \
  | head -3 \
  | while IFS= read -r line; do
      echo "  $line"
    done

echo "══════════════════════════════════════"

# Health checks
if [ "$DISK_PCT" -gt 80 ]; then
  echo "WARNING: Disk is high"
elif [ "$MEM_FREE" -lt 500 ]; then
  echo "WARNING: Low memory"
else
  echo "System looks healthy"
fi
```

---

## Summary — Quick Reference Card

| Operation | Syntax | Example |
|---|---|---|
| Capture output | `VAR=$(command)` | `DATE=$(date '+%Y-%m-%d')` |
| Use inline | `"text-$(command)"` | `"log-$(date).txt"` |
| Pipe output | `cmd1 \| cmd2` | `ps aux \| grep nginx` |
| Filter lines | `\| grep "pattern"` | `\| grep "ERROR"` |
| Exclude lines | `\| grep -v "pattern"` | `\| grep -v grep` |
| Count lines | `\| wc -l` | `ps aux \| wc -l` |
| Get Nth column | `\| awk '{print $N}'` | `\| awk '{print $2}'` |
| Remove character | `\| tr -d 'x'` | `\| tr -d '%'` |
| Sort output | `\| sort` | `\| sort -rn` |
| Unique values | `\| uniq` | `\| sort \| uniq -c` |
| First N lines | `\| head -N` | `\| head -5` |
| Last N lines | `\| tail -N` | `\| tail -10` |
| Fallback value | `$(cmd \|\| echo "default")` | `$(git log \|\| echo "no git")` |

---

## The Mental Model

```
Command Substitution = capture what a command PRINTS
                       and use it as a VALUE

Pipe                 = take what one command PRINTS
                       and feed it as INPUT to the next command

Together             = build powerful data pipelines
                       without temp files
```

---

## What's Next

**Intermediate Topic 6 — Debugging Scripts**  
Learn how to find and fix bugs in shell scripts using
`set -x`, `bash -n`, `echo` debugging, and `trap ERR` —
the toolkit every DevOps engineer needs when a script
misbehaves at 2AM.