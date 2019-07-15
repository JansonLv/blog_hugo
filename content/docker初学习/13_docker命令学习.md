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

### 重启政策（--restart）
使用--restart Docker上的标志运行，您可以指定重启策略，以了解在退出时应该或不应该重新启动容器的方式。

当容器上的重新启动策略处于活动状态时，它将显示为Up 或Restarting中docker ps。使用它docker events来查看有效的重启策略也很有用。

Docker支持以下重启策略：

|策略|	结果|
| - | - |
|no|	退出时不要自动重启容器。这是默认值。|
|on-failure[:max-retries]	|仅当容器以非零退出状态退出时才重新启动。（可选）限制Docker守护程序尝试的重新启动重试次数。|
|always|	无论退出状态如何，始终重新启动容器。当您指定always时，Docker守护程序将尝试无限期地重新启动容器。无论容器的当前状态如何，容器也将始终在守护程序启动时启动。|
|unless-stopped	|	无论退出状态如何，都要重新启动容器，包括在守护程序启动时，除非容器在Docker守护程序停止之前进入停止状态。|
在每次重启之前添加一个不断增加的延迟（前一个延迟的两倍，从100毫秒开始），以防止阻塞服务器。这意味着守护程序将等待100毫秒，然后是200毫秒，400,800,1600等，直到达到on-failure限制，或者您docker stop 或docker rm -f容器。

如果容器成功重新启动（容器启动并运行至少10秒），则延迟将重置为其默认值100 ms。

您可以指定Docker在使用故障时策略时尝试重新启动容器的最大次数。默认情况下，Docker将永远尝试重新启动容器。可以通过获得容器（尝试）重启的次数docker inspect。例如，获取容器“my-container”的重启次数;
```
$ docker inspect -f "{{ .RestartCount }}" my-container
# 2
```
或者，以便最后一次（重新）启动容器;
```
$ docker inspect -f "{{ .State.StartedAt }}" my-container
# 2015-03-04T23:47:07.691840179Z
```
将--restart（重启策略）与--rm（清理）标志组合会导致错误。在容器重启时，连接的客户端将断开连接 请参阅本页后面的使用--rm（清理）标志的示例。

#### 例子
```
$ docker run --restart=always redis
```
这将运行redis与重启策略容器总是 这样，如果容器出口，码头工人将重新启动。

```
$ docker run --restart=on-failure:10 redis
```
这将redis使用重启策略为on-failure 并且最大重启次数为10 来运行容器。如果redis容器以非零退出状态退出连续10次以上，Docker将中止尝试重新启动容器。提供最大重启限制仅对故障时策略有效 。

### 退出状态
退出代码docker run提供有关容器无法运行的原因或退出原因的信息。当docker run退出时非零码时，退出代码遵循chroot标准，见下文：

125如果错误与Docker守护程序本身有关
```
$ docker run --foo busybox; echo $?
# flag provided but not defined: --foo
  See 'docker run --help'.
  125
```
126如果无法调用包含的命令
```
$ docker run busybox /etc; echo $?
# docker: Error response from daemon: Container command '/etc' could not be invoked.
  126
```
127如果找不到包含的命令
```
$ docker run busybox foo; echo $?
# docker: Error response from daemon: Container command 'foo' not found or does not exist.
  127
```
退出代码的包含命令否则
```
$ docker run busybox /bin/sh -c 'exit 3'; echo $?
# 3
```

### 清理（​​--rm）
默认情况下，容器的文件系统即使在容器退出后仍然存在。这使得调试变得更加容易（因为您可以检查最终状态）并且默认情况下会保留所有数据。但是如果你正在运行短期前台进程，这些容器文件系统真的可以堆积起来。如果您希望Docker 在容器退出时自动清理容器并删除文件系统，则可以添加--rm标志：
```
--rm=false: Automatically remove the container when it exits
```
注意：设置--rm标志时，Docker还会在删除容器时删除与容器关联的匿名卷。这与跑步类似docker rm -v my-container。仅删除未指定名称的卷。例如，使用 docker run --rm -v /foo -v awesome:/bar busybox top，/foo将删除音量，但不会删除音量/bar。继承的卷--volumes-from将使用相同的逻辑删除 - 如果使用名称指定原始卷，则不会删除它。

#### 安全配置
```
--security-opt="label=user:USER"     : Set the label user for the container
--security-opt="label=role:ROLE"     : Set the label role for the container
--security-opt="label=type:TYPE"     : Set the label type for the container
--security-opt="label=level:LEVEL"   : Set the label level for the container
--security-opt="label=disable"       : Turn off label confinement for the container
--security-opt="apparmor=PROFILE"    : Set the apparmor profile to be applied to the container
--security-opt="no-new-privileges:true|false"   : Disable/enable container processes from gaining new privileges
--security-opt="seccomp=unconfined"  : Turn off seccomp confinement for the container
--security-opt="seccomp=profile.json": White listed syscalls seccomp Json file to be used as a seccomp filter
```
您可以通过指定--security-opt标志来覆盖每个容器的默认标签方案。在以下命令中指定级别允许您在容器之间共享相同的内容。
```
$ docker run --security-opt label=level:s0:c100,c200 -it fedora bash
```
> 注意：目前不支持自动翻译MLS标签。

要禁用此容器的安全标签与使用该--privileged标志运行 ，请使用以下命令：
```
$ docker run --security-opt label=disable -it fedora bash
```
如果要对容器内的进程实施更严格的安全策略，可以为容器指定备用类型。您可以通过执行以下命令来运行仅允许侦听Apache端口的容器：
```
$ docker run --security-opt label=type:svirt_apache_t -it centos bash
```
> 注意：您必须编写定义svirt_apache_t类型的策略。

