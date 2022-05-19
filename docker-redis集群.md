# docker-redis 集群

## 前言

- redis版本: ```redis:latest```

- 网络配置: 名称为 redis ，子网掩码为 ```192.168.0.0/24```

- 启动六个 redis
    |容器名称|容器IP|映射到宿主机的端口|挂载宿主机的配置文件和数据文件|
    |--|--|--|--|
    |redis-1|192.168.0.11|6371|/usr/local/docker/redis/node-1|
    |redis-2|192.168.0.12|6372|/usr/local/docker/redis/node-2|
    |redis-3|192.168.0.13|6373|/usr/local/docker/redis/node-3|
    |redis-4|192.168.0.14|6374|/usr/local/docker/redis/node-4|
    |redis-5|192.168.0.15|6375|/usr/local/docker/redis/node-5|
    |redis-6|192.168.0.16|6376|/usr/local/docker/redis/node-6|

## 拉取 redis 镜像

```bash
# docker pull redis:latest

# docker images
REPOSITORY                                      TAG       IMAGE ID       CREATED        SIZE
redis                                           latest    7614ae9453d1   4 months ago   113MB
```

## 创建网络

```bash
# docker network create redis --subnet 192.168.0.0/24

# docker network ls
NETWORK ID     NAME             DRIVER    SCOPE
225f82e455f7   back-net         bridge    local
69007f1e6fb0   bridge           bridge    local
95b385dfdc12   docker_default   bridge    local
fbed1fc9e76d   host             host      local
d8ac74ade811   none             null      local
f79b0ef50b9c   redis            bridge    local

# docker network inspect redis
[
    {
        "Name": "redis",
        "Id": "f79b0ef50b9c9b4c6dd7ebce8d3c8d2144dc2e3a7138a83ad6706fe793ffc73a",
        "Created": "2022-05-18T17:45:21.118128333+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.0.0/24"
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

## 在宿主机上创建 redis 目录

```bash
# mkdir -p /usr/local/docker/redis

# vim /usr/local/docker/redis/create-redis-node.sh
for index in $(seq 1 6);
do
mkdir -p /usr/local/docker/redis/node-${index}/conf
touch /usr/local/docker/redis/node-${index}/conf/redis.conf
cat << EOF >> /usr/local/docker/redis/node-${index}/conf/redis.conf
port 6379
bind 0.0.0.0
cluster-enabled yes
cluster-node-timeout 5000
cluster-announce-ip 192.168.0.1${index}
cluster-announce-port 6379
cluster-announce-bus-port 16379
appendonly yes
EOF
done

# chmod +x /usr/local/docker/redis/create-redis-node.sh

# /usr/local/docker/redis/create-redis-node.sh

# ls /usr/local/docker/redis
create-redis-node.sh  node-1  node-2  node-3  node-4  node-5  node-6
```

## 配置 docker-compose.yml

```bash
# vim /usr/local/docker/redis/create-docker-compose.sh
touch /usr/local/docker/redis/docker-compose.yml
cd /usr/local/docker/redis
echo "version: '3.9'" >> docker-compose.yml
echo >> docker-compose.yml
echo "services:" >> docker-compose.yml
for index in $(seq 1 6);
do
echo " redis-"${index}":" >> docker-compose.yml
echo "  image: redis:latest" >> docker-compose.yml
echo "  container_name: redis-"${index} >> docker-compose.yml
echo "  restart: always" >> docker-compose.yml
echo "  command: redis-server /etc/redis/redis.conf" >> docker-compose.yml
echo "  volumes:" >> docker-compose.yml
echo "   - /usr/local/docker/redis/node-"${index}"/data:/data" >> docker-compose.yml
echo "   - /usr/local/docker/redis/node-"${index}"/conf/redis.conf:/etc/redis/redis.conf" >> docker-compose.yml
echo "  ports:" >> docker-compose.yml
echo "   - 637"${index}":6379" >> docker-compose.yml
echo "   - 1637"${index}":16379" >> docker-compose.yml
echo "  networks:" >> docker-compose.yml
echo "    redis:" >> docker-compose.yml
echo "      ipv4_address: 192.168.0.1"${index} >> docker-compose.yml
done
echo "networks:" >> docker-compose.yml
echo "    redis:" >> docker-compose.yml
echo "      external: true" >> docker-compose.yml

