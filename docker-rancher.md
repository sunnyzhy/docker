# docker-rancher

## 前言

- rancher 版本: 最新版本

- 网络配置: 驱动类型为 bridge，名称为 rancher

- 容器与宿主机映射
    |容器名称|容器IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |rancher|-|8000:80<br />4430:443|192.168.204.107||

## 拉取 rancher 镜像

```bash
# docker pull rancher/rancher

# docker images | grep rancher
rancher/rancher                                    latest        f9e320b7e19c   13 months ago   1.16GB
```

## 创建网络

```bash
# docker network create rancher --subnet 192.168.20.0/24 --gateway=192.168.20.1

# docker network ls | grep rancher
9b09b03a6e6c   rancher   bridge    local

# docker network inspect rancher
[
    {
        "Name": "rancher",
        "Id": "10a091f54ddeab0546056664e4286acd2cf873c26564cc9d6bf15d477b30f6ba",
        "Created": "2023-02-07T16:43:14.197710424+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.20.0/24",
                    "Gateway": "192.168.20.1"
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
            "71d3d3af25d4b8773560fe2cd2e9fd0f19e346f94e813586374eed9d170f71eb": {
                "Name": "rancher",
                "EndpointID": "4537fab4c4cb869dac3692e446d682b7f463b6ed9eb12cb32642578f300da530",
                "MacAddress": "02:42:c0:a8:14:02",
                "IPv4Address": "192.168.20.2/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

## 在宿主机上创建 rancher 目录

```bash
# mkdir -p /usr/local/docker/rancher
```

## 配置 docker-compose.yml

```bash
# vim /usr/local/docker/rancher/docker-compose.yml
version: '3.9'

services:
 rancher:
  image: rancher/rancher
  container_name: rancher
  restart: unless-stopped
  privileged: true
  ports:
   - 8000:80
   - 4430:443
  networks:
   - rancher
networks:
  rancher:
    name: rancher
```

## 启动 docker-compose

```bash
# docker stop rancher

# docker rm rancher

# docker-compose -f /usr/local/docker/rancher/docker-compose.yml up -d
[+] Running 1/1
 ⠿ Container rancher  Started                                                                                                     4.3s

# docker ps | grep rancher
71d3d3af25d4   rancher/rancher   "entrypoint.sh"   50 seconds ago   Up 43 seconds   0.0.0.0:8000->80/tcp, :::8000->80/tcp, 0.0.0.0:4430->443/tcp, :::4430->443/tcp   rancher
```

## 访问 rancher

1. 在浏览器的地址栏输入: ```https://192.168.204.107:4430/```
2. 如果是首次登录，需要先提取登录密码(默认账号是 ```admin```):
   - docker:
      ```bash
      docker logs <container-id> 2>&1 | grep "Bootstrap Password:"
      ```
   - Helm:
      ```bash
      kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
      ```
3. 首次登录之后，可以修改密码，且密码必须是 12 个字符以上
4. 选择 ```导入已有集群```，创建集群名称，查看注册集群的提示信息:
   ```
   Run the kubectl command below on an existing Kubernetes cluster running a supported Kubernetes version to import it into Rancher:
   kubectl apply -f https://192.168.204.107:4430/v3/import/x5lng7ctj68vbfh9nn2xwbjcbmnxc8t667k9nkt9j7szcpgqtfqnw5_c-m-9wxnpnxv.yaml
   
   If you get a "certificate signed by unknown authority" error, your Rancher installation has a self-signed or untrusted SSL certificate. Run the command below    instead to bypass the certificate verification:
   curl --insecure -sfL https://192.168.204.107:4430/v3/import/x5lng7ctj68vbfh9nn2xwbjcbmnxc8t667k9nkt9j7szcpgqtfqnw5_c-m-9wxnpnxv.yaml | kubectl apply -f -
   
   If you get permission errors creating some of the resources, your user may not have the cluster-admin role. Use this command to apply it:
   kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user <your username from your kubeconfig>
    ```
5. 在 Kubernetes 集群的 master 主节点中执行如下命令:
   ```bash
   curl --insecure -sfL https://192.168.204.107:4430/v3/import/x5lng7ctj68vbfh9nn2xwbjcbmnxc8t667k9nkt9j7szcpgqtfqnw5_c-m-9wxnpnxv.yaml | kubectl apply -f -
   ```
