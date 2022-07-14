# docker-minio 集群 (host)

## 前言

- minio 版本: ```minio/minio:latest```

- 网络配置: 驱动类型为 overlay

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- minio 集群
    |容器名称|宿主机IP|端口映射(宿主机端口:容器端口)|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|
    |minio_1|192.168.5.164|9000:9000<br />9090:9090|/usr/local/docker/minio/node-1:/data|
    |minio_2|192.168.5.165|9001:9000<br />9091:9090|/usr/local/docker/minio/node-2:/data|
    |minio_3|192.168.5.165|9002:9000<br />9092:9090|/usr/local/docker/minio/node-3:/data|
    |minio_4|192.168.5.164|9003:9000<br />9093:9090|/usr/local/docker/minio/node-4:/data|

- Haproxy
    |容器名称|宿主机IP|端口映射(宿主机端口:容器端口)|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|
    |haproxy_1|192.168.5.163|19000:19000<br />19090:19090<br />18081:18081|/usr/local/docker/haproxy/node-1/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg|
    |haproxy_2|192.168.5.164|19000:19000<br />19090:19090<br />18081:18081|/usr/local/docker/haproxy/node-2/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg|
    |haproxy_3|192.168.5.165|19000:19000<br />19090:19090<br />18081:18081|/usr/local/docker/haproxy/node-3/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg|

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

- 单节点运行
    ```bash
    # docker run -p 9000:9000 -p 9090:9090 \
     --name minio \
     --net minio \
     -d --restart=always \
     -e MINIO_ROOT_USER=admin \
     -e MINIO_ROOT_PASSWORD=password \
     minio/minio server /data --console-address ":9090" --address ":9000"
    ```

## 拉取 minio 镜像

```bash
# docker pull minio/minio:latest

# docker images | grep minio
REPOSITORY                                      TAG       IMAGE ID       CREATED         SIZE
minio/minio                                     latest    e31e0721a96b   5 months ago    406MB
```

## 创建网络

在宿主机 192.168.5.163(manager) 上创建网络:

```bash
# docker network create --driver overlay iot

# docker network ls | grep iot
NETWORK ID     NAME            DRIVER    SCOPE
kdsij9mwppe2   iot             overlay   swarm
```

## 生成密钥

***Access key length should be at least 3, and secret key length at least 8 characters.***

在宿主机 192.168.5.163(manager) 上生成密钥:

```bash
# echo "admin" | docker secret create minio_access_key -

# echo "password" | docker secret create minio_secret_key -
```

## 给集群节点添加标签

在宿主机 192.168.5.163(manager) 上给集群节点添加标签:

```bash
# docker node update --label-add minio1=true --label-add minio4=true centos-docker-164

# docker node update --label-add minio2=true --label-add minio3=true centos-docker-165
```

## 在宿主机上创建 minio 目录

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```sh
#!/bin/sh

for index in $(seq 1 4);
do
    for i in $(seq 1 4);
    do
        mkdir -p /usr/local/docker/minio/node-${index}
        chmod 777 /usr/local/docker/minio/node-${index}
    done
done
```

```bash
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/minio/
/usr/local/docker/minio/
├── node-1
├── node-2
├── node-3
└── node-4
```

复制 192.168.5.163(manager) 的 minio 目录到 192.168.5.164(worker):

```bash
# scp -r /usr/local/docker/minio root@192.168.5.164:/usr/local/docker
```

复制 192.168.5.163(manager) 的 minio 目录到 192.168.5.165(worker):

```bash
# scp -r /usr/local/docker/minio root@192.168.5.165:/usr/local/docker
```

## 配置 docker-compose.yml

