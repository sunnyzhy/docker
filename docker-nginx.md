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
# docker network create nginx --subnet 192.168.2.0/24 --gateway=192.168.2.1

# docker network ls
NETWORK ID     NAME            DRIVER    SCOPE
9528cdf78a57   nginx           bridge    local

# docker network inspect nginx
[
    {
        "Name": "nginx",
        "Id": "715b74d55cf3f87994c1ab7309bd1ad359869b38bad3ecde0fb6f4026a395459",
        "Created": "2022-06-13T03:44:53.017717677-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.2.0/24",
                    "Gateway": "192.168.2.1"
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
# docker run --name nginx --net nginx -p 80:80 -d nginx:latest

# mkdir -p /usr/local/docker/nginx/{conf,html,logs}

# docker cp nginx:/etc/nginx/nginx.conf /usr/local/docker/nginx/conf/

# vim /usr/local/docker/nginx/conf/nginx.conf

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
    
    server {
        listen       80; # listen   443 ssl;
        server_name  www.docker.zy;
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}

# vim /usr/local/docker/nginx/html/index.html
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

# chmod -R 777 /usr/local/docker/nginx/{html,logs}

# ls /usr/local/docker/nginx
conf  html  logs
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
    nginx:
      ipv4_address: 192.168.2.10
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

## 查看网络

```bash
# docker network inspect nginx
[
    {
        "Name": "nginx",
        "Id": "715b74d55cf3f87994c1ab7309bd1ad359869b38bad3ecde0fb6f4026a395459",
        "Created": "2022-06-13T03:44:53.017717677-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.2.0/24",
                    "Gateway": "192.168.2.1"
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
        "Containers": {
            "b1f1794fdc33dbb61a4b3eedd132ee84fff100fa02b1a87b1fd47f0b640f884a": {
                "Name": "nginx",
                "EndpointID": "2edda3dea95aded1bc772f4f1d8d45c3ffb9608c4f685693bf2d524543300eaa",
                "MacAddress": "02:42:c0:a8:02:0a",
                "IPv4Address": "192.168.2.10/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

## 访问 nginx

可以访问:

1. 在任一物理机访问 ```$HOST:80```:
   ```bash
   # curl -XGET http://192.168.204.107
   ```
2. 在宿主机访问 ```$Container_IP:80```，即 ```curl -XGET http://192.168.2.10```
   ```bash
   # curl -XGET http://192.168.2.10
   ```

## FAQ

### directory index of "/usr/share/nginx/html/" is forbidden

- 原因
   1. ```/usr/share/nginx/html``` 下面没有 ```index.html index.htm```
   2. 默认的 nginx 配置文件的 location 没有指定 root 路径

- 解决办法
   1. 在宿主机挂载的 html 目录下创建 ```index.html index.htm```
   2. 在 nginx 配置文件里指定 root 路径:
      ```conf
      location / {
        root   html;
        index  index.html index.htm;
      }
      ```
