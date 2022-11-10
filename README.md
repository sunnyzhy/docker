# 简介

## Docker Compose 和 Docker Swarm 的区别

Docker Compose 和 Docker Swarm 都是 Docker 官方容器编排项目，但不同的是：

- Docker Compose 可以在单个服务器或主机上创建多个容器，主要解决本地 docker 容器编排问题; Docker Swarm 可以在多个服务器或主机上创建容器集群服务，主要解决跨主机的 docker 集群容器编排问题
- Docker Compose 的网络驱动类型为 bridge，用来保证在同一个主机上的容器网络互通; Docker Swarm 的网络驱动类型为 overlay，用来保证在不同主机上的容器网络互通
- Docker Compose 停止与删除容器的命令为先执行 ```docker stop $container_id```，再执行命令 ```docker rm $container_id```; Docker Swarm 停止与删除容器的命令为 ```docker service rm $service_id```

## 挂载目录

```bash
docker run -it -v <宿主机目录>:<容器目录> <IMAGE>:<TAG>
```
