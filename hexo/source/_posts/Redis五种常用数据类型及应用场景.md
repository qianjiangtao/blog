---
title: Redis五种常用数据类型及应用场景
date: 2018-11-28 13:39:44
categories:
 Redis
tags:
 Redis五种常用数据类型及应用场景
---

![](https://gitee.com/qianjiangtao/my-image/raw/master/redis/VELADA-POR-LA-PAZ-685x325.jpg)

<!--more-->

## 一.String(字符串)

> 字符串类型是Redis最基础的数据结构。首先键都是字符串类型，而且其他几种数据结构都是在字符串类型基础上构建的，所以字符串类型能为其他四种数据结构的学习奠定基础。如图2-7所示，字符串类型的值实际可以是字符串（简单的字符串、复杂的字符串（例如JSON、XML））、数字（整数、浮点数），甚至是二进制（图片、音频、视频），但是值最大不能超过512MB。 



### 1.1常用命令

#### 1.1.1 单个key操作

```shell
set key value [ex seconds] [px milliseconds] [nx|xx]
get key
```

- `EX second` ：设置键的过期时间为 `second` 秒。 `SET key value EX second` 效果等同于 `SETEX key second value` 。
- `PX millisecond` ：设置键的过期时间为 `millisecond` 毫秒。 `SET key value PX millisecond` 效果等同于 `PSETEX key millisecond value` 。
- `NX` ：只在键不存在时，才对键进行设置操作。 `SET key value NX` 效果等同于 `SETNX key value` 。
- `XX` ：只在键已经存在时，才对键进行设置操作。

案例:

```shell
# 设置key并添加过期时间
127.0.0.1:6379> set hello world ex 10
OK
127.0.0.1:6379> ttl hello
(integer) 6
127.0.0.1:6379> ttl hello
(integer) 5
# 设置key，存在则不成功 同setnx
127.0.0.1:6379> set not-exists-key Y nx
OK
127.0.0.1:6379> set not-exists-key Y nx
(nil)
```



#### 1.1.2 批量操作

```shell
# 批量赋值，取值，节省网络开销
MSET key value [key value ...]
MGET key [key ...]
```

案例

```
127.0.0.1:6379> mset key1 value1 key2 value2 key3 value3
OK
127.0.0.1:6379> mget key1 key2 key3
1) "value1"
2) "value2"
3) "value3"
```



####  1.1.3 计数

```shell
127.0.0.1:6379> set num 1
OK
# 自增默认为1，返回结果
127.0.0.1:6379> incr num
(integer) 2
127.0.0.1:6379> incr num
(integer) 3
# 自减默认为1，返回结果
127.0.0.1:6379> decr num
(integer) 2
# 自增指定数字，返回结果
127.0.0.1:6379> incrby num 5
(integer) 7
# 自减指定数字，返回结果
127.0.0.1:6379> decrby num 5
(integer) 2
```



###  1.2 内部编码



- **int 编码**：保存long 型的64位有符号整数
  ```shell
  # 赋值int数字(小于20长度) 则内部编码使用int
  127.0.0.1:6379> set count 1234567891234567891
  OK
  127.0.0.1:6379> object encoding count
  "int"
  # 赋值int数字(大于等于20长度) 则内部编码使用embstr
  127.0.0.1:6379> set count 12345678912345678912
  OK
  127.0.0.1:6379> object encoding count
  "embstr"
  ```

- **embstr 编码**：保存长度小于44字节的字符串。

  ```shell
  # 短字符串(小于44位) 则内部编码使用embstr
  127.0.0.1:6379> object encoding hello
  "embstr"
  ```

  

- **raw 编码**：保存长度大于44字节的字符串。

  ```shell
  # 长字符串(大于等于44位) 则内部编码使用raw
  127.0.0.1:6379> set hello "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
  OK
  127.0.0.1:6379> object encoding hello
  "raw"
  ```

  

###  1.3 应用场景

- 缓存功能

  Redis作为缓存层，MySQL作为存储层，绝大部分请求的数据都是从Redis中获取 

  ```java
  user_cache_key = "user:userinfo:"+userId
  //从缓存中获取用户信息
  userInfo = redis.get(user_cache_key)
  if(userInfo == null){
      //未获取到信息则从数据库中获取
      userInfo = mysql.query(userId)
      //放入缓存
      redis.set(user_cache_key,userInfo)
  }
  retrun userInfo
  ```

- 计数

  许多应用都会使用Redis作为计数的基础工具 ，如：短信发送次数限制,一分钟内登录次数等

  ```java
  send_message_limit_key = "user:login:sendMessageCount:"+username
  value = redis.get(send_message_limit_key)
  //判断用户登录发送短信次数是否超过限制
  if(value > limit){
  	retrun errorJson;
  }
  //未超过限制，则增加一次
  redis.incr(send_message_limit_key)
  ```

- 共享Session 

  分布式系统中将用户登录信息放在各个服务器中是不现实的,可以使用Redis将用户的Session进行集中管理，在这种模式下只要保证Redis是高可用和扩展性的，每次用户更新或者查询登录信息都直接从Redis中集中获取。 

  ```java
  //判断用户是否登录
  loginUser = redis.get(sessionid)
  if(loginUser == null){
      return LOGIN;
  }
  .....
  ```

- 分布式锁

  在分布式环境中我们可以基于redis实现分布式锁，利用redis单线程和setnx命令实现分布式锁

  ```java
  private static final String LOCK_SUCCESS = "OK";
  private static final String SET_IF_NOT_EXIST = "NX";
  private static final String SET_WITH_EXPIRE_TIME = "PX";
  
  /**
   * 尝试获取分布式锁
   * @param jedis Redis客户端
   * @param lockKey 锁
   * @param requestId 请求标识
   * @param expireTime 超期时间
   * @return 是否获取成功
   */
  public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {
  
      String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
  
      if (LOCK_SUCCESS.equals(result)) {
          return true;
      }
      return false;
  }
  ```



##  二.Hash（哈希表）

###  2.1常用命令



#### 2.1.1 单个key操作

```shell
HSET key field value
HGET key field
```

案例

```shell
127.0.0.1:6379> hset user:userinfo username james
(integer) 1
127.0.0.1:6379> hset user:userinfo mobile 18855556666
(integer) 1
127.0.0.1:6379> hget user:userinfo username
"james"
127.0.0.1:6379> hget user:userinfo mobile
"18855556666"
127.0.0.1:6379> hdel user:userinfo username
(integer) 1
```

####  2.1.2  批量设置

```shell
HMSET key field value [field value ...]
HMGET key field [field ...]
```

案例

```shell
# 批量设置用户信息(username,age,mobile)
127.0.0.1:6379> hmset user:userinfo username james age 19 mobile 16658585252
OK
# 批量获取
127.0.0.1:6379> hmget user:userinfo username age mobile
1) "james"
2) "19"
3) "16658585252"
```



###  2.2 内部编码

哈希类型的内部编码有两种：

- ziplist（压缩列表）：当哈希类型元素个数小于hash-max-ziplist-entries配置（默认512个）、同时所有值都小于hash-max-ziplist-value配置（默认64字节）时，Redis会使用ziplist作为哈希的内部实现，ziplist使用更加紧凑的结构实现多个元素的连续存储，所以在节省内存方面比hashtable更加优秀。
- hashtable（哈希表）：当哈希类型无法满足ziplist的条件时，Redis会使用hashtable作为哈希的内部实现，因为此时ziplist的读写效率会下降，而hashtable的读写时间复杂度为O（1）

```shell
# 设置短字符串
127.0.0.1:6379> hmset user:userinfo username james age 19 mobile 16658585252
OK
# 查看内部编码使用的是ziplist
127.0.0.1:6379> object encoding user:userinfo
"ziplist"
# 设置长字符串
127.0.0.1:6379> hset user:info content "Redis modules are dynamic libraries, that can be loaded into Redis at startup or using the MODULE LOAD command. Redis exports a C API, in the form of a single C header file called redismodule.h. Modules are meant to be written in C, however it will be possible to use C++ or other languages that have C binding functionalities.Modules are designed in order to be loaded into different versions of Redis, so a given module does not need to be designed, or recompiled, in order to run with a specific version of Redis. For this reason"
(integer) 1
# 查看内部编码使用的是hashtable
127.0.0.1:6379> object encoding user:info
"hashtable"
```

###  2.3 应用场景

hash 类型十分适合存储对象类数据，相对于在 string 中介绍的把对象转化为 json 字符串存储，hash 的结构可以任意添加或删除‘字段名’，更加高效灵活。 

```shell
hmset user:1 username james email james@163.com mobile 15858585858
```



##  三 . List（列表）

### 3.1 常用命令

####  3.1.1单个key操作

```shell
# 从左边向列表中插入数据
LPUSH key value [value ...]
# 从右边向列表中插入数据
RPUSH key value [value ...]
# 从左边弹出
LPOP key
# 从右边弹出(删除)
RPOP key
#查询指定范围内的数据
LRANGE key start stop
```

案例

```shell
127.0.0.1:6379> lpush list a b c d e
(integer) 5
127.0.0.1:6379> lrange list 0 -1
1) "e"
2) "d"
3) "c"
4) "b"
5) "a"
127.0.0.1:6379> rpush list 1
(integer) 6
127.0.0.1:6379> lrange list 0 -1
1) "e"
2) "d"
3) "c"
4) "b"
5) "a"
6) "1"
127.0.0.1:6379> lpop list
"e"
127.0.0.1:6379> rpop list
"1"
```



### 3.2 内部编码

关于list的内部编码，不同版本的方法有所区别，先看Redis 3.2版本之前的做法：

- ziplist (压缩列表，当list中的元素个数小于list-max-ziplist-entries，默认512，且每个元素的大小都小于list-max-ziplist-value，默认64字节)
- linkedlist（链表，当list中的元素不满足ziplist的条件时，内部实现使用linkedlist）

在Redis 3.0版本的源码的`redis.conf`文件中可以看到：

```xml
list-max-ziplist-entries 512
list-max-ziplist-value 64
```

​	在Redis 3.2版本以及之后的版本，引入了一种叫做`quicklist`的数据结构，`quicklist`综合了`ziplist`和`linkedlist`的优点。从外部看，`quicklist`也是一个`linkedlist`，不过它的每个节点都是一个`ziplist`。 

```shell
127.0.0.1:6379> lpush liststr a b c d
(integer) 4
127.0.0.1:6379> object encoding liststr
"quicklist"
```



### 3.3 应用场景

- lpush+lpop=Stack（栈）
- lpush+rpop=Queue（队列）
- lpsh+ltrim=Capped Collection（有限集合）
- lpush+brpop=Message Queue（消息队列）



##  四. Set(集合)

### 4.1 常用命令

```shell
# 将一个或多个 member 元素加入到集合 key 当中，已经存在于集合的 member 元素将被忽略
SADD key member [member ...]
# 移除集合 key 中的一个或多个 member 元素，不存在的 member 元素会被忽略
SREM key member [member ...]
# 返回集合 key 的基数(集合中元素的数量)
SCARD key
# 判断 member 元素是否集合 key 的成员
SISMEMBER key member
# 返回一个集合的全部成员，该集合是所有给定集合的交集
SINTER key [key ...]
# 返回一个集合的全部成员，该集合是所有给定集合之间的差集。
SDIFF key [key ...]
```



案例

```shell
127.0.0.1:6379> sadd k1 v1 v2 v3
(integer) 0
127.0.0.1:6379> sadd k2 v2 v3 v4
(integer) 3
# 集合中元素的数量
127.0.0.1:6379> scard k1
(integer) 3
# 判断 member 元素是否集合 key 的成员
127.0.0.1:6379> sismember k1 v1
(integer) 1
# 求交集
127.0.0.1:6379> sinter k1 k2
1) "v3"
2) "v2"
# 求差集
127.0.0.1:6379> sdiff k1 k2
1) "v1"
```



### 4.2 内部编码

集合类型的内部编码有两种：

- intset（整数集合）：当集合中的元素都是整数且元素个数小于set-max-intset-entries配置（默认512个）时，Redis会选用intset来作为集合的内部实现，从而减少内存的使用。
- hashtable（哈希表）：当集合类型无法满足intset的条件时，Redis会使用hashtable作为集合的内部实现

```shell
# 当保存整数类型且少于512 内部编码使用intset
127.0.0.1:6379> sadd intkey 1 2 3 4 5 6
(integer) 6
127.0.0.1:6379> object encoding intkey
"intset"
# 当保存整数类型且大于512 内部编码使用hashtable
127.0.0.1:6379> sadd intkey 1 2 3 4 5 6 ... 512 513
127.0.0.1:6379> object encoding intkey
"hashtable"
# 保存为字符型的使用hashtable
127.0.0.1:6379> sadd strkey a b c d
(integer) 4
127.0.0.1:6379> object encoding strkey
"hashtable"

```



###  4.3 应用场景

​	集合类型比较典型的使用场景是标签（tag）。例如一个用户可能对娱乐、体育比较感兴趣，另一个用户可能对历史、新闻比较感兴趣，这些兴趣点就是标签。有了这些数据就可以得到喜欢同一个标签的人，以及用户的共同喜好的标签，这些数据对于用户体验以及增强用户黏度比较重要。例如一个电子商务的网站会对不同标签的用户做不同类型的推荐，比如对数码产品比较感兴趣的人，在各个页面或者通过邮件的形式给他们推荐最新的数码产品，通常会为网站带来更多的利益 。

```shell
# 1.给用户添加标签
sadd user:1:tags tag1 tag2 tag5
sadd user:2:tags tag2 tag3 tag5
 ...
sadd user:k:tags tag1 tag2 tag4

# 2.给标签添加用户
sadd tag1:users user:1 user:3
sadd tag2:users user:1 user:2 user:3
...
sadd tagk:users user:1 user:2

# 3. 删除用户下的标签 删除标签下的用户（同一个事务中）
srem user:1:tags tag1 tag5
srem tag1:users user:1
srem tag5:users user:1

# 4.计算用户共同感兴趣的标签
sinter user:1:tags user:2:tags
```



**其他场景**

- sadd=Tagging（标签）
- spop/srandmember=Random item（生成随机数，比如抽奖）
- sadd+sinter=Social Graph（社交需求）



##  五.SortedSet(有序集合)

### 5.1 常用命令

```shell
# 将一个或多个 member 元素及其 score 值加入到有序集 key 当中
ZADD key score member [[score member] [score member] ...]
# 返回有序集 key 的基数
ZCARD key
# 返回有序集 key 中，指定区间内的成员。其中成员的位置按 score 值递增(从小到大)来排序
ZRANGE key start stop [WITHSCORES]
# 返回有序集 key 中，指定区间内的成员。其中成员的位置按 score 值递减(从大到小)来排列
ZREVRANGE key start stop [WITHSCORES]
```

 案例

```shell
127.0.0.1:6379> zadd user 21 user1
(integer) 1
127.0.0.1:6379> zadd user 17 user2
(integer) 1
127.0.0.1:6379> zadd user 13 user3
(integer) 1
127.0.0.1:6379> zadd user 28 user4
(integer) 1
127.0.0.1:6379> zcard user
(integer) 4
127.0.0.1:6379> zrange user 0 -1
1) "user3"
2) "user2"
3) "user1"
4) "user4"
127.0.0.1:6379> zrevrange user 0 -1
1) "user4"
2) "user1"
3) "user2"
4) "user3"

```



### 5.2 内部编码

有序集合类型的内部编码有两种：

- ziplist（压缩列表）：当有序集合的元素个数小于zset-max-ziplist-entries配置（默认128个），同时每个元素的值都小于zset-max-ziplist-value配置（默认64字节）时，Redis会用ziplist来作为有序集合的内部实现，ziplist可以有效减少内存的使用。

- skiplist（跳跃表）：当ziplist条件不满足时，有序集合会使用skiplist作为内部实现，因为此时ziplist的读写效率会下降

  ```shell
  # 默认使用ziplist
  127.0.0.1:6379> zadd user 61 user:age
  (integer) 1
  127.0.0.1:6379> zadd user 23 user2:age
  (integer) 1
  127.0.0.1:6379> object encoding user
  "ziplist"
  # 元素个数超过128 内部编码使用skiplist
  127.0.0.1:6379> zadd user 16 k1 28 k2 23 k3...24 k130
  "skiplist"
  ```

  

### 5.3 应用场景

​	有序集合比较典型的使用场景就是排行榜系统。例如视频网站需要对用户上传的视频做排行榜，榜单的维度可能是多个方面的：按照时间、按照播放数量、按照获得的赞数。本节使用赞数这个维度，记录每天用户上传视频的排行榜。主要需要实现以下4个功能 ：

```shell
# 1.添加用户赞数
zadd user:ranking:2018-11-28 jack 3 #jack发布视频获得3个赞
# 2.如果之后再获得一个赞，可以使用zincrby
zincrby user:ranking:2018-11-28 jack 1
# 3.取消用户赞数,由于各种原因（例如用户注销、用户作弊）需要将用户删除
zrem user:ranking:2018-11-28 jack
# 4.展示获取赞数最多的十个用户
zrevrangebyrank user:ranking:2018-11-28 0 9 
# 5.展示用户信息以及用户分数,此功能将用户名作为键后缀，将用户信息保存在哈希类型中，至于用户的分数和排名可以使用zscore和zrank两个功能
hgetall user:info:jack
zscore user:ranking:2018-11-28 jack
zrank user:ranking:2018-11-28 jack
```

