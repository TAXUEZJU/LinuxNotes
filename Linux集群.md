### Linux 集群
一台服务器的硬件配置是有限的，当服务器上运行的资源超过服务器的承载能力时，将导致该服务器崩溃。在生产环境中，多数企业会使用多台服务器搭建成一个集群来运行应用程序，这样不仅可以避免单点故障，还能提升服务器的承载能力。
集群从功能实现上分为两种：高可用集群和负载均衡集群。高可用，顾名思义，当一台服务器宕机不能提供服务，有备份的服务器顶替，以实现对用户的不间断服务。负载均衡集群，是根据某种负载策略把请求分摊到不同服务器上，让整个服务器集群来处理请求

#### 搭建高可用集群
高可用集群，即“HA”集群，也常称作“双机热备”，用于关键业务。常见实现高可用的开源软件有keepalived，核心原理是通过心跳线连接两台服务器，正常情况下由一台服务器提供服务，当这台服务器宕机，备用服务器顶替
##### 1. keepalived工作原理
虚拟路由冗余协议VRRP（Virtual Router Redundancy Protocol）是实现路由高可用的一种通信协议，该协议将多台功能相同的路由组成一个小组，小组中有1个master和多个backup。工作时，master通过组播向各个backup发送VRRP协议的数据包，当backup收不到master发来的数据包时，就会认为master宕机了。此时根据各backup的优先级决定谁成为新的master。
keepalived就是采用VRRP协议实现的高可用方案，主要有三个模块：core，check，vrrp。core模块为keepalived的核心，负责主进程的启动、维护以及全局配置文件的加载和解析；check模块负责健康检查；vrrp模块用来实现VRRP协议
CentOS默认的yum源里有keepalived包，执行`yum install keepalived`即可安装
##### 2. keepalived + Nginx 实现 Web 高可用
生产环境中，Nginx很多时候作为负载均衡器使用，一旦宕机将会导致整个站点无法访问，所以有必要准备一台备用Nginx，keepalived在这种场景下非常合适，这里只采用两台，一台master，一台backup的形式做示范：
```
master: 192.168.126.130  keepalived + Nginx
backup: 192.168.126.131  keepalived + Nginx
VIP: 192.168.126.100
```
VIP即“Virtual IP”，这个IP是keepalived给服务器配置的，服务器靠VIP对外提供服务，当master机器宕机，VIP被分配到backup上，在用户看来是无感的
##### 3. Nginx配置
- 下载最新稳定版Nginx，解压
    ``` bash
    [root@hdp-node-01 apps]# tar -xzvf nginx-1.12.2.tar.gz
    ```
- 配置，编译，安装
    ``` bash
    [root@hdp-node-01 nginx-1.12.2]# ./configure --prefix=/usr/local/nginx
    ...
    [root@hdp-node-01 nginx-1.12.2]# make
    [root@hdp-node-01 nginx-1.12.2]# make install
    ```
    出现以下错误解决办法如下：
    ```
    ./configure: error: the HTTP rewrite module requires the PCRE library.

    yum -y install pcre-devel

    ./configure: error: the HTTP gzip module requires the zlib library.

    yum install -y zlib-devel
    ```
- 编写Nginx启动脚本，并加入系统服务
    ``` bash
    [root@hdp-node-01 nginx-1.12.2]# vim /etc/init.d/nginx
    ...
    #!/bin/bash
    # chkconfig: - 30 21
    # description: http service.
    # Source Function Library
    . /etc/init.d/functions
    # Nginx Settings

    NGINX_SBIN="/usr/local/nginx/sbin/nginx"
    NGINX_CONF="/usr/local/nginx/conf/nginx.conf"
    NGINX_PID="/usr/local/nginx/logs/nginx.pid"
    RETVAL=0
    prog="Nginx"

    start() {
            echo -n $"Starting $prog: "
            mkdir -p /dev/shm/nginx_temp
            daemon $NGINX_SBIN -c $NGINX_CONF
            RETVAL=$?
            echo
            return $RETVAL
    }

    stop() {
            echo -n $"Stopping $prog: "
            killproc -p $NGINX_PID $NGINX_SBIN -TERM
            rm -rf /dev/shm/nginx_temp
            RETVAL=$?
            echo
            return $RETVAL
    }

    reload(){
            echo -n $"Reloading $prog: "
            killproc -p $NGINX_PID $NGINX_SBIN -HUP
            RETVAL=$?
            echo
            return $RETVAL
    }

    restart(){
            stop
            start
    }

    configtest(){
        $NGINX_SBIN -c $NGINX_CONF -t
        return 0
    }

    case "$1" in
      start)
            start
            ;;
      stop)
            stop
            ;;
      reload)
            reload
            ;;
      restart)
            restart
            ;;
      configtest)
            configtest
            ;;
      *)
            echo $"Usage: $0 {start|stop|reload|restart|configtest}"
            RETVAL=1
    esac
    exit $RETVAL
    ```
    保存后更改权限
    ```bash
    [root@hdp-node-01 nginx-1.12.2]# chmod 755 /etc/init.d/nginx
    [root@hdp-node-01 nginx-1.12.2]# chkconfig --add nginx
    ```
    开机启动Nginx
    `[root@hdp-node-01 nginx-1.12.2]# chkconfig nginx on`
