---
title: 自动化运维之cobbler批量部署服务器系统
author: opengers
layout: post
permalink: /linux/automatic-operations-cobbler-system/
categories: linux
tags:
  - cobbler
  - pxe
  - kickstart
format: quote
---

><small>借助cobbler完成服务器系统的批量安装  
cobbler封装了tftp, pxe, kickstart, dhcp这些技术, 而且不会使网段中多出一台dhcp服务器  
文中客户端是相对cobbler服务器来说，即为需要安装系统的服务器</small> 

## 介绍  
**关于PXE**  

现代服务器的网卡(NIC)一般都支持PXE(Pre-boot Execution Environment)启动。即常说的网络启动。PXE协议分为 client 和 server 端，PXE client 在网卡的 ROM 中，当计算机启动时，BIOS 把 PXE client 调入内存执由 PXE client将放置在远端的文件通过网络下载到本地运行。运行PXE协议需要设置DHCP服务器和TFTP服务器。DHCP服务器用来给PXE client(将要安装系统的主机)分配一个IP地址。PXE Client 通过 TFTP 协议到 TFTP Server 上下载所需的文件(PXEclient的ROM中，已经存在了TFTP Client)。

**关于KickStart**  

KickStart是RedHat提供的一种无人值守安装系统的方式。KickStart的工作原理是通过记录安装过程中所需人工干预填写的各种参数，然后生成一个名为ks.cfg的文件；其后，只要提供给引导程序此ks文件位置，引导程序便能够完成后续的安装

**关于cobbler**  

相比早期的tftp+dhcp+kickstart方式，cobbler配置简单，可以管理多系统的安装，而且可以通过指定客户端mac地址的方式来确保只有指定的某些服务器可以从dhcp服务器获取IP，这样不会干扰网络中正常的dhcp服务器
我们由一批DELL服务器上架的流程来看cobbler所处的地位

1. 服务器到货，机房上架，配置IDRAC(远程管理)
1. IDRAC远程管理，设置BIOS，RAID（BIOS，RAID也可以通过IDRAC用命令操作实现）
1. 记录下每台服务器网卡MAC地址
1. 网段内部署cobbler服务器，定制ks文件
1. 根据上面获取的MAC地址，cobbler服务器添加部署任务
1. 服务器从指定网卡启动，cobbler中的DHCP服务器验证其MAC地址，通过后，运行后续自动化部署步骤

## cobbler安装配置  
**安装软件包**    

httpd 依赖 mod_wsgi，tftp 需要 syslinux

``` shell
yum install cobbler dhcp xinetd tftp-server createrepo pykickstart cman libwrap mod_wsgi
```

**启动服务**  

``` shell
service httpd start
service xinetd start
service cobblerd start
```

**cobbler check**   

check命令可以检查cobbler配置是否正确,根据提示修改配置

``` shell
cobbler check
#debmirror package is not installed,  这个是针对debian类系统的，根据需要安装
```

**设定server,next-server**   
   
我们需要设置`/etc/cobbler/settings`中的 `server` 和 `next_server`    

`server`为客户端指定安装镜像源IP，cobbler中使用httpd为客户端提供http访问，这里就是cobbler服务器地址  
`next_server`为客户端指定TFTP服务器，用以获取引导所需内核文件, 我们是使用cobbler来管理tftp，dhcp，httpd等，因此这里还是cobbler服务器地址  

cobbler服务器可能有多个网卡接口，用以连接不通网段，此时, `server`, `next_server`应为连接客户端那个网段的网卡IP

**使用Cobbler接管dhcp, tftp**  
  
cobbler之前的pxe安装方式需要手动配置系统中的`dhcp，tftp`服务，但是在cobbler中我们不再需要直接去配置系统中的`dhcp，tftp`服务, 只需更改`/etc/cobbler`下的`dhcp.template`，`tftpd.template`，然后执行`cobbler sync`，cobbler会根据我们配置的这两个`template`文件去配置系统中的`dhcp,tftp`服务  

首先允许cobbler接管dhcp，tftp

``` shell
sed -i 's/manage_dhcp: 0/manage_dhcp: 1/g' /etc/cobbler/settings
sed -i 's/manage_tftpd: 0/manage_tftpd: 1/g' /etc/cobbler/settings
#确保`manage_dhcp`, `manage_tftpd`参数为`1`
```

然后在cobbler中配置tftp  

``` shell
cat /etc/cobbler/tftpd.template 
# default: off
# description: The tftp server serves files using the trivial file transfer \
#       protocol.  The tftp protocol is often used to boot diskless \
#       workstations, download configuration files to network-aware printers, \
#       and to start the installation process for some operating systems.
service tftp
{
        disable                 = no     #确保disable为no
        socket_type             = dgram
        protocol                = udp
        wait                    = yes
        user                    = $user
        server                  = $binary
        server_args             = -B 1380 -v -s $args
        per_source              = 11
        cps                     = 100 2
        flags                   = IPv4
}
``` 

