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

![1553397983056](.\img\RabbitMq.png)

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

### HelloWord

```java
			ConnectionFactory factory = new ConnectionFactory();
        	factory.setHost(host);
        	factory.setPort(port);
        	factory.setUsername(userName);
        	factory.setPassword(password);
      		Connection connection =  factory.newConnection();
            //创建信道
	        Channel channel = connection.createChannel();
			//创建交换器
            channel.exchangeDeclare("myExchangeDeclare","direct",true,false,null);
			//创建队列
            channel.queueDeclare("myQueue",true,false,false,null);
            /*
             * 绑定交换器和队列
             */
            channel.queueBind("myQueue","myExchangeDeclare","routingKey_demo");
			//发送
            String message = "Hello world";
            channel.basicPublish("myExchangeDeclare","routingKey_demo",
                    MessageProperties.PERSISTENT_TEXT_PLAIN,
                    message.getBytes()
                    );
            channel.close();
            connection.close();
```

### 创建队列和交换器API

```java
exchangeDeclare(String exchangeName,String type,boolean durable,boolean,  	                             autoDelete,boolean internal,Map<String,Object> argument)
```

```java
queueDeclare(String queueName,boolean durable,boolean exclusive,boolean                                   autoDelete,Map<String,Object> argument)
```

```java
queueBind(String queueName,String exchangeName,String routingKey,                                         Map<String,Object> arguments)
```

# Kafka

## 术语

【消息和批次】

Kafka的数据单元被称为消息。可以把消息看成是数据库里的一个"数据行"或一条记录。消息由字节数组组成。

为了提高效率，消息被分批次写入Kafka.批次就是一组消息，这些消息属于同一个主题和分区

【主题和分区】

Kafka的消息通过主题进行分类。主题好比数据库的表。

主题可以分为若干个分区，一个分区就是一个提交日志。消息以追加的方式写入分区，然后以先入先出的顺序读取。

【注意】一个主题可以包含几个分区，因此无法在整个主题的范围内保证消息的顺序，但是可以保证消息在分区内的顺序。

【生产者和消费者】

生产者创建消息。一般情况下，一个消息会被发布到一个特定的主题上。生产者在默认情况下把消息均衡的分布到主题的所有分区上。

消费者读取消息。消费者订阅一个或多个主题，并安装消息生产的顺序读取它们。消费者通过检查消息的偏移量来区分已经读取过的消息。

消费者是消费者群组的一部分，也就是说，会由一个或多个消费者共同读取一个主题。群组保证每个分区只能被一个消费者使用。

【broker和集群】

一个独立的Kafka服务器被称为broker.

broker接收来自生产者的消息。为消息设置偏移量，并提交消息到磁盘保存。

broker是集群的组成部分，每个集群都有一个Broker同时充当集群控制器的角色。

## 为什么选择Kafka

> 多个生产者
>
> > Kafka可以无缝的支持多个生产者，不管客户端在使用单个主题还是多个主题。
> >
> > 所以它很适合从多个前端系统收集数据
>
> 多个消费者
>
> > Kafka支持多个消费者从一个单独的消息流上读取数据，而且消费者之间互不影响。
> >
> > > 这与其他队列系统不同：其他队列系统消息一旦被一个客户端读取，其他客户端就无法再读取它
> >
> > 另外，多个消费者可以组成一个群组，共享一个消息流；并且保证整个群组对给定的消息只处理一次
>
> 基于磁盘的数据存储
>
> 伸缩性
>
> 高性能



## 安装

### 配置

#### 常规配置

【broker.id】

​	每个broker都需要一个标识符，使用broker.id表示。他的默认值是0,也可以被设置位任何整数。这个值在整个Kafka集群里必须是唯一的。

【port】

​	如果使用配置样本来使用Kafka,他会监听9092端口。修改port参数可以把它设置为其他任何可用的端口。

【zookeeper.connect】

​	用于保存broker元数据的Zookeeper地址是通过该参数指定的。

【log.dirs】

​	Kafka把所有信息都保存在磁盘上，存放这些日志片段的目录是通过log.dirs指定的。用逗号分隔，如果指定多个路径，那么broker会根据“最少使用”原则，把同一个分区的日志片段保存在同一路径下。

