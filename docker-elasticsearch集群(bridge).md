# docker-elasticsearch 集群 (bridge)

## 前言

- elasticsearch 版本: ```docker.elastic.co/elasticsearch/elasticsearch:7.12.1```

- 网络配置: 驱动类型为 bridge，名称为 elasticsearch ，子网掩码为 ```192.168.3.0/24```，网关为 ```192.168.3.1```

- 容器与宿主机映射
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |elasticsearch-1|192.168.3.11|9200:9200<br />9300:9300|192.168.204.107|/usr/local/docker/elasticsearch/node-1/config:/usr/share/elasticsearch/config<br />/usr/local/docker/elasticsearch/elasticsearch-1/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/elasticsearch-1/logs:/usr/share/elasticsearch/logs|
    |elasticsearch-2|192.168.3.12|9201:9200<br />9301:9300|192.168.204.107|/usr/local/docker/elasticsearch/node-2/config:/usr/share/elasticsearch/config<br />/usr/local/docker/elasticsearch/elasticsearch-2/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/elasticsearch-2/logs:/usr/share/elasticsearch/logs|
    |elasticsearch-3|192.168.3.13|9202:9200<br />9302:9300|192.168.204.107|/usr/local/docker/elasticsearch/node-3/config:/usr/share/elasticsearch/config<br />/usr/local/docker/elasticsearch/elasticsearch-3/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/elasticsearch-3/logs:/usr/share/elasticsearch/logs|

- docker.elastic.co/elasticsearch/elasticsearch:7.12.1 镜像配置文件 elasticsearch.yml
    ```yml
    cluster.name: "docker-cluster"
    network.host: 0.0.0.0
    ```

- 单节点启动:
    ```bash
    # docker run -d --name elasticsearch --net elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.12.1
    ```

## 拉取 elasticsearch 镜像

```bash
# docker pull docker.elastic.co/elasticsearch/elasticsearch:7.12.1

# docker images
REPOSITORY                                      TAG       IMAGE ID       CREATED         SIZE
docker.elastic.co/elasticsearch/elasticsearch   7.12.1    41dc8ea0f139   14 months ago   851MB
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

## 创建 Docker 数据卷

```bash
# docker volume create certs

# docker volume create esdata01

# docker volume create esdata02

# docker volume create esdata03

# docker volume inspect certs
[
    {
        "CreatedAt": "2022-06-15T20:47:58-04:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/certs/_data",
        "Name": "certs",
        "Options": {},
        "Scope": "local"
    }
]
```

## 在宿主机上创建 elasticsearch 目录

```bash
# mkdir -p /usr/local/docker/elasticsearch
```

## 配置 docker-compose.yml

### 配置 .env

```bash
# > /usr/local/docker/elasticsearch/.env

# vim /usr/local/docker/elasticsearch/.env
```

```env
# Password for the 'elastic' user (at least 6 characters)
ELASTIC_PASSWORD=elastic

# Version of Elastic products
STACK_VERSION=7.12.1

# Set the cluster name
CLUSTER_NAME=docker-cluster

# Set to 'basic' or 'trial' to automatically start the 30-day trial
LICENSE=basic
#LICENSE=trial

# Port to expose Elasticsearch HTTP API to the host
ES_PORT=9200
#ES_PORT=127.0.0.1:9200

# Increase or decrease based on the available host memory (in bytes)
MEM_LIMIT=1073741824
```

### 配置 docker-compose.yml

![elasticsearch 集群](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html?baymax=rec&rogue=pop-1&elektra=guide 'elasticsearch 集群')

```bash
# > /usr/local/docker/elasticsearch/create-docker-compose.sh

# vim /usr/local/docker/elasticsearch/create-docker-compose.sh
```

```sh
#!/bin/sh
> /usr/local/docker/elasticsearch/docker-compose.yml
cat << EOF >> /usr/local/docker/elasticsearch/docker-compose.yml
version: '3.9'

