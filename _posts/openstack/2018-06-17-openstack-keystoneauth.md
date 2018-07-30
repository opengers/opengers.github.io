---
title: openstack中统一认证组件keystoneauth介绍及使用                                                       
author: opengers
layout: post
permalink: /openstack/openstack-keystoneauth/
categories: openstack
tags:
  - openstacksdk
  - openstack
---   

* TOC
{:toc}    

# keystoneauth组件介绍                  

与openstack api交互有多种方法    

1. 可以自己开发client直接调用api           
1. 可以直接使用openstack提供的封装sdk项目[openstacksdk](https://docs.openstack.org/openstacksdk/latest/)                     
1. 可以使用python-xxxclient等项目，比如`python-novaclient`，`python-neutronclient`      
1. 可以直接使用命令行工具`nova`，`neutron`，`openstack`等        
     
首先本文中client指那些与openstack api交互的项目，比如python-xxxclient或openstacksdk，这些client封装了具体的api请求过程，提供给用户更友好的操作方式， 比如nova命令来自`python-novaclieent`，neutron命令来自`python-neutronclient`，openstack命令来自`python-openstackclient`。       

这些client的共性是他们都需要认证和发起请求，因此认证和请求这部分可以独立出来实现，各个client就不需要都自己实现一套认证请求流程了，这就是keystoneauth的作用，各个client可以专注于自己的功能实现      
          
keystoneauth是一个相对独立的组件，它在整个openstack生态中提供统一认证和服务请求，因此client中不再需要处理token/headers/api地址这些信息，client需要做的只是import keystoneauth，提供必要的参数(region/用户凭据/api版本等)并调用keystoneauth中的接口，keystoneauth负责管理token和组装完整的api url(根据catalog和api的path)并最终发出api请求，client可以对返回值做友好处理后返回给用户                                                   

keystoneauth中实现认证和服务请求的是`keystoneauth1.session.Session`类，我们可以用它来初始化一个Session对象，此对象可以存储用户凭据(用户名密码)和连接信息(比如token)，使用同一个Session对象的各个client之间会共享认证和请求信息。 我们先来看看如何初始化一个Session对象           

``` shell
#[root@controller openstack]# cat ses.py
#----
#!/usr/bin/env python

from keystoneauth1.identity import v3
from keystoneauth1 import session

auth = v3.Password(username='admin',
                   password='admin',
                   project_name='admin',
                   user_domain_name='Default',
                   project_domain_name='Default',
                   auth_url='http://controller:35357/v3')
sess = session.Session(auth=auth)
#----

#实例化一个Session对象    
[root@controller openstack]# python
Python 2.7.5 (default, Nov  6 2016, 00:28:07)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-11)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from ses import sess
```

如上，我们初始化了一个Session类的实例对象`sess`，初始化Session需要提供必要的参数，我们在使用命令行之前需要source的环境变量(`OS_PROJECT_NAME`,`OS_USERNAME`,`OS_PASSWORD`,...)其实最后也是传递给了keystoneauth。现在来看看能用这个对象做什么                   

# keystoneauth组件可以做什么      

## 获取token          

``` shell
>>> sess.auth
<keystoneauth1.identity.v3.password.Password object at 0x3c17750>

>>> sess.get_token()
'gAAAAABbWX72UZT-RJnHtuFXpqecNuGFIhEAke_Rira3gfUesIuyvM5w2sEV6bnXC_uyo7rOn5RmYbZwADn2eT6AuvHXkRGHVoE25A6bkkXr6vWGZAd8fXHITK751UBrg8obFBGGNWoZpPhG87qFtmZ1yuLM3uebFFB4lfCTXoJ70D0my0X1GRc'      

>>> sess.get_auth_headers()
{'X-Auth-Token': 'gAAAAABbWX72UZT-RJnHtuFXpqecNuGFIhEAke_Rira3gfUesIuyvM5w2sEV6bnXC_uyo7rOn5RmYbZwADn2eT6AuvHXkRGHVoE25A6bkkXr6vWGZAd8fXHITK751UBrg8obFBGGNWoZpPhG87qFtmZ1yuLM3uebFFB4lfCTXoJ70D0my0X1GRc'}

``` 

如上，当执行`get_token()`后，使用`sess.auth`里保存的用户凭证向keystone api提交POST请求(`POST http://controller:35357/v3/auth/tokens`)获取一个token        

此Token可以被多个使用此sess对象的client共享，这样各个client就不需要都走一边完整的获取token流程       

##过滤endpoint地址                   

我们平常在使用openstack命令时并没有指定具体的api地址，实际上，组装出完整的api地址也是keystoneauth的责任，那么Session对象是如何知道完整的api地址呢                 

``` shell
#指定endpoint过滤条件
>>> identity_endpoint_filter={'service_type': 'identity',
... 'interface': 'admin',
... 'region_name': 'bjff-1',
... 'min_version': '2.0',
... 'max_version': '3.4',}

#根据endpoint_filter获取对应的endpoint url
>>> sess.get_endpoint(sess.auth, **identity_endpoint_filter)
u'http://controller:35357/v3/'

#service_type改为compute，再获取endpoint url
>>> identity_endpoint_filter['service_type'] = 'compute'
>>> sess.get_endpoint(sess.auth, **identity_endpoint_filter)
u'http://controller:8774/v2.1/1987639927c94519ab8aaf3413a68df9'
```     

openstack中的各个服务在安装时都要向endpoint注册(`openstack endpoint list`可以看到当前openstack中已注册的所有服务)，`identity_endpoint_filter`过滤参数作用就是在openstack endpoint中找出匹配项的url，当然，我们这里是手动指定的`identity_endpoint_filter`参数，实际上的过滤参数是client根据用户输入的命令所属的服务类型以及环境变量生成的                                      

简单说下过滤参数含义：  

- service_type： 服务类型，比如`identity`, `compute`, `volume`等(openstack endpoint list中可以看到)             
- min_version,max_version: 用于过滤在其范围内的api主版本，openstack中各个服务都有其主版本，比如目前keystone是v3版本，nova是v2版本，注意不是microversion            

上面也提到，组装出完整的api地址是keystoneauth的责任，client只需要提供必要的endpoint过滤参数以及访问路径(比如`/users`)，最终完整api url就是endpoint url + '/users'          

## 直接发送api请求                    

事实上，我们可以直接用sess对象发起api请求，其它client的api请求最终也是调用的sess对象                             

``` shell
#确认下endpoint_filter
>>> identity_endpoint_filter
{'service_type': 'identity', 'interface': 'admin', 'min_version': '2.0', 'max_version': '3.4', 'region_name': 'bjff-1'}
#获取所有user(user list)
>>> response = sess.get('/users',endpoint_filter=identity_endpoint_filter)
>>> response.json()
{u'users': [{u'password_expires_at': None, u'links': {u'self': u'http://controller:35357/v3/users/006cbddb9dde423695d00d94e68f1b19'}, u'enabled': True, u'id': u'006cbddb9dde423695d00d94e68f1b19... }

#service type改为compute
>>> compute_endpoint_filter['service_type'] = 'compute'
>>> compute_endpoint_filter
{'service_type': 'compute', 'interface': 'admin', 'min_version': '2.0', 'max_version': '3.4', 'region_name': 'bjff-1'}
#获取所有虚拟机(server list)
>>> response = sess.get('/servers',endpoint_filter=compute_endpoint_filter)
>>> response.json()
{u'servers': [{u'id': u'0a57d543-1e8f-49c3-a731-c9198dd3ecb7', u'links': [{u'href': u'http://controller:8774/v2.1/1987639927c94519ab8aaf3413a68df9/servers/0a57d543-1e8f-49c3-a731-c9198dd3ecb7', u'rel': u'self'}, {u'href': ...}

#当然还可以获取所有虚拟机详细信息  
response = sess.get('/servers/detail',endpoint_filter=compute_endpoint_filter)
```

简单说下`sess.get('/servers',endpoint_filter=compute_endpoint_filter)`处理过程如下：       

1. 根据`compute_endpoint_filter`过滤出正确的endpoint url(http://controller:8774/v2.1/1987639927c94519ab8aaf3413a68df9)            
1. 组合出完整api地址`http://controller:8774/v2.1/1987639927c94519ab8aaf3413a68df9/servers`             
1. sess.get()发起请求，带有必要的参数，请求headers等信息，请求的token来自sess.get_auth_headers()                 
1. sess.get接口其实是调用的sess.request(url=URL, method=GET, json=BODY, ...)                   

## 配合client                  

上面那种直接调用Session对象发送api还是太麻烦，client对此有更高层次的封装      

比如平常使用的`nova list`命令来自`python-novaclient`项目，下面演示了`python-novaclient`中是如何使用Session对象       

列出所有镜像              

``` shell
>>> from glanceclient import Client as gsclient
>>> glance_client = gsclient('2', session=sess)
>>> glance_client.images.list()
<generator object list at 0x32b2c80>
```    

列出所有虚拟机        

``` shell
>>> from novaclient import client as noclient
>>> nova_client = noclient.Client('2', session=sess)
>>> nova_client.servers.list()
[<Server: bjff-nginx02>, <Server: bjff-nginx03>, <Server: bjff-nginx01>]

#当然，也可以创建虚拟机
>>> nova_client.servers.create('instance1',
... image='a88fe8b1-15ed-41af-898a-162447ca8d66',
... flavor='82f7cfad-d792-4d39-bde1-e89369c36244',
... nics=[{'net-id': '2bcb3404-9f7c-4bb9-9bac-521c97be19e2'}],
... min_count=1,max_count=1)
<Server: instance1>
```



