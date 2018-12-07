---
title: Redis学习笔记（上）
date: 2018-11-06 14:53:06
tags:
cover: https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/14411799-ilustraci%C3%B3n-de-los-ni%C3%B1os-jugando-al-f%C3%BAtbol-sobre-un-fondo-blanco.jpg
---

Redis是单线程架构，一次只运行一条命令，不会同时运行两条命令。注意不要在生产环境上执行慢命令（如keys，flushall等）

单线程快的原因：

* 纯内存(**√**)
* 非阻塞IO
* 避免线程切换和竞态消耗


## 通用命令

1. keys [pattern] 遍历所有的key

```sh
127.0.0.1:6379> keys *
1) "laravel:tag:tymon.jwt:key"
2) "u:0:info"
3) "laravel:company:13:tokenValidSince"
4) "u:1821:info"
5) "c:13:info"
127.0.0.1:6379> keys u:*:info
1) "u:0:info"
2) "u:1821:info"
```

> keys 命令一般不在生产环境使用

2. dbsize 计算key的总数

```sh
127.0.0.1:6379> dbsize
(integer) 5
```

3. exists [key] 检查key是否存在

```sh
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> exists hello
(integer) 1
127.0.0.1:6379> del hello
(integer) 1
127.0.0.1:6379> exists hello
(integer) 0
```

4. del [key] 删除指定key-value

```sh
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> get hello
"world"
127.0.0.1:6379> del hello
(integer) 1
127.0.0.1:6379> get hello
(nil)
```

5. expire、ttl、persist

* expire [key] [seconds] key在seconds秒后过期

* ttl [key] 查看key剩余的过期时间

* persist [key] 去掉key的过期时间

```sh
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> expire hello 20
(integer) 1
127.0.0.1:6379> ttl hello
(integer) 18
127.0.0.1:6379> get hello
"world"
127.0.0.1:6379> ttl hello
(integer) 11
127.0.0.1:6379> ttl hello
(integer) -2
127.0.0.1:6379> get hello
(nil)
```

> -2代表key已经不存在了

```sh
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> expire hello 20
(integer) 1
127.0.0.1:6379> ttl hello
(integer) 17
127.0.0.1:6379> persist hello
(integer) 1
127.0.0.1:6379> ttl hello
(integer) -1
127.0.0.1:6379> get hello
"world"
```

> -1代表key存在，并且没有过期时间

6. type [key] 返回key的类型

```sh
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> type hello
string
127.0.0.1:6379> sadd myset 1 2 3
(integer) 3
127.0.0.1:6379> type myset
set
```

> type的几种返回值类型：string、hash、list、set、zset、none


## 字符串(string)

> 使用场景

* 缓存
* 计数器
* 分布式锁
* 等等

---

1. get、set

* get [key] 获取key对应的value
* set [key] [value] 设置key-value

```sh
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> get hello
"world"
```

2. incr、decr、incrby、decrby

* incr [key] key自增1，如果key不存在，自增后get(key)=1
* decr [key] key自减1，如果key不存在，自减后get(key)=-1
* incrby [key] [k] key自增k，如果key不存在，自增后get(key)=k
* decrby [key] [k] key自减k，如果key不存在，自减后get(key)=-k

```sh
127.0.0.1:6379> get number
(nil)
127.0.0.1:6379> incr number
(integer) 1
127.0.0.1:6379> get number
"1"
127.0.0.1:6379> incrby number 99
(integer) 100
127.0.0.1:6379> decr number
(integer) 99
127.0.0.1:6379> get number
"99"
127.0.0.1:6379> decrby number 99
(integer) 0
```

3. set、setnx、set xx、set ex

* set [key] [value] 不管key是否存在，都设置
* setnx [key] [value] key不存在时，才设置（添加）
* set [key] [value] [xx] key存在时，才设置（更新）
* set [key] [value] [ex] [second] 设置key的同时设置过期时间，set 和 expire 命令的组合

