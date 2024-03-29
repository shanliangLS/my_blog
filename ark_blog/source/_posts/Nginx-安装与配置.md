---
title: Nginx 安装与配置
date: 2022-09-01 15:05:00
categories:
- 快速上手
tags:
- Nginx
---

Nginx 软件安装、配置

<!-- more -->

环境：CentOS 7.6+

## 安装

从 http://nginx.org/en/download.html 下载 nginx-1.23.1.tar.gz

```bash
tar -zxvf nginx-1.23.1.tar.gz
cd nginx-1.23.1
./configure --prefix=/usr/local/nginx
make && make install
```

`/usr/local/nginx`：nginx 安装目录，可自定义位置

启动

```bash
/usr/local/nginx/sbin/nginx
```

重启

```bash
/usr/local/nginx/sbin/nginx -s reload
```

停止

```bash
/usr/local/nginx/sbin/nginx -s stop
```

### 安装新模块

如 sll 模块 ngx_http_ssl_module 和 日志模块 ngx_http_log_module

```bash
tar -zxvf nginx-1.23.1.tar.gz
cd nginx-1.23.1
./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_log_module
make
# 覆盖安装
make install
# 或者备份安装
#mv /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx_back
#cp ./sbin/nginx /usr/local/nginx/sbin/ /usr/local/nginx/sbin/
# 重启
/usr/local/nginx/sbin/nginx -s reload
```

查看当前模块

```bash
/usr/local/nginx/sbin/nginx -V
```

## 配置

编辑 `/usr/local/nginx/conf/nginx.conf`

### 部署系统

nginx.conf 配置文件：

```bash
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

http {
	include       mime.types;
    default_type  application/octet-stream;
    
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;
    
	# websocket配置
	map $http_upgrade $connection_upgrade {
		default keep-alive;
		'websocket' upgrade;
    }
    
    server{
        port  80;
        server_name  localhost;

		# 请求实体大小限制
        client_max_body_size 10m;

        location / {
            root   html;
            index  index.html index.htm;
        }
        
        # 转到后台
        location /api {
            proxy_pass http://127.0.0.1:81/api;
        }
    }
}
```

### 正向代理

nginx.conf 配置文件：

```bash
http {
    server{
        port  80;
        server_name  localhost;

        location / {
            proxy_pass  http://192.168.2.321:81;
        }
    }
}
```

### https 转发

获取 ssl 证书，一个 `.key` 文件和一个 `.pem` 文件，拷贝到 `/opt/cert/`中

nginx 需要安装 ssl 模块 ngx_http_ssl_module

nginx.conf 配置文件：

```bash
http {
    server{
        port  443 ssl;
        server_name  abc123.com;

		ssl_certificate		/opt/cert/xxx_abc123.com.pem;
        ssl_certificate_key	/opt/cert/xxx_abc123.com.key;

        location / {
            proxy_pass  https://192.168.2.321:81;
        }
    }
}
```

#### 同端口 http 自动跳转 https

参考 [nginx_非标准端口_同端口_http_自动跳转_https](https://www.cnblogs.com/haolb123/p/16553020.html)

用 error_page 拦截 497 实现 http 跳转 https，在 `server { .. }` 中加入一行

```bash
error_page 497 https://$host:$server_port$request_uri; #默认用302,临时重定向
#或
error_page 497 =301 https://$host:$server_port$request_uri; #永久重定向
#或
error_page 497 =307 https://$host:$server_port$request_uri; #临时重定向,不改变请求的方法(如post还是post)
```

### websocket 转发

nginx.conf 配置文件：

```bash
http {
    server{
        ......

        # 自定义变量 $connection_upgrade
        map $http_upgrade $connection_upgrade {
            default          keep-alive;  #默认为keep-alive 可以支持 一般http请求
            'websocket'      upgrade;     #如果为websocket 则为 upgrade 可升级的。
        }

        location /websocket/ {
            proxy_pass http://127.0.0.1:8080/websocket/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade; #此处配置 上面定义的变量
            proxy_set_header Connection $connection_upgrade;
        }
    }
}
```

### stream 转发

nginx.conf 配置文件：

```bash
stream {
  server {
    listen 80;
    proxy_pass 192.168.2.321:81;
  }
}

http {
    server{
        ......
    }
}
```

注意 stream 与 http 平级

### gzip 压缩

nginx.conf 的 http 中加入以下片断

```bash
# gzip config
gzip on;
gzip_min_length 10k;
gzip_buffers 4 16k;
gzip_http_version 1.1;
gzip_comp_level 6;
gzip_types text/plain text/css text/javascript text/xml application/json application/javascript application/x-javascript application/xml application/x-httpd-php image/jpeg image/gif image/png;
gzip_vary on;
gzip_disable "MSIE [1-6]\.";
```

前端需要使用 compression-webpack-plugin 插件压缩文件

### 日志

nginx 需要安装日志模块 ngx_http_log_module

nginx.conf 配置文件：

```bash
http {
    server{
        port  80;
        server_name  localhost;

        location / {
            proxy_pass  http://192.168.2.321:81;
            access_log  /usr/local/nginx/logs/a.log;
        }
    }
}
```

重启后通过 `location /` 的日志将会输出到 `/usr/local/nginx/logs/a.log`

### location 与 proxy_pass 的 url 后加与不加 / 的区别

参考 [NGINX配置 —— location 与 proxy_pass的url后 加与不加 / 的区别](https://www.cnblogs.com/baixing/p/15847820.html)

请求网站为：192.168.1.123:80/static/a.html
nginx.conf 配置文件：

```bash
server{
    port  80;
    server_name  localhost;

    location /static {
        proxy_pass  192.168.2.321:81;
    }

    location /static {
        proxy_pass  192.168.2.321:81/;
    }

    location /static/ {
        proxy_pass  192.168.2.321:81;
    }

    location /static/ {
        proxy_pass  192.168.2.321:81/;
    }
}
```

- 第一种 location 后无 `/` proxy_pass 后无 `/`

```bash
# 192.168.1.123->server name
# :80 ---------> port
# /statc ------->location
# /a.html ------>proxy_pass 

location /static {
	proxy_pass  192.168.2.321:81;
}

最后网址经过nginx转向到的网址是 192.168.2.321:81/static/a.html
```

- 第二种 location 后无 `/` proxy_pass 后有 `/`

```bash
# 192.168.1.123---->server name
# :80 ------------> port
# /statc ---------->location
# /a.html --------->proxy_pass 

location /static {
	proxy_pass  192.168.2.321:81/;
}

最后网址经过nginx转向到的网址是 192.168.2.321:81/a.html
```

- 第三种 location 后有 `/` proxy_pass 后无 `/`

```bash
# 192.168.1.123-->server name
# :80 ------------> port
# /statc/ ---------->location
# a.html --------->proxy_pass 

location /static/ {
	proxy_pass  192.168.2.321:81;
}

最后网址经过nginx转向到的网址是 192.168.2.321:81/static/a.html
```

- 第四种 location 后有 `/` proxy_pass 后有 `/`

```bash
# 192.168.1.123-->server name
# :80 ------------> port
# /statc/ ---------->location（path1）
# a.html --------->proxy_pass （path2）

location /static/ {
	proxy_pass  192.168.2.321:81/;
}

最后网址经过nginx转向到的网址是 192.168.2.321:81/a.html
```

#### 总结
当 proxy_pass 内容是 ip:port/ 时nginx最后代理到的网址是 proxy_pass(内容)
当 proxy_pass 内容是 ip:port 时nginx最后代理到的网址是 location(内容) + proxy_pass(内容)

location内容是什么不太影响结果
