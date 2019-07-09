# Network 

## Network drivers
docker的网络子系统默认是可插拔的，默认提供一下几个驱动程序：
1. bridge：默认网络驱动程序。如果未指定驱动程序，则这是要创建的网络类型。当应用程序在需要通信的独立容器中运行时，通常会使用桥接网络。

2. host：对于独立容器，删除容器和Docker主机之间的网络隔离，并直接使用主机的网络。host 仅适用于Docker 17.06及更高版本的swarm服务。

3. overlay：覆盖网络将多个Docker守护程序连接在一起，并使群集服务能够相互通信。还可以使用覆盖网络来促进群集服务和独立容器之间的通信，或者在不同Docker守护程序上的两个独立容器之间进行通信。此策略消除了在这些容器之间执行OS级别路由的需要。请参阅覆盖网络。

4. macvlan：Macvlan网络允许为容器分配MAC地址，使其显示为网络上的物理设备。Docker守护程序通过其MAC地址将流量路由到容器。macvlan 在处理期望直接连接到物理网络的传统应用程序时，使用驱动程序有时是最佳选择，而不是通过Docker主机的网络堆栈进行路由。见 Macvlan网络。

5. none：对于此容器，禁用所有网络。通常与自定义网络驱动程序一起使用。none不适用于群组服务。请参阅 禁用容器网络。

