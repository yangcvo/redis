---
title: Redis Sentinel实现redis主从自动切换
date: 2015-11-19 11:44:54
tags:
- redis高可用监控sentinel配置和使用
categories: redis
---

## Redis-主从复制原理及分析

### Slave向Master同步的实现是：
``` bash
Slave向Master发出同步请求（发送sync命令），Master先dump出rdb文件，然后将rdb文件全量传输给slave，然后Master把缓存的写命令转发给Slave，初次同步完成。
```
     
     
### 以及以后的同步实现是：
``` bash     
 Master将变量的快照直接实时依次发送给各个Slave。
 但不管什么原因导致Slave和Master断开重连都会重复以上两个步骤的过程。
 Redis的主从复制是建立在内存快照的持久化基础上的，只要有Slave就一定会有内存快照发生。
```
### 主从复制原理总结几点：

``` bash 
1.Slave启动后，无论是第一次连接还是重连到Master，它都会主动发出一个SYNC命令
2.当Master收到SYNC命令之后，将会执行BGSAVE(后台存盘进程)，即在后台保存数据到磁盘（rdb快照文件），同时收集所有新收到的写入和修改数据集的命令存入缓冲区（非查询类）
3.Master在后台把数据保存到快照文件完成后，会传送整个数据库文件到Slave
4.Slave接收到数据库文件后，会把内存清空，然后加载该文件到内存中以完成一次完全同步
5.然后Master会把之前收集到缓冲区中的命令和新的修改命令依次传送给Slave
6.Slave接受到之后在本地执行这些数据修改命令，从而达到最终的数据同步
7.之后Master与Slave之间将会不断的通过异步方式进行命令的同步，从而保证数据的时时同步
8.如果Master和Slave之间的链接出现断连，Slave可以自动重连Master。

根据版本的不同，断连后同步的方式也不同：

2.8之前：重连成功之后，一次全量同步操作将被自动执行
2.8之后：重连成功之后，进行部分同步操作
```
### Redis-sentinel原理分析

``` bash
Redis-sentinel是Redis实例的监控管理、通知和实例失效备援服务，是Redis集群的管理工具。在一般的分布式中心节点数据库中，Redis-sentinel的作用是中心节点的工作，监控各个其他节点的工作情况并且进行故障恢复，来提高集群的高可用性。

Redis-sentinel是Redis的作者antirez在2013年6月份完成的，因为Redis实例在各个大公司的应用，每个公司都需要一个Redis集群的管理工具，被迫都自己写管理工具来管理Redis集群，antirez考虑到社区的急迫需要，花了几个星期写出了Redis-sentinel。
```

### Redis-sentinel三大功能


``` bash
Redis-sentinel的三大功能：监测、通知、自动故障恢复。首先Redis-sentinel要建立一个监控的master列表，然后针对master列表的每个master获取监控其的sentinels和slaves供以后故障恢复使用。
```
### 前言：

网站的访问量慢慢上来了。为了网站的性能方面，开始用了redis做缓存策略。刚开始的时候，redis是一个单点，当一台机器岩机的时候，redis的 服务完全停止，这时就会影响其他服务的正常运行。费话不多说了，下面利用redis sentinel做一个主从切换的集群管理。做这个集群管理的时候，查过很多资料才完全了解，从数据类型、主从原理、持久化方式等方面着手看了不少资料，也进行了一些实践操作。redis的配置都比较简单，网络上相关资料比较多，把实践的过程记录下来以备查阅。

### 补充摘要：

```bash
redis 安装包：http://redis.io/download
中文redis社区:http://redis.cn/community.html
参考资料：http://redis.io/topics/sentinel
```

### 系统环境环境：

这里我用kvm 虚拟出三台服务器：然后搭好了环境。

``` bash
virsh # list --all
 Id      名称              				状态
------------------------------------------
 1   redis_sentinel                 running
 2   LAN_redis2                     running
 3   LAN_redis1                     running

```
集群配置最少需要三台机器，那么我就三台虚拟机,三台虚拟机分别安装同样的redis的环境

ip分别：

* 192.168.1.172  （redis sentinel 集群监控）
* 192.168.1.173  （redis master 节点）
* 192.168.1.174  （redis slave  节点）

