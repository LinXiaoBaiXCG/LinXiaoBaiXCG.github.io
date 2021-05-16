---
title: MyBatis中$和#的区别
date: 2019-06-16 23:00:00
tags: [框架]
categories: 编程
---

## 介绍

动态SQL是Mybatis的主要特性之一，在mapper中定义的参数传到xml中之后，在查询之前Mybatis会对其进行动态解析。Mybatis为我们提供了两种支持动态SQL 的语法：#{} 以及 ${}。

#{}：占位符号，可以防止SQL注入（替换结果会增加单引号''）

${}：SQL拼接符号（替换结果不会增加单引号''，表名作为变量、like和order by时使用，存在sql注入问题，需手动代码中过滤）



**#{ }**

例如，Mapper.xml中如下的 sql 语句：

```
select * from user where name = #{name};
```

动态解析为：

```
select * from user where name = ?; 
```

一个 #{ } 被解析为一个参数占位符 ? 。


**${ } **

例如，Mapper.xml中如下的 sql语句：

```
select * from user where name = ${name};
```

当我们传递的参数为 "Java" 时，上述 sql 的解析为：

```
select * from user where name = "Java";
```

预编译之前的 SQL 语句已经不包含变量了，完全已经是常量数据了。

综上所得， ${ } 变量的替换阶段是在动态 SQL 解析阶段，而 #{ }变量的替换是在 DBMS 中。

## 区别

#{}是预编译处理，${}是字符串替换。

Mybatis在处理#{}时，会将SQL中的#{}替换为？，调用PreparedStatement的set方法来赋值。

Mybatis在处理${}时，就是把${}替换成变量的值。

使用#{}可以有效的防止SQL注入，提高系统安全性。

## 用法

**1、能使用 #{ } 的地方就用 #{ }**

- 首先这是为了性能考虑的，相同的预编译 sql 可以重复利用。
- 其次，${ } 在预编译之前已经被变量替换了，这会存在 sql 注入问题。

例如，如下的 sql：

```
select * from ${tableName} where name = #{name}
```

假如，我们的参数 tableName 为 user; delete user; --，那么 SQL 动态解析阶段之后，预编译之前的 sql 将变为：

```
select * from user; delete user; -- where name = ?;
```

-- 之后的语句将作为注释，不起作用，因此本来的一条查询语句偷偷的包含了一个删除表数据的 SQL。

**2. 表名作为变量时，必须使用 ${ }**
这是因为，表名是字符串，使用 sql 占位符替换字符串时会带上单引号 ''，这会导致 sql 语法错误，例如：

```
select * from #{tableName} where name = #{name};
```

预编译之后的sql 变为：

```
select * from ? where name = ?;
```

假设我们传入的参数为 tableName = "user" , name = "Java"，那么在占位符进行变量替换后，sql 语句变为：

```
select * from 'user' where name='Java';
```

上述 sql 语句是存在语法错误的

## SQL预编译

**1. 定义：**
sql 预编译指的是数据库驱动在发送 sql 语句和参数给 DBMS 之前对 sql 语句进行编译，这样 DBMS 执行 sql 时，就不需要重新编译。

**2. 为什么需要预编译**

- JDBC 中使用对象 PreparedStatement 来抽象预编译语句，使用预编译。
- 预编译阶段可以优化 sql 的执行。预编译之后的 sql 多数情况下可以直接执行，DBMS 不需要再次编译，越复杂的sql，编译的复杂度将越大，预编译阶段可以合并多次操作为一个操作。
- 预编译语句对象可以重复利用。把一个 sql 预编译后产生的 PreparedStatement 对象缓存下来，下次对于同一个sql，可以直接使用这个缓存的 PreparedState 对象。
- mybatis 默认情况下，将对所有的 sql 进行预编译。