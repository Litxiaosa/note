**本文基于 Redis 6.0 版本**

### **一：Redis 常见数据结构以及使用场景分析**

  **参考:** [Redis 命令参考](http://redisdoc.com/index.html)

###### **1：String**

​		**基本介绍**：`String` 数据结构是简单的 `key-value` 类型, 一个 `Redis` 中字符串 `value`  最多可以是512M。

​		**应用场景** ：一般常用在需要计数的场景，比如用户的访问次数、热点文章的点赞、转发数量等等。

###### **2：List**

​		**基本介绍：** `Redis`的链表是**双向链表**，`left` `right`都可以插入添加。

​		**应用场景：**关注列表、粉丝列表、消息队列等。

###### **3: Set**

​		**基本介绍：** `Redis`中的 `set` 类型是一种无序集合，不能重复的集合。并且 `set` 提供了判断某个成员是否在一个 `set` 集合内的重要							接口，这个也是 `list` 所不能提供的。可以基于 `set` 轻易实现交集、并集、差集的操作。

​		**应用场景：** 需要存放的数据不能重复以及需要获取多个数据源交集和并集等场景。比如：共同关注、共同粉丝、共同喜好等功能。

###### **4：zset**

​		**基本介绍：** `Redis` 的 `zset` 和 `set` 一样也是`String`类型元素的集合,且不允许重复的成员。 不同的是他内部实现了一个叫跳跃表的							 数据结构每个元素都会关联一个`Double`类型的权重分数。 redis正是通过分数来为集合中的成员进行从小到大的排序							`zset`的成员是唯一的,但分数(`score`)却可以重复。

​		**应用场景：** 需要对数据根据某个权重进行排序的场景。比如在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物 排行							榜弹幕消息（可以理解为按消息维度的消息排行榜）等信息。

###### **5：Hash**

​		**基本介绍：**`hash` 是一个 `String` 类型的 `field` 和 `value` 的映射表，**特别适合用于存储对象**。

​		**应用场景：**系统中对象数据的存储。



### **二：过期策略**

- **定期删除:** 指的是 `Redis` 默认是每隔 100ms 就随机抽取一些设置了过期时间的 key，检查其是否过期，如果过期就删除。

- **惰性删除:**  获取 key 的时候，如果此时 key 已经过期，就删除，不会返回任何东西。

所以，`Redis`会结合这两种过期策略，因为**定期删除**很可能会导致很多过期的 key 没有被删除，所以`Redis`在获取 key 的时候，会检查一下，这个 key 有没有设置过期时间并且是否过期，如果是，就把该 key 删除，不会返回任何东西。

但是实际上这还是有问题的，如果定期删除漏掉了很多过期 key，然后你也没及时去查，也就没走惰性删除，此时会怎么样？如果大量过期 key 堆积在内存里，导致 `Redis` 内存块耗尽了，咋整？

答案是：**走内存淘汰机制**



### **三：内存淘汰机制**

 `Redis` 的内存淘汰机制有一下几种：

- **volatile-lru：**当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的 key，LRU 算法。
- **allkeys-lru：**当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key，LRU 算法。
- **volatile-lfu：**当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除使用频率最少的 key，LFU 算法。
- **allkeys-lfu：**当内存不足以容纳新写入数据时，在键空间中，移除使用频率最少的 key，LFU 算法。
- **volatile-random：**当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个 key。
- **allkeys-random：**当内存不足以容纳新写入数据时，在键空间中，随机移除某个 key。
- **volatile-ttl：**当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的 key 优先移除。

- **noeviction：**当内存不足以容纳新写入数据时，新写入操作会报错。

**内存淘汰策略默认是注释掉的，可根据实际需求启用相应的策略，LRU 算法其实并不是很精确，所以出现了 LFU 算法**。



### **四：持久化**

###### **方式一：RDB**

RDB 俗称快照，当满足条件时将生成数据集的时间点快照。此条件是可以由用户配置`Redis`实例来控制，RDB 是 `fork`函数产生一个子进程，简单的理解就是基于当前线程复制了一个进程，主进程和子进程会共享内存里面的代码块和数据段。所以，RDB 持久化完全可以交给子进程处理，主进程则继续处理客户端请求。

默认的`redis.conf`文件的配置：

``` shell
# save ""
# 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
# 这里表示900秒（15分钟）内有1个更改，300秒（5分钟）内有10个更改以及60秒内有10000个更改
# 如果想禁用RDB持久化的策略，只要不设置任何save指令，或者给save传入一个空字符串参数也可以
save 900 1
save 300 10
save 60 10000
```

我们还可以设置 RDB 文件的名字

``` shell
# The filename where to dump the DB
# rdb文件的名字。
dbfilename dump.rdb
```

**RDB的优缺点：**

- RDB 会生成多个数据文件，每个数据文件都代表了某一个时刻中 `Redis` 的数据，这种多个数据文件的方式，非常适合做冷备
- RDB 对 Redis 对外提供的读写服务，影响非常小，可以让 `Redis` 保持高性能，因为 `Redis` 主进程只需要 fork 一个子进程，让子进程执行磁盘 IO 操作来进行 RDB 持久化即可。
- 相对于 AOF 持久化机制来说，直接基于 RDB 数据文件来重启和恢复 Redis 进程，更加快速。
- 如果想要在 Redis 故障时，尽可能少的丢失数据，那么 RDB 没有 AOF 好。一般来说，RDB 数据快照文件，都是每隔 5 分钟，或者更长时间生成一次，这个时候就得接受一旦 Redis 进程宕机，那么会丢失最近 5 分钟的数据
- RDB 每次在 fork 子进程来执行 RDB 快照数据文件生成的时候，如果数据文件特别大，可能会导致对客户端提供的服务暂停数毫秒，或者甚至数秒。

我们知道 RDB 的持久方案我们可以在`redis.conf`文件里配置，那如果我这一秒刚发生了 RDB 持久化，接下来的时间在下一个 RDB持久化发生前，我 `Redis` 实例宕机了，那么在这期间的数据就没有了，这个时间可能会很长，导致丢失的数据就会很多，这是不能被允许的情况，所以我们还有一种持久化方案：**AOF**



###### **方式二：AOF**

`AOF:  Append Only File` - 仅追加文件,  它的工作方式非常简单：执行修改内存中数据集的写操作时，都会记录该操作。假设 AOF日志记录了自 `Redis` 实例创建以来所有的修改性指令序列，那么就可以通过对一个空的 `Redis` 实例顺序执行所有的指令，也就是「重放」，来恢复 `Redis` 当前实例的内存数据结构的状态。

AOF 默认是关闭的，`redis.conf`文件的配置.

``` shell
# 是否启用aof持久化方式 
appendonly no
```

我们可以配置`Redis` 多久才将数据 `fsync` 到磁盘一次，默认的`redis.conf`文件的配置：

``` shell
# 指定更新日志条件，共有3个可选值： 
# no：表示等操作系统进行数据缓存同步到磁盘（快，持久化没保证） 
# always：同步持久化，每次发生数据变更时，立即记录到磁盘（慢，安全） 
# everysec：表示每秒同步一次（默认值,很快，但可能会丢失一秒以内的数据）
# appendfsync always
appendfsync everysec
# appendfsync no
```



**AOF重写:**  `Redis`在长期运行过程中，AOF 的日志肯定会越来越长，如果此时`Redis`实例宕机了，那么重放整个 AOF 日志将会非常耗时，将会导致`Redis`长时间如法对外提供服务，所以需要对 AOF 日志进行 **"瘦身"**。

`Redis`提供了`bgrewriteaof`指令用于对 AOF 日志进行瘦身，其原理就是：开辟一个子进程对内存进行遍历转换成一系列`Redis`的操作指令，序列化到一个新的 AOF 日志文件中，序列化完毕后再将操作期间发生的增量 AOF日志追加到这个新的 AOF 日志文件中，追加完毕后就立即替代旧的 AOF 日志文件了，瘦身工作就完成了，那么什么时候发生重写呢？

默认的`redis.conf`文件的配置：

``` shell
# 当AOF文件增长到一定大小的时候Redis能够调用 BGREWRITEAOF 对日志文件进行重写 。当AOF文件大小的增长率大于该配置项时自动开启重写。
auto-aof-rewrite-percentage 100
# 当AOF文件增长到一定大小的时候Redis能够调用 BGREWRITEAOF 对日志文件进行重写 。当AOF文件大小大于该配置项时自动开启重写
auto-aof-rewrite-min-size 64mb
```

同样，我们也可以设置该文件的名字

``` shell
# 指定更新日志（aof）文件名，默认为appendonly.aof
appendfilename "appendonly.aof"
```



**AOF 的优缺点：**

- AOF 可以更好的保护数据不丢失，一般 AOF 会每隔 1 秒，通过一个后台线程执行一次 `fsync` 操作，最多丢失 1 秒钟的数据。
- AOF 日志文件以 `append-only` 模式写入，所以没有任何磁盘寻址的开销，写入性能非常高，而且文件不容易破损，即使文件尾部破损，也很容易修复。
- AOF 日志文件即使过大的时候，出现后台重写操作，也不会影响客户端的读写。因为在 `rewrite` log 的时候，会对其中的指令进行压缩，创建出一份需要恢复数据的最小日志出来。在创建新日志文件的时候，老的日志文件还是照常写入。当新的 merge 后的日志文件 ready 的时候，再交换新老日志文件即可。
- AOF 日志文件的命令通过可读较强的方式进行记录，这个特性非常适合做灾难性的误删除的紧急恢复。比如某人不小心用 `flushall` 命令清空了所有数据，只要这个时候后台 `rewrite` 还没有发生，那么就可以立即拷贝 AOF 文件，将最后一条 `flushall` 命令给删了，然后再将该 `AOF` 文件放回去，就可以通过恢复机制，自动恢复所有数据。
- 对于同一份数据来说，AOF 日志文件通常比 RDB 数据快照文件更大。
- AOF 开启后，支持的写 QPS 会比 RDB 支持的写 QPS 低，因为 AOF 一般会配置成每秒 `fsync` 一次日志文件，当然，每秒一次 `fsync` ，性能也还是很高的。（如果实时写入，那么 QPS 会大降，Redis 性能会大大降低）。




> 如果 AOF 和 RDB 同时开启了，那么该听谁的？   答：无脑听 AOF。



###### **混合持久化**

我们在 `redis.conf` 配置文件中可以配置开启混合持久化

``` shell
# 是否开启混合持久化
aof-use-rdb-preamble yes
```

混合持久化结合了RDB持久化 和 AOF 持久化的优点, 由于绝大部分都是RDB格式，加载速度快，同时结合AOF，增量的数据以AOF       	    方式保存了，数据更少的丢失。**redis 4.0** 之前的版本不支持混合持久化。




### **五：主从架构**

通过持久化功能，`Redis`保证了即使在服务器重启的情况下也不会损失（或少量损失）数据，因为持久化会把内存中数据保存到硬		 盘上，重启会从硬盘上加载数据。但是由于数据是存储在一台服务器上的，如果这台服务器出现硬盘故障等问题，也会导致数据丢   	     失。为了避免单点故障，通常的做法是将数据库复制多个副本以部署在不同的服务器上，这样即使有一台服务器出现故障，其他服		 务器依然可以继续提供服务。为此，`Redis` 提供了复制（`replication`）功能，可以实现当一台数据库中的数据更新后，自动将更		 新的数据同步到其他数据库上。

 **Redis replication 的几点说明**

- `Redis` 采用异步方式复制数据到 `slave` 节点`, slave node` 会周期性地确认自己每次复制的数据量。
- `slave node` 做复制的时候，不会阻塞 `master node` 的正常工作。
- `slave node` 主要用来进行横向扩容，做读写分离，扩容的 `slave node` 可以提高读的吞吐量

**注意：** 如果采用了主从架构，那么建议必须开启 `master node` 的持久化，不建议用 `slave node` 作为 `master node` 的数据热备，			因为那样的话如果你关掉 `master` 的持久化，可能在 `master` 宕机重启的时候数据是空的，然后可能一经过复制slave node 的数        			据也丢了。

###### **主从复制的原理**

当启动一个 `slave node `的时候，它会发送一个 `PSYNC` 命令给 `master node`。

如果这是` slave node ` 初次连接到 `master node`，那么会触发一次 `full resynchronization` 全量复制。此时 `master` 会启动		  	   一个后台线程，开始生成一份 `RDB` 快照文件，同时还会将从客户端 `client` 新收到的所有写命令缓存在内存中。 `RDB` 文件生			  	    成完毕后， `master` 会将这个 `RDB` 发送给 `slave`，`slave` 会先写入本地磁盘，然后再从本地磁盘加载到内存中，接着 `master` 会	       将内存中缓存的写命令发送到 `slave`，`slave` 也会同步这些数据。`slave node `如果跟 `slave node `有网络故障，断开了连接，会	       自动重连，连接之后 `master node` 仅会复制给 `slave` 部分缺少的数据。

<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200819121243454.png" alt="image-20200819121243454" style="zoom:60%;" />



###### **断点续传**

`Redis`支持断点续传，`master node` 会在内存中维护一个 `backlog`，`master` 和 `slave` 都会保存一个 `replica offset `还有一个 `master run id`，`offset` 就是保存在 `backlog` 中的。如果 `master` 和 `slave` 网络连接断掉了，`slave` 会让 `master`  从上次`replica offset` 开始继续复制，如果没有找到对应的 `offset`，那么就会执行一次 `resynchronization` 。



###### **无磁盘化复制**

`master` 在内存中直接创建 `RDB` ，然后发送给 `slave`，不会在自己本地落地磁盘了。

只需要在配置文件中开启 `repl-diskless-sync yes` 即可。

``` shell
# 主从数据复制是否使用无硬盘复制功能。
repl-diskless-sync no
# repl-diskless-sync-delay 5
repl-diskless-sync-delay 5
```



###### **Salve 的过期 key 处理**

`slave` 不会过期 key，只会等待 `master` 过期 key。如果 `master` 过期了一个 key，或者通过 `LRU` 淘汰了一个 key，那么会模拟一条 	 `del` 命令发送给 `slave`。



###### **全量复制**

- `master` 执行 `bgsave` ，在本地生成一份 `rdb` 快照文件。

- `master node` 将 `rdb` 快照文件发送给 `slave node`，如果 `rdb` 复制时间超过 60秒（`repl-timeout`），那么` slave node` 就会认为复制失败，可以适当调大这个参数(对于千兆网卡的机器，一般每秒传输 100MB，6G 文件，很可能超过 60s)。

- `master node `在生成 `rdb` 时，会将所有新的写命令缓存在内存中，在 `slave node` 保存了 `rdb` 之后，再将新的写命令复制给 `slave node`。

- 如果在复制期间，内存缓冲区持续消耗超过 64MB，或者一次性超过 256MB，那么停止复制，复制失败。

  ``` shell
  client-output-buffer-limit replica 256mb 64mb 60
  
  # 设置主库批量数据传输时间或者ping回复时间间隔，默认值是60秒 。
  # repl-timeout 60
  ```
  
- `slave node` 接收到 `RDB` 之后，清空自己的旧数据，然后重新加载 `RDB` 到自己的内存中，同时基于旧的数据版本对外提供服务。

- 如果` slave node` 开启了 `AOF`，那么会立即执行` BGREWRITEAOF`，重写 `AOF`。

  

###### **增量复制**

- 如果全量复制过程中，`master-slave` 网络连接断掉，那么 `slave` 重新连接 `master` 时，会触发增量复制。
- `master` 直接从自己的 `backlog` 中获取部分丢失的数据，发送给 `slave node`，默认 `backlog` 就是 1MB。
- `master` 就是根据 `slave` 发送的 `psync` 中的 `offset` 来从 `backlog` 中获取数据的。



###### **主从架构的配置**

> 一主一从为例

**主服务器**：修改`redis.conf` 配置文件：

``` shell
# 绑定Ip 指定可以连接本实例Redis的ip  如果注释（删掉）则任意IP都可以连接, 建议绑定本机内网 IP
bind 172.17.0.4

# 如果需要Redis在后台进程运行，就把该项的值设为yes
daemonize yes

# 代表的是持久化的储存目录，RDB 或者 AOF 持久化方案的文件会生成在当前启动目录，如果下一次在其他目录启动 redis，则该持久化文件不 可用。建议修改为固定目录。(改目录手动创建)
dir /data

# 设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过AUTH <password>命令提供密码，默认关闭
requirepass 123456
```



**从服务器：**修改`redis.conf` 配置文件：

``` shell
# 绑定Ip 指定可以连接本实例Redis的ip  如果注释（删掉）则任意IP都可以连接, 建议绑定本机内网 IP
bind 172.17.0.4

# 默认情况下，redis不是在后台模式运行的，如果需要在后台进程运行，把该项的值更改为yes，默认为no 
daemonize yes

# 代表的是持久化的储存目录，RDB 或者 AOF 持久化方案的文件会生成在当前启动目录，如果下一次在其他目录启动 redis，则该持久化文件不可用。建议修改为固定目录。(改目录手动创建)
dir /data

# 当master服务设置了密码保护时，slave服务连接master的密码
masterauth 123456

# 设置当本机为slave服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步
replicaof 172.17.0.4 6379
```



启动两台服务器，然后客户端连接，在 src 目录下输入:

``` shell
# 172.17.0.4 IP 地址，  6379：端口号，  123456：上边配置的密码
./redis-cli -h 172.17.0.4 -p 6379 -a 123456
```

我们用客户端连接上以后，在主服务器中 用 `info replication` 命令可以看到有一台从服务器连接到了主服务器。至此，主从架构	   就搭建好了。

<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200823211554318.png" alt="image-20200823211554318" style="zoom:50%;" />

当主数据库遇到异常中断服务后，开发者可以通过手动的方式选择一个从数据库来升格为主数据库，以使得系统能够继续提供服务。	    然而整个过程相对麻烦且需要人工介入，难以实现自动化。 为此，`Redis` 提供了**哨兵工具**来实现自动化的系统监控和故障恢复功		能。



### **六：哨兵**

哨兵的作用就是监控`Redis`主、从数据库是否正常运行，主出现故障自动将从数据库转换为主数据库。它包含以下几点功能：

- 集群监控：负责监控 `Redis master` 和 `slave` 进程是否正常工作。
- 消息通知：如果某个 `Redis` 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。
- 故障转移：如果 `master node` 挂掉了，会自动转移到` slave node` 上。
- 配置中心：如果故障转移发生了，通知 `client` 客户端新的 `master` 地址。

故障转移时，判断一个 `master node` 是否宕机了，需要大部分的哨兵都同意才行，涉及到了分布式选举的问题。

 即使部分哨兵节点挂掉了，哨兵集群还是能正常工作的，因为如果一个作为高可用机制重要组成部分的故障转移系统本身是单点的，那就很坑爹了,所以哨兵我们一般也是集群部署的。



###### **哨兵的核心知识**

- 哨兵至少需要 3 个实例，来保证自己的健壮性。
- 哨兵 + `Redis` 主从的部署架构，是不保证数据零丢失的，只能保证 `Redis` 集群的高可用性。



###### **哨兵主从切换的数据丢失问题**

- **异步复制导致的丢失数据**。因为 `master`到 `slave` 是异步复制的，所有有可能部分数据还没复制到 `slave`, `master` 就宕机了，此时这部分数据就丢失了。
- **脑裂导致的数据丢失**。也就是说，某个 `master` 所在机器突然脱离了正常的网络，跟其他 `slave` 机器不能连接，但是实际上 `master` 还运行着。此时哨兵可能就会认为 `master` 宕机了，然后开启选举，将其他 `slave` 切换成了 `master`。这个时候，集群里就会有两个 `master` ，也就是所谓的**脑裂**。



###### **数据丢失的解决方案**

 在`redis.conf`文件中打开如下配置：

``` shell
min-replicas-to-write 3
min-replicas-max-lag 10
```

表示：要求至少有三个 `slave`，数据复制和同步的延迟不能超过 10 秒。如果说一旦少于三个那么这个时候，`master` 就不会再接收任何请求了，所以，在极端情况下，同步最多丢失 10s 的数据。



###### **主观宕机和客观宕机**

- **主观宕机：**就一个哨兵如果自己觉得一个 `master` 宕机了，那么就是主观宕机。

- **客观宕机：**如果 `quorum` 数量的哨兵都觉得一个 `master` 宕机了，那么就是客观宕机。

  主观宕机 达成的条件很简单，如果一个哨兵 `ping` 一个 `master`，超过了 `sentinel down-after-milliseconds mymaster` 指定的毫秒数之后(在`sentinel.conf`配置文件中可以修改)，就主观认为 `master` 宕机了；如果一个哨兵在指定时间内，收到了 `quorum` 数量的其它哨兵也认为那个 `master` 是 客观宕机 的，那么就认为是 主观宕机了。`quorum`：一般是哨兵实例的一半+1，可以在配置中修改。

  

###### **哨兵的自动发现**

哨兵互相之间的发现，是通过 `Redis` 的 `pub/sub` 系统实现的，每个哨兵都会往 `__sentinel__:hello` 这个 `channel` 里发送一个消息，这时候所有其他哨兵都可以消费到这个消息，并感知到其他的哨兵的存在。

每隔两秒钟，每个哨兵都会往自己监控的某个 `master+slaves` 对应的 `__sentinel__:hello` `channel` 里发送一个消息，内容是自己的 `host`、`ip` 和 `runid` 还有对这个 `master` 的监控配置。

每个哨兵也会去**监听**自己监控的每个 `master+slaves` 对应的 `__sentinel__:hello` `channel`，然后去感知到同样在监听这个 		     	   `master+slaves` 的其他哨兵的存在。

每个哨兵还会跟其他哨兵交换对 `master` 的监控配置，互相进行监控配置的同步。



###### **选举**

如果有个`master`被认为客观宕机了，那么哨兵就会执行主从切换操作，此时需要选举一个 `slave` 来切换成`master`，那么会考虑以下信息：

- 跟` master`的断开时长
- `slave`的优先级
- 复制`offset`
- `run id`

从节点中选择新的主节点：选择的原则是:

- 排除不健康的`slave`节点。

-  按照`slave`优先级进行排序，`slave priority` 越低，优先级越高。
- 如果`slave priority` 相同，那么看`replica offset`  哪个 `slave` 复制了越多的数据，`offset` 越靠后，优先级就越高。
- 如果上面两个条件都相同，那么选择一个 `run id `比较小的那个 `slave`。



###### **哨兵的配置**

**哨兵一般是结合主从架构来使用，所以哨兵的配置是以主从架构为基础。**  **一主两从+三个哨兵节点为例**

一主两从：`redis.conf` 的配置参考主从架构的配置。

三个哨兵节点的配置：

``` shell
# mymaster： 主节点的名字，172.17.0.4： 主节点的 IP， 主节点的端口号，2：一般是哨兵的半数+1，意思是这个数的哨兵认为 maser 宕# 机了，就是客观宕机。
sentinel monitor mymaster 172.17.0.4 6379 2
# 注意：如果你的主从节点设置了密码，要写上配置，不然哨兵监控不到。mymaster：主节点的名字，123456：你主节点设置的密码
# 一定写到上边那个配置的下面，不然会报没有主节点的名字。
sentinel auth-pass mymaster 123456
```

这样，就成功搭建了基于主从架构的哨兵模式。

在 `src`目录下启动主从的`Redis` 实例。

``` shell
./redis-server ../redis.conf 
```

连接客户端，如果设置了密码 就跟上  -a

``` shell
./redis-cli -h <本机 IP> -p <端口号> -a <密码> 
```

输入下面的命令可以看到机器的信息

``` shell
info replication
```

同样的，在`src`目录下启动哨兵节点

``` shell
./redis-server ../sentinel.conf  --sentinel
```

连接客户端，如果设置了密码 就跟上  -a

``` shell
./redis-cli -h <本机 IP> -p <端口号> -a <密码>
```

``` shell
# 可以看到机器的信息
info sentinel

# 可以查看该主机节点下的从节点信息
sentinel slaves <主节点名字>
```

这样，`master`宕机，哨兵会自动选举一个`slave`  为 `master`，如果宕机的机器重新启动，那么这个节点自动会成为新的`master`节点的`slave`节点。

即使使用哨兵，`Redis`每个实例也是全量存储，每个`Redis`存储的内容都是完整的数据。为了最大化利用内存，可以采用集群，就是分布式存储。即每台`Redis`存储不同的内容。那么这个就是`Redis cluster`集群



### **七：Redis Cluster**

在`Redis cluster` 架构下，每个 `Redis` 要放开两个端口号，比如一个是 6379，另外一个就是 加1w 的端口号，比如 16379，16379 端口号是用来进行节点间通信的，用来进行故障检测、配置更新、故障转移授权。

`Redis cluster` 有 `hash slot`的概念，一共有 16384 个 `hash slot` , 它会对每个进来的 key 计算`CRC16`值，然后再用 16384 取模，就可以获得该 key 对应的`hash slot`。`Redis cluster`中每个 `master` 都会持有部分 `slot`，比如有 3 个 `master`，那么可能每个 `master` 持有 5000 多个 `hash slot`。`hash slot `让`node` 的增加和移除很简单，增加一个 `master`，就将其他 `master` 的`hash slot` 移动部分过去，减少一个`master`， 就将它的` hash slot `移动到其他 `master` 上去。任何一台机器宕机，另外两个节点，不影响的。因为 key 找的是 `hash slot`，不是机器。



###### **判断节点宕机**

如果一个节点认为另外一个节点宕机，那么就是 主观宕机。如果多个节点都认为另外一个节点宕机了，那么就是客观宕机，跟哨兵的原理几乎一样。



###### **节点选举**

每个从节点，都根据自己对 `master` 复制数据的 `offset`，来设置一个选举时间，`offset` 越大（复制数据越多）的从节点，选举时间越靠前，优先进行选举。

所有的 `master node` 开始 `slave` 选举投票，给要进行选举的 `slave` 进行投票，如果大部分 `master node `（N/2 + 1) 都投票给了某个从节点，那么选举通过，那个从节点可以切换成 `master`。



