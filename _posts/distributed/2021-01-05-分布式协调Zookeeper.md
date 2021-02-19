---
layout: post
category: 分布式
---
## 概述

> Zookeeper 是一个开源的分布式的，为分布式应用提供协调服务的Apache 项目。

Zookeeper从设计模式角度来理解：是一个基于观察者模式设计的分布式服务管理框架，它负责存储和管理大家都关心的数据，然后接受观察者的注册， 一旦这些数据的状态发生变化， Zookeeper就将负责通知已经在Zookeeper上注册的那些观察者做出相应的反应。

![image-20201204203149274](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201204203149274.png)

## 集群

![image-20201204203305759](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201204203305759.png)

采用ZAB共识算法：

1. Zookeeper：一个领导者（Leader），多个跟随者（Follower）组成的集群。
2.  集群中只要有半数以上节点存活，Zookeeper集群就能正常服务。
3. 全局数据一致：每个Server保存一份相同的数据副本，Client无论连接到哪个Server，数据都是一致的。
4. 更新请求顺序进行，来自同一个Client的更新请求按其发送顺序依次执行。
5. 数据更新原子性，一次数据更新要么成功，要么失败。
6. 实时性，在一定时间范围内，Client能读到最新数据。

## 安装

### 配置参数详解

Zookeeper中的配置文件zoo.cfg中参数含义解读如下：

**1．tickTime =2000：通信心跳数，Zookeeper 服务器与客户端心跳时间，单位毫秒**

Zookeeper使用的基本时间，服务器之间或客户端与服务器之间维持心跳的时间间隔， 也就是每个tickTime时间就会发送一个心跳，时间单位为毫秒。

它用于心跳机制，并且设置最小的session超时时间为两倍心跳时间。(session的最小超时时间是2*tickTime)

**2．initLimit =10：LF 初始通信时限**

集群中的Follower跟随者服务器与Leader领导者服务器之间初始连接时能容忍的最多心跳数（tickTime的数量），用它来限定集群中的Zookeeper服务器连接到Leader的时限。

**3．syncLimit =5：LF 同步通信时限**

集群中Leader与Follower之间的最大响应时间单位，假如响应超过syncLimit * tickTime，Leader认为Follwer死掉，从服务器列表中删除Follwer。

**4．dataDir：数据文件目录+数据持久化路径**

主要用于保存 Zookeeper 中的数据。

**5．clientPort =2181：客户端连接端口**

监听客户端连接的端口。

## API应用

略

## 应用场景

提供的服务包括：统一命名服务、统一配置管理、统一集群管理、服务器节点动态上下线、软负载均衡等。

### 统一命名服务

在分布式环境下，经常需要对应用/服务进行统一命名，便于识别。

例如：IP不容易记住，而域名容易记住

![image-20201205121146325](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201205121146325.png)

### 统一配置管理

![image-20201205121426303](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201205121426303.png)

1） 分布式环境下，配置文件同步非常常见。

（1） 一般要求一个集群中，所有节点的配置信息是一致的，比如 Kafka 集群。

（2） 对配置文件修改后，希望能够快速同步到各个节点上。

2）配置管理可交由ZooKeeper实现。

（1）可将配置信息写入ZooKeeper上的一个Znode。

（2） 各个客户端服务器监听这个Znode。

（3） 一旦Znode中的数据被修改，ZooKeeper将通知各个客户端服务器。

### 统一集群管理

![image-20201205121932856](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201205121932856.png)

1）分布式环境中，实时掌握每个节点的状态是必要的。

（1）可根据节点实时状态做出一些调整。

2） ZooKeeper可以实现实时监控节点状态变化

（1） 可将节点信息写入ZooKeeper上的一个ZNode。

（2） 监听这个ZNode可获取它的实时状态变化。

### 服务上下线

![image-20201205122056897](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201205122056897.png)

### 软负载均衡

![image-20201205122124546](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201205122124546.png)

## 内部原理

### 数据结构

ZooKeeper数据模型的结构与Unix文件系统很类似，整体上可以看作是一棵树，每个节点称做一

个ZNode。每一个ZNode默认能够存储1MB的数据，每个ZNode都可以通过其路径唯一标识

![image-20201204203454759](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201204203454759.png)

### 节点类型

**持久（Persistent）**：客户端和服务器端断开连接后，创建的节点不删除 

**短暂（Ephemeral）**：客户端和服务器端断开连接后，创建的节点自己删除

