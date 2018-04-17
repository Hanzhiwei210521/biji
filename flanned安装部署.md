# flannel 安装部署 #

### flannel 安装 ###
    yum install flannel -y
### flannel 配置 ###
    [root@k8s-node2 ~]# cat /etc/sysconfig/flanneld 
    # Flanneld configuration options  

    # etcd url location.  Point this to the server where etcd runs
    FLANNEL_ETCD_ENDPOINTS="http://192.168.6.15:2379,http://192.168.6.2:2379,http://192.168.6.5:2379"

    # etcd config key.  This is the configuration key that flannel queries
    # For address range assignment
    FLANNEL_ETCD_PREFIX="/jxdd/network"

    # Any additional options that you want to pass
    FLANNEL_OPTIONS="--logtostderr=false --log_dir=/var/log/flannel/  --iface=eth0"
在配置中我们看到了三个参数，第一个参数FLANNEL\_ETCD\_ENDPOINTS指定的是etcd集群的地址和端口，第二个参数FLANNEL\_ETCD\_PREFIX是一个key，第三个参数FLANNEL_OPTIONS是其他选项，指定了flannel日志目录和绑定的网卡接口。

创建flannel日志目录
    
    [root@k8s-node2 ~]# mkdir /var/log/flannel
在etc中创建变量

    [root@k8s-node2 ~]# etcdctl --endpoints=http://192.168.6.2:2379 set /jxdd/network/config '{"Network":"172.17.0.0/16"}'
在启动flanneld服务之前，需要在etcd中添加一条网络配置记录，这个配置将用于flanneld分配给每个docker的虚拟IP地址段

查看变量

    [root@k8s-node2 ~]# etcdctl --endpoints=http://192.168.6.2:2379 get /jxdd/network/config
    {"Network":"172.17.0.0/16"}
### 启动flannel并加入自启 ###
   
    systemctl enable flanneld
    systemctl start flanneld
查看flannel启动情况
    
    [root@k8s-node2 ~]# ps -ef|grep flanneld
    root     11671     1  1 14:50 ?        00:00:00 /usr/bin/flanneld -etcd-endpoints=http://192.168.6.15:2379,http://192.168.6.2:2379,http://192.168.6.5:2379 -etcd-prefix=/jxdd/network --logtostderr=false --log_dir=/var/log/flannel/ --iface=eth0
### flannel启动过程解析 ###

我们先看一下flannel的systemd脚本

    [root@k8s-node2 ~]# cat /usr/lib/systemd/system/flanneld.service
    [Unit]
    Description=Flanneld overlay address etcd agent
    After=network.target
    After=network-online.target
    Wants=network-online.target
    After=etcd.service
    Before=docker.service

    [Service]
    Type=notify
    EnvironmentFile=/etc/sysconfig/flanneld
    EnvironmentFile=-/etc/sysconfig/docker-network
    ExecStart=/usr/bin/flanneld-start $FLANNEL_OPTIONS
    ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    RequiredBy=docker.service
flannel服务启动是主要做了一下几部工作：

* 从etcd中获取network的配置信息
* 划分subnet,并在etcd中进行注册
* 将子网信息记录到/run/flannel/subnet.env中

[root@k8s-node2 ~]# cat /run/flannel/subnet.env 

    FLANNEL_NETWORK=172.17.0.0/16
    FLANNEL_SUBNET=172.17.41.1/24
    FLANNEL_MTU=1472
    FLANNEL_IPMASQ=false
* 之后会有一个脚本将subnet.env转写成一个docker的环境变量文件/run/flannel/docker，这个过程在flannel的systemd脚本中有体现mk-docker-opts.sh

[root@k8s-node2 ~]# cat /run/flannel/docker 

    DOCKER_OPT_BIP="--bip=172.17.41.1/24"
    DOCKER_OPT_IPMASQ="--ip-masq=true"
    DOCKER_OPT_MTU="--mtu=1472"
    DOCKER_NETWORK_OPTIONS=" --bip=172.17.41.1/24 --ip-masq=true --mtu=1472"
