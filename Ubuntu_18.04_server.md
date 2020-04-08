# Ubuntu 18.04 server 修改 apt 软件源为阿里云的源

1. 复制源文件备份，以防万一 


```shell
sudo cp /etc/apt/sources.list /etc/apt/sources.list.back
```

2. 编辑源列表文件 


```shell
# 清空list
echo > /etc/apt/sources.list
sudo vim /etc/apt/sources.list

#将原有的内容注释掉，添加以下内容
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
```

3. 添加elementaryos软件源

```shell
sudo wget -O - http://package.elementaryos.cn/bionic/key/package.gpg.key | sudo apt-key add -

sudo echo 'deb http://package.elementaryos.cn/bionic/ bionic main' >> /etc/apt/sources.list
```


4. 更新软件源 


```shell
sudo apt-get update&&sudo apt-get upgrade&&sudo apt-get install -f
```
# 设置root密码
```shell
sudo passwd root
## 输入两次密码
```
# xshell上传下载文件
```shell
sudo apt install  lrzsz -y
ps -ef |grep lrzsz
rz #本地上传服务器
sz filepath #sz pro_cel.rar
```
# 安装压缩包
1、压缩功能
安装 sudo apt-get install rar
卸载 sudo apt-get remove rar
2、解压功能
安装 sudo apt-get install unrar
卸载 sudo apt-get remove unrar
压缩解压缩.rar
解压：rar x FileName.rar
压缩：rar a FileName.rar DirName

一、.tar

解包：tar xvf FileName.tar -C 路径

打包：tar cvf FileName.tar DirName

（注：tar是打包，不是压缩！）

二、.gz

解压1：gunzip FileName.gz

解压2：gzip -d FileName.gz

压缩：gzip FileName

.tar.gz

解压：tar zxvf FileName.tar.gz

压缩：tar zcvf FileName.tar.gz DirName

三、.bz2

解压1：bzip2 -d FileName.bz2

解压2：bunzip2 FileName.bz2

压缩： bzip2 -z FileName

.tar.bz2

解压：tar jxvf FileName.tar.bz2

压缩：tar jcvf FileName.tar.bz2 DirName

三、.bz

解压1：bzip2 -d FileName.bz

解压2：bunzip2 FileName.bz

.tar.bz

解压：tar jxvf FileName.tar.bz

四、.Z

解压：uncompress FileName.Z

压缩：compress FileName

.tar.Z

解压：tar Zxvf FileName.tar.Z

压缩：tar Zcvf FileName.tar.Z DirName

五、.tgz

解压：tar zxvf FileName.tgz

.tar.tgz

解压：tar zxvf FileName.tar.tgz

压缩：tar zcvf FileName.tar.tgz FileName

六、.zip

解压：unzip FileName.zip

压缩：zip FileName.zip DirName

七、.rar

解压：rar a FileName.rar

压缩：rar e FileName.rar

八、.lha

解压：lha -e FileName.lha

压缩：lha -a FileName.lha FileName

# scp传输文件

scp [参数] <源地址 (用户名@IP地址或主机名)>:<文件路径> <目的地址（用户名 @IP 地址或主机名）>:<文件路径>

参数:

-r 传输文件夹

-v 展示传输详情
```shell
scp /home/soft/jdk-7u55-linux-i586.tar.gz root@192.168.132.132:/

scp -r /home/soft root@192.168.132.132:/
```


#  配置桌面快捷方式
创建desktop文件

```shell
vim Postman.desktop

[Desktop Entry]
Encoding=UTF-8
Name=Postman
Exec=/opt/Postman/Postman/Postman
Icon=/opt/Postman/Postman/app/resources/app/assets/icon.png
Terminal=false
Type=Application
Categories=Development;

sudo cp Postman.desktop /usr/share/applications/
```
# 安装GNOME桌面

1. 输入以下命令
```shell
sudo apt-get install tasksel -y
sudo tasksel
```

2. 选择Ubuntu desktop 

 https://blog.csdn.net/zhangtao_heu/article/details/81989147 

3. 如需安装LAMP选择LAMP server

# 安装Python虚拟环境

1. 配置基本环境

```shell
sudo apt-get install -y python3-pip
sudo apt-get install build-essential libssl-dev libffi-dev python-dev
```
2. 安装venv

```shell
sudo apt install virtualenv
sudo apt install virtualenvwrapper
pip3 install virtualenv
pip3 install virtualenvwrapper
```
3. 修改配置文件
```shell
sudo vim ~/.bashrc
```
在.bashrc文件末尾添加两行：
export WORKON_HOME=$HOME/.virtualenvs
source /usr/share/virtualenvwrapper/virtualenvwrapper.sh

4. 启用配置文件
```shell
source ~/.bashrc
```
5. 检查是否可以创建虚拟环境
```shell
mkvirtualenv Django -p /usr/bin/python3
# 或者在~/.bashrc文件中设置环境变量VIRTUALENVWRAPPER_PYTHON=/usr/bin/python2.7
workon Django  激活虚拟环境
deactivate     注销当前已经被激活的虚拟环境
lsvirtualenv   显示已安装虚拟环境
```

# 查看端口
```shell
ps -ef|grep 8000
netstat -tunlp|grep 8000
```

# 防火墙
- selinux 内置防火墙
1. 查询selinux状态
getenforce
2. 暂时停止selinux
setenforce 0
3. 永久关闭selinux
vim /etc/selinux/conf
SELINUX=disabled

- 软件防火墙iptables
iptables -F #清空规则
iptables -L #查看iptables防火墙规则

- 停止防火墙服务
systemctl start/restart/stop firewalld
systemctl disable firewalld #停止iptables的开机自启

# linux 字符编码
1. 编辑字符编码文件
vim /etc/locale.conf
2. 写入如下变量
LANG="zh_CN.UTF-8"
3. 读取文件是变量生效
source /etc/locale.conf
4. 查看系统变量
echo $LANG

# DNS 配置文件
vim /etc/resolv.conf

