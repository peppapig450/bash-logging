# Bashing Logs

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

A robust, production-ready logging utility library for Bash scripts with automatic error tracing, safe trap chaining, and zero-setup crash diagnostics.

## Quick Start

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
source ./logging.lib.sh

# Initialize logging with your script name
logging::init "$0"

# Now any error will be automatically logged with context
logging::log_info "Starting deployment..."
logging::log_warn "Configuration file missing, using defaults"

# This will trigger the error trap with full diagnostics
false  # Simulated error
```

**Output:**

```text
[2025-06-10T04:24:11Z][INFO][deploy.sh] Starting deployment...
[2025-06-10T04:24:11Z][WARN][deploy.sh] Configuration file missing, using defaults
[2025-06-10T04:24:11Z][ERROR][deploy.sh] Unexpected fatal error in deploy.sh on line 8: false
```

## Key Features

- 🎨 **Color-coded output** - Clear visual distinction between log levels
- ⏰ **UTC timestamps** - ISO 8601 formatted for consistency
- 🔍 **Automatic error tracing** - Zero-setup crash diagnostics with file, line, and command
- 🔗 **Safe trap chaining** - Preserves existing ERR/EXIT handlers
- 🛡️ **Production-ready** - Defensive programming and strict error handling

## Installation

```bash
# Download the latest release
curl -fsLO https://raw.githubusercontent.com/peppapig450/bashing-logs/main/logging.lib.sh
```

```bash
# Source in your script
source ./logging.lib.sh
logging::init "$0"
```

**Requirements:** Bash 4.0+, standard Unix utilities, Perl

## Core API

```bash
# Initialization (enables automatic error tracing)
logging::init "$0"

# Basic logging
logging::log_info "Operation completed"
logging::log_warn "Retrying connection"
logging::log_error "Failed to connect"
logging::log_fatal "Critical error" # Exits with status 1
```

## Documentation

- **[Getting Started](https://peppapig450.github.io/bashing-logs/getting-started.html)** - Installation, basic usage, and configuration
- **[API Reference](https://peppapig450.github.io/bashing-logs/api-reference.html)** - Complete function documentation
- **[Advanced Features](https://peppapig450.github.io/bashing-logs/advanced-features.html)** - Trap chaining, error diagnostics, technical details
- **[Examples](https://peppapig450.github.io/bashing-logs/examples.html)** - Real-world usage patterns and best practices
- **[Contributing](https://peppapig450.github.io/bashing-logs/contributing.html)** - Development guidelines and project standards

## Use Cases

Perfect for:

- **CI/CD pipelines** - Clear timestamped logs with automatic error reporting
- **System administration** - Robust error handling and diagnostic information
- **Application deployment** - Script coordination with proper error tracing
- **Development tooling** - Consistent logging across build and utility scripts
- **Anything else** - This works well with any Bash script in general

## License

[MIT License](./LICENSE) - see LICENSE file for details.

---

**Need help?** Check the [documentation](https://peppapig450.github.com/bashing-logs) or [open an issue](https://github.com/peppapig450/bashing-logs/issues).
