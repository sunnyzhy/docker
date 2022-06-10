# docker-emqx 集群 (bridge)

## 前言

- emqx 版本: ```emqx/emqx:latest```

- 网络配置: 驱动类型为 bridge，名称为 emqx ，子网掩码为 ```192.168.5.0/24```，网关为 ```192.168.5.1```

- 容器与宿主机映射
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |emqx-1|192.168.5.10|18083:18083<br />1883:1883|192.168.204.107|/usr/local/docker/emqx/node-1/etc:/opt/emqx/etc<br />/usr/local/docker/emqx/node-1/data\:/opt/emqx/data<br />/usr/local/docker/emqx/node-1/log:/opt/emqx/log|
    |emqx-2|192.168.5.11|18084:18083<br />1884:1883|192.168.204.107|/usr/local/docker/emqx/node-2/etc:/opt/emqx/etc<br />/usr/local/docker/emqx/node-2/data\:/opt/emqx/data<br />/usr/local/docker/emqx/node-2/log:/opt/emqx/log|
    |emqx-3|192.168.5.20|18085:18083<br />1885:1883|192.168.204.107|/usr/local/docker/emqx/node-3/etc:/opt/emqx/etc<br />/usr/local/docker/emqx/node-3/data\:/opt/emqx/data<br />/usr/local/docker/emqx/node-3/log:/opt/emqx/log|

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

***从容器里拷贝配置文件到宿主机。***

```bash
# docker run -d --name emqx --privileged=true -p 18083:18083 -p 1883:1883 --network=emqx --ip 192.168.5.10 emqx/emqx:latest

# mkdir -p /usr/local/docker/emqx

# > /usr/local/docker/emqx/create-node.sh

# vim /usr/local/docker/emqx/create-node.sh
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
echo "cluster.static.seeds = emqx-1@192.168.5.10,emqx-2@192.168.5.11,emqx-3@192.168.5.12" >> /usr/local/docker/emqx/node-${index}/etc/emqx.conf
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

可以访问任一链接: ```$HOST:18083/18084/18085```，用户名/密码: ```admin/public```:

```
http://192.168.204.107:18083/
```
