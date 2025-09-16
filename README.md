# Kubernetes Installation Guide

Kubernetes is a free and open-source container orchestration platform for automating deployment, scaling, and management of containerized applications. Originally developed by Google and now maintained by the Cloud Native Computing Foundation (CNCF), Kubernetes serves as the industry standard for container orchestration. It provides enterprise-grade security, scalability, and reliability for cloud-native applications, offering a robust alternative to proprietary solutions like AWS ECS, Azure Container Instances, or Google Cloud Run without vendor lock-in.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

- **Hardware Requirements**:
  - CPU: 2+ cores for control plane, 1+ core for worker nodes
  - RAM: 2GB+ per control plane node, 1GB+ per worker node
  - Storage: 20GB+ available disk space per node (SSD recommended)
  - Network: Stable connectivity between all nodes (1Gbps+ recommended)
- **Operating System**: 
  - Linux: Any modern distribution with kernel 3.10+ (4.x+ recommended)
  - Container runtime support (containerd, Docker, CRI-O)
  - macOS: Docker Desktop with Kubernetes enabled (development only)
  - Windows: Docker Desktop with Kubernetes enabled (development only)
- **Network Requirements**:
  - Unique hostname, MAC address, and product_uuid for every node
  - Port 6443 (Kubernetes API server)
  - Port 2379-2380 (etcd server client API)
  - Port 10250 (kubelet API)
  - Port 10259 (kube-scheduler)
  - Port 10257 (kube-controller-manager)
  - Port 30000-32767 (NodePort Services)
- **Dependencies**:
  - Container runtime (containerd recommended)
  - systemd or compatible init system
  - iptables (for network rules)
  - ebtables and ethtool (for networking)
- **System Access**: root or sudo privileges required
- **Special Requirements**:
  - Swap must be disabled on all nodes
  - SELinux in permissive mode (for RHEL/CentOS)
  - Firewall configured to allow cluster communication

## System Preparation (All Distributions)

### Disable Swap (Required)
```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is disabled
free -h
swapon --show
```

### Configure Kernel Modules
```bash
# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Container Runtime Installation

### containerd (Recommended)

#### Ubuntu/Debian
```bash
# Update package list
sudo apt-get update

# Install dependencies
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker repository for containerd
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

#### RHEL/CentOS/Rocky Linux/AlmaLinux
```bash
# Install prerequisites
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# Add Docker repository
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Install containerd
sudo yum install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

#### Fedora
```bash
# Install containerd
sudo dnf install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

#### Arch Linux
```bash
# Install containerd
sudo pacman -Syu containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Enable and start containerd
sudo systemctl enable --now containerd
```

## Kubernetes Installation

### kubeadm, kubelet, kubectl Installation

#### Ubuntu/Debian
```bash
# Update package index and install packages needed for apt to use HTTPS
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Download and add the Kubernetes signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package index and install Kubernetes components
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Hold packages to prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet
```

#### RHEL/CentOS/Rocky Linux/AlmaLinux/Fedora
```bash
# Create Kubernetes repository
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# Set SELinux to permissive mode (required for cluster communication)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Install Kubernetes components
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# Enable kubelet
sudo systemctl enable --now kubelet
```

#### Arch Linux
```bash
# Install from AUR (using yay)
yay -S kubeadm-bin kubelet-bin kubectl-bin

# Or build from source
git clone https://aur.archlinux.org/kubectl-bin.git
cd kubectl-bin && makepkg -si

# Enable kubelet
sudo systemctl enable --now kubelet
```

#### openSUSE/SLES
```bash
# openSUSE Leap/Tumbleweed
sudo zypper refresh

# Add Kubernetes repository
sudo rpm --import https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
echo 'baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/' | sudo tee /etc/zypp/repos.d/kubernetes.repo

# Install Kubernetes components
sudo zypper install -y kubelet kubeadm kubectl

# SLES 15 (requires additional modules)
sudo SUSEConnect -p sle-module-containers/15.5/x86_64
sudo zypper install -y kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet
```

#### Alpine Linux
```bash
# Install containerd first
apk add --no-cache containerd

# Add community repository for Kubernetes
echo "http://dl-cdn.alpinelinux.org/alpine/edge/community" >> /etc/apk/repositories
apk update

# Install Kubernetes components (if available)
apk add --no-cache kubectl

# Or install from binary
KUBE_VERSION="v1.29.0"
curl -LO "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubeadm"
curl -LO "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubelet"

chmod +x kubectl kubeadm kubelet
sudo mv kubectl kubeadm kubelet /usr/local/bin/

# Configure OpenRC service
sudo tee /etc/init.d/kubelet > /dev/null <<'EOF'
#!/sbin/openrc-run
name="kubelet"
command="/usr/local/bin/kubelet"
command_args="--config=/var/lib/kubelet/config.yaml --kubeconfig=/etc/kubernetes/kubelet.conf"
pidfile="/var/run/kubelet.pid"
command_background="yes"
depend() {
    need net
    after containerd
}
EOF

sudo chmod +x /etc/init.d/kubelet
sudo rc-update add kubelet default
```

#### macOS (Development Only)
```bash
# Install Docker Desktop
# Download from: https://www.docker.com/products/docker-desktop

# Enable Kubernetes in Docker Desktop
# Docker Desktop → Preferences → Kubernetes → Enable Kubernetes

# Install kubectl via Homebrew
brew install kubectl

# Alternative: Install kubectl directly
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
kubectl cluster-info

# Configure kubectl context
kubectl config use-context docker-desktop
```

#### FreeBSD
```bash
# Install from ports
cd /usr/ports/sysutils/kubectl && make install clean

# Or install from packages
pkg install kubectl

# Note: Full Kubernetes cluster on FreeBSD requires manual compilation
# For development, use kubectl to connect to remote clusters

# Install container runtime (if needed)
pkg install containerd

# Configure kubectl for remote cluster access
mkdir -p ~/.kube
# Copy kubeconfig from Linux cluster to ~/.kube/config
```

#### Windows (Development Only)
```powershell
# Method 1: Install Docker Desktop
# Download from: https://www.docker.com/products/docker-desktop
# Enable Kubernetes in Docker Desktop settings

# Method 2: Install using Chocolatey
choco install kubernetes-cli

# Method 3: Install using Scoop
scoop install kubectl

# Method 4: Manual installation
# Download from https://dl.k8s.io/release/v1.29.0/bin/windows/amd64/kubectl.exe
# Add to PATH environment variable

# Enable Kubernetes in Docker Desktop
# Docker Desktop → Settings → Kubernetes → Enable Kubernetes

# Verify installation
kubectl version --client
kubectl cluster-info

# Configure PowerShell completion (optional)
kubectl completion powershell | Out-String | Invoke-Expression
```

