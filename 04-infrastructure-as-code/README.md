# Infrastructure as Code (Terraform)

## Folder Structure

```
04-infrastructure-as-code/
├── terraform/
│   ├── concepts/       # State, modules, workspaces, providers
│   ├── patterns/       # Module patterns, env separation, testing
│   └── aks-module/     # Sample modular AKS deployment (hands-on)
└── policy-as-code/     # OPA/Conftest, Sentinel examples
```

## Coverage Checklist

### Terraform Core
- [ ] State file: structure, sensitive data, remote state (Azure Blob, S3)
- [ ] State locking: backend locking mechanisms, lock file
- [ ] `terraform refresh`, `plan`, `apply`, `destroy` internals
- [ ] The dependency graph: `terraform graph`, implicit vs explicit dependencies
- [ ] `moved` block, `import` block (Terraform ≥ 1.5)
- [ ] `lifecycle` meta-argument: `create_before_destroy`, `prevent_destroy`, `ignore_changes`
- [ ] Data sources vs resources
- [ ] Terraform providers: version constraints, `required_providers`

### Module Design
- [ ] Module structure: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`
- [ ] Input validation: `validation` block, custom error messages
- [ ] Module versioning: Terraform Registry, git tags
- [ ] Module composition: root module, child modules, module nesting
- [ ] `for_each` vs `count` — when to use each
- [ ] Dynamic blocks: when justified

### Environment Strategy
- [ ] Workspace vs directory-per-environment — trade-offs
- [ ] Terragrunt: DRY config, `terragrunt.hcl`, dependency blocks
- [ ] Variable hierarchy: `.tfvars`, env vars (`TF_VAR_`), defaults

### Testing & Validation
- [ ] `terraform validate` and `terraform fmt`
- [ ] tflint: provider-specific rules, custom rules
- [ ] checkov / tfsec: security scanning, suppression
- [ ] Terratest: Go-based integration tests, idempotency checks
- [ ] infracost: cost estimation in CI

### Drift & Remediation
- [ ] Detecting drift: `terraform plan` in read-only mode, drift reports
- [ ] Reconciling drift: import vs rewrite vs accept
- [ ] Drift at scale: Atlantis, Spacelift, Terraform Cloud/Enterprise

### Policy as Code
- [ ] OPA + Conftest: writing Rego policies for Terraform plans
- [ ] Sentinel (Terraform Enterprise): policy sets, enforcement levels

## Common Interview Questions

- What happens if two engineers run `terraform apply` simultaneously?
- How do you manage secrets in Terraform? (Never store in state — use Key Vault references)
- How do you refactor a Terraform module without destroying resources?
- What's your module versioning strategy?
- How do you handle Terraform at scale with 50 engineers?

## Resources

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terratest](https://terratest.gruntwork.io/)
- [checkov](https://www.checkov.io/)
- [OpenTofu](https://opentofu.org/) — open-source Terraform fork awareness
