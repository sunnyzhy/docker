# docker-elasticsearch 集群 (host)

## 前言

- elasticsearch 版本: ```docker.elastic.co/elasticsearch/elasticsearch:7.12.1```

- 网络配置: 驱动类型为 overlay

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- 容器与宿主机映射
    |容器名称|宿主机IP|端口映射(宿主机端口:容器端口)|宿主机IP|挂载(宿主机的配置文件:容器的配置文件)|
    |--|--|--|--|--|
    |elasticsearch_1|192.168.5.163|9200:9200<br />9300:9300|/usr/local/docker/elasticsearch/node-1/certs/elastic-certificates.p12:/usr/share/elasticsearch/config/certs/elastic-certificates.p12<br />/usr/local/docker/elasticsearch/node-1/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/node-1/logs:/usr/share/elasticsearch/logs|
    |elasticsearch_2|192.168.5.164|9201:9200<br />9301:9300|/usr/local/docker/elasticsearch/node-2/certs/elastic-certificates.p12:/usr/share/elasticsearch/config/certs/elastic-certificates.p12<br />/usr/local/docker/elasticsearch/node-2/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/node-2/logs:/usr/share/elasticsearch/logs|
    |elasticsearch_3|192.168.5.165|9202:9200<br />9302:9300|/usr/local/docker/elasticsearch/node-3/certs/elastic-certificates.p12:/usr/share/elasticsearch/config/certs/elastic-certificates.p12<br />/usr/local/docker/elasticsearch/node-3/data:/usr/share/elasticsearch/data<br />/usr/local/docker/elasticsearch/node-3/logs:/usr/share/elasticsearch/logs|

- docker.elastic.co/elasticsearch/elasticsearch:7.12.1 镜像配置文件 elasticsearch.yml
    ```yml
    cluster.name: "docker-cluster"
    network.host: 0.0.0.0
    ```

- 单节点启动:
    ```bash
    # docker run -d --name elasticsearch --net elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.12.1
    ```

## 拉取 elasticsearch 镜像

```bash
# docker pull docker.elastic.co/elasticsearch/elasticsearch:7.12.1

# docker images | grep elasticsearch
REPOSITORY                                      TAG       IMAGE ID       CREATED         SIZE
docker.elastic.co/elasticsearch/elasticsearch   7.12.1    41dc8ea0f139   14 months ago   851MB
```

## 生成证书

### 创建网络

```bash
# docker network create elasticsearch --subnet 192.168.3.0/24 --gateway=192.168.3.1
```

### 启动 elasticsearch 容器

```bash
# docker run -d --name elasticsearch --net elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.12.1
```

### 进入 elasticsearch 容器

```bash
# docker exec -it -uroot elasticsearch /bin/sh
```

### 生成 ca

***此处密码输入为 ```elastic```***

```bash
sh-4.4# ./bin/elasticsearch-certutil ca
This tool assists you in the generation of X.509 certificates and certificate
signing requests for use with SSL/TLS in the Elastic stack.

The 'ca' mode generates a new 'certificate authority'
This will create a new X.509 certificate and private key that can be used
to sign certificate when running in 'cert' mode.

Use the 'ca-dn' option if you wish to configure the 'distinguished name'
of the certificate authority

By default the 'ca' mode produces a single PKCS#12 output file which holds:
    * The CA certificate
    * The CA's private key

If you elect to generate PEM format certificates (the -pem option), then the output will
be a zip file containing individual files for the CA certificate and private key

Please enter the desired output file [elastic-stack-ca.p12]: 
Enter password for elastic-stack-ca.p12 : 
```

### 生成 cert

***此处密码输入为 ```elastic```***

