# docker-redis 集群(host)

## 前言

- redis版本: ```redis:latest```

- 网络配置: 驱动类型为 host

- 启动六个 redis
    |容器名称|宿主机IP|宿主机的端口|挂载宿主机的配置文件和数据文件|
    |--|--|--|--|
    |redis-1|192.168.204.107|6371|/usr/local/docker/redis/node-1|
    |redis-2|192.168.204.107|6372|/usr/local/docker/redis/node-2|
    |redis-3|192.168.204.107|6373|/usr/local/docker/redis/node-3|
    |redis-4|192.168.204.107|6374|/usr/local/docker/redis/node-4|
    |redis-5|192.168.204.107|6375|/usr/local/docker/redis/node-5|
    |redis-6|192.168.204.107|6376|/usr/local/docker/redis/node-6|

## 拉取 redis 镜像

```bash
# docker pull redis:latest

# docker images
REPOSITORY                                      TAG       IMAGE ID       CREATED        SIZE
redis                                           latest    7614ae9453d1   4 months ago   113MB
```

## 在宿主机上创建 redis 目录

```bash
# mkdir -p /usr/local/docker/redis

# vim /usr/local/docker/redis/create-node.sh
#!/bin/sh
for index in $(seq 1 6);
do
mkdir -p /usr/local/docker/redis/node-${index}/conf
touch /usr/local/docker/redis/node-${index}/conf/redis.conf
cat << EOF >> /usr/local/docker/redis/node-${index}/conf/redis.conf
port 637${index}
bind 0.0.0.0
cluster-enabled yes
cluster-node-timeout 5000
cluster-announce-port 637${index}
cluster-announce-bus-port 1637${index}
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
  network_mode: host
EOF
done

# chmod +x /usr/local/docker/redis/create-docker-compose.sh

# /usr/local/docker/redis/create-docker-compose.sh

# ls /usr/local/docker/redis
create-docker-compose.sh  create-node.sh  docker-compose.yml  node-1  node-2  node-3  node-4  node-5  node-6
```

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
  network_mode: host
 redis-2:
  image: redis:latest
  container_name: redis-2
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-2/data:/data
   - /usr/local/docker/redis/node-2/conf/redis.conf:/etc/redis/redis.conf
  network_mode: host
 redis-3:
  image: redis:latest
  container_name: redis-3
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-3/data:/data
   - /usr/local/docker/redis/node-3/conf/redis.conf:/etc/redis/redis.conf
  network_mode: host
 redis-4:
  image: redis:latest
  container_name: redis-4
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-4/data:/data
   - /usr/local/docker/redis/node-4/conf/redis.conf:/etc/redis/redis.conf
  network_mode: host
 redis-5:
  image: redis:latest
  container_name: redis-5
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-5/data:/data
   - /usr/local/docker/redis/node-5/conf/redis.conf:/etc/redis/redis.conf
  network_mode: host
 redis-6:
  image: redis:latest
  container_name: redis-6
  restart: always
  command: redis-server /etc/redis/redis.conf
  volumes:
   - /usr/local/docker/redis/node-6/data:/data
   - /usr/local/docker/redis/node-6/conf/redis.conf:/etc/redis/redis.conf
  network_mode: host
```

## 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/redis/docker-compose.yml up -d
[+] Running 6/6
 ⠿ Container redis-5  Started                                                                                    1.2s
 ⠿ Container redis-6  Started                                                                                    0.6s
 ⠿ Container redis-1  Started                                                                                    1.2s
 ⠿ Container redis-2  Started                                                                                    1.1s
 ⠿ Container redis-3  Started                                                                                    0.9s
 ⠿ Container redis-4  Started                                                                                    0.9s

# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS     NAMES
badc2904b481   redis:latest   "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds             redis-3
a6f93b0aff1e   redis:latest   "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds             redis-6
0e7a19b8d456   redis:latest   "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds             redis-1
fe1bd8b2b6d8   redis:latest   "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds             redis-2
c9e01f06467c   redis:latest   "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds             redis-4
5f03725d0d9c   redis:latest   "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds             redis-5
```

## 创建集群

1. 进入容器:
    ```bash
    # docker exec -it redis-1 /bin/sh
    ```
