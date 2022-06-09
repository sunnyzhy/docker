# docker-compose.yml 详解

## 前言

docker-compose.yml 组成一个 project ，project 里包括多个 service，每个 service 定义了容器运行的镜像（或构建镜像），网络端口，文件挂载，参数，依赖等，每个 service 可包括同一个镜像的多个容器实例。

## docker-compose.yml 详解

```yml
version:                  # 指定 docker-compose.yml 文件的版本
services:                 # 定义所有的服务集合
  service_name:           # 自定义服务名称
    build:                # 指定包含构建上下文的路径, 或作为一个对象，该对象具有 context 和指定的 dockerfile 文件以及 args 参数值
        context:          # context: 指定 Dockerfile 文件所在的路径
        dockerfile:       # dockerfile: 指定 context 指定的目录下面的 Dockerfile 的名称(默认为 Dockerfile)
        args:             # args: Dockerfile 在 build 过程中需要的参数，等同于 docker container build --build-arg
        cache_from:       # 指定缓存的镜像列表，等同于 docker container build --cache_from
        labels:           # 设置镜像的元数据，等同于 docker container build --labels
        shm_size:         # 设置容器 /dev/shm 分区的大小，等同于 docker container build --shm-size
    command:              # 覆盖容器启动后默认执行的命令, 支持 shell 格式和 [] 格式
                          示例:
                          command: bundle exec thin -p 3000
                          ----------------------------------
                          command: [bundle,exec,thin,-p,3000]

    container_name:       # 指定容器的名称，等同于 docker run --name
    depends_on:           # 定义服务创建与移除的顺序
                          在下述示例中:
                              - 创建服务的时候，先创建 db 和 redis 服务，再创建 web 服务
                              - 删除服务的时候，先删除 web 服务，再删除 db 和 redis 服务
                          示例:
                          services:
                            web:
                              build: .
                              depends_on:
                                - db
                                - redis
                            redis:
                              image: redis
                            db:
                              image: postgres
    devices:              # 指定设备映射列表，等同于 docker run --device
                          格式: HOST_PATH:CONTAINER_PATH[:CGROUP_PERMISSIONS]
                          示例:
                          devices:
                            - "/dev/ttyUSB0:/dev/ttyUSB0"
                            - "/dev/sda:/dev/xvda:rwm"

    entrypoint:           # 覆盖容器的默认 entrypoint 指令，等同于 docker run --entrypoint
                          示例:
                          entrypoint: /code/entrypoint.sh
                          ----------------------------------
                          entrypoint:
                            - php
                            - -d
                            - zend_extension=/usr/local/lib/php/extensions/no-debug-non-zts-20100525/xdebug.so
                            - -d
                            - memory_limit=-1
                            - vendor/bin/phpunit

    env_file:             # 多环境情况下，为容器指定环境变量的配置文件, 可以是单个值或者一个文件列表
                          示例:
                          env_file: .env
                          ----------------------------------
                          env_file:
                            - ./a.env
                            - ./b.env

    environment:          # 设置环境变量， environment 的值可以覆盖 env_file 的值，可以是 Map 或者数组，等同于 docker run --env
                          示例:
                          environment:
                            RACK_ENV: development
                            SHOW: "true"
                            USER_INPUT:
                          ----------------------------------
                          environment:
                            - RACK_ENV=development
                            - SHOW=true
                            - USER_INPUT
                            
    expose:               # 暴露端口, 但是不能和宿主机建立映射关系, 类似于 Dockerfile 的 EXPOSE 指令
                          示例:
                          expose:
                            - "3000"
                            - "8000"

    external_links:       # 连接不在 compose 管理的容器
                          示例:
                          external_links:
                            - redis
                            - database:mysql
                            - database:postgresql

    extra_hosts:          # 把 hostname 映射到容器的网络配置 (/etc/hosts for Linux) 中，等同于 docker run --add-host
                          示例:
                          extra_hosts:
                            - "somehost:162.242.195.82"
                            - "otherhost:50.31.209.229"

    healthcheck:          # 定义容器健康状态检查, 类似于 Dockerfile 的 HEALTHCHECK 指令
        test:             # 检查容器检查状态的命令, 该选项必须是一个字符串或者列表, 第一项必须是 NONE, CMD 或 CMD-SHELL, 如果其是一个字符串则相当于 CMD-SHELL 加该字符串
                          示例:
                          test: ["CMD", "curl", "-f", "http://localhost"]
                          test: ["CMD-SHELL", "curl -f http://localhost || exit 1"]
                          test: curl -f https://localhost || exit 1

        interval:         # 每次检查之间的间隔时间
        timeout:          # 运行命令的超时时间
        retries:          # 重试次数
        start_period:     # 定义容器启动时间间隔
        disable:          # true/false, 表示是否禁用健康状态检测和　test: NONE 相同
                          示例:
                          healthcheck:
                            test: ["CMD", "curl", "-f", "http://localhost"]
                            interval: 1m30s
                            timeout: 10s
                            retries: 3
                            start_period: 40s

    image:                # 指定 docker 镜像, 可以是远程仓库镜像、本地镜像
                          格式: [<registry>/][<project>/]<image>[:<tag>|@<digest>]
                          示例:
                          image: redis
                          image: redis:5
                          image: redis@sha256:0ed5d5928d4737458944eb604cc8509e245c3e19d02ad83935398bc4b991aac7
                          image: library/redis
                          image: docker.io/library/redis
                          image: my_private.registry:5000/redis

    init:                 # true/false, 表示是否在容器中运行一个 init, 它接收信号并传递给进程
                          示例:
                          services:
                            web:
                              image: alpine:latest
                              init: true

    labels:               # 使用 Docker 标签将元数据添加到容器, 与 Dockerfile 中的 LABELS 类似，可以是 Map 或者数组
                          示例:
                          labels:
                            com.example.description: "Accounting webapp"
                            com.example.department: "Finance"
                            com.example.label-with-empty-value: ""
                          ----------------------------------
                          labels:
                            - "com.example.description=Accounting webapp"
                            - "com.example.department=Finance"
                            - "com.example.label-with-empty-value"

    links:                # 链接到其它服务中的容器, 可以通过 links 定义容器的启动顺序
                          示例:
                          services:
                            web:
                              links:
                                - db
                                - db:database
                                - redis

    logging:              # 设置容器日志服务
        driver:           # 指定日志记录驱动程序, 默认 json-file ，等同于 docker run --log-driver
        options:          # 指定日志的相关参数，等同于 docker run --log-opt
            max-size:     # 设置单个日志文件的大小, 当到达这个值后会进行日志滚动操作
            max-file:     # 日志文件保留的数量

    network_mode:         # 指定网络模式，等同于 docker run --net
                          值:
                          - none: 禁用所有容器的网络
                          - host: 使用宿主机网络
                          - service:{name}: 指定网络名称
                          示例:
                          network_mode: "host"
                          network_mode: "none"
                          network_mode: "service:[service name]"

    networks:             # 将容器加入指定网络，等同于 docker network connect
        aliases:          # 同一网络上的容器可以使用服务名称或别名连接到其中一个服务的容器
        ipv4_address:     # IP V4 格式
        ipv6_address:     # IP V6 格式
                          示例:
                          services:
                            some-service:
                              networks:
                                - some-network
                          networks:                         # 申明网络
                            some-network:
                              name: some-network
                              driver: bridge
                              ipam:
                                driver: default
                                config:
                                  - subnet: 172.16.238.0/24

    ports:                # 建立宿主机和容器之间的端口映射关系
                          SHORT 语法格式示例:
                          ports:ports:
                            - "3000"                            # 暴露容器的 3000 端口, 宿主机的端口由 docker 随机映射一个没有被占用的端口
                            - "3000-3005"                       # 暴露容器的 3000 到 3005 端口, 宿主机的端口由 docker 随机映射没有被占用的端口
                            - "8000:8000"                       # 容器的 8000 端口和宿主机的 8000 端口建立映射关系
                            - "9090-9091:8080-8081"
                            - "127.0.0.1:8001:8001"             # 指定映射宿主机的指定地址的
                            - "127.0.0.1:5000-5010:5000-5010"   
                            - "6060:6060/udp"                   # 指定协议

                          LONG 语法格式示例:
                          ports:
                            - target: 80                    # 容器端口
                              host_ip: 127.0.0.1
                              published: 8080               # 宿主机端口
                              protocol: tcp                 # 协议类型
                              mode: host                    # host 在每个节点上发布主机端口,  ingress 对于群模式端口进行负载均衡


    privileged:           # true/false，是否以最高权限运行容器

    restart:              # 定义容器重启策略
                          值:
                          - no: 禁止自动重启容器(默认)
                          - always: 无论如何容器都会重启
                          - on-failure: 如果退出代码指示错误, 就重新启动容器
                          - unless-stopped: 无论退出代码如何，都会重新启动容器
                          示例:
                          restart: "no"
                          restart: always
                          restart: on-failure
                          restart: unless-stopped

    tmpfs:                # 挂载目录到容器中, 作为容器的临时文件系统(等同于 docker run --tmpfs )，可以是单个值或者数组
                          示例:
                          tmpfs: /run
                          ----------------------------------
                          tmpfs:
                            - /run
                            - /tmp

    ulimits:              # 覆盖容器默认的 ulimits
                          示例:
                          ulimits:
                            nproc: 65535
                            nofile:
                              soft: 20000
                              hard: 40000
                          
    volumes:              # 定义容器和宿主机卷的挂载关系
                          SHORT 语法格式示例:
                          services:
                            web:
                              volumes:
                                - /var/lib/mysql                # 映射容器内的 /var/lib/mysql 到宿主机的一个随机目录中
                                - /opt/data:/var/lib/mysql      # 映射容器内的 /var/lib/mysql 到宿主机的 /opt/data
                                - ./cache:/tmp/cache            # 映射容器内的 /var/lib/mysql 到宿主机 compose 文件所在的位置
                                - ~/configs:/etc/configs/:ro    # 映射容器宿主机的目录到容器中去, 权限只读
                                - datavolume:/var/lib/mysql     # datavolume 为宿主机的卷名
                          volumes:
                            datavolume:                         # 如果是 volume 类型，即数据卷类型，就需要在 volumes 里申明卷名
                              name: datavolume
                              external: true

                          LONG 语法格式示例:
                          services:
                            web:
                              volumes:
                                - type: volume                  # mount 的类型, 必须是 bind、volume 或 tmpfs
                                  source: mydata                # 宿主机目录
                                  target: /data                 # 容器目录
                                  volume:                       # 配置额外的选项, 其 key 必须和 type 的值相同
                                    nocopy: true                # volume 额外的选项, 在创建卷时禁用从容器复制数据
                                - type: bind                    # volume 模式只指定容器路径即可, 宿主机路径随机生成; bind 需要指定容器和数据机的映射路径
                                  source: ./static
                                  target: /opt/app/static
                                  read_only: true               # 设置文件系统为只读文件系统
                          volumes:
                            mydata:                             # 如果是 volume 类型，即数据卷类型，就需要在 volumes 里申明卷名
                              name: mydata
                              external: true

networks:                 # 定义 networks 信息
    driver:               # 指定网络模式
                          值:
                          - bridge: Docker 默认使用 bridge 连接单个主机上的网络
                          - overlay: overlay 驱动程序创建一个跨多个节点命名的网络
                          - host: 共享主机网络名称空间，等同于 docker run --net=host
                          - none: 等同于 docker run --net=none
    driver_opts:          # 传递给驱动程序的参数, 这些参数取决于驱动程序
    attachable:           # driver 为 overlay 时使用, 如果设置为 true 则除了服务之外，独立容器也可以附加到该网络; 如果独立容器连接到该网络，则它可以与其他 Docker 守护进程连接到的该网络的服务和独立容器进行通信
    ipam:                 # 自定义 IPAM 配置. 这是一个具有多个属性的对象, 每个属性都是可选的
        driver:           # IPAM 驱动程序, bridge 或者 default
        config:           # 配置项
            - subnet:     # CIDR格式的子网，表示该网络的网段
    external:             # 外部网络, 如果设置为 true 则 docker-compose up 不会尝试创建它, 如果它不存在则引发错误
    name:                 # 为此网络设置名称
                          示例:
                          services:
                            some-service:
                              networks:
                                - some-network
                          networks:                         # 申明网络
                            some-network:
                              name: some-network
                              driver: bridge
                              ipam:
                                driver: default
                                config:
                                  - subnet: 172.16.238.0/24
```
