---
layout: post
title:  "mongo复制集的部署 & 复制集认证"
date:   2016-08-17 11:13:00
categories: Database
comments: true
---


本文讲述如何搭建最简单的1主2从的mongo复制集　以及　如何进行复制集的认证  

关于什么是mongo复制集和大多数原则的理论知识就不讲了，这里只讲部署上的完整步骤及每一步的细节  

我给的示例是全部的主从结点都放在了本机上，当然生产环境下建议把每个结点放到独立的机子上面  
并且通常情况下，生产环境下的一个复制集往往是集群中的一个分片  

环境版本 : 3.2.4  


### 一. 复制集的架构 ###

        复制集的架构为1个主结点，2个从结点  
        
        三个结点都部署在本地（当然主机充足的话，你也可以部署在不同机子上）  
        
        主结点的端口号为28001，从结点的端口号为28002和28003  


### 二. 结点28001的配置文件 28001.conf ###

        port=28001  
        
                该结点的端口号  
        
        bind_ip=127.0.0.1  
        
                该结点的绑定ip  
        
        dbpath=/home/seven/program/mongodb/data/db28001/  
        
                该结点的数据目录(绝对路径)  
        
        logpath=/home/seven/program/mongodb/log/28001.log  
        
                该结点的日志文件路径(绝对路径)  
        
        pidfilepath=/home/seven/program/mongodb/data/db28001/28001.pid  
        
                该结点的pid文件路径(绝对路径)  
        
        logappend=true  
        
                日志是否是追加方式(建议开启)  
        
        fork=true  
        
                是否以后台进程方式运行(建议开启)  
        
        replSet=warringstates  
        
                复制集的名称(同一个复制集下的主从结点的该值应相同)  
        
        profile=1  
        
                只在操作时间超过slowms时，才记录慢日志  
                profile=0表示永不记录，profile=1表示只记录慢日志，profile=2表示所有日志都记录  
        
        slowms=100  
        
                慢日志的时间阀值是100ms  
                
                另外，这两项配置推荐在所有结点上都进行配置  
                并且可以通过db.getProfilingStatus()这条命令来查看慢日志的配置信息  
                
                其实，不推荐通过配置文件的方式强制开启慢日志  
                相反，推荐使用db.setProfilingLevel(int)的方式动态调整profile的等级  
                因为，慢日志通常用在内测时候，真正上线后为了性能，是不会开启慢日志的，或者说是不会长期开着的  
                一般是每隔一段时间，动态地开启一段时间来获取线上测试情况  


### 三. 其他两个结点的配置文件 28002.conf 28003.conf ###

        和 28001.conf 相似  
        
        修改 :  端口，IP，数据目录，日志目录，pid目录  
        

### 四. 启动脚本 ###

init.sh　：  

        禁用hugepage，否则启动mongod进程的时候会有warning  
        另外这个脚本要以root方式运行，所以权限最好设置为700，所属组设置为root:root  

        echo never > /sys/kernel/mm/transparent_hugepage/enabled  
        
        echo never > /sys/kernel/mm/transparent_hugepage/defrag  


mongo28001.sh　：　启动结点28001的mongod进程的脚本，这个脚本最好不要以root身份运行  

        cd /home/seven/program/mongodb/bin  
        
                进入mongo的bin目录  
        
        ./mongod -f ../conf/28001.conf  
        
                启动mongod实例，-f后面跟上该结点的配置文件的路径  
                "../conf/28001.conf"是一个相对路径，即上面的28001.conf这个配置文件的路径  


mongo28002.sh　：　启动结点28002的mongod进程的脚本，这个脚本最好不要以root身份运行  

        cd /home/seven/program/mongodb/bin  
        
                进入mongo的bin目录  
        
        ./mongod -f ../conf/28002.conf  
        
                启动mongod实例，-f后面跟上该结点的配置文件的路径  
                "../conf/28002.conf"是一个相对路径，即上面的28002.conf这个配置文件的路径  


mongo28003.sh　：　启动结点28003的mongod进程的脚本，这个脚本最好不要以root身份运行  

        cd /home/seven/program/mongodb/bin  
        
                进入mongo的bin目录  
        
        ./mongod -f ../conf/28003.conf  
        
                启动mongod实例，-f后面跟上该结点的配置文件的路径  
                "../conf/28003.conf"是一个相对路径，即上面的28003.conf这个配置文件的路径  


### 五. 启动复制集 ###

        su root  
        
                切换为root身份，以便运行init.sh脚本来禁用hugepage  
        
        ./init.sh  
        
                禁用hugepage，（当然这里你需要进到你自己机子上存放该脚本的目录）  
        
        exit  
        
                退出root身份，以便启动mongod实例  
        
        ./mongo28001.sh  
        
                启动结点28001进程（当然这里你需要进到你自己机子上存放该脚本的目录）  
        
        ./mongo28002.sh  
        
                启动结点28002进程（当然这里你需要进到你自己机子上存放该脚本的目录）  
        
        ./mongo28003.sh  
        
                启动结点28003进程（当然这里你需要进到你自己机子上存放该脚本的目录）  


