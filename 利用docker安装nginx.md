---
title: 利用docker创建mysql
date: 2019-8-8 14:14:00
tags: docker,mysql
---
首先安装docker-compose，我觉得用起来比较方便。

```shell
curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
然后可以在一个你喜欢的目录下，创建docker-compose.yml文件。我是在/use/local/docker/nginx/下创建的。

1. docker-compose.yml

   ```yaml
   version: '3.1'
   services:
     nginx:
       restart: always
       image: nginx
       container_name: nginx
       ports:
         - 81:80
       volumes:
         - ./conf/nginx.conf:/etc/nginx/nginx.conf
         - ./wwwroot:/usr/share/nginx/wwwroot
   ```

   注：在创建./conf 目录下创建nginx.conf文件，在./wwwroot下创建 html80文件夹

2. nginx.conf配置内容

   **a. 基于端口的虚拟主机**

   ```json
   worker_processes  1;
   
   events {
       worker_connections  1024;
   }
   
   http {
       include       mime.types;
       default_type  application/octet-stream;
   
       sendfile        on;
       
       keepalive_timeout  65;
       # 配置虚拟主机 192.168.75.145
       server {
   	# 监听的ip和端口，配置 192.168.75.145:80
           listen       80;
   	# 虚拟主机名称这里配置ip地址
           server_name  192.168.75.145;
   	# 所有的请求都以 / 开始，所有的请求都可以匹配此 location
           location / {
   	    # 使用 root 指令指定虚拟主机目录即网页存放目录
   	    # 比如访问 http://ip/index.html 将找到 /usr/local/docker/nginx/wwwroot/html80/index.html
   	    # 比如访问 http://ip/item/index.html 将找到 /usr/local/docker/nginx/wwwroot/html80/item/index.html
   
               root   /usr/share/nginx/wwwroot/html80;
   	    # 指定欢迎页面，按从左到右顺序查找
               index  index.html index.htm;
           }
   
       }
       # 配置虚拟主机 192.168.75.245
       server {
           listen       8080;
           server_name  192.168.75.145;
   
           location / {
               root   /usr/share/nginx/wwwroot/html8080;
               index  index.html index.htm;
           }
       }
   }
   ```

  **b. 基于域名的虚拟主机**

   ```json
   user  nginx;
   worker_processes  1;
   
   events {
       worker_connections  1024;
   }
   
   http {
       include       mime.types;
       default_type  application/octet-stream;
   
       sendfile        on;
   
       keepalive_timeout  65;
       server {
           listen       80;
           server_name  admin.service.itoken.funtl.com;
           location / {
               root   /usr/share/nginx/wwwroot/htmlservice;
               index  index.html index.htm;
           }
   
       }
   
       server {
           listen       80;
           server_name  admin.web.itoken.funtl.com;
   
           location / {
               root   /usr/share/nginx/wwwroot/htmlweb;
               index  index.html index.htm;
           }
       }
   }
   ```

   **c. 反向代理**

   ```json
   user  nginx;
   worker_processes  1;
   
   events {
       worker_connections  1024;
   }
   
   http {
       include       mime.types;
       default_type  application/octet-stream;
   
       sendfile        on;
   
       keepalive_timeout  65;
   	
   	# 配置一个代理即 tomcat1 服务器
   	upstream tomcatServer1 {
   		server 192.168.75.145:9090;
   	}
   
   	# 配置一个代理即 tomcat2 服务器
   	upstream tomcatServer2 {
   		server 192.168.75.145:9091;
   	}
   
   	# 配置一个虚拟主机
   	server {
   		listen 80;
   		server_name admin.service.itoken.funtl.com;
   		location / {
   				# 域名 admin.service.itoken.funtl.com 的请求全部转发到 tomcat_server1 即 tomcat1 服务上
   				proxy_pass http://tomcatServer1;
   				# 欢迎页面，按照从左到右的顺序查找页面
   				index index.jsp index.html index.htm;
   		}
   	}
   
   	server {
   		listen 80;
   		server_name admin.web.itoken.funtl.com;
   
   		location / {
   			# 域名 admin.web.itoken.funtl.com 的请求全部转发到 tomcat_server2 即 tomcat2 服务上
   			proxy_pass http://tomcatServer2;
   			index index.jsp index.html index.htm;
   		}
   	}
   }
   ```

   **注意：新版 Nginx 的 upstream 配置中的名称不可以有下划线("_")，否则会报 400 错误**

   **d.负载均衡**

   ```json
   user  nginx;
   worker_processes  1;
   
   events {
       worker_connections  1024;
   }
   
   http {
       include       mime.types;
       default_type  application/octet-stream;
   
       sendfile        on;
   
       keepalive_timeout  65;
   	
   	upstream myapp1 {
   		server 192.168.75.145:9090 weight=10;
   		server 192.168.75.145:9091 weight=10;
   	}
   
   	server {
   		listen 80;
   		server_name nginx.funtl.com;
   		location / {
   			proxy_pass http://myapp1;
   			index index.jsp index.html index.htm;
   		}
   	}
   }
   ```

   相关配置说明：

   ```json
   # 定义负载均衡设备的 Ip及设备状态 
   upstream myServer {
       server 127.0.0.1:9090 down;
       server 127.0.0.1:8080 weight=2;
       server 127.0.0.1:6060;
       server 127.0.0.1:7070 backup;
   }
   ```

   - `upstream`：每个设备的状态:
   - `down`：表示当前的 `server` 暂时不参与负载
   - `weight`：默认为 1 `weight` 越大，负载的权重就越大。
   - `max_fails`：允许请求失败的次数默认为 1 当超过最大次数时，返回 `proxy_next_upstream` 模块定义的错误
   - `fail_timeout`:`max_fails` 次失败后，暂停的时间。
   - `backup`：其它所有的非 `backup` 机器 `down` 或者忙的时候，请求 `backup` 机器。所以这台机器压力会最轻