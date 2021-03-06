# Redis

## 第1章 Redis 初识

+ 速度快，10Wops（每秒可以进行 10 万次读写）
+ 持久化  
  + Redis 所有数据保持在内存中，对数据的更新将异步地保存到磁盘上
  + RDB、AOF 两种方式
+ 多种数据结构
+ 支持多种客户端语言
+ 功能丰富
  + 发布订阅
  + 事务
  + lua 脚本
  + pipeline
+ 简单
  + 单机版本，23000 lines of code
  + 不依赖外部库（like libevent）
  + 单线程模型
+ 主从复制
  + 为高可用和分布式提供基础
+ 高可用、分布式
  + Redis-Sentinel（v2.8）支持高可用
  + Redis-CLuster（v3.0）支持分布式
+ 典型应用场景
  + 缓存系统
  + 计数器
  + 消息队列系统
  + 排行榜
  + 社交网络
  + 实时系统

### 1.1 Redis 安装

+ 所有 release 版本链接：http://download.redis.io/releases/
+ Redis 官网安装页面：https://redis.io/download
+ 具体安装步骤如下：
  + 下载：`wget http://download.redis.io/releases/redis-3.2.12.tar.gz`
  + 解压：`tar -xzf redis-3.2.12.tar.gz`
  + 建一个软连接（可选）：`ln -s redis-3.2.12 redis`
  + 进入 redis 目录：`cd redis`
  + 编译（会在 redis/src 目录下生成可执行文件）：`make`
  + 安装（会将 redis/src 目录下的可执行文件拷贝到 /user/local/bin 中）：`sudo make install`
+ 注意事项：
  + 通过 `echo $PATH` 命令，可以发现 `/usr/local/bin` 在环境变量中，因此执行完 `make install` 命令后，可以在任意目录下直接运行类似 `redis-server` 等可执行文件
  + 执行 `make isntall` 命令，需要 root 权限，这是因为 `/usr/local/bin` 目录只有 root 用户才拥有写权限，而拷贝文件到目录中，需要拥有对目录的写权限
  + 具体细节可以参看 `redis/src/Makefile`

### 1.2 Redis 可执行文件说明

安装完毕后，可以在 `reids/src` 或 `/usr/local/bin` 目录下看到 6 个可执行文件，如下：

+ redis-server：Redis 服务器
+ redis-cli：Redis 命令行客户端
+ redis-benchmark：Redis 性能测试工具
+ reids-check-aof：AOF 文件修复工具（断电可能会导致文件损坏）
+ redis-check-dump：RDB 文件修复工具
+ redis-sentinel：Redis 2.8 以后，提供了高可用版本，即 Sentinel 服务器

### 1.3 Redis 三种启动方法  

+ 最简启动
+ 动态参数启动
+ 配置文件启动
+ 比较

#### 1.3.1 最简启动 Reids

+ 编译安装后，直接执行 `redis-server`，使用 Redis 的默认配置进行启动
+ 验证启动的方法：
  + 查看进程：`ps -ef | grep redis`
  + 查看端口是否为 listen 状态：`netstat -antpl | grep redis`
  + 使用 `redis-cli -h ip -p port ping`

#### 1.3.2 动态参数启动 Redis

+ 例如：`redis-server --port 6380`
+ Redis 使用 6379 作为默认端口，如果想使用 6380 端口启动 Redis，就可以使用如上方式

#### 1.3.3 配置启动 Redis

+ 将需要的配置写在配置文件中
+ `redis-server configPath` 

#### 1.3.4 三种启动方式比较

+ 生产环境选择「配置启动」
  + Redis 是单线程模型，为了利用服务器的多核优势（资源的合理利用），通常会在一台机器上部署多个 Redis，此时使用默认启动或者动态参数启动较为麻烦
+ 单机多实例配置文件可以用端口区分开 

### 1.4 Redis 客户端连接

```bash
acjx@acjx:~$ redis-cli -h localhost -p 6379
localhost:6379> ping
PONG
localhost:6379> set hello world
OK
localhost:6379> get hello
"world"
```

