---
title: "spice在kvm虚拟机中的应用(一)"
author: opengers
layout: post
permalink: /virtualization/spice-kvm-usbredir-qxl-1/
categories: virtualization
tags:
  - virtualization
  - kvm
  - usbredir
  - qxl
format: quote
---

> <small>本文测试服务器为centos6.4
本系类其它文章    
[spice在kvm虚拟机中的应用(二)](http://www.isjian.com/virtualization/spice-kvm-usbredir-qxl-2/)
[spice在kvm虚拟化中的应用(三)](http://www.isjian.com/virtualization/spice-kvm-usbredir-qxl-3/)</small>

### spice介绍

##### spice简介
spice是由Qumranet开发的开源网络协议，2008年红帽收购了Qumranet获得了这个协议。SPICE是红帽在虚拟化领域除了KVM的又一“新兴技术”，它提供与虚拟桌面设备的远程交互实现.
目前,spice主要目标是为qemu虚拟机提供高质量的远程桌面访问,它致力于克服传统虚拟桌面的一些弊端,并且强调用户体验

![spice1-1](/images/virtualization/spice-kvm-usbredir-qxl-1/spice1-1.png)

spice包含有3个组件  
- SPICE Driver：SPICE驱动器 存在于每个虚拟桌面内的组件  
- SPICE server：SPICE服务器 存在于红帽企业虚拟化Hypervisor内的组件
- SPICE Client： SPICE客户端 存在于终端设备上的组件，可以是瘦客户机或专用的PC，用于接入每个虚拟桌面。
这三个组件协作运行，确定处理图形的最高效位置，以能够最大程度改善用户体验并降低系统负荷。如果客户机足够强大，SPICE向客户机发送图形命令，并在客户机中对图形进行处理，显著减轻服务器的负荷。另一方面，如果客户机不够强大，SPICE在主机处理图形，从CPU的角度讲，图形处理并不需要太多费用  
<small>以上简介参考http://os.51cto.com/art/201201/311464.htm</small>

##### spice架构

![spice2](/images/virtualization/spice-kvm-usbredir-qxl-1/spice2.png)

Spice agent运行在客户机（虚拟机）操作系统中。Spice server和Spice client利用spice agent来执行一些需要在虚拟机里执行的任务，如配置分辨率，另外还有通过剪贴板来拷贝文件等。从上图可以看出，Spice client与server与Spice Agent的通信需要借助一些其他的软件模块，如在客户机里面，Spice Agent需要通过VDIPort Driver与主机上 QEMU的VDIPort Device进行交互，他们的交互通过一种叫做输入/输出的环进行。Spice Client和Server产生的消息被写入到设备的输出环中，由VDI Port Driver读取；而Spice Agent发出的消息则通过VDI Port Driver先写入到VDI Port Device输入环中,被QEMU读入到Spice server的缓冲区中，然后再根据消息决定由Spice Server直接处理，还是被发往Spice Client中
<small>以上参考http://blog.csdn.net/hbsong75/article/details/9465683</small>

##### spice的不足
- spice目标是提供一个高性能,高用户体验的远程桌面连接,就像本地桌面一样展现给用户. 其目前实现的功能有usb重定向,音视频传输,剪贴板,鼠标同步,2D图形支持,任意调整分辨率(qxl驱动)等  

- spice目前不支持虚拟机中的3D效果,对于windows7系统虚拟机,其aero桌面特效也无法启用,因为spice使用远程连接,所以其高度依赖网络,如果网络环境不好,使用起来将会是一间很痛苦的事情

### 虚拟机设置spice支持

##### 测试服务器信息

``` shell
#服务器系统环境
RHEL6 IP:192.168.11.166

#需要服务器硬件支持虚拟化技术(Intel VT-d,AMD-V)。可以使用如下命令检查,有输出信息表示支持
egrep "(vmx|svm)" --color /proc/cpuinfo

#检查 KVM 模块是否成功安装。如果有输出结果，那么 KVM 模块已成功安装
lsmod | grep kvm
```

##### 安装KVM软件
``` shell
#安装kvm/qemu工具,以及virt-manager,libvirtd
yum install qemu-kvm qemu-kvm-tools virt-manager libvirt libvirt-devel libvirt-client virt-manager virt-viewer
```

##### 服务器上安装spice-server

``` shell
yum -y install spice-protocol spice-server xorg-x11-drv-qxl spice-gli
#需要安装以上软件，才能启用spice
```

##### 客户端安装spice client
- centos客户端  

``` shell
    yum -y install virt-viewer
```

- windows7客户端  
下载链接: [virt-viewer](http://virt-manager.org/download/sources/virt-viewer/virt-viewer-x64-1.0.msi)

##### 新建centos6.4虚拟机
新建虚拟机步骤参考以前博文，有具体介绍，本文假设你已经能够成功启动虚拟机，注意下面配置  

``` html
<graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'>
    <listen type='address' address='0.0.0.0'/>
</graphics>
```
默认情况下,kvm使用vnc建立远程连接,监听地址为0.0.0.0(`listen`),其端口为自动分配(`port=-1 autoport='yes'`)    
`port=-1`表示端口自动分配,分配范围`5900+N`,即第一个虚拟机会是5900端口，第二个虚拟机会是5900+1，以此类推


**关于virt-manager工具**  
virt-manager是一个图形化的虚拟机管理工具,它可以方便地创建虚拟机,修改虚拟机配置,添加新设备等. 但是由于其是图形界面管理,所以效率不是很高,而且对网络也有要求

##### 启动虚拟机

``` shell
#加入virsh管理
virsh define cdesk1.xml
#启动虚拟机
virsh start cdesk1

#查看其端口，默认为vnc协议端口
netstat -ntpl | grep qemu-kvm
```

##### 使用vnc连接
当kvm使用vnc协议时候，windows下可以使用vnc客户端进行连接，如下

![image2014-12-25 19-29-4](/images/virtualization/spice-kvm-usbredir-qxl-1/image2014-12-25-19-29-4.png)

### 配置spice远程连接

##### 修改xml文件

``` html
<!-- 首先virsh命令关闭虚拟机,使用`virsh edit domain`编辑 -->
    <!--  添加 -->  
<channel type='spicevmc'>
    <target type='virtio' name='com.redhat.spice.0'/>
</channel>
<!--  修改 -->
<graphics type='spice' port='5902' tlsPort='5903' autoport='yes' listen='0.0.0.0'>
</graphics>
<video>
    <model type='qxl' heads='1'/>
    <alias name='video0'/>
</video>
```
`type`由`vnc`改为`spice`  
指定`port`为`5902`端口，也可以使用`-1`自动分配端口  
`tlsPort`为加密端口，spice也支持加密的连接，这会启动两个端口，`5902`是未加密端口，`5903`是加密端口，测试时可以不需要此参数    

##### 支持音频传输
spice也支持音视频，这对于windows虚拟机来说，可能是必须的配置  
**Linux虚拟机`model='ich6'`,windows虚拟机`model='ac97'`**

``` html
    <sound model='ich6'>
       <alias name='sound0'/>
     </sound>
```

##### 启动虚拟机  
``` shell
#修改xml文件，之后可以启动虚拟机
virsh start cdesk1

#验证spice端口是否启用
ps -ef |grep spice
```

##### windows下客户端

``` shell
#virt-viewer支持vnc和spice两种协议，连接地址格式如下
spice://192.168.11.66:5902  #spice协议
vnc://192.168.11.66:5902  #vnc协议
#192.168.11.66 为测试物理机地址
#5902为此虚拟机连接端口(注意此端口是物理机上的端口)
```

![image2014-12-25 20-2-0](/images/virtualization/spice-kvm-usbredir-qxl-1/image2014-12-25-20-2-0.png)

##### 使用virt-manager图形工具配置spice  
以上步骤中是修改xml文件配置spice连接的,也可以使用virt-manager图形界面操作

- 首先关闭虚拟机
- 服务器上运行virt-manager命令,打开图形界面(需要开启服务器上的X11转发)
- 如下图Display中更改Type为spice

![123](/images/virtualization/spice-kvm-usbredir-qxl-1/123.png)

- video中更改Model为qxl,修改完成之后,启动虚拟机

![2](/images/virtualization/spice-kvm-usbredir-qxl-1/2.png)

### 提高虚拟机性能(鼠标同步,共享剪贴板,音视频传输等)

我们在客户端使用spice client远程连接虚拟机,如果想要虚拟机中播放的音视频传输到本地客户端,或者在虚拟机和客户机之间共享剪贴板,则需要在虚拟机中安装相应增强工具,对于windows系统和Linux系统,需要安装的增强工具不太一样

##### windows虚拟机配置
windows虚拟机需要安装增强工具spice-guest-tools(类似vmware中的vmtool工具)
下载地址: [spice-guest-tools](http://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-0.74.exe) ，这个软件包包含了一个qxl视频卡驱动,还包括SPICE guest agent,可以实现同步剪贴板,鼠标,任意调整虚拟机分辨率等功能

##### Linux虚拟机配置
centos gnome桌面版虚拟机,需要安装以下软件包

``` shell
yum install xorg-x11-drv-qxl spice-vdagent
#设置开机启动
/etc/init.d/spice-vdagentd start
chkconfig spice-vdagentd on
```

##### 把虚拟机中的音视频传输到客户端
前面是通过在虚拟机xml文件中添加sound标签,实现虚拟机和客户机的音视频传输,也可以使用virt-manager工具

用virt-manager工具添加音频设备

![image2014-12-25 20-31-28](/images/virtualization/spice-kvm-usbredir-qxl-1/image2014-12-25-20-31-28.png)

选择Sound的Model(windows虚拟机是ac97)