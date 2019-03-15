# Linux下常用操作

## 1、时区设置

```shell
data -R 
Wed, 02 Jan 2019 22:00:10 -0500
# -0500 西五区，我们是东八区，相差13个小时
#系统对应的时区文件存放在：/etc/localtime 文件中
#每个时区对应的文件存放在：/usr/share/zoneinfo/目录下
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime 
# 修改为上海时区
```

## 2、jvm时区

```shell
#修改JVM获取时区的配置文件 /etc/timezone文件
# vim /etc/timezone
Asia/Shanghai
```

## 3、redis安装

```shell
# 先将redis压缩包解压 本地位置：/usr/local/reids
tar -zxvf  redis-5.0.3.tar.gz
# 进入redis-5.0.3目录,运行make命令
make
# 报错，gcc命令不存在
make[3]: gcc: Command not found
make[3]: *** [net.o] Error 127
make[3]: Leaving directory `/usr/local/redis/redis-5.0.3/deps/hiredis'
make[2]: *** [hiredis] Error 2
make[2]: Leaving directory `/usr/local/redis/redis-5.0.3/deps'
make[1]: [persist-settings] Error 2 (ignored)
    CC adlist.o
/bin/sh: cc: command not found
make[1]: *** [adlist.o] Error 127
make[1]: Leaving directory `/usr/local/redis/redis-5.0.3/src'
make: *** [all] Error 2
# 安装gcc命令
yum install gcc
# 去掉第一次安装过程中的残余文件
make distclean
# 清理完成后，再次运行make命令
make
# 再次运行
make install

```

## 4、JDK安装

```shell
# vim /etc/profile
export JAVA_HOME=/selfPath
export JRE_HOME=/selfPath
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME:bin
```



# 命令