【num.recovery.threads.per.data.dir】

​	对于如下3种情形，Kafka会使用可配置的线程池来处理日志片段：

​	1、服务器正常启动，用户打开每个分区的日志片段

 	2、服务器崩溃后重启，用于检查和截短每个分区的日志片段

​	3、服务器正常关闭，用户关闭日志片段。

​	设置此参数时，所配置的数字对应的是log.dirs指定的单个日志目录。即：如果num.recovery.threads.per.data.dir设为8，log.dir指定了3个路径，则总的线程数为24.

【auto.create.topics.enable】

​		默认情况下，Kafka会在如下几种情形下自动创建主题：

* 当一个生产者开始往主题写入消息
* 当一个消费者开始从主题读取消息
* 当任意一个客户端往主题发送元数据请求

#### 主题的默认配置

【num.partitions】

​		指定了新建的主题将包含多少个分区。

【log.retention.ms】

​		kafak通常根据时间来决定数据可以被保留多久。默认使用log.retention.hours参数来配置时间，默认值为168个小时。还有其他两个参数：log.retention.minutes和log.retention.ms。

​		如果指定了不知一个参数，默认使用最小值			

【log.retention.bytes】

​		通过保留的消息字节数来判断消息是否过期。它的值通过log.retention.bytes来指定。作用于每个分区上。

​		就是说，如果一个包含8个分区的主题，并且log.retention.bytes被设置为1GB,那么这个主题最多可以保存8GB的数据。

【log.segment.ms】

【message.max.bytes】

### 硬件的选择

#### 磁盘吞吐量

​	生产者客户端的性能直接受到服务器端磁盘吞吐量的影响。生产者生成的消息必须被提交到服务器保存。大多数客户端在发送消息后会一直等待。直到至少一个服务器确认消息已经成功提交为止。【磁盘写入速度越快，生成消息的延迟就越低】

#### 磁盘容量

#### 内存

​	服务器可用内存是影响客户端性能的主要因素。磁盘性能影响生产者，而内存影响消费者。消费者一般从分区尾部读取消息，如果有生产者存在，就紧跟在生产者后面。

## 向Kakfa写入数据

向Kakfa写入消息的步骤图：

![1558333276182](.\img\1558333276182.png)

#### 客户端

```java
package com.shengling.kafkatest;

import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import sun.applet.Main;


import java.util.Properties;
import java.util.concurrent.ExecutionException;

/**
 * @author shengling23
 * @create 2019-05-20 14:23
 */
public class SendDemo {

    private Properties kafkaProp = new Properties();
    private KafkaProducer<String,String> producer;

    private class ProducterCallBack implements Callback{

        @Override
        public void onCompletion(RecordMetadata recordMetadata, Exception e) {
            if (e != null){
                e.printStackTrace();
            }
            System.out.println("topicName"+recordMetadata.topic());
            System.out.println("toString"+recordMetadata.toString());
        }
    }

    public void init(){
        kafkaProp.put("bootstrap.servers","localhost:9092");
        kafkaProp.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");
        kafkaProp.put("value.serializer","org.apache.kafka.common.serialization.StringSerializer");
        producer = new KafkaProducer(kafkaProp);
    }

    /**
     * 发送并忘记
     * 把消息发送给服务器，并不关心它是否正常到达
     */
    public void fireAndForgetSend(){
        ProducerRecord<String,String> record =
                new ProducerRecord<>("CustomerCountry","Products","France");

        producer.send(record);

    }

    /**
     * 同步发送
     * 使用send发送消息，返回一个Future对象。调用get方法进行等待，就可以知道是否发送成功。
     */
    public void syncSend(){
        ProducerRecord<String,String> record =
                new ProducerRecord<>("CustomerCountry","Products","Raiss");

        try {
            producer.send(record).get();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }

    /**
     * 异步发送
     * 使用send发送消息，并指定一个回调方法，服务器在返回响应式，回调该方法
     */
    public void asyncSend(){
        ProducerRecord<String,String> record =
                new ProducerRecord<>("CustomerCountry","Products","England");

        producer.send(record,new ProducterCallBack());
    }

    public static void main(String[] args) {
        SendDemo sendDemo = new SendDemo();
        sendDemo.init();
        sendDemo.syncSend();
    }
}
```

