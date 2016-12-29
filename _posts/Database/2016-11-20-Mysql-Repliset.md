---
layout: post
title:  "基于日志点的Mysql主从复制"
date:   2016-11-20 16:05:00
categories: Database
comments: true
---


本篇讲述我学习mysql基于二进制日志的复制  

我的环境是 :  

        主结点 : ipA:3306  
        从结点 : ipB:3306  
        
        // iptables已配置，允许访问3306端口  


### 一. 在主结点上建立用来复制的用户 ###

        create user 'seven'@'%' identified by 'xxxxxx';  
        
            // 创建用户seven，并且是任何网段的（即slave可以是任何网段下的），用来作为复制的用户  
        
        grant replication slave on *.* to 'seven'@'%' identified by 'xxxxxx';  
        
            // 为这个用户授权，可以复制任何库的任何表  


### 二. 备份主结点的数据###

        mysqldump \  
            --single-transaction \  
            --master-data=2 \  
            --triggers \  
            --routines \  
            university -h localhost -u root -p > university.sql  
            
            // 导出university这个库  
            // 参数 --single-transaction 是用来保证事务完整性  
            // 参数 --master-data = 1    是表示在备份文件中不注销change-master命令  
            // 参数 --master-data = 2    是表示在备份文件中会注销change-master命令  
            // 参数 --triggers           是表示把触发器也导出  
            // 参数 --routines           是表示把存储过程也导出  

        more university.sql  
        
            // 查看导出的备份文件，里面有一行 -- CHANGE MASTER TO MASTER_LOG_FILE='bin.000002', MASTER_LOG_POS=120;  
            // 其中的 --              表示是注释，对应了刚才的 --master-data = 2 这个参数  
            // 其中的 MASTER_LOG_FILE 表示日志文件，会在之后用到  
            // 其中的 MASTER_LOG_POS  表示日志点，会在之后用到  
            // 注意，这两个值也可以通过　show master status\G; 来查看  
        
        把导出的这个备份文件拷到从结点上（可以使用sftp,scp等）  
        

### 三.　在每个从结点上恢复数据 ###

        mysql -h localhost -u root -p university < university.sql  
        
            // 注意，事先要创建好university这个库，然后才能导入  


### 四. change master（在每个从结点上操作） ###

        进入从结点的mysql  
        
        change master to master_host = 'ipA',  
            master_port = 3306,  
            master_user = 'seven',  
            master_password = 'xxxxxx',  
            master_log_file = 'bin.000002',  
            master_log_pos = 120;  
        
            // 参数 master_host     是主结点的地址  
            // 参数 master_user     是主结点上之前创建的用来复制的用户  
            // 参数 master_password 是主结点上之前创建的用来复制的用户的密码  
            // 参数 master_log_file 是上面提到的备份文件中的那一行里的日志文件  
            // 参数 master_log_pos  是上面提到的备份文件中的那一行里的日志点  
        
        start slave;  
            
            // 启动slave复制进程  
            // 啊，提示报错　The server is not configured as slave; fix in config file or with CHANGE MASTER TO  
            // 我的mysql版本是5.6，查资料说是5.7以后才能不需要改配置文件，没办法只好在配置文件里设置了  
            
            修改主结点的配置文件 :  
                log_bin       = /var/run/mysqld/bin.log  
                binlog_format = mixed   # 采用混合模式（state模式 & row模式 的混合）  
                server-id     = 10  
            
            修改从结点的配置文件 :  
                skip_slave_start = 1    # 启动从结点mysql的时候，不会自动启动复制进程，而是需要用start slave来手动启动  
                read_only        = 1    # 这个从结点是只读的  
                server-id        = 20   # 该值必须和主结点，以及其他从结点不同  
                // 注意，从结点不需要再配置log_bin了，因为只需要主结点一份就够了  
            
            重启主结点和从结点的mysql  
            
            再次执行change master命令，以及start slave命令  
        
        show slave status \G;  
        
            // 查看slave进程的状态，若发现 :  
            // Slave_IO_Running : Yes  
            // Slave_SQL_Running: Yes  
            // 则主从配置基本上可以放心成功了  


### 五.　验证主从配置 ###

        在主结点上插入几条数据  
        
        在从结点上查询是否有了那几条新数据  


### 六.　多线程复制（在每个从结点上操作） ###

        注意 :  
            I . mysql5.7之前就已经支持多线程复制，但是，5.7之前的版本，一个线程只能作用于一个库，  
                当我只有一个库的时候，或者某个库的写操作占整体写操作的大比例的时候，多线程复制的效率反而不如单线程  
            II. mysql5.7开始，多线程复制是一个线程作用于一个表  
        
        show variables like 'slave_parallel%';  
        
            // 查看该从结点上，有关多线程复制的参数  
            // 可以看到有　         slave_parallel_workers　这个参数  
            // 若是mysql5.7，还会有 slave_parallel_type　   这个参数  
            // 其中，slave_parallel_type的默认值是'database'，可选值是'logical_clock'（即逻辑时钟的类型，该值相同的复制可以并发执行）  
        
        stop slave;  
        
            // 先停止该从结点上的slave服务  
        
        set global slave_parallel_type='logical_clock';  
        
            // 修改多线程复制的方式为逻辑时钟方式  
        
        set global slave_parallel_workers=4;  
        
            // 修改多线程复制的线程数为4（该值请根据CPU数量和能力来设置）  
        
        start slave;  
        
            // 再次启动该从结点上的slave服务    
