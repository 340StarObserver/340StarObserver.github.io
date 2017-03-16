---
layout: post
title:  "Mysql -- GTID主从方案的学习"
date:   2017-03-15 15:13:00
categories: Mysql
comments: true
---


本篇讲述我学习mysql-GTID主从复制，主要内容是 :

        1. 基于事务的主从配置
        2. 主从切换


我的环境是 :

        主结点 ipA : 3306
        从结点 ipB : 3306
        // iptables已配置，允许访问3306端口


#### 1. 我的配置文件 ####

        [mysqld]
        log_bin                      = /mysql/logs/bin.log
        binlog-format                = row
        binlog-checksum              = CRC32
        binlog-rows-query-log-events = 1
        
        log-slave-updates      = true
        skip_slave_start       = 1
        slave-parallel-workers = 2
        
        gtid-mode                 = on
        enforce-gtid-consistency  = true
        master-info-repository    = TABLE
        relay-log-info-repository = TABLE
        
        datadir = /var/lib/mysql
        socket  = /mysql/sock/mysql.sock
        
        server-id = 10
        user      = mysql
        port      = 3306
        
        character-set-server = utf8
        symbolic-links       = 0
        sql_mode             = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
        
        [client]
        default-character-set = utf8
        socket = /mysql/sock/mysql.sock
        
        [mysqld_safe]
        log-error = /var/log/mysqld.log
        pid-file  = /mysql/pids/mysqld.pid
        socket    = /mysql/sock/mysql.sock
        
        // 注意
        // 1. 上面出现的路径，确保所属人和所属组都是mysql
        // 2. log_bin       不能位于临时目录下
        // 3. binlog-format 必须是 row
        // 4. binlog-rows-query-log-events=1 表示把sql语句也记下来
        // 5. log-slave-updates=true         表示从结点上也要更新日志
        // 6. skip_slave_start=1             表示从结点启动后，不会自动开启slave进程
        // 7. slave-parallel-workers=2       表示多线程复制，建议业务拆分，分成多个库，有效利用多线程复制
        // 8. 三个socket路径要一致
        // 9. 主结点和从结点的配置文件中，唯一不同的是 server-id


#### 2. 检查连接是否正常 ####

        在所有结点上，都试试看 :
            mysql -h localhost  -P 3306 -u root -p
            mysql -h 该结点的地址 -P 3306 -u root -p
        
        经常会出现改完配置文件后，发现无法登入的情况，此时需要重设root密码 + 增加远程连接的权限
        
        mysqld_safe --skip-grant-tables
            // 绕过权限验证来启动mysql
            // 注意，执行它之前，请先关闭mysql
        
        另外开一个Terminal窗口，执行以下操作 :
            mysql
            > use mysql
            > update user set password=password("NewPwd") where user="root";
            > flush privileges;
            > exit
            
            pkill -KILL -t pts/0
                // 杀掉原先用mysqld_safe方式启动的mysql
            
            再次启动mysql
            
            mysql -h localhost -P 3306 -u root -p
            > grant all privileges on *.* to 'root'@'localhost' identified by 'NewPwd' with grant option;
                // 若出现报错 ERROR 1820 (HY000): You must SET PASSWORD before executing this statement
                // 则需要先执行 set password = password('NewPwd');
                // 再次执行grant命令
            > grant all privileges on *.* to 'root'@'%'         identified by 'NewPwd' with grant option;
            > flush privileges;
            > exit
        
        然后再次用localhost，内网地址，外网地址都连接看看


#### 3. 创建复制用户 ####

        mysql -h localhost -P 3306 -u root -p
        > create user 'ReplicaUser'@'%' identified by 'xxxxxx';
        > grant replication slave on *.* to 'ReplicaUser'@'%' identified by 'xxxxxx';
        > flush privileges;
        > exit
        
        // 注意
        // 1. 只需主结点上创建用户
        // 2. 此处我创建的这个用户，允许从结点来自任何网段，如想限制从结点网段则可以把'%'换掉
        // 3. 此处我创建的这个用户，允许从结点复制任何库的任何表


#### 4. 在主结点上备份数据 ####

        innobackupex \
        > --defaults-file=/etc/my.cnf \
        > --host=ipA \
        > --port=3306 \
        > --user=root \
        > --password=xxxxxx \
        > /mydata/mysqlbackup
            // 进行一次全备
            // 1. 备份到 /mydata/mysqlbackup 这个目录下，会产生一个以时间为名的目录
            // 2. 建议像这样把所有库一起备份，包括 库mysql 和 库performance schema
        
        innobackupex \
        > --defaults-file=/etc/my.cnf \
        > --host=ipA \
        > --port=3306 \
        > --user=root \
        > --password=xxxxxx \
        > --apply-log \
        > /mydata/mysqlbackup/2017-03-16_14-42-01
            // 对刚才那份全备做事务一致性
            // 1. 2017-03-16_14-42-01 便是刚才全备生成的目录，需要拷贝到每个从结点
            // 2. 查看一下全备目录下的 xtrabackup_binlog_info
            //    记下事务号，比如我的是 02a9ad8e-0984-11e7-963e-00163e00e4e0:1-19
            //    这个事务号之后会用到


#### 5. 在从结点上恢复数据 ####

        先关闭从结点上的mysql
        
        把数据目录/var/lib/mysql压缩留一个备份，然后把原数据目录清空
        
        innobackupex \
        > --defaults-file=/etc/my.cnf \
        > --host=ipB \
        > --port=3306 \
        > --user=root \
        > --password=xxxxxx \
        > --copy-back /mydata/mysqlbackup/2017-03-16_14-42-01
        
        chown -R mysql:mysql /var/lib/mysql


#### 6. 在从结点上启动复制 ####

        启动从结点上的mysql，并登入
        
        reset master;
        set global gtid_purged='02a9ad8e-0984-11e7-963e-00163e00e4e0:1-19';
            // 这是之前记好的事务号
            // 如果不执行这两条命令，会导致之后　"Slave_SQL_Running: No"
        
        change master to master_host='ipA',
            master_port=3306,
            master_user='ReplicaUser',
            master_password='YourPwd',
            master_auto_position=1;
        
        start slave;
        
        show slave status \G
        
        set global read_only=1;
            // 别忘了把从结点设为只读


#### 7. 主从切换 ####

        在原主结点上 :
            set global read_only=1;
                // 原主作为新从，便要只读
            flush logs;
                // 刷新binlog
        
        在某原从结点上( 选它做新主 ) :
            stop slave;
            set global read_only=0;
                // 新主是可写的
        
        在其他的原从结点上( 仍作为从结点 ) :
            stop slave;
        
        在所有的新从结点上 :
            change master to master_host='新主地址',
                master_port=3306,
                master_user='ReplicaUser',
                master_password='YourPwd',
                master_auto_position=1;
                // 相比基于日志点的复制，事务复制在主从切换的时候，不用设置File和Position
            start slave;
            show slave status \G
