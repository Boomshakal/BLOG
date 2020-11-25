# Ubuntu 18.04 server 
# 修改 apt 软件源为阿里云的源

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


# 关闭sudo密码

```shell
sudo visudo

# %sudo ALL=(ALL:ALL) ALL
%sudo ALL=(ALL:ALL) NOPASSWD:ALL
```



# 修改系统主机名三种方法

- 方法1:修改配置文件
```shell
sudo nano /etc/hosts
```
- 方法2：hostnamectl命令
```shell
sudo hostnamectl set-hostname <newhostname>
```
- 方法3：hostname命令进行临时更改
```shell
sudo hostname <new-hostname>
```

# 文件共享

## NFS

```shell
1. 安装nfs服务端
$ sudo apt install nfs-kernel-server -y
2. 创建目录
$ sudo mkdir -p /mnt/sharedfolder
3. 使任何客户端均可访问
$ sudo chown nobody:nogroup /mnt/sharedfolder/
$ sudo chmod 755 /mnt/sharedfolder
4. 配置/etc/exports文件, 使任何ip均可访问(加入以下语句)
/mnt/sharedfolder *(rw,sync,no_subtree_check)
5. 检查nfs服务的目录
$ sudo exportfs -ra (重新加载配置)
$ sudo showmount -e (查看共享的目录和允许访问的ip段)
6. 重启nfs服务使以上配置生效
$ sudo systemctl restart nfs-kernel-server
7. 测试nfs服务是否成功启动
　　7.1 安装nfs 客户端
　　　　$ sudo apt-get install nfs-common
　　7.2 创建挂载目录
　　　　$ sudo mkdir /mnt/sharedfolder_for_client
   7.3 查看nfs服务的状态是否为active状态:active(exited)或active(runing)
       $ systemctl status nfs-kernel-server
　　7.4 在主机上的Linux中测试是否正常
　　　　$ sudo mount localhost:/mnt/sharedfolder /mnt/sharedfolder_for_client (挂载成功，说明nfs服务正常)
```

## FTP

```shell
sudo apt-get install vsftpd
sudo vim /etc/vsftpd.conf

# 这个是设置是否允许匿名登录ftp服务器，不允许。
anonymous_enable=NO
# 是否允许本机用户登录
local_enable=YES
# 允许上传文件到ftp服务器
write_enable=YES

chroot_local_user=YES
chroot_list_enable=YES
# (default follows) 允许chroot_list文件中配置的用户登录此ftp服务器。
chroot_list_file=/etc/vsftpd.chroot_list

# 配置ftp服务器的上传下载文件所在的目录。
local_root=/home/li/ftpfile

# 给ftp服务器配置使用用户等信息
vim /etc/vsftpd.chroot_list
li

mkdir -p /home/li/ftpfile
# 重启ftp
sudo /etc/init.d/vsftpd restart
```

## samb

```shell
sudo apt-get install samba
sudo apt-get install smbclient
samba -V

sudo vim /etc/samba/smb.conf
[share]
   comment = share folder
   browseable = yes
   path = /home/li/share
   create mask = 0777
   directory mask = 0777
   valid users = li
   force user = nobody
   force group = nogroup
   public = yes
   available = yes
   writable=yes

# 创建文件夹，并修改其权限
mkdir share
chmod 777 share
# samba服务器添加用户
sudo smbpasswd -a li
# 重启samba服务器
sudo /etc/init.d/smbd restart
```

# DNS服务器

```shell
apt install bind9
```

```shell
cd /etc/bind/
vim named.conf.options
options {
        directory "/var/cache/bind";
        listen-on port 53 { 192.168.1.102; 192.168.1.0/24; };
        allow-query     { any; };
        forwarders      { 192.168.1.1; }; #上一层DNS地址(网关或公网DNS)
        recursion yes;
};
vim named.conf.local
zone "host.com" IN {
        type  master;
        file  "host.com.zone";
        allow-update { 192.168.1.102; };
};
# 添加自定义业务域
zone "lhm.com" IN {
        type  master;
        file  "lhm.com.zone";
        allow-update { 192.168.1.102; };
};
cd /var/cache/bind
vim host.com.zone
$ORIGIN host.com.
$TTL 600    ; 10 minutes
@       IN SOA  dns.host.com. dnsadmin.host.com. (
                2020041601 ; serial
                10800      ; refresh (3 hours)
                900        ; retry (15 minutes)
                604800     ; expire (1 week)
                86400      ; minimum (1 day)
                )
            NS   dns.host.com.
$TTL 60 ; 1 minute
dns                A    192.168.1.102
k8s-master         A    192.168.1.157
k8s-node1          A    192.168.1.205
k8s-node2          A    192.168.1.206

vim lhm.com.zone 
$ORIGIN lhm.com.
$TTL 600    ; 10 minutes
@       IN SOA  dns.lhm.com. dnsadmin.lhm.com. (
                2020041601 ; serial
                10800      ; refresh (3 hours)
                900        ; retry (15 minutes)
                604800     ; expire (1 week)
                86400      ; minimum (1 day)
                )
            NS   dns.lhm.com.
$TTL 60 ; 1 minute
dns                A    192.168.1.102

sudo systemctl enable bind9
sudo systemctl restart bind9

named-checkconf
dig -t A k8s-master.host.com @192.168.1.102 +short
192.168.1.157
```



# RAID磁盘阵列

1. 识别组件设备

```shell
# 查看原始磁盘的标识符
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT
```

2. 创建数组

```shell
# RAID0
sudo mdadm --create --verbose /dev/md0 --level=0 --raid-devices=2 /dev/sda /dev/sdb
# 检查/proc/mdstat文件来确保成功创建RAID
cat /proc/mdstat


# RAID1
sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb

# RAID5
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sda /dev/sdb /dev/sdc

# RAID6
sudo mdadm --create --verbose /dev/md0 --level=6 --raid-devices=4 /dev/sda /dev/sdb /dev/sdc /dev/sdd

# RAID10
sudo mdadm --create --verbose /dev/md0 --level=10 --raid-devices=4 /dev/sda /dev/sdb /dev/sdc /dev/sdd
```

3. 创建和挂载文件系统

```shell
# 创建文件系统
sudo mkfs.ext4 -F /dev/md0
# 创建挂载点
sudo mkdir -p /mnt/md0
# 挂载文件系统
sudo mount /dev/md0 /mnt/md0
# 检查新空间是否可用
df -h -x devtmpfs -x tmpfs
```
4. 保存数组布局

```shell
# 为了确保在引导时自动重新组装阵列，我们将不得不调整/etc/mdadm/mdadm.conf文件。您可以通过键入以下内容自动扫描活动数组并附加文件
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
# 之后，您可以更新initramfs或初始RAM文件系统，以便在早期启动过程中阵列可用
sudo update-initramfs -u
# 将新的文件系统挂载选项添加到/etc/fstab文件中以便在引导时自动挂载
echo '/dev/md0 /mnt/md0 ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```

5. 删除RAID

```shell
# 卸载数组
sudo umount /dev/md0
# 停止并删除数组
sudo mdadm --stop /dev/md0

# 清零以删除RAID元数据前,先检查/dev/sd* 名称有无变化
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT

# 清零以删除RAID元数据
sudo mdadm --zero-superblock /dev/sdc
sudo mdadm --zero-superblock /dev/sdd

# 编辑/etc/fstab文件并注释掉或删除对数组的引用
sudo nano /etc/fstab
. . .
# /dev/md0 /mnt/md0 ext4 defaults,nofail,discard 0 0

# 注释掉或从/etc/mdadm/mdadm.conf文件中删除数组定义
sudo nano /etc/mdadm/mdadm.conf
. . .
# ARRAY /dev/md0 metadata=1.2 name=mdadmwrite:0 UUID=7261fb9c:976d0d97:30bc63ce:85e76e91

# initramfs再次更新
sudo update-initramfs -u
```








# 关闭UTC时间同步(双系统)

```shell
timedatectl set-local-rtc 1 --adjust-system-clock
```



# ubuntu修改时间为北京时间

```shell
# 查看当前时区
date
# 修改时区
tzselect
# 复制文件到/etc目录下
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime


网上同步时间
1. 安装ntpdate工具
# sudo apt-get install ntpdate
2. 设置系统时间与网络时间同步
# ntpdate cn.pool.ntp.org
3. 将系统时间写入硬件时间
# hwclock --systohc
```





# 安装**安装 zsh 和 oh-my-zsh**

```shell
cat /etc/shells

# 安装 zsh
apt install zsh

# 将 zsh 设置为系统默认 shell
# sudo chsh -s /bin/zsh
chsh -s $(which zsh)

# 自动安装，如果你没安装 git 需要先安装 git
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | sh

# 或者也可以选择手动安装
git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```
## ZSH 配置

```shell
vim ~/.zshrc

alias cls='clear'
alias ll='ls -l'
alias la='ls -a'
alias vi='vim'
alias grep="grep --color=auto"

# 或者选择 zsh 的主题
# oh-my-zsh 内置了很多主题，对应的主题文件存放在 ~/.oh-my-zsh/themes 目录下，你可以根据自己的喜好选择或者编辑主题。
ZSH_THEME="robbyrussell"
```
## ZSH 插件安装

oh-my-zsh 还支持各种插件，存放在 ~/.oh-my-zsh/plugins 目录下。这里推荐几款：

- autojump：快速切换目录插件

```shell
# 安装
apt install autojump

# 使用
j Document/
```
- zsh-autosuggestions：命令行命令键入时的历史命令建议插件

```shell
# 安装
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```
- zsh-syntax-highlighting：命令行语法高亮插件

```shell
# 安装
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

插件安装好后需要在 ~/.zshrc 文件里配置后方可使用，配置如下：
# 打开 ~/.zshrc 文件，找到如下这行配置代码，在后面追加插件名
plugins=(其他插件名 autojump zsh-autosuggestions zsh-syntax-highlighting)
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



# 修改IP地址

```shell
sudo vim /etc/netplan/00-installer-init.yaml

network:
  ethernets:
    ens33:
      addresses: [192.168.1.205/24]
      gateway4: 192.168.1.1
      dhcp4: no
      optional: true
      nameservers:
        addresses: [8.8.8.8,114.114.114.114]
  version: 2

sudo netplan apply
```





# 配置全局环境变量

查看PATH：echo $PATH
以添加mongodb server为列
修改方法一：
export PATH=/usr/local/mongodb/bin:$PATH
//配置完后可以通过echo $PATH查看配置结果。
生效方法：立即生效
有效期限：临时改变，只能在当前的终端窗口中有效，当前窗口关闭后就会恢复原有的path配置
用户局限：仅对当前用户

 

修改方法二：
通过修改.bashrc文件:
vim ~/.bashrc
//在最后一行添上：
export PATH=/usr/local/mongodb/bin:$PATH
生效方法：（有以下两种）
1、关闭当前终端窗口，重新打开一个新终端窗口就能生效
2、输入“source ~/.bashrc”命令，立即生效
有效期限：永久有效
用户局限：仅对当前用户

 

修改方法三:
通过修改profile文件:
vim /etc/profile
/export PATH //找到设置PATH的行，添加
export PATH=/usr/local/mongodb/bin:$PATH
生效方法：系统重启
有效期限：永久有效
用户局限：对所有用户

```shell
# 添加环境变量
echo 'export PATH=$PATH:/usr/local/mongodb/bin' >>  /etc/profile 
source /etc/profile
```



 

修改方法四:
通过修改environment文件:
vim /etc/environment
在PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games"中加入“:/usr/local/mongodb/bin”
生效方法：系统重启
有效期限：永久有效
用户局限：对所有用户



## 创建软链接

```shell
ln -s /usr/local/python3/bin/python3.7 /usr/bin/python3

ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3
```





# scp传输文件

scp [参数] <源地址 (用户名@IP地址或主机名)>:<文件路径> <目的地址（用户名 @IP 地址或主机名）>:<文件路径>

参数:

-r 传输文件夹

-v 展示传输详情
```shell
scp /home/soft/jdk-7u55-linux-i586.tar.gz root@192.168.132.132:/

scp -r /home/soft root@192.168.132.132:/
```

## Permission denied

```shell
vim /etc/ssh/sshd_config
# PermitRootLogin no / without-password
PermitRootLogin yes
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


# rmp格式安装包转deb

```shell
# 添加 Universe 仓库（如果未添加）
sudo add-apt-repository universe

# 更新
sudo apt update

# 安装 Alien
sudo apt install alien

# 将.rpm 包转换为.deb 包
#（当前目录下会生成一个 deb 安装包，比如: XMind-2020.deb）
sudo alien XMind-2020.rpm

# 安装
sudo dpkg -i XMind-2020.deb
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
rmvirtualenv   删除虚拟环境
```

# pip 国内源

```shell
pip install xxx -i https://pypi.douban.com/simple/
pip install -r requirements.txt -i https://pypi.douban.com/simple/
```

```shell
mkdir ~/.pip
vim ~/.pip/pip.conf
# 然后将下面这两行复制进去就好了
[global]
index-url = https://mirrors.aliyun.com/pypi/simple
```

国内其他pip源

  清华：https://pypi.tuna.tsinghua.edu.cn/simple
  中国科技大学 https://pypi.mirrors.ustc.edu.cn/simple/
  华中理工大学：http://pypi.hustunique.com/
  山东理工大学：http://pypi.sdutlinux.org/
  豆瓣：http://pypi.douban.com/simple/

注意：不管你用的是pip3还是pip，方法都是一样的，都是创建pip文件夹。



# Snap包安装

```shell
sudo snap install picgo.snap --dangerous
```



# 命令后台运行

### 后台运行

这种命令要满足1.要运行一段时间2.不需要与用户交互

命令在后台运行
命令 &
这种会绑定终端，终端一关，进程结束

ctrl+Z 放到后台暂停

### 让命令在后台持久运行

将命令放到/etc/rc.local中，系统启动时执行里面命令，因不是终端启动所以不受影响

 

脱离终端，关了终端也可运行
nohup 命令 &





