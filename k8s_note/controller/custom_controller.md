# custom_controller

“声明式 API”并不像“命令式 API”那样有着明显的执行逻辑

基于声明式 API 的业务功能实现，往往需要通过控制器模式来“监视”API 对象的变化（比如，创建或者删除 Network），然后以此来决定实际要执行的具体工作

## 编写自定义控制器代码

1. 编写main函数
2. 编写自定义控制器的定义
3. 编写控制器的业务逻辑


### 编写main函数

```
func main() {
  ...
  
  cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
  ...
  kubeClient, err := kubernetes.NewForConfig(cfg)
  ...
  networkClient, err := clientset.NewForConfig(cfg)
  ...
  
  networkInformerFactory := informers.NewSharedInformerFactory(networkClient, ...)
  
  controller := NewController(kubeClient, networkClient,
  networkInformerFactory.Samplecrd().V1().Networks())
  
  go networkInformerFactory.Start(stopCh)
 
  if err = controller.Run(2, stopCh); err != nil {
    glog.Fatalf("Error running controller: %s", err.Error())
  }
}
```

第一步，main函数先根据master配置，创建K8S的Client和network对象的clientcmd.

    没有提供Master配置的情况下，main函数会直接使用InClusterConfig的方式来创建client，该方式会假定自定义控制器是以Pod的方式运行在K8S集群里的

第二步，main函数为network对象创建一个InformerFactory，并使用该工程生成一个Network对象的Informer，传递给控制器。

第三步，main函数启动Informer，然后执行controller.Run，启动自定义控制器

### 编写控制器定义

例

```
func NewController(
  kubeclientset kubernetes.Interface,
  networkclientset clientset.Interface,
  networkInformer informers.NetworkInformer) *Controller {
  ...
  controller := &Controller{
    kubeclientset:    kubeclientset,
    networkclientset: networkclientset,
    networksLister:   networkInformer.Lister(),
    networksSynced:   networkInformer.Informer().HasSynced,
    workqueue:        workqueue.NewNamedRateLimitingQueue(...,  "Networks"),
    ...
  }
    networkInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: controller.enqueueNetwork,
    UpdateFunc: func(old, new interface{}) {
      oldNetwork := old.(*samplecrdv1.Network)
      newNetwork := new.(*samplecrdv1.Network)
      if oldNetwork.ResourceVersion == newNetwork.ResourceVersion {
        return
      }
      controller.enqueueNetwork(new)
    },
    DeleteFunc: controller.enqueueNetworkForDelete,
 return controller
}
```

main函数中创建了两个client，在上面的代码中使用者两个client和之前创建的Informer初始化了自定义控制器

    代码中的自定义控制器里，设置了一个工作队列，负责同步Informer和控制循环之间的数据
    K8S项目中提供了很多个工作队列的实现，可以根据需要选择合适的库直接使用

以上代码注册了三个 Handler（AddFunc、UpdateFunc 和 DeleteFunc），分别对应 API 对象的“添加”“更新”和“删除”事件。而具体的处理操作，都是将该事件对应的 API 对象加入到工作队列中。

    实际入队的不是API对象本身，而是API对象的Key ： <namesapce>/<name>

### 控制循环

```

func (c *Controller) Run(threadiness int, stopCh <-chan struct{}) error {
 ...
  if ok := cache.WaitForCacheSync(stopCh, c.networksSynced); !ok {
    return fmt.Errorf("failed to wait for caches to sync")
  }
  
  ...
  for i := 0; i < threadiness; i++ {
    go wait.Until(c.runWorker, time.Second, stopCh)
  }
  
  ...
  return nil
}
```

### 自定义控制器工作流程

![自定义控制器工作流程图](./custom_controller.png)

控制器需要通过Informer从K8S的APIServer里获取它关系的对象，Informer与API对象是一一对应的。


### Informer

所谓 Informer，其实就是一个带有本地缓存和索引机制的、可以注册 EventHandler 的 client；是自定义控制器跟 APIServer 进行数据同步的重要组件

#### 职责

#### 缓存同步

Informer使用了Reflector包中的ListAndWatch方法，来获取监听对象的实例的变化。在 ListAndWatch 机制下，一旦 APIServer 端有新的 Network 实例被创建、删除或者更新，Reflector 都会收到“事件通知”。这时，该事件及它对应的 API 对象这个组合，就被称为增量（Delta），它会被放进一个 Delta FIFO Queue中。

另一方面，Informe 会不断地从这个 Delta FIFO Queue 里读取（Pop）增量。每拿到一个增量，Informer 就会判断这个增量里的事件类型，然后创建或者更新本地对象的缓存。这个缓存，在 Kubernetes 里一般被叫作 Store。

如果事件类型是 Added（添加对象），那么 Informer 就会通过一个叫作 Indexer 的库把这个增量里的 API 对象保存在本地缓存中，并为它创建索引。相反，如果增量的事件类型是 Deleted（删除对象），那么 Informer 就会从本地缓存中删除这个对象

#### 根据事件类型触发EventHandler

这些 Handler，需要在创建控制器的时候注册给它对应的 Informer

#### resync

每经过 resyncPeriod 指定的时间，Informer 维护的本地缓存都会使用最近一次 LIST 返回的结果强制更新一次，从而保证缓存的有效性。在 Kubernetes 中，这个强制更新缓存的操作就叫作resync

    这个定时 resync 操作，也会触发 Informer 注册的“更新”事件。但此时，这个“更新”事件对应的 Network 对象实际上并没有发生变化，即：新、旧两个 Network 对象的 ResourceVersion 是一样的。


### 思考题

为什么 Informer 和你编写的控制循环之间，一定要使用一个工作队列来进行协作呢？

    Informer 和控制循环分开是为了解耦，防止控制循环执行过慢把Informer 拖死

    一般这种工作队列结构主要是为了匹配双方速度不一致，也为了decouple双方。比如典型生产者消费者问题

如果一个master 管理的node非常多 通过ListAndWatch 会对master的性能有影响吧

    这就是kubernetes 当前规模是5000的原因