![image-20201204203727433](https://gitee.com/tostringcc/blog/raw/master/2020/image-20201204203727433.png)

说明：创建znode时设置顺序标识，znode名称后会附加一个值，顺序号是一个单调递增的计数器，由父节点维护

注意：在分布式系统中，顺序号可以被用于为所有的事件进行全局排序，这样客户端可以通过顺序号推断事件的顺序

1. 持久化目录节点客户端与Zookeeper断开连接后，该节点依旧存在
2. 持久化顺序编号目录节点客户端与Zookeeper断开连接后，该节点依旧存在，只是Zookeeper给该节点名称进行顺序编号
3. 临时目录节点客户端与Zookeeper断开连接后，该节点被删除
4. 临时顺序编号目录节点客户端与Zookeeper 断开连接后， 该节点被删除， 只是Zookeeper给该节点名称进行顺序编号。

### Stat 结构体

- czxid-创建节点的事务 zxid

每次修改ZooKeeper 状态都会收到一个zxid 形式的时间戳，也就是 ZooKeeper 事务 ID。事务 ID 是 ZooKeeper 中所有修改总的次序。每个修改都有唯一的 zxid，如果 zxid1 小于 zxid2，那么 zxid1 在 zxid2 之前发生。

- ctime - znode 被创建的毫秒数(从 1970 年开始) 

- mzxid - znode 最后更新的事务 zxid

- mtime - znode 最后修改的毫秒数(从 1970 年开始) 

- pZxid-znode 最后更新的子节点 zxid

- cversion - znode 子节点变化号，znode 子节点修改次数

- dataversion - znode 数据变化号

-  aclVersion - znode 访问控制列表的变化号

- ephemeralOwner- 如果是临时节点，这个是 znode 拥有者的 session id。如果不是临时节点则是 0。

-  dataLength- znode 的数据长度

-  numChildren - znode 子节点数量

在ZooKeeper Java shell中，可以使用`stat`或`ls2`命令查看znode的stat结构。 具体说明如下：

- 使用`stat`命令查看znode的`stat`结构：

```
[zk: localhost(CONNECTED) 0] stat /zookeeper
cZxid = 0x0
ctime = Thu Jan 01 05:30:00 IST 1970
mZxid = 0x0
mtime = Thu Jan 01 05:30:00 IST 1970
pZxid = 0x0
cversion = -1
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 1
```

- 使用`ls2`命令查看znode的stat结构：

```
[zk: localhost(CONNECTED) 1] ls2 /zookeeper
[quota]
cZxid = 0x0
ctime = Thu Jan 01 05:30:00 IST 1970
mZxid = 0x0
mtime = Thu Jan 01 05:30:00 IST 1970
pZxid = 0x0
cversion = -1
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 1
```

### 监听器原理



1、监听原理详解：

-  首先要有一个main()线程
- 在main线程中创建Zookeeper客户端，这时就会创建两个线程，一个负责网络连接通信（connet），一个负责监听（listener）。
-  通过connect线程将注册的监听事件发送给Zookeeper。
- 在Zookeeper的注册监听器列表中将注册的监听事件添加到列表中。
-  Zookeeper监听到有数据或路径变化，就会将这个消息发送给listener线程。
- listener线程内部调用了process()方法。

2、常见的监听

- 监听节点数据的变化

`get path [watch]`

- 监听子节点增减的变化

`ls path [watch]`

### 选举机制

ZAB leader选举

### 写数据流程

ZAB log复制

1. Client 向 ZooKeeper 的Server1 上写数据，发送一个写请求。
2. 如果Server1不是Leader，那么Server1 会把接受到的请求进一步转发给Leader，因为每个ZooKeeper的Server里面有一个是Leader。这个Leader 会将写请求广播给各个Server，比如Server1和Server2，各个Server会将该写请求加入待  写队列，并向Leader发送成功信息。
3. 当Leader收到半数以上 Server 的成功信息，说明该写操作可以执行。Leader会向各个Server 发送提交信息，各个Server收到信息后会落实队列里的写请求，此时写成功。
4. Server1会进一步通知 Client 数据写成功了，这时就认为整个写操作成功

## 服务上下线实战

某分布式系统中，主节点可以有多台，可以动态上下线，任意一台客户端都能实时感知到主节点服务器的上下线。



##  参考

[w3cschool Zookeeper教程](https://www.w3cschool.cn/zookeeper/)

[Zookeeper官方文档](https://zookeeper.apache.org/doc/r3.6.2/)

[Zookeeper思维导图](https://blog.csdn.net/WuLex/article/details/103345933)

[bibi zookeeper视频](https://www.bilibili.com/video/BV1PW411r7iP?from=search&seid=16421878241949941390)

[zookeeper学习笔记](https://blog.csdn.net/u011863024/article/details/107434932)

[【ZooKeeper系列】3.ZooKeeper源码环境搭建](https://segmentfault.com/a/1190000021451833)