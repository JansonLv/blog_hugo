
# 参考和指南

## 服务配置参考
Compose文件是定义服务， 网络和 卷的YAML文件 。Compose文件的默认路径是。./docker-compose.yml

> 提示：您可以使用此文件的扩展名.yml或.yaml扩展名。他们都工作。

服务定义包含应用于为该服务启动的每个容器的配置，就像将命令行参数传递给 docker container create。同样，网络和卷定义类似于 docker network create和docker volume create。

正如docker container create在Dockerfile指定选项，如CMD， EXPOSE，VOLUME，ENV，在默认情况下尊重-你不需要再次指定它们docker-compose.yml。

您可以在配置值中使用具有类似Bash ${VARIABLE}语法的环境变量 - 请参阅 变量替换以获取完整详细信息。

本节包含版本3中服务定义支持的所有配置选项的列表。

### build 
在构建时应用的配置选项。

build 可以指定为包含构建上下文路径的字符串：
```
version: "3.7"
services:
  webapp:
    build: ./dir
```
或者，作为具有在上下文中指定的路径的对象以及可选的Dockerfile和args：
```
version: "3.7"
services:
  webapp:
    build:
      context: ./dir
      dockerfile: Dockerfile-alternate
      args:
        buildno: 1
```
如果您指定image以及build，则使用以下指定的webapp和可选项对构建的图像进行命名：tagimage
```
build: ./dir
image: webapp:tag
```
这导致一个名为webapp和标记的图像tag，由./dir。

> 注意： 使用（版本3）Compose文件在群集模式下部署堆栈时，将忽略此选项 。该docker stack命令仅接受预先构建的图像。

#### 上下文
包含Dockerfile的目录的路径，或者是git存储库的url。

当提供的值是相对路径时，它被解释为相对于Compose文件的位置。此目录也是发送到Docker守护程序的构建上下文。

使用生成的名称撰写构建并标记它，然后使用该图像。
```
build:
  context: ./dir
```
#### DOCKERFILE
备用Dockerfile。
Compose使用备用文件来构建。还必须指定构建路径。
```
build:
  context: .
  dockerfile: Dockerfile-alternate
```

#### ARGS
添加构建参数，这些参数只能在构建过程中访问。

首先，在Dockerfile中指定参数：
```
ARG buildno
ARG gitcommithash

RUN echo "Build number: $buildno"
RUN echo "Based on commit: $gitcommithash"
```
然后在build键下指定参数。您可以传递映射或列表：
```
build:
  context: .
  args:
    buildno: 1
    gitcommithash: cdc3b19
build:
  context: .
  args:
    - buildno=1
    - gitcommithash=cdc3b19
```
> 注意：在Dockerfile中，如果ARG在FROM指令之前指定， ARG则在构建说明中不可用FROM。如果您需要在两个位置都可以使用参数，请在FROM指令下指定它。请参阅了解ARGS和FROM如何与用户详细信息进行交互。

您可以在指定构建参数时省略该值，在这种情况下，它在构建时的值是运行Compose的环境中的值。
```
args:
  - buildno
  - gitcommithash
```
> 注：YAML布尔值（true，false，yes，no，on，off）必须用引号括起来，这样分析器会将它们解释为字符串。

#### CACHE_FROM
> 注意：此选项是v3.2中的新选项

用于缓存图像列表。
```
build:
  context: .
  cache_from:
    - alpine:latest
    - corp/web_app:3.14
```
#### 标签
注意：此选项是v3.3中的新选项

使用Docker标签将元数据添加到生成的图像中。您可以使用数组或字典。

我们建议您使用反向DNS表示法来防止您的标签与其他软件使用的标签冲突。
```
build:
  context: .
  labels:
    com.example.description: "Accounting webapp"
    com.example.department: "Finance"
    com.example.label-with-empty-value: ""
build:
  context: .
  labels:
    - "com.example.description=Accounting webapp"
    - "com.example.department=Finance"
    - "com.example.label-with-empty-value"
```
#### SHM_SIZE
在3.5版文件格式中添加

设置/dev/shm此构建容器的分区大小。指定为表示字节数的整数值或表示字节值的字符串。
```
build:
  context: .
  shm_size: '2gb'
build:
  context: .
  shm_size: 10000000
```
#### 目标
在3.4版文件格式中添加

