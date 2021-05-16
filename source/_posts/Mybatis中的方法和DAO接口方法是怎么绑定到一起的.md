---
title: Mybatis中的方法和DAO接口方法是怎么绑定到一起的
date: 2019-08-13 22:00:00
tags: [框架]
categories: 编程
---

> 在未理解代理前，一直有个疑问，Mybatis中DAO层接口没有写实现类，Mapper中的方法和DAO接口方法是怎么绑定到一起的，其内部是怎么实现的

# 原理

根据网上的一些知识点，讲一下原理:

Mybatis通过JDK的动态代理方式，在启动加载配置文件时，根据配置mapper的xml去生成Dao的实现。

session.getMapper()使用了代理，当调用一次此方法，都会产生一个代理class的instance,看看这个代理class的实现。

```
public class MapperProxy implements InvocationHandler { 
... 
public static <T> T newMapperProxy(Class<T> mapperInterface, SqlSession sqlSession) { 
    ClassLoader classLoader = mapperInterface.getClassLoader(); 
    Class<?>[] interfaces = new Class[]{mapperInterface}; 
    MapperProxy proxy = new MapperProxy(sqlSession); 
    return (T) Proxy.newProxyInstance(classLoader, interfaces, proxy); 
  } 

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable { 
    if (!OBJECT_METHODS.contains(method.getName())) { 
      final Class<?> declaringInterface = findDeclaringInterface(proxy, method); 
      final MapperMethod mapperMethod = new MapperMethod(declaringInterface, method, sqlSession); 
      final Object result = mapperMethod.execute(args); 
      if (result == null && method.getReturnType().isPrimitive()) { 
        throw new BindingException("Mapper method '" + method.getName() + "' (" + method.getDeclaringClass() + ") attempted to return null from a method with a primitive return type (" + method.getReturnType() + ")."); 
      } 
      return result; 
    } 
    return null; 
  } 
```

这里是用到了JDK的代理。 newMapperProxy()可以取得实现interfaces 的class的代理类的实例。

当执行interfaces中的方法的时候，会自动执行invoke()方法，其中public Object invoke(Object proxy, Method method, Object[] args)中 method参数就代表你要执行的方法.

MapperMethod类会使用method方法的methodName 和declaringInterface去取 sqlMapxml 取得对应的sql，也就是拿declaringInterface的类全名加上 sql-id