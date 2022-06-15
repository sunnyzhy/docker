# docker-mysql 集群 PXC(bridge)

## 前言

- 技术栈: ```PXC + Haproxy + Keepalived```

- PXC 版本: ```percona/percona-xtradb-cluster:5.7```

- 网络配置: 驱动类型为 bridge，名称为 mysql ，子网掩码为 ```192.168.1.0/24```，网关为 ```192.168.1.1```，如果已经配置了相同网络的 mysql ，就不用再配置

- PXC 集群
    |容器名称|容器IP|容器的端口|宿主机IP|映射到宿主机的端口|数据卷(挂载宿主机的数据目录)|
    |--|--|--|--|--|--|
    |pxc-1|192.168.1.20|3306|192.168.204.107|3306|pxc_1(/var/lib/docker/volumes/pxc_1/\_data)|
    |pxc-2|192.168.1.21|3306|192.168.204.107|3307|pxc_2(/var/lib/docker/volumes/pxc_2/\_data)|
    |pxc-3|192.168.1.22|3306|192.168.204.107|3308|pxc_3(/var/lib/docker/volumes/pxc_3/\_data)|

- Haproxy 双机热备
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |haproxy-1|192.168.1.30|33060:3306<br />5050:18081|192.168.204.107|/usr/local/docker/haproxy/node-1/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg|
    |haproxy-2|192.168.1.31|33061:3306<br />5051:18081|192.168.204.107|/usr/local/docker/haproxy/node-2/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg|

- Keepalived 双机热备
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |keepalived-master|-|-|192.168.204.107|/usr/local/docker/keepalived/master/config/keepalived.conf:/usr/local/etc/keepalived/keepalived.conf|
    |keepalived-backup|-|-|192.168.204.107|/usr/local/docker/keepalived/backup/config/keepalived.conf:/usr/local/etc/keepalived/keepalived.conf|

## 安装 PXC

### 拉取 PXC 镜像

```bash
# docker pull percona/percona-xtradb-cluster:5.7

# docker images
REPOSITORY                       TAG       IMAGE ID       CREATED         SIZE
percona/percona-xtradb-cluster   5.7       e37bc006844e   6 months ago    287MB
```

### 创建网络

***注: 如果已经创建了相同网络的 mysql ，就不用再创建。***

```bash
# docker network create mysql --subnet 192.168.1.0/24 --gateway=192.168.1.1

# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
6aa431529e03   bridge    bridge    local
3724599c2452   host      host      local
d61017ea048b   mysql     bridge    local
cad7d065639d   none      null      local

# docker network inspect mysql
[
    {
        "Name": "mysql",
        "Id": "d61017ea048b00bb399b4be8f4f7c37a8d32c130a773dff82e361a8a58c484e0",
        "Created": "2022-05-23T22:45:07.81170215-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.1.0/24",
                    "Gateway": "192.168.1.1"
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

### 创建 Docker 数据卷

pxc 不允许把数据保存到容器之外，也就是说 pxc 无法直接映射宿主机的目录，又由于数据卷是容器内的目录，而该目录是宿主机目录，所以 pxc 可以通过映射 docker 所创建的容器卷的方式来实现与宿主机目录的映射。

```bash
# docker volume create pxc_1

# docker volume create pxc_2

# docker volume create pxc_3

# docker volume inspect pxc_1
[
    {
        "CreatedAt": "2022-05-31T02:21:34-04:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/pxc_1/_data",
        "Name": "pxc_1",
        "Options": {},
        "Scope": "local"
    }
]
```

***数据挂载点是宿主机的 ```/var/lib/docker/volumes/{pxc_1,pxc_2,pxc_3}/_data``` 目录。***

### 在宿主机上创建 pxc 目录

```bash
# mkdir -p /usr/local/docker/pxc
```

### 配置 docker-compose.yml

```bash
# > /usr/local/docker/pxc/create-docker-compose.sh

# vim /usr/local/docker/pxc/create-docker-compose.sh
#!/bin/sh
> /usr/local/docker/pxc/docker-compose-first.yml
cat << EOF >> /usr/local/docker/pxc/docker-compose-first.yml
version: '3.9'

services:
 pxc-1:
  image: percona/percona-xtradb-cluster:5.7
  container_name: pxc-1
  restart: always
  volumes:
   - pxc_1:/var/lib/mysql
  environment:
   MYSQL_ROOT_PASSWORD: root
   XTRABACKUP_PASSWORD: root
   CLUSTER_NAME: pxc
  ports:
   - 3306:3306
  networks:
   mysql:
    ipv4_address: 192.168.1.20
networks:
  mysql:
   name: mysql
