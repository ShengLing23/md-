# 架构

![1551667792172](.\img\1551667792172.png)

节点角色说明：

* Provider：服务提供方
* Consumer：服务消费者
* Registry：服务注册与发现中心
* Monitor：监控中心
* Container：服务运行容器

调用关系说明：

* 0、服务容器负责启动，加载、运行服务提供者
* 1、服务提供者在启动时，向注册中心注册自己提供的服务
* 2、服务消费者在启动时，向注册中心订阅自己所需的服务
* 3、注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者
* 4、服务消费者，从提供地址列表总，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用
* 5、服务消费者和提供者，在内存中累积调用次数和调用时间，定时每分钟发送一次统计数据到监控中心

连通性：

* 服务消费者和提供者，只在启动时与注册中心进行交互
* 服务消费者、注册中心、服务提供者三者间均未长连接。
* 注册中心通过长连接感知服务提供者的存在，服务提供者宕机，注册中心立即推送事件通知消费者
* 注册中心和监控中心全部宕机，不影响已运行的服务提供者和消费者，消费者在本地缓存了提供者列表

# 用法

## 本地服务与远程服务对比

1、本地服务（spring 配置）

```xml
<bean id="xxxService" class="com.xxx.xxxServiceImpl"/>
<bean id="xxxAction" class="com.xxx.xxxAction"/>
	<property name="xxxService" ref="xxxService" />
</bean>
```

2、远程服务（spring配置）

```xml
<!--和本地服务一样实现远程服务 -->
<bean id="xxxService" class="com.xxx.xxxServiceImpl"/>
<!--增加暴露远程服务配置 -->
<dubbbo:service interface="com.xxx.xxxService" ref="xxxService" />

```

```xml
<!-- 增加引用远程配置服务 -->
<dubbo:reference id="xxxService" interface="com.xxx.xxxService" />
<!--和本地服务一样使用远程服务 -->
<bean id="xxxAction" class="com.xxx.xxxAction"/>
	<property name="xxxService" ref="xxxService" />
</bean>
```

## 快速启动

### 服务提供者

1、定义服务接口（该接口单独打包），该接口在服务消费方和提供方，共享

2、在服务提供方实现接口。

3、3在spring配置声明暴露服务

![1551669545274](.\img\1551669545274.png)

### 服务消费者

1、在spring配置中，引入远程配置

![1551669682348](.\img\1551669682348.png)

2、需要调用的远程服务接口，服务消费端也得存在

## 启动时检查

* 【dubbo】缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止Spring初始化完成。默认```check=true```
* 如果Spring容器是懒加载，请关闭check，否则服务临时不可用时，会抛异常，拿到Null引用。如果check=false，总是会返回引用，当服务恢复时，能自动连上

```properties
# 关闭某个服务的检查
# <dubbo:reference interface="com.foo.BarService" check="false" />
dubbo.reference.com.foo.BarService.check=flase
dubbo.reference.check=false
# 关闭所有服务的启动时检查
# <dubbo:consumer check="false">
dubbo.consumer.check=false

# 关闭注册中心启动时检测
# <dubbo:registry check="false">
dubbo.registry.check=false
```

【注意】

* dubbo.reference.check=false 强制改变所有的reference的check值。即是配置中有声明，也会被覆盖
* dubbo.consumer.check=false 是设置check的缺省值，如果配置中有显示声明，不受其影响
* dubbo.registry.check=false 前面两个都是指订阅成功，当提供者列表是否为空是否保存。如果注册订阅失败时，也允许启动，则使用此选项

## 集群容错

![1551671089504](.\img\1551671089504.png)

节点关系：

* Invoker：是provider的一个可调用Service的抽象，Invoker封装了Provider地址及Service接口信息
* Directory：代表了多个Invoker,可以把它看成List<Invoker>,但是与List不同的是，它的值可能是动态变化的
* Cluster将Directory中的多个Invoker伪装成一个Invoker，对上层透明。
* Router：负责从多个Invoker中按路由规则选出子集，比如读写分离，应用隔离等
* LoadBalance：负责从多个Invoker中选出具体的一个用于本次调用。选的过程包含了负载均衡算法，调用失败后，需要重新选择

### 容错方式

* Failover Cluster（缺省）
  + 失败自动切换，当出现失败，重试其他服务器
  + 通常用于读操作，重试会带来更长的延迟
  + 可以通过```retries="2"``` 来设置重试次数（不含第一次）
* Failfast Cluster
  * 快速失败，只发起一次调用，失败立即报错。
  * 通常用于非幂等性的写操作，比如新增记录
* Failsafe Cluster
  * 失败安全，出现异常时，直接忽略
  * 通常用于写入审计日志等操作
* Failback Cluster
  * 失败自动恢复，后台记录失败请求，定时重发
  * 通常用于消息通知等操作
* Focking Cluster
  * 并行调用多个服务，只要一个成功即返回
* Broadcast Cluster
  * 广播调用所有提供者，逐个调用，任意一台报错则报错

## 负载均衡

* Random LoadBalance （缺省）
  * 随机，按权重设置随机概率
* RoundRobin LoadBalance 
  * 轮询，按公约后的权重设置轮询比例
* LeastActive LoadBalance 
  * 最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差
* ConsistentHash LoadBalance 
  * 一致性Hash，相同参数的请求总是发到同一提供者







































































































