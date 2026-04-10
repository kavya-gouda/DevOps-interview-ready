# CI/CD & GitOps — 90 Scenarios

> **ID prefix:** C- | Types: Design, Implement, Troubleshoot, Explain | Difficulty: Medium → Expert

---

## Section 1: GitHub Actions (C-001 to C-025)

---

### C-001 | Reusable Workflows Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a reusable GitHub Actions workflow for building and pushing Docker images that can be called from 50 different service repositories.

```yaml
# .github/workflows/build-push.yml (in shared repo or .github repo)
on:
  workflow_call:
    inputs:
      image_name:
        required: true
        type: string
      dockerfile_path:
        default: "Dockerfile"
        type: string
      push:
        default: true
        type: boolean
    outputs:
      image_digest:
        description: "Pushed image SHA digest"
        value: ${{ jobs.build.outputs.digest }}
    secrets:
      ACR_LOGIN_SERVER:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
    - uses: actions/checkout@v4
    - name: Log in to ACR
      uses: azure/docker-login@v1
      with:
        login-server: ${{ secrets.ACR_LOGIN_SERVER }}
    - name: Build and push
      id: push
      uses: docker/build-push-action@v5
      with:
        push: ${{ inputs.push }}
        tags: ${{ secrets.ACR_LOGIN_SERVER }}/${{ inputs.image_name }}:${{ github.sha }}
```
- Caller: `uses: .github/workflows/build-push.yml@main` from any repo
- Digest output: use immutable digest in deployment manifests, not mutable tag
- Secret inheritance: caller passes secrets to reusable workflow via `secrets:` block

---

### C-002 | OIDC Authentication to Azure
**Type:** Implement | **Difficulty:** Hard | ⬜

> Replace a stored Azure service principal secret in GitHub Actions with OIDC federated identity. Walk through the full setup.

```yaml
# Workflow permissions
permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    steps:
    - name: Azure Login via OIDC
      uses: azure/login@v2
      with:
        client-id: ${{ vars.AZURE_CLIENT_ID }}
        tenant-id: ${{ vars.AZURE_TENANT_ID }}
        subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
        # No secret! OIDC token from GitHub exchanged for Azure access token
```
```bash
# Azure setup: create federated credential on app registration
az ad app federated-credential create \
  --id <app-object-id> \
  --parameters '{
    "name": "github-actions-prod",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:org/repo:environment:production",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```
- Subject filters: `repo:org/repo:ref:refs/heads/main`, `repo:org/repo:environment:production`, `repo:org/repo:pull_request`
- Security: no long-lived secret stored in GitHub, credential scoped to specific repo/branch/env

---

### C-003 | Matrix Strategy — Fan-Out Testing
**Type:** Implement | **Difficulty:** Medium | ⬜

> Use GitHub Actions matrix strategy to run integration tests across 3 Kubernetes versions and 2 cloud regions simultaneously.

```yaml
jobs:
  integration-test:
    strategy:
      fail-fast: false   # Don't cancel other matrix jobs on one failure
      matrix:
        k8s_version: ["1.28", "1.29", "1.30"]
        region: ["eastus", "westeurope"]
        exclude:
          - k8s_version: "1.28"
            region: "westeurope"  # Skip this combination
    
    steps:
    - name: Run tests
      env:
        K8S_VERSION: ${{ matrix.k8s_version }}
        AZURE_REGION: ${{ matrix.region }}
      run: make test-integration
```
- Matrix produces 6 jobs (3 × 2) minus exclusions = 5 jobs, run in parallel
- `fail-fast: false`: see all failures, not just first
- Max parallelism: controlled by runner capacity

---

### C-004 | Actions Runner Controller (ARC) on Kubernetes
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a self-hosted GitHub Actions runner infrastructure using ARC on AKS. How do you handle autoscaling, isolation, and secret management?

**What you must cover:**
- ARC deployment: `actions-runner-system` namespace, `AutoscalingRunnerSet` CR
- Runner pods: ephemeral (Kubernetes mode), one runner per pod, destroyed after job
- Autoscaling: ARC scales runner pods based on queued workflow jobs (via GitHub API webhook or polling)
- Isolation: separate runner sets per team namespace + RBAC (each team's runs can't access other namespaces)
- Secret access: workload identity on runner SA → access Azure Key Vault, ACR, storage
- Node pool: dedicated node pool for runners (taint + toleration) — prevent runner workloads from consuming app pool
- Spot nodes: runners are stateless and ephemeral — perfect for spot/preemptible nodes (cost optimization)

---

### C-005 | Composite Actions for DRY Pipelines
**Type:** Implement | **Difficulty:** Medium | ⬜

> Create a composite action that handles: checkout, cache dependencies, and run tests — reusable across 20 repositories.

