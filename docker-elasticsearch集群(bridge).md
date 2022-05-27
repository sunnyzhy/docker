# docker-elasticsearch 集群 (bridge)

## 前言

- elasticsearch 版本: ```elasticsearch:7.12.1```

- 网络配置: 驱动类型为 bridge，名称为 elasticsearch ，子网掩码为 ```192.168.3.0/24```，网关为 ```192.168.3.1```

- 容器与宿主机映射
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |elasticsearch|192.168.3.11|9200:9200<br />9300:9300|192.168.204.107|/usr/local/docker/elasticsearch/node-1/config:/usr/share/elasticsearch/config<br />/usr/local/docker/elasticsearch/elasticsearch-1/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/elasticsearch-1/logs:/usr/share/elasticsearch/logs|
    |elasticsearch|192.168.3.12|9201:9200<br />9301:9300|192.168.204.107|/usr/local/docker/elasticsearch/node-2/config:/usr/share/elasticsearch/config<br />/usr/local/docker/elasticsearch/elasticsearch-2/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/elasticsearch-2/logs:/usr/share/elasticsearch/logs|
    |elasticsearch|192.168.3.13|9202:9200<br />9302:9300|192.168.204.107|/usr/local/docker/elasticsearch/node-3/config:/usr/share/elasticsearch/config<br />/usr/local/docker/elasticsearch/elasticsearch-3/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/elasticsearch-3/logs:/usr/share/elasticsearch/logs|

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
# docker network create elasticsearch --subnet 192.168.3.0/24 --gateway=192.168.3.1

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
                    "Subnet": "192.168.3.0/24",
                    "Gateway": "192.168.3.1"
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
# docker run -d --name elasticsearch --net elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.12.1

# mkdir -p /usr/local/docker/elasticsearch

# > /usr/local/docker/elasticsearch/create-node.sh

# vim /usr/local/docker/elasticsearch/create-node.sh
num=0
for index in $(seq 1 3);
do
mkdir -p /usr/local/docker/elasticsearch/node-${index}
> /usr/local/docker/elasticsearch/node-${index}/config/elasticsearch.yml
docker cp elasticsearch:/usr/share/elasticsearch/config/ /usr/local/docker/elasticsearch/node-${index}/
docker cp elasticsearch:/usr/share/elasticsearch/data/ /usr/local/docker/elasticsearch/node-${index}/
docker cp elasticsearch:/usr/share/elasticsearch/logs/ /usr/local/docker/elasticsearch/node-${index}/
echo "http.cors.enabled: true" >> /usr/local/docker/elasticsearch/node-${index}/config/elasticsearch.yml
echo "http.cors.allow-origin: \"*\"" >> /usr/local/docker/elasticsearch/node-${index}/config/elasticsearch.yml
echo "node.name: elasticsearch-"${index} >> /usr/local/docker/elasticsearch/node-${index}/config/elasticsearch.yml
echo "discovery.seed_hosts: [\"192.168.3.11\", \"192.168.3.12\", \"192.168.3.13\"]" >> /usr/local/docker/elasticsearch/node-${index}/config/elasticsearch.yml
echo "cluster.initial_master_nodes: [\"elasticsearch-1\"]" >> /usr/local/docker/elasticsearch/node-${index}/config/elasticsearch.yml
done

# chmod +x /usr/local/docker/elasticsearch/create-node.sh

# /usr/local/docker/elasticsearch/create-node.sh

# ls /usr/local/docker/elasticsearch
create-node.sh  node-1  node-2  node-3

# cat /usr/local/docker/elasticsearch/node-1/config/elasticsearch.yml
cluster.name: "docker-cluster"
network.host: 0.0.0.0
node.master: true
node.data: false
http.cors.enabled: true
http.cors.allow-origin: "*"
node.name: elasticsearch-1
discovery.seed_hosts: ["192.168.3.11", "192.168.3.12", "192.168.3.13"]
cluster.initial_master_nodes: ["elasticsearch-1"]
```

## 配置 docker-compose.yml

```bash
# > /usr/local/docker/elasticsearch/create-docker-compose.sh

