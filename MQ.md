# RabbitMQ

## 特点

* 可靠性：使用一些机制来保证可靠性，如：持久化，传输确认和发布确认
* 灵活的路由：消息进入队列之前，通过交换器来路由消息
* 扩展性：多个RabbitMQ节点可以组成一个集群。
* 高可用：队列可以在集群的机器上设置镜像
* 管理界面
* 插件机制

## 相关概念

### 结构模型

![1553397983056](D:\data\md笔记\img\RabbitMq.png)

* Producer：生产者
* Consumer：消费者
* Broker：消息中间件的服务节点

### 队列

* RabbitMQ中的消息只能存储在队列中。
* 多个消费者可以订阅同一个队列，这时队列中的消息会被平均分摊给多个消费者进行处理。而不是每个消费者都收到所有的消息并处理
* RabbitMQ不支持队列层面的广播消费

### 交换器、路由键、绑定

1、Exchange：交换器。

​	生产者将消息发送到Exchange，由交换器将消息路由到一个或多个队列中。

2、RoutingKey：路由键

​	生产者将消息发给交换器的时候，一般会指定一个RoutingKey，用来指定这个消息的路由规则，这个RoutingKey需要与交换器类型和绑定键联合使用才生效

3、Binding：绑定

​	RabbitMQ中通过绑定将交换器和队列关联起来，在绑定的时候一般会指定一个绑定键（BindingKey）.

### 交换器类型

* fanout

  把所有发送到该交换器的消息路由到所有与该交换器绑定的队列中

* direct

  会把消息路由到那些BindingKey和RoutingKey完全匹配的队列中

* topic

  与direct类似，但是匹配规则有些不同，他约定

  +  RoutingKey为一个点号”.“分割的字符串
  + BindingKey和RoutingKey一样也是点号”.“分割的字符串
  + BindingKey中可以存在两种特殊的字符串”*“和”#“，用于做模糊匹配，其”*“用于匹配一个单词，#用于匹配多个单词

* headers

  header类型的交换器不依赖路由键的匹配规则来路由消息，而是根据发送的消息内容中的headers属性进行匹配。【不实用】

##  安装

### Erlang安装

### RabbitMQ 安装

## RabbitMQ命令

```bash
# -detached 以守护进程的方式在后台运行
rabbitmq-server -detached
# 查看rabbitmq 是否正常启动
rabbitmqctl status
# 查看集群新
rabbitmqctl cluster_status

# 查看当前所有用户
rabbitmqctl list_users
# 删除用户
rabbitmqctl delete_user guest
# 添加用户
rabbitmqctl add_user username password
# 赋予权限
rabbitmqctl set_permissions -p / username ".*" ".*" ".*"
# 查看用户权限
rabbitmqctl list_user_permissions username 
```

### 开启web

```shell
# rabbitmq-plugins enable rabbitmq_management
# 默认端口 15672
```

### 用户角色

* 【administrator】——超级管理员

  可登陆管理控制台，可查看所有用户，并且可以对用户，策列进行操作

* 【monitoring】—— 监控者

  可登陆管理控制台，同时可以查看rabbitmq节点的相关信息

* 【policymaker】——策略制定者

  可以登录管理控制台，同时对策略进行管理。

* 【management】——普通管理者

  仅可登陆管理控制台，无法看到节点信息，也无法对策略进行管理

  ```shell
  # 对用户进行设置角色
  rabbitmqctl set_user_tags username 角色
  ```




## 客户端开发

### 连接RabbitMQ

