# docker-rocketmq 集群 (host)

## 前言

- rocketmq 版本: ```apache/rocketmq:latest```

- 网络配置: 驱动类型为 overlay

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- 容器与宿主机映射
    |容器名称|宿主机IP|端口映射(宿主机端口:容器端口)|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|
    |mqnamesrv_1|192.168.5.163|9876:9876|/usr/local/docker/rocketmq/namesrv-1/logs:/home/rocketmq/logs|
    |mqnamesrv_2|192.168.5.164|9877:9876|/usr/local/docker/rocketmq/namesrv-2/logs:/home/rocketmq/logs|
    |mqbroker_a|192.168.5.164|10909:10909|/usr/local/docker/rocketmq/broker-a/conf:/home/rocketmq/rocketmq-4.9.2/conf<br />/usr/local/docker/rocketmq/broker-a/logs:/home/rocketmq/logs<br />/usr/local/docker/rocketmq/broker-a/store:/home/rocketmq/store|
    |mqbroker_a_s|192.168.5.165|10910:10909|/usr/local/docker/rocketmq/broker-a-s/conf:/home/rocketmq/rocketmq-4.9.2/conf<br />/usr/local/docker/rocketmq/broker-a-s/logs:/home/rocketmq/logs<br />/usr/local/docker/rocketmq/broker-a-s/store:/home/rocketmq/store|
    |mqbroker_b|192.168.5.165|10911:10909|/usr/local/docker/rocketmq/broker-b/conf:/home/rocketmq/rocketmq-4.9.2/conf<br />/usr/local/docker/rocketmq/broker-b/logs:/home/rocketmq/logs<br />/usr/local/docker/rocketmq/broker-b/store:/home/rocketmq/store|
    |mqbroker_b_s192.168.5.164|10912:10909|/usr/local/docker/rocketmq/broker-b-s/conf:/home/rocketmq/rocketmq-4.9.2/conf<br />/usr/local/docker/rocketmq/broker-b-s/logs:/home/rocketmq/logs<br />/usr/local/docker/rocketmq/broker-b-s/store:/home/rocketmq/store|
    |mqdashboard|192.168.5.163|8080:8080|/usr/local/docker/rocketmq/dashboard/conf/application.properties:/application.properties<br />/usr/local/docker/rocketmq/dashboard/conf/users.properties:/tmp/rocketmq-console/data/users.properties|

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
# docker images | grep rocketmq
REPOSITORY                          TAG       IMAGE ID       CREATED         SIZE
apache/rocketmq                     latest    2cee00481a8c   7 months ago    519MB
apacherocketmq/rocketmq-dashboard   latest    eae6c5db5d11   7 months ago    738MB
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

mkdir -p /usr/local/docker/rocketmq/{namesrv-1,namesrv-2,broker-a,broker-a-s,broker-b,broker-b-s}
mkdir -p /usr/local/docker/rocketmq/dashboard/conf

if [ ! -d /usr/local/docker/rocketmq/namesrv-1/logs ]; then
    docker network create rocketmq --subnet 192.168.4.0/24 --gateway=192.168.4.1
    docker run -d --name mqnamesrv --privileged=true -p 9876:9876 -e "MAX_POSSIBLE_HEAP=100000000" --network=rocketmq --ip 192.168.4.10 apache/rocketmq:latest sh mqnamesrv
    docker run -d --name mqbroker --privileged=true -p 10911:10911 -p 10909:10909  -e "NAMESRV_ADDR=192.168.4.10:9876" -e "MAX_POSSIBLE_HEAP=200000000" --network=rocketmq --ip 192.168.4.20 apache/rocketmq:latest sh mqbroker
    docker run -d --name mqdashboard --privileged=true -p 8080:8080 -e "NAMESRV_ADDR=192.168.4.10:9876" --network=rocketmq --ip 192.168.4.30 -t apacherocketmq/rocketmq-dashboard:latest
    sleep 15s
	  
    ##--------------------------------------------------------------------
    ## 拷贝容器目录到宿主机
    ##--------------------------------------------------------------------
    docker cp mqnamesrv:/home/rocketmq/logs /usr/local/docker/rocketmq/namesrv-1
    docker cp mqbroker:/home/rocketmq/rocketmq-4.9.2/conf /usr/local/docker/rocketmq/broker-a
    docker cp mqbroker:/home/rocketmq/logs /usr/local/docker/rocketmq/broker-a
    docker cp mqbroker:/home/rocketmq/store /usr/local/docker/rocketmq/broker-a
    docker cp mqdashboard:/root/logs /usr/local/docker/rocketmq/dashboard
	
    docker stop mqdashboard mqbroker mqnamesrv
    docker rm mqdashboard mqbroker mqnamesrv
    docker network rm rocketmq
