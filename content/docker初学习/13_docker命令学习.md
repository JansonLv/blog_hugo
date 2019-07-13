## Docker运行参考
Docker在隔离的容器中运行进程。容器是在主机上运行的进程。主机可以是本地的或远程的。当一个操作符执行时docker run，运行的容器进程是孤立的，因为它有自己的文件系统，它自己的网络，以及它自己独立于主机的独立进程树。

此页面详细说明了如何使用该docker run命令在运行时定义容器的资源。

### 一般形式
基本docker run命令采用以下形式：

$ docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [COMMAND] [ARG...]
该docker run命令必须指定一个IMAGE 以从中派生容器。图像开发人员可以定义与以下相关的图像默认值：

* 分离或前景运行
* 容器识别
* 网络设置
* CPU和内存的运行时限制
随着docker run [OPTIONS]操作者可以添加或覆盖由开发者设置的图像的默认值。此外，运算符可以覆盖Docker运行时本身设置的几乎所有默认值。操作员覆盖图像和Docker运行时默认值的能力是运行具有比任何其他docker命令更多选项的原因 。

要了解如何解释类​​型[OPTIONS]，请参阅选项类型。

> 注意：根据您的Docker系统配置，可能需要在docker run命令前加上sudo。为避免必须使用sudo该docker命令，系统管理员可以创建一个名为的Unix组docker并向其添加用户。有关此配置的更多信息，请参阅适用于您的操作系统的Docker安装文档。

选项参数选择
只有操作员（执行人员docker run）才能设置以下选项。

* 分离vs前景
  * 分离（-d）
  * 前景
* 集装箱识别
    * 名称（ - 名称）
    * PID等价物
* IPC设置（--ipc）
* 网络设置
* 重启政策（--restart）
* 清理（​​--rm）
* 资源的运行时约束
* 运行时权限和Linux功能

### 分离vs前台
启动Docker容器时，必须首先确定是否要在“分离”模式或默认前台模式下在后台运行容器：
```
-d=false: Detached mode: Run container in the background, print new container id
```
#### 分离（-d）
要以分离模式启动容器，请使用-d=true或仅使用-d选项。按照设计，当用于运行容器的根进程退出时，以分离模式启动的容器将退出，除非您还指定了该--rm选项。如果使用 -dwith --rm，则在退出时或守护程序退出时删除容器，以先发生者为准。

不要将service x start命令传递给分离的容器。例如，此命令尝试启动该nginx服务。
```
$ docker run -d -p 80:80 my_image service nginx start
```
这成功启动nginx了容器内的服务。但是，它失败了分离的容器范例，因为root process（service nginx start）返回并且分离的容器按设计停止。因此， nginx服务已启动但无法使用。相反，要启动nginxWeb服务器等进程，请执行以下操作：
```
$ docker run -d -p 80:80 my_image nginx -g 'daemon off;'
```
要使用分离的容器执行输入/输出，请使用网络连接或共享卷。这些是必需的，因为容器不再监听docker run运行的命令行。

要重新连接到分离的容器，请使用 **docker attach**命令。

#### 前景
在前台模式（-d未指定默认值）时，docker run可以在容器中启动进程并将控制台附加到进程的标准输入，输出和标准错误。它甚至可以伪装成TTY（这是大多数命令行可执行文件所期望的）并传递信号。所有这些都是可配置的：
```
-a=[]           : Attach to `STDIN`, `STDOUT` and/or `STDERR`
-t              : Allocate a pseudo-tty
--sig-proxy=true: Proxy all received signals to the process (non-TTY mode only)
-i              : Keep STDIN open even if not attached
```
如果你没有指定，-a那么Doc​​ker将附加到stdout和stderr 。您可以指定其中三个标准流（STDIN，STDOUT， STDERR）你想，而不是连接，如：
```
$ docker run -a stdin -a stdout -i -t ubuntu /bin/bash
```
对于交互式进程（如shell），必须-i -t一起使用才能为容器进程分配tty。-i -t通常-it 会在后面的例子中看到。-t当客户端从管道接收标准输入时，禁止指定，如：
```
$ echo test | docker run -i busybox cat
```
注意：在容器内作为PID 1运行的进程由Linux专门处理：它忽略具有默认操作的任何信号。因此，该过程不会终止SIGINT或SIGTERM除非它被编码为这样做。

### 容器识别
#### 名称 -name
操作员可以通过三种方式识别容器：

|标识符类型|	示例值|
| - | - |
|UUID长标识符|	“f78375b1c487e03c9438c729345e54db9d20cfa2ac1fc3494b6eb60872e74778”|
|UUID短标识符|	“f78375b1c487”|
|name|	“evil_ptolemy”|
UUID标识符来自Docker守护程序。如果未使用该--name选项指定容器名称，则守护程序会为您生成随机字符串名称。定义a name可以是为容器添加含义的便捷方式。如果指定a name，则可以在引用Docker网络中的容器时使用它。这适用于后台和前台Docker容器。