- 更改Nginx的配置文件
    ``` bash
    [root@hdp-node-01 nginx-1.12.2]# > /usr/local/nginx/conf/nginx.conf
    [root@hdp-node-01 nginx-1.12.2]# vim /usr/local/nginx/conf/nginx.conf
    ...
    user nobody nobody;
    worker_processes 2;
    error_log /usr/local/nginx/logs/nginx_error.log crit;
    pid /usr/local/nginx/logs/nginx.pid;
    worker_rlimit_nofile 51200;
    events
    {
        use epoll;
        worker_connections 6000;
    }
    http
    {
        include mime.types;
        default_type application/octet-stream;
        server_names_hash_bucket_size 3526;
        server_names_hash_max_size 4096;
        log_format combined_realip '$remote_addr $http_x_forwarded_for [$time_local]'
        '$host "$request_uri" $status'
        '"$http_referer" "$http_user_agent"';
        sendfile on;
        tcp_nopush on;
        keepalive_timeout 30;
        client_header_timeout 3m;
        client_body_timeout 3m;
        send_timeout 3m;
        connection_pool_size 256;
        client_header_buffer_size 1k;
        large_client_header_buffers 8 4k;
        request_pool_size 4k;
        output_buffers 4 32k;
        postpone_output 1460;
        client_max_body_size 10m;
        client_body_buffer_size 256k;
        client_body_temp_path /usr/local/nginx/client_body_temp;
        proxy_temp_path /usr/local/nginx/proxy_temp;
        fastcgi_temp_path /usr/local/nginx/fastcgi_temp;
        fastcgi_intercept_errors on;
        tcp_nodelay on;
        gzip on;
        gzip_min_length 1k;
        gzip_buffers 4 8k;
        gzip_comp_level 5;
        gzip_http_version 1.1;
        gzip_types text/plain application/x-javascript text/css text/htm application/xml;

        server
        {
            listen 80;
            server_name localhost;
            index index.html index.htm index.php;
            root /usr/local/nginx/html;

            location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass unix:/tmp/php-fcgi.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME /usr/local/nginx/html$fastcgi_script_name;
            }
        }
    }
    ```
    配置完后，检验是否有错误
    ``` bash
    [root@hdp-node-01 nginx-1.12.2]# /usr/local/nginx/sbin/nginx -t
    nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
    nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
    ```
- 启动Nginx
    ``` bash
    [root@hdp-node-01 nginx-1.12.2]# service nginx start
    Starting nginx (via systemctl):                            [  确定  ]
    ```
##### 4. keepalived配置
- 编辑master（130）的keepalived配置文件
    ```
    [root@hdp-node-01 ~]# > /etc/keepalived/keepalived.conf
    [root@hdp-node-01 ~]# vim /etc/keepalived/keepalived.conf
    ...
    global_defs {
       notification_email {
         root@example.com
       }
       notification_email_from root@example.com
       smtp_server 127.0.0.1
       smtp_connect_timeout 30
       router_id LVS_DEVEL
    }
    vrrp_script chk_nginx {
        script "/usr/local/sbin/check_ng.sh"    //自定义脚本，监控Nginx服务
        interval 3
    }
    vrrp_instance VI_1 {
        state MASTER    //角色为master
        interface ens33    //针对监听VIP的网卡
        virtual_router_id 51
        priority 100    //权重100，master比backup大
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass centos7    //自定义密码
        }
        virtual_ipaddress {
            192.168.126.100    //定义VIP
        }
        track_script {
            chk_nginx    //定义监控脚本，这里和上面vrrp_script后面的字符一致
        }
    }
    ```
