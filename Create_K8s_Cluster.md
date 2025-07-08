# Create your K8s Cluster

## Cluster Specs

Setup Two Virtual Machines (one as **MASTER** and another as **WORKER**):

- **Name as:** *cks-master* & *cks-worker*
- **OS:** Ubuntu 24.04 LTS
- **Disk:** 50GB
- **CPU:** 2
- **RAM:** 4GB

Then run **install_master.sh** on master and **install_worker.sh** on the worker node.

> [!Important]
> To follow along Create [GCP Account](https://console.cloud.google.com/) and activate free trial to get $300 free credit.

## Install and configure `gcloud` command

Follow the steps from [Google Cloud SDK Installation Docs](https://cloud.google.com/sdk/docs/install)

### `gcloud` Setup commands

```bash
gcloud auth login
gcloud projects list
gcloud config set project <PROJECT_ID>
```

## Create the CKS Course K8s Cluster in GCP

### VM specifications (create 2 VM)

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

### SSH into instances locally via `gcloud`

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