如果要阻止容器进程获得其他权限，可以执行以下命令：
```
$ docker run --security-opt no-new-privileges -it centos bash
```
这意味着提升权限的命令将会su或sudo不再有效。它还会导致在删除权限后稍后应用任何seccomp过滤器，这可能意味着您可以拥有更严格的过滤器集。有关更多详细信息，请参阅内核文档。

### 指定init进程
您可以使用该--init标志指示应将init进程用作容器中的PID 1。指定init进程可确保在创建的容器内执行init系统的常规职责，例如收获僵尸进程。

使用的默认init进程是docker-initDocker守护程序进程的系统路径中找到的第一个可执行文件。此docker-init二进制文件包含在默认安装中，由tini支持。

### 指定自定义cgroup
使用该--cgroup-parent标志，您可以传递特定的cgroup来运行容器。这允许您自己创建和管理cgroup。您可以为这些cgroup定义自定义资源，并将容器放在公共父组下。

### 资源的运行时约束
操作员还可以调整容器的性能参数：

|选项|	描述|
| - | - |
|-m， --memory=""|	内存限制（格式:) \<number>[\<unit>]。Number是正整数。单位可以是一个b，k，m，或g。最低为4M。|
|--memory-swap=""	|总内存限制（内存+交换，格式:) \<number>[\<unit>]。Number是正整数。单位可以是一个b，k，m，或g。|
|--memory-reservation=""	|内存软限制（格式:) \<number>[\<unit>]。Number是正整数。单位可以是一个b，k，m，或g。|
|--kernel-memory=""	|内核内存限制（格式:) \<number>[\|<unit>]。Number是正整数。单位可以是一个b，k，m，或g。最低为4M。|
|-c， --cpu-shares=0	|CPU份额（相对权重）|
|--cpus=0.000	|CPU数量。数字是一个小数。0.000表示没有限制。
|--cpu-period=0|	限制CPU CFS（完全公平计划程序）期间|
|--cpuset-cpus=""|	允许执行的CPU（0-3,0,1）|
|--cpuset-mems=""	|允许执行的存储器节点（MEM）（0-3,0,1）。仅对NUMA系统有效。|
|--cpu-quota=0	|限制CPU CFS（完全公平计划程序）配额|
|--cpu-rt-period=0	|限制CPU实时周期。以微秒为单位。需要设置父cgroup并且不能高于父cgroup。还要检查rtprio ulimits。|
|--cpu-rt-runtime=0	|限制CPU实时运行时。以微秒为单位。需要设置父cgroup并且不能高于父cgroup。还要检查rtprio ulimits。|
|--blkio-weight=0	|块IO重量（相对重量）接受10到1000之间的重量值。|
|--blkio-weight-device=""	|块IO重量（相对设备重量，格式：DEVICE_NAME:WEIGHT）|
|--device-read-bps=""|	限制设备的读取速率（格式:) \<device-path>:\<number>[\<unit>]。Number是正整数。单位可以是一个kb，mb或gb。|
|--device-write-bps=""|	限制设备的写入速率（格式:) \<device-path>:\<number>[\<unit>]。Number是正整数。单位可以是一个kb，mb或gb。|
|--device-read-iops=""	|限制设备的读取速率（每秒IO）（格式:) \<device-path>:\<number>。Number是正整数。|
|--device-write-iops=""|	限制设备的写入速率（每秒IO）（格式:) \<device-path>:\<number>。Number是正整数。|
|--oom-kill-disable=false|	是否为容器禁用OOM Killer。|
|--oom-score-adj=0	|调整容器的OOM首选项（-1000到1000）|
|--memory-swappiness=""|	调整容器的内存swappiness行为。接受0到100之间的整数。|
|--shm-size=""	|大小/dev/shm。格式是\<number>\<unit>。number必须大于0。单位是可选的，可以是b（字节），k（千字节），m（兆字节）或g（千兆字节）。如果省略该单位，则系统使用字节。如果完全省略大小，则系统使用64m。|

#### 用户内存限制
我们有四种方法来设置用户内存使用量：

|选项	|结果|
| - | - |
|memory = inf，memory-swap = inf（默认）|	容器没有内存限制。容器可以根据需要使用尽可能多的内存。|
|memory = L <inf，memory-swap = inf|	（指定内存并将内存交换设置为-1）容器不允许使用超过L个字节的内存，但可以根据需要使用尽可能多的交换（如果主机支持交换内存）。|
|memory = L <inf，memory-swap = 2 * L.	|（指定没有内存交换的内存）容器不允许使用超过L字节的内存，交换加内存使用量是其两倍。|
|memory = L <inf，memory-swap = S <inf，L <= S.	|（指定内存和内存交换）容器不允许使用超过L字节的内存，交换加内存使用受S限制。|
#### 例子：
```
$ docker run -it ubuntu:14.04 /bin/bash
```
我们没有设置内存，这意味着容器中的进程可以使用尽可能多的内存和交换内存。
```
$ docker run -it -m 300M --memory-swap -1 ubuntu:14.04 /bin/bash
```
我们设置内存限制和禁用交换内存限制，这意味着容器中的进程可以使用300M内存和所需的交换内存（如果主机支持交换内存）。
```
$ docker run -it -m 300M ubuntu:14.04 /bin/bash
```
我们只设置内存限制，这意味着容器中的进程可以使用300M内存和300M交换内存，默认情况下，总虚拟内存大小（--memory-swap）将被设置为内存的两倍，在这种情况下，内存+ swap将是2 * 300M，因此进程也可以使用300M交换内存。
```
$ docker run -it -m 300M --memory-swap 1G ubuntu:14.04 /bin/bash
```
我们设置了内存和交换内存，因此容器中的进程可以使用300M内存和700M交换内存。

