---
layout: post
title:  "基于Consul做主备选举"
date:   2018-03-08 20:34:00
categories: Distributed
comments: true
---

有一个数据库的管控系统，设计了如下的异步的任务执行架构 :

![frameA]({{ site.url }}/static/img/2018-03-08-Consul-Elect_elect_consul.png)

Step1. 管控页面接收用户的操作请求，将请求转发给OSS。

Step2. OSS将任务以键值对的方式写入consul，然后返回。

Step3. 有若干个消费者进程，基于consul做主备选举，主消费者从consul的固定目录下拉取任务执行。


### 一. 首先来讨论几个问题 ###

P1. OSS 往 consul 写入键值对的时候，是否需要加锁 ?

```text
不需要，直接新增键值对就行 :
curl -X PUT http://172.16.4.10:8500/v1/kv/oracleTask/task_${id} -d '${value}'

${id}    是一个随机字符串，由生产者填写，确保全局唯一就行
${value} 形如 {
    "taskId"    : "1",                 // 任务ID
    "taskTime"  : "1445599887",        // 任务时间戳
    "taskTaken" : "false",             // 表示该任务是否被领取
    "taskType"  : "CreateTableSpace",  // 任务类型
    "taskPara"  : {                    // 任务参数
        "instanceId" : "xxx",
        "initSize"   : "8000",
        "maxSize"    : "100000",
        "stepSize"   : "2000"
    }
}
```
  
P2. 若存在多个消费者，如何保证一个任务不会被重复消费 ?

```text
使用consul的乐观锁机制 :
curl -X PUT http://172.16.4.10:8500/v1/kv/oracleTask/task_${id}?cas=${原modifyIndex} -d {"taskTaken":true}
   
若返回 false，则表示该任务已经被其他消费者领走了。
若返回 true ，则表示当前消费者领取该任务成功，且taskTaken会被置为true。
```

P3. 如何让任务按照顺序被依次消费 ?

```text
消费者取出 oracleTask 目录下的任务后，过滤出 taskTaken = false 的任务，再按照 taskTime 升序排序，依次尝试对各任务加锁（尝试领走）。

同时只能有一个消费者在执行任务，这样就需要基于consul做主备选举，consul 的 acquire/release 可以做到。
```

### 二. 消费者的demo ###