### 配置docker 使用flannel网络
在systemd中引用/run/flannel/docker文件

    [root@k8s-node2 ~]# cat /usr/lib/systemd/system/docker.service
    [Unit]
    Description=Docker Application Container Engine
    Documentation=https://docs.docker.com
    After=network-online.target firewalld.service
    Wants=network-online.target

    [Service]
    Type=notify
    # the default is not to use systemd for cgroups because the delegate issues still
    # exists and systemd currently does not support the cgroup feature set required
    # for containers run by docker
    EnvironmentFile=-/run/flannel/docker    ###添加此行配置
    ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS   ###添加$DOCKER_NETWORK_OPTIONS选项
    ExecReload=/bin/kill -s HUP $MAINPID
    # Having non-zero Limit*s causes performance problems due to accounting overhead
    # in the kernel. We recommend using cgroups to do container-local accounting.
    LimitNOFILE=infinity
    LimitNPROC=infinity
    LimitCORE=infinity
    # Uncomment TasksMax if your systemd version supports it.
    # Only systemd 226 and above support this version.
    #TasksMax=infinity
    TimeoutStartSec=0
    # set delegate yes so that systemd does not reset the cgroups of docker containers
    Delegate=yes
    # kill only the docker process, not all processes in the cgroup
    KillMode=process
    # restart the docker process if it exits prematurely
    Restart=on-failure
    StartLimitBurst=3
    StartLimitInterval=60s

    [Install]
    WantedBy=multi-user.target

### 重新启动docker ###

    systemctl daemon-reload
    systemctl restart docker 
查看docker进程

    [root@k8s-node2 ~]# ps -ef|grep docker
    root     11861     1  0 14:57 ?        00:00:04 /usr/bin/dockerd --bip=172.17.41.1/24 --ip-masq=true --mtu=1472
    root     11868 11861  0 14:57 ?        00:00:13 docker-containerd --config /var/run/docker/containerd/containerd.toml
需要注意的是docker启动参数中--bip=172.17.41/24这个参数，flannel的作用就是修改了这个参数相当于为每台node主机的docker划分了子网。

我们可以在etcd中查看flannel设置的flannel0地址与物理机IP的对应规则

    [root@k8s-node2 ~]# etcdctl --endpoints=http://192.168.6.2:2379 ls /jxdd/network/subnets
    /jxdd/network/subnets/172.17.41.0-24
    /jxdd/network/subnets/172.17.7.0-24
    [root@k8s-node2 ~]# etcdctl --endpoints=http://192.168.6.2:2379 get /jxdd/network/subnets/172.17.41.0-24
    {"PublicIP":"192.168.6.2"}
# Flannel 工作原理 #

Flannel之所以可以搭建Kubernetes依赖的底层网络，是因为它能实现以下两点。

1、它能协助kubernetes，给每一个node上的docker容器分配互相不冲突的IP地址。

2、它能在这些IP地址之间建立一个覆盖网络（Overlay Network），通过这个覆盖网络，将数据包原封不动地传递到目标容器内。

现在我们通过如下图来看看Flannel是如何实现这两点的。

![](https://github.com/Hanzhiwei210521/loading/blob/master/image/flannel-01.png)

可以看到，Flannel首先创建了一个名为flannel0的网桥，而且这个网桥的一端连接docker0网桥，另一端连接一个叫做flanneld的服务进程。flanneld进程并不简单，它首先上联etcd，利用etcd来管理可分配的IP地址段资源，同时监控etcd中每个pod的实际地址，并在内存中建立了一个pod节点路由表；然后下连docker0和物理网络，使用内存中的pod节点路由表，将docker0发给它的数据包包装起来，利用物理网络的连接将数据包投递到目标flanneld上，从而完成pod到pod之间的地址通信。

Flannel之间的底层通信协议的可选余地很多，有UDP、VxLan、AWS VPC等多种方式，只要能通到对端的Flannel就可以了。源flanneld加包，目标flanneld解包，最终docker0看到的就是原始的数据，非常透明，根本感觉不到中间Flannel的存在。

Flannel完美的实现了对kubernetes网络的支持，但是它引入了多个网络组建，在网络通信时需要转到flannel0网络接口，再转到用户态的flanneld程序，到对端后还需要走这个过程的反过程，所以也会引入一些网络的时延损耗。

另外，Flannel模型默认使用了UDP作为底层传输协议，UDP本身是非可靠协议，虽然两端的TCP实现了可靠传输，但在大流量、高并发应用场景下还需要反复测试，确保没有问题。