- 监控Nginx服务的脚本
    ``` bash
    [root@hdp-node-01 ~]# vim /usr/local/sbin/check_ng.sh
    ...
    #!/bin/bash
    #时间变量，用于记录日志
    d=`date --date today +%Y%m%d_%H:%M:%S`
    #计算nginx进程数量
    n=`ps -C nginx --no-heading|wc -l`
    #如果进程为0，则启动nginx，并且再次检测nginx进程数量，
    #如果还为0，说明nginx无法启动，此时需要关闭keepalived
    if [ $n -eq "0" ]; then
            /etc/init.d/nginx start
            n2=`ps -C nginx --no-heading|wc -l`
            if [ $n2 -eq "0"  ]; then
                    echo "$d nginx down,keepalived will stop" >> /var/log/check_ng.log
                    systemctl stop keepalived
            fi
    fi
    ```
    添加执行权限
    `[root@hdp-node-01 ~]# chmod a+x /usr/local/sbin/check_ng.sh`
- 启动master上的keepalived，如果没有启动Nginx服务，他会自动拉起，并监听VIP
    ``` bash
    [root@hdp-node-01 ~]# systemctl start keepalived
    [root@hdp-node-01 ~]# ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
        link/ether 00:0c:29:e8:fa:ed brd ff:ff:ff:ff:ff:ff
        inet 192.168.126.130/24 brd 192.168.126.255 scope global ens33
           valid_lft forever preferred_lft forever
        inet 192.168.126.100/32 scope global ens33
           valid_lft forever preferred_lft forever
        inet6 fe80::1eb5:f05b:5a41:cb/64 scope link
           valid_lft forever preferred_lft forever
    ```
    可以看到master上已经配置了192.168.126.100这个IP，再看下Nginx服务是否已经启动
    ``` bash
    [root@hdp-node-01 ~]# ps aux | grep nginx
    root       1150  0.0  0.0  20496   624 ?        Ss   13:14   0:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
    nobody     1159  0.0  0.1  25024  3508 ?        S    13:14   0:00 nginx: worker process
    nobody     1160  0.0  0.1  25024  3252 ?        S    13:14   0:00 nginx: worker process
    root       3096  0.0  0.0 112676   984 pts/1    S+   15:01   0:00 grep --color=auto nginx
    ```
- 配置backup上的服务
    配置文件：
    ``` bash
    [root@hdp-node-02 ~]# vim /etc/keepalived/keepalived.conf
    ...
    global_defs {
       notification_email {
         root@example.com
       }
       notification_email_from root@aminglinux.com
       smtp_server 127.0.0.1
       smtp_connect_timeout 30
       router_id LVS_DEVEL
    }
    vrrp_script chk_nginx {
        script "/usr/local/sbin/check_ng.sh"
        interval 3
    }
    vrrp_instance VI_1 {
        state BACKUP
        interface ens33
        virtual_router_id 51
        priority 90
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass centos7
        }
        virtual_ipaddress {
            192.168.126.100
        }
        track_script {
            chk_nginx
        }
    }
    ```
    监控脚本：
    ``` bash
    [root@hdp-node-02 ~]# vim /usr/local/sbin/check_ng.sh
    ...
    #时间变量，用于记录日志
    d=`date --date today +%Y%m%d_%H:%M:%S`
    #计算nginx进程数量
    n=`ps -C nginx --no-heading|wc -l`
    #如果进程为0，则启动nginx，并且再次检测nginx进程数量，
    #如果还为0，说明nginx无法启动，此时需要关闭keepalived
    if [ $n -eq "0" ]; then
            /etc/init.d/nginx start
            n2=`ps -C nginx --no-heading|wc -l`
            if [ $n2 -eq "0"  ]; then
                    echo "$d nginx down,keepalived will stop" >> /var/log/check_ng.log
                    systemctl stop keepalived
            fi
    fi
    ```
    启动服务：
    ``` bash
    [root@hdp-node-02 ~]# systemctl start keepalived
    ```    
##### 5. 效果验证
- 区分两台机器的Nginx
    分别修改Nginx默认index.html的内容，在标题后添加master或者backup
    ``` bash
    [root@hdp-node-01 ~]# vim /usr/local/nginx/html/index.html
    ```
    访问效果如下:
    ![master](./images/Linux集群/master.png)
    ![backup](./images/Linux集群/backup.png)
    ![VIP](./images/Linux集群/VIP.png)
    可以看到，直接访问VIP时，提供服务的是master
