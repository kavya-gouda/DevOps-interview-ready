# Shell Scripting — Solutions

> Full solutions with explanation, gotchas, and what the interviewer is watching for.
> Only read AFTER you've attempted the exercise yourself.

---

## E-001 | Safe Argument Validation

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    cat >&2 <<EOF
Usage: $(basename "$0") <namespace> <deployment>

Arguments:
  namespace    Kubernetes namespace
  deployment   Deployment name

Example:
  $(basename "$0") production frontend
EOF
    exit 1
}

[[ $# -lt 2 ]] && usage

NAMESPACE="$1"
DEPLOYMENT="$2"

echo "Checking: ${NAMESPACE}/${DEPLOYMENT}"
```

**What interviewer watches for:**
- `usage()` printed to **stderr** (`>&2`) not stdout — stdout is for data, stderr is for errors/usage
- Check `$#` before accessing `$1`/`$2` — never assume args exist
- Clear example in usage message — signals production awareness

---

## E-002 | Check Required Environment Variables

```bash
#!/usr/bin/env bash
set -euo pipefail

require_env() {
    local missing=()
    for var in "$@"; do
        if [[ -z "${!var:-}" ]]; then
            missing+=("$var")
        fi
    done
    if [[ ${#missing[@]} -gt 0 ]]; then
        echo "ERROR: Required environment variables not set:" >&2
        printf '  - %s\n' "${missing[@]}" >&2
        exit 1
    fi
}

require_env DATABASE_URL REDIS_URL JWT_SECRET
echo "All required variables present."
```

**Key concepts:**
- `${!var}` — indirect expansion: expand the variable whose NAME is stored in `var`
- `${!var:-}` — indirect + default empty string to avoid `-u` error on unset
- Collect ALL missing vars before exiting — better UX than stopping at first missing one
- `printf '  - %s\n' "${missing[@]}"` — safe way to print array with formatting

---

## E-003 | Retry with Backoff

```bash
#!/usr/bin/env bash
set -euo pipefail

retry() {
    local max_attempts="${1:?max_attempts required}"
    shift
    local attempt=1
    local delay=1

    while (( attempt <= max_attempts )); do
        if "$@"; then
            return 0
        fi
        if (( attempt == max_attempts )); then
            echo "ERROR: Command failed after ${max_attempts} attempts: $*" >&2
            return 1
        fi
        echo "Attempt ${attempt}/${max_attempts} failed. Retrying in ${delay}s..." >&2
        sleep "$delay"
        delay=$(( delay * 2 ))
        (( attempt++ )) || true
    done
}

# Usage
retry 5 curl -sf https://api.example.com/health
```

**Key concepts:**
- `shift` after capturing first arg — remaining `"$@"` is the command to run
- `"$@"` preserves spaces in arguments — never use `$*` for executing commands
- `(( attempt++ )) || true` — `(( ))` returns 1 when result is 0 (falsy in arithmetic), need `|| true` with `set -e`
- Print retry messages to **stderr** so they don't pollute stdout data

---

## E-004 | Safe Temp File Usage

```bash
#!/usr/bin/env bash
set -euo pipefail

TMPFILE=""

cleanup() {
    [[ -n "$TMPFILE" && -f "$TMPFILE" ]] && rm -f "$TMPFILE"
}
trap cleanup EXIT

TMPFILE=$(mktemp)
echo "Working with temp file: $TMPFILE"

# Do work
some_command > "$TMPFILE"
process_results "$TMPFILE"

echo "Done."
# cleanup() runs automatically on EXIT
```

**Why `TMPFILE=""` before the trap is set:**
If `mktemp` fails, `TMPFILE` is still empty. The cleanup checks `[[ -n "$TMPFILE" ]]` to avoid `rm -f ""` which would be `rm -f` with no argument.

**Why `trap cleanup EXIT` not `trap cleanup INT TERM`:**
`EXIT` fires on ALL exits — normal, error (`set -e`), and signals. One trap covers everything.

---

## E-005 | Atomic Config Write

```bash
#!/usr/bin/env bash
set -euo pipefail

write_config() {
    local target="${1:?target file required}"
    local tmpfile
    tmpfile=$(mktemp "${target}.tmp.XXXXXX")

    # Trap to clean up temp file if write fails
    local cleanup_tmp() { rm -f "$tmpfile"; }
    trap cleanup_tmp RETURN  # fires when function returns (success or failure)

    # Write to temp file
    cat > "$tmpfile"

    # Validate before replacing (optional but good practice)
    # e.g., for YAML: yq eval '.' "$tmpfile" > /dev/null

    # Atomic rename — either old or new, never partial
    mv "$tmpfile" "$target"
}

# Usage
generate_config | write_config "/etc/myapp/config.yaml"
```

**Why `mv` is atomic:**
On the same filesystem, `mv` is a single `rename()` syscall. It's atomic — a reader will see either the old file or the new file, never a partial write.

**Cross-filesystem moves are NOT atomic** — if `/tmp` and `/etc` are on different filesystems, `mktemp` must create the temp file next to the target: `mktemp "${target}.tmp.XXXXXX"`.

---

## E-006 | Parse CSV and Sum a Column

```bash
#!/usr/bin/env bash
set -euo pipefail

CSV_FILE="${1:?Usage: $0 <csv_file>}"
[[ ! -f "$CSV_FILE" ]] && { echo "ERROR: File not found: $CSV_FILE" >&2; exit 1; }

awk -F',' '
NR == 1 { next }   # skip header
{
    total += $3
    name[NR] = $1
    cost[NR] = $3
}
END {
    printf "Total: %.2f\n\n", total
    print "Top 3:"

    # Build sorted index (simple bubble sort for small N)
    n = NR - 1
    for (i = 2; i <= NR; i++) idx[i-1] = i
    for (i = 1; i <= n; i++)
        for (j = i+1; j <= n; j++)
            if (cost[idx[i]] < cost[idx[j]]) {
                tmp = idx[i]; idx[i] = idx[j]; idx[j] = tmp
            }

    for (i = 1; i <= (n < 3 ? n : 3); i++)
        printf "  %s: %.2f\n", name[idx[i]], cost[idx[i]]
}
' "$CSV_FILE"
```

**Simpler pipeline approach (more readable):**
```bash
# Sum
awk -F',' 'NR>1 {sum+=$3} END {printf "Total: %.2f\n", sum}' "$CSV_FILE"

# Top 3
awk -F',' 'NR>1 {print $3, $1}' "$CSV_FILE" | \
    sort -rn | \
    head -3 | \
    awk '{printf "  %s: %s\n", $2, $1}'
```

**What interviewer watches for:** Do you skip the header (`NR>1`)? Do you format floats with `printf`?

---

## E-007 | Process Log File — Extract Error Summary

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <log_file>}"
[[ ! -f "$LOG_FILE" ]] && { echo "ERROR: File not found" >&2; exit 1; }

awk '
{
    # Field: $1=date $2=time $3=level, rest=message
    level = $3
    count[level]++

    if (level == "ERROR") {
        # Reconstruct message from field 4 onward
        msg = ""
        for (i=4; i<=NF; i++) msg = msg (i>4 ? " " : "") $i
        errors[++err_count] = $1 " " $2 " — " msg
    }
}
END {
    print "Log Level Summary:"
    for (l in count) printf "  %-8s %d\n", l":", count[l]

    if (err_count > 0) {
        print "\nERROR lines:"
        for (i=1; i<=err_count; i++) print "  " errors[i]
    }
}
' "$LOG_FILE"
```

---

## E-008 | Rotate and Compress Logs

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_DIR="${1:?Usage: $0 <log_dir> [days]}"
DAYS="${2:-7}"
COMPRESSED=0
FAILED=0
BYTES_BEFORE=0
BYTES_AFTER=0

[[ ! -d "$LOG_DIR" ]] && { echo "ERROR: Directory not found: $LOG_DIR" >&2; exit 1; }

while IFS= read -r -d '' logfile; do
    size_before=$(stat -c%s "$logfile" 2>/dev/null || stat -f%z "$logfile")
    BYTES_BEFORE=$(( BYTES_BEFORE + size_before ))

    if gzip -9 "$logfile" 2>/dev/null; then
        size_after=$(stat -c%s "${logfile}.gz" 2>/dev/null || stat -f%z "${logfile}.gz")
        BYTES_AFTER=$(( BYTES_AFTER + size_after ))
        (( COMPRESSED++ )) || true
    else
        echo "WARN: Failed to compress: $logfile" >&2
        (( FAILED++ )) || true
        BYTES_AFTER=$(( BYTES_AFTER + size_before ))
    fi
done < <(find "$LOG_DIR" -name "*.log" -mtime "+${DAYS}" -print0)

SAVED=$(( BYTES_BEFORE - BYTES_AFTER ))
echo "Compressed: ${COMPRESSED} file(s)"
echo "Failed:     ${FAILED} file(s)"
echo "Space saved: $(( SAVED / 1024 / 1024 )) MB"
```

**Key:** `-print0` + `read -r -d ''` — the only safe way to handle filenames with spaces/newlines.

---

## E-009 | Parallel Host Health Check

```bash
#!/usr/bin/env bash
set -euo pipefail

HOSTS_FILE="${1:?Usage: $0 <hosts_file>}"
SSH_TIMEOUT="${SSH_TIMEOUT:-5}"
PARALLEL="${PARALLEL:-20}"

[[ ! -f "$HOSTS_FILE" ]] && { echo "ERROR: File not found: $HOSTS_FILE" >&2; exit 1; }

check_host() {
    local host="$1"
    if ssh \
        -o ConnectTimeout="${SSH_TIMEOUT}" \
        -o StrictHostKeyChecking=no \
        -o BatchMode=yes \
        "$host" uptime 2>/dev/null; then
        echo "OK:   $host"
        return 0
    else
        echo "FAIL: $host" >&2
        return 1
    fi
}
export -f check_host
export SSH_TIMEOUT

# Parallel execution
mapfile -t hosts < "$HOSTS_FILE"
pids=()
results=()

# Throttle to PARALLEL concurrent jobs
active=0
failed=0

for host in "${hosts[@]}"; do
    check_host "$host" &
    pids+=($!)
    (( active++ )) || true

    if (( active >= PARALLEL )); then
        pid="${pids[0]}"
        pids=("${pids[@]:1}")
        wait "$pid" || (( failed++ )) || true
        (( active-- )) || true
    fi
done

for pid in "${pids[@]}"; do
    wait "$pid" || (( failed++ )) || true
done

echo "---"
echo "Failed: $failed / ${#hosts[@]}"
[[ $failed -gt 0 ]] && exit 1 || exit 0
```

---

## E-010 | Lock File — Prevent Concurrent Runs

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCK_FILE="/var/lock/$(basename "$0" .sh).lock"

acquire_lock() {
    exec 9>"$LOCK_FILE"

    if ! flock -n 9; then
        # Lock held — check if holder is still alive
        local holder_pid
        holder_pid=$(cat "$LOCK_FILE" 2>/dev/null || echo "")
        if [[ -n "$holder_pid" ]] && kill -0 "$holder_pid" 2>/dev/null; then
            echo "ERROR: Already running (PID: $holder_pid)" >&2
            exit 1
        else
            echo "WARN: Stale lock found (PID $holder_pid gone). Acquiring..." >&2
            flock -f 9  # blocking acquire of stale lock
        fi
    fi

    echo $$ > "$LOCK_FILE"
}

release_lock() {
    flock -u 9
    rm -f "$LOCK_FILE"
}

trap release_lock EXIT
acquire_lock

echo "Running exclusively (PID: $$)"
# ... your work here ...
```

---

## E-015 | Kubernetes Namespace Report

```bash
#!/usr/bin/env bash
set -euo pipefail

printf "%-20s %6s %8s %10s %10s\n" "NAMESPACE" "PODS" "RUNNING" "CPU_REQ" "MEM_REQ"
printf "%-20s %6s %8s %10s %10s\n" "$(printf '%0.s-' {1..20})" "------" "--------" "----------" "----------"

kubectl get pods --all-namespaces -o json | jq -r '
  .items |
  group_by(.metadata.namespace) |
  .[] |
  {
    ns: .[0].metadata.namespace,
    total: length,
    running: [.[] | select(.status.phase == "Running")] | length,
    cpu: [.[] | .spec.containers[].resources.requests.cpu // "0m"] |
         map(if test("m$") then (.[:-1] | tonumber) else (tonumber * 1000) end) | add,
    mem: [.[] | .spec.containers[].resources.requests.memory // "0Mi"] |
         map(if test("Mi$") then (.[:-2] | tonumber)
             elif test("Gi$") then (.[:-2] | tonumber * 1024)
             else 0 end) | add
  } |
  [.ns, (.total|tostring), (.running|tostring),
   ((.cpu|tostring) + "m"), ((.mem|tostring) + "Mi")] |
  @tsv
' | while IFS=$'\t' read -r ns total running cpu mem; do
    printf "%-20s %6s %8s %10s %10s\n" "$ns" "$total" "$running" "$cpu" "$mem"
done
```

---

## E-026 | Deployment Health Validator

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    echo "Usage: $(basename "$0") -n <namespace> -d <deployment> [-r <max_restarts>] [-t <timeout>]" >&2
    exit 1
}

NAMESPACE="" DEPLOYMENT="" MAX_RESTARTS=5 TIMEOUT=120
while getopts "n:d:r:t:" opt; do
    case $opt in
        n) NAMESPACE="$OPTARG" ;;
        d) DEPLOYMENT="$OPTARG" ;;
        r) MAX_RESTARTS="$OPTARG" ;;
        t) TIMEOUT="$OPTARG" ;;
        *) usage ;;
    esac
done
[[ -z "$NAMESPACE" || -z "$DEPLOYMENT" ]] && usage

FAILED=()

# 1. Check rollout status
echo "Checking rollout status..."
if ! kubectl rollout status deployment/"$DEPLOYMENT" -n "$NAMESPACE" --timeout="${TIMEOUT}s" 2>&1; then
    FAILED+=("Rollout did not complete within ${TIMEOUT}s")
fi

# 2. Check all pods are Running
echo "Checking pod phases..."
NOT_RUNNING=$(kubectl get pods -n "$NAMESPACE" \
    -l "$(kubectl get deployment "$DEPLOYMENT" -n "$NAMESPACE" -o jsonpath='{.spec.selector.matchLabels}' | \
    jq -r 'to_entries | map("\(.key)=\(.value)") | join(",")')" \
    --no-headers 2>/dev/null | grep -v "Running" | wc -l | tr -d ' ')

[[ "$NOT_RUNNING" -gt 0 ]] && FAILED+=("$NOT_RUNNING pod(s) not in Running state")

# 3. Check restart counts
echo "Checking restart counts..."
HIGH_RESTARTS=$(kubectl get pods -n "$NAMESPACE" -o json | jq -r --argjson max "$MAX_RESTARTS" '
    .items[].status.containerStatuses[]? |
    select(.restartCount > $max) |
    "\(.name): \(.restartCount) restarts"
')
[[ -n "$HIGH_RESTARTS" ]] && FAILED+=("High restart counts: $HIGH_RESTARTS")

# Report
if [[ ${#FAILED[@]} -gt 0 ]]; then
    echo "" >&2
    echo "VALIDATION FAILED:" >&2
    printf '  ✗ %s\n' "${FAILED[@]}" >&2
    exit 1
else
    echo ""
    echo "✓ All health checks passed for ${NAMESPACE}/${DEPLOYMENT}"
fi
```

---

## E-030 | Canary Validation Script

```bash
#!/usr/bin/env bash
set -euo pipefail

PROMETHEUS="${PROMETHEUS_URL:-http://prometheus:9090}"
ERROR_RATE_THRESHOLD=1.0
P99_THRESHOLD=500
MIN_AGE=300
NAMESPACE=""
DEPLOYMENT=""

while [[ $# -gt 0 ]]; do
    case "$1" in
        --error-rate) ERROR_RATE_THRESHOLD="$2"; shift 2 ;;
        --p99-latency) P99_THRESHOLD="$2"; shift 2 ;;
        --min-age) MIN_AGE="$2"; shift 2 ;;
        --namespace) NAMESPACE="$2"; shift 2 ;;
        --deployment) DEPLOYMENT="$2"; shift 2 ;;
        *) echo "Unknown: $1" >&2; exit 1 ;;
    esac
done

query_prometheus() {
    local query="$1"
    local result
    result=$(curl -sf --max-time 10 \
        "${PROMETHEUS}/api/v1/query" \
        --data-urlencode "query=${query}" | \
        jq -r '.data.result[0].value[1] // "NaN"')
    echo "$result"
}

compare_float() {
    awk "BEGIN { exit !($1 $2 $3) }"
}

FAILED=()

# 1. Error rate
echo "Checking error rate..."
ERROR_QUERY="sum(rate(http_requests_total{namespace=\"${NAMESPACE}\",deployment=\"${DEPLOYMENT}\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{namespace=\"${NAMESPACE}\",deployment=\"${DEPLOYMENT}\"}[5m])) * 100"
ERROR_RATE=$(query_prometheus "$ERROR_QUERY")

if [[ "$ERROR_RATE" == "NaN" ]]; then
    FAILED+=("Could not fetch error rate from Prometheus")
elif compare_float "$ERROR_RATE" ">" "$ERROR_RATE_THRESHOLD"; then
    FAILED+=("Error rate ${ERROR_RATE}% exceeds threshold ${ERROR_RATE_THRESHOLD}%")
else
    echo "  ✓ Error rate: ${ERROR_RATE}% (threshold: ${ERROR_RATE_THRESHOLD}%)"
fi

# 2. P99 latency
echo "Checking P99 latency..."
P99_QUERY="histogram_quantile(0.99, sum(rate(http_request_duration_milliseconds_bucket{namespace=\"${NAMESPACE}\",deployment=\"${DEPLOYMENT}\"}[5m])) by (le))"
P99=$(query_prometheus "$P99_QUERY")

if [[ "$P99" != "NaN" ]] && compare_float "$P99" ">" "$P99_THRESHOLD"; then
    FAILED+=("P99 latency ${P99}ms exceeds threshold ${P99_THRESHOLD}ms")
else
    echo "  ✓ P99 latency: ${P99}ms (threshold: ${P99_THRESHOLD}ms)"
fi

# 3. Minimum age
echo "Checking canary age..."
DEPLOY_TS=$(kubectl get deployment "$DEPLOYMENT" -n "$NAMESPACE" \
    -o jsonpath='{.metadata.creationTimestamp}' 2>/dev/null)
DEPLOY_EPOCH=$(date -d "$DEPLOY_TS" +%s 2>/dev/null || date -j -f "%Y-%m-%dT%H:%M:%SZ" "$DEPLOY_TS" +%s)
AGE=$(( $(date +%s) - DEPLOY_EPOCH ))

if (( AGE < MIN_AGE )); then
    FAILED+=("Canary age ${AGE}s is less than minimum ${MIN_AGE}s")
else
    echo "  ✓ Canary age: ${AGE}s (minimum: ${MIN_AGE}s)"
fi

# Final result
if [[ ${#FAILED[@]} -gt 0 ]]; then
    echo "" >&2
    echo "CANARY VALIDATION FAILED:" >&2
    printf '  ✗ %s\n' "${FAILED[@]}" >&2
    exit 1
fi

echo ""
echo "✓ Canary validation passed. Safe to promote."
```

---

## E-031 | Certificate Expiry Monitor

```bash
#!/usr/bin/env bash
set -euo pipefail

WARN_DAYS=30
CRITICAL_DAYS=14
DOMAINS=()
EXIT_CODE=0

while [[ $# -gt 0 ]]; do
    case "$1" in
        --warn-days)     WARN_DAYS="$2";     shift 2 ;;
        --critical-days) CRITICAL_DAYS="$2"; shift 2 ;;
        *) DOMAINS+=("$1"); shift ;;
    esac
done

[[ ${#DOMAINS[@]} -eq 0 ]] && { echo "Usage: $0 [--warn-days N] [--critical-days N] domain..." >&2; exit 1; }

check_cert() {
    local domain="$1"
    local expiry days_left status

    expiry=$(echo | timeout 5 openssl s_client -servername "$domain" \
        -connect "${domain}:443" 2>/dev/null | \
        openssl x509 -noout -enddate 2>/dev/null | \
        cut -d= -f2)

    if [[ -z "$expiry" ]]; then
        printf "%-8s %-40s %s\n" "ERROR" "$domain" "Could not retrieve certificate"
        return 1
    fi

    days_left=$(( ( $(date -d "$expiry" +%s 2>/dev/null || date -j -f "%b %d %T %Y %Z" "$expiry" +%s) - $(date +%s) ) / 86400 ))

    if (( days_left < CRITICAL_DAYS )); then
        status="CRITICAL"
    elif (( days_left < WARN_DAYS )); then
        status="WARNING"
    else
        status="OK"
    fi

    printf "%-8s %-40s %d days (%s)\n" "$status" "$domain" "$days_left" "$expiry"
    [[ "$status" == "CRITICAL" ]] && return 1 || return 0
}

export -f check_cert
export WARN_DAYS CRITICAL_DAYS

for domain in "${DOMAINS[@]}"; do
    check_cert "$domain" || EXIT_CODE=1
done

exit $EXIT_CODE
```

---

## E-037 | Pipeline Failure Notifier

```bash
#!/usr/bin/env bash
set -euo pipefail

# Works with GitHub Actions or GitLab CI environment variables
PIPELINE_STATUS="${1:-${CI_JOB_STATUS:-unknown}}"

[[ "$PIPELINE_STATUS" == "success" ]] && exit 0

WEBHOOK_URL="${SLACK_WEBHOOK_URL:?SLACK_WEBHOOK_URL not set}"

# GitHub Actions vars → GitLab CI fallbacks
REPO="${GITHUB_REPOSITORY:-${CI_PROJECT_PATH:-unknown}}"
BRANCH="${GITHUB_REF_NAME:-${CI_COMMIT_REF_NAME:-unknown}}"
COMMIT="${GITHUB_SHA:-${CI_COMMIT_SHA:-unknown}}"
COMMIT_SHORT="${COMMIT:0:8}"
ACTOR="${GITHUB_ACTOR:-${GITLAB_USER_LOGIN:-unknown}}"
PIPELINE_URL="${GITHUB_SERVER_URL:-https://gitlab.com}/${REPO}/actions/runs/${GITHUB_RUN_ID:-${CI_PIPELINE_ID:-}}"

PAYLOAD=$(jq -n \
    --arg repo "$REPO" \
    --arg branch "$BRANCH" \
    --arg commit "$COMMIT_SHORT" \
    --arg actor "$ACTOR" \
    --arg url "$PIPELINE_URL" \
    '{
        text: ":x: *Pipeline Failed*",
        attachments: [{
            color: "danger",
            fields: [
                {title: "Repository", value: $repo, short: true},
                {title: "Branch",     value: $branch, short: true},
                {title: "Commit",     value: $commit, short: true},
                {title: "Triggered by", value: $actor, short: true}
            ],
            actions: [{
                type: "button",
                text: "View Pipeline",
                url: $url
            }]
        }]
    }')

curl -sf \
    -H "Content-Type: application/json" \
    -d "$PAYLOAD" \
    "$WEBHOOK_URL"

echo "Failure notification sent."
```

---

## E-035 | Infrastructure Drift Detector

```bash
#!/usr/bin/env bash
set -euo pipefail

CONFIG_FILE="${1:?Usage: $0 <config_file> [--fix]}"
FIX_MODE=false
[[ "${2:-}" == "--fix" ]] && FIX_MODE=true

DRIFT_COUNT=0

while IFS='=' read -r resource expected || [[ -n "$resource" ]]; do
    # Skip comments and empty lines
    [[ "$resource" =~ ^#|^[[:space:]]*$ ]] && continue

    namespace=$(cut -d'/' -f1 <<< "$resource")
    deployment=$(cut -d'/' -f2 <<< "$resource")
    expected_replicas=$(cut -d':' -f1 <<< "$expected")
    expected_tag=$(cut -d':' -f2 <<< "$expected")

    # Get actual state
    actual=$(kubectl get deployment "$deployment" -n "$namespace" \
        -o json 2>/dev/null || echo "{}")

    actual_replicas=$(jq -r '.spec.replicas // "MISSING"' <<< "$actual")
    actual_tag=$(jq -r '.spec.template.spec.containers[0].image // "MISSING"' <<< "$actual" | cut -d':' -f2)

    replica_drift=false
    tag_drift=false

    [[ "$actual_replicas" != "$expected_replicas" ]] && replica_drift=true
    [[ "$actual_tag" != "$expected_tag" ]] && tag_drift=true

    if $replica_drift || $tag_drift; then
        echo "DRIFT: ${namespace}/${deployment}"
        $replica_drift && echo "  replicas: expected=${expected_replicas} actual=${actual_replicas}"
        $tag_drift     && echo "  image tag: expected=${expected_tag} actual=${actual_tag}"
        (( DRIFT_COUNT++ )) || true

        if $FIX_MODE && $replica_drift; then
            echo "  → Fixing replica count..."
            kubectl scale deployment "$deployment" -n "$namespace" --replicas="$expected_replicas"
        fi
    else
        echo "OK:    ${namespace}/${deployment}"
    fi
done < "$CONFIG_FILE"

echo "---"
echo "Drift found in ${DRIFT_COUNT} deployment(s)"
[[ $DRIFT_COUNT -gt 0 ]] && exit 1 || exit 0
```

---

## Key Patterns to Memorize

### Pattern 1 — The Safe Script Template
```bash
#!/usr/bin/env bash
set -euo pipefail

usage() { echo "Usage: $0 <arg1> [arg2]" >&2; exit 1; }
[[ $# -lt 1 ]] && usage

cleanup() { rm -f "${TMPFILE:-}"; }
trap cleanup EXIT

TMPFILE=$(mktemp)
```

### Pattern 2 — Read File Safely
```bash
while IFS= read -r line || [[ -n "$line" ]]; do
    echo "$line"
done < "$FILE"
```

### Pattern 3 — Parallel Jobs with Exit Code Tracking
```bash
pids=(); failed=0
for item in "${items[@]}"; do
    process "$item" &
    pids+=($!)
done
for pid in "${pids[@]}"; do
    wait "$pid" || (( failed++ )) || true
done
[[ $failed -gt 0 ]] && exit 1
```

### Pattern 4 — Float Comparison
```bash
compare_float() { awk "BEGIN { exit !($1 $2 $3) }"; }
compare_float "3.14" ">" "2.71" && echo "yes"
```

### Pattern 5 — Kubernetes JSON + jq
```bash
kubectl get pods --all-namespaces -o json | jq -r '
    .items[] |
    select(.status.phase != "Running") |
    [.metadata.namespace, .metadata.name, .status.phase] |
    @tsv
' | column -t
```

### Pattern 6 — Atomic File Write
```bash
tmpfile=$(mktemp "${target}.XXXXXX")
trap "rm -f $tmpfile" EXIT
generate_content > "$tmpfile"
mv "$tmpfile" "$target"
```

### Pattern 7 — getopts Argument Parsing
```bash
while getopts "n:d:t:h" opt; do
    case $opt in
        n) NAMESPACE="$OPTARG" ;;
        d) DEPLOYMENT="$OPTARG" ;;
        t) TIMEOUT="${OPTARG}" ;;
        h|*) usage ;;
    esac
done
shift $(( OPTIND - 1 ))
```

---

## Section 5: Log File Analysis — Solutions

---

## L-001 | Count HTTP Status Codes

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <access_log>}"
[[ ! -f "$LOG_FILE" ]] && { echo "ERROR: File not found: $LOG_FILE" >&2; exit 1; }

awk '{print $9}' "$LOG_FILE" | sort | uniq -c | sort -rn
```

**Interviewer-level answer: single awk + pipeline:**
```bash
awk '{counts[$9]++} END {for (code in counts) printf "%6d %s\n", counts[code], code}' \
    "$LOG_FILE" | sort -rn
```

**Gotcha:** `$9` in combined log is the status code. But some malformed lines may have `-` (no status). Handle with:
```bash
awk '$9 ~ /^[0-9]{3}$/ {counts[$9]++} ...'
```

---

## L-002 | Extract Slow Requests

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <log_file> [threshold_ms]}"
THRESHOLD="${2:-1000}"

[[ ! -f "$LOG_FILE" ]] && { echo "ERROR: File not found: $LOG_FILE" >&2; exit 1; }

awk -v threshold="$THRESHOLD" '
$NF > threshold {
    printf "%dms\t%-6s %-25s [%s]\n", $NF, $2, $3, $1
}
' "$LOG_FILE" | sort -k1 -rn
```

**Key concepts:**
- `$NF` — last field (number of fields), portable regardless of field count
- `-v threshold="$THRESHOLD"` — pass shell variable into awk safely (no injection via quoting issues)
- `sort -k1 -rn` — sort by first field (the ms value) numerically reversed

---

## L-003 | Unique IPs with Request Count

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <access_log>}"
[[ ! -f "$LOG_FILE" ]] && { echo "ERROR: File not found: $LOG_FILE" >&2; exit 1; }

awk '{print $1}' "$LOG_FILE" | sort | uniq -c | sort -rn | tee /dev/stderr | wc -l | \
    xargs -I{} echo "Total unique IPs: {}"
```

**Cleaner version (two passes, more readable):**
```bash
awk '{print $1}' "$LOG_FILE" | sort | uniq -c | sort -rn > /tmp/ip_counts.$$
cat /tmp/ip_counts.$$
echo "---"
echo "Total unique IPs: $(wc -l < /tmp/ip_counts.$$)"
rm -f /tmp/ip_counts.$$
```

**Best version — single awk, no temp file:**
```bash
awk '
{
    ips[$1]++
}
END {
    # Collect into array for sorting (awk does not sort natively)
    n = 0
    for (ip in ips) {
        count[n] = ips[ip]
        addr[n] = ip
        n++
    }
    # Bubble sort by count desc (fine for <1000 IPs in interviews)
    for (i = 0; i < n; i++)
        for (j = i+1; j < n; j++)
            if (count[i] < count[j]) {
                tc=count[i]; count[i]=count[j]; count[j]=tc
                ta=addr[i]; addr[i]=addr[j]; addr[j]=ta
            }
    for (i = 0; i < n; i++)
        printf "%6d %s\n", count[i], addr[i]
    printf "---\nTotal unique IPs: %d\n", n
}
' "$LOG_FILE"
```

---

## L-004 | Error Rate per Minute

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <log_file> [--threshold N]}"
THRESHOLD=5

# Handle --threshold argument anywhere in args
for i in "$@"; do
    case "$i" in --threshold) THRESHOLD="${2:-5}"; break ;; esac
done

[[ ! -f "$LOG_FILE" ]] && { echo "ERROR: File not found: $LOG_FILE" >&2; exit 1; }

awk -v threshold="$THRESHOLD" '
/ERROR/ {
    # Truncate ISO timestamp to minute: 2026-04-11T10:05:32Z → 2026-04-11T10:05
    minute = substr($1, 1, 16)
    errors[minute]++
    all_minutes[minute] = 1
}
NF > 0 {
    minute = substr($1, 1, 16)
    all_minutes[minute] = 1
}
END {
    printf "%-22s %-8s %s\n", "MINUTE", "ERRORS", "STATUS"
    n = asorti(all_minutes, sorted)   # sort chronologically (gawk)
    for (i = 1; i <= n; i++) {
        m = sorted[i]
        e = errors[m] + 0
        status = (e > threshold) ? "⚠ EXCEEDED (threshold: " threshold ")" : "OK"
        printf "%-22s %-8d %s\n", m, e, status
    }
}
' "$LOG_FILE"
```

**Note on `asorti`:** This is gawk-specific. For pure POSIX awk, pipe through `sort`:
```bash
awk '/ERROR/ {minute=substr($1,1,16); errors[minute]++}
     END { for (m in errors) printf "%s %d\n", m, errors[m] }' \
    "$LOG_FILE" | sort -k1
```

---

## L-005 | Parse JSON-Lines Log

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <jsonl_log_file>}"
[[ ! -f "$LOG_FILE" ]] && { echo "ERROR: File not found: $LOG_FILE" >&2; exit 1; }

