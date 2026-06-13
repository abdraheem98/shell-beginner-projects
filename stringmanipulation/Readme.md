# Intermediate Topic 2 — String Manipulation

> **Session:** Shell Scripting for DevOps — Intermediate Concepts  
> **Duration:** ~20 minutes  
> **Prerequisite:** Topic 1 (Exit Codes) + Topic 1+ (Script Safety Patterns)

---

## Table of Contents

- [What Is String Manipulation?](#what-is-string-manipulation)
- [Why DevOps Engineers Need This](#why-devops-engineers-need-this)
- [String Basics — Length & Concatenation](#string-basics--length--concatenation)
- [Extracting Parts of a String](#extracting-parts-of-a-string)
- [Removing Parts of a String](#removing-parts-of-a-string)
- [Replacing Parts of a String](#replacing-parts-of-a-string)
- [Changing Case](#changing-case)
- [Checking String Content](#checking-string-content)
- [Splitting a String](#splitting-a-string)
- [Real DevOps Examples](#real-devops-examples)
- [Common Mistakes](#common-mistakes)
- [Quick Exercise](#quick-exercise)

---

## What Is String Manipulation?

String manipulation means **working with text inside bash** — cutting
it, replacing parts of it, checking what it contains, and transforming it.

The good news: **bash can do all of this without any extra tools.**
No `grep`, no `awk`, no `sed` needed for basic string operations.
It is all built in using a special `${}` syntax.

---

## Why DevOps Engineers Need This

String manipulation shows up constantly in real scripts:

| Situation | What You Need |
|---|---|
| Extract version number from `app-v1.4.2.tar.gz` | Remove prefix and suffix |
| Get filename without extension | Remove extension |
| Get directory from a full path | Extract up to last `/` |
| Convert `PROD` to `prod` | Lowercase |
| Check if a string contains `error` | Pattern match |
| Split `host:port` into two variables | Split on `:` |
| Build a log filename from date | Concatenation |

All of these happen inside real scripts every week.

---

## String Basics — Length & Concatenation

### String length

```bash
#!/bin/bash
set -euo pipefail

NAME="myapp"
echo "Length: ${#NAME}"    # 5
```

```bash
VERSION="1.4.2"
echo "Length: ${#VERSION}"    # 5
```

> `${#VARIABLE}` gives you the number of characters in the string.

### Concatenation — joining strings together

```bash
#!/bin/bash
set -euo pipefail

APP="myapp"
VERSION="1.4.2"
ENV="prod"

# Just put variables next to each other
FILENAME="${APP}-${VERSION}.tar.gz"
echo "$FILENAME"    # myapp-1.4.2.tar.gz

# Build a log filename with date
DATE=$(date '+%Y%m%d')
LOG_FILE="${APP}-${DATE}.log"
echo "$LOG_FILE"    # myapp-20240612.log

# Build a full path
BASE_DIR="/opt/apps"
APP_DIR="${BASE_DIR}/${APP}/${ENV}"
echo "$APP_DIR"    # /opt/apps/myapp/prod
```

---

## Extracting Parts of a String

### Substring — extract by position

```bash
#!/bin/bash
set -euo pipefail

# Syntax: ${VARIABLE:start:length}
#          start = position to start from (0-based)
#          length = how many characters to take

VERSION="1.4.2"
echo "${VERSION:0:1}"    # 1   (start at 0, take 1 char)
echo "${VERSION:2:3}"    # 4.2 (start at 2, take 3 chars)
echo "${VERSION:0:3}"    # 1.4 (start at 0, take 3 chars)

# Without length — takes everything from start position
FILENAME="deploy-20240612.log"
echo "${FILENAME:7}"     # 20240612.log (everything from position 7)
```

---

## Removing Parts of a String

This is the most useful section. Learn these four patterns well.

### Remove from the front — `#` and `##`

```bash
#!/bin/bash
set -euo pipefail

FILEPATH="/opt/myapp/releases/1.4.2/app.js"

# ${VAR#pattern}  — remove SHORTEST match from the FRONT
echo "${FILEPATH#/}"          # opt/myapp/releases/1.4.2/app.js

# ${VAR##pattern} — remove LONGEST match from the FRONT
echo "${FILEPATH##*/}"        # app.js  (removes everything up to last /)
```

**DevOps use — get just the filename:**
```bash
FULL_PATH="/opt/myapp/releases/1.4.2/app.js"
FILENAME="${FULL_PATH##*/}"
echo "$FILENAME"    # app.js
```

**DevOps use — strip a prefix from version:**
```bash
TAG="v1.4.2"
VERSION="${TAG#v}"     # remove the 'v' from the front
echo "$VERSION"        # 1.4.2
```

---

### Remove from the end — `%` and `%%`

```bash
#!/bin/bash
set -euo pipefail

FILENAME="myapp-1.4.2.tar.gz"

# ${VAR%pattern}  — remove SHORTEST match from the END
echo "${FILENAME%.gz}"       # myapp-1.4.2.tar  (removes .gz)

# ${VAR%%pattern} — remove LONGEST match from the END
echo "${FILENAME%%.*}"       # myapp-1           (removes everything after first .)
echo "${FILENAME%%.tar.gz}"  # myapp-1.4.2       (removes .tar.gz)
```

**DevOps use — get directory from full path:**
```bash
FULL_PATH="/opt/myapp/releases/1.4.2/app.js"
DIRECTORY="${FULL_PATH%/*}"
echo "$DIRECTORY"    # /opt/myapp/releases/1.4.2
```

**DevOps use — get filename without extension:**
```bash
FILENAME="backup-20240612.tar.gz"
NAME_ONLY="${FILENAME%.tar.gz}"
echo "$NAME_ONLY"    # backup-20240612
```

---

### Memory trick for `#` and `%`

```
# = front of string  (hash is on the LEFT of your keyboard)
% = end of string    (percent is on the RIGHT of your keyboard)

One symbol  = shortest match
Two symbols = longest match
```

---

## Replacing Parts of a String

```bash
#!/bin/bash
set -euo pipefail

# Syntax: ${VAR/find/replace}    — replace FIRST match
#         ${VAR//find/replace}   — replace ALL matches

MESSAGE="Hello World World"

# Replace first match only
echo "${MESSAGE/World/DevOps}"    # Hello DevOps World

# Replace all matches
echo "${MESSAGE//World/DevOps}"   # Hello DevOps DevOps

# Replace with nothing — effectively deletes the pattern
VERSION="v1.4.2"
echo "${VERSION/v/}"              # 1.4.2  (same as removing prefix)
```

**DevOps use — fix a path:**
```bash
OLD_PATH="/opt/myapp/staging/app.js"
NEW_PATH="${OLD_PATH/staging/prod}"
echo "$NEW_PATH"    # /opt/myapp/prod/app.js
```

**DevOps use — replace dashes with underscores in a name:**
```bash
APP_NAME="my-app-name"
ENV_VAR="${APP_NAME//-/_}"    # replace ALL dashes
echo "$ENV_VAR"               # my_app_name
```

---

## Changing Case

```bash
#!/bin/bash
set -euo pipefail

ENV="PRODUCTION"
app="MYAPP"

# Lowercase — ${VAR,,}
echo "${ENV,,}"     # production
echo "${app,,}"     # myapp

# Uppercase — ${VAR^^}
echo "${ENV^^}"     # PRODUCTION
echo "${app^^}"     # MYAPP

# Lowercase first letter only — ${VAR,}
echo "${ENV,}"      # pRODUCTION

# Uppercase first letter only — ${VAR^}
name="john"
echo "${name^}"     # John
```

**DevOps use — normalize environment name:**
```bash
# User might type PROD, Prod, or prod — normalize to lowercase
RAW_INPUT="PROD"
ENV="${RAW_INPUT,,}"    # prod
echo "Deploying to: $ENV"
```

---

## Checking String Content

### Check if a string is empty or not

```bash
#!/bin/bash
set -euo pipefail

NAME=""
VERSION="1.4.2"

# Check if empty
if [ -z "$NAME" ]; then
  echo "NAME is empty"
fi

# Check if NOT empty
if [ -n "$VERSION" ]; then
  echo "VERSION is set: $VERSION"
fi
```

| Check | Meaning |
|---|---|
| `[ -z "$VAR" ]` | True if string IS empty |
| `[ -n "$VAR" ]` | True if string is NOT empty |

---

### Check if a string equals another

```bash
#!/bin/bash
set -euo pipefail

ENV="prod"

if [ "$ENV" = "prod" ]; then
  echo "This is production — be careful!"
fi

if [ "$ENV" != "dev" ]; then
  echo "Not a dev environment"
fi
```

---

### Check if a string contains a pattern

```bash
#!/bin/bash
set -euo pipefail

LOGLINE="2024-06-12 [ERROR] Failed to connect to database"
FILENAME="myapp-v1.4.2.tar.gz"

# Use [[ ]] with =~ for pattern matching
if [[ "$LOGLINE" =~ "ERROR" ]]; then
  echo "Error found in log line"
fi

if [[ "$FILENAME" =~ \.tar\.gz$ ]]; then
  echo "This is a tar.gz file"
fi

if [[ "$FILENAME" =~ ^myapp ]]; then
  echo "This file belongs to myapp"
fi
```

> Always use `[[ ]]` (double brackets) for pattern matching.
> Single brackets `[ ]` do not support `=~`.

---

## Splitting a String

### Split using `IFS` (Internal Field Separator)

```bash
#!/bin/bash
set -euo pipefail

# Split a colon-separated string
HOST_PORT="prod-server:3000"

# Method 1 — using IFS with read
IFS=':' read -r HOST PORT <<< "$HOST_PORT"
echo "Host: $HOST"    # prod-server
echo "Port: $PORT"    # 3000
```

**DevOps use — split version into parts:**
```bash
VERSION="1.4.2"

IFS='.' read -r MAJOR MINOR PATCH <<< "$VERSION"
echo "Major: $MAJOR"    # 1
echo "Minor: $MINOR"    # 4
echo "Patch: $PATCH"    # 2
```

**DevOps use — split comma-separated list:**
```bash
SERVERS="web01,web02,web03"

IFS=',' read -ra SERVER_LIST <<< "$SERVERS"

for server in "${SERVER_LIST[@]}"; do
  echo "Deploying to: $server"
done
```

**Output:**
```
Deploying to: web01
Deploying to: web02
Deploying to: web03
```

---

## Real DevOps Examples

### Example 1 — Parse artifact filename

```bash
#!/bin/bash
set -euo pipefail

ARTIFACT="myapp-v1.4.2-prod.tar.gz"

# Extract app name (everything before first -)
APP="${ARTIFACT%%-*}"
echo "App: $APP"                  # myapp

# Extract version (remove prefix up to v, remove suffix from -)
TEMP="${ARTIFACT#*-v}"            # 1.4.2-prod.tar.gz
VERSION="${TEMP%%-*}"             # 1.4.2
echo "Version: $VERSION"

# Extract environment
TEMP2="${ARTIFACT##*-}"           # prod.tar.gz
ENV="${TEMP2%.tar.gz}"            # prod
echo "Environment: $ENV"

# Check it is a tar.gz
if [[ "$ARTIFACT" =~ \.tar\.gz$ ]]; then
  echo "Valid artifact format ✓"
fi
```

**Output:**
```
App: myapp
Version: 1.4.2
Environment: prod
Valid artifact format ✓
```

---

### Example 2 — Build release directory name

```bash
#!/bin/bash
set -euo pipefail

APP="My-App"
VERSION="v1.4.2"
DATE=$(date '+%Y%m%d')

# Normalize — lowercase, remove v prefix, add date
APP_CLEAN="${APP,,}"              # my-app
VERSION_CLEAN="${VERSION#v}"      # 1.4.2
RELEASE_DIR="${APP_CLEAN}-${VERSION_CLEAN}-${DATE}"

echo "Release dir: $RELEASE_DIR"    # my-app-1.4.2-20240612
```

---

### Example 3 — Validate a version string

```bash
#!/bin/bash
set -euo pipefail

validate_version() {
  local version=$1

  # Must match pattern: number.number.number
  if [[ "$version" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Valid version: $version ✓"
    return 0
  else
    echo "Invalid version: $version"
    echo "Expected format: X.Y.Z (e.g. 1.4.2)"
    return 1
  fi
}

validate_version "1.4.2"     # Valid
validate_version "v1.4.2"    # Invalid — has v prefix
validate_version "latest"    # Invalid
validate_version "2.0.0"     # Valid
```

**Output:**
```
Valid version: 1.4.2 ✓
Invalid version: v1.4.2
Expected format: X.Y.Z (e.g. 1.4.2)
Invalid version: latest
Expected format: X.Y.Z (e.g. 1.4.2)
Valid version: 2.0.0 ✓
```

---

## Common Mistakes

### Mistake 1 — Forgetting quotes around variables

```bash
# Wrong — breaks if string has spaces
NAME="my app"
if [ $NAME = "my app" ]; then    # error: too many arguments
  echo "match"
fi

# Correct — always quote variables
if [ "$NAME" = "my app" ]; then
  echo "match"
fi
```

---

### Mistake 2 — Using single `[ ]` for pattern matching

```bash
# Wrong — single brackets do not support =~
if [ "$VERSION" =~ ^[0-9] ]; then
  echo "starts with number"
fi

# Correct — use double brackets for =~
if [[ "$VERSION" =~ ^[0-9] ]]; then
  echo "starts with number"
fi
```

---

### Mistake 3 — Confusing `#` and `%`

```bash
FILE="myapp.tar.gz"

# # removes from the FRONT
echo "${FILE#myapp}"    # .tar.gz  ✓

# % removes from the END
echo "${FILE%.gz}"      # myapp.tar  ✓

# Mixing them up — common beginner mistake
echo "${FILE%myapp}"    # myapp.tar.gz  — wrong, % works from end
echo "${FILE#.gz}"      # myapp.tar.gz  — wrong, # works from front
```

---

### Mistake 4 — Not using `<<<` with `read`

```bash
# Wrong — pipe creates a subshell, variables are lost
echo "prod-server:3000" | IFS=':' read -r HOST PORT
echo "$HOST"    # empty! variables lost after pipe

# Correct — use <<< (here string)
IFS=':' read -r HOST PORT <<< "prod-server:3000"
echo "$HOST"    # prod-server ✓
```

---

## Quick Exercise

> **Task:** Write a script called `parse-artifact.sh` that:
>
> Given this artifact name: `webapp-v2.1.0-staging.tar.gz`
>
> 1. Extract and print the **app name** → `webapp`
> 2. Extract and print the **version** → `2.1.0` (without the `v`)
> 3. Extract and print the **environment** → `staging`
> 4. Convert the environment to **UPPERCASE** → `STAGING`
> 5. Build and print the **release directory** name → `webapp-2.1.0-staging`
> 6. Validate the version matches `X.Y.Z` format — print `Valid` or `Invalid`
> 7. Check if the file ends in `.tar.gz` — print `Valid format` or `Wrong format`

**Expected output:**
```
App name   : webapp
Version    : 2.1.0
Environment: staging
Env upper  : STAGING
Release dir: webapp-2.1.0-staging
Version    : Valid
Format     : Valid format
```

**Expected solution:**

```bash
#!/bin/bash
set -euo pipefail

ARTIFACT="webapp-v2.1.0-staging.tar.gz"

# 1. App name — everything before first -
APP="${ARTIFACT%%-*}"
echo "App name   : $APP"

# 2. Version — remove prefix up to v, remove from second -
TEMP="${ARTIFACT#*-v}"
VERSION="${TEMP%%-*}"
echo "Version    : $VERSION"

# 3. Environment — remove app-vX.Y.Z- prefix, remove .tar.gz suffix
TEMP2="${ARTIFACT##*[0-9]-}"
ENV="${TEMP2%.tar.gz}"
echo "Environment: $ENV"

# 4. Uppercase
ENV_UPPER="${ENV^^}"
echo "Env upper  : $ENV_UPPER"

# 5. Release dir
RELEASE="${APP}-${VERSION}-${ENV}"
echo "Release dir: $RELEASE"

# 6. Validate version
if [[ "$VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
  echo "Version    : Valid"
else
  echo "Version    : Invalid"
fi

# 7. Check format
if [[ "$ARTIFACT" =~ \.tar\.gz$ ]]; then
  echo "Format     : Valid format"
else
  echo "Format     : Wrong format"
fi
```

---

## Summary — Quick Reference Card

| Operation | Syntax | Example |
|---|---|---|
| Length | `${#VAR}` | `${#VERSION}` → `5` |
| Substring | `${VAR:start:len}` | `${VER:0:1}` → `1` |
| Remove shortest from front | `${VAR#pattern}` | `${F#/}` |
| Remove longest from front | `${VAR##pattern}` | `${F##*/}` → filename |
| Remove shortest from end | `${VAR%pattern}` | `${F%.gz}` |
| Remove longest from end | `${VAR%%pattern}` | `${F%%.tar.gz}` |
| Replace first | `${VAR/old/new}` | `${V/staging/prod}` |
| Replace all | `${VAR//old/new}` | `${V//-/_}` |
| Lowercase | `${VAR,,}` | `${ENV,,}` → `prod` |
| Uppercase | `${VAR^^}` | `${ENV^^}` → `PROD` |
| Is empty | `[ -z "$VAR" ]` | check empty |
| Not empty | `[ -n "$VAR" ]` | check has value |
| Pattern match | `[[ "$VAR" =~ regex ]]` | check format |
| Split string | `IFS='x' read -r A B <<< "$VAR"` | split on delimiter |

---

## What's Next

**Intermediate Topic 3 — Arrays**  
Learn how to store a list of values — server names, file paths,
environment names — and loop over them cleanly.
Arrays are what turn a single-server script into a
multi-server automation tool.