``` bash
[root@sentinel ~]# cat /etc/issue
CentOS release 6.7 (Final)
Kernel \r on an \m
```

### redis安装配置:
下载，解压和编译Redis的：master默认配置：

### master：

``` bash 
[root@LAN_redis1 ~]# curl -O http://download.redis.io/releases/redis-3.2.1.tar.gz -C /tmp/.
[root@LAN_redis1 ~]# tar xzf redis-3.2.1.tar.gz -C /usr/local/.
[root@LAN_redis1 ~]# cd /usr/local/redis-3.2.1/
[root@LAN_redis1 redis-3.2.1]# make
[root@LAN_redis1 redis-3.2.1]# src/redis-server
4772:C 20 Jun 18:43:42.681 # Warning: no config file specified, using the default config. In order to specify a config file use src/redis-server /path/to/redis.conf
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.2.1 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 4772
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

4772:M 20 Jun 18:43:42.683 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
4772:M 20 Jun 18:43:42.683 # Server started, Redis version 3.2.1
4772:M 20 Jun 18:43:42.683 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
4772:M 20 Jun 18:43:42.683 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
4772:M 20 Jun 18:43:42.683 * The server is now ready to accept connections on port 6379
这里提醒：Ctrl+C 退出 以后执行：src/redis-server &
这样让服务都在后台执行。就不需要重新开启终端了。

[root@LAN_redis1 redis-3.2.1]# netstat -ntulp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address               Foreign Address             State       PID/Program name
tcp        0      0 0.0.0.0:6379                0.0.0.0:*                   LISTEN      4780/src/redis-serv
tcp        0      0 0.0.0.0:22                  0.0.0.0:*                   LISTEN      1381/sshd
tcp        0      0 127.0.0.1:25                0.0.0.0:*                   LISTEN      1461/master
tcp        0      0 :::6379                     :::*                        LISTEN      4780/src/redis-serv
tcp        0      0 :::22                       :::*                        LISTEN      1381/sshd
tcp        0      0 ::1:25                      :::*                        LISTEN      1461/master

这里我们都先关闭服务：

[root@LAN_redis1 redis-3.2.1]# /usr/local/redis-3.2.1/src/redis-cli -n 6379 shutdown

```
### redis master 配置：

``` bash 
创建日志目录
[root@LAN_redis1 redis-3.2.1]# mkdir /var/log/redis
[root@LAN_redis1 redis-3.2.1]# touch /var/log/redis/redis.log
创建数据目录
[root@LAN_redis2 redis-3.2.1]# mkdir -p /data/redis

找到配置文件/usr/local/redis-3.2.1/redis.conf
[root@LAN_redis1 redis-3.2.1]# vim redis.conf
修改如下内容：
#bind 127.0.0.1 # 注释
daemonize no 改为 yes # 是否后台运行
port 6379 这里端口我保持默认不做修改。
logfile "/var/log/redis/redis.log"  ## 添加日志保存绝对路径。
dir /data/redis/    # 数据目录 
masterauth "ghost+db@hz2016"
requirepass "ghost+db@hz2016" 设置密码
```


### slave:

``` bash 
[root@LAN_redis2 ~]# curl -O http://download.redis.io/releases/redis-3.2.1.tar.gz -C /tmp/.
[root@LAN_redis2 ~]# tar xzf redis-3.2.1.tar.gz -C /usr/local/.
[root@LAN_redis2 ~]# cd /usr/local/redis-3.2.1/
[root@LAN_redis2 redis-3.2.1]#  make
[root@LAN_redis2 redis-3.2.1]# src/redis-server
7263:C 20 Jun 18:46:14.140 # Warning: no config file specified, using the default config. In order to specify a config file use src/redis-server /path/to/redis.conf
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.2.1 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 7263
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

7263:M 20 Jun 18:46:14.142 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
7263:M 20 Jun 18:46:14.142 # Server started, Redis version 3.2.1
7263:M 20 Jun 18:46:14.142 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
7263:M 20 Jun 18:46:14.143 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
7263:M 20 Jun 18:46:14.143 * The server is now ready to accept connections on port 6379

这里提醒：Ctrl+C 退出 以后执行：src/redis-server &
这样让服务都在后台执行。就不需要重新开启终端了。

测试：
[root@LAN_redis2 redis-3.2.1]# src/redis-cli
127.0.0.1:6379> set foo bar
OK
127.0.0.1:6379> get foo
"bar"
127.0.0.1:6379>

[root@LAN_redis2 redis-3.2.1]# netstat -ntulp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address               Foreign Address             State       PID/Program name
tcp        0      0 0.0.0.0:6379                0.0.0.0:*                   LISTEN      7266/src/redis-serv
tcp        0      0 0.0.0.0:22                  0.0.0.0:*                   LISTEN      3810/sshd
tcp        0      0 127.0.0.1:25                0.0.0.0:*                   LISTEN      1363/master
tcp        0      0 :::6379                     :::*                        LISTEN      7266/src/redis-serv
tcp        0      0 :::22                       :::*                        LISTEN      3810/sshd
tcp        0      0 ::1:25                      :::*                        LISTEN      1363/master
```

