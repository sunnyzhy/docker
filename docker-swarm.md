# docker swarm 集群

## 前言

|物理机IP|角色|
|--|--|
|192.168.5.163|manager|
|192.168.5.164|worker|
|192.168.5.165|worker|

## 创建 swarm 集群

在 192.168.5.163 上初始化集群:

```bash
# docker swarm init --advertise-addr 192.168.5.163
Swarm initialized: current node (0po8w4ww3gwpywxw7mnognizo) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2ryiu75c3eqrbdt1bi1ov5pvxul4aywzpth2qdt6pzxnvd5vzy-aodjy6y3amx1zqql8peyxkzph 192.168.5.163:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

***执行上述命令后，192.168.5.163 会作为管理节点并自动加入到 swarm 集群。创建的集群token，作为集群唯一标识，后续将其他节点加入到该集群都会用到这个 token 值。***

```bash
# docker node ls
ID                            HOSTNAME            STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
0po8w4ww3gwpywxw7mnognizo *   centos-docker-163   Ready     Active         Leader           22.06.0-beta.0
```

***如果忘记了上述命令生成的 token，可以运行如下命令来查看：***

```bash
# docker swarm join-token manager
```

集群初始化之后会自动创建好专用的驱动类型为 overlay 的 ingress 网络:

```bash
# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
d9e7b4ddd54f   bridge    bridge    local
aa3bc2069f02   host      host      local
bfmpfq4xzuzv   ingress   overlay   swarm
b149c59789ce   none      null      local
```

## 添加工作节点到 swarm 集群

在 192.168.5.164 执行以下命令将 192.168.5.164 添加到 swarm 集群:

```bash
# docker swarm join --token SWMTKN-1-2ryiu75c3eqrbdt1bi1ov5pvxul4aywzpth2qdt6pzxnvd5vzy-aodjy6y3amx1zqql8peyxkzph 192.168.5.163:2377
This node joined a swarm as a worker.
```

在 192.168.5.165 执行以下命令将 192.168.5.164 添加到 swarm 集群:

```bash
# docker swarm join --token SWMTKN-1-2ryiu75c3eqrbdt1bi1ov5pvxul4aywzpth2qdt6pzxnvd5vzy-aodjy6y3amx1zqql8peyxkzph 192.168.5.163:2377
This node joined a swarm as a worker.
```

## 查看集群详情

在管理节点上执行以下命令:

```bash
# docker node ls
ID                            HOSTNAME            STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
0po8w4ww3gwpywxw7mnognizo *   centos-docker-163   Ready     Active         Leader           22.06.0-beta.0
hmlxhroizc2phkuf0t0c5me2v     centos-docker-164   Ready     Active                          22.06.0-beta.0
bluq5311ji9n2i16a3ggnk6eg     centos-docker-165   Ready     Active                          22.06.0-beta.0
```
