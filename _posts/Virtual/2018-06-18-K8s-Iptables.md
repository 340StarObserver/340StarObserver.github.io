---
layout: post
title:  "k8s集群中数据包的流向"
date:   2018-06-18 15:48:00
categories: Virtual
comments: true
---

团队已经把大多数服务迁移到k8s平台上，然而我只是会使用（比如Dockerfile、容器编排等），对深入一点的（比如实现原理、网络方案、故障处理等）不了解。这次决定从k8s容器网络入手进行学习。**本文以实际环境下的一次访问请求为例，分析数据包的完整链路，侧重于iptables在k8s集群中的应用。**

我画了一张数据包的流向图：

![k8s_net]({{ site.url }}/static/img/k8s_net.png)

如上图所示，在外部访问 web.tdsql.pr.fsphere.cn，它是某数据库的管理端页面。数据包先后经过几大步骤：
- 第一步：通过外部dns，将域名解析到前置的nginx上（它是公网接入机，因为k8s集群是位于VPC里的，不建议分配公网IP）。且凡是需要在外部访问的域名，都需要配置外部dns。
- 第二步：前置nginx将匹配 *.pr.fsphere.cn 的请求，以upstream的方式，分摊到每个k8s-node上。
- 第三步：iptables截获数据包，以dnat的方式，分摊给ingress的每个Pod。
- 第四步：ingress按照请求的域名和路由进行规则匹配，转发至k8s集群的一个内部域名。
- 第五步：kube-dns将内部域名解析到一个ClusterIP上（web.tdsql这个Service的vip）。
- 第六步：在Service上，以kube-proxy的方式，分摊至各RS上。

下面重点论述第三步～第六步的具体实现。

<br>

### 环境信息 ###

```text
集群节点：
k8s-node1 : 10.10.2.199
k8s-node2 : 10.10.2.200
k8s-node3 : 10.10.2.198

// 获取方式：kubectl describe nodes | grep InternalIP
```

```text
namespace = ingress 下的 nginx 服务：
type : NodePort
vip  : 192.168.192.30:80
rs   : 192.168.245.59:80, 192.168.231.18:80, 192.168.235.61:80

// 获取方式：kubectl describe svc nginx -n ingress
// nginx.ingress是一个裁剪版的nginx容器，所有需要在外部访问的http服务，均可通过它统一接入。只要保证各服务之间，baseDomain一致，fullDomain或者后缀路由不同就行。
```

```text
namespace = tdsql 下的 web 服务：
type : ClusterIP
vip  : 192.168.192.64:80
rs   : 192.168.231.58:80, 192.168.245.34:80, 192.168.245.57:80

// 获取方式：kubectl describe svc web -n tdsql
// web.tdsql是内部一个数据库的管理端页面的服务。
```

<br>

### 第三步的实现 ###

先来看看iptables截获数据包之前，数据包原本应该发给哪个进程？

```shell
[root@node1 ~]# netstat -ltnp | grep :80
tcp        0      0 10.10.2.199:8010        0.0.0.0:*               LISTEN      73329/ssh           
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      19850/hyperkube     
tcp        0      0 10.10.2.199:8080        0.0.0.0:*               LISTEN      25427/ssh           
tcp        0      0 127.0.0.1:8082          0.0.0.0:*               LISTEN      73287/./dcos_oob_de 
tcp        0      0 127.0.0.1:8083          0.0.0.0:*               LISTEN      73910/./dcos_oob_de 
tcp        0      0 10.10.2.199:8019        0.0.0.0:*               LISTEN      91587/java          
tcp        0      0 127.0.0.1:8085          0.0.0.0:*               LISTEN      73103/dcos_cmdb     
tcp        0      0 127.0.0.1:8087          0.0.0.0:*               LISTEN      75529/./dcos_upgrad 
tcp        0      0 127.0.0.1:8090          0.0.0.0:*               LISTEN      75693/./dcos_alarm_ 
tcp        0      0 127.0.0.1:8091          0.0.0.0:*               LISTEN      75708/./dcos_alarm_ 
tcp6       0      0 :::80                   :::*                    LISTEN      53584/hyperkube     
tcp6       0      0 :::8081                 :::*                    LISTEN      64176/java
```

