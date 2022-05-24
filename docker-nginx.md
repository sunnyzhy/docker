# docker-nginx

## 前言

- nginx 版本: ```nginx:latest```

- 网络配置: 驱动类型为 bridge

- nginx:latest 镜像配置文件 nginx.conf
    ```conf

    user  nginx;
    worker_processes  auto;

    error_log  /var/log/nginx/error.log notice;
    pid        /var/run/nginx.pid;


    events {
        worker_connections  1024;
    }


    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        sendfile        on;
        #tcp_nopush     on;

        keepalive_timeout  65;

        #gzip  on;

        include /etc/nginx/conf.d/*.conf;
    }
    ```

## 拉取 nginx 镜像

```bash
# docker pull nginx:latest

# docker images
REPOSITORY               TAG       IMAGE ID       CREATED        SIZE
nginx                    latest    605c77e624dd   4 months ago   141MB
```

## 在宿主机上创建 nginx 目录

```bash
# mkdir -p /usr/local/docker/nginx/{conf,html,logs}

# ls /usr/local/docker/nginx
conf  html  logs
```

## 把容器中的 nginx.conf 复制到宿主机的 nginx/conf 目录

```bash
# docker run --name nginx -p 80:80 -d nginx:latest

# docker cp nginx:/etc/nginx/nginx.conf /usr/local/docker/nginx/conf/
```

## 配置 docker-compose.yml

```bash
# vim /usr/local/docker/nginx/docker-compose.yml
version: '3.9'

services:
 nginx:
  image: nginx:latest
  container_name: nginx
  restart: always
  volumes:
   - /usr/local/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf
   - /usr/local/docker/nginx/html:/usr/share/nginx/html
   - /usr/local/docker/nginx/logs:/var/log/nginx
  ports:
   - 80:80
```

## 启动 docker-compose

```bash
# docker stop nginx

# docker rm nginx

# docker-compose -f /usr/local/docker/nginx/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container nginx  Started                                                                                      1.2s

# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED             STATUS             PORTS                                                                                            NAMES
93a18489fe0f   nginx:latest             "/docker-entrypoint.…"   23 seconds ago      Up 22 seconds      0.0.0.0:80->80/tcp, :::80->80/tcp                                                                nginx
```

## 访问 nginx

```
http://${宿主机IP}/
```
