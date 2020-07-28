# daemonset

用来运行DaemonPod

    这个Pod运行在K8S集群的每一个Node上
    每个节点上只能有一个这种Pod实例
    当有新的节点加入K8S集群之后，该Pod会自动的在新节点上被创建出来
    当旧节点被删除后，上面的Pod也会被回收

例子

    网络插件的Agent组件
    存储插件的Agent组件
    监控组件和日志组件

DaemonSet的运行时机很多时候比K8S集群出现的时机都要早

### 如何保证每个 Node 上有且只有一个被管理的 Pod

DaemonSet Controller，首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的 Node。检查当前Node上是否有一个携带了指定表情的Pod在运行


1. 没有这种 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod；
2. 有这种 Pod，但是数量大于 1，那就说明要把多余的 Pod 从这个 Node 上删除掉；
3. 正好只有一个这种 Pod，那说明这个节点是正常的。


删除多余的Pod

    直接调用 Kubernetes API

在指定的Node上创建新Pod

    nodeAffinity

### 添加tolerations来解决

添加tolerations字段意味着这个 Pod，会“容忍”（Toleration）某些 Node 的“污点”（Taint）

    DaemonSet 自动加上的 tolerations 字段，格式如下所示
    apiVersion: v1
    kind: Pod
    metadata:
      name: with-toleration
    spec:
      tolerations:
      - key: node.kubernetes.io/unschedulable
        operator: Exists
        effect: NoSchedule

默认情况下K8S集群不允许用户在Master节点部署Pod

    添加tolerations来解决

    
    tolerations:
    - key: node-role.kubernetes.io/master
      effect: NoSchedule


在 DaemonSet 上，一般都应该加上 resources 字段，来限制它的 CPU 和内存使用，防止它占用过多的宿主机资源

### 版本控制

Deployment使用一个版本对应一个ReplicaSet对象的方式来实现版本控制

DaemonSet控制器直接操作Pod，没有ReplicaSet对象。

    使用ControllerRevision API对象

    ControllerRevision 对象实际上是在 Data 字段保存了该版本对应的完整的 DaemonSet 的 API 对象。并且，在 Annotation 字段保存了创建这个对象所使用的 kubectl 命令