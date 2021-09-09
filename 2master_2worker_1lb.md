Below are our requirements for the installation:
```
1.  1 servers for HAProxy with Keepalived, running CentOS 7
2.  2 servers for control-plane nodes, running CentOS 7
3.  2 servers for worker nodes, running CentOS 7
4.  Full network connectivity between all machines in the cluster.
5.  sudo privileges on all machines.
6.  SSH access from one device to all nodes in the system.
7.  A DNS server with a DNS entry for the HAProxy load balancer pointing to a virtual IP address 192.168.125.89.
```
My DNS record for HAProxy:

**$ host bastion**
bastion has address 192.168.125.89

Homelab details can be seen in the table below.

Software
--------

Kubernetes development continues to grow at a rapid pace, and keeping up to date can be a challenge. Therefore it’s important to know which software versions can work together without breaking things.

Software used in this article:
```
1.  CentOS 7
2.  calico 3.17
3.  kubeadm 1.19.7
4.  kubelet 1.19.7
5.  kubectl 1.19.7
6.  kubernetes-cni 0.8.7
7.  docker-ce 19.03
```
According to Calico project documentation, Calico 3.17 has been tested against the following Kubernetes versions: 1.17, 1.18, 1.19. Kubernetes 1.20 is not on the list yet, therefore we are going to use 1.19.

Unfortunatelly I could not find supported Docker versions in the Relase Notes for Kubernetes 1.19, so I decided to use docker-ce 19.03.

SELinux set to enforcing mode and firewalld is enabled on all servers.

1 Install and Configure HAProxy Load Balancer with Keepalived
-------------------------------------------------------------

Run these commands on both servers **master-1** and **master-2**.

### 1.1 Configure Firewalld
```
Configure firewall to allow inbound HAProxy traffic on kube-apiserver port:

$ sudo firewall-cmd --permanent --add-port=6443/tcp
$ sudo firewall-cmd --reload

Configure firewall to allow inbound traffic for HAProxy stats:

$ sudo firewall-cmd --permanent --add-port=8080/tcp
$ sudo firewall-cmd --reload
```
Configure firewall to allow VRRP traffic to pass between the keepalived nodes:
```
$ sudo firewall-cmd --permanent --add-rich-rule='rule protocol value="vrrp" accept'
$ sudo firewall-cmd --reload
```
### 1.2 Configure SELinux

Allow HAProxy to listen on kube-apiserver port 6443:
```
$ sudo semanage port -a -t http_cache_port_t 6443 -p tcp
```
### 1.3 Install Packages
```
$ sudo yum install -y haproxy keepalived psmisc
```
### 1.4 Configure HAProxy

Add the following configuration to file `/etc/haproxy/haproxy.cfg`, keeping in mind that our virtual IP address is 192.168.125.89:
```
global
  log /dev/log  local0
  log /dev/log  local1 notice
  stats socket /var/lib/haproxy/stats level admin
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon

defaults
  log global
  mode  http
  option  httplog
  option  dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000

frontend kubernetes
    bind 192.168.125.89:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server master-1-master 192.168.125.84:6443 check fall 3 rise 2
    server master-2-master 192.168.125.87:6443 check fall 3 rise 2

listen stats 192.168.125.89:8080
    mode http
    stats enable
    stats uri
    stats realm HAProxy Statistics
    stats auth admin:haproxy
```
Enable and start the haproxy service:
```
$ sudo systemctl enable --now haproxy
```
You can access HAProxy stats page by navigating to the following URL: 
`http://192.168.125.89:8080/` 
Username is “admin”, and password is “haproxy”.
`
![](https://www.lisenet.com/wp-content/uploads/2021/01/homelab-haproxy-stats.png)

Note: the screenhost was taken after deploying Kubernetes.

2 Install Docker Packages
-------------------------

Run these commands on all Kubernetes servers.

Install the yum-utils package (which provides the yum-config-manager utility) and set up the stable repository.
```
$ sudo yum install -y yum-utils
$ sudo yum-config-manager \
  --add-repo https://download.docker.com/linux/centos/docker-ce.repo

Install Docker engine:

$ sudo yum install -y \
  docker-ce-19.03.14-3.el7.x86_64 \
  docker-ce-cli-19.03.14-3.el7.x86_64 \
  containerd.io-1.4.3-3.1.el7.x86_64

Start and enable service:

