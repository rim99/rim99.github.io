---
layout: post
title: "K3d的使用经验"
date: 2023-01-18 17:10:13 +0800
categories: 原创
---

K3d是对Rancher公司开发的K3s的一个轻度封装，允许在Docker内部运行一个小型Kubernetes集群。这一特性对于开发者来说，能够非常方便地实现本地调试K8s集群。

## 竞品对比

### MiniKube

MiniKube是出现在官方K8s文档里的本地集群方案。但是很多功能都只考虑国际用户，对中国区用户不是很友好，还得费劲去gcr拉取K8s基础设施的容器镜像了。

而Rancher在中国有开发团队，所以K3s在中国的本地化做得更好。镜像都是维护在Docker Hub上的。对应mirror站点很好找。

### MicroK8s

MicroK8s是Canonical公司开发的K8s集群的边缘场景部署方案。本地用着也很不错。很多常见的设施，例如私有容器镜像，Cilium，helm等等都是集成好的addon插件，可以一条命令就开启，比较方便。不过，MicroK8s内部使用了同公司的Multipass虚拟机产品。这一套组合在MacOS上的运行并不稳定：
- 有时正常使用的集群可能第二天突然无法登陆
- 有时会发现很久没有启动的虚拟机无法启动
- 有时K8s集群功能还算正常，但是虚拟机node无法ssh登陆

想比之下，K3d是运行在Docker内部的集群。我的集群开启好几个月了，有时用一下，有时好几周都不碰它，一直都还正常，没有出什么妖蛾子。

此外，MicroK8s的K8s基础设施镜像也需要从gcr拉取。

> *在openSUSE上说些K3s赞美之词，em，我大概是SUSE的真粉丝。*

## 安装

我的K3d集群使用[AutoK3s](https://docs.rancher.cn/docs/k3s/autok3s/_index/)启动起来的。只需要在最开始的时候运行一下AutoK3s的Web GUI配置一下就好了。把新集群的kube config配置在本地，例如`~/.kube/config`。后面一直本地用kubectl命令行工具访问就行了。

## 使用经验小结

这里总结几个典型场景的解决方案，方便以后查阅。

### 配置私有Registry

开发者往往需要本地构建镜像，并在K3d集群里进行测试。但是本地构建的镜像存储在Docker运行时环境中。K3d有自己独立的运行时，无法直接接触到Docker环境里的镜像。这就需要为K3d集群配置镜像Registry。

如果公司能够提供公有Registry供开发测试用，那自然更好。但很多时候，开发者可能没有那么幸运。好在，Docker提供了一个简易的Registry可以实现容器部署，所以我们可以很方便的在本地建立一个私有Registry仅供K3d集群使用。

#### 确定K3d的网络配置

需要注意的是，K3d使用docker-compose部署，建立了私有的network。我们可以利用`docker inspect {container-id}`来查看其所属的network：`[0].NetworkSettings.Networks`的子JSON的Key名字。例如，下面例子中，我们要找的network名字就是`net-xxx`。

```json
[
  {
    ...
    "NetworkSettings": {
        "Networks": {
            "net-xxx": {
                ...
            } 
        }
    }  
  }
]
```

#### 启动Registry

[Docker官方的Registry](https://hub.docker.com/_/registry)可以一键启动。

```
docker run -d --name registry -p 5000:5000 --network net-xxx registry:latest
```

当registry容器启动后，我们在K3d的容器节点中应当能够访问到hostname: `registry`。例如，`ping`或者`nslookup`等等。

#### 配置Containerd

我们需要对K3s做一些配置以便从刚刚启动好的Registry中拉取镜像。

*目前，我还没有找到很好的方法实现配置持久化，暂时以hack的方式来实现*

在K3d的agent容器中的`/etc/rancher/k3s`目录下，放入这个文件

```
/etc/rancher/k3s # cat registries.yaml

mirrors:
  registry:5000:
    endpoint:
    - "http://registry:5000"
```

要想让新配置生效需要**重启K3s的进程**。

#### 使用

在K3s完成重启后，就可以通过`registry:5000/a/b`来拉取私有Registry上的`a/b`镜像了。

### 配置外部服务

有时需要部署一些服务在K3d集群外。如果服务是部署在开发机之外当然方便。但如果只能在本地启动，那就需要动另一番脑筋了。

Pod里面对域名的查找是通过集群的CoreDNS服务来查询的。所以，我们需要对其内部的DNS record做一些改动。

K8s的CoreDNS的域名记录可以通过修改其对应ConfigMap资源来实现。

```
kubectl edit cm coredns -m kube-system 
```

修改里面的`NodeHosts`对象就可以了

```
NodeHosts: |
    172.24.0.1 example-svc
```

*在这里我们需要IP地址`172.24.0.1`在K3s agent容器中能够联通*。

然后重启CoreDNS服务

```
kubectl rollout restart -n kube-system deployment/coredns
```

当CoreDNS完成重启后，我们就可以在K8s的Pod中访问新增的服务了。例如，

```
curl http://example-svc:8080
```
