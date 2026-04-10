# Terraform & IaC — 80 Scenarios

> **ID prefix:** T- | Types: Implement, Design, Troubleshoot, Explain | Difficulty: Medium → Expert

---

## Section 1: Terraform Internals (T-001 to T-020)

---

### T-001 | Terraform State Deep Dive
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain the Terraform state file structure. What gets stored? What are the risks of secrets in state?

**What you must cover:**
- State file: JSON, maps resource addresses to provider-specific IDs and all attributes (including sensitive)
- Sensitive values: even `sensitive = true` variables are stored in state in plaintext — state must be encrypted at rest
- Remote state: Azure Blob (backend "azurerm") with Azure Storage encryption + SAS or Managed Identity access
- State locking: Azure Blob uses Blob Lease (pessimistic locking) — prevents concurrent applies
- State isolation: workspace per environment OR directory per environment
- Secret risk: database passwords, API keys, certificates end up in state — use Vault for dynamic secrets to avoid this

---

### T-002 | Remote State Backend Configuration
**Type:** Implement | **Difficulty:** Medium | ⬜

> Configure Terraform remote state backend on Azure Blob Storage with proper locking and access control.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstatecompany001"
    container_name       = "tfstate"
    key                  = "prod/aks-cluster.tfstate"
    use_oidc             = true  # Federated identity (no stored keys)
  }
}
```
```bash
# Storage account: enable soft delete (30 days), versioning, encryption
az storage account update \
  --name tfstatecompany001 \
  --enable-soft-delete --blob-soft-delete-retention 30 \
  --encryption-services blob

# Lock: prevent accidental deletion of state storage
az lock create --name "prevent-delete" --resource-group terraform-state-rg \
  --lock-type CanNotDelete
```
- Network: restrict storage account to trusted IPs or VNet service endpoint
- CI access: OIDC from GitHub Actions → Azure → no stored storage keys

---

### T-003 | Terraform Plan Output Analysis
**Type:** Explain | **Difficulty:** Medium | ⬜

> Given a `terraform plan` output, explain how to identify which changes will cause service disruption vs zero-downtime updates.

**What you must cover:**
- `~ update in-place`: usually safe — attribute changes applied to existing resource
- `-/+ destroy and recreate`: disrupting — resource deleted and recreated (e.g., changing VM name, AKS node pool VM size)
- `create_before_destroy = true` lifecycle: new resource created first, then old destroyed — reduces downtime window
- Red flags in plan: `forces replacement`, `destroy`, `resource must be replaced`
- AKS: changing `vm_size` on node pool forces replacement of the node pool — use blue-green node pool strategy instead
- Azure Resource IDs: immutable in Terraform — changing any ID-forming attribute forces replace

---

### T-004 | Terraform `moved` Block
**Type:** Implement | **Difficulty:** Hard | ⬜

> You've renamed a module or refactored resources inside a module. How do `moved` blocks prevent unnecessary resource destruction?

```hcl
# Old: resource "azurerm_resource_group" "rg"
# New: module "networking" → resource "azurerm_resource_group" "main"

moved {
  from = azurerm_resource_group.rg
  to   = module.networking.azurerm_resource_group.main
}
```
- Without `moved`: Terraform destroys old resource, creates new — downtime!
- With `moved`: state is updated, no physical resource change — plan shows no-op
- `moved` blocks can be removed after the migration is complete and state is committed
- `terraform state mv`: older approach, manual, error-prone, not VCS-tracked

---

### T-005 | Terraform Import (1.5+ block syntax)
**Type:** Implement | **Difficulty:** Hard | ⬜

> Resources exist in Azure that were created manually. Import them into Terraform state without destroying and recreating.

```hcl
# Terraform 1.5+ declarative import
import {
  to = azurerm_resource_group.existing
  id = "/subscriptions/<sub-id>/resourceGroups/my-rg"
}

# Generate config (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf
```
```bash
# Classic approach (still works)
terraform import azurerm_resource_group.existing \
  /subscriptions/<sub-id>/resourceGroups/my-rg

# Then write matching config to avoid drift
```
- After import: `terraform plan` should show no planned changes if config matches reality
- Gotcha: imported resource may have attributes Terraform doesn't manage — `ignore_changes` for unmanaged attributes

---

### T-006 | Lifecycle Rules — When to Use Each
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain `create_before_destroy`, `prevent_destroy`, `ignore_changes`, and `replace_triggered_by`. Give a real scenario for each.

**What you must cover:**
```hcl
resource "azurerm_kubernetes_cluster" "main" {
  lifecycle {
    # Prevent destroy on production cluster
    prevent_destroy = true

    # Ignore manual node count changes (managed by CAS)
    ignore_changes = [default_node_pool[0].node_count]

    # Recreate if certificate changes
    replace_triggered_by = [azurerm_key_vault_certificate.tls]
  }
}
```
- `create_before_destroy`: new resource ready before old is torn down — use for LBs, DNS records
- `prevent_destroy`: hard error if plan includes destroy — use on stateful prod resources
- `ignore_changes`: prevent Terraform from reverting external changes (e.g., autoscaler-managed node count)
- `replace_triggered_by`: trigger replacement of one resource when another changes — use for rolling cert updates

---

### T-007 | Module Design Patterns
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a Terraform module for an AKS cluster. Define its inputs, outputs, and validation.

```hcl
# variables.tf
variable "cluster_name" {
  type = string
  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,39}$", var.cluster_name))
    error_message = "Cluster name must be 3-40 lowercase alphanumeric with hyphens."
  }
}

