# docker 镜像导出与导入

## 导出

- 命令格式

    ```bash
    # docker export 容器ID > 文件名
    ```
    或
    ```bash
    # docker export 容器名称 > 文件名
    ```

- 示例

    ```bash
    # docker export e7d01f498fd4 > /usr/local/docker/containers/nginx.tar
    ```
    ```bash
    # docker export nginx > /usr/local/docker/containers/nginx.tar
    ```

## 导入

- 命令格式

    ```bash
    # docker import 文件名 容器名称
    ```

- 示例

    ```bash
    # docker import /usr/local/docker/containers/nginx.tar nginx
    ```
