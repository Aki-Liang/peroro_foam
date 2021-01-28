
# deployment

Deployment 实际上是一个两层控制器。它通过 ReplicaSet 的个数来描述应用的版本；然后，它再通过 ReplicaSet 的属性（比如 replicas 的值），来保证 Pod 的副本数量

适用于封装“无状态应用”（Stateless Application），尤其是 Web 服务，非常好用

```
  Deployment 控制 ReplicaSet（版本）
  ReplicaSet 控制 Pod（副本数）
```

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

一定要使用 Pod 的 Health Check 机制检查应用的运行状态，而不是简单地依赖于容器的 Running 状态。要不然的话，虽然容器已经变成 Running 了，但服务很有可能尚未启动，“滚动更新”的效果也就达不到了
```

为了进一步保证服务的连续性，Deployment Controller 还会确保，在任何时间窗口内，只有指定比例的 Pod 处于离线状态。同时，它也会确保，在任何时间窗口内，只有指定比例的新 Pod 被创建出来。这两个比例的值都是可以配置的，默认都是 DESIRED 值的 25%

这个策略，是 Deployment 对象的一个字段，名叫 RollingUpdateStrategy

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

maxSurge 指定的是除了 DESIRED 数量之外，在一次“滚动”中，Deployment 控制器还可以创建多少个新 Pod
 maxUnavailable 指的是，在一次“滚动”中，Deployment 控制器可以删除多少个旧 Pod

这两个配置还可以用百分比形式来表示，比如：maxUnavailable=50%，指的是最多可以一次删除“50%*DESIRED 数量”个 Pod


#### 回滚

回滚到上一个版本

* `kubectl rollout undo` 

回滚到更早之前的版本

* 使用 `kubectl rollout history` 命令，查看每次 Deployment 变更对应的版本
* kubectl rollout undo deployment/nginx-deployment --to-revision=<your_version>

对 Deployment 进行的每一次更新操作都会生成一个新的 ReplicaSet 对象.

`kubectl rollout pause`和`kubectl rollout resume`之间所有修改都不会触发滚动更新，也不会创建新的ReplicaSet，只会在resume之后统一触发一次

## 思考题

你听说过金丝雀发布（Canary Deployment）和蓝绿发布（Blue-Green Deployment）吗？你能说出它们是什么意思吗？

金丝雀部署：优先发布一台或少量机器升级，等验证无误后再更新其他机器。优点是用户影响范围小，不足之处是要额外控制如何做自动更新。
蓝绿部署：2组机器，蓝代表当前的V1版本，绿代表已经升级完成的V2版本。通过LB将流量全部导入V2完成升级部署。优点是切换快速，缺点是影响全部用户。

## 实践

https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/canary