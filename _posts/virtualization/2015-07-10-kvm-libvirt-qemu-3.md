---
layout: article
title: "kvm libvirt qemu实践系列(三)-虚拟机xml文件使用"
categories: virtualization
---

>- 本篇文章介绍通过xml文件来管理虚拟机
- 本系列之前文章  
  [kvm libvirt qemu实践系列(一)](http://www.isjian.com/virtualization/kvm-libvirt-qemu-1/)  
  [kvm libvirt qemu实践系列(二)](http://www.isjian.com/virtualization/kvm-libvirt-qemu-2/)
- 测试环境为centos6.5，kvm程序来自rpm包  

### 关于backing file

kvm虚拟机创建的磁盘默认使用qcow2格式，qcow2格式一个特性是copy-on-write，即写入时复制
现在有一台虚拟机openstack1-1，用的磁盘是openstack1-1.disk，查看它磁盘的详细信息

``` shell
    [root@controller1 openstack]# qemu-img info openstack1-1.disk 
    image: openstack1-1.disk
    file format: qcow2
    virtual size: 40G (42949672960 bytes)
    disk size: 10G
    cluster_size: 65536
    backing file: /data/images/centos65x64.qcow2
```
现在，可以很方便创建多个虚拟机，centos65x64.qcow2为磁盘openstack1-1.disk的后端镜像(`backing file`)，这个centos65x64.qcow2磁盘是制作好的镜像，包含完整的系统和定制的的软件包，新建的磁盘openstack1-1.disk可以使用后端镜像的文件，用户操作虚拟机时，读文件是从后端镜像读，写新文件是写入到新磁盘openstack1-1.disk，关于具体介绍，参考[kvm libvirt qemu实践系列(五)](http://www.isjian.com/virtualization/kvm-libvirt-qemu-5/)中关于`backing file`的介绍

``` shell
qemu-img create -b /data/images/centos65x64.qcow2 -f qcow2 openstack1-1.disk 20G
qemu-img create -b /data/images/centos65x64.qcow2 -f qcow2 openstack1-2.disk 20G
qemu-img create -b /data/images/centos65x64.qcow2 -f qcow2 openstack1-3.disk 20G
qemu-img create -b /data/images/centos65x64.qcow2 -f qcow2 openstack1-4.disk 20G
```
这四台虚拟机都使用同一个后端镜像，但是四台虚拟机互不影响，后端镜像centos65x64.qcow2是只读的，四台虚拟机都从后端镜像读数据，但是写入新数据是写入到各自的新磁盘

### 虚拟机xml文件

下面步骤来具体解释虚拟机xml文件是如何控制虚拟机运行的，假设你已经有了一个backin file`centos65x64.qcow2`，可以参考[这一篇文章](http://www.isjian.com/virtualization/kvm-libvirt-qemu-2/)来运行一个虚拟机

##### 创建虚拟磁盘

``` shell
qemu-img create -b /data/images/centos65x64.qcow2 -f qcow2 kvm-1.disk 40G
```

##### 一个具体的xml文件

下面是一个标准的xml文件，其虚拟了虚拟机系统运行所必须的硬件环境，注意修改`<disk>...</disk>`，此标签指定了虚拟机所使用的虚拟磁盘

``` html
[root@controller1 openstack]$ cat kvm-1.xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>kvm-1</name>
  <memory unit='KiB'>8388608</memory>
  <currentMemory unit='KiB'>8388608</currentMemory>
  <vcpu placement='static'>4</vcpu>
  <os>
    <type arch='x86_64' machine='rhel6.5.0'>hvm</type>
    <boot dev='hd'/>
    <boot dev='cdrom'/>
    <bootmenu enable='yes'/>
    <bios useserial='yes' rebootTimeout='0'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <pae/>
  </features>
  <cpu mode='host-model'>
    <model fallback='allow'/>
  </cpu>
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none'/>
      <source file='/data/vhosts/jython/openstack/kvm-1.disk'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <controller type='ide' index='0'>
    </controller>
    <controller type='virtio-serial' index='0'>
    </controller>
    <controller type='usb' index='0'>
    </controller>
    <interface type='bridge'>
      <source bridge='br-ex'/>
      <model type='virtio'/>
    </interface>
    <interface type='bridge'>
      <source bridge='br-ex'/>
      <model type='virtio'/>
    </interface>
    <console type='pty'>
    </console>
    <input type='mouse' bus='ps2'/>
    <graphics type='vnc' autoport='yes' listen='0.0.0.0'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <video>
      <model type='cirrus' heads='1'/>
    </video>
    <memballoon model='virtio'>
    </memballoon>
  </devices>
  <qemu:commandline>
    <qemu:env name='SPICE_DEBUG_ALLOW_MC' value='1'/>
  </qemu:commandline>
</domain>
```

##### 启动虚拟机

``` shell
virsh define kvm-1.xml
#kvm-1.xml中包括了启动虚拟机的所有参数，比如：指定了磁盘路径，虚拟机内存，cpu，网络连接方式，vnc端口等
#define之后，就能使用virsh list命令查看虚拟机，virsh list只能看到正在运行的虚拟机
[root@controller1 openstack]$ virsh list
 Id    Name                           State
----------------------------------------------------
 1     kvm-1                          running
 
#启动虚拟机
 virsh start kvm-1
```

##### 更改虚拟机配置

``` shell
virsh edit kvm-1
#修改的是虚拟机xml文件，修改之后，需要关闭，再启动虚拟机才会生效，注意：重启不能生效
```

##### virsh其它命令
下面是管理虚拟机常用命令，你可以使用`virsh help`查看所有virsh提供的命令
<font color="red">virsh管理的对象是`domain`，即用`virsh list`看到的虚拟机`name`</font>

``` shell
#关机
virsh shutdown kvm-1
 
#重启
virsh reboot kvm-1

#查看虚拟机信息
virsh dominfo kvm-1

#查看虚拟机使用磁盘
virsh domblklist kvm-1
 
#查看虚拟机网卡信息
virsh domiflist kvm-1
```

### xml文件解析

- 如果虚拟机已经存在，要修改虚拟配置，首先使用vish list查看要修改哪台虚拟机，然后使用virsh edit name 编译虚拟机xml定义文件，编辑完成退出时，会自动检测xml文件，如果xml文件有语法错误，则系统会自动把此模块移除，因此退出之后，需要再次virsh edit name 进去查看添加修改的模块是否真的存在

- xml文件中有许多配置项不需要手动添加，`edit`退出之后，系统会自动添加这些，比如xml文件中有很多 <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>，编辑时，直接删掉这行，退出系统就会自动生成这行，其它会自动添加的有UUID，虚拟机mac地址等

- xml配置项解析

``` shell
#虚拟化类型为kvm(type='kvm')，可选的还有qemu
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
#虚拟机名字 openstack1-1 
 <name>openstack1-1</name>
#虚拟机预分配内存8388608K,这个是宿主机允许虚拟机使用的最大内存，并不是在虚拟机里用free看到的内存
  <memory unit='KiB'>8388608</memory>
#虚拟机当前定义内存(8388608)，free看到的内存，可以使用virsh setmem调整内存
  <currentMemory unit='KiB'>8388608</currentMemory>
#虚拟机cpu个数  
<vcpu placement='static'>4</vcpu>
  <os>
#模拟的系统架构x86_64,模拟机器类型rhel6.5
    <type arch='x86_64' machine='rhel6.5.0'>hvm</type>
#虚拟机开机引导项，hd：硬盘，cdrom：光盘，即先硬盘，后光盘
    <boot dev='hd'/>
    <boot dev='cdrom'/>
    <bootmenu enable='yes'/>
    <bios useserial='yes' rebootTimeout='0'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <pae/>
  </features>
#虚拟机cpu模拟类型，host-model，使用宿主机cpu的所有可使用特性
  <cpu mode='host-model'>
    <model fallback='allow'/>
  </cpu>
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <devices>
#运行虚拟机的程序，qemu-kvm，可以在宿主机使用ps -ef | grep qemu-kvm 看到
    <emulator>/usr/libexec/qemu-kvm</emulator>
#定义虚拟机磁盘
    <disk type='file' device='disk'>
#虚拟机磁盘为qcow2格式，如果你创建或使用的磁盘是raw格式，需要修改为raw
      <driver name='qemu' type='qcow2' cache='none'/>
#磁盘路径
      <source file='/data/vhosts/jython/openstack/openstack1-1.disk'/>
#第一块为vda，第二块就为vdb，不能重复，重复虚拟机启动报错
      <target dev='vda' bus='virtio'/>
    </disk>
    <controller type='ide' index='0'>
    </controller>
    <controller type='virtio-serial' index='0'>
    </controller>
    <controller type='usb' index='0'>
    </controller>
#虚拟机网络为桥接模式bridge，桥接网桥为br-ex，要确保网桥br-ex存在，并且能使用
    <interface type='bridge'>
      <source bridge='br-ex'/>
      <model type='virtio'/>
    </interface>
#第二张网卡，如果需要多块网卡，就复制多次
    <interface type='bridge'>
      <source bridge='br-ex'/>
      <model type='virtio'/>
    </interface>
    <console type='pty'>
    </console>
    <input type='mouse' bus='ps2'/>
#使用vnc协议，autoport='yes':自动分配端口，从5900开始
    <graphics type='vnc' autoport='yes' listen='0.0.0.0'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <video>
      <model type='cirrus' heads='1'/>
    </video>
#气球内存技术，kvm特性之一
    <memballoon model='virtio'>
    </memballoon>
  </devices>
#下面三行是为了实现多vnc客户端连接，即多个用户使用vnc客户端连接到同一台虚拟机，操作实时同步
  <qemu:commandline>
    <qemu:env name='SPICE_DEBUG_ALLOW_MC' value='1'/>
  </qemu:commandline>
</domain>
```

### 常用xml文件模块

- 使用CDROM

``` html
<disk type='file' device='cdrom'>
    <driver name='qemu' type='raw'/>
    <source file='/data/iso/XenServer-6.5.0-xenserver.org-install-cd.iso'/>
    <target dev='hdc' bus='ide'/>
    <readonly/>
</disk> 
```
    
- 配置cpu穿透

``` html
<cpu mode='host-passthrough'>
  <topology sockets='1' cores='2' threads='2'/>
</cpu>
<cpu mode='host-model'>
  <model fallback='allow'/>
</cpu>
```
    
- 配置cpu插槽核数

``` html
<cpu>
  <topology sockets='1' cores='2' threads='4'/>
</cpu>
```

- 使用spice连接

``` html
 <channel type='spicevmc'>
    <target type='virtio' name='com.redhat.spice.0'/>
  </channel>
 <graphics type='spice' autoport='yes' listen='0.0.0.0' passwd='zijian'>
    <listen type='address' address='0.0.0.0'/>
  </graphics>
```

- 设置最大虚拟内存,vcpu

``` html
<memory unit='KiB'>4194304</memory>
<currentMemory unit='KiB'>4194304</currentMemory>
```

- 设置vcpu个数,亲和性

``` html
<vcpu placement='static' cpuset='0-9,20-29' current='4'>8</vcpu>
```

- 设置虚拟机磁盘

``` html
 <disk type='file' device='disk'>
    <driver name='qemu' type='qcow2' cache='writethrough' io='threads'/>
    <source file='/data/vhosts/openstack/controller1.qcow2'/>
    <target dev='vda' bus='virtio'/>
  </disk>
```

- 配置虚拟机启动日志输出到文件

``` html
  <serial type='file'>
    <source path='/data/tmp/console.log'/>
    <target port='0'/>
  </serial>
  <serial type='pty'>
    <target port='1'/>
  </serial>
  <console type='file'>
    <source path='/data/tmp/console.log'/>
    <target type='serial' port='0'/>
  </console>
```

- 配置virtio驱动（pci总线）

``` html
  <disk type='file' device='disk'>
    <driver name='qemu' type='raw' cache='none'/>
    <source file='/root/server2012/ws2012.disk'/>
    <target dev='vda' bus='virtio'/>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
  </disk>
  <controller type='virtio-serial' index='0'>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
  </controller>
```

- 配置ide驱动

``` html
 <disk type='file' device='disk'>
    <driver name='qemu' type='raw' cache='none'/>
    <source file='/root/server2012/ws2012.disk'/>
    <target dev='vda' bus='ide'/>
     <address type='drive' controller='0' bus='0' target='0' unit='0'/>
  </disk>
  <controller type='ide' index='0'>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
  </controller>
```

- 配置usb输入设备（vnc连接鼠标同步）

``` html
  <input type='tablet' bus='usb'/>
  <input type='mouse' bus='ps2'/>
```

- 配置开机启动项

``` html
<os>
  <type arch='x86_64' machine='rhel6.6.0'>hvm</type>
  <boot dev='hd'/>
  <boot dev='cdrom'/>
  <bootmenu enable='yes'/>
</os>
```

### 创建虚拟机快照

kvm创建快照有多种方法，但是最简单的是使用qemu-img命令，这种方式创建的快照内置于虚拟磁盘内，是同一个文件，因此如果创建过多快照，虚拟磁盘会变得很大，迁移不方便
关于虚拟机创建快照详细信息，请参考本博客其它博文  

创建快照，恢复快照前最好关闭虚拟机

``` shell
#如果要为test-1这台创建快照，先查看下磁盘信息
qemu-img info test-1.disk 
image: test-1.disk
file format: qcow2
virtual size: 60G (64424509440 bytes)
disk size: 6.0G
cluster_size: 65536
backing file: /data/images/centos65x64.qcow2

#创建快照
qemu-img snapshot -c test-1-snap1 test-1.disk 
#上面命令中，-c表示创建快照，test-1-snap1是快照名称，test-1.disk是虚拟磁盘，即为磁盘test-1.disk创建一个名称为test-1-snap1的快照
 
#再次查看快照信息
qemu-img info test-1.disk 
image: test-1.disk
file format: qcow2
virtual size: 60G (64424509440 bytes)
disk size: 6.0G
cluster_size: 65536
backing file: /data/images/centos65x64.qcow2
Snapshot list:
ID        TAG                 VM SIZE                DATE       VM CLOCK
1         test-1-snap1              0 2015-07-10 11:50:37   00:00:00.000
 
#恢复虚拟机到快照test-1-snap1状态
qemu-img snapshot -a test-1-snap1 test-1.disk
 
#查看磁盘test-1.disk的所有快照
qemu-img snapshot -l test-1.disk 
Snapshot list:
ID        TAG                 VM SIZE                DATE       VM CLOCK
1         test-1-snap1              0 2015-07-10 11:50:37   00:00:00.000
#删除test-1.disk虚拟磁盘的test-1-snap1这个快照
qemu-img snapshot -d test-1-snap1 test-1.disk
```