fi

if [ ! -f /usr/local/docker/rocketmq/broker-a/wait-for-it.sh ]; then
    wget -P /usr/local/docker/rocketmq/broker-a https://gitee.com/sunny906/docker/raw/master/wait-for/wait-for-it.sh
	chmod +x /usr/local/docker/rocketmq/broker-a/wait-for-it.sh
fi
cp /usr/local/docker/rocketmq/broker-a/wait-for-it.sh /usr/local/docker/rocketmq/broker-a-s
cp /usr/local/docker/rocketmq/broker-a/wait-for-it.sh /usr/local/docker/rocketmq/broker-b
cp /usr/local/docker/rocketmq/broker-a/wait-for-it.sh /usr/local/docker/rocketmq/broker-b-s
cp /usr/local/docker/rocketmq/broker-a/wait-for-it.sh /usr/local/docker/rocketmq/dashboard

##--------------------------------------------------------------------
## 修改集群配置文件
##--------------------------------------------------------------------
cat << EOF >> /usr/local/docker/rocketmq/broker-a/conf/2m-2s-async/broker-a.properties
namesrvAddr=192.168.5.163:9876;192.168.5.164:9876
brokerIP1=192.168.5.164
EOF
sed -i -e 's+brokerName=broker-a+brokerName=broker-a-s+' -e 's+brokerId=0+brokerId=1+' -e 's+brokerRole=ASYNC_MASTER+brokerRole=SLAVE+' /usr/local/docker/rocketmq/broker-a/conf/2m-2s-async/broker-a-s.properties
cat << EOF >> /usr/local/docker/rocketmq/broker-a/conf/2m-2s-async/broker-a-s.properties
namesrvAddr=192.168.5.163:9876;192.168.5.164:9876
brokerIP1=192.168.5.165
EOF
cat << EOF >> /usr/local/docker/rocketmq/broker-a/conf/2m-2s-async/broker-b.properties
namesrvAddr=192.168.5.163:9876;192.168.5.164:9876
brokerIP1=192.168.5.165
EOF
sed -i -e 's+brokerName=broker-b+brokerName=broker-b-s+' -e 's+brokerId=0+brokerId=1+' -e 's+brokerRole=ASYNC_MASTER+brokerRole=SLAVE+' /usr/local/docker/rocketmq/broker-a/conf/2m-2s-async/broker-b-s.properties
cat << EOF >> /usr/local/docker/rocketmq/broker-a/conf/2m-2s-async/broker-b-s.properties
namesrvAddr=192.168.5.163:9876;192.168.5.164:9876
brokerIP1=192.168.5.164
EOF

