# Intermediate Topic 1 — Exit Codes & `$?`

> **Session:** Shell Scripting for DevOps — Intermediate Concepts  
> **Duration:** ~20 minutes  
> **Prerequisite:** Basics completed (variables, loops, functions, grep/awk/sed)

---

## Table of Contents

- [What Is an Exit Code?](#what-is-an-exit-code)
- [Why DevOps Engineers Need This](#why-devops-engineers-need-this)
- [How to Read an Exit Code](#how-to-read-an-exit-code)
- [Common Exit Codes](#common-exit-codes)
- [Using Exit Codes in Scripts](#using-exit-codes-in-scripts)
- [The exit Command](#the-exit-command)
- [Common Mistakes](#common-mistakes)
- [Quick Exercise](#quick-exercise)

---

## What Is an Exit Code?

Every command you run in Linux — whether it succeeds or fails —
returns a number when it finishes. That number is called an **exit code**.

Think of it like a report card:

> **0 = Pass (Success)**  
> **Non-zero = Fail (Something went wrong)**

That's it. Simple rule. Always remember it.

---

## Why DevOps Engineers Need This

Without exit codes, your script has no way to know if something worked.

**Real example:**

> You write a script that copies a file and then restarts a service.  
> The file copy fails silently.  
> The script restarts the service anyway — with missing files.  
> The service crashes.  
> You have no idea why.

Exit codes let you **catch the failure** at the copy step and
stop the script before it causes more damage.

---

## How to Read an Exit Code

The special variable `$?` always holds the exit code of the
**last command that ran**.

```bash
# Example 1 — Successful command
ls /tmp
echo "Exit code: $?"
```

**Output:**
```
/tmp
Exit code: 0
```

```bash
# Example 2 — Failed command
ls /this-folder-does-not-exist
echo "Exit code: $?"
```

**Output:**
```
ls: cannot access '/this-folder-does-not-exist': No such file or directory
Exit code: 2
```

```bash
# Example 3 — Always read $? immediately after the command
ls /tmp
echo "Hello"          # this runs BEFORE you read $?
echo "Exit code: $?"  # shows exit code of echo, not ls!
```

> **Important:** Read `$?` immediately after the command.
> Every new command overwrites `$?` with its own exit code.

**Correct way:**
```bash
ls /tmp
EXIT_CODE=$?          # save it immediately
echo "Hello"
echo "Exit code was: $EXIT_CODE"   # safe to use now
```

---

## Common Exit Codes

| Code | Meaning | Example |
|---|---|---|
| `0` | Success | Command worked fine |
| `1` | General error | Script logic failed |
| `2` | Misuse of command | Wrong flags or arguments |
| `126` | Permission denied | Script not executable |
| `127` | Command not found | Typo in command name |
| `130` | Script cancelled | User pressed Ctrl+C |

**See them in action:**

```bash
# Exit code 0 — success
echo "hello"
echo $?         # 0

# Exit code 1 — general error
grep "xyz" /etc/passwd
echo $?         # 1 — pattern not found

# Exit code 127 — command not found
blahblah
echo $?         # 127

# Exit code 126 — not executable
bash /etc/hosts
echo $?         # 126
```

---

## Using Exit Codes in Scripts

### Check if a command succeeded

```bash
#!/bin/bash

# Method 1 — using $?
cp important-file.txt /backup/
if [ $? -eq 0 ]; then
  echo "Copy succeeded"
else
  echo "Copy FAILED"
fi
```

```bash
#!/bin/bash

# Method 2 — shorter, more common in real scripts
if cp important-file.txt /backup/; then
  echo "Copy succeeded"
else
  echo "Copy FAILED"
fi
```

> Method 2 is cleaner — use this style in real scripts.

---

### Stop the script when something fails

```bash
#!/bin/bash

echo "Step 1: Copying file..."
cp app.tar.gz /opt/myapp/ || exit 1

echo "Step 2: Extracting..."
tar -xzf /opt/myapp/app.tar.gz -C /opt/myapp/ || exit 1

echo "Step 3: Restarting service..."
systemctl restart myapp || exit 1

echo "All steps completed successfully!"
```

**What `|| exit 1` means:**
- `||` = "OR" — only runs the right side if the left side FAILS
- If `cp` succeeds — skip `exit 1`, continue
- If `cp` fails — run `exit 1`, stop the script

---

### Check a specific exit code

```bash
#!/bin/bash

ping -c 1 google.com
CODE=$?

if [ $CODE -eq 0 ]; then
  echo "Internet is working"
elif [ $CODE -eq 1 ]; then
  echo "Host unreachable"
elif [ $CODE -eq 2 ]; then
  echo "Unknown host — check DNS"
else
  echo "Something else went wrong — code: $CODE"
fi
```

---

### Use exit codes in functions

```bash
#!/bin/bash

check_file() {
  local file=$1

  if [ -f "$file" ]; then
    echo "File exists: $file"
    return 0    # success
  else
    echo "File NOT found: $file"
    return 1    # failure
  fi
}

# Call the function and check its exit code
check_file "/etc/passwd"
if [ $? -eq 0 ]; then
  echo "Proceeding..."
fi

check_file "/etc/does-not-exist"
if [ $? -ne 0 ]; then
  echo "Cannot proceed — file missing"
  exit 1
fi
```

> Inside functions use `return` not `exit`.
> `exit` would close the whole script.
> `return` only exits the function.

---

## The `exit` Command

Use `exit` to stop your script and send an exit code back to whoever called it:

```bash
#!/bin/bash

# exit 0 — tell the caller everything worked
exit 0

# exit 1 — tell the caller something failed
exit 1

# exit with any number 0-255
exit 42
```

**Why this matters for CI/CD pipelines:**

```
GitHub Actions / Jenkins calls your script
         |
         v
   script exits 0  ->  pipeline step shows GREEN  (continue)
   script exits 1  ->  pipeline step shows RED    (stop)
```

If your script always exits 0 — even when it fails — the pipeline
will never catch your errors. Always `exit 1` on failure.

---

## Common Mistakes

### Mistake 1 — Not reading `$?` immediately

```bash
# Wrong
ls /nonexistent
echo "some other command"
echo $?    # shows exit code of echo, not ls

# Correct
ls /nonexistent
CODE=$?    # save immediately
echo "some other command"
echo $CODE
```

---

### Mistake 2 — Using `exit` inside a function

```bash
# Wrong — closes the whole script
check_disk() {
  if [ ! -d "/opt/myapp" ]; then
    exit 1    # kills everything
  fi
}

# Correct — only exits the function
check_disk() {
  if [ ! -d "/opt/myapp" ]; then
    return 1
  fi
}
```

---

### Mistake 3 — Ignoring exit codes completely

```bash
# Wrong — no checks, runs blindly
cp app.tar.gz /opt/myapp/
tar -xzf /opt/myapp/app.tar.gz
systemctl restart myapp
echo "Done!"    # prints Done even if everything above failed

# Correct — stops on first failure
cp app.tar.gz /opt/myapp/       || exit 1
tar -xzf /opt/myapp/app.tar.gz  || exit 1
systemctl restart myapp         || exit 1
echo "Done!"
```

---

### Mistake 4 — Confusing return code with printed output

```bash
# A function can both PRINT something and return an exit code
# These are two different things

get_status() {
  echo "active"    # this is printed output
  return 0         # this is the exit code
}

# Capture the output
output=$(get_status)
echo "Output : $output"   # active
echo "Code   : $?"        # 0
```

---

## Quick Exercise

> **Task:** Write a script called `check.sh` that:
>
> 1. Checks if the file `/etc/passwd` exists
>    - If yes — print `"passwd file found"` and continue
>    - If no  — print `"passwd file missing"` and exit with code `1`
>
> 2. Checks if the command `curl` is installed
>    - Use `command -v curl` to check
>    - If found — print `"curl is available"`
>    - If not   — print `"curl is missing"` and exit with code `1`
>
> 3. Ping `google.com` once
>    - If it works — print `"Network OK"`
>    - If it fails — print `"Network FAILED"` and exit with code `1`
>
> 4. At the end print `"All checks passed!"` and exit with code `0`

**Expected output (when everything works):**
```
passwd file found
curl is available
Network OK
All checks passed!
```

**Expected solution:**

```bash
#!/bin/bash

# Check 1 — file exists
if [ -f "/etc/passwd" ]; then
  echo "passwd file found"
else
  echo "passwd file missing"
  exit 1
fi

# Check 2 — curl installed
if command -v curl > /dev/null 2>&1; then
  echo "curl is available"
else
  echo "curl is missing"
  exit 1
fi

# Check 3 — network
if ping -c 1 google.com > /dev/null 2>&1; then
  echo "Network OK"
else
  echo "Network FAILED"
  exit 1
fi

echo "All checks passed!"
exit 0
```

**Run it:**
```bash
chmod +x check.sh
./check.sh
echo "Script exit code: $?"
```

---

## Summary

| Concept | Remember |
|---|---|
| Exit code `0` | Always means success |
| Exit code non-zero | Always means failure |
| `$?` | Holds exit code of the last command |
| Save `$?` immediately | It gets overwritten by the next command |
| `return` in functions | Only exits the function |
| `exit` in scripts | Exits the whole script |
| `command \|\| exit 1` | Stop the script if command fails |
| CI/CD pipelines | Non-zero exit = red build |

---

## What's Next

**Intermediate Topic 2 — String Manipulation**  
Learn how to trim, extract, replace, and split strings entirely
inside bash — no grep, no awk, just built-in bash syntax.