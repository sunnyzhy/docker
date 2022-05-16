# 创建 docker 镜像

## 创建docker目录

``` bash
# mkdir /usr/local/docker

# cd /usr/local/docker
```

## 准备jar项目

``` bash
# ls
mythreejs-0.0.1-SNAPSHOT.jar
```

## 拉取 java 环境

``` bash
# ls
docker pull java:8
```

## 创建Dockerfile

``` bash
# vim Dockerfile
FROM java:8
LABEL maintainer="zhy"
ADD mythreejs-0.0.1-SNAPSHOT.jar /usr/local/java/app.jar
EXPOSE 8006
ENTRYPOINT ["java","-jar","/usr/local/java/app.jar"]

# ls
Dockerfile  mythreejs-0.0.1-SNAPSHOT.jar
```

## 创建镜像

``` bash
# docker build -t zhy/threejs .
Sending build context to Docker daemon  27.9 MB
Step 1 : FROM openjdk
 ---> 74c95c985a85
Step 2 : ADD mythreejs-0.0.1-SNAPSHOT.jar /usr/local/java/threejs.jar
 ---> c3ff00dbc5d5
Removing intermediate container 0cfb71ed25be
Step 3 : EXPOSE 8006
 ---> Running in a7f69ecb2dd4
 ---> 2fbe9e154cc4
Removing intermediate container a7f69ecb2dd4
Step 4 : ENTRYPOINT java -jar /usr/local/java/threejs.jar
 ---> Running in 1aed9554e90c
 ---> 5f1cb4b65d99
Removing intermediate container 1aed9554e90c
Successfully built 5f1cb4b65d99
```

## 查看镜像

``` bash
# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
zhy/threejs         latest              04565f4995f8        56 seconds ago      220.4 MB
hello-world         latest              f054dc87ed76        3 weeks ago         1.84 kB
```

## 创建容器并运行

``` bash
# docker run -p 8006:8006 -d --name threejs zhy/threejs
daa19a6095d80856e710b0bad2b54d0f4c7b9653dab68e4fc6b5db8ff6fd7704
```

## 查看容器

``` bash
# docker ps -a
CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS                         PORTS                    NAMES
daa19a6095d8        zhy/threejs          "java -jar /usr/local"   14 minutes ago      Up 14 minutes                  0.0.0.0:8006->8006/tcp   threejs
d749f0f35395        hello-world          "/hello"                 About an hour ago   Exited (0) About an hour ago                            kickass_thompson
```
