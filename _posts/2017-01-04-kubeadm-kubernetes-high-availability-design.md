---
layout: post
title: kubernetes集群高可用架构及实现
categories: DevOps
description: kubernetes集群高可用架构及实现
keywords: Kubernetes, kubeadm
---

本文是基于kubeadm方式部署的kubernetes集群高可用架构设计。阅读对象为熟悉kubernetes及kubeadm基本概念的同学。

### 说明
本方案基于 kubernetes v1.4.5 版本，理论上kubernetes v1.4版本以上都应该适用。
实验环境：CentOS 7.2

### 阅读对象
阅读对象为熟悉kubernetes和kubeadm基本概念的同学。

# 1 背景说明
从kubernetes 1.4版本开始，kubeadm部署工具的出现使得部署一个单master节点的kubernetes集群变得易如反掌，简单来说，在准备好集群依赖的docker环境、相关组件安装的rpm包及docker镜像包的情况下，接下来只需执行两条命令（在master节点执行kubeadm init命令初始化集群、生成token，然后在node节点逐个执行kubeadm join命令加入集群），这样分分钟就能把一个kubernetes集群搭建起来，最后部署Calico/Canal/Romana/Weave等插件形式的容器网络方案，该kubernetes集群基本上就能投入小规模使用环境。

然而另一方面，基于kubeadm部署的kubernetes集群，其组件的存在形式跟以前相比发生了变化。例如master节点的一些关键组件，本来是以进程的形式存在，包括kube-apiserver、kube-dns、kube-discovery、kube-control-manager、kube-scheduler、kubernetes-cni、kube-proxy，现在这些组件则是以Pod的形式纳入kubernetes集群自己管理。此外，目前来看，不管是1.4以前版本繁琐的安装形式，还是现在通过kubeadm的简化安装，其集群的master节点始终是一个单点，这无疑是个瓶颈。

从整个kubernetes集群的运维层面来看，master节点作为整个集群中负责管理和调度集群整体资源的重要基础设施，只要master节点及其组件实现了高可用，那么整个kubernetes集群基本上就实现了高可用。

# 2 设计目标
实现master节点组件的高可用；
实现master节点的高可用。

# 3 具体方案
## 3.1 集群规划
| 服务器  | IP | 用途 | 
|:----|:----:|:----:|
| 10-10-102-91.master | 10.10.102.91 | k8s master1 | 
| 10-10-102-92.master | 10.10.102.92 | k8s master2 |
| 10-10-102-93.master | 10.10.102.93 | k8s master3 |
| 10-10-103-96.node | 10.10.103.96 | k8s node1, etcd01 |
| 10-10-103-97.node | 10.10.103.97 | k8s node2, etcd02 |
| VIP | 10.10.102.21 |  |
确保VIP和master节点集群在同一网段。

## 3.2 搭建etcd集群
搭建好etcd集群之后，配置：
[root@localhost etcd]# vi /etc/etcd/etcd.conf
```
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd/etcd01"
ETCD_LISTEN_PEER_URLS="http://10.10.103.96:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.103.96:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.103.96:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.103.96:2379"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="huan-etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd01=http://10.10.103.96:2380,etcd02=http://10.10.103.97:2380"

```
关闭防火墙：
```
[root@localhost etcd]# iptables -F
```
重启etcd：
```
[root@localhost etcd]# systemctl restart etcd
```
查看集群：
```
[root@localhost etcd]# etcdctl member list
521e455ec3a5e123: name=etcd01 peerURLs=http://10.10.103.96:2380 clientURLs=http://10.10.103.96:2379 isLeader=false
bead30a31455c82b: name=etcd02 peerURLs=http://10.10.103.97:2380 clientURLs=http://10.10.103.97:2379 isLeader=true
```

不过这里有个坑，尽量不要在master集群所在的几个节点创建etcd。
因为kubeadm init的时候会去检查当前节点是否含有/etc/kubernetes目录以及/var/lib/etcd目录，如有这些目录会提示删除，否则init不成功。
如果一不小心把etcd集群建在了几个master节点，那么解决办法是修改etcd安装目录。
kubernetes的所有信息都存在这里创建的etcd。


