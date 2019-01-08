## Linux下常用操作

### 1、时区设置

```shell
data -R 
Wed, 02 Jan 2019 22:00:10 -0500
# -0500 西五区，我们是东八区，相差13个小时
#系统对应的时区文件存放在：/etc/localtime 文件中
#每个时区对应的文件存放在：/usr/share/zoneinfo/目录下
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime 
# 修改为上海时区
```

### 2、jvm时区

```shell
#修改JVM获取时区的配置文件 /etc/timezone文件
# vim /etc/timezone
Asia/Shanghai
```



```java
public void static main(String[] args){
    
}
```





