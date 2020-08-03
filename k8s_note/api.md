# api

### 声明式api

kubectl apply 命令

    kubectl replace 的执行过程，是使用新的 YAML 文件中的 API 对象，替换原有的 API 对象；
    而 kubectl apply，则是执行了一个对原有 API 对象的 PATCH 操作

    kubectl set image 和 kubectl edit 也是对已有 API 对象的修改。

这意味着 kube-apiserver 在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），一次能处理多个写操作，并且具备 Merge 能力

* 首先，所谓“声明式”，指的就是我只需要提交一个定义好的 API 对象来“声明”，我所期望的状态是什么样子。
* 其次，“声明式 API”允许有多个 API 写端，以 PATCH 的方式对 API 对象进行修改，而无需关心本地原始 YAML 文件的内容。
* 最后，也是最重要的，有了上述两个能力，Kubernetes 项目才可以基于对 API 对象的增、删、改、查，在完全无需外界干预的情况下，完成对“实际状态”和“期望状态”的调谐（Reconcile）过程。

### API对象在Etcd中的完整资源路径

有Group(API组)、Version(API版本)和Resource(API资源类型)三个部分组成

例

```
apiVersion: batch/v2alpha1
kind: CronJob
...
```

资源类型（Resource）：CronJob

组（Group）：batch

版本（Version）:v2alpha1

#### 资源定位

1. K8S匹配API对象的组
```
    K8S核心API对象：Pod Node等，不需要Group。K8S直接在/api层级进行下一步的匹配

    非K8S核心API对象，必须在/apis这个层级查找对应的Group
```

2. 进一步匹配API对象的版本号

```
    在 Kubernetes 中，同一种 API 对象可以有多个版本
```

3. 最后，Kubernetes 会匹配 API 对象的资源类型


### 思考题

你是否对 Envoy 项目做过了解？你觉得为什么它能够击败 Nginx 以及 HAProxy 等竞品，成为 Service Mesh 体系的核心？

    因为envoy提供了api形式的配置入口，更方便做流量治理
    编程友好的api，方便容器化，配置方便