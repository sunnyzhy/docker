# docker-elasticsearch 集群 (host)

## 前言

- elasticsearch 版本: ```elasticsearch:7.12.1```

- 网络配置: 驱动类型为 host

- 容器与宿主机映射
    |容器名称|宿主机IP|宿主机端口|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|
    |elasticsearch-1|192.168.204.107|9200<br />9300|/usr/local/docker/elasticsearch/node-1/config:/usr/share/elasticsearch/config<br />/usr/local/docker/elasticsearch/elasticsearch-1/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/elasticsearch-1/logs:/usr/share/elasticsearch/logs|
    |elasticsearch-2|192.168.204.107|9201<br />9301|/usr/local/docker/elasticsearch/node-2/config:/usr/share/elasticsearch/config<br />/usr/local/docker/elasticsearch/elasticsearch-2/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/elasticsearch-2/logs:/usr/share/elasticsearch/logs|
    |elasticsearch-3|192.168.204.107|9202<br />9302|/usr/local/docker/elasticsearch/node-3/config:/usr/share/elasticsearch/config<br />/usr/local/docker/elasticsearch/elasticsearch-3/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/elasticsearch-3/logs:/usr/share/elasticsearch/logs|

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

## 在宿主机上创建 elasticsearch 目录

```bash
# docker run -d --name elasticsearch --net elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.12.1

# mkdir -p /usr/local/docker/elasticsearch

# > /usr/local/docker/elasticsearch/create-node.sh

# vim /usr/local/docker/elasticsearch/create-node.sh
#!/bin/sh
num=0
for index in $(seq 1 3);
do
mkdir -p /usr/local/docker/elasticsearch/node-${index}
docker cp elasticsearch:/usr/share/elasticsearch/config/ /usr/local/docker/elasticsearch/node-${index}/
docker cp elasticsearch:/usr/share/elasticsearch/data/ /usr/local/docker/elasticsearch/node-${index}/
docker cp elasticsearch:/usr/share/elasticsearch/logs/ /usr/local/docker/elasticsearch/node-${index}/
> /usr/local/docker/elasticsearch/node-${index}/config/elasticsearch.yml
cat << EOF >> /usr/local/docker/elasticsearch/node-${index}/config/elasticsearch.yml
cluster.name: "docker-cluster"
network.host: 0.0.0.0
network.publish_host: 192.168.204.107
http.port: 920$(expr ${index} - 1)
transport.tcp.port: 930$(expr ${index} - 1)
node.name: elasticsearch-${index}
discovery.seed_hosts: ["192.168.204.107:930${num}", "192.168.204.107:930$(expr ${num} + 1)", "192.168.204.107:930$(expr ${num} + 2)"]
cluster.initial_master_nodes: ["elasticsearch-1"]
http.cors.enabled: true
http.cors.allow-origin: "*"
EOF
done

# chmod +x /usr/local/docker/elasticsearch/create-node.sh

# /usr/local/docker/elasticsearch/create-node.sh

# ls /usr/local/docker/elasticsearch
create-node.sh  node-1  node-2  node-3

# cat /usr/local/docker/elasticsearch/node-1/config/elasticsearch.yml
cluster.name: "docker-cluster"
network.host: 0.0.0.0
network.publish_host: 192.168.204.107
http.port: 9200
transport.tcp.port: 9300
node.name: elasticsearch-1
discovery.seed_hosts: ["192.168.204.107:9300", "192.168.204.107:9301", "192.168.204.107:9302"]
cluster.initial_master_nodes: ["elasticsearch-1"]
http.cors.enabled: true
http.cors.allow-origin: "*"
```

## 配置 docker-compose.yml

```bash
# > /usr/local/docker/elasticsearch/create-docker-compose.sh

# vim /usr/local/docker/elasticsearch/create-docker-compose.sh
#!/bin/sh
> /usr/local/docker/elasticsearch/docker-compose.yml
cat << EOF >> /usr/local/docker/elasticsearch/docker-compose.yml
version: '3.9'

services:
EOF
for index in $(seq 1 3);
do
cat << EOF >> /usr/local/docker/elasticsearch/docker-compose.yml
 elasticsearch-${index}:
  image: elasticsearch:7.12.1
  container_name: elasticsearch-${index}
  restart: always
  volumes:
   - /usr/local/docker/elasticsearch/node-${index}/config:/usr/share/elasticsearch/config
   - /usr/local/docker/elasticsearch/node-${index}/data:/usr/share/elasticsearch/data
   - /usr/local/docker/elasticsearch/node-${index}/logs:/usr/share/elasticsearch/logs
  network_mode: host
EOF
done

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
  network_mode: host
 elasticsearch-2:
  image: elasticsearch:7.12.1
  container_name: elasticsearch-2
  restart: always
  volumes:
   - /usr/local/docker/elasticsearch/node-2/config:/usr/share/elasticsearch/config
   - /usr/local/docker/elasticsearch/node-2/data:/usr/share/elasticsearch/data
   - /usr/local/docker/elasticsearch/node-2/logs:/usr/share/elasticsearch/logs
  network_mode: host
 elasticsearch-3:
  image: elasticsearch:7.12.1
  container_name: elasticsearch-3
  restart: always
  volumes:
   - /usr/local/docker/elasticsearch/node-3/config:/usr/share/elasticsearch/config
   - /usr/local/docker/elasticsearch/node-3/data:/usr/share/elasticsearch/data
   - /usr/local/docker/elasticsearch/node-3/logs:/usr/share/elasticsearch/logs
  network_mode: host
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

## 访问 elasticsearch

```bash
# curl -XGET http://192.168.204.107:9200/_cat/nodes?v
ip              heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
192.168.204.107           23          93  13    6.47    7.07     9.17 cdfhilmrstw *      elasticsearch-1
192.168.204.107           30          94  11    5.43    2.42     4.47 cdfhilmrstw -      elasticsearch-2
192.168.204.107           50          93   9    2.39    2.62     4.13 cdfhilmrstw -      elasticsearch-3
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
