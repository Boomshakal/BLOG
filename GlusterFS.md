# GlusterFS

## Install

```shell
root@node01:~# apt -y install glusterfs-server
root@node01:~# systemctl enable --now glusterd
root@node01:~# gluster --version
```

```shell
root@node01:~# mkdir -p /glusterfs/distributed
root@node01:~# gluster peer probe node02
root@node01:~# gluster peer status

root@node01:~# gluster volume create vol_distributed transport tcp \
node01:/glusterfs/distributed \
node02:/glusterfs/distributed

root@node01:~# gluster volume start vol_distributed
root@node01:~# gluster volume info
```

## ClusterFS Client

```shell
root@client:~# apt -y install glusterfs-client
root@client:~# mount -t glusterfs node01:/vol_distributed /mnt
root@client:~# df -hT
```

## Gluster + NFS-Ganesha

```shell
# OK if [nfs.disable: on] (default setting)
root@node01:~# gluster volume get vol_distributed nfs.disable
Option                                  Value
------                                  -----
nfs.disable                             on

# if [nfs.disable: off], turn to disable
root@node01:~# gluster volume set vol_distributed nfs.disable on
volume set: success
# if NFS server is running, disable it
root@node01:~# systemctl disable --now nfs-server
```

```shell
root@node01:~# apt -y install nfs-ganesha-gluster
root@node01:~# mv /etc/ganesha/ganesha.conf /etc/ganesha/ganesha.conf.org
root@node01:~# vi /etc/ganesha/ganesha.conf

# create new
NFS_CORE_PARAM {
    # possible to mount with NFSv3 to NFSv4 Pseudo path
    mount_path_pseudo = true;
    # NFS protocol
    Protocols = 3,4;
}
EXPORT_DEFAULTS {
    # default access mode
    Access_Type = RW;
}
EXPORT {
    # uniq ID
    Export_Id = 101;
    # mount path of Gluster Volume
    Path = "/vol_distributed";
    FSAL {
    	# any name
        name = GLUSTER;
        # hostname or IP address of this Node
        hostname="10.0.0.51";
        # Gluster volume name
        volume="vol_distributed";
    }
    # config for root Squash
    Squash="No_root_squash";
    # NFSv4 Pseudo path
    Pseudo="/vfs_distributed";
    # allowed security options
    SecType = "sys";
}
LOG {
    # default log level
    Default_Log_Level = WARN;
}

root@node01:~# systemctl restart nfs-ganesha
root@node01:~# systemctl enable nfs-ganesha
# show exports list
root@node01:~# showmount -e localhost
Export list for localhost:
/vfs_distributed (everyone)
```

```shell
root@client:~# apt -y install nfs-common
# specify Pseudo path set on [Pseudo=***] in ganesha.conf
root@client:~# mount -t nfs4 node01:/vfs_distributed /mnt
root@client:~# df -hT
```

## Add Nodes

```shell
root@node01:~# gluster peer probe node03
root@node01:~# gluster peer status
root@node01:~# gluster volume info
root@node01:~# gluster volume add-brick vol_distributed node03:/glusterfs/distributed
# after adding new node, run rebalance volume
root@node01:~# gluster volume rebalance vol_distributed fix-layout start

# OK if [Status] turns to [completed]
root@node01:~# gluster volume status
```

## Remove Nodes

```shell
root@node01:~# gluster volume info
root@node01:~# gluster volume remove-brick vol_distributed node03:/glusterfs/distributed start
# confirm status
root@node01:~# gluster volume remove-brick vol_distributed node03:/glusterfs/distributed status
# after [status] turning to [completed], commit removing
root@node01:~# gluster volume remove-brick vol_distributed node03:/glusterfs/distributed commit
# confirm volume info
root@node01:~# gluster volume info
```

## Replication Configuration

```shell
# node01 node02 node03
root@node01:~# apt -y install glusterfs-server
root@node01:~# systemctl enable --now glusterd
root@node01:~# gluster --version
root@node01:~# mkdir -p /glusterfs/replica
```

```shell
# probe nodes
root@node01:~# gluster peer probe node02
peer probe: success.
root@node01:~# gluster peer probe node03
peer probe: success.
# confirm status
root@node01:~# gluster peer status
Number of Peers: 2

Hostname: node02
Uuid: f2fce535-c10e-41cb-8c9f-c6636ae38eff
State: Peer in Cluster (Connected)

Hostname: node03
Uuid: 014a1e8f-967d-4709-bac4-ea1de8ef96cb
State: Peer in Cluster (Connected)

# create volume
root@node01:~# gluster volume create vol_replica replica 3 transport tcp \
node01:/glusterfs/replica \
node02:/glusterfs/replica \
node03:/glusterfs/replica
volume create: vol_replica: success: please start the volume to access data
# start volume
root@node01:~# gluster volume start vol_replica
volume start: vol_replica: success
# confirm volume info
root@node01:~# gluster volume info
```

## Distributed GlusterFS

```shell
# create Distributed volume(分布式) 
gluster volume create vol_dist-replica replica 3 arbiter 1 transport tcp \
node01:/glusterfs/dist-replica \
node02:/glusterfs/dist-replica \
node03:/glusterfs/dist-replica \
node04:/glusterfs/dist-replica \
node05:/glusterfs/dist-replica \
node06:/glusterfs/dist-replica
```

## Dispersed GlusterFS

```shell
# create volume
gluster volume create vol_dispersed disperse-data 4 redundancy 2 transport tcp \
node01:/glusterfs/dispersed \
node02:/glusterfs/dispersed \
node03:/glusterfs/dispersed \
node04:/glusterfs/dispersed \
node05:/glusterfs/dispersed \
node06:/glusterfs/dispersed
```

## Stripe volume

```shell
gluster volume create stripe-volume stripe 2 transport tcp server1:/dir1 server2:/dir2
```

https://www.cnblogs.com/zhijiyiyu/p/15339674.html#21-%E5%88%86%E5%B8%83%E5%BC%8F%E5%8D%B7distribute-volume





























