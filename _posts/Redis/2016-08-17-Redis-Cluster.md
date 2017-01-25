---
layout: post
title:  "Redis-Cluster 分布式缓存 "
date:   2016-08-17 09:34:00
categories: Redis
comments: true
---


本文讲述如何使用Redis的集群来对key-value的数据进行分布式的缓存。  

至于Redis的原理和集群的机制，本文不作论述，我也不太了解，本文专注于集群的部署和使用。  

因此，本文的重点在于Redis集群的详细配置文件，以及详细的部署过程。  

要点：  

        1.　Redis的通用配置　与　结点个性配置  
        
        2.　部署缓存集群  
        
        3.　主结点扩展　与　从结点扩展  
        
        4.　主结点移除　与　从结点移除  
        
        5.　数据迁移均衡  


### 一.　版本架构 ###

        系统 : ubuntu15.10  

        版本 : redis-3.0.7-64bits  

        架构 : 三个主结点  

            ( 因为我是用作分布式缓存，所以没必要设置用来备份的从结点 )  
            ( 实际环境下不同主结点要部署在不同机子上 )  
            主结点A : 127.0.0.1:7001  
            主结点B : 127.0.0.1:7002  
        主结点C : 127.0.0.1:7003  


### 二.　所有结点的公共配置文件 common.conf ###

        daemonize yes  

                以后台方式运行  

        tcp-backlog 511  

                每个结点tcp最大连接数  

        timeout 0  

                客户端连接超时时间(秒)，该段时间内客户端没有发出任何命令，则关闭该连接  
                0表示禁用此项设置  

        tcp-keepalive 60  

                定期向客户端发送心跳探测(秒)  

        loglevel notice  
        
        databases 2  

                数据库的数量，默认数据库编号0，可用select命令在连接时指定库id  
                强烈推荐使用时用不同的库存放不同类的信息  

        stop-writes-on-bgsave-error yes  

                持久化出错时，停止写操作  

        rdbcompression yes  
        
        rdbchecksum yes  
        
        slave-serve-stale-data yes  
        
        slave-read-only yes  

                从结点只能处理读请求  

        repl-diskless-sync no  
        
        repl-diskless-sync-delay 5  
        
        repl-disable-tcp-nodelay yes  
        
        slave-priority 100  
        
        slowlog-log-slower-than 10000  

                超过10ms的命令将被记录慢日志  

        slowlog-max-len 128  
        
        latency-monitor-threshold 0  
        
        notify-keyspace-events ""  

        save 900 1  

                每隔900s，存在1个key被修改，发起快照保存  

        save 300 100  

                每隔300s，存在100个key被修改，发起快照保存  

        save 60 10000  

                每隔60s，存在10000个key被修改，发起快照保存  

        appendonly no  

                禁用aof持久化  
                这里我是把集群用作分布式缓存，所以禁用了aof持久化  
                如果你需要持久化，那么设置appendonly yes  

        appendfsync everysec  
        
        no-appendfsync-on-rewrite yes  
        
        auto-aof-rewrite-min-size 64mb  
        
        lua-time-limit 5000  
        
        aof-rewrite-incremental-fsync yes  
        
        hash-max-ziplist-entries 512  
        
        hash-max-ziplist-value 64  
        
        list-max-ziplist-entries 512  
        
        list-max-ziplist-value 64  
        
        set-max-intset-entries 512  
        
        zset-max-ziplist-entries 128  
        
        zset-max-ziplist-value 64  
        
        hll-sparse-max-bytes 3000  
        
        activerehashing yes  
        
        client-output-buffer-limit normal 0 0 0  
        
        client-output-buffer-limit slave 256mb 64mb 60  
        
        client-output-buffer-limit pubsub 32mb 8mb 60  
        
        hz 10  
        
        cluster-enabled yes  

                启用集群  

        cluster-node-timeout 15000  

                集群结点间超时(微秒)  

        cluster-migration-barrier 2  

                当一个主结点拥有至少2个从结点时，能够把自己的1个从结点让给没有从结点的主结点  

        cluster-require-full-coverage yes  
        
        cluster-slave-validity-factor 10  


### 三. 主结点A的配置文件 master7001.conf ###

        include /home/seven/program/redis/conf/common.conf  
        
                引入结点公共配置  
        
        port 7001  
        
                此结点的端口  
        
        pidfile /home/seven/program/redis/pid/master7001.pid  
        
                此结点的pid进程文件(推荐绝对路径)  
        
        logfile /home/seven/program/redis/log/master7001.log  
        
                此结点的日志文件(推荐绝对路径)  
        
        cluster-config-file /home/seven/program/redis/auto/master7001.conf  
        
                此结点启动后生成的配置信息(不同于本身的配置文件，它是自动生成的)  
        
        dir /home/seven/program/redis/data/  
        
                此结点的数据文件目录(推荐绝对路径)  
        
        appendfilename master7001.aof  
        
                此结点的aof型持久化文件  
                (只需文件名，且完整路径是上面的dir加上这个，即/home/seven/program/redis/data/master7001.aof)  
        
        dbfilename master7001.rdb  
        
                此结点的rdb型持久化文件  
                (只需文件名，且完整路径是上面的dir加上这个，即/home/seven/program/redis/data/master7001.rdb)  
        
        maxmemory 64mb  
        
                此结点的内存分配  
        
        maxmemory-policy allkeys-lru  
        
                此结点的所有键值对均采用LRU置换算法  
        
        auto-aof-rewrite-percentage 80-100  


