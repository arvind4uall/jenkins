# 🚀 KOPS Setup Guide for Kubernetes on AWS

## 📋 Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step-by-Step Setup](#step-by-step-setup)
  - [1. Launch EC2 Instance](#1-launch-ec2-instance)
  - [2. Configure IAM Role](#2-configure-iam-role)
  - [3. Update Bash Profile](#3-update-bash-profile)
  - [4. Generate SSH Keys](#4-generate-ssh-keys)
  - [5. Install kubectl](#5-install-kubectl)
  - [6. Install KOPS](#6-install-kops)
  - [7. Create S3 Bucket for KOPS State](#7-create-s3-bucket-for-kops-state)
  - [8. Create Kubernetes Cluster](#8-create-kubernetes-cluster)
  - [9. Validate Cluster](#9-validate-cluster)
  - [10. Delete Cluster](#10-delete-cluster)
- [Troubleshooting](#troubleshooting)

---

## 🎯 Overview

**KOPS** (Kubernetes Operations) is a production-grade tool for creating, upgrading, and managing Kubernetes clusters on AWS. This guide walks you through setting up a complete Kubernetes cluster using KOPS on Amazon Web Services.

---

## 📦 Prerequisites

- AWS Account with appropriate permissions
- Basic understanding of Kubernetes concepts
- Familiarity with AWS services (EC2, S3, IAM)

---

## 🛠️ Step-by-Step Setup

### 1. Launch EC2 Instance

**Action:** Launch an Amazon Linux machine that will serve as your KOPS management node.

**Why?** This machine acts as a control plane from which you'll execute all KOPS commands to create and manage your Kubernetes cluster.

---

### 2. Configure IAM Role

**Action:** Create an IAM Role with **AdministratorAccess** policy and attach it to your Amazon Linux EC2 instance.

**Why?** The KOPS tool needs extensive AWS permissions to:
- Create EC2 instances (control plane and worker nodes)
- Set up VPCs, subnets, and security groups
- Configure ELB/ALB for load balancing
- Manage Route53 DNS records
- Create and manage S3 buckets
- Configure IAM roles and policies for cluster components

⚠️ **Security Note:** While AdministratorAccess is convenient for testing, in production environments, follow the principle of least privilege and use a more restrictive IAM policy.

---

### 3. Update Bash Profile

#### Command 1: Add to PATH
```bash
export PATH=$PATH:/usr/local/bin/
```

**Explanation:**
- This command adds `/usr/local/bin/` to your system's PATH environment variable
- The PATH variable tells your shell where to look for executable programs
- KOPS and kubectl binaries will be installed in `/usr/local/bin/`, so this ensures they can be executed from any directory

**How to add:** Open your bash profile with `vi ~/.bashrc` or `nano ~/.bashrc` and add the above line at the end.

#### Command 2: Reload Bash Profile
```bash
source ~/.bashrc
```

**Explanation:**
- The `source` command reloads your bash profile without needing to log out and back in
- This immediately applies the PATH changes you just made
- After running this, your terminal session will recognize commands in `/usr/local/bin/`

---

### 4. Generate SSH Keys

#### Command 1: Generate SSH Key Pair
```bash
ssh-keygen
```

**Explanation:**
- Generates a public/private SSH key pair
- By default, stores keys in `~/.ssh/` directory
- The private key (`id_rsa`) stays on your machine
- The public key (`id_rsa.pub`) will be distributed to cluster nodes for authentication
- When prompted, you can press Enter to accept default location and optionally set a passphrase

#### Command 2: Copy Public Key
```bash
cp /root/.ssh/id_rsa.pub my-keypair.pub
```

**Explanation:**
- Copies the public SSH key to a file named `my-keypair.pub` in the current directory
- This file will be used by KOPS to configure SSH access to cluster nodes
- The public key will be added to the `authorized_keys` file on all cluster instances

#### Command 3: Change Permissions
```bash
chmod 777 my-keypair.pub
```

**Explanation:**
- `chmod` changes file permissions
- `777` gives read, write, and execute permissions to everyone (owner, group, others)
- **Breakdown:** `7 = 4(read) + 2(write) + 1(execute)`

⚠️ **Security Warning:** Using `777` is overly permissive and not recommended for security-sensitive files. A better permission would be `644` (read/write for owner, read-only for others):
```bash
chmod 644 my-keypair.pub  # Recommended alternative
```

---

### 5. Install kubectl

#### Command 1: Download kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

**Explanation:**
- `curl -LO`: Downloads a file while following redirects (`-L`) and saves it with the original name (`-O`)
- **Inner command:** `$(curl -L -s https://dl.k8s.io/release/stable.txt)`
  - Fetches the latest stable Kubernetes version number
  - `-s` runs in silent mode (no progress bar)
  - This is command substitution - the output becomes part of the URL
- **Outer command:** Downloads the kubectl binary for Linux (amd64 architecture)
- Downloads to current directory with filename `kubectl`

**What is kubectl?** The command-line tool for interacting with Kubernetes clusters. You'll use it to deploy apps, inspect resources, and manage cluster operations.

#### Command 2: Make kubectl Executable
```bash
chmod +x kubectl
```

**Explanation:**
- Adds execute permission to the kubectl binary
- `+x` adds execute permission for all users
- Without this, you couldn't run kubectl as a program

#### Command 3: Move kubectl to System Path
```bash
mv kubectl /usr/local/bin/kubectl
```

**Explanation:**
- Moves the kubectl binary to `/usr/local/bin/`
- This directory is in your system PATH (from step 3)
- Now you can run `kubectl` from any directory without specifying the full path

---

### 6. Install KOPS

#### Command 1: Download KOPS
```bash
wget https://github.com/kubernetes/kops/releases/download/v1.32.0/kops-linux-amd64
```

**Explanation:**
- `wget`: Downloads files from the web
- Downloads KOPS version 1.32.0 for Linux (amd64 architecture)
- Retrieves from the official Kubernetes KOPS GitHub releases page

**Note:** Check for the latest version at https://github.com/kubernetes/kops/releases

#### Command 2: Make KOPS Executable
```bash
chmod +x kops-linux-amd64
```

**Explanation:**
- Same as with kubectl - adds execute permission to the KOPS binary

#### Command 3: Move KOPS to System Path
```bash
mv kops-linux-amd64 /usr/local/bin/kops
```

**Explanation:**
- Moves the KOPS binary to `/usr/local/bin/` and renames it to `kops`
- Makes the command available system-wide as `kops`

---

### 7. Create S3 Bucket for KOPS State

#### Command 1: Create S3 Bucket
```bash
aws s3api create-bucket --bucket fsdarvind.k8s.local --region ap-south-1 --create-bucket-configuration LocationConstraint=ap-south-1
```

**Explanation:**
- `aws s3api create-bucket`: AWS CLI command to create an S3 bucket
- `--bucket fsdarvind.k8s.local`: Name of the bucket (must be globally unique across all AWS accounts)
- `--region ap-south-1`: AWS region (Mumbai in this case)
- `--create-bucket-configuration LocationConstraint=ap-south-1`: Required for regions outside us-east-1; ensures bucket is created in the specified region

**Why S3 Bucket?** KOPS stores cluster state and configuration in S3, including:
- Cluster specification
- Instance group configurations
- SSH keys
- Cluster secrets
- Enables cluster state sharing across team members

#### Command 2: Enable Bucket Versioning
```bash
aws s3api put-bucket-versioning --bucket fsdarvind.k8s.local --region ap-south-1 --versioning-configuration Status=Enabled
```

**Explanation:**
- `put-bucket-versioning`: Enables S3 versioning on the bucket
- `--versioning-configuration Status=Enabled`: Activates versioning

**Why Versioning?**
- Keeps history of all state changes
- Allows rollback to previous cluster configurations if something goes wrong
- Protects against accidental deletions or overwrites
- Essential for production environments

#### Command 3: Export KOPS State Store
```bash
export KOPS_STATE_STORE=s3://fsdarvind.k8s.local
```

**Explanation:**
- Sets an environment variable that tells KOPS where to store/retrieve cluster state
- `s3://fsdarvind.k8s.local`: S3 bucket URI
- Must be set before running any KOPS commands
- Add this to your `~/.bashrc` to make it persistent across sessions

---

### 8. Create Kubernetes Cluster

#### Command: Create Cluster Configuration
```bash
kops create cluster \
  --name=fsdarvind.k8s.local \
  --zones=ap-south-1a,ap-south-1b \
  --control-plane-count=1 \
  --control-plane-size=t3.medium \
  --node-count=2 \
  --node-size=t3.small \
  --node-volume-size=20 \
  --control-plane-volume-size=20 \
  --ssh-public-key=my-keypair.pub \
  --image=ami-02d26659fd82cf299 \
  --networking=calico \
  --topology=public
```

**Parameter Breakdown:**

| Parameter | Value | Explanation |
|-----------|-------|-------------|
| `--name` | `fsdarvind.k8s.local` | Cluster name. Using `.k8s.local` creates a gossip-based cluster (no Route53 required) |
| `--zones` | `ap-south-1a,ap-south-1b` | AWS availability zones for high availability. Distributes nodes across multiple zones |
| `--control-plane-count` | `1` | Number of master nodes. Use 3 or 5 for production (HA) |
| `--control-plane-size` | `t3.medium` | EC2 instance type for control plane nodes (2 vCPU, 4GB RAM) |
| `--node-count` | `2` | Number of worker nodes where your applications run |
| `--node-size` | `t3.small` | EC2 instance type for worker nodes (2 vCPU, 2GB RAM) |
| `--node-volume-size` | `20` | EBS volume size (GB) for worker nodes |
| `--control-plane-volume-size` | `20` | EBS volume size (GB) for control plane nodes |
| `--ssh-public-key` | `my-keypair.pub` | Path to SSH public key for node access |
| `--image` | `ami-02d26659fd82cf299` | AMI ID for Ubuntu/Debian (region-specific) |
| `--networking` | `calico` | CNI plugin. Options: calico, flannel, weave, amazon-vpc-routed-eni |
| `--topology` | `public` | Cluster topology. `public` = internet-facing, `private` = internal only |

**Note:** This command only creates the configuration. It doesn't actually provision resources yet.

#### Command: Apply Cluster Configuration
```bash
kops update cluster --name fsdarvind.k8s.local --yes --admin
```

**Explanation:**
- `update cluster`: Applies the cluster configuration and creates AWS resources
- `--name fsdarvind.k8s.local`: Specifies which cluster to update
- `--yes`: Auto-confirms the action (skips confirmation prompt)
- `--admin`: Generates admin credentials with cluster-admin privileges

**What happens:**
- Creates VPC, subnets, internet gateway, route tables
- Launches EC2 instances (1 control plane + 2 worker nodes)
- Configures security groups
- Sets up load balancers
- Installs Kubernetes components
- This process takes 5-15 minutes

---

### 9. Validate Cluster

#### Command: Validate Cluster
```bash
kops validate cluster --wait 10m
```

**Explanation:**
- `validate cluster`: Checks if all cluster components are running correctly
- `--wait 10m`: Waits up to 10 minutes for cluster to become ready
- Polls every few seconds until validation succeeds or timeout occurs

**What it validates:**
- All nodes are registered and ready
- Control plane components (API server, scheduler, controller manager) are healthy
- DNS is working
- Node-to-node networking is functional
- All system pods are running

**Output when successful:**
```
Your cluster fsdarvind.k8s.local is ready
```

After validation, you can use kubectl to interact with your cluster:
```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

---

### 10. Delete Cluster

#### Command: Delete Cluster
```bash
kops delete cluster --name fsdarvind.k8s.local --yes
```

**Explanation:**
- `delete cluster`: Destroys all AWS resources created by KOPS
- `--name fsdarvind.k8s.local`: Cluster name to delete
- `--yes`: Auto-confirms deletion (dangerous!)

**What gets deleted:**
- All EC2 instances (control plane and worker nodes)
- Load balancers
- Security groups
- VPC and networking components
- EBS volumes
- IAM roles and policies created by KOPS

**⚠️ WARNING:** This action is **irreversible**. All data and applications in the cluster will be permanently lost.

**Best practice:** Run without `--yes` first to see what will be deleted:
```bash
kops delete cluster --name fsdarvind.k8s.local
```

**Note:** The S3 state bucket is NOT deleted. You must delete it manually if needed:
```bash
aws s3 rb s3://fsdarvind.k8s.local --force
```

---

## 🐛 Troubleshooting

### Common Issues

#### 1. **Cluster validation fails**
- **Solution:** Wait longer (clusters can take 10-15 minutes)
- Check AWS console for EC2 instance states
- Verify IAM role has sufficient permissions

#### 2. **kubectl commands don't work**
- **Solution:** Ensure `kops export kubecfg --name <cluster-name>` is run
- Check if `~/.kube/config` exists and has correct cluster info

#### 3. **S3 bucket creation fails**
- **Solution:** Bucket names must be globally unique - try a different name
- Ensure you have S3 permissions in your IAM role

#### 4. **SSH access to nodes fails**
- **Solution:** Verify security group allows SSH (port 22)
- Ensure you're using the correct private key

#### 5. **Cost concerns**
- Monitor your AWS costs - running clusters can be expensive
- Always delete clusters when not in use
- Consider using spot instances for cost savings

---
