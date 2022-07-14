# docker-mysql 集群 PXC(host)

## 前言

- 技术栈: ```PXC + Haproxy + Keepalived```

- 网络配置: 驱动类型为 overlay

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- PXC 版本: ```percona/percona-xtradb-cluster:5.7```

- PXC 集群
    |容器名称|宿主机IP|端口映射(宿主机端口:容器端口)|数据卷(挂载宿主机的数据目录)|
    |--|--|--|--|
    |pxc_1|192.168.5.163|3306:3306|pxc(/var/lib/docker/volumes/pxc/\_data)|
    |pxc_2|192.168.5.164|3307:3306|pxc(/var/lib/docker/volumes/pxc/\_data)|
    |pxc_3|192.168.5.165|3308:3306|pxc(/var/lib/docker/volumes/pxc/\_data)|

- Haproxy
    |容器名称|宿主机IP|端口映射(宿主机端口:容器端口)|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|
    |haproxy_1|192.168.5.163|13306:3306<br />18081:18081|/usr/local/docker/haproxy/node-1/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg|
    |haproxy_2|192.168.5.164|13306:3306<br />18081:18081|/usr/local/docker/haproxy/node-2/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg|
    |haproxy_3|192.168.5.165|13306:3306<br />18081:18081|/usr/local/docker/haproxy/node-3/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg|

    注:
       1. 网络驱动类型必须是 host

- Keepalived
    |容器名称|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|
    |keepalived_1|192.168.5.163|/usr/local/docker/keepalived/node-1/config/keepalived.conf:/usr/local/etc/keepalived/keepalived.conf<br />/usr/local/docker/keepalived/node-1/check-haproxy.sh:/usr/bin/check-haproxy.sh|
    |keepalived_2|192.168.5.164|/usr/local/docker/keepalived/node-2/config/keepalived.conf:/usr/local/etc/keepalived/keepalived.conf<br />/usr/local/docker/keepalived/node-2/check-haproxy.sh:/usr/bin/check-haproxy.sh|
    |keepalived_3|192.168.5.165|/usr/local/docker/keepalived/node-3/config/keepalived.conf:/usr/local/etc/keepalived/keepalived.conf<br />/usr/local/docker/keepalived/node-3/check-haproxy.sh:/usr/bin/check-haproxy.sh|
    
    注:
       1. 网络驱动类型必须是 host
       2. 虚拟IP的端口必须与真实IP的端口保持一致，因为 Keepalived 虚拟的只是 IP 而不是端口

## 安装 PXC

### 拉取 PXC 镜像

```bash
# docker pull percona/percona-xtradb-cluster:5.7

# docker images | grep percona
REPOSITORY                       TAG       IMAGE ID       CREATED         SIZE
percona/percona-xtradb-cluster   5.7       e37bc006844e   6 months ago    287MB
```

## 在宿主机上创建 PXC 目录

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```sh
#!/bin/sh

for index in $(seq 1 3);
do
    mkdir -p /usr/local/docker/pxc/node-${index}

    if [ ${index} == 1 ]; then
        if [ ! -f /usr/local/docker/pxc/node-${index}/wait-for-it.sh ]; then
            wget -P /usr/local/docker/pxc/node-${index} https://gitee.com/sunny906/docker/raw/master/wait-for/wait-for-it.sh
        fi
    else
        if [ ! -f /usr/local/docker/pxc/node-${index}/wait-for-it.sh ]; then
            cp /usr/local/docker/pxc/node-1/wait-for-it.sh /usr/local/docker/pxc/node-${index}/
        fi
    fi

    chmod 777 /usr/local/docker/pxc/node-${index}/wait-for-it.sh
done
```

```bash
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/pxc/
/usr/local/docker/pxc/
├── node-1
│   └── wait-for-it.sh
├── node-2
│   └── wait-for-it.sh
└── node-3
    └── wait-for-it.sh
```

复制 192.168.5.163(manager) 的 pxc 目录到 192.168.5.164(worker):

