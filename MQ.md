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
> > Kafka不仅支持多个消费者，还允许消费者非实时的读取消息。
> >
> > 这要归功于Kafka的数据保留特性。消息被提交到磁盘，根据设置的保留规则进行保存。
>
> 伸缩性
>
> 高性能



## 安装

## 环境

#### 安装java

#### 安装Zookeeper

#### 命令行操作

```shell
# 启动服务
bin/kafka-server-start.sh 
# 创建主题
kafka-topic.sh --create --zookeeper localhost:2181 --replication-factor 1  --partitions --topic test
# 验证主题
kafka-topic.sh --zookeeper localhost:2181 --describe --topic test
# 发布消息
kafka-console-producer.sh --broker-list localhost:9092 --topic test
# 读取消息
kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
```

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

>1、设置生产者属性,Kafka生产者有3个必须的属性
>
>> bootstrap.servers
>>
>> > 指定broker的地址清单，地址格式为`hots:port`
>> >
>> > 清单里不需要包含所有broker地址，生产者会从给定的broker里查找其他的broker信息。不过建议至少提供两个broker信息
>>
>> key.serializer
>>
>> >broker希望接收到的消息的键和值都是字节数组。
>> >
>> >该属性设置键的序列化类
>>
>> value.serializer
>>
>> > 该属性设置值得序列化类
>>
>> ```java
>> Properties kafkaProp = new Properties()
>> kafkaProp.put("bootstrap.servers","localhost:9092");     kafkaProp.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");       kafkaProp.put("value.serializer","org.apache.kafka.common.serialization.StringSerializer");
>> producer = new KafkaProducer(kafkaProp);
>> ```
>
>2、发送消息，有3种发送方式
>
>>发送并忘记
>>
>>> 把消息发送给服务器，并不关心它是否会正常到达
>>>
>>> 可能会丢失一些信息
>>
>>同步发送
>>
>>> 使用send发送，返回一个Future对象，调用get方法进行等待。
>>
>>异步发送
>>
>>> 调用send反复，并指定一个回调函数

代码示例：

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

```shell
# 当有多个消息需要被发送到同意各分区时，生产者会把它们放在同一个批次里。
# 该参数制定了一个批次可以使用的内存的大小，按照字节数计算。
# 当批次被填满时，批次里的所有消息会被发送出去。不过亦不一定
```

【linger.ms】

```shell
# 该参数指定了生产者在发送批次之前等待更多消息加入批次的时间。
```

【client.id】

```shell
# 该参数可以是任意的字符串，服务器会用它来识别消息的来源。还可以用在日志和配额指标里
```

【max.in.flight.requests.per.connection】

```shell
# 该蚕食指定了生产者在收到服务器响应之前可以发送多少个消息。它的值越高，就会占用越多的内存，不过也会提升吞吐量。把它设为1，可以保证消息安装发送的顺序写入服务器的 
```

【timeout.ms、request.timeout.ms和metadata.fetch.timeout.ms】

```shell
# request.timeout.ms 指定了生产者在发送数据时等待服务器返回响应的时间。
# metadata.fetch.timeout.ms 制定了生产者在获取源数据时等待服务器响应的时间，如果超时，那么生产者要么重试发送数据，要么返回一个错误
# timeout.ms 指定了broker等待同步副本返回消息确认的时间，与asks的配置去匹配，如果在指定的时间内没有收到同步副本的确认，那么broker会返回一个错误。
```

【max.block.ms】

```shell
# 该参数指定了在调用send()方法或使用partitionsFor()方法获取元数据时生产者的阻塞时间
```

【max.request.size】

```shell
# 该参数用于控制生产者发送的请求大小。
```

【recevie.buffer.bytes和send.buffer.bytes】

```shell
# 这两个参数指定了TCP socket接收和发送数据报的缓冲区大小。如果置为1，使用操作系统的默认值
```

### 序列化器

## 从Kafka读取数据

