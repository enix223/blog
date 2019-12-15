# 微信小程序开发系列 - 本地服务调试技巧

## 1. 搭建本地服务器

当我们开发微信小程序的时候，需要访问后台API接口，我们往往需要在微信小程序的后台添加授权的服务器地址，但是我们又未必想立刻把后端程序部署到服务器上。
我们更希望在开发阶段，可以通过本地服务器来快速调试小程序。这样后端程序可以快速的调试并修改。
通过本地服务器来模拟真实服务器需要解决如下两个问题：
* 域名解析问题
* SSL证书问题

因为微信小程序强制使用https来访问后端接口，所以我们必须要搭建一个支持ssl的web服务器。

### 1.1 域名解析

通过修改本机的hosts文件，我们可以把域名直接映射到我们的后端服务器的IP地址，从而把请求流量引流到我们的本地服务器。

例如，我们在微信小程序后台添加的域名为`api.enixyu.com:8000`，同时我们的本地服务器地址为`192.168.1.100:8000`，我们可以修改`hosts`文件实现域名的本地解析。
```
192.168.1.100 api.enixyu.com
```

### 1.2 HTTPS服务器
域名解析的问题解决后，接下来就是`https`的问题。搭建https web服务器，我们可以借助nginx实现反向代理把请求转发到后端服务器上，并通过配置ssl证书，实现https的支持。
为了方便搭建nginx服务器，我们可以使用docker容器，免去环境配置，依赖安装等问题。此处我们使用的是[enix223/awesome-nginx](https://hub.docker.com/r/enix223/awesome-nginx) docker image。

`nginx/conf.d/api.conf`的配置文件如下：

```
# 
# server config for backend server
# 
upstream app_server {
  # fail_timeout=0 means we always retry an upstream even if it failed
  # to return a good HTTP response

  # for UNIX domain socket setups
  # server unix:/tmp/gunicorn.sock fail_timeout=0;

  # for a TCP configuration
  server 192.168.1.1000:9000 fail_timeout=0;
}

# API server
server {
  listen [::]:8000 ssl http2;
  listen 8000 ssl http2;

  server_name api.enixyu.com;

  location / {
    # checks for static file, if not found proxy to app
    try_files $uri @proxy_to_app;
  }

  location @proxy_to_app {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    # enable this if and only if you use HTTPS
    # proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Host $http_host;

    # we don't want nginx trying to do something clever with
    # redirects, we set the Host: header above already.
    proxy_redirect off;

    proxy_pass http://app_server;
  }

  include h5bp/ssl/ssl_engine.conf;

  # certificates
  ssl_certificate /etc/nginx/certs/public.pem;
  ssl_certificate_key /etc/nginx/certs/private.key;

  include h5bp/ssl/policy_intermediate.conf;
  include h5bp/basic.conf;
  include h5bp/errors/custom_errors.conf;
  include h5bp/security/strict-transport-security.conf;
}
```

配置文件说明：

* 第12行配置我们的本地服务器地址和端口，当请求达到后，nginx会把请求转发到该地址
* 第17，18行配置我们的https服务器的监听端口，此处为8000。
* 第20行配置服务器的域名，也就是我们在微信小程序后台配置的域名，此处为api.enixyu.com。
* 第43，44行为ssl证书的路径。各大云服务商基本都提供免费的个人证书申请。

配置文件准备就绪后，我们即可创建docker container。

```
docker run -it -d \
  -p 8000:8000 \
  --name example-nginx \
  -v $PWD/nginx/certs/:/etc/nginx/certs/ \
  -v $PWD/nginx/conf.d:/etc/nginx/conf.d/ \
  enix223/awesome-nginx:alpine
```

创建nginx的container后，本地服务器的配置基本完成。我们可以通过微信开发者工具进行开发，调用后台服务就如正式环境一样。

## 2. 真机调试

有时候本地调试未必能满足我们的需求，真机调试必不可少。但是我们在手机上调试，往往无法直接修改手机的hosts文件，这时候，我们需要一个http代理服务器，帮我们把后端请求的流量转发到本地服务器上。

### 2.1 创建squid代理服务器

在linux世界里，http代理服务器最出名的就数squid。但是我们又希望可以像nginx一样，可以通过docker容器实现快速的部署。
幸好，squid也有很多docker image，此处我们使用[sameersbn/squid](https://hub.docker.com/r/sameersbn/squid) image。可通过如下命令创建squid的container：

```
docker run --name squid -d \
        --publish 3128:3128 \
		--volume $PWD/squid/conf/squid.conf:/etc/squid/squid.conf \
		--volume $PWD/squid/conf/hosts:/etc/hosts \
		sameersbn/squid:latest
```

squid的配置文件需要注意如下几点：

* http_access 默认为deny all，需要改为allow all，否则当请求https的时候，会出现403错误。
* 删除其他的http_access配置，只留上面allow all一行。


squid配置好了之后，还需要修改container里面的`hosts`文件，否则代理的流量仍然会转发至真实的域名。

hosts文件修改如下:
```
192.168.1.100 api.enixyu.com
```

### 2.2 手机使用代理服务器

完成了以上的配置后，最后一步就是配置手机的代理服务器。假如运行squid容器的主机IP为`192.168.1.100`，则http的代理设置如下所示：
```
服务器 192.168.1.100
端口 3128
```

大功告成，此时手机上的小程序可以无缝连接我们的本地服务器。