### 1.5 Redis 客户端返回值

+ 状态回复

```bash
localhost:6379> ping
PONG
```

+ 错误回复
  + 以下例子使用获取哈希的命令执行在字符串的 key 上，返回错误

```bash
localhost:6379> hget hello field
(error) WRONGTYPE Operation against a key holding the wrong kind of value
```

+ 整数回复

```bash
localhost:6379> incr cnt
(integer) 1
```

+ 字符串回复

```bash
localhost:6379> get hello
"world"
```

+ 多行字符串回复

```bash
localhost:6379> mget hello foo
1) "world"
2) "bar"
```

### 1.6 Redis 常用配置

+ daemonize：是否以守护进程的方式启动
  + 默认配置是 no，即不以守护进程的方式启动
  + 建议使用 yes
  + 当使用 yes 时，启动日志会打印到配置的日志文件中
+ port：Redis 对外端口号，在单机多实例的情况下必须进行配置，默认端口是 6379
+ logfile：Redis 系统日志文件名，Reids 的工作情况以及发生的异常都会被记录在日志中
+ dir：Redis 工作目录，日志文件、持久化文件均会被存储在工作目录，注意上面的 logfile 只是文件名
+ 可以通过 `localhost:6379> config get *` 的方式查看有多少配置文件，均以 key-value 形式存储

## 第2章 API 的理解和使用

### 2.1 通用命令

通用命令针对任意数据结构，如下：

+ keys
  + keys 命令一般不在生产环境使用，因为这是一个 O(n) 的命令，而 Redis 是单线程的，如果数据量非常大（例如有100万条数据），这个命令会阻塞其他命令
  + keys * 怎么用？
    + 使用 `scan` 命令代替
    + 可以在「热备从节点」上执行一些比较重的命令，因为通常来说，从节点不会给生产环境去使用
+ dbsize：计算 key 的总数
  + O(1)
  + 可以在生产环境使用，因为 Redis 内部维护了一个计数器，会实时去维护 key 的总数，而不需要每次遍历才能得到
+ exists key：检查 key 是否存在
  + O(1)，可以在线上使用
  + 如果存在，返回 1 ，否则返回 0
+ del key [key ...]：删除指定的 key-value
  + 成功删除返回 1，否则返回 0
+ expire key seconds：给 key 设置秒级别的过期时间
  + 为 key 设置过期时间，经过所设定的时间后，key 会被自动删除
+ ttl key：查看 key 剩余的过期时间
  + 返回值如果大于等于 0，表示还剩多少过期时间
  + 返回值如果为 -1，说明 key 没有设置过期时间，永久存在
  + 返回值如果为 -2，说明 key 不存在，可能到了过期时间已被删除，或者原来就不存在
+ persist key：去掉 key 的过期时间
+ type key：返回 key 的类型

#### 2.1.1 时间复杂度

<img src="image/时间复杂度.jpg" width=360px/>

### 2.2 字符串键值结构

+ 使用场景
  + 缓存
  + 计数器
  + 分布式锁
+ API
  + get/set/del 
    + get key：获取 key 对应的 value，时间复杂度 O(1) 
    + set key value：设置 key-value，时间复杂度 O(1)
    + del key：删除 key-value，前面讲过，这是一个通用命令，时间复杂度 O(1)
  + incr/decr/incrby/decrby
    + incr key：key 自增 1，如果 key 不存在，自增后 get(key)=1，时间复杂度 O(1)
    + decr key：key 自减 1，如果 key 不存在，自减后 get(key)=-1，时间复杂度 O(1)
    + incrby key k：key 自增 k，如果 key 不存在，自增后 get(key)=k，时间复杂度 O(1)
    + decrby key k：key 自减 k，如果 key 不存在，自减后 get(key)=-k，时间复杂度 O(1)
  + set/setnx/set xx
    + set key value：不管 key 是否存在，都设置，时间复杂度 O(1)
    + setnx key value：key 不存在，才设置，可以理解为添加，时间复杂度 O(1)
    + set key value xx：key 存在，才设置，可以理解为更新，时间复杂度 O(1)
  + mget/mset
    + mget key1 key2 key3...：批量获取 key，原子操作，时间复杂度 O(n)
    + mset key1 value1 key2 value2...：批量设置 key-value，时间复杂度 O(n)
  + getset/append/strlen
    + getset key newvalue：set key newvalue 并返回旧的 value，时间复杂度 O(1)
    + append key value：将 value 追加到旧的 value，时间复杂度 O(1)
    + strlen key：返回字符串的长度（注意中文），时间复杂度 O(1) 
  + incrbyfloat/getrange/setrange
    + incrbyfloat key 3.5：增加 key 对应的值 3.5，注意，并没有提供浮点数自减的命令，当然可以传负值，达到自减的效果，时间复杂度 O(1)
    + getrange key start end：获取字符串指定下标所有的值，时间复杂度 O(1)
    + setrange key index value：设置指定下标所有对应的值，时间复杂度 O(1)

