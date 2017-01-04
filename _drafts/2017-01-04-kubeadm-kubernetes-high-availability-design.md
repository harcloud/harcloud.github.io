---
layout: post
title: 基于kubeadm方式部署的kubernetes高可用架构设计
categories: DevOps
description: 基于kubeadm方式部署的kubernetes高可用架构设计
keywords: Kubernetes, kubeadm
---

本文是基于kubeadm方式部署的kubernetes高可用架构设计。阅读对象为熟悉kubernetes和kubeadm基本概念的八零后以及九零后。

### 说明
本方案基于 kubernetes v1.4.5 版本，理论上v1.4版本以上都应该适用。

### 阅读对象
熟悉kubernetes和kubeadm基本概念的八零后以及九零后。

## k8s集群架构
```
master1: 10.10.102.91
master2: 10.10.102.92
master3: 10.10.102.93
node1:   10.10.103.96
node2:   10.10.103.97
vip:     10.10.102.21
```

## 组件优化

## kube-dns组件
导出原先kube-dns组件的yaml文件
修改yaml文件：
将Deployment改造成DaemonSet
删除原先kube-dns的Service和Deployment
创建改造后的kube-dns

## kube-discovery组件
导出原先kube-discovery组件的yaml文件
修改yaml文件：
将Deployment改造成DaemonSet
删除原先kube-discovery的Deployment
创建改造后的kube-discovery

重启所有master节点的kubelet
```
systemctl daemon-reload
systemctl restart kubelet
```

## kube-apiserver组件
在所有节点安装keepalived
修改keepalived.conf配置文件：
设置每个节点的权重
设置vip
编写track_script


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

## 节点添加label
为master节点添加label: role=master
为node节点添加label: role=node
```
[root@10-10-102-91 ~]# kubectl label node 10-10-102-91.node kubeadm.alpha.kubernetes.io/role=master
[root@10-10-102-91 ~]# kubectl label node 10-10-102-92.node kubeadm.alpha.kubernetes.io/role=master
[root@10-10-102-91 ~]# kubectl label node 10-10-102-93.node kubeadm.alpha.kubernetes.io/role=master
[root@10-10-102-91 ~]# kubectl label node 10-10-103-96.node kubeadm.alpha.kubernetes.io/role=node
[root@10-10-102-91 ~]# kubectl label node 10-10-103-97.node kubeadm.alpha.kubernetes.io/role=node
```
这样可以指定经过改造的组件kube-dns、kube-discovery部署在master节点上，logapi应用则部署在node节点上。

scale out应用：
```
[root@10-10-102-91 ~]# kubectl scale deploy/logapi --replicas=10
```
看一下应用部署情况：
```
[root@10-10-102-91 ~]# kubectl get pods -o wide
NAME                      READY     STATUS    RESTARTS   AGE       IP            NODE
logapi-1223020174-0ds81   1/1       Running   0          1m        10.42.0.3     10-10-103-96.node
logapi-1223020174-1b7ot   1/1       Running   0          17s       10.42.0.4     10-10-103-96.node
logapi-1223020174-6qxhx   1/1       Running   0          43s       10.43.128.5   10-10-103-97.node
logapi-1223020174-71x0z   1/1       Running   0          17s       10.43.128.7   10-10-103-97.node
logapi-1223020174-acvnq   1/1       Running   0          17s       10.43.128.6   10-10-103-97.node
logapi-1223020174-d2qp0   1/1       Running   0          17s       10.42.0.6     10-10-103-96.node
logapi-1223020174-dqrrx   1/1       Running   0          17s       10.43.128.8   10-10-103-97.node
logapi-1223020174-mqj8o   1/1       Running   0          17s       10.42.0.5     10-10-103-96.node
logapi-1223020174-nx73o   1/1       Running   0          17s       10.42.0.2     10-10-103-96.node
logapi-1223020174-ogqw0   1/1       Running   0          1m        10.43.128.4   10-10-103-97.node

```
看到logapi全部只部署在node节点上。

这里如果不指定logapi，可能会将Pod部署到master节点上去。

测试kube-apiserver的keepalived情况：

