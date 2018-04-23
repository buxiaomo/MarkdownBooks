# Dockerfile实践指南

# 满足以下准则编写
1、选择合适的基础镜像

2、镜像大小

3、交付大小

4、尽量减少层级

## 选择合适的基础镜像

### 并不是任何环境都适合使用alpine

​	目前来看，alpine镜像仅仅才5M大小，如果官方有alpine的镜像可以进行测试然后上线，如果官方没有镜像，那自己测试之后再决定用啥。

举个栗子：

​	Kafka、Zookeeper这类服务，我刚开始尝试的时候就是使用的alpine镜像，但是总会有莫名其妙的报错，最后放弃，选择了java:8u111-jre版本的镜像。

### 不要使用ImageID或latest写Dockerfile

​	最近看到很多人都是使用的ImageID写Dockerfile，虽然我不知道为什么会有这样的想法，但是我不建议这样。首先，写ImageID这个方法没错，但是从长期角度来看，你今天写的可能知道你引用的是那个镜像，后天也知道，时间长了呢。你还知道吗？说白了，不方便你后期的管理和更新，所以不建议。

​	不要使用latest tag本质和上面说的一样。自己琢磨去吧。不听的话，后面踩到坑不能怪我没提醒你。

### 不要重复造轮子

​	如果你的是一个java应用，那么你应该使用java作为基础应用，如果你是tomcat应用，你应该使用tomcat作为基础应用，而不是按照虚拟机的思维，把Java装好，然后装应用；tomcat也一样，装java，装tomcat，装应用。。。

​	首先你这样写是没问题，但是没必要。你这样做只会增加你的时间成本，其次，java更新一个版本或者修复一个补丁，你也要更新自己的版本，同样也增加了时间成本。

## 镜像大小

​	并不是镜像越小越好，而是满足运行时环境的镜像越小越好。

举个栗子：

​	ubuntu下安装软件会使用apt-get update去更新缓存，当你安装完之后这些环境对于你的应用来说是没用的，所以删掉。再或者，你下载的压缩文件，但只需要里面的二进制文件，那么这个压缩文件也是多余的东西。

```
ENV ZOOKEEPER_VERSION 3.4.10
RUN apt-get update \
    && apt-get install axel --no-install-recommends -y \
    && axel -n 20 --output=/usr/local/src/zookeeper-${ZOOKEEPER_VERSION}.tar.gz http://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-${ZOOKEEPER_VERSION}/zookeeper-${ZOOKEEPER_VERSION}.tar.gz \
    && tar -zxf /usr/local/src/zookeeper-${ZOOKEEPER_VERSION}.tar.gz -C /usr/local/ \
    && rm -rf /usr/local/src/zookeeper-${ZOOKEEPER_VERSION}.tar.gz \
    && apt-get remove axel -y \
    && apt-get clean all \
    && rm -rf /var/lib/apt/lists/*
```

## 交付大小

​	这里的交付大小是指把你的Dockerfile所在目录打包（排除应用代码）之后的大小，一般不超过1-10M。

​	为什么这样说呢，你写了一个tomcat的运行环境的镜像，内容如下

```
FROM ubuntu:16.04
COPY tomcat.tar.gz /opt/
COPY java.tar.gz /opt/
.....
```

一个java软件包200M，一个tomcat几十兆，内网无所谓，公网呢，gitlab、github呢，构建的时候呢，时间全浪费了，，体现不出Docker的敏捷。。。，具体我什么意思，自己琢磨吧。

## 尽量合并层级

实例1：

```
RUN apt-get update \
    && apt-get install axel --no-install-recommends -y \
    && axel -n 20 --output=/usr/local/src/zookeeper-${ZOOKEEPER_VERSION}.tar.gz http://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-${ZOOKEEPER_VERSION}/zookeeper-${ZOOKEEPER_VERSION}.tar.gz \
    && tar -zxf /usr/local/src/zookeeper-${ZOOKEEPER_VERSION}.tar.gz -C /usr/local/ \
    && rm -rf /usr/local/src/zookeeper-${ZOOKEEPER_VERSION}.tar.gz \
    && apt-get remove axel -y \
    && apt-get clean all \
    && rm -rf /var/lib/apt/lists/*
```

实例2：

```
RUN apt-get update \
RUN apt-get install axel --no-install-recommends -y
RUN axel -n 20 --output=/usr/local/src/zookeeper-${ZOOKEEPER_VERSION}.tar.gz http://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-${ZOOKEEPER_VERSION}/zookeeper-${ZOOKEEPER_VERSION}.tar.gz
RUN tar -zxf /usr/local/src/zookeeper-${ZOOKEEPER_VERSION}.tar.gz -C /usr/local/
RUN rm -rf /usr/local/src/zookeeper-${ZOOKEEPER_VERSION}.tar.gz
RUN apt-get remove axel -y
RUN apt-get clean all
RUN rm -rf /var/lib/apt/lists/*
```

实例1为合并层级，实例2为没有合并的，实例2中最后4行相当于没执行。为什么这样呢，因为每次你RUN的时候是在新的层级上面，而不是你想删除的地方。