如上所示，k8s-node上的80端口，被hyperkube进程监听。

**这个hyperkube进程哪来的？**　这是因为nginx.ingress这个Service，被定义为nodePort类型的。k8s会对每个nodePort类型的Service，在每个k8s-node上，都监听对应的端口。这样一来，访问任意一个k8s-node的 nodeIP:nodePort 都行。

**这个hyperkube进程什么用？**　因为不管是Pod的ip，还是Service的vip，都是容器网络下的地址。而k8s集群外面的前置nginx无法直接访问k8s的容器网络，只能访问物理网络，这个hyperkube进程充当了容器网络和物理网络的桥梁，提供在物理网络下能够让前置nginx访问的方式。

<br>

**下面来看iptables是如何截获母机80端口的数据包，并dnat到nginx.ingress的各Pod上的？**

```shell
[root@node1 ~]# iptables -S -t nat | grep 'dport 80'
-A KUBE-NODEPORTS -p tcp -m comment --comment "ingress/nginx:http" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "ingress/nginx:http" -m tcp --dport 80 -j KUBE-SVC-ILTDUQYRTBDG2DYW
...
...
```

如上所示，母机80端口的数据包，首先匹配到上述规则，采取的操作是 KUBE-MARK-MASQ 和 KUBE-SVC-ILTDUQYRTBDG2DYW。我们知道，iptables -j 后面跟的可以是 ACCEPT、DROP、REJECT。这里的两个KUBE开头的字符串，明显是扩展的。

<br>

**按照顺序，我们先来看下 KUBE-MARK-MASQ 这条链的作用？**

```shell
[root@node1 ~]# iptables -S -t nat | grep KUBE-MARK-MASQ
-N KUBE-MARK-MASQ
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
...
...
```

如上所示， KUBE-MARK-MASQ 是用来给数据包打上标记的（标记值为 0x4000/0x4000），那么这个标记又有啥用呢？此时，我们按照这个标记值进行grep一下：