##--------------------------------------------------------------------
## 拷贝容器目录到宿主机
##--------------------------------------------------------------------
cp -r /usr/local/docker/rocketmq/namesrv-1/logs /usr/local/docker/rocketmq/namesrv-2
cp -r /usr/local/docker/rocketmq/broker-a/conf /usr/local/docker/rocketmq/broker-a-s
cp -r /usr/local/docker/rocketmq/broker-a/logs /usr/local/docker/rocketmq/broker-a-s
cp -r /usr/local/docker/rocketmq/broker-a/store /usr/local/docker/rocketmq/broker-a-s
cp -r /usr/local/docker/rocketmq/broker-a/conf /usr/local/docker/rocketmq/broker-b
cp -r /usr/local/docker/rocketmq/broker-a/logs /usr/local/docker/rocketmq/broker-b
cp -r /usr/local/docker/rocketmq/broker-a/store /usr/local/docker/rocketmq/broker-b
cp -r /usr/local/docker/rocketmq/broker-a/conf /usr/local/docker/rocketmq/broker-b-s
cp -r /usr/local/docker/rocketmq/broker-a/logs /usr/local/docker/rocketmq/broker-b-s
cp -r /usr/local/docker/rocketmq/broker-a/store /usr/local/docker/rocketmq/broker-b-s

chown -R 3000:3000 /usr/local/docker/rocketmq

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
rocketmq.config.namesrvAddr=192.168.5.163:9876;192.168.5.164:9876
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
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/rocketmq
/usr/local/docker/rocketmq
├── broker-a
│   ├── conf
│   │   ├── 2m-2s-async
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-a-s.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-b-s.properties
│   │   ├── 2m-2s-sync
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-a-s.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-b-s.properties
│   │   ├── 2m-noslave
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-trace.properties
│   │   ├── broker.conf
│   │   ├── dledger
│   │   │   ├── broker-n0.conf
│   │   │   ├── broker-n1.conf
│   │   │   └── broker-n2.conf
│   │   ├── logback_broker.xml
│   │   ├── logback_namesrv.xml
│   │   ├── logback_tools.xml
│   │   ├── plain_acl.yml
│   │   └── tools.yml
│   ├── logs
│   │   └── rocketmqlogs
│   │       ├── broker_default.log
│   │       ├── broker.log
│   │       ├── commercial.log
│   │       ├── filter.log
│   │       ├── lock.log
│   │       ├── protection.log
│   │       ├── remoting.log
│   │       ├── stats.log
│   │       ├── storeerror.log
│   │       ├── store.log
│   │       ├── transaction.log
│   │       └── watermark.log
│   ├── store
│   │   ├── abort
│   │   ├── checkpoint
│   │   ├── commitlog
│   │   ├── config
│   │   │   ├── consumerFilter.json
│   │   │   ├── consumerOffset.json
│   │   │   └── delayOffset.json
│   │   ├── consumequeue
│   │   └── lock
│   └── wait-for-it.sh
├── broker-a-s
│   ├── conf
│   │   ├── 2m-2s-async
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-a-s.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-b-s.properties
│   │   ├── 2m-2s-sync
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-a-s.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-b-s.properties
│   │   ├── 2m-noslave
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-trace.properties
│   │   ├── broker.conf
│   │   ├── dledger
│   │   │   ├── broker-n0.conf
│   │   │   ├── broker-n1.conf
│   │   │   └── broker-n2.conf
│   │   ├── logback_broker.xml
│   │   ├── logback_namesrv.xml
│   │   ├── logback_tools.xml
│   │   ├── plain_acl.yml
│   │   └── tools.yml
│   ├── logs
│   │   └── rocketmqlogs
│   │       ├── broker_default.log
│   │       ├── broker.log
│   │       ├── commercial.log
│   │       ├── filter.log
│   │       ├── lock.log
│   │       ├── protection.log
│   │       ├── remoting.log
│   │       ├── stats.log
│   │       ├── storeerror.log
│   │       ├── store.log
│   │       ├── transaction.log
│   │       └── watermark.log
│   ├── store
│   │   ├── abort
│   │   ├── checkpoint
│   │   ├── commitlog
│   │   ├── config
│   │   │   ├── consumerFilter.json
│   │   │   ├── consumerOffset.json
│   │   │   └── delayOffset.json
│   │   ├── consumequeue
│   │   └── lock
│   └── wait-for-it.sh
├── broker-b
│   ├── conf
│   │   ├── 2m-2s-async
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-a-s.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-b-s.properties
│   │   ├── 2m-2s-sync
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-a-s.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-b-s.properties
│   │   ├── 2m-noslave
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-trace.properties
│   │   ├── broker.conf
│   │   ├── dledger
│   │   │   ├── broker-n0.conf
│   │   │   ├── broker-n1.conf
│   │   │   └── broker-n2.conf
│   │   ├── logback_broker.xml
│   │   ├── logback_namesrv.xml
│   │   ├── logback_tools.xml
│   │   ├── plain_acl.yml
│   │   └── tools.yml
│   ├── logs
│   │   └── rocketmqlogs
│   │       ├── broker_default.log
│   │       ├── broker.log
│   │       ├── commercial.log
│   │       ├── filter.log
│   │       ├── lock.log
│   │       ├── protection.log
│   │       ├── remoting.log
│   │       ├── stats.log
│   │       ├── storeerror.log
│   │       ├── store.log
│   │       ├── transaction.log
│   │       └── watermark.log
│   ├── store
│   │   ├── abort
│   │   ├── checkpoint
│   │   ├── commitlog
│   │   ├── config
│   │   │   ├── consumerFilter.json
│   │   │   ├── consumerOffset.json
│   │   │   └── delayOffset.json
│   │   ├── consumequeue
│   │   └── lock
│   └── wait-for-it.sh
├── broker-b-s
│   ├── conf
│   │   ├── 2m-2s-async
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-a-s.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-b-s.properties
│   │   ├── 2m-2s-sync
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-a-s.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-b-s.properties
│   │   ├── 2m-noslave
│   │   │   ├── broker-a.properties
│   │   │   ├── broker-b.properties
│   │   │   └── broker-trace.properties
│   │   ├── broker.conf
│   │   ├── dledger
│   │   │   ├── broker-n0.conf
│   │   │   ├── broker-n1.conf
│   │   │   └── broker-n2.conf
│   │   ├── logback_broker.xml
│   │   ├── logback_namesrv.xml
│   │   ├── logback_tools.xml
│   │   ├── plain_acl.yml
│   │   └── tools.yml
│   ├── logs
│   │   └── rocketmqlogs
│   │       ├── broker_default.log
│   │       ├── broker.log
│   │       ├── commercial.log
│   │       ├── filter.log
│   │       ├── lock.log
│   │       ├── protection.log
│   │       ├── remoting.log
│   │       ├── stats.log
│   │       ├── storeerror.log
│   │       ├── store.log
│   │       ├── transaction.log
│   │       └── watermark.log
│   ├── store
│   │   ├── abort
│   │   ├── checkpoint
│   │   ├── commitlog
│   │   ├── config
│   │   │   ├── consumerFilter.json
│   │   │   ├── consumerOffset.json
│   │   │   └── delayOffset.json
│   │   ├── consumequeue
│   │   └── lock
│   └── wait-for-it.sh
├── dashboard
│   ├── conf
│   │   ├── application.properties
│   │   └── users.properties
│   ├── logs
│   │   └── consolelogs
│   │       └── rocketmq-console.log
│   └── wait-for-it.sh
├── namesrv-1
│   └── logs
│       └── rocketmqlogs
│           ├── namesrv_default.log
│           └── namesrv.log
└── namesrv-2
    └── logs
        └── rocketmqlogs
            ├── namesrv_default.log
            └── namesrv.log
