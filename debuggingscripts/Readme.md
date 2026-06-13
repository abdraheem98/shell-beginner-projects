# Intermediate Topic 6 — Debugging Scripts

> **Session:** Shell Scripting for DevOps — Intermediate Concepts  
> **Duration:** ~20 minutes  
> **Prerequisite:** Topics 1, 1+, 2, 3, 4, 5 must be complete

---

## Table of Contents

- [What Is Debugging?](#what-is-debugging)
- [Why DevOps Engineers Need This](#why-devops-engineers-need-this)
- [Tool 1 — bash -n Syntax Check](#tool-1--bash--n-syntax-check)
- [Tool 2 — set -x Trace Mode](#tool-2--set--x-trace-mode)
- [Tool 3 — echo Debugging](#tool-3--echo-debugging)
- [Tool 4 — trap ERR for Error Location](#tool-4--trap-err-for-error-location)
- [Tool 5 — set -v Verbose Mode](#tool-5--set--v-verbose-mode)
- [Debugging Workflow — Step by Step](#debugging-workflow--step-by-step)
- [Common Bug Patterns](#common-bug-patterns)
- [Real DevOps Examples](#real-devops-examples)
- [Common Mistakes](#common-mistakes)
- [Quick Exercise](#quick-exercise)

---

## What Is Debugging?

Debugging means **finding and fixing the cause of a bug** in your script.

A bug is anything that makes your script behave differently
from what you expected:

- Script crashes with a confusing error
- Script runs but produces wrong output
- Script appears to work but does nothing
- Script hangs and never finishes

Debugging is a skill — the better you are at it,
the faster you fix problems at 2AM when something is broken in prod.

---

## Why DevOps Engineers Need This

Shell scripts fail in unexpected ways. The error messages are often
unhelpful. Without debugging tools you are guessing in the dark.

**Common real scenarios:**

| Problem | Without Debugging | With Debugging |
|---|---|---|
| Script fails at step 7 of 10 | You don't know which step | `set -x` shows exactly which line |
| Variable has wrong value | You can't see the value | `echo` shows it before it's used |
| Syntax error in 200-line script | Script crashes, no line number | `bash -n` shows the exact line |
| Script exits but you don't know why | Silent failure | `trap ERR` catches and reports it |

---

## Tool 1 — `bash -n` Syntax Check

**Before running a script — always check syntax first.**

`bash -n` reads the script and checks for syntax errors
**without actually running it**.

```bash
bash -n script.sh
```

### Example — catch a syntax error

```bash
# broken.sh — has a syntax error
#!/bin/bash

if [ "$ENV" = "prod" ]   # missing 'then'
  echo "production"
fi
```

**Run syntax check:**
```bash
bash -n broken.sh
```

**Output:**
```
broken.sh: line 5: syntax error near unexpected token 'fi'
broken.sh: line 5: 'fi'
```

> Tells you the **line number** and **what is wrong**
> before the script runs and breaks something.

### Always run `bash -n` before running a new script

```bash
# Your workflow for every new script:
# Step 1 — write the script
vim deploy.sh

# Step 2 — check syntax
bash -n deploy.sh

# Step 3 — only run if syntax is clean
./deploy.sh
```

### What `bash -n` catches

| It catches | It does NOT catch |
|---|---|
| Missing `then`, `do`, `fi`, `done` | Wrong variable names |
| Unclosed quotes | Logic errors |
| Invalid syntax | Runtime errors |
| Malformed `if/for/while` | Wrong command usage |

---

## Tool 2 — `set -x` Trace Mode

**The most powerful debugging tool in bash.**

`set -x` prints every command **before it runs**,
with all variables already substituted with their values.

```bash
#!/bin/bash
set -x    # turn on trace mode

NAME="myapp"
VERSION="1.4.2"
echo "Deploying $NAME version $VERSION"
```

**Output:**
```
+ NAME=myapp
+ VERSION=1.4.2
+ echo 'Deploying myapp version 1.4.2'
Deploying myapp version 1.4.2
```

The `+` at the start of each line means "this is a traced command".

### The real power — you see variable values

```bash
#!/bin/bash
set -x

ENV=""          # accidentally empty
DIR="/opt/$ENV/releases"
mkdir -p "$DIR"
```

**Output:**
```
+ ENV=
+ DIR=/opt//releases
+ mkdir -p /opt//releases
```

> Without `set -x` you would have no idea why the path looks wrong.
> With `set -x` you immediately see `DIR=/opt//releases` — empty ENV.

### Turn trace on and off around a specific section

You do not have to trace the whole script.
Turn it on only around the suspicious section:

```bash
#!/bin/bash
set -euo pipefail

echo "Step 1 — this works fine"

# Turn on tracing for just this section
set -x
RESULT=$(some_command | awk '{print $2}')
echo "Result: $RESULT"
set +x    # turn trace OFF
# ─────────────────────────────────────────

echo "Step 3 — back to normal output"
```

**Output:**
```
Step 1 — this works fine
+ some_command
+ awk '{print $2}'
+ RESULT=somevalue
+ echo 'Result: somevalue'
Result: somevalue
Step 3 — back to normal output
```

### Run a script with trace from outside

You do not need to add `set -x` inside the script.
Run it from the terminal with `-x`:

```bash
bash -x deploy.sh
bash -x deploy.sh -e prod -v 1.4.2
```

> Use this when you do not want to modify the script.

### Trace output to a file (useful for long scripts)

```bash
# Send trace output to a file while still seeing normal output
bash -x deploy.sh 2> trace.log

# View the trace
cat trace.log
```

---

## Tool 3 — `echo` Debugging

The simplest debugging technique — add `echo` statements
to print variable values at key points.

```bash
#!/bin/bash
set -euo pipefail

ENV="prod"
VERSION="1.4.2"

# Debug — print values before using them
echo "DEBUG: ENV=$ENV"
echo "DEBUG: VERSION=$VERSION"

ARTIFACT_PATH="s3://artifacts/$ENV/$VERSION/app.tar.gz"

# Debug — print the final path
echo "DEBUG: ARTIFACT_PATH=$ARTIFACT_PATH"

aws s3 cp "$ARTIFACT_PATH" /tmp/
```

**Output:**
```
DEBUG: ENV=prod
DEBUG: VERSION=1.4.2
DEBUG: ARTIFACT_PATH=s3://artifacts/prod/1.4.2/app.tar.gz
```

### Make debug output conditional — `DEBUG` flag

```bash
#!/bin/bash
set -euo pipefail

# Set DEBUG=1 from outside to enable debug output
# DEBUG=1 ./script.sh
DEBUG="${DEBUG:-0}"

debug() {
  [ "$DEBUG" = "1" ] && echo "DEBUG: $*" >&2
}

ENV="prod"
VERSION="1.4.2"

debug "ENV=$ENV"
debug "VERSION=$VERSION"

ARTIFACT_PATH="s3://artifacts/$ENV/$VERSION/app.tar.gz"
debug "ARTIFACT_PATH=$ARTIFACT_PATH"

echo "Deploying..."
```

**Normal run — no debug output:**
```bash
./script.sh
```
```
Deploying...
```

**Debug run — see everything:**
```bash
DEBUG=1 ./script.sh
```
```
DEBUG: ENV=prod
DEBUG: VERSION=1.4.2
DEBUG: ARTIFACT_PATH=s3://artifacts/prod/1.4.2/app.tar.gz
Deploying...
```

> This is how production scripts handle debug output.
> Turn it on without changing the script.

---

## Tool 4 — `trap ERR` for Error Location

When a script crashes, bash does not always tell you which line failed.
`trap ERR` catches the error and tells you exactly where it happened:

```bash
#!/bin/bash
set -euo pipefail

# Register error handler
handle_error() {
  local exit_code=$?
  local line_number=$1
  echo ""
  echo "════ ERROR CAUGHT ════════════════════"
  echo "  Line number : $line_number"
  echo "  Exit code   : $exit_code"
  echo "  Command     : $BASH_COMMAND"
  echo "  Script      : $0"
  echo "══════════════════════════════════════"
}
trap 'handle_error $LINENO' ERR

echo "Step 1: Starting..."
echo "Step 2: Checking files..."
cp missing-file.txt /tmp/         # this fails
echo "Step 3: This never runs"
```

**Output:**
```
Step 1: Starting...
Step 2: Checking files...
cp: cannot stat 'missing-file.txt': No such file or directory

════ ERROR CAUGHT ════════════════════
  Line number : 16
  Exit code   : 1
  Command     : cp missing-file.txt /tmp/
  Script      : ./script.sh
══════════════════════════════════════
```

> You now know **exactly** which line failed, which command caused it,
> and what exit code it returned — no guessing needed.

### Add call stack for deeper debugging

```bash
#!/bin/bash
set -euo pipefail

handle_error() {
  local exit_code=$?
  local line_number=$1

  echo ""
  echo "════ ERROR ══════════════════════════"
  echo "  Line    : $line_number"
  echo "  Code    : $exit_code"
  echo "  Command : $BASH_COMMAND"
  echo ""
  echo "  Call stack:"
  local i=0
  while caller $i; do
    i=$(( i + 1 ))
  done
  echo "══════════════════════════════════════"
}
trap 'handle_error $LINENO' ERR

run_deploy() {
  echo "Running deploy..."
  aws s3 cp s3://missing-bucket/app.tar.gz /tmp/   # fails here
}

echo "Starting..."
run_deploy
```

**Output:**
```
Starting...
Running deploy...
aws: error: ...

════ ERROR ══════════════════════════
  Line    : 21
  Code    : 1
  Command : aws s3 cp s3://missing-bucket/app.tar.gz /tmp/

  Call stack:
  21 run_deploy script.sh
  24 main script.sh
══════════════════════════════════════
```

---

## Tool 5 — `set -v` Verbose Mode

`set -v` prints each line of the script **as it is read**,
before variable substitution:

```bash
#!/bin/bash
set -v

NAME="myapp"
echo "Hello $NAME"
```

**Output:**
```
NAME="myapp"
NAME=myapp
echo "Hello $NAME"
Hello myapp
```

### `set -v` vs `set -x`

| | `set -v` | `set -x` |
|---|---|---|
| Shows | Raw line as written | Line with variables substituted |
| Useful for | Seeing what is in the script | Seeing actual values at runtime |
| Use when | Script has syntax confusion | Script has wrong variable values |

### Combine both for maximum detail

```bash
#!/bin/bash
set -xv
```

> Very noisy — only use this for short scripts or small sections.

---

## Debugging Workflow — Step by Step

When a script is broken, follow this order:

```
Step 1 — bash -n script.sh
         Check for syntax errors first
         Fix any before continuing
              │
              ▼
Step 2 — bash -x script.sh
         Run with trace — watch which line fails
         Look for wrong variable values
              │
              ▼
Step 3 — Add echo/debug statements
         Print variable values just before the failing line
         Confirm what the variable actually contains
              │
              ▼
Step 4 — Add trap ERR
         Catch the exact line, command, and exit code
         Add to the script permanently for future debugging
              │
              ▼
Step 5 — Isolate the problem
         Copy the failing section into a tiny test script
         Run it alone with set -x
         Fix it, then put it back
```

---

## Common Bug Patterns

These are the bugs freshers hit most often — and how to spot them with debugging tools:

### Bug 1 — Empty variable

```bash
#!/bin/bash
set -x

# Bug: variable is empty because of a typo
ENVIORNMENT="prod"   # typo — should be ENVIRONMENT

DIR="/opt/$ENVIRONMENT/app"   # uses correctly spelled name — gets empty!
mkdir -p "$DIR"
```

**Trace output reveals it:**
```
+ ENVIORNMENT=prod
+ DIR=/opt//app         ← empty! see the double slash
+ mkdir -p /opt//app
```

**Fix:** correct the typo to `ENVIRONMENT="prod"`

---

### Bug 2 — Wrong quotes

```bash
#!/bin/bash
set -x

MESSAGE='Hello $USER'    # single quotes — $USER not expanded
echo "$MESSAGE"
```

**Trace output:**
```
+ MESSAGE='Hello $USER'
+ echo 'Hello $USER'
Hello $USER              ← variable not expanded
```

**Fix:** use double quotes — `MESSAGE="Hello $USER"`

---

### Bug 3 — Command not found

```bash
#!/bin/bash
set -euo pipefail

# Bug: jq is not installed
RESPONSE=$(curl -sf https://api.example.com/status)
VERSION=$(echo "$RESPONSE" | jq -r '.version')
echo "Version: $VERSION"
```

**Error:**
```
./script.sh: line 6: jq: command not found
```

**Fix:** check for jq before using it:
```bash
command -v jq > /dev/null 2>&1 || { echo "ERROR: jq not installed"; exit 1; }
```

---

### Bug 4 — Subshell variable loss

```bash
#!/bin/bash
set -euo pipefail

COUNT=0
cat servers.txt | while IFS= read -r line; do
  COUNT=$(( COUNT + 1 ))
done
echo "Count: $COUNT"    # always prints 0
```

**Debug with echo inside the loop:**
```bash
cat servers.txt | while IFS= read -r line; do
  COUNT=$(( COUNT + 1 ))
  echo "DEBUG: inside loop, COUNT=$COUNT"   # shows COUNT increasing
done
echo "DEBUG: after loop, COUNT=$COUNT"      # shows COUNT is 0 — subshell!
```

**Fix:** use redirect instead of pipe:
```bash
while IFS= read -r line; do
  COUNT=$(( COUNT + 1 ))
done < servers.txt
echo "Count: $COUNT"    # correct now
```

---

### Bug 5 — Silent command failure

```bash
#!/bin/bash
# No set -euo pipefail — dangerous

aws s3 cp s3://wrong-bucket/app.tar.gz /tmp/   # fails silently
tar -xzf /tmp/app.tar.gz                        # nothing to extract
echo "Done!"                                    # prints anyway
```

**Fix:**
```bash
#!/bin/bash
set -euo pipefail   # script stops at aws failure now
trap 'echo "Failed at line $LINENO: $BASH_COMMAND"' ERR

aws s3 cp s3://wrong-bucket/app.tar.gz /tmp/
```

---

## Real DevOps Examples

### Debug a broken deploy script

```bash
#!/bin/bash
set -euo pipefail

# Add this error handler at the top of any script you are debugging
handle_error() {
  echo ""
  echo "FAILED at line $1"
  echo "Command : $BASH_COMMAND"
  echo "Exit    : $?"
  echo ""
  echo "Variable dump:"
  echo "  ENV     = ${ENV:-unset}"
  echo "  VERSION = ${VERSION:-unset}"
  echo "  APP_DIR = ${APP_DIR:-unset}"
}
trap 'handle_error $LINENO' ERR

ENV="${1:-}"
VERSION="${2:-}"

echo "Starting deploy: env=$ENV version=$VERSION"

APP_DIR="/opt/myapp/$ENV"
ARTIFACT="s3://artifacts/$ENV/$VERSION/app.tar.gz"

echo "Downloading artifact..."
aws s3 cp "$ARTIFACT" /tmp/app.tar.gz

echo "Extracting..."
mkdir -p "$APP_DIR"
tar -xzf /tmp/app.tar.gz -C "$APP_DIR"

echo "Deploy complete"
```

**Run with wrong args to see the error handler:**
```bash
bash -x ./deploy.sh prod
```

---

### Add a DEBUG mode to any script

```bash
#!/bin/bash
set -euo pipefail

# Activate with: DEBUG=1 ./script.sh
DEBUG="${DEBUG:-0}"
[ "$DEBUG" = "1" ] && set -x    # turn on trace if DEBUG=1

debug() {
  [ "$DEBUG" = "1" ] && echo "[DEBUG] $*" >&2
}

APP_NAME="myapp"
VERSION="1.4.2"

debug "Starting with APP_NAME=$APP_NAME VERSION=$VERSION"

ARTIFACT="s3://artifacts/$APP_NAME/$VERSION/app.tar.gz"
debug "Artifact path: $ARTIFACT"

echo "Deploying $APP_NAME version $VERSION..."
```

**Normal run:**
```bash
./script.sh
```
```
Deploying myapp version 1.4.2...
```

**Debug run:**
```bash
DEBUG=1 ./script.sh
```
```
+ APP_NAME=myapp
+ VERSION=1.4.2
[DEBUG] Starting with APP_NAME=myapp VERSION=1.4.2
+ ARTIFACT=s3://artifacts/myapp/1.4.2/app.tar.gz
[DEBUG] Artifact path: s3://artifacts/myapp/1.4.2/app.tar.gz
+ echo 'Deploying myapp version 1.4.2...'
Deploying myapp version 1.4.2...
```

---

## Common Mistakes

### Mistake 1 — Running script without syntax check first

```bash
# Wrong — run first, discover syntax error in production
./deploy.sh

# Correct — always check syntax first
bash -n deploy.sh && ./deploy.sh
```

---

### Mistake 2 — Leaving `set -x` in production scripts

```bash
#!/bin/bash
set -x    # left here accidentally — floods logs in production

echo "sensitive-password=$DB_PASS"   # credentials printed in trace!
```

```bash
# Correct — only trace in debug mode
[ "${DEBUG:-0}" = "1" ] && set -x
```

---

### Mistake 3 — Forgetting `trap ERR` in long scripts

```bash
#!/bin/bash
set -euo pipefail
# No trap ERR

# 150 lines of script...
# Line 83 fails
# All you see is:
# some-command: not found
# (no line number, no context, nothing)
```

```bash
#!/bin/bash
set -euo pipefail
trap 'echo "Error at line $LINENO: $BASH_COMMAND"' ERR
# Now every failure tells you exactly where it happened
```

---

### Mistake 4 — Not checking syntax after editing

```bash
# You edit the script to fix one bug
# Accidentally introduce a syntax error
# Commit and deploy
# Production script crashes on startup

# Prevention — always run bash -n after editing
bash -n deploy.sh
echo "Syntax OK"
```

---

## Quick Exercise

> **Task:** The following script has **5 bugs**. Use debugging tools
> to find and fix all of them.
>
> ```bash
> #!/bin/bash
>
> ENVIRNOMENT='prod'
> APP="myapp
> VERSION=1.4.2
>
> DEPLOY_DIR=/opt/$APP/$ENVIRNOMENT
>
> echo 'Deploying $APP version $VERSION to $ENVIRNOMENT'
>
> count=0
> cat servers.txt | while read server; do
>   count=$((count + 1))
> done
> echo "Total servers: $count"
> ```
>
> **Hint:** Use `bash -n` first, then `set -x`, then read carefully.

**The 5 bugs:**

| # | Bug | Fix |
|---|---|---|
| 1 | Typo in variable name `ENVIRNOMENT` | Change to `ENVIRONMENT` — and use it consistently |
| 2 | Missing closing quote `APP="myapp` | Change to `APP="myapp"` |
| 3 | Missing quotes on `DEPLOY_DIR` assignment | Change to `DEPLOY_DIR="/opt/$APP/$ENVIRONMENT"` |
| 4 | Single quotes in echo — variables not expanded | Change to double quotes |
| 5 | Pipe subshell — `count` always 0 | Replace `cat file \| while` with `while ... done < file` |

**Fixed script:**

```bash
#!/bin/bash
set -euo pipefail
trap 'echo "Error at line $LINENO: $BASH_COMMAND"' ERR

ENVIRONMENT="prod"
APP="myapp"
VERSION="1.4.2"

DEPLOY_DIR="/opt/$APP/$ENVIRONMENT"

echo "Deploying $APP version $VERSION to $ENVIRONMENT"

count=0
while IFS= read -r server; do
  count=$(( count + 1 ))
done < servers.txt
echo "Total servers: $count"
```

---

## Summary — Debugging Toolkit

| Tool | Command | Use When |
|---|---|---|
| Syntax check | `bash -n script.sh` | Before running any new/edited script |
| Trace mode | `bash -x script.sh` | Script fails and you don't know where |
| Partial trace | `set -x` ... `set +x` | Suspicious section only |
| Echo debug | `echo "DEBUG: VAR=$VAR"` | Need to see variable value |
| Conditional debug | `DEBUG=1 ./script.sh` | Need trace without editing script |
| Error location | `trap 'handle_error $LINENO' ERR` | Know exactly which line failed |
| Verbose mode | `set -v` | See raw lines as they are read |

---

## The Debugging Mantra

```
See a bug?

  1. bash -n    → Is the syntax correct?
  2. bash -x    → What is actually running?
  3. echo       → What do the variables contain?
  4. trap ERR   → Which exact line is failing?
  5. Isolate    → Copy the broken part, test it alone
```

> 90% of shell script bugs are one of:
> - Wrong variable value
> - Missing quotes
> - Subshell variable loss
> - Command not found
> - Syntax error
>
> These five tools catch all of them.

---

## What's Next

**Intermediate Topic 7 — Writing Reusable Scripts**  
Learn how to write scripts that others can use —
proper argument handling with `$1 $2 $@`,
sourcing shared functions across multiple scripts,
writing a usage/help message, and making scripts
portable and safe to run anywhere.