```sh
127.0.0.1:6379> exists php
(integer) 0
127.0.0.1:6379> set php good
OK
127.0.0.1:6379> setnx php bad
(integer) 0
127.0.0.1:6379> set php best xx
OK
127.0.0.1:6379> get php
"best"
127.0.0.1:6379> exists java
(integer) 0
127.0.0.1:6379> setnx java best
(integer) 1
127.0.0.1:6379> set java easy xx
OK
127.0.0.1:6379> get java
"easy"
127.0.0.1:6379> exists lua
(integer) 0
127.0.0.1:6379> set lua hehe xx
(nil)
127.0.0.1:6379> set go nice ex 10
OK
127.0.0.1:6379> ttl go
(integer) 8
127.0.0.1:6379> ttl go
(integer) -2
127.0.0.1:6379> get go
(nil)
```

4. mget、mset

* mget [key1] [key2]... 批量获取key
* mset [key1] [value1] [key2] [value2]... 批量设置key-value

```sh
127.0.0.1:6379> mset hello world java best php good
OK
127.0.0.1:6379> mget hello java php
1) "world"
2) "best"
3) "good"
```

5. getset、append、strlen

* getset [key] [newvalue] 设置新的value并返回旧的value
* append [key] [value] 将value追加到旧的value
* strlen [key] 返回字符串的长度（注意中文）
 
```sh
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> getset hello php
"world"
127.0.0.1:6379> append hello ",java"
(integer) 8
127.0.0.1:6379> get hello
"php,java"
127.0.0.1:6379> strlen hello
(integer) 8
127.0.0.1:6379> set hello "足球"
OK
127.0.0.1:6379> strlen hello
(integer) 6
```

6. incrbyfloat、getrange、setrange

* incrbyfloat [key] [value] 增加key对应的浮点值
* getrange [key] [start] [end] 获取字符串指定下标所有的值
* setrange [key] [index] [value] 设置指定下标所有对应的值

```sh
127.0.0.1:6379> incr counter
(integer) 1
127.0.0.1:6379> incrbyfloat counter 1.1
"2.1"
127.0.0.1:6379> get counter
"2.1"
127.0.0.1:6379> set hello javabest
OK
127.0.0.1:6379> getrange hello 0 2
"jav"
127.0.0.1:6379> setrange hello 4 p
(integer) 8
127.0.0.1:6379> get hello
"javapest"
```

## 哈希(hash)

1. hget、hset、hdel

* hget [key] [field]  获取hash key对应的field的value
* hset [key] [field] [value] 设置hash key对应的field的value
* hdel [key] [field] 删除hash key对应的field的value

```sh
127.0.0.1:6379> hset user:1:info age 23
(integer) 1
127.0.0.1:6379> hget user:1:info age
"23"
127.0.0.1:6379> hset user:1:info name kang
(integer) 1
127.0.0.1:6379> hgetall user:1:info
1) "age"
2) "23"
3) "name"
4) "kang"
127.0.0.1:6379> hdel user:1:info age
(integer) 1
127.0.0.1:6379> hgetall user:1:info
1) "name"
2) "kang"
```

2. hexists、hlen

* hexists [key] [field] 判断hash key是否有field
* hlen [key] 获取hash key field的数量

```sh
127.0.0.1:6379> hgetall user:1:info
1) "name"
2) "kang"
127.0.0.1:6379> hexists user:1:info name
(integer) 1
127.0.0.1:6379> hlen user:1:info
(integer) 1
```

3. hmget、hmset

* hmget [key] [field1] [field2] ... 批量获取hash key的一批field对应的值
* hmset [key] [field1] [value1] [field2] [value2] ... 批量设置hash key的一批field value

```sh
127.0.0.1:6379> hmset user:2:info name june age 18
OK
127.0.0.1:6379> hmget user:2:info name age
1) "june"
2) "18"
```