```bash
sh-4.4# ./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
This tool assists you in the generation of X.509 certificates and certificate
signing requests for use with SSL/TLS in the Elastic stack.

The 'cert' mode generates X.509 certificate and private keys.
    * By default, this generates a single certificate and key for use
       on a single instance.
    * The '-multiple' option will prompt you to enter details for multiple
       instances and will generate a certificate and key for each one
    * The '-in' option allows for the certificate generation to be automated by describing
       the details of each instance in a YAML file

    * An instance is any piece of the Elastic Stack that requires an SSL certificate.
      Depending on your configuration, Elasticsearch, Logstash, Kibana, and Beats
      may all require a certificate and private key.
    * The minimum required value for each instance is a name. This can simply be the
      hostname, which will be used as the Common Name of the certificate. A full
      distinguished name may also be used.
    * A filename value may be required for each instance. This is necessary when the
      name would result in an invalid file or directory name. The name provided here
      is used as the directory name (within the zip) and the prefix for the key and
      certificate files. The filename is required if you are prompted and the name
      is not displayed in the prompt.
    * IP addresses and DNS names are optional. Multiple values can be specified as a
      comma separated string. If no IP addresses or DNS names are provided, you may
      disable hostname verification in your SSL configuration.

    * All certificates generated by this tool will be signed by a certificate authority (CA)
      unless the --self-signed command line option is specified.
      The tool can automatically generate a new CA for you, or you can provide your own with
      the --ca or --ca-cert command line options.

By default the 'cert' mode produces a single PKCS#12 output file which holds:
    * The instance certificate
    * The private key for the instance certificate
    * The CA certificate

If you specify any of the following options:
    * -pem (PEM formatted output)
    * -keep-ca-key (retain generated CA key)
    * -multiple (generate multiple certificates)
    * -in (generate certificates from an input file)
then the output will be be a zip file containing individual certificate/key files

Enter password for CA (elastic-stack-ca.p12) : 
Please enter the desired output file [elastic-certificates.p12]: 
Enter password for elastic-certificates.p12 : 

Certificates written to /usr/share/elasticsearch/elastic-certificates.p12

This file should be properly secured as it contains the private key for 
your instance.

This file is a self contained file and can be copied and used 'as is'
For each Elastic product that you wish to configure, you should copy
this '.p12' file to the relevant configuration directory
and then follow the SSL configuration instructions in the product guide.

For client applications, you may only need to copy the CA certificate and
configure the client to trust this certificate.
```

### 查看生成的证书

```bash
sh-4.4# ls /usr/share/elasticsearch | grep elastic-
elastic-certificates.p12
elastic-stack-ca.p12
```

***其中 elastic-certificates.p12 就是需要在配置文件里使用的。***

### 拷贝证书文件到宿主机

```bash
# docker cp elasticsearch:/usr/share/elasticsearch/elastic-certificates.p12 /usr/local/docker/

# ls /usr/local/docker | grep elastic-certificates.p12
elastic-certificates.p12
```

### 删除 elasticsearch 容器

```bash
# docker stop elasticsearch

# docker rm elasticsearch

# docker network rm elasticsearch
```

## 在宿主机上创建 elasticsearch 目录

```bash
# mkdir -p /usr/local/docker

# vim /usr/local/docker/docker-compose.sh
```

```sh
#!/bin/sh

for index in $(seq 1 3);
do
    mkdir -p /usr/local/docker/elasticsearch/node-${index}/{certs,data,logs}
    
    cp /usr/local/docker/elastic-certificates.p12 /usr/local/docker/elasticsearch/node-${index}/certs

    chmod 777 /usr/local/docker/elasticsearch/node-${index}/{certs/elastic-certificates.p12,data,logs}
done
```

```bash
# chmod +x /usr/local/docker/docker-compose.sh

# /usr/local/docker/docker-compose.sh >/dev/null 2>&1

# tree /usr/local/docker/elasticsearch
/usr/local/docker/elasticsearch
├── node-1
│   ├── certs
│   │   └── elastic-certificates.p12
│   ├── data
│   └── logs
├── node-2
│   ├── certs
│   │   └── elastic-certificates.p12
│   ├── data
│   └── logs
└── node-3
    ├── certs
    │   └── elastic-certificates.p12
    ├── data
    └── logs
```

复制 192.168.5.163(manager) 的 elasticsearch 目录到 192.168.5.164(worker):

```bash
# scp -r /usr/local/docker/elasticsearch root@192.168.5.164:/usr/local/docker
```

复制 192.168.5.163(manager) 的 elasticsearch 目录到 192.168.5.165(worker):

