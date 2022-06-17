# docker-minio 集群 (bridge)

## 前言

- minio 版本: ```minio/minio:latest```

- 网络配置: 驱动类型为 bridge，名称为 minio ，子网掩码为 ```192.168.7.0/24```，网关为 ```192.168.7.1```

- minio 集群
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |minio-1|192.168.7.11|18083:18083<br />1883:1883<br />8883:8883|192.168.204.107|/usr/local/docker/emqx/node-1/etc:/opt/emqx/etc<br />/usr/local/docker/emqx/node-1/data\:/opt/emqx/data<br />/usr/local/docker/emqx/node-1/log:/opt/emqx/log|
    |minio-2|192.168.7.12|18084:18083<br />1884:1883<br />8884:8883|192.168.204.107|/usr/local/docker/emqx/node-2/etc:/opt/emqx/etc<br />/usr/local/docker/emqx/node-2/data\:/opt/emqx/data<br />/usr/local/docker/emqx/node-2/log:/opt/emqx/log|
    |minio-3|192.168.7.13|18085:18083<br />1885:1883<br />8885:8883|192.168.204.107|/usr/local/docker/emqx/node-3/etc:/opt/emqx/etc<br />/usr/local/docker/emqx/node-3/data\:/opt/emqx/data<br />/usr/local/docker/emqx/node-3/log:/opt/emqx/log|
    |minio-3|192.168.7.14|18085:18083<br />1885:1883<br />8885:8883|192.168.204.107|/usr/local/docker/emqx/node-3/etc:/opt/emqx/etc<br />/usr/local/docker/emqx/node-3/data\:/opt/emqx/data<br />/usr/local/docker/emqx/node-3/log:/opt/emqx/log|

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

- 单节点运行
    ```bash
    # docker run -p 9000:9000 -p 9090:9090 \
     --name minio \
     --net minio \
     -d --restart=always \
     -e MINIO_ROOT_USER=admin \
     -e MINIO_ROOT_PASSWORD=miniominio \
     minio/minio server /data --console-address ":9090" --address ":9000"
    ```

## 拉取 minio 镜像

```bash
# docker pull minio/minio:latest

# docker images
REPOSITORY                                      TAG       IMAGE ID       CREATED         SIZE
minio/minio                                     latest    e31e0721a96b   5 months ago    406MB
```

## 创建网络

```bash
# docker network create minio --subnet 192.168.7.0/24 --gateway=192.168.7.1

# docker network ls
NETWORK ID     NAME            DRIVER    SCOPE
601873486f2c   minio           bridge    local

# docker network inspect minio
[
    {
        "Name": "minio",
        "Id": "601873486f2c9a8c4a6838660522f74b06565727ef66f4a61d9abf22aa76265e",
        "Created": "2022-06-16T21:00:08.164200419-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.7.0/24",
                    "Gateway": "192.168.7.1"
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

## 在宿主机上创建 minio 目录

```bash
# mkdir -p /usr/local/docker/minio

# > /usr/local/docker/minio/create-node.sh

# vim /usr/local/docker/minio/create-node.sh
```

```sh
#!/bin/sh
mkdir -p /usr/local/docker/minio/{config,node}
for index in $(seq 1 4);
do
  mkdir -p /usr/local/docker/minio/node/node-${index}
  for i in $(seq 1 4);
  do
    mkdir -p /usr/local/docker/minio/node/node-${index}/data-${i}
  done
done
```

```bash
# chmod +x /usr/local/docker/minio/create-node.sh

# /usr/local/docker/minio/create-node.sh

# ls /usr/local/docker/minio
config  create-node.sh  node

# ls /usr/local/docker/minio/node
node-1  node-2  node-3  node-4

# ls /usr/local/docker/minio/node/node-1
data-1  data-2  data-3  data-4
```

## 配置 docker-compose.yml

[minio 集群](http://docs.minio.org.cn/docs/master/distributed-minio-quickstart-guide 'minio 集群')

```bash
# > /usr/local/docker/minio/create-docker-compose.sh

# vim /usr/local/docker/minio/create-docker-compose.sh
```

```sh
#!/bin/sh
> /usr/local/docker/minio/docker-compose.yml
cat << EOF >> /usr/local/docker/minio/docker-compose.yml
version: '3.9'

services:
EOF