###### **Redis cluster 集群搭建**

集群的搭建有两种选择，第一种原生的搭建，就是自己手动修改配置一步一步来，第二种就是利用`Redis`的脚本，很显然我们用第二种。 `Redis cluster`集群需要至少要三个`master`节点。

如下图，我这里用了三个小集群，一共 9 个`Redis`实例。

<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200820173058451.png" alt="image-20200820173058451" style="zoom:50%;" />

9 台`Redis` 实例的`redis.conf` 的配置:

``` shell
# 绑定Ip 指定可以连接本实例Redis的ip  如果注释（删掉）则任意IP都可以连接, 建议绑定本机内网 IP
bind 172.17.0.4

# 如果需要Redis在后台进程运行，就把该项的值设为yes
daemonize yes

# 代表的是持久化的储存目录，RDB 或者 AOF 持久化方案的文件会生成在当前启动目录，如果下一次在其他目录启动 redis，则该持久化文件不 可用。建议修改为固定目录。(改目录手动创建)
dir /data

# 启动集群模式
cluster-enabled yes

# 节点互连超时的阀值。集群节点超时毫秒数
cluster- node-timeout 5000

# 在进行故障转移的时候，全部slave都会请求申请为master，但是有些slave可能与master断开连接一段时间了，导致数据过于陈旧，这样的pslave不应该被提升为master。该参数就是用来判断slave节点与master断线的时间是否过长。判断方法是：比较slave断开连接的时间和(node-timeout * slave-validity-factor) + repl-ping-slave-period，如果节点超时时间为三十秒, 并且slave-validity-factor为10,假设默认的repl-ping-slave-period是10秒，即如果超过310秒slave将不会尝试进行故障转移，可能出现由于某主节点失联却没有从节点能顶上的情况，从而导致集群不能正常工作，在这种情况下，只有等到原来的主节点重新回归到集群，集群才恢复运作如果设置成０，则无论从节点与主节点失联多久，从节点都会尝试升级成主节
cluster-replica-validity-factor 10

# master的slave数量大于该值，slave才能迁移到其他孤立master上，如这个参数若被设为2，那么只有当一个主节点拥有2c个可工作的从节点时，它的一个从节点会尝试迁移。主节点需要的最小从节点数，只有达到这个数，主节点失败时，它从节点才会进行迁移。
cluster-migration-barrier 1
```

