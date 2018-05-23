# k8s安装部署 #

### 环境准备 ###

* k8s-master: 192.168.6.12
* k8s-node1: 192.168.6.7
* k8s-node2: 192.168.6.10

<table>
    <tr>
        <td>软件</td>
        <td>版本</td>
    </tr>
    <tr>
        <td>Linux操作系统</td>
        <td>CentOS Linux release 7.2.1511 (Core)</td>
    </tr>
    <tr>
        <td>内核版本</td>
        <td>4.16.2-1.el7.elrepo.x86_64</td>
    </tr>
    <tr>
        <td>k8s</td>
        <td>1.9</td>
    </tr> 
     <tr>
        <td>Docker</td>
        <td>18.05.0-ce</td>
    </tr>    
    <tr>
        <td>etcd</td>
        <td>3.2</td>
    </tr>    
    <tr>
        <td>flannel</td>
        <td>v0.9.1</td>
    </tr> 
</table>



<table>
    <tr>
        <td>角色</td>
        <td>IP</td>
        <td>组件</td>
    </tr>
    <tr>
        <td>k8s-master</td>
        <td>192.168.6.12</td>
        <td>kube-apiserver  kube-controller-manager  kube-scheduler  etcd</td>
    </tr>
    <tr>
        <td>k8s-node1</td>
        <td>192.168.6.7</td>
        <td>kubelet kube-proxy docker flannel etcd</td>
    </tr>
    <tr>
        <td>k8s-node2</td>
        <td>192.168.6.10</td>
        <td>kubelet kube-proxy docker flannel etcd</td>
    </tr>    
</table>

### 创建TLS证书和秘钥 ###

kubernetes系统的各组件需要使用TLS证书对通信进行加密，本文档使用CloudFlare的PKI工具集cfssl来生成Certificate Authority (CA) 和其它证书；

生成的CA证书和秘钥文件如下：

* ca-key.pem
* ca.pem
* server-key.pem
* server.pem
* kube-proxy.pem
* kube-proxy-key.pem
* admin.pem
* admin-key.pem

#####　使用证书的组件如下： #####

* etcd：使用 ca.pem、server-key.pem、server.pem；
* kube-apiserver：使用 ca.pem、server-key.pem、server.pem；
* kubelet：使用 ca.pem；
* kube-proxy：使用 ca.pem、kube-proxy-key.pem、kube-proxy.pem；
* kubectl：使用 ca.pem、admin-key.pem、admin.pem；
* kube-controller-manager：使用 ca-key.pem、ca.pem

注： 以上证书至于要在一台机器上生成，然后拷贝到其他节点即可

##### 安装CFSSL #####

    [root@k8s-master ~]# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
    [root@k8s-master ~]# chmod +x cfssl_linux-amd64
    [root@k8s-master ~]# mv cfssl_linux-amd64 /usr/local/bin/cfssl

    [root@k8s-master ~]# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
    [root@k8s-master ~]# chmod +x cfssljson_linux-amd64
    [root@k8s-master ~]# mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

    [root@k8s-master ~]# wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
    [root@k8s-master ~]# chmod +x cfssl-certinfo_linux-amd64
    [root@k8s-master ~]# mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
    [root@k8s-master ~]# source /etc/profile

##### 生成证书 #####

    [root@k8s-master ~]# mkdir /usr/local/src/ssl/
    [root@k8s-master ~]# cd /usr/local/src/ssl/
    [root@k8s-master ssl]# ll certificate.sh #证书生成脚本
     -rw-r--r-- 1 root root 2262 May 22 15:13 certificate.sh
    [root@k8s-master ssl]# sh certificate.sh
