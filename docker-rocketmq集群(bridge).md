# docker-rocketmq 集群 (bridge)

## 前言

- rocketmq 版本: ```apache/rocketmq:latest```

- 网络配置: 驱动类型为 bridge，名称为 rocketmq ，子网掩码为 ```192.168.4.0/24```，网关为 ```192.168.4.1```

- 容器与宿主机映射
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |mqnamesrv-1|192.168.4.10|9876:9876|192.168.204.107|/usr/local/docker/rocketmq/node/namesrv-1/logs:/home/rocketmq/logs|
    |mqnamesrv-2|192.168.4.11|9877:9876|192.168.204.107|/usr/local/docker/rocketmq/node/namesrv-2/logs:/home/rocketmq/logs|
    |mqbroker-a|192.168.4.20|10911:10911|192.168.204.107|/usr/local/docker/rocketmq/conf:/home/rocketmq/rocketmq-4.9.2/conf<br />/usr/local/docker/rocketmq/node/broker-a/logs:/home/rocketmq/logs<br />/usr/local/docker/rocketmq/node/broker-a/store:/home/rocketmq/store|
    |mqbroker-a-s|192.168.4.21|10912:10911|192.168.204.107|/usr/local/docker/rocketmq/conf:/home/rocketmq/rocketmq-4.9.2/conf<br />/usr/local/docker/rocketmq/node/broker-a-s/logs:/home/rocketmq/logs<br />/usr/local/docker/rocketmq/node/broker-a-s/store:/home/rocketmq/store|
    |mqbroker-b|192.168.4.22|10913:10911|192.168.204.107|/usr/local/docker/rocketmq/conf:/home/rocketmq/rocketmq-4.9.2/conf<br />/usr/local/docker/rocketmq/node/broker-b/logs:/home/rocketmq/logs<br />/usr/local/docker/rocketmq/node/broker-b/store:/home/rocketmq/store|
    |mqbroker-b-s|192.168.4.23|10914:10911|192.168.204.107|/usr/local/docker/rocketmq/conf:/home/rocketmq/rocketmq-4.9.2/conf<br />/usr/local/docker/rocketmq/node/broker-b-s/logs:/home/rocketmq/logs<br />/usr/local/docker/rocketmq/node/broker-b-s/store:/home/rocketmq/store|
    |mqdashboard|192.168.4.30|8080:8080|192.168.204.107|/usr/local/docker/rocketmq/dashboard/conf/application.properties:/application.properties<br />/usr/local/docker/rocketmq/dashboard/conf/users.properties:/tmp/rocketmq-console/data/users.properties|

- apache/rocketmq:latest 镜像配置文件 broker-a.properties
    ```properties
    # Licensed to the Apache Software Foundation (ASF) under one or more
    # contributor license agreements.  See the NOTICE file distributed with
    # this work for additional information regarding copyright ownership.
    # The ASF licenses this file to You under the Apache License, Version 2.0
    # (the "License"); you may not use this file except in compliance with
    # the License.  You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.
    brokerClusterName=DefaultCluster
    brokerName=broker-a
    brokerId=0
    deleteWhen=04
    fileReservedTime=48
    brokerRole=ASYNC_MASTER
    flushDiskType=ASYNC_FLUSH
    ```

## 拉取 rocketmq 镜像

### 拉取 rocketmq 镜像

```bash
# docker pull apache/rocketmq:latest
```

### 拉取 rocketmq-dashboard 镜像

```bash
# docker pull apacherocketmq/rocketmq-dashboard:latest
```

### 查看镜像

```bash
# docker images
REPOSITORY                          TAG       IMAGE ID       CREATED         SIZE
apache/rocketmq                     latest    2cee00481a8c   7 months ago    519MB
apacherocketmq/rocketmq-dashboard   latest    eae6c5db5d11   7 months ago    738MB
```

## 创建网络

