## SpringCloud

#### 1、微服务

##### 微服务应该具备的功能

* 服务的注册和发现
* 服务的负载均衡
* 服务的容错
* 服务的网关
* 服务配置的统一管理
* 链路追踪
* 实时日志

#### 2、简介

​	spring cloud的首要目标是通过一系列的开发组件和框架，帮助开发者迅速搭建一个分布式的微服务系统

​	spring cloud提供了一些常用的组件，如：服务注册和发现，配置中心，熔断器，智能路由，微代理，控制总线，全局锁，分布式会话等。



## Spring cloud Netflix组件

### 服务注册与发现—Eureka

#### 1、是什么

​	和zookeeper类似，Euraka是一个用于服务注册和发现的组件。

#### 2、基本架构

​	Euraka主要由3中角色组成：

+ Register Service：服务注册中心，是一个Euraka Service,提供服务注册和发现的功能

+ Provider Service : 服务提供者，是一个Euraka Client，提供服务

+ Consumer Service:服务消费者，是一个Euraka Client，消费服务

  ```shell
  # 注册过程如下：
  # 1、首先需要，一个服务注册中心
  # 2、服务提供者向服务注册中心注册，将自己的信息(服务名和服务的IP地址等)，通过REST API提交给服务注册中心
  # 3、服务消费者，也向服务中心注册，同时获取一份服务注册列表的消息，该列表包含了所有向注册中心注册的服务消息，获取列表后，消费者就知道服务提供者的IP地址，然后通过Http远程调用
  ```

#### 3、解析Eureka

##### 1、概念

###### (1) Register——服务注册

​	当Eureka Client向Eureka Service 注册时，Eureka Client提供自身的元数据。

###### (2)Renew——服务续约

​	Eureka Client在默认情况下，每隔30秒发送一次心跳来进行服务续约。通过服务续约来告知Eureka Sever该Eureka Client依旧可用。

​	正常情况下，如果Eureka Service在90s内没有收到Eureka Client的心跳，会将该实例从注册列表删除。

###### (3) Fetch Registries——获取服务注册列表信息

​	Eureka Client从Eureka Server获取注册表信息，并缓存到本地

###### (4) Cancel——服务下线

​	关闭程序时，可以向Eureka Server附送下线请求。需要调用如下代码：

```java
DiscoveryManager.getInstance().shutdownComponent();
```

###### (5)Eviction——服务剔除

​	90内没有发送心跳，server会将服务剔除

#### 4、demo

```shell
# EurekaTest
#	--eureka-server
#	--eureka-client

# demo采用Maven多Modele的结构
```

##### 1、主pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cn.shengling</groupId>
    <artifactId>eureka.test</artifactId>
    <version>1.0-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.3.RELEASE</version>
        <relativePath/>
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

#####  2、eurekaServer

###### 1、pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>cn.shengling</groupId>
		<artifactId>eureka.test</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>

	<groupId>cn.shengling</groupId>
	<artifactId>eurekaserver</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>eurekaserver</name>
	<description>Demo project for Spring Boot</description>
    
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>
	</dependencies>
</project>

```

###### 2、application.properties

```properties
server.port=8761
eureka.instance.hostname=localhost
//防止自己注册自己
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.service-url.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
spring.application.name=eureka-server
```

###### 3、启动类

```java
package cn.shengling.eurekaserver;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer   //开启EurekaServer功能
public class EurekaserverApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaserverApplication.class, args);
	}

}
```

##### 3、Eurekaclient

###### 1、pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>cn.shengling</groupId>
		<artifactId>eureka.test</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>

	<groupId>cn.shengling</groupId>
	<artifactId>eurkaclient</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>eurkaclient</name>
	<description>Demo project for Spring Boot</description>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>

</project>
```

###### 2、application.properties

