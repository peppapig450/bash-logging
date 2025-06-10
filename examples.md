# Examples

Real-world usage patterns and best practices for the Bashing Logs library.

## Basic Usage Patterns

### Simple Script Template

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# Load and initialize logging
source ./logging.lib.sh
logging::init "$0"

main() {
    logging::log_info "Script started with arguments: $*"
    
    # Your script logic here
    process_data "$@"
    
    logging::log_info "Script completed successfully"
}

process_data() {
    logging::log_info "Processing data files..."
    
    for file in "$@"; do
        if [[ -f "$file" ]]; then
            logging::log_info "Processing: $file"
            # Process file
        else
            logging::log_warn "File not found: $file"
        fi
    done
}

# Run main function
main "$@"
```

### Error Handling Patterns

```bash
#!/usr/bin/env bash
source ./logging.lib.sh
logging::init "$0"

# Pattern 1: Continue on error
attempt_operation() {
    if ! risky_command; then
        logging::log_error "Operation failed, but continuing"
        return 1
    fi
    logging::log_info "Operation successful"
}

# Pattern 2: Retry with backoff
retry_operation() {
    local max_attempts=3
    local delay=1
    
    for attempt in $(seq 1 $max_attempts); do
        logging::log_info "Attempt $attempt/$max_attempts"
        
        if command_that_might_fail; then
            logging::log_info "Operation succeeded on attempt $attempt"
            return 0
        fi
        
        if [[ $attempt -lt $max_attempts ]]; then
            logging::log_warn "Attempt $attempt failed, retrying in ${delay}s"
            sleep $delay
            delay=$((delay * 2))  # Exponential backoff
        fi
    done
    
    logging::log_fatal "Operation failed after $max_attempts attempts"
}

# Pattern 3: Graceful degradation
optional_operation() {
    if ! optional_command; then
        logging::log_warn "Optional operation failed, using fallback"
        fallback_command
    fi
}
```

---

## CI/CD Pipeline Examples

### GitHub Actions Integration

```bash
#!/usr/bin/env bash
# .github/scripts/build.sh
set -Eeuo pipefail

source "$(dirname "$0")/../../lib/logging.lib.sh"
logging::init "github-build"

build_application() {
    logging::log_info "Starting build process"
    logging::log_info "Node version: $(node --version)"
    logging::log_info "NPM version: $(npm --version)"
    
    logging::log_info "Installing dependencies..."
    npm ci || logging::log_fatal "Failed to install dependencies"
    
    logging::log_info "Running linter..."
    npm run lint || logging::log_fatal "Linting failed"
    
    logging::log_info "Running tests..."
    if ! npm test; then
        logging::log_error "Tests failed"
        logging::log_info "Uploading test artifacts..."
        # Upload test results even on failure
        upload_test_results
        exit 1
    fi
    
    logging::log_info "Building application..."
    npm run build || logging::log_fatal "Build failed"
    
    logging::log_info "Build completed successfully"
}

upload_test_results() {
    if [[ -d "test-results" ]]; then
        logging::log_info "Uploading test results to artifacts"
        # GitHub Actions artifact upload logic
    fi
}

build_application
```

### Docker Build Pipeline

```bash
#!/usr/bin/env bash
# scripts/docker-build.sh
set -Eeuo pipefail

source ./logging.lib.sh
logging::init "docker-build"

readonly IMAGE_NAME="${1:-myapp}"
readonly TAG="${2:-latest}"
readonly DOCKERFILE="${3:-Dockerfile}"

build_docker_image() {
    logging::log_info "Building Docker image: $IMAGE_NAME:$TAG"
    logging::log_info "Using Dockerfile: $DOCKERFILE"
    
    # Check prerequisites
    command -v docker >/dev/null || logging::log_fatal "Docker not found"
    [[ -f "$DOCKERFILE" ]] || logging::log_fatal "Dockerfile not found: $DOCKERFILE"
    
    # Build with progress logging
    logging::log_info "Starting Docker build..."
    if docker build -t "$IMAGE_NAME:$TAG" -f "$DOCKERFILE" . 2>&1 | \
       while IFS= read -r line; do
           logging::log_info "DOCKER: $line"
       done; then
        logging::log_info "Docker build successful"
    else
        logging::log_fatal "Docker build failed"
    fi
    
    # Verify image
    if docker image inspect "$IMAGE_NAME:$TAG" >/dev/null 2>&1; then
        local size
        size=$(docker image inspect "$IMAGE_NAME:$TAG" --format='{{.Size}}' | numfmt --to=iec)
        logging::log_info "Image created successfully, size: $size"
    else
        logging::log_fatal "Image verification failed"
    fi
}

