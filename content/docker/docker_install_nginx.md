---
desc: 由 genpost (https://github.com/hidevopsio/genpost) 代码生成器生成
title: docker nginx安装
date: 2018-02-25T10:19:22+08:00
author: jansonlv
draft: false
tags:
- docker nginx
---

> 本文来自[菜鸟教程](https://www.runoob.com/docker/docker-install-nginx.html)，并优化配置多个conf的问题。

### 拉取镜像
    docker pull nginx
### 启动镜像获取配置文件
    docker run --name runoob-nginx-test -p 8081:80 -d nginx
* runoob-nginx-test 容器名称。
* the -d设置容器在在后台一直运行。
* the -p 端口进行映射，将本地 8081 端口映射到容器内部的 80 端口。

执行以上命令会生成一串字符串，类似 6dd4380ba70820bd2acc55ed2b326dd8c0ac7c93f68f0067daecad82aef5f938，这个表示容器的 ID，一般可作为日志的文件名。

我们可以使用 docker ps 命令查看容器是否有在运行：
```
$ docker ps
CONTAINER ID        IMAGE        ...               PORTS                  NAMES
6dd4380ba708        nginx        ...      0.0.0.0:8081->80/tcp   runoob-nginx-test
```
PORTS 部分表示端口映射，本地的 8081 端口映射到容器内部的 80 端口。

在浏览器中打开 http://127.0.0.1:8081/，效果如下：
![](media/15608457827843.jpg)

## nginx 部署
### 创建目录 nginx, 用于存放后面的相关东西。
        
        $ mkdir -p ~/nginx/www ~/nginx/logs ~/nginx/conf
拷贝容器内 Nginx 默认配置文件到本地当前目录下的 conf 目录，容器 ID 可以查看 docker ps 命令输入中的第一列：
    
    docker cp 6dd4380ba708:/etc/nginx/ ~/nginx/conf
    
www: 目录将映射为 nginx 容器配置的虚拟目录。
logs: 目录将映射为 nginx 容器的日志目录。
conf: 目录里的配置文件将映射为 nginx 容器的配置文件。
### 部署命令
    $ docker run -d -p 80:80 --name nginx-web -v ~/nginx/www:/usr/share/nginx/html -v ~/nginx/conf:/etc/nginx -v ~/nginx/logs:/var/log/nginx nginx

命令说明：
-p 80:80： 将容器的 80 端口映射到主机的 80 端口。
--name nginx-web：将容器命名为 nginx-web。
-v ~/nginx/www:/usr/share/nginx/html：将我们自己创建的 www 目录挂载到容器的 /usr/share/nginx/html。
-v ~/nginx/conf:/etc/nginx：将我们自己创建的/nginx/conf文件夹 挂载到容器的 /etc/nginx文件夹。
-v ~/nginx/logs:/var/log/nginx：将我们自己创建的 logs 挂载到容器的 /var/log/nginx。

### 启动以上命令后进入 ~/nginx/www 目录：
$ cd ~/nginx/www
创建 index.html 文件，内容如下：
```
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>菜鸟教程(runoob.com)</title>
</head>
<body>
    <h1>我的第一个标题</h1>
    <p>我的第一个段落。</p>
</body>
</html>
```
相关命令
如果要重新载入 NGINX 可以使用以下命令发送 HUP 信号到容器：

    $ docker kill -s HUP container-name
重启 NGINX 容器命令：

    $ docker restart container-name
    
