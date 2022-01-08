## ceph cluster

Generate SSH key-pair on [Monitor Daemon] Node (call it Admin Node on here) and set it to each Node.
Configure key-pair with no-passphrase as [root] account on here.

```shell
ssh-keygen
vi ~/.ssh/config
Host k3s-m1
    Hostname 10.4.7.21
    User root
Host k3s-m2
    Hostname 10.4.7.22
    User root
Host k3s-m3
    Hostname 10.4.7.23
    User root
    
chmod 600 ~/.ssh/config
ssh-copy-id k3s-m1
ssh-copy-id k3s-m2
ssh-copy-id k3s-m3
```
Install Ceph to each Node from Admin Node.
```
for NODE in k3s-m1 k3s-m2 k3s-m3
do
    ssh $NODE "apt update; apt -y install ceph"
done 
```

Configure [Monitor Daemon], [Manager Daemon] on Admin Node.
```shell
uuidgen
72840c24-3a82-4e28-be87-cf9f905918fb

# create new config
# file name ⇒ (any Cluster Name).conf
# set Cluster Name [ceph] (default) on this example ⇒ [ceph.conf]
vi /etc/ceph/ceph.conf
[global]
# specify cluster network for monitoring
cluster network = 10.4.7.0/24
# specify public network
public network = 10.4.7.0/24
# specify UUID genarated above
fsid = 72840c24-3a82-4e28-be87-cf9f905918fb
# specify IP address of Monitor Daemon
mon host = 10.4.7.21
# specify Hostname of Monitor Daemon
mon initial members = k3s-m1
osd pool default crush rule = -1
# mon.(Node name)
[mon.k3s-m1]
# specify Hostname of Monitor Daemon
host = k3s-m1
# specify IP address of Monitor Daemon
mon addr = 10.4.7.21
# allow to delete pools
mon allow pool delete = true

# generate secret key for Cluster monitoring
ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
# generate secret key for Cluster admin
ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
# generate key for bootstrap
ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'
# import generated key
ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
# generate monitor map
FSID=$(grep "^fsid" /etc/ceph/ceph.conf | awk {'print $NF'})
NODENAME=$(grep "^mon initial" /etc/ceph/ceph.conf | awk {'print $NF'})
NODEIP=$(grep "^mon host" /etc/ceph/ceph.conf | awk {'print $NF'})
monmaptool --create --add $NODENAME $NODEIP --fsid $FSID /etc/ceph/monmap
# create a directory for Monitor Daemon
# directory name ⇒ (Cluster Name)-(Node Name)
mkdir /var/lib/ceph/mon/ceph-k3s-m1
# assosiate key and monmap to Monitor Daemon
# --cluster (Cluster Name)
ceph-mon --cluster ceph --mkfs -i $NODENAME --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring

chown ceph. /etc/ceph/ceph.*
chown -R ceph. /var/lib/ceph/mon/ceph-k3s-m1 /var/lib/ceph/bootstrap-osd
systemctl enable --now ceph-mon@$NODENAME
# enable Messenger v2 Protocol
ceph mon enable-msgr2
# enable Placement Groups auto scale module
ceph mgr module enable pg_autoscaler
# create a directory for Manager Daemon
# directory name ⇒ (Cluster Name)-(Node Name)
mkdir /var/lib/ceph/mgr/ceph-k3s-m1
# create auth key
ceph auth get-or-create mgr.$NODENAME mon 'allow profile mgr' osd 'allow *' mds 'allow *'
ceph auth get-or-create mgr.k3s-m1 | tee /etc/ceph/ceph.mgr.admin.keyring
cp /etc/ceph/ceph.mgr.admin.keyring /var/lib/ceph/mgr/ceph-k3s-m1/keyring
chown ceph. /etc/ceph/ceph.mgr.admin.keyring
chown -R ceph. /var/lib/ceph/mgr/ceph-k3s-m1
systemctl enable --now ceph-mgr@$NODENAME

# close auth_allow_insecure_global_id_reclaim
ceph config set mon auth_allow_insecure_global_id_reclaim false
# 允许删除pool
ceph config set mon mon_allow_pool_delete true
```
Confirm Cluster status
```
ceph -s
```
Configure OSD (Object Storage Device) to each Node from Admin Node.
Block devices ([/dev/sdb] on this example) are formatted for OSD, Be careful if some existing data are saved.
```
for NODE in k3s-m1 k3s-m2 k3s-m3
do
    if [ ! ${NODE} = "k3s-m1" ]
    then
        scp /etc/ceph/ceph.conf ${NODE}:/etc/ceph/ceph.conf
        scp /etc/ceph/ceph.client.admin.keyring ${NODE}:/etc/ceph
        scp /var/lib/ceph/bootstrap-osd/ceph.keyring ${NODE}:/var/lib/ceph/bootstrap-osd
    fi
    ssh $NODE \
    "chown ceph. /etc/ceph/ceph.* /var/lib/ceph/bootstrap-osd/*; \
    parted --script /dev/sdb 'mklabel gpt'; \
    parted --script /dev/sdb "mkpart primary 0% 100%"; \
    ceph-volume lvm create --data /dev/sdb1"
done 
```

