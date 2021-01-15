Ubuntu无盘工作站安装详细步骤


【正文第一部分】服务器端（机器名为ubuntu）准备


首先，本文中的服务器端名为ubuntu（IP是192.168.1.88），而客户机端名为netfs，和“天使之翼”文中的命名方法相同，以方便拷贝相关命令和内容。以下显示为root@ubuntu的即指在服务器端操作，而显示为root@netfs的即指在客户机端操作。

【正文第一部分-1】服务器端三大服务

服务器端需准备tftp、dhcp、nfs这三个服务，其过程如下：
（以下全部来自“天使之翼”原文，仅加了必要的注解）

第一步 安装tftp服务器

1 安装
root@ubuntu:/# apt-get install tftpd-hpa

root@ubuntu:/#

2 设置tftpd
root@ubuntu:~# nano /etc/default/tftpd-hpa
\#Defaults for tftpd-hpa
RUN_DAEMON="yes"

\#上面这句表示启动守护进程，tftpd工作
OPTIONS="-l -s /var/lib/tftpboot"
\#上面这句表示tftp客户端能取得的文件所存放的位置

3 启动服务
root@ubuntu:/# /etc/init.d/tftpd-hpa start
Starting HPA's tftpd: in.tftpd.
root@ubuntu:/# ps aux|grep tftp
root 26853 0.0 0.1 2196 288 ? Ss 17:26 0:00 /usr/sbin/in.tftpd -l -s /var/lib/tftpboot
root 26862 0.0 0.2 3180 748 pts/1 R+ 17:27 0:00 grep tftp
root@ubuntu:/#

4 查看服务是否开始工作
root@ubuntu:/# netstat -pna|grep tft
udp 0 0 0.0.0.0:69 0.0.0.0:* 26853/in.tftpd
unix 2 [ ] DGRAM 164700 26853/in.tftpd
root@ubuntu:/#


第二步 安装dhcp服务器

1 服务器环境
root@ubuntu:/# uname -a
Linux ubuntu 2.6.22-14-generic #1 SMP Sun Oct 14 23:05:12 GMT 2007 i686 GNU/Linux
root@ubuntu:/#

2 安装命令
root@ubuntu:/# apt-get install dhcp3-server

3 设置dhcpd工作接口
root@ubuntu:~# nano /etc/default/dhcp3-server
\# Defaults for dhcp initscript
\# sourced by /etc/init.d/dhcp
\# installed at /etc/default/dhcp3-server by the maintainer scripts
\#
\# This is a POSIX shell fragment
\#

\# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
\# Separate multiple interfaces with spaces, e.g. "eth0 eth1".

\# 下面这句用来定义工作接口，如果是多个就中间空格
\# 比如INTERFACES="eth0 eth1 eth2"
INTERFACES="eth0"
（注*上面一行指明服务器端通过哪一块网卡提供dhcp服务）

4 主要设置
root@ubuntu:~# nano /etc/dhcp3/dhcpd.conf

\#
\# Sample configuration file for ISC dhcpd for Debian
\#
\# $Id: dhcpd.conf,v 1.1.1.1 2002/05/21 00:07:44 peloy Exp $
\#

\# The ddns-updates-style parameter controls whether or not the server will
\# attempt to do a DNS update when a lease is confirmed. We default to the
\# behavior of the version 2 packages ('none', since DHCP v2 didn't
\# have support for DDNS.)
ddns-update-style none;

\#下面是全局设置，这里定义的信息全dhcp服务器生效
\#我一般注释掉了，下面可以分不同的子网进行设置
\# option definitions common to all supported networks...
\#option domain-name "apt-get.cn";
\#option domain-name-servers 202.103.0.117, 202.103.24.68;
\#default-lease-time 600;
\#max-lease-time 7200;

\# If this DHCP server is the official DHCP server for the local
\# network, the authoritative directive should be uncommented.
\#authoritative;

\# Use this to send dhcp log messages to a different log file (you also
\# have to hack syslog.conf to complete the redirection).
log-facility local7;