## 3.3 搭建kubernetes集群
### 3.3.1 准备工作
包括：
各节点单独设置hostname：
设置/etc/hostname：
```
hostnamectl --static set-hostname  10-10-102-91.master
hostnamectl --static set-hostname  10-10-102-92.master
hostnamectl --static set-hostname  10-10-102-93.master
hostnamectl --static set-hostname  10-10-103-96.node
hostnamectl --static set-hostname  10-10-103-97.node
```
设置/etc/hosts：
```
echo '10.10.102.91 10-10-102-91.master
10.10.102.92 10-10-102-92.master
10.10.102.93 10-10-102-93.master
10.10.103.96 10-10-103-96.node
10.10.103.97 10-10-103-97.node'>> /etc/hosts
```
或者直接 vi /etc/hosts 改成：
```
127.0.0.1    10-10-102-91.master
::1          10-10-102-91.master
10.10.102.91 10-10-102-91.master
10.10.102.92 10-10-102-92.master
10.10.102.93 10-10-102-93.master
10.10.103.96 10-10-103-96.node
10.10.103.97 10-10-103-97.node
```

安装docker环境；加载kubernetes相关组件的docker镜像；安装kubernetes组件的相关rpm安装包；安装docker环境，加载，安装kubernetes相关组件的rpm包：
在master1部署二进制包：
```
cat <<EOF> /etc/yum.repos.d/k8s.repo
[kubelet]
name=kubelet
baseurl=http://files.rm-rf.ca/rpms/kubelet/
enabled=1
gpgcheck=0
EOF

yum makecache
yum install -y socat kubelet kubeadm kubectl kubernetes-cni
```
rpm安装方式（建议）：
```
拷贝安装包：
share@share-ThinkCentre-E73:~/下载$ scp k8s-v1.4.5-rpms.tar.gz root@10.10.102.91:/root/
安装：
cd k8s-v1.4.5-rpms
yum localinstall *.rpm -y
包含：
socat-1.7.2.2-5.el7.x86_64.rpm
kubeadm-1.5.0-0.alpha.1.409.714f816a349e79.x86_64.rpm
kubectl-1.4.1-0.x86_64.rpm
kubelet-1.4.1-0.x86_64.rpm
kubernetes-cni-0.3.0.1-0.07a8a2.x86_64.rpm
```

在master1安装docker:
```
tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

yum install docker-engine -y
```

rpm安装方式（建议）：
从官方镜像仓库下载：
https://yum.dockerproject.org/repo/main/centos/7/Packages/
```
拷贝安装包：
share@share-ThinkCentre-E73:~/下载$ scp docker-engine-1.12.4-1.el7.centos.x86_64.rpm  root@10.10.102.91:/root/
安装：
cd ~
yum localinstall docker-engine-1.12.4-1.el7.centos.x86_64.rpm -y
```
systemctl enable docker
systemctl start docker
systemctl status docker

在master1 load镜像：
tar zcvf kubernates-v1.4.5-images.tar.gz kubernates-v1.4.5-images/
cd kubernates-v1.4.5-images
```
tars=(etcd-amd64.tar exechealthz-amd64.tar  kube-discovery-amd64.tar kubedns-amd64.tar kube-dnsmasq-amd64.tar kube-proxy-amd64.tar  pause-amd64.tar kube-apiserver-amd64.tar kube-controller-manager-amd64.tar kube-scheduler-amd64.tar kubernetes-dashboard-amd64.tar)
for tar in ${tars[@]} ; do
  docker load -i $tar
done
```

在master1启动kubelet
systemctl enable kubelet
systemctl start kubelet


### 3.3.2 初始化master1节点
包括：绑定VIP；kubeadm init初始化master1，指定VIP地址，指定etcd地址，指定kubernetes版本号；记录token