```

注:

1. 从容器里拷贝配置文件到宿主机
2. 开启控制台 dashboard 密码验证: ```admin:admin```，测试验证:
    ```bash
    # docker run -d --name mqdashboard --privileged=true -p 8080:8080 -e "NAMESRV_ADDR=192.168.4.10:9876" --network=rocketmq --ip 192.168.4.30 -v /usr/local/docker/rocketmq/dashboard/conf/application.properties:/application.properties -v /usr/local/docker/rocketmq/dashboard/conf/users.properties:/tmp/rocketmq-console/data/users.properties -t apacherocketmq/rocketmq-dashboard:latest
    ```

复制 192.168.5.163(manager) 的 rocketmq 目录到 192.168.5.164(worker):

```bash
# scp -r /usr/local/docker/rocketmq root@192.168.5.164:/usr/local/docker
```

复制 192.168.5.163(manager) 的 rocketmq 目录到 192.168.5.165(worker):

```bash
# scp -r /usr/local/docker/rocketmq root@192.168.5.165:/usr/local/docker
```

给 192.168.5.163(manager) 的 rocketmq 目录授权:

```bash
# chown -R 3000:3000 /usr/local/docker/rocketmq
```

给 192.168.5.164(worker) 的 rocketmq 目录授权:

```bash
# chown -R 3000:3000 /usr/local/docker/rocketmq
```

给 192.168.5.165(worker) 的 rocketmq 目录授权:

```bash
# chown -R 3000:3000 /usr/local/docker/rocketmq
```

## 配置 docker-compose.yml

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# vim /usr/local/docker/docker-compose.yml
```