```shell
# confirm cluster status
# that's OK if [HEALTH_OK]
ceph -s
# confirm OSD tree
ceph osd tree
ceph df 
ceph osd df 
```



## Block Device

```shell
# transfer public key
root@k3s-m1:~# ssh-copy-id dlp
# install required packages
root@k3s-m1:~# ssh dlp "apt -y install ceph-common"
# transfer required files to Client Host
root@k3s-m1:~# scp /etc/ceph/ceph.conf dlp:/etc/ceph/
root@k3s-m1:~# scp /etc/ceph/ceph.client.admin.keyring dlp:/etc/ceph/
root@k3s-m1:~# ssh dlp "chown ceph. /etc/ceph/ceph.*"
```

```shell
# create default RBD pool [rbd]
root@dlp:~# ceph osd pool create rbd 128
# enable Placement Groups auto scale mode
root@dlp:~# ceph osd pool set rbd pg_autoscale_mode on
# initialize the pool
root@dlp:~# rbd pool init rbd
root@dlp:~# ceph osd pool autoscale-status
# create a block device with 10G
root@dlp:~# rbd create --size 10G --pool rbd rbd01
# confirm
root@dlp:~# rbd ls -l
# map the block device
root@dlp:~# rbd map rbd01
# confirm
root@dlp:~# rbd showmapped
# format with XFS
root@dlp:~# mkfs.xfs /dev/rbd0
root@dlp:~# mount /dev/rbd0 /mnt
root@dlp:~# df -hT
```

```shell
# unmount
root@dlp:~# unmount /mnt
# unmap
root@dlp:~# rbd unmap /dev/rbd/rbd/rbd01
# delete a block device
root@dlp:~# rbd rm rbd01 -p rbd
# delete a pool
# ceph osd pool delete [Pool Name] [Pool Name] ***
root@dlp:~# ceph osd pool delete rbd rbd --yes-i-really-really-mean-it
```

## File System

```shell
# transfer public key
root@k3s-m1:~# ssh-copy-id dlp
# install required packages
root@k3s-m1:~# ssh dlp "apt -y install ceph-fuse"
# transfer required files to Client Host
root@k3s-m1:~# scp /etc/ceph/ceph.conf dlp:/etc/ceph/
ceph.conf                                     100%  195    98.1KB/s   00:00
root@k3s-m1:~# scp /etc/ceph/ceph.client.admin.keyring dlp:/etc/ceph/
ceph.client.admin.keyring                     100%  151    71.5KB/s   00:00
root@k3s-m1:~# ssh dlp "chown ceph. /etc/ceph/ceph.*"
```

```shell
# create directory
# directory name ⇒ (Cluster Name)-(Node Name)
root@k3s-m1:~# mkdir -p /var/lib/ceph/mds/ceph-k3s-m1
root@k3s-m1:~# ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-k3s-m1/keyring --gen-key -n mds.k3s-m1
creating /var/lib/ceph/mds/ceph-k3s-m1/keyring
root@k3s-m1:~# chown -R ceph. /var/lib/ceph/mds/ceph-k3s-m1
root@k3s-m1:~# ceph auth add mds.k3s-m1 osd "allow rwx" mds "allow" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-k3s-m1/keyring
added key for mds.k3s-m1
root@k3s-m1:~# systemctl enable --now ceph-mds@k3s-m1
```