```properties
server.port=8763
spring.application.name=cli-hi
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

###### 3、启动类

```java
package cn.shengling.eurkaclient;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@SpringBootApplication
@EnableEurekaClient //开启Eureka Client功能
public class EurkaclientApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurkaclientApplication.class, args);
	}

}
```

### 负载均衡—Ribbon

#### 1、RestTemplate简介

​	RestTemplate 是spring Resources中一个访问第三方RESTful API接口的网络请求框架。

​	RestTemplate支持常见的Http协议的请求方法，如Post,put,delete等

​	RestTemplate支持xml，json数据格式，可以自动将JSON字符转换为实体

#### 2、Ribbon简介

​	负载均衡：将负载分摊到多个执行单元上，常见的负载均衡有两种方式：一种是独立进程单元，通过负载均衡策略，将请求转发到不同的执行单元上，例如：Nginx;另一种是将负载均衡逻辑以代码的形式封装到服务消费者的客户端上，服务消费者客户端维护一份服务提供者的信息列表。

​	Ribbon，就是属于第二种的负载均衡，将负载均衡逻辑封装在客户端中，并且运行在客户端的进程里。

​	在Spring Cloud构建的微服务中，Ribbon作为服务消费者的负载均衡器，有两种使用方式，一种时和RestTemplate向结合，另一种是和Feign向结合。

​	Fegin已经默认集成了Ribbon.

#### 3、Demo

##### 1、pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>cn.shengling</groupId>
		<artifactId>eureka.test</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>
	<groupId>cn.shengling</groupId>
	<artifactId>ribbontest</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>ribbontest</name>
	<description>Demo project for Spring Boot</description>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>
</project>

```

##### 2、application.properties

```properties
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
server.port=8764
spring.application.name=robbon-server
```

##### 3、启动类

```java
package cn.shengling.ribbontest;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
public class RibbontestApplication {

	public static void main(String[] args) {
		SpringApplication.run(RibbontestApplication.class, args);
	}

	@Bean
	@LoadBalanced
	RestTemplate restTemplate(){
		return new RestTemplate();
	}
}
```

##### 4、server

```java
package cn.shengling.ribbontest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

/**
 * @author shengling23
 * @create 2018-12-31 13:46
 */
@Service
public class HelloTest {
    @Autowired
    private RestTemplate restTemplate;

    public String hiServer(String name){
        return restTemplate.getForObject("http://CLI-HI/hi?name="+name,String.class);
    }
}
```

### 远程调用—Feign

#### 1、简介

​	Feign采用了声明式API接口的风格，将java Http客户端绑定到它内部。

​	Feign首要目标是将java Http客户端的调用过程变的简单

#### 2、Demo

##### 1、pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>cn.shengling</groupId>
		<artifactId>eureka.test</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>

	<groupId>cn.shengling</groupId>
	<artifactId>feigntest</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>feigntest</name>
	<description>Demo project for Spring Boot</description>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-openfeign</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>
</project>

```

##### 2、application.properties

```properties
spring.application.name=eureka-feign-client
server.port=8765
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

##### 3、启动类

```java
package cn.shengling.feigntest;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
@EnableFeignClients
public class FeigntestApplication {

	public static void main(String[] args) {
		SpringApplication.run(FeigntestApplication.class, args);
	}

}
```

##### 4、调用远程服务接口

```java
package cn.shengling.feigntest;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

/**
 * @author shengling23
 * @create 2019-01-12 16:20
 */
@FeignClient("CLI-HI") //远程调用的服务名
public interface ServerHi {

    //远程调用服务的接口
    @RequestMapping(value = "/hi",method = RequestMethod.GET)
    String sayHi(@RequestParam("name") String name);
}
```

##### 5、本地使用

```java
package cn.shengling.feigntest;

import com.netflix.discovery.converters.Auto;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author shengling23
 * @create 2019-01-12 16:24
 */
@RestController
public class FetchController {

    //编译器报错，无视；因为这个bean是在启动的时候注入的，编译器感知不到，所以报错
    @Autowired
    ServerHi serverHi;

    @GetMapping("/hi")
    public String sayHi(@RequestParam("name") String name){
       return serverHi.sayHi(name);
    }
}
```

###  熔断器—Hystrix

#### 1、是什么？

​	在分布式的服务中，服务于服务之间的依赖错综复杂，一种不可避免的情况就是某些服务会出现故障，导致其他依赖它们的其他服务出现远程调度的线程阻塞。

​	Hystrix提供了熔断器功能，能够阻止分布式系统中出现联动故障。

​	Hystrix是通过隔离服务的访问点阻止联动障碍的，并提供了障碍的解决方案，从而提高了整个分布式的弹性

#### 2、解决了什么问题？

​	在高并发的情况下，单个服务的延迟会导致整个请求都处于延迟状态。

​	某个服务的但个点的请求故障会导致用户的请求处于阻塞状态，最终导致整个服务的线程资源耗尽，由于微服务的依赖性，从而导致依赖该故障服务的其他服务也处于阻塞状态，从而导致整个服务的不可用，即雪崩效应。

#### 3、设计原则

