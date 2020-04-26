## Redis基础

### 什么是redis

redis全称remote dictionary server，是一个key-value类型的内存数据库。整个数据库通通加载在内存中进行操作，定期通过异步操作将数据flush到硬盘进行保存。redis支持多种数据结构，单个value最大限制为1GB

### redis与memcached相比的优势

- memcached所有值都是简单字符串，redis支持更多的数据类型
- redis速度更快
- redis可以持久化

### redis 为什么快

- 完全基于内存，绝大部分请求是纯粹的内存操作
- 采用大多数采用单线程，避免了不必要的上下文切换和竞争，不用考虑家所释放锁的操作
- 使用多路IO复用模型，非阻塞IO；多路就是多个网络请求，复用就是使用一个线程处理事件

### redis支持哪些数据类型

- String
- List
- HashMap
- Set
- Sorted-Set

### redis的过期策略

redis采用定期删除+惰性删除

- 定期删除
  - 每隔100MS随机抽取一些key来检查和删除
- 惰性删除
  - 查询key的时候判断key是否过期然后处理
- 没来得及删除，大量key堆积在内存，导致内存满了
  - 内存淘汰机制：1、报错 2、所有key参与淘汰  3、过期key淘汰
  - neoviction 内存不足新写入操作报错
  - allkey-lru    移除最近最少使用的key （least recently used）
  - allkey-random  随机移除某个key
  - volatile-lru   设置过期时间的key中移除最近最少使用的key
  - volatile-random  设置过期时间的key中随机移除
  - volatile-ttl    在设置过期时间的键中，将更早的过期时间的key移除

