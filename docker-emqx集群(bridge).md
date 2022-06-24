# docker-emqx 集群 (bridge)

## 前言

- emqx 版本: ```emqx/emqx:latest```

- 网络配置: 驱动类型为 bridge，名称为 emqx ，子网掩码为 ```192.168.5.0/24```，网关为 ```192.168.5.1```

- emqx 集群
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |emqx-1|192.168.5.10|18083:18083<br />1883:1883<br />8883:8883|192.168.204.107|/usr/local/docker/emqx/node-1/etc:/opt/emqx/etc<br />/usr/local/docker/emqx/node-1/data\:/opt/emqx/data<br />/usr/local/docker/emqx/node-1/log:/opt/emqx/log|
    |emqx-2|192.168.5.11|18084:18083<br />1884:1883<br />8884:8883|192.168.204.107|/usr/local/docker/emqx/node-2/etc:/opt/emqx/etc<br />/usr/local/docker/emqx/node-2/data\:/opt/emqx/data<br />/usr/local/docker/emqx/node-2/log:/opt/emqx/log|
    |emqx-3|192.168.5.12|18085:18083<br />1885:1883<br />8885:8883|192.168.204.107|/usr/local/docker/emqx/node-3/etc:/opt/emqx/etc<br />/usr/local/docker/emqx/node-3/data\:/opt/emqx/data<br />/usr/local/docker/emqx/node-3/log:/opt/emqx/log|

- Haproxy 双机热备
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |haproxy-1|192.168.5.20|1883:1883<br />8883:8883<br />18083:18083<br />5050:18081|192.168.204.107|/usr/local/docker/haproxy/node-1/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg|
    |haproxy-2|192.168.5.21|1884:1883<br />8884:8883<br />18084:18083<br />5051:18081|192.168.204.107|/usr/local/docker/haproxy/node-2/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg|

- Keepalived 双机热备
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |keepalived-master|-|-|192.168.204.107|/usr/local/docker/keepalived/master/config/keepalived.conf:/usr/local/etc/keepalived/keepalived.conf|
    |keepalived-backup|-|-|192.168.204.107|/usr/local/docker/keepalived/backup/config/keepalived.conf:/usr/local/etc/keepalived/keepalived.conf|

