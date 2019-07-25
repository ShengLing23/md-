# netty核心组件

## Channel

​	Channel是Java NIO的一个基本构造

​	它代表一个到实体的开放连接，如读操作和写操作

## 回调

## Future

​	Future提供了另一种在操作完成时通知应用程序的方式。

​	Netty提供了ChannelFuture,用于在执行异步操作的时候使用。

​	ChannelFuture提供了集中额外的方法，这些方法使得我们能够注册一个或者多个ChannelFutureListener实例。

​	监听器的回调方法operationComplete()，将会在对应的操作完成时被调用。

## 事件和ChannelHandler

​	Netty使用不同的事件来通知我们状态的改变或者时操作的状态。

​	每个事件都可以被分发给ChannelHandler类中的某个用户实现的方法。

# Netty的组件和设计

## Channel、EventLoop和ChannelFuture

*  Channel —— Socket
* EventLoop——控制流、多线程处理、并发
* ChannelFuture——异步通知

### 1、Channel

​	基本的IO操作(bind()、connect()、read()、write()）依赖底层网络传输提供的原语。在基于Java的网络编程中，基本的构造是Socket。

​	Netty提供的Channel接口所提供的API,大大的降低了直接使用Socket的复杂性。

### 2、EventLoop

​	EventLoop定义了Netty的核心抽象，用于处理连接的生命周期中所发生的事件。

![1552184121310](.\img\netty1.png)

* 一个EventLoopGroup包含一个或者多个EventLoop
* 一个EventLoop在它的生命周期内只和一个Thread绑定
* 所有由EventLoop处理的IO事件都将在它专有的Thread上被处理
* 一个Channel在它的生命周期内只能注册于一个EventLoop
* 一个EventLoop可能会分配给一个或多个Channel

### 3、ChannelFuture

​	nettty中所有的IO接口都是异步的。因为一个操作可能不会立即返回。所以我们需要一种在之后的某个时间点确定其结果的方法。

​	ChannelFuture接口，其addListtener()方法注册了一个ChannelFutureListener,以便在某个操作完成时得到通知。

## ChannelHander和ChannelPipeline

