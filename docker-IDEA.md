# IDEA 配置 docker

## 开启 Docker 服务的 2375 端口

```bash
# vim /usr/lib/systemd/system/docker.service
```

在 ```ExecStart=/usr/bin/dockerd``` 后面加上 ```-H tcp://0.0.0.0:2375```:

```service
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock -H tcp://0.0.0.0:2375
```

```bash
# systemctl daemon-reload

# systemctl restart docker
```

## IDEA 安装 Docker 插件

## 在 IDEA 里配置 Docker Server

在 IDEA 里配置 Docker Server，```20.0.0.252```是 Docker 服务所在机器的 IP 地址，如果连接成功会提示 ```Connection successful```:

![Docker Server](./images/idea-01.png 'Docker Server')
