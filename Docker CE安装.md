# Centos 安装docker CE #

### 系统要求 ###
Docker CE支持64位版本CentOS 7,并且要求内核版本不低于3.10，CentOS 7满足最低内核的要求，但由于内核版本比较低，部分功能（如overlay2 存储驱动）无法使用，并且部分功能可能不太稳定。内核版本从3.18+开始，Docker支持overlay驱动，4.0内核，官方推荐的是overlay2，先扔一个坑在这，默认xfs文件系统与overlay/overlay2不兼容，本次安装采用ext4文件系统。

### 系统信息 ###
    [root@localhost ~]# cat /etc/redhat-release 
    CentOS Linux release 7.4.1708 (Core) 
    [root@localhost ~]# uname -r
     4.16.2-1.el7.elrepo.x86_64
### 文件系统 ##
    [root@localhost ~]# blkid 
    /dev/sda1: UUID="2719dfbd-8458-4470-89b7-f5b1e5301451" TYPE="ext4" 
    /dev/sda2: UUID="5966fdbb-e50f-402b-b756-04065ed96248" TYPE="ext4"
## 安装开始： ##

##### 卸载旧版本 #####

    $ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
##### 安装依赖包 #####

    $ sudo yum install -y yum-utils \
      device-mapper-persistent-data \
      lvm2
##### 安装docker源 #####
鉴于国内网络问题，强烈建议使用国内源

国内源：

    $ sudo yum-config-manager \
    --add-repo \
    https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo

官方源：

    #  $ sudo yum-config-manager \
    #  --add-repo \
    #  https://download.docker.com/linux/centos/docker-ce.repo
##### 安装最新版本Docker CE #####
    $ sudo yum-config-manager --enable docker-ce-edge

##### 安装docker #####
    $ sudo yum install docker-ce
##### 启动Docker CE #####
    $ sudo systemctl enable docker
    $ sudo systemctl start docker
##### 查看docker版本信息 #####
    [root@localhost ~]# docker info
    Containers: 0
     Running: 0
     Paused: 0
     Stopped: 0
    Images: 0
    Server Version: 18.03.0-ce
    Storage Driver: overlay2
     Backing Filesystem: extfs
     Supports d_type: true
     Native Overlay Diff: true
    Logging Driver: json-file
    Cgroup Driver: cgroupfs
    Plugins:
     Volume: local
     Network: bridge host macvlan null overlay
     Log: awslogs fluentd gcplogs gelf journald json-file logentries splunk syslog
    Swarm: inactive
    Runtimes: runc
    Default Runtime: runc
    Init Binary: docker-init
    containerd version: cfd04396dc68220d1cecbe686a6cc3aa5ce3667c
    runc version: 4fc53a81fb7c994640722ac585fa9ca548971871
    init version: 949e6fa
    Security Options:
     seccomp
      Profile: default
    Kernel Version: 4.16.2-1.el7.elrepo.x86_64
    Operating System: CentOS Linux 7 (Core)
    OSType: linux
    Architecture: x86_64
    CPUs: 1
    Total Memory: 3.831GiB
    Name: localhost
    ID: XKDP:UCVC:ZKML:JGR7:NBC6:KWIS:F5T3:AC6O:D7LG:UXR3:6O6M:3XFW
    Docker Root Dir: /var/lib/docker
    Debug Mode (client): false
    Debug Mode (server): false
    Registry: https://index.docker.io/v1/
    Labels:
    Experimental: false
    Insecure Registries:
     127.0.0.0/8
    Live Restore Enabled: false

    WARNING: bridge-nf-call-iptables is disabled
    WARNING: bridge-nf-call-ip6tables is disabled
##### 修改内核参数 #####
    $ sudo tee -a /etc/sysctl.conf <<-EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
重新加载内核参数

    $ sudo sysctl -p
至于为什么报此错误，为什么这样解决，don't no !
##### 配置国内镜像加速器 #####

    $ vim /etc/docker/daemon.json
 
    {
     "registry-mirrors": ["https://registry.docker-cn.com"]
    }
##### 重启docker #####

    [root@localhost ~]# systemctl restart docker

---------------------------------------------------------------------------

# 上面提到xfs的一个坑，下面我们来聊聊这个问题 #

Overlay和Overlay2是docker支持的两种存储驱动程序，它们都依赖于overlayfs文件系统。而在overlayfs代码中（它是Linux内核的一部分），用到了d\_tpye这个东东，d\_tpye信息被用来访问并用来确保某些文件操作被正确处理。overlayfs中有代码专门检查是否存在d\_type特征，并在底层文件系统中不存在时打印警告信息。在Overlay/Overlay2存储驱动程序上运行时，docker需要d_type功能才能正常工作。那这跟xfs有什么关系呢？

对于某些文件系统，d\_type支持是可选的。 这包括Red Hat Enterprise Linux 7中的默认文件系统XFS，它是CentOS 7的上游基础。不幸的是，Red Hat / CentOS安装程序和mkfs.xfs命令都默认创建了XFS文件系统，但没有启用d\_type功能......一团糟！

作为一个快速规则，如果您使用的是RHEL 7或CentOS 7，并且默认情况下创建文件系统而不指定参数，则几乎可以100％确定文件系统上的d_type未打开。 要检查确定，请按照以下步骤操作。

首先你需要找出你当前使用的文件系统。 虽然XFS在安装过程中是默认的，但有些人或托管服务提供商可能会选择使用Ext4。 如果是这样的话，那么放松一下，d_type是支持的。

如果您是XFS文件系统，那么可以用xfs_info命令查看
    
    [root@localhost ~]# xfs_info /
    meta-data=/dev/sda3              isize=256    agcount=4, agsize=524224 blks
             =                       sectsz=512   attr=2, projid32bit=1
             =                       crc=0        finobt=0
    data     =                       bsize=4096   blocks=2096896, imaxpct=25
             =                       sunit=0      swidth=0 blks
    naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
    log      =internal               bsize=4096   blocks=2560, version=2
             =                       sectsz=512   sunit=0 blks, lazy-count=1
    realtime =none                   extsz=4096   blocks=0, rtextents=0

注意到ftype=0没有，很不幸，0代表不支持。

如何避免或者解决这个问题？

1、装系统时采用ext4文件系统

2、如果系统已经装完，那么只能重新创建文件系统，步骤如下：

* 备份数据
* 用XFS的正确参数重新创建文件系统，或者改为创建一个Ext4文件系统。
* 恢复您的数据

如果您选择了Ext4文件系统，那么很简单，只需运行：

    mkfs.ext4 /path/to/your/device

如果您选择XFS文件系统，则正确的命令是：

    mkfs.xfs -n ftype=1 /path/to/your/device

不幸的是重新创建文件系统，要先umount设备，否则无法创建，但是，如果系统和数据在一个盘，umount后，就会发现找不见原先的设备了，真是一个悲剧。


 文章参考： 

<https://docs.docker.com/install/linux/docker-ce/centos/#upgrade-docker-ce>   
<https://yeasy.gitbooks.io/docker_practice/content/install/mirror.html>
<https://linuxer.pro/2017/03/what-is-d_type-and-why-docker-overlayfs-need-it/>