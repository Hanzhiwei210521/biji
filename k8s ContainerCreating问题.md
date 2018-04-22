# k8s创建容器pod ContainerCreating问题 #

k8s集群部署完，我们在master尝试启动一个容器试试

    [root@k8s-master ~]# kubectl run nginx --image=nginx --replicas=2
    deployment "nginx" created
    [root@k8s-master ~]# kubectl get pod
    NAME                   READY     STATUS              RESTARTS   AGE
    nginx-8586cf59-88rsd   0/1       ContainerCreating   0          15s
    nginx-8586cf59-tdmgz   0/1       ContainerCreating   0          15s
等了一会，发现一直处于这个状态，状态不对，使用describe查看一下

    [root@k8s-master ~]# kubectl describe pod
    Name:           nginx-8586cf59-88rsd
    Namespace:      default
    Node:           192.168.6.5/192.168.6.5
    Start Time:     Wed, 18 Apr 2018 09:25:03 +0800
    Labels:         pod-template-hash=41427915
                run=nginx
    Annotations:    <none>
    Status:         Pending
    IP:             
    Controlled By:  ReplicaSet/nginx-8586cf59
    Containers:
      nginx:
        Container ID:   
        Image:          nginx
        Image ID:       
        Port:           <none>
        State:          Waiting
          Reason:       ContainerCreating
        Ready:          False
        Restart Count:  0
        Environment:    <none>
        Mounts:         <none>
    Conditions:
      Type           Status
      Initialized    True 
      Ready          False 
      PodScheduled   True 
    Volumes:         <none>
    QoS Class:       BestEffort
    Node-Selectors:  <none>
    Tolerations:     <none>
    Events:
      Type     Reason                  Age                From                  Message
      ----     ------                  ----               ----                  -------
      Normal   Scheduled               11m                default-scheduler     Successfully assigned nginx-8586cf59-88rsd to 192.168.6.5
      Warning  FailedCreatePodSandBox  0s (x24 over 10m)  kubelet, 192.168.6.5  Failed create pod sandbox.
返现最后有一个Failed create pod sandbox ,原因为kubelet在创建pod时会先下载一个pause镜像，这个镜像用于容器基础网络管理，但是这个镜像仓库（gcr.io/google_containers/pause-amd64:3.0）是国外的，不能顺利下载。

#### 解决办法 ####

1、在每个node节点手动下载（用国内的源），然后打一个tag

    docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0
    docker tag registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0

2、在kubelet选项指定下载源

     --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0

3、翻墙

在node1执行1方法后

    [root@kub-master ~]# kubectl get pod
    NAME                                READY     STATUS              RESTARTS   AGE
    nginx-deployment-569477d6d8-lqnsz   0/1       ContainerCreating   0          27m
    nginx-deployment-569477d6d8-n7wgq   1/1       Running             0          27m
可以看到已经创建成功，另一个节点因为没有源还无法下载。