## jdk源码阅读笔记

#### Object

##### 1、native关键字

```shell
# native 即 Java native interface
# 
# java10 以上废除了javah命令，使用javac -h 命令替代，命令格式为：
# javac -h  <生产文件地址>  源.java
```



##### 2、方法

```java
//native 方法
private static native void registerNatives();
public final native Class<?> getClass();
public native int hashCode();
protected native Object clone() throws CloneNotSupportedException;
public final native void notify();
public final native void notifyAll();
public final native void wait(long timeoutMillis) throws InterruptedException;
//方法实现
public boolean equals(Object obj)  //return (this == obj);
public String toString() //类名 + @ + 16进制的hashcode
public final void wait() throws InterruptedException // wait(0l)
public final void wait(long timeoutMillis, int nanos) throws InterruptedException
										// wait(timeoutMillis)
```



```
transient
```