可以看到`tftpd.template`内容与系统中的tftp服务`/etc/xinetd.d/tftp`内容一样，直接修改`/etc/xinetd.d/tftp`文件是无效的，因为每当执行一次`cobbler sync`cobbler就会用`tftpd.template`覆盖`/etc/xinetd.d/tftp`文件，下面的dhcp.template同样是这样

接下来配置dhcp，修改`/etc/cobbler/dhcp.template`文件    

``` shell
subnet 192.168.6.0 netmask 255.255.255.0 {             #网段
     option routers             192.168.6.254;         #客户端网关
     default-lease-time         21600;
     max-lease-time             43200;
     next-server                192.168.6.249;         #cobbler IP
     class "pxeclients" {
          match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
          if option pxe-system-type = 00:02 {
                  filename "ia64/elilo.efi";
          } else if option pxe-system-type = 00:06 {
                  filename "grub/grub-x86.efi";
          } else if option pxe-system-type = 00:07 {
                  filename "grub/grub-x86_64.efi";
          } else {
                  filename "pxelinux.0";
          }
     }
}
```

注意，dhcp.template中有一个`range dynamic-boot`参数，它指定了dhcp服务器分配IP的范围，不要启动此参数，否则网段内会多出一台dhcp服务器
`authoritative`参数也不需要

**修改密码**

``` bash
penssl passwd -1 -salt 'random-phrase-here' '123456'
#vi /etc/cobbler/settings 　#default_password_crypted参数
```

**不要重复安装系统**

``` shell
sed -i 's/pxe_just_once: 0/pxe_just_once: 1/g' /etc/cobbler/settings
```

**加载启动引导文件**

``` shell
cobbler get-loaders
```

**重启服务**

``` shell
cobbler check
service cobblerd start
cobbler sync
service httpd restart
service xinetd restart
service dhcpd restart
#先执行`cobbler sync`同步配置，然后重启dhcp,tftp
```

cobbler配置完成，可以再使用`cobbler check`检查所有配置是否正确

## cobbler安装系统

如下，需要安装两台不同用途的客户端系统，准备安装的系统为centos7

name  | ip  | mac地址  | 用途 
--------- | -------- | -------- | -------- 
client1  | 192.168.6.170  | d4:a2:52:b9:d1:25  | kvm  
client2  | 192.168.6.171  | d4:a2:52:b9:d2:26  | nginx  

**导入centos7的镜像**  

``` shell
#挂载CentOS7.2镜像到/mnt目录,运行如下命令导入镜像
cobbler import --arch=x86_64 --path=/mnt/ --name=CentOS7.2
#`--name`是镜像名称，可以通过`cobbler distro list`查看

#导入镜像的同时，cobbler会自动创建一个profile关联此镜像，查看
cobbler profile list
```

**创建安装client1的profile**  

``` shell
#可以更改profile名字
cobbler profile rename --name=CentOS7.2-x86_64 --newname=centos7.2-kvm
#默认profile使用cobbler提供的一个ks文件，我们需要修改成自己编写的ks文件
cobbler profile edit --name=centos7.2-kvm --kickstart=/var/lib/cobbler/kickstarts/centos7.2-kvminstall.ks
```

上面我们新建的`profile`name 为`centos7.2-kvm`，关联了一个centos7的镜像和一个我们自己编写的ks文件，因此用这个profile安装的系统将会是我们关联的centos7系统，并且安装过程中会使用`centos7.2-kvminstall.ks`这个ks文件  

**创建安装client2的profile** 

``` shell
#为另一台nginx应用建立profile，使用同一个安装镜像(distro)
cobbler profile add --name=centos7.2-nginx --distro=CentOS7.2
cobbler profile edit --name=centos7.2-nginx --kickstart=/var/lib/cobbler/kickstarts/centos7.2-nginxinstall.ks
#`--distro` 是上面导入的镜像名称

#查看这两个profile
cobbler profile list
#查看某个profile使用的具体配置
cobbler profile report --name=centos7.2-nginx
```

**启动两个安装任务**  

下面启动安装client1，client2任务，注意使用的profile  

``` shell
cobbler system add --name=install_170 --profile=CentOS7.2-kvm --ip-address=192.168.6.170 --mac-address=d4:a2:52:b9:d1:25 --interface=eth0 --netboot-enabled=1
cobbler system add --name=install_171 --profile=CentOS7.2-nginx --ip-address=192.168.6.171 --mac-address=d4:a2:52:b9:d2:26 --interface=eth0 --netboot-enabled=1
#`--name` 任务名称
#`--profile` 任务使用的profile
#`--ip-address` 指定分配给客户端的ip地址
#`--mac-address` 某个客户端匹配到这个mac地址，才分配上面的ip地址给它，否则拒绝分配IP，以及安装系统
#`--interface` 这个任务使用cobbler服务器哪个网卡，跨网段安装
#若有多台客户端，可以用脚本完成
```

最后不要忘了执行sync，才能真正添加这两个任务

    cobbler sync

查看所有的安装任务

    cobbler system list
    