4. hgetall、hvals、hkeys

* hgetll [key] 返回hash key对应所有的field和value
* hvals [key] 返回hash key对应所有的value
* hkeys [key] 返回hash key对应所有的field

```sh
127.0.0.1:6379> hgetall user:1:info
1) "name"
2) "kaka"
3) "age"
4) "30"
5) "page"
6) "50"
127.0.0.1:6379> hvals user:1:info
1) "kaka"
2) "30"
3) "50"
127.0.0.1:6379> hkeys user:1:info
1) "name"
2) "age"
3) "page"
```

5. hsetnx、hincrby、hincybyfloat

* hsetnx [key] [field] [value] 设置hash key对应field的value（如果field已经存在，则失败）
* hincrby [key] [field] [count] hash key对应field的value自增count
* hincrbyfloat [key] [field] [count] hincyby浮点数版

```sh
127.0.0.1:6379> hsetnx user:1:info name haha
(integer) 0
127.0.0.1:6379> hsetnx user:1:info sex 1
(integer) 1
127.0.0.1:6379> hincrby user:1:info age 5
(integer) 35
127.0.0.1:6379> hsetnx user:1:info num 1.2
(integer) 1
127.0.0.1:6379> hincrbyfloat user:1:info num 2.2
"3.4"
127.0.0.1:6379> hgetall user:1:info
 1) "name"
 2) "kaka"
 3) "age"
 4) "35"
 5) "page"
 6) "50"
 7) "sex"
 8) "1"
 9) "num"
10) "3.4"
```

## 列表(list)

1、rpush、lpush、linsert

* rpush [key] [value1] [value2] ... 从列表右端插入值（1-N个）
* lpush [key] [value1] [value2] ... 从列表左端插入值（1-N个）
* linsert [key] [before|after] [value] [newValue] 在list指定的值前|后插入newValue

```sh
127.0.0.1:6379> rpush list1 a b c
(integer) 3
127.0.0.1:6379> lrange list1 0 -1
1) "a"
2) "b"
3) "c"
127.0.0.1:6379> lpush list2 a b c
(integer) 3
127.0.0.1:6379> lrange list2 0 -1
1) "c"
2) "b"
3) "a"
127.0.0.1:6379> linsert list1 before b java
(integer) 4
127.0.0.1:6379> linsert list1 after b php
(integer) 5
127.0.0.1:6379> lrange list1 0 -1
1) "a"
2) "java"
3) "b"
4) "php"
5) "c"
```

2. lpop、rpop、lrem

* lpop [key] 从列表左侧弹出一个item
* rpop [key] 从列表右侧弹出一个item
* lrem [key] [count] [value] 根据count值，从列表中删除所有value相等的项   
(1) count>0，从左到右，删除最多count个value相等的项   
(2) count<0，从右到左，删除最多Math.abs(count)个value相等的项   
(3) count=0，删除所有value相等的项

```sh
127.0.0.1:6379> lpop list1
"a"
127.0.0.1:6379> lrange list1 0 -1
1) "java"
2) "b"
3) "php"
4) "c"
127.0.0.1:6379> rpop list1
"c"
127.0.0.1:6379> lrange list1 0 -1
1) "java"
2) "b"
3) "php"
127.0.0.1:6379> lrem list1 0 b
(integer) 1
127.0.0.1:6379> lrange list1 0 -1
1) "java"
2) "php"
```

3. ltrim

* ltrim [key] [start] [end] 按照索引范围修剪列表

```sh
127.0.0.1:6379> rpush list3 a b c d e f
(integer) 6
127.0.0.1:6379> lrange list3 0 -1
1) "a"
2) "b"
3) "c"
4) "d"
5) "e"
6) "f"
127.0.0.1:6379> ltrim list3 1 4
OK
127.0.0.1:6379> lrange list3 0 -1
1) "b"
2) "c"
3) "d"
4) "e"
```

