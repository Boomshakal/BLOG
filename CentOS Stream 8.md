# CentOS Stream 8

## [阿里云源](https://developer.aliyun.com/mirror/centos?spm=a2c6h.13651102.0.0.3e221b11YGZDPv)

```shell
# 阿里云Base源
mv /etc/yum.repos.d/CentOS-Linux-BaseOS.repo /etc/yum.repos.d/CentOS-Base.repo.backup
curl -o /etc/yum.repos.d/CentOS-Linux-BaseOS.repo https://mirrors.aliyun.com/repo/Centos-8.repo
yum makecache
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Linux-BaseOS.repo

# 阿里云epel源
dnf -y install epel-release
mv /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel.repo.backup
mv /etc/yum.repos.d/epel-testing.repo /etc/yum.repos.d/epel-testing.repo.backup
yum install -y https://mirrors.aliyun.com/epel/epel-release-latest-8.noarch.rpm
sed -i 's|^#baseurl=https://download.example/pub|baseurl=https://mirrors.aliyun.com|' /etc/yum.repos.d/epel*
sed -i 's|^metalink|#metalink|' /etc/yum.repos.d/epel*
yum makecache
```

## 关闭防火墙

```shell
systemctl status firewalld
systemctl stop firewalld
systemctl disable firewalld
getenforce
sed -i  "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
grubby --update-kernel ALL --args selinux=0
reboot
grubby --update-kernel ALL --remove-args selinux
getenforce
```

## 网络设置

```shell
hostnamectl set-hostname dlp.srv.world

nmcli device
nmcli connection modify enp1s0 ipv4.addresses 172.4.7.30/24
nmcli connection modify enp1s0 ipv4.gateway 172.4.7.1
nmcli connection modify enp1s0 ipv4.dns 172.4.7.1
nmcli connection modify enp1s0 ipv4.dns-search srv.world
nmcli connection modify enp1s0 ipv4.method manual
nmcli connection down enp1s0; nmcli connection up enp1s0
nmcli device show enp1s0
ip address show

# 禁用IPv6
grubby --update-kernel ALL --args ipv6.disable=1
grubby --info DEFAULT
reboot
ip address show
grubby --update-kernel ALL --remove-args ipv6.disable
```

## 服务状态

```shell
# the list of services that are active now
systemctl -t service
# list of all services
systemctl list-unit-files -t service

systemctl stop nis-domainname
systemctl disable nis-domainname
# to disable and stop with a command, run like follows
systemctl disable --now nis-domainname
```

## 更新系统

```shell
[root@dlp ~]# which yum
/usr/bin/yum
[root@dlp ~]# ll /usr/bin/yum
lrwxrwxrwx. 1 root root 5 Aug 5 2020 /usr/bin/yum -> dnf-3
[root@dlp ~]# which dnf
/usr/bin/dnf
[root@dlp ~]# ll /usr/bin/dnf
lrwxrwxrwx. 1 root root 5 Oct 25 23:06 /usr/bin/dnf -> dnf-3
[root@dlp ~]# ll /usr/bin/dnf-3
-rwxr-xr-x. 1 root root 1942 Oct 25 23:06 /usr/bin/dnf-3
# installed [yum] package
[root@dlp ~]# rpm -q yum
yum-4.10.0-1.el9.noarch
[root@dlp ~]# rpm -ql yum
/etc/dnf/protected.d/yum.conf
/etc/yum.conf
/etc/yum/pluginconf.d
/etc/yum/protected.d
/etc/yum/vars
/usr/bin/yum
/usr/share/man/man1/yum-aliases.1.gz
/usr/share/man/man5/yum.conf.5.gz
/usr/share/man/man8/yum-shell.8.gz
/usr/share/man/man8/yum.8.gz

# included files are all links to [dnf]
[root@dlp ~]# ll /etc/yum.conf /etc/yum/vars /etc/yum/pluginconf.d
lrwxrwxrwx. 1 root root 12 Oct 25 23:06 /etc/yum.conf -> dnf/dnf.conf
lrwxrwxrwx. 1 root root 14 Oct 25 23:06 /etc/yum/pluginconf.d -> ../dnf/plugins
lrwxrwxrwx. 1 root root 11 Oct 25 23:06 /etc/yum/vars -> ../dnf/vars
```