# chmod +x /usr/local/docker/redis/create-docker-compose.sh

# /usr/local/docker/redis/create-docker-compose.sh

# ls /usr/local/docker/redis
create-docker-compose.sh  create-redis-node.sh  docker-compose.yml  node-1  node-2  node-3  node-4  node-5  node-6
```

***注，关于自定义 network 的使用方法:***

1. 使用 ```docker network create <network-name>``` 命令创建 network
2. 使用声明的 network
    ```yml
    services:
      service-name:
        networks:
          - network-name
    ```
    或
    ```yml
    services:
     service-name:
      networks:
        network-name:
          ipv4_address: 192.168.0.xx
    ```
3. 声明 network
    ```yml
    networks:
        network-name:
          external: true
    ```
4. 否则在使用 ```docker-compose up -d``` 时, 会报错 ```ERROR: Service "service-name" uses an undefined network "network-name"```

## 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/redis/docker-compose.yml up -d
[+] Running 6/6
 ⠿ Container redis-1  Started                                                                                    2.8s
 ⠿ Container redis-4  Started                                                                                    3.9s
 ⠿ Container redis-5  Started                                                                                    2.8s
 ⠿ Container redis-2  Started                                                                                    3.1s
 ⠿ Container redis-6  Started                                                                                    4.2s
 ⠿ Container redis-3  Started                                                                                    3.1s

# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                                                            NAMES
1405050ffb23   redis:latest             "docker-entrypoint.s…"   28 seconds ago   Up 25 seconds   0.0.0.0:6372->6379/tcp, :::6372->6379/tcp, 0.0.0.0:16372->16379/tcp, :::16372->16379/tcp         redis-2
9b904d54075a   redis:latest             "docker-entrypoint.s…"   28 seconds ago   Up 24 seconds   0.0.0.0:6374->6379/tcp, :::6374->6379/tcp, 0.0.0.0:16374->16379/tcp, :::16374->16379/tcp         redis-4
ba0aef2f5880   redis:latest             "docker-entrypoint.s…"   28 seconds ago   Up 24 seconds   0.0.0.0:6376->6379/tcp, :::6376->6379/tcp, 0.0.0.0:16376->16379/tcp, :::16376->16379/tcp         redis-6
0ec14eb5d645   redis:latest             "docker-entrypoint.s…"   28 seconds ago   Up 25 seconds   0.0.0.0:6375->6379/tcp, :::6375->6379/tcp, 0.0.0.0:16375->16379/tcp, :::16375->16379/tcp         redis-5
83bcb0c72101   redis:latest             "docker-entrypoint.s…"   28 seconds ago   Up 25 seconds   0.0.0.0:6371->6379/tcp, :::6371->6379/tcp, 0.0.0.0:16371->16379/tcp, :::16371->16379/tcp         redis-1
640ad956d77f   redis:latest             "docker-entrypoint.s…"   28 seconds ago   Up 25 seconds   0.0.0.0:6373->6379/tcp, :::6373->6379/tcp, 0.0.0.0:16373->16379/tcp, :::16373->16379/tcp         redis-3
```

## 创建集群

1. 进入容器:
    ```bash
    # docker exec -it redis-1 /bin/sh
    ```
