# docker - nginx(host)

## 前言

- nginx 版本: ```nginx:latest```

- 网络配置: 驱动类型为 overlay

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nginx 容器与宿主机映射

    |容器名称|宿主机IP|端口映射(宿主机端口:容器端口)|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |nginx|192.168.5.163|80:80|/usr/local/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf<br />/usr/local/docker/nginx/upload:/usr/local/upload<br />/usr/local/docker/nginx/html:/usr/share/nginx/html<br />/usr/local/docker/nginx/logs:/var/log/nginx|

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
                alias /usr/local/upload; # 同理，alias 应为绝对路径
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

## 安装 nginx

### 拉取 nginx 镜像

```bash
# docker pull nginx:latest

# docker images | grep nginx
REPOSITORY               TAG       IMAGE ID       CREATED        SIZE
nginx                    latest    605c77e624dd   4 months ago   141MB
```

### 在宿主机上创建 nginx 目录

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```bash
#!/bin/sh

mkdir -p /usr/local/docker/nginx/{conf,html,logs,upload}
> /usr/local/docker/nginx/conf/nginx.conf
cat << EOF >> /usr/local/docker/nginx/conf/nginx.conf

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
        server_name  localhost;
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        location /upload {
            alias /usr/local/upload;
            autoindex on;
        }
    }
}
EOF

chmod 777 /usr/local/docker/nginx/{html,logs,upload}
```

```bash
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/nginx/
/usr/local/docker/nginx/
├── conf
│   └── nginx.conf
├── html
├── logs
└── upload
```

### 配置 docker-compose.yml

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# vim /usr/local/docker/docker-compose.yml
```

```yml
version: '3.9'

services:
 nginx:
   image: nginx:latest
   volumes:
     - /usr/local/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf
     - /usr/local/docker/nginx/upload:/usr/local/upload
     - /usr/local/docker/nginx/html:/usr/share/nginx/html
     - /usr/local/docker/nginx/logs:/var/log/nginx
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   ports:
     - 80:80
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-163
         - node.role == manager
   networks:
     - iot

networks:
  iot:
    driver: overlay
    external: true
```

### 启动 docker-compose

在宿主机 192.168.5.163(manager) 上启动 nginx 服务:

```bash
# docker stack deploy -c /usr/local/docker/docker-compose.yml iot
Creating service iot_nginx

# docker stack ps iot | grep nginx
zwffbnblsyuc   iot_nginx.1              nginx:latest                                           centos-docker-163   Running         Running about a minute ago                              

# docker service ls | grep nginx
54hbyb2hs9y4   iot_nginx             replicated   1/1        nginx:latest                                           *:80->80/tcp
```

### 查看 nginx 容器

查看宿主机 192.168.5.163 的 nginx 容器:

```bash
# docker ps | grep nginx
1bf65f84ae21   nginx:latest                                           "/docker-entrypoint.…"   About a minute ago   Up About a minute     80/tcp                                 iot_nginx.1.zwffbnblsyuc3k3ibxard2xxa
```

### 访问 nginx

在任一物理机访问 ```$HOST_IP:80```:

```bash
# curl -XGET http://192.168.5.163
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