### 四. 主结点B,C的配置文件 master7002.conf master7003.conf ###

        主结点B,C的配置文件，与A的配置只是端口和路径变了  


### 五. 安装 ruby ###

        sudo apt-get update  
        
        sudo apt-get install ruby  
        
        ruby -v  


### 六. 安装 rubygems ###

        https://rubygems.org/pages/download  
        
                下载rubygems，然后解压，进入它的目录  
        
        ruby setup.rb  
        
                安装rubygems，此命令需要root或者管理员身份  
        
        sudo apt-get install gem  
        
                安装gem  
        
        vi /var/lib/locales/supported.d/local  
        
                如果gem安装失败，则修改这个文件，追加一行内容 zh_CN.UTF-8 UTF-8  
        
        sudo locale-gen  
        
                如果gem安装失败，则继续上一条命令后执行此命令  
        
        sudo dpkg-reconfigure locales  
        
                如果gem安装失败，则继续上一条命令后执行此命令  
        
        gem install redis  
        
                安装rubygems-reids，此命令需要root或者管理员身份  
                然而发现报错，类似While executing gem ... (Gem::RemoteFetcher::FetchError)  
                多半是由于GFW的问题，需要更换源的网址  
        
        gem source -r https://rubygems.org/  
        
                删除原来的源网址  
        
        gem source -a https://ruby.taobao.org/  
        
                增加新的源网址  
        
        gem source -l  
        
                列出当前所有的源网址  
        
        gem install redis  
        
                再次安装rubygems-redis，此命令需要root或者管理员身份，这次发现成功安装  


### 七. 启动脚本 ###

init.sh　：　禁用hugepage。另外这个脚本要以root方式运行，所以权限最好设置为700，所属组设置为root:root  

        echo never > /sys/kernel/mm/transparent_hugepage/enabled  
        
        echo never > /sys/kernel/mm/transparent_hugepage/defrag  


start.sh　：　各结点的启动脚本  

        redis-server /home/seven/program/redis/conf/master7001.conf  

                指定配置文件，启动主结点A  

        redis-server /home/seven/program/redis/conf/master7002.conf  

                指定配置文件，启动主结点B  

        redis-server /home/seven/program/redis/conf/master7003.conf  

                指定配置文件，启动主结点C  


clusterset.sh　：　初始化集群的脚本  

        cd /home/seven/Downloads/redis/src  

                进入你的redis的src目录，即redis-trib.rb这个脚本的位置  

        ./redis-trib.rb create --replicas 0 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003  

                配置集群的信息  
                其中，replicas后面跟的0表示每个主结点有0个从结点，即此时没有从结点  
                再后面的三个字符串表示三个主结点的地址和端口  
                
                另外，如果你有备份的需求（一般当需要持久化的时候），比如你的每个主结点需要有一个从结点，加入有三个从结点，  
                它们的地址也是本地，端口分别是8001,8002,8003，那么修改后的命令是这样的  
                
                ./redis-trib.rb create --replicas 1  
                    127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003  
                    127.0.0.1:8001 127.0.0.1:8002 127.0.0.1:8003  
                    
                    // 其中 --replicas 1 表示每个主结点有一个从结点，且之后写完主结点列表后依次把各主结点的从结点写上  


### 八. 启动集群 ###

        su root  
        
                进入root身份，以便执行init.sh来禁用hugepage  
        
        ./init.sh  
        
                禁用hugepage，此脚本必须以root身份执行  
        
        exit  
        
                退出root身份，以便以非root身份来启动redis集群  
        
        ./start.sh  
        
                启动redis的各个结点  
        
        ./clusterset.sh  
        
                初始化集群配置  
                第二次启动集群的时候，是不需要执行这个脚本的，就算执行也会报错的，除非你把之前的数据删光了  
        
        cd /home/seven/Downloads/redis/src  
        
                进入你的redis的src目录，即redis-trib.rb这个脚本的位置  
        
        ./redis-trib.rb check 127.0.0.1:7001  
        
                通过主结点A来查看集群  
        
        ./redis-trib.rb check 127.0.0.1:7002  
        
                通过主结点B来查看集群  
        
        ./redis-trib.rb check 127.0.0.1:7003  
        
                通过主结点C来查看集群  