for index in $(seq 1 4);
do
cat << EOF >> /usr/local/docker/minio/docker-compose.yml
 minio-${index}:
  image: minio/minio:latest
  container_name: minio-${index}
  restart: always
  volumes:
   - /usr/local/docker/minio/config:/root/.minio
   - /usr/local/docker/minio/node/node-${index}/data-1:/data/data-1
   - /usr/local/docker/minio/node/node-${index}/data-2:/data/data-2
   - /usr/local/docker/minio/node/node-${index}/data-3:/data/data-3
   - /usr/local/docker/minio/node/node-${index}/data-4:/data/data-4
   - /etc/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  environment:
   - MINIO_ROOT_USER=admin
   - MINIO_ROOT_PASSWORD=miniominio
  command: server --address ":9000" --console-address ":9090" http://192.168.7.1{1...4}/data/data-{1...4}
  ports:
   - 900$(expr ${index} - 1):9000
   - 909$(expr ${index} - 1):9090
  networks:
    minio:
      ipv4_address: 192.168.7.1${index}
EOF
done
cat << EOF >> /usr/local/docker/minio/docker-compose.yml
networks:
    minio:
      name: minio
EOF
```

```bash
# chmod +x /usr/local/docker/minio/create-docker-compose.sh

# /usr/local/docker/minio/create-docker-compose.sh

# ls /usr/local/docker/minio
config  create-docker-compose.sh  create-node.sh  docker-compose.yml  node
```

***完整的 docker-compose.yml:***

```yml
version: '3.9'

services:
 minio-1:
  image: minio/minio:latest
  container_name: minio-1
  restart: always
  volumes:
   - /usr/local/docker/minio/config:/root/.minio
   - /usr/local/docker/minio/node/node-1/data-1:/data/data-1
   - /usr/local/docker/minio/node/node-1/data-2:/data/data-2
   - /usr/local/docker/minio/node/node-1/data-3:/data/data-3
   - /usr/local/docker/minio/node/node-1/data-4:/data/data-4
   - /etc/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  environment:
   - MINIO_ROOT_USER=admin
   - MINIO_ROOT_PASSWORD=miniominio
  command: server --address ":9000" --console-address ":9090" http://192.168.7.1{1...4}/data/data-{1...4}
  ports:
   - 9000:9000
   - 9090:9090
  networks:
    minio:
      ipv4_address: 192.168.7.11
 minio-2:
  image: minio/minio:latest
  container_name: minio-2
  restart: always
  volumes:
   - /usr/local/docker/minio/config:/root/.minio
   - /usr/local/docker/minio/node/node-2/data-1:/data/data-1
   - /usr/local/docker/minio/node/node-2/data-2:/data/data-2
   - /usr/local/docker/minio/node/node-2/data-3:/data/data-3
   - /usr/local/docker/minio/node/node-2/data-4:/data/data-4
   - /etc/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  environment:
   - MINIO_ROOT_USER=admin
   - MINIO_ROOT_PASSWORD=miniominio
  command: server --address ":9000" --console-address ":9090" http://192.168.7.1{1...4}/data/data-{1...4}
  ports:
   - 9001:9000
   - 9091:9090
  networks:
    minio:
      ipv4_address: 192.168.7.12
 minio-3:
  image: minio/minio:latest
  container_name: minio-3
  restart: always
  volumes:
   - /usr/local/docker/minio/config:/root/.minio
   - /usr/local/docker/minio/node/node-3/data-1:/data/data-1
   - /usr/local/docker/minio/node/node-3/data-2:/data/data-2
   - /usr/local/docker/minio/node/node-3/data-3:/data/data-3
   - /usr/local/docker/minio/node/node-3/data-4:/data/data-4
   - /etc/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  environment:
   - MINIO_ROOT_USER=admin
   - MINIO_ROOT_PASSWORD=miniominio
  command: server --address ":9000" --console-address ":9090" http://192.168.7.1{1...4}/data/data-{1...4}
  ports:
   - 9002:9000
   - 9092:9090
  networks:
    minio:
      ipv4_address: 192.168.7.13
 minio-4:
  image: minio/minio:latest
  container_name: minio-4
  restart: always
  volumes:
   - /usr/local/docker/minio/config:/root/.minio
   - /usr/local/docker/minio/node/node-4/data-1:/data/data-1
   - /usr/local/docker/minio/node/node-4/data-2:/data/data-2
   - /usr/local/docker/minio/node/node-4/data-3:/data/data-3
   - /usr/local/docker/minio/node/node-4/data-4:/data/data-4
   - /etc/timezone:/etc/timezone
   - /etc/localtime:/etc/localtime
  environment:
   - MINIO_ROOT_USER=admin
   - MINIO_ROOT_PASSWORD=miniominio
  command: server --address ":9000" --console-address ":9090" http://192.168.7.1{1...4}/data/data-{1...4}
  ports:
   - 9003:9000
   - 9093:9090
  networks:
    minio:
      ipv4_address: 192.168.7.14
