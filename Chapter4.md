## Linux平台日常运维管理

#### 查看系统负载
* `w`  
第一行从左面开始显示的信息依次为：时间、系统运行时间、登录用户数、平均负载。  
第二行以下主要是显示当前登录的用户以及他们从何处登录，`JCPU`指的是和该终端连接的所有正在运行进程占用的时间，`PCPU`指当前进程所占用的时间
```
10:42:00 up 1 day,  1:09,  1 user,  load average: 0.59, 0.72, 0.67
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
taxue    pts/0    :0               二09    0.00s  0.08s  0.00s w
```
`load average`后面的三个数值依次表示１、５、15分钟内系统的平均负载值，这个值的意义是单位时间段内CPU活动进程数。一般情况下数值不超过CPU数量就没关系。
* `cat /proc/cpuinfo`可以查看CPU信息  
`grep -c 'processor' /proc/cpuinfo`可以查看CPU数量
* `uptime`也可以查看系统负载，显示的结果与`w`第一行一致

#### vmstat详解
* `vmstat 1`执行结果如下，其中参数`1`表示１秒显示一次，后面可以再加一个数字参数表示一共显示几次，比如`vmstat 1 5`就是１秒１次，显示５次
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0 201812 663368     28 2809320    0    1    21    34   51   97  5  2 93  0  0
 0  0 201812 662360     28 2809896    0    0     0     0 1762 11263  5  1 93  1  0
 0  0 201812 663072     28 2809900    0    0     0     0 2415 9899  4  2 94  0  0
```
* `procs`显示进程相关信息  
`r`表示置于运行队列中的进程数目（包括正在执行和等待CPU的），如果长期大于CPU个数，说明CPU不够用  
`b`表示等待资源的进程数。比如等待I/O、内存等，值如果长时间大于１，则需要关注
* `memroy`内存相关信息  
`swpd`表示切换到交换分区中内存数量  
`free`当前空闲的内存数量  
`buff`缓冲大小（即将写入磁盘）  
`cache`缓存大小（从磁盘中读取）
* `swap`内存交换情况  
`si`由交换区写入内存的数据量  
`so`由内存写入交换区的数据量
* `io`磁盘使用情况  
`bi`从块设备读取数据的量（读磁盘）  
`bo`从块设备写入数据的量（写磁盘）
* `system`显示采集间隔内发生的中断次数  
`in`表示某一时间间隔中观测到的每秒中断数  
`cs`表示每秒产生的上下文切换次数
* `cpu`显示cpu的使用状态  
`us`显示用户进程执行时间百分比  
`sy`显示系统进程执行时间百分比  
`id`表示cpu处于空闲状态的时间百分比  
`wa`表示I/O等待所占用cpu时间百分比  
`st`标好似被偷走的cpu时间百分比（一般为０，不用关注）
* 通常关注`r`、`b`、`wa`。如果磁盘I/O压力很大时，`bi`与`bo`数值也会较高。当`si`与`so`较高并不断变化时，说明内存不够，内存中数据频繁交换到交换分区，对系统性能影响极大。

#### top动态查看负载
* `top`动态监控进程所占系统资源，每3秒变更一次，特点是将占用系统资源（CPU，内存，磁盘I/O）最高的进程放到最前面，`RES`为进程所占内存大小。
* `top`状态下，按`M`即`shift+m`可以按照内存使用大小排序，按数字`1`可以列出各cpu的使用状态
* `top -bn1`非动态打印系统资源使用情况，会一次性输出所有信息

#### sar命令
`sar`可以监控系统所有资源状态  
* `sar -n DEV`查看网卡流量，`-f`选项可以查看历史流量，后面跟文件名。Redhat和CentOS发行版`sar`的库文件一定是在`/var/log/sa/`目录下
* `ls /var/log/sa`会看到两种不同的文件，一个以`sa`开头加日期，一个以`sar`开头加日期，前一个只能用`sar -f`查看，后一种可以`cat`查看，它们都是记录的系统状态历史信息
* `sar -q`查看历史负载
* `sar`可以查看cpu情况，`-b`可以查看I/O情况等

#### free查看内存
* `free`执行结果如下  
```
total       used       free     shared    buffers     cached
Mem:       8072124    7163300     908824     497780         32    1857360
-/+ buffers/cache:    5305908    2766216
Swap:      8385532     435632    7949900
```
`-m`和`-g`参数可以分别以M或G为单位显示结果，其中`buffers`为缓冲，将分散的写操作集中进行，以减少反复寻道，提升系统性能。`cached`是将一部分读取过的内容保存，重新读取就不需命中硬盘，所以简单来说，前者是内存中写入磁盘的，后者是从磁盘中读取出的。

#### ps查看进程
* `ps aux`执行结果如下（截取部分）
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  34364  5212 ?        Ss   4月05   0:09 /usr/lib/systemd/systemd --switched-root --system --deserialize 21
root         2  0.0  0.0      0     0 ?        S    4月05   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        S    4月05   0:01 [ksoftirqd/0]
root         5  0.0  0.0      0     0 ?        S<   4月05   0:00 [kworker/0:0H]
root         7  0.1  0.0      0     0 ?        S    4月05   5:06 [rcu_preempt]
```
* `PID`表示进程id，可以用`kill [pid]`来终止进程，如果遇到不能杀掉的情况，加参数`-9`即可（无条件终止）
* `STAT`表示进程的状态，主要有以下几种  
`D`不能中断的进程（通常为I/O）  
`R`正在运行中的进程  
`S`已经中断的进程，通常情况下，系统中大部分进程都是这个状态  
`T`已经停止或者暂停的进程  
`<`高优先级进程  
`N`低优先级进程  
`s`主进程  
`l`多线程进程  
`+`前台运行的进程  
`Z`僵尸进程，杀不掉，仅占用少量系统资源  
`L`在内存中被锁了内存分页
* 可以用`ps aux | grep -c keyword`查看相关进程