$ sudo systemctl enable --now docker
```
3 Install Kubernetes Packages and Disable Swap
----------------------------------------------

Run these commands on all Kubernetes servers.

Our PXE boot servers don’t have swap configured by default, but in case you do, disable it. Running kubelet with swap on is not supported.

`$ sudo swapoff -a`

Set up the repository. Add the following to `/etc/yum.repos.d/kubernetes.repo`:
```
\[kubernetes\]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
```
We exclude all Kubernetes packages from any system upgrades because they have a special process that has to be followed.

Install kubeadm, kubelet and kubectl:
```
$ sudo yum install -y --disableexcludes=kubernetes \
  kubernetes-cni-0.8.7-0.x86_64 \
  kubelet-1.19.7-0.x86_64 \
  kubectl-1.19.7-0.x86_64 \
  kubeadm-1.19.7-0.x86_64
```
Enable kubelet service:
```
$ sudo systemctl enable --now kubelet
```
Note that the kubelet is now in a crashloop and restarting every few seconds, as it waits for kubeadm to tell it what to do.

4 Configure kubelet Eviction Thresholds
---------------------------------------

Run these commands on all Kubernetes servers.

Unless resources are set aside for system daemons, Kubernetes pods and system daemons will compete for resources and eventually lead to resource starvation issues. Kubelet has the extra args parameter to specify eviction thresholds that trigger the kubelet to reclaim resources.
```
$ echo "KUBELET_EXTRA_ARGS=--eviction-hard=memory.available<256Mi,nodefs.available<1Gi,imagefs.available<1Gi"|sudo tee /etc/sysconfig/kubelet
```
Restart kubelet service:
```
$ sudo systemctl restart kubelet
```
5 Let iptables see Bridged Traffic
----------------------------------

Run these commands on all Kubernetes servers.
```
$ echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee /etc/sysctl.d/k8s-iptables.conf
$ echo "net.bridge.bridge-nf-call-ip6tables=1" | sudo tee /etc/sysctl.d/k8s-ip6tables.conf
$ sudo sysctl --system
```
6 Configure Firewalld
---------------------

### 6.1 Firewall Rules for Control-plane Nodes

Run these commands on Kubernetes control-plane nodes only.

We are going to open the following ports on the control-plane nodes:
```
1.  8443 – Kubernetes API server
2.  2379,2380 – etcd server client API
3.  2381 – etcd metrics API
4.  10250 – Kubelet API
5.  10251 – kube-scheduler
6.  10252 – kube-controller-manager
7.  179 – Calico networking (BGP)
```
```
$ sudo firewall-cmd --permanent --add-port={6443,2379-2381,10250-10252}/tcp
$ sudo firewall-cmd --permanent --add-port=179/tcp
$ sudo firewall-cmd --permanent --add-masquerade
$ sudo firewall-cmd --reload
```
One interesting note here, I kept getting CoreDNS crashes like this one:

_CoreDNS crashes with error “Failed to list v1.Service: Get https://10.96.0.1:443/api/v1/\*\*\*: dial tcp 10.96.0.1:443: connect: no route to host”._

I added masquerade to firewalld and I think it helped fix the problem.

### 6.2 Firewall Rules for Worker Nodes

Run these commands on Kubernetes worker nodes only.

We are going to open the following ports on the worker nodes:
```
1.  10250 – Kubelet API
2.  30000-32767 – NodePort Services
3.  179 – Calico networking (BGP)
```
```
$ sudo firewall-cmd --permanent --add-port={10250,30000-32767}/tcp
$ sudo firewall-cmd --permanent --add-port=179/tcp
$ sudo firewall-cmd --permanent --add-masquerade
$ sudo firewall-cmd --reload
```
7 Initialise the First Control Plane Node
-----------------------------------------

### 7.1 Initialise One Master Node Only

Run these commands on the first control plane node **master-1** only.
```
$ sudo kubeadm init \
  --kubernetes-version "1.19.7" \
  --pod-network-cidr "192.168.0.0/16" \
  --service-dns-domain "apps" \
  --control-plane-endpoint "bastion:6443" \
  --upload-certs
```
Command output (note that your output will be different than what is provided below):
```
_W0116 15:49:53.753185   12133 configset.go:348\] WARNING: kubeadm cannot validate component configs for API groups \[kubelet.config.k8s.io kubeproxy.config.k8s.io\]
\[init\] Using Kubernetes version: v1.19.7
\[preflight\] Running pre-flight checks
	\[WARNING Firewalld\]: firewalld is active, please ensure ports \[6443 10250\] are open or your cluster may not function correctly
	\[WARNING IsDockerSystemdCheck\]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