nameserver 119.29.29.29
nameserver 223.5.5.5

/etc/hosts #本地dns强制解析域名文件

nslookup 



# 创建自己的BLOG

## [Sphinx](http://sphinx-doc.org/)生成网页

1. 安装Sphinx

```shell
pip install sphinx sphinx-autobuild sphinx_rtd_theme
```

2. 创建一个工程项目

```shell
mkdir docs
cd docs
sphinx-quickstart

# 其他按照默认设置
Project name: LHM's BLOG
Author name(s): LHM
Project release []: 1.0
Project language [en]: zh_CN

tree
...
index.rst   #index文件
conf.py      #配置文件
...

# 生成html
make html
```



## Markdown

1. 安装recommonmark

```shell
pip install recommonmark
```

2. 在conf.py添加下面内容

```python
from recommonmark.parser import CommonMarkParser

source_parsers = {
    '.md': CommonMarkParser,
}

source_suffix = ['.rst', '.md']
```



## 修改主题

```shell
import sphinx_rtd_theme

html_theme = 'sphinx_rtd_theme'
html_theme_path = [sphinx_rtd_theme.get_html_theme_path()]
```





## Pandoc格式转换

markdown文件转换rst

```shell
pandoc -V mainfont="SimSun" -f markdown -t rst Ubuntu_18.04_server.md -o Ubuntu_18.04_server.rst
```



# Linux 定时任务
1. 查看crontab
crontab -l
2. 编辑crontab
crontab -e
3. 编辑配置文件
vim /etc/crontab
```shell
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed #命令必须绝对路径
  分 时 日  月 周
  0  0 1   *  * /usr/bin/systemctl restart nginx
  每月1日0点重启nginx
```

# mysql安装

1. 使用apt get 命令即可安装MySQL

```shell
sudo apt-get install -y mysql-server mysql-client
```

2. 启动、关闭和重启MySQL 服务的命令如下

```shell
sudo service mysql start
sudo service mysql stop
sudo service mysql restart
```
3. 初始化

```shell
mysql_secure_installation

Enter current password for root (entry for none):entry
Set root password? Y  #设置root密码
Remove anonymous users? Y  #删除匿名用户
Disallow root login remotely? n  #允许远程登陆
Remove test database and access to it? Y #删除测试数据库
Reload privilege tables now? Y #重新加载权限表
```

4. 因为安装的过程中没让设置密码，可能密码为空，但无论如何都进不去mysql 

1.  修改配置文件

```shell
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
配置文件中的[mysqld]这一块中加入
character-set-server=utf8
collation-server=utf8_general_ci
skip-grant-tables
   
service mysql restart  重新启动mysql
```

2. 修改mysql  root密码

```shell
mysql -u root -p    遇见输入密码的提示直接回车即可,进入mysql

use mysql;
update user set authentication_string=password("你的密码") where user="root";  
flush privileges;
select user,plugin from user;  
# 如果plugin='auth_socket'
update user set plugin='mysql_native_password' where user='root';
select user,host from user;  
# 如果host='localhost'
update user set host = '%' where user = 'root';  远程连接
     quit 		退出mysql
```

3. 注释配置文件

```shell
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
#  skip-grant-tables

# bind-address  = 127.0.0.1   远程连接
service mysql restart  重新启动mysql
```

4. 创建非root账号

```shell
-- 创建普通用户，权限非常低
create user uroot@'%' identified by 'uroot';
```

5. 添加权限

```shell
-- 对所有库和所有表授权所有权限
grant all privileges on *.* to uroot@'%' ;
-- 授权root用户需identified 密码
grant all privileges on *.* to root@'%'  identified by 'root';
-- 刷新授权表
flush privileges; 
```


## mysql数据备份与恢复
1. 导出数据库

```shell
mysqldump -u root -p --all-databases > /data/AllMysql.dump
```

2. 登录mysql 恢复数据库

```shell
source /data/AllMysql.dump;
```

3. 或者命令行恢复数据库

```shell
mysql -uroot -p  <  /data/AllMysql.dump
```



## mysql集群(多实例)

### 什么是MySQL多实例

MySQL多实例就是在一台机器上开启多个不同的服务端口（如：3306，3307，3308），运行多个MySQL服务进程，通过不同的socket监听不同的服务端口来提供各自的服务。

### MySQL多实例的特点有以下几点

- 有效利用服务器资源，当单个服务器资源有剩余时，可以充分利用剩余的资源提供更多的服务。
- 节约服务器资源
- 资源互相抢占问题，当某个服务实例服务并发很高时或者开启慢查询时，会消耗更多的内存、CPU、磁盘IO资源，导致服务器上的其他实例提供服务的质量下降；

### 部署mysql多实例的两种方式

第一种是使用`多个配置文件`启动不同的进程来实现多实例，这种方式的优势逻辑简单，配置简单，缺点是管理起来不太方便；

第二种是通过官方自带的`mysqld_multi`使用单独的配置文件来实现多实例，这种方式定制每个实例的配置不太方面，优点是管理起来很方便，集中管理；



### 同一开发环境下安装多个数据库，必须处理以下问题

- 配置文件安装路径不能相同
- 数据库目录不能相同
- 启动脚本不能同名
- 端口不能相同
- socket文件的生成路径不能相同



## mysql多实例搭建

### mysqld_multi搭建

1. 下载免编译二进制包

