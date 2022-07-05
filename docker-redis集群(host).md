# docker-redis 集群(host)

## 前言

- redis版本: ```redis:latest```

- 网络配置: 驱动类型为 host

- 启动六个 redis
    |容器名称|宿主机IP|宿主机的端口|挂载宿主机的配置文件和数据文件|
    |--|--|--|--|
    |redis-1|192.168.5.163|6379|/usr/local/docker/redis/node-1/conf/redis.conf:/etc/redis/redis.conf<br />/usr/local/docker/redis/node-1/data\:/data<br />/usr/local/docker/redis/node-1/log:/var/log/redis|
    |redis-2|192.168.5.163|6380|/usr/local/docker/redis/node-2/conf/redis.conf:/etc/redis/redis.conf<br />/usr/local/docker/redis/node-2/data\:/data<br />/usr/local/docker/redis/node-2/log:/var/log/redis|
    |redis-3|192.168.5.164|6379|/usr/local/docker/redis/node-3/conf/redis.conf:/etc/redis/redis.conf<br />/usr/local/docker/redis/node-3/data\:/data<br />/usr/local/docker/redis/node-3/log:/var/log/redis|
    |redis-4|192.168.5.164|6380|/usr/local/docker/redis/node-4/conf/redis.conf:/etc/redis/redis.conf<br />/usr/local/docker/redis/node-4/data\:/data<br />/usr/local/docker/redis/node-4/log:/var/log/redis|
    |redis-5|192.168.5.165|6379|/usr/local/docker/redis/node-5/conf/redis.conf:/etc/redis/redis.conf<br />/usr/local/docker/redis/node-5/data\:/data<br />/usr/local/docker/redis/node-5/log:/var/log/redis|
    |redis-6|192.168.5.165|6380|/usr/local/docker/redis/node-6/conf/redis.conf:/etc/redis/redis.conf<br />/usr/local/docker/redis/node-6/data\:/data<br />/usr/local/docker/redis/node-6/log:/var/log/redis|

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

### 在宿主机 192.168.5.163 创建 redis 目录

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```sh
#!/bin/sh

mkdir -p /usr/local/docker/redis/{node-1/{conf,data,log},node-2/{conf,data,log}}

chmod 777 /usr/local/docker/redis/{node-1/log,node-2/log}

if [ ! -f /usr/local/docker/redis/node-1/conf/redis.conf ]; then
    wget -P /usr/local/docker/redis https://download.redis.io/releases/redis-6.2.6.tar.gz
    tar -xzvf /usr/local/docker/redis/redis-6.2.6.tar.gz -C /usr/local/docker/redis
    cp /usr/local/docker/redis/redis-6.2.6/redis.conf /usr/local/docker/redis/node-1/conf
    rm -rf /usr/local/docker/redis/redis-6.2.6*
fi
if [ ! -f /usr/local/docker/redis/node-2/conf/redis.conf ]; then
    cp /usr/local/docker/redis/node-1/conf/redis.conf /usr/local/docker/redis/node-2/conf
fi
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
└── node-2
    ├── conf
    │   └── redis.conf
    ├── data
    └── log
```

### 在宿主机 192.168.5.164 创建 redis 目录

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```sh
#!/bin/sh

mkdir -p /usr/local/docker/redis/{node-3/{conf,data,log},node-4/{conf,data,log}}

chmod 777 /usr/local/docker/redis/{node-3/log,node-4/log}

if [ ! -f /usr/local/docker/redis/node-3/conf/redis.conf ]; then
    wget -P /usr/local/docker/redis https://download.redis.io/releases/redis-6.2.6.tar.gz
    tar -xzvf /usr/local/docker/redis/redis-6.2.6.tar.gz -C /usr/local/docker/redis
    cp /usr/local/docker/redis/redis-6.2.6/redis.conf /usr/local/docker/redis/node-3/conf
    rm -rf /usr/local/docker/redis/redis-6.2.6*
fi
if [ ! -f /usr/local/docker/redis/node-4/conf/redis.conf ]; then
    cp /usr/local/docker/redis/node-3/conf/redis.conf /usr/local/docker/redis/node-4/conf
fi
```

```bash
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/redis/
/usr/local/docker/redis/
├── node-3
│   ├── conf
│   │   └── redis.conf
│   ├── data
│   └── log
└── node-4
    ├── conf
    │   └── redis.conf
    ├── data
    └── log
```

### 在宿主机 192.168.5.165 创建 redis 目录

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```sh
#!/bin/sh

mkdir -p /usr/local/docker/redis/{node-5/{conf,data,log},node-6/{conf,data,log}}

chmod 777 /usr/local/docker/redis/{node-5/log,node-6/log}

if [ ! -f /usr/local/docker/redis/node-5/conf/redis.conf ]; then
    wget -P /usr/local/docker/redis https://download.redis.io/releases/redis-6.2.6.tar.gz
    tar -xzvf /usr/local/docker/redis/redis-6.2.6.tar.gz -C /usr/local/docker/redis
    cp /usr/local/docker/redis/redis-6.2.6/redis.conf /usr/local/docker/redis/node-5/conf
    rm -rf /usr/local/docker/redis/redis-6.2.6*
fi
if [ ! -f /usr/local/docker/redis/node-6/conf/redis.conf ]; then
    cp /usr/local/docker/redis/node-5/conf/redis.conf /usr/local/docker/redis/node-6/conf
fi
```

