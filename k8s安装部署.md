# k8s安装部署 *

k8s1.9下载地址：<https://v1-9.docs.kubernetes.io/docs/imported/release/notes/>

下载如下两个二进制包：

* kubernetes-server-linux-amd64.tar.gz
* kubernetes-node-linux-amd64.tar.gz

### 环境准备：###
* 192.168.6.15 k8s-master
* 192.168.6.5 k8s-node1
* 192.168.6.2 k8s-node2

### 操作系统及内核版本 ###

    [root@k8s-master ~]# cat /etc/redhat-release 
    CentOS Linux release 7.2.1511 (Core) 
    [root@k8s-master ~]# uname -r
     4.16.2-1.el7.elrepo.x86_64
同时，每台机器部署一个etcd，组成etcd集群

    [root@k8s-master ~]# etcdctl --endpoints=http://192.168.6.15:2379 member list
    18e03f3590bf045c: name=etcd-2 peerURLs=http://192.168.6.2:2380 clientURLs=http://192.168.6.2:2379 isLeader=false
    bb1bb690e007828d: name=etcd-1 peerURLs=http://192.168.6.15:2380 clientURLs=http://192.168.6.15:2379 isLeader=false
    de68aea055a90bca: name=etcd-3 peerURLs=http://192.168.6.5:2380 clientURLs=http://192.168.6.5:2379 isLeader=true
    [root@k8s-master ~]# etcdctl --endpoints=http://192.168.6.15:2379 cluster-health
    member 18e03f3590bf045c is healthy: got healthy result from http://192.168.6.2:2379
    member bb1bb690e007828d is healthy: got healthy result from http://192.168.6.15:2379
    member de68aea055a90bca is healthy: got healthy result from http://192.168.6.5:2379
    cluster is healthy

### k8s master部署 ###
    [root@k8s-master ~]# cd /usr/local/src/
    [root@k8s-master src]# tar xf kubernetes-server-linux-amd64.tar.gz  
    [root@k8s-master src]# cd kubernetes
    [root@k8s-master kubernetes]# mkdir /usr/local/src/kubernetes/kubernetes-src
    [root@k8s-master kubernetes]# mkdir /opt/kubernetes/{cfg,bin} -p
    [root@k8s-master kubernetes]# tar xf kubernetes-src.tar.gz -C /usr/local/src/kubernetes/kubernetes-src
    [root@k8s-master kubernetes]# cp server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler} /opt/kubernetes/bin/
    [root@k8s-master kubernetes]# cp kubernetes-src/cluster/centos/master/scripts/{apiserver.sh,controller-manager.sh,scheduler.sh} .
上面进行了一大通操作，其实也没啥东西，先是将二进制包解压，然后创建工作目录，拷贝可执行文件到工作目录下，kubernetes-src.tar.gz 包内为k8s的一些源码文件，我们主要是用里面的脚本生成k8s的配置文件 