> 注意：链接自定义bridge网络上的容器以按名称进行通信。

#### PID等价物
最后，为了帮助实现自动化，您可以让Docker将容器ID写入您选择的文件中。这类似于某些程序可能会将其进程ID写入文件（您已将其视为PID文件）：
```
--cidfile="": Write the container ID to the file
```
#### 图像[:lable]
虽然不是严格意义上的标识容器，但您可以通过添加image[:tag]命令来指定要运行容器的映像版本。例如，docker run ubuntu:14.04。

#### 图像[@digest]
使用v2或更高版本图像格式的图像具有称为摘要的内容可寻址标识符。只要用于生成图像的输入不变，摘要值就是可预测和可引用的。

以下示例alpine使用sha256:9cacb71397b640eca97488cf08582ae4e4068513101088e9f96c9814bfda95e0摘要从映像 运行容器：
```
$ docker run alpine@sha256:9cacb71397b640eca97488cf08582ae4e4068513101088e9f96c9814bfda95e0 date
```

### PID设置（--pid）
```
--pid=""  : Set the PID (Process) Namespace mode for the container,
             'container:\<name|id>': joins another container's PID namespace
             'host': use the host's PID namespace inside the container
```
默认情况下，所有容器都启用了PID命名空间。

PID命名空间提供进程分离。PID命名空间删除系统进程的视图，并允许重用进程ID，包括pid 1。

功能：在某些情况下，您希望容器共享主机的进程名称空间，基本上允许容器内的进程查看系统上的所有进程。例如，您可以使用strace或等调试工具构建容器gdb，但是在调试容器内的进程时希望使用这些工具。

#### 示例：在容器内运行htop
创建此Dockerfile：