启动安装任务后，就可以在客户端从网络启动，自动安装系统了，上面的`--ip-address`，`--mac-address`保证了只有符合此mac地址的客户端才能安装系统  

## 理解distro，profile，system

**distro**   

cobbler中可以导入多个系统镜像，比如centos7.2，centos7.1,centos6，镜像name可以唯一标示它们,使用以下命令查看所有cobbler管理的镜像  

``` shell
cobbler distro list
cobbler distro report --name=distro_name
```

![cobbler](/images/linux/cobbler/cobbler-1.png)

**profile**    

`profile`可以理解为一种配置，一个镜像(distro)可以有多个配置(profile)，这些`profile`可能使用不同的ks文件。考虑以下情况，我们有两批服务器，都安装centos7.2，但两批服务器所需磁盘分区方式不同，预装的软件包也不一样，显然此时无法使用同一个ks文件，这时就需要借助cobbler的`profile`，我们建立两个`profile`，在两个`profile`中指定不同的ks文件，同一个`distro`，这样就实现差异化安装系统

``` shell
#查看所有的profile
cobbler profile list
#查看某个profile的具体配置(可以看到其所使用的ks文件，distro镜像源)
cobbler profile report --name=profile_name
```

![cobbler](/images/linux/cobbler/cobbler-2.png)

**system**  

每一个`system`都指定了mac地址，profile，可以说是为每个客户端量身定制

``` shell
cobbler system list
cobbler system report --name=install_170
```

## cobbler其它  
上面是安装centos7系统，其它系统步骤类似，你可以不用考虑不同系统的引导启动文件不一样，cobbler会自动处理这些  

`system`任务中指定mac，ip等信息可以只给匹配mac的客户端安装系统，不干扰网段其它机器，假如网段其它机器意外从网络启动，也不会被刷掉系统，但你也可以不指定这些信息，那么同网段所有客户端都可以从cobbler中的dhcp获取IP地址，以及安装系统  

cobbler接管了系统dhcpd服务，但还是使用系统的dhcpd来给客户端分配IP地址，只是我们不需要直接配置`dhcpd.conf`，而是配置`/etc/cobbler/dhcp.template`，`cobbler sync`后cobbler会自动同步配置给`dhcpd.conf`，查看下`/etc/cobbler/dhcp.template`的内容，会发现下面代码，在我们添加`system`任务，`cobbler sync`之后，cobbler会更新这个文件到`/etc/dhcp/dhcpd.conf`，这段代码也会更新，作用就是匹配`system`里指定的mac地址，然后分配ip地址，没有匹配到mac地址的客户端全部拒绝，这就是dhcpd的功能，并不是cobbler自身实现的

如下我们配置dhcp.template文件  

``` shell
#cat /etc/cobbler/dhcp.template
...
#for dhcp_tag in $dhcp_tags.keys():
    ## group could be subnet if your dhcp tags line up with your subnets
    ## or really any valid dhcpd.conf construct ... if you only use the
    ## default dhcp tag in cobbler, the group block can be deleted for a
    ## flat configuration
# group for Cobbler DHCP tag: $dhcp_tag
group {
        #for mac in $dhcp_tags[$dhcp_tag].keys():
            #set iface = $dhcp_tags[$dhcp_tag][$mac]
    host $iface.name {
        hardware ethernet $mac;
        #if $iface.ip_address:
        fixed-address $iface.ip_address;
        #end if
        #if $iface.hostname:
        option host-name "$iface.hostname";
        #end if
        #if $iface.netmask:
        option subnet-mask $iface.netmask;
        #end if
        #if $iface.gateway:
        option routers $iface.gateway;
        #end if
        #if $iface.enable_gpxe:
        if exists user-class and option user-class = "gPXE" {
            filename "http://$cobbler_server/cblr/svc/op/gpxe/system/$iface.owner";
        } else if exists user-class and option user-class = "iPXE" {
            filename "http://$cobbler_server/cblr/svc/op/gpxe/system/$iface.owner";
        } else {
            filename "undionly.kpxe";
        }
        #else
        filename "$iface.filename";
        #end if
        ## Cobbler defaults to $next_server, but some users
        ## may like to use $iface.system.server for proxied setups
        next-server $next_server;
        ## next-server $iface.next_server;
    }
        #end for
}
#end for
```

然后执行`cobbler sync`  

接着查看`/etc/dhcp/dhcpd.conf`文件，会发现cobbler已经同步配置到dhcpd.conf

``` shell
#cat /etc/dhcp/dhcpd.conf
...
# group for Cobbler DHCP tag: default
group {
    host generic1 {
        hardware ethernet d4:ae:52:b9:d1:25;
        fixed-address 192.168.6.170;
        filename "/pxelinux.0";
        next-server 192.168.6.249;
    }
    host generic2 {
        hardware ethernet d4:ae:52:b9:d2:26;
        fixed-address 192.168.6.171;
        filename "/pxelinux.0";
        next-server 192.168.6.249;
    }
}
```