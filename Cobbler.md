##Cobbler自动化部署
######版本： `Version 1.0`
---
* 实验目的
>快速使用cobbler搭建项目的基本环境,来实现系统环境的快速部署.
>在工作中，经常会面临几千台甚至上万台的服务器时，仅仅安装操作系统如
>果不能够自动化完成，那比登天还难.

###Cobbler简介
---
<p>Cobbler由python语言开发（15k行python代码），Cobbler是一个快速网络安装linux的服务，而且在经过调整也可以支持网络安装windows。小巧轻便（才，使用简单的命令即可完成PXE网络安装环境的配置，同时还可以管理DHCP、DNS、TFTP、RSYNC以及yum仓库、构造系统ISO镜像。
Cobbler支持命令行管理，web界面管理，还提供了API接口，可以方便二次开发使用。
Cobbler客户端Koan支持虚拟机安装和操作系统重新安装，使重装系统更便捷。</p>
###kickstart\Cobbler对比
> kickstart

以前自动化安装系统得先设置一个网络环境，可是设置网络环境涉及到许多步骤，才能为开始安装系统做好准备

*  配置服务，比如 DHCP、TFTP、DNS、HTTP、FTP 和 NFS
*  在 DHCP 和 TFTP 配置文件中填入各个客户端机器的信息
*  创建自动部署文件（比如 kickstart 和 autoinst）
*  将安装媒介解压缩到 HTTP/FTP/NFS 存储库中

** 每一项都要对系统配置进行手动干预 **

> cobbler

不仅能够向Kickstart无人值守安装系统，Cobbler设置了一个PXE引导环境，当希望安装一台新机器时，Cobbler可以：

* 使用一个以前定义的模板来配置DHCP服务（如果启用了DHCP功能）
* 将一个存储库建立镜像或解压缩一个媒介，以注册一个新操作系统
* 在 DHCP 配置文件中为需要安装的机器创建一个条目，并使用您指定的参数（IP 和 MAC 地址）
* 在 TFTFP 服务目录下创建适当的 PXE 文件
* 重新启动 DHCP服务以反映更改
* 重新启动机器以开始安装（如果电源管理已启用）
* 使用 koan 客户端，Cobbler 可从客户端配置虚拟机并重新安装系统。