## Initial Configuration

### First-Run Setup

1. **Verify system requirements**:
```bash
# Check if swap is disabled
free -h
swapon --show

# Verify required ports are available
sudo ss -tlnp | grep -E ':(6443|2379|2380|10250|10259|10257)'

# Check container runtime
sudo systemctl status containerd
```

2. **Configure container runtime**:
```bash
# Default containerd configuration directory
sudo mkdir -p /etc/containerd

# Generate default configuration
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup for better resource management
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart containerd with new configuration
sudo systemctl restart containerd
```

3. **Initialize cluster networking**:
```bash
# Load required kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl for networking
echo 'net.bridge.bridge-nf-call-iptables=1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

4. **Set up kubelet configuration**:
```bash
# Create kubelet configuration directory
sudo mkdir -p /var/lib/kubelet

# Set kubelet to automatically restart
sudo systemctl enable kubelet
```

### Testing Initial Setup

```bash
# Verify container runtime is working
sudo crictl version
sudo crictl info

# Check if kubelet is ready
sudo systemctl status kubelet

# Verify kernel modules are loaded
lsmod | grep -E 'overlay|br_netfilter'

# Test container runtime with a simple container
sudo crictl pull k8s.gcr.io/pause:3.9
sudo crictl images
```

**WARNING:** Ensure all prerequisite steps are completed before proceeding to cluster initialization!

## 5. Service Management

### systemd (RHEL, Debian, Ubuntu, Arch, openSUSE)

```bash
# Enable Kubernetes services to start on boot
sudo systemctl enable kubelet
sudo systemctl enable containerd

# Start services
sudo systemctl start containerd
sudo systemctl start kubelet

# Stop services
sudo systemctl stop kubelet
sudo systemctl stop containerd

# Restart services
sudo systemctl restart containerd
sudo systemctl restart kubelet

# Check service status
sudo systemctl status kubelet
sudo systemctl status containerd

# View service logs
sudo journalctl -u kubelet -f
sudo journalctl -u containerd -f

# Reload systemd daemon after config changes
sudo systemctl daemon-reload
```

### OpenRC (Alpine Linux)

```bash
# Enable services to start on boot
sudo rc-update add containerd default
sudo rc-update add kubelet default

# Start services
sudo rc-service containerd start
sudo rc-service kubelet start

# Stop services
sudo rc-service kubelet stop
sudo rc-service containerd stop

# Restart services
sudo rc-service containerd restart
sudo rc-service kubelet restart

# Check service status
sudo rc-service kubelet status
sudo rc-service containerd status

# View logs
sudo tail -f /var/log/kubelet.log
```

### launchd (macOS with Docker Desktop)

```bash
# Docker Desktop manages Kubernetes services automatically
# Use Docker Desktop interface to start/stop Kubernetes

# Check Kubernetes status
kubectl cluster-info

# Restart Kubernetes through Docker Desktop
# Docker Desktop → Preferences → Kubernetes → Reset Kubernetes Cluster

# View logs through Docker Desktop
# Docker Desktop → Troubleshoot → Clean / Purge data
```

### Windows Service Manager

```powershell
# Docker Desktop manages Kubernetes services on Windows
# Use Docker Desktop interface for management

# Check Kubernetes status
kubectl cluster-info

# Restart Docker Desktop service
Restart-Service com.docker.service

# View Docker Desktop logs
Get-EventLog -LogName Application -Source "Docker Desktop"
```

## Cluster Initialization

### Control Plane Setup (Master Node)
```bash
# Initialize cluster with security best practices
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --apiserver-advertise-address=$(hostname -I | awk '{print $1}') \
  --node-name=$(hostname) \
  --ignore-preflight-errors=NumCPU

# Configure kubectl for regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Alternative: Root user configuration
export KUBECONFIG=/etc/kubernetes/admin.conf
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bashrc

# Verify control plane is running
kubectl cluster-info
kubectl get nodes
kubectl get pods --all-namespaces
```

### Network Plugin Installation

#### Flannel (Simple, recommended for beginners)
```bash
# Install Flannel CNI
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Verify Flannel pods are running
kubectl get pods -n kube-flannel
```

#### Calico (Advanced networking and network policies)
```bash
# Install Calico CNI
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Download and apply Calico custom resources
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml -O
kubectl create -f custom-resources.yaml

# Verify Calico is running
kubectl get pods -n calico-system
```

#### Cilium (eBPF-based networking)
```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Install Cilium
cilium install

# Verify installation
cilium status --wait
```

### Worker Node Setup
```bash
# On worker nodes, use the join command from control plane initialization
# Example (replace with your actual token and hash):
sudo kubeadm join 192.168.1.100:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef...

# If you need to get the join command again:
# On control plane:
kubeadm token create --print-join-command

# Verify nodes joined successfully
kubectl get nodes -o wide
```

## Security Hardening (2024 Best Practices)

### RBAC Configuration
```bash
# Create service account with limited permissions
kubectl create serviceaccount developer-sa -n default

# Create role with specific permissions
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: developer-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
EOF

# Create role binding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: developer-sa
  namespace: default
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Test RBAC configuration
kubectl auth can-i create deployments --as=system:serviceaccount:default:developer-sa
```

### Pod Security Standards
```bash
# Enable Pod Security Standards (baseline level)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline
EOF

# For restricted security (recommended for production)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: restricted-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
EOF

# Example secure pod configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: restricted-namespace
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    ports:
    - containerPort: 80
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
    - name: cache-volume
      mountPath: /var/cache/nginx
  volumes:
  - name: tmp-volume
    emptyDir: {}
  - name: cache-volume
    emptyDir: {}
EOF
```

### Network Policies
```bash
# Default deny all network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Allow specific communication
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
EOF

# Allow egress for DNS
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
EOF
```

### etcd Security
```bash
# Check etcd encryption at rest
kubectl get secrets --all-namespaces -o json | kubectl replace -f-

# Create encryption configuration
cat <<EOF | sudo tee /etc/kubernetes/enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps
  - pandas.awesome.bears.example
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: $(head -c 32 /dev/urandom | base64)
  - identity: {}
