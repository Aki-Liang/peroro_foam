# pod

### 概念

Pod是Kubernetes中最小编排单位

Pod扮演的的是传统部署环境里虚拟机的角色

    为了使用户从传统环境向Kubernetes环境的迁移更加平滑
Pod 这个看似复杂的 API 对象，实际上就是对容器的进一步抽象和封装而已

Pod级别的属性

*  调度
*  网络
*  存储
*  安全

Pod 的设计，是要里面的容器尽可能多地共享 Linux Namespace，仅保留必要的隔离和限制能力

凡是跟容器的 Linux Namespace 相关的属性，一定是 Pod 级别的

凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义


### 重要字段

#### NodeSelector

将Pod与Node绑定的字段

    apiVersion: v1
    kind: Pod
    ...
    spec:
        nodeSelector:
            disktype:ssd

以上配置表示，该pod只能运行在携带了`disktype:ssd`标签的节点上

#### NodeName

一旦Pod的这个字段被赋值，Kubernetes会认为这个Pod已经经过了调度

该字段一般由调度器负责设置

#### HostAliases

定义Pod的host文件（如/etc/hosts）里的内容

    apiVersion: v1
    kind: Pod
    ...
    spec: 
        hostAliases: 
        - ip: "10.1.2.3" 
          hostnames: 
          - "foo.remote" 
          - "bar.remote"
    ...

该Pod启动后 /etc/hosts文件内容如下

    cat /etc/hosts
    # Kubernetes-managed hosts file.
    127.0.0.1 localhost
    ...
    10.244.135.10 hostaliases-pod
    10.1.2.3 foo.remote
    10.1.2.3 bar.remote

在 Kubernetes 项目中，一定要通过HostAliases来设置 hosts 文件里的内容

如果直接修改hosts 文件，Pod 被删除重建之后，kubelet 会自动覆盖掉被修改的内容

#### ImagePullPolicy

定义了镜像拉取的策略

默认值是 Always, 每次创建 Pod 都重新拉取一次镜像

    当容器的镜像是类似于 nginx 或者 nginx:latest 这样的名字时，ImagePullPolicy 也会被认为 Always

#### Lifecycle

Container Lifecycle Hooks,在容器状态发生变化时触发一系列“钩子”

### Pod生命周期

Pod 生命周期的变化，主要体现在 Pod API 对象的 Status 部分

* Pending
    ```
    Pod的YAML文件已经提交给Kubernetes，API对象已经被创建并保存到Etcd，但是该容器因为某些原因不能被顺利创建
    ```
* Running
    ```
    Pod已经调度成功，并和一个具体的的节点绑定。Pod包含的容器都已经创建成功，且至少有一个正在运行
    ```

* Succeeded
    ```
    Pod中所有的容器都正常运行完毕，且全部退出。
    多见于一次性任务
    ```
* Failed
    ```
    Pod中至少有一个容器以不正常状态退出（非0返回码）
    需要Debug容器应用，查看Pod Events和日志
    ```

* unknown
    ```
    Pod状态无法持续地被kubelet汇报给kube-apiserver
    有可能是主从节点（Master和Kubelet）间的通信出现了问题
    ```

Status字段还能再细分出一组Conditions，用于描述造成当前Status的具体原因

* PodScheduled
* Ready
* Initialized
* Unschedulable