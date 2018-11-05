---
title: Docker 环境下的前后端分离部署与运维
date: 2018-10-18 22:34:35
tags:
cover: https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/44e818397558c3770b38c6978f35cbe7.jpg
---

# 安装 PXC 集群

1. 安装 PXC 镜像

```
docker pull percona/percona-xtradb-cluster:5.7.21
```

目前推荐安装 `5.7.21` 版本的 PXC 镜像，兼容性最好，在容器内可以执行apt-get安装各种程序包。最新版的PXC镜像内，无法执行apt-get，也就没法安装热备份工具了。


2. 重命名 PXC 镜像

```
docker tag percona/percona-xtradb-cluster:5.7.21 pxc
```

3. 创建 net1 网段

```
docker network create --subnet=172.18.0.0/16 net1
```

4. 创建5个数据卷

```
docker volume create v1
docker volume create v2
docker volume create v3
docker volume create v4
docker volume create v5
```

5. 创建备份数据卷（用于热备份数据）

```
docker volume create backup
```

6. 创建5节点的 PXC 集群

```sh
# 创建第1个MySQL节点
docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -v v1:/var/lib/mysql -v backup:/data --privileged --name=node1 --net=net1 --ip 172.18.0.2 pxc

#创建第2个MySQL节点
docker run -d -p 3307:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 -v v2:/var/lib/mysql -v backup:/data --privileged --name=node2 --net=net1 --ip 172.18.0.3 pxc

#创建第3个MySQL节点
docker run -d -p 3308:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 -v v3:/var/lib/mysql --privileged --name=node3 --net=net1 --ip 172.18.0.4 pxc

#创建第4个MySQL节点
docker run -d -p 3309:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 -v v4:/var/lib/mysql --privileged --name=node4 --net=net1 --ip 172.18.0.5 pxc

#创建第5个MySQL节点
docker run -d -p 3310:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 -v v5:/var/lib/mysql -v backup:/data --privileged --name=node5 --net=net1 --ip 172.18.0.6 pxc
```

> 注意，每个MySQL容器创建之后，因为要执行PXC的初始化和加入集群等工作，耐心等待1分钟左右再用客户端连接MySQL。另外，必须第1个MySQL节点启动成功，用MySQL客户端能连接上之后，再去创建其他MySQL节点。


# 负载均衡，双机热备

1. 安装 Haproxy 镜像

```
docker pull haproxy
```

2. 宿主机上编写 Haproxy 配置文件

```
vi /home/soft/haproxy/haproxy.cfg
```

配置文件如下：
```sh
global
    #工作目录
    chroot /usr/local/etc/haproxy
    #日志文件，使用rsyslog服务中local5日志设备（/var/log/local5），等级info
    log 127.0.0.1 local5 info
    #守护进程运行
    daemon

defaults
    log  global
    mode  http
    #日志格式
    option  httplog
    #日志中不记录负载均衡的心跳检测记录
    option  dontlognull
    #连接超时（毫秒）
    timeout connect 5000
    #客户端超时（毫秒）
    timeout client  50000
    #服务器超时（毫秒）
    timeout server  50000

#监控界面
listen  admin_stats
    #监控界面的访问的IP和端口
    bind  0.0.0.0:8888
    #访问协议
    mode  http
    #URI相对地址
    stats uri /dbs
    #统计报告格式
    stats realm Global\ statistics
    #登陆帐户信息
    stats auth admin:abc123456

#数据库负载均衡
listen  proxy-mysql
    #访问的IP和端口
    bind  0.0.0.0:3306
    #网络协议
    mode  tcp
    #负载均衡算法（轮询算法）
    #轮询算法：roundrobin
    #权重算法：static-rr
    #最少连接算法：leastconn
    #请求源IP算法：source 
    balance  roundrobin
    #日志格式
    option  tcplog
    #在MySQL中创建一个没有权限的haproxy用户，密码为空。Haproxy使用这个账户对MySQL数据库心跳检测
    option  mysql-check user haproxy
    server  MySQL_1 172.18.0.2:3306 check weight 1 maxconn 2000
    server  MySQL_2 172.18.0.3:3306 check weight 1 maxconn 2000
    server  MySQL_3 172.18.0.4:3306 check weight 1 maxconn 2000
    server  MySQL_4 172.18.0.5:3306 check weight 1 maxconn 2000
    server  MySQL_5 172.18.0.6:3306 check weight 1 maxconn 2000
    #使用keepalive检测死链
    option  tcpka
```

3. 创建两个 Haproxy 容器

```sh
#创建第1个Haproxy负载均衡服务器
docker run -it -d -p 4001:8888 -p 4002:3306 -v /home/soft/haproxy:/usr/local/etc/haproxy --name h1 --privileged --net=net1 --ip 172.18.0.7 haproxy

#进入h1容器，启动Haproxy
docker exec -it h1 bash
haproxy -f /usr/local/etc/haproxy/haproxy.cfg

#创建第2个Haproxy负载均衡服务器
docker run -it -d -p 4003:8888 -p 4004:3306 -v /home/soft/haproxy:/usr/local/etc/haproxy --name h2 --privileged --net=net1 --ip 172.18.0.8 haproxy

#进入h2容器，启动Haproxy
docker exec -it h2 bash
haproxy -f /usr/local/etc/haproxy/haproxy.cfg
```