FROM alpine:latest
RUN apk add --update htop && rm -rf /var/cache/apk/*
CMD ["htop"]
构建Dockerfile并将图像标记为myhtop：

$ docker build -t myhtop .
使用以下命令htop在容器内运行：

$ docker run -it --rm --pid=host myhtop
加入另一个容器的pid命名空间可用于调试该容器。

#### 例
启动运行redis服务器的容器：
```
$ docker run --name my-redis -d redis
```
通过运行另一个包含strace的容器来调试redis容器：
```
$ docker run -it --pid=container:my-redis my_strace_docker_image bash
$ strace -p 1
```

### UTS设置（ --uts ）
```
--uts=""  : Set the UTS namespace mode for the container,
       'host': use the host's UTS namespace inside the container
```
UTS名称空间用于设置主机名和对该名称空间中正在运行的进程可见的域。默认情况下，所有容器（包括那些容器）--network=host都有自己的UTS命名空间。该host设置将导致容器使用与主机相同的UTS名称空间。请注意， --hostname在hostUTS模式下无效。

如果您希望容器的主机名随主机的主机名更改而更改，您可能希望与主机共享UTS命名空间。更高级的用例是从容器更改主机的主机名。

IPC设置（--ipc）
```
--ipc="MODE"  : Set the IPC mode for the container
```
接受以下值：

|值	|描述|
| - | - |
|""|	使用守护程序的默认值。|
|none|	拥有私有IPC命名空间，未安装/ dev / shm。|
|private	|拥有私有IPC名称空间。|
|shareable	|拥有私有IPC命名空间，可以与其他容器共享。|
|“container：\<_name-or-ID_>”|	加入另一个（“可共享”）容器的IPC命名空间。|
|host|	使用主机系统的IPC命名空间。|
如果未指定，则使用守护程序缺省值，该守护程序可以是"private" 或"shareable"，具体取决于守护程序版本和配置。

IPC（POSIX / SysV IPC）命名空间提供命名共享内存段，信号量和消息队列的分离。

共享内存段用于以内存速度加速进程间通信，而不是通过管道或通过网络堆栈。共享内存通常被数据库和定制构建（通常是C / OpenMPI，C ++ /使用boost库）用于科学计算和金融服务行业的高性能应用程序使用。如果将这些类型的应用程序分解为多个容器，则可能需要使用"shareable"主要（即“捐赠者”）容器的模式以及"container:\<donor-name-or-ID>"其他容器来共享容器的IPC机制。

### 网络设置
```
--dns=[]           : Set custom dns servers for the container
--network="bridge" : Connect a container to a network
                      'bridge': create a network stack on the default Docker bridge
                      'none': no networking
                      'container:<name|id>': reuse another container's network stack
                      'host': use the Docker host network stack
                      '<network-name>|<network-id>': connect to a user-defined network
--network-alias=[] : Add network-scoped alias for the container
--add-host=""      : Add a line to /etc/hosts (host:IP)
--mac-address=""   : Sets the container's Ethernet device's MAC address
--ip=""            : Sets the container's Ethernet device's IPv4 address
--ip6=""           : Sets the container's Ethernet device's IPv6 address
--link-local-ip=[] : Sets one or more container's Ethernet device's link local IPv4/IPv6 addresses
```
默认情况下，所有容器都启用了网络，并且可以进行任何传出连接。操作员可以完全禁用网络docker run --network none，禁用所有传入和传出网络。在这样的情况下，你将I / O通过文件或执行 STDIN和STDOUT只。

发布端口和链接到其他容器仅适用于默认（桥接）。链接功能是一项传统功能。您应该总是喜欢使用Docker网络驱动程序而不是链接。

默认情况下，您的容器将使用与主机相同的DNS服务器，但您可以使用--dns。

默认情况下，使用分配给容器的IP地址生成MAC地址。您可以通过--mac-address参数（格式12:34:56:78:9a:bc:) 提供MAC地址来明确设置容器的MAC地址。请注意，Docker不会检查手动指定的MAC地址是否唯一。

支持的网络：

|网络|	描述|
| - | - |
|none|	容器中没有网络。|
|bridge（默认）	|通过veth接口将容器连接到网桥。|
|host|	在容器内使用主机的网络堆栈。|
|container：\<name \| id\>|	使用通过其名称或ID指定的另一个容器的网络堆栈。|
|network|	将容器连接到用户创建的网络（使用docker network create命令）|

##### 网络：没有
随着网络是none一个容器将无法访问任何外部路由。容器仍将 loopback在容器中启用接口，但它没有任何到外部流量的路由。

##### bridge
将网络设置为bridge容器将使用docker的默认网络设置。在主机上设置桥，通常命名为 docker0，并且veth将为容器创建一对接口。该veth对的一侧将保留在连接到桥的主机上，而另一侧除了loopback接口外还将放置在容器的命名空间内。将为网桥上的容器分配IP地址，并通过此网桥将流量路由到容器。

容器默认情况下可以通过其 **IP地址** 进行通信。要按名称进行通信，必须将它们链接起来。

##### 网络：host
将网络设置为host容器将共享主机的网络堆栈，并且主机的所有接口都可供容器使用。容器的主机名将与主机系统上的主机名匹配。请注意，--mac-address在hostnetmode中无效。即使在host 网络模式下，默认情况下容器也有自己的UTS命名空间。因此 --hostname在host网络模式下是允许的，并且只会更改容器内的主机名。类似--hostname的--add-host，--dns，--dns-search，和 --dns-option选项可以用在host网络模式。这些选项更新 /etc/hosts或/etc/resolv.conf在容器内。没有变化的，以制作 /etc/hosts，并/etc/resolv.conf在主机上。

与默认bridge模式相比，该host模式提供了明显 更好的网络性能，因为它使用主机的本机网络堆栈，而网桥必须通过docker守护程序进行一级虚拟化。建议在网络性能至关重要时以此模式运行容器，例如生产负载均衡器或高性能Web服务器。

> 注意：--network="host"使容器可以完全访问本地系统服务，例如D-bus，因此被认为是不安全的。

##### 网络：CONTAINER
将网络设置为container容器将共享另一个容器的网络堆栈。必须以格式提供另一个容器的名称--network container:\<name\|id>。请注意，--add-host --hostname --dns --dns-search --dns-option并--mac-address在无效的container网络模式，并且--publish --publish-all --expose也是无效的container网络模式。

使用Redis绑定运行Redis容器localhost然后运行redis-cli命令并通过localhost接口连接到Redis服务器的 示例。
```
$ docker run -d --name redis example/redis --bind 127.0.0.1
$ # use the redis container's network stack to access localhost
$ docker run --rm -it --network container:redis example/redis-cli -h 127.0.0.1
```

##### 用户定义的网络
您可以使用Docker网络驱动程序或外部网络驱动程序插件创建网络。您可以将多个容器连接到同一网络。一旦连接到用户定义的网络，容器就可以仅使用另一个容器的IP地址或名称轻松地进行通信。

对于overlay支持多主机连接的网络或自定义插件，连接到同一多主机网络但从不同引擎启动的容器也可以通过这种方式进行通信。

以下示例使用内置bridge网络驱动程序创建网络，并在创建的网络中运行容器
```
$ docker network create -d bridge my-net
$ docker run --network=my-net -itd --name=container3 busybox
```

#### 管理/ etc / hosts
您的容器将有行，/etc/hosts其中定义容器本身的主机名以及localhost一些其他常见的东西。该 --add-host标志可用于添加其他行/etc/hosts。
```
$ docker run -it --add-host db-static:86.75.30.9 ubuntu cat /etc/hosts
172.17.0.22     09d03f76bf2c
fe00::0         ip6-localnet
ff00::0         ip6-mcastprefix
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
127.0.0.1       localhost
::1	            localhost ip6-localhost ip6-loopback
86.75.30.9      db-static
```
如果容器连接到默认桥接网络和linked 其他容器，则/etc/hosts使用链接容器的名称更新容器的文件。

> 注意由于Docker可能会更新容器的/etc/hosts文件，因此可能存在容器内的进程最终读取空/etc/hosts文件或不完整文件的情况。在大多数情况下，再次重试读取应该可以解决问题。