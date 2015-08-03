---
layout: post
title:  "利用Docker构建jekyll自动更新镜像"
date:   2015-08-03 23:33:40
categories: Linux
---
#利用Docker构建jekyll自动更新镜像

jekyll博客可以托管在github page上，相当方便。但是github时不时被攻击河蟹，利用github webhook，我们便可将博客同步到其他服务器上。之所以使用docker，一是方便部署，二是可以将博客托管在像Daocloud这样的docker容器平台上。

github webhook的大致原理是，仓库发生指定事件（比如push）时，github会向指定的服务器发送一个post请求，服务器收到请求后再执行相应的指令。实现webhook服务器的方法非常多，比如官方推荐的[Sinatra](http://www.sinatrarb.com/)，其他python和nodejs等都可以实现。这里为了方便构建docker镜像，webhook服务使用nginx+shell+lua，更方便一点我直接使用[openresty](https://openresty.org/)。

使用官方的ruby镜像总有意想不到的情况，所以我们从头开始构建，分为jekyll的安装、openrestry的安装和配置nginx server。

**Dockerfile以及所有的文件在[https://github.com/imcczy/jekyll_docker](https://github.com/imcczy/jekyll_docker)
**

###安装依赖

jekyll的额外依赖有nodejs和python2，我在后面使用了supervisior守护nginx进程，可以不用单独安装python。

```bash
RUN apt-get update && apt-get install -y supervisor  git curl nodejs \
    libreadline-dev libncurses5-dev libpcre3-dev libssl-dev perl wget && \
    apt-get clean
```

###安装jekyll

jekyll依赖于ruby，ruby的安装参考Ruby China社区的[ wiki](https://ruby-china.org/wiki/install_ruby_guide)，需要注意的几点：
1 安装前需要导入ruby的公钥

```bash
RUN　gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
```

2 docker构建镜像的时候默认/bin/sh，载入rvm环境和使用gem命令需要使用bash，将RUN命令改成

```bash
ENV RUBY_VERSION 2.2.1

RUN  /bin/bash -l -c "source /etc/profile.d/rvm.sh" &&  /bin/bash -l -c "rvm install ${RUBY_VERSION}" && \
     /bin/bash -l -c "rvm ${RUBY_VERSION} --default"
```

注意这里和wiki上的"source ~/.rvm/scripts/rvm"不同。

安装jekyll只需：

```bash
RUN　/bin/bash -l -c "gem install jekyll --no-rdoc --no-ri"
```

###安装openresty

[openresty](https://openresty.org/)是一个国人维护的nginx bundle，集成了lua,需要下载源码编译安装。

```bash
ENV OPENRESTY_VERSION 1.7.10.2

RUN wget http://openresty.org/download/ngx_openresty-${OPENRESTY_VERSION}.tar.gz && \
    tar -xzvf ngx_openresty-*.tar.gz && \
    rm -f ngx_openresty-*.tar.gz && \
    cd /ngx_openresty-${OPENRESTY_VERSION} && \
    cd bundle/LuaJIT-* && \
    make clean && make && make install && \
    ln -sf luajit-2.1.0-alpha /usr/local/bin/luajit && \
    cd /ngx_openresty-${OPENRESTY_VERSION} && \
    ./configure --prefix=/usr/servers --with-luajit && \
    make && make install && make clean && \
    cd / && \
    rm -rf ngx_openresty-*

```

上面编译安装了luajit，并将openresty安装在/usr/servsers，nginx可执行文件路径为

```bash
/usr/servers/nginx/sbin/nginx 
```

###配置nginx

**1 nginx配置文件修改**

nginx的配置文件在/usr/servers/nginx/conf/nginx.conf，需要修改两处

1 为nginx指定一个用户，由于Docker里是root环境，这里我直接指定了root。

2 在配置文件的http段增加：

```bash
include /usr/servers/conf/*.conf;
```

我们将nginx server配置存于/usr/servers/conf/*.conf。

方便起见直接将原配置文件覆盖

```bash
RUN rm /usr/servers/nginx/conf/nginx.conf
ADD nginx.conf /usr/servers/nginx/conf/nginx.conf
```

**2 添加一个server**

server配置如下，这里webhook的post地址为yourdomian/build

```bash
#jekyll.conf
server {

    listen       80;
    server_name  _;
    #网站根目录
    root /usr/servers/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
    error_page 404 /404.html;

    lua_code_cache on;
    #webhook的post地址，交由lua处理
    location /build {
        content_by_lua_file /usr/servers/conf/jekyll.lua;
    }
}
```

jekyll.lua，前面用于key验证，后面执行build.sh

```bash
local signature = ngx.req.get_headers()["X-Hub-Signature"]
local key = "yourkey"
if signature == nil then
    return ngx.exit(404)
end
ngx.req.read_body()
local t = {}
for k, v in string.gmatch(signature, "(%w+)=(%w+)") do
    t[k] = v
end
local str = require "resty.string"
local digest = ngx.hmac_sha1(key, ngx.req.get_body_data())
if not str.to_hex(digest) == t["sha1"] then
    return ngx.exit(404)
end
os.execute("bash /usr/servers/conf/build.sh");
ngx.say("OK")
ngx.exit(200)
```

build.sh调用jekyll，生成网站，这里还可以优化下，不用每次都git clone。

```bash
#!/bin/bash

WORKDIR=/usr/servers
GITROOT=$WORKDIR/site
WBEROOT=$WORKDIR/html

# Update Git Repo
rm -r $GITROOT
git clone "github page repository HTTPS clone URL" $GITROOT

# Build Site
rm -r $WBEROOT
cd $GITROOT
source /etc/profile.d/rvm.sh
jekyll  build --destination $WBEROOT
```

**3 使用supervisior守护nginx进程**

supervisord.conf

```bash
[supervisord]
nodaemon=true

[program:nginx]
command=/usr/servers/nginx/sbin/nginx -g "daemon off;"
```

将所有配置文件添加到镜像，给build.sh添加执行权限。

```bash
RUN mkdir /usr/servers/conf 

ADD jekyll.conf /usr/servers/conf/jekyll.conf
ADD jekyll.lua /usr/servers/conf/jekyll.lua
ADD build.sh /usr/servers/conf/build.sh

RUN chmod +x /usr/servers/conf/build.sh

ADD supervisord.conf /etc/supervisor/conf.d/supervisord.conf
```

**最后**
最后开放镜像80端口,启动运行supervisor

```bash
EXPOSE 80

CMD supervisord -c /etc/supervisor/conf.d/supervisord.conf
```

Ps：jekyll.lua验证key部分可以不要，相应的在github上设置仓库的webhook也不需要设置key

参考
[利用github webhook自动更新hexo](http://blog.xmost.com/2015/06/use-github-webhooks-to-deploy-hexo/)
