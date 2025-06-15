# Advanced Features

Deep dive into the sophisticated features that make Bashing Logs production-ready.

## Safe Trap Chaining

One of the most sophisticated features of this library is its ability to safely chain with existing trap handlers. Most logging libraries simply overwrite existing traps, breaking error handling in complex scripts.

### The Problem with Traditional Approaches

Bash trap syntax is notoriously difficult to parse correctly:

```bash
# Simple trap
trap 'echo "error"' ERR

# Complex trap with quotes and special characters  
trap 'echo "Error in $0"; cleanup || true' ERR

# Multiple commands with different quoting
trap 'cleanup; notify_admin "Script failed"; exit 1' ERR
```

Traditional approaches fail because they:

- Use fragile regex that breaks on special characters
- Don't handle nested quotes properly
- Overwrite existing functionality instead of chaining

### The Solution: Perl-Based Parsing

This library uses Perl to safely parse existing trap output:

```perl
perl -lne '
    if (/^trap -- '\''([^'\'']*)'\'' ERR$/) {
        print "$1"
    }
'
```

This regex precisely extracts commands from `trap -p` output by:

- Matching the exact `trap --` format
- Handling escaped single quotes within commands
- Preserving spaces and special characters
- Supporting multiple commands separated by semicolons

### Chaining Process

1. **Extract existing trap:** Parse current ERR/EXIT trap using Perl
2. **Validate command:** Ensure the extracted command is safe to chain
3. **Create new trap:** Combine existing command with library handler
4. **Install safely:** Set the new compound trap

```bash
# Before: existing trap
trap 'cleanup_database' ERR

# After: chained trap
trap 'cleanup_database; logging::trap_err_handler' ERR
```

### Chaining Examples

**Single existing command:**

```bash
# Original
trap 'cleanup' ERR

# After logging::add_err_trap
trap 'cleanup; logging::trap_err_handler' ERR
```

**Multiple existing commands:**

```bash
# Original  
trap 'cleanup; notify_admin; exit 1' ERR

# After logging::add_err_trap
trap 'cleanup; notify_admin; exit 1; logging::trap_err_handler' ERR
```

**Complex commands with quotes:**

```bash
# Original
trap 'echo "Failed in $0" | logger' ERR

# After logging::add_err_trap  
trap 'echo "Failed in $0" | logger; logging::trap_err_handler' ERR
```

---

## Zero-Setup Error Diagnostics

After calling `logging::init`, any command failure automatically provides detailed diagnostics.

### What Gets Logged

For every error, the trap handler captures:

1. **Source file:** Which script encountered the error
2. **Line number:** Exact line where the failure occurred  
3. **Failed command:** The command that caused the error
4. **Context:** Script name and timestamp

### Error Context Variables

The trap handler uses these Bash built-in arrays:

```bash
${BASH_SOURCE[1]}   # Source file of the error
${BASH_LINENO[0]}   # Line number of the error
${BASH_COMMAND}     # The command that failed
```

### Example Diagnostic Output

```bash
# Script: deploy.sh, line 42
rsync -av /source/ /nonexistent/

# Automatic output:
[2024-12-15T10:30:47Z][ERROR][deploy.sh] Unexpected fatal error in deploy.sh on line 42: rsync -av /source/ /nonexistent/
```

### Error Scenarios Covered

**Command not found:**

```bash
nonexistent_command arg1 arg2
# Output: Unexpected fatal error in script.sh on line 15: nonexistent_command arg1 arg2
```

**File operation failures:**

```bash
cp /protected/file /destination/
# Output: Unexpected fatal error in script.sh on line 23: cp /protected/file /destination/
```

**Pipeline failures (with `set -o pipefail`):**

```bash
cat missing_file | grep pattern
# Output: Unexpected fatal error in script.sh on line 31: cat missing_file | grep pattern
```

---

## Technical Implementation Details

### Shell Detection

The library includes robust shell detection to ensure Bash compatibility:

```bash
_detect_shell() {
  # First try /proc filesystem (Linux)
  if [ -r "/proc/$$/exe" ]; then
    basename "$(readlink /proc/$$/exe)"
  # Fall back to ps command (macOS, BSD)
  else
    basename "$(ps -p $$ -o comm= 2>/dev/null)"
  fi
}
```

**Detection methods:**

- **Linux:** Uses `/proc/$$/exe` symlink to current executable
- **macOS/BSD:** Uses `ps` with process ID to get command name
- **Fallback:** Returns "unknown" if both methods fail

### Source-Only Execution

Prevents accidental direct execution of the library:

```bash
(return 0 2>/dev/null) || {
    printf "This script is meant to be sourced, not executed.\n" >&2
    exit 1
}
```

**How it works:**

- `return` succeeds only when sourced (not executed)
- `2>/dev/null` suppresses error output in execution context
- Parentheses create subshell to isolate the test

### Strict Mode Compatibility

