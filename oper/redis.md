## **五种数据类型**

1. **String**
   get key 
   set key value ex 30
   ttl key
   incr key 
   decrby key  
   del key
2. **List**
   lpush key value 
   rpush key value 
   lpop key
   rpop key
   lrange key 0 -1 范围内的数据
   lindex key 3 获取指定index
3. **Set**
   sadd key value
   scard key 所有数
   smembers key 列出所有值
   sismember key value 判断是否包含
4. **Hash**
   hset key field value
   hget key field
   hgetall key
   hdel key field
5. **Zset**

**为什么要持久化**
防止宕机后的数据保存

## **两种持久化模式**

1. **RDB**

   数据快照的模式，可手动（save或bgsave，其中bgsave是主线程fork一个子线程进行备份，只在fork的时候堵塞）或自动（conf里配置 save m n m秒内操作n次就出发备份 或 主从复制时触发）
   核心是copy on write （写时复制）：当main主线程进行fork子线程复制时，如果main进行读操作，则不影响，如果main执行写操作，那么会会进行写时的备份，然后再由bgsave的子线程写入RDB文件。
   快照期间宕机，则返回上一次的RDB文件

2.  **AOF**
采用的是先进行内存写，再后写日志，这样的好处是不会把错误的命令也记录下来，但是坏处是会内容损失
命令追加：当AOF持久化功能打开了，服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器的 aof_buf 缓冲区。
文件写入和同步：关于何时将 aof_buf 缓冲区的内容写入AOF文件中，Redis提供了三种写回策略：
    1.always:同步写回
    2.everysec:每秒写回
    3.no:操作系统控制的写回  
为避免AOF文件越来越大，存在AOF重写瘦身，是主进程fork出的子线程进行重写，由文件大小和差值的计算方式触发AOF
重写过程总结为：“一个拷贝，两处日志”。在fork出子进程时的拷贝，以及在重写时，如果有新数据写入，主线程就会将命令记录到两个aof日志内存缓冲区中。如果AOF写回策略配置的是always，则直接将命令写回旧的日志文件，并且保存一份命令至AOF重写缓冲区，当完成重写后，通知主线程，主线程会将AOF重写缓冲中的命令追加到新的日志文件后面，最后切换文件
3.  **RDB和AOF 混合模式**
内存快照以一定的频率执行，在两次快照之间，使用 AOF 日志记录这期间的所有命令操作。



## redis事务
multi  开启事务后
所有命令入队
discard 取消事务
exec 执行事务
只会因为错误的语法而失败，不会回滚事务
可以在multi前watch key （乐观锁机制）如果在事务执行前有变化就会取消事务

## redis主从复制

主可读写，从可读，只能主复制到从 保证高可用

新版本re老版本sl

```shell
replicaof ip 端口	
```

```shell
slaveof ip 端口
```

![image-20240409153654283](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240409153654283.png)

增量同步

repl_backlog_buffer：是为了从库断开之后，如何找到主从差异数据而设计的环形缓冲区，从而避免全量复制带来的性能开销，如果环状被覆盖那么就是全量（使用RDB而不用AOF：RDB二进制快，文件小）

![image-20240409153829383](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240409153829383.png)



## redis哨兵

- **监控（Monitoring）**：哨兵会不断地检查主节点和从节点是否运作正常。

- **自动故障转移（Automatic failover）**：当主节点不能正常工作时，哨兵会开始自动故障转移操作，它会将失效主节点的其中一个从节点升级为新的主节点，并让其他从节点改为复制新的主节点。

- **配置提供者（Configuration provider）**：客户端在初始化时，通过连接哨兵来获得当前Redis服务的主节点地址。

- **通知（Notification）**：哨兵可以将故障转移的结果发送给客户端。

  ![image-20240410094802592](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240410094802592.png)

![image-20240410094827716](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240410094827716.png)

### 选举机制

就是一个Raft选举算法： **选举的票数大于等于num(sentinels)/2+1时，将成为领导者，如果没有超过，继续选举** （哨兵总数的一半以上）



## 缓存问题

**透击崩**

## 缓存穿透

- **问题来源**

缓存穿透是指**缓存和数据库中都没有的数据**，而用户不断发起请求。由于缓存是不命中时被动写的，并且出于容错考虑，如果从存储层查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到存储层去查询，失去了缓存的意义。

在流量大时，可能DB就挂掉了，要是有人利用不存在的key频繁攻击我们的应用，这就是漏洞。

如发起为id为“-1”的数据或id为特别大不存在的数据。这时的用户很可能是攻击者，攻击会导致数据库压力过大。

- **解决方案**

1. 接口层增加校验，如用户鉴权校验，id做基础校验，id<=0的直接拦截；
2. 从缓存取不到的数据，在数据库中也没有取到，这时也可以将key-value对写为key-null，缓存有效时间可以设置短点，如30秒（设置太长会导致正常情况也没法使用）。这样可以防止攻击用户反复用同一个id暴力攻击
3. 布隆过滤器。bloomfilter就类似于一个hash set，用于快速判某个元素是否存在于集合中，其典型的应用场景就是快速判断一个key是否存在于某容器，不存在就直接返回。布隆过滤器的关键就在于hash算法和容器大小，

---

## 缓存击穿

- **问题来源**

缓存击穿是指**缓存中没有但数据库中有的数据**（一般是缓存时间到期），这时由于并发用户特别多，同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大，造成过大压力。

- **解决方案**

1、设置热点数据永远不过期。

2、接口限流与熔断，降级。重要的接口一定要做好限流策略，防止用户恶意刷接口，同时要降级准备，当接口中的某些 服务 不可用时候，进行熔断，失败快速返回机制。

3、加互斥锁

---

## 缓存雪崩

- **问题来源**

缓存雪崩是指缓存中**数据大批量到过期时间，而查询数据量巨大，引起数据库压力过大甚至down机**。和缓存击穿不同的是，缓存击穿指并发查同一条数据，缓存雪崩是不同数据都过期了，很多数据都查不到从而查数据库。

- **解决方案**

1. 缓存数据的过期时间设置随机，防止同一时间大量数据过期现象发生。
2. 如果缓存数据库是分布式部署，将热点数据均匀分布在不同的缓存数据库中。
3. 设置热点数据永远不过期。

------

缓存一致性

采用先更新数据库，再删除缓存的方式，一个更新进程先更新DB后，一个查询进程先拿了cache的旧数据，但是之后马上就被更新进程失效，放入新数据。

如果是先删除缓存，在更新数据，一个更新进程先删除cache后，一个查询进程拿不到数据，去DB拿了未更新的数据，并放入缓存，会导致缓存一直是脏数据。