启动这九台`Redis`实例，然后在`src` 目录下执行`redis-cli`命令，创建集群。

``` shell
# 有几台实例，后边就跟几个 IP 地址:端口号  ，2：代表一个主:从的比例。
./redis-cli --cluster create <IP地址>:<端口号> --cluster-replicas 2
```

连接任意一个客户端，在`src`目录下启动 : 

``` shell
# -c表示集群模式（只有 redis-cli 有这个功能），指定ip地址和端口号
./redis-cli -c -h <IP地址> -p <端口号>

# 查看集群信息
cluster info

# 查看节点列表
cluster nodes
```

<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200821172937531.png" alt="image-20200821172937531" style="zoom:50%;" />



> 最前面那一串数字：a73691faa9782d05c4ef01996fa792365cf5473a   :   系统生成的唯一的 id。
>
> salve 节点 后边跟的一串数 ：2b66feda8f5a9758a1838d5896c3698cb28c44a  是他的主节点。
>
> master： 主服务器
>
> slave： 从服务器
>
> 0-5460， 5461-10922， 10923-16383 ：自动分配的 hash  slot ，只有主节点会有。
>
> 

关闭集群需要逐个关闭，在`src`目录下使用命令：

``` shell
./redis-cli -c -h <IP地址> -p <端口号> shutdown
```