#### 生产者配置

【acks】

```shell
# acks参数指定了必须有多少个分区副本收到消息，生产者才会认为消息写入是成功的。
acks=0 
	# 生产者在成功写入消息之前不会等待来自任何服务器的响应。
	# 如果出现问题，导致服务器没有收到消息，那么生产者无从得知。消息就丢失了
	# 生产者不需要等待服务器的响应，它可以以网络能够支持的最大速度发消息，从而达到很高的吞吐量
acks=1
	# 只要集群的首领节点收到消息，生产者就会收到一个来自服务器的成功响应。
acks=all
	# 只有当所有参与复制的节点全部收到消息时，生产者才会收到一个来自服务器的成功响应
```

【buffer.memory】

```shell
# 该参数用来设置生产者内存缓冲区的大小。
```

【compression.type】

```shell
# 默认情况下，消息发送时不会被压缩。该参数可以设置为：snappy,gzip或lz4
```

【retries】

```shell
# 生产者从服务器收到的错误有可能是临时的错误。在这种情况下，retries参数的值决定了生产者可以重发消息的次数
```

【batch.size】

### 序列化器

## 从Kafka读取数据





## 深入Kafka

### 1、集群成员关系

​	Kafka使用zookeeper来维护集群成员关系。

​	每个broker都有唯一标识符。

​	在关闭broker时，它对应的节点会消失，不过它的ID会继续存在于其他数据结构中。如果相同ID的一个全新的broker启动，它会立即加入集群，并用拥有与旧broker相同的分区和主题。

### 2、控制器

​	控制器其实就是一个broker。只不过它除了具有一般broker的功能外，还负责分区首领的选举。

​	kafka使用zookeeper的临时节点来选举控制器。并且使用epoch来避免“脑裂”。

### 3、复制

​	kafka使用主题来组织数据。每个主题被分为若干个分区。每个分区有多个副本。那些副本被保存在broker上，每个broker可以保存成百上千个属于不同主体和分区的副本

​	副本有两种类型：

* 首领副本

  每个分区都有一个首领副本。为了保证一致性，所有生产者和消费者请求都会经过这个副本。

* 跟随者副本

  首领以外的副本都是跟随者副本。它们唯一的任务就是从首领那里复制消息，保持与首领一致的状态。如果首领发生崩溃，其中一个跟随者就会被提升为新首领。

 首领的另一个任务是搞清楚哪个跟随者的状态与自己是一致的。

为了与首领保持同步，跟随者向首领发送获取数据的请求，这种请求与消费者为了读取消息而发送的请求是一致的。【请求消息总包含了跟随者想要获取消息的偏移量】。

​		首领通过查看每个跟随者请求的最新偏移量，知道每个跟随者的进度。如果跟随者在10s内没有任何请求消息，或10s内没有请求最新的数据，那么它就会认为是不同步的。

​		不同步的副本在首领失效时是不可能成为新首领的。

### 4、处理请求

​	broker大部分工作时处理客户端、分区副本和控制器发送给分区首领的请求。

​	几种最常见的请求类型：

* 生产者请求

  生产者发送的请求，它包含客户端要写入broker的消息

* 获取请求

  在消费者和跟随者副本需要从broker读取消息时发送的请求。

生产请求和获取请求都必须发送给分区的首领副本。

##### 生产请求



## 流式处理

> 什么是数据流(也称之为：事件流或流数据)
>
> > 特点：
> >
> > 1、数据流是无边界数据集的抽象表示
> > 	无边界意味着无限和持续增长。
> > 2、事件流是有序的
> > 3、不可变的数据记录
> > 4、事件流是可重播的
>
> 流式处理的一些概念
>
> > 流式处理的很多方面与普通的数据处理时相似的：
> >
> > > 写一些代码来接收数据
> > >
> > > 对数据进行处理（可能做一些转换，聚合和增强的操作）
> > >
> > > 然后把结果输出到某些个地方
> >
> > 时间
> >
> > > 时间或许是流式处理的最为重要的概念，
> >
> > 





