```bash
# scp -r /usr/local/docker/pxc root@192.168.5.164:/usr/local/docker
```

复制 192.168.5.163(manager) 的 pxc 目录到 192.168.5.165(worker):

```bash
# scp -r /usr/local/docker/pxc root@192.168.5.165:/usr/local/docker
```

### 配置 docker-compose.yml

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# vim /usr/local/docker/docker-compose.yml
```

```yml
version: '3.9'

services:
 pxc_1:
   image: percona/percona-xtradb-cluster:5.7
   volumes:
     - pxc_1:/var/lib/mysql
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   environment:
     MYSQL_ROOT_PASSWORD: root
     XTRABACKUP_PASSWORD: root
     CLUSTER_NAME: pxc
   ports:
     - 3306:3306
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-163
         - node.role == manager
   networks:
     - iot
 pxc_2:
   image: percona/percona-xtradb-cluster:5.7
   volumes:
     - pxc_2:/var/lib/mysql
     - /usr/local/docker/pxc/node-2/wait-for-it.sh:/etc/wait-for-it.sh
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   depends_on:
     - pxc_1
   command: sh -c '/etc/wait-for-it.sh 192.168.5.163:3306 -- mysqld'
   environment:
     MYSQL_ROOT_PASSWORD: root
     XTRABACKUP_PASSWORD: root
     CLUSTER_NAME: pxc
     CLUSTER_JOIN: pxc_1
   ports:
     - 3307:3306
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.hostname == centos-docker-164
         - node.role == worker
   networks:
     - iot
 pxc_3:
   image: percona/percona-xtradb-cluster:5.7
   volumes:
     - pxc_3:/var/lib/mysql
     - /usr/local/docker/pxc/node-3/wait-for-it.sh:/etc/wait-for-it.sh
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   depends_on:
     - pxc_2
   command: sh -c '/etc/wait-for-it.sh 192.168.5.164:3306 -- mysqld'
   environment:
     MYSQL_ROOT_PASSWORD: root
     XTRABACKUP_PASSWORD: root
     CLUSTER_NAME: pxc
     CLUSTER_JOIN: pxc_1
   ports:
     - 3308:3306
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
volumes:
  pxc_1:
   name: pxc_1
   external: true
  pxc_2:
   name: pxc_2
   external: true
  pxc_3:
   name: pxc_3
   external: true
```

### 启动 docker-compose

***注意: 只有当第一个节点完全启动后(在本地连接成功后)，才可以创建第二个、第三个节点。***

在宿主机 192.168.5.163(manager) 上启动 pxc 服务:

```bash
# docker stack deploy -c /usr/local/docker/docker-compose.yml iot
Creating service iot_pxc_1
Creating service iot_pxc_2
Creating service iot_pxc_3

# docker stack ps iot | grep pxc
ke7cm5ymjghn   iot_pxc_1.1     percona/percona-xtradb-cluster:5.7   centos-docker-163   Running         Running about a minute ago             
y9q3rdw5j3kt   iot_pxc_2.1     percona/percona-xtradb-cluster:5.7   centos-docker-164   Running         Running about a minute ago             
chzpvftb1r0y   iot_pxc_3.1     percona/percona-xtradb-cluster:5.7   centos-docker-165   Running         Running about a minute ago             

