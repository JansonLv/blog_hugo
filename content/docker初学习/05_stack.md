# stack

## 介绍
在这一部分中，我们到达分布式应用程序层次结构的顶部：stack。stack是一组相互关联的服务，它们共享依赖关系，并且可以协调和缩放在一起。单个堆栈能够定义和协调整个应用程序的功能（尽管非常复杂的应用程序可能希望使用多个堆栈）。

一些好消息是，从第3部分开始，当创建Compose文件并使用时，在技术上一直在使用堆栈docker stack deploy。但这是在单个主机上运行的单个服务堆栈，这通常不是生产中发生的。在这里，可以学习所学内容，使多个服务相互关联，并在多台计算机上运行它们。

## 添加新服务并重新部署
将服务添加到我们的docker-compose.yml文件很容易。首先，让我们添加一个免费的可视化服务，让我们看看我们的swarm如何调度容器。

1. docker-compose.yml在编辑器中打开并用以下内容替换其内容。请务必更换username/repo:tag图像详细信息。
```
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: jansonlv/get-started:v0.2
    deploy:
      replicas: 5
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
    ports:
      - "80:80"
    networks:
      - webnet
  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
networks:
  webnet:
```
这里唯一新的是对等服务web，命名visualizer。注意这里有两个新东西：一个volumes键，让可视化工具访问Docker的主机套接字文件，以及一个placement密钥，确保该服务只能在一个swarm管理器上运行 - 绝不是一个工作者。这是因为这个容器是由Docker创建的一个开源项目构建的，它显示了在图中的swarm上运行的Docker服务。

我们马上谈论更多关于布局约束和数量的内容。

2. 确保shell配置为与之通信myvm1（完整示例在此处）。

运行docker-machine ls以列出计算机并确保已连接到myvm1，如旁边的星号所示。

如果需要，请重新运行docker-machine env myvm1，然后运行给定命令以配置shell。

在Mac或Linux上，命令是：

    eval $(docker-machine env myvm1)

3. docker stack deploy在管理器上重新运行该命令，并更新需要更新的任何服务：
```
$ docker stack deploy -c docker-compose.yml getstartedlab
Updating service getstartedlab_web (id: angi1bf5e4to03qu9f93trnxm)
Creating service getstartedlab_visualizer (id: l9mnwkeq2jiononb5ihz9u7a4)
```

4. 看看可视化工具。

在Compose文件中看到在visualizer端口8080上运行docker-machine ls。通过运行获取其中一个节点的IP地址。转到端口8080的IP地址，可以看到运行的可视化工具.

visualizer正如所期望的那样，单个副本正在管理器上运行，并且5个实例web分布在整个群集中。可以通过运行docker stack ps <stack>以下来证实此可视化：

    docker stack ps getstartedlab
可视化工具是一个独立的服务，可以在任何包含它的应用程序中运行。它不依赖于任何其他东西。现在让我们创建一个服务不会有依赖性：Redis的服务，提供访客计数器。

## 持久化数据
让我们再次通过相同的工作流程来添加Redis数据库来存储应用数据。

保存这个新docker-compose.yml文件，最后添加一个Redis服务。
```
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: jansonlv/get-started:v0.2
    deploy:
      replicas: 5
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
    ports:
      - "80:80"
    networks:
      - webnet
  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
  redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - "/home/docker/data:/data"
    deploy:
      placement:
        constraints: [node.role == manager]
    command: redis-server --appendonly yes
    networks:
      - webnet
networks:
  webnet:

```
Redis在Docker库中有一个官方图像，并且已被授予imagejust 的简称redis，因此username/repo这里没有注释。Redis端口6379已由Redis预先配置为从容器暴露给主机，在我们的Compose文件中，我们将它从主机暴露给外界，因此实际上可以输入任何IP的IP如果愿意，可以将节点导入Redis Desktop Manager并管理此Redis实例。

最重要的是，redis规范中有一些事情会使数据在此堆栈的部署之间保持不变：

redis 总是在管理器上运行，所以它总是使用相同的文件系统。
* redis访问主机文件系统中的任意目录作为/data容器内部，这是Redis存储数据的位置。
* 总之，这是在主机的物理文件系统中为Redis数据创建“真实来源”。如果没有这个，Redis会将其数据存储 /data在容器的文件系统中，如果重新部署该容器，将会被删除。

这个真相来源有两个组成部分：

* 放置在Redis服务上的放置约束，确保它始终使用相同的主机。
* 创建的容器允许容器访问./data（在主机上）/data（在Redis容器内）。当容器来来往往时，存储在./data指定主机上的文件仍然存在，从而实现连续性。
已准备好部署新的Redis-using堆栈。

2. ./data在管理器上创建一个目录：

    docker-machine ssh myvm1 "mkdir ./data"
3. 确保shell配置为与之通信myvm1（完整示例在此处）。

运行docker-machine ls以列出计算机并确保已连接到myvm1，如旁边的星号所示。

如果需要，请重新运行docker-machine env myvm1，然后运行给定命令以配置shell。

    eval $(docker-machine env myvm1)

4. 再跑docker stack deploy一次。

    $ docker stack deploy -c docker-compose.yml getstartedlab
```
docker stack deploy -c docker-compose.yml getstartedlab
Creating service getstartedlab_redis
Updating service getstartedlab_web (id: lgv9avgavco8fkg1uc5if6yq1)
Updating service getstartedlab_visualizer (id: tgox1rptcjilsbwraj5cqurgb)
```

5. 运行docker service ls以验证三个服务是否按预期运行。
```
docker service ls
ID                  NAME                       MODE                REPLICAS            IMAGE                             PORTS
x6pp9y0v9heb        getstartedlab_redis        replicated          1/1                 redis:latest                      *:6379->6379/tcp
tgox1rptcjil        getstartedlab_visualizer   replicated          1/1                 dockersamples/visualizer:stable   *:8080->8080/tcp
lgv9avgavco8        getstartedlab_web          replicated          5/5                 jansonlv/get-started:v0.2         *:80->80/tcp
```

6. 检查一个节点上的网页，例如http://192.168.99.105，查看访问者计数器的结果，该计数器现已存在并在Redis上存储信息。
```
<h3>Hello World!</h3><b>Hostname:</b> a99626390ba0<br/><b>Visits:</b> 4%                                                                                                       jansonlv@lvdongchengdeMacBook-Pro-2  ~/go/src/startDockerCompose  curl http://192.168.99.105
<h3>Hello World!</h3><b>Hostname:</b> a99626390ba0<br/><b>Visits:</b> 5%                                                                                                       jansonlv@lvdongchengdeMacBook-Pro-2  ~/go/src/startDockerCompose  curl http://192.168.99.105
<h3>Hello World!</h3><b>Hostname:</b> a99626390ba0<br/><b>Visits:</b> 6%                                                                                                       jansonlv@lvdongchengdeMacBook-Pro-2  ~/go/src/startDockerCompose  curl http://192.168.99.106     
<h3>Hello World!</h3><b>Hostname:</b> a99626390ba0<br/><b>Visits:</b> 7%                                                                                                       jansonlv@lvdongchengdeMacBook-Pro-2  ~/go/src/startDockerCompose  curl http://192.168.99.107     
<h3>Hello World!</h3><b>Hostname:</b> a99626390ba0<br/><b>Visits:</b> 8%        
```