### redis slave 配置：

``` bash 
创建日志目录
[root@LAN_redis2 redis-3.2.1]# mkdir /var/log/redis
[root@LAN_redis2 redis-3.2.1]# touch /var/log/redis/redis.log
创建数据目录
[root@LAN_redis2 redis-3.2.1]# mkdir -p /data/redis
找到配置文件/usr/local/redis-3.2.1/redis.conf

[root@LAN_redis2 redis-3.2.1]# vim redis.conf
修改如下内容：
#bind 127.0.0.1 # 注释
daemonize no 改为 yes # 是否后台运行
port 6379 这里端口我保持默认不做修改。
logfile "/var/log/redis/redis.log"  ## 添加日志保存绝对路径。
dir /data/redis/    # 数据目录 
slaveof  192.168.1.173 6379  ## slaveof master的ip master的端口
masterauth "ghost+db@hz2016"
requirepass "ghost+db@hz2016"  设置密码


```
###配置附件：剩下的后面默认就好。master 按这个 slave 记得添加：slaveof  192.168.1.173 6379  master主机和端口
这里主从我设置了密码：master和slave 都需要设置密码，不然同步不成功。
不成功报错日志：
``` bash 
8395:S 21 Jun 14:09:22.073 * MASTER <-> SLAVE sync started
8395:S 21 Jun 14:09:22.073 # Error condition on socket for SYNC: Connection refused
```


``` bash
protected-mode yes  
port 6379        # 监听端口
tcp-backlog 511   # TCP接收队列长度，受/proc/sys/net/core/somaxconn和tcp_max_syn_backlog这两个内核参数的影响
timeout 0         ## 一个客户端空闲多少秒后关闭连接(0代表禁用，永不关闭)
tcp-keepalive 300
daemonize yes     # 守护进程模式
supervised no
pidfile /var/run/redis_6379.pid
masterauth "ghost+db@hz2016"  # 当master服务设置了密码保护时，slav服务连接master的密码
loglevel notice
logfile "/var/log/redis/redis.log"  # 指明日志文件名
databases 16
save 900 1 ## 900秒内有1次修改将会保存
save 300 10 ## 300秒内有10次修改将会保存
save 60 10000 ## 60秒内有10000次修改保存
stop-writes-on-bgsave-error yes  # 默认如果开启RDB快照(至少一条save指令)并且最新的后台保存失败，Redis将会停止接受写操作
# 这将使用户知道数据没有正确的持久化到硬盘，否则可能没人注意到并且造成一些灾难
rdbcompression yes ## 开启rdb压缩
rdbchecksum yes  # 版本5的RDB有一个CRC64算法的校验和放在了文件的最后。这将使文件格式更加可靠。
dbfilename dump.rdb ## DB名字
dir  /data/redis/    ## 工作目录，一定要指定
requirepass "ghost+db@hz2016" 设置redis访问的密码 # 密码验证
# 警告：因为Redis太快了，所以外面的人可以尝试每秒150k的密码来试图破解密码。这意味着你需要
# 一个高强度的密码，否则破解太容易了
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes

```

### 启动master：