\[preflight\] Pulling images required for setting up a Kubernetes cluster
\[preflight\] This might take a minute or two, depending on the speed of your internet connection
\[preflight\] You can also perform this action in beforehand using 'kubeadm config images pull'
\[certs\] Using certificateDir folder "/etc/kubernetes/pki"
\[certs\] Generating "ca" certificate and key
\[certs\] Generating "apiserver" certificate and key
\[certs\] apiserver serving cert is signed for DNS names \[bastion kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.apps master-1\] and IPs \[10.96.0.1 192.168.125.84\]
\[certs\] Generating "apiserver-kubelet-client" certificate and key
\[certs\] Generating "front-proxy-ca" certificate and key
\[certs\] Generating "front-proxy-client" certificate and key
\[certs\] Generating "etcd/ca" certificate and key
\[certs\] Generating "etcd/server" certificate and key
\[certs\] etcd/server serving cert is signed for DNS names \[localhost master-1\] and IPs \[192.168.125.84 127.0.0.1 ::1\]
\[certs\] Generating "etcd/peer" certificate and key
\[certs\] etcd/peer serving cert is signed for DNS names \[localhost master-1\] and IPs \[192.168.125.84 127.0.0.1 ::1\]
\[certs\] Generating "etcd/healthcheck-client" certificate and key
\[certs\] Generating "apiserver-etcd-client" certificate and key
\[certs\] Generating "sa" key and public key
\[kubeconfig\] Using kubeconfig folder "/etc/kubernetes"
\[kubeconfig\] Writing "admin.conf" kubeconfig file
\[kubeconfig\] Writing "kubelet.conf" kubeconfig file
\[kubeconfig\] Writing "controller-manager.conf" kubeconfig file
\[kubeconfig\] Writing "scheduler.conf" kubeconfig file
\[kubelet-start\] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
\[kubelet-start\] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
\[kubelet-start\] Starting the kubelet
\[control-plane\] Using manifest folder "/etc/kubernetes/manifests"
\[control-plane\] Creating static Pod manifest for "kube-apiserver"
\[control-plane\] Creating static Pod manifest for "kube-controller-manager"
\[control-plane\] Creating static Pod manifest for "kube-scheduler"
\[etcd\] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
\[wait-control-plane\] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
\[apiclient\] All control plane components are healthy after 32.032492 seconds
\[upload-config\] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
\[kubelet\] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
\[upload-certs\] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
\[upload-certs\] Using certificate key:
c4161499f1f614322ea788831b4f72529175712c7c6f8888cdc14f5aab83fbce
\[mark-control-plane\] Marking the node master-1 as control-plane by adding the label "node-role.kubernetes.io/master=''"
\[mark-control-plane\] Marking the node master-1 as control-plane by adding the taints \[node-role.kubernetes.io/master:NoSchedule\]
\[bootstrap-token\] Using token: 8rd7kq.kcg3bkzkzdus8v54
\[bootstrap-token\] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
\[bootstrap-token\] configured RBAC rules to allow Node Bootstrap tokens to get nodes
\[bootstrap-token\] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
\[bootstrap-token\] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
\[bootstrap-token\] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
\[bootstrap-token\] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
\[kubelet-finalize\] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
\[addons\] Applied essential addon: CoreDNS
\[addons\] Applied essential addon: kube-proxy
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
You should now deploy a pod network to the cluster.
Run "kubectl apply -f \[podnetwork\].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:
```
  kubeadm join bastion:6443 --token 8rd7kq.kcg3bkzkzdus8v54 \
    --discovery-token-ca-cert-hash sha256:4efb75f7e09c0a2a9db0d317dd40ac9cf9906c31109a670428d0d49981264904 \
    --control-plane --certificate-key c4161499f1f614322ea788831b4f72529175712c7c6f8888cdc14f5aab83fbce
```
Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
`"kubeadm init phase upload-certs --upload-certs" `
to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:
```
kubeadm join bastion:6443 --token 8rd7kq.kcg3bkzkzdus8v54 \
    --discovery-token-ca-cert-hash sha256:4efb75f7e09c0a2a9db0d317dd40ac9cf9906c31109a670428d0d49981264904_ 
```
### 7.2 Configure Kube Config on Your Local Machine

Run the following commands on your local machine (e.g. your laptop):
```
$ mkdir -p $HOME/.kube
$ scp master-1:/etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### 7.3 Install Calico Pod Network

Run the following commands on your local machine (e.g. your laptop) where you have kubectl configured.
```
$ kubectl apply -f https://docs.projectcalico.org/archive/v3.17/manifests/calico.yaml
```
Make sure that the control-plane node status is ready:
```
$ kubectl get nodes
NAME    STATUS   ROLES    AGE   VERSION
master-1   Ready    master   10m   v1.19.7
```
8 Join Other Control Plane Nodes to the Cluster
-----------------------------------------------

Run the following command on control-plane nodes **master-2**

Note that your actual command will be different than what is provided below!
```
$ sudo kubeadm join bastion:6443 --token 8rd7kq.kcg3bkzkzdus8v54 \
  --discovery-token-ca-cert-hash sha256:4efb75f7e09c0a2a9db0d317dd40ac9cf9906c31109a670428d0d49981264904 \
  --control-plane --certificate-key c4161499f1f614322ea788831b4f72529175712c7c6f8888cdc14f5aab83fbce
