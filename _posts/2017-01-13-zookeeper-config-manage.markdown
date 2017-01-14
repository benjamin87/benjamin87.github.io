---
layout: post
title:  "基于Zookeeper实现配置管理"
date:   2017-01-13 09:21:59 +0800
categories: tech
author: "Benjamin"
tags:
    - zookeeper
    - 分布式
---

[Zookeeper][zookeeper]是[Apache Hadoop][Hadoop]的一个子项目，现已正式成为了Apache的顶级项目。Zookeeper旨在解决分布式系统环境的同步协调问题：统一命名、状态同步、配置管理、分布式锁等。本文就 **配置管理** 这一典型的应用场景，实现一个基于Zookeeper的解决方案。

---

#### 需求描述

线上运行中的服务拥有众多配置项，例如数据库连接串、后端HTTP URL、内存中的运算常量等等。随着线上服务的不断扩张，需要管理的服务以及相应的配置项数量也与日俱增。当被要求修改一个线上配置，如果逐一服务执行修改后重启，对于运维人员而言，这将会是一场与耐心抗衡的比拼。稍有疏忽，就可能导致上线错误、部分服务遗漏，甚至线上服务宕机的危险。所以在运维过程中，我们迫切需要的是一种不重启服务、应对集群式的配置管理方案。搜刮一下程序员工具箱，Zookeeper项目跃然映入我们眼帘。

![scenario]({{ site.baseurl }}/img/20170113/01.jpg){: .center-image }

图中简化地描述了基于Zookeeper的服务配置管理方案：

- Zookeeper中的配置项均以数据节点存在，不同服务配置项创建在不同的节点下
- 多个同类型的服务可以同时监听一个节点下的配置项变更情况
- 当修改了对应配置项后，监听该配置改动的服务会及时收到推送消息，并完成本地服务的配置项修改

---

#### Zookeeper

##### 数据节点

Zookeeper管理的是一系列的数据节点，这些数据节点类似Unix中的文件系统，构成了一棵树状结构。每个数据节点都有生命周期。根据生命周期以及命名方式的不同，Zookeeper中的节点有如下四种类型：

- 持久节点（PERSISTENT）：直到主动删除后才会清除的节点
- 持久顺序节点（PERSISTENT_SEQUENTIAL）：创建时被Zookeeper自动顺序编号的持久节点
- 临时节点（EPHEMERAL）：客户端会话断开即被自动清除的节点
- 临时顺序节点（EPHEMERAL_SEQUENTIAL）:创建时被Zookeeper自动顺序编号的临时节点

当实现配置管理需求时，我们只需要关注持久节点，这也是Zookeeper中最常见的节点类型。在实践中，当我们确定好了一个服务所需要的配置项后，就可以直接在Zookeeper服务中创建对应的持久节点，并在持久节点中存入配置项的值即可。

这些操作可以通过官方提供zkCli.sh客户端的set接口完成。

```shell
$ set /my-zk "myData"
cZxid = 0x2
ctime = Fri Jan 10 23:01:08 CST 2017
mZxid = 0xb
mtime = Sat Jan 11 16:26:53 CST 2017
pZxid = 0x7
cversion = 1
dataVersion = 2
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 6
numChildren = 1
```

##### Watcher

一旦创建了对应配置的持久节点，服务启动连上Zookeeper集群后就可以正确地读取到初始值。不忘初心，我们最想要的是线上直接不重启、统一修改配置的功能。此时，就该Zookeeper的Watcher登场了。Watcher是Zookeeper实现分布式系统数据的发布/订阅功能的核心。

![watcher]({{ site.baseurl }}/img/20170113/02.jpg){: .center-image }

上图展示了一个Watcher工作的基本流程：

1. Zookeeper的客户端向服务端注册Watcher，来监听感兴趣的数据节点变化
2. 读取节点的初始值
3. 当监听的节点发生变化时，Zookeeper服务端通知监听客户端
4. 读取新的节点值

Zookeeper的监听模型是推拉结合式的。