volumes:
  pxc_1:
   name: pxc_1
   external: true
EOF

> /usr/local/docker/pxc/docker-compose-second.yml
cat << EOF >> /usr/local/docker/pxc/docker-compose-second.yml
version: '3.9'

services:
EOF
for index in $(seq 2 3);
do
cat << EOF >> /usr/local/docker/pxc/docker-compose-second.yml
 pxc-${index}:
  image: percona/percona-xtradb-cluster:5.7
  container_name: pxc-${index}
  restart: always
  volumes:
   - pxc_${index}:/var/lib/mysql
  environment:
   MYSQL_ROOT_PASSWORD: root
   XTRABACKUP_PASSWORD: root
   CLUSTER_NAME: pxc
   CLUSTER_JOIN: pxc-1
  ports:
   - 330$(expr ${index} + 5):3306
  networks:
   mysql:
    ipv4_address: 192.168.1.2$(expr ${index} - 1)
EOF
done
cat << EOF >> /usr/local/docker/pxc/docker-compose-second.yml
networks:
  mysql:
   name: mysql
volumes:
EOF
for index in $(seq 2 3);
do
cat << EOF >> /usr/local/docker/pxc/docker-compose-second.yml
  pxc_${index}:
   name: pxc_${index}
   external: true
EOF
done

# chmod +x /usr/local/docker/pxc/create-docker-compose.sh

# /usr/local/docker/pxc/create-docker-compose.sh

