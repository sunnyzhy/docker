# docker-influxdb2.x

## 前言

- influxdb 版本: ```influxdb:latest```

- 网络配置: 驱动类型为 bridge，名称为 influxdb

- 容器与宿主机映射
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |influxdb|192.168.6.0|8086:8086|192.168.204.107|/usr/local/docker/influxdb/conf/influxdb.conf:/etc/influxdb/influxdb.conf<br />/usr/local/docker/influxdb/data:\/var/lib/influxdb|

- influxdb:latest 镜像配置文件 influxdb.conf
    ```conf
    [meta]
      dir = "/var/lib/influxdb/meta"

    [data]
      dir = "/var/lib/influxdb/data"
      engine = "tsm1"
      wal-dir = "/var/lib/influxdb/wal"
    ```

- 单节点启动:
    ```
    # docker run -d --name influxdb2 \
          --net influxdb \
          -p 8086:8086 \
          -e DOCKER_INFLUXDB_INIT_MODE=setup \
          -e DOCKER_INFLUXDB_INIT_USERNAME=admin \
          -e DOCKER_INFLUXDB_INIT_PASSWORD=influxdb \
          -e DOCKER_INFLUXDB_INIT_ORG=zhy \
          -e DOCKER_INFLUXDB_INIT_BUCKET=zhy \
          influxdb:latest
    ```

## 拉取 influxdb 镜像

```bash
# docker pull influxdb:latest

# docker images
REPOSITORY                                      TAG       IMAGE ID       CREATED         SIZE
influxdb                                        latest    1f5a04237d8e   5 months ago    346MB
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

## 创建 Docker 数据卷

```bash
# docker volume create influxdb2

# docker volume inspect influxdb2
[
    {
        "CreatedAt": "2022-06-16T03:02:20-04:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/influxdb2/_data",
        "Name": "influxdb2",
        "Options": {},
        "Scope": "local"
    }
]
```

## 在宿主机上创建 influxdb 目录

```bash
# mkdir -p /usr/local/docker/influxdb2
```

## 配置 docker-compose.yml

[influxdb2](https://docs.influxdata.com/influxdb/v2.0/upgrade/v1-to-v2/docker/ 'influxdb2')

```bash
# echo 'Asia/Shanghai' > /etc/timezone/timezone

# vim /usr/local/docker/influxdb2/docker-compose.yml
```

```yml
version: '3.9'

services:
 influxdb2:
  image: influxdb:latest
  container_name: influxdb2
  restart: always
  volumes:
   - influxdb2:/var/lib/influxdb2
   - /etc/timezone/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  ports:
   - 8086:8086
  environment:
    - DOCKER_INFLUXDB_INIT_MODE=setup
    - DOCKER_INFLUXDB_INIT_USERNAME=admin
    - DOCKER_INFLUXDB_INIT_PASSWORD=influxdb
    - DOCKER_INFLUXDB_INIT_ORG=zhy
    - DOCKER_INFLUXDB_INIT_BUCKET=zhy
  networks:
   influxdb:
     ipv4_address: 192.168.6.10

volumes:
  influxdb2:
    name: influxdb2
    external: true

networks:
  influxdb:
    name: influxdb
```

## 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/influxdb2/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container influxdb2  Started                                                                                  0.6s

# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                                                            NAMES
d2dc44746fa3   influxdb:latest          "/entrypoint.sh infl…"   19 seconds ago   Up 19 seconds   0.0.0.0:8086->8086/tcp, :::8086->8086/tcp                                                        influxdb2
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
            "d2dc44746fa3db8761cebc8acc2cee5878747d606550c43ad4a90d2ee49de63b": {
                "Name": "influxdb2",
                "EndpointID": "8f8eacf7f331407d10ca968fb9c6900f08cc5dc730dd37e8815ab585e1f2bcfc",
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

## 执行 influxdb2 客户端命令行

```bash
# docker exec -it -u root influxdb2 /bin/sh

# influx version
Influx CLI 2.2.1 (git: 31ac783) build_date: 2021-11-09T21:24:22Z

# influx bucket list -o "zhy"
ID			Name		Retention	Shard group duration	Organization ID		Schema Type
1ae215f264372e5d	_monitoring	168h0m0s	24h0m0s			f884e97bd91f961b	implicit
dd411414f11e7814	_tasks		72h0m0s		24h0m0s			f884e97bd91f961b	implicit
0dc2234075563b9d	zhy		infinite	168h0m0s		f884e97bd91f961b	implicit
```

## 浏览器访问 influxdb2

访问 ```https://$HOST_IP:8086```, 用户名/密码: ```admin/influxdb```, 如:

```
http://192.168.204.107:8086
```

***注: API Tokens 包含当前用户的 Token，与后端整合时需要用到。**