```
[root@LAN_redis1 redis-3.2.1]# /usr/local/redis-3.2.1/src/redis-server /usr/local/redis-3.2.1/redis.conf &
[1] 5738
[root@LAN_redis1 redis-3.2.1]#
[1]+  Done                    /usr/local/redis-3.2.1/src/redis-server /usr/local/redis-3.2.1/redis.conf
[root@LAN_redis1 redis-3.2.1]#
[root@LAN_redis1 redis-3.2.1]#
[root@LAN_redis1 redis-3.2.1]# netstat -ntulp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address               Foreign Address             State       PID/Program name
tcp        0      0 192.168.1.173:10050         0.0.0.0:*                   LISTEN      1479/zabbix_agentd
tcp        0      0 0.0.0.0:6379                0.0.0.0:*                   LISTEN      5739/redis-server *
tcp        0      0 0.0.0.0:22                  0.0.0.0:*                   LISTEN      1381/sshd
tcp        0      0 127.0.0.1:25                0.0.0.0:*                   LISTEN      1461/master
tcp        0      0 :::6379                     :::*                        LISTEN      5739/redis-server *
tcp        0      0 :::22                       :::*                        LISTEN      1381/sshd
tcp        0      0 ::1:25                      :::*                        LISTEN      1461/master

```

### 启动 slave ：

``` bash
[root@LAN_redis2 redis-3.2.1]#/usr/local/redis-3.2.1/src/redis-server /usr/local/redis-3.2.1/redis.conf &
[1]+  Done                    /usr/local/redis-3.2.1/src/redis-server /usr/local/redis-3.2.1/redis.conf
 
[root@LAN_redis2 redis-3.2.1]# netstat -ntulp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address               Foreign Address             State       PID/Program name
tcp        0      0 192.168.1.174:10050         0.0.0.0:*                   LISTEN      3998/zabbix_agentd
tcp        0      0 0.0.0.0:6379                0.0.0.0:*                   LISTEN      8192/redis-server *
tcp        0      0 0.0.0.0:22                  0.0.0.0:*                   LISTEN      3810/sshd
tcp        0      0 127.0.0.1:25                0.0.0.0:*                   LISTEN      1363/master
tcp        0      0 :::6379                     :::*                        LISTEN      8192/redis-server *
tcp        0      0 :::22                       :::*                        LISTEN      3810/sshd
tcp        0      0 ::1:25                      :::*                        LISTEN      1363/master
```

### 主从测试是否同步：

master :
``` bash 
[root@LAN_redis1 redis-3.2.1]# /usr/local/redis-3.2.1/src/redis-cli
127.0.0.1:6379>
127.0.0.1:6379> set name test
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth ghost+db@hz2016
OK
127.0.0.1:6379> set name test
OK
127.0.0.1:6379> save
OK
```
slave:
``` bash
[root@LAN_redis2 redis-3.2.1]# /usr/local/redis-3.2.1/src/redis-cli
127.0.0.1:6379>
127.0.0.1:6379>
127.0.0.1:6379> get aa
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth ghost+db@hz2016
OK
127.0.0.1:6379> get aa
"123"
127.0.0.1:6379>
127.0.0.1:6379> get name
"test" 
```
### 查看主从关系：
``` bash 
[root@LAN_redis1 etc]#  /usr/local/redis-3.2.1/src/redis-cli -a ghost+db@hz2016 -p 6379 info Replication
# Replication
role:master
connected_slaves:2
slave0:ip=192.168.1.174,port=6379,state=online,offset=57,lag=1
slave1:ip=192.168.1.172,port=6379,state=online,offset=43,lag=1
master_repl_offset:57
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:56
```

### 安装Redis Sentinel配置 
Sentinel是一个管理多个redis实例的工具，它可以实现对redis的监控、通知、自动故障转移。

sentinel不断的检测redis实例是否可以正常工作，通过API向其他程序报告redis的状态，如果redis master不能工作，则会自动启动故障转移进程，将其中的一个slave提升为master，其他的slave重新设置新的master实例。也就是说，它提供了：

监控（Monitoring）： Sentinel 会不断地检查你的主实例和从实例是否正常。
通知（Notification）： 当被监控的某个 Redis 实例出现问题时， Sentinel 进程可
以通过 API 向管理员或者其他应用程序发送通知。

