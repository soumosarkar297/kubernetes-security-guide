# Kubernetes Security Specialist (CKS) – Study Guide

A comprehensive study guide and practical resource for the **Certified Kubernetes Security Specialist (CKS)** exam. This repository covers essential K8s security concepts, best practices, and hands-on labs to help you master Kubernetes security fundamentals and pass the CKS certification.

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

---

## Create your K8s Cluster

### Cluster Specs

Setup Two Virtual Machines (one as **MASTER** and another as **WORKER**):

- **Name as:** *cks-master* & *cks-worker*
- **OS:** Ubuntu 24.04 LTS
- **Disk:** 50GB
- **CPU:** 2
- **RAM:** 4GB

Then run **install_master.sh** on master and **install_worker.sh** on the worker node.

> [!Important]
> To follow along Create [GCP Account](https://console.cloud.google.com/) and activate free trial to get $300 free credit.

### Install and configure `gcloud` command

Follow the steps from [Google Cloud SDK Installation Docs](https://cloud.google.com/sdk/docs/install)

#### `gcloud` Setup commands

```bash
gcloud auth login
gcloud projects list
gcloud config set project <PROJECT_ID>
```

### Create the CKS Course K8s Cluster in GCP

#### VM specifications (create 2 VM)

- **Name:** cks-master | cks-worker
- **Region:** any near you
- **Machine:** e2-medium (2 vCPU, 1 core, 4 GB memory)
- **OS and Storage:**
  - **Operating system:** Ubuntu
  - **Version:** Ubuntu 24.04 LTS (x86/64, amd64 noble image built on 2025-07-03)
  - **Boot disk type:** Standard persistent disk
  - **Size:** 50 GB

> [!Note]
> Firstly, you need to enable **Compute Engine API** in Google Cloud Account.

#### SSH into instances locally via `gcloud`

```bash
gcloud compute instances list
gcloud compute ssh cks-master
```

Into your `cks-master` node:

```bash
sudo -i
bash <(curl -s https://raw.githubusercontent.com/soumosarkar297/kubernetes-security-guide/main/cluster-setup/scripts/install_master.sh)
```

Similar way, SSH into `cks-worker`:

```bash
sudo -i
bash <(curl -s https://raw.githubusercontent.com/soumosarkar297/kubernetes-security-guide/main/cluster-setup/scripts/install_worker.sh)

# run the printed kubeadm-join-command from the master on the worker
kubeadm join <cks-master_Internal-IP>:6443 --token xxxx.xxxx --discovery-token-ca-cert-hash sha256:xxxx
```

> [!Note]
> Instead of Docker, we installed **containerd** as container runtime which implements the **CRI** which is a requirement to work with Kubernetes.
>
> Use `crictl ps` for containerd

### Make Cluster accessible from outside

From your local terminal, run the following:

```bash
gcloud compute firewall-rules create nodeports --allow tcp:30000-40000
```

We will make use of it later.

> [!Important]
> Stop your instances when you are not doing practical. To save your free credits.
>
> Also you can Reset your cluster after every section, as each new section works with a fresh cluster environment.

---

## Container Isolation in action

> [!Note]
> You must be familiar with Kubernetes architecture and basic components. For basic introduction, you can should check out [Foundation.md](./Foundation.md)

### Create two containers and check if they can see each other

```bash
podman run --name c1 -d ubuntu sh -c 'sleep 1d'
podman exec c1 ps aux

podman run --name c2 -d ubuntu sh -c 'sleep 999d'
podman exec c2 ps aux

ps aux | grep sleep
```

Then create two container in the same namespace

```bash
podman rm c2 --force

# Run the c2 in the PID namespace as c1
podman run --name c2 --pid=container:c1 -d ubuntu sh -c 'sleep 999d'

podman exec c2 ps aux
podman exec c1 ps aux
```

---
