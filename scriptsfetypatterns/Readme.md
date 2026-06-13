# Intermediate Topic 1+ — Script Safety Patterns

> **Session:** Shell Scripting for DevOps — Intermediate Concepts  
> **Duration:** ~20 minutes  
> **Prerequisite:** Topic 1 — Exit Codes & `$?`

---

## Table of Contents

- [What Are Script Safety Patterns?](#what-are-script-safety-patterns)
- [Why Every Real Script Needs This](#why-every-real-script-needs-this)
- [Pattern 1 — set -e](#pattern-1--set--e)
- [Pattern 2 — set -u](#pattern-2--set--u)
- [Pattern 3 — set -o pipefail](#pattern-3--set--o-pipefail)
- [Pattern 4 — The Holy Trinity Together](#pattern-4--the-holy-trinity-together)
- [Pattern 5 — trap for Cleanup](#pattern-5--trap-for-cleanup)
- [Putting It Together — Safe Script Template](#putting-it-together--safe-script-template)
- [Common Mistakes](#common-mistakes)
- [Quick Exercise](#quick-exercise)

---

## What Are Script Safety Patterns?

Script safety patterns are **a small set of rules you put at the top
of every script** that make the script behave safely — stopping
immediately when something goes wrong instead of running blindly
through failures.

Think of it like a car's safety systems:

> Without safety patterns → your script is a car with no brakes,
> no seatbelt, and no airbags.
>
> With safety patterns → it stops the moment something goes wrong,
> before it crashes into something expensive.

---

## Why Every Real Script Needs This

**Without safety patterns — what actually happens:**

```bash
#!/bin/bash

# This script has no safety patterns
cp app.tar.gz /opt/myapp/           # FAILS — file doesn't exist
tar -xzf /opt/myapp/app.tar.gz      # runs anyway — nothing to extract
systemctl restart myapp             # restarts with broken/missing files
echo "Deployment complete!"         # prints success — LIES
```

**Output:**
```
cp: cannot stat 'app.tar.gz': No such file or directory
tar: /opt/myapp/app.tar.gz: Cannot open: No such file or directory
Deployment complete!
```

> The script reports success. Your app is broken.
> Nobody knows. Users are hitting errors.

**With safety patterns — what should happen:**

```bash
#!/bin/bash
set -euo pipefail

cp app.tar.gz /opt/myapp/           # FAILS — script stops HERE
tar -xzf /opt/myapp/app.tar.gz      # never runs
systemctl restart myapp             # never runs
echo "Deployment complete!"         # never runs
```

**Output:**
```
cp: cannot stat 'app.tar.gz': No such file or directory
```

> Script stops immediately. No damage done.

---

## Pattern 1 — `set -e`

### What it does

Stops the script immediately when **any command fails** (returns non-zero exit code).

### Without `set -e`

```bash
#!/bin/bash

echo "Step 1"
ls /fake-folder          # fails — but script continues
echo "Step 2"            # runs anyway
echo "Step 3"            # runs anyway
echo "All done!"         # runs anyway — false success
```

**Output:**
```
Step 1
ls: cannot access '/fake-folder': No such file or directory
Step 2
Step 3
All done!
```

### With `set -e`

```bash
#!/bin/bash
set -e

echo "Step 1"
ls /fake-folder          # fails — script STOPS here
echo "Step 2"            # never runs
echo "Step 3"            # never runs
echo "All done!"         # never runs
```

**Output:**
```
Step 1
ls: cannot access '/fake-folder': No such file or directory
```

> Script exits immediately with a non-zero code.
> CI/CD pipeline sees the failure and marks the build red.

### When to allow a command to fail (use `|| true`)

Sometimes you expect a command might fail and that's okay.
Use `|| true` to tell `set -e` to ignore that specific failure:

```bash
#!/bin/bash
set -e

# This might fail if the directory doesn't exist — that's fine
rm -rf /tmp/old-release || true     # allowed to fail

# This must succeed
cp app.tar.gz /opt/myapp/           # must work — stops if not
```

---

## Pattern 2 — `set -u`

### What it does

Stops the script immediately when you use a **variable that was never set**.
This catches typos in variable names before they silently deploy wrong values.

### Without `set -u`

```bash
#!/bin/bash

ENVIRONMENT="prod"

# Typo — ENVIORNMENT instead of ENVIRONMENT
echo "Deploying to: $ENVIORNMENT"
```

**Output:**
```
Deploying to:
```

> Empty string. Silently deploys to nothing.
> Nobody notices the typo until something breaks in prod.

### With `set -u`

```bash
#!/bin/bash
set -u

ENVIRONMENT="prod"

# Typo — ENVIORNMENT instead of ENVIRONMENT
echo "Deploying to: $ENVIORNMENT"
```

**Output:**
```
bash: ENVIORNMENT: unbound variable
```

> Script stops. Typo caught before any damage is done.

### How to give a variable a default value safely

Sometimes a variable might not be set and that is okay —
you want a default value instead of an error:

```bash
#!/bin/bash
set -u

# Syntax: ${VARIABLE:-default_value}
LOG_LEVEL="${LOG_LEVEL:-INFO}"       # use INFO if LOG_LEVEL not set
RETRY_COUNT="${RETRY_COUNT:-3}"      # use 3 if RETRY_COUNT not set
OUTPUT_DIR="${OUTPUT_DIR:-/tmp}"     # use /tmp if OUTPUT_DIR not set

echo "Log level  : $LOG_LEVEL"
echo "Retry count: $RETRY_COUNT"
echo "Output dir : $OUTPUT_DIR"
```

**Run without setting any variables:**
```bash
./script.sh
```

**Output:**
```
Log level  : INFO
Retry count: 3
Output dir : /tmp
```

**Run with custom values:**
```bash
LOG_LEVEL=DEBUG RETRY_COUNT=5 ./script.sh
```

**Output:**
```
Log level  : DEBUG
Retry count: 5
Output dir : /tmp
```

> This is how real scripts are written — defaults that can be
> overridden from outside without changing the script.

---

## Pattern 3 — `set -o pipefail`

### What it does

Makes the script detect failures that happen **inside a pipe** (`|`).

### The problem with pipes

Without `pipefail`, if the first command in a pipe fails,
bash ignores it and only checks the exit code of the last command:

```bash
#!/bin/bash
set -e    # only set -e, no pipefail

# cat fails — file doesn't exist
# grep succeeds — it ran (found nothing)
# The pipe's exit code = grep's exit code = 0 (success!)
cat missing-file.log | grep "ERROR"
echo "Exit code: $?"    # prints 0 — wrong!
echo "This still runs"  # runs — should not
```

**Output:**
```
cat: missing-file.log: No such file or directory
Exit code: 0
This still runs
```

> `set -e` did not catch the failure because the pipe reported success.

### With `set -o pipefail`

```bash
#!/bin/bash
set -eo pipefail

cat missing-file.log | grep "ERROR"
echo "Exit code: $?"    # never runs
echo "This still runs"  # never runs
```

**Output:**
```
cat: missing-file.log: No such file or directory
```

> Now the pipe failure is caught correctly.

### Real DevOps example where pipefail matters

```bash
#!/bin/bash
set -eo pipefail

# Parse a log file and count errors
# If the log file is missing, cat fails
# Without pipefail — grep's 0 exit code hides the cat failure
# With pipefail    — the script stops at the cat failure

error_count=$(cat /var/log/myapp/app.log | grep -c "\[ERROR\]")
echo "Errors found: $error_count"
```

---

## Pattern 4 — The Holy Trinity Together

These three options are always written together on **line 2 of every script**:

```bash
#!/bin/bash
set -euo pipefail
```

Breaking it down:

| Option | Full form | What it does |
|---|---|---|
| `-e` | `set -e` | Stop on any error |
| `-u` | `set -u` | Stop on undefined variable |
| `-o pipefail` | `set -o pipefail` | Stop on pipe failures |

**Write this from memory. Every script. Always. No exceptions.**

```bash
#!/bin/bash
set -euo pipefail

# Your script starts here — now it is safe
```

> This is the single most important habit in shell scripting.
> Senior DevOps engineers can spot a beginner script immediately —
> it has no `set -euo pipefail` at the top.

---

## Pattern 5 — `trap` for Cleanup

### What it does

Runs a function **automatically** when the script exits — whether it
succeeds, fails, or is cancelled with Ctrl+C.

This is how you make sure temp files and lockfiles are always cleaned
up — even when the script crashes halfway through.

### Basic trap syntax

```bash
trap 'command_or_function' SIGNAL
```

Common signals:

| Signal | When it fires |
|---|---|
| `EXIT` | Any exit — success, failure, or Ctrl+C |
| `ERR` | When a command fails |
| `INT` | When user presses Ctrl+C |
| `TERM` | When the process is killed |

### Simple example — cleanup temp files

```bash
#!/bin/bash
set -euo pipefail

TEMP_FILE="/tmp/myapp-work-$$.txt"    # $$ = current process ID

# Register cleanup — runs no matter how the script exits
cleanup() {
  echo "Cleaning up..."
  rm -f "$TEMP_FILE"
  echo "Temp file removed"
}
trap cleanup EXIT

# Do some work
echo "Creating temp file..."
echo "some data" > "$TEMP_FILE"

echo "Processing..."
cat "$TEMP_FILE"

# Simulate a failure
ls /nonexistent-folder

echo "This never runs if above fails"
```

**Output:**
```
Creating temp file...
some data
Processing...
ls: cannot access '/nonexistent-folder': No such file or directory
Cleaning up...
Temp file removed
```

> Even though the script crashed on `ls`,
> the cleanup function still ran automatically.

### Practical lockfile example

A lockfile prevents two copies of the same script running at the same time:

```bash
#!/bin/bash
set -euo pipefail

LOCKFILE="/tmp/deploy.lock"

cleanup() {
  rm -f "$LOCKFILE"
  echo "Lockfile removed"
}
trap cleanup EXIT

# Check if already running
if [ -f "$LOCKFILE" ]; then
  echo "ERROR: Script is already running. Exiting."
  exit 1
fi

# Create lockfile
touch "$LOCKFILE"
echo "Lockfile created — script is now running"

# Your work here
echo "Doing important work..."
sleep 5
echo "Work complete"

# Lockfile is automatically removed by trap on exit
```

**Try running two copies at the same time:**

```bash
./script.sh &     # runs in background
./script.sh       # second copy — should be blocked
```

**Output:**
```
Lockfile created — script is now running
ERROR: Script is already running. Exiting.
```

---

## Putting It Together — Safe Script Template

This is the **starting template** for every script you write from now on.
Copy this, fill in your logic, and you have a safe production-ready script:

```bash
#!/bin/bash
# script-name.sh — what this script does
# Usage: ./script-name.sh

set -euo pipefail

# ── Configuration ─────────────────────────────────────
SCRIPT_NAME=$(basename "$0")
TEMP_DIR="/tmp/$SCRIPT_NAME-$$"
LOCKFILE="/tmp/$SCRIPT_NAME.lock"
LOG_LEVEL="${LOG_LEVEL:-INFO}"

# ── Logging ───────────────────────────────────────────
log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$1] $2"
}

# ── Cleanup ───────────────────────────────────────────
cleanup() {
  local exit_code=$?
  [ -d "$TEMP_DIR"  ] && rm -rf "$TEMP_DIR"
  [ -f "$LOCKFILE"  ] && rm -f  "$LOCKFILE"
  if [ $exit_code -ne 0 ]; then
    log "ERROR" "Script failed with exit code $exit_code"
  else
    log "INFO" "Script completed successfully"
  fi
}
trap cleanup EXIT

# ── Lockfile ──────────────────────────────────────────
if [ -f "$LOCKFILE" ]; then
  log "ERROR" "Already running — exiting"
  exit 1
fi
touch "$LOCKFILE"
mkdir -p "$TEMP_DIR"

# ── Your logic goes here ──────────────────────────────
log "INFO" "Script started"

# ... do your work ...

log "INFO" "Script finished"
```

**Test it:**

```bash
chmod +x script-name.sh
./script-name.sh
```

**Output:**
```
[2024-06-12 09:15:32] [INFO] Script started
[2024-06-12 09:15:32] [INFO] Script finished
[2024-06-12 09:15:32] [INFO] Script completed successfully
```

---

## Common Mistakes

### Mistake 1 — Forgetting `set -euo pipefail`

```bash
#!/bin/bash
# No set -euo pipefail
# This script will run blindly through failures
```

> Every real DevOps script starts with `set -euo pipefail`. Period.

---

### Mistake 2 — Registering trap after the work starts

```bash
#!/bin/bash
set -euo pipefail

LOCKFILE="/tmp/deploy.lock"
touch "$LOCKFILE"          # lockfile created

# Oops — trap registered AFTER the file is created
# If script crashes before this line, lockfile is never cleaned up
trap "rm -f $LOCKFILE" EXIT
```

```bash
#!/bin/bash
set -euo pipefail

LOCKFILE="/tmp/deploy.lock"

# Correct — register trap FIRST, then create the file
trap "rm -f $LOCKFILE" EXIT
touch "$LOCKFILE"
```

---

### Mistake 3 — Not handling expected failures with `|| true`

```bash
#!/bin/bash
set -euo pipefail

# This fails if directory doesn't exist — kills the script
rm -rf /tmp/old-build

# Correct — rm is allowed to fail
rm -rf /tmp/old-build || true
```

---

### Mistake 4 — Using `set -e` but not `pipefail`

```bash
#!/bin/bash
set -e    # only -e — pipe failures still slip through

# This pipe failure goes undetected
cat /missing-file | grep "pattern"
echo "This runs — should not"    # runs because grep exited 0
```

```bash
#!/bin/bash
set -euo pipefail    # all three — nothing slips through
```

---

## Quick Exercise

> **Task:** Write a script called `safe-backup.sh` using the safe
> script template that:
>
> 1. Uses `set -euo pipefail`
> 2. Creates a temp directory at `/tmp/backup-$$`
> 3. Registers a `trap` to clean up the temp dir on exit
> 4. Uses a lockfile at `/tmp/backup.lock`
> 5. Copies `/etc/passwd` into the temp directory
> 6. Counts the number of lines in the copy
> 7. Prints `"Backup done — N lines copied"`
> 8. Uses `LOG_LEVEL` with a default of `"INFO"`

**Expected output:**
```
[2024-06-12 09:15:32] [INFO] Starting backup
[2024-06-12 09:15:32] [INFO] Copying files...
[2024-06-12 09:15:32] [INFO] Backup done — 52 lines copied
[2024-06-12 09:15:32] [INFO] Script completed successfully
```

**Expected solution:**

```bash
#!/bin/bash
# safe-backup.sh — safe backup with cleanup and locking
set -euo pipefail

SCRIPT_NAME=$(basename "$0")
TEMP_DIR="/tmp/backup-$$"
LOCKFILE="/tmp/backup.lock"
LOG_LEVEL="${LOG_LEVEL:-INFO}"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$1] $2"
}

cleanup() {
  local exit_code=$?
  [ -d "$TEMP_DIR" ] && rm -rf "$TEMP_DIR"
  [ -f "$LOCKFILE" ] && rm -f  "$LOCKFILE"
  if [ $exit_code -ne 0 ]; then
    log "ERROR" "Script failed with exit code $exit_code"
  else
    log "INFO" "Script completed successfully"
  fi
}
trap cleanup EXIT

if [ -f "$LOCKFILE" ]; then
  log "ERROR" "Backup already running — exiting"
  exit 1
fi
touch "$LOCKFILE"
mkdir -p "$TEMP_DIR"

log "INFO" "Starting backup"

log "INFO" "Copying files..."
cp /etc/passwd "$TEMP_DIR/passwd.bak"

line_count=$(wc -l < "$TEMP_DIR/passwd.bak")
log "INFO" "Backup done — $line_count lines copied"
```

**Run and test failure:**
```bash
# Normal run
chmod +x safe-backup.sh
./safe-backup.sh

# Test lockfile — run two copies at once
./safe-backup.sh &
./safe-backup.sh

# Test set -e — break the copy step
cp /nonexistent "$TEMP_DIR/" 2>/dev/null || true
```

---

## Summary

| Pattern | What It Does | Always Use? |
|---|---|---|
| `set -e` | Stop on any command failure | Yes |
| `set -u` | Stop on undefined variable | Yes |
| `set -o pipefail` | Stop on pipe failures | Yes |
| `set -euo pipefail` | All three together | Yes — line 2, every script |
| `trap cleanup EXIT` | Cleanup always runs | Yes — before any work |
| Lockfile | Prevent duplicate runs | For long-running scripts |
| `\|\| true` | Allow specific failures | When failure is expected |
| `${VAR:-default}` | Safe default values | Whenever a var might not be set |

---

## The Two Lines Every Script Must Start With

```bash
#!/bin/bash
set -euo pipefail
```

> Memorise these. Write them before anything else.
> If your script does not have these two lines — it is not production ready.

---

## What's Next

**Intermediate Topic 2 — String Manipulation**  
Learn how to trim, extract, replace, and split strings entirely
inside bash — no grep, no awk, just built-in bash syntax.
These are the operations you use every day when working with
file paths, version numbers, config values, and log messages.