```python
# -*- coding: utf-8 -*-

import os
import time
import json
import random
import base64
import urllib2

# ------------------------------
# constant variables

CONSUL_HOST = os.popen("/usr/sbin/ifconfig eth0 | grep inet | sed 's/^[ \t]*//g' | cut -d ' ' -f 2").read().strip("\r\n").strip("\n")
CONSUL_PORT = 8500
CONSUL_KEYPREFIX = "oracleTask/"

CONSUL_LEADER_KEY = "consumerLeader"
CONSUL_LEADER_VALUE = "kv for consumer leader election"

CONSUL_SESSION_LOCKDELAY = "120s"
CONSUL_SESSION_TTL = "120s"            # 要求此值必须大于一个任务的执行时间
CONSUL_SESSION_NAME = "consumer-lock"
CONSUL_SESSION_Behavior = "release"

CONSUL_CONSUMER_SLEEP = 3

# ------------------------------
# tool functions ( 关于领取任务 )

def sendHttpRequest(method, url, data):
    """ 发送HTTP请求 """
    req = urllib2.Request(url, data)
    req.get_method = lambda : method
    req.add_header("Content-Type", "application/json; charset=utf-8")
    res = urllib2.urlopen(req)
    return res.read()

def getTaskList():
    """ 获取所有未被领走的oracle任务 """
    reqMethod = "GET"
    reqUrl    = "http://%s:%d/v1/kv/%s?recurse" % (CONSUL_HOST, CONSUL_PORT, CONSUL_KEYPREFIX)
    rspData   = sendHttpRequest(reqMethod, reqUrl, None)
    rspData   = json.loads(rspData)

    taskList  = []
    n = len(rspData)
    i = 0
    while i < n:
        if  rspData[i]["Value"] is not None:
            rspData[i]["Value"] = json.loads(base64.b64decode(rspData[i]["Value"]))
            if rspData[i]["Value"]["taskTaken"] is False:
                taskList.append(rspData[i])
        i += 1
    
    taskList.sort(cmp = lambda lhs, rhs : lhs["Value"]["taskTime"] - rhs["Value"]["taskTime"])
    return taskList

def tryTakeTask(kvValue):
    """ 尝试领走一个任务, 返回true成功, false失败 """
    reqMethod = "PUT"
    reqUrl    = "http://%s:%d/v1/kv/%s?cas=%d" % (CONSUL_HOST, CONSUL_PORT, kvValue["Key"], kvValue["ModifyIndex"])
    reqData   = kvValue["Value"]
    reqData["taskTaken"] = True
    reqData   = json.dumps(reqData).encode("utf-8")
    if sendHttpRequest(reqMethod, reqUrl, reqData) == "true":
        return True
    return False

def deleteTask(key):
    """ 删除一个做完的任务 """
    reqMethod = "DELETE"
    reqUrl    = "http://%s:%d/v1/kv/%s" % (CONSUL_HOST, CONSUL_PORT, key)
    sendHttpRequest(reqMethod, reqUrl, None)

# ------------------------------
# tool functions ( 关于主备选举 )

def ensureElectKvExist():
    """ 保证用作consumer选举的键值对存在 """
    reqUrl = "http://%s:%d/v1/kv/%s" % (CONSUL_HOST, CONSUL_PORT, CONSUL_LEADER_KEY)
    try:
        sendHttpRequest("GET", reqUrl, None)
    except urllib2.HTTPError as err:
        sendHttpRequest("PUT", reqUrl, CONSUL_LEADER_VALUE)

def getNodeList():
    """ 获取node列表 """
    rsp = sendHttpRequest("GET", "http://%s:%d/v1/catalog/nodes" % (CONSUL_HOST, CONSUL_PORT), None)
    rsp = json.loads(rsp)
    return rsp

def createSession():
    """ 创建 session """
    nodeList  = getNodeList()
    reqMethod = "PUT"
    reqUrl    = "http://%s:%d/v1/session/create" % (CONSUL_HOST, CONSUL_PORT)
    reqData   = {
        "Node"      : nodeList[random.randint(0, len(nodeList) - 1)]["Node"],
        "Name"      : CONSUL_SESSION_NAME,
        "Behavior"  : CONSUL_SESSION_Behavior,
        "TTL"       : CONSUL_SESSION_TTL,
        "LockDelay" : CONSUL_SESSION_LOCKDELAY
    }
    reqData   = json.dumps(reqData).encode("utf-8")
    response  = sendHttpRequest(reqMethod, reqUrl, reqData)
    return json.loads(response)["ID"]

def deleteSession(sessionId):
    """ 销毁session """
    reqMethod = "PUT"
    reqUrl    = "http://%s:%d/v1/session/destroy/%s" % (CONSUL_HOST, CONSUL_PORT, sessionId)
    sendHttpRequest(reqMethod, reqUrl, None)

def renewSession(sessionId):
    """ renew session 以防止session过期 """
    reqMethod = "PUT"
    reqUrl    = "http://%s:%d/v1/session/renew/%s" % (CONSUL_HOST, CONSUL_PORT, sessionId)
    sendHttpRequest(reqMethod, reqUrl, None)

def tryLock(sessionId):
    """ 尝试抢锁, 返回true成功, false失败 """
    reqMethod = "PUT"
    reqUrl    = "http://%s:%d/v1/kv/%s?acquire=%s" % (CONSUL_HOST, CONSUL_PORT, CONSUL_LEADER_KEY, sessionId)
    rspData   = sendHttpRequest(reqMethod, reqUrl, CONSUL_LEADER_VALUE)
    if rspData == "true":
        return True
    return False

def tryRelease(sessionId):
    """ 尝试解锁, 返回true成功, false失败 """
    reqMethod = "PUT"
    reqUrl    = "http://%s:%d/v1/kv/%s?release=%s" % (CONSUL_HOST, CONSUL_PORT, CONSUL_LEADER_KEY, sessionId)
    rspData   = sendHttpRequest(reqMethod, reqUrl, CONSUL_LEADER_VALUE)
    if rspData == "true":
        return True
    return False

def whoIsLeader():
    """ 当前主节点的sessionId """
    reqMethod = "GET"
    reqUrl    = "http://%s:%d/v1/kv/%s" % (CONSUL_HOST, CONSUL_PORT, CONSUL_LEADER_KEY)
    rspData   = sendHttpRequest(reqMethod, reqUrl, None)
    rspData   = json.loads(rspData)
    if len(rspData) != 0 and "Session" in rspData[0]:
        return rspData[0]["Session"]
    return None

# ------------------------------
# 主流程 :

def execJob(taskDetail):
    """
    您可以重写该函数, 以实现您的任务执行逻辑
    返回true表示成功, false失败
    """
    print "start  execute a task"
    print "task detail is : " + json.dumps(taskDetail)
    print "finish execute a task"
    return True

def mainProcess():
    # 确保主备选举的键值对存在
    ensureElectKvExist()

    # 声明 session id
    session_id = None

    try:
        while True:
            print "=================================================="

            # 当前谁是主消费者
            leader = whoIsLeader()
            if leader is None:
                print "no one is leader, may be in election"
            else:
                print "leader is : " + leader
        
            # 创建session
            session_id = createSession()

            if tryLock(session_id):
                print "I am the leader"

                # 主消费者不断地工作
                while True:
                    # 1. 取出所有未被领走的任务
                    task_list = getTaskList()

                    # 2. 依次尝试各任务
                    n = len(task_list)
                    i = 0
                    while i < n:
                        if tryTakeTask(task_list[i]):
                            print "take task success : " + task_list[i]["Key"]

                            if execJob(task_list[i]["Value"]):
                                # 执行任务成功, 可选删除任务或将其移动至另外一个目录中
                                print "exec task success : " + task_list[i]["Key"]
                                deleteTask(task_list[i]["Key"])
                            else:
                                # 执行任务失败, 可选回写一些信息
                                print "exec task failed  : " + task_list[i]["Key"]
                        else:
                            print "take task failed  : " + task_list[i]["Key"]
                        
                        renewSession(session_id)
                        i += 1
                        print ""
                    
                    # 3. 续签 session
                    renewSession(session_id)

                    time.sleep(CONSUL_CONSUMER_SLEEP)
            else:
                print "I am not the leader"

            time.sleep(CONSUL_CONSUMER_SLEEP)
    except:
        # 退出时, 要解锁并销毁session, 以让其他备结点成为主节点
        if session_id is not None:
            tryRelease(session_id)
            deleteSession(session_id)
    finally:
        print "bye ~"

if __name__ == "__main__":
    mainProcess()
```

