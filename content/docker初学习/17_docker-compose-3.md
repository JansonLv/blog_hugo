## Compose中的环境变量

### 在Compose文件中替换环境变量
您可以在shell中使用环境变量来填充Compose文件中的值：
```
web:
  image: "webapp:${TAG}"
```
有关更多信息，请参阅Compose文件参考中的 [变量替换](https://docs.docker.com/compose/compose-file/#variable-substitution)部分。

### 在容器中设置环境变量
您可以使用'environment'键在服务的容器中设置环境变量 ，就像 docker run -e VARIABLE=VALUE ...：
```
web:
  environment:
    - DEBUG=1
```

### 将环境变量传递给容器
您可以使用“环境”键将shell中的环境变量直接传递到服务的容器，而 不是为它们提供值，就像docker run -e VARIABLE ...：
```
web:
  environment:
    - DEBUG
```
所述的值DEBUG在容器变量是从值取为在其中撰写运行在壳中的相同变量。

### “env_file”配置选项
您可以使用'env_file'选项将多个环境变量从外部文件传递到服务的容器，就像docker run --env-file=FILE ...：
```
web:
  env_file:
    - web-variables.env
```

### 使用'docker-compose run'设置环境变量
就像使用一样docker run -e，您可以在一次性容器上设置环境变量docker-compose run -e：
```
docker-compose run -e DEBUG=1 web python console.py
```
您也可以通过不给它赋值来从shell传递变量：
```
docker-compose run -e DEBUG web python console.py
```
所述的值DEBUG在容器变量是从值取为在其中撰写运行在壳中的相同变量。

### “.env”文件
您可以在Compose文件中引用的任何环境变量的默认值，或用于配置Compose，在 名为的环境文件中.env：
```
$ cat .env
TAG=v1.5
```
```
$ cat docker-compose.yml
version: '3'
services:
  web:
    image: "webapp:${TAG}"
```
运行时docker-compose up，web上面定义的服务使用图像webapp:v1.5。您可以使用config命令对此进行验证，该 命令将已解析的应用程序配置打印到终端：
```
$ docker-compose config

version: '3'
services:
  web:
    image: 'webapp:v1.5'
```
shell中的值优先于.env文件中指定的值。如果TAG在shell中设置了不同的值，则替换image 使用它：
```
$ export TAG=v2.0
$ docker-compose config

version: '3'
services:
  web:
    image: 'webapp:v2.0'
```
在多个文件中设置相同的环境变量时，这是Compose用于选择要使用的值的优先级：

1. 撰写文件
2. Shell环境变量
3. 环境文件
4. Dockerfile

### 变量未定义
在下面的示例中，我们在Environment文件和Compose文件上设置相同的环境变量：
```
$ cat ./Docker/api/api.env
NODE_ENV=test
```
```
$ cat docker-compose.yml
version: '3'
services:
  api:
    image: 'node:6-alpine'
    env_file:
     - ./Docker/api/api.env
    environment:
     - NODE_ENV=production
```
运行容器时，Compose文件中定义的环境变量优先。
```
$ docker-compose exec api node

> process.env.NODE_ENV
'production'
```
有任何ARG或ENV在设置Dockerfile只有当不存在用于多克撰写的条目评估板environment或env_file。

## 使用环境变量配置Compose
有几个环境变量可供您配置Docker Compose命令行行为。它们以COMPOSE_或开始DOCKER_记录，并记录在CLI环境变量中。

## 链接创建的环境变量
在v1 Compose文件中使用“links”选项时 ，会为每个链接创建环境变量。它们记录在Link环境变量参考中。
>但是，这些变量已被弃用。请使用链接别名作为主机名。

## 共享撰写文件和项目之间的配置

### Compose支持两种共享通用配置的方法：

1. 使用多个Compose文件扩展整个 Compose文件
2. 延伸出来的单独服务，该extends场（用于撰写文件版本到2.1）
### 多个撰写文件
使用多个Compose文件，您可以为不同的环境或不同的工作流程自定义Compose应用程序。

#### 了解多个Compose文件
默认情况下，Compose会读取两个文件，一个docker-compose.yml和一个可选 docker-compose.override.yml文件。按照惯例，docker-compose.yml 包含您的基本配置。顾名思义，覆盖文件可以包含现有服务或全新服务的配置覆盖。

如果在两个文件中都定义了服务，Compose将使用添加和覆盖配置中描述的规则合并配置。

要使用多个覆盖文件或具有不同名称的覆盖文件，可以使用该-f选项指定文件列表。根据在命令行中指定的顺序撰写合并文件。有关使用的更多信息，请参阅docker-compose 命令参考-f。

使用多个配置文件时，必须确保文件中的所有路径都相对于基本Compose文件（指定的第一个Compose文件-f）。这是必需的，因为覆盖文件无需有效撰写文件。覆盖文件可以包含小的配置片段。跟踪服务的哪个片段相对于哪个路径是困难和混乱的，因此为了使路径更容易理解，必须相对于基本文件定义所有路径。

##### 用例示例
在本节中，有两个常见用例可用于多个Compose文件：更改不同环境的Compose应用程序，以及针对Compose应用程序运行管理任务。

不同的环境
多个文件的常见用例是为类似生产的环境（可能是生产，测试或CI）更改开发Compose应用程序。要支持这些差异，您可以将Compose配置拆分为几个不同的文件：

从定义服务的规范配置的基本文件开始。

docker-compose.yml
```
web:
  image: example/my_web_app:latest
  links:
    - db
    - cache

db:
  image: postgres:latest

cache:
  image: redis:latest
```
在此示例中，开发配置将一些端口公开给主机，将我们的代码作为卷安装，并构建Web映像。

docker-compose.override.yml
```
web:
  build: .
  volumes:
    - '.:/code'
  ports:
    - 8883:80
  environment:
    DEBUG: 'true'

db:
  command: '-d'
  ports:
    - 5432:5432

cache:
  ports:
    - 6379:6379
```
运行docker-compose up时会自动读取覆盖。

现在，在生产环境中使用此Compose应用程序会很不错。因此，创建另一个覆盖文件（可能存储在不同的git仓库中或由不同的团队管理）。

docker-compose.prod.yml
```
web:
  ports:
    - 80:80
  environment:
    PRODUCTION: 'true'

cache:
  environment:
    TTL: '500'
```
要使用此生产Compose文件进行部署，您可以运行
```
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
该部署使用配置中的所有三个服务 docker-compose.yml和docker-compose.prod.yml（但不是在开发配置docker-compose.override.yml）。

见生产有关撰写生产更多的信息。

### 备份任务
另一个常见用例是针对Compose应用程序中的一个或多个服务运行adhoc或管理任务。此示例演示如何运行数据库备份。

从docker-compose.yml开始。
```
web:
  image: example/my_web_app:latest
  links:
    - db

db:
  image: postgres:latest
```
在docker-compose.admin.yml中添加新服务以运行数据库导出或备份。
```
dbadmin:
  build: database_admin/
  links:
    - db
```
开始正常的环境运行docker-compose up -d。要运行数据库备份，请同时包含该数据库备份
docker-compose.admin.yml。
```
docker-compose -f docker-compose.yml -f docker-compose.admin.yml \
    run dbadmin db-backup
```
### 扩展服务
注意：extends早期Compose文件格式支持关键字，最高为Compose file version 2.1（请参阅v1中的扩展并在v2中扩展），但Compose版本3.x不支持该关键字。请参阅添加和删​​除的密钥的版本3摘要，以及有关如何升级的信息。请参阅 moby / moby＃31101，以了解extends在未来版本中以某种形式添加支持的可能性的讨论主题。

Docker Compose的extends关键字可以在不同文件之间共享通用配置，甚至可以完全共享不同的项目。如果您有多个服务可以重用一组通用配置选项，则扩展服务非常有用。使用extends您可以在一个位置定义一组通用服务选项，并从任何地方引用它。

请这种心态links，volumes_from和depends_on从不使用的服务之间共享extends。存在这些异常以避免隐式依赖; 你总是定义links和volumes_from本地。这可确保在读取当前文件时，服务之间的依赖关系清晰可见。在本地定义这些也可以确保对引用文件的更改不会破坏任何内容。

#### 了解扩展配置
在定义任何服务时docker-compose.yml，您可以声明您正在扩展另一个服务，如下所示：
```
web:
  extends:
    file: common-services.yml
    service: webapp
```
这指示Compose重新使用文件中webapp定义的服务的配置common-services.yml。假设common-services.yml 看起来像这样：
```
webapp:
  build: .
  ports:
    - "8000:8000"
  volumes:
    - "/data"
```
在这种情况下，你会得到完全相同的结果，如果你写了 docker-compose.yml与同build，ports和volumes配置值直属定义web。

您可以进一步在本地定义（或重新定义）配置 docker-compose.yml：
```
web:
  extends:
    file: common-services.yml
    service: webapp
  environment:
    - DEBUG=1
  cpu_shares: 5

important_web:
  extends: web
  cpu_shares: 10
```
您还可以编写其他服务并将服务链接web到他们：
```
web:
  extends:
    file: common-services.yml
    service: webapp
  environment:
    - DEBUG=1
  cpu_shares: 5
  links:
    - db
db:
  image: postgres
```
#### 用例示例
当您拥有具有通用配置的多个服务时，扩展单个服务非常有用。下面的示例是具有两个服务的Compose应用程序：Web应用程序和队列工作程序。两种服务都使用相同的代码库并共享许多配置选项。

在common.yml中我们定义了通用配置：
```
app:
  build: .
  environment:
    CONFIG_FILE_PATH: /code/config
    API_KEY: xxxyyy
  cpu_shares: 5
```
在docker-compose.yml中，我们定义了使用通用配置的具体服务：
```
webapp:
  extends:
    file: common.yml
    service: app
  command: /code/run_web_app
  ports:
    - 8080:8080
  links:
    - queue
    - db

queue_worker:
  extends:
    file: common.yml
    service: app
  command: /code/run_worker
  links:
    - queue
```
### 添加和覆盖配置
撰写从原始服务到本地服务的副本配置。如果在原始服务和本地服务中都定义了配置选项，则本地值将替换或扩展原始值。

对于单值选项image，command或者mem_limit，新值替换旧值。
```
# original service
command: python app.py

# local service
command: python otherapp.py

# result
command: python otherapp.py
```
> build并image在Compose文件版本1中

对于build和image使用 Compose文件格式的版本1时，在本地服务中使用一个选项会导致Compose丢弃另一个选项（如果它是在原始服务中定义的）。

例如，如果原始服务定义image: webapp并且本地服务定义，build: .则生成的服务具有a build: .和no image选项。

这是因为build并且image不能在版本1文件中一起使用。


对于多值的选项 ports，expose，external_links，dns， dns_search，和tmpfs，撰写会连接两组的值：
```
# original service
expose:
  - "3000"

# local service
expose:
  - "4000"
  - "5000"

# result
expose:
  - "3000"
  - "4000"
  - "5000"
```
在的情况下environment，labels，volumes，和devices，撰写“合并”的条目连同本地定义的值取优先。对于 environment和labels，环境变量或标签名称确定使用哪个值：
```
# original service
environment:
  - FOO=original
  - BAR=original

# local service
environment:
  - BAR=local
  - BAZ=local

# result
environment:
  - FOO=original
  - BAR=local
  - BAZ=local
```
条目volumes和devices使用在容器安装路径合并：
```
# original service
volumes:
  - ./original:/foo
  - ./original:/bar

# local service
volumes:
  - ./local:/bar
  - ./local:/baz

# result
volumes:
  - ./original:/foo
  - ./local:/bar
  - ./local:/baz
```

## 撰写网络
此页面适用于Compose文件格式版本2及更高版本。Compose file version 1（legacy）不支持网络功能。

默认情况下，Compose 为您的应用设置单个 网络。服务的每个容器都加入默认网络，并且该网络上的其他容器都可以访问它们，并且它们可以在与容器名称相同的主机名上发现。

> 注意：您的应用程序网络的名称基于“项目名称”，该名称基于其所在目录的名称。您可以使用--project-name 标志或COMPOSE_PROJECT_NAME环境变量覆盖项目名称。

例如，假设您的应用程序位于名为的目录中myapp，您的docker-compose.yml样子如下所示：
```
version: "3"
services:
  web:
    build: .
    ports:
      - "8000:8000"
  db:
    image: postgres
    ports:
      - "8001:5432"
```
运行时docker-compose up，会发生以下情况：

1. 创建了一个名为myapp_default的网络。
2. 使用web配置创建容器。它以web名义加入网络 myapp_default。
3. 使用db配置创建容器。它db以名义加入网络myapp_default。
> 在v2.1 +中，覆盖网络总是如此 attachable
  从Compose文件格式2.1开始，覆盖网络始终创建为 attachable，并且这是不可配置的。这意味着独立容器可以连接到覆盖网络。
在Compose文件格式3.x中，您可以选择将attachable属性设置为false。

现在，每个容器都可以查找主机名web或db获取相应容器的IP地址。例如，web应用程序代码可以连接到URL postgres://db:5432并开始使用Postgres数据库。

要注意区分是很重要的HOST_PORT和CONTAINER_PORT。在上面的例子中，db中，HOST_PORTIS 8001和容器端口是 5432（postgres的默认值）。网络化的服务到服务通信使用CONTAINER_PORT。当HOST_PORT定义，服务是群外部访问也是如此。

在web容器中，您的连接字符串db看起来像 postgres://db:5432，并且从主机，连接字符串看起来像postgres://{DOCKER_IP}:8001。

### 更新容器
如果对服务进行配置更改并运行docker-compose up以对其进行更新，则会删除旧容器，新容器将以不同的IP地址加入网络，但名称相同。运行容器可以查找该名称并连接到新地址，但旧地址停止工作。

如果任何容器的连接打开旧容器，它们将被关闭。容器有责任检测此情况，再次查找名称并重新连接。

### 链接
通过链接，您可以定义额外的别名，通过该别名可以从其他服务访问服务。它们不需要启用服务进行通信 - 默认情况下，任何服务都可以通过该服务的名称访问任何其他服务。在以下示例中，db可以从web主机名访问，db并且database：
```
version: "3"
services:

  web:
    build: .
    links:
      - "db:database"
  db:
    image: postgres
```
有关更多信息，请参阅链接参考。

### 多主机网络
> 注意：本节中的说明适用于传统的Docker Swarm操作，仅在定位传统Swarm集群时有效。有关将组合项目部署到较新的集成群模式的说明，请参阅Docker Stacks文档。

当部署一个应用程序撰写的群簇，您可以利用内置的overlay驱动程序，以使在不改变您撰写的文件或应用程序代码容器之间的多主机通信。

请参阅多主机网络入门，了解如何设置Swarm集群。overlay默认情况下，群集使用驱动程序，但如果您愿意，可以明确指定它 - 请参阅下文，了解如何执行此操作。

> 指定自定义网络
您可以使用顶级networks密钥指定自己的网络，而不仅仅使用默认的应用程序网络。这使您可以创建更复杂的拓扑并指定自定义网络驱动程序和选项。您还可以使用它将服务连接到不由Compose管理的外部创建的网络。

每个服务都可以使用服务级别 networks密钥指定要连接的网络，服务级别密钥是引用顶级 networks密钥下的条目的名称列表。

这是一个定义两个自定义网络的Compose文件示例。该proxy服务与服务隔离db，因为它们不共享共享网络 - 只能app与两者通信。
```
version: "3"
services:

  proxy:
    build: ./proxy
    networks:
      - frontend
  app:
    build: ./app
    networks:
      - frontend
      - backend
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    # Use a custom driver
    driver: custom-driver-1
  backend:
    # Use a custom driver which takes special options
    driver: custom-driver-2
    driver_opts:
      foo: "1"
      bar: "2"
```
通过为每个连接的网络设置ipv4_address和/或ipv6_address，可以为网络配置静态IP地址。

网络也可以获得自定义名称（自3.5版本起）：
```
version: "3.5"
networks:
  frontend:
    name: custom_frontend
    driver: custom-driver-1
```
### 配置默认网络
您可以通过在networks命名下定义条目来更改应用程序范围的默认网络的设置，而不是（或同样）指定您自己的网络default：
```
version: "3"
services:

  web:
    build: .
    ports:
      - "8000:8000"
  db:
    image: postgres

networks:
  default:
    # Use a custom driver
    driver: custom-driver-1
```
### 使用预先存在的网络
如果您希望容器加入预先存在的网络，请使用以下external选项：
```
networks:
  default:
    external:
      name: my-pre-existing-network
```
[projectname]_defaultCompose 不是尝试创建一个名为的网络，而是寻找一个被调用的网络my-pre-existing-network并将应用程序的容器连接到它。

## 在生产中使用Compose
使用Compose in development定义应用程序时，可以使用此定义在不同的环境（如CI，登台和生产）中运行应用程序。

部署应用程序的最简单方法是在单个服务器上运行它，类似于运行开发环境的方式。如果要扩展应用程序，可以在Swarm集群上运行Compose应用程序。

### 修改您的Compose文件以进行生产
您可能需要对应用配置进行更改，以便为生产做好准备。这些变化可能包括：

* 删除应用程序代码的任何卷绑定，以便代码保留在容器内，不能从外部更改
* 绑定到主机上的不同端口
* 以不同方式设置环境变量，例如当您需要降低日志记录的详细程度或启用电子邮件发送时）
* 指定重启策略restart: always以避免停机
* 添加额外服务，例如日志聚合器
因此，请考虑定义另一个Compose文件，例如 production.yml，它指定适合生产的配置。此配置文件只需要包含您要从原始Compose文件中进行的更改。可以在原始文件上应用其他Compose文件docker-compose.yml以创建新配置。

获得第二个配置文件后，告诉Compose将其与-f选项一起使用 ：
```
docker-compose -f docker-compose.yml -f production.yml up -d
```
有关更完整的示例，请参阅使用多个撰写文件。

### 部署更改
当您更改应用程序代码时，请记住重建图像并重新创建应用程序的容器。要重新部署名为的服务 web，请使用：
```
$ docker-compose build web
$ docker-compose up --no-deps -d web
```
这首先重建图像web，然后停止，销毁，并重新创建 刚的web服务。该--no-deps标志可以防止Compose重新创建web依赖于的任何服务。

### 在单个服务器上运行Compose
您可以通过适当地设置，和环境变量DOCKER_HOST，使用Compose将应用程序部署到远程Docker主机 。对于这样的任务， Docker Machine可以非常轻松地管理本地和远程Docker主机，即使您没有远程部署，也建议使用它。DOCKER_TLS_VERIFYDOCKER_CERT_PATH

一旦设置了环境变量，所有正常docker-compose 命令都无需进一步配置。

### 在Swarm集群上运行Compose
Docker Swarm是一个Docker本地群集系统，它将单个Docker主机暴露给相同的API，这意味着您可以对Swarm实例使用Compose并在多个主机上运行您的应用程序。

### 在Compose中控制启动和关闭顺序
您可以使用depends_on选项控制服务启动和关闭的 顺序。撰写总是开始和依赖顺序，其中依赖性是通过确定停止容器 depends_on，links，volumes_from，和network_mode: "service:..."。

但是，对于启动，Compose不会等到容器“准备好”（对于您的特定应用程序而言意味着什么） - 直到它正在运行。这是有充分理由的。

等待数据库（例如）准备就绪的问题实际上只是分布式系统的更大问题的一个子集。在生产中，您的数据库可能会变得不可用或随时移动主机。您的应用程序需要能够适应这些类型的故障。

要处理此问题，请将应用程序设计为在发生故障后尝试重新建立与数据库的连接。如果应用程序重试连接，它最终可以连接到数据库。

最佳解决方案是在启动时以及出于任何原因丢失连接时，在应用程序代码中执行此检查。但是，如果您不需要此级别的弹性，则可以使用包装器脚本解决此问题：

使用wait-for-it， dockerize或sh-compatible 等待等工具。这些是小包装脚本，您可以将其包含在应用程序的映像中，以轮询给定的主机和端口，直到它接受TCP连接。

例如，要使用wait-for-it.sh或wait-for包装服务的命令：
```
version: "2"
services:
  web:
    build: .
    ports:
      - "80:8000"
    depends_on:
      - "db"
    command: ["./wait-for-it.sh", "db:5432", "--", "python", "app.py"]
  db:
    image: postgres
```
提示：第一个解决方案存在局限性。例如，它不验证特定服务何时真正准备就绪。如果向命令添加更多参数，请使用bash shift带循环的命令，如下一个示例所示。

或者，编写自己的包装器脚本以执行更多特定于应用程序的运行状况检查。例如，您可能希望等到Postgres完全准备好接受命令：
```
#!/bin/sh
# wait-for-postgres.sh

set -e

host="$1"
shift
cmd="$@"

until PGPASSWORD=$POSTGRES_PASSWORD psql -h "$host" -U "postgres" -c '\q'; do
  >&2 echo "Postgres is unavailable - sleeping"
  sleep 1
done

>&2 echo "Postgres is up - executing command"
exec $cmd
```
您可以通过设置以下方式将其用作上一示例中的包装脚本：
```
command: ["./wait-for-postgres.sh", "db", "python", "app.py"]
```