services:
  setup:
    image: docker.elastic.co/elasticsearch/elasticsearch:\${STACK_VERSION}
    container_name: elasticsearch-setup
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
    user: "0"
    command: >
      bash -c '
        if [ x\${ELASTIC_PASSWORD} == x ]; then
          echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
          exit 1;
        fi;
        if [ ! -f config/certs/ca.zip ]; then
          echo "Creating CA";
          bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
          unzip config/certs/ca.zip -d config/certs;
        fi;
        if [ ! -f config/certs/certs.zip ]; then
          echo "Creating certs";
          echo -ne \
          "instances:\n"\
          "  - name: es01\n"\
          "    dns:\n"\
          "      - es01\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 192.168.3.11\n"\
          "  - name: es02\n"\
          "    dns:\n"\
          "      - es02\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 192.168.3.12\n"\
          "  - name: es03\n"\
          "    dns:\n"\
          "      - es03\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 192.168.3.13\n"\
          > config/certs/instances.yml;
          bin/elasticsearch-certutil cert --silent --pem -out config/certs/certs.zip --in config/certs/instances.yml --ca-cert config/certs/ca/ca.crt --ca-key config/certs/ca/ca.key;
          unzip config/certs/certs.zip -d config/certs;
        fi;
        echo "Setting file permissions"
        chown -R root:root config/certs;
        find . -type d -exec chmod 750 \{\} \;;
        find . -type f -exec chmod 640 \{\} \;;
        echo "Waiting for Elasticsearch availability";
        until curl -s --cacert config/certs/ca/ca.crt https://es01:9200 | grep -q "missing authentication credentials"; do sleep 30; done;
        echo "All done!";
      '
    healthcheck:
      test: ["CMD-SHELL", "[ -f config/certs/es01/es01.crt ]"]
      interval: 1s
      timeout: 5s
      retries: 120

EOF
for index in $(seq 1 3);
do
cat << EOF >> /usr/local/docker/elasticsearch/docker-compose.yml
  es0${index}:
    depends_on:
EOF

if [ ${index} == 1 ]; then
cat << EOF >> /usr/local/docker/elasticsearch/docker-compose.yml
      setup:
        condition: service_healthy
EOF
else
echo "      - es0$(expr ${index} - 1)" >> /usr/local/docker/elasticsearch/docker-compose.yml
fi

