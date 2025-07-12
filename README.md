# Kubernetes Cluster Bootstrap Guide (July 2025)

## Secure Setup with Kubeadm, Firewalld, Flannel, and MetalLB

This guide provides a comprehensive, step-by-step process for bootstrapping a secure and modern Kubernetes cluster. It integrates best practices for `firewalld` configuration using a multi-zone approach, ensuring robust network security from the ground up.

### Cluster Architecture

* **Control Plane:** 1 Node
* **Worker Nodes:** 3 Nodes
* **Operating System:** [Fedora Server](https://fedoraproject.org/server/)
* **Container Runtime:** [cri-o](https://cri-o.io/)
* **CNI (Pod Network):** [flannel-io](https://github.com/flannel-io/flannel)
* **Load Balancer:** [MetalLB (Layer 2 Mode)](https://metallb.io/)

---

## Part 1: Initial Node Preparation (All Nodes)

**Perform these steps on your control plane node and all worker nodes.**

### 1.1 System Update & Hostname Configuration

Ensure all packages are up-to-date. Install `iptables` and `iproute-tc`.

```bash
sudo dnf update -y
sudo dnf install iptables iproute-tc -y
```

Set a unique hostname for each node. This is critical for Kubernetes.

```bash
# On the control plane node:
sudo hostnamectl set-hostname k8s-control-plane

# On worker node 1:
sudo hostnamectl set-hostname k8s-worker-01

# On worker node 2:
sudo hostnamectl set-hostname k8s-worker-02

# On worker node 3:
sudo hostnamectl set-hostname k8s-worker-03
```

(Optional) Next, edit `/etc/hosts` on **ALL** nodes to include the IP address and hostname of every node in the cluster. This ensures reliable name resolution.

```bash
# Example /etc/hosts - EDIT WITH YOUR ACTUAL IPs AND HOSTNAMES
# Add these lines to the /etc/hosts file on every single node.
192.168.122.10 k8s-control-plane
192.168.122.21 k8s-worker-01
192.168.122.22 k8s-worker-02
192.168.122.23 k8s-worker-03
```

### 1.2 Disable Swap

Kubernetes requires that swap be disabled on all nodes.  
These steps are for **Fedora Server** and they may be different for other distributions.

```bash
sudo systemctl stop swap-create@zram0
sudo dnf remove zram-generator-defaults -y
sudo reboot now
```

### 1.3 Configure Kernel Modules and Parameters

We need to load kernel modules required for container networking and ensure IP forwarding is enabled.

Create a configuration file to load modules on boot:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Create a `sysctl` configuration file for Kubernetes networking:

```bash
sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl settings without reboot
sudo sysctl --system
```

---

## Part 2: Secure `firewalld` Configuration

This is a critical step to secure the cluster's host network. We will create a three-zone setup for maximum security and clarity.

### 2.1 Base Configuration (All Nodes)

Perform these steps on the control plane and all worker nodes.

**1. Enable IP Masquerading:** Allows pods to communicate with the outside world.

```bash
sudo firewall-cmd --add-masquerade --permanent
```

**2. Create Custom Zones:** We create dedicated zones for the host network (`k8s-cluster`) and the pod network (`flannel`).

```bash
sudo firewall-cmd --permanent --new-zone=k8s-cluster
sudo firewall-cmd --permanent --new-zone=flannel
```

**3. Assign Network Sources to Zones:** This tells `firewalld` what traffic belongs to which zone.

```bash
# IMPORTANT: Replace 192.168.122.0/24 with YOUR cluster's HOST subnet.
sudo firewall-cmd --permanent --zone=k8s-cluster --add-source=192.168.122.0/24

# NOTE: 10.244.0.0/16 is the default for Flannel.
sudo firewall-cmd --permanent --zone=flannel --add-source=10.244.0.0/16
```

**4. Set `flannel` Zone Target to ACCEPT:** We trust traffic from our internal pod network. Pod-to-pod security should be managed by Kubernetes NetworkPolicies, not the host firewall.

```bash
sudo firewall-cmd --permanent --zone=flannel --set-target=ACCEPT
```

### 2.2 Role-Specific Firewall Rules

Now, apply specific rules based on the node's role.

#### On the Control Plane Node ONLY

```bash
# --- Control Plane Ports (k8s-cluster zone) ---
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=6443/tcp # Kube-API Server

# --- Flannel & MetalLB Ports (k8s-cluster zone) ---
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=8472/udp # Flannel (VXLAN)
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=7946/tcp # MetalLB L2
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=7946/udp # MetalLB L2

# --- External Access (public zone) ---
sudo firewall-cmd --permanent --zone=public --add-service=ssh
```

#### On ALL Worker Nodes ONLY

```bash
# --- Worker & Network Plugins Ports (k8s-cluster zone) ---
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=10250/tcp   # Kubelet API
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=8472/udp    # Flannel (VXLAN)
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=7946/tcp    # MetalLB L2
sudo firewall-cmd --permanent --zone=k8s-cluster --add-port=7946/udp    # MetalLB L2

# --- Optional External Access for NodePort Services (public zone) ---
sudo firewall-cmd --permanent --zone=public --add-port=30000-32767/tcp # NodePort Services
sudo firewall-cmd --permanent --zone=public --add-port=30000-32767/udp # NodePort Services

# --- External Access (public zone) ---
sudo firewall-cmd --permanent --zone=public --add-service=ssh
```

### 2.3 Reload Firewall (All Nodes)

Apply all the firewall changes on every node.

```bash
sudo firewall-cmd --reload
```

---

## Part 3: Install Container Runtime & K8s Tools (All Nodes)

**Perform these steps on your control plane node and all worker nodes.**

### 3.1 Install `cri-o`

We can install `cri-o` directly from Fedora repositories.

```bash
sudo dnf install cri-o containernetworking-plugins -y
```

### 3.2 Start and anable `cri-o`

```bash
sudo systemctl enable --now crio
```

### 3.3 Install Kubernetes Tools

We can install `kubelet`, `kubeadm`, and `kubectl` directly from Fedora repositories.

```bash
sudo dnf install kubernetes kubernetes-kubeadm kubernetes-client -y
```

### 3.4 Start and anable `kubelet`

```bash
sudo systemctl enable --now kubelet
```

---

## Part 4: Bootstrap the Control Plane

**Perform this step on the control plane node ONLY.**

### 4.1 Initialize the Cluster

Run `kubeadm init`. We specify the pod network CIDR that matches the Flannel default and our `firewalld` rule.

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

**CRITICAL:** The output of this command will contain a `kubeadm join` command with a token. **Copy this command and save it somewhere safe.** You will need it to join your worker nodes to the cluster.

### 4.2 Configure `kubectl` Access

After `kubeadm init` succeeds, run these commands to configure `kubectl` access for your regular user.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Part 5: Install the CNI (Flannel)

**Perform this step from your control plane node.**

Apply the Flannel manifest to install the pod network.

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Wait a minute or two, then verify that all system pods are running and the control plane node is `Ready`.

```bash
kubectl get pods -n kube-system
kubectl get pods -n kube-flannel
kubectl get nodes
```

---

## Part 6: Joining Worker Nodes

**Perform these steps on each worker node.**

Use the `kubeadm join` command that you saved from the `kubeadm init` output. It will look something like this:

```bash
# This is an EXAMPLE. Use the command from YOUR `kubeadm init` output.
sudo kubeadm join 192.168.122.10:6443 --token abcdef.1234567890abcdef \
 --discovery-token-ca-cert-hash sha256:1234...cdef
```

After joining each worker, go back to your control plane and verify that the nodes appear and eventually enter the `Ready` state.

```bash
# Run on the control plane
kubectl get nodes -o wide
```

---

## Part 7: Installing the Load Balancer (MetalLB)

**Perform these steps from your control plane node.**

### 7.1 Apply the MetalLB Manifest

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

Wait for the MetalLB pods to be in a `Running` state.

```bash
kubectl get pods -n metallb-system -w
```

### 7.2 Configure MetalLB

Create a configuration file named `metallb-config.yaml`. This tells MetalLB which IP addresses it is allowed to use for `LoadBalancer` services.

**IMPORTANT:** The IP address range you provide here must be on the same subnet as your cluster nodes (`192.168.122.0/24` in our example) and should **NOT** conflict with any existing IPs or DHCP ranges.

```yaml
# metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.122.240-192.168.122.250 # EDIT with a free range on your network
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

Apply the configuration:

```bash
kubectl apply -f metallb-config.yaml
```

---

## Part 8: Verification and Example Deployment

Let's deploy a sample application and expose it to verify the entire setup works.

### 8.1 Deploy NGINX

```bash
# Run on the control plane
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

### 8.2 Check the Service

Find the external IP address assigned by MetalLB.

```bash
kubectl get svc nginx
```

The output will show an `EXTERNAL-IP` (e.g., `192.168.122.240`).

### 8.3 Test Access

From a machine on the same network (but outside the cluster), use `curl` to access the `EXTERNAL-IP` of the NGINX service.

```bash
curl http://192.168.122.240
```

You should see the "Welcome to nginx!" page.

**Congratulations! You have successfully bootstrapped a secure Kubernetes cluster.**
