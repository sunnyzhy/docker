# 可视化管理工具 - Portainer

## 查看 Portainer 镜像

```bash
# docker search portainer | head -n 6
NAME                                   DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
portainer/portainer                    This Repo is now deprecated, use portainer/p…   2213                 
portainer/portainer-ce                 Portainer CE - a lightweight service deliver…   1148                 
portainer/agent                        An agent used to manage all the resources in…   150                  
portainer/templates                    App Templates for Portainer http://portainer…   25                   
portainer/portainer-ee                 Portainer BE - a fully featured service deli…   20

# docker pull portainer/portainer-ce
```

***portainer/portainer 已经被标注为弃用。新版使用 portainer/portainer-ce***

## 安装 Portainer

新建一个卷 portainer_data 来存 Portainer 数据:

```bash
# docker volume create portainer_data
```

### 基于本地容器

启动 Portainer:

```bash
# docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```

***端口 9443 默认启用 HTTPS*** ，用浏览器访问:

```
https://$host:9443
```

### 基于远程容器

修改将要被远程连接的客户机的 docker.service 文件，开通 docker 的远程管理:

```bash
# vim /lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock -H tcp://0.0.0.0:2375

# systemctl daemon-reload

# systemctl restart docker
```

启动 Portainer:

```bash
# docker run -d -p 9000:9000 --name portainer --restart always -v portainer_data:/data portainer/portainer-ce -H tcp://<REMOTE_HOST>:<REMOTE_PORT>
```

***此处的 <REMOTE_PORT> 就是在上述 docker.service 文件里配置的 2375***

用浏览器访问:

```
http://$host:9000
```

## 升级 Portainer

```bash
# docker pull cr.portainer.io/portainer/portainer-ce:xx.xx

# docker stop portainer

# docker rm portainer

# docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:xx.xx
```