variable "node_count" {
  type    = number
  default = 3
  validation {
    condition     = var.node_count >= 1 && var.node_count <= 100
    error_message = "Node count must be between 1 and 100."
  }
}

# outputs.tf
output "cluster_id" {
  value     = azurerm_kubernetes_cluster.main.id
  sensitive = false
}

output "kube_config" {
  value     = azurerm_kubernetes_cluster.main.kube_config_raw
  sensitive = true  # Keeps out of CLI output
}
```
- Module versioning: publish to Terraform registry or private GitHub releases with semver tags
- `required_providers` in module: specify minimum provider version constraint
- `description` on every variable: mandatory for usable modules

---

### T-008 | Terraform Workspaces vs Directory per Environment
**Type:** Explain | **Difficulty:** Hard | ⬜

> Compare Terraform workspaces vs directory-per-environment for managing dev/staging/prod. What are the risks of each?

**What you must cover:**
- Workspaces: same codebase, different state files. Easy for simple differences, hard to maintain large env divergence
- Workspace risk: `terraform workspace select prod` then `terraform destroy` — easy mistake in terminal
- Directory per environment: `environments/dev/`, `environments/staging/`, `environments/prod/`. Explicit, auditable, different variable files.
- Terragrunt: DRY wrapper — single module source, per-env `terragrunt.hcl` with `inputs = {}`. Best of both worlds.
- Recommendation: directory per environment for meaningful differences; workspaces for trivial differences (region, size only)
- State isolation: separate state file per environment regardless of approach

---

### T-009 | `for_each` vs `count` — When to Use Each
**Type:** Explain | **Difficulty:** Medium | ⬜

> What are the trade-offs between `count` and `for_each`? When does `count` cause problems?

```hcl
# DO NOT use count for ordered sets where middle elements might be removed
# count problem: remove item at index 1 → all subsequent resources renamed/recreated
resource "azurerm_resource_group" "rgs" {
  count    = length(var.regions)
  name     = "rg-${var.regions[count.index]}"
}

# Use for_each for stable keys
resource "azurerm_resource_group" "rgs" {
  for_each = toset(var.regions)
  name     = "rg-${each.key}"
}
```
- `for_each` with map: `each.key` and `each.value` available — stable addressing by key
- `count.index` addressing: `azurerm_resource_group.rgs[0]` — breaks if list order changes
- `for_each` on module: each module instance addressable by key in plan output
- Use count: only for resources where index doesn't matter and list won't have removals from middle

---

### T-010 | Dynamic Blocks
**Type:** Implement | **Difficulty:** Hard | ⬜

> Implement a dynamic block to conditionally create security rules on an NSG based on a variable list.

```hcl
variable "inbound_rules" {
  type = list(object({
    name       = string
    priority   = number
    protocol   = string
    port       = string
    source     = string
  }))
  default = []
}

resource "azurerm_network_security_group" "main" {
  name                = var.nsg_name
  location            = var.location
  resource_group_name = var.resource_group_name

  dynamic "security_rule" {
    for_each = var.inbound_rules
    content {
      name                       = security_rule.value.name
      priority                   = security_rule.value.priority
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = security_rule.value.protocol
      destination_port_range     = security_rule.value.port
      source_address_prefix      = security_rule.value.source
      destination_address_prefix = "*"
    }
  }
}
```
- Avoid dynamic blocks for simple cases — harder to read. Use only when the block count is truly variable.

---

### T-011 | Terraform Providers and Provider Locking
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain `.terraform.lock.hcl`. Why must it be committed to version control? What happens if it's not?

**What you must cover:**
- Lock file: records exact provider version + checksums — ensures reproducible `terraform init`
- Without lock file: `terraform init` might pull a newer provider version, potentially breaking your config
- Commit: MUST be committed to Git — treated like `package-lock.json` or `Pipfile.lock`
- Update provider: `terraform init -upgrade` — updates lock file to new provider version
- Provider checksum: verifies downloaded provider matches expected binary SHA256
- CI: `terraform init` with committed lock file = deterministic, no network surprises

---

### T-012 | Terraform Sensitive Variables and State Exposure
**Type:** Design | **Difficulty:** Hard | ⬜

> A team is storing database passwords as Terraform variables and they're ending up in CI logs and state. Design a secure approach.

**What you must cover:**
- Problem 1: `terraform plan -var="db_password=..."` leaks in CI if logs aren't masked
- Problem 2: even `sensitive = true` outputs are stored plaintext in state
- Solution: don't put secrets in Terraform at all — generate them in Vault, reference them
```hcl
# Use Vault secrets engine - dynamic DB credentials
data "vault_database_secret_backend_creds" "db" {
  backend = "database"
  role    = "my-role"
}