cat << EOF >> /usr/local/docker/elasticsearch/docker-compose.yml
    image: docker.elastic.co/elasticsearch/elasticsearch:\${STACK_VERSION}
    container_name: elasticsearch-${index}
    restart: always
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - esdata0${index}:/usr/share/elasticsearch/data
    ports:
      - 920$(expr ${index} - 1):9200
      - 930$(expr ${index} - 1):9300
    environment:
      - node.name=es0${index}
      - cluster.name=\${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
EOF

if [ ${index} == 1 ]; then
echo "      - discovery.seed_hosts=es02,es03" >> /usr/local/docker/elasticsearch/docker-compose.yml
fi
if [ ${index} == 2 ]; then
echo "      - discovery.seed_hosts=es01,es03" >> /usr/local/docker/elasticsearch/docker-compose.yml
fi
if [ ${index} == 3 ]; then
echo "      - discovery.seed_hosts=es01,es02" >> /usr/local/docker/elasticsearch/docker-compose.yml
fi

cat << EOF >> /usr/local/docker/elasticsearch/docker-compose.yml
      - ELASTIC_PASSWORD=\${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es0${index}/es0${index}.key
      - xpack.security.http.ssl.certificate=certs/es0${index}/es0${index}.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es0${index}/es0${index}.key
      - xpack.security.transport.ssl.certificate=certs/es0${index}/es0${index}.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=\${LICENSE}
    networks:
      elasticsearch:
        ipv4_address: 192.168.3.1${index}
    mem_limit: \${MEM_LIMIT}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://192.168.3.1${index}:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120
EOF
done

cat << EOF >> /usr/local/docker/elasticsearch/docker-compose.yml

volumes:
  certs:
    name: certs
    external: true
EOF
for index in $(seq 1 3);
do
cat << EOF >> /usr/local/docker/elasticsearch/docker-compose.yml    
  esdata0${index}:
    name: esdata0${index}
    external: true
EOF
done
cat << EOF >> /usr/local/docker/elasticsearch/docker-compose.yml
networks:
    elasticsearch:
      name: elasticsearch
EOF
```

```bash
# chmod +x /usr/local/docker/elasticsearch/create-docker-compose.sh

# /usr/local/docker/elasticsearch/create-docker-compose.sh

# ls /usr/local/docker/elasticsearch
create-docker-compose.sh  create-node.sh  docker-compose.yml  node-1  node-2  node-3
```

***完整的 docker-compose.yml:***

```yml
version: '3.9'

services:
  setup:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: elasticsearch-setup
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
    user: "0"
    command: >
      bash -c '
        if [ x${ELASTIC_PASSWORD} == x ]; then
          echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
          exit 1;
        fi;
        if [ ! -f config/certs/ca.zip ]; then
          echo "Creating CA";
          bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
          unzip config/certs/ca.zip -d config/certs;
        fi;
        if [ ! -f config/certs/certs.zip ]; then
          echo "Creating certs";
          echo -ne           "instances:\n"          "  - name: es01\n"          "    dns:\n"          "      - es01\n"          "      - localhost\n"          "    ip:\n"          "      - 192.168.3.11\n"          "  - name: es02\n"          "    dns:\n"          "      - es02\n"          "      - localhost\n"          "    ip:\n"          "      - 192.168.3.12\n"          "  - name: es03\n"          "    dns:\n"          "      - es03\n"          "      - localhost\n"          "    ip:\n"          "      - 192.168.3.13\n"          > config/certs/instances.yml;
          bin/elasticsearch-certutil cert --silent --pem -out config/certs/certs.zip --in config/certs/instances.yml --ca-cert config/certs/ca/ca.crt --ca-key config/certs/ca/ca.key;
          unzip config/certs/certs.zip -d config/certs;
        fi;
        echo "Setting file permissions"
        chown -R root:root config/certs;
        find . -type d -exec chmod 750 \{\} \;;
        find . -type f -exec chmod 640 \{\} \;;
        echo "Waiting for Elasticsearch availability";
        until curl -s --cacert config/certs/ca/ca.crt https://es01:9200 | grep -q "missing authentication credentials"; do sleep 30; done;
        echo "All done!";
      '
    healthcheck:
      test: ["CMD-SHELL", "[ -f config/certs/es01/es01.crt ]"]
      interval: 1s
      timeout: 5s
      retries: 120

  es01:
    depends_on:
      setup:
        condition: service_healthy
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: elasticsearch-1
    restart: always
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - esdata01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9300:9300
    environment:
      - node.name=es01
      - cluster.name=${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es02,es03
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es01/es01.key
      - xpack.security.http.ssl.certificate=certs/es01/es01.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es01/es01.key
      - xpack.security.transport.ssl.certificate=certs/es01/es01.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=${LICENSE}
    networks:
      elasticsearch:
        ipv4_address: 192.168.3.11
    mem_limit: ${MEM_LIMIT}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://192.168.3.11:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120
  es02:
    depends_on:
      - es01
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: elasticsearch-2
    restart: always
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - esdata02:/usr/share/elasticsearch/data
    ports:
      - 9201:9200
      - 9301:9300
    environment:
      - node.name=es02
      - cluster.name=${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es03
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es02/es02.key
      - xpack.security.http.ssl.certificate=certs/es02/es02.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es02/es02.key
      - xpack.security.transport.ssl.certificate=certs/es02/es02.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=${LICENSE}
    networks:
      elasticsearch:
        ipv4_address: 192.168.3.12
    mem_limit: ${MEM_LIMIT}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://192.168.3.12:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120
  es03:
    depends_on:
      - es02
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: elasticsearch-3
    restart: always
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - esdata03:/usr/share/elasticsearch/data
    ports:
      - 9202:9200
      - 9302:9300
    environment:
      - node.name=es03
      - cluster.name=${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es02
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es03/es03.key
      - xpack.security.http.ssl.certificate=certs/es03/es03.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es03/es03.key
      - xpack.security.transport.ssl.certificate=certs/es03/es03.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=${LICENSE}
    networks:
      elasticsearch:
        ipv4_address: 192.168.3.13
    mem_limit: ${MEM_LIMIT}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://192.168.3.13:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120

volumes:
  certs:
    name: certs
    external: true
  esdata01:
    name: esdata01
    external: true
  esdata02:
    name: esdata02
    external: true
  esdata03:
    name: esdata03
    external: true
networks:
    elasticsearch:
      name: elasticsearch
```

## 启动 docker-compose

```bash
# docker stop elasticsearch

# docker rm elasticsearch

# docker-compose -f /usr/local/docker/elasticsearch/docker-compose.yml up -d
[+] Running 4/4
 ⠿ Container elasticsearch-setup  Healthy                                                                        2.2s
 ⠿ Container elasticsearch-1      Started                                                                        2.8s
 ⠿ Container elasticsearch-2      Started                                                                        4.9s
 ⠿ Container elasticsearch-3      Started                                                                        6.9s

# docker ps
CONTAINER ID   IMAGE                                                  COMMAND                  CREATED              STATUS                        PORTS                                                                                            NAMES
31697f4a024d   docker.elastic.co/elasticsearch/elasticsearch:7.12.1   "/bin/tini -- /usr/l…"   About a minute ago   Up About a minute (healthy)   0.0.0.0:9202->9200/tcp, :::9202->9200/tcp, 0.0.0.0:9302->9300/tcp, :::9302->9300/tcp             elasticsearch-3
6bebe55627b9   docker.elastic.co/elasticsearch/elasticsearch:7.12.1   "/bin/tini -- /usr/l…"   About a minute ago   Up About a minute (healthy)   0.0.0.0:9201->9200/tcp, :::9201->9200/tcp, 0.0.0.0:9301->9300/tcp, :::9301->9300/tcp             elasticsearch-2
a4ece8bd0829   docker.elastic.co/elasticsearch/elasticsearch:7.12.1   "/bin/tini -- /usr/l…"   About a minute ago   Up About a minute (healthy)   0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 0.0.0.0:9300->9300/tcp, :::9300->9300/tcp             elasticsearch-1
8bdc5e7c7f51   docker.elastic.co/elasticsearch/elasticsearch:7.12.1   "/bin/tini -- /usr/l…"   About a minute ago   Up About a minute (healthy)   9200/tcp, 9300/tcp                                                                               elasticsearch-setup
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

1. 访问 ```https://$HOST_IP:[9200|9201|9202]/_cat/nodes?v```，比如: ```https://192.168.204.107:9202/_cat/nodes?v``` :
    ```bash
    # curl -XGET --insecure -u elastic:elastic https://192.168.204.107:9202/_cat/nodes?v
    ip           heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
    192.168.3.13           62          81   3    0.03    0.35     1.04 cdfhilmrstw -      es03
    192.168.3.11           54          81   3    0.03    0.35     1.04 cdfhilmrstw *      es01
    192.168.3.12           25          81   3    0.03    0.35     1.04 cdfhilmrstw -      es02
    ```

2. 访问 ```https://$CONTAINER_IP:9200/_cat/nodes?v```，比如: ```https://192.168.3.11:9200/_cat/nodes?v``` :
    ```bash
    # curl -XGET --cacert /var/lib/docker/volumes/certs/_data/ca/ca.crt -u elastic:elastic https://192.168.3.11:9200/_cat/nodes?v
    ip           heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
    192.168.3.11           69          81   4    0.32    1.92     1.89 cdfhilmrstw *      es01
    192.168.3.12           63          81   4    0.32    1.92     1.89 cdfhilmrstw -      es02
    192.168.3.13           42          80   4    0.32    1.92     1.89 cdfhilmrstw -      es03
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