\# No service will be given on this subnet, but declaring it helps the
\# DHCP server to understand the network topology.

\#subnet 10.152.187.0 netmask 255.255.255.0 {
\#}

\# This is a very basic subnet declaration.

\#subnet 10.254.239.0 netmask 255.255.255.224 {
\# range 10.254.239.10 10.254.239.20;
\# option routers rtr-239-0-1.example.org, rtr-239-0-2.example.org;
\#}

\# This declaration allows BOOTP clients to get dynamic addresses,
\# which we don't really recommend.

\#subnet 10.254.239.32 netmask 255.255.255.224 {
\# range dynamic-bootp 10.254.239.40 10.254.239.60;
\# option broadcast-address 10.254.239.31;
\# option routers rtr-239-32-1.example.org;
\#}

\# A slightly different configuration for an internal subnet.
\#subnet设置一个子网192.168.1.0/24
\#range定义可以分配出去的地址为1.50到1.70
\#option domain-name-servers定义dns为202.103.0.117等三个，这里注意每个之间要有个逗号
\#option domain-name定义域名称
\#option routers定义网关地址
\#broadcast-address定义广播地址
\#default-lease-time默认租约时间
\#max-lease-time 最大租约时间
（注*下面这一段是生效部分，请按照实际情况修改）
subnet 192.168.1.0 netmask 255.255.255.0 {
range 192.168.1.50 192.168.1.70;
（注*上行的动态IP范围请不要与系统中已有的dhcp服务器冲突，比如无线路由器上自带的dhcp，但是也不需要把原有的关掉，只要范围不冲突就可以了，因为客户端在启动时会自动使用服务器端的dhcp所分配的地址、而不使用无线路由器上分配的）
option domain-name-servers 202.103.0.117,202.103.24.68,202.103.150.44;
option domain-name "apt-get.cn";
（注*上行的domain-name在联网系统里会起作用，所以请选择一个你肯定不会去访问的名字、即使你并不知道它是否被注册成为域名）
option routers 192.168.1.1;
option broadcast-address 192.168.1.255;
default-lease-time 864000;
max-lease-time 86400000;
filename "pxelinux.0";
（注*上面这一行是要手工加的很关键的信息，实际就是启动无盘工作站网卡的方式，而其中的pxelinux.0其实是一个文件名，下文将谈到这个文件如何生成）
}

\# Hosts which require special configuration options can be listed in
\# host statements. If no address is specified, the address will be
\# allocated dynamically (if possible), but the host-specific information
\# will still come from the host declaration.

\#host passacaglia {
\# hardware ethernet 0:0:c0:5d:bd:95;
\# filename "vmunix.passacaglia";
\# server-name "toccata.fugue.com";
\#}

\# Fixed IP addresses can also be specified for hosts. These addresses
\# should not also be listed as being available for dynamic assignment.
\# Hosts for which fixed IP addresses have been specified can boot using
\# BOOTP or DHCP. Hosts for which no fixed address is specified can only
\# be booted with DHCP, unless there is an address range on the subnet
\# to which a BOOTP client is connected which has the dynamic-bootp flag
\# set.
\#host fantasia {
\# hardware ethernet 08:00:07:26:c0:a5;
\# fixed-address fantasia.fugue.com;
\#}

\# You can declare a class of clients and then do address allocation
\# based on that. The example below shows a case where all clients
\# in a certain class get addresses on the 10.17.224/24 subnet, and all
\# other clients get addresses on the 10.0.29/24 subnet.

\#class "foo" {
\# match if substring (option vendor-class-identifier, 0, 4) = "SUNW";
\#}

\#shared-network 224-29 {
\# subnet 10.17.224.0 netmask 255.255.255.0 {
\# option routers rtr-224.example.org;
\# }
\# subnet 10.0.29.0 netmask 255.255.255.0 {
\# option routers rtr-29.example.org;
\# }
\# pool {
\# allow members of "foo";
\# range 10.17.224.10 10.17.224.250;
\# }
\# pool {
\# deny members of "foo";
\# range 10.0.29.10 10.0.29.230;
\# }
\#}