```
The output should contain the following lines:

_\[...\]
This node has joined the cluster and a new control plane instance was created:

\* Certificate signing request was sent to apiserver and approval was received.
\* The Kubelet was informed of the new secure connection details.
\* Control plane (master) label and taint were applied to the new node.
\* The Kubernetes control plane instances scaled up.
\* A new etcd member was added to the local/stacked etcd cluster.
\[...\]_

9 Join Worker Nodes to the Cluster
----------------------------------

Run the following command on worker nodes **worker-1**, **worker-2** and **srv36**.

Note that your actual command will be different than what is provided below!

$ sudo kubeadm join bastion:6443 --token 8rd7kq.kcg3bkzkzdus8v54 \
  --discovery-token-ca-cert-hash sha256:4efb75f7e09c0a2a9db0d317dd40ac9cf9906c31109a670428d0d49981264904 

The output should contain the following lines:

_\[...\]
This node has joined the cluster:
\* Certificate signing request was sent to apiserver and a response was received.
\* The Kubelet was informed of the new secure connection details.
\[...\]_

10 Verify the Cluster
---------------------

To verify, run the following commands on your local machine where you have kubectl configured.
`$ kubectl get nodes -o wide`
```
NAME    STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
master-1   Ready    master   6h16m   v1.19.7   192.168.125.84    none          CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.14
master-2   Ready    master   6h2m    v1.19.7   192.168.125.87    none          CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.14
worker-1   Ready    none     5h55m   v1.19.7   192.168.125.88    none          CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.14
worker-2   Ready    none     5h52m   v1.19.7   192.168.125.90    none          CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.14
```
`$ kubectl get all -A`
```
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
kube-system   pod/calico-kube-controllers-744cfdf676-thmgj   1/1     Running   0          6h3m
kube-system   pod/calico-node-2gjtd                          1/1     Running   0          5h55m
kube-system   pod/calico-node-cpx67                          1/1     Running   0          5h52m
kube-system   pod/calico-node-p57h8                          1/1     Running   0          5h59m
kube-system   pod/calico-node-qkn55                          1/1     Running   0          5h46m
kube-system   pod/calico-node-rg8nz                          1/1     Running   0          6h3m
kube-system   pod/calico-node-xwrth                          1/1     Running   0          5h49m
kube-system   pod/coredns-f9fd979d6-6mx72                    1/1     Running   0          6h13m
kube-system   pod/coredns-f9fd979d6-vszn8                    1/1     Running   0          6h13m
kube-system   pod/etcd-master-1                                 1/1     Running   0          6h13m
kube-system   pod/etcd-master-2                                 1/1     Running   0          5h59m
kube-system   pod/kube-apiserver-master-1                       1/1     Running   0          6h13m
kube-system   pod/kube-apiserver-master-2                       1/1     Running   0          5h59m
kube-system   pod/kube-controller-manager-master-2              1/1     Running   0          5h59m
kube-system   pod/kube-proxy-8dpxx                           1/1     Running   0          5h49m
kube-system   pod/kube-proxy-jk5zw                           1/1     Running   0          5h52m
kube-system   pod/kube-proxy-mv6k9                           1/1     Running   0          5h46m
kube-system   pod/kube-proxy-n5pxd                           1/1     Running   0          5h55m
kube-system   pod/kube-proxy-rbhqp                           1/1     Running   0          6h13m
kube-system   pod/kube-scheduler-master-1                       1/1     Running   1          6h13m
kube-system   pod/kube-scheduler-master-2                       1/1     Running   0          5h59m

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    none          443/TCP                  6h13m
kube-system   service/kube-dns     ClusterIP   10.96.0.10   none          53/UDP,53/TCP,9153/TCP   6h13m

NAMESPACE     NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/calico-node   6         6         6       6            6           kubernetes.io/os=linux   6h3m
kube-system   daemonset.apps/kube-proxy    6         6         6       6            6           kubernetes.io/os=linux   6h13m

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           6h3m
kube-system   deployment.apps/coredns                   2/2     2            2           6h13m

