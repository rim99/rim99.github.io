---
layout: post
title: "一种去中心化的Service Mesh方案"
date: 2025-06-14 17:01:52 +0800
categories: 原创
---

## Service Mesh

Service Mesh是一种基于代理软件的，负责服务间通信的分布式软件系统，可以提供流量控制，服务间鉴权，可观测性数据采集等功能。目前主流方案，会采用Envoy，Traefik等代理软件。

## 服务注册与发现

Envoy，Traefik这样的代理软件不具备服务注册/发现的能力。这一部分功能通常依赖于第三方软件系统。例如，早期Spring Cloud使用Eureka做服务注册中心，后来用Consul。甚至Istio，Linkerd干脆基于Kubernetes实现。

不管是Eureka，Consul，还是Kubernetes，这些系统都是需要单独部署的，增加了部署复杂度和成本。

此外，Consul或者etcd（Kubernetes）这样的强一致系统，会可能因为网络分区问题而影响到功能的可用性。

### Serf

Serf是Hashicorp公司的一个去中心化的服务发现软件。Serf本身依赖于memberlist库，采用gossip协议与集群其他成员通信，是一个AP属性的最终一致性系统。

Serf一个比较巧妙的设计是，对于集群成员变更等事件的本地处理，是基于Shell环境的。我们可以自定义脚本逻辑，甚至实现更复杂的可执行程序，來处理事件。基于Shell的好处就是中立性：
- 语言中立：具体的服务可以用任何编程语言实现。对比Spring Cloud这样的侵入式SDK，Serf更加灵活。
- 工具中立：集群成员发生变化后，假如我们想改DNS解析行为，可以去修改DNS服务器记录，也可以选择更新本地`/etc/hosts`文件。当然在这里，我们会更改反向代理软件的配置。

## 组合方案

### 服务注册

Serf的集群是去中心化的。各个节点发布自己的标签。利用这个方式，我们可以发布各自节点上的服务信息。例如，使用标签`services=app1,app2`，让集群其他成员知晓该节点上有服务app1和app2可用。

```
serf tags -set services="app1,app2" -delete services # 删除已经存在的标签，并增加新的标签内容，实现一次更新
```

### 服务发现

每一个serf agent加入集群，都是先连接种子节点，发布自己的信息，并逐渐获取集群全部节点的信息。

从整体来看，serf集群可以逐渐的将成员的变更事件传递到达每一个节点。而让集群全部达成一致的延迟取决于集群自身的规模。

```shell
$> serf members
vm1  192.168.122.128:7946  alive  zone=a,services=app1
vm2  192.168.122.116:7946  alive  zone=b,services=app2
...
```

### Serf集群成员事件消费

Serf成员的tag里的service最终都需要被其他的服务来调用。

而在Service mesh里，我们用反向代理作为本地服务网关：
- 处理本地服务发起的远程请求
- 处理远程服务发起的远程请求

Serf使用shell脚本作为集群成员事件的消费者。在这里，我们可以根据所知晓的服务在各个虚拟机上的分布情况，动态的生成反向代理软件的配置。

### Demo: Serf + Traefik

Repo: https://github.com/rim99/serf-traefik-svc-mesh

在这个Demo中，每一个虚拟机上部署一套Traefik系统。各自监听两个端口：
- 18080：用于接收来自远程服务的请求，转发给本地服务
- 28080：用于接收来自本地服务的请求，转发给其他虚拟机的服务网关

我们可以根据serf的member列表，结合简单的模板渲染工具，生成Traefik的配置文件。Traefik能够实时监听配置文件夹里的变化，并自动重载配置。

![示意图](/assets/images/serf-traefik-service-mesh.png)

## 总结

以上，我们使用Serf和Traefik的组合实现了一套简单的去中心化的Service Mesh方案PoC。

特点是：
- 高可用
- 语言中立
- 低运维复杂度

相比Kubernetes，这套系统更为简单（或者简陋）。有一些功能缺失，需要结合实际情况另外实现：
- **健康检查**。这个方案里没有涉及健康检查。因为本地运行服务可以是多样的，可以是基于Systemd，也可以使用Docker/Podman等容器方案。对于容器方案，可以考虑利用其自带的Healthcheck设施。统一的要求是，能够周期性检查本地所有服务，并及时更新serf agent的tag信息。
- **服务部署**。这个方案里同样没有涉及这一部分。可以利用Terraform，或者云服务商的提供的工具，例如AWS AutoScaling Group， Azure Virtual Machine Scale Sets等。

## 参考资料

- [What is a service mesh? - Redhat](https://www.redhat.com/en/topics/microservices/what-is-a-service-mesh)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Envoy](https://www.envoyproxy.io/)
- [hashicorp/serf - Github](https://github.com/hashicorp/serf)
- [AWS AutoScaling Group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
- [What are Virtual Machine Scale Sets? - Azure](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview)