```yaml
# .github/actions/setup-and-test/action.yml
name: 'Setup and Test'
description: 'Checkout, setup Python, cache deps, run tests'

inputs:
  python_version:
    required: false
    default: '3.12'
  test_command:
    required: false
    default: 'pytest tests/'

runs:
  using: composite
  steps:
  - uses: actions/checkout@v4
  
  - uses: actions/setup-python@v5
    with:
      python-version: ${{ inputs.python_version }}
  
  - uses: actions/cache@v4
    with:
      path: ~/.cache/pip
      key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
  
  - name: Install dependencies
    run: pip install -r requirements.txt
    shell: bash
  
  - name: Run tests
    run: ${{ inputs.test_command }}
    shell: bash
```
- Caller: `uses: ./.github/actions/setup-and-test` (local) or `uses: org/.github/.github/actions/setup-and-test@v1`
- Composite vs reusable workflow: composite = steps in a single job. Reusable = full job(s).

---

### C-006 | Concurrency Groups and Deployment Gates
**Type:** Implement | **Difficulty:** Hard | ⬜

> Prevent multiple simultaneous deployments to the same environment using concurrency groups.

```yaml
jobs:
  deploy-prod:
    concurrency:
      group: deploy-prod-${{ github.ref }}
      cancel-in-progress: false  # Queue, don't cancel (use true for PR previews)
    
    environment:
      name: production
      url: https://app.company.com
    
    steps:
    - name: Deploy
      run: |
        argocd app sync myapp --revision ${{ github.sha }}
        argocd app wait myapp --health --timeout 300
```
- `cancel-in-progress: true`: new push to PR cancels previous in-progress run (good for PRs)
- `cancel-in-progress: false`: queue subsequent runs (good for prod deployments — don't skip)
- Environment protection rules: required reviewers for production deployments

---

### C-007 | GitHub Actions Security Hardening
**Type:** Design | **Difficulty:** Hard | ⬜

> List 10 GitHub Actions security best practices and implement the top 3.

**What you must cover:**
1. Pin third-party actions to commit SHA: `uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`
2. Minimal permissions: `permissions: contents: read` — not default write
3. OIDC instead of secrets for cloud auth
4. Limit `pull_request_target` scope — dangerous action trigger
5. No secrets in environment variables if they can be avoided
6. Secret scanning enabled on repo
7. Code scanning (CodeQL) required status check
8. Required reviewers for environment deployments
9. `CODEOWNERS` for workflow file changes
10. Audit GitHub Actions usage regularly (`gh api /orgs/org/actions/permissions`)

---

### C-008 | Dependency Caching Strategy
**Type:** Implement | **Difficulty:** Medium | ⬜

> Implement an optimal caching strategy for a Node.js + Docker build pipeline that reduces CI time from 8 minutes to under 3.

```yaml
steps:
# Node modules cache
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# Docker layer cache via registry
- uses: docker/build-push-action@v5
  with:
    cache-from: type=registry,ref=myacr.azurecr.io/myapp:buildcache
    cache-to: type=registry,ref=myacr.azurecr.io/myapp:buildcache,mode=max
    push: true
    tags: myacr.azurecr.io/myapp:${{ github.sha }}
```
- `hashFiles()`: cache key changes when lockfile changes — stale cache invalidated automatically
- Restore keys: fallback to partial cache match when exact key not found
- BuildKit `mode=max`: caches all intermediate layers, not just final

---

### C-009 | Scheduled Workflow for Security Scanning
**Type:** Implement | **Difficulty:** Medium | ⬜

> Create a scheduled GitHub Actions workflow that runs nightly dependency and container vulnerability scans and creates GitHub issues for findings.

```yaml
on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM UTC daily

jobs:
  security-scan:
    steps:
    - uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myacr.azurecr.io/myapp:latest'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'
    
    - name: Create issue on critical findings
      if: failure()
      uses: actions/github-script@v7
      with:
        script: |
          await github.rest.issues.create({
            owner: context.repo.owner,
            repo: context.repo.repo,
            title: '🚨 Critical vulnerability detected',
            body: 'Trivy scan found CRITICAL vulnerabilities. Check CI run: ' + context.serverUrl + '/' + context.repo.owner + '/' + context.repo.repo + '/actions/runs/' + context.runId,
            labels: ['security', 'critical']
          })
```

---

### C-010 | GitHub Actions for Multi-Service Monorepo
**Type:** Design | **Difficulty:** Expert | ⬜

> Design GitHub Actions workflows for a monorepo with 50 services. Only trigger builds for changed services.

```yaml
jobs:
  detect-changes:
    outputs:
      services: ${{ steps.filter.outputs.changes }}
    steps:
    - uses: dorny/paths-filter@v3
      id: filter
      with:
        filters: |
          service-a: services/service-a/**
          service-b: services/service-b/**
          shared-lib: shared/**   # All services rebuild if shared lib changes

  build-services:
    needs: detect-changes
    if: needs.detect-changes.outputs.services != '[]'
    strategy:
      matrix:
        service: ${{ fromJSON(needs.detect-changes.outputs.services) }}
```
- Affected services: rebuild any service that depends on a changed shared library (dependency graph)
- Nx/Turborepo: native affected detection — `nx affected:build --base=origin/main`
- Build caching: Bazel remote cache → only changed targets rebuilt

---

### C-011 to C-025 | GitHub Actions Rapid-Fire

---

### C-011 | A GitHub Actions workflow is consuming too many runner minutes per month. How do you optimize? | Design | Hard | ⬜
**Hint:** Cancel stale runs on push. Skip unchanged services. Cache aggressively. Move slow E2E tests to nightly. Self-hosted runners (cheaper). `workflow_run` to build on merge only, not every push.

### C-012 | A workflow needs to pass a secret from one job to another. How do you do this securely? | Implement | Hard | ⬜
**Hint:** Secrets can't pass as job outputs (logged). Use OIDC to re-fetch the secret in each job. Or write to Azure Key Vault, read in next job. Never use environment variables for sensitive content across jobs.

### C-013 | Implement a workflow that fails fast if any changed file has a TODO comment containing a password pattern. | Implement | Hard | ⬜
**Hint:** `gitleaks` or custom `grep` step: `git diff --name-only HEAD~1 | xargs grep -l "password\s*=" | head`. Fail on any match.

### C-014 | Explain `workflow_dispatch` input types and how you'd use them for an on-demand deployment with environment selection. | Implement | Medium | ⬜
**Hint:** `inputs.environment` with `type: choice`, `options: [dev, staging, prod]`. Conditionally apply environment protection rules per selection.

### C-015 | A GitHub Actions self-hosted runner is intermittently failing with "Error: ENOSPC" — no space left on device. How do you fix? | Troubleshoot | Hard | ⬜
**Hint:** Docker images accumulate on runner. `docker system prune -af` in pre-job step. Ephemeral runners (ARC) avoid this — destroyed after each job.

### C-016 | Implement a deployment workflow that rollbacks automatically on health check failure. | Implement | Expert | ⬜
**Hint:** Deploy → sleep 60 → `curl /health` → if fails: `argocd app rollback` or `helm rollback`. Alert Slack on rollback.

### C-017 | How do you implement environment-specific secrets management in GitHub Actions for 10 environments? | Design | Hard | ⬜
**Hint:** GitHub Environments with environment-specific secrets. OIDC with per-environment federated credentials (subject filter `environment:production`). Avoid repo-level secrets for environment-specific data.

### C-018 | Explain `pull_request_target` — why is it dangerous and when must you use it? | Explain | Expert | ⬜
**Hint:** `pull_request_target` runs in context of target branch (has secrets access) — dangerous for forks (attacker PR can access secrets). Only use if you explicitly verify the PR is safe. Use `pull_request` for untrusted contributions.

### C-019 | Implement a GitHub Actions pipeline that only deploys on semantic version tags (v1.2.3). | Implement | Medium | ⬜
**Hint:** `on: push: tags: ['v[0-9]+.[0-9]+.[0-9]+']`. Extract version: `RELEASE_VERSION=${GITHUB_REF#refs/tags/}`.

### C-020 | Design a GitHub Actions pipeline that produces SLSA Level 3 provenance for container images. | Design | Expert | ⬜
**Hint:** `slsa-framework/slsa-github-generator` action generates signed provenance. Reusable workflow for hermetic build. `cosign verify-attestation` to verify before deploy.

### C-021 | How do you implement status checks that block PR merges if CI hasn't passed on all required paths? | Implement | Medium | ⬜
**Hint:** GitHub branch protection `required status checks`. Use `paths-filter` to conditionally trigger jobs. Required check must always run (or use a path-dependent check aggregator job).

### C-022 | Implement a canary deployment workflow on GitHub Actions: deploy to 10% traffic, verify, promote or rollback. | Implement | Expert | ⬜
**Hint:** Deploy → `argocd app set --parameter canary.weight=10` → wait → query Prometheus for error rate → if ok: increase to 50% → 100%. On failure: `argocd app rollback`.

### C-023 | A GitHub Actions workflow needs to run on a Windows runner for .NET build and a Linux runner for Docker build in the same pipeline. | Implement | Medium | ⬜
**Hint:** Two jobs: `build-windows: runs-on: windows-latest` → upload artifact. `build-docker: runs-on: ubuntu-latest` → download artifact → `docker build`. `needs: build-windows` for sequencing.

### C-024 | Implement GitHub Deployments API integration to track deployment status and environment URLs in PR. | Implement | Hard | ⬜
**Hint:** `actions/github-script` → `github.rest.repos.createDeployment()` → `github.rest.repos.createDeploymentStatus()`. Status shown in PR timeline and Checks tab.

### C-025 | Your GitHub Actions workflow fails with "Resource not accessible by integration" on a PR from a fork. Why? | Troubleshoot | Hard | ⬜
**Hint:** Forks don't get access to secrets or write permissions on `pull_request` trigger. Use `pull_request_target` only if safe (non-code-executing steps). Or: manual re-run by maintainer with required permissions.

---

## Section 2: GitLab CI and Jenkins (C-026 to C-050)

---

### C-026 | GitLab CI DAG Pipeline Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a GitLab CI pipeline using DAG (`needs` keyword) for a microservice that has: test, security scan, build, and integration test — with optimal parallelism.

```yaml
stages: [test, build, deploy]

unit-tests:
  stage: test
  script: pytest tests/unit/

sast:
  stage: test
  script: semgrep --config=auto src/

dependency-check:
  stage: test  
  script: pip-audit -r requirements.txt

# Build starts as soon as unit-tests pass (doesn't wait for sast/dependency-check)
docker-build:
  stage: build
  needs: [unit-tests]
  script:
  - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
  - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

integration-tests:
  stage: build  # Same stage, runs after build via needs
  needs: [docker-build]
  script: pytest tests/integration/ --image=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```
- Without `needs`: each stage waits for all jobs in previous stage to complete — slow
- With `needs`: DAG — jobs run as soon as their direct dependencies complete

---

### C-027 | GitLab CI Include and Component Catalog
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a shared CI template library in GitLab using `include` and the Component Catalog for 50 repositories.

```yaml
# Template repository: templates/docker.yml
.docker-build-push:
  image: docker:24
  before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
  - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
  - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# Consumer repository: .gitlab-ci.yml
include:
- project: 'platform/ci-templates'  
  ref: 'v2.1.0'                     # Pin to tag
  file: '/templates/docker.yml'

build:
  extends: .docker-build-push        # Inherit template job
  variables:
    DOCKERFILE: Dockerfile.prod      # Override
```
- GitLab 17+ Component Catalog: versioned, discoverable templates with input parameters
- Template versioning: tag templates, consumers pin to specific version

---

### C-028 | GitLab Runner Scaling with Kubernetes Executor
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a GitLab Runner infrastructure using Kubernetes executor on AKS for autoscaling CI capacity.

**What you must cover:**
- Kubernetes executor: each CI job gets a fresh pod (executor → helper + build containers)
- Runner Helm chart: `gitlab/gitlab-runner` — registered with GitLab, watches for jobs
- Autoscaling: Kubernetes handles pod scheduling; node autoscaler adds nodes as needed
- Resource requests: `builds.requestedCPU`, `builds.requestedMemory` — set per runner
- Privileged mode: avoid — use kaniko or BuildKit for Docker-in-Docker builds without privilege
- Job isolation: each job runs in isolated pod, clean environment
- Cache: distributed cache via S3-compatible (Azure Blob) or GitLab cache backend

---

### C-029 | Jenkins Shared Library Design
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a Jenkins Shared Library that provides standard stages for: checkout, build, test, security scan, and deploy — usable by 100 Jenkinsfile across teams.

```groovy
// vars/standardPipeline.groovy
def call(Map config = [:]) {
    pipeline {
        agent { kubernetes { yaml kubernetesPodTemplate() } }
        
        stages {
            stage('Checkout') {
                steps { checkout scm }
            }
            stage('Test') {
                steps {
                    sh "make test"
                    junit 'reports/**/*.xml'
                }
            }
            stage('Security Scan') {
                steps {
                    sh "trivy image --exit-code 1 --severity CRITICAL ${config.imageName}:${env.BUILD_NUMBER}"
                }
            }
            stage('Deploy') {
                when { branch 'main' }
                steps {
                    sh "kubectl set image deployment/${config.deploymentName} ${config.containerName}=${config.imageName}:${env.BUILD_NUMBER}"
                }
            }
        }
    }
}
```
```groovy
// Service Jenkinsfile (minimal)
@Library('platform-shared-lib@v3.2.0') _
standardPipeline(imageName: 'myapp', deploymentName: 'myapp', containerName: 'app')
```

---

### C-030 | Jenkins Kubernetes Dynamic Agents
**Type:** Implement | **Difficulty:** Hard | ⬜

> Configure Jenkins to dynamically launch Kubernetes pods as build agents using the Kubernetes plugin.

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: python
    image: python:3.12-slim
    command: [cat]
    tty: true
    resources:
      requests: {cpu: 500m, memory: 512Mi}
      limits: {cpu: 2, memory: 2Gi}
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.21.0-debug
    command: [/busybox/cat]
    tty: true
'''
        }
    }
    stages {
        stage('Test') {
            steps {
                container('python') {
                    sh 'pip install -r requirements.txt && pytest'
                }
            }
        }
        stage('Build Image') {
            steps {
                container('kaniko') {
                    sh '/kaniko/executor --dockerfile=Dockerfile --destination=myacr.azurecr.io/myapp:${BUILD_NUMBER}'
                }
            }
        }
    }
}
```

---

### C-031 to C-050 | CI/CD Rapid-Fire (GitLab, Jenkins, Supply Chain)

---

### C-031 | A GitLab CI pipeline for a monorepo triggers all jobs on every commit. Implement job filtering by changed paths. | Implement | Hard | ⬜
**Hint:** `rules: changes: - services/my-service/**`. Or: `needs` with `when: never` jobs as path gates.

### C-032 | Jenkins pipeline is stuck at 'Waiting for node' for 20 minutes. Walk through debugging. | Troubleshoot | Hard | ⬜
**Hint:** Kubernetes plugin: pod didn't start. `kubectl get pods -n jenkins`. Common: image pull failure, resource quota exceeded, PVC not bound, pod security policy blocking.

### C-033 | Implement a GitLab CI pipeline that only runs on merge requests and skips on direct pushes to main. | Implement | Medium | ⬜
**Hint:** `rules: - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'`. Or: `only: merge_requests`.

### C-034 | Design a multi-project GitLab pipeline where a platform pipeline triggers service pipelines. | Design | Hard | ⬜
**Hint:** GitLab Trigger API or `trigger:` keyword in CI YAML. Downstream pipeline: separate project. Pass variables/artifacts between parent and child pipelines.

### C-035 | Implement a Jenkins pipeline with blue/green deployment using Kubernetes label switching. | Implement | Expert | ⬜
**Hint:** Deploy to `green` selector → test → patch Service selector from `blue` to `green` → wait → delete old `blue` deployment.

### C-036 | Your Jenkins shared library tests are failing because of a change in the library. How do you test library changes before merging? | Implement | Hard | ⬜
**Hint:** Jenkins Library version pinning: Jenkinsfile uses `@Library('shared-lib@feature-branch')`. Test on a non-production Jenkins folder. `JenkinsPipelineUnit` for unit testing library Groovy.

### C-037 | A GitLab pipeline fails with "Job's log exceeded limit of 4 MiB". How do you fix? | Troubleshoot | Medium | ⬜
**Hint:** Reduce CI log verbosity (`-v` flags off). Redirect verbose output to artifact file, not stdout. Split job into smaller pieces. Increase limit in GitLab admin (if self-hosted).

### C-038 | How do you implement GitLab Review Apps for Kubernetes environments? | Implement | Hard | ⬜
**Hint:** `environment: name: review/$CI_COMMIT_REF_NAME url: https://$CI_ENVIRONMENT_SLUG.preview.example.com`. Deploy job: create K8s namespace, apply manifests. Stop environment job: delete namespace.

### C-039 | Jenkins is running Bash scripts inline in Jenkinsfile. Why is this a problem and how do you fix it? | Design | Medium | ⬜
**Hint:** Inline scripts: hard to test, no error handling, no reuse. Fix: move scripts to `scripts/` directory, reference as `sh 'scripts/test.sh'`, test scripts independently.

### C-040 | Implement a GitLab CI pipeline that publishes a Python package to PyPI on version tag. | Implement | Medium | ⬜
**Hint:** `rules: - if: '$CI_COMMIT_TAG =~ /^v[0-9]+/'`. `twine upload` with `PYPI_TOKEN` as masked CI variable. Build with `python -m build`.

### C-041 | Design a GitLab CI environment with protected variables for production — only trusted users can see/use. | Design | Hard | ⬜
**Hint:** GitLab Environment: protected. Protected branches only: `main`. Protected variable: only pipelines on protected branches can access. Masked: hidden in logs.

### C-042 | Jenkins JNLP agent can't connect to Jenkins master. Diagnose. | Troubleshoot | Hard | ⬜
**Hint:** K8s pod status: running but can't connect? Jenkins master URL accessible from pod? `jenkins-agent` container logs. Firewall between agent and master (port 50000 / 443 for WebSocket). Outdated JNLP JAR version.

### C-043 | Implement a deployment gate in GitLab that requires a manual approval step in production. | Implement | Medium | ⬜
**Hint:** `when: manual` on deploy-prod job. `allow_failure: false`. Combined with protected environment: only maintainers can trigger.

### C-044 | How do you parallelise a test suite in GitLab CI when tests take 30 minutes total? | Implement | Hard | ⬜
**Hint:** `parallel: 5` — GitLab splits test runners. With pytest: `pytest --splits=5 --group=${CI_NODE_INDEX}` (pytest-split plugin). Each parallel job runs 1/5 of tests → total time ≈ 6 min.

### C-045 | Describe how to implement GitLab CI/CD pipeline for deploying Terraform changes to Azure with manual approval for prod. | Design | Expert | ⬜
**Hint:** Stages: validate → plan → apply-dev (auto) → apply-staging (manual) → apply-prod (manual, protected env). Plan output as artifact. Apply reads plan artifact (deterministic).

### C-046 | A Jenkins credential was accidentally exposed in a build log. Walk through incident response. | Troubleshoot | Hard | ⬜
**Hint:** Immediately revoke/rotate credential. Jenkins: delete old credential, add new. Check if anyone used the exposed credential (audit logs of the target service). Mask credentials: use `credentials()` wrapper, never echo.

### C-047 | How do you implement GitLab merge request approvals as a mandatory quality gate? | Implement | Medium | ⬜
**Hint:** GitLab Merge Request Approval Rules (Premium): minimum approvers, optional CODEOWNERS integration, "reset approvals on push" for invalidating stale approvals.

### C-048 | Design a Jenkins pipeline that respects feature flags — skips certain stages when flags are disabled. | Design | Hard | ⬜
**Hint:** `when { expression { env.SECURITY_SCAN_ENABLED == 'true' } }`. Flags as Jenkins parameters or Vault secrets pulled at runtime.

### C-049 | Implement GitLab CI with Docker in Docker (DinD) — explain the security implications and Docker socket alternatives. | Implement | Expert | ⬜
**Hint:** DinD: `--privileged` required — insecure. Alternative: Docker socket bind mount — shares host Docker (also risky). Best: Kaniko (rootless, no socket, no privilege), BuildKit rootless, ko for Go.

### C-050 | A GitLab pipeline triggers twice for the same commit (push + merge request events). How do you deduplicate? | Troubleshoot | Medium | ⬜
**Hint:** `workflow: rules: - if: '$CI_PIPELINE_SOURCE == "push" && $CI_OPEN_MERGE_REQUESTS' when: never`. Or: use merge request pipelines exclusively and skip push-based pipelines.

---

## Section 3: GitOps and Release Strategies (C-051 to C-090)

---

### C-051 | ArgoCD App of Apps Pattern
**Type:** Design | **Difficulty:** Expert | ⬜

> Design an ArgoCD App of Apps pattern for managing 50 applications across dev/staging/prod.

```yaml
# Parent "app of apps" application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops
    targetRevision: HEAD
    path: apps/production  # Contains Application CRs for each service
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
- `apps/production/` directory: each file is an Application CR pointing to the service's Helm chart
- Benefit: adding a new service = adding one Application CR to the apps directory
- `prune: true`: resources removed from Git are deleted from cluster
- `selfHeal: true`: manual kubectl changes reverted by ArgoCD

---

### C-052 | ArgoCD ApplicationSet — Cluster Generator
**Type:** Implement | **Difficulty:** Expert | ⬜

> Use ArgoCD ApplicationSet with a cluster generator to deploy a base workload to all registered clusters.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: monitoring-stack
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          monitoring: enabled
  template:
    metadata:
      name: '{{name}}-monitoring'
    spec:
      project: platform
      source:
        repoURL: https://github.com/org/gitops
        targetRevision: HEAD
        path: monitoring/
        helm:
          valueFiles:
          - clusters/{{name}}/monitoring-values.yaml  # Cluster-specific overrides
      destination:
        server: '{{server}}'
        namespace: monitoring
      syncPolicy:
        automated:
          prune: true
```

---

### C-053 | ArgoCD Sync Waves
**Type:** Implement | **Difficulty:** Hard | ⬜

> Use sync waves to control resource deployment order: CRDs first, then operators, then applications.

```yaml
# Wave 0: CRDs (must exist before operators)
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"

# Wave 1: Operator deployment  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cert-manager
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    argocd.argoproj.io/hook: Sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded

# Wave 2: Issuer (depends on cert-manager running)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```
- Waves are integers (negative allowed for very early resources)
- ArgoCD waits for all resources in wave N to be healthy before proceeding to wave N+1

---

### C-054 | Argo Rollouts Canary with Analysis
**Type:** Implement | **Difficulty:** Expert | ⬜

> Implement an Argo Rollouts canary rollout that automatically promotes or rolls back based on Prometheus error rate.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      canaryService: myapp-canary
      stableService: myapp-stable
      trafficRouting:
        nginx:
          stableIngress: myapp-ingress
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: error-rate-check
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
  - name: error-rate
    interval: 1m
    count: 5
    successCondition: result[0] < 0.01  # < 1% error rate
    failureLimit: 1
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"5..",app="myapp",version="canary"}[1m])) /
          sum(rate(http_requests_total{app="myapp",version="canary"}[1m]))
```

---

### C-055 | Flux GitOps Setup
**Type:** Implement | **Difficulty:** Hard | ⬜

> Set up Flux to manage a Kubernetes cluster from a GitHub repository. What components are installed and how does reconciliation work?

```bash
# Bootstrap Flux
flux bootstrap github \
  --owner=myorg \
  --repository=gitops-fleet \
  --branch=main \
  --path=clusters/prod \
  --personal

# Flux installs: source-controller, kustomize-controller, helm-controller, notification-controller
```
```yaml
# HelmRelease
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: prometheus
  namespace: monitoring
spec:
  interval: 10m
  chart:
    spec:
      chart: kube-prometheus-stack
      version: ">=60.0.0 <61.0.0"
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
  values:
    grafana:
      enabled: true
```
- `source-controller`: watches Git repos, OCI registries, Helm repos
- `kustomize-controller`: watches Kustomizations, applies changes when source changes
- Reconciliation: every `interval`, controller checks for changes and applies

---

### C-056 to C-090 | GitOps and Pipeline Rapid-Fire Scenarios

---

### C-056 | ArgoCD shows "OutOfSync" for a resource not managed in Git. What causes this and how do you handle it? | Troubleshoot | Hard | ⬜
**Hint:** External admission webhook mutated the resource after sync. `argocd.argoproj.io/compare-options: IgnoreExtraneous` per-resource. Global ignoreDifferences in App spec for known mutations.

### C-057 | Implement a GitOps promotion workflow: dev auto-deploys, staging requires PR, prod requires PR + 2 approvers. | Design | Expert | ⬜
**Hint:** ArgoCD: dev app with `automated`. Staging/prod: no `automated`, triggered by PR to update image tag. GitHub Environment protection rules for required reviewers.

### C-058 | Argo Rollouts rollback command vs ArgoCD rollback — what's the difference? | Explain | Hard | ⬜
**Hint:** `argocd rollback` reverts to previous ArgoCD sync (Git revision). `kubectl argo rollouts undo` reverts Rollout's stable version in-cluster. After Argo Rollouts undo: still out of sync with Git — must update Git too.

### C-059 | A Flux HelmRelease is stuck in "Failed" state. Walk through debugging. | Troubleshoot | Hard | ⬜
**Hint:** `flux get helmreleases -A`. `flux describe helmrelease myapp`. Check: chart values incorrect, dependency HelmRepository unreachable, resource validation failed, timeout expired.

### C-060 | Design a multi-cluster promotion pipeline: deploy to us-east1 first, then eu-west1 after validation. | Design | Expert | ⬜
**Hint:** ArgoCD ApplicationSet with sequential rollout. Kargo (Progressive delivery) for multi-cluster promotion. GitHub Actions: deploy-us → health check → deploy-eu.

### C-061 | GitLab Auto DevOps vs manual pipeline — when should you use each? | Explain | Medium | ⬜
**Hint:** Auto DevOps: quick start, convention over config, good for standard apps. Manual: custom build steps, non-standard runtime, fine-grained control over security gates.

### C-062 | Implement feature flag rollout using ArgoCD ApplicationSet and Kustomize overlays. | Implement | Expert | ⬜

### C-063 | A blue/green deployment has both blue and green running simultaneously and gets traffic. How do you debug the routing? | Troubleshoot | Hard | ⬜
**Hint:** Service selector: check which selector is pointing to which Deployment. Argo Rollouts: verify AnalysisRun completed and stable service was switched. Ingress weight: both weights nonzero.

### C-064 | Explain ArgoCD Projects and how they enforce GitOps security boundaries between teams. | Explain | Hard | ⬜
**Hint:** Project: restricts source repos, destination clusters/namespaces, resource whitelists. Team A's project: only deploys from their repo to their namespace — can't deploy cluster-level resources.

### C-065 | Implement a Flagger progressive delivery controlled by Istio traffic shifting and Prometheus metrics. | Implement | Expert | ⬜

### C-066 | How do you roll back a GitOps-managed application when the Git repository is unavailable? | Troubleshoot | Expert | ⬜
**Hint:** ArgoCD: `argocd app rollback myapp <revision>` uses cached state. Or `kubectl rollout undo`. After recovery: ensure Git change is reverted to match rolled-back state.

### C-067 | Design a ChatOps deployment workflow: deploy commands via Slack trigger CI/CD pipeline. | Design | Hard | ⬜
**Hint:** Slack App → Lambda/Azure Function webhook → trigger GitHub Actions `workflow_dispatch` via API → response posted to Slack thread. Authenticate Slack commands (signing secret).

### C-068 | Implement semantic release to automate versioning and CHANGELOG generation in GitHub Actions. | Implement | Medium | ⬜
**Hint:** `semantic-release` npm package. Conventional commits: `feat:` → minor, `fix:` → patch, `BREAKING CHANGE:` → major. Runs on merge to main: bumps version, creates tag, generates changelog.

### C-069 | Explain how ArgoCD handles CRD management. What happens if you deploy a Helm chart with CRDs via ArgoCD? | Explain | Expert | ⬜
**Hint:** ArgoCD syncs CRDs as regular manifests. Helm doesn't manage CRD upgrades (puts them in `crds/` directory). ArgoCD can manage CRDs in sync waves. Ordering: CRDs before CRs.

### C-070 | Implement a GitHub Actions workflow that builds and signs a container image with Cosign using OIDC keyless signing. | Implement | Expert | ⬜
**Hint:** `sigstore/cosign-installer` action. `COSIGN_EXPERIMENTAL=1 cosign sign` with OIDC token. No private key required — uses Fulcio CA and Rekor transparency log.

### C-071 | A production deployment went through GitOps but broke the app. How do you investigate what changed and in which commit? | Troubleshoot | Medium | ⬜
**Hint:** `argocd app history myapp --output yaml`. `git log --oneline` on the GitOps repo. `git diff <old-revision> <new-revision>`. ArgoCD health status in UI shows what was different.

### C-072 | Design a pipeline that generates an SBOM for every release and attaches it to the OCI image as an attestation. | Design | Expert | ⬜
**Hint:** `syft` generates SBOM (CycloneDX or SPDX). `cosign attest --predicate sbom.json --type cyclonedx` attaches to image. `cosign verify-attestation` in admission controller before deploy.

### C-073 | Explain Kargo (Akuity) for multi-stage progressive delivery. How does it differ from basic GitOps? | Explain | Expert | ⬜
**Hint:** Kargo: Freight (artifact bundle) + Stages (environments). Promotes freight from stage to stage with verification. Tracks which version is in which environment. Git-first GitOps with promotion automation.

### C-074 | Implement a Kubernetes pre-deployment validation job using a Helm test hook. | Implement | Hard | ⬜
**Hint:** `helm.sh/hook: test` annotation on a Pod. `helm test <release>` runs the pod, checks exit code 0. Integrate into CD: `helm upgrade && helm test`.

### C-075 | A GitLab MR pipeline triggers but the SAST job is skipped randomly. How do you investigate? | Troubleshoot | Hard | ⬜
**Hint:** Check job `rules` conditions. `CI_PIPELINE_SOURCE` not matching expected value. `allow_failure: true` causing it to appear skipped. Check `when` condition evaluations in pipeline details.

### C-076 | Design a pipeline for database schema migrations with GitOps — how do you ensure migrations run exactly once? | Design | Expert | ⬜
**Hint:** Kubernetes Job with `helm.sh/hook: pre-upgrade`. Migration tool with version tracking (Flyway, Alembic). Migration state stored in DB — running twice is idempotent (skip applied). ArgoCD sync wave: run before app deployment.

### C-077 | Implement a deploy notification system that posts deployment events to Datadog, PagerDuty, and Slack. | Implement | Hard | ⬜
**Hint:** GitHub Actions post-deploy step: `curl -X POST` to Datadog events API, Slack webhook, PagerDuty change events. ArgoCD: notification controller with Slack/Datadog triggers.

### C-078 | Explain the difference between GitOps push model and pull model. Which is more secure and why? | Explain | Medium | ⬜
**Hint:** Push: CI pipeline pushes changes to cluster (needs cluster credentials in CI). Pull: agent in cluster (ArgoCD/Flux) pulls from Git (no outbound cluster access needed). Pull is more secure — cluster credentials stay in cluster.

### C-079 | How do you implement mandatory SBOM generation and supply chain verification as a deployment gate? | Design | Expert | ⬜
**Hint:** Kyverno `verifyImages` policy: reject pods if image doesn't have valid Cosign signature + SBOM attestation. Chain: CI generates+attaches SBOM → Kyverno verifies before deployment.

### C-080 | Design a release train model (release branch per sprint) with GitOps for a 30-team organization. | Design | Expert | ⬜
**Hint:** Release branch tagged after stabilization: `release/2026-Q2-W15`. ArgoCD apps target `release/...` branch in staging, promote to prod after sign-off. Feature flags for pre-release features.

### C-081 | Implement a rollback that includes database migration rollback in GitOps. | Design | Expert | ⬜

### C-082 | Explain ArgoCD resource hooks (PreSync, Sync, PostSync, SyncFail). Give a use case for each. | Explain | Hard | ⬜
**Hint:** PreSync: run DB migration before deploying new pods. Sync: apply manifests. PostSync: run smoke test after deployment. SyncFail: send alert if sync fails.

### C-083 | A CD pipeline deploys to 20 countries with different regulatory requirements (data residency). Design the pipeline. | Design | Expert | ⬜

### C-084 | How do you implement progressive delivery for stateful services (databases, queues)? | Design | Expert | ⬜
**Hint:** Stateful services can't be easily canary'd. Strategy: feature flags in application code, blue-green at connection pool level, shadow mode for new schema. Never route partial traffic to different DB schemas.

### C-085 | Implement image tag promotion from dev → staging → prod using GitOps with image automation. | Implement | Expert | ⬜
**Hint:** Flux ImageAutomation: watches registry, updates image tag in Git for dev. Manual PR to promote to staging/prod. Or Kargo for automated promotion based on health verification.

### C-086 | A Jenkins pipeline runs fine locally but fails in CI with "command not found". How do you systematically fix? | Troubleshoot | Medium | ⬜
**Hint:** Docker agent: tool not installed in CI image. Add tool to `Dockerfile.ci` or install in `before_script`. Pin tool versions. Test agent image locally.

### C-087 | Design a DORA metrics tracking system for your CI/CD platform. How do you collect deployment frequency, lead time, MTTR, and change failure rate? | Design | Expert | ⬜
**Hint:** Deployment frequency: count successful CD pipeline runs per day from CI logs. Lead time: time from first commit to production deploy (git log timestamps). MTTR: time from incident alert to resolution (PagerDuty API). CFR: failed deployments / total (pipeline exit codes). Dashboard in Grafana.

### C-088 | Implement supply chain security in a GitHub Actions pipeline using SLSA framework requirements. | Implement | Expert | ⬜
**Hint:** SLSA L3: hermetic build (no network access during build), signed provenance (`slsa-framework/slsa-github-generator`), immutable artifacts. Verify with `slsa-verifier`.

### C-089 | How do you migrate a Jenkins Pipeline to GitHub Actions without disrupting teams in the middle of feature work? | Design | Hard | ⬜
**Hint:** Phase 1: run both in parallel (GHA for new branches, Jenkins for existing). Phase 2: feature parity validation. Phase 3: cut over by team. Phase 4: decommission Jenkins after all teams migrated.

### C-090 | Design a pipeline that deploys to Kubernetes using Helm but also manages external dependency provisioning (Azure SQL, Service Bus) via Terraform in the same pipeline run. | Design | Expert | ⬜
**Hint:** Stage 1: Terraform (provision Azure resources, output connection strings). Stage 2: write outputs to Key Vault. Stage 3: Helm deploy (ESO pulls secrets from KV). Ensure Terraform apply is idempotent.