#### 2.2.1 例子

```bash
localhost:6379> get counter
(nil)
localhost:6379> incr counter
(integer) 1
localhost:6379> incrby counter 99
(integer) 100
localhost:6379> decr counter
(integer) 99
localhost:6379> get counter
"99"
```

```bash
localhost:6379> exists php
(integer) 0
localhost:6379> set php good          # 无论存在与否，都会设置
OK
localhost:6379> setnx php bad         # 添加失败，只有当不存在时才会添加
(integer) 0
localhost:6379> set php best xx       # 更新成功，存在才会更新
OK
localhost:6379> get php
"best"
localhost:6379> exists java
(integer) 0
localhost:6379> setnx java best       # 添加成功，不存在，顺利添加
(integer) 1
localhost:6379> set java easy xx      # 更新成功，存在才会更新
OK
localhost:6379> get java
"easy"
localhost:6379> exists lua
(integer) 0
localhost:6379> set lua hehe xx       # 更新失败，存在才会更新
(nil)
```

```bash
localhost:6379> mset hello world java best php good
OK
localhost:6379> mget hello java php
1) "world"
2) "best"
3) "good"
```

```bash
localhost:6379> set hello world
OK
localhost:6379> getset hello php           # 设置新值，返回旧值
"world"
localhost:6379> append hello ",java"       # 追加
(integer) 8
localhost:6379> get hello
"php,java"
localhost:6379> strlen hello               # 返回字符串的长度
(integer) 8
localhost:6379> set hello "足球"            # 中文使用 utf-8 存储，每个 word 占用 3 个字节
OK
localhost:6379> strlen hello
(integer) 6
```

```bash
localhost:6379> incr counter
(integer) 1
localhost:6379> incrbyfloat counter 1.1     # 自增一个浮点数
"2.1"
localhost:6379> get counter
"2.1"
localhost:6379> set hello cppbest           
OK
localhost:6379> getrange hello 0 2          # 获取字符串指定下标范围的值 
"cpp"
localhost:6379> get hello
"cppbest"
localhost:6379> setrange hello 3 g          # 将字符串下标为 3 的字符设置为‘g’（下标从0开始计算）
(integer) 7
localhost:6379> get hello
"cppgest"
localhost:6379> setrange hello 4 ood!       # 从字符串下标为 4 的位置开始，设置为 'ood!'
(integer) 8
localhost:6379> get hello
"cppgood!"
```


#### 2.2.2 n 次 `get` 与 `mget` 的比较

+ n 次 `get`

<img src="image/n次get.jpg" width=360px/>

+ 1 次 `mget`

<img src="image/1次mget.jpg" width=360px/>

#### 2.2.3 字符串常用命令小结

<img src="image/字符串总结.jpg" width=360px/>

### 2.3 哈希键值结构

<img src="image/哈希键值结构.jpg" width=360px/>

在哈希中可以为 key 直接添加一个新的值（field-value），这一点区别于字符串键值结构。

如果存储在字符串键值结构中，通常会将上述图中的用户信息封装成一个对象（无论使用什么语言），然后序列化成字符串的 value，存入 Redis 中；
如果需要添加属性的话，需要将 key 对应的 value 拿出来，然后反序列化成对象，添加完属性后，继续序列化成字符串的 value，存入 Redis 中。

