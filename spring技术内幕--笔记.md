

# spring技术内幕--笔记

## BeanFactory

### Ioc容器接口设计图

![BeanFactory](G:\md笔记\img\BeanFactory.png)

| 接口                       | 说明                                                         |
| -------------------------- | ------------------------------------------------------------ |
| BeanFactory                | 基本的IOC容器规范                                            |
| HierarchialBeanFactory     | 继承BeanFactory后，增加了getParentBeanFactory()方法，使得BeanFactory具备了双亲IOC容器的管理功能 |
| ConfigurableBeanFactory    | 主要定义了对BeanFactory的配置功能                            |
| ListableBeanFactory        | 细化了许多BeanFactory的接口功能                              |
| AutowireCapableBeanFactory |                                                              |

​	接口系统以BeanFactory和ApplicationContext为核心。

​	治理涉及的是主要的接口关系，而具体的Ioc容器都是在这个接口体系下实现的，比如：DefaultListableBeanFactory,这个基本的IOC容器的实现就是实现了ConfigurableBeanFactory，从而成为了一个简单的Ioc容器的实现。其他的Ioc容器，都是在DefaultListableBeanFactory的基础上做扩展，同样的，ApplicationContext的实现也是如此。

### 编程式使用IOC容器

​	通过编程式使用容器的过程，可以很清楚的揭示了在Ioc容器实现中哪些关键的类之间的相互关系。

```java
ClassPathResource res = new ClassPathResource("beans.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(res);
```

​	在使用IOC容器时，需要如下几个步骤：

​	1、创建配置文件的抽象资源，定位资源

​	2、创建beanFactory

​	3、创建一个beanDefinition的读取器，

​	4、从定义好的资源位置读入配置信息

### Ioc容器的初始化过程

#### 1、Resource定位过程

​	![filesystemXml](G:\md笔记\img\filesystemXml.png)





#### 2、BeanDefinition的载入过程

#### 3、向IOC容器注册这些BeanDefinition的过程































