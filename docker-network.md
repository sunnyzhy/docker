# docker - network

## docker network 命令

### 查看网络列表

```bash
# docker network ls
```

### 创建网络

- 不指定驱动类型时，默认为 ```bridge```:
    ```bash
    # docker network create <network-name>
    ```
- 使用参数 ```-d``` 指定驱动类型:
    ```bash
    # docker network create -d overlay <network-name>
    ```

***注: host 类型的网络只能有一个实例，所以自定义的网络不能是 host 类型。***

### 查看网络详情

```bash
# docker network inspect <network-name>
```

### 删除网络

- 删除指定网络:
    ```bash
    # docker network rm <network-name>
    ```
- 删除所有未使用的网络:
    ```bash
    # docker network prune
    ```

### 将容器连接到网络中

```bash
# docker network connect <network-name> <container-id/name>
```

### 从网络中断开容器的链接

```bash
# docker network disconnect <network-name> <container-id/name>
```

## docker 网络介绍

在 Docker 中，默认情况下容器与容器、容器与外部宿主机的网络是隔离开来的。在安装 Docker 的时候，Docker 会创建一个桥接器 docker0 ，通过它才让容器与容器之间、与宿主机之间通信。

```bash
# ifconfig docker0
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:dcff:fe3b:7b6e  prefixlen 64  scopeid 0x20<link>
        ether 02:42:dc:3b:7b:6e  txqueuelen 0  (Ethernet)
        RX packets 12035  bytes 19495058 (18.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 15798  bytes 1644163 (1.5 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

## docker 网络模式

docker 容器的网络有五种模式：

- bridge 模式(默认): ```--net=bridge```
   这是 dokcer 网络的默认模式，为容器创建独立的网络命名空间，容器具有独立的网卡等所有单独的网络栈，是最常用的使用方式。在 ```docker run``` 启动容器的时候，如果不加 ```–net``` 参数，就默认采用这种网络模式。安装完 docker，系统会自动添加一个供 docker 使用的网桥 docker0 ，我们创建一个新的容器时，容器通过 DHCP 获取一个与 docker0 同网段的 IP 地址，并默认连接到 docker0 网桥，以此实现容器与宿主机的网络互通。

- host 模式: ```--net=host```
   Docker 使用了 Linux 的 Namespaces 技术来进行资源隔离，如 PID Namespace 隔离进程，Mount Namespace 隔离文件系统，Network Namespace 隔离网络等。一个 Network Namespace 提供了一份独立的网络环境，包括网卡、路由、Iptable 规则等都与其他的 Network Namespace 隔离。一个 Docker 容器一般会分配一个独立的 Network Namespace 。但如果启动容器的时候使用 host 模式，那么这个容器将不会获得一个独立的 Network Namespace，而是和宿主机共用一个 Network Namespace。容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口。

- none 模式: ```--net=none```
   为容器创建独立网络命名空间，但不为它做任何网络配置，容器中只有 lO ，用户可以在此基础上，对容器网络做任意定制。这个模式下，dokcer 不为容器进行任何网络配置。需要我们自己为容器添加网卡，配置 IP。因此，若想使用 pipework 配置 docker 容器的 ip 地址，必须要在 none 模式下才可以。

- 其他容器模式（container 模式/join 模式）: ```--net=container:NAME_or_ID```
   与 host 模式类似，只是容器将与指定的容器共享网络命名空间。这个模式就是指定一个已有的容器，共享该容器的 IP 和端口。除了网络方面两个容器共享，其他的如文件系统，进程等还是隔离开的。

- 自定义网络: 使用 ```docker network create <network-name>``` 命令创建

## 删除 docker 虚拟网卡

docker 虚拟网卡以 ```br-``` 开头，如:

```bash
# ifconfig
br-601873486f2c: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.7.1  netmask 255.255.255.0  broadcast 192.168.7.255
        ether 02:42:5c:73:cc:82  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

删除 docker 虚拟网卡:

1. 停止虚拟网卡
```bash
# ifconfig br-601873486f2c down
```

2. 删除虚拟网卡
```bash
# brctl delbr br-601873486f2c
```
