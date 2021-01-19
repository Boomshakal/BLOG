# Ubuntu无盘工作站安装详细步骤

## 服务器端三大服务

### tftp

```shell
apt-get install tftpd-hpa

vim /etc/default/tftpd-hpa
# /etc/default/tftpd-hpa
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/var/lib/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="-l -s"

/etc/init.d/tftpd-hpa start
netstat -pna|grep tftp
```

### DHCP

```shell
apt-get install isc-dhcp-server

vim /etc/dhcp/dhcpd.conf
allow booting;
allow bootp;

subnet 10.4.7.0 netmask 255.255.255.0 {
  range 10.4.7.50 10.4.7.80;
  option broadcast-address 10.4.7.255;
  option routers 10.4.7.2;
  option domain-name-servers 10.4.7.2;

  filename "/pxelinux.0";
}

sudo service isc-dhcp-server restart
netstat -aunp|grep dhcp
```

### nfs

```shell
apt-get install nfs-common nfs-kernel-server nfs-client
vim /etc/exports
/home/li/netboot 10.4.7.0/24(rw,no_root_squash,no_subtree_check,sync)

/etc/init.d/nfs-kernel-server start
exportfs -r
```

### 服务器端其他必要准备

```shell
apt-get install syslinux pxelinux initramfs-tools

sudo cp /usr/lib/PXELINUX/pxelinux.0 /var/lib/tftpboot
sudo mkdir -p /var/lib/tftpboot/boot
sudo cp -r /usr/lib/syslinux/modules/bios /var/lib/tftpboot/boot/isolinux
mkdir /var/lib/tftpboot/pxelinux.cfg
vim /var/lib/tftpboot/pxelinux.cfg/default
DEFAULT ubuntu
LABEL ubuntu
kernel linux
append initrd=initrd.nfs root=/dev/nfs nfsroot=10.4.7.31:/home/li/netboot/root ip=dhcp rw
PROMPT 1
TIMEOUT 3
```

### 为无盘客户机端建立基本系统

```shell
cd /home/li/netboot
mkdir root tftpboot

debootstrap --arch=amd64 focal /home/li/netboot/root http://mirrors.aliyun.com/ubuntu

vim /home/li/netboot/root/etc/fstab
# UNCONFIGURED FSTAB FOR BASE SYSTEM
proc /proc proc defaults 0 0
10.4.7.31:/home/li/netboot/root / nfs defaults,rw 0 0

mkinitramfs -o /home/li/netboot/tftpboot/initrd.nfs
cp /boot/vmlinuz-`uname -r` /home/li/netboot/tftpboot/linux

cp /home/li/netboot/tftpboot/* /var/lib/tftpboot/
chmod 777 /var/lib/tftpboot/

mv /home/li/netboot/root/etc/apt/sources.list /home/li/netboot/root/etc/apt/sources.list.bak

cp /etc/apt/sources.list /home/li/netboot/root/etc/apt/

# 切换客户端系统
cd /home/li/netboot/root
chroot .
cat etc/issue
uname -a

adduser li
passwd root
```

## 客户端配置

### 安装ubuntu-desktop

```shell
apt-get update
apt-get install ubuntu-desktop
```

## 高阶配置


高阶配置除了nfs的安全性问题，还有多个nfs远程目录的挂载（即除了虚拟根目录外，将服务器的其他目录开放给客户机端），这部分请仔细研究nfs服务本身就很容易解决。
在实际应用中比较重要的高阶配置，就是上文已经提到过的多个客户机端同时存在，此时主要涉及服务器端的dhcp的设置和对应的pxelinux配置文件。在此引用“天使之翼”的补充说明：

常常无盘网络有多个客户端，这个时候我们需要修改dhcp服务器和tftp下的pxelinux.cfg下的设置
我们首先来说一下dhcp3-server的修改
首先我们打开全局设置的部分，统一的设置网关 dns 租约期限等统一的信息
\#设置所在域的名称
option domain-name "apt-get.cn";
\#设置dns解析服务器
option domain-name-servers 202.103.0.117, 202.103.24.68;
\#下面的时间使用-1表示永久租约
default-lease-time -1;
max-lease-time -1;
然后找到我们先设置的地方
没改的时候如下
\# A slightly different configuration for an internal subnet.
subnet 192.168.1.0 netmask 255.255.255.0 {
range 192.168.1.50 192.168.1.70;
option domain-name-servers 202.103.0.117,202.103.24.68,202.103.150.44;
option domain-name "apt-get.cn";
option routers 192.168.1.1;
option broadcast-address 192.168.1.255;
default-lease-time 864000;
max-lease-time 86400000;
filename "pxelinux.0";
}

修改方式如下
subnet 192.168.1.0 netmask 255.255.255.0 {
range 192.168.1.50 192.168.1.50
option routers 192.168.1.1;
option broadcast-address 192.168.1.255;

host A01 {
hardware ethernet 00:0c:29:81:bf:41;
option host-name "A001";
fixed-address 192.168.1.50;
filename "pxelinux.0";
}

host A02 {
hardware ethernet 00:0c:29:df:38:be;
option host-name "A002";
fixed-address 192.168.1.51;
filename "pxelinux.0";
}

}

多客户机的dhcpd的修改方法就是上面这样，下面我们设置一下pxelinux.cfg里面的内容
root@ubuntu:/var/lib/tftpboot/pxelinux.cfg# ls
01-00-0c-29-81-bf-41 01-00-0c-29-df-38-be default
root@ubuntu:/var/lib/tftpboot/pxelinux.cfg#
看一下，把default文件复制一个文件名修改为
01-开头，后面是无盘客户机的mac地址，看清楚中间是-不是: 字母必须是小写，否则启动报错
也可以把名字修改为16进制的数字，比如192.168.1.50
修改为C0A80132
比如192转成了16进制的C0
168转成了16进制的A8
这里必须是大写，小写启动的时候报错
如果启动的时候nfs连接的位置不同，记得修改这个文件里面的内容哦
（注*也就是修改其中的nfsroot=192.168.1.88:/home/cache/netboot/root ip=dhcp rw这一行，用不同客户端的虚拟根目录所在目录更改/home/cache/netboot/root这个default客户端所用的路径，当然前提是在nfs自身的/etc/exports里面已经开放了这个目录的共享。
随后，如果客户机与第一台default是同型号的，可以直接把后者已经生成的虚拟根目录里的文件系统在服务器端完整拷贝到相应的目录的，但是要记得修改hosts、hostname、interfaces等配置文件以适应前者的情况，以免启动后发生IP、机器名冲突等情况）











