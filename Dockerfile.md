# Dockerfile

Dockerfile由一行行命令语句组成，并且支持用“#”开头作为注释，一般的，Dockerfile分为四部分：基础镜像信息，维护者信息，镜像操作指令和容器启动时执行的指令。

## FROM

```
格式：FROM <image>或 FROM <image>:<tag>

第一条指令必须为FROM指令，并且，如果在同一个Dockerfile中创建多个镜像时，可以使用多个FROM指令（每个镜像一次）。


镜像仓库详见docker hub。
```

## LABEL

```
格式：LABEL <key>=<value> <key>=<value> <key>=<value> ...

LABEL 为镜像增加元数据，一个 LABEL 是键值对，多个键值对之间使用空格分开，命令换行时是使用反斜杠 \
```

示例:

```
LABEL maintainer="zhy"
```

## RUN

```
格式：RUN <command> 或 RUN ["", "", ""]

每条指令将在当前镜像基础上执行，并提交为新的镜像。（可以用“\”换行）
```

## EXPOSE

```
格式：EXPOSE <port>  [ <port> ...]

告诉Docker服务端暴露端口，在容器启动时需要通过 -p 做端口映射
```

## ENV

```
格式：ENV <key> <value>

指定环境变量，会被RUN指令使用，并在容器运行时保存
```

## ADD

```
格式：ADD  <src>  <dest>

复制指定的<src>到容器的<dest>中，<src>可以是Dockerfile所在的目录的一个相对路径；可以是URL，也可以是tar.gz（自动解压）
```

## COPY

```
格式：COPY <src>  <dest>

复制本地主机的 <src> （ 为 Dockerfile 所在目录的相对路径）到容器中的 <dest> （当使用本地目录为源目录时，推荐使用 COPY）
```

***注: ADD 与 COPY 功能相似，但是 ADD 可以自动解压 tar.gz***

## CMD

```
格式：CMD ["","",""]

指定启动容器时执行的命令，每个Dockerfile只能有一条CMD指令，如果指定了多条指令，则最后一条执行。（会被启动时 docker run 指定的命令覆盖）
```

## ENTRYPOINT

```
格式：ENTRYPOINT ["","",""]

配置容器启动后执行的命令，并且不可被 docker run 提供的参数覆盖。（每个 Dockerfile 中只能有一个 ENTRYPOINT ，当指定多个时，只有最后一个起效）
```

***注: CMD 与 ENTRYPOINT 功能相似，但是 CMD 会被启动时 docker run 指定的命令覆盖, ENTRYPOINT 不会被 docker run 提供的参数覆盖***

## VOLUME

```
格式：VOLUME ["/data"]

创建一个匿名数据卷挂载点，这里的 ```/data``` 目录会在运行时自动挂载为匿名卷，任何向 ```/data``` 中写入的信息都不会记录进容器存储层，从而保证了容器存储层的无状态化。
```

***注: VOLUME 指令只是声明容器中的目录为匿名卷，但是并没有将匿名卷绑定到宿主机指定目录的功能。即 volume只是指定了一个容器目录，即使在启动时，用户忘记指定 -v 参数，也可以保证不会把数据写入容器的存储层。***

## USER

```
格式：USER daemon

指定运行容器时的用户名或 UID，后续的 RUN 也会使用指定用户。
```

## WORKDIR

```
格式：WORKDIR /path/to/workdir

类似CD指令，为后续的 RUN 、 CMD 、 ENTRYPOINT 指令配置工作目录。（可以使用多个 WORKDIR 指令，后续命令如果参数是相对路径， 则会基于之前命令指定的路径）
```

## ONBUILD

```
格式：ONBUILD [INSTRUCTION]

配置当所创建的镜像作为其它新创建镜像的基础镜像时，所执行的操作指令
```