printf "%-24s %-6s %-12s %-14s %s\n" "TIMESTAMP" "LEVEL" "DURATION_MS" "PATH" "MESSAGE"
printf '%s\n' "$(printf '%.0s-' {1..80})"

# Process line by line — skip malformed lines with warning
while IFS= read -r line; do
    if ! result=$(echo "$line" | jq -r '
        select(.level == "error" and .duration_ms > 500) |
        [.timestamp, .level, (.duration_ms|tostring), .path, .msg] |
        @tsv
    ' 2>/dev/null); then
        echo "WARN: Malformed JSON line skipped" >&2
        continue
    fi
    [[ -z "$result" ]] && continue
    while IFS=$'\t' read -r ts lvl dur path msg; do
        printf "%-24s %-6s %-12s %-14s %s\n" "$ts" "$lvl" "$dur" "$path" "$msg"
    done <<< "$result"
done < "$LOG_FILE"
```

**Faster alternative — jq in streaming mode:**
```bash
jq -r '
    select(.level == "error" and .duration_ms > 500) |
    [.timestamp, .level, (.duration_ms|tostring), .path, .msg] | @tsv
' "$LOG_FILE" 2>/dev/null | \
    awk 'BEGIN { printf "%-24s %-6s %-12s %-14s %s\n", "TIMESTAMP","LEVEL","DUR_MS","PATH","MSG" }
         { printf "%-24s %-6s %-12s %-14s %s\n", $1, $2, $3, $4, $5 }'
```

**Why `select()` not `if`:** `select()` filters — if condition is false, jq produces no output for that line. `if/then/else` always produces output.

---

## L-006 | Nginx Top Endpoints by Error Rate

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <nginx_access_log> [min_requests]}"
MIN_REQUESTS="${2:-10}"

awk -v min_req="$MIN_REQUESTS" '
{
    # $7 is request path in combined log (e.g., /api/users?foo=bar)
    path = $7
    # Strip query string
    sub(/\?.*/, "", path)

    total[path]++
    if ($9 ~ /^5/) errors[path]++
}
END {
    printf "%-35s %8s %8s %10s\n", "PATH", "TOTAL", "ERRORS", "ERROR_RATE"
    printf "%-35s %8s %8s %10s\n", \
        "-----------------------------------", "--------", "--------", "----------"

    # Build array for sorting
    n = 0
    for (p in total) {
        if (total[p] >= min_req) {
            rate = (errors[p] + 0) / total[p] * 100
            rates[n] = rate
            paths[n] = p
            totals[n] = total[p]
            errs[n] = errors[p] + 0
            n++
        }
    }

    # Sort by error rate descending (bubble sort)
    for (i = 0; i < n; i++)
        for (j = i+1; j < n; j++)
            if (rates[i] < rates[j]) {
                tr=rates[i]; rates[i]=rates[j]; rates[j]=tr
                tp=paths[i]; paths[i]=paths[j]; paths[j]=tp
                tt=totals[i]; totals[i]=totals[j]; totals[j]=tt
                te=errs[i]; errs[i]=errs[j]; errs[j]=te
            }

    limit = (n < 10) ? n : 10
    for (i = 0; i < limit; i++)
        printf "%-35s %8d %8d %9.2f%%\n", paths[i], totals[i], errs[i], rates[i]
}
' "$LOG_FILE"
```

---

## L-008 | Extract Stack Traces from Multiline Log

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <log_file>}"
[[ ! -f "$LOG_FILE" ]] && { echo "ERROR: File not found: $LOG_FILE" >&2; exit 1; }

