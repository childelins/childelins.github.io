---
title: Swoole 从入门到放弃
date: 2018-10-13 11:54:19
tags:
cover:https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/44e818397558c3770b38c6978f35cbe8.jpg
---

# 简介

Swoole 使用纯 C 语言编写，是面向生产环境的 PHP 异步网络通信引擎。使 PHP 开发人员可以编写高性能的异步并发 TCP、UDP、Unix Socket、HTTP，WebSocket 服务。

* PHP 异步网络通信引擎
* 最终编译为 so 文件作为 PHP 的扩展

# 安装

1. 下载

> https://github.com/swoole/swoole-src/releases


2. 编译安装

```sh
cd swoole/
phpize
./configure --enable-swoole-debug --with-php-config=/usr/local/php/bin/php-config
make
make install
```

3. 配置

编译安装成功后，修改 `php.ini` 加入
```
extension=swoole.so
```

通过 `php -m` 来查看是否成功加载了 swoole。


# 快速起步

## TCP 服务

`swoole_server` 是异步服务器，所以是通过监听事件的方式来编写程序的。当对应的事件发生时底层会主动回调指定的PHP函数。如当有新的TCP连接进入时会执行 `onConnect` 事件回调，当某个连接向服务器发送数据时会回调 `onReceive` 函数。

tcp_server.php

```php
<?php

// 创建 Server 对象，监听 127.0.0.1:9501 端口
$serv = new swoole_server('127.0.0.1', 9501);

// 设置运行时参数
$serv->set([
    'worker_num' => 8, // worker进程数
    'max_request' => 10000,
]);

// 监听连接进入事件
// fd: 客户端连接的唯一标识
// reactor_id: 线程id
$serv->on('connect', function ($serv, $fd, $reactor_id) {
    echo "Client: {$reactor_id} - {$fd} - Connect.\n";
});

// 监听数据接收事件
$serv->on('receive', function ($serv, $fd, $from_id, $data) {
    // 服务段发送消息给客户端
    $serv->send($fd, "Server: {$from_id} - {$fd} -" . $data);
});

//监听连接关闭事件
$serv->on('close', function ($serv, $fd) {
    echo "Client: {$fd} - Close.\n";
});

//启动服务器
$serv->start();
```

运行程序：

```
php tcp_server.php
```

> 使用 `netstat -anp | grep 端口`，查看端口是否已经被打开处于 `Listening` 状态


这时可以使用 `telnet ` 工具连接到服务器:
```
[vagrant@docker-host ~]$ telnet 127.0.0.1 9501
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
hello
Server: 0 - 1 -hello
```

也可以创建一个TCP客户端，来连接到我们的服务器。

tcp_client.php

```php
<?php

$client = new swoole_client(SWOOLE_SOCK_TCP);

// 连接到服务器
if (!$client->connect('127.0.0.1', 9501)) {
    die('连接失败');
}

// php cli常量
fwrite(STDOUT, '请输入消息：');
$data = trim(fgets(STDIN));

// 向服务器发送数据
if (!$client->send($data)) {
    die('发送失败');
}

// 从服务器接收数据
$data = $client->recv();
if (!$data) {
    die('接收失败');
}

echo $data . PHP_EOL;

// 关闭连接
$client->close();

```

运行程序：
```
php tcp_client.php
```

运行结果：
```sh
[vagrant@docker-host client]$ php tcp_client.php 
请输入消息：hello
Server: 0 - 2 -hello
```


## UDP 服务

略


## HTTP 服务

Swoole内置Http服务器的支持，通过几行代码即可写出一个异步非阻塞多进程的Http服务器。Http服务器只需要关注请求响应即可，所以只需要监听一个 `onRequest` 事件。当有新的Http请求进入就会触发此事件。

```php
<?php

$http = new swoole_http_server('0.0.0.0', 8811);

$http->set([
    'document_root' => '/home/vagrant/learn/learning/swoole/data', // 配置静态文件根目录
    'enable_static_handler' => true,
]);

$http->on('request', function ($request, $response) {
    // 获取请求参数
    $gets = $request->get;

    // 设置cookie
    $response->cookie('swoole', '123', time() + 1800);

    // 输出一段 HTML内容，并结束此请求
    $response->end('<h1> Hello Swoole. #' . rand(1000, 9999) . '</h1>' . json_encode($gets));
});

$http->start();
```