- emqx 负载均衡官网(Haproxy/Nginx): [Haproxy/Nginx负载均衡](https://www.emqx.io/docs/zh/v4.4/tutorial/deploy.html 'Haproxy/Nginx负载均衡')

   注: [haproxy 合并生成证书](https://www.emqx.com/zh/blog/emqx-haproxy 'haproxy 合并生成证书')

### 启动 emqx 集群的方式

***以 emqx-1 节点为例。***

- 修改 emqx.conf:
   ```conf
   allow_anonymous = false
   cluster.discovery = static
   node.name = emqx-1@192.168.5.10
   cluster.static.seeds = emqx-1@192.168.5.10,emqx-2@192.168.5.11,emqx-3@192.168.5.12
   listener.ssl.external.keyfile = /opt/emqx/etc/certs/emqx.key
   listener.ssl.external.certfile = /opt/emqx/etc/certs/emqx.pem
   listener.ssl.external.cacertfile = /opt/emqx/etc/certs/ca.pem
   listener.ssl.external.verify = verify_peer
   listener.ssl.external.fail_if_no_peer_cert = true
   ```
- 配置环境变量
   ```yml
     environment:
      - EMQX_NAME=emqx-1
      - EMQX_HOST=192.168.5.10
      - EMQX_ALLOW__ANONYMOUS=false
      - EMQX_CLUSTER__DISCOVERY=static
      - EMQX_NODE__NAME=emqx-1@192.168.5.10
      - EMQX_CLUSTER__STATIC__SEEDS=emqx-1@192.168.5.10,emqx-2@192.168.5.11,emqx-3@192.168.5.12
      - EMQX_LISTENER__SSL__EXTERNAL__KEYFILE=/opt/emqx/etc/certs/emqx.key
      - EMQX_LISTENER__SSL__EXTERNAL__CERTFILE=/opt/emqx/etc/certs/emqx.pem
      - EMQX_LISTENER__SSL__EXTERNAL__CACERTFILE=/opt/emqx/etc/certs/ca.pem
      - EMQX_LISTENER__SSL__EXTERNAL__VERIFY=verify_peer
      - EMQX_LISTENER__SSL__EXTERNAL__FAIL_IF_NO_PEER_CERT=true
```

## 拉取 emqx 镜像

```bash
# docker pull emqx/emqx:latest

# docker images
REPOSITORY                          TAG       IMAGE ID       CREATED         SIZE
emqx/emqx                           latest    0ef9bc19d70e   5 months ago    146MB
```

## 创建网络

```bash
# docker network create emqx --subnet 192.168.5.0/24 --gateway=192.168.5.1

# docker network ls
NETWORK ID     NAME                    DRIVER    SCOPE
b744e71bc3da   emqx                    bridge    local

# docker network inspect emqx
[
    {
        "Name": "emqx",
        "Id": "b744e71bc3da2163dca4822c93bff2330046aeffaa8125531dfbd9be0692174f",
        "Created": "2022-06-08T23:54:00.086850763-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.5.0/24",
                    "Gateway": "192.168.5.1"
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

## 在宿主机上创建 emqx 目录

### 编辑 create-node.sh (创建 emqx 目录的 shell 脚本)

```bash
# docker run -d --name emqx --privileged=true -p 18083:18083 -p 1883:1883 --network=emqx --ip 192.168.5.10 emqx/emqx:latest

# mkdir -p /usr/local/docker/emqx

# > /usr/local/docker/emqx/create-node.sh

# vim /usr/local/docker/emqx/create-node.sh
```

```sh
#!/bin/sh
mkdir -p /usr/local/docker/emqx/{node-1,node-2,node-3}

##--------------------------------------------------------------------
## 拷贝容器目录到宿主机
##--------------------------------------------------------------------
docker cp emqx:/opt/emqx/etc /usr/local/docker/emqx/node-1
docker cp emqx:/opt/emqx/data /usr/local/docker/emqx/node-1
docker cp emqx:/opt/emqx/log /usr/local/docker/emqx/node-1
cp -r /usr/local/docker/emqx/node-1/* /usr/local/docker/emqx/node-2
cp -r /usr/local/docker/emqx/node-1/* /usr/local/docker/emqx/node-3

##--------------------------------------------------------------------
## 修改集群配置文件
## 1. 禁止匿名登录
## 2. 配置集群发现为静态模式
## 3. 配置 node.name
## 4. 配置 cluster.static.seeds
## 5. 配置 emqx_auth_mnesia 插件的用户名和密码
##--------------------------------------------------------------------
for index in $(seq 1 3);
do
sed -i -e 's+allow_anonymous = true+allow_anonymous = false+' -e 's+cluster.discovery = manual+cluster.discovery = static+' -e 's+node.name = emqx@127.0.0.1+node.name = emqx-1@192.168.5.1'"$(expr ${index} - 1)"'+' /usr/local/docker/emqx/node-${index}/etc/emqx.conf
cat << EOF >> /usr/local/docker/emqx/node-${index}/etc/emqx.conf

cluster.static.seeds = emqx-1@192.168.5.10,emqx-2@192.168.5.11,emqx-3@192.168.5.12
EOF
cat << EOF >> /usr/local/docker/emqx/node-${index}/etc/plugins/emqx_auth_mnesia.conf

auth.client.1.clientid = admin
auth.client.1.password = admin

auth.user.1.username = admin
auth.user.1.password = admin
EOF
done

##--------------------------------------------------------------------
## 数据目录授权
##--------------------------------------------------------------------
chmod -R 777 /usr/local/docker/emqx/node-1/{data,log}
chmod -R 777 /usr/local/docker/emqx/node-2/{data,log}
chmod -R 777 /usr/local/docker/emqx/node-3/{data,log}
```

### 配置 SSL/TLS

***如果不使用 SSL/TLS 连接，可以跳过该步骤。***

#### 生成证书

1. 创建 ca 目录
    ```bash
    # mkdir -p /usr/local/docker/ca
    # cd /usr/local/docker/ca
    ```

2. 生成根证书私钥
    ```bash
    # openssl genrsa -out ca.key 2048
    ```

3. 生成根证书，提示需要输入省份、城市等信息，可随便填写
    ```bash
    # openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.pem
    ```

4. 生成服务端证书私钥
    ```bash
    # openssl genrsa -out emqx.key 2048
    ```

5. 生成服务端证书请求
    ```bash
    # vim openssl.cnf
    ```
    
    ```cnf
    [req]
    default_bits  = 2048
    distinguished_name = req_distinguished_name
    req_extensions = req_ext
    x509_extensions = v3_req
    prompt = no
    [req_distinguished_name]
    countryName = CN
    stateOrProvinceName = Hubei
    localityName = Wuhan
    organizationName = EMQX
    commonName = CA
    [req_ext]
    subjectAltName = @alt_names
    [v3_req]
    subjectAltName = @alt_names
    [alt_names]
    IP.1 = 192.168.204.107
    DNS.1 = 8.8.8.8
    ```
    
    ```bash
    # openssl req -new -key ./emqx.key -config openssl.cnf -out emqx.csr
    ```
    ***IP.1 和 DNS.1 根据实际情况填写。***

6. 生成服务端证书
    ```bash
    # openssl x509 -req -in ./emqx.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out emqx.pem -days 3650 -sha256 -extensions v3_req -extfile openssl.cnf
    ```

7. 生成客户端证书私钥
    ```bash
    # openssl genrsa -out client.key 2048
    ```

8. 生成客户端证书请求
    ```bash
    # openssl req -new -key client.key -out client.csr -subj "/C=CN/ST=CN/L=CN/O=EMQX/CN=client"
    ```

9. 生成客户端证书
    ```bash
    # openssl x509 -req -days 3650 -in client.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out client.pem
    ```

10. 查看 ca 目录
    ```bash
    # ls /usr/local/docker/ca
    ca.key  ca.pem  ca.srl  client.csr  client.key  client.pem  emqx.csr  emqx.key  emqx.pem  openssl.cnf
    ```

#### 配置双向认证

在 create-node.sh 文件里添加以下内容:

```bash
# vim /usr/local/docker/emqx/create-node.sh
```

```sh
##--------------------------------------------------------------------
## 配置双向认证
## 1. 拷贝证书到挂载的宿主机目录
## 2. 修改 emqx.conf 里的证书配置项
##--------------------------------------------------------------------
for index in $(seq 1 3);
do
cp /usr/local/docker/ca/{emqx.key,emqx.pem,ca.pem} /usr/local/docker/emqx/node-${index}/etc/certs/
sed -i -e 's+listener.ssl.external.keyfile = etc/certs/key.pem+listener.ssl.external.keyfile = /opt/emqx/etc/certs/emqx.key+' -e 's+listener.ssl.external.certfile = etc/certs/cert.pem+listener.ssl.external.certfile = /opt/emqx/etc/certs/emqx.pem+' /usr/local/docker/emqx/node-${index}/etc/emqx.conf
cat << EOF >>  /usr/local/docker/emqx/node-${index}/etc/emqx.conf

listener.ssl.external.cacertfile = /opt/emqx/etc/certs/ca.pem
listener.ssl.external.verify = verify_peer
listener.ssl.external.fail_if_no_peer_cert = true
EOF
done
```

### 创建 emqx 目录

```bash
# chmod +x /usr/local/docker/emqx/create-node.sh

# /usr/local/docker/emqx/create-node.sh

# ls /usr/local/docker/emqx
create-node.sh  node-1  node-2  node-3

# ls /usr/local/docker/emqx/node-1/etc
acl.conf  certs  emqx.conf  lwm2m_xml  plugins  psk.txt  ssl_dist.conf  vm.args
```

## 配置 docker-compose.yml

```bash
# > /usr/local/docker/emqx/create-docker-compose.sh

# vim /usr/local/docker/emqx/create-docker-compose.sh
```

```sh
#!/bin/sh
> /usr/local/docker/emqx/docker-compose.yml
cat << EOF >> /usr/local/docker/emqx/docker-compose.yml
version: '3.9'

services:
EOF
num=2
for index in $(seq 1 3);
do
cat << EOF >> /usr/local/docker/emqx/docker-compose.yml
 emqx-${index}:
  image: emqx/emqx:latest
  container_name: emqx-${index}
  restart: always
  volumes:
   - /usr/local/docker/emqx/node-${index}/etc:/opt/emqx/etc
   - /usr/local/docker/emqx/node-${index}/data:/opt/emqx/data
   - /usr/local/docker/emqx/node-${index}/log:/opt/emqx/log
   - /etc/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  environment:
   EMQX_NAME: emqx-${index}
   EMQX_HOST: 192.168.5.1$(expr ${index} - 1)
  ports:
   - 1808$(expr ${num} + ${index}):18083
   - 188$(expr ${num} + ${index}):1883
   - 888$(expr ${num} + ${index}):8883
  networks:
    emqx:
      ipv4_address: 192.168.5.1$(expr ${index} - 1)
EOF
done
cat << EOF >> /usr/local/docker/emqx/docker-compose.yml
networks:
    emqx:
      name: emqx
EOF
```

```bash
# chmod +x /usr/local/docker/emqx/create-docker-compose.sh

# /usr/local/docker/emqx/create-docker-compose.sh

# ls /usr/local/docker/emqx
create-docker-compose.sh  create-node.sh  docker-compose.yml  node-1  node-2  node-3
```

***完整的 docker-compose.yml:***

```yml
version: '3.9'

services:
 emqx-1:
  image: emqx/emqx:latest
  container_name: emqx-1
  restart: always
  volumes:
   - /usr/local/docker/emqx/node-1/etc:/opt/emqx/etc
   - /usr/local/docker/emqx/node-1/data:/opt/emqx/data
   - /usr/local/docker/emqx/node-1/log:/opt/emqx/log
   - /etc/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  environment:
   EMQX_NAME: emqx-1
   EMQX_HOST: 192.168.5.10
  ports:
   - 18083:18083
   - 1883:1883
   - 8883:8883
  networks:
    emqx:
      ipv4_address: 192.168.5.10
 emqx-2:
  image: emqx/emqx:latest
  container_name: emqx-2
  restart: always
  volumes:
   - /usr/local/docker/emqx/node-2/etc:/opt/emqx/etc
   - /usr/local/docker/emqx/node-2/data:/opt/emqx/data
   - /usr/local/docker/emqx/node-2/log:/opt/emqx/log
   - /etc/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  environment:
   EMQX_NAME: emqx-2
   EMQX_HOST: 192.168.5.11
  ports:
   - 18084:18083
   - 1884:1883
   - 8884:8883
  networks:
    emqx:
      ipv4_address: 192.168.5.11
 emqx-3:
  image: emqx/emqx:latest
  container_name: emqx-3
  restart: always
  volumes:
   - /usr/local/docker/emqx/node-3/etc:/opt/emqx/etc
   - /usr/local/docker/emqx/node-3/data:/opt/emqx/data
   - /usr/local/docker/emqx/node-3/log:/opt/emqx/log
   - /etc/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  environment:
   EMQX_NAME: emqx-3
   EMQX_HOST: 192.168.5.12
  ports:
   - 18085:18083
   - 1885:1883
   - 8885:8883
  networks:
    emqx:
      ipv4_address: 192.168.5.12
networks:
    emqx:
      name: emqx
```

## 启动 docker-compose

```bash
# docker stop emqx

# docker rm emqx

# docker-compose -f /usr/local/docker/emqx/docker-compose.yml up -d
[+] Running 3/3
 ⠿ Container emqx-1  Started                                                                                     1.0s
 ⠿ Container emqx-2  Started                                                                                     1.1s
 ⠿ Container emqx-3  Started                                                                                     1.1s

# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                                                                                                                                            NAMES
d6f4d573d1c2   emqx/emqx:latest         "/usr/bin/docker-ent…"   12 seconds ago   Up 11 seconds   4369-4370/tcp, 5369/tcp, 6369-6370/tcp, 8081/tcp, 8083-8084/tcp, 8883/tcp, 0.0.0.0:1883->1883/tcp, :::1883->1883/tcp, 0.0.0.0:18083->18083/tcp, :::18083->18083/tcp, 11883/tcp   emqx-1
e8a09da25e34   emqx/emqx:latest         "/usr/bin/docker-ent…"   12 seconds ago   Up 11 seconds   4369-4370/tcp, 5369/tcp, 6369-6370/tcp, 8081/tcp, 8083-8084/tcp, 8883/tcp, 11883/tcp, 0.0.0.0:1884->1883/tcp, :::1884->1883/tcp, 0.0.0.0:18084->18083/tcp, :::18084->18083/tcp   emqx-2
c199651cf56b   emqx/emqx:latest         "/usr/bin/docker-ent…"   12 seconds ago   Up 11 seconds   4369-4370/tcp, 5369/tcp, 6369-6370/tcp, 8081/tcp, 8083-8084/tcp, 8883/tcp, 11883/tcp, 0.0.0.0:1885->1883/tcp, :::1885->1883/tcp, 0.0.0.0:18085->18083/tcp, :::18085->18083/tcp   emqx-3
```

## 查看网络

```bash
# docker network inspect emqx
[
    {
        "Name": "emqx",
        "Id": "b744e71bc3da2163dca4822c93bff2330046aeffaa8125531dfbd9be0692174f",
        "Created": "2022-06-08T23:54:00.086850763-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.5.0/24",
                    "Gateway": "192.168.5.1"
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
            "c199651cf56beab96acff309530ef147ed9dc0a6dbbdc3689b037ee6cd08097b": {
                "Name": "emqx-3",
                "EndpointID": "bab6dd81ae3a984adc2f83337bcf56eeaba9fa027679d94c210ef11888e57dd6",
                "MacAddress": "02:42:c0:a8:05:0c",
                "IPv4Address": "192.168.5.12/24",
                "IPv6Address": ""
            },
            "d6f4d573d1c2093f03e0556855a426a9a263085d28be33019824d121c7f1df44": {
                "Name": "emqx-1",
                "EndpointID": "f13ab9f1f8bb1b6ec9d6b473488994376c6f795c43b17d4dab5041bf2e5617ba",
                "MacAddress": "02:42:c0:a8:05:0a",
                "IPv4Address": "192.168.5.10/24",
                "IPv6Address": ""
            },
            "e8a09da25e347e16c5e095303d57dd75188a45d5ea53a52d08c59c9700eaee92": {
                "Name": "emqx-2",
                "EndpointID": "af58b45b494da02886c69d4d69693ad1e4c3227ec24000b7feda1d13bf55030d",
                "MacAddress": "02:42:c0:a8:05:0b",
                "IPv4Address": "192.168.5.11/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

## 开启 emqx_auth_mnesia 插件

1. 开启 emqx-1 容器的 emqx_auth_mnesia 插件
    ```bash
    # docker exec -it -u root emqx-1 /bin/sh
    /opt/emqx # ./bin/emqx_ctl plugins load emqx_auth_mnesia
    Plugin emqx_auth_mnesia loaded successfully.
    ```
2. 开启 emqx-2 容器的 emqx_auth_mnesia 插件
    ```bash
    # docker exec -it -u root emqx-2 /bin/sh
    /opt/emqx # ./bin/emqx_ctl plugins load emqx_auth_mnesia
    Plugin emqx_auth_mnesia loaded successfully.
    ```
3. 开启 emqx-3 容器的 emqx_auth_mnesia 插件
    ```bash
    # docker exec -it -u root emqx-3 /bin/sh
    /opt/emqx # ./bin/emqx_ctl plugins load emqx_auth_mnesia
    Plugin emqx_auth_mnesia loaded successfully.
    ```

## 访问 dashboard

### 访问 dashboard

可以访问任一连接: ```$HOST_IP:18083/18084/18085```，用户名/密码: ```admin/public```:

```
http://192.168.204.107:18083/
```

### dashboard - emqx 集群

![emqx 集群](./images/emqx/emqx-01.png 'emqx 集群')

### MQTTX 连接配置 (SSL/TLS)

连接: ```$HOST_IP:8883/8884/8885```，用户名/密码: ```admin/admin```:

![MQTTX 连接配置 (SSL/TLS)](./images/emqx/emqx-02.png 'MQTTX 连接配置 (SSL/TLS)')

### dashboard - 连接的客户端

![连接的客户端](./images/emqx/emqx-03.png '连接的客户端')

## 安装 Haproxy

配置 emqx 负载均衡，此示例以 Haproxy 双机热备为例。

### 拉取 haproxy 镜像

```bash
# docker pull haproxy:latest

# docker images
REPOSITORY                       TAG       IMAGE ID       CREATED         SIZE
haproxy                          latest    575a5788d81a   5 months ago    101MB
```

### 在宿主机上创建 haproxy 目录

```bash
# mkdir -p /usr/local/docker/haproxy

# > /usr/local/docker/haproxy/create-node.sh

# vim /usr/local/docker/haproxy/create-node.sh
```

```sh
#!/bin/sh
for index in $(seq 1 2);
do
mkdir -p /usr/local/docker/haproxy/node-${index}/{config,certs/emqx}

> /usr/local/docker/haproxy/node-${index}/certs/emqx/emqx.pem

## 合并 emqx.key 和 emqx.pem, 生成新的 emqx.pem 文件
cat /usr/local/docker/ca/emqx.pem /usr/local/docker/ca/emqx.key > /usr/local/docker/haproxy/node-${index}/certs/emqx/emqx.pem

> /usr/local/docker/haproxy/node-${index}/config/haproxy.cfg
cat << EOF >> /usr/local/docker/haproxy/node-${index}/config/haproxy.cfg
global
	#日志文件，使用rsyslog服务中local5日志设备（/var/log/local5），等级info
	log 127.0.0.1 local5 info
	#守护进程运行
	daemon

defaults
	log	global
	mode	http
	#日志格式
	option	httplog
	#日志中不记录负载均衡的心跳检测记录
	option	dontlognull
        #连接超时（毫秒）
	timeout connect 5000
        #客户端超时（毫秒）
	timeout client  50000
	#服务器超时（毫秒）
        timeout server  50000

#监控界面
listen  admin_stats
	#监控界面的访问的IP和端口
	bind  0.0.0.0:18081
	#访问协议
        mode        http
	#URI相对地址
        stats uri   /monitor
	#统计报告格式
        stats realm     Global\ statistics
	#登陆帐户信息
        stats auth  admin:admin
#负载均衡
listen  proxy-emqx-tcp
	#访问的IP和端口
	bind  0.0.0.0:1883
        #网络协议
	mode  tcp
	#负载均衡算法（轮询算法）
	#轮询算法：roundrobin
	#权重算法：static-rr
	#最少连接算法：leastconn
	#请求源IP算法：source
        balance  roundrobin
        #日志格式
        option  tcplog
        server  emqx-tcp-1 192.168.5.10:1883 check weight 1 maxconn 2000
        server  emqx-tcp-2 192.168.5.11:1883 check weight 1 maxconn 2000
        server  emqx-tcp-3 192.168.5.12:1883 check weight 1 maxconn 2000
        #使用keepalive检测死链
        option  tcpka
listen  proxy-emqx-ssl
        #访问的IP和端口
        bind  0.0.0.0:8883 ssl crt /etc/ssl/emqx/emqx.pem no-sslv3
        #网络协议
        mode  tcp
        #负载均衡算法（轮询算法）
        #轮询算法：roundrobin
        #权重算法：static-rr
        #最少连接算法：leastconn
        #请求源IP算法：source
        balance  source
        #日志格式
        option  tcplog
        server  emqx-ssl-1 192.168.5.10:1883 check inter 10000 fall 2 rise 5 weight 1
        server  emqx-ssl-2 192.168.5.11:1883 check inter 10000 fall 2 rise 5 weight 1
        server  emqx-ssl-3 192.168.5.12:1883 check inter 10000 fall 2 rise 5 weight 1
        #使用keepalive检测死链
        option  tcpka
listen  proxy-emqx-monitor
        #访问的IP和端口
        bind  0.0.0.0:18083
        #网络协议
        mode  tcp
        #负载均衡算法（轮询算法）
        #轮询算法：roundrobin
        #权重算法：static-rr
        #最少连接算法：leastconn
        #请求源IP算法：source
        balance  roundrobin
        #日志格式
        option  tcplog
        server  emqx-monitor-1 192.168.5.10:18083 check weight 1 maxconn 2000
        server  emqx-monitor-2 192.168.5.11:18083 check weight 1 maxconn 2000
        server  emqx-monitor-3 192.168.5.12:18083 check weight 1 maxconn 2000
        #使用keepalive检测死链
        option  tcpka
EOF
done
```

```bash
# chmod +x /usr/local/docker/haproxy/create-node.sh

# /usr/local/docker/haproxy/create-node.sh

# ls /usr/local/docker/haproxy/
create-node.sh  node-1  node-2
```

### 配置 docker-compose.yml

```bash
# > /usr/local/docker/haproxy/create-docker-compose.sh

# vim /usr/local/docker/haproxy/create-docker-compose.sh
```

```sh
#!/bin/sh
> /usr/local/docker/haproxy/docker-compose.yml
cat << EOF >> /usr/local/docker/haproxy/docker-compose.yml
version: '3.9'

services:
EOF
for index in $(seq 1 2);
do
cat << EOF >> /usr/local/docker/haproxy/docker-compose.yml
 haproxy-${index}:
  image: haproxy:latest
  container_name: haproxy-${index}
  restart: always
  volumes:
   - /usr/local/docker/haproxy/node-${index}/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
   - /usr/local/docker/haproxy/node-${index}/certs/emqx:/etc/ssl/emqx
  ports:
   - $(expr 1883 - ${index}):1883
   - $(expr 8883 - ${index}):8883
   - $(expr 18083 - ${index}):18083
   - 505$(expr ${index} - 1):18081
  networks:
    emqx:
      ipv4_address: 192.168.5.2$(expr ${index} - 1)
EOF
done
cat << EOF >> /usr/local/docker/haproxy/docker-compose.yml
networks:
    emqx:
      name: emqx
EOF
```

```bash
# chmod +x /usr/local/docker/haproxy/create-docker-compose.sh

# /usr/local/docker/haproxy/create-docker-compose.sh

# ls /usr/local/docker/haproxy
create-docker-compose.sh  create-node.sh  docker-compose.yml  node-1  node-2
```

***完整的 docker-compose.yml:***

```yml
version: '3.9'

services:
 haproxy-1:
  image: haproxy:latest
  container_name: haproxy-1
  restart: always
  volumes:
   - /usr/local/docker/haproxy/node-1/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
   - /usr/local/docker/haproxy/node-1/certs/emqx:/etc/ssl/emqx
  ports:
   - 1882:1883
   - 8882:8883
   - 18082:18083
   - 5050:18081
  networks:
    emqx:
      ipv4_address: 192.168.5.20
 haproxy-2:
  image: haproxy:latest
  container_name: haproxy-2
  restart: always
  volumes:
   - /usr/local/docker/haproxy/node-2/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
   - /usr/local/docker/haproxy/node-2/certs/emqx:/etc/ssl/emqx
  ports:
   - 1881:1883
   - 8881:8883
   - 18081:18083
   - 5051:18081
  networks:
    emqx:
      ipv4_address: 192.168.5.21
networks:
    emqx:
      name: emqx
```

### 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/haproxy/docker-compose.yml up -d
[+] Running 2/2
 ⠿ Container haproxy-2  Started                                                                                  2.6s
 ⠿ Container haproxy-1  Started                                                                                  2.5s

# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                                                                                                                                                                             NAMES
6b862b2068da   haproxy:latest           "docker-entrypoint.s…"   13 seconds ago   Up 10 seconds   0.0.0.0:1882->1883/tcp, :::1882->1883/tcp, 0.0.0.0:8882->8883/tcp, :::8882->8883/tcp, 0.0.0.0:5050->18081/tcp, :::5050->18081/tcp, 0.0.0.0:18082->18083/tcp, :::18082->18083/tcp                                  haproxy-1
91a364b9627b   haproxy:latest           "docker-entrypoint.s…"   13 seconds ago   Up 10 seconds   0.0.0.0:1881->1883/tcp, :::1881->1883/tcp, 0.0.0.0:8881->8883/tcp, :::8881->8883/tcp, 0.0.0.0:5051->18081/tcp, :::5051->18081/tcp, 0.0.0.0:18081->18083/tcp, :::18081->18083/tcp                                  haproxy-2
```

### 访问 emqx-dashboard

可以访问任一连接: ```$HOST_IP:18082/18081```，用户名/密码: ```admin/public```:

```
http://192.168.204.107:18082/
```

### MQTTX 连接配置 (SSL/TLS)

连接: ```$HOST_IP:8882/8881```，用户名/密码: ```admin/admin```，证书配置同上。

### 访问 haproxy 监控界面

连接 ```$HOST_IP:5050/5051``` :

```bash
# curl -XGET http://192.168.204.107:5050/monitor
```

## 安装 Keepalived

### 安装 ipvsadm

```bash
# yum install -y ipvsadm

# ipvsadm
```

### 拉取 keepalived 镜像

```bash
# docker pull osixia/keepalived

# docker images
REPOSITORY                       TAG       IMAGE ID       CREATED         SIZE
osixia/keepalived                latest    d04966a100a7   2 years ago     72.9MB
```

### 在宿主机上创建 keepalived 目录

查看宿主机的网卡，以绑定 VIP 的网卡:

```bash
# ip addr
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:a1:ab:0a brd ff:ff:ff:ff:ff:ff
    inet 192.168.204.107/24 brd 192.168.204.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ed8d:b0e:23a3:cee4/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

创建 keepalived 目录:

```bash
# mkdir -p /usr/local/docker/keepalived

# > /usr/local/docker/keepalived/create-node.sh

# vim /usr/local/docker/keepalived/create-node.sh
```

```sh
#!/bin/sh
mkdir -p /usr/local/docker/keepalived/master/config
> /usr/local/docker/keepalived/master/config/keepalived.conf
cat << EOF >> /usr/local/docker/keepalived/master/config/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.204.100
    }
}

virtual_server 192.168.204.100 1883 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.20 1883 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 1883
       }
    }
}

virtual_server 192.168.204.100 8883 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.20 8883 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 8883
       }
    }
}

virtual_server 192.168.204.100 18083 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.20 18083 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 18083
       }
    }
}

virtual_server 192.168.204.100 18081 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.20 18081 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 18081
       }
    }
}
EOF

> /usr/local/docker/keepalived/master/check-haproxy.sh
cat << EOF >> /usr/local/docker/keepalived/master/check-haproxy.sh
#!/bin/bash
count=\`netstat -apn | grep 1882 | wc -l\`
if [ \$count -gt 0 ]; then
    exit 0
else
    exit 1
fi
EOF

chmod +x /usr/local/docker/keepalived/master/check-haproxy.sh

mkdir -p /usr/local/docker/keepalived/backup/config
> /usr/local/docker/keepalived/backup/config/keepalived.conf
cat << EOF >> /usr/local/docker/keepalived/backup/config/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.204.100
    }
}

virtual_server 192.168.204.100 1883 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.21 1883 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 1883
       }
    }
}

virtual_server 192.168.204.100 8883 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.21 8883 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 8883
       }
    }
}

virtual_server 192.168.204.100 18083 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.21 18083 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 18083
       }
    }
}

virtual_server 192.168.204.100 18081 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.21 18081 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 18081
       }
    }
}
EOF

> /usr/local/docker/keepalived/backup/check-haproxy.sh
cat << EOF >> /usr/local/docker/keepalived/backup/check-haproxy.sh
#!/bin/bash
count=\`netstat -apn | grep 1881 | wc -l\`
if [ \$count -gt 0 ]; then
    exit 0
else
    exit 1
fi
EOF

chmod +x /usr/local/docker/keepalived/backup/check-haproxy.sh
```

```bash
# chmod +x /usr/local/docker/keepalived/create-node.sh

# /usr/local/docker/keepalived/create-node.sh

# ls /usr/local/docker/keepalived/
backup  create-node.sh  master
```

***完整的 check-haproxy.sh:***

```sh
#!/bin/bash
count=`netstat -apn | grep 1882 | wc -l`
if [ $count -gt 0 ]; then
    exit 0
else
    exit 1
fi
```

### 配置 docker-compose.yml

```bash
# > /usr/local/docker/keepalived/docker-compose.yml

# vim /usr/local/docker/keepalived/docker-compose.yml
```

```yml
version: '3.9'

services:
 keepalived-master:
  image: osixia/keepalived:latest
  container_name: keepalived-master
  restart: always
  cap_add:
   - NET_BROADCAST
   - NET_ADMIN
   - NET_RAW
  volumes:
   - /usr/local/docker/keepalived/master/config/keepalived.conf:/usr/local/etc/keepalived/keepalived.conf
   - /usr/local/docker/keepalived/master/check-haproxy.sh:/usr/bin/check-haproxy.sh
   - /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime
  network_mode: host
 keepalived-backup:
  image: osixia/keepalived:latest
  container_name: keepalived-backup
  restart: always
  cap_add:
   - NET_BROADCAST
   - NET_ADMIN
   - NET_RAW
  volumes:
   - /usr/local/docker/keepalived/backup/config/keepalived.conf:/usr/local/etc/keepalived/keepalived.conf
   - /usr/local/docker/keepalived/backup/check-haproxy.sh:/usr/bin/check-haproxy.sh
   - /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime
  network_mode: host
```

### 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/keepalived/docker-compose.yml up -d
[+] Running 2/2
 ⠿ Container keepalived-backup  Started                                                                          0.9s
 ⠿ Container keepalived-master  Started                                                                          0.9s

# docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS                                                                                            NAMES
e63635c59da9   osixia/keepalived:latest             "/container/tool/run"    39 seconds ago   Up 38 seconds                                                                                                    keepalived-backup
4358cebd4edd   osixia/keepalived:latest             "/container/tool/run"    39 seconds ago   Up 38 seconds                                                                                                    keepalived-master                                                                                               keepalived-master
```

查看宿主机的网卡:

```bash
# ip addr
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:a1:ab:0a brd ff:ff:ff:ff:ff:ff
    inet 192.168.204.107/24 brd 192.168.204.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.204.100/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ed8d:b0e:23a3:cee4/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

### 访问 emqx-dashboard

可以访问任一连接: ```$VIP:18083```，用户名/密码: ```admin/public```:

```
http://192.168.204.100:18083/
```

### MQTTX 连接配置 (SSL/TLS)

连接: ```$VIP:8883```，用户名/密码: ```admin/admin```。

### 访问 haproxy 监控界面

连接 ```$VIP:18081``` :

```bash
# curl -XGET http://192.168.204.100:18081/monitor
```

### FAQ

#### IPVS: Can't initialize ipvs: Protocol not available

- 解决方法

   需要安装 ```ipvsadm```，命令行:
   ```bash
   # yum install -y ipvsadm
   ```
