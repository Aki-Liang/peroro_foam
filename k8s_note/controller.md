# controller


### kube-controller-manager组件

控制器的集合


### 控制循环编排模式

pkg/controller目录下的控制器遵循Kubernetes项目中的通用编排模式，控制循环（control loop）

```
for { 
    实际状态 := 获取集群中对象X的实际状态（Actual State） 
    期望状态 := 获取集群中对象X的期望状态（Desired State） 
    if 实际状态 == 期望状态{ 
        什么都不做 
    } else { 
        执行编排动作，将实际状态调整为期望状态 
    }
}
```

实际状态来自Kubernetes集群本身

* kubelet 通过心跳汇报的容器状态和节点状态
* 监控系统中保存的应用监控数据
* 等
  
期望状态一般来自用户提交的YAML文件, 这些信息往往都保存在 Etcd

* Deployment 对象中 Replicas 字段的值

### Deployment对控制器模型的实现

1. Deployment 控制器从 Etcd 中获取到所有携带了“app: nginx”标签的 Pod，然后统计它们的数量，这就是实际状态
2. Deployment 对象的 Replicas 字段的值就是期望状态
3. Deployment 控制器将两个状态做比较，然后根据比较结果，确定是创建 Pod，还是删除已有的 Pod
   
这个操作被称为调谐（Reconcile）。这个调谐的过程，则被称作“Reconcile Loop”（调谐循环）或者“Sync Loop”（同步循环）【都是指的控制循环】

调谐的最终结果，都是对被控制对象的某种写操作。

Deployment 这种控制器的设计原理，就是“用一种对象管理另一种对象”

Deployment 定义的 template 字段，在 Kubernetes 项目中有一个专有的名字，叫作 PodTemplate。
大多数控制器都会使用PodTemplate来统一定义它所要管理的Pod。


```
Kubernetes 使用的这个“控制器模式”，跟我们平常所说的“事件驱动”，有什么区别和联系


“事件驱动”，对于控制器来说是被动，只要触发事件则执行，对执行后不负责，无论成功与否，没有对一次操作的后续进行“监控”
“控制器模式”，对于控制器来说是主动的，自身在不断地获取信息，起到事后“监控”作用，知道同步完成，实际状态与期望状态一致
事件往往是一次性的，如果操作失败比较难处理，但是控制器是循环一直在尝试的，更符合kubernetes申明式API，最终达到与申明一致

```

### Deployment实现Pod的水平扩展/收缩

Deployment遵循滚动更新的方式来升级现有容器

滚动更新依赖ReplicaSet来实现

```
ReplicaSet YAML file

apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-set
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
```

一个 ReplicaSet 对象，其实就是由副本数目的定义和一个 Pod 模板组成的

Deployment 控制器实际操纵的是ReplicaSet 对象，而不是 Pod 对象。

对于一个 Deployment 所管理的 Pod，它的 ownerReference 是谁？ReplicaSet

ReplicaSet 负责通过“控制器模式”，保证系统中 Pod 的个数永远等于指定的个数，这也正是 Deployment 只允许容器的 restartPolicy=Always 的主要原因：只有在容器能保证自己始终是 Running 状态的前提下，ReplicaSet 调整 Pod 的个数才有意义

Deployment 同样通过“控制器模式”，来操作 ReplicaSet 的个数和属性，进而实现“水平扩展 / 收缩”和“滚动更新”这两个编排动作

水平扩展 / 收缩”非常容易实现，Deployment Controller 只需要修改它所控制的 ReplicaSet 的 Pod 副本个数就可以了

```

$ kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
```

### 滚动更新的实现

1. 当修改了 Deployment 里的 Pod 定义之后，Deployment Controller 会使用这个修改后的 Pod 模板，创建一个新的 ReplicaSet，这个新的 ReplicaSet 的初始 Pod 副本数是：0
2. Deployment Controller 开始将这个新的 ReplicaSet 所控制的 Pod 副本数从 0 个变成 1 个，即：“水平扩展”出一个副本。
3. Deployment Controller 又将旧的 ReplicaSet所控制的旧 Pod 副本数减少一个，即：“水平收缩”成两个副本。
4. 如此交替进行

好处

```
在升级刚开始的时候，集群里只有 1 个新版本的 Pod。如果这时，新版本 Pod 有问题启动不起来，那么“滚动更新”就会停止，从而允许开发和运维人员介入。而在这个过程中，由于应用本身还有两个旧版本的 Pod 在线，所以服务并不会受到太大的影响。
```