> 在一个大范围的list，如果直接执行del key，会阻塞运行，开销很大。这时可以用ltrim不断修剪范围，来达到删除的效果。

4. lrange、lindex、llen

* lrange [key] [start] [end] 获取列表指定索引范围所有item（包含end）
* lindex [key] [index] 获取列表指定索引的item
* llen [key] 获取列表长度

```sh
127.0.0.1:6379> lrange list3 0 -1
1) "b"
2) "c"
3) "d"
4) "e"
127.0.0.1:6379> lrange list3 0 2
1) "b"
2) "c"
3) "d"
127.0.0.1:6379> lindex list3 0
"b"
127.0.0.1:6379> lindex list3 -1
"e"
127.0.0.1:6379> llen list3
(integer) 4
```

5. lset

* lset [key] [index] [newValue] 设置列表指定索引值为newValue

```sh
127.0.0.1:6379> lset list3 1 php
OK
127.0.0.1:6379> lrange list3 0 -1
1) "b"
2) "php"
3) "d"
4) "e"
```

6. blpop、brpop

* blpop [key] [timeout] lpop阻塞版本，timeout是阻塞超时时间，timeout=0为永远不阻塞
* brpop [key] [timeout] rpop阻塞版本，timeout是阻塞超时时间，timeout=0为永远不阻塞

> 1. LPUSH + LPOP = Stack
> 2. LPUSH + RPOP = Queue
> 3. LPUSH + LTRIM = Capped Collection
> 4. LPUSH + BRPOP = Message Queue


## 集合(set)

1. sadd、srem

* sadd [key] [element] 向集合key添加element(如果element已经存在，添加失败)
* srem [key] [element] 将集合key中的element移除掉

```sh
127.0.0.1:6379> sadd set1 a b c d e
(integer) 5
127.0.0.1:6379> srem set1 d
(integer) 1
127.0.0.1:6379> smembers set1
1) "c"
2) "a"
3) "b"
4) "e"
```

2. scard、sismember、srandmember、spop、smembers

* scard [key] 集合中的元素个数
* sismember [key] [member] 元素在集合中是否存在
* srandmember [key] [count] 从集合中随机取出count个元素
* spop [key] 从集合中随机弹出一个元素
* smembers [key] 取出集合中的所有元素

```sh
127.0.0.1:6379> scard set1
(integer) 4
127.0.0.1:6379> sismember set1 a
(integer) 1
127.0.0.1:6379> srandmember set1 1
1) "b"
127.0.0.1:6379> srandmember set1 2
1) "b"
2) "e"
127.0.0.1:6379> smembers set1
1) "c"
2) "a"
3) "b"
4) "e"
127.0.0.1:6379> spop set1
"e"
127.0.0.1:6379> smembers set1
1) "a"
2) "b"
3) "c"
```

> 注意smembers获取的元素内容是无序的。如果是一个大范围的集合，直接使用smembers可能会阻塞运行，这时可以使用sscan来处理。

3. sdiff、sinter、sunion

* sdiff [key1] [key2] 集合间的差集
* sinter [key1] [key2] 集合间的交集
* sunion [key1] [key2] 集合间的并集

> sdiff|sinter|sunion + store destkey ...  将差集、交集、并集的结果保存在destkey中

```sh
127.0.0.1:6379> sadd user:1:follow it music his sports
(integer) 4
127.0.0.1:6379> sadd user:2:follow it news ent sports
(integer) 4
127.0.0.1:6379> sdiff user:1:follow user:2:follow
1) "music"
2) "his"
127.0.0.1:6379> sinter user:1:follow user:2:follow
1) "it"
2) "sports"
127.0.0.1:6379> sunion user:1:follow user:2:follow
1) "sports"
2) "it"
3) "news"
4) "music"
5) "ent"
6) "his"
```

## 有序集合(zset)

1. zadd、zrem

* zadd [key] [score] [element] ...  添加score和element
* zrem [key] [element] ... 删除元素

