##Cobbler自动化部署
######版本： `Version 1.0`



---

> 目标一览
>
>1. 了解cobbler
	* 实验目的
	* Cobbler简介
	* kickstart\Cobbler对比
>2. 基础环境准备
>3. 实验开始
	* 配置文件了解
	* 安装cobblerC7
	* 安装多内核C7+C6
>5. 自动重装系统
>6. 定制化安装
	* 自定义安装
	* 自定义登录界面
	* web管理Cobbler
	* 使用api自定义安装
>7. 定义yum仓库
>8. 总结

---
##了解Cobbler
###实验目的
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


##基础环境准备
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
----
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
####如图即选择好内核后可自动安装，可有多个内核,添加多个系统
![Cobbler安装界面](https://raw.githubusercontent.com/lihao-unix/edu-docs/master/image/TITLE.png)
###添加C6内核
---
>挂载C6光盘
<pre>
mount /dev/cdrom /mnt
cobbler import --path=/mnt --name=CentOS-6.7-x86_64 --arch=x86_64
* --path镜像路径
* --name为安装源定义一个名字
* --arch指定安装源是32位还是64位
注：cobbler会将镜像中的所有安装文件拷贝到本地一份放在/var/www/cobbler/ks_mirror/CentOS-6.7-x86_64目录下，需要系统有足够的安装空间
</pre>

<pre>
cobbler distro list #列出所有distro
[root@linux-nodel ~]# cobbler distro list
   CentOS-6.7-x86_64
   CentOS-7.1-distro-x86_64

cobbler profile list #导入distro会自动生成profile

注:可以使用--kickstart=/path/to/kickstart_file进行导入
   so>> cobbler --import会自动为导入的distro生成一个profile
</pre>

<pre>
#ks文件放到这里并通过--kickstart指出来：cd /var/lib/cobbler/kickstarts 
[root@linux-nodel kickstarts]# cobbler profile edit --name=CentOS-7.1-distro-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg
cobbler profile report
cobbler sync  #每次修改profile都需要同步
</pre>
效果如图：

![RENAME-TITLE](https://raw.githubusercontent.com/lihao-unix/edu-docs/master/image/C7C6TITLE.png)
#自动化重装系统
`客户端`

	yum -y install koan
	koan --server=192.168.56.11 --list=profiles #列出cobbler系统中可以重装的系统 地址为cobbler的地址
	[root@bogon ~]# koan --server=192.168.56.11 --list=profiles
	koan --replace-self --server=192.168.56.11 --profile=CentOS-7-x86_64 #指定要重装的系统
	reboot
	* 然后重启后系统会直接安装指定系统不会进入选择界面 *
	如何解决koan安装错误机器，或者cobbler自动化安装错误机器。
	环境设计：装机Vlan

#定制化安装
* 自定义安装
* 自定义登录界面
* web管理Cobbler

###自定义安装
---
根据机器的MAC地址，自动绑定IP，网关，dns等
<pre>
[root@cobbler-node1 ~]# cobbler system add --name=xuliangwei-pc --mac=00:0C:29:6E:41:CB --profile=Centos7.1-profile-x86_64 \
--ip-address=192.168.56.13 --subnet=255.255.255.0 --gateway=10.0.0.2 --interface=eth0 \
--static=1 --hostname=lihao.pw 
--name-servers=”114.114.114.114 8.8.8.8″
</pre>
######新版的Web界面使用的是https,登录：https://192.168.56.11/cobbler_web
######cobbler_web支持多种认证方式，如`authn_configfil`、`authn_ldap`或`authn_pam`等，默认为`authn_denyall`


###更改装机界面
---
>即上图中的cobbler|http://cobbler.github.io

<pre>
[root@cobbler-node1 ~]# grep “lihao” /etc/cobbler/pxe/pxedefault.template #自定义装机页面
MENU TITLE lihao | http://lihao.com
[root@cobbler-node1 ~]# cobbler sync #同步
</pre>

![RENAME-TITLE](https://raw.githubusercontent.com/lihao-unix/edu-docs/master/image/RETITLE.png)

###web管理Cobbler
####登录web界面 `https://192.168.56.11/cobbler_web`

![cobbler-login](https://raw.githubusercontent.com/lihao-unix/edu-docs/master/image/cobbler-login.png)
<pre>更改cobbler密码：
[root@linux-nodel cobbler]# pwd
/etc/cobbler
定义用户的配置：
[root@linux-nodel cobbler]# tail -3 users.conf 
admin = ""
cobbler = ""
定义密码的配置：
[root@linux-nodel cobbler]# cat users.digest 
cobbler:Cobbler:a2d6bae81669d707b72c0bd9806e01f3

</pre>
<pre>
Cobbler-web修改密码：
[root@linux-nodel cobbler]# htdigest /etc/cobbler/users.digest "Cobbler" cobbler #"用户描述"  用户名
Changing password for user cobbler in realm Cobbler
New password:123456
Re-type new password:123456 

[root@linux-nodel cobbler]# rpm -qf `which htdigest`
httpd-tools-2.4.6-40.el7.centos.1.x86_64
</pre>

###使用api自定义安装(自定义装机平台)
cd /etc/http/conf.d/
cat cobbler.conf
21 ProxyPass /cobbler_api http://localhost:25151/
22 ProxyPassReverse /cobbler_api http://localhost:25151/

例子1：
vim cobbler_list.py
#！/usr/bin/python
import xmlrpclib
server = xmlrpclib.Server("http://192.168.56.11/cobble_api")
#print server.get_distros()
print server.get_profiles()
print server.get_systems()
#print server.get_images()
print server.get_repos()

python cobbler_list.py

###自定义yum源
<pre>
#添加openstack-mitaka最新yum源，为后期做准备
cobbler repo add --name=openstack-mitaka --mirror=http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-mitaka/ --arch=x86_64 --breed=yum

vim /etc/cobbler/settings
vim /var/lib/cobbler/CentOS-7-x86_64.cfg
#开始同步
cobbler reposync
#添加repo到对应的profile
cobbler profile edit --name=CentOS-7-x86_64 --repos="openstack-mitaka"
修改kickstart文件，添加。（到%post %end中间）
%post
systemctl disable postfix.service

$yum_config_stanza
%end
添加定时任务，定期同步repo
echo "1 3 * * * cobbler"
</pre>

##总结##
<pre>
备注：自动化装机流程

1. 服务器采购
2. 服务器验收并设置raid
3. 服务商提供验收单，运维验收负责人签字。
4. 服务器上架
5. 资产录入。
6. 开始自动化安装。   
	1. 将新服务器划入装机vlan
	2. 根据资产清单上的mac地址，自定义安装。
		1. 机房 
		2. 机房区域 
		3. 机柜  
		4. 服务器位置
		5. 服务器网线接入端口 
		6. 该端口mac地址 
		7. profile 操作系统 分区等 预分配的ip地址  主机名 子网  网关 dns  角色。
						
	3.自动化装机平台，安装。
	00:50:56:31:6C:DF
	IP:192.168.56.12  
	主机名：linux-node2.oldboyedu.com 
	掩码：255.255.255.0 
	网关：192.168.56.2 
	DNS：192.168.56.2
cobbler system add --name=linux-node2.oldboyedu.com --mac=00:50:56:31:6C:DF --profile=CentOS-7-x86_64 \
--ip-address=192.168.56.12 --subnet=255.255.255.0 --gateway=192.168.56.2 --interface=eth0 \
--static=1 --hostname=linux-node2.oldboyedu.com --name-servers="192.168.56.2" \
--kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg

</pre>

