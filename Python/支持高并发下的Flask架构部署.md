@[TOC](支持高并发下的Flask架构部署)
## 1. Why Flask+Gunicorn+Nginx
>Flask+Gunicorn+Nginx是最常用的Flask部署方案，大家深究过为何用这样的搭配么？

### 1.1 Why?
Flask 是一个web框架，而非web server，直接用Flask拉起的web服务仅限于开发环境使用，生产环境不够稳定，也无法承受大量请求的并发，在生茶环境下需要使用服务器软件来处理各种请求，如Gunicorn、 Nginx或Apache，而Gunicorn+Nginx的搭配，好处多多，一方面基于Nginx转发Gunicorn服务，在生产环境下能补充Gunicorn服务在某些情况下的不足，另一方面，如果做一个Web网站，除了服务外，还有很多静态文件需要被托管，这是Nginx的强项，也是Gunicorn不适合做的事情。所以，基于Flask开发的网站，部署时用Gunicorn和Nginx，是一个很好的选择。

### 1.2 Anything More?
#### 1、为什么需要Nginx转发Gunicorn服务？
Nginx功能强大，使用Nginx有诸多好处，但用Nginx转发Gunicorn服务，重点是解决“慢客户端行为”给服务器带来的性能降低问题；另外，在互联网上部署HTTP服务时，还要考虑的“快客户端响应”、SSL处理和高并发等问题，而这些问题在Nginx上一并能搞定，所以在Gunicorn服务之上加一层Nginx反向代理，是个一举多得的部署方案。

#### 2、为什么会有“慢客户端行为”带来的服务性能降低问题？
服务器和客户端的通信，我们简略的分为三个部分：request，request handling，和response，即客户端向服务器发起请求，服务器端响应并处理请求，和将请求结果返回客户端，这三个过程。

通常，request handling这部分即服务端的计算，拼的是服务器的性能，处理是比较高效和稳定的，而request和response部分，影响因素比较多，如果这三个过程放到同一个进程中同步处理，如果request和response部分耗时比较多，会使计算资源被占据并无法及时释放，导致计算资源无法有效利用，降低服务器的处理能力。

上述“慢客户端行为”，指的就是request（或response）部分耗时比较多的情况，Gunicorn恰好会把上面三个过程放到同一个进程中，当出现“慢客户端行为”时，效率很低：

Gunicorn 是一个pre-forking的软件，这类软件对低延迟的通信，如负载均衡或服务间的互相通信，是非常有效的。但pre-forking系统的不足是，每个通信都会独占一个进程，当向服务器发出的请求多于服务器可用的进程时，由于服务器端没有更多进程响应新的请求，其响应效率会降低。

对于Web网站或服务而言，由于request和response延时是不可控的，我们需要在考虑处理高延迟客户端请求的情况。这些请求会占据服务器端的进程。当慢客户端直接与服务通信时，由于慢客户端请求会占据进程，可用于处理新请求的进程就会减少，如果有很多慢客户端请求把所有进程都占据后，新的请求只能等待有进程被释放掉后，得到响应。另外，如果应用希望有更高的并发，服务器与客户端的通信要更高效，异步的通信会比同步的通信更有效。

Nginx这类异步的服务器软件擅长用很少的内存和cpu开销来处理大量的请求。由于他们擅长于同时处理大量客户端请求，所以慢客户端请求对他们影响不大。就Nginx而言，现在一般的服务器硬件条件下，同时处理上万个请求都不在话下。

所以把Nginx挡在pre-forking服务前面处理请求是一种很好的选择。Nginx能够异步、高并发的响应客户端request（慢客户端请求对Nginx影响不大），Nginx一旦接收到的请求后立刻转给Gunicorn服务处理，处理结果再由Nginx以response的形式发回给客户端。这样，整个服务端和客户端的通信，就由原来仅通过Gunicorn的同步通信，变成了基于Nginx和Gunicorn的异步通信，通信效率和并发能力得到大大提升。

