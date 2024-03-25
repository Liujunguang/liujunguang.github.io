+++
title = 'Redis Handbook'
date = 2024-01-29T22:03:31+08:00
draft = false
+++

## 简介

[Redis](https://redis.io)
[github](https://github.com/redis/redis)
The open-source, In-memory data store used by millions of developers as a cache, vector database, document database, streaming engine, and message broker.

## 部署

### 1. docker 容器部署

[docker hub](https://hub.docker.com/_/redis)

1. 准备目录、配置文件
```shell script
mkdir -p /data/redis/data
cd /data/redis
wget -O 'redis.conf' 'http://download.redis.io/redis-stable/redis.conf'
```

2. 修改 `redis.conf` 默认配置
```text
#注释掉这部分，这是限制redis只能本地访问
bind 127.0.0.1

#默认yes，开启保护模式，限制为本地访问
protected-mode no

#默认no，改为yes意为以守护进程方式启动，可后台运行，除非kill进程，改为yes会使配置文件方式启动redis失败
databases 16

#输入本地redis数据库存放文件夹（可选）
dir ./

#redis持久化（可选）
appendonly yes

#配置redis访问密码
requirepass 密码
```

3. 启动
```shell script
docker run -p 6379:6379 \
--name redis -d --restart=always \
-v /data/redis/redis.conf:/etc/redis/redis.conf \
-v /data/redis/data:/data \
redis redis-server /etc/redis/redis.conf --appendonly yes
```

## 登录

通过 `redis-cli` 连接服务
```shell script
docker exec -it redis bash

redis-cli

# 如果是连接其它 redis 实例，可以指定 host port
redis-cli -h 127.0.0.1 -p 6379

# 操作前需要认证
> auth 密码

# ping, 检测 redis 服务是否启动
> ping
PONG
```


## 常用操作

### 配置
```shell script
# 查看所有 redis 配置
> config get *

# 查看 redis 库数量
> config get databases
1) "databases"
2) "16"

# 设置密码
config set requirepass <密码>

# 删除密码
config set requirepass ''
```

### 状态和统计信息
```shell script
> info

# 查看 cpu 统计信息
> info cpu

# 查看 key 统计信息
> info keyspace
db1:keys=2,expires=0,avg_ttl=0
```

### 库
```shell script
# 切换 1 号库
> SELECT 1
```

### set
```shell script
# 存放数据
redis> SET mykey "Hello"
"OK"
redis> GET mykey
"Hello"
redis> SET anotherkey "will expire in a minute" EX 60
"OK"
```
- EX seconds -- Set the specified expire time, in seconds (a positive integer).
- PX milliseconds -- Set the specified expire time, in milliseconds (a positive integer).
- EXAT timestamp-seconds -- Set the specified Unix time at which the key will expire, in seconds (a positive integer).
- PXAT timestamp-milliseconds -- Set the specified Unix time at which the key will expire, in milliseconds (a positive integer).
- NX -- Only set the key if it does not already exist.
- XX -- Only set the key if it already exists.
- KEEPTTL -- Retain the time to live associated with the key.
- GET -- Return the old string stored at key, or nil if key did not exist. An error is returned and SET aborted if the value stored at key is not a string.

### get
```shell script
redis> GET nonexisting
(nil)
redis> SET mykey "Hello"
"OK"
redis> GET mykey
"Hello"
```

### del
```shell script
# 删除 keys
# DEL key [key ...]
DEL mykey
```

## String（字符串）
`SET`、`GET`、`INCR` 等操作

## Hash（哈希表）
```shell script
# 设置 hash
# HSET key field value [field value ...]
> hset num int 123
> hset num float 3.14

# 获取单个
# HGET key field
> hget num int

# 获取所有 hash keys
> hkeys num
1) "int"
2) "float"

# 获取所有 hash keys values
> hgetall num
1) "int"
2) "123"
3) "float"
4) "3.14"
```

## List（列表）
```shell script
# 创建 List
# 将一个或多个值 value 插入到列表 key 的表头（从左边插入）
> LPUSH mylist a b c
(integer) 3

# 查询全部 List
> LRANGE mylist 0 -1
1) "c"
2) "b"
3) "a"

# 从右边插入
> RPUSH mylist a b c
> LRANGE mylist 0 -1
1) "a"
2) "b"
3) "c"

# 设置 List 对应 index 的值
# LSET key index value
> LSET mylist 0 cc
```

## Set（集合）
```shell script
# 创建集合
> SADD bbs "discuz.net"
(integer) 1

# 重复的元素添加，只能存在一个
> SADD bbs "discuz.net"
(integer) 0

> SADD bbs "tianya.cn" "groups.google.com"

# 查询
> SMEMBERS bbs
1) "discuz.net"
2) "groups.google.com"
3) "tianya.cn"
```

## SortedSet（有序集合）

## Redis 持久化

Redis 是一个内存数据库，数据保存在内存中，但我们知道，内存的数据变化很快，也容易丢失。Redis 提供了两种持久化方式：
RDB（Redis DataBase）和AOF（Append Only File）。

### 1. RDB

定时将内存中的数据快照保存到磁盘的一个压缩二进制文件中，是默认的持久化方式，默认的文件名为 dump.rdb。

修改 `redis.conf`
```toml
################################ SNAPSHOTTING  ################################

# Save the DB to disk
save 900 1      # 900秒内至少1个键被修改则触发保存
save 300 10     # 300秒内至少10个键被修改则触发保存
save 60 10000   # 60秒内至少10000个键被修改则触发保存
dbfilename dump.rdb  # RDB文件名
dir ./  # RDB文件存储目录
```

如果不需要持久化，那么你可以注释掉所有的 save 行来停用保存功能或设置为save "" 。
Redis在进行快照的过程中不会修改RDB文件，只有快照结束后才会将旧的文件替换成新的，也就是说任何时候RDB文件都是完整的。

#### 优点

- RDB 可以最大化 Redis 性能：在保存 RDB 文件时，会 fork 出一个子进程，执行接下来的保存工作，父进程不需要执行任何磁盘 I/O 操作。
- RDB 在恢复大数据集时，速度比 AOF 恢复速度快。

#### 缺点

- fork 大数据集时，可能比较耗时，造成服务器在一段时间内停止处理客户端请求。
- 子进程在存储快照期间，父进程的内存修改，不会反应在子进程中，所以快照期间的修改可能丢失数据。

### 2. AOF

每个写命令都通过append操作保存到文件中。在服务重启时，通过重放这些命令来恢复数据。

修改 `redis.conf`
```toml
appendonly yes  # 开启AOF
appendfilename "appendonly.aof"  # AOF文件名
appendfsync everysec  # 每秒同步一次至磁盘
# appendfsync always
# appendfsync no
```

#### AOF 保存模式

Redis目前支持三种AOF保存模式，它们分别是：不保存、每秒钟保存一次、每执行一个命令保存一次。

- AOF_FSYNC_NO（不保存）
- AOF_FSYNC_EVERYSEC（每一秒钟保存一次）
- AOF_FSYNC_ALWAYS（每执行一个命令保存一次（不推荐））：SAVE 是由 Redis 主进程执行的，所以 SAVE 期间，主进程会阻塞，不能接收命令请求。

### 3. 生产环境使用

注意：在实际生产环境中，应当根据数据的重要性和服务的可用性要求来选择合适的持久化策略。如果对数据持久性要求极高，应使用AOF，并配合适当的文件系统备份策略。如果对性能有较高要求，可以使用RDB，并适当调整保存快照的频率。
