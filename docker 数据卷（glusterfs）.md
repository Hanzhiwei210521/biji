# docker持久化数据卷 #

### glusterfs安装部署 ###

安装gluster源

    yum install centos-release-gluster -y
安装glusterfs组件

    yum install -y glusterfs glusterfs-server glusterfs-fuse glusterfs-rdma glusterfs-geo-replication glusterfs-devel
创建glusterfs目录

    mkdir /opt/glusterd
修改glusterd目录

    sed -i 's/var\/lib/opt/g' /etc/glusterfs/glusterd.vol
启动glusterfs，加入开机自启

    systemctl start glusterd.service
    systemctl enable glusterd.service
配置glusterfs

    [root@k8s-master ~]# vi /etc/hosts
    192.168.6.12 k8s-master
    192.168.6.7 k8s-node1
    192.168.6.10 k8s-node2
存储主机加入存储信任池

    [root@k8s-master ~]# gluster peer probe k8s-node1
    [root@k8s-master ~]# gluster peer probe k8s-node2
gluster节点状态查看

    [root@k8s-master ~]# gluster peer status
    Number of Peers: 2

    Hostname: k8s-node1
    Uuid: f656750d-9270-475e-8ce7-d98054342b22
    State: Peer in Cluster (Connected)

    Hostname: k8s-node2
    Uuid: 3bdc35de-066f-4cd1-bf16-a45f1a1f8e63
    State: Peer in Cluster (Connected)
对磁盘进行分区

    [root@k8s-master ~]# fdisk -l

    Disk /dev/vda: 53.7 GB, 53687091200 bytes, 104857600 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0x000d402f

       Device Boot      Start         End      Blocks   Id  System
    /dev/vda1   *        2048     1026047      512000   83  Linux
    /dev/vda2         1026048    21997567    10485760   83  Linux
    /dev/vda3        21997568   104857599    41430016   83  Linux
创建存储目录
    
    [root@k8s-master ~]# mkdir /storage/brick0 -p
    
格式化磁盘并挂载
    
    [root@k8s-master ~]# mkfs.xfs -f /dev/vda3   
    [root@k8s-master ~]# mount /dev/vda3 /storage/brick0
    [root@k8s-master ~]# echo "/dev/vda3 /storage/brick0  xfs  defaults 0 0" >>/etc/fstab
创建volume

GlusterFS中的volume的模式有很多中，包括以下几种：

* 分布卷（默认模式）：即DHT, 也叫 分布卷: 将文件已hash算法随机分布到 一台服务器节点中存储。
* 复制模式：即AFR, 创建volume 时带 replica x数量: 将文件复制到 replica x 个节点中。
* 条带模式：即Striped, 创建volume 时带 stripe x 数量： 将文件切割成数据块，分别存储到 stripe x 个节点中 ( 类似raid 0 )。
* 分布式条带模式：最少需要4台服务器才能创建。创建volume时 stripe 2 server = 4 个节点： 是DHT 与 Striped 的组合型。
* 分布式复制模式：最少需要4台服务器才能创建。创建volume时 replica 2 server = 4 个节点：是DHT 与 AFR 的组合型。
* 条带复制卷模式：最少需要4台服务器才能创建。创建volume时 stripe 2 replica 2 server = 4 个节点： 是 Striped 与 AFR 的组合型。
* 三种模式混合： 至少需要8台服务器才能创建。 stripe 2 replica 2 , 每4个节点 组成一个组。

在这我们创建复制式卷

    [root@k8s-master ~]# gluster volume create k8s-volume replica 3 k8s-master:/storage/brick0 k8s-node1:/storage/brick0 k8s-node2:/storage/brick0 force
查看volume状态

    [root@k8s-master ~]# gluster volume info
 
    Volume Name: k8s-volume
    Type: Replicate
    Volume ID: 6f9362c9-9352-4537-843a-59cc1d94b7d4
    Status: Created
    Snapshot Count: 0
    Number of Bricks: 1 x 3 = 3
    Transport-type: tcp
    Bricks:
    Brick1: k8s-master:/storage/brick0
    Brick2: k8s-node1:/storage/brick0
    Brick3: k8s-node2:/storage/brick0
    Options Reconfigured:
    transport.address-family: inet
    nfs.disable: on
    performance.client-io-threads: off
启动volume

    [root@k8s-master ~]# gluster volume start k8s-volume
创建/data目录并挂载volume

    [root@k8s-master /]# mkdir /data
    [root@k8s-master /]# mount -t glusterfs 127.0.0.1:/k8s-volume /data
k8s中配置glusterfs
    
    [root@k8s-master ~]# mkdir glusterfs
    [root@k8s-master ~]# cd glusterfs/
    # 上传json文件
    [root@k8s-master glusterfs]# ls
    glusterfs-endpoints.json  glusterfs-service.json
    [root@k8s-master glusterfs]# kubectl apply -f glusterfs-endpoints.json 
    endpoints "glusterfs-cluster" created
    [root@k8s-master glusterfs]# kubectl apply -f glusterfs-service.json 
    service "glusterfs-cluster" created
    [root@k8s-master glusterfs]# kubectl get ep
    NAME                ENDPOINTS                                           AGE
    glusterfs-cluster   192.168.6.10:110,192.168.6.12:110,192.168.6.7:110   13s
    kubernetes          192.168.6.12:6443                                   18h
创建PV，PVC

    [root@k8s-master ~]# cd /data/
    [root@k8s-master data]# mkdir /data/{svn,redis} -p
    # 上传yaml文件
    [root@k8s-master ~]# mkdir {svn,redis}
    [root@k8s-master ~]# ls svn/
    svn-pvc.yaml  svn-pv.yaml
    [root@k8s-master svn]# kubectl create -f svn-pv.yaml 
    [root@k8s-master svn]# kubectl create -f svn-pvc.yaml
    [root@k8s-master ~]# ls redis/
    redis-pvc.yaml  redis-pv.yaml
    [root@k8s-master redis]# kubectl create -f redis-pv.yaml 
    [root@k8s-master redis]# kubectl create -f redis-pvc.yaml 
    [root@k8s-master redis]# kubectl get pv
    NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM               STORAGECLASS   REASON    AGE
    pv-redis   5Gi        RWX            Retain           Bound     default/pvc-redis                            9s
    pv-svn     1Gi        RWX            Retain           Bound     default/pvc-svn                              1m
    [root@k8s-master redis]# kubectl get pvc
    NAME        STATUS    VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    pvc-redis   Bound     pv-redis   5Gi        RWX                           10s
    pvc-svn     Bound     pv-svn     1Gi        RWX                           1m
给node节点打标签

    [root@k8s-master ~]# kubectl label node 192.168.6.7 storagenode=glusterfs
    [root@k8s-master ~]# kubectl label node 192.168.6.10 storagenode=glusterfs
创建secret

    kubectl create secret docker-registry registry-secret --docker-server=harbor.jinxiudadi.com --docker-username=hanzhiwei --docker-password=******* --docker-email=xxx@xxx.com -n default