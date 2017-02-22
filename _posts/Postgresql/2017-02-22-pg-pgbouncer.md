---
layout: post
title:  "记一次配置 pgbouncer 的几个踩坑点"
date:   2017-02-22 14:30:00
categories: Postgresql
comments: true
---


用 pgbouncer 来作为 pg的连接池，我配置时的主要疑惑点和踩坑点 :

        1. pg_hba.conf   的配合
        2. pgbouncer.ini 中的几个注意点
        3. userlist.txt  的配合


### 一.　简单描述我的环境 ###

        数据库们 : mydb, deepnote
        
        数据库的超级用户 : postgres
        数据库的普通用户 : seven (它可以读写这两个库中的某些表)
        
        启动pg和pgbouncer的linux用户 : postgres


### 二.　pg_hba.conf   的配合 ###

        # TYPE  DATABASE        USER            ADDRESS                 METHOD
        host    all             all             127.0.0.1/32            md5
        
        // 注意，认证方法必须是 md5，因为要与 pgbouncer.ini中呼应
        // 否则的话，在登陆pgbouncer的时候会一直报错密码错误


### 三.　pgbouncer.ini 中的几个注意点 ###

        [databases]
        db1 = host=127.0.0.1 port=5432 dbname=deepnote
        db2 = host=127.0.0.1 port=5432 dbname=mydb
        // 注意 :
        // 1. 你一共有多少个需要在应用层使用的库，就要配置多少行
        // 2. host, port, dbname 必须和实际情况一致
        // 3. 第一行开头的db1，是你为 deepnote 这个库起的别名，意味着 pgbouncer.db1 映射到 pg.deepnote
        //    第二行开头的db2，是你为 mydb     这个库起的别名，意味着 pgbouncer.db2 映射到 pg.mydb
        // 4. 在这个部分，建议不要配置 user :
        //    因为如果你写成 db2 = host=127.0.0.1 port=5432 dbname=mydb user=xxx password=yyy
        //    则，在使用pgbouncer连接到该库的时候，只能使用xxx这一个用户，也即该库只享有一个连接池
        //    然而我们实际往往是某个库有很多个用户，分别拥有不同的读写权限，处理着不同的业务
        
        
        [pgbouncer]
        listen_addr = 127.0.0.1
        listen_port = 6432
        
        logfile = /usr/local/pgsql/pgbouncer/pgbouncer.log
        pidfile = /usr/local/pgsql/pgbouncer/pgbouncer.pid
        
        unix_socket_dir = /var/run/pgbouncer
        unix_socket_mode = 0777
        unix_socket_group =
        // 注意 :
        // 在 /var/run 这个目录下，所有的东西，在每次重启机器的时候都会清空
        // 所以，每次启动pgbouncer之前，都去建立 unix_socket_dir，且设为只有 postgres 能访问(即权限700)
        
        auth_type = md5
        auth_file = /home/postgres/pgbouncer-install/conf/userlist.txt
        // 注意 :
        // 当 auth_type=md5 的时候，你的 pg_hba.conf 中也必须设置为md5认证方式
        // 否则的话，在登陆pgbouncer的时候会一直报错密码错误
        
        admin_users = postgres
        // 注意 :
        // 它指的是哪些用户可以管理pgbouncer，你可能会疑惑，它到底是某些linux用户呢，还是某些数据库用户呢
        // 保险起见，不管它应该填linux用户，还是数据库用户
        // 你只需创建一个 数据库超级用户 (且与启动pg和pgbouncer的 linux用户 同名)，就可以无忧了
        
        pool_mode = transaction
        
        max_client_conn = 100
        default_pool_size = 20
        min_pool_size = 0
        reserve_pool_size = 5
        reserve_pool_timeout = 3


### 四.　userlist.txt  的配合 ###

        1. 先以超级用户的身份登陆pg的任意一个库，执行下述命令 :
        
            \o userlist.txt
            select rolname,rolpassword from pg_authid where rolname = 'seven' or rolname = 'postgres';
            \o
            \q
            
            // 查询条件必须是 所有 需要在应用层中使用的数据库用户
            
            
        2. 生成的 userlist.txt
        
             rolname  |             rolpassword             
            ----------+-------------------------------------
             seven    | md5528a5677b92d65f47a4b1713d5b7ead3
             postgres | md567879684dac8ab24662f675b312e1635
            (2 rows)
        
        
        3. 改成下列样子
        
            "seven" "填该用户的原密码"
            "seven" "md5528a5677b92d65f47a4b1713d5b7ead3"
            "postgres" "填该用户的原密码"
            "postgres" "md567879684dac8ab24662f675b312e1635"


### 五.　如何连接 pgbouncer ###

        1. 如何登入pgbouncer进行管理
        
            psql -h 127.0.0.1 -p 6432 -U postgres pgbouncer
            // 注意 :
            // 1. 端口是pgbouncer的端口
            // 2. 用户 postgres 是 pgbouncer.ini 中的 admin_users
            //    虽然我上面建议 admin_users 是启动pg和pgbouncer的linux用户
            //    但是注意密码是要输入对应的数据库用户的密码
            // 3. 没有 -d，直接填数据库的名字，这一定要与直接连接pg时的"-d 库名" 区别开来
            //    此处库名是 pgbouncer，对了你一定发现了，它既不是 pgbouncer.ini 中的 db1，也不是 db2
            //    为何凭空冒出这个东西呢 ?
            //    因为我不是要通过pgbouncer来连接pg，这里，我只是要登入管理pgbouncer，所以库名填它
            
            show config;
            
            show clients;


        2. 如何通过pgbouncer间接连接pg
        
            psql -h 127.0.0.1 -p 6432 -U seven db1
            // 注意 :
            // 1. 端口仍是pgbouncer的端口
            // 2. 用户是 userlist.txt 中的一个
            // 3. 没有 -d，直接填数据库的别名(注意是别名)，即 pgbouncer.ini 中的 [databases]中对应那行的开头那个单词
