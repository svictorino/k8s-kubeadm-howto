# k8s-kubeadm-howto

* * *

## Enable iptables Bridged Traffic on all the Nodes

```
Execute the following commands on all the nodes for IPtables to see bridged traffic.

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
```

## Disable swap on all the Nodes

```
For kubeadm to work properly, you need to disable swap on all the nodes using the following command.

sudo swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```

## Installing CRI-O

```
sudo yum -y update
sudo yum -y install epel-release
```

### Set the variables of what you're about to install, k8s version and OS version

```
VERSION=1.26 ## K8S Version being installed
OS=CentOS_8
```

### Add CRI-O repo to your distro

```
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:/kubic:/libcontainers:/stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
```

### Install CRI-O

```
sudo yum install cri-o cri-tools
rpm -qi cri-o ##check if it was installed
```

### This block helped me solve a problem while trying to start things up. I recommend doing it

```
cat /etc/containers/registries.conf
unqualified-search-registries = ["docker.io"] ##this line already exists, probably, if it does, leave it as it is...

[[registry]]
prefix = "registry.k8s.io"
insecure = false
blocked = false
location = "registry.k8s.io"

[[registry.mirror]]
location = "user.private.repo"
```

### Enable CRI-O

```
sudo systemctl enable --now crio (ativa o serviço)
```

* * *

## KUBEADM KUBELET KUBECTL

### Add k8s tools repo

```
sudo cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

### Install the tools

```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

### End Kubelet setup

```
sudo apt-get install -y jq
local_ip="$(ip --json a s | jq -r '.[] | if .ifname == "eth1" then .addr_info[] | if .family == "inet" then .local else empty end else empty end')"
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF
```

* * *

## Start bringing things up - \[master node\]

```
sudo systemctl enable kubelet.service
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo systemctl mask firewalld

export IPADDR="10.0.0.10" ## POD NET
export NODENAME=$(hostname -s)
export POD_CIDR="192.168.0.0/16" ## POD NET
```

* * *

### Put it to work! - \[master node\]

This command will start your K8S. I had to add "--ignore-preflight-errors=NumCPU" because OCI gives us a lot of free stuff but we have a limit of CPUs available too use so, I use only 1 CPU per host.

```
sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap --ignore-preflight-errors=NumCPU
```

You should seer something like this. At the end of this block I added "--ignore-preflight-errors=cri". As you know, use it if you find any problem

```
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf


You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join XXX.XX.X.XXX:6443 --token lksahflkasdjfhalkd \
        --discovery-token-ca-cert-hash sha256:lkjghdfslgkdhfglkhlkfjghdfkljghdfkljdfh6c722f5b --ignore-preflight-errors=cri
```

### Enable Networkng

This part nice. I always opted for flannel because I find it easier, but now I would recommend you go with calico.

To go with flannel:

```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Edit the file and find the network field. Put the network you have opted to use before
Save and apply

```
kubectl create -f kube-flannel.yml
```

You verify all the cluster component health statuses using the following command.

```
kubectl get --raw='/readyz?verbose'
```