脚本查看请点击：[certificate.sh](https://github.com/Hanzhiwei210521/loading/blob/master/image/certificate.sh)

##### 安装etcd ###

    #master节点
    [root@k8s-master etcd]# pwd
    /usr/local/src/etcd
    [root@k8s-master etcd]# ls
    etcd.sh  etcd-v3.2.12-linux-amd64.tar.gz
    [root@k8s-master etcd]# tar xf etcd-v3.2.12-linux-amd64.tar.gz
    [root@k8s-master etcd]# cp etcd-v3.2.12-linux-amd64/etcd* /opt/kubernetes/bin/
    [root@k8s-master etcd]# mkdir /opt/kubernetes/{bin,cfg,ssl} -p
    [root@k8s-master etcd]# cp ../ssl/ca*.pem ssl/server*.pem /opt/kubernetes/ssl/
    [root@k8s-master etcd]# ./etcd.sh etcd01 192.168.6.12 etcd01=https://192.168.6.12:2380,etcd02=https://192.168.6.7:2380,etcd03=https://192.168.6.10:2380
    #在node节点
    [root@k8s-master etcd]# scp -r /opt/kubernetes 192.168.6.7:/opt/
    [root@k8s-master etcd]# scp etcd.sh 192.168.6.7:/usr/local/src/
    [root@k8s-master etcd]# scp -r /opt/kubernetes 192.168.6.10:/opt/
    [root@k8s-master etcd]# scp etcd.sh 192.168.6.10:/usr/local/src/
    #node1节点
    [root@k8s-node1 ~]# cd /usr/local/src/
    [root@k8s-node1 src]# ./etcd.sh etcd02 192.168.6.7 etcd01=https://192.168.6.12:2380,etcd02=https://192.168.6.7:2380,etcd03=https://192.168.6.10:2380
    #node2节点
    [root@k8s-node2 ~]# cd /usr/local/src/
    [root@k8s-node2 src]# ./etcd.sh etcd03 192.168.6.10 etcd01=https://192.168.6.12:2380,etcd02=https://192.168.6.7:2380,etcd03=https://192.168.6.10:2380
    添加环境变量   
    echo "PATH=$PATH:/opt/kubernetes/bin/" >>/etc/profile
    source /etc/profile
    查看etcd集群member
    [root@k8s-node2 src]# etcdctl --ca-file=/opt/kubernetes/ssl/ca.pem --cert-file=/opt/kubernetes/ssl/server.pem --key-file=/opt/kubernetes/ssl/server-key.pem --endpoints "https://192.168.6.10:2379" member list
    1fbd8b71e70bde1b: name=etcd03 peerURLs=https://192.168.6.10:2380 clientURLs=https://192.168.6.10:2379 isLeader=false
    47856ed020c3771a: name=etcd01 peerURLs=https://192.168.6.12:2380 clientURLs=https://192.168.6.12:2379 isLeader=true
    d684656876d2bcf8: name=etcd02 peerURLs=https://192.168.6.7:2380 clientURLs=https://192.168.6.7:2379 isLeader=false
    查看etcd集群健康状况
    [root@k8s-node2 src]# etcdctl --ca-file=/opt/kubernetes/ssl/ca.pem --cert-file=/opt/kubernetes/ssl/server.pem --key-file=/opt/kubernetes/ssl/server-key.pem --endpoints "https://192.168.6.10:2379" cluster-health
    member 1fbd8b71e70bde1b is healthy: got healthy result from https://192.168.6.10:2379
    member 47856ed020c3771a is healthy: got healthy result from https://192.168.6.12:2379
    member d684656876d2bcf8 is healthy: got healthy result from https://192.168.6.7:2379
    cluster is healthy
* [etcd.sh](https://github.com/Hanzhiwei210521/loading/blob/master/image/etcd.sh)
* [etcd-v3.2.12-linux-amd64.tar.gz](https://github.com/Hanzhiwei210521/loading/blob/master/image/etcd-v3.2.12-linux-amd64.tar.gz)

##### 安装docker #####
[Docker-CE部署安装](https://github.com/Hanzhiwei210521/loading/blob/master/Docker%20CE%E5%AE%89%E8%A3%85.md)

##### 安装flannel #####

创建flannel的子网段
    
    [root@k8s-node2 src]# etcdctl --ca-file=/opt/kubernetes/ssl/ca.pem --cert-file=/opt/kubernetes/ssl/server.pem --key-file=/opt/kubernetes/ssl/server-key.pem --endpoints=https://192.168.6.10:2379 set /jxdd/network/config '{"Network":"172.17.0.0/16","Backend": {"Type": "vxlan"}}'

flannel

    [root@k8s-node2 flannel]# pwd
    /usr/local/src/flannel
    [root@k8s-node2 flannel]# ls
    flanneld.sh  flannel-v0.9.1-linux-amd64.tar.gz
    [root@k8s-node2 flannel]# tar xf flannel-v0.9.1-linux-amd64.tar.gz
    [root@k8s-node2 flannel]# cp flanneld mk-docker-opts.sh /opt/kubernetes/bin/
    [root@k8s-node2 flannel]# ./flanneld.sh https://192.168.6.12:2379,https://192.168.6.7:2379,https://192.168.6.10:2379

* [flanneld.sh](https://github.com/Hanzhiwei210521/loading/blob/master/image/flanneld.sh)
* [flannel-v0.9.1-linux-amd64.tar.gz](https://github.com/Hanzhiwei210521/loading/blob/master/image/flannel-v0.9.1-linux-amd64.tar.gz)

查看网络：

    [root@k8s-node2 flannel]# ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
        link/ether fa:16:3e:70:96:7c brd ff:ff:ff:ff:ff:ff
        inet 192.168.6.10/16 brd 192.168.255.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::f816:3eff:fe70:967c/64 scope link 
           valid_lft forever preferred_lft forever
    3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
        link/ether a6:62:09:1c:d3:98 brd ff:ff:ff:ff:ff:ff
        inet 172.17.7.0/32 scope global flannel.1
           valid_lft forever preferred_lft forever
        inet6 fe80::a462:9ff:fe1c:d398/64 scope link 
           valid_lft forever preferred_lft forever
    4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
        link/ether 02:42:d5:95:aa:01 brd ff:ff:ff:ff:ff:ff
        inet 172.17.7.1/24 brd 172.17.7.255 scope global docker0
           valid_lft forever preferred_lft forever

##### 安装k8s #####

master部署

    [root@k8s-master ~]# cd /usr/local/src/
    [root@k8s-master src]# tar xf kubernetes-server-linux-amd64.tar.gz  
    [root@k8s-master src]# cd kubernetes
    [root@k8s-master src]# cp server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl} /opt/kubernetes/bin/
    [root@k8s-master kubernetes]# cd ../ssl/
    [root@k8s-master ssl]# sh kubeconfig.sh 
    [root@k8s-master ssl]# cp token.csv /opt/kubernetes/cfg/
    [root@k8s-master ssl]# scp *.kubeconfig 192.168.6.7:/opt/kubernetes/cfg/  
    [root@k8s-master ssl]# cd ../kubernetes
    #上传apiserver等脚本
    [root@k8s-master kubernetes]# ll *.sh
    -rw-r--r-- 1 root root 1564 May 22 22:24 apiserver.sh
    -rw-r--r-- 1 root root 1054 May 22 22:26 controller-manager.sh
    -rw-r--r-- 1 root root  637 May 22 22:25 scheduler.sh
    [root@k8s-master kubernetes]# chmod +x *.sh
    [root@k8s-master kubernetes]# ./apiserver.sh 192.168.6.12 https://192.168.6.12:2379,https://192.168.6.7:2379,https://192.168.6.10:2379
    [root@k8s-master kubernetes]# ./controller-manager.sh 
    [root@k8s-master kubernetes]# ./scheduler.sh
 查看启动是否正常
    
    [root@k8s-master kubernetes]# kubectl get cs
    NAME                 STATUS    MESSAGE              ERROR
    controller-manager   Healthy   ok                   
    scheduler            Healthy   ok                   
    etcd-2               Healthy   {"health": "true"}   
    etcd-1               Healthy   {"health": "true"}   
    etcd-0               Healthy   {"health": "true"} 
* [kubeconfig.sh](https://github.com/Hanzhiwei210521/loading/blob/master/image/kubeconfig.sh)
* [controller-manager.sh](https://github.com/Hanzhiwei210521/loading/blob/master/image/controller-manager.sh)
* [scheduler.sh](https://github.com/Hanzhiwei210521/loading/blob/master/image/scheduler.sh)

node部署

    [root@k8s-node1 ~]# cd /usr/local/src/
    [root@k8s-node1 src]# tar xf kubernetes-node-linux-amd64.tar.gz
    [root@k8s-node1 src]# cd kubernetes
    [root@k8s-node1 kubernetes]# cp node/bin/{kube-proxy,kubelet} /opt/kubernetes/bin/ 
    #上传kubelet.sh等脚本
    [root@k8s-node1 kubernetes]# ls
    kubelet.sh  kubernetes-src.tar.gz  LICENSES  node  proxy.sh
    [root@k8s-node1 kubernetes]# chmod +x *.sh
    #在master创建用户
    [root@k8s-master kubernetes]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
    #启动kubelet
    [root@k8s-node1 kubernetes]# ./kubelet.sh 192.168.6.7
    #在master上查看加入集群请求
    [root@k8s-master kubernetes]# kubectl get csr
    NAME                                                   AGE       REQUESTOR           CONDITION
    node-csr-vhKXLQJ_dFest4pzYP7A6KJ2q6w4RmsVM95k2CYTKN0   4m        kubelet-bootstrap   Pending
    #同意加入集群
    [root@k8s-master kubernetes]# kubectl certificate approve node-csr-vhKXLQJ_dFest4pzYP7A6KJ2q6w4RmsVM95k2CYTKN0
    certificatesigningrequest "node-csr-vhKXLQJ_dFest4pzYP7A6KJ2q6w4RmsVM95k2CYTKN0" approved
    [root@k8s-master kubernetes]# kubectl get node
    NAME          STATUS     ROLES     AGE       VERSION
    192.168.6.7   Ready     <none>    53s       v1.9.1
    #启动kube-proxy
    [root@k8s-node1 kubernetes]# ./proxy.sh 192.168.6.7
* [kubelet.sh](https://github.com/Hanzhiwei210521/loading/blob/master/image/kubelet.sh)
* [proxy.sh](https://github.com/Hanzhiwei210521/loading/blob/master/image/proxy.sh)

##### k8s dashboard安装 #####

    [root@k8s-master ~]# mkdir ui
    [root@k8s-master ~]# cd ui/
    #上传yaml文件
    [root@k8s-master ui]# ls
    dashboard-deployment.yaml  dashboard-rbac.yaml  dashboard-service.yaml
    [root@k8s-master ui]# kubectl create -f dashboard-rbac.yaml
    [root@k8s-master ui]# kubectl create -f dashboard-deployment.yaml
    [root@k8s-master ui]# kubectl create -f dashboard-service.yaml
    [root@k8s-master ui]# kubectl get pod -o wide -n kube-system
    NAME                                    READY     STATUS    RESTARTS   AGE       IP            NODE
    kubernetes-dashboard-698bb888c5-6vrm6   1/1       Running   0          1m        172.17.66.3   192.168.6.7
    [root@k8s-master ui]# kubectl get svc -n kube-system
    NAME                   TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
    kubernetes-dashboard   NodePort   10.10.10.99   <none>        80:37046/TCP   48s
dashboard访问url:http://192.168.6.7:37046

![](https://github.com/Hanzhiwei210521/loading/blob/master/image/image010.png)

* [dashboard-rbac.yaml](https://github.com/Hanzhiwei210521/loading/blob/master/image/dashboard-rbac.yaml)
* [dashboard-deployment.yaml](https://github.com/Hanzhiwei210521/loading/blob/master/image/dashboard-deployment.yaml)
* [dashboard-service.yaml](https://github.com/Hanzhiwei210521/loading/blob/master/image/dashboard-service.yaml)
