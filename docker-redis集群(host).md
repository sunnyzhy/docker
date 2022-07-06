# docker-redis 集群(host)

## 前言

- redis版本: ```redis:latest```

- 网络配置: 驱动类型为 overlay

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- 启动六个 redis
    |容器名称|宿主机IP|宿主机的端口|挂载宿主机的配置文件和数据文件|
    |--|--|--|--|
    |redis_1|192.168.5.163|6379|/usr/local/docker/redis/node-1/conf:/etc/redis<br />/usr/local/docker/redis/node-1/data\:/data<br />/usr/local/docker/redis/node-1/log:/var/log/redis|
    |redis_2|192.168.5.163|6380|/usr/local/docker/redis/node-2/conf:/etc/redis<br />/usr/local/docker/redis/node-2/data\:/data<br />/usr/local/docker/redis/node-2/log:/var/log/redis|
    |redis_3|192.168.5.164|6379|/usr/local/docker/redis/node-3/conf:/etc/redis<br />/usr/local/docker/redis/node-3/data\:/data<br />/usr/local/docker/redis/node-3/log:/var/log/redis|
    |redis_4|192.168.5.164|6380|/usr/local/docker/redis/node-4/conf:/etc/redis<br />/usr/local/docker/redis/node-4/data\:/data<br />/usr/local/docker/redis/node-4/log:/var/log/redis|
    |redis_5|192.168.5.165|6379|/usr/local/docker/redis/node-5/conf:/etc/redis<br />/usr/local/docker/redis/node-5/data\:/data<br />/usr/local/docker/redis/node-5/log:/var/log/redis|
    |redis_6|192.168.5.165|6380|/usr/local/docker/redis/node-6/conf:/etc/redis<br />/usr/local/docker/redis/node-6/data\:/data<br />/usr/local/docker/redis/node-6/log:/var/log/redis|

- ```--cluster-replicas 1```: 表示 ```主节点数和从节点数的比例等于 1``` 。在创建集群的时候，将按照 ```IP:PORT``` 的顺序，先生成 3 个主节点，再生成 3 个从节点。

- 拉取 redis.conf
   - 当前版本 redis.conf:
      ```bash
      # docker image inspect redis | grep REDIS_VERSION | sed 's/[," ]//g'
      REDIS_VERSION=6.2.6
      REDIS_VERSION=6.2.6
      
      # wget -P /usr/local/docker/redis https://download.redis.io/releases/redis-6.2.6.tar.gz
      
      # tar -xzvf /usr/local/docker/redis/redis-6.2.6.tar.gz -C /usr/local/docker/redis
      
      # cp /usr/local/docker/redis/redis-6.2.6/redis.conf /usr/local/docker/redis
      ```
   - 最新版 redis.conf:
      ```bash
      # wget -P /usr/local/docker/redis http://download.redis.io/redis-stable/redis.conf
      ```

## 拉取 redis 镜像

```bash
# docker pull redis:latest

# docker images
REPOSITORY                                      TAG       IMAGE ID       CREATED        SIZE
redis                                           latest    7614ae9453d1   4 months ago   113MB
```

## 在宿主机上创建 redis 目录

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```sh
#!/bin/sh

for index in $(seq 1 6);
do
    mkdir -p /usr/local/docker/redis/node-${index}/{conf,data,log}

    if [ ${index} == 1 ]; then
        if [ ! -f /usr/local/docker/redis/node-${index}/conf/redis.conf ]; then
            wget -P /usr/local/docker/redis https://download.redis.io/releases/redis-6.2.6.tar.gz
            tar -xzvf /usr/local/docker/redis/redis-6.2.6.tar.gz -C /usr/local/docker/redis
            cp /usr/local/docker/redis/redis-6.2.6/redis.conf /usr/local/docker/redis/node-${index}/conf
            rm -rf /usr/local/docker/redis/redis-6.2.6*
        fi
    else
        if [ ! -f /usr/local/docker/redis/node-${index}/conf/redis.conf ]; then
            cp /usr/local/docker/redis/node-1/conf/redis.conf /usr/local/docker/redis/node-${index}/conf
        fi
    fi

    chmod 777 -R /usr/local/docker/redis/node-${index}/{conf,data,log}
done
```