awk '
# Non-indented line
/^[^ \t]/ {
    if (in_trace) {
        # Print the buffered trace block
        printf "%s\n\n", buffer
    }
    in_trace = 0
    buffer = ""

    if (/ERROR/) {
        in_trace = 1
        buffer = $0
    }
    next
}

# Indented line (continuation of stack trace)
/^[ \t]/ {
    if (in_trace) {
        buffer = buffer "\n" $0
    }
}

END {
    # Flush last block
    if (in_trace && buffer != "") {
        printf "%s\n\n", buffer
    }
}
' "$LOG_FILE"
```

**State machine pattern** — this is the core skill: tracking state across lines in awk. The interviewer wants to see:
- A flag (`in_trace`) to track whether you're inside a multi-line block
- A buffer to accumulate lines
- Proper flush on both non-indented line AND `END`

---

## L-009 | Real-time Error Spike Detector

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    echo "Usage: $(basename "$0") --log <file> [--threshold N] [--window seconds]" >&2
    exit 1
}

LOG_FILE="" THRESHOLD=20 WINDOW=60
while [[ $# -gt 0 ]]; do
    case "$1" in
        --log)       LOG_FILE="$2"; shift 2 ;;
        --threshold) THRESHOLD="$2"; shift 2 ;;
        --window)    WINDOW="$2"; shift 2 ;;
        *) usage ;;
    esac
done
[[ -z "$LOG_FILE" ]] && usage

LAST_ALERT_TIME=0
ALERT_COOLDOWN=300  # 5 minutes between same alert

# Ring buffer: store timestamps of recent ERROR lines
declare -a RING
RING_SIZE=$WINDOW
RING_HEAD=0
RING_COUNT=0

cleanup() { echo "Detector stopped." >&2; }
trap cleanup EXIT INT TERM

send_alert() {
    local count="$1"
    local samples="$2"
    local now
    now=$(date +%s)

    if (( now - LAST_ALERT_TIME < ALERT_COOLDOWN )); then
        return 0
    fi

    echo "ALERT [$(date -Iseconds)]: ${count} errors in last ${WINDOW}s" >&2
    echo "  Sample errors: ${samples}" >&2
    LAST_ALERT_TIME=$now

    # Optional: Slack notification
    if [[ -n "${SLACK_WEBHOOK_URL:-}" ]]; then
        local payload
        payload=$(jq -n --argjson count "$count" --arg samples "$samples" --arg window "$WINDOW" \
            '{text: ":fire: Error spike: \($count) errors in \($window)s\nSamples: \($samples)"}')
        curl -sf -H "Content-Type: application/json" -d "$payload" "$SLACK_WEBHOOK_URL" 2>/dev/null || true
    fi
}

echo "Monitoring $LOG_FILE (threshold: $THRESHOLD errors/$WINDOW seconds)" >&2

# tail -F handles log rotation — reopens file if it disappears
tail -F "$LOG_FILE" 2>/dev/null | while IFS= read -r line; do
    [[ "$line" != *ERROR* ]] && continue

    NOW=$(date +%s)
    CUTOFF=$(( NOW - WINDOW ))

    # Add current timestamp to ring buffer
    RING[$RING_HEAD]=$NOW
    RING_HEAD=$(( (RING_HEAD + 1) % RING_SIZE ))
    (( RING_COUNT < RING_SIZE )) && (( RING_COUNT++ )) || true

    # Count how many entries are within the window
    COUNT=0
    for ts in "${RING[@]:-}"; do
        [[ -z "${ts:-}" ]] && continue
        (( ts >= CUTOFF )) && (( COUNT++ )) || true
    done

    if (( COUNT >= THRESHOLD )); then
        SAMPLE="${line:0:120}"
        send_alert "$COUNT" "$SAMPLE"
    fi
done
```

