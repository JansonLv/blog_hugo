## docker-compose CLI概述
此页面提供docker-composeCommand 的使用信息。

### 命令选项概述和帮助
您还可以通过docker-compose --help从命令行运行来查看此信息。
```
Define and run multi-container applications with Docker.

Usage:
  docker-compose [-f <arg>...] [options] [COMMAND] [ARGS...]
  docker-compose -h|--help

Options:
  -f, --file FILE             Specify an alternate compose file
                              (default: docker-compose.yml)
  -p, --project-name NAME     Specify an alternate project name
                              (default: directory name)
  --verbose                   Show more output
  --log-level LEVEL           Set log level (DEBUG, INFO, WARNING, ERROR, CRITICAL)
  --no-ansi                   Do not print ANSI control characters
  -v, --version               Print version and exit
  -H, --host HOST             Daemon socket to connect to

  --tls                       Use TLS; implied by --tlsverify
  --tlscacert CA_PATH         Trust certs signed only by this CA
  --tlscert CLIENT_CERT_PATH  Path to TLS certificate file
  --tlskey TLS_KEY_PATH       Path to TLS key file
  --tlsverify                 Use TLS and verify the remote
  --skip-hostname-check       Don't check the daemon's hostname against the
                              name specified in the client certificate
  --project-directory PATH    Specify an alternate working directory
                              (default: the path of the Compose file)
  --compatibility             If set, Compose will attempt to convert deploy
                              keys in v3 files to their non-Swarm equivalent

Commands:
  build              Build or rebuild services
  bundle             Generate a Docker bundle from the Compose file
  config             Validate and view the Compose file
  create             Create services
  down               Stop and remove containers, networks, images, and volumes
  events             Receive real time events from containers
  exec               Execute a command in a running container
  help               Get help on a command
  images             List images
  kill               Kill containers
  logs               View output from containers
  pause              Pause services
  port               Print the public port for a port binding
  ps                 List containers
  pull               Pull service images
  push               Push service images
  restart            Restart services
  rm                 Remove stopped containers
  run                Run a one-off command
  scale              Set number of containers for a service
  start              Start services
  stop               Stop services
  top                Display the running processes
  unpause            Unpause services
  up                 Create and start containers
  version            Show the Docker-Compose version information
```
您可以使用Docker Compose二进制文件docker-compose [-f \<arg>...] [options] [COMMAND] [ARGS...]来构建和管理Docker容器中的多个服务。

### 使用-f指定名称和一个或多个文件撰写路径
使用该-f标志指定Compose配置文件的位置。

#### 指定多个Compose文件
您可以提供多个-f配置文件。当您提供多个文件时，Compose会将它们合并为一个配置。Compose按照提供文件的顺序构建配置。后续文件将覆盖并添加到其前任文件中。

例如，请考虑以下命令行：
```
$ docker-compose -f docker-compose.yml -f docker-compose.admin.yml run backup_db
```
该docker-compose.yml文件可能指定webapp服务。
```
webapp:
  image: examples/web
  ports:
    - "8000:8000"
  volumes:
    - "/data"
```
如果docker-compose.admin.yml还指定了相同的服务，则任何匹配的字段都会覆盖以前的文件。新值，添加到webapp服务配置。
```
webapp:
  build: .
  environment:
    - DEBUG=1
```
使用-fwith -（破折号）作为文件名从中读取配置 stdin。何时stdin使用配置中的所有路径都相对于当前工作目录。

该-f标志是可选的。如果未在命令行上提供此标志，Compose将遍历工作目录及其父目录，以查找a docker-compose.yml和docker-compose.override.yml文件。您必须至少提供该docker-compose.yml文件。如果两个文件都存在于同一目录级别，则Compose会将这两个文件合并为一个配置。

docker-compose.override.yml文件中的配置应用于文件中的值以及除docker-compose.yml文件中的值之外。

#### 指定单个Compose文件的路径
您可以使用-fflag通过命令行或通过在shell或环境文件中设置COMPOSE_FILE环境变量来指定Compose文件的路径，该文件不在当前目录中 。

有关-f在命令行中使用该选项的示例，假设您正在运行Compose Rails示例，并且docker-compose.yml在名为的目录中有一个文件sandbox/rails。你可以使用像docker-compose pull这样的命令db，通过使用-f标志从任何地方获取服务的postgres图像，如下所示：docker-compose -f ~/sandbox/rails/docker-compose.yml pull db