```bash
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/redis/
/usr/local/docker/redis/
├── node-1
│   ├── conf
│   │   └── redis.conf
│   ├── data
│   └── log
├── node-2
│   ├── conf
│   │   └── redis.conf
│   ├── data
│   └── log
├── node-3
│   ├── conf
│   │   └── redis.conf
│   ├── data
│   └── log
├── node-4
│   ├── conf
│   │   └── redis.conf
│   ├── data
│   └── log
├── node-5
│   ├── conf
│   │   └── redis.conf
│   ├── data
│   └── log
└── node-6
    ├── conf
    │   └── redis.conf
    ├── data
    └── log
```

复制 192.168.5.163(manager) 的 redis 目录到 192.168.5.164(worker):

```bash
# scp -r /usr/local/docker/redis root@192.168.5.164:/usr/local/docker
```

复制 192.168.5.163(manager) 的 redis 目录到 192.168.5.165(worker):

```bash
# scp -r /usr/local/docker/redis root@192.168.5.165:/usr/local/docker
```

## 配置 docker-compose.yml

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# vim /usr/local/docker/docker-compose.yml
```

```yml
version: '3.9'

services:
 redis_1:
  image: redis:latest
  command: redis-server /etc/redis/redis.conf --port 6379 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --cluster-announce-ip 192.168.5.163 --cluster-config-file /etc/redis/nodes.conf --cluster-announce-port 6379 --cluster-announce-bus-port 16379 --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-1/conf:/etc/redis
    - /usr/local/docker/redis/node-1/data:/data
    - /usr/local/docker/redis/node-1/log:/var/log/redis
    - /etc/timezone:/etc/timezone
    - /etc/localtime:/etc/localtime
  ports:
    - 6379:6379
    - 16379:16379
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.hostname == centos-docker-163
        - node.role == manager
  networks:
    - iot
 redis_2:
  image: redis:latest
  command: redis-server /etc/redis/redis.conf --port 6380 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --cluster-announce-ip 192.168.5.163 --cluster-config-file /etc/redis/nodes.conf --cluster-announce-port 6380 --cluster-announce-bus-port 16380 --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-2/conf:/etc/redis
    - /usr/local/docker/redis/node-2/data:/data
    - /usr/local/docker/redis/node-2/log:/var/log/redis
    - /etc/timezone:/etc/timezone
    - /etc/localtime:/etc/localtime
  ports:
    - 6380:6380
    - 16380:16380
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.hostname == centos-docker-163
        - node.role == manager
  networks:
    - iot
 redis_3:
  image: redis:latest
  command: redis-server /etc/redis/redis.conf --port 6381 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --cluster-announce-ip 192.168.5.164 --cluster-config-file /etc/redis/nodes.conf --cluster-announce-port 6381 --cluster-announce-bus-port 16381 --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-3/conf:/etc/redis
    - /usr/local/docker/redis/node-3/data:/data
    - /usr/local/docker/redis/node-3/log:/var/log/redis
    - /etc/timezone:/etc/timezone
    - /etc/localtime:/etc/localtime
  ports:
    - 6381:6381
    - 16381:16381
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.hostname == centos-docker-164
        - node.role == worker
  networks:
    - iot
 redis_4:
  image: redis:latest
  command: redis-server /etc/redis/redis.conf --port 6382 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --cluster-announce-ip 192.168.5.164 --cluster-config-file /etc/redis/nodes.conf --cluster-announce-port 6382 --cluster-announce-bus-port 16382 --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-4/conf:/etc/redis
    - /usr/local/docker/redis/node-4/data:/data
    - /usr/local/docker/redis/node-4/log:/var/log/redis
    - /etc/timezone:/etc/timezone
    - /etc/localtime:/etc/localtime
  ports:
    - 6382:6382
    - 16382:16382
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.hostname == centos-docker-164
        - node.role == worker
  networks:
    - iot
 redis_5:
  image: redis:latest
  command: redis-server /etc/redis/redis.conf --port 6383 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --cluster-announce-ip 192.168.5.165 --cluster-config-file /etc/redis/nodes.conf --cluster-announce-port 6383 --cluster-announce-bus-port 16383 --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-5/conf:/etc/redis
    - /usr/local/docker/redis/node-5/data:/data
    - /usr/local/docker/redis/node-5/log:/var/log/redis
    - /etc/timezone:/etc/timezone
    - /etc/localtime:/etc/localtime
  ports:
    - 6383:6383
    - 16383:16383
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.hostname == centos-docker-165
        - node.role == worker
  networks:
    - iot
 redis_6:
  image: redis:latest
  command: redis-server /etc/redis/redis.conf --port 6384 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --cluster-announce-ip 192.168.5.165 --cluster-config-file /etc/redis/nodes.conf --cluster-announce-port 6384 --cluster-announce-bus-port 16384 --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-6/conf:/etc/redis
    - /usr/local/docker/redis/node-6/data:/data
    - /usr/local/docker/redis/node-6/log:/var/log/redis
    - /etc/timezone:/etc/timezone
    - /etc/localtime:/etc/localtime
  ports:
    - 6384:6384
    - 16384:16384
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