NAMESPACE     NAME                                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/calico-kube-controllers-744cfdf676   1         1         1       6h3m
kube-system   replicaset.apps/coredns-f9fd979d6                    2         2         2       6h13m
```
This concludes the Kubernetes homelab cluster installation using kubeadm.Below are our requirements for the installation:
```
1.  1 servers for HAProxy with Keepalived, running CentOS 7
2.  2 servers for control-plane nodes, running CentOS 7
3.  2 servers for worker nodes, running CentOS 7
4.  Full network connectivity between all machines in the cluster.
5.  sudo privileges on all machines.
6.  SSH access from one device to all nodes in the system.
7.  A DNS server with a DNS entry for the HAProxy load balancer pointing to a virtual IP address 192.168.125.89.
```
My DNS record for HAProxy:

**$ host bastion**
bastion has address 192.168.125.89

Homelab details can be seen in the table below.

Software
--------

Kubernetes development continues to grow at a rapid pace, and keeping up to date can be a challenge. Therefore it’s important to know which software versions can work together without breaking things.

Software used in this article:
```
1.  CentOS 7
2.  calico 3.17
3.  kubeadm 1.19.7
4.  kubelet 1.19.7
5.  kubectl 1.19.7
6.  kubernetes-cni 0.8.7
7.  docker-ce 19.03
```
According to Calico project documentation, Calico 3.17 has been tested against the following Kubernetes versions: 1.17, 1.18, 1.19. Kubernetes 1.20 is not on the list yet, therefore we are going to use 1.19.

Unfortunatelly I could not find supported Docker versions in the Relase Notes for Kubernetes 1.19, so I decided to use docker-ce 19.03.

SELinux set to enforcing mode and firewalld is enabled on all servers.

1 Install and Configure HAProxy Load Balancer with Keepalived
-------------------------------------------------------------

Run these commands on both servers **master-1** and **master-2**.

### 1.1 Configure Firewalld
```
Configure firewall to allow inbound HAProxy traffic on kube-apiserver port:

$ sudo firewall-cmd --permanent --add-port=6443/tcp
$ sudo firewall-cmd --reload

Configure firewall to allow inbound traffic for HAProxy stats:

$ sudo firewall-cmd --permanent --add-port=8080/tcp
$ sudo firewall-cmd --reload
```
Configure firewall to allow VRRP traffic to pass between the keepalived nodes:
```
$ sudo firewall-cmd --permanent --add-rich-rule='rule protocol value="vrrp" accept'
$ sudo firewall-cmd --reload
```
### 1.2 Configure SELinux

Allow HAProxy to listen on kube-apiserver port 6443:
```
$ sudo semanage port -a -t http_cache_port_t 6443 -p tcp
```
### 1.3 Install Packages
```
$ sudo yum install -y haproxy keepalived psmisc
```
### 1.4 Configure HAProxy

Add the following configuration to file `/etc/haproxy/haproxy.cfg`, keeping in mind that our virtual IP address is 192.168.125.89:
```
global
  log /dev/log  local0
  log /dev/log  local1 notice
  stats socket /var/lib/haproxy/stats level admin
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon

defaults
  log global
  mode  http
  option  httplog
  option  dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000

frontend kubernetes
    bind 192.168.125.89:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server master-1-master 192.168.125.84:6443 check fall 3 rise 2
    server master-2-master 192.168.125.87:6443 check fall 3 rise 2

listen stats 192.168.125.89:8080
    mode http
    stats enable
    stats uri /
    stats realm HAProxy Statistics
    stats auth admin:haproxy
```
Enable and start the haproxy service:
```
$ sudo systemctl enable --now haproxy
```
You can access HAProxy stats page by navigating to the following URL: 
`http://192.168.125.89:8080/` 
Username is “admin”, and password is “haproxy”.
`
![](https://www.lisenet.com/wp-content/uploads/2021/01/homelab-haproxy-stats.png)

Note: the screenhost was taken after deploying Kubernetes.

2 Install Docker Packages
-------------------------

Run these commands on all Kubernetes servers.

Install the yum-utils package (which provides the yum-config-manager utility) and set up the stable repository.
```
$ sudo yum install -y yum-utils
$ sudo yum-config-manager \
  --add-repo https://download.docker.com/linux/centos/docker-ce.repo

Install Docker engine:

$ sudo yum install -y \
  docker-ce-19.03.14-3.el7.x86_64 \
  docker-ce-cli-19.03.14-3.el7.x86_64 \
  containerd.io-1.4.3-3.1.el7.x86_64

Start and enable service:

