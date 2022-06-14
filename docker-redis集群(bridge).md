# docker-redis 集群 (bridge)

## 前言

- redis版本: ```redis:latest```

- 网络配置: 驱动类型为 bridge，名称为 redis ，子网掩码为 ```192.168.0.0/24```，网关为 ```192.168.0.1```

- 启动六个 redis
    |容器名称|容器IP|容器的端口|宿主机IP|映射到宿主机的端口|挂载宿主机的配置文件和数据文件|
    |--|--|--|--|--|--|
    |redis-1|192.168.0.11|6379|192.168.204.107|6371|/usr/local/docker/redis/node-1|
    |redis-2|192.168.0.12|6379|192.168.204.107|6372|/usr/local/docker/redis/node-2|
    |redis-3|192.168.0.13|6379|192.168.204.107|6373|/usr/local/docker/redis/node-3|
    |redis-4|192.168.0.14|6379|192.168.204.107|6374|/usr/local/docker/redis/node-4|
    |redis-5|192.168.0.15|6379|192.168.204.107|6375|/usr/local/docker/redis/node-5|
    |redis-6|192.168.0.16|6379|192.168.204.107|6376|/usr/local/docker/redis/node-6|

## 拉取 redis 镜像

```bash
# docker pull redis:latest

# docker images
REPOSITORY                                      TAG       IMAGE ID       CREATED        SIZE
redis                                           latest    7614ae9453d1   4 months ago   113MB
```

## 创建网络

```bash
# docker network create redis --subnet 192.168.0.0/24 --gateway=192.168.0.1

# docker network ls
NETWORK ID     NAME             DRIVER    SCOPE
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
                    "Subnet": "192.168.0.0/24",
                    "Gateway": "192.168.0.1"
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

# > /usr/local/docker/redis/create-node.sh

# vim /usr/local/docker/redis/create-node.sh
#!/bin/sh
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

# chmod +x /usr/local/docker/redis/create-node.sh

# /usr/local/docker/redis/create-node.sh

# ls /usr/local/docker/redis
create-node.sh  node-1  node-2  node-3  node-4  node-5  node-6
```

## 配置 docker-compose.yml

```bash
# > /usr/local/docker/redis/create-docker-compose.sh

# vim /usr/local/docker/redis/create-docker-compose.sh
#!/bin/sh
> /usr/local/docker/redis/docker-compose.yml
touch /usr/local/docker/redis/docker-compose.yml
cat << EOF >> /usr/local/docker/redis/docker-compose.yml
version: '3.9'

services:
EOF
for index in $(seq 1 6);
do
cat << EOF >> /usr/local/docker/redis/docker-compose.yml
 redis-${index}:
  image: redis:latest
  container_name: redis-${index}
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-${index}/data:/data
   - /usr/local/docker/redis/node-${index}/conf/redis.conf:/etc/redis/redis.conf
  ports:
   - 637${index}:6379
   - 1637${index}:16379
  networks:
    redis:
      ipv4_address: 192.168.0.1${index}
EOF
done
cat << EOF >> /usr/local/docker/redis/docker-compose.yml
networks:
    redis:
      name: redis
EOF

# chmod +x /usr/local/docker/redis/create-docker-compose.sh

# /usr/local/docker/redis/create-docker-compose.sh

# ls /usr/local/docker/redis
create-docker-compose.sh  create-node.sh  docker-compose.yml  node-1  node-2  node-3  node-4  node-5  node-6
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
          name: network-name
    ```
4. 否则在使用 ```docker-compose up -d``` 时, 会报错 ```ERROR: Service "service-name" uses an undefined network "network-name"```

***完整的 docker-compose.yml:***

