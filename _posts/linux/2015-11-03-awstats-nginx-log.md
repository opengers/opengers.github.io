---
title: 使用awstats分析nginx日志
author: opengers
layout: post
permalink: /linux/awstats-nginx-log/
categories: linux
tags:
  - awstats
  - nginx
  - log
format: quote
---


## 1.awstats介绍





<blockquote>本文主要是记录centos6.5下安装配置awstats，并统计nginx访问日志</blockquote>





#### 1.1 awstats介绍


awstats是一款日志统计工具，它使用Perl语言编写，可统计的日志类型包括appache，nginx，ftp，mail等，awstats对nginx日志统计非常详细，如统计项

<!-- more -->



	
  * 按参观时间:  按月历史统计   按日期统计   按星期   每小时浏览次数 

	
  * 按参观者:  国家或地区   城市   IP   最近参观日期   无法反解译的IP地址   搜索引擎网站的机器人

	
  * 浏览器统计:  每次参观所花时间   文件类别   下载  存取次数  入站处   出站处   操作系统版本  浏览器版本

	
  * 反向链接:  来源网址   由那些搜索引擎转介

	
  * HTTP:  错误码   错误次数 (400)   错误次数 (403)   错误次数 (404) 

	
  * 搜索:  用以搜索的短语 用以搜索的关键词





#### 1.2 awstats目录


1.2.1 awstats有二进制rpm包，也有源码包，文中使用源码包安装，安装目录是/usr/local/awstats, wwwroot是web视图需要的资源文件，tools是一些统计脚本

    
    awstats/
    ├── docs
    ├── README.md
    ├── tools
    └── wwwroot


1.2.2 awstats配置文件目录/etc/awstats/（默认目录）

    
    awstats.www.test.com.conf    #站点www.test.com的配置文件
    awstats.bbs.test.com.conf    #站点bbs.test.com配置文件


1.2.3 awstats数据文件存放目录/var/lib/awstats/
awstats从日志统计的数据文件保存在这个目录，这些数据文件可用于生成最终的html页面

#以上目录都可以在配置文件中指定




#### 1.3 awstats脚本用法


1.3.1 awstats_configure.pl 主要用它生成各个站点配置文件



    
    /usr/local/awstats/tools/awstats_configure.pl --help
    ...
     - Create one AWStats config file (if asked)




1.3.2 awstats.pl用来生成或更新awstats数据文件

它会分析nginx的访问日志，把分析结果更新到/var/lib/awstats数据文件中去，因此分析完的日志即使删除，只要/var/lib/awstats中的数据文件不丢失，就不会影响统计结果

    
    /usr/bin/perl /usr/local/awstats/wwwroot/cgi-bin/awstats.pl -update -config=www.test.com -configdir=/etc/awstats
    -update: 更新统计数据文件
    -config: 更新www.test.com这个站点
    -configdir: 配置文件目录，/etc/awstats是默认位置，awstats会读取此目录下由"-config"参数指定的站点的配置文件(awstats.www.test.com.conf)
    


1.3.3 awstats_updateall.pl是awstats.pl的封装，更新所有站点统计数据

    
    /usr/local/awstats/tools/awstats_updateall.pl 
    ...
    Where options are:
      -awstatsprog=pathtoawstatspl
      -configdir=directorytoscan
      -excludeconf=conftoexclude[,conftoexclude2,...] (Note: awstats.model.conf is always excluded)
    
    /usr/local/awstats/tools/awstats_updateall.pl now -awstatsprog=/usr/local/awstats/wwwroot/cgi-bin/awstats.pl -excludeconf=www.test.com
    -awstatsprog: 更新脚本路径
    -excludeconf: 可选项，更新所有站点数据，但是排除www.test.com这个站点
    
    #此脚本最终调用的还是awstats.pl