```yml
version: '3.9'

services:
 mqnamesrv-1:
   image: apache/rocketmq:latest
   volumes:
     - /usr/local/docker/rocketmq/namesrv-1/logs:/home/rocketmq/logs
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: sh mqnamesrv
   ports:
     - 9876:9876
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-163
         - node.role == manager
   networks:
     - iot
 mqnamesrv-2:
   image: apache/rocketmq:latest
   volumes:
     - /usr/local/docker/rocketmq/namesrv-2/logs:/home/rocketmq/logs
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: sh mqnamesrv
   ports:
     - 9877:9876
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-164
         - node.role == worker
   networks:
     - iot
 mqbroker-a:
   depends_on:
     - mqnamesrv-1
     - mqnamesrv-2
   image: apache/rocketmq:latest
   volumes:
     - /usr/local/docker/rocketmq/broker-a/conf:/home/rocketmq/rocketmq-4.9.2/conf
     - /usr/local/docker/rocketmq/broker-a/logs:/home/rocketmq/logs
     - /usr/local/docker/rocketmq/broker-a/store:/home/rocketmq/store
     - /usr/local/docker/rocketmq/broker-a/wait-for-it.sh:/etc/wait-for-it.sh
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: sh -c '/etc/wait-for-it.sh 192.168.5.163:9876  -t 60 && /etc/wait-for-it.sh 192.168.5.164:9876 -t 60 -- /home/rocketmq/rocketmq-4.9.2/bin/mqbroker -c /home/rocketmq/rocketmq-4.9.2/conf/2m-2s-async/broker-a.properties'
   ports:
     - 10909:10909
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-164
         - node.role == worker
   networks:
     - iot
 mqbroker-a-s:
   depends_on:
     - mqnamesrv-1
     - mqnamesrv-2
   image: apache/rocketmq:latest
   volumes:
     - /usr/local/docker/rocketmq/broker-a-s/conf:/home/rocketmq/rocketmq-4.9.2/conf
     - /usr/local/docker/rocketmq/broker-a-s/logs:/home/rocketmq/logs
     - /usr/local/docker/rocketmq/broker-a-s/store:/home/rocketmq/store
     - /usr/local/docker/rocketmq/broker-a-s/wait-for-it.sh:/etc/wait-for-it.sh
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: sh -c '/etc/wait-for-it.sh 192.168.5.163:9876  -t 60 && /etc/wait-for-it.sh 192.168.5.164:9876 -t 60 -- /home/rocketmq/rocketmq-4.9.2/bin/mqbroker -c /home/rocketmq/rocketmq-4.9.2/conf/2m-2s-async/broker-a-s.properties'
   ports:
     - 10910:10909
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-165
         - node.role == worker
   networks:
     - iot
 mqbroker-b:
   depends_on:
     - mqnamesrv-1
     - mqnamesrv-2
   image: apache/rocketmq:latest
   volumes:
     - /usr/local/docker/rocketmq/broker-b/conf:/home/rocketmq/rocketmq-4.9.2/conf
     - /usr/local/docker/rocketmq/broker-b/logs:/home/rocketmq/logs
     - /usr/local/docker/rocketmq/broker-b/store:/home/rocketmq/store
     - /usr/local/docker/rocketmq/broker-a-s/wait-for-it.sh:/etc/wait-for-it.sh
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: sh -c '/etc/wait-for-it.sh 192.168.5.163:9876  -t 60 && /etc/wait-for-it.sh 192.168.5.164:9876 -t 60 -- /home/rocketmq/rocketmq-4.9.2/bin/mqbroker -c /home/rocketmq/rocketmq-4.9.2/conf/2m-2s-async/broker-b.properties'
   ports:
     - 10911:10909
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-165
         - node.role == worker
   networks:
     - iot
 mqbroker-b-s:
   depends_on:
     - mqnamesrv-1
     - mqnamesrv-2
   image: apache/rocketmq:latest
   volumes:
     - /usr/local/docker/rocketmq/broker-b-s/conf:/home/rocketmq/rocketmq-4.9.2/conf
     - /usr/local/docker/rocketmq/broker-b-s/logs:/home/rocketmq/logs
     - /usr/local/docker/rocketmq/broker-b-s/store:/home/rocketmq/store
     - /usr/local/docker/rocketmq/broker-b-s/wait-for-it.sh:/etc/wait-for-it.sh
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: sh -c '/etc/wait-for-it.sh 192.168.5.163:9876  -t 60 && /etc/wait-for-it.sh 192.168.5.164:9876 -t 60 -- /home/rocketmq/rocketmq-4.9.2/bin/mqbroker -c /home/rocketmq/rocketmq-4.9.2/conf/2m-2s-async/broker-b-s.properties'
   ports:
     - 10912:10909
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-164
         - node.role == worker
   networks:
     - iot
 mqdashboard:
   depends_on:
     - mqnamesrv-1
     - mqnamesrv-2
   image: apacherocketmq/rocketmq-dashboard:latest
   volumes:
     - /usr/local/docker/rocketmq/dashboard/conf/application.properties:/application.properties
     - /usr/local/docker/rocketmq/dashboard/conf/users.properties:/tmp/rocketmq-console/data/users.properties
     - /usr/local/docker/rocketmq/dashboard/wait-for-it.sh:/etc/wait-for-it.sh
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: sh -c '/etc/wait-for-it.sh 192.168.5.163:9876  -t 60 && /etc/wait-for-it.sh 192.168.5.164:9876 -t 60 -- java $JAVA_OPTS -jar /rocketmq-dashboard.jar'
   ports:
     - 8080:8080
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-163
         - node.role == manager
   networks:
     - iot
networks:
  iot:
    driver: overlay
    external: true
```

