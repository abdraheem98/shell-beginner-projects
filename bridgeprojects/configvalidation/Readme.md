# Bridge 2 — Config Validation

> **Session:** Bridge — Intermediate to Advanced  
> **Duration:** ~20 minutes  
> **Prerequisite:** Bridge 1 (Better Logging) complete  
> **File to upgrade:** `server-monitor/monitor.sh` + `config/monitor.conf`

---

## Table of Contents

- [What We Are Improving](#what-we-are-improving)
- [The Problem With No Validation](#the-problem-with-no-validation)
- [Upgrade 1 — Validate Required Values](#upgrade-1--validate-required-values)
- [Upgrade 2 — Validate Value Formats](#upgrade-2--validate-value-formats)
- [Upgrade 3 — Print Active Config Summary](#upgrade-3--print-active-config-summary)
- [Upgrade 4 — Separate Config from Script](#upgrade-4--separate-config-from-script)
- [Final Upgraded monitor.sh Sections](#final-upgraded-monitorsh-sections)
- [Before vs After](#before-vs-after)
- [Key Takeaways](#key-takeaways)

---

## What We Are Improving

After loading `monitor.conf`, the current script immediately
starts running checks — without verifying anything is correct.

```bash
# Current flow — no validation gate
load_config    # reads the file
run_checks     # immediately starts checking — no gate
```

This means:
- A typo in the config causes a confusing crash halfway through
- Missing values are only discovered when the check fails
- Nobody knows what config values were active when something went wrong

We are going to add a **validation gate** between loading
and running — so the script fails clearly at startup
instead of silently mid-way through.

---

## The Problem With No Validation

### Problem 1 — Typo causes cryptic error mid-run

```bash
# monitor.conf — typo in disk path
DISK:/opt/myaap,85    # typo — should be myapp
```

**Current output — confusing:**
```
[INFO ] Disk:
df: /opt/myaap: No such file or directory
```

> Script crashes mid-run with a system error message —
> not your message, not your format, no context.

---

### Problem 2 — Invalid threshold silently ignored

```bash
# monitor.conf — invalid threshold
DISK:/opt,abc    # abc is not a number
```

**Current output:**
```
[INFO ] Disk:
[ERROR] / — % used (threshold: abc%) ✗
```

> Script runs with garbage values — comparison fails silently,
> check always reports failure even when disk is fine.

---

### Problem 3 — No record of what config was used

When an alert fires at 3AM, the on-call engineer asks:
*"What thresholds were configured when this ran?"*

With the current script — they have to open the config file manually.
With a config summary in the log — it is right there.

---

## Upgrade 1 — Validate Required Values

Add a `validate_config()` function that runs after loading
the config and before any checks:

```bash
# Add to monitor.sh — after the config loading section

validate_config() {
  log "INFO" "Validating configuration..."
  local errors=0

  # At least one service must be defined
  if [ ${#SERVICES[@]} -eq 0 ] && \
     [ ${#DISK_CHECKS[@]} -eq 0 ] && \
     [ -z "$CHECK_PORT" ]; then
    log "ERROR" "Config has no checks defined"
    log "ERROR" "Add at least one SERVICE:, DISK:, or PORT: line to monitor.conf"
    errors=$(( errors + 1 ))
  fi

  # Validate each service name is not empty
  for service in "${SERVICES[@]}"; do
    if [ -z "$service" ]; then
      log "ERROR" "Empty service name found in config"
      errors=$(( errors + 1 ))
    fi
  done

  # Validate each disk entry
  for entry in "${DISK_CHECKS[@]}"; do
    local path threshold
    IFS=':' read -r path threshold <<< "$entry"

    # Path must not be empty
    if [ -z "$path" ]; then
      log "ERROR" "Empty disk path found in config"
      errors=$(( errors + 1 ))
      continue
    fi

    # Threshold must be a number
    if [[ ! "$threshold" =~ ^[0-9]+$ ]]; then
      log "ERROR" "Invalid threshold '$threshold' for path '$path'"
      log "ERROR" "Threshold must be a number between 1 and 100"
      errors=$(( errors + 1 ))
      continue
    fi

    # Threshold must be 1-100
    if [ "$threshold" -lt 1 ] || [ "$threshold" -gt 100 ]; then
      log "ERROR" "Threshold '$threshold' for '$path' is out of range (1-100)"
      errors=$(( errors + 1 ))
    fi
  done

  # Validate port if set
  if [ -n "$CHECK_PORT" ]; then
    if [[ ! "$CHECK_PORT" =~ ^[0-9]+$ ]]; then
      log "ERROR" "Invalid port '$CHECK_PORT' — must be a number"
      errors=$(( errors + 1 ))
    elif [ "$CHECK_PORT" -lt 1 ] || [ "$CHECK_PORT" -gt 65535 ]; then
      log "ERROR" "Port '$CHECK_PORT' is out of range (1-65535)"
      errors=$(( errors + 1 ))
    fi
  fi

  # Fail if any errors found
  if [ "$errors" -gt 0 ]; then
    log "ERROR" "Config validation failed — $errors error(s) found"
    log "ERROR" "Fix $CONFIG_FILE and try again"
    exit 1
  fi

  log "INFO" "Config validation passed ✓"
}
```

**Test it — break the config:**

```bash
# Edit monitor.conf — add a bad line
echo "DISK:/opt,abc" >> config/monitor.conf

# Run monitor
./monitor.sh
```

```
[INFO ] Loading config: config/monitor.conf
[ERROR] Invalid threshold 'abc' for path '/opt'
[ERROR] Threshold must be a number between 1 and 100
[ERROR] Config validation failed — 1 error(s) found
[ERROR] Fix config/monitor.conf and try again
```

> Script stops immediately — before touching anything.
> Error message is clear and actionable.

---

## Upgrade 2 — Validate Value Formats

Add format validation for the email field
and check that disk paths actually exist:

```bash
# Add inside validate_config() — after existing checks

  # Validate email format if set
  if [ -n "$ALERT_EMAIL" ]; then
    if [[ ! "$ALERT_EMAIL" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
      log "ERROR" "Invalid email address: $ALERT_EMAIL"
      errors=$(( errors + 1 ))
    fi
  fi

  # Validate disk paths exist on this server
  for entry in "${DISK_CHECKS[@]}"; do
    local path threshold
    IFS=':' read -r path threshold <<< "$entry"

    if [ ! -d "$path" ]; then
      log "WARN" "Disk path does not exist on this server: $path"
      log "WARN" "Check will be skipped at runtime"
    fi
  done
```

**Test with bad email:**

```bash
# Edit monitor.conf
# ALERT_EMAIL:not-an-email

./monitor.sh
```

```
[ERROR] Invalid email address: not-an-email
[ERROR] Config validation failed — 1 error(s) found
```

**Test with missing path:**

```bash
# Edit monitor.conf
# DISK:/nonexistent,80

./monitor.sh
```

```
[WARN ] Disk path does not exist on this server: /nonexistent
[WARN ] Check will be skipped at runtime
[INFO ] Config validation passed ✓
```

> Paths that do not exist are a warning — not a hard error.
> Maybe the mount is not attached yet.
> The script warns and continues.

---

## Upgrade 3 — Print Active Config Summary

After validation passes, print exactly what config is active.
This goes into the log — so when something goes wrong,
the on-call engineer sees exactly what was configured:

```bash
# Add to monitor.sh — after validate_config()

print_config_summary() {
  separator
  log "INFO" "Active Configuration"
  separator

  log "INFO" "Config file  : $CONFIG_FILE"
  log "INFO" "Log file     : $LOG_FILE"
  log "INFO" "Log level    : $LOG_LEVEL"
  log "INFO" "Report dir   : $REPORT_DIR"
  log "INFO" ""

  log "INFO" "Services to check (${#SERVICES[@]}):"
  for service in "${SERVICES[@]}"; do
    log "INFO" "  → $service"
  done

  log "INFO" ""
  log "INFO" "Disk paths to check (${#DISK_CHECKS[@]}):"
  for entry in "${DISK_CHECKS[@]}"; do
    local path threshold
    IFS=':' read -r path threshold <<< "$entry"
    log "INFO" "  → $path (alert at ${threshold}%)"
  done

  if [ -n "$CHECK_PORT" ]; then
    log "INFO" ""
    log "INFO" "Port to check : $CHECK_PORT"
  fi

  if [ -n "$ALERT_EMAIL" ]; then
    log "INFO" "Alert email   : $ALERT_EMAIL"
  fi

  separator
}
```

**Output when script starts:**

```
────────────────────────────────────────
[INFO ] Active Configuration
────────────────────────────────────────
[INFO ] Config file  : /opt/server-monitor/config/monitor.conf
[INFO ] Log file     : /opt/server-monitor/reports/monitor.log
[INFO ] Log level    : INFO
[INFO ] Report dir   : /opt/server-monitor/reports
[INFO ]
[INFO ] Services to check (3):
[INFO ]   → nginx
[INFO ]   → ssh
[INFO ]   → cron
[INFO ]
[INFO ] Disk paths to check (3):
[INFO ]   → / (alert at 90%)
[INFO ]   → /tmp (alert at 80%)
[INFO ]   → /var/log (alert at 85%)
[INFO ]
[INFO ] Port to check : 3000
[INFO ] Alert email   : ops@company.com
────────────────────────────────────────
```

> Every run tells you exactly what it is checking and at what thresholds.
> No more guessing what the config was when an alert fired.

---

## Upgrade 4 — Separate Config from Script

Right now some values are still hardcoded in `monitor.sh`:

```bash
# Still hardcoded in monitor.sh
MAX_LOG_SIZE_MB=50
WARN_LOG_SIZE_MB=40
REPORT_DIR="$SCRIPT_DIR/reports"
```

Move them into `monitor.conf` as well:

```bash
# Add to config/monitor.conf

# ── Log settings ──────────────────────────────────────
MAX_LOG_MB:50
WARN_LOG_MB:40

# ── Report settings ───────────────────────────────────
REPORT_DIR:/opt/server-monitor/reports
```

**Update the config loading section in `monitor.sh`:**

```bash
# Add these to the while read loop in load_config

  elif [[ "$line" =~ ^MAX_LOG_MB:([0-9]+)$ ]]; then
    MAX_LOG_SIZE_MB="${BASH_REMATCH[1]}"

  elif [[ "$line" =~ ^WARN_LOG_MB:([0-9]+)$ ]]; then
    WARN_LOG_SIZE_MB="${BASH_REMATCH[1]}"

  elif [[ "$line" =~ ^REPORT_DIR:(.+)$ ]]; then
    REPORT_DIR="${BASH_REMATCH[1]}"
```

**Add validation for the new fields:**

```bash
# Add inside validate_config()

  # MAX_LOG_MB must be a number
  if [[ ! "$MAX_LOG_SIZE_MB" =~ ^[0-9]+$ ]]; then
    log "ERROR" "Invalid MAX_LOG_MB: $MAX_LOG_SIZE_MB — must be a number"
    errors=$(( errors + 1 ))
  fi

  # WARN must be less than MAX
  if [ "$WARN_LOG_SIZE_MB" -ge "$MAX_LOG_SIZE_MB" ]; then
    log "ERROR" "WARN_LOG_MB ($WARN_LOG_SIZE_MB) must be less than MAX_LOG_MB ($MAX_LOG_SIZE_MB)"
    errors=$(( errors + 1 ))
  fi
```

> Now ops can change log rotation thresholds by editing the config file.
> No script changes needed.

---

## Final Upgraded `monitor.sh` Sections

Here are the exact sections to update in `monitor.sh`.
Replace or add each one in order:

### After `load_config` — add validation and summary

```bash
# ── Validate config ───────────────────────────────────
validate_config() {
  log "INFO" "Validating configuration..."
  local errors=0

  # Must have at least one check
  if [ ${#SERVICES[@]} -eq 0 ] && \
     [ ${#DISK_CHECKS[@]} -eq 0 ] && \
     [ -z "$CHECK_PORT" ]; then
    log "ERROR" "No checks defined in config — add SERVICE:, DISK:, or PORT:"
    errors=$(( errors + 1 ))
  fi

  # Validate disk entries
  for entry in "${DISK_CHECKS[@]}"; do
    local path threshold
    IFS=':' read -r path threshold <<< "$entry"

    [ -z "$path" ] && {
      log "ERROR" "Empty disk path in config"
      errors=$(( errors + 1 ))
      continue
    }

    [[ "$threshold" =~ ^[0-9]+$ ]] || {
      log "ERROR" "Invalid threshold '$threshold' for '$path' — must be a number"
      errors=$(( errors + 1 ))
      continue
    }

    { [ "$threshold" -ge 1 ] && [ "$threshold" -le 100 ]; } || {
      log "ERROR" "Threshold '$threshold' for '$path' must be 1-100"
      errors=$(( errors + 1 ))
    }

    [ -d "$path" ] || \
      log "WARN" "Disk path not found on this server: $path (check will be skipped)"
  done

  # Validate port
  if [ -n "$CHECK_PORT" ]; then
    [[ "$CHECK_PORT" =~ ^[0-9]+$ ]] || {
      log "ERROR" "Invalid port '$CHECK_PORT' — must be a number"
      errors=$(( errors + 1 ))
    }
    { [ "$CHECK_PORT" -ge 1 ] && [ "$CHECK_PORT" -le 65535 ]; } 2>/dev/null || {
      log "ERROR" "Port '$CHECK_PORT' out of range (1-65535)"
      errors=$(( errors + 1 ))
    }
  fi

  # Validate email
  if [ -n "$ALERT_EMAIL" ]; then
    [[ "$ALERT_EMAIL" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]] || {
      log "ERROR" "Invalid email: $ALERT_EMAIL"
      errors=$(( errors + 1 ))
    }
  fi

  if [ "$errors" -gt 0 ]; then
    log "ERROR" "Config validation failed — $errors error(s)"
    log "ERROR" "Fix $CONFIG_FILE and try again"
    exit 1
  fi

  log "INFO" "Config validation passed ✓"
}

# ── Config summary ────────────────────────────────────
print_config_summary() {
  separator
  log "INFO" "Active Configuration"
  separator
  log "INFO" "Config file : $CONFIG_FILE"
  log "INFO" "Log level   : $LOG_LEVEL"
  log "INFO" ""
  log "INFO" "Services (${#SERVICES[@]}): ${SERVICES[*]:-none}"
  log "INFO" "Disk checks (${#DISK_CHECKS[@]}):"
  for entry in "${DISK_CHECKS[@]}"; do
    IFS=':' read -r path threshold <<< "$entry"
    log "INFO" "  $path — alert at ${threshold}%"
  done
  [ -n "$CHECK_PORT"   ] && log "INFO" "Port        : $CHECK_PORT"
  [ -n "$ALERT_EMAIL"  ] && log "INFO" "Alert email : $ALERT_EMAIL"
  separator
}
```

### Updated main section

```bash
# ── Main ──────────────────────────────────────────────
mkdir -p "$REPORT_DIR"

require_binary "df"
require_binary "ss"
require_binary "systemctl"

warn_log_size           # from Bridge 1
validate_config         # new in Bridge 2
print_config_summary    # new in Bridge 2

run_checks
generate_report
```

---

## Before vs After

### Before — no validation gate

```
load_config
    ↓
run_checks     ← crashes here with system error if config is wrong
```

```
[INFO ] Disk:
df: /opt/myaap: No such file or directory
```

### After — validation gate in place

```
load_config
    ↓
validate_config   ← stops here with clear error if anything is wrong
    ↓
print_config_summary   ← logs what is active
    ↓
run_checks
```

```
[INFO ] Loading config: config/monitor.conf
[ERROR] Invalid threshold 'abc' for path '/opt/myaap'
[ERROR] Config validation failed — 1 error(s)
[ERROR] Fix config/monitor.conf and try again
```

---

## Key Takeaways

| What Changed | Why It Matters |
|---|---|
| `validate_config()` runs before checks | Script fails early and clearly — not mid-run |
| Format checks on every value | No garbage values reach the check functions |
| Path existence check | Warn before the disk check fails at runtime |
| `print_config_summary()` | Always know what was configured when something happened |
| Config values from file | Nothing hardcoded — ops changes config, not code |
| Error counter pattern | Collect ALL errors before stopping — not just the first |

---

## The Error Counter Pattern — Why It Matters

Notice this pattern in `validate_config()`:

```bash
local errors=0

# Check 1
[ -z "$path" ] && {
  log "ERROR" "..."
  errors=$(( errors + 1 ))    # count it, don't stop yet
}

# Check 2
[[ ! "$threshold" =~ ^[0-9]+$ ]] && {
  log "ERROR" "..."
  errors=$(( errors + 1 ))    # count it, don't stop yet
}

# Check ALL then stop
if [ "$errors" -gt 0 ]; then
  exit 1    # stop here after showing ALL errors
fi
```

**Why this matters:**

```bash
# Bad pattern — stops at first error
# User fixes it → runs again → hits second error → fixes it → hits third...
# Takes 5 runs to see all the problems

# Good pattern — shows ALL errors at once
# User sees everything wrong → fixes them all → one successful run
```

> Show all errors at once. Never make someone run the script
> five times to discover five separate problems.

---

## Connection to the Advanced Series

This upgrade is exactly **Concept 4 — Config Management**
from the advanced series.

When you get there, you will recognize every piece:

```
Bridge 2                        Advanced Concept 4
─────────────────────────────────────────────────────
validate_config()           →   Same function, same pattern
Error counter               →   Same collect-then-fail approach
print_config_summary()      →   Same active config print block
Config values from file     →   Same separation of data and logic
Format validation with =~   →   Same regex patterns
```

> The advanced version adds environment-specific config files
> (dev.env, staging.env, prod.env) — but the validation
> and summary patterns are identical to what you just built.

---