build_docker_image
```

---

## System Administration Scripts

### Backup Script

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

source ./logging.lib.sh
logging::init "backup-script"

readonly BACKUP_SOURCE="${1:-/var/www}"
readonly BACKUP_DEST="${2:-/backup/$(date +%Y%m%d)}"
readonly RETENTION_DAYS="${3:-30}"

perform_backup() {
    logging::log_info "Starting backup process"
    logging::log_info "Source: $BACKUP_SOURCE"
    logging::log_info "Destination: $BACKUP_DEST"
    
    # Pre-backup checks
    check_prerequisites
    create_backup_directory
    
    # Perform backup
    logging::log_info "Creating backup..."
    local start_time
    start_time=$(date +%s)
    
    if rsync -av --progress "$BACKUP_SOURCE/" "$BACKUP_DEST/" 2>&1 | \
       while IFS= read -r line; do
           # Log rsync progress (simplified)
           if [[ "$line" =~ ^[0-9]+% ]]; then
               logging::log_info "Progress: $line"
           fi
       done; then
        local end_time duration
        end_time=$(date +%s)
        duration=$((end_time - start_time))
        logging::log_info "Backup completed in ${duration}s"
    else
        logging::log_fatal "Backup failed"
    fi
    
    # Post-backup tasks
    verify_backup
    cleanup_old_backups
    
    logging::log_info "Backup process completed successfully"
}

check_prerequisites() {
    logging::log_info "Checking prerequisites..."
    
    [[ -d "$BACKUP_SOURCE" ]] || logging::log_fatal "Source directory not found: $BACKUP_SOURCE"
    command -v rsync >/dev/null || logging::log_fatal "rsync not found"
    
    # Check available space
    local available_space
    available_space=$(df "$(dirname "$BACKUP_DEST")" | awk 'NR==2 {print $4}')
    if [[ $available_space -lt 1000000 ]]; then  # Less than 1GB
        logging::log_warn "Low disk space: ${available_space}KB available"
    fi
}

create_backup_directory() {
    if [[ ! -d "$BACKUP_DEST" ]]; then
        logging::log_info "Creating backup directory: $BACKUP_DEST"
        mkdir -p "$BACKUP_DEST" || logging::log_fatal "Failed to create backup directory"
    fi
}

verify_backup() {
    logging::log_info "Verifying backup integrity..."
    
    local source_count backup_count
    source_count=$(find "$BACKUP_SOURCE" -type f | wc -l)
    backup_count=$(find "$BACKUP_DEST" -type f | wc -l)
    
    if [[ $source_count -eq $backup_count ]]; then
        logging::log_info "Backup verification successful: $backup_count files"
    else
        logging::log_error "Backup verification failed: $source_count source files, $backup_count backup files"
    fi
}

cleanup_old_backups() {
    logging::log_info "Cleaning up backups older than $RETENTION_DAYS days..."
    
    local base_dir
    base_dir=$(dirname "$BACKUP_DEST")
    
    find "$base_dir" -type d -name "????????" -mtime +$RETENTION_DAYS | while read -r old_backup; do
        logging::log_info "Removing old backup: $old_backup"
        rm -rf "$old_backup"
    done
}

perform_backup
```

### Service Deployment Script

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

source ./logging.lib.sh
logging::init "deploy-service"

readonly SERVICE_NAME="${1:?Service name required}"
readonly VERSION="${2:?Version required}"
readonly ENVIRONMENT="${3:-staging}"

deploy_service() {
    logging::log_info "Deploying $SERVICE_NAME v$VERSION to $ENVIRONMENT"
    
    # Pre-deployment checks
    validate_environment
    check_service_health
    
    # Deployment steps
    backup_current_version
    download_new_version
    update_configuration
    restart_service
    verify_deployment
    
    logging::log_info "Deployment completed successfully"
}

