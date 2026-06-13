# Intermediate Topic 7 — Writing Reusable Scripts

> **Session:** Shell Scripting for DevOps — Intermediate Concepts  
> **Duration:** ~20 minutes  
> **Prerequisite:** Topics 1, 1+, 2, 3, 4, 5, 6 must be complete

---

## Table of Contents

- [What Makes a Script Reusable?](#what-makes-a-script-reusable)
- [Why DevOps Engineers Need This](#why-devops-engineers-need-this)
- [Part 1 — Positional Arguments](#part-1--positional-arguments)
- [Part 2 — Special Argument Variables](#part-2--special-argument-variables)
- [Part 3 — Validating Arguments](#part-3--validating-arguments)
- [Part 4 — Writing a Usage Message](#part-4--writing-a-usage-message)
- [Part 5 — Sourcing Shared Functions](#part-5--sourcing-shared-functions)
- [Part 6 — Making Scripts Portable](#part-6--making-scripts-portable)
- [Putting It All Together](#putting-it-all-together)
- [Common Mistakes](#common-mistakes)
- [Quick Exercise](#quick-exercise)

---

## What Makes a Script Reusable?

A reusable script is one that:

- **Accepts inputs** — works for different values without editing the code
- **Validates inputs** — tells you clearly when something is wrong
- **Has a help message** — anyone can run it without reading the code
- **Shares functions** — common code lives in one place, used everywhere
- **Works anywhere** — does not break depending on where it is run from

Compare:

```bash
# NOT reusable — hardcoded, only works for one case
#!/bin/bash
cp /opt/myapp/v1.4.2/app.tar.gz /backup/prod/

# REUSABLE — works for any app, version, environment
#!/bin/bash
./backup.sh -a myapp -v 1.4.2 -e prod
```

---

## Why DevOps Engineers Need This

A script you write once and never reuse is just a note.
A script you can call with different arguments from CI/CD pipelines,
cron jobs, and other scripts — that is a **tool**.

Real DevOps tools are always reusable:

| Tool | How It Is Called |
|---|---|
| `deploy.sh` | `./deploy.sh -e prod -v 1.4.2` |
| `backup.sh` | `./backup.sh -d /opt/myapp -t full` |
| `health-check.sh` | `./health-check.sh -h web01 -p 3000` |
| `log-rotate.sh` | `./log-rotate.sh -f app.log -s 100` |

---

## Part 1 — Positional Arguments

When someone runs your script with arguments,
bash stores them in special variables:

```bash
./script.sh hello world 42
#            $1    $2   $3
```

| Variable | Meaning |
|---|---|
| `$0` | The script name itself |
| `$1` | First argument |
| `$2` | Second argument |
| `$3` | Third argument |
| `$@` | All arguments as separate items |
| `$*` | All arguments as one string |
| `$#` | Number of arguments passed |

### Basic example

```bash
#!/bin/bash
set -euo pipefail

echo "Script name : $0"
echo "First arg   : $1"
echo "Second arg  : $2"
echo "All args    : $@"
echo "Arg count   : $#"
```

**Run it:**
```bash
./script.sh prod 1.4.2
```

**Output:**
```
Script name : ./script.sh
First arg   : prod
Second arg  : 1.4.2
All args    : prod 1.4.2
Arg count   : 2
```

### Use arguments as variables

```bash
#!/bin/bash
set -euo pipefail

ENV=$1
VERSION=$2

echo "Deploying version $VERSION to $ENV..."
```

**Run it:**
```bash
./deploy.sh prod 1.4.2
./deploy.sh staging 2.0.0
./deploy.sh dev 1.5.0-beta
```

> Same script — different behavior each time. That is reusability.

---

## Part 2 — Special Argument Variables

### `$#` — Count of arguments

```bash
#!/bin/bash
set -euo pipefail

echo "You passed $# argument(s)"

if [ $# -eq 0 ]; then
  echo "No arguments given"
elif [ $# -eq 1 ]; then
  echo "One argument: $1"
else
  echo "Multiple arguments: $@"
fi
```

### `$@` — All arguments, keep them separate

```bash
#!/bin/bash
set -euo pipefail

# Deploy to multiple servers passed as arguments
# ./deploy-all.sh web01 web02 web03

echo "Deploying to $# server(s)..."

for server in "$@"; do
  echo "  Deploying to: $server"
done
```

**Run it:**
```bash
./deploy-all.sh web01 web02 web03
```

**Output:**
```
Deploying to 3 server(s)...
  Deploying to: web01
  Deploying to: web02
  Deploying to: web03
```

### Default values for arguments

```bash
#!/bin/bash
set -euo pipefail

# If argument not given — use a default
ENV="${1:-dev}"           # default: dev
VERSION="${2:-latest}"    # default: latest
PORT="${3:-3000}"         # default: 3000

echo "Environment : $ENV"
echo "Version     : $VERSION"
echo "Port        : $PORT"
```

**Run with no arguments — uses defaults:**
```bash
./script.sh
```
```
Environment : dev
Version     : latest
Port        : 3000
```

**Run with arguments — overrides defaults:**
```bash
./script.sh prod 1.4.2 8080
```
```
Environment : prod
Version     : 1.4.2
Port        : 8080
```

### `shift` — consume arguments one by one

```bash
#!/bin/bash
set -euo pipefail

echo "All args: $@"

# shift removes $1 and shifts everything left
shift
echo "After shift: $@"

shift
echo "After second shift: $@"
```

**Run:**
```bash
./script.sh one two three
```
```
All args: one two three
After shift: two three
After second shift: three
```

> `shift` is useful when processing arguments in a loop.

---

## Part 3 — Validating Arguments

Never trust input — always validate before using:

### Check required arguments

```bash
#!/bin/bash
set -euo pipefail

ENV="${1:-}"
VERSION="${2:-}"

# Check both are provided
if [ -z "$ENV" ] || [ -z "$VERSION" ]; then
  echo "ERROR: Both environment and version are required"
  echo "Usage: $0 <environment> <version>"
  echo "Example: $0 prod 1.4.2"
  exit 1
fi

echo "Deploying $VERSION to $ENV..."
```

**Run with missing args:**
```bash
./deploy.sh prod
```
```
ERROR: Both environment and version are required
Usage: ./deploy.sh <environment> <version>
Example: ./deploy.sh prod 1.4.2
```

### Check argument values are valid

```bash
#!/bin/bash
set -euo pipefail

ENV="${1:-}"
VERSION="${2:-}"

# Required check
[ -z "$ENV"     ] && { echo "ERROR: environment required"; exit 1; }
[ -z "$VERSION" ] && { echo "ERROR: version required";     exit 1; }

# Value check — only allow known environments
if [[ ! "$ENV" =~ ^(dev|staging|prod)$ ]]; then
  echo "ERROR: Invalid environment '$ENV'"
  echo "       Allowed: dev | staging | prod"
  exit 1
fi

# Format check — version must be X.Y.Z
if [[ ! "$VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
  echo "ERROR: Invalid version '$VERSION'"
  echo "       Expected format: X.Y.Z (e.g. 1.4.2)"
  exit 1
fi

echo "All inputs valid ✓"
echo "Deploying $VERSION to $ENV..."
```

**Run with invalid environment:**
```bash
./deploy.sh production 1.4.2
```
```
ERROR: Invalid environment 'production'
       Allowed: dev | staging | prod
```

**Run with invalid version:**
```bash
./deploy.sh prod latest
```
```
ERROR: Invalid version 'latest'
       Expected format: X.Y.Z (e.g. 1.4.2)
```

---

## Part 4 — Writing a Usage Message

Every script someone else runs needs a help message.
A good usage message answers: *"How do I use this?"*

```bash
#!/bin/bash
set -euo pipefail

usage() {
  cat << EOF

  Usage: $(basename "$0") <environment> <version> [options]

  Arguments:
    environment    Target environment: dev | staging | prod
    version        App version: e.g. 1.4.2

  Options:
    -h, --help     Show this help message
    -d, --dry-run  Show what would happen without doing it

  Examples:
    $(basename "$0") prod 1.4.2
    $(basename "$0") staging 2.0.0-beta
    $(basename "$0") dev 1.5.0 --dry-run

EOF
  exit 0
}

# Show usage if -h or --help passed OR no arguments given
if [ $# -eq 0 ] || [ "${1:-}" = "-h" ] || [ "${1:-}" = "--help" ]; then
  usage
fi

ENV="$1"
VERSION="$2"

echo "Deploying $VERSION to $ENV..."
```

**Run with no args or -h:**
```bash
./deploy.sh
./deploy.sh -h
./deploy.sh --help
```

**Output:**
```

  Usage: deploy.sh <environment> <version> [options]

  Arguments:
    environment    Target environment: dev | staging | prod
    version        App version: e.g. 1.4.2

  Options:
    -h, --help     Show this help message
    -d, --dry-run  Show what would happen without doing it

  Examples:
    deploy.sh prod 1.4.2
    deploy.sh staging 2.0.0-beta
    deploy.sh dev 1.5.0 --dry-run

```

### Good usage message rules

| Rule | Why |
|---|---|
| Show the script name with `$(basename "$0")` | Works even if script is renamed |
| List all arguments | No guessing required |
| Show allowed values | e.g. `dev \| staging \| prod` |
| Give real examples | Most useful part — copy and paste |
| Exit with `0` | Help is not an error |

---

## Part 5 — Sourcing Shared Functions

When multiple scripts need the same functions —
`log()`, `notify()`, `check_disk()` — put them in one file
and **source** it from every script that needs it.

### Create a shared library file

```bash
# lib/common.sh — shared functions used by all scripts

# Logging function
log() {
  local level=$1; shift
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*"
}

# Check if a binary exists
require_binary() {
  local bin=$1
  command -v "$bin" > /dev/null 2>&1 || {
    log "ERROR" "Required binary not found: $bin"
    exit 1
  }
}

# Check if a file exists
require_file() {
  local file=$1
  [ -f "$file" ] || {
    log "ERROR" "Required file not found: $file"
    exit 1
  }
}

# Check disk space
check_disk() {
  local path=$1
  local min_mb=$2
  local free_mb
  free_mb=$(df "$path" | awk 'NR==2{print $4}')
  free_mb=$(( free_mb / 1024 ))
  if [ "$free_mb" -lt "$min_mb" ]; then
    log "ERROR" "Insufficient disk space on $path — ${free_mb}MB free, ${min_mb}MB needed"
    exit 1
  fi
  log "INFO" "Disk space OK — ${free_mb}MB free on $path"
}
```

### Source it in any script

```bash
#!/bin/bash
set -euo pipefail

# Find the directory where this script lives
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Source the shared library
source "$SCRIPT_DIR/lib/common.sh"

# Now all functions are available
log "INFO" "Starting deployment"
require_binary "aws"
require_binary "curl"
check_disk "/opt" 1024

log "INFO" "All checks passed"
```

### Project structure with shared library

```
myapp-ops/
├── deploy.sh              # sources lib/common.sh
├── backup.sh              # sources lib/common.sh
├── health-check.sh        # sources lib/common.sh
└── lib/
    └── common.sh          # all shared functions live here
```

> Fix a bug in `log()` once in `common.sh` —
> it is fixed in every script automatically.

### `source` vs executing a script

| | `source ./lib/common.sh` | `./lib/common.sh` |
|---|---|---|
| Runs in | Current shell | New subshell |
| Functions available after? | ✅ Yes | ❌ No |
| Variables available after? | ✅ Yes | ❌ No |
| Use for | Shared functions and variables | Running standalone scripts |

---

## Part 6 — Making Scripts Portable

A portable script works correctly no matter where it is called from.

### Problem — script breaks when called from a different directory

```bash
# broken — uses relative paths
#!/bin/bash
source lib/common.sh      # works only if run from /opt/myapp-ops/
source config/prod.env    # breaks if run from anywhere else
```

```bash
# ./deploy.sh              ← works (run from /opt/myapp-ops/)
# /opt/myapp-ops/deploy.sh ← breaks (run from /home/user/)
```

### Fix — always calculate the script's own directory

```bash
#!/bin/bash
set -euo pipefail

# Always find where THIS script is located
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Now all paths are relative to the script — not where you called it from
source "$SCRIPT_DIR/lib/common.sh"
source "$SCRIPT_DIR/config/prod.env"
CONFIG_FILE="$SCRIPT_DIR/config/app.conf"

log "INFO" "Script dir: $SCRIPT_DIR"
log "INFO" "Config    : $CONFIG_FILE"
```

**Now it works from anywhere:**
```bash
./deploy.sh                      # from the script directory
/opt/myapp-ops/deploy.sh         # from anywhere
cd /tmp && /opt/myapp-ops/deploy.sh   # still works
```

### Make the script executable

```bash
# Set execute permission
chmod +x deploy.sh
chmod +x backup.sh
chmod +x lib/common.sh

# Now run without bash prefix
./deploy.sh prod 1.4.2    # instead of: bash deploy.sh prod 1.4.2
```

### Add the shebang line correctly

```bash
#!/bin/bash          # always bash — not sh (sh has fewer features)
```

---

## Putting It All Together

A complete reusable script using every concept from this topic:

```bash
#!/bin/bash
# deploy.sh — Reusable deployment script
# Usage: ./deploy.sh <environment> <version>
# Run:   ./deploy.sh --help

set -euo pipefail

# ── Find script directory ──────────────────────────────
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# ── Source shared functions ────────────────────────────
source "$SCRIPT_DIR/lib/common.sh"

# ── Usage message ──────────────────────────────────────
usage() {
  cat << EOF

  Usage: $(basename "$0") <environment> <version>

  Arguments:
    environment    Target: dev | staging | prod
    version        Version: e.g. 1.4.2

  Options:
    -h, --help     Show this message
    -n, --dry-run  Simulate deploy without making changes

  Examples:
    $(basename "$0") prod 1.4.2
    $(basename "$0") staging 2.0.0
    $(basename "$0") prod 1.4.2 --dry-run

EOF
  exit 0
}

# ── Parse arguments ────────────────────────────────────
[ $# -eq 0 ] && usage
[ "${1:-}" = "-h" ] || [ "${1:-}" = "--help" ] && usage

ENV="${1:-}"
VERSION="${2:-}"
DRY_RUN=false
[ "${3:-}" = "--dry-run" ] || [ "${3:-}" = "-n" ] && DRY_RUN=true

# ── Validate ───────────────────────────────────────────
[ -z "$ENV"     ] && { log "ERROR" "Environment required"; usage; }
[ -z "$VERSION" ] && { log "ERROR" "Version required";     usage; }

[[ "$ENV" =~ ^(dev|staging|prod)$ ]] || {
  log "ERROR" "Invalid environment: $ENV. Allowed: dev | staging | prod"
  exit 1
}

[[ "$VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]] || {
  log "ERROR" "Invalid version: $VERSION. Expected: X.Y.Z"
  exit 1
}

# ── Main logic ─────────────────────────────────────────
log "INFO" "Deploy requested: v$VERSION → $ENV"

if $DRY_RUN; then
  log "INFO" "DRY RUN — no changes will be made"
  log "INFO" "Would deploy v$VERSION to $ENV"
  log "INFO" "Would restart service myapp"
  exit 0
fi

require_binary "aws"
require_binary "curl"
check_disk "/opt" 1024

log "INFO" "Deploying v$VERSION to $ENV..."
log "INFO" "Deploy complete ✓"
```

**Usage examples:**
```bash
# Show help
./deploy.sh --help

# Deploy to staging
./deploy.sh staging 1.4.2

# Deploy to prod
./deploy.sh prod 1.4.2

# Dry run — see what would happen
./deploy.sh prod 1.4.2 --dry-run

# Missing args — shows usage
./deploy.sh
```

---

## Common Mistakes

### Mistake 1 — Hardcoding instead of using arguments

```bash
# Wrong — not reusable
ENV="prod"
VERSION="1.4.2"

# Correct — accept from caller
ENV="${1:-}"
VERSION="${2:-}"
```

---

### Mistake 2 — No validation on arguments

```bash
# Wrong — crashes with confusing error later
ENV="$1"
DIR="/opt/myapp/$ENV"
mkdir "$DIR"    # if $1 is empty: mkdir /opt/myapp/

# Correct — validate first
ENV="${1:-}"
[ -z "$ENV" ] && { echo "ERROR: environment required"; exit 1; }
```

---

### Mistake 3 — Using relative paths in sourced files

```bash
# Wrong — breaks when called from a different directory
source lib/common.sh

# Correct — always use SCRIPT_DIR
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/lib/common.sh"
```

---

### Mistake 4 — Forgetting usage message

```bash
# Wrong — no help, users have to read the code
./deploy.sh          # silent failure or confusing error

# Correct — show help when no args given
[ $# -eq 0 ] && usage
```

---

### Mistake 5 — Using `$*` instead of `$@` for arguments

```bash
# Wrong — $* joins all args into one string, breaks with spaces
for arg in $*; do
  process "$arg"
done

# Correct — $@ keeps args separate
for arg in "$@"; do
  process "$arg"
done
```

---

## Quick Exercise

> **Task:** Write a reusable script called `backup.sh` that:
>
> 1. Accepts two arguments: `<source_directory>` and `<backup_name>`
> 2. Has a `usage()` function that shows when no args are given or `-h` is passed
> 3. Validates:
>    - Both arguments are provided
>    - Source directory exists
>    - Backup name contains only letters, numbers, and dashes
> 4. Sources a shared file `lib/common.sh` that contains the `log()` function
> 5. Uses `$SCRIPT_DIR` to find `lib/common.sh` regardless of where it is called from
> 6. Creates a backup file named `<backup_name>-<date>.tar.gz` in `/tmp/`
> 7. Prints a success message with the backup file path

**Expected usage:**
```bash
./backup.sh                           # shows usage
./backup.sh -h                        # shows usage
./backup.sh /opt/myapp myapp-backup   # runs backup
./backup.sh /nonexistent name         # ERROR: directory not found
./backup.sh /opt/myapp "my backup"    # ERROR: invalid backup name
```

**Expected output (success):**
```
[2024-06-12 09:15:32] [INFO] Starting backup
[2024-06-12 09:15:32] [INFO] Source : /opt/myapp
[2024-06-12 09:15:32] [INFO] Output : /tmp/myapp-backup-20240612.tar.gz
[2024-06-12 09:15:32] [INFO] Backup complete ✓
```

**Expected solution:**

```bash
#!/bin/bash
# backup.sh — Reusable backup script
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/lib/common.sh"

usage() {
  cat << EOF

  Usage: $(basename "$0") <source_directory> <backup_name>

  Arguments:
    source_directory   Directory to back up (must exist)
    backup_name        Name for the backup (letters, numbers, dashes only)

  Options:
    -h, --help         Show this message

  Examples:
    $(basename "$0") /opt/myapp myapp-backup
    $(basename "$0") /etc/nginx nginx-config

EOF
  exit 0
}

# Show usage if no args or help flag
[ $# -eq 0 ]                          && usage
[ "${1:-}" = "-h" ]                   && usage
[ "${1:-}" = "--help" ]               && usage

SOURCE_DIR="${1:-}"
BACKUP_NAME="${2:-}"

# Validate
[ -z "$SOURCE_DIR"  ] && { log "ERROR" "Source directory required"; usage; }
[ -z "$BACKUP_NAME" ] && { log "ERROR" "Backup name required";      usage; }

[ -d "$SOURCE_DIR" ] || {
  log "ERROR" "Source directory not found: $SOURCE_DIR"
  exit 1
}

[[ "$BACKUP_NAME" =~ ^[a-zA-Z0-9-]+$ ]] || {
  log "ERROR" "Invalid backup name: $BACKUP_NAME"
  log "ERROR" "Use only letters, numbers, and dashes"
  exit 1
}

# Build output path
DATE=$(date '+%Y%m%d')
OUTPUT="/tmp/${BACKUP_NAME}-${DATE}.tar.gz"

# Run backup
log "INFO" "Starting backup"
log "INFO" "Source : $SOURCE_DIR"
log "INFO" "Output : $OUTPUT"

tar -czf "$OUTPUT" -C "$(dirname "$SOURCE_DIR")" \
  "$(basename "$SOURCE_DIR")"

log "INFO" "Backup complete ✓"
log "INFO" "File: $OUTPUT ($(du -sh "$OUTPUT" | cut -f1))"
```

---

## Summary — Quick Reference Card

| Concept | Syntax | Example |
|---|---|---|
| First argument | `$1` | `ENV="$1"` |
| Second argument | `$2` | `VERSION="$2"` |
| All arguments | `"$@"` | `for arg in "$@"` |
| Argument count | `$#` | `[ $# -eq 0 ]` |
| Script name | `$0` or `$(basename "$0")` | In usage message |
| Default value | `"${1:-default}"` | `ENV="${1:-dev}"` |
| Shift args | `shift` | Move `$2` to `$1` |
| Show usage | `[ $# -eq 0 ] && usage` | No args → help |
| Source library | `source "$SCRIPT_DIR/lib/common.sh"` | Load shared functions |
| Script's own dir | `$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)` | Portable path |
| Make executable | `chmod +x script.sh` | Run as `./script.sh` |

---

## What's Next

**Intermediate Topic 8 — Real-World Mini Project**  
Everything from Topics 1–7 comes together in one complete script.
You will build a server monitoring tool from scratch that accepts
arguments, reads a config file, checks multiple services,
logs structured output, and sends a summary report.
This is the capstone of the intermediate series.