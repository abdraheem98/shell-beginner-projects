# Intermediate Topic 3 — Arrays

> **Session:** Shell Scripting for DevOps — Intermediate Concepts  
> **Duration:** ~20 minutes  
> **Prerequisite:** Topic 1 (Exit Codes) + Topic 1+ (Safety Patterns) + Topic 2 (String Manipulation)

---

## Table of Contents

- [What Is an Array?](#what-is-an-array)
- [Why DevOps Engineers Need This](#why-devops-engineers-need-this)
- [Creating Arrays](#creating-arrays)
- [Reading from Arrays](#reading-from-arrays)
- [Looping Over Arrays](#looping-over-arrays)
- [Adding and Removing Items](#adding-and-removing-items)
- [Array of Commands Output](#array-of-commands-output)
- [Associative Arrays — Key Value Pairs](#associative-arrays--key-value-pairs)
- [Real DevOps Examples](#real-devops-examples)
- [Common Mistakes](#common-mistakes)
- [Quick Exercise](#quick-exercise)

---

## What Is an Array?

A regular variable holds **one value**:

```bash
SERVER="web01"
```

An array holds **multiple values** in one variable:

```bash
SERVERS=("web01" "web02" "web03")
```

Think of it like a numbered list:

```
SERVERS[0] = web01
SERVERS[1] = web02
SERVERS[2] = web03
```

---

## Why DevOps Engineers Need This

Without arrays, scripts repeat themselves badly:

```bash
# Without arrays — messy and hard to maintain
deploy web01
deploy web02
deploy web03
deploy web04
deploy web05
# add a new server? edit 6 lines
```

With arrays — clean and maintainable:

```bash
# With arrays — add a server? edit ONE line
SERVERS=("web01" "web02" "web03" "web04" "web05")

for server in "${SERVERS[@]}"; do
  deploy "$server"
done
```

Arrays are used every day in DevOps for:

| Use Case | Example |
|---|---|
| List of servers | `("web01" "web02" "web03")` |
| List of services to check | `("nginx" "mysql" "redis")` |
| List of files to back up | `("/etc/nginx" "/etc/mysql")` |
| List of required binaries | `("curl" "aws" "jq")` |
| List of environments | `("dev" "staging" "prod")` |

---

## Creating Arrays

### Method 1 — All items on one line

```bash
#!/bin/bash
set -euo pipefail

# Values separated by spaces, wrapped in ()
SERVERS=("web01" "web02" "web03")
SERVICES=("nginx" "mysql" "redis")
NUMBERS=(1 2 3 4 5)
```

### Method 2 — One item per line (easier to read)

```bash
#!/bin/bash
set -euo pipefail

SERVERS=(
  "web01"
  "web02"
  "web03"
  "web04"
)
```

> Use Method 2 when your list is long — much easier to
> add, remove, or comment out individual items.

### Method 3 — Empty array, add items later

```bash
#!/bin/bash
set -euo pipefail

SERVERS=()    # start empty

SERVERS+=("web01")    # add web01
SERVERS+=("web02")    # add web02
SERVERS+=("web03")    # add web03

echo "${#SERVERS[@]}"    # 3
```

---

## Reading from Arrays

```bash
#!/bin/bash
set -euo pipefail

SERVERS=("web01" "web02" "web03")

# Read one item by index (starts at 0)
echo "${SERVERS[0]}"    # web01
echo "${SERVERS[1]}"    # web02
echo "${SERVERS[2]}"    # web03

# Read ALL items
echo "${SERVERS[@]}"    # web01 web02 web03

# Count how many items
echo "${#SERVERS[@]}"   # 3

# Get the last item
echo "${SERVERS[-1]}"   # web03

# Get all indexes
echo "${!SERVERS[@]}"   # 0 1 2
```

### The most important syntax to memorize

| Syntax | Meaning |
|---|---|
| `${ARRAY[0]}` | First item |
| `${ARRAY[-1]}` | Last item |
| `${ARRAY[@]}` | ALL items |
| `${#ARRAY[@]}` | Count of items |
| `${!ARRAY[@]}` | All indexes |

> `[@]` means "all elements" — you will use this constantly.
> Always wrap it in double quotes: `"${ARRAY[@]}"` not `${ARRAY[@]}`.

---

## Looping Over Arrays

This is the most common thing you will do with arrays:

### Basic loop

```bash
#!/bin/bash
set -euo pipefail

SERVERS=("web01" "web02" "web03")

for server in "${SERVERS[@]}"; do
  echo "Processing: $server"
done
```

**Output:**
```
Processing: web01
Processing: web02
Processing: web03
```

### Loop with index

```bash
#!/bin/bash
set -euo pipefail

SERVERS=("web01" "web02" "web03")

for i in "${!SERVERS[@]}"; do
  echo "Server $i: ${SERVERS[$i]}"
done
```

**Output:**
```
Server 0: web01
Server 1: web02
Server 2: web03
```

### Loop and do real work

```bash
#!/bin/bash
set -euo pipefail

SERVICES=("nginx" "mysql" "redis")

echo "Checking services..."
for service in "${SERVICES[@]}"; do
  if systemctl is-active --quiet "$service" 2>/dev/null; then
    echo "  $service — running ✓"
  else
    echo "  $service — STOPPED ✗"
  fi
done
```

**Output:**
```
Checking services...
  nginx — running ✓
  mysql — running ✓
  redis — STOPPED ✗
```

---

## Adding and Removing Items

### Adding items

```bash
#!/bin/bash
set -euo pipefail

SERVERS=("web01" "web02")

# Add one item
SERVERS+=("web03")

# Add multiple items
SERVERS+=("web04" "web05")

echo "${SERVERS[@]}"     # web01 web02 web03 web04 web05
echo "${#SERVERS[@]}"    # 5
```

### Removing items

```bash
#!/bin/bash
set -euo pipefail

SERVERS=("web01" "web02" "web03" "web04")

# Remove by index — unset leaves a gap
unset 'SERVERS[1]'
echo "${SERVERS[@]}"      # web01 web03 web04
echo "${#SERVERS[@]}"     # 3 (gap is gone from count)

# Remove item by value — rebuild the array without it
TARGET="web03"
NEW_SERVERS=()
for server in "${SERVERS[@]}"; do
  if [ "$server" != "$TARGET" ]; then
    NEW_SERVERS+=("$server")
  fi
done
SERVERS=("${NEW_SERVERS[@]}")
echo "${SERVERS[@]}"    # web01 web04
```

### Slicing — get a portion of an array

```bash
#!/bin/bash
set -euo pipefail

SERVERS=("web01" "web02" "web03" "web04" "web05")

# Get items from index 1, take 3 items
SLICE=("${SERVERS[@]:1:3}")
echo "${SLICE[@]}"    # web02 web03 web04

# Get everything from index 2 onwards
REST=("${SERVERS[@]:2}")
echo "${REST[@]}"     # web03 web04 web05
```

---

## Array of Commands Output

Very common pattern — capture command output into an array:

```bash
#!/bin/bash
set -euo pipefail

# Get list of running services into an array
mapfile -t RUNNING_SERVICES < <(systemctl list-units --type=service \
  --state=running --no-legend | awk '{print $1}')

echo "Running services: ${#RUNNING_SERVICES[@]}"
for svc in "${RUNNING_SERVICES[@]}"; do
  echo "  $svc"
done
```

```bash
#!/bin/bash
set -euo pipefail

# Get list of files into an array
mapfile -t LOG_FILES < <(find /var/log -name "*.log" -type f)

echo "Found ${#LOG_FILES[@]} log files"
for log in "${LOG_FILES[@]}"; do
  size=$(du -sh "$log" | cut -f1)
  echo "  $log — $size"
done
```

```bash
#!/bin/bash
set -euo pipefail

# Get list of releases into an array
RELEASE_DIR="/opt/myapp/releases"
mapfile -t RELEASES < <(ls -dt "$RELEASE_DIR"/*/ 2>/dev/null | xargs -I{} basename {})

echo "Available releases: ${#RELEASES[@]}"
for release in "${RELEASES[@]}"; do
  echo "  $release"
done
```

> `mapfile -t ARRAY < <(command)` is the cleanest way to fill an
> array from command output. The `-t` removes trailing newlines.

---

## Associative Arrays — Key Value Pairs

A regular array uses numbers as indexes (0, 1, 2).
An **associative array** uses text as keys — like a dictionary:

```bash
#!/bin/bash
set -euo pipefail

# Must declare with -A for associative arrays
declare -A SERVER_IPS

# Set key-value pairs
SERVER_IPS["web01"]="10.0.1.10"
SERVER_IPS["web02"]="10.0.1.11"
SERVER_IPS["web03"]="10.0.1.12"

# Read a value by key
echo "${SERVER_IPS["web01"]}"    # 10.0.1.10

# Get all keys
echo "${!SERVER_IPS[@]}"    # web01 web02 web03

# Get all values
echo "${SERVER_IPS[@]}"     # 10.0.1.10 10.0.1.11 10.0.1.12

# Loop over key-value pairs
for server in "${!SERVER_IPS[@]}"; do
  echo "  $server → ${SERVER_IPS[$server]}"
done
```

**Output:**
```
web01 → 10.0.1.10
web02 → 10.0.1.11
web03 → 10.0.1.12
```

### Real DevOps use — environment to bucket mapping

```bash
#!/bin/bash
set -euo pipefail

declare -A S3_BUCKETS
S3_BUCKETS["dev"]="s3://dev-artifacts"
S3_BUCKETS["staging"]="s3://staging-artifacts"
S3_BUCKETS["prod"]="s3://prod-artifacts"

ENV="staging"
BUCKET="${S3_BUCKETS[$ENV]}"
echo "Uploading to: $BUCKET"    # s3://staging-artifacts
```

### Real DevOps use — service to port mapping

```bash
#!/bin/bash
set -euo pipefail

declare -A SERVICE_PORTS
SERVICE_PORTS["nginx"]="80"
SERVICE_PORTS["myapp"]="3000"
SERVICE_PORTS["mysql"]="3306"
SERVICE_PORTS["redis"]="6379"

for service in "${!SERVICE_PORTS[@]}"; do
  port="${SERVICE_PORTS[$service]}"
  if ss -tlnp | grep -q ":$port "; then
    echo "  $service (port $port) — listening ✓"
  else
    echo "  $service (port $port) — NOT listening ✗"
  fi
done
```

---

## Real DevOps Examples

### Example 1 — Check all required binaries

```bash
#!/bin/bash
set -euo pipefail

REQUIRED_BINARIES=("aws" "curl" "tar" "jq" "npm" "systemctl")

echo "Checking required binaries..."
MISSING=()

for bin in "${REQUIRED_BINARIES[@]}"; do
  if command -v "$bin" > /dev/null 2>&1; then
    echo "  $bin ✓"
  else
    echo "  $bin — NOT FOUND ✗"
    MISSING+=("$bin")
  fi
done

if [ ${#MISSING[@]} -gt 0 ]; then
  echo ""
  echo "Missing binaries: ${MISSING[*]}"
  echo "Install them and try again"
  exit 1
fi

echo ""
echo "All binaries present ✓"
```

**Output:**
```
Checking required binaries...
  aws ✓
  curl ✓
  tar ✓
  jq — NOT FOUND ✗
  npm ✓
  systemctl ✓

Missing binaries: jq
Install them and try again
```

---

### Example 2 — Deploy to multiple servers

```bash
#!/bin/bash
set -euo pipefail

SERVERS=("web01" "web02" "web03")
VERSION="1.4.2"
FAILED=()

echo "Deploying v$VERSION to ${#SERVERS[@]} servers..."

for server in "${SERVERS[@]}"; do
  echo ""
  echo "Deploying to: $server"

  if ssh "deploy@$server" "./deploy.sh -v $VERSION" 2>/dev/null; then
    echo "  $server — SUCCESS ✓"
  else
    echo "  $server — FAILED ✗"
    FAILED+=("$server")
  fi
done

echo ""
echo "─── Deploy Summary ───────────────────────"
echo "Total   : ${#SERVERS[@]}"
echo "Success : $(( ${#SERVERS[@]} - ${#FAILED[@]} ))"
echo "Failed  : ${#FAILED[@]}"

if [ ${#FAILED[@]} -gt 0 ]; then
  echo "Failed servers: ${FAILED[*]}"
  exit 1
fi

echo "All servers deployed successfully ✓"
```

---

### Example 3 — Cleanup old releases, keep last N

```bash
#!/bin/bash
set -euo pipefail

RELEASE_DIR="/opt/myapp/releases"
KEEP=5

# Get all releases sorted newest first
mapfile -t ALL_RELEASES < <(ls -dt "$RELEASE_DIR"/*/ 2>/dev/null)

TOTAL=${#ALL_RELEASES[@]}
echo "Total releases: $TOTAL | Keeping: $KEEP"

if [ $TOTAL -le $KEEP ]; then
  echo "Nothing to clean up"
  exit 0
fi

# Everything after index KEEP-1 gets deleted
TO_DELETE=("${ALL_RELEASES[@]:$KEEP}")

echo "Deleting ${#TO_DELETE[@]} old release(s)..."
for release in "${TO_DELETE[@]}"; do
  echo "  Removing: $(basename "$release")"
  rm -rf "$release"
done

echo "Cleanup complete ✓"
```

---

## Common Mistakes

### Mistake 1 — Missing quotes around `[@]`

```bash
SERVERS=("web01" "web02" "web 03")    # "web 03" has a space

# Wrong — space in "web 03" breaks it into two items
for server in ${SERVERS[@]}; do
  echo "$server"
done
# Output: web01  web02  web  03   ← wrong!

# Correct — always quote [@]
for server in "${SERVERS[@]}"; do
  echo "$server"
done
# Output: web01  web02  web 03   ← correct
```

---

### Mistake 2 — Using `*` instead of `@`

```bash
SERVERS=("web01" "web02" "web03")

# Wrong — * joins everything into one string
for server in "${SERVERS[*]}"; do
  echo "$server"    # prints "web01 web02 web03" as ONE item
done

# Correct — @ keeps items separate
for server in "${SERVERS[@]}"; do
  echo "$server"    # prints each item separately
done
```

> Rule: **always use `@`**, never `*` when looping.

---

### Mistake 3 — Forgetting `declare -A` for associative arrays

```bash
# Wrong — without declare -A it treats it as a regular array
SERVER_IPS["web01"]="10.0.1.10"    # error or unexpected behavior

# Correct — always declare first
declare -A SERVER_IPS
SERVER_IPS["web01"]="10.0.1.10"
```

---

### Mistake 4 — Array index starts at 0, not 1

```bash
SERVERS=("web01" "web02" "web03")

echo "${SERVERS[1]}"    # web02 — NOT web01!
echo "${SERVERS[0]}"    # web01 — first item is index 0
```

---

### Mistake 5 — Using `echo` to print arrays

```bash
SERVERS=("web01" "web02" "web03")

# Wrong — only prints first item
echo "$SERVERS"         # web01

# Correct — use [@]
echo "${SERVERS[@]}"    # web01 web02 web03
```

---

## Quick Exercise

> **Task:** Write a script called `server-check.sh` that:
>
> 1. Defines an array of services: `nginx`, `mysql`, `redis`, `ssh`
> 2. Loops through each service and checks if it is running
>    using `systemctl is-active`
> 3. Adds running services to a `RUNNING` array
> 4. Adds stopped services to a `STOPPED` array
> 5. At the end prints a summary:
>    ```
>    Running (3): nginx mysql ssh
>    Stopped (1): redis
>    ```
> 6. Exits with code `1` if ANY service is stopped
> 7. Exits with code `0` if ALL services are running

**Expected output (if redis is down):**
```
Checking services...
  nginx  — running ✓
  mysql  — running ✓
  redis  — STOPPED ✗
  ssh    — running ✓

─── Summary ───────────────────────
Running (3): nginx mysql ssh
Stopped (1): redis

Some services are down — check above
```

**Expected solution:**

```bash
#!/bin/bash
set -euo pipefail

SERVICES=("nginx" "mysql" "redis" "ssh")
RUNNING=()
STOPPED=()

echo "Checking services..."
for service in "${SERVICES[@]}"; do
  if systemctl is-active --quiet "$service" 2>/dev/null; then
    echo "  $service — running ✓"
    RUNNING+=("$service")
  else
    echo "  $service — STOPPED ✗"
    STOPPED+=("$service")
  fi
done

echo ""
echo "─── Summary ───────────────────────"
echo "Running (${#RUNNING[@]}): ${RUNNING[*]:-none}"
echo "Stopped (${#STOPPED[@]}): ${STOPPED[*]:-none}"

if [ ${#STOPPED[@]} -gt 0 ]; then
  echo ""
  echo "Some services are down — check above"
  exit 1
fi

echo ""
echo "All services running ✓"
exit 0
```

---

## Summary — Quick Reference Card

| Operation | Syntax | Example |
|---|---|---|
| Create array | `ARR=("a" "b" "c")` | `SERVERS=("web01" "web02")` |
| Get one item | `${ARR[0]}` | First item |
| Get last item | `${ARR[-1]}` | Last item |
| Get all items | `"${ARR[@]}"` | All items — always quoted |
| Count items | `${#ARR[@]}` | Number of elements |
| Get all indexes | `${!ARR[@]}` | 0 1 2 ... |
| Add item | `ARR+=("new")` | Append to end |
| Remove by index | `unset 'ARR[1]'` | Remove second item |
| Loop over items | `for x in "${ARR[@]}"` | Iterate each item |
| Loop with index | `for i in "${!ARR[@]}"` | Get index and value |
| Slice array | `"${ARR[@]:1:3}"` | Items from index 1, take 3 |
| Fill from command | `mapfile -t ARR < <(cmd)` | Command output to array |
| Associative array | `declare -A MAP` | Key-value pairs |
| Set key value | `MAP["key"]="val"` | Add to map |
| Get by key | `${MAP["key"]}` | Read from map |
| All keys | `${!MAP[@]}` | Keys of map |

---

## What's Next

**Intermediate Topic 4 — Reading Files Line by Line**  
Learn how to open a file and process it one line at a time —
reading config files, parsing CSVs, scanning log files,
and processing lists of servers or hostnames.
This is one of the most common tasks in real DevOps scripts.