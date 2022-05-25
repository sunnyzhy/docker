# docker-nginx

## 前言

- nginx 版本: ```nginx:latest```

- 网络配置: 驱动类型为 bridge，名称为 nginx

- 容器与宿主机映射
    |容器名称|容器IP|容器的端口|宿主机IP|映射到宿主机的端口|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|--|
    |nginx|-|80|192.168.204.107|80|/usr/local/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf<br />/usr/local/docker/nginx/html:/usr/share/nginx/html<br />/usr/local/docker/nginx/logs:/var/log/nginx|

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

## 创建网络

```bash
# docker network create nginx

# docker network ls
NETWORK ID     NAME            DRIVER    SCOPE
b0308476ccc3   bridge          bridge    local
3724599c2452   host            host      local
d61017ea048b   mysql           bridge    local
9528cdf78a57   nginx           bridge    local
cad7d065639d   none            null      local

# docker network inspect nginx
[
    {
        "Name": "nginx",
        "Id": "9528cdf78a579e4b2e809e092b2ce349581587df9f81a1487571150f1f50af7c",
        "Created": "2022-05-24T23:03:56.56606096-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "Gateway": "172.21.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

## 在宿主机上创建 nginx 目录

```bash
# mkdir -p /usr/local/docker/nginx/{conf,html,logs}

# ls /usr/local/docker/nginx
conf  html  logs
```

## 把容器中的 nginx.conf 复制到宿主机的 nginx/conf 目录

```bash
# docker run --name nginx --net nginx -p 80:80 -d nginx:latest

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
  networks:
   - nginx
networks:
  nginx:
    name: nginx
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

```bash
# curl -XGET http://192.168.204.107
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
