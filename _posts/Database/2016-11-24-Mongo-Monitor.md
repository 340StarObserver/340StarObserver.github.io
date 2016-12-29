---
layout: post
title:  "mongo监控命令与工具 "
date:   2016-11-24 20:00:00
categories: Database
comments: true
---


## 学习笔记 -- mongodb监控 ##


### 法一.　使用　explain 来分析语句执行情况 ###

        例如 db.your_collection.find({ x : 123, y : 456 }).explain("allPlansExecution")，查看你的语句的工作情况 :  
        
        比如实际用了哪个索引来查  
        比如扫描了多少文档  
        ...  
        
        explain 也常用来测试比较多个索引方案的优劣  



### 法二.　慢日志 ###

        不推荐在配置文件中设置，而是推荐在mongo-shell中动态地设置（一般需要测试的时候动态地开启，测试完后及时关闭）  
        
        db.setProfilingLevel( profile, slowms )  
            // 第一个参数表示慢日志的级别，取值是 0 1 2  
                profile=0  表示不记录慢日志  
                profile=1  表示只记录慢日志  
                profile=2  表示所有日志都记录  
            // 第二个参数表示慢日志的时间阀值，单位是毫秒  
        
        db.getProfilingStatus()  
            // 查看当前慢日志的配置信息，比如我的结果是 :  
            // { "was" : 0, "slowms" : 100 }  
            // 它说明目前 profile=0 & slowms=100  
        
        开启慢日志后，在当前库下就会出现system.profile这个collection  
            // 注意，你在哪个库下面开启慢日志，就只有那个库会有慢日志，每个开启慢日志的库下面都会有system.profile这个collection  
            // 注意，如果你在配置文件里设置慢日志的话，会导致每个库都开启慢日志  
        
        你可以查询当前库下的system.profile集合，获取你想要的慢日志  
            // 因为不同版本的mongo，这个集合里的document的字段数目和名称略有不同  
            // 推荐你用 db.system.profile.find().limit(2).sort({ ts : -1 }).pretty() 查出最近的2条慢日志  
            // 看看具体有哪些字段，就知道怎么写你想要的查询了  



### 法三.　查看最近15分钟内的服务状态 ###

        db.serverStatus()  
            // 必须在admin库下，才能使用这条命令  
            // 举个例子，我只打印出部分信息，db.serverStatus()['extra_info']，结果是 :  
                    
            {  
                "note"             : "field vary by platform",  
                "heap_usage_bytes" : 62236488,  
                "page_faults"      : 24  
            }  
                    
            // 其中 heap_usage_bytes 表示最近15分钟内，内存中的热数据占用的字节数  
            // 其中 page_faults      表示最近15分钟内，缺页中断的次数  



### 法四.　使用mongodb自带的 mongostat ###

        该脚本位于mongodb的安装目录的bin目录下  
        举个例子 :  
            
        > mongostat \  
        > --host host:port \  
        > --username admin库的用户 \  
        > --password admin库的密码 \  
        > --authenticationDatabase admin  
            
        注意，authenticationDatabase这个参数必须是admin  



### 法五.　查看当前系统的IO情况 ###

        举个例子 :  
        iostat -x 1  
            // 每秒输出一次  
            // 注意，它是linux-shell下执行的，并不是mongo-shell里的  
            
        示例输出 :  
        Linux 4.2.0-30-generic (localhost) 	2016年11月20日 	_x86_64_	(4 CPU)  
        avg-cpu:  %user   %nice %system %iowait  %steal   %idle  
                    6.37    0.01    1.74    0.92    0.00   90.96  
        Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util  
        sda               0.10     1.17    1.98    1.31    42.79   119.86    98.73     0.08   24.54   16.75   36.27   4.66   1.54  
            // 关注几个数据 :  
            // %iowait  是指CPU等待IO的比例  
            // %idle    是指CPU空闲的比例  
            // %util    是指设备（上述结果中可以看到是sda这块硬盘）在这一秒内的IO带宽的使用百分比  



### 法六.　使用mongo-shell自带的 benchRun ###

        举个例子 :  
        res = benchRun({  
            host        : '地址:端口',  
            username    : 'admin库的用户名',  
            password    : 'admin库的密码',  
            db          : '你要测试的库名',  
            ops         : [  
                {  
                    ns  : '你要测试的库名.集合名',  
                    op  : 'insert',  
                    doc : { 'x' : { '#RAND_INT' : [ 0, 100 ] } }  
                }  
            ],  
            parallel    : 20,  
            seconds     : 5  
        })  
        注意，benchRun需要在admin库下才能执行  
        参数 parallel 是指我要用多少个线程（连接）去并发地插入，注意它并不是说每秒20个并发，因为每个线程每秒可以完成许多次插入  
        参数 seconds  是指这个benchRun要执行多长时间  
            
        返回结果 :  
        {  
            "note"                       : "values per second",  
            "errCount"                   : NumberLong(0),  
            "trapped"                    : "error: not implemented",  
            "insertLatencyAverageMicros" : 11.217961300363536,  
            "totalOps"                   : NumberLong(344396),  
            "totalOps/s"                 : 68697.5910483047,  
            "findOne"                    : 0,  
            "insert"                     : 68697.5910483047,  
            "delete"                     : 0,  
            "update"                     : 0,  
            "query"                      : 0,  
            "command"                    : 0  
        }  
        注意，上述结果中的数值都是每秒平均值  
        例如insert=68697 是指每秒执行了这么多次插入  
        例如errCount     是指每秒的平均错误数  