5 启动服务器
root@ubuntu:/# /etc/init.d/dhcp3-server start
\* Starting DHCP server dhcpd3 [ OK ]
root@ubuntu:/#

6 查看服务是否已经正常监听
root@ubuntu:/# netstat -aunp|grep dhcp
udp 0 0 0.0.0.0:67 0.0.0.0:* 23011/dhcpd3
已经在67号udp口上开始监听了

第三步 安装配置nfs服务器

1 安装
root@ubuntu:/# apt-get install nfs-common nfs-kernel-server nfs-client

2 配置
root@ubuntu:~# nano /etc/exports

\# /etc/exports: the access control list for filesystems which may be exported
\# to NFS clients. See exports(5).
\#
\# Example for NFSv2 and NFSv3:
\# /srv/homes hostname1(rw,sync) hostname2(ro,sync)
\#
\# Example for NFSv4:
\# /srv/nfs4 gss/krb5i(rw,sync,fsid=0,crossmnt)
\# /srv/nfs4/homes gss/krb5i(rw,sync)
/home/li/netboot 192.168.1.0/24(rw,no_root_squash,sync)
（注*上面这一行是服务器端提供的磁盘空间的位置，可以是服务器的任一目录，建议将一个单独的磁盘分区挂在这个目录下。但是请注意：这个服务器端的/home/li/netboot并不是将来客户端的虚拟根目录，因为在/home/li/netboot下面将会有一个名为root的子目录，而这个/home/li/netboot/root才是本文中的客户端的虚拟根目录，在启动完成后、实际运行过程中，工作就仅局限在/home/li/netboot/root中了。建立root的问题下文将会讲到）

3 启动nfs或者重新加载
启动nfs
root@ubuntu:/# /etc/init.d/nfs-kernel-server start
\* Exporting directories for NFS kernel daemon...
exportfs: /etc/exports [1]: Neither 'subtree_check' or 'no_subtree_check' specified for export "192.168.1.0/24:/home/li/netboot".
Assuming default behaviour ('no_subtree_check').
NOTE: this default has changed since nfs-utils version 1.0.x
...done.
\* Starting NFS kernel daemon
...done.
如果是修改了/etc/exports 配置文件，不需要重新启动nfs服务器，只需要刷新一下，命令如下
root@ubuntu:/# exportfs -r
exportfs: /etc/exports [1]: Neither 'subtree_check' or 'no_subtree_check' specified for export "192.168.1.0/24:/home/li/netboot".
Assuming default behaviour ('no_subtree_check').
NOTE: this default has changed since nfs-utils version 1.0.x


【正文第一部分-2】服务器端其他必要准备

服务器端还需准备上文提到的pxelinux.0（也即远程启动客户端网卡的核心模块），其过程如下：
（以下全部来自“天使之翼”原文，仅加了必要的注解）

第四步 安装syslinux
1 安装syslinux，其实也就是为了要里面的pxelinux部分的文件
root@ubuntu:/# apt-get install syslinux
正在读取软件包列表... 完成
正在分析软件包的依赖关系树
Reading state information... 完成
syslinux 已经是最新的版本了。
共升级了 0 个软件包，新安装了 0 个软件包，要卸载 0 个软件包，有 0 个软件未被升级。

2 拷贝pxelinux.0文件到tftpboot目录
root@ubuntu:/# cp /usr/lib/syslinux/pxelinux.0 /var/lib/tftpboot/
root@ubuntu:/#
（注*本文中将会涉及两个不同的tftpboot目录，这里是其中的第一个，请不要和下文的/home/li/netboot/tftpboot混淆）

3 在tftpboot目录建立pxelinux.cfg目录，然后在pxelinux.cfg目录下建立default文件
也可以是以某个ip地址为文件名称
root@ubuntu:/# nano /var/lib/tftpboot/pxelinux.cfg/default
(注*所谓default其实是假设系统中只有一台客户端时的简易操作，如果有多台客户端，则需建立多个文件、以客户端各自的IP地址为文件名。服务器也就是通过这种方法来区别不同的客户端，以导入各自不同的虚拟根目录)

