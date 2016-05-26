---
title: RabbitMQ初探
tags: [develop,MQ]
date: 2016-05-13 16:08:25
---
# 队列那点儿事儿
提高程序的性能很重要的一个思路就是将请求进行**异步化**处理，而在服务端异步处理的方式可以由**消息队列**来完成。

之前在处理消息推送等场景的时候曾经接触过“队列”的概念，当时是在DB中手工维护一张“业务队列表”，然后依靠定时任务来消耗表中的任务。这只是队列应用的其中一个简单的场景。

面对大型程序，这种自建队列的机制显然麻烦而缺乏通用性，于是诞生了基于AMQP协议的这种消息队列组件，如：**RabbitMQ**
# 使用队列的原因
在DB集中访问的时候，压力会很大，为了缓解这种情况，将insert的操作放入队列中，即时返回客户端操作成功，由消费者处理队列，将insert的操作异步化。

总结来说，就是：
1. 异步
2. 解耦（生产和消费分离，多系统解耦，即便跨语言，amqp协议都可以进行完美通信）
3. 高并发的缓冲器

# 一些需要理解的概念
当消费者为空的时候，消息是不会消失的

多个消费者订阅到同一个队列，任务只会被一个拿走

如果同一个消息，两个系统都想要这份数据，请开两条队列分别订阅，否则会导致数据不完整。

队列中的消息是“有序”的（先进先出嘛）
消息发送是有序的，所以，如果你的业务逻辑需要支持有序操作，那么，请不要开多个进程，这样多进程消费消息会打乱消息的顺序。（在设计上尽量杜绝有序业务）

消息有确认机制（ack命令），每条消息必须确认后才能读取下一条。所以当程序异常退出后，只要没有确认过，是可以重新处理的。（不建议开启自动确认）

交换器常用的类型有：direct交换器，fanout交换器，topic交换器
其中fanout交换器为广播模式，会把消息给到所有绑定到他上面的队列上

服务器重启，交换器和队列会丢失

交换器，队列，消息可以做持久化，但是性能很低（需要写入文件），不建议使用


# 环境搭建
## 服务端搭建（windows）
**RabbitMQ Server下载**
http://www.rabbitmq.com/download.html
安装的时候提示需要安装erlang，否则web页面没有办法访问。

erlang下载地址：
http://www.erlang.org/download.html

使用exe进行安装即可。

## php客户端搭建
下载php扩展包：
http://pecl.php.net/package/amqp/1.4.0/windows

将php_amqp.dll放在php的ext目录里，然后修改php.ini文件，在文件的最后面添加两行

```
[amqp]
extension=php_amqp.dll
```

将rabbitmq.1.dll文件放在php的根目录里(也就是ext目录的父级目录)，然后修改apache的httpd.con文件，文件尾部添加一行

```
LoadFile  "D:/xampp/php/rabbitmq.1.dll"
```

重启apache，并查看phpinfo信息。只要看到amqp 字样即可。



# 使用
php调用示例
```php
$data=array(
    'device_platform'=>$type,
    'activation_params'=>$arr_activation_params,
);
$this->load->library('amqp/publisher', array('connect_key' => 'index_activation'));
$this->config->load("amqp");
$amqpConf = $this->config->item("amqp_index_activation");
$re=$this->publisher->inint($amqpConf['exchange'], $amqpConf['exchange_type'], $amqpConf['durable']);
$str_activation_info = json_encode($data);
$result = $this->publisher->send($str_activation_info, $amqpConf['routing_key']);
$this->load->library('monolog_client');
if (!$result) {//写入队列失败
    $this->monolog_client->error('error: code=501 info=写入队列失败 str_activation_info='.$str_activation_info, 'api_error');
}
```
发送后会出现在**exchanges**标签中，消费后会出现在**queues**标签中。

# 数据监控
RabbitMQ 有个很和谐的web管理插件叫 rabbitmq_management
启动方式：
打开命令行模式cmd：
依次输入：
```
D:\RabbitMQ Server\rabbitmq_server-3.6.1\sbin>rabbitmq-plugins.bat enable rabbitmq_management
The following plugins have been enabled:
  mochiweb
  webmachine
  rabbitmq_web_dispatch
  amqp_client
  rabbitmq_management_agent
  rabbitmq_management
```
```
D:\RabbitMQ Server\rabbitmq_server-3.6.1\sbin>rabbitmq-service.bat stop
D:\erl7.3\erts-7.3\bin\erlsrv: Service RabbitMQ stopped.
```
```
D:\RabbitMQ Server\rabbitmq_server-3.6.1\sbin>rabbitmq-service.bat install
RabbitMQ service is already present - only updating service parameters
```
```
D:\RabbitMQ Server\rabbitmq_server-3.6.1\sbin>rabbitmq-service.bat start
D:\erl7.3\erts-7.3\bin\erlsrv: Service RabbitMQ started.
```

登录 http://127.0.0.1:15672
账号密码都是 guest



# 参考资料
http://www.360doc.com/content/14/0911/17/15077656_408714488.shtml
https://www.kakata.com/archives/1187
