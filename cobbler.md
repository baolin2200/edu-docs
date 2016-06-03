# Cobbler自动化部署最佳实践
> Version 1.0  2016-06-01
***
## DNS集群实验目的

    为了快速使用cobbler搭建项目的基本环境，也为了让让我们更快的了解cobbler技术，在遇到新技术、新问题的时候，我们面对问题、解决问题的态度、思路等，公司项目负责人布置了搭建DNS集群的测试平台任务。
    该平台采用一种软件cobbler的最新版本来实现系统环境的快速部署。
    要求：测试环境中所有用到的机器都要自动快速部署。
## 自动化部署实验基本规划
### 实验环境硬件及操作系统规划
#### 实验环境用到的硬件和操作系统以及虚拟机的相关规划 见表1和表2

	表1 硬件笔记本测试搭建实验环境的配置
    	----------------------------------------------------------------------
      	  设备名称		          配置描述
           	CPU	            	英特尔 第二代酷睿 i5-2540M @ 2.60GHz 双核
	   		内存		    		8 GB
	  	 	硬盘             	主：80G/固态硬盘   从：1T/机械硬盘

    表2 软件配置
    	----------------------------------------------------------------------
		 	软件名称			 	配置描述
    	window7 操作系统	    	以64位 window7 系统作为 VM 的宿主机
    	VMware Workstation		采用VMware Workstation 10，虚拟 1 个VM
    	Centos7 操作系统			虚拟主机采用Centos7 内核版本为3.10.0 的64位系统

#### 实验节点的IP地址及主机名规划

	ip地址及主机名规划见表3和表4
    
    表3 集群节点ip地址及主机名规划
    ------------------------------------------------------
	服务器说明		ip地址	  网卡规划		主机名规划
	  主机		192.168.56.11	eth0		linux-node2
				192.168.2.11	eth1		linux-node2

	表4 ip地址及软件版本规划
    -------------------------------------------------------
	服务器说明		ip地址		软件规划	 	软件版本
	  主机	   	192.168.56.11	cobbler		2.6.11
				192.168.2.11	cobbler		2.6.11
#### 实验虚拟机网络规划
    由于本实验在VMware上进行，而且使用到了网络，所以需要对虚拟机进行基本的网络规划。
    
    表5 虚拟机网络规划
    ------------------------------------------------------------------------------------------
		实体机ip	     	虚拟机ip	  		网络连接    网卡   		外部连接    
	  192.168.1.101  192.168.56.11   	NAT模式    eth0   英特尔 82579LM Gigabit Network Connection
				   	 192.168.2.11		仅主机模式  eth1   VMware Virtual Ethernet Adapter for VMnet2

    表6 虚拟机 resolv.conf 具体内容
    -----------------------------------------
	nameserver 202.106.0.20
	nameserver 219.141.136.10
**注意：nat模式，网卡的网关和dns地址都要是nat网卡的ip地址**

## 机器的基本信息
> cat /etc/redhat-release 

		CentOS Linux release 7.2.1511 (Core)
 
> uname -r

		3.10.0-327.18.2.el7.x86_64

> getenforce 

		Disabled

> systemctl stop firewalld
> ifconfig eth0|awk -F '[ :]+' 'NR==2 {print $3}'

		192.168.56.11

> hostname

		linux-node2.example.com
## 配置软件基本环境
	rpm -ivh  http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
	yum groupinstall "Compatibility libraries" "Base" "Development tools" "debugging Tools" -y
	yum update upgrade -y
	
## 时间同步配置
	yum -y install ntpdate && ntpdate -u  s2m.time.edu.cn
	echo "*/5 * * * * /usr/sbin/ntpdate time.nist.gov >/dev/null 2>&1" >>/var/spool/cron/root
## 安装cobbler环境软件包[一般是需要安装33个包]
	yum install cobbler cobbler-web pykickstart httpd tftp dhcp xinetd -y
## 配置tftp
	vim /etc/xinetd.d/tftp
		disable                 = no

## 启动软件
	systemctl start httpd
	systemctl start rsyncd
	systemctl start cobblerd
## 检查cobbler配置文件
	cobbler check