```shell
wget http://mirrors.sohu.com/mysql/MySQL-5.7/mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz

tar -zxvf mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz
mv mysql-5.7.23-linux-glibc2.12-x86_64 /usr/local/mysql

# 关闭iptables
service iptables stop  #临时关闭
chkconfig iptables off  #永久关闭

# 关闭selinux
vi /etc/sysconfig/selinux 
SELINUX=DISABLED

# 创建mysql系统用户和组
groupadd -g 27 mysql
useradd -u 27 -g mysql mysql
id mysql

# 创建mysql目录
mkdir -p /data/mysql/mysql_3306/data
mkdir -p /data/mysql/mysql_3306/log
mkdir -p /data/mysql/mysql_3306/tmp
mkdir -p /data/mysql/mysql_3307/data
mkdir -p /data/mysql/mysql_3307/log
mkdir -p /data/mysql/mysql_3307/tmp
mkdir -p /data/mysql/mysql_3308/data
mkdir -p /data/mysql/mysql_3308/log
mkdir -p /data/mysql/mysql_3308/tmp

# 更改目录权限
chown -R mysql:mysql /data/mysql/ 
chown -R mysql:mysql /usr/local/mysql/
# 添加环境变量
echo 'export PATH=$PATH:/usr/local/mysql/bin' >>  /etc/profile 
source /etc/profile

# 复制my.cnf文件到etc目录 会将原来的my.cnf文件删除了
cp /etc/my.cnf /etc/my.cnf.bak
cp /usr/local/mysql/support-files/my-default.cnf /etc/my.cnf
```

2. 修改my.cnf（在一个文件中修改即可）

```shell
vim /etc/my.conf

[client]
port=3306
socket=/tmp/mysql.sock 

[mysqld_multi]
mysqld = /usr/local/mysql/bin/mysqld_safe
mysqladmin = /usr/local/mysql/bin/mysqladmin
log = /data/mysql/mysqld_multi.log 

[mysqld]
basedir = /usr/local/mysql
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES 

#3306数据库

[mysqld3306]
mysqld=mysqld
mysqladmin=mysqladmin
datadir=/data/mysql/mysql_3306/data
port=3306
server_id=3306
socket=/tmp/mysql_3306.sock
log-output=file
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /data/mysql/mysql_3306/log/slow.log
log-error = /data/mysql/mysql_3306/log/error.log
binlog_format = mixed
log-bin = /data/mysql/mysql_3306/log/mysql3306_bin 

#3307数据库

[mysqld3307]
mysqld=mysqld
mysqladmin=mysqladmin
datadir=/data/mysql/mysql_3307/data
port=3307
server_id=3307
socket=/tmp/mysql_3307.sock
log-output=file
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /data/mysql/mysql_3307/log/slow.log
log-error = /data/mysql/mysql_3307/log/error.log
binlog_format = mixed
log-bin = /data/mysql/mysql_3307/log/mysql3307_bin 

#3308数据库

[mysqld3308]
mysqld=mysqld
mysqladmin=mysqladmin
datadir=/data/mysql/mysql_3308/data
port=3308
server_id=3308
socket=/tmp/mysql_3308.sock
log-output=file
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /data/mysql/mysql_3308/log/slow.log
log-error = /data/mysql/mysql_3308/log/error.log
binlog_format = mixed
log-bin = /data/mysql/mysql_3308/log/mysql3308_bin
```

3. 初始化数据库

```shell
# 初始化3306数据库
/usr/local/mysql/scripts/mysql_install_db --basedir=/usr/local/mysql/ --datadir=/data/mysql/mysql_3306/data --defaults-file=/etc/my.cnf  

/usr/local/mysql/scripts/mysql_install_db --basedir=/usr/local/mysql/ --datadir=/data/mysql/mysql_3307/data --defaults-file=/etc/my.cnf

/usr/local/mysql/scripts/mysql_install_db --basedir=/usr/local/mysql/ --datadir=/data/mysql/mysql_3308/data --defaults-file=/etc/my.cnf
```

4. 查看数据库初始化是否成功

```shell
cd /data/mysql/mysql_3306/data/
cd /data/mysql/mysql_3307/data/
cd /data/mysql/mysql_3308/data/
```

5. 设置启动文件

```shell
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
```

6. mysqld_multi进行多实例管理

```shell
启动全部实例：/usr/local/mysql/bin/mysqld_multi start

查看全部实例状态：/usr/local/mysql/bin/mysqld_multi report 

启动单个实例：/usr/local/mysql/bin/mysqld_multi start 3306 

停止单个实例：/usr/local/mysql/bin/mysqld_multi stop 3306 

查看单个实例状态：/usr/local/mysql/bin/mysqld_multi report 3306 

# 启动全部实例
/usr/local/mysql/bin/mysqld_multi start
/usr/local/mysql/bin/mysqld_multi report

# 查看启动进程
ps -aux | grep mysql

# 查看sock文件
cd /tmp
ls mysql*.sock

# 修改密码
mysql -S /tmp/mysql_3306.sock
set password for root@'localhost'=password('xxxxxx');
flush privileges;

# 新建用户及授权
grant ALL PRIVILEGES on *.* to admin@'%' identified by 'xxxxxx';
flush privileges
```



## mysql主从复制

1. 设置master主库的配置文件

```shell
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
配置文件中的[mysqld]这一块中加入
server-id=1  #标注 主库的身份id
log-bin=mysql-bin  #binlog的文件名 

service mysql restart  重新启动mysql

# 查看主库的状态,日志文件的名字及数据起始位置
show master status;
```

2. 创建用于主从数据同步的账号

```sql
create user 'slace_copy'@'%' identified by 'P@ssw0rd';

-- 授予主从同步账号的复制数据权限
grant replication slave on *.* to 'slace_copy'@'%';

--进行数据库的锁表，防止数据写入
flush table with read lock;
```

3. 将数据导出

```
mysqldump -u root -p --all-databases > /data/zhucong.dump

scp /data/zhucong.dump   root@slace_ip:/data/
```

4. 登录slace从库导入主库数据，保持数据一致性

```shell
mysql -uroot -p
source /data/zhucong.dump
```



5. 从库配置

```shell
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
配置文件中的[mysqld]这一块中加入
server-id=2  #标注 从库的身份id

service mysql restart  重新启动mysql

show variables like 'server_id';
show variables like 'log_bin';
```

6. 通过一条命令，开启主从同步

```shell
change master to master_host='192.168.129.128',
master_user='slace_copy',
master_password='P@ssw0rd',
master_log_file='mysql-bin.000001',
master_log_pos=位置;   position值
```

7. 开启从库的slave 同步

