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
# mkdir -p /etc/docker

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

## 修改 docker-ce.repo 中的 docker 地址

安装 docker 后，使用 yum 安装出现以下问题:

```
https://download.docker.com/linux/centos/7/x86_64/stable/repodata/repomd.xml: [Errno 14] curl#7 - "Failed to connect to 2600:9000:215a:e000:3:db06:4200:93a1: Network is unreachable"
```

解决方法: 修改 docker-ce.repo 中的 docker 地址，把 ```download.docker.com``` 修改为 ```mirrors.aliyun.com/docker-ce```

```bash
# cd /etc/yum.repos.d

# cp docker-ce.repo docker-ce.repo.backup

# sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' docker-ce.repo
```

### 官方的 docker-ce.repo

```repo
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-debuginfo]
name=Docker CE Stable - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/debug-$basearch/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-source]
name=Docker CE Stable - Sources
baseurl=https://download.docker.com/linux/centos/$releasever/source/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test]
name=Docker CE Test - $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/$basearch/test
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-debuginfo]
name=Docker CE Test - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/debug-$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-source]
name=Docker CE Test - Sources
baseurl=https://download.docker.com/linux/centos/$releasever/source/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly]
name=Docker CE Nightly - $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly-debuginfo]
name=Docker CE Nightly - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/debug-$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly-source]
name=Docker CE Nightly - Sources
baseurl=https://download.docker.com/linux/centos/$releasever/source/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg
```

### 修改为阿里云镜像的 docker-ce.repo

```repo
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/$releasever/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-stable-debuginfo]
name=Docker CE Stable - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/$releasever/debug-$basearch/stable
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-stable-source]
name=Docker CE Stable - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/$releasever/source/stable
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test]
name=Docker CE Test - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/$releasever/$basearch/test
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test-debuginfo]
name=Docker CE Test - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/$releasever/debug-$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test-source]
name=Docker CE Test - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/$releasever/source/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-nightly]
name=Docker CE Nightly - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/$releasever/$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-nightly-debuginfo]
name=Docker CE Nightly - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/$releasever/debug-$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-nightly-source]
name=Docker CE Nightly - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/$releasever/source/nightly
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
```

## 修改 docker 的数据存储路径

docker 的数据存储路径默认是 ```/var/lib/docker```

1. 查看当前 docker 的数据存储路径
    ```bash
    # docker info | grep Dir
     Docker Root Dir: /var/lib/docker
    ```

2. 停止 docker 服务
    ```bash
    # systemctl stop docker
    ```

3. 在 docker.service 里使用 ```--data-root``` 参数指定 docker 的数据存储位置
    ```bash
    # vim /usr/lib/systemd/system/docker.service
    ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --data-root=/home/lib/docker 
    ```

4. 把 ```/var/lib/docker``` 迁移至 ```/home/lib/docker```
    ```bash
    # rsync -avzP /var/lib/docker /home/lib

    # rm -rf /var/lib/docker
    ```

5. 启动 docker 服务
    ```bash
    # systemctl daemon-reload

    # systemctl start docker
    ```

6. 查看当前 docker 的数据存储路径
    ```bash
    # docker info | grep Dir
     Docker Root Dir: /home/lib/docker
    ```

## 修改容器内的 apt-get 源

1. 进入容器
    ```bash
    # docker exec -u 0 -it <container id/name> /bin/sh
    ```
2. 查看 apt-get 源
    ```bash
    # cat /etc/apt/sources.list
    ```
    内容:
    ```
    # deb http://snapshot.debian.org/archive/debian/20211220T000000Z bullseye main
    deb http://deb.debian.org/debian bullseye main
    # deb http://snapshot.debian.org/archive/debian-security/20211220T000000Z bullseye-security main
    deb http://security.debian.org/debian-security bullseye-security main
    # deb http://snapshot.debian.org/archive/debian/20211220T000000Z bullseye-updates main
    deb http://deb.debian.org/debian bullseye-updates main
    ```
3. 修改 apt-get 源为阿里云，apt-get 源是其他域名（此处是 deb.debian.org）的修改方法相同
    ```bash
    # sed -i s@/deb.debian.org/@/mirrors.aliyun.com/@g /etc/apt/sources.list
    # sed -i s@/security.debian.org/@/mirrors.aliyun.com/@g /etc/apt/sources.list
    ```
4. 查看 apt-get 源
    ```bash
    # cat /etc/apt/sources.list
    ```
    内容:
    ```
    # deb http://snapshot.debian.org/archive/debian/20211220T000000Z bullseye main
    deb http://mirrors.aliyun.com/debian bullseye main
    # deb http://snapshot.debian.org/archive/debian-security/20211220T000000Z bullseye-security main
    deb http://mirrors.aliyun.com/debian-security bullseye-security main
    # deb http://snapshot.debian.org/archive/debian/20211220T000000Z bullseye-updates main
    deb http://mirrors.aliyun.com/debian bullseye-updates main
    ```
5. 更新 apt-get
    ```bash
    # apt-get clean
    # apt-get update
    ```