```shell
root@k3s-m1:~# ceph osd pool create cephfs_data 64
pool 'cephfs_data' created
root@k3s-m1:~# ceph osd pool create cephfs_metadata 64
pool 'cephfs_metadata' created
root@k3s-m1:~# ceph fs new cephfs cephfs_metadata cephfs_data
new fs with metadata pool 4 and data pool 3
root@k3s-m1:~# ceph fs ls
ame: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
root@k3s-m1:~# ceph mds stat
cephfs:1 {0=k3s-m1=up:active}
root@k3s-m1:~# ceph fs status cephfs
cephfs - 0 clients
======
RANK  STATE    MDS       ACTIVITY     DNS    INOS
 0    active  k3s-m1  Reqs:    0 /s    10     13
      POOL         TYPE     USED  AVAIL
cephfs_metadata  metadata  1536k  74.9G
  cephfs_data      data       0   74.9G
MDS version: ceph version 15.2.14 (d289bbdec69ed7c1f516e0a093594580a76b78d0) octopus (stable)
```

```shell
# Base64 encode client key
root@dlp:~# ceph-authtool -p /etc/ceph/ceph.client.admin.keyring > admin.key
root@dlp:~# chmod 600 admin.key
root@dlp:~# mount -t ceph k3s-m1:6789:/ /mnt -o name=admin,secretfile=admin.key
root@dlp:~# df -hT
Filesystem                        Type      Size  Used Avail Use% Mounted on
udev                              devtmpfs  1.9G     0  1.9G   0% /dev
tmpfs                             tmpfs     394M  1.1M  393M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv ext4       25G  3.1G   21G  14% /
tmpfs                             tmpfs     2.0G     0  2.0G   0% /dev/shm
tmpfs                             tmpfs     5.0M     0  5.0M   0% /run/lock
tmpfs                             tmpfs     2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/loop0                        squashfs   55M   55M     0 100% /snap/core18/1880
/dev/vda2                         ext4      976M  197M  713M  22% /boot
/dev/loop1                        squashfs   56M   56M     0 100% /snap/core18/1885
/dev/loop2                        squashfs   72M   72M     0 100% /snap/lxd/16100
/dev/loop3                        squashfs   71M   71M     0 100% /snap/lxd/16926
/dev/loop4                        squashfs   30M   30M     0 100% /snap/snapd/8542
/dev/loop5                        squashfs   30M   30M     0 100% /snap/snapd/8790
tmpfs                             tmpfs     394M     0  394M   0% /run/user/0
10.4.7.21:6789:/                  ceph       75G     0   75G   0% /mnt
```

## Dashboard

```shell
root@k3s-m1:~# apt -y install ceph-mgr-dashboard
root@k3s-m1:~# ceph mgr module enable dashboard
root@k3s-m1:~# ceph mgr module ls | grep -A 5 enabled_modules
    "enabled_modules": [
        "dashboard",
        "iostat",
        "restful"
    ],
    "disabled_modules": [

# create self-signed certificate
root@k3s-m1:~# ceph dashboard create-self-signed-cert
Self-signed certificate created
# create a user for Dashboard
# dashboard ac-user-create <username> [<rolename>] -i <file>
root@k3s-m1:~# echo "P@ssw0rd" > password.txt
root@k3s-m1:~# ceph dashboard ac-user-create admin  administrator -i password.txt
{"username": "admin", "password": "$2b$12$88au7Ki.BMT/3SCYaOzHquo8O.gua3WCWaumsot3oQx58x/Fh.JH.", "roles": ["administrator"], "name": null, "email": null, "lastUpdate": 1636689093, "enabled": true, "pwdExpirationDate": null, "pwdUpdateRequired": false}

# confirm Dashboard URL
root@k3s-m1:~# ceph mgr services
{
    "dashboard": "https://k3s-m1:8443/"
}
```



## cephadm

| 主机名 | public-ip | **磁盘**                   | **角色**                            |
| ------ | --------- | -------------------------- | ----------------------------------- |
| node1  | 10.4.7.21 | 系统盘: sda<br/>osd盘: sdb | cephadm,monitor,mgr,rgw,mds,osd,nfs |
| node2  | 10.4.7.22 | 系统盘: sda<br/>osd盘: sdb | monitor,mgr,rgw,mds,osd,nfs         |
| node3  | 10.4.7.23 | 系统盘: sda<br/>osd盘: sdb | monitor,mgr,rgw,mds,osd,nfs         |