```shell
start slave;
-- 查看slave IO、SQL Running Yes
show slave status\G;

-- 如果Slave_SQL_Running：no
stop slave;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
start slave;
show slave status\G;
-- 如果Slave_IO_Running：no
slave stop; 
CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=位置;
start slave;
show slave status\G;
```

8. 解锁主库表

```shell
unlock tables;
```







# 安装Redis

1. 安装Redis

```shell
sudo apt-get -y install redis-server  
redis-cli    可以进入就说安装成功
sudo vim /etc/redis/redis.conf 
# bind-address  = 127.0.0.1   远程连接
# requirepass foobared        Redis 设置密码
protected-mode  no           无密码登录
```

2. 重启、停止和启动 Redis服务

```shell
sudo /etc/init.d/redis-server restart
sudo /etc/init.d/redis-server stop
sudo /etc/init.d/redis-server start
```

3. 源码安装redis

```shell
wget http://download.redis.io/releases/redis-4.0.10.tar.gz

tar -zxvf redis-4.0.10.tar.gz
cd redis-4.0.10
make && make install

vim /opt/redis-4.0.10/redis.conf
daemonize yes   #后台启动
bind 0.0.0.0
port 6380
requirepass P@ssw0rd
# 启动服务
redis-server redis.conf

# 客户端登录密码
redis-cli -p 6380
auth P@ssw0rd
# -a参数登录，密码暴露这终端可能不安全
redis-cli -p 6380 -a P@ssw0rd
```

4. 过滤空白行和注释行

```shell
sudo grep -v "^#" /etc/redis/redis.conf | grep -v "^$"
```

## redis 持久化与发布订阅

### 发布订阅

- 发布者 PUBLISH
  PUBLISH music reidslearn    给频道music发布消息
- 接受者 SUBSCRIBE
  SUBSCRIBE music 订阅music频道
  PSUBSCRIBE 频道*  支持模糊订阅频道
- 频道 music

### 持久化

- RDB持久化(数据快照)

```shell
# rdb.conf
daemonize yes   #后台启动
bind 0.0.0.0
port 6379
requirepass P@ssw0rd
logfile /data/6379/redis.log
dir /data/6379
dbfilename  dump.rdb
save 900 1								#每900秒 有1个修改记录
save 300 10							  #每300秒 有10个修改记录
save 60 10000						#每60秒 有10000个修改记录
```

  

- AOF持久化

```shell
# aof.conf
daemonize yes   #后台启动
bind 0.0.0.0
port 6379
requirepass P@ssw0rd
logfile /data/6379/redis.log
dir /data/6379
  
appendonly yes
appendfsync always #总是修改类的操作
  							everysec #每秒做一次持久化
  							no #依赖于系统自带的缓存大小机制
  
# 查看aof文件变化
tail -f appendonly.aof
```

### redis不重启之rdb数据切换到aof数据

- 准备rdb的redis服务端
  redis-server rdb.conf

- 切换rdb到aof

```shell
redis-cli #登录redis
CONFIG SET appendonly yes  #用命令激活aof持久化(临时生效，注意写到配置文件)
CONFIG SET save ""  #关闭rdb持久化
```

```shell
vim rdb.conf
daemonize yes   #后台启动
bind 0.0.0.0
port 6379
requirepass P@ssw0rd
logfile /data/6379/redis.log
dir /data/6379
#dbfilename  dump.rdb
#save 900 1								#每900秒 有1个修改记录
#save 300 10							  #每300秒 有10个修改记录
#save 60 10000	
  
appendonly yes
appendfsync everysec
```

- 测试aof数据持久化

```shell
kill -9 进程号
# pkill redis-server   根据服务名杀死进程，可以杀死所有有关redis-server
redis-server rdb.conf
```

## redis 主从同步

1. 准备三个.conf 文件

```shell
touch redis-6379.conf
touch redis-6380.conf
touch redis-6381.conf

# redis-6379.conf
daemonize yes   #后台启动
port 6379
pidfile /data/6379/redis.pid
loglevel notice
logfile "/data/6379/redis.log"
dbfilename dump.rdb
dir /data/6379

# sed -i 替换当前文件  s:替换  6379 被替换者 6380 替换为 g:全局 写到>redis-6380.conf
sed "s/6379/6380/g" redis-6379.conf > redis-6380.conf
sed "s/6379/6381/g" redis-6379.conf > redis-6381.conf
```

2. 配置主从关系

```shell
echo "slaveof 127.0.0.1 6379" >> redis-6380.conf
echo "slaveof 127.0.0.1 6379" >> redis-6381.conf
```

3. 开启三个服务

```shell
redis-server redis-6379.conf
redis-server redis-6380.conf
redis-server redis-6381.conf
```

4. 查看info

```shell
redis-cli -p 6379 info [replication]
```

### 手动切换主从关系

```shell
# 杀死6379的主库实例
kill - 9 主库PID

# 手动切换主从身份
    # 登录redis-6380,通过命令，去掉自己的从库身份,等待连接
    slaveof no one

    # 登录redis-6381,通过命令，生成新的主人
    slaveof 127.0.0.1 6380
# 测试新的主从数据同步
```

## redis哨兵 Redis-Sentinel

1. 什么是哨兵呢？保护redis主从集群，正常运转，当主库挂掉之后，自动的在从库中挑选新的主库，进行同步数据
2. 准备三个redis数据库实例
3. 准备三个哨兵配置文件

```shell
touch redis-sentinel-26379.conf
touch redis-sentinel-26380.conf
touch redis-sentinel-26381.conf

# redis-sentinel-26379.conf
daemonize yes   #后台启动
port 26379
dir /var/redis/data/
logfile "26379.log"

sentinel monitor mymater 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000


sed "s/6379/6380/g" redis-sentinel-26379.conf > redis-sentinel-26380.conf
sed "s/6379/6381/g" redis-sentinel-26379.conf > redis-sentinel-26381.conf
```

4. 开启三个哨兵

```shell
reids-sentinel redis-sentinel-26379.conf
reids-sentinel redis-sentinel-26380.conf
reids-sentinel redis-sentinel-26381.conf
```

