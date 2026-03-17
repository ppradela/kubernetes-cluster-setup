# Kubernetes Cluster Bootstrap — Production-Ready Setup Guide

> **A complete, security-hardened Kubernetes cluster using kubeadm, CRI-O, Flannel, and MetalLB on Fedora Server. Multi-zone firewalld configuration with host-level network isolation built in from the start.**

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.x-326CE5?logo=kubernetes&logoColor=white)
![OS](https://img.shields.io/badge/OS-Fedora%20Server-51A2DA?logo=fedora&logoColor=white)
![CNI](https://img.shields.io/badge/CNI-Flannel-purple)
![MetalLB](https://img.shields.io/badge/LoadBalancer-MetalLB%20v0.15-blue)
![License](https://img.shields.io/badge/license-MIT-green)

---

## Features

- **kubeadm bootstrap** — production-grade control plane init with pod network CIDR pre-configured for Flannel
- **CRI-O container runtime** — lightweight, Kubernetes-native runtime without Docker dependency
- **Flannel CNI** — simple and reliable pod networking with VXLAN backend (`10.244.0.0/16`)
- **MetalLB Layer 2 mode** — bare-metal load balancer assigning real IPs from a local address pool
- **Multi-zone firewalld** — three dedicated zones (`public`, `k8s-cluster`, `flannel`) with least-privilege rules per role
- **Ready-to-apply manifests** — MetalLB config and example deployment included in `manifests/`

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                      │
│                                                          │
│  ┌──────────────────────┐                               │
│  │   Control Plane      │  192.168.122.x                │
│  │   k8s-control-plane  │  Port 6443 (API Server)       │
│  └──────────┬───────────┘                               │
│             │                                            │
│    ┌────────┼────────┐                                  │
│    ▼        ▼        ▼                                   │
│  ┌──────┐ ┌──────┐ ┌──────┐                            │
│  │ w-01 │ │ w-02 │ │ w-03 │  Workers — 192.168.122.x   │
│  └──────┘ └──────┘ └──────┘                            │
│                                                          │
│  Pod Network:    10.244.0.0/16  (Flannel VXLAN)         │
│  LB IP Pool:     192.168.122.240-192.168.122.250        │
└─────────────────────────────────────────────────────────┘
```

| Component | Choice | Reason |
|---|---|---|
| OS | Fedora Server | Modern kernel, dnf packaging, SELinux ready |
| Runtime | CRI-O | Native CRI, no dockershim overhead |
| CNI | Flannel | Simple VXLAN overlay, Flannel CIDR matches kubeadm default |
| Load Balancer | MetalLB L2 | No BGP router required — works on flat home/lab networks |
| Firewall | firewalld (multi-zone) | Per-role rules, masquerade, pod traffic accepted in flannel zone |

---

## Step 1 — Initial Node Preparation

Perform these steps on **all nodes**.

### Update system and install dependencies

```bash
sudo dnf update -y
sudo dnf install iptables iproute-tc -y
```

### Set hostnames

```bash
sudo hostnamectl set-hostname k8s-control-plane  # control plane node
sudo hostnamectl set-hostname k8s-worker-01       # worker 1
sudo hostnamectl set-hostname k8s-worker-02       # worker 2
sudo hostnamectl set-hostname k8s-worker-03       # worker 3
```

Add all nodes to `/etc/hosts` on every machine:

```
192.168.122.10  k8s-control-plane
192.168.122.11  k8s-worker-01
192.168.122.12  k8s-worker-02
192.168.122.13  k8s-worker-03
```

### Disable swap

Kubernetes requires swap to be off. On Fedora, `zram` is the default swap device:

```bash
sudo systemctl stop swap-create@zram0
sudo dnf remove zram-generator-defaults -y
sudo reboot now
```

### Load kernel modules and apply sysctl

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
# → Applying /etc/sysctl.d/k8s.conf ...
```

---

## Step 2 — Firewalld Configuration

Three zones are used to isolate traffic by type and role:

| Zone | Source | Purpose |
|---|---|---|
| `public` | default | SSH, NodePort services |
| `k8s-cluster` | `192.168.122.0/24` | API server, kubelet, MetalLB speaker |
| `flannel` | `10.244.0.0/16` | Pod-to-pod traffic — fully accepted |

### Base setup — all nodes

```bash
sudo firewall-cmd --add-masquerade --permanent
sudo firewall-cmd --permanent --new-zone=k8s-cluster
sudo firewall-cmd --permanent --new-zone=flannel

sudo firewall-cmd --permanent --zone=k8s-cluster --add-source=192.168.122.0/24
sudo firewall-cmd --permanent --zone=flannel --add-source=10.244.0.0/16
sudo firewall-cmd --permanent --zone=flannel --set-target=ACCEPT
```

### Control plane rules

```bash
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=6443/tcp    # API Server
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=8472/udp    # Flannel VXLAN
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=7946/tcp    # MetalLB
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=7946/udp    # MetalLB
sudo firewall-cmd --permanent --zone=public --add-service=ssh
```

### Worker node rules

```bash
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=10250/tcp   # kubelet
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=8472/udp    # Flannel VXLAN
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=7946/tcp    # MetalLB
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=7946/udp    # MetalLB
sudo firewall-cmd --permanent --zone=public --add-port=30000-32767/tcp  # NodePort range
sudo firewall-cmd --permanent --zone=public --add-port=30000-32767/udp  # NodePort range
sudo firewall-cmd --permanent --zone=public --add-service=ssh
```

### Reload

```bash
sudo firewall-cmd --reload
```

---

## Step 3 — Install Container Runtime & Kubernetes Tools

Perform these steps on **all nodes**.

### CRI-O

```bash
sudo dnf install cri-o containernetworking-plugins -y
sudo systemctl enable --now crio
# → Created symlink /etc/systemd/system/crio.service
```

### Kubernetes tools

```bash
sudo dnf install kubernetes kubernetes-kubeadm kubernetes-client -y
sudo systemctl enable --now kubelet
```

---

## Step 4 — Bootstrap the Control Plane

Run on the **control plane node only**.

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# → Your Kubernetes control-plane has initialized successfully!
```

> **Note:** Save the `kubeadm join` command printed at the end — you will need it in Step 6.

Configure `kubectl` for your user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Step 5 — Install the CNI (Flannel)

Run on the **control plane node**.

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
# → namespace/kube-flannel created
# → daemonset.apps/kube-flannel-ds created
```

Verify pods are running:

```bash
kubectl get pods -n kube-system
kubectl get nodes
# → NAME                 STATUS   ROLES           AGE   VERSION
# → k8s-control-plane   Ready    control-plane   1m    v1.x
```

---

## Step 6 — Join Worker Nodes

Run on **each worker node** using the join command from Step 4:

```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

> **Note:** If the token has expired (default TTL is 24h), generate a new one on the control plane: `kubeadm token create --print-join-command`

Verify all nodes are joined from the control plane:

```bash
kubectl get nodes -o wide
# → NAME                STATUS   ROLES           AGE   VERSION   INTERNAL-IP
# → k8s-control-plane  Ready    control-plane   5m    v1.x      192.168.122.10
# → k8s-worker-01      Ready    <none>          2m    v1.x      192.168.122.11
# → k8s-worker-02      Ready    <none>          2m    v1.x      192.168.122.12
# → k8s-worker-03      Ready    <none>          1m    v1.x      192.168.122.13
```

---

## Step 7 — Install the Load Balancer (MetalLB)

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

Wait for MetalLB pods to be ready:

```bash
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

Apply the IP pool configuration from this repository:

```bash
kubectl apply -f manifests/metallb-config.yaml
# → ipaddresspool.metallb.io/first-pool created
# → l2advertisement.metallb.io/l2-advert created
```

The pool `192.168.122.240–192.168.122.250` will be assigned to `LoadBalancer` type services.

---

## Step 8 — Verification & Example Deployment

Deploy the example NGINX app from this repository:

```bash
kubectl apply -f manifests/nginx-example.yaml
# → deployment.apps/nginx created
# → service/nginx created
```

Watch until the external IP is assigned:

```bash
kubectl get svc nginx --watch
# → NAME    TYPE           CLUSTER-IP      EXTERNAL-IP        PORT(S)        AGE
# → nginx   LoadBalancer   10.96.x.x       192.168.122.240    80:3xxxx/TCP   30s
```

Test the endpoint:

```bash
curl http://192.168.122.240
# → <!DOCTYPE html>
# → <html><head><title>Welcome to nginx!</title>...
```

---

## Troubleshooting

### Nodes stuck in NotReady

```bash
kubectl describe node <node-name>
# → check for CNI not initialized — ensure Flannel pods are Running
kubectl get pods -n kube-flannel
```

### Pods can't reach each other across nodes

Verify that the `flannel` firewalld zone is accepting pod CIDR traffic:

```bash
sudo firewall-cmd --zone=flannel --list-all
# → flannel (active)
# →   target: ACCEPT
# →   sources: 10.244.0.0/16
```

### MetalLB not assigning IPs

```bash
kubectl logs -n metallb-system -l app=metallb,component=speaker
# → look for "ARM not configured" — means metallb-config.yaml was not applied
kubectl apply -f manifests/metallb-config.yaml
```

### kubeadm join token expired

```bash
# On the control plane — generate a new join command
kubeadm token create --print-join-command
```

---

## License

MIT

---

## Author

**Przemyslaw Pradela** — built and validated on a real home-lab cluster running Fedora Server.

[![GitHub](https://img.shields.io/badge/GitHub-ppradela-181717?logo=github)](https://github.com/ppradela)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-przemyslaw--pradela-0A66C2?logo=linkedin)](https://www.linkedin.com/in/przemyslaw-pradela)

Contributions and issue reports welcome.
