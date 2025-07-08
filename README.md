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

## Network Policies

- Firewall Rules in Kubernetes
- Implemented by the **Network Plugins CNI** (Calico / Weave)
- Namespace level
- Restrict the ingress and/or Egress for a group of Pods based on certain rules and conditions

### Without NetworkPolicies

- By default every pod can access every pod
- Pods are NOT isolated

### Example Visualization of NetworkPolicies

![Egress NetworkPolicies](./diagram/network-policies_egress.png)
![Ingress NetworkPolicies](./diagram/network-policies_ingress.png)
![Namespace NetworkPolicies](./diagram/network-policies_namespace.png)
![IpBlock NetworkPolicies](./diagram/network-policies_ipBlock.png)

### Example of `yaml` Declarative Configurations of NetworkPolicy

```yaml
kind: NetworkPolicy
metadata:
  name: example
  namespace: default
spec:
  podSelector:
    matchLabels:
      id: frontend
  policyTypes:
  - Egress
```

The above example is a valid Network policy which:

- **denies** all outgoing traffic
- from pods with label id=frontend
- in namespace default

```yaml
kind: NetworkPolicy
metadata:
  name: example
  namespace: default
spec:
  podSelector:
    matchLabels:
      id: frontend  # will be applied to these pods
  policyTypes:
  - Egress  # will be about outgoing traffic
  egress:
  - to:
  # allow outgoing traffic to namespace with label id=ns1 AND port 80
    - namespaceSelector:
        matchLabels:
          id: ns1
    ports:
    - protocol: TCP
      port: 80
  - to:
  # allow outgoing traffic to pods with label id=backend in same namespace (default)
    - podSelector:
        matchLabels:
          id: backend
```

In the above example two egress rules are connected with "OR".

### Multiple NetworkPolicies

- Possible to have multiple NPs selecting the same pods
- If a pod has more than one NP
  - then the union of all NPs is applied
  - order doesn't affect policy result

You can check out the [`example.yaml`](./NetworkPolicy/merged-policy/example.yaml) which is being a merged-policy of [`example2a.yaml`](./NetworkPolicy/merged-policy/example2a.yaml) and [`example2b.yaml`](./NetworkPolicy/merged-policy/example2b.yaml)

### Default Deny NetworkPolicy

We'll create a very simple scenario with one frontend pod and one backend pod, and we'll check the connectivity between each pod before and after of creating our NetworkPolicy.

```bash
kubectl run frontend --image=nginx:alpine
kubectl run backend --image=nginx:alpine

kubectl expose pod frontend --port 80
kubectl expose pod backend --port 80

kubectl get pod,svc

# Now check connectivity from frontend to backend, & vice versa
kubectl exec frontend -- curl backend
kubectl exec backend -- curl frontend

# Now we'll create our network policy
vim default-deny.yaml
kubectl -f default-deny.yaml create

kubectl exec frontend -- curl backend
kubectl exec backend -- curl frontend
```

Use the example [`default-deny.yaml`](./NetworkPolicy/default-deny/default-deny.yaml) for practice.

### Allow frontend pods to talk to backend pods

-- based on `podSelectors`

We will specifically allow frontend pod to connect to backend pod e.g., we'll create one network policy to allow outgoing traffic from frontend and one incoming traffic from frontend to backend.

```bash
vim frontend.yaml
kubectl -f frontend.yaml create

kubectl exec frontend -- curl backend

vim backend.yaml
kubectl -f backend.yaml create

kubectl exec frontend -- curl backend
# It will still not work
```

> [!Important]
> Our **default-deny policy** even denies default DNS traffic (port 53), as we need DNS resolution for frontend to connect with backend.

```bash
kubectl get pod --show-labels -owide
kubectl exec frontend -- curl <backend-IP>
```

> [!Note]
> If you would like to allow DNS resolution, you have to extend your default-deny policy where you would allow Egress to the port 53.
>
> You can check out [`allow-dns-resolution.yaml`](./NetworkPolicy/default-deny/allow-dns-resolution.yaml)

### Allow backend pods to talk to database pods

-- based on `namespaceSelectors`