###### **扩容集群**

 **1：添加集群**

​    当然了，要先准备机器，配置好以后，启动。

​    使用`redis-cli` 语法，在`src` 目录下：

``` shell
# 添加集群，如果添加主节点
./redis-cli --cluster add-node <新的IP>:<新的端口号> <集群中已存在的IP>:<对应的端口号>

#如果添加从节点
./redis-cli --cluster add-node <新的IP>:<新的端口号> <集群中已存在的IP>:<对应的端口号> --cluster-slave --cluster-master-id <对应master的id>
```

​	这样，就把新的机器添加到集群里面了，但是你会发现，新添加的主节点没有 `host slot`,  虽然集群可以正常工作，但是新加进来	       的机器是不能储存数据的，所以，我们要给它分配 `host slot`，当然了，这个`redis.conf` 文件可以配置。



 **2: `host slot` 迁移**

​    	使用`redis-cli` 语法，在`src` 目录下：

``` shell
./redis-cli --cluster reshard <集群中已存在的 IP>:<对应的端口号>
```

 

![image-20200821194055018](https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200821194055018.png)



解释一下：（对应 `host slot` 的数据也会跟随一同迁移）

​		我们执行上边的命令后，系统会问你要分配多少个`host slost` 出去？ --->  要分配到哪个主节点，输入该主节点的 id--->