#### netstat查看端口
* `netstat -lnp`打印当前系统启动哪些端口
* `netstat -an`打印网络连接状况

#### 抓包工具tcpdump和tshark
* `tcpdump -n -i eth0`该命令可以查看指定网卡（eth0）的数据流向，`-i`是指定设备，`-n`可以避免将ip和端口号转换为主机名和服务名
* `tcpdump -nn -i eth1 host 192.168.1.100 and port 80 -c 100 -w 1.cap`  
该命令的一种用法，可以用`host`指定ip，`port`指定端口，`-c`指定包数量，`-w`写入指定文件。此时`1.cap`文件里是包的内容，不加`-w`则直接显示的是数据流向，包的内容无法用`cat`查看，可以用`tcpdump -r file`或者`wireshark`查看，要抓取完整的包需要参数`-s0`
* `tshark`也可用于抓包，需要先安装`wireshark`
* `tshark -n -t a -R http.request -T fields -e "frame.time" -e "ip.sec" -e "http.host" -e "http.request.method" -e "http.request.uri"`  
__该用法可以显示访问http请求的域名以及uri__
* `tshark -n -i eth1 -R 'mysql.query' -T fields -e "ip.src" -e "mysql.query"`  
__该指令可以抓取eth1上对mysql的查询__
* `tshark -n -i eth1 -R 'mysql matches "SELECT|INSERT|DELETE|UPDATE"' -T fields -e "ip.src" -e "mysql.query"`  
__抓取指定类型的mysql查询__
* `tshark -n -q -z http,stat, -z http，tree`  
__统计http的状态__

#### selinux介绍
`Selinux`是一种安全机制，实际使用中因为配置繁琐可能导致一些问题而通常情况下将其关闭。  
* 编辑`/etc/selinux/config`，将`SELINUX=enforcing`改为`SELINUX=disabled`，保存更改，重启即可生效
* `setenforce 0`可临时关闭`selinux`
* `getenforce`可获得当前`selinux`的状态

#### iptables详解
`netfilter`是linux的一种防火墙，实现工具是`iptables`，`iptables`有三个表，默认是`filter`，`nat`，`mangle`，当未指定操作对象时，默认是对`filter`表进行操作
* `filter`表用于过滤包，是系统预设的表，内建三个链，`INPUT`，`OUTPUT`，`FORWARD`，依次作用于进入本机的包，本机送出的包，跟本机无关的包；  
`nat`用于网络地址转换，三个链`PREROUTING`，`OUTPUT`，`POSTROUTING`，作用依次为在包刚到达防火墙时改变它的目的地址，改变本地产生包的目的地址，及在包就要离开防火墙之前改变其源地址。  
`mangle`用于给数据包标记，然后根据标记操作包，基本用不到。  

__iptables语法__
* `iptables -t filter -nvL`该指令指定查看`filter`的详细规则，`-t`指定查看对象，`-nvL`查看规则，`-Z`把包以及流量计数器置零。`-F`临时清除当前规则，但重启后会重新加载
* `iptables -A INPUT -s 10.71.11.12 -p tcp --sport 1234 -d 10.72.137.159 --dport 80 -j DROP`  
该指令增加了一条规则，`-A`表示增加规则，`-I`表示插入规则，两者的区别在于前者新增规则出现在规则表的最下面，后者出现在最上面，而且后者插入的规则优先生效。`-D`删除一条规则  
`INPUT`为链名称，还可以是`OUTPUT`或者`FORWARD`。`-s`后跟源地址ip，`-p`协议(tcp,udp,icmp)，`--sport/--dport`后跟源端口／目标端口，`-d`后跟目的IP，`-j`后跟动作(DROP丢掉,REJECT拒绝,ACCEPT允许)，`-i`指定网卡，不常用。  
* `iptables -nvL --line-numbers`，此时显示的规则有数字标记，删除时只要执行`iptables -D 链 number`即可便捷删除某条规则  

__保存及备份iptables规则__
* `service iptables save`保存规则，通常情况下规则保存在`/etc/sysconfig/iptables`
* `iptables-save > backup.rule`可将规则备份到文件`backup.rule`
* `iptables-restore < backup.rule`从备份文件恢复规则
* 注意默认情况下操作的对象是`filter`表

#### cron计划任务
Linux任务计划功能的操作都是通过`crontab`完成的，常用的选项有：  
* `-u`，指定某用户，不加即为当前用户
* `-e`，制定计划任务，该选项会使用_vim_打开_crontab_配置文件
* `-l`，列出计划任务
* `-r`，删除计划任务
* `01 10 05 06 3 echo "ok" >/root/cron.log`  
这是一条计划任务，从左到右依次为：分，时，日，月，周，命令，上例含义是6月5日（必须是星期3）的10点01分执行命令`echo "ok" >/root/cron.log`
* 每隔八小时，在小时位上可以写成`*/8`，当遇到多个数，需要用逗号隔开，比如`1，12，18`，时间段可以用`n-m`的方式表示
* 查看服务是否启动，CentOS上是`service crond status`,一些系统比如Ubuntu，openSUSE上是`service cron status`
