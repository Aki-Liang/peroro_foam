# storage

K8S容器持久化存储

## PV （Persistent Volume）

`持久化存储数据卷`

这个 API 对象主要定义的是一个持久化存储在宿主机上的目录，比如一个 NFS 的挂载目录

通常情况下，PV 对象是由运维人员事先创建在 Kubernetes 集群里待用的

## PVC （Persistent Volume Claim）

`Pod 所希望使用的持久化存储的属性`

PVC 对象通常由开发人员创建；或者以 PVC 模板的方式成为 StatefulSet 的一部分，然后由 StatefulSet 控制器负责创建带编号的 PVC

而用户创建的 PVC 要真正被容器使用起来，就必须先和某个符合条件的 PV 进行绑定。这里要检查的条件，包括两部分

1. PV 和 PVC 的 spec 字段。比如，PV 的存储（storage）大小，就必须满足 PVC 的要求。
2. PV 和 PVC 的 storageClassName 字段必须一样

## StorageClass