networks:
    minio:
      name: minio
```

## 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/minio/docker-compose.yml up -d
[+] Running 4/4
 ⠿ Container minio-2  Started                                                                                    1.5s
 ⠿ Container minio-4  Started                                                                                    1.2s
 ⠿ Container minio-3  Started                                                                                    1.5s
 ⠿ Container minio-1  Started                                                                                    1.4s

# docker ps
CONTAINER ID   IMAGE                           COMMAND                  CREATED          STATUS          PORTS                                                                                            NAMES
f662ea93fd2d   minio/minio:latest              "/usr/bin/docker-ent…"   20 seconds ago   Up 18 seconds   0.0.0.0:9001->9000/tcp, :::9001->9000/tcp, 0.0.0.0:9091->9090/tcp, :::9091->9090/tcp             minio-2
90dd167f6c5e   minio/minio:latest              "/usr/bin/docker-ent…"   20 seconds ago   Up 18 seconds   0.0.0.0:9000->9000/tcp, :::9000->9000/tcp, 0.0.0.0:9090->9090/tcp, :::9090->9090/tcp             minio-1
a124ff55f91c   minio/minio:latest              "/usr/bin/docker-ent…"   20 seconds ago   Up 18 seconds   0.0.0.0:9002->9000/tcp, :::9002->9000/tcp, 0.0.0.0:9092->9090/tcp, :::9092->9090/tcp             minio-3
63edf7d9b48e   minio/minio:latest              "/usr/bin/docker-ent…"   20 seconds ago   Up 18 seconds   0.0.0.0:9003->9000/tcp, :::9003->9000/tcp, 0.0.0.0:9093->9090/tcp, :::9093->9090/tcp             minio-4
```

## 查看网络

```bash
# docker network inspect minio
[
    {
        "Name": "minio",
        "Id": "601873486f2c9a8c4a6838660522f74b06565727ef66f4a61d9abf22aa76265e",
        "Created": "2022-06-16T21:00:08.164200419-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.7.0/24",
                    "Gateway": "192.168.7.1"
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
            "63edf7d9b48e5a7b089d9b5a579d2248655edb0b057ef14db779aecce178b690": {
                "Name": "minio-4",
                "EndpointID": "5db5356c3b17330e058bc57328caf55e434d6602e0cc77095f49a77ea5994669",
                "MacAddress": "02:42:c0:a8:07:0e",
                "IPv4Address": "192.168.7.14/24",
                "IPv6Address": ""
            },
            "90dd167f6c5e4af3d1f25e42d1c5ad176825e213709218847273cd97f686b550": {
                "Name": "minio-1",
                "EndpointID": "b2892e12366830b14d4764c629d76593e3dcab6d933a0b3a4f79398420e5b364",
                "MacAddress": "02:42:c0:a8:07:0b",
                "IPv4Address": "192.168.7.11/24",
                "IPv6Address": ""
            },
            "a124ff55f91cf048f1f450d6a381cc52fb531373836837113c6eb615f9f39fc0": {
                "Name": "minio-3",
                "EndpointID": "06e40311e7ea4c6e53cdcf9aa5a28b996465e3c92dafe992d793f4b86be374ab",
                "MacAddress": "02:42:c0:a8:07:0d",
                "IPv4Address": "192.168.7.13/24",
                "IPv6Address": ""
            },
            "f662ea93fd2da5e3d06631a679db345832ada11cdfb705b827e72b86ff537436": {
                "Name": "minio-2",
                "EndpointID": "634dd6d50e632b9aae2680792c59ddf335989e83fa8eef96d50ff87aec35c250",
                "MacAddress": "02:42:c0:a8:07:0c",
                "IPv4Address": "192.168.7.12/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

## 访问 dashboard

可以访问任一连接: ```$HOST_IP:9090/9091/9092/9093```，用户名/密码: ```admin/miniominio```:

```
http://192.168.204.107:9090
```

## 安装 Haproxy

配置 minio 负载均衡，此示例以 Haproxy 双机热备为例。

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
mkdir -p /usr/local/docker/haproxy/node-${index}/config
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
listen  minio-tcp
	#访问的IP和端口
	bind  0.0.0.0:9000
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
        server  minio-tcp-1 192.168.7.10:9000 check weight 1 maxconn 2000
        server  minio-tcp-2 192.168.7.12:9000 check weight 1 maxconn 2000
        server  minio-tcp-3 192.168.7.13:9000 check weight 1 maxconn 2000
        server  minio-tcp-4 192.168.7.14:9000 check weight 1 maxconn 2000
        #使用keepalive检测死链
        option  tcpka
listen  minio-http
        #访问的IP和端口
        bind  0.0.0.0:9090
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
        server  minio-http-1 192.168.7.10:9090 check weight 1 maxconn 2000
        server  minio-http-2 192.168.7.12:9090 check weight 1 maxconn 2000
        server  minio-http-3 192.168.7.13:9090 check weight 1 maxconn 2000
        server  minio-http-4 192.168.7.14:9090 check weight 1 maxconn 2000
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
  ports:
   - $(expr 9000 - ${index}):9000
   - $(expr 9090 - ${index}):9090
   - 505$(expr ${index} - 1):18081
  networks:
    minio:
      ipv4_address: 192.168.7.2${index}
EOF
done
cat << EOF >> /usr/local/docker/haproxy/docker-compose.yml
networks:
    minio:
      name: minio
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
  ports:
   - 8999:9000
   - 9089:9090
   - 5050:18081
  networks:
    minio:
      ipv4_address: 192.168.7.21
 haproxy-2:
  image: haproxy:latest
  container_name: haproxy-2
  restart: always
  volumes:
   - /usr/local/docker/haproxy/node-2/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
  ports:
   - 8998:9000
   - 9088:9090
   - 5051:18081
  networks:
    minio:
      ipv4_address: 192.168.7.22
networks:
    minio:
      name: minio
```

