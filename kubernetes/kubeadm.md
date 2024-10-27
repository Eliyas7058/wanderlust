##  Setup Kubernetes [Kubeadm] Cluster (Version: 1.29)

### On both master & worker nodes
- <i>  Become root user </i>
```bash
sudo su
```

- <i>  Updating System Packages </i>
```bash
sudo apt-get update
```

- <i> Create a shell script 1.sh and paste the below code and run it on both nodes:
```bash
#!/bin/bash
# disable swap
sudo swapoff -a

# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

## Install CRIO Runtime
sudo apt-get update -y
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates gpg

sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o

sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service

echo "CRI runtime installed successfully"

# Add Kubernetes APT repository and install required packages
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet="1.29.0-*" kubectl="1.29.0-*" kubeadm="1.29.0-*"
sudo apt-get update -y
sudo apt-get install -y jq

sudo systemctl enable --now kubelet
sudo systemctl start kubelet
```

### On Master node
- <i> Create a shell script 2.sh and paste the below code and run it </i>
```bash
sudo kubeadm config images pull

sudo kubeadm init

mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config


# Network Plugin = calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

kubeadm token create --print-join-command
```

### On Worker node
- <i> Go to master node security group > edit inbound rule and open 6443 port and save changes otherwise you will get error in logs after execurting below join command - request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers) </i>
- <i> Paste the join command you got from the master node and append --v=5 at the end </i>

```bash
<join-command> --v=5
```

### On Both Nodes
- <i> Install docker on both nodes </i>

```bash
sudo apt install docker.io -y
```

```bash

sudo chmod 777 /var/run/docker.sock
```

#### Troubleshooting

- <i> if you install docker before kubernetes installation then it will show below error after ececuting join command on worker node
Found multiple CRI endpoints on the host. Please define which one do you wish to use by setting the 'criSocket' field in the kubeadm configuration file: unix:///var/run/containerd/containerd.sock, unix:///var/run/crio/crio.sock <I>

- <i> To resolve this issue pass -cri-socket /var/run/crio/crio.sock before --v=5 <i>
...

  - <i>sometimes if you are executing kubectl get nodes with ubuntu user you will encounter below error
  The connection to the server localhost:8080 was refused - did you specify the right host or port? <i>

  <i>Reason - KUBECONFIG environment variable not set hence connection refused issue occured <i>

<i> Solution - execute below command with youruser/ubuntu user
mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config <i>
  





