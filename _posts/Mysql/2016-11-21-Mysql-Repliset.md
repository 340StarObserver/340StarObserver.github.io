---
layout: post
title:  "基于日志点的Mysql主从复制"
date:   2016-11-21 19:13:00
categories: Mysql
comments: true
---


本篇讲述我学习mysql基于二进制日志的复制，主要内容是 :  

        1. 方案一 : 基于 mysqldump    的主从  
        2. 方案二 : 基于 innobackupex 的主从( 推荐 )  
        3. 主从切换  


我的环境是 :  

        主结点 : ipA:3306  
        从结点 : ipB:3306  
        // iptables已配置，允许访问3306端口  


### 一.　基于 mysqldump 的主从 ###


#### 1-1. 在主结点上建立用来复制的用户 ####

        // 首先，主结点的配置文件中以下几项要设置 :  
        
        log_bin = /var/run/mysqld/bin.log
        server-id = 10
        
        // 进入主结点，进行如下操作 :  

        create user 'seven'@'%' identified by 'xxxxxx';  
            // 创建用户seven，并且是任何网段的（即slave可以是任何网段下的），用来作为复制的用户  
        
        grant replication slave on *.* to 'seven'@'%' identified by 'xxxxxx';  
            // 为这个用户授权，可以复制任何库的任何表  


#### 1-2. 备份主结点的数据 ####

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
        

#### 1-3.　在从结点上恢复数据 ####

        // 首先，从结点的配置文件中以下几项要配置 :  
        
        skip_slave_start = 1    # manually use start slave to start the slave process
        read_only        = 1    # this slave is read-only
        server-id        = 20
            // 注意，从结点上建议关闭 log_bin  
            // 注意，每个结点的server-id是不同的  

        mysql -h localhost -u root -p university < university.sql  
            // 注意，事先要创建好university这个库，然后才能导入  


#### 1-4. 在从结点上 change master ####

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
        
        show slave status \G  
            // 查看slave进程的状态，若发现 :  
            // Slave_IO_Running : Yes  
            // Slave_SQL_Running: Yes  
            // 则主从配置基本上可以放心成功了  


#### 1-5.　验证主从配置 ####

        在主结点上插入几条数据  
        
        在从结点上查询是否有了那几条新数据  

---


### 二.　基于 innobackupex 的主从 ###


#### 2-1.　预准备 : 配置文件 + 创建用来复制的用户 ####

        主结点 :  
        log_bin = /var/run/mysqld/bin.log
        binlog_format = mixed
        server-id = 10
        skip_slave_start = 1
        
        从结点 :  
        log_bin = /var/run/mysqld/bin.log
        binlog_format = mixed
        server-id = 20
        skip_slave_start = 1
            // 注意，主结点和从结点上，建议都开启log_bin，因为这样便于进行主从切换  
            // 注意，主结点和所有的从结点，server-id一定不能相同  
        
        
        在所有结点上(主结点和从结点都要，便于进行主从切换)，创建用来复制的用户 :  
        
        create user 'seven'@'%' identified by 'xxxxxx';  
            // 创建用户seven，并且是任何网段的（即slave可以是任何网段下的），用来作为复制的用户  
        
        grant replication slave on *.* to 'seven'@'%' identified by 'xxxxxx';  
            // 为这个用户授权，可以复制任何库的任何表  


#### 2-2.　备份主结点的数据 ####

        innobackupex \  
            --defaults-file=/etc/my.cnf \  
            --host=ipA --port=3306 \  
            --user=root --password=xxxxxx /mydata/mysqlbackup  
            
            // 进行全备份  
            // 备份到 /mydata/mysqlbackup 这个目录下，会产生一个以时间戳为名字的目录  
            // 注意，强烈推荐像这样把所有库一起备份  
            // 　　否则单单备份某个库，即mysql系统本身的一些库没有被备份的话  
            // 　　之后恢复的时候，把数据目录一删，会导致恢复后无法读出数据的  


        innobackupex \  
            --defaults-file=/etc/my.cnf \  
            --host=ipA --port=3306 \  
            --user=root --password=xxxxxx \  
            --apply-log /mydata/mysqlbackup/2017-01-29_21-03-48  
            
            // 用来保持事务的一致性  
            // 此处的 /mydata/mysqlbackup/2017-01-29_21-03-48 便是刚才全备份的时候产出的目录  
            // 　　需要把该目录拷贝到新的从结点上  
        
        
        下面是线上添加从结点的操作  