- 模拟master宕机
    关闭master上的keepalived服务
    ``` bash
    [root@hdp-node-01 ~]# systemctl stop keepalived
    ```
    随后访问VIP
    ![crash](./images/Linux集群/crash.png)
    很明显，此时提供服务的是backup
    我们看一下backup上的日志 `/var/log/messages`，其中有如下内容：
    ```
    Nov 28 16:02:11 hdp-node-02 Keepalived_vrrp[5277]: (VI_1): received an invalid passwd!
    Nov 28 16:02:11 hdp-node-02 Keepalived_vrrp[5277]: bogus VRRP packet received on ens33 !!!
    Nov 28 16:02:11 hdp-node-02 Keepalived_vrrp[5277]: VRRP_Instance(VI_1) ignoring received advertisment...
    Nov 28 16:02:15 hdp-node-02 Keepalived_vrrp[5277]: VRRP_Instance(VI_1) Transition to MASTER STATE
    Nov 28 16:02:16 hdp-node-02 Keepalived_vrrp[5277]: VRRP_Instance(VI_1) Entering MASTER STATE
    Nov 28 16:02:16 hdp-node-02 Keepalived_vrrp[5277]: VRRP_Instance(VI_1) setting protocol VIPs.
    Nov 28 16:02:16 hdp-node-02 Keepalived_vrrp[5277]: Sending gratuitous ARP on ens33 for 192.168.126.100
    Nov 28 16:02:16 hdp-node-02 Keepalived_vrrp[5277]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on ens33 for 192.168.126.100
    ```
    可以看到backup在规定时间内没有接收到vrrp数据，就转换成了master提供服务