EOF

# Update kube-apiserver configuration
sudo sed -i '/--encryption-provider-config=/d' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sed -i '/- kube-apiserver/a\    - --encryption-provider-config=/etc/kubernetes/enc.yaml' /etc/kubernetes/manifests/kube-apiserver.yaml

# Mount encryption config in kube-apiserver
sudo sed -i '/volumeMounts:/a\    - mountPath: /etc/kubernetes/enc.yaml\n      name: encryption-config\n      readOnly: true' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sed -i '/volumes:/a\  - hostPath:\n      path: /etc/kubernetes/enc.yaml\n      type: FileOrCreate\n    name: encryption-config' /etc/kubernetes/manifests/kube-apiserver.yaml
```

## Alternative Installation Methods

### k3s (Lightweight Kubernetes)
```bash
# Install k3s on control plane
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644

# Get node token for workers
sudo cat /var/lib/rancher/k3s/server/node-token

# Install on worker nodes
curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -

# Configure kubectl
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config

# Verify cluster
kubectl get nodes
```

### k0s (Zero-deps Kubernetes)
```bash
# Download k0s
curl -sSLf https://get.k0s.sh | sudo sh

# Initialize controller
sudo k0s install controller --single

# Start k0s
sudo systemctl start k0scontroller

# Generate worker join token
sudo k0s token create --role=worker

# On worker nodes:
sudo k0s install worker --token-file /path/to/token/file
sudo systemctl start k0sworker

# Configure kubectl
mkdir -p ~/.kube
sudo k0s kubeconfig admin > ~/.kube/config
```

### MicroK8s (Ubuntu/Snap)
```bash
# Install MicroK8s
sudo snap install microk8s --classic

# Add user to microk8s group
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s

# Enable essential addons
microk8s enable dns dashboard storage

# Configure kubectl alias
echo 'alias kubectl="microk8s kubectl"' >> ~/.bashrc
source ~/.bashrc

# Get cluster info
microk8s kubectl cluster-info
```

### Minikube (Development)
```bash
# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster with specific configuration
minikube start \
  --driver=containerd \
  --cpus=4 \
  --memory=8g \
  --disk-size=50g \
  --kubernetes-version=v1.29.0

# Enable addons
minikube addons enable dashboard
minikube addons enable metrics-server
minikube addons enable ingress
minikube addons enable registry

# Configure kubectl context
kubectl config use-context minikube

# Access dashboard
minikube dashboard
```

## Essential Add-ons Installation

### Metrics Server
```bash
# Install metrics-server for resource monitoring
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For development clusters, may need to add --kubelet-insecure-tls
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Verify metrics server
kubectl top nodes
kubectl top pods --all-namespaces
```

### Kubernetes Dashboard
```bash
# Install dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Create admin service account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

# Get access token
kubectl -n kubernetes-dashboard create token admin-user

# Access dashboard
kubectl proxy &
# Visit: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

### Ingress Controller (NGINX)
```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# For bare metal installations
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# Create sample ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF
```

## Reverse Proxy Setup

### NGINX Ingress Controller Configuration

```bash
# Install NGINX Ingress Controller with custom configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  chart: ingress-nginx
  repo: https://kubernetes.github.io/ingress-nginx
  targetNamespace: ingress-nginx
  valuesContent: |-
    controller:
      replicaCount: 2
      service:
        type: LoadBalancer
        externalTrafficPolicy: Local
      config:
        ssl-redirect: "true"
        force-ssl-redirect: "true"
        ssl-protocols: "TLSv1.2 TLSv1.3"
        ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384"
        client-body-buffer-size: "64k"
        client-body-timeout: "60"
        client-header-timeout: "60"
        large-client-header-buffers: "4 64k"
        proxy-body-size: "50m"
        server-name-hash-bucket-size: "128"
      metrics:
        enabled: true
        serviceMonitor:
          enabled: true
EOF
```

### Traefik Ingress Controller

```bash
# Install Traefik with custom configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: traefik-system
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: traefik
  namespace: argocd
spec:
  project: default
  source:
    chart: traefik
    repoURL: https://traefik.github.io/charts
    targetRevision: 21.1.0
    helm:
      values: |
        deployment:
          replicas: 2
        service:
          type: LoadBalancer
        ingressRoute:
          dashboard:
            enabled: true
        logs:
          general:
            level: INFO
          access:
            enabled: true
        metrics:
          prometheus:
            enabled: true
        certificatesResolvers:
          letsencrypt:
            acme:
              email: admin@example.com
              storage: /data/acme.json
              httpChallenge:
                entryPoint: web
  destination:
    server: https://kubernetes.default.svc
    namespace: traefik-system
EOF
```

### HAProxy Load Balancer for API Server

```bash
# Install HAProxy for external load balancing
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-config
  namespace: kube-system
data:
  haproxy.cfg: |
    global
        log stdout local0
        daemon
        
    defaults
        mode tcp
        log global
        option tcplog
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms
        
    frontend k8s-api-frontend
        bind *:6443
        mode tcp
        default_backend k8s-api-backend
        
    backend k8s-api-backend
        mode tcp
        balance roundrobin
        option tcp-check
        server master1 10.0.1.10:6443 check
        server master2 10.0.1.11:6443 check
        server master3 10.0.1.12:6443 check
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: haproxy-lb
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: haproxy-lb
  template:
    metadata:
      labels:
        app: haproxy-lb
    spec:
      hostNetwork: true
      containers:
      - name: haproxy
        image: haproxy:2.8-alpine
        ports:
        - containerPort: 6443
          hostPort: 6443
        volumeMounts:
        - name: haproxy-config
          mountPath: /usr/local/etc/haproxy
      volumes:
      - name: haproxy-config
        configMap:
          name: haproxy-config
EOF
```

## Database Setup

### PostgreSQL StatefulSet with HA