-  **all**: 从已经有 `host slot` 的主节点，平均拿出来一部分, 我们可以输入 all  ----->  yes   结束
- 选择具体的主节点的 id，可以选择不止一个，然后选择完了之后，done ------> yes  ------>  结束



###### **缩容集群**

使用`redis-cli` 语法，在`src` 目录下：

``` shell
# 移除 host slot
./redis-cli --cluster reshard <集群中已存在节点ip>:<对应的端口> --cluster-from <要迁出节点ID> --cluster-to <接收槽节点ID> --cluster-slots <迁出槽数量>

# 删除节点，尽量先删除从节点，如果先删除主节点，redis cluster 会执行故障转移，避免不必要的开销。
./redis-cli --cluster del-node <集群中已存在节点ip>:<对应的端口> <要删除节点的 id>
```

``` shell
# 默认情况下，集群全部的slot有节点分配，集群状态才为ok，才能提供服务。设置为no，可以在slot没有全部分配的时候提供服务。
# 不建议打开该配置，这样会造成分区的时候，小分区的mster一直在接受写请求，而造成很长时间数据不一致。
# 在部分key所在的节点不可用时，如果此参数设置为”yes”(默认值), 则整个集群停止接受操作；如果此参数设置为”no”，则集群依然为可达节点上的key提供读操作
# cluster-require-full-coverage yes
```



 **注意：所有命令忘了的，可以再`src `目录下用下边的命令查看。**

