# swarms
说明：由于windows hyper环境问题和个人能力有限，后续无法开展，故切换到mac环境

## 介绍
群集是一组运行Docker并加入群集的计算机。在此之后，继续运行习惯使用的Docker命令，但现在它们由群集管理器在群集上执行。群中的机器可以是物理的或虚拟的。加入群组后，它们被称为节点。

Swarm管理器可以使用多种策略来运行容器，例如“emptiest node” - 它使用容器填充利用率最低的机器。或“global”，它确保每台机器只获得指定容器的一个实例。指示swarm管理器在Compose文件中使用这些策略，就像已经使用的那样。

群集管理器是swarm中唯一可以执行命令的机器，或授权其他机器作为工作者加入群集。节点只是在那里提供能力，没有权力告诉任何其他机器它能做什么和不能做什么。

到目前为止，一直在本地计算机上以单主机模式使用Docker。但Docker也可以切换到swarm模式，这就是使用群集的原因。立即启用群集模式使当前计算机成为群集管理器。从那时起，Docker就会运行在管理的swarm上执行的命令，而不仅仅是在当前机器上。

## 设置你的群
群由多个节点组成，可以是物理或虚拟机。基本概念很简单：运行docker swarm init以启用swarm模式并使当前计算机成为一个swarm管理器，然后docker swarm join在其他计算机上运行 以使它们作为worker加入swarm。选择下面的标签，了解它在各种情况下的效果。我们使用VM快速创建一个双机群集并将其转换为群集。

除了win10是有的hpyer外的其他环境都需要安装Oracle VirtualBox；不过win10测试失败。

安装好使用命令：
```
docker-machine create --driver virtualbox myvm1
docker-machine create --driver virtualbox myvm2
docker-machine create --driver virtualbox myvm3
```
创建样例
```
jansonlv@lvdongchengdeMacBook-Pro-2  ~/docker/mysql  docker-machine create --driver virtualbox myvm2
Running pre-create checks...
Creating machine...
(myvm2) Copying /Users/jansonlv/.docker/machine/cache/boot2docker.iso to /Users/jansonlv/.docker/machine/machines/myvm2/boot2docker.iso...
(myvm2) Creating VirtualBox VM...
(myvm2) Creating SSH key...
(myvm2) Starting the VM...
(myvm2) Check network to re-create if needed...
(myvm2) Waiting for an IP...
^@Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with boot2docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env myvm2
```
使用 docker-machine ls
```
jansonlv@lvdongchengdeMacBook-Pro-2  ~  docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
myvm1   -        virtualbox   Running   tcp://192.168.99.105:2376           v18.09.7
myvm2   -        virtualbox   Running   tcp://192.168.99.106:2376           v18.09.7
myvm3   -        virtualbox   Running   tcp://192.168.99.107:2376           v18.09.7
```

## 初始化SWARM并添加节点
第一台机器充当管理器，它执行管理命令并验证工人加入群，第二台是工人。

可以使用命令向VM发送命令docker-machine ssh。指示myvm1 成为一个swarm管理器docker swarm init并查找如下输出：
```
 jansonlv@lvdongchengdeMacBook-Pro-2  ~  docker-machine ssh myvm1 "docker swarm init --advertise-addr '192.168.99.105'"
Swarm initialized: current node (i66f3wr3l0cbtpsm6dixodu4c) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-0hy32lmxzzvexfrzp7j731eax8p8b5lbuuvtcne8w8xcp6mvt9-d6ouxdg9ilu1rsys7n5ywgqsg 192.168.99.105:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
如所见，响应docker swarm init包含一个预先配置的 docker swarm join命令，可以在要添加的任何节点上运行该命令。复制此命令，并将其发送到myvm2via docker-machine ssh以myvm2 将新群组作为工作者加入：
```
jansonlv@lvdongchengdeMacBook-Pro-2  ~  docker-machine ssh myvm2 "docker swarm join --token SWMTKN-1-0hy32lmxzzvexfrzp7j731eax8p8b5lbuuvtcne8w8xcp6mvt9-d6ouxdg9ilu1rsys7n5ywgqsg 192.168.99.105:2377"

This node joined a swarm as a worker.
 jansonlv@lvdongchengdeMacBook-Pro-2  ~  docker-machine ssh myvm3 "docker swarm join --token SWMTKN-1-0hy32lmxzzvexfrzp7j731eax8p8b5lbuuvtcne8w8xcp6mvt9-d6ouxdg9ilu1rsys7n5ywgqsg 192.168.99.105:2377"