### 六. 初始化复制集 ###

        mongo 127.0.0.1:28001/admin  
        
                进入结点28001（其实随便进哪个结点都行）的shell，并指定一开始进入admin库  
        
        rs.status()  
        
                查看复制集状态  
                发现返回值是类似"run rs.initiate(...) if not yet done for the set"  
                说明还需要手动地初始化复制集  
        
        config = {  
                    _id : "warringstates",  
                    members :  
                    [  
                        { _id : 0, host : "127.0.0.1:28001" },  
                        { _id : 1, host : "127.0.0.1:28002" },  
                        { _id : 2, host : "127.0.0.1:28003" }
                    ]  
                }  
        
                创建复制集配置对象，其中id=warringstates是表示复制集的名称  
                且应该和配置文件中的一致  
                之后的一个数组中的每一个json对象都表示一个结点的id和地址和端口  
        
        rs.initiate(config)  
        
                使刚才的配置对象生效  
                过一会，敲回车键，会发现当前shell的提示符变成了"warringstates:PRIMARY>"，  
                或者先变成"warringstates:SECONDARY"后变成"warringstates:PRIMARY"，  
                则表示复制集初始化成功  
        
        rs.status()  
        
                再次查看复制集的状态，会发现多了很多内容  
        
        rs.conf()  
        
                查看复制集的配置  


### 七. 验证复制集（在主结点） ###

        use mydb  
        
                在主结点上创建名为mydb的库  
                它只是进入，并不会真的立即创建库  
                只有在真正有数据的时候，才会创建库  
        
        db.students.insert({"name":"lvyang","age":20})  
        
                在主结点上的mydb库中的students集合中插入一条数据  
        
        db.students.insert({"name":"shangyang","age":44})  
        
                在主结点上的mydb库中的students集合中插入一条数据  
        
        db.students.insert({"name":"yingji","age":72})  
        
                在主结点上的mydb库中的students集合中插入一条数据  
        
        show tables  
        
                查看主结点上的当前库下的集合列表  
        
        db.students.find()  
        
                查看主结点上的当前库下的students集合中的所有数据  


### 八. 验证复制集（在从结点） ###

        use mydb  
        
                在从结点上进入mydb库  
        
        show tables  
        
                在从结点上查看mydb下的集合列表。发现报错，提示未开启slaveOk  
        
        rs.slaveOk(true)  
        
                开启从结点的slaveOk，每次想要进到从结点的shell里查看数据，都要开启slaveOk  
        
        db.students.find()  
        
                在从结点上查看mydb库下的students集合里的数据，此时能够查到数据了  


### 九. 生成密钥文件（此步骤开始配置复制集的认证） ###

        openssl rand -base64 64 > keyfile.dat  

                生成64字节的密钥文件  

        chmod 600 keyfile.dat  

                建议把密钥文件的权限设置为600（针对启动mongo实例的那个用户）  
                接着需要把这个密钥文件拷贝到每个复制集结点上（包括主结点和从结点）  


### 十. 创建复制集的用户 ###

        mongo 127.0.0.1:28001/admin  

                进入主结点（我这里此时28001是主结点）的admin库  
                你可能配的时候28001不是主结点，请进你的主结点  
                注 : 主结点在任一时刻是唯一的，但是是可变的，即随着时间推移，可能换了一个结点成了主结点  

        db.createUser(  
            {  
                user  : your username of this admin库,  
                pwd   : your password,  
                roles :  
                [  
                    {  
                        role : "root",  
                        db   : "admin"  
                    }  
                ]  
            }  
        )  

                创建admin库的用户（身份为所有库的用户的管理者）  

        use 新库  

                进入你之前创建的新库  

        db.createUser(  
            {  
                user  : your username of this 新库,  
                pwd   : your password,  
                roles :  
                [  
                    {  
                        role : "dbOwner",  
                        db   : 此库  
                    }  
                ]  
            }  
        )  

                创建这个库的用户（身份为此库的拥有者）  

        注意，只需要在主结点上创建这两个用户  
        由于复制集的关系，每个从结点会自动创建这两个相同的用户  


### 十一. 开启复制集认证配置 ###

        keyFile=path/keyfile.dat  

                在每个结点（主结点，从结点）的配置文件中加入keyFile的配置项  

        auth=true  

                在每个结点（主结点，从结点）的配置文件中加入auth=true的配置项  

        db.shutdownServer()  

                关闭原先的集群  
                注意，最好按照 从结点 -> 主结点 的顺序，依次关闭各结点的进程  
                在关闭某个进程时，需要先进入admin库，然后执行这条命令  

        重新启动集群，命令同第五步  


### 十二. 验证复制集的认证（在主结点上进行验证） ###

        mongo 127.0.0.1:28001/admin  
        
                进入主结点（我这里是28001，你可能主结点不是28001，因为主结点时刻可能变化）的admin库  
        
        show users  
        
                查看admin库中的用户  
                发现查到了用户，说明复制集环境下，查用户是不需要认证的  
        
        show collections  
        
                查看admin库中的集合列表  
                发现报错，提示未进行认证，说明查集合和查数据之类的操作，是需要认证的  
        
        db.auth(admin库的用户，此用户的密码)  
        
                进行认证  
        
        show collections  
        
                再次查看admin库下的集合列表，这次成功了  
        
        use 新库  
        
                进入你之前创建的那个新库  
        
        db.auth(该库的用户，此用户的密码)  
        
                进行认证  
        
        show collections  
        
                查看这个库下的集合列表  
        
        db.某集合.find()  
        
                查看该库下的某个集合中的数据  


### 十三. 验证复制集的认证（在从结点上进行验证） ###

        mongo 127.0.0.1:28002/admin  
        
                进入从结点28002的admin库  
                当然你也可以进入28003这个从结点  
        
        db.auth(admin库的用户，该用户的密码)  
        
                进行认证  
        
        use 新库  
        
                切换到你之前创建的那个新库中  
        
        db.auth(该库的用户，该用户的密码)  
        
                进行认证  
        
        show collections  
        
                查看该库下的集合列表  
                发现报错，咦，明明认证成功了，为何会如此 ?  
                记起来了吧，从结点上查数据，需要 rs.slaveOk这条命令  
        
        rs.slaveOk(true)  
        
                允许该从结点读数据  
        
        show collections  
        
                再次查看该库下的集合列表，这次成功了  
