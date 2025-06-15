# Getting started

This guide will help you install and configure the Bashing Logs library in your Bash scripts.

## Installation

### Basic Installation

Simply download and source the library:

```bash
# Download the library
curl -fsLO https://raw.githubusercontent.com/peppapig450/bashing-logs/main/logging.lib.sh
```

Include this in your script:

```bash
source ./logging.lib.sh
```

### Project Integration

For projects with multiple scripts, place `logging.lib.sh` in a common location:

```text
project/
├── lib/
│   └── logging.lib.sh
├── scripts/
│   ├── deploy.sh
│   └── backup.sh
└── README.md
```

Then source with a relative path:

```bash
#!/usr/bin/env bash
SOURCE_DIR="$(cd -Pe -- "$(dirname -- "${BASH_SOURCE[0]}")" && echo "${PWD}")"
source "${SOURCE_DIR}/../lib/logging.lib.sh"
```

## Requirements

### System Requirements

- **Bash 4.0+** (automatically checked at runtime)
- **Standard Unix utilities:** `date`, `basename`, `realpath`
- **Perl** (for safe trap parsing)

### Compatibility

The library is tested and works on:

- Linux (all major distros except Alpine)
- macOS
- Windows (via WSL/Cygwin)
- CI environments (Github Actions, Gitlab CI, Jenkins)

## Basic Setup

### Minimal Example

```bash
#!/usr/bin/env bash
source ./logging.lib.sh

logging::log_info "Script started"
logging::log_warn "This is a warning"
logging::log_error "This is an error"
```

### Recommended Setup

For production scripts, use this template:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail # Strict mode

# Source the logging library
source ./logging.lib.sh

# Initialize with automatic error trapping
logging::init "$0"

# Your script logic here
main() {
    logging::log_info "Starting main process"
    # ... your code ...
    logging::log_info "Process completed successfully!"
}

# Run main function
main "$@"
```

## Configuration

### Script Name

The library can automatically prefix logs with your script name:

```bash
# Initialize with script name
logging::init "$0"
```

Or set it manually:

```bash
logging::init "my-lovely-script"
```

This produces logs like:

```text
[2025-06-10T04:24:11Z][INFO][my-lovely-script] Hello world!
```

### Strict Mode Compatibility

The library is designed to work with Bash strict mode:

```bash
set -Eeuo pipefail
```

This combination provides:

- **`-e`**: Exit on any command failure
- **`-E`**: ERR trace inheritance
- **`-u`**: Error on undefined
- **`-o`**: Pipeline failure detection

Without `set -E` automatic error tracing will **not** work in functions or subshells.

## Understanding the output

### Log Format

All logs follow this structure:

```text
[TIMESTAMP][LEVEL][SCRIPT_NAME] MESSAGE
```

- **TIMESTAMP**: UTC time in ISO 8601/RFC3339 format (`YYYY-MM-DDTHH:MM:SSZ`)
- **LEVEL**: `INFO`, `WARN`, or `ERROR`
- **SCRIPT_NAME**: Your script's basename (optional)
- **MESSAGE**: Your log content

### Color Coding

When outputting to a terminal:

- **INFO**: Green text
- **WARN**: Yellow text
- **ERROR**: Red text

Colors are automatically disabled when redirecting to files or in non-interactive environments.

### Output Destination

All logs go to **stderr** (standard error), following Unix conventions. This allows you to:

```bash
# Separate logs from script output
./script.sh > output.txt 2> logs.txt

# Or combine them
./script.sh > combined.txt 2>&1
```

## Error Handling

### Automatic Error Trapping

When you call `logging::init`, the library automatically sets up error trapping:

```bash
#!/usr/bin/env bash
source ./logging.lib.sh
logging::init "$0"

# Any command that fails will be automatically logged
false  # This will trigger: "Unexpected fatal error in script.sh on line 6: false"
```

### Manual Error Handling

You can still handle errors manually when needed:

```bash
if ! risky_command; then
    logging::log_error "Risky command failed, but we'll continue"
    # Continue execution
fi

# Or for fatal errors
command || logging::log_fatal "Critical command failed"
```

## Next Steps

- **[API Reference](api-reference.md)** - Learn about all available functions
- **[Advanced Features](advanced-features.md)** - Understand trap chaining and technical details
- **[Examples](examples.md)** - See real-world usage patterns

## Troubleshooting

### Common Issues

**"This script requires Bash" error:**

```bash
# Check your shebang line
#!/usr/bin/env bash  # ✓ Correct
#!/bin/sh            # ✗ Wrong
```

**Perl not found:**
Install Perl on your system.

**Colors not working:**

- Colors are automatically disabled in non-interactive environments
- To force colors: ensure your terminal supports ANSI codes
- To disable colors: redirect stderr to a file

### Getting Help

- Check the [examples](examples.md) for common patterns
- Review the [API reference](api-reference.md) for detailed function docs
- Open an issue on GitHub for bugs or feature requests