###Cobbler各对象之间的关系图
![Cobbler各对象之间的关系图](http://www.it165.net/uploadfile/2013/1105/20131105102832391.png)

###Cobbler工作原理图
![Cobbler工作原理图](http://www.it165.net/uploadfile/2013/1105/20131105102833695.png)

###基础环境准备
<pre>
>>系统版本
[root@linux-nodel ~]# cat /etc/redhat-release 
CentOS Linux release 7.2.1511 (Core) 
>>系统内核
[root@linux-nodel ~]# uname -r
3.10.0-327.el7.x86_64
>>实验ip
[root@linux-nodel ~]# ifconfig eth0|awk -F '[ :]+' 'NR==2{print $3}'
192.168.56.11
>>主机名
[root@linux-nodel ~]# hostname
linux-nodel.example.com
>>阿里epel源
[root@linux-nodel ~]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo 

>>关闭防火墙 selinux
[root@cobbler-node1 ~]# getenforce
Disabled
[root@cobbler-node1 ~]# systemctl stop firewalld
注：因为内网多个dhcp服务易产生冲突，所以使用NAT模式
</pre>


##实验开始
###安装Cobbler
<pre>yum -y install cobbler cobbler-web pykickstart httpd dhcp tftp xinetd

* cobbler 			#cobbler程序包
* cobbler-web 			#cobbler的web服务包
* pykickstart			#cobbler检查kickstart语法错误
* httpd      			#Apache web服务
* dhcp   			#Dhcp服务
* tftp  			#tftp服务
</pre>
---

>/etc/cobbler      #配置文件目录

* /etc/cobbler/settings      	   # cobbler主配置文件
* /etc/cobbler/dhcp.template      	   # DHCP服务的配置模板
* /etc/cobbler/tftpd.template      	   # tftp服务的配置模板
* /etc/cobbler/rsync.template      	   # rsync服务的配置模板
* /etc/cobbler/iso      	   # iso模板配置文件目录
* /etc/cobbler/pxe      	   # pxe模板文件目录
* /etc/cobbler/power      	   # 电源的配置文件目录
* /etc/cobbler/users.conf      	   # Web服务授权配置文件
* /etc/cobbler/users.digest      	   # web访问的用户名密码配置文件
* /etc/cobbler/dnsmasq.template      	   # DNS服务的配置模板
* /etc/cobbler/modules.conf      	   # Cobbler模块配置文件
>/var/lib/cobbler #cobbler数据目录

* /var/lib/cobbler/config  #配置文件
* /var/lib/cobbler/kickstarts #默认存放kickstart文件
* /var/lib/cobbler/loaders	#存放的各种引导程序

>/var/www/cobbler #系统安装镜像目录

* /var/www/cobbler/ks_mirror    # 导入的系统镜像列表
* /var/www/cobbler/images       # 导入的系统镜像启动文件
* /var/www/cobbler/repo_mirror  # yum源存储目录

>/var/log/cobbler               # 日志目录

* /var/log/cobbler/install.log  # 客户端系统安装日志
* /var/log/cobbler/cobbler.log  # cobbler日志

<pre>
[root@linux-nodel ~]# vim /etc/httpd/conf/httpd.conf +96
ServerName 127.0.0.1
[root@linux-nodel ~]# systemctl start httpd	#启动httpd
[root@linux-nodel ~]# systemctl start cobblerd  #启动cobblerd
</pre>

>检查配置文件(only after start httpd and cobblerd)

<pre>
1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : change 'disable' to 'no' in /etc/xinetd.d/tftp
4 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
5 : enable and start rsyncd.service with systemctl
6 : debmirror package is not installed, it will be required to manage debian deployments and repositories
7 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
8 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them
--------
cobbler check
修改：第一步+第二步
vim /etc/cobbler/settings  server:192.168.56.11 next server:192.168.56.11
manage_dhcp:1
[root@linux-nodel ~]# sed -i 's/server: 127.0.0.1/server: 192.168.56.11/g' /etc/cobbler/settings
[root@cobbler-node1 ~]# sed -i ‘s/next_server: 127.0.0.1/next_server: 192.168.56.11/’ /etc/cobbler/settings
第三步
修改/etc/xinetd.d/tftp文件中的disable参数修改为 disable = no
第四步
执行 systemctl enable rsyncd命令即可
vim /etc/init.d/xinetd.d/rsync  
change 'disable' to 'no'
/etc/init.d/xinetd restart
第七步
openssl passwd -l -salt 'oldboy' 'oldboy'
[root@linux-nodel ~]# 
grep "default_password_crypted" /etc/cobbler/settings
	default_password_crypted: "$1$oldboy$fXF8f078vI9J/q9XyXA8e/"
5，6，8 不用装
yum -y install debmirror yum-utils fence-agents

vim /etc/cobbler/dhcp.template #改完后会生成dhcpd
subnet 192.168.56.0 netmask 255.255.255.0 {
     option routers             192.168.56.2;
     option domain-name-servers 192.168.56.2;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        192.168.56.200 192.168.56.250;
systemctl restart cobblerd
</pre>

>同步cobbler
<pre>
[root@cobbler-node1~]# systemctl restart xinetd   #重启xinetd
[root@cobbler-node1~]# systemctl restart cobblerd #重启cobbler
[root@cobbler-node1~]# cobbler sync #同步最新cobbler配置，可以看具体做了哪些操作
</pre>

>配置dhcp
<pre>
[root@linux-nodel ~]# sed -i 's#manage_dhcp: 0#manage_dhcp: 1#g' /etc/cobbler/settings  # 开启cobbler管理dhcp
vim /etc/cobbler/dhcp.template #修改cobbler的dhcp模版，cobbler会自动替换。
subnet 192.168.56.0 netmask 255.255.255.0 {
     option routers             192.168.56.2;
     option domain-name-servers 192.168.56.2;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        192.168.56.200 192.168.56.250;
</pre>

>再次同步cobbler

<pre>
[root@cobbler-node1~]# systemctl restart xinetd   #重启xinetd
[root@cobbler-node1~]# systemctl restart cobblerd #重启cobbler
[root@cobbler-node1~]# cobbler sync #同步最新cobbler配置
</pre>

>挂载光盘

<pre>
mount /dev/cdrom /mnt
cobbler import --path=/mnt --name=CentOS-7.1-x86_64 --arch=x86_64
* --path镜像路径
* --name为安装源定义一个名字
* --arch指定安装源是32位还是64位
注：cobbler会将镜像中的所有安装文件拷贝到本地一份放在/var/www/cobbler/ks_mirror/CentOS-7.1-x86_64目录下，需要系统有足够的安装空间
</pre>

<pre>
cobbler distro list #列出所有distro
cobbler profile list #导入distro会自动生成profile
注:可以使用--kickstart=/path/to/kickstart_file进行导入
   so>> cobbler --import会自动为导入的distro生成一个profile
</pre>


>为了标准运维化，我们需要修改我们常用的eth0,使用下面的参数。CentOS7才需要
>CentOS6不需要
<pre>
#ks文件放到这里并通过--kickstart指出来：cd /var/lib/cobbler/kickstarts 
[root@linux-nodel kickstarts]# cobbler profile edit --name=CentOS-7.1-distro-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7.1-x86_64.cfg --kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg
#修改内核参数
[root@linux-nodel kickstarts]# cobbler profile edit --name=CentOS-7.1-distro-x86_64 --kopts='net.ifnames=0 biosdevname=0'
cobbler profile report
cobbler sync  #每次修改profile都需要同步
</pre>
###Cobbler安装界面
![Cobbler安装界面](http://cdn.xuliangwei.com/qiniu/446/image/5e2f9a2bd3560b9655ca1926fa0fd481.png?imageView2/2/w/529/h/254)
