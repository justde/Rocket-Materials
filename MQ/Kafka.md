# Kafka

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

### 消费这组内分区分配策略

- RoundRobin 按照组划分
  - 消息按照组内消费者的顺序轮询发送
    - message1->consumer1;
    - message2->consumer2;
    - message3->consumer1;
    - message4->consumer2;
  - 当有两个topic多个partition时，根据对象TopicAndPartition进行hash，然后进行轮询
  - 问题：部分消费者消费时间长会阻塞  
- Range（默认按照topic划分）
  - 一个topic中有7个partition，三个consumer
  - 第一个consumer负责partition0-2
  - 第二个consumer负责partition3-4
  - 第三个consumer负责partition5-6
  - 问题：分布不均匀总是第一个处理的最多，当订阅的主题越多，处理的工作量差别越大

### kafka消息采用pull还是push

- kafka中producer将消息推送到broker，consumer从broker中拉取
- 部分消息系统采用push，push由broker决定了推送的速率，如果broker推送速度大于consumer的速度，consumer没办法及时处理
- pull可以根据consumer的消费能力决定获取消息
- pull的缺点：如果没有可供消费的消息，导致consumer不断循环轮询，直到新消息到达。为了避免这一情况有个参数可以让consumer阻塞直到有新消息到达

### kafka存储在硬盘上的消息格式是什么样的

- 消息由固定长度的头部和变长的消息组成，头部包含了版本号和CRC32校验码
- 消息长度(1+4+n)
  - 版本号：1 byte
  - CRC校验码：4 byte
  - 具体消息：n byte

### kafka高效文件存储设计的特点

- kafka把topic中一个partition大文件段分为多个小文件段，通过多个小文件段，容易定期清理或者删除，减少磁盘占用
- 文件只追加写入，顺序读写，避免了缓慢的io操作
- 一个partition分为多个segment，每个segment目录下有index和log文件，index文件以最小的offset命名。
- 查询时根据二分查找定位index文件，扫描index文件找到数据在log文件的具体位置

### kafka中controller的作用

kafka中有一个broker被选举为controller，负责集群broker的上下线，所有topic的分区副本分配和leader选举。controller的工作依赖于zookeeper

### kafka中有哪些地方需要选举

- controller
  - 需要依赖zookeeper抢占资源
- leader
  - 从ISR中选取，主要取决同步时间

### kafka的有序性

kafka会将partition以消息日志的方式落盘存储，通过顺序IO和缓存将消息写盘提高速度

### kafka如何保证有序性

kafka只能保证partition内有序，每个partition只能由consumer group中的一个consumer消费，因此可以保证这一个partition是有序的

### kafka如何保证数据可靠

- 生产者
  - 开启acks=all，partition的leader会等待所有的follower都同步消息后，才认为本次消息接收成功，返回给生产者，即所有副本都同步完成消息，才算完成
- broker
  - 设置多个分区副本，这样partition的leader故障，分区的副本能够选举成为leader，维持功能正常
  - 设置分区至少与一个副本保持感知
  - 生产者设置重试次数为max，每次失败后都重试
- 消费者
  - 关闭自动提交offset，保证消费完成后才会给partition发送offset

### kafka的应答机制

- 根据对可靠性要求的差异，kafka提供了三种可靠级别，通过配置acks参数
- acks=0
  - producer不等待broker的ack，broker接收到消息就返回，当broker故障时会丢失数据
- acks=1
  - producer等待broker的ack，partition的leader落盘成功即可返回ack，如果follower同步之前leader故障，那么将会丢失数据
- acks=-1/all
  - producer等待broker的ack，partition的leader和ISR中的follower同步完成

### ISR/OSR/AR是什么

- ISR：in-sync-Replicas副本同步队列
- OSR：out-sync-replicas 脱离副本同步队列
- AR：assigned Replicas 所有副本

kafka采用的ack=-1 全部集合同步完成返回ack，如果某一个follower发生故障不能与leader同步，那么leader一直等下去，生产者也会认为没有同步结束

ISR由leader维护，follower从leader同步数据有延迟超过相应的阈值，会把follower剔除出ISR，放入OSR列表，新加入的follower也会先放在OSR

### LEO/HW/LSO/LW是什么

- LEO  log end offset，代表当前partition中最后一个offset
- HW   hight water，代表位置信息，取partition对应的ISR中最小的LEO作为HW，consumer最多能消费到HW所在位置的上一条信息
- 高水位存在的意义：假如leader的offset为5，其他follower为3，如果leader宕机，follower选举称为leader，当consumer来取offset为5的消息，follower没有，会报错，因此需要知道一个高水位，即所有partition中offset最小的值

### kafka中的分区器、序列化器、拦截器顺序

顺序：拦截器->序列化器->分区器

### kafka创建topic如何将分区放置在不同的broker

-  第一个分区(编号为0)的第一个副本放置位置是从brokerLIst中随机选取
- 后续分区的第一个副本放置位置是相对于第一个分区以此向后移
- 剩下的副本相对于第一个副本位置是由nextReplicaShift方法决定的，随机生成一个数

### 通过kafka-topics.sh创建或删除一个topic，kafka背后执行什么逻辑

1. 会在zookeeper的beokers/topics节点创建一个新的topic节点
2. 触发controller监听程序
3. kafka controller负责topic的创建工作并更新metadata cache