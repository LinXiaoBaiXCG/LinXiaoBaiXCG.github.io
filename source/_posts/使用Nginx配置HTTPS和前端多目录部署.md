---
title: 使用Nginx配置HTTPS和前端多目录部署
date: 2020-09-28 20:00:00
tags: [中间件]
categories: 编程
---

>  Nginx的实践 

# 开启Nginx的SSL模块

Nginx如果未开启SSL模块，配置Https时提示如下错误：

```
nginx: [emerg] the "ssl" parameter requires ngx_http_ssl_module in /usr/local/nginx/conf/nginx.conf
```

# Nginx配置HTTPS并做自动转发

1. 将证书放到服务器目录中
2. 在server模块中配置HTTPS

```
server {
        listen       443 ssl;
        server_name  xxxx.com;
		root html;
		index index.html index.htm;
		ssl_certificate cert/xxxx.com.pem; #证书中pem文件地址
		ssl_certificate_key cert/xxxx.com.key;  #证书中key文件地址
		ssl_session_timeout 5m;
		ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
		ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
		ssl_prefer_server_ciphers on; 

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   /usr/local/test_project/admin-web/dist;
            index  index.html index.htm;
	    try_files $uri $uri/ /index.html;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
```

3. HTTP自动转发HTTPS

```
server {
        listen       80;
        server_name  xxxx.com; #多域名用空格分开
		rewrite ^(.*)$ https://$host$1 permanent;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;
		
		location / {
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
```

# 多目录部署前端项目

```
server {
        listen 80;
        server_name  xxxx.com;
        
        location /aaa { 
                alias /data/web-a/dist/; 
                index index.html;
        }
        
        location /bbb { 
                alias /data/web-b/dist/;
                index index.html;
        }
}
```

# Nginx中root和alias的区别

root配置：nginx直接转发至 root后面地址+location 后面地址（返回的时候回带上location后面的地址）

alias配置： nginx直接转发至 alias的地址，不会加上location后的内容

注意：alias配置最后一定要 “/” 结尾 ，root随意

