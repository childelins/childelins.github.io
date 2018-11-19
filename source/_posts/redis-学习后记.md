---
title: redis 学习后记
date: 2018-11-19 21:59:17
tags:
---

## 时间复杂度


| 命令 | 复杂度 |
| --- | --- |
| hget | O(1) |
| hset | O(1) |
| hdel | O(1) |
| hexists | O(1) |
| hlen | O(1) |
| hmget | O(n) |
| hmset | O(n) |



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