```bash
# docker network create rocketmq --subnet 192.168.4.0/24 --gateway=192.168.4.1

# docker network ls
NETWORK ID     NAME                    DRIVER    SCOPE
1517e7770185   rocketmq                bridge    local

# docker network inspect rocketmq
[
    {
        "Name": "rocketmq",
        "Id": "1517e77701850895cd0d78a030c5e4101bdadb3db5b3922315a27a5844c6598d",
        "Created": "2022-06-06T01:41:39.734552136-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.4.0/24",
                    "Gateway": "192.168.4.1"
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

## 在宿主机上创建 rocketmq 目录

```bash
# docker run -d --name mqnamesrv --privileged=true -p 9876:9876 -e "MAX_POSSIBLE_HEAP=100000000" --network=rocketmq --ip 192.168.4.10 apache/rocketmq:latest sh mqnamesrv

# docker run -d --name mqbroker --privileged=true -p 10911:10911 -p 10909:10909  -e "NAMESRV_ADDR=192.168.4.10:9876" -e "MAX_POSSIBLE_HEAP=200000000" --network=rocketmq --ip 192.168.4.20 apache/rocketmq:latest sh mqbroker

# docker run -d --name mqdashboard --privileged=true -p 8080:8080 -e "NAMESRV_ADDR=192.168.4.10:9876" --network=rocketmq --ip 192.168.4.30 -t apacherocketmq/rocketmq-dashboard:latest

# mkdir -p /usr/local/docker/rocketmq

# > /usr/local/docker/rocketmq/create-node.sh

# vim /usr/local/docker/rocketmq/create-node.sh
```

```sh
#!/bin/sh
mkdir -p /usr/local/docker/rocketmq/node/{namesrv-1,namesrv-2,broker-a,broker-a-s,broker-b,broker-b-s}

##--------------------------------------------------------------------
## 拷贝容器目录到宿主机
##--------------------------------------------------------------------
docker cp mqnamesrv:/home/rocketmq/logs /usr/local/docker/rocketmq/node/namesrv-1
cp -r /usr/local/docker/rocketmq/node/namesrv-1/logs /usr/local/docker/rocketmq/node/namesrv-2
docker cp mqbroker:/home/rocketmq/rocketmq-4.9.2/conf /usr/local/docker/rocketmq

##--------------------------------------------------------------------
## 修改集群配置文件
##--------------------------------------------------------------------
cat << EOF >> /usr/local/docker/rocketmq/conf/2m-2s-async/broker-a.properties
namesrvAddr=192.168.4.10:9876;192.168.4.11:9876
listenPort=10911
EOF
cat << EOF >> /usr/local/docker/rocketmq/conf/2m-2s-async/broker-a-s.properties
namesrvAddr=192.168.4.10:9876;192.168.4.11:9876
listenPort=10911
EOF
cat << EOF >> /usr/local/docker/rocketmq/conf/2m-2s-async/broker-b.properties
namesrvAddr=192.168.4.10:9876;192.168.4.11:9876
listenPort=10911
EOF
cat << EOF >> /usr/local/docker/rocketmq/conf/2m-2s-async/broker-b-s.properties
namesrvAddr=192.168.4.10:9876;192.168.4.11:9876
listenPort=10911
EOF

##--------------------------------------------------------------------
## 拷贝容器目录到宿主机
##--------------------------------------------------------------------
docker cp mqbroker:/home/rocketmq/logs /usr/local/docker/rocketmq/node/broker-a
docker cp mqbroker:/home/rocketmq/store /usr/local/docker/rocketmq/node/broker-a
chmod -R 777 /usr/local/docker/rocketmq/node/broker-a
cp -r /usr/local/docker/rocketmq/node/broker-a/logs /usr/local/docker/rocketmq/node/broker-a-s
cp -r /usr/local/docker/rocketmq/node/broker-a/store /usr/local/docker/rocketmq/node/broker-a-s
chmod -R 777 /usr/local/docker/rocketmq/node/broker-a-s
cp -r /usr/local/docker/rocketmq/node/broker-a/logs /usr/local/docker/rocketmq/node/broker-b
cp -r /usr/local/docker/rocketmq/node/broker-a/store /usr/local/docker/rocketmq/node/broker-b
chmod -R 777 /usr/local/docker/rocketmq/node/broker-b
cp -r /usr/local/docker/rocketmq/node/broker-a/logs /usr/local/docker/rocketmq/node/broker-b-s
cp -r /usr/local/docker/rocketmq/node/broker-a/store /usr/local/docker/rocketmq/node/broker-b-s
chmod -R 777 /usr/local/docker/rocketmq/node/broker-b-s