This node joined a swarm as a worker.
```
恭喜你，你已经创建了你的第一个群！

docker node ls在管理器上运行以查看此群中的节点：
```
docker-machine ssh myvm1 "docker node ls"
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
i66f3wr3l0cbtpsm6dixodu4c *   myvm1               Ready               Active              Leader              18.09.7
t1x19zozp7f62dezy08p1f7vy     myvm2               Ready               Active                                  18.09.7
sle7i6budpu9diq5vt0h608fq     myvm3               Ready               Active                                  18.09.7
```
如果要重新开始，可以docker swarm leave从每个节点运行。

## 在群集群集上部署应用程序
艰难的部分结束了。现在，只需重复上一章节03_services中使用的过程即可部署到新的swarm上。请记住，只有集群管理员可以myvm1执行Docker命令; worker只是为了能力。

到目前为止，已经将Docker命令包装在docker-machine ssh与VM通信中。另一种选择是运行docker-machine env \<machine>以获取并运行一个命令，该命令将当前shell配置为与VM上的Docker守护程序通信。此方法适用于下一步，因为它允许使用本地docker-compose.yml文件“远程”部署应用程序，而无需将其复制到任何位置。

键入docker-machine env myvm1，然后复制粘贴并运行作为输出的最后一行提供的命令，以配置要与之通信的shell（myvm1swarm管理器）。
```
jansonlv@lvdongchengdeMacBook-Pro-2  ~  docker-machine env myvm1
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.105:2376"
export DOCKER_CERT_PATH="/Users/jansonlv/.docker/machine/machines/myvm1"
export DOCKER_MACHINE_NAME="myvm1"
# Run this command to configure your shell:
# eval $(docker-machine env myvm1)
 jansonlv@lvdongchengdeMacBook-Pro-2  ~  eval $(docker-machine env myvm1)
 jansonlv@lvdongchengdeMacBook-Pro-2  ~  docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
myvm1   *        virtualbox   Running   tcp://192.168.99.105:2376           v18.09.7
myvm2   -        virtualbox   Running   tcp://192.168.99.106:2376           v18.09.7
myvm3   -        virtualbox   Running   tcp://192.168.99.107:2376           v18.09.7
```
## 在swarm管理器上部署应用程序
现在myvm1，可以使用其作为群组管理器的功能，通过使用docker stack deploy在第3部分中使用的相同命令myvm1以及本地副本来部署应用程序docker-compose.yml.。此命令可能需要几秒钟才能完成，部署需要一些时间才能完成。使用docker service ps <service_name>swarm管理器上的 命令验证是否已重新部署所有服务。

myvm1通过docker-machineshell配置进行连接，仍然可以访问本地主机上的文件。确保与以前位于同一目录中，其中包括docker-compose.yml在第3部分中创建的 文件。

与以前一样，运行以下命令以部署应用程序myvm1。

docker stack deploy -c docker-compose.yml getstartedlab
就是这样，应用程序部署在一个群集集群中！
```
注意：如果映像存储在私有注册表而不是Docker Hub上，则需要使用登录docker login <your-registry>，然后需要将--with-registry-auth标志添加到上述命令中。例如：

docker login registry.example.com

docker stack deploy --with-registry-auth -c docker-compose.yml getstartedlab
这会使用加密的WAL日志将登录令牌从本地客户端传递到部署服务的swarm节点。有了这些信息，节点就可以登录注册表并提取图像。
```

```
jansonlv@lvdongchengdeMacBook-Pro-2  ~  docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: jansonlv
Password:
Login Succeeded

docker stack deploy --with-registry-auth -c docker-compose.yml getstartedlab
Creating network getstartedlab_webnet
Creating service getstartedlab_web

docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                       PORTS
lgv9avgavco8        getstartedlab_web   replicated          0/4                 jansonlv/get-started:v0.2   *:8081->80/tcp

docker service ps getstartedlab_web
ID                  NAME                  IMAGE                       NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
gvprfzm9v8l6        getstartedlab_web.1   jansonlv/get-started:v0.2   myvm2               Running             Running 46 seconds ago
r7zovwchh006        getstartedlab_web.2   jansonlv/get-started:v0.2   myvm2               Running             Running 45 seconds ago
q3otu7vz9zhl        getstartedlab_web.3   jansonlv/get-started:v0.2   myvm3               Running             Running 47 seconds ago
d5v03zbs05vw        getstartedlab_web.4   jansonlv/get-started:v0.2   myvm1               Running             Running 41 seconds ago

docker stack ps getstartedlab和docker service ps getstartedlab_web效果一样
````
三个节点部署了四个service，其中myvm2中部署了两个

### 访问群集
你可以从IP地址来访问你的应用程序要么 myvm1或myvm2或者myvm3。

