# Setup a K8s Cluster via Kubeadm
## Kubeadm Setup Prerequisites
Following are the prerequisites for Kubeadm Kubernetes cluster setup.

* Minimum two Ubuntu nodes [One master and one worker node]. You can have more worker nodes as per your requirement.
* The master node should have a minimum of 2 vCPU and 2GB RAM.
* For the worker nodes, a minimum of 1vCPU and 2 GB RAM is recommended.
* 10.X.X.X/X network range with static IPs for master and worker nodes. We will be using the 192 series as the pod network range that will be used by the Calico network plugin. Make sure the Node IP range and pod IP range don’t overlap.

## Kubeadm Port Requirements
Please refer to the following image and make sure all the ports are allowed for the control plane (master) and the worker nodes. If you set up this on a cloud, make sure you allow the ports in the firewall configuration.
![image](https://user-images.githubusercontent.com/81817735/187059532-3991b917-5041-45dc-9333-93f0438c7046.png)
If you are using vagrant-based Ubuntu VMs, the firewall would be disabled by default. So you don’t have to do any firewall configurations.

## Kubernetes Cluster Setup Using Kubeadm
Following are the high-level steps involved in setting up a Kubernetes cluster using kubeadm.

* Install container runtime on all nodes- We will be using Docker.
* Install Kubeadm, Kubelet, and kubectl on all the nodes.
* Initiate Kubeadm control plane configuration on the master node.
* Save the node join command with the token.
* Install the Calico network plugin.
* Join worker node to the master node (control plane) using the join command.
* Validate all cluster components and nodes.
* Install Kubernetes Metrics Server
* Deploy a sample app and validate the app

### Enable iptables Bridged Traffic
Execute the following commands for IPtables to see bridged traffic.
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

### Disable swap on all the Nodes
For kubeadm to work properly, you need to disable swap on all the nodes using the following command.
```
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

The fstab entry will make sure the swap if off on system reboots.  
You can also, control swap errors using the kubeadm parameter --ignore-preflight-errors Swap we will look at it in the latter part.

### Install Docker Container Runtime On All The Nodes
The basic requirement for a Kubernetes cluster is a container runtime. You can have any one of the following container runtimes.
* containerd
* CRI-O
* Docker

We will be using Docker for this setup.
* Install the required packages for Docker.
```
sudo apt-get update -y
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

* Add the Docker GPG key and apt repository.
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

* Install the Docker community edition.
```
sudo apt-get update -y
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```

* Add the docker daemon configurations to use systemd as the cgroup driver.
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

* Start and enable the docker service.
```
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Install Kubeadm & Kubelet & Kubectl on all Nodes
* Install the required dependencies.
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

* Add the GPG key and apt repository.
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

* Update apt and install kubelet, kubeadm, and kubectl.
```
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
```

**Note: If you want to install a specific version of kubernetes, you can use the following commands to find the latest versions.**
```
apt update
apt-cache madison kubeadm | tac
```

* you can specify the version as shown below.
```
sudo apt-get install -y kubelet=1.22.8-00 kubectl=1.22.8-00 kubeadm=1.22.8-00
```

* Add hold to the packages to prevent upgrades.
```
sudo apt-mark hold kubelet kubeadm kubectl
```

Now we have all the required utilities and tools for configuring Kubernetes components using kubeadm.

### Initialize Kubeadm On Master Node To Setup Control Plane
Execute the commands in this section only on the master node.

* First, set two environment variables. Replace 10.0.0.10 with the IP of your master node.
```
IPADDR="192.168.5.123"
NODENAME=$(hostname -s)
```

* Now, initialize the master node control plane configurations using the following kubeadm command.
```
sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=192.168.0.0/16 --node-name $NODENAME --ignore-preflight-errors Swap
```

`--ignore-preflight-errors` Swap is actually not required as we disabled the swap initially. I just added it for the safer side.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```
kubectl get po -n kube-system
```

### Installing a CNI is beyond the scope of this repo, please install a desired CNI.