<img src="image/哈希和表.jpg" width=360px/>

如上图所示，可以将 Redis 的「哈希键值结构」与关系数据库中的「表」联系起来，每个 key 连同其 value 相当于表中的一行，key 相当于 ID，
将 value 中的每个属性平铺开来，类似于表中的字段。区别在于「哈希键值结构」比较松散，比如，users:1 有 email 属性，而 users:2 可以没有这个属性。

#### 2.3.1 特点

+ Mapmap？
  + 大的 Map 是 key-value，而 value 中又包含了 field-value
+ Small redis
+ field 不能相同，value可以相同
  + 和 Redis 一样，key 不能相同，而 value 可以相同

#### 2.3.2 API

+ 所有哈希的命令均以 H 开头
+ hget/hset/hdel
  + hget key field：获取 hash key 对应的 field 的 value，O(1)
  + hset key field value：设置 hash key 对应 field 的 value，O(1)
  + hdel key field：删除 hash key 对应 field 的 value，O(1)
+ hexists/hlen
  + hexists key field：判断 hash key 是否有 field，O(1)
  + hlen key：获取 hash key field 的数量，O(1)
+ hmget/hmset
  + hmget key field1 field2...fieldN：批量获取 hash key 的一批 field 对应的值，O(n)
  + hmset key field1 value1 field2 value2...fieldN valueN：批量设置 hash key 的一批 field value，O(n)
+ hgetall/hvals/hkeys
  + htetall key：返回 hash key 对应所有的 field 和 value，O(n)
  + hvals key：返回 hash key 对应所有 field 的 value，O(n)
  + hkeys key：返回 hash key 对用所有 filed，O(n)
+ hsetnx/hincrby/hincybyfloat
  + hsetnx key field value：设置 hash key 对应 field 的 value（如果 field 已经存在，则失败），O(1)
  + hincrby key field intCounter：hash key 对应的 field 的 value 自增 intCounter，O(1)
  + hincrbyfloat key field floatCounter：hincrby 浮点数版，O(1)

#### 2.3.3 例子

+ hget/hset/hdel

```bash
127.0.0.1:6379> hset user:1:info age 23
(integer) 1
127.0.0.1:6379> hget user:1:info age
"23"
127.0.0.1:6379> hset user:1:info name ronaldo
(integer) 1
127.0.0.1:6379> hgetall user:1:info             # 获取所有属性的 field-value 值
1) "age"
2) "23"
3) "name"
4) "ronaldo"
127.0.0.1:6379> hdel user:1:info age
(integer) 1
127.0.0.1:6379> hgetall user:1:info
1) "name"
2) "ronaldo"
```

+ hexists/hlen

```bash
127.0.0.1:6379> hgetall user:1:info
1) "name"
2) "ronaldo"
127.0.0.1:6379> hexists user:1:info name
(integer) 1
127.0.0.1:6379> hlen user:1:info
(integer) 1
```

+ hmget/hmset

```bash
127.0.0.1:6379> hmset user:2:info age 30 name kaka page 50
OK
127.0.0.1:6379> hlen user:2:info
(integer) 3
127.0.0.1:6379> hmget user:2:info age name
1) "30"
2) "kaka"
```

#### 2.3.4 实战

+ 实现功能：记录网站每个用户个人主页的访问量
  + 回忆一下在字符串键值结构中是如何实现的？
    + 定义一个 key，例如 `user:1:pageview:count`
    + 这样的话，每个用户的个人主页访问量都作为一个 key
    + 然后使用 `incr` 或者 `incrby` 的方式
  + 在哈希键值结构中可以使用 `hincrby user:1:info pageview count`
    + 这样实现的话，是在用户信息这个 key 中，添加了一个 pageview 的属性
    + 整个 user，仍然是一个完整的整体
    + 而使用字符串的话，每个 user 的主页访问量都是单独一个 key

### 2.4 列表键值结构

