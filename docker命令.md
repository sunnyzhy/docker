# 启动 & 停止docker服务
``` javascript
# systemctl start docker

# systemctl stop docker

# systemctl restart docker
```

# 查看docker版本
``` javascript
# docker -v
```

# 创建镜像（. 表示当前路径）
``` javascript
# docker build -t <image name> .
```

# 查看本地的镜像
``` javascript
# docker images
```

# 创建容器并运行
``` javascript
# docker run -p <host port>:<container port> -d --name <container name> <image name>
```

# 查看本地的容器
``` javascript
# docker ps -a
```

# 启动 & 停止容器
``` javascript
# docker start <container name>|<container id>

# docker stop <container name>|<container id>
```

# 删除容器
``` javascript
# docker rm <container name>|<container id>
```

# 删除镜像
``` javascript
# docker rmi <image name>|<image id>
```

# 进入docker容器
``` javascript
# docker exec -it <container id> /bin/sh

# ls
bin   dev  home  lib64	mnt  proc  run	 srv  tmp  var
boot  etc  lib	 media	opt  root  sbin  sys  usr

# exit
```

# 从主机拷贝文件或文件夹到容器
``` javascript
# docker cp <src> <container id>:<dest>
```

# 从容器拷贝文件或文件夹到主机
``` javascript
# docker cp <container id>:<src> <dest>
```

# 设置docker时间与宿主机时间同步
### 方法1 在docker容器里修改时区
``` javascript
# docker exec -it <container id> /bin/sh
# cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
### 方法2 从主机拷贝时区文件到容器
``` javascript
# docker cp /usr/share/zoneinfo/Asia/Shanghai <container id>:/etc/localtime
```
### 方法3 修改Dockerfile
``` bash
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

# 设置jdk读取时间文件
### 方法1 在docker容器里修改时区
``` javascript
# docker exec -it <container id> /bin/sh
# echo "Asia/shanghai" > /etc/timezone
```

### 方法2 修改Dockerfile
``` bash
RUN echo "Asia/shanghai" > /etc/timezone
```
