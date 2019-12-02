---
title: php+apache站点迁移至docker实战
date: 2018-01-03 15:27:27
tags: docker
categories: [Docker]
---

最近又换服务器了，把原来的 centos 6.9 换成了 ubuntu。centos 毕竟大而全，需要手动配置的地方太多，而 ubuntu 就比较适合个人使用，少折腾。但是原服务器上的个人站是用的 typecho，php+apache+mysql 环境，免不了又要迁移。于是就想到了 [docker](https://www.docker.com/)

docker 的介绍就不多说了，其优点是轻量，而且可以免于频繁的搭建环境。首先安装 docker，可以见官网 [安装 dock](https://docs.docker.com/engine/installation/#time-based-release-schedule)。我安装的是 CE 版，docker 支持主流的 linux 和 macos 还有 windows。

安装的时候会发现，docker 官方的源超慢... 所以在安装完后第一件事就是换成国内的源，目前我用的是 [Daocloud 加速](http://www.daocloud.io/mirror)。

然后可以跑一下 hello-world `sudo docker run hello-world`

如果没问题的话会有提示安装完成，接下来下一个 lamp 的镜像，可以运行 `sudo docker search lamp` 搜索镜像，我用了 star 数最高的 `linode/lamp` ，可以运行 `sudo docker pull linode/lamp` 下载此镜像。

> [文章参考](http://blog.csdn.net/MasonQAQ/article/details/78048112?locationNum=5&fps=1)

此镜像内已集成 mysql、apache2、php5.5 的环境，所以可以通过文件映射的方式将宿主机的文件挂载到 docker 镜像内运行。以网站目录`mywebsite`为例，以下是我写好的 sh 脚本，放在 mywebsite 根目录下：

```bash
#!/bin/bash
pwd=`pwd`
echo 'enter a port:'
read port
docker run -p $port:80 -v $pwd/mysql:/var/lib/mysql -v $pwd/site:/var/www/site -v $pwd/apache2:/etc/apache2/sites-enabled -v $pwd/apache2:/etc/apache2/sites-available -v $pwd/apache2.conf:/etc/apache2/apache2.conf -t -i linode/lamp /bin/bash
```

mywebsite 目录结构为：

mywebsite</br>
├─cmd.sh //shell 脚本，防止忘了命令...</br>
├─site //网站目录</br>
│ ├─log //日志目录</br>
│ └─public_html //网站目录</br>
│ └─index.html</br>
├─mysql //mysql 数据库文件目录</br>
├─apache2 //apache2 网站配置</br>
├─apache2.conf //apache2 全局配置</br>
└─php.ini //php 全局配置</br>

```
docker run : 运行一个镜像，此处为 linode/lamp
-t -i linode/lamp /bin/bash : 使用 linode/lamp 并且打开 bash命令
-p : 将宿主机的端口映射至容器的端口
-v : 将宿主机的目录或文件映射至容器，可以多次使用
```

上面的脚本里接收一个输入的端口映射至容器的 80 端口，并且将 mysql 文件和 apache 配置以及网站目录映射至容器。如果有需求修改其它配置比如 php 或者 mysql 也可以按照此方法映射至容器

启用: `sudo ./cmd.sh`

启用成功后会进入容器里并打开 bash，首先要启动 apache 和 mysql 服务

```bash
service apache2 start
service mysql start
```

> 我在启动 mysql 的时候报错，经排查发现是权限问题，由于 mysql 数据库文件是映射进来的，所以要修改权限。运行如下命令添加权限

```bash
chown -R mysql:mysql /var/lib/mysql/
chmod -R 755 /var/lib/mysql/
```

> apache2 运行时可以报错 `[core:warn] [pid 14655] AH00111: Config variable ${APACHE_LOCK_DIR} is not defined` ，可以运行 `source /etc/apache2/envvars` 解决，详见 [apache2](http://serverfault.com/questions/558283/apache2-config-variable-is-not-defined)

接下来管理 mysql，修改默认密码

```bash
mysql -uroot -p
#默认密码 : Admin2015

#修改root可远程登录：可以不用
mysql>use mysql;
mysql>update user set host = '%' where user = 'root' and host='127.0.0.1';

#修改密码：由于我是外部挂载的mysql文件，所以密码已经修改，这步没有运行
mysql>update user set password=password("your_password");

#刷新权限
FLUSH PRIVILEGES;
exit;
```

由于 apache2 默认没有安装 mysql 的 pdo，所以要安装一下

```bash
apt-get update
apt-get install php5-mysql -y
apt-get install php5-gd -y

#然后重启apache2:
service apache2 restart
```

然后按 `ctrl+p,ctrl+q` 退出容器，想要再进入可以在宿主机运行 `sudo docker ps` 获取容器的 id，然后运行 `sudo docker attach [ID]` 重新进入容器

如果 apache2 配置没错的话在宿主机访问 localhost:8888 就能看到容器里的页面了

关于 apache2 启动 rewrite，apache 默认并没有启用 rewrite，可以在 apache2.conf 里加上一句`LoadModule rewrite_module /usr/lib/apache2/modules/mod_rewrite.so` 启用 rewrite 模块，然后配置：

```conf
<Directory /var/www/>
	Options Indexes FollowSymLinks
	AllowOverride All
	Require all granted
</Directory>
```

中把 AllowOverride 改为 All，之后就能在网站目录下放一个 .htaccess 文件来设置 rewrite 规则了。本例中为 typecho 启用的 rewrite 文件例子：

```
# BEGIN
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ /index.php/$1 [L]
</IfModule>
# END
```

还有 https~就懒得说了，基本是宿主机的 nginx 反向代理配置

最后，感觉用别人的镜像还是比较麻烦的，还是自己搭个镜像比较放心
