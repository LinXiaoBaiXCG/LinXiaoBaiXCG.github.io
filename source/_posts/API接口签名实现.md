---
title: API接口签名实现
date: 2021-02-04 20:00:00
tags: [安全]
categories: 编程
---

>  近期，公司小游戏涉及上传得分操作，为防止用户通过篡改接口数据刷游戏榜单获得奖金。最终通过签名的方式设计接口，解决数据被篡改问题。

# 实现原理

接口签名是主流的通信安全解决方案。

优点：

1. 能有效防止数据被篡改
2. 能检验请求来源是否合法
3. 能检验请求是否具有唯一性

接口签名主要是请求方和接口提供方约定一个加密规则。请求方按照约定好的算法生成签名字符串，接口提供方验算签名即可知是否合法。

步骤通常如下：

1. 接口提供方给出appid和appsecret
2. 调用方根据appid和appsecret以及请求参数和时间戳，按照约定好的算法生成签名sign
3. 接口提供方验证签名

项目中需求是防止数据被篡改，最终简化为如下步骤：

1. 调用方对请求参数排序后加盐进行MD5加密，然后转为大写
2. 然后使用公钥对MD5再次加密，生成最终的签名sign
3. 接口提供方对签名sign进行校验

这里用到签名算法是MD5摘要算法和RSA非对称加密算法。[软件系统中常见的加密算法和实现](https://linxiaobaixcg.github.io/2020/10/10/%E8%BD%AF%E4%BB%B6%E7%B3%BB%E7%BB%9F%E4%B8%AD%E5%B8%B8%E8%A7%81%E7%9A%84%E5%8A%A0%E5%AF%86%E7%AE%97%E6%B3%95/)

以上步骤能有效防止数据被篡改，但无法校验请求来源是否合法和唯一性

# 代码实现

## 客户端加密示例

```java
// 参数拼接和排序
String data = "a:" + "a" + "b:" + "b" + "salt:" + "salt";
// MD5加密后转大写
Digester md5 = new Digester(DigestAlgorithm.MD5);
String digestHex = md5.digestHex(data).toUpperCase();
// 使用公钥加密得到sign
RSA rsa = new RSA(null, "这里是公钥");
String sign = rsa.encryptBase64(digestHex, KeyType.PublicKey);
```

## 服务端

```java
// 拼接MD5加密数据
String data = "a:" + "a" + "b:" + "b" + "salt:" + "salt";
// 对得分和盐进行MD5加密并转为大写
Digester md5 = new Digester(DigestAlgorithm.MD5);
String upperMD5 = md5.digestHex(data).toUpperCase();
// 对sign进行私钥解密
RSA rsa = new RSA(privateKey, null);
String decryptSign = new String(rsa.decrypt("调用方请求的sign", KeyType.PrivateKey));
// 用sign和解密字符串进行比较
if (upperMD5.equals(decryptSign)) {
    
}
```