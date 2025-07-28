# kubernetes-installation
# Kubernetes Installation Guide on Fedora (K8s v1.33.x)
Kubernetes Installation Guide on Fedora (K8s v1.33.x)
________________________________________
1. Prepare Both Master and Worker Nodes

sudo dnf check-update && sudo dnf install -y dnf-plugins-core curl
sudo -i
Install basic utilities and get root shell.
________________________________________
2. Add Google GPG Keys for Kubernetes Packages

sudo rpm --import https://packages.cloud.google.com/yum/doc/yum-key.gpg
________________________________________
3. Add Kubernetes Repository (v1.33)
bash
CopyEdit
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/repodata/repomd.xml.key
EOF
________________________________________
4. Set SELinux to Permissive Mode

sudo setenforce 0                       # Temporarily set permissive
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config  # Permanent change
________________________________________
5. Disable Swap (required for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
________________________________________
6. Enable Kernel Modules and Sysctl Settings

# Load br_netfilter module immediately
sudo modprobe br_netfilter

# Load at boot
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf

# Verify module is loaded
lsmod | grep br_netfilter

# Configure sysctl for Kubernetes networking
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl settings
sudo sysctl --system
________________________________________
7. Remove Docker Repo if Present (Avoid conflicts)

sudo dnf config-manager --disable docker-ce-stable || true
sudo rm -f /etc/yum.repos.d/docker-ce.repo || true
sudo dnf clean all
________________________________________
8. Install and Configure containerd

# Install containerd
sudo dnf install -y containerd

# Create config directory
sudo mkdir -p /etc/containerd

# Generate default config
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# Set systemd cgroup driver (required by Kubernetes)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Enable and start containerd
sudo systemctl enable --now containerd

# Verify containerd status
sudo systemctl status containerd
________________________________________
9. Install Kubernetes Tools (kubeadm, kubelet, kubectl)

sudo dnf install -y kubelet kubeadm kubectl
________________________________________
10. Enable and Start kubelet Service

sudo systemctl daemon-reexec
sudo systemctl enable --now kubelet
________________________________________
Master Node Specific Steps
________________________________________
11. Initialize Kubernetes Master Node
bash
CopyEdit
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
•	This initializes the control plane.
•	Use 192.168.0.0/16 if planning to use Calico (adjust if using Flannel or others).
________________________________________
12. Configure kubectl for Your User (Master Only)

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
________________________________________
13. Deploy Pod Network (Calico Recommended)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
This sets up networking between pods.
________________________________________
Worker Node Specific Steps
________________________________________
14. Join Worker Nodes to the Cluster
After the master node is initialized, it will print a kubeadm join command, for example:

kubeadm join 172.31.5.96:6443 --token f5au9h.7tgukvfbasvf6ij2 --discovery-token-ca-cert-hash sha256:c7f8a95dd92cbbda9ab8128208dd72415d67addbf93a876aaafff81739179762
Run this command on each worker node to join them to the cluster.
If you lost the command, get it from master:
kubeadm token create --print-join-command

