# 介绍

- 基于内存的数据库，读写操作都是在内存中完成，读写速度特别快，常用于缓存，消息队列，分布式锁等场景
- 命令的执行有单线程负责，不存在并发竞争的问题
- 高性能，高并发

# 数据类型

## String

最基本的kv结构，v可以是字符串或数字，数据最多占用512MB

### API

以下操作复杂度均为O(1)

- set
- get
- exists
- strlen
- del
- incr key
- incrby key n
- expire key sec
- ttl key
- -ex 设置过期时间
- -nx 不存在才插入

以下操作时间复杂度为O(K)，K = 操作元素数量

- mset
- mget

### 应用场景

- 缓存对象，如将对象json序列化后存储
- 计数，incr，不用考虑并发
- 分布式锁，分布式服务共享数据
  - 防止死锁要加过期时间

### 内部实现

- int
- sds
  - 简单动态字符串
  - 不仅可以保存文本数据，还可以保存二进制数据
  - 获取长度时间复杂度O(1)
  - 可细分为embstr和raw，大于39字节为raw
  - embstr对内存做了优化，但是只读

## List

简单字符串列表，按插入顺序排序，可从头尾插入

最大长度为2^32 -1，约40亿

### API

- lpush
- rpush
- lpop
- rpop
- lrange key start stop
- blpop key1 key2 阻塞pop

### 应用场景

- 消息队列
  - 消息保序 lpush + rpop
  - 阻塞读取 brpop
  - 重复消息处理 生产者实现全局唯一ID
  - 可靠性 使用brpoplpush
  - 不支持消费者组，即多个消费者不可以消费同一条消息

### 内部实现

- 双向链表
- 压缩列表
  - 元素数量小于512（可配置）且每个元素小于64B（可配置）
- quicklist
  - 3.2以后全部使用这个数据结构

## Hash

键值对集合，特别适合存对象

### API

- hset
- hget
- hmset
- hmget
- hdel
- hlen
- hgetall
- hincrby key field n

### 应用场景

- 缓存对象
  - 一般对象用string
  - 经常修改的对象用hash

### 内部实现

- 压缩列表
  - 元素数小于512（可配置）且元素小于64B（可配置）
  - 省空间
- 哈希表
- listpack
  - 7.0版本替代压缩列表

## Set

无序且唯一的键值集合

最多存储2^32-1个元素

### API

- sadd
- srem
- smembers 获取所有元素
- scard 获取元素个数
- sismember key member 判断member是否存在集合中
- srandmember 随机读
- spop 随机读删
- sinter key1 key2 交集运算
- sinterstore destination key1 key2 交集运算并存储
- sunion 交集运算
- sdiff 差集运算

### 应用场景

- 聚合计算（并交差集）

### 内部实现

- 哈希表
- 整数集合
  - 元素都是整数且个数小于512（可配置）

## Zset

有序集合，相较于set多了元素分值

### API

- zadd
- zrem
- zscore
- zcard
- zincrby
- zrange key start stop获取范围
- zrevrange 倒序获取

### 应用场景

- 排序场景

### 内部实现

- 压缩列表
- 跳表
- listpack
  - 7.0替换了压缩列表

## 不常用

- BitMap（2.2） 二进制值统计场景，如用户登录状态
- HyperLogLog（2.8） 海量数据基数统计场景
- GEO（3.2） 存储地理位置信息场景
- Stream（5.0）消息队列，相较于List，可以自动生成全局唯一消息ID，支持以消费者组形式消费数据

# 持久化

## AOF

redis每执行一条写操作命令，就把该命令以追加的方式写入一个文件里

重启redis的时候，先去读取这个文件里的命令并且执行它，以此来恢复数据

### 写文件顺序

先执行命令，再记录日志

- 优点
  - 避免额外的检查开销。如果是先写日志，则写日志前需要先进行命令语法检查，增加了额外的开销
  - 不会阻塞当前写操纵命令的执行
- 缺点
  - 如果执行命令后redis还没来得及写硬盘服务器发生宕机，数据有丢失风险
  - 写入操作如果IO压力大，会给下一个命令带来阻塞

### 写回策略