内存预留是一种内存软限制，允许更大的内存共享。在正常情况下，容器可以根据需要使用尽可能多的内存，并且仅受使用-m/ --memory选项设置的硬限制的约束 。设置内存预留时，Docker会检测内存争用或内存不足，并强制容器将其消耗限制为预留限制。

始终将内存预留值设置为低于硬限制，否则硬限制优先。预约0与设置无预约相同。默认情况下（无预留设置），内存预留与硬内存限制相同。

内存预留是一种软限制功能，不保证不会超出限制。相反，该功能尝试确保在内存严重争用时，根据预留提示/设置分配内存。

以下示例将memory（-m）限制为500M，并将内存预留设置为200M。
```
$ docker run -it -m 500M --memory-reservation 200M ubuntu:14.04 /bin/bash
```
在此配置下，当容器消耗的内存超过200M且小于500M时，下一次系统内存回收会尝试将容器内存缩小到200M以下。

以下示例将内存预留设置为1G而没有硬内存限制。
```
$ docker run -it --memory-reservation 1G ubuntu:14.04 /bin/bash
```
容器可以根据需要使用尽可能多的内存。内存预留设置可确保容器长时间不占用太多内存，因为每次内存回收都会将容器的消耗量缩减到预留状态。

默认情况下，如果发生内存不足（OOM）错误，内核会终止容器中的进程。若要更改此行为，请使用该--oom-kill-disable选项。仅在已设置-m/--memory选项的容器上禁用OOM杀手 。如果-m未设置该标志，则可能导致主机内存不足并需要终止主机的系统进程以释放内存。

以下示例将内存限制为100M并禁用此容器的OOM杀手：
```
$ docker run -it -m 100M --oom-kill-disable ubuntu:14.04 /bin/bash
```
以下示例说明了使用该标志的危险方法：
```
$ docker run -it --oom-kill-disable ubuntu:14.04 /bin/bash
```
容器具有无限的内存，这可能导致主机耗尽内存并需要杀死系统进程以释放内存。该--oom-score-adj 参数是可以改变的选择优先权，容器就会被杀死，当系统内存不足，负得分使他们不太可能被杀害，并积极分数的可能性较大。

#### 内核内存限制
内核内存与用户内存根本不同，因为内核内存无法换出。无法交换使容器可能通过占用过多的内核内存来阻止系统服务。内核内存包括：

* 堆栈页面
* 平板页面
* 插座内存压力
* |tcp内存压力
您可以设置内核内存限制来约束这些类​​型的内存。例如，每个进程都会占用一些堆栈页面。通过限制内核内存，可以防止在内核内存使用率过高时创建新进程。

内核内存永远不会完全独立于用户内存。相反，您在用户内存限制的上下文中限制内核内存。假设“U”是用户内存限制，“K”是内核限制。设置限制有三种可能的方法：

|选项|	结果|
| - | - |
|U！= 0，K = inf（默认）|	这是使用内核内存之前已经存在的标准内存限制机制。内核内存完全被忽略。|
|U！= 0，K <U|	内核内存是用户内存的子集。此设置在过量使用每个cgroup的内存总量的部署中很有用。绝对不建议过度使用内核内存限制，因为该框仍然可以用完不可回收的内存。在这种情况下，您可以配置K，以便所有组的总和永远不会大于总内存。然后，以系统的服务质量为代价自由设置U.|
|U！= 0，K> U.|	由于内核内存费也被馈送到用户计数器，因此对于两种内存的容器都会触发回收。此配置为管理员提供统一的内存视图。它对于只想跟踪内核内存使用情况的用户也很有用。|
#### 例子：
```
$ docker run -it -m 500M --kernel-memory 50M ubuntu:14.04 /bin/bash
```
我们设置了内存和内核内存，因此容器中的进程总共可以使用500M内存，在这个500M内存中，它可以是50M内核内存。
```
$ docker run -it --kernel-memory 50M ubuntu:14.04 /bin/bash
```
我们在没有-m的情况下设置内核内存，因此容器中的进程可以使用他们想要的内存，但是它们只能使用50M的内核内存。

#### Swappiness约束
默认情况下，容器的内核可以换出一定比例的匿名页面。要为容器设置此百分比，请指定--memory-swappiness0到100之间的值。值0将关闭匿名页面交换。值100将所有匿名页面设置为可交换。默认情况下，如果您不使用 --memory-swappiness，则内存swappiness值将从父级继承。

例如，您可以设置：
```
$ docker run -it --memory-swappiness=0 ubuntu:14.04 /bin/bash
```
如果--memory-swappiness要保留容器的工作集并避免交换性能损失，设置该选项很有用。

#### CPU份额约束
默认情况下，所有容器都获得相同比例的CPU周期。可以通过相对于所有其他正在运行的容器的权重更改容器的CPU份额权重来修改此比例。

要修改默认值1024的比例，请使用-c或--cpu-shares 标记将权重设置为2或更高。如果设置为0，系统将忽略该值并使用默认值1024。

该比例仅适用于CPU密集型进程运行时。当一个容器中的任务空闲时，其他容器可以使用剩余的CPU时间。实际的CPU时间量将根据系统上运行的容器数量而变化。

例如，考虑三个容器，一个容器的cpu-share为1024，另外两个容器的cpu-share设置为512.当所有三个容器中的进程尝试使用100％的CPU时，第一个容器将获得50％的容量。总CPU时间。如果添加第四个容器，其cpu-share为1024，则第一个容器只占CPU的33％。其余容器分别占CPU的16.5％，16.5％和33％。

