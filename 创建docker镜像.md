# 创建 docker 镜像

## 创建docker目录

``` bash
# mkdir -p /usr/local/docker/app/demo
```

## 准备jar项目

``` bash
# ls /usr/local/docker/app/demo
demo-0.0.1.jar
```

日志配置:

```yml
logging:
  file: ./log/run.log
```

## 拉取 java 环境

``` bash
# docker pull java:8
```

## 编辑 Dockerfile

``` bash
# vim /usr/local/docker/app/demo/Dockerfile
```

```Dockerfile
FROM java:8

LABEL maintainer="zhy"

ENV JAVA_OPTS="-XX:+HeapDumpOnOutOfMemoryError -Xmx256m -Xms256m"

WORKDIR /usr/local/app/demo

COPY ./demo-0.0.1.jar ./demo-0.0.1.jar

CMD java -jar ${JAVA_OPTS} ./demo-0.0.1.jar

EXPOSE 8081
```

```bash
# ls /usr/local/docker/app/demo
demo-0.0.1.jar  Dockerfile
```

## 创建镜像

### 用 docker 创建镜像

#### 创建镜像

- 命令行: ```docker build -t 镜像名称:镜像tag Dockerfile所在的目录名```, 目录名可以是相对路径也可以是绝对路径

``` bash
# docker build -t demo:0.0.1 /usr/local/docker/app/demo
Sending build context to Docker daemon  60.82MB
Step 1/6 : FROM java:8
 ---> d23bdf5b1b1b
Step 2/6 : LABEL maintainer="zhy"
 ---> Running in b0585a9f6953
Removing intermediate container b0585a9f6953
 ---> 1e432578f2e6
Step 3/6 : ENV JAVA_OPTS="-XX:+HeapDumpOnOutOfMemoryError -Xmx256m -Xms256m"
 ---> Running in 8c1b914ffff1
Removing intermediate container 8c1b914ffff1
 ---> 2f9fd50c2801
Step 4/6 : COPY ./demo-0.0.1.jar /usr/local/app/demo/demo-0.0.1.jar
 ---> 4ba357114fac
Step 5/6 : CMD java -jar ${JAVA_OPTS} /usr/local/app/demo/demo-0.0.1.jar
 ---> Running in 355f435acc60
Removing intermediate container 355f435acc60
 ---> 05d91b958797
Step 6/6 : EXPOSE 8088
 ---> Running in 4276e4c4afc2
Removing intermediate container 4276e4c4afc2
 ---> 7709d3bbc4dc
Successfully built 7709d3bbc4dc
Successfully tagged demo:0.0.1
```

#### 创建容器并运行

``` bash
# docker run -d -p 8081:8081 --name demo -v /usr/local/docker/app/demo/log:/usr/local/app/demo/log demo:0.0.1
```

### 用 docker-compose 创建镜像

#### 编辑 docker-compose.yml

```bash
# vim /usr/local/docker/app/docker-compose.yml
```

```yml
version: '3.9'

services:
 demo:
  build: ./demo
  image: demo:0.0.1
  container_name: demo
  restart: always
  volumes:
   - /usr/local/docker/app/demo/log:/usr/local/app/demo/log
   - /etc/timezone/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  ports:
   - 8081:8081
```

```bash
# ls /usr/local/docker/app
demo  docker-compose.yml

# ls /usr/local/docker/app/demo
demo-0.0.1.jar  Dockerfile
```

#### 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/app/docker-compose.yml up -d
```

## 查看宿主机日志

```bash
# ls /usr/local/docker/app/demo/log
run.log

# tail -f /usr/local/docker/app/demo/log/run.log 
```

## 查看镜像

``` bash
# docker images
REPOSITORY                                      TAG       IMAGE ID       CREATED         SIZE
demo                                            0.0.1     7709d3bbc4dc   2 minutes ago   704MB
```

## 查看容器

``` bash
# docker ps
CONTAINER ID   IMAGE                           COMMAND                  CREATED          STATUS          PORTS                                                                                            NAMES
a6c804538506   demo:0.0.1                      "/bin/sh -c 'java -j…"   20 seconds ago   Up 18 seconds   0.0.0.0:8081->8081/tcp, :::8081->8081/tcp                                                        demo
```