+ 防止单个服务的故障耗尽整个服务的Servlet容器的线程资源
+ 快速失败机制，如歌某个服务出现了故障，则调用该服务的请求快速失败，而不是线程等待
+ 提供回退（fallback）方案，在请求发生故障时，提供设定好的回退方案
+ 使用熔断机制，防止故障扩散到其他服务
+ 提供熔断器的监视组件Hystrix Dashboard，可以实时监控熔断器的状态

#### 4、工作机制

​	1. 首先，当服务的某个API接口的失败次数在一定时间内小于设定的阈值时，熔断器处于关闭状态，该API正常提供服务

​	2、当该API接口处理请求的失败次数大于设定的阈值，Hystrix判定该API接口出现了故障，打开熔断器，这时请求该API接口会执行快速失败的逻辑（fallback）,不执行业务逻辑，请求的线程不会处于阻塞状态

​	3、处于打开状态的熔断器，一段时间后会处于半打开状态，并将一定数量的请求执行正常的逻辑，剩余的请求会执行快速失败；若执行正常的逻辑的请求失败了，则熔断器继续打开，若成功了，则熔断器关闭

```shell
# 阈值  5s 20次
```

#### 5、在RestTemplate 和Ribbon上使用熔断器

##### 1、pom文件的改动

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

##### 2、启动类的修改

```java
package cn.shengling.ribbontest;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
@EnableHystrix   //新增，开启Hystrix的熔断器功能
public class RibbontestApplication {

	public static void main(String[] args) {
		SpringApplication.run(RibbontestApplication.class, args);
	}

	@Bean
	@LoadBalanced
	RestTemplate restTemplate(){
		return new RestTemplate();
	}
}
```

##### 3、调用服务的方法修改

```java
package cn.shengling.ribbontest;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

/**
 * @author shengling23
 * @create 2018-12-31 13:46
 */
@Service
public class HelloTest {
    @Autowired
    private RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "hiError") //新增
    public String hiServer(String name){
        return restTemplate.getForObject("http://CLI-HI/hi?name="+name,String.class);
    }

    public String hiError(String name){
        return "hi,"+name+",sorry,error";
    }
}
```

##### 4、问题

​	在一接口存在对个服务时，若其中一个断开，另一个还能用，则会轮询，一个能正常访问，一个会快速失败，访问无法全部到达能正常访问的服务上。

#### 6、在Feign上使用熔断器

##### 1、说明

​	由于Feign的起步依赖中已经引入了Hystrix的依赖，所以在Feign中使用Hystrix不需要引入依赖

##### 2 、application.properties修改

```properties
spring.application.name=eureka-feign-client
server.port=8765
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
#开启hystrix的功能
feign.hystrix.enabled=true
```

##### 3、调用远程服务的接口修改

```java
package cn.shengling.feigntest;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

/**
 * @author shengling23
 * @create 2019-01-12 16:20
 */
@FeignClient(value ="CLI-HI",fallback=ServerHiError.class)
//添加fallback参数
public interface ServerHi {

    @RequestMapping(value = "/hi",method = RequestMethod.GET)
    String sayHi(@RequestParam("name") String name);
}
```

##### 4、创建fallback的类

```java
package cn.shengling.feigntest;
import org.springframework.stereotype.Component;

/**
 * @author shengling23
 * @create 2019-01-12 17:50
 */
@Component
public class ServerHiError implements ServerHi {
    @Override
    public String sayHi(String name) {
        return name+",error";
    }
}
```

####  7、在Ribbon中使用Hystrix Dashboard监视熔断器状态

##### 1、pom文件

​	加入Actuator的起步依赖，Hystrix Dashboard的起步依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

##### 2、启动类

```java
package cn.shengling.ribbontest;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
@EnableHystrix
@EnableHystrixDashboard  //开启Hystrix Dashboard功能
public class RibbontestApplication {

	public static void main(String[] args) {
		SpringApplication.run(RibbontestApplication.class, args);
	}

	@Bean
	@LoadBalanced
	RestTemplate restTemplate(){
		return new RestTemplate();
	}
    
    //新增，否则，会出现hystrix dashboard Unable to connect to Command Metric Stream
    @Bean
	public ServletRegistrationBean getServlet() {
		HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
		ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
		registrationBean.setLoadOnStartup(1);
		registrationBean.addUrlMappings("/hystrix.stream");
		registrationBean.setName("HystrixMetricsStreamServlet");
		return registrationBean;
	}
}
```

##### 3 网址

```http
http://localhost:8764/hystrix
```

#### 8、在Feign中使用Hystrix Dashboard

#####  1、pom文件