> 消费者和消费者群组
>
> > Kafka消费者从属于消费者群组。一个群组里的消费者订阅的是同一个主题，每个消费者接收主题一部分分区的消息
> >
> > > 消费者的数量不应该超过主题的分区数量，否则超过的部分会被闲置，不会接收任何消息
>
> 消费者群组和分区再均衡
>
> > 群组里的消费者共同读取主题的分区。一个新的消费者加入群组时，它读取的是原本由其他消费者读取的消息。当一个消息被关闭或发生奔溃时，它会离开群组，原本由它读取的分区将由群组里的其他消费者来读取。
> >
> > 分区的所有权从一个消费者转移到另一个消费者，这样的行为被称之为再均衡。
> >
> > > 在均衡非常重要，它为消费者群组提供了高可用性和伸缩性。
> > >
> > > 在再均衡期间，消费者无法读取消息，造成整个群组一小段时间不可用。
> > >
> > > 另外，当分区被重新分配给另一个消费者时，消费者当前的读取状态会丢失。

### 客户端

>1、指定消费者属性,3个必须的属性
>
>> bootstrap.servers
>>
>> key.deserializer
>>
>> value.deserializer
>>
>> group.id
>>
>> ​	指定消费者属于哪个群组
>>
>> ```shell
>> kafkaProp.put("bootstrap.servers","localhost:9092");   kafkaProp.put("key.deserializer","org.apache.kafka.common.serialization.StringDeserializer");     kafkaProp.put("value.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
>> kafkaProp.put("group.id","consumerGroup");
>> consumer = new KafkaConsumer(kafkaProp);
>> ```
>
>2、订阅主题
>
>>```java
>>consumer.subscribe(Collections.singletonList("CustomerCountry"));
>>```
>>
>>可以在subscribe方法传入正则表达式，订阅多个主题
>
>3、轮询
>
>> 消息轮询是消费者API的核心，通过一个简单的轮询向服务器请求数据。
>>
>> 一旦消费者订阅了主题，轮询会处理所有的细节，包括群组协调，分区再均衡，发送心跳和获取数据。
>>
>> 开发者只需要使用一组简单的API来处理从分区返回的数据。
>>
>> ```JAVA
>> try {
>> 	while (true) { //1
>>        //轮询获取信息
>>        ConsumerRecords<String,String> records =                                                 consumer.poll(100L); //2
>>         //3
>> 	   for(ConsumerRecord<String,String> record : records){
>> 			System.out.println("Entry...");
>> 	        System.out.println("record-key"+record.key());
>> 			System.out.println("record- value"+record.value());
>>        }
>> 
>>      }
>> }finally {
>>             consumer.close(); // 4
>> }
>> ```
>>
>> > 1、无限循环。消费者实际上是一个长期运行的应用程序，通过持续轮询向Kafka请求数据
>> >
>> > 2、非常重要。参数用来控制poll方法的阻塞时间。如果为0，poll立即返回，否则会在指定的毫秒数内一直等待broker返回数据
>
>实例代码
>
>> ```java
>> public class ConsumerDemo {
>> 
>>     private Properties kafkaProp = new Properties();
>>     private KafkaConsumer<String,String> consumer;
>> 
>>     public void init(){
>>         kafkaProp.put("bootstrap.servers","localhost:9092");
>>         kafkaProp.put("key.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
>>         kafkaProp.put("value.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
>>         kafkaProp.put("group.id","consumerGroup");
>>         consumer = new KafkaConsumer(kafkaProp);
>>     }
>> 
>>     public void getMessage(){
>>         consumer.subscribe(Collections.singletonList("CustomerCountry"));
>>         try {
>>             while (true) {
>>                 //轮询获取信息
>>                 ConsumerRecords<String,String> records = consumer.poll(100L);
>>                 for(ConsumerRecord<String,String> record : records){
>>                     System.out.println("Entry...");
>>                     System.out.println("record-key"+record.key());
>>                     System.out.println("record-value"+record.value());
>>                 }
>> 
>>             }
>>         }finally {
>>             consumer.close();
>>         }
>> 
>>     }
>> 
>>     public static void main(String[] args) {
>>         ConsumerDemo demo = new ConsumerDemo();
>>         demo.init();
>>         demo.getMessage();
>>     }
>> }
>> ```

