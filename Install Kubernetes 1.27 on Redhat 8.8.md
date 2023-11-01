1- Disable Swap, SELinux, Firewall
2- Hostnames


CRIO
-----
export OS="CentOS_8"
export VERSION="1.27"
export FULL_VERSION="1.27.1"

curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION:/$FULL_VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION:$FULL_VERSION.repo

sudo yum install cri-o -y
sudo systemctl enable --now crio


=================================================================================================================================================================
Kube [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/]
-----
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum install -y kubelet-1.27.5-0 kubeadm-1.27.5-0 kubectl-1.27.5-0 --disableexcludes=kubernetes


Flannel [https://github.com/flannel-io/flannel#deploying-flannel-manually]
-----------
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml


MetalLB [https://docs.okd.io/4.11/networking/metallb/about-advertising-ipaddresspool.html]
--------
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.11/config/manifests/metallb-native.yaml

cat << EOF | kubectl apply -f - 
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pool-1
  namespace: metallb-system
spec:
  addresses:
  - 10.0.12.84-10.0.12.90
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


Ingress-nginx
-------------
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml