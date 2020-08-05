# storage

K8S容器持久化存储



## PV （Persistent Volume）

`持久化存储数据卷`

这个 API 对象主要定义的是一个持久化存储在宿主机上的目录，比如一个 NFS 的挂载目录

通常情况下，PV 对象是由运维人员事先创建在 Kubernetes 集群里待用的

持久化 Volume，指宿主机上的目录，具备“持久性”。这个目录里面的内容，既不会因为容器的删除而被清理掉，也不会跟当前的宿主机绑定。这样，当容器被重启或者在其他节点上重建出来之后，它仍然能够通过挂载这个 Volume 来访问到这些内容

大多数情况下，持久化 Volume 的实现，往往依赖于一个远程存储服务，比如：远程文件存储（比如，NFS、GlusterFS）、远程块存储（比如，公有云提供的远程磁盘）等等

### 两阶段处理

#### Attach

将存储资源挂载到Pod所在的宿主机上

如果 Volume 类型是远程块存储，比如 Google Cloud 的 Persistent Disk（GCE 提供的远程磁盘服务），kubelet 就需要先调用 Goolge Cloud 的 API，将它所提供的 Persistent Disk 挂载到 Pod 所在的宿主机上

如果 Volume 类型是远程文件存储（比如 NFS）的话，kubelet 可以跳过Attach的操作，远程文件存储并没有一个“存储设备”需要挂载在宿主机上。

#### Mount

格式化磁盘设备，并将其挂载到宿主机的指定挂载点上（Volume的宿主机目录）

#### 区别

在具体的 Volume 插件的实现接口上，Kubernetes 分别给这两个阶段提供了两种不同的参数列表：

* 对于“第一阶段”（Attach），Kubernetes 提供的可用参数是 nodeName，即宿主机的名字。
* 对于“第二阶段”（Mount），Kubernetes 提供的可用参数是 dir，即 Volume 的宿主机目录。



    备注：对应地，在删除一个 PV 的时候，Kubernetes 也需要 Unmount 和 Dettach 两个阶段来处理。这个过程我就不再详细介绍了，执行“反向操作”即可。


## PVC （Persistent Volume Claim）

`Pod 所希望使用的持久化存储的属性`

PVC 对象通常由开发人员创建；或者以 PVC 模板的方式成为 StatefulSet 的一部分，然后由 StatefulSet 控制器负责创建带编号的 PVC

而用户创建的 PVC 要真正被容器使用起来，就必须先和某个符合条件的 PV 进行绑定。这里要检查的条件，包括两部分

1. PV 和 PVC 的 spec 字段。比如，PV 的存储（storage）大小，就必须满足 PVC 的要求。
2. PV 和 PVC 的 storageClassName 字段必须一样

## StorageClass

`Static Provisioning`： 人工管理 PV 的方式

`Dynamic Provisioning`： Kubernetes 提供的一套可以自动创建 PV 的机制

`StorageClass`： Dynamic Provisioning 机制工作的核心

    StorageClass 并不是专门为了 Dynamic Provisioning 而设计的。


StorageClass 对象会定义如下两个部分内容：

1. PV 的属性。比如，存储类型、Volume 的大小等等。
2. 创建这种 PV 需要用到的存储插件。比如，Ceph 等等

应用开发者只需要在 PVC 里指定要使用的 StorageClass 名字，在创建PVC对象后Kubernetes会自动创建PV

    Kubernetes 只会将 StorageClass 相同的 PVC 和 PV 绑定起来

StorageClass 示例：
```
Volume 的类型是 GCE 的 Persistent Disk的例子

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: kubernetes.io/gce-pd //Kubernetes 内置的 GCE PD 存储插件
parameters:
  type: pd-ssd             //“SSD 格式的 GCE 远程磁盘”
```


```
Rook的例子

apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  #The value of "clusterNamespace" MUST be the same as the one in which your rook cluster exist
  clusterNamespace: rook-ceph
```

## Volume Controller

专门处理持久化存储的控制器

维护着多个控制循环

### 控制循环PersistentVolumeController

作用是撮合 PV 和 PVC，PersistentVolumeController 会不断地查看当前每一个 PVC，是不是已经处于 Bound（已绑定）状态。如果不是，那它就会遍历所有的、可用的 PV，并尝试将其与这个未绑定的 PVC 进行绑定

    绑定: 将 PV 对象的名字，填在 PVC 对象的 spec.volumeName 字段上

### 控制循环AttachDetachController

维护Attach和Detach操作

不断地检查每一个 Pod 对应的 PV，和这个 Pod 所在宿主机之间挂载情况。从而决定，是否需要对这个 PV 进行 Attach（或者 Dettach）操作

    作为一个 Kubernetes 内置的控制器，Volume Controller 是 kube-controller-manager 的一部分
    AttachDetachController 一定是运行在 Master 节点上的

### 控制循环VolumeManagerReconciler

维护Mount和Unmount操作

由于Mount和Unmount必须发生在 Pod 对应的宿主机上，所以它必须是 kubelet 组件的一部分

是一个独立于 kubelet 主循环的 Goroutine

    why：
    通过这样将 Volume 的处理同 kubelet 的主循环解耦，Kubernetes 就避免了这些耗时的远程挂载操作拖慢 kubelet 的主控制循环，进而导致 Pod 的创建效率大幅下降的问题
    kubelet 的一个主要设计原则，就是它的主控制循环绝对不可以被 block