---
layout: post
title:  "Consul -- 分布式配置中心"
date:   2018-03-08 15:40:00
categories: Distributed
comments: true
---

项目中需要一个配置中心，以取代原先各服务在各自配置文件中写一坨依赖其他服务的配置，达到配置集中管理的目的。以前接触过zookeeper，最近发现consul也可以完成类似的事情，简略预研了一下它们之间的差别 :

```text
组件名      | 一致性的算法 | 数据中心 | 可视化的页面 | 提供接口方式 | 健康检查方式   | 基于组件来做业务进程的主备选举
zookeeper  | Paxos      | 支持单个 | no         | TCP        | 对业务 有 侵入 | 自带，可靠
consul     | Raft       | 支持多个 | WebUI      | HTTP/DNS   | 对业务 无 侵入 | 借助会话和kv操作，不可靠，会话过期会导致多主，需要续签

所以 :
1. 做配置管理，健康检查。则 consul 优于 zookeeper
2. 做主备选举。则 zookeeper 优于 consul
```

本篇讲述如何用 consul 来做 配置中心 和 健康检查


### 一. 搭建 consul 集群 ###

首先，我们需要搭建consul集群，consul集群中的每个结点，以 server 或 client 模式启动。

**1-1. 环境资源**

```text
172.16.4.10 ( server )
172.16.4.16 ( server )
172.16.4.15 ( server )

172.16.4.4  ( client )
172.16.4.12 ( client )
172.16.4.13 ( client )
```

**1-2. 安装**

consul不需要依赖其他组件，不像zookeeper还要依赖jdk，consul就一个安装包，在所有结点上执行下面的命令 :

```text
unzip consul_1.0.6_linux_amd64.zip
rm -f consul_1.0.6_linux_amd64.zip

mv consul /usr/local/bin/

mkdir -p /data/consul/log
mkdir -p /data/consul/conf
mkdir -p /data/consul/data
```

**1-3. 配置文件**

consul的结点，有 server 和 client 之分，配置文件的路径均为 /data/consul/conf/consul.json

server结点的配置文件，以 172.16.4.10 为例 :

```json
{
    "datacenter"         : "dc_1",                          // 数据中心
    "node_name"          : "consul_1",                      // 该结点的名字
    "bootstrap_expect"   : 1,                               // 集群中，只需要一个server结点的该值为1，其余都为0
    "bind_addr"          : "172.16.4.10",                   // 本机IP
    "client_addr"        : "172.16.4.10",                   // 本机IP
    "retry_join"         : ["172.16.4.16", "172.16.4.15"],  // 其他的server结点的地址
    "server"             : true,                            // 以server模式运行
    "ui"                 : true,                            // 是否在该结点上开启WebUI
    "raft_protocol"      : 3,
    "enable_debug"       : false,
    "enable_syslog"      : false,
    "data_dir"           : "/data/consul/data",
    "log_level"          : "INFO",
    "retry_interval"     : "3s",
    "rejoin_after_leave" : true
}
```

client结点的配置文件，以 172.16.4.4 为例 :

```json
{
    "datacenter"         : "dc_1",
    "node_name"          : "consul_4",
    "bootstrap_expect"   : 0,                                              // client结点的该值一律为0
    "bind_addr"          : "172.16.4.4",
    "client_addr"        : "172.16.4.4",
    "retry_join"         : ["172.16.4.10", "172.16.4.16", "172.16.4.15"],  // server结点列表
    "server"             : false,                                          // 以client模式运行
    "ui"                 : true,
    "raft_protocol"      : 3,
    "enable_debug"       : false,
    "enable_syslog"      : false,
    "data_dir"           : "/data/consul/data",
    "log_level"          : "INFO",
    "retry_interval"     : "3s",
    "rejoin_after_leave" : true
}
```

不管是server模式还是client模式，配置文件中的 retry_join 都不是要写全的。因为consul使用的是gossip协议来传递信息，只要蔓延开来最终可以不漏掉任何一个server结点就行。

**1-4. 集群启停**

```text
启动 : consul agent -config-dir=/data/consul/conf > /data/consul/log/start.log 2>&1 &

关闭 : kill -INT $(ps -ef | grep consul | grep -v grep | awk '{print $2}')

查看集群成员 : consul members -http-addr=172.16.4.10:8500
// 请求地址可以选 任意一个结点
```

**1-5. WebUI**

一般情况下，consul集群是部署在内网环境下的，我们通常先登到一个公网跳板机，再从跳板机登到consul结点。

此时，我们要访问内网环境的consul的WebUI，可以通过ssh隧道的方式，将 consul_ip:8500 映射到 127.0.0.1:8500，然后访问 http://127.0.0.1:8500/ui 。


### 二. 配置中心 ###

用键值对的方式来管理配置项 :

```text
Read   : curl -X PUT http://ip:8500/v1/kv/键名 -d '值'
         // 键名可以是多级的，例如 aaa 例如 aaa/bbb
         // 插入 和 更新 都是此命令

Write  : curl -X GET http://ip:8500/v1/kv/键名            单键查询
         curl -X GET http://ip:8500/v1/kv/键名?recurse    递归查出所有子目录的

Delete : curl -X DELETE http://ip:8500/v1/kv/键名         单键删除
         curl -X DELETE http://ip:8500/v1/kv/键名?recurse 递归删除所有子目录的
```

使用配置中心，需要各业务进行配合 :

各业务启动的时候，将自己本业务的信息（例如自己的地址，端口，密码之类的），写入到consul中，以供其他业务使用。

各业务启动的时候，从consul中读取自己所需要的其他依赖服务的配置，写到自己的内存中，方便使用。


### 三. 健康检查 ###

首先，我在 172.16.4.15 上，起了一个nginx，用作测试的服务。

**3-1. 定义服务**

```text
curl -X PUT http://172.16.4.10:8500/v1/agent/service/register -d '{
    "ID"      : "service_nginx",
    "Name"    : "service_nginx",
    "Tags"    : [ "singleNginx" ],
    "Address" : "172.16.4.15",
    "Port"    : 80,
    "Check"   : {
        "DeregisterCriticalServiceAfter" : "60s",
        "HTTP"     : "http://172.16.4.15",
        "Interval" : "10s"
    },
    "EnableTagOverride" : false
}'

ID      : 服务ID
Name    : 服务名
Tags    : 服务标签
Address : 服务地址
Port    : 服务端口
Check   : 健康检查（以HTTP方式进行检查，当 http://172.16.4.15 返回 2xx 则表示健康。每隔10秒检查一次。当不健康超过60秒则移除该服务）
```

**3-2. 为健康检查定义监听**

consul watch -http-addr=172.16.4.10:8500 -type=checks -service=service_nginx nohup python /root/healthWatch.py &

- -http-addr   可以是任何结点
- -type=checks 表示监听的类型是健康检查
- -service     指定了监听的服务ID
- nohup python /root/healthWatch.py & 则是健康检查的结果发生变化时，触发的命令

下面给出一个 healthWatch.py 的示例

```python
# -*- coding: utf-8 -*-

import json

# consul watch 会把健康检查的结果，通过标准输入的方式传给脚本，所以要这样获取数据
serviceStatus = raw_input()
serviceStatus = json.loads(serviceStatus)

# 实际使用的时候，你可以根据健康检查的结果，执行（上报服务挂掉 or 重启服务 or 上报服务恢复）之类的逻辑
print serviceStatus
```
