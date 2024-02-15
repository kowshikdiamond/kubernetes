# Setting up multi master k8s cluster using kubeadm with HAProxy

## What is kubeadm
kubeadm is a command-line tool used for installing and managing Kubernetes clusters. It is part of the Kubernetes project and is designed to simplify the process of setting up a basic and production-ready Kubernetes cluster.

## What is HAProxy
HAProxy is used as a load balancer to distribute traffic across multiple Kubernetes clusters. It plays a crucial role in maintaining high availability by redirecting traffic to available clusters. In the provided setup, it acts as an entry point, routing requests to the appropriate Kubernetes cluster.

## Prequisities
Launch 4 ubuntu EC2 instances from AWS. Where one instance is used to install HAProxy and other 3 as Masters

### ON HAProxy server
Switch to root user
```
sudo su
```
Install haproxy
```
apt update && apt install -y haproxy
```
Configure 
```
nano /etc/haproxy/haproxy.cfg
```
Add the following to end of the file
```
frontend kubernetes-frontend
   bind private_ip_hxproxy:6443
   mode tcp
   option tcplog
   default_backend kubernetes-backend

backend kubernetes-backend
   mode tcp
   option tcp-check
   balance roundrobin
   server kmaster1 master1_privateip:6443 check fall 3 rise 2
   server kmaster2 master2_privateip:6443 check fall 3 rise 2
   server kmaster3 master3_privateip:6443 check fall 3 rise 2
```

**Note**: Replace ip address with appropriate ip address of your ec2 instances

Restart HAProxy service
```
systemctl restart haproxy
```

#### Do this on all ec2 instances:
```
nano /etc/hosts
```
Add the following
```
ip_address k8master1.node.com node.com k8master1
ip_address k8master2.node.com node.com k8master2
ip_address k8master3.node.com node.com k8master3
ip_address lb.node.com node.com lb
```
**Note**:Replace ip address with appropriate ip address of your ec2 instances

### On ALL 3 Master NODES
Switch to root user
```
sudo su
```
Disable Firewall and reboot
```
ufw disable
```
```
reboot
```
Again switch to root user
```
sudo su
```
Turn off Swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
Run the following commands
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
_This command will create a file named k8s.conf in the /etc/modules-load.d/ directory with the content "overlay" and "br_netfilter." These are kernel modules required for Kubernetes. The use of tee with sudo allows the command to write to a system directory that requires elevated permissions._

Load the specified kernel modules into the Linux kernel by running following commands
```
modprobe overlay
```
_This command loads the overlay kernel module. In the context of containerization and technologies like Docker and Kubernetes, the overlay filesystem is commonly used to provide a union mount for layers in container images. It allows for efficient use of disk space and enables the stacking of multiple filesystem layers_

```
modprobe br_netfilter
```
_This command loads the br_netfilter kernel module. The br_netfilter module is used for enabling advanced packet filtering and connection tracking for Linux bridge devices. In the context of Kubernetes, this module is often needed to enable network communication between containers running on different nodes in the cluster_

Run the following commands

