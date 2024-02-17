# Setting up multi master k8s cluster using kubeadm with HAProxy

**Introduction:**

In this tutorial will guide you through the process of setting up a multi-master Kubernetes cluster on AWS using kubeadm for cluster management and HAProxy as a load balancer. This configuration ensures high availability and load distribution across multiple Kubernetes master nodes.

**What is kubeadm** 

kubeadm is a command-line tool used for installing and managing Kubernetes clusters. It is part of the Kubernetes project and is designed to simplify the process of setting up a basic and production-ready Kubernetes cluster.

**What is HAProxy**

HAProxy is used as a load balancer to distribute traffic across multiple Kubernetes clusters. It plays a crucial role in maintaining high availability by redirecting traffic to available clusters. In the provided setup, it acts as an entry point, routing requests to the appropriate Kubernetes cluster.

## Prequisities
Launch 4 ubuntu EC2 instances from AWS. Where one instance is used to install HAProxy and other 3 as Masters

### Update hosts file on all instances:
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

### Setting Up HAProxy:

Install HAProxy on the designated server

Switch to root user
```
sudo su
```
Install haproxy
```
apt update && apt install -y haproxy
```
Configure HAProxy by editing `/etc/haproxy/haproxy.cfg`:
```
nano /etc/haproxy/haproxy.cfg
```
Add the following configuration:
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
**Note**:Replace ip address with appropriate ip address of your ec2 instances

### Setting Up Kubernetes Master Nodes
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
Run the following commands:
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

Configure Sysctl Settings, 

_Recommended for setting up a Linux system to work with container orchestration platforms like Kubernetes, ensuring proper networking and connectivity between containers_
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```
- `net.bridge.bridge-nf-call-iptables = 1` This setting enables iptables to process bridged traffic. In the context of Kubernetes, it is often necessary to ensure that network traffic between containers can be properly filtered by iptables rules.
- `net.bridge.bridge-nf-call-ip6tables = 1` Similar to the previous setting but for IPv6 traffic. It allows iptables rules to be applied to IPv6 traffic in bridged configurations.
- `net.ipv4.ip_forward = 1` Enabling IPv4 forwarding allows the Linux kernel to forward network packets from one network interface to another, which is a requirement for routing and forwarding traffic between containers in different subnets

Apply sysctl params without reboot
```
sysctl --system
```

Install the necessary packages 
```
apt-get install -y apt-transport-https ca-certificates curl
```

- apt-transport-https: This package provides support for downloading packages over HTTPS. Many software repositories use HTTPS for secure package transmission, and this package ensures that the system can fetch packages using the HTTPS protocol

- ca-certificates: This package contains the certificate authorities (CA) certificates needed for secure communication. It is necessary to have a trustworthy set of CA certificates to verify the authenticity of SSL/TLS connections, including when fetching packages over HTTPS.

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
```
sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
```

- _sh -c "containerd config default > /etc/containerd/config.toml": This command uses the containerd command-line interface (CLI) to generate a default configuration and then redirects the output to a file named config.toml in the /etc/containerd/ directory. The sh -c is used to execute the command within a shell_

- _sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml: This command uses the sed (stream editor) tool to replace the value of SystemdCgroup from false to true in the config.toml file. This modification is likely made to enable containerd to use Systemd as the cgroup manager_

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
### Initialize Kubernetes on MASTER1
```
kubeadm init --control-plane-endpoint="haproxy_private_ip:6443" --upload-certs --apiserver-advertise-address=master1_private_ip --pod-network-cidr=192.168.0.0/16
```
After running the above command you will get 

```
kubeadm join haproxy_ip:6443 --token efjw3c.zf27mbef1kl0iau1 --discovery-token-ca-cert-hash sha256:549d3eee3c05330e2f1e815beee97a4daa724bed6bf5072bf57862f14818afc7 --control-plane --certificate-key 740395f352abdbd7a4a0e9ceb55724a3863613419315a28de45ca2345d518d8
```
You also gets kubeadm join key that is used to join worker nodes.
### Join Other Master Nodes

Do the following on other 2 masters

Run the modified kubeadm join command on other master nodes, adding `--apiserver-advertise-address=PrivateIP-of-MasterNode`.
Command looks like this
```
kubeadm join 172.31.23.43:6443 --token efjw3c.zf27mbef1kl0iau1 --discovery-token-ca-cert-hash sha256:549d3eee3c05330e2f1e815beee97a4daa724bed6bf5072bf57862f14818afc7 --control-plane --certificate-key 740395f352abdbd7a4a0e9ceb55724a3863613419315a28de45ca2345d518d8 --apiserver-advertise-address=privateip_master2
```

### Configure Kubeconfig on Masters
```
mkdir -p $HOME/.kube
```
```
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
```
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Let’s validate whether both Worker Nodes are joined in the Kubernetes Cluster by running the below command.
```
kubectl get nodes
```
The state will be not ready due to no network connection between master nodes.

To setup add the Calico networking components in the Kubernetes Cluster.
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```
### Setting Up HAProxy for Application Deployment

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


**Note:** When you try to deploy any pods the pods won’t be deployed due to absence of worker nodes so we have to configure our yaml file.

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
TO deploy this create a service file with NodePort.

Now Configure `/etc/haproxy/haproxy.cfg` to use haproxy as loadbalancer for our nginx application.
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

**Conclusion:**

By following these detailed steps, you have successfully set up a robust, high-availability Kubernetes cluster with multiple master nodes using kubeadm and HAProxy. Feel free to explore further and deploy your applications in this powerful, scalable environment.