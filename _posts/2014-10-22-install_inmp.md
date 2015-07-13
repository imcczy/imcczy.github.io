---
layout: post
title:  "Ubuntu/Debian安装lnmp"
date:   2014-10-22 23:20:40
categories: Server
---
lnmp是一系列的开源软件所组成的服务器环境安装包，取自linux，nginx，mysql和php四个单词的首字母。曾经试用过军哥的lnmp一键安装包，安装简单，配置也简单。不过我这种强迫症不喜欢现成的，说实话在这块鼓捣了好久。因为我一直搞不明白nginx的配置。

**安装mysql:**

```bash
sudo apt-get install mysql-server
```

安装过程中会要求输入root密码,输入即可。安装完成后执行

```bash
sudo mysql_secure_installation
```bash

按提示操作即可。建议禁止root远程登入。

**安装nginx:**

Nginx ("engine x") 是一个高性能的 HTTP 和 反向代理 服务器，由一家俄罗斯公司开发。具体请百度百科。

```bash
sudo apt-get install nginx
```bash

**安装php:**

```bash
sudo apt-get install php5-fpm php5-mysql
```

至此所有安装完成。

**启动nginx:**

```bash
sudo service nginx start
```

这一步若提示[emerg] bind() to 0.0.0.0:80 failed (98:address already in use ),一般是80端口被apache占用所致，杀死所有apache进程：

```bash
sudo killall apache2
```

关闭apache的开机启动：

```bash
sudo update-rc.d -f apache2 remove
```

在浏览器里访问你的IP,出现Welcome to Nginx，则说明nginx启动成功。

**nginx配置:**

nginx的配置文件在/etc/nginx/nginx.conf，具体设置请参考[Nginx战斗准备 —— 优化指南](http://blog.jobbole.com/51861/)。注意到这个其中的两行：

```bash
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/;
```

nginx下的一个虚拟主机就相当于一个网站，每个虚拟主机的配置文件保存在上面两条路径。为了便于管理，我都是在/etc/nginx/conf.d/下新建conf配置文件。打开nginx的默认配置文件，修改使其支持php：

```bash
sudo nano /etc/nginx/sites-enable/default
```

找到server模块，按下修改：

```bash
root /usr/share/nginx/www;#网站根目录
index index.php index.html index.htm;
server_name your_IP_address;
```

往下找到location ~ .php$ ，去掉部分注释：

```bash
location ~ .php$ {
	fastcgi_split_path_info ^(.+.php)(/.+)$;
#	# NOTE: You should have "cgi.fix_pathinfo = 0;" in php.ini
#
#	# With php5-cgi alone:
#	fastcgi_pass 127.0.0.1:9000;
#	# With php5-fpm:
	fastcgi_pass unix:/var/run/php5-fpm.sock;
	fastcgi_index index.php;
	include fastcgi_params;
}
```

保存修改退出，并重启nginx：

```bash
sudo service nginx restart
```

新建一个phpinfo.php:

```bash
sudo nano /usr/share/nginx/www/phpinfo.php
```

输入：

```bash
<;?php phpinfo(); ?>;
```

保存退出，浏览器访问ipaddress/phpinfo.php，能正常显示php5-fpm的配置信息说明lnmp环境就搭好了。

**建议安装phpmyadmin:**

```bash
sudo apt-get install phpmyadmin
sudo ln -s /usr/share/phpmyadmin/ /usr/share/nginx/www 
sudo service nginx reatsrt
```

安装过程中会提示输入mysql密码和设置root登录密码，安装完成访问ipaddress/phpmyadmin，进行管理数据库。

**新建一个虚拟主机（添加网站）:**

复制/etc/nginx/sites-enable/default到/etc/nginx/conf.d/*.conf：

```bash
sudo cp /etc/nginx/sites-enable/default /etc/nginx/conf.d/*.conf
```

视情况修改IP和网站根目录，可参考[http://www.nginx.cn/doc/example/virtualhost.html](http://www.nginx.cn/doc/example/virtualhost.html)
[http://www.nginx.cn/doc/example/virtualhost.html](http://www.nginx.cn/doc/example/virtualhost.html)

参考文章:[Debian 7 使用Dotdeb源安装LNMP服务器](http://dearroy.com/linux/2013/06/20/install-lnmp-on-debian-7.html)
