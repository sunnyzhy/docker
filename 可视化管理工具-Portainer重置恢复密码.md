# 可视化管理工具 - Portainer 重置恢复密码

## 前言

- portainer 版本: ```portainer/helper-reset-password```

## 拉取 portainer 镜像

```bash
# docker pull portainer/helper-reset-password

# docker images | grep portainer
portainer/portainer-ce                          latest    0df02179156a   7 months ago    273MB
portainer/helper-reset-password                 latest    9652765a8b26   2 years ago     6.11MB
```

## 停止 portainer 容器

```bash
# docker ps | grep portainer
b1eb033e8e32   portainer/portainer-ce:latest        "/portainer"              46 hours ago   Up 29 hours   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp, 0.0.0.0:9443->9443/tcp, :::9443->9443/tcp, 9000/tcp   portainer

# docker stop b1eb033e8e32
b1eb033e8e32
```

## 查找 portainer 容器的挂载信息

```bash
# docker inspect b1eb033e8e32 | sed 's/[" ,]//g' | grep Source
Source:portainer_data
Source:/var/run/docker.sock
Source:/var/lib/docker/volumes/portainer_data/_data
Source:/etc/timezone
Source:/etc/localtime
```

此处 portainer 容器的挂载信息为挂载卷 ```portainer_data``` 。

## 重置/恢复密码

```bash
# docker run --rm -v portainer_data:/data portainer/helper-reset-password
2022/07/08 06:24:59 Password succesfully updated for user: admin
2022/07/08 06:24:59 Use the following password to login: Z~8Rm"XC2*0e5@En>6d)\s^91IK4NyMJ
```

此时用户名 ```admin``` 的密码被重置为了 ```Z~8Rm"XC2*0e5@En>6d)\s^91IK4NyMJ```

## 启动容器

```bash
# docker start b1eb033e8e32
b1eb033e8e32
```

## 登录

用浏览器访问 https://$HOST_IP:9443, 输入用户名/密码: admin/Z~8Rm"XC2*0e5@En>6d)\s^91IK4NyMJ:

```
https://192.168.204.107:9443
```

## 修改密码

```左侧菜单 -> Users``` ，点击 ```admin``` 修改密码
