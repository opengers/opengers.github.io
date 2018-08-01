---
title: 理解openstack统一认证组件keystoneauth         
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

# openstack api && client                           

openstack的精华在于其丰富的api，与openstack api交互有多种方法               

- 可以直接基于openstack api自己开发client                
- 可以使用openstack官方提供的sdk[openstacksdk](https://docs.openstack.org/openstacksdk/latest/)                     
- 可以使用python-xxxclient，比如`python-novaclient`，`python-neutronclient`，python-xxxclient只是提供单个项目方面的封装                          
- 可以直接使用命令行工具`nova`，`neutron`，`openstack`等              

虽然有众多的client，但这些client都只是实现了api的部分功能，幸好各个client都易于扩展，如有需要，我们可以修改client添加自己想要的功能                      

本文中client指那些需要与openstack api交互的项目，比如python-xxxclient或openstacksdk，这些client封装了具体的api请求过程，提供给用户更友好的操作方式， 比如nova命令来自`python-novaclieent`，neutron命令来自`python-neutronclient`，openstack命令来自`python-openstackclient`。       

![openstack-api-client](/images/openstack/openstack-keystoneauth/openstack-api-client.png)            

不管哪个client，他们访问api都需要认证和发起请求，因此认证和请求这部分可以独立出来实现，这就是keystoneauth组件的作用，各个client只需传递必要的参数给keystoneauth用以控制认证和请求过程，client可以专注于自己的功能实现      

# keystoneauth组件          
          
keystoneauth是一个相对独立的组件，它在整个openstack生态中提供统一认证和服务请求，因此client中不再需要处理token/headers/api地址这些信息，client需要做的只是提供必要的参数(region/用户凭据/api版本等/请求path)并调用keystoneauth中的接口，keystoneauth负责管理token和组装完整的api url(根据catalog和api的path)并最终发出api请求，client可以对返回值做友好处理后返回给用        

keystoneauth中实现认证和服务请求的是`keystoneauth1.session.Session`类，我们可以用它来初始化一个Session对象，此对象可以存储用户凭据(用户名密码)和连接信息(比如token)，使用同一个Session对象的各个client之间会共享认证和请求信息。 我们先来看看如何初始化一个Session对象           

``` shell
#[root@controller openstack]# cat ses.py
#----
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

如上，我们初始化了一个Session类的实例对象`sess`，这里初始化Session需要提供必要的参数，这些参数其实就是我们在使用命令行之前需要source的环境变量(`OS_PROJECT_NAME`,`OS_USERNAME`,`OS_PASSWORD`,...)。现在来看看能用这个对象做什么                   

# keystoneauth的使用      

## 获取token          

``` shell
>>> sess.auth
<keystoneauth1.identity.v3.password.Password object at 0x3c17750>

>>> sess.get_auth_headers()
{'X-Auth-Token': 'gAAAAABbWX72UZT-RJnHtuFXpqecNuGFIhEAke_Rira3gfUesIuyvM5w2sEV6bnXC_uyo7rOn5RmYbZwADn2eT6AuvHXkRGHVoE25A6bkkXr6vWGZAd8fXHITK751UBrg8obFBGGNWoZpPhG87qFtmZ1yuLM3uebFFB4lfCTXoJ70D0my0X1GRc'}
``` 

如上，当执行`get_auth_headers()`后，sess使用`sess.auth`里保存的用户凭证向keystone api提交POST请求(`POST http://controller:35357/v3/auth/tokens`)获取一个token，此Token可以被多个使用此sess对象的client共享     

## 服务发现                          

在openstack api中，一个完整的请求url由`endpoint url(service url)`和`request path`组成       

openstack中的各个服务在安装时都要向endpoint注册其服务类型及endpoint url，命令`openstack endpoint list`可以看到当前openstack中已注册的所有服务，在请求token的api调用返回中也包含有endpoint url(catalog中)。openstack集群中有多个服务，因此client在请求api时需要向Session对象指明请求的服务类型(compute,network,...)以及服务的版本，这样Session对象可以获取到对应服务的正确endpoint      url，如下是获取compute服务v2版本的endpoint url                      

``` shell
>>> compute_endpoint_filter={'service_type': 'compute',
... 'interface': 'admin',
... 'region_name': 'bjff-1',
... 'min_version': '2.0',
... 'max_version': '3.4',}

>>> sess.get_endpoint(sess.auth, **compute_endpoint_filter)
u'http://controller:8774/v2.1/1987639927c94519ab8aaf3413a68df9'
```      

简单说下过滤参数含义：   

- service_type： 服务类型，比如`identity`, `compute`, `volume`等(openstack endpoint list中可以看到)                 
- min_version,max_version: 用于过滤在其范围内的api版本，openstack中各个项目都有多个版本，比如目前keystone是v3版本(v1,v2版本弃用)，nova是v2.1版本，注意这里的api版本指的是"major versions"，不是microversion，每一个"major versions"都有其专用endpoint url，而microversion是每一个主版本支持的微版本范围，microversion没有专用url，它是在请求headers中指定。 比如你可以说Compute API v2.0版本不支持microversion。Compute API v2.1支持microversion，其支持的微版本范围是`2.1 ~ 2.38`。关于api版本可以看[API Versions](https://developer.openstack.org/api-ref/compute/#api-versions)，关于microversion可以看[Microversions](https://developer.openstack.org/api-guide/compute/microversions.html)        
- region_name: endpoint所属region，就是从哪个region中过滤endpoint url                             

上面说的是Session如何获取endpoint url，同时client也会向Session对象提供请求方法和请求path，比如`GET /servers/detail`，`POST /servers`(完整的api列表可以看[Compute API](https://developer.openstack.org/api-ref/compute/))，这样，Session对象就能组装出此请求完整的url，比如获取所有虚拟机详情的url`http://controller:8774/v2.1/1987639927c94519ab8aaf3413a68df9/servers/detail`        

## 使用Session对象直接发送api请求                    

事实上，我们可以直接用sess对象发起api请求，其它client的api请求最终也是调用的sess对象                             

确认下endpoint_filter     

``` shell
>>> identity_endpoint_filter
{'service_type': 'identity', 'interface': 'admin', 'min_version': '2.0', 'max_version': '3.4', 'region_name': 'bjff-1'}
```   

获取所有user列表       

``` shell              
>>> response = sess.get('/users',endpoint_filter=identity_endpoint_filter)        
>>> response.json()
{u'users': [{u'password_expires_at': None, u'links': {u'self': u'http://controller:35357/v3/users/006cbddb9dde423695d00d94e68f1b19'}, u'enabled': True, u'id': u'006cbddb9dde423695d00d94e68f1b19... }
```

当然，也可以获取所有虚拟机列表        

``` shell
>>> compute_endpoint_filter
{'service_type': 'compute', 'interface': 'admin', 'min_version': '2.0', 'max_version': '3.4', 'region_name': 'bjff-1'}
>>> response = sess.get('/servers',endpoint_filter=compute_endpoint_filter)
>>> response.json()
{u'servers': [{u'id': u'0a57d543-1e8f-49c3-a731-c9198dd3ecb7', u'links': [{u'href': u'http://controller:8774/v2.1/1987639927c94519ab8aaf3413a68df9/servers/0a57d543-1e8f-49c3-a731-c9198dd3ecb7', u'rel': u'self'}, {u'href': ...}
```

openstack API可能会经常有些小更新，microversion的引入使得openstack API可以保持向后兼容，API每次小更新都会增加microversion的版本号，但是client完全不用改动，API有处理不同microversion版本的能力，同时如果你想使用API的新功能，只需要调整microversion到新功能最低需求版本即可        

上面说过，Compute API v2.1支持microversion，因此我们可以在请求中指定需要的microversion版本号，如果不显式指定microversion版本号，默认使用最小版本2.1        

``` shell
>>> response = sess.get('/servers', microversion='2.20', endpoint_filter=compute_endpoint_filter)    
```

简单说下`sess.get('/servers',endpoint_filter=compute_endpoint_filter)`处理过程如下：       

- 根据sess.auth凭证申请对应Token              
- 根据`compute_endpoint_filter`过滤出正确的endpoint url(`http://controller:8774/v2.1/1987639927c94519ab8aaf3413a68df9`)            
- 根据请求path`/servers`组合出完整api url(`http://controller:8774/v2.1/1987639927c94519ab8aaf3413a68df9/servers`)                   
- sess.get()发起请求，带有必要的参数，请求headers等信息，请求的token来自sess.get_auth_headers()                 
- sess.get()接口其实是调用的sess.request(url=API URL, method=GET, json=BODY, ...)                   

## client集成keystoneauth                            

上面那种直接调用Session对象发送请求还是太麻烦，更方便的是使用文章开头提到的各种client           

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



