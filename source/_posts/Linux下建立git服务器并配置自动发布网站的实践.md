---
title: Linux下建立git服务器并配置自动发布网站的实践
tags: ['linux','git','nginx']
date: 2016-12-11 17:40:00
categories: [运维]
---

本人使用阿里云的CentOS,版本为 Linux release 7.2

## 1\. 安装

### 1\. git

直接yum安装,没啥好说的

### 2\. nginx

需要配置一下,在 `/etc/yum.repos.d/` 目录下新建文件nginx.repo
输入内容：

``` conf
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
```

之后就能使用yum安装nginx了

* * *

## 2\. 配置

### 1\. git

首先添加git用户和组 `$ user add git` ,再建立git文件夹比如/git
添加公钥,客户端执行 `$ ssh-keygen -t rsa -C “youremail@email.com”` 生成秘钥,找到生成的id_rsa.pub公钥文件比将内容复制到服务端的git公钥文件 `/home/git/.ssh/authorized_keys` 中
在/git中初始化一个git仓库 `$ git init --bare project.git`
更改所有者 `$ chown -R git project.git`
客户端可以使用 `$ git clone git@yourdomain:/git/project.git` 来建立本地仓库
如果有本地仓库,可以使用 `$ git remote add origin git@yourdomain:/git/project.git` 添加remote建立联系

### 2\. nginx

建立网站文件夹比如/www/project
假设nginx安装在/etc/nginx下,那么 nginx.conf 就是配置文件,配置文件中有一行 `include /etc/nginx/conf.d/*.conf;`
表示会把 /etc/nginx/conf.d/ 目录下的任意以 .conf结尾的配置文件引入进来
那么进入 /etc/nginx/conf.d/ 下并建立文件 git.conf 输入简单配置
``` conf
server {
        server_name yordomain.com;
        listen 80;

        location / {
                root /www/project;
                index index.html;
        }
}
```

### 3\. hook!

git提交到服务器的是版本控制文件,如果需要自动发布还需要源文件,这可以通过git的hook来实现
首先进入git仓库的hook `$ cd /git/project.git/hooks`
创建 post-receive 文件,输入内容：
``` sh
#!/bin/sh
GIT_WORK_TREE=/www/project git checkout -f
```

保存,并且修改权限 `$ chmod +x post-receive`,同时确保/www/project针对git也有写权限
表示每次客户端push的时候就将源文件复制到相应目录下,通过域名就能直接访问了