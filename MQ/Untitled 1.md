# Kafka核心题目

### kafka有什么特点

​		kafka是分布式的发布-订阅消息系统

- 高吞吐、低延迟：每秒处理数十万消息，延迟最低几毫秒
- 可扩展性：kafka集群支持热扩展
- 持久化：消息被持久化到本地磁盘，并且支持数据备份防止数据丢失
- 容错性：允许集群中节点失败，若副本数量为n，则允许n-1个节点失败
- 高并发：支持数千个客户端同事读写

### kafka中的consumer group

- consumer group 是consumer的集合，一个partition只能由consumer group中的一个consumer消费，可以使consumer group中的consumer数量与partition数量一样，提高消费效率
- 消费者组是逻辑上的一个订阅者

### kafka分区的好处

- 实现负载均衡
- 对于消费者提高并发度，提升效率

### kafka如何判断一个节点存活

- 节点必须可以维护和zookeeper链接，zookeeper可以通过心跳机制检查每个节点的链接
- 如果节点是个follower，必须能及时的同步leader的写操作，延迟不能太久

### producer是否直接将数据发送给broker的leader

- producer直接将数据发送给broker的leader，不需要在多个节点进行分发。
- 所有的kafka节点都知道哪些节点是活动的，目标topic分区的leader在哪里，这样producer就可以直接发送给目的地