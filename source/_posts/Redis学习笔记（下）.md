---
title: Redis学习笔记（下）
date: 2018-12-06 11:00:02
tags:
cover: https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/ninos-jugando-diferentes-instrumentos-musicales_1639-3289.jpg
---

## 持久化

### 什么是持久化

Redis的所有数据都保存在内存中，持久化会对数据的更新将异步地保存到磁盘上。

### 持久化方式

* 快照 (RDB)
* 写日志 (AOF)

#### RDB

> 1. 什么是RDB

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181130145409.png)

> 2. RDB的三种触发方式

* save(同步)
* bgsave(异步)
* 自动生成

---
save 命令

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181205151001.png)

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181205151717.png)

因为 save 命令是同步执行的，所以在数据量比较大时会造成Redis的阻塞。

---
bgsave 命令

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181205152555.png)

文件策略和复杂度与save命令相同。

---
自动生成RDB

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20181206094201.png)

通过配置来自动生成RDB。如果在xx秒内，改变了x条数据，就自动生成RDB文件。


最佳配置
![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181206102912.png)


> 3. 其他触发RDB方式

* 全量复制 (主从复制时，主会自动生成RDB文件)
* debug reload
* shutdown (参数save)
   
---
   
> 4. RDB 总结

* RDB是Redis内存到硬盘的快照，用户持久化。
* save通常会阻塞Redis。
* bgsave不会阻塞Redis，但是会fork新进程。
* save自动配置只要满足任一条件就会被执行，但不建议使用。
* 有些触发机制不容忽视。
