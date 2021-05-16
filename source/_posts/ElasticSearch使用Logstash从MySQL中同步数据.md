---
title: ElasticSearch使用Logstash从MySQL中同步数据
date: 2020-06-16 00:00:00
tags: [中间件]
categories: 编程
---

>  项目中的搜索功能需要使用ElasticSearch，现需使用Logstash来导入数据到ElasticSearch中。

# 安装ElasticSearch

1. 下载安装包并解压，亦可用Docker安装
2. 进入装目录下的config文件夹中，修改elasticsearch.yml 文件，Docker安装进入容器内进行配置即可
3. 修改的主要内容

```
#配置es的集群名称，默认是elasticsearch，es会自动发现在同一网段下的es，如果在同一网段下有多个集群，就可以用这个属性来区分不同的集群。
cluster.name: my-es
#节点名称
node.name: node-1
#设置索引数据的存储路径
path.data: /usr/local/elasticsearch/data
#设置日志的存储路径
path.logs: /usr/local/elasticsearch/logs
#设置当前的ip地址,通过指定相同网段的其他节点会加入该集群中
network.host: 0.0.0.0
#设置对外服务的http端口
http.port: 9200
#设置集群中master节点的初始列表，可以通过这些节点来自动发现新加入集群的节点
discovery.zen.ping.unicast.hosts: ["127.0.0.1","10.10.10.34:9200"]
```

# 安装Logstash

同样也是下载压缩到后解压即可，然后到解压目录执行

```
./bin/logstash -e
```

查看日志能正常启动说明安装成功

# 编写同步脚本

同步数据需要使用`logstash-input-jdbc`插件，在`logstash-6.1.1 `以后已经默认支持 `logstash-input-jdbc`插件，所以不需要再单独安装了。

在安装目录下新建connector、script文件夹用于分别存放MySQL 的驱动文件和同步脚本

编写脚本

```
input {
    jdbc {
      # mysql 数据库链接
      jdbc_connection_string => "jdbc:mysql://host：port/database"
      # 用户名和密码
      jdbc_user => "xxxx"
      jdbc_password => "xxxx"
      # MySQL的驱动文件地址
      jdbc_driver_library => "/usr/local/logstash-6.5.4/connector/mysql-connector-java-5.1.45.jar"
      # 驱动类名
      jdbc_driver_class => "com.mysql.jdbc.Driver"
      #直接执行sql语句, sql_last_value 是内置的变量，表示上一次 sql 执行中 update_time 的值,and update_time<now()解决临界点数据问题
      statement =>"select * from blzq_article_v3 where is_show = 1 and update_time >= :sql_last_value and update_time<now()"
      # 执行的sql 文件路径+名称
      #statement_filepath => "/usr/local/logstash-6.5.4/xxxx"
      #设置监听间隔  各字段含义（由左至右）分、时、天、月、年，全部为*默认含义为每分钟都更新
      schedule => "* * * * *"
      # 索引类型
      #type => "article"
    }
}
# 输出到elastsicearch
output {
    elasticsearch {
    	#elasticsearch集群地址
        hosts => ["127.0.0.1:9200","127.0.0.1:8200","127.0.0.1:8000"]
        # 索引值，查询的时候会用到；需要先在elasticsearch中创建对应的mapping，也可以采用默认的mapping
        index => "article"
        document_type => "article"
        #指定插入elasticsearch文档ID，对应input中sql字段id
        document_id => "%{id}"
    }
    stdout {
        codec => json_lines
    }
}

```

使用命令 `./bin/logstash -f ./script/mysql.conf` 执行导入脚本。

# 总结

1. 使用Logstash同步数据环境的安装相对简单，主要是配置导入脚本
2. 上面的导入配置脚本使用的是update_time方式的增量同步，该方式在数据库中物理删除是无法实时更新，可在项目中执行删除MySQL数据的时候同步删除ES中的数据
3. 当对实时性和数据一致性有高要求时，可使用MQ进行同步
4. 亦可使用ali的canal进行同步数据。项目地址：https://github.com/alibaba/canal