---
layout: post
title:  "nginx代理多个flask"
date:   2016-09-08 19:30:00
categories: Linux
comments: true
---


本文讲述如何用nginx代理多个flask，从而进行负载均衡  

所需准备 :  

    1. nginx  
    
            它安装依赖的东西还不少，可以参考 http://www.cnblogs.com/skynet/p/4146083.html  
    
    2. flask  
    
            sudo pip install Flask  
    
    3. uwsgi  
    
            sudo pip install uwsgi  
        
            使用uwsgi的原因是，如果光溜溜的flask是很容易崩溃的，  
            外面套一层uwsgi，实践检验能让flask强壮很多  
        
            若无法安装uwsgi这个库，则大多数原因是因为少了两个东西 :  
        
            ubuntu系列 :  
                sudo apt-get install python-dev  
                sudo apt-get install setuptools  
        
            centos系列 :  
                sudo yum install python-devel  
                sudo yum install setuptools  


下面举个例子，讲讲具体的配置  


### 一.　flask程序的信息 ###

    1. 我的用flask库写的web服务端程序的根目录 :  
    
            /home/seven/program/openmind_server/  
    
    2. 在这个目录下，我的程序的入口文件是 main.py  
    
            即入口文件的绝对地址是 /home/seven/program/openmind_server/main.py  
            
            且为了使用方便，最好让这个脚本接受两个参数，分别作为这个进程绑定的地址和端口  
            例如 :  
            python main.py 127.0.0.1 8081  
            python main.py 127.0.0.1 8082  
    
    3. 在这个脚本中，我的app的名称是 Server_App  
    
            即 main.py 中，有类似这样的语句 :  
            
            Server_App = flask.Flask(__name__)  
            Server_App.secret_key = '\r\x9d1\xd1\xccW\x9e\xa6\x9a\x97[\xb1=\x93\x87\x15s<\xe8\xe3\x13DL?'  
            
            # 注意，若你的flask程序在不同的机子上（一般生产环境下都是这样，是真正的多机负载均衡）  
            # 则，这个 secret_key 要保持一致，否则session可能无法正常工作  


### 二.　uwsgi配置文件  uwsgi8081.ini ###

    [uwsgi]  
    
    socket = 127.0.0.1:8081  
        绑定监听的地址和端口  
            
        实际生产环境下 :  
            1. 建议nginx单独一台服务器，然后其他的flask都在同一网段的其他机子上  
            2. uwsgi要在 nginx结点，及每台flask结点上 都要安装  
            3. flask只要在每台flask结点上安装  
            4. 这个uwsgi配置文件和对应的flask程序是在同一台机子上的  
        
    
    master = true  
    
    pidfile = /mydata/openmind_server/pids/uwsgi8081.pid  
        这个uwsgi进程的pid文件的路径（建议使用绝对路径）  
    
    chdir = /home/seven/program/openmind_server/  
        你的用flask库编写的服务端程序的根目录的路径  
        （建议使用绝对路径，且最后有'/'）  
        （若flask程序位于不同机子上，则要保证各机子上的路径是一致的）  
    
    wsgi-file = main.py  
        在上述目录下，你的入口脚本的名字  
    
    callable = Server_App  
        在上述入口脚本中，你的flask的app对象的名字  
    
    processes = 2  
        该值建议与CPU核数相同  
    
    threads = 4  
        占用的线程数  
    
    stats = 127.0.0.1:9091  
        查询状态信息的端口  
    
    logdate=true  
    
    daemonize=/mydata/openmind_server/logs/flask8081.log  
        以后台方式运行，且指定日志文件的路径（建议使用绝对路径）  


    （ uwsgi8082.ini， uwsgi8083.ini， uwsgi8084.ini 与之类似，改端口和路径即可）  