6. 网络插件：可以使用Docker安装和使用第三方网络插件。这些插件可从 [Docker Hub](https://hub.docker.com/search?category=network&q=&type=plugin) 或第三方供应商处获得。有关安装和使用给定网络插件的信息，请参阅供应商的文档。

### 应用场景
* 当需要多个容器在同一个Docker主机上进行通信时，**User-defined bridge networks**是最佳选择。
* 当不与主机网络Docker隔离时，而希望隔离容器的其他方面时，**host**网络是最好的。
* 当需要在不同Docker主机上运行的容器进行通信时，或者当多个应用程序使用swarm服务协同工作时，**overlay**网络是最佳选择。
* 当从VM设置迁移或需要容器看起来像网络上的物理主机时，**Macvlan**网络是最佳的，每个主机都具有唯一的MAC地址。
*第三方网络插件允许将Docker与专用网络堆栈集成。

## bridge networks
在网络方面，桥接网络是在网络段之间转发流量的链路层设备。网桥可以是硬件设备或在主机内核中运行的软件设备。

就Docker而言，桥接网络使用软件桥，该桥接器允许连接到同一桥接网络的容器进行通信，同时提供与未连接到该桥接网络的容器的隔离。Docker桥驱动程序自动在主机中安装规则，以便不同网桥上的容器无法直接相互通信。

桥接网络适用于在同一个 Docker守护程序主机上运行的容器。对于在不同Docker守护程序主机上运行的容器之间的通信，可以在操作系统级别管理路由，也可以使用覆盖网络。

启动Docker时，会自动创建默认桥接网络（也称为bridge），并且除非另行指定，否则新启动的容器将连接到该网络。还可以创建用户定义的自定义网桥。用户定义的网桥优于默认bridge 网络。

### 用户定义的网桥与默认网桥之间的差异
* **用户定义的桥接器可在容器化应用程序之间提供更好的隔离和互操作性。**
    连接到同一用户定义的网桥的容器会自动将所有端口相互暴露，并且不会向外界显示任何端口。这使得容器化应用程序可以轻松地相互通信，而不会意外地打开对外界的访问。

    想象一下具有Web前端和数据库后端的应用程序。外部世界需要访问Web前端（可能在端口80上），但只有后端本身需要访问数据库主机和端口。使用用户定义的网桥，只需要打开Web端口，并且数据库应用程序不需要打开任何端口，因为Web前端可以通过用户定义的网桥访问它。

    如果在默认网桥上运行相同的应用程序堆栈，则需要打开Web端口和数据库端口，并使用 每个的标记-p或--publish标记。这意味着Docker主机需要通过其他方式阻止对数据库端口的访问。

* **用户定义的桥接器在容器之间提供自动DNS解析。**

    默认网桥上的容器只能通过IP地址相互访问，除非使用被认为是遗留的--link选项。在用户定义的桥接网络上，容器可以通过名称或别名相互解析。

    想象一下与前一点相同的应用程序，具有Web前端和数据库后端。如果你打电话给你的容器web和db，Web容器可以在连接到数据库容器db，无论哪个码头工人托管应用程序堆栈上运行。

    如果在默认桥接网络上运行相同的应用程序堆栈，则需要在容器之间手动创建链接（使用旧--link 标志）。这些链接需要在两个方向上创建，因此可以看到这对于需要通信的两个以上容器而言变得复杂。或者，可以操作/etc/hosts容器中的文件，但这会产生难以调试的问题。

* **容器可以在运行中与用户定义的网络连接和分离。**

    在容器的生命周期中，可以动态地将其与用户定义的网络连接或断开连接。要从默认桥接网络中删除容器，需要停止容器并使用不同的网络选项重新创建容器。

* **每个用户定义的网络都会创建一个可配置的网桥。**

    如果容器使用默认网桥，则可以对其进行配置，但所有容器都使用相同的设置，例如MTU和iptables规则。此外，配置默认桥接网络发生在Docker本身之外，并且需要重新启动Docker。

    使用创建和配置用户定义的网桥 docker network create。如果不同的应用程序组具有不同的网络要求，则可以在创建时单独配置每个用户定义的网桥。

* **默认桥接网络上的链接容器共享环境变量。**

    最初，在两个容器之间共享环境变量的唯一方法是使用--link标志链接它们。用户定义的网络无法实现这种类型的变量共享。但是有更好的方法来共享环境变量。一些想法：

    * 多个容器可以使用Docker卷装入包含共享信息的文件或目录。

    * 可以一起启动多个容器docker-compose，并且compose文件可以定义共享变量。

    * 可以使用swarm服务而不是独立容器，并利用共享机密和 配置。

连接到同一用户定义的网桥的容器有效地将所有端口相互暴露。对于可以访问不同网络上的容器或非Docker主机的端口，必须使用-p或者--publish标志发布该端口。

### 管理用户定义的桥
使用此docker network create命令可以创建用户定义的桥接网络。

    $ docker network create my-net
可以指定子网，IP地址范围，网关和其他选项。有关详细信息，请参阅 docker network create reference或输出docker network create --help。

使用此docker network rm命令删除用户定义的桥接网络。如果容器当前已连接到网络， 请先断开它们 。

    $ docker network rm my-net


### 将容器连接到用户定义的网桥
创建新容器时，可以指定一个或多个--network标志。此示例将Nginx容器连接到my-net网络。它还将容器中的端口80发布到Docker主机上的端口8080，因此外部客户端可以访问该端口。连接到my-net 网络的任何其他容器都可以访问my-nginx容器上的所有端口，反之亦然。

    $ docker create --name my-nginx \
    --network my-net \
    --publish 8080:80 \
    nginx:latest
要将正在运行的容器连接到现有的用户定义的桥，请使用该 docker network connect命令。以下命令将已在运行的my-nginx容器连接 到已存在的my-net网络：

    $ docker network connect my-net my-nginx
断开容器与用户定义的桥接器的连接
要断开正在运行的容器与用户定义的桥接器的连接，请使用该docker network disconnect命令。以下命令将my-nginx 容器与my-net网络断开连接。

    $ docker network disconnect my-net my-nginx

### 使用IPv6
如果需要对Docker容器的IPv6支持，则需要 在创建任何IPv6网络或分配容器IPv6地址之前启用 Docker守护程序上的选项并重新加载其配置。

创建网络时，可以指定--ipv6标志以启用IPv6。无法在默认bridge网络上有选择地禁用IPv6支持。

在Docker容器或swarm服务中使用IPv6之前，需要在Docker守护程序中启用IPv6支持。之后，可以选择将IPv4或IPv6（或两者）与任何容器，服务或网络一起使用。

> 注意：仅在Linux主机上运行的Docker守护程序上支持IPv6网络。

编辑/etc/docker/daemon.json并将ipv6密钥设置为true。
```
{
  "ipv6": true
}
```
保存文件。

重新加载Docker配置文件。

    $ systemctl reload docker
现在可以使用该--ipv6标志创建网络，并使用该--ip6标志分配容器IPv6地址。

###  启用从Docker容器转发到外部世界
默认情况下，来自连接到默认网桥的容器的流量不会转发到外部世界。要启用转发，需要更改两个设置。这些不是Docker命令，它们会影响Docker主机的内核。

配置Linux内核以允许IP转发。

    $ sysctl net.ipv4.conf.all.forwarding=1
将策略的iptables FORWARD策略更改DROP为 ACCEPT。

    $ sudo iptables -P FORWARD ACCEPT
这些设置在重新启动时不会持续存在，因此可能需要将它们添加到启动脚本中。

### 使用默认桥接网络
默认bridge网络被视为Docker的遗留细节，不建议用于生产。配置它是一种手动操作，它有技术缺点。

#### 将容器连接到默认桥接网络
如果未使用该--network标志指定网络，并且指定了网络驱动程序，则默认情况下容器将连接到默认bridge网络。连接到默认bridge网络的容器只能通过IP地址进行通信，除非它们使用旧--link标志进行链接 。

#### 配置默认网桥
要配置默认bridge网络，请在中指定选项daemon.json。这是一个daemon.json指定了几个选项的示例。仅指定需要自定义的设置。
```json
{
  "bip": "192.168.1.5/24",
  "fixed-cidr": "192.168.1.5/25",
  "fixed-cidr-v6": "2001:db8::/64",
  "mtu": 1500,
  "default-gateway": "10.20.1.1",
  "default-gateway-v6": "2001:db8:abcd::89",
  "dns": ["10.20.1.2","10.20.1.3"]
}
```
#### 重新启动Docker以使更改生效。

将IPv6与默认网桥一起使用
如果将Docker配置为支持IPv6（请参阅使用IPv6），则还会自动为IPv6配置默认网桥。与用户定义的网桥不同，无法在默认网桥上有选择地禁用IPv6。



## overlay

overlay会在多个Docker守护程序主机之间创建分布式网络。该网络位于（覆盖）特定于主机的网络之上，允许连接到它的容器（包括群集服务上的容器）安全地进行通信。Docker透明地处理每个数据包与正确的Docker守护程序主机和正确的目标容器的路由。

初始化swarm或将Docker主机加入现有swarm时，会在该Docker主机上创建两个新网络：
* 一个叫做ingress的overlay network，处理与群集服务相关的控制和数据流量。创建群组服务时，如果不将其连接到用户定义的覆盖网络，则ingress 默认情况下会连接到网络。
* 一个名为docker_gwbridge的桥接网络，它将各个Docker守护程序连接到参与swarm的其他守护进程。

可以使用与创建用户定义overlay网络docker network create相同的方式创建用户定义的bridge网络。服务或容器一次可以连接到多个网络。服务或容器只能通过它们各自连接的网络进行通信。

虽然可以将swarm服务和独立容器连接到覆盖网络，但默认行为和配置问题是不同的。因此，本主题的其余部分分为适用于所有覆盖网络的操作，适用于群集服务网络的操作以及适用于独立容器使用的覆盖网络的操作。

### overlay网络的操作
#### 1. 前提条件
* 使用覆盖网络的Docker守护程序的防火墙规则
需要以下端口打开来往于覆盖网络上参与的每个Docker主机的流量：

    * 用于集群管理通信的TCP端口2377
    * TCP和UDP端口7946用于节点之间的通信
    * UDP端口4789用于覆盖网络流量

* 在创建覆盖网络之前，需要将Docker守护程序初始化为群集管理器，docker swarm init或者使用它将其连接到现有群集docker swarm join。这些中的任何一个都会创建默认ingress覆盖网络，默认情况下 由群服务使用。即使从未计划使用群组服务，也需要执行此操作。之后，可以创建其他用户定义的覆盖网络。

要创建用于swarm服务的覆盖网络，请使用如下命令：

    $ docker network create -d overlay my-overlay
要创建可由群集服务或 独立容器用于与在其他Docker守护程序上运行的其他独立容器通信的覆盖网络，请添加--attachable标志：

    $ docker network create -d overlay --attachable my-attachable-overlay
可以指定IP地址范围，子网，网关和其他选项。详情 docker network create --help请见。

### 加密覆盖网络上的流量
默认情况下，使用GCM模式下的AES算法加密所有群集服务管理流量 。群中的管理器节点每隔12小时轮换用于加密八卦数据的密钥。

要加密应用程序数据，请--opt encrypted在创建覆盖网络时添加。这样可以在vxlan级别启用IPSEC加密。此加密会产生不可忽视的性能损失，因此应该在生产中使用它之前测试此选项。

启用覆盖加密后，Docker会在所有节点之间创建IPSEC隧道，在这些节点上为连接到覆盖网络的服务安排任务。这些隧道还在GCM模式下使用AES算法，管理器节点每12小时自动旋转密钥。

### SWARM模式覆盖网络和独立容器
可以将覆盖网络功能与两者一起使用--opt encrypted --attachable ，并将非托管容器附加到该网络：

    $ docker network create --opt encrypted --driver overlay --attachable my-attachable-multi-host-network

### 自定义默认入口网络
大多数用户从不需要配置ingress网络，但Docker 17.05及更高版本允许这样做。如果自动选择的子网与网络上已存在的子网冲突，或者需要自定义其他低级网络设置（如MTU），则此功能非常有用。

自定义ingress网络涉及删除和重新创建它。这通常在在swarm中创建任何服务之前完成。如果具有发布端口的现有服务，则需要先删除这些服务，然后才能删除ingress网络。

在没有ingress网络存在的时间内，不发布端口的现有服务继续运行但不是负载平衡的。这会影响发布端口的服务，例如发布端口80的WordPress服务。
1. 检查ingress网络使用docker network inspect ingress，并删除其容器连接到它的任何服务。这些是发布端口的服务，例如发布端口80的WordPress服务。如果未停止所有此类服务，则下一步失败。

2. 删除现有ingress网络：
```
$ docker network rm ingress

WARNING! Before removing the routing-mesh network, make sure all the nodes
in your swarm run the same docker engine version. Otherwise, removal may not
be effective and functionality of newly created ingress networks will be
impaired.
Are you sure you want to continue? [y/N]
```
使用--ingress标志创建新的覆盖网络，以及要设置的自定义选项。此示例将MTU设置为1200，将子网设置为10.11.0.0/16，并将网关设置为10.11.0.2。
```
$ docker network create \
  --driver overlay \
  --ingress \
  --subnet=10.11.0.0/16 \
  --gateway=10.11.0.2 \
  --opt com.docker.network.driver.mtu=1200 \
  my-ingress
```
注意：可以将ingress网络命名为除了ingress之外的其他内容 ，但只能拥有一个。尝试创建第二个失败。

重新启动在第一步中停止的服务。

### 自定义docker_gwbridge接口
docker_gwbridge是一个虚拟网桥，将overlay network（包括ingress网络）连接到单个Docker守护程序的物理网络。Docker初始化swarm或将Docker主机加入swarm时会自动创建它，但它不是Docker设备。它存在于Docker主机的内核中。如果需要自定义其设置，则必须在将Docker主机加入swarm之前或从群集中临时删除主机之后执行此操作。

1. 停止Docker。

2. 删除现有docker_gwbridge界面。
```
    $ sudo ip link set docker_gwbridge down
    $ sudo ip link del dev docker_gwbridge
```
3. 启动Docker。不要加入或初始化群。

4. docker_gwbridge使用docker network create命令使用自定义设置手动创建或重新创建桥。此示例使用子网10.11.0.0/16。有关可自定义选项的完整列表，请参阅Bridge驱动程序选项。
```
$ docker network create \
--subnet 10.11.0.0/16 \
--opt com.docker.network.bridge.name=docker_gwbridge \
--opt com.docker.network.bridge.enable_icc=false \
--opt com.docker.network.bridge.enable_ip_masquerade=true \
docker_gwbridge
```
5. 初始化或加入群。由于桥已经存在，Docker不会使用自动设置创建它。

### 群组服务的操作
#### 在覆盖网络上发布端口
连接到同一覆盖网络的群集服务有效地将所有端口相互暴露。对于可在服务外部访问的端口，必须使用或标记on 或发布该端口。支持遗留冒号分隔语法和较新的逗号分隔值语法。较长的语法是首选，因为它有点自我记录。-p --publish docker service createdocker service update

#### 绕过群集服务的路由网格
默认情况下，发布端口的swarm服务使用路由网格来实现。当连接到任何swarm节点上的已发布端口（无论它是否正在运行给定服务）时，将被透明地重定向到正在运行该服务的worker。实际上，Docker充当群服务的负载均衡器。使用路由网格的服务以虚拟IP（VIP）模式运行。即使在每个节点上运行的服务（通过--mode global 标志）也使用路由网格。使用路由网格时，无法保证哪个Docker节点服务客户端请求。

要绕过路由网格，可以使用DNS循环（DNSRR）模式启动服务，方法是将--endpoint-mode标志设置为dnsrr。必须在服务前面运行自己的负载均衡器。Docker主机上的服务名称的DNS查询返回运行该服务的节点的IP地址列表。配置负载均衡器以使用此列表并平衡节点之间的流量。

#### 单独的控制和数据流量
默认情况下，与群组管理相关的控制流量以及进出应用程序的流量都在同一网络上运行，尽管群集控制流量已加密。可以将Docker配置为使用单独的网络接口来处理两种不同类型的流量。当你初始化或者加入群，指定--advertise-addr和--datapath-addr分别。必须为加入群的每个节点执行此操作。

### overlay上独立容器的操作
#### 将独立容器连接到覆盖网络
ingress创建的网络没有--attachable标志，这意味着只有swarm服务可以使用它，而不是独立的容器。可以将独立容器连接到使用该--attachable标志创建的用户定义的覆盖网络。这使得在不同Docker守护程序上运行的独立容器能够进行通信，而无需在各个Docker守护程序主机上设置路由。
#### 容器发现
在大多数情况下，应该连接到服务名称，该名称是负载平衡的，并由支持该服务的所有容器（“任务”）处理。要获取支持该服务的所有任务的列表，请执行DNS查找tasks.\<service-name>.

### host 主机网络
如果host对容器使用网络驱动程序，则该容器的网络堆栈不会与Docker主机隔离。例如，如果运行绑定到端口80 host的容器并使用网络，则容器的应用程序将在主机IP地址的端口80上可用。

主机网络驱动程序仅适用于Linux主机，并且不支持Docker Desktop for Mac，Docker Desktop for Windows或Docker EE for Windows Server。

在Docker 17.06及更高版本中，还可以host通过传递--network host给docker container create命令将网络用于群组服务。在这种情况下，控制流量（与管理群集和服务相关的流量）仍然通过覆盖网络发送，但各个群集服务容器使用Docker守护程序的主机网络和端口发送数据。这会产生一些额外的限制。例如，如果服务容器绑定到端口80，则只有一个服务容器可以在给定的swarm节点上运行。

如果容器或服务未发布端口，则主机网络无效。

### macvlan使用macvlan网络
某些应用程序，尤其是遗留应用程序或监视网络流量的应用程序，希望直接连接到物理网络。在这种情况下，可以使用macvlan网络驱动程序为每个容器的虚拟网络接口分配MAC地址，使其看起来像是直接连接到物理网络的物理网络接口。在这种情况下，需要在Docker主机上指定一个物理接口，用于该接口macvlan以及子网和网关macvlan。甚至可以macvlan使用不同的物理网络接口隔网络。请记住以下事项：

* 由于IP地址耗尽或“VLAN传播”，很容易无意中损坏网络，在这种情况下，网络中存在大量不合适的MAC地址。

* 网络设备需要能够处理“混杂模式”，其中一个物理接口可以分配多个MAC地址。

* 如果应用程序可以使用桥接器（在单个Docker主机上）或覆盖层（跨多个Docker主机进行通信），那么从长远来看，这些解决方案可能会更好。

#### 创建一个macvlan网络
创建macvlan网络时，它可以处于桥接模式或802.1q中继桥接模式。

* 在桥接模式下，macvlan流量通过主机上的物理设备。

* 在802.1q中继桥接模式下，流量通过Docker在运行中创建的802.1q子接口。这允许在更细粒度的级别上控制路由和筛选。

#### 桥接模式
要创建macvlan与给定物理网络接口桥接的网络，请使用--driver macvlan该docker network create命令。还需要指定parent，这是流量将在Docker主机上实际通过的接口。
```
$ docker network create -d macvlan \
  --subnet=172.16.86.0/24 \
  --gateway=172.16.86.1 \
  -o parent=eth0 pub_net
```
如果需要排除在macvlan网络中使用的IP地址，例如当已使用给定的IP地址时，请使用--aux-addresses：
```
$ docker network create -d macvlan \
  --subnet=192.168.32.0/24 \
  --ip-range=192.168.32.128/25 \
  --gateway=192.168.32.254 \
  --aux-address="my-router=192.168.32.129" \
  -o parent=eth0 macnet32
```
#### 802.1q中继桥接模式
如果指定parent包含点的接口名称，例如eth0.50，Docker将其解释为子接口eth0并自动创建子接口。
```
$ docker network create -d macvlan \
    --subnet=192.168.50.0/24 \
    --gateway=192.168.50.1 \
    -o parent=eth0.50 macvlan50
```
#### 使用ipvlan而不是macvlan
在上面的示例中，仍在使用L3网桥。可以ipvlan 改为使用，并获得L2桥接。指定-o ipvlan_mode=l2。
```
$ docker network create -d ipvlan \
    --subnet=192.168.210.0/24 \
    --subnet=192.168.212.0/24 \
    --gateway=192.168.210.254 \
    --gateway=192.168.212.254 \
     -o ipvlan_mode=l2 ipvlan210
```
#### 使用IPv6
如果已将Docker守护程序配置为允许IPv6，则可以使用双栈IPv4 / IPv6 macvlan网络。
```
$ docker network create -d macvlan \
    --subnet=192.168.216.0/24 --subnet=192.168.218.0/24 \
    --gateway=192.168.216.1 --gateway=192.168.218.1 \
    --subnet=2001:db8:abc8::/64 --gateway=2001:db8:abc8::10 \
     -o parent=eth0.218 \
     -o macvlan_mode=bridge macvlan216
```

### 禁用容器的网络连接

如果要完全禁用容器上的网络堆栈，可以--network none在启动容器时使用该标志。在容器内，仅创建环回设备。以下示例说明了这一点。

1. 创建容器。
```
$ docker run --rm -dit \
  --network none \
  --name no-net-alpine \
  alpine:latest \
  ash
```
2. 通过在容器中执行一些常见的网络命令来检查容器的网络堆栈。请注意，没有eth0创建。
```
$ docker exec no-net-alpine ip link show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN qlen 1
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
```
```
$ docker exec no-net-alpine ip route
```
第二个命令返回空，因为没有路由表。

3. 停止容器。它会自动删除，因为它是使用--rm标志创建的。
```
$ docker container rm no-net-alpine
```

## Bridge Network教程

* 使用默认桥接网络演示了如何使用bridgeDocker自动为您设置的默认网络。该网络不是生产系统的最佳选择。

* 使用用户定义的桥接网络显示如何创建和使用自己的自定义桥接网络，以连接在同一Docker主机上运行的容器。建议在生产中运行的独立容器使用此选项。

尽管覆盖网络通常用于群集服务，但Docker 17.06及更高版本允许您将覆盖网络用于独立容器。

### 使用默认 bridge network
1. 打开终端窗口。在执行任何其他操作之前列出当前网络。如果您从未在此Docker守护程序上添加网络或初始化群组，那么您应该看到以下内容。您可能会看到不同的网络，但至少应该看到这些（网络ID将不同）：
```
$ docker network ls

NETWORK ID          NAME                DRIVER              SCOPE
17e324f45964        bridge              bridge              local
6ed54d316334        host                host                local
7092879f2cc8        none                null                local
```

bridge列出了默认网络以及host和none。后两者不是完全成熟的网络，但用于启动直接连接到Docker守护程序主机的网络堆栈的容器，或用于启动没有网络设备的容器。本教程将两个容器连接到bridge网络。

2. 启动两个alpine容器运行ash，这是Alpine的默认shell而不是bash。该-dit标志意味着要首先分离容器（背景），互动（与输入到它的能力），并与TTY（这样你就可以看到输入和输出）。由于您正在启动它，因此您不会立即连接到容器。而是打印容器的ID。由于您尚未指定任何 --network标志，因此容器将连接到默认bridge网络。
```
$ docker run -dit --name alpine1 alpine ash

$ docker run -dit --name alpine2 alpine ash
```
检查两个容器是否实际启动：
```
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
0836095fee86        alpine              "ash"                    7 seconds ago       Up 5 seconds                                            alpine2
328e242261c0        alpine              "ash"                    13 seconds ago      Up 10 seconds                                           alpine1
```
3. 检查bridge网络以查看连接到它的容器。
```
[
    {
        "Name": "bridge",
        "Id": "fe2be5e36428bf142ea408476a3aa19bc30c161d1ce2385b4ec0572b93883374",
        "Created": "2019-06-28T03:45:49.6418433Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "0836095fee86a81df23bd5ceff24e3e4164f51c79d7dea5d895bcbad70ed6d36": {
                "Name": "alpine2",
                "EndpointID": "8df12524265fc3b38a0b24a6cebf68f34f615acaf4bd3d25413cce44346396d9",
                "MacAddress": "02:42:ac:11:00:04",
                "IPv4Address": "172.17.0.4/16",
                "IPv6Address": ""
            },
            "328e242261c00cbe724275f9586dc13b777d1f8d2eaf16b372e78c30409b2cdb": {
                "Name": "alpine1",
                "EndpointID": "06ce879769fdbbc21cf1e901f61d8ce81923666f29a2787f14127123bde083e3",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "75880bdb364037fc4b2c67fd1dae4ee722c959de0596019ce7219a529573ce13": {
                "Name": "postgres",
                "EndpointID": "16bc09a4fcae9cc465b2bef62682b98bfb5abad6cf673b0e9692ed870230bbfd",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```
在顶部附近，bridge列出了有关网络的信息，包括Docker主机和bridge 网络之间的网关的IP地址（172.17.0.1）。在Containers密钥下，列出了每个连接的容器，以及有关其IP地址（172.17.0.3for alpine1和172.17.0.4for alpine2）的信息。

4. 容器在后台运行。使用docker attach 命令连接到alpine1。
```
$ docker attach alpine1

/ #
```
提示符更改为#表示您是root容器中的用户。使用该ip addr show命令显示alpine1容器内的网络接口：
```
ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN qlen 1
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
301: eth0@if302: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```
第一个接口是环回设备。暂时忽略它。请注意，第二个接口具有IP地址172.17.0.2，该地址alpine1与上一步中显示的地址相同。
5. 从内部alpine1，确保您可以通过ping连接到互联网www.baidu.com。该-c 2标志将命令限制为两次ping 尝试。
```
 ping -c 2 baidu.com
PING baidu.com (123.125.114.144): 56 data bytes
64 bytes from 123.125.114.144: seq=0 ttl=37 time=97.175 ms
64 bytes from 123.125.114.144: seq=1 ttl=37 time=96.981 ms

--- baidu.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 96.981/97.078/97.175 ms
````

6. 现在尝试ping第二个容器。首先，通过其IP地址ping它 172.17.0.4：
```
ping -c 2 127.17.0.4
PING 127.17.0.4 (127.17.0.4): 56 data bytes
64 bytes from 127.17.0.4: seq=0 ttl=64 time=0.065 ms
64 bytes from 127.17.0.4: seq=1 ttl=64 time=0.164 ms

--- 127.17.0.4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.065/0.114/0.164 ms
```

这成功了。接下来，尝试alpine2按容器名称ping 容器。这将失败。
```
# ping -c 2 alpine2

ping: bad address 'alpine2'
```
7. 停止并移除两个容器。
```
$ docker container stop alpine1 alpine2
$ docker container rm alpine1 alpine2
```
> ***请记住，bridge不建议将默认网络用于生产。***

### 使用用户定义的网桥
在此示例中，我们再次启动两个alpine容器，但将它们附加到alpine-net我们已创建的用户定义的网络中。这些容器根本没有连接到默认bridge网络。然后我们启动第三个alpine连接到bridge网络但未连接的容器alpine-net，以及alpine连接到两个网络的第四个容器。

1. 创建alpine-net网络。您不需要该--driver bridge标志，因为它是默认值，但此示例显示了如何指定它。
```
$ docker network create --driver bridge alpine-net
```
2. 列出Docker的网络：
```
$ docker network ls
```
检查alpine-net网络。这将显示其IP地址以及没有容器连接到它的事实：
```
$ docker network inspect alpine-net
[
    {
        "Name": "alpine-net",
        "Id": "0283fee29fb297da7c8c27b1e4bf2195de9eb0f7fe0c5587836fd1e0c95caaed",
        "Created": "2019-07-09T06:42:41.2679737Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.27.0.0/16",
                    "Gateway": "172.27.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```
请注意，此网络的网关与网关172.27.0.1的默认网桥相对172.17.0.1。每个人系统上的确切IP地址可能有所不同。

3. 创建您的四个容器。注意--network标志。您只能在docker run命令期间连接到一个网络，因此您需要在 docker network connect以后连接alpine4到bridge 网络。
```
$ docker run -dit --name alpine1 --network alpine-net alpine ash

$ docker run -dit --name alpine2 --network alpine-net alpine ash

$ docker run -dit --name alpine3 alpine ash

$ docker run -dit --name alpine4 --network alpine-net alpine ash

$ docker network connect bridge alpine4
```

4. 查看bridge网络和alpine-net网络：
```
docker network inspect alpine-net
[
    {
        "Name": "alpine-net",
        "Id": "0283fee29fb297da7c8c27b1e4bf2195de9eb0f7fe0c5587836fd1e0c95caaed",
        "Created": "2019-07-09T06:42:41.2679737Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.27.0.0/16",
                    "Gateway": "172.27.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "23af7b258d910394f188f139af9e9e8a58083c0eee867b27cc6a7b21ef37ec81": {
                "Name": "alpine1",
                "EndpointID": "9b6f63166c9382dafceb4f90ae66693f77e498193da9ad30de8f0d5cad2d7078",
                "MacAddress": "02:42:ac:1b:00:02",
                "IPv4Address": "172.27.0.2/16",
                "IPv6Address": ""
            },
            "4ac65b1d4abab441802be227972eb3ab976f73d55b309ba8a75be1a3c26d3e81": {
                "Name": "alpine4",
                "EndpointID": "2d051d0153cfe98e30f72d7e2f3bf086a82be7223da04b641ab5a2d746d92920",
                "MacAddress": "02:42:ac:1b:00:04",
                "IPv4Address": "172.27.0.4/16",
                "IPv6Address": ""
            },
            "c037123df18e326f9fffba03183af32e11051f48610c1f7e65e8dce310ce8409": {
                "Name": "alpine2",
                "EndpointID": "8b54d444058b501325e086b2841482de133945ce2e166398ee6375b13af82463",
                "MacAddress": "02:42:ac:1b:00:03",
                "IPv4Address": "172.27.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```
```
[
    {
        "Name": "bridge",
        "Id": "fe2be5e36428bf142ea408476a3aa19bc30c161d1ce2385b4ec0572b93883374",
        "Created": "2019-06-28T03:45:49.6418433Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "4ac65b1d4abab441802be227972eb3ab976f73d55b309ba8a75be1a3c26d3e81": {
                "Name": "alpine4",
                "EndpointID": "865129e4b49692ae4a8cfc7d3e83127fc92d49d97b4212f2261a6749239bf51f",
                "MacAddress": "02:42:ac:11:00:04",
                "IPv4Address": "172.17.0.4/16",
                "IPv6Address": ""
            },
            "75880bdb364037fc4b2c67fd1dae4ee722c959de0596019ce7219a529573ce13": {
                "Name": "postgres",
                "EndpointID": "16bc09a4fcae9cc465b2bef62682b98bfb5abad6cf673b0e9692ed870230bbfd",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "91f1b3e30b99c1a56e76c5c48547bd90dbd53b0e2176df66b95155d24b784c24": {
                "Name": "alpine3",
                "EndpointID": "b5deef8a530ce8c02e10ab6766f74048d7d9c8d3ff7265ce8916abf1955ce385",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

5. 在用户定义的网络上alpine-net，容器不仅可以通过IP地址进行通信，还可以将容器名称解析为IP地址。此功能称为自动服务发现。让我们连接alpine1并测试一下。alpine1应该能够解析 alpine2和alpine4（和alpine1本身）IP地址。
```
docker container attach alpine1


/ # ping -c 2 alpine1
PING alpine1 (172.27.0.2): 56 data bytes
64 bytes from 172.27.0.2: seq=0 ttl=64 time=0.067 ms
64 bytes from 172.27.0.2: seq=1 ttl=64 time=0.096 ms

--- alpine1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.067/0.081/0.096 ms
/ # ping -c 2 alpine2
PING alpine2 (172.27.0.3): 56 data bytes
64 bytes from 172.27.0.3: seq=0 ttl=64 time=0.159 ms
64 bytes from 172.27.0.3: seq=1 ttl=64 time=0.134 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.134/0.146/0.159 ms
/ # ping -c 2 alpine4
PING alpine4 (172.27.0.4): 56 data bytes
64 bytes from 172.27.0.4: seq=0 ttl=64 time=0.278 ms
64 bytes from 172.27.0.4: seq=1 ttl=64 time=0.150 ms

--- alpine4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.150/0.214/0.278 ms
/ # ping -c 2 alpine3
ping: bad address 'alpine3'
```
在alpine1中1、2、4正常ping通，3无法ping通

测试alpine3
```
docker container attach alpine3


/ # ping -c alpine1
ping: invalid number 'alpine1'
/ # ping -c alpine2
ping: invalid number 'alpine2'
/ # ping -c alpine3
ping: invalid number 'alpine3'
/ # ping -c alpine4
ping: invalid number 'alpine4'
```
在alpine3中，所有都服务名无法ping通，ip可以ping

6. 请记住，alpine4它连接到默认bridge网络和alpine-net。它应该能够到达所有其他容器。但是，您需要alpine3通过其IP地址进行寻址。连接到它并运行测试。
```
$ docker container attach alpine4

# ping -c 2 alpine1

PING alpine1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.074 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.082 ms

--- alpine1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.074/0.078/0.082 ms

# ping -c 2 alpine2

PING alpine2 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.075 ms
64 bytes from 172.18.0.3: seq=1 ttl=64 time=0.080 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.075/0.077/0.080 ms

# ping -c 2 alpine3
ping: bad address 'alpine3'

# ping -c 2 172.17.0.2

PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.089 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.075 ms

--- 172.17.0.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.075/0.082/0.089 ms

# ping -c 2 alpine4

PING alpine4 (172.18.0.4): 56 data bytes
64 bytes from 172.18.0.4: seq=0 ttl=64 time=0.033 ms
64 bytes from 172.18.0.4: seq=1 ttl=64 time=0.064 ms

--- alpine4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.033/0.048/0.064 ms
```

7. 作为最终测试，请确保您的容器都可以通过ping来连接到Internet google.com。你已经附上了，alpine4所以从那里开始尝试。接下来，分离alpine4并连接alpine3 （仅连接到bridge网络），然后重试。最后，连接到alpine1（仅连接到alpine-net网络）并再试一次。
```
# ping -c 2 google.com

PING google.com (172.217.3.174): 56 data bytes
64 bytes from 172.217.3.174: seq=0 ttl=41 time=9.778 ms
64 bytes from 172.217.3.174: seq=1 ttl=41 time=9.634 ms

--- google.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 9.634/9.706/9.778 ms

CTRL+p CTRL+q

$ docker container attach alpine3

# ping -c 2 google.com

PING google.com (172.217.3.174): 56 data bytes
64 bytes from 172.217.3.174: seq=0 ttl=41 time=9.706 ms
64 bytes from 172.217.3.174: seq=1 ttl=41 time=9.851 ms

--- google.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 9.706/9.778/9.851 ms

CTRL+p CTRL+q

$ docker container attach alpine1

# ping -c 2 google.com

PING google.com (172.217.3.174): 56 data bytes
64 bytes from 172.217.3.174: seq=0 ttl=41 time=9.606 ms
64 bytes from 172.217.3.174: seq=1 ttl=41 time=9.603 ms

--- google.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 9.603/9.604/9.606 ms
```
CTRL+p CTRL+q
8. 停止并删除所有容器和alpine-net网络。
```
$ docker container stop alpine1 alpine2 alpine3 alpine4

$ docker container rm alpine1 alpine2 alpine3 alpine4

$ docker network rm alpine-net
```
8. 总结：
在默认bridge情况下，服务名是无法连接的，只有ip地址可以连接，但是ip并不是固定的，因此不适合生成环境。而自定义的bridge是可以直接连接服务名的。

## host网络教程
目标
本教程的目标是启动一个nginx直接绑定到Docker主机上的端口80 的容器。从网络的角度来看，这与隔离级别相同，就好像nginx进程直接在Docker主机上运行而不是在容器中运行一样。但是，在所有其他方式（例如存储，进程命名空间和用户命名空间）中，nginx进程与主机隔离。

### 先决条件
* 此过程要求端口80在Docker主机上可用。要使Nginx侦听其他端口，请参阅该图像的 文档nginx。

* 在host网络驱动程序仅适用于Linux主机。

### 流程

1. 未创建容器前执行ip addr show和创建容器后做对比
```
ip addr show
```
2. 创建并启动容器作为分离进程。该--rm选项意味着一旦退出/停止就移除容器。该-d标志表示启动容器分离（在后台）。
```
docker run --rm -d --network host --name my_nginx nginx
```
3. 通过浏览到http：// localhost：80 /来访问Nginx 。

4. 使用以下命令检查网络堆栈：
* 检查所有网络接口并验证是否未创建新接口。
```
ip addr show
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:1e:ab:59 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe1e:ab59/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:de:18:e9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.99.109/24 brd 192.168.99.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fede:18e9/64 scope link
       valid_lft forever preferred_lft forever
4: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:87:2a:0a:6b brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
 ```
* 使用该netstat命令验证哪个进程绑定到端口80 。您需要使用，sudo因为该进程由Docker守护程序用户拥有，否则您将无法看到其名称或PID。

```shell
sudo netstat -tulpn | grep :80
```

4. 停止容器。它将在使用该--rm选项启动时自动删除。
```shell
docker container stop my_nginx
```


## overlay
本主题包括三个不同的教程。您可以在Linux，Windows或Mac上运行它们，但是对于最后两个，您需要在其他地方运行第二个Docker主机。

1. 使用默认覆盖网络演示了如何使用Docker在初始化或加入群集时自动为您设置的默认覆盖网络。该网络不是生产系统的最佳选择。

2. 使用用户定义的覆盖网络显示如何创建和使用您自己的自定义覆盖网络，以连接服务。建议用于生产中运行的服务。

3. 对独立容器使用覆盖网络 显示如何使用覆盖网络在不同Docker守护程序上的独立容器之间进行通信。

### 先决条件
这些要求您至少拥有一个单节点群，这意味着您已启动Docker并docker swarm init在主机上运行。您也可以在多节点群上运行示例。

最后一个示例需要Docker 17.06或更高版本。

### 默认overlay网络
在此示例中，您将从alpine各个服务容器的角度启动服务并检查网络的特征。

本教程不涉及有关如何实现覆盖网络的操作系统特定细节，而是着重于从服务的角度来看覆盖的功能。
#### 先决条件
本教程需要三个物理或虚拟Docker主机，它们都可以相互通信，所有主机都运行Docker 17.03或更高版本的新安装。本教程假定三台主机在同一网络上运行，不涉及防火墙。

这些主机将被称为manager，worker-1和worker-2。该 manager主机将作为既是经理和工人，这意味着它可以运行服务任务和管理群。worker-1并将worker-2仅作为工人，

如果您没有方便的三台主机，一个简单的解决方案是在云提供商（如Amazon EC2）上设置三个Ubuntu主机，所有这些主机都位于同一网络上，所有通信都允许该网络上的所有主机（使用诸如EC2安全组），然后按照Ubuntu上Docker CE的 安装说明进行操作。

#### 流程
##### 创建集群
在此过程结束时，所有三个Docker主机将连接到群集，并将使用名为的覆盖网络连接在一起ingress。

1. 在manager。初始化群。如果主机只有一个网络接口，则该--advertise-addr标志是可选的。
```
$ docker swarm init --advertise-addr=<IP-ADDRESS-OF-MANAGER>
```
记下打印的文本，因为它包含您将用于加入worker-1和使用worker-2swarm 的标记。将令牌存储在密码管理器中是个好主意。

2. 在worker-1，加入群。如果主机只有一个网络接口，则该--advertise-addr标志是可选的。
```
$ docker swarm join --token <TOKEN> \
--advertise-addr <IP-ADDRESS-OF-WORKER-1> \
<IP-ADDRESS-OF-MANAGER>:2377
```
3. 在worker-2，加入群。如果主机只有一个网络接口，则该--advertise-addr标志是可选的。
```
$ docker swarm join --token <TOKEN> \
--advertise-addr <IP-ADDRESS-OF-WORKER-2> \
<IP-ADDRESS-OF-MANAGER>:2377
```
4. 在manager，列出所有的节点。此命令只能在manager上完成。
```
$ docker node ls

ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
d68ace5iraw6whp7llvgjpu48 *   ip-172-31-34-146    Ready               Active              Leader
nvp5rwavvb8lhdggo8fcf7plg     ip-172-31-35-151    Ready               Active
ouvx2l7qfcxisoyms8mtkgahw     ip-172-31-36-89     Ready               Active
```
您还可以使用该--filter标志按角色进行过滤：
```
$ docker node ls --filter role=manager

ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
d68ace5iraw6whp7llvgjpu48 *   ip-172-31-34-146    Ready               Active              Leader

$ docker node ls --filter role=worker

ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
nvp5rwavvb8lhdggo8fcf7plg     ip-172-31-35-151    Ready               Active
ouvx2l7qfcxisoyms8mtkgahw     ip-172-31-36-89     Ready               Active
```
5. 列出网络，并注意到它们中的每一个现在都有一个名为的覆盖网络和一个名为的桥接网络。
```
$ docker network ls

NETWORK ID          NAME                DRIVER              SCOPE
52e5ef6e6cb9        bridge              bridge              local
1287db03f0d7        docker_gwbridge     bridge              local
dd3949c011d3        host                host                local
zjkg5b7kc7ub        ingress             overlay             swarm
e6c281ce5f45        none                null                local
```
在docker_gwbridge连接ingress网络到主机的网络接口，以便通信可以流入和流出managr和worker。如果您创建集群服务但未指定网络，则它们将默认连接到ingress网络。建议您为可以协同工作的每个应用程序或应用程序组使用单独的覆盖网络。在下一过程中，您将创建两个覆盖网络并将服务连接到每个网络。

##### 创建服务
1. 在manager，创建一个名为的新覆盖网络nginx-net：
```
$ docker network create -d overlay nginx-net
```
您不需要在其他节点上创建覆盖网络，因为当其中一个节点开始运行需要它的服务任务时，它将自动创建。

2. manager，创建连接到的5副本Nginx服务nginx-net。该服务将向外界发布端口80。所有服务任务容器都可以相互通信而无需打开任何端口。

> 注意：只能在manager创建服务。
```
$ docker service create \
  --name my-nginx \
  --publish target=80,published=80 \
  --replicas=5 \
  --network nginx-net \
  nginx
```
默认的发布模式ingress，当你不指定所使用mode的--publish标志，意味着如果你浏览器到端口80上manager，worker-1或者worker-2，即使哪一个主机上没有这个服务，你也会被连接到端口80上的5项服务任务之一，任务当前正在您浏览的节点上运行。如果要使用host模式发布端口 ，可以添加mode=host到--publish输出。但是，您也应该使用--mode global而不是--replicas=5在这种情况下，因为只有一个服务任务可以绑定到某一节点上的给定端口。

3. 运行docker service ls以监视服务启动的进度，这可能需要几秒钟。

4. 检查nginx-net网络manager，worker-1和worker-2。请记住，您不需要手动创建它worker-1， worker-2因为Docker会为您创建它。输出将很长，但请注意Containers和Peers部分。Containers列出从该主机连接到覆盖网络的所有服务任务（或独立容器）。

5. manager，检查服务使用docker service inspect my-nginx 并注意有关服务使用的端口和端点的信息。

6. 创建一个新网络nginx-net-2，然后更新服务以使用此网络而不是nginx-net：
```
$ docker network create -d overlay nginx-net-2
$ docker service update \
  --network-add nginx-net-2 \
  --network-rm nginx-net \
  my-nginx
  ```
7. 运行docker service ls以验证服务是否已更新并且已重新部署所有任务。运行docker network inspect nginx-net以验证没有容器连接到它。运行相同的命令， nginx-net-2并注意所有服务任务容器都连接到它。

> 注意：即使根据需要在swarm工作节点上自动创建覆盖网络，也不会自动删除它们。

8. 清理服务和网络。从manager，运行以下命令。经理将指导工人自动删除网络。
```
$ docker service rm my-nginx
$ docker network rm nginx-net nginx-net-2
```

### 自定义overlay network
#### 先决条件
本教程假设已经设置了swarm并且在manager。

#### 流程

1. 创建用户定义的覆盖网络。
```
$ docker network create -d overlay my-overlay
```
使用覆盖网络启动服务，并将端口80发布到Docker主机上的端口8080。
```
$ docker service create \
  --name my-nginx \
  --network my-overlay \
  --replicas 1 \
  --publish published=8080,target=80 \
  nginx:latest
  ```
通过查看该部分，运行docker network inspect my-overlay并验证my-nginx服务任务是否已连接到该服务任务Containers。
```
[
    {
        "Name": "my-overlay",
        "Id": "ec9a56w2yalfx3am1wgpo0tzs",
        "Created": "2019-07-09T12:27:08.132025172Z",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "10.0.2.0/24",
                    "Gateway": "10.0.2.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "8774fbc496106773dd8b891e62083b4db66466dd237566e5f3024dec45058d2c": {
                "Name": "my-nginx.1.wyu1cd5z5pky7ys3vn5ud3nym",
                "EndpointID": "2a4eaa44addcddfa99421b47ef873928dfea1c2909ee7be47f21a4abca8f11d2",
                "MacAddress": "02:42:0a:00:02:05",
                "IPv4Address": "10.0.2.5/24",
                "IPv6Address": ""
            },
            "lb-my-overlay": {
                "Name": "my-overlay-endpoint",
                "EndpointID": "5c80d9fdfb50f26f811d6597303562b1a5b87b069bdbefe653f9c5d6a90595d4",
                "MacAddress": "02:42:0a:00:02:06",
                "IPv4Address": "10.0.2.6/24",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4099"
        },
        "Labels": {},
        "Peers": [
            {
                "Name": "2bfc64415c0f",
                "IP": "192.168.99.110"
            }
        ]
    }
]
```
删除服务和网络。
```
$ docker service rm my-nginx

$ docker network rm my-overlay
```
### 将覆盖网络用于独立容器
此示例演示DNS容器发现 - 特别是如何使用覆盖网络在不同Docker守护程序上的独立容器之间进行通信。步骤是：

* 在host1，将节点初始化为swarm（管理器）。
* 单击host2，将节点加入swarm（worker）。
* 在host1，创建一个可附加的覆盖网络（test-net）。
* 在host1，运行交互式alpine容器（alpine1）test-net。
* 打开host2，运行交互式，分离的高山容器（alpine2）test-net。
* 在ping host1的会话中。alpine1alpine2
#### 先决条件
对于此测试，您需要两个可以相互通信的不同Docker主机。每个主机必须具有Docker 17.06或更高版本，并在两个Docker主机之间打开以下端口：

* TCP端口2377
* TCP和UDP端口7946
* UDP端口4789
设置此功能的一种简单方法是拥有两个虚拟机（本地或云提供商，如AWS），每个虚拟机都安装并运行Docker。如果您使用的是AWS或类似的云计算平台，最简单的配置是使用一个安全组，打开两个主机之间的所有传入端口和客户端IP地址的SSH端口。

这个例子将我们swarm中的两个节点称为host1和host2。此示例也使用Linux主机，但相同的命令适用于Windows。

##### 流程
1. 创建集群。

打开manager，初始化一个群（如果提示，则用于--advertise-addr 指定与群中其他主机通信的接口的IP地址，例如AWS上的私有IP地址）：
```
$ docker swarm init
Swarm initialized: current node (vz1mm9am11qcmo979tlrlox42) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-5g90q48weqrtqryq4kj6ow0e8xm9wmv9o6vgqc5j320ymybd5c-8ex8j0bc40s6hgvy5ui5gl4gy 172.31.47.252:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
2. 打开host2，按照上面的说明加入群组：
```
$ docker swarm join --token <your_token> <your_ip_address>:2377
This node joined a swarm as a worker.
```
如果节点无法加入群集，则docker swarm join命令超时。要解决，运行docker swarm leave --force上host2，验证您的网络和防火墙设置，然后再试一次。

3. host1创建一个可附加的覆盖网络，称为test-net：
```
$ docker network create --driver=overlay --attachable test-net
7ei2ygjqjq7sf0ixhxae0d3fo
```
> 请注意返回的NETWORK ID - 当您从中连接时，您将再次看到它host2。

4. host1启动一个interactive（-it）容器（alpine1）连接到test-net：
```
$ docker run -it --name alpine1 --network test-net alpine
/ #
```
5. 在host2列出可用的网络 - 通知test-net尚不存在：
```
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
cf27cdfba97b        bridge              bridge              local
196c48b50956        docker_gwbridge     bridge              local
b8b621d2fd5e        host                host                local
zjkg5b7kc7ub        ingress             overlay             swarm
f6bc2d1a6a5c        none                null                local
```
6. 在host2，启动一个连接到test-net的容器（alpine2）：
```
$ docker run -dit --name alpine2 --network test-net alpine
fb635f5ece59563e7b8b99556f816d24e6949a5f6a5b1fbd92ca244db17a4342
```
> 自动DNS容器发现仅适用于唯一的容器名称。

7. host2，验证test-net被创建（和具有相同的网络ID为test-net上host1）：
```
 $ docker network ls
 NETWORK ID          NAME                DRIVER              SCOPE
 ...
 7ei2ygjqjq7s        test-net            overlay             swarm
 ```
8. 在host1，alpine2交互式终端内ping alpine1：
```
/ # ping -c 2 alpine2
PING alpine2 (10.0.3.6): 56 data bytes
64 bytes from 10.0.3.6: seq=0 ttl=64 time=0.793 ms
64 bytes from 10.0.3.6: seq=1 ttl=64 time=0.562 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.562/0.677/0.793 ms
````
两个容器与连接两个主机的覆盖网络通信。如果您运行的另一个容器host2是不沾边的，可以ping alpine1从host2（在这里，我们添加 删除选项自动清理容器）：
```
$ docker run -it --rm --name alpine3 --network test-net alpine
/ # ping -c 2 alpine1
/ # exit
```
9. 打开host1，关闭alpine1会话（也会停止容器）：
```
/ # exit
```
10. 清理容器和网络：

您必须单独停止并删除每个主机上的容器，因为Docker守护程序是独立运行的，并且这些是独立容器。您只需要删除网络，host1因为当您停止 alpine2时host2，test-net消失。

打开host2，停止alpine2，检查test-net已删除，然后删除alpine2：
```
$ docker container stop alpine2
$ docker network ls
$ docker container rm alpine2
```
打开host1，删除alpine1和test-net：
```
$ docker container rm alpine1
$ docker network rm test-net
```


### 使用macvlan网络进行联网
本系列教程涉及连接到macvlan网络的网络独立容器。在这种类型的网络中，Docker主机在其IP地址接受对多个MAC地址的请求，并将这些请求路由到适当的容器。有关其他网络主题，请参阅 概述。

#### 目标
这些教程的目标是设置桥接macvlan网络并将容器连接到它，然后设置802.1q中继macvlan网络并将容器连接到它。

#### 先决条件
* 大多数云提供商阻止macvlan网络。您可能需要物理访问您的网络设备。

* 在macvlan网络驱动程序仅适用于Linux主机，并且不支持多克Mac版桌面，多克尔Windows版桌面，或码头EE的Windows Server。

* 至少需要Linux内核版本3.9，建议使用4.0或更高版本。

* 这些示例假设您的以太网接口是eth0。

#### 桥的例子
在简单的网桥示例中，您的流量通过eth0，Docker使用其MAC地址将流量路由到您的容器。要在网络上联网设备，您的容器似乎物理连接到网络。

1. 创建一个macvlan名为的网络my-macvlan-net。将subnet,, gateway和parent值修改为在您的环境中有意义的值。
```
$ docker network create -d macvlan \
  --subnet=172.16.86.0/24 \
  --gateway=172.16.86.1 \
  -o parent=eth0 \
  my-macvlan-net
```
您可以使用docker network ls和docker network inspect my-macvlan-net 命令来验证网络是否存在并且是macvlan网络。

2. 启动alpine容器并将其连接到my-macvlan-net网络。该 -dit标志在后台启动容器，但让你重视它。该--rm标志表示容器在停止时被移除。
```
$ docker run --rm -itd \
  --network my-macvlan-net \
  --name my-macvlan-alpine \
  alpine:latest \
  ash
```
3. 检查my-macvlan-alpine容器并注意MacAddress密钥中的Networks密钥：

```
$ docker container inspect my-macvlan-alpine
...truncated...
 "Networks": {
                "my-macvlan-net": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": [
                        "3a19ff6a1593"
                    ],
                    "NetworkID": "4390172bffb1ff6cc9edc74c8fbdaa16cd2285d277544c85353bcfad4c6e6878",
                    "EndpointID": "a4d62de35c64c19f1af7e104797b230d7baaf18d4ceb23895f6fb1bc64bc1f36",
                    "Gateway": "172.16.86.1",
                    "IPAddress": "172.16.86.2",
                    "IPPrefixLen": 24,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:10:56:02",
                    "DriverOpts": null
                }
            }
