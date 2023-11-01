
# ⚓ Install Kubernetes 1.27 on Red Hat Enterprise Linux (RHEL 8) 

>  OS &emsp;&emsp;&emsp;&ensp;         Red Hat Enterprise Linux release 8.8 (Ootpa)
>  <br/> Revison &emsp;&nbsp;          1.0.0

&nbsp;
### **Contents**
- [External Prerequisites](#external-prerequisites)
- [Pre-installation Configurations](#pre-installation-configurations-for-all-nodes)
    1. [Install NFS Utils](#1-install-nfs-utils)
    2. [Open Firewall Ports](#2-open-firewall-ports)
    3. [Disable SELinux](#3-disable-selinux)
    4. [Turn-off the Swap](#4-turn-off-the-swap)
    5. [Configure iptables to see bridged traffic](#5-configure-iptables-to-see-bridged-traffic)
    6. [Install CRI-O](#6-install-cri-o)
    7. [Install Kubernetes](#7-install-kubernetes)
- [Initialize the Cluster (Master Nodes)](#initialize-the-cluster-master-nodes)
    1. [Initialize Kubernetes HA](#1-initialize-kubernetes--high-availability-with-haproxy)
    2. [Configure Kubernetes Auto-Completion](#2-configure-kubernetes-auto-completion)
- [Install Essentials Apps for Kubernetes Cluster](#install-essentials-apps-for-kubernetes-cluster)
    1. [Install Flannel CNI](#1-install-flannel-cni)
    2. [Install Metal-LB](#2-install-metal-lb)
    3. [Install Ingress-Nginx](#3-install-ingress-nginx)
    4. [Install Helm](#4-install-helm)
    5. [Install NFS Subdir External Provisioner](#5-install-nfs-subdir-external-provisioner)
- [Install Additional Apps](#install-additional-apps)
    1. [Install Kubernetes Dashboard](#1-install-kubernetes-dashboard)
    2. [Install Kubernetes Metrics Server](#2-install-kubernetes-metrics-server)
<br />

#
## [# External Prerequisites](#contents)
<br/>

> - NFS Server IP & Path
> - HAProxy Server
> - (Optional) Proxy Server IP
&nbsp;

#
## [# Pre-installation Configurations (For All Nodes)](#contents)
<br/>

> - Check Internet Connectivity or Configure Proxy IP
> - Check Local RHEL Repo
> - Modify hosts file


&nbsp;
### [1. Install NFS Utils](#contents)
``` bash
sudo yum install -y nfs-utils
```

&nbsp;
### [2. Open Firewall Ports](#contents)
``` bash
sudo firewall-cmd --zone=public --add-service=kube-apiserver --permanent
sudo firewall-cmd --zone=public --add-service=etcd-client --permanent
sudo firewall-cmd --zone=public --add-service=etcd-server --permanent
# kubelet API
sudo firewall-cmd --zone=public --add-port=10250/tcp --permanent
# kube-scheduler
sudo firewall-cmd --zone=public --add-port=10251/tcp --permanent
# kube-controller-manager
sudo firewall-cmd --zone=public --add-port=10252/tcp --permanent
# NodePort Services
sudo firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent
# apply changes
sudo firewall-cmd --reload
```


&nbsp;
### [3. Disable SELinux](#contents)
``` bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

&nbsp;
### [4. Turn-off the Swap](#contents)
``` bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

&nbsp;
### [5. Configure iptables to see bridged traffic](#contents)
``` bash
# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-arptables = 1
EOF

# Apply sysctl params
sudo sysctl --system
```

&nbsp;
### [6. Install CRI-O](#contents)
``` bash
export OS="CentOS_8"
export VERSION="1.27"
export FULL_VERSION="1.27.1"

# Add CRI-O Repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION:/$FULL_VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION:$FULL_VERSION.repo

# Install & Start CRI-O
sudo yum install cri-o -y
sudo systemctl enable --now crio
```

&nbsp;
### [7. Install Kubernetes](#contents)
``` bash
# Add Kubernetes yum repository
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Install Kublet, Kubeadm, Kubectl Packages
sudo yum install -y kubelet-1.27.5-0 kubeadm-1.27.5-0 kubectl-1.27.5-0 --disableexcludes=kubernetes

# Enable and Start kubelet service
sudo systemctl enable --now kubelet
```

<br />

## [# Initialize the Cluster (Master Nodes)](#contents)
#

&nbsp;
### [1. Initialize Kubernetes | High Availability with HAProxy](#contents)
``` bash
sudo kubeadm config images pull
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --upload-certs \
--control-plane-endpoint="<LOAD_BALANCER_IP>:6443" --apiserver-advertise-address=<MASTER1_IP>
```

&nbsp;
### [2. Configure Kubernetes Auto-Completion](#contents)
``` bash
# Bash AutoCompletion
sudo yum install bash-completion -y
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl

# Kubectl AutoCompletion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source <(kubectl completion bash)
```


#
## [# Install Essentials Apps for Kubernetes Cluster](#contents)
<br />

&nbsp;
### [1. Install Flannel CNI](#contents)
``` bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```


&nbsp;
### [2. Install Metal-LB](#contents)
``` bash
# Deploy MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.11/config/manifests/metallb-native.yaml

# Create IPAdressPool
cat << EOF | kubectl apply -f - 
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pool-1
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.84-10.0.0.90
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - pool-1
EOF
```

&nbsp;
### [3. Install Ingress-Nginx](#contents)
``` bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```

&nbsp;
### [4. Install Helm](#contents)
``` bash
sudo curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
sudo chmod 700 get_helm.sh
sudo ./get_helm.sh
```

&nbsp;
### [5. Install NFS Subdir External Provisioner](#contents)
``` bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-sc nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=0.0.0.0 \
    --set nfs.path=/to/path \
    --set storageClass.name=nfs-sc \
    --set storageClass.defaultClass=true \
    --namespace nfs-storage \
    --create-namespace
```

#
## [# Install Additional Apps](#contents)
<br />

&nbsp;
### [1. Install Kubernetes Dashboard](#contents)

> Deploy Kubernetes Dashboard Yaml

``` bash
kubectl apply -f kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

> Create Dashboard User

``` bash
# Create Service Account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

# Create ClusterRoleBinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

> Get Dashboard User's Token

``` bash
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user \
-o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```

&nbsp;
### [2. Install Kubernetes Metrics Server](#contents)

``` bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

> ## Common Issue:
> metrics-server-847dcc659d-kxpxq stucks at ``0/1 Running`` <br/>
> Warning: Readiness probe failed: HTTP probe failed with statuscode: 500 <br/>
>
>> When try to check the status of Mertics Servers <br/>
>> `` $ kubectl get apiservices | grep metrics `` <br/>
>> v1beta1.metrics.k8s.io &emsp; kube-system/metrics-server &emsp; False (MissingEndpoints) &emsp; 5m31s
>
> <br/>
>
> ### Solution:
> <br/>
>
> #### Edit Metrics Server Deployment
> ``` bash
> kubectl edit deployments.apps -n kube-system metrics-server
> ```
> #### Add
> #### &emsp; ``--kubelet-insecure-tls=true``
> ``` yaml
>    spec:
>      containers:
>      - args:
>        - --cert-dir=/tmp
>        - --secure-port=4443
>        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
>        - --kubelet-use-node-status-port
>        - --metric-resolution=60s
>        - --kubelet-insecure-tls=true
> ```
>
> &nbsp;