给master1节点绑定VIP
```
#绑定
ip addr add 10.10.102.21/24 dev eth0
或 
ifconfig eth0:0 10.10.102.21 netmask 255.255.255.0 up (推荐）

#停止：
sudo ifconfig eth0:0 down
```


master1节点初始化：
```
kubeadm init --api-advertise-addresses=10.10.102.21 --external-etcd-endpoints=http://10.10.103.96:2379,http://10.10.103.97:2379 --use-kubernetes-version v1.4.5 
```

如果遇到错误
```
/etc/kubernetes is not empty
/var/lib/etcd is not empty
```
则执行
```
rm -rf /etc/kubernetes
rm -rf /var/lib/etcd
```

如果遇到错误
```
Running pre-flight checks
preflight check errors:
	Port 2379 is in use
```
是因为前面load了etcd镜像，而这里我们将使用外部的etcd集群，因此需要把该镜像remove
执行：
```
docker rmi etcd_imageid
```
然后重新init即可

以下是init成功的日志，记录token：
```
[root@10-10-102-91 ~]# kubeadm init --api-advertise-addresses=10.10.102.21 --external-etcd-endpoints=http://10.10.103.96:2379,http://10.10.103.97:2379 --use-kubernetes-version v1.4.5 
Flag --external-etcd-endpoints has been deprecated, this flag will be removed when componentconfig exists
Running pre-flight checks
WARNING: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
<master/tokens> generated token: "6c7750.6c759069627a3c78"
<master/pki> generated Certificate Authority key and certificate:
Issuer: CN=kubernetes | Subject: CN=kubernetes | CA: true
Not before: 2016-12-27 04:23:53 +0000 UTC Not After: 2026-12-25 04:23:53 +0000 UTC
Public: /etc/kubernetes/pki/ca-pub.pem
Private: /etc/kubernetes/pki/ca-key.pem
Cert: /etc/kubernetes/pki/ca.pem
<master/pki> generated API Server key and certificate:
Issuer: CN=kubernetes | Subject: CN=kube-apiserver | CA: false
Not before: 2016-12-27 04:23:53 +0000 UTC Not After: 2017-12-27 04:23:53 +0000 UTC
Alternate Names: [10.10.102.21 10.0.0.1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local]
Public: /etc/kubernetes/pki/apiserver-pub.pem
Private: /etc/kubernetes/pki/apiserver-key.pem
Cert: /etc/kubernetes/pki/apiserver.pem
<master/pki> generated Service Account Signing keys:
Public: /etc/kubernetes/pki/sa-pub.pem
Private: /etc/kubernetes/pki/sa-key.pem
<master/pki> created keys and certificates in "/etc/kubernetes/pki"
<util/kubeconfig> created "/etc/kubernetes/kubelet.conf"
<util/kubeconfig> created "/etc/kubernetes/admin.conf"
<master/apiclient> created API client configuration
<master/apiclient> created API client, waiting for the control plane to become ready
<master/apiclient> all control plane components are healthy after 14.911231 seconds
<master/apiclient> waiting for at least one node to register and become ready
<master/apiclient> first node is ready after 4.006288 seconds
<master/apiclient> attempting a test deployment
<master/apiclient> test deployment succeeded
<master/discovery> created essential addon: kube-discovery, waiting for it to become ready
<master/discovery> kube-discovery is ready after 3.506789 seconds
<master/addons> created essential addon: kube-proxy
<master/addons> created essential addon: kube-dns

Kubernetes master initialised successfully!

You can now join any number of machines by running the following on each node:

kubeadm join --token=6c7750.6c759069627a3c78 10.10.102.21

```



### 3.3.3  搭建其他master节点
将master1节点初始化的/etc/kubernetes目录复制到其他master节点即可。
按照以上顺序安装master2节点以及master3节点
```
#复制master1 的/etc/kubernetes/并启动kubelet
[root@10-10-102-91 ~]# scp -r /etc/kubernetes/* root@10.10.102.92:/etc/kubernetes/
[root@10-10-102-91 ~]# scp -r /etc/kubernetes/* root@10.10.102.93:/etc/kubernetes/


```
### 3.3.4  部署网络方案