在多核系统上，CPU时间的份额分布在所有CPU核心上。即使容器的CPU时间限制在100％以下，它也可以使用每个CPU核心的100％。

例如，考虑具有三个以上内核的系统。如果你开始一个容器{C0}与-c=512运行的一个过程，而另一个容器 {C1}与-c=1024运行的两个过程，这可能会导致CPU份额如下划分：
```
PID    container	CPU	CPU share
100    {C0}		0	100% of CPU0
101    {C1}		1	100% of CPU1
102    {C1}		2	100% of CPU2
```
### CPU周期约束
默认CPU CFS（完全公平调度程序）周期为100毫秒。我们可以 --cpu-period用来设置CPU的周期来限制容器的CPU使用率。通常--cpu-period应该合作--cpu-quota。

例子：
```
$ docker run -it --cpu-period=50000 --cpu-quota=25000 ubuntu:14.04 /bin/bash
```
如果有1个CPU，这意味着容器每50ms可以获得50％的CPU运行时间。

除了使用--cpu-period和--cpu-quota设置CPU周期约束外，还可以--cpus使用浮点数指定以实现相同的目的。例如，如果有1个CPU，那么--cpus=0.5将获得与设置--cpu-period=50000和--cpu-quota=25000（50％CPU）相同的结果。

--cpusis 的默认值0.000，表示没有限制。

有关更多信息，请参阅有关带宽限制的CFS文档。

### Cpuset约束
我们可以设置cpus来允许执行容器。

例子：
```
$ docker run -it --cpuset-cpus="1,3" ubuntu:14.04 /bin/bash
```
这意味着容器中的进程可以在cpu 1和cpu 3上执行。
```
$ docker run -it --cpuset-cpus="0-2" ubuntu:14.04 /bin/bash
```
这意味着容器中的进程可以在cpu 0，cpu 1和cpu 2上执行。

我们可以设置允许执行容器的mems。仅对NUMA系统有效。

例子：
```
$ docker run -it --cpuset-mems="1,3" ubuntu:14.04 /bin/bash
```
此示例将容器中的进程限制为仅使用内存节点1和3中的内存。
```
$ docker run -it --cpuset-mems="0-2" ubuntu:14.04 /bin/bash
```
此示例将容器中的进程限制为仅使用内存节点0,1和2中的内存。

#### CPU配额约束
该--cpu-quota标志限制容器的CPU使用率。默认的0值允许容器占用100％的CPU资源（1个CPU）。CFS（完全公平调度程序）处理执行进程的资源分配，是内核使用的默认Linux调度程序。将此值设置为50000以将容器限制为CPU资源的50％。对于多个CPU，请--cpu-quota根据需要进行调整。有关更多信息，请参阅有关带宽限制的CFS文档。

#### 阻止IO带宽（Blkio）约束
默认情况下，所有容器都获得相同比例的块IO带宽（blkio）。此比例为500.要修改此比例，请使用--blkio-weight标志更改容器的blkio重量相对于所有其他正在运行的容器的权重。

注意： blkio重量设置仅适用于直接IO。目前不支持缓冲IO。

该--blkio-weight标志可以将权重设置为10到1000之间的值。例如，下面的命令创建两个具有不同blkio权重的容器：
```
$ docker run -it --name c1 --blkio-weight 300 ubuntu:14.04 /bin/bash
$ docker run -it --name c2 --blkio-weight 600 ubuntu:14.04 /bin/bash
```
如果您同时在两个容器中阻止IO，例如：
```
$ time dd if=/mnt/zerofile of=test.out bs=1M count=1024 oflag=direct
```
您会发现时间的比例与两个容器的blkio重量的比例相同。

该--blkio-weight-device="DEVICE_NAME:WEIGHT"标志设置特定的设备权重。这DEVICE_NAME:WEIGHT是一个包含冒号分隔的设备名称和权重的字符串。例如，要将/dev/sda设备权重设置为200：
```
$ docker run -it \
    --blkio-weight-device "/dev/sda:200" \
    ubuntu
```
如果同时指定--blkio-weight和--blkio-weight-device，则Docker使用--blkio-weight默认权重作为默认权重，并使用--blkio-weight-device 特定设备上的新值覆盖此默认值。以下示例使用默认权重300并覆盖此默认/dev/sda设置，将权重设置为200：
```
$ docker run -it \
    --blkio-weight 300 \
    --blkio-weight-device "/dev/sda:200" \
    ubuntu
```
该--device-read-bps标志限制了设备的读取速率（每秒字节数）。例如，此命令创建一个容器，并将读取速率限制为1mb 每秒/dev/sda：
```
$ docker run -it --device-read-bps /dev/sda:1mb ubuntu
```
该--device-write-bps标志限制了设备的写入速率（每秒字节数）。例如，此命令创建一个容器并将写入速率限制为1mb 每秒/dev/sda：
```
$ docker run -it --device-write-bps /dev/sda:1mb ubuntu
```
两个标志都采用\<device-path>:\<limit>[unit]格式限制。读取和写入速率都必须是正整数。您可以以kb （千字节），mb（兆字节）或gb（千兆字节）指定速率。

该--device-read-iops标志限制了设备的读取速率（每秒IO）。例如，此命令创建一个容器，并将读取速率限制为每秒 1000IO /dev/sda：
```
$ docker run -ti --device-read-iops /dev/sda:1000 ubuntu
```
该--device-write-iops标志将写入速率（每秒IO）限制为设备。例如，此命令创建一个容器并将写入速率限制为每秒 1000IO /dev/sda：
```
$ docker run -ti --device-write-iops /dev/sda:1000 ubuntu
```
两个标志都采用\<device-path>:\<limit>格式限制。读取和写入速率都必须是正整数。