5. 查看哨兵通信状态

```shell
reids-cli -p 26379 info sentinel
```

6. 杀死一个redis主库，6379节点，等待30s，检查6380和6381的节点状态

```shell
kill 6379 PID
reids-cli -p 6380 info replication
reids-cli -p 6381 info replication
# 如果切换的主从身份之后，（原理就是 更改redis的配置文件，切换主从身份）
```

7. 恢复6379数据库，查看是否将6379添加为新的slave身份

```shell
reids-cli -p 6379 info replication
```

## Redis集群 redis-cluster

1. 环境准备

```shell
# redis-7000.conf

port 7000
daemonize yes
dir "/opt/redis/data"
logfile "7000.log"
dbfilename "dump-7000.rdb"
cluster-enabled yes   #开启集群模式
cluster-config-file nodes-7000.conf　　#集群内部的配置文件
cluster-require-full-coverage no　　#redis cluster需要16384个slot都正常的时候才能对外提供服务，换句话说，只要任何一个slot异常那么整个cluster不对外提供服务。 因此生产环境一般为no
```

2. redis支持多实例的功能，我们在单机演示集群搭建，需要6个实例，三个是主节点，三个是从节点，数量为6个节点才能保证高可用的集群。

```shell
sed "s/7000/7001/g" redis-7000.conf > redis-7001.conf
sed "s/7000/7002/g" redis-7000.conf > redis-7002.conf
sed "s/7000/7003/g" redis-7000.conf > redis-7003.conf
sed "s/7000/7004/g" redis-7000.conf > redis-7004.conf
sed "s/7000/7005/g" redis-7000.conf > redis-7005.conf
```

3. 运行redis实例

```shell
redis-server redis-7000.conf
redis-server redis-7001.conf
redis-server redis-7002.conf
redis-server redis-7003.conf
redis-server redis-7004.conf
redis-server redis-7005.conf

# 检查日志文件
cat 7000.log
# 检查redis服务的端口、进程
ps -ef|grep redis

# 此时集群还不可用，可以通过登录redis查看
redis-cli -p 7000
set hello world

(error)CLUSTERDOWN The cluster is down
```

4. 准备ruby环境

```shell
#下载ruby
wget https://cache.ruby-lang.org/pub/ruby/2.3/ruby-2.3.1.tar.gz

#安装ruby
tar -xvf ruby-2.3.1.tar.gz
./configure --prefix=/opt/ruby/
make && make install

#准备一个ruby命令
#准备一个gem软件包管理命令
#拷贝ruby命令到path下/usr/local/ruby
cp /opt/ruby/bin/ruby /usr/local/
cp bin/gem /usr/local/bin

# 安装ruby gem 包管理工具
wget http://rubygems.org/downloads/redis-3.3.0.gem

gem install -l redis-3.3.0.gem

# 查看gem有哪些包
gem list -- check redis gem

# 安装redis-trib.rb命令
cp /opt/redis/src/redis-trib.rb /usr/local/bin/
```

5. 一键开启redis-cluster集群
   
```shell
#每个主节点，有一个从节点，代表--replicas 1
redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
     
#集群自动分配主从关系  7000、7001、7002为 7003、7004、7005 主动关系
```

6.  查看集群状态

```shell
redis-cli -p 7000 cluster info  

redis-cli -p 7000 cluster nodes  #等同于查看nodes-7000.conf文件节点信息

# 集群主节点状态
redis-cli -p 7000 cluster nodes | grep master
# 集群从节点状态
redis-cli -p 7000 cluster nodes | grep slave

# 安装完毕后，检查集群状态
redis-cli -p 7000 cluster info
```

7. 测试写入集群数据，登录集群必须使用redis-cli -c -p 7000必须加上-c参数

```shell
redis-cli -c -p 7000
set name li
keys *
get name
```



# 安装MongoDB

[官网安装链接]( https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/ "This")

1. 导入包管理系统使用的公钥

```shell
wget -qO - https://www.mongodb.org/static/pgp/server-4.2.asc | sudo apt-key add -
```

2. 为MongoDB创建一个列表文件

```shell
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/4.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.2.list
```

3. 重新加载本地包数据库

```shell
sudo apt-get update
```

4. 安装MongoDB包

```shell
sudo apt-get install -y mongodb-org

# /data/db  chmod 777
mongod --port 27017 -- dbpath /data/db
```

5. mongo创建管理员

```shell
mongo --port 27017
use admin
db.createUser({user:"root",pwd:"root",roles:["userAdminAnyDatabase"]})
```

6. 配置文件设置

```shell
sudo vim /etc/mongod.conf

net:
	port:27017
	bindIp:0.0.0.0

# 无密码登录无需配置
security:
	authorization: enabled
```

7. 重启、停止和启动 MongoDB

```shell
sudo service mongod restart
sudo service mongod stop
sudo service mongod start
```



# Nginx配置

技术栈:

收费版技术栈

apache web+java+tomcat应用服务器+oracle +memcached+redhat企业版+svn(代码版本管理工具)

开源技术栈

nginx(负载均衡)+python(virtualenv)+uwsgi(python应用服务器，启动10个django dfr)+mysql+redis+linu(centos7)+git+vue(前端代码服务器)

1. 安装依赖环境

```shell
yum install gcc patch libffi-devel python-devel  zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel openssl openssl-devel -y
```

2. Nginx安装

```shell
# 1.下载源码包
wget -c https://nginx.org/download/nginx-1.12.0.tar.gz
# 2.解压缩源码
tar -zxvf nginx-1.12.0.tar.gz
# 3.配置，编译安装  开启nginx状态监测功能
./configure --prefix=/opt/nginx1-12/ --with-http_ssl_module --with-http_stub_status_module 
make && make install 
# 4.启动nginx，进入sbin目录,找到nginx启动命令
cd sbin
./nginx #启动
./nginx -s stop #关闭
./nginx -s reload #重新加载
```

3. 安装完成后检测服务

