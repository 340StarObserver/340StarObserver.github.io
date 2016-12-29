---
layout: post
title:  "mongo分片集群的部署以及集群认证"
date:   2016-08-18 20:20:00
categories: Database
comments: true
---


本文适合已经对mongo集群的理论知识已经有所了解的读者  

本文着重讲述集群部署上的完整步骤和细节，以及如何在集群上使用用户认证功能  

我给的示例的集群结构 ：  

    {  
        1个路由mongos结点（本地，27017端口）  
    
        1个元配置config结点（本地，27018端口）  
    
        2个分片shard结点（本地，28001以及28002端口）  
    }  

环境版本 :  

    {  
        mongo版本 : 3.2.4  
        
        系统版本 : ubuntu15.04  
    }  

当然，生产环境下，建议多个mongos结点和多个config结点，  
其中mongo推荐的config结点的数量是至少三个  
而且生产环境下，每个分片shard结点可以是一个复制集的主结点  

下面列出所有的步骤：  



### 先来不带认证的 ###



### 一. 第一个shard分片数据结点的配置文件 shard28001.conf ###

        port=28001  
        
                shard1的端口  
        
        bind_ip=127.0.0.1  
        
                shard1的绑定地址  
        
        dbpath=/home/seven/program/mongodb/data/shard28001/  
        
                shard1的数据目录(绝对路径)  
        
        logpath=/home/seven/program/mongodb/log/shard28001.log  
        
                shard1的日志目录(绝对路径)  
        
        pidfilepath=/home/seven/program/mongodb/data/shard28001/shard28001.pid  
        
                shard1的pid路径(绝对路径)  
        
        logappend=true  
        
                shard1的日志以追加方式  
        
        fork=true  
        
                shard1以后台方式运行  
        
        shardsvr=true  
        
                表明该mongo进程是一个分片结点  

        profile=1  
        
                只在操作时间超过slowms时，才记录慢日志  
                profile=0表示永不记录，profile=1表示只记录慢日志，profile=2表示所有日志都记录  
        
        slowms=100  
        
                慢日志的时间阀值是100ms  
                
                另外，这两项配置推荐在主结点和从结点上都进行配置  
                并且可以通过db.getProfilingStatus()这条命令来查看慢日志的配置信息  
                
                其实，不推荐通过配置文件的方式强制开启慢日志  
                相反，推荐使用db.setProfilingLevel(int)的方式动态调整profile的等级  
                因为，慢日志通常用在内测时候，真正上线后为了性能，是不会开启慢日志的，或者说是不会长期开着的  
                一般是每隔一段时间，动态地开启一段时间来获取线上测试情况  


### 二. 第二个shard分片数据结点的配置文件 shard28002.conf ###

        port=28002  
        
                shard2的端口  
        
        bind_ip=127.0.0.1  
        
                shard2的绑定地址  
        
        dbpath=/home/seven/program/mongodb/data/shard28002/  
        
                shard2的数据目录(绝对路径)  
        
        logpath=/home/seven/program/mongodb/log/shard28002.log  
        
                shard2的日志目录(绝对路径)  
        
        pidfilepath=/home/seven/program/mongodb/data/shard28002/shard28002.pid  
        
                shard2的pid路径(绝对路径)  
        
        logappend=true  
        
                shard2的日志以追加方式  
        
        fork=true  
        
                shard2以后台方式运行  
        
        shardsvr=true  
        
                表明该mongo进程是一个分片结点  
        
        profile=1  
        
        slowms=100  


### 三. config元数据结点的配置文件 config27018.conf ###

        port=27018  
        
                config结点的端口  
        
        bind_ip=127.0.0.1  
        
                config结点的绑定地址  
        
        dbpath=/home/seven/program/mongodb/data/config27018/  
        
                config结点的数据目录(绝对路径)  
        
        logpath=/home/seven/program/mongodb/log/config27018.log  
        
                config结点的日志路径(绝对路径)  
        
        pidfilepath=/home/seven/program/mongodb/data/config27018/config27018.pid  
        
                config结点的pid路径(绝对路径)  
        
        logappend=true  
        
                config结点的日志以追加方式  
        
        fork=true  
        
                config结点以后台方式运行  
        
        configsvr=true  
        
                表明该mongo进程是一个元数据结点  

        profile=1  
        
        slowms=100  