mkdir -p /usr/local/docker/rocketmq/dashboard/conf

##--------------------------------------------------------------------
## 拷贝 dashboard 容器目录到宿主机
##--------------------------------------------------------------------
docker cp mqdashboard:/root/logs /usr/local/docker/rocketmq/dashboard

##--------------------------------------------------------------------
## 修改 dashboard 配置文件
##--------------------------------------------------------------------
> /usr/local/docker/rocketmq/dashboard/conf/application.properties
cat << EOF >> /usr/local/docker/rocketmq/dashboard/conf/application.properties
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

server.address=0.0.0.0
server.port=8080

### SSL setting
#server.ssl.key-store=classpath:rmqcngkeystore.jks
#server.ssl.key-store-password=rocketmq
#server.ssl.keyStoreType=PKCS12
#server.ssl.keyAlias=rmqcngkey

#spring.application.index=true
spring.application.name=rocketmq-dashboard
spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
spring.http.encoding.force=true
logging.level.root=INFO
logging.config=classpath:logback.xml
#if this value is empty,use env value rocketmq.config.namesrvAddr  NAMESRV_ADDR | now, you can set it in ops page.default localhost:9876
rocketmq.config.namesrvAddr=192.168.4.10:9876;192.168.4.11:9876
#if you use rocketmq version < 3.5.8, rocketmq.config.isVIPChannel should be false.default true
rocketmq.config.isVIPChannel=
#timeout for mqadminExt, default 5000ms
rocketmq.config.timeoutMillis=
#rocketmq-console's data path:dashboard/monitor
rocketmq.config.dataPath=/tmp/rocketmq-console/data
#set it false if you don't want use dashboard.default true
rocketmq.config.enableDashBoardCollect=true
#set the message track trace topic if you don't want use the default one
rocketmq.config.msgTrackTopicName=
rocketmq.config.ticketKey=ticket

#Must create userInfo file: \${rocketmq.config.dataPath}/users.properties if the login is required
rocketmq.config.loginRequired=true

#set the accessKey and secretKey if you used acl
#rocketmq.config.accessKey=
#rocketmq.config.secretKey=
rocketmq.config.useTLS=false
EOF

##--------------------------------------------------------------------
## 修改 dashboard 认证文件
##--------------------------------------------------------------------
> /usr/local/docker/rocketmq/dashboard/conf/users.properties
cat << EOF >> /usr/local/docker/rocketmq/dashboard/conf/users.properties
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

# This file supports hot change, any change will be auto-reloaded without Dashboard restarting.
# Format: a user per line, username=password[,N] #N is optional, 0 (Normal User); 1 (Admin)

# Define Admin
admin=admin,1

# Define Users
user=user
EOF
```

```bash
# chmod +x /usr/local/docker/rocketmq/create-node.sh

# /usr/local/docker/rocketmq/create-node.sh

# ls /usr/local/docker/rocketmq
conf  create-node.sh  dashboard  node

# ls /usr/local/docker/rocketmq/node
broker-a  broker-a-s  broker-b  broker-b-s  namesrv-1  namesrv-2

# ls /usr/local/docker/rocketmq/conf
2m-2s-async  2m-noslave   dledger             logback_namesrv.xml  plain_acl.yml
2m-2s-sync   broker.conf  logback_broker.xml  logback_tools.xml    tools.yml
```

注:

1. 从容器里拷贝配置文件到宿主机
2. 开启控制台 dashboard 密码验证: ```admin:admin```，测试验证:
    ```bash
    # docker run -d --name mqdashboard --privileged=true -p 8080:8080 -e "NAMESRV_ADDR=192.168.4.10:9876" --network=rocketmq --ip 192.168.4.30 -v /usr/local/docker/rocketmq/dashboard/conf/application.properties:/application.properties -v /usr/local/docker/rocketmq/dashboard/conf/users.properties:/tmp/rocketmq-console/data/users.properties -t apacherocketmq/rocketmq-dashboard:latest
    ```

## 配置 docker-compose.yml

```bash
# > /usr/local/docker/rocketmq/docker-compose.yml