### 九. 检验集群 ###

        redis-cli -c -h 127.0.0.1 -p 7001  
        
                连接本地的7001端口的redis结点  
                注意，一定要加 -c 参数，这样才是以集群方式登入  
        
        set name shangyang  
        
                通过本地7001端口的redis结点，设置一个键值对  
        
        get name  
        
                通过本地7001端口的redis结点，获取name的值  
        
        quit  
        
                退出redis-cli  
        
        redis-cli -c -h 127.0.0.1 -p 7002  
        
                连接本地的7002端口的redis结点  
        
        get name  
        
                通过本地7002端口的redis结点，获取name的值  
        
        set name yingquliang  
        
                通过本地7002端口的redis结点，设置name的值  
        
        quit  
        
                退出redis-cli  


### 十. 增加主结点 ###

        "edit a new configure file"  
        
                编辑一个新的配置文件，例如 master7004.conf  
        
        redis-server /yourpath/master7004.conf  
        
                启动这个新的主结点  
        
        cd /home/seven/Downloads/redis/src  
        
                进入你的redis-trib.rb文件的目录  
        
        ./redis-trib.rb add-node 127.0.0.1:7004 127.0.0.1:7001  
        
                加入这个新的空结点到集群  
                第一个参数是这个新结点的ip:port  
                第二个参数是原来集群中任意一个结点的ip:port  
        
        ./redis-trib.rb reshard 127.0.0.1:7004  
        
                为这个新的空结点分配槽  
                ( 整个集群中主结点的总槽数是16384，尽量每个主结点的槽数差不多 )  
                如果不给新结点分配槽，该结点就无法存储数据  
                ( 新的集群有4个主结点，平均每个主结点16384/4=4096槽，新结点从原来的3个结点中每个拿过来4096/3=1365槽 )  
            
                接下来根据提示，选择: 要迁移的槽数量 + 要迁移到哪个结点上 + 从哪个结点上迁移  
                假设原来有N个结点，建议执行N次reshard命令，每次只从一个单独的结点上迁移过来16384/(N+1)/N个槽  
            
                这个命令一般用来进行在线的数据均衡  


### 十一. 增加从结点 ###

        "edit a new configure file"  
        
                编辑一个新的配置文件，例如 slave8001.conf  
        
        redis-server /yourpath/slave8001.conf  
        
                启动这个新的从结点  
        
        cd /home/seven/Downloads/redis/src  
        
                进入你的redis-trib.rb文件的目录  
        
        ./redis-trib.rb add-node 127.0.0.1:8001 127.0.0.1:7001  
        
                加入这个新的空结点到集群  
                第一个参数是这个新结点的ip:port  
                第二个参数是原来集群中任意一个结点的ip:port  
        
        redis-cli -c -h 127.0.0.1 -p 8001  
        
                进入这个8001结点的shell  
                注意，要加 -c 参数，这样才是以集群方式登入  
        
        cluster replicate xxxxxxxxxxxxx  
        
                后面的参数表示这个从结点的主结点的id  
                关于id，可以用./redis-trib.rb check ip:port 来查看  
                在执行这条命令的时候，对应的主结点很可能无法提供服务，所以请在压力小的时候进行从结点的扩展  
                另外，从结点是没有槽的，因为从结点只是用作备份  


### 十二. 删除从结点 ###

        cd /home/seven/Downloads/redis/src  

                进入你的redis-trib.rb文件的目录  

        ./redis-trib.rb del-node 127.0.0.1:8001 xxxxxxxxx  

                删除从结点  
                第一个参数是你要删除的从结点的ip:port  
                第二个参数是那个从结点的id，关于id，可以用./redis-trib.rb check ip:port 来查看  

        redis-cli -c -h 127.0.0.1 -p 8001 shutdown  

                关闭这个结点  
                既然它都从集群中移除了，再开着它也没用了，不如及早关闭它  


### 十三. 删除主结点 ###

        删除主结点之前，必须把该主结点的槽迁移到其他的主结点上  
        以下便是迁移槽的步骤 :  
        
        ./redis-trib.rb reshard 127.0.0.1:7001  
        
                目标 : 把要删除的7004主结点上的部分槽迁移到7001主结点上  
                假设目前7004结点上有x个槽，删除7004后如果剩下n个主结点，那么建议迁移x/n个槽到7001结点上  
                同理迁移x/n个槽到7002,7003上  
        
        ./redis-trib.rb reshard 127.0.0.1:7002  
        
                目标 : 把要删除的7004主结点上的部分槽迁移到7002主结点上  
        
        ./redis-trib.rb reshard 127.0.0.1:7003  
        
                目标 : 把要删除的7004主结点上的部分槽迁移到7003主结点上  
        
        ./redis-trib.rb del-node 127.0.0.1:7004 xxxxxxxxx  
        
                删除空的主结点  
                第一个参数是你要删除的主结点的ip:port  
                第二个参数是那个主结点的id，关于id，可以用./redis-trib.rb check ip:port 来查看  

        redis-cli -c -h 127.0.0.1 -p 7001 shutdown  

                关闭这个结点  
                既然它都从集群中移除了，再开着它也没用了，不如及早关闭它  
        
        ./redis-trib.rb check 127.0.0.1:7001  
        
                通过7001结点来检查集群  
        
        ./redis-trib.rb info 127.0.0.1:7001  
        
                通过7001结点来查看集群信息  
