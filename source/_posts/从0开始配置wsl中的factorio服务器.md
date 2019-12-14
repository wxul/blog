---
title: 从0开始配置wsl中的factorio服务器
date: 2019-12-14 17:28:40
tags: [wsl]
categories: [wsl]
---

首先，搭服务器肯定是为了联机。其次，异星工厂无头版服务器只有linux下能运行。家里多出来的笔记本装的是windows系统而且还需要当打印服务器，所以只能使用wsl子系统运行异星工厂服务器了。

<!--more-->

## 1. 启用WSL

首先win10最好是更新至最新版，那启用wsl直接就是wsl2了。wsl升级不在此范围内~

启用方式网上有很多教程，稍微总结一下步骤

管理员打开powershell并运行
``` powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

进入microsoft store搜索linux选择发行版下载，我选择的是ubuntu

> 我这里选择安装时无响应，打开cmd输入 `wsreset` 清空store缓存后解决

安装完毕后可以在任意命令行输入 `wsl -l` 查看已安装的子系统

输入 `wsl` 进入子系统，首次进入会设置用户名以及密码。之后就可以通过此命令进入子系统了

## 2. 配置

首先更换源
``` bash
cd /etc/apt/
# 备份
sudo cp sources.list sources.list.bak
# vim
sudo vim sources.list
```
然后通过命令替换成阿里的源
``` vim
:%s/security.ubuntu/mirrors.aliyun/g
:%s/archive.ubuntu/mirrors.aliyun/g
```
保存退出后可以运行以下命令升级
``` bash
sudo apt-get update
sudo apt-get upgrade
```

然后配置ssh
``` bash
sudo vim /etc/ssh/sshd_config
```
需要注意的几个配置项有：

`PORT` : 一般为22

`PasswordAuthentication` : 是否启用密码登录

接着配置frp，用于内网穿透，可以把ssh和工厂服务器暴露至公网环境，需要先有一个公网的服务器，具体配置可以参考 [frp](https://github.com/fatedier/frp)。注意异星工厂是基于UDP协议

最后是配置开机启动了，需要开机就启动wsl并且运行ssl和frp

现在wsl里创建初始化脚本 `sudo vim /etc/init.wsl`:
``` bash
#! /bin/sh
# 启用ssh
/etc/init.d/ssh $1
# 启用守护进程
/etc/init.d/supervisor $1
```

然后在windows里命令行输入 `shell:startup` 打开启动文件目录，创建文件 `wsl-startup.vbs`，输入以下内容：
``` vbs
Set ws = WScript.CreateObject("WScript.Shell")
ws.run "wsl -d Ubuntu -u root /etc/init.wsl start", vbhide
```
这样在windows开机之后会自动执行 `/etc/init.wsl start`，然后启用ssh和supervisor

接下来配置守护进程启动frp：
``` bash
cd /etc/supervisor/conf.d
vim frpc.conf
```
输入以下内容并保存：
``` conf
[program:frpc]
command=/home/admin/frp/frpc -c /home/admin/frp/frpc.ini # 此处放自己的frp启动命令
directory=/home/admin   # 执行目录
user=admin # 执行用户
stopsignal=INT
autostart=true # 自动启动
autorestart=true # 自动重启
startsecs=3 # 重启间隔
stderr_logfile=/var/log/frpc.err.log # 日志
stdout_logfile=/var/log/frpc.out.log # 日志
```

自动启动就配置完毕了

## 3. 异星工厂配置

先创建个目录放游戏相关的文件 `mkdir ~/factorio-server && cd factorio-server`，然后创建个下载&更新脚本 `vim update.sh`:
``` bash
#! /bin/bash

echo "begin download..."
rm -rf linux64
# 下载最新版
wget https://www.factorio.com/get-download/stable/headless/linux64 --no-check-certificate
# 解压
tar xvJf linux64
```

工厂相关的配置都在 `factorio/data/` 下，建立服务器主要的是 `server-settings.example.json`
``` bash
# 复制一份进行修改
cp server-settings.example.json server-settings.json
```
具体的配置作用可以参考文件内注释

然后返回 factorio-server 目录创建启动脚本 `vim start.sh`:
``` bash
#! /bin/bash
fpwd=$pwd/factorio
# map.zip为地图文件，需要生成后上传过去
$fpwd/bin/x64/factorio --port 33222 --server-settings $fpwd/data/server-settings.json --start-server $fpwd/saves/map.zip
```

以上两个脚本文件都要记得改执行权限 `chmod +x ./*.sh`

然后就可以通过screen在后台启用服务了 `screen ./start.sh`

## TODO