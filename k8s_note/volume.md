# volume

## Projected Volume 投射数据卷

为容器提供预先定义好的数据。

    Volume中的信息被Kubernetes投射入容器

目前Projected Volume一共有四种

* Secret
* ConfigMap
* Downward API
* ServiceAccountToken

Secret、ConfigMap，以及 Downward API 这三种 Projected Volume 定义的信息，大多还可以通过环境变量的方式出现在容器里。但是，通过环境变量获取这些信息的方式，不具备自动更新的能力

### Secret

把Pod想要访问的加密数据存放到Etcd中，然后在Pod的容器里通过挂载Volume的方式访问

#### 创建方式

命令行 `kubectl create secret`

```
$ cat ./username.txt
admin
$ cat ./password.txt
c1oudc0w!

$ kubectl create secret generic user --from-file=./username.txt
$ kubectl create secret generic pass --from-file=./password.txt
```

编写YAML的方式

Secret 对象要求这些数据必须是经过 Base64 转码的，以免出现明文密码的安全隐患。

```

apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YWRtaW4=
  pass: MWYyZDFlMmU2N2Rm
```

在真正的生产环境中，需要在 Kubernetes 中开启 Secret 的加密插件，增强数据的安全性.

#### 数据更新

kubelet组件定时维护这些Volume

一旦Secret对应的Etcd里的数据被更新，Volume里的文件内容也会被更新。

更新会有延时

### ConfigMap

保存不需要加密的应用所需的配置信息，用法几乎与Secret相同

#### 创建方式

* kubectl create configmap 

* YAML

### Downward API

让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息。

只能获取到Pod里容器进程启动之前就能确定下来的信息

### ServiceAccountToken

Service Account: kubernetes进行权限分配的对象

Service Account的授权信息和文件保存在ServiceAccountToken里

任何运行在Kubernetes集群上的应用都必须使用ServiceAccountToken中的授权信息才能合法访问API Server