- 推：服务器端发现节点变化后，只把节点变化的消息发送给监听的客服端，并没有把变化值一起发送
- 拉：当客户端收到变化推送的消息后，主动去拉取最新的节点值

这样设计看似增加了网络上的多次传输，但其实有效地降低Watcher本身数据结构的设计。所有的通知消息都可以统一表现为状态、类型、节点三个变量，使得Watcher本身的设计十分轻量。

---

#### 实践

在理解了Zookeeper的数据节点和Watcher机制后，我们就可以进行实践操作了。

首先搭建下实验拓扑，我们利用2台机器：

- 模拟Zookeeper集群，其上安装了Zookeeper-3.4.9，IP为192.168.1.249
- 模拟线上服务，集成了[Curator][Curator](Zookeeper客户端，同为Apache项目)的Java服务，IP为192.168.1.103

![practice]({{ site.baseurl }}/img/20170113/03.jpg){: .center-image }

通过zkCli.sh的create命令创建/my-zk和/my-zk/factor节点:

```shell
$ create /my-zk ""
$ create /my-zk/factor "1.0"
```

导入curator依赖 (Gradle)

```gradle
// Curator 2.XX 与 Zookeeper 3.4.X 对应
compile("org.apache.curator:curator-recipes:2.11.1")
compile("org.apache.curator:curator-framework:2.11.1")
```

创建Curator客服端，并打开连接：

```java
final String zookeeperConnectionString = "192.168.1.249:2181";
final String ZK_PATH = "/my-zk";

RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client = CuratorFrameworkFactory.newClient(zookeeperConnectionString, retryPolicy);
client.start();
```

注册Watcher，监听/my-zk节点变化：

```java
PathChildrenCache watcher = new PathChildrenCache(client, ZK_PATH, true);
watcher.getListenable().addListener((client1, event) -> {
  ChildData data = event.getData();
  if (data == null) {
    System.out.println("No data in event[" + event + "]");
  } else {
    System.out.println("Receive event: "
        + "type=[" + event.getType() + "]"
        + ", path=[" + data.getPath() + "]"
        + ", data=[" + new String(data.getData()) + "]"
        + ", stat=[" + data.getStat() + "]");
  }
});
watcher.start(StartMode.BUILD_INITIAL_CACHE);
```

启动后，读取到了初始值: 1.0

```shell
Receive event: type=[CHILD_ADDED], path=[/my-zk/factor], data=[1.0], stat=[7,8,1484319858396,1484319868212,1,0,0,0,6,0,7]
```

通过zkCli.sh的set命令修改factor值: 0.9

```shell
$ set /my-zk/test "0.9"
cZxid = 0x7
ctime = Fri Jan 10 23:04:18 CST 2017
mZxid = 0x11
mtime = Sat Jan 11 17:19:24 CST 2017
pZxid = 0x7
cversion = 0
dataVersion = 3
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 3
numChildren = 0
```

再查看运行中的client服务，已经收到了最新的更新值: 0.9

```shell
Receive event: type=[CHILD_UPDATED], path=[/my-zk/test], data=[0.9], stat=[7,17,1484319858396,1484385564558,3,0,0,0,3,0,7]
```

---

#### 总结

随着微服务概念的提出，线上服务被拆分得越来越细，这也直接导致了线上服务的量级呈现出爆炸式的增长。程序员再也不应该简单地开发完成任务后，将服务包扔给线上运维人员去听天由命。在开发的过程中选择合适的工具，提高线上运维与修改容错的成本，同样也应该纳入开发阶段的设计中。只有在设计阶段考虑得更加细致，才能给线上处理问题时，提供更多的工具。这也许正是DevOps理念提出后，程序员需要所修行的更多内容吧。

---

#### 参考

- 《从Paxos到Zookeeper 分布式一致性原理与实践》

---

[zookeeper]: https://zookeeper.apache.org/
[Hadoop]: http://hadoop.apache.org/
[Curator]: http://curator.apache.org/
