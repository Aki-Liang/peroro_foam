# kubeadm

2017年社区发起的独立部署工具
（不可用于生产环境）

#### kubeadm init工作流程

1. Preflight Check
2. 生成证书和对应的目录
3. 生成配置文件 `/etc/kubernetes/xxx.conf`
4. 为Master组件生成Pod配置文件
   1. kube-apiserver
   2. kube-controller-manager
   3. kube-scheduler
   
    配置文件路径`/etc/kubernetes/manifests`
5. 生成ETCD的Pod yaml文件
6. kubelet根据yaml文件创建Pod
7. 生成bootstrap token
8. 将证书和Master节点信息保存到ETCD `cluster-info`
9. 安装默认插件`kube-proxy`和`DNS`

#### kubeadm join工作流程

1. 拿到 kubeadm inti生成的bootstrap token后,在任意一台安装了 kubelet 和 kubeadm 的机器上执行 kubeadm join
2. 使用token来获取ConfigMap 中的 cluster-info（保存了 APIServer 的授权信息）
3. 有了 cluster-info 里的 kube-apiserver 的地址、端口、证书，kubelet 就可以以“安全模式”连接到 apiserver 上

#### 配置kubeadm的部署参数

使用命令`kubeadm init --config kubeadm.yaml`