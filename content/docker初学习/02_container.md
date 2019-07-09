## 构建程序
```python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```
在此过程中需要安装Python，pip，flask，redis包；包管理器pip是Python自带，然后使用pip install xx即可
再安装一个pip install pipreqs，然后使用pipreqs ./ 生成所需requirements.txt
```
Flask==1.1.0
redis==3.2.1
```
本机运行测试python web.py
请求连接127.0.0.1:80
```
Hello world!
Hostname: DESKTOP-IRMQRD4
Visits: cannot connect to Redis, counter disabled
```
最后，因为Redis没有运行（因为我们只安装了Python库，而不是Redis本身），我们应该期望在这里使用它的尝试失败并产生错误消息。
注意：在容器内部访问容器ID时，访问主机名称，这类似于正在运行的可执行文件的进程ID。
所以后面想要在容器中访问redis主机，直接开启一个容器名为redis的容器化服务即可。

## 建立dockerfile
```dockerfile
# Use an official Python runtime as a parent image
FROM python:3.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

## 构建
```
PS C:\Users\Administrator\Documents\blog_hugo\content\docker初学习\demo\01_start> ls     

    目录: C:\Users\Administrator\Documents\blog_hugo\content\docker初学习\demo\01_start

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         2019/7/6     10:34            530 Dockerfile
-a----         2019/7/6     10:23             28 requirements.txt
-a----         2019/7/6     10:13            687 web.py
```
```
PS C:\Users\Administrator\Documents\blog_hugo\content\docker初学习\demo\01_start> docker build -t friendlyhello .        
Sending build context to Docker daemon   5.12kB
Step 1/7 : FROM python:3.7-slim
Step 2/7 : WORKDIR /app
 ---> Running in 0cb56707a00d
Removing intermediate container 0cb56707a00d
 ---> a8145cb6dcb4
Step 3/7 : ADD . /app
 ---> d71f8db01165
Step 4/7 : RUN pip install --trusted-host pypi.python.org -r requirements.txt
 ---> Running in 8d39c847b823