resource "kubernetes_secret" "db_creds" {
  data = {
    username = data.vault_database_secret_backend_creds.db.username
    password = data.vault_database_secret_backend_creds.db.password
  }
}
```
- Alternative: generate random password in Terraform with `random_password`, store in Key Vault resource — still ends up in state but encrypted at Key Vault level

---

### T-013 | Terraform Testing with Terratest
**Type:** Implement | **Difficulty:** Expert | ⬜

> Write a Terratest test for an AKS module that verifies: cluster is created, has 3 nodes, and kubeconfig is valid.

```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
    "k8s.io/client-go/tools/clientcmd"
)

func TestAKSModule(t *testing.T) {
    opts := &terraform.Options{
        TerraformDir: "../modules/aks",
        Vars: map[string]interface{}{
            "cluster_name": "test-cluster",
            "node_count":   3,
        },
    }
    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    clusterName := terraform.Output(t, opts, "cluster_name")
    assert.Equal(t, "test-cluster", clusterName)

    kubeconfig := terraform.Output(t, opts, "kube_config")
    _, err := clientcmd.RESTConfigFromKubeConfig([]byte(kubeconfig))
    assert.NoError(t, err, "kubeconfig should be valid")
}
```
- Cost: Terratest creates real Azure resources — use dedicated test subscription, destroy always runs
- Idempotency test: apply twice, verify no changes on second apply

---

### T-014 | checkov and tfsec in CI
**Type:** Implement | **Difficulty:** Medium | ⬜

> Integrate checkov into a GitHub Actions CI pipeline to block PRs that introduce Terraform security violations.

```yaml
- name: Checkov IaC Security Scan
  uses: bridgecrewio/checkov-action@master
  with:
    directory: terraform/
    framework: terraform
    output_format: sarif
    output_file_path: results.sarif
    soft_fail: false  # Fail CI on HIGH findings
    skip_check: CKV_AZURE_1,CKV_AZURE_2  # Intentional suppressions with justification

- name: Upload SARIF to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: results.sarif
```
- Inline suppression (with justification):
```hcl
resource "azurerm_storage_account" "main" {
  #checkov:skip=CKV_AZURE_44:Soft delete enabled via policy, not per-account
}
```
- tfsec: `tfsec terraform/ --minimum-severity HIGH`
- infracost: `infracost diff --path terraform/` → comment cost delta on PR

---

### T-015 | Terraform State Manipulation Commands
**Type:** Implement | **Difficulty:** Hard | ⬜

> When would you use `terraform state rm`, `terraform state mv`, and `terraform state pull/push`? Give a production scenario for each.

**What you must cover:**
- `terraform state rm`: remove a resource from state without destroying it (e.g., resource now managed by another team's Terraform)
```bash
terraform state rm module.networking.azurerm_virtual_network.main
```
- `terraform state mv`: rename or move resource in state (before `moved` block existed)
```bash
terraform state mv azurerm_resource_group.old module.rg.azurerm_resource_group.main
```
- `terraform state pull`: download remote state to stdout — use for debugging or manual analysis
- `terraform state push`: upload modified local state to remote — use with extreme caution (can corrupt state)
- Always back up state before manipulation: `terraform state pull > backup-$(date +%Y%m%d).tfstate`

---

### T-016 | Terragrunt Deep Dive
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a Terragrunt-based repository structure for 3 environments (dev/staging/prod) with shared modules and per-env config.

```
.
├── modules/          # Source modules (or reference registry)
├── live/
│   ├── dev/
│   │   ├── terragrunt.hcl    # Root config: backend, provider, global inputs
│   │   ├── networking/
│   │   │   └── terragrunt.hcl  # Source module + env-specific inputs
│   │   └── aks/
│   │       └── terragrunt.hcl
│   ├── staging/
│   └── prod/
└── _envcommon/
    └── aks.hcl       # Shared non-secret inputs for all envs
```
```hcl
# live/prod/aks/terragrunt.hcl
terraform {
  source = "git::https://github.com/org/tf-modules.git//aks?ref=v2.3.0"
}

dependency "networking" {
  config_path = "../networking"
  mock_outputs = {
    vnet_id    = "mock-vnet-id"
    subnet_ids = ["mock-subnet"]
  }
}

