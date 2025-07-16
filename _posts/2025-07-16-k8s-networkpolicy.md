---
layout: post
title: K8s NetworkPolicy
categories: [linux]
tags: []
---

> 

#### 1. 介绍
网络策略（Network Policy ）是 Kubernetes 的一种资源。Network Policy 通过 Label 选择 Pod，并指定其他 Pod 或外界如何与这些 Pod 通信。
Pod的网络流量包含流入（Ingress）和流出（Egress）两种方向。默认情况下，所有 Pod 是非隔离的，即任何来源的网络流量都能够访问 Pod，没有任何限制。当为 Pod定义了 Network Policy，只有 Policy 允许的流量才能访问 Pod。
Kubernetes的网络策略功能也是由第三方的网络插件实现的，因此，只有支持网络策略功能的网络插件才能进行配置网络策略，比如Calico、Canal、kube-router等等。


#### 2. Ingress
允许Label包含name=test的pod访问Label包含name=myapp的pod

```yaml
piVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-myapp-ingress
  namespace: prod
spec:
  podSelector:
    matchLabels:
      name: myapp
  policyTypes ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: test
    ports:
    - protocol: TCP
      port: 80
```

#### 3. Egress
允许Label包含name=myqpp的pod访问Label包含name=test的pod的8080和53端口

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-access-networkpolicy
  namespace: prod
spec:
  policyTypes:
  - Egress
  podSelector:
    matchLabels:
      name: myqpp
  egress:
  - to:
    - podSelector: 
        matchLabels:
          name: test
    ports:
    - protocol: TCP
      port: 8080
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```

#### 3. 一个例子
禁止myapp的所有出站访问

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: prod
spec:
  podSelector: 
    matchLabels:
      name: myapp
  policyTypes:
  - Egress
```

#### 4. 参考

* [网络策略](https://kubernetes.io/zh-cn/docs/concepts/services-networking/network-policies/)