**Key techniques:**
- `tail -F` (capital F) — follows by filename, not file descriptor. Handles log rotation.
- Ring buffer with modulo index — O(1) insert, O(n) count for window
- Cooldown to suppress alert storms
- `|| true` on arithmetic increments to avoid `set -e` false exits

---

## L-011 | Log-Based SLI Calculator

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <access_log> [date]}"
TARGET_DATE="${2:-$(date +%Y-%m-%d)}"
AVAIL_SLO=99.9
LATENCY_SLO=95.0
LATENCY_THRESHOLD_MS=300

[[ ! -f "$LOG_FILE" ]] && { echo "ERROR: File not found: $LOG_FILE" >&2; exit 1; }

awk -v date="$TARGET_DATE" \
    -v latency_thresh="$LATENCY_THRESHOLD_MS" \
    -v avail_slo="$AVAIL_SLO" \
    -v latency_slo="$LATENCY_SLO" '
{
    # Filter by date if present in timestamp
    if (date != "" && $1 !~ date) next

    total++

    # Count non-5xx as "good" for availability
    if ($9 !~ /^5/) good_avail++

    # Last field = response time ms
    if ($NF ~ /^[0-9]+$/) {
        total_latency++
        if ($NF + 0 < latency_thresh) good_latency++
    }
}
END {
    if (total == 0) {
        print "No data for period: " date
        exit 0
    }

    avail_sli = (good_avail / total) * 100
    latency_sli = (total_latency > 0) ? (good_latency / total_latency) * 100 : 0

    # Error budget: 100% - SLO% gives the allowed error %
    # Budget consumed = (SLO - actual) / (100 - SLO) * 100
    avail_budget_remaining = 100
    if (avail_sli < avail_slo) {
        avail_budget_consumed = (avail_slo - avail_sli) / (100 - avail_slo) * 100
        avail_budget_remaining = 100 - avail_budget_consumed
    }

    printf "\n=== SLI Report: %s ===\n", date
    printf "Total requests: %d\n\n", total

    avail_status = (avail_sli >= avail_slo) ? "✓ OK" : "❌ BREACHED"
    printf "Availability SLI:    %6.2f%%  [SLO: %.2f%%] %s\n", avail_sli, avail_slo, avail_status

    latency_status = (latency_sli >= latency_slo) ? "✓ OK" : "❌ BREACHED"
    printf "Latency SLI:         %6.2f%%  [SLO: %.2f%%] %s\n", latency_sli, latency_slo, latency_status

    printf "Error Budget (avail): %.0f%% remaining", avail_budget_remaining
    if (avail_budget_remaining < 0) printf " (EXHAUSTED)"
    printf "\n"
}
' "$LOG_FILE"
```

**Why this matters in senior interviews:**
Interviewers at product companies often ask "how would you calculate SLI from raw logs?" — this is the exact implementation they want to see. Key: availability SLI = good/total, latency SLI = fast/total (separate denominators).

---

## L-012 | Kubernetes Pod Log Analyzer

```bash
#!/usr/bin/env bash
set -euo pipefail