inputs = {
  cluster_name = "prod-aks-eastus"
  vnet_id      = dependency.networking.outputs.vnet_id
  node_count   = 5
}
```
- `run-all apply`: apply entire environment (resolves dependency order)
- `mock_outputs`: allows planning without real dependency state (for plan-only CI)

---

### T-017 | Terraform Cloud / Enterprise vs Local CI
**Type:** Explain | **Difficulty:** Medium | ⬜

> Compare Terraform Cloud, Spacelift, and Atlantis for managing Terraform at scale across a 50-engineer team.

**What you must cover:**
- Atlantis: self-hosted, PR-based applying (`atlantis plan` / `atlantis apply` comments), open source, simple
- Terraform Cloud: hosted state, remote execution, workspace per env, Sentinel policies, team management
- Spacelift: GitOps-first, stack (workspace) concept, policy testing, K8s operator, supports multi-vendor
- Recommendation at 50 engineers: Atlantis (self-hosted, audit-friendly) or Terraform Cloud (less ops)
- Key requirement: all applies go through CI, no manual `terraform apply` from laptops in prod

---

### T-018 | Drift Detection and Reconciliation
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a drift detection system that runs `terraform plan` periodically and alerts on unexpected changes.

```yaml
# GitHub Actions scheduled workflow
on:
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours

jobs:
  drift-check:
    steps:
    - name: Terraform Plan
      run: terraform plan -out=drift.tfplan -detailed-exitcode
      # Exit 0: no changes. Exit 2: changes detected. Exit 1: error.
      continue-on-error: true

    - name: Alert on drift
      if: steps.plan.outputs.exitcode == '2'
      run: |
        curl -X POST $SLACK_WEBHOOK \
          -d '{"text": "Drift detected in prod/aks! Check CI: ${{ github.run_url }}"}'
```
- `-detailed-exitcode`: exit code 2 = changes exist (drift)
- Schedule frequency: balance against API rate limits (AzureRM: 1200 requests/minute)
- Drift response policy: auto-apply for additive changes (new tag), manual review for destructive changes

---

### T-019 | Terraform Module Registry and Versioning
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a module versioning strategy for a company with 20 modules shared across 10 teams. How do you handle breaking changes?

**What you must cover:**
- Semantic versioning: `v1.2.3` — breaking changes bump major (v1→v2)
- Module source: `git::https://github.com/org/terraform-modules.git//aks?ref=v2.1.0`
- Private registry: Terraform Cloud module registry, GitHub Packages, Artifactory
- Breaking change process: publish v2.0.0 → deprecation notice on v1.x → 90-day migration window → remove v1
- Testing: Terratest runs on every module PR, tag gates the version publish
- Changelog: `CHANGELOG.md` per module, linked from release tag

---

### T-020 | OPA/Conftest for Terraform Policy
**Type:** Implement | **Difficulty:** Expert | ⬜

> Write a Conftest/OPA policy that blocks Terraform plans which would create any storage account without HTTPS-only enabled.

```rego
# policy/azure/storage.rego
package main

import future.keywords.if

deny[msg] if {
  resource := input.planned_values.root_module.resources[_]
  resource.type == "azurerm_storage_account"
  not resource.values.enable_https_traffic_only
  msg := sprintf(
    "Storage account '%s' must have HTTPS-only traffic enabled",
    [resource.address]
  )
}
```
```bash
# In CI: convert plan to JSON and test
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan > plan.json
conftest test plan.json --policy policy/
```
- Policy-as-code in CI: shift-left security enforcement — catch before `apply`
- Sentinel (TF Enterprise): managed policy sets applied in Terraform Cloud workspace

---

## Section 2: IaC Patterns and Design (T-021 to T-050)

---

### T-021 | Modular AKS Deployment Structure
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a modular Terraform codebase for deploying AKS with networking, identity, monitoring, and GitOps in separate modules with dependencies.

**Module structure:**
```
modules/
├── resource-group/     # RG creation
├── networking/         # VNet, subnets, NSGs
├── identity/           # Managed identities, role assignments
├── aks-cluster/        # Core AKS resource
├── aks-addons/         # Monitoring, policy, workload identity
├── acr/                # Container registry
└── monitoring/         # Azure Monitor workspace, Grafana
```
- Module dependencies: `aks-cluster` depends on `networking` and `identity`
- Output chaining: networking outputs subnet IDs → aks-cluster input
- Root module: orchestrates all modules, passes outputs between them
- Separate state per logical boundary: `networking.tfstate` separate from `aks.tfstate`

---

### T-022 | Terraform for Kubernetes Resources
**Type:** Explain | **Difficulty:** Hard | ⬜

> When should you use Terraform (kubernetes provider) vs Helm vs kubectl/kustomize to manage Kubernetes resources?

**What you must cover:**
- Terraform kubernetes provider: good for cluster-level resources (namespaces, RBAC, storage classes, CRDs) — managed alongside infrastructure
- Helm: application deployments, lifecycle management (upgrade, rollback), templating for app configs
- kubectl/kustomize + GitOps (ArgoCD/Flux): stateless app manifests, GitOps-managed, change tracking via Git
- Pattern: Terraform provisions cluster + platform infrastructure → Helm/ArgoCD deploys applications
- Avoid: Terraform managing Deployment/Pod resources — Kubernetes handles reconciliation better than Terraform
- Danger: `helm_release` in Terraform — Helm state and Terraform state diverge on manual Helm operations

