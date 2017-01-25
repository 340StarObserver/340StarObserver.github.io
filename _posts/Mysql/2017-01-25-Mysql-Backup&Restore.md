---
layout: post
title:  "Mysql - 全备份 & 增量备份 & 恢复"
date:   2017-01-25 14:09:00
categories: Mysql
comments: true
---


本篇讲述mysql的日常备份，包括 :  

        1. 逻辑备份 : 全备份 + 增量备份  
        2. 物理备份 : 直接把数据目录和日志目录全拷贝放到安全的地方  
        3. 恢复  


### 方案一.　mysqldump全备 + mysqlbinlog增备 ###

        // 该方案适合小数据量，且会锁表，只建议在业务低峰时使用  

        首先，确保配置文件中有这一项 :  
        [mysqld]  
        log_bin = /var/log/mysql/mysql-bin.log  
            // 最好让这个日志与数据库数据目录处于不同的硬盘上  
            // 在上述日志目录下，发现此时最新的日志文件的名字是 mysql-bin.000002  
        
        
        先进行一次全备 :  
        mysqldump \  
            --single-transaction \  
            --flush-logs \  
            --master-data=2 \  
            --triggers \  
            --routines \  
            university -u root -p > university-20170125.sql  
                // 把 university 这个库进行备份  
                // 由于有了 --triggers   这个参数，该库相关的触发器也被备份  
                // 由于有了 --routines   这个参数，该库相关的存储过程也被备份  
                // 由于有了 --flush-logs 这个参数  
                // 　　会在上述日志目录下生成一个新的日志文件mysql-bin.000003  
                // 　　且这个新的mysql-bin.000003便是之后拿来做增量恢复的  
        
        
        在这次全备完成后，我们对该库的数据做一些更改，来模拟生产环境下来自客户端的读写请求  
        
        
        再进行增备 :  
            把 mysql-bin.000003 这个增量日志，拷贝到安全的地方  
            // 通常，在生产环境下，建议每隔一小时，便对这个增量日志进行备份  
            // 这件事做得越勤，增量日志备份　与　实际最新数据　的差距便越小，一旦发生事故，损失也就越小  
        
        
        然后，我们模拟一次事故( 进行了一些误操作，导致数据被破坏 )  
        
        
        接下来我们恢复数据 ：  
            事故之前的最近数据 = 最近全备 + 该全备之后的最新增量日志  
            首先，要把该库清空  
            然后 :  
            mysql -u root -p university < university-20170125.sql  
                // 导入最近的全备  
            mysqlbinlog mysql-bin.000003 | mysql -u root -p  
                // 导入该全备之后的最新增量日志  
                // 注意，此处的 mysql-bin.000003 一定不能是日志目录下的运行时日志  
                // 　　　而是在事故发生前就拷贝到安全地方的最新的那份  
                // 　　　( 因为运行时日志已经由于误操作的影响而被污染了 )  
                // 　　　( 而且因为每隔一小时就备份一次增量日志，故取最新的那份 )  



### 方案二.　 innobackupex全备 + innobackupex增备 ###

        // 该方案适合大数据量，不会锁表  
        
        需要安装 xtrabackup :  
            源码编译，按官网来就行 https://www.percona.com/doc/percona-xtrabackup/2.4/installation/compiling_xtrabackup.html  
            然后将 /usr/local/xtrabackup/bin 添加到PATH中  
        
        
        先进行一次全备 :  
        innobackupex \  
            --defaults-file=YourPath/mysqld.cnf \  
            --host=127.0.0.1 --port=3306 \  
            --user=root --password=xxxxxx /home/seven/try/test  
                // 进行全备份  
                // 备份到/home/seven/try/test这个目录下，会产生一个以时间戳为名字的目录  
                // 注意，强烈推荐像这样把所有库一起备份  
                // 　　否则单单备份某个库，即mysql系统本身的一些库没有被备份的话  
                // 　　之后恢复的时候，把数据目录一删，会导致恢复后无法读出数据的  


        在这次全备完成后，我们对数据做一些更改，来模拟生产环境下来自客户端的读写请求  
        
        
        再进行第一次增备 :  
        innobackupex \  
            --incremental-basedir=/home/seven/try/test/2017-01-25_23-20-48 \  
            --incremental /home/seven/try/test \  
            --host=127.0.0.1 --port=3306 \  
            --user=root --password=xxxxxx  
                // 其中，--incremental-basedir 是刚才全备时产生的目录  
                // 其中，--incremental         是增量存放的目录，同样会产生一个以时间戳为名字的目录  
        
        
        在这次增备完成后，我们再对数据做一些更改  
        
        
        然后我们进行第二次增备 :  
            // 由于这次增备是在上一次增备基础上的，所以 --incremental-basedir 要填上一次增量产生的时间戳目录  
            // 勤于增备，每小时一次为佳  


![全备份的checkpoints]({{ site.url }}/static/img/3.png)

        可以看到 :  
            增量1.from_lsn = 全备 .to_lsn  
            增量2.from_lsn = 增量1.to_lsn  
            // 说明上述操作正常  
        
        
        然后，我们模拟一次事故( 进行了一些误操作，导致数据被破坏 )  
        
        
        < 接下来，我们进行恢复 >  
        
        首先，要清空mysql的数据目录( 保险起见，在清空前，把该目录压缩拷贝到安全的地方 )  
            // 注意，清空数据目录，意味着mysql自己的系统库(保存着用户信息和表信息之类)也会被删掉  
            // 注意，所以，我前面推荐不要单单备份某个库，而是全部一起备份  
        
        然后 :  
        innobackupex --apply-log --use-memory=64M --redo-only base_dir  
        innobackupex --apply-log --use-memory=64M --redo-only base_dir --incremental-dir=incre_dir_1  
        innobackupex --apply-log --use-memory=64M --redo-only base_dir --incremental-dir=incre_dir_2  
        ...  
        ...  
        innobackupex --apply-log --use-memory=64M --redo-only base_dir --incremental-dir=incre_dir_N-1  
        innobackupex --apply-log --use-memory=64M base_dir --incremental-dir=incre_dir_N  
            // base_dir 是之前全备份时产出的那个目录  
            // 假设你在那次全备后，做了N次增备，它们各自的产出目录分别是 incre_dir_1 ~ incre_dir_N  
            // 注意，最后对 incre_dir_N 操作的时候，是没有 --redo-only 这个参数的  
        
        然后 :  
        innobackupex --apply-log --use-memory=64M base_dir  
        innobackupex --copy-back base_dir  
        
        最后 :  
        chown -R mysql:mysql /var/lib/mysql/*  
            // 把mysql数据目录下的所有文件，全部改成mysql:mysql所属  