自动故障迁移（Automatic failover）： 当一个主redis实例失效时， Sentinel 会开始记性一次failover， 它会将失效主实例的其中一个从实例升级为新的主实例， 并让失效主实例的其他从实例改为复制新的主实例； 而当客户端试图连接失效的主实例时， 集群也会向客户端返回新主实例的地址， 使得集群可以使用新主实例代替失效实例。
Redis Sentinel自身也是一个分布式系统， 你可以在一个架构中运行多个 Sentinel 进程， 这些进程使用流言协议（gossip protocols)来接收关于主Redis实例是否失效的信息， 然后使用投票协议来决定是否执行自动failover，以及评选出从Redis实例作为新的主Redis实例。

同样在192.168.1.172 安装最新版的3.2.1 redis
### 启动sentinel的方法
当前Redis stable版已经自带了redis-sentinel这个工具。虽然 Redis Sentinel 已经提供了一个单独的可执行文件 redis-sentinel ， 但实际上它只是一个运行在特殊模式下的 Redis实例， 你可以在启动一个普通 Redis实例时通过给定 –sentinel 选项来启动 Redis Sentinel 实例。也就是说：
             
    redis-sentinel /etc/sentinel.conf
等同于
     
    redis-server /etc/sentinel.conf --sentinel

其中sentinel.conf是redis的配置文件，Redis sentinel会需要写入配置文件来保存sentinel的当前状态。当配置文件无法写入时，Sentinel启动失败。并正确设置防火墙.

### sentinel部署须知
一个稳健的redis集群，应该使用至少三个sentinel实例，并且保证讲这些实例放到不同的机器上，甚至不同的物理区域。
sentinel无法保证强一致性, 大概分布式环境都会有这方面的权衡.
确保client库支持sentinel.
sentinel需要通过不断的测试和观察，才能保证高可用。
sentinel配合docker使用时，要注意端口映射带来的影响.



``` bash 
安装好了redis 
[root@sentinel redis-3.2.1]# cp /usr/local/redis-3.2.1/sentinel.conf /etc/sentinel.conf
[root@LAN_redis2 redis-3.2.1]# cp /usr/local/redis-3.2.1/sentinel.conf /etc/sentinel.conf
[root@LAN_redis1 redis-3.2.1]# cp /usr/local/redis-3.2.1/sentinel.conf /etc/sentinel.conf
机器所有都拷贝一份

配置redis.conf 跟主从配置相同，然后启动。

```
      
      [root@sentinel redis-3.2.1]# touch /var/log/redis/sentinel.log
      [root@sentinel redis-3.2.1]# touch /var/run/sentinel.pid
配置:

``` bash
port 26379
daemonize yes
logfile "/var/log/redis/sentinel.log"
pidfile "/var/run/sentinel.pid"
dir "/tmp"

#Master 6379
sentinel monitor mymaster 192.168.1.173 6379 2    #配置master名、ip、port、需要多少个sentinel才能判断[客观下线]（2）
sentinel down-after-milliseconds mymaster 5000 ##配置sentinel向master发出ping，最大响应时间、超过则认为主观下线 默认1s检测一次，这里配置超时5000毫秒为宕机。
sentinel parallel-syncs mymaster 10000         ## #配置在进行故障转移时，运行多少个slave进行数据备份同步(越少速度越快)
sentinel auth-pass mymaster redis4Hz@2015
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

其他两台一样配置。
启动3个sentinel节点。

``` bash
[root@LAN_redis1 etc]# /usr/local/redis-3.2.1/src/redis-sentinel /etc/sentinel.conf  --sentinel
```
查看节点是否都已经启动：

``` bash
[root@LAN_redis1 etc]#  ps -ef | grep redis-sentinel
root      5992     1  0 15:06 ?        00:00:00 /usr/local/redis-3.2.1/src/redis-sentinel *:26379 [sentinel]
root      5996  5574  0 15:07 pts/1    00:00:00 grep redis-sentinel
```
然后查看日志：

``` bash 
[root@sentinel redis-3.2.1]# tailf /var/log/redis/sentinel.log
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