```bash
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/redis/
/usr/local/docker/redis/
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

## 配置 docker-compose.yml

### 在宿主机 192.168.5.163 配置 docker-compose.yml

```bash
# vim /usr/local/docker/docker-compose.yml
```

```yml
version: '3.9'

services:
 redis-1:
  image: redis:latest
  container_name: redis-1
  restart: always
  command: redis-server /etc/redis/redis.conf --port 6379 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-1/conf/redis.conf:/etc/redis/redis.conf
    - /usr/local/docker/redis/node-1/data:/data
    - /usr/local/docker/redis/node-1/log:/var/log/redis
  network_mode: host
 redis-2:
  image: redis:latest
  container_name: redis-2
  restart: always
  command: redis-server /etc/redis/redis.conf --port 6380 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-2/conf/redis.conf:/etc/redis/redis.conf
    - /usr/local/docker/redis/node-2/data:/data
    - /usr/local/docker/redis/node-2/log:/var/log/redis
  network_mode: host
```

### 在宿主机 192.168.5.164 配置 docker-compose.yml

```bash
# vim /usr/local/docker/docker-compose.yml
```

```yml
version: '3.9'

services:
 redis-3:
  image: redis:latest
  container_name: redis-3
  restart: always
  command: redis-server /etc/redis/redis.conf --port 6379 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-3/conf/redis.conf:/etc/redis/redis.conf
    - /usr/local/docker/redis/node-3/data:/data
    - /usr/local/docker/redis/node-3/log:/var/log/redis
  network_mode: host
 redis-4:
  image: redis:latest
  container_name: redis-4
  restart: always
  command: redis-server /etc/redis/redis.conf --port 6380 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-4/conf/redis.conf:/etc/redis/redis.conf
    - /usr/local/docker/redis/node-4/data:/data
    - /usr/local/docker/redis/node-4/log:/var/log/redis
  network_mode: host
```

### 在宿主机 192.168.5.165 配置 docker-compose.yml

```bash
# vim /usr/local/docker/docker-compose.yml
```

```yml
version: '3.9'

services:
 redis-5:
  image: redis:latest
  container_name: redis-5
  restart: always
  command: redis-server /etc/redis/redis.conf --port 6379 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-5/conf/redis.conf:/etc/redis/redis.conf
    - /usr/local/docker/redis/node-5/data:/data
    - /usr/local/docker/redis/node-5/log:/var/log/redis
  network_mode: host
 redis-6:
  image: redis:latest
  container_name: redis-6
  restart: always
  command: redis-server /etc/redis/redis.conf --port 6380 --requirepass 123456 --masterauth 123456 --bind 0.0.0.0 --cluster-enabled yes --appendonly yes --logfile /var/log/redis/redis.log
  volumes:
    - /usr/local/docker/redis/node-6/conf/redis.conf:/etc/redis/redis.conf
    - /usr/local/docker/redis/node-6/data:/data
    - /usr/local/docker/redis/node-6/log:/var/log/redis
  network_mode: host
```

## 启动 docker-compose

### 启动宿主机 192.168.5.163 的 redis 服务

```bash
# docker-compose -f /usr/local/docker/docker-compose.yml up -d
[+] Running 2/2
 ⠿ Container redis-2  Started                                                                                    0.2s
 ⠿ Container redis-1  Started                                                                                    0.2s

# docker ps
CONTAINER ID   IMAGE          COMMAND                   CREATED          STATUS          PORTS     NAMES
ed511e72ba6b   redis:latest   "docker-entrypoint.s…"   15 seconds ago   Up 15 seconds             redis-1
78c4aa66896c   redis:latest   "docker-entrypoint.s…"   15 seconds ago   Up 15 seconds             redis-2
```

### 启动宿主机 192.168.5.164 的 redis 服务

```bash
# docker-compose -f /usr/local/docker/docker-compose.yml up -d
[+] Running 2/2
 ⠿ Container redis-3  Started                                                                                    0.2s
 ⠿ Container redis-4  Started                                                                                    0.2s

# docker ps
CONTAINER ID   IMAGE          COMMAND                   CREATED          STATUS          PORTS     NAMES
ed511e72ba6b   redis:latest   "docker-entrypoint.s…"   15 seconds ago   Up 15 seconds             redis-4
78c4aa66896c   redis:latest   "docker-entrypoint.s…"   15 seconds ago   Up 15 seconds             redis-3
```

### 启动宿主机 192.168.5.165 的 redis 服务

```bash
# docker-compose -f /usr/local/docker/docker-compose.yml up -d
[+] Running 2/2
 ⠿ Container redis-6  Started                                                                                    0.2s
 ⠿ Container redis-5  Started                                                                                    0.2s