validate_environment() {
    logging::log_info "Validating deployment environment..."
    
    case "$ENVIRONMENT" in
        staging|production)
            logging::log_info "Deploying to $ENVIRONMENT environment"
            ;;
        *)
            logging::log_fatal "Invalid environment: $ENVIRONMENT"
            ;;
    esac
    
    # Check if service exists
    if ! systemctl list-units --full --all | grep -q "$SERVICE_NAME"; then
        logging::log_fatal "Service not found: $SERVICE_NAME"
    fi
}

check_service_health() {
    logging::log_info "Checking current service health..."
    
    if systemctl is-active --quiet "$SERVICE_NAME"; then
        logging::log_info "Service is currently running"
        
        # Check service endpoint if available
        if curl -sf "http://localhost:8080/health" >/dev/null 2>&1; then
            logging::log_info "Service health check passed"
        else
            logging::log_warn "Service health check failed, proceeding anyway"
        fi
    else
        logging::log_warn "Service is not currently running"
    fi
}

backup_current_version() {
    logging::log_info "Backing up current version..."
    
    local backup_dir="/backup/deployments/$(date +%Y%m%d_%H%M%S)"
    mkdir -p "$backup_dir"
    
    cp -r "/opt/$SERVICE_NAME" "$backup_dir/" || {
        logging::log_error "Backup failed, but continuing deployment"
    }
}

download_new_version() {
    logging::log_info "Downloading version $VERSION..."
    
    local download_url="https://releases.example.com/$SERVICE_NAME/$VERSION.tar.gz"
    local temp_file="/tmp/$SERVICE_NAME-$VERSION.tar.gz"
    
    if curl -L -o "$temp_file" "$download_url"; then
        logging::log_info "Download completed"
        
        # Verify checksum if available
        if curl -sf "${download_url}.sha256" >/dev/null 2>&1; then
            logging::log_info "Verifying checksum..."
            cd "$(dirname "$temp_file")"
            curl -L "${download_url}.sha256" | sha256sum -c || logging::log_fatal "Checksum verification failed"
        fi
        
        # Extract
        logging::log_info "Extracting archive..."
        tar -xzf "$temp_file" -C "/opt/" || logging::log_fatal "Extraction failed"
        rm "$temp_file"
    else
        logging::log_fatal "Download failed: $download_url"
    fi
}

update_configuration() {
    logging::log_info "Updating configuration..."
    
    local config_file="/opt/$SERVICE_NAME/config.yml"
    if [[ -f "$config_file.template" ]]; then
        envsubst < "$config_file.template" > "$config_file"
        logging::log_info "Configuration updated from template"
    fi
}

restart_service() {
    logging::log_info "Restarting service..."
    
    systemctl restart "$SERVICE_NAME" || logging::log_fatal "Failed to restart service"
    
    # Wait for service to be ready
    local attempts=30
    while [[ $attempts -gt 0 ]]; do
        if systemctl is-active --quiet "$SERVICE_NAME"; then
            logging::log_info "Service started successfully"
            return 0
        fi
        logging::log_info "Waiting for service to start... ($attempts attempts remaining)"
        sleep 2
        ((attempts--))
    done
    
    logging::log_fatal "Service failed to start within timeout"
}

verify_deployment() {
    logging::log_info "Verifying deployment..."
    
    # Check service status
    systemctl status "$SERVICE_NAME" --no-pager || true
    
    # Check application health
    sleep 5  # Give app time to initialize
    if curl -sf "http://localhost:8080/health" >/dev/null 2>&1; then
        logging::log_info "Health check passed"
    else
        logging::log_error "Health check failed"
        systemctl stop "$SERVICE_NAME"
        logging::log_fatal "Deployment verification failed"
    fi
    
    # Check version endpoint
    local reported_version
    if reported_version=$(curl -sf "http://localhost:8080/version" 2>/dev/null); then
        if [[ "$reported_version" == "$VERSION" ]]; then
            logging::log_info "Version verification passed: $reported_version"
        else
            logging::log_warn "Version mismatch: expected $VERSION, got $reported_version"
        fi
    fi
}