### 3.3.5  加入node节点
node节点使用kubeadm join加入集群
```
[root@10-10-103-96 ~]# kubeadm join --token=6c7750.6c759069627a3c78 10.10.102.21

```

遇到错误：
```
<util/tokens> validating provided token
<node/discovery> created cluster info discovery client, requesting info from "http://10.10.102.21:9898/cluster-info/v1/?token-id=6c7750"
<node/discovery> failed to request cluster info, will try again: [Get http://10.10.102.21:9898/cluster-info/v1/?token-id=6c7750: dial tcp 10.10.102.21:9898: getsockopt: no route to host]
<node/discovery> failed to request cluster info, will try again: [Get http://10.10.102.21:9898/cluster-info/v1/?token-id=6c7750: dial tcp 10.10.102.21:9898: getsockopt: no route to host]

```
解决办法：关闭selinux防火墙

```
setenforce 0
systemctl stop firewalld
systemctl disable firewalld
```

最后看到成功日志：
```
[root@10-10-103-96 ~]# kubeadm join --token=6c7750.6c759069627a3c78 10.10.102.21
Running pre-flight checks
<util/tokens> validating provided token
<node/discovery> created cluster info discovery client, requesting info from "http://10.10.102.21:9898/cluster-info/v1/?token-id=6c7750"
<node/discovery> failed to request cluster info, will try again: [Get http://10.10.102.21:9898/cluster-info/v1/?token-id=6c7750: dial tcp 10.10.102.21:9898: getsockopt: no route to host]
<node/discovery> failed to request cluster info, will try again: [Get http://10.10.102.21:9898/cluster-info/v1/?token-id=6c7750: dial tcp 10.10.102.21:9898: getsockopt: no route to host]
<node/discovery> failed to request cluster info, will try again: [Get http://10.10.102.21:9898/cluster-info/v1/?token-id=6c7750: dial tcp 10.10.102.21:9898: getsockopt: no route to host]
<node/discovery> failed to request cluster info, will try again: [Get http://10.10.102.21:9898/cluster-info/v1/?token-id=6c7750: dial tcp 10.10.102.21:9898: getsockopt: no route to host]
<node/discovery> failed to request cluster info, will try again: [Get http://10.10.102.21:9898/cluster-info/v1/?token-id=6c7750: dial tcp 10.10.102.21:9898: getsockopt: no route to host]
<node/discovery> cluster info object received, verifying signature using given token
<node/discovery> cluster info signature and contents are valid, will use API endpoints [https://10.10.102.21:6443]
<node/bootstrap> trying to connect to endpoint https://10.10.102.21:6443
<node/bootstrap> detected server version v1.4.5
<node/bootstrap> successfully established connection with endpoint https://10.10.102.21:6443
<node/csr> created API client to obtain unique certificate for this node, generating keys and certificate signing request
<node/csr> received signed certificate from the API server:
Issuer: CN=kubernetes | Subject: CN=system:node:10-10-103-96.node | CA: false
Not before: 2016-12-29 11:56:00 +0000 UTC Not After: 2017-12-29 11:56:00 +0000 UTC
<node/csr> generating kubelet configuration
<util/kubeconfig> created "/etc/kubernetes/kubelet.conf"

Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.

```