2. 进入容器之后，执行创建集群的命令，提示输入的时候，输入 ```yes```:
    ```bash
    # redis-cli --cluster create 192.168.0.11:6379 192.168.0.12:6379 192.168.0.13:6379 192.168.0.14:6379 192.168.0.15:6379 192.168.0.16:6379 --cluster-replicas 1
    >>> Performing hash slots allocation on 6 nodes...
    Master[0] -> Slots 0 - 5460
    Master[1] -> Slots 5461 - 10922
    Master[2] -> Slots 10923 - 16383
    Adding replica 192.168.0.15:6379 to 192.168.0.11:6379
    Adding replica 192.168.0.16:6379 to 192.168.0.12:6379
    Adding replica 192.168.0.14:6379 to 192.168.0.13:6379
    M: 628f579b269a2792dff63e6b5e15d91d908268a6 192.168.0.11:6379
       slots:[0-5460] (5461 slots) master
    M: aeac13e8c2135d0f61aede9a528ad7facf1e730f 192.168.0.12:6379
       slots:[5461-10922] (5462 slots) master
    M: a5c774037229a6c3aa1fb56e608ee7ad49cadae7 192.168.0.13:6379
       slots:[10923-16383] (5461 slots) master
    S: b50edeaffa1b262300dacc511da586a391acb7aa 192.168.0.14:6379
       replicates a5c774037229a6c3aa1fb56e608ee7ad49cadae7
    S: c1baf0206dc80cf58fcf490eb200e5e48afdd5cc 192.168.0.15:6379
       replicates 628f579b269a2792dff63e6b5e15d91d908268a6
    S: af5a985a56024894fe18e2a0ab6d3f1b74d825dd 192.168.0.16:6379
       replicates aeac13e8c2135d0f61aede9a528ad7facf1e730f
    Can I set the above configuration? (type 'yes' to accept): yes      
    >>> Nodes configuration updated
    >>> Assign a different config epoch to each node
    >>> Sending CLUSTER MEET messages to join the cluster
    Waiting for the cluster to join
    .
    >>> Performing Cluster Check (using node 192.168.0.11:6379)
    M: 628f579b269a2792dff63e6b5e15d91d908268a6 192.168.0.11:6379
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    S: c1baf0206dc80cf58fcf490eb200e5e48afdd5cc 192.168.0.15:6379
       slots: (0 slots) slave
       replicates 628f579b269a2792dff63e6b5e15d91d908268a6
    S: b50edeaffa1b262300dacc511da586a391acb7aa 192.168.0.14:6379
       slots: (0 slots) slave
       replicates a5c774037229a6c3aa1fb56e608ee7ad49cadae7
    M: aeac13e8c2135d0f61aede9a528ad7facf1e730f 192.168.0.12:6379
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
    S: af5a985a56024894fe18e2a0ab6d3f1b74d825dd 192.168.0.16:6379
       slots: (0 slots) slave
       replicates aeac13e8c2135d0f61aede9a528ad7facf1e730f
    M: a5c774037229a6c3aa1fb56e608ee7ad49cadae7 192.168.0.13:6379
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.
    ```
3. 查看集群:
    ```bash
    # redis-cli
    127.0.0.1:6379> cluster nodes
    628f579b269a2792dff63e6b5e15d91d908268a6 192.168.0.11:6379@16379 myself,master - 0 1652933957000 1 connected 0-5460
    c1baf0206dc80cf58fcf490eb200e5e48afdd5cc 192.168.0.15:6379@16379 slave 628f579b269a2792dff63e6b5e15d91d908268a6 0 1652933959336 1 connected
    b50edeaffa1b262300dacc511da586a391acb7aa 192.168.0.14:6379@16379 slave a5c774037229a6c3aa1fb56e608ee7ad49cadae7 0 1652933958000 3 connected
    aeac13e8c2135d0f61aede9a528ad7facf1e730f 192.168.0.12:6379@16379 master - 0 1652933959000 2 connected 5461-10922
    af5a985a56024894fe18e2a0ab6d3f1b74d825dd 192.168.0.16:6379@16379 slave aeac13e8c2135d0f61aede9a528ad7facf1e730f 0 1652933959000 2 connected
    a5c774037229a6c3aa1fb56e608ee7ad49cadae7 192.168.0.13:6379@16379 master - 0 1652933959000 3 connected 10923-16383
    ```
