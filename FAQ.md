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

## 无法删除镜像 Error：No such image：镜像ID

```bash
# docker images -a
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
hello-world   latest    feb5d9fea6a5   9 months ago   13.3kB

# docker rmi feb5d9fea6a5
Error: No such image: feb5d9fea6a5
```

解决方法:

1. 查找 ```镜像ID=feb5d9fea6a5``` 的文件id
    ```bash
    # docker image inspect feb5d9fea6a5 | sed 's/,/\n/g' | grep "Id" | sed 's/:/\n/g' | sed '1d' | sed 's/"//g' | sed '1d'
    feb5d9fea6a5e9606aa995e879d862b825965ba48de054caab5ef356dc6b3412
    ```

2. 删除 id 是 ```feb5d9fea6a5e9606aa995e879d862b825965ba48de054caab5ef356dc6b3412``` 的文件
    ```bash
    # ls /var/lib/docker/image/overlay2/imagedb/content/sha256
    605c77e624ddb75e6110f997c58876baa13f8754486b461117934b24a9dc3a85
    d23bdf5b1b1b1afce5f1d0fd33e7ed8afbc084b594b9ccf742a5b27080d8a4a8
    feb5d9fea6a5e9606aa995e879d862b825965ba48de054caab5ef356dc6b3412

    # rm -rf /var/lib/docker/image/overlay2/imagedb/content/sha256/feb5d9fea6a5e9606aa995e879d862b825965ba48de054caab5ef356dc6b3412

    # ls /var/lib/docker/image/overlay2/imagedb/content/sha256
    605c77e624ddb75e6110f997c58876baa13f8754486b461117934b24a9dc3a85
    d23bdf5b1b1b1afce5f1d0fd33e7ed8afbc084b594b9ccf742a5b27080d8a4a8

    # docker images -a
    REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
    ```
    或
    ```bash
    # rm -rf /var/lib/docker/image/overlay2/imagedb/content/sha256/$(docker image inspect feb5d9fea6a5 | sed 's/,/\n/g' | grep "Id" | sed 's/:/\n/g' | sed '1d' | sed 's/"//g' | sed '1d')
    ```

3. ***终极解决方法:***
    ```bash
    # systemctl stop docker

    # rm -rf /var/lib/docker

    # systemctl start docker
    ```
