# GPU集群使用

## 环境配置

### 配置VPN连接

需要配置VPN以建立和集群的网络连接，其中VPN的账户密码的获取请联系尹伟老师。建立了VPN连接后，在secoclient中加入指定路由以使得对集群的访问通过VPN：

|IP地址|子网掩码|备注|
|-----|-------|----|
|172.16.0.0|255.255.0.0|**西安GPU集群**|
|10.1.0.0|255.255.0.0|蒙明伟集群物理机|
|10.2.0.0|255.255.0.0|蒙明伟集群虚拟机|

### kubectl安装

`kubectl` 为检查集群资源，创建、删除和更新组件，查看新集群，并启动实例应用程序的命令行工具。最详细的安装方式请参考[官方文档](https://kubernetes.io/zh/docs/tasks/tools/install-kubectl/)。

#### 从Github下载kubectl

若因为一些原因不能安装成功 `kubectl`，则可以从Github上直接下载。进入 Kubernetes 的[Github release 页面](https://github.com/kubernetes/kubernetes/releases)，在最新的release下找到CHANGELOG链接。以`v1.19.0`为例:

> Additional binary downloads are linked in the [CHANGELOG/CHANGELOG-1.19.md](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.19.md#downloads-for-v1190).

在`Client Binaries`一节找到自己系统对应的二进制文件：
|系统|二进制文件名|
|---|----------|
|MacOS|kubernetes-client-darwin-amd64.tar.gz|
|Linux（64位）|kubernetes-client-linux-amd64.tar.gz|
|Linux（32位）|kubernetes-client-linux-386.tar.gz|
|Windows（64位）|kubernetes-client-windows-amd64.tar.gz|
|Windows（32位）|kubernetes-client-windows-386.tar.gz|

解压并放入 `PATH` 中（Windows下将`kubectl.exe`加入环境变量）：

```bash
# MacOS or Linux
tar xzf kubernetes-client-*
chmod +x kubernetes/client/bin/kubectl
mv kubernetes/client/bin/kubectl /usr/local/bin/kubectl
```

确认安装成功

```bash
kubectl version
```

### kubeconfig获取

`kubectl`需要一个[kubeconfig 配置文件](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)使其找到并访问`Kubernetes`集群。通过VPN与集群网络建立连接后，用户可以[自主访问配置生成页下载](http://172.16.112.173/)用以访问集群的`kubeconfig`配置文件（用户名密码同VPN用户名密码，请咨询尹伟老师），同时集群会分配一个namespace给每个用户，**用户只有在各自的命名空间内有增删改查的权限**。

最后将`config`文件放在`~/.kube`下，并通过获取集群状态检查`kubectl`是否被正确配置：

```bash
kubectl cluster-info
```

## 集群使用

> **_NOTE:_** 集群的连接均采用命令行工具`kubectl`，建议阅读详细的 `kubectl` 教程 [kubectl book](https://kubectl.docs.kubernetes.io/)。

当创建了集群账号后集群会自动给用户分配[命名空间](http://172.16.112.173/)，在连接集群时需要指定具体的namespace，否则会提示权限错误。为了避免每次均输入命名空间参数，可以利用alias：

```bash
# .bashrc
alias kubectl="kubectl -n zhangsan1"  # zhangsan1为分配的namespace
```

之后都默认在自己的 namespace 下操作。

### 镜像列表

Docker镜像仓库位置：`172.16.112.173:30006/library/`。以下是提供的支持GPU的镜像列表，其他镜像可以从 Docker Hub 拉取。

|镜像|版本|GPU支持|备注|
|---|----|------|---|
|ubuntu-tensorflow|1.14.1, 2.3.0|是|Tensorflow版本1.14.1/2.3.0，CUDA版本10.1|
|ubuntu-pytorch|1.5.0|是|Pytorch版本1.5.0，CUDA版本10.1|

为了便于复制，这里列出yaml中镜像一行的常用选择

```yaml
image: 172.16.112.173:30006/library/ubuntu-tensorflow:2.3.0
image: 172.16.112.173:30006/library/ubuntu-tensorflow:1.14.1
image: 172.16.112.173:30006/library/ubuntu-pytorch:1.5.0
```

### 操作容器

以创建一个运行着支持 CUDA 和 Tensorflow 的 Ubuntu 18.04 容器为例。首先创建yaml配置文件（假设命名为myconfig.yaml，参考`ubuntu-tf-example.yaml`），之后根据配置部署：

```bash
kubectl apply -f myconfig.yaml
```

查看开启的 POD：

```bash
$ kubectl get pods
NAME                                    READY   STATUS             RESTARTS   AGE
my-first-ubuntu-tf-75b9d4ff7d-grk42   1/1     Running            0          19s
```

当 POD 状态为运行中时，可以连接到该Pod：

```bash
kubectl exec -it my-first-ubuntu-tf-75b9d4ff7d-grk42 -- bash
```

> **_NOTE:_** 查看是否启用了GPU支持可以使用`orion-smi`命令（`nvidia-smi`指令无效），但虚拟的GPU只有处于运行时才会显示。可以用`screen`或者`nohup`等命令后台执行GPU命令时查看。

> **_NOTE:_** 一机多卡情形下，开启的POD中虚拟显卡可能和虚拟CPU位于不同的物理机，需要注意CPU和GPU之间数据转移的效率问题。

测试成功获取了虚拟 GPU：

```bash
# pytorch
$ python -c "import torch; print(torch.cuda.is_available())"
...
True

# tensorflow
$ python -c "import tensorflow as tf; print(tf.test.is_gpu_available())"
...
True
```

删除容器：

```bash
kubectl delete deployment my-first-ubuntu-tf
```

### 持久化存储

POD 的本地文件是临时的，在每次重启（手动或失败重启）后都会恢复到最初的镜像状态。为了保持在 POD 中做的状态变化（例如创建了数据文件、日志文件等），需要向集群申请持久化存储资源。目前提供两种持久化存储方案：直接挂载NFS的共享盘和申请[PVC](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)，两种方式挂载成功后都可以通过`/share`访问持久化存储卷。

> **_NOTE:_** 提供持久化存储服务的硬盘并不保证和 POD 处于同一台物理机上，因此如果有大量的文件IO，建议先将数据拷贝到 POD 同一台物理机的磁盘，再从本地磁盘读写。

#### 直接挂载 NFS 共享盘

集群 NFS 向所有用户提供了持久化的共享存储空间，可以理解为一个远程的目录，请每位用户自行以自己分配到的集群命名空间为名建立文件夹（例如命名空间为`zhangsan1`则建立`share/zhangsan1`）。通过直接挂载NFS共享盘的方式参考`ubuntu-tf+nfs-direct-example.yaml`。

> **_WARNING:_** 严禁修改、删除他人的工作目录！

#### 申请 PVC

使用 PVC 和 POD 相似，都向集群申请临时资源，不同的是 PVC 申请的是存储资源，POD 申请的是计算资源。PVC 和 POD 的生命周期是独立的，重启 POD 后 PVC 中的数据并不会消失。通过 PVC 获得持久化存储参考`pvc-example.yaml`。申请 PVC 并在 POD 中挂载 PVC 参考`ubuntu-tf+pvc-example.yaml` （如果已经单独申请过 PVC，可以删去yaml文件开头申请 PVC 的部分）。

#### 挂载本地磁盘

每台物理机本地有两块SSD可供挂载，读写速度会比 NFS 共享盘和 PVC 高很多。不过本地磁盘只做临时缓存用，并不保证容错，可能会被清空，因此建议和持久盘配合使用。挂载本地磁盘参考`ubuntu-tf+local-disk-example.yaml`。需要同时申请持久化存储并挂载本地磁盘，可参考`ubuntu-tf+local-disk+pvc-example`。
