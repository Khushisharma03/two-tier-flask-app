# Stackboard — Two-Tier Flask Application on Kubernetes

> A production-grade two-tier Flask + MySQL application deployed on a self-managed Kubernetes cluster bootstrapped with kubeadm on AWS EC2.

---

## Architecture Overview

```
                        ┌─────────────────────────────────────┐
                        │          AWS EC2 Cluster            │
                        │                                     │
                        │  ┌─────────────┐  ┌─────────────┐   │
                        │  │ Master Node │  │ Worker Node │   │
                        │  │ (Control    │  │             │   │
                        │  │  Plane)     │  │  Flask Pods │   │
                        │  └─────────────┘  │  MySQL Pod  │   │
                        │                   └─────────────┘   │
                        │         Calico CNI Networking       │
                        └─────────────────────────────────────┘
                                        │
                              ┌─────────┴──────────┐
                              │   Flask App (x4)   │
                              │   NodePort Service │
                              └─────────┬──────────┘
                                        │
                              ┌─────────┴──────────┐
                              │   MySQL Database   │
                              │   ClusterIP Service│
                              └────────────────────┘
```


## Tech Stack

| Technology | Purpose |
|---|---|
| Flask | Python web application framework |
| MySQL | Relational database |
| Docker | Containerization |
| Kubernetes (kubeadm) | Container orchestration |
| Calico CNI | Pod networking and network policies |
| AWS EC2 | Cloud infrastructure |
| containerd | Container runtime |

---

## Project Structure

```
two-tier-flask-app/
├── app.py                  # Flask application
├── Dockerfile              # Docker image definition
├── docker-compose.yml      # Local development setup
├── requirements.txt        # Python dependencies
├── templates/
│   └── index.html          # Frontend template
└── k8s/
    ├── stackboard-deployment.yml    # Flask Deployment + replicas
    ├── stackboard-svc.yml       # NodePort Service for Flask
    ├── mysql-deployment.yml    # MySQL Deployment
    └── mysql-service.yml       # ClusterIP Service for MySQL
```

## Prerequisites

- AWS EC2 instances (t3.medium recommended — 2 vCPUs, 4GB RAM minimum)
- Ubuntu 22.04 LTS
- Ports open in Security Group:
  - `6443` — Kubernetes API server
  - `30000-32767` — NodePort range
  - `179` — Calico BGP
  - `2379-2380` — etcd
  - `10250` — kubelet

---

## Cluster Setup

### 1. Install dependencies on all nodes (Master + Worker)

```bash
sudo apt update && sudo apt upgrade -y

# Install containerd
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Enable kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 2. Initialize the Master Node

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3. Install Calico CNI

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

### 4. Join Worker Nodes

```bash
# Run the kubeadm join command from the master init output
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### 5. Verify Cluster

```bash
kubectl get nodes
# Both nodes should show Ready status
```

---

## Deploy the Application

### 1. Clone the repository

```bash
git clone https://github.com/Khushisharma03/two-tier-flask-app.git
cd two-tier-flask-app
```

### 2. Apply Kubernetes manifests

```bash
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/mysql-svc.yml
kubectl apply -f k8s/stackboard-deployment.yml
kubectl apply -f k8s/stackboard-svc.yml
```

### 3. Verify deployment

```bash
# Check all pods are running
kubectl get pods -o wide

# Check services
kubectl get svc

# Check deployments
kubectl get deployments
```

### 4. Access the application

```bash
# Get the NodePort
kubectl get svc stackboard-service

# Access via browser
http://<worker-node-public-ip>:<NodePort>
```

---

## Environment Variables

The Flask deployment requires these environment variables — make sure they match exactly in your `stackboard-deployment.yaml`:

| Variable | Description | Example |
|---|---|---|
| `MYSQL_HOST` | Kubernetes MySQL service name | `mysql` |
| `MYSQL_USER` | Database username | `admin` |
| `MYSQL_PASSWORD` | Database password | `admin` |
| `MYSQL_DATABASE` | Database name | `mydb` |

## Common Issues & Fixes

### CrashLoopBackOff
```bash
# Check logs of crashed pod
kubectl logs <pod-name> --previous

# Check pod events
kubectl describe pod <pod-name>
```
Most common cause — environment variable name mismatch between `app.py` and deployment YAML.

### kubeadm join failure
- Token expired → regenerate: `kubeadm token create --print-join-command`
- Network interface binding issue → check `--apiserver-advertise-address` flag

### containerd runtime issues
```bash
sudo systemctl restart containerd
sudo systemctl status containerd
```

### Pods stuck in Pending
```bash
kubectl describe pod <pod-name>
# Check Events section for resource or scheduling issues
```

## Local Development (Docker Compose)

```bash
docker-compose up -d
# Access at http://localhost:5000