9541:X 21 Jun 15:05:15.398 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
9541:X 21 Jun 15:05:15.406 # Sentinel ID is c1143e64b85a9ce321d4b33c5b463b7958477f37
9541:X 21 Jun 15:05:15.406 # +monitor master mymaster 192.168.1.173 6379 quorum 2
9541:X 21 Jun 15:05:20.406 # +sdown master mymaster 192.168.1.173 6379
```
这里的ID 在看看/etc/sentinel 的配置。

master：配置

``` bash 
port 26379
daemonize yes
logfile "/var/log/redis/sentinel.log"
pidfile "/var/run/sentinel.pid"
dir "/tmp"
#Master 6379
sentinel monitor mymaster 192.168.1.173 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel auth-pass mymaster redis4Hz@2015
sentinel config-epoch mymaster 2919
sentinel leader-epoch mymaster 2919
sentinel known-slave mymaster 192.168.1.174 6379
# Generated by CONFIG REWRITE
sentinel known-sentinel mymaster 192.168.1.174 26379 a3a6df103a7acaa56e4e6e9af63808dc9b525d1e
sentinel known-sentinel mymaster 192.168.1.172 26379 c1143e64b85a9ce321d4b33c5b463b7958477f37
sentinel current-epoch 2926
```
slave :配置：

``` bash 
port 26379
daemonize yes
logfile "/var/log/redis/sentinel.log"
pidfile "/var/run/sentinel.pid"
dir "/tmp"

#Master 6379
sentinel monitor mymaster 192.168.1.173 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel auth-pass mymaster redis4Hz@2015
sentinel config-epoch mymaster 2919
sentinel leader-epoch mymaster 2918
sentinel known-slave mymaster 192.168.1.174 6379
# Generated by CONFIG REWRITE
sentinel known-sentinel mymaster 192.168.1.173 26379 ec452bf207d6cc720d3289180f66e9fa5a945f67
sentinel known-sentinel mymaster 192.168.1.172 26379 c1143e64b85a9ce321d4b33c5b463b7958477f37
sentinel current-epoch 2926
```
sentinel 配置：

```
port 26379
daemonize yes
logfile "/var/log/redis/sentinel.log"
pidfile "/var/run/sentinel.pid"
dir "/tmp"

#Master 6379
sentinel monitor mymaster 192.168.1.173 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel auth-pass mymaster redis4Hz@2015
sentinel config-epoch mymaster 2919
sentinel leader-epoch mymaster 2926
sentinel known-slave mymaster 192.168.1.173 6379
# Generated by CONFIG REWRITE
sentinel known-sentinel mymaster 192.168.1.174 26379 a3a6df103a7acaa56e4e6e9af63808dc9b525d1e
sentinel known-sentinel mymaster 192.168.1.173 26379 ec452bf207d6cc720d3289180f66e9fa5a945f67
sentinel current-epoch 2926
```

然后在重新启动一遍。

然后在查看下每台日志：

``` bash 
master :
[root@LAN_redis2 redis-3.2.1]# tailf /var/log/redis/sentinel.log
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

8510:X 21 Jun 15:36:41.973 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
8510:X 21 Jun 15:36:41.974 # Sentinel ID is 59bd001bd3a719a2a53669db766ff6caf93cca33
8510:X 21 Jun 15:36:41.974 # +monitor master mymaster 192.168.1.173 6379 quorum 2
8510:X 21 Jun 15:36:46.999 # +sdown master mymaster 192.168.1.173 6379
8510:X 21 Jun 15:36:47.000 # +sdown sentinel c1143e64b85a9ce321d4b33c5b463b7958477f37 192.168.1.172 26379 @ mymaster 192.168.1.173 6379
8510:X 21 Jun 15:36:47.000 # +sdown sentinel ec452bf207d6cc720d3289180f66e9fa5a945f67 192.168.1.173 26379 @ mymaster 192.168.1.173 6379

slave :
master:

[root@LAN_redis2 redis-3.2.1]# tailf /var/log/redis/sentinel.log
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

