---
title: "kvm libvirt qemu实践系列(五)-虚拟机快照链"
author: opengers
layout: post
permalink: /virtualization/kvm-libvirt-qemu-5/
categories: virtualization
tags:
  - virtualization
  - kvm
  - libvirt
format: quote
---

>关于kvm，qemu的介绍，可以参考这篇文章：[kvm libvirt qemu实践系列(一)](https://opengers.github.io/virtualization/kvm-libvirt-qemu-1/)  


## 测试物理环境

#### 系统版本   

Red Hat Enterprise Linux Server release 6.5 (Santiago)

#### libvirt && QEMU rpm版本   

``` shell
[root@controller2 scripts]# rpm -qa | egrep "(qemu|libvirt)"
qemu-img-0.12.1.2-2.415.el6.3ceph.x86_64
qemu-kvm-tools-0.12.1.2-2.415.el6.3ceph.x86_64
libvirt-python-0.10.2-29.el6.x86_64
gpxe-roms-qemu-0.9.7-6.10.el6.noarch
libvirt-0.10.2-29.el6.x86_64
qemu-kvm-0.12.1.2-2.415.el6.3ceph.x86_64
libvirt-client-0.10.2-29.el6.x86_64
```

#### 关于QEMU版本  

rhel6上qemu默认版本为0.12，它不支持开机状态的`blockcommit`，目前最新版本2.1.2可以支持，这部分可以参考qemu官网的changelog   

支持开机状态的快照合并     

qemu 1.3版本[changelog](http://wiki.qemu.org/ChangeLog/1.3)   
>A new block job is supported: live block commit (also known as "snapshot deletion") moves data from an image to another in the backing file chain. With the current implementation of QEMU 1.3, the "source" image may not be the active one.</blockquote>

支持当前avtive快照的合并   

qemu 2.0版本[changelog](http://wiki.qemu.org/ChangeLog/2.0)

>Live snapshot merge (...-commit) can be used to merge the active layer of an image into one of the snapshots</blockquote>

## 什么是虚拟机快照链(snapshot chains)

虚拟机快照保存了虚拟机在某个指定时间点的状态（包括操作系统和所有的程序),利用快照,我们可以恢复虚拟机到某个以前的状态,比如测试软件的时候经常需要回滚系统   

快照链就是多个快照组成的关系链,这些快照按照创建时间排列成链,像下面这样,本文章要解释的就是怎么创建这条链,链中快照的相互关系,缩短链,以及如何利用这条链回滚我们的虚拟机到某个状态   

``` shell
base-image<--guest1<--snap1<--snap2<--snap3<--snap4<--当前(active)
```

如上,base-image是制作好的一个qcow2格式的磁盘镜像文件,它包含有完整的OS以及引导程序,现在以这个base-image为模板创建多个虚拟机,简单点方法,每创建一个虚拟机我们就把这个镜像完整复制一份,但这种做法效率底下,满足不了生产需要,这是就用到了qcow2镜像的特性copy-on-write  

qcow2(qemu copy-on-write)格式镜像支持快照,具有创建一个base-image,以及在base-image(backing file)基础上创建多个copy-on-write overlays镜像的能力   

解释下`backing file`和`overlay`上面那条链中,我们为base-image创建一个guest1,那么此时base-image就是guest1的backing file,guest1就是base-image的overlay,同理,为guest1虚拟机创建了一个快照snap1,此时guest1就是snap1的backing file,snap1是guest1的overlay,backing files和overlays十分有用,可以快速的创建瘦装备实例,特别是在开发测试过程中可以快速回滚到之前某个状态

如下,我们有一个centosbase的原始镜像(包含完整OS和引导程序),现在用它作为模板创建多个虚拟机,每个虚拟机都可以创建多个快照组成快照链,当然不能直接为centosbase创建快照  

![centosbase](/images/virtualization/kvm-libvirt-qemu-5/centosbase.png)   

以CentOS系统来说,我们制作了一个qcow2格式的虚拟机镜像,想要以它作为模板来创建多个虚拟机实例,有两种方法创建实例    
1. 每新建一个实例,把centosbase模板复制一份,创建速度慢
1. 使用copy-on-write技术(qcow2格式的特性),创建基于模板的实例,创建速度很快,可以查看磁盘文件大小比较一下

上图中centos1,centos2,centos3等是基于centosbase模板创建的虚拟机(guest),接下来做的测试需要用到,centos1_sn1,centos1_sn2,centos1_sn3等是实例centos1的快照链    

我们可以只用一个backing files创建多个虚拟机实例(overlays),然后可以对每个虚拟机实例做多个快照   

注意:backing files总是只读的文件,换言之,一旦新快照被创建,他的后端文件就不能更改(快照依赖于后端这种状态),参考后面的blockcommit了解更多   


## 为虚拟机创建瘦装备实例(domain)

domain是指libvirt创建的虚拟机

qemu-img是QEMU的磁盘管理工具,qemu编译之后,默认会提供这个工具,如下关系链

``` shell
                   /----- <- [centos1.qcow2] <- [centos1_A.qcow2] <- [centos1_B.qcow2]
    [(centosbase)]|                
                   \----- <- [centos2.qcow2] <- [centos2_A.qcow2] <- [centos2_B.qcow2]
```

使用模板镜像centosbase(backing file)创建两个虚拟机(基于centosbase),20G不是必须的参数   

``` shell
qemu-img create -b centosbase.qcow2 -f qcow2 centos1.qcow2 20G
qemu-img create -b centosbase.qcow2 -f qcow2 centos2.qcow2 20G
```

现在创建出来的centos1和centos2都可以用来启动一个虚拟机,因为他们依赖于backing file,所以这两个磁盘只有几百个字节大小,只有新的文件才会被写入此磁盘

``` sehll
#查看镜像信息,包括虚拟磁盘大小,实际占用空间,backing file
qemu-img info centos1.qcow2
```  

## 内置快照介绍（Internal Snapshots） 

#### 5.1 内置磁盘快照


单个qcow2镜像文件存储快照点的磁盘状态,没有新磁盘文件产生,虚拟机运行状态和关闭状态都可以创建,Libvirt 使用 'qemu-img' 命令创建关机状态的磁盘快照.


#### 5.2 内置系统还原点


使用virsh save/restore命令
可以在虚机开机状态下（内存）保存内存状态，设备状态和磁盘状态到一个指定文件中,还原的时候虚机关机，然后restore回去
多用于测试场景中，我们经常需要不断的将vm还原到某个起点，然后重新开始部署和测试。


## 6 外置快照介绍（External Snapshots）




#### 6.1 外置磁盘快照（External disk snapshot）


当一个快照被创建时，创建时当前的状态保存在当前使用的磁盘文件中，即成为一个backing file,此时一个新的overlay被创建出来保存以后写入的数据


#### 6.2 外置系统还原点（External system checkpoint）


虚拟机的磁盘状态将被保存到一个文件中，内存和设备的状态将被保存到另外一个新的文件中


## 7.内置磁盘快照创建,回滚及删除


使用centos1这个虚拟机测试内置磁盘快照的操作(参考第3节原理图)
#此虚拟机所使用磁盘为centos1.qcow2,其backing file为centosbase.qcow2,创建过程中可以观察磁盘大小的变化

    
    #查看虚拟机信息
    qemu-img info centos1.qcow2
    
    #创建快照1(centos1运行时)
    virsh snapshot-create-as centos1 centos1_sn1 centos1_sn1-desc
    
    #创建快照2(centos1关闭)
    virsh shutdown centos1
    virsh snapshot-create-as centos1 centos1_sn2 centos1_sn2-desc
    
    #查看所有快照
    virsh snapshot-list centos1
    Name Creation Time State
    ------------------------------------------------------------
    centos1_sn1 2014-12-09 16:16:23 +0800 running
    centos1_sn2 2014-12-09 16:18:38 +0800 shutoff
    centos1_sn3 2014-12-09 16:19:59 +0800 shutoff
    centos1_sn4 2014-12-09 16:21:22 +0800 running
    #running表示在虚拟机开启时创建
    
    #快照回滚
    virsh snapshot-revert --domain centos1 centos1_sn1
    virsh snapshot-revert --domain centos1 centos1_sn3
    #内置磁盘快照可以随意回滚,比如先回滚到sn1,在回滚到sn3都是OK的
    #注意一点是虚拟机开启状态下,不能回滚到State为running的快照点
    
    #快照删除
    virsh snapshot-delete centos1 centos1_sn2
    或者
    virsh snapshot-delete --domain centos1 --snapshotname centos1_sn2







## 8 外置磁盘快照创建


使用centos2这个虚拟机测试外置磁盘快照的操作(参考第3节原理图)
此虚拟机所使用磁盘为centos2.qcow2,虚拟机开启


#### 1.首先启动centos2虚拟机,查看当前所使用磁盘






    
    virsh start centos2
    virsh domblklist centos2
    Target Source
    ------------------------------------------------
    vda /data_lij/vhosts/centos2.qcow2
    hdc -


#可以看到,当前所使用磁盘为centos2.qcow2,之前说过,外置磁盘快照创建时,会保存正在使用磁盘作为backing file(此磁盘不再接受新数据,只保存快照前的数据),并创建一个新的磁盘作为overlays以等待写入新数据


#### 2.创建外置快照1(centos2启动)



    
    virsh snapshot-create-as --domain centos2 centos2_sn1 centos2_sn1-desc --disk-only --diskspec vda,snapshot=external,file=/data_lij/vhosts/centos2_sn1.qcow2 --atomic
    
    #查看快照
    virsh snapshot-list centos2
    
    #查看centos2当前所使用磁盘
    virsh domblklist centos2
    ...
    vda /data_lij/vhosts/centos2_sn1.qcow2
    ...
    #所使用磁盘已经更新到新创建的磁盘




#### 3.查看新磁盘centos2_sn1.qcow2信息



    
    qemu-img info centos2_sn1.qcow2
    ...
    backing file: /data_lij/vhosts/centos2.qcow2
    backing file format: qcow2
    ...
    #其backing file为创建快照前使用的磁盘centos2.qcow2
    #快照2,3(centos2关闭)
    virsh snapshot-create-as --domain centos2 centos2_sn2 centos2_sn2-desc --disk-only --diskspec vda,snapshot=external,file=/data_lij/vhosts/centos2_sn2.qcow2 --atomic
    virsh snapshot-create-as --domain centos2 centos2_sn3 centos2_sn3-desc --disk-only --diskspec vda,snapshot=external,file=/data_lij/vhosts/centos2_sn3.qcow2 --atomic




#### 4.查看所有外置快照



    
    virsh snapshot-list centos2
    Name Creation Time State
    ------------------------------------------------------------
    centos2_sn1 2014-12-09 16:35:44 +0800 disk-snapshot
    centos2_sn2 2014-12-09 16:41:39 +0800 shutoff
    centos2_sn3 2014-12-09 16:43:06 +0800 shutoff
    centos2_sn4 2014-12-09 16:44:46 +0800 shutoff




#### 5.查看当前使用磁盘












    
    virsh domblklist centos2
    vda /data_lij/vhosts/centos2_sn4.qcow2











#虚拟机centos2使用的是最后一个快照的磁盘(称作active),重点要理解的是快照之间是相互依赖的(上一个依赖下一个),每一部分都保存有数据,所有的快照合起来保存虚拟机的全部数据


#### 6.查看虚拟机centos2的完整快照链(centos2_sn4.qcow2为当前使用磁盘)



    
    qemu-img info --backing-chain centos2_sn4.qcow2




## 9.外置磁盘快照的合并




#### 1.合并方式


外置快照非常有用，但这里有一个问题就是如何合并快照文件来缩短链的长度，不能直接删除某个快照,因为每个快照都保存有相应的数据
有两种方式实现
**blockcommit: 从 top 合并数据到 base (即合并overlays至backing files)**
**blockpull: 将backing file数据合并至overlay中.从 base 到 top**


#### 2. blockcommit向下合并


#blockcommit可以让你将'top'镜像(在同一条backing file链中)合并至底层的'base'镜像,一旦 blockcommit #执行完成，处于最上面的overlay链关系将被自动指向到底层的overlay或base, 这在创建了很长一条链之后用来缩短链长度的时候十分有用.
#测试发现:
A qemu1.3以下版本不支持live blockcommit,
B qemu2.0以下版本不支持合并'Active'层(最顶部的overlay,即当前使用磁盘)至backing_files

#下面是示意图

![](http://images.cnitblog.com/blog2015/673203/201505/121910391739164.png)

上面这张图中,上面是我们之前给centos2虚拟机创建的4个相互依赖的外置磁盘快照,如下表示关系

当前: [base] <- sn1 <- sn2 <- sn3 <- sn4(当前使用磁盘)
目标: [base] <- sn1 <--------------- sn4

我们需要做的是合并sn2,sn3到sn1中,并删除sn2,sn3快照,下面有两种方式

    
    (method-a):
    virsh blockcommit --domain f17 vda --base /export/vmimages/sn1.qcow2 --top /export/vmimages/sn3.qcow2 --wait --verbose
    或者
    (method-b):
    virsh blockcommit --domain centos2 vda --base centos2_sn2.qcow2 --top centos2_sn3.qcow2 --wait --verbose
    virsh blockcommit --domain centos2 vda --base centos2_sn1.qcow2 --top centos2_sn2.qcow2 --wait --verbose


注: 如果手工执行*qemu-img*命令完成的话, 现在还只能用method-b. 我们还可以让centos2_sn4(active)直接指向centos2


#### 3.blockpull向上合并


#blockpull（qemu中也称作'block stream'）可以将backing合并至active，与blockcommit正好相反.
#在qemu最新版本2.1.2上测试发现,当前只能将backing file合并至正在使用的active中

    
    centosbase <-- centos2 <-- centos2_sn1 <-- centos2_sn2 <-- centos2_sn3 <-- centos2_sn4(active)
    #在上面快照链中,可以合并sn1/sn2/sn3到sn4(active),但不能合并sn1/sn2到sn3,因为sn3非当前active磁盘


#下面是blockpull合并示意图

![](http://images.cnitblog.com/blog2015/673203/201505/121910577823181.png)

使用blockpull我们可以将centos2_sn1/2/3中的数据合并至active层，最终centos2将变成active的直接后端.

我们需要做的是合并sn1,sn2,sn3到sn4(active)中,并删除sn1,sn2,sn3快照,如下表示关系
当前: [base(centos2)] <- sn1 <- sn2 <- sn3 <- sn4(当前使用磁盘)
目标: [base(centos2)] <---------------------- sn4

    
    virsh blockpull --domain centos_2 --path /data_lij/vhosts/centos2_sn4.qcow2 --base /data_lij/vhosts/centos2.qcow2 --wait --verbose
    
    #清理掉不用的快照
    virsh snapshot-delete --domain centos2 centos2_sn1 --metadata
    virsh snapshot-delete --domain centos2 centos2_sn2 --metadata
    virsh snapshot-delete --domain centos2 centos2_sn3 --metadata


#查看当前信息

    
    #查看centos2的快照
    virsh snapshot-list centos2
    
    #查看centos2当前使用磁盘
    virsh domblklist centos2
    
    #查看快照sn4的backing file
    qemu-img info centos2_sn4.qcow2


#如果要迁移虚拟机centos2,可能要将所有backing files合并至centos2_sn4(active),然后迁移centos2_sn4(active)至目的位置

centosbase <-- centos2 <-- centos2_sn1 <-- centos2_sn2 <-- centos2_sn3 <-- centos2_sn4

centosbase --> centos2 --> centos2_sn1 --> centos2_sn2 --> centos2_sn3 --> centos2_sn4

    
    #所有的backing files 都合并到centos2_sn4(active)
    virsh blockpull --domain centos2 --path /data_lij/vhosts/centos2_sn4.qcow2 --wait --verbose
    qemu-img info centos2_sn4.qcow2


#合并之后,centos2_sn4是一个完整的镜像,包括centosbase,sn1/2/3所有的数据,此时其不再需要backing files


## 10 外置快照的删除(qemu-img commit/rebase)




#### 1.方法


我们需要删除快照sn2
当前: [centosbase] <-- centos2 <-- centos2_sn1 <-- centos2_sn2 <-- centos2_sn3 <-- centos2_sn4(当前使用磁盘)
目标: [centosbase] <-- centos2 <-- centos2_sn1 <------------------ centos2_sn3 <-- centos2_sn4(当前使用磁盘)
现在删除第二个快照(sn2).有两种方式:
(1): 复制sn2数据到后端sn1,将会commit所有sn2中的数据到sn2的backing file(sn1),和virsh blockcommit类似
(2): 复制sn2数据到前段sn3,将会commit所有sn2中的数据到sn2的overlays,和virsh blockpull类似
注意: 必须保证sn1没有被其他快照作为后端(即centos2_sn1只被当前链使用)


#### 2.(1): 复制sn2数据到后端sn1



    
    qemu-img commit centos2_sn2.qcow2
    qemu-img rebase -u -b centos2_sn1.qcow2 centos2_sn3.qcow2     #让sn3指向sn1


#现在sn1中包含了之前的sn1/sn2中的数据,所以此时不再需要sn2中的数据,直接让sn3指向sn1即可,可以直接删除sn2
注意: -u代表'Unsafe mode' -- 此模式下仅仅修改了指向到的backing file名字(不复制数据)


#### 3.(2): 复制sn2数据到前段sn3



    
    qemu-img rebase -b centos2_sn1.qcow2 centos2_sn3.qcow2


#未使用-u模式的rebase将把数据也一并合并过去，即把sn2的数据写入到sn3并修改sn3指向sn1,此为默认模式
#rebase是向前段合并,commit是向后端合并

本文系参考网上资料自己测试得出，如有错误请指出，固定链接：https://opengers.github.io/virtualization/kvm-libvirt-qemu-5/


<blockquote>参考文章

http://itxx.sinaapp.com/blog/content/130

https://kashyapc.fedorapeople.org/virt/lc-2012/snapshots-handout.html</blockquote>
