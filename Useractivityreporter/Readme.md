# Practice Project 2 — User Activity Reporter

> **Session:** Shell Scripting for DevOps — Practice Projects  
> **Duration:** 15–20 minutes  
> **Skills:** Exit codes, safety patterns, reading files, string manipulation,
>              arrays, loops, command substitution, pipes, writing reports

---

## Table of Contents

- [What We Are Building](#what-we-are-building)
- [What It Will Do](#what-it-will-do)
- [Project Structure](#project-structure)
- [Step 1 — Setup](#step-1--setup)
- [Step 2 — Read Users from passwd](#step-2--read-users-from-passwd)
- [Step 3 — Check Each User](#step-3--check-each-user)
- [Step 4 — Write the Report](#step-4--write-the-report)
- [Step 5 — Print the Summary](#step-5--print-the-summary)
- [Full Script](#full-script)
- [Sample Output](#sample-output)
- [Challenge Extensions](#challenge-extensions)

---

## What We Are Building

A script called `user-report.sh` that scans all system users,
checks their status, and writes a clean formatted report.

This is a real task that system administrators and DevOps engineers
run regularly — especially during security audits and access reviews.

---

## What It Will Do

```
1. Read every user from /etc/passwd
2. For each user — check:
   a. Does their home directory exist?
   b. Do they have a valid login shell?
   c. When did they last log in?
3. Categorize each user:
   - ACTIVE    → valid shell + home exists
   - NO-LOGIN  → shell is /sbin/nologin or /bin/false
   - NO-HOME   → home directory missing
4. Write a formatted report to reports/user-report-<date>.txt
5. Print a summary to the terminal
6. Exit with code 1 if any NO-HOME users are found
```

---

## Project Structure

```
user-reporter/
├── user-report.sh        ← main script
└── reports/
    └── (generated here)  ← report files saved here
```

---

## Step 1 — Setup

Create the project folder:

```bash
mkdir -p user-reporter/reports
cd user-reporter
touch user-report.sh
chmod +x user-report.sh
```

Start with the safe script template:

```bash
#!/bin/bash
# user-report.sh — System user activity reporter
# Usage: ./user-report.sh
# Usage: ./user-report.sh --help

set -euo pipefail

# ── Script directory ───────────────────────────────────
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# ── Config ────────────────────────────────────────────
REPORT_DIR="$SCRIPT_DIR/reports"
DATE=$(date '+%Y-%m-%d')
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
REPORT_FILE="$REPORT_DIR/user-report-${DATE}.txt"
PASSWD_FILE="/etc/passwd"

# ── Logging ───────────────────────────────────────────
log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$1] $2"
}

# ── Error handler ─────────────────────────────────────
handle_error() {
  log "ERROR" "Failed at line $1 — command: $BASH_COMMAND"
  exit 1
}
trap 'handle_error $LINENO' ERR

# ── Usage ─────────────────────────────────────────────
usage() {
  cat << EOF

  Usage: $(basename "$0") [options]

  Options:
    -h, --help     Show this message

  Output:
    Terminal  — summary of all users
    File      — full report saved to $REPORT_DIR

  Example:
    $(basename "$0")

EOF
  exit 0
}

[ "${1:-}" = "-h" ] || [ "${1:-}" = "--help" ] && usage
```

---

## Step 2 — Read Users from passwd

`/etc/passwd` has one user per line in this format:

```
username:password:uid:gid:comment:home_dir:shell
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
```

We need fields 1 (username), 6 (home dir), and 7 (shell).

```bash
# ── Read users from /etc/passwd ───────────────────────
read_users() {
  log "INFO" "Reading users from: $PASSWD_FILE"

  [ -f "$PASSWD_FILE" ] || {
    log "ERROR" "File not found: $PASSWD_FILE"
    exit 1
  }

  [ -r "$PASSWD_FILE" ] || {
    log "ERROR" "Cannot read: $PASSWD_FILE"
    exit 1
  }

  local count
  count=$(wc -l < "$PASSWD_FILE")
  log "INFO" "Found $count entries in $PASSWD_FILE"
}
```

---

## Step 3 — Check Each User

Add arrays to collect results, then loop through passwd:

```bash
# ── Counters and result arrays ────────────────────────
ACTIVE_USERS=()
NOLOGIN_USERS=()
NOHOME_USERS=()

# ── No-login shells ───────────────────────────────────
# These shells mean the user cannot log in interactively
NO_LOGIN_SHELLS=(
  "/sbin/nologin"
  "/usr/sbin/nologin"
  "/bin/false"
  "/usr/bin/false"
)

is_nologin_shell() {
  local shell=$1
  for nologin in "${NO_LOGIN_SHELLS[@]}"; do
    [ "$shell" = "$nologin" ] && return 0
  done
  return 1
}

# ── Check one user ────────────────────────────────────
check_user() {
  local username=$1
  local home_dir=$2
  local shell=$3

  # Check 1 — no-login shell
  if is_nologin_shell "$shell"; then
    NOLOGIN_USERS+=("$username")
    return 0
  fi

  # Check 2 — home directory missing
  if [ ! -d "$home_dir" ]; then
    NOHOME_USERS+=("$username")
    log "WARN" "  $username — home dir missing: $home_dir"
    return 0
  fi

  # Check 3 — active user
  ACTIVE_USERS+=("$username")
}

# ── Loop through all users ────────────────────────────
process_users() {
  log "INFO" "Processing users..."

  while IFS=':' read -r username _ uid _ _ home_dir shell; do
    # Skip blank lines and comments
    [[ -z "$username" || "$username" =~ ^# ]] && continue

    # Skip system users (UID < 100) — optional, remove if you want all users
    if [ "$uid" -lt 100 ]; then
      log "DEBUG" "  Skipping system user: $username (UID $uid)"
      continue
    fi

    log "DEBUG" "  Checking: $username (home=$home_dir shell=$shell)"
    check_user "$username" "$home_dir" "$shell"

  done < "$PASSWD_FILE"

  log "INFO" "Done — processed all users"
}
```

---

## Step 4 — Write the Report

Write a clean formatted report to a file:

```bash
# ── Write report to file ──────────────────────────────
write_report() {
  mkdir -p "$REPORT_DIR"

  log "INFO" "Writing report to: $REPORT_FILE"

  cat > "$REPORT_FILE" << EOF
════════════════════════════════════════════
  User Activity Report
  Generated : $TIMESTAMP
  Host      : $(hostname)
════════════════════════════════════════════

── Active Users (${#ACTIVE_USERS[@]}) ──────────────────────
$(for user in "${ACTIVE_USERS[@]:-}"; do
  [ -z "$user" ] && continue
  # Get last login — last returns "never logged in" if no history
  last_login=$(last -1 "$user" 2>/dev/null \
    | head -1 \
    | awk '{print $4, $5, $6, $7}' \
    | xargs)
  [ -z "$last_login" ] && last_login="never"
  printf "  %-20s last login: %s\n" "$user" "$last_login"
done)

── No-Login Users (${#NOLOGIN_USERS[@]}) ───────────────────
$(for user in "${NOLOGIN_USERS[@]:-}"; do
  [ -z "$user" ] && continue
  printf "  %s\n" "$user"
done)

── Missing Home Directory (${#NOHOME_USERS[@]}) ────────────
$(for user in "${NOHOME_USERS[@]:-}"; do
  [ -z "$user" ] && continue
  printf "  %s\n" "$user"
done)

════════════════════════════════════════════
  Summary
  Active   : ${#ACTIVE_USERS[@]}
  No-Login : ${#NOLOGIN_USERS[@]}
  No-Home  : ${#NOHOME_USERS[@]}
  Total    : $(( ${#ACTIVE_USERS[@]} + ${#NOLOGIN_USERS[@]} + ${#NOHOME_USERS[@]} ))
════════════════════════════════════════════
EOF

  log "INFO" "Report saved ✓"
}
```

---

## Step 5 — Print the Summary

Print a clean summary to the terminal and exit with the right code:

```bash
# ── Print terminal summary ────────────────────────────
print_summary() {
  local total=$(( ${#ACTIVE_USERS[@]} + \
                  ${#NOLOGIN_USERS[@]} + \
                  ${#NOHOME_USERS[@]} ))

  echo ""
  echo "════════════════════════════════════════"
  echo "  User Report Summary — $(hostname)"
  echo "════════════════════════════════════════"
  printf "  %-20s %s\n" "Total users:"   "$total"
  printf "  %-20s %s\n" "Active:"        "${#ACTIVE_USERS[@]}"
  printf "  %-20s %s\n" "No-login:"      "${#NOLOGIN_USERS[@]}"
  printf "  %-20s %s\n" "Missing home:"  "${#NOHOME_USERS[@]}"
  echo "════════════════════════════════════════"

  if [ ${#NOHOME_USERS[@]} -gt 0 ]; then
    echo ""
    echo "  ⚠  Users with missing home directories:"
    for user in "${NOHOME_USERS[@]}"; do
      echo "     → $user"
    done
    echo ""
  fi

  echo "  Report saved to:"
  echo "  $REPORT_FILE"
  echo "════════════════════════════════════════"
  echo ""
}

# ── Exit code ─────────────────────────────────────────
determine_exit() {
  if [ ${#NOHOME_USERS[@]} -gt 0 ]; then
    log "WARN" "Exiting with code 1 — ${#NOHOME_USERS[@]} user(s) have missing home directories"
    exit 1
  fi
  log "INFO" "All users look healthy — exiting with code 0"
  exit 0
}
```

---

## Full Script

Here is the complete `user-report.sh`:

```bash
#!/bin/bash
# user-report.sh — System user activity reporter
# Usage: ./user-report.sh
# Usage: ./user-report.sh --help

set -euo pipefail

# ── Script directory ───────────────────────────────────
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# ── Config ────────────────────────────────────────────
REPORT_DIR="$SCRIPT_DIR/reports"
DATE=$(date '+%Y-%m-%d')
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
REPORT_FILE="$REPORT_DIR/user-report-${DATE}.txt"
PASSWD_FILE="/etc/passwd"
LOG_LEVEL="${LOG_LEVEL:-INFO}"

# ── Logging ───────────────────────────────────────────
get_level_weight() {
  case $1 in DEBUG) echo 0 ;; INFO) echo 1 ;; WARN) echo 2 ;; ERROR) echo 3 ;; *) echo 1 ;; esac
}

log() {
  local level=$1; shift
  local msg_weight current_weight
  msg_weight=$(get_level_weight "$level")
  current_weight=$(get_level_weight "$LOG_LEVEL")
  [ "$msg_weight" -ge "$current_weight" ] || return 0
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$(printf '%-5s' "$level")] $*"
}

# ── Error handler ─────────────────────────────────────
handle_error() {
  log "ERROR" "Failed at line $1 — command: $BASH_COMMAND"
}
trap 'handle_error $LINENO' ERR

# ── Usage ─────────────────────────────────────────────
usage() {
  cat << EOF

  Usage: $(basename "$0") [options]

  Options:
    -h, --help     Show this message

  Output:
    Terminal  — summary of all users
    File      — full report in $REPORT_DIR

  Example:
    $(basename "$0")

EOF
  exit 0
}

[ "${1:-}" = "-h" ] || [ "${1:-}" = "--help" ] && usage

# ── No-login shells ───────────────────────────────────
NO_LOGIN_SHELLS=(
  "/sbin/nologin"
  "/usr/sbin/nologin"
  "/bin/false"
  "/usr/bin/false"
)

is_nologin_shell() {
  local shell=$1
  for nologin in "${NO_LOGIN_SHELLS[@]}"; do
    [ "$shell" = "$nologin" ] && return 0
  done
  return 1
}

# ── Result arrays ─────────────────────────────────────
ACTIVE_USERS=()
NOLOGIN_USERS=()
NOHOME_USERS=()

# ── Check one user ────────────────────────────────────
check_user() {
  local username=$1
  local home_dir=$2
  local shell=$3

  if is_nologin_shell "$shell"; then
    NOLOGIN_USERS+=("$username")
    return 0
  fi

  if [ ! -d "$home_dir" ]; then
    NOHOME_USERS+=("$username")
    log "WARN" "  $username — home dir missing: $home_dir"
    return 0
  fi

  ACTIVE_USERS+=("$username")
  log "DEBUG" "  $username — active ✓"
}

# ── Process all users ─────────────────────────────────
process_users() {
  log "INFO" "Reading: $PASSWD_FILE"

  [ -f "$PASSWD_FILE" ] || { log "ERROR" "File not found: $PASSWD_FILE"; exit 1; }
  [ -r "$PASSWD_FILE" ] || { log "ERROR" "Cannot read: $PASSWD_FILE";    exit 1; }

  local count
  count=$(wc -l < "$PASSWD_FILE")
  log "INFO" "Found $count entries — processing..."

  while IFS=':' read -r username _ uid _ _ home_dir shell; do
    [[ -z "$username" || "$username" =~ ^# ]] && continue
    [ "$uid" -lt 100 ] 2>/dev/null && {
      log "DEBUG" "  Skipping system user: $username (UID $uid)"
      continue
    }
    check_user "$username" "$home_dir" "$shell"
  done < "$PASSWD_FILE"

  log "INFO" "Processing complete"
}

# ── Write report to file ──────────────────────────────
write_report() {
  mkdir -p "$REPORT_DIR"
  log "INFO" "Writing report: $REPORT_FILE"

  {
    echo "════════════════════════════════════════════"
    echo "  User Activity Report"
    echo "  Generated : $TIMESTAMP"
    echo "  Host      : $(hostname)"
    echo "════════════════════════════════════════════"
    echo ""

    echo "── Active Users (${#ACTIVE_USERS[@]}) ──────────────────────"
    for user in "${ACTIVE_USERS[@]:-}"; do
      [ -z "$user" ] && continue
      last_login=$(last -1 "$user" 2>/dev/null \
        | head -1 \
        | awk '{print $4, $5, $6, $7}' \
        | xargs 2>/dev/null || echo "never")
      [ -z "${last_login// /}" ] && last_login="never"
      printf "  %-20s last login: %s\n" "$user" "$last_login"
    done

    echo ""
    echo "── No-Login Users (${#NOLOGIN_USERS[@]}) ───────────────────"
    for user in "${NOLOGIN_USERS[@]:-}"; do
      [ -z "$user" ] && continue
      printf "  %s\n" "$user"
    done

    echo ""
    echo "── Missing Home Directory (${#NOHOME_USERS[@]}) ────────────"
    for user in "${NOHOME_USERS[@]:-}"; do
      [ -z "$user" ] && continue
      printf "  %s\n" "$user"
    done

    echo ""
    echo "════════════════════════════════════════════"
    echo "  Summary"
    printf "  %-12s %s\n" "Active:"   "${#ACTIVE_USERS[@]}"
    printf "  %-12s %s\n" "No-Login:" "${#NOLOGIN_USERS[@]}"
    printf "  %-12s %s\n" "No-Home:"  "${#NOHOME_USERS[@]}"
    printf "  %-12s %s\n" "Total:" \
      "$(( ${#ACTIVE_USERS[@]} + ${#NOLOGIN_USERS[@]} + ${#NOHOME_USERS[@]} ))"
    echo "════════════════════════════════════════════"

  } > "$REPORT_FILE"

  log "INFO" "Report saved ✓"
}

# ── Terminal summary ──────────────────────────────────
print_summary() {
  local total
  total=$(( ${#ACTIVE_USERS[@]} + ${#NOLOGIN_USERS[@]} + ${#NOHOME_USERS[@]} ))

  echo ""
  echo "════════════════════════════════════════"
  echo "  User Report — $(hostname)"
  echo "════════════════════════════════════════"
  printf "  %-20s %s\n" "Total users:"  "$total"
  printf "  %-20s %s\n" "Active:"       "${#ACTIVE_USERS[@]}"
  printf "  %-20s %s\n" "No-login:"     "${#NOLOGIN_USERS[@]}"
  printf "  %-20s %s\n" "Missing home:" "${#NOHOME_USERS[@]}"
  echo "════════════════════════════════════════"

  if [ ${#ACTIVE_USERS[@]} -gt 0 ]; then
    echo ""
    echo "  Active users:"
    for user in "${ACTIVE_USERS[@]}"; do
      echo "    → $user"
    done
  fi

  if [ ${#NOHOME_USERS[@]} -gt 0 ]; then
    echo ""
    echo "  ⚠  Missing home directories:"
    for user in "${NOHOME_USERS[@]}"; do
      echo "    → $user"
    done
  fi

  echo ""
  echo "  Full report: $REPORT_FILE"
  echo "════════════════════════════════════════"
  echo ""
}

# ── Main ──────────────────────────────────────────────
log "INFO" "Starting user activity report"

process_users
write_report
print_summary

# Exit with 1 if any users have missing home directories
if [ ${#NOHOME_USERS[@]} -gt 0 ]; then
  log "WARN" "Exiting with code 1 — ${#NOHOME_USERS[@]} user(s) have missing homes"
  exit 1
fi

log "INFO" "All users look healthy"
exit 0
```

---

## Sample Output

```bash
./user-report.sh
```

```
[2024-06-12 09:15:32] [INFO ] Starting user activity report
[2024-06-12 09:15:32] [INFO ] Reading: /etc/passwd
[2024-06-12 09:15:32] [INFO ] Found 42 entries — processing...
[2024-06-12 09:15:32] [WARN ] john — home dir missing: /home/john
[2024-06-12 09:15:32] [INFO ] Processing complete
[2024-06-12 09:15:32] [INFO ] Writing report: reports/user-report-2024-06-12.txt
[2024-06-12 09:15:32] [INFO ] Report saved ✓

════════════════════════════════════════
  User Report — prod-server-01
════════════════════════════════════════
  Total users:         5
  Active:              3
  No-login:            1
  Missing home:        1
════════════════════════════════════════

  Active users:
    → ubuntu
    → deploy
    → ashraf

  ⚠  Missing home directories:
    → john

  Full report: reports/user-report-2024-06-12.txt
════════════════════════════════════════

[2024-06-12 09:15:32] [WARN ] Exiting with code 1 — 1 user(s) have missing homes

# Check exit code
echo $?
# 1
```

---

## Debug Run

```bash
LOG_LEVEL=DEBUG ./user-report.sh
```

```
[2024-06-12 09:15:32] [INFO ] Starting user activity report
[2024-06-12 09:15:32] [INFO ] Reading: /etc/passwd
[2024-06-12 09:15:32] [DEBUG] Skipping system user: root (UID 0)
[2024-06-12 09:15:32] [DEBUG] Skipping system user: daemon (UID 1)
[2024-06-12 09:15:32] [DEBUG] ubuntu — active ✓
[2024-06-12 09:15:32] [DEBUG] deploy — active ✓
[2024-06-12 09:15:32] [WARN ] john — home dir missing: /home/john
...
```

---

## Challenge Extensions

Once the base script works, try these to go further:

**Easy:**
- Add a `--show-nologin` flag to include no-login users in the terminal summary
- Count how many users have `/bin/bash` as their shell vs other shells
- Show the total number of system users (UID < 100) that were skipped

**Medium:**
- Add a check for users whose password has not been changed in over 90 days using `chage -l`
- Sort active users alphabetically in the report
- Add a `--user <name>` flag to check just one specific user

**Hard:**
- Compare today's report with yesterday's — flag any new users that appeared
- Add a check for users who have not logged in for more than 30 days
- Export the report as CSV instead of plain text using a `--csv` flag

---

## Skills Used in This Project

| Skill | Where Used |
|---|---|
| `set -euo pipefail` | Line 2 — always |
| `trap ERR` | Catch and log failures with line number |
| Reading files line by line | `while IFS=':' read -r ...` on `/etc/passwd` |
| Field splitting with IFS | Split the 7 passwd fields cleanly |
| Arrays | `ACTIVE_USERS`, `NOLOGIN_USERS`, `NOHOME_USERS` |
| Loop over arrays | Print each category in summary and report |
| String manipulation | `[ -z "$user" ]`, shell comparison |
| Command substitution | `$(hostname)`, `$(date)`, `$(wc -l)`, `$(last ...)` |
| Pipes | `last -1 | head -1 | awk | xargs` |
| Writing to files | Heredoc + `{} > file` block |
| Exit codes | `exit 0` healthy, `exit 1` missing homes found |
| Usage message | `-h / --help` flag |
| `LOG_LEVEL` | `DEBUG=1` without changing script |
| `SCRIPT_DIR` | Portable paths regardless of where called from |