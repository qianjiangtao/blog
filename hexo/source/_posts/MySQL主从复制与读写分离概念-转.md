---
title: MySQL主从复制与读写分离概念(转)
date: 2018-08-27 15:45:24
categories:
- mysql
tags:
- mysql
---

#### 1.MySQL主从复制入门



首先，我们看一个图： 

![wKiom1Q6MmzyVOk1AAC1h1tFlco436.jpg](https://gitee.com/qianjiangtao/my-image/raw/master/mysql/master-slave.jpg) 

影响MySQL-A数据库的操作，在数据库执行后，都会写入本地的日志系统A中。假设，实时的将变化了的日志系统中的数据库事件操作，在MYSQL-A的3306端口，通过网络发给MYSQL-B。MYSQL-B收到后，写入本地日志系统B，然后一条条的将数据库事件在数据库中完成。那么，MYSQL-A的变化，MYSQL-B也会变化，这样就是所谓的MYSQL的复制，即MYSQL replication。

​	在上面的模型中，MYSQL-A就是主服务器，即master，MYSQL-B就是从服务器，即slave。日志系统A，其实它是MYSQL的日志类型中的二进制日志，也就是专门用来保存修改数据库表的所有动作，即bin log。【注意MYSQL会在执行语句之后，释放锁之前，写入二进制日志，确保事务安全】日志系统B，并不是二进制日志，由于它是从MYSQL-A的二进制日志复制过来的，并不是自己的数据库变化产生的，有点接力的感觉，称为中继日志，即relay log。可以发现，通过上面的机制，可以保证MYSQL-A和MYSQL-B的数据库数据一致，但是时间上肯定有延迟，即MYSQL-B的数据是滞后的。【即便不考虑什么网络的因素，MYSQL-A的数据库操作是可以并发的执行的，但是MYSQL-B只能从relay log中读一条，执行下。因此MYSQL-A的写操作很频繁，MYSQL-B很可能跟不上。】



#### 2.主从复制的几种方式



**同步复制:**

所谓的同步复制，意思是master的变化，必须等待slave-1,slave-2,...,slave-n完成后才能返回。这样，显然不可取，也不是MYSQL复制的默认设置。比如，在WEB前端页面上，用户增加了条记录，需要等待很长时间。

**异步复制:**

如同AJAX请求一样。master只需要完成自己的数据库操作即可。至于slaves是否收到二进制日志，是否完成操作，不用关心。MYSQL的默认设置。

**半同步复制:**

master只保证slaves中的一个操作成功，就返回，其他slave不管。这个功能，是由google为MYSQL引入的。



#### 3.主从复制分析 



<u>问题1：master的写操作，slaves被动的进行一样的操作，保持数据一致性，那么slave是否可以主动的进行写操作？</u>

假设slave可以主动的进行写操作，slave又无法通知master，这样就导致了master和slave数据不一致了。因此slave不应该进行写操作，至少是slave上涉及到复制的数据库不可以写。实际上，这里已经揭示了读写分离的概念。



<u>问题2：主从复制中，可以有N个slave,可是这些slave又不能进行写操作，要他们干嘛？</u>

可以实现数据备份。类似于高可用的功能，一旦master挂了，可以让slave顶上去，同时slave提升为master。异地容灾，比如master在北京，地震挂了，那么在上海的slave还可以继续。主要用于实现scale out,分担负载,可以将读的任务分散到slaves上。【很可能的情况是，一个系统的读操作远远多于写操作，因此写操作发向master，读操作发向slaves进行操作】



<u>问题3：主从复制中有master,slave1,slave2,...等等这么多MYSQL数据库，那比如一个JAVA WEB应用到底应该连接哪个数据库?</u>

当然，我们在应用程序中可以这样，insert/delete/update这些更新数据库的操作，用connection(for master)进行操作，select用connection(for slaves)进行操作。那我们的应用程序还要完成怎么从slaves选择一个来执行select，例如简单的轮循算法。这样的话，相当于应用程序完成了SQL语句的路由，而且与MYSQL的主从复制架构非常关联，一旦master挂了，某些slave挂了，那么应用程序就要修改了。能不能让应用程序与MYSQL的主从复制架构没有什么太多关系呢？可以看下面的图：

![wKioL1Q6Z3aj0qD3AAFTRVV5DP00dd.jpg](https://gitee.com/qianjiangtao/my-image/raw/master/mysql/read-write.jpg) 

找一个组件，application program只需要与它打交道，用它来完成MYSQL的代理，实现SQL语句的路由。mysql proxy并不负责，怎么从众多的slaves挑一个？可以交给另一个组件(比如haproxy)来完成。这就是所谓的MYSQL READ WRITE SPLITE，MYSQL的读写分离。



<u>问题4：如果mysql proxy , direct , master他们中的某些挂了怎么办？</u>

总统一般都会弄个副总统，以防不测。同样的，可以给这些关键的节点来个备份。

<u>问题5：当master的二进制日志每产生一个事件，都需要发往slave，如果我们有N个slave,那是发N次，还是只发一次？</u> 

如果只发一次，发给了slave-1，那slave-2,slave-3,...它们怎么办？显然，应该发N次。实际上，在MYSQL master内部，维护N个线程，每一个线程负责将二进制日志文件发往对应的slave。master既要负责写操作，还的维护N个线程，负担会很重。可以这样，slave-1是master的从，slave-1又是slave-2,slave-3,...的主，同时slave-1不再负责select。slave-1将master的复制线程的负担，转移到自己的身上。这就是所谓的多级复制的概念。

<u>问题6：当一个select发往mysql proxy，可能这次由slave-2响应，下次由slave-3响应，这样的话，就无法利用查询缓存了。</u> 

应该找一个共享式的缓存，比如memcache来解决。将slave-2,slave-3,...这些查询的结果都缓存至mamcache中。 

<u>问题7：随着应用的日益增长，读操作很多，我们可以扩展slave，但是如果master满足不了写操作了，怎么办呢？</u> 

scale on ?更好的服务器？ 没有最好的，只有更好的，太贵了。。。scale out ? 主从复制架构已经满足不了。

可以分库【垂直拆分】，分表【水平拆分】。

[[原文地址]](http://blog.51cto.com/zhangfengzhe/1563032)