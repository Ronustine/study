[TOC]

## 安装与使用
```bash
# 下载
wget http://download.redis.io/releases/redis-6.0.4.tar.gz
# 解压
tar -zxvf redis-6.0.4.tar.gz
# 进到解压后的redis目录下编译
make
# 编译靠gcc，没有就安装
yum install gcc
# 编译完会在redis目录下生成一个src目录，命令都在这里

# 修改redis.conf配置：daemonize yes，启动时指定配置文件。这样才能使用后台启动
src/redis‐server redis.conf

# set get简单使用
set dx 2020
get dx
quit

# 杀服务
pkill redis‐server
kill pid
src/redis‐cli shutdown
```

## 常用数据结构使用
#### String
```bash
# 命令：单值
set key value
get key
# 命令：多个
mset key value [key2 value2 ...]
mget key [key2 ...]
# 命令：存入不存在的键才返回成功（1），否则失败（0）。即不允许覆盖
setnx key value
# 命令：删除
del key

# 应用：缓存对象方式，mset好，方便修改。set还要反序列化回来操作
set user:1 json化数据
mset user:1:name dx user:1:age 18
# 应用：分布式锁，是对setnx的使用。
# 当对一个商品的库存进行增减前，多个服务先使用setnx product:sore clientId
# 只有成功的才能继续操作，其余的自旋等待。操作完了记得del。详细使用在后面。

# 命令：原子加减
incr key
decr key
incrby key increment
decrby key decrement 

# 应用：阅读量计数器
incr article:readcount:{id}
# 应用：web Session共享
# 应用：分布式系统全局序列号，注意减压：需要批量获取。
```

#### Hash
类似map中再嵌套map
适合将一类数据归类在一起，注意超时只能对key生效。
```bash
# 命令：单个
hset key field value
hget key field
# 命令：多个
hmset key field value [field1 value1 ...]
hmget ket field [field1 ...]
# 命令：同setnx效果
hsetnx key field value
# 命令：大小
hlen key
# 命令：删除
hdel key
# 命令：获取全部内容
hgetall key
# 命令：为key某个增加增量increament
hinreby key field increament

# 应用：适合存一张表中的对象。注意不能太大，比如大量用户全放里面
hmset user 1:name dx 1:age 18
hmget user 1:name 1:age
```

#### List
可以结合API做出队列、栈等线性数据结构的效果
```bash
# 命令：左边压入
lpush key value1 [value2 ...]
# 命令：右边压入
rpush key value1 [value2 ...]
# 命令：左边弹出
lpop key
# 命令：右边弹出
rpop key
# 命令：从左开始start到stop，闭区间。注意没有从右开始。
lrange key start stop
# 命令：阻塞弹出，没有则登台timeou，如果timeout=0则一直阻塞。考虑到redis是单线程的，少用
blpop key [key2 ...] timeout
brpop key [key2 ...] timeout

# 索引的说明：从左开始数0开始，从右开始数-1开始。即左数数字-总数=右数数字
# 应用：微博的消息流。关注的大V发了消息后可以往每个用户专属的队列放一个消息id
lpush msg:userId msgId
lrange msg:userId 0 5
```

#### Set
```bash
# 命令：添加元素
sadd key member [member2 ...]
# 命令：移除元素
srem key member [member2 ...]
# 命令：获取key中的全部元素
smembers key
# 命令：获取key的数量
scard key
# 命令：是否存在
sismember key member
# 命令：随机抽出count个元素，不删除set中的内容
srandmember key count
# 命令：随机弹出count个元素，并从中删除
spop key count

# 应用：点赞，点赞人列表，是否点赞。同理 收藏、标签功能
# 应用：抽奖，使用随机抽出的命令即可
```

特别注意：集合运算
```bash
# 集合的操作
# 命令：交集
sinter key [key1 ...]
# 命令：并集
sunion key [key1 ...]
# 命令：差集
sdiff key [key2 ...]

# 应用：关注模型
# 共同关注，交集
# 我关注的人也关注了他，对我关注的每个人做sismember他
# 可能认识，差集，别人关注集合减去我的关注集合

# 应用：手机商品筛选
# 对品牌、操作系统、内存、cpu、分辨率等各维度信息建立set，并放入手机商品id，然后对选择了的条件做一个交集即可。
```

#### ZSet有序集合
```bash
# 命令：添加有分值的
zadd key score member [score1 member1 ...]
# 命令：移除
zrem key member [member1 ...]
# 命令：返回分值
zscore key member
# 命令：增加增量为increment的分值
zincrby key increment member
# 命令：个数
zscrad key
# 命令：分值的正序获取有序集合key从start下标到stop下标的元素。WITHSCORES指带上分值返回
ZRANGE key start stop [WITHSCORES]
# 命令：分值的倒序获取有序集合key从start下标到stop下标的元素。WITHSCORES指带上分值返回
ZREVRANGE key start stop [WITHSCORES]
# 命令：并集计算，新建一个destkey存放结果，numkeys指多少个key
ZUNIONSTORE destkey numkeys key [key ...]
# 命令：交集计算，新建一个destkey存放结果，numkeys指多少个key
ZINTERSTORE destkey numkeys key [key ...]

# 应用：微博热搜排行榜，添加带分值的member，点击量加1，获取前20个member
# 应用：百度7日榜单，对7天做并集运算，获取前n个member
```

