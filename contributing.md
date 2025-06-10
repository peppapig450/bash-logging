# Contributing

Thank you for your interest in contributing to Bashing Logs! This document provides guidelines for contributing to the project.

## Development Setup

### Prerequisites

- **Bash 4.0+** (for testing the library)
- **Perl** (for trap parsing functionality)
- **Git** (for version control)
- **ShellCheck** (for static analysis)
- **Bats** (for testing - optional but recommended)

### Setup Instructions

1. **Fork and clone the repository:**

   ```bash
   git clone https://github.com/yourusername/bashing-logs.git
   cd bashing-logs
   ```

2. **Install development tools:**

   ```bash
   # Install ShellCheck
   # Ubuntu/Debian
   sudo apt-get install shellcheck
   
   # macOS
   brew install shellcheck
   
   # Install Bats (optional)
   git clone https://github.com/bats-core/bats-core.git
   cd bats-core
   ./install.sh /usr/local
   ```

3. **Verify setup:**

   ```bash
   # Test the library
   bash -c "source ./logging.lib.sh; logging::log_info 'Test message'"
   
   # Run ShellCheck
   shellcheck logging.lib.sh
   ```

---

## Code Standards

### Shell Style Guide

This project follows the [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html) with these specific conventions:

#### Function Naming

All functions must use the `namespace::function` convention:

```bash
# ✓ Correct
logging::log_info() { ... }
logging::init() { ... }

# ✗ Incorrect
log_info() { ... }
init_logging() { ... }
```

#### Variable Naming

- **Global variables:** `UPPER_CASE` with namespace prefix when appropriate
- **Local variables:** `lower_case`
- **Constants:** `readonly UPPER_CASE`

```bash
# Global variables
declare -g LOGGING_SCRIPT_NAME
readonly LOGGING_VERSION="1.0.0"

# Local variables
logging::init() {
    local script_name="$1"
    local config_file="config.yml"
}
```

#### Error Handling

- Always use `set -Eeuo pipefail` for strict error handling
- Provide meaningful error messages
- Use appropriate exit codes

```bash
# ✓ Good error handling
if [[ ! -f "$config_file" ]]; then
    logging::log_fatal "Configuration file not found: $config_file"
fi

# ✗ Poor error handling
[[ -f "$config_file" ]] || exit 1
```

### Code Formatting

#### Indentation and Spacing

- **Indentation:** 2 spaces (no tabs)
- **Line length:** Maximum 80 characters where practical
- **Function spacing:** One blank line between functions

```bash
# ✓ Correct formatting
logging::log_info() {
  local message="$*"
  logging::log INFO "$message"
}

logging::log_warn() {
  local message="$*"
  logging::log WARN "$message"
}
```

#### Quoting

- Quote variables to prevent word splitting: `"$variable"`
- Use single quotes for literal strings: `'literal text'`
- Use `$()` for command substitution, not backticks

```bash
# ✓ Correct quoting
local timestamp="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
local message="Processing file: $filename"

# ✗ Incorrect quoting
local timestamp=`date -u +%Y-%m-%dT%H:%M:%SZ`
local message=Processing file: $filename
```

### Documentation Standards

#### Function Documentation

All public functions must include documentation:

```bash
# logging::init SCRIPT_NAME
# Initializes the logging system with automatic error trapping.
#
# Sets the global LOGGING_SCRIPT_NAME variable and installs
# error handlers for comprehensive crash diagnostics.
#
# Arguments:
#   SCRIPT_NAME - Name to use in log prefixes (typically $0)
#
# Usage:
#   logging::init "$0"
logging::init() {
    # Implementation...
}
```

#### Comment Style

- Use `#` for single-line comments
- Use `# ===` for section headers
- Explain complex logic and non-obvious decisions

```bash
# ==============================================================================
# Trap handling functions
# ==============================================================================

# Extract existing ERR trap command using Perl for safe parsing.
# We use Perl because shell quoting rules are complex and error-prone
# when parsing trap output that may contain special characters.
existing="$(perl -lne '...' <<< "$(trap -p ERR)" || true)"
```

---

## Testing Guidelines

### Manual Testing

Before submitting changes, test your modifications:

```bash
# Create test script
cat > test_changes.sh << 'EOF'
#!/usr/bin/env bash
set -Eeuo pipefail

source ./logging.lib.sh
logging::init "$0"

# Test basic logging
logging::log_info "Testing info message"
logging::log_warn "Testing warning message"
logging::log_error "Testing error message"

# Test error trapping
echo "About to trigger error trap..."
false  # This should be caught and logged
EOF

chmod +x test_changes.sh
./test_changes.sh
```

### Automated Testing

If you have Bats installed, run the test suite:

```bash
# Run all tests
bats tests/

# Run specific test file
bats tests/logging_test.bats
```