```shell
[root@node1 ~]# iptables -S -t nat | grep 0x4000/0x4000
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

看到了吧，凡是带有 0x4000/0x4000 标记的数据包，都会被应用 MASQUERADE 规则（与snat类似，都是用来修改源地址的。区别是snat必须要指定ip，而 MAXQUERADE则可以动态地根据网卡地址来执行）。而且你看 comment 也提到了，这条规则是用来在 "k8s service traffic" 的时候实现 SNAT的。

**所以，KUBE-MARK-MASQ 链的作用是把数据包的源地址改写为当前母机的本机地址。**这样可以避免当数据包之后若做dnat，也不会出现三角流量。

<br>

**既然 KUBE-MARK-MASQ 并不是用来将数据包dnat至各Pod的。那么就只能是剩下的 KUBE-SVC-ILTDUQYRTBDG2DYW 这条链，也就是说，数据包到了这条链上。**我们接下来看看这条链的详细规则：

<br>

```shell
[root@node1 ~]# iptables -S -t nat | grep KUBE-SVC-ILTDUQYRTBDG2DYW
-N KUBE-SVC-ILTDUQYRTBDG2DYW
-A KUBE-NODEPORTS -p tcp -m comment --comment "ingress/nginx:http" -m tcp --dport 80 -j KUBE-SVC-ILTDUQYRTBDG2DYW
-A KUBE-SERVICES -d 192.168.192.30/32 -p tcp -m comment --comment "ingress/nginx:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-ILTDUQYRTBDG2DYW
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-DMJPT5NIA7G72QSV --mask 255.255.255.255 --rsource -j KUBE-SEP-DMJPT5NIA7G72QSV
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-PQFXUY3XRL2SFBRY --mask 255.255.255.255 --rsource -j KUBE-SEP-PQFXUY3XRL2SFBRY
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-BAVC27VMIGG6TXQV --mask 255.255.255.255 --rsource -j KUBE-SEP-BAVC27VMIGG6TXQV
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-DMJPT5NIA7G72QSV
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-PQFXUY3XRL2SFBRY
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -j KUBE-SEP-BAVC27VMIGG6TXQV
```

如上所示，**到达 KUBE-SVC-ILTDUQYRTBDG2DYW 这条链的数据包，会按照百分比转发到三条链上。**33%可能性会被转到 KUBE-SEP-DMJPT5NIA7G72QSV 链，50%可能性会被转到 KUBE-SEP-PQFXUY3XRL2SFBRY 链，17%可能性背会转到 KUBE-SEP-BAVC27VMIGG6TXQV 链。

咦，这儿按照百分比进行转发，已经和 "dnat到各Pod上" 有点像了呀。接着看这三条链的细节。

<br>

```shell
[root@node1 ~]# iptables -S -t nat | grep KUBE-SEP-DMJPT5NIA7G72QSV
-N KUBE-SEP-DMJPT5NIA7G72QSV
-A KUBE-SEP-DMJPT5NIA7G72QSV -s 192.168.231.18/32 -m comment --comment "ingress/nginx:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-DMJPT5NIA7G72QSV -p tcp -m comment --comment "ingress/nginx:http" -m recent --set --name KUBE-SEP-DMJPT5NIA7G72QSV --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 192.168.231.18:80
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-DMJPT5NIA7G72QSV --mask 255.255.255.255 --rsource -j KUBE-SEP-DMJPT5NIA7G72QSV
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-DMJPT5NIA7G72QSV
```

如上所示，到达 KUBE-SEP-DMJPT5NIA7G72QSV 链的数据包，被dnat至 192.168.231.18:80，发现了吧，它就是 nginx.ingress 的其中一个Pod的地址～。

```shell
[root@node1 ~]# iptables -S -t nat | grep KUBE-SEP-PQFXUY3XRL2SFBRY
-N KUBE-SEP-PQFXUY3XRL2SFBRY
-A KUBE-SEP-PQFXUY3XRL2SFBRY -s 192.168.235.61/32 -m comment --comment "ingress/nginx:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-PQFXUY3XRL2SFBRY -p tcp -m comment --comment "ingress/nginx:http" -m recent --set --name KUBE-SEP-PQFXUY3XRL2SFBRY --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 192.168.235.61:80
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-PQFXUY3XRL2SFBRY --mask 255.255.255.255 --rsource -j KUBE-SEP-PQFXUY3XRL2SFBRY
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-PQFXUY3XRL2SFBRY
```

如上所示，到达 KUBE-SEP-PQFXUY3XRL2SFBRY 链的数据包，被dnat至 192.168.235.61:80，它是 nginx.ingress 的其中另外一个Pod的地址。

```shell
[root@node1 ~]# iptables -S -t nat | grep KUBE-SEP-BAVC27VMIGG6TXQV
-N KUBE-SEP-BAVC27VMIGG6TXQV
-A KUBE-SEP-BAVC27VMIGG6TXQV -s 192.168.245.59/32 -m comment --comment "ingress/nginx:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-BAVC27VMIGG6TXQV -p tcp -m comment --comment "ingress/nginx:http" -m recent --set --name KUBE-SEP-BAVC27VMIGG6TXQV --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 192.168.245.59:80
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-BAVC27VMIGG6TXQV --mask 255.255.255.255 --rsource -j KUBE-SEP-BAVC27VMIGG6TXQV
-A KUBE-SVC-ILTDUQYRTBDG2DYW -m comment --comment "ingress/nginx:http" -j KUBE-SEP-BAVC27VMIGG6TXQV
```

如上所示，到达 KUBE-SEP-BAVC27VMIGG6TXQV 链的数据包，被dnat至 192.168.245.59:80，它是 nginx.ingress 的其中另外一个Pod的地址。

**如此，数据包就被分发到nginx.ingress的三个Pod上了。**

<br>

到此，你可能会有个疑问，**分摊到三个Pod的时候，为何不是均匀分摊**，即百分比为啥不同？
- 启动第一个Pod的时候，Pod1的百分比是100%。
- 启动第二个Pod的时候，平均百分比是50%，从当前百分比超过50%的Pod中，挑选一个，划出50%给新的Pod。这样，Pod1的百分比是50%，Pod2的百分比也是50%。
- 启动第三个Pod的时候，平均百分比是33%，同理，从当前百分比超过33%的Pod中，挑选一个，划出33%给新的Pod。这样，原来的两个Pod中，有一个仍然是50%，有一个只剩下17%，而新的那个是33%。

<br>

### 第四步的实现 ###

前面也提到，nginx.ingress 其实是个nginx容器，它基于一个裁剪版的nginx镜像。所以，能够在k8s的ConfigMap中，找到该容器的配置文件：

```shell
[root@node1 ~]# cat /data/kubespray-yaml/ingress/config/ingress.cm.nginx.confd/ingress.cm.web.tdsql.nginx.conf 
server {
    listen 80;
    server_name web.tdsql.pr.fsphere.cn;
    error_log /var/log/nginx/error.log;
    access_log /var/log/nginx/web.tdsql-access.log;

    charset utf-8;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    client_max_body_size 512m;
    client_body_buffer_size 256k;
    proxy_connect_timeout 30;
    proxy_send_timeout 30;
    proxy_read_timeout 60;
    proxy_buffer_size 256k;
    proxy_buffers 4 256k;
    proxy_busy_buffers_size 256k;
    proxy_temp_file_write_size 256k;
    proxy_next_upstream error timeout invalid_header http_500 http_503 http_404;
    proxy_max_temp_file_size 128m;

    location / {
        proxy_set_header X-Real-IP       $remote_addr;
        proxy_set_header Host            $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        add_header Access-Control-Allow-Credentials true;
        proxy_pass http://web.tdsql;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root html;
    }
}
```

如上所示，请求的域名 web.tdsql.pr.fsphere.cn 匹配到该配置文件中的规则，请求被进一步，以proxy_pass的方式，转发到 http://web.tdsql 上。

<br>

### 第五步的实现 ###

k8s集群内部，如何将 web.tdsql 这个域名，解析到某个地址上？

```shell
[root@node1 ~]# nslookup web.tdsql
Server:     192.168.192.3
Address:    192.168.192.3#53

