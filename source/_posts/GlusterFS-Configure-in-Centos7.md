---
title: 在 CentOS 7 下安装配置 GlusterFS 
tags: ["Linux","Centos","GlusterFS"]
date: 2018-04-26 16:44:24
categories: ["GlusterFS"]
top_img: 
description: 在 CentOS 7 下 GlusterFS 搭建配置及使用。
---

# GLusterFS

GlusterFS 是一个分布式文件系统，具有强大的横向扩展能力。主要提供的的存储功能为文件存储，当然也可通过其他方式支持快存储和对象存储。与存储相关的分布式文件系统还有 Ceph。



## Enviroment

OS: CentOS 7



## Install GlusterFS

```shell
# yum -y install centos-release-gluster
# yum install glusterfs-server
```



### Configure GlusterFS

启动服务进程：

```shell
# systemctl enable glusterd.service
# systemctl start glusterd.service
```



GLusterFS 节点之间通信用到了 TCP 端口 24007-24008，所以启用所需端口：

```shell
# firewall-cmd --zone=public --add-port=24007-24008/tcp --permanent
# firewall-cmd --reload
```

或者选择关闭防火墙。



## GlusterFS 管理

这里简单介绍一下 GlusterFS 服务器端的管理程序 gluster 的使用。

这里假设拥有 4 台服务器：

| HOSTNAME | IP-address      |
| -------- | --------------- |
| node1    | xxx.xxx.xxx.xx1 |
| node2    | xxx.xxx.xxx.xx2 |
| node3    | xxx.xxx.xxx.xx3 |
| node4    | xxx.xxx.xxx.xx4 |



### Peer 操作

GlusterFS 通过添加 Peer 来创建集群。



#### Peer probe

添加 Peer 需要指定该节点的 IP 或者 HOSTNAME。

```shell
$ gluster peer probe { <HOSTNAME> | <IP-address> }
```

例如在 node1 上：

```shell
$ gluster peer probe node2
$ gluster peer probe node3
$ gluster peer probe node4
```

这样就能将 node1、node2、node3 和 node4 四台服务器连接，组成一个服务器集群。



#### Peer status

查询集群中的节点信息：

```shell
$ gluster peer status
```



#### Peer detach

移除 Peer 需要指定该节点的 IP 或者 HOSTNAME。

```shell
$ gluster peer detach { <HOSTNAME> | <IP-address> } [force] 
```



例如在 node1 移除 node2：

```shell
$ gluster peer probe node2
```

不能在当前服务器从集群中移除当前服务器。



### Volume 操作

GLusterFS 通过创建 Volume 来创建一个虚拟存储池，通过 Volume 的不同类型来提供不同的存储功能。



#### Volume create

创建卷需要指定卷的名字，类型和 Brick（Brcik 是指服务器上的一个目录，后端实际存储的目录）。

```shell
$ gluster volume create <NEW-VOLNAME> [stripe <COUNT>] [replica <COUNT> [arbiter <COUNT>]] [disperse [<COUNT>]] [disperse-data <COUNT>] [redundancy <COUNT>] [transport <tcp|rdma|tcp,rdma>] <NEW-BRICK>?<vg_name>... [force]
```

不同类型的 Volume 对 Brick 的数量也有不同的要求。

假设有如下 Brick：

| HOSTNAME | Brick                 |
| -------- | --------------------- |
| node1    | node1:/bricks/brick01 |
| node2    | node2:/bricks/brick01 |
| node3    | node3:/bricks/brick01 |
| node4    | node4:/bricks/brick01 |

创建一个 replica 类型的 Volume，其 COUNT 至少为2 ,至少需要指定 2 个 Brick

```shell
$ gluster volume create voltest1 replica 2 node1:/bricks/brick01 node2:/bricks/brick01 node3::/bricks/brick01 node4:/bricks/brick01
```

该类型 volume 可以在存储文件的时候，在服务器保存一份该文件的副本。当然 GlusterFS 还有其他很多类型的 volume 这里就不一一介绍了。



