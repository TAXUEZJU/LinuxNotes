### 安装MySQL并配置远程连接
CentOS7上通过`yum`安装MySQL，参考官方教程：[A Quick Guide to Using the MySQL Yum Repository](https://dev.mysql.com/doc/mysql-yum-repo-quick-guide/en/)
MySQL的Yum源提供了Linux平台上MySQL服务器，客户端以及其他组件的RPM包，通过添加源可以方便的安装升级MySQL
#### 1. 添加MySQL的Yum源并安装
- 访问MySQL的Yum源下载地址，https://dev.mysql.com/downloads/repo/yum/
- 下载与发行版匹配的包
- 安装该rpm包
    ```
    rpm -Uvh mysql57-community-release-el7-11.noarch.rpm
    ```
- 注意，当使用MySQL Yum源，默认安装的是MySQL的最新GA版本，如果有特殊需求，请访问上述链接启用不同的子仓库，这里直接安装
    ```
    yum install mysql-community-server
    ```
#### 2. 开启MySQL服务
- 使用如下命令开启MySQL服务
    ```
    systemctl start mysqld.service
    ```
    可以通过如下命令查看MySQL服务器的状态
    ```
    systemctl status mysqld.service
    ```
- 在服务初始化的时候将会发生以下事件
    - 初始化服务
    - 数据目录中存在SSL的证书和密钥文件
    - validate_password 插件安装并启用
    - 创建超级用户`root@localhost`，密码存储在error日志文件中，使用以下命令查看初始密码
        ```
        grep 'temporary password' /var/log/mysqld.log
        ```
- 登陆并修改密码
    ```
    shell> mysql -uroot -p
    mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewPass6!';
    ```
    注意由于MySQL的<u>validate_password</u> 插件，设置的密码必须符合以下条件：
    1. 至少一个大写字母；
    2. 至少一个小写字母；
    3. 至少一个数字；
    4. 至少一个特殊字符；
    5. 密码总长度不小于8；
#### 3. 设置MySQL远程登陆
- 进入数据库，然后输入如下指令
    ```
    grant all privileges on *.* to 'root'@'%' identified by 'password';
    flush privileges;
    ```
    其中`password`自定义，如果不关闭插件则需要符合上述密码规则
    上面命令中，`*.*`表示任意数据库的任意表，`root`表示远程登陆使用的用户名，可以自定义，`%`表示允许任意IP访问，如果需要指定IP，则替换为所需内容即可