INPUT="${1:?Usage: $0 <pod_logs_file_or_dash>}"

# Accept stdin when arg is -
if [[ "$INPUT" == "-" ]]; then
    TMPFILE=$(mktemp)
    trap "rm -f $TMPFILE" EXIT
    cat > "$TMPFILE"
    INPUT="$TMPFILE"
fi

[[ ! -f "$INPUT" ]] && { echo "ERROR: File not found: $INPUT" >&2; exit 1; }

echo "=== Pod Log Analysis ==="
echo ""

# 1. OOMKilled detection
OOM_COUNT=$(grep -cE "(OOMKilled|out of memory|Killed process|memory limit)" "$INPUT" 2>/dev/null || true)
if (( OOM_COUNT > 0 )); then
    echo "⚠ OOMKilled signal: ${OOM_COUNT} occurrence(s)"
    grep -E "(OOMKilled|out of memory|Killed process|memory limit)" "$INPUT" | head -3 | \
        sed 's/^/  /'
fi

# 2. Repeated crash patterns
echo ""
echo "Repeated error patterns (3+ occurrences):"
grep -i "error\|exception\|fatal\|panic" "$INPUT" | \
    sed 's/^[0-9T:Z\.-]* //' |  \
    sed 's/[0-9a-f]\{8\}-[0-9a-f]\{4\}-[0-9a-f]\{4\}-[0-9a-f]\{4\}-[0-9a-f]\{12\}/UUID/g' | \
    sed 's/[0-9]\+/N/g' | \
    sort | uniq -c | sort -rn | \
    awk '$1 >= 3 { printf "  %4d× %s\n", $1, substr($0, index($0,$2)) }' | \
    head -10

