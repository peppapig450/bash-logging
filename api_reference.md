# API Reference

Complete documentation for all functions provided by the Bashing Logs library.

## Core Logging Functions

### `logging::log_info MESSAGE`

Logs an informational message with green color coding.

**Parameters:**

- `MESSAGE`: The message to log (can contain spaces)

**Example:**

```bash
logging::log_info "Database connection established"
logging::log_info "Processing file: $filename"
```

**Output:**

```text
[2025-06-10T04:24:11Z][INFO][script.sh] Database connection established
```

---

### `logging::log_warn MESSAGE`

Logs a warning message with yellow color coding.

**Parameters:**

- `MESSAGE`: The warning message to log

**Example:**

```bash
logging::log_warn "Configuration file not found, using defaults"
logging::log_warn "API rate limit approaching: $remaining_calls remaining"
```

**Output:**

```text
[2025-06-10T04:24:11Z][WARN][script.sh] Configuration file not found, using defaults
```

---

### `logging::log_error MESSAGE`

Logs an error message with red color coding. Does not exit the script.

**Parameters:**

- `MESSAGE`: The error message to log

**Example:**

```bash
logging::log_error "Failed to connect to database after 3 attempts"
logging::log_error "Invalid configuration in section: $section_name"
```

**Output:**

```text
[2025-06-10T04:24:11Z][ERROR][script.sh] Failed to connect to database after 3 attempts
```

---

### `logging::log_fatal MESSAGE`

Logs an error message with red color coding and exits the script with status 1.

**Parameters:**

- `MESSAGE`: The fatal error message to log

**Example:**

```bash
logging::log_fatal "Critical dependency missing: $required_tool"
logging::log_fatal "Unable to acquire exclusive lock"
```

**Output:**

```text
[2025-06-10T04:24:11Z][ERROR][script.sh] Critical dependency missing: docker
```

**Note:** The script will exit immediately after logging.

---

## Initialization Functions

### `logging::init SCRIPT_NAME`

Initializes the logging system with automatic error trapping and cleanup.

**Parameters:**

- `SCRIPT_NAME`: Name to use in log prefixes (typically `$0`)

**Behavior:**

- Sets the global `LOGGING_SCRIPT_NAME` variable
- Installs error trap handler for automatic crash diagnostics
- Sets up cleanup handler for script exit
- Safely chains with any existing ERR/EXIT traps

**Example:**

```bash
#!/usr/bin/env bash
source ./logging.lib.sh

# Initialize logging (recommended as early as possible)
logging::init "$0"

# Now any command failure will be automatically logged
false  # This will trigger the error trap
```

**Output:**

```text
[2025-06-10T04:24:11Z][ERROR][script.sh] Unexpected fatal error in script.sh on line 7: false
```

---

## Advanced Trap Functions

### `logging::add_err_trap`

Manually adds the error trap handler without full initialization.

**Use Case:** When you want error trapping but not the full initialization (e.g., in libraries or when you need custom setup).

**Example:**

```bash
# Add only error trapping
logging::add_err_trap

# Set script name manually if desired
LOGGING_SCRIPT_NAME="custom-name"
```

**Note:** Called automatically by `logging::init`.

---

### `logging::add_exit_trap`

Manually adds the cleanup handler for script exit.

**Use Case:** When you need custom cleanup behavior or want to add cleanup without error trapping.

**Example:**

```bash
# Add only exit cleanup
logging::add_exit_trap
```

**Note:** Called automatically by `logging::init`.

---

### `logging::setup_traps`

Sets up both ERR and EXIT traps. Equivalent to calling both `logging::add_err_trap` and `logging::add_exit_trap`.

**Example:**

```bash
# Manual trap setup (without setting script name)
logging::setup_traps
```

**Note:** Called automatically by `logging::init`.

---

## Internal Functions

These functions are used internally but may be useful for advanced use cases.

### `logging::log LEVEL MESSAGE`

Internal function used by all logging functions. Handles timestamp generation, color coding, and formatting.

**Parameters:**

- `LEVEL`: Log level (INFO, WARN, ERROR)
- `MESSAGE`: Message to log

**Example:**

```bash
# Generally you should use the specific functions instead
logging::log INFO "This is equivalent to logging::log_info"
logging::log WARN "This is equivalent to logging::log_warn"
```

---

### `logging::trap_err_handler`

The actual error trap handler function. Logs fatal errors with context and exits.

**Use Case:** Advanced trap customization.

**Example:**

```bash
# Custom trap that does additional cleanup
trap 'cleanup_function; logging::trap_err_handler' ERR
```

---

### `logging::cleanup`

Cleanup function that unsets global logging state.

**Use Case:** Manual cleanup or custom exit handling.

**Example:**

```bash
# Manual cleanup
logging::cleanup
```

---

## Global Variables

### `LOGGING_SCRIPT_NAME`

Global variable that stores the script name for log prefixing.

**Set by:** `logging::init`
**Type:** String
**Scope:** Global export

**Example:**

```bash
# Set automatically
logging::init "$0"
echo "Script name: $LOGGING_SCRIPT_NAME"

# Or set manually
LOGGING_SCRIPT_NAME="my-custom-script"
logging::log_info "This will show [my-custom-script] in the log"
```

---

## Function Naming Convention

All functions follow the `namespace::function` convention from the Google Shell Style Guide:

- **Namespace:** `logging::`
- **Purpose:** Prevents naming conflicts with other libraries
- **Consistency:** All functions are prefixed, making them easy to identify

---

## Error Handling

### Return Codes

- **Logging functions:** Always return 0 (success)
- **`logging::log_fatal`:** Exits with code 1
- **`logging::init`:** Returns 1 if called with wrong number of arguments

### Error Scenarios

**Invalid log level:**

```bash
logging::log INVALID "message"  # Will exit with error
```

**Missing arguments:**

```bash
logging::init  # Returns 1 and logs warning
```

**Shell compatibility:**

```bash
# If not running in Bash, library will fail to load
#!/bin/sh
source ./logging.lib.sh  # Will exit with error message
```

---

## Output Specifications

### Timestamp Format

- **Format:** ISO 8601/RFC3339 UTC (`YYYY-MM-DDTHH:MM:SSZ`)
- **Timezone:** Always UTC for consistency across systems
- **Example:** `2025-06-10T04:24:11Z`

### Color Codes

When outputting to a terminal:

- **INFO:** `\033[0;32m` (green)
- **WARN:** `\033[0;33m` (yellow)  
- **ERROR:** `\033[0;31m` (red)
- **Reset:** `\033[0m`

Colors are automatically disabled when:

- Output is redirected to a file
- Terminal doesn't support colors
- Running in non-interactive mode

### Output Destination

All logging output goes to **stderr** (file descriptor 2), following Unix conventions for diagnostic output.

---

## Thread Safety

The logging functions are **not** thread-safe. If using in multi-process environments:

- Each process will have its own logging state
- Multiple processes writing to the same log file may interleave output
- Consider using process-specific log files or external log aggregation

---

## Performance Considerations

- **Timestamp generation:** Calls `date` command for each log entry
- **Color detection:** Minimal overhead, cached per session
- **Script name resolution:** Done once during initialization
- **Trap parsing:** Uses Perl, minimal overhead during setup

For high-frequency logging, consider buffering messages or using dedicated logging solutions.
