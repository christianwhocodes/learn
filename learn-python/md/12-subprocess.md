# Python `subprocess` Module Guide

## Table of Contents

1. [Overview](#1-overview)
2. [Getting Started with subprocess.run()](#2-getting-started-with-subprocessrun)
   - 2.1 [The Main Function](#21-the-main-function)
   - 2.2 [Basic Example](#22-basic-example)
   - 2.3 [Why Use a List of Arguments?](#23-why-use-a-list-of-arguments)
3. [Essential Parameters](#3-essential-parameters)
   - 3.1 [Core Parameters Reference](#31-core-parameters-reference)
   - 3.2 [Understanding text=True vs text=False](#32-understanding-texttrue-vs-textfalse)
   - 3.3 [When to Use Each Mode](#33-when-to-use-each-mode)
4. [Handling Success and Failure](#4-handling-success-and-failure)
   - 4.1 [Pattern A: Manual Return Code Inspection](#41-pattern-a-manual-return-code-inspection)
   - 4.2 [Pattern B: Exception-Based Error Handling](#42-pattern-b-exception-based-error-handling)
   - 4.3 [Choosing the Right Pattern](#43-choosing-the-right-pattern)
5. [Exception Handling](#5-exception-handling)
   - 5.1 [Common subprocess Exceptions](#51-common-subprocess-exceptions)
   - 5.2 [Comprehensive Exception Handling Example](#52-comprehensive-exception-handling-example)
6. [Best Practices](#6-best-practices)
   - 6.1 [Security and Safety](#61-security-and-safety)
   - 6.2 [Output Handling](#62-output-handling)
   - 6.3 [Error Management](#63-error-management)
   - 6.4 [Timeout Handling](#64-timeout-handling)
   - 6.5 [Exception Handling](#65-exception-handling)
   - 6.6 [Compatibility](#66-compatibility)
7. [Real-World Example](#7-real-world-example)
   - 7.1 [System Info Report Generator](#71-system-info-report-generator)
   - 7.2 [What This Example Demonstrates](#72-what-this-example-demonstrates)
8. [Quick Reference](#8-quick-reference)
9. [Further Reading](#9-further-reading)

---

## 1. Overview

The `subprocess` module lets your Python program communicate with other programs (like `git`, `ls`, or custom tools). Think of it as calling a friend to do a job and listening to what they say back.

---

## 2. Getting Started with subprocess.run()

### 2.1 The Main Function

`subprocess.run(...)` is your primary tool for running external commands:

- Commands are passed as a list: `["prog", "arg1", "arg2"]`
- Returns a `CompletedProcess` object containing:
  - `.returncode` — exit status (0 typically means success)
  - `.stdout` — captured standard output (if requested)
  - `.stderr` — captured error output (if requested)

### 2.2 Basic Example

Run `python --version` and capture the output:

```python
import subprocess
import sys

cp = subprocess.run([sys.executable, "--version"], capture_output=True, text=True)

print("Return Code:", cp.returncode)  # Should be 0 for success
print("Standard Output:", cp.stdout)  # Should print "Python X.Y.Z\n"
print("Standard Error:", cp.stderr)  # Usually empty
```

**Output:**

```
Return Code: 0
Standard Output: Python 3.14.0

Standard Error:
```

### 2.3 Why Use a List of Arguments?

Using a list is safer and avoids shell parsing surprises. Avoid `shell=True` unless absolutely necessary (e.g., Windows shell built-ins like `cd`).

```python
import subprocess

# Using shell=True with a string command
cp = subprocess.run(["cd /somepath"], shell=True, capture_output=True, text=True)

print("Return Code:", cp.returncode)
print("Standard Output:", cp.stdout)
print("Standard Error:", cp.stderr)
```

**Output:**

```
Return Code: 1
Standard Output:
Standard Error: The system cannot find the path specified.
```

---

## 3. Essential Parameters

### 3.1 Core Parameters Reference

| Parameter        | Default    | Description                                        |
| ---------------- | ---------- | -------------------------------------------------- |
| `args`           | (required) | Command to execute as a list: `["git", "status"]`  |
| `check`          | `False`    | Raises `CalledProcessError` on non-zero exit code  |
| `capture_output` | `False`    | Captures both `stdout` and `stderr`                |
| `text`           | `False`    | Returns output as strings instead of bytes         |
| `timeout`        | `None`     | Maximum seconds to wait (raises `TimeoutExpired`)  |
| `encoding`       | `None`     | Character encoding for text mode (e.g., `"utf-8"`) |

> **Note:** `capture_output` was added in Python 3.7. For older versions, use `stdout=subprocess.PIPE, stderr=subprocess.PIPE`.

### 3.2 Understanding text=True vs text=False

#### With text=True (String Output)

```python
import subprocess
import sys

cp_as_str: subprocess.CompletedProcess[str] = subprocess.run(
    [sys.executable, "--version"],
    capture_output=True,
    text=True,
    encoding="utf-8"
)

print("Return Code:", cp_as_str.returncode)
print("Standard Output (strings):", cp_as_str.stdout)
print("Standard Error (strings):", cp_as_str.stderr)
```

**Output:**

```
Return Code: 0
Standard Output (strings): Python 3.14.0

Standard Error (strings):
```

#### With text=False (Bytes Output)

```python
cp_as_bytes: subprocess.CompletedProcess[bytes] = subprocess.run(
    [sys.executable, "--version"],
    capture_output=True,
    # text=False by default
)

print("Return Code:", cp_as_bytes.returncode)
print("Standard Output (bytes):", cp_as_bytes.stdout)
print("Standard Error (bytes):", cp_as_bytes.stderr)
```

**Output:**

```
Return Code: 0
Standard Output (bytes): b'Python 3.14.0\r\n'
Standard Error (bytes): b''
```

### 3.3 When to Use Each Mode

- **Use `text=True`**: When working with text output (logs, version strings, human-readable data)
- **Use `text=False`**: When handling binary data (images, archives, precise byte-level operations)
- **Tip**: Explicitly set `encoding="utf-8"` for consistent, reproducible decoding

---

## 4. Handling Success and Failure

### 4.1 Pattern A: Manual Return Code Inspection

Use this pattern when non-zero return codes are expected or you want to handle them manually:

```python
import subprocess

cp: subprocess.CompletedProcess[str] = subprocess.run(
    ["git", "status"],
    capture_output=True,
    text=True,
    # check=False by default
)

if cp.returncode == 0:
    print("Success:", cp.stdout)
else:
    print("Failed with code", cp.returncode)
    print("Error:", cp.stderr)
```

**Output:**

```
Success: On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  ...
```

### 4.2 Pattern B: Exception-Based Error Handling

Use this pattern when non-zero exits should be treated as exceptions (fail-fast approach):

```python
import subprocess

try:
    cp: subprocess.CompletedProcess[str] = subprocess.run(
        ["git", "status"],
        capture_output=True,
        text=True,
        check=True
    )
except subprocess.CalledProcessError as e:
    print("Command failed:", e.cmd)
    print("Return Code:", e.returncode)
    print("Standard Output:", e.stdout)
    print("Standard Error:", e.stderr)
else:
    print("Command succeeded. Output:")
    print(cp.stdout)
```

### 4.3 Choosing the Right Pattern

| Use Pattern A When                  | Use Pattern B When                   |
| ----------------------------------- | ------------------------------------ |
| Non-zero exits are normal           | Failures should stop execution       |
| You need custom logic per exit code | You want automatic error propagation |
| You're checking multiple conditions | You want clean success path code     |

**Key Difference:**

- Pattern A: `cp` is always a `CompletedProcess` object
- Pattern B: `e` only exists when `check=True` and the command fails

---

## 5. Exception Handling

### 5.1 Common subprocess Exceptions

| Exception                       | When It Occurs                                   | Parent Class      |
| ------------------------------- | ------------------------------------------------ | ----------------- |
| `FileNotFoundError`             | Executable not found (e.g., `git` not installed) | `OSError`         |
| `PermissionError`               | File exists but lacks execute permission         | `OSError`         |
| `subprocess.TimeoutExpired`     | Process exceeds `timeout` seconds                | `SubprocessError` |
| `subprocess.CalledProcessError` | Non-zero exit when `check=True`                  | `SubprocessError` |
| `OSError`                       | General OS-level errors                          | Base exception    |
| `subprocess.SubprocessError`    | Base class for subprocess-specific errors        | Base exception    |

### 5.2 Comprehensive Exception Handling Example

```python
import subprocess

try:
    cp: subprocess.CompletedProcess[str] = subprocess.run(
        ["myprogram"],
        check=True,
        capture_output=True,
        text=True,
        timeout=5
    )
except FileNotFoundError as fe:
    print(f"Program not found: {fe.filename}")
    print("Is it installed and in your PATH?")

except PermissionError:
    print("Permission denied: cannot execute 'myprogram'")

except subprocess.TimeoutExpired as te:
    print(f"Timed out after {te.timeout} seconds")
    print(f"Partial output: {te.stdout}")

except subprocess.CalledProcessError as cpe:
    print(f"Command {cpe.cmd} failed")
    print(f"Return code: {cpe.returncode}")
    print(f"Error output: {cpe.stderr}")

else:
    print("Success! Output:")
    print(cp.stdout)
```

> **Tip:** Order exception handlers from specific to general (`FileNotFoundError` before `OSError`) to catch the most precise error type first.

---

## 6. Best Practices

### 6.1 Security and Safety

- ✅ **Always use a list of arguments**: `["prog", "arg1", "arg2"]`
- ❌ **Avoid `shell=True`**: Prevents shell injection vulnerabilities
- ✅ **Validate user input**: Never pass unsanitized user input to commands
  <!-- TODO: Show how to validate user input using shlex, especially when `shell=True` -->

### 6.2 Output Handling

- Use `capture_output=True` to capture both stdout and stderr
- Set `text=True` for text processing, `text=False` for binary data
- Specify `encoding="utf-8"` for reproducible text decoding

### 6.3 Error Management

- Use `check=True` when failures should stop your script
- Catch `CalledProcessError` to log detailed failure information
- Use manual `returncode` checking when non-zero exits are expected

### 6.4 Timeout Handling

- Always set `timeout=...` for potentially long-running processes
- Catch `TimeoutExpired` to access partial output via `.stdout` and `.stderr`

### 6.5 Exception Handling

- Catch specific exceptions rather than broad `Exception`
- Order handlers from specific to general
- Always log or handle errors appropriately

### 6.6 Compatibility

- `capture_output=True` requires Python 3.7+
- For older versions: `stdout=subprocess.PIPE, stderr=subprocess.PIPE`

---

## 7. Real-World Example

### 7.1 System Info Report Generator

This example demonstrates a practical script that generates a system information report. It showcases comprehensive exception handling with try-except-else blocks, multiple subprocess calls, and argument parsing - following the same structure as your publish.py.

**Key Features:**

- Runs multiple system commands safely
- Comprehensive exception handling (FileNotFoundError, TimeoutExpired, CalledProcessError)
- Uses try-except-else pattern
- argparse integration for CLI options
- Returns meaningful exit codes
<!-- TODO: Confirm that this code works -->
```python
import argparse
import subprocess
import sys
from enum import IntEnum
from typing import NoReturn


class ExitCode(IntEnum):
    SUCCESS = 0
    ERROR = 1


def generate_report(output_file: str, verbose: bool = False) -> ExitCode:
    """
    Generate a system information report and save it to a file.

    Args:
        output_file: Path to save the report
        verbose: If True, show detailed progress messages

    Returns:
        Exit code (0 for success, 1 for failure/error)
    """
    if verbose:
        print(f"Generating system report: {output_file}\n")

    report_lines = ["=" * 60, "SYSTEM INFORMATION REPORT", "=" * 60, ""]

    try:
        # Get system hostname
        if verbose:
            print("→ Getting hostname...")
        hostname_result: subprocess.CompletedProcess[str] = subprocess.run(
            ["hostname"],
            check=True,
            capture_output=True,
            text=True,
            encoding="utf-8",
            timeout=5,
        )
        report_lines.append(f"Hostname: {hostname_result.stdout.strip()}")
        report_lines.append("")

        # Get current user
        if verbose:
            print("→ Getting current user...")
        user_result: subprocess.CompletedProcess[str] = subprocess.run(
            ["whoami"],
            check=True,
            capture_output=True,
            text=True,
            encoding="utf-8",
            timeout=5,
        )
        report_lines.append(f"Current User: {user_result.stdout.strip()}")
        report_lines.append("")

        # Get disk usage
        if verbose:
            print("→ Getting disk usage...")
        disk_result: subprocess.CompletedProcess[str] = subprocess.run(
            ["df", "-h"],
            check=True,
            capture_output=True,
            text=True,
            encoding="utf-8",
            timeout=10,
        )
        report_lines.append("Disk Usage:")
        report_lines.append(disk_result.stdout.strip())
        report_lines.append("")

        # Get memory info
        if verbose:
            print("→ Getting memory info...")
        memory_result: subprocess.CompletedProcess[str] = subprocess.run(
            ["free", "-h"],
            check=True,
            capture_output=True,
            text=True,
            encoding="utf-8",
            timeout=10,
        )
        report_lines.append("Memory Usage:")
        report_lines.append(memory_result.stdout.strip())
        report_lines.append("")

        # Get top 5 processes by memory
        if verbose:
            print("→ Getting top processes...")
        process_result: subprocess.CompletedProcess[str] = subprocess.run(
            ["ps", "aux", "--sort=-%mem"],
            check=True,
            capture_output=True,
            text=True,
            encoding="utf-8",
            timeout=10,
        )
        lines = process_result.stdout.strip().split("\n")[:6]  # Header + top 5
        report_lines.append("Top 5 Processes by Memory:")
        report_lines.extend(lines)
        report_lines.append("")

    except FileNotFoundError as e:
        print(f"ERROR: Command not found: {e.filename}")
        print(
            "Make sure required commands (hostname, whoami, df, free, ps) are installed"
        )
        return ExitCode.ERROR

    except subprocess.TimeoutExpired as e:
        print(f"ERROR: Command timed out after {e.timeout}s: {' '.join(e.cmd)}")
        if e.stdout:
            print(f"Partial stdout: {e.stdout[:200]}...")
        return ExitCode.ERROR

    except subprocess.CalledProcessError as e:
        print(f"ERROR: Command failed: {' '.join(e.cmd)}")
        print(f"Return code: {e.returncode}")
        if e.stdout:
            print(f"stdout: {e.stdout}")
        if e.stderr:
            print(f"stderr: {e.stderr}")
        return ExitCode.ERROR

    except Exception as e:
        print(f"ERROR: Unexpected error when generating: {str(e)}")
        return ExitCode.ERROR

    else:
        # Only runs if no exception was raised
        if verbose:
            print("→ Finalizing report...")

        report_lines.append("=" * 60)
        report_lines.append("Report generated successfully")
        report_lines.append("=" * 60)

        try:
            # Write report to file
            with open(output_file, "w", encoding="utf-8") as f:
                f.write("\n".join(report_lines))

        except IOError as e:
            print(f"ERROR: Failed to write report to {output_file}: {e}")
            return ExitCode.ERROR

        except Exception as e:
            print(f"ERROR: Unexpected error when writing to report file: {str(e)}")
            return ExitCode.ERROR

        else:
            if verbose:
                print(f"\n✅ Report saved successfully to: {output_file}")
            else:
                print(f"Report saved: {output_file}")

            return ExitCode.SUCCESS


def main() -> NoReturn:
    parser = argparse.ArgumentParser(
        description="Generate a system information report with hostname, disk, memory, and process info.",
        epilog=(
            "Examples:\n"
            "  %(prog)s report.txt              # Generate report\n"
            "  %(prog)s report.txt --verbose    # Show detailed progress"
        ),
        formatter_class=argparse.RawDescriptionHelpFormatter,
    )

    parser.add_argument(
        "output",
        type=str,
        help="Output file path for the report",
    )

    parser.add_argument(
        "-v",
        "--verbose",
        action="store_true",
        help="Show detailed progress messages",
    )

    args: argparse.Namespace = parser.parse_args()
    sys.exit(generate_report(args.output, args.verbose))


if __name__ == "__main__":
    main()
```

### 7.2 What This Example Demonstrates

**subprocess Best Practices:**

- Multiple subprocess calls with `check=True` for fail-fast behavior
- Timeout handling on all commands (5-10 seconds)
- Proper `capture_output=True` and `text=True` usage
- Explicit `encoding="utf-8"` for reproducible text handling

**Exception Handling Pattern (try-except-else):**

- `try`: All subprocess calls that might fail
- `except FileNotFoundError`: Command not found in PATH
- `except TimeoutExpired`: Command took too long (shows partial output)
- `except CalledProcessError`: Command returned non-zero exit code
- `except Exception`: Catch-all for unexpected errors
- `else`: Success path - only executes if no exceptions occurred

**Code Structure (matches publish.py):**

- Two functions: `main()` and `generate_report()`
- `main()`: Parses arguments and passes to worker function
- Worker function: Contains all logic with try-except-else block
- Returns `ExitCode` enum for meaningful exit codes
- Type hints throughout for clarity

**argparse Integration:**

- Positional argument (output file)
- Optional flag (--verbose)
- Help text with examples
- `RawDescriptionHelpFormatter` for formatted epilog

**Note:** This script uses Unix commands (`hostname`, `whoami`, `df`, `free`, `ps`). On Windows, you'd use different commands like `hostname.exe`, `whoami.exe`, `wmic`, `tasklist`, etc.

📚 **Learn about argument parsing:** [13-`argparse`](./13-argparse.ipynb)

---

## 8. Quick Reference

```python
import subprocess

# Simple command
subprocess.run(["ls", "-la"])

# Capture output as text
cp = subprocess.run(
    ["git", "status"],
    capture_output=True,
    text=True,
    encoding="utf-8"
)

# Fail-fast on errors
try:
   cp: subprocess.CompletedProcess[str] = subprocess.run(
       ["pytest"],
       check=True,
       capture_output=True,
       text=True,
       encoding="utf-8"
       timeout=300
   )
except subprocess.CalledProcessError as e:
   print(e.cmd)
   print(e.returncode)
   print(e.stderr)
else:
   print(cp.stdout)

# Handle errors manually
cp = subprocess.run(["make", "build"], capture_output=True, text=True)
if cp.returncode != 0:
    print(f"Build failed: {cp.stderr}")
```

---

## 9. Further Reading

- **Official Documentation:** [subprocess — Python Docs](https://docs.python.org/3/library/subprocess.html)
- **Next Module:** [13-`argparse`](./13-argparse.ipynb) — Learn to build CLI tools
- **Related Topics:** `os.system()`, `shlex`, `pathlib`
