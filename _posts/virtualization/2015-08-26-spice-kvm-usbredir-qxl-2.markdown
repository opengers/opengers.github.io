---
author: lijian
comments: true
date: 2015-08-26 02:27:09+00:00
layout: post
link: http://blog.isjian.com/2015/08/spice-kvm-usbredir-qxl-2/
slug: spice-kvm-usbredir-qxl-2
title: spice在kvm虚拟机中的应用(二)
wordpress_id: 360
categories:
- 虚拟化
tags:
- qemu-kvm
- spice
---

<blockquote>

> 
> ## 本系类其它文章
> 
> 

> 
> # [spice在kvm虚拟机中的应用(一)](http://blog.isjian.com/2015/08/spice-kvm-usbredir-qxl-1/)
> 
> 

> 
> # [spice在kvm虚拟化中的应用(三)](http://blog.isjian.com/2015/08/spice-kvm-usbredir-qxl-3/)
> 
> 
</blockquote>




## 1.系统环境




#### 1.1 测试环境


centos6.4最小化安装(centos6.x桌面版也适用)
使用yum源为163源加EPEL源
<!-- more -->


#### 1.2 spice客户端介绍


spice作为远程连接工具,可以支持远程桌面显示,鼠标拖拽,自适应分辨率,spice应用在桌面云中,主要是连接windows桌面,其支持的USB映射可以使在终端插入的U盘映射到云端的windows系统中,其效果就相当于在远端的windows系统上插入U盘
在windows平台下的spice客户端为virt-viewer,其不支持usb映射,若需要添加usb映射功能,则需要重新编译客户端,加入usb映射功能,对于非开发人员来说,其编译相当复杂

![](http://images.cnitblog.com/blog2015/673203/201503/121854370113137.png)

在centos6.x平台下可以使用yum安装virt-viewer,其也不支持usb转发,下面是编译centos6.x下的spice客户端的具体过程


## 2 下载spice所需软件




#### 2.1 安装编译工具



    
    yum groupinstall "Development tools"




#### 2.2 下载软件包



    
    #usbredir支持usb映射
    wget http://www.spice-space.org/download/usbredir/usbredir-0.7.tar.bz2
    #spice-gtk
    wget http://www.spice-space.org/download/gtk/spice-gtk-0.28.tar.bz2
    #spice客户端
    wget http://virt-manager.org/download/sources/virt-viewer/virt-viewer-2.0.tar.gz




## 3 编译安装usbredir




#### 3.1 依赖包



    
    yum install libusb*




#### 3.2 安装



    
    ./configure --prefix=/usr/local/remote-viewer
    make
    make install




## 4 编译安装spice-gtk




#### 4.1 依赖包



    
    yum install pixman*
    yum install openssl*
    yum install gtk2-devel
    yum install pulseaudio
    yum install pulseaudio-devel
    yum install pulseaudio-libs-devel
    yum install libjpeg*
    yum install libusb*
    yum install usbredir*
    yum install *gudev*




#### 4.2 安装spice-gtk



    
    ./configure --prefix=/usr/local/remote-viewer --enable-usbredir=yes --enable-smartcard=no --with-gtk=2.0
    make
    make install




## 5 安装spice客户端




#### 5.1 依赖包



    
    yum install libxml*
    yum install spice-gtk*




#### 5.2 安装spice客户端



    
    ./configure --prefix=/usr/local/remote-viewer --with-gtk=2.0 --with-spice-gtk
    make
    make install




## 6 spice客户端字体无法显示解决


由于测试机使用的是最小化安装,所以会造成spice客户端图形界面下字体无法显示,如果你使用centos桌面版编译安装这个客户端,就不会存在这个问题,解决办法是安装下面软件包

    
    yum install xorg-x11-fonts-Type1.noarch
