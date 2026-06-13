# Kubernetes Two-Node Cluster Setup on DigitalOcean

## Overview

This guide documents the installation of a Kubernetes v1.36.2 cluster on two DigitalOcean droplets using `kubeadm`, with Flannel as the CNI and Traefik as the Ingress Controller.

---

## Infrastructure

| Node | Hostname | Public IP (eth0) | Private IP (eth1) | Role |
|------|----------|-----------------|-------------------|------|
| Master | kube-master | 139.59.34.218 | 10.122.0.2 | control-plane |
| Worker | kube-slave | 168.144.152.53 | 10.122.0.3 | worker |

- **OS:** Ubuntu 24.04 LTS
- **Kubernetes:** v1.36.2
- **Container Runtime:** containerd
- **CNI:** Flannel
- **Ingress:** Traefik v3.7.4

---

## Network Architecture

```
Internet
    │
    ├── 139.59.34.218 (kube-master eth0)
    └── 168.144.152.53 (kube-slave eth0)

Private Network (Cluster Traffic)
    ├── 10.122.0.2 (kube-master eth1)
    └── 10.122.0.3 (kube-slave eth1)

Pod Network (Flannel)
    ├── 10.244.0.0/24 (kube-master pods)
    └── 10.244.1.0/24 (kube-slave pods)
```

---

## Step 1: Prepare Both Nodes

Run on **both nodes**:

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## Step 2: Install containerd (Both Nodes)

```bash
sudo apt-get update
sudo apt-get install -y containerd

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (required for kubeadm)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart and enable
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## Step 3: Install kubeadm, kubelet, kubectl (Both Nodes)

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes v1.36 apt repo
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Pin versions
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet
```

---

## Step 4: Initialize Control Plane (Master Only)

```bash
sudo kubeadm init \
  --apiserver-advertise-address=10.122.0.2 \
  --apiserver-cert-extra-sans=139.59.34.218 \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///run/containerd/containerd.sock
```

> `--apiserver-advertise-address` uses the private IP for cluster traffic.  
> `--apiserver-cert-extra-sans` adds the public IP to the TLS cert for external access.

### Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Step 5: Join Worker Node (Worker Only)

Get the join command from the master:

```bash
# Run on master to generate join command
kubeadm token create --print-join-command
```

Run the output on **kube-slave**:

```bash
sudo kubeadm join 10.122.0.2:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## Step 6: Install Flannel CNI (Master Only)

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Verify nodes are Ready:

```bash
kubectl get nodes
# NAME          STATUS   ROLES           AGE   VERSION
# kube-master   Ready    control-plane   5m    v1.36.2
# kube-slave    Ready    <none>          2m    v1.36.2
```

---

## Step 7: Install Traefik Ingress Controller (Master Only)

### Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Step 1 — Check Control Plane Taint

```bash
kubectl describe node kube-master | grep Taint
```

- If output shows `Taints: <none>` → apply the taint:
```bash
kubectl taint nodes kube-master node-role.kubernetes.io/control-plane:NoSchedule
```

- If output shows `node-role.kubernetes.io/control-plane:NoSchedule` → taint already present, nothing to do.

> **Why keep the taint?** For a 2-node cluster, we keep the taint but give Traefik a **toleration** so it runs on both nodes, while regular app pods stay on the worker only:
> ```
> kube-master  → runs: Traefik ✅  |  app pods ❌
> kube-slave   → runs: Traefik ✅  |  app pods ✅
> ```

### Step 2 — Install Traefik as DaemonSet with hostPort + Toleration

Using DaemonSet mode with `hostPort` binds Traefik directly to port 80/443 on each node — no NodePort or LoadBalancer needed. The toleration allows Traefik to run on the tainted master node.

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set deployment.kind=DaemonSet \
  --set service.type=ClusterIP \
  --set ports.web.hostPort=80 \
  --set ports.websecure.hostPort=443 \
  --set ingressRoute.dashboard.enabled=true \
  --set "tolerations[0].key=node-role.kubernetes.io/control-plane" \
  --set "tolerations[0].effect=NoSchedule" \
  --set "tolerations[0].operator=Exists"
```

### Step 3 — Verify Pods

```bash
kubectl get pods -n traefik -o wide
# Should show one pod on each node (kube-master and kube-slave)
```

### Step 4 — Fix Service Type (LoadBalancer → ClusterIP)

After install, the Traefik Service may show as `LoadBalancer` with `<pending>` external IP. Patch it to `ClusterIP`:

```bash
kubectl patch svc traefik -n traefik -p '{"spec": {"type": "ClusterIP"}}'
```

Verify:

```bash
kubectl get svc -n traefik
# NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
# traefik   ClusterIP   10.110.114.137  <none>        80/TCP,443/TCP   5m
```

> **Why does `<pending>` not matter?** With `hostPort`, traffic flows directly from the node's network interface into the Traefik pod — the Kubernetes Service is completely bypassed for external traffic.

### How hostPort Works

```
Internet :80/:443
    │
    ▼
hostPort on node (bypasses Kubernetes Service entirely)
    │
    ▼
Traefik pod directly
    │
    ▼
Routes to backend services
```

---

## Step 8: Domain Setup (ranjan.host)

### DNS Configuration

Add a wildcard A record in your DNS provider:

```
Type: A
Host: *
Value: 139.59.34.218
TTL:  300
```

This maps `*.ranjan.host` → master's public IP.

Verify DNS propagation:

```bash
ping traefik.ranjan.host
# Should resolve to 139.59.34.218
```

---

## Step 9: Expose Traefik Dashboard via Ingress

> **Note:** Standard Kubernetes `Ingress` cannot use `api@internal` (a Traefik-internal service name).
> A real Kubernetes Service must be created first.

### Create Dashboard Service

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: traefik-dashboard
  namespace: traefik
spec:
  selector:
    app.kubernetes.io/name: traefik
  ports:
  - port: 8080
    targetPort: 8080
EOF
```

### Create Ingress

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-dashboard
  namespace: traefik
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik
  rules:
  - host: traefik.ranjan.host
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: traefik-dashboard
            port:
              number: 8080
EOF
```

### Access Points

| Service | URL |
|---------|-----|
| HTTP traffic | `http://<your-app>.ranjan.host` |
| HTTPS traffic | `https://<your-app>.ranjan.host` |
| Traefik Dashboard | `http://traefik.ranjan.host/dashboard/` (trailing slash required) |

---

## External kubectl Access

To manage the cluster from your local machine:

```bash
# Copy kubeconfig from master
scp root@139.59.34.218:/etc/kubernetes/admin.conf ~/.kube/config

# Point to public IP
sed -i 's/10.122.0.2/139.59.34.218/g' ~/.kube/config

# Test
kubectl get nodes
```

---

## Required Firewall Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 6443 | TCP | Kubernetes API server |
| 2379-2380 | TCP | etcd |
| 10250 | TCP | kubelet API |
| 80 | TCP | Traefik HTTP (hostPort) |
| 443 | TCP | Traefik HTTPS (hostPort) |
| 8472 | UDP | Flannel VXLAN |

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Nodes `NotReady` | CNI not installed — apply Flannel manifest |
| Join token expired | Run `kubeadm token create --print-join-command` on master |
| Reset cluster | `sudo kubeadm reset && sudo rm -rf /etc/cni /etc/kubernetes ~/.kube` |
| Check pod logs | `kubectl logs <pod-name> -n <namespace>` |
| Describe pod | `kubectl describe pod <pod-name> -n <namespace>` |