#### Volume info

查看 Volume 的信息：

```shell
$ gluster volume info
```



#### Volume status

查看 Volume 状态：

```shell
$ gluster volume status
```



#### Volume delete

删除 Volume：

```shell
$ gluster volume delete <VOLNAME>
```



### Brick 操作

在创建完 Volume 后，如果需要更大的空间，可以通过添加新的 Brick 来提高整体容量，同时添加 Brick 时不会停止服务。



#### Brick add-bricks

命令：

```shell
$ gluster volume add-brick <VOLNAME> [<stripe|replica> <COUNT> [arbiter <COUNT>]] <NEW-BRICK> ... [force]
```



一次添加 Brick 的数量要根据 Volume 的类型来决定。

例如 `voltest1` 是 `replica 2` 则添加时需要添加 2 的整数倍的 Brick。

要将 `node5/bricks/brick01`  `node6:/bricks/brick01`加入到 `voltest1`  Volume 中 

```shell
$ gluster volume add-brick voltest1 node5/bricks/brick01 node6:/bricks/brick01
```



#### Rebalance

这时需要重新分配所保存文件的位置，这样可以确保每一个 Brick 都会被分配到文件，而不会过于集中在特定某一个 Brick。

```shell
$ gluster volume rebalance <VOLNAME> {{fix-layout start} | {start [force]|stop|status}}
```



查看 rebalance 状态

```shell
$ gluster volume rebalance voltest1 status
```



#### Brick 替换

命令：

```shell
$ gluster volume replace-brick <VOLNAME> <SOURCE-BRICK> <NEW-BRICK> {commit force}
```



假设名称为 `voltest1` 的 Volume，使用 `node7:/bricks/brick01`替换 `node1:/bricks/birck01`

```shell
$ gluster volume replace-brick voltest1 node1:/bricks/brick01 node7:/bricks/brick01 start
```



这时可以查看替换的状态

```shell
$ gluster volume replace-brick voltest1 node1:/bricks/brick01 node5:/bricks/brick01 status
```



最后 commit 确认移除旧的 Brick

```shell
$ gluster volume replace-brick voltest1 node1:/bricks/brick01 node5:/bricks/brick01 commit
```



#### Brick 移除

命令：

```shell
$ gluster volume remove-brick <VOLNAME> [replica <COUNT>] <BRICK> ... <start|stop|status|commit|force>
```



移除 Brick 的数量也是由 Volume 的类型决定，

假设移除 `voltest1` 类型为 replica 2 的 Brick 那么移除的 Brick 数为 replica 的整数倍。

```shell
$ gluster volume remove-brick voltest1 node1:/bricks/brick01 node2:/bricks/brick01
```



### Snapshot 操作

Gluster 使用 LVM 的Snapshot 完成快照功能，也就是说您的 Brick 必须在 LVM 的环境中快照功能才能被使用。

使用 LVM 上的 Brick 创建的 Volume 可以通过快照来实现数据的快速恢复。



#### Snapshot create  

命令：

```shell
$ gluster snapshot create <snapname> <volname> [no-timestamp] [description <description>] [force]
```



#### Snapshot activate

在创建后需要激活才能使用：

```shell
$ gluster snapshot activate <snapname> [force]
```



#### Snapshot info

查看快照信息：

```shell
$ gluster snapshot info
```



#### Snapshot restore

GlusterFS 可以将 Volume 数据恢复到快照时的状态，但进行操作时必须停用 Volume，完成后重启。

```shell
$ gluster snapshot restore <snapname>
```

虽然这样不影响工作，但会照成 Brick 位置的变更，若非必要，不要直接使用Snapshot 还原整个Volume。



#### Snapshot clone

GlusterFS 提供从一个现有的快照，创建一个新的 Volume。

```shell
$ gluster snapshot clone <cloname> <snapname>
```



## GlusterFS Client

GlusterFS 可以通过自己的客户端 Glusterfs 或者 Samba、NFS 来访问。



### 开启防火墙

