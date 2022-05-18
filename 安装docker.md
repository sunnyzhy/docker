# 安装 docker

## 卸载旧版本

```bash
# yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

## 安装 repository

``` bash
# yum install -y yum-utils

# yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# yum-config-manager --enable docker-ce-nightly

# yum-config-manager --enable docker-ce-test

# yum-config-manager --disable docker-ce-nightly
```

## 安装 Docker Engine

``` bash
# yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# docker -v
Docker version 20.10.16, build aa7e414
```

## 启动 & 停止 docker

``` bash
# systemctl start docker

# systemctl stop docker
```

## 配置阿里云镜像

需要先注册阿里云的开发者账号 ```https://dev.aliyun.com/search.html```，然后记下加速器地址。

``` bash
# vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://6848w7y3.mirror.aliyuncs.com"]
}

# systemctl daemon-reload

# systemctl restart docker
```

## 验证 docker 是否安装成功

``` bash
# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
Digest: sha256:2498fce14358aa50ead0cc6c19990fc6ff866ce72aeb5546e1d59caac3d0d60f
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

## 查看本地的容器

``` bash
# docker ps -a
CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS                     PORTS               NAMES
e3c5b66e8fa6        hello-world          "/hello"                 18 hours ago        Exited (0) 18 hours ago                        big_leakey
```

## 查看本地的镜像

``` bash
# docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
docker.io/hello-world   latest              05a3bd381fc2        6 weeks ago         1.84 kB
```