```shell
netstat -tunlp |grep 80
curl -I 127.0.0.1
#如果访问不了，检查selinux，iptables防火墙
getenforce
Disabled
```

4. Nginx目录结构
   - conf 存放nginx所有配置文件的目录,主要nginx.conf
   - html 存放nginx默认站点的目录，如index.html、error.html等
   - logs 存放nginx默认日志的目录，如error.log access.log
   - sbin 存放nginx主命令的目录,sbin/nginx
5. Nginx主配置文件解析

```shell
# CoreModule核心模块

user www;                       #Nginx进程所使用的用户
worker_processes 1;             #Nginx运行的work进程数量(建议与CPU数量一致或auto)
error_log /log/nginx/error.log  #Nginx错误日志存放路径
pid /var/run/nginx.pid          #Nginx服务运行后产生的pid进程号

# events事件模块

events {            
    worker_connections  1024;//每个worker进程支持的最大连接数
    use epool;          //事件驱动模型, epoll默认
}

# http内核模块

//公共的配置定义在http{}
http {  //http层开始
...    
    //使用Server配置网站, 每个Server{}代表一个网站(简称虚拟主机)
    'server' {
        listen       80;        //监听端口, 默认80
        server_name  localhost; //提供服务的域名或主机名
        access_log host.access.log  //访问日志
        //控制网站访问路径
        'location' / {
            root   /usr/share/nginx/html;   //存放网站代码路径
            index  index.html index.htm;    //服务器返回的默认页面文件
        }
        //指定错误代码, 统一定义错误页面, 错误代码重定向到新的Locaiton
        error_page   500 502 503 504  /50x.html;
    }
    ...
    //第二个虚拟主机配置
    'server' {
    ...
    }
    
    include /etc/nginx/conf.d/*.conf;  //包含/etc/nginx/conf.d/目录下所有以.conf结尾的文件

}   //http层结束
```

6. 日志参数

```shell
log_format    记录日志的格式，可定义多种格式
accsss_log    指定日志文件的路径以及格式


 　　log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                      　　'$status $body_bytes_sent "$http_referer" '
                      　　'"$http_user_agent" "$http_x_forwarded_for"';
```

- $remote_addr    记录客户端ip
- $remote_user    远程用户，没有就是 “-”
- $time_local 　　 对应[14/Aug/2018:18:46:52 +0800]
- $request　　　 　对应请求信息"GET /favicon.ico HTTP/1.1"
- $status　　　  　状态码
- $body_bytes_sent　　571字节 请求体的大小
- $http_referer　　对应“-”　　由于是直接输入浏览器就是 -
- $http_user_agent　　客户端身份信息
- $http_x_forwarded_for　　记录客户端的来源真实ip 97.64.34.118

7. Nginx的允许、拒绝访问

```shell
server{
...
location / {
	deny 192.168.1.0/24;
	
	allow 192.168.0.0/24;
}
...
}
```

8. error_page 错误页面优化

```shell
server {
        listen       80;
        server_name  www.pythonav.cn;
        root html/python;
        location /{
            index  index.html index.htm;
        }
　	#在python路径下的40x.html错误页面
        error_page 400 403 404 405 /40x.html;
        }
```

## Nginx反向代理

1. 实验准备2个nginx服务器

   192.168.13.24 代理服务器

   192.168.13.79 web服务器

2. 配置代理服务器192.168.13.24   proxy_pass

```shell
server {
        listen       80;
        server_name  www.pythonav.cn;
        root html/python;
        location /{
        	proxy_pass http://192.168.13.79/;
            index  index.html index.htm;
        }
　	#在python路径下的40x.html错误页面
        error_page 400 403 404 405 /40x.html;
        }
```

## Nginx负载均衡

1. 准备3台计算机

   192.168.13.121  负载均衡器

   192.168.13.79 web服务器

   192.168.13.24 web服务器

2. 配置代理服务器192.168.13.121

```shell
upstream webcluster{
server 192.168.13.79;
server 192.168.13.24;
}


server {
		...
		location /{
        	proxy_pass http://webcluster;
        }
		...
}
```

3. upstream分配策略

   - 轮巡
   - weight 权重

```shell
upstream django {
       server 10.0.0.10:8000 weight=5;
       server 10.0.0.11:9000 weight=10;#这个节点访问比率是大于8000的
}
```

   - ip_hash 不能和

```shell
upstream django {
　　　　ip_hash;
       server 10.0.0.10:8000;
       server 10.0.0.11:9000;
}
```

   - backup  在非backup机器繁忙或者宕机时，请求backup机器，因此机器默认压力最小

```shell
upstream django {
       server 10.0.0.10:8000 weight=5;
       server 10.0.0.11:9000;
       server 10.0.0.12:8080 backup;
}
```



# uWSGI配置

1. 安装uWSGI

```shell
# 进入虚拟环境venv，安装uwsgi
(venv) [root@slave 192.168.11.64 /opt]$pip3 install uwsgi
# 检查uwsgi版本
(venv) [root@slave 192.168.11.64 /opt]$uwsgi --version
2.0.17.1
# 检查uwsgi python版本
uwsgi --python-version
```

2. 运行简单测试

```shell
#启动一个python
uwsgi --http :8000 --wsgi-file test.py

http :8000: 使用http协议，端口8000
wsgi-file test.py: 加载指定的文件，test.py

# test.py
def application(env, start_response):
    start_response('200 OK', [('Content-Type','text/html')])
    return [b"Hello World"] # python3
```

3. 启动django项目

```shell
#mysite/wsgi.py  确保找到这个文件
uwsgi --http :8000 --module mysite.wsgi
module mysite.wsgi: 加载指定的wsgi模块
```

4. uWSGI热加载项目

```shell
在启动命令后面加上参数
uwsgi --http :8000 --module mysite.wsgi --py-autoreload=1 
#发布命令
command= /home/venv/bin/uwsgi --uwsgi 0.0.0.0:8000 --chdir /opt/mysite --home=/home/venv --module mysite.wsgi
#此时修改django代码，uWSGI会自动加载django程序，页面生效
```

