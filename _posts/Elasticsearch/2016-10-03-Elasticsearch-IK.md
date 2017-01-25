---
layout: post
title:  "Elasticsearch 中文分词 & 安全配置"
date:   2016-10-03 15:00:00
categories: Elasticsearch
comments: true
---


### 一. 资源准备 ###

        centos7.0  
    
            注意是7.0的系统  
    
        elasticsearch-2.2.0.tar.gz  
    
            资源地址 : https://www.elastic.co/downloads/past-releases/elasticsearch-2-2-0  
            注意是 2.2.0 的版本  
    
        elasticsearch-analysis-ik-1.8.0.zip  
    
            资源地址 : https://github.com/medcl/elasticsearch-analysis-ik/releases  
            注意是 1.8.0 的版本  
        
        
        
        tar -zxvf elasticsearch-2.2.0.tar.gz  
        
            解压 elasticsearch  
        
        mv elasticsearch-2.2.0 elasticsearch  
        
            改名 elasticsearch 的安装目录(强迫症...)  
        
        mv elasticsearch-2.2.0.tar.gz elasticsearch/  
        
            把压缩包移至elasticsearch的安装目录(强迫症...)  
        
        
        
        tar -zxvf elasticsearch-analysis-ik-1.8.0.zip  
        
            解压 elasticsearch-ik  
        
        mv elasticsearch-analysis-ik-1.8.0 elastic-ik  
        
            改名 elasticsearch-ik 的安装目录(强迫症...)  
        
        mv elasticsearch-analysis-ik-1.8.0.zip elastic-ik/  
        
            把压缩包移至elastic-ik的安装目录(强迫症...)  
    


### 二. 安装 ik分词器 ###

        yum install maven  
        
            安装 maven  
        
        cd path/elastic-ik  
        
            进入你的elastic-ik的安装目录  
        
        mvn clean  
        
        mvn compile  
        
        mvn package  
        
        mkdir /home/seven/elasticsearch/plugins/ik  
        
            创建ik分词插件的目录  
        
        cp target/releases/elasticsearch-analysis-ik-1.8.0.zip /home/seven/elasticsearch/plugins/ik/  
        
            把target/releases下的压缩文件拷贝到elasticsearch的ik插件目录下  
        
        cd /home/seven/elasticsearch/plugins/ik  
        
            进入elasticsearch的ik插件目录  
        
        unzip elasticsearch-analysis-ik-1.8.0.zip  
        
            解压这个ik压缩文件  



### 三. 配置文件 elasticsearch.yml ###

        # 此配置文件位于elasticsearch安装目录下的config目录中  
        # 我只把我显示指定的配置项列了出来 :  
        
        cluster.name: waringstates  
        
            集群的名称  
            若是同一个网段下几个机子要构成集群，该配置项必须相同  
        
        node.name: node-1  
        
            该结点的名称  
            若是同一个网段下几个机子要构成集群，该配置项必须不同  
        
        path.data: /mydata/elasticsearch/data  
        
            数据存储路径  
        
        path.logs: /mydata/elasticsearch/logs  
        
            日志存储路径  
        
        network.host: ******  
        
            绑定的ip地址  
            该值可以填 : 127.0.0.1 or 内网地址 or 公网地址  
        
        http.port: 9200  
        
            绑定的端口  
        
        path.repo: ["/mydata/elasticsearch/backup"]  
        
            数据备份的路径  



### 四. 检验 elasticsearch ###

        a. 创建一个测试用的索引  
        
            curl -XPUT 127.0.0.1:9200/index  
        
        b. 创建一个mapping  
        
            curl -XPOST 127.0.0.1:9200/index/fulltext/_mapping -d'  
            {  
                "fulltext" : {  
                    "_all" : {  
                        "analyzer"        : "ik_max_word",  
                        "search_analyzer" : "ik_max_word",  
                        "term_vector"     : "no",  
                        "store"           : "false"  
                    },  
                    "properties" : {  
                        "content" : {  
                            "type"            : "text",  
                            "analyzer"        : "ik_max_word",  
                            "search_analyzer" : "ik_max_word",  
                            "include_in_all"  : "true",  
                            "boost"           : 8  
                        }  
                    }  
                }  
            }'  
        
        c. 创建一些文档  
        
            curl -XPOST 127.0.0.1:9200/index/fulltext/1 -d'{"content":"美国留给伊拉克的是个烂摊子吗"}'  
            curl -XPOST 127.0.0.1:9200/index/fulltext/2 -d'{"content":"公安部：各地校车将享最高路权"}'  
            curl -XPOST 127.0.0.1:9200/index/fulltext/3 -d'{"content":"中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"}'  
            curl -XPOST 127.0.0.1:9200/index/fulltext/4 -d'{"content":"中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"}'  
        
        d. 查询测试  
        
            curl -XPOST 127.0.0.1:9200/index/fulltext/_search -d '{"query" : { "match" : { "content" : "中国" }}}'  



### 五. 限制 IP ###

        # 因为elasticsearch目前还没有用户权限的概念，因此必须做好安全防护  
        # 当你需要在公网上访问elasticsearch的时候，建议要限制可以访问的IP  
        # 通常，我们使用 iptables 来限制ip  
        
        systemctl stop firewalld  
        systemctl mask firewalld  
        
            centos7 默认使用的是firewall，因此需要先关闭它  
        
        yum install iptables-services  
        
            安装iptables service  
        
        systemctl enable iptables  
        
            开机启动iptables-service  
        
        systemctl start iptables  
        
            启动iptables  
        
        
        iptables -I INPUT -p tcp --dport 22 -j ACCEPT  
        
            开启ssh的22端口  
            !! 注意，别忘了把你之前的一些对外的端口信息配置一下，否则会导致无法连接  
            !! 注意，尤其注意先把ssh配好，否则呵呵哒...  
        
        iptables -I INPUT -s want_ip -p tcp --dport 9200 -j ACCEPT  
        
            开启9200端口(elasticsearch)，且只对want_ip这个ip开放  
        
        service iptables save  
        
            保存iptables的配置  
        
        service iptables restart  
        
            重启iptables  
        
        
        curl -XGET 公网ip:9200/?pretty  
        
            测试访问elasticsearch  
            当你在 want_ip 这个机子上执行这条命令，是有返回值的  
            当你在 其他机子上执行这条命令，是连不上的  