---

### T-023 | Data Sources — When and How
**Type:** Implement | **Difficulty:** Medium | ⬜

> You need to reference an existing resource not managed by this Terraform workspace. How do you use data sources?

```hcl
# Reference existing resource group
data "azurerm_resource_group" "existing" {
  name = "platform-shared-rg"
}

# Reference existing VNet created by networking team's Terraform
data "azurerm_virtual_network" "hub" {
  name                = "hub-vnet"
  resource_group_name = data.azurerm_resource_group.existing.name
}

# Use in resource
resource "azurerm_subnet" "app" {
  resource_group_name  = data.azurerm_resource_group.existing.name
  virtual_network_name = data.azurerm_virtual_network.hub.name
  # ...
}
```
- Remote state data source: `data.terraform_remote_state.networking.outputs.vnet_id` — preferred for cross-workspace data
- Data source reads at plan time — if external resource changes between plan and apply, can cause issues

---

### T-024 | Terraform Null Resources and Provisioners
**Type:** Explain | **Difficulty:** Hard | ⬜

> When is it acceptable to use `null_resource` with `local-exec` or `remote-exec` in Terraform? What's the best practice?

**What you must cover:**
- Bad use case: configuring servers, running scripts, anything that should be in Ansible/cloud-init
- Acceptable use case: running a one-time script when no Terraform resource exists (e.g., calling Azure CLI for a feature not yet in provider)
- `local-exec` runs on the machine running Terraform — works in CI, fragile otherwise
- `triggers`: invalidate null_resource when inputs change: `triggers = { cluster_id = azurerm_kubernetes_cluster.main.id }`
- Alternative: `azapi_resource` or `azapi_update_resource` — uses Azure API directly, tracked in state
- Last resort: create a custom provider or use `terraform_data` (1.4+) for lightweight data

---

### T-025 | `azapi` Provider for Unsupported Resources
**Type:** Implement | **Difficulty:** Expert | ⬜

> An Azure feature is not yet in the AzureRM provider. Implement it using the `azapi` provider.

```hcl
terraform {
  required_providers {
    azapi = {
      source  = "azure/azapi"
      version = "~> 1.13"
    }
  }
}

resource "azapi_resource" "aks_preview_feature" {
  type      = "Microsoft.ContainerService/managedClusters@2024-02-01"
  name      = var.cluster_name
  parent_id = azurerm_resource_group.main.id
  location  = var.location

  body = jsonencode({
    properties = {
      agentPoolProfiles = [{
        name   = "nodepool1"
        count  = 3
        # Preview feature not yet in azurerm provider
        enableCustomCATrust = true
      }]
    }
  })
}
```
- `azapi_update_resource`: update specific properties of existing resource managed by azurerm
- Limitation: no schema validation, raw API — requires reading Azure REST API docs

---

### T-026 to T-080 | Terraform Rapid-Fire Scenarios

---

### T-026 | A `terraform apply` failed midway. Some resources were created, others not. How do you recover? | Troubleshoot | Hard | ⬜
**Hint:** State contains partial resources. Review what was created. Fix the error in config. Re-run `terraform apply` — Terraform will create missing resources, skip existing. Never manually alter state unless absolutely needed.

### T-027 | Two engineers ran `terraform apply` simultaneously. What happens? How do you prevent this? | Troubleshoot | Hard | ⬜
**Hint:** Remote backend with locking: second apply gets "Error acquiring the state lock". If lock file is stale (crash): `terraform force-unlock <lock-id>` — dangerous, confirm no one is actually applying.

### T-028 | A team wants to destroy dev infrastructure every Friday and recreate Monday. Design the automation. | Design | Medium | ⬜
**Hint:** GitHub Actions scheduled workflow: `terraform destroy -auto-approve` Friday 6pm. `terraform apply -auto-approve` Monday 7am. State preserved in remote backend.

### T-029 | Explain `terraform refresh` and when it can be dangerous. | Explain | Hard | ⬜
**Hint:** `refresh` syncs state with actual cloud resources, discards local state divergences. Dangerous: if cloud resources were intentionally modified, refresh overwrites your knowledge of drift. `terraform plan -refresh-only` is safer (shows what refresh would change).

### T-030 | You need to rename a resource in Terraform without destroying it. Walk through the options. | Implement | Medium | ⬜
**Hint:** Option 1: `moved` block (cleanest, tracked in VCS). Option 2: `terraform state mv` (manual, immediate). Option 3: `terraform import` the resource under the new name + `terraform state rm` the old name.