```sh
127.0.0.1:6379> zadd user:1:ranking 1 kris 91 mike 200 frank 220 chris
(integer) 4
127.0.0.1:6379> zadd user:1:ranking 225 tom
(integer) 1
127.0.0.1:6379> zrem user:1:ranking tom
(integer) 1
127.0.0.1:6379> zrange user:1:ranking 0 -1
1) "kris"
2) "mike"
3) "frank"
4) "chris"
```

2. zscore、zincrby、zcard

* zscore [key] [element] 返回元素的分数
* zincrby [key] [incrScore] [element] 增加或减少元素的分数
* zcard [key] 返回元素中的个数

```sh
127.0.0.1:6379> zscore user:1:ranking chris
"220"
127.0.0.1:6379> zincrby user:1:ranking 9 mike
"100"
127.0.0.1:6379> zcard user:1:ranking
(integer) 4
```

3. zrank、zrange、zrangebyscore

* zrank [key] [element] 获取排名(从小到大)
* zrange [key] [start] [end] [WITHSCORES] 获取指定范围内的元素，并带上分值
* zrangebyscore [key] [minScore] [maxScore] [WITHSCORES] 获取指定分数范围内的元素，并带上分值

```sh
127.0.0.1:6379> zrank user:1:ranking frank
(integer) 2
127.0.0.1:6379> zrange user:1:ranking 0 -1 WITHSCORES
1) "kris"
2) "1"
3) "mike"
4) "100"
5) "frank"
6) "200"
7) "chris"
8) "220"
127.0.0.1:6379> zrangebyscore user:1:ranking 90 210 withscores
1) "mike"
2) "100"
3) "frank"
4) "200"
```

4. zcount、zremrangebyrank、zremrangebyscore

* zcount [key] [minScore] [maxScore] 获取指定分数范围内的元素个数
* zremrangebyrank [key] [start] [end] 删除指定排名范围内的元素
* zremrangebyscore [key] [minScore] [maxScore] 删除指定分数范围内的元素

```sh
127.0.0.1:6379> zcount user:1:ranking 200 221
(integer) 2
127.0.0.1:6379> zremrangebyrank user:1:ranking 1 2
(integer) 2
127.0.0.1:6379> zrange user:1:ranking 0 -1 withscores
1) "kris"
2) "1"
3) "chris"
4) "220"
127.0.0.1:6379> zremrangebyscore user:1:ranking 1 100
(integer) 1
127.0.0.1:6379> zrange user:1:ranking 0 -1 withscores
1) "chris"
2) "220"
```

5. zunionstore、zinterstore

集合间操作


## 其他功能

### 慢查询

1. 生命周期

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181128095642.png)

> 两点说明：   
> (1) 慢查询发生在第3阶段   
> (2) 客户端超时不一定是慢查询，但慢查询是客户端超时的一个可能因素

2. 两个配置

(1) slowlog-max-len 固定长度   
(2) slowlog-log-slower-than 慢查询阙值(单位：微秒)

慢查询是一个**先进先出的队列**，简单的说就是一条命令在第3步的执行过程中被列入慢查询的范围内，它就会进入一个队列。这个队列是用redis的列表来实现的，而且这个队列是一个**固定长度**的。并且它是**保存在内存中**的，随着redis的重启而消失。

3. 配置方法

(1) 默认值

* slowlog-max-len = 128
* slowlog-log-slower-than = 10000

(2) 修改配置文件重启   
    在第一次redis配置时修改，生产环境不建议重启。

(3) 动态配置

```sh
127.0.0.1:6379> config set slowlog-max-len 1000
OK
127.0.0.1:6379> config set slowlog-log-slower-than 10000
OK
```

4. 三个命令

* slowlog get [n] 获取n条慢查询队列记录
* slowlog len 获取慢查询队列长度
* slowlog reset 清空慢查询队列

5. 运维经验

