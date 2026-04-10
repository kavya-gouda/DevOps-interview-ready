# Shell Scripting — Coding Exercises

> 60 exercises. Easy → Expert. Attempt each WITHOUT looking at solutions.md first.
> After writing your solution, ask: is it production-safe? Does it handle edge cases?

---

## Difficulty Guide
- 🟢 Easy — syntax and basic concepts
- 🟡 Medium — logic, text processing, error handling
- 🔴 Hard — production patterns, concurrency, senior-level
- ⭐ Expert — DevOps-specific, what interviewers actually ask

---

## Section 1: Core Bash (E-001 to E-015)

---

### E-001 🟢 | Safe Argument Validation
**Task:** Write a script that accepts 2 required arguments: `<namespace>` and `<deployment>`. If either is missing, print a usage message and exit with code 1.

**Must handle:** Missing args, extra args ignored, usage printed to stderr.

**Concepts:** `$#`, `${1:?}`, `usage()`, `>&2`

---

### E-002 🟢 | Check Required Environment Variables
**Task:** Write a function `require_env` that accepts a list of variable names and exits with a clear error if any are unset or empty.

```bash
require_env DATABASE_URL REDIS_URL JWT_SECRET
```

**Must handle:** Multiple variables, clear error saying WHICH variable is missing.

**Concepts:** `${!var}` indirect expansion, arrays, loops

---

### E-003 🟢 | Retry with Backoff
**Task:** Write a function `retry` that runs a command up to N times with exponential backoff. Print which attempt number on each retry.

```bash
retry 5 curl -sf https://api.example.com/health
```

**Must handle:** Success on first try (no unnecessary retries), failure after all retries (exit 1), backoff doubling (1s, 2s, 4s, 8s...)

**Concepts:** `"$@"`, loops, `sleep`, `(( ))`

---

### E-004 🟢 | Safe Temp File Usage
**Task:** Write a script that writes data to a temp file, processes it, and guarantees cleanup even if the script is interrupted or errors.

**Must handle:** `Ctrl+C`, script error, normal exit — all must clean up.

**Concepts:** `mktemp`, `trap`, `EXIT`

---

### E-005 🟢 | Atomic Config Write
**Task:** Write a function `write_config` that writes a new config file atomically — readers should never see a partial write.

**Must handle:** The write fails midway — old config must stay intact.

**Concepts:** `mktemp`, `mv` (atomic rename), `trap` for cleanup on failure

---

### E-006 🟡 | Parse CSV and Sum a Column
**Task:** Given a CSV file with header `name,region,cost`, print the total cost and the top 3 most expensive entries.

```
name,region,cost
frontend,eastus,1200.50
backend,westus,3400.00
database,eastus,5600.75
cache,westus,800.25
```

**Output:**
```
Total: 11001.50
Top 3:
  database: 5600.75
  backend: 3400.00
  frontend: 1200.50
```

**Concepts:** `awk`, skip header (`NR>1`), sorting, `printf`

---

### E-007 🟡 | Process Log File — Extract Error Summary
**Task:** Parse a log file where each line is: `YYYY-MM-DD HH:MM:SS LEVEL message`. Print a count of each log level and all ERROR lines with timestamps.

**Input sample:**
```
2026-04-07 10:00:01 INFO  Service started
2026-04-07 10:00:05 ERROR Database connection failed
2026-04-07 10:00:06 WARN  Retry attempt 1
2026-04-07 10:00:10 ERROR Timeout after 5s
```

**Output:**
```
Log Level Summary:
  INFO:  1
  WARN:  1
  ERROR: 2

ERROR lines:
  2026-04-07 10:00:05 — Database connection failed
  2026-04-07 10:00:10 — Timeout after 5s
```

**Concepts:** `awk`, associative arrays, `printf`

---

### E-008 🟡 | Rotate and Compress Logs
**Task:** Write a script that finds all `.log` files in a directory older than N days, compresses them with gzip, and prints a summary of how many were compressed and total space saved.

```bash
./rotate-logs.sh /var/log/myapp 7
```

**Must handle:** No files found (graceful), gzip failure (skip and warn), directory doesn't exist (error).

**Concepts:** `find`, `stat`, arithmetic, loops, error handling

---

### E-009 🟡 | Parallel Host Health Check
**Task:** Given a file of hostnames (one per line), SSH to each and run `uptime`. Run all checks in parallel. Print `OK: hostname` or `FAIL: hostname` for each. Exit 1 if any failed.

**Must handle:** Up to 50 hosts, timeout per host (5s), capture and report failures.

**Concepts:** Background jobs, `wait`, PID arrays, exit code tracking

---

### E-010 🟡 | Lock File — Prevent Concurrent Runs
**Task:** Write a script that uses a lock file to prevent two instances running simultaneously. If already running, print who holds the lock and exit. On finish, release the lock.

**Must handle:** Script crash → lock not left permanently, `Ctrl+C` → lock released, stale lock (process died) → detect and clean up.

**Concepts:** `flock`, PID in lock file, `/proc/$PID` existence check

---

### E-011 🟡 | Find Duplicate Files by Checksum
**Task:** Given a directory, find all files that have identical content (duplicates). Print groups of duplicates with their paths and sizes.

**Concepts:** `md5sum`/`sha256sum`, `sort`, `awk`, associative arrays

---

### E-012 🟡 | Monitor Disk Usage and Alert
**Task:** Check disk usage on all mounted filesystems. For any filesystem above a threshold (default 85%), print a warning. Accept threshold as an optional argument. Exit 1 if any are critical (>95%).