### T-031 | A Terraform module has 50 variables with no defaults. Design a better API. | Design | Hard | ⬜
**Hint:** Group related variables into objects. Provide sensible defaults. Use `optional()` for optional object attributes (TF 1.3+). Keep required variables to the essential minimum.

### T-032 | Implement a Terraform configuration that uses different provider configurations per environment. | Implement | Hard | ⬜
**Hint:** `provider "azurerm" { alias = "prod" subscription_id = var.prod_subscription }`. Resources: `provider = azurerm.prod`. Or use separate Terraform workspaces/directories.

### T-033 | Explain Terraform's dependency graph. How do you express an implicit vs explicit dependency? | Explain | Medium | ⬜
**Hint:** Implicit: reference another resource's attribute (`${azurerm_resource_group.main.name}`). Explicit: `depends_on = [azurerm_role_assignment.main]`. Use explicit only when no attribute reference exists (execution order dependency only).

### T-034 | A Terraform plan shows 100 resources changing due to a provider upgrade. How do you validate this is safe before applying? | Troubleshoot | Hard | ⬜
**Hint:** Review provider changelog for breaking changes. Use `terraform show -json plan.tfplan | jq` to analyze changes programmatically. Apply to staging first. Use `-target` to apply a subset.

### T-035 | Implement tagging enforcement for all Azure resources via a Terraform module. | Implement | Medium | ⬜
**Hint:** `variable "required_tags" { type = map(string) }` + `merge(var.required_tags, local.default_tags)`. Pass to all resources. Provider-level `default_tags` block in azurerm provider.

### T-036 | How do you handle Terraform configuration for resources that take a long time to create (30+ minutes)? | Design | Hard | ⬜
**Hint:** `timeouts` block in resource: `create = "60m"`. Background: Terraform waits synchronously. Workaround: async resource creation via azapi. Split long resources into separate apply stages.

### T-037 | Implement Terraform to deploy a GitHub Actions self-hosted runner on AKS. | Implement | Hard | ⬜
**Hint:** Terraform: AKS cluster, namespace, RBAC. Helm release: actions-runner-controller (ARC). Kubernetes manifest: RunnerDeployment or ScaledActionRunner.

### T-038 | Explain `terraform output` — how to use outputs in CI to drive downstream deployments. | Explain | Medium | ⬜
**Hint:** `terraform output -json > outputs.json`. Pass AKS cluster name, resource group to `az aks get-credentials`. Pass ACR name to `docker push`. Use `terraform_remote_state` for cross-workspace consumption.

### T-039 | A terraform plan shows a change you didn't make. What are the likely causes? | Troubleshoot | Hard | ⬜
**Hint:** Provider upgrade changed default values. Someone made a manual change (drift). Azure automatically modified a property (tags by Azure Policy). `ignore_changes` is needed. Random resource regenerating.

### T-040 | Design a Terraform approval workflow using GitHub Pull Requests and Atlantis. | Design | Hard | ⬜
**Hint:** PR → Atlantis auto-plans → plan posted as PR comment → reviewer approves + comments `atlantis apply` → Atlantis applies → PR merged. For prod: require explicit approval from platform team.

### T-041 | What happens when you add a `required_providers` constraint that's stricter than the lock file? | Explain | Medium | ⬜
**Hint:** `terraform init` fails because locked provider doesn't satisfy new constraint. Fix: update lock file with `terraform init -upgrade` (if new constraint is compatible with locked version).

### T-042 | Implement Terraform to dynamically create Azure Role Assignments for a list of service principals. | Implement | Hard | ⬜
**Hint:** `for_each = { for sp in var.service_principals : sp.name => sp }`. `scope = each.value.scope`. `role_definition_name = each.value.role`.

### T-043 | A junior engineer split a large Terraform state into smaller states. Now you can't reference data across them. How do you fix? | Troubleshoot | Hard | ⬜
**Hint:** `data "terraform_remote_state"` to read outputs from another state. Or use a shared variable store (Consul KV, Azure App Config, Vault).

### T-044 | Explain Terraform Cloud workspaces vs local workspaces. Key differences? | Explain | Medium | ⬜
**Hint:** TF Cloud workspace: remote state, remote execution, variables, team access. Local workspace (`terraform workspace`): only isolates state file, same code+config, same local execution.

### T-045 | How do you implement Terraform configuration for multiple Azure subscriptions managed by one team? | Design | Expert | ⬜
**Hint:** Provider aliases per subscription. Terragrunt: per-subscription sub-directory, `inputs = { subscription_id = ... }`. Management Group: centralized Terraform manages all.

### T-046 | Implement a Terraform module that creates different resources based on a `var.environment` (dev vs prod). | Implement | Hard | ⬜
**Hint:** Use locals + conditional expressions: `local.is_prod = var.environment == "prod"`. `count = local.is_prod ? 1 : 0` on prod-only resources. Or: `node_count = local.is_prod ? 5 : 1`.

