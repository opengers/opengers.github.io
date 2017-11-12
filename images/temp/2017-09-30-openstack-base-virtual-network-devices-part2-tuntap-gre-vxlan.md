---
title: openstack底层技术-各种虚拟网络设备之二(tun/tap,veth,vxlan/grep)      
author: opengers
layout: post
permalink: /openstack/openstack-base-virtual-network-devices-part2-tuntap-gre-vxlan/
categories: openstack
tags:
  - openstack
  - tun/tap
  - veth
  - vxlan/gre
---

[openstack底层技术-各种虚拟网络设备之一(Bridge,VLAN)](http://www.isjian.com/openstack/openstack-base-virtual-network-devices-part1-bridge-and-vlan/)     
[openstack底层技术-各种虚拟网络设备之二(tun/tap,veth,vxlan/grep)](http://www.isjian.com/openstack/openstack-base-virtual-network-devices-part2-tuntap-gre-vxlan/)     

* TOC
{:toc}    

上文介绍了Bridgge和VLAN设备，本文继续介绍tun/tap，vxlan/gre等设备，很多时候这些设备不是独立使用的，下文会看到         

我们知道KVM虚拟化中单个虚拟机是主机上的一个普通qemu进程，虚拟机当然也需要网卡，提供虚拟网卡的就是主机上的tap设备(当然可以直接使用物理网卡)。那从主机的角度看，这个`qemu-kvm`进程是如何使用tap设备作为其网卡呢，先介绍下`tun/tap`设备概念                        

# tun/tap     

`tun/tap`设备是操作系统内核中的虚拟网络设备，他们为用户层程序提供数据的接收与传输。`tun/tap`设备使用内核模块为`tun`，其模块介绍为`Universal TUN/TAP device driver`，该模块输出了一个设备接口文件`/dev/net/tun`供用户层程序读写       

``` shell
[root@compute01 ~]# modinfo tun
filename:       /lib/modules/3.10.0-514.16.1.el7.x86_64/kernel/drivers/net/tun.ko
alias:          devname:net/tun
...
description:    Universal TUN/TAP device driver
...

[root@compute01 ~]# ls /dev/net/tun 
/dev/net/tun
```

为了使用该模块，用户层程序需要打开`/dev/net/tun`设备文件获得一个file descriptor(FD)，并且调用ioctl()向内核注册一个TUN或TAP设备，设备的名字可能是tunXX，tapXX这种。用户层程序通过对该文件的读写来向内核协议栈接收数据或向内核协议栈注入数据。当用户层程序关闭文件描述符后，其注册的TUN或TAP设备以及路由表相关条目(使用tun设备可能会产生路由表条目，比如openvpn)都会被内核释放。可以把tun/tap看成数据管道，它一端连接主机协议栈，另一端连接用户程序。  

TUN和TAP设备区别在于他们所在的协议栈层次不同，TAP等同于一个以太网设备，用户层程序向tap设备读写的是二层数据包如以太网数据帧，tap设备最常用的就是作为虚拟机网卡。TUN则模拟了网络层设备，操作第三层数据包比如IP数据包，`openvpn`就是使用的TUN设备             

可以看到`tun/tap`设备与主机上物理网卡有所不同，物理网卡收到的数据来自物理网络，而`tun/tap`设备收到的数据来自绑定该设备的用户空间程序，反之从物理网卡发送的数据会进入物理网络，而从`tun/tap`设备发送的数据会进入绑定该设备的用户空间程序。下面结合实例分别解释下TAP和TUN设备工作原理     

# tap设备作为虚拟机网卡     

本部分具体分析下虚拟机使用tap设备与其所在主机或外部主机通信数据流走向                            

本文依然使用下面这张图作为参考                                    

![bridge](/images/openstack/openstack-virtual-devices/bridge.png)     

如下，用户空间程序`qemu-kvm`(虚拟机P2)使用`tap0`虚拟网卡，`tap0`桥接在br0网桥上                      

``` shell
#virsh list --all
 Id    Name                           State
----------------------------------------------------
 23    P2                             running

#virsh domiflist 9
Interface  Type       Source     Model       MAC
-------------------------------------------------------
tap0       bridge     br0        virtio      fa:16:3e:b1:70:52
...
```

查看此虚拟机进程    

``` shell
#ps -ef |grep P2
qemu      7748     1  0 Nov07 ?        00:22:09 /usr/libexec/qemu-kvm -name guest=P2 ... -netdev tap,fd=26,id=hostnet0,vhost=on,vhostfd=28 ...      
```

进程PID为7748，其网络部分参数中，`-netdev tap,fd=26` 表示连接主机上的tap设备，`fd=26`表示使用文件描述符`26`来读写`/dev/net/tun`设备文件。我们可以使用`lsof -p 7748`来验证该进程使用的文件描述符    

``` shell
# lsof -p 7748
COMMAND    PID USER   FD      TYPE             DEVICE    SIZE/OFF     NODE NAME
...
qemu-kvm 24571 qemu   26u      CHR             10,200         0t0    17439 /dev/net/tun
...

#PID为24571的进程打开了文件/dev/net/tun，分配的文件描述符为26，打开设备文件类型为CHR             
```

因此，在虚拟机P2启动时，其打开了设备文件`/dev/net/tun`并获得了读写该文件的文件描述符(FD)26，同时向内核注册了一个tap设备`tap0`作为其网卡，`tap0`与FD 26关联。  


# openvpn中使用的tun设备                