​	引入起步依赖：Actuator,Hystrix和Hystrix Dashboard

```xml
<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
		</dependency>
```

##### 2、启动类

```java
package cn.shengling.feigntest;

import com.netflix.hystrix.contrib.metrics.eventstream.HystrixMetricsStreamServlet;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
@EnableHystrixDashboard
public class FeigntestApplication {

    public static void main(String[] args) {
        SpringApplication.run(FeigntestApplication.class, args);
    }

    @Bean
    public ServletRegistrationBean getServlet() {
        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
        registrationBean.setLoadOnStartup(1);
        registrationBean.addUrlMappings("/hystrix.stream");
        registrationBean.setName("HystrixMetricsStreamServlet");
        return registrationBean;
    }
}
```

#### 使用Turbine聚合监控

### 路由网关Zuul

#### 1、为什么需要zuul

+ zuul、ribbon以及Eureka相结合，可以实现智能路由和负载均衡的作用
+ 网关能够将所有服务的API统一聚合，并统一对外暴露。
+ 网关服务可以做用户身份认证和权限认证，防止非法请求操作API接口
+ 网关可以实现监控功能，实时日志输出，对请求进行记录
+ 可以实现流量监控，在高流量的情况下，对服务进行降级

#### 2、工作原理

​	Zuul是用过Servlet来实现的。Zuul通过自定义的ZuulServlet来对请求进行控制。

​	Zuul的核心是一系列的过滤器，可以在http请求的发起和响应返回期间执行一系列的操作

​	Zuul包括以下四种过滤器

| 名称          | 作用                                                         |
| ------------- | ------------------------------------------------------------ |
| PRE过滤器     | 他是在请求路由到具体的服务之前进行的，这种过滤器可以做安全验证，例如身份验证，参数验证 |
| ROUTING过滤器 | 它用于将请求路由到具体的微服务实例。默认请求下，使用Http Client进行网络请求 |
| POST过滤器    | 它是在请求已经被路由到微服务后执行的。一般情况下，用作收集统计信息，指标，以及将响应传输到客户端 |
| ERROR过滤器   | 它是在其他过滤器发生错误时执行的                             |

​	Zuul采取动态读取，编译和执行这些过滤器。这些过滤器之间不能相互直接通信，而是通过RequestContenxt对象来共享数据，每个请求都会创建一个RequestContext对象。

​	Zuul过滤器对象具有以下关键特性：

+ Type(类型)：过滤器的类型，决定了过滤器在哪个阶段起作用

+ Execution Order(执行顺序) ：规定了过滤器的执行顺序，Order越小，越先执行

+ Criteria(标准)：Filter执行所需条件

+ Action(行动) ：如果符合执行条件，则执行Action

     Zuul请求的生命周期：当一个Request请求进入Zuul网关服务时，首先进入“pre filter"，进行一系列的验证、操作或者判断。然后交给“routing filter" 进行路由转发，转发到具体的服务实例进行逻辑处理、返回数据。最后由"post filter"进行处理，将Response返回。

#### 3、demo

##### 1、pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>cn.shengling</groupId>
		<artifactId>eureka.test</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>
	<groupId>cn.shengling</groupId>
	<artifactId>zuultest</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>zuultest</name>
	<description>Demo project for Spring Boot</description>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
</project>
```

##### 2、application.yml

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
server:
  port: 5000
spring:
  application:
    name: service-zuul
zuul:
  routes:
    hiapi:
      path: /hiapi/**
      serviceId: CLI-HI
    ribbonapi:
      path: /ribbonapi/**
      serviceId: robbon-server

    feignapi:
      path: /feignapi/**
      serviceId: eureka-feign-client
```

##### 3、启动类

```java
package cn.shengling.zuultest;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@SpringBootApplication
@EnableEurekaClient
@EnableZuulProxy
public class ZuultestApplication {

	public static void main(String[] args) {
		SpringApplication.run(ZuultestApplication.class, args);
	}

}
```

#### 4、在Zuul上配置熔断器

​	在zuul中实现熔断功能需要实现：FallbackProvider接口。

```java
package cn.shengling.zuultest;

import org.springframework.cloud.netflix.zuul.filters.route.FallbackProvider;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.client.ClientHttpResponse;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;

/**
 * @author shengling23
 * @create 2019-01-13 15:26
 */
@Component
public class MyFallbackProvider implements FallbackProvider{
    //在哪个路由上加熔断器，如果所有路由服务全部加熔断器，则 return "*";
    @Override
    public String getRoute() {
        return "CLI-HI";
    }

    @Override
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.OK;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return 200;
            }

            @Override
            public String getStatusText() throws IOException {
                return "ok";
            }

            @Override
            public void close() {

            }

            @Override
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream("fallback".getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                return new HttpHeaders();
            }
        };
    }
}
```

