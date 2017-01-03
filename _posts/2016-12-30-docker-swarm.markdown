---
layout: post
title:  "初识Docker Swarm"
date:   2016-12-30 15:01:22 +0800
categories: tech
author: "Benjamin"
tags:
    - docker
    - 集群
---

[Docker][docker] v1.12 版本将原有的swarm功能并入了 **docker engine**，即 **[swarm mode][swarm]**，这大大简化了搭建与调度docker集群流程。本文遵循官方文档，搭建并实践swarm使用场景。

---

#### Swarm概念

Swarm是一组安装docker的机器集群，每一台安装docker的机器（既可以是物理机，也可以是虚拟机）被称为一个[node][swarm node]。Node一般分为两种：**manager node** 和 **worker nodes**。顾名思义，前者在swarm集群中扮演着leader的角色。通常用户提交的[services][services]执行请求会送达manager node，再通过manager node将请求封装为[tasks][tasks]分配给worker nodes。

---

#### 准备工作

本文的实践中，我们准备了3台物理机，并分别安装好了docker engine。

- manager(centOS) 192.168.1.248
- worker1(centOS) 192.168.1.249
- worker2(macOS)  192.168.1.105

此外，还需要在每台机器的防火墙中打开以下端口类型：

- 2377/tcp 用于集群管理通信
- 7946/tcp、7946/udp 用于node间通信
- 4789/tcp、4789/udp 用于overlay（swarm使用docker中的overlay类型的network）network

---

#### 组建Swarm

##### manager node

首先我们通过docker swarm init命令创建一个swarm。在manager机器上执行

```shell
    $ docker swarm init --listen-addr 0.0.0.0 --advertise-addr 192.168.1.248
```

成功后回显为

```shell
Swarm initialized: current node (55d9q4jqy0tn8tbuigmj53h6x) is now a manager.

    To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-4e86aqwe9q4q4vmlznxsxcu12vbix0flaym02hck3fyis9bqfp-avx3z06jdts45x5m9v7svlf3b \
    192.168.1.248:2377

    To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

##### worker nodes

根据提示可知：swarm init命令生成了token，当worker nodes加入该swarm时，需要提供该token。然后，我们就可以在另外2台worker nodes上执行加入swarm命令了。直接复制回显的docker swarm join命令即可。

```shell
$ docker swarm join \
    --token SWMTKN-1-4e86aqwe9q4q4vmlznxsxcu12vbix0flaym02hck3fyis9bqfp-avx3z06jdts45x5m9v7svlf3b \
    192.168.1.248:2377
```

添加完成后，回到manager node上，执行docker node ls查看swarm节点列表。

```shell
ID                           HOSTNAME      STATUS  AVAILABILITY  MANAGER STATUS
55d9q4jqy0tn8tbuigmj53h6x *  manager       Ready   Active        Leader
9iywbpn0ey5h5262c8jelr55n    worker1       Ready   Active
6daeompzdo6bl39yrdeptzizb    worker2       Ready   Active
```

通过输出列表中的status和availability列信息可知，swarm中的3台机器均已经正常工作了！

---

#### 部署services

搭建完swarm集群，我们就可以开始部署自己的服务了。在manager node上执行命令

```shell
$ docker service create --replicas 1 --name helloworld alpine ping docker.com
bs4r4zll7m774cnqhyyeeplln

$ docker service ls
ID            NAME        REPLICAS  IMAGE   COMMAND
bs4r4zll7m77  helloworld  1/1       alpine  ping docker.com

$ docker service ps helloworld
ID                         NAME          IMAGE   NODE     DESIRED STATE  CURRENT STATE          ERROR
0xb2gnjb571w0lhjmjspx71or  helloworld.1  alpine  manager  Running        Running 2 minutes ago

$ docker ps
CONTAINER ID  IMAGE          COMMAND            CREATED         STATUS        PORTS  NAMES
e319ea3fa588  alpine:latest  "ping docker.com"  1 minutes ago   Up 1 minutes        helloworld.1.0xb2gnjb571w0lhjmjspx71or
```

通过docker service create命令，我们在该swarm集群中创建了一个名为helloworld的service。通过docker service ls命令，我们确认了该service已经在正常运行了。其中replicas表示该服务有几个实例正在运行。swarm中service分两种：

- replicated services 该类型service在创建时，就定义了实例个数，并可以动态调节，由manager统一分配
- global services 全局服务无须指定实例个数，默认就在swarm中的所有节点上创建一个实例

执行docker service ps helloworld后，我们发现，该helloworld实例被分配在了manager节点上。因为manager节点本身也是一个worker node，所以在当前manager机器上直接执行docker ps就能看到运行中的docker container了（实践中也会发生被分配在另外2个节点的可能）

#### Scale service

执行命令docker service scale可以很简便地增加、减少service实例。

```shell
$ docker service scale helloworld=3
helloworld scaled to 3

$ docker service ps helloworld
ID                         NAME          IMAGE   NODE     DESIRED STATE  CURRENT STATE           ERROR
0xb2gnjb571w0lhjmjspx71or  helloworld.1  alpine  manager  Running        Running 15 minutes ago
4vu6lwtqjzo0zzv4l19mvj02d  helloworld.2  alpine  worker1  Running        Running 5 seconds ago
1g5sf5m86qvabpojhmvujpqmr  helloworld.3  alpine  worker2  Running        Running 5 seconds ago
```

---

#### 总结

通过docker相关内建命令(swarm/node/service)，我们可以快速地搭建swarm集群并部署我们的service。同时，在scaling的操作上也非常便捷。接下来，我们就可以利用搭建的swarm集群部署我们的代码，搭建适合的测试环境啦！

[docker]:https://www.docker.com/
[swarm]:https://docs.docker.com/engine/swarm/
[swarm node]:https://docs.docker.com/engine/swarm/key-concepts/#/node
[services]:https://docs.docker.com/engine/swarm/key-concepts/#Services-and-tasks
[tasks]::https://docs.docker.com/engine/swarm/key-concepts/#Services-and-tasks
