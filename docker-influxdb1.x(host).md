# docker - influxdb1.x(host)

## 前言

- influxdb 版本: ```influxdb:1.8.1```

- 网络配置: 驱动类型为 overlay

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- 容器与宿主机映射
    |容器名称|宿主机IP|端口映射(宿主机端口:容器端口)|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|
    |influxdb|192.168.5.163|8086:8086|/usr/local/docker/influxdb/conf/influxdb.conf:/etc/influxdb/influxdb.conf<br />/usr/local/docker/influxdb/data:\/var/lib/influxdb|

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

# docker images | influxdb
REPOSITORY                          TAG       IMAGE ID       CREATED         SIZE
influxdb                            1.8.10    e20d1d5e2cbd   5 months ago    287MB
```

## 在宿主机上创建 influxdb 目录

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```sh
#!/bin/sh

mkdir -p /usr/local/docker/influxdb/{conf,data}
if [ ! -f /usr/local/docker/influxdb/conf/influxdb.conf ]; then
    docker network create influxdb --subnet 192.168.6.0/24 --gateway=192.168.6.1
    docker run --name influxdb --net influxdb -p 8086:8086 -d influxdb:1.8.10
    sleep 15s

    ##--------------------------------------------------------------------
    ## 拷贝容器目录到宿主机
    ##--------------------------------------------------------------------
    docker cp influxdb:/etc/influxdb/influxdb.conf /usr/local/docker/influxdb/conf
    docker cp influxdb:/var/lib/influxdb/data /usr/local/docker/influxdb/data
    docker cp influxdb:/var/lib/influxdb/meta /usr/local/docker/influxdb/data
	
    docker stop influxdb
    docker rm influxdb
    docker network rm influxdb
fi

##--------------------------------------------------------------------
## 数据目录授权
##--------------------------------------------------------------------
chmod -R 777 /usr/local/docker/influxdb/data
```

```bash
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/influxdb/
/usr/local/docker/influxdb/
├── conf
│   └── influxdb.conf
└── data
    ├── data
    │   └── _internal
    │       ├── monitor
    │       │   └── 1
    │       │       └── fields.idx
    │       └── _series
    │           ├── 00
    │           │   └── 0000
    │           ├── 01
    │           │   └── 0000
    │           ├── 02
    │           │   └── 0000
    │           ├── 03
    │           │   └── 0000
    │           ├── 04
    │           │   └── 0000
    │           ├── 05
    │           │   └── 0000
    │           ├── 06
    │           │   └── 0000
    │           └── 07
    │               └── 0000
    └── meta
        └── meta.db
```

复制 192.168.5.163(manager) 的 influxdb 目录到 192.168.5.165(worker):

```bash
# scp -r /usr/local/docker/influxdb root@192.168.5.165:/usr/local/docker
```

## 配置 docker-compose.yml

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# vim /usr/local/docker/docker-compose.yml
```

```yml
version: '3.9'

services:
 influxdb:
   image: influxdb:1.8.10
   volumes:
     - /usr/local/docker/influxdb/conf/influxdb.conf:/etc/influxdb/influxdb.conf
     - /usr/local/docker/influxdb/data:/var/lib/influxdb
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   ports:
     - 8086:8086
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-165
         - node.role == worker
   networks:
     - iot

networks:
  iot:
    driver: overlay
    external: true
```

## 启动 docker-compose

在宿主机 192.168.5.163(manager) 上启动 influxdb 服务:

```bash
# docker stack deploy -c /usr/local/docker/docker-compose.yml iot
Creating service iot_influxdb

# docker stack ps iot | grep influxdb
z42vl8aafdpm   iot_influxdb.1           influxdb:1.8.10                                        centos-docker-165   Running         Running 31 seconds ago                                          

# docker service ls | grep influxdb
vhuisdngnj4p   iot_influxdb          replicated   1/1        influxdb:1.8.10                                        *:8086->8086/tcp
```

## 查看 influxdb 容器

查看宿主机 192.168.5.165 的 influxdb 容器:

```bash
# docker ps | grep influxdb
31271823d152   influxdb:1.8.10                                        "/entrypoint.sh infl…"   About a minute ago   Up About a minute   8086/tcp                               iot_influxdb.1.z42vl8aafdpms8sxv27v7bonj
```

## 执行 influxdb 客户端命令行

在宿主机 192.168.5.165 上执行以下命令:

```bash
# docker exec -it 31271823d152 /bin/sh

# cd /usr/bin

# ./influx
Connected to http://localhost:8086 version 1.8.10
InfluxDB shell version: 1.8.10
> 
```
