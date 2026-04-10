# Security & DevSecOps

> ⭐ Priority area — product companies increasingly test security depth at senior level

## Folder Structure

```
08-security-devsecops/
├── supply-chain/       # SBOM, Cosign, Sigstore, SLSA
├── secrets-management/ # Vault, ESO, SOPS, sealed secrets
├── runtime-security/   # Falco, seccomp, AppArmor, PSS
├── compliance/         # SOC2, ISO27001, OWASP, audit logging
└── threat-models/      # STRIDE threat models for infra
```

## Coverage Checklist

### Supply Chain Security
- [ ] SLSA framework: levels 1–4, provenance, build integrity
- [ ] SBOM: CycloneDX vs SPDX formats, generation (Syft, Anchore)
- [ ] Sigstore: Cosign image signing, Rekor transparency log, Fulcio CA
- [ ] Container image scanning: Trivy (vuln DB, misconfig, SBOM), Grype, Snyk
- [ ] Base image hardening: distroless, scratch, non-root user, read-only rootfs
- [ ] Software composition analysis (SCA): dependency vulnerability tracking

### Secrets Management
- [ ] HashiCorp Vault: auth methods (K8s, AppRole, AWS IAM), secrets engines
- [ ] Vault dynamic secrets: database, PKI, cloud credentials
- [ ] External Secrets Operator (ESO): ExternalSecret, ClusterExternalSecret, SecretStore
- [ ] SOPS: age encryption, KMS, `.sops.yaml` config, git integration
- [ ] Sealed Secrets (Bitnami): asymmetric encryption, cluster-scoped
- [ ] Anti-patterns: secrets in Docker ENV, in Git, in K8s Secret as base64 (not encrypted at rest)
- [ ] Kubernetes secret encryption at rest: EncryptionConfiguration, KMS plugin

### Runtime Security
- [ ] Falco: rules engine, syscall monitoring, K8s audit integration
- [ ] seccomp profiles: RuntimeDefault, custom profiles, SeccompProfile resource
- [ ] AppArmor: profiles, K8s annotation vs label (K8s 1.30+)
- [ ] Pod Security Standards (PSS): Privileged / Baseline / Restricted
- [ ] Pod Security Admission (PSA): enforce / audit / warn modes
- [ ] Rootless containers, non-root user enforcement
- [ ] Capabilities: drop ALL, add only what's needed

### Network Security
- [ ] Kubernetes NetworkPolicy: default-deny baseline, ingress/egress rules
- [ ] Cilium Network Policy: Cilium-specific resources, DNS-based policies
- [ ] Zero Trust: never trust always verify, microsegmentation
- [ ] mTLS everywhere: SPIFFE/SPIRE identity, Istio PeerAuthentication

### Compliance & Governance
- [ ] SOC2 Type II: security, availability, confidentiality controls in DevOps context
- [ ] ISO 27001: control mapping to infrastructure controls
- [ ] OWASP Top 10: mapped to CI/CD pipeline controls
- [ ] CIS Benchmarks: for K8s, Docker, Linux — tools (kube-bench, docker-bench)
- [ ] Audit logging design: K8s audit policy, centralised log aggregation
- [ ] Access reviews: just-in-time access, break-glass procedures

### IAM & RBAC
- [ ] Principle of least privilege
- [ ] Kubernetes RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding
- [ ] ServiceAccount best practices: automount: false, bound tokens
- [ ] Workload Identity (Azure/GCP/AWS): OIDC federation, no long-lived keys
- [ ] Azure RBAC: built-in vs custom roles, PIM for privileged access

## Threat Modeling

Use STRIDE for infra threat modeling:
- **S**poofing → Identity verification, mTLS, OIDC
- **T**ampering → Code signing, IaC drift detection, RBAC
- **R**epudiation → Audit logging, immutable logs  
- **I**nformation Disclosure → Secrets management, network policies, encryption at rest
- **D**enial of Service → Rate limiting, resource quotas, autoscaling
- **E**levation of Privilege → PSS, non-root, capability dropping, RBAC audit

## Resources

- [SLSA Framework](https://slsa.dev/)
- [Sigstore / Cosign](https://docs.sigstore.dev/)
- [HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [Falco Documentation](https://falco.org/docs/)
- [OWASP DevSecOps Guideline](https://owasp.org/www-project-devsecops-guideline/)
- [kube-bench](https://github.com/aquasecurity/kube-bench)