$ sudo systemctl enable --now docker
```
3 Install Kubernetes Packages and Disable Swap
----------------------------------------------

Run these commands on all Kubernetes servers.

Our PXE boot servers don’t have swap configured by default, but in case you do, disable it. Running kubelet with swap on is not supported.

`$ sudo swapoff -a`

Set up the repository. Add the following to `/etc/yum.repos.d/kubernetes.repo`:
```
\[kubernetes\]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
```
We exclude all Kubernetes packages from any system upgrades because they have a special process that has to be followed.

Install kubeadm, kubelet and kubectl:
```
$ sudo yum install -y --disableexcludes=kubernetes \
  kubernetes-cni-0.8.7-0.x86_64 \
  kubelet-1.19.7-0.x86_64 \
  kubectl-1.19.7-0.x86_64 \
  kubeadm-1.19.7-0.x86_64
```
Enable kubelet service:
```
$ sudo systemctl enable --now kubelet
```
Note that the kubelet is now in a crashloop and restarting every few seconds, as it waits for kubeadm to tell it what to do.

4 Configure kubelet Eviction Thresholds
---------------------------------------

Run these commands on all Kubernetes servers.

Unless resources are set aside for system daemons, Kubernetes pods and system daemons will compete for resources and eventually lead to resource starvation issues. Kubelet has the extra args parameter to specify eviction thresholds that trigger the kubelet to reclaim resources.
```
$ echo "KUBELET_EXTRA_ARGS=--eviction-hard=memory.available<256Mi,nodefs.available<1Gi,imagefs.available<1Gi"|sudo tee /etc/sysconfig/kubelet
```
Restart kubelet service:
```
$ sudo systemctl restart kubelet
```
5 Let iptables see Bridged Traffic
----------------------------------

Run these commands on all Kubernetes servers.
```
$ echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee /etc/sysctl.d/k8s-iptables.conf
$ echo "net.bridge.bridge-nf-call-ip6tables=1" | sudo tee /etc/sysctl.d/k8s-ip6tables.conf
$ sudo sysctl --system
```
6 Configure Firewalld
---------------------

### 6.1 Firewall Rules for Control-plane Nodes

Run these commands on Kubernetes control-plane nodes only.

We are going to open the following ports on the control-plane nodes:
```
1.  8443 – Kubernetes API server
2.  2379,2380 – etcd server client API
3.  2381 – etcd metrics API
4.  10250 – Kubelet API
5.  10251 – kube-scheduler
6.  10252 – kube-controller-manager
7.  179 – Calico networking (BGP)
```
```
$ sudo firewall-cmd --permanent --add-port={6443,2379-2381,10250-10252}/tcp
$ sudo firewall-cmd --permanent --add-port=179/tcp
$ sudo firewall-cmd --permanent --add-masquerade
$ sudo firewall-cmd --reload
```
One interesting note here, I kept getting CoreDNS crashes like this one:

_CoreDNS crashes with error “Failed to list v1.Service: Get https://10.96.0.1:443/api/v1/\*\*\*: dial tcp 10.96.0.1:443: connect: no route to host”._

I added masquerade to firewalld and I think it helped fix the problem.

### 6.2 Firewall Rules for Worker Nodes

Run these commands on Kubernetes worker nodes only.

We are going to open the following ports on the worker nodes:
```
1.  10250 – Kubelet API
2.  30000-32767 – NodePort Services
3.  179 – Calico networking (BGP)
```
```
$ sudo firewall-cmd --permanent --add-port={10250,30000-32767}/tcp
$ sudo firewall-cmd --permanent --add-port=179/tcp
$ sudo firewall-cmd --permanent --add-masquerade
$ sudo firewall-cmd --reload
```
7 Initialise the First Control Plane Node
-----------------------------------------

### 7.1 Initialise One Master Node Only

Run these commands on the first control plane node **master-1** only.
```
$ sudo kubeadm init \
  --kubernetes-version "1.19.7" \
  --pod-network-cidr "192.168.0.0/16" \
  --service-dns-domain "apps" \
  --control-plane-endpoint "bastion:6443" \
  --upload-certs
```
Command output (note that your output will be different than what is provided below):
```
_W0116 15:49:53.753185   12133 configset.go:348\] WARNING: kubeadm cannot validate component configs for API groups \[kubelet.config.k8s.io kubeproxy.config.k8s.io\]
\[init\] Using Kubernetes version: v1.19.7
\[preflight\] Running pre-flight checks
	\[WARNING Firewalld\]: firewalld is active, please ensure ports \[6443 10250\] are open or your cluster may not function correctly
	\[WARNING IsDockerSystemdCheck\]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
