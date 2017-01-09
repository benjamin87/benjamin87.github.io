---
layout: post
title:  "搭建私有Docker Registry(V2)"
date:   2017-01-04 12:15:30 +0800
categories: tech
author: "Benjamin"
tags:
    - docker
---

[Docker Registry][docker registry]是管理docker repository的服务，docker官方registry称为[Docker Hub][docker hub]。当然，我们也可以搭建自己的私有registry，这样就可以在内网中合理地管理repository。本文将基于Docker Registry(V2)构建一个简单的私有registry。

>docker registry中管理的是docker repository，每个repository中管理的是docker images，只不过这些image被标记为了不同的tag。

---

#### 生成证书

Docker Registry在启动时需要开启TLS，所以我们第一步需要生成证书。由于只是内网使用，我们直接通过openssl工具生成一张自签发的证书。

```shell
$ cd docker
$ mkdir -p certs && openssl req \
-newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
-x509 -days 365 -out certs/domain.crt
```

在交互填写证书信息过程中，需要注意的是 **Common Name (e.g. server FQDN or YOUR name) []:** 项。此处需要填写一个该Registry服务器内网可达的域名。

>如果模拟CA签发过程的话，需要先生成证书请求文件(CSR,Cerificate Signing Request)和私钥key文件，在用CA的私钥对csr签名后得到公钥。本文出于实验需要，此处直接生成了公钥文件。

大致填写信息如下：

```shell
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Shanghai
Locality Name (eg, city) []:Shanghai
Organization Name (eg, company) [Internet Widgits Pty Ltd]:tcg
Organizational Unit Name (eg, section) []:tcg
Common Name (e.g. server FQDN or YOUR name) []:test.thosecoolguys.com
Email Address []:
```

>如果内网机器少，可以直接修改访问客户端的hosts文件。亦或者，直接在路由器设置或者域名商处绑定创建CNAME。要确保该域名能访问运行Registry的机器即可。

---

#### 信任证书

>如果证书的来源是正规CA机构，可以直接跳过该章节。

由于我们使用了自签名证书，需要将crt文件加入系统的CA bundle文件当中，使操作系统信任我们的自签名证书。笔者使用的系统有两种：CentOS和macOS。
- CentOS:
```shell
cat domain.crt >> /etc/pki/tls/certs/ca-bundle.crt
```
- macOS
直接双击crt文件，导入mac的钥匙串访问即可

---

#### 启动registry

万事俱备，只欠东风。一旦拥有了以上证书，就可以启动我们的私有Registry。执行如下docker命令，注意替换/path/to/certs部分，将生成命令中创建的certs目录映射进docker。

```shell
docker run -d -p 5000:5000 --restart=always --name registry \
-v /path/to/certs:/certs \
-e REGISTRY_HTTP_SECRET=mytokensecret \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
registry:2
```

为了将我们本地的镜像push入该私有registry，先重新tag下本次的原有镜像。

```shell
$ docker tag registry test.thosecoolguys.com:5000/registry
$ docker images
REPOSITORY                           TAG     IMAGE ID     CREATED     SIZE
test.thosecoolguys.com:5000/registry latest  182810e6ba8c 12 days ago 37.6 MB
```

然后我们就可以见证奇迹的一刻了，直接使用push命令推入我们的registry吧！

```shell
$ docker push test.thosecoolguys.com:5000/registry
The push refers to a repository [test.thosecoolguys.com:5000/registry]
96b4f94bc221: Pushed
a05358fe6900: Pushed
e4bdfaa6de6b: Pushed
c161fb93c3f2: Pushed
7cbcbac42c44: Mounted from alpine
latest: digest: sha256:946480a23b33480b8e7cdb89b82c1bd6accae91a8e66d017e21e8b56551f6209 size: 1364
```

通过V2提供的Restful API，我们可以查看刚才push入的registry镜像信息。

```shell
$ curl https://test.thosecoolguys.com:5000/v2/_catalog
{"repositories":["alpine","registry"]}
```

再尝试pull一下吧！

```shell
$ docker pull test.thosecoolguys.com:5000/registry
Using default tag: latest
latest: Pulling from registry
b7f33cc0b48e: Already exists
46730e1e05c9: Pull complete
b7574c100797: Pull complete
2e5b74c7b611: Pull complete
ba42bd458a59: Pull complete
Digest: sha256:946480a23b33480b8e7cdb89b82c1bd6accae91a8e66d017e21e8b56551f6209
Status: Downloaded newer image for test.thosecoolguys.com:5000/registry:latest
```

---

#### 总结

Docker官方提供的registry:2的镜像，可以很快捷地实现一个本地私有的registry仓库。本文利用自签名的证书，搭建了一个私有registry，简单实现了push/pull的功能。围绕这些功能，我们可以实现更多更复杂的测试开发、运维等需求。再也不用为网速而烦恼了；同样，也不用担心数据放置在公共网络中的安全性问题。

---

#### 参考文献

  - [实战 | 如何搭建安全可靠的Docker Registry?][dockerone_reference]
  - [Deploying a registry server][deploying_registry]

[docker registry]:https://docs.docker.com/registry/
[docker hub]: https://hub.docker.com/
[dockerone_reference]: http://dockone.io/article/627
[deploying_registry]: https://docs.docker.com/registry/deploying/
