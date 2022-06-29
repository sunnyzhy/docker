# docker 镜像导出与导入

## 导出

- 命令格式

    ```bash
    # docker save -o 文件名  镜像
    ```
    或
    ```bash
    # docker save > 文件名  镜像
    ```

- 示例

    ```bash
    # docker save -o /usr/local/docker/images/nginx.tar nginx:latest
    ```
    ```bash
    # docker save > /usr/local/docker/images/nginx.tar nginx:latest
    ```

## 导入

- 命令格式

    ```bash
    # docker load --input 文件名
    ```
    或
    ```bash
    # docker load < 文件名
    ```

- 示例

    ```bash
    # docker load --input /usr/local/docker/images/nginx.tar
    ```
    ```bash
    # docker load < /usr/local/docker/images/nginx.tar
    ```

## 批量导出

- 批量导出所有的镜像
    ```bash
    # docker save $(docker images | grep -v REPOSITORY | awk 'BEGIN{OFS=":";ORS=" "}{print $1,$2}') -o images.tar
    ```

- 批量导出以 ```hello-``` 开头的镜像
    ```bash
    docker save $(docker images | grep hello- | awk 'BEGIN{OFS=":";ORS=" "}{print $1,$2}') -o images.tar
    ```

- 批量导出不以 ```hello-``` 开头的镜像
    ```bash
    docker save $(docker images | grep -v REPOSITORY | grep -v hello- | awk 'BEGIN{OFS=":";ORS=" "}{print $1,$2}') -o images.tar
    ```

## 批量导入

```bash
docker load -i images.tar
```
