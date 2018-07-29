---
layout: post
title:  "多步骤的异步任务模型"
date:   2018-07-28 18:14:00
categories: Distributed
comments: true
---

## 一、异步任务框架 ##

以往在处理异步任务的时候，有过三种选择 ：

**第一种选择**

- 描述 ：子线程。
- 劣势 ：线程间协调不好处理。代码耦合。

**第二种选择**

- 描述 ：把任务写进DB，由另一个进程每隔一段时间轮询DB并执行。（注，相比进程内部定期轮询，用crontab定期拉起消费者进程则更为可用。）
- 劣势 ：DB轮询压力大。实时性不高。

**第三种选择**

- 描述 ：把任务写进MQ，由另一个进程收取消息并执行。（注，性能和实时性均高于第二种选择。）
- 劣势 ：一个任务往往是由多个步骤组成的，不同任务之间，往往会存在很多相同的子步骤，造成很多重复代码。

当我们把场景定位于服务端的多步骤的复杂耗时的异步任务时，上述三种选择均不可取。其实第三种选择只是不够优雅，如果能消除重复的子步骤代码，那就是比较好的一个方案了。讨论下图的模型（参考了一些服务端模型后总结） ：

![async_frame]({{ site.url }}/static/img/2018-07-28-Async-Task_async_frame.png)

**优雅的做法如上图**

- 1、消费者被拆成多个模块，各模块只专注做好自己的事情。
- 2、各模块监听 非同名的 队列，以免相互干扰。
- 3、OSS 收到请求后，把整个任务写进DB，并且把任务的第一个子步骤写到 该子步骤对应的队列。
- 4、第一个子步骤 对应的 执行模块，收到消息，执行第一步。然后查DB，把下一个子步骤写到MQ中，以此串起来。

下面重点论述 任务、消息的数据结构 以及 OSS、执行模块的概要逻辑。

<br>

## 二、DB 中的 整个任务 ##

```json
{
    "TaskId"      : 任务编号,
    "TaskStatus"  : 任务状态,
    "TaskMessage" : 任务报错,
    "TaskCursor"  : 游标编号（进行到了第几子步骤）,

    "TimeCreate"  : 任务创建时间,
    "TimeStart"   : 任务开始时间,
    "TimeEnd"     : 任务结束时间,

    "Parameters"  : 子步骤共享参数,

    "Steps"       : [
        {
            "Code"            : 子步骤结果,
            "Message"         : 子步骤报错,
            "TimeStart"       : 子步骤开始时间,
            "TimeEnd"         : 子步骤结束时间,

            "NormalModule"    : 正常流程的处理模块,
            "NormalCommand"   : 正常流程的处理命令,
            "NormalTimeout"   : 正常流程的时间限制,
            "NormalRetry"     : 正常流程的重试限制,

            "RollbackModule"  : 回滚流程的处理模块,
            "RollbackCommand" : 回滚流程的处理命令,
            "RollbackTimeout" : 回滚流程的时间限制,
            "RollbackRetry"   : 回滚流程的重试限制
        },
        ...
        ...
    ]
}
```

**任务状态**

- 0 成功
- 1 未开始
- 2 执行中
- 3 回滚中
- other 失败

**子步骤共享参数**

- 包含 OSS 收到的参数 + 各子步骤均需要的公共参数 + 各子步骤的结果（以便后续子步骤能获取上游的执行结果）。

**处理模块 & 处理命令**

- 比如有一个叫 image 的执行模块，它可以处理的命令有 download_image，check_image，backup_image 等。

<br>

## 三、MQ 中的 步骤消息 ##

```json
{
    "TaskId"     : 所属任务,
    "Cursor"     : 当前游标,
    "Type"       : 当前类型,
    "Pamameters" : 共享数据,
    "Module"     : 执行模块,
    "Command"    : 执行命令,
    "Timeout"    : 执行时间限制,
    "Retry"      : 执行重试次数限制
}
```

**当前游标**

- 若一路的子步骤都是正常的，那么 ：消息中的游标 = 任务中的游标。
- 若在第n子步骤失败了，那么 ：任务中的游标就固定在n不会变了（它表示任务最远进行到步骤n）；而消息中的游标是会不断减一（它表示依次向前回滚）。

**当前类型**

- 0 正常流程
- 1 回滚流程

<br>

## 四、API 创建任务的逻辑 ##

例如，用户发起了一个 创建数据库实例 的任务。

**1、OSS查询任务流程配置**

