# How to install Kubernetes on Ubuntu 

```
sudo apt-get update && apt-get upgrade
```

## Setup for master and worker nodes

## a) Install Docker (latest version)

### **Setup the repository**

1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:<br>
```
    sudo apt-get install \
      apt-transport-https \
      ca-certificates \
      curl \
      gnupg \
      lsb-release
```
2. Add Docker’s official GPG key:<br>
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

### **Install Docker Engine**

1. Update the apt package index, and install the latest version of Docker Engine and containerd, or go to the next step to install a specific version:<br>
```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

### **Manage Docker as a non-root user**

1. Add your user to the docker group.<br>
```
sudo usermod -aG docker cloud_user
```
2. Restart Docker.<br>
```
sudo systemctl restart docker.service
```

## b) Set up Kubernetes (latest version)

### **Installing kubeadm**

1. Make sure that the `br_netfilter` module is loaded. This can be done by running `lsmod | grep br_netfilter`. To load it explicitly call `sudo modprobe br_netfilter`.<br>
Use
```
lsmod | grep br_netfilter 
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; or try

```
sudo modprobe br_netfilter
```

2. As a requirement for your Linux Node's iptables to correctly see bridged traffic, you should ensure `net.bridge.bridge-nf-call-iptables` is set to 1 in your sysctl config, e.g.<br>
```

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

```
sudo sysctl --system
```

### **Container runtimes**

1. On each of your nodes, install the Docker for your Linux distribution as per Install **Docker Engine**. You can find the latest validated version of Docker in this dependencies file.

2. Configure the Docker daemon, in particular to use systemd for the management of the container’s cgroups.

```
sudo mkdir /etc/docker
```

```
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

3. Restart Docker and enable on boot:

```
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### **Installing kubeadm, kubelet and kubectl**

1. Update the `apt` package index and install packages needed to use the Kubernetes apt repository:

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

2. Download the Google Cloud public signing key:

```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

3. Add the Kubernetes `apt` repository:

```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
4. Update `apt` package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Ensure that kubelet is running if not try
```
systemctl enable kubelet
```

## Master steps

1. Initialize the cluster using the IP range for Flannel - set network model.
```
kubeadm init --pod-network-cidr=10.244.0.0/16
```

2. Copy the `kubeadmn join` command that is in the output. We will need this later.

3. Exit `sudo`, copy the `admin.conf` to your home directory, and take ownership. 

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

4. Deploy Flannel

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

5. Check the cluster state.

```
kubectl get pods --all-namespaces
```

## To add more nodes

1. Run the `join` command (from step **2**), this requires running the command prefaced with `sudo` on the nodes (if we hadn't run `sudo su` to begin with). Then we'll check the nodes from the master.

```
kubectl get nodes
```