# docker service ls | grep pxc
vh1pz8dls7q9   iot_pxc_1     replicated   1/1        percona/percona-xtradb-cluster:5.7   *:3306->3306/tcp
mnunrao51xrf   iot_pxc_2     replicated   1/1        percona/percona-xtradb-cluster:5.7   *:3307->3306/tcp
cz5xbbt70mv4   iot_pxc_3     replicated   1/1        percona/percona-xtradb-cluster:5.7   *:3308->3306/tcp
```

## 查看 pxc 容器

查看宿主机 192.168.5.163 的 pxc 容器:

```bash
# docker ps | grep pxc
e467d2b3924b   percona/percona-xtradb-cluster:5.7   "/entrypoint.sh mysq…"   11 minutes ago   Up 11 minutes   3306/tcp, 4567-4568/tcp                                                                          iot_pxc_1.1.ke7cm5ymjghn732lb3o5rxksn
```

查看宿主机 192.168.5.164 的 pxc 容器:

```bash
# docker ps | grep pxc
33e5968e1e36   percona/percona-xtradb-cluster:5.7   "/entrypoint.sh sh -…"   9 minutes ago   Up 9 minutes   3306/tcp, 4567-4568/tcp   iot_pxc_2.1.4z51rloaebk2o5i84gtwfs80r
```

查看宿主机 192.168.5.165 的 pxc 容器:

```bash
# docker ps | grep pxc
a2b25dda8a88   percona/percona-xtradb-cluster:5.7   "/entrypoint.sh sh -…"   11 minutes ago   Up 11 minutes   3306/tcp, 4567-4568/tcp   iot_pxc_3.1.chzpvftb1r0y1rbkxryt9cw35
```

### 数据同步

1. 进入 pxc_1 容器
    ```bash
    # docker exec -it -u root e467d2b3924b /bin/sh
    ```
2. 分别连接 192.168.5.163/164/165，并查看各自的数据库
    ```bash
    # mysql -uroot -proot -h192.168.5.163 -P3306
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+

    # mysql -uroot -proot -h192.168.5.164 -P3306
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+

    # mysql -uroot -proot -h192.168.5.165 -P3306
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
3. 连接 192.168.5.163，并新建 test 数据库
    ```bash
    # mysql -uroot -proot -h192.168.5.163 -P3306

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
4. 分别连接 192.168.5.164/165，并查看各自的数据库
    ```bash
    # mysql -uroot -proot -h192.168.5.164 -P3306
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

    # mysql -uroot -proot -h192.168.5.165 -P3306
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
    # mysql -uroot -proot -h192.168.5.163 -P3306
    mysql> use mysql;

    mysql> create user 'haproxy'@'%' IDENTIFIED BY '';
    ```

## 安装 Haproxy

配置数据库负载均衡。

### 拉取 haproxy 镜像

```bash
# docker pull haproxy:latest

# docker images | grep haproxy
REPOSITORY                       TAG       IMAGE ID       CREATED         SIZE
haproxy                          latest    575a5788d81a   5 months ago    101MB
```

### 在宿主机上创建 haproxy 目录

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```sh
#!/bin/sh

for index in $(seq 1 3);
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
	bind  0.0.0.0:13306
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
	#server  mysql-1 192.168.5.163:3306 check weight 1 maxconn 2000
	#mysql-1: 自定义服务名  172.19.0.2:3306: 容器ip:端口   check: 发送心跳监测  weight 1: 权重采用权重算法才会生效  maxconn 2000: 最大连接数	 
        option  mysql-check user haproxy
        server  mysql-1 192.168.5.163:3306 check weight 1 maxconn 2000
        server  mysql-2 192.168.5.164:3306 check weight 1 maxconn 2000
        server  mysql-3 192.168.5.165:3306 check weight 1 maxconn 2000
        #使用keepalive检测死链
        option  tcpka
EOF
done

for index in $(seq 1 3);
do
> /usr/local/docker/haproxy/node-${index}/docker-compose.yml 
cat << EOF >> /usr/local/docker/haproxy/node-${index}/docker-compose.yml 
version: '3.9'

services:
 haproxy_$index:
   image: haproxy:latest
   container_name: haproxy_$index
   restart: always
   volumes:
     - /usr/local/docker/haproxy/node-$index/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   ports:
     - 13306:13306
     - 18081:18081
   network_mode: host
EOF
done
```

```bash
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/haproxy
/usr/local/docker/haproxy
├── node-1
│   ├── config
│   │   └── haproxy.cfg
│   └── docker-compose.yml
├── node-2
│   ├── config
│   │   └── haproxy.cfg
│   └── docker-compose.yml
└── node-3
    ├── config
    │   └── haproxy.cfg
    └── docker-compose.yml
