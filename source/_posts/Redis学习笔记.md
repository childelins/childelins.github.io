---
title: Redis学习笔记
date: 2018-11-06 14:53:06
tags:
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


## 字符串

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

## 哈希

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

## 列表




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

