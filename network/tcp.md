# TCP

## Socket 5元组
1. 源IP
2. 源端口
3. 目的IP
4. 目的端口
5. 类型：TCP:UDP

例：

```
访问百度的时候，socket五元组可能是：

    [180.172.35.150:45678, tcp, 180.97.33.108:80]
```

## TIME_WAIT 和 CLOSE_WAIT

关闭socket需要通过四次挥手来完成
* 主动关闭连接的一方，调用close()；协议层发送FIN包
* 被动关闭的一方收到FIN包后，协议层回复ACK；然后被动关闭的一方，进入CLOSE_WAIT状态，主动关闭的一方等待对方关闭，则进入FIN_WAIT_2状态；此时，主动关闭的一方 等待 被动关闭一方的应用程序，调用close操作
* 被动关闭的一方在完成所有数据发送后，调用close()操作；此时，协议层发送FIN包给主动关闭的一方，等待对方的ACK，被动关闭的一方进入LAST_ACK状态；
* 主动关闭的一方收到FIN包，协议层回复ACK；此时，主动关闭连接的一方，进入TIME_WAIT状态；而被动关闭的一方，进入CLOSED状态
* 等待2MSL时间，主动关闭的一方，结束TIME_WAIT，进入CLOSED状态

结论
1. 主动关闭socket一方会进入TIME_WAIT状态
2. 被动关闭连接的一方会进入CLOSE_WAIT的中间状态，需要应用程序调用close
3. TIME_WAIT默认等待2MSL后才能进去CLOSED状态
4. 一个连接在进入CLOSED状态之前无法被重用

### TIME_WAIT 作用

TIME_WAIT是为了解决网络丢包和网络不稳定带来的其他问题：

1. 防止前一个连接上延迟的数据包或丢失重传的数据包被复用的连接接收
2. 确保连接方能在时间范围内关闭自己的连接【也是防止丢包】
```
* 主动关闭方关闭了连接，发送了FIN；
* 被动关闭方回复ACK同时也执行关闭动作，发送FIN包；此时，被动关闭的一方进入LAST_ACK状态
* 主动关闭的一方回去了ACK，主动关闭一方进入TIME_WAIT状态；
* 但是最后的ACK丢失，被动关闭的一方还继续停留在LAST_ACK状态
* 此时，如果没有TIME_WAIT的存在，或者说，停留在TIME_WAIT上的时间很短，则主动关闭的一方很快就进入了CLOSED状态，也即是说，如果此时新建一个连接，源随机端口如果被复用，在connect发送SYN包后，由于被动方仍认为这条连接【五元组】还在等待ACK，但是却收到了SYN，则被动方会回复RST
* 造成主动创建连接的一方，由于收到了RST，则连接无法成功
```

### 为什么TIME_WAIT状态会持续2MSL（2倍的max segment lifetime）

RFC 793中定义了TIME_WAIT状态持续2MSL。 

用于确保最后丢失了ACK时，被动关闭的一方再次重发FIN并等待回复ACK一来一去的两个来回。

内核中写死了MSL时间为30秒。（RFC中建议的MSL时间是2分钟）

### TIME_WAIT过多带来的问题

通过 `ss -tan state time-wait | wc -l `查看TIME_WAIT数量

1. 占用内存
    * 内核里使用一个hash table保存所有连接
    * 还有一个hash table用来保存所有的bound ports
2. 消耗CPU
    * 寻找随机端口时需要遍历bound ports

一万条TIME_WAIT大概消耗1M左右的内存

### TIME_WAIT调优

```
打开 sysctl.conf 文件，修改以下几个参数：

net.ipv4.tcp_tw_recycle = 1

net.ipv4.tcp_tw_reuse = 1

net.ipv4.tcp_timestamps = 1
```

1. net.ipv4.tcp_timestamps
    * RFC 1323 在TCP Reliability一节中引入了timestamp的TCP option——两个4字节的时间戳字段，第一个4字节字段用来保存发送该数据包的时间，第二个4字节字段用来保存最近一次接受到对方数据的时间。
    * tcp_tw_reuse和tcp_tw_recycle依赖该字段

2. net.ipv4.tcp_tw_reuse
    * 复用TIME_WAIT状态的连接
    * 只有主动关闭连接的一方再次向对方发起连接请求的时候才能复用TIME_WAIT状态的连接
    * 适用于某一方不断地通过短连接连接其他服务器的场景。总是自己先关闭连接，关闭后又不断重新连接对方
    * 复用连接后这条连接的时间被更新为当前的时间，内核根据收到数据中的timestamp是否小于当前连接中的timestamp来判断是否是延迟数据
    * 该配置需要连接双方同时支持net.ipv4.tcp_timestamps，且这个配置仅影响outbound连接（作为客户端的角色）

3. net.ipv4.tcp_tw_recycle
    * 开启后内核会快速回收TIME_WAIT状态的socket连接
    * 不再是2MSL，而是1RTO（retransmission timeout，数据包重传的timeout时间，），RTO根据RTT动态计算得出，远小于2MSL
    * 开启该配置后内核会记录TIME_WAIT socket的统计数据，包括最近一次收到数据包时间。晚于该时间的数据包会被丢掉
    * 该配置需要连接双方同时支持net.ipv4.tcp_timestamps，且这个配置仅影响inbound连接，作为服务端的角色主动关闭连接时，快速回收状态处于TIME_WAIT状态的socket




## 资料
https://mp.weixin.qq.com/s/8WmASie_DjDDMQRdQi1FDg
