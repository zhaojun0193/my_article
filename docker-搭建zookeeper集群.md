---
title: docker 搭建zookeeper集群
date: 2019-4-27 10:30:26
tags: Dubbo
---

#### zookeeper镜像下载

```shell
docker pull zookeeper
```

#### 安装docker-compose

````shell
curl -L https://github.com/docker/compose/releases/download/1.24.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
````

#### 创建docker-compose.yml 文件

```shell
vi docker-compose.yml
```

```yaml
version: '3.1'

services:
  zoo1:
    image: zookeeper:latest
    restart: always
    hostname: zoo1
    container_name: zookeeper_1
    ports:
      - 2181:2181
    volumes:
      - /usr/local/docker_app/zookeeper/zoo1/data:/data
      - /usr/local/docker_app/zookeeper/zoo1/datalog:/datalog
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

  zoo2:
    image: zookeeper:latest
    restart: always
    hostname: zoo2
    container_name: zookeeper_2
    ports:
      - 2182:2181
    volumes:
      - /usr/local/docker_app/zookeeper/zoo2/data:/data
      - /usr/local/docker_app/zookeeper/zoo2/datalog:/datalog
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

  zoo3:
    image: zookeeper:latest
    restart: always
    hostname: zoo3
    container_name: zookeeper_3
    ports:
      - 2183:2181
    volumes:
      - /usr/local/docker_app/zookeeper/zoo3/data:/data
      - /usr/local/docker_app/zookeeper/zoo3/datalog:/datalog
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
```

#### 启动容器

在当前目录下输入以下命令

```shell
docker-compose -f docker-compose.yml up -d
```

#### 查看容器运行状态

````shell
docker-compose ps
````

```reStructuredText
 Name                  Command               State                     Ports                   
------------------------------------------------------------------------------------------------
zookeeper_1   /docker-entrypoint.sh zkSe ...   Up      0.0.0.0:2181->2181/tcp, 2888/tcp, 3888/tcp
zookeeper_2   /docker-entrypoint.sh zkSe ...   Up      0.0.0.0:2182->2181/tcp, 2888/tcp, 3888/tcp
zookeeper_3   /docker-entrypoint.sh zkSe ...   Up      0.0.0.0:2183->2181/tcp, 2888/tcp, 3888/tcp
```

####  进入容器查看

````shell
docker exec -it zookeeper_1 /bin/bash
bin/zkServer.sh status
````

```reStructuredText
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
Mode: follower
```

利用docker创建zookeeper集群就介绍完了。



转载请注明出处。