### 四. mongos路由结点的配置文件 mongos27017.conf ###

        port=27017  
        
                mongos结点的端口  
        
        bind_ip=127.0.0.1  
        
                mongos结点的绑定地址  
        
        logpath=/home/seven/program/mongodb/log/mongos27017.log  
        
                mongos结点的日志路径(绝对路径)  
        
        pidfilepath=/home/seven/program/mongodb/data/mongos27017/mongos27017.pid  
        
                mongos结点的pid路径(绝对路径)  
        
        logappend=true  
        
                mongos结点的日志以追加方式  
        
        fork=true  
        
                mongos结点以后台方式运行  
        
        configdb=127.0.0.1:27018  
        
                表明该mongo进程是一个路由结点，且它只有一个元数据结点并且地址为本地且端口为27018  
        
        注意mongos路由结点是不需要也不能定义dbpath的  
        注意路由结点上不需要配置profile和slowms，同样在路由结点上也无法查看这两项设置  


### 五. 启动脚本 ###

init.sh　：  

        禁用hugepage，否则启动mongod进程的时候会有warning  
        另外这个脚本要以root方式运行，所以权限最好设置为700，所属组设置为root:root  

        echo never > /sys/kernel/mm/transparent_hugepage/enabled  
        
        echo never > /sys/kernel/mm/transparent_hugepage/defrag  


shard28001.sh　：　分片结点1的启动脚本，这个脚本最好不要以root身份运行  

        cd /home/seven/program/mongodb/bin  
        
                进入mongodb的bin目录  
        
        ./mongod -f ../conf/shard28001.conf  
        
                启动shard1结点(-f指定配置文件)  


shard28002.sh　：　分片结点2的启动脚本，这个脚本最好不要以root身份运行  

        cd /home/seven/program/mongodb/bin  
        
                进入mongodb的bin目录  
        
        ./mongod -f ../conf/shard28002.conf  
        
                启动shard2结点(-f指定配置文件)  


config27018.sh　：　元数据结点的启动脚本，这个脚本最好不要以root身份运行  

        cd /home/seven/program/mongodb/bin  
        
                进入mongodb的bin目录  
        
        ./mongod -f ../conf/config27018.conf  
        
                启动config结点(-f指定配置文件)  


mongos27017.sh　：　路由结点的启动脚本，这个脚本最好不要以root身份运行  

        cd /home/seven/program/mongodb/bin  
        
                进入mongodb的bin目录  
        
        ./mongos -f ../conf/mongos27017.conf  
        
                以mongos形式启动路由结点(-f指定配置文件)  


### 六. 启动各实例 ###

        su root  
        
                切换为root身份，以便运行init.sh脚本来禁用hugepage  
        
        ./init.sh  
        
                禁用hugepage，（当然这里你需要进到你自己机子上存放该脚本的目录）  
        
        exit  
        
                退出root身份，以便启动mongod实例  

        ./shard28001.sh  
        
                启动第一个分片结点  
        
        ./shard28002.sh  
        
                启动第二个分片结点  
        
        ./config27018.sh  
        
                启动元数据结点  
        
        ./mongos27017.sh  
        
                启动路由结点  
                若你的config结点不满3个，它会提示你在生产环境下最好至少三个config结点  


### 七. 添加分片成员 ###

        mongo 127.0.0.1:27017/admin  
        
                进入mongos（路由结点）的shell  
        
        db.runCommand({ "addShard" : "127.0.0.1:28001" })  
        
                添加分片结点28001到集群中  
                如果不事先进入admin库，会报错，所以需要连接时指定admin库，或者use admin  
        
        db.runCommand({ "addShard" : "127.0.0.1:28002" })  
        
                添加分片结点28002到集群中  


