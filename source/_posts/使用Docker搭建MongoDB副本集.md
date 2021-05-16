---
title: 使用Docker搭建MongoDB副本集
date: 2021-05-02 20:00:00
tags: [DevOps]
categories: 编程
---

>  实现MongoDB高可用服务

# 简介

mongodb集群搭建有三种方式。

1. Master-Slave模式

2. Replica-Set方式

3. Sharding方式

其中，第一种方式基本没什么意义，官方也不推荐这种方式搭建。因为主从复制虽然可以承受一定的负载压力，但这种方式仍然是一个单点，如果主库挂了，数据写入就成了风险。另外两种分别就是副本集和分片的方式。分片用于存储海量的数据时使用。

## Replica Sets复制集

MongoDB在1.6版本对开发了新功能replica set，这比之前的主从复制功能要强大一 些，增加了故障自动切换和自动修复成员节点，各个DB之间数据完全一致，大大降低了维护成本。

MongoDB复制是将数据同步在多个服务器的过程。

复制提供了数据的冗余备份，并在多个服务器上存储数据副本，提高了数据的可用性， 并可以保证数据的安全性。

复制还允许您从硬件故障和服务中断中恢复数据。

# 实现

## 生成keyFile

- MongoDB使用keyfile认证，副本集中的每个mongod实例使用keyfile内容作为认证其他成员的共享密码。mongod实例只有拥有正确的keyfile才可以加入副本集。
- keyFile的内容必须是6到1024个字符的长度，且副本集所有成员的keyFile内容必须相同。
- 有一点要注意是的：在UNIX系统中，keyFile必须没有组权限或完全权限（也就是权限要设置成X00的形式）。Windows系统中，keyFile权限没有被检查。
- 可以使用任意方法生成keyFile。例如，如下操作使用openssl生成复杂的随机的1024个字符串。然后使用chmod修改文件权限，只给文件拥有者提供读权限。

~~~
# 400权限是要保证安全性，否则mongod启动会报错
openssl rand -base64 756 > mongodb.key
chmod 400 mongodb.key
~~~

将生成的keyFile复制到其他节点服务器上，保证每一个副本集成员使用相同的keyFile文件

## 通过docker-compose启动MongoDB

~~~java
version: '3.3'

services:
  mongo:
    image: mongo:latest
    volumes:
      - ./data/mongo:/data/db
      - ./data/mongo/log:/data/log
      - ./mongodb.key:/data/mongodb.key
    user: root
    environment:
      - MONGO_INITDB_ROOT_USERNAME=username
      - MONGO_INITDB_ROOT_PASSWORD=password
      - MONGO_INITDB_DATABASE=init
    container_name: mongodb
    ports:
      - 27017:27017
    command: mongod --replSet mongos --keyFile /data/mongodb.key
    restart: always
    entrypoint:
      - bash
      - -c
      - |
        chmod 400 /data/mongodb.key
        chown 999:999 /data/mongodb.key
        exec docker-entrypoint.sh $$@
~~~

文件详解

* `chown 999:999 /data/mongodb.key` 999用户是容器中的mongod用户，通过chown修改文件用户权限
* `mongod --replSet mongos --keyFile /data/mongodb.key` 启动命令
* `--replSet mongos` 以副本集形式启动并将副本集名字命名为 mongos 
* `--keyFile /data/mongodb.key` 设置keyFile，用于副本集通信，文件通过 volumes 映射到容器内

在各个节点服务器启动mongo

```
docker-compose -f  docker-compose-mongo.yml up -d
```

## 配置

1. 进入其中一个mongo容器内

~~~
docker exec -it mongodb /bin/bash
~~~

2. 登录

~~~
mongo -u 账号 -p 密码
~~~

3. 登录成功可以查看状态

```
> rs.status()
{
	"ok" : 0,
	"errmsg" : "no replset config has been received",
	"code" : 94,
	"codeName" : "NotYetInitialized"
}
```

