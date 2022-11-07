# docker 命令

## 启动 & 停止 docker 服务

``` bash
# systemctl start docker

# systemctl stop docker

# systemctl restart docker
```

## 查看 docker 版本

``` bash
# docker -v
```

## 创建镜像（. 表示当前路径）

``` bash
# docker build -t <image name> .
```

## 查看本地的镜像

``` bash
# docker images
```

## 创建容器并运行

``` bash
# docker run -p <host port>:<container port> -d --name <container name> <image name>
```

## 查看本地的容器

``` bash
# docker ps -a
```

## 启动 & 停止容器

``` bash
# docker start <container name>|<container id>

# docker stop <container name>|<container id>
```

## 删除容器

``` bash
# docker rm <container name>|<container id>
```

## 删除镜像

``` bash
# docker rmi <repository>|<image id>

# docker rmi <repository>:<tag>
```

***删除镜像的时候，要先删除容器，再删除镜像。***

## 进入 docker 容器

``` bash
# docker exec -it <container id> /bin/sh

# ls
bin   dev  home  lib64	mnt  proc  run	 srv  tmp  var
boot  etc  lib	 media	opt  root  sbin  sys  usr

# exit
```

## 查看日志

``` bash
# docker logs <container id/name>
```

## 从主机拷贝文件或文件夹到容器

``` bash
# docker cp <src> <container id>:<dest>
```

## 从容器拷贝文件或文件夹到主机

``` bash
# docker cp <container id>:<src> <dest>
```

## 设置 docker 时间与宿主机时间同步

### 方法1 在 docker 容器里修改时区

``` bash
# docker exec -it <container id> /bin/sh
# cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### 方法2 从主机拷贝时区文件到容器

``` bash
# docker cp /usr/share/zoneinfo/Asia/Shanghai <container id>:/etc/localtime
```

### 方法3 修改 Dockerfile
``` bash
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

## 设置 jdk 读取时间文件

### 方法1 在 docker 容器里修改时区

``` bash
# docker exec -it <container id> /bin/sh
# echo "Asia/shanghai" > /etc/timezone
```

### 方法2 修改 Dockerfile

``` bash
RUN echo "Asia/shanghai" > /etc/timezone
```

## 清理数据卷/镜像/容器

### 查看没有被使用的数据卷

```bash
# docker volume ls -qf dangling=true
```

### 删除没有被使用的数据卷

```bash
# docker volume rm $(docker volume ls -qf dangling=true)
```

### 删除没有被使用的镜像

- 删除名称或标签为 none 的镜像
    ```bash
    # docker rmi $(docker images | grep "<none>" | awk "{print $3}")
    ```
- 删除没有被使用的镜像
    ```bash
    # docker image prune -a
    ```

### 删除已经停止运行的容器

```bash
# docker rm $(docker ps --all -q -f status=exited)
```

### 删除关闭的容器、被使用的数据卷和网络

```bash
# docker system prune
```