```bash
# Deploy PostgreSQL cluster with replication
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: postgresql
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgresql-cluster
  namespace: postgresql
spec:
  instances: 3
  primaryUpdateStrategy: unsupervised
  
  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"
      effective_cache_size: "1GB"
      maintenance_work_mem: "64MB"
      checkpoint_completion_target: "0.9"
      wal_buffers: "16MB"
      default_statistics_target: "100"
      random_page_cost: "1.1"
      effective_io_concurrency: "200"
      
  bootstrap:
    initdb:
      database: app_database
      owner: app_user
      secret:
        name: postgresql-credentials
        
  storage:
    storageClass: "fast-ssd"
    size: "100Gi"
    
  monitoring:
    enabled: true
    
  backup:
    retentionPolicy: "30d"
    barmanObjectStore:
      destinationPath: s3://postgresql-backups/cluster1
      s3Credentials:
        accessKeyId:
          name: backup-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: backup-credentials
          key: SECRET_ACCESS_KEY
      wal:
        retention: "5d"
      data:
        retention: "30d"
---
apiVersion: v1
kind: Secret
metadata:
  name: postgresql-credentials
  namespace: postgresql
type: kubernetes.io/basic-auth
data:
  username: $(echo -n 'app_user' | base64)
  password: $(echo -n 'secure_database_password_123!' | base64)
EOF
```

### MySQL Cluster with Percona Operator

```bash
# Deploy MySQL cluster using Percona Operator
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: mysql
---
apiVersion: pxc.percona.com/v1-12-0
kind: PerconaXtraDBCluster
metadata:
  name: mysql-cluster
  namespace: mysql
spec:
  crVersion: 1.12.0
  allowUnsafeConfigurations: false
  secretsName: mysql-secrets
  vaultSecretName: ""
  sslSecretName: ""
  sslInternalSecretName: ""
  logCollectorSecretName: ""
  
  pxc:
    size: 3
    image: percona/percona-xtradb-cluster:8.0.32-24.2
    autoRecovery: true
    configuration: |
      [mysqld]
      wsrep_provider_options="debug=1;gcache.size=1G;gcache.page_size=1G"
      wsrep_debug=1
      wsrep_cluster_address=gcomm://
      binlog_format=ROW
      default_storage_engine=InnoDB
      innodb_autoinc_lock_mode=2
      innodb_locks_unsafe_for_binlog=1
      max_connections=350
      innodb_buffer_pool_size=512M
      
    resources:
      requests:
        memory: 1G
        cpu: 600m
      limits:
        memory: 1G
        cpu: "1"
        
    volumeSpec:
      persistentVolumeClaim:
        storageClassName: fast-ssd
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 80Gi
            
    affinity:
      antiAffinityTopologyKey: "kubernetes.io/hostname"
      
  haproxy:
    enabled: true
    size: 2
    image: percona/percona-xtradb-cluster-operator:1.12.0-haproxy
    
    resources:
      requests:
        memory: 256M
        cpu: 250m
      limits:
        memory: 256M
        cpu: 500m
        
  proxysql:
    enabled: false
    
  backup:
    image: percona/percona-xtradb-cluster-operator:1.12.0-pxc8.0-backup
    schedule:
      - name: "daily-backup"
        schedule: "0 2 * * *"
        keep: 7
        storageName: s3-backup-storage
        
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secrets
  namespace: mysql
type: Opaque
data:
  root: $(echo -n 'secure_mysql_root_password!' | base64)
  xtrabackup: $(echo -n 'backup_password_123!' | base64)
  monitor: $(echo -n 'monitor_user_password!' | base64)
  clustercheck: $(echo -n 'cluster_check_password!' | base64)
  proxysql: $(echo -n 'proxysql_admin_password!' | base64)
  operator: $(echo -n 'operator_user_password!' | base64)
EOF
```

### Redis Cluster Deployment

```bash
# Deploy Redis cluster with Redis Operator
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: redis
---
apiVersion: redis.redis.opstreelabs.in/v1beta1
kind: RedisCluster
metadata:
  name: redis-cluster
  namespace: redis
spec:
  clusterSize: 6
  clusterVersion: v7
  persistenceEnabled: true
  redisSecret:
    name: redis-secret
    key: password
  redisConfig:
    redis-config: |
      maxmemory 512mb
      maxmemory-policy allkeys-lru
      save 900 1
      save 300 10
      save 60 10000
      tcp-keepalive 60
      tcp-backlog 8192
      timeout 300
      
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi
            
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 512Mi
      cpu: 500m
      
  nodeSelector:
    node-type: "redis-optimized"
    
  podSecurityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    
  securityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    runAsNonRoot: true
    capabilities:
      drop:
      - ALL
---
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: redis
type: Opaque
data:
  password: $(echo -n 'secure_redis_password_123!' | base64)
EOF
```

## Storage Configuration

### Persistent Volumes and Storage Classes
```bash
# Create local storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
EOF

# Create persistent volume
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-1
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disk1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
EOF

# Create persistent volume claim
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 5Gi
EOF
```

### NFS Storage (Shared volumes)
```bash
# Install NFS client utilities (all nodes)
# Ubuntu/Debian
sudo apt install -y nfs-common

# RHEL/CentOS
sudo yum install -y nfs-utils

# Create NFS storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: example.com/nfs
parameters:
  server: 192.168.1.200
  path: /exported/path
  readOnly: "false"
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - hard
  - nfsvers=4.1
EOF
```

## Firewall Configuration (Cross-Platform)

### Required Ports
```bash
# Control plane ports
sudo firewall-cmd --permanent --add-port=6443/tcp    # API server
sudo firewall-cmd --permanent --add-port=2379-2380/tcp  # etcd
sudo firewall-cmd --permanent --add-port=10250/tcp  # kubelet
sudo firewall-cmd --permanent --add-port=10259/tcp  # kube-scheduler
sudo firewall-cmd --permanent --add-port=10257/tcp  # kube-controller-manager

# Worker node ports
sudo firewall-cmd --permanent --add-port=10250/tcp  # kubelet
sudo firewall-cmd --permanent --add-port=30000-32767/tcp  # NodePort services

# CNI ports (Flannel)
sudo firewall-cmd --permanent --add-port=8285/udp   # Flannel
sudo firewall-cmd --permanent --add-port=8472/udp   # Flannel VXLAN

sudo firewall-cmd --reload

# UFW (Ubuntu/Debian)
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10259/tcp
sudo ufw allow 10257/tcp
sudo ufw allow 30000:32767/tcp
sudo ufw allow 8285/udp
sudo ufw allow 8472/udp

# iptables (manual configuration)
sudo iptables -A INPUT -p tcp --dport 6443 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 2379:2380 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 10250 -j ACCEPT
```