**Must handle:** Percentage parsing, multiple filesystems, different `df` output formats.

**Concepts:** `df`, `awk`, integer comparison, exit codes

---

### E-013 🟡 | Template File Substitution
**Task:** Write a script that reads a template file (with `{{VARIABLE}}` placeholders) and substitutes values from environment variables, writing the result to an output file.

```bash
# template.txt contains: "Host: {{DB_HOST}}, Port: {{DB_PORT}}"
./render-template.sh template.txt output.txt
```

**Must handle:** Missing variable → error (don't substitute empty string silently), output file creation.

**Concepts:** `sed`, environment variables, input validation

---

### E-014 🟡 | JSON Config Validator
**Task:** Write a script that validates a JSON config file has all required keys. Accept the config file and a comma-separated list of required keys as arguments.

```bash
./validate-config.sh config.json "database.host,database.port,app.secret"
```

**Must handle:** Nested key paths (dot-notation), missing `jq` (check and error), invalid JSON.

**Concepts:** `jq`, nested key traversal, error handling

---

### E-015 🟡 | Generate Kubernetes Namespace Report
**Task:** Write a script that for each namespace prints: namespace name, number of pods, number of running pods, total CPU requests, total memory requests.

**Output format:**
```
NAMESPACE          PODS  RUNNING  CPU_REQ   MEM_REQ
default              12       11    1500m     2048Mi
kube-system           8        8     800m     1024Mi
```

**Concepts:** `kubectl`, `jq`, `awk`, column formatting with `printf` or `column`

---

## Section 2: Text Processing Mastery (E-016 to E-025)

---

### E-016 🟡 | Nginx Access Log Analyzer
**Task:** Parse an nginx access log in combined format. Output:
1. Top 10 most requested URLs
2. Count of each HTTP status code
3. Top 5 IPs by request count
4. Total data transferred (bytes)

**Combined log format:**
```
127.0.0.1 - - [07/Apr/2026:10:00:01 +0000] "GET /api/v1/users HTTP/1.1" 200 1234 "-" "curl/7.68"
```

**Concepts:** `awk`, multiple passes or single pass with arrays, `sort | uniq -c | sort -rn`

---

### E-017 🟡 | Extract and Validate IP Addresses
**Task:** Given a file, extract all IPv4 addresses, validate they are valid (0-255 in each octet), deduplicate, and sort. Report how many invalid ones were found.

**Concepts:** `grep -oE`, regex, `awk` for validation, `sort -u`

---

### E-018 🟡 | Config File Diff Report
**Task:** Given two config files (key=value format), report: keys added, keys removed, keys with changed values, keys unchanged.

```
ADDED:   NEW_KEY=value
REMOVED: OLD_KEY
CHANGED: DB_HOST: old-host → new-host
```

**Concepts:** `sort`, `comm`, `join`, associative arrays in awk

---

### E-019 🔴 | Multi-File Log Merge and Sort
**Task:** Given a directory of log files with ISO timestamp at start of each line, merge all files and output lines sorted by timestamp. Handle files being added while processing.

**Concepts:** `sort -m` (merge sorted files), `sort -k1,1`, process substitution

---

### E-020 🔴 | Prometheus Metrics Parser
**Task:** Parse a Prometheus `/metrics` text exposition format. For a given metric name pattern, extract all labels and values. Output as a table.

```bash
./parse-metrics.sh metrics.txt "http_requests_total"
```

**Sample input:**
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200",path="/api"} 1234
http_requests_total{method="POST",status="201",path="/api"} 456
```

**Concepts:** `grep`, `sed`, `awk`, regex groups

---

### E-021 🟡 | Word Frequency Counter
**Task:** Count word frequency in a text file. Print top N words (configurable). Exclude common stop words (a, the, is, are, was, were, in, on, at).

**Concepts:** `tr`, `sort`, `uniq -c`, arrays for stop words, `grep -v`

---

### E-022 🟡 | CSV to Markdown Table Converter
**Task:** Convert a CSV file to a Markdown table. First row is the header.

**Input:**
```
Name,Role,Team
Alice,Engineer,Platform
Bob,Lead,SRE
```

**Output:**
```
| Name  | Role     | Team     |
|-------|----------|----------|
| Alice | Engineer | Platform |
| Bob   | Lead     | SRE      |
```

**Concepts:** `awk`, field widths, string padding

---

### E-023 🟡 | Multi-line Record Parser
**Task:** Parse a file where records span multiple lines separated by blank lines (like `kubectl describe` output). Extract specific fields from each record.

```
Name: pod-abc123
Namespace: production
Status: Running

Name: pod-def456
Namespace: staging
Status: CrashLoopBackOff
```

**Task:** Print only records where Status is NOT Running.

**Concepts:** `awk` multi-line records (`RS=""`), field parsing within records

---

### E-024 🔴 | Real-time Log Tailer with Filtering
**Task:** Write a script that tails a log file in real-time, filters to only show lines matching a pattern, and highlights matching text. Support multiple patterns via arguments.

```bash
./tail-filter.sh /var/log/app.log ERROR WARN "connection refused"
```

**Must handle:** File rotation (reopen if file disappears), `Ctrl+C` exits cleanly.

**Concepts:** `tail -F`, `grep --color`, process substitution, signals

---

### E-025 🔴 | Log Anomaly Detector
**Task:** Monitor a log file. Track the rate of ERROR lines per minute. If the rate exceeds a threshold (default: 10/min), print an alert. Alert clears when rate drops below threshold for 2 consecutive minutes.

**Concepts:** `tail -f`, `date +%s`, circular buffer logic in bash, rate calculation

---

## Section 3: DevOps and Kubernetes Operations (E-026 to E-045)

---

### E-026 ⭐ | Deployment Health Validator
**Task:** After a deployment, validate it is healthy:
1. Rollout completed successfully
2. All pods are Running
3. No pod has restarted more than N times in the last hour
4. Readiness probe is passing (check endpoints)

```bash
./validate-deploy.sh -n production -d frontend -r 3
```

**Exit 0** only if ALL checks pass. Exit 1 with specific failure message.

**Concepts:** `kubectl rollout status`, `kubectl get pods -o json | jq`, endpoint checking

---

### E-027 ⭐ | Kubernetes Resource Audit
**Task:** Across all namespaces, find and report:
- Pods with no resource requests set
- Pods with no resource limits set
- Pods running as root (UID 0 or no runAsNonRoot)
- Deployments with only 1 replica
- Services with no matching pods (orphaned services)

Output as a structured report grouped by finding type.

**Concepts:** `kubectl get -o json`, `jq` complex queries, report formatting

---

### E-028 ⭐ | Rolling Restart with Health Check
**Task:** Restart pods in a deployment one at a time (rolling), waiting for each to become healthy before proceeding. Abort if any pod fails to become healthy within timeout.

```bash
./rolling-restart.sh -n production -d backend -t 120
```

**Must NOT use kubectl rollout restart** — implement the pod-by-pod logic manually.

**Concepts:** `kubectl get pods`, `kubectl delete pod`, `kubectl wait`, loops

---

### E-029 ⭐ | Namespace Cleanup — Remove Stuck Terminating
**Task:** Find all namespaces stuck in Terminating state. For each, remove all finalizers and force the deletion. Print what was cleaned up.

**Must handle:** Dry-run mode (`--dry-run` flag), no stuck namespaces (print and exit 0).

**Concepts:** `kubectl get namespace -o json`, `jq` to remove finalizers, `kubectl replace --raw`

---

### E-030 ⭐ | Canary Validation Script
**Task:** After promoting a canary deployment, validate:
1. Error rate for canary is below threshold
2. P99 latency for canary is below SLO
3. Canary has been running for minimum stabilization time

Query Prometheus for both metrics. Accept thresholds as arguments.

```bash
./validate-canary.sh --error-rate 1.0 --p99-latency 500 --min-age 300
```

**Concepts:** `curl` to Prometheus API, `jq` to parse response, float comparison with `awk` or `bc`

---

### E-031 ⭐ | Certificate Expiry Monitor
**Task:** Given a list of domains (from file or args), check TLS certificate expiry for each. Print status (OK/WARNING/CRITICAL) with days remaining. Exit 1 if any are CRITICAL (< 14 days).

```bash
./check-certs.sh --warn-days 30 --critical-days 14 api.company.com app.company.com
```

**Concepts:** `openssl s_client`, `openssl x509 -enddate`, date arithmetic, parallel execution

---

### E-032 ⭐ | etcd Backup Script
**Task:** Take an etcd snapshot, verify it, upload to Azure Blob Storage, and clean up local snapshots older than 7 days. Send a Slack notification on success or failure.

**Must handle:** etcdctl not found, snapshot verification failure, upload failure — each with appropriate error and Slack alert.

**Concepts:** `etcdctl snapshot save`, `etcdctl snapshot status`, `az storage blob upload`, `curl` for Slack webhook

---

### E-033 ⭐ | Multi-Cluster Context Switcher
**Task:** Write a script that:
1. Lists available kubectl contexts with numbers
2. Prompts user to select one (or accept via argument)
3. Validates connectivity after switching
4. Shows current context and cluster info

```bash
./kctx.sh              # interactive
./kctx.sh production   # direct switch
```

**Concepts:** `kubectl config get-contexts`, `select` for menus, input validation, `kubectl cluster-info`

---

### E-034 ⭐ | Helm Release Health Checker
**Task:** Check health of all Helm releases across namespaces. Report:
- Releases not in `deployed` status
- Releases where pod count doesn't match chart's expected replica count
- Releases older than N days without update (potential drift)

**Concepts:** `helm list --output json`, `jq`, `kubectl`, date arithmetic

---

### E-035 ⭐ | Infrastructure Drift Detector
**Task:** Given a config file defining expected replica counts and image tags for deployments, compare against actual cluster state. Print drift findings. Exit 1 if drift found.

**Config format:**
```
# namespace/deployment=replicas:image-tag
production/frontend=3:v1.2.3
production/backend=5:v2.0.1
```

**Concepts:** File parsing with `while IFS= read`, string splitting, `kubectl get deployment -o json | jq`

---

### E-036 ⭐ | Kubernetes Secret Rotation
**Task:** Fetch a secret value from Azure Key Vault and update the corresponding Kubernetes secret. Trigger a rolling restart of deployments that mount the secret. Verify rollout succeeds.

**Must handle:** Key Vault not accessible, K8s secret doesn't exist (create vs update), rollout timeout.

**Concepts:** `az keyvault secret show`, `kubectl create secret --dry-run=client -o yaml | kubectl apply`, `kubectl rollout restart`

---

### E-037 ⭐ | Pipeline Failure Notifier
**Task:** Write a script to be called at the end of a CI/CD pipeline. If the pipeline failed, post a formatted message to Slack with: repo, branch, commit SHA, stage that failed, link to pipeline, who triggered it.

Accept data via environment variables (as set by GitHub Actions/GitLab CI).

**Concepts:** `curl` POST to Slack webhook, environment variables, JSON construction with `jq -n`

---

### E-038 ⭐ | Kubernetes Pod Log Aggregator
**Task:** Given a deployment name, collect logs from all its pods simultaneously (last N lines or since time). Prefix each line with the pod name. Sort output by timestamp. Write to a file and print summary.

```bash
./collect-logs.sh -n production -d backend --lines 1000 -o /tmp/logs.txt
```

**Concepts:** Parallel `kubectl logs`, temp files per pod, `sort -m` for merge, `paste` or `awk` for prefixing

---

### E-039 ⭐ | Node Drain with Safeguards
**Task:** Drain a Kubernetes node with safeguards:
1. Check if draining would violate any PodDisruptionBudgets
2. Taint the node first, wait for new pods to not schedule there
3. Then drain with configured timeout
4. Verify all pods migrated successfully

**Must NOT drain** if PDB would be violated — print which PDB blocks it.

**Concepts:** `kubectl cordon`, `kubectl drain`, `kubectl get pdb -o json | jq`, validation logic

---

### E-040 ⭐ | Cost Report Generator
**Task:** Query Azure Cost Management API, generate a report grouped by:
- Resource Group (top 10)
- Resource Type breakdown for each
- Month-over-month change percentage

Output as a formatted text table and also write a CSV file.

**Concepts:** `az costmanagement query`, `jq` aggregation, CSV writing, `printf` for table formatting

---

### E-041 🔴 | Service Dependency Graph Builder
**Task:** By querying Kubernetes NetworkPolicies and Service selectors, build a dependency graph showing which services can talk to which. Output in DOT format for GraphViz.

**Concepts:** `kubectl get networkpolicy -o json`, `jq`, graph logic in bash

---

### E-042 🔴 | Zero-Downtime Deploy Validation
**Task:** During a deployment, continuously monitor the service's error rate and latency via Prometheus. If either crosses threshold during the rollout, automatically trigger a rollback and report.

**Concepts:** Background monitoring loop, Prometheus API polling, `kubectl rollout undo`, signal handling to stop monitor after deploy completes

---

### E-043 🔴 | Automated Incident Response Script
**Task:** Given a PagerDuty alert payload (via stdin as JSON), parse the alert type and run the appropriate automated response:
- `HighErrorRate` → restart affected deployment
- `PodOOMKilled` → increase memory limits by 25% and restart
- `DiskPressure` → find and compress old logs

Report actions taken and whether they resolved the alert.

**Concepts:** `jq` for parsing, case/switch pattern, modular function design, Kubernetes API patches

---

### E-044 🔴 | Blue-Green Traffic Switcher
**Task:** Given `blue` and `green` deployments behind a Service, switch 100% traffic from blue to green by updating the Service selector. Validate the switch worked (health check), with rollback option.

```bash
./bg-switch.sh -n production -s frontend-svc --to green --validate --rollback-on-failure
```

**Concepts:** `kubectl patch service`, selector manipulation, health check loop, rollback logic

---

### E-045 🔴 | Chaos Injection Script
**Task:** For chaos testing, write a script that can:
1. Kill a random pod in a namespace (with label filter)
2. Inject network latency on a node using tc (traffic control)
3. Fill disk on a pod by writing a temp file

Each action must:
- Have a clearly defined blast radius (namespace/label scoped)
- Auto-revert after N seconds
- Log all actions to an audit file

**Concepts:** `kubectl delete pod --field-selector`, `ssh` + `tc qdisc`, `kubectl exec`, traps for cleanup

---

## Section 4: Expert Level (E-046 to E-060)

---

### E-046 ⭐ | Script Self-Update Mechanism
**Task:** Write a script that checks GitHub for a newer version of itself (comparing semantic versions), downloads it, validates the checksum, and replaces itself atomically.

```bash
./my-tool.sh --self-update
```

**Concepts:** Semver comparison in bash, `curl`, `sha256sum` verification, atomic replace

---

### E-047 ⭐ | Multi-Environment Config Generator
**Task:** Given a base Helm values file and environment-specific override files, deep-merge them and output the final merged config. Support `dev`, `staging`, `prod` environments.

```bash
./gen-config.sh --base values.yaml --env prod --output values-final.yaml
```

**Concepts:** `yq` for YAML merging, or implement deep merge in `jq`, validation of required fields

---

### E-048 ⭐ | Prometheus Alert Rule Validator
**Task:** Given a directory of Prometheus alert rule YAML files, validate:
1. All rules have `summary` and `description` annotations
2. Counter metrics use `rate()` not `increase()` for alerting
3. No alert has a duration (`for:`) of less than 1 minute
4. Labels include required fields (`severity`, `team`)

Report rule name + file + violation for each finding.

**Concepts:** `yq`/`jq`, regex in bash, YAML traversal, report generation

---

### E-049 ⭐ | GitOps Drift Monitor
**Task:** For each ArgoCD application, compare the Git-defined replica count vs actual running replica count. Report apps out of sync. Can be run as a cron job.

```bash
./gitops-drift.sh --argocd-server argocd.company.com --token-file /run/secrets/argocd-token
```

**Concepts:** ArgoCD API (`/api/v1/applications`), `curl` with auth, `jq` comparison, exit codes

---

### E-050 ⭐ | Dynamic Inventory Generator for Ansible
**Task:** Query Azure VMSS and AKS node pools to generate an Ansible dynamic inventory JSON. Group hosts by: environment tag, region, role tag.

**Output format:** Standard Ansible dynamic inventory JSON with `_meta.hostvars`.

**Concepts:** `az vmss list`, `az aks nodepool list`, `jq` to build nested JSON, Ansible inventory format

---

### E-051 🔴 | Bash Unit Test Framework
**Task:** Implement a minimal bash testing framework with:
- `assert_equal`, `assert_not_equal`, `assert_exit_code` functions
- Test discovery (functions starting with `test_`)
- Colored pass/fail output
- Exit code = number of failed tests

Then write tests for one of the earlier exercises.

**Concepts:** Function pointers in bash, color codes, `declare -F`, test isolation via subshells

---

### E-052 🔴 | Rate-Limited API Poller
**Task:** Poll an API endpoint at a controlled rate (max N requests/second). If rate limit response received (429), back off exponentially. Collect all paginated results. Write output to file atomically.

**Concepts:** Rate limiting with `sleep`, 429 detection, pagination via `Link` header or `nextPage` field in JSON, atomic write

---

### E-053 🔴 | Kubernetes Admission Webhook Tester
**Task:** Write a script that sends test admission webhook payloads to a webhook server and validates the responses. Test both allowed and denied scenarios.

```bash
./test-webhook.sh --url https://webhook.company.com/validate --test-dir ./test-cases/
```

**Concepts:** `curl` with TLS, JSON payload construction with `jq`, response validation, test case directory structure

---

### E-054 🔴 | Distributed Lock Using etcd
**Task:** Implement a distributed lock in bash using etcd's lease mechanism. Multiple instances of a script should only have one active at a time.

```bash
./with-lock.sh --key "backup-job" --ttl 300 -- ./run-backup.sh
```

**Concepts:** `etcdctl lease grant`, `etcdctl put --lease`, `etcdctl lease keep-alive` in background, cleanup on exit

---

### E-055 🔴 | Terraform Plan Analyzer
**Task:** Parse a `terraform plan -json` output. Report:
- Resources being destroyed (EXIT 1 if any in protected list)
- Resources being replaced (recreated)
- Net change: adds / changes / removes
- Estimate blast radius (affected downstream resources)

```bash
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan | ./analyze-plan.sh --protect "azurerm_kubernetes_cluster,azurerm_postgresql_server"
```

**Concepts:** `jq` on Terraform plan JSON schema, `resource_changes` array, `actions` field

---

### E-056 🔴 | Event-Driven Script Orchestrator
**Task:** Write a script that watches a directory for new JSON files (events). For each file, parse the event type and dispatch to a handler function. Process events in order (FIFO). Handle concurrent file creation.

**Concepts:** `inotifywait` (Linux) or polling, FIFO queue, dispatch pattern, lock for ordering

---

### E-057 🔴 | Multi-Region Health Dashboard
**Task:** Check health of services across multiple Azure regions simultaneously. For each region and service combination, check: HTTP health endpoint, DNS resolution time, TLS cert validity. Output a live-updating ASCII dashboard.

**Concepts:** Parallel execution, `printf` with ANSI cursor control (`\033[H\033[2J`), result aggregation

---

### E-058 🔴 | Kubernetes RBAC Auditor
**Task:** Find all ClusterRoleBindings and RoleBindings that grant dangerous permissions:
- Ability to create/update Secrets cluster-wide
- Ability to exec into pods (`pods/exec`)
- Any binding using `cluster-admin` role

For each finding, output: subject (user/group/SA), namespace, role, dangerous permission.

**Concepts:** `kubectl get clusterrolebinding,rolebinding -o json`, `jq` recursive search, permission mapping

---

### E-059 🔴 | Automated SLO Report Generator
**Task:** Query Prometheus for the last 30-day window. For each service in a config file, calculate:
- Actual availability (1 - error rate)
- P50/P95/P99 latency
- Error budget remaining
- Error budget burn rate (last 1h vs 6h vs 24h)

Generate a formatted report as both text and markdown.

**Concepts:** Prometheus range queries via API, `jq` math, date range calculation, multi-window burn rate formula

---

### E-060 🔴 | Pipeline as Code Runner
**Task:** Write a mini pipeline runner that reads a YAML pipeline definition and executes stages and steps. Support:
- Sequential stages
- Parallel steps within a stage
- Step timeouts
- Retry on failure
- Environment variable passing between steps via a shared context file

```yaml
stages:
  - name: build
    steps:
      - name: compile
        run: make build
        timeout: 120
      - name: test
        run: make test
        retry: 2
  - name: deploy
    steps:
      - name: push
        run: docker push image:tag
```

**Concepts:** `yq` to parse YAML, parallel execution with timeout (`timeout` command), step output capture, retry logic

---

## Section 5: Log File Analysis (L-001 to L-020)

> Real log parsing is a core DevOps interview skill. These exercises cover Apache/nginx, app logs, systemd journal, Kubernetes pod logs, and structured JSON logs.
> Formats covered: combined log, JSON-lines, key=value, syslog, Prometheus exposition.

---

### L-001 🟢 | Count HTTP Status Codes from Apache/Nginx Log

**Task:** Given an access log in combined format, count each HTTP status code and print sorted by frequency descending.

**Input:**
```
192.168.1.1 - - [11/Apr/2026:10:00:01 +0000] "GET /api/users HTTP/1.1" 200 512 "-" "curl"
192.168.1.2 - - [11/Apr/2026:10:00:02 +0000] "POST /login HTTP/1.1" 401 89 "-" "Mozilla"
192.168.1.1 - - [11/Apr/2026:10:00:03 +0000] "GET /api/orders HTTP/1.1" 500 0 "-" "curl"
192.168.1.3 - - [11/Apr/2026:10:00:04 +0000] "GET /health HTTP/1.1" 200 2 "-" "probe"
```

**Expected output:**
```
    2 200
    1 401
    1 500
```

**Constraints:** Single awk command preferred. No temp files.

**Concepts:** `awk '{print $9}'`, `sort | uniq -c | sort -rn`

---

### L-002 🟢 | Extract Slow Requests from Access Log

**Task:** Parse a log where the last field is response time in milliseconds. Print all requests slower than a threshold (default: 1000ms), sorted by response time descending.

**Log format:**
```
2026-04-11T10:00:01Z GET /api/users 200 1523
2026-04-11T10:00:02Z POST /login 200 342
2026-04-11T10:00:03Z GET /api/orders 200 4201
2026-04-11T10:00:04Z GET /health 200 12
```

**Output:**
```
4201ms  GET /api/orders   [2026-04-11T10:00:03Z]
1523ms  GET /api/users    [2026-04-11T10:00:01Z]
```

**Constraints:** Accept threshold as argument, default 1000. Output sorted by time desc.

**Concepts:** `awk '$NF > threshold'`, `sort -k1 -rn`, `printf` alignment

---

### L-003 🟢 | Extract Unique IPs from Access Log

**Task:** Given a log file (combined or custom format), extract all unique source IPs and print with request count, sorted by count descending. Print the total unique IP count at the end.

**Expected output:**
```
    45 10.0.1.25
    32 10.0.1.10
     8 192.168.0.5
---
Total unique IPs: 3
```

**Constraints:** Handle IPv6 addresses too. Must work for combined log format (IP in field 1).

**Concepts:** `awk`, `sort | uniq -c | sort -rn`, `wc -l`

---

### L-004 🟡 | Parse Application Logs — Error Rate per Minute

**Task:** Given a log file with lines like `2026-04-11T10:05:32Z ERROR connection pool exhausted`, calculate the error rate (errors per minute) for each minute window. Print minutes where error rate exceeds a threshold.

**Expected output:**
```
MINUTE              ERRORS  THRESHOLD
2026-04-11T10:05    8       ⚠ EXCEEDED (threshold: 5)
2026-04-11T10:06    2       OK
2026-04-11T10:07    12      ⚠ EXCEEDED (threshold: 5)
```

**Constraints:** Accept `--threshold N` arg, default 5. Handle log files spanning multiple hours.

**Concepts:** `awk` with timestamp truncation to minute, associative arrays for bucketing, `printf`

---

### L-005 🟡 | Parse JSON-Lines Log File

**Task:** Given a log file where each line is a JSON object (JSON-lines format), extract all events where `level == "error"` and `duration_ms > 500`. Print as a formatted table.

**Input (one JSON per line):**
```json
{"timestamp":"2026-04-11T10:00:01Z","level":"info","msg":"Request OK","duration_ms":120,"path":"/api/users"}
{"timestamp":"2026-04-11T10:00:02Z","level":"error","msg":"DB timeout","duration_ms":5200,"path":"/api/orders"}
{"timestamp":"2026-04-11T10:00:03Z","level":"error","msg":"Auth fail","duration_ms":45,"path":"/login"}
{"timestamp":"2026-04-11T10:00:04Z","level":"error","msg":"Slow query","duration_ms":1800,"path":"/api/search"}
```

**Expected output:**
```
TIMESTAMP              LEVEL  DURATION_MS  PATH         MESSAGE
2026-04-11T10:00:02Z   error  5200         /api/orders  DB timeout
2026-04-11T10:00:04Z   error  1800         /api/search  Slow query
```

**Constraints:** Use `jq` with streaming. Handle malformed lines (skip with warning to stderr).

**Concepts:** `jq -c 'select(.level == "error" and .duration_ms > 500)'`, `printf` table formatting, `jq` error handling

---

### L-006 🟡 | Nginx Log — Top Endpoints by Error Rate

**Task:** Parse an nginx access log. For each unique URL path (ignore query strings), calculate: total requests, error count (5xx), error rate %. Print top 10 by error rate, minimum 10 total requests.

**Output:**
```
PATH                    TOTAL   ERRORS  ERROR_RATE
/api/checkout           523     48      9.18%
/api/payment/confirm    210     19      9.05%
/api/search             1204    22      1.83%
```

**Constraints:** Strip query strings from URLs (`/api/users?id=123` → `/api/users`). Filter out paths with < 10 requests.

**Concepts:** `awk`, URL stripping with `sub(/\?.*/, "")`, sorting by computed field

---

### L-007 🟡 | Log File Diff — What Changed Between Deployments

**Task:** Given two log files from before and after a deployment (same service), compare error patterns. Find error messages that:
1. Are NEW (appear in new log, not in old)
2. DISAPPEARED (were in old log, not in new)
3. INCREASED (error count up by > 50%)
4. DECREASED

**Constraints:** Normalize log lines before comparing (strip timestamps and IDs). Accept two file arguments.

**Concepts:** `sed` for normalization (timestamp stripping), `sort | uniq -c`, `comm`, awk for percentage comparison

---

### L-008 🟡 | Extract Stack Traces from Multiline Log

**Task:** Log file has mixed single-line entries and Java/Python stack traces (indented lines following an ERROR line). Extract all complete stack traces (ERROR header + all indented continuation lines).

**Input:**
```
2026-04-11T10:00:01Z INFO Request processed
2026-04-11T10:00:02Z ERROR NullPointerException
    at com.app.Service.process(Service.java:42)
    at com.app.Controller.handle(Controller.java:18)
2026-04-11T10:00:03Z INFO Next request
2026-04-11T10:00:04Z ERROR ConnectionRefused
    at com.app.DB.connect(DB.java:55)
```

**Expected output:** All complete stack trace blocks (one per block, separated by blank line).

**Constraints:** A stack trace ends when the next non-indented line appears. Must handle files with no stack traces gracefully.

**Concepts:** `awk` with record state machine (`in_trace` flag), `RS` and `ORS` handling

---

### L-009 🔴 | Real-time Error Spike Detector

**Task:** Monitor a log file live (like `tail -f`). Track error count in a sliding 60-second window. Alert (print to stderr + optionally POST to Slack) when errors exceed threshold. Alert should include: spike count, representative error messages (last 3 unique), time of spike.

```bash
./spike-detector.sh --log /var/log/app.log --threshold 20 --window 60
```

**Must handle:** Log file rotation (reopen on disappear), duplicate alerts suppressed for 5 minutes after first, `Ctrl+C` exits cleanly.

**Concepts:** `tail -F`, `date +%s`, circular buffer (array with modulo index), Slack webhook via `curl`, process cleanup with `trap`

---

### L-010 🔴 | Multi-Service Log Correlator

**Task:** Given log files from 3 services (frontend, backend, database), correlate entries by a `request_id` field. For each request_id visible in all 3 logs, print the complete trace across all services in chronological order.

```bash
./correlate-logs.sh --frontend frontend.log --backend backend.log --db db.log --request-id abc123
# OR: correlate all request IDs
./correlate-logs.sh --frontend frontend.log --backend backend.log --db db.log --all
```

**Output:**
```
REQUEST: abc123
  [frontend]  2026-04-11T10:00:01.001Z  GET /api/orders - forwarding to backend
  [backend]   2026-04-11T10:00:01.050Z  Received GET /api/orders - querying DB
  [database]  2026-04-11T10:00:01.120Z  SELECT orders WHERE user_id=42 (15ms)
  [backend]   2026-04-11T10:00:01.140Z  Response 200 OK
  [frontend]  2026-04-11T10:00:01.180Z  Returning 200 to client
```

**Concepts:** `grep` for request_id extraction, prefix with service name, `sort` by timestamp, handling different timestamp formats

---

### L-011 🔴 | Log-Based SLI Calculator

**Task:** Given an nginx access log for a 24-hour period, calculate:
- **Availability SLI:** (non-5xx requests / total requests) × 100
- **Latency SLI:** % of requests completing under 300ms (assume last field is response time ms)
- **Error budget consumed:** Based on SLO targets (availability: 99.9%, latency: 95%)

**Output:**
```
=== SLI Report: 2026-04-11 ===
Availability SLI:    99.73%  [SLO: 99.90%] ❌ BREACHED
Latency SLI:         96.21%  [SLO: 95.00%] ✓ OK
Error Budget (avail): 27.00% remaining (consumed 73%)
```

**Constraints:** Accept date as argument to filter log. Handle empty log (no data for period).

**Concepts:** `awk` multi-metric accumulation, float math with `printf "%.2f"`, SLO/error budget arithmetic

---

### L-012 🔴 | Kubernetes Pod Log Analyzer

**Task:** Given output of `kubectl logs --previous <pod>` (or a saved file), detect:
1. OOMKilled signals (memory limit hit)
2. Repeated crash patterns (same error 3+ times)
3. Startup failures (pod never emitted a "ready" or "started" message)
4. Provide a root cause summary with top 3 most likely causes

```bash
./analyze-pod-logs.sh pod-logs.txt
# Or live:
kubectl logs --previous my-pod | ./analyze-pod-logs.sh -
```

**Must handle:** stdin as input (when file is `-`), empty log, logs without clear patterns.

**Concepts:** Reading from stdin vs file, `grep -c`, pattern counting, associative arrays for dedup, heuristic scoring

---

### L-013 🟡 | Parse systemd Journal Export

**Task:** Parse `journalctl -o json` output (one JSON per line). For a given service unit, extract:
- All CRITICAL/ERR priority messages
- Boot session boundaries (`MESSAGE_ID=fc2e22bc...`)
- Total log volume per priority level

```bash
journalctl -u nginx -o json | ./parse-journal.sh --unit nginx --since "1 hour ago"
```

**Concepts:** `jq` with `PRIORITY` field (0-7 numeric), `SYSLOG_IDENTIFIER`, `__REALTIME_TIMESTAMP` (microseconds), date formatting

---

### L-014 🟡 | Generate Log Summary Report

**Task:** Write a script that, given any log directory, produces a single-page HTML summary report with:
- Total log files, total lines, total size
- Top 10 most common error messages (deduped, normalized)
- Timeline: errors-per-hour bar chart (ASCII)
- Files with highest error density

```bash
./log-report.sh /var/log/myapp/ > report.html
```

**Constraints:** Handle gzip-compressed `.log.gz` files too (read with `zcat`). Output valid HTML.

**Concepts:** `find`, `zcat`, `awk` for HTML generation, ASCII bar chart with `printf`, error normalization (strip timestamps/IDs with `sed`)

---

### L-015 🟡 | Key=Value Log Parser (Logfmt)

**Task:** Parse logfmt-style logs (used by many Go services). Each line: `timestamp=... level=... msg="..." latency=42ms service=api`.
Extract all unique keys, let user query by key=value filter, output matching lines as a formatted table.

**Input:**
```
timestamp=2026-04-11T10:00:01Z level=info msg="Request complete" latency=120ms service=api path=/users
timestamp=2026-04-11T10:00:02Z level=error msg="DB connection failed" latency=5002ms service=api path=/orders
timestamp=2026-04-11T10:00:03Z level=info msg="Cache hit" latency=2ms service=cache path=/orders
```

**Usage:**
```bash
./parse-logfmt.sh app.log level=error
./parse-logfmt.sh app.log service=api latency=">1000ms"
```

**Concepts:** `awk` for key=value splitting, associative arrays, comparison with numeric extraction from `latency` field

---

### L-016 🟡 | Access Log Bandwidth Report

**Task:** From an nginx/apache access log, calculate bandwidth:
- Total bytes transferred for the period
- Top 10 endpoints by bytes transferred
- Top 10 IPs by bytes transferred
- Hourly bandwidth breakdown (bytes per hour)

**Constraints:** Bytes field (`$10` in combined log) can be `-` for 0 bytes — handle that. Format large numbers as KB/MB/GB.

**Concepts:** `awk` with bytes accumulation, conditional field (`$10 == "-" ? 0 : $10`), `printf` with magnitude formatting

---

### L-017 🔴 | Log Pattern Alerting Engine

**Task:** Write a configurable alerting engine driven by a rules file. Each rule: pattern to match (regex), threshold count, time window (seconds), alert message template.

**Rules file format:**
```
# pattern | threshold | window_seconds | alert_message
ERROR.*database | 5 | 60 | Database errors spiking: {count} in last {window}s
FATAL | 1 | 300 | Fatal error detected: {first_match}
timeout.*exceeded | 10 | 120 | Timeouts elevated: {count} occurrences
```

```bash
./alert-engine.sh --log /var/log/app.log --rules /etc/alert-rules.conf
```

**Must handle:** Multiple rules firing simultaneously, deduplication (same rule can't fire more than once per window), graceful exit on SIGTERM.

**Concepts:** `tail -f`, regex matching in bash, per-rule circular buffers, template variable substitution, `trap SIGTERM`

---

### L-018 🔴 | Log Archive Search Tool

**Task:** Given a log archive directory containing `.log`, `.log.gz`, and `.log.tar.gz` files named with dates (e.g., `app-2026-04-10.log.gz`), search across all files efficiently. Support:
- Date range filtering (`--from`, `--to`)
- Regex pattern search
- Output file:line references
- Count-only mode

```bash
./log-search.sh --dir /archive/logs --from 2026-04-01 --to 2026-04-11 --pattern "FATAL|CRITICAL"
```

**Must handle:** Parallel decompression for speed, `.tar.gz` files, binary file detection (skip), progress indicator.

**Concepts:** `find` with date filtering, `zcat`/`tar -xzO`, parallel jobs, `grep -n` for line numbers, `file` command for binary detection

---

### L-019 🔴 | Distributed Trace Log Reconstructor

**Task:** Microservices emit logs with `trace_id`, `span_id`, `parent_span_id`. Reconstruct the full trace tree from a mixed log file.

**Input:**
```
{"trace_id":"abc","span_id":"001","parent_span_id":null,"service":"gateway","msg":"Request received","duration_ms":350}
{"trace_id":"abc","span_id":"002","parent_span_id":"001","service":"auth","msg":"Token validated","duration_ms":25}
{"trace_id":"abc","span_id":"003","parent_span_id":"001","service":"backend","msg":"Order fetched","duration_ms":180}
{"trace_id":"abc","span_id":"004","parent_span_id":"003","service":"db","msg":"Query exec","duration_ms":45}
```

**Expected output:**
```
TRACE: abc  (total: 350ms)
└── [001] gateway      350ms  Request received
    ├── [002] auth      25ms  Token validated
    └── [003] backend  180ms  Order fetched
        └── [004] db    45ms  Query exec
```

**Constraints:** Handle missing spans gracefully. Sort siblings by start time if available.

**Concepts:** `jq` for tree building, recursive logic in bash (or iterative with arrays), tree drawing with ASCII characters

---

### L-020 ⭐ | Production Log Pipeline

**Task:** Build a complete log processing pipeline as a single script:

1. **Ingest:** Watch a directory for new log files (any format: JSON-lines, logfmt, or combined)
2. **Parse:** Auto-detect format, normalize to a common structure: `timestamp | level | service | message`
3. **Enrich:** Lookup IP addresses against an IP-to-service map file (key=value)
4. **Filter:** Drop DEBUG-level lines
5. **Aggregate:** Count errors per service per minute
6. **Output:** Write normalized lines to a rotated output file; write error counts to a metrics file in Prometheus exposition format

```bash
./log-pipeline.sh --watch-dir /var/log/services/ --output-dir /processed/ --ip-map /etc/ip-service.map
```

**This is a system design exercise in bash.** Interviewers want to see: modular functions, separation of concerns, graceful degradation when one format parser fails, cleanup logic.

**Concepts:** `inotifywait` or polling, format detection heuristics, pipeline with functions as filters, Prometheus text format output, `flock` for output file rotation safety


