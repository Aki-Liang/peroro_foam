# volume

## Projected Volume 投射数据卷

为容器提供预先定义好的数据。

    Volume中的信息被Kubernetes投射入容器

目前Projected Volume一共有四种

* Secret
* ConfigMap
* Downward API
* ServiceAccountToken

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