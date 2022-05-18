# docker-compose

## 前言

- docker 版本: 20.10.16
- docker-compose 版本: 2.5.0

***注: 版本一定要对应好，否则 docker-compose 在 build 的时候容易出问题。***

## 安装 docker-compose

[docker-compose官网](https://github.com/docker/compose/releases 'docker-compose')

```bash
# curl -L https://github.com/docker/compose/releases/download/v2.5.0/docker-compose-linux-x86_64 > /usr/local/bin/docker-compose

# chmod +x /usr/local/bin/docker-compose

# vim /etc/profile
export PATH=$PATH:/usr/local/bin

# source /etc/profile

# docker-compose -v
Docker Compose version v2.5.0
```

如果 ```/usr/local/bin``` 已经配置到了环境变量，就不用再配置。

## 通过 image 指定镜像

### 配置 docker-compose.yml

```bash
# mkdir -p /usr/local/docker

# cd /usr/local/docker

# vim docker-compose.yml
version: "3.9"
services:
  nginx:
    container_name: nginx
    image: nginx:latest
    restart: always
    ports:
      - 80:80
    volumes:
      - /usr/local/www:/usr/share/nginx/html
```

参数说明:

- version: 指定语法格式的版本
- services: 定义服务
   - nginx: 自定义服务的名称
      - container_name: 容器的名称
      - image: 镜像
      - restart: always, 随 docker 服务的启动而启动
      - ports: 映射的端口
      - volumes: 本地与容器挂载的目录，格式: ```宿主服务器的文件目录:docker容器的文件目录```

### 启动

```bash
# cd /usr/local/docker

# docker-compose up -d

# docker ps
CONTAINER ID        IMAGE                                               COMMAND                  CREATED             STATUS              PORTS                  NAMES
42eccfb6b6f5        nginx:latest                                        "/docker-entrypoin..."   16 seconds ago      Up 5 seconds        0.0.0.0:80->80/tcp     nginx

# echo "hello world" > /usr/local/www/hello.html

# curl -XGET http://localhost:80/hello.html
hello world

# docker-compose stop
```

***注: 必须在 ```.yml``` 配置文件的目录下执行 ```docker-compose up```；或者使用以下方式执行 ```docker-compose up```:***

```bash
# docker-compose -f /usr/local/docker/docker-compose.yml up -d
```

## 通过 build （需要 Dockerfile）生成镜像

### 配置 Dockerfile

```bash
# mkdir /usr/local/dockerfile

# vim /usr/local/dockerfile/Dockerfile-nginx
FROM nginx:latest
EXPOSE 80
```

### 配置 docker-compose.yml

```bash
# cd /usr/local/docker

# vim dockerfile-compose.yml
version: "3.9"
services:
  nginx:
    build:
      context: /usr/local/dockerfile
      dockerfile: Dockerfile-nginx
    container_name: nginx
    image: zhy/nginx:latest
    restart: always
    ports:
      - 80:80
    volumes:
      - /usr/local/www:/usr/share/nginx/html

# docker-compose -f /usr/local/docker/dockerfile-compose.yml up -d

# docker ps
CONTAINER ID   IMAGE              COMMAND                  CREATED         STATUS         PORTS                                   NAMES
19a04e250bfe   zhy/nginx:latest   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8018->80/tcp, :::8018->80/tcp   nginx

# docker images
REPOSITORY                                      TAG       IMAGE ID       CREATED        SIZE
nginx                                           latest    605c77e624dd   4 months ago   141MB
zhy/nginx                                       latest    910b61fb39f5   4 months ago   141MB
```

参数说明:

- build: 指定 Dockerfile 所在文件夹的路径（可以是绝对路径，或者相对 docker-compose.yml 文件的路径），Compose 将会利用它自动构建这个镜像，然后使用这个镜像
    ```yml
    # 绝对路径
    build: /path/build

    # 相对路径
    build: ./build

    # 设定上下文跟目录，以此目录指定 Dockerfile
    build:
        context: ../
        dockerfile: path/Dockerfile-alternate

    # 给 Dockerfile 构建的镜像命名
    build: ./dir
    image: zhy/nginx:latest

    # 构建过程中指定环境变量，构建成功后取消
    build:
       context: .
       args:
          buildno: 1
          password: secret
    ```
- image: 如果同时指定 ```build``` 和 ```image``` 两个标签，那么 Compose 先构建镜像，然后把镜像命名为 ```image``` 指定的名字

### 启动

```bash
# cd /usr/local/docker

# docker-compose -f /usr/local/docker/dockerfile-compose.yml up -d

# docker ps
CONTAINER ID   IMAGE              COMMAND                  CREATED         STATUS         PORTS                                   NAMES
19a04e250bfe   zhy/nginx:latest   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8018->80/tcp, :::8018->80/tcp   nginx

# docker images
REPOSITORY                                      TAG       IMAGE ID       CREATED        SIZE
nginx                                           latest    605c77e624dd   4 months ago   141MB
zhy/nginx                                       latest    910b61fb39f5   4 months ago   141MB

# echo "hello docker" > /usr/local/www/docker.html

# curl -XGET http://localhost:80/docker.html
hello docker

# docker-compose -f /usr/local/docker/dockerfile-compose.yml stop
```
