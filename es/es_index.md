# es_index


## 基本概念

### 文档

ES是面向文档的，文档是所有可搜索数据的最小单位

文档会被序列化成json格式

    格式灵灵活，不需要预先定义格式
    支持数组，支持嵌套
    字段类型包括：
      字符串
      数值
      布尔
      日期
      二进制
      范围类型
    字段类型可以指定，或者通过es自动推算
  
每个文档都有一个UniqueID

    可以自己指定
    也可以由ES自动生成

元数据

    _index      文档所属索引名
    _type       文档所属类型名
    _id         文档唯一id
    _source     文档原始json数据
    _version    文档版本信息
    _score      相关性打分

## 分布式
----
水平库容

高可用

### 分布式架构
----
不同的集群通过不同的名字来区分，默认“elasticsearch”，可以通过配置文件修改，或者在命令行中-E cluster.name=<your_name>进行设定

### 节点
----

每个节点都有名字，通过配置文件配置，或者-E node.name=node1指定

每个节点都会分配一个UID，保存在data目录下

开发环境中一个节点可以承担多种角色

生产环境中应该设置单一角色的节点

#### Master-eligible node和master node
----
每个节点启动时默认是Master-eligible node

    设置node.master:false可禁止

Master-eligible node可参加选主流程成为Master节点

第一个节点启动时会将自己选举成为Master节点

每个节点都会保存集群状态，但是只有Master几点才能修改集群的状态信息

    集群状态Cluster State 维护了一个集群中必要的信息
        所有节点信息
        所有的索引及其相关的Mapping和Setting信息
        分片路由信息

    如果任意节点都能修改信息会导致数据的不一致性

#### Data Node

保存数据的节点

#### Coordinating Node

负责接收client的请求，将请求分发到合适的节点，最终把结果汇集到一起

每个节点都默认起到Coordinating Node的职责

####  Hot & Warm Node

冷热节点，降低集群部署成本

#### Machine Learning Node

负责跑机器学习的Job 

### 分片

Primary shard 主分片

    用以解决数据水平扩展的问题，通过主分片可以将数据分布到集群内的所有节点上

    一个分片是一个运行的lucene实例

    主分片数在索引创建时指定，后续不允许修改，除非Reindex

Replica shard 副分片

    用以解决高可用的问题，是主分片的拷贝

    副本分片数可以动态调整

    增加副本数，可以在一定程度上提高服务的可用性（读取的吞吐）