### 八. 创建一个测试库和集合 ###

        use 新库名  
        
                创建一个新的库（当然是在路由结点，即mongos的shell中进行）  
        
        db.新表名.insert({...})  
        
                插入若干条数据  
        
        db.新表名.ensureIndex({...})  
        
                为这个表建一个索引吧  
                例如我是db.usertable.ensureIndex({ userid : 1 },{ unique : true })  
        
        db.runCommand({ enablesharding : 新库名 })  
        
                使刚才创建的库能够被分片  
                （发现报错，提示必须在admin环境下，所以use admin，再执行这条命令）  
        
        db.runCommand({ shardcollection : "库名.表名", key : {...} })  
        
                为某个库中的某个表添加分片片键,例如我是 :  
                db.runCommand(  
                    {  
                        shardcollection : "weshare.usertable",  
                        key             : { "userid" : 1 },  
                        unique          : true  
                    }  
                )  
                注意，强烈推荐片键是索引的一部分前缀，有利于达到查询的局部化  


### 九. 查看集群状态 ###

        use config  
        
                进入config库，即元数据库  
                （当然这里还是在路由结点的mongos的shell里执行命令）  
        
        db.shards.find()  
        
                查看当前集群中有哪些shard结点  
        
        use 新库名  
        
                进入你刚才创建的那个库  
        
        db.表名.stats()  
        
                查看该库中的该表的集群状态  
                若发现里面最开始的sharded字段是true，则表示你的集群创建成功了  
        
        db.printShardingStatus()  
        
                查看分片状态  


### 再来加入认证的配置 ###



### 十. 生成密钥文件 ###

        openssl rand -base64 64 > keyfile.dat  

                生成64字节的密钥文件  

        chmod 600 keyfile.dat  

                建议把密钥文件的权限设置为600（针对启动mongo实例的那个用户）  
                接着需要把这个密钥文件拷贝到集群中每一个结点上（路由结点，元配置结点，分片结点上都要有这个密钥文件）  


### 十一. 创建集群用户 ###

        mongo 127.0.0.1:27017/admin  

                进入mongos路由结点（我这里是本地的27017端口，请根据自身的地址和端口来进行）  

        db.createUser(  
            {  
                user  : yourusername,  
                pwd   : yourpassword,  
                roles :  
                [  
                    { role : "root", db : "admin" },  
                    { role : "clusterAdmin", db : "admin" }  
                ]  
            }  
        )  

                创建针对admin库的管理员用户  

        use 新库  

                进入你之前创建的新库  

        db.createUser(  
            {  
                user  : yourusername,  
                pwd   : yourpassword,  
                roles :  
                [  
                    { role : "dbOwner", db : yourdb },  
                    { role : "clusterAdmin", db : "admin" }  
                ]  
            }  
        )  

                创建针对某个这个新库的用户  

        注意，上述的两个用户，需要在每个结点（每个分片结点，每个路由结点）上都要创建  


### 十二. 开启集群认证 ###

        keyFile=path/keyfile.dat  

                在每个结点（路由结点，元配置结点，分片结点）的配置文件中加入keyFile的配置项  

        auth=true  

                在每个元配置结点和分片结点（即除了mongos结点）的配置文件中加入auth=true的配置项  

        db.shutdownServer()  

                关闭原先的集群  
                注意，需要按照 路由结点 -> 配置结点 -> 分片结点 的顺序，依次关闭各结点的进程  
                在关闭某个进程时，需要先进入admin库，然后执行这条命令  

        重新启动集群，命令同第六步  


### 十三. 验证集群的认证 ###

        mongo 127.0.0.1:27017/admin  
        
                进入路由结点（地址和端口请按照你的实际情况）  
        
        db.auth(yourusername,yourpassword)  
        
                admin库的认证  
        
        use 新库  
        
                进入之前你创建的新库  
        
        db.新表.insert({...})  
        
                往这个新库的新表里插入一条数据  
                发现报错没有权限，因为你需要进行auth，先执行db.auth(yourusr,yourpwd)  
                然后再执行这条命令  
        
        db.新表名.ensureIndex({...})  
        
                为这个新表添加索引  
        
        db.runCommand({ shardcollection : "库名.表名", key : {...} })  
        
                为这个新表添加片键。注意，强烈推荐片键是索引的一部分前缀，这样能够达到查询的局部化  
                注意，要先进入admin库，否则这条命令不会成功  
        
        db.表名.stats()  
        
                查看该库中的该表的集群状态  
                若发现里面最开始的sharded字段是true，则表示添加认证功能后的集群没有出现异常  
        
        db.printShardingStatus()  
        
                查看分片状态  
