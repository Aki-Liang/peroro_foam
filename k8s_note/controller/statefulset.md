# statefulset

针对有状态应用的编排。

记录状态，在Pod被重新创建时为Pod恢复状态

StatefulSet 其实就是一种特殊的 Deployment,它的每个 Pod 都被编号了。这个编号会体现在 Pod 的名字和 hostname 等标识信息上，这不仅代表了 Pod 的创建顺序，也是 Pod 的重要网络标识

有了这个编号，StatefulSet 使用 Kubernetes 里的两个标准功能：Headless Service 和 PV/PVC，实现了对 Pod 的拓扑状态和存储状态的维护

StatefulSet 可以说是 Kubernetes 项目中最为复杂的编排对象

## StatefulSet状态抽象

### 拓扑状态

应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动
新创建出来的 Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod

### 存储状态

应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例