运行程序：
```
php http_server.php
```

打开浏览器 `http://192.168.10.15:8811/?aaa=123&bbb=456` 查看请求结果：
```
Hello Swoole. #6898

{"aaa":"123","bbb":"456"}
```

可以在静态文件目录下增加 `index.html` 文件,  访问 `http://192.168.10.15:8811/index.html` ,此时不走 `onRequest` 回调：
```
我是通过 http server 获取的静态资源内容
```

## WebSocket 服务

**什么是 WebSocket**

WebSocket 协议是基于 TCP 的一种新的网络协议。它实现了浏览器与服务器的全双工（full-duplex）通信  —— ***允许服务器主动发送消息给客户端***。

**为什么需要 WebSocket**

缺陷：HTTP 的通信只能由客户端发起

**WebSocket 的特点**

* 建立在TCP协议之上 
* 性能开销小通信高效
* 客户端可以和任意服务器通信
* 协议标识符 ws wss
* 持久化网络通信协议

---

WebSocket服务器是建立在Http服务器之上的长连接服务器，客户端首先会发送一个Http的请求与服务器进行握手。握手成功后会触发 `onOpen` 事件，表示连接已就绪。建立连接后客户端与服务器端就可以双向通信了。

ws_server.php

面向过程写法：

```php
<?php

// 创建 websocket 服务器对象，监听 0.0.0.0:9502 端口
$server = new swoole_websocket_server('0.0.0.0', 9502);

// 监听 WebSocket 连接打开事件
$server->on('open', function ($server, $request) {
    echo "server: handshake success with fd{$request->fd}\n";
});

// 监听 WebSocket 消息事件
$server->on('message', function ($server, $frame) {
    echo "receive from {$frame->fd}:{$frame->data},opcode:{$frame->opcode},fin:{$frame->finish}\n";

    // 服务器发送消息给客户端
    $server->push($frame->fd, 'this is server');
});

// 监听 WebSocket 连接关闭事件
$server->on('close', function ($ser, $fd) {
    echo "client {$fd} closed\n";
});

$server->start();
```

面向对象写法：

```php
<?php

class Ws
{
    public $ws = null;

    public function __construct($host, $port)
    {
        $this->ws = new swoole_websocket_server($host, $port);

        $this->ws->on('open', [$this, 'onOpen']);
        $this->ws->on('message', [$this, 'onMessage']);
        $this->ws->on('close', [$this, 'onClose']);
    }

    public function start()
    {
        $this->ws->start();
    }

    public function onOpen($server, $request)
    {
        echo "客户端 {$request->fd} 连接成功\n";
    }

    public function onMessage($server, $frame)
    {
        echo "接收到客户端 {$frame->fd} 的消息： {$frame->data}\n";
        $server->push($frame->fd, '服务端发送的消息：' . date('Y-m-d H:i:s'));
    }

    public function onClose($server, $fd)
    {
        echo "客户端 {$fd} 退出啦~";
    }
}

$ws = new Ws('0.0.0.0', 9502);
$ws->start();
```

ws_client.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <h1>swoole websocket 测试</h1>

    <script>
        // 实例化  WebSocket 对象
        var ws = new WebSocket('ws://192.168.10.15:9502');
        
        // 连接建立时触发
        ws.onopen = function (evt) {
            console.log('connection open');
            // 客户端发送消息给服务器
            ws.send('hello swoole');
        };

        // 客户端接收服务端数据时触发
        ws.onmessage = function (evt) {
            console.log("received message: " + evt.data);
        };

        // 	连接关闭时触发
        ws.onclose = function (evt) {
            console.log("connection closed.");
        };

        // 	通信发生错误时触发
        ws.onerror = function (evt) {
            console.log("error");
        };
    </script>
</body>
</html>
```

运行结果：
```
[vagrant@docker-host server]$ php ws.php 
客户端 1 连接成功
接收到客户端 1 的消息： hello swoole
客户端 1 退出啦~客户端 2 连接成功
接收到客户端 2 的消息： hello swoole
客户端 2 退出啦~客户端 3 连接成功
接收到客户端 3 的消息： hello swoole
```

# 参考

* [Swoole 官方文档](https://wiki.swoole.com/)
* [WebSocket 教程](http://www.ruanyifeng.com/blog/2017/05/websocket.html)
* [Swoole入门到实战打造高性能赛事直播平台](https://coding.imooc.com/class/197.html)