# Private Kubernetes Cluster Setup on Ubuntu 20.04

This README provides a step-by-step guide to install and configure a Kubernetes cluster on Ubuntu 20.04. By the end, you'll have a functional Kubernetes environment capable of running and managing containerized applications. 

This setup is for a private Kubernetes cluster, which can be deployed on-premises or on any cloud platform as a self-hosted cluster. can you change the title accordingly

---

## Table of Contents
1. Prerequisites
2. Lab Setup
3. Step-by-Step Installation
   - [1. Set Hostnames of Each Node](#1-set-hostnames-of-each-node)
   - [2. Disable Swap and Add Kernel Modules](#2-disable-swap-and-add-kernel-modules)
   - [3. Install Containerd Runtime](#3-install-containerd-runtime)
   - [4. Install Kubectl, Kubelet, and Kubeadm](#4-install-kubectl-kubelet-and-kubeadm)
   - [5. Install Kubernetes Cluster](#5-install-kubernetes-cluster)
   - [6. Add Worker Nodes to Cluster](#6-add-worker-nodes-to-cluster)
   - [7. Deploy Calico Pod Network Add-on](#7-deploy-calico-pod-network-add-on)
   - [8. Test and Verify Kubernetes Installation](#8-test-and-verify-kubernetes-installation)
4. Conclusion

---

## Prerequisites
Ensure the following requirements are met before proceeding:
- **OS:** Ubuntu 20.04 (clean installation).
- **Hardware:** Minimum of 2 CPU cores, 2 GB of RAM, and sufficient disk space per node.
- **Network:** Static IP addresses assigned to all nodes.
- **Access:** SSH access to all nodes from the installer machine.
- **Permissions:** Privileged user with `sudo` rights.
- **Internet:** Stable connection.

---

![Kubernetes Cluster Architecture](https://github.com/agentnovax/agentnovax-devzone/raw/main/kubernetes/assets/AgentNovaX-Kubernetes.png)

---

## Lab Setup
The cluster will consist of one control-plane node and two worker nodes:

| Node Name             | Role                | IP Address      |
|-----------------------|---------------------|-----------------|
| `k8s-master-node`    | Control Plane Node  | 192.168.1.40    |
| `k8s-worker-node-01` | Worker Node         | 192.168.1.41    |
| `k8s-worker-node-02` | Worker Node         | 192.168.1.42    |

---

## Step-by-Step Installation

### 1. Set Hostnames of Each Node
Set the hostname on each node using the `hostnamectl` command.

**Commands:**
```bash
# On the control plane node:
sudo hostnamectl set-hostname "k8s-master-node"

# On worker node 01:
sudo hostnamectl set-hostname "k8s-worker-node-01"

# On worker node 02:
sudo hostnamectl set-hostname "k8s-worker-node-02"
```

Update the `/etc/hosts` file on **all nodes** with the following entries:
```
192.168.1.40    k8s-master-node
192.168.1.41    k8s-worker-node-01
192.168.1.42    k8s-worker-node-02
```

**Verification:**
```bash
hostnamectl
# Output on the control-plane node:
Static hostname: k8s-master-node
```

---

### 2. Disable Swap and Add Kernel Modules

**Disable Swap:**
1. Open `/etc/fstab` and comment out any lines with swap entries.
   ```bash
   sudo vi /etc/fstab
   ```
2. Disable swap immediately:
   ```bash
   sudo swapoff -a
   ```

**Add Kernel Modules:**
1. Load required modules:
   ```bash
   sudo tee /etc/modules-load.d/containerd.conf <<EOF
   overlay
   br_netfilter
   EOF

   sudo modprobe overlay
   sudo modprobe br_netfilter
   ```
2. Configure sysctl parameters:
   ```bash
   sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   EOF

   sudo sysctl --system
   ```

**Verification:**
```bash
sysctl net.bridge.bridge-nf-call-iptables
# Output:
net.bridge.bridge-nf-call-iptables = 1
```

---

### 3. Install Containerd Runtime

Install and configure `containerd` on **all nodes**:

**Commands:**
```bash
# Install dependencies:
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# Add Docker repository:
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Install containerd:
sudo apt update
sudo apt install -y containerd.io

# Configure containerd:
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Start and enable service:
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Verification:**
```bash
sudo systemctl status containerd
# Output:
● containerd.service - containerd container runtime
   Loaded: loaded (/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
   Active: active (running)
```

---

### 4. Install Kubectl, Kubelet, and Kubeadm

Install Kubernetes tools on **all nodes**:

**Commands:**
```bash
# Add Kubernetes repository:
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install tools:
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet service:
sudo systemctl enable --now kubelet
```

**Verification:**
```bash
kubelet --version
# Output:
kubelet version v1.32
```

---

### 5. Install Kubernetes Cluster
On the control plane node (`k8s-master-node`):

**Commands:**
```bash
sudo kubeadm init --control-plane-endpoint=k8s-master-node
```

**Save the `kubeadm join` command from the output for adding worker nodes.**

Post-installation setup for kubectl:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### 6. Add Worker Nodes to Cluster

Log in to each worker node and run the `kubeadm join` command saved from Step 5:

Example:
```bash
sudo kubeadm join k8s-master-node:6443 --token <token> \
--discovery-token-ca-cert-hash sha256:<hash>
```

**Verification:**
```bash
kubectl get nodes
# Output:
NAME                  STATUS   ROLES                  AGE   VERSION
k8s-master-node       Ready    control-plane,master   1m    v1.32
k8s-worker-node-01    Ready    <none>                30s   v1.32
k8s-worker-node-02    Ready    <none>                30s   v1.32
```

---

### 7. Deploy Calico Pod Network Add-on

On the control plane node:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

**Verification:**
```bash
kubectl get pods --all-namespaces
```

---

### 8. Test and Verify Kubernetes Installation

Deploy an Nginx deployment:
```bash
kubectl create deployment nginx-web --image=nginx
```

Scale the deployment:
```bash
kubectl scale --replicas=4 deployment nginx-web
```

Expose a pod via NodePort:
```bash
kubectl run http-web --image=httpd --port=80
kubectl expose pod http-web --name=http-service --port=80 --type=NodePort
```

Verify NodePort:
```bash
kubectl get service http-service
```

Access the application using the node IP and NodePort.

---

## Conclusion
Congratulations! You’ve successfully set up a Kubernetes cluster on Ubuntu 20.04. Your cluster is now ready to deploy and manage containerized applications. Explore further Kubernetes features to unleash its full potential.

---

## Contact 📧

For any queries, feel free to reach out via  
![wecare@agentnovax.com](https://raw.githubusercontent.com/agentnovax/www.agentnovax.com/main/assets/icons/agentnovax-email.png)

---

- ## 📬 Stay Connected

- Website: [AgentNovaX Platform](https://agentnovax.com)  
- Social Media:
  - [YouTube](https://www.youtube.com/@agentnovaxp)
  - [Instagram](https://www.instagram.com/agentnovax/)
  - [X (formerly Twitter)](https://x.com/agentnovax)
  - [Facebook](https://www.facebook.com/profile.php?id=61571252049491)
  - [GitHub](https://github.com/agentnovax/)
  - [LinkedIn](https://www.linkedin.com/company/agentnovax/)
  - [Discord](https://discord.gg/GDPE6YSJ)

---

## NovaLeaf Concept 🌱

**NovaLeaf** is an initiative focused on environmental sustainability, aiming to contribute to a greener planet. Through this initiative, **AgentNovaX** is committed to planting trees, fostering green projects, and encouraging eco-friendly practices among individuals and communities.

- **Plant a Tree, Empower a Community**: For every milestone achieved in our platform, a tree will be planted in a designated area.
- **Green Nova Trees**: These trees represent our growth and commitment to sustainability, and each one is named for the cause it supports.
- **Join the Movement**: Become part of the **NovaLeaf** family and help us plant the future, one tree at a time. 🌳

🌟 **Please consider starring this repository to support the NovaLeaf initiative** 🌟

For more information, visit [NovaLeaf](https://novaleaf.agentnovax.com).

---

## DataFlux 🔄

**DataFlux** provides free tools for data conversion, JSON/YAML beautification, and validation to help developers and data enthusiasts streamline their workflow.

- Tools available: JSON/YAML Beautifiers and Validators, JSON/YAML conversion, Text Compare, JavaScript Validators, and more.
- Visit [DataFlux](https://dataflux.agentnovax.com) to explore our tools and enhance your productivity.

---

## Disclaimer

The **AgentNovaX Development Resources Repository** is provided "as-is," without warranty of any kind, express or implied. AgentNovaX does not guarantee the accuracy or completeness of the content in this repository. By accessing and using these resources, you agree that AgentNovaX is not liable for any loss, damage, or legal issues arising from the use of these resources. Users are encouraged to review and validate the content independently before applying it to their own projects.

---

