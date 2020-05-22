## 如何使用centos7安装kubernetes 

```bash
[root@localhost ~]# hostnamectl set-hostname  k8s-master
[root@k8s-master ~]# reboot
[root@k8s-master ~]# yum -y install policycoreutils-python*
[root@k8s-master ~]# wget http://ftp.riken.jp/Linux/cern/centos/7/extras/x86_64/Packages/container-selinux-2.68-1.el7.noarch.rpm
[root@k8s-master ~]# rpm -ivh container-selinux-2.68-1.el7.noarch.rpm
准备中...                          ################################# [100%]
正在升级/安装...
   1:container-selinux-2:2.68-1.el7   ################################# [100%]
[root@k8s-master ~]# yum install -y libltdl.so* pigz* bridge-utils*
[root@k8s-master ~]# curl -sSL https://get.docker.com/ | sudo sh
[root@k8s-master ~]# vim /etc/yum.repos.d/kubernetes.repo
[root@k8s-master ~]# cat /etc/yum.repos.d/kubernetes.repo
[kuberneten]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

[root@k8s-master ~]# yum makecache
[root@k8s-master ~]# swapoff -a
[root@k8s-master ~]# sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
[root@k8s-master ~]# echo "net.bridge.bridge-nf-call-iptables=1">> /etc/sysctl.conf
[root@k8s-master ~]# echo "1" > /proc/sys/net/ipv4/ip_forward
[root@k8s-master ~]# modprobe br_netfilter
[root@k8s-master ~]# sysctl -p
[root@k8s-master ~]# yum install -y kubelet kubeadm kubectl kubernetes-cni
[root@k8s-master ~]# systemctl enable docker && systemctl start docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
[root@k8s-master ~]# systemctl enable kubelet && systemctl start kubelet
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /usr/lib/systemd/system/kubelet.service.
[root@k8s-master ~]# 
[root@k8s-master ~]# kubeadm config images list  //查看所需镜像，并下载相应版本
W0802 14:50:44.611344    1866 version.go:98] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
W0802 14:50:44.611436    1866 version.go:99] falling back to the local client version: v1.15.1
k8s.gcr.io/kube-apiserver:v1.15.1
k8s.gcr.io/kube-controller-manager:v1.15.1
k8s.gcr.io/kube-scheduler:v1.15.1
k8s.gcr.io/kube-proxy:v1.15.1
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
[root@k8s-master ~]# docker pull k8s.gcr.io/kube-apiserver:v1.15.1 && docker pull k8s.gcr.io/kube-controller-manager:v1.15.1 && docker pull k8s.gcr.io/kube-scheduler:v1.15.1 && docker pull k8s.gcr.io/kube-proxy:v1.15.1 && docker pull k8s.gcr.io/pause:3.1 && docker pull k8s.gcr.io/etcd:3.3.10 && docker pull k8s.gcr.io/coredns:1.3.1
[root@k8s-master ~]# kubeadm init   --pod-network-cidr=10.244.0.0/16
[root@k8s-master ~]# mkdir -p $HOME/.kube
[root@k8s-master ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@k8s-master ~]# chown $(id -u):$(id -g) $HOME/.kube/config
[root@k8s-master ~]# kubectl taint nodes --all node-role.kubernetes.io/master-  //Master节点默认不参与工作负载，需执行命令来解除限制
node/k8s-master untainted
[root@k8s-master ~]# docker pull quay.io/coreos/flannel:v0.10.0-amd64  //安装cni 网络插件
v0.10.0-amd64: Pulling from coreos/flannel
ff3a5c916c92: Pull complete 
8a8433d1d437: Pull complete 
306dc0ee491a: Pull complete 
856cbd0b7b9c: Pull complete 
af6d1e4decc6: Pull complete 
Digest: sha256:88f2b4d96fae34bfff3d46293f7f18d1f9f3ca026b4a4d288f28347fcb6580ac
Status: Downloaded newer image for quay.io/coreos/flannel:v0.10.0-amd64
[root@k8s-master ~]# 
[root@k8s-master ~]# mkdir -p /etc/cni/net.d/
[root@k8s-master ~]# vi /etc/cni/net.d/10-flannel.conf
[root@k8s-master ~]# cat /etc/cni/net.d/10-flannel.conf
{"name":"cbr0","type":"flannel","delegate": {"isDefaultGateway": true}}
[root@k8s-master ~]# mkdir /usr/share/oci-umount/oci-umount.d -p
[root@k8s-master ~]# mkdir /run/flannel/
[root@k8s-master ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds created
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   8m14s   v1.15.1
[root@k8s-master ~]# vim dig.yml  //测试coredns功能
[root@k8s-master ~]# cat dig.yml 
apiVersion: v1
kind: Pod
metadata:
  name: dig
  namespace: default
spec:
  containers:
  - name: dig
    image:  docker.io/azukiapp/dig
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
[root@k8s-master ~]# kubectl apply -f dig.yml
pod/dig created
[root@k8s-master ~]# 
[root@k8s-master ~]# 
[root@k8s-master ~]# kubectl get pods
NAME   READY   STATUS    RESTARTS   AGE
dig    1/1     Running   0          49s
[root@k8s-master ~]# kubectl exec -ti dig -- nslookup kubernetes  //如有以下返回说明dns安装成功
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	kubernetes.default.svc.cluster.local
Address: 10.96.0.1


```

