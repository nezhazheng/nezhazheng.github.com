---
layout: post
title:  "一次MYSQL Replication 测试"
categories: "database"
---

###环境

##### Master:

物理机OS X环境，已经有了一个运行了一段时间的mysql，mysql版本5.6.14

##### Slave:

虚拟机Ubuntu环境，通过apt-get装了一个干净的mysql，mysql版本5.5

### Step1 - 创建复制账号

MYSQL会赋予一些特殊的权限给复制线程，在备库运行的I/O线程会建立一个到主库的TCP/IP连接，这意味着必须在主库创建一个用户，并赋予其合适的权限。备库I/O线程以该用户名连接到主库并读取其二进制日志。

`GRANT REPLICATION SLAVE, REPLICATION CLENT ON *.* TO repl@‘192.168.1.%’ IDENTIFIED BY ‘password’;`

### Step2 - 配置主库和备库

#### 主库配置

log_bin=mysql-bin

server_id = 10

必须明确地指定一个唯一的server_id，log_bin用于开启二进制日志。

可以使用SHOW MASTER STATUS;检查二进制日志文件是否已经在主库上创建。

#### 备库配置

ubuntu默认的my.cnf文件在/etc/mysql下

server-id          = 2

relay_log          = /var/lib/mysql/mysql-relay-bin

log_bin               = mysql-bin

log_slave_updates     = 1

### Step3 - 启动复制
{% highlight sql %}
CHANGE MASTER TO MASTER_HOST=’server1’,
               MASTER_USER=‘repl’,
               MASTER_PASSWORD=‘pass4word’,
               MASTER_LOG_FILE=‘mysql-bin.000001’,
               MASTER_LOG_POS=0;`
               
START SLAVE;
{% endhighlight %}

由于主库是有数据的，备库是干净的没同步过得，上述步骤执行过后，通过`show slave status`查看复制情况，会出现一个带有关键字checksun的fail msg。

如果表engine是innodb的话可以通过mysqldump来同步主库和备库的数据，这种方式的好处是热备不用停数据库。

`mysqldump -uroot -pxxx --single-transaction --all-database --master-data=1 --host=192.168.1.102 | mysql -uroot -pxxx --host=192.168.1.104`

执行后，因为mysql版本不同步，备库的版本要低，所以出现了一系列error。

通过下面的步骤升级ubuntu上的mysql到5.6版本。

1.删除老版本mysql

`apt-get remove mysql-common mysql-server-5.5 mysql-server-core-5.5 mysqlclient-5.5 mysql-client-core-5.5`

`apt-get autoremove`

2.下载5.6版本mysql并安装

`wget -O mysql-5.6.deb http://cdn.mysql.com/Downloads/MySQL-5.6/mysql-5.6.14-debian6.0-x86_64.deb`

`dpkg -i mysql-5.6.deb`

`apt-get install libaio1`

3.设置5.6启动脚本
 
`cp /opt/mysql/server-5.6/support-files/mysql.server /etc/init.d/mysql.server`

`update-rc.d -f mysql remove`

4.更新环境变量

`PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/opt/mysql/server-5.6/bin"`

5.启动

`service mysql.server start`

再次执行mysqldump命令同步数据，提示成功！

再次执行`start slave`启动复制。

使用`show slave status\G`命令查看复制状态，出现下面这两行，说明复制成功。

Slave_IO_Running: Yes

Slave_SQL_Runing: Yes

另外如果想限制只复制一部分数据，可以使用复制过滤器。

`replicate_wild_do_table: db1.%`

这个配置会在重复备库relay_log时，进行数据的过滤。