deploy_service
```

---

## Multi-Script Coordination

### Orchestration Script

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

source ./logging.lib.sh
logging::init "orchestrator"

readonly SCRIPTS_DIR="./scripts"
readonly PARALLEL_JOBS=3

run_pipeline() {
    logging::log_info "Starting pipeline orchestration"
    
    # Phase 1: Preparation (sequential)
    run_phase "preparation" "prepare-environment.sh" "download-dependencies.sh"
    
    # Phase 2: Processing (parallel)
    run_phase_parallel "processing" "process-data-1.sh" "process-data-2.sh" "process-data-3.sh"
    
    # Phase 3: Cleanup (sequential)
    run_phase "cleanup" "generate-reports.sh" "cleanup-temp.sh"
    
    logging::log_info "Pipeline completed successfully"
}

run_phase() {
    local phase_name="$1"
    shift
    
    logging::log_info "Starting phase: $phase_name"
    
    for script in "$@"; do
        run_script "$script"
    done
    
    logging::log_info "Phase completed: $phase_name"
}

run_phase_parallel() {
    local phase_name="$1"
    shift
    
    logging::log_info "Starting parallel phase: $phase_name"
    
    # Create temporary directory for job tracking
    local job_dir
    job_dir=$(mktemp -d)
    local -a pids=()
    
    # Start jobs
    for script in "$@"; do
        (
            run_script "$script"
            echo $? > "$job_dir/$(basename "$script").exit"
        ) &
        pids+=($!)
        
        # Limit concurrent jobs
        if [[ ${#pids[@]} -ge $PARALLEL_JOBS ]]; then
            wait_for_job pids "$job_dir"
        fi
    done
    
    # Wait for remaining jobs
    while [[ ${#pids[@]} -gt 0 ]]; do
        wait_for_job pids "$job_dir"
    done
    
    # Check all exit codes
    local failed_scripts=()
    for script in "$@"; do
        local exit_file="$job_dir/$(basename "$script").exit"
        if [[ -f "$exit_file" ]]; then
            local exit_code
            exit_code=$(cat "$exit_file")
            if [[ $exit_code -ne 0 ]]; then
                failed_scripts+=("$script")
            fi
        fi
    done
    
    rm -rf "$job_dir"
    
    if [[ ${#failed_scripts[@]} -gt 0 ]]; then
        logging::log_fatal "Failed scripts in $phase_name: ${failed_scripts[*]}"
    fi
    
    logging::log_info "Parallel phase completed: $phase_name"
}

wait_for_job() {
    local -n pids_ref=$1
    local job_dir="$2"
    
    # Wait for first job to complete
    wait "${pids_ref[0]}"
    
    # Remove completed job from array
    pids_ref=("${pids_ref[@]:1}")
}

run_script() {
    local script="$1"
    local script_path="$SCRIPTS_DIR/$script"
    
    if [[ ! -f "$script_path" ]]; then
        logging::log_fatal "Script not found: $script_path"
    fi
    
    if [[ ! -x "$script_path" ]]; then
        logging::log_fatal "Script not executable: $script_path"
    fi
    
    logging::log_info "Executing: $script"
    
    # Execute script with timeout
    if timeout 300 "$script_path" 2>&1 | while IFS= read -r line; do
        logging::log_info "[$script] $line"
    done; then
        logging::log_info "Script completed: $script"
    else
        local exit_code=${PIPESTATUS[0]}
        if [[ $exit_code -eq 124 ]]; then
            logging::log_error "Script timed out: $script"
        else
            logging::log_error "Script failed with exit code $exit_code: $script"
        fi
        return $exit_code
    fi
}

run_pipeline
```

---

## Best Practices Examples

### Error Recovery and Resilience

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

source ./logging.lib.sh
logging::init "resilient-script"

# Circuit breaker pattern
declare -g FAILURE_COUNT=0
declare -g MAX_FAILURES=5
declare -g CIRCUIT_OPEN=false

