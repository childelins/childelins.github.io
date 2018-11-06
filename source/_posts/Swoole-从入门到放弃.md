---
title: Swoole 从入门到放弃
date: 2018-11-04 11:54:19
tags:
cover: https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/depositphotos_12385452-stock-illustration-kids-playing.jpg 
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
        // 实例化 WebSocket 对象
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

        // 连接关闭时触发
        ws.onclose = function (evt) {
            console.log("connection closed.");
        };

        // 通信发生错误时触发
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

## 异步Task

Swoole提供了异步任务处理的功能，可以投递一个异步任务到 `task_worker` 进程池中执行，不影响 `worker` 进程请求的处理速度。要使用 `Task` 功能，必须先设置 `task_worker_num`，并且必须设置 Server 的 `onTask` 和 `onFinish` 事件回调函数。

**使用场景**

* 执行耗时的操作（发送邮件、广播等）

---

基于上面的 WebSocket 服务器只需要增加 `onTask` 和 `onFinish` 2个事件回调函数即可。另外需要设置 task 进程数量，可以根据任务的耗时和任务量配置适量的 task 进程。

ws.php

```php
<?php

class Ws
{
    public $ws = null;

    public function __construct($host, $port)
    {
        $this->ws = new swoole_websocket_server($host, $port);

        $this->ws->set([
            'work_num' => 4, // 设置启动的worker进程数
            'task_worker_num' => 4, // 设置Task进程的数量
        ]);

        $this->ws->on('open', [$this, 'onOpen']);
        $this->ws->on('message', [$this, 'onMessage']);
        $this->ws->on('task', [$this, 'onTask']);
        $this->ws->on('finish', [$this, 'onFinish']);
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

        // todo 10s
        $data = [
            'fd' => $frame->fd,
            'value' => 'hello',
        ];
        // 投递异步任务
        $server->task($data);

        $server->push($frame->fd, '服务端发送的消息：' . date('Y-m-d H:i:s'));
    }

    // 处理异步任务
    public function onTask($serv, $task_id, $from_id, $data)
    {
        echo "New AsyncTask[id=$task_id]" . PHP_EOL;

        // 模拟耗时场景
        sleep(10);

        //返回任务执行的结果
        $serv->finish(json_encode($data) . ' -> OK');
    }

    // 处理异步任务返回的结果
    public function onFinish($serv, $task_id, $data)
    {
        echo "AsyncTask[$task_id] Finish: $data" . PHP_EOL;
    }

    public function onClose($server, $fd)
    {
        echo "客户端 {$fd} 退出啦~";
    }
}

$ws = new Ws('0.0.0.0', 9502);
$ws->start();
```

运行结果：
```
[vagrant@docker-host server]$ php ws.php 
客户端 1 连接成功
接收到客户端 1 的消息： hello swoole
New AsyncTask[id=0]
AsyncTask[0] Finish: {"fd":1,"value":"hello"} -> OK
客户端 1 退出啦~客户端 2 连接成功
接收到客户端 2 的消息： hello swoole
New AsyncTask[id=1]
AsyncTask[1] Finish: {"fd":2,"value":"hello"} -> OK
```

## 毫秒定时器

swoole提供了类似JavaScript的 `setInterval/setTimeout` 异步高精度定时器，粒度为毫秒级。

```php
// 每隔2s触发一次
swoole_timer_tick(2000, function ($timer_id) {
    echo "tick-2000ms\n";
});

// 3s后执行此函数
swoole_timer_after(3000, function () {
    echo "after 3000ms.\n";
});
```

## 异步文件IO

## 异步MySQL

## 异步Redis

## 进程

进程就是正在运行的程序的一个实例。Swoole增加了一个进程管理模块，用来替代PHP的pcntl扩展。需要注意的是Process进程在系统是非常昂贵的资源，创建进程消耗很大。另外创建的进程过多会导致进程切换开销大幅上升。

process.php

```php
<?php

// 创建子进程
$process = new swoole_process(function (swoole_process $worker) {
    // 执行一个外部程序
    $worker->exec('/usr/local/php/bin/php', [
        __DIR__ . '/../server/http_server.php'
    ]);
}, true);

// 开启进程
$pid = $process->start();
echo $pid . PHP_EOL;

// 子进程结束后进行回收
swoole_process::wait();
```

一个多进程使用场景：
```php
<?php

echo 'process-start-time:' . date('H:i:s') . PHP_EOL;

$processes = [];
$urls = [
    'http://www.baidu.com',
    'http://www.sina.com.cn',
    'http://www.qq.com',
    'http://www.baidu.com?search=imooc',
    'http://www.baidu.com?search=fun',
    'http://www.baidu.com?search=sina',
];

$count = count($urls);
for ($i = 0; $i < $count; $i++) {
    // 多进程请求
    $process = new swoole_process(function (swoole_process $worker) use ($urls, $i) {
        $content = curlData($urls[$i]);
        // 向管道内写入数据
        $worker->write($content); // echo $content;
    }, true);

    $pid = $process->start();
    $processes[$pid] = $process;
}

foreach ($processes as $process) {
    // 从管道中读取数据
    echo $process->read();
}

/**
 * 模拟请求URL的内容
 *
 * @param [type] $url
 * @return void
 */
function curlData($url)
{
    // curl or file_get_contents
    sleep(1);

    return $url . ' - success' . PHP_EOL;
}

echo 'process-end-time:' . date('H:i:s') . PHP_EOL;
```

