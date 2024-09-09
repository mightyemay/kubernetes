# kubernetes
Kubernetes cluster on Ubuntu 24
Here’s a concise, step-by-step instruction for installing a Kubernetes cluster using best practices, based on the configuration you've used and the repository setup for Kubernetes v1.31.

---

## Kubernetes Cluster Setup on Ubuntu 24.04.1 LTS

### Nodes:
- **Master Node**: `195.117.36.22` (`master1.k8cluster.eu`, hostname: `master1`)
- **Worker Node**: `195.117.36.58` (`worker1.k8cluster.eu`, hostname: `worker1`)

### Prerequisites:

1. **Update System**:
   Ensure your system is up to date:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Set Hostnames**:
   Configure the hostname for each node:
   - On **Master Node**:
     ```bash
     sudo hostnamectl set-hostname master1
     ```
   - On **Worker Node**:
     ```bash
     sudo hostnamectl set-hostname worker1
     ```

3. **Disable Swap**:
   Kubernetes requires swap to be disabled:
   ```bash
   sudo swapoff -a
   sudo sed -i '/swap/d' /etc/fstab
   ```

4. **Install `chrony` for time synchronization** (on both nodes):
   ```bash
   sudo apt update
   sudo apt install chrony -y
   sudo systemctl enable chrony
   sudo systemctl start chrony
   ```

### Step 1: Install Docker and containerd on All Nodes

1. **Install Docker** (or containerd):
   ```bash
   sudo apt update
   sudo apt install -y docker.io
   ```

2. **Enable and Start Docker**:
   ```bash
   sudo systemctl enable docker
   sudo systemctl start docker
   ```

3. **Configure containerd**:
   If `containerd` is being used, configure it properly:
   ```bash
   sudo mkdir -p /etc/containerd
   sudo containerd config default | sudo tee /etc/containerd/config.toml
   sudo nano /etc/containerd/config.toml
   ```

   Make sure to set:
   ```toml
   SystemdCgroup = true
   ```

   Then restart containerd:
   ```bash
   sudo systemctl restart containerd
   sudo systemctl enable containerd
   ```

### Step 2: Add Kubernetes Repository and Install Components

#### Repository Setup:
Use the Kubernetes repository you provided:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
```

#### Install Kubernetes components:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 3: Initialize the Master Node

On the **Master Node**:

1. **Initialize Kubernetes control plane**:
   ```bash
   sudo kubeadm init --control-plane-endpoint "master1.k8cluster.eu:6443" --pod-network-cidr=192.168.0.0/16
   ```

   - Save the `kubeadm join` command generated (to be used for worker nodes).

2. **Configure `kubectl` for Master Node**:
   Set up `kubectl` for the current user on the master node:
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

### Step 4: Install Network Addon (Calico)

Install the Calico network plugin on the master node:

```bash
kubectl apply -f https://docs.projectcalico.org/v3.25/manifests/calico.yaml
```

### Step 5: Join Worker Nodes to the Cluster

On the **Worker Node**:

1. **Run the `kubeadm join` command** received from the master node after initialization:
   ```bash
   sudo kubeadm join master1.k8cluster.eu:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```

2. **Verify that the Worker Node has joined**:
   On the **Master Node**, check the status of the nodes:
   ```bash
   kubectl get nodes
   ```

### Step 6: Label Worker Node

To clarify the role of the worker node, label it:

```bash
kubectl label node worker1 node-role.kubernetes.io/worker=worker
```

### Summary:
You have now successfully set up a working Kubernetes cluster on Ubuntu 24.04.1 LTS, using the provided repository and installed the necessary components. The cluster includes one master node (`master1`) and one worker node (`worker1`), and it is configured to use Calico for networking.