...truncated
```
4. 通过运行几个docker exec命令，查看容器如何看到自己的网络接口。
```
$ docker exec my-macvlan-alpine ip addr show eth0

54: eth0@sit0: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:10:56:02 brd ff:ff:ff:ff:ff:ff
    inet 172.16.86.2/24 brd 172.16.86.255 scope global eth0
       valid_lft forever preferred_lft forever
```
```    
$ docker exec my-macvlan-alpine ip route

default via 172.16.86.1 dev eth0
172.16.86.0/24 dev eth0 scope link  src 172.16.86.2
```
5. 停止容器（Docker因--rm标志而删除它），然后删除网络。
```
$ docker container stop my-macvlan-alpine

$ docker network rm my-macvlan-net
```

#### 802.1q集群桥示例
在802.1q中继网桥示例中，您的流量通过eth0（被叫eth0.10）子接口流动，Docker使用其MAC地址将流量路由到您的容器。要在网络上联网设备，您的容器似乎物理连接到网络。

1. 创建一个macvlan名为的网络my-8021q-macvlan-net。将subnet,, gateway和parent值修改为 在您的环境中有意义的值。
```
$ docker network create -d macvlan \
  --subnet=172.16.86.0/24 \
  --gateway=172.16.86.1 \
  -o parent=eth0.10 \
  my-8021q-macvlan-net