```shell
dnf -y upgrade
```

## Web Admin Console

```shell
systemctl enable --now cockpit.socket
ss -napt
firewall-cmd --list-service
# if [cockpit] is not allowed, set it to allow
[root@dlp ~]# firewall-cmd --add-service=cockpit
success
[root@dlp ~]# firewall-cmd --runtime-to-permanent
success
```

## Vim设置

```shell
dnf -y install vim-enhanced

vi ~/.bashrc
# add alias to the end
alias vi='vim'
source ~/.bashrc

mkdir /vim_backup
[root@dlp ~]# vi ~/.vimrc
" use extended function of vim
" - no compatible with vi
set nocompatible
" specify encoding
set encoding=utf-8
" specify file encoding
set fileencodings=utf-8",iso-2022-jp,sjis,euc-jp
" specify file formats
set fileformats=unix,dos
" take backup
" - if not, specify [ set nobackup ]
set backup
" specify backup directory
set backupdir=/vim_backup
" take 50 search histories
set history=50
" ignore Case
set ignorecase
" distinct Capital if you mix it in search words
set smartcase
" highlights matched words
" - if not, specify [ set nohlsearch ]
set hlsearch
" use incremental search
" - if not, specify [ set noincsearch ]
set incsearch
" show line number
" - if not, specify [ set nonumber ]
set number
" visualize break ( $ ) or tab ( ^I )
set list
" highlights parentheses
set showmatch
" not insert LF at the end of file
set binary noeol
" set auto indent
" - if not, specify [ noautoindent ]
set autoindent
" show color display
" - if not, specify [ syntax off ]
syntax on
" change colors for comments if [ syntax on ] is set
highlight Comment ctermfg=LightCyan
" wrap lines
" - if not, specify [ set nowrap ]
set wrap
```

## sudo设置

```shell
useradd li && echo "123" | passwd --stdin li

visudo
# line 25 : add
# for example, set aliase for the kind of shutdown commands
# 关机重启等
Cmnd_Alias SHUTDOWN = /usr/sbin/halt, /usr/sbin/shutdown, \
/usr/sbin/poweroff, /usr/sbin/reboot, /usr/sbin/init, /usr/bin/systemctl
# 用户管理
Cmnd_Alias USERMGR = /usr/sbin/useradd, /usr/sbin/userdel, /usr/sbin/usermod, \
/usr/bin/passwd

li  ALL=(ALL)       ALL, !SHUTDOWN
%usermgr ALL=(ALL) USERMGR

groupadd usermgr
usermod -G usermgr li
# verify with user
su - li
sudo useradd testuser && echo "123" | passwd --stdin testuser

# add to the end : settings for each user
testuser  ALL=(ALL)       /usr/sbin/visudo

## sudo 日志
visudo
# add to the end
# for example, output logs to [local1] facility
Defaults syslog=local1
[root@dlp ~]# vi /etc/rsyslog.conf
# line 46,47 : add like follows
*.info;mail.none;authpriv.none;cron.none                /var/log/messages
local1.*                                                /var/log/sudo.log

# The authpriv file has restricted access.
authpriv.*                                              /var/log/secure

systemctl restart rsyslog
```

## NTP

```shell
dnf -y install chrony
vi /etc/chrony.conf
# line3 change servers to synchronize
3 #pool 2.centos.pool.ntp.org iburst$
4 pool ntp.aliyun.com iburst$

systemctl enable chronyd
systemctl restart chronyd
# If Firewalld is running, allow NTP service. NTP uses [123/UDP].
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload

chronyc sources

# NTP Client
dnf -y install ntpstat
ntpstat
```

## SSH Server

```shell
vi /etc/ssh/sshd_config

PermitRootLogin yes
systemctl restart sshd
# If Firewalld is running, allow SSH service. SSH uses [22/TCP].
firewall-cmd --add-service=ssh --permanent
firewall-cmd --reload

# ssh client
dnf -y install openssh-clients
ssh root@dlp.srv.world
# ssh后执行命令
ssh root@dlp.srv.world "cat /etc/passwd"

scp ./test.txt root@node01.srv.world:~/
sftp root@node01.srv.world
```





























