# rbac

基于角色的权限控制

Kubernetes 中所有的 API 对象，都保存在 Etcd 里。

对这些 API 对象的操作，一定都是通过访问 kube-apiserver 实现的。

其中一个非常重要的原因，就是需要 APIServer 来做授权工作

在 Kubernetes 项目中，负责完成授权（Authorization）工作的机制，就是 RBAC：基于角色的访问控制（Role-Based Access Control

## 基本概念

1. Role: 角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限
2. Subject: 被作用者，既可以是“人”，也可以是“机器”，也可以是在 Kubernetes 里定义的“用户”
3. RoleBinding：定义了“被作用者”和“角色”的绑定关系。

### Role

Role本身是一个K8S的API对象

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

rules字段定义权限规则，上面的例子中允许“被作用者”，对 mynamespace 下面的 Pod 对象，进行 GET、WATCH 和 LIST 操作

### RoleBinding

指定Role的“被作用者”

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

subjects字段指定“被作用者”，类型为User，即 Kubernetes 里的用户，这个用户的名字是 example-user（然而K8S中并没有叫做User的API对象）

    K8S中的User是授权系统中的逻辑概念，需要通过外部认证服务来提供。或者直接给APIServer指定一个用户名密码文件。在大多数私有的使用环境中，只要使用 Kubernetes 提供的“内置用户”就足够了

roleRef字段定义被作用者和Role之间的绑定关系，通过这个字段RoleBinding对象就可以直接通过名字来引用Role对象。

    Role和RoleBinding对象都是Namespaced对象，它们对权限的限制规则仅在自己的Namespace内有效。roleRef也只能引用当前Namespace里的Role对象

## 非Namespaced（Non-namespaced）对象的授权（或一个Role想作用于所有的Namespace）

### ClusterRole

### ClusterRoleBinding

## 内置用户

Kubernetes 负责管理的“内置用户”：ServiceAccount

### 定义ServiceAccount

```

apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
```

### 用户和用户组

一个 ServiceAccount，在 Kubernetes 里对应的“用户”的名字是

    system:serviceaccount:<Namespace名字>:<ServiceAccount名字>

对应的内置“用户组”的名字是

    system:serviceaccounts:<Namespace名字>

## 系统保留ClusterRole

名字以system:开头

一般来说，这些系统 ClusterRole，是绑定给 Kubernetes 系统组件对应的 ServiceAccount 使用的

### 预定义给用户使用的ClusterRole

* cluster-admin；
* admin；
* edit；
* view。