全部节点加入后，看到：
```
[root@10-10-102-91 ~]# kubectl get nodes
NAME                  STATUS    AGE
10-10-102-91.master   Ready     2d
10-10-102-92.master   Ready     2d
10-10-102-93.master   Ready     2d
10-10-103-96.node     Ready     2m
10-10-103-97.node     Ready     14s

```
全部节点加入后，查看集群pod情况：
```
[root@10-10-102-91 ~]# kubectl get pods -n kube-system
NAME                                          READY     STATUS    RESTARTS   AGE
dummy-2088944543-96asu                        1/1       Running   0          2d
kube-apiserver-10-10-102-91.master            1/1       Running   0          2d
kube-apiserver-10-10-102-92.master            1/1       Running   0          2d
kube-apiserver-10-10-102-93.master            1/1       Running   0          2d
kube-controller-manager-10-10-102-91.master   1/1       Running   1          2d
kube-controller-manager-10-10-102-92.master   1/1       Running   0          2d
kube-controller-manager-10-10-102-93.master   1/1       Running   0          2d
kube-discovery-982812725-f1694                1/1       Running   0          2d
kube-dns-2247936740-unfa9                     3/3       Running   0          2d
kube-proxy-amd64-cn4i6                        1/1       Running   0          2d
kube-proxy-amd64-i2pdh                        1/1       Running   0          2d
kube-proxy-amd64-u95su                        1/1       Running   0          7m
kube-proxy-amd64-xt5n2                        1/1       Running   0          2d
kube-proxy-amd64-yfhjn                        1/1       Running   0          11m
kube-scheduler-10-10-102-91.master            1/1       Running   0          2d
kube-scheduler-10-10-102-92.master            1/1       Running   0          2d
kube-scheduler-10-10-102-93.master            1/1       Running   0          2d
weave-net-g8dhf                               2/2       Running   3          3m
weave-net-nrdsp                               2/2       Running   0          11m
weave-net-qsjgn                               2/2       Running   0          7m
weave-net-xmtlu                               2/2       Running   2          2m
weave-net-yyj45                               2/2       Running   3          1m
```

### 3.3.6  各节点添加label
目的：这样可以指定组件kube-dns、kube-discovery仅部署在master节点，logapi应用则部署在node节点上。
由于这里我们有多个master节点，集群会将多个master节点也看作是node节点，因此如果不给node节点约定一个label的话，后面部署应用时，必然会出现应用的部分Pod会调度到master节点而部署不成功或者不妥的情况。

```
[root@10-10-102-91 ~]# kubectl label node 10-10-102-91.node kubeadm.alpha.kubernetes.io/role=master
[root@10-10-102-91 ~]# kubectl label node 10-10-102-92.node kubeadm.alpha.kubernetes.io/role=master
[root@10-10-102-91 ~]# kubectl label node 10-10-102-93.node kubeadm.alpha.kubernetes.io/role=master
[root@10-10-102-91 ~]# kubectl label node 10-10-103-96.node kubeadm.alpha.kubernetes.io/role=node
[root@10-10-102-91 ~]# kubectl label node 10-10-103-97.node kubeadm.alpha.kubernetes.io/role=node
```

### 3.3.7 改造kube-dns组件
kube-dns组件主要包括两部分：kube-dns和dnsmasq。
我在安装调试时，查看了其组件日志，得知kube-dns主要是从kubernetes master中不停地监听变动的service和endpoint信息，并建立从service ip到service name的映射关系，对于没有service的则建立pod ip和相应域名的映射。
kube-dns将这些信息存放在本地缓存，并监听127.0.0.1:10053提供服务。
dnsmasq则是一个简单的域名服务、缓存和转发工具。这里主要利用它的转发功能将kube-dns的dns服务转接到外部，参数-server=127.0.0.1:10053。

方案1：Deployment方式
步骤：
1)导出kube-dns组件的原yaml文件；
```
kubectl get deployment kube-dns -n kube-system -o yaml > kube-dns-deployment.yaml
```

2)改变其nodeSelector
```
nodeSelector:
        #beta.kubernetes.io/arch: amd64
        kubeadm.alpha.kubernetes.io/role: master
```
3)删除原先kube-dns的Deployment
```
kubectl delete deployment kube-dns -n kube-system
```	
4)部署改造后的kube-dns组件
```
kubectl apply -f kube-dns-deployment.yaml
```
5)对kube-dns组件扩容
这里按master节点个数，对其进行扩容
```
kubectl scale deployment kube-dns -n kube-system --replicas=3
```
改造后可以看到，新的kube-dns组件部署在各master节点。