# 3. Startup failure detection
STARTED=$(grep -cE "(Started|Listening on|Ready to serve|server started|Application started)" \
    "$INPUT" 2>/dev/null || true)
if (( STARTED == 0 )); then
    echo ""
    echo "⚠ STARTUP FAILURE: No 'started' or 'ready' message found in logs"
fi

# 4. Root cause heuristic
echo ""
echo "=== Root Cause Summary ==="
SCORE_DB=0;  SCORE_OOM=0; SCORE_CRASH=0
(( OOM_COUNT > 0 )) && SCORE_OOM=$(( OOM_COUNT * 10 )) || true
DB_ERRORS=$(grep -ciE "(connection refused|connection reset|db error|database|timeout.*connect)" "$INPUT" 2>/dev/null || true)
(( DB_ERRORS > 0 )) && SCORE_DB=$(( DB_ERRORS * 5 )) || true
CRASH_ERRORS=$(grep -ciE "(panic|segfault|nil pointer|index out of range)" "$INPUT" 2>/dev/null || true)
(( CRASH_ERRORS > 0 )) && SCORE_CRASH=$(( CRASH_ERRORS * 8 )) || true

declare -A SCORES=([OOM_Kill]=$SCORE_OOM [DB_Connectivity]=$SCORE_DB [Crash_Panic]=$SCORE_CRASH)
# Print in descending order of score
for cause in "${!SCORES[@]}"; do
    echo "${SCORES[$cause]} $cause"