### 其他团体
--group-add: Add additional groups to run as
默认情况下，docker容器进程运行，并为指定用户查找补充组。如果想要向该组列表添加更多内容，则可以使用此标志：
```
$ docker run --rm --group-add audio --group-add nogroup --group-add 777 busybox id
uid=0(root) gid=0(root) groups=10(wheel),29(audio),99(nogroup),777
```
运行时权限和Linux功能
```
--cap-add: Add Linux capabilities
--cap-drop: Drop Linux capabilities
--privileged=false: Give extended privileges to this container
--device=[]: Allows you to run devices inside the container without the --privileged flag.
```
默认情况下，Docker容器是“非特权”的，例如，不能在Docker容器中运行Docker守护程序。这是因为默认情况下不允许容器访问任何设备，但“特权”容器可以访问所有设备（请参阅cgroups设备上的文档）。

当操作员执行时docker run --privileged，Docker将启用对主机上所有设备的访问，并在AppArmor或SELinux中设置一些配置，以允许容器几乎与主机上运行容器外部的进程一样访问主机。有关运行的其他信息，--privileged请访问 Docker博客。

如果要限制对特定设备的访问，可以使用该--device标志。它允许您指定一个或多个可在容器中访问的设备。
```
$ docker run --device=/dev/snd:/dev/snd ...
```
默认情况下，容器就可以read，write和mknod这些设备。这可以使用第三:rwm组选项覆盖每个--device标志：
```
$ docker run --device=/dev/sda:/dev/xvdc --rm -it ubuntu fdisk  /dev/xvdc

Command (m for help): q
$ docker run --device=/dev/sda:/dev/xvdc:r --rm -it ubuntu fdisk  /dev/xvdc
You will not be able to write the partition table.

Command (m for help): q

$ docker run --device=/dev/sda:/dev/xvdc:w --rm -it ubuntu fdisk  /dev/xvdc
    crash....

$ docker run --device=/dev/sda:/dev/xvdc:m --rm -it ubuntu fdisk  /dev/xvdc
fdisk: unable to open /dev/xvdc: Operation not permitted
```
此外--privileged，操作员可以使用--cap-add和对细节进行细粒度控制--cap-drop。默认情况下，Docker具有保留的默认功能列表。下表列出了默认情况下允许且可以删除的Linux功能选项。

|能力密钥|	能力描述|
| - | - |
|SETPCAP	|修改流程功能。|
|MKNOD	|使用mknod（2）创建特殊文件。|
|AUDIT_WRITE	|将记录写入内核审核日志。|
|CHOWN	|对文件UID和GID进行任意更改（请参阅chown（2））。|
|NET_RAW	|使用RAW和PACKET套接字。|
|DAC_OVERRIDE|	绕过文件读取，写入和执行权限检查。|
|FOWNER	|绕过对通常需要进程的文件系统UID匹配文件UID的操作的权限检查。|
|FSETID|	修改文件时，请勿清除set-user-ID和set-group-ID权限位。|
|kill|	绕过权限检查发送信号。|
|SETGID|	对流程GID和补充GID列表进行任意操作。|
|SETUID|	对进程UID进行任意操作。|
|NET_BIND_SERVICE|	将套接字绑定到Internet域特权端口（端口号小于1024）。|
|SYS_CHROOT|	使用chroot（2），更改根目录。|
|SETFCAP|	设置文件功能。|
下表显示了默认情况下未授予的功能，可以添加这些功能。

|能力密钥|	能力描述|
| - | - |
|SYS_MODULE|	加载和卸载内核模块。|
|SYS_RAWIO	|执行I / O端口操作（iopl（2）和ioperm（2））。|
|SYS_PACCT|	使用acct（2），打开或关闭流程计费。|
|SYS_ADMIN|	执行一系列系统管理操作。|
|SYS_NICE	|提高进程nice值（nice（2），setpriority（2））并更改任意进程的nice值。|
|SYS_RESOURCE|	覆盖资源限制。|
|SYS_TIME|	设置系统时钟（settimeofday（2），stime（2），adjtimex（2））; 设置实时（硬件）时钟。|
|SYS_TTY_CONFIG	|使用vhangup（2）; 在虚拟终端上使用各种特权ioctl（2）操作。|
|AUDIT_CONTROL	|启用和禁用内核审计; 更改审核过滤规则; 检索审核状态和过滤规则。|
|MAC_ADMIN	|允许MAC配置或状态更改。为Smack LSM实施。|
|MAC_OVERRIDE|	覆盖强制访问控制（MAC）。为Smack Linux安全模块（LSM）实现。|
|NET_ADMIN|	执行各种与网络相关的操作。|
|SYSLOG|	执行特权syslog（2）操作。|
|DAC_READ_SEARCH|	绕过文件读取权限检查和目录读取和执行权限检查。|
|LINUX_IMMUTABLE	|设置FS_APPEND_FL和|FS_IMMUTABLE_FL |i节点标志。
|NET_BROADCAST	|进行套接字广播，并收听多播。|
|IPC_LOCK	|锁定内存（mlock（2），mlockall（2），mmap（2），shmctl（2））。|
|IPC_OWNER|	绕过权限检查System V IPC对象上的操作。|
|SYS_PTRACE|	使用ptrace（2）跟踪任意进程。|
|SYS_BOOT|	使用reboot（2）和kexec_load（2），重新启动并加载新内核以便以后执行。|
|LEASE|	建立任意文件的租约（参见fcntl（2））。|
|WAKE_ALARM|	触发唤醒系统的东西。|
|BLOCK_SUSPEND	|使用可阻止系统挂起的功能。|
有关功能的更多参考信息（7） - Linux手册页

