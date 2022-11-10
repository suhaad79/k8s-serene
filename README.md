# k8s-serene
master and worker nodes complete installation

1: set root Password  

2: set current admin user Password  

3: enable sshd
sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
sudo service sshd restart

4: set hostname
sudo hostnamectl set-hostname 


===================== K8S installation for both master and nodes =========================================

# make sure your server is up to date
#switch to root.

sudo su 

#Turn Off Swap Space for both  master k8s node and slave nodes
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab  

<<note
vim /etc/fstab and make sure to stop swap from reboot which is needed by putting a comment on this line 
# "/swap.img       none    swap    sw      0       0 #
note


# for all the nodes
#after the above,  a reboot will not turn on swap  


#Update the apt package index and install packages needed to use the Kubernetes apt repository:
apt-get update
apt-get install -y apt-transport-https ca-certificates curl fish nano unzip



#Add the Kubernetes apt repository:
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list

#Download the Google Cloud public signing key:
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg |  apt-key add -


echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list



#Update apt package index, kubeadm and kubectl, kubernetes-cni and pin their version:

apt-get update

apt-get install -y vim wget unzip curl fish tmux
apt-get install -y kubeadm
apt-get install -y kubectl
apt-get install -y kubelet

apt-mark hold kubelet kubeadm kubectl  
#The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.

# Setup required sysctl params, these persist across reboots.
tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system

# setup containerd and load the modules required
cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

# Ensure sysctl params are set
# Enable kernel modules 
modprobe overlay
modprobe br_netfilter

# make sure that the br_netfilter module is loaded:
lsmod | grep br_netfilter

# Add some settings to sysctl and configure sysctl.
# Setup required sysctl params, these persist across reboots.

# Apply sysctl params without reboot
sysctl --system

#===============container runtime (docker, containerd or CRI-O)==================================
#installing containerd
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

apt update

apt install -y containerd.io

# Configure containerd:
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

apt update

# Restart containerd:
systemctl start containerd
systemctl enable containerd   
systemctl restart containerd

# enable kubelet:
systemctl start kubelet
systemctl enable kubelet
systemctl restart kubelet

#kubelet will start running only when a cluster is up

apt update

===================== END of K8S installation for both master and nodes =========================================
==============================================================================================


Before running kubeadm, make sure all the required ports are open

6443, 2379-2380, 10248-10252, 30000-32767, HTTP(80), HTTPS(443)

#For master Node, run: it will initialize the masternode to become a cluster/control plane to manage the cluster
kubeadm init   # --cri-socket /run/containerd/containerd.sock #container run time is needed to run our container which we choose containerd


======================================copy and save the result========================================================  

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

or use weave-network below:

   kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

Then you can join any number of worker nodes by running the following on each as root:


#use what your cluster will give you not this:
#Important! this is a joint script to join your master with your worker nodes

kubeadm join 172.31.35.147:6443 --token 96rq7b.zczwxy7lvs2vasc0 \
        --discovery-token-ca-cert-hash sha256:7bb283e6790c64b36e86d8e50d39ec78f4ab27b44a13374a834a753ec31df0da


#for master to run  pods as well because by default it is not allowed, run the following command. This means accessing the pods from the master server but this isnt allowed in production grade. pods are to run in the worker nodes. 
kubectl taint node $(hostname) node-role.kubernetes.io/control-plane:NoSchedule-

#==========

#=====NEXT IS TO JOIN THE NODES ON THE MASTER node=====
kubectl get nodes # to list all your nodes 

#you can watch how you join your nodes using the watch command. 
watch -n2 "kubectl get nodes"

# Now you can join on the master using the result gotten from your workers installation
kubeadm join 172.31.38.223:6443 --token tk03j8.n2jfpps32izjtn6c \
        --discovery-token-ca-cert-hash sha256:d10b2d7985e67edd0d508b4385384b3b25e5e9ff40e5882895ca800fee0ffb29