在内部定义构建指定的阶段Dockerfile。有关详细信息，请参阅 多阶段构建文档。
```
build:
  context: .
  target: prod
```

### cap_add，cap_drop
添加或删除容器功能。有关man 7 capabilities完整列表，请参阅。
```
cap_add:
  - ALL

cap_drop:
  - NET_ADMIN
  - SYS_ADMIN
```
> 注意： 使用（版本3）Compose文件在swarm模式下部署堆栈时，将忽略这些选项 。

### cgroup_parent
为容器指定可选的父cgroup。

### cgroup_parent: m-executor-abcd
注意： 使用（版本3）Compose文件在群集模式下部署堆栈时，将忽略此选项 。

### 命令
覆盖默认命令。
```
command: bundle exec thin -p 3000
```
该命令也可以是一个列表，方式类似于 dockerfile：
```
command: ["bundle", "exec", "thin", "-p", "3000"]
```

### CONFIGS
使用每服务configs 配置以每个服务为基础授予对配置的访问权限。支持两种不同的语法变体。

注意：配置必须已存在或 在configs 此堆栈文件的顶级配置中定义，否则堆栈部署将失败。

有关配置的更多信息，请参阅配置。

#### 语法短
短语法变体仅指定配置名称。这将授予容器对配置的访问权限并将其安装在/\<config_name> 容器中。源名称和目标安装点都设置为配置名称。

以下示例使用短语法授予redis对my_config和my_other_configconfigs 的服务访问权限。值的值 my_config设置为文件的内容./my_config.txt，并被 my_other_config定义为外部资源，这意味着它已经在Docker中定义，可以通过运行docker config create 命令或通过其他堆栈部署来定义。如果外部配置不存在，则堆栈部署失败并显示config not found错误。

注意：config定义仅在compose文件格式的3.3及更高版本中受支持。
```
version: "3.7"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - my_config
      - my_other_config
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```
#### 语法很长
长语法提供了在服务的任务容器中如何创建配置的更细粒度。

* source：Docker中存在的配置名称。
* target：要在服务的任务容器中装入的文件的路径和名称。/\<source>如果未指定，则默认为。
* uid和gid：在服务的任务容器中拥有已装入的配置文件的数字UID或GID。0如果未指定，则默认为Linux。Windows不支持。
* mode：以八进制表示法在服务的任务容器中装入的文件的权限。例如，0444 代表世界可读。默认是0444。配置无法写入，因为它们安装在临时文件系统中，因此如果设置可写位，则会将其忽略。可以设置可执行位。如果您不熟悉UNIX文件权限模式，则可能会发现此 权限计算器 很有用。

以下示例在容器中设置my_configto 的名称redis_config，将模式设置为0440（group-readable）并将用户和组设置为103。该redis服务无权访问该my_other_config 配置。
```
version: "3.7"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - source: my_config
        target: /redis_config
        uid: '103'
        gid: '103'
        mode: 0440
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```
您可以授予对多个配置的服务访问权限，您可以混合使用长短语法。定义配置并不意味着授予服务访问权限。

### CONTAINER_NAME
指定自定义容器名称，而不是生成的默认名称。
```
container_name: my-web-container
```
由于Docker容器名称必须是唯一的，因此如果指定了自定义名称，则无法将服务扩展到1个容器之外。试图这样做会导致错误。

> 注意： 使用（版本3）Compose文件在群集模式下部署堆栈时，将忽略此选项 。

### credential_spec
> 注意：此选项已在v3.3中添加。

配置托管服务帐户的凭据规范。此选项仅用于使用Windows容器的服务。在credential_spec必须在格式file://\<filename>或registry://\<value-name>。

使用时file:，引用的文件必须存在于CredentialSpecs Docker数据目录的子目录中，默认为C:\ProgramData\Docker\ Windows。以下示例从名为的文件加载凭据规范 C:\ProgramData\Docker\CredentialSpecs\my-credential-spec.json：
```
credential_spec:
  file: my-credential-spec.json
```
使用时registry:，将从守护程序主机上的Windows注册表中读取凭据规范。具有给定名称的注册表值必须位于：
```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Virtualization\Containers\CredentialSpecs
```
以下示例从my-credential-spec 注册表中指定的值加载凭据规范：
```
credential_spec:
  registry: my-credential-spec
```
### depends_on 依赖于取决于
服务依赖关系之间的Express依赖关系会导致以下行为：

