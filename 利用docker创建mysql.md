---
title: 利用docker创建mysql
date: 2019-8-5 14:16:00
tags: docker,mysql
---
###一、 命令行方式

1. 拉取官方镜像（我们这里选择5.7.22）

   ```shell
   docker pull mysql:5.7.22
   ```

2. 运行容器

   ```shell
   docker run -p 3306:3306 --name mysql \
   -v /usr/local/docker/mysql/conf:/etc/mysql \
   -v /usr/local/docker/mysql/logs:/var/log/mysql \
   -v /usr/local/docker/mysql/data:/var/lib/mysql \
   -e MYSQL_ROOT_PASSWORD=123456 \
   -d mysql:5.7.22
   ```


### 二、docker-compose方式

1. 安装docker-compose

   ```shell
   curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
   chmod +x /usr/local/bin/docker-compose
   ```

2. 创建docker-compose.yml文件

   ```shell
   mkdir /usr/local/docker/mysql
   vi docker-compose.yml
   ```

3. 文件内容如下

   ```yaml
   version: '3.1'
   services:
     mysql:
       restart: always
       image: mysql:5.7.22
       container_name: mysql
       ports:
         - 3306:3306
       environment:
         TZ: Asia/Shanghai
         MYSQL_ROOT_PASSWORD: 123456
       command:
         --character-set-server=utf8mb4
         --collation-server=utf8mb4_general_ci
         --explicit_defaults_for_timestamp=true
         --lower_case_table_names=1
         --max_allowed_packet=128M
         --sql-mode="STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO"
       volumes:
         - mysql-data:/var/lib/mysql
   
   volumes:
     mysql-data:
   ```

4. 启动

   ```shell
   docker-compose up -d
   ```

   