## 启动 docker-compose

***注意: 只有当 namesrv 完全启动后，才可以启动 broker***

在宿主机 192.168.5.163(manager) 上启动 roqketmq 服务:

```bash
# docker stack deploy -c /usr/local/docker/docker-compose.yml iot
Creating service iot_mqnamesrv-1
Creating service iot_mqnamesrv-2
Creating service iot_mqbroker-a
Creating service iot_mqbroker-a-s
Creating service iot_mqbroker-b
Creating service iot_mqbroker-b-s
Creating service iot_mqdashboard

# docker stack ps iot | grep mq
iliy6gdt7ago   iot_mqbroker-a-s.1      apache/rocketmq:latest                                 centos-docker-165   Running         Running 35 seconds ago                                           
ua34jax30grz   iot_mqbroker-a.1        apache/rocketmq:latest                                 centos-docker-164   Running         Running 52 seconds ago                                           
igx1ftfp3pme   iot_mqbroker-b-s.1      apache/rocketmq:latest                                 centos-docker-164   Running         Running about a minute ago                                       
11imo2nuo965   iot_mqbroker-b.1        apache/rocketmq:latest                                 centos-docker-165   Running         Running 33 seconds ago                                           
qtt51idc551f   iot_mqdashboard.1       apacherocketmq/rocketmq-dashboard:latest               centos-docker-163   Running         Running about a minute ago                                       
rbdyxzh95kog   iot_mqnamesrv-1.1       apache/rocketmq:latest                                 centos-docker-163   Running         Running about a minute ago                                       
x9xwhlstyk78   iot_mqnamesrv-2.1       apache/rocketmq:latest                                 centos-docker-164   Running         Running about a minute ago                                       

# docker service ls | grep mq
xe6dnxbs5s7p   iot_mqbroker-a        replicated   1/1        apache/rocketmq:latest                                 
xbbs37g0uip8   iot_mqbroker-a-s      replicated   1/1        apache/rocketmq:latest                                 
zmxo5dm9jl6e   iot_mqbroker-b        replicated   1/1        apache/rocketmq:latest                                 
naoal8zws1qo   iot_mqbroker-b-s      replicated   1/1        apache/rocketmq:latest                                 
la7nbubkdtfe   iot_mqdashboard       replicated   1/1        apacherocketmq/rocketmq-dashboard:latest               *:8080->8080/tcp
hsli03bp2lnq   iot_mqnamesrv-1       replicated   1/1        apache/rocketmq:latest                                 *:9876->9876/tcp
ng1wuiikhdul   iot_mqnamesrv-2       replicated   1/1        apache/rocketmq:latest                                 *:9877->9876/tcp
```