#### pod-frontend → pod-backend → pod-cassandra

```bash
kubectl create ns cassandra

# Add "ns: cassandra" labels
kubectl edit ns cassandra
# OR
kubectl label namespace cassandra ns=cassandra

kubectl -n cassandra run cassandra --image=nginx:alpine
kubectl -n cassandra get pod -owide

kubectl exec backend -- curl <cassandra-IP>
```

Now we'll allow backend pods to have egress traffic the namespace `cassandra`, where our database pod is running.

```bash
vim backend.yaml # edit to add Egress policy
kubectl -f backend.yaml apply

vim cassandra-deny.yaml
kubectl -f backend.yaml create

vim cassandra.yaml
kubectl -f cassandra.yaml create

kubectl exec backend -- curl <cassandra-IP>
# FAILS! Try to debug yourself :)
```

> [!Note]
> In previous step, we didn't modified `default` namespace to add labels "ns: default"

```bash
kubectl edit ns default

# Now try
kubectl exec backend -- curl <cassandra-IP>

# You can also allow DNS traffic in cassandra namespace
vim cassandra-deny.yaml
kubectl -f cassandra-deny.yaml apply # Add egress to port 53 (TCP & UDP)

# Expose cassandra as a service
kubectl -n cassandra expose pod cassandra --port 80
kubectl exec backend -- curl cassandra.cassandra
```

### Extend restriction between backend & cassandra

- based on additional pod label
- and additional port

You can check extended [`cassandra.yaml`](./NetworkPolicy/extended-restrictions/cassandra.yaml)

---

## GUI Elements and the Dashboard

- only expose sevices externally if needed
- cluster internal services/dashboards can also be accessed using kubectl port-forward

### Tesla Hack 2018

- Kubernetes Dashboard **had too many privileges** on the cluster
  - without RBAC or too broad roles
- Kubernetes Dashboard was **exposed to the internet**
  - which is isn't by default

### Kubectl proxy

- Creates a proxy server between localhost and the Kubernetes API Server
- uses connection as configured in the kubeconfig
- allows to access API locally just over http and without authentication

![kubectl-proxy](./diagram/kubectl-proxy.png)

### Kubectl port-forward

- forwards connections from a localhost-port to a pod-port
- more generic than kubectl proxy
- can be used for all TCP traffic not just HTTP

![kubectl port-forward](./diagram/kubectl_port-forward.png)

> [!Note]
> If you have dashboard and you want to expose it externally without using `kubectl`, you should have some precautions. You could do it for example: with an Ingress (Nginx Ingress) and URL of your custom domain. Then you have to implement some authentication.

### Install & Access the Kubernetes Dashboard

Refer to the Official [Kubernetes Dashboard GitHub Repo](https://github.com/kubernetes/dashboard)

> [!Important]
> You have to install Helm first to install latest kubernetes-dashboard (v7+). Refer to Official [Helm Docs for installation](https://helm.sh/docs/intro/install/).

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard

kubectl get ns
kubectl -n kubernetes-dashboard get pod,svc
```

### Make Dashboard Available Externally on HTTP

> [!Important]
> Don't do this in production!

Check out [Dashboard Arguments docs](https://github.com/kubernetes/dashboard/blob/master/docs/common/arguments.md) in official [kubernetes/dashboard repo](https://github.com/kubernetes/dashboard)

```bash
kubectl -n kubernetes-dashboard get pod,deploy,svc

kubectl -n kubernetes-dashboard edit deploy kubernetes-dashboard-api
# add --insecure-port=8000 to `spec.template.spec.containers.args`

kubectl -n kubernetes-dashboard edit svc kubernetes-dashboard-web
# Change type to "NodePort"

kubectl -n kubernetes-dashboard get svc
```

> Try to access the Kubernetes Dashboard on: `http://<worker-node_External-IP>:<web-NodePort>`

It will ask for Bearer Token, most probably it won't work like this!

---

## Author

- [Soumo Sarkar](https://www.linkedin.com/in/soumo-sarkar/)

## Reference

- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Kubernetes CKS Full Course by @KillerShell](https://www.youtube.com/watch?v=d9xfB5qaOfg)
