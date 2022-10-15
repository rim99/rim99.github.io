---
layout: post
title: "关于K8S Pod生命周期的笔记"
date: 2022-10-15 12:08:14 +0800
categories: 原创
---

Pod是K8S的最小部署单元，其中运行着用户的服务。

## Pod Status

在查看Pod的时候，我们可以看到Pod的属性中有一个`status`字段，其中定义了Pod的状态。

状态      | 描述
:-       | :-
Pending  | 正在部署中
Running  | 所有Container已经创建成功。其中至少有一个正在运行
Succeeded| 所有Container都已经正确关闭（`exit code == 0`）
Failed   | 所有Container都已经关闭。其中至少有一个非正确关闭（`exit code ！= 0`）
Unknown  | 无法获取Pod状态

### Pod Condition

Pod的Status还包括`condition`字段。

状态            | 描述
:-             | :-
PodScheduled   | 已经找到运行该Pod的Node
PodHasNetwork  | Pod沙盒创建成功，网络配置完成
ContainersReady| 所有Container都已经Ready
Initialized    | 所有init Container都已经正确关闭
Ready          | Pod可以正常服务请求了

Pod处于Ready状态需要满足两点要求：
1. 所有Container都处于Ready状态
2. Pod的readinessGates所列举的所有属性（由用户自定义）在Pod的conditions中为True

## Container

Pod的状态和其中的Container状态的关系非常密切。

Container是Pod的内部组成部分。一个Pod中可以有多个Container。每一个Container都运行着一些进程。不同的Container可以基于不同的image运行。

尽管同一个Container内部可以同时运行多个进程，但是Container的主进程只有一个，其**pid为1**。只要pid为1的进程还在持续运行，那么Container就不会关闭。

同样的，如果想要让同一Container里面的其他进程的输出展示在Container的console log里面，必须将内容重定向到该进程的标准输出。具体来说就是，`echo "log" > /proc/1/fd/1`。相应的，`/proc/1/fd/2`就是其标准错误。

尽管不同的Container的log是单独输出，但是他们共享了Pod的文件系统、网络配置等等。

Container可以在Pod的整个生命周期内持续运行。Pod可以定义Container的重启策略。来决定Container是否需要（以及如何）在关闭后重启。该策略是Pod级别的，会在对其中所有的Container生效。

除此之外，还有一种init Container，只在Pod的创建初期顺序运行，运行完毕后直接关闭。只有当所有init Container运行完毕后，Container才会创建并运行。

### Container Status

状态         | 描述
:-          | :-
Running     | 正在运行中
Terminated  | Container里面的进程结束运行，可能正确结束，也可能非正确。
Waiting     | 除上述以外的场景都有可能。例如正在拉取image等等。具体原因可以查看`Reason`字段

#### Container状态探针

每一个Container支持使用自定义探针来决定自身状态。探针可以是shell命令，也可以是网络请求，例如HTTP GET请求、GRPC请求或者TCP请求。

探针类型有三种：
* `livenessProbe`。如果该探针执行失败，Container就会被kubelet关闭。然后根据Pod的restartPolicy决定是否重启。
* `readinessProbe`。如果该探针执行失败，Pod就会从相关的Service的Endpoints列表中清除，不再接受网络请求，直到探针成功（P.S. 此时对外发送请求没有问题）。
* `startupProbe`。如果该探针执行失败，Container就会被kubelet关闭。然后根据Pod的restartPolicy决定是否重启。在该探针执行成功之前，前述探针均保持关闭。

当没有配置相应探针时，默认其为成功。

### Container Lifecycle Hook

每一个Container支持自定义自己的hook来更好的管理服务进程。hook有两种：`postStart`和`preStop`，分别在容器内进程**启动时**和**关闭前**执行。

具体来说，
* 在Container成为`Running`状态之前， `postStart` hook已经执行完毕。
* 在Container成为`Terminated`状态之前， `preStop` hook已经执行完毕。

但是，`postStart` hook和Container的entrypoint会同时执行。而`preStop` hook则是需要在向Container发送`SIGTERM`信号之前结束执行。

Hook的实现可以有两种：
* shell命令。Shell命令在Container内部执行，会占用Container定义的Resource。
* http GET请求。http请求由kubelet进程执行。

K8S保证自定义hook至少执行一次。因此有可能执行多次。用户需要自行确保其幂等性。

#### 如何Debug Hook

hook执行成功时，Container console不会显示其log。如果需要查看，就需要用到前述的重定向方式。

K8S可以查看到hook执行失败的记录。例如`kubectl describe pod XXX`或者干脆`kubectl get events | grep YYY`。

## 关于Pod的优雅关闭

优雅关闭对于网络服务的服务质量非常重要。

Pod在接收到关闭请求之前会先尝试优雅关闭，同时计时开始。当达到时间限制之后，Pod才会被直接关闭。

例如当用户使用`kubectl`发送关闭指令。

K8S首先打开计时器，将Pod标记为`Terminating`。

然后并行的关闭执行每一个Container：如果有preStop hook就先执行hook，然后向container发送`SIGTERM`信号给Container的主进程（pid为1）。

Pod在被标记为`Terminating`的同时，也被移出了Service的Endpoints列表，不再会接收到外部请求。根据本人实测，此时从Container向Pod外请求也是失败的。

如果在优雅关闭时限内，Pod没有完成关闭。其所有存活的Container内的所有进程都会接收到`SIGKILL`信号。同时，kubelet会要求API Server将Pod对象强制清理干净。待API Server完成清理后，Pod不再可见。

优雅关闭时间限制由Pod属性`terminationGracePeriodSeconds`来定义。

## 参考资料

* [Pod Lifecycle - Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
* [Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/)
* [How to redirect PreStop hook output to pod logs](https://shyamjos.com/redirect-prestop-hook-logs/)
* [Pod Ready++](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/580-pod-readiness-gates)