4. 配置副本集

~~~
config = { _id:"mongos", members:[{_id:0,host:"ip:27017"},{_id:1,host:"ip:27017"},
{_id:2,host:"ip:27017"}]}
rs.initiate(config);
{ "ok" : 1 }
~~~

上面提示ok就是表示成功了，这时候会选举出Primary节点。重新通过`rs.status()`查看状态就能看到了。

通过`rs.status()`的输出我们就能分出那个是`PRIMARY`节点了。

# 测试

1. 在主节点插入数据

~~~shell
mongos:PRIMARY> use test
switched to db test
mongos:PRIMARY> db
test
mongos:PRIMARY> db.test.insert({"name":"测试"})
WriteResult({ "nInserted" : 1 })
~~~

2. 查看从节点数据

~~~
mongos:SECONDARY> use test
switched to db test
mongos:SECONDARY> db
test
mongos:SECONDARY> db.test.find()
Error: error: {
	"topologyVersion" : {
		"processId" : ObjectId("6093a0eb00c54d058f6c6419"),
		"counter" : NumberLong(4)
	},
	"operationTime" : Timestamp(1620290083, 1),
	"ok" : 0,
	"errmsg" : "not master and slaveOk=false",
	"code" : 13435,
	"codeName" : "NotPrimaryNoSecondaryOk",
	"$clusterTime" : {
		"clusterTime" : Timestamp(1620290083, 1),
		"signature" : {
			"hash" : BinData(0,"1yNWjsrfQeHQyfQFkXeo0qxhJco="),
			"keyId" : NumberLong("6959084154784841732")
		}
	}
}
mongos:SECONDARY> rs.slaveOk()
WARNING: slaveOk() is deprecated and may be removed in the next major release. Please use secondaryOk() instead.
mongos:SECONDARY> rs.secondaryOk()
mongos:SECONDARY> db.test.find()
{ "_id" : ObjectId("6093a741215b2bd65f67b68b"), "name" : "测试" }
~~~

测试通过，在从节点可以看到在主节点添加的数据。

注意：执行db.test.find()报"not master and slaveOk=false"，这是正常情况，因为初始化的时候slave默认不允许读写

# 添加成员

在主节点执行rs.add("ip:27017")

~~~
mongos:PRIMARY> rs.add("ip:27019")
{
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1620291932, 1),
		"signature" : {
			"hash" : BinData(0,"7rHK33BLGBnUI/Wcm1RUpkGglEY="),
			"keyId" : NumberLong("6959084154784841732")
		}
	},
	"operationTime" : Timestamp(1620291932, 1)
}
~~~

再次通过rs.status()查看

# 更换主节点

在主节点修改配置

