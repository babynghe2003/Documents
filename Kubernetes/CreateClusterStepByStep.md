# Step by step create K8S cluster by Kubeadm and Containerd

> By Minh Phi Ho

## System Info

I'll build cluster on 5 server with 1 master node, 3 worker nodes and 1 NFS server for <b>Storage Class</b> of this cluster

| IP Address   | Node Type          | OS                 | RAM  | SSD |
| ------------ | ------------------ | ------------------ | ---- | --- |
| 10.39.61.62  | Control plane node | Linux Ubuntu 18.04 | 64GB | 5TB |
| 10.39.61.58  | Worker node        | Linux Ubuntu 18.04 | 64GB | 5TB |
| 10.39.61.12  | Worker node        | Linux Ubuntu 18.04 | 64GB | 5TB |
| 10.39.61.218 | Worker node        | Linux Ubuntu 18.04 | 64GB | 5TB |
| 10.39.61.142 | NFS Server         | Linux Ubuntu 18.04 | 64GB | 5TB |

## 1. Set up a proxy for the server (if the server requires a proxy to access the internet).

All my servers has server proxy <b>http://10.39.152.30:3128/</b>

### 1.1 Setup proxy for APT

Create file <b>/etc/apt/apt.conf</b> and write 2 lines below

```shell
Acquire::http::Proxy "http://10.39.152.30:3128";
Acquire::https::Proxy "http://10.39.152.30:3128";
```

After that you can <b>sudo apt update</b> to update your apt

### 1.2 Setup proxy for Git

Setup git proxy for pull repo or download source from github.com

```bash
git config --global http.proxy http://10.39.152.30:3128/
git config --global https.proxy http://10.39.152.30:3128/
```

### 1.3 Setup proxy for access the internet

This variable is typically available to the current process and its child processes. So you need to set again in other processes

```bash
export HTTPS_PROXY=http://10.39.152.30:3128/
export HTTP_PROXY=http://10.39.152.30:3128/
```

## 2. Before you begin

Follow [Install Kubeadm Tutorial](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), you need to setup some things before you begin.

```shell
#disable swap
sudo swapoff -a

# keeps the swaf off during reboot
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
sudo apt-get update -y

# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

#Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

You must be create cluster in <b>root</b> user, run <b>sudo su</b> before run any command in below step

## 3. Install Containerd

Run in all node

```zsh
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
 "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
 $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
 sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install containerd.io

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

mkdir -p /etc/systemd/system/containerd.service.d/
cat >  /etc/systemd/system/containerd.service.d/http-proxy.conf << EOF
[Service]
Environment="HTTP_PROXY=http://10.39.152.30:3128/"
Environment="HTTPS_PROXY=http://10.39.152.30:3128/"
Environment="NO_PROXY=localhost"
EOF

sudo systemctl daemon-reload
sudo systemctl enable containerd --now

echo "Containerd installed susccessfully"

```

## 4. Install Kubeadm, kubelet, kubectl

Run in all nodes

```zsh
sudo apt-get update -y
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

echo "Install kubenetes successfully"
```

## 5. Install Helm

I will use Helm for install charts more eassily, more info [Helm Chart](https://helm.sh/docs/)

```zsh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## 6. Init cluster

On control plane node, i pull config images first for smooth init cluster by:

> NOTE: make sure you export Proxy variable on above step first

```zsh
sudo kubeadm config images pull
```

After pull successfully, you need to unset Proxy variable before init by <b>unset</b>, the proxy variable may cause errors when initializing a cluster using kubeadm.

```zsh
sudo kubeadm init
```

The result of this command like this:

```
Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a Pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    /docs/concepts/cluster-administration/addons/

    You can now join any number of machines by running the following on each node
    as root:

    kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Base on result, save <b>kubeadm join command</b> and run these command for apply config for <b>kubectl</b> and <b>helm</b>

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Switch to other worker nodes and run the join command with token in above

```zsh
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

> NOTE: token in <b>join</b> command will expire for after 24 hours, you can get new token by using <b>sudo kubeadm token create --print-join-command</b> command

After join node in cluster, you can using this command for verify

```shell
sudo kubectl get nodes -o wide
```

If hostname of server appear in result, it mean it join successfully

> NOTE: After joining the node, the status will be <b>NotReady</b> because containerd does not have CNI yet. You need to continue with the steps below to make it <b>Ready</b>.

## 6. Setup CNI by Cilium and Helm

I install [Cilium](https://cilium.io/) for CNI of this cluster, you can use [Flannel](https://github.com/flannel-io/cni-plugin) for [Calico](https://docs.tigera.io/calico/latest/about/) instead

```zsh
sudo helm repo add cilium https://helm.cilium.io/
sudo helm install cilium cilium/cilium --namespace kube-system
```

After install CNI, restart containerd and kubelet in all worker to apply CNI

```zsh
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

> NOTE: If status some worker node still NotReady, you need to view logs of that node by command
>
> ```zsh
> sudo kubectl describe node <failed-node-name>
> ```
>
> If status is about CNI init failed, you can fix it by stop <b>apparmor</b> service and restart it
>
> ```zsh
> systemctl stop apparmor
> systemctl disable apparmor
> systemctl restart containerd
> systemctl restart kubelet
> ```
>
> You can get nodes again and status will be Ready

## 7. Setup NFS Server

In NFS Server Node, install <b>nfs kernel server</b> by APT

```shell
sudo apt update
sudo apt install nfs-kernel-server
```

Setup export for nfs

```shell
sudo vi /etc/exports

# add this line
/export/nfs-share *(rw,sync,no_subtree_check)
```

Then export and restart nfs service

```shell
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

## 8. Setup Storage class by nfs-subdir-external-provisioner

Pull the nfs-subdir-external-provisioner in

```shell
git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git
cd nfs-subdir-external-provisioner/charts/nfs-subdir-external-provisioner/
```

Modify this bellow variable

```yaml
nfs.server: 10.39.61.142
nfs.path: /export/nfs-share
storageClass.create: true
storageClass.defaultClass: true
storageClass.name: nfs-client
storageClass.allowVolumeExpansion: true
```

Save it and run this command for install Chart

```shell
sudo helm install nfs-subdir-external-provisioner .
```

After install nfs-subdir-external-provisioner, you can use <b>test-claim.yaml</b> and <b>test-pod.yaml</b> in <b>nfs-subdir-external-provisioner/deploy</b> dir to test storage class