These are sysctl settings, recommended for setting up a Linux system to work with container orchestration platforms like Kubernetes, ensuring proper networking and connectivity between containers
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```
1. net.bridge.bridge-nf-call-iptables = 1: This setting enables iptables to process bridged traffic. In the context of Kubernetes, it is often necessary to ensure that network traffic between containers can be properly filtered by iptables rules.

2. net.bridge.bridge-nf-call-ip6tables = 1: Similar to the previous setting but for IPv6 traffic. It allows iptables rules to be applied to IPv6 traffic in bridged configurations.

3. net.ipv4.ip_forward = 1: Enabling IPv4 forwarding allows the Linux kernel to forward network packets from one network interface to another, which is a requirement for routing and forwarding traffic between containers in different subnets

Apply sysctl params without reboot
```
sysctl --system
```

Install the necessary packages that are commonly required for securely fetching and installing packages from repositories that use HTTPS on a Debian-based system
```
apt-get install -y apt-transport-https ca-certificates curl
```

1. apt-transport-https: This package provides support for downloading packages over HTTPS. Many software repositories use HTTPS for secure package transmission, and this package ensures that the system can fetch packages using the HTTPS protocol

2. ca-certificates: This package contains the certificate authorities (CA) certificates needed for secure communication. It is necessary to have a trustworthy set of CA certificates to verify the authenticity of SSL/TLS connections, including when fetching packages over HTTPS.

Install Docker
```
apt install docker.io -y
```
```
mkdir /etc/containerd
```
```
sh -c "containerd config default > /etc/containerd/config.toml"
```

_sh -c "containerd config default > /etc/containerd/config.toml": This command uses the containerd command-line interface (CLI) to generate a default configuration and then redirects the output to a file named config.toml in the /etc/containerd/ directory. The sh -c is used to execute the command within a shell_
```
sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
```
_sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml: This command uses the sed (stream editor) tool to replace the value of SystemdCgroup from false to true in the config.toml file. This modification is likely made to enable containerd to use Systemd as the cgroup manager_

Restart the containerd service 
```
systemctl restart containerd.service
```
Add apt repo for kubernetes
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
 ```
 ```
apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```
Install Kubernetes components
```
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

Restart kubelet service 
```
sudo systemctl restart kubelet.service
```
Enable kubelet service on boot
```
sudo systemctl enable kubelet.service
```
### Only on MASTER1
```
kubeadm init --control-plane-endpoint="haproxy_private_ip:6443" --upload-certs --apiserver-advertise-address=master1_private_ip --pod-network-cidr=192.168.0.0/16
```
After running the above command you will get 

```
kubeadm join 172.31.23.43:6443 --token efjw3c.zf27mbef1kl0iau1 --discovery-token-ca-cert-hash sha256:549d3eee3c05330e2f1e815beee97a4daa724bed6bf5072bf57862f14818afc7 --control-plane --certificate-key 740395f352abdbd7a4a0e9ceb55724a3863613419315a28de45ca2345d518d8
```
You also gets kubeadm join key that is used to join worker nodes.
### ON Other 2 Master nodes

Do the following on other 2 masters

We have to use the kubeadm join command, but we need to add one more thing with the command, (add `--apiserver-advertise-address=PrivateIP-of-MasterNode_ur running`
Command looks like this
```
kubeadm join 172.31.23.43:6443 --token efjw3c.zf27mbef1kl0iau1 --discovery-token-ca-cert-hash sha256:549d3eee3c05330e2f1e815beee97a4daa724bed6bf5072bf57862f14818afc7 --control-plane --certificate-key 740395f352abdbd7a4a0e9ceb55724a3863613419315a28de45ca2345d518d8 --apiserver-advertise-address=172.31.31.230
```

### Run these on all master nodes after doing above steps
```
mkdir -p $HOME/.kube
```
```
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
```
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### Run on any one Master Node

Let’s validate whether both Worker Nodes are joined in the Kubernetes Cluster by running the below command.
```
kubectl get nodes
```
Run the below command to add the Calico networking components in the Kubernetes Cluster.
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```
### Now again go to HAProxy server 

Add apt repo for kubernetes
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
 ```
 ```
apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```
Install Kubernetes components
```
apt update
```
```
apt-get install -y kubectl 
```

After that copy .kube/config file from master1 and place it in same directory

Now you can run k8s commands from HAProxy server also.


Note: When you try to deploy any pods the pods won’t be deployed due to absence of worker nodes so we have to configure our yaml file.

Here is an example yaml file for nginx:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: nginx
        image: nginx:alpine
```
TO deploy this create a service file and configure

Configure 
`
nano /etc/haproxy/haproxy.cfg`

Add the following to end of the file
```
frontend nginx-frontend
   bind private_ip_hxproxy:port
   mode tcp
   option tcplog
   default_backend nginx-backend

backend nginx-backend
   mode tcp
   option tcp-check
   balance roundrobin
   server kmaster1 master1_privateip:port check fall 3 rise 2
   server kmaster2 master2_privateip:port check fall 3 rise 2
   server kmaster3 master3_privateip:port check fall 3 rise 2
```
Now you can browser your nginx on haproxy_ip:port