DEFAULT ubuntu
LABEL ubuntu
kernel linux
append initrd=initrd.nfs root=/dev/nfs nfsroot=192.168.1.88:/home/li/netboot/root ip=dhcp rw
（注*注意这一行出现的/home/li/netboot/root就是上文说到的default客户端将来的虚拟根目录在服务器端上的位置，而非其上级的/home/li/netboot。当有多个客户端时，其各自的以IP命名的文件中，这一行的目录路径应该是不同的、以此来区别各自独立的虚拟根目录）
PROMPT 1
TIMEOUT 3


【正文第三部分】客户端从没有磁盘开始安装

从这里开始，是本人全新编写的部分，主要参考了“Ubuntu高地”一文中的简要描述，对该文没有讲明的一些细节，下文将着重进行补充。首先让我们回顾一下，在【正文第一部分】服务器端准备完成后，我们已经作了哪些工作：
1、服务器端已经运行了三大服务tftp、dhcp和nfs
2、PXE启动必须的pxelinux.0 已经生成并且放在服务器端var/lib/tftpboot里
3、与每个客户端对应的虚拟根目录位置信息已经在服务器端的/var/lib/tftpboot/pxelinux.cfg/这个目录下的对应文件里配置好
所以说，我们下面要做的，实际就是要在客户机端没有磁盘的情况下，在服务器端的为其已经准备好（即nfs服务已经export）的目录里，生成其可以使用的操作系统，体现为根目录下常见的bin、usr、home……文件结构。对应［正文第二部分］的有盘客户端安装的内容（未阅读该部分也无妨，见下列列表即可），这个系统里必须有：
1、能够作为nfs-client，因为启动后虚拟根目录需要从服务器端nfs出来
2、必须有initramfs-tools，并且initramfs.conf要修改为支持nfs启动
3、修改fstab、mtab、hosts、hostname、interfaces、udev里的rules以符合客户机端正常运行时的情况
以上三条体现于虚拟根目录下，也就是服务器端的/home/li/netboot/root目录，但是别忘了还有与其平行的/home/li/netboot/tftpboot下必须有：
4、启动操作系统的两个文件initrd.nfs和linux（是vmlinuz的改名）
所以，以下的工作就是围绕上述1—4这四个目标来做：

【正文第三部分-1】为无盘客户机端建立基本系统而在服务器端进行的工作

第一步 利用debootstrap生成一个基本的可登录系统

因为客户机端一片空白、且没有磁盘，所以这一步要在服务器端进行，目标是在/home/li/netboot/root下生成一个最基本的可登录系统。首先：
root@ubuntu:/# cd /home/li/netboot;mkdir root tftpboot创建两个目录
root@ubuntu:/#debootstrap --arch=i386 hardy /home/li/netboot/root http://debian.nctu.edu.tw/ubuntu
（注意arch前面是连续两个-，而http://debian.nctu.edu.tw/ubuntu是我这里最快的源，也可以改成其他任何一个源）

该命令的结果是在/home/li/netboot/root里面生成了一个可以字符登录的最基本的系统，该系统不依赖于任何特定的客户机硬件平台，只要是i386芯片的客户机端在接下来都可以登录进去。当然，如果是amd64的芯片，则在上述debootstrap命令后跟的就是—arch=amd64了。
经过检查发现，nfs-client和initramfs-tools的功能已经随着基本的系统而自然生成，下面只需修改/home/li/netboot/root/etc/initramfs-tools/initramfs.conf：
\# nfs - Boot using an NFS drive as the root of the drive.
\#
BOOT=nfs
\#
此时已经完成了我们四大目标中的1和2

第二步 修改几个重要的配置文件

/home/li/netboot/root/etc/fstab文件修改成：
proc /proc proc defaults 0 0
192.168.1.88:/home/li/netboot/root / nfs defaults,rw 0 0
（注意：与上文“天使之翼”只保留proc那一行不同，我在客户端将来的fstab里写明了要用到的nfs根目录，这样更保险）

