---
title: gitlab(7.9)升级到8.0.1
author: opengers
layout: post
permalink: /linux/gitlab7-update-to-8/
categories: linux
tags:
  - gitlab
  - update
format: quote
---

# gitlab8.0更新说明

GitLab 8.0 现在完全集成了持续集成工具 (GitLab CI) ，此外还完全重写了 UI，节省了至少 50% 的磁盘空间。更快的合并，内置持续集成（CI）到 GitLab 本身，提高了界面和导航，以及“通过电子邮件回复”功能，它可以使用户通过移动设备就能够对某个问题上迅速发表评论，或者合并请求。   
- GitLab 8.0 主要改进：
- 更好的 HTTP 支持
- 邮件快速回复
- Gmail 快速打开
- 改善文件上传功能
- 改进 Mattermost
- Web hooks 的 SSL 认证   

以上摘自[gitlab-8-0](http://www.oschina.net/news/66441/gitlab-8-0), 更新请查看gitlab官网https://github.com/gitlabhq/gitlabhq/blob/master/CHANGELOG

# gitlab7.9环境

### gitlab7.9安装说明  

系统为centos6.7，gitlab7.9为源码安装，默认位置为/home/git/gitlab，我安装位置：/usr/local/git/gitlab，因此请注意你的gitlab安装位置是否使用默认，相应更改其配置  

### 升级准备

由于gitlab使用git用户管理，因此我们应当以git身份执行相关配置，升级前请检查gitlab运行正常  

``` shell    
sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production
```

### 备份gitlab

升级有风险，因此先做好备份    

``` shell
sudo -u git -H bundle exec rake gitlab:backup:create RAILS_ENV=production
```

如果gitlab运行在虚拟机中，最好给虚拟机做快照，以便更新失败时恢复   

### 停止gitlab

升级前先停止gitlab运行  

    service gitlab stop


### 检查gitlab相关组件版本

``` shell
#gitlab8.0组件版本要求
Redis version >= 2.4.0
Ruby version >= 2.1.0
Git version >= 1.7.10
GitLab Shell version >= 2.6.5
```

组件版本要求如下，如果版本过低请升级，升级参考上面步骤    

### 升级git版本

``` shell
#检查git版本
git --version
#升级git
cd /opt/
curl -L --progress https://www.kernel.org/pub/software/scm/git/git-2.4.3.tar.gz | tar xz
./configure
make prefix=/usr/local all -j 4
make prefix=/usr/local install
```

### 升级ruby版本  

``` shell    
#下载
wget ftp://ftp.ruby-lang.org/pub/ruby/2.1/ruby-2.1.6.tar.gz
#安装
tar xvf ruby-2.1.6.tar.gz
cd ruby-2.1.6
./configure --disable-install-rdoc
make -j 4
make install
#配置
gem install bundler --no-ri --no-rdoc
#如果/usr/bin/下存在ruby链接，先删除
ln -s /usr/local/bin/ruby /usr/bin/ruby
ln -s /usr/local/bin/gem /usr/bin/gem
ln -s /usr/local/bin/bundle /usr/bin/bundle
```

# 升级gitlab到8.0.1

### 使用git命令升级gitlab到8.0.1  

``` shell
#进入gitlab安装目录
cd /usr/local/git/gitlab
sudo -u git -H git fetch --all
sudo -u git -H git checkout -- Gemfile.lock db/schema.rb
#获取最新版本，保存在LATEST_TAG变量中
LATEST_TAG=$(git describe --tags `git rev-list --tags --max-count=1`)
#commit是为了提交更改，不然无法cheout到最新版本
sudo -u git git commit -a
sudo -u git -H git checkout $LATEST_TAG -b $LATEST_TAG
#验证当前版本
sudo -u git git branch -l
#注意：checkout到gitlab8.0.1之后，/usr/local/git/gitlab下的文件为8.0.1版本，但是config/gitlab.yml,config/database.yml等配置文件仍位旧版本7.9的配置文件，后面会更新这些文件
```

### 升级gitlab-shell  

``` shell    
#进入gitlab-shell主目录
cd /usr/local/git/gitlab-shell/
sudo -u git -H git fetch
#注意下面路径/usr/local/git/gitlab为gitlab安装目录，如果你的位于/home/git/gitlab请更改下面命令
sudo -u git -H git checkout v`cat /usr/local/git/gitlab/GITLAB_SHELL_VERSION` -b v`cat /usr/local/git/gitlab/GITLAB_SHELL_VERSION`
#验证gitlab版本
sudo -u git git branch -l
```

### 安装gitlab-git-http-server

下载go 1.5版本  

``` shell
#下载go 1.5版本
cd /opt
curl -O --progress https://storage.googleapis.com/golang/go1.5.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.5.linux-amd64.tar.gz
ln -sf /usr/local/go/bin/{go,godoc,gofmt} /usr/local/bin/
ln -s /usr/local/bin/go /usr/bin/
```

安装gitlab-git-http-server    

``` shell
cd /usr/local/git
sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-git-http-server.git
cd gitlab-git-http-server/
sudo -u git -H make
```

### 安装ruby依赖包    

``` shell
#由于ruby官网链接过慢，因此更改ruby源
cd /usr/local/git/gitlab
#修改Gemfile文件
source "https://rubygems.org/" 改为 source "http://ruby.taobao.org/"
#修改Gemfile.lock文件
remote: https://rubygems.org/ 改为 remote: http://ruby.taobao.org/
#执行安装（使用mysql，如果你使用postgress，请更改下面postgress为mysql）
yum install cmake
sudo -u git -H bundle install --without postgres development test aws --deployment
```
  
### 迁移数据库  

安装依赖
    
    yum install nodejs

建立表结构，如果不导入表结构，迁移数据库时会提示找不到表ci_builds  

``` shell
cat gitlab_builds.sql

DROP TABLE IF EXISTS `ci_builds`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `ci_builds` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`project_id` int(11) DEFAULT NULL,
`status` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
`finished_at` datetime DEFAULT NULL,
`trace` longtext COLLATE utf8_unicode_ci,
`created_at` datetime DEFAULT NULL,
`updated_at` datetime DEFAULT NULL,
`started_at` datetime DEFAULT NULL,
`runner_id` int(11) DEFAULT NULL,
`coverage` float DEFAULT NULL,
`commit_id` int(11) DEFAULT NULL,
`commands` text COLLATE utf8_unicode_ci,
`job_id` int(11) DEFAULT NULL,
`name` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
`deploy` tinyint(1) DEFAULT '0',
`options` text COLLATE utf8_unicode_ci,
`allow_failure` tinyint(1) NOT NULL DEFAULT '0',
`stage` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
`trigger_request_id` int(11) DEFAULT NULL,
PRIMARY KEY (`id`),
KEY `index_ci_builds_on_commit_id` (`commit_id`) USING BTREE,
KEY `index_ci_builds_on_project_id_and_commit_id` (`project_id`,`commit_id`) USING BTREE,
KEY `index_ci_builds_on_project_id` (`project_id`) USING BTREE,
KEY `index_ci_builds_on_runner_id` (`runner_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;
/*!40101 SET character_set_client = @saved_cs_client */;DROP TABLE IF EXISTS `ci_commits`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `ci_commits` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`project_id` int(11) DEFAULT NULL,
`ref` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
`sha` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
`before_sha` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
`push_data` mediumtext COLLATE utf8_unicode_ci,
`created_at` datetime DEFAULT NULL,
`updated_at` datetime DEFAULT NULL,
`tag` tinyint(1) DEFAULT '0',
`yaml_errors` text COLLATE utf8_unicode_ci,
`committed_at` datetime DEFAULT NULL,
PRIMARY KEY (`id`),
KEY `index_ci_commits_on_project_id_and_committed_at_and_id` (`project_id`,`committed_at`,`id`) USING BTREE,
KEY `index_ci_commits_on_project_id_and_committed_at` (`project_id`,`committed_at`) USING BTREE,
KEY `index_ci_commits_on_project_id_and_sha` (`project_id`,`sha`) USING BTREE,
KEY `index_ci_commits_on_project_id` (`project_id`) USING BTREE,
KEY `index_ci_commits_on_sha` (`sha`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;
/*!40101 SET character_set_client = @saved_cs_client */;
```

导入表结构  

``` shell
mysql -uroot -p gitlab_production < gitlab_builds.sql
```

执行迁移

    sudo -u git -H bundle exec rake db:migrate RAILS_ENV=production

重建缓存

    sudo -u git -H bundle exec rake assets:clean assets:precompile cache:clear RAILS_ENV=production

### 配置更新

上面说过，虽然用git拉取了gitlab最新版本（8.0.1）但相关配置文件仍为7.9版本，因此需要更新   

**更新启动文件**  

``` shell
#主目录
cd /usr/local/git/gitlab
#更新启动程序
cp lib/support/init.d/gitlab /etc/init.d/gitlab
#更新gitlab配置
cp lib/support/init.d/gitlab.default.example /etc/default/gitlab
#需要修改/etc/default/gitlab文件，因为我们的gitlab安装在/usr/local/git下，并非默认目录/home/git
sed -i "s/home/usr\/local/g" /etc/default/gitlab
```

**更新日志**  

    cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab

**更新nginx配置文件**  

``` shell
#更新gitlab的nginx配置
cp lib/support/nginx/gitlab /etc/nginx/conf.d/gitlab.conf
#同样，gitlab并非安装在默认目录，因此需要修改配置
sed -i "s/home/usr\/local/g" /etc/nginx/conf.d/gitlab.conf
```

**更新gitlab.yml**  

``` shell
cp config/gitlab.yml.example config/gitlab.yml
#当然，需要更改gitlab.yml中的配置，可以参考旧版本配置文件，新配置文件更改了一些结构，但大部分变量都沿用旧版本的
```

**更新database.yml**  

``` shell
cp config/database.yml.mysql config/database.yml
#也需要更新配置
```

**更新gitlab-shell配置文件**

``` shell
cd /usr/local/git/gitlab-shell/
cp config.yml.example config.yml
#相关配置也要更新
```

# 测试gitlab8.0.1

### 启动gitlab

    /etc/init.d/gitlab start

### 变量检查  

    sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production

### 检查所有模块是否运行正常

    sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production

参考：  

><small>https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/update/7.14-to-8.0.md</small>