2. 进入容器之后，执行创建集群的命令，提示输入的时候，输入 ```yes```:
    ```bash
    # redis-cli --cluster create 192.168.204.107:6371 192.168.204.107:6372 192.168.204.107:6373 192.168.204.107:6374 192.168.204.107:6375 192.168.204.107:6376 --cluster-replicas 1
    >>> Performing hash slots allocation on 6 nodes...
    Master[0] -> Slots 0 - 5460
    Master[1] -> Slots 5461 - 10922
    Master[2] -> Slots 10923 - 16383
    Adding replica 192.168.204.107:6375 to 192.168.204.107:6371
    Adding replica 192.168.204.107:6376 to 192.168.204.107:6372
    Adding replica 192.168.204.107:6374 to 192.168.204.107:6373
    >>> Trying to optimize slaves allocation for anti-affinity
    [WARNING] Some slaves are in the same host as their master
    M: 9d9154672275178bdc47a32c91c41a8855d47529 192.168.204.107:6371
       slots:[0-5460] (5461 slots) master
    M: d79c03f58d20d478df0099d4b606a75178be1813 192.168.204.107:6372
       slots:[5461-10922] (5462 slots) master
    M: f96cbcff15985311d8d2970bc161e20a5b694c9f 192.168.204.107:6373
       slots:[10923-16383] (5461 slots) master
    S: 095788fde6f6b663fd1b582d861b4ea9d1f49c08 192.168.204.107:6374
       replicates d79c03f58d20d478df0099d4b606a75178be1813
    S: e8d7bcf3340393a1f38cb0ef4141bd1850c950e1 192.168.204.107:6375
       replicates f96cbcff15985311d8d2970bc161e20a5b694c9f
    S: b841e1be29c3964f2cba6734f3ba4d598a1c94bf 192.168.204.107:6376
       replicates 9d9154672275178bdc47a32c91c41a8855d47529
    Can I set the above configuration? (type 'yes' to accept): yes  
    >>> Nodes configuration updated
    >>> Assign a different config epoch to each node
    >>> Sending CLUSTER MEET messages to join the cluster
    Waiting for the cluster to join
    .
    >>> Performing Cluster Check (using node 192.168.204.107:6371)
    M: 9d9154672275178bdc47a32c91c41a8855d47529 192.168.204.107:6371
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    M: f96cbcff15985311d8d2970bc161e20a5b694c9f 192.168.204.107:6373
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
    S: b841e1be29c3964f2cba6734f3ba4d598a1c94bf 192.168.204.107:6376
       slots: (0 slots) slave
       replicates 9d9154672275178bdc47a32c91c41a8855d47529
    S: e8d7bcf3340393a1f38cb0ef4141bd1850c950e1 192.168.204.107:6375
       slots: (0 slots) slave
       replicates f96cbcff15985311d8d2970bc161e20a5b694c9f
    M: d79c03f58d20d478df0099d4b606a75178be1813 192.168.204.107:6372
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
    S: 095788fde6f6b663fd1b582d861b4ea9d1f49c08 192.168.204.107:6374
       slots: (0 slots) slave
       replicates d79c03f58d20d478df0099d4b606a75178be1813
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.
    ```
3. 查看集群:
    ```bash
    # redis-cli -p 6371
    127.0.0.1:6371> cluster nodes
    9d9154672275178bdc47a32c91c41a8855d47529 192.168.204.107:6371@16371 myself,master - 0 1653008233000 1 connected 0-5460
    f96cbcff15985311d8d2970bc161e20a5b694c9f 192.168.204.107:6373@16373 master - 0 1653008233000 3 connected 10923-16383
    b841e1be29c3964f2cba6734f3ba4d598a1c94bf 192.168.204.107:6376@16376 slave 9d9154672275178bdc47a32c91c41a8855d47529 0 1653008233736 1 connected
    e8d7bcf3340393a1f38cb0ef4141bd1850c950e1 192.168.204.107:6375@16375 slave f96cbcff15985311d8d2970bc161e20a5b694c9f 0 1653008234745 3 connected
    d79c03f58d20d478df0099d4b606a75178be1813 192.168.204.107:6372@16372 master - 0 1653008234000 2 connected 5461-10922
    095788fde6f6b663fd1b582d861b4ea9d1f49c08 192.168.204.107:6374@16374 slave d79c03f58d20d478df0099d4b606a75178be1813 0 1653008233000 2 connected
    ```
4. 操作示例:
    ```bash
    # redis-cli -p 6371 -c
    127.0.0.1:6371> set x 100
    -> Redirected to slot [16287] located at 192.168.204.107:6373
    OK
    192.168.204.107:6373> get x
    "100"
    ```
