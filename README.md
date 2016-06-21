# redis

市面上太多kv的缓存，最常用的就属memcache了，但是memcache存在单点问题，不过小日本有复制版本，但是使用的人比较少，redis的出现让kv内存存储的想法成为现实。今天主要内容便是redis主从实现简单的集群，实际上redis的安装配置砸门ttlsa之前就有个文章，废话少说，进入正题吧


##Redis简介

```
redis是一个key-value存储系统。和Memcached类似，它支持存储的value类型相对更多，包括string(字符串)、 list(链表)、set(集合)和zset(有序集合)。这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操 作，而且这些操作都是原子性的。在此基础上，redis支持各种不同方式的排序。与memcached一样，为了保证效率，数据都是缓存在内存中。区别的 是redis会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件，并且在此基础上实现了master-slave(主从)同步。
Redis 是一个高性能的key-value数据库。 redis的出现，很大程度补偿了memcached这类key/value存储的不足，在部 分场合可以对关系数据库起到很好的补充作用。它提供了Python，Ruby，Erlang，PHP客户端，使用很方便。
Redis是一个开源，先进的key-value存储，并用于构建高性能，可扩展的Web应用程序的完美解决方案。
Redis从它的许多竞争继承来的三个主要特点：
Redis数据库完全在内存中，使用磁盘仅用于持久性。
相比许多键值数据存储，Redis拥有一套较为丰富的数据类型。
Redis可以将数据复制到任意数量的从服务器。

```

##Redis 优势
```
Redis是一个开源，先进的key-value存储，并用于构建高性能，可扩展的Web应用程序的完美解决方案。
Redis从它的许多竞争继承来的三个主要特点：
Redis数据库完全在内存中，使用磁盘仅用于持久性。
相比许多键值数据存储，Redis拥有一套较为丰富的数据类型。
Redis可以将数据复制到任意数量的从服务器。

异常快速：Redis的速度非常快，每秒能执行约11万集合，每秒约81000+条记录。
支持丰富的数据类型：Redis支持最大多数开发人员已经知道像列表，集合，有序集合，散列数据类型。这使得它非常容易解决各种各样的问题，因为我们知道哪些问题是可以处理通过它的数据类型更好。
操作都是原子性：所有Redis操作是原子的，这保证了如果两个客户端同时访问的Redis服务器将获得更新后的值。
多功能实用工具：Redis是一个多实用的工具，可以在多个用例如缓存，消息，队列使用(Redis原生支持发布/订阅)，任何短暂的数据，应用程序，如Web应用程序会话，网页命中计数等。
```

## Redis主从复制的实现

##  Redis Sentinel;Redis Sentinel配置;redis集群;redis故障转移


可关注：http://blog.yangcvo.me/categories/redis/ 


## Redis 配置文件：
我整理下Redis配置文件做了解析：