call_external_service() {
    if [[ "$CIRCUIT_OPEN" == "true" ]]; then
        logging::log_warn "Circuit breaker open, skipping service call"
        return 1
    fi
    
    if external_service_call; then
        # Reset on success
        FAILURE_COUNT=0
        logging::log_info "Service call successful"
        return 0
    else
        ((FAILURE_COUNT++))
        logging::log_error "Service call failed (failure count: $FAILURE_COUNT)"
        
        if [[ $FAILURE_COUNT -ge $MAX_FAILURES ]]; then
            CIRCUIT_OPEN=true
            logging::log_warn "Circuit breaker opened after $MAX_FAILURES failures"
        fi
        return 1
    fi
}

# Graceful degradation
process_with_fallback() {
    local data="$1"
    
    if ! primary_processor "$data"; then
        logging::log_warn "Primary processor failed, trying secondary"
        
        if ! secondary_processor "$data"; then
            logging::log_warn "Secondary processor failed, using basic fallback"
            basic_processor "$data"
        fi
    fi
}

# Resource cleanup
cleanup_resources() {
    logging::log_info "Cleaning up resources..."
    
    # Remove temporary files
    if [[ -n "${TEMP_DIR:-}" && -d "$TEMP_DIR" ]]; then
        rm -rf "$TEMP_DIR"
        logging::log_info "Removed temporary directory: $TEMP_DIR"
    fi
    
    # Close file descriptors
    exec 3>&- 2>/dev/null || true
    exec 4>&- 2>/dev/null || true
    
    # Kill background processes
    if [[ -n "${BG_PID:-}" ]]; then
        kill "$BG_PID" 2>/dev/null || true
        logging::log_info "Terminated background process: $BG_PID"
    fi
}

# Install cleanup trap
trap cleanup_resources EXIT
```

### Configuration and Environment Management

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

source ./logging.lib.sh
logging::init "config-manager"

# Configuration loading with validation
load_configuration() {
    local config_file="${1:-config.env}"
    
    if [[ ! -f "$config_file" ]]; then
        logging::log_fatal "Configuration file not found: $config_file"
    fi
    
    logging::log_info "Loading configuration from: $config_file"
    
    # Source configuration
    set -a  # Export all variables
    source "$config_file"
    set +a
    
    # Validate required variables
    validate_required_config
    
    # Log configuration (without sensitive values)
    log_configuration
}

validate_required_config() {
    local required_vars=(
        "DATABASE_URL"
        "API_KEY"
        "SERVICE_PORT"
        "LOG_LEVEL"
    )
    
    logging::log_info "Validating configuration..."
    
    for var in "${required_vars[@]}"; do
        if [[ -z "${!var:-}" ]]; then
            logging::log_fatal "Required configuration missing: $var"
        fi
    done
    
    # Validate specific formats
    if [[ ! "$SERVICE_PORT" =~ ^[0-9]+$ ]] || [[ "$SERVICE_PORT" -lt 1 || "$SERVICE_PORT" -gt 65535 ]]; then
        logging::log_fatal "Invalid SERVICE_PORT: $SERVICE_PORT"
    fi
    
    if [[ ! "$LOG_LEVEL" =~ ^(DEBUG|INFO|WARN|ERROR)$ ]]; then
        logging::log_fatal "Invalid LOG_LEVEL: $LOG_LEVEL"
    fi
    
    logging::log_info "Configuration validation passed"
}

log_configuration() {
    logging::log_info "Active configuration:"
    logging::log_info "  SERVICE_PORT=$SERVICE_PORT"
    logging::log_info "  LOG_LEVEL=$LOG_LEVEL"
    logging::log_info "  DATABASE_URL=${DATABASE_URL/password=*/password=***}"
    logging::log_info "  API_KEY=${API_KEY:0:8}..."
}

# Environment-specific handling
setup_environment() {
    local environment="${ENVIRONMENT:-development}"
    
    case "$environment" in
        development)
            export DEBUG=true
            export LOG_LEVEL=DEBUG
            logging::log_info "Development environment configured"
            ;;
        staging)
            export DEBUG=false
            export LOG_LEVEL=INFO
            logging::log_info "Staging environment configured"
            ;;
        production)
            export DEBUG=false
            export LOG_LEVEL=WARN
            logging::log_info "Production environment configured"
            ;;
        *)
            logging::log_fatal "Unknown environment: $environment"
            ;;
    esac
}
```

These examples demonstrate production-ready patterns that leverage the full power of the Bashing Logs library, from simple scripts to complex orchestration systems.
