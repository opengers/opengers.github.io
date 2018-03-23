---
title: "kvm libvirt qemu实践系列(一)-kvm介绍"
author: opengers
layout: post
permalink: /virtualization/kvm-libvirt-qemu-1/
categories: virtualization
tags:
  - virtualization
  - kvm
  - libvirt
format: quote
---

>- 文中使用centos6.5系统, Intel CPU, Linux 2.6.32内核  
- 本篇文章介绍kvm，qemu，libvirt概念，以及在centos6.x平台下使用rpm包，源码两种方式搭建kvm虚拟化环境   
- 文中VMM为虚拟机管理程序，即hypervisor，guest为由KVM创建的虚拟机，Host为宿主机，即安装虚机的物理机，vcpu即为虚拟机的cpu，vmemory即为虚拟机内存  

# KVM简介
KVM是由Quramnet 开发，08年被RedHat收购，目前KVM由RedHat工程师开发维护，准确来说，KVM(Kernel-based Virtual Machine)只是一个linux内核模块，包括核心虚拟化模块kvm.ko，以及针对特定CPU的模块kvm-intel.ko或者kvm-amd.ko，其实现需要宿主机CPU支持硬件虚拟化，在x86平台下，CPU硬件虚拟化技术有Intel的VT-x，AMD的 AMD-V

目前，由于kvm良好发展，其能够运行在x86, ARM (Cortex A15, AArch64), MIPS32等平台下，因为vcpu是由Host硬件模拟，x86平台下Host上安装的kvm能够虚拟出一颗x86架构的vcpu，却无法虚拟出arm,Powerpc等其它架构的vcpu，同样arm平台下安装的kvm只能够虚拟出arm架构的vcpu.   

### kvm模块
``` shell
#查看宿主机CPU是否在硬件上支持虚拟化扩展特性
cat /proc/cpuinfo | grep -E "(vmx|svm)"

从Linux内核版本2.6.20开始，kvm模块就包含在Linux内核中，只需加载此模块即可
#查看kvm模块
modprobe -l | grep kvm

#加载kvm模块（Intel VT）
modprobe kvm
modprobe kvm-intel
#注意：如果加载失败，说明服务器硬件不支持或BIOS中未开启虚拟化扩展
```

如上, Linux内核加载KVM模块之后

- 从宿主机中来看，Linux内核成为一个hypervisor（VMM），虚拟机则实现为标准的Linux进程，启动一台虚拟机就是在宿主机上运行了一个进程(可以使用命令查看某个虚拟机对应的进程：`ps -ef | grep "虚拟机名字"`)，因此虚拟机可以接受linux调度程序的管理，vcpu实现为Linux线程

- KVM需要借助Linux内核，因此KVM运行在操作系统之上，幸运的是， KVM模块已包含在Linux 内核2.6.20版本及其以上版本中

- 与KVM不同，Xen的hypervisor代码完全由自己开发，本身就是一套完整的虚拟机管理程序，因此Xen可以直接安装在裸机上，不需要先安装操作系统，就像esxi

- 内核加载kvm模块后，其会暴露一个`/dev/kvm`接口来与用户空间程序（qemu）交互，提供模拟vcpu，vmemory的功能  

- 单靠内核中的kvm模块并不能启动一台虚拟机，其只能模拟vcpu,vmemory, 像io设备的模拟还需要借助用户空间程序qemu

# KVM 与 qemu
前面说过，KVM只是一个内核模块，它可以模拟虚拟机的CPU，但虚拟机的I/O设备是通过qemu这个用户空间程序来模拟的  
qemu本身就是一套完整的开源的全虚拟化解决方案，它有两种使用方式

- 第一种是单独使用，对宿主机硬件没什么要求，也并不要求宿主机CPU支持虚拟化，qemu为虚拟机操作系统模拟整套硬件环境，虚拟机操作系统感觉不到自己运行在模拟的硬件环境中，这种纯软件模拟效率很低，但可以模拟出各种硬件设备，包括像软盘驱动器这样的老旧设备  