# vim /usr/local/docker/rocketmq/docker-compose.yml
```

```yml
version: '3.9'

services:
 mqnamesrv-1:
  image: apache/rocketmq:latest
  container_name: mqnamesrv-1
  restart: always
  volumes:
   - /usr/local/docker/rocketmq/node/namesrv-1/logs:/home/rocketmq/logs
  command: sh mqnamesrv
  ports:
   - 9876:9876
  networks:
    rocketmq:
      ipv4_address: 192.168.4.10
  healthcheck:
    test: "netstat -apn | grep 9876 | wc -l"
    timeout: 10s
    interval: 10s
    retries: 10
 mqnamesrv-2:
  image: apache/rocketmq:latest
  container_name: mqnamesrv-2
  restart: always
  volumes:
   - /usr/local/docker/rocketmq/node/namesrv-2/logs:/home/rocketmq/logs
  command: sh mqnamesrv
  ports:
   - 9877:9876
  networks:
    rocketmq:
      ipv4_address: 192.168.4.11
  healthcheck:
    test: "netstat -apn | grep 9876 | wc -l"
    timeout: 10s
    interval: 10s
    retries: 10
 mqbroker-a:
  depends_on:
    mqnamesrv-1:
      condition: service_healthy
    mqnamesrv-2:
      condition: service_healthy
  image: apache/rocketmq:latest
  container_name: mqbroker-a
  restart: always
  volumes:
   - /usr/local/docker/rocketmq/conf:/home/rocketmq/rocketmq-4.9.2/conf
   - /usr/local/docker/rocketmq/node/broker-a/logs:/home/rocketmq/logs
   - /usr/local/docker/rocketmq/node/broker-a/store:/home/rocketmq/store
  command: sh mqbroker -c /home/rocketmq/rocketmq-4.9.2/conf/2m-2s-async/broker-a.properties
  ports:
   - 10911:10911
  networks:
    rocketmq:
      ipv4_address: 192.168.4.20
  healthcheck:
    test: "netstat -apn | grep 10911 | wc -l"
    timeout: 10s
    interval: 10s
    retries: 10
 mqbroker-a-s:
  depends_on:
    mqnamesrv-1:
      condition: service_healthy
    mqnamesrv-2:
      condition: service_healthy
  image: apache/rocketmq:latest
  container_name: mqbroker-a-s
  restart: always
  volumes:
   - /usr/local/docker/rocketmq/conf:/home/rocketmq/rocketmq-4.9.2/conf
   - /usr/local/docker/rocketmq/node/broker-a-s/logs:/home/rocketmq/logs
   - /usr/local/docker/rocketmq/node/broker-a-s/store:/home/rocketmq/store
  command: sh mqbroker -c /home/rocketmq/rocketmq-4.9.2/conf/2m-2s-async/broker-a-s.properties
  ports:
   - 10912:10911
  networks:
    rocketmq:
      ipv4_address: 192.168.4.21
  healthcheck:
    test: "netstat -apn | grep 10911 | wc -l"
    timeout: 10s
    interval: 10s
    retries: 10
 mqbroker-b:
  depends_on:
    mqnamesrv-1:
      condition: service_healthy
    mqnamesrv-2:
      condition: service_healthy
  image: apache/rocketmq:latest
  container_name: mqbroker-b
  restart: always
  volumes:
   - /usr/local/docker/rocketmq/conf:/home/rocketmq/rocketmq-4.9.2/conf
   - /usr/local/docker/rocketmq/node/broker-b/logs:/home/rocketmq/logs
   - /usr/local/docker/rocketmq/node/broker-b/store:/home/rocketmq/store
  command: sh mqbroker -c /home/rocketmq/rocketmq-4.9.2/conf/2m-2s-async/broker-b.properties
  ports:
   - 10913:10911
  networks:
    rocketmq:
      ipv4_address: 192.168.4.22
  healthcheck:
    test: "netstat -apn | grep 10911 | wc -l"
    timeout: 10s
    interval: 10s
    retries: 10
 mqbroker-b-s:
  depends_on:
    mqnamesrv-1:
      condition: service_healthy
    mqnamesrv-2:
      condition: service_healthy
  image: apache/rocketmq:latest
  container_name: mqbroker-b-s
  restart: always
  volumes:
   - /usr/local/docker/rocketmq/conf:/home/rocketmq/rocketmq-4.9.2/conf
   - /usr/local/docker/rocketmq/node/broker-b-s/logs:/home/rocketmq/logs
   - /usr/local/docker/rocketmq/node/broker-b-s/store:/home/rocketmq/store
  command: sh mqbroker -c /home/rocketmq/rocketmq-4.9.2/conf/2m-2s-async/broker-b-s.properties
  ports:
   - 10914:10911
  networks:
    rocketmq:
      ipv4_address: 192.168.4.23
  healthcheck:
    test: "netstat -apn | grep 10911 | wc -l"
    timeout: 10s
    interval: 10s
    retries: 10
 mqdashboard:
  depends_on:
    mqnamesrv-1:
      condition: service_healthy
    mqnamesrv-2:
      condition: service_healthy
    mqbroker-a:
      condition: service_healthy
    mqbroker-a-s:
      condition: service_healthy
    mqbroker-b:
      condition: service_healthy
    mqbroker-b-s:
      condition: service_healthy
  image: apacherocketmq/rocketmq-dashboard:latest
  container_name: mqdashboard
  restart: always
  volumes:
   - /usr/local/docker/rocketmq/dashboard/conf/application.properties:/application.properties
   - /usr/local/docker/rocketmq/dashboard/conf/users.properties:/tmp/rocketmq-console/data/users.properties
  ports:
   - 8080:8080
  networks:
    rocketmq:
      ipv4_address: 192.168.4.30