```

复制 192.168.5.163(manager) 的 haproxy 目录到 192.168.5.164(worker):

```bash
# scp -r /usr/local/docker/haproxy root@192.168.5.164:/usr/local/docker
```

复制 192.168.5.163(manager) 的 haproxy 目录到 192.168.5.165(worker):

```bash
# scp -r /usr/local/docker/haproxy root@192.168.5.165:/usr/local/docker
```

## 启动 docker-compose

在宿主机 192.168.5.163 上启动 haproxy 服务:

```bash
# docker-compose -f /usr/local/docker/haproxy/node-1/docker-compose.yml up -d 
[+] Running 1/1
 ⠿ Container haproxy_1  Started                                                                                  0.2s

# docker ps | grep haproxy
1e37d6105088   haproxy:latest                       "docker-entrypoint.s…"   10 seconds ago      Up 9 seconds                                      haproxy_1
```

在宿主机 192.168.5.164 上启动 haproxy 服务:

```bash
# docker-compose -f /usr/local/docker/haproxy/node-2/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container haproxy_2  Started                                                                                  0.2s

# docker ps -a | grep haproxy
66cc63a75d11   haproxy:latest                       "docker-entrypoint.s…"   17 seconds ago      Up 16 seconds                                haproxy_2
```

在宿主机 192.168.5.165 上启动 haproxy 服务:

```bash
# docker-compose -f /usr/local/docker/haproxy/node-3/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container haproxy_3  Started                                                                                  0.2s

# docker ps -a | grep haproxy
cebf5f242df9   haproxy:latest                       "docker-entrypoint.s…"   18 seconds ago      Up 17 seconds                                haproxy_3
```

### 连接 mysql

连接 ```$HOST_IP:13306``` :

```bash
# mysql -uroot -proot -h192.168.5.165 -P13306
```

### 访问 haproxy 监控界面

连接 ```$HOST_IP:18081```，用户名/密码: ```admin/admin``` :

```bash
# curl -XGET http://192.168.5.165:18081/monitor
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

# docker images | grep keepalived
REPOSITORY                       TAG       IMAGE ID       CREATED         SIZE
osixia/keepalived                latest    d04966a100a7   2 years ago     72.9MB
```

### 在宿主机上创建 keepalived 目录

查看宿主机的网卡，以绑定 VIP 的网卡:

```bash
# ip addr show ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:8a:8a:d3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.163/24 brd 192.168.5.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::42de:6cd4:dde4:b13e/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

创建 keepalived 目录:

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```sh
#!/bin/sh

for index in $(seq 1 3);
do
    mkdir -p /usr/local/docker/keepalived/node-${index}/config
done

> /usr/local/docker/keepalived/node-1/config/keepalived.conf
cat << EOF >> /usr/local/docker/keepalived/node-1/config/keepalived.conf
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
        192.168.5.100
    }
}

virtual_server 192.168.5.100 13306 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.163 13306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 13306
       }
    }
}

virtual_server 192.168.5.100 18081 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.163 18081 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 18081
       }
    }
}
EOF

> /usr/local/docker/keepalived/node-2/config/keepalived.conf
cat << EOF >> /usr/local/docker/keepalived/node-2/config/keepalived.conf
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
        192.168.5.100
    }
}

virtual_server 192.168.5.100 13306 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.164 13306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 13306
       }
    }
}

virtual_server 192.168.5.100 18081 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.164 18081 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 18081
       }
    }
}
EOF

> /usr/local/docker/keepalived/node-3/config/keepalived.conf
cat << EOF >> /usr/local/docker/keepalived/node-3/config/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 98
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.5.100
    }
}

virtual_server 192.168.5.100 13306 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.165 13306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 13306
       }
    }
}

virtual_server 192.168.5.100 18081 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.165 18081 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 18081
       }
    }
}
EOF

for index in $(seq 1 3);
do
> /usr/local/docker/keepalived/node-$index/check-haproxy.sh
cat << EOF >> /usr/local/docker/keepalived/node-$index/check-haproxy.sh
#!/bin/bash

count=\`netstat -apn | grep 13306 | wc -l\`
if [ \$count -gt 0 ]; then
    exit 0
else
    exit 1
fi
EOF
chmod +x /usr/local/docker/keepalived/node-$index/check-haproxy.sh
done

for index in $(seq 1 3);
do
> /usr/local/docker/keepalived/node-${index}/docker-compose.yml 
cat << EOF >> /usr/local/docker/keepalived/node-${index}/docker-compose.yml 
version: '3.9'

services:
 keepalived_$index:
   image: osixia/keepalived:latest
   container_name: keepalived_$index
   restart: always
   cap_add:
     - NET_BROADCAST
     - NET_ADMIN
     - NET_RAW
   volumes:
     - /usr/local/docker/keepalived/node-${index}/config/keepalived.conf:/usr/local/etc/keepalived/keepalived.conf
     - /usr/local/docker/keepalived/node-${index}/check-haproxy.sh:/usr/bin/check-haproxy.sh
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   network_mode: host
EOF
done
```

