# 🚀 K3s Installation Guide for Kubernetes

## 📋 Overview

K3s is a lightweight, certified Kubernetes distribution designed for production workloads in resource-constrained environments. This guide walks you through the complete installation and configuration process for K3s on a Linux system.

---

## 📦 Prerequisites

- Linux operating system (Ubuntu, CentOS, RHEL, etc.)
- Root or sudo access
- Minimum 512MB RAM (1GB+ recommended)
- Internet connectivity

---

## 🛠️ Installation Steps

### Step 1: Install kubectl

**kubectl** is the command-line tool for interacting with Kubernetes clusters. Let's download the latest stable version.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

**What this does:** This command downloads the latest stable kubectl binary for Linux AMD64 architecture. The inner `curl` command fetches the current stable version number, which is then used to download the appropriate binary.

---

### Step 2: Make kubectl Executable

```bash
chmod +x kubectl
```

**What this does:** This command modifies the file permissions to make the kubectl binary executable, allowing you to run it as a command.

---

### Step 3: Move kubectl to System Path

```bash
sudo mv kubectl /usr/local/bin/kubectl
```

**What this does:** This moves the kubectl binary to `/usr/local/bin/`, which is in your system's PATH. This allows you to execute `kubectl` commands from any directory without specifying the full path to the binary.

---

### Step 4: Verify kubectl Installation

```bash
kubectl version --client
```

**What this does:** Confirms that kubectl is properly installed and accessible from your PATH.

---

### Step 5: Install K3s

```bash
curl -sfL https://get.k3s.io | sh -
```

**What this does:** This command downloads and executes the K3s installation script. K3s will be installed as a systemd service and will start automatically. The installation includes:
- K3s server (control plane)
- kubectl bundled with K3s
- Container runtime (containerd)
- All necessary Kubernetes components

**Flags explained:**
- `-s`: Silent mode (only shows errors)
- `-f`: Fail silently on server errors
- `-L`: Follow redirects

---

### Step 6: Check K3s Service Status

```bash
systemctl status k3s
```

**What this does:** Displays the current status of the K3s systemd service, showing whether it's active, running, and any recent log entries. You should see `active (running)` in green if everything is working correctly.

**Additional useful commands:**
```bash
# Stop K3s service
sudo systemctl stop k3s

# Start K3s service
sudo systemctl start k3s

# Restart K3s service
sudo systemctl restart k3s

# Enable K3s to start on boot (enabled by default)
sudo systemctl enable k3s
```

---

## ⚙️ Configuration

### Understanding K3s Configuration

K3s stores all cluster configuration and credentials in a kubeconfig file located at:

```
/etc/rancher/k3s/k3s.yaml
```

**What this file contains:**
- Cluster API server endpoint
- Cluster certificate authority
- Admin user credentials
- Context configuration

By default, this file requires root permissions to access, which means you'd need to run kubectl commands as root or specify the kubeconfig path each time.

---

### Option 1: Specify kubeconfig for Each Command (Not Recommended)

```bash
kubectl get pods --kubeconfig=/etc/rancher/k3s/k3s.yaml
```

**Why this is tedious:** You have to include `--kubeconfig=/etc/rancher/k3s/k3s.yaml` with every kubectl command, which is cumbersome and error-prone.

---

### Option 2: Configure Persistent Access (Recommended)

#### Step 1: Grant Read Permissions

