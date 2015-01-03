---
layout: post
title:  "docker学习笔记"
date:   2014-10-23 23:20:40
categories: Server
---
感觉docker真是很好玩啊，要是玩得转，部署任何应用很方便。这几天看了[The docker book](http://www.dockerbook.com/)。把博客[blog.imcczy.com](blog.imcczy.com)部署到了docker容器上。

一些docker的参数命令：
    
    --name 指定容器名字。
    -d 后台运行
    docker ps 显示容器
    docker log containername 显示容器的日志
    docker top containername 显示进程
    docker inspect containername 显示容器全部信息 -f=--form     at，格式化输出
    -v 从本地目录挂载一个volume到容器 例：-v $PWD/website:/var/www/html/website。挂载当前目录下的website到容器的/var/www/html/website
    --volumes-from:name 
    --link name:alias 容器间连接，容器名:别名。
    --rm 

###把typecho部署到docker容器上
看了官方wordpress镜像的Dockerfile，发现用的多是lamp，可是apache不熟，而且网站目录都是在image里内建的，不适合typecho。只好自己动手了。用的还是nginx，主要用到两个docker镜像：

> mysql数据库镜像
nginx-php镜像

nginx-php里设置一个VOLUME，把typecho目录挂上去，便于修改。

    sudo docker run -d --name dbsql -p 3306:3306 tutum/mysql #docker los 查看密码，修改密码
    sudo docker run -d --name typecho -v $PWD/typecho:/app -p 80:80 imcczy/nginx-php

用--link链接数据库应该也可以，没成功，以后再试。
Dockerfile在[github](https://github.com/imcczy/mydocker)