对于网站而言，除了要考虑上面介绍的情况，还要考虑各种静态文件的托管问题。静态文件既包括CSS、JavaScript等前端文件，也包括图片、视频和各类文档等，所以静态文件要么可能会比较大，要么会调用比较频繁，静态文件的托管功能，就是要保证各类静态能正常的加载、预览或下载，这其实就是Response耗时长的“慢客户端行为”。用Gunicorn托管静态文件，也会严重影响Gunicorn的响应效率，而这恰恰又是Nginx擅长的工作，所以静态文件的托管也交给Nginx搞定就好。

### 2. Flask网站如何部署
> 结合上一节的解释，Flask网站如何部署也很明确了：
1. 用一个服务器软件（如Gunicorn）把Flask写好的应用拉起来
2. 用Nginx给上一步拉起的应用做一个反向代理
3. 网站涉及到的静态文件，用Nginx做文件托管

常见的服务器软件是Gunicorn和uWSGI，由于Gunicorn配置使用简单，效率也不错，Gunicorn拉起Flask网站的配置极为简单，所以通常用Gunicorn来部署Flask网站是最常见的部署方案。（另，Gevin个人还有一个喜欢Gunicorn的原因是，Gunicorn是纯Python写的，直接pip安装即可，而uwsgi还要系统装额外的依赖，这在与docker配合使用时，Gunicorn的简单性尤为突出）

对于静态文件的托管，由于在开发阶段通常会基于Flask框架做静态文件托管的实现，所以当用Gunicorn拉起Flask网站时，网站已经实现了基于Gunicorn的文件托管功能，所以配置Nginx的静态文件托管URL时，可以直接配置成与基于Gunicorn托管一致的文件路径，这样能简化开发和部署的逻辑，而且由于Nginx比Gunicorn更靠外一层，客户端请求静态文件时，Nginx就直接返回Response了，不用担心会请求到Gunicorn中去影响服务器效率。