/home/li/netboot/root/etc/hosts文件修改成：
127.0.0.1 localhost
127.0.1.1 netfs
（注意：netfs是修改后的结果，在修改前这个位置实际是你现在所在的服务器的名字）

/home/li/netboot/root/etc/hostname文件修改成：
netfs
（修改前这个位置实际也是你现在所在的服务器的名字ubuntu）

/home/li/netboot/root/etc/network/interfaces文件修改成：
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
address 192.168.1.28
netmask 255.255.255.0
gateway 192.168.1.1
（注意：与“天使之翼”不同的是我直接指定了客户机的静态IP192.168.1.28，在ubuntu里静态IP的定义好像一定要这样写得很完全）

/home/li/netboot/root/etc/mtab文件修改成：
192.168.1.88:/home/li/netboot/root / nfs rw 0 0
proc /proc proc rw 0 0
devpts /dev/pts devpts rw,gid=5,mode=620 0 0
（注意：与“天使之翼”删除mtab不同，原因与上述fstab的修改一样，实践证明可行。特别注意第1行末尾与fstab第一行是不一样的）

/home/li/netboot/root/etc/udev/rules.d/70-persistent-net.rules文件的修改：
root@ubuntu:/#:>/home/li/netboot/root/etc/udev/rules.d/70-persistent-net.rules
（这一步其实可以省略，因为在基本的arch系统里这个文件本来就是空的）

除此之外，还有一个很重要的步骤，就是将服务器/etc/apt/sources.list的内容拷贝到/home/li/netboot/root/etc/apt/sources.list里，否则到时候客户机端的源里只有生成arch-i386时的一条（在本文中就是http://debian.nctu.edu.tw/ubuntu），会导致很多软件包找不到。

这样，我们就完成了四大目标中的3

第三步 创建支持nfs的initrd.nfs、linux（vmlinuz）文件

这一步同样需在服务器端完成，只要服务器端本身具有nfs功能（因为我们在一开头已经配置了服务器端的nfs服务），那么可以通过以下命令生成initrd.nfs：
root@ubuntu:/# mkinitramfs -o /home/li/netboot/tftpboot/initrd.nfs
至于linux（也就是vmlinuz），其实内核文件随处可见并且是通用的，可以：
\#cp /boot/vmlinuz-2.6.24-16-generic /home/li/netboot/tftpboot/linux
也就是将服务器端本身的vmlinuz拷贝到/home/li/netboot/tftpboot中并且改名为系统所要求的linux。
当然随后必要忘记把两个文件拷贝到/var/lib里的tftp目录下：
root@ubuntu:~# cp /home/li/netboot/tftpboot/* /var/lib/tftpboot/

就这样，四大目标中的第4也完成了。如果一切正常，此时无盘的客户机端应该可以通过PXE启动方式，进入一个字符终端。

【正文第三部分-2】完全无盘客户机端的后续安装

此时，我们终于可以来到客户机端进行后续的工作。如果一切正常，客户机端可以用root用户和空口令登入（这个root是无盘客户机端的root，从现在起客户机端的用户系统与服务器端已没有关系），第一步就应该是用#passwd给root建一个口令，否则在后面xsession启动的时候，没有口令的root是无法登录的。
接下来的工作其实有点类似从一个纯字符的ubuntu服务器上建立x windows应用，只不过后者是在本地硬盘上安装文件，而我们的无盘工作站是把文件通过nfs远程安装到服务器端上的虚拟根目录（也就是/home/li/netboot/root）里。
当然，我们开始工作的这个只有arch的系统，比已经是字符型服务器的系统少了很多东西，所以需要一步步地添加进去。在“Ubuntu高地”一文中，这部分是最容易让人迷惑的，因为文章里看起来很容易，但是如果不是亲自尝试，就很难想象会有那么多细节性的问题，而每一个问题带来的都是字符终端失去反应的后果。
现在让我们按顺序来进行配置：


