---
layout: article
title: "基于ISO制作openstack使用的coreOS镜像"
categories: virtualization
---

><small>本篇文章是使用ISO镜像手动制作openstack云平台使用的qcow2镜像文件，关于coreOS的介绍，可以看[CoreOS 实战：CoreOS 及管理工具介绍](http://www.infoq.com/cn/articles/what-is-coreos)</small>

### 下载coreOS镜像（444.5.0版本）  
``` shell
#可能需要FQ
#coreOS安装文件（coreos-install脚本会从官网自动下载，这里手动下载，可以节省时间）
wget http://stable.release.core-os.net/amd64-usr/444.5.0/coreos_production_image.bin.bz2
wget http://stable.release.core-os.net/amd64-usr/444.5.0/coreos_production_image.bin.bz2.sig
    
#iso镜像文件
wget http://stable.release.core-os.net/amd64-usr/444.5.0/coreos_production_iso_image.iso
```

### 创建虚拟磁盘  
``` shell
qemu-img create -f qcow2 coreOS_v1.qcow2 20G
#qcow2格式, 20G大小
```

### 使用virt-install工具启动ISO镜像（从光盘引导系统）  
``` shell
virt-install -n core -r 1024 -c /data_lij/coreOS/coreos_production_iso_image.iso --disk path=/data/coreOS/coreos_test.qcow2,device=disk,bus=virtio,size=5,format=qcow2 --vnc --vncport=5900 --vnclisten=0.0.0.0 -v
#-c 使用上面下载的iso
#vnc 5900端口
```

### 设置cloud-config  
><small>cloud-config介绍  
CoreOS allows you to declaratively customize various OS-level items, such as network configuration, user accounts, and systemd units. This document describes the full list of items we can configure. The `coreos-cloudinit` program uses these files as it configures the OS after startup or during runtime.  
Your cloud-config is processed during each boot. Invalid cloud-config won't be processed but will be logged in the journal. You can validate your cloud-config with the [CoreOS validator](https://coreos.com/validate) or by running `coreos-cloudinit -validate`</small>

由于coreOS默认使用密钥登陆，所以我们必须想办法注入一个公钥到系统中，就是使用cloud-config.yaml文件，最简单的cloud-config.yaml只包括一个ssh_authorized_keys字段，用于密钥注入  
新建一个cloud-config.yaml文件，ssh-rsa 后面粘贴需要注入的公钥

``` shell
#cat cloud-config
ssh_authorized_keys:
    - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0g+ZTxC7weoIJLUafOgrm+h...
#cloud_config使用yaml语法
```

### 安装coreOS到虚拟磁盘  
客户端使用vnc viewer工具连接虚拟机，当前运行的系统是我们下载的ISO镜像coreos_production_iso_image.iso，不同于centos可以直接安装系统到磁盘，对于coreOS，我们需要使用这个ISO提供的安装工具coreos-install去安装coreOS到磁盘  
*注意：最好使用第6步中的本地下载方法*

``` shell
coreos-install -d /dev/vda -C stable -V 444.5.0 -c cloud-config.yaml
#-d：参数指定安装磁盘，这里指第二步创建的虚拟磁盘
#-C：使用版本，stable稳定版
#-V：要安装的coreOS系统版本，coreos-install会根据这里指定的版本去官网下载相应版本安装程序
#-c：指定一个启动后可以执行的cloud-config配置文件，用于注入密钥
```

### 使用本地安装文件  
执行上一步的安装命令后，coreos-install会自动调用下面命令下载所需安装文件，并自动安装

``` shell
    wget http://stable.release.core-os.net/amd64-usr/444.5.0/coreos_production_image.bin.bz2
    wget http://stable.release.core-os.net/amd64-usr/444.5.0/coreos_production_image.bin.bz2.sig
```

由于网络的原因，下载可能不会成功，或者下载很慢，我们可以设置让coreos-install从本地地址下载代替从官网下载，从而节省时间  

1. 首先需要设置解析

``` shell
#使stable.release.core-os.net解析为本地IP
echo "192.168.11.166 stable.release.core-os.net" >> /etc/hosts
```

1. 配置一台http主机代替官网地址
我们需要另外使用一台虚拟机，在其上搭建一个http服务器，替代`http://stable.release.core-os.net/amd64-usr/444.5.0`这个地址，假设其IP为`192.168.11.166`

1. 创建目录结构

``` shell
mkdir /data/coreos/amd64-usr/444.5.0 -p
cd /data/coreos/amd64-usr/444.5.0
```

1. 复制第一步下载的安装文件到本目录

``` shell
cp coreos_production_image.bin.bz2 coreos_production_image.bin.bz2.sig .
```

1. 进入/data/coreos目录

``` shell
#使用python启动一个http服务，其根目录为python运行目录
cd /data/coreos
python -m SimpleHTTPServer 80
```

1. 现在coreos-install可以使用本地安装文件了，重新执行下面命令

``` shell
coreos-install -d /dev/vda -C stable -V 444.5.0 -c cloud-config.yaml
```

### 开机启动脚本cloud-init
安装完成之后，为了使其可以从openstack获取主机名，密钥等，需要添加一个开机启动脚本

``` shell
cat cloudinit.sh
#!/bin/bash

#get the env
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/bin

STATUS_CODE=`curl -I -m 10 -o /dev/null -s -w %{http_code} http://169.254.169.254/latest`
if [ ! "$STATUS_CODE" -eq "200" ]; then
    /bin/sleep 3
fi

# set the root password using user data
STATUS_CODE=`curl -I -m 10 -o /dev/null -s -w %{http_code} http://169.254.169.254/latest/user-data`
if [ "$STATUS_CODE" -eq "200" ]; then
    PASS=`curl -m 10 -s http://169.254.169.254/latest/user-data | awk -F '"' '{for(i=1;i<=NF;i++){if($i ~ /password/) print $(i+2)}}'`
    if [ "$PASS" != " " ]; then
    /usr/bin/echo "root:${PASS}" > tmp.txt
    /usr/sbin/chpasswd < tmp.txt
    rm -f tmp.txt
    fi
fi

# set the hostname using the meta-data service
STATUS_CODE=`curl -I -m 10 -o /dev/null -s -w %{http_code} http://169.254.169.254/latest/meta-data/hostname`
if [ "$STATUS_CODE" -eq "200" ]; then
    curl -f http://169.254.169.254/latest/meta-data/hostname > /tmp/metadata-hostname 2>/dev/null
    if [ $? -eq 0 ]; then
        TEMP_HOST=`cat /tmp/metadata-hostname | awk -F '.novalocal' '{print $1}'`
        /usr/bin/hostnamectl set-hostname ${TEMP_HOST}
        /usr/bin/hostname $TEMP_HOST
        rm -f /tmp/metadata-hostname
    fi
fi

# get the user ssh key using the meta-data service
STATUS_CODE=`curl -I -m 10 -o /dev/null -s -w %{http_code} http://169.254.169.254/latest/meta-data/public-keys/0/openssh-key`
if [ "$STATUS_CODE" -eq "200" ]; then
    mkdir -p /root/.ssh
    /usr/bin/echo >> /root/.ssh/authorized_keys
    curl -m 10 -s http://169.254.169.254/latest/meta-data/public-keys/0/openssh-key | grep 'ssh-rsa' >> /root/.ssh/authorized_keys
    chmod 0700 /root/.ssh
    chmod 0600 /root/.ssh/authorized_keys
fi
```

### 设置开机启动  
coreOS使用systemd管理启动项，关于systemd的介绍[systemd](http://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/index.html)

##### 新建一个开机启动配置文件cloudinit.service

``` shell
cd /etc/systemd/system
#cat cloudinit.service
[Unit]
Description=OpenStack nova
Requires=coreos-setup-environment.service
After=coreos-setup-environment.service
Before=user-config.target

[Service]
Type=oneshot
RemainAfterExit=yes
EnvironmentFile=-/etc/environment
ExecStart=/usr/bin/bash /etc/cloud-init.sh          #执行的脚本文件cloud-init.sh

[Install]
WantedBy=multi-user.target
```

##### 加入systemd管理(设置开机启动)
``` shell
systemctl enable cloudinit.service
#执行enable之后，cloudinit.service这个服务就会开机启动，从而我们的脚本cloud-init.sh就可以执行
``` [本文链接](http://www.isjian.com)