#### 2-3.　在从结点上恢复数据 ####

        // 在mysql关闭的情况下，执行恢复数据的操作  
        
        // 首先，需要清空datadir  
        // ( 建议清空它之前，先压缩拷贝到安全的地方，万一之后的恢复操作失败，也不至于让mysql崩溃无法启动 )  
        
        // 然后 :  
        innobackupex \  
            --defaults-file=/etc/my.cnf \  
            --host=ipB --port=3306 \  
            --user=root --password=xxxxxx \  
            --copy-back /mydata/mysqlbackup/2017-01-29_21-03-48  
            
        // 最后要把datadir中所有文件改成 mysql:mysql所属 :  
        chown -R mysql:mysql /var/lib/mysql/*  


#### 2-4.　在从结点上 change master ####

        more /mydata/mysqlbackup/2017-01-29_21-03-48/xtrabackup_binlog_info  
            // 看一下这个目录中的 xtrabackup_binlog_info  
            // 我这边的结果是 bin.000001	1984  
            // 这两个数据会派用处  

        启动从结点上的mysql，并进入它的shell  
        
        change master to master_host = 'ipA',  
            master_port = 3306,  
            master_user = 'seven',  
            master_password = 'xxxxxx',  
            master_log_file = 'bin.000001',  
            master_log_pos = 1984;  

        start slave;  
            // 启动slave复制进程  
        
        show slave status \G  
            // Slave_IO_Running : Yes  
            // Slave_SQL_Running: Yes  
            // 则成功  
        
        set global read_only=1;  
            // 最后，要在所有从结点上，设置只读  
            // 注意，也可在配置文件中设置只读，但不推荐那么做；而通过在线命令的方式进行设置，便于进行主从切换  

---


### 三.　主从切换 ###

        我是基于方案二的，也就是说 :
            1. 主结点和从结点的配置文件中，都开启了log_bin
            2. 从结点的配置文件中，并没有设置只读，而是通过在线命令的方式设置只读的
            3. 主结点和从结点上，都创建了用来复制的用户
        
        
        首先，在原来的主结点上 :
            set global read_only=1;
            show variables like 'read_only';
                // 把原来的主结点设成只读，这样，在进行主从切换的这段时间里，就不会有新的写操作过来
                // 改完后，通过show variables确认一下
            flush logs;
                // 刷新log_bin
            show master status \G
                // 查看主结点状态
        
        
        然后，在原来的某个从结点上(你要选它做新的主结点) :
            show slave status \G
                // 查看原slave状态
                // 若看到"Slave_SQL_Running_State: Slave has read all relay log"
                // 　　则说明主从数据完全一致，可以继续下面的操作
                // 　　否则，你需要等待一会
            stop slave;
                // 停止原来这个从结点上的slave进程
            show master status \G
                // 查看这个新的主结点的master进程的状态，记下结果中的 File 和 Position 字段的值
            set global read_only=0;
                // 原来这个结点作为从结点时，是只读的，现在它当主结点了，要把只读关掉
        
        
        然后，在原来的其他从结点上(它们仍然做从结点，但是它们的主变了) :
            show slave status \G
            stop slave;
                // 仍然是看到"Slave has read all relay log"，才可以关掉slave进程
                // 注意，当所有从结点做完这些操作后，才可以进行下面的操作
        
        
        然后，在新主结点之外的其他所有结点上(即 原主 + 原其他从) :
            change master to master_host = '新主的ip',
                master_port = 3306,
                master_user = 'seven',
                master_password = 'xxxxxx',
                master_log_file = '刚才要你记下的File的值',
                master_log_pos = 刚才要你记下的Position的值;
            start slave;
                // 设置它的主是谁，并启动slave进程
            show slave status \G
                // Slave_IO_Running : Yes　且　Slave_SQL_Running: Yes，则可以