---
title: Redis学习笔记（下）
date: 2018-12-06 11:00:02
tags:
cover: https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/ninos-jugando-diferentes-instrumentos-musicales_1639-3289.jpg
---

## 持久化

### 什么是持久化

Redis的所有数据都保存在内存中，持久化会对数据的更新将异步地保存到磁盘上。

持久化方式

* 快照 (RDB)
* 写日志 (AOF)

### RDB

#### 什么是RDB

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181130145409.png)

#### RDB的三种触发方式

* save(同步)
* bgsave(异步)
* 自动生成

1. save 命令

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181205151001.png)

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181205151717.png)

因为 save 命令是同步执行的，所以在数据量比较大时会造成Redis的阻塞。

2. bgsave 命令

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181205152555.png)

文件策略和复杂度与save命令相同。

3. 自动生成RDB

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20181206094201.png)

通过配置来自动生成RDB。如果在xx秒内，改变了x条数据，就自动生成RDB文件。

最佳配置
![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181206102912.png)


#### 其他触发RDB方式

* 全量复制 (主从复制时，主会自动生成RDB文件)
* debug reload
* shutdown (参数save)
   
#### RDB 总结

* RDB是Redis内存到硬盘的快照，用于持久化。
* save通常会阻塞Redis。
* bgsave不会阻塞Redis，但是会fork新进程。
* save自动配置只要满足任一条件就会被执行，但不建议使用。
* 有些触发机制不容忽视。
* info memory 查看redis内存使用情况。


### AOF

#### RDB现存问题

* 耗时、耗性能

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181218094229.png)


* 不可控，容易丢失数据

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181218094416.png)


#### 什么是AOF

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181218094802.png)

执行Redis命令时会以AOF文件的格式写一条到AOF文件中。

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181218095350.png)

数据恢复可以通过AOF文件载入到Redis中。


#### AOF的三种策略

* always

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181218102355.png)

* everysec（默认值）

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181218102415.png)

* no

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181218102456.png)


优缺点

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181218102659.png)

通常采用默认值everysec。


#### AOF重写

由于AOF的文件策略可以将每个写入命令都写到AOF文件中，这样就会存在一个问题，随着时间的推移，这个AOF文件会越来越大，这样我们通过这个AOF文件来恢复数据就会非常的慢。为了解决这个问题，Redis提供了一个AOF重写的机制。

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181219092345.png)

其实AOF重写非常简单，就是把过期的，重复的，以及一些可以优化的命令都进行化简，达到如下目的：

* 减少磁盘占用量
* 加速恢复速度

AOF重写的两种方式：

* bgrewriteaof

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181219093903.png)

AOF重写针对的是对内存中的数据进行回溯重写成AOF文件，而不是对之前的AOF文件重写成新的AOF文件。

* AOF重写配置

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181219095242.png)

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181219095432.png)

AOF重写流程：

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181219100106.png)

AOF相关配置项：

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181219100412.png)


### RDB和AOF的抉择

#### RDB和AOF的比较

![](https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20181224111013.png)

#### RDB最佳策略

#### AOF最佳策略

#### 最佳策略
