# 查看端口
```shell
ps -ef|grep 8000
netstat -tunlp|grep 8000
# netstat -apn | grep 8000
kill -9 4438
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

sudo /etc/init.d/networking restart

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

8. 增删改查

```sql
show databases; #查看已有数据库
use dataName; #选择数据库，如果不存在库，则会自动创建。
show tables; # 查看已有的表
show collections # 同上,
db.createCollection('表名');#建表
db.表名.drop(); #删除表

注:table在mongodb里叫collections
```

> 增加数据，语法: db.collectionName.isnert(document)
>
> 删除数据，语法: db.collection.remove(查询表达式, 选项)。选项是指需要删除的文档数，{0/1}，默认是0，删除全部文档
>
> 修改数据，语法: db.collection.update(查询表达式,新值);
>
> 查找数据，语法: db.collection.find(查询表达式,查询的列)。





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

```shell
apt install gcc openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev make
```



2. Nginx安装

```shell
# 1.下载源码包
wget -c https://nginx.org/download/nginx-1.12.0.tar.gz
# 2.解压缩源码
tar -zxvf nginx-1.12.0.tar.gz
# 3.配置，编译安装  开启nginx状态监测功能
./configure --prefix=/opt/nginx1-12/ --with-http_ssl_module --with-http_stub_status_module 
### --with-stream 反向代理TCP
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
# 如果访问不了，检查selinux，iptables防火墙
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

supervisorctl stop program_name
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



# Ubuntu搭建KMS服务器

