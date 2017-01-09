---
layout: wiki
title: 基于kubeadm部署kubernetes集群
categories: DevOps
description: 基于kubeadm部署kubernetes集群
keywords: Kubernetes, kubeadm
---

本文是基于kubeadm部署kubernetes集群。

## 1 环境准备

```
CentOS版本7.2，Docker版本1.12+
kubernetes版本为v1.4.5
10.10.102.38 10-10-102-38.master
10.10.102.39 10-10-102-39.node
10.10.102.40 10-10-102-40.node
```

## 2 关于二进制文件、镜像包

需要安装的二进制文件包括：

kubelet、kubeadm、kubectl、kubernetes-cni；

其他以镜像包的方式安装。

## 3 搭建集群

### 3.1 主机名处理

```
# 写入 hostname(node 节点后缀改成 .node)
echo "10-10-102-38.master" > /etc/hostname 

# 加入 hosts
echo "127.0.0.1   10-10-102-38.master" >> /etc/hosts

# 不重启情况下使内核生效
sysctl kernel.hostname=10-10-102-38.master

# 验证是否修改成功
hostname
10-10-102-38.master
```

### 3.2 安装docker

Docker版本1.12+

（略）

### 3.3 load镜像

```
images=(kube-proxy-amd64:v1.4.5 kube-discovery-amd64:1.0 kubedns-amd64:1.7 kube-scheduler-amd64:v1.4.5 kube-controller-manager-amd64:v1.4.5 kube-apiserver-amd64:v1.4.5 etcd-amd64:2.2.5 kube-dnsmasq-amd64:1.3 exechealthz-amd64:1.1 pause-amd64:3.0 kubernetes-dashboard-amd64:v1.4.1)
for imageName in ${images[@]} ; do
  docker pull huanwei/$imageName
  docker tag huanwei/$imageName gcr.io/google_containers/$imageName
  docker rmi huanwei/$imageName
done
```

### 3.4 安装二进制文件

```
cat <<EOF> /etc/yum.repos.d/k8s.repo
[kubelet]
name=kubelet
baseurl=http://files.rm-rf.ca/rpms/kubelet/
enabled=1
gpgcheck=0
EOF
```

```
# 清理缓存
yum makecache

# 安装
yum install -y socat kubelet kubeadm kubectl kubernetes-cni
```

### 3.5 初始化master

首先启动kubelet

```
# 启动kubelet
systemctl enable kubelet
systemctl start kubelet
```

然后初始化master，记录控制台最后一行输出的token。

```
# master初始化并指定apiserver监听地址
kubeadm init --api-advertise-addresses=10.10.102.38 --use-kubernetes-version v1.4.5
```

### 3.6 加入node

```
# node加入集群
kubeadm join --token=5e0880.010ed6fd5bcbe364 10.10.102.38
```

### 3.7 部署weave网络

```
# 先把weave的yaml文件搞下来
wget https://git.io/weave-kube -O weave-kube.yaml

# 然后pull镜像并tag一下
docker pull huanwei/weave-kube:1.8.2
docker tag huanwei/weave-kube:1.8.2 weaveworks/weave-kube:1.8.2
docker rmi huanwei/weave-kube:1.8.2

docker pull huanwei/weave-npc:1.8.2
docker tag huanwei/weave-npc:1.8.2 weaveworks/weave-npc:1.8.2
docker rmi huanwei/weave-npc:1.8.2

# 安装即可
kubectl create -f weave-kube.yaml
```

### 3.8 部署dashboard

dashboard 的命令也跟 weave 的一样，不过有个大坑，默认的 yaml 文件中对于 image 拉取策略的定义是无论何时都会去拉取镜像，导致即使你 load 进去也无卵用，所以还得先把 yaml 搞下来然后改一下镜像拉取策略，最后再 create -f 即可。

```
# 下载yaml文件
wget https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml -O kubernetes-dashboard.yaml
```

```
# 编辑 yaml 改一下 imagePullPolicy，把 Always 改成 IfNotPresent(本地没有再去拉取) 或者 Never(从不去拉取) ，再修改一下需要拉取的image版本（查看docker images，这里是v1.4.1）
image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.4.1
```

```
# 安装
kubectl create -f kubernetes-dashboard.yaml
```

## 4 遇到的一些坑

### 4.1 初始化master的问题

```
# 在初始化master时遇到了一个问题：
[root@k8s-master ~]# kubeadm init --api-advertise-addresses=10.10.102.38 --use-kubernetes-version v1.4.5
Running pre-flight checks
preflight check errors:
	/etc/kubernetes is not empty
```

解决办法：执行

```
[root@k8s-master ~]# rm -rf /etc/kubernetes/manifests/
```

重新kubeadm init即可。

### 4.2 安装Pod网络的问题

dns pod在pod网络方案安装之前是不工作的，pod网络方案安装好了之后就变为running了。

### 4.3 安装dashboard的问题

上面说到的那个大坑要注意，否则会一直pull远程镜像不成功。

### 4.4 网络的问题

镜像和安装包时遇到的网络问题已经绕过或者解决。

### 4.5 dashboard地址

```
http://10.10.102.38:32608/#/node?namespace=default
```