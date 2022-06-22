# docker-nginx

## 前言

- 技术栈: ```nginx + ftp```，***ftp 主要用于接收客户端上传的文件并与 nginx 共享宿主机的挂载目录***

- nginx 版本: ```nginx:latest```; ftp 版本: ```bogem/ftp:latest```

- 网络配置: 驱动类型为 bridge，名称为 nginx

- nginx 容器与宿主机映射

    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|--|
    |nginx|192.168.2.10|80:80|192.168.204.107|/usr/local/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf<br />/usr/local/docker/nginx/upload:/usr/local/upload<br />/usr/local/docker/nginx/html:/usr/share/nginx/html<br />/usr/local/docker/nginx/logs:/var/log/nginx|

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

- nginx.conf 配置注意事项:
    ```conf
        server {
            listen       80; # listen   443 ssl;
            server_name  192.168.204.107; # server_name 应明确为宿主机 ip
            location / {
                root   /usr/share/nginx/html; # root 应为绝对路径
                index  index.html index.htm;
            }

            location /upload {
                alias /usr/share/nginx/upload; # 同理，alias 应为绝对路径
                autoindex on;
            }

            location /api {
                proxy_pass http://192.168.204.107:8765; # proxy_pass 应明确为宿主机 ip
                proxy_set_header host $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header referer "-";

                proxy_redirect default;
            }
        }
    ```

- ftp 容器与宿主机映射

    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|--|
    |ftp|192.168.2.11|20:20<br />21:21<br />47400-47470:47400-47470<br />|192.168.204.107|/usr/local/docker/nginx/upload:/home/vsftpd|


***注:***

1. ftp 和 nginx 部署在同一个宿主机里
2. ftp 挂载的宿主机目录和 nginx 挂载的宿主机目录相同
3. ftp 客户端上传的文件可以通过 nginx 访问

## 安装 nginx

### 拉取 nginx 镜像

```bash
# docker pull nginx:latest

# docker images
REPOSITORY               TAG       IMAGE ID       CREATED        SIZE
nginx                    latest    605c77e624dd   4 months ago   141MB
```

### 创建网络

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

### 在宿主机上创建 nginx 目录

```bash
# docker run --name nginx --net nginx -p 80:80 -d nginx:latest

# mkdir -p /usr/local/docker/nginx/{conf,html,logs}

# docker cp nginx:/etc/nginx/nginx.conf /usr/local/docker/nginx/conf/

# vim /usr/local/docker/nginx/conf/nginx.conf
```

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
    
    server {
        listen       80; # listen   443 ssl;
        server_name  www.docker.zy;
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}
```

```bash
# vim /usr/local/docker/nginx/html/index.html
```

```html
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

```bash
# chmod -R 777 /usr/local/docker/nginx/{html,logs}

# ls /usr/local/docker/nginx
conf  html  logs
```

### 配置 docker-compose.yml

```bash
# vim /usr/local/docker/nginx/docker-compose.yml
```

```yml
version: '3.9'

services:
 nginx:
  image: nginx:latest
  container_name: nginx
  restart: always
  volumes:
   - /usr/local/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf
   - /usr/local/docker/nginx/upload:/usr/local/upload
   - /usr/local/docker/nginx/html:/usr/share/nginx/html
   - /usr/local/docker/nginx/logs:/var/log/nginx
   - /etc/timezone/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  ports:
   - 80:80
  networks:
    nginx:
      ipv4_address: 192.168.2.10
networks:
  nginx:
    name: nginx
```

### 启动 docker-compose

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

### 查看网络

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

### 访问 nginx

可以访问:

1. 在任一物理机访问 ```$HOST_IP:80```:
   ```bash
   # curl -XGET http://192.168.204.107
   ```
2. 在宿主机访问 ```$Container_IP:80```，即 ```curl -XGET http://192.168.2.10```
   ```bash
   # curl -XGET http://192.168.2.10
   ```

### FAQ

#### directory index of "/usr/share/nginx/html/" is forbidden

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

## 安装 ftp

### 拉取 ftp 镜像

```bash
# docker pull bogem/ftp

# docker images
REPOSITORY                                      TAG           IMAGE ID       CREATED         SIZE
bogem/ftp                                       latest        a40e9c43c530   5 years ago     175MB
```

### 在宿主机上创建 ftp 目录

```bash
# mkdir -p /usr/local/docker/ftp
```

### 配置 docker-compose.yml

```bash
# vim /usr/local/docker/ftp/docker-compose.yml
```

```yml
version: '3.9'

services:
 ftp:
  image: bogem/ftp:latest
  container_name: ftp
  restart: always
  volumes:
   - /usr/local/docker/nginx/upload:/home/vsftpd
   - /etc/timezone/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  environment:
   FTP_USER: admin
   FTP_PASS: admin
   PASV_ADDRESS: 192.168.204.107   # PASV_ADDRESS 应明确为宿主机 ip
  ports:
   - 20:20
   - 21:21
   - 47400-47470:47400-47470
  networks:
    nginx:
      ipv4_address: 192.168.2.11
networks:
  nginx:
    name: nginx
```

### 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/ftp/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container ftp  Started                                                                                        4.4s

# docker ps
CONTAINER ID   IMAGE                           COMMAND                  CREATED          STATUS          PORTS                                                                                                                  NAMES
d12e72ffe24d   bogem/ftp:latest                "/usr/sbin/run-vsftp…"   27 seconds ago   Up 22 seconds   0.0.0.0:20-21->20-21/tcp, :::20-21->20-21/tcp, 0.0.0.0:47400-47470->47400-47470/tcp, :::47400-47470->47400-47470/tcp   ftp
```

### 访问 ftp

在文件资源管理器的地址栏访问 ```ftp://$HOST_IP:21```, 用户名/密码: ```admin/admin``` :

```
ftp://192.168.204.107:21
```
