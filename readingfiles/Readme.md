# Intermediate Topic 4 — Reading Files Line by Line

> **Session:** Shell Scripting for DevOps — Intermediate Concepts  
> **Duration:** ~20 minutes  
> **Prerequisite:** Topic 1 (Exit Codes) + Topic 1+ (Safety Patterns) + Topic 2 (Strings) + Topic 3 (Arrays)

---

## Table of Contents

- [What Does Reading Files Mean?](#what-does-reading-files-mean)
- [Why DevOps Engineers Need This](#why-devops-engineers-need-this)
- [Method 1 — while read Loop](#method-1--while-read-loop)
- [Method 2 — Reading into a Variable](#method-2--reading-into-a-variable)
- [Method 3 — Reading with a Field Separator](#method-3--reading-with-a-field-separator)
- [Skipping Lines — Comments and Blanks](#skipping-lines--comments-and-blanks)
- [Reading Specific Columns from a File](#reading-specific-columns-from-a-file)
- [Writing to Files](#writing-to-files)
- [Checking Files Before Reading](#checking-files-before-reading)
- [Real DevOps Examples](#real-devops-examples)
- [Common Mistakes](#common-mistakes)
- [Quick Exercise](#quick-exercise)

---

## What Does Reading Files Mean?

Reading a file line by line means your script opens a file,
takes **one line at a time**, and does something with it —
checks it, transforms it, extracts values from it.

Think of it like a cashier scanning items one by one:

```
Line 1 → read → process → next
Line 2 → read → process → next
Line 3 → read → process → next
...
```

---

## Why DevOps Engineers Need This

Files are everywhere in DevOps — and scripts need to process them constantly:

| File Type | What You Do With It |
|---|---|
| Server list (`servers.txt`) | Loop and deploy to each server |
| Config file (`app.env`) | Read key=value pairs |
| CSV report | Extract specific columns |
| Log file | Scan for errors line by line |
| Package list | Install each package |
| IP whitelist | Check if an IP is allowed |

Without file reading — you hardcode everything in the script.
With file reading — you separate data from logic cleanly.

---

## Method 1 — `while read` Loop

This is the **standard way** to read a file line by line in bash.

### Basic syntax

```bash
while IFS= read -r line; do
  echo "$line"
done < filename.txt
```

Breaking it down:

| Part | Meaning |
|---|---|
| `while` | Keep looping as long as there are lines |
| `IFS=` | Do not strip leading/trailing spaces |
| `read` | Read one line at a time |
| `-r` | Do not interpret backslashes (`\n` stays as `\n`) |
| `line` | Variable that holds the current line |
| `done < filename.txt` | Feed the file into the loop |

### Simple example

Create a file called `servers.txt`:

```
web01
web02
web03
db01
```

Read it line by line:

```bash
#!/bin/bash
set -euo pipefail

while IFS= read -r line; do
  echo "Processing server: $line"
done < servers.txt
```

**Output:**
```
Processing server: web01
Processing server: web02
Processing server: web03
Processing server: db01
```

### Always use `IFS= read -r`

```bash
# Wrong — loses leading spaces, misreads backslashes
while read line; do
  echo "$line"
done < file.txt

# Correct — preserves everything exactly as it is in the file
while IFS= read -r line; do
  echo "$line"
done < file.txt
```

---

## Method 2 — Reading into a Variable

Sometimes you want to read the whole file or specific lines
into a variable instead of looping:

```bash
#!/bin/bash
set -euo pipefail

# Read entire file into a variable
CONTENT=$(cat servers.txt)
echo "$CONTENT"

# Read just the first line
FIRST_LINE=$(head -1 servers.txt)
echo "First server: $FIRST_LINE"

# Read just the last line
LAST_LINE=$(tail -1 servers.txt)
echo "Last server: $LAST_LINE"

# Read line number 3
LINE_3=$(sed -n '3p' servers.txt)
echo "Line 3: $LINE_3"

# Count total lines
LINE_COUNT=$(wc -l < servers.txt)
echo "Total lines: $LINE_COUNT"
```

---

## Method 3 — Reading with a Field Separator

When each line has multiple fields separated by a delimiter
(like `:`, `,`, or `=`), split them while reading:

### Reading colon-separated values

```bash
#!/bin/bash
set -euo pipefail

# File: server-ports.txt
# web01:80
# web02:443
# db01:3306

while IFS=':' read -r server port; do
  echo "Server: $server | Port: $port"
done < server-ports.txt
```

**Output:**
```
Server: web01 | Port: 80
Server: web02 | Port: 443
Server: db01  | Port: 3306
```

### Reading key=value config files

```bash
#!/bin/bash
set -euo pipefail

# File: app.env
# APP_NAME=myapp
# APP_PORT=3000
# DB_HOST=localhost
# DB_PORT=5432

while IFS='=' read -r key value; do
  echo "Key: $key | Value: $value"
done < app.env
```

**Output:**
```
Key: APP_NAME | Value: myapp
Key: APP_PORT | Value: 3000
Key: DB_HOST  | Value: localhost
Key: DB_PORT  | Value: 5432
```

### Reading CSV files

```bash
#!/bin/bash
set -euo pipefail

# File: servers.csv
# web01,prod,10.0.1.10,active
# web02,prod,10.0.1.11,active
# web03,staging,10.0.2.10,inactive

while IFS=',' read -r name env ip status; do
  echo "Name: $name | Env: $env | IP: $ip | Status: $status"
done < servers.csv
```

**Output:**
```
Name: web01 | Env: prod    | IP: 10.0.1.10 | Status: active
Name: web02 | Env: prod    | IP: 10.0.1.11 | Status: active
Name: web03 | Env: staging | IP: 10.0.2.10 | Status: inactive
```

---

## Skipping Lines — Comments and Blanks

Real config files have blank lines and comment lines starting with `#`.
Always skip them:

```bash
#!/bin/bash
set -euo pipefail

# File: servers.txt
# # This is a comment
# web01
#
# web02
# # Another comment
# web03

while IFS= read -r line; do

  # Skip blank lines
  [[ -z "$line" ]] && continue

  # Skip lines that start with #
  [[ "$line" =~ ^# ]] && continue

  # Process the valid line
  echo "Valid server: $line"

done < servers.txt
```

**Output:**
```
Valid server: web01
Valid server: web02
Valid server: web03
```

### One-liner version

```bash
while IFS= read -r line || [[ -n "$line" ]]; do
  # Skip blanks and comments in one condition
  [[ -z "$line" || "$line" =~ ^[[:space:]]*# ]] && continue
  echo "Processing: $line"
done < servers.txt
```

> The `|| [[ -n "$line" ]]` part handles files that do not have
> a newline at the very end — a common real-world issue.

---

## Reading Specific Columns from a File

When you only need some fields from a line — not all of them:

```bash
#!/bin/bash
set -euo pipefail

# File: /etc/passwd — colon separated
# root:x:0:0:root:/root:/bin/bash
# ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash

# Read only username (field 1) and shell (field 7)
while IFS=':' read -r username _ _ _ _ _ shell; do
  echo "User: $username | Shell: $shell"
done < /etc/passwd
```

**Output:**
```
User: root   | Shell: /bin/bash
User: ubuntu | Shell: /bin/bash
...
```

> The `_` variable is used to throw away fields you do not need.
> It is a convention — you can use any name, but `_` is clear.

```bash
#!/bin/bash
set -euo pipefail

# File: servers.csv
# web01,prod,10.0.1.10,active,us-east-1

# Read only name and IP — skip env, status, region
while IFS=',' read -r name _ ip _; do
  echo "Pinging $name at $ip..."
  ping -c 1 "$ip" > /dev/null 2>&1 && \
    echo "  $name is reachable ✓" || \
    echo "  $name is UNREACHABLE ✗"
done < servers.csv
```

---

## Writing to Files

Reading and writing go together — learn both at the same time:

### Overwrite a file — `>`

```bash
#!/bin/bash
set -euo pipefail

# Create or overwrite
echo "web01" > servers.txt
echo "web02" >> servers.txt    # append
echo "web03" >> servers.txt    # append

cat servers.txt
```

### Append to a file — `>>`

```bash
#!/bin/bash
set -euo pipefail

LOG_FILE="/var/log/myapp/deploy.log"

# >> appends — never overwrites
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Deploy started" >> "$LOG_FILE"
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Deploy complete" >> "$LOG_FILE"
```

### Write multiple lines — heredoc

```bash
#!/bin/bash
set -euo pipefail

# Write a config file with heredoc
cat > /tmp/app.env << EOF
APP_NAME=myapp
APP_PORT=3000
DB_HOST=localhost
DB_PORT=5432
LOG_LEVEL=INFO
EOF

cat /tmp/app.env
```

**Output:**
```
APP_NAME=myapp
APP_PORT=3000
DB_HOST=localhost
DB_PORT=5432
LOG_LEVEL=INFO
```

### Write only if file does not exist

```bash
#!/bin/bash
set -euo pipefail

CONFIG_FILE="/opt/myapp/.env"

if [ ! -f "$CONFIG_FILE" ]; then
  echo "Creating default config..."
  cat > "$CONFIG_FILE" << EOF
APP_NAME=myapp
APP_PORT=3000
LOG_LEVEL=INFO
EOF
  echo "Config created: $CONFIG_FILE"
else
  echo "Config already exists — skipping"
fi
```

---

## Checking Files Before Reading

Always check the file exists and is readable before processing it:

```bash
#!/bin/bash
set -euo pipefail

SERVER_FILE="servers.txt"

# Check 1 — file exists
if [ ! -f "$SERVER_FILE" ]; then
  echo "ERROR: File not found: $SERVER_FILE"
  exit 1
fi

# Check 2 — file is readable
if [ ! -r "$SERVER_FILE" ]; then
  echo "ERROR: Cannot read file: $SERVER_FILE"
  exit 1
fi

# Check 3 — file is not empty
if [ ! -s "$SERVER_FILE" ]; then
  echo "ERROR: File is empty: $SERVER_FILE"
  exit 1
fi

echo "File is ready — processing..."
while IFS= read -r line; do
  echo "  $line"
done < "$SERVER_FILE"
```

### File check reference

| Check | Meaning |
|---|---|
| `[ -f "$FILE" ]` | File exists and is a regular file |
| `[ -d "$FILE" ]` | Path exists and is a directory |
| `[ -r "$FILE" ]` | File is readable |
| `[ -w "$FILE" ]` | File is writable |
| `[ -s "$FILE" ]` | File exists and is NOT empty |
| `[ ! -f "$FILE" ]` | File does NOT exist |

---

## Real DevOps Examples

### Example 1 — Deploy to servers from a file

```bash
#!/bin/bash
set -euo pipefail

SERVER_FILE="servers.txt"
VERSION="${1:-}"
FAILED=()

[ -z "$VERSION" ] && { echo "Usage: $0 <version>"; exit 1; }
[ -f "$SERVER_FILE" ] || { echo "ERROR: $SERVER_FILE not found"; exit 1; }

echo "Deploying v$VERSION..."
echo ""

while IFS= read -r server; do
  # Skip blank lines and comments
  [[ -z "$server" || "$server" =~ ^# ]] && continue

  echo "  Deploying to: $server"
  if ssh "deploy@$server" "./deploy.sh -v $VERSION" 2>/dev/null; then
    echo "  $server — SUCCESS ✓"
  else
    echo "  $server — FAILED ✗"
    FAILED+=("$server")
  fi
done < "$SERVER_FILE"

echo ""
echo "─── Summary ───────────────────────────────"
echo "Failed: ${#FAILED[@]}"
[ ${#FAILED[@]} -gt 0 ] && echo "Servers: ${FAILED[*]}" && exit 1
echo "All deployments successful ✓"
```

---

### Example 2 — Load a `.env` config file

```bash
#!/bin/bash
set -euo pipefail

load_env() {
  local env_file=$1

  [ -f "$env_file" ] || { echo "ERROR: $env_file not found"; exit 1; }

  echo "Loading config from: $env_file"

  while IFS='=' read -r key value; do
    # Skip blank lines and comments
    [[ -z "$key" || "$key" =~ ^# ]] && continue

    # Skip lines without a value
    [ -z "$value" ] && continue

    # Export the variable so child processes can use it
    export "$key=$value"
    echo "  Loaded: $key"

  done < "$env_file"

  echo "Config loaded ✓"
}

load_env "./config/prod.env"

# Now you can use the variables
echo "App: $APP_NAME running on port $APP_PORT"
```

---

### Example 3 — Scan log file for errors

```bash
#!/bin/bash
set -euo pipefail

LOG_FILE="/var/log/myapp/app.log"
ERROR_LOG="/tmp/errors-$(date '+%Y%m%d').txt"
ERROR_COUNT=0

[ -f "$LOG_FILE" ] || { echo "Log not found: $LOG_FILE"; exit 1; }

echo "Scanning: $LOG_FILE"
echo "Errors found:" > "$ERROR_LOG"

while IFS= read -r line; do
  if [[ "$line" =~ \[ERROR\] ]]; then
    ERROR_COUNT=$(( ERROR_COUNT + 1 ))
    echo "  $line" >> "$ERROR_LOG"
  fi
done < "$LOG_FILE"

echo ""
echo "Total errors found : $ERROR_COUNT"
echo "Error log saved to : $ERROR_LOG"

if [ $ERROR_COUNT -gt 0 ]; then
  echo ""
  echo "Last 5 errors:"
  tail -5 "$ERROR_LOG"
fi
```

---

### Example 4 — Generate report from CSV

```bash
#!/bin/bash
set -euo pipefail

# File: servers.csv
# name,environment,ip,status
# web01,prod,10.0.1.10,active
# web02,prod,10.0.1.11,active
# web03,staging,10.0.2.10,inactive

CSV_FILE="servers.csv"
HEADER_SKIPPED=false
ACTIVE=0
INACTIVE=0

[ -f "$CSV_FILE" ] || { echo "ERROR: $CSV_FILE not found"; exit 1; }

echo "Server Report"
echo "─────────────────────────────────────────"
printf "%-10s %-10s %-15s %-10s\n" "NAME" "ENV" "IP" "STATUS"
echo "─────────────────────────────────────────"

while IFS=',' read -r name env ip status; do
  # Skip header line
  if [ "$HEADER_SKIPPED" = false ]; then
    HEADER_SKIPPED=true
    continue
  fi

  printf "%-10s %-10s %-15s %-10s\n" "$name" "$env" "$ip" "$status"

  if [ "$status" = "active" ]; then
    ACTIVE=$(( ACTIVE + 1 ))
  else
    INACTIVE=$(( INACTIVE + 1 ))
  fi

done < "$CSV_FILE"

echo "─────────────────────────────────────────"
echo "Active: $ACTIVE | Inactive: $INACTIVE"
```

**Output:**
```
Server Report
─────────────────────────────────────────
NAME       ENV        IP              STATUS
─────────────────────────────────────────
web01      prod       10.0.1.10       active
web02      prod       10.0.1.11       active
web03      staging    10.0.2.10       inactive
─────────────────────────────────────────
Active: 2 | Inactive: 1
```

---

## Common Mistakes

### Mistake 1 — Using `for` to read files

```bash
# Wrong — splits on spaces, not lines
# If a line has spaces it breaks into multiple items
for line in $(cat servers.txt); do
  echo "$line"
done

# Correct — while read handles spaces in lines correctly
while IFS= read -r line; do
  echo "$line"
done < servers.txt
```

---

### Mistake 2 — Forgetting `-r` in read

```bash
# Wrong — backslashes get interpreted
# A line like "web\01" becomes "web1"
while read line; do
  echo "$line"
done < servers.txt

# Correct — -r preserves backslashes
while IFS= read -r line; do
  echo "$line"
done < servers.txt
```

---

### Mistake 3 — Piping into while (subshell problem)

```bash
# Wrong — variables set inside while are lost after the loop
# because pipe creates a subshell
COUNT=0
cat servers.txt | while IFS= read -r line; do
  COUNT=$(( COUNT + 1 ))
done
echo "Count: $COUNT"    # always prints 0 !

# Correct — use redirect < instead of pipe
COUNT=0
while IFS= read -r line; do
  COUNT=$(( COUNT + 1 ))
done < servers.txt
echo "Count: $COUNT"    # prints correct count
```

> This is one of the most common and confusing bugs for freshers.
> **Always use `< file` not `cat file |`**

---

### Mistake 4 — Not checking if file exists

```bash
# Wrong — crashes with confusing error if file missing
while IFS= read -r line; do
  echo "$line"
done < servers.txt   # error: No such file

# Correct — check first, fail clearly
[ -f "servers.txt" ] || { echo "ERROR: servers.txt not found"; exit 1; }
while IFS= read -r line; do
  echo "$line"
done < servers.txt
```

---

### Mistake 5 — Forgetting to skip blank lines and comments

```bash
# Wrong — processes blank lines and comments as real data
while IFS= read -r server; do
  ssh "deploy@$server" "./deploy.sh"    # fails on blank lines and # lines
done < servers.txt

# Correct — skip them
while IFS= read -r server; do
  [[ -z "$server" || "$server" =~ ^# ]] && continue
  ssh "deploy@$server" "./deploy.sh"
done < servers.txt
```

---

## Quick Exercise

> **Task:** Create a file called `packages.txt` with this content:
> ```
> # System packages
> curl
> wget
> git
>
> # Monitoring
> htop
> netstat
> ```
>
> Then write a script called `install-packages.sh` that:
>
> 1. Checks `packages.txt` exists — exit with error if not
> 2. Reads the file line by line
> 3. Skips blank lines and lines starting with `#`
> 4. For each valid package — prints `"Installing: <package>"`
> 5. Simulates install with: `echo "  apt install $package -y"`
> 6. Counts how many packages were processed
> 7. At the end prints: `"Done — installed N packages"`

**Expected output:**
```
Reading packages from: packages.txt

Installing: curl
  apt install curl -y
Installing: wget
  apt install wget -y
Installing: git
  apt install git -y
Installing: htop
  apt install htop -y
Installing: netstat
  apt install netstat -y

Done — installed 5 packages
```

**Expected solution:**

```bash
#!/bin/bash
set -euo pipefail

PACKAGE_FILE="packages.txt"
COUNT=0

[ -f "$PACKAGE_FILE" ] || {
  echo "ERROR: $PACKAGE_FILE not found"
  exit 1
}

echo "Reading packages from: $PACKAGE_FILE"
echo ""

while IFS= read -r package; do
  # Skip blank lines
  [[ -z "$package" ]] && continue

  # Skip comment lines
  [[ "$package" =~ ^# ]] && continue

  echo "Installing: $package"
  echo "  apt install $package -y"

  COUNT=$(( COUNT + 1 ))

done < "$PACKAGE_FILE"

echo ""
echo "Done — installed $COUNT packages"
```

---

## Summary — Quick Reference Card

| Operation | Syntax |
|---|---|
| Read file line by line | `while IFS= read -r line; do ... done < file` |
| Read with field split | `while IFS=',' read -r a b c; do ... done < file` |
| Skip blank lines | `[[ -z "$line" ]] && continue` |
| Skip comment lines | `[[ "$line" =~ ^# ]] && continue` |
| Skip unused fields | Use `_` as variable name |
| Read whole file | `CONTENT=$(cat file)` |
| Read first line | `FIRST=$(head -1 file)` |
| Read last line | `LAST=$(tail -1 file)` |
| Count lines | `COUNT=$(wc -l < file)` |
| Write to file | `echo "text" > file` |
| Append to file | `echo "text" >> file` |
| Write multiple lines | `cat > file << EOF ... EOF` |
| Check file exists | `[ -f "$FILE" ]` |
| Check file not empty | `[ -s "$FILE" ]` |
| Check file readable | `[ -r "$FILE" ]` |

---

## The Golden Rule

```
Never use:   cat file | while read line
Always use:  while IFS= read -r line; do ... done < file
```

> The pipe version creates a subshell — variables set inside
> the loop disappear after the loop ends.
> The redirect version keeps all variables alive.

---

## What's Next

**Intermediate Topic 5 — Command Substitution & Pipes**  
Learn how to capture command output into variables,
chain commands together with pipes, and build powerful
one-liners that real DevOps engineers use every day.