+ 列表结构

<img src="image/列表键值结构.jpg" width=360px/>

+ 操作 1：LPUSH, LPOP, RPUSH, RPOP

<img src="image/列表操作1.jpg" width=360px/>

+ 操作 2：获取列表长度、删除某个元素、获取子列表、按照索引获取列表指定元素

<img src="image/列表操作2.jpg" width=360px/>

+ 特点
  + 有序
  + 可以重复
    + 列表元素可以是类似 a-b-a-a-b-a 这样的结构
  + 左右两边插入弹出

#### 2.4.1 API

+ 增：rpush/lpush/linsert
  + rpush key value1 value2 ... valueN：从列表右端插入值（1-N 个），O(1~n)
    + 例子：`rpush listkey c b a`，得到 c--b--a
  + lpush key value1 value2 ... valueN：从列表左端插入值（1-N 个），O(1~n)
    + 例子：`lpush listkey c b a`，得到 a--b--c
  + linsert key before|after value newValue：在列表指定的值前|后插入 newValue，O(n)
    + 例子：假设当前列表是 a--b--c--d，执行 `linsert listkey before b java`，得到 a--java--b--c--d
    + 参数 value，是指按照从左到右顺序，出现的第一个 value
+ 删：lpop/rpop/lrem/ltrim
  + lpop key：从列表左侧弹出一个 item，O(1)
    + 例子：假设当前列表是 a--b--c--d，执行 `lpop listkey`，得到 b--c--d
  + rpop key：从列表右侧弹出一个 item，O(1)
    + 例子：假设当前列表是 a--b--c--d，执行 `rpop listkey`，得到 a--b--c
  + lrem key count value：根据 count 的值，从列表中删除对应与 value 相等的 item，O(n)
    + count > 0，从左到右，删除最多 count 个与 value 相等的 item
    + count < 0，从右到左，删除最多 Math.abs(count) 个与 value 相等的 item
    + count = 0，删除所有与 value 相等的 item
    + 例子，假设当前列表为是 a--c--a--c--b--f
      + 执行 `lrem listkey 0 a`，得到 c--c--b--f
      + 继续执行 `lrem listkey -1 c`，得到 c--b--f（删除了最右边的 c）
  + ltrim key start end：按照索引范围修剪列表，O(n)
    + 例子，假设当前列表为 a--b--c--d--e--f，执行 `ltrim listkey 1 4`，得到 b--c--d--e
+ 查：lrange/lindex/llen
  + lrange key start end（包含 end）：获取列表指定索引范围所有 item，O(n)
  + lindex key index：获取列表指定索引的 item，O(n)
  + llen key：获取列表长度，O(1)
+ 改：lset
  + lset key index newValue：设置列表指定索引值为 newValue，O(n)
+ 不常用命令：blpop/brpop
  + blpop key timeout：lpop 阻塞版本，timeout 是阻塞超时时间，如果 timeout=0，如果列表中没有元素，会一直等待 
    + 当我们对一个空的列表执行 lpop 或者 rpop 时，通常会立即返回。如果在某些场景下，我们希望列表一有新的元素就立刻弹出，可以使用阻塞弹出，这在生产者消费者或者消息队列场景下是比较适用的
  + brpop key timeout：rpop 阻塞版本，timeout 是阻塞超时时间，如果 timeout=0，如果列表中没有原色，会一直等待
+ Tips
  + LPUSH + LPOP = Stack
  + LPUSH + RPOP = Queue
  + LPUSH + LTRIM = Capped Collection （控制列表大小）
  + LPUSH + BRPOP = Message Queue

### 2.5 集合键值结构

## 第3章 Redis 客户端的使用

## 第4章 瑞士军刀 Redis 其他功能

## 第5章 Redis 持久化的取舍和选择

## 第6章 常见的持久化开发运维问题

## 第7章 Redis 复制的原理与优化

## 第8章 Redis Sentinel

## 第9章 初识 Redis Cluster

## 第10章 深入 Redis Cluster

## 第11章 缓存设计与优化

## 第12章 Redis 云平台 CacheCloud