为 Glusterfs、Samba 和 NFS 开启防火墙。

```shell
# firewall-cmd --zone=public --add-service=nfs --add-service=samba --add-service=samba-client --permanent

# firewall-cmd --zone=public --add-port=111/tcp --add-port=139/tcp --add-port=445/tcp --add-port=965/tcp --add-port=2049/tcp \
--add-port=38465-38469/tcp --add-port=631/tcp --add-port=111/udp --add-port=963/udp --add-port=49152-49251/tcp  --permanent

# firewall-cmd --reload
```

当然也可选择关闭防火墙。



### Glusterfs client

安装 GLusterfs client：

```shell
# yum install glusterfs glusterfs-fuse attr -y
```



挂载:

```shell
# mount -t glusterfs node1:/glustervol1 /mnt/
```

node1 为一个服务器节点。



### Samba 

安装：

```shell
# yum install samba samba-client samba-common samba-vfs-glusterfs selinux-policy-targeted -y
```



启动服务：

```shell
# systemctl start smb.service
# systemctl enable smb.service
# systemctl start nmb.service
# systemctl enable nmb.service
```



在每个服务器端，修改 */etc/samba/smb.conf* 文件内的 GlusterFS 已创建 Volume 的配置：

```ini
[gluster-glustervol1]
comment = For samba share of volume glustervol1
vfs objects = glusterfs
glusterfs:volume = glustervol1
glusterfs:logfile = /var/log/samba/glusterfs-glustervol1.%M.log
glusterfs:loglevel = 7
path = /
read only = no
guest ok = yes
```

在后面添加一行 **kernel share modes = No**。这个是为了解决挂载 samba 后创建文件后出现同时创建了多个文件的问题。具体原因不知道，如果有知道的可以交流一下。

修改后为：

```ini
[gluster-glustervol1]
comment = For samba share of volume glustervol1
vfs objects = glusterfs
glusterfs:volume = glustervol1
glusterfs:logfile = /var/log/samba/glusterfs-glustervol1.%M.log
glusterfs:loglevel = 7
path = /
read only = no
guest ok = yes
kernel share modes = No
```



为了能够访问 Samba 需要创建用户，并设置密码。

```shell
# smbpasswd -a sambauser
```

sambauser 是一个用户名，按需要进行创建。



设置 SELinux 运行 Samba 共享：

```shell
# setsebool -P samba_share_fusefs on
# setsebool -P samba_load_libgfapi on
```



当然也可以选择关闭 SELinux

编辑` /etc/sysconfig/selinux`

将

```ini
SELINUX=enforcing
```

改为

```ini
SELINUX=disabled
```



挂载：

```shell
# mount -t cifs \\\\gluster1.example.com\\gluster-glustervol1 /mnt/ -o user=sambauser,pass=mypassword
```

在 Windows 下 可以通过在资源管理器里添加一个新的位置来链接 Samba 共享。



### NFS

需要开启 RPC 服务：

```shell
# systemctl enable rpcbind
# systemctl start rpcbind
```



挂载：

```shell
# mount -t nfs gluster1.example.com:/glustervol1 /mnt/
```





## Block Storage

块存储是非常常用的存储方式，GlusterFS 主要是文件存储，对于块存储要通过镜像文件的方式创建 iscsi target。

这里推荐一个工具来创建帮助快速创建 [**gluster-block**](https://github.com/gluster/gluster-block.git) 。





## 引用

> CentOS Wiki: [gluster-Quickstart](https://wiki.centos.org/zh/SpecialInterestGroup/Storage/gluster-Quickstart?highlight=%28GlusterFS%29)  [GlusterFSonCentOS](https://wiki.centos.org/zh/HowTos/GlusterFSonCentOS?highlight=%28GlusterFS%29)
>
> Gluster-Storage_GitBook: [http://www.l-penguin.idv.tw/book/Gluster-Storage_GitBook/](http://www.l-penguin.idv.tw/book/Gluster-Storage_GitBook/)
>
> 

