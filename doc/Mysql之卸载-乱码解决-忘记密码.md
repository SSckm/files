## 问题描述

```
有时候大家会卸载Mysql（但是我也实在是想不出来什么情况下会卸载mysql），这篇文章主要写的是忘记mysql密码更改密码，如何卸载mysql，以及解决Mysql乱码的问题。
```

## 环境准备

```
Ubuntu 16.4
mysql 5.7.18
```

## 忘记密码解决（如果是因为忘记密码可以先看这里）

### 1.暂停mysql的服务

```
sudo killall mysqld
```

### 2.修改配置文件

```
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf（非源码编译路径，如果是自己源码编译的可在安装路径下面找到配置文件）
在[mysqld]下面加上下面的数据
skip-grant-tables, 加上后代码如下：

[mysqld]
#
# * Basic Settings
#
user            = mysql
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
port            = 3306
basedir         = /usr
datadir         = /var/lib/mysql
tmpdir          = /tmp
lc-messages-dir = /usr/share/mysql
skip-external-locking
character-set-server=utf8
skip-grant-tables
```

### 3.启动mysql

```
sudo service mysql start
```

### 4.登陆mysql

```
mysql -u root (注意此时无需密码)
```

### 5.修改密码

注意：因为在mysql 5.7.18的版本中 mysql已经去掉了password字段，密码字段加密后存储在authentication_string字段中。（修改的时候可以看下user表中的存储密码的字段）

```
use mysql;
update mysql.user set authentication_string=password('xxxxxxx') where user='root' and Host = 'localhost'
```

此时修改密码已经完成。

### 6恢复数据

```
把原来的skip-grant-tables这行从配置文件中删除掉，重启mysql就可以用新修改的密码来登陆了。
```

## 彻底删除mysql

```
sudo apt-get autoremove --purge mysql-server
sudo apt-get remove mysql-server
sudo apt-get remove mysql-common
```

## 修改mysql的编码（utf-8）

```
mysql> show variables like "%char%"; 
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | latin1                     |
| character_set_connection | latin1                     |
| character_set_database   | latin1                     |
| character_set_filesystem | binary                     |
| character_set_results    | latin1                     |
| character_set_server     | latin1                     |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+


mysql> status; (可以看到编码都是latin1)
--------------
Connection id:		7
Current database:	
....
Connection:		Localhost via UNIX socket
Server characterset:	latin1
Db     characterset:	latin1
Client characterset:	latin1
Conn.  characterset:	latin1

sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
在[mysqld]下加入
character-set-server=utf8

现在的mysql 5.7.18在配置文件中找不到[client]标签了，如果只在[mysqld]下面加的话可以看下字符集：

mysql> show variables like "%char%";
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | latin1                     |
| character_set_connection | latin1                     |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | latin1                     |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
说明还是没有改过来。

此时只需要在配置文件的末尾加上下面两行代码：

[client]
default-character-set=utf8

mysql>\q (此时需要退出，然后在用用户进入)

mysql -uroot -p
mysql> show variables like "%char%";
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
此时已经全部改过来了。
```