done | sort -rn | head -3 | awk '$1 > 0 { printf "  %d pts — %s\n", $1, $2 }'

echo ""
echo "Total log lines: $(wc -l < "$INPUT")"
```

---

## L-015 | Logfmt Parser

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <log_file> [filter_key=value ...]}"
shift
FILTERS=("$@")

parse_logfmt_line() {
    local line="$1"
    # Use awk to parse key=value and key="quoted value" pairs
    echo "$line" | awk '
    {
        result = ""
        i = 1
        while (i <= NF) {
            field = $i
            eq = index(field, "=")
            if (eq == 0) { i++; continue }
            key = substr(field, 1, eq-1)
            val = substr(field, eq+1)

            # Handle quoted values: key="foo bar"
            if (substr(val, 1, 1) == "\"") {
                val = substr(val, 2)
                while (substr(val, length(val)) != "\"" && i < NF) {
                    i++
                    val = val " " $i
                }
                val = substr(val, 1, length(val)-1)
            }
            printf "%s=%s\n", key, val
            i++
        }
    }'
}

while IFS= read -r line; do
    [[ -z "$line" ]] && continue

    # Check all filters — skip line if any filter does not match
    match=true
    for filter in "${FILTERS[@]}"; do
        key="${filter%%=*}"
        val="${filter#*=}"

        # Extract value for key from this line
        line_val=$(echo "$line" | grep -oP "(?<=${key}=)[^\s]+|(?<=${key}=\")[^\"]+")
        [[ -z "$line_val" ]] && { match=false; break; }

        # Support comparison operators for numeric fields (e.g., latency=>1000ms)
        if [[ "$val" =~ ^\> ]]; then
            num_val="${val#>}"
            num_val="${num_val//[^0-9]/}"
            line_num="${line_val//[^0-9]/}"
            (( line_num > num_val )) || { match=false; break; }
        else
            [[ "$line_val" == "$val" ]] || { match=false; break; }
        fi
    done

    $match && echo "$line"
done < "$LOG_FILE"
```

