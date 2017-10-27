# 安装docker
``` javascript
# yum install -y docker
```

# 启动 & 停止docker
``` javascript
# systemctl start docker

# systemctl stop docker
```

# 验证docker是否安装成功
``` javascript
# docker -v
Docker version 1.12.6, build 85d7426/1.12.6

# docker run hello-world
Unable to find image 'hello-world:latest' locally
Trying to pull repository docker.io/library/hello-world ...
latest: Pulling from docker.io/library/hello-world
5b0f327be733: Pull complete
Digest: sha256:07d5f7800dfe37b8c2196c7b1c524c33808ce2e0f74e7aa00e603295ca9a0972

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```

# 查看本地的容器
``` javascript
# docker ps -a
CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS                     PORTS               NAMES
e3c5b66e8fa6        hello-world          "/hello"                 18 hours ago        Exited (0) 18 hours ago                        big_leakey
```

# 查看本地的镜像
``` javascript
# docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
docker.io/hello-world   latest              05a3bd381fc2        6 weeks ago         1.84 kB
```

# 配置阿里云镜像
需要先注册阿里云的开发者账号https://dev.aliyun.com/search.html，然后记下加速器地址。
``` javascript
# vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://6848w7y3.mirror.aliyuncs.com"]
}

# systemctl daemon-reload

# systemctl restart docker
```