docker-compose up以依赖顺序启动服务。在以下示例中，db并redis在之前启动web。

docker-compose up SERVICE自动包含SERVICE依赖项。在以下示例中，docker-compose up web还创建并启动db和redis。

docker-compose stop按依赖顺序停止服务。在以下示例中，web在db和之前停止redis。

简单的例子：
```
version: "3.7"
services:
  web:
    build: .
    depends_on:
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```
> 使用时需要注意以下几点depends_on：

* depends_on在开始之前不会等待db并redis“准备好” web- 直到它们已经开始。如果您需要等待服务准备就绪，请参阅[控制启动顺序](https://docs.docker.com/compose/startup-order/)以 获取有关此问题的更多信息以及解决此问题的策略。

* 版本3不再支持condition形式depends_on。

* 使用版本3 Compose文件在swarm模式下部署堆栈depends_on时，将忽略该选项 。

### 部署
仅限第3版。

指定与部署和运行服务相关的配置。这只能部署到时生效群与 泊坞窗堆栈部署，并且被忽略docker-compose up和docker-compose run。
```
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      replicas: 6
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
```
有几个子选项可供选择：

#### ENDPOINT_MODE
为连接到群集的外部客户端指定服务发现方法。

仅限3.3版。

* endpoint_mode: vip - Docker为服务分配虚拟IP（VIP），作为客户端到达网络服务的前端。Docker在客户端和服务的可用工作节点之间路由请求，而无需客户端知道有多少节点参与服务或其IP地址或端口。（这是默认设置。）

* endpoint_mode: dnsrr - DNS循环（DNSRR）服务发现不使用单个虚拟IP。Docker为服务设置DNS条目，以便服务名称的DNS查询返回IP地址列表，客户端直接连接到其中一个。在您要使用自己的负载均衡器或混合Windows和Linux应用程序的情况下，DNS循环法非常有用。
```
version: "3.7"

services:
  wordpress:
    image: wordpress
    ports:
      - "8080:80"
    networks:
      - overlay
    deploy:
      mode: replicated
      replicas: 2
      endpoint_mode: vip

  mysql:
    image: mysql
    volumes:
       - db-data:/var/lib/mysql/data
    networks:
       - overlay
    deploy:
      mode: replicated
      replicas: 2
      endpoint_mode: dnsrr

volumes:
  db-data:

networks:
  overlay:
```
这些选项endpoint_mode也可用作swarm模式CLI命令docker service create上的标志 。有关所有与swarm相关的docker命令的快速列表，请参阅Swarm模式CLI命令。

要了解有关群集模式下的服务发现和网络的更多信息，请参阅 在群集模式主题中配置服务发现。

#### labels 标签
指定服务的标签。这些标签仅在服务上设置，而不在服务的任何容器上设置。
```
version: "3.7"
services:
  web:
    image: web
    deploy:
      labels:
        com.example.description: "This label will appear on the web service"
```
要在容器上设置标签，请使用以下labels键deploy：
```
version: "3.7"
services:
  web:
    image: web
    labels:
      com.example.description: "This label will appear on all containers for the web service"
```
#### mode 模式
* global
* replicated
任一global（正好一个每群节点容器）或replicated（一个指定的数量的容器）。默认是replicated。（要了解更多信息，请参阅群组主题 中的复制和全局服务。）
```
version: "3.7"
services:
  worker:
    image: dockersamples/examplevotingapp_worker
    deploy:
      mode: global
```

#### PLACEMENT 放置
指定约束和首选项的位置。有关语法的完整描述以及[约束](https://docs.docker.com/engine/reference/commandline/service_create/#specify-service-constraints-constraint)和[首选项](https://docs.docker.com/engine/reference/commandline/service_create/#specify-service-placement-preferences-placement-pref)的可用类型，请参阅docker service create documentation 。
```
version: "3.7"
services:
  db:
    image: postgres
    deploy:
      placement:
        constraints:
          - node.role == manager
          - engine.labels.operatingsystem == ubuntu 14.04
        preferences:
          - spread: node.labels.zone
```
#### replicas 副本
如果服务是replicated（默认值），请指定在任何给定时间应运行的容器数。
```
version: "3.7"
services:
  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 6
```
### limits 资源
配置资源限制。

> 注意：这取代了旧的资源约束选项在撰写非群模式文件之前版本3（ ，cpu_shares，cpu_quota，cpuset， mem_limit，memswap_limit），mem_swappiness如在升级版本2.x到3.x。

这些中的每一个都是单个值，类似于其docker service create对应物。

在这个一般示例中，redis服务被限制为使用不超过50M的内存和0.50（单核的50％）可用处理时间（CPU），并且保留20M了内存和0.25CPU时间（始终可用）。
```
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 50M
        reservations:
          cpus: '0.25'
          memory: 20M
```
以下主题描述了为群集中的服务或容器设置资源约束的可用选项。

寻找在非群模式容器上设置资源的选项？

> 此处描述的选项特定于 deploy键和群模式。如果要在非群集部署上设置资源约束，请使用 Compose file format version 2 CPU，内存和其他资源选项。如果您还有其他问题，请参阅GitHub问题docker / compose / 4513上的讨论。

#### Out of Memory Exceptions（OOME）
如果您的服务或容器尝试使用的内存超过系统可用的内存，则可能会遇到内存不足异常（OOME），并且内核OOM杀手可能会杀死容器或Docker守护程序。要防止这种情况发生，请确保您的应用程序在具有足够内存的主机上运行，​​并参阅了解内存不足的风险。

#### RESTART_POLICY
配置是否以及如何在容器退出时重新启动容器。取代 restart。

* condition：一个none，on-failure或any（默认:) any。
* delay：重新启动尝试之间等待的时间，指定为 持续时间（默认值：0）。
* max_attempts：在放弃之前尝试重启容器的次数（默认值：永不放弃）。如果在配置中未成功重新启动 window，则此尝试不会计入配置的max_attempts值。例如，如果max_attempts设置为“2”，并且第一次尝试时重新启动失败，则可能尝试重新启动两次以上。
* window：在决定重启是否成功之前等待多长时间，指定为持续时间（默认值：立即决定）。
```
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```

#### ROLLBACK_CONFIG
3.7版文件格式及以上

配置在更新失败的情况下应如何回滚服务。

* parallelism：一次回滚的容器数。如果设置为0，则所有容器同时回滚。
* delay：每个容器组的回滚之间等待的时间（默认为0）。
* failure_action：如果回滚失败该怎么办。一个continue或pause（默认pause）
* monitor：每次更新任务后的持续时间以监视失败(ns|us|ms|s|m|h)（默认为0）。
* max_failure_ratio：回滚期间容忍的失败率（默认为0）。
* order：回滚期间的操作顺序。其中一个stop-first（旧任务在启动新任务之前停止），或者start-first（首先启动新任务，并且正在运行的任务暂时重叠）（默认stop-first）。

#### UPDATE_CONFIG
配置服务应如何更新。用于配置滚动更新。

* parallelism：一次更新的容器数。
* delay：更新一组容器之间的等待时间。
* failure_action：如果更新失败该怎么办。其中一个continue，rollback或者pause （默认：pause）。
* monitor：每次更新任务后的持续时间以监视失败(ns|us|ms|s|m|h)（默认为0）。
* max_failure_ratio：更新期间容忍的失败率。
* order：更新期间的操作顺序。其中一个stop-first（旧任务在启动新任务之前停止），或者start-first（新任务首先启动，运行任务暂时重叠）（默认stop-first）注意：仅支持v3.4及更高版本。
注意：order仅支持v3.4及更高版本的撰写文件格式。
```
version: "3.7"
services:
  vote:
    image: dockersamples/examplevotingapp_vote:before
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
        order: stop-first
```
#### 不支持 DOCKER STACK DEPLOY
下面的子选项（支持docker-compose up和docker-compose run）是不支持的docker stack deploy或deploy关键的。

* build 建立
* cgroup_parent
* CONTAINER_NAME
* devices 设备
* TMPFS
* external_links 外部链接
* links 链接
* network_mode 网络模式
* restart 重新开始
* security_opt
* sysctls
* userns_mode
> 提示：请参阅有关如何为服务，swarms和docker-stack.yml文件配置卷的部分。卷的支持，但有群和服务工作，为名为卷或与被限制为节点可以访问必要的卷服务相关联的，他们必须进行配置。

### devices 设备
设备映射列表。使用与--devicedocker client create选项相同的格式。
```
devices:
  - "/dev/ttyUSB0:/dev/ttyUSB0"
```
注意： 使用（版本3）Compose文件在群集模式下部署堆栈时，将忽略此选项 。

### DNS
自定义DNS服务器。可以是单个值或列表。

dns: 8.8.8.8
dns:
  - 8.8.8.8
  - 9.9.9.9
dns_search
自定义DNS搜索域。可以是单个值或列表。

dns_search: example.com
dns_search:
  - dc1.example.com
  - dc2.example.com
入口点
覆盖默认入口点。

entrypoint: /code/entrypoint.sh
入口点也可以是一个列表，方式类似于 dockerfile：

entrypoint:
    - php
    - -d
    - zend_extension=/usr/local/lib/php/extensions/no-debug-non-zts-20100525/xdebug.so
    - -d
    - memory_limit=-1
    - vendor/bin/phpunit
注意：设置entrypoint两者都会覆盖使用ENTRYPOINTDockerfile指令在服务图像上设置的任何默认入口点，并 清除图像上的任何默认命令 - 这意味着如果CMD Dockerfile中有指令，则会被忽略。

env_file
从文件添加环境变量。可以是单个值或列表。

如果已指定Compose文件docker-compose -f FILE，则path in env_file相对于该文件所在的目录。

在环境部分中 声明的环境变量会覆盖这些值 - 即使这些值为空或未定义，这也适用。

env_file: .env
env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/secrets.env
Compose期望env文件中的每一行都是VAR=VAL格式化的。以...开头的行#被视为注释并被忽略。空行也被忽略。

# Set Rails/Rack environment
RACK_ENV=development
注意：如果您的服务指定了构建选项，则在构建期间，环境文件中定义的变量不会自动显示。使用args子选项build定义构建时环境变量。

值按VAL原样使用，根本不修改。例如，如果值由引号括起（通常是shell变量的情况），则引号包含在传递给Compose的值中。

请记住，列表中文件的顺序对于确定分配给多次显示的变量的值非常重要。列表中的文件从上到下进行处理。对于在文件中指定的相同变量a.env并在文件中 分配不同的值b.env，如果b.env列在下面（后），则来自b.envstand 的值。例如，给出以下声明docker-compose.yml：

services:
  some-service:
    env_file:
      - a.env
      - b.env
以下文件：

# a.env
VAR=1
和

# b.env
VAR=hello
$VAR是hello。

环境
添加环境变量。您可以使用数组或字典。任何布尔值; true，false，yes no，需要用引号括起来，以确保YML解析器不会将它们转换为True或False。

仅具有密钥的环境变量将解析为计算机正在运行的计算机上的值，这对于特定于机密或特定于主机的值很有用。

environment:
  RACK_ENV: development
  SHOW: 'true'
  SESSION_SECRET:
environment:
  - RACK_ENV=development
  - SHOW=true
  - SESSION_SECRET
注意：如果您的服务指定了构建选项，environment则在构建期间不会自动显示定义的变量。使用args子选项build定义构建时环境变量。

暴露
暴露端口而不将它们发布到主机 - 它们只能被链接服务访问。只能指定内部端口。

expose:
 - "3000"
 - "8000"
外部链接
链接到此外部docker-compose.yml甚至是Compose外部的容器，尤其是对于提供共享或公共服务的容器。 在指定容器名称和链接别名（）时，external_links遵循与遗留选项类似的语义。linksCONTAINER:ALIAS

external_links:
 - redis_1
 - project_db_1:mysql
 - project_db_1:postgresql
笔记：

如果您使用的是版本2或更高版本的文件格式，则外部创建的容器必须至少连接到与链接到它们的服务相同的网络之一。链接是一种传统选择。我们建议使用网络。

在群集模式下 使用（版本3）Compose文件部署堆栈时，将忽略此选项。

extra_hosts
添加主机名映射。使用与docker client --add-host参数相同的值。

extra_hosts:
 - "somehost:162.242.195.82"
 - "otherhost:50.31.209.229"
在/etc/hosts此服务的内部容器中创建具有ip地址和主机名的条目，例如：

162.242.195.82  somehost
50.31.209.229   otherhost
健康检查
版本2.1文件格式及以上。

配置运行的检查以确定此服务的容器是否“健康”。有关 healthchecks如何工作的详细信息，请参阅HEALTHCHECK Dockerfile指令的文档 。

healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
  start_period: 40s
interval，timeout并start_period指定为持续时间。

注意：start_period仅支持v3.4及更高版本的撰写文件格式。

test必须是字符串或列表。如果是列表，则第一项必须是NONE，CMD或者CMD-SHELL。如果它是一个字符串，则相当于指定CMD-SHELL后跟该字符串。

# Hit the local web app
test: ["CMD", "curl", "-f", "http://localhost"]
如上所述，但包裹在内/bin/sh。以下两种形式都是等同的。

test: ["CMD-SHELL", "curl -f http://localhost || exit 1"]
test: curl -f https://localhost || exit 1
要禁用图像设置的任何默认运行状况检查，您可以使用disable: true。这相当于指定test: ["NONE"]。

healthcheck:
  disable: true
图片
指定图像以从中启动容器。可以是存储库/标记或部分图像ID。

image: redis
image: ubuntu:14.04
image: tutum/influxdb
image: example-registry.com:4000/postgresql
image: a4bc65fd
如果图像不存在，Compose尝试拉取它，除非您还指定了构建，在这种情况下，它使用指定的选项构建它并使用指定的标记对其进行标记。

在里面
在3.7版文件格式中添加。

在容器内运行init，转发信号并重新获得进程。将此选项设置true为为服务启用此功能。

version: "3.7"
services:
  web:
    image: alpine:latest
    init: true
使用的默认init二进制文件是Tini，并安装在/usr/libexec/docker-init守护程序主机上。您可以将守护程序配置为通过init-path配置选项使用自定义init二进制文件 。

隔离
指定容器的隔离技术。在Linux上，唯一支持的值是default。在Windows中，可接受的值是default，process和 hyperv。 有关详细信息，请参阅 Docker Engine文档。

标签
使用Docker标签向容器添加元数据。您可以使用数组或字典。

建议您使用反向DNS表示法来防止标签与其他软件使用的标签冲突。

labels:
  com.example.description: "Accounting webapp"
  com.example.department: "Finance"
  com.example.label-with-empty-value: ""
labels:
  - "com.example.description=Accounting webapp"
  - "com.example.department=Finance"
  - "com.example.label-with-empty-value"
链接
警告：该--link标志是Docker的遗留功能。它最终可能被删除。除非您绝对需要继续使用它，否则我们建议您使用用户定义的网络 来促进两个容器之间的通信，而不是使用--link。用户定义的网络不支持您可以使用的一个功能 --link是在容器之间共享环境变量。但是，您可以使用其他机制（如卷）以更可控的方式在容器之间共享环境变量。

链接到另一个服务中的容器。指定服务名称和链接别名（SERVICE:ALIAS），或仅指定服务名称。

web:
  links:
   - db
   - db:database
   - redis
链接服务的容器可以在与别名相同的主机名上访问，如果未指定别名，则可以访问服务名称。

启用服务进行通信不需要链接 - 默认情况下，任何服务都可以通过该服务的名称访问任何其他服务。（另请参阅“ 撰写网络中的 链接”主题。）

链接还以与depends_on相同的方式表达服务之间的依赖关系 ，因此它们确定服务启动的顺序。

笔记

如果同时定义链接和网络，则它们之间具有链接的服务必须共享至少一个共同的网络才能进行通信。

在群集模式下 使用（版本3）Compose文件部署堆栈时，将忽略此选项 。

记录
记录服务的配置。

logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
该driver 名称指定服务容器的日志记录驱动程序，与--log-driverdocker run选项一起使用（此处记录）。

默认值为json-file。

driver: "json-file"
driver: "syslog"
driver: "none"
注意：只有json-file和journald驱动程序可以直接从docker-compose up和使用日志docker-compose logs。使用任何其他驱动程序不会打印任何日志。

使用options密钥指定日志记录驱动程序的日志记录选项，与--log-opt选项一样docker run。

记录选项是键值对。syslog选项的一个例子：

driver: "syslog"
options:
  syslog-address: "tcp://192.168.0.42:123"
默认驱动程序json-file具有限制存储日志量的选项。为此，请使用键值对来获得最大存储大小和最大文件数：

options:
  max-size: "200k"
  max-file: "10"
上面显示的示例将存储日志文件，直到它们达到max-size200kB，然后旋转它们。存储的各个日志文件的数量由max-file值指定。随着日志超出最大限制，将删除较旧的日志文件以允许存储新日志。

以下是docker-compose.yml限制日志记录存储的示例文件：

version: "3.7"
services:
  some-service:
    image: some-service
    logging:
      driver: "json-file"
      options:
        max-size: "200k"
        max-file: "10"
可用的日志选项取决于您使用的日志记录驱动程序

上面用于控制日志文件和大小的示例使用特定于json文件驱动程序的选项。其他日志记录驱动程序不提供这些特定选项。有关支持的日志记录司机及其选项的完整列表，请参阅 记录司机。

网络模式
网络模式。使用与docker client --network参数相同的值以及特殊表单service:[service name]。

network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
笔记

在群集模式下使用（版本3）Compose文件部署堆栈时，将忽略此选项 。

network_mode: "host"不能与链接混在一起。

网络
要加入的网络，引用顶级networks密钥下的条目 。

services:
  some-service:
    networks:
     - some-network
     - other-network
别名
网络上此服务的别名（备用主机名）。同一网络上的其他容器可以使用服务名称或此别名连接到其中一个服务的容器。

由于aliases是网络范围的，因此相同的服务可以在不同的网络上具有不同的别名。

注意：网络范围的别名可以由多个容器共享，甚至可以由多个服务共享。如果是，则无法保证名称解析为哪个容器。

一般格式如下所示。

services:
  some-service:
    networks:
      some-network:
        aliases:
         - alias1
         - alias3
      other-network:
        aliases:
         - alias2
在下面的例子中，提供了三种服务（web，worker，和db），其中两个网络（沿new和legacy）。该db服务是在到达的主机名db或database上new网络，并db或mysql将上legacy网络。

version: "3.7"

services:
  web:
    image: "nginx:alpine"
    networks:
      - new

  worker:
    image: "my-worker-image:latest"
    networks:
      - legacy

  db:
    image: mysql
    networks:
      new:
        aliases:
          - database
      legacy:
        aliases:
          - mysql

networks:
  new:
  legacy:
IPV4_ADDRESS，IPV6_ADDRESS
在加入网络时为此服务指定容器的静态IP地址。

顶级网络部分中的相应网络配置 必须具有ipam包含每个静态地址的子网配置的 块。

如果需要IPv6寻址，则enable_ipv6 必须设置该选项，并且必须使用版本2.x Compose文件。 IPv6选项目前不适用于群集模式。

一个例子：

version: "3.7"

services:
  app:
    image: nginx:alpine
    networks:
      app_net:
        ipv4_address: 172.16.238.10
        ipv6_address: 2001:3984:3989::10

networks:
  app_net:
    ipam:
      driver: default
      config:
        - subnet: "172.16.238.0/24"
        - subnet: "2001:3984:3989::/64"
PID
pid: "host"
将PID模式设置为主机PID模式。这打开了容器和主机操作系统之间的PID地址空间共享。使用此标志启动的容器可以访问和操作裸机计算机命名空间中的其他容器，反之亦然。

港口
暴露端口。

注意：端口映射与。不兼容network_mode: host

语法短
指定ports（HOST:CONTAINER）或仅指定容器端口（选择短暂的主机端口）。

注意：以HOST:CONTAINER格式映射端口时，使用低于60的容器端口时可能会遇到错误的结果，因为YAML会将格式xx:yy中的数字解析为base-60值。因此，我们建议始终将端口映射明确指定为字符串。

ports:
 - "3000"
 - "3000-3005"
 - "8000:8000"
 - "9090-9091:8080-8081"
 - "49100:22"
 - "127.0.0.1:8001:8001"
 - "127.0.0.1:5000-5010:5000-5010"
 - "6060:6060/udp"
语法很长
长格式语法允许配置无法以简短形式表示的其他字段。

target：容器内的端口
published：公开暴露的港口
protocol：端口协议（tcp或udp）
mode：host用于在每个节点上发布主机端口，或者ingress用于负载平衡的群集模式端口。
ports:
  - target: 80
    published: 8080
    protocol: tcp
    mode: host
注意：长语法是v3.2中的新增功能

重新开始
no是默认的重新启动策略，它不会在任何情况下重新启动容器。当always指定时，容器总是重新启动。该 on-failure如果退出代码指示的故障错误政策重启的容器。

restart: "no"
restart: always
restart: on-failure
restart: unless-stopped
注意： 使用（版本3）Compose文件在群集模式下部署堆栈时，将忽略此选项 。请改用restart_policy。
