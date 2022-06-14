# docker-influxdb

## 前言

- influxdb 版本: ```influxdb:1.8.1```

- 网络配置: 驱动类型为 bridge，名称为 influxdb

- 容器与宿主机映射
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |influxdb|192.168.6.0|8083:8083<br />8086:8086|192.168.204.107|/usr/local/docker/influxdb/conf/influxdb.conf:/etc/influxdb/influxdb.conf<br />/usr/local/docker/influxdb/data:\/var/lib/influxdb|

- influxdb:1.8.1 镜像配置文件 influxdb.conf
    ```conf
    [meta]
      dir = "/var/lib/influxdb/meta"

    [data]
      dir = "/var/lib/influxdb/data"
      engine = "tsm1"
      wal-dir = "/var/lib/influxdb/wal"
    ```

## 拉取 influxdb 镜像

```bash
# docker pull influxdb:1.8.10

# docker images
REPOSITORY                          TAG       IMAGE ID       CREATED         SIZE
influxdb                            1.8.10    e20d1d5e2cbd   5 months ago    287MB
```

## 创建网络

```bash
# docker network create influxdb --subnet 192.168.6.0/24 --gateway=192.168.6.1

# docker network ls
NETWORK ID     NAME                    DRIVER    SCOPE
dc695bb337ba   influxdb                bridge    local

# docker network inspect influxdb
[
    {
        "Name": "influxdb",
        "Id": "feceb7453dc7bbdf17faf7e457508c0b7b941f021797e2b443679a7d011bd2f4",
        "Created": "2022-06-13T02:12:42.393745775-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.6.0/24",
                    "Gateway": "192.168.6.1"
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

## 在宿主机上创建 influxdb 目录

```bash
# docker run --name influxdb --net influxdb -p 8083:8083 -p 8086:8086 -d influxdb:1.8.10

# mkdir -p /usr/local/docker/influxdb

# > /usr/local/docker/influxdb/create-node.sh

# vim /usr/local/docker/influxdb/create-node.sh
```

```sh
#!/bin/sh
mkdir -p /usr/local/docker/influxdb/{conf,data}

##--------------------------------------------------------------------
## 拷贝容器目录到宿主机
##--------------------------------------------------------------------
docker cp influxdb:/etc/influxdb/influxdb.conf /usr/local/docker/influxdb/conf
docker cp influxdb:/var/lib/influxdb/data /usr/local/docker/influxdb/data
docker cp influxdb:/var/lib/influxdb/meta /usr/local/docker/influxdb/data
docker cp influxdb:/var/lib/influxdb/wal /usr/local/docker/influxdb/data

##--------------------------------------------------------------------
## 数据目录授权
##--------------------------------------------------------------------
chmod -R 777 /usr/local/docker/influxdb/data
```

```bash
# chmod +x /usr/local/docker/influxdb/create-node.sh

# /usr/local/docker/influxdb/create-node.sh

# ls /usr/local/docker/influxdb
conf  create-node.sh  data
```

## 配置 docker-compose.yml

```bash
# echo 'Asia/Shanghai' > /etc/timezone/timezone

# vim /usr/local/docker/influxdb/docker-compose.yml
```

```yml
version: '3.9'

services:
 influxdb:
  image: influxdb:1.8.10
  container_name: influxdb
  restart: always
  volumes:
   - /usr/local/docker/influxdb/conf/influxdb.conf:/etc/influxdb/influxdb.conf
   - /usr/local/docker/influxdb/data:/var/lib/influxdb
   - /etc/timezone/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  ports:
   - 8083:8083
   - 8086:8086
  networks:
   influxdb:
     ipv4_address: 192.168.6.10
networks:
  influxdb:
    name: influxdb
```

## 启动 docker-compose

```bash
# docker stop influxdb

# docker rm influxdb

# docker-compose -f /usr/local/docker/influxdb/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container influxdb  Started                                                                                   0.7s

# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                                                            NAMES
b4bc84f5acbc   influxdb:1.8.10          "/entrypoint.sh infl…"   27 seconds ago   Up 25 seconds   0.0.0.0:8083->8083/tcp, :::8083->8083/tcp, 0.0.0.0:8086->8086/tcp, :::8086->8086/tcp             influxdb                                                            nginx
```

## 查看网络

```bash
# docker network inspect influxdb
[
    {
        "Name": "influxdb",
        "Id": "feceb7453dc7bbdf17faf7e457508c0b7b941f021797e2b443679a7d011bd2f4",
        "Created": "2022-06-13T02:12:42.393745775-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.6.0/24",
                    "Gateway": "192.168.6.1"
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
            "f85c34e2139713830d755e2e6b08650db360ee78e68417d476da3d6df586b29c": {
                "Name": "influxdb",
                "EndpointID": "d65a64b87d3ed228372798795afff026c2da17c0e0ca00f80b861039b2269bf0",
                "MacAddress": "02:42:c0:a8:06:0a",
                "IPv4Address": "192.168.6.10/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

## 执行 influxdb 客户端命令行

```bash
# docker exec -it influxdb /bin/sh

# cd /usr/bin

# ./influx
Connected to http://localhost:8086 version 1.8.10
InfluxDB shell version: 1.8.10
> 
```
