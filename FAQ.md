# FAQ

## WARN[0000] Found orphan containers ([mmm-1 nnn-2]) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up. 

- 原因

    如果将多个 docker-compose.yml 放在同一个目录下，在 docker 运行时生成的镜像实例会有相同的前缀，就是当前的目录名，也就是说默认相同前缀的是同一组实例，所以在当前目录下还有别的 xxx-docker-compose.yml 配置文件，在运行时就会出现以上警告

- 解决办法
   1. 在启动时重命名实例
      ```bash
      # docker-compose -p xxx -f /usr/local/docker/rocketmq/xxx-docker-compose.yml up -d
      ```
   2. 将文件放在不同的目录下运行

##  error mounting "/etc/timezone" to rootfs at "/etc/timezone": mount /etc/timezone:/etc/timezone (via /proc/self/fd/6), flags: 0x5000: not a directory

- 原因

     centos7.6 中 ```/etc/timezone``` 是一个文件夹，而不是一个文件。

- 解决办法

   1. 先在宿主机执行如下命令:
      ```bash
      # echo 'Asia/Shanghai' > /etc/timezone/timezone
      ```
   2. 在 docker-compose 中挂载
      ```yml
      volumes:
        - /etc/timezone/timezone:/etc/timezone
        - /etc/localtime:/etc/localtime
      ```

## image has dependent child images

```
Error response from daemon: conflict: unable to delete 镜像ID (cannot be forced) - image has dependent child images
```

- 原因

   该镜像被别的镜像所依赖，不能强制删除

- 解决办法
   
   ```bash
   # docker image inspect --format='{{.RepoTags}} {{.Id}} {{.Parent}}' $(docker image ls -q --filter since=镜像ID)
   ```
   查找 child images，再逐个删除