Collecting Flask==1.1.0 (from -r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/c3/31/6904ac846fc65a7fa6cac8b4ddc392ce96ca08ee67b0f97854e9575bbb26/Flask-1.1.0-py2.py3-none-any.whl (94kB)Collecting redis==3.2.1 (from -r requirements.txt (line 2))
  Downloading https://files.pythonhosted.org/packages/ac/a7/cff10cc5f1180834a3ed564d148fb4329c989cbb1f2e196fc9a10fa07072/redis-3.2.1-py2.py3-none-any.whl (65kB)Collecting click>=5.1 (from Flask==1.1.0->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/fa/37/45185cb5abbc30d7257104c434fe0b07e5a195a6847506c074527aa599ec/Click-7.0-py2.py3-none-any.whl (81kB)
Collecting itsdangerous>=0.24 (from Flask==1.1.0->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/76/ae/44b03b253d6fade317f32c24d100b3b35c2239807046a4c953c7b89fa49e/itsdangerous-1.1.0-py2.py3-none-any.whl
Collecting Jinja2>=2.10.1 (from Flask==1.1.0->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/1d/e7/fd8b501e7a6dfe492a433deb7b9d833d39ca74916fa8bc63dd1a4947a671/Jinja2-2.10.1-py2.py3-none-any.whl (124kB)
Collecting Werkzeug>=0.15 (from Flask==1.1.0->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/9f/57/92a497e38161ce40606c27a86759c6b92dd34fcdb33f64171ec559257c02/Werkzeug-0.15.4-py2.py3-none-any.whl (327kB)
Collecting MarkupSafe>=0.23 (from Jinja2>=2.10.1->Flask==1.1.0->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/98/7b/ff284bd8c80654e471b769062a9b43cc5d03e7a615048d96f4619df8d420/MarkupSafe-1.1.1-cp37-cp37m-manylinux1_x86_64.whl
Installing collected packages: click, itsdangerous, MarkupSafe, Jinja2, Werkzeug, Flask, redis
Successfully installed Flask-1.1.0 Jinja2-2.10.1 MarkupSafe-1.1.1 Werkzeug-0.15.4 click-7.0 itsdangerous-1.1.0 redis-3.2.1
Removing intermediate container 8d39c847b823
 ---> e5856401a1de
Step 5/7 : EXPOSE 80
 ---> Running in 92e54457acf3
Removing intermediate container 92e54457acf3
 ---> 796dcd5951d4
Step 6/7 : ENV NAME World
 ---> Running in 95e7845100e7
Removing intermediate container 95e7845100e7
 ---> 386f86d928bb
Step 7/7 : CMD ["python", "app.py"]
 ---> Running in e96b64dbec72
Removing intermediate container e96b64dbec72
 ---> e95dbe02e826
Successfully built e95dbe02e826
Successfully tagged friendlyhello:latest
SECURITY WARNING: You are building a Docker image from Windows against a non-Windows Docker host. All files and directories added to build context will have '-rwxr-xr-x' permissions. It is recommended to double check and reset permissions for sensitive files and directories.
```
查看镜像
```
PS C:\Users\Administrator> docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
friendlyhello       latest              e95dbe02e826        2 minutes ago       153MB
redis               latest              bb0ab8a99fe6        2 days ago          95MB
python              3.7-slim            338ae06dfca5        3 weeks ago         143MB
mysql               latest              c7109f74d339        3 weeks ago         443MB
hello-world         latest              fce289e99eb9        6 months ago        1.84kB
```
成功构建，大小153M
```
PS C:\Users\Administrator> docker run -p 4000:80 friendlyhello
python: can't open file 'app.py': [Errno 2] No such file or directory
```
报错了，无法找到这个文件，因为我的文件名不是app.py ，这个时候就需要重新构建，或者其他办法,我这边追求一个命令，覆盖原dockerfile中最后一个CMD命令
```
PS C:\Users\Administrator> docker run -p 4000:80 friendlyhello
python: can't open file 'app.py': [Errno 2] No such file or directory
PS C:\Users\Administrator> docker run -p 4000:80 friendlyhello python web.py
 * Serving Flask app "web" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
 ```
 应该看到Python正在为应用提供服务的消息http://0.0.0.0:80。但是该消息来自容器内部，它不知道将该容器的端口80映射到4000，从而生成正确的URL http://localhost:4000。

点击CTRL+C你的终端退出。

注意：在Windows系统上，CTRL+C不会停止容器。因此，首先键入CTRL+C以获取提示（或打开另一个shell），然后键入 docker container ls以列出正在运行的容器，然后 docker container stop <Container NAME or ID>停止容器。否则，当尝试在下一步中重新运行容器时，会从守护程序收到错误响应。
```
PS C:\Users\Administrator> docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
d8ca49999467        friendlyhello       "python web.py"          3 minutes ago       Up 3 minutes        0.0.0.0:4000->80/tcp     upbeat_driscoll
66bdfb17f623        redis               "docker-entrypoint.s…"   About an hour ago   Up About an hour    0.0.0.0:6379->6379/tcp   redis_test
PS C:\Users\Administrator> docker stop d8ca49999467
d8ca49999467
```

## 分享镜像
类似于Git和github
1. 创建dockerhub用户并登陆
```
    docker login
```


2. 打标签

将本地映像与注册表上的存储库相关联的表示法是 username/repository:tag。标签是可选的，但建议使用，因为它是注册管理机构用来为Docker镜像提供版本的机制。为存储库提供存储库和标记有意义的名称，例如 get-started:part2。这会将图像放入get-started存储库并将其标记为part2。

现在，把它们放在一起来标记图像。docker tag image使用用户名，存储库和标记名称运行，以便将图像上载到所需的目标位置。该命令的语法是：
```
docker tag image username/repository:tag
```
例如：
```
docker tag friendlyhello jansonlv/get-started:v0.1
```
运行docker image ls以查看新标记的图像。
```
PS C:\Users\Administrator> docker image ls
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
jansonlv/get-started   v0.1                e95dbe02e826        14 minutes ago      153MB
friendlyhello          latest              e95dbe02e826        14 minutes ago      153MB
redis                  latest              bb0ab8a99fe6        2 days ago          95MB
python                 3.7-slim            338ae06dfca5        3 weeks ago         143MB
mysql                  latest              c7109f74d339        3 weeks ago         443MB
hello-world            latest              fce289e99eb9        6 months ago        1.84kB
```

3. 发布镜像
将标记的图像上传到存储库：
```
docker push username/repository:tag
```
完成后，此上传的结果将公开发布。如果登录到Docker Hub，则会看到其中的新图像及其pull命令。

从远程存储库中拉出并运行映像
从现在开始，可以使用docker run以下命令在任何计算机上使用和运行应用程序：
```
docker run -p 4000:80 username/repository:tag
```

## 回顾
```
docker build -t friendlyhello .  # Create image using this directory's Dockerfile
docker run -p 4000:80 friendlyhello  # Run "friendlyname" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyhello         # Same thing, but in detached mode
docker container ls                                # List all running containers
docker container ls -a             # List all containers, even those not running
docker container stop <hash>           # Gracefully stop the specified container
docker container kill <hash>         # Force shutdown of the specified container
docker container rm <hash>        # Remove specified container from this machine
docker container rm $(docker container ls -a -q)         # Remove all containers
docker image ls -a                             # List all images on this machine
docker image rm <image id>            # Remove specified image from this machine
docker image rm $(docker image ls -a -q)   # Remove all images from this machine
docker login             # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag  # Tag <image> for upload to registry
docker push username/repository:tag            # Upload tagged image to registry
docker run username/repository:tag  
```