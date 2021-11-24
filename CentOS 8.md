# CentOS 8

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

## 修改IP

```shell
# vim /etc/sysconfig/network-scripts/ifcfg-ens33

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=066b4926-b40c-4c28-a5b4-2310d2b96613
DEVICE=ens33
ONBOOT=yes
IPADDR=10.4.7.21
NETMASK=255.255.255.0
GATEWAY=10.4.7.2
DNS1=10.4.7.2
DNS1=8.8.8.8
PREFIX=24

# 重启网卡
nmcli c reload
nmcli c up ens33

ip a
```

## 关闭防火墙

```shell
# 关闭防火墙和selinux
systemctl disable --now firewalld
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```









