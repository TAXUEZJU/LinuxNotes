##  Linux初讲

### 虚拟机设置及安装
* 一般分 */boot*，*/swap*， */* 即可
*	*boot*分区200M基本足够，不选择加密否则无法正常启动；*swap*分区大小一般为内存的两倍，但不多于8G
* 网络环境中有路由器，可以自动获取ip时虚拟机网络模式可以设为“桥接”，否则设为NAT
* 设置grub密码可防止别人进入单用户模式修改root密码

### IP设置
* `ifconfig`查看网卡ip，其中`lo`是回环网卡，用于机器内部通信，可通过`ipconfig -a`查看所有网卡
* 若网络环境中有dhcp服务器，可以执行`dhclient`自动获得ip
* `route`  

指令查看ip路由表，这里主要用来查看 *NETMASK* 和 *GATEWAY*
* `/etc/sysconfig/network-scripts/ifcfg-eth0`  
为网卡配置文件，注意网卡一般为`eth0`，具体视情况而定
* 文件内容主要修改 **ONBOOT** 为 *yes*，为开机启动网卡； **BOOTPROTO** 改为 *static*，添加<u>ip地址，子网掩码，网关，DNS</u>，示例:

>TYPE=Ethernet
>BOOTPROTO=static
>DEFROUTE=yes
>PEERDNS=yes
>PEERROUTES=yes
>IPV4_FAILURE_FATAL=no
>IPV6INIT=yes
>IPV6_AUTOCONF=yes
>IPV6_DEFROUTE=yes
>IPV6_PEERDNS=yes
>IPV6_PEERROUTES=yes
>IPV6_FAILURE_FATAL=no
>IPV6_ADDR_GEN_MODE=stable-privacy
>NAME=ens33
>UUID=8c942f27-ec92-43c6-89c7-136692acb217
>DEVICE=ens33
>ONBOOT=yes
>IPADDR=192.168.126.129
>NETMASK=255.255.255.0
>GATEWAY=192.168.126.2
>DNS1=119.29.29.29

*	重启网络服务`systemctl restart netwrok.service`，能ping通外网说明设置成功了
* `ifdown`与`ifup`可以关闭或和开启网卡

### 系统启动流程
*	详见教材P21~24，了解即可，多用于排障

### 远程连接工具使用
<u>
SSH支持密钥认证，其中公钥用于加密，私钥用于解密，可以用puttygen生成密钥对
</u>

*	公钥内容复制到`/root/.ssh/authorized_keys`中
* 更改目录文件权限，  
`chmod 700 /root/.ssh`  
`chmod 600 /root/.ssh/authorized_keys`
* 关闭防火墙，iptables和SELinux，  
`setenforce 0`  
该指令暂时关闭SELinux，永久关闭SELinux需要修改配置文件  
`vi /etc/selinux/config`  
将`SELINUX=enforcing`修改为`disabled`，然后重启系统
* `iptables -F`  
临时清除规则  
`service iptables save`  
将规则保存到配置文件中，配置文件为`/etc/sysconfig/iptables`

### 运行级别
* **init 0**　halt，关机
* **init 1**　单用户模式
* **init 2**　多用户模式，无NFS服务
* **init 3**　完整多用户模式
* 　**4**　　为预留模式
* **init 5**　X11，图形界面模式
* **init 6**　重启  
init 6  ==  reboot ==  shutdown -r now

### 单用户模式
<u>
开机时按回车进入grub选项，选中发行版，按`e`进入编辑模式，在内核启动项后加`single`或者`s`或者`1`，回车后按`b`即启动进入单用户模式
</u>

* 单用户模式下可用`passwd`指令修改root密码

### 救援模式
当grub损坏或者因为配置文件修改出错导致系统无法进入，需要使用救援模式
* 在bios中调整光盘启动优先，光盘启动后选择**Rescue installed system**
* Rescue环境会找到已安装的linux系统，并挂载到`/mnt/sysimage`
* 如果需要root环境，执行  
`chroot /mnt/sysimage`

### 快捷键使用
* `Ctrl+C`　终止当前指令
* `Tab`　　 补全命令和目录
* `Ctrl+D`　退出当前终端，或使用exit
* `Ctrl+Z`　暂停当前进程，暂停后可用`fg`恢复
* `Ctrl+L`　清屏，使光标移至第一行，与`clear`同
* `Ctrl+U`　快速删除光标前面的所有字符
* `Ctrl+K`　快速删除光标后面的所有字符
* `Ctrl+A`　快速把光标定位到行首
* `Ctrl+E`　快速把光标定位到行尾

### 环境变量
* `which`用来查询命令的绝对路径，可以查看命令别名和位置
* `alias`可查看所有别名
* `alias`可为较长命令设置别名，但尽限当此登录，若想永久有效，需将设置写入`.bashrc`，当前用户配置文件为`~/.bashrc`  
**alias**语法示例  
`alias shadowsocks-hk='sudo sslocal -c /etc/shadowsocks/hk02.json -d start'`
* `echo $PATH`可显示环境变量**PATH**的值，可用如下命令添加  
`PATH=$PATH:/(目录)`

### 一些指令
* `cd`切换目录，注意绝对路径和相对路径的使用  
`./`表示当前目录，`../`表示当前目录的上一级目录
* `pwd`显示当前目录
* `ls`命令  
`-l` 列出详细信息  
`-a` 列出所有文件，包括隐藏文件  
`-t` 按文件的最后更改时间排序  
`-d` 只列出当前目录本身
