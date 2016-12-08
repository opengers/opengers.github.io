---
title: "kvm虚拟化环境中的时区设置"
author: opengers
layout: post
permalink: /virtualization/kvm-guest-clock-timezone/
categories: virtualization
tags:
  - virtualization
  - kvm
  - libvirt
format: quote
---

><small>本文主要介绍宿主机为centos6环境下，虚拟机的时间保持问题</small>   

## guest OS时间保持   

kvm技术是全虚拟化，guest OS并不需要做修改就可以直接运行，然而在计时方面却存在问题，guest OS计时的一种方式是通过时钟中断计数，进而换算得到，但host产生的时钟中断不能及时到达所有guest OS，因为guest OS中的中断并不是真正的硬件中断，它是由host注入的中断    

许多网络应用，web中的sessions验证等，都会调用系统时间，guest OS中若时间有误，则会引起程序出错，为此，kvm给guest vms提供了一个半虚拟化时钟(kvm-clock),在RHEL5.5及其以上版本中，都使用了kvm-clock作为默认时钟源，可以在guest 中使用下面命令查看是否使用了kvm-clock    

``` shell
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
kvm-clock
```

在kvm-clock方式下，guest OS不能直接访问Host时钟，Host把系统时间写入一个guest可以读取的内存页，这样guest就可以读取此内存页设置自身硬件时间，但是Host并不是实时更新时间到此内存页，而是在发生一个vm event(vm关机，重启等)时才更新，因此此种方式也不能保持guest时间准确无误   

在继续之前，我们要先理解系统时间和硬件时间等概念    


## UTC时间 与 本地时间

- UTC时间：又称世界标准时间、世界统一时间，UTC是以原子钟校准的，世界其它地区是以此为基准时间加上自己时区来设置其本地时间的   

- 本地时间：由于处在不同的时区，本地时间一般与UTC是不同的，换算方法就是  

``` shell
    本地时间 = UTC + 时区 或 UTC = 本地时间 - 时区
```

时区东为正，西为负，在中国，时区为东八区，也就是+8区，本地时间都使用北京时间，在linux上显示就是 CST, 所以CST=UTC+(+8小时) 或 UTC=CST-(+8小时)   

`CST`是China Standard Time，中国标准时，注意美国的中部标准时Central Standard Time也缩写为CST）  


## 系统时间 与 硬件时间

硬件时间:主板上BIOS中的时间，由主板电池供电来维持运行，系统开机时要读取这个时间，并根据它来设定系统时间（注意：系统启动时根据硬件时间设定系统时间的过程可能存在时区换算，根据/etc/localtime）  

系统时间: 就是我们执行`date` 命令看到的时间，linux系统下所有的时间调用（除了直接访问硬件时间的命令）都是使用的这个时间。   

`/etc/localtime`这个文件用来设置系统的时区，将`/usr/share/zoneinfo/`中相应时区文件拷贝到/etc下并重命名为`localtime`即可修改时区设置(也可以使用tzselect命令修改)，而且这种修改对`date`命令是即时生效的。不论是 date 还是hwclock都会用到这个文件，系统会根据这个文件的时区设置来进行UTC和本地时间之间的换算。    

## libvirt中设置虚拟机硬件时钟

kvm虚拟机一般使用libvirt进行管理，在虚拟机配置的xml文件中，有关于虚拟机硬件时钟设置项

``` shell
    <clock offset='localtime'>
    </clock>
```

clock的offset属性有`"utc","localtime","timezone","variable"`四个可选项    

如果guest OS是Linux系统，应该选用`utc`，guest OS在启动时便会向host同步一次utc时间，然后根据`/etc/localtime`中设置的时区，来计算系统时间    

如果guest OS是windows系统，则应该选用`localtime`，guest OS在启动时向host同步一次系统时间     

## 集群环境


在虚拟机集群环境中，不能够只依赖kvm管理进程提供的计时功能，文章开头也提及这种计时并不很精确，而libvirt中配置的`clock offset`选项，是用于配置虚拟机硬件时钟，虚拟机中的系统时间还要再加上相应的时区换算得到，根据以上分析，集群环境多台虚拟机时间配置需要以下几步   

``` shell
  1. libvirt中设置正确的"clock offset"  
  2. 虚拟机中设置正确的时区（/etc/localtime）   
  3. 搭建内部ntp服务器，每个五分钟进行一次时间同步   
```

