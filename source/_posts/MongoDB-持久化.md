---
title: MongoDB-持久化
date: 2019-04-20 00:00:00
tags: [数据库]
categories: 编程
---

> Mongodb在1.8版本之后开始支持journal，就是我们常说的redo log，用于故障恢复和持久化。
> MongoDB可以保证当服务器崩溃或服务器硬关机或硬盘故障时的数据完整性

# MongoDB-Journaling

## 备份和恢复

当执行写操作时，MongoDB创建一个journal来包含确切磁盘位置和改变的字节。因此，如果服务器突然崩溃， 可以在启动时对日志进行重放，从而重新执行哪些停机前没有刷新到磁盘的操作。 

 数据文件默认每60s刷盘一次。数据文件存放到/data/db/journal目录里。 

 数据库被正常关闭后，journal日志文件被刷盘，journal被清空。 

##  **批量提交** 

Mongodb默认每隔100毫秒或者写入数据达到若干兆字节后，便会将这些操作写入到日志。即Mongodb不会在每次写入的时候都写入到日志，而是会成批量的提交更改。因此，在默认设置情况下，Mongodb不会丢失超过100毫秒的数据。
**因此，在Mongodb中存在丢失数据的可能。** 

# Journa工作原理

当系统启动时，mongodb会将数据文件映射到一块内存区域，称之为Shared view，在不开启journal的系统中，数据直接写入shared view，然后返回，系统每60s刷新这块内存到磁盘，这样，如果断电或down机，就会丢失很多内存中未持久化的数据。当系统开启了journal功能，系统会再映射一块内存区域供journal使用，称之为private view，mongodb默认每100ms刷新privateView到journal，也就是说，断电或宕机，有可能丢失这100ms数据，一般都是可以忍受的，如果不能忍受，那就用程序写log吧。这也是为什么开启journal后mongod使用的虚拟内存是之前的两倍。Mongodb的隔离级别是read_uncommitted，不管使用不使用journal，都是以内存中的数据为准，只不过，不开启journal，数据从shared view读取，开启journal，数据从private view读取。 

在开启journal的系统中，写操作从请求到写入磁盘共经历5个步骤，在serverStatus()中已经列出各个步骤消耗的时间。 

1.  Write to privateView 
2.  prepLogBuffer 

 Private view(PV) 中的数据并不是直接刷新到journal文件，而是通过一个中间内存块（journalbuffer，或者alogned buffer）一部分一部分的刷新到journal，这样可以提高并发。preplogbuffer即是将PV中的数据写入到aligned buffer中的过程。这个过程有两部分，basic write 操作和非 basic write操作（e.g.create file）。一次preplogbuffer是以一个commitJob为一个单位，可能会有很多个commitJob写入到aligned buffer，然后提交。一个commitJob中包含多个basic write 和非basic write 操作，basic write是存在Writeintent结构体中的，Writeintent记录了写操作的地址信息。非basic write 操作存在一个vector中。 

3. WritetoJournal 

 writetoJournal操作是将alignedbuffer刷新到JournalFile的过程。默认100ms刷新一次，由--journalCommitInterval 参数控制。writetoJournal会做一些checksum验证，将alignedbuffer进行压缩，然后将压缩过后的alignedbuffer写入到磁盘。写入磁盘后将删除已经满的Journal文件，更新lsn号到lsn文件。写操作到这一步就是安全的了，因为数据已经在磁盘上，如果使用getlasterror（j=true），这一步即可返回。 

4.  WritetoDataFile 

 WritetoDataFile是将未压缩的aligned buffer写入到shared view的过程，然后由操作系统刷新到磁盘文件中。WritetoDataFile首先会对aligned buffer进行严格的验证，确保没有改变过，然后解析aligned buffer，通过memcpy函数拷贝到shareview 

5.  RemaptoPrivateView 

RemaptoprivateView会将持久化的数据重新映射到PV，以减小PV的大小，防止它不断扩大，按照源码上说，RemaptoprivateView会两秒钟重新映射一次，大约有1000个view，不是一次全做完，而是一部分一部分的做。由于读操作是读取PV，所以在映射完成之后会有短暂的时间读取磁盘。

经过这四步，一个写操作就完成了，journal提高了数据的安全性，并不像想象中的会丢数据，重要的是如何使用和维护。