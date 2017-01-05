---
layout: post
title: kubernetes集群高可用架构设计及实现
categories: DevOps
description: kubernetes集群高可用架构设计及实现
keywords: Kubernetes, kubeadm
---

本文是基于kubeadm方式部署的kubernetes高可用架构设计。阅读对象为熟悉kubernetes和kubeadm基本概念的同学。

### 说明
本方案基于 kubernetes v1.4.5 版本，理论上v1.4版本以上都应该适用。

### 阅读对象
阅读对象为熟悉kubernetes和kubeadm基本概念的同学。

## kubernetes集群架构
```
master1: 10.10.102.91
master2: 10.10.102.92
master3: 10.10.102.93
node1:   10.10.103.96
node2:   10.10.103.97
vip:     10.10.102.21
```


##创建etcd集群
比较简单，略。

不过这里有个坑，尽量不要在master集群所在的几个节点创建etcd。
因为kubeadm init的时候会去检查当前节点是否含有/etc/kubernetes目录以及/var/lib/etcd目录，如有这些目录会提示删除，否则init不成功。
如果一不小心把etcd集群建在了几个master节点，那么解决办法是修改etcd所在目录。
kubernetes的所有信息都存在这里创建的etcd。

##部署vip
首先在master1节点部署一个vip
略。

##创建kubernetes集群
kubeadm init命令初始化master1，可以参考我写的文档。
注意将etcd指向上面创建的etcd集群，将master ip指向vip。
初始化后记录token。
然后
scp -r /etc/kubernetes/* root@10.10.102.92:/etc/kubernetes/
scp -r /etc/kubernetes/* root@10.10.102.93:/etc/kubernetes/
即初始化好了其他两个master节点。

##加入node节点
略。

##部署网络方案
略，因前两天玩单个master节点的kubernetes集群时已经制作了好了weave网络方案的镜像，现在正好拿来实验。
生产环境可以按需选择合适的网络方案。目前比较成熟的网络方案基本上都是以DaemonSet插件的形式部署。


##改造kube-apiserver组件
kube-apiserver是master节点的核心组件，对外提供REST API服务接口。因此首先需要对该组件实现高可用。
这里以keepalived+vip解决。
比较简单，略。

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


##添加节点label
为master节点及node节点分别添加以下label
```
[root@10-10-102-91 ~]# kubectl label node 10-10-102-91.node kubeadm.alpha.kubernetes.io/role=master
[root@10-10-102-91 ~]# kubectl label node 10-10-102-92.node kubeadm.alpha.kubernetes.io/role=master
[root@10-10-102-91 ~]# kubectl label node 10-10-102-93.node kubeadm.alpha.kubernetes.io/role=master
[root@10-10-102-91 ~]# kubectl label node 10-10-103-96.node kubeadm.alpha.kubernetes.io/role=node
[root@10-10-102-91 ~]# kubectl label node 10-10-103-97.node kubeadm.alpha.kubernetes.io/role=node
```
目的：这样可以指定组件kube-dns、kube-discovery仅部署在master节点，logapi应用则部署在node节点上。
由于这里我们有多个master节点，集群会将多个master节点也看作是node节点，因此如果不给node节点约定一个label的话，后面部署应用时，必然会出现应用的部分Pod会调度到master节点而部署不成功或者不妥的情况。


##改造kube-dns组件
kube-dns组件主要包括两部分：kube-dns和dnsmasq。
我在安装调试时，查看了其组件日志，得知kube-dns主要是从kubernetes master中不停地监听变动的service和endpoint信息，并建立从service ip到service name的映射关系，对于没有service的则建立pod ip和相应域名的映射。
kube-dns将这些信息存放在本地缓存，并监听127.0.0.1:10053提供服务。
dnsmasq则是一个简单的域名服务、缓存和转发工具。这里主要利用它的转发功能将kube-dns的dns服务转接到外部，参数-server=127.0.0.1:10053。

方案1：Deployment方式
步骤：
1)导出kube-dns组件的原yaml文件；
```

```

2)改变其nodeSelector
```
nodeSelector:
        #beta.kubernetes.io/arch: amd64
        kubeadm.alpha.kubernetes.io/role: master
```
3)删除原先kube-dns的Deployment
```

```	
4)部署改造后的kube-dns组件
```

```
5)对kube-dns组件扩容
这里按master节点个数，对其进行扩容
```

```
改造后可以看到，新的kube-dns组件部署在各master节点
```

```

方案2：DaemonSet方式
步骤：
1)跟上面一样，导出kube-dns组件的原yaml文件；
```

```

2)改造为DaemonSet
主要是把Deployment改成DaemonSet、改变其nodeSelector、删除DaemonSet不需要的内容如：Strategy、Replica、Status等等。
实际改造的时候，有不少坑，可按照下面重新部署时给出的提示作相应调整即可。

3)删除原先kube-dns的Deployment
```

```	
4)部署改造后的kube-dns组件
实际上面改造DaemonSet的时候，可以参考这里部署时命令行返回的提示，根据提示内容作相应的调整即可。

改造后可以看到，新的kube-dns组件以DaemonSet的形式部署在各master节点。
```

```


##改造kube-discovery组件
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
对其扩容，指定replicas=3，即可看到其均匀分布在各master节点：
```

```

##kube-controller-manager组件和kube-scheduler组件
查看两者的yaml文件，发现两者已实现自动leader-elect，因此无需对其进行任何修改。
当我们对kube-discovery组件扩容后，这两个组件也自动扩容。



##其他的一些设置
```
vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf 
设置
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.0.0.10 --cluster-domain=cluster.local"
与kube-dns service的cluster ip相同
```



##master节点集群的运维
###故障及恢复
####进程故障及恢复
监控：
监控各master节点的kubelet等k8s进程是否存活。
故障处理：
如发现kubelet进程挂掉，则杀掉该节点keepalived
故障的恢复：
重启该节点的kubelet进程，重启该节点keepalived

####网络故障及恢复


####Node故障及恢复
说明：这里的Node指master节点所在虚拟机或物理机。
监控：
监控各master节点所在Node是否存活。
故障处理：
无需处理
故障的恢复：
重启该Node，并重启kubelet等进程，重启该节点keepalived





##可用性测试
###应用层测试
测试当master2节点故障时，对容器化应用进行弹性伸缩、滚动更新、自我恢复等测试。
说明：这里的logapi是一个简单的微服务，用以向客户端提供集群日志的api服务。
注意：部署应用的时候，良好的习惯是以yaml方式部署，这里可以指定nodeSelector为上面设定的label。
这里我是先用kubectl run命令部署，然后导出，yaml文件，改造一下，指定nodeSelector，然后删除deployment，再使用kubectl apply -f 命令重新部署。

```
#扩容
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

###运维层测试
####测试进程故障及恢复


####测试网络故障及恢复

####测试机器故障及恢复