#### 5、在Zuul中使用过滤器

```java
package cn.shengling.zuultest;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.netflix.zuul.exception.ZuulException;
import org.springframework.stereotype.Component;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

/**
 * @author shengling23
 * @create 2019-01-13 15:52
 */
@Component
public class MyFilter extends ZuulFilter {
    @Override
    public String filterType() {
        //参数有：pre ,post,routing,error
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        Object token = request.getParameter("token");
        if(null == token){
            ctx.setSendZuulResponse(false);//不再往下走
            ctx.setResponseStatusCode(401);//返回码
            try {
                ctx.getResponse().getWriter().write("token is enpty");
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return null;
    }
}

```

## 配置中心—Spring Cloud Config

### 描述

​	spring cloud config的指示，分为4个部分

+ Config Server从本地读取配置文件
+ Config Server从远程Git读取配置文件
+ 搭建高可用的Config Server集群
+ 使用spring Cloud bus刷新配置

### 1、从本地读取配置文件

#### 1、服务端

##### 1、pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>cn.shengling</groupId>
		<artifactId>springconfigtest</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>
	<groupId>cn.shengling</groupId>
	<artifactId>configservertest</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>configservertest</name>
	<description>Demo project for Spring Boot</description>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
</project>

```

##### 2、appliction.properties

```properties
spring.cloud.config.server.native.search-locations=classpath:/shared
spring.profiles.active=native
spring.application.name=config-server
server.port=8769
```

##### 3、shared下的文件--config-client-dev.yml

```yml
server:
  port: 8762
foo: foo version 1
```

##### 4、启动类

```JAVA
package cn.shengling.configservertest;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer  //开启功能
public class ConfigservertestApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigservertestApplication.class, args);
	}

}
```

#### 2、客户端

##### 1、pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>cn.shengling</groupId>
		<artifactId>springconfigtest</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>
	<groupId>cn.shengling</groupId>
	<artifactId>configclienttest</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>configclienttest</name>
	<description>Demo project for Spring Boot</description>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>
</project>
```

##### 2、bootstrap.properties(不是application了)

```properties
spring.application.name=config-client
#服务的地址
spring.cloud.config.uri=http://localhost:8769
spring.cloud.config.fail-fast=true
spring.profiles.active=dev
#{spring.application.name}-${spring.profiles.active}，构成了向Config Server读取的配置文件名
```

##### 3、启动类

```java
package cn.shengling.configclienttest;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class ConfigclienttestApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigclienttestApplication.class, args);
	}

	@Value("${foo}")
	String foo;

	@RequestMapping("/foo")
	public String hi(){
		return foo;
	}
}
```



### 2、从远程GIT仓库读取

#### 1、说明

​	好处是：将配置同意进行管理，并且可以通过Spring Cloud Bus在不人工启动程序的情况下对Config Client的配置进行刷新

#### 2、服务端配置文件的修改

```properties
spring.application.name=config-server
server.port=8769
# git地址
spring.cloud.config.server.git.uri=https://github.com/ShengLing23/SpringCloud
#文件夹
spring.cloud.config.server.git.search-paths=config
#git的分支名
spring.cloud.config.label=master
# 如果是私有仓库
# spring.cloud.config.server.git.username和spring.cloud.config.server.git.password 
# 是必须添加的
```

### 3、高可用

​	将Config Server和Config Client 都作为Eureka client 即可

### 4、使用 Spring Cloud Bus刷新配置

#### 1、spring cloud bus 简介

​	Spring cloud bus使用轻量的消息代理将分布式的节点连接起来，可以用来广播配置文件的更改或者服务的监控管理。

#### 2、为什么使用

​	如果有几十个微服务，而每一个微服务又是多实例，当更改配置时，需要重新启动多个微服务实例，非常麻烦。spring cloud bus的一个功能可以让这个过程变的简单

#### 3、修改

​	修改客户端config-client工程

##### 1、pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>cn.shengling</groupId>
		<artifactId>springconfigtest</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>
	<groupId>cn.shengling</groupId>
	<artifactId>configclienttest</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>configclienttest</name>
	<description>Demo project for Spring Boot</description>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
        <!- 引入spring cloud bus的起步依赖-->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-bus</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>
</project>
```

##### 2、未完

​	