* slowlog-max-len 不要设置过小，通常设置1000左右
* slowlog-log-slower-than 不要设置过大，默认10ms，通常设置1ms
* 定期持久化慢查询


### pipeline

1. 什么是流水线

批量处理命令，1次pipeline(n条命令) = 1次网络时间 + n次命令时间

> redis命令的执行时间通常是微妙级别   
> pipeline其实就是为了解决n次网络时间耗时的问题

2. 使用建议

* 注意每次pipeline携带的数据量
* pipeline每次只能作用在一个Redis节点上
* M操作与pipeline的区别，mGet、mSet这些都是原子操作，pipeline会拆分执行，是非原子操作


### 发布订阅

1. 角色

* 发布者(publisher)
* 订阅者(subscriber)
* 频道(channel)

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181129105007.png)

2. 相关命令

* publish [channel] [message] 发布
* subscribe [channel] 订阅
* unsubscribe [channel] 取消订阅

```sh
127.0.0.1:6379> publish sohu:tv "hello world"
(integer) 0 # 订阅者个数
127.0.0.1:6379> subscribe sohu:tv
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "sohu:tv"
3) (integer) 1
127.0.0.1:6379> publish sohu:tv "cool"
(integer) 1
127.0.0.1:6379> subscribe sohu:tv
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "sohu:tv"
3) (integer) 1
1) "message"
2) "sohu:tv"
3) "cool"
127.0.0.1:6379> unsubscribe sohu:tv
1) "unsubscribe"
2) "sohu:tv"
3) (integer) 0
```

3. 其他命令

* psubscribe [pattern...] 订阅模式
* punsubscribe [pattern...] 退订指定的模式
* pubsub channels 列出至少有一个订阅者的频道
* pubsub numsub [channel...] 列出给定频道的订阅者数量
* pubsub numpat 列出被订阅模式的数量

4. 消息队列模型

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181129111720.png)

消息队列与发布订阅的区别：

* 发布订阅是发布消息后所有订阅者都可以收到
* 消息队列是发布消息后只有一个订阅者可以收到，需要自己实现(如基于列表的阻塞)


## 时间负责度

| 命令 | 时间复杂度 |
| -- | -- |
| keys | O(n) |
| dbsize | O(1) |
| del | O(1) |
| exists | O(1) |
| expire | O(1) |
| type | O(1) |
| get | O(1) |
| set | O(1) |
| incr | O(1) |
| decr | O(1) |
| incrby | O(1) |
| decrby | O(1) |
| setnx | O(1) |
| set xx | O(1) |
| set ex | O(1) |
| mget | O(n) |
| mset | O(n) |
| getset | O(1) |
| append | O(1) |
| strlen | O(1) |
| incrbyfloat | O(1) |
| getrange | O(1) |
| setrange | O(1) |
| hget | O(1) |
| hset | O(1) |
| hdel | O(1) |
| hexists | O(1) |
| hlen | O(1) |
| hmget | O(n) |
| hmset | O(n) |
| hgetall | O(n) |
| hvals | O(n) |
| hkeys | O(n) |
| hsetnx | O(1) |
| hincrby | O(1) |
| hincybyfloat | O(1) |
| rpush | O(1~n) |
| lpush | O(1~n) |
| linsert | O(n) |
| lpop | O(1) |
| rpop | O(1) |
| lrem | O(n) |
| ltrim | O(n) |
| lrange | O(n) |
| lindex | O(n) |
| llen | O(1) |
| lset | O(n) |
| blpop | O(1) |
| brpop | O(1) |
| sadd | O(1) |
| srem | O(1) |
| zadd | O(logN) |
| zrem | O(1) |
| zscore | O(1) |
| zincrby | O(1) |
| zcard | O(1) |
| zrange | O(log(n)+m) |
| zrangebyscore | O(log(n)+m) |
| zcount | O(log(n)+m) |
| zremrangebyrank | O(log(n)+m) |