### Testing Checklist

- [ ] All log levels work correctly
- [ ] Error trapping functions properly
- [ ] Trap chaining preserves existing handlers
- [ ] Script name prefixing works
- [ ] Color output appears correctly in terminals
- [ ] No colors when redirected to files
- [ ] Compatible with `set -Eeuo pipefail`
- [ ] No ShellCheck warnings
- [ ] Works on different shells (Bash 4.0+)

---

## Contribution Process

### 1. Issue First

For significant changes, please open an issue first to discuss:

- New features or enhancements
- Breaking changes
- Performance improvements
- Documentation restructuring

### 2. Branch Naming

Use descriptive branch names:

```bash
# Feature branches
git checkout -b feature/add-debug-level
git checkout -b feature/json-output-format

# Bug fixes
git checkout -b fix/trap-chaining-quotes
git checkout -b fix/color-detection-ci

# Documentation
git checkout -b docs/api-reference-update
git checkout -b docs/examples-expansion
```

### 3. Commit Messages

Follow conventional commit format:

```
type(scope): description

- feat: new features
- fix: bug fixes
- docs: documentation changes
- style: formatting changes
- refactor: code restructuring
- test: adding or updating tests
- chore: maintenance tasks

Examples:
feat(logging): add support for custom log levels
fix(traps): handle complex quoting in existing traps
docs(examples): add CI/CD pipeline examples
```

### 4. Pull Request Process

1. **Update documentation** if you've changed functionality
2. **Add tests** for new features
3. **Run ShellCheck** and fix any warnings
4. **Test manually** with the provided test scripts
5. **Update CHANGELOG.md** if applicable

#### Pull Request Template

When creating a PR, include:

```markdown
## Description
Brief description of the changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Documentation update
- [ ] Performance improvement

## Testing
- [ ] Manual testing completed
- [ ] ShellCheck passes
- [ ] Existing functionality unaffected

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No breaking changes (or clearly marked)
```

---

## Project Standards

### Backward Compatibility

- **Maintain API stability:** Existing function signatures should not change
- **Deprecation process:** Mark old features as deprecated before removal
- **Version semantic:** Follow semantic versioning (MAJOR.MINOR.PATCH)

### Performance Considerations

- **Minimize external commands:** Each `date` call adds ~1-5ms overhead
- **Avoid unnecessary work:** Don't parse traps if not needed
- **Memory efficiency:** Clean up temporary variables

### Security Guidelines

- **Input validation:** Validate all user inputs
- **No command injection:** Properly quote all variables
- **Safe defaults:** Fail securely when possible
- **Minimal privileges:** Don't require elevated permissions

---

## Release Process

### Version Numbering

Follow semantic versioning:

- **MAJOR:** Breaking changes to public API
- **MINOR:** New features, backward compatible
- **PATCH:** Bug fixes, backward compatible

### Release Checklist

1. **Update version numbers** in relevant files
2. **Update CHANGELOG.md** with release notes
3. **Tag the release** with `git tag vX.Y.Z`
4. **Create GitHub release** with release notes
5. **Update documentation** if needed

---

## Getting Help

### Communication Channels

- **GitHub Issues:** Bug reports and feature requests
- **GitHub Discussions:** Questions and general discussion
- **Pull Request Reviews:** Code-specific feedback

### Code Review Process

All contributions go through code review:

1. **Automated checks:** ShellCheck, basic tests
2. **Maintainer review:** Code quality, design, documentation
3. **Community feedback:** Other contributors may provide input
4. **Approval and merge:** Once all feedback is addressed

### Response Times

- **Initial response:** Within 48 hours
- **Review completion:** Within 1 week for small changes
- **Complex features:** May require multiple review cycles

---

## Recognition

Contributors are recognized in:

- **CONTRIBUTORS.md** file
- **Release notes** for significant contributions
- **Git history** with proper attribution

### Types of Contributions

All contributions are valued:

- **Code contributions:** Features, bug fixes, improvements
- **Documentation:** Examples, guides, API docs
- **Testing:** Test cases, bug reports, compatibility testing
- **Community:** Answering questions, code reviews

---

## Code of Conduct

This project follows a code of conduct based on respect and inclusivity:

### Our Pledge

We pledge to make participation in our project a harassment-free experience for everyone, regardless of background or identity.

### Standards

- **Respectful communication:** Be kind and professional
- **Constructive feedback:** Focus on code and ideas, not individuals
- **Inclusive language:** Avoid terminology that could exclude others
- **Collaborative approach:** Work together toward common goals

### Enforcement

Issues can be reported to project maintainers. All reports will be reviewed and investigated promptly.

---

Thank you for contributing to Bashing Logs! Your efforts help make Bash scripting more robust and maintainable for everyone.