1. 在任意环境中，下载最新的vlmcsd releases版本，[下载地址](https://github.com/Wind4/vlmcsd/releases)。如在linux中，可以使用wget下载

```shell
wget https://github.com/Wind4/vlmcsd/releases/download/svn1111/binaries.tar.gz
```

2. 解压安装包

```shell
tar -zxvf binaries.tar.gz
```

3. 运行脚本

```shell
cd binaries/Linux/intel/static/
./vlmcsd-x64-musl-static

# 查看1688端口
netstat -lnp|grep 1688
ps aux | grep vlmcsd
```

4. 激活windows系统

```shell
slmgr /skms 你的Ubuntu的IP地址（Ubuntu中使用ifconfig查看IP地址）
slmgr /ato

# 验证激活
slmgr.vbs -dlv
```



# Ubuntu 安装Docker

## 1. 安装Docker
```shell
wget -qO- https://get.docker.com/ | sh
# 国内镜像
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
# 查看dockers版本
docker --version
# 给普通用户添加权限
sudo usermod -aG docker $USER
```

## 2. [Docker 加速器](https://www.daocloud.io/mirror#accelerator-doc)

```shell
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s https://b33dfgq9.mirror.aliyuncs.com
```

## 3. 安装 `docker-compose`

```shell
sudo apt install docker-compose
docker-compose --version
```

## 4. [Docker图形化工具](https://www.cnblogs.com/reasonzzy/p/11377324.html)

```shell
docker pull portainer/portainer && \
docker run -d --name portainerUI -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock --restart=always portainer/portainer
```



## 5. [相关命令](https://www.jianshu.com/p/ca1623ac7723)

```shell
# 进入容器
docker exec -it redis-test /bin/bash
```

## 6. docker常用命令
   1. 镜像相关

- docker search java：在Docker Hub（或阿里镜像）仓库中搜索关键字（如java）的镜像

- docker pull java:8：从仓库中下载镜像，若要指定版本，则要在冒号后指定

- docker images：列出已经下载的镜像

- docker rmi java：删除本地镜像

- docker rmi -f \`docker images -q\`  删除所有镜像

- docker tag 355627632b2 li/ubuntu:v1  设置标签

- docker save -o /home/user/images/ubuntu_14.04.tar ubuntu:14.04 ：导出镜像

- docker load --input ubuntu_14.04.tar ：导入镜像

- docker build -t image_name:version Dockerfile_path ：构建镜像

2. 容器相关

- docker run  -d -p 91:80 nginx ：在后台运行nginx，若没有镜像则先下载，并将容器的80端口映射为宿主机的91端口。
  
  - -it: 交互式终端
  - -d：守护式daemon后台运行
  - -P：随机端口映射
  - -p：指定端口映射
  - -net：网络模式
  - --name : 容器名称
  - --rm : 交互式容器退出时自动删除容器
  - --restart=no - 容器退出时，不重启容器；
  
  ​    				    on-failure - 只有在非0状态退出时才从新启动容器；
  
  ​    				    always - 无论退出状态是如何，都重启容器；
  
- docker ps：列出运行中的容器

- docker ps -a ：列出所有的容器

- docker stop 容器id：停止容器

- docker kill 容器id：强制停止容器

- docker start 容器id：启动已停止的容器

- docker inspect 容器id：查看容器的所有信息

- docker container logs 容器id：查看容器日志

- docker top 容器id：查看容器里的进程

- docker exec -it 容器id /bin/bash：进入容器

- docker attach 容器id ：进入容器(ctrl + P,Q) 切换后台

- exit：退出容器

- docker logs -tf --tail 10 容器id：查看容器日志

- docker rm 容器id：删除已停止的容器

- docker rm -f 容器id：删除正在运行的容器

- docker rm -f \`docker ps -a  -q\`  删除所有容器

3. 容器的网络访问

```shell
# 指定映射(docker 会自动添加一条iptables规则来实现端口映射)
    -p hostPort:containerPort
    -p ip:hostPort:containerPort 
    -p ip::containerPort(随机端口)
    -p hostPort:containerPort/udp
    -p 81:80 –p 443:443
    
# 随机映射
    docker run -P 80（随机端口）
    
# 查看容器IP    
docker inspect --format='{{.NetworkSettings.IPAddress}}' ID/NAMES
```

4. 容器持久化存储——volume卷管理

```shell
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-
docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH
```

```shell
docker run -d -p 8083:80 --name "http8083" -v /opt/Volume/httpd:/usr/local/apache2/htdocs httpd
docker run -d -p 8084:80 --name "http8084" -v /opt/Volume/httpd:/usr/local/apache2/htdocs httpd

# 数据卷容器
docker run -it  --name "httpd_volumes" \
-v /opt/Volume/httpd_volume/conf:/usr/local/apache2/conf \
-v /opt/Volume/httpd_volume/html:/usr/local/apache2/htdocs \
centos:6.9 /bin/bash

# 拷贝数据到数据卷中
/opt/Volume/httpd_volume/html
/opt/Volume/httpd_volume/conf
docker  cp  DOCKERNAME:/opt/a.txt  /opt
# 使用数据卷容器
docker run -d  -p 8085:80 --volumes-from  httpd_volumes --name "http8085"  httpd
docker run -d  -p 8086:80 --volumes-from  httpd_volumes --name "http8086"  httpd

# 使用数据卷容器进行备份
docker run --volumes-from  httpd_volumes --name "httpd_volumesbak" --rm  -v /backup:/backup:rw  centos:6.9   tar cvf /backup/conf.tar /usr/local/apache2/conf

docker run --volumes-from  centosv1 --name "centosrestore" --rm  -v /backup:/backup:rw  centos   tar xvf  /backup/conf.tar
```

5. Docker镜像制作

- 基于容器制作镜像

```shell
mkdir -p /var/ftp/centos6.9
mkdir -p /var/ftp/centos7.5

mount -o loop /mnt/CentOS-6.9-x86_64-bin-DVD1.iso /var/ftp/centos6.9
```

```shell
docker run -it --name "li_sshv1" centos:6.9 /bin/bash
mv /etc/yum.repos.d/*.repo /tmp
 echo -e "[ftp]\nname=ftp\nbaseurl=ftp://172.17.0.1/centos6\ngpgcheck=0">/etc/yum.repos.d/ftp.repo
yum makecache fast && yum install openssh-server -y
/etc/init.d/sshd start     ----->重要:ssh第一次启动时,需要生成秘钥,生成pam验证配置文件
/etc/init.d/sshd stop
echo "123456" | passwd --stdin root

docker commit li_sshv1 li/sshd:v1

"hang" 运行sshd,并丢到后台
/usr/sbin/sshd -D
docker run -d --name=sshd li/sshd /usr/sbin/sshd -D
```

- 基于Dockerfile构建简易镜像

```shell
FROM
	Syntax:
		FROM <repo>:[:<tag>]
		or
		FROM  <repo>@<ImageID>
              
LABEL
	Syntax：
		LABEL DEV="li <362169885@qq.com>"

RUN
	Syntax：
		mv /etc/yum.repos.d/*.repo /tmp && echo -e "[ftp]\nname=ftp\nbaseurl=ftp://172.17.0.1/centos6\ngpgcheck=0">/etc/yum.repos.d/ftp.repo && yum makecache fast && yum install openssh-server -y
		["myusqld","--initialize-insecure","--user=mysql","--basedir=/usr/local/musql","--datadir=/data/mysql/data"]

EXPOSE 
	Syntax：
		EXPOSE 22
CMD 
	Syntax：
		["/usr/sbin/sshd","-D"]
COPY
	Syntax：
		<src> ....<dest>
ADD # 比COPY可以自动解压.tar* 的软件包到目标目录下
	Syntax：
		<src> ....<dest>
		url <dest>

VOLUME 挂载卷                           
WORKDIR 工作路径
 
ENV 设定变量
ENV CODEDIR /var/www/html
ENV DATADIR /data/mysql/data
 
VOLUME ["${CODEDIR}","${DATADIR}"]
 
ENTRYPOINT ["<executeable>","<param1>","<param2>",...]
# 防止docker run command 替换CMD命令
```

```shell
# 基于的基础镜像
FROM python:3.6.9
# 维护者信息
MAINTAINER LHM  lhm@ikahe.com
# 代码添加到code文件夹
ADD ./docker_test /code
# 设置code文件夹是工作目录
WORKDIR /code
# 安装支持
RUN pip install -r requirements.txt
CMD ["python", "/code/server.py"]
```

- 基于 docker-compose 构建

https://github.com/Boomshakal/Django/tree/master/eshop_docker

```shell
# 如何在docker-compose.yml文件中command执行多条命令
command: python3 manage.py migrate && python3 manage.py runserver 0.0.0.0:8000

command:
  - /bin/sh
  - -c
  - |
    python3 manage.py migrate
    # ...随意添加任意脚本...
    python3 manage.py runserver 0.0.0.0:8000
```

## 7. 网络模型

- Docker本地网络类型

```shell
docker network ls

各类网络类型
 docker run --network=type
none : 无网络模式
bridge ： 默认模式，相当于NAT
host : 公用宿主机Network NameSapce
container：与其他容器公用Network Namespace
```

- Docker跨主机网络介绍
- Docker跨主机访问-macvlan实现

```shell
docker network create --driver macvlan --subnet=10.0.0.0/24 --gateway=10.0.0.254 -o parent=eth0 macvlan_1

ip link set eth0 promsic on (ubuntu或其他版本需要)

docker run -it --network macvlan_1 --ip 10.0.0.10 centos:6.9 /bin/bash
docker run -it --network macvlan_1 --ip 10.0.0.11 centos:6.9 /bin/bash
```

- Docker 跨主机访问-overlay实现

```shell
# 启用consul服务，实现网络的统一配置管理
docker run -d -p 8500:8500 -h consul --name consul progrium/consul -server -bootstrap

consul：kv类型的存储数据库（key:value）
docker01、02上：
vim  /etc/docker/daemon.json
{
  "hosts":["tcp://0.0.0.0:2376","unix:///var/run/docker.sock"],
  "cluster-store": "consul://10.0.0.100:8500",
  "cluster-advertise": "10.0.0.100:2376"
}

vim /etc/docker/daemon.json 
vim /usr/lib/systemd/system/docker.service
systemctl daemon-reload 
systemctl restart docker

2）创建overlay网络
docker network create -d overlay --subnet 172.16.0.0/24 --gateway 172.16.0.254  ol1

3）启动容器测试
docker run -it --network ol1 --name oldboy01  busybox /bin/bash
每个容器有两块网卡,eth0实现容器间的通讯,eth1实现容器访问外网
```










## 搭建Nginx

```shell
docker pull nginx

mkdir -p /data/nginx/{conf,conf.d,html,logs}

docker run --name mynginx -d -p 80:80  \
-v /data/nginx/conf/nginx.conf:/etc/nginx/nginx.conf  \
-v /data/nginx/logs:/var/log/nginx \
-d nginx
```

[nginx.conf](https://trac.nginx.org/nginx/export/HEAD/nginx/conf/nginx.conf)官方下载




## 搭建Mysql

```shell
docker pull mysql
docker run -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=root -d mysql

# 建立目录映射
docker run -p 3306:3306 --name mysql \
-v /usr/local/docker/mysql/conf:/etc/mysql \
-v /usr/local/docker/mysql/logs:/var/log/mysql \
-v /usr/local/docker/mysql/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql
```

## 搭建Redis

```shell
# pull镜像
docker pull redis

# 在/usr/local目录下创建docker目录
mkdir /usr/local/docker
cd /usr/local/docker
# 再在docker目录下创建redis目录
mkdir redis&&cd redis
# 创建配置文件，并将官网redis.conf文件配置复制下来进行修改
touch redis.conf
# 创建数据存储目录data
mkidr data
```

[redis.conf](http://download.redis.io/redis-stable/redis.conf)官网下载地址

```shell
修改启动默认配置(从上至下依次)：

bind 127.0.0.1 #注释掉这部分，这是限制redis只能本地访问
protected-mode no #默认yes，开启保护模式，限制为本地访问
daemonize no#默认no，改为yes意为以守护进程方式启动，可后台运行，除非kill进程，改为yes会使配置文件方式启动redis失败
databases 16 #数据库个数（可选），我修改了这个只是查看是否生效。。
dir  ./ #输入本地redis数据库存放文件夹（可选）
appendonly yes #redis持久化（可选）
requirepass  密码 #配置redis访问密码
```

```shell
# 创建并启动redis容器
docker run -p 6379:6379 --name redis \
-v /usr/local/docker/redis/redis.conf:/etc/redis/redis.conf \
-v /usr/local/docker/redis/data:/data \
-d redis redis-server /etc/redis/redis.conf --appendonly yes
```



## 搭建MongoDB

```shell
docker pull mongo

# --auth 需要密码才能访问容器服务
docker run -itd --name mongo -p 27017:27017 mongo --auth

docker exec -it mongo mongo admin
>  db.createUser({ user:'admin',pwd:'123456',roles:[ { role:'userAdminAnyDatabase', db: 'admin'}]});
> db.auth('admin', '123456')
```





## 搭建Sentry

```shell
# 下载Sentry
git clone https://github.com/getsentry/onpremise.git
# 创建目录
mkdir -p data/{sentry,postgres} 
# 获取项目的 key
docker-compose run --rm web config generate-secret-key
# 复制获取到的 key 字符串
vim docker-compose.yml
SENTRY_SECRET_KEY
# 创建项目的 superuser
docker-compose run --rm web upgrade
# 开启 sentry 服务
docker-compose up -d 
```

## 搭建elasticsearch

```shell
docker pull elasticsearch:版本号
# 不选择版本就是最新的

docker run --name=test_es -d -p 9200:9200 -p 9300:9300 docker.io/elasticsearch

# 查看容器运行状态
docker ps 
docker ps -a # 查看所有的容器
# 未启动查看 logs
docker logs test_es

## 发现虚拟机内存太小了
# OpenJDK 64-Bit Server VM warning: INFO: os::commit_memory(0x0000000094cc0000, 1798569984, 0) failed; 
# error='Cannot allocate memory' (errno=12)
#
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (mmap) failed to map 1798569984 bytes for committing reserved memory.
# An error report file with more information is saved as:
# /tmp/hs_err_pid1.log

# 启动test_es 容器
docker start test_es

# 删除容器
docker rm test_es
```

## 搭建Plex Media Player

[PLEX_CLAIM获取地址](https://www.plex.tv/claim)

```shell
docker run \
-d \
--name plex \
-p 32400:32400/tcp \
-e PUID=1000 \
-e PGID=1000 \
-e TZ="Asia/Shanghai" \
-e PLEX_CLAIM="claim-gnc6azt4qdYs_pwNJBUy" \
-v database:/config \
-v transcode/temp:/transcode \
-v /srv/dev-disk-by-label-RAID5/media:/data \
--net=bridge \
--device=/dev/dri:/dev/dri \
plexinc/pms-docker
```

## 加密私有仓库registry

```shell
docker pull registry
sudo apt-get install apache2-utils

# 添加insecure-registries
vim /etc/docker/daemon.json 
{   ...
   "insecure-registries": [
      "192.168.1.159:5000"
    ]
}

systemctl daemon-reload && systemctl restart docker

mkdir certs
# 私有仓库生成证书和key
openssl req  \
-newkey rsa:4096 -nodes -sha256 -keyout certs/harbor.lhm.com.key \
-x509 -days 365 -out certs/harbor.lhm.com.crt

Country Name (2 letter code) [AU]:cn
State or Province Name (full name) [Some-State]:zhejiang
Locality Name (eg, city) []:taizhou
Organization Name (eg, company) [Internet Widgits Pty Ltd]:ikahe
Organizational Unit Name (eg, section) []:linux
Common Name (e.g. server FQDN or YOUR name) []:ikahe.com
Email Address []:lhm@ikahe.net

cp ~/certs/ikahe.com.crt /etc/docker/certs.d/ikahe.com/ca.crt

docker run -d \
--restart=always \
--name registry \
-v "$(pwd)"/certs:/certs \
-e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/ikahe.com.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/ikahe.com.key \
-p 443:443 \
-v /opt/registry:/var/lib/registry \
registry

docker tag nginx:latest ikahe.com/nginx:latest
docker images
docker push ikahe.com/nginx:latest

# scp ca.crt到192.168.1.148主机
scp /etc/docker/certs.d/ikahe.com/ca.crt root@192.168.1.148:/etc/docker/certs.d/ikahe.com/

# 192.168.1.148主机
vim /etc/hosts
docker pull ikahe.com/nginx:latest
docker images

# 证书&加密
mkdir auth
htpasswd -Bbn admin 123456 >> auth/htpasswd

docker run -d \
--restart=always \
--name registry \
-v "$(pwd)"/certs:/certs \
-e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/ikahe.com.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/ikahe.com.key \
-p 443:443 \
-v /opt/registry:/var/lib/registry \
-v "$(pwd)"/auth:/auth \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
registry

# 登录
docker login
Login Succeeded
# 查看登录信息
cat ~/.docker/config.json
{
        "auths": {
                "https://index.docker.io/v1/": {}
        },
        "HttpHeaders": {
                "User-Agent": "Docker-Client/19.03.12 (linux)"
        },
        "credsStore": "desktop.exe"
}

docker tag hello-world:latest ikahe.com/hello-world:v1
docker push ikahe.com/hello-world:v1
docker pull ikahe.com/hello-world:v1
```

## docker企业级镜像仓库harbor

https://github.com/goharbor/harbor/releases

```shell
第一步：安装docker和docker-compose
第二步：下载harbor-offline-installer-v1.x.x.tgz
第三步：上传到/opt,并解压
第四步：修改harbor.cfg配置文件
hostname = 10.0.0.101
harbor_admin_password = 123456
第五步：执行install.sh
```







## 项目部署

[http://www.jayden5.cn/2020/01/04/docker-%E4%BD%BF%E7%94%A8/](http://www.jayden5.cn/2020/01/04/docker-使用/)

```shell
docker run -it -p 8080:80 --name mysite3-nginx \
 -v /home/li/mysite3/static:/usr/share/nginx/html/static \
 -v /home/li/mysite3/media:/usr/share/nginx/html/media \
 -v /home/li/mysite3/compose/nginx/log:/var/log/nginx \
 -d mynginx:v1
```



# K8S

## 系统检查

```shell
# 修改主机名称
hostnamectl set-hostname k8s-master
# 修改hosts文件
echo "192.168.150.59 k8s-master" >> /etc/hosts
# 查看ufw状态
ufw status
# 临时关闭swap
swapoff -a
# 永久关闭swap
vi /etc/fstab
# /swap.img     none    swap    sw      0       0
free -m
# 同时调整k8s的swappiness参数
echo "vm.swappiness=0" >> /etc/sysctl.d/k8s.conf
sysctl -p /etc/sysctl.d/k8s.conf
```

## docker-ce

```shell
# 安装必要的一些系统工具
apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
# 写入软件源信息
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# 更新并安装 Docker-CE
apt-get -y update
apt-get -y install docker-ce
# 配置docker-hub源
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s https://b33dfgq9.mirror.aliyuncs.com
# 修改进程隔离工具
# docker默认cgroupfs  k8s使用systemd
vim /etc/docker/daemon.json
{
...
	"exec-opts": [ "native.cgroupdriver=systemd" ]
}
# 重启docker
systemctl daemon-reload && systemctl restart docker
```



## kubeadm

```shell
# 安装https
apt-get update && apt-get install -y apt-transport-https
# 添加密钥
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
# 添加源
add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"
# 更新源
apt-get update
# 查看1.15的最新版本
apt-cache madison kubelet kubectl kubeadm |grep '1.15.12-00'
# 安装指定的版本
apt install -y kubelet=1.15.12-00 kubectl=1.15.12-00 kubeadm=1.15.12-00
# 查看kubelet版本
kubectl version --client=true -o yaml
# 配置kubelet禁用swap
echo 'KUBELET_EXTRA_ARGS="--fail-swap-on=false"' > /etc/default/kubelet
# 重启kubelet
systemctl daemon-reload && systemctl restart kubelet
# 查看kubelet服务启动状态(因缺少缺少很多参数配置文件,需要等待kubeadm init 后生成)
journalctl -u kubelet -f
```

至此，单个虚拟机配置完毕，接下来会clone一个虚拟机来配置集群环境

## 初始化k8s

```shell
kubeadm init \
  --image-repository registry.aliyuncs.com/google_containers \
  --pod-network-cidr=10.244.0.0/16 \
  --ignore-preflight-errors=Swap
  
# 加入nodes
kubeadm join 你的IP地址:6443 --token 你的TOKEN --discovery-token-ca-cert-hash sha256:你的CA证书哈希

token. 通过命令Kubeadm token list找回
ca-cert. 执行命令openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'找回

# 如果token失效(24小时有效期)，重新生成token
kubeadm token create --print-join-command

# 移除nodes
kubectl get node
kubectl get pods -o wide

# 封锁node，排干node1上的pod
kubectl drain k8s-node1 --delete-local-data --force --ignore-daemonsets
# 删除node1节点
kubectl delete node k8s-node1
kubectl get pods -o wide

# 在k8s-node1节点
kubeadm reset
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
```



## kubectl配置调用

```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 如果The connection to the server localhost:8080 was refused - did you specify the right host or port?
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile

kubectl get pods --all-namespaces
```

## k8s网络flannel

https://github.com/coreos/flannel

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl get pods -A
# 如果拉取不下来，自己pull就可以
grep -i image kube-flannel.yml
docker pull quay.io/coreos/flannel:v0.12.0-amd64
```

## dashboard

[最新版dashboard](##kubernetes-dashboard)

```shell
kubectl get nodes

# 下载dashboard.yaml
wget https://k8s-1252147235.cos.ap-chengdu.myqcloud.com/dashboard/dashboard.yaml
# 拉取镜像
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1
# 创建服务
kubectl apply -f dashboard.yaml
# 官方推荐
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml

kubectl get pods -A
kubectl get pod -n kube-system
kubectl get pod,svc -n kube-system
kubectl get namespaces
kubectl delete -f dashboard.yaml
```

## 解决kubernetes-dashboard本地打开的问题

https://www.cnblogs.com/rainingnight/p/deploying-k8s-dashboard-ui.html

```shell
kubectl cluster-info
# 访问地址//请注意namespace
https://<master-ip>:<apiserver-port>/api/v1/namespaces/xxxxxxxx/services/https:kubernetes-dashboard:/proxy/
```
- 创建服务账号

```shell
#vim admin-user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

kubectl create -f admin-user.yaml
```
- 绑定角色

```shell
#vim admin-user-role-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
  
kubectl create -f  admin-user-role-binding.yaml
```

- 获取Token

```shell
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

## 制作证书

k8s默认启用了RBAC，并为未认证用户赋予了一个默认的身份：anonymous

```shell
我们使用client-certificate-data和client-key-data生成一个p12文件，可使用下列命令：
# 生成client-certificate-data
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt

# 生成client-key-data
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key

# 生成p12
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```

## 访问

https://<HOST_IP>:30001

进去，输入token即可进入,注意：token的值一行，不要分行

## 单节点k8s,默认pod不被调度在master节点

```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## 集成Heapster-[InfluxDB](https://github.com/kubernetes-retired/heapster/blob/master/docs/influxdb.md)

Heapster是容器集群监控和性能分析工具，天然的支持Kubernetes和CoreOS。

https://github.com/kubernetes-retired/heapster/archive/v1.6.0-beta.1.zip

```shell
git clone https://github.com/kubernetes-retired/heapster.git
kubectl create -f deploy/kube-config/influxdb/
kubectl create -f deploy/kube-config/rbac/heapster-rbac.yaml

kubectl get pods --namespace=kube-system
```

## kubectl get cs Unhealthy   

https://llovewxm1314.blog.csdn.net/article/details/108458197

```shell
vim /etc/kubernetes/manifests/kube-controller-manager.yaml
...
 27   #  - --port=0
vim /etc/kubernetes/manifests/kube-scheduler.yam
...
 19   #  - --port=0
 
systemctl restart kubelet.service

# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
```



## Pod.yaml

pod-infrastructure

node：（所有node节点）

```shell
vim /etc/kubernetes/kubelet
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=10.0.0.11:5000/oldguo/pod-infrastructure:latest"

systemctl restart kubelet.service
```

```shell
kubectl create -f
kubectl get pod
kubectl get pod -o wide
kubectl get pods -o wide -l app=web
kubectl descrie pod
kubectl delete pod nginx
kubectl replace --force -f k8s_pod.yaml
```

```shell
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: 10.0.0.11:5000/oldguo/nginx:v1
      ports:
        - containerPort: 80
```

```shell
kubectl create -f k8s_pod.yml
kubectl delete pod nginx
```

## RC.yml 

```shell
# k8s_nginx_rc.yml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: 10.0.0.11:5000/oldguo/nginx:v1
        ports:
        - containerPort: 80   
```

```shell
kubectl create -f k8s_nginx_rc.yml
kubectl get  rc
kubectl delete   rc nginx
kubectl replace  -f k8s_nginx_rc.yml
kubectl scale rc RCNAME --replicas=4

滚动升级及回滚：
cp k8s_nginx_rc.yml k8s_nginx2_rc.yml
kubectl rolling-update nginx -f k8s_nginx2_rc.yml  --update-period=10s
注：
在升级过程中，可以进行回退。
# kubectl rolling-update OLDRCNAME NEWRCNAME --rollback 
如果升级完成，则不可以，使用这条指令进行回退。
# kubectl rolling-update OLDRCNAME -f  RCFILE --update-period=10s 
```



## Deployment.yaml

```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inspect-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inspect-server
  template:
    metadata:
      labels:
        app: inspect-server
    spec:
      containers:
        - name: inspect-server
          image: tcp_server:laster
          imagePullPolicy: Never
          ports:
            - containerPort: 9999
```

```shell
vim  k8s_nginx_dev.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: 10.0.0.11:5000/oldguo/nginx:v2
        ports:
        - containerPort: 80
```

```shell
kubectl create -f k8s_nginx_dev.yml
kubectl get deployment

# deployment滚动升级
kubectl set image deployment/nginx nginx=10.0.0.11:5000/oldguo/nginx:v1
kubectl rollout undo deployment/nginx
```

## HPA弹性伸缩

```shell
# 实现自动pod伸缩
kubectl autoscale deployment nginx --min=2 --max=6 --cpu-percent=80

# horizontalpodautoscalers
kubectl get horizontalpodautoscalers 
kubectl edit horizontalpodautoscalers nginx
```



## Server.yaml

```shell
# server.yaml
cat > inspect_svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: inspect-server
spec:
  ports:
    - port: 9999
      protocol: TCP
      targetPort: 9999
      nodePort: 30000
  selector:
    app: inspect-server
  type: NodePort
EOF
```

```shell
kubectl create -f server.yml
```



## pvc/pv卷持久化

```shell
vim nfs_pv_data.yml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001
  labels:
    type: nfs001
spec:
  capacity:
    storage: 10Gi 
  accessModes:
    - ReadWriteMany 
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: "/data"
    server: 10.0.0.11
    readOnly: false
```

```shell
cat nfs_pvc_mysql.yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-mysql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

```shell
kubectl patch pvc data-mysql-0 -p '{"metadata":{"finalizers": null}}' -n mt-math

kubectl patch pv pv-nfs-mysql01 -p '{"metadata":{"finalizers":null}}'
```

## 卸载k8s

```shell
kubeadm reset
```

## kubernetes-dashboard

[kubernetes](https://github.com/kubernetes)/**[dashboard](https://github.com/kubernetes/dashboard)**

```shell
# 最新dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml

# 后台开启proxy模式
nohup kubectl proxy --address=10.4.7.21 --disable-filter=true &

# 修改为type:NodePort
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
spec:
  type: NodePort
  ports:
	  ...
      nodePort: 30001

# 创建dashboard-admin
vim k8s-admin.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
  
# 获取token值
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
# kubectl describe secret dashboard-admin-token -n kube-system
```

### 汉化

```shell
spec:
      containers:
      - args:
        - --auto-generate-certificates
        - --namespace=kubernetes-dashboard
        env:
        - name: ACCEPT_LANGUAGE
          value: zh-Hans
```



## k8s二进制安装

![mark](http://noah-pic.oss-cn-chengdu.aliyuncs.com/pic/20200416/142233443.png)

| **主机名** | **IP地址** | 用途          |
| :--------: | :--------: | ------------- |
|  hdss7-11  | 10.4.7.11  | proxy1        |
|  hdss7-12  | 10.4.7.12  | proxy2        |
|  hdss7-21  | 10.4.7.21  | master1,node1 |
|  hdss7-22  | 10.4.7.22  | master2,node2 |
| hdss7-200  | 10.4.7.200 | 运维主机      |

### 部署DNS服务bind9

#### 安装配置DNS服务

```shell
apt install bind9
```

#### 修改并校验配置文件

```shell
vim named.conf.options

directory "/var/cache/bind";
listen-on port 53 { 10.4.7.11; }; 
allow-query     { any; };
forwarders      { 10.4.7.2; }; #上一层DNS地址(网关或公网DNS)
recursion yes;
dnssec-enable no;
dnssec-validation no

named-checkconf
```

```shell
vim named.conf.local
# 添加自定义主机域
zone "host.com" IN {
        type  master;
        file  "host.com.zone";
        allow-update { 10.4.7.11; };
};
# 添加自定义业务域
zone "zq.com" IN {
        type  master;
        file  "zq.com.zone";
        allow-update { 10.4.7.11; };
};
```

> host.com和zq.com都是我们自定义的域名,一般用host.com做为主机域
> zq.com为业务域,业务不同可以配置多个

```shell
cat >/var/cache/bind/host.com.zone <<'EOF'
$ORIGIN host.com.
$TTL 600    ; 10 minutes
@       IN SOA  dns.host.com. dnsadmin.host.com. (
                2020041601 ; serial
                10800      ; refresh (3 hours)
                900        ; retry (15 minutes)
                604800     ; expire (1 week)
                86400      ; minimum (1 day)
                )
            NS   dns.host.com.
$TTL 60 ; 1 minute
dns                A    10.4.7.11
HDSS7-11           A    10.4.7.11
HDSS7-12           A    10.4.7.12
HDSS7-21           A    10.4.7.21
HDSS7-22           A    10.4.7.22
HDSS7-200          A    10.4.7.200

EOF
```

```shell
cat >/var/cache/bind/zq.com.zone <<'EOF'
$ORIGIN zq.com.
$TTL 600    ; 10 minutes
@       IN SOA  dns.zq.com. dnsadmin.zq.com. (
                2020041601 ; serial
                10800      ; refresh (3 hours)
                900        ; retry (15 minutes)
                604800     ; expire (1 week)
                86400      ; minimum (1 day)
                )
            NS   dns.zq.com.
$TTL 60 ; 1 minute
dns                A    10.4.7.11

EOF
```

> host.com域用于主机之间通信,所以要先增加上所有主机
> zq.com域用于后面的业务解析用,因此不需要先添加主机

#### 启动并验证DNS服务

```shell
named-checkconf 
systemctl start bind9
ss -lntup|grep 53

[root@hdss7-11 ~]# dig -t A hdss7-11.host.com @10.4.7.11 +short
10.4.7.11
[root@hdss7-11 ~]# dig -t A hdss7-21.host.com @10.4.7.11 +short
10.4.7.21
```

#### 所有主机修改网络配置

```shell
vim /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    ens33:
      addresses: [10.4.7.11/24]
      gateway4: 10.4.7.2
      dhcp4: no
      optional: true
      nameservers:
        search: [host.com]
        addresses: [10.4.7.11]
  version: 2


cat /etc/resolv.conf
# Generated by NetworkManager
search host.com
nameserver 10.4.7.11


dig -t A hdss7-21.host.com +short
```



### 自签发证书环境准备

```shell
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/bin/cfssl
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/bin/cfssl-json
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/bin/cfssl-certinfo
chmod +x /usr/bin/cfssl*
```

```shell
mkdir /opt/certs
cat >/opt/certs/ca-csr.json <<EOF
{
    "CN": "zqcd",
    "hosts": [
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "chengdu",
            "L": "chengdu",
            "O": "zq",
            "OU": "ops"
        }
    ],
    "ca": {
        "expiry": "175200h"
    }
}
EOF
```

> CN: Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。非常重要。浏览器使用该字段验证网站是否合法
> C: Country， 国家
> ST: State，州，省
> L: Locality，地区，城市
> O: Organization Name，组织名称，公司名称
> OU: Organization Unit Name，组织单位名称，公司部门

```shell
cd /opt/certs
cfssl gencert -initca ca-csr.json | cfssl-json -bare ca
[root@hdss7-200 certs]# ll
total 16
-rw-r--r-- 1 root root  989 Apr 16 20:53 cacsr
-rw-r--r-- 1 root root  324 Apr 16 20:52 ca-csr.json
-rw------- 1 root root 1679 Apr 16 20:53 ca-key.pem
-rw-r--r-- 1 root root 1330 Apr 16 20:53 ca.pem
```

### 安装docker环境准备

```shell
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
apt install docker-compose -y
mkdir /etc/docker/
cat >/etc/docker/daemon.json <<EOF
{
  "graph": "/data/docker", 
  "storage-driver": "overlay2",
  "insecure-registries": ["registry.access.redhat.com","quay.io","harbor.zq.com"],
  "registry-mirrors": ["https://q2gr04ke.mirror.aliyuncs.com"],
  "bip": "172.7.21.1/24",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true
}
EOF
```

> 注意:bip要根据宿主机ip变化
> hdss7-21.host.com bip 172.7.21.1/24
> hdss7-22.host.com bip 172.7.22.1/24
> hdss7-200.host.com bip 172.7.200.1/24

```shell
mkdir -p /data/docker
systemctl start docker
systemctl enable docker
docker --version
```

### 部署harbor私有仓库

https://github.com/goharbor/harbor/releases/download/v1.8.5/harbor-offline-installer-v1.8.5.tgz

```shell
tar xf harbor-offline-installer-v1.8.5.tgz -C /opt/
cd /opt/
mv harbor/ harbor-v1.8.5
ln -s /opt/harbor-v1.8.5/ /opt/harbor
```

```shell
vi /opt/harbor/harbor.yml
hostname: harbor.zq.com
http:
  port: 180
 harbor_admin_password:Harbor12345
data_volume: /data/harbor
log:
    level:  info
    rotate_count:  50
    rotate_size:200M
    location: /data/harbor/logs

mkdir -p /data/harbor/logs
```

```shell
cd /opt/harbor/
sh /opt/harbor/install.sh 
docker-compose ps
docker ps -a
```

```shell
[root@hdss7-11 ~]# vi /var/named/zq.com.zone
2020032002 ; serial   #每次修改DNS解析后,都要滚动此ID
harbor             A    10.4.7.200
[root@hdss7-11 ~]# systemctl restart named
[root@hdss7-11 ~]# dig -t A harbor.zq.com +short
10.4.7.200
```

```shell
apt install nginx -y
vi /etc/nginx/conf.d/harbor.zq.com.conf
server {
    listen       80;
    server_name  harbor.zq.com;

    client_max_body_size 1000m;

    location / {
        proxy_pass http://127.0.0.1:180;
    }
}

nginx -t
systemctl start nginx
systemctl enable nginx
```

> 浏览器输入：harbor.zq.com
> 用户名：admin 密码：Harbor12345
> 新建项目：public 访问级别：公开

```shell
docker login harbor.zq.com -uadmin -pHarbor12345
docker pull kubernetes/pause
docker pull nginx:1.17.9

docker tag kubernetes/pause:latest harbor.zq.com/public/pause:latest
docker tag nginx:1.17.9 harbor.zq.com/public/nginx:v1.17.9

docker push harbor.zq.com/public/pause:latest
docker push harbor.zq.com/public/nginx:v1.17.9
```

```shell
# 创建一个nginx虚拟主机,用来提供文件访问访问,主要依赖nginx的autoindex属性
cat >/etc/nginx/conf.d/k8s-yaml.zq.com.conf <<EOF
server {
    listen       80;
    server_name  k8s-yaml.zq.com;

    location / {
        autoindex on;
        default_type text/plain;
        root /data/k8s-yaml;
    }
}
EOF

# 启动nginx
mkdir -p /data/k8s-yaml/coredns
nginx -t
nginx -s reload
```

```shell
[root@hdss7-11 ~]# vi /var/named/zq.com.zone
# 在最后添加一条解析记录
k8s-yaml           A    10.4.7.200

[root@hdss7-11 ~]# systemctl restart bind9.service
[root@hdss7-11 ~]# dig -t A k8s-yaml.zq.com +short
```

### etcd

#### 创建生成CA证书的JSON配置文件

```shell
cat >/opt/certs/ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "175200h"
        },
        "profiles": {
            "server": {
                "expiry": "175200h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "175200h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "175200h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
} 
EOF
```

> 证书时间统一为10年,不怕过期
> 证书类型
> client certificate：客户端使用，用于服务端认证客户端,例如etcdctl、etcd proxy、fleetctl、docker客户端
> server certificate：服务端使用，客户端以此验证服务端身份,例如docker服务端、kube-apiserver
> peer certificate：双向证书，用于etcd集群成员间通信

#### 创建生成自签发请求(csr)的json配置文件

```shell
cat >/opt/certs/etcd-peer-csr.json <<EOF
{
    "CN": "k8s-etcd",
    "hosts": [
        "10.4.7.11",
        "10.4.7.12",
        "10.4.7.21",
        "10.4.7.22"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "zq",
            "OU": "ops"
        }
    ]
}
EOF
```

#### 生成etcd证书文件

```shell
cd /opt/certs/
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
  -config=ca-config.json -profile=peer \
  etcd-peer-csr.json |cfssl-json -bare etcd-peer

[root@hdss7-200 certs]# ll
total 36
-rw-r--r-- 1 root root  837 Apr 19 15:35 ca-config.json
-rw-r--r-- 1 root root  989 Apr 16 20:53 ca.csr
-rw-r--r-- 1 root root  324 Apr 16 20:52 ca-csr.json
-rw------- 1 root root 1679 Apr 16 20:53 ca-key.pem
-rw-r--r-- 1 root root 1330 Apr 16 20:53 ca.pem
-rw-r--r-- 1 root root 1062 Apr 19 15:35 etcd-peer.csr
-rw-r--r-- 1 root root  363 Apr 19 15:35 etcd-peer-csr.json
-rw------- 1 root root 1679 Apr 19 15:35 etcd-peer-key.pem
-rw-r--r-- 1 root root 1419 Apr 19 15:35 etcd-peer.pem
```



#### 安装启动etcd集群

```shell
useradd -s /sbin/nologin -M etcd
wget https://github.com/etcd-io/etcd/archive/v3.3.25.tar.gz
tar xf etcd-v3.3.25-linux-amd64.tar.gz -C /opt/
cd /opt/
mv etcd-v3.3.25-linux-amd64/ etcd-v3.3.25
ln -s /opt/etcd-v3.3.25/ /opt/etcd
```

```shell
mkdir -p /opt/etcd/certs /data/etcd /data/logs/etcd-server
chown -R etcd.etcd /opt/etcd-v3.3.25/
chown -R etcd.etcd /data/etcd/
chown -R etcd.etcd /data/logs/etcd-server/
```

```shell
cd /opt/etcd/certs
scp hdss7-200:/opt/certs/ca.pem .
scp hdss7-200:/opt/certs/etcd-peer.pem .
scp hdss7-200:/opt/certs/etcd-peer-key.pem .
chown -R etcd.etcd /opt/etcd/certs
```

#### 创建etcd启动脚本

```shell
cat >/opt/etcd/etcd-server-startup.sh <<'EOF'
#!/bin/sh
./etcd \
    --name etcd-server-7-12 \
    --data-dir /data/etcd/etcd-server \
    --listen-peer-urls https://10.4.7.12:2380 \
    --listen-client-urls https://10.4.7.12:2379,http://127.0.0.1:2379 \
    --quota-backend-bytes 8000000000 \
    --initial-advertise-peer-urls https://10.4.7.12:2380 \
    --advertise-client-urls https://10.4.7.12:2379,http://127.0.0.1:2379 \
    --initial-cluster  etcd-server-7-12=https://10.4.7.12:2380,etcd-server-7-21=https://10.4.7.21:2380,etcd-server-7-22=https://10.4.7.22:2380 \
    --ca-file ./certs/ca.pem \
    --cert-file ./certs/etcd-peer.pem \
    --key-file ./certs/etcd-peer-key.pem \
    --client-cert-auth  \
    --trusted-ca-file ./certs/ca.pem \
    --peer-ca-file ./certs/ca.pem \
    --peer-cert-file ./certs/etcd-peer.pem \
    --peer-key-file ./certs/etcd-peer-key.pem \
    --peer-client-cert-auth \
    --peer-trusted-ca-file ./certs/ca.pem \
    --log-output stdout
EOF

chmod +x /opt/etcd/etcd-server-startup.sh
```

> --name    #节点名字 --listen-peer-urls		#监听其他节点所用的地址 
>
> --listen-client-urls	#监听etcd客户端的地址 
>
> --initial-advertise-peer-urls	#与其他节点交互信息的地址 
>
> --advertise-client-urls	#与etcd客户端交互信息的地址

#### supervisor启动etcd

```shell
apt install supervisor -y

find / -name supervisor.sock

unlink /run/supervisor.sock
```

```shell
cat >/etc/supervisor/conf.d/etcd-server.conf <<EOF
[program:etcd-server]  ; 显示的程序名,类型my.cnf,可以有多个
command=sh /opt/etcd/etcd-server-startup.sh
numprocs=1             ; 启动进程数 (def 1)
directory=/opt/etcd    ; 启动命令前切换的目录 (def no cwd)
autostart=true         ; 是否自启 (default: true)
autorestart=true       ; 是否自动重启 (default: true)
startsecs=30           ; 服务运行多久判断为成功(def. 1)
startretries=3         ; 启动重试次数 (default 3)
exitcodes=0,2          ; 退出状态码 (default 0,2)
stopsignal=QUIT        ; 退出信号 (default TERM)
stopwaitsecs=10        ; 退出延迟时间 (default 10)
user=etcd              ; 运行用户
redirect_stderr=true   ; 是否重定向错误输出到标准输出(def false)
stdout_logfile=/data/logs/etcd-server/etcd.stdout.log
stdout_logfile_maxbytes=64MB  ; 日志文件大小 (default 50MB)
stdout_logfile_backups=4      ; 日志文件滚动个数 (default 10)
stdout_capture_maxbytes=1MB   ; 设定capture管道的大小(default 0)
;子进程还有子进程,需要添加这个参数,避免产生孤儿进程
killasgroup=true
stopasgroup=true
EOF
```

```shell
supervisorctl update
supervisorctl status
netstat -lntup|grep etcd
```

```shell
/opt/etcd/etcdctl cluster-health
/opt/etcd/etcdctl member list
```



### kube-apiserver

https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.15.md

#### 签发client端证书

```shell
cat >/opt/certs/client-csr.json <<EOF
{
    "CN": "k8s-node",
    "hosts": [
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "zq",
            "OU": "ops"
        }
    ]
}
EOF
```

```shell
cfssl gencert \
      -ca=ca.pem \
      -ca-key=ca-key.pem \
      -config=ca-config.json \
      -profile=client \
      client-csr.json |cfssl-json -bare client

[root@hdss7-200 certs]# ll|grep client
-rw-r--r-- 1 root root  993 Apr 20 21:30 client.csr
-rw-r--r-- 1 root root  280 Apr 20 21:30 client-csr.json
-rw------- 1 root root 1675 Apr 20 21:30 client-key.pem
-rw-r--r-- 1 root root 1359 Apr 20 21:30 client.pem
```

#### 签发apiserver证书

```shell
cat >/opt/certs/apiserver-csr.json <<EOF
{
    "CN": "k8s-apiserver",
    "hosts": [
        "127.0.0.1",
        "192.168.0.1",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local",
        "10.4.7.10",
        "10.4.7.21",
        "10.4.7.22",
        "10.4.7.23"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "zq",
            "OU": "ops"
        }
    ]
}
EOF
```

```shell
cfssl gencert \
      -ca=ca.pem \
      -ca-key=ca-key.pem \
      -config=ca-config.json \
      -profile=server \
      apiserver-csr.json |cfssl-json -bare apiserver

[root@hdss7-200 certs]# ll|grep apiserver
-rw-r--r-- 1 root root 1249 Apr 20 21:31 apiserver.csr
-rw-r--r-- 1 root root  566 Apr 20 21:31 apiserver-csr.json
-rw------- 1 root root 1675 Apr 20 21:31 apiserver-key.pem
-rw-r--r-- 1 root root 1590 Apr 20 21:31 apiserver.pem
```

#### 部署apiserver服务

```shell
# 上传并解压缩
tar xf kubernetes-server-linux-amd64-v1.15.12.tar.gz  -C /opt
cd /opt
mv kubernetes/ kubernetes-v1.15.12
ln -s /opt/kubernetes-v1.15.12/ /opt/kubernetes

# 清理源码包和docker镜像
cd /opt/kubernetes
rm -rf kubernetes-src.tar.gz
cd server/bin
rm -f *.tar
rm -f *_tag

# 创建命令软连接到系统环境变量下
ln -s /opt/kubernetes/server/bin/kubectl /usr/bin/kubectl
```

```shell
# 创建目录
mkdir -p /opt/kubernetes/server/bin/cert
cd /opt/kubernetes/server/bin/cert

# 拷贝三套证书
scp hdss7-200:/opt/certs/ca.pem .
scp hdss7-200:/opt/certs/ca-key.pem .
scp hdss7-200:/opt/certs/client.pem .
scp hdss7-200:/opt/certs/client-key.pem .
scp hdss7-200:/opt/certs/apiserver.pem .
scp hdss7-200:/opt/certs/apiserver-key.pem .
```

#### 创建audit配置

```shell
mkdir /opt/kubernetes/server/conf

cat >/opt/kubernetes/server/conf/audit.yaml <<'EOF'
apiVersion: audit.k8s.io/v1beta1 # This is required.
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" doesn't match requests to any subresource of pods,
      # which is consistent with the RBAC policy.
      resources: ["pods"]
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Don't log requests to a configmap called "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  # Don't log authenticated requests to certain non-resource URL paths.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    # This rule only applies to resources in the "kube-system" namespace.
    # The empty string "" can be used to select non-namespaced resources.
    namespaces: ["kube-system"]

  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  # Log all other resources in core and extensions at the Request level.
  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions" # Version of group should NOT be included.

  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    # Long-running requests like watches that fall under this rule will not
    # generate an audit event in RequestReceived.
    omitStages:
      - "RequestReceived"
EOF
```

#### 创建apiserver启动脚本

```shell
cat >/opt/kubernetes/server/bin/kube-apiserver.sh <<'EOF'
#!/bin/bash
./kube-apiserver \
  --apiserver-count 2 \
  --audit-log-path /data/logs/kubernetes/kube-apiserver/audit-log \
  --audit-policy-file ../conf/audit.yaml \
  --authorization-mode RBAC \
  --client-ca-file ./cert/ca.pem \
  --requestheader-client-ca-file ./cert/ca.pem \
  --enable-admission-plugins NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
  --etcd-cafile ./cert/ca.pem \
  --etcd-certfile ./cert/client.pem \
  --etcd-keyfile ./cert/client-key.pem \
  --etcd-servers https://10.4.7.12:2379,https://10.4.7.21:2379,https://10.4.7.22:2379 \
  --service-account-key-file ./cert/ca-key.pem \
  --service-cluster-ip-range 192.168.0.0/16 \
  --service-node-port-range 3000-29999 \
  --target-ram-mb=1024 \
  --kubelet-client-certificate ./cert/client.pem \
  --kubelet-client-key ./cert/client-key.pem \
  --log-dir  /data/logs/kubernetes/kube-apiserver \
  --tls-cert-file ./cert/apiserver.pem \
  --tls-private-key-file ./cert/apiserver-key.pem \
  --v 2
EOF

# 授权
chmod +x /opt/kubernetes/server/bin/kube-apiserver.sh
```

#### supervisor启动服务并检查

```shell
cat >/etc/supervisor/conf.d/kube-apiserver.conf <<EOF
[program:kube-apiserver]      ; 显示的程序名,类似my.cnf,可以有多个
command=sh /opt/kubernetes/server/bin/kube-apiserver.sh
numprocs=1                    ; 启动进程数 (def 1)
directory=/opt/kubernetes/server/bin
autostart=true                ; 是否自启 (default: true)
autorestart=true              ; 是否自动重启 (default: true)
startsecs=30                  ; 服务运行多久判断为成功(def. 1)
startretries=3                ; 启动重试次数 (default 3)
exitcodes=0,2                 ; 退出状态码 (default 0,2)
stopsignal=QUIT               ; 退出信号 (default TERM)
stopwaitsecs=10               ; 退出延迟时间 (default 10)
user=root                     ; 运行用户
redirect_stderr=true          ; 重定向错误输出到标准输出(def false)
stdout_logfile=/data/logs/kubernetes/kube-apiserver/apiserver.stdout.log
stdout_logfile_maxbytes=64MB  ; 日志文件大小 (default 50MB)
stdout_logfile_backups=4      ; 日志文件滚动个数 (default 10)
stdout_capture_maxbytes=1MB   ; 设定capture管道的大小(default 0)
;子进程还有子进程,需要添加这个参数,避免产生孤儿进程
killasgroup=true
stopasgroup=true
EOF
```

```shell
mkdir -p /data/logs/kubernetes/kube-apiserver
supervisorctl update
supervisorctl status
netstat -nltup|grep kube-api
```

apiserve、controller-manager、kube-scheduler三个服务所需的软件在同一套压缩包里面的，因此后两个服务不需要在单独解包
而且这三个服务是在同一个主机上，互相之间通过`http://127.0.0.1`,也不需要证书

### controller-manager

#### 创建controller-manager启动脚本

```shell
cat >/opt/kubernetes/server/bin/kube-controller-manager.sh <<'EOF'
#!/bin/sh
./kube-controller-manager \
  --cluster-cidr 172.7.0.0/16 \
  --leader-elect true \
  --log-dir /data/logs/kubernetes/kube-controller-manager \
  --master http://127.0.0.1:8080 \
  --service-account-private-key-file ./cert/ca-key.pem \
  --service-cluster-ip-range 192.168.0.0/16 \
  --root-ca-file ./cert/ca.pem \
  --v 2 
EOF

chmod +x /opt/kubernetes/server/bin/kube-controller-manager.sh
```

#### supervisor启动服务并检查

```shell
cat >/etc/supervisor/conf.d/kube-conntroller-manager.conf <<EOF
[program:kube-controller-manager] ; 显示的程序名
command=sh /opt/kubernetes/server/bin/kube-controller-manager.sh
numprocs=1                    ; 启动进程数 (def 1)
directory=/opt/kubernetes/server/bin
autostart=true                ; 是否自启 (default: true)
autorestart=true              ; 是否自动重启 (default: true)
startsecs=30                  ; 服务运行多久判断为成功(def. 1)
startretries=3                ; 启动重试次数 (default 3)
exitcodes=0,2                 ; 退出状态码 (default 0,2)
stopsignal=QUIT               ; 退出信号 (default TERM)
stopwaitsecs=10               ; 退出延迟时间 (default 10)
user=root                     ; 运行用户
redirect_stderr=true          ; 重定向错误输出到标准输出(def false)
stdout_logfile=/data/logs/kubernetes/kube-controller-manager/controller.stdout.log
stdout_logfile_maxbytes=64MB  ; 日志文件大小 (default 50MB)
stdout_logfile_backups=4      ; 日志文件滚动个数 (default 10)
stdout_capture_maxbytes=1MB   ; 设定capture管道的大小(default 0)
;子进程还有子进程,需要添加这个参数,避免产生孤儿进程
killasgroup=true
stopasgroup=true
EOF
```

```shell
mkdir -p /data/logs/kubernetes/kube-controller-manager
supervisorctl update
supervisorctl status
```

### kube-scheduler

#### 创建kube-scheduler启动脚本

```shell
cat >/opt/kubernetes/server/bin/kube-scheduler.sh <<'EOF'
#!/bin/sh
./kube-scheduler \
  --leader-elect  \
  --log-dir /data/logs/kubernetes/kube-scheduler \
  --master http://127.0.0.1:8080 \
  --v 2
EOF

chmod +x  /opt/kubernetes/server/bin/kube-scheduler.sh
```

#### supervisor启动服务并检查

```shell
cat >/etc/supervisor/conf.d/kube-scheduler.conf <<EOF
[program:kube-scheduler]
command=sh /opt/kubernetes/server/bin/kube-scheduler.sh
numprocs=1                    ; 启动进程数 (def 1)
directory=/opt/kubernetes/server/bin
autostart=true                ; 是否自启 (default: true)
autorestart=true              ; 是否自动重启 (default: true)
startsecs=30                  ; 服务运行多久判断为成功(def. 1)
startretries=3                ; 启动重试次数 (default 3)
exitcodes=0,2                 ; 退出状态码 (default 0,2)
stopsignal=QUIT               ; 退出信号 (default TERM)
stopwaitsecs=10               ; 退出延迟时间 (default 10)
user=root                     ; 运行用户
redirect_stderr=true          ; 重定向错误输出到标准输出(def false)
stdout_logfile=/data/logs/kubernetes/kube-scheduler/scheduler.stdout.log
stdout_logfile_maxbytes=64MB  ; 日志文件大小 (default 50MB)
stdout_logfile_backups=4      ; 日志文件滚动个数 (default 10)
stdout_capture_maxbytes=1MB   ; 设定capture管道的大小(default 0)
;子进程还有子进程,需要添加这个参数,避免产生孤儿进程
killasgroup=true
stopasgroup=true
EOF
```

```shell
mkdir -p /data/logs/kubernetes/kube-scheduler
supervisorctl update
supervisorctl statusa
```

### **keepalived**

```shell
apt install nginx keepalived -y
```

```shell
mkdir /etc/nginx/tcp.d/
echo 'include /etc/nginx/tcp.d/*.conf;' >>/etc/nginx/nginx.conf

cat >/etc/nginx/tcp.d/apiserver.conf <<EOF
stream {
    upstream kube-apiserver {
        server 10.4.7.21:6443     max_fails=3 fail_timeout=30s;
        server 10.4.7.22:6443     max_fails=3 fail_timeout=30s;
    }
    server {
        listen 7443;
        proxy_connect_timeout 2s;
        proxy_timeout 900s;
        proxy_pass kube-apiserver;
    }
}
EOF
```

```shell
nginx -t
systemctl start nginx
systemctl enable nginx
```

#### 创建端口监测脚本

```shell
cat >/etc/keepalived/check_port.sh <<'EOF'
#!/bin/bash
#keepalived 监控端口脚本
#使用方法：等待keepalived传入端口参数,检查改端口是否存在并返回结果
CHK_PORT=$1
if [ -n "$CHK_PORT" ];then
        PORT_PROCESS=`ss -lnt|grep $CHK_PORT|wc -l`
        if [ $PORT_PROCESS -eq 0 ];then
                echo "Port $CHK_PORT Is Not Used,End."
                exit 1
        fi
else
        echo "Check Port Cant Be Empty!"
fi
EOF

chmod +x /etc/keepalived/check_port.sh
```

```shell
# keepalived主
cat >/etc/keepalived/keepalived.conf <<'EOF'
! Configuration File for keepalived
global_defs {
   router_id 10.4.7.11
}
vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 251
    priority 100
    advert_int 1
    mcast_src_ip 10.4.7.11
    nopreempt

    authentication {
        auth_type PASS
        auth_pass 11111111
    }
    track_script {
         chk_nginx
    }
    virtual_ipaddress {
        10.4.7.10
    }
}
EOF
```

```shell
# keepalived从
cat >/etc/keepalived/keepalived.conf <<'EOF'
! Configuration File for keepalived
global_defs {
    router_id 10.4.7.12
}
vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 251
    mcast_src_ip 10.4.7.12
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 11111111
    }
    track_script {
        chk_nginx
    }
    virtual_ipaddress {
        10.4.7.10
    }
}
EOF
```

```shell
systemctl start  keepalived
systemctl enable keepalived
ip addr|grep '10.4.7.10'
```

### node

#### 签发kubelet证书

```shell
cd /opt/certs/
cat >/opt/certs/kubelet-csr.json <<EOF
{
    "CN": "k8s-kubelet",
    "hosts": [
    "127.0.0.1",
    "10.4.7.10",
    "10.4.7.21",
    "10.4.7.22",
    "10.4.7.23",
    "10.4.7.24",
    "10.4.7.25",
    "10.4.7.26",
    "10.4.7.27",
    "10.4.7.28"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "zq",
            "OU": "ops"
        }
    ]
}
EOF
```

```shell
cfssl gencert \
      -ca=ca.pem \
      -ca-key=ca-key.pem \
      -config=ca-config.json \
      -profile=server \
      kubelet-csr.json | cfssl-json -bare kubelet

[root@hdss7-200 certs]# ll |grep kubelet
-rw-r--r-- 1 root root 1115 Apr 22 22:17 kubelet.csr
-rw-r--r-- 1 root root  452 Apr 22 22:17 kubelet-csr.json
-rw------- 1 root root 1679 Apr 22 22:17 kubelet-key.pem
-rw-r--r-- 1 root root 1460 Apr 22 22:17 kubelet.pem
```

#### 创建kubelet服务

```shell
cd /opt/kubernetes/server/bin/cert
scp hdss7-200:/opt/certs/kubelet.pem .
scp hdss7-200:/opt/certs/kubelet-key.pem .
```

#### 创建kubelet配置

> 创建kubelet的配置文件`kubelet.kubeconfig`比较麻烦,需要四步操作才能完成

```shell
cd /opt/kubernetes/server/conf/

kubectl config set-cluster myk8s \
    --certificate-authority=/opt/kubernetes/server/bin/cert/ca.pem \
    --embed-certs=true \
    --server=https://10.4.7.10:7443 \
    --kubeconfig=kubelet.kubeconfig
```

```shell
kubectl config set-credentials k8s-node \
    --client-certificate=/opt/kubernetes/server/bin/cert/client.pem \
    --client-key=/opt/kubernetes/server/bin/cert/client-key.pem \
    --embed-certs=true \
    --kubeconfig=kubelet.kubeconfig
```

```shell
kubectl config set-context myk8s-context \
    --cluster=myk8s \
    --user=k8s-node \
    --kubeconfig=kubelet.kubeconfig
```

```shell
kubectl config use-context myk8s-context --kubeconfig=kubelet.kubeconfig

cat kubelet.kubeconfig 
```

#### 创建k8s-node.yaml配置文件

```shell
cat >/opt/kubernetes/server/conf/k8s-node.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-node
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: k8s-node
EOF

kubectl create -f /opt/kubernetes/server/conf/k8s-node.yaml
kubectl get clusterrolebinding k8s-node
kubectl get clusterrolebinding k8s-node -o yaml
kubectl get nodes
No resources found.
```

#### 创建kubelet启动脚本

```shell
cat >/opt/kubernetes/server/bin/kubelet.sh <<'EOF'
#!/bin/sh
./kubelet \
  --hostname-override hdss7-21.host.com \
  --anonymous-auth=false \
  --cgroup-driver systemd \
  --cluster-dns 192.168.0.2 \
  --cluster-domain cluster.local \
  --runtime-cgroups=/systemd/system.slice \
  --kubelet-cgroups=/systemd/system.slice \
  --fail-swap-on="false" \
  --client-ca-file ./cert/ca.pem \
  --tls-cert-file ./cert/kubelet.pem \
  --tls-private-key-file ./cert/kubelet-key.pem \
  --image-gc-high-threshold 20 \
  --image-gc-low-threshold 10 \
  --kubeconfig ../conf/kubelet.kubeconfig \
  --log-dir /data/logs/kubernetes/kube-kubelet \
  --pod-infra-container-image harbor.zq.com/public/pause:latest \
  --root-dir /data/kubelet
EOF

# 创建目录&授权
chmod +x /opt/kubernetes/server/bin/kubelet.sh
mkdir -p /data/logs/kubernetes/kube-kubelet
mkdir -p /data/kubelet
```

#### supervisor启动服务并检查

```shell
cat >/etc/supervisor/conf.d/kube-kubelet.conf  <<EOF
[program:kube-kubelet]
command=sh /opt/kubernetes/server/bin/kubelet.sh
numprocs=1                    ; 启动进程数 (def 1)
directory=/opt/kubernetes/server/bin    
autostart=true                ; 是否自启 (default: true)
autorestart=true              ; 是否自动重启 (default: true)
startsecs=30                  ; 服务运行多久判断为成功(def. 1)
startretries=3                ; 启动重试次数 (default 3)
exitcodes=0,2                 ; 退出状态码 (default 0,2)
stopsignal=QUIT               ; 退出信号 (default TERM)
stopwaitsecs=10               ; 退出延迟时间 (default 10)
user=root                     ; 运行用户
redirect_stderr=true          ; 重定向错误输出到标准输出(def false)
stdout_logfile=/data/logs/kubernetes/kube-kubelet/kubelet.stdout.log
stdout_logfile_maxbytes=64MB  ; 日志文件大小 (default 50MB)
stdout_logfile_backups=4      ; 日志文件滚动个数 (default 10)
stdout_capture_maxbytes=1MB   ; 设定capture管道的大小(default 0)
;子进程还有子进程,需要添加这个参数,避免产生孤儿进程
killasgroup=true
stopasgroup=true
EOF
```

```shell
supervisorctl update
supervisorctl status
kubectl get nodes
```

#### 部署其他node节点

```shell
# 拷贝证书
cd /opt/kubernetes/server/bin/cert
scp hdss7-200:/opt/certs/kubelet.pem .
scp hdss7-200:/opt/certs/kubelet-key.pem .

# 拷贝配置文件
cd /opt/kubernetes/server/conf/
scp hdss7-21:/opt/kubernetes/server/conf/kubelet.kubeconfig .
```

#### 检查所有节点并给节点打上标签

```shell
kubectl get nodes
kubectl label node hdss7-21.host.com node-role.kubernetes.io/master=
kubectl label node hdss7-21.host.com node-role.kubernetes.io/node=

[root@hdss7-22 cert]# kubectl get nodes
NAME                STATUS   ROLES         AGE   VERSION
hdss7-21.host.com   Ready    master,node   9m    v1.15.5
hdss7-22.host.com   Ready    <none>        64s   v1.15.5
```



### kube-proxy

#### 签发kube-proxy证书

```shell
cd /opt/certs/
cat >/opt/certs/kube-proxy-csr.json <<EOF
{
    "CN": "system:kube-proxy",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "zq",
            "OU": "ops"
        }
    ]
}
EOF
```

```shell
cfssl gencert \
      -ca=ca.pem \
      -ca-key=ca-key.pem \
      -config=ca-config.json \
      -profile=client \
      kube-proxy-csr.json |cfssl-json -bare kube-proxy-client
      
[root@hdss7-200 certs]# ll |grep proxy
-rw-r--r-- 1 root root 1005 Apr 22 22:54 kube-proxy-client.csr
-rw------- 1 root root 1675 Apr 22 22:54 kube-proxy-client-key.pem
-rw-r--r-- 1 root root 1371 Apr 22 22:54 kube-proxy-client.pem
-rw-r--r-- 1 root root  267 Apr 22 22:54 kube-proxy-csr.json
```

#### 创建kube-proxy服务

```shell
cd /opt/kubernetes/server/bin/cert
scp hdss7-200:/opt/certs/kube-proxy-client.pem .
scp hdss7-200:/opt/certs/kube-proxy-client-key.pem .
```

#### 创建kube-proxy配置

> 同样是四步操作,类似kubelet

```shell
cd /opt/kubernetes/server/conf/

kubectl config set-cluster myk8s \
    --certificate-authority=/opt/kubernetes/server/bin/cert/ca.pem \
    --embed-certs=true \
    --server=https://10.4.7.10:7443 \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
    --client-certificate=/opt/kubernetes/server/bin/cert/kube-proxy-client.pem \
    --client-key=/opt/kubernetes/server/bin/cert/kube-proxy-client-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig
    
kubectl config set-context myk8s-context \
    --cluster=myk8s \
    --user=kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig
    
kubectl config use-context myk8s-context --kubeconfig=kube-proxy.kubeconfig
```

#### 加载ipvs模块以备kube-proxy启动用

```shell
# 创建开机ipvs脚本
cat >/etc/ipvs.sh <<'EOF'
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir|grep -o "^[^.]*")
do
  /sbin/modinfo -F filename $i &>/dev/null
  if [ $? -eq 0 ];then
    /sbin/modprobe $i
  fi
done
EOF

# 执行脚本开启ipvs
sh /etc/ipvs.sh 

lsmod |grep ip_vs
```

#### 创建kube-proxy启动脚本

```shell
cat >/opt/kubernetes/server/bin/kube-proxy.sh <<'EOF'
#!/bin/sh
./kube-proxy \
  --hostname-override hdss7-21.host.com \
  --cluster-cidr 172.7.0.0/16 \
  --proxy-mode=ipvs \
  --ipvs-scheduler=nq \
  --kubeconfig ../conf/kube-proxy.kubeconfig
EOF

# 授权
chmod +x /opt/kubernetes/server/bin/kube-proxy.sh 
```

#### 创建kube-proxy的supervisor配置

```shell
cat >/etc/supervisor/conf.d/kube-proxy.conf <<'EOF'
[program:kube-proxy]
command=sh /opt/kubernetes/server/bin/kube-proxy.sh
numprocs=1                    ; 启动进程数 (def 1)
directory=/opt/kubernetes/server/bin
autostart=true                ; 是否自启 (default: true)
autorestart=true              ; 是否自动重启 (default: true)
startsecs=30                  ; 服务运行多久判断为成功(def. 1)
startretries=3                ; 启动重试次数 (default 3)
exitcodes=0,2                 ; 退出状态码 (default 0,2)
stopsignal=QUIT               ; 退出信号 (default TERM)
stopwaitsecs=10               ; 退出延迟时间 (default 10)
user=root                     ; 运行用户
redirect_stderr=true          ; 重定向错误输出到标准输出(def false)
stdout_logfile=/data/logs/kubernetes/kube-proxy/proxy.stdout.log
stdout_logfile_maxbytes=64MB  ; 日志文件大小 (default 50MB)
stdout_logfile_backups=4      ; 日志文件滚动个数 (default 10)
stdout_capture_maxbytes=1MB   ; 设定capture管道的大小(default 0)
;子进程还有子进程,需要添加这个参数,避免产生孤儿进程
killasgroup=true
stopasgroup=true
EOF
```

#### 启动服务并检查

```shell
mkdir -p /data/logs/kubernetes/kube-proxy
supervisorctl update
supervisorctl status
kubectl get svc

apt install ipvsadm -y
ipvsadm -Ln
```

#### 部署其他kube-proxy节点

```shell
# 拷贝证书文件
cd /opt/kubernetes/server/bin/cert
scp hdss7-200:/opt/certs/kube-proxy-client.pem .
scp hdss7-200:/opt/certs/kube-proxy-client-key.pem .

# 拷贝配置文件
cd /opt/kubernetes/server/conf/
scp hdss7-21:/opt/kubernetes/server/conf/kube-proxy.kubeconfig .
```



### 验证kubernetes集群

```shell
cat >/root/nginx-ds.yaml <<'EOF'
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ds
spec:
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: harbor.zq.com/public/nginx:v1.17.9
        ports:
        - containerPort: 80
EOF

kubectl create -f /root/nginx-ds.yaml

kubectl get cs
kubectl get nodes 
kubectl get pods
kubectl get pods -o wide
```

### flannel

#### 在etcd中写入网络信息

```shell
/opt/etcd/etcdctl set /coreos.com/network/config '{"Network": "172.7.0.0/16", "Backend": {"Type": "host-gw"}}'

/opt/etcd/etcdctl get /coreos.com/network/config
```

####  部署准备

##### 下载软件

```shell
wget https://github.com/coreos/flannel/releases/download/v0.12.0/flannel-v0.12.0-linux-amd64.tar.gz
mkdir /opt/flannel-v0.12.0
tar xf flannel-v0.12.0-linux-amd64.tar.gz -C /opt/flannel-v0.12.0/
ln -s /opt/flannel-v0.12.0/ /opt/flannel
```

##### 拷贝证书

```shell
cd /opt/flannel
mkdir cert
scp hdss7-200:/opt/certs/ca.pem         cert/ 
scp hdss7-200:/opt/certs/client.pem     cert/ 
scp hdss7-200:/opt/certs/client-key.pem cert/ 
```

##### 配置子网信息

```shell
cat >/opt/flannel/subnet.env <<EOF
FLANNEL_NETWORK=172.7.0.0/16
FLANNEL_SUBNET=172.7.21.1/24
FLANNEL_MTU=1500
FLANNEL_IPMASQ=false
EOF
```

#### 创建flannel启动脚本

```shell
cat >/opt/flannel/flanneld.sh <<'EOF'
#!/bin/sh
./flanneld \
  --public-ip=10.4.7.21 \
  --etcd-endpoints=https://10.4.7.12:2379,https://10.4.7.21:2379,https://10.4.7.22:2379 \
  --etcd-keyfile=./cert/client-key.pem \
  --etcd-certfile=./cert/client.pem \
  --etcd-cafile=./cert/ca.pem \
  --iface=ens33 \
  --subnet-file=./subnet.env \
  --healthz-port=2401
EOF

chmod +x flanneld.sh
```

> 注意:
> public-ip为节点IP,注意按需修改
> iface为网卡,若本机网卡不是ens33,注意修改

#### supervisor启动服务并检查

```shell
cat >/etc/supervisor/conf.d/flannel.conf <<EOF
[program:flanneld]
command=sh /opt/flannel/flanneld.sh
numprocs=1
directory=/opt/flannel
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/flanneld/flanneld.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
;子进程还有子进程,需要添加这个参数,避免产生孤儿进程
killasgroup=true
stopasgroup=true
EOF
```

```shell
mkdir -p /data/logs/flanneld
supervisorctl update
supervisorctl status
```

```shell
route -n|egrep -i '172.7|des'

ping 172.7.22.2
ping 172.7.21.2
```

#### 优化iptables规则

1. Docker容器的跨网络隔离与通信，借助了iptables的机制

2. 因此虽然K8S我们使用了ipvs调度,但是宿主机上还是有iptalbes规则

3. 而docker默认生成的iptables规则为：
   若数据出网前，先判断出网设备是不是本机`docker0`设备(容器网络)
   如果不是的话，则进行SNAT转换后再出网，具体规则如下

   ```sh
   [root@hdss7-21 ~]# iptables-save |grep -i postrouting|grep docker0
   -A POSTROUTING -s 172.7.21.0/24 ! -o docker0 -j MASQUERADE
   ```

4. 由于`gw`模式产生的数据,是从`eth0`流出,因而不在此规则过滤范围内

5. 就导致此跨宿主机之间的POD通信,使用了该条SNAT规则

**解决办法是:**

- 修改此IPTABLES规则,增加过滤目标:**过滤目的地是宿主机网段的流量**

```shell
iptables-save |grep -i postrouting|grep docker0
-A POSTROUTING -s 172.7.21.0/24 ! -o docker0 -j MASQUERADE

apt install iptables -y
iptables -t nat -D POSTROUTING -s 172.7.21.0/24 ! -o docker0 -j MASQUERADE
iptables -t nat -I POSTROUTING -s 172.7.21.0/24 ! -d 172.7.0.0/16 ! -o docker0  -j MASQUERADE

iptables-save |grep -i postrouting|grep docker0

iptables-save > /etc/iptables.up.rules 
vim /etc/network/interfaces
auto ens33 
iface ens33 inet dhcp 
pre-up iptables-restore < /etc/iptables.up.rules 

# 重启docker 会重新生成iptables规则，需重新删掉
systemctl restart docker
iptables-save |grep -i postrouting|grep docker0
iptables-restore /etc/sysconfig/iptables
iptables-save |grep -i postrouting|grep docker0
```

#### 结果验证

```shell
docker pull busybox
[hdss7-22 #] kubectl logs nginx-ds-j777c --tail=10 -f
docker run --rm -it busybox
wget 172.7.22.2
```

### coredns

**在K8S集群中，POD有以下特性：**

1. 服务动态性强
   容器在k8s中迁移会导致POD的IP地址变化
2. 更新发布频繁
   版本迭代快，新旧POD的IP地址会不同
3. 支持自动伸缩
   大促或流量高峰需要动态伸缩，IP地址会动态增减

**service资源解决POD服务发现：**
为了解决pod地址变化的问题，需要部署service资源，用service资源代理后端pod，通过暴露service资源的固定地址(集群IP)，来解决以上POD资源变化产生的IP变动问题

**那service资源的服务发现呢？**

service资源提供了一个不变的集群IP供外部访问，但

1. IP地址毕竟难以记忆
2. service资源可能也会被销毁和创建
3. 能不能将service资源名称和service暴露的集群网络IP对于
4. 类似域名与IP关系，则只需记服务名就能自动匹配服务IP
5. 岂不就达到了service服务的自动发现

**在k8s中，coredns就是为了解决以上问题。**

#### 获取coredns的docker镜像

```shell
docker pull docker.io/coredns/coredns:1.6.1
docker tag coredns:1.6.1 harbor.zq.com/public/coredns:v1.6.1
docker push harbor.zq.com/public/coredns:v1.6.1
```

```shell
mkdir -p /data/k8s-yaml/coredns
```

#### rbac集群权限清单

```shell
cat >/data/k8s-yaml/coredns/rbac.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
EOF
```

#### configmap配置清单

```shell
cat >/data/k8s-yaml/coredns/cm.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        log
        health
        ready
        kubernetes cluster.local 192.168.0.0/16  #service资源cluster地址
        forward . 10.4.7.11   #上级DNS地址
        cache 30
        loop
        reload
        loadbalance
       }
EOF
```

#### depoly控制器清单

```shell
cat >/data/k8s-yaml/coredns/dp.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      containers:
      - name: coredns
        image: harbor.zq.com/public/coredns:v1.6.1
        args:
        - -conf
        - /etc/coredns/Corefile
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
EOF
```

#### service资源清单

```shell
cat >/data/k8s-yaml/coredns/svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: 192.168.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
  - name: metrics
    port: 9153
    protocol: TCP
EOF
```

#### 创建资源

```shell
dig -t A harbor.zq.com  +short

kubectl create -f http://k8s-yaml.zq.com/coredns/rbac.yaml
kubectl create -f http://k8s-yaml.zq.com/coredns/cm.yaml
kubectl create -f http://k8s-yaml.zq.com/coredns/dp.yaml
kubectl create -f http://k8s-yaml.zq.com/coredns/svc.yaml
```

```shell
kubectl get all -n kube-system
kubectl get svc -o wide -n kube-system
dig -t A www.baidu.com @192.168.0.2 +short
dig -t A harbor.zq.com @192.168.0.2 +short  (forward . 10.4.7.11)
```

#### 创建一个service资源来验证

```shell
kubectl create deployment nginx-dp --image=harbor.zq.com/public/nginx:v1.17.9 -n kube-public
kubectl expose deployment nginx-dp --port=80 -n kube-public

dig -t A nginx-dp @192.168.0.2 +short
dig -t A nginx-dp.kube-public.svc.cluster.local. @192.168.0.2 +short

# 进入到pod内部再次验证
kubectl -n kube-public exec -it nginx-dp-568f8dc55-rxvx2 /bin/bash
apt update && apt install curl
ping nginx-dp

cat /etc/resolv.conf 
nameserver 192.168.0.2
search kube-public.svc.cluster.local svc.cluster.local cluster.local host.com
options ndots:5
```



### Ingress-traefik

https://github.com/containous/traefik

#### 准备docker镜像

```shell
docker pull traefik:v1.7.2-alpine
docker tag  traefik:v1.7.2-alpine harbor.zq.com/public/traefik:v1.7.2
docker push harbor.zq.com/public/traefik:v1.7.2
```

```shell
mkdir -p /data/k8s-yaml/traefik
```

#### rbac授权清单

```shell
cat >/data/k8s-yaml/traefik/rbac.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
EOF
```

#### delepoly资源清单

```shell
cat >/data/k8s-yaml/traefik/ds.yaml <<EOF
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: traefik-ingress
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress
        name: traefik-ingress
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: harbor.zq.com/public/traefik:v1.7.2
        name: traefik-ingress
        ports:
        - name: controller
          containerPort: 80
          hostPort: 81
        - name: admin-web
          containerPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
        - --insecureskipverify=true
        - --kubernetes.endpoint=https://10.4.7.10:7443
        - --accesslog
        - --accesslog.filepath=/var/log/traefik_access.log
        - --traefiklog
        - --traefiklog.filepath=/var/log/traefik.log
        - --metrics.prometheus
EOF
```

#### service清单

```shell
cat >/data/k8s-yaml/traefik/svc.yaml <<EOF
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress
  ports:
    - protocol: TCP
      port: 80
      name: controller
    - protocol: TCP
      port: 8080
      name: admin-web
EOF
```

#### ingress清单

```shell
cat >/data/k8s-yaml/traefik/ingress.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik.zq.com
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-ingress-service
          servicePort: 8080
EOF
```

#### 创建资源

```shell
kubectl create -f http://k8s-yaml.zq.com/traefik/rbac.yaml
kubectl create -f http://k8s-yaml.zq.com/traefik/ds.yaml
kubectl create -f http://k8s-yaml.zq.com/traefik/svc.yaml
kubectl create -f http://k8s-yaml.zq.com/traefik/ingress.yaml
```

#### 前端nginx上做反向代理

```shell
# 在7.11和7.12上,都做反向代理,将泛域名的解析都转发到traefik上去
cat >/etc/nginx/conf.d/zq.com.conf <<'EOF'
upstream default_backend_traefik {
    server 10.4.7.21:81    max_fails=3 fail_timeout=10s;
    server 10.4.7.22:81    max_fails=3 fail_timeout=10s;
}
server {
    server_name *.zq.com;
  
    location / {
        proxy_pass http://default_backend_traefik;
        proxy_set_header Host       $http_host;
        proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    }
}
EOF

# 重启nginx服务
nginx -t
nginx -s reload
```

```shell
vi /var/named/zq.com.zone
........
traefik            A    10.4.7.10

systemctl restart bind9.service
dig -t A traefik.zq.com +short

http://traefik.zq.com
```

### dashboard

#### 获取1.8.3版本的dsashboard

```shell
docker pull k8scn/kubernetes-dashboard-amd64:v1.8.3
docker tag  k8scn/kubernetes-dashboard-amd64:v1.8.3 harbor.zq.com/public/dashboard:v1.8.3
docker push harbor.zq.com/public/dashboard:v1.8.3
```

#### 获取1.10.1版本的dashboard

```shell
docker pull loveone/kubernetes-dashboard-amd64:v1.10.1
docker tag  loveone/kubernetes-dashboard-amd64:v1.10.1 harbor.zq.com/public/dashboard:v1.10.1
docker push harbor.zq.com/public/dashboard:v1.10.1
```

- 1.8.3版本授权不严格,方便学习使用
- 1.10.1版本授权严格,学习使用麻烦,但生产需要

```shell
mkdir -p /data/k8s-yaml/dashboard
```

#### 创建rbca授权清单

```shell
cat >/data/k8s-yaml/dashboard/rbac.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system
EOF
```

#### 创建depoloy清单

```shell
cat >/data/k8s-yaml/dashboard/dp.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      priorityClassName: system-cluster-critical
      containers:
      - name: kubernetes-dashboard
        image: harbor.zq.com/public/dashboard:v1.8.3
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 50m
            memory: 100Mi
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          # PLATFORM-SPECIFIC ARGS HERE
          - --auto-generate-certificates
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard-admin
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
EOF
```

#### 创建service清单

```shell
cat >/data/k8s-yaml/dashboard/svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    k8s-app: kubernetes-dashboard
  ports:
  - port: 443
    targetPort: 8443
EOF
```

#### 创建ingress清单暴露服务

```shell
cat >/data/k8s-yaml/dashboard/ingress.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: dashboard.zq.com
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
EOF
```

#### 创建相关资源

```shell
kubectl create -f http://k8s-yaml.zq.com/dashboard/rbac.yaml
kubectl create -f http://k8s-yaml.zq.com/dashboard/dp.yaml
kubectl create -f http://k8s-yaml.zq.com/dashboard/svc.yaml
kubectl create -f http://k8s-yaml.zq.com/dashboard/ingress.yaml
```

#### 添加域名解析

```shell
vi /var/named/zq.com.zone
dashboard          A    10.4.7.10
# 注意前滚serial编号

systemctl restart bind9.service
http://dashboard.zq.com
```

#### 升级dashboard版本

```shell
kubectl edit deploy kubernetes-dashboard -n kube-system

kubectl -n kube-system get pod|grep dashboard
```

#### 使用token登录

```shell
# 首先获取secret资源列表
kubectl get secret  -n kube-system
# 获取角色的详情
kubectl -n kube-system describe secrets kubernetes-dashboard-admin-token-85gmd
```

**因为必须使用https登录,所以需要申请证书**

#### 创建json文件

```shell
cd /opt/certs/
cat >/opt/certs/dashboard-csr.json <<EOF
{
    "CN": "*.zq.com",
    "hosts": [
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "zq",
            "OU": "ops"
        }
    ]
}
EOF
```

#### 申请证书

```shell
cfssl gencert -ca=ca.pem \
      -ca-key=ca-key.pem \
      -config=ca-config.json \
      -profile=server \
      dashboard-csr.json |cfssl-json -bare dashboard

[root@hdss7-200 certs]# ll |grep dash
-rw-r--r-- 1 root root  993 May  4 12:08 dashboard.csr
-rw-r--r-- 1 root root  280 May  4 12:08 dashboard-csr.json
-rw------- 1 root root 1675 May  4 12:08 dashboard-key.pem
-rw-r--r-- 1 root root 1359 May  4 12:08 dashboard.pem
```

#### 前端nginx服务部署证书

```shell
# 7-11,7-12
mkdir /etc/nginx/certs
scp 10.4.7.200:/opt/certs/dashboard.pem /etc/nginx/certs
scp 10.4.7.200:/opt/certs/dashboard-key.pem /etc/nginx/certs
```

**创建nginx配置**

```shell
cat >/etc/nginx/conf.d/dashboard.zq.com.conf <<'EOF'
server {
    listen       80;
    server_name  dashboard.zq.com;

    rewrite ^(.*)$ https://${server_name}$1 permanent;
}
server {
    listen       443 ssl;
    server_name  dashboard.zq.com;

    ssl_certificate     "certs/dashboard.pem";
    ssl_certificate_key "certs/dashboard-key.pem";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://default_backend_traefik;
        proxy_set_header Host       $http_host;
        proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    }
}
EOF
```

```shell
# 重启nginx服务
nginx -t
nginx -s reload
```



## Dubbo微服务

### 部署ZK集群

#### 二进制安装JDK

```shell
mkdir /opt/src
mkdir /usr/java
cd /opt/src

wget http://k8s-yaml.zq.com/jdk/jdk-8u261-linux-x64.tar.gz
tar -xf jdk-8u261-linux-x64.tar.gz -C /usr/java/
ln -s /usr/java/jdk1.8.0_261/ /usr/java/jdk
```
```shell
cat >>/etc/profile <<'EOF'
#JAVA HOME
export JAVA_HOME=/usr/java/jdk
export PATH=$JAVA_HOME/bin:$JAVA_HOME/bin:$PATH
export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
EOF

source /etc/profile

java -version
```

#### 二进制安装zk

https://archive.apache.org/dist/zookeeper/

##### 下载zookeeper

```shell
wget https://archive.apache.org/dist/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
tar -zxf zookeeper-3.4.14.tar.gz -C /opt/
ln -s /opt/zookeeper-3.4.14/ /opt/zookeeper
```

##### 创建zk配置文件

```shell
cat >/opt/zookeeper/conf/zoo.cfg <<'EOF'
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/logs
clientPort=2181
server.1=zk1.zq.com:2888:3888
server.2=zk2.zq.com:2888:3888
server.3=zk3.zq.com:2888:3888
EOF
```

```shell
mkdir -p /data/zookeeper/data
mkdir -p /data/zookeeper/logs
```

##### 创建集群配置

```shell
#7-11上
echo 1 > /data/zookeeper/data/myid
#7-12上
echo 2 > /data/zookeeper/data/myid
#7-21上
echo 3 > /data/zookeeper/data/myid
```

##### 修改dns解析

```shell
vim /var/cache/bind/zq.com.zone
...
zk1        A    10.4.7.11
zk2        A    10.4.7.12
zk3        A    10.4.7.21

systemctl restart bind9.service
dig -t A zk1.zq.com  +short
```

#### 启动zk集群

```shell
/opt/zookeeper/bin/zkServer.sh start
ss -ln|grep 2181
/opt/zookeeper/bin/zkServer.sh status

ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: follower

ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: leader
```

### 准备java运行底包

#### 拉取原始底包

```shell
docker pull stanleyws/jre8:8u112
docker tag stanleyws/jre8:8u112 harbor.zq.com/public/jre:8u112
docker push harbor.zq.com/public/jre:8u112
```

#### 制作新底包

```shell
mkdir -p /data/dockerfile/jre8/
cd /data/dockerfile/jre8/
```

#### 制作dockerfile

```shell
cat >Dockerfile <<'EOF'
FROM harbor.zq.com/public/jre:8u112
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&\
    echo 'Asia/Shanghai' >/etc/timezone
ADD config.yml /opt/prom/config.yml
ADD jmx_javaagent-0.3.1.jar /opt/prom/
WORKDIR /opt/project_dir
ADD entrypoint.sh /entrypoint.sh
CMD ["sh","/entrypoint.sh"]
EOF
```
#### 准备dockerfile需要的文件

```shell
# 添加config.yml
cat >config.yml <<'EOF'
--- 
rules: 
 - pattern: '.*'
EOF

# 下载jmx_javaagent,监控jvm信息
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.3.1/jmx_prometheus_javaagent-0.3.1.jar -O jmx_javaagent-0.3.1.jar

# 创建entrypoint.sh启动脚本
cat >entrypoint.sh <<'EOF'
#!/bin/sh
M_OPTS="-Duser.timezone=Asia/Shanghai -javaagent:/opt/prom/jmx_javaagent-0.3.1.jar=$(hostname -i):${M_PORT:-"12346"}:/opt/prom/config.yml"
C_OPTS=${C_OPTS}
JAR_BALL=${JAR_BALL}
exec java -jar ${M_OPTS} ${C_OPTS} ${JAR_BALL}
EOF
```

#### 构建底包并上传

在harbor中创建名为`base`的公开仓库,用来存放自己自定义的底包

```shell
docker build . -t harbor.zq.com/base/jre8:8u112
docker push  harbor.zq.com/base/jre8:8u112
```

### 交付jenkins到k8s集群

#### 下载官方镜像

```shell
docker pull jenkins/jenkins:2.190.3
docker tag jenkins/jenkins:2.190.3 harbor.zq.com/public/jenkins:v2.190.3
docker push harbor.zq.com/public/jenkins:v2.190.3
```

#### 修改官方镜像

```shell
mkdir -p /data/dockerfile/jenkins/
cd /data/dockerfile/jenkins/

# 创建dockerfile
cat >/data/dockerfile/jenkins/Dockerfile <<'EOF'
FROM harbor.zq.com/public/jenkins:v2.190.3

#定义启动jenkins的用户
USER root

#修改时区为东八区
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&\ 
    echo 'Asia/Shanghai' >/etc/timezone

#加载用户密钥，使用ssh拉取dubbo代码需要
ADD id_rsa /root/.ssh/id_rsa

#加载运维主机的docker配置文件，里面包含登录harbor仓库的认证信息。
ADD config.json /root/.docker/config.json

#在jenkins容器内安装docker客户端，docker引擎用的是宿主机的docker引擎
ADD get-docker.sh /get-docker.sh

# 跳过ssh时候输入yes的交互步骤，并执行安装docker
RUN echo "    StrictHostKeyChecking no" >/etc/ssh/ssh_config &&\
    /get-docker.sh  
EOF
```

#### 准备dockerfile所需文件

```shell
ssh-keygen -t rsa -b 2048 -C "362169885@qq.com" -N "" -f /root/.ssh/id_rsa
cp /root/.ssh/id_rsa /data/dockerfile/jenkins/
```

> 邮箱请根据自己的邮箱自行修改
> 创建完成后记得把公钥放到gitee的信任中

##### 获取docker.sh脚本

```shell
curl -fsSL get.docker.com -o /data/dockerfile/jenkins/get-docker.sh
chmod u+x /data/dockerfile/jenkins/get-docker.sh

cp /root/.docker/config.json /data/dockerfile/jenkins/
```

#### harbor中创建私有仓库infra

在harbor中创建名为`infra`的私有仓库

```shell
cd /data/dockerfile/jenkins/
docker build . -t harbor.zq.com/infra/jenkins:v2.190.3
docker push harbor.zq.com/infra/jenkins:v2.190.3
```

### 准备jenkins运行环境

#### 创建专有namespace

```shell
# 创建专有namespace
kubectl create ns infra
# 创建访问harbor的secret规则
kubectl create secret docker-registry harbor \
    --docker-server=harbor.zq.com \
    --docker-username=admin \
    --docker-password=Harbor12345 \
    -n infra
# 查看结果
kubectl -n infra get secrets 
```

> 解释命令：
> 创建一条secret，资源类型是docker-registry，名字是 harbor
> 并指定docker仓库地址、访问用户、密码、仓库名

#### 创建NFS共享存储



```shell
# 7-200
apt install nfs-kernel-server -y
echo '/data/nfs-volume 10.4.7.0/24(rw,no_root_squash)' >>/etc/exports
mkdir -p /data/nfs-volume/jenkins_home
systemctl restart nfs-kernel-server
systemctl status nfs-kernel-server
showmount -e

# 7-21，7-22
apt-get install nfs-common
mkdir nfs-test
mount hdss7-200:/data/nfs-volume ./nfs-test/
umount ./nfs-test
```

#### 创建jenkins资源清单

```shell
mkdir /data/k8s-yaml/jenkins
```

##### 创建depeloy清单

```shell
cat >/data/k8s-yaml/jenkins/dp.yaml <<EOF
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: jenkins
  namespace: infra
  labels: 
    name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: jenkins
  template:
    metadata:
      labels: 
        app: jenkins 
        name: jenkins
    spec:
      volumes:
      - name: data
        nfs: 
          server: hdss7-200
          path: /data/nfs-volume/jenkins_home
      - name: docker
        hostPath: 
          path: /run/docker.sock   
          type: ''
      containers:
      - name: jenkins
        image: harbor.zq.com/infra/jenkins:v2.190.3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          protocol: TCP
        env:
        - name: JAVA_OPTS
          value: -Xmx512m -Xms512m
        volumeMounts:
        - name: data
          mountPath: /var/jenkins_home
        - name: docker
          mountPath: /run/docker.sock
      imagePullSecrets:
      - name: harbor
      securityContext: 
        runAsUser: 0
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxUnavailable: 1
      maxSurge: 1
  revisionHistoryLimit: 7
  progressDeadlineSeconds: 600
EOF
```

##### 创建service清单

```shell
cat >/data/k8s-yaml/jenkins/svc.yaml <<EOF
kind: Service
apiVersion: v1
metadata: 
  name: jenkins
  namespace: infra
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  selector:
    app: jenkins
EOF
```

##### 创建ingress清单

```shell
cat >/data/k8s-yaml/jenkins/ingress.yaml <<EOF
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: jenkins
  namespace: infra
spec:
  rules:
  - host: jenkins.zq.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: jenkins
          servicePort: 80
EOF
```

#### 部署jenkins

```shell
kubectl create -f http://k8s-yaml.zq.com/jenkins/dp.yaml
kubectl create -f http://k8s-yaml.zq.com/jenkins/svc.yaml
kubectl create -f http://k8s-yaml.zq.com/jenkins/ingress.yaml

kubectl get pod -n infra
```

##### 验证jenkins容器状态

```shell
docker exec -it 8ff92f08e3aa /bin/bash
# 查看用户
whoami
# 查看时区
date
# 查看是否能用宿主机的docker引擎
docker ps 
# 看是否能免密访问gitee
ssh -i /root/.ssh/id_rsa -T git@gitee.com
# 是否能访问是否harbor仓库
docker login harbor.zq.com
```

##### 查看持久化结果和密码

```shell
ll /data/nfs-volume/jenkins_home
cat /data/nfs-volume/jenkins_home/secrets/initialAdminPassword

# 替换jenkins插件源
cd /data/nfs-volume/jenkins_home/updates
sed -i 's#http:\/\/updates.jenkins-ci.org\/download#https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins#g' default.json
sed -i 's#http:\/\/www.google.com#https:\/\/www.baidu.com#g' default.json
```

#### 解析jenkins

```shell
vim /var/cache/bind/zq.com.zone
...
jenkins         A    10.4.7.10

systemctl restart bind9.service
http://jenkins.zq.com
```

#### 初始化jenkins

浏览器访问`http://jenkins.zq.com`,使用前面的密码进入jenkins
进入后操作:

1. 跳过安装自动安装插件的步骤
2. 在`manage jenkins`->`Configure Global Security`菜单中设置
   2.1 允许匿名读：勾选`allow anonymous read access`
   2.2 允许跨域：勾掉`prevent cross site request forgery exploits`
3. 搜索并安装蓝海插件`blue ocean`
4. 设置用户名密码为`admin:admin123`
5. Jenkins的更新网站 替换http://mirror.xmission.com/jenkins/updates/update-center.json 来进行更新

#### 给jenkins配置maven环境

```shell
wget https://archive.apache.org/dist/maven/maven-3/3.6.1/binaries/apache-maven-3.6.1-bin.tar.gz
tar -zxf apache-maven-3.6.1-bin.tar.gz -C /data/nfs-volume/jenkins_home/
mv /data/nfs-volume/jenkins_home/{apache-,}maven-3.6.1
cd /data/nfs-volume/jenkins_home/maven-3.6.1
```

```shell
cat >conf/settings.xml  <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd"> 
  <pluginGroups>
  </pluginGroups>
  <proxies>
  </proxies>
  <servers>
  </servers>
  <mirrors>
	<mirror>
	  <id>nexus-aliyun</id>
	  <mirrorOf>*</mirrorOf>
	  <name>Nexus aliyun</name>
	  <url>http://maven.aliyun.com/nexus/content/groups/public</url>
	</mirror>
  </mirrors>
  <profiles>
  </profiles>
</settings>
EOF
```

### 交付dubbo-service到k8s

#### 准备资源清单

```shell
mkdir /data/k8s-yaml/dubbo-server/
cd /data/k8s-yaml/dubbo-server
```

##### 创建depeloy清单

```shell
cat >dp.yaml <<EOF
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: dubbo-demo-service
  namespace: app
  labels: 
    name: dubbo-demo-service
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: dubbo-demo-service
  template:
    metadata:
      labels: 
        app: dubbo-demo-service
        name: dubbo-demo-service
    spec:
      containers:
      - name: dubbo-demo-service
        image: harbor.zq.com/app/dubbo-demo-service:master_200930_1700
        ports:
        - containerPort: 20880
          protocol: TCP
        env:
        - name: JAR_BALL
          value: dubbo-server.jar
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      securityContext: 
        runAsUser: 0
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxUnavailable: 1
      maxSurge: 1
  revisionHistoryLimit: 7
  progressDeadlineSeconds: 600
EOF
```

> 需要根据自己构建镜像的tag来修改image
> dubbo的server服务,只向zk注册并通过zk与dobbo的web交互,不需要对外提供服务
> 因此不需要service资源和ingress资源

##### 创建app名称空间

```shell
kubectl create namespace app
```

##### 创建secret资源

```shell
kubectl -n app \
    create secret docker-registry harbor \
    --docker-server=harbor.zq.com \
    --docker-username=admin \
    --docker-password=Harbor12345
```

##### 应用资源清单

```shell
kubectl apply -f http://k8s-yaml.zq.com/dubbo-server/dp.yaml
```

**3分钟后检查启动情况**

```shell
kubectl -n app get pod
kubectl -n app logs dubbo-demo-service-56df6957dc-d99n4 --tail=10
```

**到zk服务器检查是否有服务注册**

```shell
sh /opt/zookeeper/bin/zkCli.sh
[zk: localhost:2181(CONNECTED) 0] ls /
[dubbo, zookeeper]
[zk: localhost:2181(CONNECTED) 1] ls /dubbo
[com.od.dubbotest.api.HelloService]
```



































