### SELinux Configuration (RHEL/CentOS)
```bash
# Configure SELinux for Kubernetes
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Alternative: Configure SELinux policies instead of disabling
sudo setsebool -P container_manage_cgroup true
sudo setsebool -P container_use_cgroup true

# Install SELinux policies for containers
sudo yum install -y container-selinux

# Check for denials
sudo ausearch -m AVC,USER_AVC -ts recent
```

## High Availability Setup

### Multi-Master Cluster with kubeadm
```bash
# On first control plane node
sudo kubeadm init \
  --control-plane-endpoint="k8s-cluster.example.com:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# Note the commands to join additional control plane nodes and workers

# On additional control plane nodes:
sudo kubeadm join k8s-cluster.example.com:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:1234... \
  --control-plane \
  --certificate-key 1234...

# Configure load balancer (HAProxy example)
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
global
    log stdout local0
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    mode tcp
    log global
    option tcplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend k8s-api
    bind *:6443
    mode tcp
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    balance roundrobin
    server k8s-master-1 192.168.1.101:6443 check
    server k8s-master-2 192.168.1.102:6443 check
    server k8s-master-3 192.168.1.103:6443 check
EOF

sudo systemctl restart haproxy
```

### External etcd Cluster
```bash
# Install etcd on dedicated nodes
ETCD_VER=v3.5.9
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzf etcd-${ETCD_VER}-linux-amd64.tar.gz
sudo mv etcd-${ETCD_VER}-linux-amd64/{etcd,etcdctl} /usr/local/bin/

# Create etcd configuration
sudo tee /etc/systemd/system/etcd.service > /dev/null <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
User=etcd
ExecStart=/usr/local/bin/etcd \\
  --name=etcd-1 \\
  --data-dir=/var/lib/etcd \\
  --listen-client-urls=https://192.168.1.201:2379 \\
  --advertise-client-urls=https://192.168.1.201:2379 \\
  --listen-peer-urls=https://192.168.1.201:2380 \\
  --initial-advertise-peer-urls=https://192.168.1.201:2380 \\
  --initial-cluster=etcd-1=https://192.168.1.201:2380,etcd-2=https://192.168.1.202:2380,etcd-3=https://192.168.1.203:2380 \\
  --initial-cluster-token=etcd-cluster-1 \\
  --initial-cluster-state=new \\
  --cert-file=/etc/etcd/pki/server.crt \\
  --key-file=/etc/etcd/pki/server.key \\
  --peer-cert-file=/etc/etcd/pki/peer.crt \\
  --peer-key-file=/etc/etcd/pki/peer.key \\
  --trusted-ca-file=/etc/etcd/pki/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/pki/ca.crt \\
  --peer-client-cert-auth \\
  --client-cert-auth
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Create etcd user and directories
sudo useradd -r etcd
sudo mkdir -p /var/lib/etcd /etc/etcd/pki
sudo chown etcd:etcd /var/lib/etcd
sudo systemctl enable --now etcd
```

## Application Deployment Examples

### Secure Application Deployment
```bash
# Create namespace with network policies
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: myapp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: myapp
  name: myapp-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: myapp
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: myapp
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: myapp-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: cache-volume
          mountPath: /var/cache/nginx
      volumes:
      - name: tmp-volume
        emptyDir: {}
      - name: cache-volume
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

### StatefulSet with Persistent Storage
```bash
# Deploy StatefulSet application (database example)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-statefulset
  namespace: myapp
spec:
  serviceName: mysql-service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      securityContext:
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
  volumeClaimTemplates:
  - metadata:
      name: mysql-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "local-storage"
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: myapp
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: myapp
type: Opaque
data:
  root-password: $(echo -n 'secure_mysql_password' | base64)
EOF
```

## Backup and Disaster Recovery

### etcd Backup
```bash
# Create etcd backup script
sudo tee /usr/local/bin/etcd-backup.sh > /dev/null <<'EOF'
#!/bin/bash
BACKUP_DIR="/backup/etcd"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p ${BACKUP_DIR}

# Create etcd snapshot
ETCDCTL_API=3 etcdctl snapshot save ${BACKUP_DIR}/etcd-backup-${DATE}.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status ${BACKUP_DIR}/etcd-backup-${DATE}.db -w table

# Keep only last 7 backups
find ${BACKUP_DIR} -name "etcd-backup-*.db" -type f -mtime +7 -delete

echo "etcd backup completed: etcd-backup-${DATE}.db"
EOF

sudo chmod +x /usr/local/bin/etcd-backup.sh

# Schedule backup
echo "0 2 * * * root /usr/local/bin/etcd-backup.sh" | sudo tee -a /etc/crontab
```

### Cluster State Backup
```bash
# Backup all cluster resources
kubectl get all --all-namespaces -o yaml > cluster-backup-$(date +%Y%m%d).yaml

# Backup specific resource types
kubectl get configmaps,secrets,persistentvolumes,persistentvolumeclaims --all-namespaces -o yaml > cluster-data-backup-$(date +%Y%m%d).yaml

# Create backup script for all resources
sudo tee /usr/local/bin/k8s-backup.sh > /dev/null <<'EOF'
#!/bin/bash
BACKUP_DIR="/backup/kubernetes"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p ${BACKUP_DIR}

# Backup all cluster resources
kubectl get all --all-namespaces -o yaml > ${BACKUP_DIR}/cluster-all-${DATE}.yaml

# Backup critical resources separately
kubectl get configmaps,secrets,persistentvolumes,persistentvolumeclaims --all-namespaces -o yaml > ${BACKUP_DIR}/cluster-data-${DATE}.yaml

# Backup custom resources
kubectl get crd -o yaml > ${BACKUP_DIR}/cluster-crd-${DATE}.yaml

# Backup RBAC
kubectl get clusterroles,clusterrolebindings,roles,rolebindings --all-namespaces -o yaml > ${BACKUP_DIR}/cluster-rbac-${DATE}.yaml

# etcd backup
/usr/local/bin/etcd-backup.sh