如需加入节点：
```bash
[root@localhost ~]# hostnamectl set-hostname  k8s-node1
[root@k8s-node1 ~]# reboot
[root@k8s-node1 ~]# yum -y install policycoreutils-python*
[root@k8s-node1 ~]# wget http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.68-1.el7.noarch.rpm
[root@k8s-node1 ~]# rpm -ivh container-selinux-2.68-1.el7.noarch.rpm
准备中...                          ################################# [100%]
正在升级/安装...
   1:container-selinux-2:2.68-1.el7   ################################# [100%]
[root@k8s-node1 ~]# yum install -y libltdl.so* pigz* bridge-utils*
[root@k8s-node1 ~]# curl -sSL https://get.docker.com/ | sudo sh
[root@k8s-node1 ~]# vim /etc/yum.repos.d/kubernetes.repo
[root@k8s-node1 ~]# cat /etc/yum.repos.d/kubernetes.repo
[kuberneten]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

[root@k8s-node1 ~]# yum makecache
[root@k8s-node1 ~]# swapoff -a
[root@k8s-node1 ~]# sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
[root@k8s-node1 ~]# echo "net.bridge.bridge-nf-call-iptables=1">> /etc/sysctl.conf
[root@k8s-node1 ~]# echo "1" > /proc/sys/net/ipv4/ip_forward
[root@k8s-node1 ~]# modprobe br_netfilter
[root@k8s-node1 ~]# sysctl -p
[root@k8s-node1 ~]# yum install -y kubelet kubeadm kubectl kubernetes-cni
[root@k8s-node1 ~]# systemctl enable docker && systemctl start docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
[root@k8s-node1 ~]# systemctl enable kubelet && systemctl start kubelet
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /usr/lib/systemd/system/kubelet.service.
[root@k8s-node1 ~]# 
[root@k8s-node1 ~]# mkdir -p /etc/cni/net.d/
[root@k8s-node1 ~]# vi /etc/cni/net.d/10-flannel.conf
[root@k8s-node1 ~]# cat /etc/cni/net.d/10-flannel.conf
{"name":"cbr0","type":"flannel","delegate": {"isDefaultGateway": true}}

```
在master上查看token：
```bash
[root@k8s-master ~]# kubeadm token list  //token已过期
TOKEN                     TTL         EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
giluwm.lzwt5hgbkxjr42wb   <invalid>   2019-08-06T13:45:14+08:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
[root@k8s-master ~]# 
[root@k8s-master ~]# 
[root@k8s-master ~]# kubeadm  token create  //重新生成
igpl9v.9sfix2xeu9lmekrd
```
在node上继续执行：
```bash
[root@k8s-node1 ~]# kubeadm join --token=igpl9v.9sfix2xeu9lmekrd 192.168.47.146:6443 --discovery-token-unsafe-skip-ca-verification

```
查看错误：
```bash
[root@localhost ~]# journalctl -f -u kubelet
```
