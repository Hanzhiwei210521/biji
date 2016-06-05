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

![1465124327437][]

 [1465124327437]: data:image/png;base64,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

###自动重装系统###
* [root@localhost ~]#  yum install -y koan  

###查看可选系统
* [root@localhost ~]# koan --server=192.168.56.11 --list=profiles
    <pre>
   - looking for Cobbler at http://192.168.56.11:80/cobbler_api
    CentOS-6.6-x86_64
    CentOS-7-x86_64
    </pre>

###指定要重装的系统
* [root@localhost ~]# koan --replace-self --server=192.168.56.11 --profile=CentOS-6.6-x86_64
* [root@localhost ~]# reboot

