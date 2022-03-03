---
title: cloud-init 快速开始
date: 2022-03-03 05:57:01
categories: 云计算
---

问题描述
----

以前，在操作系统的安装过程中：首先，我们需要准备操作系统镜像（通常是 ISO 镜像），并制作启动盘以从中启动。然后，在系统安装的过程中，用户与安装界面交互，填写必要的操作系统安装参数。最后，操作系统安装工具根据用户填写的参数完成系统安装与配置。

后来，出现无人值守安装：即预先将在以往安装过程中需要填写的操作参数写入配置文件。然后，向操作系统安装工具传递该配置文件的路径或读取方法。最后，操作系统安装工具在启动时，自动读取配置文件，以自动完成操作系统安装。

现在，我们发现：对于已安装完成的操作系统，能够通过克隆复制，直接得到可运行的操作系统。例如：对于安装到硬盘的操作系统，我们能够使用硬盘工具，直接克隆到新硬盘；对于安装到虚拟机的操作系统，我们能够直接复制虚拟机磁盘文件。所有没有必要再进行复杂、冗长的操作系统安装过程。

解决方案
----

cloud-init，正是这样一个工具，通过 cloud-init 装置，我们能够彻底省略安装过程，利用已完成安装的虚拟磁盘文件，更快速的完成操作系统部署过程。

cloud-init 是行业标准的跨平台云实例初始化的多分发方法，所有主要的公共云提供商、私有云基础设施的供应系统和裸机安装都支持它。多数云环境都是通过这种方式来完成操作系统的快速安装部署。

### 原理简述

为了使用 cloud-init 装置，我们需要准备 镜像 与 配置 两样东西：
* 1）镜像：是已经安装完成的操作系统镜像文件（比如虚拟机磁盘 VMDK 文件），而不再需要操作系统安装镜像；
* 2）配置：我们再编写描述操作系统信息的配置文件，该配置文件包含 主机名、帐号密码、网络配置、磁盘配置 等等配置信息；
* 3）当镜像启动时，在镜像内置的 cloud-init 进程随之启动，通过读取该配置文件，在启动过程中直接完成操作系统的配置；

概念术语
----

### Cloud Image

我们前面提到的“镜像”，在 cloud-init 中，被称为“Cloud Image”：
* 1）它是个已完成操作系统安装的磁盘文件；
* 2）每个虚拟机的磁盘都是 Cloud Image 的克隆；

### Datasource

我们前面提到的“配置”，在 cloud-init 中，被称为“Datasource”：
* 1）为每个虚拟机提供各种配置，比如 Hostname、Network Configuration、Password 等等；
* 2）该配置由用户负责编写；

Datasource 主要提供两个配置文件：user-data；meta-data；

获取 Datasource 的常用方法有两种：
* 1）HTTP：通过 HTTP 获取配置文件地址，而地址已预先硬编码到 Cloud Image 中（很多云厂商使用这种方式）；
* 2）NoCloud：将配置文件打包到 ISO 镜像，并挂载到虚拟机中；

### cloud-init

被集成到 Cloud Image 中，当镜像启动时，将运行 cloud-init 进程；

当运行 cloud-init 服务时，主要完成两项工作：
* 1）探测并读取 Datasource 配置；
* 2）将这些配置应用到当前的虚拟机实例中；

安装使用
----

### 第一步、创建镜像

下载 Cloud Image 文件：<https://cloud-images.ubuntu.com/>

	# .img 文件为 QCOW2 格式
	wget http://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
	
	# 创建磁盘文件 
	qemu-img convert -f qcow2 -O raw focal-server-cloudimg-amd64.img focal-server-cloudimg-amd64.raw
	qemu-img resize focal-server-cloudimg-amd64.raw 100G
	
	# 创建磁盘文件：此方式使用 QCOW2 的写时复制特性，hal9000.img 引用 focal-server-cloudimg-amd64.img 文件，空间占用小
	qemu-img create -b focal-server-cloudimg-amd64.img -f qcow2 -F qcow2 hal9000.img 10G


### 第二步、创建配置

	apt-get install whois cloud-image-utils
	
	mkpasswd -m sha512crypt 123456 -S "12345342"
	
	cat > user-data <<EOF
	#cloud-config
	
	hostname: focal-server
	manage_etc_hosts: localhost
	
	users:
	  - name: root
	    lock_passwd: false
	    hashed_passwd: '<the output of mkpassword...>'
	    ssh_authorized_keys:
	      - ssh-rsa AAAAB3NzaC1y...
	
	# SSH
	ssh_pwauth: True
	disable_root: false
	EOF
	
	cloud-localds user-data.iso user-data


### 第三步、创建虚拟机

然后，在虚拟机中同时挂载 focal-server-cloudimg-amd64.raw 与 user-data.iso 文件；

	virt-install              \
	    --vcpus=4             \
	    --ram=8192            \
	    --import              \
	    --os-variant=ubuntu20.04      \
	    --network network=cluster-network,model=virtio                \
	    --graphics vnc,listen=0.0.0.0 \
	    --noautoconsole               \
	    --disk path=/srv/isos/user-data.iso,device=cdrom              \
	    --disk path=/srv/image/focal-server-cloudimg-amd64.raw,format=qcow2   \
	    --name=focal-server


当开机启动完成后，便可通过 root/123456 进行登录；

参考文献
----

[cloud-init Documentation — cloud-init 22.1 documentation](https://cloudinit.readthedocs.io/en/latest/ )

[12.04 - Default username/password for Ubuntu Cloud image? - Ask Ubuntu](https://askubuntu.com/questions/451673/default-username-password-for-ubuntu-cloud-image )

[Networking Config Version 2 — cloud-init 22.1 documentation](https://cloudinit.readthedocs.io/en/latest/topics/network-config-format-v2.html# )

[linux - How do I set a password on an Ubuntu cloud image? - Server Fault](https://serverfault.com/questions/920117/how-do-i-set-a-password-on-an-ubuntu-cloud-image )

[How to set root password with cloud-config? · Issue #659 · vmware/photon · GitHub](https://github.com/vmware/photon/issues/659 )

[12.04 - Default username/password for Ubuntu Cloud image? - Ask Ubuntu](https://askubuntu.com/questions/451673/default-username-password-for-ubuntu-cloud-image )

[Cloud-init To Enable Cloud Image Root Login – TCC Consulting Limited](https://www.tcc-consulting.com.hk/110/cloud-technology/cloud-init-to-enable-cloud-image-root-login/ )

[mcwhirter.com.au/craige/blog/2015/Enable Root Login Over SSH With Cloud-Init on OpenStack](https://mcwhirter.com.au/craige/blog/2015/Enable_Root_Login_Over_SSH_With_Cloud-Init_On_OpenStack/ )

[Creating a VM using Libvirt, Cloud Image and Cloud-Init | Sumit’s Dreams of Electric Sheeps](https://sumit-ghosh.com/articles/create-vm-using-libvirt-cloud-images-cloud-init/ )