方案2：DaemonSet方式
步骤：
1)导出kube-dns组件的原yaml文件；
```
kubectl get deployment kube-dns -n kube-system -o yaml > kube-dns-daemonset.yaml
```

2)改造为DaemonSet
主要是把Deployment改成DaemonSet、改变其nodeSelector、删除DaemonSet不需要的内容如：Strategy、Replica、Status等等。
实际改造的时候，有不少坑，可按照下面重新部署时给出的提示作相应调整即可。

3)删除原先kube-dns的Deployment
```
kubectl delete deployment kube-dns -n kube-system
```	
4)部署改造后的kube-dns组件
```
kube-dns-daemonset.yaml
```
实际上面改造DaemonSet的时候，可以参考这里部署时命令行返回的提示，根据提示内容作相应的调整即可。

改造后可以看到，新的kube-dns组件以DaemonSet的形式部署在各master节点。

### 3.3.8 改造kube-discovery组件
kube-discovery 主要负责集群密钥的分发,如果这个组件不正常, 将无法正常新增节点kubeadm join；
kube-discovery组件可以参考上面kube-dns也改造成DaemonSet的形式。
不过，看了它的yaml文件后，发现它已经默认设定了一个以下的annotation:
```
annotations:
        scheduler.alpha.kubernetes.io/affinity: '{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchExpressions":[{"key":"kubeadm.alpha.kubernetes.io/role","operator":"In","values":["master"]}]}]}}}'
```
其效果等同于指定nodeSelector为role=master，即默认已将kube-discovery所运行的Pod限制为必须在master节点运行。
粗略看了其源码，其设计初衷：
由于kube-discovery的主要功能是证书及token等配置的管理与分发，并且后续的node节点加入时只需要一个简单的master ip信息，因此将kube-discovery限制到了master节点运行，以此统一服务的入口。

因此这里也可以不改造。
对其扩容，指定replicas=3，即可看到其均匀分布在各master节点。

### 3.3.9 kube-controller-manager组件和kube-scheduler组件
kube-controller-manager和/kube-scheduler已经通过leader-elect实现了分布式锁，所以3个Master节点可以正常工作。
[root@10-10-102-92 manifests]# vi /etc/kubernetes/manifests/kube-controller-manager.json 
```
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "kube-controller-manager",
    "namespace": "kube-system",
    "creationTimestamp": null,
    "labels": {
      "component": "kube-controller-manager",
      "tier": "control-plane"
    }
  },
  "spec": {
    "volumes": [
     略...
    ],
    "containers": [
      {
        "name": "kube-controller-manager",
        "image": "gcr.io/google_containers/kube-controller-manager-amd64:v1.4.5",
        "command": [
          "/usr/local/bin/kube-controller-manager",
          "--v=4",
          "--address=127.0.0.1",
          "--leader-elect",  
          "--master=127.0.0.1:8080",
          "--cluster-name=kubernetes",
          "--root-ca-file=/etc/kubernetes/pki/ca.pem",
          "--service-account-private-key-file=/etc/kubernetes/pki/apiserver-key.pem",
          "--cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem",
          "--cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem",
          "--insecure-experimental-approve-all-kubelet-csrs-for-group=system:kubelet-bootstrap"
        ],
略...
```
[root@10-10-102-92 manifests]# vi /etc/kubernetes/manifests/kube-scheduler.json 
```
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "kube-scheduler",
    "namespace": "kube-system",
    "creationTimestamp": null,
    "labels": {
      "component": "kube-scheduler",
      "tier": "control-plane"
    }
  },
  "spec": {
    "containers": [
      {
        "name": "kube-scheduler",
        "image": "gcr.io/google_containers/kube-scheduler-amd64:v1.4.5",
        "command": [
          "/usr/local/bin/kube-scheduler",
          "--v=4",
          "--address=127.0.0.1",
          "--leader-elect",
          "--master=127.0.0.1:8080"
        ],
略...
```

### 3.3.10 实现kube-apiserver组件的高可用
kube-apiserver作为核心入口，使用keepalived 实现高可用
keepalived模式为：主-从-从
包括：安装keepalived；配置keepalived.conf；编写track_script脚本