### 三. 遗留问题 ###

问题一. 由于consul采用的是gossip协议，故OSS连接consul的时候，是请求consul集群中的任何一个结点，但万一请求的那个结点挂了呢 ?

问题二. consul的会话是有过期时间的，尽管可以不断地续签会话，以保持会话有效，从而让其他备消费者无法也成为主消费者。但是若是一个任务的执行时间，已经超过了会话的有效期，那么当它还没来得及续签的时候，其他备消费者中的一个，也可以成为主消费者。这样的话，就会导致有多个主消费者领取任务（虽然领取任务是互斥的，可以保证多个主消费者不会领取到同一个任务，但是若任务之间可能存在冲突，则多消费者就有问题了）

问题三. consul的watch机制，是consul去拉起一个进程，每当有新的任务，就会创建一个新的进程，这样开销太大。而让消费者主动去拉取任务，性能也是不高的。

### 四. 合理的架构 ###

![frameB]({{ site.url }}/static/img/2018-03-08-Consul-Elect_elect_zk.png)

Step1. 多个MQ组成集群，通过HAproxy提供一个vip。（解决上述的问题一）。

Step2. OSS通过HAproxy提供的vip，将任务写入MQ。

Step3. 若干个消费者，基于zookeeper做主备选举（可靠），主消费者将会与MQ建立连接。（解决上述的问题二）。

Step4. MQ收到任务后，主动推给与之建立连接的主消费者。（解决上述的问题三）。