# vim /usr/local/docker/elasticsearch/create-docker-compose.sh
> /usr/local/docker/elasticsearch/docker-compose.yml
touch /usr/local/docker/elasticsearch/docker-compose.yml
cd /usr/local/docker/elasticsearch
echo "version: '3.9'" >> docker-compose.yml
echo >> docker-compose.yml
echo "services:" >> docker-compose.yml
for index in $(seq 1 3);
do
echo " elasticsearch-"${index}":" >> docker-compose.yml
echo "  image: elasticsearch:7.12.1" >> docker-compose.yml
echo "  container_name: elasticsearch-"${index} >> docker-compose.yml
echo "  restart: always" >> docker-compose.yml
echo "  volumes:" >> docker-compose.yml
echo "   - /usr/local/docker/elasticsearch/node-${index}/config:/usr/share/elasticsearch/config" >> docker-compose.yml
echo "   - /usr/local/docker/elasticsearch/node-${index}/data:/usr/share/elasticsearch/data" >> docker-compose.yml
echo "   - /usr/local/docker/elasticsearch/node-${index}/logs:/usr/share/elasticsearch/logs" >> docker-compose.yml
echo "  ports:" >> docker-compose.yml
echo "   - 920"$(expr ${index} - 1)":9200" >> docker-compose.yml
echo "   - 930"$(expr ${index} - 1)":9300" >> docker-compose.yml
echo "  networks:" >> docker-compose.yml
echo "    elasticsearch:" >> docker-compose.yml
echo "      ipv4_address: 192.168.3.1"${index} >> docker-compose.yml
done
echo "networks:" >> docker-compose.yml
echo "    elasticsearch:" >> docker-compose.yml
echo "      name: elasticsearch" >> docker-compose.yml

# chmod +x /usr/local/docker/elasticsearch/create-docker-compose.sh

# /usr/local/docker/elasticsearch/create-docker-compose.sh

# ls /usr/local/docker/elasticsearch
create-docker-compose.sh  create-node.sh  docker-compose.yml  node-1  node-2  node-3
```

***完整的 docker-compose.yml:***

```yml
version: '3.9'

services:
 elasticsearch-1:
  image: elasticsearch:7.12.1
  container_name: elasticsearch-1
  restart: always
  volumes:
   - /usr/local/docker/elasticsearch/node-1/config:/usr/share/elasticsearch/config
   - /usr/local/docker/elasticsearch/node-1/data:/usr/share/elasticsearch/data
   - /usr/local/docker/elasticsearch/node-1/logs:/usr/share/elasticsearch/logs
  ports:
   - 9200:9200
   - 9300:9300
  networks:
    elasticsearch:
      ipv4_address: 192.168.3.11
 elasticsearch-2:
  image: elasticsearch:7.12.1
  container_name: elasticsearch-2
  restart: always
  volumes:
   - /usr/local/docker/elasticsearch/node-2/config:/usr/share/elasticsearch/config
   - /usr/local/docker/elasticsearch/node-2/data:/usr/share/elasticsearch/data
   - /usr/local/docker/elasticsearch/node-2/logs:/usr/share/elasticsearch/logs
  ports:
   - 9201:9200
   - 9301:9300
  networks:
    elasticsearch:
      ipv4_address: 192.168.3.12
 elasticsearch-3:
  image: elasticsearch:7.12.1
  container_name: elasticsearch-3
  restart: always
  volumes:
   - /usr/local/docker/elasticsearch/node-3/config:/usr/share/elasticsearch/config
   - /usr/local/docker/elasticsearch/node-3/data:/usr/share/elasticsearch/data
   - /usr/local/docker/elasticsearch/node-3/logs:/usr/share/elasticsearch/logs
  ports:
   - 9202:9200
   - 9302:9300
  networks:
    elasticsearch:
      ipv4_address: 192.168.3.13