\[preflight\] Pulling images required for setting up a Kubernetes cluster
\[preflight\] This might take a minute or two, depending on the speed of your internet connection
\[preflight\] You can also perform this action in beforehand using 'kubeadm config images pull'
\[certs\] Using certificateDir folder "/etc/kubernetes/pki"
\[certs\] Generating "ca" certificate and key
\[certs\] Generating "apiserver" certificate and key
\[certs\] apiserver serving cert is signed for DNS names \[bastion kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.apps master-1\] and IPs \[10.96.0.1 192.168.125.84\]
\[certs\] Generating "apiserver-kubelet-client" certificate and key
\[certs\] Generating "front-proxy-ca" certificate and key
\[certs\] Generating "front-proxy-client" certificate and key
\[certs\] Generating "etcd/ca" certificate and key
\[certs\] Generating "etcd/server" certificate and key
\[certs\] etcd/server serving cert is signed for DNS names \[localhost master-1\] and IPs \[192.168.125.84 127.0.0.1 ::1\]
\[certs\] Generating "etcd/peer" certificate and key
\[certs\] etcd/peer serving cert is signed for DNS names \[localhost master-1\] and IPs \[192.168.125.84 127.0.0.1 ::1\]
\[certs\] Generating "etcd/healthcheck-client" certificate and key
\[certs\] Generating "apiserver-etcd-client" certificate and key
\[certs\] Generating "sa" key and public key
\[kubeconfig\] Using kubeconfig folder "/etc/kubernetes"
\[kubeconfig\] Writing "admin.conf" kubeconfig file
\[kubeconfig\] Writing "kubelet.conf" kubeconfig file
\[kubeconfig\] Writing "controller-manager.conf" kubeconfig file
\[kubeconfig\] Writing "scheduler.conf" kubeconfig file
\[kubelet-start\] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
\[kubelet-start\] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
\[kubelet-start\] Starting the kubelet
\[control-plane\] Using manifest folder "/etc/kubernetes/manifests"
\[control-plane\] Creating static Pod manifest for "kube-apiserver"
\[control-plane\] Creating static Pod manifest for "kube-controller-manager"
\[control-plane\] Creating static Pod manifest for "kube-scheduler"
\[etcd\] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
\[wait-control-plane\] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
\[apiclient\] All control plane components are healthy after 32.032492 seconds
\[upload-config\] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
\[kubelet\] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
\[upload-certs\] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
\[upload-certs\] Using certificate key:
c4161499f1f614322ea788831b4f72529175712c7c6f8888cdc14f5aab83fbce
\[mark-control-plane\] Marking the node master-1 as control-plane by adding the label "node-role.kubernetes.io/master=''"
\[mark-control-plane\] Marking the node master-1 as control-plane by adding the taints \[node-role.kubernetes.io/master:NoSchedule\]
\[bootstrap-token\] Using token: 8rd7kq.kcg3bkzkzdus8v54
\[bootstrap-token\] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
\[bootstrap-token\] configured RBAC rules to allow Node Bootstrap tokens to get nodes
\[bootstrap-token\] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
\[bootstrap-token\] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
\[bootstrap-token\] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
\[bootstrap-token\] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
\[kubelet-finalize\] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
\[addons\] Applied essential addon: CoreDNS
\[addons\] Applied essential addon: kube-proxy
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
You should now deploy a pod network to the cluster.
Run "kubectl apply -f \[podnetwork\].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:
```
  kubeadm join bastion:6443 --token 8rd7kq.kcg3bkzkzdus8v54 \
    --discovery-token-ca-cert-hash sha256:4efb75f7e09c0a2a9db0d317dd40ac9cf9906c31109a670428d0d49981264904 \
    --control-plane --certificate-key c4161499f1f614322ea788831b4f72529175712c7c6f8888cdc14f5aab83fbce
```
Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
`"kubeadm init phase upload-certs --upload-certs" `
to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:
```
kubeadm join bastion:6443 --token 8rd7kq.kcg3bkzkzdus8v54 \
    --discovery-token-ca-cert-hash sha256:4efb75f7e09c0a2a9db0d317dd40ac9cf9906c31109a670428d0d49981264904_ 
```
### 7.2 Configure Kube Config on Your Local Machine

Run the following commands on your local machine (e.g. your laptop):
```
$ mkdir -p $HOME/.kube
$ scp master-1:/etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### 7.3 Install Calico Pod Network

Run the following commands on your local machine (e.g. your laptop) where you have kubectl configured.
```
$ kubectl apply -f https://docs.projectcalico.org/archive/v3.17/manifests/calico.yaml
```
Make sure that the control-plane node status is ready:
```
$ kubectl get nodes
NAME    STATUS   ROLES    AGE   VERSION
master-1   Ready    master   10m   v1.19.7
```
8 Join Other Control Plane Nodes to the Cluster
-----------------------------------------------

Run the following command on control-plane nodes **master-2**

Note that your actual command will be different than what is provided below!
```
$ sudo kubeadm join bastion:6443 --token 8rd7kq.kcg3bkzkzdus8v54 \
  --discovery-token-ca-cert-hash sha256:4efb75f7e09c0a2a9db0d317dd40ac9cf9906c31109a670428d0d49981264904 \
  --control-plane --certificate-key c4161499f1f614322ea788831b4f72529175712c7c6f8888cdc14f5aab83fbce