```bash
# scp -r /usr/local/docker/elasticsearch root@192.168.5.165:/usr/local/docker
```

## 配置 docker-compose.yml

![elasticsearch 集群](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html?baymax=rec&rogue=pop-1&elektra=guide 'elasticsearch 集群')

在宿主机 192.168.5.163(manager) 上执行以下命令:

```bash
# vim /usr/local/docker/docker-compose.yml
```

```yml
version: '3.9'

services:
  elasticsearch_1:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.12.1
    volumes:
      - /usr/local/docker/elasticsearch/node-1/certs/elastic-certificates.p12:/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - /usr/local/docker/elasticsearch/node-1/data:/usr/share/elasticsearch/data
      - /usr/local/docker/elasticsearch/node-1/logs:/usr/share/elasticsearch/logs
      - /etc/timezone:/etc/timezone
      - /etc/localtime:/etc/localtime
    ports:
      - 9200:9200
      - 9300:9300
    environment:
      - node.name=es01
      - cluster.name=es-cluster
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=elasticsearch_2,elasticsearch_3
      - ELASTIC_PASSWORD=elastic
      - bootstrap.memory_lock=true
      - network.publish_host=elasticsearch_1
      - xpack.security.enabled=true
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.keystore.type=PKCS12
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.type=PKCS12
      - xpack.security.transport.ssl.keystore.password=elastic
      - xpack.security.transport.ssl.truststore.password=elastic
      - xpack.security.audit.enabled=true
    deploy:
      resources:
        limits:
          memory: 4G
      replicas: 1
      placement:
        constraints:
          - node.hostname == centos-docker-163
          - node.role == manager
    networks:
      - iot
    ulimits:
      memlock:
        soft: -1
        hard: -1
  elasticsearch_2:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.12.1
    volumes:
      - /usr/local/docker/elasticsearch/node-2/certs/elastic-certificates.p12:/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - /usr/local/docker/elasticsearch/node-2/data:/usr/share/elasticsearch/data
      - /usr/local/docker/elasticsearch/node-2/logs:/usr/share/elasticsearch/logs
      - /etc/timezone:/etc/timezone
      - /etc/localtime:/etc/localtime
    ports:
      - 9201:9200
      - 9301:9300
    environment:
      - node.name=es02
      - cluster.name=es-cluster
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=elasticsearch_1,elasticsearch_3
      - ELASTIC_PASSWORD=elastic
      - bootstrap.memory_lock=true
      - network.publish_host=elasticsearch_2
      - xpack.security.enabled=true
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.keystore.type=PKCS12
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.type=PKCS12
      - xpack.security.transport.ssl.keystore.password=elastic
      - xpack.security.transport.ssl.truststore.password=elastic
      - xpack.security.audit.enabled=true
    deploy:
      resources:
        limits:
          memory: 4G
      replicas: 1
      placement:
        constraints:
          - node.hostname == centos-docker-164
          - node.role == worker
    networks:
      - iot
    ulimits:
      memlock:
        soft: -1
        hard: -1
  elasticsearch_3:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.12.1
    volumes:
      - /usr/local/docker/elasticsearch/node-3/certs/elastic-certificates.p12:/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - /usr/local/docker/elasticsearch/node-3/data:/usr/share/elasticsearch/data
      - /usr/local/docker/elasticsearch/node-3/logs:/usr/share/elasticsearch/logs
      - /etc/timezone:/etc/timezone
      - /etc/localtime:/etc/localtime
    ports:
      - 9202:9200
      - 9302:9300
    environment:
      - node.name=es03
      - cluster.name=es-cluster
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=elasticsearch_1,elasticsearch_2
      - ELASTIC_PASSWORD=elastic
      - bootstrap.memory_lock=true
      - network.publish_host=elasticsearch_3
      - xpack.security.enabled=true
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.keystore.type=PKCS12
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.type=PKCS12
      - xpack.security.transport.ssl.keystore.password=elastic
      - xpack.security.transport.ssl.truststore.password=elastic
      - xpack.security.audit.enabled=true
    deploy:
      resources:
        limits:
          memory: 4G
      replicas: 1
      placement:
        constraints:
          - node.hostname == centos-docker-165
          - node.role == worker     
    networks:
      - iot
    ulimits:
      memlock:
        soft: -1
        hard: -1

networks:
  iot:
    driver: overlay
    external: true
```