- 第二种是作为一个用户空间工具，和运行在内核中的KVM配合完成硬件环境的模拟，qemu1.3版本之前，其有一个专门的分支版本qemu-kvm作为KVM的用户空间程序(centos6.x yum源中就是这个)，qemu-kvm通过`ioctl`调用`/dev/kvm`这个接口与KVM交互，这样KVM在内核空间模拟虚拟机CPU，qemu-kvm负责模拟虚拟机I/O设备。qemu1.3及其以后版本中，qemu-kvm分支代码已经合并到qemu的master分支中，因此在qemu 1.3以上版本中，只需在编译qemu时开启`--enable-kvm`选项就能够是qemu支持kvm，具体说明可以查看[qemu官网](http://wiki.qemu.org/)  

KVM  centos6.x yum源中，提供了一个qemu-kvm包，版本为0.12，也就是qemu的分支版本，安装此rpm包后，可以使用`/usr/libexec/qemu-kvm`命令来创建虚拟机

创建的虚机可以使用纯软件虚拟，也可以与/dev/kvm交互，因为文章开头提到的，kvm只能模拟出与宿主机cpu同架构的虚拟cpu，因此在x86架构的宿主机上，只有当创建的虚机同样是x86架构时，才能与/dev/kvm进行交互，完成虚机cpu的模拟  

当启动虚拟机时，需要指定虚拟机的CPU，内存大小，使用的虚拟磁盘，网络使用NAT还是桥接方式，有几张网卡，磁盘等，这些复杂的配置参数需要有其它程序管理，所以Libvirt就登场了  

# Libvirt 与 KVM
centos6.x下使用yum安装的qemu-kvm(qemu分支版本)，查看帮助

``` shell
/usr/libexec/qemu-kvm --help
```

前面说过，KVM虚拟机实现为标准的Linux进程，所以我们可以用ps命令查看某个指定的虚拟机进程

``` shell
[root@compute02 ~]# ps -ef | grep instance-00000078
qemu     10556     1  0 12:27 ?        00:02:15 /usr/libexec/qemu-kvm -name instance-00000078 -S -M rhel6.5.0 -cpu SandyBridge,+erms,+smep,+fsgsbase,+pdpe1gb,+rdrand,+f16c,+osxsave,+dca,+pcid,+pdcm,+xtpr,+tm2,+est,+smx,+vmx,+ds_cpl,+monitor,+dtes64,+pbe,+tm,+ht,+ss,+acpi,+ds,+vme -enable-kvm -m 4096 -realtime mlock=off -smp 2,sockets=2,cores=1,threads=1 -uuid 17f3f3a6-21fc-4608-8995-159eaaf4 -smbios type=1,product=OpenStack Nova,version=2014.1-2.el6,serial=00020003-0004-0005-0006-000700009,uuid=17f3f3a6-21fc-4608-895-159eaaf3cf74 -nodefconfig -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/instance-00000078.monitor,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc,driftfix=slew -no-kvm-pit-reinjection -no-shutdow...
#如上是一个虚拟机进程，运行命令为qemu-kvm，后面为虚拟机配置参数，这么多配置参数当然不方便虚拟机的创建和管理，因此实际操作中，我们需要一个工具能够管理这些参数
```

### libvirt
libvirt是为了更方便地管理各种Hypervisor而设计的一套虚拟化库，libvirt作为中间适配层，让底层Hypervisor对上层用户空间的管理工具(virsh，virt-manager)做到完全透明，因为libvirt屏蔽了底层各种Hypervisor的细节，为上层管理工具提供了一个统一的、较稳定的接口（API） 更多参考这个[libvirt简介](http://smilejay.com/2013/03/libvirt-introduction/)  
libvirt项目最初是为Xen设计的一套API，但是目前对KVM等其他Hypervisor的支持也非常的好。libvirt支持多种Hypervisor，既支持包括KVM、QEMU、Xen、VMware、VirtualBox等在内的平台虚拟化方案，又支持OpenVZ、LXC等Linux容器虚拟化系统，还支持用户态Linux（UML）的虚拟化。

libvirt是目前使用最为广泛的对KVM虚拟机进行管理的工具和应用程序接口（API），而且一些常用的虚拟机管理工具和云计算框架平台（如OpenStack、OpenNebula、Eucalyptus等）都在底层使用libvirt的应用程序接口，结构如下图

![qemu-1](/images/virtualization/kvm-libvirt-qemu-1/qemu-1.jpg)

和手动使用qemu-kvm命令启动虚拟机不同，Libvirt使用xml文件来定义虚拟机的配置细节，就像上面提到的配置虚拟机CPU，内存大小，网络，磁盘这些都可以在xml文件中定义，然后使用这些定义启动虚拟机，所谓更改虚拟机配置，其实就是更改这个虚拟机的xml文件参数（更改某些参数需要重启虚拟机才会生效）

# rpm包安装kvm
centos6.x下yum 安装的kvm程序为`qemu-kvm`，版本为0.12，其为qemu的分支版本

``` shell
#安装用户空间程序qemu-kvm
yum install qemu-kvm -y
#安装虚拟机磁盘管理工具
yum install qemu-img -y
#安装libvirt库
yum install libvirt libvirt-python libvirt-client libvirt-devel -y
#安装guestfish工具,guestfish工具是libguestfs库的封装
yum install libguestfs libguestfs-tools-c libguestfs-tools libguestfs-devel -y
```
安装完成之后，主程序是：`/usr/libexec/qemu-kvm`，如果手工编译qemu，则是`qemu-system-x86_64`这个，我们就是使用它来启动一台虚拟机


# 源码安装
centos6.5系统，Linux 内核2.6.32-573下，若要搭建kvm运行环境，需要下面几步

- 加载内核模块kvm,kvm_intel  

- 编译安装用户空间程序qemu  

- 编译安装libvirt，虚拟机创建，管理工具  

- 编译安装libguestfs，查看和修改虚拟机文件系统，此工具可选安装  

### 编译安装qemu

``` shell
#内核模块kvm，kvm_intel 可以在随后启动虚拟机时加载
#最小化安装系统，需要安装大量依赖包
yum -y install gcc gcc-c++ make cmake automake autoconf cpp ncurses ncurses-devel
yum -y install nc telnet screen python-setuptools ftp lrzsz mpfr cpp ppl cloog-ppl libstdc++-devel autoconf libgomp libarchive kernel-headers glibc-headers glibc-devel gcc gcc-c++ cmake automake ncurses-devel zlib-devel cyrus-sasl-devel libjpeg-turbo-devel libpng-devel libuuid-devel glib2-devel numactl-devel libXext freetype fontconfig libICE libXrender libSM libXt libXpm libXmu libXfixes urw-fonts libXinerama atk libfontenc libXfont xorg-x11-font-utils ghostscript-fonts libXdamage libXcursor Xaw3d libXaw gd libXft libXrandr libXi libXcomposite pixman cairo libunwind gperftools-libs gperftools-devel avahi-libs gnutls cups-libs ghostscript gv hicolor-icon-theme libthai pango gtk2 graphviz pprof gperftools libogg celt051 spice-server spice-protocol nspr-devel nss-util-devel keyutils-libs-devel nss-softokn-freebl-devel nss-softokn-devel nss-devel libogg-devel celt051-devel alsa-lib-devel libsepol-devel libselinux-devel pixman-devel libcom_err-devel krb5-devel openssl-devel libcacard libcacard-devel spice-server-devel libtool snappy-devel bzip2-devel libgssglue libtirpc rpcbind libgssglue-devel python-argparse libevent keyutils nfs-utils-lib nfs-utils nfs-utils-lib-devel libgpg-error-devel libgcrypt-devel sparse libcap-ng-devel libaio-devel xz-devel gnutls-devel

#安装qemu
cd /opt/ && wget http://wiki.qemu-project.org/download/qemu-2.4.1.tar.bz2
tar xvf qemu-2.4.1.tar.bz2 && cd /opt/qemu-2.4.1

#编译参数，关于具体作用，请参考./configure --help
./configure --prefix=/usr/local/qemu --target-list=x86_64-softmmu --disable-xen --enable-kvm --enable-linux-aio --enable-vhost-net --disable-smartcard-nss --enable-debug --disable-rbd --enable-numa --enable-tcmalloc --enable-spice --enable-vnc --enable-vnc-tls --enable-vnc-jpeg --enable-vnc-png --enable-guest-agent --enable-uuid --enable-coroutine-pool --enable-bzip2 --enable-snappy --enable-sparse --enable-cap-ng
#--target-list=x86_64-softmmu: 编译平台为x86平台
#--disable-xen：禁用xen支持
#--enable-kvm: 启用kvm支持(与内核中的kvm交互完成整个虚拟机硬件环境的模拟)，若不启用，则建立的虚拟机包括vcpu，vmemory都为qemu软件模拟，效率很低
#--enable-linux-aio: 启用linux aio支持，可以获得更高性能，关于异步io(aio)资料，请自行google
#--enable-vhost-net: 虚拟机网卡使用vhost支持，相比virtio，vhost效率更高
#--disable-smartcard-nss: 禁用智能卡设置
#--disable-rbd: 禁用rbd块设备支持，这个根据需要开启
#--enable-numa: libnuma支持
#--enable-spice: spice协议支持，spice协议比vnc协议拥有更好的体验，关于spice协议，请查看https://opengers.github.io/virtualization/spice-kvm-usbredir-qxl-1/

make && make install

#建立命令软链接,不建立软链接，后续编译的libvirt会找不到qemu-kvm程序
ln -s /usr/local/qemu/bin/qemu-system-x86_64 /usr/bin/qemu-kvm
ln -s /usr/local/qemu/bin/qemu-img /usr/bin/qemu-img
```

### 编译libvirt
若要在宿主机上通过libvirt进行虚机的创建和管理，则需要启动libvirtd进程，libvirtd是一个运行在宿主机上的守护进程，它负责接收libvirt客户端管理工具(virsh，virt-manager,...)发送给它的虚拟机创建、管理指令；libvirt的客户端工具virsh可以连接到本地或者远程宿主机节点上libvirtd进程，以便能够管理本地或远程节点上的虚拟机，默认情况下，我们在本地宿主机上直接执行`virsh list`等命令即可，不需要指明连接参数，若要连接到远程宿主机上，则需要指明连接参数，具体可查看[这篇文章](http://www.chenyudong.com/archives/libvirt-connect-to-libvirtd-with-tcp-qemu.html) 

``` shell
#依赖包
yum -y install libblkid-devel libidn-devel libcurl-devel libssh2-devel libudev-devel libnl-devel libattr-devel device-mapper-devel libpciaccess-devel libxml2-devel libcgroup numad parted-devel dbus-devel xml-common sgml-common xhtml1-dtds yajl-devel

#编译安装
cd /opt/ && wget http://libvirt.org/sources/libvirt-1.2.21.tar.gz
tar xvf libvirt-1.2.21.tar.gz && cd /opt/libvirt-1.2.21
./configure --prefix=/usr/local/libvirt --with-openssl --with-blkid --with-curl --with-numactl --with-ssh2 --with-udev --with-qemu --with-lxc --with-remote --with-libvirtd --with-pm-utils --with-sysctl --with-numad --with-network --with-storage-dir --with-storage-fs --with-storage-lvm --without-storage-scsi --with-storage-disk --with-virtualport --with-libxml=/usr/ --without-selinux --without-selinux-mount --without-hyperv --with-esx
#以上编译参数，根据需要改变，默认情况下configure会检查这些参数项当前系统是否支持，如果支持就启用
make
make install

#libvirtd配置文件
/bin/cp /usr/local/libvirt/etc/sysconfig/libvirtd /etc/sysconfig/

#libvirtd守护进程
/bin/cp /usr/local/libvirt/etc/rc.d/init.d/libvirtd /etc/init.d/

#建立必要的软链接
/bin/ln -s -f /usr/local/libvirt/bin/virsh /usr/bin/
/bin/ln -s -f /usr/local/libvirt/lib/libvirt.so.0 /usr/lib64/
/bin/ln -s -f /usr/local/libvirt/bin/virt-host-validate /usr/bin/
/bin/ln -s -f /usr/local/libvirt/sbin/libvirtd /usr/bin/
ldconfig

#复制libvirtd的man手册
/bin/cp -ar /usr/local/libvirt/share/man /usr/share/

#修改/etc/init.d/libvirtd中以下内容,根据实际安装位置
sed -r -i "s/^PROCESS.*$/PROCESS=\/usr\/local\/libvirt\/sbin\/libvirtd/g" /etc/init.d/libvirtd
sed -r -i "/functions/d" /etc/init.d/libvirtd
sed -r -i "/Source function library/a . /etc/rc.d/init.d/functions" /etc/init.d/libvirtd
sed -r -i "s/^LIBVIRTD_CONFIG.*/LIBVIRTD_CONFIG=\/usr\/local\/libvirt\/etc\/libvirt\/libvirtd.conf/g" /etc/init.d/libvirtd

#开机启动libvirtd
/etc/init.d/libvirtd restart
chkconfig libvirtd on
```

### 编译安装libguestfs
``` shell
#依赖包
yum -y install xz-devel redhat-rpm-config ocaml-runtime gdbm-devel patch gdb rpm-build ocaml ocaml-findlib e2fsprogs-devel glibc-static gperf genisoimage flex-devel flex bison bison-devel pcre-devel augeas-devel augeas
yum -y install augeas-devel augeas perl-Digest-SHA perl-Test-Harness perl-ExtUtils-ParseXS perl-devel perl-ExtUtils-MakeMaker perl-CPAN ncurses-term ncurses-static libXtst libasyncns libvorbis libgudev1-147 flac libsndfile pulseaudio-libs pulseaudio-libs-glib2 spice-glib avahi-glib glib-networking ocaml-findlib-devel readline-devel perl-gettext cvs gettext po4a libconfig libcap-devel yajl yajl-devel tcl expect expect-devel ocaml-camlp4 ocaml-fileutils ocaml-gettext bash-completion ocaml-fileutils-devel ocaml-gettext-devel

#下载软件包
cd /opt/
wget http://libguestfs.org/download/1.30-stable/libguestfs-1.30.4.tar.gz
wget http://libguestfs.org/download/supermin/supermin-5.1.13.tar.gz
#supermin旧版本叫做febootstrap，guestfish是依靠它才能修改虚拟机文件系统，可以查看libguestfs源码目录下的README文件了解更多信息
tar xvf supermin-5.1.10.tar.gz && cd supermin-5.1.10
./configure --prefix=/usr/local/supermin
make && make install

#建立软链接，libguestfs编译中会调用supermin程序
ln -s /usr/local/supermin/bin/supermin /usr/bin/
#测试supermin是否成功安装
supermin --list-drivers
#有detected那行表示supermin成功识别当前系统包管理器

#编译libguestfs
cd /opt/ && tar xvf libguestfs-1.30.4.tar.gz
cd /opt/libguestfs-1.30.4

#libguestfs编译中会使用libvirt库文件，我们的libvirt是自定义的安装目录，因此需要设置PKG_CONFIG_PATH，使libguestfs能够找到libvirt库文件
export PKG_CONFIG_PATH=/usr/local/libvirt/lib/pkgconfig:/usr/share/pkgconfig:/usr/lib64/pkgconfig

#不使用autoreconf可能在make阶段会报错，错误提示为automake版本不对
autoreconf
#编译
./configure --prefix=/usr/local/guestfish --with-libvirt --with-python-installdir
#运行make之后，会在libguestfs源码目录下生成run脚本，libguestfs建议使用./run guestfish args ... 方式运行命令，而不建议使用make install安装二进制命令，具体可查看源码目录下的README
make
#需要使用REALLY_INSTALL强制安装二进制命令
make install REALLY_INSTALL=yes

#guestfish,virt-* 等命令的man手册
/bin/cp -ar /usr/local/guestfish/share/man /usr/share/

#建立软链接
ln -s /usr/local/guestfish/lib/libguestfs.so.0 /usr/lib64/
ln -s /usr/local/guestfish/bin/guestfish /usr/bin/
ldconfig
for i in `ls /usr/local/guestfish/bin/virt-*`
do
        /bin/ln -s -f $i /usr/bin/
done  
```

><small>[本文链接](https://opengers.github.io/virtualization/kvm-libvirt-qemu-1/)  
以上介绍了kvm及其安装，关于创建虚拟机，及其管理方面，参考其它博文</small>