Name:   web.tdsql.svc.cluster.mgmt
Address: 192.168.192.64
```

如上所示，该域名对应的地址是 192.168.192.64，根据 <环境信息> 小节中来看，它就是 web.tdsql 这个Service的ClusterIP，习惯上也叫做vip。这样一来，有了IP地址，数据包便能真正到达 web.tdsql 的Service上。

而 192.168.192.3 这个dns服务器，又是哪里来的？

```shell
[root@node1 ~]# kubectl describe svc kube-dns -n kube-system
2018-06-18 15:17:48.802440 I | proto: duplicate proto type registered: google.protobuf.Any
2018-06-18 15:17:48.803242 I | proto: duplicate proto type registered: google.protobuf.Duration
2018-06-18 15:17:48.803295 I | proto: duplicate proto type registered: google.protobuf.Timestamp
Name:           kube-dns
Namespace:      kube-system
Labels:         addonmanager.kubernetes.io/mode=Reconcile
            k8s-app=kube-dns
            kubernetes.io/cluster-service=true
            kubernetes.io/name=KubeDNS
Annotations:        <none>
Selector:       k8s-app=kube-dns
Type:           ClusterIP
IP:         192.168.192.3
Port:           dns 53/UDP
Endpoints:      192.168.231.34:53,192.168.231.38:53,192.168.235.8:53
Port:           dns-tcp 53/TCP
Endpoints:      192.168.231.34:53,192.168.231.38:53,192.168.235.8:53
Session Affinity:   None
Events:         <none>
```

如上所示，192.168.192.3 其实是 kube-system 这个名字空间下的 kube-dns 这个Service的vip。kube-dns是k8s自带的，用以实现服务发现。

<br>

### 第六步的实现 ###

现在，数据包到达 web.tdsql 这个 Service了，最后如何以 kube-proxy 的方式，分摊到 web.tdsql 的各Pod呢？

首先，要查出与 访问该Service的vip:vport 匹配的规则，也就是 192.168.192.64:80。

```shell
[root@node1 ~]# iptables -S -t nat | grep 192.168.192.64 | grep 80
-A KUBE-SERVICES ! -s 192.168.224.0/19 -d 192.168.192.64/32 -p tcp -m comment --comment "tdsql/web: cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 192.168.192.64/32 -p tcp -m comment --comment "tdsql/web: cluster IP" -m tcp --dport 80 -j KUBE-SVC-BJMJPDVPPZKAQKK7
```

如上所示，**请求 192.168.192.64:80 的数据包，匹配到规则，将会到达 KUBE-SVC-BJMJPDVPPZKAQKK7 这条链。**

<br>

接下来，和前面 <第四步的实现> 小节中的，就是一个道理了。即看 KUBE-SVC-BJMJPDVPPZKAQKK7 这条链的具体规则：

```shell
[root@node1 ~]# iptables -S -t nat | grep KUBE-SVC-BJMJPDVPPZKAQKK7
-N KUBE-SVC-BJMJPDVPPZKAQKK7
-A KUBE-SERVICES -d 192.168.192.64/32 -p tcp -m comment --comment "tdsql/web: cluster IP" -m tcp --dport 80 -j KUBE-SVC-BJMJPDVPPZKAQKK7
-A KUBE-SVC-BJMJPDVPPZKAQKK7 -m comment --comment "tdsql/web:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-GVR3A2XHGAQS7LDR
-A KUBE-SVC-BJMJPDVPPZKAQKK7 -m comment --comment "tdsql/web:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-RY5WDFQPQ3P6KIT5
-A KUBE-SVC-BJMJPDVPPZKAQKK7 -m comment --comment "tdsql/web:" -j KUBE-SEP-ANYCT6LXRLLWZRMQ
```

如上所示，**到达 KUBE-SVC-BJMJPDVPPZKAQKK7 链的数据包，按照 33%、50%、17%的比例，分别转到 KUBE-SEP-GVR3A2XHGAQS7LDR、KUBE-SEP-RY5WDFQPQ3P6KIT5、KUBE-SEP-ANYCT6LXRLLWZRMQ 这三条链上。**

<br>

```shell
[root@node1 ~]# iptables -S -t nat | grep KUBE-SEP-GVR3A2XHGAQS7LDR
-N KUBE-SEP-GVR3A2XHGAQS7LDR
-A KUBE-SEP-GVR3A2XHGAQS7LDR -s 192.168.231.58/32 -m comment --comment "tdsql/web:" -j KUBE-MARK-MASQ
-A KUBE-SEP-GVR3A2XHGAQS7LDR -p tcp -m comment --comment "tdsql/web:" -m tcp -j DNAT --to-destination 192.168.231.58:80
-A KUBE-SVC-BJMJPDVPPZKAQKK7 -m comment --comment "tdsql/web:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-GVR3A2XHGAQS7LDR