1.3.4 awstats_buildstaticpages.pl用于读取数据文件/var/lib/awstats/*并创建或更新html页面

这个脚本默认只是根据/var/lib/awstats/下的数据文件更新html，如果要在html页面看到新日志的统计，要先使用awstats.pl更新数据文件，然后再更新html

    
    /usr/local/awstats/tools/awstats_buildstaticpages.pl --help
    /usr/local/awstats/tools/awstats_buildstaticpages.pl -update -config=www.test.com -lang=cn -awstatsprog=/usr/local/awstats/wwwroot/cgi-bin/awstats.pl -dir=/home/awstats-logs/web/
    -update:更新html页面之前，先更新数据文件(调用awstats.pl脚本)
    -lang: 输出语言为cn
    -dir: html页面存放目录




#### 1.4 awstats工作流程


awstats工作流程如下：



	
  1. 触发更新进程，并指定此次更新的站点

	
  2. awstats根据站点名读取/etc/awstats下相应的配置文件

	
  3. 根据配置文件中指定的日志路径分析日志，并把数据文件存放在/var/lib/awstats目录

	
  4. 随后执行构建html页面脚本，从数据文件构建html到指定目录中

	
  5. 配置web服务器，通过web访问这些html页面，得到统计结果

	
  6. 设置定时任务，自动触发更新进程，循环1，2，3，4，5步骤





#### 1.5 awstats统计结果




### awstats统计结果可以配置为静态html，也可以配置使用动态页面（动态页面支持网页端更新日志，指定查看某一个日期的统计结果，但需要cgi支持），两者的区别如下：





	
  * 静态页面中展示了具体统计结果，awstats提供有脚本用于更新统计结果，可以设置crontab实现定时更新新产生的日志

	
  * 动态页面使用GET请求来获取统计结果，GET请求中带有查询参数，这些参数最终会传递给awstats.pl脚本，awstats本身并不提供CGI支持,需要自己配置CGI支持


上面是对awstats整体介绍，下面我们以站点www.test.com,bbs.test.com为例说明配置过程，需要建立两个配置文件，在两个配置文件中分别指定www,bbs的日志路径以及其它参数


## 2.awstats配置




#### 2.1 安装



    
    cd /opt
    wget http://prdownloads.sourceforge.net/awstats/awstats-7.4.tar.gz
    tar xvf awstats-7.4.tar.gz
    mv awstats-7.4 /usr/local/awstats




#### 2.2 生成www站点配置文件


由于www站点配置文件和bbs配置文件只是读取的日志不一样，其它参数设置可以保持一致，因此本文先建立www站点配置文件，并修改好必要参数，这样就可以以此配置文件为模板，建立其它站点配置文件

配置文件默认存放于/etc/awstats下，生成配置文件步骤如下

    
    [root@lnmptest tools]# ./awstats_configure.pl 
    ...
    Example: /etc/httpd/httpd.conf
    Example: /usr/local/apache2/conf/httpd.conf
    Example: c:\Program files\apache group\apache\conf\httpd.conf
    Config file path ('none' to skip web server setup):
    > none  #我们这里配置的是nginx，所以输入none
    
    Your web server config file(s) could not be found.
    You will need to setup your web server manually to declare AWStats
    script as a CGI, if you want to build reports dynamically.
    See AWStats setup documentation (file docs/index.html)
    
    -----> Update model config file '/usr/local/awstats/wwwroot/cgi-bin/awstats.model.conf'
      File awstats.model.conf updated.
    
    -----> Need to create a new config file ?
    Do you want me to build a new AWStats config/profile
    file (required if first install) [y/N] ? y      #y表示新建一个配置文件
    
    -----> Define config file name to create
    What is the name of your web site or profile analysis ?
    Example: www.mysite.com
    Example: demo
    Your web site, virtual server or profile name:
    > www.test.com     #这个配置文件是统计www.test.com这个站点的，名字可以随便写，与在awstats.pl中"-config"参数指定的站点名保持一致就好
    
    -----> Define config file path
    In which directory do you plan to store your config file(s) ?
    Default: /etc/awstats
    Directory path to store config file(s) (Enter for default):
    >               #直接回车
    
    -----> Create config file '/etc/awstats/awstats.www.testme.com.conf'
     Config file /etc/awstats/awstats.www.testme.com.conf created.
    ...
    /usr/local/awstats/tools/awstats_updateall.pl now
    Press ENTER to continue... 
    ...
    Press ENTER to finish...




#### 2.3 修改www站点配置文件


2.3.1 LogFile参数

    
    #vim /etc/awstats/awstats.www.test.com.conf
    
    #LogFile="/usr/local/awstats/tools/logresolvemerge.pl /home/logs/www/*.log |"
    #LogFile="/home/logs/www/%YYYY-480%MM-480%DD-480.www.test.access.log"
    #LogFile="/home/logs/www/www.test.com.access.log"
    
    #如上,LogFile参数有多种写法，配置文件中已经解释很详细，主要如下
    #%YYYY%MM%DD: #此变量表示当前本地时间，比如今天是2015年11月8号，则此变量为"20151108"(不包括双引号，下同)
    #%YYYY-24%MM-24%DD-24: #当前时间的24小时之前，即昨天此刻，"20151107"
    #%YYYY-240%MM-240%DD-240: #240小时之前，即10天前此刻,"20151029"
    #%YYYY-240%MM-2400%DD-240.www.test.access.log: #会匹配20151029.www.test.access.log 这条日志
    #%YYYY-240%MM-2400%DD-240.*.log: #会匹配以"20151029"开头，以".log"结尾的所有日志
    #"/usr/local/awstats/tools/logresolvemerge.pl /home/logs/www/*.log |": 使用logresolvemerge.pl合并统计/home/logs/www下所有以".log"结尾的文件，最后的管道符号"|"不能少
    


2.3.2  LogFormat参数

不同的web日志具有不同格式，此参数定义awstats要读取的日志格式，默认情况下，LogFormat值为1

    
    LogFormat=1
    #关于具体情况，参考此配置文件中的注释


如果LogFormat值设为1，则awstats能够读取的日志格式为[NCSA Combined Log Format](http://publib.boulder.ibm.com/tividd/td/ITWSA/ITWSA_info45/en_US/HTML/guide/c-logs.html#combined), nginx默认的输出格式也符合NCSA Combined Log Format, 因此awstats能够读取nginx的输出日志，nginx输出日志格式如下：



    
      log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" $http_x_forwarded_for';


注意：nginx日志最后一个字段"$http_x_forwarded_for"并不会被awstats所使用, awstats会使用第一个字段"$remote_addr"作为终端用户IP进行统计，但是如果你的网站做的CDN, 或者使用代理访问, 那最后一个字段记录的IP才是真正终端用户的IP(此部分内容请参考nginx相关资料)，因此如果你的网站做过CDN，或者使用代理，那么就需要做下面的修改以使awstats统计到真正的用户IP，而非CDN节点或代理的IP

第一种方法，我们可以自定义awstats读取日志的格式，需要修改LogFormat参数



    
    LogFormat = "%otherquot - %host_r %time1 %methodurl %code %bytesd %refererquot %uaquot %host"
    #这一行告诉awstats传入的nignx日志中每一个字段的意义，第一个参数"%otherquot"告诉awstats不要读取此字段所代表的内容，即忽略"$remote_addr"的内容，nginx日志的最后一个字段"http_x_forwarded_for"才是awstats应该使用的"%host"


第二种方法不修改LogFormat参数，修改nginx日志记录格式，把"$http_x_forwarded_for"字段调到首字段，"$remote_addr"调到最后一字段，或者使用脚本把nginx日志中最后一个字段记录的用户真实IP和第一个字段对调





2.3.3 其它参数

    
    #日志类型，nginx日志使用W即可
    # Enter the log file type you want to analyze.
    # Possible values:
    #  W - For a web log file
    #  S - For a streaming log file
    #  M - For a mail log file
    #  F - For a ftp log file
    # Example: W
    # Default: W
    #
    LogType=W
    
    #数据文件保存目录
    DirData="/var/lib/awstats"
    
    #网页端支持刷新数据（静态页面不需要此参数，只有使用动态页面才需要）
    AllowToUpdateStatsFromBrowser=1
    
    #开启地理位置支持（显示IP所在国家，城市统计）
    LoadPlugin="geoip GEOIP_STANDARD  /usr/share/GeoIP/GeoIP.dat"
    LoadPlugin="geoip_city_maxmind GEOIP_STANDARD /usr/share/GeoIP/GeoIPCity.dat"
    #yum install GeoIP GeoIP-data GeoIP-devel perl-Geo-IP -y   #先安装软件包
    
    #解决中文乱码（去掉注释）
    LoadPlugin="decodeutfkeys"


2.3.4 生成其它站点配置文件



    
    cp /etc/awstats/awstats.www.test.com.conf /usr/local/awstats/wwwroot/cgi-bin/awstats.model.conf
    #/usr/local/awstats/wwwroot/cgi-bin/awstats.model.conf 此配置文件为awstats_configure.pl脚本生成配置文件时的模板


生成bbs站点配置文件

    
    cd /usr/local/awstats/tools/
    ./awstats_configure.pl
    ...
    #参考生成www步骤


由于使用了www的模板，因此上面步骤生成的bbs配置文件将会与前面www配置文件一样，只需要修改LogFile参数即可

    
    #cat /etc/awstats/awstats.bbs.test.com.conf | grep LogFile
    LogFile="/usr/local/awstats/tools/logresolvemerge.pl /home/logs/bbs/*.log |"




### 2.4 生成静态统计结果




#### 2.4.1 更新所有站点数据文件



    
    /usr/local/awstats/tools/awstats_updateall.pl now -awstatsprog=/usr/local/awstats/wwwroot/cgi-bin/awstats.pl
    #这个脚本会更新所有站点配置文件，默认从/etc/awstats目录寻找，如果你配置文件不在/etc/awstats目录，则需要指定目录




#### 2.4.2 另一种方法是直接使用awstats.pl脚本，逐个站点进行更新



    
    /usr/bin/perl /usr/local/awstats/wwwroot/cgi-bin/awstats.pl -update -config=www.test.com -configdir=/etc/awstats
    /usr/bin/perl /usr/local/awstats/wwwroot/cgi-bin/awstats.pl -update -config=bbs.test.com -configdir=/etc/awstats
    -configdir: 这个参数可有可无




#### 2.4.3 生成静态html统计页面


上面2.4.1或2.4.2都可以统计日志，把结果更新到数据文件，下面是根据数据文件生成最终可以用web访问的html

    
    /usr/local/awstats/tools/awstats_buildstaticpages.pl -config=www.test.com -lang=cn -awstatsprog=/usr/local/awstats/wwwroot/cgi-bin/awstats.pl -dir=/home/awstats/web/
    /usr/local/awstats/tools/awstats_buildstaticpages.pl -config=bbs.test.com -lang=cn -awstatsprog=/usr/local/awstats/wwwroot/cgi-bin/awstats.pl -dir=/home/awstats/web/
    #/home/awstats/web 为html存放目录




#### 2.4.4 配置web访问


我们使用nginx作为web服务器，下面配置一个虚拟主机，nginx配置虚拟主机方法请自行搜索

    
    #cat /usr/local/nginx/conf/vhosts/awstats.conf
    server {
        listen       8081;
        server_name  192.168.0.107;
        auth_basic "admin";
        auth_basic_user_file /usr/local/nginx/conf/vhosts/admin.pass;
        access_log  /home/logs/awstats.www.test.com.access.log;
        root /home/awstats/web;    #html存放目录
        index index.html;
    
        location /awstats/classes/ {
            alias /usr/local/awstats/wwwroot/classes/;
        }
    
        location /awstats/css/ {
            alias /usr/local/awstats/wwwroot/css/;
        }
    
        location /awstats/icon/ {
            alias /usr/local/awstats/wwwroot/icon/;
        }
    
        location /awstats-icon/ {
            alias /usr/local/awstats/wwwroot/icon/;
        }
    
        location /awstats/js/ {
            alias /usr/local/awstats/wwwroot/js/;
        }
    
        location /awstatsicons/ {
            alias /usr/local/awstats/wwwroot/icon/;
        }
     
        location ~ ^/icon/ 
    {
            root   /usr/local/awstats/wwwroot;
            index  index.html;
            access_log off;
            error_log off;
        }
    }
