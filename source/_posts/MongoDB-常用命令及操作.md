---
title: MongoDB-常用命令及操作
date: 2019-12-08 00:00:00
tags: [数据库]
categories: 编程
---

# MongoDB

MongoDB 是一个跨平台的，面向文档的数据库，是当前 NoSQL 数据库产品中最热门的一种。它介于关系数据库和非关系数据库之间，是非关系数据库当中功能最丰富，最像关系数据库的产品。它支持的数据结构非常松散，是类似 JSON 的 BSON 格式，因此可以存储比较复杂的数据类型。
​ MongoDB 的官方网站地址是：http://www.mongodb.org/

# MongoDB的体系结构
在MongoDB中基本的概念是数据库、集合、文档

MongoDB和MySQL概念对比

| MySQL       | MongoDB     | 说明                                |
| ----------- | ----------- | ----------------------------------- |
| database    | database    | 数据库                              |
| table       | collection  | 数据库表/集合                       |
| row         | document    | 数据记录行/文档                     |
| column      | field       | 数据字段/域                         |
| index       | index       | 索引                                |
| table joins |             | 表连接,MongoDB不支持                |
| primary key | primary key | 主键,MongoDB自动将_id字段设置为主键 |

# MongoDB的数据类型

1. null：用于表示空值或者不存在的字段，{“x”:null}
2. 布尔型：布尔类型有两个值true和false，{“x”:true}
3. 数值：shell默认使用64为浮点型数值。{“x”：3.14}或{“x”：3}。对于整型值，可以使用NumberInt（4字节符号整数）或NumberLong（8字节符号整数），{“x”:NumberInt(“3”)}{“x”:NumberLong(“3”)}
4. 字符串：UTF-8字符串都可以表示为字符串类型的数据，{“x”：“呵呵”}
5. 日期：日期被存储为自新纪元依赖经过的毫秒数，不存储时区，{“x”:new Date()}
6. 正则表达式：查询时，使用正则表达式作为限定条件，语法与JavaScript的正则表达式相同，{“x”:/[abc]/}
7. 数组：数据列表或数据集可以表示为数组，{“x”： [“a“，“b”,”c”]}
8. 内嵌文档：文档可以嵌套其他文档，被嵌套的文档作为值来处理，{“x”:{“y”:3 }}
9. 对象Id：对象id是一个12字节的字符串，是文档的唯一标识，{“x”: objectId() }
10. 二进制数据：二进制数据是一个任意字节的字符串。它不能直接在shell中使用。如果要将非utf-字符保存到数据库中，二进制数据是唯一的方式。
11. 代码：查询和文档中可以包括任何JavaScript代码，{“x”:function(){/*…*/}} 

# MongoDB常用命令

1. show dbs -- 显示所有数据库

2. db -- 显示当前数据库对象或集合

3. use DATABASE_NAME -- 如果数据库不存在，则创建数据库，否则切换到指定数据库。

4. db.dropDatabase() --  删除数据库

5. db.COLLECTION_NAME.drop() -- 删除集合
6. db.createCollection(name, options) -- 创建集合（name: 要创建的集合名称；options: 可选参数, 指定有关内存大小及索引的选项）

注意：创建集合后添加文档才算正式创建成功
7. show collections 或 show tables -- 查看已有集合
8. db.COLLECTION_NAME.insert(document) 或 db.COLLECTION_NAME.save(document)  -- 插入文档
注意：插入文档你也可以使用 db.col.save(document) 命令。如果不指定 _id 字段 save() 方法类似于 insert() 方法。如果指定 _id 字段，则会更新该 _id 的数据。

9. db.collection.update(
   query,
   update,
   {
     upsert: boolean,
     multi: boolean,
     writeConcern: document
   }） -- 用于更新已存在的文档
   

参数说明：
* query : update的查询条件，类似sql update查询内where后面的。
* update : update的对象和一些更新的操作符（如$,$inc...）等，也可以理解为sql update查询内set后面的
* upsert : 可选，这个参数的意思是，如果不存在update的记录，是否插入objNew,true为插入，默认是false，不插入。
* multi : 可选，mongodb 默认是false,只更新找到的第一条记录，如果这个参数为true,就把按条件查出来多条记录全部更新。
* writeConcern :可选，抛出异常的级别。

10. db.collection.remove(
       query,
       justOne
    ) --删除文档

参数说明：
* query :（可选）删除的文档的条件。
* justOne : （可选）如果设为 true 或 1，则只删除一个文档，如果不设置该参数，或使用默认值
*  false，则删除所有匹配条件的文档。
* writeConcern :（可选）抛出异常的级别。

11. db.collection.find(query, projection) -- 查询文档
12. 条件操作符：
* (>) 大于 - $gt
* (<) 小于 - $lt
* (>=) 大于等于 - $gte
* (<= ) 小于等于 - $lte

13. db.COLLECTION_NAME.find().limit(NUMBER) -- 读取指定数量的数据记录，该参数指定从MongoDB中读取的记录条数。
14. db.COLLECTION_NAME.find().sort({KEY:1}) -- 使用 sort() 方法对数据进行排序，sort() 方法可以通过参数指定排序的字段，并使用 1 和 -1 来指定排序的方式，其中 1 为升序排列，而 -1 是用于降序排列。
15. db.COLLECTION_NAME.count() -- 统计记录
16. 模糊查询：MongoDB的模糊查询是通过正则表达式的方式实现的。格式为：/模糊查询字符串/
17. 包含与不包含：包含使用$in操作符。不包含使用$nin操作符。
18. 条件连接：$and操作符将条件进行关联
19. 列值自增：对某列值在原有值的基础上进行增加或减少，可以使用$inc运算符来实现