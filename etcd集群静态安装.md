# 通过静态发现方式部署ETCD集群 ##

etcd构建高可用有三种方式：

* 静态发现
* 动态发现
* DNS动态发现

更多详细信息请参考：<https://github.com/coreos/etcd/blob/master/Documentation/op-guide/clustering.md#static>

环境准备：
    
    [root@node1 ~]# cat /etc/redhat-release 
     CentOS Linux release 7.3.1611 (Core) 

节点IP：

    etcd-1: 192.168.6.13
    etcd-2：192.168.6.14 
    etcd-3: 192.168.0.77

ETCD下载：<https://github.com/coreos/etcd/releases/download/v3.2.18/etcd-v3.2.18-linux-amd64.tar.gz>

安装步骤：

    cd /usr/local/src/
    tar xf etcd-v3.2.18-linux-amd64.tar.gz
    mv etcd-v3.2.18-linux-amd64 /usr/local/etcd-v3.2.18
    cd /usr/local/etcd-v3.2.18/
    cp /usr/local/etcd-v3.2.18/etcd* /usr/bin/
    mkdir /etc/etcd
节点一配置：

    cat /etc/etcd/conf.yml
    name: etcd-1  
    data-dir: /opt/etcd/data  
    listen-client-urls: http://192.168.6.13:2379 
    advertise-client-urls: http://192.168.6.13:2379 
    listen-peer-urls: http://192.168.6.13:2380 
    initial-advertise-peer-urls: http://192.168.6.13:2380  
    initial-cluster: etcd-1=http://192.168.6.13:2380,etcd-2=http://192.168.6.14:2380,etcd-3=http://192.168.0.77:2380  
    initial-cluster-token: etcd-cluster-token  
    initial-cluster-state: new 
节点二配置：

    cat /etc/etcd/conf.yml
    name: etcd-2  
    data-dir: /opt/etcd/data  
    listen-client-urls: http://192.168.6.14:2379 
    advertise-client-urls: http://192.168.6.14:2379 
    listen-peer-urls: http://192.168.6.14:2380 
    initial-advertise-peer-urls: http://192.168.6.14:2380  
    initial-cluster: etcd-1=http://192.168.6.13:2380,etcd-2=http://192.168.6.14:2380,etcd-3=http://192.168.0.77:2380  
    initial-cluster-token: etcd-cluster-token  
    initial-cluster-state: new 
节点三配置：

    cat /etc/etcd/conf.yml
    name: etcd-3 
    data-dir: /opt/etcd/data  
    listen-client-urls: http://192.168.0.77:2379 
    advertise-client-urls: http://192.168.0.77:2379 
    listen-peer-urls: http://192.168.0.77:2380 
    initial-advertise-peer-urls: http://192.168.0.77:2380  
    initial-cluster: etcd-1=http://192.168.6.13:2380,etcd-2=http://192.168.6.14:2380,etcd-3=http://192.168.0.77:2380  
    initial-cluster-token: etcd-cluster-token  
    initial-cluster-state: new 

启动命令：

    /usr/bin/etcd --config-file=/etc/etcd/conf.yml
查看集群列表：
 
    [root@kub-master ~]# /usr/bin/etcdctl --endpoints "http://192.168.6.13:2379 " member list
    4861972b3d540ea5: name=etcd-1 peerURLs=http://192.168.6.13:2380 clientURLs=http://192.168.6.13:2379  isLeader=false
    54fcb2edbfd0604e: name=etcd-2 peerURLs=http://192.168.6.14:2380 clientURLs=http://192.168.6.14:2379 isLeader=false
    fe81c1aadc444d79: name=etcd-3 peerURLs=http://192.168.0.77:2380 clientURLs=http://192.168.0.77:2379 isLeader=true
查看集群健康状态：

    [root@bogon ~]# /usr/bin/etcdctl --endpoints "http://192.168.0.77:2379 " cluster-health
    member 4861972b3d540ea5 is healthy: got healthy result from http://192.168.6.13:2379 
    member 54fcb2edbfd0604e is healthy: got healthy result from http://192.168.6.14:2379 
    member fe81c1aadc444d79 is healthy: got healthy result from http://192.168.0.77:2379 
    cluster is healthy

配置ETCD为启动服务：

    [root@kub-master ~]# cat /usr/lib/systemd/system/etcd.service
    [Unit]  
    Description=Etcd Server  
    After=network.target  
    After=network-online.target  
    Wants=network-online.target  
  
    [Service]  
    Type=notify  
    WorkingDirectory=/usr/local/etcd-v3.2.18  
    # User=etcd  
    ExecStart=/usr/bin/etcd --config-file=/etc/etcd/conf.yml  
    Restart=on-failure  
    LimitNOFILE=65536  
  
    [Install]  
    WantedBy=multi-user.target

加载配置并启动：

    systemctl daemon-reload
    systemctl enable etcd
    systemctl start etcd
    