两个标志都支持该值ALL，因此如果运营商希望拥有所有功能，但MKNOD他们可以使用：
```
$ docker run --cap-add=ALL --cap-drop=MKNOD ...
```
对于与网络堆栈进行交互，而不是使用--privileged它们应该--cap-add=NET_ADMIN用来修改网络接口。
```
$ docker run -it --rm  ubuntu:14.04 ip link add dummy0 type dummy
RTNETLINK answers: Operation not permitted
$ docker run -it --rm --cap-add=NET_ADMIN ubuntu:14.04 ip link add dummy0 type dummy
```
要安装一个FUSE基于文件系统，你需要在两个结合--cap-add和 --device：
```
$ docker run --rm -it --cap-add SYS_ADMIN sshfs sshfs sven@10.10.10.20:/home/sven /mnt
fuse: failed to open /dev/fuse: Operation not permitted
$ docker run --rm -it --device /dev/fuse sshfs sshfs sven@10.10.10.20:/home/sven /mnt
fusermount: mount failed: Operation not permitted
$ docker run --rm -it --cap-add SYS_ADMIN --device /dev/fuse sshfs
# sshfs sven@10.10.10.20:/home/sven /mnt
The authenticity of host '10.10.10.20 (10.10.10.20)' can't be established.
ECDSA key fingerprint is 25:34:85:75:25:b0:17:46:05:19:04:93:b5:dd:5f:c6.
Are you sure you want to continue connecting (yes/no)? yes
sven@10.10.10.20's password:
root@30aa0cfaf1b5:/# ls -la /mnt/src/docker
total 1516
drwxrwxr-x 1 1000 1000   4096 Dec  4 06:08 .
drwxrwxr-x 1 1000 1000   4096 Dec  4 11:46 ..
-rw-rw-r-- 1 1000 1000     16 Oct  8 00:09 .dockerignore
-rwxrwxr-x 1 1000 1000    464 Oct  8 00:09 .drone.yml
drwxrwxr-x 1 1000 1000   4096 Dec  4 06:11 .git
-rw-rw-r-- 1 1000 1000    461 Dec  4 06:08 .gitignore
....
```
默认的seccomp配置文件将调整为所选功能，以便允许使用功能允许的功能，因此您不必调整此功能，因为Docker 1.12。在Docker 1.10和1.11中，这没有发生，可能需要使用自定义的seccomp配置文件或--security-opt seccomp=unconfined在添加功能时使用。

### 记录驱动程序（--log-driver）
容器可以具有与Docker守护程序不同的日志记录驱动程序。使用--log-driver=VALUEwith docker run命令配置容器的日志记录驱动程序。支持以下选项：

|驱动|	描述|
| - | - |
|none|	禁用容器的任何日志记录。docker logs将不适用于此驱动程序。|
|json-file|	Docker的默认日志记录驱动程序。将JSON消息写入文件。此驱动程序不支持任何日志记录选项。|
|syslog|	Docker的Syslog日志驱动程序。将日志消息写入syslog。|
|journald	|Docker的日志记录驱动程序。将日志消息写入journald。|
|gelf|	Docker的Graylog扩展日志格式（GELF）日志记录驱动程序。将日志消息写入GELF端点，如Graylog或Logstash。|
|fluentd	|Docker的流利日志驱动程序。将日志消息写入fluentd（转发输入）。|
|awslogs|	Amazon CloudWatch记录Docker的日志记录驱动程序。将日志消息写入Amazon CloudWatch Logs|
|splunk|	Docker的Splunk日志记录驱动程序。splunk使用Event Http Collector 将日志消息写入。|
该docker logs命令仅适用于json-file和journald 记录驱动程序。有关使用日志驱动程序的详细信息，请参阅 配置日志驱动程序。

### 覆盖Dockerfile图像默认值
当开发人员从Dockerfile构建映像 或提交映像时，开发人员可以设置一些默认参数，这些参数在映像作为容器启动时生效。

在Dockerfile命令的四个不能在运行时被覆盖：FROM， MAINTAINER，RUN，和ADD。其他一切都有相应的覆盖docker run。我们将介绍开发人员可能在每个Dockerfile指令中设置的内容以及操作员如何覆盖该设置。

* CMD（默认命令或选项）
* ENTRYPOINT（在运行时执行的默认命令）
* EXPOSE（传入端口）
* ENV（环境变量）
* 健康检查
* VOLUME（共享文件系统）
* 用户
* WORKDIR

#### CMD（默认命令或选项）
回想一下COMMANDDocker命令行中的可选项：
```
$ docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [COMMAND] [ARG...]
```
此命令是可选的，因为创建它的人IMAGE可能已COMMAND使用Dockerfile CMD 指令提供了默认值。作为操作员（从图像运行容器的人），您可以CMD通过指定新的指令来覆盖该指令 COMMAND。

如果图像还指定的ENTRYPOINT，则CMD或COMMAND 获得附加作为参数传递给ENTRYPOINT。

#### ENTRYPOINT（在运行时执行的默认命令）
```
--entrypoint="": Overwrite the default entrypoint set by the image
```
该ENTRYPOINT图像是类似COMMAND，因为它指定了可执行文件运行容器启动时，但它是（故意）更难以覆盖。在ENTRYPOINT给出了一个容器，它的默认性质或行为，所以，当你设置一个 ENTRYPOINT可以运行的容器，就好像它是二进制，完全使用默认选项，并且可以在通过更多的选择传球 COMMAND。但是，有时操作员可能希望在容器内运行其他东西，因此您可以ENTRYPOINT在运行时通过使用字符串来指定新的默认值来覆盖默认值ENTRYPOINT。下面是一个如何在容器中运行shell的示例，该容器已设置为自动运行其他内容（如/usr/bin/redis-server）：
```
$ docker run -it --entrypoint /bin/bash example/redis
```
或两个如何将更多参数传递给该ENTRYPOINT的示例：
```
$ docker run -it --entrypoint /bin/bash example/redis -c ls -l
$ docker run -it --entrypoint /usr/bin/redis-cli example/redis --help
```
您可以通过传递空字符串来重置容器入口点，例如：
```
$ docker run -it --entrypoint="" mysql bash
```
> 注意：传递--entrypoint将清除图像上的任何默认命令集（即CMD用于构建它的Dockerfile中的任何指令）。

