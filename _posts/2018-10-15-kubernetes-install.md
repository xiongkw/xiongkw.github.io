---
layout: post
title: Kubernetes安装
categories: [编程, docker, linux]
tags: [docker, k8s]
---


> 早期的`kubernetes`的安装是非常复杂的，现在有了`kubeadm`这样的一键部署工具

#### 1. 安装docker

使用阿里`yum`源
```
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
```

安装
```
$ yum install docker-ce -y

$ docker --version
Docker version 18.06.1-ce, build e68fc7a

$ systemctl start docker
```

设置开机启动

```
$ systemctl enable docker
```

#### 2. 安装kubeadm

使用阿里`yum`源
```
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

安装
```
$ yum install kubeadm kubelet kubectl -y
```

设置`kubelet`开机启动
```
$ systemctl enable kubelet

$ systemctl start kubelet
```

#### 3. 在master上安装k8s

##### 3.1 环境准备

```
# 修改hostname
hostnamectl set-hostname master
echo 127.0.0.1  master >> /etc/hosts

# 关闭firewall
systemctl disable firewalld
systemctl stop firewalld

# 关闭selinux(vi /etc/sysconfig/selinux)
SELINUX=disabled

setenforce 0

# 关闭swap
swapoff -a
```

##### 3.2 修改kubeadm配置

获取`kubeadm`默认配置并修改
```
$ kubeadm config print-default > kubeadm.yml

# 修改token ttl为0s，即永不过期
# 因为是多网卡主机，这里修改了apiserver的地址
$ vi kubeadm.yml
```

##### 3.3 拉取依赖镜像

由于网络原因，无法拉取`k8s.gcr.io`的镜像，这里通过在[Docker Hub](https://hub.docker.com)中拉取并重新打`tag`的方式实现

```
$ docker pull mirrorgooglecontainers/kube-apiserver:v1.12.1
$ docker image tag mirrorgooglecontainers/kube-apiserver:v1.12.1 k8s.gcr.io/kube-apiserver:v1.12.1

$ docker pull mirrorgooglecontainers/kube-controller-manager:v1.12.1
$ docker image tag mirrorgooglecontainers/kube-controller-manager:v1.12.1 k8s.gcr.io/kube-controller-manager:v1.12.1

$ docker pull mirrorgooglecontainers/kube-proxy:v1.12.1
$ docker image tag mirrorgooglecontainers/kube-proxy:v1.12.1 k8s.gcr.io/kube-proxy:v1.12.1

$ docker pull mirrorgooglecontainers/kube-scheduler:v1.12.1
$ docker image tag mirrorgooglecontainers/kube-scheduler:v1.12.1 k8s.gcr.io/kube-scheduler:v1.12.1

$ docker pull mirrorgooglecontainers/etcd:3.2.24
$ docker image tag mirrorgooglecontainers/etcd:3.2.24 k8s.gcr.io/etcd:3.2.24

$ docker pull mirrorgooglecontainers/pause:3.1
$ docker image tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1

$ docker pull coredns/coredns:1.2.2
$ docker image tag coredns/coredns:1.2.2 k8s.gcr.io/coredns:1.2.2
```

> 尝试过修改`kubeadm.yml`中的`imageRepository`，不过貌似后面`pod`起不来，也没去深究

##### 3.4 初始化Master

```
# 把日志输出到kubeadm.out文件
$ kubeadm init --config kubeadm.yml > kubeadm.out &
```

> 查看`kubeadm.out`文件`Your Kubernetes master has initialized successfully!`，后面还有`join token`，表示初始化成功

##### 3.5 安装网络插件

这里用`weave-net`
```
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=1.12.1"
```

##### 3.6 验证

获取节点列表
```
$ kubectl get nodes
NAME         STATUS   ROLES    AGE    VERSION
master      Ready    master   3m   v1.12.1
```

获取`pod`列表
```
$ kubectl get pods -n kube-system
NAME                                 READY   STATUS    RESTARTS   AGE
coredns-576cbf47c7-6nhr4             1/1     Running   0          4m
coredns-576cbf47c7-cljf6             1/1     Running   0          4m
etcd-k8s-master                      1/1     Running   0          4m
kube-apiserver-k8s-master            1/1     Running   0          4m
kube-controller-manager-k8s-master   1/1     Running   0          4m
kube-proxy-s5jj8                     1/1     Running   0          4m
kube-scheduler-k8s-master            1/1     Running   0          4m
weave-net-nf7dq                      2/2     Running   0          1m
```

> 可以看到`k8s`的核心套件`apiserver、controller-manager、scheduler、etcd、proxy`及其插件`coredns、weave-net`都是以`pod`方式运行的，而`kubelet`组件则是运行在宿主机上的

#### 4. 发布应用

##### 4.1 污点设置

`master`节点默认是不能发布`pod`的，查看污点
```
$ kubectl describe node master | grep Taints
Taints:             node-role.kubernetes.io/master:NoSchedule
```

删除该污点
```
$ kubectl taint nodes --all node-role.kubernetes.io/master-

$ kubectl describe node master | grep Taints
Taints:             <none>
```

> 另一种方法是配置`pod`容忍该污点，例如：

```
tolerations:
  - effect: NoSchedule
    operator: Exists
```

##### 4.2 编写Deployment

```
$ vi nginx.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.5
        ports:
        - containerPort: 80
```

##### 4.2 发布
```
$ kubectl apply -f nginx.yml

$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-7d779b595-8jwqp   1/1     Running   0          6s
```

##### 4.3 删除

```
kubectl delete -f nginx.yml
```

#### 5. 参考

* [Docker Hub](https://hub.docker.com/)

* [Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)