4. 创建心跳检测账户

```
CREATE USER 'haproxy'@'%' IDENTIFIED BY '';
```

5. Haproxy容器内安装Keepalived，设置虚拟IP

```sh
#进入h1容器
docker exec -it h1 bash

#更新软件包
apt-get update

#安装VIM
apt-get install vim

#安装Keepalived
apt-get install keepalived

#编辑Keepalived配置文件（参考下方配置文件）
vim /etc/keepalived/keepalived.conf

#启动Keepalived
service keepalived start

#宿主机执行ping命令
ping 172.18.0.201
```

配置文件内容如下：
```
vrrp_instance  VI_1 {
    state  MASTER
    interface  eth0
    virtual_router_id  51
    priority  100
    advert_int  1
    authentication {
        auth_type  PASS
        auth_pass  123456
    }
    virtual_ipaddress {
        172.18.0.201
    }
}
```

```sh
#进入h2容器
docker exec -it h2 bash

#更新软件包
apt-get update

#安装VIM
apt-get install vim

#安装Keepalived
apt-get install keepalived

#编辑Keepalived配置文件
vim /etc/keepalived/keepalived.conf

#启动Keepalived
service keepalived start

#宿主机执行ping命令
ping 172.18.0.201
```

配置文件内容如下：
```
vrrp_instance  VI_1 {
    state  MASTER
    interface  eth0
    virtual_router_id  51
    priority  100
    advert_int  1
    authentication {
        auth_type  PASS
        auth_pass  123456
    }
    virtual_ipaddress {
        172.18.0.201
    }
}
```

6. 宿主机安装Keepalived，实现双机热备

```sh
#宿主机执行安装Keepalived
yum -y install keepalived

#修改Keepalived配置文件
vi /etc/keepalived/keepalived.conf

#启动Keepalived
service keepalived start
```

keepalived配置文件如下：
```
vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.10.150
    }
}

virtual_server 192.168.10.150 8888 {
    delay_loop 3
    lb_algo rr 
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 172.18.0.201 8888 {
        weight 1
    }
}

virtual_server 192.168.10.150 3306 {
    delay_loop 3
    lb_algo rr 
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 172.18.0.201 3306 {
        weight 1
    }
}
```

# 热备份数据

```sh
#进入node1容器
docker exec -it node1 bash

#更新软件包
apt-get update

#安装热备工具
apt-get install percona-xtrabackup-24

#全量热备
innobackupex --user=root --password=abc123456 /data/backup/full
```

# 冷还原数据

停止其余4个节点，并删除节点
```
docker stop node2
docker stop node3
docker stop node4
docker stop node5
docker rm node2
docker rm node3
docker rm node4
docker rm node5
```

node1容器中删除MySQL的数据
```sh
#删除数据
rm -rf /var/lib/mysql/*

#清空事务
innobackupex --user=root --password=abc123456 --apply-back /data/backup/full/2018-04-15_05-09-07/

#还原数据
innobackupex --user=root --password=abc123456 --copy-back  /data/backup/full/2018-04-15_05-09-07/
```

然后重新创建其余4个节点，组件PXC集群


# 配置 RedisCluster 集群

1. 安装 redis 镜像

```
docker pull yyyyttttwwww/redis
```

2. 创建 net2 网段

```
docker network create --subnet=172.19.0.0/16 net2
```

3. 创建6节点 redis 容器

```
docker run -it -d --name r1 -p 5001:6379 --net=net2 --ip 172.19.0.2 redis bash
docker run -it -d --name r2 -p 5002:6379 --net=net2 --ip 172.19.0.3 redis bash
docker run -it -d --name r3 -p 5003:6379 --net=net2 --ip 172.19.0.4 redis bash
docker run -it -d --name r4 -p 5004:6379 --net=net2 --ip 172.19.0.5 redis bash
docker run -it -d --name r5 -p 5005:6379 --net=net2 --ip 172.19.0.6 redis bash
docker run -it -d --name r5 -p 5006:6379 --net=net2 --ip 172.19.0.7 redis bash
```

> 注意：redis配置文件里必须要设置bind 0.0.0.0，这是允许其他IP可以访问当前redis。如果不设置这个参数，就不能组建Redis集群。


4. 启动6节点 redis 服务器

```sh
#进入r1节点
docker exec -it r1 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf

#进入r2节点
docker exec -it r2 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf

#进入r3节点
docker exec -it r3 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf

#进入r4节点
docker exec -it r4 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf

#进入r5节点
docker exec -it r5 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf

#进入r6节点
docker exec -it r6 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf
```

5. 创建 Cluster 集群

```sh
#在r1节点上执行下面的指令
cd /usr/redis/src
mkdir -p ../cluster
cp redis-trib.rb ../cluster/
cd ../cluster

#创建Cluster集群
./redis-trib.rb create --replicas 1 172.19.0.2:6379 172.19.0.3:6379 172.19.0.4:6379 172.19.0.5:6379 172.19.0.6:6379 172.19.0.7:6379

# redis-cli 客户端查看
./redis-cli -c

# 查看集群节点
127.0.0.1:6379> cluster nodes
```