# ls /usr/local/docker/pxc
create-docker-compose.sh  docker-compose-first.yml  docker-compose-second.yml
```

***完整的 docker-compose.yml:***

- docker-compose-first.yml
    ```yml
    version: '3.9'

    services:
     pxc-1:
      image: percona/percona-xtradb-cluster:5.7
      container_name: pxc-1
      restart: always
      volumes:
       - pxc_1:/var/lib/mysql
      environment:
       MYSQL_ROOT_PASSWORD: root
       XTRABACKUP_PASSWORD: root
       CLUSTER_NAME: pxc
      ports:
       - 3306:3306
      networks:
       mysql:
        ipv4_address: 192.168.1.20
    networks:
      mysql:
       name: mysql
    volumes:
      pxc_1:
       name: pxc_1
       external: true
    ```
- docker-compose-second.yml
    ```yml
    version: '3.9'

    services:
     pxc-2:
      image: percona/percona-xtradb-cluster:5.7
      container_name: pxc-2
      restart: always
      volumes:
       - pxc_2:/var/lib/mysql
      environment:
       MYSQL_ROOT_PASSWORD: root
       XTRABACKUP_PASSWORD: root
       CLUSTER_NAME: pxc
       CLUSTER_JOIN: pxc-1
      ports:
       - 3307:3306
      networks:
       mysql:
        ipv4_address: 192.168.1.21
     pxc-3:
      image: percona/percona-xtradb-cluster:5.7
      container_name: pxc-3
      restart: always
      volumes:
       - pxc_3:/var/lib/mysql
      environment:
       MYSQL_ROOT_PASSWORD: root
       XTRABACKUP_PASSWORD: root
       CLUSTER_NAME: pxc
       CLUSTER_JOIN: pxc-1
      ports:
       - 3308:3306
      networks:
       mysql:
        ipv4_address: 192.168.1.22
    networks:
      mysql:
       name: mysql
    volumes:
      pxc_2:
       name: pxc_2
       external: true
      pxc_3:
       name: pxc_3
       external: true
    ```

### 启动 docker-compose

***注意: 只有当第一个节点完全启动后(在本地连接成功后)，才可以创建第二个、第三个节点。***

1. 先启动 pxc-1
    ```bash
    # docker-compose -p pxc-f -f /usr/local/docker/pxc/docker-compose-first.yml up -d
    [+] Running 1/1
     ⠿ Container pxc-1  Started
    ```
2. 再启动 pxc-2/3
    ```bash
    # docker-compose -p pxc-s -f /usr/local/docker/pxc/docker-compose-second.yml up -d
    [+] Running 2/2
     ⠿ Container pxc-2  Started                                                                                      2.0s
     ⠿ Container pxc-3  Started                                                                                      1.9s
    ```
3. 查看容器
    ```bash
    # docker ps
    CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS                                                                                            NAMES
    e965159dec65   percona/percona-xtradb-cluster:5.7   "/entrypoint.sh mysq…"   11 minutes ago   Up 11 minutes   4567-4568/tcp, 0.0.0.0:3307->3306/tcp, :::3307->3306/tcp                                         pxc-2
    62129a1c224b   percona/percona-xtradb-cluster:5.7   "/entrypoint.sh mysq…"   11 minutes ago   Up 22 seconds   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 4567-4568/tcp                                         pxc-1
    c60f63153d97   percona/percona-xtradb-cluster:5.7   "/entrypoint.sh mysq…"   11 minutes ago   Up 1 second     4567-4568/tcp, 0.0.0.0:3308->3306/tcp, :::3308->3306/tcp                                         pxc-3
    ```

### 查看网络

```bash
# docker network inspect mysql
[
    {
        "Name": "mysql",
        "Id": "b0ab153c738762a03d1ed0c3e82dacd81573a6f179ec261d89dd4abb504c6b68",
        "Created": "2022-05-25T23:31:30.143462282-04:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.1.0/24",
                    "Gateway": "192.168.1.1"
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
            "62129a1c224bf54bf6f757e6ad241a9f327bac67e133e24450de8261fe965144": {
                "Name": "pxc-1",
                "EndpointID": "aa903e4b294003bee1f2f39313a8ac9dd0d68ba712a076b14dda03cc58a27f63",
                "MacAddress": "02:42:c0:a8:01:14",
                "IPv4Address": "192.168.1.20/24",
                "IPv6Address": ""
            },
            "c60f63153d97ba363912d33896b8c121f39c28bfd2e4785e4ae0c20c8a419cd1": {
                "Name": "pxc-3",
                "EndpointID": "93e5891a3c84c4864a2139ab61e1fd6b73d75d292b8559520c55b9cbcac73265",
                "MacAddress": "02:42:c0:a8:01:16",
                "IPv4Address": "192.168.1.22/24",
                "IPv6Address": ""
            },
            "e965159dec650676d82f6b50361a6fe3dc676174dd4df007a6d8c2ef6f63ae81": {
                "Name": "pxc-2",
                "EndpointID": "01bde1a6b22249526a4fe9bd878ef9ee51fd48663d74bef6828a6e796d052eb8",
                "MacAddress": "02:42:c0:a8:01:15",
                "IPv4Address": "192.168.1.21/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

### 数据同步

1. 进入 pxc-1 容器
    ```bash
    # docker exec -it pxc-1 /bin/sh
    ```
2. 分别连接 192.168.1.20/21/22，并查看各自的数据库
    ```bash
    $ mysql -uroot -proot -h192.168.1.20 -P3306
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+

    $ mysql -uroot -proot -h192.168.1.21 -P3306
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+

    $ mysql -uroot -proot -h192.168.1.22 -P3306
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    ```
3. 连接 192.168.1.20，并新建 test 数据库
    ```bash
    $ mysql -uroot -proot -h192.168.1.20 -P3306

    mysql> create database `test` default character set utf8 collate utf8_general_ci;

    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    | test               |
    +--------------------+
    ```
4. 分别连接 192.168.1.21/22，并查看各自的数据库
    ```bash
    $ mysql -uroot -proot -h192.168.1.21 -P3306
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    | test               |
    +--------------------+

    $ mysql -uroot -proot -h192.168.1.22 -P3306
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    | test               |
    +--------------------+
    ```
5. 随便连接一个数据库，创建 haproxy 账号以供 Haproxy 使用
    ```bash
    # docker exec -it pxc-1 /bin/sh

    $ mysql -uroot -proot -h192.168.1.20 -P3306
    mysql> use mysql;

    mysql> create user 'haproxy'@'%' IDENTIFIED BY '';
    ```

## 安装 Haproxy

配置数据库负载均衡，此示例以 Haproxy 双机热备为例。

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
#数据库负载均衡
listen  proxy-mysql
	#访问的IP和端口
	bind  0.0.0.0:3306
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
	#在MySQL中创建一个没有权限的haproxy用户，密码为空。Haproxy使用这个账户对MySQL数据库心跳检测
	#server  mysql-db-1 192.168.1.20:3306 check weight 1 maxconn 2000
	#mysql-db-1: 自定义服务名  172.19.0.2:3306: 容器ip:端口   check: 发送心跳监测  weight 1: 权重采用权重算法才会生效  maxconn 2000: 最大连接数	 
        option  mysql-check user haproxy
        server  mysql-db-1 192.168.1.20:3306 check weight 1 maxconn 2000
        server  mysql-db-2 192.168.1.21:3306 check weight 1 maxconn 2000
        server  mysql-db-3 192.168.1.22:3306 check weight 1 maxconn 2000
        #使用keepalive检测死链
        option  tcpka
EOF
done

# chmod +x /usr/local/docker/haproxy/create-node.sh

# /usr/local/docker/haproxy/create-node.sh

# ls /usr/local/docker/haproxy/
create-node.sh  node-1  node-2
```

### 配置 docker-compose.yml

```bash
# > /usr/local/docker/haproxy/create-docker-compose.sh

# vim /usr/local/docker/haproxy/create-docker-compose.sh
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
   - 3306$(expr ${index} - 1):3306
   - 505$(expr ${index} - 1):18081
  networks:
    mysql:
      ipv4_address: 192.168.1.3$(expr ${index} - 1)
EOF
done
cat << EOF >> /usr/local/docker/haproxy/docker-compose.yml
networks:
    mysql:
      name: mysql
EOF

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
   - 33060:3306
   - 5050:18081
  networks:
    mysql:
      ipv4_address: 192.168.1.30
 haproxy-2:
  image: haproxy:latest
  container_name: haproxy-2
  restart: always
  volumes:
   - /usr/local/docker/haproxy/node-2/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
  ports:
   - 33061:3306
   - 5051:18081
  networks:
    mysql:
      ipv4_address: 192.168.1.31
networks:
    mysql:
      name: mysql
```

### 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/haproxy/docker-compose.yml up -d
[+] Running 2/2
 ⠿ Container haproxy-2  Started                                                                                  2.6s
 ⠿ Container haproxy-1  Started                                                                                  2.5s

# docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS                                                                                            NAMES
0a675f30b045   haproxy:latest                       "docker-entrypoint.s…"   35 seconds ago   Up 32 seconds   0.0.0.0:33061->3306/tcp, :::33061->3306/tcp, 0.0.0.0:5051->18081/tcp, :::5051->18081/tcp         haproxy-2
0bfb74b47450   haproxy:latest                       "docker-entrypoint.s…"   35 seconds ago   Up 32 seconds   0.0.0.0:33060->3306/tcp, :::33060->3306/tcp, 0.0.0.0:5050->18081/tcp, :::5050->18081/tcp         haproxy-1
```

### 连接 mysql

连接 ```HOST:33060/33061``` :

```bash
# mysql -uroot -proot -h192.168.204.107 -P33060
```

### 访问 haproxy 监控界面

连接 ```HOST:5050/5051``` :

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

virtual_server 192.168.204.100 3306 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.1.30 3306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 3306
       }
    }
}