![img](https://img-blog.csdnimg.cn/img_convert/4eeef4dd1bedd2ffe0b84d4eaa0dbdea.png)

写文件逻辑如上图

1. redis执行写操作后，命令追加到 aof_buf 缓冲区
2. write()系统调用，将缓冲区数据写入AOF文件。此时数据没有真正写入到硬盘，而是拷贝到了内核缓冲区，等待内核将数据写入硬盘
3. 内核缓冲区数据什么时候写入硬盘，由内核控制，可以通过调用fsync()立即写入。调用fsync时机可配置



控制fsync()时机

- always 每执行一条写命令就调用fsync，同步写回，会影响主线程性能
- no 不调用fsync，具体写入时间由操作系统决定，不可预知
- everysec 每秒调用fsync，最推荐的配置

### AOF重写

随着写操作命令越来越多，文件大小越来越大，重启Redis恢复过程就很慢，所以提供AOF重写机制

#### 重写时机

AOF文件大小超过所设定的阈值时

#### 重写逻辑

主线程发现满足AOF重写条件时，启动后台子进程读取当前数据库中所有的键值对，将每一个键值对用一条命令记录到新的AOF文件中，然后替换旧的AOF文件。操作会将历史无用数据剔除

#### 重写时数据发生变化

主线程记录命令并写入aof重写缓冲区，aof重写完成后将aof重写缓冲区的数据追加到新的aof文件尾

## RDB

RDB其实就是redis内存全量快照，存的是二进制数据。恢复时直接载入内存就可以

redis没有提供主动执行rdb恢复的接口，只能在服务重启时自动执行

RDB是一种比较重的操作，执行太频繁会对redis性能产生影响。执行不频繁会造成重启丢失更多数据

save执行后主线程修改数据，发生写时复制，rdb复制旧数据

### 执行方式

- save
  - 主线程执行，会阻塞主线程的命令执行。生产环境不要使用
- bgsave
  - fork子进程执行，避免主线程的阻塞

### 执行时机

- 客户端手动执行
- 配置文件设置，满足条件使用bgsave自动执行。xx秒内对数据进行了xx次修改

## RDB+AOF

redis 4.0新功能

混合使用AOF日志和内存快照。工作在AOF重写过程

1. aof重写子进程将快照以RDB方式写入AOF文件
2. 主线程继续处理的操作命令记录在重写缓冲区
3. RDB数据写完后，重写缓冲区的增量命令追加写入到AOF文件



# 高可用

## 主从复制

![图片](https://img-blog.csdnimg.cn/img_convert/22c7fe97ce5d3c382b08d83a4d8a5b96.png)

### 目的

避免单点故障导致服务不可用，将数据备份到其他服务器上。

这样一台服务器故障，其他服务器顶上来继续提供服务

### 数据一致性

redis提供主从复制模式，保证多台服务器数据一致性

### 读写分离

主库负责写数据，从库同步数据并提供读数据服务，降低主库压力

且保证了数据顺序写入，不用考虑并发、死锁

### 流程

可以分为全量复制和命令传播两部分

#### 全量复制

![图片](https://cdn.xiaolincoding.com//mysql/other/ea4f7e86baf2435af3999e5cd38b6a26.png)

1. 从库执行复制命令，指定主库
2. 从库建立连接，发送psync命令，表示要进行数据同步。命令有两个参数
   - runID 标识主服务器ID，第一次同步时为空
   - offset 标识复制进度，第一次同步时为-1
3. 主库收到命令后，返回FULLRESYNC说明此次是全量复制。返回runID和复制进度offset，从库记录这两个值
4. 主库执行bgsave生成rdb快照文件，并发送给从库
5. 从库收到rdb后，清空当前数据，载入rdb快照
6. 主库在生成rdb文件开始到从库加载rdb接数记录写操作并写入replication buffer
7. 主库将replication buffer发送给从库，从库执行，数据一致

#### 命令传播

![图片](https://cdn.xiaolincoding.com//mysql/other/03eacec67cc58ff8d5819d0872ddd41e.png)

全量复制后建立了长连接，将写操作传播给从库

### 分摊主库压力

![图片](https://cdn.xiaolincoding.com//mysql/other/4d850bfe8d712d3d67ff13e59b919452.png)

复制操作占用大量资源

采用树状结构，主库的从库作为其他从库的主库，相当于代理

### 增量复制

若网络断开导致命令传播异常，主库从库数据不一致

2.8之前重新执行全量复制

2.8之后采用增量复制

根据从库复制偏移量确定命令是否还在主库的复制备份缓冲区，在则增量同步

## 哨兵

redis主从架构中，主从配置都是手动的，产生故障也没办法自动转移，需要人工介入

哨兵提供了主从节点故障转移功能，哨兵会检测主节点是否存活，如果主节点挂了，会选举一个从节点切换为主节点，并将其他从节点的主节点更新为新的主节点

### 如何工作

哨兵是特殊的redis进程，负责监控、选主、通知

#### 监控

- 每秒ping所有节点，相应正常说明运行正常
- 没有在规定时间内正常相应则被标记为主观下线
- 节点被判断为主观下线后，会向其他哨兵发送命令，投票决定是否客观下线

#### 选主

- 哨兵集群选一个节点执行主从故障转移
- 首先标记主观下线的节点作为候选者

#### 通知

1. 从已下线节点的从节点选一个作为主节点
2. 让已下线节点的从节点更新主节点信息
3. 将新主节点信息通知客户端
4. 已下线节点上线时，将其设为新主节点的从节点

### 如何判断主节点故障

### 转移流程

### 哨兵集群

## 集群

# 内存

## 过期删除

redis key 有过期时间，需要有相应机制将已过期的键值对删除

### 如何设置过期时间

- expire	 秒后过期
- pexpiren    毫秒后过期
- expireat     某个秒级时间戳到期
- pexpireat   某个毫秒级时间戳到期
- setex
- setpx
- ttl 查看剩余存活时间
- persist 取消过期时间

### 如何判断key是否过期

过期字典记录key和其过期时间

key是只想键值对对象的指针，value是过期时间

查询一个key时收钱判断key是否存在于过期字典中

- 不在过期字段中，正常读取值
- 在过期字典中，对比时间判断是否过期

### 过期删除策略

以下是常见的过期删除策略

- 定时删除
  - 设置key的过期时间同时创建一个定时事件，时间到达时由事件处理器执行删除操作
  - 优点
    - 可以保证key被尽快删除，内存被释放
  - 缺点
    - 性能影响大
- 惰性删除
  - 不主动删除过期键，只在访问时检测key是否过期，过期则删除
  - 优点
    - 占用的系统资源少，CPU友好
  - 缺点
    - key如果一直不被访问，内存不会被释放，造成空间浪费，内存不友好
- 定期删除
  - 每隔一段时间随即从数据库取出一定数量的key进行检查，并删除其中过期的key
  - 优点
    - 限制删除操作的时长和频率，折中的策略
  - 缺点
    - 执行太频繁，CPU不友好
    - 执行不频繁，内存不友好

### Redis的过期删除策略

惰性删除+定期删除

- 惰性删除
  - 访问key前，进行检查。如果过期则删除，没有过期正常返回
- 定期删除
  - 每隔一段时间随机从数据库取出一定数量的key进行检查，并删除其中过期的key
  - 执行间隔10s，可配置
  - 抽查数量 20

定期删除是一个循环的过程，超过25ms退出

1. 从过期字典随机抽取20个key
2. 检查20个key是否过期，并删除已过期的key
3. 如果本轮检查的已过期key数量超过25%，重复步骤1.否则返回

## 内存淘汰

内存过期删除策略是针对已过期的key，而内存淘汰策略是redis在运行内存超过限制后，删除符合套件的key来保障redis的高效运行

### redis最大运行内存

可配置，当运行内存达到配置的上限，会触发内存淘汰

- 64位系统下，默认没有大小限制，超出现之后崩溃
- 32位系统下，默认值为3GB

### 内存淘汰策略

- 不淘汰
  - noeviction
    - redis3.0后默认的内存淘汰策略，超出现之后通知禁止写入
- 淘汰设置了过期时间的
  - volatile-random
    - 随机淘汰设置了过期时间的任意键值
  - volatile-ttl
    - 优先淘汰更早过期时间的键值
  - volatile-lru
    - 淘汰所有设置了过期时间键值中，最久未使用的
  - volatile-lfu
    - 淘汰所有过期时间键值中，最少使用的
- 所有数据范围内淘汰
  - allkeys-random
    - 随机淘汰任意键值
  - allkeys-lru
    - 淘汰最久未使用
  - allkeys-lft
    - 淘汰最少使用

### LRU和LFU

#### LRU

- 淘汰最近最少使用的数据
- 基于链表结构，最新操作的元素移动到表头
- 淘汰时，删除尾部元素即可
- 需要链表管理所有数据，额外内存开销
- 数据被访问时，要进行链表移动操作，很耗时

#### Redis的LRU

- 在Redis对象结构体中添加一个额外字段用于记录最后一次访问时间
- 进行淘汰时，随机采样5个，淘汰最久没有使用的那个
- 节省空间
- 没有移动链表项操作，提升性能
- 无法解决缓存污染：大量数据只被读取一次，却在内存中存留很久

#### LFU

- 淘汰最近最不常用的数据，根据数据访问次数淘汰数据
- 记录每个数据访问次数

#### Redis的LFU

- 在Redis结构体记录数据访问频次

# 缓存

## 缓存雪崩

大量缓存数据在同一时间过期（失效）或Redis故障宕机，导致请求都访问数据库，形成连锁反应

- 大量数据过期
  - 均匀设置过期时间，避免将大量数据设置成同一过期时间，加上随机数
  - 互斥锁，保证同一时间只有一个请求来构建缓存
  - 后台更新缓存？？
- Redis故障宕机
  - 服务熔断或请求限流，减少数据库压力，防止数据库崩溃
  - 构建高可用架构，Redis Sentinel

## 缓存击穿

热点数据过期导致大量请求穿过缓存到达数据库，是缓存雪崩的一个子集

- 互斥锁方案
- 不给热点数据设置过期时间

## 缓存穿透

用户要访问的数据不在缓存也不再数据库，数据库没办法构建缓存来服务后续请求。一般是业务误操作或黑客恶意攻击

- 非法请求限制
- 缓存空值/默认值
- 布隆过滤器，确认缓存失效后用布隆过滤器快速判断
  - 布隆过滤器说数据不存在，则一定不存在

## 一致性

延时双删

# QA

## 大key

- 影响持久化，aof写入可能很久，如果设置了always会阻塞主线程
- 客户端超时阻塞
- 网络阻塞
- unlink删除大key

# IO多路复用

对流的监听有两种模式

- 阻塞等待，不消耗CPU
- 轮询等待，占用CPU资源

一个进程或线程只能阻塞或等待一个事件

- 多进程模型或多线程模型可以同时监听多个事件，但消耗内存
- 遍历轮询等待多个事件可以同时监听多个事件，但消耗CPU

操作系统提过了IO多路复用功能，实现一个进程或线程能够同时阻塞等待多个事件

在网络层就是一个进程或线程监听多个Socket

IO多路复用有以下几种模式

## select

#### 流程

1. 将已连接的Socket都放到一个文件描述符集合，然后调用select函数将文件描述符集合拷贝到内核，让内核来检查是否有网络事件产生
2. 内核通过遍历来检查文件描述符，当检查到有事件产生，将Socket标记为可读或可写
3. 内核将文件描述符集合从内核态拷贝到用户态，应用程序再遍历一次找到可读或可写的Socket

#### 特征

- 每次select返回需要进行2次遍历和两次拷贝
- select使用固定长度的BitsMap，默认1024，监听的文件描述符数量有限

## poll

不再使用BitsMap存储文件描述符，而是用动态数组，以链表形式组织，突破了select的文件描述符个数限制。但会受到Linux可打开限制，系统级、用户级、应用程序级

与select没有本质区别

## epoll

#### 流程

1. 通过epoll_create创建epoll对象epfd
2. 通过epoll_ctl添加修改或删除要监听的描述符和事件
3. 执行epoll_wait等待数据返回

#### 特征

- 使用红黑树跟踪文件描述符，红黑树增删改查一般是O(logN)，每次调用epoll_ctl只用传一个文件描述符，select要传全部的，减少了内核态用户态数据拷贝的开销
- epoll使用事件驱动机制，epoll对象维护一个链表记录就绪事件，当有事件发生就通过回调函数将事件加入到就绪列表
- epoll调用epoll_wait只返回有事件发生的文件描述符，不需要像select那样再次遍历fd集合

#### 水平触发

当有事件发生时，服务器不断从epoll_wait苏醒，知道数据读取完

- 安全性较高，能使数据被处理完。但数据一直不处理

#### 边缘触发

同一个事件发生只会让服务器从epoll_wait苏醒一次，epoll认为下次调用wait时之前的数据已处理完毕