```
2. 您可以使用docker network ls和docker network inspect my-8021q-macvlan-net 命令来验证网络是否存在，是否为macvlan网络，以及是否具有父网络eth0.10。您可以ip addr show在Docker主机上使用以验证接口是否eth0.10存在并具有单独的IP地址

3. 启动alpine容器并将其连接到my-8021q-macvlan-net 网络。该-dit标志在后台启动容器，但让你重视它。该--rm标志表示容器在停止时被移除。
````
$ docker run --rm -itd \
  --network my-8021q-macvlan-net \
  --name my-second-macvlan-alpine \
  alpine:latest \
  ash
```
4. 检查my-second-macvlan-alpine容器并注意MacAddress 密钥中的Networks密钥：
```
$ docker container inspect my-second-macvlan-alpine

...truncated...
"Networks": {
  "my-8021q-macvlan-net": {
      "IPAMConfig": null,
      "Links": null,
      "Aliases": [
          "12f5c3c9ba5c"
      ],
      "NetworkID": "c6203997842e654dd5086abb1133b7e6df627784fec063afcbee5893b2bb64db",
      "EndpointID": "aa08d9aa2353c68e8d2ae0bf0e11ed426ea31ed0dd71c868d22ed0dcf9fc8ae6",
      "Gateway": "172.16.86.1",
      "IPAddress": "172.16.86.2",
      "IPPrefixLen": 24,
      "IPv6Gateway": "",
      "GlobalIPv6Address": "",
      "GlobalIPv6PrefixLen": 0,
      "MacAddress": "02:42:ac:10:56:02",
      "DriverOpts": null
  }
}
...truncated
```
5. 通过运行几个docker exec命令，查看容器如何看到自己的网络接口。

```
$ docker exec my-second-macvlan-alpine ip addr show eth0

11: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
link/ether 02:42:ac:10:56:02 brd ff:ff:ff:ff:ff:ff
inet 172.16.86.2/24 brd 172.16.86.255 scope global eth0
   valid_lft forever preferred_lft forever
$ docker exec my-second-macvlan-alpine ip route

default via 172.16.86.1 dev eth0
172.16.86.0/24 dev eth0 scope link  src 172.16.86.2
```

6. 停止容器（Docker因--rm标志而删除它），然后删除网络。

```
$ docker container stop my-second-macvlan-alpine
$ docker network rm my-8021q-macvlan-net
```

## 相关配置
[ipv6支持](https://docs.docker.com/config/daemon/ipv6/)
[Docker和iptables 防火墙](https://docs.docker.com/network/iptables/)
