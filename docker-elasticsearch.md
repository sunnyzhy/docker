# docker-elasticsearch

## 前言

- elasticsearch 版本: ```elasticsearch:7.12.1```

- 网络配置: 驱动类型为 bridge，名称为 elasticsearch

- 容器与宿主机映射
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |elasticsearch|-|9200:9200<br />9300:9300|192.168.204.107|/usr/local/docker/elasticsearch/config:/usr/share/elasticsearch/config<br />/usr/local/docker/elasticsearch/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/logs:/usr/share/elasticsearch/logs|

- elasticsearch:7.12.1 镜像配置文件 elasticsearch.yml
    ```yml
    cluster.name: "docker-cluster"
    network.host: 0.0.0.0
    ```

## 拉取 elasticsearch 镜像

```bash
# docker pull elasticsearch:7.12.1

# docker images
REPOSITORY               TAG       IMAGE ID       CREATED         SIZE
elasticsearch            7.12.1    41dc8ea0f139   13 months ago   851MB
```

## 创建网络

```bash
# docker network create elasticsearch

# docker network ls
NETWORK ID     NAME            DRIVER    SCOPE
b0308476ccc3   bridge          bridge    local
02377983e431   elasticsearch   bridge    local
3724599c2452   host            host      local
d61017ea048b   mysql           bridge    local
cad7d065639d   none            null      local

# docker network inspect elasticsearch
[
    {
        "Name": "elasticsearch",
        "Id": "02377983e4310a6e2310a07d71b41a2067ab81fa7f6cfbbf7405512fe4b619d2",
        "Created": "2022-05-24T22:45:11.299851491-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.20.0.0/16",
                    "Gateway": "172.20.0.1"
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

## 在宿主机上创建 elasticsearch 目录

```bash
# mkdir -p /usr/local/docker/elasticsearch
```

## 把容器中的 elasticsearch 相关目录复制到宿主机的 elasticsearch 目录

```bash
# docker run -d --name elasticsearch --net elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.12.1

# docker cp elasticsearch:/usr/share/elasticsearch/config/ /usr/local/docker/elasticsearch/

# docker cp elasticsearch:/usr/share/elasticsearch/data/ /usr/local/docker/elasticsearch/

# docker cp elasticsearch:/usr/share/elasticsearch/logs/ /usr/local/docker/elasticsearch/

# ls /usr/local/docker/elasticsearch
config  data  logs
```

## 配置 docker-compose.yml

```bash
# vim /usr/local/docker/elasticsearch/docker-compose.yml
version: '3.9'

services:
 elasticsearch:
  image: elasticsearch:7.12.1
  container_name: elasticsearch
  restart: always
  volumes:
   - /usr/local/docker/elasticsearch/config:/usr/share/elasticsearch/config
   - /usr/local/docker/elasticsearch/data:/usr/share/elasticsearch/data
   - /usr/local/docker/elasticsearch/data:/usr/share/elasticsearch/logs
  ports:
   - 9200:9200
   - 9300:9300
  environment:
   discovery.type: single-node
  networks:
   - elasticsearch
networks:
  elasticsearch:
    name: elasticsearch
```

## 启动 docker-compose

```bash
# docker stop elasticsearch

# docker rm elasticsearch

# docker-compose -f /usr/local/docker/elasticsearch/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container elasticsearch  Started                                                                              1.2s

# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                                                            NAMES
a276bad7e374   elasticsearch:7.12.1     "/bin/tini -- /usr/l…"   51 seconds ago   Up 49 seconds   0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 0.0.0.0:9300->9300/tcp, :::9300->9300/tcp             elasticsearch                                                       nginx
```

## 访问 elasticsearch

```bash
# curl -XGET http://192.168.204.107:9200
{
  "name" : "a276bad7e374",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "cDCZCNL5Q020C1-dqxeOvA",
  "version" : {
    "number" : "7.12.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "3186837139b9c6b6d23c3200870651f10d3343b7",
    "build_date" : "2021-04-20T20:56:39.040728659Z",
    "build_snapshot" : false,
    "lucene_version" : "8.8.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