[使用 Docker Swarm 部署 MinIO](http://docs.minio.org.cn/docs/master/deploy-minio-on-docker-swarm '使用 Docker Swarm 部署 MinIO')

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# vim /usr/local/docker/docker-compose.yml
```

```yml
version: '3.9'

services:
 minio_1:
   image: minio/minio:latest
   hostname: minio1
   volumes:
     - /usr/local/docker/minio/node-1:/data
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: server --address ":9000" --console-address ":9090" http://minio{1...4}/data
   ports:
     - 9000:9000
     - 9090:9090
   secrets:
     - secret_key
     - access_key
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.labels.minio1==true
   networks:
     - iot
 minio_2:
   image: minio/minio:latest
   hostname: minio2
   volumes:
     - /usr/local/docker/minio/node-2:/data
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: server --address ":9000" --console-address ":9090" http://minio{1...4}/data
   ports:
     - 9001:9000
     - 9091:9090
   secrets:
     - secret_key
     - access_key
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.labels.minio2==true
   networks:
     - iot
 minio_3:
   image: minio/minio:latest
   hostname: minio3
   volumes:
     - /usr/local/docker/minio/node-3:/data
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: server --address ":9000" --console-address ":9090" http://minio{1...4}/data
   ports:
     - 9002:9000
     - 9092:9090
   secrets:
     - secret_key
     - access_key
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.labels.minio3==true
   networks:
     - iot
 minio_4:
   image: minio/minio:latest
   hostname: minio4
   volumes:
     - /usr/local/docker/minio/node-4:/data
     - /etc/timezone:/etc/timezone
     - /etc/localtime:/etc/localtime
   command: server --address ":9000" --console-address ":9090" http://minio{1...4}/data
   ports:
     - 9003:9000
     - 9093:9090
   secrets:
     - secret_key
     - access_key
   deploy:
     replicas: 1
     placement:
       constraints:
         - node.labels.minio4==true
   networks:
     - iot

secrets:
  secret_key:
    external: true
    name: minio_secret_key
  access_key:
    external: true
    name: minio_access_key

networks:
  iot:
    driver: overlay
    external: true
```

### 启动 docker-compose

在宿主机 192.168.5.163(manager) 上启动 minio 服务:

```bash
# docker stack deploy -c /usr/local/docker/docker-compose.yml iot
Creating service iot_minio_2
Creating service iot_minio_3
Creating service iot_minio_4
Creating service iot_minio_1

# docker stack ps iot | grep minio
y6013fo1bfwm   iot_minio_1.1            minio/minio:latest                                     centos-docker-164   Running         Running about a minute ago                                       
eb53mnp6wdeh   iot_minio_2.1            minio/minio:latest                                     centos-docker-165   Running         Running about a minute ago                                       
irjrqtj47jz8   iot_minio_3.1            minio/minio:latest                                     centos-docker-165   Running         Running about a minute ago                                       
rm06uthf54bn   iot_minio_4.1            minio/minio:latest                                     centos-docker-164   Running         Running about a minute ago                                       

# docker service ls | grep minio
mia6lnpvl016   iot_minio_1           replicated   1/1        minio/minio:latest                                     *:9000->9000/tcp, *:9090->9090/tcp
nnn95g32751o   iot_minio_2           replicated   1/1        minio/minio:latest                                     *:9001->9000/tcp, *:9091->9090/tcp
t62bc7c1zizf   iot_minio_3           replicated   1/1        minio/minio:latest                                     *:9002->9000/tcp, *:9092->9090/tcp
ww0hvbj6qpy8   iot_minio_4           replicated   1/1        minio/minio:latest                                     *:9003->9000/tcp, *:9093->9090/tcp
```

## 查看 minio 容器

查看宿主机 192.168.5.164 的 minio 容器:

```bash
# docker ps | grep minio
d79f1a47bacf   minio/minio:latest                                     "/usr/bin/docker-ent…"   2 minutes ago    Up 2 minutes    9000/tcp                               iot_minio_1.1.y6013fo1bfwmm4dtipgkmokfn
e77fae07c34f   minio/minio:latest                                     "/usr/bin/docker-ent…"   2 minutes ago    Up 2 minutes    9000/tcp                               iot_minio_4.1.rm06uthf54bn8u6kti6obsift
```

查看宿主机 192.168.5.165 的 minio 容器:

```bash
# docker ps | grep minio
a19258807e17   minio/minio:latest                                     "/usr/bin/docker-ent…"   2 minutes ago   Up 2 minutes   9000/tcp                               iot_minio_3.1.irjrqtj47jz80yj8dvt3zsm0l
ced1efd71ef6   minio/minio:latest                                     "/usr/bin/docker-ent…"   3 minutes ago   Up 3 minutes   9000/tcp                               iot_minio_2.1.eb53mnp6wdehb3k2fthivvfy2
```

## 访问 dashboard

可以访问任一连接: ```$HOST_IP:9090/9091/9092/9093```，用户名/密码: ```admin/password```:

```
http://192.168.5.163:9090
```

## 安装 Haproxy

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
#负载均衡
listen  minio-tcp
	#访问的IP和端口
	bind  0.0.0.0:19000
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
        server  minio-tcp-1 192.168.5.164:9000 check weight 1 maxconn 2000
        server  minio-tcp-2 192.168.5.165:9001 check weight 1 maxconn 2000
        server  minio-tcp-3 192.168.5.165:9002 check weight 1 maxconn 2000
        server  minio-tcp-4 192.168.5.164:9003 check weight 1 maxconn 2000
        #使用keepalive检测死链
        option  tcpka
listen  minio-http
        #访问的IP和端口
        bind  0.0.0.0:19090
        #网络协议
        mode  http
        #负载均衡算法（轮询算法）
        #轮询算法：roundrobin
        #权重算法：static-rr
        #最少连接算法：leastconn
        #请求源IP算法：source
        balance  roundrobin
        #日志格式
        option  tcplog
        server  minio-http-1 192.168.5.164:9090 check weight 1 maxconn 2000
        server  minio-http-2 192.168.5.165:9091 check weight 1 maxconn 2000
        server  minio-http-3 192.168.5.165:9092 check weight 1 maxconn 2000
        server  minio-http-4 192.168.5.164:9093 check weight 1 maxconn 2000
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

### 启动 docker-compose

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

### 访问 minio 的 dashboard

可以访问任一连接: ```$HOST_IP:19090```，用户名/密码: ```admin/password```:

```
http://192.168.5.163:19090
```

### 访问 haproxy 监控界面

连接 ```$HOST_IP:18081```，用户名/密码: ```admin/admin``` :

```bash
# curl -XGET http://192.168.5.163:18081/monitor
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
#!/bin/bash

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

virtual_server 192.168.5.100 19000 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.163 19000 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 19000
       }
    }
}

virtual_server 192.168.5.100 19090 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.163 19090 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 9090
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
    state MASTER
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

virtual_server 192.168.5.100 19000 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.164 19000 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 19000
       }
    }
}

virtual_server 192.168.5.100 19090 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.164 19090 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 9090
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
    state MASTER
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

virtual_server 192.168.5.100 19000 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.165 19000 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 19000
       }
    }
}

virtual_server 192.168.5.100 19090 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.165 19090 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            delay_before_retry 3
            connect_port 9090
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

count=\`netstat -apn | grep 19000 | wc -l\`
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

### 访问 minio 的 dashboard

连接: ```$VIP:19090```，用户名/密码: ```admin/password```:

```
http://192.168.5.100:19090
```

### 访问 haproxy 监控界面

连接 ```#VIP:18081```，用户名/密码: ```admin/admin``` :

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