networks:
    rocketmq:
      name: rocketmq
```

## 启动 docker-compose

```bash
# docker stop mqnamesrv mqbroker mqdashboard

# docker rm mqnamesrv mqbroker mqdashboard

# docker-compose -f /usr/local/docker/rocketmq/docker-compose.yml up -d
[+] Running 7/7
 ⠿ Container mqnamesrv-2   Healthy                                                                              18.5s
 ⠿ Container mqnamesrv-1   Healthy                                                                              18.5s
 ⠿ Container mqbroker-a    Healthy                                                                              28.0s
 ⠿ Container mqbroker-b    Healthy                                                                              30.6s
 ⠿ Container mqbroker-b-s  Healthy                                                                              29.0s
 ⠿ Container mqbroker-a-s  Healthy                                                                              29.0s
 ⠿ Container mqdashboard   Started                                                                             298.7s

# docker ps
CONTAINER ID   IMAGE                                      COMMAND                  CREATED         STATUS                   PORTS                                                                                            NAMES
15e0c81beb28   apacherocketmq/rocketmq-dashboard:latest   "sh -c 'java $JAVA_O…"   6 minutes ago   Up About a minute        0.0.0.0:8080->8080/tcp, :::8080->8080/tcp                                                        mqdashboard
1087c228dc53   apache/rocketmq:latest                     "sh mqbroker -c /hom…"   6 minutes ago   Up 5 minutes (healthy)   9876/tcp, 10909/tcp, 10912/tcp, 0.0.0.0:10911->10911/tcp, :::10911->10911/tcp                    mqbroker-a
62aa40e27e62   apache/rocketmq:latest                     "sh mqbroker -c /hom…"   6 minutes ago   Up 5 minutes (healthy)   9876/tcp, 10909/tcp, 10912/tcp, 0.0.0.0:10913->10911/tcp, :::10913->10911/tcp                    mqbroker-b
2319df209fef   apache/rocketmq:latest                     "sh mqbroker -c /hom…"   6 minutes ago   Up 5 minutes (healthy)   9876/tcp, 10909/tcp, 10912/tcp, 0.0.0.0:10912->10911/tcp, :::10912->10911/tcp                    mqbroker-a-s
865f2c383087   apache/rocketmq:latest                     "sh mqbroker -c /hom…"   6 minutes ago   Up 5 minutes (healthy)   9876/tcp, 10909/tcp, 10912/tcp, 0.0.0.0:10914->10911/tcp, :::10914->10911/tcp                    mqbroker-b-s
66bbde187a4c   apache/rocketmq:latest                     "sh mqnamesrv"           6 minutes ago   Up 5 minutes (healthy)   10909/tcp, 0.0.0.0:9876->9876/tcp, :::9876->9876/tcp, 10911-10912/tcp                            mqnamesrv-1
5634ec3aaa98   apache/rocketmq:latest                     "sh mqnamesrv"           6 minutes ago   Up 5 minutes (healthy)   10909/tcp, 10911-10912/tcp, 0.0.0.0:9877->9876/tcp, :::9877->9876/tcp                            mqnamesrv-2
```

## 查看网络

```bash
# docker network inspect rocketmq
[
    {
        "Name": "rocketmq",
        "Id": "1517e77701850895cd0d78a030c5e4101bdadb3db5b3922315a27a5844c6598d",
        "Created": "2022-06-06T01:41:39.734552136-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.4.0/24",
                    "Gateway": "192.168.4.1"
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
            "25f533d8c186cbafa8d52440e975f4fc4c03ff654ab45b14135bdd40c5a08dc7": {
                "Name": "mqbroker-b-s",
                "EndpointID": "b9cf55dd997e10fa93c0d2e9de8f3a2e6e28c2767b0cb02f404647bd64e32a73",
                "MacAddress": "02:42:c0:a8:04:17",
                "IPv4Address": "192.168.4.23/24",
                "IPv6Address": ""
            },
            "3aa3b7561cb93dfe65d3361ccd3f0912eec11a665e6b72c8631108a7bf7be580": {
                "Name": "mqbroker-b",
                "EndpointID": "c3977daf450d7941f6e90557f6c785b38185d423ad383722ae6e3e01d4efe55f",
                "MacAddress": "02:42:c0:a8:04:16",
                "IPv4Address": "192.168.4.22/24",
                "IPv6Address": ""
            },
            "477dbe3e59b6f8f03e634a4194fbf03e2ef7818118e06d2a8b3f0c3ab39bdf72": {
                "Name": "mqbroker-a",
                "EndpointID": "5c43fe7834bd53e8a4acdc6a175948653250f0fe9b7d144c8e15b7d664909b12",
                "MacAddress": "02:42:c0:a8:04:14",
                "IPv4Address": "192.168.4.20/24",
                "IPv6Address": ""
            },
            "ab23b41c93e010696d8159fc7b9c365fc68a8a0c54fda3001109ac496488be31": {
                "Name": "mqnamesrv-2",
                "EndpointID": "fa01c50877bb106fa8ddc28bed4a98d4fa997837869b7540168e3bbf3e69bc19",
                "MacAddress": "02:42:c0:a8:04:0b",
                "IPv4Address": "192.168.4.11/24",
                "IPv6Address": ""
            },
            "cef1077384558e315e9f738a613ec5d38024f9ce1829a35dd6d25255233f28ab": {
                "Name": "mqnamesrv-1",
                "EndpointID": "c44cc476c5b10c0a772a1990d2125aed15a8221f03dbd81773523e9c74e9c00b",
                "MacAddress": "02:42:c0:a8:04:0a",
                "IPv4Address": "192.168.4.10/24",
                "IPv6Address": ""
            },
            "d8504763ff74cab7d6c021d969a5c439095514c2e07eb8d267b962868bfba656": {
                "Name": "mqbroker-a-s",
                "EndpointID": "286cb839e368e21a0a82eb39fd52e89d9510881652717b06e82b0156e072e57d",
                "MacAddress": "02:42:c0:a8:04:15",
                "IPv4Address": "192.168.4.21/24",
                "IPv6Address": ""
            },
            "ff4737d454dff9ac2076515bf7fb0718805472f1697c7a131055883a5919568d": {
                "Name": "mqdashboard",
                "EndpointID": "8fc6b2daa5325d2e7960f7d6634092166f77ba0eebc8076924bf39cc760cff4f",
                "MacAddress": "02:42:c0:a8:04:1e",
                "IPv4Address": "192.168.4.30/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

## 访问 dashboard

```
http://192.168.204.107:8080/
```
