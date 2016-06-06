# cobbler自动化部署Linux系统 #

###基础环境准备
###系统版本

    [root@linux-node1 ~]# cat /etc/redhat-release   
    CentOS Linux release 7.2.1511 (Core)
###内核版本
   
    [root@linux-node1 ~]# uname -r
    3.10.0-327.18.2.el7.x86_64
###检查selinux是否关闭

    [root@linux-node1 ~]# getenforce 
    Disabled
###关闭防火墙

    [root@linux-node1 ~]# systemctl stop firewalld
    [root@linux-node1 ~]# systemctl disable firewalld.service
###本机IP
<pre>
[root@linux-node1 ~]# ifconfig eth0|awk 'NR==2{print $2}'
192.168.56.11
</pre>
###主机名
<pre>
[root@linux-node1 ~]# hostname
linux-node1.example.com
</pre>
###安装epel源
<pre>
[root@linux-node1 ~]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo 
</pre> 
提示：
虚拟机网卡要采用NAT模式，因为我们会搭建DHCP服务器，在同一局域网多个DHCP服务会有冲突，并且导致实验失败。
###安装服务包
<pre>
[root@linux-node1 ~]# yum install cobbler cobbler-web pykickstart httpd dhcp xinetd -y
</pre>
###启动cobbler服务及http服务
<pre>
[root@linux-node1 ~]# systemctl start httpd
[root@linux-node1 ~]# systemctl start cobblerd
</pre>
###检查cobbler语法
<pre>
[root@linux-node1 ~]# cobbler check
1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : change 'disable' to 'no' in /etc/xinetd.d/tftp
4 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
5 : enable and start rsyncd.service with systemctl
6 : debmirror package is not installed, it will be required to manage debian deployments and repositories
7 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
8 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.
</pre>
###根据报错信息进行修改配置文件

* [root@linux-node1 ~]# vi /etc/cobbler/settings 
    
    <pre>
   next_server: 192.168.56.11   #tftp服务器地址
   server: 192.168.56.11        #主机地址
   manage_dhcp: 1               #设置为1表示用cobbler来管理dhcp服务
    </pre>   
* [root@linux-node1 ~]# vi /etc/xinetd.d/tftp
   
    <pre>
    disable                 = no
    </pre>
* [root@linux-node1 ~]# cobbler get-loaders
* [root@linux-node1 ~]# systemctl enable rsyncd.service
* [root@linux-node1 ~]# openssl passwd -1 -salt 'cobbler' 'cobbler'
   
    <pre>
    $1$cobbler$M6SE55xZodWc9.vAKLJs6.
    </pre>
* [root@linux-node1 ~]# vi /etc/cobbler/settings
   
    <pre>
    default_password_crypted: "$1$cobbler$M6SE55xZodWc9.vAKLJs6."
    </pre>
* [root@linux-node1 ~]# yum install cman fence-agents -y
* [root@linux-node1 ~]# systemctl restart cobblerd

###修改cobbler的dhcp模块，用cobbler管理dhcp服务
* [root@linux-node1 ~]# vi /etc/cobbler/dhcp.template

    <pre>
     subnet 192.168.56.0 netmask 255.255.255.0 {
        option routers             192.168.56.2;
        option domain-name-servers 192.168.56.2;
        option subnet-mask         255.255.255.0;
        range dynamic-bootp        192.168.56.100 192.168.56.254;
        default-lease-time         21600;
        max-lease-time             43200;
        next-server                $next_server;
    </pre>
* [root@linux-node1 ~]# systemctl restart xinetd
* [root@linux-node1 ~]# systemctl restart cobblerd

###同步最新cobbler配置，可以看具体做了哪些操作
* [root@linux-node1 ~]# cobbler sync  
  
    <pre>
task started: 2016-06-02_163039_sync
task started (id=Sync, time=Thu Jun  2 16:30:39 2016)
running pre-sync triggers
cleaning trees
removing: /var/lib/tftpboot/grub/images
copying bootloaders
trying hardlink /var/lib/cobbler/loaders/pxelinux.0 -> /var/lib/tftpboot/pxelinux.0
trying hardlink /var/lib/cobbler/loaders/menu.c32 -> /var/lib/tftpboot/menu.c32
trying hardlink /var/lib/cobbler/loaders/yaboot -> /var/lib/tftpboot/yaboot
trying hardlink /usr/share/syslinux/memdisk -> /var/lib/tftpboot/memdisk
trying hardlink /var/lib/cobbler/loaders/grub-x86.efi -> /var/lib/tftpboot/grub/grub-x86.efi
trying hardlink /var/lib/cobbler/loaders/grub-x86_64.efi -> /var/lib/tftpboot/grub/grub-x86_64.efi
copying distros to tftpboot
copying images
generating PXE configuration files
generating PXE menu structure
rendering DHCP files
generating /etc/dhcp/dhcpd.conf
rendering TFTPD files
generating /etc/xinetd.d/tftp
cleaning link caches
running post-sync triggers
running python triggers from /var/lib/cobbler/triggers/sync/post/*
running python trigger cobbler.modules.sync_post_restart_services
running: dhcpd -t -q
received on stdout: 
received on stderr: 
running: service dhcpd restart
received on stdout: 
received on stderr: Redirecting to /bin/systemctl restart  dhcpd.service
running shell triggers from /var/lib/cobbler/triggers/sync/post/*
running python triggers from /var/lib/cobbler/triggers/change/*
running python trigger cobbler.modules.scm_track
running shell triggers from /var/lib/cobbler/triggers/change/*
*** TASK COMPLETE ***
   </pre>
###系统挂载并导入镜像
* [root@linux-node1 ~]# mount /dev/cdrom /mnt 

* [root@linux-node1 ~]# cobbler import --path=/mnt/ --name=CentOS-7-x86\_64 --arch=x86_64

###导入ks.cfg文件
* [root@linux-node1 ~]# cobbler profile edit --name=CentOS-7-X86\_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg

###更改内核参数，统一网卡名称
* [root@linux-node1 ~]# cobbler profile edit --name=CentOS-7-x86_64 --kopts='net.ifnames=0 biosdevname=0'

###同步cobbler配置
* [root@linux-node1 ~]# cobbler sync 
###部署操作系统

新建一台虚拟机，通过网络启动
![](https://raw.githubusercontent.com/Hanzhiwei210521/loading/master/image/image001.png)

###自动重装系统###
* [root@localhost ~]#  yum install -y koan  

###查看可选系统
* [root@localhost ~]# koan --server=192.168.56.11 --list=profiles
    <pre>
   -looking for Cobbler at http://192.168.56.11:80/cobbler_api
    CentOS-6.6-x86_64
    CentOS-7-x86_64
    </pre>

###指定要重装的系统
* [root@localhost ~]# koan --replace-self --server=192.168.56.11 --profile=CentOS-6.6-x86_64
* [root@localhost ~]# reboot

