# Centos7升级内核版本 #

最近在学习docker过程中，发现内核版本越新，docker对其越友好，换言之，如果要使用最新版本的docker，相应的也需要使用新版本的内核。

在Centos7内核升级前，我们先看看现在的系统版本和内核信息，如下：

    [root@bogon ~]# cat /etc/redhat-release 
    CentOS Linux release 7.3.1611 (Core) 
    [root@bogon ~]# uname -a
    Linux bogon 3.10.0-514.21.1.el7.x86_64 #1 SMP Thu May 25 17:04:51 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
通常情况下我们会使用yum update升级软件包和系统内核，但是这只会升级内核到仓库中可用的最新版本，而不是在<https://www.kernel.org/>中可用的最新版本。

我使用yum update升级了一下，如下：

    [root@bogon ~]# yum update -y
    [root@bogon ~]# reboot
    [root@bogon ~]# cat /etc/redhat-release 
    CentOS Linux release 7.4.1708 (Core) 
    [root@bogon ~]# uname -a
    Linux bogon 3.10.0-693.21.1.el7.x86_64 #1 SMP Thu May 25 17:04:51 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
对比一下，发现系统版本升级了，但是内核只升级了小版本，下面我们使用ELRepo升级，这是一个第三方仓库，可以将内核升级到最新版本。

    [root@bogon ~]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
    [root@bogon ~]# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
仓库启用后，你可以使用下面的命令列出可用的内核相关包：

    [root@bogon ~]# yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
    Loaded plugins: fastestmirror
    elrepo-kernel                                                                                                                                      | 2.9 kB  00:00:00     
    elrepo-kernel/primary_db                                                                                                                           | 1.7 MB  00:00:02     
    Loading mirror speeds from cached hostfile
     * elrepo-kernel: hkg.mirror.rackspace.com
    Available Packages
    kernel-lt.x86_64                                                                     4.4.127-1.el7.elrepo                                                    elrepo-kernel
    kernel-lt-devel.x86_64                                                               4.4.127-1.el7.elrepo                                                    elrepo-kernel
    kernel-lt-doc.noarch                                                                 4.4.127-1.el7.elrepo                                                    elrepo-kernel
    kernel-lt-headers.x86_64                                                             4.4.127-1.el7.elrepo                                                    elrepo-kernel
    kernel-lt-tools.x86_64                                                               4.4.127-1.el7.elrepo                                                    elrepo-kernel
    kernel-lt-tools-libs.x86_64                                                          4.4.127-1.el7.elrepo                                                    elrepo-kernel
    kernel-lt-tools-libs-devel.x86_64                                                    4.4.127-1.el7.elrepo                                                    elrepo-kernel
    kernel-ml.x86_64                                                                     4.16.2-1.el7.elrepo                                                     elrepo-kernel
    kernel-ml-devel.x86_64                                                               4.16.2-1.el7.elrepo                                                     elrepo-kernel
    kernel-ml-doc.noarch                                                                 4.16.2-1.el7.elrepo                                                     elrepo-kernel
    kernel-ml-headers.x86_64                                                             4.16.2-1.el7.elrepo                                                     elrepo-kernel
    kernel-ml-tools.x86_64                                                               4.16.2-1.el7.elrepo                                                     elrepo-kernel
    kernel-ml-tools-libs.x86_64                                                          4.16.2-1.el7.elrepo                                                     elrepo-kernel
    kernel-ml-tools-libs-devel.x86_64                                                    4.16.2-1.el7.elrepo                                                     elrepo-kernel
    perf.x86_64                                                                          4.16.2-1.el7.elrepo                                                     elrepo-kernel
    python-perf.x86_64 

安装最新的主线稳定内核：

    [root@bogon ~]# yum --enablerepo=elrepo-kernel install kernel-ml -y

内核升级完后，不会立即生效，需要重启，重启就需要我们先修改grub默认的内核版本。

原文件如下：

    [root@kub-master ~]# cat /etc/default/grub 
    GRUB_TIMEOUT=5
    GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
    GRUB_DEFAULT=saved  ###需要将 “saved” 改为 “0”
    GRUB_DISABLE_SUBMENU=true
    GRUB_TERMINAL_OUTPUT="console"
    GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap  rhgb quiet"
    GRUB_DISABLE_RECOVERY="true"

修改后如下：

    [root@bogon ~]# cat /etc/default/grub 
    GRUB_TIMEOUT=5
    GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
    GRUB_DEFAULT=0
    GRUB_DISABLE_SUBMENU=true
    GRUB_TERMINAL_OUTPUT="console"
    GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet"
    GRUB_DISABLE_RECOVERY="true"
运行grub2-mkconfig 命令来重新创建内核配置，如下：

    [root@bogon ~]# grub2-mkconfig -o /boot/grub2/grub.cfg
重启系统，查看内核版本：
  
    [root@bogon ~]# reboot
    [root@bogon ~]# uname -a
    Linux bogon 4.16.2-1.el7.elrepo.x86_64 #1 SMP Thu Apr 12 09:08:05 EDT 2018 x86_64 x86_64 x86_64 GNU/Linux



















