---
title: consul简单使用
date: 2018-10-31 10:28:12
categories:
- consul
tags:
- consul
---

![](https://gitee.com/qianjiangtao/my-image/raw/master/blog/2018-11-1-14-19.jpg)

<!--more-->

consul key/value使用（命令使用）



1.查看全部key/value 

```shell
# ?recurse参数指定查询多个kv
curl -v http://localhost:8500/v1/kv/?recurse
```

```
* About to connect() to localhost port 8500 (#0)
*   Trying ::1...
* Connected to localhost (::1) port 8500 (#0)
> GET /v1/kv/?recurse HTTP/1.1
> User-Agent: curl/7.29.0
> Host: localhost:8500
> Accept: */*
> 
< HTTP/1.1 404 Not Found
< Vary: Accept-Encoding
< X-Consul-Index: 1
< X-Consul-Knownleader: true
< X-Consul-Lastcontact: 0
< Date: Wed, 31 Oct 2018 02:18:12 GMT
< Content-Length: 0
< 
* Connection #0 to host localhost left intact
```

2.增加一个key/value

```shell
# key:'test/key001';value:'test' PUT:添加请求(注意必须大写)
curl -X PUT -d 'test' http://localhost:8500/v1/kv/test/key001
```

```shell
# 返回true表示添加成功
true
```

```shell
# 查询添加的key/value
curl http://localhost:8500/v1/kv/test/key001
# 结果 value：base64编码
[{"LockIndex":0,"Key":"test/key001","Flags":0,"Value":"dGVzdDAwMQ==","CreateIndex":6601,"ModifyIndex":6622}]
```



3.修改key/value

```shell
# 跟添加一样，只是保持key不变
curl -X PUT -d 'test003' http://localhost:8500/v1/kv/test/key001
```

```shell
# 查询
curl http://localhost:8500/v1/kv/test/key001
# 结果 注意modifyIndex也会随着修改的次数变更
[{"LockIndex":0,"Key":"test/key001","Flags":0,"Value":"dGVzdDAwMw==","CreateIndex":6601,"ModifyIndex":6750}]
```



4.删除key/value

```shell
curl -X DELETE http://localhost:8500/v1/kv/test/key001
true
# 查询key
空
```



consul key/value使用（UI界面操作）



![1540954148780](https://gitee.com/qianjiangtao/my-image/raw/master/consul/1540954148780.png)