## 修改cobbler的配置文件
	sed -i 's/server: 127.0.0.1/server: 192.168.56.11/' /etc/cobbler/settings
	sed -i 's/next_server: 127.0.0.1/next_server: 192.168.56.11/' /etc/cobbler/settings
	sed -i 's#manage_dhcp: 0#manage_dhcp: 1#g' /etc/cobbler/settings
## 生成cobbler部署环境的用户名和密码
	openssl passwd -1 -salt 'cobler' 'cobler'
		$1$cobler$XJnisBweZJlhL651HxAM00
    vim /etc/cobbler/settings
		将刚才openssl生成的密码，复制到第101行的引号内
		default_password_crypted: "$1$cobler$XJnisBweZJlhL651HxAM00"
## 编辑cobbler的dhcp配置文件中的ip段和网关地址
	vim /etc/cobbler/dhcp.template
		subnet 192.168.56.0 netmask 255.255.255.0 {
		     option routers             192.168.56.2;
		     option domain-name-servers 192.168.56.2;
		     option subnet-mask         255.255.255.0;
		     range dynamic-bootp        192.168.56.100 192.168.56.254;

## 重新启动和检查cobbler
	cobbler get-loaders
	systemctl restart cobblerd
	cobbler check
	cobbler sync

## 配置虚拟主机的光驱，
	在虚拟机的光驱配置处，浏览一个Centos-7-x86_64的系统，然后连接成功【设备状体为：已连接】后，登录系统
	mount /dev/cdrom /mnt
## 加载光驱系统文件
	cobbler import --path=/mnt/ --name=CentOS-7-x86_64 --arch=x86_64
**注意：导入的过程很慢，毕竟要拷贝文件到虚拟机**

> 导入的文件在哪里呢？ 

	ls /var/www/cobbler/ks_mirror/
>当Centos7导入完毕后，umount卸掉光驱，我们再重复刚才的那个步骤，调整光驱系统为Centos6，然后从新导入一次。
	
	cobbler import --path=/mnt/ --name=CentOS-6-x86_64 --arch=x86_64
>上传成功后，重新查看
	
	ls /var/www/cobbler/ks_mirror/
		CentOS-6-x86_64  CentOS-7-x86_64  config
	cobbler profile
查看当前可以安装的系统列表
> cobbler profile list
		CentOS-6-x86_64
		CentOS-7-x86_64
查看当前可以安装的系统列表的详细信息
	cobbler profile report

## 上传ks文件
> ls /var/lib/cobbler/kickstarts/Cen*

	/var/lib/cobbler/kickstarts/CentOS-6-x86_64.cfg  /var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg

> cobbler profile edit --name=CentOS-7-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg
> cobbler profile edit --name=CentOS-6-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-6-x86_64.cfg

## 修改启动时候的内核参数
> cobbler profile edit --name=CentOS-7-x86_64 --kopts='net.ifnames=0 biosdevname=0'                                                     
> cobbler sync

## 观察日志
> tail -f /var/log/messages

		Jun  2 06:27:01 linux-node2 dhcpd: No subnet declaration for eth1 (192.168.2.11).
		Jun  2 06:27:01 linux-node2 dhcpd: ** Ignoring requests on eth1.  If this is not what
		Jun  2 06:27:01 linux-node2 dhcpd:   you want, please write a subnet declaration
		Jun  2 06:27:01 linux-node2 dhcpd:   in your dhcpd.conf file for the network segment
		Jun  2 06:27:01 linux-node2 dhcpd:   to which interface eth1 is attached. **
		Jun  2 06:27:01 linux-node2 dhcpd: 
		Jun  2 06:27:01 linux-node2 dhcpd: Listening on LPF/eth0/00:0c:29:d5:d6:be/192.168.56.0/24
		Jun  2 06:27:01 linux-node2 dhcpd: Sending on   LPF/eth0/00:0c:29:d5:d6:be/192.168.56.0/24
		Jun  2 06:27:01 linux-node2 dhcpd: Sending on   Socket/fallback/fallback-net
		Jun  2 06:27:01 linux-node2 systemd: Started DHCPv4 Server Daemon.【dhcp状态正常】

## 然后新建立一台虚拟机
所有参数都默认，然后打开电源即可，观察是否连接dhcp后直接就启动了

![开机启动](http://i.imgur.com/34pRUTQ.jpg)

**注意:这个地方必须在makedown自己上传图片后才能生效**
## 继续观察，刚才的日志，确认是否正常开机
		Jun  2 06:29:56 linux-node2 dhcpd: DHCPDISCOVER from 00:0c:29:ac:6a:0b via eth0
		Jun  2 06:29:57 linux-node2 dhcpd: DHCPOFFER on 192.168.56.101 to 00:0c:29:ac:6a:0b via eth0
		Jun  2 06:29:58 linux-node2 dhcpd: DHCPREQUEST for 192.168.56.101 (192.168.56.11) from 00:0c:29:ac:6a:0b v
		ia eth0Jun  2 06:29:58 linux-node2 dhcpd: DHCPACK on 192.168.56.101 to 00:0c:29:ac:6a:0b via eth0
		Jun  2 06:29:58 linux-node2 xinetd[1320]: START: tftp pid=2299 from=192.168.56.101
		Jun  2 06:29:58 linux-node2 in.tftpd[2300]: tftp: client does not accept options
		Jun  2 06:30:01 linux-node2 systemd: Started Session 75 of user root.
		Jun  2 06:30:01 linux-node2 systemd: Starting Session 75 of user root.
		Jun  2 06:30:01 linux-node2 systemd: Started Session 74 of user root.
		Jun  2 06:30:01 linux-node2 systemd: Starting Session 74 of user root.
## 安装好系统后，登录web管理页面
### 配置web管理界面的用户名和密码
	htdigest /etc/cobbler/users.digest "Cobbler" cobbler
### 登录管理界面
	https://192.168.56.11/cobbler_web
**此处需要有图片**

自动装机实验成功

## 重装系统
在装好的机器上，我感觉这个系统没有安装好，需要重装一下另外版本系统
### 安装软件
    yum install -y koan
### 查看可安装的系统列表
	koan --server=192.168.56.11 --list=profiles
		- looking for Cobbler at http://192.168.56.11:80/cobbler_api
		CentOS-7-x86_64
		CentOS-6-x86_64
### 开始重装系统
	koan --replace-self --server=192.168.56.11 --profile=CentOS-6-x86_64


### 修改启动时候的提示单

> vim /etc/cobbler/pxe/pxedefault.template

	DEFAULT menu
	PROMPT 0
	MENU TITLE Cobbler | http://cobbler.github.io
	TIMEOUT 200
	TOTALTIMEOUT 6000
	ONTIMEOUT $pxe_timeout_profile
	
	LABEL local
	        MENU LABEL (local)
	        MENU DEFAULT
	        LOCALBOOT -1
	
	$pxe_menu_items
	
	MENU end

> cobbler sync

## 构建私有的yum仓库
### 1. 添加 repo
> cobbler repo add --name=openstack-mitaka --mirror=http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-mitaka/ --arch=x86_64 --breed=yum

**注意：**

时间稍长
### 2. 同步 repo
> cobbler reposync

将阿里云镜像里的文件都下载下来，然后自动创建repo文件。
### 添加repo到对应的profile中
> cobbler profile edit --name=CentOS-7-x86_64 --repo=http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-mitaka/
  
### 修改kickstart文件
 添加。（些到%post %end中间）                
> vim /var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg

	。。。
	%post
	systemctl disable postfix.service
	$yum_config_stanza    #添加这条代码
	%end
> cobbler reposync

	task started: 2016-05-29_001545_reposync
	task started (id=Reposync, time=Sun May 29 00:15:45 2016)
	hello, reposync
### 5.添加定时任务，定期同步repo

## 搭建环境指定ip：

	cobbler system list

	cobbler system add --name=linux-node3.oldboyedu.com --mac=00:50:56:3B:41:00 --profile=CentOS-7-x86_64 \
	--ip-address=192.168.56.13 --subnet=255.255.255.0 --gateway=192.168.56.2 --interface=eth0 \
	--static=1 --hostname=linux-node2.oldboyedu.com --name-servers="192.168.56.2" \
	--kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg

开启对应的虚拟机，虚拟机就直接运行装系统了。

## cobbler api

vim cobbler_list.py

	#!/usr/bin/python
	import xmlrpclib
	server = xmlrpclib.Server("http://192.168.56.11/cobbler_api")
	print server.get_distros()
	print server.get_profiles()
	print server.get_systems()
	print server.get_images()
	print server.get_repos()

 python  cobbler_list.py 

	[{'comment': '', 'kernel': '/var/www/cobbler/ks_mirror/CentOS-7-x86_64/images/pxeboot/vmlinuz', 'uid': 'MTQ2NDQ0NTUwOS44ODY3MTMzODIuNjkwMDQ', 'kernel_options_post': {}, 'redh
	at_management_key': '<<inherit>>', 'kernel_options': {}, 'redhat_management_server': '<<inherit>>', 'initrd': '/var/www/cobbler/ks_mirror/CentOS-7-x86_64/images/pxeboot/initrd.img', 'mtime': 1464445514.341205, 'template_files': {}, 'ks_meta': {'tree': 'http://@@http_server@@/cblr/links/CentOS-7-x86_64'}, 'boot_files': {}, 'breed': 'redhat', 'os_version': 'rhel7', 'mgmt_classes': [], 'fetchable_files': {}, 'tree_build_time': 0, 'arch': 'x86_64', 'name': 'CentOS-7-x86_64', 'owners': ['admin'], 'ctime': 1464445509.877032, 'source_repos': [['http://@@http_server@@/cobbler/ks_mirror/config/CentOS-7-x86_64-0.repo', 'http://@@http_server@@/cobbler/ks_mirror/CentOS-7-x86_64']], 'depth': 0}, {'comment': '', 'kernel': '/var/www/cobbler/ks_mirror/CentOS-6-x86_64/images/pxeboot/vmlinuz', 'uid': 'MTQ2NDQ0NjA2Ni40MjI1NjA4MS44MTEzNzM', 'kernel_options_post': {}, 'redhat_management_key': '<<inherit>>', 'kernel_options': {}, 'redhat_management_server': '<<inherit>>', 'initrd': '/var/www/cobbler/ks_mirror/CentOS-6-x86_64/images/pxeboot/initrd.img', 'mtime': 1464446066.905749, 'template_files': {}, 'ks_meta': {'tree': 'http://@@http_server@@/cblr/links/CentOS-6-x86_64'}, 'boot_files': {}, 'breed': 'redhat', 'os_versio。。。

## cobbler-API-2

vim cobbler-api.py

	#!/usr/bin/env python 
	# -*- coding: utf-8 -*-
	import xmlrpclib 
	
	class CobblerAPI(object):
	    def __init__(self,url,user,password):
	        self.cobbler_user= user
	        self.cobbler_pass = password
	        self.cobbler_url = url
	    
	    def add_system(self,hostname,ip_add,mac_add,profile):
	        '''
	        Add Cobbler System Infomation
	        '''
	        ret = {
	            "result": True,
	            "comment": [],
	        }
	        #get token
	        remote = xmlrpclib.Server(self.cobbler_url) 
	        token = remote.login(self.cobbler_user,self.cobbler_pass) 
			
			#add system
	        system_id = remote.new_system(token) 
	        remote.modify_system(system_id,"name",hostname,token) 
	        remote.modify_system(system_id,"hostname",hostname,token) 
	        remote.modify_system(system_id,'modify_interface', { 
	            "macaddress-eth0" : mac_add, 
	            "ipaddress-eth0" : ip_add, 
	            "dnsname-eth0" : hostname, 
	        }, token) 
	        remote.modify_system(system_id,"profile",profile,token) 
	        remote.save_system(system_id, token) 
	        try:
	            remote.sync(token)
	        except Exception as e:
	            ret['result'] = False
	            ret['comment'].append(str(e))
	        return ret
	
	def main():
	    cobbler = CobblerAPI("http://192.168.56.11/cobbler_api","cobbler","cobbler")
	    ret = cobbler.add_system(hostname='cobbler-api-test',ip_add='192.168.56.111',mac_add='00:50:56:25:C2:AA',profile='CentOS-7-x86_64')
	    print ret
	
	if __name__ == '__main__':
    main()

在创建虚拟机的时候，需要生成mac地址
然后修改py脚本，在main() 代码块中，修改cobbler服务器ip地址和用户名，密码 这个密码就是上面openssl创建的密码