3个kube-apiserver实例通过keepalived+VIP方案实现主-备-备模式。
架构：
```
10.10.102.91 kube-apiserver主
10.10.102.92 kube-apiserver备
10.10.102.93 kube-apiserver备
```
安装keepalived
```
 yum install -y keepalived
```
主节点关键配置：
vi /etc/keepalived/keepalived.conf

```
! Configuration File for keepalived

global_defs {
  # notification_email {
   #  huanwei@harmonycloud.cn
   #}
   #notification_email_from huanwei@harmonycloud.cn
   #smtp_server 192.168.200.1
   #smtp_connect_timeout 30

   router_id LVS_K8S
   #vrrp_skip_check_adv_addr
   #vrrp_strict
   #vrrp_garp_interval 0
   #vrrp_gna_interval 0
}

vrrp_script check_k8s_master {
    script "curl -k http://127.0.0.1:8080/healthz"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    #mcast_src_ip 10.10.102.91
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script {
        check_k8s_master
    }
    virtual_ipaddress {
        10.10.102.21/24
    }
}

```

2个从节点参考主节点相应配置即可。

启动keepalived：
```
[root@10-10-102-91 keepalived]# systemctl restart keepalived
[root@10-10-102-91 keepalived]# systemctl enable keepalived
Created symlink from /etc/systemd/system/multi-user.target.wants/keepalived.service to /usr/lib/systemd/system/keepalived.service.
```
在所有master节点keepalived以后，查看node情况：
```
[root@10-10-102-91 ~]# kubectl get nodes
NAME                  STATUS    AGE
10-10-102-91.master   Ready     2d
10-10-102-92.master   Ready     2d
10-10-102-93.master   Ready     2d
10-10-103-96.node     Ready     1h
10-10-103-97.node     Ready     1h
```



### 其他的一些设置
```
vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf 
设置
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.0.0.10 --cluster-domain=cluster.local"
与kube-dns service的cluster ip相同
```

### etcd排错
etcd集群的其中一个节点宕机后，重启无影响etcd
集群的其中一个节点重装后，需要删除每一个节点的/var/lib/etcd/etcd0x ...等目录，然后systemctl restart etcd重启，然后etcdctl member list查看集群发现恢复。。否则启动etcd不成功。

### keepalived日志重定向
```
Keepalived默认所有的日志都是写入到/var/log/message下的，由于message的日志太多了，而Keepalived的日志又很难分离出来，所以本文提供了一个调整Keepalived日志输出路径的方法。
具体操作步骤如下：
一、修改 /etc/sysconfig/keepalived
把KEEPALIVED_OPTIONS="-D" 修改为KEEPALIVED_OPTIONS="-D -d -S 0"
#其中-S指定syslog的facility
二、重启服务
service keepalived restart
三、设置syslog，修改/etc/syslog.conf，添加内容如下
# keepalived -S 0
local0.* /var/log/keepalived.log
注意：local0是l是字符L的小写
```

## master节点集群的运维
### 故障及恢复
#### 组件进程故障及恢复
监控：
监控各master节点的kubelet等k8s进程是否存活。
故障处理：
如发现kubelet进程挂掉，则杀掉该节点keepalived
故障的恢复：
重启该节点的kubelet进程，重启该节点keepalived

#### 节点网络故障及恢复
监控：
监控各master节点间网络是否连通。
故障处理：
摘除连通失败的节点，杀掉该节点的keepalived，防止该节点可能priority较高，重新接入集群后造成keepalived split brain问题。
故障的恢复：
节点重新接入集群后，重启该节点keepalived。

#### 节点宕机故障及恢复
监控：
监控各master节点所在Node是否存活。
故障处理：
无需处理
故障的恢复：
重启该Node，并重启kubelet等进程，重启该节点keepalived