Designed to work seamlessly with Bash strict mode:

```bash
set -Eeuo pipefail
```

**Compatibility features:**

- **`-e` (errexit):** Library functions handle errors gracefully
- **`-E` (errtrace):** ERR trap inheritance works with chaining
- **`-u` (nounset):** All variables are properly initialized
- **`-o pipefail`:** Pipeline failures are caught by error traps

### Color Detection and Handling

Automatic color detection based on output destination:

```bash
# Colors enabled when:
# - Output is to a terminal (tty)
# - Terminal supports ANSI escape sequences
# - Not running in CI (unless explicitly enabled)

# Colors disabled when:
# - Output redirected to file
# - Terminal doesn't support colors
# - Running in non-interactive environment
```

---

## Performance Considerations

### Timestamp Generation

Each log call executes `date -u`:

- **Cost:** ~1-5ms per call on modern systems
- **Alternative:** Could cache for microsecond precision, but adds complexity
- **Trade-off:** Accuracy vs. performance (accuracy chosen)

### Trap Parsing Overhead

Perl parsing for trap chaining:

- **When:** Only during initialization (`logging::init`)
- **Cost:** ~10-50ms one-time setup cost
- **Frequency:** Once per script execution
- **Alternative:** Shell-only parsing (fragile and error-prone)

### Memory Usage

Global variables used:

- `LOGGING_SCRIPT_NAME`: Script basename (~10-50 bytes)
- Function definitions: ~2-4KB total
- **No persistent buffers or caches**

---

## Integration Patterns

### Library Integration

For libraries that want to use logging without forcing initialization:

```bash
# In your library
my_library::function() {
    # Check if logging is available
    if declare -F logging::log_info >/dev/null 2>&1; then
        logging::log_info "Library operation completed"
    fi
}
```

### Multi-Script Coordination

For scripts that call other scripts:

```bash
# main.sh
#!/usr/bin/env bash
source ./logging.lib.sh
logging::init "$0"

logging::log_info "Starting batch process"

# Child scripts inherit the library
for script in process-*.sh; do
    # Each script can have its own logging::init call
    ./"$script"
done
```

### CI/CD Pipeline Integration

For continuous integration environments:

```bash
# In your CI script
source ./logging.lib.sh
logging::init "ci-pipeline"

# All output is properly timestamped and level-coded
logging::log_info "Starting build process"
make build 2>&1 | while read -r line; do
    logging::log_info "BUILD: $line"
done
```

---

## Debugging and Troubleshooting

### Debug Mode

Enable debug output to see trap chaining:

```bash
# Set before sourcing library
export LOGGING_DEBUG=1
source ./logging.lib.sh
```

### Common Issues

**Trap not working:**

```bash
# Check if strict mode is enabled
set -E  # Ensure ERR trap inheritance

# Verify trap installation
trap -p ERR
```

**Color issues:**

```bash
# Force colors in CI
export TERM=xterm-256color

# Disable colors entirely
export NO_COLOR=1
```

**Performance concerns:**

```bash
# For high-frequency logging, consider batching
{
    echo "Message 1"
    echo "Message 2"  
    echo "Message 3"
} | while read -r msg; do
    logging::log_info "$msg"
done
```

---

## Security Considerations

### Command Injection

The library is designed to prevent command injection:

- All user input is properly quoted
- No `eval` or dynamic command construction
- Perl parsing uses safe regex patterns

### Information Disclosure

Error messages may reveal:

- File paths and script names
- Command arguments (potentially sensitive)
- System usernames (in paths)

**Mitigation:** Review error messages before production deployment.

### Privilege Escalation

The library doesn't:

- Create files or directories
- Modify system configuration
- Execute arbitrary commands
- Require special privileges

---

## Extending the Library

### Custom Log Levels

Add new log levels by extending the color mapping:

```bash
# Add to logging::log function
case "${level}" in
    INFO) color=$'\033[0;32m' ;;
    WARN) color=$'\033[0;33m' ;;
    ERROR) color=$'\033[0;31m' ;;
    DEBUG) color=$'\033[0;36m' ;;  # Cyan
    TRACE) color=$'\033[0;37m' ;;  # Gray
    *)
        printf "Invalid log level: %s\n" "${level}"
        exit 1
        ;;
esac

# Add convenience function
logging::log_debug() { logging::log DEBUG "$@"; }
```

### Custom Formatters

Override the logging format:

```bash
# Custom timestamp format
logging::custom_timestamp() {
    date '+%Y-%m-%d %H:%M:%S'
}

# Override in logging::log function
ts="$(logging::custom_timestamp)"
```

### Integration Hooks

Add hooks for external log aggregation:

```bash
logging::log() {
    # ... existing implementation ...
    
    # Send to external system (optional)
    if command -v logger >/dev/null 2>&1; then
        logger -t "${LOGGING_SCRIPT_NAME:-script}" "${level}: $*"
    fi
}
```