```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

**What this does:** Modifies the file permissions to allow read access for all users while keeping write access restricted to root. This is more secure than `chmod +777`, which grants full permissions to everyone.

**Permission breakdown:**
- `6` (owner): read + write
- `4` (group): read only
- `4` (others): read only

> ⚠️ **Security Note:** If you're on a shared system, consider copying the file to your home directory instead:
> ```bash
> mkdir -p ~/.kube
> sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
> sudo chown $USER:$USER ~/.kube/config
> chmod 600 ~/.kube/config
> ```

---

#### Step 2: Set KUBECONFIG Environment Variable

Add the following line to your shell profile file:

**For bash (`~/.bashrc` or `~/.bash_profile`):**
```bash
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
```

**For zsh (`~/.zshrc`):**
```bash
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.zshrc
```

**What this does:** Sets the `KUBECONFIG` environment variable, which tells kubectl where to find the cluster configuration file. This eliminates the need to specify `--kubeconfig` with every command.

---

#### Step 3: Reload Shell Configuration

```bash
source ~/.bashrc
```

**For zsh users:**
```bash
source ~/.zshrc
```

**What this does:** Reloads your shell profile to apply the changes immediately without requiring a logout or terminal restart.

---

## ✅ Verification

### Test Your Setup

Now you can run kubectl commands without any additional flags:

```bash
# View all pods across all namespaces
kubectl get pods --all-namespaces

# View all nodes in the cluster
kubectl get nodes

# View cluster information
kubectl cluster-info

# View all resources
kubectl get all --all-namespaces

# Check K3s version
kubectl version
```

---

## 📚 Common kubectl Commands

Here are some essential commands to get you started:

```bash
# View pods in default namespace
kubectl get pods

# View pods in all namespaces
kubectl get pods -A

# View services
kubectl get services

# View deployments
kubectl get deployments

# View nodes with details
kubectl get nodes -o wide

# Describe a specific pod
kubectl describe pod <pod-name>

# View pod logs
kubectl logs <pod-name>

# Execute command in a pod
kubectl exec -it <pod-name> -- /bin/sh

# Apply a configuration file
kubectl apply -f <filename.yaml>

# Delete a resource
kubectl delete pod <pod-name>
```

---

## 🔧 Troubleshooting

### K3s Service Won't Start

```bash
# Check service logs
sudo journalctl -u k3s -f

# Check system logs
sudo tail -f /var/log/syslog
```

### kubectl Connection Refused

```bash
# Verify K3s is running
sudo systemctl status k3s

# Check if the API server is listening
sudo netstat -tlnp | grep 6443
```

### Permission Denied Errors

```bash
# Verify KUBECONFIG is set
echo $KUBECONFIG

# Check file permissions
ls -la /etc/rancher/k3s/k3s.yaml
```

---

## 🗑️ Uninstallation

If you need to completely remove K3s:

```bash
# Uninstall K3s
sudo /usr/local/bin/k3s-uninstall.sh
```

**What this does:** Stops all K3s services, removes installed files, and cleans up resources.

---

## 🌟 Next Steps

Now that you have K3s up and running, you can:

1. **Deploy your first application**
   ```bash
   kubectl create deployment nginx --image=nginx
   kubectl expose deployment nginx --port=80 --type=NodePort
   ```

2. **Install Helm** (Kubernetes package manager)
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

3. **Explore K3s features**
   - Traefik Ingress Controller (included by default)
   - Local storage provider
   - Service load balancer (servicelb)

4. **Set up kubectl autocompletion**
   ```bash
   # For bash
   echo 'source <(kubectl completion bash)' >> ~/.bashrc
   
   # For zsh
   echo 'source <(kubectl completion zsh)' >> ~/.zshrc
   ```

---

## 📖 Additional Resources

- [Official K3s Documentation](https://docs.k3s.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [K3s GitHub Repository](https://github.com/k3s-io/k3s)

---

## 💡 Tips & Best Practices

- **Regular Updates**: Keep K3s updated for security patches
  ```bash
  curl -sfL https://get.k3s.io | sh -
  ```

- **Backup Configuration**: Regularly backup `/etc/rancher/k3s/k3s.yaml`

- **Resource Monitoring**: Use `kubectl top nodes` and `kubectl top pods` to monitor resource usage (requires metrics-server)

- **Namespace Organization**: Use namespaces to organize your applications
  ```bash
  kubectl create namespace my-app
  kubectl get pods -n my-app
  ```

---

**Happy Kubernetes clustering with K3s! 🎉**

