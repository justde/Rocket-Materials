# Redis基础

## 基本知识

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

## 实际问题

### 缓存雪崩

- 什么是缓存雪崩
  - 当某一时刻发生大规模的缓存失效情况，比如缓存服务宕机，大量请求直接打在DB，DB撑不住这么大的访问量，就会挂掉
- 解决办法
  - 事前：使用缓存集群，保证缓存服务的高可用
    - 使用主从+哨兵，Redis cluster避免Redis全盘崩溃的情况
    - 如果是key集体失效的情况，可以将缓存时下时间分散开，比如在原有失效时间上加上随机值，避免集体试下
  - 事中：ehcache本地缓存+hystrix限流+降级，减少DB压力
    - 用户发请求，先查询本地ecache，如果没有查找到再查Redis，如果ehcache和redis都没有，再查数据库，将数据库结果写入ecache和redis
    - 使用hystrix进行限流和降级，比如1s来了5000个请求，设置只能由2000个请求通过该组件，剩下的3000请求会走限流或降级逻辑
    - 然后调用开发的降级组件，比如设置默认值之类的，保护DB不被请求打死
  - 事后：通过Redis持久化，尽快恢复缓存集群

### 缓存击穿

- 什么是缓存击穿

  - 某个热点key失效，此时对于此key有大量的请求，就会导致请求都打到数据库上。原本应该访问缓存的请求都转移到DB，对DB的CPU和内存造成很大压力，严重会导致数据库宕机。从而形成一系列连锁反应，造成系统崩溃
  - 以一个key为目标击穿了缓存

- 解决办法

  1. 加锁或者队列保证不会有大量线程对数据库一次性同时读写，从而避免失效时大量请求落到存储。

     但是此方法只是减轻了数据库的压力，没有提高系统吞吐，在高并发下，会导致用户等待超时，治标不治本。

  2. 使用过期标记，缓存过期前异步更新缓存

     1. 缓存标记：记录缓存数据是否过期，如果过期会通知另外的线程在后台去更新实际key的缓存
     2. 缓存数据：过期时间比缓存标记的时间延长一倍，这样当缓存标记key过期后，实际的缓存还在，能把旧数据返回给调用端，另外再起一个线程在后台完成更新
     3. 伪代码	![](../assets/%E7%BC%93%E5%AD%98%E5%87%BB%E7%A9%BF.png)

### 缓存穿透

- 什么是缓存穿透

  用户请求数据库中不存在的数据，缓存中也不会有，这样导致用户查询的时候缓存找不到，每次都要再去数据库查一次，相当于两次无用的查询

- 解决办法

  - 缓存控制，设置一个短的过期时间，避免查询不到引发缓存穿透
  - bloomFilter
    - 将所有可能存在的数据hash到一个足够大的bitmap中，一个一定不存在的数据会被这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。
    - bloomFilter类似一个hashset用来判断某个元素是否存在于某个集合中
    - 这种方案就是在缓存之前加一层bloomFilter，在查询的时候先去bloomFilter查询key是否存在，如果不存在就直接返回，存在则走正常流程