5. 配置文件启动uWSGI

```shell
uwsgi支持ini、xml等多种配置方式，本文以 ini 为例， 在/etc/目录下新建uwsgi_nginx.ini，添加如下配置：

# mysite_uwsgi.ini file
[uwsgi]

# Django-related settings
# the base directory (full path)
chdir           = /opt/mysite
# Django's wsgi file
module          = mysite.wsgi
# the virtualenv (full path)
home            = /opt/venv
# process-related settings
# master
master          = true
# maximum number of worker processes
processes       = 1
# the socket (use the full path to be safe
# 必须配合Nginx使用socket
socket          = 0.0.0.0:8000
# 单独使用
http          = 0.0.0.0:8000
# ... with appropriate permissions - may be needed
# chmod-socket    = 664
# clear environment on exit
vacuum          = true
# 后台运行
daemonize = yes
```

6. #### 指定配置文件启动命令

```shell
uwsgi --ini  /etc/uwsgi_nginx.ini
```

7. 配置nginx

```shell
worker_processes  1;
error_log  logs/error.log;
pid        logs/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  logs/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
　　#nginx反向代理uwsgi
    server {
        listen       80;
        server_name  192.168.11.64;
        location / {
　　　　  #nginx自带ngx_http_uwsgi_module模块，起到nginx和uwsgi交互作用
         #通过uwsgi_pass设置服务器地址和协议，讲动态请求转发给uwsgi处理
         include  /opt/nginx1-12/conf/uwsgi_params;
         uwsgi_pass 0.0.0.0:8000;
            root   html;
            index  index.html index.htm;
        }
　　　　  #nginx处理静态页面资源
　　　　  location /static{
　　　　　　　　alias /opt/nginx1-12/static;　　　
         }
　　　　　#nginx处理媒体资源
　　　　　location /media{
　　　　　　　　alias /opt/nginx1-12/media;　　
         }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```





# supervisor 进程管理工具

1. 将linux进程运行在后台的方法有哪些

   - 命令后面加上 & 符号

   - 使用nohup命令
   - 使用进程管理工具supervisor

2. 安装supervisor,使用Python2的包管理工具easy_install

```shell
# apt install python-setuptools  #如果没有easy_install命令
easy_install supervisor
```

3. 通过命令生成supervisor的配支文件

```shell
echo_supervisord_conf > /etc/supervisord.conf
```

4. 再/etc/supervisord.conf末尾添加上如下代码！！！！！！

```shell
[program:my]
#command=/opt/venv/bin/uwsgi --ini  /etc/uwsgi_nginx.ini  #这里是结合virtualenv的命令 和supervisor的精髓！！！！
command= /home/venv/bin/uwsgi --uwsgi 0.0.0.0:8000 --chdir /opt/mysite --home=/home/venv --module mysite.wsgi
#--home指的是虚拟环境目录  --module找到 mysite/wsgi.py
```

```shell
# supervisord.conf配置文件参数解释
# [program:xx]是被管理的进程配置参数，xx是进程的名称
[program:xx]
command=/opt/apache-tomcat-8.0.35/bin/catalina.sh run  ; 程序启动命令
autostart=true       ; 在supervisord启动的时候也自动启动
startsecs=10         ; 启动10秒后没有异常退出，就表示进程正常启动了，默认为1秒
autorestart=true     ; 程序退出后自动重启,可选值：[unexpected,true,false]，默认为unexpected，表示进程意外杀死后才重启
startretries=3       ; 启动失败自动重试次数，默认是3
user=tomcat          ; 用哪个用户启动进程，默认是root
priority=999         ; 进程启动优先级，默认999，值小的优先启动
redirect_stderr=true ; 把stderr重定向到stdout，默认false
stdout_logfile_maxbytes=20MB  ; stdout 日志文件大小，默认50MB
stdout_logfile_backups = 20   ; stdout 日志文件备份数，默认是10
; stdout 日志文件，需要注意当指定目录不存在时无法正常启动，所以需要手动创建目录（supervisord 会自动创建日志文件）
stdout_logfile=/opt/apache-tomcat-8.0.35/logs/catalina.out
stopasgroup=false     ;默认为false,进程被杀死时，是否向这个进程组发送stop信号，包括子进程
killasgroup=false     ;默认为false，向进程组发送kill信号，包括子进程
```

5. 启动supervisord服务端，指定配置文件启动

```shell
supervisord -c /etc/supervisord.conf
```



6.  重新加载supervisor

```shell
一、添加好配置文件后

二、更新新的配置到supervisord    

supervisorctl update
三、重新启动配置中的所有程序

supervisorctl reload
四、启动某个进程(program_name=你配置中写的程序名称)

supervisorctl start program_name
五、查看正在守候的进程

supervisorctl
六、停止某一进程 (program_name=你配置中写的程序名称)

pervisorctl stop program_name
七、重启某一进程 (program_name=你配置中写的程序名称)

supervisorctl restart program_name
八、停止全部进程

supervisorctl stop all
注意：显示用stop停止掉的进程，用reload或者update都不会自动重启。
```



  



# 安装nodejs、npm

1. 安装
```shell
sudo apt-get install nodejs
sudo apt-get install npm
```
2. 升级npm、nodejs
```shell
sudo npm install npm -g  
sudo npm install –g n   
sudo n latest   #(升级node.js到最新版)  
# or $ n stable（升级node.js到最新稳定版）
```

3. 切换淘宝镜像

```shell
sudo npm install -g cnpm --registry=https://registry.npm.taobao.org
```

4. 安装yarn

```shell
sudo apt-get update
sudo apt-get upgrade
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt-get update
sudo apt-get install yarn
# 如果用的是nvm，要这样安装：
# sudo apt install --no-install-recommends yarn
yarn --version

#yarn换淘宝源
yarn config set registry https://registry.npm.taobao.org
```

5. 安装vue