断开其中一台master节点，如master1节点，则根据权重设置，vip将漂移到master2节点，如下所示：
```
[root@10-10-102-92 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:50:56:bc:6a:fc brd ff:ff:ff:ff:ff:ff
    inet 10.10.102.92/24 brd 10.10.102.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.10.102.21/24 scope global secondary eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:febc:6afc/64 scope link
       valid_lft forever preferred_lft forever
...（略）
```

此时查看node情况如下：
```
[root@10-10-102-91 ~]# kubectl get nodes
NAME                  STATUS     AGE
10-10-102-91.master   NotReady   2d
10-10-102-92.master   NotReady   2d
10-10-102-93.master   NotReady   2d
10-10-103-96.node     NotReady   2h
10-10-103-97.node     NotReady   2h

```
测试一下应用的访问是否受影响：

```
[root@10-10-102-91 ~]# curl http://10.10.103.96:31070/api/log/greeting
{"id":1,"content":"Hello, World! Request from 10.42.0.2 hostname logapi-1223020174-nx73o"}
```

测试一下应用的scale是否收影响：
scale in:
```
[root@10-10-102-91 ~]# kubectl scale deploy/logapi --replicas=5
deployment "logapi" scaled

[root@10-10-102-91 ~]# kubectl get pods -o wide
NAME                      READY     STATUS        RESTARTS   AGE       IP            NODE
logapi-1223020174-0ds81   1/1       Running       0          1h        10.42.0.3     10-10-103-96.node
logapi-1223020174-1b7ot   1/1       Terminating   0          1h        10.42.0.4     10-10-103-96.node
logapi-1223020174-6qxhx   1/1       Running       0          1h        10.43.128.5   10-10-103-97.node
logapi-1223020174-71x0z   1/1       Terminating   0          1h        10.43.128.7   10-10-103-97.node
logapi-1223020174-acvnq   1/1       Terminating   0          1h        10.43.128.6   10-10-103-97.node
logapi-1223020174-d2qp0   1/1       Terminating   0          1h        10.42.0.6     10-10-103-96.node
logapi-1223020174-dqrrx   1/1       Running       0          1h        10.43.128.8   10-10-103-97.node
logapi-1223020174-mqj8o   1/1       Running       0          1h        10.42.0.5     10-10-103-96.node
logapi-1223020174-nx73o   1/1       Terminating   0          1h        10.42.0.2     10-10-103-96.node
logapi-1223020174-ogqw0   1/1       Running       0          1h        10.43.128.4   10-10-103-97.node

```
继续scale out:
```
[root@10-10-102-91 ~]# kubectl get pods -o wide
NAME                      READY     STATUS        RESTARTS   AGE       IP            NODE
logapi-1223020174-0ds81   1/1       Running       0          1h        10.42.0.3     10-10-103-96.node
logapi-1223020174-1b7ot   1/1       Terminating   0          1h        10.42.0.4     10-10-103-96.node
logapi-1223020174-5ibja   0/1       Pending       0          17s       <none>
logapi-1223020174-6qxhx   1/1       Running       0          1h        10.43.128.5   10-10-103-97.node
logapi-1223020174-71x0z   1/1       Terminating   0          1h        10.43.128.7   10-10-103-97.node
logapi-1223020174-76mn5   0/1       Pending       0          17s       <none>
logapi-1223020174-acvnq   1/1       Terminating   0          1h        10.43.128.6   10-10-103-97.node
logapi-1223020174-d2qp0   1/1       Terminating   0          1h        10.42.0.6     10-10-103-96.node
logapi-1223020174-dqrrx   1/1       Running       0          1h        10.43.128.8   10-10-103-97.node
logapi-1223020174-m53hz   0/1       Pending       0          17s       <none>
logapi-1223020174-mqj8o   1/1       Running       0          1h        10.42.0.5     10-10-103-96.node
logapi-1223020174-new3s   0/1       Pending       0          17s       <none>
logapi-1223020174-nx73o   1/1       Terminating   0          1h        10.42.0.2     10-10-103-96.node
logapi-1223020174-ogqw0   1/1       Running       0          1h        10.43.128.4   10-10-103-97.node
logapi-1223020174-xa4ve   0/1       Pending       0          17s       <none>
```
结果看到一堆状态是Terminating和Pending的pod，
describe 一下Pending状态的pod，看到提示调度失败，原因是"no nodes available to schedule pods"。

分析原因，估计是master2节点发现不了node节点。