运行结果:
```
[vagrant@docker-host process]$ php curl.php 
process-start-time:09:17:51
http://www.baidu.com - success
http://www.sina.com.cn - success
http://www.qq.com - success
http://www.baidu.com?search=imooc - success
http://www.baidu.com?search=fun - success
http://www.baidu.com?search=sina - success
process-end-time:09:17:52
```

## 内存

* table

swoole_table是一个基于共享内存和锁实现的超高性能，并发数据结构。用于解决多进程/多线程数据共享和同步加锁问题。

```php
<?php

// 创建内存表
$table = new swoole_table(1024);

// 内存表增加一列
$table->column('id', $table::TYPE_INT, 4);
$table->column('name', $table::TYPE_STRING, 64);
$table->column('age', $table::TYPE_INT, 4);
$table->create();

// 设置一行数据
$table->set('boy', ['id' => 1, 'name' => 'kangkang', 'age' => 18]);

// 获取一行数据
print_r($table->get('boy'));

// 另外一种方式
$table['girl'] = ['id' => 2, 'name' => 'june', 'age' => 16];
print_r($table['girl']);

// 自增
$table->incr('girl', 'age', 2);
print_r($table['girl']);

// 自减
$table->decr('girl', 'age', 3);
print_r($table['girl']);

// 删除
$table->del('girl');
print_r($table['girl']);
```

## 协程

协程可以理解为纯用户态的线程，其通过协作而不是抢占来进行切换。相对于进程或者线程，协程所有的操作都可以在用户态完成，创建和切换的消耗更低。Swoole可以为每一个请求创建对应的协程，根据IO的状态来合理的调度协程，这会带来了以下优势：

* 开发者可以无感知的用同步的代码编写方式达到异步IO的效果和性能，避免了传统异步回调所带来的离散的代码逻辑和陷入多层回调中导致代码无法维护。

> 基于swoole_server或者swoole_http_server进行开发，目前只支持在onRequet, onReceive, onConnect等事件回调函数中使用协程

```php
<?php

$http = new swoole_http_server('0.0.0.0', 8001);

$http->on('request', function ($request, $response) {
    $redis = new Swoole\Coroutine\Redis();
    $redis->connet('127.0.0.1', 6379);
    $value = $redis->get($request->get['a']);

    $response->header('Content-Type', 'text/plain');
    $response->end($value);
});

$http->start();
```

# Swoole进程模型

Swoole 是一个多进程模式的框架，当启动一个 Swoole 应用时，一共会创建 `2 + n + m` 个进程。其中 n 为 Worker 进程数，m 为 TaskWorker 进程数，2 为一个 Master 进程和一个 Manager 进程。它们之间的关系如下图所示：

![](http://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/structure.png)


## Master 进程

Master进程为主进程，该进程会创建 Manager进程、Reactor线程等。

主进程内的回调函数：

* onStart
* onShutdown
* onMasterConnect
* onMasterClose
* onTimer

## Reactor线程

Swoole的主进程是一个多线程的程序。其中有一组很重要的线程，称之为Reactor线程。它就是真正处理TCP连接，收发数据的线程。

Swoole的主线程在Accept新的连接后，会将这个连接分配给一个固定的Reactor线程，并由这个线程负责监听此socket。在socket可读时读取数据，并进行协议解析，将请求投递到Worker进程。在socket可写时将数据发送给TCP客户端。

## Manager 进程

Manager进程为管理进程，该进程的作用是创建、管理所有的Worker进程和TaskWorker进程。

* 子进程结束运行时，manager进程负责回收此子进程，避免成为僵尸进程。并创建新的子进程
* 服务器关闭时，manager进程将发送信号给所有子进程，通知子进程关闭服务
* 服务器重启时，manager进程会逐个关闭/重启子进程

管理进程内的回调函数：

* onManagerStart
* onManagerStop

## Worker 进程

Worker进程作为Swoole的工作进程，所有的业务逻辑代码均在此进程上运行。当Reactor线程接收到来自客户端的数据后，会将数据打包通过管道发送给某个Worker进程。

当一个Worker进程被成功创建后，会调用onWorkerStart回调，随后进入事件循环等待数据。当通过回调函数接收到数据后，开始处理数据。如果处理数据过程中出现严重错误导致进程退出，或者Worker进程处理的总请求数达到指定上限，则Worker进程调用onWorkerStop回调并结束进程。主进程会重新拉起新的Worker进程。

Worker进程内的回调函数：

* onWorkerStart
* onWorkerStop
* onConnect
* onClose
* onReceive
* onTimer
* onFinish

## TaskWorker 进程

Task Worker是Swoole中一种特殊的工作进程，该进程的作用是处理一些耗时较长的任务，以达到释放Worker进程的目的。Worker进程可以通过swoole_server对象的task方法投递一个任务到Task Worker进程.

Worker进程通过Unix Sock管道将数据发送给Task Worker，这样Worker进程就可以继续处理新的逻辑，无需等待耗时任务的执行。需要注意的是，由于Task Worker是独立进程，因此无法直接在两个进程之间共享全局变量，需要使用Redis、MySQL或者swoole_table来实现进程间共享数据。

task_worker进程内的回调函数：

* onTask
* onorkerStart


# 参考

* [Swoole 官方文档](https://wiki.swoole.com/)
* [WebSocket 教程](http://www.ruanyifeng.com/blog/2017/05/websocket.html)
* [Swoole入门到实战打造高性能赛事直播平台](https://coding.imooc.com/class/197.html)