# Kubernetes Security Specialist (CKS)

## K8s Security Best Practices

- Security combines many different complex processes
- Environments change e.g., security cannot stay in a certain state
- Attackers have advantage
  - They decide time
  - They pick what to attack like weakest link

### Security Principles

1. Defense in Depth
2. Least Privilege
3. Limiting the Attack Surface

> [!Note]
> **Redundancy** is good... in security. In case of application development, follow **DRY** ("Don't repeat yourself"), but in security consider Layered defense and Redundancy.

### Kubernetes Security Categories

1. Host Operating System Security 🐧
2. Kubernetes Cluster Security ☸️
    - It's already done in Managed K8s Service e.g., by AWS or Google
3. Application Security 🐳

#### Host OS Security

- Kubernetes Nodes should only do one thing Kubernetes
- Reduce Attack Surface
  - Remove unncessary applications
  - Keep up to date
- Runtime Security Tools
- Find and identify malicious processes
- Restrict IAM / SSH access

#### Kubernetes Cluster Security

- Kubernetes components are running secure and up-to date:
  - API Server
  - kubelet
  - ETCD
- Restrict (external) access
- Use Authentication → Authorization
- AdmissionControllers
  - NodeRestriction
  - Custom Policies (OPA)
- Enable Audit Logging
- Security Benchmarking
- Encrypt Traffic to ETCD

#### Application Security

- Use Secrets / no hardcoded credentials
- RBAC
- Container Sandboxing
- Container Hardening
  - Attack Surface
  - Run as user
  - Readonly filesystem
- Vulnerability Scanning
- mTLS / ServiceMeshes