### 基础配置
```shell
# Centos版本
cat /etc/redhat-release
CentOS Linux release 8.4.2105
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

# ceph octopus版本源
dnf -y install centos-release-ceph-octopus
# python3安装
dnf install -y python3
```

```shell
hostnamectl set-hostname node1
hostnamectl set-hostname node2
hostnamectl set-hostname node3


cat >> /etc/hosts <<EOF
10.4.7.21 node1
10.4.7.22 node2
10.4.7.23 node3
EOF

# 关闭防火墙和selinux
systemctl disable --now firewalld
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# 配置时间同步
yum install -y chrony
systemctl enable --now chronyd
```

### 安装docker

```shell
#配置阿里云yum源
dnf config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#安装containerd.io
dnf install -y https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.13-3.1.el7.x86_64.rpm

#安装docker-ce
dnf install -y docker-ce

#配置docker镜像加速
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": ["https://b33dfgq9.mirror.aliyuncs.com"]
}
EOF

#启动docker服务
systemctl enable --now docker

# docker version
# docker info
```

### 安装cephadm

```shell
dnf -y install cephadm

[root@node1 ~]# which cephadm
/usr/sbin/cephadm

[root@node1 ~]# cephadm --help

[root@node1 ~]# cephadm version

mkdir -p /etc/ceph
cephadm bootstrap --mon-ip 10.4.7.21
```

- 在本地主机上为新集群创建monitor 和 manager daemon守护程序。
- 为Ceph集群生成一个新的SSH密钥，并将其添加到root用户的/root/.ssh/authorized_keys文件中。
- 将与新群集进行通信所需的最小配置文件保存到/etc/ceph/ceph.conf。
- 向/etc/ceph/ceph.client.admin.keyring写入client.admin管理（特权！）secret key的副本。
- 将public key的副本写入/etc/ceph/ceph.pub

```shell
[root@node1 ~]# ll /etc/ceph/
total 12
-rw------- 1 root root  63 Jun 20 08:05 ceph.client.admin.keyring
-rw-r--r-- 1 root root 177 Jun 20 08:05 ceph.conf
-rw-r--r-- 1 root root 595 Jun 20 08:05 ceph.pub

[root@node1 ~]# cat /etc/ceph/ceph.conf 
# minimal ceph.conf for e5e3d7ca-45ec-11ec-80d4-000c29f57e17
[global]
        fsid = e5e3d7ca-45ec-11ec-80d4-000c29f57e17
        mon_host = [v2:10.4.7.21:3300/0,v1:10.4.7.21:6789/0]
```

```shell
[root@node1 ~]# docker images
REPOSITORY           TAG       IMAGE ID       CREATED         SIZE
ceph/ceph-grafana    6.7.4     557c83e11646   3 months ago    486MB
ceph/ceph            v15       2cf504fded39   5 months ago    1.03GB
prom/prometheus      v2.18.1   de242295e225   18 months ago   140MB
prom/alertmanager    v0.20.0   0881eb8f169f   23 months ago   52.1MB
prom/node-exporter   v0.18.1   e5a616e4b9cf   2 years ago     22.9MB

[root@node1 ~]# docker ps -a
```

```shell
[root@node1 ~]# ceph orch ps 
[root@node1 ~]# ceph orch ps --daemon-type mds
[root@node1 ~]# ceph orch ps --daemon-type mgr
[root@node1 ceph]# cephadm shell
[ceph: root@node1 /]# ceph status
```

```shell
# 安装ceph-common包
cephadm add-repo --release octopus
cephadm install ceph-common
```

```shell
cp /etc/yum.repos.d/ceph.repo{,.bak}

cat > ceph.repo << 'EOF'
[Ceph]
name=Ceph $basearch
baseurl=http://mirrors.aliyun.com/ceph/rpm-octopus/el8/$basearch
enabled=1
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/ceph/keys/release.asc

[Ceph-noarch]
name=Ceph noarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-octopus/el8/noarch
enabled=1
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/ceph/keys/release.asc

[Ceph-source]
name=Ceph SRPMS
baseurl=http://mirrors.aliyun.com/ceph/rpm-octopus/el8/SRPMS
enabled=1
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/ceph/keys/release.asc
EOF
```

```shell
[root@node1 ~]# ceph -v
ceph version 15.2.15 (2dfb18841cfecc2f7eb7eb2afd65986ca4d95985) octopus (stable)
```

### 将主机添加到集群中