第一步 安装内核
对，安装内核。arch-i386虽然已经可以登录，但是内核的部分其实还缺很多，所以需要安装，具体来说，装的应该是全功能系统需要的很多库和模块。安装的命令是：
root@netfs:/#apt-get update
root@netfs:/#apt-get dist-upgrade
root@netfs:/#apt-get install linux-image-2.6.24-16-generic
前面两条是更新源、升级发行版，最后1条才是安装内核包。注意：内核一定要用linux-image-2.6.24-16-generic而不能用linux-image-2.6.24-16-386，虽然两者其实是一样的，但是因为两者在/home/li/netboot/root/lib/modules/里生成的目录名是不同的，只有-generic生成的那个不会带来后续的路径问题。

第二步 安装console-data
root@netfs:/#apt-get install console-data
此处安装过程中会有类似dos那样的彩色字符窗口来设置键盘布局，需要按tab键选择ok等按钮、用方向键选择项目，鼠标此时是不起作用的。

第三步 安装xserver-xorg
root@netfs:/#apt-get install xserver-xorg

第四步 安装ubuntu-desktop
root@netfs:/#apt-get install ubuntu-desktop
注意：按照“Ubuntu高地”的文章，在xserver-xorg完成后就可以安装gnome或者xfce了，但实际上是不行的，如果那样做的话，在gnome安装的最后会停下来报错说找不到fontpath，而fontpath就是在安装ubuntu-desktop的时候自动配置的，所以到时候还是要补充安装ubuntu-desktop。那么按照报错的信息提示，所缺乏的东西应该提前安装好，也就是ubuntu-desktop应该在gnome之前安装。
装完ubuntu-desktop之后，其实已经可以用startx命令进入gnome桌面，但是实际上还缺一些文件，所以最好再继续安装gnome或xfce。

第五步 安装gnome或者xfce
这时候就可以安装真正完整的桌面系统了。我个人建议是安装gnome，因为就像“Ubuntu高地”说的，xfce在我们这种安装方式下会缺少很多包，需要手工补齐，所以不如利用gnome把这些包都装上，在装完gnome之后还是可以继续安装xfce供选用。
安装 gnome的时候，99%可能遇到臭名昭著的“gnome-keyring-manager”问题。解决的方法也很简单，因为我们有服务器端，可以直接在服务器端下载后拷贝到/home/li/netboot/root下、也就是客户机端此时的根目录下，然后在客户机端用dpkg -i命令手工安装（这一步应该在安装gnome之前就做好）。下载的地址是：
[http://packages.debian.org/gnome-keyring-manager](https://packages.debian.org/gnome-keyring-manager)
唯一需要注意的是下载的版本，不能下载稳定版的2-16-0版、而只能下载2-20-0的非稳定版，因为这是gnome需要的最低版本。正因为是非稳定版，所以安装这个包的时候会报错，但是其实不影响下面的工作，需要的东西还是会安装。这个问题解决后，就可以：
root@netfs:/#apt-get install gnome
完成后如果还需要xfce，可以继续：
root@netfs:/#apt-get install xubuntu-desktop

第六步 启动xwindows
在gnome或者xfce顺利安装完成后，用以下命令启动x登录界面：
root@netfs:/#startx
然后进入一个英文的xsession，进去后再更改语言、增删其他的软件包等就比较简单了。到此我们的工作可以说是顺利完成了。


【正文第四部分】高阶配置


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
（注*也就是修改其中的nfsroot=192.168.1.88:/home/li/netboot/root ip=dhcp rw这一行，用不同客户端的虚拟根目录所在目录更改/home/li/netboot/root这个default客户端所用的路径，当然前提是在nfs自身的/etc/exports里面已经开放了这个目录的共享。
随后，如果客户机与第一台default是同型号的，可以直接把后者已经生成的虚拟根目录里的文件系统在服务器端完整拷贝到相应的目录的，但是要记得修改hosts、hostname、interfaces等配置文件以适应前者的情况，以免启动后发生IP、机器名冲突等情况）