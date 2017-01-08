---
layout: post
title: ����kubeadm����kubernetes��Ⱥ
categories: DevOps
description: ����kubeadm����kubernetes��Ⱥ
keywords: Kubernetes, kubeadm
---

�����ǻ���kubeadm����kubernetes��Ⱥ��

## 1 ����׼��

```
CentOS�汾7.2��Docker�汾1.12+
kubernetes�汾Ϊv1.4.5
10.10.102.38 10-10-102-38.master
10.10.102.39 10-10-102-39.node
10.10.102.40 10-10-102-40.node
```

## 2 ���ڶ������ļ��������

��Ҫ��װ�Ķ������ļ�������

kubelet 

kubeadm 

kubectl 

kubernetes-cni

�����Ծ�����ķ�ʽ��װ��

## 3 ���Ⱥ

### 3.1 ����������

```
# д�� hostname(node �ڵ��׺�ĳ� .node)
echo "10-10-102-38.master" > /etc/hostname 
# ���� hosts
echo "127.0.0.1   10-10-102-38.master" >> /etc/hosts
# �����������ʹ�ں���Ч
sysctl kernel.hostname=10-10-102-38.master
# ��֤�Ƿ��޸ĳɹ�
hostname
10-10-102-38.master
```

### 3.2 ��װdocker

Docker�汾1.12+

���ԣ�

### 3.3 load����

```
images=(kube-proxy-amd64:v1.4.5 kube-discovery-amd64:1.0 kubedns-amd64:1.7 kube-scheduler-amd64:v1.4.5 kube-controller-manager-amd64:v1.4.5 kube-apiserver-amd64:v1.4.5 etcd-amd64:2.2.5 kube-dnsmasq-amd64:1.3 exechealthz-amd64:1.1 pause-amd64:3.0 kubernetes-dashboard-amd64:v1.4.1)
for imageName in ${images[@]} ; do
  docker pull huanwei/$imageName
  docker tag huanwei/$imageName gcr.io/google_containers/$imageName
  docker rmi huanwei/$imageName
done
```

### 3.4 ��װ�������ļ�

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
# ������
yum makecache
# ��װ
yum install -y socat kubelet kubeadm kubectl kubernetes-cni
```

### 3.5 ��ʼ��master

��������kubelet

```
# ����kubelet
systemctl enable kubelet
systemctl start kubelet
```

Ȼ���ʼ��master����¼����̨���һ�������token��

```
# master��ʼ����ָ��apiserver������ַ
kubeadm init --api-advertise-addresses=10.10.102.38 --use-kubernetes-version v1.4.5
```

### 3.6 ����node

```
# node���뼯Ⱥ
kubeadm join --token=5e0880.010ed6fd5bcbe364 10.10.102.38
```

### 3.7 ����weave����

```
# �Ȱ�weave��yaml�ļ�������
wget https://git.io/weave-kube -O weave-kube.yaml
# Ȼ��pull����tagһ��
docker pull huanwei/weave-kube:1.8.2
docker tag huanwei/weave-kube:1.8.2 weaveworks/weave-kube:1.8.2
docker rmi huanwei/weave-kube:1.8.2

docker pull huanwei/weave-npc:1.8.2
docker tag huanwei/weave-npc:1.8.2 weaveworks/weave-npc:1.8.2
docker rmi huanwei/weave-npc:1.8.2

# ��װ����
kubectl create -f weave-kube.yaml
```

### 3.8 ����dashboard

dashboard ������Ҳ�� weave ��һ���������и���ӣ�Ĭ�ϵ� yaml �ļ��ж��� image ��ȡ���ԵĶ��������ۺ�ʱ����ȥ��ȡ���񣬵��¼�ʹ�� load ��ȥҲ�����ã����Ի����Ȱ� yaml ������Ȼ���һ�¾�����ȡ���ԣ������ create -f ���ɡ�

```
# ����yaml�ļ�
wget https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml -O kubernetes-dashboard.yaml
```

```
# �༭ yaml ��һ�� imagePullPolicy���� Always �ĳ� IfNotPresent(����û����ȥ��ȡ) ���� Never(�Ӳ�ȥ��ȡ) �����޸�һ����Ҫ��ȡ��image�汾���鿴docker images��������v1.4.1��
image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.4.1
```

```
# ��װ
kubectl create -f kubernetes-dashboard.yaml
```

## 4 ������һЩ��

### 4.1 ��ʼ��master������

```
�ڳ�ʼ��masterʱ������һ�����⣺
[root@k8s-master ~]# kubeadm init --api-advertise-addresses=10.10.102.38 --use-kubernetes-version v1.4.5
Running pre-flight checks
preflight check errors:
	/etc/kubernetes is not empty
```

����취��ִ��

```
[root@k8s-master ~]# rm -rf /etc/kubernetes/manifests/
```

����kubeadm init���ɡ�

### 4.2 ��װPod���������

dns pod��pod���緽����װ֮ǰ�ǲ������ģ�pod���緽����װ����֮��ͱ�Ϊrunning�ˡ�

### 4.3 ��װdashboard������

����˵�����Ǹ����Ҫע�⣬�����һֱpullԶ�̾��񲻳ɹ���

### 4.4 ���������

����Ͱ�װ��ʱ���������������Ѿ��ƹ����߽����

### 4.5 dashboard��ַ

```
http://10.10.102.38:32608/#/node?namespace=default
```