消费者参数

> fetch.min.bytes
>
> > 指定了消费者从服务器获取记录的最小字节数
> >
> > broker收到消费者请求时，如果可用的数据量小于该参数，会等到有足够的可用数据时才把它返回给消费者。
>
> fetch.max.wait.ms
>
> > 通过fetch.min.bytes参数告诉Kafka，等到有足够的数据时才返回给消费者。
> >
> > 而该参数则用于指定：broker的等待时间，默认为500ms。
> >
> > Kafka要么返回fetch.min.bytes指定的数据量，要么延迟指定的时间后，返回。
>
> max.partition.fetch.bytes
>
> >指定了服务器从每个分区返回给消费者的最大字节数。默认是1MB
>
> session.timeout.ms
>
> >指定了消费者在被认为死亡之前可以与服务器断开连接的时间。默认是3s
>
> auto.offset.reset
>
> >指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下，该如何处理
> >
> >默认是latest。意思：在偏移量无效时，消费者从最新的记录开始读取数据
> >
> >另一个earliest。意思：在偏移量无效时，从起始位置读取分区的记录
>
> enable.auto.commit
>
> > 该属性指定了消费者是否自动提交偏移量。默认是true.
>
> partition.assignment.strategy
>
> > 分区会被分配给群组里的消费者。PartitionAssignor根据给定的消费者和主题。决定哪些分区应该被分配给那个消费者。Kafka有两个默认的分配策略
> >
> > > Pange
> > >
> > > RoundRobin
>
> client.id
>
> > 该属性可以是任意字符串。broker用它来标识客户端发送过来的消息
>
> max.poll.records
>
> > 该属性用于单词调用call方法能够返回的记录数量
>
> receive.buffer.bytes和send.buffer.bytes
>
> > TCP缓冲区大小



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

## 流式处理

什么是数据流(也称之为：事件流或流数据)

特点：

1、数据流是无边界数据集的抽象表示
	无边界意味着无限和持续增长。
2、事件流是有序的
3、不可变的数据记录
4、事件流是可重播的

###  流式处理的一些概念

流式处理的很多方面与普通的数据处理时相似的：

写一些代码来接收数据

对数据进行处理（可能做一些转换，聚合和增强的操作）

然后把结果输出到某些个地方

#### 时间窗口

> > > 大部分针对流的操作都是基于时间窗口的，比如移动平均数、一周内销量最好的产品，系统的99百分位等。
> > >
> > > 两个流的合并操作也是基于时间窗口的，我们会合并相同时间片段上的事件。
> > >
> > > 示例：在计算平均数时，需要知道一下几个问题
> > >
> > > > 窗口的大小
> > > >
> > > > > 是基于5分钟进行平均，还是15分钟。
> > > > >
> > > > > 窗口越小，越能越快的发现变更。
> > > > >
> > > > > 窗口越大，变更越平缓，不过延迟越小
> > > >
> > > > 窗口的移动频率（移动间隔）
> > > >
> > > > > 5分钟的平均数可以每分钟变化一次，或者每秒钟变化一次，或者每当有新事件到达时发生变化。
> > > > >
> > > > > 如果移动间隔与窗口大小相等，这种情况被称为”滚动窗口“
> > > > >
> > > > > 如果窗口随每一条记录移动，这种情况称之为“滑动窗口”
> > > >
> > > > 窗口的可更新时间多长
> > > >
> > > > > 假设计算了00:00~00:05之间的移动平均数，一个小时后又得到了一些事件时间是00:02的事件，那么需要更新00:00~00:05这个窗口的结果吗？
> > > > >
> > > > > 或者就这么算了？理想情况下，可以定义一个时间段，在这个时间段内，事件可以被添加到与他们相应的时间片段里。否则，则忽略它们
> >

### 流式处理的设计模式

#### 单个事件处理

#### 使用本地状态

#### 多阶段处理和重分区

#### 使用外部查找—流和表的连接

#### 流与流的连接

#### 乱序的事件

#### 重新处理