```bash
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/keepalived
/usr/local/docker/keepalived
├── node-1
│   ├── check-haproxy.sh
│   ├── config
│   │   └── keepalived.conf
│   └── docker-compose.yml
├── node-2
│   ├── check-haproxy.sh
│   ├── config
│   │   └── keepalived.conf
│   └── docker-compose.yml
└── node-3
    ├── check-haproxy.sh
    ├── config
    │   └── keepalived.conf
    └── docker-compose.yml
```

复制 192.168.5.163(manager) 的 keepalived 目录到 192.168.5.164(worker):

```bash
# scp -r /usr/local/docker/keepalived root@192.168.5.164:/usr/local/docker
```

复制 192.168.5.163(manager) 的 keepalived 目录到 192.168.5.165(worker):

```bash
# scp -r /usr/local/docker/keepalived root@192.168.5.165:/usr/local/docker
```

## 启动 docker-compose

在宿主机 192.168.5.163 上启动 keepalived 服务:

```bash
# docker-compose -p keepalived -f /usr/local/docker/keepalived/node-1/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container keepalived_1  Started                                                                               0.2s

# docker ps -a | grep keepalived
cbc2cec6a3f0   osixia/keepalived:latest             "/container/tool/run"     23 seconds ago   Up 23 seconds                                  keepalived_1

# ip addr show ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:8a:8a:d3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.163/24 brd 192.168.5.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.5.100/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::42de:6cd4:dde4:b13e/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

在宿主机 192.168.5.164 上启动 keepalived 服务:

```bash
# docker-compose -p keepalived -f /usr/local/docker/keepalived/node-2/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container keepalived_2  Started                                                                               0.3s

# docker ps -a | grep keepalived
533e677d289f   osixia/keepalived:latest             "/container/tool/run"     54 seconds ago   Up 53 seconds                             keepalived_2

# ip addr show ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:18:48:fd brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.164/24 brd 192.168.5.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::359b:b3ea:7251:4819/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

在宿主机 192.168.5.165 上启动 keepalived 服务:

```bash
# docker-compose -p keepalived -f /usr/local/docker/keepalived/node-3/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container keepalived_3  Started                                                                               0.2s

# docker ps -a | grep keepalived
51902886dce0   osixia/keepalived:latest             "/container/tool/run"     17 seconds ago   Up 17 seconds                             keepalived_3

# ip addr show ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:bc:27:63 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.165/24 brd 192.168.5.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::aa8c:7c8e:4e1f:83/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

### 连接 mysql

连接 ```$VIP:13306``` :

```bash
# mysql -uroot -proot -h192.168.5.100 -P13306
```

### 访问 haproxy 监控界面

连接 ```$VIP:18081``` :

```bash
# curl -XGET http://192.168.5.100:18081/monitor
```

### FAQ

#### IPVS: Can't initialize ipvs: Protocol not available

- 解决方法

   需要安装 ```ipvsadm```，命令行:
   ```bash
   # yum install -y ipvsadm
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
