---
layout: article
title: "自动化运维之cobbler批量部署服务器系统"
categories: linux
---
><small><small>借助cobbler完成服务器系统的批量安装  
cobbler封装了tftp,pxe,kickstart,dhcp这些技术, 而且不会使网段中多出一台dhcp服务器  
我们由一批服务器上架的流程来看cobbler所处的地位  
1. 服务器机房上架，配置好IDRAC(远程管理)  
2. 从IDRAC中启动虚拟控制台，设置服务器BIOS，RAID（BIOS参数也可以通过IDRAC用命令设置）  
3. 记录下每台服务器MAC地址(ks文件中指定pxe启动的网卡)  
4. 网段内部署cobbler服务器，定制ks文件  
5. cobbler服务器添加部署此批服务器任务  
6. 服务器从网卡启动，运行后续自动化部署步骤</small></small>

### 关于PXE  
现代服务器的网卡(NIC)一般都支持PXE(Pre-boot Execution Environment)启动。即常说的网络启动。PXE协议分为 client 和 server 端，PXE client 在网卡的 ROM 中，当计算机启动时，BIOS 把 PXE client 调入内存执由 PXE client将放置在远端的文件通过网络下载到本地运行。运行PXE协议需要设置DHCP服务器和TFTP服务器。DHCP服务器用来给PXE client(将要安装系统的主机)分配一个IP地址。PXE Client 通过 TFTP 协议到 TFTP Server 上下载所需的文件(PXEclient的ROM中，已经存在了TFTP Client)。

### 关于KickStart
KickStart是RedHat提供的一种无人值守安装系统的方式。KickStart的工作原理是通过记录安装过程中所需人工干预填写的各种参数，然后生成一个名为ks.cfg的文件；其后，只要提供给引导程序此ks文件位置，引导程序便能够完成后续的安装

### 安装cobbler
``` shell
yum install cobbler dhcp xinetd tftp-server createrepo pykickstart cman libwrap mod_wsgi
#httpd 依赖 mod_wsgi
#tftp 需要 syslinux
```

### 启动相关服务  
``` shell
service httpd start
service xinetd start
service cobblerd start
```

### 配置cobbler  
- cobbler check
  
``` shell
cobbler check
#check命令可以检查cobbler配置是否正确
#debmirror package is not installed,  这个提示不需要，针对debian的
```

- 设定server,next-server  
设定`/etc/cobbler/settings`中的 `server` 和 `next_server`  
`server`为客户端安装程序获取安装源IP，cobbler接管了httpd，因此这里就是cobbler服务器地址
一个实际的例子是cobbler服务器可能配置了多个IP，这样cobler就可以管理多个网段的服务器安装，这里需要指定的是客户机所在网段  
`next_server`为客户端指定TFTP服务器，用以获取引导所需内核文件, 我们是使用cobbler来管理tftp，dhcp，httpd等，因此这里还是cobbler服务器地址  

- 启用tftp  
修改`/etc/xinetd.d/tftp`文件,将`disable = yes` 改成 `disable = no`

- Cobbler管理dhcp, tftp
 
``` shell
sed -i 's/manage_dhcp: 0/manage_dhcp: 1/g' /etc/cobbler/settings
sed -i 's/manage_tftpd: 0/manage_tftpd: 1/g' /etc/cobbler/settings
#上面两行确保`manage_dhcp`, `manage_tftpd`参数为`1`
```

- 修改密码

``` shell
openssl passwd -1 -salt 'random-phrase-here' '123456'
#vi /etc/cobbler/settings 　#default_password_crypted参数
```

- ks脚本关闭pxe  

``` shell
sed -i 's/pxe_just_once: 0/pxe_just_once: 1/g' /etc/cobbler/settings
```

- 加载启动引导文件  

``` shell
cobbler get-loaders
```

### 重启服务  
``` shell
cobbler check
service httpd start
service xinetd start
service cobblerd start
cobbler sync
```

### centos7  
``` shell
#挂载CentOS7.2镜像到/mnt目录,运行如下命令
cobbler import --arch=x86_64 --path=/mnt/ --name=CentOS7.2
#导入镜像的同时，cobbler会自动创建一个配置文件
cobbler profile list
#更改profile名字
cobbler profile rename --name=CentOS7.2-x86_64 --newname=CentOS7.2-default-x86_64
#修改指定profile使用的ks文件
cobbler profile edit --name=CentOS7.2-default-x86_64 --kickstart=/var/lib/cobbler/kickstarts/ centos7.2-default.ks
#添加一个profile
cobbler profile add --name=centos6.3 --distro=centos6.3-x86_64
#查看profile
cobbler profile list
```

### 理解profile  
- cobbler中可以导入多个系统镜像，比如centos7，centos6，镜像name可以唯一标示它们(上步骤的`--name`参数)

``` shell
cobbler distro list
cobbler distro report --name=iso_name
```

![numa](/images/linux/cobbler/cobbler-1.png)


cobbler system add --name=centos7.2-xj --profile=CentOS7.2-oldmachine-x86_64 --ip-address=192.168.6.170 --mac-address=d4:ae:52:b9:d1:15 --interface=eth0 --netboot-enabled=1