```shell
# 在新主机的根用户authorized_keys文件中安装集群的公共SSH密钥
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node3

[root@node1 ~]# ceph orch host add node2
Added host 'node2'
[root@node1 ~]# ceph orch host add node3
Added host 'node3'

[root@node1 ~]# ceph orch host ls
HOST   ADDR   LABELS  STATUS
node1  node1
node2  node2
node3  node3

[root@node1 ~]# ceph -s
  cluster:
    id:     55e5485a-b292-11ea-8087-000c2993d00b
    health: HEALTH_WARN
            Reduced data availability: 1 pg inactive
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 3 daemons, quorum node1,node2,node3 (age 2m)
    mgr: node1.dzrkgj(active, since 16m), standbys: node2.dygsih
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     100.000% pgs unknown
             1 unknown
```

### 部署其他Monitors（可选）

```shell
# ceph config set mon public_network *<mon-cidr-network>*
ceph config set mon public_network 10.1.2.0/24
# ceph orch apply mon *<number-of-monitors>*
# ceph orch apply mon *<host1,host2,host3,...>*

# ceph orch host label add *<hostname>* mon
# ceph orch host ls
# ceph orch host label add host1 mon
# ceph orch host label add host2 mon
# ceph orch host label add host3 mon

# ceph orch host ls
HOST   ADDR   LABELS  STATUS
host1         mon
host2         mon
host3         mon
host4
host5

# ceph orch apply mon label:mon
# ceph orch apply mon --unmanaged
# ceph orch daemon add mon *<host1:ip-or-network1> [<host1:ip-or-network-2>...]*
# ceph orch apply mon --unmanaged
# ceph orch daemon add mon newhost1:10.1.2.123
# ceph orch daemon add mon newhost2:10.1.2.0/24
```

### 部署OSD

```shell
ceph orch device ls

# ceph orch apply osd --all-available-devices

[root@node1 ~]# ceph orch daemon add osd node1:/dev/sdb
Created osd(s) 0 on host 'node1'
[root@node1 ~]# ceph orch daemon add osd node2:/dev/sdb
Created osd(s) 1 on host 'node2'
[root@node1 ~]# ceph orch daemon add osd node3:/dev/sdb
Created osd(s) 2 on host 'node3'

[root@node1 ~]# ceph -s
  cluster:
    id:     55e5485a-b292-11ea-8087-000c2993d00b
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum node1,node2,node3 (age 4m)
    mgr: node1.dzrkgj(active, since 4m), standbys: node2.dygsih
    osd: 3 osds: 3 up (since 85s), 3 in (since 85s)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 57 GiB / 60 GiB avail
    pgs:     1 active+clean
```

### 部署的MDS

```shell
[root@node1 ~]# ceph osd pool create cephfs_data 64 64
pool 'cephfs_data' created

[root@node1 ~]# ceph osd pool create cephfs_metadata 64 64
pool 'cephfs_metadata' created

[root@node1 ~]# ceph fs new cephfs cephfs_metadata cephfs_data
new fs with metadata pool 3 and data pool 2

[root@node1 ~]# ceph fs ls
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]

[root@node1 ~]# ceph orch apply mds cephfs --placement="3 node1 node2 node3"

[root@node1 ~]# docker ps | grep mds
[root@node1 ~]# ceph -s
```

### 部署RGWS

```shell
#如果尚未创建领域，请首先创建一个领域：
radosgw-admin realm create --rgw-realm=myorg --default

#接下来创建一个新的区域组：
radosgw-admin zonegroup create --rgw-zonegroup=default --master --default

#接下来创建一个区域：
radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=cn-east-1 --master --default

#为特定领域和区域部署一组radosgw守护程序：
ceph orch apply rgw myorg cn-east-1 --placement="3 node1 node2 node3"

[root@node1 ~]# docker ps | grep rgw
[root@node1 ~]# ceph orch ls | grep rgw
```

### 部署NFS Ganesha

```shell
ceph osd pool create nfs-ganesha 64 64

ceph orch apply nfs foo nfs-ganesha nfs-ns --placement="3 node1 node2 node3"

ceph osd pool application enable nfs-ganesha cephfs

[root@node1 ~]# docker ps | grep nfs
[root@node1 ~]# ceph orch ls  | grep nfs

dnf -y install nfs-utils
mount -t nfs4 node1:/vfs_ceph /mnt
```



















































