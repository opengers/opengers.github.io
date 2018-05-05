---
title: openstack keystone api初始化流程(wsgi/PasteDeploy)                 
author: opengers
layout: post
permalink: /openstack/openstack-keystone-api-start-flow/
categories: openstack
tags:
  - openstack
  - keystone
  - pastedeploy
---   

在N版本以前，keystone的api服务可以使用httpd方式进行部署，也可以使用python eventlet方式部署，从N版本开始，eventlet这种方式已经从keystone中移除。keystone使用PasteDeploy来实现WSGI服务，关于wsgi，可以参考[理解 WSGI 框架](http://python.jobbole.com/84372/)，关于Paste+PasteDeploy+Routes+WebOb，可以参考[通过demo学习OpenStack开发--API服务(2)](http://www.infoq.com/cn/articles/OpenStack-UnitedStack-API2#)      

如下，是keystone v3 api初始化流程的时序图，v3版本中，已经不区分admin api和public api                          

![keystone-api-init](/images/openstack/openstack-keystone-api/keystone-api-start.png)     

简单解释下pipline, 如下paste文件，对于v3版本的api请求，根据`[composite:admin]`, 请求被map到`api_v3`这个pipeline处理, 在`api_v3`这个pipeline中(`[pipeline:api_v3]`)，前面`cors sizelimit ... s3_extension`都是keystone实现的各种中间件，用于对client请求进行过滤封装等处理，根据名字你可以大致猜出来这些中间件都做了什么，最后`service_v3`一个是app，即为keystone真正处理client请求的api服务。也就是说，client发出的http的请求会穿过前面一系列的中间件，最后才会到达keystone代码真正处理请求                  

``` shell
#/etc/keystone/keystone-paste.ini
[pipeline:api_v3]
pipeline = cors sizelimit http_proxy_to_wsgi osprofiler url_normalize request_id build_auth_context token_auth json_body ec2_extension_v3 s3_extension service_v3  

[composite:admin]
...
/v3 = api_v3
...
```     

时序图中有一处关键代码，如下      

``` shell
#pipeline[-1]即为上面的app service_v3
context.app_context = self.get_context(APP, pipeline[-1], global_conf)
#pipeline[:-1]为所有的中间件list
context.filter_contexts = [self.get_context(FILTER, name, global_conf) for name in pipeline[:-1]] 

_PipeLine.invoke(context)
    #初始化keystone app
    app = context.app_context.create()
    filters = [c.create() for c in context.filter_contexts]
    #所有中间件反序排列，即为：s3_extension, ec2_extension_v3, json_body, token_auth,...
    filters.reverse()
    for filter in filters
        #使用中间件封装初始的app
        app = filter(app)
    return app
```   

上面不好理解的话，我们用洋葱来举个例子，假如洋葱的心代表keystone服务(service_v3)，心外面的每一层洋葱皮都代表一个中间件。这样，keystone api的初始化过程就是洋葱的生成过程，首先反序排列中间件`filters.reverse()`, 这样最靠近洋葱心(service_v3)的中间件最先被封装，一直封装到最外层中间件`cors`。那client的请求处理，就是剥洋葱的过程，最先从最外面中间件开始处理(`cors`)，我们从返回的urlmap `{(None, '/v3'): <oslo_middleware.cors.CORS object at 0x5531e10>`中也可以看出，之后经过一层一层处理，一直到keystone服务。    

时序图绘制使用的这个[websequencediagrams](https://www.websequencediagrams.com/), 本博客的时序图完整代码在[这里](https://github.com/opengers/opengers.github.io/blob/dev/_posts/txt/openstack-keystone-api-start.txt)             