```
# 守护进程模式
daemonize yes 

# pid file
pidfile /var/run/redis.pid

# 监听端口
port 7003

# TCP接收队列长度，受/proc/sys/net/core/somaxconn和tcp_max_syn_backlog这两个内核参数的影响
tcp-backlog 511

# 一个客户端空闲多少秒后关闭连接(0代表禁用，永不关闭)
timeout 0

# 如果非零，则设置SO_KEEPALIVE选项来向空闲连接的客户端发送ACK
tcp-keepalive 60

# 指定服务器调试等级
# 可能值：
# debug （大量信息，对开发/测试有用）
# verbose （很多精简的有用信息，但是不像debug等级那么多）
# notice （适量的信息，基本上是你生产环境中需要的）
# warning （只有很重要/严重的信息会记录下来）
loglevel notice

# 指明日志文件名
logfile "./redis7003.log"

# 设置数据库个数
databases 16

# 会在指定秒数和数据变化次数之后把数据库写到磁盘上
# 900秒（15分钟）之后，且至少1次变更
# 300秒（5分钟）之后，且至少10次变更
# 60秒之后，且至少10000次变更
save 900 1
save 300 10
save 60 10000


# 默认如果开启RDB快照(至少一条save指令)并且最新的后台保存失败，Redis将会停止接受写操作
# 这将使用户知道数据没有正确的持久化到硬盘，否则可能没人注意到并且造成一些灾难
stop-writes-on-bgsave-error yes

# 当导出到 .rdb 数据库时是否用LZF压缩字符串对象
rdbcompression yes

# 版本5的RDB有一个CRC64算法的校验和放在了文件的最后。这将使文件格式更加可靠。
rdbchecksum yes

# 持久化数据库的文件名
dbfilename dump.rdb

# 工作目录
dir ./

# 当master服务设置了密码保护时，slav服务连接master的密码
masterauth 0234kz9*l

# 当一个slave失去和master的连接，或者同步正在进行中，slave的行为可以有两种：
#
# 1) 如果 slave-serve-stale-data 设置为 "yes" (默认值)，slave会继续响应客户端请求，
# 可能是正常数据，或者是过时了的数据，也可能是还没获得值的空数据。
# 2) 如果 slave-serve-stale-data 设置为 "no"，slave会回复"正在从master同步
# （SYNC with master in progress）"来处理各种请求，除了 INFO 和 SLAVEOF 命令。
slave-serve-stale-data yes

# 你可以配置salve实例是否接受写操作。可写的slave实例可能对存储临时数据比较有用(因为写入salve
# 的数据在同master同步之后将很容易被删除
slave-read-only yes

# 是否在slave套接字发送SYNC之后禁用 TCP_NODELAY？
# 如果你选择“yes”Redis将使用更少的TCP包和带宽来向slaves发送数据。但是这将使数据传输到slave
# 上有延迟，Linux内核的默认配置会达到40毫秒
# 如果你选择了 "no" 数据传输到salve的延迟将会减少但要使用更多的带宽
repl-disable-tcp-nodelay no

# slave的优先级是一个整数展示在Redis的Info输出中。如果master不再正常工作了，哨兵将用它来
# 选择一个slave提升=升为master。
# 优先级数字小的salve会优先考虑提升为master，所以例如有三个slave优先级分别为10，100，25，
# 哨兵将挑选优先级最小数字为10的slave。
# 0作为一个特殊的优先级，标识这个slave不能作为master，所以一个优先级为0的slave永远不会被
# 哨兵挑选提升为master
slave-priority 100


# 密码验证
# 警告：因为Redis太快了，所以外面的人可以尝试每秒150k的密码来试图破解密码。这意味着你需要
# 一个高强度的密码，否则破解太容易了
requirepass 0234kz9*l 

# redis实例最大占用内存，不要用比设置的上限更多的内存。一旦内存使用达到上限，Redis会根据选定的回收策略（参见：
# maxmemmory-policy）删除key
maxmemory 3gb

# 最大内存策略：如果达到内存限制了，Redis如何选择删除key。你可以在下面五个行为里选：
# volatile-lru -> 根据LRU算法删除带有过期时间的key。
# allkeys-lru -> 根据LRU算法删除任何key。
# volatile-random -> 根据过期设置来随机删除key, 具备过期时间的key。 
# allkeys->random -> 无差别随机删, 任何一个key。 
# volatile-ttl -> 根据最近过期时间来删除（辅以TTL）, 这是对于有过期时间的key 
# noeviction -> 谁也不删，直接在写操作时返回错误。
maxmemory-policy volatile-lru

# 默认情况下，Redis是异步的把数据导出到磁盘上。这种模式在很多应用里已经足够好，但Redis进程
# 出问题或断电时可能造成一段时间的写操作丢失(这取决于配置的save指令)。
#
# AOF是一种提供了更可靠的替代持久化模式，例如使用默认的数据写入文件策略（参见后面的配置）
# 在遇到像服务器断电或单写情况下Redis自身进程出问题但操作系统仍正常运行等突发事件时，Redis
# 能只丢失1秒的写操作。
#
# AOF和RDB持久化能同时启动并且不会有问题。
# 如果AOF开启，那么在启动时Redis将加载AOF文件，它更能保证数据的可靠性。
appendonly no

# aof文件名
appendfilename "appendonly.aof"

# fsync() 系统调用告诉操作系统把数据写到磁盘上，而不是等更多的数据进入输出缓冲区。
# 有些操作系统会真的把数据马上刷到磁盘上；有些则会尽快去尝试这么做。
#
# Redis支持三种不同的模式：
#
# no：不要立刻刷，只有在操作系统需要刷的时候再刷。比较快。
# always：每次写操作都立刻写入到aof文件。慢，但是最安全。
# everysec：每秒写一次。折中方案。 
appendfsync everysec

# 如果AOF的同步策略设置成 "always" 或者 "everysec"，并且后台的存储进程（后台存储或写入AOF
# 日志）会产生很多磁盘I/O开销。某些Linux的配置下会使Redis因为 fsync()系统调用而阻塞很久。
# 注意，目前对这个情况还没有完美修正，甚至不同线程的 fsync() 会阻塞我们同步的write(2)调用。
#
# 为了缓解这个问题，可以用下面这个选项。它可以在 BGSAVE 或 BGREWRITEAOF 处理时阻止主进程进行fsync()。
# 
# 这就意味着如果有子进程在进行保存操作，那么Redis就处于"不可同步"的状态。
# 这实际上是说，在最差的情况下可能会丢掉30秒钟的日志数据。（默认Linux设定）
# 
# 如果你有延时问题把这个设置成"yes"，否则就保持"no"，这是保存持久数据的最安全的方式。
no-appendfsync-on-rewrite yes

# 自动重写AOF文件
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# AOF文件可能在尾部是不完整的（这跟system关闭有问题，尤其是mount ext4文件系统时
# 没有加上data=ordered选项。只会发生在os死时，redis自己死不会不完整）。
# 那redis重启时load进内存的时候就有问题了。
# 发生的时候，可以选择redis启动报错，并且通知用户和写日志，或者load尽量多正常的数据。
# 如果aof-load-truncated是yes，会自动发布一个log给客户端然后load（默认）。
# 如果是no，用户必须手动redis-check-aof修复AOF文件才可以。
# 注意，如果在读取的过程中，发现这个aof是损坏的，服务器也是会退出的，
# 这个选项仅仅用于当服务器尝试读取更多的数据但又找不到相应的数据时。
aof-load-truncated yes

# Lua 脚本的最大执行时间，毫秒为单位
lua-time-limit 5000

# Redis慢查询日志可以记录超过指定时间的查询
slowlog-log-slower-than 10000

# 这个长度没有限制。只是要主要会消耗内存。你可以通过 SLOWLOG RESET 来回收内存。
slowlog-max-len 128

# redis延时监控系统在运行时会采样一些操作，以便收集可能导致延时的数据根源。
# 通过 LATENCY命令 可以打印一些图样和获取一些报告，方便监控
# 这个系统仅仅记录那个执行时间大于或等于预定时间（毫秒）的操作, 
# 这个预定时间是通过latency-monitor-threshold配置来指定的，
# 当设置为0时，这个监控系统处于停止状态
latency-monitor-threshold 0

# Redis能通知 Pub/Sub 客户端关于键空间发生的事件，默认关闭
notify-keyspace-events ""

# 当hash只有少量的entry时，并且最大的entry所占空间没有超过指定的限制时，会用一种节省内存的
# 数据结构来编码。可以通过下面的指令来设定限制
hash-max-ziplist-entries 512
hash-max-ziplist-value 64

# 与hash似，数据元素较少的list，可以用另一种方式来编码从而节省大量空间。
# 这种特殊的方式只有在符合下面限制时才可以用
list-max-ziplist-entries 512
list-max-ziplist-value 64

# set有一种特殊编码的情况：当set数据全是十进制64位有符号整型数字构成的字符串时。
# 下面这个配置项就是用来设置set使用这种编码来节省内存的最大长度。
set-max-intset-entries 512

# 与hash和list相似，有序集合也可以用一种特别的编码方式来节省大量空间。
# 这种编码只适合长度和元素都小于下面限制的有序集合
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

# HyperLogLog稀疏结构表示字节的限制。该限制包括
# 16个字节的头。当HyperLogLog使用稀疏结构表示
# 这些限制，它会被转换成密度表示。
# 值大于16000是完全没用的，因为在该点
# 密集的表示是更多的内存效率。
# 建议值是3000左右，以便具有的内存好处, 减少内存的消耗
hll-sparse-max-bytes 3000

# 启用哈希刷新，每100个CPU毫秒会拿出1个毫秒来刷新Redis的主哈希表（顶级键值映射表）
activerehashing yes

# 客户端的输出缓冲区的限制，可用于强制断开那些因为某种原因从服务器读取数据的速度不够快的客户端
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# 默认情况下，“hz”的被设定为10。提高该值将在Redis空闲时使用更多的CPU时，但同时当有多个key
# 同时到期会使Redis的反应更灵敏，以及超时可以更精确地处理
hz 10

# 当一个子进程重写AOF文件时，如果启用下面的选项，则文件每生成32M数据会被同步
aof-rewrite-incremental-fsync yes
```