这是完整的例子：
```
$ docker-compose -f ~/sandbox/rails/docker-compose.yml pull db
Pulling db (postgres:latest)...
latest: Pulling from library/postgres
ef0380f84d05: Pull complete
50cf91dc1db8: Pull complete
d3add4cd115c: Pull complete
467830d8a616: Pull complete
089b9db7dc57: Pull complete
6fba0a36935c: Pull complete
81ef0e73c953: Pull complete
338a6c4894dc: Pull complete
15853f32f67c: Pull complete
044c83d92898: Pull complete
17301519f133: Pull complete
dcca70822752: Pull complete
cecf11b8ccf3: Pull complete
Digest: sha256:1364924c753d5ff7e2260cd34dc4ba05ebd40ee8193391220be0f9901d4e1651
Status: Downloaded newer image for postgres:latest
```
#### 使用-p指定项目名称
每个配置都有一个项目名称。如果提供-p标志，则可以指定项目名称。如果未指定标志，Compose将使用当前目录名称。另请参见COMPOSE_PROJECT_NAME环境变量。

#### 设置环境变量
您可以为各种 选项设置环境变量docker-compose，包括-f和-p标志。

例如，COMPOSE_FILE环境变量 与-f标志相关，COMPOSE_PROJECT_NAME环境变量与-p标志相关。

此外，您可以在环境文件中设置其中一些变量。

## cmd

### docker-compose build
服务构建一次，然后标记，默认情况下为project_service。例如，composetest_db。如果Compose文件指定 图像名称，则使用该名称标记图像，预先替换任何变量。见变量替换。

如果更改服务的Dockerfile或其构建目录的内容，请运行docker-compose build以重建它。

### docker-compose bundle
从Compose文件生成分布式应用程序包（DAB）。

图像必须存储摘要，这需要与Docker注册表进行交互。如果没有为所有图像存储摘要，您可以使用docker-compose pull或获取它们docker-compose push。要在捆绑时自动推送图像，请通过--push-images。只有build指定选项的服务才会推送其图像。

### docker-compose config
验证并查看Compose文件

### docker-compose create 
废弃，使用up

### docker-compose down
停止容器并删除由其创建的容器，网络，卷和图像up。

默认情况下，唯一删除的内容是：

* Compose文件中定义的服务的容器
* networks在Compose文件部分中定义的网络
* 默认网络（如果使用）
定义为的网络和卷external永远不会被删除。

### docker-compose events
```
为项目中的每个容器流式传输容器事件。

使用该--json标志，每行打印一个json对象，格式为：

{
    "time": "2015-11-20T18:01:03.615550",
    "type": "container",
    "action": "create",
    "id": "213cf7...5fc39a",
    "service": "web",
    "attributes": {
        "name": "application_web_1",
        "image": "alpine:edge"
    }
}
可以在此处看到可以使用此接收的事件。
```
### docker-compose exec
这相当于docker exec。使用此子命令，您可以在服务中运行任意命令。默认情况下，命令分配TTY，因此您可以使用命令docker-compose exec web sh来获取交互式提示。

### docker-compose kill
通过发送SIGKILL信号强制运行容器停止。可选地，可以传递信号，例如：
```
docker-compose kill -s SIGINT
```

### docker-compose logs
显示服务的日志输出。

### docker-compose pause
暂停运行服务的容器。他们可以暂停docker-compose unpause。

### docker-compose port
打印公共端口以进行端口绑定。

### docker-compose ps
列出容器。
```
$ docker-compose ps

         Name                        Command                 State             Ports
---------------------------------------------------------------------------------------------
mywordpress_db_1          docker-entrypoint.sh mysqld      Up (healthy)  3306/tcp
mywordpress_wordpress_1   /entrypoint.sh apache2-for ...   Restarting    0.0.0
```

### docker-compose pull
```
拉取与docker-compose.yml或docker-stack.yml文件中定义的服务关联的图像，但不会根据这些图像启动容器。

例如，假设您有Quickstart：Compose和Rails示例中的此docker-compose.yml文件。

version: '2'
services:
  db:
    image: postgres
  web:
    build: .
    command: bundle exec rails s -p 3000 -b '0.0.0.0'
    volumes:
      - .:/myapp
    ports:
      - "3000:3000"
    depends_on:
      - db
如果您docker-compose pull ServiceName在与docker-compose.yml定义服务的文件相同的目录中运行，则Docker会提取关联的图像。例如，要在我们的示例中调用postgres配置为db服务的映像，您将运行docker-compose pull db。

$ docker-compose pull db
Pulling db (postgres:latest)...
latest: Pulling from library/postgres
cd0a524342ef: Pull complete
9c784d04dcb0: Pull complete
d99dddf7e662: Pull complete
e5bff71e3ce6: Pull complete
cb3e0a865488: Pull complete
31295d654cd5: Pull complete
fc930a4e09f5: Pull complete
8650cce8ef01: Pull complete
61949acd8e52: Pull complete
527a203588c0: Pull complete
26dec14ac775: Pull complete
0efc0ed5a9e5: Pull complete
40cd26695b38: Pull complete
Digest: sha256:fd6c0e2a9d053bebb294bb13765b3e01be7817bf77b01d58c2377ff27a4a46dc
Status: Downloaded newer image for postgres:latest
```

