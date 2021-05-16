---
title: 使用ELK做日志收集
date: 2020-08-31 00:00:00
tags: [中间件]
categories: 编程
---

>  微服务系统的日志都保存在各自指定的目录中，如果这些微服务部署在不同的服务器上，那么日志文件也是分散在各自的服务器上。分散的日志不利于我们快速通过日志定位问题，我们可以借助ELK来收集各个微服务系统的日志并集中展示。 
> ELK即Elasticsearch、Logstash和Kibana首字母缩写。Elasticsearch用于存储日志信息，Logstash用于收集日志，Kibana用于图形化展示。 

# 搭建ELK环境

 在Windwos上搭建ELK环境较为麻烦，这里我选择在CentOS7 上通过Docker来搭建ELK环境。

## 安装docker

###  安装Docker所需要的包 

```
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```

###  设置稳定的仓库 

```
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

###  安装最新版的Docker引擎 

```
yum install docker-ce docker-ce-cli containerd.io
```

###  启动Docker 

```
systemctl start docker
```

###  查看是否安装成功 

```
docker -v
```

## 安装Docker Compose

 安装好Docker后，我们接着安装Docker Compose，官方安装教程https://docs.docker.com/compose/install/，主要步骤为： 

###  获取Docker Compose的最新稳定版本 

```
curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

###  对二进制文件授予可执行权限 

```
chmod +x /usr/local/bin/docker-compose
```

###  创建link 

```
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

### 查看是否安装成功

```
docker-compose -v
```

## Docker Compose搭建ELK

在搭建ELK之前，我们需要做一些准备工作。

正如官方所说的那样 https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html，Elasticsearch默认使用mmapfs目录来存储索引。操作系统默认的mmap计数太低可能导致内存不足，我们可以使用下面这条命令来增加内存：

```
sysctl -w vm.max_map_count=262144
```

###  创建Elasticsearch数据挂载路径 

```
mkdir -p /lxb/elasticsearch/data
```

###  对该路径授予777权限

```
chmod 777 /lxb/elasticsearch/data
```

###  创建Elasticsearch插件挂载路径 

```
mkdir -p /lxb/elasticsearch/plugins
```

###  创建Logstash配置文件存储路径 

```
mkdir -p /lxb/logstash
```

在该路径下创建`logstash-lxb.conf`配置文件（没有安装vim的话可以使用`yum install vim`命令安装)

内容如下所示:

```
input {
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4560
    codec => json_lines
  }
}
output {
  elasticsearch {
    hosts => "es:9200"
    index => "lxb-logstash-%{+YYYY.MM.dd}"
  }
}
```

###  创建ELK Docker Compose文件存储路径

```
mkdir -p /lxb/elk
```

 在该目录下创建`docker-compose.yml`文件： 

```
vim /lxb/elk/docker-compose.yml
```

 内容如下所示： 

```
version: '3'
services:
  elasticsearch:
    image: elasticsearch:6.4.1
    container_name: elasticsearch
    environment:
      - "cluster.name=elasticsearch" #集群名称为elasticsearch
      - "discovery.type=single-node" #单节点启动
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" #jvm内存分配为512MB
    volumes:
      - /lxb/elasticsearch/plugins:/usr/share/elasticsearch/plugins
      - /lxb/elasticsearch/data:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
  kibana:
    image: kibana:6.4.1
    container_name: kibana
    links:
      - elasticsearch:es #配置elasticsearch域名为es
    depends_on:
      - elasticsearch
    environment:
      - "elasticsearch.hosts=http://es:9200" #因为上面配置了域名，所以这里可以简写为http://es:9200
    ports:
      - 5601:5601
  logstash:
    image: logstash:6.4.1
    container_name: logstash
    volumes:
      - /lxb/logstash/logstash-febs.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch
    links:
      - elasticsearch:es
    ports:
      - 4560:4560
```

 切换到`/lxb/elk`目录下，使用如下命令启动： 

```
docker-compose up -d
```

 第一次启动的时候，Docker需要拉取ELK镜像，过程可能稍慢，耐心等待即可。 

## Logstash中安装json_lines插件

 使用如下命令进入到Logstash容器中 

```
docker exec -it logstash /bin/bash
```

 切换到/bin目录，安装json_lines插件，然后退出： 

```
docker exec -it logstash /bin/bash
cd /bin/
logstash-plugin install logstash-codec-json_lines
```

 使用浏览器访问[http://localhost:5601](http://localhost:5601/)便可以看到Kibana管理界面

# 修改微服务日志配置

## 引入依赖

```
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>6.1</version>
</dependency>
```

## 配置修改

 日志配置文件`logback-spring.xml`里添加如下配置

```
<!--输出到 logstash的 appender-->
<appender name="logstash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>192.168.33.10:4560</destination>
    <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>

......
<root level="info">
    ......
    <appender-ref ref="logstash" />
</root>
```

 `192.168.33.10:4560`对应我们刚刚搭建的Logstash地址。