在宿主机 192.168.5.163(manager) 上启动 redis 服务:

```bash
# docker stack deploy -c /usr/local/docker/docker-compose.yml iot
Creating service iot_redis_5
Creating service iot_redis_6
Creating service iot_redis_1
Creating service iot_redis_2
Creating service iot_redis_3
Creating service iot_redis_4

# docker stack ps iot
ID             NAME            IMAGE          NODE                DESIRED STATE   CURRENT STATE                ERROR     PORTS
49iswjsmal4v   iot_redis_1.1   redis:latest   centos-docker-163   Running         Running about a minute ago             
qm6d7djo94ge   iot_redis_2.1   redis:latest   centos-docker-163   Running         Running about a minute ago             
kaodrj49hs0u   iot_redis_3.1   redis:latest   centos-docker-164   Running         Running about a minute ago             
irutlz6qry2c   iot_redis_4.1   redis:latest   centos-docker-164   Running         Running 2 minutes ago                  
v70q1slp1a65   iot_redis_5.1   redis:latest   centos-docker-165   Running         Running 2 minutes ago                  
o9dhfx01cr9o   iot_redis_6.1   redis:latest   centos-docker-165   Running         Running about a minute ago          

# docker service ls
ID             NAME          MODE         REPLICAS   IMAGE          PORTS
zmb899ywanem   iot_redis_1   replicated   1/1        redis:latest   *:6379->6379/tcp, *:16379->16379/tcp
l630h7vv74lm   iot_redis_2   replicated   1/1        redis:latest   *:6380->6380/tcp, *:16380->16380/tcp
sk36gkco96jb   iot_redis_3   replicated   1/1        redis:latest   *:6381->6381/tcp, *:16381->16381/tcp
mox4ubyhffx9   iot_redis_4   replicated   1/1        redis:latest   *:6382->6382/tcp, *:16382->16382/tcp
jakrtm4c9oqz   iot_redis_5   replicated   1/1        redis:latest   *:6383->6383/tcp, *:16383->16383/tcp
f6ao1snregto   iot_redis_6   replicated   1/1        redis:latest   *:6384->6384/tcp, *:16384->16384/tcp
```

## 查看 redis 容器

查看宿主机 192.168.5.163 的 redis 容器:

```bash
# docker ps
CONTAINER ID   IMAGE          COMMAND                   CREATED         STATUS         PORTS      NAMES
1ab0dfe74274   redis:latest   "docker-entrypoint.s…"   2 minutes ago   Up 2 minutes   6379/tcp   iot_redis_2.1.qm6d7djo94geyrnmqmdhchkd3
7059d2279592   redis:latest   "docker-entrypoint.s…"   2 minutes ago   Up 2 minutes   6379/tcp   iot_redis_1.1.49iswjsmal4vpemyyv08ri8ur
```

查看宿主机 192.168.5.164 的 redis 容器:

```bash
# docker ps
CONTAINER ID   IMAGE          COMMAND                   CREATED         STATUS         PORTS      NAMES
5a8c795a58f8   redis:latest   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   6379/tcp   iot_redis_3.1.kaodrj49hs0uigmn4xareyydc
4cb87379b82c   redis:latest   "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes   6379/tcp   iot_redis_4.1.irutlz6qry2ca1a0ezp4izi3u
```

查看宿主机 192.168.5.165 的 redis 容器:

```bash
# docker ps
CONTAINER ID   IMAGE          COMMAND                   CREATED         STATUS         PORTS      NAMES
5da96d1c5b5f   redis:latest   "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes   6379/tcp   iot_redis_6.1.o9dhfx01cr9o7tc2scn0b4rfz
42dffbbb458b   redis:latest   "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes   6379/tcp   iot_redis_5.1.v70q1slp1a65rxpbpsz1mgn5n
```

## 创建集群

1. 进入宿主机 192.168.5.163 的 iot_redis_1 容器:
    ```bash
    # docker exec -it 7059d2279592 /bin/sh
    ```