### docker-compose push

将服务图像推送到各自的服务registry/repository。

做出以下假设：

您正在推送您在本地构建的图像

您可以访问构建密钥

例
```
version: '3'
services:
  service1:
    build: .
    image: localhost:5000/yourimage  # goes to local registry

  service2:
    build: .
    image: youruser/yourimage  # goes to youruser DockerHub registry
```

### docker-compose restart

重新启动所有已停止和正在运行的服

如果更改了docker-compose.yml配置，则在运行此命令后不会反映这些更改。

例如，对环境变量（在构建容器之后添加，但在执行容器命令之前添加）的更改在重新启动后不会更新。

如果您正在寻找配置服务的重新启动策略，请参考 重启在撰写文件v3和 重启在撰写V2。请注意，如果要在群集模式下部署堆栈，则应使用restart_policy。

### docker-compose rm
删除已停止的服务容器。

默认情况下，不会删除附加到容器的匿名卷。你可以用它来覆盖它-v。要列出所有卷，请使用docker volume ls。

任何不在卷中的数据都将丢失。

运行没有选项的命令也会删除由docker-compose up或创建的一次性容器docker-compose run：
```
$ docker-compose rm
Going to remove djangoquickstart_web_run_1
Are you sure? [yN] y
Removing djangoquickstart_web_run_1 ... done
```

### docker-compose run
```
对服务运行一次性命令。例如，以下命令启动web服务并bash作为其命令运行。
```
docker-compose run web bash
```
您使用的命令run以新容器启动，其配置由服务的配置定义，包括卷，链接和其他详细信息。但是，有两个重要的区别。

首先，通过的命令run会覆盖服务配置中定义的命令。例如，如果web启动了 服务配置bash，则使用docker-compose run web python app.py覆盖它python app.py。

第二个区别是该docker-compose run命令不会创建服务配置中指定的任何端口。这可以防止端口与已打开的端口发生冲突。如果您确实希望创建服务的端口并将其映射到主机，请指定--service-ports标志：
```
docker-compose run --service-ports web python manage.py shell
```
或者，可以使用--publish或-p选项指定手动端口映射，就像使用时一样docker run：
```
docker-compose run --publish 8080:80 -p 2022:22 -p 127.0.0.1:2021:21 web python manage.py shell
```
如果启动配置了链接的服务，则该run命令首先检查链接服务是否正在运行，并在服务停止时启动该服务。一旦所有链接服务都在运行，run执行您传递的命令。例如，您可以运行：
```
docker-compose run db psql -h db -U docker
```
这将为链接db容器打开一个交互式PostgreSQL shell 。

如果您不希望该run命令启动链接容器，请使用以下--no-deps标志：
```
docker-compose run --no-deps web python manage.py shell
```
如果要在覆盖容器的重新启动策略时运行后删除容器，请使用以下--rm标志：
```
docker-compose run --rm web python manage.py db upgrade
```
这将运行数据库升级脚本，并在完成运行时删除容器，即使在服务配置中指定了重新启动策略也是如此。
```

### docker-compose scale
注意：不推荐使用此命令。
https://docs.docker.com/compose/reference/scale/

### docker-compose start
启动服务的现有容器。

### docker-compose stop
停止运行容器而不删除它们。他们可以再次启动 docker-compose start。

### docker-compose top
```
显示正在运行的进程。

$ docker-compose top
compose_service_a_1
PID    USER   TIME   COMMAND
----------------------------
4060   root   0:00   top

compose_service_b_1
PID    USER   TIME   COMMAND
----------------------------
4115   root   0:00   top
```
### docker-compose unpause
取消暂停服务的暂停容器。

### docker-compose up
构建，（重新）创建，启动和附加到服务的容器。

除非它们已在运行，否则此命令也会启动任何链接服务。

该docker-compose up命令聚合每个容器的输出（基本上正在运行docker-compose logs -f）。命令退出时，所有容器都将停止。运行docker-compose up -d 在后台启动容器并使其运行。

如果服务存在现有容器，并且在创建容器后更改了服务的配置或映像，则docker-compose up通过停止并重新创建容器（保留已安装的卷）来获取更改。要防止Compose获取更改，请使用该--no-recreate 标志。

如果要强制Compose停止并重新创建所有容器，请使用该 --force-recreate标志。

如果进程遇到错误，则此命令的退出代码为1。
如果使用SIGINT（ctrl+ C）或者中断进程SIGTERM，则停止容器，退出代码为0。
如果SIGINT还是SIGTERM在这段停机阶段再次发送，运行容器被杀害，并退出代码2。