8510:X 21 Jun 15:36:41.973 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
8510:X 21 Jun 15:36:41.974 # Sentinel ID is 59bd001bd3a719a2a53669db766ff6caf93cca33
8510:X 21 Jun 15:36:41.974 # +monitor master mymaster 192.168.1.173 6379 quorum 2
8510:X 21 Jun 15:36:46.999 # +sdown master mymaster 192.168.1.173 6379
8510:X 21 Jun 15:36:47.000 # +sdown sentinel c1143e64b85a9ce321d4b33c5b463b7958477f37 192.168.1.172 26379 @ mymaster 192.168.1.173 6379
8510:X 21 Jun 15:36:47.000 # +sdown sentinel ec452bf207d6cc720d3289180f66e9fa5a945f67 192.168.1.173 26379 @ mymaster 192.168.1.173 6379

sentinel:
[root@sentinel redis-3.2.1]# tailf /var/log/redis/sentinel.log
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

9590:X 21 Jun 15:36:36.222 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
9590:X 21 Jun 15:36:36.223 # Sentinel ID is 54add3cbee8adc68b5b89f808a560a7425fd5166
9590:X 21 Jun 15:36:36.223 # +monitor master mymaster 192.168.1.173 6379 quorum 2
9590:X 21 Jun 15:36:41.262 # +sdown master mymaster 192.168.1.173 6379
9590:X 21 Jun 15:36:41.262 # +sdown sentinel a3a6df103a7acaa56e4e6e9af63808dc9b525d1e 192.168.1.174 26379 @ mymaster 192.168.1.173 6379
9590:X 21 Jun 15:36:41.262 # +sdown sentinel ec452bf207d6cc720d3289180f66e9fa5a945f67 192.168.1.173 26379 @ mymaster 192.168.1.173 6379
```

日志我先开着，然后我们关闭master
这里我们都先关闭服务：

``` bash 
master:

[root@LAN_redis1 etc]#  /usr/local/redis-3.2.1/src/redis-cli -a ghost+db@hz2016 -n 6379 shutdown

然后在去查看sentinel 机器信息：
[root@sentinel redis-3.2.1]#  /usr/local/redis-3.2.1/src/redis-cli -a ghost+db@hz2016 -p 6379 info Replication
# Replication
role:slave
master_host:192.168.1.174
master_port:6379
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:0
slave_repl_offset:1
master_link_down_since_seconds:1466495392
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```
这里master 已经主观转移

``` bash 
# Replication
role:slave
master_host:192.168.1.174
master_port:6379
master_link_status:up
master_last_io_seconds_ago:-1
master_sync_in_progress:0
slave_repl_offset:673
master_link_down_since_seconds:49
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

防火墙配置：

``` bash 
-A INPUT -p tcp -m state --state NEW -m tcp --dport 6379 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 26379 -j ACCEPT

```

### 启动Sentinel

使用--sentinel参数启动，并必须指定一个对应的配置文件，系统会使用配置文件来保存 Sentinel 的当前状态， 并在 Sentinel 重启时通过载入配置文件来进行状态还原。

    redis-server /etc/sentinel.conf --sentinel

使用TCP端口26379，可以使用redis-cli或其他任何客户端与其通讯。

如果启动 Sentinel 时没有指定相应的配置文件， 或者指定的配置文件不可写（not writable）， 那么 Sentinel 会拒绝启动。



这里提个说明：


### 外网访问连不上解错误分析及解决办法

错误的原因很简单，就是没有连接上redis服务，由于redis采用的安全策略，默认会只准许本地访问。需要通过简单配置，完成允许外网访问。
修改redis的配置文件，将所有bind信息全部屏蔽。

``` bash 
# bind 192.168.1.100 10.0.0.1
# bind 192.168.1.8
# bind 127.0.0.1
```
修改完成后，需要重新启动redis服务。


### 补充：

修改 Linux 的防火墙(iptables)，开启你的redis服务端口，默认是6379
-A INPUT -p tcp -m state --state NEW -m tcp --dport 6379 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 26379 -j ACCEPT

请注意，一定要将redis的防火墙配置放在 REJECT 的前面。然后执行 service iptables restart
至此，访问刚刚上面的代码，就能够链接到redis服务，并且能够正确显示了。

* 可通过info 查看当前redis的状态，默认采用info default, 可采用info all 查看所有的信息状态；
* 通过 “info cpu” 单独查看cpu的状态信息，"info memory" "info replication" "info ..."
