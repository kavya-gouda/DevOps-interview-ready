# CI/CD Pipelines & GitOps

## Folder Structure

```
05-cicd-pipelines/
├── github-actions/     # Reusable workflows, OIDC, composite actions
├── gitlab-ci/          # Templates, DAG, runners at scale
├── jenkins/            # Shared libraries, Jenkinsfile patterns
├── gitops/             # ArgoCD, Flux, App of Apps
└── release-strategies/ # Blue/green, canary, feature flags
```

## Coverage Checklist

### GitHub Actions
- [ ] Workflow triggers: `push`, `pull_request`, `workflow_dispatch`, `schedule`, `workflow_call`
- [ ] Reusable workflows: `workflow_call`, input/output/secrets passing
- [ ] Composite actions: steps without a runner, calling other actions
- [ ] OIDC federation: keyless auth to Azure/AWS/GCP — no long-lived secrets
- [ ] Environments + required reviewers: deployment protection rules
- [ ] Self-hosted runners: scaling with Actions Runner Controller (ARC) on K8s
- [ ] Caching: `actions/cache`, Docker layer caching
- [ ] Matrix strategy: fan-out testing
- [ ] Concurrency groups: prevent overlapping deployments

### GitLab CI
- [ ] `include`: templates, remote includes, `rules` vs `only/except`
- [ ] DAG pipelines: `needs`, `dependencies`
- [ ] Review Apps: dynamic environments per branch
- [ ] GitLab Runners: executor types, scaling with K8s executor
- [ ] Component catalog (GitLab 17+): reusable pipeline components
- [ ] Release management: semantic release, changelog automation

### Jenkins
- [ ] Declarative vs Scripted Pipelines
- [ ] Shared Libraries: `vars/`, `src/`, versioning strategy
- [ ] Multi-branch pipeline: branch source, webhook triggers
- [ ] Agent provisioning: Kubernetes plugin (dynamic agents)
- [ ] Blue Ocean UI vs Pipeline as Code

### GitOps (ArgoCD / Flux)
- [ ] ArgoCD: App of Apps pattern, ApplicationSet, sync waves, weights
- [ ] ArgoCD: multi-cluster management, cluster bootstrap
- [ ] Flux: Kustomization, HelmRelease, ImageUpdateAutomation
- [ ] GitOps promotion: environment branches vs path-based
- [ ] Rollback strategies: Git revert vs ArgoCD rollback
- [ ] Drift detection: ArgoCD self-heal, OutOfSync alerts

### Release Strategies
- [ ] Blue/Green: traffic switch mechanism, DB schema compatibility
- [ ] Canary: Argo Rollouts, weight-based traffic, analysis templates
- [ ] Feature flags: LaunchDarkly, flagd, OpenFeature standard
- [ ] Shadow deployment: traffic mirroring for testing
- [ ] Database migrations in CD: expand/contract pattern

### Supply Chain Security in CI/CD
- [ ] SAST: Semgrep, Bandit, CodeQL
- [ ] DAST: OWASP ZAP in pipeline
- [ ] SCA: Snyk, OWASP Dependency-Check, Dependabot
- [ ] Secret scanning: GitHub Advanced Security, TruffleHog, gitleaks
- [ ] SBOM generation: Syft, Anchore
- [ ] Image signing: Cosign + Sigstore in CI

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Flux Documentation](https://fluxcd.io/docs/)
- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)