~~~shell
//查看当前配置，存入config变量中
mongos:PRIMARY> config=rs.conf()
//修改config变量，将第二个成员的优先级设为2
mongos:PRIMARY> config.members[1].priority = 2
2
//配置生效
mongos:PRIMARY> rs.reconfig(config)
// 查看状态
mongos:PRIMARY> rs.status()
{
	"set" : "mongos",
	"date" : ISODate("2021-05-06T09:20:57.517Z"),
	"myState" : 2,
	"term" : NumberLong(2),
	"syncSourceHost" : "192.168.6.128:27019",
	"syncSourceId" : 2,
	"heartbeatIntervalMillis" : NumberLong(2000),
	"majorityVoteCount" : 2,
	"writeMajorityCount" : 2,
	"votingMembersCount" : 3,
	"writableVotingMembersCount" : 3,
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1620292840, 1),
			"t" : NumberLong(2)
		},
		"lastCommittedWallTime" : ISODate("2021-05-06T09:20:40.488Z"),
		"readConcernMajorityOpTime" : {
			"ts" : Timestamp(1620292840, 1),
			"t" : NumberLong(2)
		},
		"readConcernMajorityWallTime" : ISODate("2021-05-06T09:20:40.488Z"),
		"appliedOpTime" : {
			"ts" : Timestamp(1620292840, 1),
			"t" : NumberLong(2)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1620292840, 1),
			"t" : NumberLong(2)
		},
		"lastAppliedWallTime" : ISODate("2021-05-06T09:20:40.488Z"),
		"lastDurableWallTime" : ISODate("2021-05-06T09:20:40.488Z")
	},
	"lastStableRecoveryTimestamp" : Timestamp(1620292840, 1),
	"electionParticipantMetrics" : {
		"votedForCandidate" : true,
		"electionTerm" : NumberLong(2),
		"lastVoteDate" : ISODate("2021-05-06T09:20:28.156Z"),
		"electionCandidateMemberId" : 1,
		"voteReason" : "",
		"lastAppliedOpTimeAtElection" : {
			"ts" : Timestamp(1620292817, 1),
			"t" : NumberLong(1)
		},
		"maxAppliedOpTimeInSet" : {
			"ts" : Timestamp(1620292817, 1),
			"t" : NumberLong(1)
		},
		"priorityAtElection" : 1,
		"newTermStartDate" : ISODate("2021-05-06T09:20:28.213Z"),
		"newTermAppliedDate" : ISODate("2021-05-06T09:20:29.266Z")
	},
	"members" : [
		{
			"_id" : 0,
			"name" : "192.168.6.128:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 5418,
			"optime" : {
				"ts" : Timestamp(1620292840, 1),
				"t" : NumberLong(2)
			},
			"optimeDate" : ISODate("2021-05-06T09:20:40Z"),
			"syncSourceHost" : "192.168.6.128:27019",
			"syncSourceId" : 2,
			"infoMessage" : "",
			"configVersion" : 3,
			"configTerm" : 2,
			"self" : true,
			"lastHeartbeatMessage" : ""
		},
		{
			"_id" : 1,
			"name" : "192.168.6.128:27018",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY", //已被设置为主节点
			"uptime" : 4824,
			"optime" : {
				"ts" : Timestamp(1620292840, 1),
				"t" : NumberLong(2)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1620292840, 1),
				"t" : NumberLong(2)
			},
			"optimeDate" : ISODate("2021-05-06T09:20:40Z"),
			"optimeDurableDate" : ISODate("2021-05-06T09:20:40Z"),
			"lastHeartbeat" : ISODate("2021-05-06T09:20:55.545Z"),
			"lastHeartbeatRecv" : ISODate("2021-05-06T09:20:56.257Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncSourceHost" : "",
			"syncSourceId" : -1,
			"infoMessage" : "",
			"electionTime" : Timestamp(1620292828, 1),
			"electionDate" : ISODate("2021-05-06T09:20:28Z"),
			"configVersion" : 3,
			"configTerm" : 2
		},
		{
			"_id" : 2,
			"name" : "192.168.6.128:27019",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 924,
			"optime" : {
				"ts" : Timestamp(1620292840, 1),
				"t" : NumberLong(2)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1620292840, 1),
				"t" : NumberLong(2)
			},
			"optimeDate" : ISODate("2021-05-06T09:20:40Z"),
			"optimeDurableDate" : ISODate("2021-05-06T09:20:40Z"),
			"lastHeartbeat" : ISODate("2021-05-06T09:20:55.543Z"),
			"lastHeartbeatRecv" : ISODate("2021-05-06T09:20:56.547Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncSourceHost" : "192.168.6.128:27018",
			"syncSourceId" : 1,
			"infoMessage" : "",
			"configVersion" : 3,
			"configTerm" : 2
		}
	],
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1620292840, 1),
		"signature" : {
			"hash" : BinData(0,"OizMd+vrzvez8MC+7baAHt0Iy8M="),
			"keyId" : NumberLong("6959084154784841732")
		}
	},
	"operationTime" : Timestamp(1620292840, 1)
}
~~~