# docker-mysql 主从 (host)

## 前言

- mysql 版本: ```mysql:latest```，此示例为 ```mysql:8```

- 网络配置: 驱动类型为 host

- mysql 主从
    |容器名称|宿主机IP|宿主机的端口|挂载宿主机的配置文件和数据文件|
    |--|--|--|--|
    |mysql-master|192.168.204.107|3306|/usr/local/docker/mysql/master|
    |mysql-slave|192.168.204.107|3307|/usr/local/docker/mysql/slave|

- mysql:8 镜像配置文件 my.cnf
    ```cnf
    [mysqld]
    pid-file        = /var/run/mysqld/mysqld.pid
    socket          = /var/run/mysqld/mysqld.sock
    datadir         = /var/lib/mysql
    secure-file-priv= NULL

    # Custom config should go here
    !includedir /etc/mysql/conf.d/
    ```

## 拉取 mysql 镜像

```bash
# docker pull mysql:latest

# docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
mysql         latest    3218b38490ce   5 months ago   516MB
```

## 在宿主机上创建 mysql 目录

```bash
# mkdir -p /usr/local/docker/mysql

# vim /usr/local/docker/mysql/create-node.sh
```

```sh
#!/bin/sh
mkdir -p /usr/local/docker/mysql/{master,slave}
cd /usr/local/docker/mysql/master
mkdir conf data mysql-files
chmod 777 * -R
cd /usr/local/docker/mysql/master/conf
touch my.cnf
cat << EOF >> my.cnf
[mysqld]
port=3306
server-id=1
log-bin=mysql-bin
EOF
cd /usr/local/docker/mysql/slave
mkdir conf data mysql-files
chmod 777 * -R
cd /usr/local/docker/mysql/slave/conf
touch my.cnf
cat << EOF >> my.cnf
[mysqld]
port=3307
server-id=2
log-bin=mysql-bin
EOF
```

```bash
# chmod +x /usr/local/docker/mysql/create-node.sh

# /usr/local/docker/mysql/create-node.sh

# ls
create-node.sh  master  slave
```

## 配置 docker-compose.yml

```bash
# vim /usr/local/docker/mysql/docker-compose.yml
```

```yml
version: '3.9'

services:
 mysql-master:
  image: mysql:latest
  container_name: mysql-master
  restart: always
  volumes:
   - /usr/local/docker/mysql/master/data:/var/lib/mysql
   - /usr/local/docker/mysql/master/conf/my.cnf:/etc/mysql/my.cnf
   - /usr/local/docker/mysql/master/mysql-files:/var/lib/mysql-files
  environment:
   MYSQL_ROOT_PASSWORD: root
  network_mode: host
 mysql-slave:
  image: mysql:latest
  container_name: mysql-slave
  restart: always
  volumes:
   - /usr/local/docker/mysql/slave/data:/var/lib/mysql
   - /usr/local/docker/mysql/slave/conf/my.cnf:/etc/mysql/my.cnf
   - /usr/local/docker/mysql/master/mysql-files:/var/lib/mysql-files
  environment:
   MYSQL_ROOT_PASSWORD: root
  network_mode: host
```

## 启动 docker-compose

```bash
# docker-compose -f /usr/local/docker/mysql/docker-compose.yml up -d
[+] Running 2/2
 ⠿ Container mysql-master  Started                                                                               0.8s
 ⠿ Container mysql-slave   Started                                                                               0.8s

# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS          PORTS     NAMES
4937b929767f   mysql:latest   "docker-entrypoint.s…"   8 seconds ago   Up 6 seconds              mysql-master
91db51e1cc40   mysql:latest   "docker-entrypoint.s…"   8 seconds ago   Up 7 seconds              mysql-slave
```

## 配置主从同步

### 配置主库

```bash
# docker exec -it mysql-master /bin/sh
# mysql -uroot -proot

mysql> show global variables like 'port';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3306  |
+---------------+-------+

mysql> use mysql;

mysql> create user 'slave'@'%' identified by 'root';

mysql> grant replication slave on *.* to 'slave'@'%';

mysql> flush privileges;

mysql> select * from user where user = 'slave' \G;
Repl_slave_priv: Y
```

***记录主库 File 和 Position 项对应的值:***

```bash
mysql> show master status \G;
*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 873
```

### 配置从库

```bash
# docker exec -it mysql-slave /bin/sh
# mysql -uroot -proot

mysql> show global variables like 'port';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3307  |
+---------------+-------+

mysql> use mysql;

mysql> stop slave;

mysql> reset slave;

mysql> change master to master_host='192.168.204.107',master_port=3306,master_user='slave',master_password='root',master_log_file='mysql-bin.000003',master_log_pos=873;

mysql> start slave;
```

***查看从库状态:***

```bash
mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 192.168.204.107
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 873
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

参数说明:

- ```Slave_IO_State: Waiting for source to send event```: slave 连接到 master 的状态，```Waiting for source to send event``` 表示已经成功连接到 master，正等待二进制日志事件的到达
- ```Master_Log_File: mysql-bin.000003```: 主库 file 项对应的值
- ```Read_Master_Log_Pos: 873```: 主库 position 项对应的值
- ```Slave_IO_Running: Yes```: ```IO/thread``` 是否启动，连接到主库，并读取主库的日志到本地，生成本地日志文件，```Yes``` 表示正常
- ```Slave_SQL_Running: Yes```: ```SQL thread``` 是否启动，读取本地日志文件，并执行日志里的 SQL 命令，```Yes``` 表示正常

## 主从数据同步

1. 登录主库，创建数据库 test:
    ```bash
    # docker exec -it mysql-master /bin/sh
    # mysql -uroot -proot

    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+

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
2. 登录从库，查询数据库:
```bash
# docker exec -it mysql-slave /bin/sh
# mysql -uroot -proot

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
