# services

## 介绍
在分布式应用程序中，应用程序的不同部分称为“服务”。例如，如果您想做一个视频共享站点，它可能包括一个用于在数据库中存储应用程序数据的服务，一个用于在后台进行视频转码的服务，还有用户上传内容，前端服务等。

服务实际上只是“containers in production.”。一个服务只运行一个镜像，但它规定image运行的规则 - 它应该使用哪些端口，应该运行多少个容器副本，以便服务具有所需的容量等等。扩展服务会更改运行该软件的容器实例的数量，从而为流程中的服务分配更多计算资源。

幸运的是，使用Docker平台定义，运行和扩展服务非常容易 - 只需编写一个docker-compose.yml文件即可。

## docker-compose.yml
```yml
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: jansonlv/get-started:v0.1
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "80:80"
    networks:
      - webnet
networks:
  webnet:
```
这个docker-compose告诉docker按照以下要求运行

1. 使用jansonlv/get-started:v0.1这个image镜像
2. 运行5个副本，并且名为web，限制使用最高10%的cpu和50M内存
3. 如果有一个副本失败，重启容器
4. 映射主机80端口到容器80端口
5. 所有web服务都连接到webnet网络
6. webnet网络使用默认的配置，负载均衡的网络

## 运行新的负载均衡应用
在我们docker stack deploy首先运行命令之前：
```
docker swarm init

PS C:\Users\Administrator\Documents\blog_hugo\content\docker初学习\demo\01_start> docker swarm init
Swarm initialized: current node (u6bx1zd079pry9ne4bqx4ttrm) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2qxck60rcp6vbe9prhqawij1cjej9a7ddce9lk8k5qfqu3tc5k-auh7ej4n03xbwfjysz520o8im 192.168.65.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
现在让我们来运行吧。您需要为您的应用程序命名。在这里，它被设置为 getstartedlab：
```
docker stack deploy -c docker-compose.yml getstartedlab
```
这边发现一个遗留问题，启动后又关闭了,想了下是web文件名没有，我去更新下，和官方同步,重新推送为v0.2版本

```
PS C:\Users\Administrator\Documents\blog_hugo\content\docker初学习\demo\01_start> docker stack deploy -c docker-compose.yml getstartedlab
Updating service getstartedlab_web (id: 135w4h9xn91tre1k0i2xw7lat)
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
66bdfb17f623        redis               "docker-entrypoint.s…"   9 hours ago         Up 9 hours          0.0.0.0:6379->6379/tcp   redis_test

PS C:\Users\Administrator\Documents\blog_hugo\content\docker初学习\demo\01_start> docker ps
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                    NAMES
01f91c102caa        jansonlv/get-started:v0.2   "python app.py"          10 seconds ago      Up 3 seconds        80/tcp                   getstartedlab_web.2.6pagyw392c27lampliflnza0z
998e174de9bc        jansonlv/get-started:v0.2   "python app.py"          12 seconds ago      Up 6 seconds        80/tcp                   getstartedlab_web.4.oeijkp52f1idunzdsirvv9z3v
0895832495aa        jansonlv/get-started:v0.2   "python app.py"          14 seconds ago      Up 6 seconds        80/tcp                   getstartedlab_web.3.tp2u1hw12mhx6nai9h0nplili
81c0aff78db3        jansonlv/get-started:v0.2   "python app.py"          14 seconds ago      Up 6 seconds        80/tcp                   getstartedlab_web.1.lxka9ffhfp5uq6ni47bfkq2a4
ffcee23b7488        jansonlv/get-started:v0.2   "python app.py"          17 seconds ago      Up 12 seconds       80/tcp                   getstartedlab_web.5.melxhjp7npu7pn2ku2w1min89
66bdfb17f623        redis                       "docker-entrypoint.s…"   9 hours ago         Up 9 hours          0.0.0.0:6379->6379/tcp   redis_test
```
最终都启动完毕

现在查看服务
```
PS C:\Users\Administrator> docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                       PORTS
135w4h9xn91t        getstartedlab_web   replicated          5/5                 jansonlv/get-started:v0.2   *:80->80/tcp
```

查找服务的输出，并web以您的应用名称为前缀。如果您将其命名为与此示例中显示的相同，则名称为 getstartedlab_web。还列出了服务ID，以及副本数，映像名称和公开端口。

在服务中运行的单个容器称为任务。任务被赋予以数字递增的唯一ID，最多为replicas您定义 的数量docker-compose.yml。列出您的服务任务：
```
docker service ps getstartedlab_web
```
如果您只列出系统上的所有容器，则任务也会显示，但不会被服务过滤：
```
docker container ls -q
```
### 扩展应用程序
您可以通过更改replicas值docker-compose.yml，保存更改并重新运行docker stack deploy命令来扩展应用程序：

    docker stack deploy -c docker-compose.yml getstartedlab
Docker执行就地更新，无需首先拆除堆栈或杀死任何容器。

现在，重新运行docker container ls -q以查看已重新配置的已部署实例。如果放大副本，则会启动更多任务，从而启动更多容器。

关闭应用程序和服务，将应用程序关闭docker stack rm：

    docker stack rm getstartedlab

退出节点
    
    docker swarm leave --force

回顾一下，虽然键入docker run很简单，但生产中容器的真正实现是将其作为服务运行。服务在Compose文件中编码容器的行为，此文件可用于扩展，限制和重新部署我们的应用程序。服务的更改可以在运行时使用启动服务的相同命令来应用： docker stack deploy。

在此阶段要探索的一些命令：
```
docker stack ls                                            # List stacks or apps
docker stack deploy -c <composefile> <appname>  # Run the specified Compose file
docker service ls                 # List running services associated with an app
docker service ps <service>                  # List tasks associated with an app
docker inspect <task or container>                   # Inspect task or container
docker container ls -q                                      # List container IDs
docker stack rm <appname>                             # Tear down an application
docker swarm leave --force      # Take down a single node swarm from the manager
```