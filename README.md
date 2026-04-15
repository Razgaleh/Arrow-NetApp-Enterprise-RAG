# AI Pod Mini Implementation - Ansible Playbook

Step-by-Step Installation Guide, Including Common Mistakes and Issues We Encountered at Arrow During Implementation

## Table of Contents

- [I. Reference Documentation](#i-reference-documentation)
- [II. Software Version Table](#ii-software-version-table)
- [III. Enterprise RAG 2.0 Deployment](#iii-enterprise-rag-20-deployment)
  - [Prerequisites](#prerequisites)
  - [On Ubuntu 22.04](#on-ubuntu-2204)
  - [1. Pull Enterprise RAG 2.0 release from GitHub](#1-pull-enterprise-rag-20-release-from-github)
  - [2. Install prerequisites](#2-install-prerequisites)
  - [3. Create inventory file](#3-create-inventory-file)
  - [4. Set up passwordless SSH to each node](#4-set-up-passwordless-ssh-to-each-node)
  - [5. Verify connectivity](#5-verify-connectivity)
  - [6. Edit config file](#6-edit-config-file)
  - [7. Deploy the K8s cluster (with Trident)](#7-deploy-the-k8s-cluster-with-trident)
  - [8. Change the number of iwatch open descriptors](#8-change-the-number-of-iwatch-open-descriptors)
  - [9. Install kubectl and retrieve kubeconfig](#9-install-kubectl-and-retrieve-kubeconfig)
  - [10. Install MetalLB in your K8s cluster](#10-install-metallb-in-your-k8s-cluster)
  - [11. Configure MetalLB](#11-configure-metallb)
  - [12. Update two config files](#12-update-two-config-files)
    - [File 1: inventory/test-cluster/config.yaml](#file-1-inventorytest-clusterconfigyaml)
    - [File 2: components/edp/values.yaml](#file-2-componentsedpvaluesyaml)
  - [13. Deploy Enterprise RAG](#13-deploy-enterprise-rag)
  - [14. Create a DNS entry for the Enterprise RAG web dashboard](#14-create-a-dns-entry-for-the-enterprise-rag-web-dashboard)
  - [15. Access Enterprise RAG UI](#15-access-enterprise-rag-ui)
- [IV. Common Mistakes, Setup Tips, and Uninstall Instructions](#iv-common-mistakes-setup-tips-and-uninstall-instructions)
  - [Passwordless SSH and Sudo Not Set Up](#passwordless-ssh-and-sudo-not-set-up)
  - [Python Not Installed on All Nodes](#python-not-installed-on-all-nodes)
  - [Missing `--ask-become-pass` Flag](#missing---ask-become-pass-flag)
  - [Setting Up Passwordless SSH and Sudo on Ubuntu](#setting-up-passwordless-ssh-and-sudo-on-ubuntu)
    - [Step 1: Verify Python is installed on all Kubernetes nodes](#step-1-verify-python-is-installed-on-all-kubernetes-nodes)
    - [Step 2: Set up passwordless SSH (Ubuntu)](#step-2-set-up-passwordless-ssh-ubuntu)
    - [Step 3: Set up passwordless sudo (Ubuntu) [Optional but Recommended]](#step-3-set-up-passwordless-sudo-ubuntu-optional-but-recommended)
  - [Uninstall Instructions](#uninstall-instructions)
    - [Uninstall Infrastructure (Kubernetes and Trident)](#uninstall-infrastructure-kubernetes-and-trident)
    - [Uninstall Application (Enterprise RAG)](#uninstall-application-enterprise-rag)

## I. Reference Documentation

**Existing reference design document (in the process of being updated):**  
[https://docs.netapp.com/us-en/netapp-solutions-ai/infra/ai-minipod.html](https://docs.netapp.com/us-en/netapp-solutions-ai/infra/ai-minipod.html)

## II. Software Version Table

| Software | Version | Comment |
|----------|---------|---------|
| OPEA - Intel AI for Enterprise RAG | 2.0 | Enterprise RAG platform based on OPEA microservices |
| Container Storage Interface (CSI driver) | NetApp Trident 25.10 (installed by enterprise RAG infrastructure playbook) | Enables dynamic provisioning, NetApp Snapshot copies, and volumes. |
| Ubuntu | 22.04.5 | OS on two-node cluster |
| Container orchestration | Kubernetes 1.31.9 (installed by Enterprise RAG infrastructure playbook) | Environment to run RAG framework |
| ONTAP | ONTAP 9.16.1P4 or above | Storage OS on AFF A20. |

## III. Enterprise RAG 2.0 Deployment

**Note:** The steps below should replace the existing deployment steps in the ref design doc starting with the "Install Kubernetes" step and ending with the "Access OPEA for Intel AI for Enterprise RAG UI" step (inclusive).

### Prerequisites

Install git, python3.11, pip (for python3.11)

### On Ubuntu 22.04:

```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt upgrade
sudo apt install python3.11
python3.11 --version
```

### 1. Pull Enterprise RAG 2.0 release from GitHub

```bash
git clone https://github.com/opea-project/Enterprise-RAG.git
cd Enterprise-RAG/
git checkout tags/release-2.0.0
```

### 2. Install prerequisites

```bash
cd deployment/
sudo apt-get install python3.11-venv
Python3.11 -m venv erag-venv
source erag-venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
ansible_galaxy collection install -r requirements.yaml --upgrade
```

### 3. Create inventory file

```bash
sudo cp -a inventory/sample inventory/test-cluster
sudo nano inventory/test-cluster/inventory.ini
```

**Sample:**

```ini
# Control plane nodes
kube-3 ansible_host=<control_node_ip_address>

# Worker nodes
kube-1 ansible_host=<worker_node1_ip_address>
kube-2 ansible_host=<worker_node2_ip_address>

# Define node groups
[kube_control_plane]
kube-1
kube-2
kube-3

[kube_node]
kube-1
kube-2

[etcd:children]
kube_control_plane

[k8s_cluster:children]
kube_control_plane
kube_node

# Vars
[k8s_cluster:vars]
ansible_become=true
ansible_user=ailab
ansible_connection=ssh

```

### 4. Set up passwordless SSH to each node

```bash
ssh-copy-id REMOTE_USER@MACHINE_IP
```

### 5. Verify connectivity

```bash
ansible all -i inventory/test-cluster/inventory.ini -m ping
```

**Note:** If you do not have passwordless sudo set up on your nodes, then you will need to add `-K` or `--ask-become-pass` to this command. When using `--ask-become-pass`, it is critical that the ssh user have the SAME password on each node. See section 16 for detailed passwordless setup instructions.

### 6. Edit config file

```bash
sudo nano inventory/test-cluster/config.yaml
```

**Sample:**

```yaml
...
deploy_k8s: true
...
install_csi: "netapp-trident"
...
local_registry: false
...
trident_operator_version: "2510.0"  # Trident operator version (becomes 100.2506.0 in Helm chart)
trident_namespace: "trident"  # Kubernetes namespace for Trident
trident_storage_class: "netapp-trident"  # StorageClass name for Trident
trident_backend_name: "ontap-nas"  # Backend configuration name
...
ontap_management_lif: "<ontap_mgmt_lif>"  # ONTAP management LIF IP address
ontap_data_lif: "<ontap_nfs_data_lif>"  # ONTAP data LIF IP address
ontap_svm: "<ontap_svm>"  # Storage Virtual Machine (SVM) name
ontap_username: "<ontap_username>"  # ONTAP username with admin privileges
ontap_password: "<redacted>"  # ONTAP password
ontap_aggregate: "<ontap_aggr>"  # ONTAP aggregate name for volume creation
...
kubeconfig: "<repository-path>/deployment/inventory/test-cluster/artifacts/admin.conf"
```


### 7. Deploy the K8s cluster (with Trident)

```bash
ansible-playbook playbooks/infrastructure.yaml --tags configure,install -i inventory/test-cluster/inventory.ini -e @inventory/test-cluster/config.yaml
```

**Note:** If you do not have passwordless sudo set up on your nodes, then you will need to add `-K` or `--ask-become-pass` to this command. When using `--ask-become-pass`, it is critical that the ssh user have the SAME password on each node. See section 16 for detailed passwordless setup instructions.

If you get the below error, run `helm repo update` and then re-run the playbook. This is a bug in the 2.0 version - I have submitted a PR to fix it.

```
TASK [netapp_trident_csi_setup : Install Trident operator via Helm]
***************************************************************
Thursday 04 December 2025 20:29:28 -0500 (0:00:00.312)  0:10:36.313 *****
fatal: [localhost]: FAILED! => changed=false
command: /usr/local/bin/helm --version=100.2510.0 show chart 'netapp-trident/trident-operator'
msg: |-
  Failure when executing Helm command. Exited 1.
  stdout:
  stderr: Error: chart "trident-operator" matching 100.2510.0 not found in netapp-trident index. (try 'helm repo update'): no chart version found for trident-operator-100.2510.0
  stderr_lines: <omitted>
  stdout: ''
  stdout_lines: <omitted>
```

### 8. Change the number of iwatch open descriptors

Follow the instructions here: [https://github.com/opea-project/Enterprise-RAG/blob/release-2.0.0/docs/application_deployment_guide.md#change-number-of-iwatch-open-descriptors](https://github.com/opea-project/Enterprise-RAG/blob/release-2.0.0/docs/application_deployment_guide.md#change-number-of-iwatch-open-descriptors)

### 9. Install kubectl and retrieve kubeconfig

Install kubectl if not already installed: [https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

Retrieve your kubeconfig file from:
```
<repository path>/deployment/inventory/test-cluster/artifacts/admin.conf
```

### 10. Install MetalLB in your K8s cluster

If not already installed: [https://metallb.io/installation/](https://metallb.io/installation/)

```bash
helm repo add metallb https://metallb.github.io/metallb
helm -n metallb-system install metallb metallb/metallb --create-namespace
```

### 11. Configure MetalLB

Reference: [https://metallb.io/configuration/#layer-2-configuration](https://metallb.io/configuration/#layer-2-configuration)

**Note:**
- I used Layer 2 mode, and created an IPAddressPool and L2Advertisement as shown in the docs.
- The IP addresses in your address pool can be any unused IPs that are in the same subnet as your K8s nodes. You only need one IP address for ERAG.

```bash
sudo nano metallb-ipaddrpool-l2adv.yaml
```

**Example:**

```yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: erag
  namespace: metallb-system
spec:
  addresses:
  - <ip-subnet>-<ip-subnet>
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: metallb-l2adv
  namespace: metallb-system
```

### 12. Update two config files

#### File 1: inventory/test-cluster/config.yaml

```bash
sudo nano inventory/test-cluster/config.yaml
```

**My lab:**

```yaml
...
FQDN: "aipod-mini-erag.rtp.openenglab.netapp.com" # Provide the FQDN for the deployment
...
gmc:
  ...
  pvc:
    accessMode: ReadWriteMany
...
ingress:
  ...
  service_type: LoadBalancer
edp:
  ...
  storageType: s3compatible
  ...
  s3compatible:
    region: "us-east-1"
    accessKeyId: "<redacted>"
    secretAccessKey: "<redacted>"
    internalUrl: "https://<s3-internal-url>"
    externalUrl: "https://<s3-external-url>"
    bucketNameRegexFilter: "^erag.*"
    edpExternalCertVerify: false
    edpInternalCertVerify: false
...
```
Please note `internalUrl` could be equal to `externalUrl`. 

#### File 2: components/edp/values.yaml

```bash
sudo nano components/edp/values.yaml
```

```yaml
...
presignedUrlCredentialsSystemFallback: "true"
...
celery:
  ...
  config:
    ...
    scheduledSync:
      enabled: true
      syncPeriodSeconds: "60"
...
```

### 13. Deploy Enterprise RAG

```bash
ansible-playbook -u $USER playbooks/application.yaml --tags configure,install -e @inventory/test-cluster/config.yaml
```

**Note:** If you do not have passwordless sudo set up on your deploy node (the laptop or jump host where you are running the ansible-playbook command), then you will need to add `-K` or `--ask-become-pass` to this command. See section 16 for detailed passwordless setup instructions.

### 14. Create a DNS entry for the Enterprise RAG web dashboard

Retrieve external IP address assigned to Enterprise RAG's ingress LoadBalancer:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

Create DNS entry pointing to this IP address for the FQDN that you used in step 12.

### 15. Access Enterprise RAG UI

Navigate to the FQDN (from step 12) in your browser.

**Note:** Retrieve default UI credentials from:

```bash
cat ansible-logs/default_credentials.txt
```

## IV. Common Mistakes, Setup Tips, and Uninstall Instructions


### Passwordless SSH and Sudo Not Set Up

**Problem:** Ansible playbooks fail with permission errors or password prompts timeout.

**Solution:** Setting up passwordless SSH and passwordless sudo significantly simplifies the deployment process and prevents authentication issues. See detailed setup instructions below.

---
### Python Not Installed on All Nodes

**Problem:** Ansible cannot execute tasks on nodes without Python.

**Solution:** Ensure Python 3.11 is installed on all Kubernetes nodes (control plane and workers) before running the playbooks. See setup instructions below.

---

### Missing `--ask-become-pass` Flag

**Problem:** Playbook fails with sudo permission errors.

**Solution:** If you don't have passwordless sudo configured, add the `-K` or `--ask-become-pass` flag to your ansible-playbook commands. The sudo password must be the same on all nodes.

---

### Setting Up Passwordless SSH and Sudo on Ubuntu

#### Step 1: Verify Python is installed on all Kubernetes nodes

Before proceeding, ensure Python 3.11 is installed on **all three Kubernetes nodes** (kube-1, kube-2, and kube-3). SSH into each node and verify:

```bash
ssh REMOTE_USER@kube-1_IP "python3.11 --version"
ssh REMOTE_USER@kube-2_IP "python3.11 --version"
ssh REMOTE_USER@kube-3_IP "python3.11 --version"
```

If Python 3.11 is not installed on any node, install it:

```bash
ssh REMOTE_USER@MACHINE_IP "sudo add-apt-repository ppa:deadsnakes/ppa && sudo apt update && sudo apt install -y python3.11"
```

#### Step 2: Set up passwordless SSH (Ubuntu)

Generate an SSH key on your deploy node (if you haven't already):

```bash
ssh-keygen -t rsa -b 4096
```

Press Enter to accept default location and optionally set a passphrase.

Copy your SSH key to each Kubernetes node:

```bash
ssh-copy-id REMOTE_USER@kube-1_IP
ssh-copy-id REMOTE_USER@kube-2_IP
ssh-copy-id REMOTE_USER@kube-3_IP
```

Verify passwordless SSH works:

```bash
ssh REMOTE_USER@kube-1_IP "hostname"
ssh REMOTE_USER@kube-2_IP "hostname"
ssh REMOTE_USER@kube-3_IP "hostname"
```

#### Step 3: Set up passwordless sudo (Ubuntu) [Optional but Recommended]

On **each Kubernetes node**, add your user to sudoers with NOPASSWD:

```bash
echo "$USER ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/$USER
sudo chmod 0440 /etc/sudoers.d/$USER
```

Or manually edit sudoers:

```bash
sudo visudo -f /etc/sudoers.d/$USER
```

Add this line:

```
REMOTE_USER ALL=(ALL) NOPASSWD:ALL
```

Test passwordless sudo on each node:

```bash
ssh REMOTE_USER@kube-1_IP "sudo whoami"
ssh REMOTE_USER@kube-2_IP "sudo whoami"
ssh REMOTE_USER@kube-3_IP "sudo whoami"
```

This should return `root` without prompting for a password.

---

### Uninstall Instructions

#### Uninstall Infrastructure (Kubernetes and Trident)

To completely remove the Kubernetes cluster and Trident installation:

```bash
ansible-playbook -K playbooks/infrastructure.yaml --tags delete -i inventory/test-cluster/inventory.ini -e @inventory/test-cluster/config.yaml
```

**Note:** Add `-K` or `--ask-become-pass` if you don't have passwordless sudo set up.

#### Uninstall Application (Enterprise RAG)

To remove only the Enterprise RAG application while keeping Kubernetes and Trident:

```bash
ansible-playbook -u $USER playbooks/application.yaml --tags uninstall -e @inventory/test-cluster/config.yaml
```

**Note:** Add `-K` or `--ask-become-pass` if you don't have passwordless sudo set up on your deploy node.


-- how to choose subnet - don't set it to the default

-- NFS storage - keycloak - Storage should have been S3 and NFS - NFS utility needs to be installed on all 3 nodes 

-- setting up the loadbalancer 

-- make sure to set it on the  controller node 

-- core DNS error , debugger tool 