---

## Log Analysis Cheat Sheet — Critical Patterns

```bash
# --- FIELD EXTRACTION ---
awk '{print $9}' access.log              # HTTP status (field 9, combined format)
awk '{print $1}' access.log              # Source IP
awk '{print $7}' access.log              # Request URL
awk '{print $NF}' app.log               # Last field (response time, bytes, etc.)

# --- FREQUENCY COUNTING ---
sort | uniq -c | sort -rn               # Count + sort by frequency
sort -u                                  # Deduplicate

# --- FIELD EXTRACTION WITH STRIP ---
awk '{sub(/\?.*/, "", $7); print $7}'   # Strip query string from URL
sed 's/[0-9]\{4\}-[0-9][0-9]-[0-9][0-9}T[0-9:Z]*/TIMESTAMP/g'  # Normalize timestamps

# --- ERROR RATE BUCKETING ---
awk '/ERROR/ {minute=substr($1,1,16); c[minute]++} END {for(m in c) print m, c[m]}' | sort

# --- MULTILINE RECORDS ---
awk 'BEGIN{RS=""; FS="\n"} /CrashLoopBackOff/ {print; print "---"}'

# --- JSON LINES ---
jq -r 'select(.level=="error") | [.timestamp, .msg] | @tsv' app.jsonl

# --- BYTES TO HUMAN READABLE ---
awk '{total += $NF} END {
    if (total > 1073741824) printf "%.2f GB\n", total/1073741824
    else if (total > 1048576) printf "%.2f MB\n", total/1048576
    else printf "%.2f KB\n", total/1024
}'

# --- TAIL + FILTER (real-time) ---
tail -F /var/log/app.log | grep --line-buffered "ERROR\|FATAL"

# --- PARALLEL GREP ACROSS ARCHIVES ---
find /archive -name "*.log.gz" | xargs -P8 -I{} sh -c 'zcat {} | grep "FATAL"'
```