### T-047 | Explain GitHub Actions OIDC integration with Azure for Terraform runs with no stored credentials. | Implement | Expert | ⬜
**Hint:** `azure/login@v2` with `client-id`, `tenant-id`, `subscription-id` (no secret). OIDC token from GitHub → Azure Federated Credential on app registration or managed identity → access token.

### T-048 | A Terraform resource shows "Error: A resource with the ID already exists". How do you resolve? | Troubleshoot | Hard | ⬜
**Hint:** Resource exists in Azure but not in Terraform state. Fix: `terraform import` the existing resource. Or: allow import in resource block (Terraform 1.5+ `import {}` block).

### T-049 | Implement Terraform for a multi-region deployment of an AKS cluster with Traffic Manager failover. | Design | Expert | ⬜
**Hint:** Two AKS cluster resources in different regions (using provider aliases or separate configs). Traffic Manager profile + endpoints pointing to each cluster's Ingress IP. Geo-replication for ACR.

### T-050 | How do you test that a Terraform module is idempotent? Write a test. | Implement | Expert | ⬜
**Hint:** Terratest: Apply → Apply again → assert plan has no changes (`terraform.PlanNoChanges(t, opts)`). Any non-idempotent behavior (e.g., timestamp in resource name) fails this test.

### T-051 | Explain `terraform validate` vs `terraform plan` vs tflint. What does each catch that others don't? | Explain | Medium | ⬜

### T-052 | A network security group rule created by Terraform conflicts with an Azure Policy that auto-creates rules. How do you handle? | Troubleshoot | Hard | ⬜
**Hint:** Azure Policy adds a rule → Terraform sees drift → plans to remove it → `ignore_changes = [security_rule]`. Or: use `azurerm_network_security_rule` as separate resource to give Terraform atomistic control.

### T-053 | Implement Terraform for AKS node pool autoscaling with min/max constraints. | Implement | Medium | ⬜

### T-054 | How do you ensure Terraform changes to production always go through a 4-eyes approval? | Design | Hard | ⬜
**Hint:** GitHub branch protection + required reviewers on PR. Atlantis `apply` locked behind explicit approve command from authorized team member. Record of who applied what is in Git.

### T-055 | Explain the `precondition` and `postcondition` blocks (Terraform 1.2+). Give a real use case. | Explain | Hard | ⬜
**Hint:** `precondition`: validate assumption before resource creation. `postcondition`: validate outcome after. Example: `precondition { condition = var.node_count >= 3 error_message = "HA requires 3+ nodes" }`.

### T-056 | A Terraform plan is timing out in CI. The providers take too long to initialize. Optimize. | Troubleshoot | Medium | ⬜
**Hint:** Cache `.terraform` directory between CI runs. Use provider mirror for air-gapped environments. Lock file ensures deterministic downloads. Split monolith state into smaller focused states (faster plan).

### T-057 | Implement Terraform to enforce that all S3 buckets (or Azure Storage) have versioning enabled, or fail plan. | Implement | Hard | ⬜
**Hint:** `precondition` block inside storage module. Or: checkov in CI. Or: OPA policy against the plan JSON.

### T-058 | How do you handle Terraform state file migrations (moving from local to remote backend)? | Implement | Medium | ⬜
**Hint:** Add `backend "azurerm" {}` block → `terraform init -reconfigure` → Terraform prompts to copy state to new backend. Verify with `terraform state list`.

### T-059 | Design a Terraform module that's reusable for both AKS cluster provisioning and adding node pools to an existing cluster. | Design | Expert | ⬜
**Hint:** Two modules: `aks-cluster` (creates cluster), `aks-nodepool` (adds pool to existing cluster). Root module composes them. `aks-nodepool` takes cluster name + resource group as inputs (data source lookup).

### T-060 | Explain how Terraform handles Azure resources that Azure automatically modifies after creation (managed by Azure). | Explain | Hard | ⬜
**Hint:** Azure RBAC auto-assignment, AKS node labels auto-applied, Azure Policy auto-tag. Solution: `lifecycle { ignore_changes = [tags["CreatedBy"], tags["ManagedBy"]] }`. Or: use `prevent_destroy` and accept drift on non-critical attributes.

### T-061 | Implement a `check` block (Terraform 1.5+) that warns when an AKS cluster version is nearing end of support. | Implement | Expert | ⬜

### T-062 | How do you implement a Terraform deployment pipeline that supports different approval tiers per environment? | Design | Expert | ⬜
**Hint:** GitHub environments: `dev` (auto-approve), `staging` (1 approver), `prod` (2 approvers + manager). Each environment has its own secrets (Azure subscription, state key). Atlantis alternative: per-repo config.

### T-063 | Explain Terraform's behavior when a resource timeout occurs during a long-running Azure operation. | Explain | Hard | ⬜
**Hint:** Terraform times out → marks resource as tainted → next apply tries to replace it → can cause double-creation. Fix: increase `timeouts { create = "90m" }`. Check Azure portal for actual status before re-applying.

