# 面试题



软件工程

规范化

docker

容器镜像退出后清除文件是如何实现的

chroot命令切换root file system，重新挂载了容器进程的根目录

rootfs

演化出Mount Namespace

UnionFS：将多个不同位置的目录联合挂载

docker镜像的rootfs由多个层组成（每层都是一个增量rootfs）

使用镜像时，Docker会把这些增量rootfs挂载在一个统一的挂载点上/var/lib/docker/aufs/mnt/

镜像的层都放置在/var/lib/docker/aufs/diff目录下，然后被联合挂载在/var/lib/docker/aufs/mnt

启动一个容器时，docker把镜像的rootfs挂载到到/var/lib/docker/aufs/mnt/



k8s

负载均衡策略

服务性能优化：如何让服务性能跑满

请求先到机器，首先看机器的监控，看瓶颈在哪，io，cpu，memory

等，若果机器性能都正常，那么就进入到业务链路，中间设计到那些服务，是本身的还是外部依赖，如果是本身可控
的，如业务算法逻辑，存储优化，网络优化，如果是外部的，需要根据具体需求来处理，打个比方，丢掉部分实施
性，预先加载数据

管理问题：

1号定方案，14号发现问题20号发版，如何优化



c++虚表 多态如何实现，原理



redis 优化，

使用rdb分析





巨人一面



TCP粘包如何处理，解析字段的方法和不解析字段的方法

protobuffer的原理

项目中使用MQ的场景

GO和C++的区别

MAP和sync.MAP的区别

mysql 表锁和行锁  innodb中才有表锁和行锁，select 默认表锁，，innodb中的行锁是加在索引上的，索引当使用
索引时才会加行锁，

表锁：开销小，加锁快；不会出现死锁；锁定力度大，发生锁冲突概率高，并发度最低

行锁：开销大，加锁慢；会出现死锁；锁定粒度小，发生锁冲突的概率低，并发度高



小讯鸽



http和grpc的区别

mysql写入性能

influxdb实现原理，为何性能好

     时序db。 顺序io。page cache

redis集群

set和zset内部性能区别

redis监控打点分析（monitor指令）

kafka流计算

微服务治理（平滑重启之类）

算法