### 三.　nginx配置文件 /usr/local/nginx/conf/nginx.conf ###

    worker_processes  2;  
        # 建议与CPU核数相同  
    
    error_log  /mydata/nginx/log/error.log;  
        # 错误日志的路径（建议使用绝对路径）  
        # 且除了它，在 /usr/local/nginx/logs/error.log 也有一部分的错误日志  
    
    pid        /mydata/nginx/pid/nginx.pid;  
        # 进程号文件的路径（建议使用绝对路径）  
    
    events {  
        use epoll;  
            # epoll效率比轮询要高  
        multi_accept on;  
        worker_connections  1024;  
            # 最大连接数  
    }  
    
    
    http {  
        include       mime.types;  
        default_type  application/octet-stream;  
    
        server_names_hash_bucket_size 128;  
        client_header_buffer_size 32k;  
        client_body_buffer_size 512k;  
        client_max_body_size 32m;  
        large_client_header_buffers 4 32k;  
    
        access_log  /mydata/nginx/log/access.log;  
            # 访问日志路径（建议使用绝对路径）  
    
        sendfile        on;  
        tcp_nodelay on;  
        server_tokens off;  
        access_log off;  
        charset UTF-8;  
        keepalive_timeout  60;  
    
        open_file_cache max=1024 inactive=20s;  
        open_file_cache_valid 60s;  
        open_file_cache_min_uses 2;  
    
        upstream my_servers {  
            server 127.0.0.1:8081;  
            server 127.0.0.1:8082;  
            server 127.0.0.1:8083;  
            server 127.0.0.1:8084;  
        }  
        # 转发配置 :  
        # 一共可以转发到本机的8081,8082,8083,8084四个结点上  
        # 实际生产环境中建议flask结点在其他机子上  
    
        server {  
            listen       80;  
                # 对外暴露80端口  
            server_name  localhost default backlog=256;  
            access_log  /mydata/nginx/log/host.access.log;  
    
            location / {  
                uwsgi_pass my_servers;  
                include uwsgi_params;  
                uwsgi_param UWSGI_CHDIR /home/seven/program/openmind_server/;  
                uwsgi_param UWSGI_SCRIPT main;  
            }  
            # 转发配置 :  
            # 通过使用上面定义的 upstream my_servers 来进行负载均衡  
            # 且指定flask程序的根目录  
            # 且指定flask程序根目录下的入口脚本的名称（不包含后缀名）  
    
            # redirect server error pages to the static page /50x.html  
            error_page   500 502 503 504  /50x.html;  
            location = /50x.html {  
                root   html;  
            }  
        }  
    }  



### 四.　启动方式 ###

    uwsgi --ini path/uwsgi8081.ini  
        启动8081结点（以非root用户的身份来启动）  
    
    uwsgi --ini path/uwsgi8082.ini  
        启动8082结点（以非root用户的身份来启动）  
    
    uwsgi --ini path/uwsgi8083.ini  
        启动8083结点（以非root用户的身份来启动）  
    
    uwsgi --ini path/uwsgi8084.ini  
        启动8084结点（以非root用户的身份来启动）  
    
    cd /usr/local/nginx/sbin  
        进入nginx的脚本目录  
    
    ./nginx  
        启动nginx（注意要以root用户身份来启动）  


### 五.　重启方式 ###

    uwsgi --reload path/uwsgi8081.pid  
        重启8081结点（指定对应的pid文件，以非root身份）  
    
    uwsgi --reload path/uwsgi8082.pid  
        重启8082结点（指定对应的pid文件，以非root身份）  
    
    uwsgi --reload path/uwsgi8083.pid  
        重启8083结点（指定对应的pid文件，以非root身份）  
    
    uwsgi --reload path/uwsgi8084.pid  
        重启8084结点（指定对应的pid文件，以非root身份）  

    cd /usr/local/nginx/sbin  
        进入nginx的脚本目录  
    
    ./nginx -s stop  
        关闭nginx（注意要以root用户身份来关闭）  
    
    ./nginx  
        再次启动nginx（注意要以root用户身份来启动）  