virtual_server 192.168.204.100 18081 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.1.30 18081 {
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
count=\`netstat -apn | grep 33060 | wc -l\`
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

virtual_server 192.168.204.100 3306 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.1.31 3306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 3306
       }
    }
}

virtual_server 192.168.204.100 18081 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.1.31 18081 {
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
count=\`netstat -apn | grep 33061 | wc -l\`
if [ \$count -gt 0 ]; then
    exit 0
else
    exit 1
fi
EOF

chmod +x /usr/local/docker/keepalived/backup/check-haproxy.sh

# chmod +x /usr/local/docker/keepalived/create-node.sh

# /usr/local/docker/keepalived/create-node.sh

# ls /usr/local/docker/keepalived/
backup  create-node.sh  master
```

***完整的 check-haproxy.sh:***

```sh
#!/bin/bash
count=`netstat -apn | grep 33060 | wc -l`
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

### 连接 mysql

连接 ```VIP:3306``` :

```bash
# mysql -uroot -proot -h192.168.204.100 -P3306
```

### 访问 haproxy 监控界面

连接 ```VIP:18081``` :

```bash
# curl -XGET http://192.168.204.100:18081/monitor
```

## FAQ

### It may not be safe to bootstrap the cluster from this node. It was not the last one to leave the cluster and may not contain all the updates. To force cluster bootstrap with this node, edit the grastate.dat file manually and set safe_to_bootstrap to 1 .

- 原因
   1. 定位最近状态的节点
      ```
      当我们关闭一个节点时，其 seqno 会写入 grastate.dat 文件中，这时后续的 seqno 该节点将无法接收到。

      注意数据库开启状态或者异常关闭时 ```seqno = -1```

      当我们将所有节点关闭，准备重启时我们需要知道哪个节点是最后关闭的，并使用它来引导集群。
      ```
   2. 安全引导保护
      ```
      安全引导即 safe to bootstrap ，从 3.19 版本开始，Galera 为防止在错误的节点上引导集群，引入了安全引导的保护。

      Galera 会自动判断哪个节点是最后一个离开集群的，并将信息写入 grastate.dat 文件中。
      
      如果我们使用 ```safe_to_bootstrao = 0``` 的节点来引导，数据库将无法启动。
      ```

- 解决方法

   按指引修改 ```safe_to_bootstrap = 1```:
   ```bash
   # vim /var/lib/docker/volumes/pxc_1/_data/grastate.dat
   safe_to_bootstrap: 1
   ```
   