#### EXPOSE（传入端口）
以下run命令选项适用于容器网络：
```
--expose=[]: Expose a port or a range of ports inside the container.
             These are additional to those exposed by the `EXPOSE` instruction
-P         : Publish all exposed ports to the host interfaces
-p=[]      : Publish a container's port or a range of ports to the host
               format: ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort | containerPort
               Both hostPort and containerPort can be specified as a
               range of ports. When specifying ranges for both, the
               number of container ports in the range must match the
               number of host ports in the range, for example:
                   -p 1234-1236:1234-1236/tcp

               When specifying a range for hostPort only, the
               containerPort must not be a range.  In this case the
               container port is published somewhere within the
               specified hostPort range. (e.g., `-p 1234-1236:1234/tcp`)

               (use 'docker port' to see the actual mapping)

--link=""  : Add link to another container (<name or id>:alias or <name or id>)
```
除了该EXPOSE指令外，图像开发人员还没有太多的网络控制权。该EXPOSE指令定义了提供服务的初始传入端口。这些端口可用于容器内的进程。操作员可以使用该--expose 选项添加到公开的端口。

要公开容器的内部端口，操作员可以使用-Por -p标志启动容器。可以在主机上访问公开的端口，并且任何可以访问主机的客户端都可以使用这些端口。

该-P选项将所有端口发布到主机接口。Docker将每个公开的端口绑定到主机上的随机端口。端口范围在由...定义 的短暂端口范围内/proc/sys/net/ipv4/ip_local_port_range。使用该-p标志显式映射单个端口或端口范围。

容器内部的端口号（服务侦听的位置）不需要与容器外部（客户端连接的位置）上公开的端口号相匹配。例如，在容器内部，HTTP服务正在侦听端口80（因此图像开发人员EXPOSE 80在Dockerfile中指定）。在运行时，端口可能绑定到主机上的42800。要查找主机端口和公开端口之间的映射，请使用docker port。

如果操作员--link在默认桥接网络中启动新客户端容器时使用，则客户端容器可以通过专用网络接口访问公开的端口。如--link在网络概述中所述，在用户定义的网络中启动容器时使用If ，它将为要链接的容器提供命名别名。

#### ENV（环境变量）
Docker在创建Linux容器时自动设置一些环境变量。创建Windows容器时，Docker不会设置任何环境变量。

为Linux容器设置以下环境变量：

|变量	|值|
| - | - |
|HOME|	根据值设置 USER|
|HOSTNAME|	与容器关联的主机名|
|PATH|	包括流行的目录，例如 /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin|
|TERM|	xterm 如果容器被分配了伪TTY|
此外，操作者可以设置任何环境变量，通过使用一个或多个容器中的-e标志，即使覆盖上面提到的，或已经定义通过用Dockerfile显影剂那些ENV。如果运算符在未指定值的情况下命名环境变量，则命名变量的当前值将传播到容器的环境中：
```
$ export today=Wednesday
$ docker run -e "deep=purple" -e today --rm alpine env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=d2219b854598
deep=purple
today=Wednesday
HOME=/root
PS C:\> docker run --rm -e "foo=bar" microsoft/nanoserver cmd /s /c set
ALLUSERSPROFILE=C:\ProgramData
APPDATA=C:\Users\ContainerAdministrator\AppData\Roaming
CommonProgramFiles=C:\Program Files\Common Files
CommonProgramFiles(x86)=C:\Program Files (x86)\Common Files
CommonProgramW6432=C:\Program Files\Common Files
COMPUTERNAME=C2FAEFCC8253
ComSpec=C:\Windows\system32\cmd.exe
foo=bar
LOCALAPPDATA=C:\Users\ContainerAdministrator\AppData\Local
NUMBER_OF_PROCESSORS=8
OS=Windows_NT
Path=C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Users\ContainerAdministrator\AppData\Local\Microsoft\WindowsApps
PATHEXT=.COM;.EXE;.BAT;.CMD
PROCESSOR_ARCHITECTURE=AMD64
PROCESSOR_IDENTIFIER=Intel64 Family 6 Model 62 Stepping 4, GenuineIntel
PROCESSOR_LEVEL=6
PROCESSOR_REVISION=3e04
ProgramData=C:\ProgramData
ProgramFiles=C:\Program Files
ProgramFiles(x86)=C:\Program Files (x86)
ProgramW6432=C:\Program Files
PROMPT=$P$G
PUBLIC=C:\Users\Public
SystemDrive=C:
SystemRoot=C:\Windows
TEMP=C:\Users\ContainerAdministrator\AppData\Local\Temp
TMP=C:\Users\ContainerAdministrator\AppData\Local\Temp
USERDOMAIN=User Manager
USERNAME=ContainerAdministrator
USERPROFILE=C:\Users\ContainerAdministrator
windir=C:\Windows
```
类似地，操作员可以设置HOSTNAME（Linux）或COMPUTERNAME（Windows）-h。