### 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/haproxy/docker-compose.yml up -d
[+] Running 2/2
 ⠿ Container haproxy-1  Started                                                                                  1.0s
 ⠿ Container haproxy-2  Started                                                                                  1.1s

# docker ps
CONTAINER ID   IMAGE                           COMMAND                  CREATED          STATUS          PORTS                                                                                                                               NAMES
dcb075135e0f   haproxy:latest                  "docker-entrypoint.s…"   46 seconds ago   Up 45 seconds   0.0.0.0:8998->9000/tcp, :::8998->9000/tcp, 0.0.0.0:9088->9090/tcp, :::9088->9090/tcp, 0.0.0.0:5051->18081/tcp, :::5051->18081/tcp   haproxy-2
b67bf40d3755   haproxy:latest                  "docker-entrypoint.s…"   46 seconds ago   Up 45 seconds   0.0.0.0:8999->9000/tcp, :::8999->9000/tcp, 0.0.0.0:9089->9090/tcp, :::9089->9090/tcp, 0.0.0.0:5050->18081/tcp, :::5050->18081/tcp   haproxy-1
```

### 访问 minio 的 dashboard

可以访问任一连接: ```$HOST_IP:9089/9088```，用户名/密码: ```admin/miniominio```:

```
http://192.168.204.107:9089
```

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
#!/bin/bash
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

virtual_server 192.168.204.100 9000 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.7.21 9000 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 9000
       }
    }
}

virtual_server 192.168.204.100 9090 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.7.21 9090 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 9090
       }
    }
}

virtual_server 192.168.204.100 18081 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.7.21 18081 {
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
count=\`netstat -apn | grep 8999 | wc -l\`
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

virtual_server 192.168.204.100 9000 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.7.22 9000 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 9000
       }
    }
}

virtual_server 192.168.204.100 9090 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.7.22 9090 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 9090
       }
    }
}

virtual_server 192.168.204.100 18081 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.7.22 18081 {
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
count=\`netstat -apn | grep 8998 | wc -l\`
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
count=`netstat -apn | grep 8999 | wc -l`
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

### 访问 minio 的 dashboard

连接: ```$VIP:9090```，用户名/密码: ```admin/miniominio```:

```
http://192.168.204.100:9090
```

### 访问 haproxy 监控界面

连接 ```#VIP:18081``` :

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