创建的网络在它们之间共享并进行负载平衡。运行 docker-machine ls以获取VM的IP地址，并在浏览器上访问其中任何一个，点击刷新（或只是curl它们）。
```
<h3>Hello World!</h3><b>Hostname:</b> 137293db0f75<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>%                                                                                                                        jansonlv@lvdongchengdeMacBook-Pro-2  ~/blog/content/docker初学习/demo/01_start   master  curl http://192.168.99.105:8081
<h3>Hello World!</h3><b>Hostname:</b> 04f49ad8d50b<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>%                                                                                                                        jansonlv@lvdongchengdeMacBook-Pro-2  ~/blog/content/docker初学习/demo/01_start   master  curl http://192.168.99.105:8081
<h3>Hello World!</h3><b>Hostname:</b> 947ba6f3ea57<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>%                                                                                                                        jansonlv@lvdongchengdeMacBook-Pro-2  ~/blog/content/docker初学习/demo/01_start   master  curl http://192.168.99.105:8081
<h3>Hello World!</h3><b>Hostname:</b> 5e8f44b0fb6a<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>%                                                                                                                        jansonlv@lvdongchengdeMacBook-Pro-2  ~/blog/content/docker初学习/demo/01_start   master  curl http://192.168.99.105:8081
<h3>Hello World!</h3><b>Hostname:</b> 137293db0f75<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>%                                                                                                                        jansonlv@lvdongchengdeMacBook-Pro-2  ~/blog/content/docker初学习/demo/01_start   master  curl http://192.168.99.105:8081
<h3>Hello World!</h3><b>Hostname:</b> 04f49ad8d50b<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>%
```
有多种可能的容器ID都是随机循环的，这表明了负载平衡。

两个IP地址工作的原因是群中的节点参与入口路由网格。这可确保部署在swarm中某个端口的服务始终将该端口保留给自身，无论实际运行容器的是哪个节点。

#### 有连接麻烦？

请记住，要在群集中使用入口网络，需要在启用群集模式之前在群集节点之间打开以下端口：

端口7946 TCP / UDP用于容器网络发现。
端口4789 UDP用于容器入口网络。
仔细检查Web服务下端口部分中的内容，并确保在浏览器中输入的IP地址或卷曲反映了该地址

## 迭代和扩展应用程序
从这里，可以完成第2部分和第3部分中学到的所有知识。

通过更改docker-compose.yml文件来扩展应用程序。

通过编辑代码，然后重建并推送新图像来更改应用程序行为。（为此，请按照之前用于构建应用程序并发布图像的相同步骤）。

在任何一种情况下，只需docker stack deploy再次运行即可部署这些更改。

可以使用docker swarm join使用的相同命令将任何计算机（物理或虚拟）加入此群myvm2，并将容量添加到群集中。只需在docker stack deploy之后运行，应用就可以利用新资源

## 清理并重新启动
### 堆栈和群
你可以拆掉堆栈docker stack rm。例如（先别急着使用）：

    docker stack rm getstartedlab

> 保持群或删除某个节点？

>在某些时候，你可以删除这个群，如果你想 docker-machine ssh myvm2 "docker swarm leave"在工人和docker-machine ssh myvm1 "docker swarm leave --force"经理上，但你需要下一个部分，所以现在保持它。

### 取消设置docker-machine shell变量设置
可以docker-machine使用给定命令在当前shell中取消设置环境变量。
    
    eval $(docker-machine env -u)

这会使shell与docker-machine创建的虚拟机断开连接，并允许继续在同一个shell中工作，现在使用本机docker 命令（例如，在Docker Desktop for Mac或Docker Desktop for Windows上）。

### 重启Docker机器
如果关闭本地主机，Docker计算机将停止运行。可以通过运行来检查机器的状态docker-machine ls。

$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL   SWARM   DOCKER    ERRORS
myvm1   -        virtualbox   Stopped                 Unknown
myvm2   -        virtualbox   Stopped                 Unknown
要重新启动已停止的计算机，请运行：

docker-machine start <machine-name>

```
docker-machine create --driver virtualbox myvm1 # Create a VM (Mac, Win7, Linux)
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" myvm1 # Win10
docker-machine env myvm1                # View basic information about your node
docker-machine ssh myvm1 "docker node ls"         # List the nodes in your swarm
docker-machine ssh myvm1 "docker node inspect <node ID>"        # Inspect a node
docker-machine ssh myvm1 "docker swarm join-token -q worker"   # View join token
docker-machine ssh myvm1   # Open an SSH session with the VM; type "exit" to end
docker node ls                # View nodes in swarm (while logged on to manager)
docker-machine ssh myvm2 "docker swarm leave"  # Make the worker leave the swarm
docker-machine ssh myvm1 "docker swarm leave -f" # Make master leave, kill swarm
docker-machine ls # list VMs, asterisk shows which VM this shell is talking to
docker-machine start myvm1            # Start a VM that is currently not running
docker-machine env myvm1      # show environment variables and command for myvm1
eval $(docker-machine env myvm1)         # Mac command to connect shell to myvm1
& "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env myvm1 | Invoke-Expression   # Windows command to connect shell to myvm1
docker stack deploy -c <file> <app>  # Deploy an app; command shell must be set to talk to manager (myvm1), uses local Compose file
docker-machine scp docker-compose.yml myvm1:~ # Copy file to node's home dir (only required if you use ssh to connect to manager and deploy the app)
docker-machine ssh myvm1 "docker stack deploy -c <file> <app>"   # Deploy an app using ssh (you must have first copied the Compose file to myvm1)
eval $(docker-machine env -u)     # Disconnect shell from VMs, use native docker
docker-machine stop $(docker-machine ls -q)               # Stop all running VMs
docker-machine rm $(docker-machine ls -q) # Delete all VMs and their disk images
```