#### 其他
`keys *`：获取全部键，建议改名，不能使用。使用`scan`命令
`scan cursor [MATCH pattern] [COUNT count]`：利用游标渐进式遍历。这里返回两个结果，第一个是游标（用于执行下一个scan命令的游标值），第二个是结果。
注意：count指的是redis单次遍历字典槽的数量，不是设置返回结果的数量
`info`：查看redis服务运行信息，分为 9 大块，每个块都有非常多的参数 
- Server 服务器运行的环境参数；
- Clients 客户端相关信息；
- Memory 服务器运行内存统计数据；
- Persistence 持久化信息；
- Stats 通用统计数据；
- Replication 主从复制相关信息；
- CPU CPU 使用情况；
- Cluster 集群信息；
- KeySpace 键值对统计数量信息；

## redis的性能
> 每秒10qps/s
- 执行是在内存上操作的，速度较快；
- 单线程，没有切换上下文耗费的性能，但正因为是单线程的，需要注意每条命令不能执行耗时的操作；
- 单线程如何并发处理客户端的连接：IO的多路复用。将连接信息和事件放在队列中，给到事件分派器分发到事件处理器，处理完毕再回调返回连接

redis可以配置最大并发处理客户端的数量：maxclients 10000

## redis持久化
#### RDB快照
默认生成dump.rdb的二进制数据。
- 配置文件：`save 900 1`（900s有一次变动，即查询不算）。每隔一定的时长，并执行了多少次命令就做一次快照。可以多个，是同时生效的。如果不要这种持久化注释即可。这是异步生成的。
- 客户端：执行`save`、`bgsave`生成快照，注意bgsave会fork一个子进程异步生成。

| 命令 | save | bgsave |
| :-- | :-- | :-- |
| IO类型 | 同步 | 异步 |
| 是否阻塞redis其它命令 | 是 | 否(在生成子进程执行调用fork函 数时会有短暂阻塞) |
| 复杂度 | O(n) | O(n) |
| 优点 | 不会消耗额外内存 | 不阻塞客户端命令 |
| 缺点 | 阻塞客户端命令 | 需要fork子进程，消耗内存 |

#### AOF
配置文件：appendonly yes
append-only file，每执行一条改变数据集的命令就追加至文件，启动时重新执行这些命令就可以恢复。
可以选择追加的策略：
- appendfsync always：每次有新命令追加到AOF文件时就执行一次fsync，非常慢，也非常安全；
- appendfsync everysec：每秒fsync一次，足够快（和使用RDB持久化差不多），并且在故障时只会丢失1秒钟的数据（推荐，兼顾性能与安全）；
- appendfsync no：从不fsync，将数据交给操作系统来处理。更快，也更不安全的选择；

aof文件的格式：（RESP命令）命令占的位数 字符串长度 字符串 [字符串长度 字符串]...（*3 $3 set $2 dx $2 18）

###### AOF的重写
> 是对一些命令的优化，比如执行了多次incr key，会优化成一条语句

可以配置AOF自动重写频率 
- `auto-aof-rewrite-min-size 64mb`：aof文件至少要达到64M才会自动重写，文件太小恢复速度本来就很快，重写的意义不大；
- `auto-aof-rewrite-percentage 100`：aof文件自上一次重写后文件大小增长了100%则再次触发重写；

客户端控制重写：`bgrewriteaof`（fork子进程）

#### 对比
| 命令 | RDB | AOF |
| :-- | :-- | :-- |
| 启动优先级 | 低 | 高 |
| 体积 | 小 | 大 |
| 恢复速度 | 快 | 慢 |
| 数据安全性 | 容易丢数据 | 根据策略决定 |

#### 混合持久化
Redis 4.0 为了解决这个问题，带来了一个新的持久化选项——混合持久化。配置开启混合持久化：`aof-use-rdb-preamble yes`
如果开启了混合持久化，AOF在重写时，将重写这一刻之前的内存做RDB快照处理，并且将RDB快照内容和增量的AOF修改内存数据的命令存在一起，都写入新的AOF文件，新的文件一开始不叫appendonly.aof，等到重写完新的AOF文件才会进行改名，原子的覆盖原有的AOF文件，完成新旧两个AOF文件的替换。
在 Redis 重启的时候，可以先加载RDB的内容，然后再重放增量AOF日志就可以完全替代之前的AOF全量文件重放，重启效率大幅得到提升。

## 主从
> 只是数据的复制，变更需要手动介入

#### 搭建
```bash
# 复制一份配置文件

# 如果端口有变化，配置文件修改
port 6380
pidfile /var/run/redis_6380.pid
logfile "6380.log"
dir /usr/local/redis‐5.0.3/data/6380

# 主从
replicaof 192.168.1.108 6379
replica‐read‐only yes

# 用配置文件启动
```

问题：
从节点连接报错 Error condition on socket for SYNC: Operation now in progress

#### 主从原理
全量复制
1. 从节点建立长连接，发送psync同步命令；
2. 主节点bgsave生成rdb文件，并缓存好这个时间段产生的写命令（repl_back_buffer），发送rdb；
3. 从节点接收rdb，并接收后续的写命令，合成完整的rdb，load入内存

部分复制
1. 从节点连接断开；
2. 从节点连接；
3. 从节点发送psync(offset)同步命令；
4. 主节点检查offset是否在repl_back_buffer能找到，如果能找到则将offset后面的命令发送给从节点，如果找不到，则全量复制；

## 哨兵