[root@node1 ~]# iptables -S -t nat | grep KUBE-SEP-RY5WDFQPQ3P6KIT5
-N KUBE-SEP-RY5WDFQPQ3P6KIT5
-A KUBE-SEP-RY5WDFQPQ3P6KIT5 -s 192.168.245.34/32 -m comment --comment "tdsql/web:" -j KUBE-MARK-MASQ
-A KUBE-SEP-RY5WDFQPQ3P6KIT5 -p tcp -m comment --comment "tdsql/web:" -m tcp -j DNAT --to-destination 192.168.245.34:80
-A KUBE-SVC-BJMJPDVPPZKAQKK7 -m comment --comment "tdsql/web:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-RY5WDFQPQ3P6KIT5

[root@node1 ~]# iptables -S -t nat | grep KUBE-SEP-ANYCT6LXRLLWZRMQ
-N KUBE-SEP-ANYCT6LXRLLWZRMQ
-A KUBE-SEP-ANYCT6LXRLLWZRMQ -s 192.168.245.57/32 -m comment --comment "tdsql/web:" -j KUBE-MARK-MASQ
-A KUBE-SEP-ANYCT6LXRLLWZRMQ -p tcp -m comment --comment "tdsql/web:" -m tcp -j DNAT --to-destination 192.168.245.57:80
-A KUBE-SVC-BJMJPDVPPZKAQKK7 -m comment --comment "tdsql/web:" -j KUBE-SEP-ANYCT6LXRLLWZRMQ
```

如上所示，**到达这三条链的数据包，被分别dnat至 192.168.231.58:80、192.168.245.34:80、192.168.245.57:80 三个Pod上。**

如此，数据包终于到达了目标终点～。