## 启动 docker-compose

在宿主机 192.168.5.163(manager) 上启动 elasticsearch 服务:

```bash
# docker stack deploy -c /usr/local/docker/docker-compose.yml iot
Creating service iot_elasticsearch_2
Creating service iot_elasticsearch_3
Creating service iot_elasticsearch_1

# docker stack ps iot | grep elasticsearch
hr4xpduam8w8   iot_elasticsearch_1.1   docker.elastic.co/elasticsearch/elasticsearch:7.12.1   centos-docker-163   Running         Running 17 seconds ago                                          
pk2o18eegfhl   iot_elasticsearch_2.1   docker.elastic.co/elasticsearch/elasticsearch:7.12.1   centos-docker-164   Running         Running 12 seconds ago                                          
c6cb3kcheaji   iot_elasticsearch_3.1   docker.elastic.co/elasticsearch/elasticsearch:7.12.1   centos-docker-165   Running         Running 23 seconds ago      

# docker service ls | grep elasticsearch
c8vaq7bj7fsi   iot_elasticsearch_1   replicated   1/1        docker.elastic.co/elasticsearch/elasticsearch:7.12.1   *:9200->9200/tcp, *:9300->9300/tcp
twekxa6r25aa   iot_elasticsearch_2   replicated   1/1        docker.elastic.co/elasticsearch/elasticsearch:7.12.1   *:9201->9200/tcp, *:9301->9300/tcp
bmljvdzu0894   iot_elasticsearch_3   replicated   1/1        docker.elastic.co/elasticsearch/elasticsearch:7.12.1   *:9202->9200/tcp, *:9302->9300/tcp
```

## 查看 elasticsearch 容器

查看宿主机 192.168.5.163 的 elasticsearch 容器:

```bash
# docker ps | grep elasticsearch
5a532e0f2aa9   docker.elastic.co/elasticsearch/elasticsearch:7.12.1   "/bin/tini -- /usr/l…"   32 seconds ago   Up 26 seconds           9200/tcp, 9300/tcp             iot_elasticsearch_1.1.p161kmhg1tmy7yb2e9ta4j37j
```

查看宿主机 192.168.5.164 的 elasticsearch 容器:

```bash
# docker ps | grep elasticsearch
9257ab0f54b2   docker.elastic.co/elasticsearch/elasticsearch:7.12.1   "/bin/tini -- /usr/l…"   About a minute ago   Up About a minute               9200/tcp, 9300/tcp        iot_elasticsearch_2.1.lndrs4zmbvyx98swndsg620gu
```

查看宿主机 192.168.5.165 的 elasticsearch 容器:

```bash
# docker ps | grep elasticsearch
bd34dd76761a   docker.elastic.co/elasticsearch/elasticsearch:7.12.1   "/bin/tini -- /usr/l…"   About a minute ago   Up About a minute   9200/tcp, 9300/tcp        iot_elasticsearch_3.1.igsgl7xhgdotcrte6m0cezefj
```

## 访问 elasticsearch

访问 ```https://$HOST_IP:[9200|9201|9202]/_cat/nodes?v```，比如: ```https://192.168.5.163:9200/_cat/nodes?v``` :

```bash
# curl -XGET --insecure -u elastic:elastic http://192.168.5.163:9200/_cat/nodes?v
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
10.0.3.107           21          61   9    0.31    1.26     1.08 cdfhilmrstw -      es01
10.0.3.112           19          60  16    1.19    2.06     1.61 cdfhilmrstw -      es02
10.0.3.102            9          61  22    0.87    1.74     1.55 cdfhilmrstw *      es03
```

## FAQ

### max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

- 原因
   系统虚拟内存默认最大映射数为 65530，无法满足 ES 要求，需要把系统虚拟内存调整为 262144 以上。

- 解决办法
    ```bash
    # vim /etc/sysctl.conf
    vm.max_map_count = 262144

    # sysctl -p
    ```