```json
{
    "modify_ntp" : [
        {
            "normal"      : {
                "module"  : "resource",
                "command" : "check_resource",   // 资源检查
                "timeout" : 300,
                "retry"   : 0
            },
            "rollback"    : {
                "module"  : "monitor",
                "command" : "report_event",     // 上报告警
                "timeout" : 300,
                "retry"   : 3
            },
        },
        {
            "normal"      : {
                "module"  : "mysql",
                "command" : "init_instance",    // 初始化实例
                "timeout" : 1800,
                "retry"   : 3
            },
            "rollback"    : {
                "module"  : "mysql",
                "command" : "clean_instance",   // 清除初始化实例过程中留下的东西
                "timeout" : 900,
                "retry"   : 3
            },
        },
        {
            "normal"      : {
                "module"  : "resource",
                "command" : "deduct_resource",  // 扣除资源
                "timeout" : 200,
                "retry"   : 2
            },
            "rollback"    : {
                "module"  : "resource",
                "command" : "restore_resource", // 恢复资源
                "timeout" : 200,
                "retry"   : 2
            },
        }
    ],
    ...
    ...
}
```

即有这样的一个逻辑 ：

![async_frame]({{ site.url }}/static/img/2018-07-28-Async-Task_task_example.png)

**2、OSS生成任务，并写入DB**

```json
{
    "TaskId"      : 全局唯一的ID,
    "TaskStatus"  : 1,
    "TaskMessage" : null,
    "TaskCursor"  : 0,
    "TimeCreate"  : 当前秒级时间戳,
    "TimeStart"   : null,
    "TimeEnd"     : null,
    "Parameters"  : {
        "Cpu"     : 4,
        "Memory"  : 8,
        "Storage" : 500
    },
    "Steps"       : [
        {
            "Code"            : 0,
            "Message"         : null,
            "TimeStart"       : null,
            "TimeEnd"         : null,
            "NormalModule"    : "resource",
            "NormalCommand"   : "check_resource",
            "NormalTimeout"   : 300,
            "NormalRetry"     : 0,
            "RollbackModule"  : "monitor",
            "RollbackCommand" : "report_event",
            "RollbackTimeout" : 300,
            "RollbackRetry"   : 3
        },
        {
            "Code"            : 0,
            "Message"         : null,
            "TimeStart"       : null,
            "TimeEnd"         : null,
            "NormalModule"    : "mysql",
            "NormalCommand"   : "init_instance",
            "NormalTimeout"   : 1800,
            "NormalRetry"     : 3,
            "RollbackModule"  : "mysql",
            "RollbackCommand" : "clean_instance",
            "RollbackTimeout" : 900,
            "RollbackRetry"   : 3
        },
        {
            "Code"            : 0,
            "Message"         : null,
            "TimeStart"       : null,
            "TimeEnd"         : null,
            "NormalModule"    : "resource",
            "NormalCommand"   : "deduct_resource",
            "NormalTimeout"   : 200,
            "NormalRetry"     : 2,
            "RollbackModule"  : "resource",
            "RollbackCommand" : "restore_resource",
            "RollbackTimeout" : 200,
            "RollbackRetry"   : 2
        }
    ]
}
```

**3、OSS生成第一个子步骤的消息，并写入MQ**

```json
{
    "TaskId"      : 任务ID,
    "Cursor"      : 0,
    "Type"        : 0,
    "Parameters"  : {
        "Cpu"     : 4,
        "Memory"  : 8,
        "Storage" : 500
    },
    "Module"      : "resource",
    "Command"     : "check_resource",
    "Timeout"     : 300,
    "Retry"       : 0
}
```

<br>

## 五、各执行模块的逻辑 ##

```python
记 ${msg} 为 本模块收到的消息;

step_success = False;
step_retry   = ${msg}.Retry;

do {
    res = execStep( ${msg}.Command, ${msg}.Parameters );
    
    if ( res 表明成功 ) {
        将 res 中的必要数据 写入 ${msg}.Parameters，以供后续子步骤使用;
        step_success = True;
    }
    
    step_retry--;
} while( step_success == False && step_retry >= 0 );

查询DB，记 ${task} 为所属的任务;

if ( step_success && ${msg}.Type == 0 ) {
    // 成功 & 本次执行是正常流程，则准备 ：下一步 正常子步骤
}

if ( step_success && ${msg}.Type == 1 ) {
    // 成功 & 本次执行是回滚流程，则准备 ：上一步 回滚子步骤
}

if ( !step_success && ${msg}.Type == 0 ) {
    // 失败 & 本次执行是正常流程，则准备 ：这一步 回滚子步骤
}

if ( !step_success && ${msg}.Type == 1 ) {
    // 失败 & 本次执行是回滚流程，则准备 ：<系统异常告警>
}
```

注，上述伪代码没有将 Timeout 考虑进去，它应是在最外面套一层。