2. 进入容器之后，执行创建集群的命令，提示输入的时候，输入 ```yes```:
    ```bash
    # redis-cli -a 123456 --cluster create 192.168.5.163:6379 192.168.5.163:6380 192.168.5.164:6381 192.168.5.164:6382 192.168.5.165:6383 192.168.5.165:6384 --cluster-replicas 1
    Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
    >>> Performing hash slots allocation on 6 nodes...
    Master[0] -> Slots 0 - 5460
    Master[1] -> Slots 5461 - 10922
    Master[2] -> Slots 10923 - 16383
    Adding replica 192.168.5.164:6382 to 192.168.5.163:6379
    Adding replica 192.168.5.165:6384 to 192.168.5.164:6381
    Adding replica 192.168.5.163:6380 to 192.168.5.165:6383
    M: fc0fc475d49bc02c53a1f82423c728b7621955fc 192.168.5.163:6379
       slots:[0-5460] (5461 slots) master
    S: f4bd25ab6a7cdc0945e168939e89a2292d3f61b0 192.168.5.163:6380
       replicates c84ed4fd5b59bb285360fd2dd3df982e1b584777
    M: 9afe793e4f1fec538521b2f153ca83044269d29c 192.168.5.164:6381
       slots:[5461-10922] (5462 slots) master
    S: 793007fbf806fa7a68944a76ced8f6757eb9a265 192.168.5.164:6382
       replicates fc0fc475d49bc02c53a1f82423c728b7621955fc
    M: c84ed4fd5b59bb285360fd2dd3df982e1b584777 192.168.5.165:6383
       slots:[10923-16383] (5461 slots) master
    S: b8a3b03d610eb41d2a813ddb1ebceae72cec4dc2 192.168.5.165:6384
       replicates 9afe793e4f1fec538521b2f153ca83044269d29c
    Can I set the above configuration? (type 'yes' to accept): yes
    >>> Nodes configuration updated
    >>> Assign a different config epoch to each node
    >>> Sending CLUSTER MEET messages to join the cluster
    Waiting for the cluster to join
    ..
    >>> Performing Cluster Check (using node 192.168.5.163:6379)
    M: fc0fc475d49bc02c53a1f82423c728b7621955fc 192.168.5.163:6379
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    M: 9afe793e4f1fec538521b2f153ca83044269d29c 192.168.5.164:6381
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
    S: b8a3b03d610eb41d2a813ddb1ebceae72cec4dc2 192.168.5.165:6384
       slots: (0 slots) slave
       replicates 9afe793e4f1fec538521b2f153ca83044269d29c
    M: c84ed4fd5b59bb285360fd2dd3df982e1b584777 192.168.5.165:6383
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
    S: 793007fbf806fa7a68944a76ced8f6757eb9a265 192.168.5.164:6382
       slots: (0 slots) slave
       replicates fc0fc475d49bc02c53a1f82423c728b7621955fc
    S: f4bd25ab6a7cdc0945e168939e89a2292d3f61b0 192.168.5.163:6380
       slots: (0 slots) slave
       replicates c84ed4fd5b59bb285360fd2dd3df982e1b584777
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.
    ```
3. 查看集群:
    ```bash
    # redis-cli -p 6379 -a 123456 -c
    127.0.0.1:6379> cluster nodes
    9afe793e4f1fec538521b2f153ca83044269d29c 192.168.5.164:6381@16381 master - 0 1657093201422 3 connected 5461-10922
    b8a3b03d610eb41d2a813ddb1ebceae72cec4dc2 192.168.5.165:6384@16384 slave 9afe793e4f1fec538521b2f153ca83044269d29c 0 1657093202428 3 connected
    c84ed4fd5b59bb285360fd2dd3df982e1b584777 192.168.5.165:6383@16383 master - 0 1657093200415 5 connected 10923-16383
    793007fbf806fa7a68944a76ced8f6757eb9a265 192.168.5.164:6382@16382 slave fc0fc475d49bc02c53a1f82423c728b7621955fc 0 1657093199409 1 connected
    f4bd25ab6a7cdc0945e168939e89a2292d3f61b0 192.168.5.163:6380@16380 slave c84ed4fd5b59bb285360fd2dd3df982e1b584777 0 1657093201000 5 connected
    fc0fc475d49bc02c53a1f82423c728b7621955fc 192.168.5.163:6379@16379 myself,master - 0 1657093202000 1 connected 0-5460
    ```
4. 操作示例:
    ```bash
    127.0.0.1:6379> set x 10
    -> Redirected to slot [16287] located at 192.168.5.165:6383
    OK
    192.168.5.165:6383> get x
    "10"
    ```
