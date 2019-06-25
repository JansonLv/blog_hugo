---
desc: 由 genpost (https://github.com/hidevopsio/genpost) 代码生成器生成
title: docker文件挂载和网络
date: 2018-02-25T10:19:22+08:00
author: jansonlv
draft: false
tags:
- docker 
---
# Docker学习

## 创建容器常用选项
![](media/15562472053257.jpg)

## 容器管理
![](media/15562474749742.jpg)

## 数据挂载
Docker提供三种不同的方式将数据从宿主机挂载到容器中： volumes， bind mounts和tmpfs。
* volumes： Docker管理宿主机文件系统的一部分（ /var/lib/docker/volumes）。
* bind mounts：可以存储在宿主机系统的任意位置。
* tmpfs：挂载存储在宿主机系统的内存中，而不会写入宿主机的文件系统。
![](media/15562480009858.jpg)

###volumes
**管理卷：**
docker volume create nginx-vol
docker volume ls
docker volume inspect nginx-vol
**用卷创建一个容器：**
docker run -d -it --name=nginx-test --mount src=nginx-vol,dst=/usr/share/nginx/html nginx （推荐）
docker run -d -it --name=nginx-test -v nginx-vol:/usr/share/nginx/html nginx
**清理：**
docker container stop nginx-test
docker container rm nginx-test
docker volume rm nginx-vol

*注意：*
    1. 如果没有指定卷，自动创建。
    2. 建议使用—mount，更通用。

官方文档： https://docs.docker.com/engine/admin/volumes/volumes/#start-a-container-with-a-volume

个人理解：
1. 相当于在/var/lib/docker/volumes文件夹中创建一个文件夹，然后docker启动时直接挂载这个文件夹，在主机或者容器中修改该文件夹内的内容，另一方也会修改。
2. 多个容器可以挂载一个相同文件夹
3. 实时内容映射

### Bind Mounts
**用卷创建一个容器：**
* docker run -d -it --name=nginx-test --mount type=bind,src=/app/wwwroot,dst=/usr/share/nginx/html nginx
* docker run -d -it --name=nginx-test -v /app/wwwroot:/usr/share/nginx/html nginx

**验证绑定：**
* docker inspect nginx-test

**清理：**
* docker container stop nginx-test
* docker container rm nginx-test

*注意：*
1. 如果源文件/目录没有存在，不会自动创建，会抛出一个错误。
2. 如果挂载目标在容器中非空目录，则该目录现有内容将被隐藏。

官方文档： https://docs.docker.com/engine/admin/volumes/bind-mounts/#start-a-container-with-a-bind-mount

个人理解
1. 自定义一个文件夹，启动时挂载
2. src和dst需要手动创建
3. dst内容会隐藏
4. BindMounts与volumes区别，volumes src和dst无需创建，自动创建，BindMounts则都需要手动创建； volumes启动时，内容相互映射，BindMounts则src->dst。


## 创建lnmp应用
1. 自定义网络
docker network create lnmp
2. 创建Mysql数据库容器
docker run -itd \
--name lnmp_mysql \
--net lnmp \
-p 3306:3306 \
--mount src=mysql-vol,dst=/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
mysql --character-set-server=utf8
3. 创建所需数据库
docker exec lnmp_mysql sh \
-c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e"create database wp"'
4. 创建PHP环境容器
docker run -itd \
--name lnmp_web \
--net lnmp \
-p 88:80 \
--mount type=bind,src=/app/wwwroot,dst=/var/www/html richarvey/nginx-php-fpm
5. 以wordpress博客为例测试
wget https://cn.wordpress.org/wordpress-4.7.4-zh_CN.tar.gz
tar zxf wordpress-4.7.4-zh_CN.tar.gz -C /app/wwwroot

**浏览器测试访问**
http://IP:88/wordpress

## Dockerfile
### 指令
![](media/15562602649161.jpg)

| 指令 | 描述 | 指令 | 描述 |
| --- | --- | --- | --- |
| FROM| 构建镜像的来源镜像 | MAINTAINER | 维护这的信息和邮箱地址 |
| RUN | 构建镜像时运行的shell命令<br>例如：<br>RUN["yum","install","httpd"]<br>RUN yum install httpd | CMD | 运行容器时执行的shell命令 <br> 例如：<br>CMD [“ -c”, “/start.sh”]<br>CMD ["/usr/sbin/sshd", "-D"]<br>CMD /usr/sbin/sshd –D|
| EXPOSE | 申明容器的服务端口<br>例如： EXPOSE 80 443| ENV | 设置容器内环境变量<br>例如：<br>ENV MYSQL_ROOT_PASSWORD 123456|
|ADD| 拷贝文件或目录到镜像，如果是url或者压缩包会自动下载或解压<br>ADD <src>… <dest><br>ADD ["src",… "dest"]<br>ADD https://x.com/h.tar.gz /var/www/html<br>ADD html.tar.gz /var/www/html|COPY| 拷贝文件或目录或者镜像，同add|
|ENTRYPOINT|运行容器时执行的Shell命令 <br>例如： <br>ENTRYPOINT [“/bin/bash", “ -c", “/start.sh"] <br>ENTRYPOINT /bin/bash -c ‘/start.sh’| VOLUME| 挂载宿主机自动生成的目录或其他容器的容器目录<br>例如：<br>VOLUME ["/var/lib/mysql"]|
|USER|为RUN、 CMD和ENTRYPOINT执行命令指定运行用户<br>USER user[:group]<br>USER UID[:GID]<br>例如： USER lizhenliang|WORKDIR|为RUN、 CMD、 ENTRYPOINT、 COPY和ADD设置工作目录<br>例如： WORKDIR /data|
|HEALTHCHECK|健康检查<br>HEALTHCHECK --interval=5m --timeout=3s --retries=3 \CMD curl -f http://localhost/ \|\| exit 1| ARG | 在构建镜像时指定一些参数<br> 例如：<br>FROM centos:6 <br>ARG user # ARG user=root USER $user <br>docker build --build-arg user=lizhenliang Dockerfile . |


*注意*
1. 当RUN 后面接数组形式命令时，此刻相当于使用exec并行执行这个命令
2. ENTRYPOINT 可以接受docker run 启动时带入的参数，dockerfile的最后一条CMD会被覆盖

### Build镜像命令
Usage: docker image build [OPTIONS] PATH | URL | -
Options:
-t, --tag list # 镜像名称
-f, --file string # 指定Dockerfile文件位置
示例：-t 镜像名 -f dockerfile路径 最后一个是上下文环境
docker build .
docker build -t shykes/myapp .
docker build -t shykes/myapp -f /path/Dockerfile /path

## 镜像仓库
### 搭建私有镜像仓库
Docker Hub作为Docker默认官方公共镜像；如果想自己搭建私有镜像仓库，官方也提供registry镜像，使得搭建私有仓库非常简单。
1. 下载registry镜像并启动
docker pull registry
docker run -d -v /opt/registry:/var/lib/registry -p 5000:5000 --restart=always --name registry registry
2. 测试，查看镜像仓库中所有镜像
curl http://192.168.0.212:5000/v2/_catalog
{"repositories":[]}

### 私有镜像仓库管理
1. 配置私有仓库可信任
vi /etc/docker/daemon.json
{"insecure-registries":["192.168.0.212:5000"]}
systemctl restart docker
2. 打标签
docker tag centos:6 192.168.0.212:5000/centos:6
3. 上传
docker push 192.168.0.212:5000/centos:6
4. 下载
docker pull 192.168.0.212:5000/centos:6
5. 列出镜像标签
curl http://192.168.0.212:5000/v2/centos/tags/list

## Docker Hub公共镜像仓库使用
1. 注册账号
https://hub.docker.com
2. 登录Docker Hub
 docker login
或
 docker login --username=lizhenliang --password=123456
3. 镜像打标签
 docker tag wordpress:v1 lizhenliang/wordpress:v1
4. 上传
docker push lizhenliang/wordpress:v1
搜索测试：
docker search lizhenliang
5. 下载
docker pull lizhenliang/wordpress:v1