# Compress backups
tar -czf ${BACKUP_DIR}/k8s-complete-backup-${DATE}.tar.gz ${BACKUP_DIR}/*-${DATE}.yaml

# Keep only last 7 backups
find ${BACKUP_DIR} -name "*-${DATE:0:8}*" -type f -mtime +7 -delete

echo "Kubernetes backup completed: ${DATE}"
EOF

sudo chmod +x /usr/local/bin/k8s-backup.sh
echo "0 3 * * * root /usr/local/bin/k8s-backup.sh" | sudo tee -a /etc/crontab
```

## Verification and Testing

### Cluster Health Checks
```bash
# Check cluster components
kubectl get componentstatuses
kubectl cluster-info
kubectl get nodes -o wide

# Check all pods in system namespaces
kubectl get pods --all-namespaces
kubectl get events --all-namespaces --sort-by=.metadata.creationTimestamp

# Test DNS resolution
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default

# Test pod networking
kubectl run test-pod-1 --image=nginx --port=80
kubectl expose pod test-pod-1 --port=80 --type=ClusterIP
kubectl run test-pod-2 --image=busybox --rm -it --restart=Never -- wget -qO- test-pod-1

# Check resource usage
kubectl top nodes
kubectl top pods --all-namespaces

# Verify RBAC
kubectl auth can-i create deployments
kubectl auth can-i get secrets --as=system:serviceaccount:default:default

# Test persistent storage
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-storage-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "Storage test" > /data/test.txt && cat /data/test.txt && sleep 3600']
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc
EOF

kubectl logs test-storage-pod
kubectl delete pod test-storage-pod
kubectl delete pvc test-pvc
```

### Security Validation
```bash
# Run CIS Kubernetes Benchmark
docker run --rm -v $(pwd):/tmp aquasec/kube-bench:latest run --targets master,node,etcd,policies

# Check pod security policies
kubectl get psp  # For older versions
kubectl get podsecuritypolicies  # For older versions

# Verify network policies are working
kubectl describe networkpolicy default-deny-all

# Check for privileged containers
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.securityContext.privileged}{"\n"}{end}' | grep true

# Audit security contexts
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.securityContext}{"\n"}{end}'

# Check for containers running as root
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].securityContext.runAsUser}{"\n"}{end}' | grep -E '\t0$|\t$'
```

## 6. Troubleshooting (Cross-Platform)

### Common Issues and Solutions
```bash
# Node not ready issues
kubectl describe node <node-name>
kubectl get events --sort-by=.metadata.creationTimestamp

# Check kubelet logs
sudo journalctl -u kubelet -f

# Check container runtime
sudo systemctl status containerd
sudo crictl pods

# Network issues
kubectl get pods -n kube-system
kubectl describe pod <cni-pod-name> -n kube-system

# Permission issues (SELinux)
sudo ausearch -m AVC -ts recent
sudo setsebool -P container_manage_cgroup true

# Certificate issues
sudo kubeadm certs check-expiration
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout

# Resource exhaustion
kubectl describe node <node-name>
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory

# etcd issues
sudo etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Reset cluster (if needed)
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config
```

### Debug Pod Issues
```bash
# Debug failing pods
kubectl describe pod <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl get events --field-selector involvedObject.name=<pod-name>

# Debug networking
kubectl run debug-pod --image=nicolaka/netshoot --rm -it --restart=Never

# Check resource constraints
kubectl describe resourcequota -n <namespace>
kubectl describe limitrange -n <namespace>

# Debug storage issues
kubectl describe pvc <pvc-name>
kubectl get events --field-selector involvedObject.name=<pvc-name>

# Debug service connectivity
kubectl run debug --image=busybox --rm -it --restart=Never -- nslookup <service-name>
kubectl get endpoints <service-name>

# Debug ingress issues
kubectl describe ingress <ingress-name>
kubectl get events --field-selector involvedObject.name=<ingress-name>
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

## Performance Optimization

### System-Level Tuning

```bash
# Kernel optimization for Kubernetes
sudo tee -a /etc/sysctl.conf > /dev/null <<EOF
# Kubernetes performance tuning
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 2000000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 10
net.ipv4.ip_local_port_range = 1024 65000
net.core.rmem_default = 262144
net.core.rmem_max = 134217728
net.core.wmem_default = 262144
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
fs.file-max = 2097152
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 1048576
vm.swappiness = 0
vm.overcommit_memory = 1
vm.dirty_ratio = 80
vm.dirty_background_ratio = 5
kernel.pid_max = 4194304
EOF

sudo sysctl -p

# Set resource limits
sudo tee -a /etc/security/limits.conf > /dev/null <<EOF
root soft nofile 65536
root hard nofile 65536
* soft nofile 65536
* hard nofile 65536
* soft nproc 65536
* hard nproc 65536
EOF

# Configure systemd limits for containerd and kubelet
sudo mkdir -p /etc/systemd/system/containerd.service.d/
sudo tee /etc/systemd/system/containerd.service.d/limits.conf > /dev/null <<EOF
[Service]
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
OOMScoreAdjust=-999
EOF

sudo mkdir -p /etc/systemd/system/kubelet.service.d/
sudo tee /etc/systemd/system/kubelet.service.d/limits.conf > /dev/null <<EOF
[Service]
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
OOMScoreAdjust=-999
EOF

sudo systemctl daemon-reload
sudo systemctl restart containerd kubelet
```

### Kubernetes Performance Configuration

```bash
# Optimize kubelet configuration
sudo tee /var/lib/kubelet/config.yaml > /dev/null <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 0
authentication:
  webhook:
    enabled: true
authorization:
  mode: Webhook
clusterDomain: cluster.local
clusterDNS:
- 10.96.0.10
maxPods: 250
podsPerCore: 10
cgroupDriver: systemd
containerLogMaxSize: 50Mi
containerLogMaxFiles: 5
eventRecordQPS: 50
eventBurst: 100
kubeAPIQPS: 50
kubeAPIBurst: 100
serializeImagePulls: false
registryPullQPS: 10
registryBurst: 20
syncFrequency: 1m
fileCheckFrequency: 20s
httpCheckFrequency: 20s
nodeStatusUpdateFrequency: 10s
imageMinimumGCAge: 2m
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
volumeStatsAggPeriod: 1m
systemReserved:
  cpu: 200m
  memory: 512Mi
  ephemeral-storage: 2Gi
kubeReserved:
  cpu: 200m
  memory: 512Mi
  ephemeral-storage: 2Gi
EOF

# Optimize API server configuration
sudo sed -i '/- kube-apiserver/a\
    - --max-requests-inflight=2000\
    - --max-mutating-requests-inflight=1000\
    - --watch-cache-sizes=nodes#100,pods#1000,replicationcontrollers#500\
    - --target-ram-mb=2048\
    - --event-ttl=168h0m0s' /etc/kubernetes/manifests/kube-apiserver.yaml

# Optimize etcd configuration
sudo tee -a /etc/kubernetes/manifests/etcd.yaml > /dev/null <<EOF
    - --max-request-bytes=33554432
    - --quota-backend-bytes=8589934592
    - --snapshot-count=10000
    - --heartbeat-interval=100
    - --election-timeout=1000
EOF
```

## Integration Examples

### Python Client Library

```python
# kubernetes-client example
from kubernetes import client, config
import json

# Load kubeconfig
config.load_kube_config()  # or config.load_incluster_config() for in-cluster

# Initialize API clients
v1 = client.CoreV1Api()
apps_v1 = client.AppsV1Api()
networking_v1 = client.NetworkingV1Api()

# Create a namespace
namespace = client.V1Namespace(
    metadata=client.V1ObjectMeta(name="python-app")
)
v1.create_namespace(body=namespace)

# Create a deployment
deployment = client.V1Deployment(
    metadata=client.V1ObjectMeta(name="nginx-deployment"),
    spec=client.V1DeploymentSpec(
        replicas=3,
        selector=client.V1LabelSelector(
            match_labels={"app": "nginx"}
        ),
        template=client.V1PodTemplateSpec(
            metadata=client.V1ObjectMeta(
                labels={"app": "nginx"}
            ),
            spec=client.V1PodSpec(
                containers=[
                    client.V1Container(
                        name="nginx",
                        image="nginx:alpine",
                        ports=[client.V1ContainerPort(container_port=80)],
                        resources=client.V1ResourceRequirements(
                            requests={"cpu": "100m", "memory": "128Mi"},
                            limits={"cpu": "500m", "memory": "512Mi"}
                        )
                    )
                ]
            )
        )
    )
)

apps_v1.create_namespaced_deployment(
    namespace="python-app", 
    body=deployment
)

# Create a service
service = client.V1Service(
    metadata=client.V1ObjectMeta(name="nginx-service"),
    spec=client.V1ServiceSpec(
        selector={"app": "nginx"},
        ports=[
            client.V1ServicePort(port=80, target_port=80)
        ],
        type="LoadBalancer"
    )
)

v1.create_namespaced_service(namespace="python-app", body=service)

# Monitor pods
def monitor_pods():
    pods = v1.list_namespaced_pod(namespace="python-app")
    for pod in pods.items:
        print(f"Pod: {pod.metadata.name}, Status: {pod.status.phase}")

monitor_pods()

# Stream logs
def stream_logs(pod_name):
    for line in v1.read_namespaced_pod_log(
        name=pod_name, 
        namespace="python-app", 
        follow=True, 
        _preload_content=False
    ).stream():
        print(line.decode('utf-8'), end='')

# Clean up
apps_v1.delete_namespaced_deployment(name="nginx-deployment", namespace="python-app")
v1.delete_namespaced_service(name="nginx-service", namespace="python-app")
v1.delete_namespace(name="python-app")
```

### Node.js Client Example

```javascript
// kubernetes-example.js
const k8s = require('@kubernetes/client-node');

// Load kubeconfig
const kc = new k8s.KubeConfig();
kc.loadFromDefault();

const k8sApi = kc.makeApiClient(k8s.CoreV1Api);
const k8sAppsApi = kc.makeApiClient(k8s.AppsV1Api);

const namespace = 'nodejs-app';

async function createNamespace() {
    const namespaceManifest = {
        metadata: {
            name: namespace
        }
    };
    
    try {
        await k8sApi.createNamespace(namespaceManifest);
        console.log(`Namespace ${namespace} created`);
    } catch (error) {
        console.error('Error creating namespace:', error.response?.body || error.message);
    }
}

async function createDeployment() {
    const deploymentManifest = {
        metadata: {
            name: 'nginx-deployment'
        },
        spec: {
            replicas: 3,
            selector: {
                matchLabels: {
                    app: 'nginx'
                }
            },
            template: {
                metadata: {
                    labels: {
                        app: 'nginx'
                    }
                },
                spec: {
                    containers: [{
                        name: 'nginx',
                        image: 'nginx:alpine',
                        ports: [{
                            containerPort: 80
                        }],
                        resources: {
                            requests: {
                                cpu: '100m',
                                memory: '128Mi'
                            },
                            limits: {
                                cpu: '500m',
                                memory: '512Mi'
                            }
                        }
                    }]
                }
            }
        }
    };
    
    try {
        await k8sAppsApi.createNamespacedDeployment(namespace, deploymentManifest);
        console.log('Deployment created: nginx-deployment');
    } catch (error) {
        console.error('Error creating deployment:', error.response?.body || error.message);
    }
}

async function createService() {
    const serviceManifest = {
        metadata: {
            name: 'nginx-service'
        },
        spec: {
            selector: {
                app: 'nginx'
            },
            ports: [{
                port: 80,
                targetPort: 80
            }],
            type: 'LoadBalancer'
        }
    };
    
    try {
        await k8sApi.createNamespacedService(namespace, serviceManifest);
        console.log('Service created: nginx-service');
    } catch (error) {
        console.error('Error creating service:', error.response?.body || error.message);
    }
}

async function listPods() {
    try {
        const response = await k8sApi.listNamespacedPod(namespace);
        console.log(`Found ${response.body.items.length} pods:`);
        response.body.items.forEach(pod => {
            console.log(`Pod: ${pod.metadata.name}, Status: ${pod.status.phase}`);
        });
    } catch (error) {
        console.error('Error listing pods:', error.response?.body || error.message);
    }
}

async function cleanup() {
    try {
        await k8sAppsApi.deleteNamespacedDeployment('nginx-deployment', namespace);
        await k8sApi.deleteNamespacedService('nginx-service', namespace);
        await k8sApi.deleteNamespace(namespace);
        console.log('Resources cleaned up');
    } catch (error) {
        console.error('Error during cleanup:', error.response?.body || error.message);
    }
}

async function main() {
    await createNamespace();
    await createDeployment();
    await createService();
    
    // Wait a bit for pods to start
    setTimeout(async () => {
        await listPods();
        await cleanup();
    }, 5000);
}

main().catch(console.error);
```

### Java Client Example

```java
// KubernetesExample.java
import io.kubernetes.client.openapi.ApiClient;
import io.kubernetes.client.openapi.ApiException;
import io.kubernetes.client.openapi.Configuration;
import io.kubernetes.client.openapi.apis.AppsV1Api;
import io.kubernetes.client.openapi.apis.CoreV1Api;
import io.kubernetes.client.openapi.models.*;
import io.kubernetes.client.util.Config;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

public class KubernetesExample {
    public static void main(String[] args) throws Exception {
        // Load kubeconfig
        ApiClient client = Config.defaultClient();
        Configuration.setDefaultApiClient(client);
        
        CoreV1Api coreV1Api = new CoreV1Api();
        AppsV1Api appsV1Api = new AppsV1Api();
        
        String namespace = "java-app";
        
        // Create namespace
        V1Namespace ns = new V1Namespace()
            .metadata(new V1ObjectMeta().name(namespace));
        
        try {
            coreV1Api.createNamespace(ns, null, null, null, null);
            System.out.println("Namespace created: " + namespace);
        } catch (ApiException e) {
            System.err.println("Failed to create namespace: " + e.getResponseBody());
        }
        
        // Create deployment
        Map<String, String> labels = new HashMap<>();
        labels.put("app", "nginx");
        
        V1Deployment deployment = new V1Deployment()
            .metadata(new V1ObjectMeta().name("nginx-deployment"))
            .spec(new V1DeploymentSpec()
                .replicas(3)
                .selector(new V1LabelSelector().matchLabels(labels))
                .template(new V1PodTemplateSpec()
                    .metadata(new V1ObjectMeta().labels(labels))
                    .spec(new V1PodSpec()
                        .containers(Collections.singletonList(
                            new V1Container()
                                .name("nginx")
                                .image("nginx:alpine")
                                .ports(Collections.singletonList(
                                    new V1ContainerPort().containerPort(80)
                                ))
                                .resources(new V1ResourceRequirements()
                                    .requests(Map.of(
                                        "cpu", Quantity.fromString("100m"),
                                        "memory", Quantity.fromString("128Mi")
                                    ))
                                    .limits(Map.of(
                                        "cpu", Quantity.fromString("500m"),
                                        "memory", Quantity.fromString("512Mi")
                                    ))
                                )
                        ))
                    )
                )
            );
            
        try {
            appsV1Api.createNamespacedDeployment(namespace, deployment, null, null, null, null);
            System.out.println("Deployment created: nginx-deployment");
        } catch (ApiException e) {
            System.err.println("Failed to create deployment: " + e.getResponseBody());
        }
        
        // Create service
        V1Service service = new V1Service()
            .metadata(new V1ObjectMeta().name("nginx-service"))
            .spec(new V1ServiceSpec()
                .selector(labels)
                .ports(Collections.singletonList(
                    new V1ServicePort().port(80).targetPort(new IntOrString(80))
                ))
                .type("LoadBalancer")
            );
            
        try {
            coreV1Api.createNamespacedService(namespace, service, null, null, null, null);
            System.out.println("Service created: nginx-service");
        } catch (ApiException e) {
            System.err.println("Failed to create service: " + e.getResponseBody());
        }
        
        // List pods
        try {
            V1PodList pods = coreV1Api.listNamespacedPod(namespace, null, null, null, null, null, null, null, null, null, null);
            System.out.println("Found " + pods.getItems().size() + " pods:");
            for (V1Pod pod : pods.getItems()) {
                System.out.println("Pod: " + pod.getMetadata().getName() + 
                                 ", Status: " + pod.getStatus().getPhase());
            }
        } catch (ApiException e) {
            System.err.println("Failed to list pods: " + e.getResponseBody());
        }
        
        // Cleanup
        try {
            appsV1Api.deleteNamespacedDeployment("nginx-deployment", namespace, null, null, null, null, null, null);
            coreV1Api.deleteNamespacedService("nginx-service", namespace, null, null, null, null, null, null);
            coreV1Api.deleteNamespace(namespace, null, null, null, null, null, null);
            System.out.println("Resources cleaned up");
        } catch (ApiException e) {
            System.err.println("Failed to cleanup: " + e.getResponseBody());
        }
    }
}
```

## Maintenance

### Update Procedures

```bash
# Check current Kubernetes version
kubectl version --short

# Plan upgrade with kubeadm
sudo kubeadm upgrade plan

# Upgrade kubeadm first
sudo apt update && sudo apt-mark unhold kubeadm
sudo apt install -y kubeadm=1.29.1-00
sudo apt-mark hold kubeadm

# Or for RHEL/CentOS
sudo yum update -y kubeadm-1.29.1

# Upgrade control plane
sudo kubeadm upgrade apply v1.29.1

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt install -y kubelet=1.29.1-00 kubectl=1.29.1-00
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Drain and upgrade worker nodes
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data
# On worker node:
sudo kubeadm upgrade node
sudo apt install -y kubelet=1.29.1-00 kubectl=1.29.1-00
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon worker node
kubectl uncordon <worker-node>
```

### Maintenance Tasks

```bash
# Weekly maintenance script
#!/bin/bash
# k8s-maintenance.sh

# Check cluster health
echo "=== Cluster Health Check ==="
kubectl get nodes -o wide
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# Check resource usage
echo "=== Resource Usage ==="
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=cpu | head -10

# Check certificate expiry
echo "=== Certificate Expiry ==="
sudo kubeadm certs check-expiration

# Clean up completed jobs
echo "=== Cleanup ==="
kubectl get jobs --all-namespaces -o json | jq -r '.items[] | select(.status.conditions[]?.type == "Complete") | "\(.metadata.namespace) \(.metadata.name)"' | xargs -l bash -c 'kubectl delete job $1 -n $0'

# Clean up evicted pods
kubectl get pods --all-namespaces --field-selector=status.phase=Failed -o json | jq -r '.items[] | "\(.metadata.namespace) \(.metadata.name)"' | xargs -l bash -c 'kubectl delete pod $1 -n $0'

# Check for security updates
echo "=== Security Updates Available ==="
sudo apt list --upgradable | grep -i security

# Backup etcd
echo "=== etcd Backup ==="
sudo etcdctl snapshot save /backup/etcd-$(date +%Y%m%d_%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

echo "Maintenance completed at: $(date)"
```

## Additional Resources

- [Official Documentation](https://kubernetes.io/docs/)
- [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)
- [Security Best Practices](https://kubernetes.io/docs/concepts/security/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [OWASP Kubernetes Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [Kubernetes Academy](https://kubernetes.academy/)
- [CNCF Landscape](https://landscape.cncf.io/)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection.