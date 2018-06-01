---
title: openstack keystone tips                            
author: opengers
layout: post
permalink: /openstack/openstack-keystone-tips/
categories: openstack
tags:
  - openstack
  - keystone
  - keystoneauth1
  - 
---   

* TOC
{:toc}    

## keystone介绍  

## token

**如何查看Fernet token内容**    

Fernet Token不需要持久存储，因为



   

## keystoneauth1组件    

## keystonemiddleware


keystone概念
    * User: represent an individual API consumer. user必须属于一个特定的Domain, 因此user并不是全局唯一，只在其Domain内唯一
    * Project：项目拥有的资源的合集，A project itself must be owned by a specific domain
    * Group：组可以跟项目或者域建立相应关系
    * Service：OpenStack服务，比如Nova、Swift,每个service提供一个或者多个endpoint供用户访问其资源
    * Role: assignment to a user or group on a domain or project
    * Credentials 凭证，用户名密码
    * Endpoint：一个网络可访问的服务地址，通过它你可以访问一个服务。不同 region 有不同的service endpoint。比如，当 Nova 需要访问 Glance 服务去获取 image 时，Nova 通过访问 Keystone 拿到 Glance 的 endpoint，然后通过访问该 endpoint 去获取Glance服务。我们可以通过Endpoint的 region 属性去定义多个 region
    * Policy: 用来控制某一个User在某个Project中某个操作的权限，policy是根据token中的角色信息来判断用户是否有相应的操作权限。一个基于规则的身份验证引擎，通过配置文件来定义各种动作与用户角色的匹配关系
    * unscoped token可以有查询Project列表的功能 
    * scoped token 只有通过与某个特定项目或者域绑定的令牌，才可以访问此项目或者域内的资源
    * Catalog 对外提供一个服务的查询目录，或者说是每个服务的可访问Endpoint列表，服务之间的资源访问首先需要获取到该资源的Endpoint列表，目前版本，Keystone提供的服务目录是与有访问范围的令牌同时返回给用户的
    * keystone Architecture https://docs.openstack.org/keystone/latest/getting-started/architecture.html
    * keystone架构和功能分析 http://www.pisces.ml/2016/04/24/keystone-architecture/
    * Authentication(认证) Token(令牌) Catalog(服务目录) Policy(访问控制)

keystone权限组织结构

    * 
一个 RegionOne中包含多个Domain
    * 
Domain1中包含 3 个 Projects
    * 
可以通过 Group1 将 Role admin直接赋予 Domain,那么 Group1 中的所有用户将会对 Domain1中的所有 Projects 都拥有admin权限
    * 
也可以通过 Group2 将 Role demo 只赋予 Project3,这样 Group2 中的 User 就只拥有对Project3有相应的权限，而不会影响其它 Projects


keystone源码目录架构

    1. 
内部服务如Identity、Token、Catalog 、Resource、Assignment和 Policy等
他们的架构类似都是通过router进行url路由到相应的controller上，controller进行逻辑处理，通过指定的driver调用相应的后台处理
    2. 
server目录
服务入口，主要是加载配置启动服务监听相应端口
    3. 
middleware目录
中间件支持，用于预处理request请求
    4. 
keystone通过paste deploy组织app




keystone架构







keystonemiddleware
keystone提供的身份验证中间件，用于其它openstack项目与keystone API集成，比如nova项目

#/etc/nova/api-paste.ini
[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory

[composite:openstack_compute_api_v21]
keystone = cors http_proxy_to_wsgi compute_req_id faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v21

#/etc/nova/nova.conf
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = db_mq:11211
token_cache_time = 10
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova

keystoneauth
Keystoneauth提供了在OpenStack生态系统内执行身份验证和服务请求的标准方式。它旨在与现有的OpenStack client结合使用，并简化编写新client的过程
keystoneauth1.session.Session类被引入到keystoneauth1中，它试图为各种OpenStack client提供统一的接口，这些客户端在各种服务之间共享公共认证和请求参数

#!/usr/bin/env python

from keystoneauth1.identity import v3
from keystoneauth1 import session
from keystoneclient.v3 import client as ksclient
from glanceclient import Client as gsclient
from novaclient import client as noclient
from neutronclient.v2_0 import client as neclient

auth = v3.Password(username='admin',
                   password='admin',
                   project_name='admin',
                   user_domain_name='Default',
                   project_domain_name='Default',
                   auth_url='http://controller:35357/v3')
sess = session.Session(auth=auth)
keystone = ksclient.Client(session=sess)
glance = gsclient('2', session=session)
nova = noclient.Client('2', session=sess)
neutron = neclient.Client(session=sess)

#print keystone.users.list()
#print glance.images.list()
#print nova.servers.list()
#print neutron.list_networks()

















   
## keystonemiddleware.auth_token 身份验证中间件     

文档：https://docs.openstack.org/keystonemiddleware/latest/middlewarearchitecture.html
openstack中的各个项目使用此中间件验证HTTP请求头中的Token(paste)，只有通过认证的请求才会被转发给下一个WSGI middleware或applications

#说明
At a high level, an authentication middleware component is a proxy that intercepts HTTP calls from clients and populates HTTP headers in the request context for other WSGI middleware or applications to use
The general flow of the middleware processing is:
* clear any existing authorization headers to prevent forgery
* collect the token from the existing HTTP request headers
* validate the token
    * if valid, populate additional headers representing the identity that has been authenticated and authorized
    * if invalid, or no token present, reject the request (HTTPUnauthorized) or pass along a header indicating the request is unauthorized (configurable in the middleware)
    * if the keystone service is unavailable to validate the token, reject the request with HTTPServiceUnavailable.
The following shows the default behavior of an Authentication Component deployed in front of an OpenStack service.

The Authentication Component, or middleware, will reject any unauthenticated requests, only allowing authenticated requests through to the OpenStack service.
#配置
比如nova服务配置Token验证中间件

#/etc/nova/api-paste.ini中添加authtoken中间件
[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory

[composite:openstack_compute_api_v21]
keystone = cors http_proxy_to_wsgi compute_req_id faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v21

#nova.conf中配置认证参数
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = db_mq:11211
#缓存时间
token_cache_time = 1
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova

#Token cache
以nova为例，在每个client的每次向nova-api请求中都要验证身份，这很影响nova和keystone的性能，因此keystonemiddleware能够缓存keystone返回的验证responses，这样后续不需要再次向keystone发验证请求，缓存位置根据配置可以在service进程内存中或者使用memcached

#请求验证流程(nova)
    * 
client向nova-api请求服务
curl -XGET http://172.16.207.231:8774/v2.1/servers/38a42678-da63-4377-8f20-faae59af9221 -H "X-Auth-Token:xxx"
    * 
client请求首先需要通过nova paste中配置的一系列中间件，之后才能到达nova-api服务处理
    * 
身份验证中间件keystonemiddleware.auth_token会根据client提交的Token验证用户身份
    * 
若Token使用memcached做缓存([keystone_authtoken] memcached_servers)
  1. 若缓存中找到用户提交的Token信息,则身份验证通过，转发请求到下一个中间件
  2. 若缓存未命中，则根据[keystone_authtoken]中配置的user/pass向keystone请求一个Token，然后使用此有效Token向keystone验证用户提交的Token是否有效, 验证通过后把keystone返回的验证信息存入memcached并且转发用户请求到下个中间件 
    * 
若Token不指定memcached_servers做缓存时，token会缓存在本地进程(nova-api)中, 并且会出现下面warning：


     WARNING keystonemiddleware.auth_token [-] Using the in-process token cache is deprecated as of the 4.2.0 release and may be removed in the 5.0.0 release or the 'O' development cycle. The in-process cache causes inconsistent results and high memory usage. When the feature is removed the auth_token middleware will not cache tokens by default which may result in performance issues. It is recommended to use  memcache for the auth_token token cache by setting the memcached_servers option


http://www.99cloud.net/html/2016/jiuzhouyuanchuang_0617/178.html


## oslo.Cache    

OpenStack中可以使用cache层来缓存数据，主要有以下几种场景：
（1）存储函数执行结果 keystone heat nova等项目把一些固定的属性和查询请求的结果放到cache里面，加速访问。
（2）存储keystone token token创建完成之后，不需要修改，会有大量的读操作，适合放到cache中。
（3）存储keystonemiddleware token 为neutron，cinder，nova等各个项目缓存从keystone获得的token。
（4）存储horizon用户会话数据 主要是django支持使用。


存储函数执行结果
其优势在于，很多查询函数的结果是固定的，但是又比较常用。查询一次之后，按照key-value存到cache中，再次查询时不需要访问数据库，直接从内存缓存中根据key将结果取出，可以提高很多速度。或者还有一些查询请求，查询的参数千变万化，但是结果只有固定的几类，这种更适合使用cache加速。
Liberty 版本openstack主要是nova和keystone这方面用的比较多，来看nova中的一个例子:
def get_instance_availability_zone(context, instance):
    """Return availability zone of specified instance."""
    host = instance.get('host')
    if not host:
        # 如果虚拟机还没有被分配到主机上，就把创建虚拟机时指定的zone信息取出返回
        az = instance.get('availability_zone')
        return az
    #虚拟机所在的zone取决于虚拟机所在物理机的归属的zone，所以可以生成一个
    #'azcache-主机名'样式的字符串作为cache key     
    cache_key = _make_cache_key(host)
    #获取一个连接cache的client
    cache = _get_cache()
    #尝试取出这个key在cache中的值，作为zone信息，可能为空，也可能直接是结果
    az = cache.get(cache_key)
    #取出实例对象中的availability_zone属性
    az_inst = instance.get('availability_zone')
    if az_inst is not None and az != az_inst:
        #对象属性中有zone信息，且和cache取到的zone信息不一致，那么需要重新获取zone信息
        #并且更新到cache中
        az = None
    if not az:
        #如果cache中没有取到zone信息，或者是在上一步中zone信息被清空，重新获取zone信息
        elevated = context.elevated()
        az = get_host_availability_zone(elevated, host)
        #更新cache
        cache.set(cache_key, az)
    #返回zone信息
    return az

这样除第一次查询到这台物理机上的虚拟机外，查询这台物理机上的任何虚拟机所属的zone，再也不需要访问数据库，这样极大地减少了对数据库的请求数量，提高了响应速度。 这里支持的cache后端包括memcached,redis,mongondb或者是python的dict.目前主流openstack发行版推荐的选项是memcached，简单稳定，性能和功能够用。

存储keystone token
keystone中除了Fernet格式的token外，其他格式的token都需要keystone由存储起来，存储支持以下几种driver:
•  sql 将token存放在数据库中，使用这种方法需要定期清理过期token，防止token表太大影响性能。
•  memcache 将token存放到memcache中，token本身创建之后不会被修改，只会被读，所以很适合放到cache中，加速访问，放到cache中也不需要写脚本定期清理数据库中的过期token。
•  memcache_pool 在memcache的基础上，实现了memcache的连接池，可以在线程之间复用连接。
一个常见的memcache token driver的配置可能是这样的:
[token]
caching = true
provider = keystone.token.providers.uuid.Provider
driver = keystone.token.persistence.backends.memcache.Token
[memcache]
servers = controller-1:11211,controller-2:11211,controller-3:11211
memcache driver的缺点在于，memcache本身是分布式的设计，但是并不是高可用的，如果controller-1上的的cache服务被重启，这个节点上的所有token都会丢失掉，会带来一些错误。
比这更困难的是，如果controller1网络不可达或者宕机，那么我们会观察到几乎每个openstack api请求都会有3s以上的卡顿。这是因为openstack默认使用python-memcached访问memcache，它提供的操作keystone的client继承自Thread.local类，和构建它的线程绑定。openstack服务启动后，会启动一定数量的子进程，每个http request到达，会有一个子进程接收，孵化一个线程去处理这个请求。如果用到memcache，线程会调用python-memcached 构建一个client类，通过这个client的实例对memcache进行操作。如果访问到网络不可达的memcache节点，卡住，操作超时，将这个节点标识为30秒内不可用，在这个线程内，不会再因此而卡住，但是这个http请求结束之后，下一个http请求过来，重新孵化的线程会reinit这个client，新的client丢失了旧的client的状态，还是可能会访问到卡住的memcache节点上。
社区之所以要做memcache_pool，就是为了解决这个问题，将client统一由pool管理起来，memcache节点的状态，也由pool管理起来，这样每个子进程里只会卡第一次，所以强烈推荐使用memcache_pool驱动而不是memcache。社区将memcache_pool的代码从keystone复制到oslo_cache项目中，希望所有使用memcache的项目都通过它的memcachepool去访问，避免这个问题。其中，nova在M版本支持，heat在L版本支持。
具体的各个服务如何配置使用memcache_pool driver这里不再赘述。

存储keystonemiddleware token
我们请求任何openstack服务时，该服务都要校验请求中提供的token是否合理，这部分代码显然不是每个项目都自己实现一遍，它在keystonemiddleware项目实现，并作为filter配置在各个项目的api-paste.ini文件中,如下所示:
[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
当请求到达服务时，由keystonemiddleware访问keystone来查询请求携带的token是否合法，通常我们一个token会使用很多次，所以keystonemiddleware建议使用memcache缓存，把从keystone取到的token缓存一段时间，默认是300秒，以减少对keystone的压力，提高性能。kolla项目中nova keystonemiddleware配置示例如下:
[keystone_authtoken]
[keystone_authtoken]
auth_uri = {{ internal_protocol }}://{{ kolla_internal_fqdn }}:{{ keystone_public_port }}
auth_url = {{ admin_protocol }}://{{ kolla_internal_fqdn }}:{{ keystone_admin_port }}
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = {{ nova_keystone_user }}
password = {{ nova_keystone_password }}
memcache_security_strategy = ENCRYPT
memcache_secret_key = {{ memcache_secret_key }}
memcached_servers = {% for host in groups['memcached'] %}{{ hostvars[host]['ansible_' + hostvars[host]['api_interface']]['ipv4']['address'] }}:{{ memcached_port }}{% if not loop.last %},{% endif %}{% endfor %}
一直以来，如果不配置其中的memcached_servers的的话，每个服务每个子进程都会创建一个字典作为cache存放从keystone获得的token，但是字典内容都是相似的，如果使用统一的memcache，就不会有这方面的问题。现在master版本keystonemiddleware认为这是不合适的，会导致不一致的结果和较高的内存占用，计划在N或者O版本移除这个特性。
补充：存储horizon用户会话数据
这个是django支持的功能，这里不作讨论，有兴趣的可以看The Django Book 2.0--中文版
cache相关的具体的配置项可以参考kolla项目中的cache配置，应该还是准确合适的