```shell
yarn global add @vue/cli

sudo find / -name "vue"
# /home/li/.yarn/bin/vue

echo '# set PATH for yarn
if [ -d "$HOME/.yarn/bin" ] ; then
    PATH="$HOME/.yarn/bin:$PATH"
fi' >> ~/.profile
. ~/.profile
vue --version
```

# git clone 提速

```shell
nslookup github.global.ssl.fastly.Net
nslookup github.com

sudo vim /etc/hosts
# 添加对应的域名地址
github.com 13.250.177.223
github.global.ssl.fastly.Net 31.13.69.86

# 重启DNS
sudo /etc/init.d/networking restart
```

# 安装 systemback 备份IOS镜像

```shell
#Ubuntu 16.04的Systemback二进制文件与Ubuntu 18.04/18.10兼容，因此我们可以使用以下命令在18.04/18.10上添加Ubuntu 16.04 PPA
sudo add-apt-repository "deb http://ppa.launchpad.net/nemh/systemback/ubuntu xenial main"

#导入此PPA的GPG签名密钥
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 382003C2C8B7B4AB813E915B14E4942973C62A1B

# 更新
sudo apt update

# 安装systemback
sudo apt install systemback
```

## Systemback制作大于4G的Ubuntu系统镜像

```shell
# 解压 .sblive 文件
mkdir sblive
tar -xf /home/systemback_live_2018-10-15.sblive -C sblive

# 重命名syslinux 至 isolinux
mv sblive/syslinux/syslinux.cfg sblive/syslinux/isolinux.cfg
mv sblive/syslinux sblive/isolinux

# 安装cdtools
sudo apt install aria2
aria2c -s 10 https://nchc.dl.sourceforge.net/project/cdrtools/alpha/cdrtools-3.02a07.tar.gz
tar -xvf cdrtools-3.02a07.tar.gz
cd cdrtools-3.02
make
sudo make install

# 生成ISO文件
/opt/schily/bin/mkisofs -iso-level 3 -r -V sblive -cache-inodes -J -l -b isolinux/isolinux.bin -no-emul-boot -boot-load-size 4 -boot-info-table -c isolinux/boot.cat -o sblive.iso sblive
```

# FastDFS文件管理系统

1. 下载FastDFS软件包、依赖库libfastcommon、Nginx、fastdfs-nginx-module
   - https://github.com/happyfish100/FastDFS
   - https://github.com/happyfish100/libfastcommon/releases
   - https://nginx.org/download/
   - https://github.com/happyfish100/fastdfs-nginx-module/releases

2. 安装libfastcommon依赖包

```shell
unzip libfastcommon-master.zip
cd libfastcommon-master
./make.sh
./make.sh install
```

3. 安装FastDFS

```shell
unzip fastdfs-master.zip 
cd fastdfs-master
./make.sh 
./make.sh install
```

4. 配置跟踪服务器tracker

```shell
cp /etc/fdfs/tracker.conf.sample /etc/fdfs/tracker.conf
mkdir -p /home/li/fastdfs/tracker

vim /etc/fdfs/tracker.conf
# 修改base_path
base_path=/home/li/fastdfs/tracker
```

5. 配置存储服务器storage

```shell
cp /etc/fdfs/storage.conf.sample /etc/fdfs/storage.conf
mkdir –p /home/li/fastdfs/storage

vim /etc/fdfs/storage.conf
# 修改base_path、store_path0、tracker_server
base_path=/home/li/fastdfs/storage
store_path0=/home/li/fastdfs/storage
tracker_server=自己ubuntu虚拟机的ip地址:22122
```

6. 启动tracker 和 storage

```shell
sudo service fdfs_trackerd start
sudo service fdfs_storaged start
```

7. 配置client

```shell
cp /etc/fdfs/client.conf.sample /etc/fdfs/client.conf

vim /etc/fdfs/client.conf
# 修改base_path、tracker_server
base_path=/home/li/fastdfs/tracker
tracker_server=自己ubuntu虚拟机的ip地址:22122
```

8. 测试是否可以上传

```shell
fdfs_upload_file /etc/fdfs/client.conf ./下载/photo.jpg
# 输出如下格式
group1/M00/00/00/wKgygF6KuQCAVACRAAQkAzlJUnE969.jpg
```

## 安装nginx及fastdfs-nginx-module

1. 安装依赖库

```shell
sudo apt-get install libpcre3 libpcre3-dev libpcrecpp0v5 libssl-dev zlib1g-dev -y

# 安装gcc g++的依赖库
apt-get install build-essential libtool
# 安装 pcre依赖库
apt-get install libpcre3 libpcre3-dev
# 安装 zlib依赖库
apt-get install zlib1g-dev
# 安装 ssl依赖库
apt-get install openssl
```

2. 解压源码包

```shell
tar -xvf nginx-1.12.2.tar 
unzip fastdfs-nginx-module-master.zip
```

3. 安装nginx

```shell
cd nginx-1.12.2
./configure --prefix=/usr/local/nginx/ --add-module=/home/li/下载/fastdfs-module-master/src

make && make install
```

4. 修改配置文件

```shell
cp fastdfs-nginx-module-master/src/mod_fastdfs.conf /etc/fdfs/

vim /etc/fdfs/mod_fastdfs.conf

connect_timeout=10
tracker_server=自己ubuntu虚拟机的ip地址:22122
url_have_group_name=true
store_path0=/home/li/fastdfs/storage
```

5. 将fastdfs-master/conf下的http.conf、mime.types复制到/etc/fdfs/

```shell
cp http.conf /etc/fdfs/
cp mime.types /etc/fdfs/
```

6. 修改nginx配置文件添加如下

```shell
vim /usr/local/nginx/conf/nginx.conf

server {
            listen       8888;
            server_name  localhost;
            location ~/group[0-9]/ {
                ngx_fastdfs_module;
            }
            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
            root   html;
            }
        }
```

7. 启动nginx

```shell
sudo /usr/local/nginx/sbin/nginx
```

8. python客户端上传测试

```shell
pip install fdfs_client-py-master.zip
```



