#### 搭建负载均衡集群
负载均衡集群容易理解，简单来说就是让多台服务器均衡地承载压力，实现负载均衡的开源软件有LVS，keepalived，haproxy，Nginx等，这里以开源的LVS为主
##### 1. LVS介绍
LVS（Linux Virtual Server）是国内大佬章文嵩开发的，流行程度不亚于Apache的httpd，它是一款四层的负载均衡软件，是针对TCP/IP做的转发和路由，所以稳定性和效率相当高。在LVS的架构中有一个核心的角色叫做调度器（Load Balancer），用来分发用户的请求，还有诸多的真实服务器（Real Server），也就是处理用户请求的服务器。LVS根据实现方式的不同，主要分为三种类型：NAT模式、IP Tunnel（IP隧道）模式、DR模式，这里主要搭建NAT模式的LVS
![LVS](./images/Linux集群/lvs-architecture.jpg)
##### 2. NAT模式
该模式下，调度器会将用户的请求通过预设的iptables规则转发给后端的真实服务器。其中调度器有两个IP，一个是公网IP，一个是内网IP，而真实服务器只有内网IP。用户访问时请求的是调度器的公网IP，调度器把用户的请求转发到真实服务器的内网IP上。这种模式的好处是节省公网IP，但调度器会成为瓶颈，NAT模式架构如下图所示：
![LVS_NAT](./images/Linux集群/lvs-nat.jpg)
客户通过Virtual IP Address（虚拟服务的IP地址）访问网络服务时，请求报文到达调度器，调度器根据连接调度算法从一组真实服务器中选出一台服务器，将报文的目标地址Virtual IP Address改写成选定服务器的地址，报文的目标端口改写成选定服务器的相应端口，最后将修改后的报文发送给选出的服务器。同时，调度器在连接Hash表中记录这个连接，当这个连接的下一个报文到达时，从连接Hash表中可以得到原选定服务器的地址和端口，进行同样的改写操作，并将报文传给原选定的服务器。当来自真实服务器的响应报文经过调度器时，调度器将报文的源地址和源端口改为Virtual IP Address和相应的端口，再把报文发给用户。我们在连接上引入一个状态机，不同的报文会使得连接处于不同的状态，不同的状态有不同的超时值。当连接终止或超时，调度器将这个连接从连接Hash表中删除。
##### 3. NAT模式LVS搭建
NAT模式下，调度器需要有两个IP，这里准备了三台服务器来搭建，三台IP分配如下：
```
Load Balancer: 192.168.126.130（内网IP），192.168.109.128（公网IP，vmware仅主机模式）
Real Server 1: 192.168.126.131
Real Server 2: 192.168.126.132
```
注意调度器上需要两块网卡，作为内网的网卡使用NAT网络，“公网”网卡使用的是仅主机网络，这里的公网是模拟的，非真实意义的公网。开始之前还需要一些准备工作，真实服务器1和真实服务器2需要把内网的网关设为调度器的内网IP（192.168.126.130），随后把三台服务器的iptables规则清空并保存，依次执行 `iptables -F`，`service iptables save`，如果遇到如下报错：“The service command supports only basic LSB actions (start, stop, restart, try-restart, reload, force-reload, status). For other actions, please try to use systemctl.” 解决方法如下：
``` bash
systemctl stop firewalld         //关闭防火墙
yum install iptables-services    //安装或更新iptables服务
systemctl enable iptables        //启动iptables
systemctl start iptables         //打开iptables
```
之后重新执行即可
在Load Balancer上安装ipvsadm，这是实现LVS的核心工具
``` bash
[root@hdp-node-01 ~]# yum install ipvsadm
```
继续在该主机上编写一个脚本：
``` bash
[root@hdp-node-01 ~]# vim /usr/local/sbin/lvs_nat.sh
...
#! /bin/bash
# director 服务器上开启路由转发功能
echo 1 > /proc/sys/net/ipv4/ip_forward
# 关闭icmp的重定向
echo 0 > /proc/sys/net/ipv4/conf/all/send_redirects
echo 0 > /proc/sys/net/ipv4/conf/default/send_redirects
# 注意区分网卡名字，两个网卡分别为ens33和ens37
echo 0 > /proc/sys/net/ipv4/conf/ens33/send_redirects
echo 0 > /proc/sys/net/ipv4/conf/ens37/send_redirects
# director 设置nat防火墙
iptables -t nat -F
iptables -t nat -X
iptables -t nat -A POSTROUTING -s 192.168.126.0/24  -j MASQUERADE
# director设置ipvsadm
IPVSADM='/usr/sbin/ipvsadm'
$IPVSADM -C
$IPVSADM -A -t 192.168.109.128:80 -s wlc -p 300
$IPVSADM -a -t 192.168.109.128:80 -r 192.168.126.131:80 -m -w 1
$IPVSADM -a -t 192.168.109.128:80 -r 192.168.126.132:80 -m -w 1
```
上面的配置文件中，ipvsadm的`-C`清空规则，避免受之前规则的影响；`-A`增加Virtual Server，`-t`为TCP，`-s`指定调度算法，wlc为带权重的最小连接算法，`-p`指定超时时间（300为300秒，表示300秒内相同用户的请求会一直被调度到同一台Real Server上，这里为了后续检验结果，先设为300），`-a`增加Real Server，`-r`指定Real Server的IP，`-m`表示LVS的模式为NAT，如果是`-g`表示为DR模式，`-i`表示为IP Tunnel模式，`-w`指定权重。保存之后执行即完成配置
``` bash
[root@hdp-node-01 ~]# bash /usr/local/sbin/lvs_nat.sh
```
下面测试效果，如果调度器上开启有Nginx服务器，需要关闭，否则影响实验效果：
``` bash
[root@hdp-node-01 ~]# killall nginx
```
为了便于区分，我们给两台Real Server设置不同的默认主页
``` bash
[root@hdp-node-02 ~]# echo "Real Server 1" > /usr/local/nginx/html/index.html
[root@hdp-node-03 ~]# echo "Real Server 2" > /usr/local/nginx/html/index.html
```
在调度器上分别访问两台真实服务器，结果如下：
``` bash
[root@hdp-node-01 ~]# curl 192.168.126.131
Real Server 1
[root@hdp-node-01 ~]# curl 192.168.126.132
Real Server 2
```
接着在调度器上访问调度器的外网IP（192.168.109.128）
``` bash
[root@hdp-node-01 ~]# curl 192.168.109.128
Real Server 2
[root@hdp-node-01 ~]# curl 192.168.109.128
Real Server 2
[root@hdp-node-01 ~]# curl 192.168.109.128
Real Server 2
```
连续访问多次，一直都在Real Server2上，这是因为设置了`-p`参数，理论上300秒内请求都会转发到Real Server2上。现在重新编辑脚本，删除该参数，重新执行脚本，然后再次测试：
``` bash
[root@hdp-node-01 ~]# curl 192.168.109.128
Real Server 2
[root@hdp-node-01 ~]# curl 192.168.109.128
Real Server 1
[root@hdp-node-01 ~]# curl 192.168.109.128
Real Server 2
[root@hdp-node-01 ~]# curl 192.168.109.128
Real Server 1
```
可以发现两台服务器交替提供服务，这样就做到了负载均衡
**注意**，如果在上面crul访问公网IP长时间无响应，需要查看是否关闭防火墙，`systemctl stop firewalld`，以及关闭SELinux（排查了好久结果是SELinux没关）