### T-064 | Can you run Terraform against an Azure resource protected by a lock? What happens? | Troubleshoot | Medium | ⬜
**Hint:** Azure Management Lock (CanNotDelete, ReadOnly) → Terraform apply/destroy fails with "AuthorizationFailed" or "ResourceReadOnly". Must remove lock first. Terraform doesn't manage locks in-flow automatically.

### T-065 | Implement Terraform to create an AKS cluster with Workload Identity enabled and register a service account federation. | Implement | Expert | ⬜

### T-066 | Explain why `data.azurerm_subscription.current` is useful in Terraform and when it can cause problems. | Explain | Medium | ⬜
**Hint:** Useful: get subscription ID without hardcoding. Problem: if Terraform runs under a different subscription context in CI, data source returns wrong values — always validate.

### T-067 | How do you structure Terraform code so developers can deploy feature branch environments without touching production state? | Design | Hard | ⬜
**Hint:** `var.environment` variable → dynamic naming. Separate state key per environment (using `prefix` or workspace). Dev permissions: can apply to dev state bucket prefix only.

### T-068 | A Terraform destroy fails because a resource has dependencies Azure won't let you remove in order. | Troubleshoot | Hard | ⬜
**Hint:** Example: deleting VNet before private endpoints are deleted. Terraform destroys in reverse dependency order. Fix: ensure all child resources are in Terraform so Terraform manages deletion order. Or: manual pre-removal of blocking resources.

### T-069 | Implement module input validation that checks an IP CIDR block is within an allowed range. | Implement | Hard | ⬜
**Hint:** `validation { condition = cidrnetmask(var.address_space) == "255.255.0.0" error_message = "Must be /16" }`. Or use regex for RFC1918 ranges.

### T-070 | Explain `terraform graph` output. How do you use it to debug unexpected resource ordering? | Explain | Medium | ⬜

### T-071 | How do you handle provider feature flags and preview features in Terraform azurerm provider? | Implement | Hard | ⬜
**Hint:** `provider "azurerm" { features { key_vault { purge_soft_deleted_secrets_on_destroy = false } } }`. Feature flags override default behavior — critical for production safety.

### T-072 | Design a strategy for gradually migrating 200 manually created Azure resources into Terraform management. | Design | Expert | ⬜
**Hint:** Phase 1: `terraform import` + `generated.tf`. Phase 2: fix drift (align TF config to actual). Phase 3: CI enforcement (no manual changes). Priority: highest-risk resources first (networking, AKS).

### T-073 | Implement Terraform to deploy a Helm chart to AKS as part of cluster provisioning. | Implement | Hard | ⬜
**Hint:** `helm_release` resource after `azurerm_kubernetes_cluster` with `depends_on`. Provider: `kubernetes` + `helm`. Get cluster credentials from azurerm outputs.

### T-074 | A `for_each` loop has a value that's not known until apply. Terraform complains. How do you fix? | Troubleshoot | Expert | ⬜
**Hint:** `for_each` and `count` must be known at plan time. If value comes from resource output, workaround: use `var` instead of resource output for for_each key, or restructure to avoid unknown-at-plan values as instance keys.

### T-075 | How do you implement Terraform for AKS with multiple environments using GitHub Actions matrix strategy? | Implement | Hard | ⬜

### T-076 | Explain the difference between `az resource lock` and Terraform `prevent_destroy`. | Explain | Medium | ⬜
**Hint:** Azure lock: platform-level, prevents deletion even outside Terraform. `prevent_destroy`: Terraform-only, errors during plan, doesn't prevent Azure portal/CLI deletion.

### T-077 | Implement Terraform with Azure Private DNS integration that auto-creates records for new services. | Implement | Expert | ⬜

### T-078 | A checkov scan shows 15 violations but 10 are intentional exceptions. How do you manage suppressions without losing audit ability? | Design | Hard | ⬜
**Hint:** Inline `#checkov:skip=CKV_AZURE_X:Justification reason` with mandatory justification comment. PR review: reviewer must explicitly acknowledge each suppression. Track suppressions in a registry doc.

### T-079 | Explain how Terraform manages Azure Tags across resources. What's the `merge()` pattern? | Implement | Medium | ⬜
**Hint:** `locals { common_tags = { environment = var.environment, team = var.team, managed_by = "terraform" } }`. Resource: `tags = merge(local.common_tags, var.additional_tags)`. Provider `default_tags`: applies to all resources automatically.

### T-080 | Design a Terraform root module for a complete production environment: networking, AKS, ACR, Key Vault, monitoring. What are the apply phases? | Design | Expert | ⬜
**Hint:** Phase 1: resource groups + networking (foundation). Phase 2: identities + key vault (security). Phase 3: AKS cluster. Phase 4: ACR + monitoring addons. Phase 5: workload infrastructure (namespaces, RBAC). `-target` or separate Terragrunt stacks.
