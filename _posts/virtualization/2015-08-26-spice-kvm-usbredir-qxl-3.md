---
title: "spice在kvm虚拟机中的应用(三)"
author: opengers
layout: post
permalink: /virtualization/spice-kvm-usbredir-qxl-3/
categories: virtualization
tags:
  - virtualization
  - kvm
  - usbredir
  - qxl
format: quote
---

><small>本系类其它文章   
[spice在kvm虚拟机中的应用(一)](http://www.isjian.com/virtualization/spice-kvm-usbredir-qxl-1/)   
[spice在kvm虚拟化中的应用(二)](http://www.isjian.com/virtualization/spice-kvm-usbredir-qxl-2/)</small>   

# spice的USB重定向  

### 介绍  

使用usb重定向,在client上插入的U盘会被重定向到虚拟机中. 其有两种实现方式,自动重定向(所有插入client中的U盘都被重定向),或者手动选择需要重定向的U盘  

USB重定向需要为虚拟机添加USB2 EHCI驱动,以及若干个Spice channels,Spice channels的个数决定了客户端一次可以有多少个USB设备被重定向到guest, 更多参考：  
[USB redirection](http://people.freedesktop.org/~teuf/spice-doc/html/ch02s06.html)  
[UsbRedir](http://www.spice-space.org/page/UsbRedir)  
[Features/UsbNetworkRedirection](http://fedoraproject.org/wiki/Features/UsbNetworkRedirection)  

### 服务器上安装软件  

``` shell
[root@controller2 vhosts]# rpm -qa | grep usb
usbredir-0.5.1-1.el6.x86_64
libusb-0.1.12-23.el6.x86_64
usbutils-003-4.el6.x86_64
libusb1-1.0.9-0.6.rc1.el6.x86_64
```

### 虚拟机xml文件中添加USB redirection驱动  

首先关闭虚拟机，然后修改其xml文件，添加下面标签  

``` html
<!-- 移除xml文件中其它的USB设备,然后添加下面的部分 -->
<controller type='usb' index='0' model='ich9-ehci1'/>
<controller type='usb' index='0' model='ich9-uhci1'>
  <master startport='0'/>
</controller>
<controller type='usb' index='0' model='ich9-uhci2'>
  <master startport='2'/>
</controller>
<controller type='usb' index='0' model='ich9-uhci3'>
  <master startport='4'/>
</controller>
<redirdev bus='usb' type='spicevmc'/>
<redirdev bus='usb' type='spicevmc'/>
<redirdev bus='usb' type='spicevmc'/>
<redirdev bus='usb' type='spicevmc'/>
```

上面是在xml文件中添加usb驱动，其对应的命令行参数如下(当使用qemu-kvm命令行启动虚拟机时需要)：

``` shell 
-device ich9-usb-ehci1,id=usb \
-device ich9-usb-uhci1,masterbus=usb.0,firstport=0,multifunction=on \
-device ich9-usb-uhci2,masterbus=usb.0,firstport=2 \
-device ich9-usb-uhci3,masterbus=usb.0,firstport=4 \
-chardev spicevmc,name=usbredir,id=usbredirchardev1 \
-device usb-redir,chardev=usbredirchardev1,id=usbredirdev1 \
-chardev spicevmc,name=usbredir,id=usbredirchardev2 \
-device usb-redir,chardev=usbredirchardev2,id=usbredirdev2 \
-chardev spicevmc,name=usbredir,id=usbredirchardev3 \
-device usb-redir,chardev=usbredirchardev3,id=usbredirdev3
```

### 客户端配置  

客户端连接工具使用`virt-viewer`, windows7版本的`virt-viewer`默认不支持USB重定向,需要手动重新编译,Linux下的客户端可以通过编译源码支持USB重定向  
virt-viewer源码: [virt-viewer-1.0.tar.gz](http://virt-manager.org/download/sources/virt-viewer/virt-viewer-1.0.tar.gz)  
virt-viewer windows客户端: [http://virt-manager.org/download/sources/virt-viewer/virt-viewer-x64-1.0.msi](http://virt-manager.org/download/sources/virt-viewer/virt-viewer-x64-1.0.msi)  

# spice使用TLS和密码实现双重认证  

默认情况下，客户端和虚拟机传输的数据是未加密的，下面的步骤中将使用TLS加密客户端和虚拟机之间的连接   

### 生成CA证书，服务器证书

**创建证书存放目录**  

``` shell
cd /etc/pki
mkdir libvirt-spice
cd libvirt-spice
```

**使用下面脚本创建证书**  

注意：脚本生成的`ca-cert.pem`文件，最后输出的变量”SUBJECT“值都需要拷贝到客户端   

``` shell
#!/bin/bash
 
SERVER_KEY=server-key.pem
# creating a key for our ca
if [ ! -e ca-key.pem ]; then
 openssl genrsa -des3 -out ca-key.pem 1024
fi
# creating a ca
if [ ! -e ca-cert.pem ]; then
 openssl req -new -x509 -days 1095 -key ca-key.pem -out ca-cert.pem  -subj "/C=IL/L=Raanana/O=Red Hat/CN=my CA"
fi
# create server key
if [ ! -e $SERVER_KEY ]; then
 openssl genrsa -out $SERVER_KEY 1024
fi
# create a certificate signing request (csr)
if [ ! -e server-key.csr ]; then
 openssl req -new -key $SERVER_KEY -out server-key.csr -subj "/C=IL/L=Raanana/O=Red Hat/CN=my server"
fi
# signing our server certificate with this ca
if [ ! -e server-cert.pem ]; then
 openssl x509 -req -days 1095 -in server-key.csr -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 -out server-cert.pem
fi
 
# now create a key that doesn't require a passphrase
openssl rsa -in $SERVER_KEY -out $SERVER_KEY.insecure
mv $SERVER_KEY $SERVER_KEY.secure
mv $SERVER_KEY.insecure $SERVER_KEY
 
# show the results (no other effect)
openssl rsa -noout -text -in $SERVER_KEY
openssl rsa -noout -text -in ca-key.pem
openssl req -noout -text -in server-key.csr
openssl x509 -noout -text -in server-cert.pem
openssl x509 -noout -text -in ca-cert.pem
 
# copy *.pem file to /etc/pki/libvirt-spice
if [[ -d "/etc/pki/libvirt-spice" ]]
then
 cp ./*.pem /etc/pki/libvirt-spice
else
 mkdir /etc/pki/libvirt-spice
     cp ./*.pem /etc/pki/libvirt-spice
fi
 
# echo SUBJECT
echo "SUBJECT is:" \" `openssl x509 -noout -text -in server-cert.pem | grep Subject: | cut -f 10- -d " "` \"
```

### 虚拟机加载证书  

默认不管vnc还是spice都是监听在`127.0.0.1`上,这样肯定不能从网络中访问, 下面的设置默认会使所有的虚拟机开启两个端口,一个普通端口,一个为使用ssl加密的安全端口，并且监听本机所有端口(不安全的网络环境中慎用)  

``` shell
#/etc/libvirt/qemu.conf
spice_listen="0.0.0.0"
#默认开启ssl安全端口
spice_tls=1
spice_tls_x509_cert_dir="/etc/pki/libvirt-spice"

#下面的为默认密码认证,仅当虚拟机xml文件中没有设置passwd参数时才生效,为了能够使用不同密码,这里不启用,改在xml文件中设置密码
#spice_password = "123456"
```

重启libvirtd加载证书  

    /etc/init.d/libvirtd restart

### 在虚拟机xml文件中设置密码及安全端口    

xml文件中安全端口可以有不同设置方法   

A方法：  

``` html
    <graphics type='spice' autoport='yes' listen='0.0.0.0' passwd='123456'>
```

B方法：  

``` html
    <graphics type='spice' port='5901' autoport='no' listen='0.0.0.0' passwd='123456'>
```

C方法：  

``` html
    <graphics type='spice' tlsPort='-1' autoport='no' listen='0.0.0.0' passwd='123456'>
```

- A: 每台虚拟机自动配置两个端口,普通端口和安全端口,并且端口号自动分配(5900+N)  
- B: 不自动配置端口,手动指定一个普通端口,不开启安全端口  
- C: 不自动配置端口,只开启安全端口,并且安全端口自动分配(5900+N)  
- passwd=123456  设置使用密码认证,即客户端连接虚拟机时,会弹出密码验证窗口  

### windows客户端中使用spice加密连接  
 
**拷贝ca-cert.pem证书**      

拷贝服务器上脚本生成的`ca-cert.pem`文件到windows下某个目录,比如`F:\files\ca\`

**windows中添加环境变量**    

``` shell
变量名: SUBJECT 
变量值: C=IL, L=Raanana, O=Red Hat, CN=my server  
```

(变量值为脚本最后输出内容),添加环境变量不是必须的操作,是为了下面能够使用%SUBJECT%这个变量代替一长串字符`C=IL, L=Raanana, O=Red Hat, CN=my server`  

**在cmd中测试连接**  

打开cmd, 进入`remote-viewer.exe`程序所在目录,默认为`C:\Program Files\VirtViewer\bin`

``` shell
#运行命令
remote-viewer.exe --spice-ca-file F:\ca\ca-cert.pem spice://192.168.11.166?tls-port=5905 --spice-host-subject="%SUBJECT%"
```

### Linux客户端中使用spice加密连接

``` shell
#首先安装virt-viewer客户端
yum install virt-viewer

#使用remote-viewer工具
remote-viewer --spice-ca-file ca-cert.pem --spice-host-subject 'C=IL,L=Raanana,O=Red Hat,CN=my server' spice://192.168.11.166/?tls-port=5903

#也可以把'C=IL,L=Raanana,O=Red Hat,CN=my server'部分设置为一个全局环境变量SUBJECT,以简化命令
```

# spice的多显示器支持       

spice允许客户端使用多个显示器连接到同一台虚拟机,为了实现这个功能,虚拟机必须配置有多个qxl设备驱动(对于Windows虚拟机),或者有一个配置为支持多个heads的qxl设备驱动(Linux虚拟机)  
为了支持多显示器,必须为虚拟机配置qxl驱动,虚拟机中也需要安装qxl驱动支持`xorg-x11-drv-qxl`,参考[http://www.spice-space.org/download.html](http://www.spice-space.org/download.html)中guest部分  

### Linux虚拟机配置  

对于Linux虚拟机,配置好qxl驱动之后,默认会启用多显示器支持.如果Linux系统版本过旧,可以参考这个[Much improved multiple monitor support for Linux guests now available in Fedora-18+](http://hansdegoede.livejournal.com/12969.html)

### windows虚拟机配置  

修改xml文件,添加多个video标签,然后重新启动虚拟机

``` html
    <video>
        <model type='qxl'>
    </video>
    <video>
        <model type='qxl'>
    </video>
```

# 多客户端支持   

多客户端支持允许多个用户连接同一台虚拟机，参考[MultipleClients](http://www.spice-space.org/page/Features/MultipleClients)  

### 使用qemu-kvm命令行  

对于使用qemu-kvm命令行创建的虚拟机，只需要给宿主机添加下面的环境变量  

``` shell
export SPICE_DEBUG_ALLOW_MC=1
```

添加之后，用qemu-kvm命令创建虚拟机，可以看到输出中多了一行，表示spice已经启用多客户端支持   


### 使用libvirt  

对于使用libvirt管理的虚拟机，添加上面的环境变量不生效，需要修改虚拟机xml文件, 如下，使用`qemu:commandline`标签传递变量`SPICE_DEBUG_ALLOW_MC`值给虚拟机  
 
``` html
    <!-- 更改xml文件第一行为下面这行 -->
    <domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
    <!-- 在下面类似位置添加 -->
    <domain>
      <devices>
      ...
      </devices>
      <qemu:commandline>
        <qemu:env name='SPICE_DEBUG_ALLOW_MC' value='1'/>
      </qemu:commandline>
    </domain>
```

添加上面的之后，重启虚拟机，即可生效，如果要验证添加的参数是否生效，可以在启动虚拟机时，查看虚拟机日志输出  

``` shell
tail -f /var/log/libvirt/qemu/cos_v1.log
2014-12-26 10:06:10.763+0000: starting up
LC_ALL=C PATH=/sbin:/usr/sbin:/bin:/usr/bin HOME=/root USER=root LOGNAME=root QEMU_AUDIO_DRV=spice SPICE_DEBUG_ALLOW_MC=1 /usr/libexec/qemu-kvm -name cos_v1 -S -M rhel6.5.0 
...
...
char device redirected to /dev/pts/7
(null):29858): Spice-Warning **: reds.c:4010:do_spice_init: spice: allowing multiple client connections (crashy)
#这行表明添加成功
```