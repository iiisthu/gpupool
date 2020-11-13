# 自定义镜像

我们可以在集群里从自定义镜像拉起 POD，以支持快速的实验环境配置。自定义镜像的思路是**在`ubuntu-tensorflow`、`ubuntu-pytorch`或`orion-client-2.4.2`的基础上，配置自己的环境**。

## 信任集群 Harbor

自定义镜像需要从 Harbor 拉取，因此我们需要在 Docker 中添加对集群 Harbor 的信任。在Mac下用 Docker Desktop 可以直接在客户端里加入`insecure-registries`项：

![Mac docker config](assets/images/mac_docker_config.jpg)

若未使用 Docker Desktop，则在`/etc/docker/deamon.json`中添加（若该文件不存在则创建）：

```json
{
  "insecure-registries": [
    "harbor.iiis.co:30006"
  ]
}
```

## 制作镜像

制作镜像的方式有基于 Dockerfile 和 `docker commit`命令两种形式。我们这里推荐基于 Dockerfile 方式，`docker commit`方式请参考[官方文档](https://docs.docker.com/engine/reference/commandline/commit/)。

> **_NOTE:_** 在[这里]()可以找到我们在这一节所使用的例子。

## 从自定义镜像创建 Pod
