---
desc: 由 genpost (https://github.com/hidevopsio/genpost) 代码生成器生成
title: docker 初学习
date: 2018-04-25T15:54:16+09:00
author: jansonlv
draft: false
tags:
- docker 
---
# Docker

## 简介
Docker概念
Docker是开发人员和系统管理员 使用容器开发，部署和运行应用程序的平台。使用Linux容器部署应用程序称为容器化。容器不是新的，但它们用于轻松部署应用程序。

## 概念
1. image： 图像，一个可执行的包，包括运行一个应用程序所需的一切。
2. container : 容器， 一个image运行时的状态
   
## 安装
windows和mac直接下载desktop版本
Linux参考官方文档

## 测试
查看版本
```
PS C:\Users\Administrator> docker --version
Docker version 18.09.2, build 6247962
```
查看信息
```
PS C:\Users\Administrator> docker info
Containers: 1
 Running: 0
 Paused: 0
 Stopped: 1
Images: 3
Server Version: 18.09.2
Storage Driver: overlay2
 Backing Filesystem: extfs
 Supports d_type: true
 Native Overlay Diff: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: 9754871865f7fe2f4e74d43e2fc7ccd237edcbce
runc version: 09c8266bf2fcf9519a651b04ae54c967b9ab86ec
init version: fec3683
Security Options:
 seccomp
  Profile: default
Kernel Version: 4.9.125-linuxkit
Operating System: Docker for Windows
OSType: linux
Architecture: x86_64
CPUs: 2
Total Memory: 1.952GiB
Name: linuxkit-00155d1f5a01
ID: U4GC:5CKT:TBDG:PDKQ:HKWQ:FALW:HX4P:MY4N:77VR:YGRZ:5PL7:A4O4
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): true
 File Descriptors: 23
 Goroutines: 47
 System Time: 2019-07-06T01:41:44.186994Z
 EventsListeners: 1
Registry: https://index.docker.io/v1/
Labels:
Experimental: false
Insecure Registries:
 127.0.0.0/8
Live Restore Enabled: false
Product License: Community Engine
```
测试安装、运行容器
```
PS C:\Users\Administrator> docker run hello-world
// 本地不存在
Unable to find image 'hello-world:latest' locally
// 去线上拉取镜像
latest: Pulling from library/hello-world
1b930d010525: Pull complete                                                                                             Digest: sha256:41a65640635299bab090f783209c1e3a3f11934cf7756b09cb2f1e02147c6ed8
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

列出本地的镜像
```
PS C:\Users\Administrator> docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
redis               latest              bb0ab8a99fe6        47 hours ago        95MB
mysql               latest              c7109f74d339        3 weeks ago         443MB
hello-world         latest              fce289e99eb9        6 months ago        1.84kB
```

列出在运行镜像(redis在运行)
```
PS C:\Users\Administrator> docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
66bdfb17f623        redis               "docker-entrypoint.s…"   9 seconds ago       Up 8 seconds        0.0.0.0:6379->6379/tcp   redis_test
```

列出hello-world在显示其消息后退出的容器（由图像生成）。如果它仍在运行，您将不需要--all选项(docker ps功能一样，加上--all和docker ps -a一样)：
```
PS C:\Users\Administrator> docker container ls --all
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS                    NAMES
66bdfb17f623        redis               "docker-entrypoint.s…"   51 seconds ago      Up 50 seconds               0.0.0.0:6379->6379/tcp   redis_test
fbc1a7a48d03        hello-world         "/hello"                 10 minutes ago      Exited (0) 10 minutes ago                            dreamy_napier
```

## 回顾
```
## List Docker CLI commands
docker
docker container --help

## Display Docker version and info
docker --version
docker version
docker info

## Execute Docker image
docker run hello-world

## List Docker images(没有s)
docker image ls

## List Docker containers (running, all, all in quiet mode)
docker container ls = docker ps
docker container ls --all  = docker ps -a
显示docker所有的容器id
docker container ls -aq
显示docker在运行的容器id
docker container ls -q
```