``` shell
./redis-cli --cluster help
```



### **八：穿透、击穿、雪崩**

###### 	**一：穿透**

穿透：查询缓存没有数据，数据库也没有数据。常见的场景就是黑客攻击。 

 解决方案：

- 缓存空对象 
- 布隆过滤器

###### **二：击穿**

 击穿： 就是说某个 key 非常热点，访问非常频繁，处于集中式高并发访问的情况，当这个 key 在失效的瞬间，大量的请求就击穿了			 缓存，直接请求数据库，就像是在一道屏障上凿开了一个洞。

不同场景下的解决方式可如下：

- 若缓存的数据是基本不会发生更新的，则可尝试将该热点数据设置为永不过期。
- 若缓存的数据更新不频繁，且缓存刷新的整个流程耗时较少的情况下，则可以采用基于 `Redis`、`zookeeper` 等分布式中间件的分布式互斥锁，或者本地互斥锁以保证仅少量的请求能请求数据库并重新构建缓存，其余线程则在锁释放后能访问到新缓存。
- 若缓存的数据更新频繁或者在缓存刷新的流程耗时较长的情况下，可以利用定时线程在缓存过期前主动地重新构建缓存或者延后缓存的过期时间，以保证所有的请求能一直访问到对应的缓存。

###### **三：雪崩**

