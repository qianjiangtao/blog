---
title: consul安装
date: 2018-10-31 09:52:11
categories:
- consul
tags:
- consul
---

![](https://gitee.com/qianjiangtao/my-image/raw/master/blog/1534836753620.jpg)

<!--more-->

## consul安装(单机)

1.下载consul zip包([地址](https://www.consul.io/))，本人选择的是`consul_1.3.0_linux_386.zip`这个版本

2.解压zip,并将解压后的consul拷贝到`/usr/local/bin/`目录下方便使用

```shell
unzip consul_1.3.0_linux_386.zip
mv consul   /usr/locao/bin/
```

3.启动consul

```
consul agent -dev
```

```shell
2018/10/30 15:53:52 [DEBUG] agent: Using random ID "7e72c970-eb83-5d77-9d9b-9a1869632e9c" as node ID
2018/10/30 15:53:52 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:7e72c970-eb83-5d77-9d9b-9a1869632e9c Address:127.0.0.1:8300}]
2018/10/30 15:53:52 [INFO] serf: EventMemberJoin: lvmama05.dc1 127.0.0.1
2018/10/30 15:53:52 [INFO] serf: EventMemberJoin: lvmama05 127.0.0.1
2018/10/30 15:53:52 [INFO] agent: Started DNS server 127.0.0.1:8600 (udp)
2018/10/30 15:53:52 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
2018/10/30 15:53:52 [INFO] consul: Adding LAN server lvmama05 (Addr: tcp/127.0.0.1:8300) (DC: dc1)
2018/10/30 15:53:52 [INFO] consul: Handled member-join event for server "lvmama05.dc1" in area "wan"
2018/10/30 15:53:52 [DEBUG] agent/proxy: managed Connect proxy manager started
2018/10/30 15:53:52 [INFO] agent: Started DNS server 127.0.0.1:8600 (tcp)
2018/10/30 15:53:52 [INFO] agent: Started HTTP server on 127.0.0.1:8500 (tcp)
2018/10/30 15:53:52 [INFO] agent: started state syncer
2018/10/30 15:53:52 [INFO] agent: Started gRPC server on 127.0.0.1:8502 (tcp)
2018/10/30 15:53:52 [WARN] raft: Heartbeat timeout from "" reached, starting election
2018/10/30 15:53:52 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
2018/10/30 15:53:52 [DEBUG] raft: Votes needed: 1
2018/10/30 15:53:52 [DEBUG] raft: Vote granted from 7e72c970-eb83-5d77-9d9b-9a1869632e9c in term 2. Tally: 1
2018/10/30 15:53:52 [INFO] raft: Election won. Tally: 1
2018/10/30 15:53:52 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
2018/10/30 15:53:52 [INFO] consul: cluster leadership acquired
2018/10/30 15:53:52 [INFO] connect: initialized CA with provider "consul"
2018/10/30 15:53:52 [DEBUG] consul: Skipping self join check for "lvmama05" since the cluster is too small
2018/10/30 15:53:52 [INFO] consul: member 'lvmama05' joined, marking health alive
2018/10/30 15:53:52 [INFO] consul: New leader elected: lvmama05
2018/10/30 15:53:53 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
2018/10/30 15:53:53 [INFO] agent: Synced node info
2018/10/30 15:53:54 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
2018/10/30 15:53:54 [DEBUG] agent: Node info in sync
2018/10/30 15:53:54 [DEBUG] agent: Node info in sync
```



## consul服务注册与服务查询



### 服务注册

2.1.1 创建一个存放服务文件的文件夹

```shell
sudo mkdir /etc/consul.d #.d做后缀：表示一系列配置文件的存放目录（directory）
sudo chmod -R 777 /etc/consul.d/ #先改变文件夹权限
```

2.1.2 创建服务并且写入存放服务的文件夹

```shell
# 一个服务是以json文件的格式
echo '{"service":{"name":"firstservice","tags":["dev"],"port":80}}' > /etc/consul.d/firstservice.json

#也可以多个服务注册到一个文件中
# echo '{"service":【"name":"firstservice","tags":["dev"],"port":80}}' ,
# "name":"secondservice","tags":["dev"],"port":80}'】
# > /etc/consul.d/firstservice.json
```

2.1.3 启动consul服务

```shell
# -config-dir表示要加载的配置文件的目录，Consul将使用后缀“.json”或“.hcl”加载此目录中的所有文件。
consul agent -dev -config-dir /etc/consul.d/ 
```



### 服务发现



DNS方式

```shell
# '{"service":{"name":"firstservice","tags":["dev"],"port":80}}'
# -p 8600 dns端口，consul启动时控制台可以查看
#  tag.servicename.service.consul  tag:dev 和servicename:firstservice 都是创建服务的时候配置的
dig @127.0.0.1 -p 8600 dev.firstservice.service.consul
```

```shell
; <<>> DiG 9.9.4-RedHat-9.9.4-29.el7 <<>> @127.0.0.1 -p 8600 dev.firstservice.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12122
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;dev.firstservice.service.consul. IN	A

;; ANSWER SECTION:
dev.firstservice.service.consul. 0 IN	A	127.0.0.1

;; ADDITIONAL SECTION:
dev.firstservice.service.consul. 0 IN	TXT	"consul-network-segment="

;; Query time: 2 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: 二 10月 30 16:18:19 CST 2018
;; MSG SIZE  rcvd: 112
```



HTTP方式



```shell
# 访问的路径：host:port/版本号/catalog/service/服务名
curl http://localhost:8500/v1/catalog/service/firstservice
```

```json
[
    {
        "ID": "4e23fa9f-7e44-9e6c-9345-8a201568d155",
        "Node": "lvmama05",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "wan": "127.0.0.1"
        },
        "NodeMeta": {
            "consul-network-segment": ""
        },
        "ServiceKind": "",
        "ServiceID": "firstservice",
        "ServiceName": "firstservice",
        "ServiceTags": [
            "dev"
        ],
        "ServiceAddress": "",
        "ServiceWeights": {
            "Passing": 1,
            "Warning": 1
        },
        "ServiceMeta": {},
        "ServicePort": 80,
        "ServiceEnableTagOverride": false,
        "ServiceProxyDestination": "",
        "ServiceProxy": {},
        "ServiceConnect": {},
        "CreateIndex": 10,
        "ModifyIndex": 10
    }
]
```



### 服务检测

```shell
# 查询服务的健康状态
# 访问路径：端口/版本号/health/service/服务名称
curl http://localhost:8500/v1/health/service/firstservice?passing
```

```json
[
    {
        "Node": {
            "ID": "4e23fa9f-7e44-9e6c-9345-8a201568d155",
            "Node": "lvmama05",
            "Address": "127.0.0.1",
            "Datacenter": "dc1",
            "TaggedAddresses": {
                "lan": "127.0.0.1",
                "wan": "127.0.0.1"
            },
            "Meta": {
                "consul-network-segment": ""
            },
            "CreateIndex": 9,
            "ModifyIndex": 10
        },
        "Service": {
            "ID": "firstservice",
            "Service": "firstservice",
            "Tags": [
                "dev"
            ],
            "Address": "",
            "Meta": null,
            "Port": 80,
            "Weights": {
                "Passing": 1,
                "Warning": 1
            },
            "EnableTagOverride": false,
            "ProxyDestination": "",
            "Proxy": {},
            "Connect": {},
            "CreateIndex": 10,
            "ModifyIndex": 10
        },
        "Checks": [
            {
                "Node": "lvmama05",
                "CheckID": "serfHealth",
                "Name": "Serf Health Status",
                "Status": "passing",
                "Notes": "",
                "Output": "Agent alive and reachable",
                "ServiceID": "",
                "ServiceName": "",
                "ServiceTags": [],
                "Definition": {},
                "CreateIndex": 9,
                "ModifyIndex": 9
            }
        ]
    }
]
```



## consul集群部署

分别在三台机器上单独部署consul

```shell
# 已server的方式启动 -client 0.0.0.0使得客户端可以直接通过url访问服务端的consul ui
agent -server -bootstrap-expect 3 -data-dir /tmp/consul -node=agent-001 -bind=192.168.187.15 -client 0.0.0.0 -ui
agent -server -bootstrap-expect 3 -data-dir /tmp/consul -node=agent-002 -bind=192.168.187.16 -client 0.0.0.0 -ui
agent -server -bootstrap-expect 3 -data-dir /tmp/consul -node=agent-003 -bind=192.168.187.17 -client 0.0.0.0 -ui
```

```shell
# 此时日志显示未能选举出leader
2018/10/30 19:22:21 [INFO] agent: (LAN) joined: 1 Err: <nil>
2018/10/30 19:22:23 [ERR] agent: failed to sync remote state: No cluster leader
2018/10/30 19:22:49 [ERR] agent: Coordinate update error: No cluster leader
2018/10/30 19:22:51 [ERR] agent: failed to sync remote state: No cluster leader
2018/10/30 19:23:14 [ERR] agent: Coordinate update error: No cluster leader
2018/10/30 19:23:18 [ERR] agent: failed to sync remote state: No cluster leader
2018/10/30 19:23:47 [ERR] agent: Coordinate update error: No cluster leader
2018/10/30 19:23:50 [ERR] agent: failed to sync remote state: No cluster leader
2018/10/30 19:24:12 [ERR] agent: failed to sync remote state: No cluster leader
```



三台独立的consul服务启动后，发现其实agent-001和agent-002，agent-003三个彼此谁都不知道谁，需要执行相互join才能形成集群

```shell
# 分别在不同的机器上执行
consul join 192.168.187.16 192.168.187.17
consul join 192.168.187.15 192.168.187.17
consul join 192.168.187.15 192.168.187.16
```

```shell
# 成功完成选举 agent-003为leader
2018/10/30 19:24:58 [INFO] serf: EventMemberJoin: agent-003 192.168.187.17
2018/10/30 19:24:58 [INFO] consul: Adding LAN server agent-003 (Addr: tcp/192.168.187.17:8300) (DC: dc1)
2018/10/30 19:24:58 [INFO] consul: Existing Raft peers reported by agent-003, disabling bootstrap mode
2018/10/30 19:24:58 [INFO] serf: EventMemberJoin: agent-003.dc1 192.168.187.17
2018/10/30 19:24:58 [INFO] consul: Handled member-join event for server "agent-003.dc1" in area "wan"
2018/10/30 19:25:05 [DEBUG] raft-net: 192.168.187.16:8300 accepted connection from: 192.168.187.17:55620
2018/10/30 19:25:05 [DEBUG] raft-net: 192.168.187.16:8300 accepted connection from: 192.168.187.17:39509
2018/10/30 19:25:05 [WARN] raft: Failed to get previous log: 1 log not found (last: 0)
2018/10/30 19:25:05 [INFO] consul: New leader elected: agent-003
2018/10/30 19:25:06 [INFO] agent: Synced node info
```



检测信息

```shell
consul members
Node       Address              Status  Type    Build  Protocol  DC   Segment
agent-001  192.168.187.15:8301  alive   server  1.3.0  2         dc1  <all>
agent-002  192.168.187.16:8301  alive   server  1.3.0  2         dc1  <all>
agent-003  192.168.187.17:8301  alive   server  1.3.0  2         dc1  <all>
```

```shell
# 使用DNS方式访问
# dig @host -p 端口+ Node名称 + node + DC + consul
dig @127.0.0.1 -p 8600 agent-001.node.dc1.consul
```

```
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 7199
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;agent-001.node.dc1.consul.	IN	A

;; ANSWER SECTION:
agent-001.node.dc1.consul. 0	IN	A	192.168.187.15

;; ADDITIONAL SECTION:
agent-001.node.dc1.consul. 0	IN	TXT	"consul-network-segment="

;; Query time: 6 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: 三 10月 31 09:26:38 CST 2018
;; MSG SIZE  rcvd: 106
```



consul ui界面

```shell
# 访问ui界面
http://host:8500/ui
```

![1540950161859](https://gitee.com/qianjiangtao/my-image/raw/master/consul/1540950161859.png)