我们先看一个配置：

    [root@k8s-master kubernetes]# cat apiserver.sh 
    #!/bin/bash

    # Copyright 2014 The Kubernetes Authors.
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.


    MASTER_ADDRESS=${1:-"8.8.8.18"}
    ETCD_SERVERS=${2:-"https://8.8.8.18:2379"}
    SERVICE_CLUSTER_IP_RANGE=${3:-"10.10.10.0/24"}
    ADMISSION_CONTROL=${4:-""}

    cat <<EOF >/opt/kubernetes/cfg/kube-apiserver
    # --logtostderr=true: log to standard error instead of files
    KUBE_LOGTOSTDERR="--logtostderr=true"

    # --v=0: log level for V logs
    KUBE_LOG_LEVEL="--v=4"

    # --etcd-servers=[]: List of etcd servers to watch (http://ip:port),
    # comma separated. Mutually exclusive with -etcd-config
    KUBE_ETCD_SERVERS="--etcd-servers=${ETCD_SERVERS}"

    # --etcd-cafile="": SSL Certificate Authority file used to secure etcd communication.
    #KUBE_ETCD_CAFILE="--etcd-cafile=/srv/kubernetes/etcd/ca.pem"

    # --etcd-certfile="": SSL certification file used to secure etcd communication.
    #KUBE_ETCD_CERTFILE="--etcd-certfile=/srv/kubernetes/etcd/client.pem"

    # --etcd-keyfile="": key file used to secure etcd communication.
    #KUBE_ETCD_KEYFILE="--etcd-keyfile=/srv/kubernetes/etcd/client-key.pem"

    # --insecure-bind-address=127.0.0.1: The IP address on which to serve the --insecure-port.
    KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"

    # --insecure-port=8080: The port on which to serve unsecured, unauthenticated access.
    KUBE_API_PORT="--insecure-port=8080"

    # --kubelet-port=10250: Kubelet port
    NODE_PORT="--kubelet-port=10250"

    # --advertise-address=<nil>: The IP address on which to advertise
    # the apiserver to members of the cluster.
    KUBE_ADVERTISE_ADDR="--advertise-address=${MASTER_ADDRESS}"

    # --allow-privileged=false: If true, allow privileged containers.
    KUBE_ALLOW_PRIV="--allow-privileged=false"

    # --service-cluster-ip-range=<nil>: A CIDR notation IP range from which to assign service cluster IPs.
    # This must not overlap with any IP ranges assigned to nodes for pods.
    KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=${SERVICE_CLUSTER_IP_RANGE}"

    # --admission-control="AlwaysAdmit": Ordered list of plug-ins
    # to do admission control of resources into cluster.
    # Comma-delimited list of:
    #   LimitRanger, AlwaysDeny, SecurityContextDeny, NamespaceExists,
    #   NamespaceLifecycle, NamespaceAutoProvision, AlwaysAdmit,
    #   ServiceAccount, DefaultStorageClass, DefaultTolerationSeconds, ResourceQuota
    KUBE_ADMISSION_CONTROL="--admission-control=${ADMISSION_CONTROL}"

    # --client-ca-file="": If set, any request presenting a client certificate signed
    # by one of the authorities in the client-ca-file is authenticated with an identity
    # corresponding to the CommonName of the client certificate.
    #KUBE_API_CLIENT_CA_FILE="--client-ca-file=/srv/kubernetes/ca.crt"

    # --tls-cert-file="": File containing x509 Certificate for HTTPS.  (CA cert, if any,
    # concatenated after server cert). If HTTPS serving is enabled, and --tls-cert-file
    # and --tls-private-key-file are not provided, a self-signed certificate and key are
    # generated for the public address and saved to /var/run/kubernetes.
    #KUBE_API_TLS_CERT_FILE="--tls-cert-file=/srv/kubernetes/server.cert"

    # --tls-private-key-file="": File containing x509 private key matching --tls-cert-file.
    #KUBE_API_TLS_PRIVATE_KEY_FILE="--tls-private-key-file=/srv/kubernetes/server.key"
    EOF

    KUBE_APISERVER_OPTS="   \${KUBE_LOGTOSTDERR}         \\
                            \${KUBE_LOG_LEVEL}           \\
                            \${KUBE_ETCD_SERVERS}        \\
                            \${KUBE_API_ADDRESS}         \\
                            \${KUBE_API_PORT}            \\
                            \${NODE_PORT}                \\
                            \${KUBE_ADVERTISE_ADDR}      \\
                            \${KUBE_ALLOW_PRIV}          \\
                            \${KUBE_SERVICE_ADDRESSES}   \\
                            \${KUBE_ADMISSION_CONTROL}"


    cat <<EOF >/usr/lib/systemd/system/kube-apiserver.service
    [Unit]
    Description=Kubernetes API Server
    Documentation=https://github.com/kubernetes/kubernetes

    [Service]
    EnvironmentFile=-/opt/kubernetes/cfg/kube-apiserver
    ExecStart=/opt/kubernetes/bin/kube-apiserver ${KUBE_APISERVER_OPTS}
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    EOF

    systemctl daemon-reload
    systemctl enable kube-apiserver
    systemctl restart kube-apiserve
上面的配置是经过修改过的，只是将一些认证的配置注释掉了，其余没修改，controller-manager.sh也需要注释认证文件，文件的大致意思是需要传三个参数，第一个为k8s master的IP地址，第二个为etcd集群的地址，第三个为 SERVICE_CLUSTER_IP的地址段；然后执行脚本会自动生成k8s的配置文件，以及加入开机自启。

k8s master跑三个进程，apiserver,controller-manager,scheduler, 依次执行脚本

    [root@k8s-master kubernetes]# sh apiserver.sh 192.168.6.15 http://192.168.6.15:2379,http://192.168.6.2:2379,http://192.168.6.5:2379 10.10.10.0/24
    [root@k8s-master kubernetes]# sh controller-manager.sh 192.168.6.15
    [root@k8s-master kubernetes]# sh scheduler.sh 192.168.6.15
### k8s node部署 ###
k8s node部署方式同master，node节点主要运行kubelet和kube-proxy

    [root@k8s-node1 ~]# cd /usr/local/src/
    [root@k8s-node1 ~]# tar xf kubernetes-node-linux-amd64.tar.gz 
    [root@k8s-node1 ~]# cd kubernetes
    [root@k8s-node1 ~]# mkdir kubernetes-src
    [root@k8s-node1 ~]# mkdir /opt/kubernetes/{cfg,bin} -p
    [root@k8s-node1 kubernetes]# cp node/bin/{kubelet,kube-proxy} /opt/kubernetes/bin/
    [root@k8s-node1 kubernetes]# cp kubernetes-src/cluster/centos/node/scripts/{kubelet.sh,proxy.sh} .
    [root@k8s-node1 kubernetes]# sh kubelet.sh 192.168.6.15 192.168.6.5 10.10.10.2
    [root@k8s-node1 kubernetes]# sh proxy.sh 192.168.6.15 192.168.6.5
k8s node2部署同上，三台机器依次配置环境变量：

    [root@k8s-master ~]# echo "PATH=$PATH:/opt/kubernetes/bin/" >>/etc/profile
    [root@k8s-master ~]# sort /etc/profile

k8s master查看node节点：
 
    [root@k8s-master ~]# kubectl get node
    NAME          STATUS    ROLES     AGE       VERSION
    192.168.6.2   Ready     <none>    4h        v1.9.1
    192.168.6.5   Ready     <none>    5h        v1.9.1
部署完毕，有没有很简单，哈哈！！！
    
    