```yml
version: '3.9'

services:
 redis-1:
  image: redis:latest
  container_name: redis-1
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-1/data:/data
   - /usr/local/docker/redis/node-1/conf/redis.conf:/etc/redis/redis.conf
  ports:
   - 6371:6379
   - 16371:16379
  networks:
    redis:
      ipv4_address: 192.168.0.11
 redis-2:
  image: redis:latest
  container_name: redis-2
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-2/data:/data
   - /usr/local/docker/redis/node-2/conf/redis.conf:/etc/redis/redis.conf
  ports:
   - 6372:6379
   - 16372:16379
  networks:
    redis:
      ipv4_address: 192.168.0.12
 redis-3:
  image: redis:latest
  container_name: redis-3
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-3/data:/data
   - /usr/local/docker/redis/node-3/conf/redis.conf:/etc/redis/redis.conf
  ports:
   - 6373:6379
   - 16373:16379
  networks:
    redis:
      ipv4_address: 192.168.0.13
 redis-4:
  image: redis:latest
  container_name: redis-4
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-4/data:/data
   - /usr/local/docker/redis/node-4/conf/redis.conf:/etc/redis/redis.conf
  ports:
   - 6374:6379
   - 16374:16379
  networks:
    redis:
      ipv4_address: 192.168.0.14
 redis-5:
  image: redis:latest
  container_name: redis-5
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-5/data:/data
   - /usr/local/docker/redis/node-5/conf/redis.conf:/etc/redis/redis.conf
  ports:
   - 6375:6379
   - 16375:16379
  networks:
    redis:
      ipv4_address: 192.168.0.15
 redis-6:
  image: redis:latest
  container_name: redis-6
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-6/data:/data
   - /usr/local/docker/redis/node-6/conf/redis.conf:/etc/redis/redis.conf
  ports:
   - 6376:6379
   - 16376:16379
  networks:
    redis:
      ipv4_address: 192.168.0.16
networks:
    redis:
      name: redis
```

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

## 查看网络

```bash
# docker network inspect redis
[
    {
        "Name": "redis",
        "Id": "efec85c6b337dd2b0a0223b8bedbda2d77aa1083523e73c2a0280abc84553ee7",
        "Created": "2022-05-25T21:31:06.419183514-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.0.0/24",
                    "Gateway": "192.168.0.1"
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
            "4d79a4dc2a29eba7496797747dec11b61974c9287e0e68e7e504137b46170749": {
                "Name": "redis-2",
                "EndpointID": "203c906b40a34063dde7fb790a6c1a9f8fcdd6b865008e006110469c44a58c5a",
                "MacAddress": "02:42:c0:a8:00:0c",
                "IPv4Address": "192.168.0.12/24",
                "IPv6Address": ""
            },
            "54f02643022c01d48925f7511bedd8c6a0b14497a725f062e05a058776c04602": {
                "Name": "redis-5",
                "EndpointID": "d9d75d9bbc9feb12582241e01ffdd2221517381461aac6423290430f135118a1",
                "MacAddress": "02:42:c0:a8:00:0f",
                "IPv4Address": "192.168.0.15/24",
                "IPv6Address": ""
            },
            "6441cb03a85409131f0f935c7a6b2609d786b29a4d6e9d00d80e9a991acd735f": {
                "Name": "redis-3",
                "EndpointID": "85712e08f8dba4ed22b8223343be902a6c762f52c4a1c676e1b15e4a47dccee0",
                "MacAddress": "02:42:c0:a8:00:0d",
                "IPv4Address": "192.168.0.13/24",
                "IPv6Address": ""
            },
            "833b8c8c96e95474641e78e9eb1041862d729853633312ce89ed07586270f677": {
                "Name": "redis-4",
                "EndpointID": "0d26640bbd82281858ad48b1e206b4dc4a7d70e6a85d004f09c0ebf32c53dd79",
                "MacAddress": "02:42:c0:a8:00:0e",
                "IPv4Address": "192.168.0.14/24",
                "IPv6Address": ""
            },
            "94c1e015d2272a7e1f0e5c79f8d3e41613267aee3ec8aaa1773dbf41a59b0d65": {
                "Name": "redis-1",
                "EndpointID": "b2e97b4425c8b148c5e204bd3a6406c994be11080d0760c72338a47a43d9a0a7",
                "MacAddress": "02:42:c0:a8:00:0b",
                "IPv4Address": "192.168.0.11/24",
                "IPv6Address": ""
            },
            "fccfc328759a319f7c6d13f68a6d9a23c98169dd152c3d274859ad17eb6591fb": {
                "Name": "redis-6",
                "EndpointID": "8857040b2e2835dfd5844d70eab9a42f96c855bd895bca9cb3ee0465b04b3bff",
                "MacAddress": "02:42:c0:a8:00:10",
                "IPv4Address": "192.168.0.16/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

***容器 redis-1,redis-2,redis-3,redis-4,redis-5,redis-6 都已经加入到了 redis 网络中。***

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
4. 操作示例:
    ```bash
    # redis-cli -c    
    127.0.0.1:6379> set x 10
    -> Redirected to slot [16287] located at 192.168.0.13:6379
    OK
    192.168.0.13:6379> get x
    "10"
    ```
