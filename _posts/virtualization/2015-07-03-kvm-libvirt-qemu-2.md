---
title: "kvm libvirt qemu实践系列(二)-虚拟机管理"
author: opengers
layout: post
permalink: /virtualization/kvm-libvirt-qemu-2/
categories: virtualization
tags:
  - virtualization
  - kvm
  - libvirt
format: quote
---

>- 在上一篇文章中[kvm libvirt qemu实践系列(一)](https://opengers.github.io/virtualization/kvm-libvirt-qemu-1/)，介绍了kvm，libvirt，以及qemu关系，本篇文章中则介绍如何创建，以及管理虚拟机  
- 本文章测试服务器为centos6.6，Intel-VT虚拟化，使用ssh远程连接（已开启ssh图形转发，文中使用virt-manager工具会需要这个）
- 在下列文章中，默认使用libvirt作为虚拟机创建，管理工具，但文中也会提及如何不通过libvirt，直接使用qemu-kvm(或qemu-system-x86-64)命令创建虚拟机  

# 为了能够创建虚拟机，我们需要设置步骤：

- 宿主机安装kvm相关软件  
- 创建一个虚拟磁盘，用于安装虚拟机系统  
- 使用libvirt安装虚拟机系统  
- 设置虚拟机cpu，内存，网卡等参数  
- 使用virsh启动虚拟机  

### 本文中使用的软件版本如下

``` shell
#qemu-kvm主程序，以及qemu-img虚拟磁盘管理工具
[root@localhost ~]# rpm -qa | grep qemu
qemu-img-0.12.1.2-2.448.el6_6.4.x86_64
qemu-kvm-0.12.1.2-2.448.el6_6.4.x86_64
gpxe-roms-qemu-0.9.7-6.12.el6.noarch

#宿主机内核版本
[root@localhost ~]# uname -r
3.18.3-1.el6.elrepo.x86_64

#libvirt版本
[root@localhost ~]# rpm -qa | grep libvirt
libvirt-devel-0.10.2-46.el6_6.6.x86_64
libvirt-python-0.10.2-46.el6_6.6.x86_64
libvirt-snmp-0.0.2-4.el6.x86_64
libvirt-client-0.10.2-46.el6_6.6.x86_64
libvirt-0.10.2-46.el6_6.6.x86_64

#启动libvirtd进程
[root@localhost kvm]# service libvirtd status
libvirtd (pid 17735) is running...

#确保宿主机加载kvm相关模块
[root@localhost ~]# lsmod | grep kvm
kvm_intel 148361 61 
kvm 468666 1 kvm_intel
```

<font color="red">kvm虚拟机就是一个标准的linux进程，而运行虚拟机的命令是qemu-kvm， 如下是物理机上的一个进程，其代表一个虚拟机</font>

``` shell
[root@test ~]# ps -ef | grep qemu-kvm
qemu      2723     1  9 16:11 ?        00:24:39 /usr/libexec/qemu-kvm -name centos -S -M rhel6.6.0 -enable-kvm -m 1024 -realtime mlock=off -smp 1,sockets=1,cores=1,threads=1 -uuid 45b2d2ff-0150-0d17-2a66-5ecfa182db8f -nodefconfig -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/centos.monitor,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc -no-reboot -no-shutdown -device ich9-usb-ehci1,id=usb,bus=pci.0,addr...
```

# 创建虚拟机磁盘

### raw 和 qcow2 
首先我们需要一个虚拟磁盘，这个虚拟磁盘就相当于物理机的硬盘，它用于安装虚拟机操作系统，虚拟磁盘有多种不同的格式，常见的格式有raw,qcow2,vmdk,vdi等，raw格式是最常见的，随便创建一个文件，或者使用dd生成一个文件，都是raw格式  
qcow2 镜像格式是qemu模拟器支持的一种磁盘镜像。它也是可以用一个文件的形式来表示一块固定大小的块设备磁盘。与普通的 raw 格式的镜像相比，有以下特性

- 更小的空间占用，即使文件系统不支持空洞(holes)；

- 支持写时拷贝（COW, copy-on-write），镜像文件只反映底层磁盘的变化；

- 支持快照（snapshot），镜像文件能够包含多个快照的历史；

- 可选择基于 zlib 的压缩方式

- 可以选择 AES 加密

``` shell
#raw格式
[root@localhost kvm]# qemu-img info /etc/passwd
image: /etc/passwd
file format: raw
virtual size: 3.0K (3072 bytes)
disk size: 4.0K

#qcow2格式
[root@localhost kvm]# qemu-img info centos65x64-2.6kernel.qcow2 
image: centos65x64-2.6kernel.qcow2
file format: qcow2
virtual size: 20G (21474836480 bytes)
disk size: 826M
cluster_size: 65536
```

关于磁盘格式的详细说明，请参考文章：http://www.cnblogs.com/feisky/archive/2012/07/03/2575167.html  
关于qcow2的完整介绍，请参考文章：https://people.gnome.org/~markmc/qcow-image-format.html  
关于写入时复制技术(copy-on-write)，请参考：http://www.cnblogs.com/chenglei/archive/2009/08/06/1540175.html  
<font color="red">qcow2格式特点，以及copy-on-write技术是理解kvm虚拟机创建，以及虚拟机快照链的关键，所以请尽量理解它</font>

### 创建qcow2格式磁盘

下面我们使用qemu-img工具创建一个qcow2格式的磁盘

``` shell
#查看qemu-img帮助
[root@localhost kvm]# qemu-img --help
#-f:指定磁盘格式
#[size]：创建的虚拟磁盘大小，就是设置这台虚拟机可使用的磁盘大小为20G，但是宿主机并不马上为这个虚拟磁盘分20G空间出来，它是随着虚拟机所实际使用磁盘大小增长的，最大增长到20G(qcow2磁盘特性)

#创建一个qcow2格式，磁盘名为kvm-1.disk，大小为20G的虚拟磁盘
[root@localhost kvm]# qemu-img create -f qcow2 kvm-1.disk 20G
Formatting 'kvm-1.disk', fmt=qcow2 size=21474836480 encryption=off cluster_size=65536

#查看磁盘大小，只有193k
[root@localhost kvm]# ls -lh kvm-1.disk 
-rw-r--r-- 1 root root 193K Jul 8 00:07 kvm-1.disk
```

# 安装虚拟机系统

安装系统，有多种方法，但是基本的原理是一样的，一个虚拟磁盘，还需要一个ISO安装文件，然后设置虚拟机从光盘启动，安装系统到虚拟磁盘上  
 
kvm有很多有用的管理工具，下面是常用工具：

- virsh 虚拟机管理的主要工具，来自libvirt

- qemu-img 一个操作虚拟磁盘的工具
    
- guestfish 一个虚拟机磁盘管理工具，有许多有用的命令，命令都以`virt-`开头，比如`virt-cat`
    
- virt-manager 图形化的虚拟机管理软件
    
- virt-viewer 虚拟机终端连接工具（需要ssh开启图形转发）

``` shell
yum install guestfish virt-install virt-viewer
#guestfish 是一套虚拟机磁盘管理工具
```

### 使用virt-install安装  
virt-install参数说明

``` shell
virt-install -n centos -r 1024 -c CentOS-6.6-x86_64-minimal.iso --disk path=kvm-1.disk,device=disk,bus=virtio,size=5,format=qcow2 --vnc --vncport=5907 --vnclisten=0.0.0.0 -v --network bridge=virbr0,model=virtio
#-n: 要创建的虚拟机名字为centos
#-r：为虚拟机分配1024M内存
#-c：安装系统使用的ISO光盘镜像
#--disk path：虚拟磁盘kvm-1.disk（之前使用qemu-img创建的）
#format：虚拟磁盘格式为qcow2
#--vnc --vncport:vnc端口为5907
-#-vnclisten：监听本机所有地址的5907端口
#--network bridge：虚拟机使用桥接网络，桥接网卡为virbr0（启动libvirtd进程后自动创建这个网桥）
```

要安装虚拟机系统，还需要一个安装光盘，从光盘启动安装系统，只不过这个系统是安装在一个虚拟磁盘上，以下是安装步骤  

``` shell
[root@test kvm]# ls -lh
total 384M
-rw-r--r-- 1 qemu qemu 383M Jul  8 15:58 CentOS-6.6-x86_64-minimal.iso
-rw-r--r-- 1 qemu qemu 193K Jul  8 15:35 kvm-1.disk

#virt-install
[root@test kvm]# virt-install -n centos -r 1024 -c CentOS-6.6-x86_64-minimal.iso --disk path=kvm-1.disk,device=disk,bus=virtio,size=5,format=qcow2 --vnc --vncport=5907 --vnclisten=0.0.0.0 -v --network bridge=virbr0,model=virtio

Starting install...
Creating domain...                                                                    

#出现上面的Creating domain说明虚拟机启动成功，后面的警告信息是因为系统没有安装virt-viewer这个工具
```

virt-install 在宿主机上开启了590X端口，此端口映射到虚拟机console上，因此连接宿主机上的这个端口，就可以访问虚拟机，继续后续安装步骤，vnc客户端工具有很多，比如vncviewer，tigervnc，结果如下

![qemu-2-1](/images/virtualization/kvm-libvirt-qemu-2/11.png)

vnc连接之后，就是ISO安装系统，本文不会对具体的安装步骤做过多讲解


### 使用qemu-kvm命令行安装  
我们可以不通过libvirt提供的工具`virt-install`安装，而是直接使用qemu-kvm工具安装,查看qemu-kvm帮助文档

``` shell
/usr/libexec/qemu-kvm --help
```

qemu-kvm程序有太多参数，但实际中我们只需要很少的参数就能够启动虚拟机，生产环境都是通过libvirt的xml文件来定义虚拟机参数的，不会直接使用qemu-kvm命令行  
下面是安装命令  

``` shell
[root@test kvm]# /usr/libexec/qemu-kvm -enable-kvm -m 512 -smp 1 -boot order=dc -hda kvm-1.disk -cdrom CentOS-6.6-x86_64-minimal.iso -vnc :7
#同样，qemu-kvm命令把虚拟机终端映射到宿主机的5907端口

#-boot order=dc ：虚拟机引导启动项顺序，floppy (a), hard disk (c), CD-ROM (d), network (n)，dc即先从光盘启动
#-enable-kvm：启用kvm支持
#-m 512：分配内存512M
#-hda：虚拟机磁盘
#-cdrom：iso镜像，设置从此iso启动可以安装系统
#-vnc :7 ：端口是从5900+n开始的，例如-vnc:n就是5900+n端口
```

通过`qemu-kvm`创建的虚拟机并没有使用libvirt，而是直接使用`qemu-kvm`命令运行虚拟机，一旦终端断开连接，或者Ctrl+C结束命令，虚拟机也就关机了


### 使用virt-manager图形界面安装  
如果你安装了virt-manager，那么可以使用virt-manager这个图形工具安装kvm，这个工具运行需要ssh图形转发支持，如下

![qemu-2-2](/images/virtualization/kvm-libvirt-qemu-2/22.png)

接下来，点击New按钮，进入新建虚拟机向导，虚拟机名字kvm-2，使用ISO光盘方式安装

![qemu-2-3](/images/virtualization/kvm-libvirt-qemu-2/33.png)

使用ISO文件CentOS-6.6-x86_64-minimal.iso，设置OS type等

![qemu-2-4](/images/virtualization/kvm-libvirt-qemu-2/44.png)

接下来分配内存，cpu，然后继续下一步，到创建磁盘界面  
有两种方式，创建一个新磁盘，或者使用之前用qemu-img建好的磁盘，点击下一步

![qemu-2-5](/images/virtualization/kvm-libvirt-qemu-2/55.png)

接下来的界面中可以设置网络，默认为NAT方式（即桥接virbr0网卡，网络设置可以以后使用virsh命令修改），新建的虚拟机网络为`192.168.122.0/24`这个子网

![qemu-2-6](/images/virtualization/kvm-libvirt-qemu-2/66.png)

这一步之后，虚拟机就配置完毕，接下来在virt-manager界面中就可以启动虚拟机，进行系统的安装  
virt-manager提供了图形化的界面操作，但其后台调用命令与第一种安装方式基本一样，默认vnc端口是从5900开始分配的，即如果建立了两台虚拟机，那么宿主机上就会有5900,5901两个监听端口  
可以在界面中看到，其中的localhost(QEMU)，意思是virt-manager连接的本地hypervisor，virt-manager也可以通过网络连接其它服务器的hypervisor，通过右键也可以管理虚拟机  

![qemu-2-7](/images/virtualization/kvm-libvirt-qemu-2/77.png)

# virsh基本命令

安装完系统之后，可以使用virsh管理虚拟机，要查看virsh所有命令：可以使用`virsh --help`

``` shell
[root@test kvm]# virsh list
 Id    Name                           State
----------------------------------------------------
 3     centos                         running

#查看虚拟机信息
[root@test softs]# virsh dominfo centos

#关机
virsh shutdown centos
#重启
virsh reboot centos
#查看虚拟机信息
virsh dominfo centos
#查看虚拟机磁盘
virsh domblklist centos
#查看虚拟网卡
virsh domiflist centos
Interface Type Source Model MAC
-------------------------------------------------------
vnet0 bridge virbr0 virtio 52:54:00:79:5b:07
#更改虚拟机配置,libvirt使用xml文件来定义虚拟机配置
virsh edit centos
```

上面命令只是很基本的vish命令，关于virsh命令的具体用法，我会在下一篇文章中介绍
[本文固定链接](https://opengers.github.io/virtualization/kvm-libvirt-qemu-2/)