### 健康检查
```
  --health-cmd            Command to run to check health
  --health-interval       Time between running the check
  --health-retries        Consecutive failures needed to report unhealthy
  --health-timeout        Maximum time to allow one check to run
  --health-start-period   Start period for the container to initialize before starting health-retries countdown
  --no-healthcheck        Disable any container-specified HEALTHCHECK
```
例：
```
$ docker run --name=test -d \
    --health-cmd='stat /etc/passwd || exit 1' \
    --health-interval=2s \
    busybox sleep 1d
$ sleep 2; docker inspect --format='{{.State.Health.Status}}' test
healthy
$ docker exec test rm /etc/passwd
$ sleep 2; docker inspect --format='{{json .State.Health}}' test
{
  "Status": "unhealthy",
  "FailingStreak": 3,
  "Log": [
    {
      "Start": "2016-05-25T17:22:04.635478668Z",
      "End": "2016-05-25T17:22:04.7272552Z",
      "ExitCode": 0,
      "Output": "  File: /etc/passwd\n  Size: 334       \tBlocks: 8          IO Block: 4096   regular file\nDevice: 32h/50d\tInode: 12          Links: 1\nAccess: (0664/-rw-rw-r--)  Uid: (    0/    root)   Gid: (    0/    root)\nAccess: 2015-12-05 22:05:32.000000000\nModify: 2015..."
    },
    {
      "Start": "2016-05-25T17:22:06.732900633Z",
      "End": "2016-05-25T17:22:06.822168935Z",
      "ExitCode": 0,
      "Output": "  File: /etc/passwd\n  Size: 334       \tBlocks: 8          IO Block: 4096   regular file\nDevice: 32h/50d\tInode: 12          Links: 1\nAccess: (0664/-rw-rw-r--)  Uid: (    0/    root)   Gid: (    0/    root)\nAccess: 2015-12-05 22:05:32.000000000\nModify: 2015..."
    },
    {
      "Start": "2016-05-25T17:22:08.823956535Z",
      "End": "2016-05-25T17:22:08.897359124Z",
      "ExitCode": 1,
      "Output": "stat: can't stat '/etc/passwd': No such file or directory\n"
    },
    {
      "Start": "2016-05-25T17:22:10.898802931Z",
      "End": "2016-05-25T17:22:10.969631866Z",
      "ExitCode": 1,
      "Output": "stat: can't stat '/etc/passwd': No such file or directory\n"
    },
    {
      "Start": "2016-05-25T17:22:12.971033523Z",
      "End": "2016-05-25T17:22:13.082015516Z",
      "ExitCode": 1,
      "Output": "stat: can't stat '/etc/passwd': No such file or directory\n"
    }
  ]
}
```
运行状况也会显示在运行状态中docker ps。

### TMPFS（mount tmpfs文件系统）
```
--tmpfs=[]: Create a tmpfs mount with: container-dir[:<options>],
            where the options are identical to the Linux
            'mount -t tmpfs -o' command.
```
下面的例子中安装一个空的tmpfs与容器rw， noexec，nosuid，和size=65536k选项。
```
$ docker run -d --tmpfs /run:rw,noexec,nosuid,size=65536k my_image
```
### VOLUME（共享文件系统）
```
-v, --volume=[host-src:]container-dest[:<options>]: Bind mount a volume.
The comma-delimited `options` are [rw|ro], [z|Z],
[[r]shared|[r]slave|[r]private], and [nocopy].
The 'host-src' is an absolute path or a name value.

If neither 'rw' or 'ro' is specified then the volume is mounted in
read-write mode.

The `nocopy` mode is used to disable automatically copying the requested volume
path in the container to the volume storage location.
For named volumes, `copy` is the default mode. Copy modes are not supported
for bind-mounted volumes.

--volumes-from="": Mount all volumes from the given container(s)
```
> 注意：当使用systemd管理Docker守护程序的启动和停止时，在systemd单元文件中有一个选项来控制Docker守护程序本身的挂载传播，称为MountFlags。此设置的值可能导致Docker无法看到在安装点上进行的安装传播更改。例如，如果此值为slave，则可能无法在卷上使用shared或rshared传播。

卷命令非常复杂，可以在“ 使用卷”一节中找到自己的文档。开发人员可以定义VOLUME与图像关联的一个或多个，但只有操作员可以从一个容器到另一个容器（或从容器到安装在主机上的卷）进行访问。

在container-dest必须始终是绝对路径，例如/src/docs。的host-src可以是一个绝对路径或name值。如果为host-dirDocker绑定安装提供了指定路径的绝对路径。如果您提供a name，Docker会创建一个命名卷name。

甲name值必须以字母数字字符，接着启动a-z0-9，_（下划线）， .（周期）或-（连字符）。绝对路径以/（正斜杠）开头。

例如，您可以指定/foo或foo为host-src值。如果提供该/foo值，Docker将创建一个绑定装载。如果提供foo规范，Docker将创建一个命名卷。

### 用户
root（id = 0）是容器中的默认用户。图像开发人员可以创建其他用户。这些用户可以通过名称访问。传递数字ID时，用户不必存在于容器中。

开发人员可以设置默认用户以使用Dockerfile USER指令运行第一个进程。启动容器时，操作员可以USER通过传递-u选项来覆盖指令。
```
-u="", --user="": Sets the username or UID used and optionally the groupname or GID for the specified command.

The followings examples are all valid:
--user=[ user | user:group | uid | uid:gid | user:gid | uid:group ]
```
> 注意：如果传递数字uid，则它必须在0-2147483647范围内。

### WORKDIR
在容器中运行二进制文件的默认工作目录是根目录（/），但开发人员可以使用Dockerfile WORKDIR命令设置不同的默认值。操作员可以用以下方式覆盖：

-w="": Working directory inside the container