#### 2.1 Gunicorn
> Gunicorn如何部署Flask网站，直接看Flask或[Gunicorn](https://gunicorn.org/)官方文件即可，通常只要执行类似下面的一行命令：
```
/usr/local/bin/gunicorn -w 2 -b :4000 manage:app
```

其中`/usr/local/bin/gunicorn`是`gunicorn`安装后的路径
`-w`表示当前服务使用的线程数
`-b`表示当前拉起的服务的访问地址，不写默认为`localhost`，后面加上端口号
`manage:app`有两个点，前面的`manage`是你的flask的启动文件的路径，后面不带`.py`后缀，冒号后面的是你想要设置的这个服务的实例名

#### 2.2 Nginx
用Nginx做反向代理和托管静态文件，也很简单，我这里提供一个可用Demo，更多玩法大家自行查阅Nginx文档吧
```
server {
    listen 80;
    server_name localhost;
    access_log  /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;
    location / {
        proxy_pass         http://localhost:8000/;
        proxy_redirect     off;
        proxy_set_header   Host             $http_host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
    }
    location /media  {
        alias /usr/share/nginx/html/media;  
    }
    location /static  {
        alias /usr/share/nginx/html/static;  
    }
}
```
其中，做反向代理的配置是：
```
    location / {
        proxy_pass         http://localhost:8000/;
        proxy_redirect     off;

        proxy_set_header   Host             $http_host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;

    }
```
做静态文件托管的配置是：
```
    location /media  {
        alias /usr/share/nginx/html/media;  
    }

    location /static  {
        alias /usr/share/nginx/html/static;  
    }
```
我这里对两个文件夹的文件做了托管。

### 3. 基于Docker的Flask网站部署
Docker具有一次部署，到处运行的好处，把上述传统部署的方法，封装到docker image中，然后配合Docker Compose编排服务，在实践中更方便。

#### 3.1 构建Flask网站的镜像
通常，镜像中包含Flask网站的运行环境，然后把Gunicorn拉起服务作为image的运行命令即可，比如，我的OctBlog的Dockerfile编写如下：
```
# MAINTAINER        Gevin <flyhigher139@gmail.com>
# DOCKER-VERSION    18.03.0-ce, build 0520e24FROM python:3.6.5-alpine3.7
LABEL maintainer="flyhigher139@gmail.com"

RUN mkdir -p /usr/src/app  && \
    mkdir -p /var/log/gunicorn

WORKDIR /usr/src/app
COPY requirements.txt /usr/src/app/requirements.txt

RUN pip install --no-cache-dir gunicorn && \
    pip install --no-cache-dir -r /usr/src/app/requirements.txt

COPY . /usr/src/app

ENV PORT 8000
EXPOSE 8000 5000
CMD ["/usr/local/bin/gunicorn", "-w", "2", "-b", ":8000", "manage:app"]
```
这里，Gevin直接用了最小的Python-alpine镜像作为基础镜像，大大减少了即将构建的Flask应用镜像的体积。对于alpine这种只有几M的极简image而言，不安装其他系统依赖，直接`pip install uwsgi`就会报错。

#### 3.2 Nginx 相关的配置
Nginx上主要做反向代理和静态文件托管，和上面的配置文件一致，如：
```
server {
    listen 80;

    server_name localhost;

    access_log  /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    location / {
        proxy_pass         http://blog:8000/;
        proxy_redirect     off;

        proxy_set_header   Host             $http_host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;

    }

    location /media  {
        alias /usr/share/nginx/html/media;  
    }

    location /static  {
        alias /usr/share/nginx/html/static;  
    }
}
```
这个配置文件和上一章节的唯一区别是，第10行的`proxy_pass         http://blog:8000/;` 这里的反向代理的服务为blog，是下面Docker-compose中配置的OctBlog网站服务。

#### 3.3 用Docker-compose编排服务
OctBlog的Docker-compose编排文件如下：
```
version: '3'
services:
  blog:
    # restart: always
    image: gevin/octblog:0.4.1
    volumes:
      - blog-static:/usr/src/app/static
    env_file: .env
    networks:
      - webnet

  mongo:
    # restart: always
    image: mongo:3.2
    volumes:
      - /Users/gevin/projects/data/mongodb:/data/db
    networks:
      - webnet

  blog-proxy:
    # restart: always
    image: nginx:stable-alpine
    ports:
      - "8080:80"
    volumes:
      - ./default.conf:/etc/nginx/conf.d/default.conf
      - blog-static:/usr/share/nginx/html/static:ro
      - blog-static:/usr/share/nginx/html/media:ro
    networks:
      - webnet

volumes:
  blog-static:
networks:
  webnet:
```
其中，为了让多个服务能互通，创建了自定义的network webnet，为了让文件能在多个服务之间共享，公用了volume blog-static。

### 4. 其他Python web网站的部署
基于上面的内容，举一反三，我们也可以梳理出Python 各类web框架做网站部署时的一般套路：
用一个Python WSGI HTTP Server来部署代码，如:
```
Green Unicorn

uWSGI

mod_wsgi

CherryPy
```
用Nginx做反向代理
做好静态文件的托管

### 5. 注：
服务器软件除了Nginx外，还有Apache等其他选项，Gevin对其他服务器软件部署Python网站理解不深入，所以本文并未涉及。

OctBlog的源码托管在GitHub上，大家可以搜索flyhigher139/OctBlog这个repo查看

本文撰写时参考了以下内容，大家可以做延伸扩展：
```
Why flask + nginx + gunicorn:

What benefit is added by using Gunicorn + Nginx + Flask?

Why is setting Nginx as a reverse proxy a good idea?

What are the differences between nginx and gunicorn?

Why do I need Nginx and something like Gunicorn?

About Preforking:

The Preforking Model

Nginx serve flask static files:

How to serve Flask static files using Nginx?

nginx + uwsgi - what is serving static files?
```