## 可用性测试
### 应用层测试
测试当master2节点故障时，对容器化应用进行弹性伸缩、滚动更新、自我恢复等测试。
说明：这里的logapi是一个简单的微服务，用以向客户端提供集群日志的api服务。
注意：部署应用的时候，良好的习惯是以yaml方式部署，这里可以指定nodeSelector为上面设定的label。
这里我是先用kubectl run命令部署，然后导出，yaml文件，改造一下，指定nodeSelector，然后删除deployment，再使用kubectl apply -f 命令重新部署。

```
#扩容前
[root@10-10-102-93 ~]# kubectl get pods -o wide
NAME                      READY     STATUS    RESTARTS   AGE       IP          NODE
logapi-1223020174-37mbb   1/1       Running   0          3m        10.34.0.4   10-10-103-96.node
logapi-1223020174-4h848   1/1       Running   0          8m        10.34.0.2   10-10-103-96.node
logapi-1223020174-c18wb   1/1       Running   0          3m        10.34.0.3   10-10-103-96.node
logapi-1223020174-ceqh6   1/1       Running   0          3m        10.38.0.2   10-10-103-97.node
logapi-1223020174-dlahm   1/1       Running   0          8m        10.34.0.1   10-10-103-96.node
logapi-1223020174-wf490   1/1       Running   0          3m        10.34.0.5   10-10-103-96.node

#扩容
[root@10-10-102-93 ~]# kubectl scale deployment logapi --replicas=12
deployment "logapi" scaled

#扩容后
[root@10-10-102-93 ~]# kubectl get pods -o wide
NAME                      READY     STATUS    RESTARTS   AGE       IP          NODE
logapi-1223020174-37mbb   1/1       Running   0          3m        10.34.0.4   10-10-103-96.node
logapi-1223020174-4aumj   1/1       Running   0          11s       10.38.0.5   10-10-103-97.node
logapi-1223020174-4h848   1/1       Running   0          9m        10.34.0.2   10-10-103-96.node
logapi-1223020174-c18wb   1/1       Running   0          3m        10.34.0.3   10-10-103-96.node
logapi-1223020174-ceqh6   1/1       Running   0          3m        10.38.0.2   10-10-103-97.node
logapi-1223020174-dlahm   1/1       Running   0          9m        10.34.0.1   10-10-103-96.node
logapi-1223020174-eeq7q   1/1       Running   0          11s       10.38.0.1   10-10-103-97.node
logapi-1223020174-jr3ct   1/1       Running   0          11s       10.38.0.3   10-10-103-97.node
logapi-1223020174-nxklc   1/1       Running   0          11s       10.38.0.6   10-10-103-97.node
logapi-1223020174-srwtj   1/1       Running   0          11s       10.34.0.6   10-10-103-96.node
logapi-1223020174-us0cs   1/1       Running   0          11s       10.38.0.4   10-10-103-97.node
logapi-1223020174-wf490   1/1       Running   0          3m        10.34.0.5   10-10-103-96.node

#缩容
[root@10-10-102-93 ~]# kubectl scale deployment logapi --replicas=3
deployment "logapi" scaled

#缩容后
[root@10-10-102-93 ~]# kubectl get pods -o wide
NAME                      READY     STATUS    RESTARTS   AGE       IP          NODE
logapi-1223020174-37mbb   1/1       Running   0          15m       10.34.0.4   10-10-103-96.node
logapi-1223020174-4h848   1/1       Running   0          20m       10.34.0.2   10-10-103-96.node
logapi-1223020174-dlahm   1/1       Running   0          20m       10.34.0.1   10-10-103-96.node

#滚动更新前，测试应用的服务接口
[root@10-10-102-91 ~]# curl http://10.10.103.96:31070/api/log/greeting
{"id":1,"content":"Hello, World! Request from 10.42.0.2 hostname logapi-1223020174-37mbb"}
说明运行正常。

#滚动更新


#滚动更新后，测试应用的服务接口


说明已经更新


#自我恢复
```

### 运维层测试
#### 测试进程故障及恢复


#### 测试网络故障及恢复

#### 测试机器故障及恢复