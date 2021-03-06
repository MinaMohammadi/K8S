
# kubernetes
With modern web services, users expect applications to be available 24/7, and developers expect to deploy new versions of those applications several times a day. Containerization helps package software to serve these goals, enabling applications to be released and updated without downtime.

Kubernetes helps you make sure those containerized applications run where and when you want, and helps them find the resources and tools they need to work. Kubernetes is a production-ready, open source platform designed with Google's accumulated experience in container orchestration, combined with best-of-breed ideas from the community.

## Kubernetes Components
When you deploy Kubernetes, you get a cluster. A Kubernetes cluster consists of a set of worker machines, called nodes, that run containerized applications. Every cluster has at least one worker node.

The control plane manages the worker nodes and the Pods in the cluster. In production environments, the control plane usually runs across multiple computers and a cluster usually runs multiple nodes, providing fault-tolerance and high availability.


***

# Set up a Highly Available Kubernetes Cluster using kubeadm
Follow this documentation to set up a highly available Kubernetes cluster using Ubuntu 20.04 LTS with keepalived and haproxy

This documentation guides you in setting up a cluster with three master nodes, tress worker nodes and two load balancer node using HAProxy and Keepalived.


## Set up load balancer nodes (loadbalancer1 & loadbalancer2)

### Install and configure keepalived

```
apt update && apt install -y keepalived 
```
configure keepalived:
On both nodes create the health check script /etc/keepalived/check_apiserver.sh.

```
cat >> /etc/keepalived/check_apiserver.sh <<EOF
#!/bin/sh

errorExit() {
  echo "*** $@" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 172.16.16.100; then
  curl --silent --max-time 2 --insecure https://172.16.16.100:6443/ -o /dev/null || errorExit "Error GET https://172.16.16.100:6443/"
fi
EOF

chmod +x /etc/keepalived/check_apiserver.sh
```

Create keepalived config /etc/keepalived/keepalived.conf.

```
cat >> /etc/keepalived/keepalived.conf <<EOF
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        172.16.16.100
    }
    track_script {
        check_apiserver
    }
}
EOF
```
Enable & start keepalived service:

```
systemctl enable --now keepalived
```
### Install and configure haproxy

```
apt update && apt install -y haproxy
```
Configure haproxy:
Update /etc/haproxy/haproxy.cfg.

```
cat >> /etc/haproxy/haproxy.cfg <<EOF

frontend kubernetes-frontend
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-backend

backend kubernetes-backend
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
    server master01 172.16.16.101:6443 check fall 3 rise 2
    server master02 172.16.16.102:6443 check fall 3 rise 2
    server master03 172.16.16.103:6443 check fall 3 rise 2

EOF
```
Enable & restart haproxy service:

```
systemctl enable haproxy && systemctl restart haproxy
```
***

## Pre-requisites on all kubernetes nodes (masters & workers)

Disable swap:

```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
systemctl disable --now ufw:

``` 
systemctl disable --now ufw
```
Enable and Load Kernel modules:

```
{
cat >> /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
}
```
Add Kernel settings:

```
{
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
}
```
Install containerd runtime:

```
{
  apt update
  apt install -y containerd apt-transport-https
  mkdir /etc/containerd
  containerd config default > /etc/containerd/config.toml
  systemctl restart containerd
  systemctl enable containerd
}
```
Add apt repo for kubernetes:

```
{
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
}
```
Install Kubernetes components:

```
{
  apt update
  apt install -y kubeadm=1.22.0-00 kubelet=1.22.0-00 kubectl=1.22.0-00
}
```

## Bootstrap the cluster

### On master01
Initialize Kubernetes Cluster:

```
kubeadm init --control-plane-endpoint="172.16.16.100:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```
Copy the commands to join other master nodes and worker nodes.

Deploy Flannel network:

```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
### Join other master/worker nodes to the cluster

Use the respective kubeadm join commands you copied from the output of kubeadm init command on the first master.

### Verifying the cluster

```
kubectl cluster-info
kubectl get nodes
```

***

