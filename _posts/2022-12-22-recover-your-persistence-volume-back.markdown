---
layout: post
title: "恢复误删的PV"
date: 2022-12-22 16:57:39 +0800
categories: 原创
---

K8s将持久化存储资源抽象为PersistentVolumeClaim （PVC）和 PersistentVolume（PV）的组合。其中PVC直接面相于Statefulset，而PV则隐藏于PVC之后。PVC可以隐式地与PV关联，也可以显式地声明关联。

## PersistentVolumeClaim （PVC）

PVC是对持久化资源的声明，本身并不会对某一个具体的硬件设备相关联。

在空白K8s集群上单独创建一个PVC：

```
➜   cat pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
➜   kubectl apply -f pvc.yaml
persistentvolumeclaim/pvc-test created
➜   kubectl get pvc
NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test   Pending                                      manual         5s
```

我们可以发现此时PVC的状态是`pending`。因为此刻并没有匹配的PV供关联。

## PersistentVolume（PV）

PV是对具体存储设施的抽象。其背后可以是本地存储空间，也可以云上持久化设备，例如AWS EBS等。这一点取决于具体的StorageClass。

在空白K8s集群上单独创建一个PV：

```
➜   cat pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-test
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp"
➜   kubectl apply -f pv.yaml
persistentvolume/pv-test created
➜   kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-test   1Gi        RWO            Retain           Available           manual                  4s
```

我们可以发现新创建的PV的`STATUS`是`Available`，对应`CLAIM`为空。这是因为这个PV没有与任何PVC关联。


## 隐式关联

PVC可以隐式地与PV关联。K8s可以不断寻找第一个满足PVC的spec要求的PV，并将两者绑定。

例如我们同时创建前述的PVC和PV，就可以发现

```
➜   kubectl get pvc
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test   Bound    pv-test   1Gi        RWO            manual         3s
➜   kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
pv-test   1Gi        RWO            Retain           Bound    default/pvc-test   manual                  7m53s
```

PVC和PV的`STATUS`都变成了`Bound`，而且PV的`CLAIM`也指向了新创建的PVC。

## 显式关联

另一种关联方式，可以通过在PVC的声明中指定PV来完成。

在空白集群中创建两个新PV，有相同的StorageClass和storage capacity。

```
➜   kubectl get pv
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-test     1Gi        RWO            Retain           Available           manual                  2m59s
pv-test-2   1Gi        RWO            Retain           Available           manual                  3s
```

此时如果新创建一个PVC依赖于隐式关联，那么其所绑定的PV是不明确的。

那么我们创建一个指定了PV的PVC：

```
➜  cat pvc2.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: pv-test # 指定了PV的name
➜   kubectl apply -f pvc2.yaml
persistentvolumeclaim/pvc-test created
➜   kubectl get pvc
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test   Bound    pv-test   1Gi        RWO            manual         10s
➜   kubectl get pv
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM              STORAGECLASS   REASON   AGE
pv-test-2   1Gi        RWO            Retain           Available                      manual                  2m17s
pv-test     1Gi        RWO            Retain           Bound       default/pvc-test   manual                  5m13s
```

在上述例子中，我们明确指定了要去绑定PV `pv-test`。所以结果是明确的。

## 删除PVC

如果我们删除一个之前绑定了PV的PVC，我们会发现对应PV仍然可见。

```
➜   kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM              STORAGECLASS   REASON   AGE
pv-test   1Gi        RWO            Retain           Released   default/pvc-test   manual                  12m
```

PV的`STATUS`变成了`Released`，而`CLAIM`保持不变，尽管对应的PVC已经不存在了。

删除PVC而能够保留PV。这一点其实收益于PV的`RECLAIM POLICY`。PV的`RECLAIM POLICY`，默认继承自StorageClass的配置，可以有三种值：
1. Retain：默认值。删除PVC后保留PV，留待手动处理。
2. Recycle：擦除原有资源，并准备再利用。
3. Delete：删除PVC则会删掉对应的持久化资源，例如AWS EBS。

### 恢复误删的PVC

假设我们误删了PVC，但PV还保留着，那如果想恢复PVC呢？

仅仅按照原来的配置直接创建PVC是不够的：

```
➜   kubectl apply -f pvc2.yaml
persistentvolumeclaim/pvc-test created
➜   kubectl get pvc
NAME       STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test   Pending   pv-test   0                         manual         5s
➜   kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM              STORAGECLASS   REASON   AGE
pv-test   1Gi        RWO            Retain           Released   default/pvc-test   manual                  19m
```

只是因为之前的PV仍然绑定在不存在的PVC上，没有真正的释放出来。

想要解除PV上的绑定，可以修改PV的配置，删除整个`spec.claimRef`配置段。

```
➜   kubectl edit pv pv-test
persistentvolume/pv-test edited
```

再检查一下：

```
➜   kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
pv-test   1Gi        RWO            Retain           Bound    default/pvc-test   manual                  23m
➜   kubectl get pvc
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test   Bound    pv-test   1Gi        RWO            manual         4m2s
```

大功告成！


## 参考资料

* [Kubernetes Documentation / Concepts / Storage / Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
* [Kubernetes Documentation / Reference / Kubernetes API / Config and Storage Resources / PersistentVolumeClaim](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-claim-v1/#PersistentVolumeClaimSpec)
* [How to reuse a PersistentVolume/PV in Kubernetes](https://letsdocloud.com/2020/09/how-to-reuse-a-persistentvolume-pv-in-kubernetes/)