雪崩：缓存机器意外发生了全盘宕机，或者缓存机器中大部分数据失效了，一瞬间所有访问全部打到数据库了，导致数据库挂掉了，			这是缓存雪崩。

解决方案：

- 规避缓存雪崩：`Redis`搭建高可用集群，并且`Redis`要做持久化。
- 出现雪崩时：限流、降级。



### **九：布隆过滤器**

######  **一：什么是布隆过滤器？**

​       它实际上 是一个很长的二进制向量和一系列随机映射函数 ，当布隆过滤器说某个值存在时，这个值可能存在；当它说不存在			   时，那么 一定不存在。打个比方，当它说不认识你时，那就是真的不认识，但是当它说认识你的时候，可能是因为你长得像它认			  识的另外一个朋友，所以误判认识你。      

###### **二：布隆过滤器的原理**

布隆过滤器本质上是由长度为 `m` 的位向量或位列表（仅包含 `0` 或 `1` 位值的列表）组成，最初所有的值均设置为 `0`，所以我们先来创建一个稍微长一些的位向量用作展示：

![image-20200823122025190](https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200823122025190.png)

当我们向布隆过滤器中添加数据时，会使用 ==**多个**== `hash` 函数对 `key` 进行运算，算得一个证书索引值，然后对位数组长度进行取模运算得到一个位置，每个 `hash` 函数都会算得一个不同的位置。再把位数组的这几个位置都置为 `1` 就完成了 `add` 操作。

例如：添加一个 `wmyskxz`：

![image-20200823122657304](https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200823122657304.png)

向布隆过滤器查询 `key` 是否存在时，跟 `add` 操作一样，会把这个 `key` 通过相同的多个 `hash` 函数进行运算，查看对应的位置是否 都为 `1`，**只要有一个位为 0**，那么说明布隆过滤器中这个 `key` 不存在。如果这几个位置都是 `1`，并不能说明这个 `key` 一定存在，只能说极有可能存在，因为这些位置的 `1` 可能是因为其他的 `key` 存在导致的。

就比如我们在 `add` 了一定的数据之后，查询一个 **不存在** 的 `key`：

![image-20200823122712250](https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200823122712250.png)

很明显，`1/3/5` 这几个位置的 `1` 是因为上面第一次添加的 `wmyskxz` 而导致的，所以这里就存在 **误判**。幸运的是，布隆过滤器有一个可以预判误判率的公式，比较复杂，感兴趣的朋友可以自行去阅读。只需要记住以下几点就好了：

- 使用时不要让实际元素数量远大于初始化数量。
- 当实际元素数量超过初始化数量时，应该对布隆过滤器进行重建，重新分配一个 `size` 更大的过滤器，再将所有的历史元素批量 `add` 进行。



###### **三：布隆过滤器的使用**

**`Redis`官方的布隆过滤器**

  	 `Redis` 布隆过滤器有四个主要的命令

​			**1:  `bf.add <key> <value>`**   : 添加元素到布隆过滤器中，一次只能添加一条数据。

​            **2:  `bf.exists <key> <value>` :** 判断元素是否在过滤器中，一次只能判断一条数据。

​			**3:  `bf.madd <key> <value>`**   : 添加元素到布隆过滤器中，一次能添加多条数据。