## 查看 rocketmq 容器

查看宿主机 192.168.5.163 的 rocketmq 容器:

```bash
# docker ps | grep mq
ce8395f750d8   apache/rocketmq:latest                                 "sh mqnamesrv"            6 minutes ago   Up 5 minutes          9876/tcp, 10909/tcp, 10911-10912/tcp   iot_mqnamesrv-1.1.rbdyxzh95kog19v60qm5r8hnx
23b093c68d33   apacherocketmq/rocketmq-dashboard:latest               "sh -c 'java $JAVA_O…"   6 minutes ago   Up 6 minutes                                                 iot_mqdashboard.1.qtt51idc551f125vxi2hxcmno                                                                     iot_pxc_1.1.ke7cm5ymjghn732lb3o5rxksn
```

查看宿主机 192.168.5.164 的 rocketmq 容器:

```bash
# docker ps | grep mq
12c6272227cb   apache/rocketmq:latest                                 "sh -c '/etc/wait-fo…"   6 minutes ago    Up 6 minutes    9876/tcp, 10909/tcp, 10911-10912/tcp   iot_mqbroker-a.1.ua34jax30grzq8weda42nn05z
e4485b66cdfd   apache/rocketmq:latest                                 "sh mqnamesrv"            6 minutes ago    Up 6 minutes    9876/tcp, 10909/tcp, 10911-10912/tcp   iot_mqnamesrv-2.1.x9xwhlstyk78lhj9a9j40k3xl
1df6b1db8d20   apache/rocketmq:latest                                 "sh -c '/etc/wait-fo…"   7 minutes ago    Up 7 minutes    9876/tcp, 10909/tcp, 10911-10912/tcp   iot_mqbroker-b-s.1.igx1ftfp3pmemsn2q40hrmx8w
```

查看宿主机 192.168.5.165 的 rocketmq 容器:

```bash
# docker ps | grep mq
8d9c98bce682   apache/rocketmq:latest                                 "sh -c '/etc/wait-fo…"   6 minutes ago   Up 6 minutes   9876/tcp, 10909/tcp, 10911-10912/tcp   iot_mqbroker-b.1.11imo2nuo965j45nc06us2msq
aeb940c268e0   apache/rocketmq:latest                                 "sh -c '/etc/wait-fo…"   6 minutes ago   Up 6 minutes   9876/tcp, 10909/tcp, 10911-10912/tcp   iot_mqbroker-a-s.1.iliy6gdt7ago6yq7tdwr5rgl8
```

## 访问 dashboard

连接: ```$HOST:8080```，用户名/密码: ```admin/admin```

```
http://192.168.5.163:8080/
```