# docker ps
CONTAINER ID   IMAGE          COMMAND                   CREATED          STATUS          PORTS     NAMES
ed511e72ba6b   redis:latest   "docker-entrypoint.s…"   15 seconds ago   Up 15 seconds             redis-5
78c4aa66896c   redis:latest   "docker-entrypoint.s…"   15 seconds ago   Up 15 seconds             redis-6
```

## 创建集群

1. 进入容器:
    ```bash
    # docker exec -it redis-1 /bin/sh
    ```
2. 进入容器之后，执行创建集群的命令，提示输入的时候，输入 ```yes```:
    ```bash
    # redis-cli -a 123456 --cluster create 192.168.5.163:6379 192.168.5.163:6380 192.168.5.164:6379 192.168.5.164:6380 192.168.5.165:6379 192.168.5.165:6380 --cluster-replicas 1
    Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
    >>> Performing hash slots allocation on 6 nodes...
    Master[0] -> Slots 0 - 5460
    Master[1] -> Slots 5461 - 10922
    Master[2] -> Slots 10923 - 16383
    Adding replica 192.168.5.164:6380 to 192.168.5.163:6379
    Adding replica 192.168.5.165:6380 to 192.168.5.164:6379
    Adding replica 192.168.5.163:6380 to 192.168.5.165:6379
    M: 03039dc513c46055fbb9ace9898a679b41e538fb 192.168.5.163:6379
       slots:[0-5460] (5461 slots) master
    S: f7a242f6fbb94672485d6cdf4bb907188ccb56b1 192.168.5.163:6380
       replicates 5cb08f28f5a10dea69d8fb9fc50c6a67f64d8a39
    M: 4fe0aaf8905d197956717c217b78f9d6f242d781 192.168.5.164:6379
       slots:[5461-10922] (5462 slots) master
    S: 0b941158b5a1479aba8d7250673791672acf4067 192.168.5.164:6380
       replicates 03039dc513c46055fbb9ace9898a679b41e538fb
    M: 5cb08f28f5a10dea69d8fb9fc50c6a67f64d8a39 192.168.5.165:6379
       slots:[10923-16383] (5461 slots) master
    S: e60ea1663ce81c92d7f8a1161e85857e8aaf648c 192.168.5.165:6380
       replicates 4fe0aaf8905d197956717c217b78f9d6f242d781
    Can I set the above configuration? (type 'yes' to accept): yes
    >>> Nodes configuration updated
    >>> Assign a different config epoch to each node
    >>> Sending CLUSTER MEET messages to join the cluster
    Waiting for the cluster to join

    >>> Performing Cluster Check (using node 192.168.5.163:6379)
    M: 03039dc513c46055fbb9ace9898a679b41e538fb 192.168.5.163:6379
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    M: 5cb08f28f5a10dea69d8fb9fc50c6a67f64d8a39 192.168.5.165:6379
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
    S: f7a242f6fbb94672485d6cdf4bb907188ccb56b1 192.168.5.163:6380
       slots: (0 slots) slave
       replicates 5cb08f28f5a10dea69d8fb9fc50c6a67f64d8a39
    M: 4fe0aaf8905d197956717c217b78f9d6f242d781 192.168.5.164:6379
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
    S: 0b941158b5a1479aba8d7250673791672acf4067 192.168.5.164:6380
       slots: (0 slots) slave
       replicates 03039dc513c46055fbb9ace9898a679b41e538fb
    S: e60ea1663ce81c92d7f8a1161e85857e8aaf648c 192.168.5.165:6380
       slots: (0 slots) slave
       replicates 4fe0aaf8905d197956717c217b78f9d6f242d781
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.
    ```
3. 查看集群:
    ```bash
    # redis-cli -p 6379 -a 123456 -c
    127.0.0.1:6379> cluster nodes
    5cb08f28f5a10dea69d8fb9fc50c6a67f64d8a39 192.168.5.165:6379@16379 master - 0 1656929782000 5 connected 10923-16383
    03039dc513c46055fbb9ace9898a679b41e538fb 192.168.5.163:6379@16379 myself,master - 0 1656929781000 1 connected 0-5460
    f7a242f6fbb94672485d6cdf4bb907188ccb56b1 192.168.5.163:6380@16380 slave 5cb08f28f5a10dea69d8fb9fc50c6a67f64d8a39 0 1656929781000 5 connected
    4fe0aaf8905d197956717c217b78f9d6f242d781 192.168.5.164:6379@16379 master - 0 1656929781511 3 connected 5461-10922
    0b941158b5a1479aba8d7250673791672acf4067 192.168.5.164:6380@16380 slave 03039dc513c46055fbb9ace9898a679b41e538fb 0 1656929782518 1 connected
    e60ea1663ce81c92d7f8a1161e85857e8aaf648c 192.168.5.165:6380@16380 slave 4fe0aaf8905d197956717c217b78f9d6f242d781 0 1656929781000 3 connected
    ```
4. 操作示例:
    ```bash
    127.0.0.1:6379> set x 100
    -> Redirected to slot [16287] located at 192.168.5.165:6379
    OK
    192.168.5.165:6379> get x
    "100"
    ```