### 法七.　使用 mtools 工具 ###

        mtools最主要的功能是用来分析mongodb的日志  
        它的github的地址是 : https://github.com/rueckstiess/mtools  
        使用说明文档 地址是 : https://github.com/rueckstiess/mtools/wiki  
        
        最常用的是三个模块 :  
        mlogfilter   :  根据时间对日志切片，慢日志过滤，日志转json  
        mloginfo     :  日志统计  
        mplotqueries :  根据日志绘图  
        
        列举一些命令 :  
        
        
        < 7-1. mlogfilter   模块 >  
        
            mlogfilter xx.log --namespace A库.B集合 --slow 100 --json | mongoimport \  
                --host 127.0.0.1 \  
                --port 28001 \  
                --username 你想要入库的库的拥有写权限的一个用户 \  
                --password 对应用户的密码 \  
                --db 你想要入库的库名 \  
                --collection 集合名  
                // 过滤慢日志（属于A库B集合的，超过100ms的），并以json的形式输出  
                // 把输出结果，通过管道，传给mongoimport，再导入到你想要的库里  
        
        
        < 7-2. mloginfo     模块 >  
        
            mloginfo xx.log --distinct  
                // 把日志分类（比如接受连接啊，关闭啊），统计各种分类下的数量  
            
            mloginfo xx.log --connections  
                // 统计日志中连接的来源情况   
            
            mloginfo xx.log --queries  
                // 统计日志中慢查询的分类  
        
        
        < 7-3. mplotqueries 模块 >  
        
            mplotqueries xx.log --group operation --output-file 1.png  
                // 绘制慢查询的散点分布图（按operation的类型分组）  
            
            mlogfilter xx.log --namespace 库名.集合名 | mplotqueries --output-file 2.png  
                // 绘制慢查询的散点分布图（只看指定的集合）  
            
            mlogfilter xx.log --operation query | mplotqueries --type histogram --bucketsize 3600 --output-file 3.png  
                // 过滤出查询的日志，绘制每3600秒里面查询操作的情况  
            
            mplotqueries log1 log2 ... --type rsstate --output-file 5.png  
                // 根据多个日志文件（复制集里的每个结点的日志），绘制出该复制集状态的变动  
        
        
        < 7-4. mgenerate    模块 >  
        
            比如我定义了一个模板文件 template.json，内容如下 :  
                {  
                    "user"       : {  
                        "first"  : { "$choose" : [ "aaa", "bbb", "ccc" ] },  
                        "last"   : { "$choose" : [ "AAA", "BBB", "CCC" ] }  
                    },  
                    "gender"     : { "$choose" : [ "male", "female" ] },  
                    "age"        : "$number",  
                    "address"    : {  
                        "street" : { "$string" : { "length" : 10 } },  
                        "code"   : { "$number" : [ 10000, 99999 ] }  
                    }  
                }  
            
            mgenerate template.json --stdout --pretty --number 1  
                // 指定模板文件，定义结果输出是屏幕，并以友好方式显示结果，指定生产的document的数量是一个  
                // 结果如下（因为定义了随机数，故每次结果可能不同）:  
                    {  
                        "gender"     : "male",  
                        "age"        : 14,  
                        "user"       : {  
                            "last"   :"BBB",   
                            "first"  : "aaa"  
                        },  
                        "address"    : {  
                            "code"   : 10668,  
                            "street" : "o9ZKibDmck"  
                        }  
                    }  
            
            mgenerate template.json \  
                --number 100 \  
                --host 127.0.0.1 \  
                --port 28001 \  
                --database 库名 \  
                --collection 集合名 \  
                --processes 4  
                // 指定模板文件，指定产生100个document，并把结果写入mongodb（指定地址，端口，库名，集合名）  
                // processes参数是用来指定执行此操作的进程数（默认是CPU的数量）  
                // 注意，mgenerate，目前还不支持用户认证，所以 :  
                // 请在你的业务库之外，另建一个无需认证的库来做（因为mgenerate本身就是用来产生测试数据的）  
