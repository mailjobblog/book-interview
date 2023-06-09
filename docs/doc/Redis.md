# Redis面试题

### Redis单线程为什么这么快

- 才用了 select/epoll 多路复用机制
- 大部分请求基于内存，所以十分迅速
- Redis具有高效的底层数据结构

### Redis如何解决数据存储hash碰撞问题

- 对于碰撞的entry，采用链表保存，然后将该链表的指针指向到hash位，读取数据链表顺序读取
- 如果数据量超过hash链表的70%，那么redis就会进行rehash

### 5种数据类型以及应用场景



### 5种数据类型底层数据结构



### 持久化机制

RDB（默认）：指的是在指定时间内将内存中的数据写入到磁盘中的方式，它的优点是在大规模的数据恢复方面拥有良好的性能，缺点是数据完整性较差。

AOF：指的是追加写日志的一种持久化方式，优点是数据完整性较高，缺点是日志文件的内容较大，不过redis也提供了瘦身机制，当触发配置参数后，redis会fork出一个进程，将aof日志的文件写入到一个临时文件中，并且替换旧的aof文件。

### 内存淘汰策略

LRU：淘汰最长时间未被使用的页面。内部维护的是一个链表结构，当新插入的数据进来或者读取某一个数据的时候，这个key会被移动到链表的头部，当触发内存淘汰策略后，会先淘汰链表尾部的key。

LFU：淘汰一定时间内未被使用的页面。内部维护一个计数器，一般是以分钟为单位（可以修改），当触发淘汰机制的时候，会先淘汰计数最好的页面。

Random：随机淘汰。

### 如何理解分布式锁



### 分布式锁如何续期

“看门狗” 机制

### Zset排名如何保证分数一样的按照最先插入的排名

模式情况下，对于相同的分数插入到 zset 后会按照 key 值的 ASCI 进行排行

- 采用float拼接毫秒小数：例如都是80分。A先来的，先获取当前毫秒数据，例如是：111111，则A的分数是：80.111111。B是后面来的，则B的分数是：80.222222。
- 高位和低位解决：高位保存时间，低位保存分数。例如玩家等级最大100级，玩家A的等级是70，当前时间是 111111，则高位数据是 999999-111111 = 888888，低位数据是70，所以分数是 888888 * 1000 + 80 = 888888070。过了一定时间后，当前时间是 222222，则高位数据是 999999-222222 = 777777，低位数据是70，所以分数是 777777 * 1000 + 80 = 777777070。所以最终可以用计算的结果作为分数，再取数据的时候使用除模取余算法（分数%1000 = 真实分数）即可得到真实分数。

### Zset排名如何保证分数一样的同样的排名

对于取出的数据在Services层再处理一遍

### 如何解决Cache和DB数据一致性问题

对数据一致性要求不高：可以采用异步写会策略，就是DB在Update后然后删除Cache，线程X再次读取数据会重新生成缓存。

对数据一致性要求高：

- 如果是Insert数据，直接写入到DB中，读取的时候交给读取请求构建缓存，所以不会有一致性问题
- 如果是 Update/Delete 数据，可以使用 “重试机制” 解决，就是把要 Update 的数据放到队列中执行，如果更新DB失败或者更新Cache失败，可以重新读取队列处理
- 如果不采用异步，可以用 “延迟双删策略” 解决，线程A先删除缓存，然后更新DB，Sleep睡眠一定时间后（如果线程B在线程A删除缓存前读取缓存，线程A删除缓存后线程B刚好写入缓存，那么会造成脏读，所以设置Sleep的时间要大于线程B的写入时间），然后再次删除缓存。那么，线程X再次读取数据会重新生成缓存。

### Redis热key发现与解决

**如何发现**

- 凭借业务经验，预估哪些是热key
- 在客户端进行收集，但是会对客户端代码造成入侵
- 在Proxy层做收集上报，但是缺点很明显，并非所有的redis集群架构都有proxy

**如何解决**

- 数据缓存到客户端
- Java可以将热key缓存到jvm里面，这样就不用从redis取数据了
- Redis服务做集群，流量打散

### Redis大Key发现与解决

**如何发现**

- 使用 redis-cli --bigkeys 命令。不阻塞，获取速度快。但是获取的信息较少，内容不够精确
- 使用 redis-rdb-tools 工具。会阻塞，获取速度慢。但是获取的信息比较完善
- Redis4.0之后，新增 `memory usage` 命令，随机抽样估算key的大小。优点是：获取的信息准确且及时。缺点是：命令执行会影响线上业务，所以需要做好熔断和监控。

**如何解决**

- 数据缓存到客户端

- 分拆成几个key-value， 也可以将这个存储在一个hash中
- 做Redis集群，将key保存到不同的实例中

**如何删除**

首先不能立即执行删除，因为redis的读写命令是通过单线程执行的，所以有可能引发阻塞问题，进而引发线上的问题

- 业务低峰期删除
- scan命令分批删除
- Redis4.0后可以采用unlink代替delete进行异步线程删除

### Redis服务器CPU飙升如何排查问题

- 首先在Linux使用top命令确定是否redis服务的问题
- 是否存在连接没有被释放
- 查看是否存在大key，如果存在进行拆分
- 查看是否存在热key，如果存在进行客户端缓存或者做redis集群
- 是否有开发者使用了 `keys *` 命令扫描全部内存，知道的命令阻塞
- 查看redis慢日志