```
The output should contain the following lines:

_\[...\]
This node has joined the cluster and a new control plane instance was created:

\* Certificate signing request was sent to apiserver and approval was received.
\* The Kubelet was informed of the new secure connection details.
\* Control plane (master) label and taint were applied to the new node.
\* The Kubernetes control plane instances scaled up.
\* A new etcd member was added to the local/stacked etcd cluster.
\[...\]_

9 Join Worker Nodes to the Cluster
----------------------------------

Run the following command on worker nodes **worker-1**, **worker-2** and **srv36**.

Note that your actual command will be different than what is provided below!

$ sudo kubeadm join bastion:6443 --token 8rd7kq.kcg3bkzkzdus8v54 \
  --discovery-token-ca-cert-hash sha256:4efb75f7e09c0a2a9db0d317dd40ac9cf9906c31109a670428d0d49981264904 

The output should contain the following lines:

_\[...\]
This node has joined the cluster:
\* Certificate signing request was sent to apiserver and a response was received.
\* The Kubelet was informed of the new secure connection details.
\[...\]_

10 Verify the Cluster
---------------------

To verify, run the following commands on your local machine where you have kubectl configured.
`$ kubectl get nodes -o wide`
```
NAME    STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
master-1   Ready    master   6h16m   v1.19.7   192.168.125.84    none          CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.14
master-2   Ready    master   6h2m    v1.19.7   192.168.125.87    none          CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.14
worker-1   Ready    none     5h55m   v1.19.7   192.168.125.88    none          CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.14
worker-2   Ready    none     5h52m   v1.19.7   192.168.125.90    none          CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.14
```
`$ kubectl get all -A`
```
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
kube-system   pod/calico-kube-controllers-744cfdf676-thmgj   1/1     Running   0          6h3m
kube-system   pod/calico-node-2gjtd                          1/1     Running   0          5h55m
kube-system   pod/calico-node-cpx67                          1/1     Running   0          5h52m
kube-system   pod/calico-node-p57h8                          1/1     Running   0          5h59m
kube-system   pod/calico-node-qkn55                          1/1     Running   0          5h46m
kube-system   pod/calico-node-rg8nz                          1/1     Running   0          6h3m
kube-system   pod/calico-node-xwrth                          1/1     Running   0          5h49m
kube-system   pod/coredns-f9fd979d6-6mx72                    1/1     Running   0          6h13m
kube-system   pod/coredns-f9fd979d6-vszn8                    1/1     Running   0          6h13m
kube-system   pod/etcd-master-1                                 1/1     Running   0          6h13m
kube-system   pod/etcd-master-2                                 1/1     Running   0          5h59m
kube-system   pod/kube-apiserver-master-1                       1/1     Running   0          6h13m
kube-system   pod/kube-apiserver-master-2                       1/1     Running   0          5h59m
kube-system   pod/kube-controller-manager-master-2              1/1     Running   0          5h59m
kube-system   pod/kube-proxy-8dpxx                           1/1     Running   0          5h49m
kube-system   pod/kube-proxy-jk5zw                           1/1     Running   0          5h52m
kube-system   pod/kube-proxy-mv6k9                           1/1     Running   0          5h46m
kube-system   pod/kube-proxy-n5pxd                           1/1     Running   0          5h55m
kube-system   pod/kube-proxy-rbhqp                           1/1     Running   0          6h13m
kube-system   pod/kube-scheduler-master-1                       1/1     Running   1          6h13m
kube-system   pod/kube-scheduler-master-2                       1/1     Running   0          5h59m

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    none          443/TCP                  6h13m
kube-system   service/kube-dns     ClusterIP   10.96.0.10   none          53/UDP,53/TCP,9153/TCP   6h13m

NAMESPACE     NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/calico-node   6         6         6       6            6           kubernetes.io/os=linux   6h3m
kube-system   daemonset.apps/kube-proxy    6         6         6       6            6           kubernetes.io/os=linux   6h13m

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           6h3m
kube-system   deployment.apps/coredns                   2/2     2            2           6h13m

NAMESPACE     NAME                                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/calico-kube-controllers-744cfdf676   1         1         1       6h3m
kube-system   replicaset.apps/coredns-f9fd979d6                    2         2         2       6h13m
```
This concludes the Kubernetes homelab cluster installation using kubeadm.
