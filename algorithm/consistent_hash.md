# 一致性hash

用于解决数据分布问题，在不存储全局字典的情况下如何确认某个数据在集群中的哪个节点上。

## Mod-N hashing

对Key的hash结果取余来确认节点编号的解决方案, N=节点总数

```
server := serverList[hash(key) % N]
```

###  选择hash函数

原则：尽量快

#### 加密hash，分布均匀但是开销大
* SHA-1
* MD5

#### 非加密hash，计算开销小

* [MurmurHash](https://en.wikipedia.org/wiki/MurmurHash)

以下三种比MurmurHash稍好一些(why?)
* [xxHash](https://github.com/cespare/xxhash)
* [MetroHash](https://github.com/dgryski/go-metro)
* [SipHash1-3](https://github.com/dgryski/go-sip13)

### 优点

* 易于理解
* 计算开销小（取余的开销远小于hash计算，尤其在N=2的情况下）
### 缺点

* 节点总数发生变化时，基本上所有的key都会被映射到错误的节点

## 期待的哈希算法

* 当节点发生变化，只有1/N的key需要迁移
```
    新增节点
        key永远只从旧节点移动到新节点，旧节点之间永远不会出现key的迁移情况

    移除节点
        被移除的节点key应该平均迁移到其他节点
```
    
* 如非必要不要移动key

### 论文

1997年
* [Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web](https://www.akamai.com/es/es/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf)

2007年
* [Ketama memcached client](https://www.last.fm/user/RJ/journal/2007/04/10/rz_libketama_-_a_consistent_hashing_algo_for_memcache_clients)

* [Dynamo: Amazon’s Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)

### Ring-based Consistent Hashing

* 在实践中，每个节点都可以在哈希环上出现多次。多出来的节点被称为虚节点（"virtual nodes"/"vnodes"）。这种做法可以减少节点负载差异，而且借助虚节点可以为不同的节点分配不同数量的key
* 简单易懂

然而ring hash算法的节点负载分布依旧不够均匀。在每个节点创建100个vnodes的情况下，负载的标准偏差约为10%。存储桶大小的99％置信区间为平均负载的0.76至1.28。这种可变性导致预估容量变得棘手。将每个节点的vnodes增加到1000个可以将标准偏差降低到3.2%左右，且将99%置信区间缩小到0.92-1.09之间。但是这种做法带来了更多的内存和搜索节点的开销，1000个vnodes大概需要4MB，搜索算法（二分查找）时间复杂度为O(log n)，而且在没有缓存竞争的时候也无法命中高速缓存


### Jump Hash

2014年Google发布论文[A Fast, Minimal Memory, Consistent Hash Algorithm](https://arxiv.org/abs/1406.2294)

解决Ring Hash的两个缺点
* 没有内存开销
* bucket标准偏差为0.000000764%，99％置信区间为0.99999998至1.00000002

优点
* 快，搜索算法时间复杂度O(ln n)
* 计算完全在寄存器中完成，不会有缓存未命中的开销








## 参考资料

[Consistent Hashing: Algorithmic Tradeoffs](https://medium.com/@dgryski/consistent-hashing-algorithmic-tradeoffs-ef6b8e2fcae8)