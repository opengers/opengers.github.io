---
title: "KVM虚拟化之嵌套虚拟化nested"
author: opengers
layout: post
permalink: /virtualization/kvm-nested-virtualization/
categories: virtualization
tags:
  - virtualization
  - kvm
  - nested
format: quote
---

><small>本文测试物理机为centos6.5    
物理机使用Intel-V虚拟化架构,安装qemu-kvm版本0.12</small>

## 问题由来     

我们知道,在Intel处理器上,KVM使用Intel的vmx(virtul machine eXtensions)来提高虚拟机性能, 即硬件辅助虚拟化技术, 现在如果我们需要测试一个openstack集群,又或者单纯的需要多台具备"vmx"支持的主机, 但是又没有太多物理服务器可使用, 如果我们的虚拟机能够和物理机一样支持"vmx",那么问题就解决了,而正常情况下,一台虚拟机无法使自己成为一个hypervisors并在其上再次安装虚拟机,因为这些虚拟机并不支持"vmx"    

嵌套式虚拟nested是一个可通过内核参数来启用的功能。它能够使一台虚拟机具有物理机CPU特性,支持vmx或者svm(AMD)硬件虚拟化,关于nested的具体介绍,可以看[这里](https://www.kernel.org/doc/Documentation/virtual/kvm/nested-vmx.txt)    

首先查看一台普通的KVM虚拟机CPU信息   

``` shell
[root@localhost ~]# lscpu 
Architecture:          x86_64
...
Vendor ID:             GenuineIntel
Hypervisor vendor:     KVM
Virtualization type:   full
...

[root@localhost ~]# cat /pro/cpuinfo
processor    : 1
vendor_id    : GenuineIntel
cpu family    : 6
model        : 13
model name    : QEMU Virtual CPU version (cpu64-rhel6)
...
flags        : fpu de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pse36 clflush mmx fxsr sse sse2 syscall nx lm unfair_spinlock pni cx16 hypervisor lahf_lm
```

可以看到,这个虚拟机虚拟化类型为全虚拟化,使用的CPU为QEMU软件模拟出来的CPU,并不支持硬件虚拟化(flags中没有vmx)

## 物理服务器上开启nested支持    

要使物理机内核支持`nested`,第一步需要升级系统内核到`kernel 3.X`版本,第二步要为内核添加新的引导参数    

默认情况下,系统并不支持nested    

``` shell
#查看当前系统是否支持nested
systool -m kvm_intel -v   | grep -i nested
nested              = "N"
#或者这样查看
cat /sys/module/kvm_intel/parameters/nested 
N
```

第一步升级内核,用3.18内核做测试,升级内核很简单,下载编译好的内核rpm包,[这里是下载地址](http://mirrors.neterra.net/elrepo/kernel/el6/x86_64/RPMS/),安装,然后修改`grub.conf`使用新内核   

第二步添加引导参数同样很简单,只需要在 kernel 那一行的末端加上`kvm-intel.nested=1`参数  
    
``` shell
#升级内核
rpm -ivh kernel-ml-3.18.3-1.el6.elrepo.x86_64.rpm

#修改grub.conf
default=0              #使用新内核
timeout=5
splashimage=(hd0,0)/grub/splash.xpm.gz
hiddenmenu
title Red Hat Enterprise Linux Server (3.18.3-1.el6.elrepo.x86_64)
    root (hd0,0)
    kernel /vmlinuz-3.18.3-1.el6.elrepo.x86_64 ro root=UUID=9c1afc64-f751-473c-aaa6-9161fff08f6f rd_NO_LUKS rd_NO_LVM LANG=en_US.UTF-8 rd_NO_MD SYSFONT=latarcy
rheb-sun16 crashkernel=auto  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM rhgb quiet kvm-intel.nested=1
    ...
```

上面修改之后,重启系统,用"uname -r"查看系统内核,并检查nested是否支持   

## 建立一台支持"vmx"的虚拟机

如果你使用libvirt管理虚拟机,需要修改虚拟机xml文件中CPU的定义,下面三种定义都可以

``` shell
#可以使用这种
  <cpu mode='custom' match='exact'>
    <model fallback='allow'>core2duo</model>
    <feature policy='require' name='vmx'/>
  </cpu>
#这种方式为虚拟机定义需要模拟的CPU类型"core2duo",并且为虚拟机添加"vmx"特性

#也可以使用这种
  <cpu mode='host-model'>
    <model fallback='allow'/>
  </cpu>

#或者这样
 <cpu mode='host-passthrough'>
    <topology sockets='2' cores='2' threads='2'/>
  </cpu>
#CPU穿透,在虚拟机中看到的vcpu将会与物理机的CPU同样配置,这种方式缺点在于如果要对虚拟机迁移,迁移的目的服务器硬件配置必须与当前物理机一样
```

如果你使用qemu-kvm命令行启动虚拟机,那么可以简单的添加

``` shell
-enable-kvm -cpu qemu64,+vmx
#设置虚拟机CPU为qemu64型号,添加vmx支持
```

然后启动虚拟机,查看配置

``` shell    
#下面虚拟机CPU定义为"host-model"
cat /proc/cpuinfo 
processor    : 0
vendor_id    : GenuineIntel
cpu family    : 6
model        : 26
model name    : Intel Core i7 9xx (Nehalem Class Core i7)
...
wp        : yes
flags        : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc unfair_spinlock pni vmx ssse3 cx16 pcid sse4_1 sse4_2 x2apic
...
```

**参考文章**    

><small>http://kashyapc.com/2012/01/14/nested-virtualization-with-kvm-intel/  
http://networkstatic.net/nested-kvm-hypervisor-support/</blockquote></small>