​            **4:  `bf.mexists <key> <value>` :** 判断元素是否在过滤器中，一次能判断多条数据。

   因为`Redis`的布隆过滤器是以模块形式提供的，所以，我们要 [下载该模块](https://github.com/RedisBloom/RedisBloom/releases/tag/v2.2.4)，找到最新版本下载即可。

- 下载该模块，然后解压，进入到该文件下 执行`make`命令。

- 在`Redis.conf`文件里加入该模块。

  ``` shell
  ################################## MODULES #####################################
  
  # Load modules at startup. If the server is not able to load modules
  # it will abort. It is possible to use multiple loadmodule directives.
  #
  # loadmodule /path/to/my_module.so
  # loadmodule /path/to/other_module.so
  loadmodule /opt/RedisBloom-2.2.4/redisbloom.so
  ```

- 正常启动`Redis`即可。



### **十：数据库与缓存的数据一致性**

​	**先删缓存再更新数据库**

​			为什么要删除缓存呢？因为很多时候，在复杂点的缓存场景，缓存不单单是数据库中直接取出来的值。其实删除缓存，而不是		 更新缓存，就是一个 lazy 计算的思想，不要每次都重新做复杂的计算，不管它会不会用到，而是让它到需要被使用的时候再重新计	     算。



### **十一： 分布式锁**

我们在生产中，`Redis`一般都是用集群情形，因为`Redis`的主从复制是异步的，这可能会出现在数据同步过程中，`master`宕 			       机，`slave`来不及同步数据就被选为`master`，从而数据丢失。

 流程如下：

- 客户端 1 从`master` 获得了锁
- `master`宕机了，但是储存锁的`key`还没有来得及同步到`slave`
- `slave`升级为了`master`
- 导致锁丢失

所以，作者提出了**RedLock算法**, 步骤大概是这样，假设我们有 N 个`master`节点。

- 获取当前时间 (毫秒)
- 轮流用相同的 `key` 和具有唯一性的`value`在N个节点上请求锁，在这一步里，客户端在每个`master`上请求锁时，客户端应该设置一个网络连接和响应超时时间, 这个超时时间应该小于锁的失效时间。比如如果锁自动释放时间是10秒钟，那每个节点锁请求的超时时间可能是5-50毫秒的范围，这个可以防止一个客户端在某个宕掉的`master`节点上阻塞过长时间，如果一个`master`节点不可用了，我们应该尽快尝试下一个`master`节点。
- 客户端计算第二步中获取锁所花的时间，只有当客户端在大多数`master`节点上成功获取了锁（N/2+1），而且总共消耗的时间不超过锁释放时间，这个锁就认为是获取成功了。
- 如果锁获取成功了，那现在锁自动释放时间就是最初的锁释放时间减去之前获取锁所消耗的时间。
- 如果锁获取失败了，不管是因为获取成功的锁不超过一半（N/2+1)还是因为总消耗时间超过了锁释放时间，客户端都会到每个`master`节点上释放锁，即便是那些他认为没有获取成功的锁。

`Redisson` 做了`RedLock`算法的分布式锁封装，首先看下大佬的总结图：

<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200824142013358.png" alt="image-20200824142013358" style="zoom:70%;" />

​	

​		项目中引入`Redisson`依赖    

``` xml
<!-- https://mvnrepository.com/artifact/org.redisson/redisson -->
<dependency>
	<groupId>org.redisson</groupId>
	<artifactId>redisson</artifactId>
	<version>3.13.3</version>
</dependency>
```


​	**哨兵模式下：**

``` java
public class TestLock {

    public static void main(String[] args) {
        Config config = new Config();
        config.useSentinelServers().addSentinelAddress(
                "redis://172.29.3.245:26378","redis://172.29.3.245:26379", "redis://172.29.3.245:26380")
                .setMasterName("mymaster")
                .setPassword("a123456").setDatabase(0);

        // 构造RedissonClient
        RedissonClient redissonClient = Redisson.create(config);
        // 设置锁定资源名称
        RLock disLock = redissonClient.getLock("DISLOCK");
        boolean isLock;
        try {
            //尝试获取分布式锁
            isLock = disLock.tryLock(500, 15000, TimeUnit.MILLISECONDS);
            if (isLock) {
                //TODO if get lock success, do something;
                Thread.sleep(15000);
            }
        } catch (Exception e) {
        } finally {
            // 无论如何, 最后都要解锁
            disLock.unlock();
        }
    }
}      
```



**Redis  cluster 模式下：**

``` java
public class TestLock {

    public static void main(String[] args) {
        Config config = new Config();
        config.useClusterServers().addNodeAddress(
                "redis://172.29.3.245:6375","redis://172.29.3.245:6376", "redis://172.29.3.245:6377",
                "redis://172.29.3.245:6378","redis://172.29.3.245:6379", "redis://172.29.3.245:6380")
                .setPassword("a123456").setScanInterval(5000);

        // 构造RedissonClient
        RedissonClient redissonClient = Redisson.create(config);
        // 设置锁定资源名称
        RLock disLock = redissonClient.getLock("DISLOCK");
        boolean isLock;
        try {
            //尝试获取分布式锁
            isLock = disLock.tryLock(500, 15000, TimeUnit.MILLISECONDS);
            if (isLock) {
                //TODO if get lock success, do something;
                Thread.sleep(15000);
            }
        } catch (Exception e) {
        } finally {
            // 无论如何, 最后都要解锁
            disLock.unlock();
        }
    }
}
```



通过代码可知，经过`Redisson`的封装，实现Redis`分布式锁非常方便`， 那么实现分布式锁的一个非常重要的点就是`set`的`value`要具有唯一性，`Redisson`的`value`是怎样保证`value`的唯一性呢？ 答案是 `UUID` 和当前线程`ID`	

入口在 `redissonClient.getLock("REDLOCK_KEY")`

![image-20200824144217443](/Users/xiaosa/Library/Application Support/typora-user-images/image-20200824144217443.png)



### **十二：Redis 的多线程**

`Redis`的性能瓶颈一般是内存和网络`I/O`，那么能不能优化网络`I/O` 来减少时间呢，利用多核的优势，所以在`Redis 6.0`版本引入了多线程。

**1：  `Redis`默认是不开启多线程的。**

``` shell
# io-threads-do-reads no
```

**2： 如果开启了多线程，那么多少线程数合适呢？**

``` shell
# io-threads 4
```

关于线程数的设置，官方的建议是：4核的机器建议设置为2或3个线程，8核的建议设置为6个线程，线程数一定要小于机器核数。还		需要注意的是，线程数并不是越大越好，官方认为超过了8个基本就没什么意义了。

**3：开启多线程以后我们需要考虑并发的安全问题吗？**

​	 不需要的，从实现机制看，`Redis`的多线程部分只是用来处理网络数据的读写和协议解析，执行命令仍然是单线程顺序执行。

