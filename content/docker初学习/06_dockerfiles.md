
# dockerfile
参考：
* [docker-从入门到实践](https://docker_practice.gitee.io/image/dockerfile/onbuild.html)
* [官方文档1](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
* [官方文档2](https://docs.docker.com/engine/reference/builder/)
## 基础构建 
### 了解构建上下文 
 个人简单理解来说就是以当前位置作为相对相对路径

### 注意点 
1. 选择合适的镜像
2. 上下文中不要有多余的文件，否则造成镜像臃肿,构建时间长等缺点
3. 每构建一个层是隔绝的
4. 只有RUN，COPY，ADD创建图层。其他指令创建临时中间图像，并不增加构建的大小。

### 使用stdin构建（无文件复制的情况下）
下面两种方式等效
```
echo -e 'FROM busybox\nRUN echo "hello world"' | docker build -

docker build - <<EOF
FROM busybox
RUN echo "hello world"
EOF
```
使用此语法可以使用Dockerfilefrom 构建映像stdin，而无需将其他文件作为构建上下文发送。hyphen（-）获取的位置PATH，并指示Docker从而不是Dockerfile从stdin目录中读取构建上下文。
在Dockerfile 不需要将文件复制到映像中的情况下，省略构建上下文非常有用，并且可以提高构建速度，因为没有文件发送到守护程序。

### 排除.dockerignore
要排除与构建无关的文件（不重构源存储库），请使用.dockerignore文件。此文件支持与.gitignore文件类似的排除模式。有关创建一个的信息，请参阅 .dockerignore文件。

### 使用多阶段构建
多阶段构建允许您大幅减小最终图像的大小，而无需减少中间层和文件的数量。

由于图像是在构建过程的最后阶段构建的，因此可以通过利用构建缓存来最小化图像层。

例如，如果您的构建包含多个图层，则可以从较不频繁更改（以确保构建缓存可重用）到更频繁更改的顺序对它们进行排序：

* 安装构建应用程序所需的工具

* 安装或更新库依赖项

* 生成您的应用程序

Go应用程序的Dockerfile可能如下所示：
```
FROM golang:1.11-alpine AS build

# Install tools required for project
# Run `docker build --no-cache .` to update dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.toml and Gopkg.lock
# These layers are only re-built when Gopkg files are updated
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# Install library dependencies
RUN dep ensure -vendor-only

# Copy the entire project and build it
# This layer is rebuilt when a file changes in the project directory
COPY . /go/src/project/
RUN go build -o /bin/project

# This results in a single layer image
FROM scratch
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]

```

### 不要安装不必要的包
为了降低复杂性，依赖性，文件大小和构建时间，请避免安装额外的或不必要的软件包，因为它们可能“很好”。例如，您不需要在数据库映像中包含文本编辑器。

### 解耦应用程序
每个容器应该只有一个问题。将应用程序分离到多个容器中可以更容易地水平扩展和重用容器。例如，Web应用程序堆栈可能包含三个独立的容器，每个容器都有自己独特的映像，以分离的方式管理Web应用程序，数据库和内存缓存。

将每个容器限制为一个进程是一个很好的经验法则，但它不是一个硬性规则。例如，不仅可以使用init进程生成容器 ，而且某些程序可能会自行生成其他进程。例如，Celery可以生成多个工作进程，Apache可以为每个请求创建一个进程。

使用您的最佳判断，尽可能保持容器清洁和模块化。如果容器彼此依赖，则可以使用Docker容器网络 来确保这些容器可以进行通信。 

### 最小化层数
在旧版本的Docker中，最大限度地减少图像中的图层数量以确保它们具有高性能非常重要。添加了以下功能以减少此限制：

只有RUN，COPY，ADD创建图层。其他指令创建临时中间图像，并不增加构建的大小。

在可能的情况下，使用多阶段构建，并仅将所需的工件复制到最终图像中。这允许您在中间构建阶段中包含工具和调试信息，而不会增加最终图像的大小。

### 对多行参数进行排序
只要有可能，通过按字母顺序排序多行参数来缓解以后的更改。这有助于避免重复包并使列表更容易更新。这也使PR更容易阅​​读和审查。在反斜杠（\）之前添加空格也有帮助。

下面是来自一个示例buildpack-deps图像：
```
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion
```
### 利用构建缓存
构建映像时，Docker会逐步Dockerfile执行您的指令， 按指定的顺序执行每个指令。在检查每条指令时，Docker会在其缓存中查找可以重用的现有映像，而不是创建新的（重复）映像。

如果您根本不想使用缓存，则可以使用命令中的--no-cache=true 选项docker build。但是，如果您确实让Docker使用其缓存，那么了解它何时可以找到匹配的图像非常重要。Docker遵循的基本规则概述如下：

从已经在高速缓存中的父图像开始，将下一条指令与从该基本图像导出的所有子图像进行比较，以查看它们中的一个是否使用完全相同的指令构建。如果不是，则缓存无效。

在大多数情况下，简单地将Dockerfile其中一个子图像中的指令进行比较就足够了。但是，某些说明需要更多的检查和解释。

对于ADD和COPY指令，检查图像中文件的内容，并计算每个文件的校验和。在这些校验和中不考虑文件的最后修改时间和最后访问时间。在高速缓存查找期间，将校验和与现有映像中的校验和进行比较。如果文件中的任何内容（例如内容和元数据）发生了任何更改，则缓存将失效。

除了ADD和COPY命令之外，缓存检查不会查看容器中的文件来确定缓存匹配。例如，在处理RUN apt-get -y update命令时，不检查容器中更新的文件以确定是否存在缓存命中。在这种情况下，只需使用命令字符串本身来查找匹配项。

一旦高速缓存失效，所有后续Dockerfile命令都会生成新图像，并且不使用高速缓存。

## dockerfile指令
### .gitignore

### FROM
```
FROM <image> [AS <name>]
FROM <image>[:<tag>] [AS <name>]
FROM <image>[@<digest>] [AS <name>]
```
该FROM指令初始化新的构建阶段并为后续指令设置 基本映像。因此，有效Dockerfile必须以FROM指令开头。图像可以是任何有效图像 - 通过从公共存储库中提取图像来启动它尤其容易。
[了解ARG 第一个之前发生的任何指令声明的变量FROM](https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact)

### LABEL
您可以为图像添加标签，以帮助按项目组织图像，记录许可信息，辅助自动化或其他原因。对于每个标签，添加LABEL以一个或多个键值对开头的行。以下示例显示了不同的可接受格式。内容包括解释性意见。(带空格的必须""或者转义)
```
# Set one or more individual labels
LABEL com.example.version="0.0.1-beta"
LABEL vendor1="ACME Incorporated"
LABEL vendor2=ZENITH\ Incorporated
LABEL com.example.release-date="2015-02-12"
LABEL com.example.version.is-production=""

# Set multiple labels on one line
LABEL com.example.version="0.0.1-beta" com.example.release-date="2015-02-12"

# Set multiple labels at once, using line-continuation characters to break long lines
LABEL vendor=ACME\ Incorporated \
      com.example.is-beta= \
      com.example.is-production="" \
      com.example.version="0.0.1-beta" \
      com.example.release-date="2015-02-12"
```

### RUN
RUN有两种形式：
* RUN \<command\>（shell表单，该命令在shell中运行，Linux中为 /bin/sh -c)
* RUN ["executable", "param1", "param2"] (exec form)

该RUN指令将在当前图像之上的新图层中执行任何命令并提交结果。生成的提交图像将用于下一步Dockerfile。

分层RUN指令和生成提交符合Docker的核心概念，其中提交很便宜，并且可以从图像历史中的任何点创建容器，就像源代码控制一样。

在EXEC形式使得能够避免壳串改写（munging），并RUN 使用不包含指定壳可执行基本图像命令。

可以使用该 命令更改shell表单的默认shell SHELL。

在shell形式中，您可以使用\（反斜杠）将单个RUN指令继续到下一行。例如，请考虑以下两行：
```
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'
```
它们一起相当于这一行：

    RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'

> 注意：要使用不同于“/bin/sh” 的shell，请使用传入所需shell的命令行。例如， RUN ["/bin/bash", "-c", "echo hello"]

> 注意：exec表单被解析为JSON数组，这意味着您必须使用双引号（“）来围绕单词而不是单引号（'）。

> **注意：** ***与shell表单不同，exec表单不会调用命令shell。这意味着不会发生正常的shell处理。例如， RUN [ "echo", "$HOME" ]不会对变量进行替换$HOME。如果你想要shell处理，那么要么使用shell表单，要么直接执行shell，例如：RUN [ "sh", "-c", "echo $HOME" ]。当使用exec表单并直接执行shell时（如shell表单的情况），它是执行环境变量扩展的shell，而不是docker。***

> 注意：在JSON表单中，必须转义反斜杠。这在反斜杠是路径分隔符的Windows上尤为重要。由于不是有效的JSON，以下行将被视为shell表单，并以意外方式失败： RUN ["c:\windows\system32\tasklist.exe"] 此示例的正确语法是： RUN ["c:\\windows\\system32\\tasklist.exe"]

RUN在下一次构建期间，指令的缓存不会自动失效。类似指令的缓存 RUN apt-get dist-upgrade -y将在下一次构建期间重用。例如，RUN可以通过使用--no-cache 标志使指令的高速缓存无效docker build --no-cache。

有关详细信息，请参阅“ Dockerfile最佳实践指南 ”。

> RUN指令的高速缓存可以通过ADD指令无效。

[apt-get命令需要关注](https://docs.docker.com/develop/develop-images/#run)

#### 使用管道
某些RUN命令依赖于使用管道符（|）将一个命令的输出传递到另一个命令的能力，如下例所示：

    RUN wget -O - https://some.site | wc -l > /number
Docker使用/bin/sh -c解释器执行这些命令，解释器仅评估管道中最后一个操作的退出代码以确定成功。在上面的示例中，只要wc -l命令成功，即使wget命令失败，此构建步骤也会成功并生成新映像。

如果您希望命令因管道中任何阶段的错误而失败，请预先set -o pipefail &&确定意外错误，以防止构建无意中成功。例如：

    RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
并非所有shell都支持该-o pipefail选项。

在诸如dash基于Debian的映像上的shell的情况下，考虑使用exec形式RUN明确选择支持该pipefail选项的shell 。例如：

    RUN ["/bin/bash", "-c", "set -o pipefail && wget -O - https://some.site | wc -l > /number"]

### CMD
该CMD指令应用于运行图像包含的软件以及任何参数。CMD应该几乎总是以形式使用CMD ["executable", "param1", "param2"…]。因此，如果图像是用于服务的，例如Apache和Rails，那么你可以运行类似的东西CMD ["apache2","-DFOREGROUND"]。实际上，建议将这种形式的指令用于任何基于服务的图像。

在大多数其他情况下，CMD应该给出一个交互式shell，例如bash，python和perl。例如，CMD ["perl", "-de0"]，CMD ["python"]，或CMD ["php", "-a"]。使用此表单意味着当您执行类似的操作时 docker run -it python，您将被放入可用的shell中，随时可以使用。 CMD应该很少的方式使用CMD ["param", "param"]会同ENTRYPOINT，除非你和你预期的用户已经非常熟悉如何ENTRYPOINT 工作的。

该CMD指令有三种形式：

* CMD ["executable","param1","param2"]（执行形式，这是首选形式）
* CMD ["param1","param2"]（作为ENTRYPOINT的默认参数）
* CMD command param1 param2（贝壳形式）

> Dockerfile中只能有一条CMD指令。如果列出多个，CMD 则只有最后一个CMD生效。

CMD的目的是为执行容器提供默认值，这些默认值可以包含可执行文件，也可以省略可执行文件，在这种情况下，您还必须指定一条ENTRYPOINT 指令。

> 注意：如果CMD用于为ENTRYPOINT 指令提供默认参数，则应使用JSON数组格式指定CMD和ENTRYPOINT指令。

> 注意：exec表单被解析为JSON数组，这意味着您必须使用双引号（“）来围绕单词而不是单引号（'）。

> 注意：与shell表单不同，exec表单不会调用命令shell。这意味着不会发生正常的shell处理。例如， CMD [ "echo", "$HOME" ]不会对变量进行替换$HOME。如果你想要shell处理，那么要么使用shell表单，要么直接执行shell，例如：CMD [ "sh", "-c", "echo $HOME" ]。当使用exec表单并直接执行shell时（如shell表单的情况），它是执行环境变量扩展的shell，而不是docker。

在shell或exec格式中使用时，该CMD指令设置在运行映像时要执行的命令。

如果你使用的shell形式CMD，那么\<command>将执行 /bin/sh -c：

    FROM ubuntu
    CMD echo "This is a test." | wc -
如果要在 \<command> 没有shell 的情况下运行，则必须将该命令表示为JSON数组，并提供可执行文件的完整路径。 此数组形式是首选格式CMD。任何其他参数必须在数组中单独表示为字符串：

    FROM ubuntu
    CMD ["/usr/bin/wc","--help"]
如果您希望容器每次都运行相同的可执行文件，那么您应该考虑ENTRYPOINT结合使用CMD。请参阅 ENTRYPOINT。

如果用户指定了参数，docker run那么它们将覆盖指定的默认值CMD。

注意：不要混淆RUN使用CMD。RUN实际上运行一个命令并提交结果; CMD在构建时不执行任何操作，但指定图像的预期命令。

### EXPOSE
    EXPOSE <port> [<port>/<protocol>...]
该EXPOSE指令通知Docker容器在运行时侦听指定的网络端口。您可以指定端口是侦听TCP还是UDP，如果未指定协议，则默认为TCP。

该EXPOSE指令实际上不发布端口。它在构建映像的人和运行容器的人之间起到一种文档的作用，关于哪些端口要发布。要在运行容器时实际发布端口，请使用-p标志on docker run 来发布和映射一个或多个端口，或使用-P标志发布所有公开的端口并将它们映射到高阶端口。

默认情况下，EXPOSE假定为TCP。您还可以指定UDP：

    EXPOSE 80/udp
要在TCP和UDP上公开，请包含两行：

    EXPOSE 80/tcp
    EXPOSE 80/udp
在这种情况下，如果使用-Pwith docker run，端口将为TCP暴露一次，对UDP则暴露一次。请记住，-P在主机上使用短暂的高阶主机端口，因此TCP和UDP的端口不同。

无论EXPOSE设置如何，您都可以使用-p标志在运行时覆盖它们。例如

    docker run -p 80:80/tcp -p 80:80/udp ...
要在主机系统上设置端口重定向，请参阅使用-P标志。该docker network命令支持创建用于容器之间通信的网络，而无需公开或发布特定端口，因为连接到网络的容器可以通过任何端口相互通信。

### ENV
    
    ENV <key> <value>
    ENV <key>=<value> ...

该ENV指令将环境变量\<key>设置为该值 \<value>。此值将在构建阶段中的所有后续指令的环境中，并且也可以在许多内联替换。

该ENV指令有两种形式。第一种形式，ENV \<key> \<value>将单个变量设置为一个值。第一个空格后的整个字符串将被视为\<value>- 包括空格字符。该值将针对其他环境变量进行解释，因此如果未对其进行转义，则将删除引号字符。

第二种形式ENV \<key>=\<value> ...允许一次设置多个变量。请注意，第二种形式在语法中使用等号（=），而第一种形式则不然。与命令行解析一样，引号和反斜杠可用于在值内包含空格。

例如：

    ENV myName="John Doe" myDog=Rex\ The\ Dog \
    myCat=fluffy
和

    ENV myName John Doe
    ENV myDog Rex The Dog
    ENV myCat fluffy
将在最终图像中产生相同的净结果。

ENV当从生成的图像运行容器时，使用的环境变量将保持不变。您可以使用docker inspect和查看值，并使用它们进行更改docker run --env \<key>=\<value>。

要为单个命令设置值，请使用 RUN \<key>=\<value> \<command>。否则会持久化

每ENV行创建一个新的中间层，就像RUN命令一样。这意味着即使您在将来的图层中取消设置环境变量，它仍然会在此图层中保留，并且可以转储其值。您可以通过创建如下所示的Dockerfile来测试它，然后构建它。
```
FROM alpine
ENV ADMIN_USER="mark"
RUN echo $ADMIN_USER > ./mark
RUN unset ADMIN_USER
```
```
docker run --rm test sh -c 'echo $ADMIN_USER'

mark
```
要防止这种情况，并且确实取消设置环境变量，请使用RUN带有shell命令的命令，在单个图层中设置，使用和取消设置变量all。您可以使用;或分隔命令&&。如果您使用第二种方法，并且其中一个命令失败，则docker build也会失败。这通常是一个好主意。使用\Linux Dockerfiles作为行继续符可以提高可读性。您还可以将所有命令放入shell脚本中，并让RUN命令运行该shell脚本。
```
FROM alpine
RUN export ADMIN_USER="mark" \
    && echo $ADMIN_USER > ./mark \
    && unset ADMIN_USER
CMD sh
```
```
docker run --rm test sh -c 'echo $ADMIN_USER'
(没有返回)
docker run --rm env:v2 sh -c 'echo mark'       
mark

```

### ADD
两种形式：
* ADD [--chown=<user>:<group>] \<src>... \<dest>
* ADD [--chown=<user>:<group>] ["\<src>",... "\<dest>"] （包含空格的路径需要此表单）
> --chown功能仅在用于构建Linux容器的Dockerfiles上受支持，并且不适用于Windows容器。

该ADD指令从中复制新文件，目录或远程文件URL \<src> ，并将它们添加到路径上图像的文件系统中\<dest>。

\<src>可以指定多个资源，但如果它们是文件或目录，则它们的路径将被解释为相对于构建上下文的源。

每个都\<src>可能包含通配符，匹配将使用Go的 filepath.Match规则完成。例如：

    ADD hom* /mydir/        # adds all files starting with "hom"
    ADD hom?.txt /mydir/    # ? is replaced with any single character, e.g., "home.txt"

\<dest>是一个绝对路径，或相对于一个路径WORKDIR，到其中的源将在目标容器内进行复制。

ADD test relativeDir/          # adds "test" to `WORKDIR`/relativeDir/
ADD test /absoluteDir/         # adds "test" to /absoluteDir/

添加包含特殊字符（例如[ and ]）的文件或目录时，需要按照Golang规则转义这些路径，以防止它们被视为匹配模式。例如，要添加名为的文件arr[0].txt，请使用以下命令;

    ADD arr[[]0].txt /mydir/    # copy a file named "arr[0].txt" to /mydir/

除非可选--chown标志指定给定用户名，组名或UID / GID组合以请求添加内容的特定所有权，否则将使用UID和GID为0创建所有新文件和目录。--chown标志的格式允许用户名和组名字符串或任意组合的直接整数UID和GID。提供没有组名的用户名或没有GID的UID将使用与GID相同的数字UID。如果提供了用户名或组名，则容器的根文件系统 /etc/passwd和/etc/group文件将分别用于执行从名称到整数UID或GID的转换。以下示例显示了该--chown标志的有效定义：

    ADD --chown=55:mygroup files* /somedir/
    ADD --chown=bin files* /somedir/
    ADD --chown=1 files* /somedir/
    ADD --chown=10:11 files* /somedir/
如果容器根文件系统不包含任何文件/etc/passwd或 /etc/group文件，并且--chown 标志中使用了用户名或组名，则构建将在ADD操作上失败。使用数字ID不需要查找，也不依赖于容器根文件系统内容。

在\<src>远程文件URL 的情况下，目标将具有600的权限。如果正在检索的远程文件具有HTTP Last-Modified标头，则来自该标头的时间戳将用于设置mtime目标文件。但是，与在处理期间处理的任何其他文件一样ADD，mtime将不包括在确定文件是否已更改且应更新缓存中。

> ***注意***：如果通过传递DockerfileSTDIN（docker build - < somefile）进行构建，则没有构建上下文，因此Dockerfile 只能包含基于URL的ADD指令。您还可以通过STDIN :( docker build - < archive.tar.gz）传递压缩存档Dockerfile，该存档位于存档的根目录，其余存档将用作构建的上下文。

> ***注意***：如果您的网址文件都使用认证保护，您将需要使用RUN wget，RUN curl或使用其它工具从容器内的ADD指令不支持验证。

>  ***注意***：ADD如果内容\<src>已更改，则第一个遇到的指令将使来自Dockerfile的所有后续指令的高速缓存无效。这包括使缓存无效以获取RUN指令。

ADD 遵守以下规则：

* 该\<src>路径必须是内部语境的构建; 你不能ADD ../something /something，因为a的第一步 docker build是将上下文目录（和子目录）发送到docker守护进程。

* 如果\<src>是URL并且<\dest>不以尾部斜杠结尾，则从URL下载文件并将其复制到\<dest>。

* 如果\<src>是URL并且\<dest>以尾部斜杠结尾，则从URL推断文件名并将文件下载到 \<dest>/\<filename>。例如，ADD http://example.com/foobar /将创建该文件/foobar。URL必须具有非常重要的路径，以便在这种情况下可以发现适当的文件名（http://example.com 不起作用）。

* 如果\<src>是目录，则复制目录的全部内容，包括文件系统元数据。

> ***注意***：不复制目录本身，只复制其内容。

* 如果\<src>是以识别的压缩格式（identity，gzip，bzip2或xz）的本地 tar存档，则将其解压缩为目录。远程 URL中的资源不会被解压缩。复制或解压缩目录时，它具有相同的行为tar -x，结果是：

1. 无论在目的地路径上存在什么，
2. 源树的内容，在逐个文件的基础上解决了有利于“2.”的冲突。
> ***注意***：文件是否被标识为可识别的压缩格式仅基于文件的内容而不是文件的名称来完成。例如，如果一个空文件碰巧以此结束，.tar.gz则不会将其识别为压缩文件，也不会生成任何类型的解压缩错误消息，而是将文件简单地复制到目标。

* 如果\<src>是任何其他类型的文件，则将其与元数据一起单独复制。在这种情况下，如果\<dest>以尾部斜杠结束/，则将其视为目录，\<src>并将写入内容\<dest>/base(\<src>)。

* 如果\<src>直接或由于使用通配符指定了多个资源，则\<dest>必须是目录，并且必须以斜杠结尾/。

* 如果\<dest>不以尾部斜杠结束，则将其视为常规文件，\<src>并将写入内容\<dest>。

* 如果\<dest>不存在，则会在其路径中创建所有缺少的目录。

很绕口，总结下，就是如果src使用通配符，则dest必须要以/结尾，否则无所谓，路径为dest+src上下文相对路径。

### COPY
COPY有两种形式：

    COPY [--chown=<user>:<group>] <src>... <dest>
    COPY [--chown=<user>:<group>] ["<src>",... "<dest>"] （包含空格的路径需要此表单）
> ***注意***：该--chown功能仅在用于构建Linux容器的Dockerfiles上受支持，并且不适用于Windows容器。

和ADD命令类似，遵循一下规则：
* 该\<src>路径必须是内部语境的构建; 你不能COPY ../something /something，因为a的第一步 docker build是将上下文目录（和子目录）发送到docker守护进程。

* 如果\<src>是目录，则复制目录的全部内容，包括文件系统元数据。

> 注意：不复制目录本身，只复制其内容。

* 如果\<src>是任何其他类型的文件，则将其与元数据一起单独复制。在这种情况下，如果\<dest>以尾部斜杠结束/，则将其视为目录，\<src>并将写入内容\<dest>/base(\<src>)。

* 如果\<src>直接或由于使用通配符指定了多个资源，则\<dest>必须是目录，并且必须以斜杠结尾/。

* 如果\<dest>不以尾部斜杠结束，则将其视为常规文件，\<src>并将写入内容\<dest>。

* 如果\<dest>不存在，则会在其路径中创建所有缺少的目录。

### ADD和COPY对比
一般而言，虽然ADD和COPY在功能上类似，但是**COPY**是优选的。那是因为它更透明ADD。COPY仅支持将本地文件基本复制到容器中，同时ADD具有一些功能（如仅限本地的tar提取和远程URL支持），这些功能并不是很明显。因此，ADD最好的用途是将本地tar文件自动提取到图像中，如ADD rootfs.tar.xz /。

如果您有多个Dockerfile步骤使用上下文中的不同文件，则COPY它们是单独的，而不是一次性完成。这可确保每个步骤的构建缓存仅在特定所需文件更改时失效（强制重新执行该步骤）。

例如：

    COPY requirements.txt /tmp/
    RUN pip install --requirement /tmp/requirements.txt
    COPY . /tmp/
导致RUN步骤的缓存失效次数少于放置 COPY . /tmp/之前的缓存失效次数。

由于图像大小很重要，ADD因此强烈建议不要使用从远程URL获取包。你应该使用curl或wget代替。这样，您可以删除提取后不再需要的文件，也不必在图像中添加其他图层。例如，你应该避免做以下事情：

    ADD http://example.com/big.tar.xz /usr/src/things/
    RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
    RUN make -C /usr/src/things all
而是做一些这样处理：

    RUN mkdir -p /usr/src/things \
    && curl -SL http://example.com/big.tar.xz \
    | tar -xJC /usr/src/things \
    && make -C /usr/src/things all
对于不需要ADDtar自动提取功能的其他项目（文件，目录），您应该始终使用COPY。

### ENTRYPOINT
ENTRYPOINT有两种形式：

* ENTRYPOINT ["executable", "param1", "param2"] （执行形式，首选）
* ENTRYPOINT command param1 param2 （贝壳形式）

ENTRYPOINT允许您配置将作为可执行文件运行的容器。

例如，以下将使用其默认内容启动nginx，侦听端口80：

    docker run -i -t --rm -p 80:80 nginx
docker run \<image>的命令行参数将附加在exec表单ENTRYPOINT中的所有元素之后，并将覆盖使用的所有指定元素CMD。这允许将参数传递给入口点，docker run \<image> -d 即将-d参数传递给入口点。您可以ENTRYPOINT使用docker run --entrypoint 标志覆盖指令。

shell防止任何CMD或run被使用命令行参数，但是缺点是ENTRYPOINT将被作为一个子命令/bin/sh -c执行，其不通过信号。这意味着可执行文件不会是容器PID 1- 并且不会收到Unix信号 - 因此您的可执行文件将不会收到 SIGTERM来自docker stop \<container>。

dockerfile只有最后一条ENTRYPOINT指令能被有效执行。

#### ENTRYPOINT+CMD
创建一个dockerfile
```
FROM ubuntu
ENTRYPOINT ["top", "-b"]
CMD ["-c"]
```
运行容器时，您可以看到这top是唯一的进程：
```
$ docker run -it --rm --name test  top -H
top - 08:25:00 up  7:27,  0 users,  load average: 0.00, 0.01, 0.05
Threads:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.1 us,  0.1 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:   2056668 total,  1616832 used,   439836 free,    99352 buffers
KiB Swap:  1441840 total,        0 used,  1441840 free.  1324440 cached Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
    1 root      20   0   19744   2336   2080 R  0.0  0.1   0:00.04 top
```
要进一步检查结果，您可以使用docker exec：
```
$ docker exec -it test ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  2.6  0.1  19752  2352 ?        Ss+  08:24   0:00 top -b -H
root         7  0.0  0.1  15572  2164 ?        R+   08:25   0:00 ps aux
```
并且您可以优雅地请求top关闭使用docker stop test。

如果直接执行docker run -it --rm --name test  top，再去查看docker exec -it test ps aux，我们会发现
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  1.7  0.1  36480  3100 pts/0    Ss+  03:44   0:00 top -b -c
root         7  0.0  0.1  34396  2800 pts/1    Rs+  03:44   0:00 ps aux
```
再执行一个docker run -it --rm --name 0501 05:v1 tail
```
top: unknown option 't'
Usage:
  top -hv | -bcHiOSs -d secs -n max -u|U user -p pid(s) -o field -w [cols]
```
总结：ENTRYPOINT+CMD cmd的命令会追加到ENTRYPOINT的尾部，并且执行。（无论dockerfiel中有没有CMD）docker run时，追加的命令会覆盖dockerfile的CMD，并追加到ENTRYPOINT的尾部。

> ***注意***：您可以使用覆盖ENTRYPOINT设置--entrypoint，但这只能将二进制设置为exec（不会sh -c使用）。

> ***注意***：与shell表单不同，exec表单不会调用命令shell。这意味着不会发生正常的shell处理。例如， ENTRYPOINT [ "echo", "$HOME" ]不会对变量进行替换$HOME。如果你想要shell处理，那么要么使用shell表单，要么直接执行shell，例如：ENTRYPOINT [ "sh", "-c", "echo $HOME" ]。当使用exec表单并直接执行shell时（如shell表单的情况），它是执行环境变量扩展的shell，而不是docker。


#### SHELL和ENTRYPOINT
您可以为其指定一个纯字符串，ENTRYPOINT它将在其中执行/bin/sh -c。此表单将使用shell处理来替换shell环境变量，并将忽略任何CMD或docker run命令行参数。为确保docker stop能ENTRYPOINT正确发出任何长时间运行的可执行文件，您需要记住启动它exec：

    FROM ubuntu
    ENTRYPOINT exec top -b
运行此图像时，您将看到单个PID 1进程：
```
top - 06:23:58 up 6 days, 23:53,  0 users,  load average: 0.35, 0.36, 0.30
Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.8 us,  1.1 sy,  0.0 ni, 98.1 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  2047036 total,   127136 free,   680916 used,  1238984 buff/cache
KiB Swap:  1048572 total,  1045792 free,     2780 used.  1183940 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0   36480   3100   2748 R   0.0  0.2   0:00.05 top
```

#### 了解CMD和ENTRYPOINT相互作用
CMD和ENTRYPOINT指令都定义了运行容器时要执行的命令。很少有规则描述他们的合作。
1. Dockerfile应至少指定一个CMD或ENTRYPOINT命令。

2. ENTRYPOINT 应该在将容器用作可执行文件时定义。

3. CMD应该用作定义ENTRYPOINT命令的默认参数或在容器中执行ad-hoc命令的方法。

4. CMD 在使用备用参数运行容器时被覆盖。

下表显示了针对不同ENTRYPOINT/ CMD组合执行的命令：

 |  |没有ENTRYPOINT|ENTRYPOINT exec_entry p1_entry| ENTRYPOINT [“exec_entry”，“p1_entry”]|
 | --- | --- |  --- | --- | --- |
 |没有CMD|错误，不允许|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry |
 |CMD [“exec_cmd”，“p1_cmd”]|exec_cmd p1_cmd|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry exec_cmd p1_cmd|
|CMD [“p1_cmd”，“p2_cmd”]|p1_cmd p2_cmd|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry p1_cmd p2_cmd|
|CMD exec_cmd p1_cmd|/bin/sh -c exec_cmd p1_cmd|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd|

### VOLUME

    VOLUME ["/data"]
该VOLUME指令创建具有指定名称的安装点，并将其标记为从本机主机或其他容器保存外部安装的卷。该值可以是JSON数组，VOLUME ["/var/log/"]或具有多个参数的普通字符串，例如VOLUME /var/log或VOLUME /var/log /var/db。

该docker run命令使用基础映像中指定位置存在的任何数据初始化新创建的卷。例如，请考虑以下Dockerfile片段：

    FROM ubuntu
    RUN mkdir /myvol
    RUN echo "hello world" > /myvol/greeting
    VOLUME /myvol
此Dockerfile会docker run且不挂载任何本地文件夹和/myvol做映射的情况下生成一个图像，该图像将导致创建新的挂载点/myvol并将greeting文件复制到新创建的卷中。否则在挂载（-v path:/myvol）的情况下，本地文件夹会覆盖容器文件夹。

#### 有关指定卷的说明
关于卷中的卷，请记住以下事项Dockerfile。
* 从Dockerfile中更改卷：如果任何构建步骤在声明后更改卷内的数据，那么这些更改将被丢弃。

* JSON格式：列表被解析为JSON数组。您必须用双引号（"）而不是单引号（'）括起单词。

* 主机目录在容器运行时声明：主机目录（mountpoint）本质上是依赖于主机的。这是为了保持图像的可移植性，因为不能保证给定的主机目录在所有主机上都可用。因此，您无法从Dockerfile中安装主机目录。该VOLUME指令不支持指定host-dir 参数。您必须在创建或运行容器时指定安装点。

### USER
    USER <user>[:<group>] or
    USER <UID>[:<GID>]
如果服务可以在没有权限的情况下运行，请使用USER更改为非root用户。首先在Dockerfile类似的东西中创建用户和组RUN groupadd -r postgres && useradd --no-log-init -r -g postgres postgres。
> ***警告***：当用户没有主要组时，图像（或下一个说明）将与该root组一起运行。

USER 指令和 WORKDIR 相似，都是改变环境状态并影响以后的层。WORKDIR 是改变工作目录，USER 则是改变之后层的执行 RUN, CMD 以及 ENTRYPOINT 这类命令的身份。

当然，和 WORKDIR 一样，USER 只是帮助你切换到指定用户而已，这个用户必须是事先建立好的，否则无法切换。

    RUN groupadd -r redis && useradd -r -g redis redis
    USER redis
    RUN [ "redis-server" ]

### WORKDIR

    WORKDIR /path/to/workdir
为了清晰和可靠，您应该始终使用绝对路径 WORKDIR。

该WORKDIR指令集的工作目录对任何RUN，CMD， ENTRYPOINT，COPY和ADD它后面的说明Dockerfile。如果WORKDIR不存在，即使它未在任何后续Dockerfile指令中使用，也将创建它。

之前提到一些初学者常犯的错误是把 Dockerfile 等同于 Shell 脚本来书写，这种错误的理解还可能会导致出现下面这样的错误：

RUN cd /app
RUN echo "hello" > world.txt
如果将这个 Dockerfile 进行构建镜像运行后，会发现找不到 /app/world.txt 文件，或者其内容不是 hello。原因其实很简单，在 Shell 中，连续两行是同一个进程执行环境，因此前一个命令修改的内存状态，会直接影响后一个命令；而在 Dockerfile 中，这两行 RUN 命令的执行环境根本不同，是两个完全不同的容器。这就是对 Dockerfile 构建分层存储的概念不了解所导致的错误。

之前说过每一个 RUN 都是启动一个容器、执行命令、然后提交存储层文件变更。第一层 RUN cd /app 的执行仅仅是当前进程的工作目录变更，一个内存上的变化而已，其结果不会造成任何文件变更。而到第二层的时候，启动的是一个全新的容器，跟第一层的容器更完全没关系，自然不可能继承前一层构建过程中的内存变化。

因此如果需要改变以后各层的工作目录的位置，那么应该使用 WORKDIR 指令。

该WORKDIR指令可以在a中多次使用Dockerfile。如果提供了相对路径，则它将相对于前一条WORKDIR指令的路径 。例如：

    WORKDIR /a
    WORKDIR b
    WORKDIR c
    RUN pwd
最终pwd命令的输出Dockerfile将是 /a/b/c。

该WORKDIR指令可以解析先前使用的环境变量 ENV。您只能使用显式设置的环境变量Dockerfile。例如：

    ENV DIRPATH /path
    WORKDIR $DIRPATH/$DIRNAME
    RUN pwd
最终pwd命令的输出Dockerfile将是 /path/$DIRNAME

### ARG
    ARG <name>[=<default value>]
该ARG指令定义了一个变量，用户可以docker build使用该--build-arg <varname>=<value> 标志在构建时将该变量传递给构建器。如果用户指定了未在Dockerfile中定义的构建参数，则构建会输出警告。

    [Warning] One or more build-args [foo] were not consumed.
Dockerfile可以包括一个或多个ARG指令。例如，以下是有效的Dockerfile：
    
    FROM busybox
    ARG user1
    ARG buildno

> ***警告***：建议不要使用构建时变量来传递github密钥，用户凭据等秘密docker history。使用该命令，构建时变量值对于映像的任何用户都是可见的。  

#### 默认值
ARG指令可以包括一个默认值：

    FROM busybox
    ARG user1=someuser
    ARG buildno=1
如果ARG指令具有默认值，并且在构建时没有传递值，则构建器将使用默认值。

#### 使用ARG变量
您可以使用ARG或ENV指令指定指令可用的变量RUN。使用该ENV指令定义的环境变量 始终覆盖ARG同名指令。考虑这个Dockerfile和一个ENV和ARG指令。

    1 FROM ubuntu
    2 ARG CONT_IMG_VER
    3 ENV CONT_IMG_VER v1.0.0
    4 RUN echo $CONT_IMG_VER
然后，假设使用此命令构建此映像：

$ docker build --build-arg CONT_IMG_VER=v2.0.1 .
在这种情况下，RUN指令使用v1.0.0而不是ARG用户传递的设置：v2.0.1此行为类似于shell脚本，其中本地范围的变量覆盖作为参数传递的变量或从其定义的环境继承的变量。

使用上面的示例，但不同的ENV规范，您可以创建更多有用的交互ARG和ENV指令：

    1 FROM ubuntu
    2 ARG CONT_IMG_VER
    3 ENV CONT_IMG_VER ${CONT_IMG_VER:-v1.0.0}
    4 RUN echo $CONT_IMG_VER
与ARG指令不同，ENV值始终保留在构建的图像中。考虑没有--build-arg标志的docker构建：

    $ docker build .
使用此Dockerfile示例CONT_IMG_VER仍然保留在图像中，但其值将是指令v1.0.0中第3行的默认值ENV。

此示例中的变量扩展技术允许您从命令行传递参数，并通过利用该ENV指令将它们保存在最终图像中 。


### ONBUILD

    ONBUILD [INSTRUCTION]
一个ONBUILD命令将当前执行后Dockerfile构建完成。 ONBUILD在任何导出FROM当前图像的子图像中执行。将该ONBUILD命令视为父母Dockerfile给孩子的指令Dockerfile。

Docker构建ONBUILD在子代中的任何命令之前执行命令 Dockerfile。ONBUILD对于将要构建FROM给定图像的图像非常有用。
[node.js的例子](https://docker_practice.gitee.io/image/dockerfile/onbuild.html)

### HEALTHCHECK
该HEALTHCHECK指令有两种形式：

    HEALTHCHECK [OPTIONS] CMD command （通过在容器内运行命令来检查容器运行状况）
    HEALTHCHECK NONE （禁用从基础映像继承的任何运行状况检查）
该HEALTHCHECK指令告诉Docker如何测试容器以检查它是否仍在工作。即使服务器进程仍在运行，这也可以检测到陷入无限循环且无法处理新连接的Web服务器等情况。

当容器指定了运行状况检查时，除了正常状态外，它还具有运行状况。这个状态最初是starting。每当健康检查通过时，它就会变成healthy（以前所处的状态）。经过一定数量的连续失败后，它就变成了unhealthy。

以前可以出现的选项CMD是：

* --interval=DURATION（默认值：30s）
* --timeout=DURATION（默认值：30s）
* --start-period=DURATION（默认值：0s）
* --retries=N（默认值：3）
运行状况检查将首先在容器启动后的间隔秒运行，然后在每次上一次检查完成后再间隔秒。

如果单次运行的检查花费的时间超过超时秒数，那么检查将被视为失败。

它需要重试连续的健康检查失败才能考虑容器unhealthy。

start period为需要时间引导的容器提供初始化时间。在此期间探测失败将不计入最大重试次数。但是，如果在启动期间运行状况检查成功，则会将容器视为已启动，并且所有连续失败将计入最大重试次数。

HEALTHCHECKDockerfile中只能有一条指令。如果列出多个，则只有最后一个HEALTHCHECK生效。

命令的退出状态指示容器的运行状况。可能的值是：

* 0：成功 - 容器健康且随时可用
* 1：不健康 - 容器无法正常工作
* 2：保留 - 不要使用此退出代码

例如，要检查每五分钟左右网络服务器能够在三秒钟内为网站的主页面提供服务：

    HEALTHCHECK --interval=5m --timeout=3s \
    CMD curl -f http://localhost/ || exit 1
为了帮助调试失败的探测器，命令在stdout或stderr上写入的任何输出文本（UTF-8编码）都将存储在运行状况中并可以查询 docker inspect。此类输出应保持较短（目前仅存储前4096个字节）。

当容器的运行状况更改时，将health_status生成具有新状态的事件。

该HEALTHCHECK功能已添加到Docker 1.12中。

为了帮助排障，健康检查命令的输出（包括 stdout 以及 stderr）都会被存储于健康状态里，可以用 docker inspect 来查看。
```
docker inspect --format '{{json .State.Health}}' web | python -m json.tool
{
    "FailingStreak": 0,
    "Log": [
        {
            "End": "2016-11-25T14:35:37.940957051Z",
            "ExitCode": 0,
            "Output": "<!DOCTYPE html>\n<html>\n<head>\n<title>Welcome to nginx!</title>\n<style>\n    body {\n        width: 35em;\n        margin: 0 auto;\n        font-family: Tahoma, Verdana, Arial, sans-serif;\n    }\n</style>\n</head>\n<body>\n<h1>Welcome to nginx!</h1>\n<p>If you see this page, the nginx web server is successfully installed and\nworking. Further configuration is required.</p>\n\n<p>For online documentation and support please refer to\n<a href=\"http://nginx.org/\">nginx.org</a>.<br/>\nCommercial support is available at\n<a href=\"http://nginx.com/\">nginx.com</a>.</p>\n\n<p><em>Thank you for using nginx.</em></p>\n</body>\n</html>\n",
            "Start": "2016-11-25T14:35:37.780192565Z"
        }
    ],
    "Status": "healthy"
}
```

### SHELL
windows使用，详见官方文档[SHELL](https://docs.docker.com/engine/reference/#shell)