networks:
    elasticsearch:
      name: elasticsearch
```

## 启动 docker-compose

```bash
# docker stop elasticsearch

# docker rm elasticsearch

# docker-compose -f /usr/local/docker/elasticsearch/docker-compose.yml up -d
[+] Running 3/3
 ⠿ Container elasticsearch-3  Started                                                                            4.6s
 ⠿ Container elasticsearch-1  Started                                                                            4.5s
 ⠿ Container elasticsearch-2  Started                                                                            4.5s

# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED              STATUS              PORTS                                                                                            NAMES
260cbfb15167   elasticsearch:7.12.1     "/bin/tini -- /usr/l…"   About a minute ago   Up About a minute   0.0.0.0:9202->9200/tcp, :::9202->9200/tcp, 0.0.0.0:9302->9300/tcp, :::9302->9300/tcp             elasticsearch-3
cd18f1196240   elasticsearch:7.12.1     "/bin/tini -- /usr/l…"   About a minute ago   Up About a minute   0.0.0.0:9201->9200/tcp, :::9201->9200/tcp, 0.0.0.0:9301->9300/tcp, :::9301->9300/tcp             elasticsearch-2
8432bad0063f   elasticsearch:7.12.1     "/bin/tini -- /usr/l…"   About a minute ago   Up About a minute   0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 0.0.0.0:9300->9300/tcp, :::9300->9300/tcp             elasticsearch-1
```

## 查看网络

```bash
# docker network inspect elasticsearch
[
    {
        "Name": "elasticsearch",
        "Id": "7957a3ac27bef2898e1b1a00faece2cc9ddf629dc3986a7dcbaf76c1e80d6a4c",
        "Created": "2022-05-26T01:51:47.592775964-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.3.0/24",
                    "Gateway": "192.168.3.1"
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
            "319faab1463b9139e5a642e055f4d7cd5f9323b8d3b75a161d1dfc569cc07613": {
                "Name": "elasticsearch-2",
                "EndpointID": "8a28f0a7e09dae27ef11f2468703d6f14379a5102735eb5fbaf0aa45c51dfc16",
                "MacAddress": "02:42:c0:a8:03:0c",
                "IPv4Address": "192.168.3.12/24",
                "IPv6Address": ""
            },
            "48ec0ffec6766b93903a058445ee8f843d82b3a52d4cbab74f47695fcfb1041f": {
                "Name": "elasticsearch-1",
                "EndpointID": "5ad3371274ab6990bd56b819f75447bbf79c84b81f1d045289674a8eddf7a815",
                "MacAddress": "02:42:c0:a8:03:0b",
                "IPv4Address": "192.168.3.11/24",
                "IPv6Address": ""
            },
            "93b9e437df51e22ebf2bcfcd806e21fb75fcb945a15a956cdc4103c6f3a8b461": {
                "Name": "elasticsearch-3",
                "EndpointID": "edb22fb81cb06f353e2b34aca54313d80fb28eccd4b580be891ad49a4294c148",
                "MacAddress": "02:42:c0:a8:03:0d",
                "IPv4Address": "192.168.3.13/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

## 访问 elasticsearch

```bash
# curl -XGET http://192.168.3.11:9200/_cat/nodes?v
ip           heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
192.168.3.11           23          93  13    6.47    7.07     9.17 cdfhilmrstw *      elasticsearch-1
192.168.3.12           30          94  11    5.43    2.42     4.47 cdfhilmrstw -      elasticsearch-2
192.168.3.13           50          93   9    2.39    2.62     4.13 cdfhilmrstw -      elasticsearch-3
```

## FAQ

### max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

- 原因
   系统虚拟内存默认最大映射数为 65530，无法满足 ES 要求，需要把系统虚拟内存调整为 262144 以上。

- 解决办法
```bash
# vim /etc/sysctl.conf
vm.max_map_count = 262144

# sysctl -p
```
